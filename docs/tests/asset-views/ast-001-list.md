## AST-001 現在資産状況取得

## 1. 概要

本ドキュメントでは、AST-001 現在資産状況取得APIに対するテスト方針および主要なテスト観点を定義する。

AST-001は、操作対象利用者に属する月末資産状況のうち、最新の確定済み対象年月を特定し、その年月における現在の資産状況を取得する読み取り専用APIである。

テストでは、主に以下が仕様どおり動作することを確認する。

* 最新の確定済み対象年月を正しく特定できること
* 未確定の月末資産状況を取得対象としないこと
* 資産口座の残高記録単位に応じて正しい資産データを使用すること
* 月末資産残高と商品別月末評価額を二重計上しないこと
* 総資産が正しく算出されること
* 対象年月時点の利用可能資産設定から利用可能資産が正しく算出されること
* 資産口座別および保有商品別の資産状況が正しく返却されること
* 他利用者のデータが混入しないこと
* API実行による副作用が発生しないこと

```text
操作対象利用者
    ↓
最新の confirmed = true
month_end_asset_snapshot を特定
    ↓
対象年月の資産データを取得
    ↓
残高記録単位に応じて資産額を算出
    ↓
対象年月時点の利用可能資産を算出
    ↓
現在資産状況として返却
```

資産額については、口座単位の資産口座では`month_end_asset_balances.balance`、商品単位の資産口座では`month_end_holding_values.value`を使用することを確認する。

特に、双方のデータが存在する場合でも残高記録単位に従って一方のみを集計し、同一資産が総資産へ重複して加算されないことを重要なテスト観点とする。

また、0円は正常な業務値として扱い、未登録やデータ不存在と区別する。確定済み月末資産状況自体が存在しない場合は、0円の資産状況を返却せず、`CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND`となることを確認する。

利用者境界については、月末資産状況、月末資産残高、商品別月末評価額および利用可能資産設定のすべてで、操作対象利用者以外のデータが集計結果へ混入しないことを確認する。

異常系では、利用者コンテキスト未指定、利用者ID形式不正、利用者不存在および確定済み月末資産状況不存在について、期待するHTTPステータスとエラーコードが返却されることを確認する。

さらに、GET APIとしてデータベースを更新しないこと、同一データ状態で繰り返し実行しても同じ結果となること、正常・異常レスポンスがAPI共通方針で定めた契約に従うことを検証する。

---

### 2 テスト観点

#### 2.1 正常系

操作対象利用者について、
確定済み月末資産状況および
必要な資産データが
存在する状態で
AST-001を実行する。

以下を確認する。

- `200 OK`となること
- 最新の確定済み対象年月が返却されること
- 総資産が正しく算出されること
- 利用可能資産が正しく算出されること
- 資産口座別資産状況が返却されること
- 商品単位の資産口座について保有商品別資産状況が返却されること
- データベースが更新されないこと

---

#### 2.2 最新の確定済み対象年月

以下の月末資産状況を用意する。

```text
2026-05
confirmed = true

2026-06
confirmed = true

2026-07
confirmed = true
```

期待結果：

```text
targetYearMonth = 2026-07
```

となること。

---

#### 2.3 最新年月が未確定

以下の状態を用意する。

```text
2026-05
confirmed = true

2026-06
confirmed = true

2026-07
confirmed = false
```

期待結果：

```text
targetYearMonth = 2026-06
```

となること。

未確定の`2026-07`を
現在資産状況へ使用しないこと。

---

#### 2.4 確定済み月末資産状況が存在しない

以下の状態を用意する。

```text
2026-05
confirmed = false

2026-06
confirmed = false
```

期待結果：

```text
404 Not Found
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

以下を確認する。

- `totalAssets = 0`の正常レスポンスを返却しないこと
- 未確定データを代替利用しないこと

---

#### 2.5 口座単位の資産口座

口座単位で
残高を記録する資産口座を用意する。

```text
balance_recording_unit = 口座単位

month_end_asset_balances.balance
    = 1000000
```

期待結果：

```text
assetAmount = 1000000
```

となること。

---

#### 2.6 商品単位の資産口座

商品単位で
評価額を記録する資産口座を用意する。

```text
商品A
value = 500000

商品B
value = 300000
```

期待結果：

```text
assetAmount = 800000
```

となること。

---

#### 2.7 商品単位の複数保有商品

商品単位の資産口座に
複数の商品別月末評価額を用意する。

以下を確認する。

- すべての`value`が資産口座単位で合計されること
- 各保有商品の`assetAmount`に個別の`value`が返却されること
- 資産口座の`assetAmount`と保有商品別合計が一致すること

---

#### 2.8 二重計上しないこと

商品単位の資産口座について、
同じ対象年月に

```text
month_end_asset_balances.balance
    = 900000

month_end_holding_values
    = 500000 + 300000
```

のようなデータを用意する。

期待結果：

```text
assetAmount = 800000
```

とし、

```text
1700000
```

とならないこと。

残高記録単位に従って
商品別月末評価額のみを
集計すること。

---

#### 2.9 口座単位で商品別評価額を加算しないこと

口座単位の資産口座に、
何らかの理由で
商品別月末評価額が存在する状態を用意する。

期待結果：

- `month_end_asset_balances.balance`のみを使用すること
- 商品別月末評価額を総資産へ加算しないこと

---

#### 2.10 複数資産口座の総資産

以下の資産口座を用意する。

```text
口座A
口座単位
assetAmount = 1000000

口座B
口座単位
assetAmount = 500000

口座C
商品単位
商品1 = 800000
商品2 = 700000
```

期待結果：

```text
totalAssets
=
1000000
+
500000
+
800000
+
700000
=
3000000
```

となること。

---

#### 2.11 totalAssetsと資産口座別合計

正常データについて、

```text
totalAssets
=
SUM(assetAccounts.assetAmount)
```

となることを確認する。

---

#### 2.12 0円の月末資産残高

口座単位の資産口座について、

```text
balance = 0
```

を用意する。

以下を確認する。

- 未登録扱いにならないこと
- `assetAmount = 0`として返却されること
- 総資産へ0円として正常に集計されること

---

#### 2.13 0円の商品別月末評価額

商品単位の保有商品について、

```text
value = 0
```

を用意する。

以下を確認する。

- 保有商品が一覧から除外されないこと
- `assetAmount = 0`として返却されること
- 未登録扱いにならないこと

---

#### 2.14 利用可能資産

複数の資産口座について、
対象年月時点の
利用可能資産設定を
以下のように用意する。

```text
口座A
assetAmount = 1000000
available = true

口座B
assetAmount = 2000000
available = false

口座C
assetAmount = 500000
available = true
```

期待結果：

```text
totalAssets = 3500000

availableAssets
=
1000000
+
500000
=
1500000
```

となること。

---

#### 2.15 すべて利用可能

すべての資産口座が
対象年月時点で
利用可能資産として扱われる状態とする。

期待結果：

```text
availableAssets
=
totalAssets
```

となること。

---

#### 2.16 利用可能資産が0円

すべての資産口座が
対象年月時点で
利用可能資産として
扱われない状態とする。

期待結果：

```text
availableAssets = 0
```

となること。

`0`をエラーとして扱わないこと。

---

#### 2.17 対象年月時点の利用可能資産設定

以下のように、
同じ資産口座について
対象年月によって
利用可能状態が異なる設定を用意する。

```text
2026-05
available = true

2026-06
available = false
```

AST-001の対象年月が
`2026-06`となる状態で実行する。

期待結果：

```text
available = false
```

となること。

現在の設定や
別年月の設定を
誤って使用しないこと。

---

#### 2.18 口座単位のholdingAssets

口座単位の資産口座について、
レスポンスを確認する。

期待結果：

```json
{
  "holdingAssets": []
}
```

となること。

架空の商品別内訳を
生成しないこと。

---

#### 2.19 商品単位のholdingAssets

商品単位の資産口座について、
対象年月に紐づく
保有商品別資産状況が
返却されることを確認する。

各項目について、

- `holdingAssetId`
- `name`
- `assetAmount`

が正しく返却されること。

---

#### 2.20 他利用者の最新月を使用しないこと

以下の状態を用意する。

```text
User A
2026-06
confirmed = true

User B
2026-07
confirmed = true
```

User Aを
`X-User-Id`として
AST-001を実行する。

期待結果：

```text
targetYearMonth = 2026-06
```

となること。

User Bの
`2026-07`を使用しないこと。

---

#### 2.21 他利用者の月末資産残高を含めないこと

User AとUser Bについて、
同一対象年月に
月末資産残高を用意する。

User Aとして
AST-001を実行する。

以下を確認する。

- User Aの月末資産残高のみ集計されること
- User Bの残高が`totalAssets`へ加算されないこと

---

#### 2.22 他利用者の商品別月末評価額を含めないこと

User AとUser Bについて、
商品別月末評価額を用意する。

User Aとして
AST-001を実行する。

以下を確認する。

- User Aの商品別月末評価額のみ集計されること
- User Bの商品別月末評価額が混入しないこと

---

#### 2.23 他利用者の利用可能資産設定を使用しないこと

User AとUser Bに
利用可能資産設定を用意する。

User Aとして
AST-001を実行する。

以下を確認する。

- User Aに属する資産口座の設定のみ使用されること
- User Bの設定によって`availableAssets`が変化しないこと

---

#### 2.24 利用者コンテキスト未指定

`X-User-Id`を
指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

---

#### 2.25 利用者ID形式不正

不正な
`X-User-Id`を指定する。

例：

```http
X-User-Id: abc
```

期待結果：

```text
400 Bad Request
INVALID_USER_ID
```

---

#### 2.26 利用者不存在

存在しない利用者IDを
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 2.27 論理削除済み利用者

論理削除済み利用者を
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 2.28 副作用

AST-001実行前後で、
以下のテーブルが
変更されていないことを確認する。

- `users`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `holding_assets`
- `asset_account_available_settings`

以下も確認する。

- 月末資産状況の確定状態が変更されないこと
- 集計結果をデータベースへ保存しないこと
- 利用可能資産設定が変更されないこと

---

#### 2.29 冪等性

同一のデータ状態で
同じリクエストを
複数回実行する。

```text
1回目
GET /api/v1/asset-summaries/current
    ↓
200 OK

2回目
GET /api/v1/asset-summaries/current
    ↓
200 OK
```

以下を確認する。

- データベースが変更されないこと
- 同一状態であれば同じ対象年月となること
- 同一状態であれば同じ集計結果となること

---

#### 2.30 新しい月末資産状況確定後の再取得

最初に以下の状態とする。

```text
2026-06
confirmed = true

2026-07
confirmed = false
```

AST-001を実行し、

```text
targetYearMonth = 2026-06
```

となることを確認する。

その後、
SNP-004によって
`2026-07`を確定する。

再度AST-001を実行する。

期待結果：

```text
targetYearMonth = 2026-07
```

となること。

これはAST-001の
非冪等性によるものではなく、
参照データの変更によるものであること。

---

#### 2.31 レスポンス契約

正常取得時に、
API共通方針で定めた
Envelope形式で返却されること。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-06",
    "totalAssets": 3500000,
    "availableAssets": 1500000,
    "assetAccounts": [
      {
        "assetAccountId": "1",
        "name": "普通預金",
        "assetAmount": 1000000,
        "available": true,
        "holdingAssets": []
      }
    ]
  }
}
```

以下を確認する。

- `data`がobjectであること
- `targetYearMonth`が`YYYY-MM`形式であること
- `totalAssets`がintegerであること
- `availableAssets`がintegerであること
- `assetAccounts`がarrayであること
- `assetAccountId`がstringであること
- `available`がbooleanであること
- `holdingAssets`がarrayであること
- `holdingAssetId`がstringであること
- JSONフィールド名がcamelCaseであること

---

#### 2.32 返却しない情報

本APIでは、
以下の情報が
レスポンスへ含まれていないことを確認する。

- `user_id`
- `month_end_asset_snapshot_id`
- `confirmed`
- `month_end_asset_balances.id`
- `month_end_holding_values.id`
- `balance_recording_unit`
- `asset_account_available_settings`
- `created_at`
- `updated_at`

---

#### 2.33 エラーレスポンス

各異常系について、
API共通方針で定めた
共通エラーレスポンス形式で
返却されることを確認する。

以下を確認する。

- `error.code`が期待するエラーコードであること
- `error.message`が設定されていること
- `error.details`が配列であること
- `requestId`が設定されていること
- SQLが含まれていないこと
- PostgreSQLの制約名が含まれていないこと
- スタックトレースが含まれていないこと
- 内部例外メッセージが含まれていないこと

---
### 3 関連ドキュメント

- [ACC-001 API詳細設計](../../api/details/asset-accounts/acc-001-list.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [ACC-001 Laravelアーキテクチャ設計](../../architecture/laravel/asset-accounts/acc-001-list.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [ACC-001 Reactアーキテクチャ設計](../../architecture/react/asset-accounts/acc-001-list.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
