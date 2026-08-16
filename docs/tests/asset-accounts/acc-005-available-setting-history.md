# ACC-005 利用可能資産設定履歴取得

## 1. 概要

本ドキュメントでは、ACC-005 利用可能資産設定履歴取得APIに対するテスト観点を定義する。

操作対象利用者に帰属する指定された資産口座について、`asset_account_available_settings`に保持された利用可能資産設定履歴を正しく取得できることに加え、利用者境界、論理削除、並び順およびレスポンス契約が設計どおりに機能することを確認する。

また、初期設定の開始年月、期間の連続性、期間重複・欠落、継続中設定の件数など、履歴全体の整合性を検証し、不整合なデータを正常レスポンスとして返却しないことを確認する。

ACC-005は参照専用APIであるため、履歴取得によって資産口座や利用可能資産設定などの業務データへ副作用が発生しないことも検証対象とする。


---

## 2. テスト観点

ACC-005では、正常な履歴一覧取得だけでなく、

- 利用者境界
- 論理削除
- 並び順
- 履歴0件
- 初期設定開始年月
- 期間整合性
- 期間重複
- 期間欠落
- 継続中設定
- レスポンス契約
- 副作用なし

を重点的に確認する。

---

### 2.1 正常系：履歴1件

以下の状態を用意する。

```text
asset_accounts.start_year_month
    = 2026-08

利用可能資産設定
2026-08 ～ NULL
is_available = true
```

ACC-005を実行し、以下を確認する。

- `200 OK`
- `data`が1件
- `startYearMonth = "2026-08"`
- `endYearMonth = null`
- `isAvailable = true`

---

### 2.2 正常系：複数履歴

以下の履歴を用意する。

```text
2026-01 ～ 2026-06
true

2026-07 ～ 2026-12
false

2027-01 ～ NULL
true
```

以下を確認する。

- `200 OK`
- 3件取得できる
- 全履歴が返却される
- 最新設定だけに限定されない

---

### 2.3 並び順

複数履歴が存在する場合、レスポンスが

```text
startYearMonth DESC
```

となることを確認する。

期待順：

```text
2027-01
2026-07
2026-01
```

---

### 2.4 idの型

DB上のIDが`bigint`であっても、レスポンスでは

```json
{
  "id": "3"
}
```

のようにstringとなることを確認する。

---

### 2.5 endYearMonth = null

最新の継続中設定について、

```json
{
  "endYearMonth": null
}
```

となることを確認する。

以下の値に変換されないこと。

```text
""
"0000-00"
"9999-12"
```

---

### 2.6 isAvailable = true

```text
is_available = true
```

の場合、

```json
{
  "isAvailable": true
}
```

となることを確認する。

---

### 2.7 isAvailable = false

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

`false`を未設定扱いしない。

---

### 2.8 X-User-Id未指定

`X-User-Id`を指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

となること。

---

### 2.9 X-User-Id形式不正

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

### 2.10 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 2.11 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 2.12 assetAccountId形式不正

以下をそれぞれ確認する。

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

### 2.13 資産口座不存在

存在しない`assetAccountId`を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 2.14 他利用者の資産口座

以下の状態を用意する。

```text
User A
    Asset Account A
        Setting A

User B
    Asset Account B
        Setting B
```

User AとしてAsset Account Bを指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

Setting Bがレスポンスへ含まれないこと。

---

### 2.15 論理削除済み資産口座

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

DBに利用可能資産設定履歴が残っていても取得できないこと。

---

### 2.16 履歴0件

利用中の資産口座を用意し、利用可能資産設定履歴を0件とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

となること。

以下も確認する。

- `data: []`を返さない
- 設定を自動登録しない
- 業務データを変更しない

---

### 2.17 初期開始年月一致

以下の状態を用意する。

```text
asset_accounts.start_year_month
    = 2026-01

最古設定
start_year_month
    = 2026-01
```

正常に取得できることを確認する。

---

### 2.18 初期開始年月不一致

以下の状態を用意する。

```text
asset_accounts.start_year_month
    = 2026-01

最古設定
start_year_month
    = 2026-02
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 2.19 利用開始前の設定

以下の状態を用意する。

```text
asset_accounts.start_year_month
    = 2026-08

最古設定
start_year_month
    = 2026-07
```

履歴不整合となることを確認する。

---

### 2.20 startYearMonth = endYearMonth

以下の設定を用意する。

```text
start_year_month = 2026-08
end_year_month   = 2026-08
```

1か月だけ有効な設定として正常に扱われることを確認する。

---

### 2.21 startYearMonth > endYearMonth

以下を用意する。

```text
start_year_month = 2026-08
end_year_month   = 2026-07
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 2.22 正常な連続期間

以下の履歴を用意する。

```text
設定A
2026-01 ～ 2026-06

設定B
2026-07 ～ 2026-12

設定C
2027-01 ～ NULL
```

正常に取得できることを確認する。

---

### 2.23 期間重複

以下を用意する。

```text
設定A
2026-01 ～ 2026-08

設定B
2026-08 ～ NULL
```

2026-08が両設定へ含まれるため、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 2.24 期間欠落

以下を用意する。

```text
設定A
2026-01 ～ 2026-05

設定B
2026-07 ～ NULL
```

2026-06が欠落しているため、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 2.25 1か月単位の連続性

以下を用意する。

```text
設定A
2026-01 ～ 2026-01

設定B
2026-02 ～ NULL
```

正常な連続履歴として扱われることを確認する。

---

### 2.26 継続中設定1件

以下のように、

```text
end_year_month = NULL
```

の設定が1件だけ存在する場合は、正常とする。

---

### 2.27 継続中設定が複数件

以下を用意する。

```text
設定A
2026-01 ～ NULL

設定B
2026-08 ～ NULL
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 2.28 継続中設定が0件

利用中資産口座について、すべての設定に`end_year_month`が存在する状態を用意する。

例えば、

```text
設定A
2026-01 ～ 2026-06

設定B
2026-07 ～ 2026-12
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 2.29 最新設定がNULL

開始年月が最も新しい設定の

```text
end_year_month
```

が`NULL`であることを正常条件として確認する。

---

### 2.30 同一startYearMonth

可能であればDB制約を迂回したテストデータ等で、

```text
同一asset_account_id
+
同一start_year_month
```

の設定を複数用意する。

履歴不整合として正常レスポンスを返さないことを確認する。

通常の登録経路では、DBのUNIQUE制約によって防止されることも確認する。

---

### 2.31 現在年月に依存しない

ACC-005は履歴全体取得APIであるため、サーバーの現在年月を変えても同じDB状態であれば同じ履歴一覧が返却されることを確認する。

ACC-003のような現在年月フィルタを行っていないことを確認する。

---

### 2.32 過去設定も返却する

終了済みの

```text
end_year_month IS NOT NULL
```

の設定もレスポンスへ含まれることを確認する。

---

### 2.33 資産口座情報を返却しない

レスポンスへ、以下が含まれないことを確認する。

- `assetAccountId`
- `assetAccountName`
- `assetType`
- `balanceRecordingUnit`
- `assetAccountStartYearMonth`
- `isEnabled`

資産口座詳細はACC-003の責務とする。

---

### 2.34 DB内部項目を返却しない

レスポンスへ、以下が含まれないことを確認する。

- `asset_account_id`
- `created_at`
- `updated_at`
- `user_id`
- `deleted_at`

---

### 2.35 isCurrentを返却しない

Phase1では、各履歴へ

```text
isCurrent
```

が追加されていないことを確認する。

現在設定は、

```text
endYearMonth = null
```

から判断できる。

---

### 2.36 正常レスポンス契約

正常時に、概念的に以下の形式となることを確認する。

```json
{
  "data": [
    {
      "id": "3",
      "startYearMonth": "2027-01",
      "endYearMonth": null,
      "isAvailable": true
    },
    {
      "id": "2",
      "startYearMonth": "2026-07",
      "endYearMonth": "2026-12",
      "isAvailable": false
    },
    {
      "id": "1",
      "startYearMonth": "2026-01",
      "endYearMonth": "2026-06",
      "isAvailable": true
    }
  ]
}
```

以下を確認する。

- HTTPステータスが`200 OK`
- `data`が配列
- 正常時は1件以上
- `id`がstring
- `startYearMonth`が`YYYY-MM`
- `endYearMonth`がstringまたは`null`
- `isAvailable`がboolean
- `startYearMonth DESC`である

---

### 2.37 副作用なし

ACC-005実行前後で、以下のテーブルに変更がないことを確認する。

- `users`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

特に、

```text
created_at
updated_at
deleted_at
```

が履歴取得によって変更されないことを確認する。

---

### 2.38 同一リクエストの再実行

サーバー状態を変更せずに、同じ

```text
X-User-Id
+
assetAccountId
```

でACC-005を複数回実行する。

同じ履歴内容、同じ並び順となることを確認する。

---

### 2.39 ACC-006後の再取得

最初にACC-005で履歴を取得する。

その後、ACC-006によって新しい利用可能資産設定を登録する。

再度ACC-005を実行し、

- 新しい履歴が追加されていること
- 旧設定の`endYearMonth`が更新されていること
- 新設定が先頭に表示されること
- 履歴全体の連続性が維持されていること

を確認する。

---

### 2.40 資産口座無効化後

ACC-005で正常取得できる資産口座を無効化する。

その後、同じ`assetAccountId`でACC-005を実行する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

履歴データ自体がDBに残っていても取得できないことを確認する。

---

### 2.41 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
INTERNAL_SERVER_ERROR
```

以下も確認する。

- `error.code`が設定されること
- `error.message`が設定されること
- `requestId`が設定されること
- SQLが含まれないこと
- PostgreSQL内部エラーが含まれないこと
- 制約名が含まれないこと
- スタックトレースが含まれないこと
- サーバーファイルパスが含まれないこと

---

### 2.42 履歴不整合ログ

以下のエラーを発生させた場合に、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

調査可能なログが記録されることを確認する。

必要に応じて、

```text
requestId
userId
assetAccountId
historyCount
invalidReason
```

などから原因を追跡できることを確認する。

---

### 2.43 INTERNAL_SERVER_ERROR

履歴取得処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

- 業務データが更新されないこと
- 内部情報がレスポンスへ公開されないこと
- サーバーログに調査情報が記録されること
- レスポンスの`requestId`からログを追跡できること

---

## 3. 関連ドキュメント

- [ACC-005 API詳細設計](../../api/details/asset-accounts/acc-005-available-setting-history.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [ACC-005 Laravelアーキテクチャ設計](../../architecture/laravel/asset-accounts/acc-005-available-setting-history.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [ACC-005 Reactアーキテクチャ設計](../../architecture/react/asset-accounts/acc-005-available-setting-history.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
