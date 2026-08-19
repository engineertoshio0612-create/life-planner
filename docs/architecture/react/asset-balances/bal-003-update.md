# BAL-003 月末資産残高更新

## 1. 概要

本ドキュメントでは、
BAL-003 月末資産残高更新APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

BAL-003では、
操作対象となる利用者について、
指定した月末資産状況に紐づく
登録済みの月末資産残高を更新する。

React・TypeScriptでは、
主に月末資産状況の入力画面から、
既存の月末資産残高を編集し、
変更後の残高をBAL-003へ送信する。

本APIの更新対象は、
残高記録単位が資産口座単位であり、
対象の月末資産状況に対して
月末資産残高がすでに登録されている
資産口座のみとする。

月末資産残高が未登録の場合は、
BAL-003による更新は行わず、
BAL-002 月末資産残高登録APIを使用する。

また、商品単位で残高を記録する
資産口座についてはBAL-003を使用せず、
商品別月末評価額を変更する場合は
VAL-003 商品別月末評価額更新APIを使用する。

フロントエンドでは、
BAL-001 月末資産残高一覧取得APIなどから取得した
月末資産残高の登録状態をもとに、
登録APIと更新APIを切り替える。

```text
balance === null
    → BAL-002 月末資産残高登録

balance !== null
    → BAL-003 月末資産残高更新
```

`balance = 0`は、
0円として登録済みであることを表すため、
未登録として扱わずBAL-003の更新対象とする。

概念的な利用構成は、
以下とする。

```text
BAL-001 月末資産残高一覧取得
    ↓
登録済みの月末資産残高を表示
    ↓
利用者が残高を変更
    ↓
フロントエンドバリデーション
    ↓
BAL-003 月末資産残高更新
    ↓
更新成功
    ↓
BAL-001 または SNP-003 を再取得
    ↓
画面を最新状態へ更新
```

フロントエンドでは、
入力値に対する基本的なバリデーションや
更新処理中の二重送信防止、
月末資産状況の確定状態に応じた
入力欄の活性・非活性制御などを行ってよい。

ただし、

* 月末資産状況の存在確認
* 月末資産状況の確定状態確認
* 資産口座の存在確認
* 対象年月時点の資産口座の利用可否判定
* 残高記録単位の判定
* 月末資産残高の存在確認

などの業務ルールについては、
フロントエンドだけで保証しない。

最終的な更新可否は、
バックエンドの判定結果に従う。

更新成功後は、
BAL-003のレスポンスだけで
画面全体の状態を管理しようとせず、
必要に応じてBAL-001 月末資産残高一覧取得APIや
SNP-003 月末資産状況詳細取得APIを再取得し、
バックエンドの最新状態と画面表示を同期する。

---

## 2. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type GetMonthEndHoldingValuesParams = {
  snapshotId: string;
};
```

一覧項目の型は、
以下とする。

```ts
export type MonthEndHoldingValueListItem = {
  holdingAssetId: string;
  assetAccountId: string;
  holdingAssetName: string;
  value: number | null;
};
```

レスポンス型は、
以下とする。

```ts
export type GetMonthEndHoldingValuesResponse = {
  data: MonthEndHoldingValueListItem[];
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetMonthEndHoldingValuesResponse>(
    `/api/v1/month-end-asset-snapshots/${snapshotId}/holding-values`,
  );
```

取得した一覧は、
商品別月末評価額一覧画面や
商品別月末評価額入力画面で利用する。

---

### 2.1 snapshotIdの扱い

`snapshotId`は、
API共通方針に従って
文字列として扱う。

```ts
const snapshotId: string = '12';
```

フロントエンド側で
数値へ変換して
業務計算には使用しない。

URL生成時も、
文字列のまま使用する。

---

### 2.2 holdingAssetIdの扱い

`holdingAssetId`は、
文字列として扱う。

```ts
const holdingAssetId: string = '5';
```

商品別月末評価額の
登録・更新対象を特定する際に使用する。

フロントエンド側で
数値へ変換する必要はない。

---

### 2.3 assetAccountIdの扱い

`assetAccountId`は、
保有商品が属する
資産口座を識別するために使用する。

```ts
const assetAccountId: string = '3';
```

商品別月末評価額一覧を
資産口座単位でグループ表示する場合などに
利用してよい。

ただし、
フロントエンド側で
利用者境界や
残高記録単位の最終判定には使用しない。

---

### 2.4 valueの扱い

`value`は、
以下の状態を表す。

```text
number
    → 商品別月末評価額登録済み

0
    → 0円として登録済み

null
    → 未登録
```

TypeScriptでは、
以下のように明示的に判定する。

```ts
if (item.value === null) {
  // 未登録
} else {
  // 登録済み
}
```

以下のような
truthy / falsyによる判定は行わない。

```ts
if (!item.value) {
  // value = 0 も未登録扱いになるため使用しない
}
```

---

### 2.5 金額表示

登録済みの`value`は、
日本円の整数値として表示する。

```ts
const formatCurrency = (
  value: number,
): string =>
  new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY',
    maximumFractionDigits: 0,
  }).format(value);
```

使用例：

```ts
const displayValue =
  item.value === null
    ? '未登録'
    : formatCurrency(item.value);
```

`value = null`を
0円として表示してはならない。

---

### 2.6 未登録状態の表示

`value = null`の場合は、
商品別月末評価額が
未登録であることを表示する。

表示例：

```text
未登録
```

必要に応じて、
VAL-002 商品別月末評価額登録APIを
利用する入力画面への導線を表示する。

---

### 2.7 0円の表示

`value = 0`の場合は、
登録済みの0円として表示する。

表示例：

```text
￥0
```

未登録とは扱わない。

そのため、

```text
value === null
```

と

```text
value === 0
```

を必ず区別する。

---

### 2.8 登録・更新APIの切り替え

VAL-001のレスポンスを使用して、
商品別月末評価額が
登録済みか未登録かを判定できる。

```ts
if (item.value === null) {
  // VAL-002 商品別月末評価額登録
} else {
  // VAL-003 商品別月末評価額更新
}
```

`value = 0`の場合は、
登録済みであるため
VAL-003を使用する。

以下の判定は行わない。

```ts
if (!item.value) {
  // 0円を未登録と誤判定するため使用しない
}
```

---

### 2.9 月末資産状況の確定状態

VAL-001のレスポンスには、
月末資産状況の`confirmed`は
含まれない。

編集可否を
画面上で補助的に制御する場合は、
SNP-003 月末資産状況詳細取得APIから
確定状態を取得する。

```text
SNP-003
    → confirmed取得

VAL-001
    → 商品別月末評価額一覧取得
```

例えば、

```ts
const canEdit =
  !snapshot.confirmed;
```

として、
確定済みの場合は
入力欄を非活性化してよい。

ただし、
登録・更新可否の最終判断は
VAL-002およびVAL-003で行う。

---

### 2.10 一覧のグループ表示

レスポンスには
`assetAccountId`が含まれるため、
資産口座単位で
商品別月末評価額を
グループ表示してよい。

例えば、

```text
証券口座A
    全世界株式       ￥850,000
    S&P500           未登録

証券口座B
    国内株式         ￥300,000
```

のように表示できる。

ただし、
VAL-001では
資産口座名を返却しない。

資産口座名が必要な場合は、
資産口座一覧取得API等で取得した
フロントエンド側のデータと
`assetAccountId`で対応付ける。

---

### 2.11 一覧順の扱い

APIレスポンスは、
以下の順序で返却される。

```text
assetAccountId相当の順序
    ↓
holdingAssetId相当の順序
```

具体的には、
バックエンドで

```text
asset_accounts.id ASC
holding_assets.id ASC
```

を適用する。

フロントエンドでは、
原則として
APIが返却した順序を
そのまま利用する。

同一の並び順ロジックを
フロントエンドへ
重複実装しない。

---

### 2.12 データなしの扱い

一覧対象となる
保有商品が存在しない場合は、
以下が返却される。

```json
{
  "data": []
}
```

フロントエンドでは、
エラーとして扱わない。

表示例：

```text
この対象年月には、
商品別評価額を記録する
保有商品がありません。
```

---

### 2.13 ローディング表示

一覧取得中は、
ローディング状態を表示する。

例えば、
React Query等を使用する場合は、
取得状態に応じて表示を切り替える。

```ts
if (isLoading) {
  return <Loading />;
}
```

`snapshotId`が変更された場合は、
以前の対象年月の一覧を
現在のデータとして
誤って表示しないようにする。

---

### 2.14 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 不正なURLまたはパラメータとして扱う |
| `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` | 月末資産状況一覧画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

商品別月末評価額が
未登録であること自体は、
エラーとして扱わない。

---

### 2.15 再取得

以下の操作後は、
必要に応じて
VAL-001を再取得する。

- VAL-002 商品別月末評価額登録
- VAL-003 商品別月末評価額更新

React Query等を使用する場合は、
対象の`snapshotId`に対応する
商品別月末評価額一覧クエリを
invalidateしてよい。

例：

```ts
queryClient.invalidateQueries({
  queryKey: [
    'monthEndHoldingValues',
    snapshotId,
  ],
});
```

これにより、
登録・更新後の最新状態を
一覧へ反映する。

---

## 3. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)
