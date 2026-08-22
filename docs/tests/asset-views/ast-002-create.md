##  AST-002 指定年月資産状況取得

### 1 概要

本ドキュメントでは、AST-002 指定年月資産状況取得APIに対するテスト方針および主要なテスト観点を定義する。

AST-002は、操作対象利用者について、パスパラメータ`targetYearMonth`で指定された年月の確定済み月末資産状況を取得し、その年月における資産状況を算出する読み取り専用APIである。

テストでは、主に以下が仕様どおり動作することを確認する。

* 指定した対象年月の確定済み月末資産状況のみを取得すること
* 最新年月へ自動的に置き換えないこと
* 指定年月が不存在または未確定の場合に別年月へフォールバックしないこと
* `targetYearMonth`の形式および年月としての妥当性を検証すること
* 資産口座の残高記録単位に応じて正しい資産データを使用すること
* 月末資産残高と商品別月末評価額を二重計上しないこと
* 総資産および利用可能資産が正しく算出されること
* 指定年月時点の利用可能資産設定が使用されること
* 他利用者のデータが混入しないこと
* API実行による副作用が発生しないこと

```text id="u5x2ma"
操作対象利用者
    +
targetYearMonth
    ↓
指定年月の
confirmed = true
month_end_asset_snapshot を特定
    ↓
対象年月の資産データを取得
    ↓
残高記録単位に応じて資産額を算出
    ↓
指定年月時点の利用可能資産を算出
    ↓
指定年月の資産状況として返却
```

特に、AST-002では指定された年月をそのまま取得条件として扱い、より新しい確定済み年月や前後の確定済み年月へ自動的に切り替わらないことを重要なテスト観点とする。

資産額については、口座単位の資産口座では`month_end_asset_balances.balance`、商品単位の資産口座では`month_end_holding_values.value`を使用し、双方のデータが存在する場合でも残高記録単位に従って一方のみを集計することを確認する。

また、0円は正常な業務値として扱い、未登録やデータ不存在と区別する。指定年月の確定済み月末資産状況が存在しない場合は、0円の資産状況や別年月のデータを返却せず、`CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND`となることを確認する。

利用可能資産については、現在時点の設定ではなく、指定した`targetYearMonth`時点の`asset_account_available_settings`が使用されることを検証する。

利用者境界については、月末資産状況、月末資産残高、商品別月末評価額および利用可能資産設定のすべてで、操作対象利用者以外のデータが取得・集計されないことを確認する。

さらに、AST-001の最新確定済み対象年月をAST-002へ指定した場合、同一データ状態で両APIの資産状況が一致することを確認する。

異常系では、対象年月形式不正、利用者コンテキスト未指定、利用者ID形式不正、利用者不存在および指定年月の確定済み月末資産状況不存在について、期待するHTTPステータスとエラーコードが返却されることを検証する。

最後に、GET APIとしてデータベースを更新しないこと、同一データ状態で繰り返し実行しても同一結果となること、および正常・異常レスポンスがAPI共通方針で定めた契約に従うことを確認する。


---

### 2 テスト観点

#### 2.1 正常系

操作対象利用者について、
指定年月の
確定済み月末資産状況および
必要な資産データが存在する状態で
AST-002を実行する。

以下を確認する。

- `200 OK`となること
- 指定した`targetYearMonth`が返却されること
- 総資産が正しく算出されること
- 利用可能資産が正しく算出されること
- 資産口座別資産状況が返却されること
- 商品単位の資産口座について保有商品別資産状況が返却されること
- データベースが更新されないこと

---

#### 2.2 最新ではない確定済み年月

以下の状態を用意する。

```text
2026-05
confirmed = true

2026-06
confirmed = true

2026-07
confirmed = true
```

以下を実行する。

```http
GET /api/v1/asset-summaries/2026-05
```

期待結果：

```text
targetYearMonth = 2026-05
```

となること。

最新の`2026-07`へ
置き換わらないこと。

---

#### 2.3 指定年月が未確定

以下の状態を用意する。

```text
2026-06
confirmed = true

2026-07
confirmed = false
```

以下を実行する。

```http
GET /api/v1/asset-summaries/2026-07
```

期待結果：

```text
404 Not Found
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

未確定の`2026-07`を
資産状況へ使用しないこと。

---

#### 2.4 指定年月が存在しない

以下の状態を用意する。

```text
2026-05
confirmed = true

2026-07
confirmed = true
```

以下を実行する。

```http
GET /api/v1/asset-summaries/2026-06
```

期待結果：

```text
404 Not Found
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

---

#### 2.5 別年月へフォールバックしないこと

以下の状態を用意する。

```text
2026-05
confirmed = true

2026-06
データなし

2026-07
confirmed = true
```

`2026-06`を指定する。

以下を確認する。

- `2026-05`を返却しないこと
- `2026-07`を返却しないこと
- `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND`となること

---

#### 2.6 targetYearMonth正常値

以下の値を指定する。

```text
2026-01
2026-06
2026-12
```

形式バリデーションを通過すること。

---

#### 2.7 targetYearMonth形式不正

以下の値を指定する。

```text
2026-1
26-05
2026/05
202605
abc
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 2.8 targetYearMonthの月が0

```text
2026-00
```

を指定する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 2.9 targetYearMonthの月が13

```text
2026-13
```

を指定する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 2.10 口座単位の資産口座

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

#### 2.11 商品単位の資産口座

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

#### 2.12 商品単位の複数保有商品

商品単位の資産口座に
複数の商品別月末評価額を用意する。

以下を確認する。

- すべての`value`が資産口座単位で合計されること
- 各保有商品の`assetAmount`に個別の`value`が返却されること
- 資産口座の`assetAmount`と保有商品別合計が一致すること

---

#### 2.13 二重計上しないこと

商品単位の資産口座について、
同じ指定年月に

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

---

#### 2.14 口座単位で商品別評価額を加算しないこと

口座単位の資産口座について、
同じ指定年月に
商品別月末評価額が存在する状態を用意する。

以下を確認する。

- `month_end_asset_balances.balance`のみを使用すること
- 商品別月末評価額を`totalAssets`へ加算しないこと

---

#### 2.15 複数資産口座の総資産

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

#### 2.16 totalAssetsと資産口座別合計

正常なデータについて、

```text
totalAssets
=
SUM(assetAccounts.assetAmount)
```

となることを確認する。

---

#### 2.17 0円の月末資産残高

口座単位の資産口座について、

```text
balance = 0
```

を用意する。

以下を確認する。

- 未登録扱いにならないこと
- `assetAmount = 0`として返却されること
- 0円として正常に集計されること

---

#### 2.18 0円の商品別月末評価額

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

#### 2.19 指定年月の利用可能資産

指定年月について、
以下の状態を用意する。

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

#### 2.20 過去月の利用可能資産設定

同一資産口座について、
以下の設定を用意する。

```text
2026-05
available = true

2026-06
available = false
```

`2026-05`を指定する。

期待結果：

```text
available = true
```

となること。

現在または
`2026-06`の設定を
誤って使用しないこと。

---

#### 2.21 すべて利用可能

指定年月時点で、
すべての資産口座が
利用可能資産として扱われる状態とする。

期待結果：

```text
availableAssets
=
totalAssets
```

となること。

---

#### 2.22 利用可能資産が0円

指定年月時点で、
すべての資産口座が
利用可能資産として
扱われない状態とする。

期待結果：

```text
availableAssets = 0
```

となること。

---

#### 2.23 口座単位のholdingAssets

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

#### 2.24 商品単位のholdingAssets

商品単位の資産口座について、
指定年月に紐づく
保有商品別資産状況が
返却されることを確認する。

各項目について、

- `holdingAssetId`
- `name`
- `assetAmount`

が正しく返却されること。

---

#### 2.25 他利用者の同一年月を使用しないこと

以下の状態を用意する。

```text
User A
2026-05
データなし

User B
2026-05
confirmed = true
```

User Aとして
`2026-05`を指定する。

期待結果：

```text
404 Not Found
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

となること。

User Bの資産状況を
返却しないこと。

---

#### 2.26 他利用者の月末資産残高を含めないこと

User AとUser Bについて、
同一対象年月に
月末資産残高を用意する。

User Aとして
AST-002を実行する。

以下を確認する。

- User Aの月末資産残高のみ集計されること
- User Bの残高が`totalAssets`へ加算されないこと

---

#### 2.27 他利用者の商品別月末評価額を含めないこと

User AとUser Bについて、
同一対象年月に
商品別月末評価額を用意する。

User Aとして
AST-002を実行する。

以下を確認する。

- User Aの商品別月末評価額のみ集計されること
- User Bの商品別月末評価額が混入しないこと

---

#### 2.28 他利用者の利用可能資産設定を使用しないこと

User AとUser Bについて、
同一対象年月に
利用可能資産設定を用意する。

User Aとして
AST-002を実行する。

以下を確認する。

- User Aに属する資産口座の設定のみ使用されること
- User Bの設定によって`availableAssets`が変化しないこと

---

#### 2.29 利用者コンテキスト未指定

`X-User-Id`を
指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

---

#### 2.30 利用者ID形式不正

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

#### 2.31 利用者不存在

存在しない利用者IDを
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 2.32 論理削除済み利用者

論理削除済み利用者を
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 2.33 AST-001との取得結果比較

AST-001の
最新確定済み対象年月が
`2026-06`である状態とする。

AST-001を実行し、
取得した資産状況を保持する。

続いて、

```http
GET /api/v1/asset-summaries/2026-06
```

を実行する。

同一時点のデータ状態であれば、
以下が一致することを確認する。

- `targetYearMonth`
- `totalAssets`
- `availableAssets`
- 資産口座別資産状況
- 保有商品別資産状況

---

#### 2.34 副作用

AST-002実行前後で、
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

- 月末資産状況の`confirmed`が変更されないこと
- 集計結果をデータベースへ保存しないこと
- 利用可能資産設定が変更されないこと

---

#### 2.35 冪等性

同一のデータ状態で
同じリクエストを
複数回実行する。

```text
1回目
GET /api/v1/asset-summaries/2026-05
    ↓
200 OK

2回目
GET /api/v1/asset-summaries/2026-05
    ↓
200 OK
```

以下を確認する。

- データベースが変更されないこと
- 同一状態であれば同じ`targetYearMonth`となること
- 同一状態であれば同じ集計結果となること

---

#### 2.36 レスポンス契約

正常取得時に、
API共通方針で定めた
Envelope形式で返却されること。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-05",
    "totalAssets": 3200000,
    "availableAssets": 1400000,
    "assetAccounts": [
      {
        "assetAccountId": "1",
        "name": "普通預金",
        "assetAmount": 900000,
        "available": true,
        "holdingAssets": []
      }
    ]
  }
}
```

以下を確認する。

- `data`がobjectであること
- `targetYearMonth`がリクエスト指定値と一致すること
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

#### 2.37 返却しない情報

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

#### 2.38 エラーレスポンス

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

- [AST-002 API詳細設計](../../api/details/asset-views/ast-002-create.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [AST-001 現在資産状況取得 テスト設計](./ast-001-list.md)
- [AST-003 資産推移取得 テスト設計](./ast-003-update.md)
- [AST-002 Laravelアーキテクチャ設計](../../architecture/laravel/asset-views/ast-002-create.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [AST-002 Reactアーキテクチャ設計](../../architecture/react/asset-views/ast-002-create.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
