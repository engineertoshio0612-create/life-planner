##  CSV-004 商品別月末評価額CSVテンプレート取得

### 1 概要

CSV-004のテストでは、商品別月末評価額CSVインポートで使用するCSVテンプレートが、CSV-005 商品別月末評価額CSVプレビューおよびCSV-006 商品別月末評価額CSV登録で利用可能な形式として正しく取得できることを確認する。

本APIは商品単位用CSVテンプレートの取得のみを行い、商品別月末評価額や月末資産状況などの業務データを変更しない。そのため、CSVテンプレートの形式、HTTPレスポンス、利用者コンテキスト、利用者固有データの非混入、副作用の有無、後続APIとの整合性を主要なテスト対象とする。

```text
CSV-004
テンプレート取得
    ↓
レスポンス確認
    ├─ HTTPステータス
    ├─ Content-Type
    └─ Content-Disposition
    ↓
CSV内容確認
    ├─ ヘッダー
    ├─ 項目順序
    ├─ データ行なし
    └─ 利用者固有データなし
    ↓
CSV-005・CSV-006との整合性確認
```

正常系では、有効な`X-User-Id`を指定した場合に`200 OK`となり、CSVファイルとして以下のヘッダーが正しい順序で返却されることを確認する。

```text
target_year_month,asset_account_name,holding_asset_name,value
```

テンプレートには余分な列や不足した列を含めず、`users.id`、`asset_accounts.id`、`holding_assets.id`などの内部管理項目や、資産口座名、保有商品名、登録済み商品別月末評価額などの利用者固有データが出力されないことを確認する。

CSV-004のテンプレートは利用者の業務データに依存しないため、資産口座、保有商品、商品単位の資産口座が0件の場合でも正常に取得できることを確認する。また、異なる有効な利用者で実行した場合でも同一のCSVテンプレートが返却されることを確認する。

利用者コンテキストについては、`X-User-Id`未指定、形式不正、利用者不存在、論理削除済み利用者を検証し、それぞれAPI共通方針に従ったステータスコードおよびエラーコードが返却されることを確認する。

```text
正常時
    → text/csv

異常時
    → application/json
```

CSV-004は参照専用APIであるため、実行前後で`asset_accounts`、`holding_assets`、`month_end_asset_snapshots`、`month_end_holding_values`などの業務テーブルにINSERT、UPDATE、DELETEが発生しないことも確認する。

また、同一利用者で複数回実行した場合でも毎回同じCSV内容が取得でき、業務データへ影響しないことを確認することで、本APIの冪等性を保証する。

特にCSV仕様については、CSV-004が生成するヘッダーとCSV-005・CSV-006が期待するCSV Definitionが一致することを確認する。

```text
CSV-004
生成ヘッダー
    =
CSV-005
期待ヘッダー
    =
CSV-006
期待ヘッダー
```

CSV-004で取得した正式なテンプレートへ正常なデータを入力した場合に、CSV-005およびCSV-006でヘッダー不一致とならないことを確認し、商品別月末評価額CSVインポート全体で一貫した入力形式を利用できることをテストで保証する。

---

### 2 テスト観点

CSV-004では、
CSVテンプレートの形式、
利用者コンテキスト、
HTTPレスポンス、
副作用がないことを
中心に確認する。

---

#### 2.1 正常系

以下を確認する。

- `200 OK`が返却されること
- `Content-Type`が`text/csv`であること
- 必要な文字コード指定が付与されること
- `Content-Disposition`が設定されること
- CSVファイル名が仕様どおりであること
- CSVヘッダーが正しいこと
- CSVヘッダー順序が正しいこと
- データ行が含まれないこと
- JSON成功Envelopeが返却されないこと

期待するCSVは、
以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

---

#### 2.2 ヘッダー

以下の順序で
出力されることを確認する。

```text
1. target_year_month
2. asset_account_name
3. holding_asset_name
4. value
```

以下が含まれないことも
確認する。

- 余分な列
- 不足した列
- 列順の変更
- CSV-004専用の独自列

---

#### 2.3 利用者固有データ

テンプレートに、
以下が含まれないことを確認する。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`
- `asset_accounts.name`
- `holding_assets.name`
- `month_end_holding_values.value`
- `asset_accounts.balance_recording_unit`
- `month_end_asset_snapshots.confirmed`

利用者に
資産口座や保有商品が
多数登録されていても、
CSV内容が変化しないことを確認する。

---

#### 2.4 資産口座0件

操作対象利用者に
資産口座が存在しない場合でも、

```http
200 OK
```

となり、
正常なテンプレートを
取得できることを確認する。

---

#### 2.5 保有商品0件

操作対象利用者に
保有商品が存在しない場合でも、
正常なテンプレートを
取得できることを確認する。

---

#### 2.6 商品単位資産口座0件

操作対象利用者に

```text
asset_accounts.balance_recording_unit
    = 商品単位
```

の資産口座が
存在しない場合でも、
正常なテンプレートを
取得できることを確認する。

---

#### 2.7 X-User-Id未指定

`X-User-Id`を
指定しない場合に、

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

となることを確認する。

CSVレスポンスを
返却しないことも確認する。

---

#### 2.8 X-User-Id形式不正

例えば、

```text
0
-1
abc
1.5
```

を指定した場合に、

```text
400 Bad Request
INVALID_USER_ID
```

となることを確認する。

---

#### 2.9 利用者不存在

存在しない
`X-User-Id`を指定した場合に、

```text
404 Not Found
USER_NOT_FOUND
```

となることを確認する。

---

#### 2.10 論理削除済み利用者

`users.deleted_at`が
設定されている利用者を指定した場合に、

```text
404 Not Found
USER_NOT_FOUND
```

となることを確認する。

---

#### 2.11 エラーレスポンス形式

異常時は、
CSVではなく
API共通のJSONエラー形式が
返却されることを確認する。

概念的には、

```text
Content-Type
    = application/json
```

となることを確認する。

---

#### 2.12 INTERNAL_SERVER_ERROR

CSV生成処理で
想定外例外が発生した場合に、

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

となることを確認する。

レスポンスへ、

- スタックトレース
- Laravel内部例外
- サーバーファイルパス

などが含まれないことも確認する。

---

#### 2.13 副作用

CSV-004実行前後で、
以下のテーブルに
変更がないことを確認する。

- `users`
- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`
- `asset_account_available_settings`

INSERT、
UPDATE、
DELETEが
発生していないことを確認する。

---

#### 2.14 冪等性

同じ利用者で
CSV-004を複数回実行した場合に、

- 毎回`200 OK`となること
- 毎回同じCSV内容となること
- 業務データが変更されないこと

を確認する。

---

#### 2.15 利用者間のテンプレート一致

異なる有効な利用者で
CSV-004を実行した場合でも、
同一のCSVテンプレートが
返却されることを確認する。

利用者固有データが
混入していないことを保証する。

---

#### 2.16 CSV-005・CSV-006との整合性

CSV-004で取得した
テンプレートのヘッダーが、
CSV-005およびCSV-006で使用する
CSV Definitionと
一致することを確認する。

概念的には、

```text
CSV-004
生成ヘッダー

=

CSV-005
期待ヘッダー

=

CSV-006
期待ヘッダー
```

となることを確認する。

CSV-004で取得した
正式なテンプレートへ
正常なデータを入力した場合に、
ヘッダー不一致を理由として
CSV-005・CSV-006で
エラーにならないことを保証する。

---

### 3 関連ドキュメント

- [CSV-004 API詳細設計](../../api/details/csv-imports/csv-004-template.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [CSV-004 Laravelアーキテクチャ設計](../../architecture/laravel/csv-imports/csv-004-template.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [CSV-004 Reactアーキテクチャ設計](../../architecture/react/csv-imports/csv-004-template.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)