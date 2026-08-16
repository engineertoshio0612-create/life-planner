# ACC-006 利用可能資産設定登録

## 1. 概要

本ドキュメントでは、ACC-006 利用可能資産設定登録APIに対するテスト観点を定義する。

操作対象利用者に帰属する指定された資産口座について、現在の利用可能資産設定を終了し、新しい設定を正しく登録できることに加え、利用者境界、入力値、開始年月、既存履歴の整合性およびレスポンス契約が設計どおりに機能することを確認する。

また、旧設定の終了と新設定の登録を単一トランザクションとして扱い、一部の更新だけが残らないことや、同一資産口座への同時更新時にも行ロックおよびDB制約によって履歴の一意性・連続性が維持されることを検証対象とする。

ACC-006は利用可能資産設定履歴のみを更新するAPIであるため、資産口座の基本情報や月末資産データ、目的達成判定履歴など、対象外の業務データへ副作用が発生しないことも確認する。

---

## 26. テスト観点

ACC-006では、入力値検証だけでなく、

* 利用者境界
* 履歴整合性
* 開始年月
* 状態変更有無
* 旧設定更新
* 新設定登録
* トランザクション
* 排他制御
* UNIQUE制約
* 副作用範囲

を重点的に確認する。

---

### 26.1 正常系：trueからfalse

現在設定を、

```text
2026-01 ～ NULL
is_available = true
```

とする。

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

実行後、

```text
旧設定
2026-01 ～ 2026-07
true

新設定
2026-08 ～ NULL
false
```

となることを確認する。

---

### 26.2 正常系：falseからtrue

現在設定を、

```text
2026-01 ～ NULL
false
```

とする。

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

実行後、

```text
2026-01 ～ 2026-07
false

2026-08 ～ NULL
true
```

となることを確認する。

---

### 26.3 成功レスポンス

正常時に、

```text
201 Created
```

となることを確認する。

概念的なレスポンス：

```json
{
  "data": {
    "id": "15",
    "startYearMonth": "2026-08",
    "endYearMonth": null,
    "isAvailable": false
  }
}
```

---

### 26.4 idの型

新規設定IDがDB上で`bigint`でも、

```json
{
  "id": "15"
}
```

のようにstringとして返却されることを確認する。

---

### 26.5 startYearMonth未指定

以下を送信する。

```json
{
  "isAvailable": false
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.6 startYearMonth = null

以下を送信する。

```json
{
  "startYearMonth": null,
  "isAvailable": false
}
```

バリデーションエラーとなり、DB更新されないことを確認する。

---

### 26.7 startYearMonth形式不正

以下を確認する。

```text
2026-8
2026/08
202608
2026-00
2026-13
abc
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.8 isAvailable未指定

以下を送信する。

```json
{
  "startYearMonth": "2026-08"
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.9 isAvailable = null

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": null
}
```

バリデーションエラーとなること。

---

### 26.10 isAvailableが文字列

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": "false"
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.11 isAvailableが数値

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": 0
}
```

boolean以外として拒否されることを確認する。

---

### 26.12 X-User-Id未指定

`X-User-Id`を指定しない。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

となること。

---

### 26.13 X-User-Id形式不正

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

### 26.14 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 26.15 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 26.16 assetAccountId形式不正

以下を確認する。

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

### 26.17 資産口座不存在

存在しない`assetAccountId`を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 26.18 他利用者の資産口座

User AとしてUser Bの資産口座を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの`asset_account_available_settings`が一切変更されないことを確認する。

---

### 26.19 論理削除済み資産口座

対象資産口座を

```text
deleted_at IS NOT NULL
```

とする。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

新しい設定が追加されないこと。

---

### 26.20 履歴0件

利用中資産口座に対して利用可能資産設定履歴を0件とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

となること。

設定を自動作成しないこと。

---

### 26.21 初期開始年月不整合

以下の状態を用意する。

```text
asset_accounts.start_year_month
    = 2026-01

最古設定.start_year_month
    = 2026-02
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 26.22 既存履歴の期間重複

以下を用意する。

```text
2026-01 ～ 2026-08
true

2026-08 ～ NULL
false
```

ACC-006を実行しても新設定を追加せず、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 26.23 既存履歴の期間欠落

以下を用意する。

```text
2026-01 ～ 2026-05
true

2026-07 ～ NULL
false
```

履歴不整合となること。

---

### 26.24 継続中設定0件

すべての設定に`end_year_month`が存在する状態を用意する。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 26.25 継続中設定複数件

以下を用意する。

```text
2026-01 ～ NULL
true

2026-08 ～ NULL
false
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

どちらか一方を任意に更新しないこと。

---

### 26.26 startYearMonthが資産口座利用開始年月より前

例えば、

```text
assetAccount.startYearMonth
    = 2026-08
```

に対して、

```json
{
  "startYearMonth": "2026-07",
  "isAvailable": false
}
```

を送信する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

となること。

---

### 26.27 startYearMonthが現在設定開始年月と同じ

現在設定：

```text
2026-08 ～ NULL
true
```

Request：

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

となること。

---

### 26.28 startYearMonthが現在設定より前

現在設定：

```text
2026-08 ～ NULL
false
```

Request：

```json
{
  "startYearMonth": "2026-05",
  "isAvailable": true
}
```

履歴途中への挿入として拒否されること。

---

### 26.29 1か月後から変更

現在設定：

```text
2026-08 ～ NULL
true
```

Request：

```json
{
  "startYearMonth": "2026-09",
  "isAvailable": false
}
```

正常に、

```text
2026-08 ～ 2026-08
true

2026-09 ～ NULL
false
```

となること。

---

### 26.30 年跨ぎ

現在設定：

```text
2026-01 ～ NULL
true
```

Request：

```json
{
  "startYearMonth": "2027-01",
  "isAvailable": false
}
```

旧設定終了年月が

```text
2026-12
```

となること。

---

### 26.31 同じisAvailable

現在設定：

```text
2026-01 ～ NULL
true
```

Request：

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

となること。

以下を確認する。

* 旧設定を更新しない
* 新設定を追加しない
* 履歴件数が変化しない

---

### 26.32 同一startYearMonth重複

同一資産口座に

```text
start_year_month = 2026-08
```

の設定が既に存在する状態で、同じ開始年月の登録を試みる。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

または、より前の業務チェックで`START_YEAR_MONTH_INVALID`となる場合は、チェック順序に従った結果となること。

---

### 26.33 UNIQUE制約

DBレベルで、

```text
asset_account_id
+
start_year_month
```

が重複できないことを確認する。

並行実行などで制約違反が発生した場合は、PostgreSQL例外がAPIへそのまま露出しないこと。

---

### 26.34 継続中設定部分UNIQUE制約

部分UNIQUEインデックスを採用する場合は、同一資産口座に

```text
end_year_month IS NULL
```

の設定を2件作成できないことを確認する。

---

### 26.35 旧設定UPDATE

正常系で、現在設定の

```text
end_year_month
```

だけが新設定開始年月の前月へ更新されることを確認する。

以下は変更されないこと。

```text
start_year_month
is_available
asset_account_id
```

---

### 26.36 新設定INSERT

正常系で、新設定が以下の値で登録されることを確認する。

```text
asset_account_id
    = 対象資産口座ID

start_year_month
    = Request値

end_year_month
    = NULL

is_available
    = Request値
```

---

### 26.37 旧設定更新後に新設定INSERT失敗

新設定INSERTで意図的に例外を発生させる。

以下を確認する。

* APIはエラーとなる
* トランザクションがロールバックされる
* 旧設定の`end_year_month`が元の`NULL`へ戻る
* 新設定が存在しない

---

### 26.38 旧設定UPDATE失敗

旧設定UPDATEで例外を発生させる。

以下を確認する。

* 新設定をINSERTしない
* トランザクションがロールバックされる
* 履歴状態が実行前と同じ

---

### 26.39 Transaction全体

正常時は、

```text
旧設定UPDATE
+
新設定INSERT
```

の両方がコミットされること。

異常時は、両方とも確定しないことを確認する。

---

### 26.40 同時実行

同一資産口座に対して複数のACC-006を並行実行する。

以下を確認する。

* 現在設定への`lockForUpdate()`が機能する
* 同時に同じ旧設定を更新しない
* 継続中設定が複数件にならない
* 履歴期間が重複しない
* 2件目は最新状態で業務条件を再評価する

---

### 26.41 同時に異なる開始年月

例えば、

```text
Request A
2026-08 / false

Request B
2026-09 / false
```

を並行実行する。

Request A成功後、Request Bが最新状態を基準に再判定されることを確認する。

不要な同一状態履歴が作成されないこと。

---

### 26.42 同時に同じ開始年月

以下を並行実行する。

```text
Request A
2026-08 / false

Request B
2026-08 / false
```

以下を確認する。

* 同じ開始年月の設定が2件作成されない
* 一方のみ成功すること
* 競合側が業務エラーへ変換されること
* DB制約違反が外部へ露出しないこと

---

### 26.43 他の関連テーブル非更新

ACC-006実行前後で、以下が変更されないことを確認する。

* `asset_accounts`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

---

### 26.44 過去の月末資産非更新

利用可能資産設定変更によって、既存の

```text
month_end_asset_balances
month_end_holding_values
month_end_asset_snapshots
```

が更新されないことを確認する。

---

### 26.45 過去判定履歴非更新

ACC-006実行によって、

```text
assessment_histories
```

の既存レコードが変更されないことを確認する。

---

### 26.46 更新対象外項目

以下のようなRequestを送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false,
  "endYearMonth": "2026-12"
}
```

未定義項目を拒否する共通方針の場合は、

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.47 currentSettingIdを送信

以下を送信する。

```json
{
  "currentSettingId": "10",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

Request指定のIDによって更新対象を変更できないことを確認する。

---

### 26.48 userIdを送信

以下を送信する。

```json
{
  "userId": "999",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

`user_id`が変更されず、別利用者の資産口座を操作できないことを確認する。

---

### 26.49 assetAccountIdをBodyへ送信

以下を送信する。

```json
{
  "assetAccountId": "999",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

URLの`assetAccountId`以外を対象決定に使用しないことを確認する。

---

### 26.50 正常レスポンス契約

正常時に、以下の形式となることを確認する。

```json
{
  "data": {
    "id": "15",
    "startYearMonth": "2026-08",
    "endYearMonth": null,
    "isAvailable": false
  }
}
```

以下を確認する。

* HTTPステータスが`201 Created`
* `id`がstring
* `startYearMonth`が`YYYY-MM`
* `endYearMonth = null`
* `isAvailable`がboolean

---

### 26.51 返却しない情報

正常レスポンスへ、以下が含まれないことを確認する。

* `assetAccountId`
* `asset_account_id`
* `userId`
* `user_id`
* `createdAt`
* `created_at`
* `updatedAt`
* `updated_at`
* 旧設定情報
* 資産口座名
* 資産種別
* 残高記録単位
* `isEnabled`

---

### 26.52 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
INTERNAL_SERVER_ERROR
```

以下も確認する。

* `error.code`が設定されること
* `error.message`が設定されること
* 必要に応じて`error.details`が設定されること
* `requestId`が設定されること
* SQLが含まれないこと
* SQLSTATEが含まれないこと
* PostgreSQL内部エラーが含まれないこと
* 制約名が含まれないこと
* スタックトレースが含まれないこと

---

### 26.53 エラー時の副作用

業務エラー発生時に、

```text
asset_account_available_settings
```

が中途半端に変更されないことを確認する。

特に、

```text
旧設定終了済み
+
新設定なし
```

や、

```text
旧設定継続中
+
新設定追加済み
```

という状態を残さないこと。

---

### 26.54 INTERNAL_SERVER_ERROR

更新処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

* トランザクションがロールバックされること
* 内部情報がレスポンスへ公開されないこと
* サーバーログに調査情報が記録されること
* レスポンスの`requestId`からログを追跡できること

---

### 3 関連ドキュメント

- [ACC-006 API詳細設計](../../api/details/asset-accounts/acc-006-create-available-setting.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [ACC-006 Laravelアーキテクチャ設計](../../architecture/laravel/asset-accounts/acc-006-create-available-setting.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [ACC-006 Reactアーキテクチャ設計](../../architecture/react/asset-accounts/acc-006-create-available-setting.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
