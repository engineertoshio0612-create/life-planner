# ACC-002 資産口座登録

## 1. 概要

本ドキュメントでは、ACC-002 資産口座登録APIに対するテスト観点を定義する。

操作対象利用者に帰属する資産口座と、その初期利用可能資産設定が正しく登録されることに加え、入力値、利用者境界、同名資産口座の重複判定およびレスポンス契約が設計どおりに機能することを確認する。

また、`asset_accounts`と`asset_account_available_settings`の登録を単一トランザクションとして扱い、一部のデータだけが残らないことや、同一利用者による同名資産口座の同時登録時にもデータの一意性と整合性が維持されることを検証対象とする。

---



## 2. テスト観点

ACC-002では、資産口座登録だけでなく、以下を重点的に確認する。

* 利用者境界
* 入力値
* 同名重複
* 初期利用可能資産設定
* トランザクション
* 同時登録

---

### 2.1 正常系

以下のような正常なリクエストを送信する。

```json
{
  "name": "証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

以下を確認する。

* `201 Created`となること
* `asset_accounts`が1件登録されること
* `asset_account_available_settings`が1件登録されること
* 両レコードが正しく関連していること
* 正常レスポンスが返却されること

---

### 2.2 user_id

新規登録された`asset_accounts.user_id`が、

```text
X-User-Id
```

から特定した操作対象利用者IDと一致することを確認する。

リクエストボディから`user_id`を設定していないことも確認する。

---

### 2.3 name

正常な資産口座名を指定し、そのまま

```text
asset_accounts.name
```

へ保存されることを確認する。

以下も確認する。

* `name`未指定
* `name = null`
* 空文字
* 最大文字数以内
* 最大文字数超過

不正な場合は、資産口座および初期利用可能資産設定が登録されないことを確認する。

---

### 2.4 assetType

以下の正常値をそれぞれ確認する。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

APIの文字列値が正しいDB保存形式へ変換されることを確認する。

以下のような未定義値も確認する。

```text
CRYPTO
UNKNOWN
```

不正な場合は、登録されないこと。

---

### 2.5 balanceRecordingUnit

以下の正常値を確認する。

```text
ACCOUNT
HOLDING
```

API用の文字列が正しいDB保存形式へ変換されることを確認する。

以下のような不正値も確認する。

```text
UNKNOWN
ASSET
```

不正時に業務データが登録されないこと。

---

### 2.6 ACCOUNT

```text
balanceRecordingUnit = ACCOUNT
```

で資産口座を正常登録できることを確認する。

また、ACC-002実行によって

```text
month_end_asset_balances
```

が自動登録されないことを確認する。

---

### 2.7 HOLDING

```text
balanceRecordingUnit = HOLDING
```

で資産口座を正常登録できることを確認する。

また、ACC-002実行によって

```text
holding_assets
```

が自動登録されないことを確認する。

保有商品登録はHLD-002の責務であることを確認する。

---

### 2.8 startYearMonth

正常値として、例えば以下を確認する。

```text
2026-01
2026-08
2027-12
```

以下の不正値も確認する。

```text
2026-1
2026/08
202608
2026-00
2026-13
abc
```

不正な場合は、`asset_accounts`および`asset_account_available_settings`が登録されないこと。

---

### 2.9 isAvailable = true

```json
{
  "isAvailable": true
}
```

を指定する。

初期利用可能資産設定の

```text
is_available = true
```

となることを確認する。

---

### 2.10 isAvailable = false

```json
{
  "isAvailable": false
}
```

を指定する。

初期利用可能資産設定の

```text
is_available = false
```

となることを確認する。

`false`を未指定扱いしないことも確認する。

---

### 2.11 isAvailableの型

以下を不正値として確認する。

```json
{
  "isAvailable": "true"
}
```

```json
{
  "isAvailable": 1
}
```

```json
{
  "isAvailable": null
}
```

boolean以外を受け付けないこと。

---

### 2.12 初期利用可能資産設定の開始年月

資産口座の

```text
start_year_month
```

と、初期利用可能資産設定の

```text
start_year_month
```

が一致することを確認する。

例えば、

```text
asset_accounts.start_year_month
    = 2026-08

asset_account_available_settings.start_year_month
    = 2026-08
```

となること。

---

### 2.13 初期利用可能資産設定の終了年月

ACC-002で登録した初期設定の

```text
end_year_month
```

が

```text
NULL
```

となることを確認する。

---

### 2.14 asset_account_id

初期利用可能資産設定の

```text
asset_account_id
```

が、同じACC-002で新規登録した

```text
asset_accounts.id
```

と一致することを確認する。

他資産口座へ紐づかないこと。

---

### 2.15 登録時の利用状態

新規資産口座の

```text
deleted_at
```

が

```text
NULL
```

となることを確認する。

正常レスポンスの

```text
isEnabled
```

が

```text
true
```

となることも確認する。

---

### 2.16 同一利用者の同名資産口座

操作対象利用者に

```text
証券口座
```

が存在する状態で、同じ利用者として再度同名を登録する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

以下を確認する。

* 新しい`asset_accounts`が登録されないこと
* 新しい`asset_account_available_settings`も登録されないこと

---

### 2.17 論理削除済み同名資産口座

同一利用者に

```text
name = 証券口座
deleted_at IS NOT NULL
```

の資産口座を用意する。

同名でACC-002を実行する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となること。

論理削除済みデータも重複判定対象となることを確認する。

---

### 2.18 他利用者の同名資産口座

以下の状態を用意する。

```text
User A
    証券口座

User B
    資産口座なし
```

User Bとして

```text
証券口座
```

を登録する。

正常に登録できることを確認する。

User Aの資産口座によって重複エラーにならないこと。

---

### 2.19 X-User-Id未指定

`X-User-Id`を指定しない。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

以下を確認する。

* `asset_accounts`が登録されないこと
* `asset_account_available_settings`が登録されないこと

---

### 2.20 X-User-Id形式不正

例えば、

```http
X-User-Id: abc
```

を指定する。

期待結果：

```text
400 Bad Request
INVALID_USER_ID
```

業務データが更新されないこと。

---

### 2.21 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 2.22 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

資産口座が登録されないこと。

---

### 2.23 userIdをリクエストへ含める

クライアントから以下のような`userId`を送信しても、その値を

```text
asset_accounts.user_id
```

として使用しないことを確認する。

```json
{
  "name": "証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "startYearMonth": "2026-08",
  "isAvailable": true,
  "userId": "999"
}
```

未定義項目を拒否する共通方針の場合は、バリデーションエラーとなることを確認する。

---

### 2.24 初期設定登録失敗時のロールバック

`asset_accounts`の登録後に、意図的に

```text
asset_account_available_settings
```

の登録を失敗させる。

概念的には、

```text
asset_accounts INSERT
    ↓
成功

available_settings INSERT
    ↓
失敗

ROLLBACK
```

となることを確認する。

最終的に、

```text
asset_accounts
    → 新規レコードなし

asset_account_available_settings
    → 新規レコードなし
```

となること。

---

### 2.25 資産口座登録失敗

`asset_accounts`のINSERTを失敗させる。

以下を確認する。

* `asset_account_available_settings`の登録へ進まないこと
* 中途半端な業務データが残らないこと

---

### 2.26 同時登録

同一利用者について、同じ`name`のACC-002を並行実行する。

以下を確認する。

* 同名資産口座が2件作成されないこと
* UNIQUE制約によって一意性が維持されること
* 一方のリクエストだけが成功すること
* 競合したリクエストが適切な`409 Conflict`となること
* PostgreSQLの制約エラーがそのまま返却されないこと

---

### 2.27 同時登録時の初期設定

同一名で並行登録した場合に、失敗側のリクエストによって孤立した

```text
asset_account_available_settings
```

が作成されないことを確認する。

---

### 2.28 正常レスポンス契約

正常時に、概念的に以下の形式となることを確認する。

```json
{
  "data": {
    "id": "10",
    "name": "証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

以下を確認する。

* HTTPステータスが`201 Created`であること
* `id`がstringであること
* `name`がstringであること
* `assetType`がAPI用文字列であること
* `balanceRecordingUnit`がAPI用文字列であること
* `isAvailable`がbooleanであること
* `startYearMonth`が`YYYY-MM`形式であること
* `isEnabled = true`であること

---

### 2.29 返却しない情報

正常レスポンスに、以下が含まれないことを確認する。

* `userId`
* `user_id`
* `deletedAt`
* `deleted_at`
* `createdAt`
* `created_at`
* `updatedAt`
* `updated_at`
* `assetAccountAvailableSettingId`
* `asset_account_available_settings.id`
* `asset_account_id`
* `endYearMonth`
* DB内部の数値コード

---

### 2.30 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
VALIDATION_ERROR
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
INTERNAL_SERVER_ERROR
```

以下も確認する。

* `error.code`が設定されること
* `error.message`が設定されること
* 必要に応じて`error.details`が設定されること
* `requestId`が設定されること
* SQLが含まれないこと
* PostgreSQLの制約名が含まれないこと
* スタックトレースが含まれないこと
* サーバーファイルパスが含まれないこと

---

### 2.31 副作用範囲

正常終了時に更新される業務テーブルが、

```text
asset_accounts
asset_account_available_settings
```

だけであることを確認する。

以下が変更されないことも確認する。

* `users`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

---

### 2.32 INTERNAL_SERVER_ERROR

資産口座または初期利用可能資産設定の登録処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

* トランザクションがロールバックされること
* 一部登録データが残らないこと
* SQLがレスポンスへ含まれないこと
* PostgreSQL内部エラーが公開されないこと
* スタックトレースが公開されないこと
* エラーログとレスポンスの`requestId`を関連付けられること

---

### 3 関連ドキュメント

- [ACC-002 API詳細設計](../../api/details/asset-accounts/acc-002-create.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [ACC-002 Laravelアーキテクチャ設計](../../architecture/laravel/asset-accounts/acc-002-create.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [ACC-002 Reactアーキテクチャ設計](../../architecture/react/asset-accounts/acc-002-create.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
