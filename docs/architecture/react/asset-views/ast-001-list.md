## AST-001 現在資産状況取得

## 1. 概要

本ドキュメントでは、AST-001 現在資産状況取得APIをReact・TypeScriptから利用する際の、フロントエンドにおける実装方針および責務分離方針を定義する。

AST-001は、操作対象利用者について、最新の確定済み対象年月における現在の資産状況を取得する読み取り専用APIである。

フロントエンドでは、主にダッシュボードや現在資産状況画面から本APIを利用し、以下の情報を表示する。

* 対象年月
* 総資産
* 利用可能資産
* 資産口座別資産状況
* 保有商品別資産状況

現在資産状況として使用する対象年月は、フロントエンド側で独自に判定せず、AST-001から返却された`targetYearMonth`を正式な対象年月として扱う。

また、総資産、利用可能資産、資産口座別資産額および保有商品別資産額についても、バックエンドで算出された値を正式値として使用する。

```text
現在資産状況画面
    ↓
AST-001 現在資産状況取得
    ↓
最新の確定済み対象年月
    ↓
targetYearMonth
totalAssets
availableAssets
assetAccounts
    ↓
Reactコンポーネントで表示
```

フロントエンドでは、資産口座の残高記録単位や利用可能資産設定をもとに資産額を再計算しない。

```text
資産状況の算出
    → バックエンドの責務

API呼び出し・状態管理
    → Query / Custom Hookの責務

金額・年月・資産内訳の表示
    → Reactコンポーネントの責務
```

AST-001は読み取り専用のGET APIであるため、React Query等ではQueryとして管理する。

取得中、正常な0円、資産口座0件、確定済み月末資産状況なし、APIエラーはそれぞれ異なる状態として扱い、未取得状態や業務上の空状態を0円として代替表示しない。

また、AST-001は最新の確定済み月末資産状況を参照するため、月末資産状況の確定・確定解除によって取得対象が変化する可能性がある。

そのため、SNP-004 月末資産状況確定およびSNP-005 月末資産状況確定解除の成功後は、AST-001のQueryをinvalidateし、最新の現在資産状況を再取得する。

過去の特定年月を表示する場合はAST-002 指定年月資産状況取得、複数月の資産推移を表示する場合はAST-003 資産推移取得を使用し、AST-001へ過剰な責務を持たせない。

本ドキュメントでは、これらを前提として、AST-001における型定義、API Client、Query Key、Custom Hook、キャッシュ、エラー処理および関連APIとの連携方針を定義する。

---

### 2 React・TypeScriptでの利用

本APIでは、
パスパラメータ、
クエリパラメータ、
リクエストボディを使用しない。

レスポンス型は、
以下とする。

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

export type CurrentAssetSummary = {
  targetYearMonth: string;
  totalAssets: number;
  availableAssets: number;
  assetAccounts: AssetAccountSummary[];
};

export type GetCurrentAssetSummaryResponse = {
  data: CurrentAssetSummary;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetCurrentAssetSummaryResponse>(
    '/api/v1/asset-summaries/current',
  );
```

ダッシュボードや
現在資産状況画面などから、
最新の確定済み対象年月における
資産状況を表示する際に利用する。

---

#### 2.1 targetYearMonthの扱い

`targetYearMonth`は、
現在資産状況として採用された
最新の確定済み対象年月を表す。

形式は、

```text
YYYY-MM
```

とする。

例：

```ts
const targetYearMonth =
  response.data.targetYearMonth;
```

フロントエンド側で
最新対象年月を独自に決定しない。

AST-001のレスポンスとして
返却された`targetYearMonth`を
現在資産状況の対象年月として扱う。

---

#### 2.2 totalAssetsの扱い

`totalAssets`は、
対象年月における
総資産額として扱う。

```ts
const totalAssets =
  response.data.totalAssets;
```

フロントエンド側で、

```text
月末資産残高
+
商品別月末評価額
```

を再集計して
総資産を算出しない。

総資産の算出責務は、
バックエンドへ集約する。

---

#### 2.3 availableAssetsの扱い

`availableAssets`は、
対象年月時点の
利用可能資産額として扱う。

```ts
const availableAssets =
  response.data.availableAssets;
```

フロントエンド側で
現在の利用可能資産設定から
再計算しない。

対象年月時点の
利用可能資産設定を反映した値として、
APIレスポンスをそのまま使用する。

---

#### 2.4 assetAccountsの扱い

`assetAccounts`は、
対象年月における
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
- 利用可能資産かどうか
- 保有商品別資産状況

---

#### 2.5 assetAccountIdの扱い

`assetAccountId`は、
API共通方針に従って
stringとして扱う。

```ts
const assetAccountId: string =
  account.assetAccountId;
```

numberへ変換して
業務計算には使用しない。

資産口座詳細画面などへの
ルーティングに使用できる。

---

#### 2.6 assetAmountの扱い

`assetAmount`は、
対象年月における
資産口座または
保有商品の資産額として扱う。

```ts
const assetAmount =
  account.assetAmount;
```

資産口座の
残高記録単位によって
フロントエンド側で
算出方法を切り替えない。

APIから返却された
`assetAmount`をそのまま表示する。

---

#### 2.7 availableの扱い

`available`は、
対象年月時点で
その資産口座が
利用可能資産として
扱われるかを表す。

```ts
if (account.available) {
  // 利用可能資産として表示
}
```

現在の資産口座設定を
別APIから取得して
AST-001の`available`を
上書きしない。

---

#### 2.8 holdingAssetsの扱い

`holdingAssets`は、
商品単位で管理する
資産口座について、
保有商品別の資産状況を表す。

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

口座単位で管理する
資産口座では、

```ts
account.holdingAssets.length === 0
```

となる。

空配列を
データ取得エラーとして
扱わない。

---

#### 2.9 holdingAssetIdの扱い

`holdingAssetId`は、
API共通方針に従って
stringとして扱う。

```ts
const holdingAssetId: string =
  holding.holdingAssetId;
```

保有商品詳細画面などへの
ルーティングに使用できる。

---

#### 2.10 金額表示

金額は、
日本円のintegerとして
APIから返却される。

画面表示時は、
フロントエンドで
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

例：

```tsx
<span>
  {formatYen(data.totalAssets)}
</span>
```

表示形式と
業務上の金額計算は分離する。

---

#### 2.11 Queryとして扱う

AST-001は、
読み取り専用のGET APIであるため、
React Query等では
Queryとして扱う。

概念例：

```ts
export const fetchCurrentAssetSummary =
  async () => {
    const response =
      await apiClient.get<GetCurrentAssetSummaryResponse>(
        '/api/v1/asset-summaries/current',
      );

    return response.data;
  };
```

---

#### 2.12 Query Key

Query Keyは、
現在資産状況を表す
固定キーとして管理できる。

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
};
```

操作対象利用者は、
`X-User-Id`によって
API Clientまたは
共通Contextから付与する。

利用者切り替え時は、
現在資産状況のキャッシュを
再取得できるようにする。

---

#### 2.13 Hook実装例

React Queryを使用する場合の
概念例は、
以下とする。

```ts
export const useCurrentAssetSummary =
  () => {
    return useQuery({
      queryKey:
        assetSummaryKeys.current,

      queryFn:
        fetchCurrentAssetSummary,
    });
  };
```

実際のAPI Client、
React Queryおよび
キャッシュ方針は、
フロントエンド共通設計に従う。

---

#### 2.14 ローディング表示

AST-001取得中は、
ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return <Loading />;
}
```

現在資産状況が
まだ取得できていない状態で、
0円の資産状況として
表示しない。

```text
未取得
≠
総資産0円
```

を区別する。

---

#### 2.15 総資産0円の扱い

正常レスポンスとして

```text
totalAssets = 0
```

が返却された場合は、
有効な資産状況として扱う。

```ts
if (
  data.totalAssets === 0
) {
  // 正常な0円表示
}
```

以下のような
truthy / falsy判定によって
未取得扱いにしない。

```ts
if (!data.totalAssets) {
  // 0円もfalseになるため使用しない
}
```

---

#### 2.16 利用可能資産0円の扱い

`availableAssets = 0`も、
正常な業務値として扱う。

例えば、
すべての資産口座が
利用可能資産対象外の場合に
0円となり得る。

```ts
const availableAssets =
  data.availableAssets;
```

0円を
エラーとして表示しない。

---

#### 2.17 資産口座0件の扱い

確定済み月末資産状況が存在し、
APIが正常レスポンスを返したうえで
`assetAccounts`が空配列となるケースが
仕様上発生する場合は、
正常な空状態として表示する。

```tsx
if (
  data.assetAccounts.length === 0
) {
  return (
    <EmptyState>
      表示できる資産口座がありません。
    </EmptyState>
  );
}
```

ただし、
確定済み月末資産状況自体が
存在しない場合とは区別する。

---

#### 2.18 CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND

`CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND`
が返却された場合は、
現在資産状況として表示できる
確定済み月末資産状況が
存在しないことを表示する。

表示例：

```text
確定済みの月末資産状況がありません。
月末資産状況を登録・確定してください。
```

必要に応じて、
月末資産管理画面への
導線を表示する。

この状態を、

```text
totalAssets = 0
```

として表示してはならない。

---

#### 2.19 USER_NOT_FOUND

`USER_NOT_FOUND`
が返却された場合は、
操作対象利用者を
現在利用できない状態として扱う。

利用者選択画面へ戻すなど、
API共通方針に従って処理する。

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
| `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` | 確定済み月末資産状況がないことを表示する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

#### 2.21 自動リトライ

AST-001は
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
```

など、
再送しても解消しない
クライアントまたは
業務状態起因のエラーは、
不要にリトライしない。

---

#### 2.22 クライアントキャッシュ

AST-001は、
React Query等による
クライアントキャッシュの
対象としてよい。

ただし、
最新の確定済み月末資産状況が
変更されると、
AST-001のレスポンスも変化する。

そのため、
月末資産状況の確定・確定解除後は、
必要に応じて
現在資産状況のQueryを
invalidateする。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey:
    assetSummaryKeys.current,
});
```

---

#### 2.23 SNP-004との連携

SNP-004 月末資産状況確定APIが
成功した場合は、
AST-001の取得対象年月が
変化する可能性がある。

例えば、

```text
確定前
2026-06 confirmed = true
2026-07 confirmed = false

    ↓ SNP-004

確定後
2026-07 confirmed = true
```

となった場合は、
AST-001を再取得する。

```text
targetYearMonth
2026-06
    ↓
2026-07
```

へ更新される。

---

#### 2.24 SNP-005との連携

SNP-005 月末資産状況確定解除APIによって、
現在資産状況として使用していた
最新年月が未確定になった場合は、
AST-001の対象年月が
1つ前の確定済み年月へ
変化する可能性がある。

そのため、
SNP-005成功後も
AST-001のQueryを
invalidateする。

---

#### 2.25 BAL・VAL系APIとの連携

BAL系およびVAL系APIは、
未確定の月末資産状況に対する
残高・評価額の登録更新を行う。

AST-001は
確定済み月末資産状況のみを
対象とするため、
未確定月のBAL・VAL更新だけでは
AST-001の結果は変化しない。

```text
BAL / VAL更新
    ↓
対象snapshotが未確定
    ↓
AST-001には反映されない

SNP-004で確定
    ↓
AST-001再取得
    ↓
反映される
```

この関係を
フロントエンドでも前提とする。

---

#### 2.26 AST-002との使い分け

AST-001は、
最新の確定済み対象年月を
自動的に取得する。

過去の特定年月を
表示したい場合は、
AST-002 指定年月資産状況取得APIを使用する。

```text
現在の資産状況
    → AST-001

2026-05の資産状況
    → AST-002
```

フロントエンド側で
AST-001を利用して
過去月を再現しない。

---

#### 2.27 AST-003との使い分け

AST-001は、
1つの対象年月について
資産状況の詳細を取得する。

複数月の推移を表示する場合は、
AST-003 資産推移取得APIを使用する。

```text
最新月の内訳
    → AST-001

複数月のグラフ
    → AST-003
```

AST-001を
月数分繰り返し呼び出して
資産推移を生成しない。

---

#### 2.28 フロントエンドで総資産を再計算しない

APIレスポンスには、

```text
totalAssets
assetAccounts[].assetAmount
```

の両方が含まれる。

フロントエンドでは、
表示時に

```ts
const totalAssets =
  data.assetAccounts.reduce(
    (sum, account) =>
      sum + account.assetAmount,
    0,
  );
```

のように
総資産を再計算して
APIの`totalAssets`を
置き換えない。

バックエンドで算出された
`totalAssets`を正式値として扱う。

資産口座別合計との一致確認は、
バックエンドテストで保証する。

---

#### 2.29 フロントエンドでavailableAssetsを再計算しない

同様に、

```ts
const availableAssets =
  data.assetAccounts
    .filter(
      (account) =>
        account.available,
    )
    .reduce(
      (sum, account) =>
        sum + account.assetAmount,
      0,
    );
```

によって
APIレスポンスの
`availableAssets`を
置き換えない。

APIから返却された
`availableAssets`を
正式値として表示する。

---

### 3 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [AST-002 指定年月資産状況取得](./ast-002-detail.md)
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
