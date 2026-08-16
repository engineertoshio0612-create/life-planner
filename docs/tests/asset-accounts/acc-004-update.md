# ACC-004 資産口座更新

## 1. 概要

本ドキュメントでは、
ACC-004 資産口座更新APIに対する
テスト方針および主要なテスト観点を定義する。

ACC-004では、
操作対象となる利用者に帰属する指定された資産口座について、
通常属性である以下の項目を更新する。

* 資産口座名
* 資産種別

一方で、残高記録単位、利用開始年月、利用可能資産区分、利用状態などは
ACC-004の更新対象としない。

テストでは、正常な更新処理だけでなく、特に以下を重点的に確認する。

* PATCHによる部分更新が正しく行われること
* 未指定項目が変更されないこと
* 更新対象外項目を変更できないこと
* `X-User-Id`による利用者境界が保証されること
* 論理削除済み利用者・資産口座を更新できないこと
* 同一利用者内で資産口座名の重複を防止できること
* 論理削除済み資産口座も名前重複判定の対象となること
* 他利用者の同名資産口座は重複として扱わないこと
* 同時更新時にもDB制約によって一意性が維持されること
* 利用可能資産設定や保有商品などの関連データが更新されないこと
* 正常・異常レスポンスがAPI契約に従うこと

また、ACC-004は資産口座の通常属性更新に責務を限定するため、
`asset_account_available_settings`や月末資産データなど、
他の業務データへ副作用が発生しないことも確認する。

同一資産口座への同時更新については、
Phase1では楽観ロックを採用しないため、Last Write Winsとなり得ることを前提とし、
想定外のエラーや不整合が発生しないことを確認する。

想定外の例外発生時には、
内部情報をレスポンスへ公開せず、
`requestId`を利用してサーバーログから原因を追跡できることも検証する。


---

## 2. テスト観点

ACC-004では、通常属性更新だけでなく、

- PATCHによる部分更新
- 利用者境界
- 論理削除
- 資産口座名重複
- 更新対象外項目
- 同時更新
- 関連データ非更新
- レスポンス契約

を重点的に確認する。

---

### 2.1 nameのみ更新

以下のリクエストを送信する。

```json
{
  "name": "メイン証券口座"
}
```

以下を確認する。

- `200 OK`となること
- `asset_accounts.name`が更新されること
- `asset_accounts.asset_type`が変更されないこと
- `balance_recording_unit`が変更されないこと
- `start_year_month`が変更されないこと
- `deleted_at`が変更されないこと

---

### 2.2 assetTypeのみ更新

以下のリクエストを送信する。

```json
{
  "assetType": "SECURITIES"
}
```

以下を確認する。

- `200 OK`となること
- `asset_accounts.asset_type`が更新されること
- `asset_accounts.name`が変更されないこと
- その他の更新対象外項目が変更されないこと

---

### 2.3 nameとassetTypeを同時更新

以下を指定する。

```json
{
  "name": "メイン証券口座",
  "assetType": "SECURITIES"
}
```

両方が正しく更新されることを確認する。

---

### 2.4 空リクエスト

以下を送信する。

```json
{}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

`asset_accounts`が更新されないことも確認する。

---

### 2.5 name未指定

`assetType`だけを指定し、`name`未指定がエラーにならないことを確認する。

PATCHとして、未指定項目が必須扱いされないこと。

---

### 2.6 assetType未指定

`name`だけを指定し、`assetType`未指定がエラーにならないことを確認する。

---

### 2.7 name = null

以下を送信する。

```json
{
  "name": null
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 2.8 name = 空文字

以下を送信する。

```json
{
  "name": ""
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 2.9 name最大文字数

テーブル定義で定めた最大文字数ちょうどの値を更新できることを確認する。

最大文字数を超えた場合は、

```text
VALIDATION_ERROR
```

となること。

---

### 2.10 assetType正常値

以下をそれぞれ確認する。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

API値が正しいDB保存形式へ変換されること。

---

### 2.11 assetType不正

例えば、

```json
{
  "assetType": "CRYPTO"
}
```

を送信する。

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 2.12 assetType = null

以下を送信する。

```json
{
  "assetType": null
}
```

バリデーションエラーとなり、業務データが更新されないこと。

---

### 2.13 自分自身と同じname

現在、

```text
id = 10
name = 証券口座
```

の資産口座に対して、

```json
{
  "name": "証券口座"
}
```

を送信する。

重複エラーにならず、正常に処理できることを確認する。

---

### 2.14 同一利用者の別資産口座とのname重複

以下の状態を用意する。

```text
User A
├─ id = 10 / 証券口座
└─ id = 20 / 銀行口座
```

`id = 20`を

```json
{
  "name": "証券口座"
}
```

へ変更する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となること。

---

### 2.15 論理削除済み資産口座とのname重複

同一利用者に、

```text
name = 証券口座
deleted_at IS NOT NULL
```

の別資産口座を用意する。

利用中資産口座を同じ名前へ変更する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となること。

---

### 2.16 他利用者の同名資産口座

以下の状態を用意する。

```text
User A
    証券口座

User B
    銀行口座
```

User Bの`銀行口座`を

```text
証券口座
```

へ変更する。

正常に更新できることを確認する。

---

### 2.17 X-User-Id未指定

`X-User-Id`を指定しない。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

資産口座が更新されないこと。

---

### 2.18 X-User-Id形式不正

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

となること。

---

### 2.19 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 2.20 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 2.21 assetAccountId形式不正

以下の値を確認する。

```text
0
-1
abc
1.5
1e3
10abc
```

期待結果：

```text
400 Bad Request
INVALID_ASSET_ACCOUNT_ID
```

となること。

---

### 2.22 資産口座不存在

存在しない`assetAccountId`を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 2.23 他利用者の資産口座

以下の状態を用意する。

```text
User A
    Asset Account A

User B
    Asset Account B
```

User AとしてAsset Account Bを更新する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの資産口座が変更されないことも確認する。

---

### 2.24 論理削除済み資産口座

対象資産口座を

```text
deleted_at IS NOT NULL
```

としてACC-004を実行する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 2.25 balanceRecordingUnitを送信

以下を送信する。

```json
{
  "balanceRecordingUnit": "ACCOUNT"
}
```

更新対象外項目を拒否する共通方針の場合は、

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

`balance_recording_unit`が変更されないことを確認する。

---

### 2.26 startYearMonthを送信

以下を送信する。

```json
{
  "startYearMonth": "2026-09"
}
```

ACC-004では変更できないことを確認する。

`asset_accounts.start_year_month`が変更されないこと。

---

### 2.27 isAvailableを送信

以下を送信する。

```json
{
  "isAvailable": false
}
```

ACC-004では利用可能資産設定を変更できないことを確認する。

```text
asset_account_available_settings
```

に変更がないこと。

---

### 2.28 isEnabledを送信

以下を送信する。

```json
{
  "isEnabled": false
}
```

ACC-004から無効化できないことを確認する。

```text
deleted_at
```

が変更されないこと。

---

### 2.29 userIdを送信

以下を送信する。

```json
{
  "name": "証券口座",
  "userId": "999"
}
```

未定義項目を拒否する共通方針の場合は、バリデーションエラーとなること。

少なくとも、

```text
asset_accounts.user_id
```

が変更されないことを確認する。

---

### 2.30 isAvailable = true

現在有効な利用可能資産設定が

```text
is_available = true
```

の場合、成功レスポンスで

```json
{
  "isAvailable": true
}
```

となることを確認する。

---

### 2.31 isAvailable = false

現在有効な設定が

```text
is_available = false
```

の場合、

```json
{
  "isAvailable": false
}
```

となることを確認する。

---

### 2.32 現在利用可能資産設定が0件

現在有効な利用可能資産設定を0件とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

となること。

資産口座更新前に設定不整合を検出する実装の場合は、`asset_accounts`が変更されないことも確認する。

---

### 2.33 現在利用可能資産設定が複数件

現在有効な設定を2件以上用意する。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

となること。

任意の1件を成功レスポンスへ使用しないこと。

---

### 2.34 同一値更新

現在値と同じ

```text
name
assetType
```

を指定する。

業務エラーにならないことを確認する。

EloquentのDirty判定によってUPDATEが発行されない場合も正常とする。

---

### 2.35 updated_at

実際に値が変更された場合は、

```text
updated_at
```

が更新されることを確認する。

同一値更新時の`updated_at`の扱いは、実装方針に従う。

---

### 2.36 同時名前更新

同一利用者に属する2つの資産口座を、並行して同じ名前へ変更する。

以下を確認する。

- 同名資産口座が2件存在する状態にならないこと
- DBのUNIQUE制約が機能すること
- 一方が成功すること
- 競合側が`409 Conflict`相当となること
- `ASSET_ACCOUNT_NAME_ALREADY_EXISTS`へ変換されること

---

### 2.37 同一資産口座への同時更新

同じ資産口座に対して異なる値を並行更新する。

Phase1では楽観ロックを行わないため、Last Write Winsとなり得ることを確認する。

想定外の500エラーやデッドロックを通常発生させないことも確認する。

---

### 2.38 関連データ非更新

ACC-004実行前後で、以下が変更されないことを確認する。

- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

---

### 2.39 正常レスポンス契約

正常時に、概念的に以下の形式となることを確認する。

```json
{
  "data": {
    "id": "10",
    "name": "メイン証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

以下を確認する。

- HTTPステータスが`200 OK`
- `id`がstring
- `name`がstring
- `assetType`がAPI用文字列
- `balanceRecordingUnit`がAPI用文字列
- `isAvailable`がboolean
- `startYearMonth`が`YYYY-MM`
- `isEnabled = true`

---

### 2.40 返却しない情報

正常レスポンスへ、以下が含まれないことを確認する。

- `user_id`
- `userId`
- `deleted_at`
- `deletedAt`
- `created_at`
- `createdAt`
- `updated_at`
- `updatedAt`
- `asset_account_available_settings.id`
- `assetAccountAvailableSettingId`
- `asset_account_id`
- `endYearMonth`
- DB内部の数値コード
- 保有商品一覧
- 月末資産データ

---

### 2.41 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
INTERNAL_SERVER_ERROR
```

以下も確認する。

- `error.code`が設定されること
- `error.message`が設定されること
- 必要に応じて`error.details`が設定されること
- `requestId`が設定されること
- SQLが含まれないこと
- PostgreSQL内部エラーが含まれないこと
- 制約名が含まれないこと
- スタックトレースが含まれないこと
- サーバーファイルパスが含まれないこと

---

### 2.42 副作用範囲

正常終了時に変更される業務データが、

```text
対象asset_accounts 1件
```

だけであることを確認する。

更新対象カラムも、リクエストで指定された

```text
name
asset_type
```

に限定されることを確認する。

---

### 2.43 INTERNAL_SERVER_ERROR

更新処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

- 内部情報がレスポンスへ公開されないこと
- サーバーログに調査情報が記録されること
- レスポンスの`requestId`からログを追跡できること

---

## 3. 関連ドキュメント

- [ACC-004 API詳細設計](../../api/details/asset-accounts/acc-004-update.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [資産口座 Laravelアーキテクチャ設計](../../architecture/laravel/asset-accounts/README.md)
- [ACC-004 Laravelアーキテクチャ設計](../../architecture/laravel/asset-accounts/acc-004-update.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [資産口座 Reactアーキテクチャ設計](../../architecture/react/asset-accounts/README.md)
- [ACC-004 Reactアーキテクチャ設計](../../architecture/react/asset-accounts/acc-004-update.md)
- [資産口座 テスト設計](./README.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)
