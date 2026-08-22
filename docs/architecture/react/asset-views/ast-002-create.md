##  AST-002 指定年月資産状況取得## 1. 概要

本ドキュメントでは、AST-002 指定年月資産状況取得APIをReact・TypeScriptから利用する際の、フロントエンドにおける実装方針および責務分離方針を定義する。

AST-002は、操作対象利用者について、指定された対象年月における確定済みの資産状況を取得する読み取り専用APIである。

フロントエンドでは、主に過去資産状況画面や資産推移画面から本APIを利用し、指定年月について以下の情報を表示する。

* 対象年月
* 総資産
* 利用可能資産
* 資産口座別資産状況
* 保有商品別資産状況

取得対象年月は、`YYYY-MM`形式の`targetYearMonth`として管理し、パスパラメータとしてAPIへ渡す。

```text
過去資産状況画面
    ↓
対象年月を選択
    ↓
targetYearMonth
    ↓
AST-002 指定年月資産状況取得
    ↓
指定年月の確定済み資産状況
    ↓
Reactコンポーネントで表示
```

資産状況として使用する総資産、利用可能資産、資産口座別資産額および保有商品別資産額は、バックエンドで算出された値を正式値として扱う。

フロントエンドでは、月末資産残高、商品別月末評価額、資産口座の残高記録単位および利用可能資産設定をもとに資産額を再計算しない。

```text
指定年月の資産状況算出
    → バックエンドの責務

対象年月の選択・URL管理
    → フロントエンドの責務

API呼び出し・キャッシュ管理
    → Query / Custom Hookの責務

資産状況の表示
    → Reactコンポーネントの責務
```

AST-002は読み取り専用のGET APIであるため、React Query等ではQueryとして管理し、`targetYearMonth`をQuery Keyへ含めることで対象年月ごとにキャッシュを分離する。

取得中、正常な0円、資産口座0件、指定年月に確定済み資産状況が存在しない状態、入力値不正およびAPIエラーは、それぞれ異なる状態として扱う。

特に、指定年月に月末資産状況が存在していても未確定である場合は、確定済み資産状況として代替表示せず、`CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND`として扱う。

AST-001 現在資産状況取得とは取得対象年月の決定方法によって使い分ける。最新の確定済み資産状況を表示する場合はAST-001、利用者が指定した過去の対象年月を表示する場合はAST-002を使用する。

また、AST-003 資産推移取得による時系列表示から特定年月を選択し、その年月の詳細をAST-002で取得する構成を想定する。

本ドキュメントでは、これらを前提として、AST-002における型定義、対象年月の管理、API Client、Query Key、Custom Hook、キャッシュ、エラー処理およびAST系API間の連携方針を定義する。

---

### 2 React・TypeScriptでの利用

本APIでは、
パスパラメータとして
`targetYearMonth`を使用する。

パスパラメータの型は、
以下とする。

```ts
export type GetAssetSummaryByMonthParams = {
  targetYearMonth: string;
};
```

資産状況のレスポンス型は、
AST-001 現在資産状況取得APIと
共通化してよい。

```ts
export type HoldingAssetSummary = {
  holdingAssetId: string;
  name: string;
  assetAmount: number;
};

export type AssetAccountSummary = {
  assetAccountId: string;
  name: string;
  assetAmount: number;
  available: boolean;
  holdingAssets: HoldingAssetSummary[];
};

export type AssetSummary = {
  targetYearMonth: string;
  totalAssets: number;
  availableAssets: number;
  assetAccounts: AssetAccountSummary[];
};

export type GetAssetSummaryByMonthResponse = {
  data: AssetSummary;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetAssetSummaryByMonthResponse>(
    `/api/v1/asset-summaries/${targetYearMonth}`,
  );
```

資産推移画面や
過去資産状況画面などから、
特定年月の資産状況を表示する際に利用する。

---

#### 2.1 targetYearMonthの扱い

`targetYearMonth`は、
`YYYY-MM`形式のstringとして扱う。

例：

```ts
const targetYearMonth =
  '2026-05';
```

フロントエンド側でも、
基本的には
stringのまま扱う。

```ts
type TargetYearMonth = string;
```

日付計算が必要な場合を除き、
`Date`へ変換して
APIへ再変換する必要はない。

---

#### 2.2 URL生成

`targetYearMonth`は、
パスパラメータとして使用する。

```ts
const url =
  `/api/v1/asset-summaries/${targetYearMonth}`;
```

例えば、

```ts
targetYearMonth = '2026-05';
```

の場合、

```http
GET /api/v1/asset-summaries/2026-05
```

となる。

---

#### 2.3 targetYearMonthの入力UI

利用者が
対象年月を選択できるUIを
提供する場合は、
年月単位で入力できる
コントロールを使用する。

概念例：

```tsx
<input
  type="month"
  value={targetYearMonth}
  onChange={(event) =>
    setTargetYearMonth(
      event.target.value,
    )
  }
/>
```

ブラウザから取得した値が、

```text
YYYY-MM
```

形式であることを前提とする。

ただし、
API側でも必ず
形式検証を行う。

---

#### 2.4 Queryとして扱う

AST-002は、
読み取り専用のGET APIであるため、
React Query等では
Queryとして扱う。

概念例：

```ts
export const fetchAssetSummaryByMonth =
  async (
    targetYearMonth: string,
  ) => {
    const response =
      await apiClient.get<GetAssetSummaryByMonthResponse>(
        `/api/v1/asset-summaries/${targetYearMonth}`,
      );

    return response.data;
  };
```

---

#### 2.5 Query Key

Query Keyには、
`targetYearMonth`を含める。

概念例：

```ts
export const assetSummaryKeys = {
  all: [
    'assetSummaries',
  ] as const,

  current: [
    'assetSummaries',
    'current',
  ] as const,

  byMonth: (
    targetYearMonth: string,
  ) =>
    [
      'assetSummaries',
      'byMonth',
      targetYearMonth,
    ] as const,
};
```

これにより、
対象年月ごとに
キャッシュを分離できる。

---

#### 2.6 Hook実装例

React Queryを使用する場合の
概念例は、
以下とする。

```ts
export const useAssetSummaryByMonth =
  (
    targetYearMonth: string,
  ) => {
    return useQuery({
      queryKey:
        assetSummaryKeys.byMonth(
          targetYearMonth,
        ),

      queryFn:
        () =>
          fetchAssetSummaryByMonth(
            targetYearMonth,
          ),

      enabled:
        targetYearMonth.length > 0,
    });
  };
```

実際のAPI Client、
React Queryおよび
キャッシュ方針は、
フロントエンド共通設計に従う。

---

#### 2.7 レスポンスのtargetYearMonth

レスポンスの
`targetYearMonth`には、
実際に取得対象となった
確定済み対象年月が返却される。

AST-002では、
正常時は
リクエストで指定した値と
一致する。

```ts
const requestedMonth =
  targetYearMonth;

const responseMonth =
  data.targetYearMonth;
```

正常時は、

```text
requestedMonth
=
responseMonth
```

となる。

---

#### 2.8 totalAssetsの扱い

`totalAssets`は、
指定年月における
総資産額として扱う。

```ts
const totalAssets =
  data.totalAssets;
```

フロントエンド側で
月末資産残高や
商品別月末評価額を
再取得して
再計算しない。

---

#### 2.9 availableAssetsの扱い

`availableAssets`は、
指定年月時点の
利用可能資産額として扱う。

```ts
const availableAssets =
  data.availableAssets;
```

現在の利用可能資産設定から
再計算しない。

AST-002から返却された値を
指定年月時点の正式値として
表示する。

---

#### 2.10 assetAccountsの扱い

`assetAccounts`は、
指定年月における
資産口座別資産状況として扱う。

概念例：

```tsx
{data.assetAccounts.map(
  (account) => (
    <AssetAccountRow
      key={account.assetAccountId}
      account={account}
    />
  ),
)}
```

各資産口座について、
以下を表示できる。

- 資産口座名
- 資産額
- 指定年月時点の利用可能状態
- 保有商品別資産状況

---

#### 2.11 holdingAssetsの扱い

商品単位で管理する
資産口座について、
`holdingAssets`から
保有商品別資産状況を表示する。

概念例：

```tsx
{account.holdingAssets.map(
  (holding) => (
    <HoldingAssetRow
      key={holding.holdingAssetId}
      holdingAsset={holding}
    />
  ),
)}
```

口座単位の資産口座では、

```text
holdingAssets = []
```

となる。

空配列を
APIエラーとして扱わない。

---

#### 2.12 IDの扱い

以下のIDは、
API共通方針に従って
stringとして扱う。

```text
assetAccountId
holdingAssetId
```

例えば、

```ts
const assetAccountId: string =
  account.assetAccountId;

const holdingAssetId: string =
  holding.holdingAssetId;
```

numberへ変換して
業務計算には使用しない。

---

#### 2.13 金額表示

金額は、
日本円のintegerとして
APIから返却される。

画面表示時は、
表示用フォーマットを適用する。

概念例：

```ts
export const formatYen =
  (amount: number): string =>
    new Intl.NumberFormat(
      'ja-JP',
      {
        style: 'currency',
        currency: 'JPY',
        maximumFractionDigits: 0,
      },
    ).format(amount);
```

例えば、

```tsx
<span>
  {formatYen(data.totalAssets)}
</span>
```

のように表示する。

---

#### 2.14 0円の扱い

以下は、
正常な業務値として扱う。

```text
totalAssets = 0
availableAssets = 0
assetAmount = 0
```

truthy / falsyによって
未取得扱いしない。

以下のような判定は避ける。

```ts
if (!data.totalAssets) {
  // 0円まで未取得扱いになるため使用しない
}
```

---

#### 2.15 ローディング表示

AST-002取得中は、
ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return <Loading />;
}
```

未取得状態と
0円の資産状況を
明確に区別する。

---

#### 2.16 targetYearMonthの変更

利用者が
対象年月を変更した場合は、
新しい`targetYearMonth`を
Query Keyへ反映し、
AST-002を再取得する。

概念例：

```ts
setTargetYearMonth(
  '2026-06',
);
```

これにより、

```text
2026-05
    ↓
2026-06
```

のように
表示対象を切り替える。

---

#### 2.17 CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND

`CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND`
が返却された場合は、
指定年月について
表示可能な確定済み資産状況が
存在しないことを表示する。

表示例：

```text
指定した年月には、
確定済みの資産状況がありません。
```

この場合、
別年月の資産状況を
フロントエンド側で
自動表示しない。

---

#### 2.18 未確定月の扱い

指定年月に
月末資産状況が存在していても、
未確定の場合は
AST-002では取得できない。

フロントエンドでは、
`CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

以下のように、
未確定データ取得用APIへ
自動的に切り替えない。

```text
AST-002失敗
    ↓
未確定データを別APIで取得
    ↓
資産状況として代替表示
```

このような処理は行わない。

---

#### 2.19 VALIDATION_ERROR

`targetYearMonth`が
不正な形式の場合は、

`VALIDATION_ERROR`

として扱う。

例えば、

```text
2026-13
```

などが該当する。

通常の年月選択UIでは
発生しないことを前提とするが、
URL直接入力などに備えて
エラー処理を行う。

---

#### 2.20 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 不正な対象年月またはURLとして扱う |
| `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` | 指定年月に確定済み資産状況がないことを表示する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

#### 2.21 自動リトライ

AST-002は、
読み取り専用GET APIであり、
冪等である。

そのため、
ネットワークエラーや
一時的な5xxエラーに対して、
React Query等の
標準的な自動リトライを
利用してよい。

ただし、

```text
400
404
422
```

など、
再送しても解消しないエラーは
不要にリトライしない。

---

#### 2.22 クライアントキャッシュ

AST-002は、
React Query等による
クライアントキャッシュの
対象としてよい。

対象年月ごとに
Query Keyを分ける。

```text
2026-05
    → 独立したキャッシュ

2026-06
    → 独立したキャッシュ
```

利用者切り替え時は、
操作対象利用者に応じて
キャッシュを再取得または
無効化する。

---

#### 2.23 AST-001との使い分け

AST-001は、
最新の確定済み対象年月を
サーバー側で決定する。

AST-002は、
利用者が指定した年月を取得する。

```text
最新の資産状況
    → AST-001

2026-05の資産状況
    → AST-002
```

フロントエンド側で
AST-002へ最新年月を毎回指定して
AST-001の代替としない。

---

#### 2.24 AST-003との連携

AST-003 資産推移取得APIで
時系列グラフを表示し、
特定年月を選択した場合に、
AST-002を使用して
その年月の詳細を取得できる。

概念例：

```text
AST-003
2026-01〜2026-06の推移表示
    ↓
2026-05を選択
    ↓
AST-002
2026-05の資産状況詳細
```

---

#### 2.25 グラフからの遷移例

概念的には、
以下のように
対象年月をURLへ含めてもよい。

```tsx
navigate(
  `/assets/2026-05`,
);
```

詳細画面では、
ルートパラメータから
`targetYearMonth`を取得し、
AST-002を呼び出す。

---

#### 2.26 フロントエンドで再集計しない

APIレスポンスとして返却された

- `totalAssets`
- `availableAssets`
- `assetAccounts[].assetAmount`

を正式値として扱う。

フロントエンド側で
月末資産データを再取得し、
独自集計して
API結果を置き換えない。

---

#### 2.27 現在の設定で過去月を書き換えない

AST-002の
`available`および
`availableAssets`は、
指定年月時点の
利用可能資産設定を反映している。

フロントエンド側で
現在の設定を取得して、

```ts
account.available =
  currentSetting.available;
```

のように
書き換えてはならない。

---

### 3 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [AST-001 現在資産状況取得](./ast-001-list.md)
- [AST-003 資産推移取得](./ast-003-list.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/asset-views.md)
- [Reactアーキテクチャ設計](../../../architecture/react/asset-views.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)

