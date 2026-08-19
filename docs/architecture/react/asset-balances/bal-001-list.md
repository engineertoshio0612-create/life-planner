# BAL-001 月末資産残高一覧取得

## 1. 概要

本ドキュメントでは、
BAL-001 月末資産残高一覧取得APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

BAL-001では、
操作対象となる利用者について、
指定した月末資産状況に紐づく
月末資産残高の一覧を取得する。

取得対象には、
月末資産残高が登録済みの資産口座だけでなく、
対象年月時点で残高記録対象であり、
月末資産残高が未登録の資産口座も含まれる。

React・TypeScriptでは、
取得した一覧をもとに、

* 対象年月における残高記録対象の資産口座
* 月末資産残高の登録状態
* 登録済みの月末資産残高

を画面へ表示する。

特に、`balance`については、
`null`を未登録、
`0`を0円で登録済みとして扱い、
両者を明確に区別する。

また、BAL-001のレスポンスには
月末資産状況の確定状態は含まれないため、
月末資産残高の登録・更新可否を
フロントエンドで制御する場合は、
SNP-003 月末資産状況詳細取得APIから取得した
`confirmed`と組み合わせて判断する。

概念的な利用構成は、
以下とする。

```text
月末資産状況画面
    ↓
snapshotIdを取得
    ↓
SNP-003 月末資産状況詳細取得
    │
    └─ confirmedを取得
    ↓
BAL-001 月末資産残高一覧取得
    ↓
資産口座ごとの
残高登録状態・残高を取得
    ↓
月末資産残高一覧を表示
    ↓
未登録
    → 登録画面への導線

登録済み
    → 更新画面への導線

確定済み
    → 編集操作を制御
```

フロントエンドでは、
APIが返却する登録状態や並び順を尊重し、
バックエンドですでに定義されている
業務ルールを重複して実装しない。

また、月末資産残高の登録・更新後は、
BAL-001を必要に応じて再取得し、
最新の月末資産残高を画面へ反映する。

---

## 2. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type GetMonthEndAssetBalancesParams = {
  snapshotId: string;
};
```

一覧項目の型は、
以下とする。

```ts
export type MonthEndAssetBalanceListItem = {
  assetAccountId: string;
  assetAccountName: string;
  balance: number | null;
};
```

レスポンス型は、
以下とする。

```ts
export type GetMonthEndAssetBalancesResponse = {
  data: MonthEndAssetBalanceListItem[];
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetMonthEndAssetBalancesResponse>(
    `/api/v1/month-end-asset-snapshots/${snapshotId}/asset-balances`,
  );
```

取得した一覧は、
月末資産残高一覧画面や
月末資産入力画面で利用する。

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

### 2.2 balanceの扱い

`balance`は、
登録状態によって
以下のいずれかとなる。

```text
整数
    → 登録済み

0
    → 0円で登録済み

null
    → 未登録
```

TypeScriptでは、
以下のように扱う。

```ts
if (item.balance === null) {
  // 未登録
} else {
  // 登録済み
}
```

以下のような
truthy / falsyによる判定は行わない。

```ts
if (!item.balance) {
  // balance = 0 も未登録扱いになるため使用しない
}
```

`0`と`null`は
明確に区別する。

---

### 2.3 金額表示

登録済みの`balance`は、
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
const displayBalance =
  item.balance === null
    ? '未登録'
    : formatCurrency(item.balance);
```

フロントエンドで
小数への変換や
独自の端数処理は行わない。

---

### 2.4 未登録状態の表示

`balance = null`の場合は、
入力漏れまたは
未入力であることを
利用者が判断できるように表示する。

表示例：

```text
未登録
```

必要に応じて、
月末資産残高登録画面への
導線を表示する。

`null`を
`0円`として表示してはならない。

---

### 2.5 0円の表示

`balance = 0`の場合は、
登録済みの0円として表示する。

表示例：

```text
￥0
```

`0円`を
未登録として表示してはならない。

---

### 2.6 一覧順の扱い

APIレスポンスは、
以下の順序で返却される。

1. 資産口座名昇順
2. 資産口座ID昇順

フロントエンドでは、
原則として
APIが返却した順序を
そのまま表示する。

同じ並び順を
フロントエンドへ
重複実装しない。

---

### 2.7 データなしの扱い

対象年月時点で
残高記録単位が口座単位の
資産口座が存在しない場合は、
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
口座単位で残高を記録する
資産口座がありません。
```

---

### 2.8 月末資産状況の確定状態

本APIのレスポンスには、
月末資産状況の`confirmed`は
含まれない。

編集可否を判断する必要がある場合は、
SNP-003 月末資産状況詳細取得APIで
確定状態を取得する。

例えば、
以下のように
複数APIの結果を組み合わせる。

```text
SNP-003
    → confirmed取得

BAL-001
    → 月末資産残高一覧取得
```

フロントエンドでは、
BAL-001のレスポンスだけを使用して
更新可否を判断しない。

---

### 2.9 登録・更新画面への遷移

`balance = null`の場合は、
月末資産残高登録APIを使用する画面へ
遷移できる。

`balance !== null`の場合は、
月末資産残高更新APIを使用する画面へ
遷移できる。

ただし、
月末資産状況が確定済みの場合は、
バックエンド側で更新が拒否される。

そのため、
フロントエンドの表示制御は
補助的なものとし、
更新API側でも
必ず業務ルールを検証する。

---

### 2.10 ローディング表示

一覧取得中は、
ローディング状態を表示する。

`snapshotId`が変更された場合は、
以前の月末資産残高一覧を
現在の対象年月の情報として
表示しない。

新しい取得が完了するまで、
以前の一覧を破棄するか
ローディング表示へ切り替える。

---

### 2.11 エラー表示

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

月末資産残高が
未登録であること自体は、
エラーとして扱わない。

---

### 2.12 再取得

以下の操作後は、
必要に応じて
BAL-001を再取得する。

- 月末資産残高登録
- 月末資産残高更新

React Query等を使用する場合は、
対象となる
月末資産残高一覧クエリを
invalidateしてよい。

これにより、
登録・更新後の最新残高を
画面へ反映する。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [BAL-002 月末資産残高登録](./bal-002-create.md)
- [BAL-003 月末資産残高更新](./bal-003-update.md)
- [SNP-003 月末資産状況詳細取得](../month-end-asset-snapshots/snp-003-detail.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/asset-balances.md)
- [Reactアーキテクチャ設計](../../../architecture/react/asset-balances.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)
