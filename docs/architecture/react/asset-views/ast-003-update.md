##  AST-003 資産推移取得

### 1. 概要

本ドキュメントでは、AST-003 資産推移取得APIをReact・TypeScriptから利用する際の、フロントエンドにおける実装方針および責務分離方針を定義する。

AST-003は、操作対象利用者について、指定期間内の確定済み月末資産状況をもとに、複数月の資産推移を取得する読み取り専用APIである。

フロントエンドでは、主に資産推移画面から本APIを利用し、対象年月ごとの以下の情報をグラフまたは一覧として表示する。

* 総資産
* 利用可能資産
* 総資産の前月差分
* 利用可能資産の前月差分
* 資産口座別資産額
* 保有商品別資産額

取得対象期間は、`YYYY-MM`形式の`from`および`to`として管理し、クエリパラメータとしてAPIへ渡す。

```text
資産推移画面
    ↓
表示期間を指定
    ↓
from / to
    ↓
AST-003 資産推移取得
    ↓
指定期間内の確定済み月末資産状況
    ↓
targetYearMonth昇順の時系列データ
    ↓
グラフ・一覧として表示
```

資産推移として使用する総資産、利用可能資産、前月差分、資産口座別資産額および保有商品別資産額は、バックエンドで算出された値を正式値として扱う。

フロントエンドでは、月末資産残高、商品別月末評価額、利用可能資産設定などを別途取得し、資産推移や前月差分を独自に再計算しない。

```text
各年月の資産状況・前月差分の算出
    → バックエンドの責務

表示期間の入力・状態管理
    → フロントエンドの責務

API呼び出し・キャッシュ管理
    → Query / Custom Hookの責務

グラフ・一覧用データへの表示変換
    → フロントエンドの責務
```

AST-003は読み取り専用のGET APIであるため、React Query等ではQueryとして管理し、`from`および`to`をQuery Keyへ含めることで表示期間ごとにキャッシュを分離する。

未確定月または月末資産状況が存在しない年月は`trends`へ含まれない。フロントエンドでは、このような欠損月を0円として補完せず、データ不存在または未確定の状態として扱う。

また、`totalAssetsDifference`および`availableAssetsDifference`が`null`の場合は、暦上の前月との比較ができない状態を表す。`null`と0円差分を区別し、フロントエンド側で配列上の直前データから差分を再計算しない。

```text
0
    → 正常な0円または増減なし

null
    → 前月比較不可

trendsに年月が存在しない
    → データ不存在または未確定
```

AST-003の結果は確定済み月末資産状況をもとに構成されるため、SNP-004 月末資産状況確定およびSNP-005 月末資産状況確定解除によって、対象年月だけでなく隣接月の前月差分も変化する可能性がある。

そのため、確定・確定解除の成功後は資産推移Queryをinvalidateし、資産推移全体を再取得する。

AST系APIは、表示目的に応じて以下のように使い分ける。

```text
最新月の資産状況
    → AST-001

特定月の資産状況
    → AST-002

複数月の資産推移
    → AST-003
```

資産推移グラフから特定年月を選択した場合は、その`targetYearMonth`を使用してAST-002へ遷移し、対象年月の詳細な資産状況を取得する構成を想定する。

本ドキュメントでは、これらを前提として、AST-003における型定義、期間指定、API Client、Query Key、Custom Hook、グラフ表示、欠損月・前月差分の扱い、キャッシュ、エラー処理および関連APIとの連携方針を定義する。


---

### 2 React・TypeScriptでの利用

本APIでは、
クエリパラメータとして
`from`および`to`を使用する。

クエリパラメータの型は、
以下とする。

```ts
export type GetAssetTrendQuery = {
  from: string;
  to: string;
};
```

資産口座別・保有商品別の型は、
AST-001およびAST-002と
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
```

1ヶ月分の
資産推移データの型は、
以下とする。

```ts
export type AssetTrendItem = {
  targetYearMonth: string;
  totalAssets: number;
  availableAssets: number;
  totalAssetsDifference: number | null;
  availableAssetsDifference: number | null;
  assetAccounts: AssetAccountSummary[];
};
```

資産推移全体の型は、
以下とする。

```ts
export type AssetTrend = {
  from: string;
  to: string;
  trends: AssetTrendItem[];
};

export type GetAssetTrendResponse = {
  data: AssetTrend;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetAssetTrendResponse>(
    '/api/v1/asset-trends',
    {
      params: {
        from,
        to,
      },
    },
  );
```

資産推移画面などから、
指定期間の資産推移を
グラフまたは一覧で
表示する際に利用する。

---

#### 2.1 fromの扱い

`from`は、
表示対象期間の開始年月として扱う。

形式は、

```text
YYYY-MM
```

とする。

例：

```ts
const from = '2026-01';
```

フロントエンド側でも、
基本的にはstringとして扱う。

---

#### 2.2 toの扱い

`to`は、
表示対象期間の終了年月として扱う。

形式は、

```text
YYYY-MM
```

とする。

例：

```ts
const to = '2026-06';
```

---

#### 2.3 表示対象期間の入力UI

利用者が
表示対象期間を変更できる場合は、
開始年月および終了年月を
年月単位で選択できるUIとする。

概念例：

```tsx
<input
  type="month"
  value={from}
  onChange={(event) =>
    setFrom(
      event.target.value,
    )
  }
/>

<input
  type="month"
  value={to}
  onChange={(event) =>
    setTo(
      event.target.value,
    )
  }
/>
```

フロントエンドでも
基本的な入力制御を行ってよいが、
正式なバリデーションは
バックエンドでも必ず実施する。

---

#### 2.4 fromとtoの前後関係

フロントエンドでは、
利用者操作時点で

```text
from <= to
```

となるように
入力制御してよい。

例えば、

```ts
const isInvalidRange =
  from > to;
```

として、
不正な期間では
検索ボタンを無効化してよい。

ただし、
バックエンド側の
バリデーションを省略しない。

---

#### 2.5 Queryとして扱う

AST-003は、
読み取り専用のGET APIであるため、
React Query等では
Queryとして扱う。

概念例：

```ts
export const fetchAssetTrend =
  async ({
    from,
    to,
  }: GetAssetTrendQuery) => {
    const response =
      await apiClient.get<GetAssetTrendResponse>(
        '/api/v1/asset-trends',
        {
          params: {
            from,
            to,
          },
        },
      );

    return response.data;
  };
```

---

#### 2.6 Query Key

Query Keyには、
`from`および`to`を含める。

概念例：

```ts
export const assetTrendKeys = {
  all: [
    'assetTrends',
  ] as const,

  range: (
    from: string,
    to: string,
  ) =>
    [
      ...assetTrendKeys.all,
      from,
      to,
    ] as const,
};
```

これにより、
表示対象期間ごとに
キャッシュを分離できる。

---

#### 2.7 Hook実装例

React Queryを使用する場合の
概念例は、
以下とする。

```ts
export const useAssetTrend =
  (
    from: string,
    to: string,
  ) => {
    return useQuery({
      queryKey:
        assetTrendKeys.range(
          from,
          to,
        ),

      queryFn:
        () =>
          fetchAssetTrend({
            from,
            to,
          }),

      enabled:
        from.length > 0
        && to.length > 0
        && from <= to,
    });
  };
```

実際のAPI Client、
React Queryおよび
キャッシュ方針は、
フロントエンド共通設計に従う。

---

#### 2.8 trendsの扱い

`trends`は、
対象期間内の
確定済み月末資産状況を
`targetYearMonth`昇順で
保持する。

概念例：

```tsx
{data.trends.map(
  (trend) => (
    <AssetTrendRow
      key={trend.targetYearMonth}
      trend={trend}
    />
  ),
)}
```

フロントエンド側で
独自に並べ替えることを
前提としない。

---

#### 2.9 欠損月の扱い

AST-003では、
未確定月または
月末資産状況が存在しない年月は、
`trends`へ含まれない。

例えば、

```text
2026-01
2026-03
```

が返却された場合、
`2026-02`を
フロントエンド側で
0円として補完しない。

以下のような処理は行わない。

```ts
const completedTrends =
  fillMissingMonthsWithZero(
    data.trends,
  );
```

欠損月は、
データ不存在または
未確定であることを意味し、
0円を意味しない。

---

#### 2.10 0円の扱い

以下は、
正常な業務値として扱う。

```text
totalAssets = 0
availableAssets = 0
assetAmount = 0
```

確定済みの対象年月として
`trends`に含まれている場合は、
0円として正常表示する。

```ts
if (
  trend.totalAssets === 0
) {
  // 正常な0円表示
}
```

0円を
データ不存在と混同しない。

---

#### 2.11 totalAssetsの扱い

`totalAssets`は、
対象年月における
総資産額として扱う。

```ts
const totalAssets =
  trend.totalAssets;
```

フロントエンド側で
`assetAccounts`から
再集計して
正式値を置き換えない。

---

#### 2.12 availableAssetsの扱い

`availableAssets`は、
対象年月時点の
利用可能資産額として扱う。

```ts
const availableAssets =
  trend.availableAssets;
```

現在の
利用可能資産設定から
再計算しない。

---

#### 2.13 totalAssetsDifferenceの扱い

`totalAssetsDifference`は、
暦上の前月からの
総資産増減額を表す。

```ts
const difference =
  trend.totalAssetsDifference;
```

正数の場合は、
総資産が増加している。

```text
+200000
```

負数の場合は、
総資産が減少している。

```text
-100000
```

`null`の場合は、
前月比較不可として扱う。

---

#### 2.14 availableAssetsDifferenceの扱い

`availableAssetsDifference`は、
暦上の前月からの
利用可能資産増減額を表す。

```ts
const difference =
  trend.availableAssetsDifference;
```

`null`の場合は、
前月比較不可として扱う。

---

#### 2.15 前月比較不可の表示

差分項目が`null`の場合は、
0円差分として表示しない。

以下のような
表示を行ってよい。

```text
前月比：-
```

概念例：

```tsx
<span>
  {trend.totalAssetsDifference === null
    ? '-'
    : formatYen(
        trend.totalAssetsDifference,
      )}
</span>
```

```text
null
≠
0
```

を明確に区別する。

---

#### 2.16 増減表示

前月差分に応じて、
表示を切り替えてよい。

概念例：

```ts
const getDifferenceLabel =
  (
    difference: number | null,
  ): string => {
    if (difference === null) {
      return '-';
    }

    if (difference > 0) {
      return `+${formatYen(
        difference,
      )}`;
    }

    return formatYen(
      difference,
    );
  };
```

色やアイコンなどの
具体的な表現は、
画面設計に従う。

---

#### 2.17 資産推移グラフ

`trends`を使用して、
総資産の推移グラフを
表示できる。

概念的なデータ変換例：

```ts
const totalAssetChartData =
  data.trends.map(
    (trend) => ({
      targetYearMonth:
        trend.targetYearMonth,

      value:
        trend.totalAssets,
    }),
  );
```

グラフライブラリの選定は、
フロントエンド設計に従う。

---

#### 2.18 利用可能資産推移グラフ

利用可能資産についても、
同様にグラフ表示できる。

```ts
const availableAssetChartData =
  data.trends.map(
    (trend) => ({
      targetYearMonth:
        trend.targetYearMonth,

      value:
        trend.availableAssets,
    }),
  );
```

---

#### 2.19 欠損月のグラフ表示

APIレスポンスに
存在しない年月を
フロントエンド側で
0円として追加しない。

グラフライブラリ上で
欠損期間をどのように
視覚表現するかは、
画面設計で定める。

ただし、
APIデータとして

```text
totalAssets = 0
```

を生成してはならない。

---

#### 2.20 資産口座別推移

各`trend.assetAccounts`を利用して、
資産口座別の
資産推移を表示できる。

例えば、
特定の`assetAccountId`について、
各年月の値を抽出する。

```ts
const accountTrend =
  data.trends.map(
    (trend) => {
      const account =
        trend.assetAccounts.find(
          (item) =>
            item.assetAccountId
            === targetAssetAccountId,
        );

      return {
        targetYearMonth:
          trend.targetYearMonth,

        assetAmount:
          account?.assetAmount
          ?? null,
      };
    },
  );
```

対象年月に
資産口座データが存在しない場合、
0円とみなすか
データなしとみなすかは、
画面要件に従う。

APIレスポンスを
勝手に0円データへ変換しない。

---

#### 2.21 保有商品別推移

商品単位の資産口座について、
`holdingAssets`から
保有商品別推移を
表示できる。

概念例：

```ts
const holdingTrend =
  data.trends.map(
    (trend) => {
      const holding =
        trend.assetAccounts
          .flatMap(
            (account) =>
              account.holdingAssets,
          )
          .find(
            (item) =>
              item.holdingAssetId
              === targetHoldingAssetId,
          );

      return {
        targetYearMonth:
          trend.targetYearMonth,

        assetAmount:
          holding?.assetAmount
          ?? null,
      };
    },
  );
```

---

#### 2.22 金額表示

金額は、
日本円のintegerとして
APIから返却される。

表示時は、
共通の金額フォーマッタを
使用する。

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

差分項目は
負数も許可する。

---

#### 2.23 IDの扱い

以下のIDは、
API共通方針に従って
stringとして扱う。

```text
assetAccountId
holdingAssetId
```

numberへ変換して
業務計算には使用しない。

---

#### 2.24 trendsが0件の場合

`trends`が空配列の場合は、
APIエラーとして扱わない。

```ts
if (
  data.trends.length === 0
) {
  // 資産推移なし表示
}
```

表示例：

```text
指定期間に表示できる
確定済み資産データがありません。
```

必要に応じて、
月末資産管理画面への
導線を表示する。

---

#### 2.25 ローディング表示

AST-003取得中は、
ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return <Loading />;
}
```

未取得状態と
`trends = []`を
区別する。

```text
未取得
≠
取得成功0件
```

とする。

---

#### 2.26 VALIDATION_ERROR

以下の場合は、
`VALIDATION_ERROR`
として扱う。

- `from`未指定
- `to`未指定
- `from`形式不正
- `to`形式不正
- `from > to`

通常の画面操作では
発生しないよう、
フロントエンドでも
入力制御を行う。

ただし、
URL直接操作などに備え、
APIエラー処理も実装する。

---

#### 2.27 USER_NOT_FOUND

`USER_NOT_FOUND`
が返却された場合は、
操作対象利用者を
現在利用できない状態として扱う。

利用者選択画面へ戻すなど、
API共通方針に従って処理する。

---

#### 2.28 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 期間指定の入力エラーとして扱う |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

指定期間内0件は、
エラー表示の対象としない。

---

#### 2.29 自動リトライ

AST-003は、
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
再送しても解消しない
エラーについては、
不要なリトライを行わない。

---

#### 2.30 クライアントキャッシュ

AST-003は、
React Query等による
クライアントキャッシュの
対象としてよい。

`from`および`to`ごとに
キャッシュを分離する。

例えば、

```text
2026-01〜2026-06
```

と、

```text
2026-01〜2026-12
```

は、
別のQuery Keyとして扱う。

---

#### 2.31 SNP-004との連携

SNP-004 月末資産状況確定APIが
成功した場合は、
AST-003の対象データが
増加する可能性がある。

例えば、

```text
確定前

2026-01 true
2026-02 false
2026-03 true
```

から、

```text
確定後

2026-01 true
2026-02 true
2026-03 true
```

となった場合、
AST-003を再取得する。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey:
    assetTrendKeys.all,
});
```

---

#### 2.32 SNP-005との連携

SNP-005 月末資産状況確定解除APIが
成功した場合も、
AST-003の対象年月が
減少する可能性がある。

そのため、
資産推移Queryを
invalidateする。

確定解除された年月は、
再取得後の`trends`から
除外される。

---

#### 2.33 前月差分の再計算

ある年月が
確定または確定解除されると、
隣接する年月の
前月差分も変化する可能性がある。

例えば、

```text
2026-01 true
2026-02 false
2026-03 true
```

では、

```text
2026-03
difference = null
```

となる。

その後、
`2026-02`が確定すると、

```text
2026-03
difference
=
2026-03
-
2026-02
```

となる。

そのため、
SNP-004やSNP-005成功後は、
対象月だけではなく
資産推移Query全体を
再取得する。

---

#### 2.34 BAL・VAL系APIとの連携

BAL系およびVAL系APIによって
未確定月の資産データが
変更されても、
その月が未確定である限り、
AST-003には反映されない。

```text
BAL / VAL更新
    ↓
snapshot未確定
    ↓
AST-003対象外

SNP-004で確定
    ↓
AST-003対象
```

フロントエンドでも、
この業務ルールを前提とする。

---

#### 2.35 AST-002との連携

資産推移グラフから
特定年月を選択した場合は、
AST-002 指定年月資産状況取得APIを
使用して、
その年月の資産状況を
詳細表示できる。

概念例：

```tsx
const handleTrendClick =
  (
    targetYearMonth: string,
  ) => {
    navigate(
      `/assets/${targetYearMonth}`,
    );
  };
```

遷移先では、
AST-002を使用する。

---

#### 2.36 AST-001との使い分け

最新の資産状況のみを
表示したい場合は、
AST-001を使用する。

複数月の資産推移を
表示する場合は、
AST-003を使用する。

```text
最新月の詳細
    → AST-001

特定月の詳細
    → AST-002

複数月の推移
    → AST-003
```

AST-001またはAST-002を
複数回呼び出して
資産推移を生成しない。

---

#### 2.37 フロントエンドで前月差分を再計算しない

APIレスポンスには、

```text
totalAssetsDifference
availableAssetsDifference
```

が含まれる。

フロントエンド側で、

```ts
current.totalAssets
-
previous.totalAssets
```

のように再計算して、
API結果を置き換えない。

特に、
欠損月がある場合に
配列上の直前要素と比較すると、
API仕様と異なる結果になる可能性がある。

バックエンドから返却された
差分値を正式値として扱う。

---

#### 2.38 フロントエンドで資産推移を再集計しない

以下について、
APIレスポンスを正式値として扱う。

- `totalAssets`
- `availableAssets`
- `totalAssetsDifference`
- `availableAssetsDifference`
- `assetAccounts[].assetAmount`
- `holdingAssets[].assetAmount`

React側で
月末資産データを別途取得して、
独自の資産推移を
再構成しない。

---

### 3 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [AST-001 現在資産状況取得](./ast-001-list.md)
- [AST-002 指定年月資産状況取得](./ast-002-create.md)
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

