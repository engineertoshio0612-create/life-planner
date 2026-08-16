# ACC-003 資産口座詳細取得

## 1. 概要

本ドキュメントでは、ACC-003 資産口座詳細取得APIに対するテスト観点を定義する。

操作対象利用者に帰属する指定された資産口座について、基本情報および現在年月に有効な利用可能資産設定を正しく取得できることに加え、利用者境界、資産口座IDの妥当性、論理削除およびレスポンス契約が設計どおりに機能することを確認する。

また、利用可能資産設定の期間境界、設定の不存在・重複といったデータ不整合を適切に検知できることを確認する。年月に依存するテストでは現在日時を固定し、再現可能な状態で検証する。

ACC-003は参照専用APIであるため、詳細取得によって資産口座や利用可能資産設定などの業務データへ副作用が発生しないことも検証対象とする。

---

## 2. テスト観点

ACC-003では、資産口座詳細の正常取得だけでなく、以下を重点的に確認する。

* 利用者境界
* 論理削除
* 現在年月判定
* 利用可能資産設定の期間境界
* 設定不存在・重複
* レスポンス契約
* 副作用なし

---

### 2.1 正常系

操作対象利用者に属する有効な資産口座と、現在有効な利用可能資産設定を1件用意する。

ACC-003を実行し、以下を確認する。

* `200 OK`となること
* `id`が対象資産口座IDとなること
* `name`が正しいこと
* `assetType`が正しいこと
* `balanceRecordingUnit`が正しいこと
* `startYearMonth`が正しいこと
* `isAvailable`が現在設定と一致すること
* `isEnabled = true`となること

---

### 2.2 X-User-Id未指定

`X-User-Id`を指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

以下を確認する。

* 資産口座検索へ進まないこと
* 業務データが更新されないこと

---

### 2.3 X-User-Id形式不正

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

### 2.4 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 2.5 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 2.6 assetAccountId形式不正

以下の値をそれぞれ確認する。

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

資産口座検索へ進まないことも確認する。

---

### 2.7 資産口座不存在

存在しない`assetAccountId`を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 2.8 他利用者の資産口座

以下の状態を用意する。

```text
User A
    Asset Account A

User B
    Asset Account B
```

User Aとして、Asset Account BのIDを指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの資産口座情報がレスポンスへ含まれないこと。

---

### 2.9 論理削除済み資産口座

以下の資産口座を用意する。

```text
deleted_at IS NOT NULL
```

そのIDでACC-003を実行する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 2.10 assetType = CASH

DB上の資産種別が現金に対応する値の場合、レスポンスが

```text
CASH
```

となることを確認する。

---

### 2.11 assetType = BANK

銀行に対応するDB値から、

```text
BANK
```

へ正しく変換されること。

---

### 2.12 assetType = SECURITIES

証券に対応するDB値から、

```text
SECURITIES
```

へ正しく変換されること。

---

### 2.13 assetType = IDECO

iDeCoに対応するDB値から、

```text
IDECO
```

へ正しく変換されること。

---

### 2.14 assetType = CORPORATE_DC

企業型DCに対応するDB値から、

```text
CORPORATE_DC
```

へ正しく変換されること。

---

### 2.15 assetType = OTHER

その他に対応するDB値から、

```text
OTHER
```

へ正しく変換されること。

---

### 2.16 balanceRecordingUnit = ACCOUNT

DB値が口座単位に対応する場合、

```text
ACCOUNT
```

として返却されることを確認する。

---

### 2.17 balanceRecordingUnit = HOLDING

DB値が商品単位に対応する場合、

```text
HOLDING
```

として返却されることを確認する。

---

### 2.18 isAvailable = true

現在有効な利用可能資産設定が

```text
is_available = true
```

の場合、

```json
{
  "isAvailable": true
}
```

となること。

---

### 2.19 isAvailable = false

現在有効な利用可能資産設定が

```text
is_available = false
```

の場合、

```json
{
  "isAvailable": false
}
```

となること。

`false`を未設定扱いしないこと。

---

### 2.20 start_year_monthが現在年月と一致

例えば、現在年月が

```text
2026-08
```

で、

```text
start_year_month = 2026-08
end_year_month   = NULL
```

の場合、現在有効な設定として取得されること。

---

### 2.21 end_year_monthが現在年月と一致

現在年月が

```text
2026-08
```

で、

```text
start_year_month = 2026-01
end_year_month   = 2026-08
```

の場合、現在有効な設定として取得されること。

---

### 2.22 end_year_monthが前月

現在年月が

```text
2026-08
```

で、

```text
end_year_month = 2026-07
```

の場合、当該設定が現在有効として取得されないこと。

---

### 2.23 start_year_monthが翌月

現在年月が

```text
2026-08
```

で、

```text
start_year_month = 2026-09
```

の場合、当該設定が現在有効として取得されないこと。

---

### 2.24 end_year_monthがNULL

現在年月が`start_year_month`以降で、

```text
end_year_month = NULL
```

の場合、現在有効な設定として取得されること。

---

### 2.25 現在有効な設定が0件

対象資産口座について、現在年月に有効な利用可能資産設定を0件とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

以下を確認する。

* `isAvailable = false`へ補完されないこと
* 正常レスポンスを返さないこと
* 業務データを自動修復しないこと

---

### 2.26 現在有効な設定が複数件

例えば、以下の2件を用意する。

```text
設定A
start_year_month = 2026-01
end_year_month   = NULL

設定B
start_year_month = 2026-06
end_year_month   = NULL
```

現在年月を`2026-08`とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

となること。

最新1件だけを任意に返却しないこと。

---

### 2.27 過去設定と現在設定

以下の履歴を用意する。

```text
設定A
2026-01 ～ 2026-06
is_available = true

設定B
2026-07 ～ NULL
is_available = false
```

現在年月が`2026-08`の場合、

```text
設定B
```

だけが選択され、

```text
isAvailable = false
```

となること。

---

### 2.28 境界が重複しない正常履歴

例えば、

```text
設定A
2026-01 ～ 2026-06

設定B
2026-07 ～ NULL
```

の場合、現在年月に応じて必ず1件だけが選択されることを確認する。

---

### 2.29 保有商品を返却しない

対象資産口座に複数の`holding_assets`が存在する状態でACC-003を実行する。

レスポンスへ、以下が含まれないことを確認する。

* `holdingAssets`
* 保有商品名
* 保有商品ID

---

### 2.30 月末資産データを返却しない

対象資産口座に月末資産データが存在する状態でも、レスポンスへ以下が含まれないことを確認する。

* `monthEndAssetSnapshots`
* `monthEndAssetBalances`
* `monthEndHoldingValues`
* 最新残高
* 資産推移

---

### 2.31 正常レスポンス契約

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

* HTTPステータスが`200 OK`
* `id`がstring
* `name`がstring
* `assetType`がstring
* `balanceRecordingUnit`がstring
* `isAvailable`がboolean
* `startYearMonth`が`YYYY-MM`
* `isEnabled`がboolean
* `isEnabled = true`

---

### 2.32 返却しない情報

正常レスポンスに、以下が含まれないことを確認する。

* `user_id`
* `userId`
* `deleted_at`
* `deletedAt`
* `created_at`
* `createdAt`
* `updated_at`
* `updatedAt`
* `asset_account_available_settings.id`
* `assetAccountAvailableSettingId`
* `asset_account_id`
* `endYearMonth`
* DB内部の数値コード

---

### 2.33 副作用なし

ACC-003実行前後で、以下のテーブルに変更がないことを確認する。

* `users`
* `asset_accounts`
* `asset_account_available_settings`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

特に、詳細取得によって

```text
updated_at
```

が更新されないことを確認する。

---

### 2.34 同一リクエストの再実行

サーバー状態を変更せずに、同じ

```text
X-User-Id
+
assetAccountId
```

でACC-003を複数回実行する。

同じレスポンス内容となることを確認する。

---

### 2.35 ACC-006後の再取得

最初に、

```text
isAvailable = true
```

となるACC-003を実行する。

その後、ACC-006によって現在設定を変更し、再度ACC-003を実行する。

最新状態の

```text
isAvailable
```

が返却されることを確認する。

---

### 2.36 資産口座無効化後の再取得

ACC-003で正常取得できる資産口座を、ACC-005で無効化する。

その後、同じIDでACC-003を実行する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 2.37 currentYearMonthのテスト固定

利用可能資産設定の対象年月判定をテストする際は、Laravelの時刻固定機能などを使用して現在日時を固定する。

例えば、

```text
現在日時
2026-08-15
```

に固定し、

```text
現在年月
2026-08
```

として境界条件を確認する。

テスト実行日によって結果が変わらないようにする。

---

### 2.38 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
INTERNAL_SERVER_ERROR
```

以下も確認する。

* `error.code`が設定されること
* `error.message`が設定されること
* `requestId`が設定されること
* SQLが含まれないこと
* PostgreSQL内部エラーが含まれないこと
* スタックトレースが含まれないこと
* サーバーファイルパスが含まれないこと

---

### 2.39 データ不整合ログ

以下を発生させた場合に、調査可能なログが記録されることを確認する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text
requestId
userId
assetAccountId
currentYearMonth
```

などから対象ログを追跡できることを確認する。

---

### 2.40 INTERNAL_SERVER_ERROR

資産口座詳細取得処理で想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

* 業務データが更新されないこと
* 内部情報がレスポンスへ公開されないこと
* サーバーログに調査情報が記録されること
* レスポンスの`requestId`からログを追跡できること

---

### 3 関連ドキュメント

- [ACC-003 API詳細設計](../../api/details/asset-accounts/acc-003-detail.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [ACC-003 Laravelアーキテクチャ設計](../../architecture/laravel/asset-accounts/acc-003-detail.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [ACC-003 Reactアーキテクチャ設計](../../architecture/react/asset-accounts/acc-003-detail.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
