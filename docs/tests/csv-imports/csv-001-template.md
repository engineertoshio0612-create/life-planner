##  CSV-001 月末資産残高CSVテンプレート取得

### 1 概要

CSV-001のテストでは、月末資産残高CSVインポートで使用するCSVテンプレートが、CSV-002 月末資産残高CSVプレビューおよびCSV-003 月末資産残高CSV登録で利用可能な形式として正しく取得できることを確認する。

本APIは業務データを変更しないCSVテンプレート取得APIであるため、正常なファイルレスポンスだけでなく、CSV仕様、利用者コンテキスト、副作用の有無、後続APIとの整合性を主要なテスト対象とする。

```text
CSV-001
テンプレート取得
    ↓
レスポンスヘッダー確認
    ├─ Content-Type
    └─ Content-Disposition
    ↓
CSV内容確認
    ├─ ヘッダー
    ├─ 項目順序
    ├─ 文字コード
    └─ 不要なデータがないこと
    ↓
CSV-002・CSV-003との整合性確認
```

正常系では、有効な`X-User-Id`を指定した場合に`200 OK`となり、`text/csv; charset=UTF-8`のCSVファイルとして以下のヘッダーが正しい順序で返却されることを確認する。

```text
target_year_month,asset_account_name,balance
```

テンプレートには`user_id`や`asset_account_id`などの内部管理項目、および操作対象利用者が実際に登録している資産口座や月末資産残高などの業務データが含まれないことを確認する。

また、CSVテンプレートは利用者固有の内容を持たないため、異なる利用者や異なる業務データの状態で取得しても同一内容となることを確認する。

利用者コンテキストについては、`X-User-Id`未指定、形式不正、利用者不存在、論理削除済み利用者を検証し、それぞれAPI共通方針に従ったステータスコードおよびエラーコードが返却されることを確認する。

```text
正常時
    → text/csv

異常時
    → application/json
```

CSV-001は参照専用APIであるため、実行前後で`asset_accounts`、`month_end_asset_snapshots`、`month_end_asset_balances`などの業務テーブルに追加・更新・削除が発生しないことも確認する。

さらに、同一条件で複数回実行した場合の冪等性、複数利用者からの同時実行、文字コード・改行コード、想定外例外発生時に内部情報が外部へ公開されないことを検証する。

特にCSV仕様については、CSV-001で取得したテンプレートへ正常なデータを入力した場合に、CSV-002およびCSV-003のヘッダー検証を通過できることを確認し、以下の整合性を保証する。

```text
CSV-001
テンプレートヘッダー
    =
CSV-002
受付ヘッダー
    =
CSV-003
受付ヘッダー
```

これにより、CSV-001が提供するテンプレートをCSVインポートフローの正式な入力形式として利用できることをテストで保証する。

---

### 2 テスト観点

本APIでは、CSVテンプレートがCSV-002およびCSV-003で使用可能な形式として正しく取得できることを確認する。

---

#### 2.1 正常系

以下を確認する。

- 有効な`X-User-Id`を指定すると`200 OK`となること
- `Content-Type`が`text/csv; charset=UTF-8`であること
- `Content-Disposition`が設定されること
- ファイル名が想定した名称であること
- CSVテンプレートを取得できること
- CSVヘッダーが正しいこと
- CSVヘッダーの順序が正しいこと
- CSVテンプレートに不要なデータ行が含まれないこと

期待するCSVヘッダーは、以下とする。

```csv
target_year_month,asset_account_name,balance
```

---

#### 2.2 CSVヘッダー

CSVテンプレートに以下の3項目が正しい順序で含まれることを確認する。

```text
target_year_month
asset_account_name
balance
```

以下のような内部管理項目が含まれていないことも確認する。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- `month_end_asset_balance_id`
- `confirmed`
- `balance_recording_unit`
- `created_at`
- `updated_at`

---

#### 2.3 利用者固有データ

CSVテンプレートに操作対象利用者の実データが含まれないことを確認する。

例えば、利用者に

```text
普通預金
証券口座
```

などの資産口座が登録されていても、テンプレートには出力されないことを確認する。

---

#### 2.4 利用者によるテンプレート差異

異なる利用者でCSV-001を実行しても、CSVテンプレートの内容が同一であることを確認する。

```text
User A
    ↓
target_year_month,asset_account_name,balance

User B
    ↓
target_year_month,asset_account_name,balance
```

---

#### 2.5 X-User-Id未指定

`X-User-Id`を指定しない場合は、

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

となることを確認する。

エラー時にCSVファイルが返却されないことも確認する。

---

#### 2.6 X-User-Id形式不正

以下のような値を指定した場合は、

```text
abc
0
-1
1.5
```

```text
400 Bad Request
INVALID_USER_ID
```

となることを確認する。

---

#### 2.7 利用者不存在

存在しない利用者IDを指定した場合は、

```text
404 Not Found
USER_NOT_FOUND
```

となることを確認する。

---

#### 2.8 論理削除済み利用者

`users.deleted_at`が設定されている利用者を指定した場合は、

```text
404 Not Found
USER_NOT_FOUND
```

となることを確認する。

---

#### 2.9 エラー時のレスポンス形式

エラー時は、CSVではなくAPI共通のJSONエラーレスポンスが返却されることを確認する。

```text
正常時
    → text/csv

異常時
    → application/json
```

となることを確認する。

---

#### 2.10 データベース非更新

CSV-001実行前後で、以下の業務テーブルにレコード追加・更新・削除が発生しないことを確認する。

- `users`
- `asset_accounts`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_account_available_settings`

---

#### 2.11 業務データ非参照

CSVテンプレート生成処理が、以下の業務データに依存しないことを確認する。

- 資産口座の登録件数
- 月末資産残高の登録件数
- 月末資産状況の有無
- 月末資産状況の確定状態
- 利用可能資産設定

これらの状態が異なっていても、同一のCSVテンプレートが取得できることを確認する。

---

#### 2.12 CSV-002・CSV-003との整合性

CSV-001で取得したテンプレートに正常なデータを入力した場合、CSV-002およびCSV-003のヘッダー検証でエラーとならないことを確認する。

特に、以下の整合性を確認する。

```text
CSV-001
テンプレートヘッダー
    =
CSV-002
受付ヘッダー
    =
CSV-003
受付ヘッダー
```

---

#### 2.13 文字コード

CSVテンプレートがCSV共通仕様で定めるUTF-8として正しく取得できることを確認する。

BOMを使用する仕様とした場合は、BOMの有無についてもテストする。

---

#### 2.14 改行コード

CSV共通仕様で改行コードを固定する場合は、期待する改行コードで出力されることを確認する。

環境によって意図せず改行コードが変化しないことを確認する。

---

#### 2.15 冪等性

同一の`X-User-Id`で複数回実行しても、業務データが変更されないことを確認する。

また、同一のシステム状態では同一のCSVテンプレートが取得できることを確認する。

---

#### 2.16 同時実行

同一利用者または複数利用者からCSV-001を同時実行しても、正常にCSVテンプレートを取得できることを確認する。

他のCSV登録処理や月末資産関連APIが実行中であっても、CSV-001が不要なロックによってブロックされないことを確認する。

---

#### 2.17 想定外例外

CSVテンプレート生成処理で想定外の例外が発生した場合は、

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

となることを確認する。

また、レスポンスへ以下の情報が公開されないことを確認する。

- スタックトレース
- Laravel内部例外メッセージ
- ファイルパス
- クラス内部情報
- SQL
- データベース接続情報

---

#### 2.18 主なテストケース

主なテストケースをまとめると、以下とする。

| No. | テスト内容 | 期待結果 |
| ---: | --- | --- |
| 1 | 有効な利用者で取得 | `200 OK` |
| 2 | CSVヘッダー確認 | `target_year_month,asset_account_name,balance` |
| 3 | `Content-Type`確認 | `text/csv; charset=UTF-8` |
| 4 | `Content-Disposition`確認 | CSVダウンロード用ヘッダーが設定される |
| 5 | 利用者固有データ確認 | CSVへ含まれない |
| 6 | 異なる利用者で取得 | 同一テンプレート |
| 7 | `X-User-Id`未指定 | `400 / USER_CONTEXT_REQUIRED` |
| 8 | 利用者ID形式不正 | `400 / INVALID_USER_ID` |
| 9 | 利用者不存在 | `404 / USER_NOT_FOUND` |
| 10 | 論理削除済み利用者 | `404 / USER_NOT_FOUND` |
| 11 | エラー時Content-Type | `application/json` |
| 12 | 業務データ更新確認 | 更新なし |
| 13 | CSV-002とのヘッダー整合性 | 一致する |
| 14 | CSV-003とのヘッダー整合性 | 一致する |
| 15 | 複数回実行 | 副作用なし |
| 16 | 同時実行 | 正常取得可能 |
| 17 | 想定外例外 | `500 / INTERNAL_SERVER_ERROR` |

---

### 3 関連ドキュメント

- [CSV-001 API詳細設計](../../api/details/csv-imports/csv-001-template.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [CSV-001 Laravelアーキテクチャ設計](../../architecture/laravel/csv-imports/csv-001-template.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [CSV-001 Reactアーキテクチャ設計](../../architecture/react/csv-imports/csv-001-template.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)