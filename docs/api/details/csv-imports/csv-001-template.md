##  CSV-001 月末資産残高CSVテンプレート取得

### 1 概要

月末資産残高CSVインポートで使用する
CSVテンプレートを取得する。

本APIでは、
口座単位で月末資産残高を登録するための
CSVテンプレートを返却する。

テンプレートには、
月末資産残高CSVで必要となる
CSVヘッダーを含める。

Phase1では、
以下の項目を
CSVテンプレートとして提供する。

```text
対象年月
資産口座名
月末残高
```

本APIでは、CSVテンプレートのみを返却し、月末資産残高の登録・更新・削除は行わない。

また、月末資産状況、資産口座、利用可能資産設定などの業務データも変更しない。

---

### 2 ユースケース

利用者は、月末資産残高CSVを作成する際に、本APIからCSVテンプレートを取得する。

基本的な利用フローは、以下とする。

```text
CSV-001
月末資産残高CSVテンプレート取得
    ↓
利用者がCSVへデータを入力
    ↓
CSV-002
月末資産残高CSVプレビュー
    ↓
入力内容・エラー確認
    ↓
CSV-003
月末資産残高CSV登録
```

本APIは、CSVファイルの入力形式を提供することに責務を限定する。

CSV内容の検証、月末資産残高の登録、月末資産状況の作成・更新などは行わない。

---

### 3 エンドポイント

```http
GET /api/v1/month-end-asset-balances/csv-template
```

---

### 4 HTTPメソッド

```text
GET
```

本APIは、CSVテンプレートを取得する読み取り専用APIである。

API実行によって、業務データの状態は変更されない。

---

### 5 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、`X-User-Id`リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

利用者IDは、以下では受け付けない。

- パスパラメータ
- クエリパラメータ
- リクエストボディ

操作対象利用者は、API共通方針に従い、`X-User-Id`から特定する。

本APIが返却するCSVテンプレートの構造は、利用者によって変化しない。

そのため、テンプレート生成のために操作対象利用者に属する以下の業務データを取得しない。

- `asset_accounts`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `asset_account_available_settings`

概念的な処理は、以下とする。

```text
X-User-Id
    ↓
利用者コンテキスト確認
    ↓
固定のCSVテンプレート生成
    ↓
CSVファイルとして返却
```

CSVテンプレートへ、利用者固有の情報を埋め込まない。

例えば、以下の情報は含めない。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- 資産口座名の実データ
- 月末資産残高の実データ
- `confirmed`
- `created_at`
- `updated_at`

本APIにおける`X-User-Id`の利用目的は、利用者ごとのCSVテンプレートを生成することではなく、Phas

---

### 6 パスパラメータ

本APIでは、パスパラメータを使用しない。

エンドポイントは、以下とする。

```http
GET /api/v1/month-end-asset-balances/csv-template
```

CSVテンプレートは、特定の月末資産残高、資産口座、対象年月などを指定して取得するリソースではない。

そのため、以下のような情報をパスパラメータとして受け付けない。

- `assetAccountId`
- `snapshotId`
- `targetYearMonth`
- `userId`

操作対象利用者は、`X-User-Id`リクエストヘッダーから特定する。

---

### 7 クエリパラメータ

本APIでは、クエリパラメータを使用しない。

CSVテンプレートの内容は、対象年月や資産口座などの条件によって変更しない。

そのため、以下のようなクエリパラメータは受け付けない。

```text
targetYearMonth
assetAccountId
format
```

CSVテンプレートの仕様は、API側で固定する。

---

### 8 リクエストヘッダー

本APIでは、以下のリクエストヘッダーを使用する。

| ヘッダー | 必須 | 内容 |
| --- | :---: | --- |
| `X-User-Id` | ○ | 操作対象となる利用者ID |

リクエスト例：

```http
GET /api/v1/month-end-asset-balances/csv-template
X-User-Id: 1
```

`X-User-Id`の詳細な扱いは、API共通方針に従う。

---

### 9 リクエストボディ

本APIでは、リクエストボディを使用しない。

本APIはGET APIであり、CSVテンプレートの取得条件としてクライアントから業務データを受け取る必要がない。

以下のような情報をリクエストボディで受け付けない。

- `targetYearMonth`
- `assetAccountId`
- `balance`
- `userId`

---

### 10 Content-Type

本APIでは、リクエストボディを使用しないため、リクエストの`Content-Type`は必須としない。

レスポンスでは、CSVファイルを返却する。

レスポンスの`Content-Type`は、以下とする。

```http
Content-Type: text/csv; charset=UTF-8
```

CSVファイルとしてダウンロード可能となるよう、`Content-Disposition`も設定する。

概念例：

```http
Content-Disposition: attachment; filename="month-end-asset-balances-template.csv"
```

実際のファイル名は、CSVテンプレートの命名規則に従う。

---

### 11 リクエスト例

```http
GET /api/v1/month-end-asset-balances/csv-template
X-User-Id: 1
```

本APIでは、パスパラメータ、クエリパラメータ、リクエストボディを指定しない。

---

### 12 バリデーション

本APIでは、CSVテンプレート取得のための業務入力値を受け取らない。

そのため、以下に対する個別のバリデーションは行わない。

- パスパラメータ
- クエリパラメータ
- リクエストボディ

ただし、`X-User-Id`については、API共通方針に従って検証する。

---

#### 12.1 X-User-Id必須

`X-User-Id`は必須とする。

以下のようにヘッダーが指定されていない場合は、正常処理を行わない。

```http
GET /api/v1/month-end-asset-balances/csv-template
```

API共通方針に従い、

```text
USER_CONTEXT_REQUIRED
```

として扱う。

---

#### 12.2 X-User-Idの形式

`X-User-Id`は、API共通方針で定める利用者ID形式を満たす必要がある。

例えば、以下のような値は不正とする。

```text
0
-1
abc
1.5
```

形式が不正な場合は、

```text
INVALID_USER_ID
```

として扱う。

---

#### 12.3 利用者の存在確認

`X-User-Id`で指定された利用者が存在することを確認する。

また、論理削除済み利用者は操作対象として扱わない。

概念的には、以下を満たす利用者を有効とする。

```sql
users.id = X-User-Id
AND
users.deleted_at IS NULL
```

利用者が存在しない場合、または論理削除されている場合は、

```text
USER_NOT_FOUND
```

として扱う。

---

#### 12.4 CSV内容のバリデーションは行わない

本APIでは、CSVファイルをアップロードしない。

そのため、以下のようなCSV内容に対するバリデーションは行わない。

- CSVファイルの存在確認
- ファイル形式
- ファイルサイズ
- 文字コード
- CSVヘッダー
- 対象年月
- 資産口座名
- 月末残高
- 行数
- 重複データ

これらの検証は、CSV-002 月末資産残高CSVプレビューで行う。

CSV-001は、CSVテンプレートを提供することに責務を限定する。

---

### 13 業務ルール

#### 13.1 CSVテンプレートの目的

本APIは、
月末資産残高CSVインポートで使用する
CSVテンプレートを提供する。

CSVテンプレートは、
CSV-002 月末資産残高CSVプレビューおよび
CSV-003 月末資産残高CSV登録で
使用可能な形式とする。

本APIでは、
CSVテンプレートの提供のみを行い、
月末資産残高の登録は行わない。

---

#### 13.2 CSVテンプレートの項目

月末資産残高CSVテンプレートには、
以下の項目を含める。

```text
対象年月
資産口座名
月末残高
```

CSVのヘッダー行は、
CSV-002およびCSV-003が
受け付けるCSV形式と一致させる。

概念例：

```csv
target_year_month,asset_account_name,balance
```

テンプレート取得後、
利用者が各行へ
インポート対象データを入力する。

---

#### 13.3 データ行

CSVテンプレートには、
利用者固有の
実データを含めない。

基本的には、
ヘッダー行のみを持つ
CSVファイルとして返却する。

概念例：

```csv
target_year_month,asset_account_name,balance
```

以下のような
登録済みデータを
テンプレートへ出力しない。

- 登録済み資産口座
- 登録済み月末資産残高
- 月末資産状況
- 利用可能資産設定

---

#### 13.4 対象年月

CSVテンプレートでは、
特定の対象年月を
あらかじめ設定しない。

利用者が
CSV作成時に
`target_year_month`を入力する。

入力値の形式や
1ファイル内で扱える対象年月のルールは、
CSV-002およびCSV-003で検証する。

CSV-001では、
対象年月の検証を行わない。

---

#### 13.5 資産口座

CSVテンプレートでは、
特定の資産口座を
あらかじめ設定しない。

利用者が
CSV作成時に
`asset_account_name`を入力する。

操作対象利用者に
入力された資産口座が存在するか、
対象年月時点で利用可能か、
残高記録単位が
口座単位であるかなどの判定は、
CSV-002およびCSV-003で行う。

---

#### 13.6 月末残高

CSVテンプレートでは、
月末残高を
あらかじめ設定しない。

利用者が
CSV作成時に
`balance`を入力する。

`balance`について、

- 必須であること
- 数値であること
- 整数であること
- 0円以上であること

などの検証は、
CSV-002およびCSV-003で行う。

CSV-001では、
月末残高の検証を行わない。

---

#### 13.7 利用者固有データを含めない

CSVテンプレートは、
すべての利用者で
同一の構造とする。

`X-User-Id`によって
テンプレートの内容を
変更しない。

例えば、
User AとUser Bが
本APIを実行した場合でも、
CSVテンプレートの
ヘッダー構造は同一とする。

```text
User A
    ↓
target_year_month,asset_account_name,balance

User B
    ↓
target_year_month,asset_account_name,balance
```

---

#### 13.8 IDをCSV入力項目としない

Phase1では、
月末資産残高CSVの
入力項目として
内部IDを使用しない。

そのため、
以下の項目を
CSVテンプレートへ含めない。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- `month_end_asset_balance_id`

資産口座の特定は、
CSV-002およびCSV-003において、
操作対象利用者と
`asset_account_name`を使用して行う。

---

#### 13.9 CSV-002・CSV-003との整合性

CSV-001で提供する
CSVテンプレートの仕様は、
CSV-002およびCSV-003が
受け付けるCSV仕様と
一致させる。

以下の状態を
発生させてはならない。

```text
CSV-001で取得したテンプレート
    ↓
正しい値を入力
    ↓
CSV-002で
ヘッダー形式不正になる
```

CSV項目を変更する場合は、
CSV-001、
CSV-002、
CSV-003を
同時に見直す。

---

#### 13.10 CSV形式

CSVテンプレートは、
CSVファイルとして返却する。

レスポンスの
`Content-Type`は、
以下とする。

```http
Content-Type: text/csv; charset=UTF-8
```

CSVの各項目は、
RFC 4180を基本として
適切にエスケープする。

Phase1では、
CSVテンプレートは
ヘッダー行のみであるため、
複雑なエスケープ処理を
必要としない。

---

#### 13.11 文字コード

CSVテンプレートの文字コードは、
UTF-8とする。

CSV-002およびCSV-003で
受け付ける文字コードと
統一する。

BOMの有無については、
CSV共通仕様として統一し、
CSV-001だけで
独自の扱いを定義しない。

---

#### 13.12 ファイル名

CSVテンプレートは、
ダウンロードファイルとして返却する。

ファイル名は、
用途を識別できる
固定名称とする。

概念例：

```text
month-end-asset-balances-template.csv
```

レスポンスヘッダーでは、
以下のように指定する。

```http
Content-Disposition: attachment; filename="month-end-asset-balances-template.csv"
```

日時や利用者IDなどを
ファイル名へ含める必要はない。

---

#### 13.13 データベースを更新しない

本APIは、
CSVテンプレートを
取得するだけのAPIである。

以下のテーブルに対する
INSERT、
UPDATE、
DELETEを行わない。

- `users`
- `asset_accounts`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `asset_account_available_settings`

また、
CSVテンプレート取得履歴を
業務テーブルへ保存しない。

---

#### 13.14 CSVプレビューを行わない

本APIでは、
CSVファイルを受け取らないため、
CSVプレビュー処理を行わない。

以下の処理は、
CSV-002の責務とする。

- CSVファイル読み込み
- CSVヘッダー検証
- CSV行検証
- 対象年月検証
- 資産口座確認
- 月末残高検証
- 重複確認
- 登録予定内容生成
- エラー内容生成

---

#### 13.15 CSV登録を行わない

本APIでは、
月末資産残高を登録しない。

以下の処理は、
CSV-003の責務とする。

- 月末資産状況の特定
- 月末資産残高の一括登録
- 重複登録判定
- 確定状態の確認
- トランザクション制御
- ロールバック

---

### 14 レスポンス

正常時は、
CSVファイルを
レスポンスボディとして返却する。

通常のJSON APIで使用する
成功レスポンスEnvelopeは
使用しない。

正常時の概念例：

```http
HTTP/1.1 200 OK
Content-Type: text/csv; charset=UTF-8
Content-Disposition: attachment; filename="month-end-asset-balances-template.csv"
```

レスポンスボディ：

```csv
target_year_month,asset_account_name,balance
```

CSVファイル自体が
レスポンスデータとなるため、
以下のようなJSON形式では返却しない。

```json
{
  "data": {
    "csv": "..."
  }
}
```

エラー時は、
API共通方針で定める
JSON形式の
エラーレスポンスを使用する。

---

#### 14.1 正常レスポンス

正常に
CSVテンプレートを生成できた場合は、

```text
200 OK
```

を返却する。

CSVテンプレートに
利用者固有の業務データが
存在しない場合でも、
正常にテンプレートを返却する。

---

#### 14.2 レスポンスヘッダー

正常時は、
主に以下の
レスポンスヘッダーを設定する。

| ヘッダー | 値 |
|---|---|
| `Content-Type` | `text/csv; charset=UTF-8` |
| `Content-Disposition` | `attachment; filename="month-end-asset-balances-template.csv"` |

API共通方針で
`X-Request-Id`等の
共通レスポンスヘッダーを
定義している場合は、
それらも付与する。

---

#### 14.3 レスポンスボディ

レスポンスボディには、
CSVテンプレートを返却する。

概念例：

```csv
target_year_month,asset_account_name,balance
```

テンプレートには、
ヘッダー行を必ず含める。

---

#### 14.4 JSON Envelopeを使用しない

本APIの正常レスポンスは、
ファイルダウンロードであるため、
通常のJSON APIで使用する

```json
{
  "data": {}
}
```

形式にはしない。

CSVファイルのバイナリまたは
文字列ストリームを
直接レスポンスとして返却する。

一方、
エラー時は
API共通エラーレスポンス形式を使用する。

そのため、
本APIは正常時と異常時で
レスポンスの`Content-Type`が異なる。

```text
正常時
    → text/csv

異常時
    → application/json
```

---

### 15 レスポンス項目

本APIの正常レスポンスは
JSONではなくCSVであるため、
通常のJSONレスポンス項目は存在しない。

CSVテンプレートには、
以下の3項目を含める。

| 項目 | 物理名 | 必須 | 内容 |
|---|---|:---:|---|
| 対象年月 | `target_year_month` | ○ | 月末資産残高の対象年月 |
| 資産口座名 | `asset_account_name` | ○ | 月末資産残高を登録する資産口座名 |
| 月末残高 | `balance` | ○ | 対象年月時点の月末残高 |

---

#### 15.1 target_year_month

月末資産残高の
対象年月を入力する項目である。

CSVテンプレートでは、
値を設定せず、
ヘッダーのみ提供する。

CSV-002およびCSV-003では、
`YYYY-MM`形式として扱う。

例：

```text
2026-07
```

---

#### 15.2 asset_account_name

月末資産残高を登録する
資産口座名を入力する項目である。

CSVテンプレートでは、
特定の資産口座名を
設定しない。

CSV-002およびCSV-003では、
操作対象利用者に属する
資産口座を特定するために使用する。

例：

```text
普通預金
```

---

#### 15.3 balance

対象年月時点の
月末残高を入力する項目である。

CSVテンプレートでは、
値を設定しない。

CSV-002およびCSV-003では、
日本円の整数として扱う。

例：

```text
1500000
```

---

#### 15.4 CSV入力例

利用者が
テンプレート取得後に
入力するCSVの概念例は、
以下とする。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,証券口座,800000
```

1つのCSVファイルでは、
1つの対象年月のみを扱う。

そのため、
以下のように
複数の対象年月を混在させない。

```csv
target_year_month,asset_account_name,balance
2026-06,普通預金,1400000
2026-07,証券口座,800000
```

このルールの検証は、
CSV-002およびCSV-003で行う。

---

#### 15.5 テンプレートに含めない項目

本APIのCSVテンプレートには、
以下の項目を含めない。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- `month_end_asset_balance_id`
- `confirmed`
- `balance_recording_unit`
- `created_at`
- `updated_at`

これらは、
CSV利用者が
直接入力する必要のない
内部管理情報である。

CSVインポートに必要な
最小限の入力項目のみを
テンプレートとして提供する。

---

### 16 エラーレスポンス

本APIでエラーが発生した場合は、
API共通方針で定義する
JSON形式のエラーレスポンスを返却する。

正常時は
`text/csv`でCSVファイルを返却するが、
エラー時は

```http
Content-Type: application/json
```

として返却する。

本APIでは、
CSVファイルのアップロードや
業務データの入力を行わないため、
CSV内容に起因する
バリデーションエラーは発生しない。

主なエラーは、
利用者コンテキストに関するものと、
サーバー内部エラーである。

---

#### 16.1 エラー一覧

本APIで想定する
主なエラーは、
以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`が指定されていない |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`の形式が不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 指定された利用者が存在しない、または論理削除済み |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | CSVテンプレート生成処理などで想定外のエラーが発生した |

---

#### 16.2 USER_CONTEXT_REQUIRED

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

リクエスト例：

```http
GET /api/v1/month-end-asset-balances/csv-template
```

この場合、
CSVテンプレートは返却しない。

---

#### 16.3 INVALID_USER_ID

`X-User-Id`の形式が
不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

例えば、
以下のような値を
不正とする。

```text
abc
0
-1
1.5
```

形式不正の場合、
利用者の存在確認や
CSVテンプレート生成処理へ
進まない。

---

#### 16.4 USER_NOT_FOUND

`X-User-Id`で指定された
利用者が存在しない場合、
または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

概念的には、
以下に該当する場合とする。

```text
users.id = X-User-Id
AND
users.deleted_at IS NULL
```

を満たす利用者が存在しない。

---

#### 16.5 INTERNAL_SERVER_ERROR

CSVテンプレート生成処理などで
想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

HTTPステータスは、

```text
500 Internal Server Error
```

とする。

Laravel内部の例外内容や
スタックトレースなどは、
レスポンスへ公開しない。

---

#### 16.6 CSV内容に関するエラー

本APIでは、
CSVファイルを受け取らない。

そのため、
以下のような
CSV内容に関するエラーは
本APIでは返却しない。

- CSVファイル未指定
- CSVファイル形式不正
- CSVヘッダー不正
- 文字コード不正
- ファイルサイズ超過
- 行数超過
- `target_year_month`不正
- `asset_account_name`不正
- `balance`不正
- 対象年月の混在
- 資産口座不存在
- 残高記録単位不一致
- CSV内重複

これらは、
CSV-002 月末資産残高CSVプレビュー、
またはCSV-003 月末資産残高CSV登録で扱う。

---

#### 16.7 エラー時のレスポンス形式

エラー時は、
CSVファイルではなく、
API共通エラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "指定された利用者が存在しません。"
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

具体的なEnvelope構造は、
API共通方針に従う。

---

#### 16.8 エラー時のContent-Type

正常時とエラー時では、
`Content-Type`が異なる。

```text
正常時
    → text/csv; charset=UTF-8

エラー時
    → application/json
```

フロントエンドでは、
HTTPステータスを確認した上で、
正常時のみCSVファイルとして扱う。

---

### 17 HTTPステータス

本APIで使用する
HTTPステータスは、
以下とする。

| HTTPステータス | 用途 |
|---|---|
| `200 OK` | CSVテンプレート取得成功 |
| `400 Bad Request` | 利用者コンテキスト未指定、利用者ID形式不正 |
| `404 Not Found` | 指定された利用者が存在しない |
| `500 Internal Server Error` | 想定外のサーバー内部エラー |

---

#### 17.1 200 OK

CSVテンプレートを
正常に生成できた場合は、

```text
200 OK
```

を返却する。

レスポンスボディには、
CSVテンプレートを返却する。

```http
HTTP/1.1 200 OK
Content-Type: text/csv; charset=UTF-8
Content-Disposition: attachment; filename="month-end-asset-balances-template.csv"
```

---

#### 17.2 400 Bad Request

リクエストとして
必要な利用者コンテキストを
正しく特定できない場合に使用する。

対象となる主なエラーコードは、
以下とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
```

---

#### 17.3 404 Not Found

指定された利用者が
存在しない場合、
または論理削除済みの場合に使用する。

対象となるエラーコードは、

```text
USER_NOT_FOUND
```

とする。

---

#### 17.4 422 Unprocessable Entityを使用しない

本APIでは、
パスパラメータ、
クエリパラメータ、
リクエストボディによる
業務入力値を受け取らない。

そのため、
CSVテンプレート取得処理において

```text
422 Unprocessable Entity
```

は使用しない。

CSV内容に関する
バリデーションエラーは、
CSV-002およびCSV-003で扱う。

---

#### 17.5 500 Internal Server Error

想定外のサーバー内部エラーが
発生した場合に使用する。

対象となるエラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

---

### 18 副作用

本APIには、
業務データに対する
副作用はない。

CSVテンプレートを
生成して返却するだけであり、
データベースに対する

```text
INSERT
UPDATE
DELETE
```

を行わない。

本API実行によって、
以下の状態を変更しない。

- 利用者
- 資産口座
- 月末資産状況
- 月末資産残高
- 利用可能資産設定
- 確定状態

また、
CSVテンプレートの
取得回数や取得日時を
業務テーブルへ保存しない。

アクセスログや
監視ログなど、
API共通基盤による
技術的なログ出力は
副作用として扱わない。

---

### 19 冪等性

本APIは、
冪等である。

同一のリクエストを
複数回実行しても、
業務データの状態は変化しない。

例えば、

```http
GET /api/v1/month-end-asset-balances/csv-template
X-User-Id: 1
```

を複数回実行しても、
月末資産残高や
その他の業務データは変更されない。

また、
CSVテンプレートの仕様が
変更されていない限り、
同一内容のテンプレートを返却する。

---

#### 19.1 Idempotency-Key

本APIでは、
`Idempotency-Key`を使用しない。

本APIは
読み取り専用のGET APIであり、
複数回実行しても
重複登録などの問題が発生しないためである。

---

#### 19.2 再実行

ネットワークエラーなどによって
クライアント側で
処理結果を確認できなかった場合でも、
本APIは安全に再実行できる。

```text
CSVテンプレート取得
    ↓
通信エラー
    ↓
再実行
    ↓
CSVテンプレート取得
```

再実行によって、
業務データの重複登録や
状態変更は発生しない。

---

#### 19.3 テンプレート仕様変更時

将来的に
CSVテンプレートの仕様が変更された場合、
同一エンドポイントから
返却される内容が
変更される可能性がある。

これは、
本APIの冪等性を損なうものではない。

冪等性は、
同一のシステム状態において
同一リクエストを複数回実行しても、
サーバー側の業務状態に
追加の変化を発生させないことを意味する。

---

### 20 関連テーブル

本APIは、
固定のCSVテンプレートを生成して返却するAPIであり、
業務データを取得するための
データベース検索は行わない。

そのため、
CSVテンプレート生成処理における
直接の関連テーブルは存在しない。

ただし、
`X-User-Id`から
操作対象利用者を特定する
API共通の利用者コンテキスト確認では、
以下のテーブルを参照する。

| テーブル | 用途 |
|---|---|
| `users` | 操作対象となる利用者の存在確認 |

CSV-002およびCSV-003では、
CSV内容の検証・登録のために
月末資産関連テーブルを使用するが、
CSV-001では参照しない。

---

#### 20.1 users

`X-User-Id`で指定された
利用者が存在することを確認する。

概念的な確認条件は、
以下とする。

```text
users.id = X-User-Id
AND
users.deleted_at IS NULL
```

本APIでは、利用者の存在確認のみを行い、`users`を更新しない。

---

#### 20.2 参照しないテーブル

CSVテンプレートは、利用者固有の業務データを含まない。

そのため、以下のテーブルはCSV-001では参照しない。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_account_available_settings`

例えば、`asset_accounts`から操作対象利用者の資産口座名を取得し、テンプレートへ出力する処理は行わない。

---

#### 20.3 更新対象テーブル

本APIでは、更新対象テーブルは存在しない。

以下のデータベース操作を行わない。

```text
INSERT
UPDATE
DELETE
```

CSVテンプレート取得履歴を業務テーブルへ保存することも行わない。

---

### 21 排他制御

本APIでは、排他制御を行わない。

CSVテンプレートを生成して返却するだけであり、業務データの更新を伴わないためである。

以下のような行ロックは使用しない。

```sql
SELECT ... FOR UPDATE
```

Laravelにおいても、以下を使用しない。

```php
lockForUpdate()
```

---

#### 21.1 同時実行

同一利用者または複数利用者が本APIを同時に実行しても、業務データの競合は発生しない。

```text
User A
    ↓
CSV-001

User B
    ↓
CSV-001

同時実行可能
```

CSVテンプレートは利用者固有の状態を持たないため、同時実行を制限しない。

---

#### 21.2 CSV登録APIとの同時実行

CSV-002、CSV-003、その他の月末資産関連APIが同時に実行されていても、CSV-001では排他制御を行わない。

例えば、

```text
CSV-001
テンプレート取得

CSV-003
月末資産残高CSV登録
```

が同時に実行されても、CSV-001は業務データを参照・更新しないため、相互にロックする必要はない。

---

### 22 トランザクション

本APIでは、業務データの更新を行わないため、明示的なトランザクションを使用しない。

Laravelでは、以下のような処理を行わない。

```php
DB::transaction(
    function () {
        // CSVテンプレート生成
    },
);
```

CSVテンプレート生成処理は、トランザクションの対象外とする。

---

#### 22.1 利用者存在確認

`users`の存在確認についても、読み取りのみであるため、明示的なトランザクションを開始しない。

API共通Middlewareなどで利用者コンテキストを確認した後、CSVテンプレート生成処理へ進む。

---

#### 22.2 CSV-003との違い

CSV-003 月末資産残高CSV登録では、複数行を一括登録するため、トランザクションが必要となる。

一方、CSV-001ではデータベース更新を行わないため、トランザクションは不要である。

```text
CSV-001
    → トランザクション不要

CSV-003
    → トランザクション必要
```

---

### 23 性能・実装上の注意

#### 23.1 データベース検索

CSVテンプレート生成処理では、業務データの検索を行わない。

以下のような処理は行わない。

```text
asset_accounts検索
    ↓
資産口座名を取得
    ↓
CSVへ出力
```

テンプレートの内容は、固定のCSV仕様から生成する。

---

#### 23.2 N+1問題

本APIでは、資産口座や月末資産残高などの一覧取得を行わないため、N+1問題は発生しない。

利用者存在確認についても、API共通の利用者コンテキスト確認で1件を確認するだけとする。

---

#### 23.3 CSV生成

CSV文字列は、CSV仕様を定義した専用クラスなどから生成する。

概念的には、以下のような構造とする。

```text
CSVテンプレート定義
    ↓
CSV生成
    ↓
HTTPレスポンス
```

ControllerへCSVヘッダー定義を直接ハードコードすることは避ける。

---

#### 23.4 CSV仕様の共通化

CSV-001で生成するCSVテンプレートのヘッダーと、CSV-002およびCSV-003で検証するCSVヘッダーは、同一の仕様を参照する。

例えば、CSV項目を定数として管理してよい。

概念例：

```php
final class MonthEndAssetBalanceCsvDefinition
{
    public const HEADERS = [
        'target_year_month',
        'asset_account_name',
        'balance',
    ];
}
```

CSV-001では、

```php
MonthEndAssetBalanceCsvDefinition::HEADERS
```

からテンプレートを生成する。

CSV-002およびCSV-003でも、同じ定義を使用してCSVヘッダーを検証する。

これにより、テンプレートとインポート仕様の不整合を防止する。

---

#### 23.5 CSVストリーム

Phase1のCSVテンプレートはヘッダー行のみであり、データ量は非常に小さい。

そのため、性能上は文字列を生成してそのまま返却しても問題ない。

ただし、LaravelのCSVダウンロード実装としてストリームレスポンスを使用してもよい。

概念例：

```php
response()->streamDownload(
    function (): void {
        // CSV出力
    },
    'month-end-asset-balances-template.csv',
    [
        'Content-Type' =>
            'text/csv; charset=UTF-8',
    ],
);
```

具体的な実装方式は、Laravel実装方針で定める。

---

#### 23.6 文字コード

CSVテンプレートは、CSV-002およびCSV-003が受け付ける文字コードと統一する。

Phase1では、UTF-8を使用する。

CSV-001だけ異なる文字コードでテンプレートを生成してはならない。

---

#### 23.7 改行コード

CSVの改行コードについても、CSV関連APIで統一する。

OSや実行環境によって意図せず形式が変化しないよう、CSV生成処理側で明示的に管理してよい。

---

#### 23.8 Content-Disposition

ブラウザからCSVファイルとして保存できるように、`Content-Disposition`を設定する。

```http
Content-Disposition: attachment; filename="month-end-asset-balances-template.csv"
```

利用者入力値をファイル名へ使用しないため、ファイル名に対するサニタイズ処理は不要とする。

---

#### 23.9 キャッシュ

Phase1では、CSVテンプレート専用のアプリケーションキャッシュを使用しない。

CSVテンプレートは非常に小さく、生成コストも低いため、キャッシュによる複雑性を追加する必要はない。

将来的に必要となった場合は、HTTPキャッシュを含めて別途検討する。

---

#### 23.10 CSV仕様変更

将来的に月末資産残高CSVの入力項目を変更する場合は、少なくとも以下を同時に確認する。

- CSV-001 テンプレート生成
- CSV-002 プレビュー
- CSV-003 登録
- CSV共通定義
- フロントエンドのCSVインポート画面
- テストケース

CSV-001だけを変更し、CSV-002またはCSV-003と仕様が異なる状態にしてはならない。

---

### 24 テスト観点

本APIでは、CSVテンプレートがCSV-002およびCSV-003で使用可能な形式として正しく取得できることを確認する。

---

#### 24.1 正常系

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

#### 24.2 CSVヘッダー

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

#### 24.3 利用者固有データ

CSVテンプレートに操作対象利用者の実データが含まれないことを確認する。

例えば、利用者に

```text
普通預金
証券口座
```

などの資産口座が登録されていても、テンプレートには出力されないことを確認する。

---

#### 24.4 利用者によるテンプレート差異

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

#### 24.5 X-User-Id未指定

`X-User-Id`を指定しない場合は、

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

となることを確認する。

エラー時にCSVファイルが返却されないことも確認する。

---

#### 24.6 X-User-Id形式不正

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

#### 24.7 利用者不存在

存在しない利用者IDを指定した場合は、

```text
404 Not Found
USER_NOT_FOUND
```

となることを確認する。

---

#### 24.8 論理削除済み利用者

`users.deleted_at`が設定されている利用者を指定した場合は、

```text
404 Not Found
USER_NOT_FOUND
```

となることを確認する。

---

#### 24.9 エラー時のレスポンス形式

エラー時は、CSVではなくAPI共通のJSONエラーレスポンスが返却されることを確認する。

```text
正常時
    → text/csv

異常時
    → application/json
```

となることを確認する。

---

#### 24.10 データベース非更新

CSV-001実行前後で、以下の業務テーブルにレコード追加・更新・削除が発生しないことを確認する。

- `users`
- `asset_accounts`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_account_available_settings`

---

#### 24.11 業務データ非参照

CSVテンプレート生成処理が、以下の業務データに依存しないことを確認する。

- 資産口座の登録件数
- 月末資産残高の登録件数
- 月末資産状況の有無
- 月末資産状況の確定状態
- 利用可能資産設定

これらの状態が異なっていても、同一のCSVテンプレートが取得できることを確認する。

---

#### 24.12 CSV-002・CSV-003との整合性

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

#### 24.13 文字コード

CSVテンプレートがCSV共通仕様で定めるUTF-8として正しく取得できることを確認する。

BOMを使用する仕様とした場合は、BOMの有無についてもテストする。

---

#### 24.14 改行コード

CSV共通仕様で改行コードを固定する場合は、期待する改行コードで出力されることを確認する。

環境によって意図せず改行コードが変化しないことを確認する。

---

#### 24.15 冪等性

同一の`X-User-Id`で複数回実行しても、業務データが変更されないことを確認する。

また、同一のシステム状態では同一のCSVテンプレートが取得できることを確認する。

---

#### 24.16 同時実行

同一利用者または複数利用者からCSV-001を同時実行しても、正常にCSVテンプレートを取得できることを確認する。

他のCSV登録処理や月末資産関連APIが実行中であっても、CSV-001が不要なロックによってブロックされないことを確認する。

---

#### 24.17 想定外例外

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

#### 24.18 主なテストケース

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

### 25 Laravel実装方針

### 25 Laravel実装方針

CSV-001では、
Action、
UseCase、
CSV Definition、
CSV Generator、
Responderを分離して実装する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ↓
CSV Definition
    ↓
CSV Generator
    ↓
Responder
    ↓
CSVダウンロードレスポンス
```

本APIでは、利用者固有の業務データをCSVテンプレートへ出力しない。

そのため、CSVテンプレート生成のためのRepositoryまたはQueryクラスは使用しない。

また、以下を使用しないため、CSV-001専用のFormRequestも作成しない。

- パスパラメータ
- クエリパラメータ
- リクエストボディ

---

#### 25.1 Action

HTTPリクエストを受け付け、CSVテンプレート取得UseCaseを呼び出し、生成結果をResponderへ渡す。

概念例：

```php
final class DownloadMonthEndAssetBalanceCsvTemplateAction
{
    public function __invoke(
        DownloadMonthEndAssetBalanceCsvTemplateUseCase $useCase,
        MonthEndAssetBalanceCsvTemplateResponder $responder,
    ): StreamedResponse {
        $template =
            $useCase->execute();

        return $responder->download(
            $template,
        );
    }
}
```

Actionでは、以下を行わない。

- CSVヘッダーの定義
- CSV文字列の生成
- CSVエスケープ処理
- 文字コード変換
- BOM付与
- ファイル名の決定
- `Content-Type`の決定
- `Content-Disposition`の生成
- 利用者存在確認
- データベース検索

Actionは、UseCaseの呼び出しとResponderへの受け渡しに責務を限定する。

---

#### 25.2 FormRequest

本APIでは、専用のFormRequestを作成しない。

本APIは、

```text
パスパラメータなし
クエリパラメータなし
リクエストボディなし
```

であるため、CSV-001固有の入力バリデーションが存在しない。

`X-User-Id`の検証は、API共通Middlewareで行う。

そのため、以下のような空のFormRequestは作成しない。

```php
final class DownloadMonthEndAssetBalanceCsvTemplateRequest
    extends FormRequest
{
}
```

不要なクラスを追加しない。

---

#### 25.3 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証し、操作対象利用者を特定する。

概念的な処理は、以下とする。

```text
X-User-Id取得
    ↓
必須チェック
    ↓
形式チェック
    ↓
users存在確認
    ↓
利用者コンテキスト設定
    ↓
Action
```

利用者の存在確認条件は、以下とする。

```text
users.id = X-User-Id
AND
users.deleted_at IS NULL
```

Action以降では、利用者存在確認を再実装しない。

---

#### 25.4 UseCase

CSVテンプレート取得のユースケース処理を担当する。

概念例：

```php
final class DownloadMonthEndAssetBalanceCsvTemplateUseCase
{
    public function __construct(
        private readonly
        MonthEndAssetBalanceCsvGenerator $csvGenerator,
    ) {
    }

    public function execute():
        MonthEndAssetBalanceCsvTemplate
    {
        return new MonthEndAssetBalanceCsvTemplate(
            content:
                $this->csvGenerator
                    ->generateTemplate(),

            fileName:
                MonthEndAssetBalanceCsvDefinition::FILE_NAME,
        );
    }
}
```

CSVテンプレートの内容は、利用者によって変化しない。

そのため、UseCaseへ`userId`を渡す必要はない。

利用者コンテキストの確認自体は、Action到達前にMiddlewareで完了していることを前提とする。

---

#### 25.5 UseCaseで行わないこと

UseCaseでは、以下を行わない。

- `users`の再検索
- `asset_accounts`の検索
- `holding_assets`の検索
- `month_end_asset_snapshots`の検索
- `month_end_asset_balances`の検索
- `month_end_holding_values`の検索
- `asset_account_available_settings`の検索
- 利用者固有データのCSV出力
- 月末資産残高の登録
- 月末資産残高の更新
- 月末資産残高の削除

CSVテンプレートの生成に必要な処理だけを行う。

---

#### 25.6 CSV Template DTO

CSV生成結果を単純な文字列だけでResponderへ渡すのではなく、必要に応じて専用DTOとして表現する。

概念例：

```php
final readonly class MonthEndAssetBalanceCsvTemplate
{
    public function __construct(
        public string $content,
        public string $fileName,
    ) {
    }
}
```

これにより、UseCaseからResponderへ渡すCSVテンプレート情報を明示的に表現できる。

将来的に、以下の情報をテンプレート単位で保持する必要が生じた場合も拡張しやすくなる。

- 文字コード
- BOM有無
- MIME Type

---

#### 25.7 CSV Definition

CSV-001、CSV-002、CSV-003で使用する月末資産残高CSV仕様は、共通Definitionへ集約する。

概念例：

```php
final class MonthEndAssetBalanceCsvDefinition
{
    public const HEADERS = [
        'target_year_month',
        'asset_account_name',
        'balance',
    ];

    public const FILE_NAME =
        'month-end-asset-balances-template.csv';
}
```

CSV-001だけで独自のヘッダーを定義しない。

CSV-002およびCSV-003でも、同じDefinitionを使用する。

---

#### 25.8 CSVヘッダー

CSVテンプレートでは、以下のヘッダーをこの順序で出力する。

```text
target_year_month
asset_account_name
balance
```

生成結果は、以下とする。

```csv
target_year_month,asset_account_name,balance
```

ヘッダー名だけでなく、ヘッダー順序もCSV仕様の一部として扱う。

---

#### 25.9 CSV Generator

CSV文字列の生成は、専用Generatorで行う。

概念例：

```php
final class MonthEndAssetBalanceCsvGenerator
{
    public function generateTemplate(): string
    {
        $stream =
            fopen(
                'php://temp',
                'r+',
            );

        if ($stream === false) {
            throw new CsvGenerationException();
        }

        try {
            fputcsv(
                $stream,
                MonthEndAssetBalanceCsvDefinition::HEADERS,
            );

            rewind($stream);

            $csv =
                stream_get_contents(
                    $stream,
                );

            if ($csv === false) {
                throw new CsvGenerationException();
            }

            return $csv;
        } finally {
            fclose(
                $stream,
            );
        }
    }
}
```

Generatorは、CSV形式の生成に責務を限定する。

以下を行わない。

- HTTPレスポンス生成
- 利用者検索
- 資産口座検索
- 業務データ取得
- 月末資産残高登録

---

#### 25.10 fputcsvの利用

CSV生成では、PHP標準の`fputcsv()`を使用する。

以下のような手動連結は基本的に行わない。

```php
implode(
    ',',
    MonthEndAssetBalanceCsvDefinition::HEADERS,
);
```

`fputcsv()`を使用することで、将来的にCSV項目へ以下を含む場合でも、CSVとして適切にエスケープできる。

```text
,
"
改行
```

---

#### 25.11 fopen失敗時

`php://temp`のオープンに失敗した場合は、そのまま処理を続行しない。

概念例：

```php
$stream =
    fopen(
        'php://temp',
        'r+',
    );

if ($stream === false) {
    throw new CsvGenerationException();
}
```

CSV生成失敗は、最終的にAPI共通Exception Handlerによって

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

---

#### 25.12 fputcsv失敗時

`fputcsv()`の戻り値も必要に応じて確認する。

概念例：

```php
$result =
    fputcsv(
        $stream,
        MonthEndAssetBalanceCsvDefinition::HEADERS,
    );

if ($result === false) {
    throw new CsvGenerationException();
}
```

CSV生成処理の失敗を正常な空CSVとして扱ってはならない。

---

#### 25.13 リソース解放

CSV Generatorでストリームを開いた場合は、処理結果にかかわらず確実に`fclose()`する。

概念的には、

```php
try {
    // CSV生成
} finally {
    fclose(
        $stream,
    );
}
```

とする。

---

#### 25.14 文字コード

CSVテンプレートは、UTF-8で生成する。

CSV-002およびCSV-003で受け付ける文字コードと統一する。

CSV-001だけ異なる文字コードへ変換しない。

---

#### 25.15 BOM

UTF-8 BOMを付与するかどうかは、CSV共通仕様として統一する。

BOMを付与する場合は、CSV Generator内で明示的に行う。

概念例：

```php
fwrite(
    $stream,
    "\xEF\xBB\xBF",
);
```

CSV-001ではBOMあり、CSV-002・CSV-003ではBOM非対応、という不整合を発生させてはならない。

---

#### 25.16 改行コード

CSVテンプレートの改行コードは、CSV関連APIで統一する。

CSV-001、CSV-002、CSV-003で異なる改行コード仕様を個別定義しない。

実行環境による差異を許容しない仕様とする場合は、CSV Generator側で明示的に制御する。

---

#### 25.17 データ行

CSV-001では、利用者固有のデータ行を出力しない。

生成対象は、ヘッダー行のみとする。

以下のような資産口座一覧取得処理を行わない。

```php
$assetAccounts =
    AssetAccount::query()
        ->where(
            'user_id',
            $userId,
        )
        ->get();
```

また、以下のようなデータ行生成も行わない。

```php
foreach (
    $assetAccounts
    as $assetAccount
) {
    fputcsv(
        $stream,
        [
            '',
            $assetAccount->name,
            '',
        ],
    );
}
```

CSVテンプレートは、業務データから独立した固定構造とする。

---

#### 25.18 Repository・Query

CSV-001では、RepositoryおよびQueryクラスを使用しない。

本APIで必要となるデータベース参照は、共通Middlewareによる`users`の存在確認のみである。

そのため、以下のようなクラスはCSV-001のためには作成しない。

```text
MonthEndAssetBalanceCsvTemplateQuery
AssetAccountQuery
MonthEndAssetSnapshotQuery
```

不要な抽象化を追加しない。

---

#### 25.19 API Resource

本APIの正常レスポンスは、JSONではなくCSVファイルである。

そのため、正常レスポンス用のLaravel API Resourceは使用しない。

以下のようなResourceは作成しない。

```php
final class MonthEndAssetBalanceCsvTemplateResource
    extends JsonResource
{
}
```

CSVファイルを直接HTTPレスポンスとして返却する。

---

#### 25.20 Responder

CSVダウンロードレスポンスの生成は、専用Responderで行う。

概念例：

```php
final class MonthEndAssetBalanceCsvTemplateResponder
{
    public function download(
        MonthEndAssetBalanceCsvTemplate $template,
    ): StreamedResponse {
        return response()->streamDownload(
            static function () use (
                $template,
            ): void {
                echo $template->content;
            },
            $template->fileName,
            [
                'Content-Type'
                    => 'text/csv; charset=UTF-8',
            ],
        );
    }
}
```

Responderでは、CSV内容そのものを生成しない。

---

#### 25.21 Responderの責務

Responderは、生成済みCSVテンプレートをHTTPレスポンスへ変換することに責務を限定する。

主に以下を扱う。

```text
HTTPステータス
Content-Type
Content-Disposition
レスポンスボディ
```

Responderでは、以下を行わない。

- データベース検索
- 利用者境界判定
- CSVヘッダー定義
- CSV生成
- CSV項目検証
- CSV内容の業務判定
- 月末資産残高登録

---

#### 25.22 Content-Type

正常時の`Content-Type`は、以下とする。

```http
Content-Type: text/csv; charset=UTF-8
```

CSVファイルをJSONとして返却しない。

---

#### 25.23 Content-Disposition

CSVファイルとしてダウンロードできるよう、`Content-Disposition`を設定する。

概念的には、以下とする。

```http
Content-Disposition: attachment; filename="month-end-asset-balances-template.csv"
```

ファイル名は、CSV Definitionで一元管理する。

ActionやResponderへ文字列リテラルとして重複定義しない。

---

#### 25.24 正常時のJSON Envelope

正常時は、API共通のJSON成功Envelopeを使用しない。

以下のような形式にはしない。

```json
{
  "data": {
    "content": "target_year_month,asset_account_name,balance"
  }
}
```

CSVファイルを直接レスポンスとして返却する。

---

#### 25.25 エラー時のレスポンス

エラー時は、CSVではなくAPI共通のJSONエラーレスポンスを使用する。

```text
正常時
    → text/csv

エラー時
    → application/json
```

MiddlewareまたはException Handlerで発生した例外は、API共通方針に従ってJSONへ変換する。

---

#### 25.26 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| CSV生成処理失敗 | `INTERNAL_SERVER_ERROR` |
| その他想定外例外 | `INTERNAL_SERVER_ERROR` |

CSV-001では、CSV内容に関する業務例外は発生しない。

---

#### 25.27 想定外例外

CSV生成処理などで想定外の例外が発生した場合は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- スタックトレース
- Laravel内部例外メッセージ
- PHP内部エラー
- ファイルパス
- クラス内部情報
- SQL
- データベース接続情報

詳細情報は、サーバーログへ記録する。

---

#### 25.28 トランザクション

CSV-001では、明示的な`DB::transaction()`を使用しない。

以下のような実装は行わない。

```php
DB::transaction(
    function () use (
        $useCase,
    ) {
        return $useCase->execute();
    },
);
```

本APIでは、業務データを更新しないため、トランザクションは不要である。

---

#### 25.29 ロック

CSV-001では、行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

CSVテンプレート取得によって、CSV登録処理や月末資産関連処理を不要にブロックしない。

---

#### 25.30 キャッシュ

Phase1では、CSVテンプレート専用のサーバー側アプリケーションキャッシュを使用しない。

CSVテンプレートはヘッダー行のみであり、生成コストが非常に小さい。

また、CSV仕様変更後に古いテンプレートがキャッシュされ続ける複雑性を避ける。

必要になった場合は、将来的にHTTPキャッシュなどを別途検討する。

---

#### 25.31 CSV-002・CSV-003との共通化

CSV-001、CSV-002、CSV-003では、同一の

```text
MonthEndAssetBalanceCsvDefinition
```

を使用する。

概念的には、以下とする。

```text
CSV-001
テンプレート生成
    ↓
MonthEndAssetBalanceCsvDefinition

CSV-002
ヘッダー検証
    ↓
MonthEndAssetBalanceCsvDefinition

CSV-003
ヘッダー検証
    ↓
MonthEndAssetBalanceCsvDefinition
```

CSVヘッダーを各APIへ重複定義しない。

---

#### 25.32 CSV Generatorの共通化範囲

CSV Generatorは、CSV-001のテンプレート生成を担当する。

CSV-002およびCSV-003は、CSVファイルを読み込む側であるため、Generator自体を共通利用する必要はない。

共通化する対象は、主に以下とする。

```text
ヘッダー
文字コード
BOM方針
改行コード
CSV仕様
```

ParserとGeneratorを無理に1クラスへ統合しない。

---

#### 25.33 ログ

CSV-001では、API共通のアクセスログを使用する。

必要に応じて、以下の情報をログコンテキストへ設定する。

```text
requestId
userId
apiId
```

`apiId`は、

```text
CSV-001
```

とする。

CSVテンプレート内容をログへ出力する必要はない。

---

#### 25.34 テスト実装方針

Laravel側では、Feature Testを中心としてCSV-001のAPI契約を確認する。

主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `404 Not Found`
- `500 Internal Server Error`
- `X-User-Id`必須
- `X-User-Id`形式検証
- 利用者存在確認
- 論理削除済み利用者の除外
- `Content-Type`
- `Content-Disposition`
- ファイル名
- CSVヘッダー
- CSVヘッダー順序
- データ行が存在しないこと
- 利用者固有データが含まれないこと
- JSON成功Envelopeを使用しないこと
- 業務データが更新されないこと
- 冪等性

---

#### 25.35 CSV DefinitionのUnit Test

CSV Definitionについて、以下のヘッダーが正しい順序で定義されていることを確認する。

```php
[
    'target_year_month',
    'asset_account_name',
    'balance',
]
```

ファイル名についても、

```text
month-end-asset-balances-template.csv
```

となることを確認する。

---

#### 25.36 CSV GeneratorのUnit Test

CSV Generatorについて、生成結果がCSV共通仕様を満たすことを確認する。

主に以下を確認する。

- ヘッダーが存在すること
- ヘッダー順序が正しいこと
- 不要なデータ行が存在しないこと
- UTF-8で生成されること
- BOMが仕様どおりであること
- 改行コードが仕様どおりであること
- 正常にCSV形式として生成されること

概念的な期待値は、以下とする。

```csv
target_year_month,asset_account_name,balance
```

---

#### 25.37 CSV Generator異常系のUnit Test

CSV Generatorでストリーム生成やCSV出力が失敗した場合に、正常な空CSVを返却せず例外となることを確認する。

概念的には、

```text
CSV生成失敗
    ↓
CsvGenerationException
    ↓
共通Exception Handler
    ↓
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

となることを確認する。

---

#### 25.38 UseCaseのUnit Test

UseCaseについて、CSV Generatorの生成結果をCSV Template DTOとして返却できることを確認する。

概念例：

```text
MonthEndAssetBalanceCsvGenerator
    ↓
CSV文字列

MonthEndAssetBalanceCsvDefinition
    ↓
ファイル名

DownloadMonthEndAssetBalanceCsvTemplateUseCase
    ↓
MonthEndAssetBalanceCsvTemplate
```

UseCase実行時に、データベース検索が必要ない構造となっていることも確認する。

---

#### 25.39 ResponderのTest

Responderについて、生成済みCSV Template DTOから正しいHTTPレスポンスを生成できることを確認する。

主に以下を確認する。

```text
HTTP 200
Content-Type
Content-Disposition
ファイル名
CSVレスポンスボディ
```

正常時に

```text
application/json
```

とならないことも確認する。

---

#### 25.40 CSV-002・CSV-003との整合性テスト

CSV-001、CSV-002、CSV-003で同一のCSV Definitionを使用することを基本とする。

少なくとも、以下の整合性を自動テストで確認する。

```text
CSV-001
生成ヘッダー
    =
MonthEndAssetBalanceCsvDefinition::HEADERS
```

```text
CSV-002
期待ヘッダー
    =
MonthEndAssetBalanceCsvDefinition::HEADERS
```

```text
CSV-003
期待ヘッダー
    =
MonthEndAssetBalanceCsvDefinition::HEADERS
```

これにより、公式テンプレートを使用したCSVがCSV-002またはCSV-003でヘッダー不正になることを防止する。

---

### 26 React・TypeScriptでの利用

CSV-001は、
月末資産残高CSVテンプレートを
CSVファイルとして取得するAPIである。

正常時は、
JSONレスポンスではなく、
`text/csv`の
ファイルレスポンスを返却する。

そのため、
通常のJSON APIとは
異なる扱いとする。

API呼び出し例は、
以下とする。

```ts
export const downloadMonthEndAssetBalanceCsvTemplate =
  async (): Promise<Blob> => {
    const response =
      await apiClient.get(
        '/api/v1/month-end-asset-balances/csv-template',
        {
          responseType: 'blob',
        },
      );

    return response.data;
  };
```

取得したBlobを使用して、
ブラウザからCSVファイルを
保存する。

---

#### 26.1 レスポンス型

CSV-001の正常レスポンスは、
JSONではない。

そのため、
以下のような
JSONレスポンス型は定義しない。

```ts
type CsvTemplateResponse = {
  data: {
    csv: string;
  };
};
```

正常時は、
`Blob`として扱う。

```ts
type MonthEndAssetBalanceCsvTemplate =
  Blob;
```

---

#### 26.2 responseType

Axios等のAPI Clientを
使用する場合は、
`responseType`に

```text
blob
```

を指定する。

概念例：

```ts
const response =
  await apiClient.get(
    '/api/v1/month-end-asset-balances/csv-template',
    {
      responseType: 'blob',
    },
  );
```

CSVレスポンスを
JSONとして解析しない。

---

#### 26.3 ファイル保存

取得したBlobから、
ブラウザ上で
ダウンロード処理を行う。

概念例：

```ts
export const saveCsvFile =
  (
    blob: Blob,
    fileName: string,
  ): void => {
    const url =
      URL.createObjectURL(
        blob,
      );

    const anchor =
      document.createElement(
        'a',
      );

    anchor.href = url;
    anchor.download = fileName;

    document.body.appendChild(
      anchor,
    );

    anchor.click();

    anchor.remove();

    URL.revokeObjectURL(
      url,
    );
  };
```

CSV-001では、
例えば以下の
ファイル名を使用する。

```text
month-end-asset-balances-template.csv
```

---

#### 26.4 Content-Dispositionの利用

可能であれば、
レスポンスの
`Content-Disposition`から
ファイル名を取得する。

概念例：

```ts
const contentDisposition =
  response.headers[
    'content-disposition'
  ];
```

ただし、
Phase1では
CSVテンプレートのファイル名は
固定であるため、
フロントエンド共通定数として
保持してもよい。

概念例：

```ts
export const
  MONTH_END_ASSET_BALANCE_CSV_TEMPLATE_FILE_NAME =
    'month-end-asset-balances-template.csv';
```

バックエンドと
フロントエンドで
ファイル名を二重管理する場合は、
変更時の整合性に注意する。

---

#### 26.5 Content-Typeの確認

正常時の
`Content-Type`は、

```text
text/csv; charset=UTF-8
```

を想定する。

ただし、
通常の画面処理では
`200 OK`かどうかを確認したうえで、
取得したBlobを
CSVファイルとして扱えばよい。

---

#### 26.6 ダウンロードボタン

CSVインポート画面では、
テンプレート取得用の
ボタンを配置する。

概念例：

```tsx
<button
  type="button"
  onClick={
    handleDownloadTemplate
  }
>
  CSVテンプレートを取得
</button>
```

クリック時に
CSV-001を実行する。

---

#### 26.7 ダウンロード処理例

概念例：

```ts
const handleDownloadTemplate =
  async (): Promise<void> => {
    const blob =
      await downloadMonthEndAssetBalanceCsvTemplate();

    saveCsvFile(
      blob,
      'month-end-asset-balances-template.csv',
    );
  };
```

実際のエラー処理や
ローディング状態管理は、
フロントエンド共通設計に従う。

---

#### 26.8 Mutationとして扱うか

CSV-001は、
HTTPメソッドとしては
GETであり、
業務データを変更しない。

React Queryを使用する場合、
通常のデータ取得Queryとして
扱うこともできる。

ただし、
CSVテンプレート取得は
利用者のボタン操作によって
都度ファイルを取得する
命令的な処理である。

そのため、
Phase1では
必ずしもReact Queryへ
載せる必要はない。

例えば、
API Clientを
直接呼び出してよい。

```ts
const blob =
  await downloadMonthEndAssetBalanceCsvTemplate();
```

---

#### 26.9 キャッシュ

CSVテンプレートは、
固定内容であるが、
Phase1では
ブラウザ側で
積極的にキャッシュする必要はない。

利用者が
テンプレート取得ボタンを押した時点で、
APIから取得する。

CSVテンプレートの仕様変更後に
古いファイルを
使い続ける可能性を減らすためにも、
明示的な長期キャッシュは行わない。

---

#### 26.10 ローディング状態

テンプレート取得中は、
ボタンを一時的に
無効化してよい。

概念例：

```tsx
<button
  type="button"
  disabled={isDownloading}
  onClick={
    handleDownloadTemplate
  }
>
  {isDownloading
    ? '取得中...'
    : 'CSVテンプレートを取得'}
</button>
```

これにより、
短時間に
同じダウンロード処理を
何度も実行することを防ぐ。

---

#### 26.11 複数回取得

CSV-001は
冪等なGET APIであるため、
利用者が複数回実行しても
問題ない。

例えば、

```text
1回目
テンプレート取得
    ↓
CSV保存

2回目
テンプレート取得
    ↓
CSV保存
```

としても、
業務データは変更されない。

---

#### 26.12 X-User-Id

`X-User-Id`は、
通常のAPIと同様に
共通API Clientから付与する。

概念例：

```ts
apiClient.interceptors.request.use(
  (config) => {
    config.headers[
      'X-User-Id'
    ] = currentuserId;

    return config;
  },
);
```

CSV-001専用処理で、
利用者IDを
URLやリクエストボディへ
設定しない。

---

#### 26.13 USER_CONTEXT_REQUIRED

`USER_CONTEXT_REQUIRED`
が返却された場合は、
利用者が
選択されていない状態として扱う。

CSVテンプレートの
保存処理は行わない。

利用者選択を
促す画面または
共通エラー処理へ遷移する。

---

#### 26.14 INVALID_USER_ID

`INVALID_USER_ID`
が返却された場合は、
共通エラーとして扱う。

通常の画面操作では、
フロントエンドが保持する
利用者IDを使用するため、
発生しないことを前提とする。

---

#### 26.15 USER_NOT_FOUND

`USER_NOT_FOUND`
が返却された場合は、
操作対象利用者を
現在利用できない状態として扱う。

利用者選択画面へ戻すなど、
API共通方針に従って処理する。

---

#### 26.16 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`
が返却された場合は、
CSVテンプレートを
保存しない。

共通の
サーバーエラー表示を行う。

必要に応じて、
再取得できる導線を表示する。

---

#### 26.17 Blobレスポンス時のエラー処理

API Clientで
`responseType: 'blob'`を指定した場合、
エラー時のJSONレスポンスも
Blobとして受け取る場合がある。

そのため、
共通API Clientで
Blobレスポンス時の
エラーJSONを解析できるようにする。

概念例：

```ts
const parseBlobError =
  async (
    blob: Blob,
  ): Promise<ApiErrorResponse> => {
    const text =
      await blob.text();

    return JSON.parse(
      text,
    ) as ApiErrorResponse;
  };
```

CSV-001専用画面で
独自のエラー形式を
定義しない。

---

#### 26.18 正常BlobとエラーBlobを区別する

HTTPステータスが
成功の場合のみ、
レスポンスBlobを
CSVファイルとして保存する。

エラー時に、

```text
JSONエラーレスポンス
```

を

```text
.csv
```

として
保存してはならない。

概念的には、

```text
200 OK
    ↓
CSVとして保存

4xx / 5xx
    ↓
APIエラーとして処理
```

とする。

---

#### 26.19 CSV内容をReact側で生成しない

CSV-001のテンプレートは、
バックエンドから取得する。

React側で、

```ts
const csv =
  'target_year_month,asset_account_name,balance';
```

のように
同じテンプレートを
独自生成しない。

テンプレート定義を
バックエンド側へ集約することで、
CSV-002・CSV-003との
仕様不整合を防止する。

---

#### 26.20 CSVヘッダーをフロントエンドで検証しない

CSV-001で取得した
テンプレートについて、
フロントエンド側で
ヘッダーが正しいかを
再検証する必要はない。

CSV仕様の保証は、
バックエンドおよび
テストの責務とする。

---

#### 26.21 CSV-002との連携

利用者は、
CSV-001で取得した
テンプレートへ
月末資産残高データを入力する。

その後、
CSV-002 月末資産残高CSVプレビューへ
ファイルを送信する。

概念的な画面フローは、
以下とする。

```text
CSVテンプレート取得
    ↓
利用者がCSV入力
    ↓
ファイル選択
    ↓
CSV-002
プレビュー
```

CSV-001実行直後に
CSV-002を
自動実行しない。

---

#### 26.22 CSV-003との連携

CSV-003は、
CSV-002で内容を確認した後に
月末資産残高を
一括登録するAPIである。

フロントエンドでは、
基本的に以下の流れとする。

```text
CSV-001
テンプレート取得
    ↓
CSV編集
    ↓
CSV-002
プレビュー
    ↓
内容確認
    ↓
CSV-003
登録
```

CSV-001から
直接CSV-003へ
連携する処理は行わない。

---

### 27 設計上の補足

#### 27.1 GETを採用する理由

CSV-001は、
CSVテンプレートを
取得するだけのAPIである。

業務データの
登録、
更新、
削除を行わないため、
HTTPメソッドには
`GET`を採用する。

---

#### 27.2 JSONではなくCSVを直接返却する理由

CSVテンプレートは、
利用者がそのまま
ファイルとして利用することを
目的としている。

そのため、

```json
{
  "data": {
    "csv": "..."
  }
}
```

のようなJSONに
CSV内容を埋め込まず、
CSVファイルを
直接レスポンスとして返却する。

これにより、
フロントエンド側でも
通常のファイルダウンロードとして
扱える。

---

#### 27.3 利用者固有データを含めない理由

CSVテンプレートの目的は、
入力形式を提供することである。

利用者の資産口座名などを
テンプレートへ含めると、
CSV-001が

```text
テンプレート取得
+
業務データ一覧取得
```

という複数の責務を
持つことになる。

そのため、
CSVテンプレートは
固定構造とする。

---

#### 27.4 X-User-Idを要求する理由

CSVテンプレート自体は
利用者非依存である。

ただし、
Phase1では
API全体で
操作対象利用者を
`X-User-Id`によって
統一的に扱う。

そのため、
CSV-001でも
利用者コンテキストを要求する。

これにより、
CSV関連APIだけ
利用者コンテキストの扱いが
異なる状態を避ける。

---

#### 27.5 IDをCSVへ含めない理由

利用者が
内部のデータベースIDを
確認してCSVへ入力する方式は、
操作性が悪い。

また、
内部IDをCSV仕様として
外部へ露出させる必要性も低い。

そのため、
Phase1では
資産口座を
`asset_account_name`によって
指定する。

CSV-002・CSV-003側で、
操作対象利用者との組み合わせにより
対象資産口座を特定する。

---

#### 27.6 CSV仕様を共通定義する理由

CSV-001、
CSV-002、
CSV-003で
ヘッダーを個別定義すると、
仕様変更時に
不整合が発生する可能性がある。

例えば、

```text
CSV-001
asset_account_name

CSV-002
account_name
```

のような差異が発生すると、
公式テンプレートを使用しても
プレビューできなくなる。

そのため、
バックエンドでは
共通のCSV Definitionを使用する。

---

#### 27.7 ヘッダー行のみを返す理由

CSVテンプレートは、
利用者が新しい
月末資産残高データを
入力するために使用する。

利用者固有の既存データを
テンプレートへ含める必要はない。

そのため、
Phase1では
ヘッダー行のみを返却する。

---

#### 27.8 CSVテンプレートにtarget_year_monthを含める理由

月末資産残高は、
対象年月ごとのデータである。

そのため、
各CSV行に
`target_year_month`を持たせる。

1ファイル内で
複数年月を許可するためではなく、
各行が
どの対象年月のデータであるかを
明示するためである。

1ファイル内では
1つの対象年月のみを扱う。

このルールは、
CSV-002・CSV-003で検証する。

---

#### 27.9 asset_account_nameを使用する理由

利用者が
CSV入力時に識別しやすい
資産口座名を使用する。

内部の
`asset_account_id`を
入力させない。

ただし、
同一利用者内で
資産口座名が一意であることを
前提とする。

CSV-002・CSV-003では、

```text
操作対象利用者
+
asset_account_name
```

によって
対象資産口座を特定する。

---

#### 27.10 balanceを整数で扱う理由

Phase1では、
金額を日本円の整数として扱う。

そのため、
CSVの`balance`も
整数として入力する。

例えば、

```text
1500000
```

のように入力する。

小数や通貨記号付き文字列は、
CSV-002・CSV-003で
不正値として扱う。

---

#### 27.11 テンプレート取得時にDB検索しない理由

CSVテンプレートの内容は、
業務データによって
変化しない。

そのため、
テンプレート生成時に

- `asset_accounts`
- `month_end_asset_snapshots`
- `month_end_asset_balances`

などを検索する必要はない。

不要なDBアクセスを避け、
テンプレート生成責務を
単純に保つ。

---

#### 27.12 トランザクションを使用しない理由

本APIは、
CSVテンプレートを
生成して返却するだけであり、
業務データを変更しない。

そのため、
更新トランザクションは不要である。

---

#### 27.13 ロックを使用しない理由

本APIは、
業務データを参照・更新しない。

他のCSV登録処理や
月末資産関連処理との
競合も発生しない。

そのため、
行ロックや
排他制御は使用しない。

---

#### 27.14 Idempotency-Keyを使用しない理由

本APIは、
読み取り専用のGET APIである。

複数回取得しても
業務データに
重複登録などの副作用が
発生しない。

そのため、
`Idempotency-Key`は使用しない。

---

#### 27.15 サーバーキャッシュを採用しない理由

CSVテンプレートは
固定かつ小さいため、
生成コストは非常に低い。

Phase1では、
キャッシュ管理の複雑性を
追加する必要はない。

CSV仕様変更時に
古いテンプレートが
キャッシュされ続けるリスクも避ける。

---

#### 27.16 CSV-002・CSV-003と一体で仕様変更する理由

CSV-001は、
CSV-002・CSV-003で
使用するファイル形式を
利用者へ提供するAPIである。

そのため、
CSV項目やヘッダーを変更する場合は、

```text
CSV-001
CSV-002
CSV-003
```

を一体として見直す。

テンプレートだけを
先行して変更しない。

---

### 28 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [エラーコード一覧](../../error-codes.md)
- [CSV-002 月末資産残高CSVプレビュー](./csv-002-preview.md)
- [CSV-003 月末資産残高CSV登録](./csv-003-create.md)
- [CSV-004 商品別月末評価額CSVテンプレート取得](./csv-004-template.md)
- [CSV-005 商品別月末評価額CSVプレビュー](./csv-005-preview.md)
- [CSV-006 商品別月末評価額CSV登録](./csv-006-create.md)
- [CSV-001 月末資産残高CSVテンプレート取得](./csv-001-create.md)
- [SNP-003 月末資産状況詳細取得](../month-end-assets/snp-003-detail.md)
- [SNP-004 月末資産状況確定](../month-end-assets/snp-004-confirm.md)
- [VAL-001 商品別月末評価額一覧取得](../month-end-assets/val-001-list.md)
- [商品別月末評価額API詳細](../month-end-assets/README.md)
- [資産口座API詳細](../asset-accounts/README.md)
- [保有商品API詳細](../holding-assets/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)
- [Laravel CSVインポート設計](../../../architecture/laravel/csv-imports.md)
- [React CSVインポート設計](../../../architecture/react/csv-imports.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)