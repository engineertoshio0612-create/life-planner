# CSVインポートAPI詳細設計

## 1. 概要

本ドキュメントでは、
CSVファイルを利用した
月末資産データの一括登録に関する
APIの詳細仕様を定義する。

Phase1では、
以下の2種類のCSVインポートを提供する。

```text
口座単位
    ↓
月末資産残高CSV

商品単位
    ↓
商品別月末評価額CSV
```

CSVインポートは、手入力による登録と同一の業務ルールを適用する。

利用者は、インポート対象に応じたCSVテンプレートを取得し、CSVファイルへ必要な情報を入力した後、登録前にプレビューを実行する。

CSVインポートの基本的な流れは、以下とする。

```text
CSVテンプレート取得
    ↓
CSVファイル作成
    ↓
CSVファイル選択
    ↓
プレビュー
    ↓
入力内容・エラー確認
    ↓
一括登録
```

1つのCSVファイルには、1つの対象年月のみを含める。

複数の対象年月を1つのCSVファイルへ含めて登録することはできない。

CSVプレビューでは、CSVファイルの内容を検証し、登録予定内容およびエラー内容を返却する。

プレビュー時点では、月末資産残高または商品別月末評価額をデータベースへ保存しない。

CSV登録では、プレビューで登録可能な内容であることを確認したCSVを一括登録する。

CSV内に登録できないデータが含まれる場合は、一部の行だけを登録せず、CSV全体を登録しない方針とする。

口座単位のCSVでは、主に以下の情報を扱う。

```text
対象年月
資産口座名
月末残高
```

商品単位のCSVでは、主に以下の情報を扱う。

```text
対象年月
資産口座名
保有商品名
月末評価額
```

CSVインポート時は、以下のような入力・業務ルールを検証する。

- 必須項目
- 対象年月形式
- 数値形式
- 整数であること
- 0円以上であること
- 資産口座が存在すること
- 対象年月時点で資産口座が有効であること
- 保有商品が存在すること
- 対象年月時点で保有商品が有効であること
- 残高記録単位とCSV種別が一致すること
- 重複登録がないこと
- 重複計上にならないこと

これらの入力チェックは、機能要件で定義されたCSVインポートの業務ルールに従う。

Phase1では、認証機能を実装しない。

操作対象となる利用者は、他の利用者依存APIと同様に、`X-User-Id`リクエストヘッダーで指定する。

CSVインポートでは、操作対象利用者に属する以下のリソースのみを検証および登録対象とする。

- 資産口座
- 保有商品
- 月末資産状況
- 月末資産残高
- 商品別月末評価額

他の利用者に属する資産情報をCSVインポートによって参照または登録対象としてはならない。

---

## 2. 対象API

本ドキュメントでは、以下の6APIを対象とする。

| API ID | 機能分類 | API名 | HTTPメソッド | URL |
| --- | --- | --- | --- | --- |
| `CSV-001` | CSVインポート | 月末資産残高CSVテンプレート取得 | GET | `/api/v1/month-end-asset-balances/csv-template` |
| `CSV-002` | CSVインポート | 月末資産残高CSVプレビュー | POST | `/api/v1/month-end-asset-balances/imports/preview` |
| `CSV-003` | CSVインポート | 月末資産残高CSV登録 | POST | `/api/v1/month-end-asset-balances/imports` |
| `CSV-004` | CSVインポート | 商品別月末評価額CSVテンプレート取得 | GET | `/api/v1/month-end-holding-values/csv-template` |
| `CSV-005` | CSVインポート | 商品別月末評価額CSVプレビュー | POST | `/api/v1/month-end-holding-values/imports/preview` |
| `CSV-006` | CSVインポート | 商品別月末評価額CSV登録 | POST | `/api/v1/month-end-holding-values/imports` |

---

### CSV-001 月末資産残高CSVテンプレート取得

口座単位で月末資産残高を登録するためのCSVテンプレートを取得する。

エンドポイントは、以下とする。

```http
GET /api/v1/month-end-asset-balances/csv-template
```

口座単位CSVの基本的な項目は、以下とする。

```text
対象年月
資産口座名
月末残高
```

本APIでは、月末資産残高の登録は行わない。

---

### CSV-002 月末資産残高CSVプレビュー

口座単位の月末資産残高CSVを受け取り、内容を検証する。

エンドポイントは、以下とする。

```http
POST /api/v1/month-end-asset-balances/imports/preview
```

主に以下を行う。

```text
CSV読み込み
    ↓
形式チェック
    ↓
資産口座確認
    ↓
残高記録単位確認
    ↓
対象年月確認
    ↓
重複確認
    ↓
登録予定内容・エラー返却
```

プレビューでは、データベースへ月末資産残高を保存しない。

---

### CSV-003 月末資産残高CSV登録

検証済みの口座単位CSVについて、月末資産残高を一括登録する。

エンドポイントは、以下とする。

```http
POST /api/v1/month-end-asset-balances/imports
```

登録処理は、トランザクション内で実行する。

CSV内に登録できないデータが存在する場合は、一部登録を行わず、CSV全体を登録しない。

また、確定済み月末資産状況に対して月末資産残高を登録してはならない。

---

### CSV-004 商品別月末評価額CSVテンプレート取得

商品単位で商品別月末評価額を登録するためのCSVテンプレートを取得する。

エンドポイントは、以下とする。

```http
GET /api/v1/month-end-holding-values/csv-template
```

商品単位CSVの基本的な項目は、以下とする。

```text
対象年月
資産口座名
保有商品名
月末評価額
```

本APIでは、商品別月末評価額の登録は行わない。

---

### CSV-005 商品別月末評価額CSVプレビュー

商品単位の商品別月末評価額CSVを受け取り、内容を検証する。

エンドポイントは、以下とする。

```http
POST /api/v1/month-end-holding-values/imports/preview
```

主に以下を行う。

```text
CSV読み込み
    ↓
形式チェック
    ↓
資産口座確認
    ↓
保有商品確認
    ↓
残高記録単位確認
    ↓
対象年月確認
    ↓
重複確認
    ↓
登録予定内容・エラー返却
```

プレビューでは、データベースへ商品別月末評価額を保存しない。

---

### CSV-006 商品別月末評価額CSV登録

検証済みの商品単位CSVについて、商品別月末評価額を一括登録する。

エンドポイントは、以下とする。

```http
POST /api/v1/month-end-holding-values/imports
```

登録処理は、トランザクション内で実行する。

CSV内に登録できないデータが存在する場合は、一部登録を行わず、CSV全体を登録しない。

また、確定済み月末資産状況に対して商品別月末評価額を登録してはならない。

---

## 3. CSV-001 月末資産残高CSVテンプレート取得

### 3.1 概要

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

### 3.2 ユースケース

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

### 3.3 エンドポイント

```http
GET /api/v1/month-end-asset-balances/csv-template
```

---

### 3.4 HTTPメソッド

```text
GET
```

本APIは、CSVテンプレートを取得する読み取り専用APIである。

API実行によって、業務データの状態は変更されない。

---

### 3.5 認証・利用者の扱い

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

### 3.6 パスパラメータ

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

### 3.7 クエリパラメータ

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

### 3.8 リクエストヘッダー

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

### 3.9 リクエストボディ

本APIでは、リクエストボディを使用しない。

本APIはGET APIであり、CSVテンプレートの取得条件としてクライアントから業務データを受け取る必要がない。

以下のような情報をリクエストボディで受け付けない。

- `targetYearMonth`
- `assetAccountId`
- `balance`
- `userId`

---

### 3.10 Content-Type

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

### 3.11 リクエスト例

```http
GET /api/v1/month-end-asset-balances/csv-template
X-User-Id: 1
```

本APIでは、パスパラメータ、クエリパラメータ、リクエストボディを指定しない。

---

### 3.12 バリデーション

本APIでは、CSVテンプレート取得のための業務入力値を受け取らない。

そのため、以下に対する個別のバリデーションは行わない。

- パスパラメータ
- クエリパラメータ
- リクエストボディ

ただし、`X-User-Id`については、API共通方針に従って検証する。

---

#### 3.12.1 X-User-Id必須

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

#### 3.12.2 X-User-Idの形式

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

#### 3.12.3 利用者の存在確認

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

#### 3.12.4 CSV内容のバリデーションは行わない

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

### 3.13 業務ルール

#### 3.13.1 CSVテンプレートの目的

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

#### 3.13.2 CSVテンプレートの項目

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

#### 3.13.3 データ行

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

#### 3.13.4 対象年月

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

#### 3.13.5 資産口座

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

#### 3.13.6 月末残高

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

#### 3.13.7 利用者固有データを含めない

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

#### 3.13.8 IDをCSV入力項目としない

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

#### 3.13.9 CSV-002・CSV-003との整合性

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

#### 3.13.10 CSV形式

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

#### 3.13.11 文字コード

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

#### 3.13.12 ファイル名

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

#### 3.13.13 データベースを更新しない

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

#### 3.13.14 CSVプレビューを行わない

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

#### 3.13.15 CSV登録を行わない

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

### 3.14 レスポンス

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

#### 3.14.1 正常レスポンス

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

#### 3.14.2 レスポンスヘッダー

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

#### 3.14.3 レスポンスボディ

レスポンスボディには、
CSVテンプレートを返却する。

概念例：

```csv
target_year_month,asset_account_name,balance
```

テンプレートには、
ヘッダー行を必ず含める。

---

#### 3.14.4 JSON Envelopeを使用しない

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

### 3.15 レスポンス項目

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

#### 3.15.1 target_year_month

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

#### 3.15.2 asset_account_name

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

#### 3.15.3 balance

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

#### 3.15.4 CSV入力例

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

#### 3.15.5 テンプレートに含めない項目

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

### 3.16 エラーレスポンス

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

#### 3.16.1 エラー一覧

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

#### 3.16.2 USER_CONTEXT_REQUIRED

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

#### 3.16.3 INVALID_USER_ID

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

#### 3.16.4 USER_NOT_FOUND

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

#### 3.16.5 INTERNAL_SERVER_ERROR

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

#### 3.16.6 CSV内容に関するエラー

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

#### 3.16.7 エラー時のレスポンス形式

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

#### 3.16.8 エラー時のContent-Type

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

### 3.17 HTTPステータス

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

#### 3.17.1 200 OK

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

#### 3.17.2 400 Bad Request

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

#### 3.17.3 404 Not Found

指定された利用者が
存在しない場合、
または論理削除済みの場合に使用する。

対象となるエラーコードは、

```text
USER_NOT_FOUND
```

とする。

---

#### 3.17.4 422 Unprocessable Entityを使用しない

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

#### 3.17.5 500 Internal Server Error

想定外のサーバー内部エラーが
発生した場合に使用する。

対象となるエラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

---

### 3.18 副作用

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

### 3.19 冪等性

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

#### 3.19.1 Idempotency-Key

本APIでは、
`Idempotency-Key`を使用しない。

本APIは
読み取り専用のGET APIであり、
複数回実行しても
重複登録などの問題が発生しないためである。

---

#### 3.19.2 再実行

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

#### 3.19.3 テンプレート仕様変更時

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

### 3.20 関連テーブル

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

#### 3.20.1 users

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

#### 3.20.2 参照しないテーブル

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

#### 3.20.3 更新対象テーブル

本APIでは、更新対象テーブルは存在しない。

以下のデータベース操作を行わない。

```text
INSERT
UPDATE
DELETE
```

CSVテンプレート取得履歴を業務テーブルへ保存することも行わない。

---

### 3.21 排他制御

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

#### 3.21.1 同時実行

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

#### 3.21.2 CSV登録APIとの同時実行

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

### 3.22 トランザクション

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

#### 3.22.1 利用者存在確認

`users`の存在確認についても、読み取りのみであるため、明示的なトランザクションを開始しない。

API共通Middlewareなどで利用者コンテキストを確認した後、CSVテンプレート生成処理へ進む。

---

#### 3.22.2 CSV-003との違い

CSV-003 月末資産残高CSV登録では、複数行を一括登録するため、トランザクションが必要となる。

一方、CSV-001ではデータベース更新を行わないため、トランザクションは不要である。

```text
CSV-001
    → トランザクション不要

CSV-003
    → トランザクション必要
```

---

### 3.23 性能・実装上の注意

#### 3.23.1 データベース検索

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

#### 3.23.2 N+1問題

本APIでは、資産口座や月末資産残高などの一覧取得を行わないため、N+1問題は発生しない。

利用者存在確認についても、API共通の利用者コンテキスト確認で1件を確認するだけとする。

---

#### 3.23.3 CSV生成

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

#### 3.23.4 CSV仕様の共通化

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

#### 3.23.5 CSVストリーム

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

#### 3.23.6 文字コード

CSVテンプレートは、CSV-002およびCSV-003が受け付ける文字コードと統一する。

Phase1では、UTF-8を使用する。

CSV-001だけ異なる文字コードでテンプレートを生成してはならない。

---

#### 3.23.7 改行コード

CSVの改行コードについても、CSV関連APIで統一する。

OSや実行環境によって意図せず形式が変化しないよう、CSV生成処理側で明示的に管理してよい。

---

#### 3.23.8 Content-Disposition

ブラウザからCSVファイルとして保存できるように、`Content-Disposition`を設定する。

```http
Content-Disposition: attachment; filename="month-end-asset-balances-template.csv"
```

利用者入力値をファイル名へ使用しないため、ファイル名に対するサニタイズ処理は不要とする。

---

#### 3.23.9 キャッシュ

Phase1では、CSVテンプレート専用のアプリケーションキャッシュを使用しない。

CSVテンプレートは非常に小さく、生成コストも低いため、キャッシュによる複雑性を追加する必要はない。

将来的に必要となった場合は、HTTPキャッシュを含めて別途検討する。

---

#### 3.23.10 CSV仕様変更

将来的に月末資産残高CSVの入力項目を変更する場合は、少なくとも以下を同時に確認する。

- CSV-001 テンプレート生成
- CSV-002 プレビュー
- CSV-003 登録
- CSV共通定義
- フロントエンドのCSVインポート画面
- テストケース

CSV-001だけを変更し、CSV-002またはCSV-003と仕様が異なる状態にしてはならない。

---

### 3.24 テスト観点

本APIでは、CSVテンプレートがCSV-002およびCSV-003で使用可能な形式として正しく取得できることを確認する。

---

#### 3.24.1 正常系

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

#### 3.24.2 CSVヘッダー

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

#### 3.24.3 利用者固有データ

CSVテンプレートに操作対象利用者の実データが含まれないことを確認する。

例えば、利用者に

```text
普通預金
証券口座
```

などの資産口座が登録されていても、テンプレートには出力されないことを確認する。

---

#### 3.24.4 利用者によるテンプレート差異

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

#### 3.24.5 X-User-Id未指定

`X-User-Id`を指定しない場合は、

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

となることを確認する。

エラー時にCSVファイルが返却されないことも確認する。

---

#### 3.24.6 X-User-Id形式不正

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

#### 3.24.7 利用者不存在

存在しない利用者IDを指定した場合は、

```text
404 Not Found
USER_NOT_FOUND
```

となることを確認する。

---

#### 3.24.8 論理削除済み利用者

`users.deleted_at`が設定されている利用者を指定した場合は、

```text
404 Not Found
USER_NOT_FOUND
```

となることを確認する。

---

#### 3.24.9 エラー時のレスポンス形式

エラー時は、CSVではなくAPI共通のJSONエラーレスポンスが返却されることを確認する。

```text
正常時
    → text/csv

異常時
    → application/json
```

となることを確認する。

---

#### 3.24.10 データベース非更新

CSV-001実行前後で、以下の業務テーブルにレコード追加・更新・削除が発生しないことを確認する。

- `users`
- `asset_accounts`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_account_available_settings`

---

#### 3.24.11 業務データ非参照

CSVテンプレート生成処理が、以下の業務データに依存しないことを確認する。

- 資産口座の登録件数
- 月末資産残高の登録件数
- 月末資産状況の有無
- 月末資産状況の確定状態
- 利用可能資産設定

これらの状態が異なっていても、同一のCSVテンプレートが取得できることを確認する。

---

#### 3.24.12 CSV-002・CSV-003との整合性

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

#### 3.24.13 文字コード

CSVテンプレートがCSV共通仕様で定めるUTF-8として正しく取得できることを確認する。

BOMを使用する仕様とした場合は、BOMの有無についてもテストする。

---

#### 3.24.14 改行コード

CSV共通仕様で改行コードを固定する場合は、期待する改行コードで出力されることを確認する。

環境によって意図せず改行コードが変化しないことを確認する。

---

#### 3.24.15 冪等性

同一の`X-User-Id`で複数回実行しても、業務データが変更されないことを確認する。

また、同一のシステム状態では同一のCSVテンプレートが取得できることを確認する。

---

#### 3.24.16 同時実行

同一利用者または複数利用者からCSV-001を同時実行しても、正常にCSVテンプレートを取得できることを確認する。

他のCSV登録処理や月末資産関連APIが実行中であっても、CSV-001が不要なロックによってブロックされないことを確認する。

---

#### 3.24.17 想定外例外

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

#### 3.24.18 主なテストケース

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

### 3.25 Laravel実装方針

### 3.25 Laravel実装方針

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

#### 3.25.1 Action

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

#### 3.25.2 FormRequest

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

#### 3.25.3 Middleware

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

#### 3.25.4 UseCase

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

#### 3.25.5 UseCaseで行わないこと

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

#### 3.25.6 CSV Template DTO

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

#### 3.25.7 CSV Definition

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

#### 3.25.8 CSVヘッダー

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

#### 3.25.9 CSV Generator

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

#### 3.25.10 fputcsvの利用

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

#### 3.25.11 fopen失敗時

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

#### 3.25.12 fputcsv失敗時

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

#### 3.25.13 リソース解放

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

#### 3.25.14 文字コード

CSVテンプレートは、UTF-8で生成する。

CSV-002およびCSV-003で受け付ける文字コードと統一する。

CSV-001だけ異なる文字コードへ変換しない。

---

#### 3.25.15 BOM

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

#### 3.25.16 改行コード

CSVテンプレートの改行コードは、CSV関連APIで統一する。

CSV-001、CSV-002、CSV-003で異なる改行コード仕様を個別定義しない。

実行環境による差異を許容しない仕様とする場合は、CSV Generator側で明示的に制御する。

---

#### 3.25.17 データ行

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

#### 3.25.18 Repository・Query

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

#### 3.25.19 API Resource

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

#### 3.25.20 Responder

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

#### 3.25.21 Responderの責務

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

#### 3.25.22 Content-Type

正常時の`Content-Type`は、以下とする。

```http
Content-Type: text/csv; charset=UTF-8
```

CSVファイルをJSONとして返却しない。

---

#### 3.25.23 Content-Disposition

CSVファイルとしてダウンロードできるよう、`Content-Disposition`を設定する。

概念的には、以下とする。

```http
Content-Disposition: attachment; filename="month-end-asset-balances-template.csv"
```

ファイル名は、CSV Definitionで一元管理する。

ActionやResponderへ文字列リテラルとして重複定義しない。

---

#### 3.25.24 正常時のJSON Envelope

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

#### 3.25.25 エラー時のレスポンス

エラー時は、CSVではなくAPI共通のJSONエラーレスポンスを使用する。

```text
正常時
    → text/csv

エラー時
    → application/json
```

MiddlewareまたはException Handlerで発生した例外は、API共通方針に従ってJSONへ変換する。

---

#### 3.25.26 例外変換

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

#### 3.25.27 想定外例外

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

#### 3.25.28 トランザクション

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

#### 3.25.29 ロック

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

#### 3.25.30 キャッシュ

Phase1では、CSVテンプレート専用のサーバー側アプリケーションキャッシュを使用しない。

CSVテンプレートはヘッダー行のみであり、生成コストが非常に小さい。

また、CSV仕様変更後に古いテンプレートがキャッシュされ続ける複雑性を避ける。

必要になった場合は、将来的にHTTPキャッシュなどを別途検討する。

---

#### 3.25.31 CSV-002・CSV-003との共通化

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

#### 3.25.32 CSV Generatorの共通化範囲

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

#### 3.25.33 ログ

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

#### 3.25.34 テスト実装方針

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

#### 3.25.35 CSV DefinitionのUnit Test

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

#### 3.25.36 CSV GeneratorのUnit Test

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

#### 3.25.37 CSV Generator異常系のUnit Test

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

#### 3.25.38 UseCaseのUnit Test

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

#### 3.25.39 ResponderのTest

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

#### 3.25.40 CSV-002・CSV-003との整合性テスト

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

### 3.26 React・TypeScriptでの利用

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

#### 3.26.1 レスポンス型

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

#### 3.26.2 responseType

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

#### 3.26.3 ファイル保存

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

#### 3.26.4 Content-Dispositionの利用

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

#### 3.26.5 Content-Typeの確認

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

#### 3.26.6 ダウンロードボタン

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

#### 3.26.7 ダウンロード処理例

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

#### 3.26.8 Mutationとして扱うか

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

#### 3.26.9 キャッシュ

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

#### 3.26.10 ローディング状態

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

#### 3.26.11 複数回取得

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

#### 3.26.12 X-User-Id

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

#### 3.26.13 USER_CONTEXT_REQUIRED

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

#### 3.26.14 INVALID_USER_ID

`INVALID_USER_ID`
が返却された場合は、
共通エラーとして扱う。

通常の画面操作では、
フロントエンドが保持する
利用者IDを使用するため、
発生しないことを前提とする。

---

#### 3.26.15 USER_NOT_FOUND

`USER_NOT_FOUND`
が返却された場合は、
操作対象利用者を
現在利用できない状態として扱う。

利用者選択画面へ戻すなど、
API共通方針に従って処理する。

---

#### 3.26.16 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`
が返却された場合は、
CSVテンプレートを
保存しない。

共通の
サーバーエラー表示を行う。

必要に応じて、
再取得できる導線を表示する。

---

#### 3.26.17 Blobレスポンス時のエラー処理

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

#### 3.26.18 正常BlobとエラーBlobを区別する

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

#### 3.26.19 CSV内容をReact側で生成しない

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

#### 3.26.20 CSVヘッダーをフロントエンドで検証しない

CSV-001で取得した
テンプレートについて、
フロントエンド側で
ヘッダーが正しいかを
再検証する必要はない。

CSV仕様の保証は、
バックエンドおよび
テストの責務とする。

---

#### 3.26.21 CSV-002との連携

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

#### 3.26.22 CSV-003との連携

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

### 3.27 設計上の補足

#### 3.27.1 GETを採用する理由

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

#### 3.27.2 JSONではなくCSVを直接返却する理由

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

#### 3.27.3 利用者固有データを含めない理由

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

#### 3.27.4 X-User-Idを要求する理由

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

#### 3.27.5 IDをCSVへ含めない理由

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

#### 3.27.6 CSV仕様を共通定義する理由

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

#### 3.27.7 ヘッダー行のみを返す理由

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

#### 3.27.8 CSVテンプレートにtarget_year_monthを含める理由

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

#### 3.27.9 asset_account_nameを使用する理由

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

#### 3.27.10 balanceを整数で扱う理由

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

#### 3.27.11 テンプレート取得時にDB検索しない理由

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

#### 3.27.12 トランザクションを使用しない理由

本APIは、
CSVテンプレートを
生成して返却するだけであり、
業務データを変更しない。

そのため、
更新トランザクションは不要である。

---

#### 3.27.13 ロックを使用しない理由

本APIは、
業務データを参照・更新しない。

他のCSV登録処理や
月末資産関連処理との
競合も発生しない。

そのため、
行ロックや
排他制御は使用しない。

---

#### 3.27.14 Idempotency-Keyを使用しない理由

本APIは、
読み取り専用のGET APIである。

複数回取得しても
業務データに
重複登録などの副作用が
発生しない。

そのため、
`Idempotency-Key`は使用しない。

---

#### 3.27.15 サーバーキャッシュを採用しない理由

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

#### 3.27.16 CSV-002・CSV-003と一体で仕様変更する理由

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

### 3.28 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)

---

## 4. CSV-002 月末資産残高CSVプレビュー

### 4.1 概要

操作対象となる利用者について、
アップロードされた
月末資産残高CSVの内容を検証し、
登録予定内容および
エラー内容をプレビューとして返却する。

本APIでは、
CSVファイルを受け取り、
各行について
月末資産残高CSVとして
登録可能な内容であるかを検証する。

主に以下を確認する。

- CSVファイル形式
- CSVヘッダー
- 対象年月
- 資産口座名
- 月末残高
- 1ファイル内の対象年月統一
- 資産口座の存在
- 資産口座の利用者境界
- 残高記録単位
- 対象年月時点での資産口座の有効性
- 月末資産状況の状態
- 既存月末資産残高との重複
- CSV内の重複

CSVプレビューでは、
CSV内容を検証するだけであり、
月末資産残高を
データベースへ登録しない。

また、
月末資産状況、
資産口座、
利用可能資産設定などの
業務データも更新しない。

概念的な処理は、
以下とする。

```text
CSVファイル受信
    ↓
ファイル検証
    ↓
CSVヘッダー検証
    ↓
CSV行解析
    ↓
対象年月検証
    ↓
資産口座検証
    ↓
月末残高検証
    ↓
重複検証
    ↓
登録予定内容・エラー内容生成
    ↓
プレビュー返却
```

---

### 4.6 パスパラメータ

本APIでは、
パスパラメータを使用しない。

エンドポイントは、
以下とする。

```http
POST /api/v1/month-end-asset-balances/imports/preview
```

対象年月、
資産口座、
月末残高などは、
CSVファイル内の各項目から取得する。

そのため、
以下の情報を
パスパラメータとして受け付けない。

- `targetYearMonth`
- `assetAccountId`
- `snapshotId`
- `userId`

---

### 4.7 クエリパラメータ

本APIでは、
クエリパラメータを使用しない。

CSVプレビュー対象となる情報は、
アップロードされたCSVファイルから取得する。

そのため、
以下のような
クエリパラメータは受け付けない。

- `targetYearMonth`
- `assetAccountId`
- `confirmed`
- `overwrite`
- `validateOnly`

本API自体が
プレビュー専用APIであるため、
`preview=true`のような
指定も行わない。

---

### 4.8 リクエストヘッダー

本APIでは、
以下のリクエストヘッダーを使用する。

| ヘッダー | 必須 | 内容 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
POST /api/v1/month-end-asset-balances/imports/preview
Accept: application/json
X-User-Id: 1
```

CSVファイルは
`multipart/form-data`で送信するため、
`Content-Type`は
HTTPクライアントによって
boundary付きで設定する。

概念例：

```http
Content-Type: multipart/form-data; boundary=...
```

クライアント側で
固定のboundaryを
手動設定しない。

---

### 4.9 リクエストボディ

本APIでは、
`multipart/form-data`形式で
CSVファイルを受け取る。

リクエスト項目は、
以下とする。

| 項目 | 型 | 必須 | 内容 |
|---|---|:---:|---|
| `file` | file | ○ | プレビュー対象となる月末資産残高CSV |

概念例：

```text
multipart/form-data

file:
  month-end-asset-balances.csv
```

本APIでは、
CSVファイル以外の
業務項目を
リクエストボディで受け付けない。

以下のような項目は、
multipart/form-dataの
追加フィールドとして
指定しない。

- `userId`
- `targetYearMonth`
- `assetAccountId`
- `balance`
- `confirmed`

これらは、
`X-User-Id`または
CSV内容から取得する。

---

### 4.10 CSVファイル仕様

月末資産残高CSVでは、
以下のヘッダーを使用する。

```csv
target_year_month,asset_account_name,balance
```

ヘッダー順序も
CSV仕様の一部として扱う。

CSV-001で取得できる
テンプレートと
同一形式とする。

CSV入力例：

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,生活用口座,800000
```

1つのCSVファイルでは、
1つの`target_year_month`のみを扱う。

---

### 4.11 バリデーション

CSV-002では、
以下の3段階で
検証する。

```text
リクエスト検証
    ↓
CSVファイル・構造検証
    ↓
CSV行・業務ルール検証
```

プレビューでは、
検証結果を返却するだけであり、
月末資産残高を
データベースへ登録しない。

---

#### 4.11.1 file 必須チェック

`file`は、
必須とする。

CSVファイルが
指定されていない場合は、
バリデーションエラーとする。

不正例：

```text
file未指定
```

この場合、
CSV解析処理へ進まない。

---

#### 4.11.2 ファイルであること

`file`は、
HTTPアップロードファイルとして
正常に受信できていることを確認する。

文字列や
通常のフォーム値を
CSVファイルとして扱わない。

---

#### 4.11.3 ファイル拡張子

アップロードファイルは、
CSVファイルであることを確認する。

Phase1では、
拡張子として

```text
.csv
```

を受け付ける。

例えば、
以下は不正とする。

```text
.xlsx
.xls
.txt
.pdf
```

拡張子のみを
唯一の判定根拠とはせず、
実際にCSVとして
読み取り可能であることも確認する。

---

#### 4.11.4 ファイルサイズ

CSVファイルサイズには、
API共通または
CSV共通仕様で定めた
上限を適用する。

上限を超えるファイルは、
CSV解析前に
バリデーションエラーとする。

具体的な上限値は、
CSV共通仕様の
最新定義に従う。

---

#### 4.11.5 空ファイル

CSVファイルが
0バイトの場合、
またはCSVとして
有効なヘッダー行を
取得できない場合は、
不正なCSVとして扱う。

以下のような状態は
正常なプレビュー対象としない。

```text
空ファイル
```

---

#### 4.11.6 文字コード

CSVファイルの文字コードは、
CSV共通仕様に従う。

Phase1では、
UTF-8を基本とする。

CSV-001で取得した
テンプレートと
同じ文字コードを
使用することを前提とする。

BOMを許可する場合は、
先頭ヘッダーに
BOMが混入しないよう
解析時に適切に除去する。

---

#### 4.11.7 CSVとして読み取り可能であること

アップロードされたファイルが、
CSVとして正常に
読み取れることを確認する。

例えば、
以下のような状態を
不正として扱う。

- ファイル破損
- CSVとして解析不能
- 不正な引用符
- 想定外の構造

解析処理で
PHP内部エラーや
例外をそのまま
クライアントへ返却しない。

---

#### 4.11.8 ヘッダー必須

CSVの1行目には、
ヘッダー行が
存在することを必須とする。

期待するヘッダーは、
以下とする。

```text
target_year_month
asset_account_name
balance
```

---

#### 4.11.9 ヘッダー名

ヘッダーは、
以下と完全一致することを
基本とする。

```csv
target_year_month,asset_account_name,balance
```

例えば、
以下のような
別名は受け付けない。

```text
targetYearMonth
assetAccountName
amount
```

CSV-001のテンプレートを
正式な入力形式とする。

---

#### 4.11.10 ヘッダー順序

ヘッダーは、
以下の順序とする。

```text
1. target_year_month
2. asset_account_name
3. balance
```

Phase1では、
ヘッダー名が同じでも
順序が異なるCSVは
不正として扱う。

これにより、
CSV-001、
CSV-002、
CSV-003で
CSV仕様を統一する。

---

#### 4.11.11 余分なヘッダー

定義されていない
余分なCSV列は
受け付けない。

例えば、

```csv
target_year_month,asset_account_name,balance,memo
```

は不正とする。

以下のような
内部管理項目も
CSV列として受け付けない。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- `confirmed`

---

#### 4.11.12 データ行の存在

ヘッダー行だけで
データ行が1件も存在しない場合は、
登録対象データが存在しないため、
CSVプレビュー対象として
不正とする。

```csv
target_year_month,asset_account_name,balance
```

のみのCSVは、
登録可能データ0件として扱う。

具体的なエラーコードは、
「エラーレスポンス」で定義する。

---

#### 4.11.13 空行

CSV末尾などの
完全な空行については、
CSV共通仕様に従って
無視してよい。

ただし、

```csv
,,
```

のように、
列だけ存在する行は
データ行として扱い、
各項目の必須チェックを行う。

---

#### 4.11.14 target_year_month 必須

各データ行の
`target_year_month`は
必須とする。

空文字の場合は、
行エラーとする。

不正例：

```csv
target_year_month,asset_account_name,balance
,普通預金,1500000
```

---

#### 4.11.15 target_year_month 形式

`target_year_month`は、
以下の形式とする。

```text
YYYY-MM
```

正常例：

```text
2026-01
2026-07
2026-12
```

不正例：

```text
2026-1
26-07
2026/07
202607
2026-00
2026-13
abc
```

年月として
有効な値であることも確認する。

---

#### 4.11.16 1ファイル1対象年月

1つのCSVファイルでは、
すべてのデータ行の
`target_year_month`が
同一であることを必須とする。

正常例：

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,生活用口座,800000
```

不正例：

```csv
target_year_month,asset_account_name,balance
2026-06,普通預金,1500000
2026-07,生活用口座,800000
```

複数の対象年月が
混在している場合は、
CSV全体として
登録不可とする。

---

#### 4.11.17 asset_account_name 必須

各データ行の
`asset_account_name`は
必須とする。

空文字の場合は、
行エラーとする。

不正例：

```csv
target_year_month,asset_account_name,balance
2026-07,,1500000
```

---

#### 4.11.18 asset_account_nameの文字列

`asset_account_name`は、
文字列として扱う。

前後空白の扱いは、
CSV共通仕様または
資産口座名の入力ルールに従う。

入力値を
暗黙的に別名へ変換して
資産口座を特定しない。

---

#### 4.11.19 資産口座の存在確認

CSVの
`asset_account_name`に対応する
資産口座が、
操作対象利用者に
存在することを確認する。

概念的には、
以下の条件で検索する。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = asset_account_name

AND

asset_accounts.deleted_at
    IS NULL
```

該当する資産口座が
存在しない場合は、
その行を登録不可とする。

---

#### 4.11.20 他利用者の同名資産口座

操作対象利用者には
該当する資産口座が存在せず、
他の利用者にのみ
同名資産口座が存在する場合でも、
正常とは判定しない。

例えば、

```text
User A
普通預金なし

User B
普通預金あり
```

で、
User Aとして

```text
asset_account_name = 普通預金
```

を指定した場合は、
資産口座不存在として扱う。

---

#### 4.11.21 残高記録単位

CSV-002は、
月末資産残高CSVの
プレビューAPIである。

そのため、
対象資産口座の
`balance_recording_unit`が
口座単位であることを確認する。

商品単位で
評価額を記録する資産口座は、
月末資産残高CSVの
登録対象としない。

概念的には、

```text
balance_recording_unit
    = 口座単位
```

を必須とする。

商品単位の資産口座については、
商品別月末評価額CSVを使用する。

---

#### 4.11.22 対象年月時点での資産口座の有効性

対象資産口座が、
CSVの`target_year_month`時点で
月末資産残高の
記録対象として有効であることを確認する。

現在時点で
有効かどうかだけではなく、
対象年月時点の状態を
基準とする。

具体的な有効期間判定は、
資産口座および関連設定の
機能要件・テーブル定義に従う。

---

#### 4.11.23 balance 必須

各データ行の
`balance`は
必須とする。

空文字の場合は、
行エラーとする。

不正例：

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,
```

---

#### 4.11.24 balance 数値形式

`balance`は、
日本円の整数として
扱える値であることを確認する。

正常例：

```text
0
1000
1500000
```

不正例：

```text
1,500,000
¥1500000
1500.5
abc
```

CSV上では、
桁区切りや
通貨記号を使用しない。

---

#### 4.11.25 balance 整数

`balance`は、
整数であることを必須とする。

以下のような
小数値は受け付けない。

```text
1000.5
```

Phase1では、
日本円の整数値として管理する。

---

#### 4.11.26 balance 0円以上

`balance`は、
0以上とする。

正常例：

```text
0
1
1500000
```

不正例：

```text
-1
-100000
```

0円は
正常な業務値として扱う。

```text
0円
≠
未入力
```

とする。

---

#### 4.11.27 月末資産状況の確認

CSVの`target_year_month`について、
操作対象利用者に属する
`month_end_asset_snapshots`を確認する。

概念的な検索条件は、
以下とする。

```text
user_id = 操作対象利用者ID
AND
target_year_month = target_year_month
```

他の利用者の
同一対象年月の
月末資産状況を
判定へ使用しない。

---

#### 4.11.28 確定済み月末資産状況

対象年月の
月末資産状況が存在し、

```text
confirmed = true
```

の場合は、
月末資産残高を
追加・変更できないため、
CSV登録不可として扱う。

CSV-002では、
この状態を
プレビューエラーとして返却する。

確定済みデータを
プレビューだからという理由で
登録可能扱いしない。

---

#### 4.11.29 月末資産状況が存在しない場合

対象年月の
`month_end_asset_snapshots`が
存在しない場合の扱いは、
CSV-003の登録仕様と
一致させる。

CSV-003で
月末資産状況を自動作成する仕様であれば、
CSV-002でも登録可能として
プレビューする。

CSV-003で
既存の月末資産状況を必須とする仕様であれば、
CSV-002でも
不存在エラーとする。

CSV-002とCSV-003で
異なる判定を行ってはならない。

---

#### 4.11.30 既存月末資産残高との重複

対象年月、
対象資産口座について、
既に
`month_end_asset_balances`が
存在するか確認する。

概念的には、

```text
month_end_asset_snapshot_id
+
asset_account_id
```

の組み合わせで確認する。

既存データが存在する場合の扱いは、
CSV-003の登録方式に従う。

Phase1でCSV登録が
新規一括登録のみである場合は、
既存データを
重複エラーとして扱う。

CSVプレビュー時点で
既存データを更新しない。

---

#### 4.11.31 CSV内重複

同一CSV内で、
同じ対象年月かつ
同じ資産口座が
複数行存在しないことを確認する。

例えば、

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,普通預金,1600000
```

は、
どちらを正式値とするか
一意に決定できないため、
重複エラーとする。

後勝ち、
先勝ちなどの
暗黙的なルールは採用しない。

---

#### 4.11.32 行番号

CSV行エラーを返却する際は、
利用者がCSV上の
該当箇所を確認できるよう、
行番号を保持する。

ヘッダーを1行目とした場合、
最初のデータ行は
2行目として扱う。

概念例：

```text
rowNumber = 2
```

CSV解析時の
内部配列インデックスを
そのまま画面用行番号として
返却しない。

---

#### 4.11.33 複数エラー

1つのCSVに
複数の入力エラーが存在する場合は、
可能な範囲で
複数のエラーをまとめて返却する。

例えば、

```text
2行目
asset_account_name 不存在

4行目
balance 負数

5行目
target_year_month 形式不正
```

のように、
利用者が一度のプレビューで
複数箇所を修正できるようにする。

最初の1件のエラーだけで
解析全体を終了する方式は
基本としない。

ただし、
CSVヘッダー不正など、
後続行を正しく解釈できない場合は、
そこでCSV行解析を
中止してよい。

---

#### 4.11.34 エラーが1件でも存在する場合

CSV内に
登録不可となるエラーが
1件でも存在する場合は、
CSV全体を登録可能とは判定しない。

```text
1行目 正常
2行目 正常
3行目 エラー
    ↓
CSV全体
登録不可
```

CSV-003で
正常行だけを部分登録する
前提にはしない。

---

#### 4.11.35 プレビュー時のDB更新禁止

CSV-002では、
検証のために
業務データを参照するが、
以下を行わない。

```text
INSERT
UPDATE
DELETE
```

特に、

- `month_end_asset_snapshots`
- `month_end_asset_balances`

を更新しない。

CSVプレビューを
複数回実行しても、
業務データの状態は変化しない。

---

#### 4.11.36 CSV-003との再検証

CSV-002で
正常判定されたCSVであっても、
CSV-003実行時には
同じ業務ルールを
再検証する。

CSV-002のプレビュー結果だけを
信頼して、
CSV-003側の検証を
省略してはならない。

例えば、
CSV-002実行後に
別処理によって
月末資産状況が確定される可能性がある。

```text
CSV-002
プレビュー時
confirmed = false
    ↓
別処理
confirmed = true
    ↓
CSV-003
再検証
    ↓
登録不可
```

とする。

CSV-002は、
CSV-003の登録可否を
保証・予約するAPIではない。

---

### 4.12 業務ルール

#### 4.12.1 プレビューの目的

本APIは、
月末資産残高CSVを
実際に登録する前に、
CSV内容および業務ルールを検証し、
登録予定内容を確認するためのAPIである。

CSV-002では、
CSV-003 月末資産残高CSV登録と
同等の登録可否判定を行う。

ただし、
プレビュー時点では
業務データを登録・更新しない。

基本的な処理は、
以下とする。

```text
CSV受信
    ↓
CSV構造検証
    ↓
各行の入力値検証
    ↓
業務ルール検証
    ↓
登録予定内容生成
    ↓
エラー内容生成
    ↓
プレビュー結果返却
```

---

#### 4.12.2 1ファイル1対象年月

1つのCSVファイルでは、
1つの対象年月のみを扱う。

すべてのデータ行で
`target_year_month`が
同一であることを必須とする。

正常例：

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,生活用口座,800000
```

不正例：

```csv
target_year_month,asset_account_name,balance
2026-06,普通預金,1400000
2026-07,生活用口座,800000
```

複数の対象年月が
混在している場合は、
CSV全体を登録不可とする。

---

#### 4.12.3 資産口座の特定

資産口座は、

```text
操作対象利用者
+
asset_account_name
```

によって特定する。

概念的な条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = asset_account_name

AND

asset_accounts.deleted_at
    IS NULL
```

`asset_account_id`を
CSV入力項目として使用しない。

---

#### 4.12.4 利用者境界

資産口座、
月末資産状況、
月末資産残高の確認では、
必ず操作対象利用者との
関連を保証する。

他の利用者に属する
同名資産口座や
同一対象年月のデータを
判定に使用してはならない。

例えば、

```text
User A
普通預金なし

User B
普通預金あり
```

の場合に、
User Aとして
`普通預金`を指定したCSVを
プレビューした場合は、
資産口座不存在として扱う。

---

#### 4.12.5 残高記録単位

本APIは、
月末資産残高CSVを
対象とする。

そのため、
対象資産口座の
`balance_recording_unit`が
口座単位であることを必須とする。

商品単位で管理する
資産口座については、
本APIの対象としない。

商品単位の資産口座は、
CSV-005 商品別月末評価額CSVプレビューを
使用する。

---

#### 4.12.6 対象年月時点の資産口座

資産口座が
CSVの`target_year_month`時点で
月末資産残高の
記録対象として有効であることを確認する。

現在時点の状態だけで
判定してはならない。

対象年月時点の
利用可能期間に基づいて
登録可否を判定する。

---

#### 4.12.7 月末資産状況

CSVの対象年月について、
操作対象利用者の
月末資産状況を確認する。

概念的な条件は、
以下とする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = target_year_month
```

他の利用者の
月末資産状況を
使用してはならない。

---

#### 4.12.8 月末資産状況が存在しない場合

対象年月の
月末資産状況が
まだ存在しない場合は、
CSV-003の登録仕様に従って
登録可能性を判定する。

CSV-003で
月末資産状況を
登録時に作成する仕様とする場合は、
CSV-002では
月末資産状況が存在しないことだけを理由に
登録不可とはしない。

ただし、
CSV-002では
月末資産状況を
実際に作成しない。

```text
CSV-002
月末資産状況なし
    ↓
登録可能としてプレビュー
    ↓
DB更新なし

CSV-003
再検証
    ↓
必要に応じて月末資産状況作成
```

---

#### 4.12.9 確定済み月末資産状況

対象年月の
月末資産状況が存在し、

```text
confirmed = true
```

の場合は、
月末資産残高を
登録できない。

この場合、
CSV全体を
登録不可として扱う。

CSVプレビューであることを理由に、
確定済み月末資産状況への
登録を可能とは判定しない。

---

#### 4.12.10 月末残高

`balance`は、
対象年月末時点における
資産口座の残高として扱う。

Phase1では、
日本円の整数として扱う。

以下を満たす必要がある。

```text
整数
AND
0以上
```

0円は
有効な月末残高とする。

```text
balance = 0
```

を未入力とは扱わない。

---

#### 4.12.11 既存月末資産残高

対象年月および
対象資産口座について、
既存の月末資産残高を確認する。

概念的には、

```text
month_end_asset_snapshot_id
+
asset_account_id
```

によって
既存データを判定する。

CSV登録を
新規一括登録として扱う場合、
既存の月末資産残高が存在する行は
重複として登録不可とする。

CSV-002では、
既存レコードを
更新・削除しない。

---

#### 4.12.12 CSV内重複

同一CSV内で、
同じ資産口座を
複数行指定してはならない。

1ファイル1対象年月であるため、
実質的には
`asset_account_name`が
CSV内で一意となる必要がある。

不正例：

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,普通預金,1600000
```

先勝ち、
後勝ち、
残高の合算などは行わない。

---

#### 4.12.13 部分的な登録可能判定を行わない

CSV内に
1件でも登録不可となるエラーが存在する場合は、
CSV全体を登録不可とする。

例えば、

```text
2行目 正常
3行目 正常
4行目 エラー
```

の場合、

```text
CSV全体
    ↓
登録不可
```

と判定する。

正常行だけを
CSV-003で登録することを
前提としない。

---

#### 4.12.14 複数エラーの返却

CSV構造そのものが
解析不能でない限り、
可能な範囲で
複数の行エラーを収集して返却する。

例えば、

```text
2行目
資産口座不存在

4行目
balanceが負数

6行目
資産口座重複
```

を1回のプレビューで
確認できるようにする。

これにより、
利用者がCSVを
繰り返しアップロードしながら
1件ずつ修正する必要を減らす。

---

#### 4.12.15 行番号

行単位のエラーおよび
プレビュー内容には、
CSV上の行番号を付与する。

ヘッダーを
1行目として扱うため、
最初のデータ行は

```text
rowNumber = 2
```

となる。

利用者が
CSVファイル上の該当行を
特定できる番号とする。

---

#### 4.12.16 プレビュー成功

CSV全体に
登録不可となるエラーが存在しない場合は、
登録可能なプレビュー結果を返却する。

概念的には、

```text
canImport = true
```

とする。

ただし、
`canImport = true`は
CSV-003での登録成功を
保証するものではない。

---

#### 4.12.17 プレビューエラー

CSV内容に
登録不可となるエラーが
1件以上存在する場合は、

```text
canImport = false
```

とする。

その場合でも、
CSVファイル自体を
正常に解析して
プレビュー結果を生成できた場合は、
行単位のエラー情報を
レスポンスへ含める。

CSV内容の業務エラーと、
APIリクエスト自体のエラーは
区別して扱う。

---

#### 4.12.18 CSV-003での再検証

CSV-002で
`canImport = true`となった場合でも、
CSV-003では
同じ入力・業務ルールを
再検証する。

CSV-002実行後に
データベースの状態が
変化する可能性があるためである。

例えば、

```text
CSV-002
confirmed = false
    ↓
canImport = true

別処理
月末資産状況を確定
    ↓

CSV-003
confirmed = true
    ↓
登録不可
```

となる。

CSV-002の結果を
登録権利の予約として扱わない。

---

#### 4.12.19 プレビュー結果を保存しない

CSV-002では、
プレビュー結果を
データベースへ保存しない。

以下のような
プレビュー管理テーブルを
Phase1では作成しない。

```text
csv_import_previews
csv_import_sessions
```

CSV-003では、
CSV内容を再度受け取り、
再検証したうえで登録する。

---

#### 4.12.20 業務データを更新しない

本APIでは、
以下の処理を行わない。

```text
INSERT
UPDATE
DELETE
```

特に、

- 月末資産状況の作成
- 月末資産残高の登録
- 月末資産残高の更新
- 月末資産残高の削除
- 確定状態の変更

を行わない。

---

### 4.13 レスポンス

CSVファイルを
正常に解析できた場合は、
プレビュー結果を
JSON形式で返却する。

正常なCSVだけでなく、
CSV内に業務エラーが存在する場合も、
CSV解析自体が正常に完了していれば
プレビュー結果として返却する。

概念的なレスポンスは、
以下とする。

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "canImport": true,
    "rows": [
      {
        "rowNumber": 2,
        "assetAccountName": "普通預金",
        "balance": 1500000,
        "errors": []
      },
      {
        "rowNumber": 3,
        "assetAccountName": "生活用口座",
        "balance": 800000,
        "errors": []
      }
    ]
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

API共通の
成功レスポンスEnvelopeに従う。

---

#### 4.13.1 登録可能な場合

CSV全体が
登録可能な場合は、

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "canImport": true,
    "rows": [
      {
        "rowNumber": 2,
        "assetAccountName": "普通預金",
        "balance": 1500000,
        "errors": []
      }
    ]
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

のように返却する。

`errors`は
空配列とする。

---

#### 4.13.2 登録不可行が存在する場合

CSVを解析できるが、
登録不可となる行が
存在する場合は、

```text
canImport = false
```

として返却する。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "canImport": false,
    "rows": [
      {
        "rowNumber": 2,
        "assetAccountName": "普通預金",
        "balance": 1500000,
        "errors": []
      },
      {
        "rowNumber": 3,
        "assetAccountName": "存在しない口座",
        "balance": 800000,
        "errors": [
          {
            "code": "ASSET_ACCOUNT_NOT_FOUND",
            "message": "指定された資産口座が存在しません。"
          }
        ]
      }
    ]
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

この場合、
CSV全体を
登録不可として扱う。

---

#### 4.13.3 複数の行エラー

1行に
複数のエラーが存在する場合は、
`errors`へ複数件格納できる。

概念例：

```json
{
  "rowNumber": 3,
  "assetAccountName": "",
  "balance": -100,
  "errors": [
    {
      "code": "ASSET_ACCOUNT_NAME_REQUIRED",
      "message": "資産口座名は必須です。"
    },
    {
      "code": "INVALID_BALANCE",
      "message": "月末残高は0以上の整数で指定してください。"
    }
  ]
}
```

---

#### 4.13.4 CSV全体に関するエラー

1ファイル1対象年月違反など、
特定の1行だけではなく
CSV全体に関係する
プレビューエラーが存在する。

その場合は、
行エラーとは別に
CSV全体のエラーを返却できる構造とする。

概念例：

```json
{
  "data": {
    "targetYearMonth": null,
    "canImport": false,
    "errors": [
      {
        "code": "MULTIPLE_TARGET_YEAR_MONTHS",
        "message": "1つのCSVファイルには1つの対象年月のみ指定できます。"
      }
    ],
    "rows": []
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 4.13.5 HTTPエラーとの区別

CSV内容を
正常に解析でき、
業務ルール違反を
プレビュー結果として返却できる場合は、
HTTPエラーとは区別する。

概念的には、

```text
CSV解析成功
+
業務エラーあり
    ↓
200 OK
canImport = false
```

とする。

一方、

```text
file未指定
CSVとして解析不能
X-User-Id不正
```

など、
プレビュー処理そのものを
成立させられない場合は、
APIエラーレスポンスとして返却する。

---

### 4.14 レスポンス項目

正常時の
`data`配下には、
以下の項目を返却する。

| 項目 | 型 | NULL | 内容 |
|---|---|:---:|---|
| `targetYearMonth` | string | ○ | CSVの対象年月。特定できない場合は`null` |
| `canImport` | boolean | × | CSV全体を登録可能か |
| `errors` | array | × | CSV全体に関するエラー |
| `rows` | array | × | CSV各行のプレビュー結果 |

---

#### 4.14.1 targetYearMonth

CSV全体の
対象年月を返却する。

正常例：

```json
"targetYearMonth": "2026-07"
```

CSV内で
対象年月が統一されていないなど、
単一の対象年月として
特定できない場合は、

```json
"targetYearMonth": null
```

としてよい。

---

#### 4.14.2 canImport

CSV全体を
CSV-003へ登録可能な状態として
判定できるかを返却する。

```text
true
    → プレビュー時点では登録可能

false
    → 登録不可となるエラーあり
```

`true`であっても、
CSV-003実行時の
登録成功を保証しない。

---

#### 4.14.3 errors

CSV全体に関する
エラーを配列で返却する。

型の概念例：

```ts
type CsvPreviewError = {
  code: string;
  message: string;
};
```

エラーが存在しない場合は、
`null`ではなく
空配列を返却する。

```json
"errors": []
```

---

#### 4.14.4 rows

CSVの各データ行について、
プレビュー結果を
配列で返却する。

型の概念例：

```ts
type MonthEndAssetBalanceCsvPreviewRow = {
  rowNumber: number;
  assetAccountName: string | null;
  balance: number | null;
  errors: CsvPreviewError[];
};
```

CSV上の並び順を維持して
返却する。

---

#### 4.14.5 rowNumber

CSVファイル上の
行番号を返却する。

ヘッダーを
1行目として扱う。

例えば、
最初のデータ行は、

```json
"rowNumber": 2
```

となる。

---

#### 4.14.6 assetAccountName

CSVへ入力された
資産口座名を返却する。

正常例：

```json
"assetAccountName": "普通預金"
```

入力値確認のための項目であり、
`asset_accounts.id`は返却しない。

必須値が欠落している行については、
`null`または入力値を
プレビュー表示に適した形で返却する。

---

#### 4.14.7 balance

CSVへ入力された
月末残高を返却する。

正常例：

```json
"balance": 1500000
```

正常に整数として
解析できない場合は、

```json
"balance": null
```

として、
対応するエラーを
`errors`へ設定してよい。

---

#### 4.14.8 rows.errors

各CSV行に関する
エラーを配列で返却する。

概念例：

```json
"errors": [
  {
    "code": "ASSET_ACCOUNT_NOT_FOUND",
    "message": "指定された資産口座が存在しません。"
  }
]
```

エラーが存在しない行では、

```json
"errors": []
```

とする。

---

#### 4.14.9 返却しない情報

本APIでは、
以下の内部管理情報は返却しない。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- `month_end_asset_balance_id`
- `balance_recording_unit`
- `confirmed`
- `created_at`
- `updated_at`

また、
他の利用者に属する
資産口座や
月末資産情報を
レスポンスへ含めてはならない。

プレビュー画面で
利用者がCSV内容を確認するために
必要な情報のみを返却する。

---

### 4.15 エラーレスポンス

本APIでは、
エラーを以下の2種類に分けて扱う。

```text
APIリクエスト自体を成立させられないエラー
    ↓
HTTPエラーレスポンス

CSVを解析できるが
登録不可となる業務エラー
    ↓
200 OK
canImport = false
```

CSV内の業務エラーを
すべてHTTPエラーとして返却してはならない。

プレビュー画面で
利用者がCSV内容を確認・修正できるよう、
解析可能なCSVについては
可能な限りプレビュー結果として返却する。

---

#### 4.15.1 エラー一覧

本APIで想定する
主なHTTPエラーは、
以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`が指定されていない |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`の形式が不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 指定された利用者が存在しない、または論理削除済み |
| `422 Unprocessable Entity` | `VALIDATION_ERROR` | `file`未指定、ファイル形式・サイズなどリクエストレベルの入力不正 |
| `422 Unprocessable Entity` | `INVALID_CSV_FORMAT` | CSVとして正常に解析できない、またはヘッダー構造が不正 |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバーエラーが発生した |

CSV行単位の
業務エラーについては、
原則として
上記HTTPエラーには変換せず、
プレビュー結果内の
`errors`として返却する。

---

#### 4.15.2 USER_CONTEXT_REQUIRED

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

この場合、
CSVファイルの解析処理へ
進まない。

---

#### 4.15.3 INVALID_USER_ID

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
0
-1
abc
1.5
```

形式不正の場合、
CSV解析や
業務データ検索は行わない。

---

#### 4.15.4 USER_NOT_FOUND

`X-User-Id`で
指定された利用者が
存在しない場合、
または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

他の利用者の
資産口座や
月末資産状況を使用して
プレビューを続行してはならない。

---

#### 4.15.5 file未指定

`file`が
指定されていない場合は、

```text
VALIDATION_ERROR
```

を返却する。

レスポンス例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "file",
        "reason": "required",
        "message": "CSVファイルを指定してください。"
      }
    ]
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 4.15.6 ファイル形式不正

CSVとして受け付けない
ファイル形式の場合は、

```text
VALIDATION_ERROR
```

として扱う。

例えば、
以下を不正とする。

```text
.xlsx
.xls
.pdf
```

拡張子だけではなく、
HTTPアップロードファイルとして
正常に受信できていることも確認する。

---

#### 4.15.7 ファイルサイズ超過

CSV共通仕様で定める
ファイルサイズ上限を
超えている場合は、

```text
VALIDATION_ERROR
```

を返却する。

CSV内容の解析処理へは
進まない。

具体的な上限値は、
CSV共通仕様の
最新定義に従う。

---

#### 4.15.8 空ファイル

CSVファイルが
0バイトの場合、
または有効なヘッダーを
取得できない場合は、

```text
INVALID_CSV_FORMAT
```

として扱う。

この場合、
行単位のプレビュー結果は
生成しない。

---

#### 4.15.9 CSV解析不能

CSVとして
正常に解析できない場合は、

```text
INVALID_CSV_FORMAT
```

を返却する。

例えば、
以下のような状態を含む。

- CSV構造が破損している
- 引用符の対応が不正
- ヘッダーを正常に取得できない
- 想定した列構造として解析できない

PHP内部の
CSV解析エラーを
そのままレスポンスへ公開しない。

---

#### 4.15.10 CSVヘッダー不正

CSVヘッダーが
期待する形式と一致しない場合は、

```text
INVALID_CSV_FORMAT
```

として扱う。

期待するヘッダーは、
以下とする。

```csv
target_year_month,asset_account_name,balance
```

例えば、
以下は不正とする。

```csv
targetYearMonth,assetAccountName,balance
```

```csv
asset_account_name,target_year_month,balance
```

```csv
target_year_month,asset_account_name,balance,memo
```

ヘッダー不正の場合、
後続行を
正しく解釈できないため、
行単位の業務検証は行わない。

---

#### 4.15.11 データ行0件

ヘッダーは正常だが、
データ行が1件も存在しない場合は、
CSV内容として登録対象がないため、
プレビュー結果として
登録不可とする。

概念例：

```json
{
  "data": {
    "targetYearMonth": null,
    "canImport": false,
    "errors": [
      {
        "code": "CSV_DATA_REQUIRED",
        "message": "登録対象となるCSVデータがありません。"
      }
    ],
    "rows": []
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

CSV構造自体は
正常に解析できているため、
`200 OK`としてよい。

---

#### 4.15.12 CSV全体に関する業務エラー

CSVとして解析可能であるが、
CSV全体に関する
業務ルール違反が存在する場合は、
HTTPエラーではなく
プレビュー結果として返却する。

例えば、

```text
複数のtarget_year_monthが混在
```

している場合は、

```text
200 OK
canImport = false
```

とする。

概念例：

```json
{
  "data": {
    "targetYearMonth": null,
    "canImport": false,
    "errors": [
      {
        "code": "MULTIPLE_TARGET_YEAR_MONTHS",
        "message": "1つのCSVファイルには1つの対象年月のみ指定できます。"
      }
    ],
    "rows": []
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 4.15.13 行単位の入力エラー

以下のような
行単位の入力不正は、
原則として
プレビュー結果内の
`rows[].errors`へ返却する。

- `target_year_month`未入力
- `target_year_month`形式不正
- `asset_account_name`未入力
- `balance`未入力
- `balance`数値形式不正
- `balance`小数
- `balance`負数

概念例：

```json
{
  "rowNumber": 3,
  "assetAccountName": "",
  "balance": null,
  "errors": [
    {
      "code": "ASSET_ACCOUNT_NAME_REQUIRED",
      "message": "資産口座名は必須です。"
    },
    {
      "code": "INVALID_BALANCE",
      "message": "月末残高は0以上の整数で指定してください。"
    }
  ]
}
```

---

#### 4.15.14 資産口座不存在

CSVで指定された
`asset_account_name`に対応する
資産口座が
操作対象利用者に存在しない場合は、
行単位の業務エラーとする。

概念的なエラーコードは、

```text
ASSET_ACCOUNT_NOT_FOUND
```

とする。

他の利用者に
同名資産口座が存在していても、
正常判定しない。

---

#### 4.15.15 残高記録単位不一致

指定された資産口座が
商品単位で
残高を管理する場合は、
月末資産残高CSVの
登録対象ではない。

概念的なエラーコードは、

```text
BALANCE_RECORDING_UNIT_MISMATCH
```

とする。

商品別月末評価額CSVの
利用を前提とする。

---

#### 4.15.16 対象年月時点で資産口座が無効

指定された資産口座が
CSVの`target_year_month`時点で
記録対象として
有効でない場合は、
行単位の業務エラーとする。

概念的なエラーコードは、

```text
ASSET_ACCOUNT_NOT_AVAILABLE
```

とする。

具体的な名称は、
既存のエラーコード体系に
合わせて統一する。

---

#### 4.15.17 確定済み月末資産状況

対象年月の
月末資産状況が

```text
confirmed = true
```

の場合は、
CSV全体を
登録不可とする。

概念的なエラーコードは、

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

とする。

この場合でも、
CSV解析自体が
正常に完了しているため、

```text
200 OK
canImport = false
```

として
プレビュー結果へ
エラーを含めて返却してよい。

---

#### 4.15.18 既存月末資産残高

同一対象年月、
同一資産口座について
既に月末資産残高が存在し、
CSV-003が新規登録のみを
許可する仕様である場合は、
重複エラーとする。

概念的なエラーコードは、

```text
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

とする。

既存レコードを
プレビュー処理によって
上書きしない。

---

#### 4.15.19 CSV内重複

同一CSV内に
同じ資産口座が
複数行存在する場合は、
CSV内重複として扱う。

概念的なエラーコードは、

```text
DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

とする。

先勝ち、
後勝ちなどによる
自動解決は行わない。

---

#### 4.15.20 複数エラー

CSV構造が解析可能な場合は、
可能な範囲で
複数のエラーを収集して
1回のレスポンスで返却する。

例えば、

```text
2行目
ASSET_ACCOUNT_NOT_FOUND

4行目
INVALID_BALANCE

5行目
DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

のように、
複数箇所をまとめて
利用者へ提示できるようにする。

---

#### 4.15.21 INTERNAL_SERVER_ERROR

CSV解析処理、
データベース検索、
プレビュー生成処理などで
想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスには、
以下の内部情報を含めない。

- SQL
- テーブル名
- カラム名
- PostgreSQLのエラー内容
- 制約名
- PHP内部エラー
- Laravel内部例外
- スタックトレース
- サーバーファイルパス

詳細は、
サーバーログへ記録する。

---

### 4.16 HTTPステータス

本APIで使用する
HTTPステータスは、
以下とする。

| HTTPステータス | 用途 |
|---|---|
| `200 OK` | CSVプレビュー処理成功。業務エラーを含む場合も使用する |
| `400 Bad Request` | 利用者コンテキストの指定不備 |
| `404 Not Found` | 指定された利用者が存在しない |
| `422 Unprocessable Entity` | ファイル未指定、ファイル形式不正、CSV構造不正など |
| `500 Internal Server Error` | 想定外のサーバー内部エラー |

---

#### 4.16.1 200 OK

以下の場合は、

```text
200 OK
```

とする。

- CSVが登録可能
- CSV内に行単位の業務エラーが存在する
- CSV全体に業務エラーが存在する
- データ行が0件
- 既存月末資産残高との重複が存在する
- 確定済み月末資産状況で登録できない

CSVを解析して
プレビュー結果を
返却できることを
HTTPレベルの成功とする。

登録可否は、

```text
canImport
```

で表現する。

---

#### 4.16.2 400 Bad Request

以下の場合に使用する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
```

CSVファイルの
業務内容には使用しない。

---

#### 4.16.3 404 Not Found

指定された利用者が
存在しない、
または論理削除済みの場合に使用する。

```text
USER_NOT_FOUND
```

CSV内で
資産口座が存在しない場合には、
`404 Not Found`を使用しない。

その場合は
プレビュー結果内の
業務エラーとして扱う。

---

#### 4.16.4 422 Unprocessable Entity

プレビュー処理自体を
正常に成立させられない
入力不正に使用する。

主に以下を対象とする。

- `file`未指定
- アップロードファイル不正
- ファイルサイズ超過
- CSVとして解析不能
- CSVヘッダー不正

行単位の
業務エラーには
原則として使用しない。

---

#### 4.16.5 500 Internal Server Error

想定外のサーバー内部エラーが
発生した場合に使用する。

対象となるエラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

---

### 4.17 副作用

本APIには、
業務データに対する
副作用はない。

CSV内容の検証のために
データベースを参照するが、
以下の操作は行わない。

```text
INSERT
UPDATE
DELETE
```

特に、
以下の処理を行わない。

- 月末資産状況の新規作成
- 月末資産残高の登録
- 月末資産残高の更新
- 月末資産残高の削除
- 月末資産状況の確定
- 月末資産状況の確定解除
- 資産口座の更新

CSV-002を
何度実行しても、
業務データの状態は変化しない。

---

#### 4.17.1 プレビュー結果を保存しない

プレビュー結果は、
Phase1では
データベースへ保存しない。

以下のような
プレビュー管理データも
作成しない。

```text
preview_id
import_session_id
preview_token
```

CSV-003では、
CSVファイルを
再度受け取り、
同じ業務ルールを
再検証する。

---

#### 4.17.2 月末資産状況を作成しない

対象年月の
月末資産状況が存在しない場合でも、
CSV-002では
`month_end_asset_snapshots`を
作成しない。

プレビュー結果として
登録可能性を判定するだけとする。

必要な月末資産状況の作成は、
CSV-003の登録処理で行う仕様とする場合に、
CSV-003側で実行する。

---

### 4.18 トランザクション

本APIは、
業務データを更新しないため、
Phase1では
更新トランザクションを使用しない。

概念的な処理は、
以下とする。

```text
CSV解析
    ↓
業務データ参照
    ↓
エラー収集
    ↓
プレビュー結果生成
```

以下のような
更新トランザクションは使用しない。

```php
DB::transaction(
    function () {
        // CSVプレビューのみ
    },
);
```

---

#### 4.18.1 複数SELECT

CSVプレビューでは、
以下のような
複数の参照処理を行う可能性がある。

- 資産口座検索
- 月末資産状況検索
- 既存月末資産残高検索

Phase1では、
これらの参照だけを理由に
明示的なトランザクションを
開始しない。

---

#### 4.18.2 CSV-003との差異

CSV-002は、
参照・検証のみであるため、

```text
トランザクション不要
```

とする。

CSV-003では、
複数行の一括登録を
原子的に行う必要があるため、

```text
トランザクション必要
```

とする。

```text
CSV-002
    → 検証のみ

CSV-003
    → 検証
       +
       一括登録
       +
       ロールバック
```

と責務を分ける。

---

### 4.19 ロック

本APIでは、
行ロックを使用しない。

以下のような
排他ロックは行わない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

CSV-002は
登録処理ではなく、
現在時点の状態を確認する
プレビューAPIである。

プレビューによって
他の月末資産関連処理を
ブロックしてはならない。

---

#### 4.19.1 プレビュー後の状態変更

CSV-002実行後に、
別処理によって

- 月末資産状況が確定される
- 月末資産残高が登録される
- 資産口座の状態が変更される

可能性がある。

CSV-002では
これらを防ぐための
ロックを保持しない。

そのため、
CSV-003実行時に
必ず再検証する。

---

### 4.20 キャッシュ

Phase1では、
CSV-002専用の
サーバー側アプリケーションキャッシュを
使用しない。

プレビュー結果は、
以下の状態によって
変化する可能性がある。

- 資産口座の状態
- 月末資産状況の存在
- 月末資産状況の確定状態
- 既存月末資産残高

そのため、
プレビュー実行ごとに
最新の業務データを参照する。

---

#### 4.20.1 プレビュー結果のキャッシュ

CSVファイル内容をキーとして
プレビュー結果を
キャッシュする方式は、
Phase1では採用しない。

同一CSVであっても、
業務データの状態が変化すれば
登録可否も変化するためである。

例えば、

```text
1回目
confirmed = false
    ↓
canImport = true

その後
confirmed = true

2回目
同じCSV
    ↓
canImport = false
```

となり得る。

---

### 4.21 冪等性

本APIは、
業務データの状態を変更しないため、
同一のシステム状態では
冪等に扱える。

同一利用者、
同一CSVファイル、
同一のデータベース状態で
複数回実行した場合は、
同じプレビュー結果となる。

例えば、

```text
1回目
CSV-002
    ↓
canImport = true

2回目
同一CSV・同一DB状態
    ↓
canImport = true
```

となる。

本APIの実行によって、
月末資産残高などが
登録されることはない。

---

#### 4.21.1 POSTと冪等性

本APIは、
HTTPメソッドとして
`POST`を使用する。

これは、
CSVファイルを
リクエストボディとして送信し、
検証処理を実行するためである。

ただし、
業務データを変更しないため、
アプリケーション上の処理としては
安全に再実行できる。

---

#### 4.21.2 Idempotency-Key

本APIでは、
`Idempotency-Key`を使用しない。

CSV-002は
登録APIではなく、
重複実行によって
業務データの重複登録が
発生しないためである。

---

#### 4.21.3 再実行

ネットワークエラーなどによって
クライアントが
プレビュー結果を
受信できなかった場合でも、
同じCSVを再送してよい。

```text
CSV-002
    ↓
通信エラー
    ↓
同じCSVを再送
    ↓
再プレビュー
```

再送によって
業務データが変更されることはない。

---

#### 4.21.4 データ状態変更後の再実行

CSV-002の実行間に
業務データが変更された場合は、
同じCSVでも
プレビュー結果が変化する可能性がある。

例えば、

```text
1回目

2026-07
confirmed = false
    ↓
canImport = true
```

その後、

```text
2026-07
confirmed = true
```

となった場合、

```text
2回目
同じCSV
    ↓
canImport = false
```

となる。

これは、
CSV-002自体の副作用によるものではなく、
参照対象となる
業務データが変更されたためである。

---

### 4.22 関連テーブル

CSV-002では、
アップロードされた
月末資産残高CSVの内容を
業務ルールに照らして検証するため、
以下のテーブルを参照する。

| テーブル | 用途 |
|---|---|
| `users` | 操作対象利用者の存在確認 |
| `asset_accounts` | CSVで指定された資産口座の存在・利用者境界・残高記録単位確認 |
| `month_end_asset_snapshots` | 対象年月の月末資産状況・確定状態確認 |
| `month_end_asset_balances` | 既存の月末資産残高との重複確認 |

本APIはプレビュー専用APIであるため、
これらのテーブルを
更新しない。

---

#### 4.22.1 users

`X-User-Id`で指定された
操作対象利用者の
存在確認に使用する。

概念的な条件は、
以下とする。

```text
users.id = X-User-Id
AND
users.deleted_at IS NULL
```

---

### 4.25 Laravel実装方針

### 4.25 Laravel実装方針

CSV-002では、
Action、
Request、
UseCase、
CSV Definition、
CSV Parser、
CSV Validator、
Query、
DTO、
API Resource、
Responderを分離して実装する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
Middleware
    ↓
Request
    ↓
Action
    ↓
UseCase
    ├─ CSV Parser
    ├─ CSV Validator
    ├─ AssetAccountQuery
    ├─ MonthEndAssetSnapshotQuery
    └─ MonthEndAssetBalanceQuery
    ↓
Preview DTO
    ↓
API Resource
    ↓
Responder
```

CSVプレビューに関するユースケース制御は、UseCaseへ集約する。

Actionへ、以下の処理を直接記述しない。

- CSV解析
- CSVヘッダー検証
- 行単位バリデーション
- 資産口座検索
- 月末資産状況検索
- 既存月末資産残高検索
- 業務ルール判定
- `canImport`判定

---

#### 4.25.1 Action

HTTPリクエストを受け付け、検証済みのCSVファイルおよび利用者コンテキストを取得する。

CSVプレビューUseCaseを呼び出し、取得結果をResponderへ渡す。

概念例：

```php
final class PreviewMonthEndAssetBalanceCsvAction
{
    public function __invoke(
        PreviewMonthEndAssetBalanceCsvRequest $request,
        PreviewMonthEndAssetBalanceCsvUseCase $useCase,
        MonthEndAssetBalanceCsvPreviewResponder $responder,
        userContext $userContext,
    ): JsonResponse {
        $preview =
            $useCase->execute(
                userId:
                    $userContext->userId,

                file:
                    $request->file('file'),
            );

        return $responder->ok(
            $preview,
        );
    }
}
```

Actionでは、以下を行わない。

- CSVヘッダー検証
- CSV行解析
- `target_year_month`の検証
- `asset_account_name`の検証
- `balance`の検証
- 1ファイル1対象年月の判定
- 資産口座検索
- 残高記録単位判定
- 対象年月時点の資産口座有効性判定
- 月末資産状況検索
- 確定状態判定
- 既存月末資産残高検索
- CSV内重複判定
- `canImport`判定
- レスポンス形式への変換

Actionは、UseCaseの呼び出しとResponderへの受け渡しに責務を限定する。

---

#### 4.25.2 Request

Requestでは、HTTPリクエストとしてCSVファイルを受け付けられる状態かを検証する。

概念例：

```php
final class PreviewMonthEndAssetBalanceCsvRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'file' => [
                'required',
                'file',
                'mimes:csv,txt',
                'max:' . config(
                    'csv.max_file_size_kb',
                ),
            ],
        ];
    }
}
```

実際の以下の扱いは、CSV共通仕様に従う。

- ファイルサイズ上限
- MIME Type
- 拡張子

Laravelのアップロードファイル検証だけに依存せず、CSV Parserでも実際にCSVとして解析可能であることを確認する。

---

#### 4.25.3 Requestで行うこと

Requestでは、主に以下を検証する。

- `file`が指定されていること
- HTTPアップロードファイルであること
- 許可されたファイル形式であること
- ファイルサイズ上限以内であること

これらは、CSV内容を解析する前に判定できるHTTPリクエストレベルのバリデーションとする。

---

#### 4.25.4 Requestで行わないこと

Requestでは、以下のCSV内容および業務ルール検証を行わない。

- CSVヘッダー検証
- CSVデータ行の存在確認
- `target_year_month`必須確認
- `target_year_month`形式確認
- 1ファイル1対象年月確認
- `asset_account_name`必須確認
- `balance`必須確認
- `balance`整数確認
- `balance`0以上確認
- 資産口座存在確認
- 残高記録単位確認
- 対象年月時点の資産口座有効性確認
- 月末資産状況確認
- 確定状態確認
- 既存月末資産残高確認
- CSV内重複確認

これらは、UseCase、CSV Validator、Queryの責務とする。

---

#### 4.25.5 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONレスポンス共通処理
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
Request
    ↓
Action
```

CSV解析処理へ進む前に、有効な操作対象利用者が確定していることを前提とする。

---

#### 4.25.6 UseCase

CSVプレビューのユースケース処理を担当する。

概念例：

```php
final class PreviewMonthEndAssetBalanceCsvUseCase
{
    public function execute(
        int $userId,
        UploadedFile $file,
    ): MonthEndAssetBalanceCsvPreview {
        // CSV解析
        // CSV構造検証
        // 行入力値検証
        // 対象年月特定
        // 業務データ取得
        // 業務ルール検証
        // Preview DTO生成
    }
}
```

UseCaseでは、概念的に以下の順序で処理する。

```text
CSV解析
    ↓
ヘッダー検証
    ↓
データ行抽出
    ↓
行入力値検証
    ↓
対象年月特定
    ↓
CSV内重複確認
    ↓
必要な業務データ一括取得
    ↓
業務ルール検証
    ↓
CSV全体エラー集約
    ↓
行エラー集約
    ↓
canImport判定
    ↓
Preview DTO生成
```

UseCaseでは、SQLやEloquent Query Builderを直接組み立てない。

データベースアクセスは、Queryへ委譲する。

---

#### 4.25.7 CSV Definition

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
}
```

CSV-002では、このDefinitionを使用してヘッダーを検証する。

以下のように、CSV-001、CSV-002、CSV-003で個別にヘッダーを定義しない。

```text
CSV-001
    ↓
MonthEndAssetBalanceCsvDefinition

CSV-002
    ↓
MonthEndAssetBalanceCsvDefinition

CSV-003
    ↓
MonthEndAssetBalanceCsvDefinition
```

---

#### 4.25.8 CSV Parser

CSVファイルの読み込みは、専用Parserへ分離する。

概念例：

```php
final class MonthEndAssetBalanceCsvParser
{
    public function parse(
        UploadedFile $file,
    ): ParsedCsv {
        // CSV解析
    }
}
```

Parserは、CSVファイルを構造化された生データへ変換する責務を持つ。

Parserでは、業務データの検索や登録可否判定を行わない。

---

#### 4.25.9 CSV Parserの責務

CSV Parserでは、主に以下を行う。

- ファイルオープン
- BOM除去
- CSV行読み込み
- ヘッダー取得
- 行番号管理
- 完全な空行の除外
- CSVとして解析不能な状態の検出

Parserは、各行を概念的に以下の形式へ変換する。

```php
final readonly class ParsedCsvRow
{
    public function __construct(
        public int $rowNumber,

        /** @var array<string, string|null> */
        public array $values,
    ) {
    }
}
```

CSV上の行番号を保持することで、プレビュー結果の`rowNumber`として利用できるようにする。

---

#### 4.25.10 CSV構造異常

以下のように、CSVそのものを正常に解析できない場合は、プレビュー結果ではなくHTTPエラーとして扱う。

- 空ファイル
- ヘッダー取得不能
- CSV解析不能
- 想定外の列構造

概念的には、

```text
CSV Parser
    ↓
InvalidCsvFormatException
    ↓
共通Exception Handler
    ↓
422 Unprocessable Entity
INVALID_CSV_FORMAT
```

とする。

---

#### 4.25.11 CSVヘッダー検証

CSVヘッダーは、共通Definitionと完全一致することを確認する。

概念例：

```php
if (
    $parsedCsv->headers
    !== MonthEndAssetBalanceCsvDefinition::HEADERS
) {
    throw new InvalidCsvFormatException();
}
```

以下を不正とする。

- ヘッダー不足
- ヘッダー名不一致
- ヘッダー順序不一致
- 余分なヘッダー

ヘッダーが不正な場合は、後続の行検証へ進まない。

---

#### 4.25.12 CSV Validator

CSV各行の入力値検証は、専用Validatorへ分離する。

概念例：

```php
final class MonthEndAssetBalanceCsvValidator
{
    public function validateRow(
        ParsedCsvRow $row,
    ): CsvRowValidationResult {
        // 行入力値検証
    }
}
```

主に以下を確認する。

```text
target_year_month
    required
    YYYY-MM

asset_account_name
    required

balance
    required
    integer
    min:0
```

CSV Validatorでは、データベース検索を行わない。

---

#### 4.25.13 行入力値の正規化

CSV入力値は、業務データ検索やDTO生成に使用する前に、必要な型へ変換する。

例えば、`balance`は正常な整数形式の場合のみintegerへ変換する。

概念例：

```php
$balance =
    filter_var(
        $rawBalance,
        FILTER_VALIDATE_INT,
    );
```

PHPの暗黙的な型変換によって、

```text
1000abc
```

などを正常値として扱ってはならない。

---

#### 4.25.14 0円の扱い

`balance = 0`は、有効な月末残高として扱う。

以下のようなtruthy / falsy判定を使用しない。

```php
if (! $balance) {
    // 0円まで未入力扱いになるため使用しない
}
```

必須判定と数値判定を明確に分離する。

---

#### 4.25.15 データ行0件

CSVヘッダーは正常だが、データ行が1件も存在しない場合は、CSV構造異常として例外にはしない。

CSV全体エラーとして、

```text
CSV_DATA_REQUIRED
```

を生成する。

概念的には、

```text
CSV解析成功
+
データ行0件
    ↓
200 OK
canImport = false
```

とする。

---

#### 4.25.16 対象年月の特定

各行の`target_year_month`が正常な形式の場合、CSV内の対象年月を収集する。

概念例：

```php
$targetYearMonths =
    collect($validRows)
        ->pluck(
            'targetYearMonth',
        )
        ->unique()
        ->values();
```

2種類以上存在する場合は、CSV全体エラーとして

```text
MULTIPLE_TARGET_YEAR_MONTHS
```

を生成する。

対象年月を1件に特定できない場合は、Preview DTOの

```text
targetYearMonth
```

を`null`としてよい。

---

#### 4.25.17 CSV内重複判定

同一CSV内で同じ対象年月かつ同じ資産口座が複数回指定されていないか確認する。

概念的なキーは、以下とする。

```text
targetYearMonth
+
assetAccountName
```

1ファイル1対象年月が成立している場合は、実質的に`assetAccountName`単位で重複確認してよい。

重複時は、

```text
DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

相当の行エラーを生成する。

先勝ち、後勝ち、残高合算などの暗黙的な解決は行わない。

---

#### 4.25.18 AssetAccountQuery

CSV内で指定された資産口座名について、操作対象利用者に属する資産口座をまとめて取得する。

概念例：

```php
$assetAccounts =
    $this->assetAccountQuery
        ->findActiveByNames(
            userId: $userId,
            names: $assetAccountNames,
        );
```

検索条件は、概念的に以下とする。

```text
user_id = 操作対象利用者ID
AND
name IN (...)
AND
deleted_at IS NULL
```

他の利用者に属する同名資産口座を取得対象へ含めない。

---

#### 4.25.19 資産口座のMap化

取得した資産口座は、名前をキーとしてMap化してよい。

概念例：

```php
$assetAccountMap =
    $assetAccounts->keyBy(
        'name',
    );
```

各CSV行では、

```php
$assetAccount =
    $assetAccountMap->get(
        $row->assetAccountName,
    );
```

として参照する。

CSV行ごとに同じ資産口座検索SQLを発行しない。

---

#### 4.25.20 資産口座不存在

操作対象利用者に該当する資産口座が存在しない場合は、HTTP 404にはしない。

行単位のプレビューエラーとして、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を生成する。

他の利用者にのみ同名資産口座が存在する場合も、同じ扱いとする。

---

#### 4.25.21 残高記録単位の判定

取得した資産口座について、`balance_recording_unit`が口座単位であることを確認する。

概念例：

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::ACCOUNT
) {
    // BALANCE_RECORDING_UNIT_MISMATCH
}
```

実際のEnum値は、共通定義に従う。

商品単位の資産口座は、月末資産残高CSVの登録対象としない。

---

#### 4.25.22 対象年月時点の資産口座有効性

資産口座がCSVの`targetYearMonth`時点で月末資産残高の記録対象として有効かを判定する。

現在日時を基準に判定してはならない。

必ず、

```text
CSVのtargetYearMonth
```

を基準とする。

具体的な判定方法は、資産口座および関連するテーブル定義・機能要件に従う。

対象年月時点で無効な場合は、

```text
ASSET_ACCOUNT_NOT_AVAILABLE
```

相当の行エラーを生成する。

---

#### 4.25.23 MonthEndAssetSnapshotQuery

対象年月を単一に特定できた場合は、操作対象利用者に属する月末資産状況を取得する。

概念例：

```php
$snapshot =
    $this->monthEndAssetSnapshotQuery
        ->findByUserAndTargetYearMonth(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,
        );
```

検索条件は、以下とする。

```text
user_id = 操作対象利用者ID
AND
target_year_month = targetYearMonth
```

他利用者の同一対象年月データを取得しない。

---

#### 4.25.24 月末資産状況不存在

対象年月の月末資産状況が存在しない場合は、CSV-003と同一の業務ルールで判定する。

CSV-003で登録時に月末資産状況を作成する仕様であれば、CSV-002では不存在のみを理由としてプレビューエラーを生成しない。

ただし、CSV-002では実際に

```text
month_end_asset_snapshots
```

を作成しない。

---

#### 4.25.25 確定状態判定

対象年月の月末資産状況が存在する場合は、`confirmed`を確認する。

概念例：

```php
if (
    $snapshot !== null
    && $snapshot->confirmed
) {
    // CSV全体を登録不可とする
}
```

確定済みの場合は、CSV全体エラーとして

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

を生成する。

この状態は、HTTPエラーとしてではなく、

```text
200 OK
canImport = false
```

のプレビュー結果として扱う。

---

#### 4.25.26 MonthEndAssetBalanceQuery

対象snapshotが既に存在する場合は、CSV対象資産口座について既存月末資産残高をまとめて取得する。

概念例：

```php
$existingBalances =
    $this->monthEndAssetBalanceQuery
        ->findBySnapshotAndAssetAccounts(
            userId:
                $userId,

            snapshotId:
                $snapshot->id,

            assetAccountIds:
                $assetAccountIds,
        );
```

`snapshotId`だけではなく、操作対象利用者との利用者境界も保証する。

---

#### 4.25.27 既存残高のMap化

既存の月末資産残高は、`asset_account_id`をキーとしてMap化してよい。

概念例：

```php
$existingBalanceMap =
    $existingBalances->keyBy(
        'asset_account_id',
    );
```

各CSV行について、既存データの有無をメモリ上で判定する。

---

#### 4.25.28 既存月末資産残高

同一対象年月、同一資産口座について既に月末資産残高が存在し、CSV-003が新規登録のみを許可する仕様の場合は、

```text
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

相当の行エラーを生成する。

CSV-002では、既存レコードを以下のように変更しない。

- 更新
- 削除
- 上書き

---

#### 4.25.29 N+1問題

CSV-002では、CSV行ごとに個別SQLを発行してはならない。

以下のような構成を避ける。

```text
1行目
    ↓
asset_accounts検索
snapshot検索
balance検索

2行目
    ↓
asset_accounts検索
snapshot検索
balance検索

3行目
    ↓
...
```

基本的には、

```text
CSV全体解析
    ↓
必要な資産口座名収集
    ↓
資産口座一括取得
    ↓
snapshot 1件取得
    ↓
既存残高一括取得
    ↓
メモリ上で業務検証
```

とする。

---

#### 4.25.30 CsvPreviewError DTO

プレビューで使用するエラーは、専用DTOとして表現する。

概念例：

```php
final readonly class CsvPreviewError
{
    public function __construct(
        public string $code,
        public string $message,
    ) {
    }
}
```

CSV全体エラーと行エラーで同じ構造を使用してよい。

---

#### 4.25.31 Preview Row DTO

各CSV行のプレビュー結果は、専用DTOとして表現する。

概念例：

```php
final readonly class MonthEndAssetBalanceCsvPreviewRow
{
    public function __construct(
        public int $rowNumber,
        public ?string $assetAccountName,
        public ?int $balance,

        /** @var CsvPreviewError[] */
        public array $errors,
    ) {
    }
}
```

CSV内部の以下の項目などは、Preview Row DTOへ含めない。

- `asset_account_id`
- `month_end_asset_snapshot_id`

---

#### 4.25.32 Preview DTO

プレビュー結果全体は、以下のようなDTOとして表現する。

概念例：

```php
final readonly class MonthEndAssetBalanceCsvPreview
{
    public function __construct(
        public ?string $targetYearMonth,
        public bool $canImport,

        /** @var CsvPreviewError[] */
        public array $errors,

        /** @var MonthEndAssetBalanceCsvPreviewRow[] */
        public array $rows,
    ) {
    }
}
```

Eloquent ModelやQuery結果をそのままResponderへ渡さない。

---

#### 4.25.33 canImportの判定

`canImport`は、CSV全体エラーおよび各行エラーをもとにUseCaseで判定する。

概念例：

```php
$hasGlobalErrors =
    count(
        $globalErrors,
    ) > 0;

$hasRowErrors =
    collect(
        $rows,
    )->contains(
        static fn (
            MonthEndAssetBalanceCsvPreviewRow $row,
        ): bool =>
            count(
                $row->errors,
            ) > 0,
    );

$canImport =
    ! $hasGlobalErrors
    && ! $hasRowErrors;
```

正常行が1件以上存在していても、エラーが1件でも存在する場合は、

```text
canImport = false
```

とする。

---

#### 4.25.34 複数エラー収集

CSV構造が正常に解析可能な場合は、可能な範囲で複数エラーを収集する。

そのため、1行目の業務エラーで即座に例外を送出して処理全体を終了する方式は基本としない。

以下のような実装は避ける。

```php
throw new AssetAccountNotFoundException();
```

代わりに、プレビュー用エラーDTOへエラーを追加する。

概念的には、

```text
2行目
    ASSET_ACCOUNT_NOT_FOUND

4行目
    INVALID_BALANCE

6行目
    DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

を1回のレスポンスで返却できるようにする。

---

#### 4.25.35 HTTPエラーとプレビューエラーの分離

以下は、HTTPエラーとして扱う。

- 利用者コンテキスト不正
- `file`未指定
- アップロードファイル不正
- ファイルサイズ超過
- CSV解析不能
- CSVヘッダー不正
- 想定外例外

一方、以下はプレビュー結果内の業務エラーとして扱う。

- データ行0件
- `target_year_month`不正
- 対象年月混在
- `asset_account_name`不正
- `balance`不正
- 資産口座不存在
- 残高記録単位不一致
- 対象年月時点の資産口座無効
- CSV内重複
- 確定済み月末資産状況
- 既存月末資産残高

UseCaseでは、この2種類を明確に分離する。

---

#### 4.25.36 API Resource

Preview DTOを、専用API ResourceによってAPIレスポンス形式へ変換する。

概念例：

```php
final class MonthEndAssetBalanceCsvPreviewResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'targetYearMonth'
                => $this->targetYearMonth,

            'canImport'
                => $this->canImport,

            'errors'
                => CsvPreviewErrorResource::collection(
                    $this->errors,
                ),

            'rows'
                => MonthEndAssetBalanceCsvPreviewRowResource::collection(
                    $this->rows,
                ),
        ];
    }
}
```

JSONフィールド名は、API共通方針に従ってcamelCaseとする。

---

#### 4.25.37 Row Resource

行単位の結果も、専用Resourceへ変換する。

概念例：

```php
final class MonthEndAssetBalanceCsvPreviewRowResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'rowNumber'
                => $this->rowNumber,

            'assetAccountName'
                => $this->assetAccountName,

            'balance'
                => $this->balance,

            'errors'
                => CsvPreviewErrorResource::collection(
                    $this->errors,
                ),
        ];
    }
}
```

---

#### 4.25.38 Error Resource

プレビュー用エラーも、専用Resourceへ変換する。

概念例：

```php
final class CsvPreviewErrorResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'code'
                => $this->code,

            'message'
                => $this->message,
        ];
    }
}
```

CSV内部で使用した技術的な情報をエラーResourceへ含めない。

---

#### 4.25.39 返却しない情報

API Resourceでは、以下の内部情報を返却しない。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- `month_end_asset_balance_id`
- `balance_recording_unit`
- `confirmed`
- `created_at`
- `updated_at`

Queryで取得したEloquent ModelをそのままJSON化しない。

---

#### 4.25.40 Responder

Responderは、生成済みPreview DTOを受け取り、API共通の成功Envelope形式へ変換する。

概念例：

```php
final class MonthEndAssetBalanceCsvPreviewResponder
{
    public function ok(
        MonthEndAssetBalanceCsvPreview $preview,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new MonthEndAssetBalanceCsvPreviewResource(
                        $preview,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの共通項目は、API共通レスポンス処理に従う。

---

#### 4.25.41 Responderの責務

Responderでは、以下を行わない。

- CSV解析
- CSVヘッダー検証
- 行入力値検証
- 対象年月特定
- CSV内重複判定
- 資産口座検索
- snapshot検索
- 既存残高検索
- 業務ルール判定
- `canImport`判定

Responderは、生成済みPreview DTOをHTTPレスポンスへ変換することに責務を限定する。

---

#### 4.25.42 Repository

CSV-002では、Repositoryを使用しない。

本APIは業務データを更新しない参照・検証専用APIであるため、以下はQueryクラスが担当する。

- 資産口座取得
- 月末資産状況取得
- 既存月末資産残高取得

登録・更新・削除処理は存在しないため、CSV-002専用Repositoryは作成しない。

---

#### 4.25.43 トランザクション

CSV-002では、業務データを更新しないため、明示的な`DB::transaction()`を使用しない。

以下のような実装は行わない。

```php
DB::transaction(
    function () {
        // CSVプレビューのみ
    },
);
```

CSV-003では、複数行を原子的に登録するためトランザクションを使用するが、CSV-002では不要とする。

---

#### 4.25.44 ロック

CSV-002では、行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

CSV-002からCSV-003までの間、DB状態をロックして登録可否を保証する設計は採用しない。

CSV-003実行時に最新状態で再検証する。

---

#### 4.25.45 プレビュー結果の保存

Phase1では、Preview DTOやプレビュー済みCSVをデータベースへ保存しない。

以下のような一時管理は行わない。

```text
CSV-002
    ↓
previewId生成
    ↓
DBへプレビュー結果保存
    ↓
CSV-003でpreviewId指定
```

CSV-003では、CSVファイルを再送し、同じ業務ルールを再検証する。

---

#### 4.25.46 CSV-003との共通化

CSV-002とCSV-003では、CSV解析および登録可否判定ロジックを可能な限り共通化する。

例えば、以下を共通利用する。

```text
MonthEndAssetBalanceCsvDefinition
MonthEndAssetBalanceCsvParser
MonthEndAssetBalanceCsvValidator
MonthEndAssetBalanceCsvImportValidator
```

CSV-002とCSV-003で同じ業務ルールを個別実装しない。

---

#### 4.25.47 CSV-003との責務差

CSV-002とCSV-003の主な違いは、検証後に登録を行うかである。

```text
CSV-002

CSV解析
    ↓
検証
    ↓
Preview DTO
    ↓
終了
```

```text
CSV-003

CSV解析
    ↓
再検証
    ↓
トランザクション開始
    ↓
必要に応じてsnapshot作成
    ↓
月末資産残高一括登録
    ↓
commit
```

CSV-002で

```text
canImport = true
```

となっていても、CSV-003では必ず再検証する。

---

#### 4.25.48 キャッシュ

Phase1では、CSV-002専用のサーバー側アプリケーションキャッシュを使用しない。

同一CSVファイルであっても、以下が変更されればプレビュー結果も変化する。

- 資産口座の状態
- 月末資産状況の存在
- `confirmed`
- 既存月末資産残高

そのため、プレビュー実行ごとに最新の業務データを参照する。

---

#### 4.25.49 ログ

CSV-002では、必要に応じて以下の情報をログコンテキストへ設定する。

```text
requestId
userId
apiId
fileName
rowCount
```

`apiId`は、

```text
CSV-002
```

とする。

以下の内容は、不要にログへ出力しない。

- CSVファイル全文
- 資産口座名の全件
- 月末残高の全件

プレビューエラーについても、通常の入力エラーを大量にエラーログへ出力しない。

---

#### 4.25.50 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `file`未指定・ファイル検証不正 | `VALIDATION_ERROR` |
| CSV解析不能 | `INVALID_CSV_FORMAT` |
| CSVヘッダー不正 | `INVALID_CSV_FORMAT` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

CSV行単位の入力・業務エラーは、原則として例外へ変換しない。

Preview DTOの`errors`へ格納する。

---

#### 4.25.51 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- SQL
- テーブル名
- カラム名
- PostgreSQLの制約名
- PostgreSQL内部エラー
- スタックトレース
- PHP内部エラー
- Laravel内部例外メッセージ
- サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

#### 4.25.52 テスト実装方針

Laravel側では、Feature Testを中心としてCSV-002のAPI契約およびプレビューフロー全体を確認する。

主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `404 Not Found`
- `422 Unprocessable Entity`
- `500 Internal Server Error`
- `file`必須
- ファイル形式
- ファイルサイズ
- CSVヘッダー
- CSVヘッダー順序
- 余分なヘッダー
- データ行0件
- `target_year_month`必須
- `target_year_month`形式
- 1ファイル1対象年月
- `asset_account_name`必須
- 資産口座不存在
- 他利用者の同名資産口座を参照しないこと
- 残高記録単位
- 対象年月時点の資産口座有効性
- `balance`必須
- `balance`整数
- `balance`0以上
- `balance = 0`
- 月末資産状況不存在
- 確定済み月末資産状況
- 既存月末資産残高
- CSV内重複
- 複数エラー収集
- `rowNumber`
- `targetYearMonth`
- `canImport`
- 業務データ非更新
- 冪等性

---

#### 4.25.53 CSV ParserのUnit Test

CSV Parserについて、以下を確認する。

- 正しいヘッダーを取得できること
- データ行を正しく解析できること
- CSV上の行番号を保持できること
- BOMを仕様どおり処理できること
- 完全な空行を仕様どおり扱うこと
- `,,`をデータ行として扱えること
- CSV解析不能時に適切な例外となること
- ヘッダー不正時に後続の業務検証へ進まないこと

---

#### 4.25.54 CSV ValidatorのUnit Test

CSV Validatorについて、以下を個別に確認する。

```text
target_year_month
    required
    YYYY-MM

asset_account_name
    required

balance
    required
    integer
    min:0
```

特に、

```text
balance = 0
```

を正常値として必ずテストする。

---

#### 4.25.55 業務検証のUnit Test

CSV内容と業務データを照合する検証ロジックについて、以下を確認する。

```text
資産口座あり
    → 正常

資産口座なし
    → ASSET_ACCOUNT_NOT_FOUND

他利用者にのみ同名口座あり
    → ASSET_ACCOUNT_NOT_FOUND

商品単位口座
    → BALANCE_RECORDING_UNIT_MISMATCH

対象年月時点で口座無効
    → ASSET_ACCOUNT_NOT_AVAILABLE

snapshot不存在
    → CSV-003の仕様に従った判定

snapshot未確定
    → 登録可能

snapshot確定済み
    → MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED

既存balanceあり
    → MONTH_END_ASSET_BALANCE_ALREADY_EXISTS

CSV内同一口座重複
    → DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

---

#### 4.25.56 QueryのDatabase Test

Queryクラスについて、必要に応じてDatabase Testを行う。

AssetAccountQueryでは、以下を確認する。

```text
user_id一致
+
name一致
+
deleted_at IS NULL
    ↓
取得できる
```

```text
他利用者
+
name一致
    ↓
取得できない
```

MonthEndAssetSnapshotQueryでは、以下を確認する。

```text
user_id一致
+
target_year_month一致
    ↓
取得できる
```

```text
他利用者
+
target_year_month一致
    ↓
取得できない
```

MonthEndAssetBalanceQueryでは、以下を確認する。

```text
操作対象利用者
+
snapshotId
+
assetAccountIds
    ↓
対象となる既存残高のみ取得
```

他利用者の以下のデータがCSV-002の判定へ混入しないことを確認する。

- 同名資産口座
- 同一対象年月snapshot
- 月末資産残高

---

#### 4.25.57 UseCaseのUnit Test

UseCaseについては、Parser、Validator、Queryの結果を組み合わせて正しいPreview DTOを生成できることを確認する。

概念的には、

```text
ParsedCsv
+
CsvRowValidationResult
+
AssetAccountQuery結果
+
MonthEndAssetSnapshotQuery結果
+
MonthEndAssetBalanceQuery結果
    ↓
PreviewMonthEndAssetBalanceCsvUseCase
    ↓
MonthEndAssetBalanceCsvPreview
```

をテストする。

特に、以下を確認する。

```text
エラーなし
    ↓
canImport = true
```

```text
CSV全体エラーあり
    ↓
canImport = false
```

```text
行エラーあり
    ↓
canImport = false
```

```text
複数行にエラーあり
    ↓
複数エラーを保持
```

また、UseCase実行によって業務データが更新されないことを確認する。

---

#### 4.25.58 CSV-003との共通検証テスト

CSV-002とCSV-003で、同一システム状態かつ同一CSVに対する登録可否判定が一致することを確認する。

概念的には、

```text
CSV-002
検証結果
    ↓
canImport = true

同一DB状態
+
同一CSV
    ↓
CSV-003
再検証
    ↓
登録可能
```

となることを確認する。

ただし、CSV-002とCSV-003の間で業務データが変更された場合は、結果が変化してよい。

例えば、

```text
CSV-002
confirmed = false
    ↓
canImport = true

その後
confirmed = true

CSV-003
    ↓
登録不可
```

となることを正常な挙動として扱う。

---

### 4.26 React・TypeScriptでの利用

CSV-002は、
利用者が選択した
月末資産残高CSVを送信し、
CSV-003で登録する前に
登録予定内容とエラー内容を確認するために使用する。

フロントエンドでは、
CSVファイルを`File`として保持し、
`FormData`へ設定して
APIへ送信する。

概念的な利用フローは、
以下とする。

```text
CSV-001
テンプレート取得
    ↓
利用者がCSV編集
    ↓
CSVファイル選択
    ↓
CSV-002
プレビュー
    ↓
canImport確認
    ├─ false
    │     ↓
    │   エラー表示
    │
    └─ true
          ↓
       登録ボタン有効化
          ↓
       CSV-003
```

---

#### 4.26.1 TypeScript型

CSVプレビュー結果は、
以下のような型として扱う。

概念例：

```ts
export type CsvPreviewError = {
  code: string;
  message: string;
};

export type MonthEndAssetBalanceCsvPreviewRow = {
  rowNumber: number;
  assetAccountName: string | null;
  balance: number | null;
  errors: CsvPreviewError[];
};

export type MonthEndAssetBalanceCsvPreview = {
  targetYearMonth: string | null;
  canImport: boolean;
  errors: CsvPreviewError[];
  rows: MonthEndAssetBalanceCsvPreviewRow[];
};

export type PreviewMonthEndAssetBalanceCsvResponse = {
  data: MonthEndAssetBalanceCsvPreview;
  requestId: string;
};
```

API共通Envelopeの
正式な型定義が存在する場合は、
共通型を利用する。

概念例：

```ts
export type PreviewMonthEndAssetBalanceCsvResponse =
  ApiResponse<MonthEndAssetBalanceCsvPreview>;
```

---

#### 4.26.2 CSVファイルの保持

CSVファイルは、
ブラウザの`File`として保持する。

概念例：

```ts
const [file, setFile] =
  useState<File | null>(
    null,
  );
```

CSV-002実行後も、
CSV-003で
同じCSVファイルを再送するため、
登録処理が完了するまで
`File`を保持する。

---

#### 4.26.3 ファイル選択

ファイル選択UIでは、
CSVファイルを
選択できるようにする。

概念例：

```tsx
<input
  type="file"
  accept=".csv,text/csv"
  onChange={(event) => {
    const selectedFile =
      event.target.files?.[0]
      ?? null;

    setFile(
      selectedFile,
    );
  }}
/>
```

`accept`属性は、
利用者のファイル選択を
補助するために使用する。

バックエンド側の
ファイル形式検証を
代替するものではない。

---

#### 4.26.4 FormData

CSVファイルは、
`FormData`へ
`file`という項目名で設定する。

概念例：

```ts
const formData =
  new FormData();

formData.append(
  'file',
  file,
);
```

以下のような
業務データは
`FormData`へ追加しない。

```text
userId
targetYearMonth
assetAccountId
balance
confirmed
```

`userId`は
`X-User-Id`で扱い、
その他の情報は
CSV内容から取得する。

---

#### 4.26.5 Content-Type

`multipart/form-data`の
`Content-Type`は、
ブラウザまたは
HTTP Clientに設定させる。

以下のように
手動設定しない。

```ts
headers: {
  'Content-Type':
    'multipart/form-data',
}
```

`FormData`を送信することで、
必要なboundaryを
HTTP Client側に生成させる。

---

#### 4.26.6 API Client

API呼び出しは、
CSVファイルを受け取る
専用関数として定義する。

概念例：

```ts
export const previewMonthEndAssetBalanceCsv =
  async (
    file: File,
  ): Promise<MonthEndAssetBalanceCsvPreview> => {
    const formData =
      new FormData();

    formData.append(
      'file',
      file,
    );

    const response =
      await apiClient.post<
        PreviewMonthEndAssetBalanceCsvResponse
      >(
        '/api/v1/month-end-asset-balances/imports/preview',
        formData,
      );

    return response.data.data;
  };
```

CSV解析や
登録可否判定を
API Client内で行わない。

---

#### 4.26.7 X-User-Id

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

CSV-002専用処理で
`userId`を
`FormData`へ追加しない。

---

#### 4.26.8 Mutationとして扱う

CSV-002は、
業務データを変更しないが、
利用者操作によって
CSVファイルを送信し、
明示的にプレビューを実行するAPIである。

TanStack Queryを使用する場合は、
Mutationとして扱ってよい。

概念例：

```ts
export const usePreviewMonthEndAssetBalanceCsv =
  () =>
    useMutation({
      mutationFn:
        previewMonthEndAssetBalanceCsv,
    });
```

画面表示時に
自動実行するQueryとしては
扱わない。

---

#### 4.26.9 プレビュー実行

利用者が
CSVファイルを選択した後、
プレビューボタンから
CSV-002を実行する。

概念例：

```ts
const previewMutation =
  usePreviewMonthEndAssetBalanceCsv();

const handlePreview =
  (): void => {
    if (file === null) {
      return;
    }

    previewMutation.mutate(
      file,
    );
  };
```

---

#### 4.26.10 プレビューボタン

CSVファイルが
選択されていない場合、
プレビューボタンを
無効化してよい。

また、
プレビュー実行中も
二重送信防止のため
無効化してよい。

概念例：

```tsx
<button
  type="button"
  disabled={
    file === null
    || previewMutation.isPending
  }
  onClick={
    handlePreview
  }
>
  {previewMutation.isPending
    ? '確認中...'
    : 'プレビュー'}
</button>
```

ただし、
`file`必須チェックは
バックエンドでも行う。

---

#### 4.26.11 プレビュー結果の保持

CSV-002成功後は、
プレビュー結果を
画面Stateへ保持してよい。

概念例：

```ts
const [preview, setPreview] =
  useState<
    MonthEndAssetBalanceCsvPreview
    | null
  >(null);
```

Mutationの
`data`をそのまま利用する場合は、
別Stateを作成しなくてもよい。

同じ情報を
複数のStateへ
不要に二重管理しない。

---

#### 4.26.12 targetYearMonth

`targetYearMonth`は、
CSV全体から特定された
対象年月を表す。

正常例：

```json
{
  "targetYearMonth": "2026-07"
}
```

対象年月が混在しているなど、
単一の対象年月として
特定できない場合は、

```json
{
  "targetYearMonth": null
}
```

となる。

そのため、
TypeScriptでは

```ts
targetYearMonth:
  string | null;
```

として扱う。

---

#### 4.26.13 canImport

`canImport`は、
プレビュー時点で
CSV全体を登録可能かを表す。

```text
true
    → プレビュー時点では登録可能

false
    → 登録不可となるエラーあり
```

登録ボタンの制御には、
この値を使用する。

フロントエンドで
`errors`配列を独自に解析して
登録可否を再計算しない。

---

#### 4.26.14 canImportは登録成功保証ではない

`canImport = true`は、
CSV-003が
必ず成功することを意味しない。

CSV-002実行後に、

- 月末資産状況が確定される
- 既存月末資産残高が登録される
- 資産口座の状態が変更される

可能性がある。

そのため、
CSV-003で
業務エラーが返却される可能性を
考慮する。

---

#### 4.26.15 CSV全体エラー

`data.errors`には、
CSV全体に関係する
エラーを表示する。

例えば、

```text
CSV_DATA_REQUIRED
MULTIPLE_TARGET_YEAR_MONTHS
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

などを対象とする。

概念例：

```tsx
{preview.errors.length > 0 && (
  <ul>
    {preview.errors.map(
      (error) => (
        <li key={error.code}>
          {error.message}
        </li>
      ),
    )}
  </ul>
)}
```

---

#### 4.26.16 rows

`rows`には、
CSV各行の
プレビュー結果が返却される。

概念例：

```tsx
<tbody>
  {preview.rows.map(
    (row) => (
      <CsvPreviewRow
        key={row.rowNumber}
        row={row}
      />
    ),
  )}
</tbody>
```

バックエンドから返却された
CSV上の並び順を維持して
表示する。

---

#### 4.26.17 rowNumber

`rowNumber`は、
CSVファイル上の
実際の行番号として扱う。

ヘッダーを1行目とするため、
最初のデータ行は

```text
rowNumber = 2
```

となる。

画面では、

```text
2行目
3行目
4行目
```

のように
利用者へ表示してよい。

---

#### 4.26.18 assetAccountName

`assetAccountName`は、
CSVへ入力された
資産口座名を表示する。

概念例：

```tsx
<td>
  {row.assetAccountName
    ?? '-'}
</td>
```

内部の

```text
asset_account_id
```

は
フロントエンドで扱わない。

---

#### 4.26.19 balance

`balance`は、
正常に整数として解析できた場合、
`number`として扱う。

```ts
balance:
  number | null;
```

表示時は、
日本円として
フォーマットしてよい。

概念例：

```tsx
<td>
  {row.balance === null
    ? '-'
    : formatYen(
        row.balance,
      )}
</td>
```

---

#### 4.26.20 0円とnull

`balance = 0`は、
正常な業務値である。

一方、

```text
balance = null
```

は、
正常な整数として
扱えなかった状態などを表す。

そのため、
以下のような
truthy / falsy判定を行わない。

```tsx
{row.balance
  ? formatYen(
      row.balance,
    )
  : '-'}
```

上記では
0円まで`-`となる。

以下のように
明示的に`null`を判定する。

```tsx
{row.balance === null
  ? '-'
  : formatYen(
      row.balance,
    )}
```

---

#### 4.26.21 行エラー

各行の`errors`には、
そのCSV行に関する
入力・業務エラーが設定される。

概念例：

```tsx
{row.errors.length > 0 && (
  <ul>
    {row.errors.map(
      (error) => (
        <li key={error.code}>
          {error.message}
        </li>
      ),
    )}
  </ul>
)}
```

1行に
複数エラーが
存在する可能性があるため、
1行1エラーを前提としない。

---

#### 4.26.22 エラー行の表示

`row.errors.length > 0`
の場合は、
該当行を
視覚的に強調してよい。

例えば、

- 背景色
- アイコン
- 枠線
- エラーメッセージ

などを使用する。

具体的な表現は、
画面設計に従う。

---

#### 4.26.23 200 OKかつcanImport=false

CSV-002では、
CSV内容に
業務エラーが存在していても、
プレビュー処理自体が
正常に完了した場合は

```text
200 OK
```

となる。

そのため、
HTTPステータスだけで
登録可能と判断しない。

必ず、

```ts
preview.canImport
```

を確認する。

---

#### 4.26.24 HTTPエラー

以下は、
プレビュー結果ではなく
HTTPエラーとして扱う。

- `X-User-Id`不正
- `file`未指定
- ファイル形式不正
- ファイルサイズ超過
- CSV解析不能
- CSVヘッダー不正
- サーバー内部エラー

これらの場合は、
`MonthEndAssetBalanceCsvPreview`として
処理しない。

---

#### 4.26.25 VALIDATION_ERROR

`VALIDATION_ERROR`の場合は、
主にアップロードファイルに関する
入力不正として扱う。

例えば、

- ファイル未指定
- 許可されないファイル形式
- ファイルサイズ超過

などを表示する。

フロントエンドで
同じ検証を完全に
再実装する必要はない。

UX向上を目的とした
事前チェックは行ってよい。

---

#### 4.26.26 INVALID_CSV_FORMAT

`INVALID_CSV_FORMAT`の場合は、
CSVをプレビュー可能な形式として
解析できなかったことを表示する。

例えば、

```text
CSVファイルの形式を確認してください。
CSVテンプレートを使用して再度作成してください。
```

のように案内する。

必要に応じて、
CSV-001の
テンプレート取得導線を表示する。

---

#### 4.26.27 利用者関連エラー

以下のエラーは、
API共通方針に従って処理する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
```

CSV-002画面だけで
独自処理を実装しない。

必要に応じて、
利用者選択画面へ戻す。

---

#### 4.26.28 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`の場合は、
共通サーバーエラーとして扱う。

CSV登録処理へは進まない。

必要に応じて、
同じCSVを
再プレビューできる導線を表示する。

---

#### 4.26.29 エラーコードと表示文言

プレビュー内の
`error.code`は、
画面処理の判別に使用できる。

ただし、
フロントエンドで
業務判定そのものを
再実装しない。

バックエンドから
`message`が返却される場合は、
基本的にその文言を表示してよい。

フロントエンド側で
文言を管理する場合は、
エラーコードとの対応を
共通定義へ集約する。

---

#### 4.26.30 ファイル変更時

利用者が
プレビュー後に
別のCSVファイルを選択した場合は、
以前のプレビュー結果を
無効化する。

概念例：

```ts
const handleFileChange =
  (
    nextFile: File | null,
  ): void => {
    setFile(
      nextFile,
    );

    setPreview(
      null,
    );
  };
```

新しいCSVファイルに対して、
以前の

```text
canImport = true
```

を流用してはならない。

---

#### 4.26.31 再プレビュー

CSV-002は、
業務データを変更しないため、
同じCSVまたは
修正したCSVを
再度プレビューできる。

概念的な操作は、
以下とする。

```text
CSV-002
    ↓
canImport = false
    ↓
利用者がCSV修正
    ↓
ファイル再選択
    ↓
CSV-002
    ↓
再プレビュー
```

---

#### 4.26.32 CSV-003登録ボタン

CSV-003の登録ボタンは、
少なくとも以下を
すべて満たす場合に
有効化する。

```text
file != null
AND
preview != null
AND
preview.canImport = true
```

概念例：

```ts
const canImport =
  file !== null
  && preview !== null
  && preview.canImport;
```

```tsx
<button
  type="button"
  disabled={!canImport}
>
  登録する
</button>
```

---

#### 4.26.33 CSV-003へ同じFileを送信する

CSV-003では、
CSV-002で使用した
同じ`File`を再送する。

```text
CSV-002
File A
    ↓
canImport = true
    ↓
CSV-003
File A
```

プレビュー結果から
CSV内容を再構築して
CSV-003へ送信しない。

---

#### 4.26.34 CSV-003成功後

CSV-003が成功した場合は、
保持していた

- CSVファイル
- プレビュー結果

をクリアしてよい。

概念例：

```ts
setFile(
  null,
);

setPreview(
  null,
);
```

また、
関連する月末資産状況や
月末資産残高のQuery Cacheを
invalidateする。

具体的なinvalidate対象は、
フロントエンド共通設計に従う。

---

#### 4.26.35 CSV-003失敗時

CSV-002で

```text
canImport = true
```

となっていても、
CSV-003で
登録不可となる可能性がある。

その場合は、
CSV-003のエラーを表示し、
必要に応じて
CSV-002を再実行する。

CSV-002の結果だけを理由に
CSV-003の業務エラーを
無視しない。

---

#### 4.26.36 CSVをReact側で正式解析しない

Phase1では、
CSV内容の正式な解析は
バックエンド側で行う。

React側で
CSV Parserライブラリを利用して、
CSV-002と同じ

- ヘッダー検証
- 対象年月検証
- 資産口座検証
- 残高検証
- 重複検証

を二重実装しない。

フロントエンドは、
CSV-002の結果を
表示・操作制御へ使用する。

---

#### 4.26.37 canImportをReact側で再計算しない

以下のような
独自判定は行わない。

```ts
const canImport =
  preview.errors.length === 0
  && preview.rows.every(
    (row) =>
      row.errors.length === 0,
  );
```

正式な登録可否は、
バックエンドが返却する

```text
canImport
```

を使用する。

これにより、
登録可否判定を
バックエンドへ集約する。

---

### 4.27 設計上の補足

#### 4.27.1 POSTを採用する理由

CSV-002は、
業務データを更新しない。

ただし、
CSVファイルを
リクエストボディとして送信し、
サーバー側で解析・検証処理を行う。

そのため、
HTTPメソッドには
`POST`を採用する。

---

#### 4.27.2 プレビュー専用APIを分ける理由

CSV-002とCSV-003を
同じAPIへ統合し、

```text
preview = true
```

などのフラグで
処理を切り替えない。

以下のように、
副作用の有無を
エンドポイントとして分離する。

```text
CSV-002
/imports/preview
    ↓
検証のみ

CSV-003
/imports
    ↓
検証
+
登録
```

これにより、
APIの責務を明確にする。

---

#### 4.27.3 canImportを返却する理由

CSVプレビューでは、
CSV全体エラーと
複数の行エラーが
存在する可能性がある。

フロントエンドが
これらを走査して
登録可否を独自判定する必要がないよう、
バックエンドから

```text
canImport
```

を返却する。

登録可否判定の
責務をバックエンドへ集約する。

---

#### 4.27.4 業務エラーを200 OKで返す理由

CSVを正常に解析でき、
プレビュー処理が
最後まで完了している場合、
API処理そのものは成功している。

例えば、

- 資産口座不存在
- 月末残高不正
- CSV内重複
- 確定済み月末資産状況

などは、
CSVプレビューによって
利用者へ発見・提示すること自体が
正常なユースケースである。

そのため、

```text
200 OK
canImport = false
```

として扱う。

---

#### 4.27.5 CSV構造不正を422とする理由

CSVヘッダー不正や
CSV解析不能の場合は、
各データ行を
正しく解釈できない。

そのため、
プレビュー結果ではなく
入力形式が成立していない状態として、

```text
422 Unprocessable Entity
```

を返却する。

---

#### 4.27.6 複数エラーを返却する理由

CSVでは、
複数行をまとめて
登録対象とする。

最初の1件だけを返却すると、

```text
プレビュー
    ↓
1件修正
    ↓
再プレビュー
    ↓
次の1件修正
```

という操作が
繰り返し必要になる。

可能な範囲で
複数エラーをまとめて返却し、
利用者が一度に
CSVを修正できるようにする。

---

#### 4.27.7 rowNumberを返却する理由

エラー内容だけでは、
利用者がCSV上の
該当行を特定しにくい。

そのため、
CSV上の実際の行番号を

```text
rowNumber
```

として返却する。

---

#### 4.27.8 内部IDを返却しない理由

CSV利用者が
確認する必要があるのは、

- CSV上の行番号
- 資産口座名
- 月末残高
- エラー内容

である。

そのため、

```text
asset_account_id
month_end_asset_snapshot_id
month_end_asset_balance_id
```

などの
内部IDは返却しない。

---

#### 4.27.9 confirmedを返却しない理由

確定済みのため
CSV登録できない場合は、
プレビューエラーとして
その状態を通知する。

React側が

```text
confirmed = true
```

を参照して
登録可否を独自判定する必要はない。

そのため、
`confirmed`自体は返却しない。

---

#### 4.27.10 プレビュー結果を保存しない理由

Phase1では、
CSV-002からCSV-003までの間に
サーバー側で
プレビュー状態を保存しない。

これにより、

- `previewId`
- preview token
- 有効期限
- 一時ファイル
- 一時データ削除

などの管理を不要とする。

CSV-003では、
CSVファイルを再送して
最新状態で再検証する。

---

#### 4.27.11 CSV-003で再検証する理由

CSV-002とCSV-003の間に、
業務データの状態が
変化する可能性がある。

例えば、

```text
CSV-002
confirmed = false
    ↓
canImport = true

別処理
confirmed = true
    ↓

CSV-003
```

となり得る。

そのため、
CSV-002の判定結果を
登録権利として扱わず、
CSV-003で必ず
最新状態を再検証する。

---

#### 4.27.12 ロックを保持しない理由

CSV-002実行後、
利用者がプレビュー画面を
長時間確認する可能性がある。

CSV-002からCSV-003まで
データベースロックを保持すると、
他の月末資産関連処理を
長時間ブロックする可能性がある。

そのため、
CSV-002ではロックを保持せず、
CSV-003実行時に
再検証する。

---

#### 4.27.13 React側で業務検証を再実装しない理由

CSV登録可否に関する
正式な業務ルールは、
バックエンドで管理する。

React側でも
同じルールを実装すると、

```text
バックエンド
    → 登録不可

React
    → 登録可能
```

などの
判定差異が発生する可能性がある。

そのため、
Reactは
CSV-002の結果を表示し、
画面操作を制御する役割に限定する。

---

#### 4.27.14 0円とnullを区別する理由

`balance = 0`は、
正常な月末残高である。

一方、

```text
balance = null
```

は、
正常な整数として
解析できなかった場合などを表す。

そのため、
TypeScriptでも

```ts
balance:
  number | null;
```

として
明確に区別する。

---

#### 4.27.15 CSV-001との仕様共通化

CSV-001で提供する
テンプレートと、
CSV-002で受け付ける
CSV形式は同一とする。

React側では、
CSVヘッダーを
独自定義しない。

正式なCSV仕様は、
バックエンドの
共通CSV Definitionへ集約する。

---

#### 4.27.16 CSV-003との検証ロジック共通化

CSV-002とCSV-003では、
CSV解析および
登録可否判定ロジックを
可能な限り共通化する。

同一システム状態かつ
同一CSVであれば、

```text
CSV-002
canImport = true

CSV-003
再検証
    ↓
登録可能
```

となることを基本とする。

ただし、
CSV-002とCSV-003の間で
業務データが変更された場合は、
結果が変わることを許容する。

---

#### 4.27.17 Idempotency-Keyを使用しない理由

CSV-002はPOST APIだが、
業務データを変更しない。

複数回実行しても
重複登録などの副作用が
発生しないため、

```text
Idempotency-Key
```

は使用しない。

---

#### 4.27.18 キャッシュしない理由

同じCSVファイルでも、

- 資産口座の状態
- 月末資産状況
- `confirmed`
- 既存月末資産残高

が変化すれば、
プレビュー結果も変化する。

そのため、
Phase1では
CSVプレビュー結果を
サーバー側でキャッシュしない。

毎回、
最新の業務データを使用して
検証する。

---

### 4.28 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)

---

## 5. CSV-003 月末資産残高CSV登録

### 5.1 概要

操作対象となる利用者について、
アップロードされた
月末資産残高CSVの内容を再検証し、
検証に成功した場合に
月末資産残高を一括登録する。

本APIでは、
CSV-002 月末資産残高CSVプレビューで
登録可能と判定されたCSVであっても、
CSV-003実行時点の最新状態をもとに
再度検証する。

主に以下を確認する。

- CSVファイル形式
- CSVヘッダー
- 対象年月
- 資産口座名
- 月末残高
- 1ファイル内の対象年月統一
- 資産口座の存在
- 資産口座の利用者境界
- 残高記録単位
- 対象年月時点での資産口座の有効性
- 月末資産状況の状態
- 既存月末資産残高との重複
- CSV内の重複

すべての検証に成功した場合のみ、
CSV内の月末資産残高を
一括登録する。

CSV内に
登録できないデータが
1件でも存在する場合は、
正常な行だけを
部分登録してはならない。

登録処理は、
トランザクション内で実行し、
途中でエラーが発生した場合は
CSV全体の登録を
ロールバックする。

概念的な処理は、
以下とする。

```text
CSVファイル受信
    ↓
ファイル検証
    ↓
CSVヘッダー検証
    ↓
CSV行解析
    ↓
入力値検証
    ↓
業務ルール再検証
    ↓
登録可否判定
    ↓
トランザクション開始
    ↓
必要に応じて
月末資産状況作成
    ↓
月末資産残高一括登録
    ↓
commit
    ↓
登録結果返却
```

本APIは、
CSV-002のプレビュー結果を
そのまま信頼して登録するAPIではない。

CSV-002とCSV-003の間で
業務データの状態が
変更される可能性があるため、
CSV-003では
必ず最新状態を再検証する。

---

### 5.2 ユースケース

利用者は、
CSV-002 月末資産残高CSVプレビューで
CSV内容を確認した後、
問題がなければ
本APIを実行して
月末資産残高を一括登録する。

基本的な利用フローは、
以下とする。

```text
CSV-001
月末資産残高CSVテンプレート取得
    ↓
利用者がCSVへデータを入力
    ↓
CSV-002
月末資産残高CSVプレビュー
    ↓
canImport = true
    ↓
利用者が登録を実行
    ↓
CSV-003
月末資産残高CSV登録
    ↓
CSV内容・業務状態を再検証
    ↓
一括登録
```

CSV-002で

```text
canImport = true
```

となっていても、
CSV-003実行時点で

- 月末資産状況が確定された
- 月末資産残高が別処理で登録された
- 資産口座の状態が変更された

などの場合は、
登録できない。

本APIでは、
CSV-002で使用した
同一のCSVファイルを
再送することを前提とする。

プレビュー結果そのものや
`previewId`などを使用して
登録する方式は採用しない。

---

### 5.3 エンドポイント

```http
POST /api/v1/month-end-asset-balances/imports
```

---

### 5.4 HTTPメソッド

```http
POST
```

本APIは、
CSVファイルを受け取り、
月末資産残高を
新規登録するAPIであるため、
`POST`を使用する。

正常に登録された場合は、
CSV内の複数行に対応する
月末資産残高が
新規作成される。

本APIは
業務データを変更するため、
CSV-002 月末資産残高CSVプレビューとは異なり、
副作用を持つ。

---

### 5.5 認証・利用者の扱い

Phase1では、
認証機能を実装しない。

操作対象となる利用者は、
`X-User-Id`
リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

利用者IDは、
以下では受け付けない。

- パスパラメータ
- クエリパラメータ
- CSV列
- `multipart/form-data`の業務項目

操作対象利用者は、
API共通方針に従い、
`X-User-Id`から特定する。

CSV内に

```text
user_id
userId
```

を持たせてはならない。

月末資産残高CSVでは、
資産口座を

```text
asset_account_name
```

によって指定する。

CSV内の
`asset_account_name`から
資産口座を特定する際は、
必ず操作対象利用者によって
絞り込む。

概念的な検索条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = CSVのasset_account_name

AND

asset_accounts.deleted_at
    IS NULL
```

他の利用者に
同名の資産口座が存在していても、
登録対象としてはならない。

例えば、

```text
User A
普通預金

User B
普通預金
```

が存在する状態で、
User Aを操作対象として
CSVに

```text
普通預金
```

が指定されている場合は、
User Aの資産口座のみを
対象とする。

User Bの資産口座へ
月末資産残高を
登録してはならない。

---

#### 5.5.1 月末資産状況の利用者境界

CSVの`target_year_month`に対応する
月末資産状況を確認する際も、
必ず操作対象利用者によって
絞り込む。

概念的な条件は、
以下とする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = CSVのtarget_year_month
```

他の利用者に
同一対象年月の
月末資産状況が存在していても、
CSV-003の登録対象として
使用してはならない。

---

#### 5.5.2 月末資産状況が存在しない場合

対象年月の
`month_end_asset_snapshots`が
存在しない場合に、
CSV-003で
月末資産状況を作成する仕様とする場合は、
必ず操作対象利用者に紐づけて作成する。

概念的には、
以下とする。

```text
user_id
    = 操作対象利用者ID

target_year_month
    = CSVのtarget_year_month

confirmed
    = false
```

他の利用者に
同じ`target_year_month`の
月末資産状況が存在することを理由に、
そのレコードを
流用してはならない。

月末資産状況の作成は、
CSV内容および
すべての業務ルール検証に
成功した後、
登録トランザクション内で行う。

---

#### 5.5.3 既存月末資産残高の利用者境界

既存の
`month_end_asset_balances`との
重複確認では、
対象となる月末資産状況と
資産口座の両方について
操作対象利用者との関連を保証する。

概念的には、

```text
month_end_asset_balances
    ↓
month_end_asset_snapshots
    ↓
user_id = 操作対象利用者ID
```

および、

```text
month_end_asset_balances
    ↓
asset_accounts
    ↓
user_id = 操作対象利用者ID
```

によって
利用者境界を保証する。

他の利用者に

- 同じ対象年月
- 同名の資産口座
- 月末資産残高

が存在していても、
CSV-003の重複判定へ
影響させてはならない。

---

#### 5.5.4 登録先資産口座の特定

Phase1では、
CSV入力項目として
`asset_account_id`を使用しない。

そのため、
登録先資産口座は

```text
操作対象利用者
+
asset_account_name
```

によって特定する。

同一利用者内では、
資産口座名が
一意であることを前提とする。

CSV-003では、
CSVから直接
データベースIDを受け取らない。

CSV解析後に
サーバー側で取得した
`asset_accounts.id`を使用して、
`month_end_asset_balances.asset_account_id`
を設定する。

---

#### 5.5.5 登録する月末資産残高の利用者境界

`month_end_asset_balances`自体に
`user_id`を保持しない場合でも、
登録する

```text
month_end_asset_snapshot_id
```

および

```text
asset_account_id
```

の両方が
操作対象利用者に属することを
登録前に保証する。

概念的には、

```text
操作対象利用者
    ↓
month_end_asset_snapshot
        +
asset_account
    ↓
month_end_asset_balance
```

という関連とする。

以下のような
利用者をまたぐ組み合わせを
登録してはならない。

```text
User Aのsnapshot
+
User Bのasset_account
    ↓
month_end_asset_balance
```

---

#### 5.5.6 他利用者データを登録可否判定へ利用しない

他の利用者に属する

- 資産口座
- 月末資産状況
- 月末資産残高

の存在を、
CSV-003の登録可否判定へ
影響させてはならない。

例えば、
操作対象利用者には
`普通預金`が存在せず、
他の利用者にのみ
`普通預金`が存在する場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

相当の
登録エラーとして扱う。

他利用者の資産口座を利用して
登録成功としてはならない。

---

#### 5.5.7 X-User-Idのエラー

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

として扱う。

形式が不正な場合は、

```text
INVALID_USER_ID
```

として扱う。

指定された利用者が
存在しない場合、
または論理削除されている場合は、

```text
USER_NOT_FOUND
```

として扱う。

これらのエラーが発生した場合は、
CSVファイルの解析、
月末資産状況の作成、
月末資産残高の登録へ
進まない。

---

#### 5.5.8 CSV-002の利用者コンテキストを引き継がない

CSV-003では、
CSV-002実行時の
利用者コンテキストを
サーバー側で保持・引き継がない。

CSV-003のリクエストでも、
改めて

```http
X-User-Id
```

を指定する。

CSV-002で
User AとしてプレビューしたCSVを、
CSV-003でUser Bとして送信した場合は、
CSV-003では
User Bの業務データを基準として
再検証する。

CSV-002の結果を
利用者境界の保証として
使用しない。

---

#### 5.5.9 利用者境界確認後に登録する

月末資産残高を
登録する前に、
少なくとも以下について
操作対象利用者との関連を確認する。

```text
asset_account
    → 操作対象利用者に属する

month_end_asset_snapshot
    → 操作対象利用者に属する

existing month_end_asset_balance
    → 操作対象利用者のデータだけを確認
```

これらの検証に
失敗した場合は、
CSV全体を登録しない。

一部の行だけを
登録してはならない。

---

#### 5.5.10 登録処理の利用者単位

1回のCSV-003リクエストでは、
`X-User-Id`で指定された
1利用者のデータだけを扱う。

1つのCSVファイルから
複数利用者の
月末資産残高を
登録することはできない。

CSVには
利用者を切り替えるための
列を持たせない。

これにより、
CSV登録処理の
利用者境界を明確にする。

---

### 5.6 パスパラメータ

本APIでは、
パスパラメータを使用しない。

エンドポイントは、
以下とする。

```http
POST /api/v1/month-end-asset-balances/imports
```

対象年月、
資産口座、
月末残高などは、
アップロードされたCSVファイルから取得する。

そのため、
以下の情報を
パスパラメータとして受け付けない。

- `targetYearMonth`
- `assetAccountId`
- `snapshotId`
- `userId`

操作対象利用者は、
`X-User-Id`
リクエストヘッダーから特定する。

---

### 5.7 クエリパラメータ

本APIでは、
クエリパラメータを使用しない。

CSV登録対象となる情報は、
アップロードされたCSVファイルから取得する。

そのため、
以下のような
クエリパラメータは受け付けない。

- `targetYearMonth`
- `assetAccountId`
- `confirmed`
- `overwrite`
- `force`
- `preview`

本APIは、
新規登録を行うCSV登録APIであり、
上書き可否などを
クエリパラメータによって
切り替えない。

---

### 5.8 リクエストヘッダー

本APIでは、
以下のリクエストヘッダーを使用する。

| ヘッダー | 必須 | 内容 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
POST /api/v1/month-end-asset-balances/imports
Accept: application/json
X-User-Id: 1
```

CSVファイルは、
`multipart/form-data`で送信する。

`Content-Type`は、
HTTPクライアントによって
boundary付きで設定する。

概念例：

```http
Content-Type: multipart/form-data; boundary=...
```

クライアント側で
boundaryを固定値として
手動設定しない。

---

### 5.9 リクエストボディ

本APIでは、
`multipart/form-data`形式で
CSVファイルを受け取る。

リクエスト項目は、
以下とする。

| 項目 | 型 | 必須 | 内容 |
|---|---|:---:|---|
| `file` | file | ○ | 登録対象となる月末資産残高CSV |

概念例：

```text
multipart/form-data

file:
  month-end-asset-balances.csv
```

本APIでは、
CSVファイル以外の
業務項目を
リクエストボディで受け付けない。

以下のような項目は、
`multipart/form-data`の
追加フィールドとして
指定しない。

- `userId`
- `targetYearMonth`
- `assetAccountId`
- `balance`
- `confirmed`
- `overwrite`
- `previewId`

`userId`は、
`X-User-Id`から取得する。

対象年月、
資産口座名、
月末残高は、
CSV内容から取得する。

---

### 5.10 CSVファイル仕様

月末資産残高CSVでは、
以下のヘッダーを使用する。

```csv
target_year_month,asset_account_name,balance
```

ヘッダー順序も
CSV仕様の一部として扱う。

CSV-001
月末資産残高CSVテンプレート取得で
提供する形式と同一とする。

CSV入力例：

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,生活用口座,800000
```

1つのCSVファイルでは、
1つの`target_year_month`のみを扱う。

---

### 5.11 バリデーション

CSV-003では、
CSV-002と同等の
CSV解析・登録可否検証を
再度実行する。

検証は、
概念的に以下の段階で行う。

```text
リクエスト検証
    ↓
CSVファイル・構造検証
    ↓
CSV行入力値検証
    ↓
業務ルール検証
    ↓
登録可否確定
    ↓
登録処理
```

CSV-002で
`canImport = true`
となっていることを、
CSV-003の
入力条件とはしない。

CSV-003単体でも、
安全に登録可否を
判定できる必要がある。

---

#### 5.11.1 file 必須

`file`は、
必須とする。

CSVファイルが
指定されていない場合は、
バリデーションエラーとする。

不正例：

```text
file未指定
```

この場合、
CSV解析処理や
データベース登録へ進まない。

---

#### 5.11.2 アップロードファイルであること

`file`は、
HTTPアップロードファイルとして
正常に受信できていることを確認する。

通常の文字列や
フォーム値を
CSVファイルとして扱わない。

---

#### 5.11.3 ファイル拡張子

アップロードファイルは、
CSVファイルであることを確認する。

Phase1では、
拡張子として

```text
.csv
```

を受け付ける。

例えば、
以下は不正とする。

```text
.xlsx
.xls
.txt
.pdf
```

ただし、
拡張子だけを
唯一の判定根拠とはせず、
実際にCSVとして
正常に読み取れることも確認する。

---

#### 5.11.4 ファイルサイズ

CSVファイルサイズには、
CSV共通仕様で定める
上限を適用する。

上限を超えるファイルは、
CSV解析前に
バリデーションエラーとする。

具体的な上限値は、
CSV共通仕様の
最新定義に従う。

---

#### 5.11.5 空ファイル

CSVファイルが
0バイトの場合、
またはCSVとして
有効なヘッダー行を
取得できない場合は、
不正なCSVとして扱う。

この場合、
登録処理へ進まない。

---

#### 5.11.6 文字コード

CSVファイルの文字コードは、
CSV共通仕様に従う。

Phase1では、
UTF-8を基本とする。

CSV-001で提供する
テンプレートと
同じ文字コードを
使用することを前提とする。

UTF-8 BOMを許可する場合は、
先頭ヘッダーへ
BOMが混入しないよう
解析時に適切に除去する。

---

#### 5.11.7 CSVとして読み取り可能であること

アップロードされたファイルが、
CSVとして正常に
読み取れることを確認する。

例えば、
以下のような状態は
不正とする。

- ファイル破損
- CSVとして解析不能
- 不正な引用符
- 想定外の列構造

CSV解析に失敗した場合は、
データベース登録へ進まない。

---

#### 5.11.8 ヘッダー必須

CSVの1行目には、
ヘッダー行が
存在することを必須とする。

期待するヘッダーは、
以下とする。

```text
target_year_month
asset_account_name
balance
```

---

#### 5.11.9 ヘッダー名

ヘッダーは、
以下と完全一致することを
基本とする。

```csv
target_year_month,asset_account_name,balance
```

以下のような
別名は受け付けない。

```text
targetYearMonth
assetAccountName
amount
```

CSV-001のテンプレートを
正式な入力形式とする。

---

#### 5.11.10 ヘッダー順序

ヘッダーは、
以下の順序とする。

```text
1. target_year_month
2. asset_account_name
3. balance
```

Phase1では、
ヘッダー名が同じでも
順序が異なるCSVは
不正とする。

CSV-001、
CSV-002、
CSV-003で
同一仕様を使用する。

---

#### 5.11.11 余分なヘッダー

定義されていない
余分なCSV列は
受け付けない。

例えば、

```csv
target_year_month,asset_account_name,balance,memo
```

は不正とする。

以下のような
内部管理項目も
CSV列として受け付けない。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- `month_end_asset_balance_id`
- `confirmed`

---

#### 5.11.12 データ行の存在

ヘッダー行だけで、
データ行が1件も存在しない場合は、
登録対象データが存在しないため、
登録不可とする。

例えば、

```csv
target_year_month,asset_account_name,balance
```

のみのCSVは
登録しない。

CSV-003では、
データ行0件の状態で
成功レスポンスを返して
何も登録しない方式は採用しない。

具体的なエラーコードは、
「エラーレスポンス」で定義する。

---

#### 5.11.13 空行

CSV末尾などの
完全な空行については、
CSV共通仕様に従って
無視してよい。

ただし、

```csv
,,
```

のように
列自体が存在する行は、
データ行として扱い、
各項目の必須チェックを行う。

---

#### 5.11.14 target_year_month 必須

各データ行の
`target_year_month`は
必須とする。

空文字の場合は、
登録不可とする。

不正例：

```csv
target_year_month,asset_account_name,balance
,普通預金,1500000
```

---

#### 5.11.15 target_year_month 形式

`target_year_month`は、
以下の形式とする。

```text
YYYY-MM
```

正常例：

```text
2026-01
2026-07
2026-12
```

不正例：

```text
2026-1
26-07
2026/07
202607
2026-00
2026-13
abc
```

形式だけでなく、
年月として有効な値であることも確認する。

---

#### 5.11.16 1ファイル1対象年月

1つのCSVファイルでは、
すべてのデータ行の
`target_year_month`が
同一であることを必須とする。

正常例：

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,生活用口座,800000
```

不正例：

```csv
target_year_month,asset_account_name,balance
2026-06,普通預金,1500000
2026-07,生活用口座,800000
```

複数の対象年月が
混在している場合は、
CSV全体を登録不可とする。

---

#### 5.11.17 asset_account_name 必須

各データ行の
`asset_account_name`は
必須とする。

空文字の場合は、
登録不可とする。

不正例：

```csv
target_year_month,asset_account_name,balance
2026-07,,1500000
```

---

#### 5.11.18 asset_account_nameの扱い

`asset_account_name`は、
資産口座を特定するための
文字列として扱う。

前後空白の扱いは、
CSV共通仕様または
資産口座名の業務ルールに従う。

入力値を
暗黙的に別名へ変換して
資産口座を特定しない。

例えば、

```text
普通預金
```

と

```text
普通預金口座
```

を
自動的に同一資産口座として
扱わない。

---

#### 5.11.19 資産口座の存在確認

CSVの
`asset_account_name`に対応する
資産口座が、
操作対象利用者に
存在することを確認する。

概念的には、
以下の条件で検索する。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = asset_account_name

AND

asset_accounts.deleted_at
    IS NULL
```

該当する資産口座が
存在しない場合は、
CSV全体を登録不可とする。

---

#### 5.11.20 他利用者の同名資産口座

操作対象利用者には
該当資産口座が存在せず、
他の利用者にのみ
同名資産口座が存在する場合でも、
正常とは判定しない。

例えば、

```text
User A
普通預金なし

User B
普通預金あり
```

で、
User Aとして

```text
asset_account_name = 普通預金
```

を指定した場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

相当の
登録エラーとする。

---

#### 5.11.21 残高記録単位

CSV-003は、
月末資産残高CSVの
登録APIである。

そのため、
対象資産口座の
`balance_recording_unit`が
口座単位であることを確認する。

概念的には、

```text
balance_recording_unit
    = 口座単位
```

を必須とする。

商品単位で
評価額を管理する資産口座は、
本APIの登録対象としない。

商品単位の資産口座については、
商品別月末評価額CSV登録APIを使用する。

---

#### 5.11.22 対象年月時点での資産口座の有効性

対象資産口座が、
CSVの`target_year_month`時点で
月末資産残高の
記録対象として有効であることを確認する。

現在時点の状態だけで
判定してはならない。

必ず、

```text
CSVのtarget_year_month
```

を基準とする。

対象年月時点で
利用できない資産口座は、
登録不可とする。

---

#### 5.11.23 balance 必須

各データ行の
`balance`は
必須とする。

空文字の場合は、
登録不可とする。

不正例：

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,
```

---

#### 5.11.24 balance 数値形式

`balance`は、
日本円の整数として
扱える値であることを確認する。

正常例：

```text
0
1000
1500000
```

不正例：

```text
1,500,000
¥1500000
1500.5
abc
```

CSV上では、
桁区切りや
通貨記号を使用しない。

---

#### 5.11.25 balance 整数

`balance`は、
整数であることを必須とする。

以下のような
小数値は受け付けない。

```text
1000.5
```

Phase1では、
日本円の整数値として管理する。

---

#### 5.11.26 balance 0円以上

`balance`は、
0以上とする。

正常例：

```text
0
1
1500000
```

不正例：

```text
-1
-100000
```

0円は
有効な月末残高として扱う。

```text
0円
≠
未入力
```

とする。

---

#### 5.11.27 月末資産状況の確認

CSVの`target_year_month`について、
操作対象利用者に属する
`month_end_asset_snapshots`を確認する。

概念的な検索条件は、
以下とする。

```text
user_id = 操作対象利用者ID
AND
target_year_month = target_year_month
```

他の利用者の
同一対象年月の
月末資産状況を
使用してはならない。

---

#### 5.11.28 月末資産状況が存在しない場合

対象年月の
`month_end_asset_snapshots`が
存在しない場合は、
CSV-003で
新しい月末資産状況を
作成する。

ただし、
CSV解析・入力値検証・
業務ルール検証が
すべて成功した後に行う。

概念的には、
以下を登録する。

```text
user_id
    = 操作対象利用者ID

target_year_month
    = CSVのtarget_year_month

confirmed
    = false
```

月末資産状況の作成は、
月末資産残高の登録と
同一トランザクション内で行う。

---

#### 5.11.29 確定済み月末資産状況

対象年月の
月末資産状況が存在し、

```text
confirmed = true
```

の場合は、
月末資産残高を
登録できない。

この場合、
CSV全体を登録不可とする。

CSV-002で
`canImport = true`
となっていたとしても、
CSV-003実行時点で
確定済みとなっていれば
登録してはならない。

---

#### 5.11.30 既存月末資産残高との重複

対象年月、
対象資産口座について、
既に
`month_end_asset_balances`が
存在するか確認する。

概念的には、

```text
month_end_asset_snapshot_id
+
asset_account_id
```

の組み合わせで
既存データを確認する。

Phase1では、
CSV-003を
新規一括登録APIとして扱う。

そのため、
既存データが存在する場合は
重複エラーとする。

既存の`balance`を
CSV値で上書きしない。

---

#### 5.11.31 overwriteを許可しない

CSV-003では、
既存の月末資産残高を
上書きするための

```text
overwrite = true
```

などのオプションを
受け付けない。

既存データを変更する場合は、
月末資産残高更新APIを使用する。

CSV登録APIへ
更新責務を持たせない。

---

#### 5.11.32 CSV内重複

同一CSV内で、
同じ対象年月かつ
同じ資産口座が
複数行存在しないことを確認する。

例えば、

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,普通預金,1600000
```

は、
登録不可とする。

先勝ち、
後勝ち、
残高合算などの
暗黙的なルールは採用しない。

---

#### 5.11.33 複数エラー

CSV内に
複数の入力・業務エラーが
存在する場合は、
CSV-002と同様に
可能な範囲で
複数エラーを検出してよい。

ただし、
CSV-003では
エラーが1件でも存在する場合、
登録処理へ進まない。

概念的には、

```text
CSV検証
    ↓
エラー収集
    ↓
1件以上あり
    ↓
登録処理中止
```

とする。

---

#### 5.11.34 一部登録を行わない

CSV内に
1件でも登録不可データが
存在する場合は、
CSV全体を登録しない。

例えば、

```text
2行目 正常
3行目 正常
4行目 エラー
```

の場合でも、

```text
2行目 登録
3行目 登録
4行目 未登録
```

とはしない。

CSV全体を
登録失敗として扱う。

---

#### 5.11.35 登録前に全件検証する

CSV行を
読み込みながら
逐次データベースへ
INSERTする方式は採用しない。

以下のような処理は
行わない。

```text
2行目
検証
    ↓
INSERT

3行目
検証
    ↓
INSERT

4行目
エラー
```

原則として、

```text
CSV全体解析
    ↓
全件検証
    ↓
登録可能確認
    ↓
トランザクション開始
    ↓
一括登録
```

とする。

---

#### 5.11.36 CSV-002と同じ検証ロジックを使用する

CSV-003では、
CSV-002と
可能な限り同一の

- CSV Definition
- CSV Parser
- CSV入力値Validator
- 業務ルールValidator

を使用する。

同一のシステム状態、
同一のCSVであれば、

```text
CSV-002
canImport = true
```

の場合に、

```text
CSV-003
登録可能
```

となることを基本とする。

CSV-002とCSV-003で
個別に検証ルールを
再実装しない。

---

#### 5.11.37 CSV-002の結果は信頼しない

CSV-003では、
CSV-002で返却された

```text
canImport
errors
rows
```

などを
リクエストとして受け取らない。

また、
クライアント側で保持している
プレビュー結果を
登録可否の根拠として使用しない。

CSV-003は、
受信したCSVファイルと
実行時点のデータベース状態だけを
基準として登録可否を判定する。

---

#### 5.11.38 登録直前の再確認

CSV全体の事前検証に
成功した場合でも、
登録トランザクション内で
競合し得る状態については
必要に応じて再確認する。

特に、

- 月末資産状況の`confirmed`
- 既存月末資産残高

については、
CSV-002実行時点ではなく
CSV-003の登録時点で
正しい状態を保証する必要がある。

具体的なロック・
再確認方法は、
「Laravel実装方針」で定義する。

---

#### 5.11.39 データベース制約も最終防衛線とする

アプリケーション側で
既存月末資産残高の
重複を事前検証する。

加えて、
テーブル定義で

```text
month_end_asset_snapshot_id
+
asset_account_id
```

に一意性を保証している場合は、
データベース制約も
最終的な重複防止として利用する。

ただし、
DB制約違反を
通常の業務フローとして
発生させることを前提にはしない。

可能な限り、
登録前に業務ルールとして
検出する。

---

#### 5.11.40 バリデーション失敗時はDB更新しない

以下のいずれかの
検証に失敗した場合は、
業務データを更新しない。

- ファイル検証
- CSV構造検証
- CSV入力値検証
- 資産口座検証
- 残高記録単位検証
- 資産口座有効性検証
- 確定状態検証
- 既存月末資産残高検証
- CSV内重複検証

以下を行ってはならない。

```text
month_end_asset_snapshots INSERT
month_end_asset_balances INSERT
```

すべての検証に成功した場合のみ、
登録処理へ進む。

---

### 5.12 業務ルール

#### 5.12.1 CSV登録の目的

本APIは、
月末資産残高CSVの内容を検証し、
登録可能な場合に
口座単位の月末資産残高を
一括登録する。

CSV-003は、
CSV-002 月末資産残高CSVプレビューで
内容を確認した後に
実行することを想定する。

基本的な利用フローは、
以下とする。

```text
CSV-001
テンプレート取得
    ↓
CSV入力
    ↓
CSV-002
プレビュー
    ↓
エラーなし
    ↓
CSV-003
登録
```

CSV-003では、
CSV-002のプレビュー結果を
サーバー側に保存・参照しない。

そのため、
CSV-003実行時に
CSV内容および
業務データの状態を
再検証する。

---

#### 5.12.2 CSV-002と同じCSV仕様を使用する

CSV-003で受け付ける
CSV形式は、
CSV-001およびCSV-002と
同一とする。

ヘッダーは、
以下とする。

```csv
target_year_month,asset_account_name,balance
```

以下についても、
CSV-002と
同一ルールを使用する。

- 文字コード
- BOM
- 改行コード
- ヘッダー名
- ヘッダー順序
- 必須項目
- 対象年月形式
- 月末残高形式
- CSV内重複判定

CSV-002では正常、
CSV-003ではCSV仕様不正となるような
実装差異を発生させない。

---

#### 5.12.3 1ファイル1対象年月

1つのCSVファイルでは、
1つの対象年月のみを扱う。

すべてのデータ行について、
`target_year_month`が
同一であることを必須とする。

正常例：

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,生活用口座,800000
```

不正例：

```csv
target_year_month,asset_account_name,balance
2026-06,普通預金,1500000
2026-07,生活用口座,800000
```

複数の対象年月が
混在している場合は、
CSV全体を登録しない。

---

#### 5.12.4 資産口座の特定

CSV上では、
内部IDを指定しない。

資産口座は、

```text
操作対象利用者
+
asset_account_name
```

によって特定する。

概念的な条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = CSVのasset_account_name

AND

asset_accounts.deleted_at
    IS NULL
```

CSVに

```text
asset_account_id
```

を持たせない。

---

#### 5.12.5 利用者境界

資産口座、
月末資産状況、
既存月末資産残高の確認では、
必ず操作対象利用者との
関連を保証する。

他の利用者に属する

- 同名資産口座
- 同一対象年月の月末資産状況
- 月末資産残高

を登録処理へ
使用してはならない。

例えば、

```text
User A
普通預金なし

User B
普通預金あり
```

の場合、
User Aとして
`普通預金`を指定したCSVを
登録しようとした場合は、
資産口座不存在として扱う。

---

#### 5.12.6 残高記録単位

CSV-003は、
口座単位の
月末資産残高を登録するAPIである。

そのため、
対象資産口座の

```text
balance_recording_unit
```

が
口座単位であることを必須とする。

商品単位の資産口座は、
本APIでは登録しない。

商品単位の資産口座については、
商品別月末評価額CSV登録APIを使用する。

---

#### 5.12.7 対象年月時点の資産口座

資産口座が、
CSVの`target_year_month`時点で
月末資産管理対象として
有効であることを確認する。

現在時点の
資産口座の状態だけで
判定してはならない。

```text
CSVのtarget_year_month
    ↓
対象年月時点の資産口座状態
    ↓
登録可否判定
```

とする。

対象年月時点で
有効でない資産口座については、
CSV全体を登録不可とする。

---

#### 5.12.8 月末残高

`balance`は、
対象年月末時点の
資産口座残高として扱う。

Phase1では、
日本円の整数とする。

以下を満たす必要がある。

```text
整数
AND
0以上
```

0円は、
正常な月末残高として扱う。

```text
balance = 0
```

を未入力扱いしない。

---

#### 5.12.9 月末資産状況の取得

CSVの対象年月について、
操作対象利用者に属する
月末資産状況を取得する。

概念的な条件は、
以下とする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = CSVのtarget_year_month
```

他の利用者の
同一対象年月の
月末資産状況を
使用しない。

---

#### 5.12.10 月末資産状況が存在しない場合

対象年月の
月末資産状況が
存在しない場合は、
CSV登録処理の中で
新規作成する。

作成する月末資産状況は、
概念的に以下とする。

```text
user_id
    = 操作対象利用者ID

target_year_month
    = CSVのtarget_year_month

confirmed
    = false
```

月末資産状況の作成は、
CSV全体の検証が
正常に完了した後に行う。

CSV検証途中で
先に月末資産状況を
作成してはならない。

---

#### 5.12.11 既存の未確定月末資産状況

対象年月について
未確定の月末資産状況が
既に存在する場合は、
新しい月末資産状況を
作成しない。

既存の月末資産状況を
登録先として使用する。

```text
snapshotあり
+
confirmed = false
    ↓
既存snapshotを使用
```

同一利用者、
同一対象年月について
複数の月末資産状況を
作成してはならない。

---

#### 5.12.12 確定済み月末資産状況

対象年月の
月末資産状況が

```text
confirmed = true
```

の場合は、
CSV登録を許可しない。

この場合、
CSV全体を登録しない。

CSV-002実行時点では
未確定であっても、
CSV-003実行時点で
確定済みとなっている場合は、
最新状態を優先する。

```text
CSV-002
confirmed = false
    ↓
プレビュー成功

別処理
confirmed = true
    ↓

CSV-003
    ↓
登録不可
```

---

#### 5.12.13 既存月末資産残高

対象年月、
対象資産口座について、
既に月末資産残高が
登録されていないことを確認する。

概念的には、

```text
month_end_asset_snapshot_id
+
asset_account_id
```

の組み合わせで
重複を確認する。

既存データが存在する場合は、
CSV登録を許可しない。

---

#### 5.12.14 既存データを上書きしない

CSV-003では、
既存の月末資産残高を
CSV値によって
上書きしない。

以下のような処理は
行わない。

```text
既存balanceあり
    ↓
CSVのbalanceでUPDATE
```

既存の月末資産残高を
修正する場合は、
BAL-003 月末資産残高更新APIを使用する。

CSV-003は、
新規一括登録に
責務を限定する。

---

#### 5.12.15 CSV内重複

同一CSV内で、
同じ資産口座を
複数行登録してはならない。

1ファイル1対象年月であるため、
実質的には
`asset_account_name`が
CSV内で一意であることを
必須とする。

不正例：

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,普通預金,1600000
```

以下のような
自動解決は行わない。

- 先勝ち
- 後勝ち
- 残高合算
- 平均値算出

---

#### 5.12.16 全件成功または全件失敗

CSV登録は、
CSVファイル単位で
原子的に処理する。

CSV内に
1件でも登録不可となるデータが
存在する場合は、
正常な行も含めて
すべて登録しない。

```text
2行目
正常

3行目
正常

4行目
エラー
    ↓
全件登録しない
```

部分成功は
許可しない。

---

#### 5.12.17 全件検証後に登録する

CSV行を
解析しながら
逐次INSERTしてはならない。

以下のような処理は
採用しない。

```text
2行目
検証成功
    ↓
INSERT

3行目
検証成功
    ↓
INSERT

4行目
検証失敗
```

基本的には、
以下の順序とする。

```text
CSV全体解析
    ↓
全行入力値検証
    ↓
全行业務ルール検証
    ↓
登録可能確認
    ↓
トランザクション開始
    ↓
一括登録
```

---

#### 5.12.18 登録トランザクション

月末資産状況の作成と
月末資産残高の登録は、
同一トランザクション内で行う。

概念的には、

```text
BEGIN
    ↓
月末資産状況取得・必要なら作成
    ↓
登録直前状態確認
    ↓
月末資産残高一括登録
    ↓
COMMIT
```

とする。

途中で
1件でも登録に失敗した場合は、

```text
ROLLBACK
```

し、
月末資産状況だけが残る、
一部の月末資産残高だけが残る、
といった状態を発生させない。

---

#### 5.12.19 月末資産状況だけを残さない

対象年月の
月末資産状況が存在しなかったため
CSV-003で新規作成した後に、
月末資産残高登録で
エラーが発生した場合は、
新規作成した月末資産状況も
ロールバックする。

```text
snapshotなし
    ↓
snapshot作成
    ↓
balance登録中にエラー
    ↓
ROLLBACK
    ↓
snapshotも残さない
```

---

#### 5.12.20 一括登録

すべてのCSV行が
登録可能であることを確認した後、
`month_end_asset_balances`へ
登録する。

各行について、
概念的に以下を設定する。

```text
month_end_asset_snapshot_id
    = 対象年月のsnapshot.id

asset_account_id
    = CSVのasset_account_nameから特定したasset_accounts.id

balance
    = CSVのbalance
```

CSV入力値の
`asset_account_name`自体を
`month_end_asset_balances`へ
保存しない。

---

#### 5.12.21 CSV行番号は保存しない

CSV解析時に使用した

```text
rowNumber
```

は、
CSVファイル上で
エラー箇所を特定するための
一時的な情報である。

`month_end_asset_balances`へ
CSV行番号を保存しない。

---

#### 5.12.22 CSVファイルを保存しない

Phase1では、
アップロードされたCSVファイル自体を
サーバーへ永続保存しない。

以下のような
CSVインポート履歴管理は
行わない。

```text
csv_import_files
csv_import_histories
csv_import_sessions
```

CSVの登録結果は、
通常の

```text
month_end_asset_snapshots
month_end_asset_balances
```

として保存する。

---

#### 5.12.23 プレビュー結果を保存しない

CSV-002で生成した
プレビュー結果は、
CSV-003で参照しない。

以下のような
情報を使用しない。

```text
previewId
previewToken
previewResult
```

CSV-003では、
受信したCSVを
再解析・再検証する。

---

#### 5.12.24 CSV-002での正常判定を登録権利としない

CSV-002の

```text
canImport = true
```

は、
その時点における
登録可能性を示すだけである。

CSV-003では、
以下を再確認する。

- CSV内容
- 資産口座
- 残高記録単位
- 対象年月時点の有効性
- 月末資産状況
- 確定状態
- 既存月末資産残高
- CSV内重複

CSV-002の結果を
登録予約として扱わない。

---

#### 5.12.25 同時実行

同一利用者、
同一対象年月について
複数のCSV-003が
同時実行される可能性を考慮する。

例えば、

```text
Request A
重複なし確認

Request B
重複なし確認

Request A
INSERT

Request B
INSERT
```

のような競合によって
重複登録が発生しないよう、
アプリケーション側の確認に加えて
データベースのUNIQUE制約を
最終防衛線として使用する。

必要な排他制御の詳細は、
Laravel実装方針で定義する。

---

#### 5.12.26 データベース制約

`month_end_asset_balances`では、
同一月末資産状況、
同一資産口座について
複数の残高を登録できないよう、
テーブル定義上の
一意性を保証する。

概念的には、

```text
UNIQUE (
    month_end_asset_snapshot_id,
    asset_account_id
)
```

とする。

CSV-003では、
この制約を
アプリケーション側の
重複チェックの代替にはしない。

---

#### 5.12.27 登録後の確定状態

CSV-003によって
月末資産残高を登録しても、
月末資産状況を
自動的に確定しない。

```text
CSV-003
月末資産残高登録
    ↓
confirmed = false
```

のままとする。

月末資産状況の確定は、
SNP-004 月末資産状況確定APIの
責務とする。

---

#### 5.12.28 未登録資産の存在

CSV-003は、
CSVに含まれる
月末資産残高を登録するAPIである。

対象年月時点で有効な
すべての資産口座が
CSVに含まれていることを、
CSV-003の登録条件とはしない。

CSV登録後に
未登録の資産口座が残っていても、
月末資産状況を
未確定のまま保持できる。

すべての必要データが
揃っているかどうかは、
月末資産状況確定時に判定する。

---

#### 5.12.29 商品単位データを登録しない

CSV-003では、
`month_end_holding_values`を
登録しない。

商品単位で管理する
資産口座および保有商品の
月末評価額登録は、
商品別月末評価額CSV登録APIの
責務とする。

CSV-003は、

```text
month_end_asset_balances
```

の登録に責務を限定する。

---

#### 5.12.30 登録件数

登録件数は、
実際に新規登録した
`month_end_asset_balances`の
件数とする。

例えば、
CSVに3件の正常なデータがあり、
3件すべてを登録した場合は、

```text
importedCount = 3
```

とする。

ヘッダー行や
完全な空行は、
登録件数へ含めない。

---

### 5.13 レスポンス

CSV登録に成功した場合は、
登録結果を
JSON形式で返却する。

正常時は、
API共通の
成功レスポンスEnvelopeを使用する。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "importedCount": 2
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

CSV内の
各月末資産残高を
レスポンスへ全件返却するのではなく、
登録結果の確認に必要な
最小限の情報を返却する。

---

#### 5.13.1 正常レスポンス

CSV内の
すべての月末資産残高を
正常に登録できた場合は、

```http
201 Created
```

を返却する。

本APIは、
複数の
`month_end_asset_balances`を
新規作成するため、
登録成功を
`201 Created`として扱う。

---

#### 5.13.2 新しい月末資産状況を作成した場合

対象年月の
月末資産状況が存在せず、
CSV-003によって
月末資産状況を新規作成した場合も、
正常レスポンス形式は
変更しない。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "importedCount": 2
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

月末資産状況を
本API内で新規作成したかどうかを
クライアントが判断する必要はないため、

```text
snapshotCreated
```

のようなフラグは
返却しない。

---

#### 5.13.3 既存の未確定月末資産状況を使用した場合

対象年月の
未確定月末資産状況が
既に存在する場合は、
その月末資産状況へ
月末資産残高を登録する。

この場合も、
レスポンス形式は同一とする。

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "importedCount": 2
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 5.13.4 登録した残高一覧を返却しない

CSV-003では、
登録した月末資産残高を
全件レスポンスへ含めない。

以下のような
レスポンスにはしない。

```json
{
  "data": {
    "balances": [
      {
        "assetAccountName": "普通預金",
        "balance": 1500000
      },
      {
        "assetAccountName": "生活用口座",
        "balance": 800000
      }
    ]
  }
}
```

登録内容は、
CSV-002のプレビューで
事前確認済みである。

登録後に
月末資産残高一覧が必要な場合は、
BAL-001 月末資産残高一覧取得APIを使用する。

---

#### 5.13.5 エラー時

CSV内容または
業務ルールの検証に失敗した場合は、
月末資産残高を登録しない。

エラー時は、
API共通方針で定める
JSON形式の
エラーレスポンスを返却する。

CSV-002とは異なり、
CSV-003では
登録できないCSVを

```text
200 OK
canImport = false
```

として返却しない。

CSV-003は登録APIであるため、
登録条件を満たさない場合は
適切なHTTPエラーとして扱う。

具体的なHTTPステータスと
独自エラーコードは、
「エラーレスポンス」で定義する。

---

#### 5.13.6 登録途中のエラー

トランザクション開始後に
登録処理でエラーが発生した場合は、
すべてロールバックする。

その場合も、
一部登録成功を示すような
レスポンスを返却しない。

以下のような
レスポンスは返却しない。

```json
{
  "data": {
    "importedCount": 2,
    "failedCount": 1
  }
}
```

本APIは、
全件成功または
全件失敗とする。

---

### 5.14 レスポンス項目

正常時の
`data`配下には、
以下の項目を返却する。

| 項目 | 型 | NULL | 内容 |
|---|---|:---:|---|
| `targetYearMonth` | string | × | 登録対象となった対象年月 |
| `importedCount` | integer | × | 新規登録した月末資産残高件数 |

---

#### 5.14.1 targetYearMonth

登録対象となった
CSVの対象年月を返却する。

形式は、

```text
YYYY-MM
```

とする。

例：

```json
{
  "targetYearMonth": "2026-07"
}
```

CSVは
1ファイル1対象年月であるため、
正常登録時には
必ず1つの対象年月を返却できる。

---

#### 5.14.2 importedCount

実際に登録した
月末資産残高の件数を返却する。

例：

```json
{
  "importedCount": 2
}
```

登録件数には、
以下を含めない。

- CSVヘッダー
- 完全な空行
- 月末資産状況の作成件数

例えば、

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,生活用口座,800000
```

を正常登録した場合は、

```json
{
  "importedCount": 2
}
```

となる。

---

#### 5.14.3 importedCountは0にならない

CSV-003では、
データ行が0件のCSVを
登録不可とする。

そのため、
正常レスポンスにおける

```text
importedCount
```

は、
1以上となる。

```json
{
  "importedCount": 0
}
```

を正常登録結果として
返却しない。

---

#### 5.14.4 返却しない情報

本APIでは、
以下の情報は返却しない。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- `month_end_asset_balance_id`
- `confirmed`
- `balance_recording_unit`
- `created_at`
- `updated_at`
- CSVファイル内容
- CSV行番号
- 登録した各資産口座名
- 登録した各`balance`
- 商品別月末評価額

登録後に
月末資産状況の詳細を確認する場合は、
SNP-003 月末資産状況詳細取得APIを使用する。

登録後に
口座単位の月末資産残高一覧を確認する場合は、
BAL-001 月末資産残高一覧取得APIを使用する。

---

#### 5.14.5 snapshotIdを返却しない

CSV-003では、
内部で

```text
month_end_asset_snapshot_id
```

を使用して
月末資産残高を登録する。

ただし、
CSV登録結果の確認に
内部のsnapshot IDは不要であるため、
レスポンスへ返却しない。

対象年月をキーとして
必要な情報を取得できる
画面フローとする。

---

#### 5.14.6 assetAccountIdを返却しない

CSV登録では、
複数の資産口座へ
月末資産残高を
一括登録する可能性がある。

そのため、
単一の

```text
assetAccountId
```

をレスポンス項目にはしない。

登録した資産口座一覧が必要な場合は、
BAL-001等の参照APIから
取得する。

---

#### 5.14.7 confirmedを返却しない

CSV-003では、
月末資産状況を
自動確定しない。

また、
確定状態の表示・取得は
月末資産状況APIの責務とする。

そのため、
CSV-003の登録結果として

```text
confirmed
```

を返却しない。

---

#### 5.14.8 レスポンス例

CSV内の
2件の月末資産残高を
正常登録した場合の
概念例は、
以下とする。

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "importedCount": 2
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

CSV-003では、
登録件数を返却することで、
利用者が
CSV登録が正常に完了したことを
確認できるようにする。

---

### 5.15 エラーレスポンス

CSV-003では、
CSV-002と異なり、
登録条件を満たさないCSVを
正常レスポンスとして返却しない。

CSV-003は
業務データを登録するAPIであるため、
登録できない状態は
APIエラーとして扱う。

概念的には、
以下とする。

```text
CSV-002
    ↓
検証・プレビュー
    ↓
業務エラーあり
    ↓
200 OK
canImport = false
```

```text
CSV-003
    ↓
再検証
    ↓
登録不可
    ↓
4xx Error
業務データ更新なし
```

CSV-003では、
エラーが発生した場合に
月末資産残高を
部分登録してはならない。

---

#### 5.15.1 エラー一覧

本APIで想定する
主なエラーは、
以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`が指定されていない |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`の形式が不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 指定された利用者が存在しない、または論理削除済み |
| `422 Unprocessable Entity` | `VALIDATION_ERROR` | `file`未指定、ファイル形式・サイズなどリクエストレベルの入力不正 |
| `422 Unprocessable Entity` | `INVALID_CSV_FORMAT` | CSV解析不能、CSVヘッダー不正 |
| `422 Unprocessable Entity` | `CSV_DATA_REQUIRED` | CSVに登録対象のデータ行が存在しない |
| `422 Unprocessable Entity` | `MULTIPLE_TARGET_YEAR_MONTHS` | 1ファイル内に複数対象年月が存在する |
| `422 Unprocessable Entity` | `ASSET_ACCOUNT_NOT_FOUND` | CSVで指定された資産口座が操作対象利用者に存在しない |
| `422 Unprocessable Entity` | `BALANCE_RECORDING_UNIT_MISMATCH` | 商品単位の資産口座が指定されている |
| `422 Unprocessable Entity` | `ASSET_ACCOUNT_NOT_AVAILABLE` | 対象年月時点で資産口座が利用できない |
| `422 Unprocessable Entity` | `INVALID_BALANCE` | 月末残高が0以上の整数ではない |
| `422 Unprocessable Entity` | `DUPLICATE_ASSET_ACCOUNT_IN_CSV` | CSV内で同一資産口座が重複している |
| `409 Conflict` | `MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED` | 対象年月の月末資産状況が確定済み |
| `409 Conflict` | `MONTH_END_ASSET_BALANCE_ALREADY_EXISTS` | 対象年月・資産口座の月末資産残高が既に存在する |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバー内部エラー |

実際の独自エラーコード名は、
`error-codes.md`の
既存命名規則と統一する。

---

#### 5.15.2 USER_CONTEXT_REQUIRED

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

この場合、
CSV解析および
データベース登録へ進まない。

---

#### 5.15.3 INVALID_USER_ID

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
0
-1
abc
1.5
```

---

#### 5.15.4 USER_NOT_FOUND

指定された利用者が
存在しない場合、
または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

他の利用者の
資産口座や
月末資産状況を使用して
登録処理を続行してはならない。

---

#### 5.15.5 file未指定

`file`が
指定されていない場合は、

```text
VALIDATION_ERROR
```

を返却する。

概念例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "file",
        "reason": "required",
        "message": "CSVファイルを指定してください。"
      }
    ]
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 5.15.6 ファイル形式不正

許可されていない
ファイル形式の場合は、

```text
VALIDATION_ERROR
```

として扱う。

例えば、
以下を対象とする。

```text
.xlsx
.xls
.pdf
```

CSV解析処理へ進まない。

---

#### 5.15.7 ファイルサイズ超過

CSV共通仕様で定める
最大ファイルサイズを超えている場合は、

```text
VALIDATION_ERROR
```

を返却する。

CSV解析や
月末資産状況検索へ進まない。

---

#### 5.15.8 空ファイル

CSVファイルが
0バイトの場合、
または有効なヘッダーを
取得できない場合は、

```text
INVALID_CSV_FORMAT
```

として扱う。

この場合、
登録処理へ進まない。

---

#### 5.15.9 CSV解析不能

CSVとして
正常に解析できない場合は、

```text
INVALID_CSV_FORMAT
```

を返却する。

例えば、
以下を対象とする。

- CSV構造が破損している
- 引用符が不正
- 列構造を正常に解釈できない
- ヘッダーを取得できない

PHPやLaravel内部の
解析エラーを
そのままレスポンスへ公開しない。

---

#### 5.15.10 CSVヘッダー不正

CSVヘッダーが
以下と一致しない場合は、

```csv
target_year_month,asset_account_name,balance
```

```text
INVALID_CSV_FORMAT
```

として扱う。

例えば、
以下は不正とする。

```csv
targetYearMonth,assetAccountName,balance
```

```csv
asset_account_name,target_year_month,balance
```

```csv
target_year_month,asset_account_name,balance,memo
```

ヘッダー不正の場合は、
後続行の業務検証へ進まない。

---

#### 5.15.11 データ行0件

CSVヘッダーは正常だが、
データ行が1件も存在しない場合は、

```text
CSV_DATA_REQUIRED
```

として登録を拒否する。

CSV-002では
プレビュー結果として
`canImport = false`を返却できるが、
CSV-003では
登録APIとして
`422 Unprocessable Entity`
を返却する。

業務データは更新しない。

---

#### 5.15.12 対象年月混在

1つのCSV内に
複数の`target_year_month`が
存在する場合は、

```text
MULTIPLE_TARGET_YEAR_MONTHS
```

として登録を拒否する。

例えば、

```csv
target_year_month,asset_account_name,balance
2026-06,普通預金,1500000
2026-07,生活用口座,800000
```

は登録しない。

---

#### 5.15.13 行単位の入力エラー

以下のような
CSV行の入力不正が存在する場合は、
CSV全体を登録しない。

- `target_year_month`未入力
- `target_year_month`形式不正
- `asset_account_name`未入力
- `balance`未入力
- `balance`整数形式不正
- `balance`小数
- `balance`負数

CSV-002と同様に
複数のエラーを収集できる設計としてよい。

ただし、
CSV-003では
エラーが1件でも存在すれば
登録処理へ進まない。

---

#### 5.15.14 資産口座不存在

CSVで指定された
`asset_account_name`に対応する
資産口座が
操作対象利用者に存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

他の利用者に
同名資産口座が存在していても、
正常とは判定しない。

---

#### 5.15.15 残高記録単位不一致

指定された資産口座が
商品単位で管理されている場合は、

```text
BALANCE_RECORDING_UNIT_MISMATCH
```

として登録を拒否する。

商品単位のデータは、
商品別月末評価額CSV登録APIで扱う。

---

#### 5.15.16 対象年月時点で資産口座が無効

指定された資産口座が
CSVの`target_year_month`時点で
利用できない場合は、

```text
ASSET_ACCOUNT_NOT_AVAILABLE
```

として登録を拒否する。

現在時点ではなく、
対象年月時点の状態によって判定する。

---

#### 5.15.17 月末残高不正

`balance`が
0以上の整数ではない場合は、

```text
INVALID_BALANCE
```

として扱う。

例えば、
以下は不正とする。

```text
-1
1000.5
abc
¥1000
1,000
```

以下は正常とする。

```text
0
1
1000
```

---

#### 5.15.18 CSV内重複

同一CSV内で
同じ資産口座が
複数回指定されている場合は、

```text
DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

として登録を拒否する。

先勝ち、
後勝ち、
残高合算などによる
自動解決は行わない。

---

#### 5.15.19 確定済み月末資産状況

CSV-003実行時点で、
対象年月の
月末資産状況が

```text
confirmed = true
```

の場合は、

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

を返却する。

HTTPステータスは、

```text
409 Conflict
```

とする。

CSV-002実行時点で
未確定であった場合でも、
CSV-003実行時点の
最新状態を優先する。

---

#### 5.15.20 既存月末資産残高

同一の

```text
month_end_asset_snapshot_id
+
asset_account_id
```

について、
既に月末資産残高が存在する場合は、

```text
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

を返却する。

HTTPステータスは、

```text
409 Conflict
```

とする。

既存レコードを
CSV値によって上書きしない。

---

#### 5.15.21 同時実行によるUNIQUE制約違反

アプリケーション側の
重複確認後に、
別リクエストによって
同じ月末資産残高が
先に登録される可能性がある。

この場合、
データベースのUNIQUE制約によって
重複登録を防止する。

概念的な一意制約は、

```text
month_end_asset_snapshot_id
+
asset_account_id
```

とする。

この制約違反は、
内部エラーとして
そのまま返却せず、

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

へ変換する。

---

#### 5.15.22 複数エラーの扱い

登録トランザクション開始前の
CSV検証段階では、
可能な範囲で
複数エラーを収集してよい。

例えば、

```text
2行目
ASSET_ACCOUNT_NOT_FOUND

4行目
INVALID_BALANCE

6行目
DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

のように、
利用者が一度に
複数箇所を修正できる形とする。

ただし、
CSV-003では
エラーが1件でも存在すれば
登録処理へ進まない。

---

#### 5.15.23 エラー詳細

CSV行に関する
複数エラーを返却する場合は、
API共通エラーの`details`を使用してよい。

概念例：

```json
{
  "error": {
    "code": "CSV_IMPORT_VALIDATION_FAILED",
    "message": "CSVの内容に誤りがあります。",
    "details": [
      {
        "rowNumber": 2,
        "field": "asset_account_name",
        "code": "ASSET_ACCOUNT_NOT_FOUND",
        "message": "指定された資産口座が存在しません。"
      },
      {
        "rowNumber": 4,
        "field": "balance",
        "code": "INVALID_BALANCE",
        "message": "月末残高は0以上の整数で指定してください。"
      }
    ]
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

具体的な
集約エラーコードの有無や
`details`構造は、
API共通エラー仕様に従う。

---

#### 5.15.24 INTERNAL_SERVER_ERROR

CSV解析、
月末資産状況作成、
月末資産残高登録などで
想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスには、
以下の内部情報を含めない。

- SQL
- PostgreSQLのエラー内容
- 制約名
- テーブル名
- カラム名
- PHP内部エラー
- Laravel内部例外
- スタックトレース
- サーバーファイルパス

詳細情報は、
サーバーログへ記録する。

---

### 5.16 HTTPステータス

本APIで使用する
HTTPステータスは、
以下とする。

| HTTPステータス | 用途 |
|---|---|
| `201 Created` | CSV内の月末資産残高を全件登録できた |
| `400 Bad Request` | 利用者コンテキストの指定不備 |
| `404 Not Found` | 指定された利用者が存在しない |
| `409 Conflict` | 確定済み月末資産状況、既存月末資産残高など現在状態との競合 |
| `422 Unprocessable Entity` | CSVファイル・CSV内容・業務入力値が登録条件を満たさない |
| `500 Internal Server Error` | 想定外のサーバー内部エラー |

CSV-003は
一括登録APIであり、
登録不可のCSVに対して

```text
200 OK
canImport = false
```

を返却しない。

---

#### 5.16.1 201 Created

CSV内の
すべてのデータについて
検証および登録が成功した場合は、

```text
201 Created
```

を返却する。

対象年月の
月末資産状況を
CSV-003内で新規作成した場合も、
同じステータスとする。

---

#### 5.16.2 400 Bad Request

以下の場合に使用する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
```

CSV内容の
入力エラーには使用しない。

---

#### 5.16.3 404 Not Found

指定された利用者が
存在しない、
または論理削除済みの場合に使用する。

```text
USER_NOT_FOUND
```

CSV内で指定された
資産口座が存在しない場合には
`404 Not Found`を使用しない。

CSV内容の登録不成立として
`422 Unprocessable Entity`を使用する。

---

#### 5.16.4 409 Conflict

リクエスト内容自体は
解釈可能だが、
現在の業務データ状態と
競合して登録できない場合に使用する。

主に以下を対象とする。

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

並行登録による
UNIQUE制約違反も、
既存月末資産残高との競合として
`409 Conflict`へ変換する。

---

#### 5.16.5 422 Unprocessable Entity

CSVファイルまたは
CSV内容が
登録条件を満たさない場合に使用する。

主に以下を対象とする。

- `file`未指定
- ファイル形式不正
- ファイルサイズ超過
- CSV解析不能
- CSVヘッダー不正
- データ行0件
- 対象年月形式不正
- 対象年月混在
- 資産口座名未入力
- 資産口座不存在
- 残高記録単位不一致
- 対象年月時点の資産口座無効
- 月末残高不正
- CSV内重複

---

#### 5.16.6 500 Internal Server Error

想定外の
サーバー内部エラーが
発生した場合に使用する。

対象となる
独自エラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

トランザクション中の場合は、
すべてロールバックしたうえで
エラーを返却する。

---

### 5.17 副作用

本APIには、
業務データに対する
副作用がある。

正常終了した場合は、
CSV内容に基づいて

```text
month_end_asset_balances
```

へ
月末資産残高を新規登録する。

また、
対象年月の
月末資産状況が存在しない場合は、

```text
month_end_asset_snapshots
```

を新規作成する。

---

#### 5.17.1 正常終了時の更新

対象年月の
月末資産状況が
既に存在する場合は、
概念的に以下となる。

```text
month_end_asset_snapshots
    → 更新なし

month_end_asset_balances
    → CSV行数分INSERT
```

対象年月の
月末資産状況が
存在しない場合は、

```text
month_end_asset_snapshots
    → 1件INSERT

month_end_asset_balances
    → CSV行数分INSERT
```

となる。

---

#### 5.17.2 更新しないデータ

CSV-003では、
以下を更新しない。

- `users`
- `asset_accounts`
- `holding_assets`
- `month_end_holding_values`
- `asset_account_available_settings`
- `net_incomes`
- `objectives`
- `assessment_histories`

また、
既存の

```text
month_end_asset_balances.balance
```

も更新しない。

---

#### 5.17.3 confirmedを変更しない

CSV登録によって
月末資産状況の

```text
confirmed
```

を変更しない。

新規作成する場合は、

```text
confirmed = false
```

とする。

既存snapshotについても、
CSV-003によって
確定状態を変更しない。

---

#### 5.17.4 エラー時の副作用

CSV解析・検証段階で
エラーとなった場合は、
業務データを
一切更新しない。

トランザクション開始後に
エラーが発生した場合は、
すべてロールバックする。

そのため、

```text
一部のbalanceだけ登録済み
```

または、

```text
snapshotだけ新規作成済み
```

という状態を
残してはならない。

---

### 5.18 トランザクション

CSV-003では、
登録処理を
データベーストランザクション内で実行する。

CSV全体の
入力・業務検証を行った後、
登録可能であることを確認してから
トランザクションを開始する。

概念的には、
以下とする。

```text
CSV解析
    ↓
全件検証
    ↓
登録可能
    ↓
BEGIN
    ↓
月末資産状況取得
    ↓
確定状態再確認
    ↓
必要ならsnapshot作成
    ↓
既存月末資産残高再確認
    ↓
月末資産残高一括登録
    ↓
COMMIT
```

---

#### 5.18.1 同一トランザクションで行う処理

主に以下を
同一トランザクション内で行う。

- 月末資産状況の取得
- 確定状態の最終確認
- 月末資産状況の必要時作成
- 既存月末資産残高の最終確認
- 月末資産残高の一括登録

これにより、
月末資産状況作成と
月末資産残高登録を
原子的に処理する。

---

#### 5.18.2 ロールバック

トランザクション内で
業務例外または
想定外例外が発生した場合は、
すべてロールバックする。

例えば、

```text
snapshot新規作成
    ↓
1件目balance登録
    ↓
2件目balance登録時にエラー
    ↓
ROLLBACK
```

となった場合は、

- 新規snapshot
- 1件目のbalance

の両方を
データベースへ残さない。

---

#### 5.18.3 CSV検証処理をトランザクションへ入れすぎない

CSVファイルの

- 読み込み
- ヘッダー検証
- 入力値検証

など、
データベース更新を必要としない処理は
原則として
トランザクション開始前に行う。

CSV解析中ずっと
トランザクションを保持しない。

これにより、
トランザクション時間を
必要最小限にする。

---

### 5.19 ロック

CSV-003では、
同一利用者・同一対象年月への
並行登録を考慮する。

登録トランザクション内では、
既存の月末資産状況が存在する場合、
必要に応じて
対象snapshotを
行ロックしてよい。

概念例：

```php
$snapshot =
    MonthEndAssetSnapshot::query()
        ->where(
            'user_id',
            $userId,
        )
        ->where(
            'target_year_month',
            $targetYearMonth,
        )
        ->lockForUpdate()
        ->first();
```

これにより、
同じsnapshotに対する
確定処理やCSV登録処理との
競合を制御する。

---

#### 5.19.1 snapshot不存在時の競合

対象年月のsnapshotが
存在しない状態で、
複数のCSV-003が
同時実行される可能性がある。

この場合、
双方が

```text
snapshot不存在
```

と判定して
重複作成しないよう、
以下の組み合わせに対する
データベースの一意制約を
最終防衛線とする。

```text
month_end_asset_snapshots.user_id
+
month_end_asset_snapshots.target_year_month
```

競合によって
一意制約違反となった場合は、
トランザクションをロールバックし、
必要に応じて
最新状態を再取得したうえで
適切な業務エラーへ変換する。

---

#### 5.19.2 月末資産残高の重複防止

月末資産残高についても、
以下の組み合わせに
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
asset_account_id
```

アプリケーション側の
事前重複確認を
複数リクエストが同時に通過しても、
データベース制約によって
重複登録を防止する。

---

#### 5.19.3 過剰なロックを行わない

CSV-003では、
操作対象利用者の
すべての資産口座を
長時間ロックするような
実装は避ける。

排他制御は、
登録対象となる
月末資産状況を中心に
必要最小限とする。

CSV解析処理中には
行ロックを取得しない。

---

### 5.20 キャッシュ

Phase1では、
CSV-003専用の
サーバー側アプリケーションキャッシュを
使用しない。

CSV登録では、

- 月末資産状況
- 月末資産残高
- 資産口座状態

の最新情報を
確認する必要があるためである。

古いキャッシュを利用して
登録可否を判定してはならない。

---

#### 5.20.1 登録後のキャッシュ

将来的に、
月末資産状況や
資産状況APIで
サーバー側キャッシュを
採用する場合は、
CSV-003成功時に
対象年月に関係するキャッシュを
無効化する必要がある。

Phase1では、
サーバー側キャッシュを
採用しないため、
明示的なキャッシュ削除処理も
実装しない。

---

### 5.21 冪等性

本APIは、
HTTP POSTを使用して
新しい月末資産残高を
一括登録するAPIである。

そのため、
HTTPメソッドとしては
冪等ではない。

同じCSVを
複数回実行した場合、
2回目に
同じ月末資産残高を
新たに登録してはならない。

---

#### 5.21.1 同一CSVの再実行

例えば、
以下のCSVを送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,生活用口座,800000
```

1回目は、

```text
CSV-003
    ↓
201 Created
importedCount = 2
```

となる。

同じCSVを
再度実行した場合は、

```text
CSV-003
    ↓
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

となる。

既存データを返して
成功扱いにはしない。

---

#### 5.21.2 値が異なる場合も上書きしない

既に、

```text
普通預金
balance = 1500000
```

が登録されている状態で、

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1600000
```

をCSV-003へ送信しても、
既存値を更新しない。

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

として扱う。

残高を変更する場合は、
BAL-003 月末資産残高更新APIを使用する。

---

#### 5.21.3 一部だけ既存の場合

CSV内の一部の資産口座について
月末資産残高が
既に存在する場合も、
未登録行だけを登録しない。

例えば、

```text
普通預金
    → 登録済み

生活用口座
    → 未登録
```

の状態で
2件を含むCSVを送信した場合は、

```text
CSV全体
    ↓
409 Conflict
    ↓
新規登録0件
```

とする。

全件成功または
全件失敗のルールを維持する。

---

#### 5.21.4 Idempotency-Key

Phase1では、
`Idempotency-Key`を採用しない。

意図しない二重登録は、

- アプリケーション側の重複確認
- トランザクション
- データベースUNIQUE制約
- フロントエンドの二重送信防止

によって制御する。

---

#### 5.21.5 二重送信

フロントエンドでは、
CSV-003実行中に
登録ボタンを非活性化し、
意図しない連続送信を防止する。

ただし、
フロントエンドの制御だけを
重複登録防止の保証としてはならない。

バックエンドおよび
データベース制約によって
最終的な整合性を保証する。

---

#### 5.21.6 通信失敗後の再送

CSV-003実行後に
クライアントが
レスポンスを受信できなかった場合、
サーバー側では
登録が完了している可能性がある。

その状態で
同じCSVを再送した場合は、

```text
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

となる可能性がある。

Phase1では、
`Idempotency-Key`を
採用しないため、
再送を
初回リクエストと同じ成功結果へ
自動的に変換しない。

必要に応じて、
登録後にBAL-001等の参照APIから
現在状態を再取得する。

---

#### 5.21.7 同時実行

同一CSVまたは
同一対象年月・同一資産口座を含む
複数のCSV-003が
同時実行された場合でも、
同一の月末資産残高を
複数件登録してはならない。

最終的には、

```text
UNIQUE (
    month_end_asset_snapshot_id,
    asset_account_id
)
```

によって
1件のみ登録可能とする。

競合したリクエストは、

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

として処理する。

本API自体は
非冪等であるが、
重複データを
許容するという意味ではない。

---



---



---



---



---

### 5.22 関連テーブル

CSV-003では、
月末資産残高CSVの内容を検証し、
登録可能な場合に
月末資産残高を一括登録するため、
以下のテーブルを使用する。

| テーブル | 用途 |
|---|---|
| `users` | 操作対象利用者の存在確認 |
| `asset_accounts` | CSVで指定された資産口座の存在・利用者境界・残高記録単位確認 |
| `month_end_asset_snapshots` | 対象年月の月末資産状況確認、必要時の新規作成、確定状態確認 |
| `month_end_asset_balances` | 既存残高の重複確認およびCSV内容の一括登録 |

本APIでは、
`month_end_asset_snapshots`および
`month_end_asset_balances`を
更新対象とする。

一方、
`users`および
`asset_accounts`は
参照のみとする。 :contentReference[oaicite:0]{index=0}

---

#### 5.22.1 users

`X-User-Id`で指定された
操作対象利用者の
存在確認に使用する。

概念的な条件は、
以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```
本APIでは、`users`を更新しない。

---

#### 5.22.2 asset_accounts

CSVの`asset_account_name`から、登録対象となる資産口座を特定するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
| --- | --- |
| `id` | `month_end_asset_balances.asset_account_id`へ設定 |
| `user_id` | 利用者境界確認 |
| `name` | CSVの`asset_account_name`との照合 |
| `balance_recording_unit` | 月末資産残高CSVの登録対象か判定 |
| `deleted_at` | 論理削除済み資産口座を除外 |

概念的な取得条件は、以下とする。

```text
user_id
    = 操作対象利用者ID

AND

name
    = CSVのasset_account_name

AND

deleted_at
    IS NULL
```

他の利用者に同名資産口座が存在していても、取得対象へ含めない。

---

#### 5.22.3 残高記録単位

`asset_accounts.balance_recording_unit`を使用して、対象資産口座が口座単位であることを確認する。

本APIでは、

```text
口座単位
```

の資産口座のみを登録対象とする。

商品単位の資産口座については、`month_end_asset_balances`へ登録しない。

---

#### 5.22.4 対象年月時点の資産口座

対象年月時点で資産口座が月末資産管理対象として有効であることを確認する。

対象年月時点の有効性判定に追加テーブルを使用する設計の場合は、そのテーブルも関連テーブルへ含める。

具体的な判定方法は、資産口座管理の機能要件およびテーブル定義に従う。

現在時点の状態だけを基準に登録可否を判定しない。

---

#### 5.22.5 month_end_asset_snapshots

CSVの`target_year_month`に対応する月末資産状況を取得するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
| --- | --- |
| `id` | 月末資産残高の登録先snapshot |
| `user_id` | 利用者境界確認 |
| `target_year_month` | CSVの対象年月との照合 |
| `confirmed` | CSV登録可否判定 |

概念的な取得条件は、以下とする。

```text
user_id
    = 操作対象利用者ID

AND

target_year_month
    = CSVのtarget_year_month
```

他の利用者に同一対象年月の月末資産状況が存在していても、取得対象へ含めない。

---

#### 5.22.6 月末資産状況が存在する場合

対象年月の月末資産状況が存在する場合は、既存レコードを使用する。

ただし、

```text
confirmed = true
```

の場合は、CSV登録を許可しない。

未確定の場合のみ、月末資産残高の登録先として使用する。

---

#### 5.22.7 月末資産状況が存在しない場合

対象年月の月末資産状況が存在しない場合は、登録トランザクション内で新規作成する。

概念的には、以下を設定する。

```text
user_id
    = 操作対象利用者ID

target_year_month
    = CSVのtarget_year_month

confirmed
    = false
```

作成した`month_end_asset_snapshots.id`を、CSV各行の`month_end_asset_balances.month_end_asset_snapshot_id`として使用する。

---

#### 5.22.8 month_end_asset_snapshotsの一意性

同一利用者、同一対象年月について複数の月末資産状況が作成されないようにする。

概念的には、以下の組み合わせに一意性を保証する。

```text
user_id
+
target_year_month
```

同時実行によって月末資産状況が重複作成されないよう、データベース制約を最終防衛線として使用する。

---

#### 5.22.9 month_end_asset_balances

CSV内容に基づいて、口座単位の月末資産残高を新規登録するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
| --- | --- |
| `id` | 月末資産残高ID |
| `month_end_asset_snapshot_id` | 対象年月の月末資産状況 |
| `asset_account_id` | 登録対象資産口座 |
| `balance` | CSVの月末残高 |
| `created_at` | 登録日時 |
| `updated_at` | 更新日時 |

CSV各行について、概念的に以下を登録する。

```text
month_end_asset_snapshot_id
    = 対象snapshot.id

asset_account_id
    = asset_account_nameから特定したasset_accounts.id

balance
    = CSVのbalance
```

---

#### 5.22.10 既存月末資産残高の確認

登録前に、同一の

```text
month_end_asset_snapshot_id
+
asset_account_id
```

について、既存の`month_end_asset_balances`が存在しないことを確認する。

既存データが存在する場合は、新規登録しない。

CSV-003では、既存の`balance`を更新・上書きしない。

---

#### 5.22.11 month_end_asset_balancesの一意性

同一snapshot、同一資産口座について複数の月末資産残高が登録されないようにする。

概念的には、以下の組み合わせに一意性を保証する。

```sql
UNIQUE (
    month_end_asset_snapshot_id,
    asset_account_id
)
```

アプリケーション側の重複確認後に並行リクエストが発生した場合でも、データベース制約によって重複登録を防止する。

---

#### 5.22.12 参照しないテーブル

月末資産残高CSV登録では、以下のテーブルを直接登録対象としない。

- `holding_assets`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

商品単位の商品別月末評価額は、CSV-006 商品別月末評価額CSV登録で扱う。

CSV-003では、`month_end_holding_values`へデータを登録しない。

---

#### 5.22.13 更新対象テーブル

正常終了時に更新する可能性があるテーブルは、以下とする。

```text
month_end_asset_snapshots
month_end_asset_balances
```

`month_end_asset_snapshots`は、対象年月のsnapshotが存在しない場合のみ新規作成する。

`month_end_asset_balances`は、CSVのデータ行数分を新規登録する。

---

#### 5.22.14 更新しないテーブル

以下のテーブルは、CSV-003では更新しない。

- `users`
- `asset_accounts`
- `holding_assets`
- `month_end_holding_values`
- `asset_account_available_settings`
- `net_incomes`
- `objectives`
- `assessment_histories`

資産口座の設定や利用可能資産設定をCSV登録処理によって変更してはならない。

---

### 5.23 関連する機能要件

CSV-003は、月末資産残高のCSV一括登録に関する機能要件と対応する。

主な関連要件は、以下とする。

- CSVインポート
  - CSVテンプレートを使用してデータを入力できる
  - 登録前にCSV内容をプレビューできる
  - CSV登録時に入力内容を再検証する
  - 1つのCSVファイルでは1つの対象年月のみを扱う
  - CSV内にエラーが存在する場合は一部登録しない
  - CSV登録は全件成功または全件失敗とする
- 月末資産残高
  - 口座単位で管理する資産口座について月末残高を登録できる
  - 月末残高は日本円の整数として扱う
  - 0円を有効な月末残高として扱う
  - 同一対象年月・同一資産口座への重複登録を防止する
  - 既存月末資産残高をCSVで上書きしない
- 月末資産状況
  - 対象年月ごとに月末資産状況を管理する
  - 対象年月の月末資産状況が存在しない場合は登録処理で作成できる
  - 新規作成する月末資産状況は未確定とする
  - 確定済み月末資産状況へ月末資産残高を追加できない
  - CSV登録によって月末資産状況を自動確定しない
- 資産口座
  - 操作対象利用者に属する資産口座のみ登録対象とする
  - CSVでは資産口座名を使用して対象口座を特定する
  - 残高記録単位が口座単位の資産口座のみ対象とする
  - 対象年月時点で有効な資産口座のみ登録対象とする
- 利用者境界
  - 他利用者の資産口座を登録対象にしない
  - 他利用者の月末資産状況を使用しない
  - 他利用者の月末資産残高を重複判定へ使用しない
- トランザクション
  - CSV内の複数行を一括して登録する
  - 登録途中でエラーが発生した場合は全件ロールバックする
  - 月末資産状況を新規作成した場合も同一トランザクションで扱う

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

### 5.24 テスト観点

#### 5.24.1 正常系

正常なCSVを送信する。

例：

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,生活用口座,800000
```

以下を確認する。

- `201 Created`となること
- `targetYearMonth = 2026-07`となること
- `importedCount = 2`となること
- 2件の`month_end_asset_balances`が登録されること
- 登録された`balance`がCSVと一致すること
- 各行が正しい`asset_account_id`へ紐づくこと
- 各行が同一の`month_end_asset_snapshot_id`へ紐づくこと
- CSV登録によって月末資産状況が確定されないこと

---

#### 5.24.2 CSV-001との整合性

CSV-001で取得したテンプレートへ正常なデータを入力し、CSV-003へ送信する。

以下を確認する。

```text
CSV-001
テンプレートヘッダー
    =
CSV-003
受付ヘッダー
```

正常なテンプレートがヘッダー不正にならないこと。

---

#### 5.24.3 CSV-002との整合性

同一システム状態で、同一CSVをCSV-002およびCSV-003へ送信する。

CSV-002で

```text
canImport = true
```

の場合に、CSV-003でも登録可能となることを確認する。

CSV-002とCSV-003でCSV解析・業務ルールの実装差異がないことを確認する。

---

#### 5.24.4 file未指定

`file`を指定せずにCSV-003を実行する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

以下も確認する。

- CSV解析を行わないこと
- snapshotを作成しないこと
- balanceを登録しないこと

---

#### 5.24.5 CSV以外のファイル

例えば、

```text
test.xlsx
```

を指定する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

業務データが更新されないこと。

---

#### 5.24.6 ファイルサイズ超過

CSV共通仕様で定める上限を超えるファイルを送信する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

登録処理へ進まないこと。

---

#### 5.24.7 空ファイル

0バイトのCSVファイルを送信する。

期待結果：

```text
422 Unprocessable Entity
INVALID_CSV_FORMAT
```

以下が新規作成されないこと。

- `month_end_asset_snapshots`
- `month_end_asset_balances`

---

#### 5.24.8 ヘッダー不正

以下のようなCSVを送信する。

```csv
targetYearMonth,assetAccountName,balance
2026-07,普通預金,1500000
```

期待結果：

```text
422 Unprocessable Entity
INVALID_CSV_FORMAT
```

登録処理へ進まないこと。

---

#### 5.24.9 ヘッダー順序不正

以下を送信する。

```csv
asset_account_name,target_year_month,balance
普通預金,2026-07,1500000
```

期待結果：

```text
422 Unprocessable Entity
INVALID_CSV_FORMAT
```

業務データが更新されないこと。

---

#### 5.24.10 余分なヘッダー

以下を送信する。

```csv
target_year_month,asset_account_name,balance,memo
2026-07,普通預金,1500000,test
```

期待結果：

```text
422 Unprocessable Entity
INVALID_CSV_FORMAT
```

となること。

---

#### 5.24.11 データ行0件

以下を送信する。

```csv
target_year_month,asset_account_name,balance
```

期待結果：

```text
422 Unprocessable Entity
CSV_DATA_REQUIRED
```

以下を確認する。

- `importedCount = 0`の正常レスポンスとしないこと
- snapshotを作成しないこと
- balanceを登録しないこと

---

#### 5.24.12 target_year_month未入力

以下を送信する。

```csv
target_year_month,asset_account_name,balance
,普通預金,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと
- snapshotが作成されないこと

---

#### 5.24.13 target_year_month形式不正

以下の値をそれぞれテストする。

```text
2026-1
2026/07
202607
2026-00
2026-13
abc
```

以下を確認する。

- `422 Unprocessable Entity`となること
- 月末資産残高が登録されないこと

---

#### 5.24.14 対象年月混在

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-06,普通預金,1500000
2026-07,生活用口座,800000
```

期待結果：

```text
422 Unprocessable Entity
MULTIPLE_TARGET_YEAR_MONTHS
```

CSV全体が登録されないこと。

---

#### 5.24.15 asset_account_name未入力

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 5.24.16 資産口座不存在

操作対象利用者に存在しない資産口座を指定する。

```csv
target_year_month,asset_account_name,balance
2026-07,存在しない口座,1500000
```

期待結果：

```text
422 Unprocessable Entity
ASSET_ACCOUNT_NOT_FOUND
```

月末資産残高が登録されないこと。

---

#### 5.24.17 他利用者にのみ同名資産口座が存在する

以下の状態を用意する。

```text
User A
普通預金なし

User B
普通預金あり
```

User Aとして、

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
```

を送信する。

期待結果：

```text
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの資産口座へ月末資産残高が登録されないこと。

---

#### 5.24.18 論理削除済み資産口座

`deleted_at`が設定された資産口座を指定する。

以下を確認する。

- 有効な資産口座として扱われないこと
- CSV全体が登録されないこと
- 論理削除済み資産口座へ残高が登録されないこと

---

#### 5.24.19 残高記録単位が口座単位

対象資産口座の

```text
balance_recording_unit
    = 口座単位
```

とする。

他の条件が正常な場合は、月末資産残高を正常に登録できること。

---

#### 5.24.20 残高記録単位が商品単位

商品単位で管理する資産口座を指定する。

期待結果：

```text
422 Unprocessable Entity
BALANCE_RECORDING_UNIT_MISMATCH
```

以下を確認する。

- `month_end_asset_balances`へ登録されないこと
- `month_end_holding_values`へも登録されないこと

---

#### 5.24.21 対象年月時点で資産口座が有効

CSVの対象年月時点で有効な資産口座を指定する。

他の条件が正常であれば、登録できること。

---

#### 5.24.22 対象年月時点で資産口座が無効

CSVの対象年月時点で利用できない資産口座を指定する。

期待結果：

```text
422 Unprocessable Entity
ASSET_ACCOUNT_NOT_AVAILABLE
```

現在時点で有効であっても、対象年月時点で無効なら登録できないこと。

---

#### 5.24.23 balance未入力

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 5.24.24 balanceが文字列

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,abc
```

期待結果：

```text
422 Unprocessable Entity
INVALID_BALANCE
```

となること。

---

#### 5.24.25 balanceが小数

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1000.5
```

期待結果：

```text
422 Unprocessable Entity
INVALID_BALANCE
```

となること。

---

#### 5.24.26 balanceが負数

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,-1
```

期待結果：

```text
422 Unprocessable Entity
INVALID_BALANCE
```

となること。

---

#### 5.24.27 balanceが0円

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,0
```

他の条件が正常であれば、登録できること。

登録後に、

```text
balance = 0
```

として保持されること。

0円を未入力扱いしないこと。

---

#### 5.24.28 桁区切り付きbalance

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,"1,500,000"
```

期待結果：

```text
422 Unprocessable Entity
INVALID_BALANCE
```

となること。

---

#### 5.24.29 通貨記号付きbalance

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,¥1500000
```

期待結果：

```text
422 Unprocessable Entity
INVALID_BALANCE
```

となること。

---

#### 5.24.30 月末資産状況が存在しない

対象年月の`month_end_asset_snapshots`が存在しない状態で、正常なCSVを送信する。

以下を確認する。

- `201 Created`となること
- `month_end_asset_snapshots`が1件作成されること
- `user_id`が操作対象利用者となること
- `target_year_month`がCSV対象年月となること
- `confirmed = false`となること
- CSV行数分の月末資産残高が登録されること

---

#### 5.24.31 月末資産状況が未確定

対象年月について、

```text
confirmed = false
```

の月末資産状況を用意する。

正常CSVを送信し、既存snapshotへ月末資産残高が登録されること。

新しいsnapshotが追加作成されないこと。

---

#### 5.24.32 月末資産状況が確定済み

対象年月について、

```text
confirmed = true
```

の月末資産状況を用意する。

期待結果：

```text
409 Conflict
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

以下を確認する。

- 月末資産残高が登録されないこと
- `confirmed`が変更されないこと

---

#### 5.24.33 他利用者の同一対象年月

以下を用意する。

```text
User A
2026-07 snapshotなし

User B
2026-07 snapshotあり
```

User Aとして正常CSVを登録する。

以下を確認する。

- User Bのsnapshotを使用しないこと
- 必要であればUser A用snapshotが新規作成されること
- User Aの残高として登録されること

---

#### 5.24.34 他利用者の同一対象年月が確定済み

以下を用意する。

```text
User A
2026-07 snapshotなし

User B
2026-07
confirmed = true
```

User AとしてCSV-003を実行する。

User Bの確定状態によってUser Aの登録が拒否されないこと。

---

#### 5.24.35 既存月末資産残高

同一対象年月、同一資産口座について、既に月末資産残高を用意する。

期待結果：

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

以下を確認する。

- 既存`balance`が変更されないこと
- CSV値で上書きされないこと
- 新規残高が追加されないこと

---

#### 5.24.36 既存値と同じbalance

既存の月末資産残高とCSVの`balance`が同じ場合でも、成功扱いにしない。

期待結果：

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

とする。

CSV-003を疑似的なupsert APIとして扱わないこと。

---

#### 5.24.37 既存値と異なるbalance

既存残高が

```text
1500000
```

で、CSVに

```text
1600000
```

が指定されている場合も、

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

となること。

既存値が`1600000`へ更新されないこと。

---

#### 5.24.38 一部だけ既存

以下の状態を用意する。

```text
普通預金
    → 登録済み

生活用口座
    → 未登録
```

両方を含むCSVを送信する。

期待結果：

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

以下を確認する。

- 生活用口座だけを登録しないこと
- CSV全体が失敗すること
- 新規登録件数が0件であること

---

#### 5.24.39 CSV内重複

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,普通預金,1600000
```

期待結果：

```text
422 Unprocessable Entity
DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

以下を確認する。

- どちらの行も登録されないこと
- 先勝ち・後勝ちにならないこと

---

#### 5.24.40 複数エラー

以下のようなCSVを送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,存在しない口座,100000
2026-07,普通預金,-1
2026-07,,500000
```

可能な範囲で複数エラーが返却されることを確認する。

以下も確認する。

- CSV全体が登録されないこと
- snapshotが新規作成されないこと
- balanceが1件も登録されないこと

---

#### 5.24.41 一部登録されないこと

3行中、2行が正常で1行がエラーとなるCSVを送信する。

以下を確認する。

```text
正常行
    → INSERTされない

エラー行
    → INSERTされない
```

CSV全体で新規登録0件となること。

---

#### 5.24.42 全件検証後に登録されること

CSV前半の行が正常で、末尾行がエラーとなるCSVを送信する。

以下を確認する。

- 前半の正常行が先に登録されないこと
- エラー発見時点でDBに部分データが存在しないこと

---

#### 5.24.43 トランザクション成功

対象年月のsnapshotが存在しない状態で、複数行の正常CSVを送信する。

以下を確認する。

```text
snapshot INSERT
+
balance複数件 INSERT
    ↓
COMMIT
```

となること。

すべてのデータが正常に保存されること。

---

#### 5.24.44 トランザクションロールバック

snapshot作成後、月末資産残高登録中に意図的に例外を発生させる。

以下を確認する。

```text
snapshot INSERT
    ↓
balance INSERT
    ↓
例外
    ↓
ROLLBACK
```

結果として、以下を確認する。

- 新規snapshotが残らないこと
- 一部のbalanceが残らないこと

---

#### 5.24.45 既存snapshot使用時のロールバック

既存の未確定snapshotへ複数件登録する途中で例外を発生させる。

以下を確認する。

- 新規登録したbalanceがすべてロールバックされること
- 既存snapshot自体は残ること
- snapshotの`confirmed`が変更されないこと

---

#### 5.24.46 snapshot重複作成防止

同一利用者、同一対象年月についてCSV-003を並行実行する。

以下を確認する。

- `month_end_asset_snapshots`が重複作成されないこと
- `user_id + target_year_month`の一意性が維持されること
- 不整合なsnapshotが残らないこと

---

#### 5.24.47 月末資産残高の同時登録

同一snapshot、同一資産口座について複数リクエストを並行実行する。

以下を確認する。

- 同一資産口座の残高が複数件登録されないこと
- UNIQUE制約によって重複が防止されること
- 競合したリクエストが適切な`409 Conflict`となること

---

#### 5.24.48 CSV登録後も未確定

CSV登録に成功した後、対象snapshotの

```text
confirmed
```

を確認する。

以下となること。

```text
confirmed = false
```

CSV-003によって自動確定されないこと。

---

#### 5.24.49 CSVに含まれない資産口座

対象年月時点で口座単位の資産口座が3件存在する状態で、そのうち2件だけをCSVへ含める。

CSVに含まれた2件が他の条件を満たしている場合は、登録できること。

CSVに含まれていない残り1件を理由としてCSV-003が失敗しないこと。

未登録データの確認や確定可否判定は、月末資産状況側の責務とする。

---

#### 5.24.50 商品別月末評価額への副作用

CSV-003実行前後で、

```text
month_end_holding_values
```

が変更されないことを確認する。

商品単位データを誤って登録・更新・削除しないこと。

---

#### 5.24.51 利用者コンテキスト未指定

`X-User-Id`を指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

以下を確認する。

- CSV解析へ進まないこと
- 業務データを更新しないこと

---

#### 5.24.52 利用者ID形式不正

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

#### 5.24.53 利用者不存在

存在しない利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 5.24.54 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 5.24.55 他利用者データの非更新

User AとしてCSV-003を実行する。

実行前後で、User Bに属する以下が変更されないことを確認する。

- `asset_accounts`
- `month_end_asset_snapshots`
- `month_end_asset_balances`

---

#### 5.24.56 正常レスポンス契約

正常登録時、API共通の成功Envelope形式で返却されることを確認する。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "importedCount": 2
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

以下を確認する。

- `201 Created`であること
- `data`がobjectであること
- `targetYearMonth`がstringであること
- `importedCount`がintegerであること
- `importedCount >= 1`であること
- JSONフィールド名がcamelCaseであること
- `requestId`が設定されること

---

#### 5.24.57 返却しない情報

正常レスポンスに、以下の情報が含まれていないことを確認する。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- `month_end_asset_balance_id`
- `confirmed`
- `balance_recording_unit`
- `created_at`
- `updated_at`
- CSVファイル内容
- CSV行番号
- 登録した各資産口座名
- 登録した各`balance`
- 商品別月末評価額

---

#### 5.24.58 エラーレスポンス契約

以下の代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

- `USER_CONTEXT_REQUIRED`
- `INVALID_USER_ID`
- `USER_NOT_FOUND`
- `VALIDATION_ERROR`
- `INVALID_CSV_FORMAT`
- `CSV_DATA_REQUIRED`
- `MULTIPLE_TARGET_YEAR_MONTHS`
- `ASSET_ACCOUNT_NOT_FOUND`
- `BALANCE_RECORDING_UNIT_MISMATCH`
- `ASSET_ACCOUNT_NOT_AVAILABLE`
- `INVALID_BALANCE`
- `DUPLICATE_ASSET_ACCOUNT_IN_CSV`
- `MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED`
- `MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`
- `INTERNAL_SERVER_ERROR`

以下も確認する。

- `error.code`が設定されること
- `error.message`が設定されること
- 必要に応じて`error.details`が設定されること
- `requestId`が設定されること
- SQLが含まれないこと
- PostgreSQLの制約名が含まれないこと
- スタックトレースが含まれないこと
- サーバー内部ファイルパスが含まれないこと

---

#### 5.24.59 同一CSVの再実行

正常なCSVを1回登録した後、同じCSVを再度送信する。

1回目：

```text
201 Created
```

2回目：

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

となること。

同じレコードが重複登録されないこと。

---

#### 5.24.60 通信失敗後の再送

サーバー側ではCSV登録が完了したが、クライアントが正常レスポンスを受信できなかった状態を想定する。

同じCSVを再送した場合に、

```text
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

となり得ることを確認する。

再送によって重複データが作成されないこと。

---

#### 5.24.61 Idempotency-Keyなし

`Idempotency-Key`を指定せずに正常登録できることを確認する。

また、`Idempotency-Key`を重複防止の前提として実装していないことを確認する。

重複防止は、

```text
アプリケーション側重複確認
+
トランザクション
+
UNIQUE制約
```

によって保証する。

---

#### 5.24.62 INTERNAL_SERVER_ERROR

登録処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

- トランザクションがロールバックされること
- 一部登録データが残らないこと
- 内部情報がレスポンスへ公開されないこと

---

### 5.25 Laravel実装方針

CSV-003では、
Action、
Request、
UseCase、
CSV Definition、
CSV Parser、
CSV Validator、
Query、
Repository、
DTO、
API Resource、
Responderを分離して実装する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
Middleware
    ↓
Request
    ↓
Action
    ↓
UseCase
    ├─ CSV Parser
    ├─ CSV Validator
    ├─ AssetAccountQuery
    ├─ MonthEndAssetSnapshotQuery
    ├─ MonthEndAssetBalanceQuery
    ├─ MonthEndAssetSnapshotRepository
    └─ MonthEndAssetBalanceRepository
    ↓
Import Result DTO
    ↓
API Resource
    ↓
Responder
```

CSV-003では、
CSV-002と同じ
CSV解析・検証ロジックを
可能な限り共通利用する。

CSV-003固有の責務は、

```text
検証済みデータ
    ↓
登録直前の業務状態再確認
    ↓
トランザクション
    ↓
必要に応じてsnapshot作成
    ↓
月末資産残高一括登録
```

とする。

Actionへ、

- CSV解析
- CSVヘッダー検証
- 行入力値検証
- 資産口座検索
- 月末資産状況検索
- 重複判定
- トランザクション制御
- 一括登録

を直接記述しない。

---

#### 5.25.1 Action

HTTPリクエストを受け付け、
検証済みCSVファイルおよび
利用者コンテキストを取得する。

CSV登録UseCaseを呼び出し、
登録結果をResponderへ渡す。

概念例：

```php
final class ImportMonthEndAssetBalanceCsvAction
{
    public function __invoke(
        ImportMonthEndAssetBalanceCsvRequest $request,
        ImportMonthEndAssetBalanceCsvUseCase $useCase,
        MonthEndAssetBalanceCsvImportResponder $responder,
        userContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                file:
                    $request->file('file'),
            );

        return $responder->created(
            $result,
        );
    }
}
```

Actionでは、
以下を行わない。

- CSVヘッダー検証
- CSV行解析
- `target_year_month`検証
- `asset_account_name`検証
- `balance`検証
- CSV内重複判定
- 資産口座検索
- 残高記録単位判定
- 対象年月時点の有効性判定
- 月末資産状況検索
- 確定状態判定
- 既存月末資産残高検索
- snapshot作成
- 月末資産残高登録
- トランザクション制御
- レスポンス形式への変換

Actionは、
UseCaseの呼び出しと
Responderへの受け渡しに
責務を限定する。

---

#### 5.25.2 Request

Requestでは、
HTTPリクエストとして
CSVファイルを受け付けられる状態かを
検証する。

概念例：

```php
final class ImportMonthEndAssetBalanceCsvRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'file' => [
                'required',
                'file',
                'mimes:csv',
                'max:' . config(
                    'csv.max_file_size_kb',
                ),
            ],
        ];
    }
}
```

実際の

- ファイルサイズ上限
- MIME Type
- 拡張子

の扱いは、
CSV共通仕様に従う。

Requestでは、
CSV内部の業務データまでは
検証しない。

---

#### 5.25.3 Requestで行うこと

Requestでは、
主に以下を検証する。

- `file`が指定されていること
- HTTPアップロードファイルであること
- 許可されたファイル形式であること
- ファイルサイズ上限以内であること

これらは、
CSV解析前に判定可能な
HTTPリクエストレベルの
バリデーションとする。

---

#### 5.25.4 Requestで行わないこと

Requestでは、
以下のCSV内容・業務ルールを
検証しない。

- CSVヘッダー
- CSVデータ行の存在
- `target_year_month`
- `asset_account_name`
- `balance`
- 1ファイル1対象年月
- CSV内重複
- 資産口座存在確認
- 残高記録単位
- 対象年月時点の有効性
- 月末資産状況
- 確定状態
- 既存月末資産残高

これらは、
CSV Validator、
UseCase、
Queryで扱う。

---

#### 5.25.5 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONレスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、
`X-User-Id`を検証し、
操作対象利用者を特定する。

概念的には、
以下とする。

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
Request
    ↓
Action
```

Action以降では、
検証済みの利用者コンテキストを使用する。

---

#### 5.25.6 UseCase

CSV登録の
ユースケース処理を担当する。

主な処理は、
以下とする。

1. 操作対象利用者IDを受け取る
2. CSVファイルを受け取る
3. CSVファイルを解析する
4. CSVヘッダーを検証する
5. 各行の入力値を検証する
6. 対象年月を特定する
7. CSV内重複を検証する
8. 対象資産口座を一括取得する
9. 資産口座に関する業務ルールを検証する
10. 月末資産状況を確認する
11. 既存月末資産残高を確認する
12. CSV全体が登録可能であることを確認する
13. トランザクションを開始する
14. 月末資産状況の状態を再確認する
15. 必要に応じて月末資産状況を作成する
16. 既存月末資産残高を再確認する
17. 月末資産残高を一括登録する
18. 登録結果DTOを生成する
19. 登録結果を返却する

概念的な処理は、
以下とする。

```text
CSV解析
    ↓
CSV構造検証
    ↓
行入力値検証
    ↓
業務データ取得
    ↓
業務ルール検証
    ↓
登録可能
    ↓
DB::transaction
    ↓
snapshot再取得
    ↓
確定状態再確認
    ↓
snapshot必要時作成
    ↓
既存balance再確認
    ↓
balance一括登録
    ↓
Import Result DTO
```

---

#### 5.25.7 CSV-002との共通化

CSV-002とCSV-003では、
可能な限り
同一のCSV解析・検証クラスを使用する。

例えば、
以下を共通化する。

```text
MonthEndAssetBalanceCsvDefinition
MonthEndAssetBalanceCsvParser
MonthEndAssetBalanceCsvValidator
MonthEndAssetBalanceCsvImportValidator
AssetAccountQuery
MonthEndAssetSnapshotQuery
MonthEndAssetBalanceQuery
```

概念的には、

```text
CSV-002
    ↓
共通解析・検証
    ↓
Preview DTO

CSV-003
    ↓
共通解析・検証
    ↓
登録処理
```

とする。

CSV-002とCSV-003で
同じ業務ルールを
別々に実装しない。

---

#### 5.25.8 CSV Definition

CSV-001、
CSV-002、
CSV-003で使用する
月末資産残高CSV仕様は、
共通Definitionへ集約する。

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

CSV-003専用の
ヘッダー定義を持たない。

---

#### 5.25.9 CSV Parser

CSVファイルの読み込みは、
CSV-002と共通の
Parserを使用する。

概念例：

```php
final class MonthEndAssetBalanceCsvParser
{
    public function parse(
        UploadedFile $file,
    ): ParsedCsv {
        // CSV解析
    }
}
```

Parserでは、
主に以下を行う。

- CSVファイルオープン
- BOM処理
- ヘッダー取得
- CSV行読み込み
- 行番号管理
- 完全な空行の除外
- CSV構造異常の検出

Parserでは、
データベース検索や
登録処理を行わない。

---

#### 5.25.10 CSVヘッダー検証

CSVヘッダーは、
共通Definitionと
完全一致することを確認する。

概念例：

```php
if (
    $parsedCsv->headers
    !== MonthEndAssetBalanceCsvDefinition::HEADERS
) {
    throw new InvalidCsvFormatException();
}
```

以下を不正とする。

- ヘッダー不足
- ヘッダー名不一致
- ヘッダー順序不一致
- 余分なヘッダー

ヘッダー不正時は、
業務検証や登録処理へ進まない。

---

#### 5.25.11 CSV Validator

CSV各行の入力値検証は、
CSV-002と共通の
Validatorを使用する。

主に以下を検証する。

```text
target_year_month
    required
    YYYY-MM

asset_account_name
    required

balance
    required
    integer
    min:0
```

CSV Validatorでは、
データベース検索を行わない。

---

#### 5.25.12 0円の扱い

`balance = 0`は、
正常値として扱う。

以下のような
truthy / falsy判定を使用しない。

```php
if (! $balance) {
    // 0円を未入力扱いしてしまう
}
```

必須チェックと
数値チェックを分離する。

---

#### 5.25.13 データ行0件

CSVヘッダーが正常でも、
データ行が1件も存在しない場合は、
登録不可とする。

CSV-002では
`canImport = false`
として扱うが、
CSV-003では

```text
CSV_DATA_REQUIRED
```

相当の業務エラーとして
登録処理を終了する。

snapshotおよび
月末資産残高は作成しない。

---

#### 5.25.14 対象年月の特定

正常な
`target_year_month`を収集し、
CSV全体の対象年月を特定する。

複数年月が存在する場合は、

```text
MULTIPLE_TARGET_YEAR_MONTHS
```

相当のエラーとする。

概念例：

```php
$targetYearMonths =
    collect($validRows)
        ->pluck('targetYearMonth')
        ->unique()
        ->values();

if (
    $targetYearMonths->count() !== 1
) {
    throw new
        MultipleTargetYearMonthsException();
}
```

---

#### 5.25.15 CSV内重複

CSV内で
同一資産口座が
複数回指定されていないことを確認する。

1ファイル1対象年月であるため、
実質的には

```text
assetAccountName
```

単位で一意であることを
確認してよい。

重複が存在する場合は、

```text
DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

相当の登録エラーとする。

---

#### 5.25.16 AssetAccountQuery

CSVで指定された
資産口座名について、
操作対象利用者の資産口座を
一括取得する。

概念例：

```php
$assetAccounts =
    $this->assetAccountQuery
        ->findActiveByNames(
            userId: $userId,
            names: $assetAccountNames,
        );
```

検索条件は、
概念的に以下とする。

```text
user_id
    = 操作対象利用者ID

AND

name IN (...)

AND

deleted_at IS NULL
```

CSV行ごとに
個別SQLを発行しない。

---

#### 5.25.17 資産口座Map

取得した資産口座は、
名前をキーとして
Map化してよい。

概念例：

```php
$assetAccountMap =
    $assetAccounts->keyBy(
        'name',
    );
```

CSV各行について、

```php
$assetAccount =
    $assetAccountMap->get(
        $row->assetAccountName,
    );
```

として参照する。

---

#### 5.25.18 資産口座不存在

操作対象利用者に
対応する資産口座が
存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

相当のエラーとする。

他利用者に
同名資産口座が存在していても、
取得対象としてはならない。

---

#### 5.25.19 残高記録単位判定

対象資産口座の
`balance_recording_unit`が
口座単位であることを確認する。

概念例：

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::ACCOUNT
) {
    throw new
        AssetAccountBalanceRecordingUnitMismatchException();
}
```

商品単位の場合は、
CSV-003の登録対象としない。

---

#### 5.25.20 対象年月時点の有効性判定

対象資産口座が、
CSVの`targetYearMonth`時点で
月末資産管理対象として
有効であることを確認する。

現在日時を基準に
判定しない。

概念的には、

```php
$this->availabilityValidator
    ->validate(
        assetAccount:
            $assetAccount,

        targetYearMonth:
            $targetYearMonth,
    );
```

とする。

対象年月時点で無効の場合は、
登録処理へ進まない。

---

#### 5.25.21 MonthEndAssetSnapshotQuery

対象年月の
月末資産状況を取得する。

概念例：

```php
$snapshot =
    $this->snapshotQuery
        ->findByUserAndTargetYearMonth(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,
        );
```

検索条件には、
必ず操作対象利用者IDを含める。

他利用者の
同一対象年月snapshotを
取得しない。

---

#### 5.25.22 確定状態の事前確認

既存snapshotが存在する場合は、
CSV全体の検証段階で
`confirmed`を確認する。

```php
if (
    $snapshot !== null
    && $snapshot->confirmed
) {
    throw new
        MonthEndAssetSnapshotConfirmedException();
}
```

確定済みの場合は、
登録処理へ進まない。

ただし、
この事前確認だけで
登録時の整合性を保証しない。

トランザクション内でも
最新状態を再確認する。

---

#### 5.25.23 MonthEndAssetBalanceQuery

既存snapshotが存在する場合は、
CSV対象資産口座について
既存月末資産残高を
一括取得する。

概念例：

```php
$existingBalances =
    $this->balanceQuery
        ->findBySnapshotAndAssetAccounts(
            userId:
                $userId,

            snapshotId:
                $snapshot->id,

            assetAccountIds:
                $assetAccountIds,
        );
```

既存残高が存在する場合は、

```text
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

相当のエラーとする。

---

#### 5.25.24 全件検証後に登録する

CSV行を読み込みながら
逐次INSERTしない。

以下のような実装は
行わない。

```php
foreach ($rows as $row) {
    $this->repository
        ->create(
            $row,
        );
}
```

ただし、
このループより前に
全件検証が完了していない場合を指す。

基本的には、

```text
CSV解析
    ↓
全件入力検証
    ↓
全行业務検証
    ↓
登録可能確認
    ↓
トランザクション
    ↓
一括登録
```

とする。

---

#### 5.25.25 Import Input DTO

CSV解析および
業務検証後の登録予定データは、
専用DTOとして保持してよい。

概念例：

```php
final readonly class MonthEndAssetBalanceImportItem
{
    public function __construct(
        public int $assetAccountId,
        public int $balance,
    ) {
    }
}
```

CSV行番号や
資産口座名など、
登録時に不要な情報を
Repositoryへ渡す必要はない。

---

#### 5.25.26 Import Result DTO

CSV登録結果は、
専用DTOとして表現する。

概念例：

```php
final readonly class MonthEndAssetBalanceCsvImportResult
{
    public function __construct(
        public string $targetYearMonth,
        public int $importedCount,
    ) {
    }
}
```

Eloquent Modelや
登録済みCollectionを
そのままResponderへ渡さない。

---

#### 5.25.27 トランザクション

CSV-003の登録処理は、
`DB::transaction()`内で実行する。

概念例：

```php
$result =
    DB::transaction(
        function () use (
            $userId,
            $targetYearMonth,
            $items,
        ): MonthEndAssetBalanceCsvImportResult {
            // snapshot再取得
            // 確定状態再確認
            // snapshot必要時作成
            // 既存balance再確認
            // balance一括登録

            return new
                MonthEndAssetBalanceCsvImportResult(
                    targetYearMonth:
                        $targetYearMonth,

                    importedCount:
                        count($items),
                );
        },
    );
```

CSV解析や
ファイルバリデーションまで
トランザクションへ含めない。

---

#### 5.25.28 トランザクション内での再取得

登録トランザクション開始後、
対象年月のsnapshotを
再度取得する。

概念例：

```php
$snapshot =
    $this->snapshotQuery
        ->findByUserAndTargetYearMonthForUpdate(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,
        );
```

CSV検証段階で取得した
snapshotの状態だけを
信頼して登録しない。

---

#### 5.25.29 snapshotのロック

既存snapshotが存在する場合は、
必要に応じて
`lockForUpdate()`を使用する。

概念例：

```php
return MonthEndAssetSnapshot::query()
    ->where(
        'user_id',
        $userId,
    )
    ->where(
        'target_year_month',
        $targetYearMonth,
    )
    ->lockForUpdate()
    ->first();
```

これにより、
同じsnapshotに対する

- CSV登録
- 月末資産状況確定
- その他の更新処理

との競合を抑制する。

---

#### 5.25.30 snapshot不存在時

トランザクション内で
snapshotが存在しない場合は、
新規作成する。

概念例：

```php
$snapshot =
    $this->snapshotRepository
        ->create(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,
        );
```

新規作成時の
`confirmed`は、

```text
false
```

とする。

---

#### 5.25.31 MonthEndAssetSnapshotRepository

月末資産状況の
新規作成を担当する。

概念例：

```php
final class MonthEndAssetSnapshotRepository
{
    public function create(
        int $userId,
        string $targetYearMonth,
    ): MonthEndAssetSnapshot {
        return MonthEndAssetSnapshot::create([
            'user_id'
                => $userId,

            'target_year_month'
                => $targetYearMonth,

            'confirmed'
                => false,
        ]);
    }
}
```

Repositoryでは、
以下を行わない。

- CSV解析
- CSV入力値検証
- 資産口座検索
- CSV内重複判定
- 月末資産状況確定
- 月末資産残高登録

---

#### 5.25.32 snapshotの重複作成防止

同一利用者、
同一対象年月について
snapshotを複数作成してはならない。

データベースでは、
概念的に以下の
UNIQUE制約を使用する。

```text
UNIQUE (
    user_id,
    target_year_month
)
```

複数リクエストが
同時にsnapshot不存在を確認した場合でも、
DB制約によって
重複作成を防止する。

---

#### 5.25.33 確定状態の最終確認

トランザクション内で
取得したsnapshotについて、
再度`confirmed`を確認する。

```php
if ($snapshot->confirmed) {
    throw new
        MonthEndAssetSnapshotConfirmedException();
}
```

CSV解析時点や
CSV-002実行時点の状態を
登録可否の最終判断として
使用しない。

---

#### 5.25.34 既存月末資産残高の最終確認

登録トランザクション内でも、
既存月末資産残高を
再確認する。

概念例：

```php
$existing =
    $this->balanceQuery
        ->existsAnyBySnapshotAndAssetAccounts(
            snapshotId:
                $snapshot->id,

            assetAccountIds:
                $assetAccountIds,
        );

if ($existing) {
    throw new
        MonthEndAssetBalanceAlreadyExistsException();
}
```

事前検証後に
別リクエストが
登録している可能性を考慮する。

---

#### 5.25.35 MonthEndAssetBalanceRepository

月末資産残高の
登録を担当する。

CSV-003では、
複数件をまとめて登録するため、
一括INSERTを使用してよい。

概念例：

```php
final class MonthEndAssetBalanceRepository
{
    /**
     * @param MonthEndAssetBalanceImportItem[] $items
     */
    public function insertAll(
        int $snapshotId,
        array $items,
    ): void {
        $now =
            now();

        $rows =
            array_map(
                static fn (
                    MonthEndAssetBalanceImportItem $item,
                ): array => [
                    'month_end_asset_snapshot_id'
                        => $snapshotId,

                    'asset_account_id'
                        => $item->assetAccountId,

                    'balance'
                        => $item->balance,

                    'created_at'
                        => $now,

                    'updated_at'
                        => $now,
                ],
                $items,
            );

        MonthEndAssetBalance::query()
            ->insert(
                $rows,
            );
    }
}
```

---

#### 5.25.36 一括INSERT

CSV-003では、
CSV行数分のINSERTを
1件ずつ実行するより、
可能であれば
一括INSERTを使用する。

```text
100行CSV
    ↓
100回INSERT
```

ではなく、

```text
100行CSV
    ↓
1回または少数回のINSERT
```

とする。

ただし、
CSV最大行数や
PostgreSQLのパラメータ上限などを考慮し、
必要であれば
一定件数でchunkしてよい。

---

#### 5.25.37 Repositoryの責務

MonthEndAssetBalanceRepositoryでは、
以下を行わない。

- CSV解析
- CSVヘッダー検証
- 資産口座存在確認
- 利用者境界判定
- 残高記録単位判定
- 確定状態判定
- 重複判定
- トランザクション開始
- HTTPレスポンス生成

Repositoryは、
UseCaseから渡された
登録可能なデータを
永続化することに
責務を限定する。

---

#### 5.25.38 UNIQUE制約

`month_end_asset_balances`では、
以下の組み合わせに
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
asset_account_id
```

アプリケーション側の
重複確認を通過した
複数リクエストが同時実行されても、
DB側で重複登録を防止する。

---

#### 5.25.39 UNIQUE制約違反

月末資産残高登録時に
UNIQUE制約違反が発生した場合は、
PostgreSQLの例外を
そのままレスポンスへ返却しない。

概念的には、

```text
UniqueViolation
    ↓
MonthEndAssetBalanceAlreadyExistsException
    ↓
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

へ変換する。

ただし、
制約名やSQLSTATEだけを
雑に判定するのではなく、
対象制約を識別したうえで
業務エラーへ変換する。

---

#### 5.25.40 snapshot UNIQUE制約違反

snapshot新規作成時に、

```text
user_id
+
target_year_month
```

のUNIQUE制約違反が
発生する可能性がある。

これは、
並行リクエストによって
別処理が先にsnapshotを
作成した可能性がある。

その場合は、
トランザクション設計に従って
最新snapshotを再取得し、
確定状態・既存残高を
再確認する。

単純に
`500 Internal Server Error`
へ変換しない。

---

#### 5.25.41 部分登録を行わない

一括登録途中で
例外が発生した場合は、
トランザクションを
ロールバックする。

```text
snapshot作成
    ↓
balance登録
    ↓
例外
    ↓
ROLLBACK
```

となる。

以下の状態を
残してはならない。

- snapshotだけ存在する
- 一部のbalanceだけ存在する
- CSV前半だけ登録済み

---

#### 5.25.42 confirmedを更新しない

CSV-003では、
snapshotを自動確定しない。

新規作成時は、

```text
confirmed = false
```

とする。

既存snapshotについても、

```php
$snapshot->update([
    'confirmed' => true,
]);
```

などの処理は行わない。

月末資産状況確定は、
SNP-004の責務とする。

---

#### 5.25.43 全資産口座の登録完了を要求しない

CSV-003では、
対象年月時点で有効な
すべての資産口座が
CSVに含まれていることを
登録条件とはしない。

CSVに含まれる
登録可能な資産口座のみを
登録する。

未登録資産が存在するかどうかは、
月末資産状況詳細表示や
確定処理で確認する。

---

#### 5.25.44 API Resource

登録結果DTOを、
専用API Resourceへ変換する。

概念例：

```php
final class MonthEndAssetBalanceCsvImportResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'targetYearMonth'
                => $this->targetYearMonth,

            'importedCount'
                => $this->importedCount,
        ];
    }
}
```

データベース内部IDを
レスポンスへ返却しない。

---

#### 5.25.45 Responder

Responderは、
登録結果DTOを受け取り、
API共通の
成功Envelopeへ変換する。

概念例：

```php
final class MonthEndAssetBalanceCsvImportResponder
{
    public function created(
        MonthEndAssetBalanceCsvImportResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new MonthEndAssetBalanceCsvImportResource(
                        $result,
                    ),
            ],
            Response::HTTP_CREATED,
        );
    }
}
```

正常時は、

```text
201 Created
```

を返却する。

`requestId`などの
共通項目は、
API共通レスポンス処理に従う。

---

#### 5.25.46 Responderの責務

Responderでは、
以下を行わない。

- CSV解析
- CSV入力値検証
- 資産口座検索
- snapshot検索
- 確定状態判定
- 重複判定
- 月末資産残高登録
- `importedCount`算出
- トランザクション制御

Responderは、
生成済み登録結果DTOを
HTTPレスポンスへ
変換することに
責務を限定する。

---

#### 5.25.47 返却しない情報

API Resourceでは、
以下の内部情報を返却しない。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- `month_end_asset_balance_id`
- `confirmed`
- `balance_recording_unit`
- `created_at`
- `updated_at`
- CSV行番号
- 登録した各`balance`
- 登録した各資産口座名

Eloquent Modelを
そのままJSON化しない。

---

#### 5.25.48 例外

業務上の登録不可状態は、
専用例外として表現する。

例えば、
以下を想定する。

```text
InvalidCsvFormatException
CsvDataRequiredException
MultipleTargetYearMonthsException
AssetAccountNotFoundException
AssetAccountBalanceRecordingUnitMismatchException
AssetAccountNotAvailableException
InvalidBalanceException
DuplicateAssetAccountInCsvException
MonthEndAssetSnapshotConfirmedException
MonthEndAssetBalanceAlreadyExistsException
```

これらを
API共通Exception Handlerで
独自エラーコードへ変換する。

---

#### 5.25.49 CSV複数エラー

CSV入力内容に
複数エラーが存在する場合は、
必要に応じて
集約用例外を使用してよい。

概念例：

```php
throw new CsvImportValidationException(
    $errors,
);
```

この例外を、

```text
422 Unprocessable Entity
```

のCSV登録エラーへ変換する。

CSV-002と同様に
複数のエラーを収集できる構成としつつ、
CSV-003では
1件でもエラーがあれば
登録処理へ進まない。

---

#### 5.25.50 想定外例外

想定外の例外は、
API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、
以下を含めない。

- SQL
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- スタックトレース
- PHP内部エラー
- Laravel内部例外メッセージ
- サーバーファイルパス

詳細情報は、
サーバーログへ記録する。

---

#### 5.25.51 N+1問題

CSV行ごとに
資産口座や既存残高を
個別検索しない。

以下のような実装を避ける。

```text
1行目
    ↓
assetAccount検索
balance検索

2行目
    ↓
assetAccount検索
balance検索

3行目
    ↓
...
```

基本的には、

```text
CSV全体解析
    ↓
資産口座名一覧抽出
    ↓
asset_accounts一括取得
    ↓
snapshot 1件取得
    ↓
既存balance一括取得
    ↓
メモリ上で全件検証
```

とする。

---

#### 5.25.52 ロック範囲

ロックは、
CSV解析中には取得しない。

以下の処理が完了してから、
登録トランザクションを開始する。

```text
ファイル解析
CSV構造検証
入力値検証
基本業務検証
```

その後、

```text
DB::transaction
    ↓
snapshotロック
    ↓
最終状態確認
    ↓
登録
```

とする。

ロック保持時間を
必要最小限にする。

---

#### 5.25.53 キャッシュ

CSV-003では、
登録可否判定に
サーバー側キャッシュを使用しない。

以下は、
データベースの最新状態を参照する。

- snapshot
- `confirmed`
- 既存月末資産残高
- 資産口座状態

古いキャッシュを利用して
登録可否を判断しない。

---

#### 5.25.54 ログ

CSV-003では、
必要に応じて
以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
targetYearMonth
rowCount
importedCount
```

`apiId`は、

```text
CSV-003
```

とする。

以下は、
不要にログへ出力しない。

- CSVファイル全文
- 月末残高全件
- 資産口座名全件

登録失敗時には、
業務エラーコードを
ログへ記録してよい。

---

#### 5.25.55 テスト実装方針

Laravel側では、
Feature Testを中心として
CSV-003のAPI契約および
一括登録フロー全体を確認する。

主に以下を確認する。

- `201 Created`
- `400 Bad Request`
- `404 Not Found`
- `409 Conflict`
- `422 Unprocessable Entity`
- `500 Internal Server Error`
- `file`必須
- CSVファイル形式
- CSVヘッダー
- CSVヘッダー順序
- データ行0件
- 1ファイル1対象年月
- `asset_account_name`必須
- 資産口座不存在
- 他利用者の同名資産口座を使用しないこと
- 残高記録単位
- 対象年月時点の資産口座有効性
- `balance`必須
- `balance`整数
- `balance`0以上
- `balance = 0`
- CSV内重複
- snapshot不存在時の新規作成
- snapshot既存時の再利用
- snapshot確定済み
- 既存月末資産残高
- 一部登録されないこと
- トランザクションロールバック
- snapshotだけ残らないこと
- 同時実行時の重複防止
- `confirmed`を変更しないこと
- `importedCount`
- API ResourceによるcamelCase変換
- 返却対象外項目
- 同一CSV再実行時の競合

---

#### 5.25.56 CSV ParserのUnit Test

CSV Parserについて、
以下を確認する。

- 正しいヘッダーを取得できること
- 各データ行を解析できること
- 行番号を保持できること
- BOMを仕様どおり扱えること
- 完全な空行を仕様どおり扱えること
- CSV解析不能時に適切な例外となること

CSV-002と
同一Parserのテストを
共通利用してよい。

---

#### 5.25.57 CSV ValidatorのUnit Test

CSV Validatorについて、
以下を確認する。

```text
target_year_month
    required
    YYYY-MM

asset_account_name
    required

balance
    required
    integer
    min:0
```

特に、

```text
balance = 0
```

を正常値として
確認する。

CSV-002と
同一Validatorを使用する場合は、
同一Unit Testで保証してよい。

---

#### 5.25.58 QueryのDatabase Test

AssetAccountQueryでは、
以下を確認する。

```text
user_id一致
+
name一致
+
deleted_at IS NULL
    ↓
取得できる
```

他利用者の
同名資産口座は
取得できないことを確認する。

MonthEndAssetSnapshotQueryでは、

```text
user_id
+
target_year_month
```

で
対象snapshotを取得できることを確認する。

MonthEndAssetBalanceQueryでは、

```text
snapshotId
+
assetAccountIds
```

によって
既存残高を取得できることを確認する。

---

#### 5.25.59 RepositoryのDatabase Test

MonthEndAssetSnapshotRepositoryでは、

```text
user_id
target_year_month
confirmed = false
```

で
snapshotを作成できることを確認する。

MonthEndAssetBalanceRepositoryでは、
複数のImport Itemから
正しい月末資産残高を
一括登録できることを確認する。

特に、

- `month_end_asset_snapshot_id`
- `asset_account_id`
- `balance`

が
正しく保存されることを確認する。

---

#### 5.25.60 UseCaseのUnit Test

UseCaseについては、
Parser、
Validator、
Query、
Repositoryを組み合わせた
ユースケース制御を確認する。

概念的には、

```text
CSV
    ↓
Parser
    ↓
Validator
    ↓
Query
    ↓
ImportMonthEndAssetBalanceCsvUseCase
    ↓
Repository
    ↓
Import Result
```

を確認する。

主に以下をテストする。

```text
正常CSV
    ↓
登録成功
```

```text
CSV入力エラー
    ↓
Repositoryを呼ばない
```

```text
snapshot確定済み
    ↓
登録しない
```

```text
既存balanceあり
    ↓
登録しない
```

```text
snapshotなし
    ↓
snapshot作成
    ↓
balance登録
```

```text
snapshotあり
    ↓
既存snapshot使用
    ↓
balance登録
```

---

#### 5.25.61 トランザクションのFeature Test

登録途中に
意図的な例外を発生させ、
トランザクションが
ロールバックされることを確認する。

例えば、

```text
snapshot作成
    ↓
balance登録
    ↓
例外
```

の場合に、

```text
snapshot
    → 残らない

balance
    → 残らない
```

ことを確認する。

既存snapshotを使用する場合は、

```text
既存snapshot
    → 残る

新規balance
    → 全件ロールバック
```

となることを確認する。

---

#### 5.25.62 UNIQUE制約のDatabase Test

以下のUNIQUE制約が
機能することを確認する。

```text
month_end_asset_snapshots

user_id
+
target_year_month
```

および、

```text
month_end_asset_balances

month_end_asset_snapshot_id
+
asset_account_id
```

同一キーで
重複登録できないことを確認する。

---

#### 5.25.63 CSV-002との整合性テスト

同一DB状態、
同一CSVについて、

```text
CSV-002
canImport = true
```

となる場合に、
CSV-003の事前検証も
登録可能となることを確認する。

CSV-002とCSV-003の
検証ロジックの差異によって、

```text
CSV-002
正常

CSV-003
同一状態なのに入力エラー
```

となることを防止する。

ただし、
CSV-002後に
DB状態が変更された場合は、
CSV-003で
登録不可となってよい。

---

#### 5.25.64 同時実行テスト

必要に応じて、
同一利用者、
同一対象年月、
同一資産口座に対する
並行登録をテストする。

最終的に、

```text
同一snapshot
+
同一assetAccount
    ↓
月末資産残高は1件のみ
```

となることを確認する。

競合したリクエストでは、
PostgreSQLの生例外ではなく、

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

へ変換されることを確認する。

---

### 5.26 React・TypeScriptでの利用

CSV-003は、
CSV-002 月末資産残高CSVプレビューで
登録可能と確認したCSVファイルを
実際に登録するために使用する。

フロントエンドでは、
CSV-002で使用した
同一の`File`を保持し、
利用者が登録操作を行ったタイミングで
CSV-003へ再送する。

概念的な利用フローは、
以下とする。

```text
CSVファイル選択
    ↓
CSV-002
プレビュー
    ↓
canImport確認
    ├─ false
    │     ↓
    │   エラー表示
    │
    └─ true
          ↓
       登録ボタン有効化
          ↓
       利用者が登録実行
          ↓
       CSV-003
          ↓
       成功
          ↓
       関連Query再取得
```

CSV-002の
`canImport = true`は、
CSV-003の登録成功を
保証するものではない。

CSV-003では、
実行時点の最新状態をもとに
再検証される。

---

#### 5.26.1 TypeScript型

CSV登録結果は、
以下のような型として扱う。

概念例：

```ts
export type MonthEndAssetBalanceCsvImportResult = {
  targetYearMonth: string;
  importedCount: number;
};

export type ImportMonthEndAssetBalanceCsvResponse = {
  data: MonthEndAssetBalanceCsvImportResult;
  requestId: string;
};
```

API共通Envelopeの
共通型が存在する場合は、
以下のように
共通型を使用する。

```ts
export type ImportMonthEndAssetBalanceCsvResponse =
  ApiResponse<MonthEndAssetBalanceCsvImportResult>;
```

---

#### 5.26.2 Fileの保持

CSV-002で
プレビューしたCSVファイルは、
CSV-003実行まで
`File`として保持する。

概念例：

```ts
const [file, setFile] =
  useState<File | null>(
    null,
  );
```

CSV-002の
プレビュー結果から
新しいCSVファイルを
生成し直さない。

---

#### 5.26.3 同じFileを使用する

CSV-003では、
CSV-002で使用した
同じ`File`を送信する。

```text
CSV-002
File A
    ↓
canImport = true
    ↓
CSV-003
File A
```

以下のように、
プレビュー結果の

- `rows`
- `targetYearMonth`
- `balance`

などから
CSVを再構築しない。

正式な登録対象は、
利用者が選択した
CSVファイルそのものとする。

---

#### 5.26.4 FormData

CSVファイルは、
`FormData`へ設定する。

概念例：

```ts
const formData =
  new FormData();

formData.append(
  'file',
  file,
);
```

項目名は、

```text
file
```

とする。

以下の業務項目を
追加送信しない。

- `userId`
- `targetYearMonth`
- `assetAccountId`
- `balance`
- `confirmed`
- `previewId`
- `canImport`

---

#### 5.26.5 Content-Type

CSV-002と同様に、
`multipart/form-data`の
`Content-Type`は、
ブラウザまたはHTTP Clientへ
設定させる。

以下のような
手動設定は行わない。

```ts
headers: {
  'Content-Type':
    'multipart/form-data',
}
```

`FormData`を送信し、
boundaryを
HTTP Client側で生成させる。

---

#### 5.26.6 X-User-Id

`X-User-Id`は、
通常のAPIと同様に
共通API Clientから付与する。

CSV-003専用処理で
`userId`を
`FormData`へ追加しない。

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

CSV-002実行時の
利用者コンテキストを
サーバー側で引き継ぐ前提にしない。

---

#### 5.26.7 API Client

CSV登録APIは、
CSVファイルを受け取る
専用関数として定義する。

概念例：

```ts
export const importMonthEndAssetBalanceCsv =
  async (
    file: File,
  ): Promise<MonthEndAssetBalanceCsvImportResult> => {
    const formData =
      new FormData();

    formData.append(
      'file',
      file,
    );

    const response =
      await apiClient.post<
        ImportMonthEndAssetBalanceCsvResponse
      >(
        '/api/v1/month-end-asset-balances/imports',
        formData,
      );

    return response.data.data;
  };
```

API Clientでは、
CSV内容の解析や
登録可否判定を行わない。

---

#### 5.26.8 Mutationとして扱う

CSV-003は、
業務データを更新するため、
TanStack Queryを使用する場合は
Mutationとして扱う。

概念例：

```ts
export const useImportMonthEndAssetBalanceCsv =
  () =>
    useMutation({
      mutationFn:
        importMonthEndAssetBalanceCsv,
    });
```

Queryとして
自動実行しない。

---

#### 5.26.9 登録可能状態

登録ボタンは、
少なくとも以下を
すべて満たす場合のみ
有効化する。

```text
file != null
AND
preview != null
AND
preview.canImport = true
AND
CSV-003実行中ではない
```

概念例：

```ts
const canSubmit =
  file !== null
  && preview !== null
  && preview.canImport
  && !importMutation.isPending;
```

---

#### 5.26.10 登録ボタン

概念例：

```tsx
<button
  type="button"
  disabled={!canSubmit}
  onClick={handleImport}
>
  {importMutation.isPending
    ? '登録中...'
    : '登録する'}
</button>
```

登録実行中は、
二重送信防止のため
ボタンを無効化する。

ただし、
フロントエンドの
ボタン制御だけを
重複登録防止の保証としない。

---

#### 5.26.11 登録処理

概念例：

```ts
const handleImport =
  (): void => {
    if (
      file === null
      || preview === null
      || !preview.canImport
    ) {
      return;
    }

    importMutation.mutate(
      file,
    );
  };
```

CSV-002の
`preview.rows`を
CSV-003へ送信しない。

---

#### 5.26.12 登録確認

CSV-003は、
複数件の
月末資産残高を
一括登録する。

誤操作防止のため、
画面設計に応じて
登録前に確認ダイアログを
表示してよい。

例えば、

```text
2026年7月の月末資産残高を
2件登録します。

登録しますか？
```

のように、

- 対象年月
- 登録予定件数

を表示してよい。

登録予定件数は、
`preview.rows.length`などを
表示用途として使用できる。

ただし、
登録可否自体は
`preview.canImport`を使用する。

---

#### 5.26.13 CSV-002結果を登録条件としてのみ利用する

フロントエンドでは、
CSV-002の

```text
canImport = true
```

を
登録ボタンの有効化に使用する。

ただし、
CSV-003のバックエンドでは
CSV-002の結果を
信頼しない。

```text
Frontend
    ↓
canImport = true
    ↓
登録ボタン有効

Backend
    ↓
CSV-003
    ↓
再解析・再検証
```

とする。

---

#### 5.26.14 登録成功

CSV-003が成功した場合は、

```text
201 Created
```

とともに、

- `targetYearMonth`
- `importedCount`

が返却される。

概念例：

```json
{
  "targetYearMonth": "2026-07",
  "importedCount": 2
}
```

画面では、
必要に応じて

```text
2026年7月の月末資産残高を
2件登録しました。
```

のように表示してよい。

---

#### 5.26.15 importedCount

`importedCount`は、
実際に新規登録された
月末資産残高件数である。

TypeScriptでは、

```ts
importedCount: number;
```

として扱う。

正常レスポンスでは、
1以上となる。

フロントエンドで
CSV行数から
登録件数を再計算する必要はない。

---

#### 5.26.16 targetYearMonth

`targetYearMonth`は、
正常登録された
対象年月を表す。

```ts
targetYearMonth: string;
```

として扱う。

CSV-003成功後の
完了メッセージや
遷移先の対象年月指定に
利用してよい。

---

#### 5.26.17 登録成功後の状態クリア

CSV-003が成功した場合は、
保持している

- `file`
- CSV-002プレビュー結果

をクリアする。

概念例：

```ts
setFile(
  null,
);

setPreview(
  null,
);
```

登録済みCSVに対する
古いプレビュー状態を
画面へ残さない。

---

#### 5.26.18 file inputのリセット

React Stateで
`file = null`にしても、
ブラウザの
`<input type="file">`の表示が
自動的にクリアされない場合がある。

必要に応じて
`ref`または
`key`を使用して
ファイル入力をリセットする。

具体的な実装は、
画面実装時に決定する。

---

#### 5.26.19 Query Cacheの無効化

CSV-003成功後は、
月末資産残高や
月末資産状況に関連する
Query Cacheを無効化する。

例えば、
TanStack Queryでは
概念的に以下を行う。

```ts
await queryClient.invalidateQueries({
  queryKey: [
    'monthEndAssetBalances',
    result.targetYearMonth,
  ],
});
```

必要に応じて、
月末資産状況も
再取得対象とする。

```ts
await queryClient.invalidateQueries({
  queryKey: [
    'monthEndAssetSnapshots',
    result.targetYearMonth,
  ],
});
```

正式なQuery Keyは、
フロントエンド共通設計に従う。

---

#### 5.26.20 資産状況系Queryの扱い

CSV-003では、
登録した月末資産状況は
未確定のままである。

AST-001、
AST-002、
AST-003は
確定済み資産状況を
参照するAPIであるため、
CSV-003成功直後に
必ずしも表示結果が
変化するとは限らない。

そのため、
資産状況系Query Cacheを
無条件にすべて再取得する必要はない。

確定処理成功後に
資産状況系Queryを
invalidateする設計としてよい。

---

#### 5.26.21 登録成功後の画面遷移

CSV-003成功後は、
画面設計に応じて

- CSVインポート画面に留まる
- 月末資産状況詳細画面へ移動する
- 月末資産残高一覧画面へ移動する

などを選択できる。

対象年月は、

```ts
result.targetYearMonth
```

を使用する。

API内部の

```text
snapshotId
```

を
遷移のために必要としない。

---

#### 5.26.22 409 Conflict

CSV-003では、
CSV-002成功後でも
`409 Conflict`が返却される可能性がある。

代表例は、
以下とする。

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

これは、
CSV-002後に
業務状態が変更された場合などに
発生し得る。

---

#### 5.26.23 確定済みエラー

`MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED`
が返却された場合は、
対象年月が
既に確定済みとなったことを
利用者へ表示する。

例えば、

```text
対象年月はすでに確定されているため、
CSVを登録できません。
```

のように表示する。

古いプレビュー結果の

```text
canImport = true
```

を保持し続けない。

必要に応じて
プレビュー結果を無効化する。

---

#### 5.26.24 既存月末資産残高エラー

`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`
が返却された場合は、
対象年月・資産口座に
既存データが存在することを
利用者へ表示する。

既存データを
CSV値で上書きするための
再送操作を提供しない。

必要な場合は、
月末資産残高更新画面へ
誘導する。

---

#### 5.26.25 422 Unprocessable Entity

CSV-003では、
CSV内容に
登録できない入力・業務エラーがある場合、

```text
422 Unprocessable Entity
```

となる。

例えば、

- CSV形式不正
- 対象年月不正
- 資産口座不存在
- 残高記録単位不一致
- 月末残高不正
- CSV内重複

などを対象とする。

CSV-002のように

```text
200 OK
canImport = false
```

とはならない。

---

#### 5.26.26 CSV複数エラー

CSV-003で
複数のCSV行エラーが
`error.details`として返却される場合は、
各行のエラーを
一覧表示してよい。

概念的な型は、
API共通エラー型に
従う。

例えば、

```ts
export type CsvImportErrorDetail = {
  rowNumber?: number;
  field?: string;
  code?: string;
  message: string;
};
```

正式な型は、
共通エラー仕様に合わせる。

---

#### 5.26.27 CSV-003失敗後のプレビュー

CSV-003が
業務状態変化によって失敗した場合は、
必要に応じて
CSV-002を再実行できるようにする。

概念的には、

```text
CSV-002
canImport = true
    ↓
CSV-003
409 Conflict
    ↓
プレビュー無効化
    ↓
CSV-002
再実行
```

とする。

ただし、
確定済みなど
再プレビューしても
登録できないことが明確な場合は、
画面設計に応じて
適切な案内を表示する。

---

#### 5.26.28 VALIDATION_ERROR

`VALIDATION_ERROR`の場合は、
アップロードファイル自体の
入力不正として扱う。

例えば、

- `file`未指定
- ファイル形式不正
- ファイルサイズ超過

などである。

CSV登録処理が
開始されていないことを前提として
エラー表示する。

---

#### 5.26.29 INVALID_CSV_FORMAT

`INVALID_CSV_FORMAT`の場合は、
CSV自体を
正常に解析できないことを表示する。

利用者へ
CSV-001のテンプレートを使用して
再作成するよう案内してよい。

---

#### 5.26.30 利用者関連エラー

以下のエラーは、
API共通方針に従って処理する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
```

CSV-003画面だけで
独自の利用者エラー処理を
作成しない。

---

#### 5.26.31 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`の場合は、
共通サーバーエラーとして扱う。

登録成功とみなして
画面状態をクリアしてはならない。

ただし、
通信状態によっては
サーバー側で登録済みかどうか
クライアントから判断できない場合がある。

必要に応じて、
参照APIを再取得して
現在状態を確認する。

---

#### 5.26.32 通信エラー

CSV-003実行後に
ネットワークエラーとなった場合、
サーバー側では
登録が完了している可能性がある。

そのため、

```text
通信エラー
    ↓
同じCSVを無条件再送
```

だけを
唯一の復旧手段としない。

必要に応じて、
BAL-001などから
現在の登録状態を
再取得できるようにする。

---

#### 5.26.33 同一CSVの再送

CSV-003成功後に
同じCSVを再度送信すると、

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

となる。

フロントエンドでは、
登録成功後に

- `file`
- `preview`

をクリアすることで、
通常操作における
意図しない再送を防止する。

---

#### 5.26.34 二重送信防止

Mutation実行中は、
登録ボタンを非活性化する。

概念例：

```tsx
disabled={
  !canSubmit
  || importMutation.isPending
}
```

ただし、
バックエンドでも

- 重複確認
- トランザクション
- UNIQUE制約

によって
重複登録を防止する。

---

#### 5.26.35 Idempotency-Key

Phase1では、
CSV-003に

```text
Idempotency-Key
```

を付与しない。

フロントエンドでも、
独自の
Idempotency Key生成処理を
実装しない。

再送時は、
バックエンドの
重複チェックによって
競合として扱われる。

---

#### 5.26.36 登録済みかをフロントエンドだけで判断しない

CSV-003実行前に、
React側の画面Stateだけを見て

```text
この対象年月は未登録
```

と判断しない。

CSV-003では、
バックエンドが
最新のデータベース状態を
再確認する。

フロントエンドのStateは、
業務整合性の保証には使用しない。

---

#### 5.26.37 CSVをReact側で再解析しない

CSV-003のために
React側でCSVを正式解析しない。

以下のロジックを
バックエンドと二重実装しない。

- ヘッダー検証
- `target_year_month`検証
- 資産口座存在確認
- 残高記録単位判定
- `balance`検証
- CSV内重複判定
- 既存残高判定

正式な登録可否は、
バックエンドへ集約する。

---

#### 5.26.38 成功後に登録済みデータをレスポンスから構築しない

CSV-003のレスポンスには、
登録した各資産口座・残高は
含まれない。

そのため、
登録後の月末資産残高一覧を
CSV-003レスポンスから
React側で再構築しない。

必要な場合は、
BAL-001を再取得する。

---

### 5.27 設計上の補足

#### 5.27.1 POSTを採用する理由

CSV-003は、
CSVファイルの内容に基づき、
複数の月末資産残高を
新規登録する。

そのため、
HTTPメソッドには
`POST`を採用する。

---

#### 5.27.2 CSV-002とCSV-003を分離する理由

CSVプレビューと
CSV登録は、
副作用の有無が異なる。

```text
CSV-002
    ↓
解析・検証
    ↓
DB更新なし
```

```text
CSV-003
    ↓
解析・再検証
    ↓
DB更新あり
```

同一APIへ

```text
preview = true
```

などを指定して
動作を切り替える方式は
採用しない。

---

#### 5.27.3 CSV-003で再検証する理由

CSV-002とCSV-003の間で、
業務状態が変化する可能性がある。

例えば、

```text
CSV-002
snapshot未確定
    ↓
canImport = true

別処理
snapshot確定
    ↓

CSV-003
```

となり得る。

そのため、
CSV-003では
CSV内容だけでなく
業務状態も再検証する。

---

#### 5.27.4 プレビュー結果を送信しない理由

CSV-003では、

```text
canImport
errors
rows
previewId
```

などを
登録リクエストとして
受け取らない。

クライアント側で
プレビュー結果が
改変される可能性があるためである。

CSV-003は、

```text
CSVファイル
+
X-User-Id
+
最新DB状態
```

だけを基準として
登録可否を判定する。

---

#### 5.27.5 Fileを再送する理由

Phase1では、
CSV-002のプレビュー内容を
サーバー側へ保存しない。

そのため、
CSV-003では
CSV-002で使用したファイルを
再送する。

これにより、

- 一時ファイル保存
- preview token
- preview session
- 有効期限管理

などを不要とする。

---

#### 5.27.6 全件成功・全件失敗とする理由

CSVは、
複数件をまとめて
登録するための入力手段である。

一部だけ登録すると、
利用者が

```text
どこまで登録されたか
```

を把握しにくくなる。

そのため、
CSV-003では

```text
全件成功
または
全件失敗
```

とする。

---

#### 5.27.7 トランザクションを使用する理由

CSV-003では、

- 必要に応じたsnapshot作成
- 複数の月末資産残高登録

を行う。

途中でエラーが発生した場合に
一部だけデータを残さないため、
同一トランザクションで処理する。

---

#### 5.27.8 snapshotを必要時に作成する理由

月末資産残高を
登録するためには、
対象年月の
月末資産状況が必要となる。

利用者が
CSV登録前に
snapshotを明示的に作成する操作を
必須とすると、
操作手順が増える。

そのため、
対象年月のsnapshotが
存在しない場合は、
CSV-003内で
未確定状態として作成する。

---

#### 5.27.9 snapshotを自動確定しない理由

CSV-003は、
口座単位の月末資産残高だけを
登録するAPIである。

対象年月には、

- 他の口座単位残高
- 商品別月末評価額

などが
まだ未登録である可能性がある。

そのため、
CSV登録成功だけを理由として
月末資産状況を
自動確定しない。

---

#### 5.27.10 既存残高を上書きしない理由

CSV-003へ
登録と更新の両方の責務を持たせると、

```text
新規登録
上書き
一部更新
```

の挙動が複雑になる。

Phase1では、
CSV-003を
新規一括登録に限定する。

既存残高の変更は、
専用更新APIへ委譲する。

---

#### 5.27.11 importedCountだけを返却する理由

CSV-003実行前には、
CSV-002で
登録内容を確認できる。

そのため、
CSV-003の成功レスポンスで
登録した全レコードを
再度返却する必要はない。

正常登録の確認に必要な

```text
targetYearMonth
importedCount
```

だけを返却する。

---

#### 5.27.12 snapshotIdを返却しない理由

`month_end_asset_snapshot_id`は、
バックエンド内部の関連IDである。

フロントエンドでは、
対象年月を使用して
月末資産関連画面へ
アクセスできる設計とする。

そのため、
CSV登録成功レスポンスへ
`snapshotId`を返却しない。

---

#### 5.27.13 409 Conflictを使用する理由

確定済みsnapshotや
既存月末資産残高は、
CSV内容そのものが
解析不能なわけではない。

CSV-003実行時点の
サーバー側状態と競合して
登録できない状態である。

そのため、

```text
409 Conflict
```

として扱う。

---

#### 5.27.14 CSV入力不正を422とする理由

CSV構造、
CSV入力値、
CSV内の業務入力が
登録条件を満たさない場合は、

```text
422 Unprocessable Entity
```

を使用する。

入力自体はHTTPとして
受信できているが、
業務処理可能な内容ではないためである。

---

#### 5.27.15 UNIQUE制約を使用する理由

アプリケーション側で

```text
既存データなし
```

を確認しても、
確認直後に
別リクエストが登録する可能性がある。

そのため、

```text
month_end_asset_snapshot_id
+
asset_account_id
```

のUNIQUE制約を
最終防衛線として使用する。

フロントエンドの
二重送信防止だけでは
整合性を保証しない。

---

#### 5.27.16 Idempotency-Keyを採用しない理由

Phase1では、
CSV登録の重複防止を

- アプリケーション側の重複確認
- トランザクション
- UNIQUE制約
- フロントエンドの二重送信防止

によって実現する。

`Idempotency-Key`を導入すると、

- Key保存
- 有効期限
- レスポンス再現
- Keyと利用者の関連管理

などの追加設計が必要になるため、
Phase1では採用しない。

---

#### 5.27.17 通信エラー後に成功判定できない理由

Idempotency-Keyを
採用していないため、
CSV-003送信後に
通信が切断された場合、

```text
登録前に失敗した
```

のか、

```text
登録成功後に
レスポンスだけ受信できなかった
```

のかを
クライアント側で
完全には判定できない。

必要に応じて、
参照APIから
現在状態を確認する。

---

#### 5.27.18 React Query Cacheを再取得する理由

CSV-003によって
月末資産残高が
新規登録されるため、
画面上に保持している
古い月末資産残高一覧は
最新状態ではなくなる。

そのため、
登録成功後は
関連Queryをinvalidateし、
サーバーから最新状態を取得する。

---

#### 5.27.19 資産推移系を即時更新しなくてよい理由

CSV-003成功後も、
対象snapshotは
未確定である。

AST系APIでは
確定済みsnapshotを使用するため、
CSV登録直後のデータは
まだ資産状況・資産推移へ
反映されない。

そのため、
資産状況系Queryの
本格的な再取得は、
snapshot確定成功後に行う設計でよい。

---

#### 5.27.20 React側で登録可否を再計算しない理由

登録可否に必要な情報には、

- 最新のsnapshot状態
- 既存月末資産残高
- 資産口座状態

など、
フロントエンドだけでは
正確に保証できない情報が含まれる。

そのため、
React側では
CSV-002の`canImport`を
画面制御に使用するだけとし、
最終登録可否は
CSV-003へ委ねる。

---

### 5.28 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [CSVインポートAPI詳細](./csv-imports.md)
- [月末資産残高API詳細](./month-end-asset-balances.md)
- [月末資産状況API詳細](./month-end-asset-snapshots.md)
- [資産状況・資産推移API詳細](./asset-views.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)

---

## 6. CSV-004 商品別月末評価額CSVテンプレート取得

### 6.1 概要

商品単位で管理する
保有商品の月末評価額を
CSVインポートするための
CSVテンプレートを取得する。

本APIでは、
以下の項目を持つ
商品単位用CSVテンプレートを返却する。

```text
対象年月
資産口座名
保有商品名
月末評価額
```

CSVテンプレートは、CSV-005 商品別月末評価額CSVプレビュー、CSV-006 商品別月末評価額CSV登録で使用する入力形式とする。

Phase1では、商品単位用CSVの基本的なフォーマットを以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

本APIでは、CSVテンプレートの取得のみを行う。

以下の処理は行わない。

- 商品別月末評価額の登録
- 商品別月末評価額の更新
- 商品別月末評価額の削除
- CSVプレビュー
- CSV入力値検証
- 対象年月の判定
- 重複登録判定
- 月末資産状況の作成
- 月末資産状況の確定

商品単位用のCSVテンプレートを提供することは、機能要件の「CSVテンプレート」および「CSVフォーマット」に対応する。

---

### 6.2 ユースケース

利用者が、商品単位で管理している複数の保有商品について、月末評価額をCSVで一括登録したい場合に使用する。

基本的な利用フローは、以下とする。

```text
CSV-004
商品別月末評価額CSVテンプレート取得
    ↓
利用者がCSVへ入力
    ↓
CSV-005
商品別月末評価額CSVプレビュー
    ↓
内容確認
    ↓
CSV-006
商品別月末評価額CSV登録
```

利用者は、取得したテンプレートへ以下を入力する。

- 対象年月
- 資産口座名
- 保有商品名
- 月末評価額

1つのCSVファイルでは、1つの対象年月のみを登録対象とする。

---

### 6.3 エンドポイント

```http
GET /api/v1/month-end-holding-values/csv-template
```

API一覧に定義された商品別月末評価額CSVテンプレート取得用のエンドポイントを使用する。

---

### 6.4 HTTPメソッド

```text
GET
```

本APIは、商品別月末評価額CSVテンプレートを取得する参照APIであるため、`GET`を使用する。

本APIの実行によって、業務データは変更しない。

以下のテーブルへの登録・更新・削除は行わない。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

CSVファイルのアップロードも行わない。

---

### 6.5 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、他の利用者依存APIと同様に、

```text
X-User-Id
```

リクエストヘッダーで指定する。

例：

```http
X-User-Id: 1
```

CSVインポートでは、操作対象利用者に属する資産情報のみを検証および登録対象とする。

商品単位CSVでは、主に以下のリソースが利用者境界の対象となる。

- 資産口座
- 保有商品
- 月末資産状況
- 商品別月末評価額

他の利用者に属する資産情報をCSVインポートによって参照または登録対象としてはならない。

ただし、CSV-004ではテンプレート取得のみを行うため、商品別月末評価額の登録対象となる業務データを検索・更新しない。

---

#### 6.5.1 X-User-Id

操作対象利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者IDは、以下では受け付けない。

- パスパラメータ
- クエリパラメータ
- リクエストボディ

本APIのエンドポイントは、利用者IDをURLへ含めない。

```http
GET /api/v1/month-end-holding-values/csv-template
```

とする。

---

#### 6.5.2 利用者存在確認

`X-User-Id`で指定された利用者が有効な利用者であることを確認する。

概念的には、以下の条件で確認する。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者が存在しない、または論理削除済みの場合は、テンプレートを返却しない。

---

#### 6.5.3 利用者固有データをテンプレートへ出力しない

CSV-004では、操作対象利用者に属する以下のデータをCSVテンプレートへ自動出力しない。

- 資産口座
- 保有商品

例えば、以下のような利用者固有のデータ行は生成しない。

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,投資信託A,
,証券口座,投資信託B,
```

テンプレートは、CSV入力形式を示すための固定構造として扱う。

---

#### 6.5.4 他利用者データを参照しない

CSV-004では、テンプレート生成のために他利用者の以下のデータを参照しない。

- 資産口座
- 保有商品
- 商品別月末評価額

操作対象利用者以外の業務データがCSVテンプレートへ含まれることはない。

---

#### 6.5.5 資産口座・保有商品の存在確認を行わない

CSV-004では、テンプレート取得時に資産口座や保有商品が存在することを必須としない。

例えば、操作対象利用者に商品単位の資産口座や保有商品がまだ登録されていない場合でも、CSVテンプレート自体は取得可能とする。

資産口座および保有商品の存在確認は、CSV-005およびCSV-006でCSV内容を検証する際に行う。

---

#### 6.5.6 残高記録単位による取得制限を行わない

CSV-004では、操作対象利用者に

```text
balance_recording_unit
    = 商品単位
```

の資産口座が存在するかどうかによって、テンプレート取得可否を変更しない。

本APIは、商品単位用CSVの入力形式を提供することに責務を限定する。

残高記録単位とCSV種別の整合性は、CSVインポート時の入力チェックで確認する。

---

#### 6.5.7 保有商品をテンプレートへ事前展開しない

商品単位CSVでは、保有商品名を入力項目として使用する。

ただし、CSV-004で

```text
holding_assets
```

を検索し、登録済み保有商品をテンプレートへ事前展開する方式は採用しない。

テンプレートは、以下のヘッダーを持つ固定形式とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

保有商品の存在確認や対象年月時点での有効性確認は、CSV-005およびCSV-006で行う。

---

#### 6.5.8 X-User-Id未指定

`X-User-Id`が指定されていない場合は、API共通方針に従って

```text
USER_CONTEXT_REQUIRED
```

として扱う。

テンプレート生成処理へ進まない。

---

#### 6.5.9 X-User-Id形式不正

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

として扱う。

例えば、

```text
abc
-1
0
```

などを有効な利用者IDとして扱わない。

---

#### 6.5.10 利用者不存在

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

として扱う。

利用者が存在しない状態でCSVテンプレートを返却しない。

---

#### 6.5.11 将来の認証導入

将来的に認証機能を導入した場合は、`X-User-Id`による利用者指定を廃止し、

```text
認証済み利用者
```

から操作対象利用者を特定する方式へ変更する。

ただし、CSVテンプレートの以下には影響させない。

- ヘッダー
- CSV形式
- CSV-005・CSV-006との関係

CSVテンプレートは、CSV-005 商品別月末評価額CSVプレビュー、CSV-006 商品別月末評価額CSV登録で使用する入力形式とする。

Phase1では、商品単位用CSVの基本的なフォーマットを以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

本APIでは、CSVテンプレートの取得のみを行う。

以下の処理は行わない。

- 商品別月末評価額の登録
- 商品別月末評価額の更新
- 商品別月末評価額の削除
- CSVプレビュー
- CSV入力値検証
- 対象年月の判定
- 重複登録判定
- 月末資産状況の作成
- 月末資産状況の確定

商品単位用のCSVテンプレートを提供することは、機能要件の「CSVテンプレート」および「CSVフォーマット」に対応する。

---

### 6.2 ユースケース

利用者が、商品単位で管理している複数の保有商品について、月末評価額をCSVで一括登録したい場合に使用する。

基本的な利用フローは、以下とする。

```text
CSV-004
商品別月末評価額CSVテンプレート取得
    ↓
利用者がCSVへ入力
    ↓
CSV-005
商品別月末評価額CSVプレビュー
    ↓
内容確認
    ↓
CSV-006
商品別月末評価額CSV登録
```

利用者は、取得したテンプレートへ以下を入力する。

- 対象年月
- 資産口座名
- 保有商品名
- 月末評価額

1つのCSVファイルでは、1つの対象年月のみを登録対象とする。

---

### 6.3 エンドポイント

```http
GET /api/v1/month-end-holding-values/csv-template
```

API一覧に定義された商品別月末評価額CSVテンプレート取得用のエンドポイントを使用する。

---

### 6.4 HTTPメソッド

```text
GET
```

本APIは、商品別月末評価額CSVテンプレートを取得する参照APIであるため、`GET`を使用する。

本APIの実行によって、業務データは変更しない。

以下のテーブルへの登録・更新・削除は行わない。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

CSVファイルのアップロードも行わない。

---

### 6.5 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、他の利用者依存APIと同様に、

```text
X-User-Id
```

リクエストヘッダーで指定する。

例：

```http
X-User-Id: 1
```

CSVインポートでは、操作対象利用者に属する資産情報のみを検証および登録対象とする。

商品単位CSVでは、主に以下のリソースが利用者境界の対象となる。

- 資産口座
- 保有商品
- 月末資産状況
- 商品別月末評価額

他の利用者に属する資産情報をCSVインポートによって参照または登録対象としてはならない。

ただし、CSV-004ではテンプレート取得のみを行うため、商品別月末評価額の登録対象となる業務データを検索・更新しない。

---

#### 6.5.1 X-User-Id

操作対象利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者IDは、以下では受け付けない。

- パスパラメータ
- クエリパラメータ
- リクエストボディ

本APIのエンドポイントは、利用者IDをURLへ含めない。

```http
GET /api/v1/month-end-holding-values/csv-template
```

とする。

---

#### 6.5.2 利用者存在確認

`X-User-Id`で指定された利用者が有効な利用者であることを確認する。

概念的には、以下の条件で確認する。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者が存在しない、または論理削除済みの場合は、テンプレートを返却しない。

---

#### 6.5.3 利用者固有データをテンプレートへ出力しない

CSV-004では、操作対象利用者に属する以下のデータをCSVテンプレートへ自動出力しない。

- 資産口座
- 保有商品

例えば、以下のような利用者固有のデータ行は生成しない。

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,投資信託A,
,証券口座,投資信託B,
```

テンプレートは、CSV入力形式を示すための固定構造として扱う。

---

#### 6.5.4 他利用者データを参照しない

CSV-004では、テンプレート生成のために他利用者の以下のデータを参照しない。

- 資産口座
- 保有商品
- 商品別月末評価額

操作対象利用者以外の業務データがCSVテンプレートへ含まれることはない。

---

#### 6.5.5 資産口座・保有商品の存在確認を行わない

CSV-004では、テンプレート取得時に資産口座や保有商品が存在することを必須としない。

例えば、操作対象利用者に商品単位の資産口座や保有商品がまだ登録されていない場合でも、CSVテンプレート自体は取得可能とする。

資産口座および保有商品の存在確認は、CSV-005およびCSV-006でCSV内容を検証する際に行う。

---

#### 6.5.6 残高記録単位による取得制限を行わない

CSV-004では、操作対象利用者に

```text
balance_recording_unit
    = 商品単位
```

の資産口座が存在するかどうかによって、テンプレート取得可否を変更しない。

本APIは、商品単位用CSVの入力形式を提供することに責務を限定する。

残高記録単位とCSV種別の整合性は、CSVインポート時の入力チェックで確認する。

---

#### 6.5.7 保有商品をテンプレートへ事前展開しない

商品単位CSVでは、保有商品名を入力項目として使用する。

ただし、CSV-004で

```text
holding_assets
```

を検索し、登録済み保有商品をテンプレートへ事前展開する方式は採用しない。

テンプレートは、以下のヘッダーを持つ固定形式とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

保有商品の存在確認や対象年月時点での有効性確認は、CSV-005およびCSV-006で行う。

---

#### 6.5.8 X-User-Id未指定

`X-User-Id`が指定されていない場合は、API共通方針に従って

```text
USER_CONTEXT_REQUIRED
```

として扱う。

テンプレート生成処理へ進まない。

---

#### 6.5.9 X-User-Id形式不正

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

として扱う。

例えば、

```text
abc
-1
0
```

などを有効な利用者IDとして扱わない。

---

#### 6.5.10 利用者不存在

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

として扱う。

利用者が存在しない状態でCSVテンプレートを返却しない。

---

#### 6.5.11 将来の認証導入

将来的に認証機能を導入した場合は、`X-User-Id`による利用者指定を廃止し、

```text
認証済み利用者
```

から操作対象利用者を特定する方式へ変更する。

ただし、CSVテンプレートの以下には影響させない。

- ヘッダー
- CSV形式
- CSV-005・CSV-006との関係

---

### 6.6 パスパラメータ

本APIでは、
パスパラメータを使用しない。

エンドポイントは、
以下とする。

```http
GET /api/v1/month-end-holding-values/csv-template
```

利用者ID、
対象年月、
資産口座ID、
保有商品IDなどを
URLへ含めない。

操作対象利用者は、
`X-User-Id`
リクエストヘッダーから特定する。

---

### 6.7 クエリパラメータ

本APIでは、
クエリパラメータを使用しない。

CSVテンプレートは、
固定形式で返却する。

そのため、
以下のような
クエリパラメータによって
テンプレート内容を変更しない。

- `targetYearMonth`
- `assetAccountId`
- `holdingAssetId`
- `includeSample`
- `encoding`
- `bom`

CSVフォーマット、
文字コード、
BOMなどは、
CSV共通仕様として統一する。

---

### 6.8 リクエストヘッダー

本APIでは、
以下のリクエストヘッダーを使用する。

| ヘッダー | 必須 | 内容 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | CSVレスポンスを受け取れる形式を指定する |

リクエスト例：

```http
GET /api/v1/month-end-holding-values/csv-template
X-User-Id: 1
Accept: text/csv
```

本APIでは、
リクエストボディを使用しないため、

```http
Content-Type
```

の指定は不要とする。

---

#### 6.8.1 X-User-Id

`X-User-Id`は、
API共通方針に従って検証する。

正常例：

```http
X-User-Id: 1
```

以下の場合は、
エラーとする。

```text
未指定
形式不正
存在しない利用者
論理削除済み利用者
```

具体的な独自エラーコードは、
「エラーレスポンス」で定義する。

---

#### 6.8.2 Accept

正常時は、
CSVファイルを返却するため、
`Accept`には

```http
text/csv
```

を指定することを基本とする。

ただし、
API共通クライアントの都合により
`*/*`などを許容する場合は、
API共通方針に従う。

正常レスポンスは
JSONではなくCSVとする。

---

### 6.9 リクエストボディ

本APIでは、
リクエストボディを使用しない。

以下の情報を
リクエストボディで
受け付けない。

- `userId`
- `targetYearMonth`
- `assetAccountId`
- `holdingAssetId`
- `assetAccountName`
- `holdingAssetName`
- `value`
- CSVテンプレート設定

本APIは、
商品別月末評価額CSVの
固定テンプレートを取得することに
責務を限定する。

---

### 6.10 レスポンスCSV仕様

本APIでは、
商品別月末評価額CSVとして
以下のヘッダーを持つ
テンプレートを返却する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

ヘッダー順序も
CSV仕様の一部とする。

---

#### 6.10.1 target_year_month

商品別月末評価額の
対象年月を入力する列とする。

形式は、

```text
YYYY-MM
```

とする。

例：

```text
2026-07
```

CSV-004では、
値を入力済みの状態では返却しない。

ヘッダーだけを提供する。

---

#### 6.10.2 asset_account_name

保有商品が属する
資産口座名を入力する列とする。

CSV-005およびCSV-006では、
この値と
操作対象利用者を組み合わせて
資産口座を特定する。

CSV-004では、
利用者の資産口座名を
テンプレートへ自動出力しない。

---

#### 6.10.3 holding_asset_name

商品別月末評価額を登録する
保有商品名を入力する列とする。

CSV-005およびCSV-006では、
資産口座と
`holding_asset_name`の組み合わせをもとに
対象保有商品を特定する。

CSV-004では、
既存の保有商品名を
テンプレートへ自動展開しない。

---

#### 6.10.4 value

対象年月末時点の
保有商品の評価額を入力する列とする。

Phase1では、
日本円の整数として扱う。

CSV-004では、
値を設定せず
ヘッダーのみを返却する。

---

#### 6.10.5 ヘッダー順序

ヘッダーは、
以下の順序で固定する。

```text
1. target_year_month
2. asset_account_name
3. holding_asset_name
4. value
```

CSV-005、
CSV-006でも
同じ順序を正式な入力形式として使用する。

---

#### 6.10.6 利用者固有データを含めない

CSVテンプレートには、
利用者固有のデータ行を含めない。

以下のような
出力は行わない。

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,投資信託A,
,証券口座,投資信託B,
```

正常なテンプレートは、
基本的に以下のみとする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

---

#### 6.10.7 内部IDを含めない

CSVテンプレートには、
以下の内部ID列を含めない。

- `user_id`
- `asset_account_id`
- `holding_asset_id`
- `month_end_asset_snapshot_id`
- `month_end_holding_value_id`

CSV入力では、
利用者が理解しやすい

- 資産口座名
- 保有商品名

を使用する。

内部IDへの変換は、
CSV-005およびCSV-006で
バックエンド側が行う。

---

### 6.11 バリデーション

本APIでは、
CSVファイルのアップロードや
業務入力値を受け付けないため、
CSV内容に対する
バリデーションは行わない。

本APIで必要となる
バリデーションは、
主に利用者コンテキストに関する
API共通検証のみとする。

概念的には、
以下となる。

```text
リクエスト受信
    ↓
X-User-Id検証
    ↓
利用者存在確認
    ↓
CSVテンプレート生成
```

---

#### 6.11.1 X-User-Id 必須

`X-User-Id`は、
必須とする。

未指定の場合は、
API共通方針に従って
エラーとする。

テンプレート生成処理へ
進まない。

---

#### 6.11.2 X-User-Id 形式

`X-User-Id`は、
有効な利用者ID形式であることを確認する。

概念的には、
1以上の整数として扱う。

正常例：

```text
1
2
100
```

不正例：

```text
0
-1
abc
1.5
```

---

#### 6.11.3 利用者存在確認

指定された利用者が
存在することを確認する。

概念的な条件は、
以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

存在しない場合、
または論理削除済みの場合は、
CSVテンプレートを返却しない。

---

#### 6.11.4 資産口座存在確認を行わない

CSV-004では、
商品単位の資産口座が
存在することを
テンプレート取得条件としない。

例えば、
操作対象利用者に
資産口座が0件でも、
CSVテンプレートは取得可能とする。

以下のような
バリデーションは行わない。

```text
商品単位の資産口座が
1件以上存在すること
```

---

#### 6.11.5 保有商品存在確認を行わない

操作対象利用者に
保有商品が存在しない場合でも、
CSVテンプレートは取得可能とする。

CSV-004では、
以下の存在確認を行わない。

```text
holding_assets
    1件以上存在
```

保有商品の存在確認は、
CSV-005およびCSV-006で
CSV内容を検証する際に行う。

---

#### 6.11.6 残高記録単位を検証しない

CSV-004では、
商品単位CSV用の
テンプレートを返却するだけである。

そのため、
利用者の資産口座について

```text
balance_recording_unit
    = 商品単位
```

であることを
取得前に検証しない。

残高記録単位の整合性確認は、
CSV-005およびCSV-006で行う。

---

#### 6.11.7 target_year_monthを検証しない

CSV-004のリクエストには、
`target_year_month`を含めない。

そのため、
対象年月の

- 必須
- `YYYY-MM`形式
- 有効年月
- 1ファイル1対象年月

などは、
CSV-004では検証しない。

これらは、
CSV-005およびCSV-006で
アップロードされたCSVに対して検証する。

---

#### 6.11.8 asset_account_nameを検証しない

CSV-004では、
`asset_account_name`を
入力値として受け取らない。

そのため、

- 必須
- 存在確認
- 利用者境界
- 残高記録単位

などの検証は行わない。

---

#### 6.11.9 holding_asset_nameを検証しない

CSV-004では、
`holding_asset_name`を
入力値として受け取らない。

そのため、

- 必須
- 保有商品存在確認
- 資産口座との関連確認
- 対象年月時点の有効性確認

などは行わない。

---

#### 6.11.10 valueを検証しない

CSV-004では、
`value`を
入力値として受け取らない。

そのため、

- 必須
- 整数
- 0以上
- 日本円として有効

などの検証は行わない。

商品別月末評価額の
入力値検証は、
CSV-005およびCSV-006で行う。

---

#### 6.11.11 CSVテンプレート定義の整合性

CSV-004では、
生成するテンプレートのヘッダーが
CSV共通Definitionと
一致することを保証する。

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

とする。

正式なヘッダーは、

```text
target_year_month
asset_account_name
holding_asset_name
value
```

とする。

---

#### 6.11.12 CSVテンプレート生成失敗

CSVテンプレート生成処理そのものが
失敗した場合は、
入力バリデーションエラーとして
扱わない。

想定外のサーバー内部エラーとして扱う。

具体的なHTTPステータスおよび
独自エラーコードは、
「エラーレスポンス」で定義する。

---

### 6.12 業務ルール

CSV-004では、
商品別月末評価額CSVを入力するための
固定形式のテンプレートを返却する。

本APIでは、
業務データの登録・更新・削除を行わない。

CSV-005 商品別月末評価額CSVプレビュー、
CSV-006 商品別月末評価額CSV登録で使用する
CSV形式の起点となる。

---

#### 6.12.1 CSVテンプレートは固定形式とする

商品別月末評価額CSVテンプレートは、
以下の固定ヘッダーを持つ。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

利用者や
登録済みデータによって
ヘッダー構成を変更しない。

---

#### 6.12.2 ヘッダー順序を固定する

ヘッダー順序は、
以下とする。

```text
1. target_year_month
2. asset_account_name
3. holding_asset_name
4. value
```

CSV-005およびCSV-006でも、
同じヘッダー名・順序を
正式なCSV形式として使用する。

---

#### 6.12.3 1行目をヘッダーとする

CSVテンプレートの
1行目は、
必ずヘッダー行とする。

概念的には、
以下となる。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

CSV-005およびCSV-006では、
1行目をヘッダーとして解析する。

---

#### 6.12.4 データ行を含めない

CSV-004では、
テンプレートに
データ行を含めない。

正常なテンプレートは、
以下の1行のみとする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

サンプルデータや
登録済みデータを
自動出力しない。

---

#### 6.12.5 利用者固有データを含めない

CSVテンプレートには、
操作対象利用者の

- 資産口座
- 保有商品
- 商品別月末評価額

を出力しない。

例えば、
以下のようなテンプレートは
生成しない。

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,全世界株式,
,証券口座,S&P500,
```

テンプレートは、
利用者に依存しない
固定形式とする。

---

#### 6.12.6 内部IDを使用しない

商品別月末評価額CSVでは、
資産口座および保有商品を
内部IDではなく名称で指定する。

そのため、
テンプレートには以下を含めない。

```text
user_id
asset_account_id
holding_asset_id
month_end_asset_snapshot_id
month_end_holding_value_id
```

CSV入力では、

```text
asset_account_name
holding_asset_name
```

を使用する。

内部IDへの変換は、
CSV-005およびCSV-006の
バックエンド処理で行う。

---

#### 6.12.7 target_year_month

`target_year_month`には、
商品別月末評価額の
対象年月を入力する。

形式は、

```text
YYYY-MM
```

とする。

例：

```text
2026-07
```

1つのCSVファイルでは、
1つの対象年月のみを扱う。

CSV-004では値を設定せず、
列のみを提供する。

---

#### 6.12.8 asset_account_name

`asset_account_name`には、
保有商品が属する
資産口座名を入力する。

CSV-005およびCSV-006では、

```text
操作対象利用者
+
asset_account_name
```

によって
対象資産口座を特定する。

他利用者の同名資産口座を
対象としてはならない。

---

#### 6.12.9 holding_asset_name

`holding_asset_name`には、
商品別月末評価額を登録する
保有商品名を入力する。

CSV-005およびCSV-006では、
概念的に

```text
操作対象利用者
+
資産口座
+
holding_asset_name
```

によって
対象保有商品を特定する。

同じ名称の保有商品が
別の資産口座に存在していても、
指定された資産口座に属する
保有商品を対象とする。

---

#### 6.12.10 value

`value`には、
対象年月末時点の
保有商品の評価額を入力する。

Phase1では、
日本円の整数として扱う。

概念例：

```text
1500000
```

CSV上では、
桁区切りや通貨記号を使用しない。

例えば、

```text
1,500,000
¥1500000
```

のような形式を
正式な入力形式とはしない。

具体的な入力値検証は、
CSV-005およびCSV-006で行う。

---

#### 6.12.11 0円を有効値とする

商品別月末評価額では、

```text
value = 0
```

を有効な値として扱う。

0円を

- 未入力
- NULL
- 登録対象外

として扱わない。

CSV-004では値を持たないが、
CSV-005およびCSV-006で
0円を正常値として扱う。

---

#### 6.12.12 商品単位の資産口座を対象とする

商品別月末評価額CSVは、
残高記録単位が
商品単位の資産口座に属する
保有商品を対象とする。

概念的には、

```text
asset_accounts.balance_recording_unit
    = 商品単位
```

であることを
CSV-005およびCSV-006で確認する。

CSV-004では、
この業務ルールの判定自体は行わない。

---

#### 6.12.13 保有商品単位で1行とする

CSVでは、
1つの保有商品について
1行を使用する。

概念例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

同一対象年月、
同一資産口座、
同一保有商品を
複数行に分けて登録することは想定しない。

重複判定は、
CSV-005およびCSV-006で行う。

---

#### 6.12.14 1ファイル1対象年月とする

1つのCSVファイルに、
複数の対象年月を混在させない。

正常例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-06,証券口座,全世界株式,1400000
2026-07,証券口座,S&P500,800000
```

CSV-004では
対象年月を受け取らないため、
この判定は行わない。

---

#### 6.12.15 テンプレート取得時に業務データを検索しない

CSV-004では、
テンプレート生成のために

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

を検索しない。

テンプレート生成は、
業務データの状態に
依存させない。

---

#### 6.12.16 CSV-005・CSV-006と同じ定義を使用する

CSV-004、
CSV-005、
CSV-006では、
商品別月末評価額CSVの
同一Definitionを使用する。

概念的には、

```text
MonthEndHoldingValueCsvDefinition
    ├─ CSV-004
    ├─ CSV-005
    └─ CSV-006
```

とする。

これにより、

```text
CSV-004で取得したテンプレート
    ↓
CSV-005では形式不正
```

となる実装差異を防止する。

---

#### 6.12.17 CSVテンプレート取得による副作用を発生させない

CSV-004を何度実行しても、
業務データの状態は変化しない。

以下を行わない。

- 月末資産状況作成
- 商品別月末評価額作成
- 資産口座更新
- 保有商品更新
- CSVインポート履歴作成

テンプレート取得回数によって
CSV内容が変化することもない。

---

### 6.13 処理フロー

正常系の処理フローは、
以下とする。

```text
GET
/api/v1/month-end-holding-values/csv-template
    ↓
X-User-Id取得
    ↓
利用者コンテキスト検証
    ↓
商品別月末評価額CSV Definition取得
    ↓
CSVヘッダー生成
    ↓
CSVレスポンス生成
    ↓
200 OK
```

本APIでは、
テンプレート生成のための
業務データ検索を行わない。

---

#### 6.13.1 利用者コンテキスト確認

API共通処理で、
`X-User-Id`から
操作対象利用者を特定する。

以下の場合は、
CSV生成へ進まない。

```text
X-User-Id未指定
X-User-Id形式不正
利用者不存在
論理削除済み利用者
```

---

#### 6.13.2 CSV Definition取得

商品別月末評価額CSVの
共通Definitionから、
ヘッダー定義を取得する。

概念的には、

```text
target_year_month
asset_account_name
holding_asset_name
value
```

を取得する。

CSV-004専用に
別のヘッダー定義を
ハードコードしない。

---

#### 6.13.3 CSV生成

取得したヘッダーを使用して、
CSVを生成する。

生成内容は、
概念的に以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

データ行は追加しない。

---

#### 6.13.4 CSVレスポンス生成

生成したCSVを、
ファイルダウンロード可能な
HTTPレスポンスとして返却する。

正常時は、

```http
200 OK
```

とする。

レスポンス本文は、
JSONではなくCSVとする。

---

#### 6.13.5 異常系

利用者コンテキストに
問題がある場合は、
API共通の
JSONエラーレスポンスを返却する。

概念的には、

```text
利用者コンテキストエラー
    ↓
JSONエラーレスポンス
```

となる。

CSV生成処理中に
想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

として扱う。

---

### 6.14 正常レスポンス

正常時は、

```http
200 OK
```

を返却する。

レスポンス本文は、
CSVファイルとする。

概念例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

本APIでは、
API共通のJSON成功Envelopeを
使用しない。

---

#### 6.14.1 Content-Type

正常レスポンスの
`Content-Type`は、

```http
text/csv
```

とする。

文字コード情報を
付与する場合は、
CSV共通仕様に従う。

概念例：

```http
Content-Type: text/csv; charset=UTF-8
```

---

#### 6.14.2 Content-Disposition

ブラウザから
CSVファイルとして
保存できるよう、

```http
Content-Disposition
```

を設定する。

概念例：

```http
Content-Disposition: attachment; filename="month-end-holding-values-template.csv"
```

正式なファイル名は、
CSV共通の命名方針に従う。

---

#### 6.14.3 レスポンスボディ

レスポンスボディは、
CSV文字列とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

JSONへ変換しない。

---

#### 6.14.4 requestId

正常レスポンスが
CSVファイルそのものであるため、
JSON本文へ

```text
requestId
```

を含めない。

リクエスト追跡が必要な場合は、
API共通方針に従って
HTTPレスポンスヘッダーなどで扱う。

エラーレスポンスについては、
API共通のJSONエラー形式を使用する。

---

### 6.15 レスポンス項目

本APIの正常レスポンスは
JSONではなくCSVであるため、
JSONレスポンス項目は存在しない。

CSVの列定義は、
以下とする。

| 順序 | CSV列名 | 内容 | 入力例 |
|---:|---|---|---|
| 1 | `target_year_month` | 商品別月末評価額の対象年月 | `2026-07` |
| 2 | `asset_account_name` | 保有商品が属する資産口座名 | `証券口座` |
| 3 | `holding_asset_name` | 商品別月末評価額を登録する保有商品名 | `全世界株式` |
| 4 | `value` | 対象年月末時点の商品別評価額 | `1500000` |

---

#### 6.15.1 target_year_month

CSV上の物理名：

```text
target_year_month
```

意味：

```text
商品別月末評価額の対象年月
```

入力形式：

```text
YYYY-MM
```

CSV-004では、
値を設定しない。

---

#### 6.15.2 asset_account_name

CSV上の物理名：

```text
asset_account_name
```

意味：

```text
保有商品が属する資産口座名
```

CSV-005およびCSV-006で、
操作対象利用者に属する
資産口座を特定するために使用する。

---

#### 6.15.3 holding_asset_name

CSV上の物理名：

```text
holding_asset_name
```

意味：

```text
商品別月末評価額を登録する保有商品名
```

CSV-005およびCSV-006で、
指定された資産口座に属する
保有商品を特定するために使用する。

---

#### 6.15.4 value

CSV上の物理名：

```text
value
```

意味：

```text
対象年月末時点の商品別月末評価額
```

Phase1では、
日本円の整数として扱う。

CSV-004では、
値を設定しない。

---

#### 6.15.5 返却しない情報

CSVテンプレートには、
以下の情報を返却しない。

- `user_id`
- `asset_account_id`
- `holding_asset_id`
- `month_end_asset_snapshot_id`
- `month_end_holding_value_id`
- `balance_recording_unit`
- `confirmed`
- `created_at`
- `updated_at`
- 利用可能資産設定
- 登録済み商品別月末評価額
- 登録済み資産口座名
- 登録済み保有商品名

CSV-004は、
商品別月末評価額CSVの
入力形式を提供することだけを
責務とする。

---

### 6.12 業務ルール

CSV-004では、
商品別月末評価額CSVを入力するための
固定形式のテンプレートを返却する。

本APIでは、
業務データの登録・更新・削除を行わない。

CSV-005 商品別月末評価額CSVプレビュー、
CSV-006 商品別月末評価額CSV登録で使用する
CSV形式の起点となる。

---

#### 6.12.1 CSVテンプレートは固定形式とする

商品別月末評価額CSVテンプレートは、
以下の固定ヘッダーを持つ。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

利用者や
登録済みデータによって
ヘッダー構成を変更しない。

---

#### 6.12.2 ヘッダー順序を固定する

ヘッダー順序は、
以下とする。

```text
1. target_year_month
2. asset_account_name
3. holding_asset_name
4. value
```

CSV-005およびCSV-006でも、
同じヘッダー名・順序を
正式なCSV形式として使用する。

---

#### 6.12.3 データ行を含めない

CSV-004では、
テンプレートに
データ行を含めない。

正常なテンプレートは、
以下の1行のみとする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

サンプルデータや
登録済みデータを
自動出力しない。

---

#### 6.12.4 利用者固有データを含めない

CSVテンプレートには、
操作対象利用者の

- 資産口座
- 保有商品
- 商品別月末評価額

を出力しない。

テンプレートは、
利用者に依存しない
固定形式とする。

---

#### 6.12.5 内部IDを使用しない

商品別月末評価額CSVでは、
資産口座および保有商品を
内部IDではなく名称で指定する。

テンプレートには、
以下の内部IDを含めない。

- `user_id`
- `asset_account_id`
- `holding_asset_id`
- `month_end_asset_snapshot_id`
- `month_end_holding_value_id`

CSV入力では、

```text
asset_account_name
holding_asset_name
```

を使用する。

内部IDへの変換は、
CSV-005およびCSV-006で
バックエンド側が行う。

---

#### 6.12.6 target_year_month

`target_year_month`には、
商品別月末評価額の
対象年月を入力する。

形式は、

```text
YYYY-MM
```

とする。

例：

```text
2026-07
```

1つのCSVファイルでは、
1つの対象年月のみを扱う。

CSV-004では、
値を設定せず
入力列のみを提供する。

---

#### 6.12.7 asset_account_name

`asset_account_name`には、
保有商品が属する
資産口座名を入力する。

CSV-005およびCSV-006では、

```text
操作対象利用者
+
asset_account_name
```

によって
対象資産口座を特定する。

他利用者の
同名資産口座を
対象としてはならない。

---

#### 6.12.8 holding_asset_name

`holding_asset_name`には、
商品別月末評価額を登録する
保有商品名を入力する。

CSV-005およびCSV-006では、

```text
操作対象利用者
+
資産口座
+
holding_asset_name
```

によって
対象保有商品を特定する。

同名の保有商品が
別の資産口座に存在する場合でも、
指定された資産口座に属する
保有商品を対象とする。

---

#### 6.12.9 value

`value`には、
対象年月末時点の
保有商品の評価額を入力する。

Phase1では、
日本円の整数として扱う。

正常例：

```text
1500000
0
```

以下のような形式は、
正式な入力形式としない。

```text
1,500,000
¥1500000
1500000.5
```

具体的な入力値検証は、
CSV-005およびCSV-006で行う。

---

#### 6.12.10 0円を有効値とする

商品別月末評価額では、

```text
value = 0
```

を有効な値として扱う。

0円を

- 未入力
- NULL
- 登録対象外

として扱わない。

---

#### 6.12.11 商品単位の資産口座を対象とする

商品別月末評価額CSVでは、
残高記録単位が
商品単位の資産口座に属する
保有商品を対象とする。

概念的には、

```text
asset_accounts.balance_recording_unit
    = 商品単位
```

であることを
CSV-005およびCSV-006で確認する。

CSV-004では、
この業務ルールの判定自体は行わない。

---

#### 6.12.12 保有商品単位で1行とする

CSVでは、
1つの保有商品について
1行を使用する。

概念例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

同一対象年月、
同一資産口座、
同一保有商品を
複数行に分けて登録しない。

重複判定は、
CSV-005およびCSV-006で行う。

---

#### 6.12.13 1ファイル1対象年月とする

1つのCSVファイルに、
複数の対象年月を混在させない。

正常例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-06,証券口座,全世界株式,1400000
2026-07,証券口座,S&P500,800000
```

CSV-004では
対象年月を受け取らないため、
この判定自体は行わない。

---

#### 6.12.14 CSV-005・CSV-006と同じ定義を使用する

CSV-004、
CSV-005、
CSV-006では、
商品別月末評価額CSVの
同一CSV定義を使用する。

概念的には、

```text
MonthEndHoldingValueCsvDefinition
    ├─ CSV-004
    ├─ CSV-005
    └─ CSV-006
```

とする。

これにより、

```text
CSV-004で取得したテンプレート
    ↓
CSV-005・CSV-006では形式不正
```

となる実装差異を防止する。

---

#### 6.12.15 テンプレート取得時に業務データを検索しない

CSV-004では、
テンプレート生成のために

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

を検索しない。

テンプレート生成は、
業務データの状態に
依存させない。

---

#### 6.12.16 副作用を発生させない

CSV-004を何度実行しても、
業務データの状態は変化しない。

以下を行わない。

- 月末資産状況作成
- 商品別月末評価額作成
- 資産口座更新
- 保有商品更新
- 月末資産状況確定

テンプレート取得回数によって
テンプレート内容が変化することもない。

---

### 6.13 処理フロー

正常系の処理フローは、
以下とする。

```text
GET
/api/v1/month-end-holding-values/csv-template
    ↓
X-User-Id取得
    ↓
利用者コンテキスト検証
    ↓
商品別月末評価額CSV Definition取得
    ↓
CSVヘッダー生成
    ↓
CSVレスポンス生成
    ↓
200 OK
```

本APIでは、
テンプレート生成のための
業務データ検索を行わない。

---

#### 6.13.1 利用者コンテキスト確認

API共通処理で、
`X-User-Id`から
操作対象利用者を特定する。

以下の場合は、
CSV生成へ進まない。

```text
X-User-Id未指定
X-User-Id形式不正
利用者不存在
論理削除済み利用者
```

---

#### 6.13.2 CSV Definition取得

商品別月末評価額CSVの
共通Definitionから、
ヘッダー定義を取得する。

概念的には、
以下となる。

```text
target_year_month
asset_account_name
holding_asset_name
value
```

CSV-004専用に
別のヘッダー定義を
持たない。

---

#### 6.13.3 CSV生成

取得したヘッダー定義から、
CSVを生成する。

生成内容は、
以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

データ行は追加しない。

---

#### 6.13.4 CSVレスポンス生成

生成したCSVを、
ファイルとして取得可能な
HTTPレスポンスへ変換する。

正常時は、

```http
200 OK
```

を返却する。

レスポンス本文は、
JSONではなくCSVとする。

---

#### 6.13.5 異常系

利用者コンテキストに
問題がある場合は、
API共通の
JSONエラーレスポンスを返却する。

CSV生成処理中に
想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

として扱う。

---

### 6.14 正常レスポンス

正常時は、

```http
200 OK
```

を返却する。

レスポンス本文は、
CSVファイルとする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

本APIでは、
API共通の
JSON成功Envelopeを使用しない。

---

#### 6.14.1 Content-Type

正常レスポンスの
`Content-Type`は、

```http
text/csv; charset=UTF-8
```

を基本とする。

正式な文字コード指定は、
CSV共通仕様に従う。

---

#### 6.14.2 Content-Disposition

ブラウザから
CSVファイルとして
取得できるよう、

```http
Content-Disposition
```

を設定する。

概念例：

```http
Content-Disposition: attachment; filename="month-end-holding-values-template.csv"
```

正式なファイル名は、
CSV共通の命名方針に従う。

---

#### 6.14.3 レスポンスボディ

レスポンスボディは、
CSV文字列とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

JSONへ変換しない。

---

#### 6.14.4 成功Envelope

本APIでは、
正常レスポンスが
CSVファイルそのものであるため、
以下のJSON成功Envelopeを使用しない。

```json
{
  "data": {},
  "requestId": "..."
}
```

CSVファイルへ
API管理用のメタ情報を
混在させない。

---

#### 6.14.5 requestId

正常レスポンスの
CSV本文には、

```text
requestId
```

を含めない。

リクエスト追跡情報を
レスポンスへ付与する場合は、
API共通方針に従って
HTTPレスポンスヘッダーで扱う。

異常時は、
API共通のJSONエラーレスポンスを使用する。

---

### 6.15 レスポンス項目

本APIの正常レスポンスは
JSONではなくCSVであるため、
JSONレスポンス項目は存在しない。

CSV列は、
以下とする。

| 順序 | CSV列名 | 内容 | 入力例 |
|---:|---|---|---|
| 1 | `target_year_month` | 商品別月末評価額の対象年月 | `2026-07` |
| 2 | `asset_account_name` | 保有商品が属する資産口座名 | `証券口座` |
| 3 | `holding_asset_name` | 商品別月末評価額を登録する保有商品名 | `全世界株式` |
| 4 | `value` | 対象年月末時点の商品別月末評価額 | `1500000` |

---

#### 6.15.1 target_year_month

CSV上の物理名：

```text
target_year_month
```

意味：

```text
商品別月末評価額の対象年月
```

入力形式：

```text
YYYY-MM
```

CSV-004では、
値を設定しない。

---

#### 6.15.2 asset_account_name

CSV上の物理名：

```text
asset_account_name
```

意味：

```text
保有商品が属する資産口座名
```

CSV-005およびCSV-006で、
操作対象利用者に属する
資産口座を特定するために使用する。

CSV-004では、
登録済み資産口座名を
出力しない。

---

#### 6.15.3 holding_asset_name

CSV上の物理名：

```text
holding_asset_name
```

意味：

```text
商品別月末評価額を登録する保有商品名
```

CSV-005およびCSV-006で、
指定された資産口座に属する
保有商品を特定するために使用する。

CSV-004では、
登録済み保有商品名を
出力しない。

---

#### 6.15.4 value

CSV上の物理名：

```text
value
```

意味：

```text
対象年月末時点の商品別月末評価額
```

Phase1では、
日本円の整数として扱う。

CSV-004では、
値を設定しない。

---

#### 6.15.5 返却しない情報

CSVテンプレートには、
以下の情報を返却しない。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`
- `asset_accounts.balance_recording_unit`
- `month_end_asset_snapshots.confirmed`
- `created_at`
- `updated_at`
- `asset_account_available_settings`
- `asset_accounts.name`
- `holding_assets.name`
- `month_end_holding_values.value`

CSV-004は、
商品別月末評価額CSVの
入力形式を提供することだけを
責務とする。

---

### 6.16 エラーレスポンス

CSV-004では、
正常時はCSVファイルを返却する。

一方、
エラー時はCSVではなく、
API共通方針に従った
JSONエラーレスポンスを返却する。

概念的には、
以下とする。

```text
正常時
    ↓
text/csv

エラー時
    ↓
application/json
```

本APIでは、
CSVファイルのアップロードや
CSV内容の検証を行わないため、
CSV入力内容に関する
業務エラーは発生しない。

---

#### 6.16.1 エラー一覧

本APIで想定する
主なエラーは、
以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`が指定されていない |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`の形式が不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 指定された利用者が存在しない、または論理削除済み |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | CSV生成処理などで想定外のエラーが発生した |

CSV-004では、
以下のようなエラーは発生しない。

- CSVヘッダー不正
- CSVデータ行不正
- 対象年月不正
- 資産口座不存在
- 保有商品不存在
- 月末評価額不正
- CSV内重複
- 月末資産状況確定済み
- 商品別月末評価額登録済み

これらは、
CSV-005およびCSV-006で扱う。

---

#### 6.16.2 USER_CONTEXT_REQUIRED

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

HTTPステータスは、

```http
400 Bad Request
```

とする。

CSVテンプレート生成処理へ
進まない。

---

#### 6.16.3 INVALID_USER_ID

`X-User-Id`の形式が
不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

HTTPステータスは、

```http
400 Bad Request
```

とする。

例えば、
以下のような値を
不正とする。

```text
0
-1
abc
1.5
```

---

#### 6.16.4 USER_NOT_FOUND

指定された利用者が
存在しない場合、
または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

HTTPステータスは、

```http
404 Not Found
```

とする。

利用者が存在しない状態で
CSVテンプレートを
返却しない。

---

#### 6.16.5 INTERNAL_SERVER_ERROR

CSVテンプレート生成処理などで
想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

HTTPステータスは、

```http
500 Internal Server Error
```

とする。

レスポンスへ、
以下の内部情報を含めない。

- スタックトレース
- PHP内部エラー
- Laravel内部例外メッセージ
- サーバーファイルパス
- クラス名
- SQL
- データベース接続情報

詳細情報は、
サーバーログへ記録する。

---

#### 6.16.6 エラーレスポンス形式

エラー時は、
API共通の
JSONエラーレスポンスを使用する。

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

正常時のCSVレスポンス形式と
エラー時のJSON形式を
混在させない。

---

### 6.17 HTTPステータス

本APIで使用する
HTTPステータスは、
以下とする。

| HTTPステータス | 用途 |
|---|---|
| `200 OK` | CSVテンプレート取得成功 |
| `400 Bad Request` | 利用者コンテキストの指定不備 |
| `404 Not Found` | 指定された利用者が存在しない |
| `500 Internal Server Error` | 想定外のサーバー内部エラー |

本APIでは、
通常、

```text
409 Conflict
422 Unprocessable Entity
```

は使用しない。

---

#### 6.17.1 200 OK

CSVテンプレートを
正常に生成できた場合は、

```http
200 OK
```

を返却する。

レスポンス本文は、
以下のCSVとする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

---

#### 6.17.2 400 Bad Request

以下の場合に使用する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
```

CSVテンプレートの
内容に関するエラーには使用しない。

---

#### 6.17.3 404 Not Found

指定された利用者が
存在しない場合、
または論理削除済みの場合に使用する。

```text
USER_NOT_FOUND
```

---

#### 6.17.4 500 Internal Server Error

CSV生成処理などで
想定外のエラーが発生した場合に使用する。

```text
INTERNAL_SERVER_ERROR
```

正常な空CSVへ
フォールバックしない。

---

### 6.18 副作用

本APIには、
業務データに対する
副作用はない。

CSV-004の実行によって、
データベースの
登録・更新・削除を行わない。

---

#### 6.18.1 更新しないテーブル

本APIでは、
以下のテーブルを更新しない。

- `users`
- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_account_available_settings`
- `net_incomes`
- `objectives`
- `assessment_histories`

CSVテンプレート取得によって、
業務データの状態を
変更してはならない。

---

#### 6.18.2 CSVインポート履歴を作成しない

Phase1では、
CSVテンプレート取得履歴を
データベースへ保存しない。

以下のような
履歴データを作成しない。

```text
csv_download_histories
csv_template_histories
csv_import_sessions
```

必要なアクセス記録は、
API共通の
サーバーログで扱う。

---

#### 6.18.3 ファイルを永続保存しない

生成したCSVテンプレートを、
サーバー上の
永続ファイルとして保存しない。

例えば、

```text
storage/app/templates/...
```

へ
リクエストごとに
ファイルを作成する方式は
採用しない。

レスポンス用に
メモリまたは一時ストリーム上で
生成する。

---

### 6.19 トランザクション

CSV-004では、
データベース更新を行わないため、
明示的な
トランザクションを使用しない。

以下のような実装は
行わない。

```php
DB::transaction(
    function () {
        // CSVテンプレート生成
    },
);
```

テンプレート生成処理に
トランザクションは不要である。

---

#### 6.19.1 読み取りトランザクションも使用しない

CSV-004では、
テンプレート生成のために
業務データを検索しない。

そのため、
読み取り整合性を目的とした
トランザクションも不要とする。

---

### 6.20 ロック

CSV-004では、
データベースの
行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

本APIの実行によって、
CSV-005、
CSV-006、
月末資産状況関連処理などを
ブロックしてはならない。

---

### 6.21 キャッシュ

Phase1では、
CSV-004専用の
サーバー側アプリケーションキャッシュを
使用しない。

CSVテンプレートは
ヘッダー1行のみであり、
生成コストが小さいためである。

また、
CSV仕様変更時に
古いテンプレートが
キャッシュされ続ける
複雑性を避ける。

---

#### 6.21.1 テンプレート内容

CSVテンプレートは
利用者固有データに依存しないため、
同じバージョンのAPIでは
同一内容となる。

概念的には、
常に以下を返却する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

---

#### 6.21.2 将来のHTTPキャッシュ

将来的に
必要となった場合は、

```text
ETag
Cache-Control
```

などの
HTTPキャッシュを検討してよい。

ただし、
Phase1では実装しない。

---

### 6.22 冪等性

本APIは、
参照専用の`GET` APIであり、
冪等である。

同じ条件で
複数回実行しても、
業務データの状態は変化しない。

---

#### 6.22.1 同一利用者での複数回実行

同じ

```text
X-User-Id
```

で
複数回実行した場合は、
同じCSVテンプレートを返却する。

概念的には、

```text
1回目
    ↓
target_year_month,asset_account_name,holding_asset_name,value

2回目
    ↓
target_year_month,asset_account_name,holding_asset_name,value
```

となる。

---

#### 6.22.2 利用者が異なる場合

CSVテンプレートは
利用者固有データを含まないため、
有効な利用者であれば
利用者が異なっても
同じテンプレートを返却する。

例えば、

```text
User A
    ↓
同じテンプレート

User B
    ↓
同じテンプレート
```

とする。

---

#### 6.22.3 業務データ変更の影響を受けない

CSV-004のテンプレート内容は、

- 資産口座の追加
- 資産口座の削除
- 保有商品の追加
- 保有商品の削除
- 商品別月末評価額の登録
- 月末資産状況の確定

などによって
変化しない。

CSV仕様そのものが
変更された場合のみ、
テンプレート内容を変更する。

---

#### 6.22.4 Idempotency-Key

本APIは
`GET`かつ参照専用であるため、

```text
Idempotency-Key
```

を使用しない。

Idempotency Keyによる
重複実行制御は不要である。

---

#### 6.22.5 複数回ダウンロード

利用者が
同じテンプレートを
複数回ダウンロードしても
問題ない。

ダウンロード回数によって、

- CSV内容
- 業務データ
- 利用者状態

を変更しない。

---

#### 6.22.6 CSV-005・CSV-006との関係

CSV-004を
何度取得しても、
CSV-005およびCSV-006の
登録・検証状態には影響しない。

CSV-004は、
CSV入力形式を提供することだけに
責務を限定する。

---

### 6.23 関連テーブル

CSV-004では、
商品別月末評価額CSVテンプレートを
固定形式で生成するため、
業務データを保持するテーブルを
直接参照しない。

テンプレート取得時に、
以下のテーブルから
CSVデータ行を生成しない。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`
- `asset_account_available_settings`

ただし、
`X-User-Id`による
利用者コンテキストの確認では、
API共通処理として
`users`を参照する。

---

#### 6.23.1 users

`X-User-Id`で指定された
利用者の存在確認に使用する。

概念的な参照条件は、
以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

CSVテンプレートの
内容生成には使用しない。

---

#### 6.23.2 asset_accounts

CSV-004では、
`asset_accounts`を参照しない。

以下の情報を
テンプレートへ出力しない。

- `asset_accounts.id`
- `asset_accounts.name`
- `asset_accounts.balance_recording_unit`

商品単位の資産口座が
存在しない場合でも、
テンプレートは取得可能とする。

---

#### 6.23.3 holding_assets

CSV-004では、
`holding_assets`を参照しない。

以下の情報を
テンプレートへ出力しない。

- `holding_assets.id`
- `holding_assets.name`

登録済み保有商品の
一覧を取得して
CSVデータ行へ展開する処理は行わない。

---

#### 6.23.4 month_end_asset_snapshots

CSV-004では、
`month_end_asset_snapshots`を参照しない。

以下の情報によって
テンプレート取得可否を
変更しない。

- 対象年月のsnapshot存在有無
- `month_end_asset_snapshots.confirmed`

月末資産状況が
1件も存在しない利用者でも、
テンプレートは取得可能とする。

---

#### 6.23.5 month_end_holding_values

CSV-004では、
`month_end_holding_values`を参照しない。

登録済みの
商品別月末評価額を
テンプレートへ出力しない。

CSV-004は、
既存データのエクスポートAPIではなく、
CSVインポート用テンプレートの
取得APIである。

---

#### 6.23.6 asset_account_available_settings

CSV-004では、
`asset_account_available_settings`を
参照しない。

利用可能資産としての
設定状態は、
CSVテンプレートの形式に
影響しない。

---

#### 6.23.7 テーブル更新

本APIでは、
関連するすべてのテーブルに対して
INSERT、
UPDATE、
DELETEを行わない。

概念的には、

```text
users
    → SELECTのみ

その他の業務テーブル
    → 参照なし・更新なし
```

とする。

---

### 6.24 インデックス

CSV-004専用の
インデックスは追加しない。

本APIでは、
テンプレート生成のために
業務テーブルを検索しないためである。

---

#### 6.24.1 users

利用者存在確認では、

```text
users.id
```

を使用する。

`id`は主キーであるため、
主キーインデックスを利用する。

CSV-004のために
追加インデックスを作成しない。

---

#### 6.24.2 業務テーブル

以下のテーブルについて、
CSV-004を理由とした
インデックス追加は行わない。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`
- `asset_account_available_settings`

CSV-005、
CSV-006などで必要となる
インデックスについては、
各APIの検索条件をもとに
別途判断する。

---

### 6.25 性能

CSV-004は、
固定されたCSVヘッダーのみを
返却するため、
処理負荷は小さい。

概念的な処理は、

```text
利用者コンテキスト確認
    ↓
CSV Definition取得
    ↓
CSVヘッダー生成
    ↓
レスポンス返却
```

のみとする。

---

#### 6.25.1 業務データ検索を行わない

テンプレート生成時に、

```text
asset_accounts
holding_assets
month_end_asset_snapshots
month_end_holding_values
```

などを検索しない。

そのため、
利用者が保有する

- 資産口座数
- 保有商品数
- 商品別月末評価額件数

によって、
CSV-004の処理量が
増加しない設計とする。

---

#### 6.25.2 N+1問題

CSV-004では、
業務データの一覧取得を行わないため、
N+1問題は発生しない。

以下のような処理は行わない。

```text
資産口座一覧取得
    ↓
各資産口座ごとに
保有商品一覧取得
```

---

#### 6.25.3 CSV生成

CSVテンプレートは、
ヘッダー1行のみであるため、
大規模なメモリ使用を伴わない。

概念的には、
以下の固定内容を生成する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

大規模CSV向けの
chunk処理や
ストリーミング処理は
Phase1では不要とする。

---

#### 6.25.4 ファイルI/O

CSVテンプレートを
サーバー上へ
永続ファイルとして保存しない。

概念的には、

```text
CSV Definition
    ↓
メモリ上でCSV生成
    ↓
HTTPレスポンス
```

とする。

以下のような処理は
原則として行わない。

```text
CSV生成
    ↓
storageへ保存
    ↓
ファイル再読込
    ↓
レスポンス
```

---

#### 6.25.5 キャッシュ

CSVテンプレートは
固定かつ生成コストが小さいため、
Phase1では
専用キャッシュを導入しない。

キャッシュ管理による
複雑性を増やさないことを
優先する。

---

### 6.26 セキュリティ

CSV-004では、
CSVテンプレート自体に
利用者固有情報を含めない。

ただし、
Phase1のAPI共通方針として
`X-User-Id`による
利用者コンテキストの検証を行う。

---

#### 6.26.1 利用者境界

`X-User-Id`で
指定された利用者が
有効であることを確認する。

CSV-004では
利用者固有の業務データを
取得しないため、
他利用者の

- 資産口座
- 保有商品
- 商品別月末評価額

がレスポンスへ
混入することはない。

---

#### 6.26.2 内部IDを公開しない

CSVテンプレートには、
以下の内部IDを含めない。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`

CSV入力では、

```text
asset_account_name
holding_asset_name
```

を使用する。

---

#### 6.26.3 業務データを公開しない

CSVテンプレートには、
以下の既存業務データを含めない。

- `asset_accounts.name`
- `holding_assets.name`
- `month_end_holding_values.value`
- `asset_accounts.balance_recording_unit`
- `month_end_asset_snapshots.confirmed`
- `asset_account_available_settings`

CSV-004を
業務データの取得手段として
利用できない設計とする。

---

#### 6.26.4 CSV Injection

CSV-004では、
利用者入力値や
データベース値を
CSVデータ行へ出力しない。

返却内容は、
システムで定義した
固定ヘッダーのみである。

そのため、
Phase1のCSV-004では
CSV Injection対策としての
利用者入力値エスケープ処理は
不要とする。

CSVへ業務データを出力するAPIを
将来追加する場合は、
別途CSV Injection対策を検討する。

---

#### 6.26.5 Content-Disposition

`Content-Disposition`の
ファイル名は、
システム側で定義した
固定値を使用する。

利用者入力値を
そのままファイル名へ
埋め込まない。

概念例：

```http
Content-Disposition: attachment; filename="month-end-holding-values-template.csv"
```

---

#### 6.26.6 エラー情報

想定外エラー時に、
以下の内部情報を
レスポンスへ含めない。

- スタックトレース
- PHP内部エラー
- Laravel内部例外メッセージ
- サーバーファイルパス
- クラス名
- SQL
- データベース接続情報

詳細情報は、
サーバーログへ記録する。

---

### 6.27 ログ

CSV-004では、
API共通ログ方針に従って
必要な情報を記録する。

概念的なログコンテキストは、
以下とする。

```text
requestId
userId
apiId
```

`apiId`は、

```text
CSV-004
```

とする。

---

#### 6.27.1 正常時

正常時は、
必要に応じて
以下を記録する。

```text
requestId
userId
apiId = CSV-004
httpStatus = 200
```

CSVテンプレート本文そのものを
ログへ記録する必要はない。

---

#### 6.27.2 異常時

異常時は、
必要に応じて
以下を記録する。

```text
requestId
userId
apiId
errorCode
httpStatus
```

想定外例外については、
調査に必要な情報を
サーバーログへ記録する。

---

#### 6.27.3 ログへ記録しない情報

以下の情報を
不要にログへ記録しない。

- CSVテンプレート全文
- 他利用者の情報
- データベース接続情報
- 不要なHTTPヘッダー全文

`X-User-Id`については、
API共通ログ方針に従って
操作対象利用者IDとして扱う。

---

### 6.28 テスト観点

CSV-004では、
CSVテンプレートの形式、
利用者コンテキスト、
HTTPレスポンス、
副作用がないことを
中心に確認する。

---

#### 6.28.1 正常系

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

#### 6.28.2 ヘッダー

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

#### 6.28.3 利用者固有データ

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

#### 6.28.4 資産口座0件

操作対象利用者に
資産口座が存在しない場合でも、

```http
200 OK
```

となり、
正常なテンプレートを
取得できることを確認する。

---

#### 6.28.5 保有商品0件

操作対象利用者に
保有商品が存在しない場合でも、
正常なテンプレートを
取得できることを確認する。

---

#### 6.28.6 商品単位資産口座0件

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

#### 6.28.7 X-User-Id未指定

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

#### 6.28.8 X-User-Id形式不正

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

#### 6.28.9 利用者不存在

存在しない
`X-User-Id`を指定した場合に、

```text
404 Not Found
USER_NOT_FOUND
```

となることを確認する。

---

#### 6.28.10 論理削除済み利用者

`users.deleted_at`が
設定されている利用者を指定した場合に、

```text
404 Not Found
USER_NOT_FOUND
```

となることを確認する。

---

#### 6.28.11 エラーレスポンス形式

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

#### 6.28.12 INTERNAL_SERVER_ERROR

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

#### 6.28.13 副作用

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

#### 6.28.14 冪等性

同じ利用者で
CSV-004を複数回実行した場合に、

- 毎回`200 OK`となること
- 毎回同じCSV内容となること
- 業務データが変更されないこと

を確認する。

---

#### 6.28.15 利用者間のテンプレート一致

異なる有効な利用者で
CSV-004を実行した場合でも、
同一のCSVテンプレートが
返却されることを確認する。

利用者固有データが
混入していないことを保証する。

---

#### 6.28.16 CSV-005・CSV-006との整合性

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

### 6.29 Laravel実装方針

CSV-004では、
Action、
UseCase、
CSV Definition、
CSV Generator、
DTO、
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
Template DTO
    ↓
Responder
    ↓
CSVダウンロードレスポンス
```

本APIでは、
利用者固有の業務データを
CSVテンプレートへ出力しない。

そのため、
テンプレート生成のための

- Repository
- Query
- API Resource

は使用しない。

また、

- パスパラメータ
- クエリパラメータ
- リクエストボディ

を使用しないため、
CSV-004専用の
FormRequestも作成しない。

---

#### 6.29.1 Action

HTTPリクエストを受け付け、
商品別月末評価額CSVテンプレート取得UseCaseを呼び出し、
生成結果をResponderへ渡す。

概念例：

```php
final class DownloadMonthEndHoldingValueCsvTemplateAction
{
    public function __invoke(
        DownloadMonthEndHoldingValueCsvTemplateUseCase $useCase,
        MonthEndHoldingValueCsvTemplateResponder $responder,
    ): StreamedResponse {
        $template =
            $useCase->execute();

        return $responder->download(
            $template,
        );
    }
}
```

Actionでは、
以下を行わない。

- CSVヘッダー定義
- CSV文字列生成
- CSVエスケープ処理
- 文字コード変換
- BOM付与
- ファイル名決定
- `Content-Type`設定
- `Content-Disposition`生成
- 資産口座検索
- 保有商品検索
- 商品別月末評価額検索
- 利用者存在確認

Actionは、
UseCaseの呼び出しと
Responderへの受け渡しに
責務を限定する。

---

#### 6.29.2 FormRequest

CSV-004専用の
FormRequestは作成しない。

本APIでは、

```text
パスパラメータなし
クエリパラメータなし
リクエストボディなし
```

であり、
CSV-004固有の
入力バリデーションが存在しないためである。

`X-User-Id`の検証は、
API共通Middlewareで行う。

以下のような
空のFormRequestは作成しない。

```php
final class DownloadMonthEndHoldingValueCsvTemplateRequest
    extends FormRequest
{
}
```

不要なクラスを
追加しない。

---

#### 6.29.3 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、
`X-User-Id`を検証し、
操作対象利用者を特定する。

概念的には、
以下とする。

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

利用者存在確認条件は、
概念的に以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

Action以降では、
利用者存在確認を
再実装しない。

---

#### 6.29.4 UseCase

商品別月末評価額CSVテンプレート取得の
ユースケース処理を担当する。

概念例：

```php
final class DownloadMonthEndHoldingValueCsvTemplateUseCase
{
    public function __construct(
        private readonly
        MonthEndHoldingValueCsvGenerator $csvGenerator,
    ) {
    }

    public function execute():
        MonthEndHoldingValueCsvTemplate
    {
        return new MonthEndHoldingValueCsvTemplate(
            content:
                $this->csvGenerator
                    ->generateTemplate(),

            fileName:
                MonthEndHoldingValueCsvDefinition::FILE_NAME,
        );
    }
}
```

CSVテンプレートの内容は、
操作対象利用者によって
変化しない。

そのため、
UseCaseへ`userId`を
渡す必要はない。

利用者コンテキストの確認自体は、
Action到達前に
Middlewareで完了していることを
前提とする。

---

#### 6.29.5 UseCaseで行わないこと

UseCaseでは、
以下を行わない。

- `users`の再検索
- `asset_accounts`の検索
- `holding_assets`の検索
- `month_end_asset_snapshots`の検索
- `month_end_holding_values`の検索
- `asset_account_available_settings`の検索
- 利用者固有データのCSV出力
- 商品別月末評価額の登録
- 商品別月末評価額の更新
- 月末資産状況の作成
- 月末資産状況の確定

CSVテンプレート生成に
必要な処理だけを行う。

---

#### 6.29.6 Template DTO

CSV生成結果は、
文字列だけを
Responderへ渡すのではなく、
必要な情報をまとめた
専用DTOとして表現する。

概念例：

```php
final readonly class MonthEndHoldingValueCsvTemplate
{
    public function __construct(
        public string $content,
        public string $fileName,
    ) {
    }
}
```

これにより、
UseCaseからResponderへ渡す
テンプレート情報を
明示的に表現できる。

将来的に、

- 文字コード
- BOM有無
- MIME Type

などを
テンプレート単位で管理する必要が
生じた場合にも拡張しやすくなる。

---

#### 6.29.7 CSV Definition

CSV-004、
CSV-005、
CSV-006で使用する
商品別月末評価額CSV仕様は、
共通Definitionへ集約する。

概念例：

```php
final class MonthEndHoldingValueCsvDefinition
{
    public const HEADERS = [
        'target_year_month',
        'asset_account_name',
        'holding_asset_name',
        'value',
    ];

    public const FILE_NAME =
        'month-end-holding-values-template.csv';
}
```

CSV-004だけで
独自のヘッダーを定義しない。

CSV-005、
CSV-006でも
同じDefinitionを使用する。

---

#### 6.29.8 CSVヘッダー

CSVテンプレートでは、
以下のヘッダーを
この順序で出力する。

```text
target_year_month
asset_account_name
holding_asset_name
value
```

生成結果は、
以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

ヘッダー名だけでなく、
ヘッダー順序も
CSV仕様の一部として扱う。

---

#### 6.29.9 CSV Generator

CSV文字列の生成は、
専用Generatorで行う。

概念例：

```php
final class MonthEndHoldingValueCsvGenerator
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
            $result =
                fputcsv(
                    $stream,
                    MonthEndHoldingValueCsvDefinition::HEADERS,
                );

            if ($result === false) {
                throw new CsvGenerationException();
            }

            rewind(
                $stream,
            );

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

Generatorは、
CSV形式の生成に
責務を限定する。

以下を行わない。

- HTTPレスポンス生成
- 利用者検索
- 資産口座検索
- 保有商品検索
- 商品別月末評価額検索
- 商品別月末評価額登録

---

#### 6.29.10 fputcsvの利用

CSV生成には、
PHP標準の`fputcsv()`を使用する。

以下のような
手動連結は
基本的に行わない。

```php
implode(
    ',',
    MonthEndHoldingValueCsvDefinition::HEADERS,
);
```

`fputcsv()`へ統一することで、
CSV生成方法を
他のCSV関連処理と
揃えやすくする。

---

#### 6.29.11 fopen失敗時

`php://temp`の
オープンに失敗した場合は、
そのまま処理を続行しない。

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

最終的には、
API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

---

#### 6.29.12 fputcsv失敗時

`fputcsv()`の戻り値が
`false`の場合は、
CSV生成失敗として扱う。

概念例：

```php
$result =
    fputcsv(
        $stream,
        MonthEndHoldingValueCsvDefinition::HEADERS,
    );

if ($result === false) {
    throw new CsvGenerationException();
}
```

失敗したCSVを
正常なテンプレートとして
返却してはならない。

---

#### 6.29.13 stream_get_contents失敗時

ストリームから
CSV内容を取得できなかった場合も、
CSV生成失敗とする。

概念例：

```php
$csv =
    stream_get_contents(
        $stream,
    );

if ($csv === false) {
    throw new CsvGenerationException();
}
```

空文字への
フォールバックは行わない。

---

#### 6.29.14 リソース解放

ストリームを開いた場合は、
正常・異常にかかわらず
確実に`fclose()`する。

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

#### 6.29.15 文字コード

商品別月末評価額CSVテンプレートは、
UTF-8で生成する。

CSV-005、
CSV-006で受け付ける
文字コードと統一する。

CSV-004だけ
異なる文字コードを
使用しない。

---

#### 6.29.16 BOM

UTF-8 BOMを
付与するかどうかは、
CSV共通仕様として統一する。

BOMを付与する場合は、
Generator内で
明示的に行う。

概念例：

```php
fwrite(
    $stream,
    "\xEF\xBB\xBF",
);
```

CSV-004、
CSV-005、
CSV-006で
BOMの扱いに
不整合を発生させない。

---

#### 6.29.17 改行コード

CSVテンプレートの
改行コードは、
CSV関連APIで統一する。

CSV-004だけ
独自の改行コード仕様を
持たせない。

実行環境による差異を
許容しない場合は、
CSV Generator側で
明示的に制御する。

---

#### 6.29.18 データ行

CSV-004では、
利用者固有の
データ行を出力しない。

生成対象は、
ヘッダー行のみとする。

以下のような
資産口座・保有商品の
検索処理を行わない。

```php
$holdingAssets =
    HoldingAsset::query()
        ->with('assetAccount')
        ->get();
```

また、
以下のような
テンプレート事前展開も行わない。

```php
foreach (
    $holdingAssets
    as $holdingAsset
) {
    fputcsv(
        $stream,
        [
            '',
            $holdingAsset
                ->assetAccount
                ->name,
            $holdingAsset->name,
            '',
        ],
    );
}
```

テンプレートは、
業務データから独立した
固定形式とする。

---

#### 6.29.19 Repository・Query

CSV-004では、
Repositoryおよび
Queryクラスを使用しない。

本APIで必要となる
データベース参照は、
共通Middlewareによる
`users`の存在確認のみである。

そのため、
以下のような
CSV-004専用クラスは作成しない。

```text
MonthEndHoldingValueCsvTemplateQuery
AssetAccountQuery
HoldingAssetQuery
```

不要な抽象化を
追加しない。

---

#### 6.29.20 API Resource

CSV-004の
正常レスポンスは、
JSONではなくCSVファイルである。

そのため、
正常レスポンス用の
Laravel API Resourceは
使用しない。

以下のような
Resourceは作成しない。

```php
final class MonthEndHoldingValueCsvTemplateResource
    extends JsonResource
{
}
```

CSVファイルを
直接HTTPレスポンスとして返却する。

---

#### 6.29.21 Responder

CSVダウンロードレスポンスは、
専用Responderで生成する。

概念例：

```php
final class MonthEndHoldingValueCsvTemplateResponder
{
    public function download(
        MonthEndHoldingValueCsvTemplate $template,
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

Responderでは、
CSV内容そのものを
生成しない。

---

#### 6.29.22 Responderの責務

Responderは、
生成済みCSVテンプレートを
HTTPレスポンスへ変換することに
責務を限定する。

主に以下を扱う。

```text
HTTPステータス
Content-Type
Content-Disposition
レスポンスボディ
```

Responderでは、
以下を行わない。

- データベース検索
- 利用者存在確認
- CSVヘッダー定義
- CSV生成
- CSV仕様判定
- 資産口座検索
- 保有商品検索

---

#### 6.29.23 Content-Type

正常時の
`Content-Type`は、
以下とする。

```http
Content-Type: text/csv; charset=UTF-8
```

CSVファイルを
JSONとして返却しない。

---

#### 6.29.24 Content-Disposition

CSVファイルとして
ダウンロードできるよう、
`Content-Disposition`を設定する。

概念的には、
以下とする。

```http
Content-Disposition: attachment; filename="month-end-holding-values-template.csv"
```

ファイル名は、
CSV Definitionで
一元管理する。

ActionやResponderへ
同じ文字列を
重複定義しない。

---

#### 6.29.25 正常時のJSON Envelope

正常時は、
API共通の
JSON成功Envelopeを使用しない。

以下のような
レスポンスにはしない。

```json
{
  "data": {
    "content": "target_year_month,asset_account_name,holding_asset_name,value"
  }
}
```

CSVファイルを
直接レスポンスとして返却する。

---

#### 6.29.26 エラー時のレスポンス

エラー時は、
CSVではなく
API共通のJSONエラーレスポンスを使用する。

```text
正常時
    → text/csv

エラー時
    → application/json
```

Middlewareまたは
Exception Handlerで発生した例外は、
API共通方針に従って
JSONへ変換する。

---

#### 6.29.27 例外変換

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| CSV生成処理失敗 | `INTERNAL_SERVER_ERROR` |
| その他想定外例外 | `INTERNAL_SERVER_ERROR` |

CSV-004では、
CSV入力内容に関する
業務例外は発生しない。

---

#### 6.29.28 想定外例外

想定外の例外は、
API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、
以下を含めない。

- スタックトレース
- Laravel内部例外メッセージ
- PHP内部エラー
- サーバーファイルパス
- クラス内部情報
- SQL
- データベース接続情報

詳細情報は、
サーバーログへ記録する。

---

#### 6.29.29 トランザクション

CSV-004では、
明示的な
`DB::transaction()`を使用しない。

以下のような実装は
行わない。

```php
DB::transaction(
    function () {
        // CSVテンプレート生成のみ
    },
);
```

業務データを更新しないため、
トランザクションは不要である。

---

#### 6.29.30 ロック

CSV-004では、
行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

CSVテンプレート取得によって、
CSV-005、
CSV-006、
月末資産関連処理を
ブロックしてはならない。

---

#### 6.29.31 N+1問題

CSV-004では、
業務データを取得しないため、
N+1問題は発生しない。

以下のような処理を
実装しない。

```text
資産口座一覧取得
    ↓
資産口座ごとに
保有商品一覧取得
```

CSV生成処理を
固定Definitionだけで
完結させる。

---

#### 6.29.32 キャッシュ

Phase1では、
CSV-004専用の
サーバー側アプリケーションキャッシュを
使用しない。

CSVテンプレートは
ヘッダー1行のみであり、
生成コストが非常に小さいためである。

また、
CSV仕様変更時に
古いテンプレートが
キャッシュされ続ける
複雑性を避ける。

---

#### 6.29.33 CSV-005・CSV-006との共通化

CSV-004、
CSV-005、
CSV-006では、
同一の

```text
MonthEndHoldingValueCsvDefinition
```

を使用する。

概念的には、
以下とする。

```text
CSV-004
テンプレート生成
    ↓
MonthEndHoldingValueCsvDefinition

CSV-005
ヘッダー検証
    ↓
MonthEndHoldingValueCsvDefinition

CSV-006
ヘッダー検証
    ↓
MonthEndHoldingValueCsvDefinition
```

CSVヘッダーを
各APIへ重複定義しない。

---

#### 6.29.34 GeneratorとParserを分離する

CSV-004は、
CSVを生成する側である。

一方、
CSV-005およびCSV-006は、
アップロードされたCSVを
解析する側である。

そのため、

```text
MonthEndHoldingValueCsvGenerator
MonthEndHoldingValueCsvParser
```

は分離する。

GeneratorとParserを
1つのクラスへ
無理に統合しない。

共通化する対象は、
主に以下とする。

```text
ヘッダー
文字コード
BOM方針
改行コード
CSV仕様
```

---

#### 6.29.35 ログ

CSV-004では、
API共通ログ方針に従う。

必要に応じて、
以下の情報を
ログコンテキストへ設定する。

```text
requestId
userId
apiId
```

`apiId`は、

```text
CSV-004
```

とする。

CSVテンプレート本文を
ログへ出力する必要はない。

---

#### 6.29.36 テスト実装方針

Laravel側では、
Feature Testを中心として
CSV-004のAPI契約を確認する。

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
- CSVファイル名
- CSVヘッダー
- CSVヘッダー順序
- データ行が存在しないこと
- 利用者固有データが含まれないこと
- 内部IDが含まれないこと
- JSON成功Envelopeを使用しないこと
- 業務データを検索しない構成であること
- 業務データが更新されないこと
- 冪等性

---

#### 6.29.37 CSV DefinitionのUnit Test

CSV Definitionについて、
以下のヘッダーが
正しい順序で定義されていることを
確認する。

```php
[
    'target_year_month',
    'asset_account_name',
    'holding_asset_name',
    'value',
]
```

ファイル名についても、
仕様どおりであることを確認する。

```text
month-end-holding-values-template.csv
```

---

#### 6.29.38 CSV GeneratorのUnit Test

CSV Generatorについて、
生成結果が
CSV共通仕様を満たすことを確認する。

主に以下を確認する。

- ヘッダーが存在すること
- ヘッダー順序が正しいこと
- データ行が存在しないこと
- UTF-8で生成されること
- BOMが仕様どおりであること
- 改行コードが仕様どおりであること
- 正常なCSVとして生成されること

期待する内容は、
以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

---

#### 6.29.39 CSV Generator異常系のUnit Test

CSV Generatorで
ストリーム生成や
CSV出力に失敗した場合に、
正常な空CSVを返却せず
例外となることを確認する。

概念的には、

```text
CSV生成失敗
    ↓
CsvGenerationException
    ↓
API共通Exception Handler
    ↓
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

となることを確認する。

---

#### 6.29.40 UseCaseのUnit Test

UseCaseについて、
CSV Generatorの生成結果を
Template DTOとして
返却できることを確認する。

概念的には、

```text
MonthEndHoldingValueCsvGenerator
    ↓
CSV文字列

MonthEndHoldingValueCsvDefinition
    ↓
ファイル名

DownloadMonthEndHoldingValueCsvTemplateUseCase
    ↓
MonthEndHoldingValueCsvTemplate
```

となることを確認する。

また、
UseCase実行時に

- `asset_accounts`
- `holding_assets`
- `month_end_holding_values`

などの
業務データ検索を
必要としない構造であることを確認する。

---

#### 6.29.41 ResponderのTest

Responderについて、
生成済みTemplate DTOから
正しいHTTPレスポンスを
生成できることを確認する。

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

とならないことを確認する。

---

#### 6.29.42 CSV-005・CSV-006との整合性テスト

CSV-004、
CSV-005、
CSV-006で
同一CSV Definitionを
使用することを基本とする。

少なくとも、
以下の整合性を
自動テストで確認する。

```text
CSV-004
生成ヘッダー
    =
MonthEndHoldingValueCsvDefinition::HEADERS
```

```text
CSV-005
期待ヘッダー
    =
MonthEndHoldingValueCsvDefinition::HEADERS
```

```text
CSV-006
期待ヘッダー
    =
MonthEndHoldingValueCsvDefinition::HEADERS
```

これにより、
CSV-004で取得した
公式テンプレートを使用したCSVが、
CSV-005またはCSV-006で
ヘッダー不正となることを防止する。

---

### 6.30 React・TypeScriptでの利用

CSV-004は、
商品別月末評価額CSVを作成するための
公式テンプレートを
フロントエンドから取得する際に使用する。

利用者は、
取得したCSVテンプレートへ

- 対象年月
- 資産口座名
- 保有商品名
- 商品別月末評価額

を入力し、
CSV-005 商品別月末評価額CSVプレビュー、
CSV-006 商品別月末評価額CSV登録で使用する。

概念的な利用フローは、
以下とする。

```text
CSV-004
テンプレート取得
    ↓
CSVファイル保存
    ↓
利用者がCSV編集
    ↓
CSV-005
プレビュー
    ↓
CSV-006
登録
```

CSV-004は、
JSON APIではなく
CSVファイルを返却するため、
通常のJSONレスポンス処理とは
分離して扱う。

---

#### 6.30.1 TypeScript型

CSV-004の正常レスポンスは、
JSONではなくCSVファイルである。

そのため、
正常レスポンス用の
業務DTO型は定義しない。

API Clientでは、
レスポンスを

```ts
Blob
```

として扱う。

概念例：

```ts
export type MonthEndHoldingValueCsvTemplate =
  Blob;
```

ただし、
単純に`Blob`を返却する場合は、
専用type aliasを
作成しなくてもよい。

---

#### 6.30.2 API Client

CSVテンプレート取得は、
通常のJSON取得APIとは分けて
専用関数として定義する。

概念例：

```ts
export const downloadMonthEndHoldingValueCsvTemplate =
  async (): Promise<Blob> => {
    const response =
      await apiClient.get<Blob>(
        '/api/v1/month-end-holding-values/csv-template',
        {
          responseType: 'blob',
          headers: {
            Accept: 'text/csv',
          },
        },
      );

    return response.data;
  };
```

`responseType`は、

```text
blob
```

とする。

CSVレスポンスを
JSONとして解析しない。

---

#### 6.30.3 X-User-Id

`X-User-Id`は、
他のAPIと同様に
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

CSV-004専用処理で
利用者IDを
URLやクエリパラメータへ
追加しない。

---

#### 6.30.4 responseType

CSV-004では、
正常レスポンスを
`Blob`として取得する。

以下のように、
JSONを前提とした
API共通型を使用しない。

```ts
apiClient.get<
  ApiResponse<SomeType>
>(...);
```

正常時は、

```text
text/csv
```

であるため、
JSON成功Envelopeは存在しない。

---

#### 6.30.5 テンプレートダウンロード

取得した`Blob`から
Object URLを生成し、
ブラウザのダウンロード処理を行う。

概念例：

```ts
export const saveBlobAsFile =
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

    anchor.href =
      url;

    anchor.download =
      fileName;

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

CSVテンプレート本文を
React Stateへ
文字列として保持する必要はない。

---

#### 6.30.6 ファイル名

Phase1では、
フロントエンド側でも
以下のファイル名を
使用してよい。

```text
month-end-holding-values-template.csv
```

概念例：

```ts
const FILE_NAME =
  'month-end-holding-values-template.csv';
```

ただし、
将来的に
`Content-Disposition`から
ファイル名を取得する
共通処理を実装する場合は、
そちらへ統一してよい。

---

#### 6.30.7 Content-Dispositionからのファイル名取得

サーバーが
`Content-Disposition`へ
ファイル名を設定しているため、
必要に応じて
レスポンスヘッダーから
ファイル名を取得してよい。

ただし、
Phase1では
固定ファイル名で十分であれば、
無理に解析処理を追加しない。

複雑な

```text
filename*
RFC 5987
```

対応などは、
必要になった時点で
共通化する。

---

#### 6.30.8 Mutationとして扱う

CSV-004は
業務データを更新しないが、
利用者が明示的に
「テンプレートを取得する」
操作によって実行する。

TanStack Queryを使用する場合は、
Mutationとして扱ってよい。

概念例：

```ts
export const useDownloadMonthEndHoldingValueCsvTemplate =
  () =>
    useMutation({
      mutationFn:
        downloadMonthEndHoldingValueCsvTemplate,
    });
```

CSVファイル取得を
画面表示時に
自動実行する必要はない。

---

#### 6.30.9 テンプレート取得ボタン

概念例：

```tsx
<button
  type="button"
  disabled={
    downloadMutation.isPending
  }
  onClick={
    handleDownloadTemplate
  }
>
  {downloadMutation.isPending
    ? '取得中...'
    : 'CSVテンプレートを取得'}
</button>
```

取得処理中は、
意図しない連続操作を避けるため
ボタンを無効化してよい。

ただし、
CSV-004自体は冪等であるため、
複数回取得されても
業務上の問題はない。

---

#### 6.30.10 テンプレート取得処理

概念例：

```ts
const downloadMutation =
  useDownloadMonthEndHoldingValueCsvTemplate();

const handleDownloadTemplate =
  (): void => {
    downloadMutation.mutate(
      undefined,
      {
        onSuccess:
          (blob) => {
            saveBlobAsFile(
              blob,
              'month-end-holding-values-template.csv',
            );
          },
      },
    );
  };
```

正常時は、
CSVファイルとして
保存処理を行う。

---

#### 6.30.11 CSV内容をReactで生成しない

商品別月末評価額CSVの
正式なテンプレートは、
CSV-004から取得する。

React側で、
以下のような
CSVヘッダーを
独自生成しない。

```ts
const headers = [
  'target_year_month',
  'asset_account_name',
  'holding_asset_name',
  'value',
];
```

正式なCSV仕様を
フロントエンドと
バックエンドで
二重管理しない。

---

#### 6.30.12 利用者固有データを追加しない

CSV-004で取得した
テンプレートへ、
React側で

- 資産口座一覧
- 保有商品一覧
- 商品別月末評価額

を自動追記して
別テンプレートを生成しない。

CSV-004で返却された
固定テンプレートを
そのまま利用者へ提供する。

---

#### 6.30.13 CSV-005への利用

利用者が
CSVテンプレートを編集した後は、
CSV-005へ
アップロードして
内容をプレビューする。

概念的には、

```text
CSV-004
テンプレート取得
    ↓
編集済みCSV
    ↓
File
    ↓
CSV-005
```

となる。

CSV-004の
レスポンスBlob自体を
CSV-005へ直接送信する
必要はない。

利用者が編集して
選択した`File`を
CSV-005へ送信する。

---

#### 6.30.14 CSV-006への利用

CSV-005で

```text
canImport = true
```

となった場合は、
同一のCSVファイルを
CSV-006へ送信する。

```text
CSV-004
テンプレート取得
    ↓
利用者編集
    ↓
CSV-005
プレビュー
    ↓
CSV-006
登録
```

CSV-004は、
この一連のCSVインポートフローの
入口として使用する。

---

#### 6.30.15 正常時のContent-Type

正常時は、

```text
text/csv
```

を受信する。

フロントエンドでは、
正常時のレスポンスを
JSONとして処理しない。

---

#### 6.30.16 エラー時のContent-Type

CSV-004では、
正常時はCSV、
エラー時はJSONとなる。

```text
成功
    → text/csv

失敗
    → application/json
```

Axiosなどで
`responseType: 'blob'`を使用すると、
エラー時のJSONも
`Blob`として受け取る可能性がある。

そのため、
ファイルダウンロードAPI用の
共通エラー変換処理を
用意してよい。

---

#### 6.30.17 Blobエラーの扱い

例えば、
Axiosで
`responseType: 'blob'`を指定している場合、
エラーレスポンスも
Blobになる可能性がある。

必要に応じて、
`Content-Type`を確認し、

```text
application/json
```

の場合は
BlobをJSONへ変換する。

概念例：

```ts
const contentType =
  error.response?.headers[
    'content-type'
  ];

if (
  contentType?.includes(
    'application/json',
  )
) {
  const text =
    await error.response.data.text();

  const apiError =
    JSON.parse(
      text,
    );

  // 共通エラー処理
}
```

この処理は、
CSV-001など
他のダウンロードAPIでも
必要となるため、
可能であれば共通化する。

---

#### 6.30.18 利用者関連エラー

以下のエラーは、
API共通方針に従って処理する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
```

CSV-004専用画面だけで
独自の利用者エラー処理を
作成しない。

---

#### 6.30.19 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`の場合は、
CSVテンプレートを
取得できなかったことを
利用者へ表示する。

例えば、

```text
CSVテンプレートを取得できませんでした。
時間をおいて再度お試しください。
```

などの
共通サーバーエラー表示を使用する。

エラー時に
空のCSVファイルを
保存させない。

---

#### 6.30.20 エラー時はダウンロードしない

API呼び出しが失敗した場合は、
取得したレスポンスを
CSVファイルとして
保存しない。

特に、
JSONエラーレスポンスを

```text
month-end-holding-values-template.csv
```

として
誤って保存しないようにする。

---

#### 6.30.21 Object URLの解放

`URL.createObjectURL()`を
使用した場合は、
ダウンロード操作後に

```ts
URL.revokeObjectURL(
  url,
);
```

を実行する。

不要なObject URLを
ブラウザ上に残さない。

---

#### 6.30.22 キャッシュ

CSV-004の
テンプレート内容は固定であるが、
Phase1では
フロントエンド側で
特別なキャッシュ管理を
行う必要はない。

利用者が
テンプレート取得操作を行うたびに
CSV-004を実行してよい。

---

#### 6.30.23 冪等性

CSV-004は
冪等なGET APIであるため、
利用者が複数回
テンプレート取得ボタンを押しても
業務データへ影響しない。

フロントエンドでは、
重複登録防止のような
厳密な排他制御は不要である。

---

#### 6.30.24 React側でCSV仕様を検証しない

CSV-004は
テンプレート取得APIであり、
CSV入力値を検証するAPIではない。

React側で
テンプレート取得時に

- `target_year_month`
- `asset_account_name`
- `holding_asset_name`
- `value`

の入力ルールを
検証する必要はない。

利用者が編集したCSVの
正式な検証は、
CSV-005およびCSV-006で行う。

---

### 6.31 設計上の補足

#### 6.31.1 GETを採用する理由

CSV-004は、
固定されたCSVテンプレートを
取得するだけであり、
業務データを変更しない。

そのため、
HTTPメソッドには
`GET`を採用する。

---

#### 6.31.2 JSONではなくCSVを直接返す理由

本APIの目的は、
利用者がそのまま編集できる
CSVファイルを提供することである。

JSONで

```json
{
  "headers": [
    "target_year_month",
    "asset_account_name",
    "holding_asset_name",
    "value"
  ]
}
```

を返却し、
React側でCSVへ変換する方式は
採用しない。

CSV生成責務を
バックエンドへ集約する。

---

#### 6.31.3 利用者固有データを含めない理由

CSV-004は、
テンプレート取得APIであり、
業務データのエクスポートAPIではない。

利用者固有データを
テンプレートへ含めると、

- 資産口座検索
- 保有商品検索
- 対象年月判定
- 利用者ごとのテンプレート差異

が発生し、
責務が増える。

Phase1では、
固定形式のテンプレートに限定する。

---

#### 6.31.4 資産口座名をテンプレートへ展開しない理由

登録済み資産口座名を
テンプレートへ事前展開すると、
CSV取得時点と
CSV登録時点で
資産口座状態が変化する可能性がある。

また、
商品単位CSVでは
資産口座と保有商品の
組み合わせも必要になる。

そのため、
CSV-004では
固定ヘッダーのみを返却し、
正式な存在確認は
CSV-005・CSV-006で行う。

---

#### 6.31.5 保有商品名をテンプレートへ展開しない理由

保有商品一覧を
CSVテンプレートへ出力すると、
CSV-004が
業務データ取得APIとしての
責務を持つことになる。

Phase1では、
CSVテンプレート生成と
業務データ参照を分離するため、
保有商品名を
自動出力しない。

---

#### 6.31.6 内部IDをCSVへ含めない理由

利用者にとって、
`asset_account_id`や
`holding_asset_id`は
意味を理解しにくい内部値である。

CSVでは、

```text
asset_account_name
holding_asset_name
```

を使用する。

バックエンド側で
操作対象利用者との関連を確認しながら
内部IDへ変換する。

---

#### 6.31.7 CSV-004・CSV-005・CSV-006でDefinitionを共通化する理由

テンプレート生成と
CSV受付側で
ヘッダー定義を別々に持つと、

```text
CSV-004
4列

CSV-005
別の4列

CSV-006
さらに別定義
```

という不整合が発生する可能性がある。

そのため、
共通の

```text
MonthEndHoldingValueCsvDefinition
```

へ集約する。

---

#### 6.31.8 GeneratorとParserを分離する理由

CSV-004は
CSVを生成する。

CSV-005・CSV-006は
CSVを解析する。

責務が異なるため、

```text
Generator
    → CSV出力

Parser
    → CSV入力
```

として分離する。

共通Definitionだけを
共有する。

---

#### 6.31.9 FormRequestを作成しない理由

CSV-004には、

- パスパラメータ
- クエリパラメータ
- リクエストボディ

が存在しない。

CSV-004専用の
入力バリデーションがないため、
空のFormRequestを作成しない。

利用者コンテキストは
共通Middlewareが担当する。

---

#### 6.31.10 API Resourceを使用しない理由

Laravel API Resourceは、
JSONレスポンス形式へ
データを変換する用途に適している。

CSV-004の正常レスポンスは
CSVファイルであるため、
API Resourceを経由させない。

CSVファイル用Responderから
直接レスポンスを生成する。

---

#### 6.31.11 Repository・Queryを使用しない理由

CSV-004では、
テンプレート生成のための
業務データ検索を行わない。

そのため、
CSV-004専用の

```text
Repository
Query
```

を作成しない。

不要な層を追加せず、
テンプレート生成に必要な
最小構成とする。

---

#### 6.31.12 Blobとして扱う理由

ブラウザで
CSVファイルとして保存するため、
フロントエンドでは
レスポンスを`Blob`として扱う。

CSV本文を
通常のJSONレスポンスとして
扱わない。

---

#### 6.31.13 ダウンロードAPIのエラー処理を共通化する理由

CSV-004のように、

```text
成功時
    → CSV

失敗時
    → JSON
```

となるAPIでは、
通常のJSON APIと
エラー処理方法が異なる場合がある。

CSV-001など
同様のダウンロードAPIでも
同じ問題が発生するため、
Blobレスポンスから
JSONエラーを解析する処理は
共通化してよい。

---

#### 6.31.14 キャッシュを使用しない理由

テンプレート生成コストは
非常に小さい。

Phase1で
キャッシュを導入すると、

- CSV仕様変更時の無効化
- APIバージョンとの対応
- キャッシュ制御

が追加で必要になる。

そのため、
単純な都度生成を採用する。

---

#### 6.31.15 Idempotency-Keyを使用しない理由

CSV-004は
副作用のないGET APIであり、
同じリクエストを
複数回実行しても
業務データが変化しない。

そのため、

```text
Idempotency-Key
```

は不要である。

---

### 6.32 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [CSVインポートAPI詳細](./csv-imports.md)
- [資産口座API詳細](./asset-accounts.md)
- [保有商品API詳細](./holding-assets.md)
- [月末資産状況API詳細](./month-end-asset-snapshots.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)

---

## 7. CSV-005 商品別月末評価額CSVプレビュー

### 7.1 概要

操作対象となる利用者について、
アップロードされた
商品別月末評価額CSVの内容を解析・検証し、
CSV-006 商品別月末評価額CSV登録を
実行する前に、
登録予定内容とエラー内容を返却する。

本APIでは、
CSVファイルの内容を検証するが、
商品別月末評価額の登録は行わない。

主に以下を確認する。

- CSVファイル形式
- CSVヘッダー
- 対象年月
- 資産口座名
- 保有商品名
- 商品別月末評価額
- 1ファイル内の対象年月統一
- 資産口座の存在
- 資産口座の利用者境界
- 残高記録単位
- 保有商品の存在
- 保有商品と資産口座の関連
- 対象年月時点での保有商品の有効性
- 月末資産状況の状態
- 既存商品別月末評価額との重複
- CSV内の重複

検証結果として、
CSV全体および
各行のエラーを返却し、
CSV-006による登録が
可能かどうかを

```text
canImport
```

で返却する。

概念的な処理は、
以下とする。

```text
CSVファイル受信
    ↓
ファイル検証
    ↓
CSVヘッダー検証
    ↓
CSV行解析
    ↓
入力値検証
    ↓
対象年月特定
    ↓
CSV内重複確認
    ↓
資産口座・保有商品確認
    ↓
月末資産状況確認
    ↓
既存商品別月末評価額確認
    ↓
エラー集約
    ↓
canImport判定
    ↓
プレビュー結果返却
```

本APIでは、
検証のために
データベースを参照するが、
業務データは更新しない。

---

### 7.2 ユースケース

利用者は、
CSV-004 商品別月末評価額CSVテンプレート取得で
取得したテンプレートへ
商品別月末評価額を入力し、
本APIへアップロードする。

基本的な利用フローは、
以下とする。

```text
CSV-004
商品別月末評価額CSVテンプレート取得
    ↓
利用者がCSVへ入力
    ↓
CSV-005
商品別月末評価額CSVプレビュー
    ↓
canImport確認
    ├─ false
    │     ↓
    │   CSV修正
    │     ↓
    │   再プレビュー
    │
    └─ true
          ↓
       CSV-006
       商品別月末評価額CSV登録
```

CSV-005では、
CSV-006で実行する
登録可否判定と
可能な限り同じルールを使用する。

ただし、
CSV-005で

```text
canImport = true
```

となった場合でも、
CSV-006実行時点で

- 月末資産状況が確定された
- 商品別月末評価額が別処理で登録された
- 資産口座または保有商品の状態が変更された

場合は、
CSV-006で
登録できない可能性がある。

そのため、
CSV-006では
同じCSVファイルを再送し、
最新状態で再検証する。

---

### 7.3 エンドポイント

```http
POST /api/v1/month-end-holding-values/imports/preview
```

---

### 7.4 HTTPメソッド

```http
POST
```

本APIは、
業務データを更新しない。

ただし、
CSVファイルを
リクエストボディとして送信し、
サーバー側で解析・検証するため、
`POST`を使用する。

本APIの実行によって、
以下の業務データは
登録・更新・削除しない。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

CSVプレビュー結果も
データベースへ保存しない。

---

### 7.5 認証・利用者の扱い

Phase1では、
認証機能を実装しない。

操作対象となる利用者は、
`X-User-Id`
リクエストヘッダーで指定する。

例：

```http
X-User-Id: 1
```

利用者IDは、
以下では受け付けない。

- パスパラメータ
- クエリパラメータ
- CSV列
- `multipart/form-data`の業務項目

商品別月末評価額CSVには、

```text
user_id
userId
```

を含めない。

CSV内では、
資産口座および保有商品を

```text
asset_account_name
holding_asset_name
```

によって指定する。

バックエンドでは、
操作対象利用者との関連を確認したうえで、
内部IDへ変換する。

---

#### 7.5.1 X-User-Id

操作対象利用者は、
以下のヘッダーから特定する。

```http
X-User-Id: 1
```

`X-User-Id`は、
API共通Middlewareで検証する。

Action以降では、
検証済みの
利用者コンテキストを使用する。

---

#### 7.5.2 利用者存在確認

指定された利用者が
存在し、
論理削除されていないことを確認する。

概念的な条件は、
以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者が存在しない場合は、
CSVファイルの解析や
業務データ検索へ進まない。

---

#### 7.5.3 資産口座の利用者境界

CSVの
`asset_account_name`から
対象資産口座を特定する際は、
必ず操作対象利用者によって
絞り込む。

概念的な検索条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = CSVのasset_account_name

AND

asset_accounts.deleted_at
    IS NULL
```

他の利用者に
同名資産口座が存在していても、
CSV-005の検証対象として
使用してはならない。

例えば、

```text
User A
証券口座

User B
証券口座
```

が存在する状態で、
User Aを操作対象として
CSVに

```text
asset_account_name = 証券口座
```

が指定されている場合は、
User Aの資産口座だけを
対象とする。

---

#### 7.5.4 保有商品の利用者境界

CSVの
`holding_asset_name`から
対象保有商品を特定する場合は、
資産口座との関連を含めて
利用者境界を保証する。

概念的には、
以下とする。

```text
holding_assets
    ↓
asset_accounts
    ↓
asset_accounts.user_id
        = 操作対象利用者ID
```

検索条件は、
概念的に以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = CSVのasset_account_name

AND

holding_assets.asset_account_id
    = asset_accounts.id

AND

holding_assets.name
    = CSVのholding_asset_name

AND

holding_assets.deleted_at
    IS NULL
```

保有商品名だけを条件として
対象商品を特定しない。

---

#### 7.5.5 同名保有商品の扱い

異なる資産口座に
同じ名前の保有商品が
存在する可能性がある。

例えば、

```text
証券口座A
    全世界株式

証券口座B
    全世界株式
```

が存在する場合、
CSVに

```text
asset_account_name = 証券口座A
holding_asset_name = 全世界株式
```

と指定されたときは、
証券口座Aに属する
`全世界株式`のみを対象とする。

`holding_asset_name`だけで
保有商品を検索してはならない。

---

#### 7.5.6 他利用者の同名保有商品

操作対象利用者には
該当する保有商品が存在せず、
他の利用者にのみ
同名保有商品が存在する場合でも、
正常とは判定しない。

例えば、

```text
User A
証券口座あり
全世界株式なし

User B
証券口座あり
全世界株式あり
```

の場合、
User Aとして

```text
asset_account_name = 証券口座
holding_asset_name = 全世界株式
```

を指定した場合は、

```text
HOLDING_ASSET_NOT_FOUND
```

相当の
プレビューエラーとして扱う。

---

#### 7.5.7 残高記録単位

CSV-005は、
商品別月末評価額CSVの
プレビューAPIである。

そのため、
対象資産口座の

```text
asset_accounts.balance_recording_unit
```

が
商品単位であることを確認する。

口座単位の資産口座は、
商品別月末評価額CSVの
対象としてはならない。

他利用者の資産口座の
残高記録単位を
判定へ使用しない。

---

#### 7.5.8 保有商品と資産口座の関連

CSVで指定された
`holding_asset_name`が、
同じ行で指定された
`asset_account_name`の
資産口座に属していることを確認する。

例えば、

```text
証券口座A
    全世界株式

証券口座B
    S&P500
```

という状態で、

```text
asset_account_name
    = 証券口座A

holding_asset_name
    = S&P500
```

と指定されても、
証券口座Bの
`S&P500`を使用しない。

指定された資産口座内に
対象保有商品が存在しないものとして
扱う。

---

#### 7.5.9 月末資産状況の利用者境界

CSVの`target_year_month`に対応する
月末資産状況を確認する際も、
操作対象利用者によって
絞り込む。

概念的には、

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = CSVのtarget_year_month
```

とする。

他の利用者に
同一対象年月の
月末資産状況が存在していても、
CSV-005の
確定状態判定へ
使用してはならない。

---

#### 7.5.10 月末資産状況が存在しない場合

CSV-006で
対象年月の
`month_end_asset_snapshots`を
必要に応じて作成する仕様である場合、
CSV-005では
snapshot不存在だけを理由として
登録不可にはしない。

概念的には、

```text
snapshot不存在
    ↓
CSV-006で作成可能
    ↓
CSV-005では
他条件が正常なら登録可能
```

とする。

ただし、
CSV-005では
実際にsnapshotを作成しない。

---

#### 7.5.11 確定済み月末資産状況

操作対象利用者の
対象年月に対応する
月末資産状況が存在し、

```text
confirmed = true
```

の場合は、
商品別月末評価額を
追加登録できない。

この場合は、
CSV全体の
プレビューエラーとして扱い、

```text
canImport = false
```

とする。

他利用者の
同一対象年月snapshotが
確定済みでも、
操作対象利用者の
登録可否には影響させない。

---

#### 7.5.12 既存商品別月末評価額の利用者境界

既存の
`month_end_holding_values`を
確認する場合は、
対象snapshotおよび
保有商品の双方について
操作対象利用者との関連を保証する。

概念的には、

```text
month_end_holding_values
    ↓
month_end_asset_snapshots
    ↓
month_end_asset_snapshots.user_id
        = 操作対象利用者ID
```

および、

```text
month_end_holding_values
    ↓
holding_assets
    ↓
asset_accounts
    ↓
asset_accounts.user_id
        = 操作対象利用者ID
```

によって
利用者境界を保証する。

---

#### 7.5.13 他利用者の既存評価額

他の利用者に

- 同一対象年月
- 同名資産口座
- 同名保有商品
- 商品別月末評価額

が存在していても、
CSV-005の
重複判定へ影響させない。

操作対象利用者に
既存の商品別月末評価額が
存在する場合だけを
重複として扱う。

---

#### 7.5.14 CSV-004の利用者コンテキストを引き継がない

CSV-004で
テンプレートを取得した際の
利用者コンテキストを、
CSV-005で
サーバー側に保持・引き継がない。

CSV-005のリクエストでも、
改めて

```http
X-User-Id
```

を指定する。

CSVテンプレート自体には
利用者情報を含めないため、
CSV-005実行時の
利用者コンテキストが
正式な検証基準となる。

---

#### 7.5.15 CSV-006への利用者コンテキスト引き継ぎを前提としない

CSV-005で
User Aとして
プレビューしたCSVを、
CSV-006でUser Bとして
送信することは
技術的には可能である。

CSV-006では、
User Aのプレビュー結果を
利用者境界の保証として
使用してはならない。

CSV-006でも
`X-User-Id`を基準として
改めて全件検証する。

---

#### 7.5.16 1リクエスト1利用者

1回のCSV-005リクエストでは、
`X-User-Id`で指定された
1利用者のデータだけを扱う。

CSVファイルに
利用者を切り替えるための
列を持たせない。

以下のような列は
CSV仕様に含めない。

```text
user_id
user_name
```

これにより、
商品別月末評価額CSVの
利用者境界を明確にする。

---

#### 7.5.17 X-User-Id未指定

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

として扱う。

CSVファイルの解析へ
進まない。

---

#### 7.5.18 X-User-Id形式不正

`X-User-Id`の形式が
不正な場合は、

```text
INVALID_USER_ID
```

として扱う。

例えば、

```text
0
-1
abc
1.5
```

などを
有効な利用者IDとして
扱わない。

---

#### 7.5.19 利用者不存在

指定された利用者が
存在しない場合、
または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

として扱う。

CSV解析、
資産口座検索、
保有商品検索、
月末資産状況検索へ
進まない。

---

#### 7.5.20 プレビューでは業務データを更新しない

利用者境界の検証に成功しても、
CSV-005では
以下を更新しない。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

CSV-005の責務は、
指定された利用者について
CSV-006で登録可能かを
確認することに限定する。

---

### 7.6 パスパラメータ

本APIでは、
パスパラメータを使用しない。

エンドポイントは、
以下とする。

```http
POST /api/v1/month-end-holding-values/imports/preview
```

以下の情報を
URLへ含めない。

- `userId`
- `targetYearMonth`
- `assetAccountId`
- `holdingAssetId`
- `snapshotId`

操作対象利用者は、
`X-User-Id`
リクエストヘッダーから特定する。

対象年月、
資産口座名、
保有商品名、
商品別月末評価額は、
アップロードされたCSVから取得する。

---

### 7.7 クエリパラメータ

本APIでは、
クエリパラメータを使用しない。

以下のような
クエリパラメータは受け付けない。

- `targetYearMonth`
- `assetAccountId`
- `holdingAssetId`
- `confirmed`
- `overwrite`
- `force`
- `preview`

本APIは、
商品別月末評価額CSVの
プレビューに責務を限定する。

登録可否や
上書き可否を
クエリパラメータによって
切り替えない。

---

### 7.8 リクエストヘッダー

本APIでは、
以下のリクエストヘッダーを使用する。

| ヘッダー | 必須 | 内容 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
POST /api/v1/month-end-holding-values/imports/preview
Accept: application/json
X-User-Id: 1
```

CSVファイルは、
`multipart/form-data`で送信する。

`Content-Type`は、
HTTPクライアントによって
boundary付きで設定する。

概念例：

```http
Content-Type: multipart/form-data; boundary=...
```

クライアント側で
boundaryを固定値として
手動設定しない。

---

### 7.9 リクエストボディ

本APIでは、
`multipart/form-data`形式で
CSVファイルを受け取る。

リクエスト項目は、
以下とする。

| 項目 | 型 | 必須 | 内容 |
|---|---|:---:|---|
| `file` | file | ○ | プレビュー対象となる商品別月末評価額CSV |

概念例：

```text
multipart/form-data

file:
  month-end-holding-values.csv
```

CSVファイル以外の
業務項目は
リクエストボディで受け付けない。

以下のような項目は、
`multipart/form-data`へ
追加しない。

- `userId`
- `targetYearMonth`
- `assetAccountId`
- `holdingAssetId`
- `value`
- `confirmed`

操作対象利用者は、
`X-User-Id`から取得する。

その他の業務項目は、
CSV内容から取得する。

---

### 7.10 CSVファイル仕様

商品別月末評価額CSVでは、
以下のヘッダーを使用する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

ヘッダー順序も
CSV仕様の一部とする。

CSV-004
商品別月末評価額CSVテンプレート取得で
提供する形式と同一とする。

CSV入力例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

1つのCSVファイルでは、
1つの`target_year_month`のみを扱う。

---

### 7.11 バリデーション

CSV-005では、
CSV-006の商品別月末評価額CSV登録と
可能な限り同じ
CSV解析・登録可否検証を行う。

ただし、
CSV-005では
業務データを登録しない。

検証は、
概念的に以下の段階で行う。

```text
リクエスト検証
    ↓
CSVファイル・構造検証
    ↓
CSV行入力値検証
    ↓
CSV全体ルール検証
    ↓
業務データ照合
    ↓
業務ルール検証
    ↓
エラー集約
    ↓
canImport判定
```

CSV構造そのものを
正常に解釈できない場合は
HTTPエラーとする。

一方、
正常に解析できたCSVの
入力・業務エラーは、
原則として
プレビュー結果へ格納し、

```text
200 OK
canImport = false
```

として返却する。

---

#### 7.11.1 file 必須

`file`は、
必須とする。

CSVファイルが
指定されていない場合は、
バリデーションエラーとする。

この場合、
CSV解析処理や
業務データ検索へ進まない。

---

#### 7.11.2 アップロードファイルであること

`file`は、
HTTPアップロードファイルとして
正常に受信できていることを確認する。

文字列や
通常のフォーム値を
CSVファイルとして扱わない。

---

#### 7.11.3 ファイル拡張子

アップロードファイルは、
CSVファイルであることを確認する。

Phase1では、
拡張子として

```text
.csv
```

を受け付ける。

例えば、
以下は不正とする。

```text
.xlsx
.xls
.pdf
```

拡張子だけを
唯一の判定根拠とはせず、
CSVとして正常に
解析可能であることも確認する。

---

#### 7.11.4 ファイルサイズ

CSVファイルサイズには、
CSV共通仕様で定める
上限を適用する。

上限を超える場合は、
CSV解析前に
バリデーションエラーとする。

具体的な上限値は、
CSV共通仕様の
最新定義に従う。

---

#### 7.11.5 空ファイル

CSVファイルが
0バイトの場合、
または有効なヘッダー行を
取得できない場合は、
不正なCSVとして扱う。

プレビュー結果を
生成できるCSVではないため、
後続の業務検証へ進まない。

---

#### 7.11.6 文字コード

CSVファイルの文字コードは、
CSV共通仕様に従う。

Phase1では、
UTF-8を基本とする。

CSV-004で提供する
テンプレートと
同じ文字コードを使用することを
前提とする。

UTF-8 BOMを許可する場合は、
先頭ヘッダーへ
BOMが混入しないよう
解析時に除去する。

---

#### 7.11.7 CSVとして読み取り可能であること

アップロードされたファイルが、
CSVとして正常に
読み取れることを確認する。

例えば、
以下は不正とする。

- CSV構造が破損している
- 引用符が不正
- ヘッダーを取得できない
- 列構造を正常に解釈できない

CSV解析に失敗した場合は、
プレビュー結果ではなく
CSV形式エラーとして扱う。

---

#### 7.11.8 ヘッダー必須

CSVの1行目には、
ヘッダー行が
存在することを必須とする。

期待するヘッダーは、
以下とする。

```text
target_year_month
asset_account_name
holding_asset_name
value
```

---

#### 7.11.9 ヘッダー名

ヘッダーは、
以下と完全一致することを
基本とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

例えば、
以下のような別名は
受け付けない。

```text
targetYearMonth
assetAccountName
holdingAssetName
amount
```

CSV-004のテンプレートを
正式な入力形式とする。

---

#### 7.11.10 ヘッダー順序

ヘッダーは、
以下の順序とする。

```text
1. target_year_month
2. asset_account_name
3. holding_asset_name
4. value
```

Phase1では、
同じヘッダー名でも
順序が異なるCSVは
不正とする。

CSV-004、
CSV-005、
CSV-006で
同一仕様を使用する。

---

#### 7.11.11 余分なヘッダー

定義されていない
余分なCSV列は
受け付けない。

例えば、

```csv
target_year_month,asset_account_name,holding_asset_name,value,memo
```

は不正とする。

以下のような
内部管理項目も
CSV列として受け付けない。

- `user_id`
- `asset_account_id`
- `holding_asset_id`
- `month_end_asset_snapshot_id`
- `month_end_holding_value_id`
- `confirmed`

---

#### 7.11.12 データ行0件

CSVヘッダーは正常だが、
データ行が1件も存在しない場合は、
CSV構造異常として
HTTPエラーにはしない。

CSV全体エラーとして扱い、

```text
canImport = false
```

とする。

概念的には、

```text
CSV解析成功
+
データ行0件
    ↓
200 OK
canImport = false
```

となる。

---

#### 7.11.13 空行

CSV末尾などの
完全な空行は、
CSV共通仕様に従って
無視してよい。

ただし、

```csv
,,,
```

のように
列として存在する行は、
データ行として扱い、
各項目の必須チェックを行う。

---

#### 7.11.14 target_year_month 必須

各データ行の
`target_year_month`は
必須とする。

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,全世界株式,1500000
```

未入力の場合は、
行エラーとして扱う。

---

#### 7.11.15 target_year_month 形式

`target_year_month`は、
以下の形式とする。

```text
YYYY-MM
```

正常例：

```text
2026-01
2026-07
2026-12
```

不正例：

```text
2026-1
26-07
2026/07
202607
2026-00
2026-13
abc
```

形式だけでなく、
年月として
有効な値であることを確認する。

---

#### 7.11.16 1ファイル1対象年月

1つのCSVファイルでは、
すべてのデータ行の
`target_year_month`が
同一であることを必須とする。

正常例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-06,証券口座,全世界株式,1400000
2026-07,証券口座,S&P500,800000
```

複数年月が存在する場合は、
CSV全体エラーとして

```text
canImport = false
```

とする。

---

#### 7.11.17 asset_account_name 必須

各データ行の
`asset_account_name`は
必須とする。

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,,全世界株式,1500000
```

未入力の場合は、
行エラーとして扱う。

---

#### 7.11.18 asset_account_nameの扱い

`asset_account_name`は、
資産口座を特定するための
文字列として扱う。

前後空白の扱いは、
CSV共通仕様または
資産口座名の業務ルールに従う。

入力値を
暗黙的に別名へ変換して
資産口座を特定しない。

---

#### 7.11.19 資産口座存在確認

CSVの
`asset_account_name`に対応する
資産口座が、
操作対象利用者に
存在することを確認する。

概念的な条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = asset_account_name

AND

asset_accounts.deleted_at
    IS NULL
```

該当する資産口座が
存在しない場合は、
行エラーとして扱う。

---

#### 7.11.20 他利用者の同名資産口座

操作対象利用者には
該当資産口座が存在せず、
他利用者にのみ
同名資産口座が存在する場合でも、
正常とは判定しない。

概念的には、

```text
ASSET_ACCOUNT_NOT_FOUND
```

相当の
行エラーとする。

---

#### 7.11.21 残高記録単位

CSV-005は、
商品別月末評価額CSVの
プレビューAPIである。

そのため、
対象資産口座の

```text
asset_accounts.balance_recording_unit
```

が
商品単位であることを確認する。

口座単位の資産口座の場合は、
商品別月末評価額CSVの
対象外として
行エラーを生成する。

---

#### 7.11.22 holding_asset_name 必須

各データ行の
`holding_asset_name`は
必須とする。

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,,1500000
```

未入力の場合は、
行エラーとして扱う。

---

#### 7.11.23 holding_asset_nameの扱い

`holding_asset_name`は、
同一行で指定された
資産口座内の保有商品を
特定するために使用する。

保有商品名だけを
システム全体から検索して
対象商品を決定しない。

---

#### 7.11.24 保有商品存在確認

CSVで指定された

```text
asset_account_name
+
holding_asset_name
```

に対応する
保有商品が存在することを確認する。

概念的な条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = asset_account_name

AND

holding_assets.asset_account_id
    = asset_accounts.id

AND

holding_assets.name
    = holding_asset_name

AND

holding_assets.deleted_at
    IS NULL
```

存在しない場合は、
行エラーとして扱う。

---

#### 7.11.25 他資産口座にのみ同名保有商品が存在する

例えば、

```text
証券口座A
    全世界株式なし

証券口座B
    全世界株式あり
```

の状態で、

```text
asset_account_name
    = 証券口座A

holding_asset_name
    = 全世界株式
```

と指定した場合は、
証券口座Bの商品を
使用しない。

概念的には、

```text
HOLDING_ASSET_NOT_FOUND
```

相当の
行エラーとする。

---

#### 7.11.26 他利用者にのみ同名保有商品が存在する

操作対象利用者には
該当する保有商品が存在せず、
他利用者にのみ
同名保有商品が存在する場合も、

```text
HOLDING_ASSET_NOT_FOUND
```

相当の
行エラーとする。

他利用者の商品を
プレビュー結果へ
紐付けてはならない。

---

#### 7.11.27 対象年月時点での保有商品の有効性

対象保有商品が、
CSVの`target_year_month`時点で
商品別月末評価額の
記録対象として有効であることを確認する。

現在日時を基準に
判定してはならない。

必ず、

```text
CSVのtarget_year_month
```

を基準とする。

対象年月時点で
無効な保有商品は、
行エラーとして扱う。

---

#### 7.11.28 value 必須

各データ行の
`value`は
必須とする。

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,
```

未入力の場合は、
行エラーとして扱う。

---

#### 7.11.29 value 数値形式

`value`は、
日本円の整数として
扱える値であることを確認する。

正常例：

```text
0
1000
1500000
```

不正例：

```text
1,500,000
¥1500000
1500.5
abc
```

CSV上では、
桁区切りや
通貨記号を使用しない。

---

#### 7.11.30 value 整数

`value`は、
整数であることを必須とする。

以下のような
小数値は受け付けない。

```text
1000.5
```

Phase1では、
日本円の整数として管理する。

---

#### 7.11.31 value 0円以上

`value`は、
0以上とする。

正常例：

```text
0
1
1500000
```

不正例：

```text
-1
-100000
```

0円は
有効な商品別月末評価額として扱う。

```text
0円
≠
未入力
```

とする。

---

#### 7.11.32 CSV内重複

同一CSV内で、
同じ対象年月、
同じ資産口座、
同じ保有商品が
複数回指定されていないことを確認する。

概念的な重複キーは、
以下とする。

```text
targetYearMonth
+
assetAccountName
+
holdingAssetName
```

1ファイル1対象年月が
成立している場合は、
実質的には

```text
assetAccountName
+
holdingAssetName
```

単位で
重複確認してよい。

---

#### 7.11.33 CSV内重複時の扱い

例えば、

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,全世界株式,1600000
```

は不正とする。

先勝ち、
後勝ち、
評価額合算などの
暗黙的な解決を行わない。

行エラーとして扱い、

```text
canImport = false
```

とする。

---

#### 7.11.34 月末資産状況の確認

CSVの`target_year_month`について、
操作対象利用者に属する
`month_end_asset_snapshots`を確認する。

概念的な条件は、
以下とする。

```text
user_id
    = 操作対象利用者ID

AND

target_year_month
    = CSVのtarget_year_month
```

他利用者の
同一対象年月snapshotを
使用しない。

---

#### 7.11.35 月末資産状況が存在しない場合

対象年月の
月末資産状況が
存在しない場合でも、
CSV-006で
必要に応じて新規作成できるため、
snapshot不存在だけを理由として
登録不可にはしない。

CSV-005では、
実際にsnapshotを作成しない。

---

#### 7.11.36 確定済み月末資産状況

対象年月の
月末資産状況が存在し、

```text
confirmed = true
```

の場合は、
商品別月末評価額を
登録できない。

CSV全体エラーとして扱い、

```text
canImport = false
```

とする。

---

#### 7.11.37 既存商品別月末評価額

対象年月、
対象保有商品について、
既に
`month_end_holding_values`が
存在するか確認する。

概念的には、

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

の組み合わせで
既存データを確認する。

既存データが存在する場合は、
行エラーとして扱う。

CSV-005では、
既存の`value`を
更新・上書きしない。

---

#### 7.11.38 他利用者の既存商品別月末評価額

他利用者に

- 同じ対象年月
- 同名資産口座
- 同名保有商品
- 商品別月末評価額

が存在していても、
操作対象利用者の
重複判定へ影響させない。

利用者境界を満たす
既存データだけを
重複判定へ使用する。

---

#### 7.11.39 複数エラーの収集

CSV構造が
正常に解析可能な場合は、
可能な範囲で
複数エラーを収集する。

例えば、

```text
2行目
ASSET_ACCOUNT_NOT_FOUND

4行目
HOLDING_ASSET_NOT_FOUND

6行目
INVALID_VALUE
```

などを、
1回のプレビューで
返却できるようにする。

最初の1件だけで
即座に処理を終了しない。

---

#### 7.11.40 canImport

CSV全体エラーまたは
行エラーが
1件でも存在する場合は、

```text
canImport = false
```

とする。

すべての検証に
成功した場合のみ、

```text
canImport = true
```

とする。

フロントエンドが
エラー一覧から
登録可否を再計算することを
前提としない。

---

#### 7.11.41 プレビュー時は登録しない

すべての検証に成功して

```text
canImport = true
```

となった場合でも、
CSV-005では
以下を行わない。

```text
month_end_asset_snapshots INSERT
month_end_holding_values INSERT
```

CSV-005は、
プレビューおよび
登録可否確認だけを行う。

実際の登録は、
CSV-006で行う。

---

#### 7.11.42 CSV-006で再検証する

CSV-005で
`canImport = true`となっても、
CSV-006では
同じ検証を再実行する。

CSV-005とCSV-006の間で、

- `confirmed`が変更された
- 商品別月末評価額が登録された
- 資産口座が変更された
- 保有商品が変更された

可能性があるためである。

CSV-005の結果を
登録権利として扱わない。

---

### 7.12 業務ルール

CSV-005では、
商品別月末評価額CSVの内容を解析・検証し、
CSV-006で登録可能な状態かを
事前確認する。

本APIでは、
商品別月末評価額の
登録・更新・削除を行わない。

CSV-005の役割は、
利用者がCSV-006を実行する前に、

- 登録予定内容
- CSV全体のエラー
- 各行のエラー
- 登録可否

を確認できるようにすることである。

---

#### 7.12.1 CSV-004と同じCSV仕様を使用する

CSV-005で受け付ける
CSV形式は、
CSV-004 商品別月末評価額CSVテンプレート取得で
提供する形式と同一とする。

ヘッダーは、
以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

以下についても、
CSV-004およびCSV-006と
共通仕様を使用する。

- ヘッダー名
- ヘッダー順序
- 文字コード
- BOM
- 改行コード
- 必須項目
- 対象年月形式
- 評価額形式

CSV-004で取得した
正式なテンプレートが、
CSV-005で
形式不正にならないことを保証する。

---

#### 7.12.2 CSV-006と同じ登録可否ルールを使用する

CSV-005とCSV-006では、
可能な限り同じ
CSV解析・業務検証ロジックを使用する。

同一のシステム状態、
同一のCSVであれば、

```text
CSV-005
canImport = true
```

の場合に、

```text
CSV-006
登録可能
```

となることを基本とする。

ただし、
CSV-005からCSV-006までの間に
業務データが変更された場合は、
CSV-006で登録不可となることを許容する。

---

#### 7.12.3 1ファイル1対象年月

1つのCSVファイルでは、
1つの対象年月のみを扱う。

すべてのデータ行について、
`target_year_month`が
同一であることを必須とする。

正常例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-06,証券口座,全世界株式,1400000
2026-07,証券口座,S&P500,800000
```

複数年月が混在している場合は、
CSV全体を登録不可とし、

```text
canImport = false
```

とする。

---

#### 7.12.4 資産口座の特定

CSV上では、
資産口座IDを指定しない。

対象資産口座は、

```text
操作対象利用者
+
asset_account_name
```

によって特定する。

概念的な条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = CSVのasset_account_name

AND

asset_accounts.deleted_at
    IS NULL
```

CSVに

```text
asset_account_id
```

を持たせない。

---

#### 7.12.5 保有商品の特定

CSV上では、
保有商品IDを指定しない。

対象保有商品は、

```text
操作対象利用者
+
asset_account_name
+
holding_asset_name
```

によって特定する。

概念的には、

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = CSVのasset_account_name

AND

holding_assets.asset_account_id
    = asset_accounts.id

AND

holding_assets.name
    = CSVのholding_asset_name

AND

holding_assets.deleted_at
    IS NULL
```

とする。

保有商品名だけで
システム全体から
対象商品を特定しない。

---

#### 7.12.6 利用者境界

CSV内容を検証する際は、
以下について
操作対象利用者との
利用者境界を保証する。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

他利用者の

- 同名資産口座
- 同名保有商品
- 同一対象年月snapshot
- 既存商品別月末評価額

を
CSV-005の登録可否判定へ
使用してはならない。

---

#### 7.12.7 資産口座の残高記録単位

商品別月末評価額CSVでは、
商品単位で管理する
資産口座のみを対象とする。

対象資産口座の

```text
asset_accounts.balance_recording_unit
```

が
商品単位であることを確認する。

口座単位の資産口座の場合は、
商品別月末評価額CSVの
対象外とする。

この場合は、
行エラーを生成し、

```text
canImport = false
```

とする。

---

#### 7.12.8 保有商品は指定資産口座に属すること

CSVで指定された
`holding_asset_name`は、
同じ行の
`asset_account_name`で指定された
資産口座に属している必要がある。

例えば、

```text
証券口座A
    全世界株式

証券口座B
    S&P500
```

という状態で、

```text
asset_account_name
    = 証券口座A

holding_asset_name
    = S&P500
```

と指定した場合は、
証券口座Bの商品を
自動的に使用しない。

指定された資産口座に
対象保有商品が存在しないものとして扱う。

---

#### 7.12.9 対象年月時点での保有商品

対象保有商品が、
CSVの`target_year_month`時点で
商品別月末評価額の
記録対象として有効であることを確認する。

現在日時点の状態だけで
判定してはならない。

必ず、

```text
CSVのtarget_year_month
```

を基準として
登録可否を判定する。

対象年月時点で
無効な保有商品は、
行エラーとする。

---

#### 7.12.10 value

`value`は、
対象年月末時点の
保有商品の評価額として扱う。

Phase1では、
日本円の整数とする。

以下を満たす必要がある。

```text
整数
AND
0以上
```

正常例：

```text
0
1000
1500000
```

不正例：

```text
-1
1000.5
abc
¥1000
1,000
```

---

#### 7.12.11 0円

```text
value = 0
```

は、
正常な商品別月末評価額として扱う。

以下のように
0円を未入力扱いしてはならない。

```text
0円
=
未入力
```

ではなく、

```text
0円
≠
未入力
```

とする。

---

#### 7.12.12 CSV内重複

同一CSV内では、
以下の組み合わせを
重複させない。

```text
target_year_month
+
asset_account_name
+
holding_asset_name
```

1ファイル1対象年月であるため、
実質的には、

```text
asset_account_name
+
holding_asset_name
```

で
一意であることを必須とする。

---

#### 7.12.13 CSV内重複を自動解決しない

同一保有商品が
CSV内に複数行存在する場合は、
登録不可とする。

例えば、

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,全世界株式,1600000
```

の場合に、

- 先勝ち
- 後勝ち
- 評価額合算
- 平均値算出

などの
暗黙的な解決を行わない。

---

#### 7.12.14 月末資産状況

CSVの対象年月について、
操作対象利用者に属する
月末資産状況を確認する。

概念的な条件は、
以下とする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = CSVのtarget_year_month
```

他利用者のsnapshotを
使用しない。

---

#### 7.12.15 月末資産状況が存在しない場合

対象年月の
月末資産状況が
存在しない場合は、
CSV-006で必要に応じて
新規作成できるものとする。

そのため、
CSV-005では
snapshot不存在だけを理由として
登録不可にしない。

```text
snapshotなし
    ↓
CSV-006で作成可能
    ↓
CSV-005では
他の条件が正常なら
canImport = true
```

とする。

CSV-005では、
実際にsnapshotを作成しない。

---

#### 7.12.16 既存の未確定月末資産状況

対象年月について
未確定の月末資産状況が
存在する場合は、
そのsnapshotを基準として
既存商品別月末評価額を確認する。

```text
snapshotあり
+
confirmed = false
    ↓
登録可否判定を継続
```

とする。

---

#### 7.12.17 確定済み月末資産状況

対象年月の
月末資産状況が

```text
confirmed = true
```

の場合は、
商品別月末評価額を
追加登録できない。

CSV全体エラーとして扱い、

```text
canImport = false
```

とする。

CSV-005では、
確定状態を変更しない。

---

#### 7.12.18 既存商品別月末評価額

同一snapshot、
同一保有商品について、
既に商品別月末評価額が
存在する場合は、
登録不可とする。

概念的な重複キーは、

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

とする。

既存の`value`を
CSV値で上書きする前提にはしない。

---

#### 7.12.19 既存値と同じ場合も重複とする

既に

```text
value = 1500000
```

が登録されている状態で、
CSVにも

```text
value = 1500000
```

が指定されていても、
登録済みデータとして扱う。

同じ値だから
登録可能とは判定しない。

---

#### 7.12.20 既存値と異なる場合も上書きしない

既存の商品別月末評価額が

```text
1500000
```

で、
CSVに

```text
1600000
```

が指定されていても、
CSV-005では
上書き可能とは判定しない。

CSV-006は
新規一括登録に責務を限定する。

既存商品別月末評価額の変更は、
専用更新APIの責務とする。

---

#### 7.12.21 データ行0件

CSVヘッダーが正常でも、
データ行が存在しない場合は、
登録不可とする。

ただし、
CSV自体は解析可能であるため、
HTTPエラーではなく
プレビュー結果として返却する。

```text
200 OK
canImport = false
```

とする。

---

#### 7.12.22 複数エラーを収集する

CSV構造を正常に解析できる場合は、
可能な範囲で
複数の入力・業務エラーを収集する。

例えば、

```text
2行目
ASSET_ACCOUNT_NOT_FOUND

3行目
HOLDING_ASSET_NOT_FOUND

5行目
INVALID_VALUE
```

を、
1回のプレビューで
まとめて返却できるようにする。

利用者が
1件ずつ修正して
再アップロードする必要を
できる限り減らす。

---

#### 7.12.23 CSV全体エラー

CSV全体に関係するエラーは、
行エラーとは分離して扱う。

例えば、

- データ行0件
- 複数対象年月
- 確定済み月末資産状況

などを
CSV全体エラーとして扱う。

プレビュー結果では、

```text
errors
```

へ格納する。

---

#### 7.12.24 行エラー

個別のCSV行に関係するエラーは、
各行の

```text
errors
```

へ格納する。

例えば、

- 資産口座不存在
- 残高記録単位不一致
- 保有商品不存在
- 対象年月時点の保有商品無効
- `value`不正
- CSV内重複
- 既存商品別月末評価額

などを対象とする。

---

#### 7.12.25 canImport

CSV全体エラーおよび
各行エラーが
1件も存在しない場合のみ、

```text
canImport = true
```

とする。

いずれかのエラーが
1件でも存在する場合は、

```text
canImport = false
```

とする。

概念的には、

```text
globalErrors = 0
AND
rowErrors = 0
    ↓
canImport = true
```

とする。

---

#### 7.12.26 正常行だけを登録対象としない

CSV-005は
プレビューAPIであるため
実際の登録は行わない。

また、
CSV-006では
CSV全体を
全件成功または全件失敗とする。

そのため、
CSV-005で

```text
正常行だけ登録可能
```

のような
部分登録用の判定を返さない。

1件でもエラーがあれば、

```text
canImport = false
```

とする。

---

#### 7.12.27 プレビュー結果を保存しない

CSV-005で生成した
プレビュー結果を
データベースへ保存しない。

以下のような
管理情報は作成しない。

```text
previewId
previewToken
csv_import_session
csv_preview_history
```

CSV-006では、
CSVファイルを再送して
最新状態で再検証する。

---

#### 7.12.28 CSVファイルを保存しない

Phase1では、
CSV-005で受信した
CSVファイル自体を
永続保存しない。

CSV解析に必要な
一時的なファイルとしてのみ扱う。

CSV-006用に
サーバー側へ
アップロードファイルを保持しない。

---

#### 7.12.29 CSV-005では副作用を発生させない

CSV-005では、
以下を行わない。

- `month_end_asset_snapshots`の作成
- `month_end_holding_values`の作成
- `month_end_holding_values`の更新
- `month_end_holding_values`の削除
- `month_end_asset_snapshots.confirmed`の変更

プレビューは、
参照・検証のみとする。

---

#### 7.12.30 CSV-006で再検証する

CSV-005で

```text
canImport = true
```

となっていても、
CSV-006では
最新状態で再検証する。

例えば、

```text
CSV-005
confirmed = false
    ↓
canImport = true

別処理
confirmed = true
    ↓

CSV-006
    ↓
登録不可
```

となり得る。

CSV-005の結果を
登録予約や
登録権利として扱わない。

---

### 7.13 処理フロー

正常系の処理フローは、
以下とする。

```text
POST
/api/v1/month-end-holding-values/imports/preview
    ↓
X-User-Id検証
    ↓
file検証
    ↓
CSV解析
    ↓
CSVヘッダー検証
    ↓
CSV行入力値検証
    ↓
対象年月特定
    ↓
CSV内重複確認
    ↓
資産口座一括取得
    ↓
保有商品一括取得
    ↓
月末資産状況取得
    ↓
既存商品別月末評価額取得
    ↓
業務ルール検証
    ↓
エラー集約
    ↓
canImport判定
    ↓
Preview DTO生成
    ↓
200 OK
```

CSV-005では、
この処理中に
業務データを更新しない。

---

#### 7.13.1 リクエスト検証

最初に、
以下を確認する。

```text
X-User-Id
file
```

利用者コンテキストまたは
アップロードファイル自体が
不正な場合は、
CSV内容の検証へ進まない。

---

#### 7.13.2 CSV解析

CSVファイルを解析し、

- ヘッダー
- データ行
- CSV上の行番号

を取得する。

CSVとして
正常に解析できない場合は、
プレビュー結果ではなく
HTTPエラーとする。

---

#### 7.13.3 ヘッダー検証

CSVヘッダーが、

```text
target_year_month
asset_account_name
holding_asset_name
value
```

と完全一致することを確認する。

ヘッダー不正の場合は、
後続の行検証や
業務データ検索へ進まない。

---

#### 7.13.4 行入力値検証

各行について、
以下を検証する。

```text
target_year_month
asset_account_name
holding_asset_name
value
```

形式不正が存在する場合は、
各行のエラーとして保持する。

可能な範囲で
後続行の検証も継続する。

---

#### 7.13.5 対象年月特定

正常に取得できた
`target_year_month`から
CSV全体の対象年月を特定する。

単一年月に
特定できない場合は、
CSV全体エラーを生成する。

---

#### 7.13.6 資産口座取得

CSV内の
`asset_account_name`を収集し、
操作対象利用者に属する
資産口座をまとめて取得する。

CSV行ごとに
個別SQLを発行しない。

取得結果は、
資産口座名をキーとして
Map化してよい。

---

#### 7.13.7 保有商品取得

CSVで指定された
資産口座・保有商品について、
操作対象利用者に属する
保有商品をまとめて取得する。

概念的な識別キーは、

```text
asset_account_id
+
holding_asset_name
```

とする。

CSV行ごとに
保有商品検索SQLを
発行しない。

---

#### 7.13.8 月末資産状況取得

対象年月を
単一に特定できた場合は、
操作対象利用者の
月末資産状況を取得する。

```text
snapshotなし
    → 登録可否判定継続

snapshotあり
+
confirmed = false
    → 登録可否判定継続

snapshotあり
+
confirmed = true
    → CSV全体エラー
```

とする。

---

#### 7.13.9 既存商品別月末評価額取得

snapshotが存在する場合は、
CSVで対象となる
保有商品について、
既存商品別月末評価額を
まとめて取得する。

既存データがある行は、
行エラーとする。

---

#### 7.13.10 エラー集約

CSV全体エラーと
各行エラーを集約する。

概念的には、

```text
globalErrors
+
rows[].errors
    ↓
canImport判定
```

とする。

---

#### 7.13.11 canImport判定

以下を満たす場合のみ、

```text
canImport = true
```

とする。

```text
CSV全体エラーなし
AND
全行エラーなし
AND
データ行1件以上
```

それ以外は、

```text
canImport = false
```

とする。

---

#### 7.13.12 Preview DTO生成

検証結果から、
以下を含む
Preview DTOを生成する。

```text
targetYearMonth
canImport
errors
rows
```

Eloquent Modelや
Query結果を
そのままレスポンスへ返却しない。

---

### 7.14 正常レスポンス

CSVファイルを
正常に解析し、
プレビュー処理を
最後まで実行できた場合は、

```http
200 OK
```

を返却する。

CSV内に
入力・業務エラーが存在する場合でも、
プレビュー処理自体が
正常に完了していれば
`200 OK`とする。

---

#### 7.14.1 登録可能な場合

すべての検証に成功した場合は、
概念的に以下を返却する。

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "canImport": true,
    "errors": [],
    "rows": [
      {
        "rowNumber": 2,
        "assetAccountName": "証券口座",
        "holdingAssetName": "全世界株式",
        "value": 1500000,
        "errors": []
      },
      {
        "rowNumber": 3,
        "assetAccountName": "証券口座",
        "holdingAssetName": "S&P500",
        "value": 800000,
        "errors": []
      }
    ]
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 7.14.2 登録不可の場合

CSVを正常に解析できるが、
入力・業務エラーが存在する場合は、

```text
200 OK
canImport = false
```

とする。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "canImport": false,
    "errors": [],
    "rows": [
      {
        "rowNumber": 2,
        "assetAccountName": "証券口座",
        "holdingAssetName": "存在しない商品",
        "value": 1500000,
        "errors": [
          {
            "code": "HOLDING_ASSET_NOT_FOUND",
            "message": "指定された保有商品が存在しません。"
          }
        ]
      }
    ]
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 7.14.3 CSV全体エラーがある場合

CSV全体に関係する
業務エラーが存在する場合は、
`data.errors`へ設定する。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "canImport": false,
    "errors": [
      {
        "code": "MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED",
        "message": "対象年月の月末資産状況は確定済みです。"
      }
    ],
    "rows": [
      {
        "rowNumber": 2,
        "assetAccountName": "証券口座",
        "holdingAssetName": "全世界株式",
        "value": 1500000,
        "errors": []
      }
    ]
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 7.14.4 データ行0件の場合

CSVヘッダーは正常だが、
データ行が存在しない場合は、
概念的に以下とする。

```json
{
  "data": {
    "targetYearMonth": null,
    "canImport": false,
    "errors": [
      {
        "code": "CSV_DATA_REQUIRED",
        "message": "登録対象のデータが存在しません。"
      }
    ],
    "rows": []
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 7.14.5 対象年月を特定できない場合

複数対象年月が存在する場合など、
CSV全体として
単一の対象年月を
特定できない場合は、

```text
targetYearMonth = null
```

としてよい。

概念例：

```json
{
  "data": {
    "targetYearMonth": null,
    "canImport": false,
    "errors": [
      {
        "code": "MULTIPLE_TARGET_YEAR_MONTHS",
        "message": "1つのCSVには1つの対象年月のみ指定してください。"
      }
    ],
    "rows": []
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 7.14.6 HTTPエラーとなる場合

以下のように、
プレビュー処理そのものを
実行できない場合は、
`200 OK`ではなく
APIエラーとする。

- `X-User-Id`不正
- `file`未指定
- ファイル形式不正
- ファイルサイズ超過
- CSV解析不能
- CSVヘッダー不正
- 想定外のサーバー内部エラー

これらの場合は、
`data.canImport`を返却しない。

---

#### 7.14.7 レスポンス順序

`rows`は、
CSV上のデータ行の
並び順を維持して返却する。

概念的には、

```text
rowNumber ASC
```

となる。

フロントエンドで
CSV行番号を確認しやすいよう、
バックエンドで
勝手に資産口座名や
保有商品名による
並び替えを行わない。

---

### 7.15 レスポンス項目

正常時の
`data`配下には、
以下の項目を返却する。

| 項目 | 型 | NULL | 内容 |
|---|---|:---:|---|
| `targetYearMonth` | string | ○ | CSVから特定した対象年月 |
| `canImport` | boolean | × | CSV-006で登録可能か |
| `errors` | array | × | CSV全体に関するエラー |
| `rows` | array | × | CSV各行のプレビュー結果 |

---

#### 7.15.1 targetYearMonth

CSVから特定した
対象年月を返却する。

形式は、

```text
YYYY-MM
```

とする。

正常例：

```json
{
  "targetYearMonth": "2026-07"
}
```

単一の対象年月として
特定できない場合は、

```json
{
  "targetYearMonth": null
}
```

とする。

---

#### 7.15.2 canImport

CSV-006で
登録可能かどうかを返却する。

```text
true
    → CSV-005実行時点では登録可能

false
    → 登録不可となるエラーあり
```

`canImport = true`でも、
CSV-006の成功を
保証するものではない。

CSV-006実行時には
最新状態で再検証する。

---

#### 7.15.3 errors

CSV全体に関係する
エラーを配列で返却する。

概念的な構造は、
以下とする。

```json
{
  "code": "MULTIPLE_TARGET_YEAR_MONTHS",
  "message": "1つのCSVには1つの対象年月のみ指定してください。"
}
```

エラーがない場合は、

```json
[]
```

とする。

---

#### 7.15.4 rows

CSV各データ行について、
プレビュー結果を返却する。

各要素は、
以下の項目を持つ。

| 項目 | 型 | NULL | 内容 |
|---|---|:---:|---|
| `rowNumber` | integer | × | CSV上の行番号 |
| `assetAccountName` | string | ○ | CSVに指定された資産口座名 |
| `holdingAssetName` | string | ○ | CSVに指定された保有商品名 |
| `value` | integer | ○ | 正常に解析できた商品別月末評価額 |
| `errors` | array | × | 当該行に関するエラー |

---

#### 7.15.5 rowNumber

CSVファイル上の
実際の行番号を返却する。

ヘッダー行を
1行目とするため、
最初のデータ行は、

```text
rowNumber = 2
```

となる。

利用者が
CSV上のエラー箇所を
特定するために使用する。

---

#### 7.15.6 assetAccountName

CSVに入力された
`asset_account_name`を
API向けのcamelCaseへ変換して返却する。

CSV列：

```text
asset_account_name
```

レスポンス：

```text
assetAccountName
```

入力値が存在しないなど、
正常な文字列として
扱えない場合は
`null`としてよい。

内部の

```text
asset_accounts.id
```

は返却しない。

---

#### 7.15.7 holdingAssetName

CSVに入力された
`holding_asset_name`を
API向けのcamelCaseへ変換して返却する。

CSV列：

```text
holding_asset_name
```

レスポンス：

```text
holdingAssetName
```

正常な値として
取得できない場合は、
`null`としてよい。

内部の

```text
holding_assets.id
```

は返却しない。

---

#### 7.15.8 value

CSVの`value`を
正常に整数として解析できた場合は、
integerとして返却する。

正常例：

```json
{
  "value": 1500000
}
```

0円の場合も、

```json
{
  "value": 0
}
```

として返却する。

未入力や
整数として解析不能な場合は、

```json
{
  "value": null
}
```

としてよい。

---

#### 7.15.9 行エラー

各行の`errors`には、
その行に関する
入力・業務エラーを返却する。

概念的な構造は、
以下とする。

```json
{
  "code": "HOLDING_ASSET_NOT_FOUND",
  "message": "指定された保有商品が存在しません。"
}
```

1行に
複数エラーが存在する可能性があるため、
配列とする。

エラーがない場合は、

```json
[]
```

とする。

---

#### 7.15.10 CSV全体エラーと行エラーを分離する

以下のように
用途を分ける。

```text
data.errors
    ↓
CSV全体に関するエラー

data.rows[].errors
    ↓
特定CSV行に関するエラー
```

フロントエンドが、
どの範囲に関するエラーかを
判別できる構造とする。

---

#### 7.15.11 内部IDを返却しない

プレビュー結果には、
以下の内部IDを返却しない。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`

CSV利用者が
確認する必要があるのは、

- CSV行番号
- 資産口座名
- 保有商品名
- 商品別月末評価額
- エラー内容

である。

---

#### 7.15.12 返却しない情報

本APIでは、
以下の情報を返却しない。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`
- `asset_accounts.balance_recording_unit`
- `month_end_asset_snapshots.confirmed`
- `created_at`
- `updated_at`
- `asset_account_available_settings`
- 既存の`month_end_holding_values.value`
- CSVファイル全文

業務判定に使用した
内部情報を
そのままレスポンスへ公開しない。

---

### 7.16 エラーレスポンス

CSV-005では、
CSVファイルを正常に解析でき、
プレビュー処理を最後まで実行できた場合、
CSV内に業務エラーが存在していても
`200 OK`として返却する。

一方、
リクエスト自体やCSV構造に問題があり、
プレビュー処理を成立させられない場合は、
API共通のJSONエラーレスポンスを返却する。

概念的には、
以下とする。

```text
CSV解析可能
+
入力・業務エラーあり
    ↓
200 OK
canImport = false
```

```text
リクエスト不正
または
CSV構造不正
    ↓
4xx Error
```

本APIでは、
プレビュー結果内の業務エラーと
HTTPエラーを明確に分離する。

---

#### 7.16.1 HTTPエラー一覧

本APIで想定する
主なHTTPエラーは、
以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`が指定されていない |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`の形式が不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 指定された利用者が存在しない、または論理削除済み |
| `422 Unprocessable Entity` | `VALIDATION_ERROR` | `file`未指定、ファイル形式・サイズなどの入力不正 |
| `422 Unprocessable Entity` | `INVALID_CSV_FORMAT` | CSV解析不能、ヘッダー不正などCSV構造が不正 |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバー内部エラー |

CSV行単位または
CSV全体の業務エラーは、
原則として
このHTTPエラー一覧には含めない。

---

#### 7.16.2 プレビュー内で扱うエラー

CSVを正常に解析できる場合は、
以下のようなエラーを
プレビュー結果内で扱う。

- `CSV_DATA_REQUIRED`
- `MULTIPLE_TARGET_YEAR_MONTHS`
- `ASSET_ACCOUNT_NOT_FOUND`
- `BALANCE_RECORDING_UNIT_MISMATCH`
- `HOLDING_ASSET_NOT_FOUND`
- `HOLDING_ASSET_NOT_AVAILABLE`
- `INVALID_VALUE`
- `DUPLICATE_HOLDING_ASSET_IN_CSV`
- `MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED`
- `MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`

これらが存在する場合は、

```text
200 OK
canImport = false
```

とする。

---

#### 7.16.3 USER_CONTEXT_REQUIRED

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

HTTPステータスは、

```http
400 Bad Request
```

とする。

CSV解析処理へ進まない。

---

#### 7.16.4 INVALID_USER_ID

`X-User-Id`の形式が
不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

HTTPステータスは、

```http
400 Bad Request
```

とする。

例えば、
以下のような値を
不正とする。

```text
0
-1
abc
1.5
```

---

#### 7.16.5 USER_NOT_FOUND

指定された利用者が
存在しない場合、
または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

HTTPステータスは、

```http
404 Not Found
```

とする。

資産口座や
保有商品の検索へ進まない。

---

#### 7.16.6 file未指定

`file`が
指定されていない場合は、

```text
VALIDATION_ERROR
```

として扱う。

HTTPステータスは、

```http
422 Unprocessable Entity
```

とする。

概念例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "file",
        "reason": "required",
        "message": "CSVファイルを指定してください。"
      }
    ]
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 7.16.7 ファイル形式不正

許可されていない
ファイル形式の場合は、

```text
VALIDATION_ERROR
```

として扱う。

例えば、
以下を対象とする。

```text
.xlsx
.xls
.pdf
```

CSV解析処理へ進まない。

---

#### 7.16.8 ファイルサイズ超過

CSV共通仕様で定める
最大ファイルサイズを
超えている場合は、

```text
VALIDATION_ERROR
```

を返却する。

CSV内容の解析や
業務データ検索へ進まない。

---

#### 7.16.9 空ファイル

CSVファイルが
0バイトの場合、
または有効なヘッダー行を
取得できない場合は、

```text
INVALID_CSV_FORMAT
```

として扱う。

HTTPステータスは、

```http
422 Unprocessable Entity
```

とする。

---

#### 7.16.10 CSV解析不能

CSVとして
正常に解析できない場合は、

```text
INVALID_CSV_FORMAT
```

を返却する。

例えば、
以下を対象とする。

- 引用符の不正
- CSV構造破損
- ヘッダー取得不能
- 列構造を正常に解釈できない

内部のParser例外を
そのままレスポンスへ公開しない。

---

#### 7.16.11 CSVヘッダー不正

CSVヘッダーが
以下と一致しない場合は、

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

```text
INVALID_CSV_FORMAT
```

として扱う。

例えば、
以下は不正とする。

```csv
targetYearMonth,assetAccountName,holdingAssetName,value
```

```csv
asset_account_name,target_year_month,holding_asset_name,value
```

```csv
target_year_month,asset_account_name,holding_asset_name,value,memo
```

ヘッダー不正の場合は、
後続の行検証へ進まない。

---

#### 7.16.12 CSV_DATA_REQUIRED

CSVヘッダーは正常だが、
データ行が存在しない場合は、

```text
CSV_DATA_REQUIRED
```

を
`data.errors`へ設定する。

HTTPステータスは、

```http
200 OK
```

とする。

概念的には、

```json
{
  "data": {
    "targetYearMonth": null,
    "canImport": false,
    "errors": [
      {
        "code": "CSV_DATA_REQUIRED",
        "message": "登録対象のデータが存在しません。"
      }
    ],
    "rows": []
  }
}
```

となる。

---

#### 7.16.13 MULTIPLE_TARGET_YEAR_MONTHS

1つのCSVに
複数対象年月が存在する場合は、

```text
MULTIPLE_TARGET_YEAR_MONTHS
```

を
CSV全体エラーとして扱う。

HTTPステータスは、

```http
200 OK
```

とする。

この場合、

```text
targetYearMonth = null
canImport = false
```

としてよい。

---

#### 7.16.14 ASSET_ACCOUNT_NOT_FOUND

CSVで指定された
資産口座が
操作対象利用者に存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を行エラーとして扱う。

他利用者に
同名資産口座が存在していても、
正常とは判定しない。

---

#### 7.16.15 BALANCE_RECORDING_UNIT_MISMATCH

指定された資産口座が
口座単位で管理されている場合は、

```text
BALANCE_RECORDING_UNIT_MISMATCH
```

を行エラーとして扱う。

商品別月末評価額CSVでは、
商品単位の資産口座のみを
対象とする。

---

#### 7.16.16 HOLDING_ASSET_NOT_FOUND

CSVで指定された

```text
asset_account_name
+
holding_asset_name
```

に対応する
保有商品が存在しない場合は、

```text
HOLDING_ASSET_NOT_FOUND
```

を行エラーとして扱う。

他資産口座または
他利用者に
同名保有商品が存在していても、
正常とは判定しない。

---

#### 7.16.17 HOLDING_ASSET_NOT_AVAILABLE

対象保有商品が
CSVの`target_year_month`時点で
商品別月末評価額の
記録対象として無効な場合は、

```text
HOLDING_ASSET_NOT_AVAILABLE
```

を行エラーとして扱う。

現在時点ではなく、
対象年月時点で判定する。

---

#### 7.16.18 INVALID_VALUE

`value`が
0以上の整数として
扱えない場合は、

```text
INVALID_VALUE
```

を行エラーとして扱う。

例えば、
以下は不正とする。

```text
-1
1000.5
abc
¥1000
1,000
```

以下は正常とする。

```text
0
1
1000
```

---

#### 7.16.19 DUPLICATE_HOLDING_ASSET_IN_CSV

同一CSV内で、
同じ

```text
asset_account_name
+
holding_asset_name
```

の組み合わせが
複数回指定されている場合は、

```text
DUPLICATE_HOLDING_ASSET_IN_CSV
```

を行エラーとして扱う。

先勝ち、
後勝ち、
評価額合算などの
自動解決は行わない。

---

#### 7.16.20 MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED

対象年月の
月末資産状況が

```text
confirmed = true
```

の場合は、

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

を
CSV全体エラーとして扱う。

CSV-005では
HTTP 409にはせず、

```text
200 OK
canImport = false
```

とする。

プレビューにおいて
登録不可理由を
確認できるようにするためである。

---

#### 7.16.21 MONTH_END_HOLDING_VALUE_ALREADY_EXISTS

同一snapshot、
同一保有商品について、
既に商品別月末評価額が
存在する場合は、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

を
行エラーとして扱う。

既存`value`が
CSV値と同じ場合でも
登録済みとして扱う。

---

#### 7.16.22 複数エラー

1つのCSVに
複数エラーが存在する場合は、
可能な範囲で
まとめて返却する。

例えば、

```text
2行目
ASSET_ACCOUNT_NOT_FOUND

3行目
HOLDING_ASSET_NOT_FOUND

5行目
INVALID_VALUE
```

を
同一レスポンス内で返却できるようにする。

---

#### 7.16.23 INTERNAL_SERVER_ERROR

CSV解析や
業務データ取得などで
想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

HTTPステータスは、

```http
500 Internal Server Error
```

とする。

レスポンスへ、
以下を含めない。

- SQL
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- スタックトレース
- PHP内部エラー
- Laravel内部例外メッセージ
- サーバーファイルパス

詳細情報は、
サーバーログへ記録する。

---

### 7.17 HTTPステータス

本APIで使用する
HTTPステータスは、
以下とする。

| HTTPステータス | 用途 |
|---|---|
| `200 OK` | CSVプレビュー処理成功。業務エラーありの場合も含む |
| `400 Bad Request` | 利用者コンテキストの指定不備 |
| `404 Not Found` | 指定された利用者が存在しない |
| `422 Unprocessable Entity` | ファイル入力またはCSV構造が不正 |
| `500 Internal Server Error` | 想定外のサーバー内部エラー |

---

#### 7.17.1 200 OK

CSVファイルを正常に解析し、
プレビュー結果を
生成できた場合は、

```http
200 OK
```

を返却する。

以下の両方を含む。

```text
canImport = true
canImport = false
```

業務エラーの存在だけを理由として
HTTPエラーにはしない。

---

#### 7.17.2 400 Bad Request

以下の場合に使用する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
```

CSV内容に関する
入力不正には使用しない。

---

#### 7.17.3 404 Not Found

指定された利用者が
存在しない場合、
または論理削除済みの場合に使用する。

```text
USER_NOT_FOUND
```

CSV内の資産口座や
保有商品が存在しない場合は、
HTTP 404にはしない。

プレビュー結果内の
行エラーとして扱う。

---

#### 7.17.4 422 Unprocessable Entity

プレビュー処理を成立させられない
入力・構造不正に使用する。

主に以下を対象とする。

- `file`未指定
- ファイル形式不正
- ファイルサイズ超過
- CSV解析不能
- CSVヘッダー不正

---

#### 7.17.5 500 Internal Server Error

想定外の
サーバー内部エラーが
発生した場合に使用する。

対象となる
独自エラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

---

### 7.18 副作用

CSV-005には、
業務データに対する
副作用はない。

本APIでは、
CSV内容の解析・検証のみを行う。

---

#### 7.18.1 更新しないテーブル

CSV-005では、
以下のテーブルを更新しない。

- `users`
- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_account_available_settings`
- `net_incomes`
- `objectives`
- `assessment_histories`

INSERT、
UPDATE、
DELETEを行わない。

---

#### 7.18.2 snapshotを作成しない

対象年月の
`month_end_asset_snapshots`が
存在しない場合でも、
CSV-005では
新規作成しない。

```text
snapshotなし
    ↓
登録可能性のみ判定
```

とする。

実際の作成は、
CSV-006で行う。

---

#### 7.18.3 商品別月末評価額を作成しない

`canImport = true`の場合でも、

```text
month_end_holding_values
```

へデータを登録しない。

CSV-005は、
登録前確認に責務を限定する。

---

#### 7.18.4 CSVファイルを永続保存しない

Phase1では、
アップロードされたCSVファイルを
永続保存しない。

以下のような
一時管理テーブルや
ファイル保存を行わない。

```text
csv_preview_files
csv_import_sessions
csv_preview_histories
```

CSV-006では、
CSVファイルを再送する。

---

#### 7.18.5 プレビュー結果を保存しない

以下の情報を
データベースへ保存しない。

```text
targetYearMonth
canImport
errors
rows
```

CSV-006は、
CSV-005の
プレビュー結果を参照せず、
最新状態で再検証する。

---

### 7.19 トランザクション

CSV-005では、
業務データを更新しないため、
明示的な
`DB::transaction()`を使用しない。

以下のような実装は
行わない。

```php
DB::transaction(
    function () {
        // CSVプレビューのみ
    },
);
```

CSV解析・業務検証を
長時間トランザクション内で
実行しない。

---

#### 7.19.1 複数SELECT

CSV-005では、
複数のQueryを使用して

- 資産口座
- 保有商品
- snapshot
- 既存商品別月末評価額

を取得する可能性がある。

Phase1では、
これらのSELECT間で
厳密なスナップショット一貫性を
保証するための
明示的なトランザクションは使用しない。

最終的な登録可否は、
CSV-006で再確認する。

---

### 7.20 ロック

CSV-005では、
行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

プレビュー中に
月末資産状況や
商品別月末評価額を
長時間ロックしない。

---

#### 7.20.1 CSV-005とCSV-006の間をロックしない

CSV-005実行後から
CSV-006実行まで、
データベースロックを
保持する設計は採用しない。

利用者が
プレビュー画面を
長時間確認する可能性があるためである。

代わりに、
CSV-006で
最新状態を再検証する。

---

### 7.21 キャッシュ

Phase1では、
CSV-005専用の
サーバー側アプリケーションキャッシュを
使用しない。

同じCSVファイルでも、
業務データの状態が変われば
プレビュー結果も変化するためである。

---

#### 7.21.1 キャッシュしない対象

特に、
以下の状態は
最新情報を使用する。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots.confirmed`
- `month_end_holding_values`

古いキャッシュをもとに
`canImport`を判定しない。

---

#### 7.21.2 同一CSVの再プレビュー

同じCSVを
再度アップロードした場合でも、
その時点の最新状態を参照して
再検証する。

そのため、

```text
1回目
canImport = true

業務状態変更

2回目
canImport = false
```

となることを許容する。

---

### 7.22 冪等性

CSV-005は、
HTTPメソッドとしては
`POST`を使用するが、
業務データを変更しない。

そのため、
業務上は冪等なAPIとして扱う。

同一状態で
同一CSVを複数回送信した場合、
同一内容のプレビュー結果となることを基本とする。

---

#### 7.22.1 同一CSVの複数回実行

同一利用者、
同一DB状態、
同一CSVで
複数回実行した場合は、
概念的に同じ

- `targetYearMonth`
- `canImport`
- `errors`
- `rows`

を返却する。

実行回数によって
商品別月末評価額が
作成されることはない。

---

#### 7.22.2 業務状態が変わった場合

CSVファイルが同じでも、
実行間で
業務状態が変化した場合は、
プレビュー結果が変化してよい。

例えば、

```text
1回目

snapshot
confirmed = false
    ↓
canImport = true
```

その後、

```text
confirmed = true
```

となった場合、

```text
2回目
    ↓
canImport = false
```

となる。

これは
非冪等な副作用ではなく、
最新状態を参照した結果の変化である。

---

#### 7.22.3 Idempotency-Key

CSV-005では、

```text
Idempotency-Key
```

を使用しない。

本APIは
業務データを変更しないため、
重複実行による
二重登録リスクが存在しない。

---

#### 7.22.4 二重送信

フロントエンドでは、
プレビュー実行中に
ボタンを無効化するなど、
不要な二重送信を
抑制してよい。

ただし、
二重送信されても
業務データが変更されないことを
バックエンド側で保証する。

---

#### 7.22.5 CSV-006との違い

CSV-005は、
複数回実行しても
業務データへ副作用がない。

一方、
CSV-006は
商品別月末評価額を登録するため、
同じCSVを再送した場合に
重複エラーとなる可能性がある。

概念的には、

```text
CSV-005
POST
+
副作用なし
+
業務上冪等
```

```text
CSV-006
POST
+
副作用あり
+
再実行時は重複確認
```

と区別する。

---

### 7.23 関連テーブル

CSV-005では、
商品別月末評価額CSVの
プレビュー処理を行うため、
以下のテーブルを参照する。

| テーブル | 用途 | 更新 |
|---|---|:---:|
| `users` | 操作対象利用者の確認 | × |
| `asset_accounts` | CSVで指定された資産口座の特定、残高記録単位の確認 | × |
| `holding_assets` | CSVで指定された保有商品の特定 | × |
| `asset_account_available_settings` | 対象年月時点での保有商品の利用可否判定に必要な設定の確認 | × |
| `month_end_asset_snapshots` | 対象年月の月末資産状況および確定状態の確認 | × |
| `month_end_holding_values` | 既存の商品別月末評価額の重複確認 | × |

CSV-005では、
すべて参照のみとし、
INSERT、
UPDATE、
DELETEを行わない。

---

#### 7.23.1 users

操作対象利用者は、
`X-User-Id`から特定する。

概念的な条件は、
以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者コンテキストの検証は、
API共通Middlewareで行う。

CSV-005固有の処理では、
検証済みの利用者コンテキストを使用する。

---

#### 7.23.2 asset_accounts

CSVの

```text
asset_account_name
```

から、
操作対象利用者に属する
資産口座を特定する。

概念的な条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    IN CSVで指定された資産口座名

AND

asset_accounts.deleted_at
    IS NULL
```

また、

```text
balance_recording_unit
```

を確認し、
商品別月末評価額を
記録可能な資産口座であることを判定する。

---

#### 7.23.3 holding_assets

CSVの

```text
asset_account_name
+
holding_asset_name
```

から、
対象保有商品を特定する。

概念的には、

```text
holding_assets.asset_account_id
    = 対象資産口座ID

AND

holding_assets.name
    = CSVのholding_asset_name

AND

holding_assets.deleted_at
    IS NULL
```

とする。

保有商品名だけを条件として
システム全体から検索しない。

---

#### 7.23.4 asset_account_available_settings

CSVの対象年月時点で、
対象保有商品が
商品別月末評価額の
記録対象として有効かを
判定するために参照する。

判定では、
現在日時ではなく、

```text
target_year_month
```

を基準とする。

対象年月に対応する
利用可能資産設定を参照し、
対象保有商品が
記録可能な状態かを判定する。

---

#### 7.23.5 month_end_asset_snapshots

操作対象利用者と
CSVの対象年月から、
月末資産状況を取得する。

概念的な条件は、
以下とする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = CSVのtarget_year_month
```

snapshotが存在する場合は、

```text
confirmed
```

を確認する。

```text
snapshotなし
    → 登録可否判定を継続

snapshotあり
+
confirmed = false
    → 登録可否判定を継続

snapshotあり
+
confirmed = true
    → canImport = false
```

とする。

CSV-005では、
snapshotを新規作成しない。

---

#### 7.23.6 month_end_holding_values

対象年月のsnapshotが
存在する場合は、
CSVで指定された保有商品について、
既存の商品別月末評価額を確認する。

概念的な条件は、
以下とする。

```text
month_end_holding_values.month_end_asset_snapshot_id
    = 対象snapshot ID

AND

month_end_holding_values.holding_asset_id
    IN CSVで対象となる保有商品ID
```

既存データが存在する場合は、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

相当の
行エラーとして扱う。

既存データの
更新や削除は行わない。

---

### 7.24 Query方針

CSV-005では、
CSV行ごとに
データベース検索を行わない。

CSV解析後に
検索条件をまとめ、
必要な業務データを
一括取得する。

概念的には、
以下の順序とする。

```text
CSV解析
    ↓
資産口座名を収集
    ↓
資産口座一括取得
    ↓
保有商品名を収集
    ↓
保有商品一括取得
    ↓
利用可能資産設定取得
    ↓
対象年月snapshot取得
    ↓
既存商品別月末評価額一括取得
```

---

#### 7.24.1 N+1を発生させない

以下のような
CSV行ごとの検索は行わない。

```text
1行目
    ↓
asset_accounts SELECT
    ↓
holding_assets SELECT
    ↓
available_settings SELECT
    ↓
month_end_holding_values SELECT

2行目
    ↓
asset_accounts SELECT
    ↓
holding_assets SELECT
    ↓
...
```

CSVの行数に比例して
SQL発行回数が増加する実装を避ける。

---

#### 7.24.2 資産口座の一括取得

CSVから
重複を除いた

```text
asset_account_name
```

を収集する。

概念的には、

```text
WHERE user_id = :userId
AND name IN (...)
AND deleted_at IS NULL
```

によって
まとめて取得する。

取得後は、

```text
assetAccountName
    → AssetAccount
```

のMapとして
利用してよい。

---

#### 7.24.3 保有商品の一括取得

対象となる資産口座IDと
CSV内の保有商品名を使用して、
保有商品をまとめて取得する。

取得後は、
概念的に

```text
assetAccountId
+
holdingAssetName
```

をキーとして
Map化する。

これにより、
異なる資産口座に
同名保有商品が存在する場合でも
正しく判定できるようにする。

---

#### 7.24.4 利用可能資産設定の取得

対象となる資産口座について、
CSVの対象年月時点で必要となる
利用可能資産設定を
まとめて取得する。

CSV行ごとに
個別検索しない。

取得結果を使用して、
対象年月時点での
記録可否を判定する。

---

#### 7.24.5 snapshotの取得

1ファイル1対象年月であるため、
対象年月を正常に特定できた場合は、
操作対象利用者について
最大1件のsnapshotを取得する。

概念的には、

```text
WHERE user_id = :userId
AND target_year_month = :targetYearMonth
```

とする。

他利用者のsnapshotを
取得しない。

---

#### 7.24.6 既存商品別月末評価額の一括取得

snapshotが存在する場合は、
CSVで正常に特定できた
保有商品IDをまとめて使用し、
既存の商品別月末評価額を取得する。

概念的には、

```text
WHERE month_end_asset_snapshot_id = :snapshotId
AND holding_asset_id IN (...)
```

とする。

CSV行ごとに
重複確認SQLを発行しない。

---

#### 7.24.7 不要なQueryを実行しない

前段階の検証結果によって
後続検索が不要な場合は、
Queryを実行しない。

例えば、

```text
CSVヘッダー不正
    ↓
業務データ検索なし
```

```text
対象年月を特定不能
    ↓
snapshot検索なし
```

```text
snapshot不存在
    ↓
month_end_holding_values検索なし
```

とする。

---

### 7.25 レスポンス変換方針

CSV-005では、
Eloquent Modelや
Query結果を
そのままレスポンスへ返却しない。

CSV解析・業務検証結果から、
プレビュー用のDTOを生成し、
APIレスポンス形式へ変換する。

概念的には、

```text
CSV Parser
    ↓
Parsed Rows
    ↓
Validator
    ↓
Validation Result
    ↓
Preview DTO
    ↓
Responder
    ↓
JSON Response
```

とする。

---

#### 7.25.1 Preview DTO

Preview DTOは、
概念的に以下を保持する。

```text
targetYearMonth
canImport
errors
rows
```

各行のDTOは、

```text
rowNumber
assetAccountName
holdingAssetName
value
errors
```

を保持する。

内部IDや
Eloquent Modelを
DTOへ公開しない。

---

#### 7.25.2 CSV入力名とAPIレスポンス名

CSVでは、
snake_caseを使用する。

```text
target_year_month
asset_account_name
holding_asset_name
value
```

APIレスポンスでは、
API共通方針に従って
camelCaseを使用する。

```text
targetYearMonth
assetAccountName
holdingAssetName
value
```

CSV仕様と
JSON API仕様を
混在させない。

---

#### 7.25.3 エラー変換

内部のValidation Resultから、
レスポンス用の

```text
code
message
```

へ変換する。

内部例外メッセージや
データベース情報を
そのまま返却しない。

---

#### 7.25.4 行順序

Preview DTOの`rows`は、
CSV上の行順を維持する。

概念的には、

```text
rowNumber ASC
```

とする。

資産口座名や
保有商品名による
並び替えは行わない。

---

### 7.26 ログ・監視

CSV-005では、
通常のプレビュー成功時に
CSV内容全体を
アプリケーションログへ出力しない。

商品別月末評価額は
利用者の資産情報であるため、
必要以上にログへ残さない。

---

#### 7.26.1 通常ログ

必要に応じて、
以下のような
処理情報を記録してよい。

```text
requestId
API ID
処理結果
CSVデータ行数
canImport
エラー件数
処理時間
```

CSV本文そのものを
ログへ記録しない。

---

#### 7.26.2 エラーログ

`INTERNAL_SERVER_ERROR`など
想定外の例外が発生した場合は、
調査に必要な情報を
サーバーログへ記録する。

例えば、

```text
requestId
API ID
例外クラス
例外メッセージ
スタックトレース
```

などを記録する。

ただし、
クライアントレスポンスへ
内部情報を公開しない。

---

#### 7.26.3 ログへ出力しない情報

原則として、
以下をログへ直接出力しない。

- CSVファイル全文
- 全行の`value`
- 資産情報一覧
- multipartリクエストボディ全体

障害調査で必要な場合でも、
最小限の情報に限定する。

---

#### 7.26.4 requestId

API共通方針に従って、
リクエスト単位の

```text
requestId
```

を利用する。

クライアントレスポンスと
サーバーログを
`requestId`で
関連付けられるようにする。

---

### 7.27 セキュリティ

CSV-005では、
アップロードファイルを扱うため、
通常のJSON APIに加えて
ファイル入力を前提とした
防御を行う。

---

#### 7.27.1 利用者境界

すべての業務データ検索で、
`X-User-Id`から特定した
操作対象利用者との
利用者境界を保証する。

CSV内の名称だけを使用して、
他利用者のデータへ
アクセスしない。

---

#### 7.27.2 内部IDを信用しない

CSV仕様には、
以下の内部IDを含めない。

```text
user_id
asset_account_id
holding_asset_id
month_end_asset_snapshot_id
month_end_holding_value_id
```

利用者がCSVから
内部IDを直接指定する設計にしない。

---

#### 7.27.3 ファイルサイズ制限

CSV共通仕様で定める
最大ファイルサイズを適用する。

過大なCSVファイルによって
メモリやCPUを
過剰消費しないようにする。

---

#### 7.27.4 CSVを実行しない

CSV内容は、
あくまでデータとして解析する。

CSV内の文字列を、

- PHPコード
- SQL
- シェルコマンド
- テンプレートコード

などとして
評価・実行しない。

---

#### 7.27.5 SQLインジェクション対策

CSVの

```text
asset_account_name
holding_asset_name
```

を検索条件として使用する場合でも、
文字列連結によって
SQLを生成しない。

Eloquent、
Query Builder、
バインドパラメータなどを使用する。

---

#### 7.27.6 CSV内容のレスポンス反映

CSVに含まれる名称を
レスポンスへ返却する場合は、
JSON文字列として
安全にエンコードする。

フロントエンドでは、
レスポンス値を
`dangerouslySetInnerHTML`などで
直接HTMLとして描画しない。

---

#### 7.27.7 一時ファイル

アップロード処理で
一時ファイルが作成される場合は、
Laravel・PHPの
通常のアップロード管理に従う。

任意のファイルパスを
利用者から指定させない。

CSV-006で使用する目的で
一時ファイルを
永続保持しない。

---

#### 7.27.8 エラー情報

以下の内部情報を
エラーレスポンスへ
含めない。

- SQL
- テーブル名
- カラム名
- 制約名
- サーバーファイルパス
- スタックトレース
- Laravel内部例外
- PostgreSQL内部エラー

---

### 7.28 性能・スケーラビリティ

CSV-005では、
CSV行数に比例して
SQL発行回数が増加しないようにする。

Phase1では、
大規模CSV処理基盤を
構築することよりも、
一括取得による
単純なN+1回避を優先する。

---

#### 7.28.1 SQL発行回数

概念的には、
CSV行数に関係なく、

```text
資産口座取得
保有商品取得
利用可能資産設定取得
snapshot取得
既存商品別月末評価額取得
```

など、
一定回数程度のQueryで
処理できる構成を目指す。

---

#### 7.28.2 CSV行数

CSV共通仕様で
現実的な最大行数を設定する場合は、
CSV-005にも
同じ上限を適用する。

CSV-005とCSV-006で
異なる行数制限を
持たせない。

---

#### 7.28.3 非同期処理

Phase1では、
CSV-005を
同期APIとして実装する。

以下は導入しない。

```text
Queue
Job
Batch
WebSocket
Polling
```

同期処理で
現実的に扱えない規模になった場合に、
将来拡張として検討する。

---

#### 7.28.4 キャッシュ

プレビュー結果は、
最新の業務状態に依存する。

そのため、
プレビュー結果や
`canImport`を
キャッシュしない。

---

### 7.29 テスト観点

CSV-005では、
正常系だけでなく、

- CSV構造
- 入力値
- 利用者境界
- 業務ルール
- エラー集約
- 副作用
- 性能

を確認する。

CSV-006と
共通化した検証ロジックについては、
同じ入力に対して
判定結果が一致することも確認する。

---

#### 7.29.1 正常系

以下を確認する。

- 正常なCSVで`200 OK`となる
- `canImport = true`となる
- `targetYearMonth`が正しく返却される
- CSV全行が`rows`へ返却される
- `rowNumber`がCSV上の行番号と一致する
- `assetAccountName`が正しく返却される
- `holdingAssetName`が正しく返却される
- `value`がintegerとして返却される
- `value = 0`を正常値として扱う
- `errors`が空配列となる
- 各行の`errors`が空配列となる
- CSV上の行順が維持される

---

#### 7.29.2 利用者

以下を確認する。

- `X-User-Id`未指定
- `X-User-Id`形式不正
- 利用者不存在
- 論理削除済み利用者
- 正常な利用者

利用者コンテキスト不正時に、
CSV解析や
業務データ検索へ
進まないことも確認する。

---

#### 7.29.3 ファイル入力

以下を確認する。

- `file`未指定
- 正常なCSVファイル
- `.xlsx`
- `.xls`
- `.pdf`
- 空ファイル
- 最大ファイルサイズ以内
- 最大ファイルサイズ超過
- CSVとして解析不能なファイル

---

#### 7.29.4 CSVヘッダー

以下を確認する。

正常：

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

異常：

- ヘッダー不足
- ヘッダー過多
- ヘッダー名不一致
- ヘッダー順序不一致
- camelCase指定
- 内部ID列追加
- ヘッダーなし

ヘッダー不正時に、
後続の業務データ検索を
行わないことも確認する。

---

#### 7.29.5 target_year_month

以下を確認する。

- 正常な`YYYY-MM`
- 未入力
- `2026-1`
- `2026/07`
- `202607`
- `2026-00`
- `2026-13`
- 文字列
- 複数対象年月

複数対象年月の場合は、

```text
canImport = false
targetYearMonth = null
```

となることを確認する。

---

#### 7.29.6 asset_account_name

以下を確認する。

- 正常な資産口座名
- 未入力
- 存在しない資産口座名
- 他利用者にのみ存在する同名資産口座
- 論理削除済み資産口座

他利用者の資産口座が
使用されないことを確認する。

---

#### 7.29.7 残高記録単位

以下を確認する。

- 商品単位の資産口座
- 口座単位の資産口座

口座単位の場合は、

```text
BALANCE_RECORDING_UNIT_MISMATCH
```

相当の
行エラーとなることを確認する。

---

#### 7.29.8 holding_asset_name

以下を確認する。

- 正常な保有商品名
- 未入力
- 存在しない保有商品名
- 他資産口座にのみ存在する同名保有商品
- 他利用者にのみ存在する同名保有商品
- 論理削除済み保有商品

対象資産口座と
保有商品の組み合わせによって
正しく特定されることを確認する。

---

#### 7.29.9 対象年月時点の有効性

以下を確認する。

- 対象年月時点で有効
- 対象年月時点で無効
- 現在は有効だが対象年月時点では無効
- 現在は無効でも対象年月時点では有効

現在日時ではなく、
`target_year_month`を基準に
判定されることを確認する。

---

#### 7.29.10 value

以下を確認する。

正常：

```text
0
1
1000
1500000
```

異常：

```text
未入力
-1
1000.5
abc
¥1000
1,000
```

0円が
未入力として扱われないことを
確認する。

---

#### 7.29.11 CSV内重複

以下のようなCSVを確認する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,全世界株式,1600000
```

重複行が

```text
DUPLICATE_HOLDING_ASSET_IN_CSV
```

相当のエラーとなり、

```text
canImport = false
```

となることを確認する。

---

#### 7.29.12 月末資産状況

以下を確認する。

```text
snapshotなし
```

```text
snapshotあり
confirmed = false
```

```text
snapshotあり
confirmed = true
```

snapshot不存在の場合は、
それだけを理由として
登録不可にならないことを確認する。

確定済みの場合は、

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

相当の
CSV全体エラーとなることを確認する。

---

#### 7.29.13 既存商品別月末評価額

以下を確認する。

- 既存データなし
- 既存データあり・CSVと同じ値
- 既存データあり・CSVと異なる値
- 他利用者にのみ既存データあり

操作対象利用者について
既存データが存在する場合は、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

相当の
行エラーとなることを確認する。

既存値と同じ場合でも
正常扱いしない。

---

#### 7.29.14 エラー集約

複数行に
異なるエラーが存在するCSVを使用し、
可能な範囲で
複数エラーが
1回のレスポンスに
集約されることを確認する。

例えば、

```text
2行目
ASSET_ACCOUNT_NOT_FOUND

3行目
HOLDING_ASSET_NOT_FOUND

4行目
INVALID_VALUE
```

を同時に返却できることを確認する。

---

#### 7.29.15 CSV全体エラーと行エラー

以下を確認する。

```text
data.errors
    → CSV全体エラー

rows[].errors
    → 行エラー
```

エラーの種類に応じて
正しい位置へ
格納されることを確認する。

---

#### 7.29.16 canImport

以下を確認する。

```text
CSV全体エラーなし
+
全行エラーなし
    ↓
canImport = true
```

```text
CSV全体エラーあり
    ↓
canImport = false
```

```text
1行以上の行エラーあり
    ↓
canImport = false
```

フロントエンド側で
再計算する必要がない
正しい値が返却されることを確認する。

---

#### 7.29.17 利用者境界

最低でも、
2利用者のデータを用意して確認する。

例えば、

```text
User A
    証券口座
        全世界株式

User B
    証券口座
        全世界株式
```

の状態で、
User Aとして実行した場合に、
User Bの

- 資産口座
- 保有商品
- snapshot
- 商品別月末評価額

が
判定へ使用されないことを確認する。

---

#### 7.29.18 副作用

CSV-005実行前後で、
以下のテーブルの
レコード数・内容が
変更されないことを確認する。

- `asset_accounts`
- `holding_assets`
- `asset_account_available_settings`
- `month_end_asset_snapshots`
- `month_end_holding_values`

特に、
snapshot不存在の場合でも
CSV-005によって
新規snapshotが
作成されないことを確認する。

---

#### 7.29.19 冪等性

同一利用者、
同一DB状態、
同一CSVで
複数回実行し、

- 同じ`targetYearMonth`
- 同じ`canImport`
- 同じ`errors`
- 同じ`rows`

が返却されることを確認する。

複数回実行しても、
業務データが
変更されないことも確認する。

---

#### 7.29.20 状態変更後の再プレビュー

1回目のCSV-005で

```text
canImport = true
```

となった後に、
対象snapshotを

```text
confirmed = true
```

へ変更する。

同じCSVで
再度CSV-005を実行し、

```text
canImport = false
```

へ変化することを確認する。

古いプレビュー結果が
キャッシュされていないことを確認する。

---

#### 7.29.21 SQL発行回数

複数行CSVを使用し、
CSV行数に比例して
SQL発行回数が
増加していないことを確認する。

特に、

```text
asset_accounts
holding_assets
asset_account_available_settings
month_end_holding_values
```

について
N+1が発生していないことを確認する。

---

#### 7.29.22 CSV-006との整合性

同一利用者、
同一DB状態、
同一CSVについて、
CSV-005とCSV-006で
共通の登録可否ルールが
適用されることを確認する。

CSV-005で

```text
canImport = true
```

となる状態では、
状態変更がなければ
CSV-006でも
登録可能であることを確認する。

CSV-005で

```text
canImport = false
```

となるCSVについては、
CSV-006でも
同じ業務ルールによって
登録が拒否されることを確認する。

---

### 7.30 Laravel実装方針

CSV-005では、
Action、
Request、
UseCase、
CSV Definition、
CSV Parser、
CSV Validator、
Query、
DTO、
API Resource、
Responderを分離して実装する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
Middleware
    ↓
Request
    ↓
Action
    ↓
UseCase
    ├─ CSV Parser
    ├─ CSV Validator
    ├─ AssetAccountQuery
    ├─ HoldingAssetQuery
    ├─ AssetAccountAvailableSettingQuery
    ├─ MonthEndAssetSnapshotQuery
    └─ MonthEndHoldingValueQuery
    ↓
Preview DTO
    ↓
API Resource
    ↓
Responder
```

CSV-005では、
CSV-006と同じ
CSV解析・登録可否判定ロジックを
可能な限り共通利用する。

Actionへ、

- CSV解析
- CSVヘッダー検証
- 行入力値検証
- 資産口座検索
- 保有商品検索
- 対象年月時点の有効性判定
- 月末資産状況検索
- 既存商品別月末評価額検索
- CSV内重複判定
- `canImport`判定

を直接記述しない。

---

#### 7.30.1 Action

HTTPリクエストを受け付け、
検証済みCSVファイルおよび
利用者コンテキストを取得する。

CSVプレビューUseCaseを呼び出し、
取得結果をResponderへ渡す。

概念例：

```php
final class PreviewMonthEndHoldingValueCsvAction
{
    public function __invoke(
        PreviewMonthEndHoldingValueCsvRequest $request,
        PreviewMonthEndHoldingValueCsvUseCase $useCase,
        MonthEndHoldingValueCsvPreviewResponder $responder,
        userContext $userContext,
    ): JsonResponse {
        $preview =
            $useCase->execute(
                userId:
                    $userContext->userId,

                file:
                    $request->file('file'),
            );

        return $responder->ok(
            $preview,
        );
    }
}
```

Actionでは、
以下を行わない。

- CSVヘッダー検証
- CSV行解析
- `target_year_month`検証
- `asset_account_name`検証
- `holding_asset_name`検証
- `value`検証
- 1ファイル1対象年月判定
- CSV内重複判定
- 資産口座検索
- 保有商品検索
- 残高記録単位判定
- 対象年月時点の有効性判定
- 月末資産状況検索
- 確定状態判定
- 既存商品別月末評価額検索
- `canImport`判定
- レスポンス形式への変換

Actionは、
UseCaseの呼び出しと
Responderへの受け渡しに
責務を限定する。

---

#### 7.30.2 Request

Requestでは、
HTTPリクエストとして
CSVファイルを受け付けられる状態かを
検証する。

概念例：

```php
final class PreviewMonthEndHoldingValueCsvRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'file' => [
                'required',
                'file',
                'mimes:csv',
                'max:' . config(
                    'csv.max_file_size_kb',
                ),
            ],
        ];
    }
}
```

実際の

- ファイルサイズ上限
- MIME Type
- 拡張子

の扱いは、
CSV共通仕様に従う。

Requestでは、
CSV内部の業務データまでは
検証しない。

---

#### 7.30.3 Requestで行うこと

Requestでは、
主に以下を検証する。

- `file`が指定されていること
- HTTPアップロードファイルであること
- 許可されたファイル形式であること
- ファイルサイズ上限以内であること

これらは、
CSV内容を解析する前に
判定可能な
HTTPリクエストレベルの
バリデーションとする。

---

#### 7.30.4 Requestで行わないこと

Requestでは、
以下のCSV内容・業務ルールを
検証しない。

- CSVヘッダー
- CSVデータ行の存在
- `target_year_month`
- `asset_account_name`
- `holding_asset_name`
- `value`
- 1ファイル1対象年月
- CSV内重複
- 資産口座存在確認
- 残高記録単位
- 保有商品存在確認
- 保有商品と資産口座の関連確認
- 対象年月時点の有効性
- 月末資産状況
- 確定状態
- 既存商品別月末評価額

これらは、
CSV Validator、
UseCase、
Queryで扱う。

---

#### 7.30.5 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONレスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、
`X-User-Id`を検証し、
操作対象利用者を特定する。

概念的には、
以下とする。

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
Request
    ↓
Action
```

Action以降では、
検証済みの
利用者コンテキストを使用する。

---

#### 7.30.6 UseCase

CSVプレビューの
ユースケース処理を担当する。

主な処理は、
以下とする。

1. 操作対象利用者IDを受け取る
2. CSVファイルを受け取る
3. CSVを解析する
4. CSVヘッダーを検証する
5. 各行の入力値を検証する
6. 対象年月を特定する
7. CSV内重複を検証する
8. 資産口座を一括取得する
9. 保有商品を一括取得する
10. 対象年月時点の有効性を確認する
11. 月末資産状況を取得する
12. 既存商品別月末評価額を取得する
13. 業務エラーを集約する
14. `canImport`を判定する
15. Preview DTOを生成する

概念的には、
以下とする。

```text
CSV解析
    ↓
CSV構造検証
    ↓
行入力値検証
    ↓
対象年月特定
    ↓
CSV内重複判定
    ↓
資産口座一括取得
    ↓
保有商品一括取得
    ↓
対象年月時点の有効性確認
    ↓
snapshot取得
    ↓
既存評価額一括取得
    ↓
エラー集約
    ↓
canImport判定
    ↓
Preview DTO
```

UseCaseでは、
SQLやEloquent Query Builderを
直接組み立てない。

データベースアクセスは、
Queryへ委譲する。

---

#### 7.30.7 CSV-006との共通化

CSV-005とCSV-006では、
可能な限り
同じCSV解析・検証ロジックを使用する。

例えば、
以下を共通化する。

```text
MonthEndHoldingValueCsvDefinition
MonthEndHoldingValueCsvParser
MonthEndHoldingValueCsvValidator
MonthEndHoldingValueCsvImportValidator
AssetAccountQuery
HoldingAssetQuery
AssetAccountAvailableSettingQuery
MonthEndAssetSnapshotQuery
MonthEndHoldingValueQuery
```

概念的には、

```text
CSV-005
    ↓
共通解析・検証
    ↓
Preview DTO
```

```text
CSV-006
    ↓
共通解析・検証
    ↓
登録処理
```

とする。

CSV-005とCSV-006で
同じ業務ルールを
別々に実装しない。

---

#### 7.30.8 CSV Definition

CSV-004、
CSV-005、
CSV-006で使用する
商品別月末評価額CSV仕様は、
共通Definitionへ集約する。

概念例：

```php
final class MonthEndHoldingValueCsvDefinition
{
    public const HEADERS = [
        'target_year_month',
        'asset_account_name',
        'holding_asset_name',
        'value',
    ];
}
```

CSV-005だけで
独自のヘッダー定義を持たない。

---

#### 7.30.9 CSV Parser

CSVファイルの読み込みは、
専用Parserへ分離する。

概念例：

```php
final class MonthEndHoldingValueCsvParser
{
    public function parse(
        UploadedFile $file,
    ): ParsedCsv {
        // CSV解析
    }
}
```

Parserでは、
主に以下を行う。

- ファイルオープン
- BOM処理
- ヘッダー取得
- データ行読み込み
- CSV上の行番号管理
- 完全な空行の除外
- CSV構造異常の検出

Parserでは、
業務データ検索や
登録可否判定を行わない。

---

#### 7.30.10 Parsed Row DTO

CSV Parserから返す
各行は、
構造化されたDTOとして
表現してよい。

概念例：

```php
final readonly class ParsedCsvRow
{
    public function __construct(
        public int $rowNumber,

        /** @var array<string, string|null> */
        public array $values,
    ) {
    }
}
```

CSV上の行番号を保持し、
プレビュー結果の

```text
rowNumber
```

として使用できるようにする。

---

#### 7.30.11 CSVヘッダー検証

CSVヘッダーは、
共通Definitionと
完全一致することを確認する。

概念例：

```php
if (
    $parsedCsv->headers
    !== MonthEndHoldingValueCsvDefinition::HEADERS
) {
    throw new InvalidCsvFormatException();
}
```

以下を不正とする。

- ヘッダー不足
- ヘッダー名不一致
- ヘッダー順序不一致
- 余分なヘッダー

ヘッダー不正時は、
後続の業務検証へ進まない。

---

#### 7.30.12 CSV Validator

CSV各行の入力値検証は、
専用Validatorへ分離する。

概念例：

```php
final class MonthEndHoldingValueCsvValidator
{
    public function validateRow(
        ParsedCsvRow $row,
    ): CsvRowValidationResult {
        // 入力値検証
    }
}
```

主に以下を確認する。

```text
target_year_month
    required
    YYYY-MM

asset_account_name
    required

holding_asset_name
    required

value
    required
    integer
    min:0
```

CSV Validatorでは、
データベース検索を行わない。

---

#### 7.30.13 valueの正規化

`value`は、
正常な整数形式の場合のみ
integerへ変換する。

PHPの暗黙的な型変換によって、
以下を正常値として扱ってはならない。

```text
1000abc
1000.5
1,000
¥1000
```

概念例：

```php
$value =
    filter_var(
        $rawValue,
        FILTER_VALIDATE_INT,
    );
```

実際には、
未入力と`0`を
正しく区別できる実装とする。

---

#### 7.30.14 0円の扱い

`value = 0`は、
正常値として扱う。

以下のような
truthy / falsy判定を使用しない。

```php
if (! $value) {
    // 0円まで未入力扱いになる
}
```

必須判定と
数値判定を明確に分離する。

---

#### 7.30.15 データ行0件

CSVヘッダーが正常でも、
データ行が存在しない場合は、
HTTP例外にはしない。

CSV全体エラーとして、

```text
CSV_DATA_REQUIRED
```

を生成する。

概念的には、

```text
CSV解析成功
+
データ行0件
    ↓
200 OK
canImport = false
```

とする。

---

#### 7.30.16 対象年月の特定

正常に取得できた
`target_year_month`を収集し、
CSV全体の対象年月を特定する。

概念例：

```php
$targetYearMonths =
    collect($validRows)
        ->pluck('targetYearMonth')
        ->unique()
        ->values();
```

2種類以上存在する場合は、
CSV全体エラーとして

```text
MULTIPLE_TARGET_YEAR_MONTHS
```

を生成する。

単一の対象年月として
特定できない場合は、
Preview DTOの

```text
targetYearMonth
```

を`null`としてよい。

---

#### 7.30.17 CSV内重複判定

同一CSV内で、
同じ資産口座・保有商品が
複数回指定されていないか確認する。

概念的なキーは、

```text
targetYearMonth
+
assetAccountName
+
holdingAssetName
```

とする。

1ファイル1対象年月が
成立している場合は、
実質的に

```text
assetAccountName
+
holdingAssetName
```

で判定してよい。

重複時は、

```text
DUPLICATE_HOLDING_ASSET_IN_CSV
```

相当の
行エラーを生成する。

---

#### 7.30.18 AssetAccountQuery

CSVで指定された
資産口座名について、
操作対象利用者の資産口座を
一括取得する。

概念例：

```php
$assetAccounts =
    $this->assetAccountQuery
        ->findActiveByNames(
            userId: $userId,
            names: $assetAccountNames,
        );
```

概念的な検索条件は、
以下とする。

```text
user_id = 操作対象利用者ID
AND
name IN (...)
AND
deleted_at IS NULL
```

CSV行ごとに
個別SQLを発行しない。

---

#### 7.30.19 資産口座Map

取得した資産口座は、
名前をキーとして
Map化してよい。

概念例：

```php
$assetAccountMap =
    $assetAccounts->keyBy(
        'name',
    );
```

各行では、

```php
$assetAccount =
    $assetAccountMap->get(
        $row->assetAccountName,
    );
```

として参照する。

---

#### 7.30.20 資産口座不存在

操作対象利用者に
該当資産口座が
存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

相当の
行エラーを生成する。

他利用者に
同名資産口座が存在していても、
取得対象へ含めない。

---

#### 7.30.21 残高記録単位判定

対象資産口座の

```text
balance_recording_unit
```

が
商品単位であることを確認する。

概念例：

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::HOLDING
) {
    // BALANCE_RECORDING_UNIT_MISMATCH
}
```

実際のEnum名・値は、
共通定義に従う。

口座単位の場合は、
行エラーを生成する。

---

#### 7.30.22 HoldingAssetQuery

CSV内で指定された
資産口座と保有商品名について、
対象保有商品を
まとめて取得する。

概念例：

```php
$holdingAssets =
    $this->holdingAssetQuery
        ->findActiveByAssetAccountsAndNames(
            assetAccountIds:
                $assetAccountIds,

            names:
                $holdingAssetNames,
        );
```

検索条件には、
資産口座との関連を
必ず含める。

保有商品名だけで
システム全体を検索しない。

---

#### 7.30.23 保有商品Map

取得した保有商品は、
概念的に

```text
assetAccountId
+
holdingAssetName
```

をキーとして
Map化してよい。

例えば、

```php
$key =
    $assetAccountId
    . ':'
    . $holdingAssetName;
```

として扱う。

これにより、
異なる資産口座に
同名商品が存在する場合でも
正しく特定できる。

---

#### 7.30.24 保有商品不存在

同じ行で指定された
資産口座内に
対象保有商品が
存在しない場合は、

```text
HOLDING_ASSET_NOT_FOUND
```

相当の
行エラーを生成する。

他資産口座や
他利用者に
同名商品が存在していても、
正常とは判定しない。

---

#### 7.30.25 対象年月時点の有効性判定

対象保有商品が、
CSVの`targetYearMonth`時点で
商品別月末評価額の
記録対象として有効かを判定する。

判定に
`asset_account_available_settings`
を使用する設計である場合は、
必要な設定をまとめて取得する。

概念的には、

```php
$this->availabilityValidator
    ->validate(
        holdingAsset:
            $holdingAsset,

        targetYearMonth:
            $targetYearMonth,

        availableSettings:
            $availableSettings,
    );
```

とする。

現在日時を基準に
判定しない。

---

#### 7.30.26 AssetAccountAvailableSettingQuery

対象年月時点の
利用可能状態判定に必要な設定を
一括取得する。

CSV行ごとに
個別Queryを発行しない。

必要な資産口座IDを収集したうえで、
対象年月に必要な設定だけを
まとめて取得する。

---

#### 7.30.27 MonthEndAssetSnapshotQuery

CSVの対象年月について、
操作対象利用者の
月末資産状況を取得する。

概念例：

```php
$snapshot =
    $this->snapshotQuery
        ->findByUserAndTargetYearMonth(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,
        );
```

検索条件には、
必ず利用者IDを含める。

---

#### 7.30.28 snapshot不存在

snapshotが存在しない場合は、
それだけを理由として
プレビューエラーを生成しない。

CSV-006で
必要に応じて作成可能であるため、

```text
snapshot = null
```

のまま
登録可否判定を継続する。

CSV-005では
snapshotを作成しない。

---

#### 7.30.29 確定状態判定

snapshotが存在する場合は、
`confirmed`を確認する。

概念例：

```php
if (
    $snapshot !== null
    && $snapshot->confirmed
) {
    // CSV全体エラー
}
```

確定済みの場合は、

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

相当の
CSV全体エラーを生成する。

---

#### 7.30.30 MonthEndHoldingValueQuery

snapshotが存在する場合は、
CSV対象保有商品について
既存の商品別月末評価額を
まとめて取得する。

概念例：

```php
$existingValues =
    $this->monthEndHoldingValueQuery
        ->findBySnapshotAndHoldingAssets(
            snapshotId:
                $snapshot->id,

            holdingAssetIds:
                $holdingAssetIds,
        );
```

CSV行ごとに
個別SQLを発行しない。

---

#### 7.30.31 既存評価額Map

既存の商品別月末評価額は、

```text
holding_asset_id
```

をキーとして
Map化してよい。

概念例：

```php
$existingValueMap =
    $existingValues->keyBy(
        'holding_asset_id',
    );
```

各CSV行で
既存データ有無を
メモリ上で判定する。

---

#### 7.30.32 既存商品別月末評価額

同一snapshot、
同一保有商品について
既存評価額が存在する場合は、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

相当の
行エラーを生成する。

既存値が
CSV値と同じ場合でも、
正常とは判定しない。

---

#### 7.30.33 N+1問題

CSV行ごとに
個別SQLを発行してはならない。

以下のような構成を避ける。

```text
1行目
    ↓
assetAccount検索
holdingAsset検索
availableSetting検索
existingValue検索

2行目
    ↓
同様の検索

3行目
    ↓
...
```

基本的には、

```text
CSV全体解析
    ↓
必要キー収集
    ↓
asset_accounts一括取得
    ↓
holding_assets一括取得
    ↓
available_settings一括取得
    ↓
snapshot 1件取得
    ↓
existingValues一括取得
    ↓
メモリ上で検証
```

とする。

---

#### 7.30.34 CsvPreviewError DTO

プレビューで使用するエラーは、
専用DTOとして表現する。

概念例：

```php
final readonly class CsvPreviewError
{
    public function __construct(
        public string $code,
        public string $message,
    ) {
    }
}
```

CSV全体エラーと
行エラーで
同じ構造を使用してよい。

---

#### 7.30.35 Preview Row DTO

各CSV行の
プレビュー結果は、
専用DTOとして表現する。

概念例：

```php
final readonly class MonthEndHoldingValueCsvPreviewRow
{
    public function __construct(
        public int $rowNumber,
        public ?string $assetAccountName,
        public ?string $holdingAssetName,
        public ?int $value,

        /** @var CsvPreviewError[] */
        public array $errors,
    ) {
    }
}
```

内部の

- `asset_account_id`
- `holding_asset_id`
- `month_end_asset_snapshot_id`

は
Preview Row DTOへ
公開しない。

---

#### 7.30.36 Preview DTO

プレビュー結果全体は、
専用DTOとして表現する。

概念例：

```php
final readonly class MonthEndHoldingValueCsvPreview
{
    public function __construct(
        public ?string $targetYearMonth,
        public bool $canImport,

        /** @var CsvPreviewError[] */
        public array $errors,

        /** @var MonthEndHoldingValueCsvPreviewRow[] */
        public array $rows,
    ) {
    }
}
```

Eloquent Modelや
Query結果を
そのままResponderへ渡さない。

---

#### 7.30.37 canImportの判定

`canImport`は、
CSV全体エラーおよび
各行エラーから
UseCaseで判定する。

概念例：

```php
$hasGlobalErrors =
    count(
        $globalErrors,
    ) > 0;

$hasRowErrors =
    collect(
        $rows,
    )->contains(
        static fn (
            MonthEndHoldingValueCsvPreviewRow $row,
        ): bool =>
            count(
                $row->errors,
            ) > 0,
    );

$canImport =
    ! $hasGlobalErrors
    && ! $hasRowErrors;
```

正常行が存在しても、
エラーが1件でも存在すれば

```text
canImport = false
```

とする。

---

#### 7.30.38 複数エラー収集

CSV構造が正常な場合は、
可能な範囲で
複数エラーを収集する。

例えば、

```text
2行目
ASSET_ACCOUNT_NOT_FOUND

4行目
HOLDING_ASSET_NOT_FOUND

6行目
INVALID_VALUE
```

を
1回のレスポンスで
返却できるようにする。

1件目の行エラーで
例外を送出して
全体を終了する方式は
基本としない。

---

#### 7.30.39 HTTPエラーとプレビューエラーの分離

以下は、
HTTPエラーとして扱う。

- 利用者コンテキスト不正
- `file`未指定
- アップロードファイル不正
- ファイルサイズ超過
- CSV解析不能
- CSVヘッダー不正
- 想定外例外

一方、
以下は
プレビュー結果内の
業務エラーとして扱う。

- データ行0件
- `target_year_month`不正
- 対象年月混在
- `asset_account_name`不正
- 資産口座不存在
- 残高記録単位不一致
- `holding_asset_name`不正
- 保有商品不存在
- 対象年月時点の保有商品無効
- `value`不正
- CSV内重複
- 確定済み月末資産状況
- 既存商品別月末評価額

この2種類を
明確に分離する。

---

#### 7.30.40 API Resource

Preview DTOを、
専用API Resourceによって
APIレスポンス形式へ変換する。

概念例：

```php
final class MonthEndHoldingValueCsvPreviewResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'targetYearMonth'
                => $this->targetYearMonth,

            'canImport'
                => $this->canImport,

            'errors'
                => CsvPreviewErrorResource::collection(
                    $this->errors,
                ),

            'rows'
                => MonthEndHoldingValueCsvPreviewRowResource::collection(
                    $this->rows,
                ),
        ];
    }
}
```

JSONフィールド名は、
API共通方針に従って
camelCaseとする。

---

#### 7.30.41 Row Resource

行単位の結果も、
専用Resourceへ変換する。

概念例：

```php
final class MonthEndHoldingValueCsvPreviewRowResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'rowNumber'
                => $this->rowNumber,

            'assetAccountName'
                => $this->assetAccountName,

            'holdingAssetName'
                => $this->holdingAssetName,

            'value'
                => $this->value,

            'errors'
                => CsvPreviewErrorResource::collection(
                    $this->errors,
                ),
        ];
    }
}
```

---

#### 7.30.42 Error Resource

プレビュー用エラーも、
専用Resourceへ変換する。

概念例：

```php
final class CsvPreviewErrorResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'code'
                => $this->code,

            'message'
                => $this->message,
        ];
    }
}
```

CSV内部で使用した
技術的な情報を
公開しない。

---

#### 7.30.43 返却しない情報

API Resourceでは、
以下の内部情報を返却しない。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`
- `asset_accounts.balance_recording_unit`
- `month_end_asset_snapshots.confirmed`
- `created_at`
- `updated_at`
- `asset_account_available_settings`

Queryで取得した
Eloquent Modelを
そのままJSON化しない。

---

#### 7.30.44 Responder

Responderは、
生成済みPreview DTOを受け取り、
API共通の成功Envelope形式へ変換する。

概念例：

```php
final class MonthEndHoldingValueCsvPreviewResponder
{
    public function ok(
        MonthEndHoldingValueCsvPreview $preview,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new MonthEndHoldingValueCsvPreviewResource(
                        $preview,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの
共通項目は、
API共通レスポンス処理に従う。

---

#### 7.30.45 Responderの責務

Responderでは、
以下を行わない。

- CSV解析
- CSVヘッダー検証
- 行入力値検証
- 対象年月特定
- CSV内重複判定
- 資産口座検索
- 保有商品検索
- 有効性判定
- snapshot検索
- 既存評価額検索
- `canImport`判定

Responderは、
Preview DTOを
HTTPレスポンスへ変換することに
責務を限定する。

---

#### 7.30.46 Repository

CSV-005では、
Repositoryを使用しない。

本APIは、
業務データを更新しない
参照・検証専用APIであるため、

- 資産口座取得
- 保有商品取得
- 利用可能設定取得
- snapshot取得
- 既存商品別月末評価額取得

はQueryクラスが担当する。

登録・更新・削除処理は
存在しないため、
CSV-005専用Repositoryは
作成しない。

---

#### 7.30.47 トランザクション

CSV-005では、
業務データを更新しないため、
明示的な
`DB::transaction()`を使用しない。

以下のような実装は
行わない。

```php
DB::transaction(
    function () {
        // CSVプレビューのみ
    },
);
```

CSV-006で
登録時に最新状態を再検証するため、
CSV-005で
長時間の読み取りトランザクションを
保持しない。

---

#### 7.30.48 ロック

CSV-005では、
行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

CSV-005からCSV-006までの間、
DB状態をロックして
登録可否を保証しない。

---

#### 7.30.49 プレビュー結果の保存

Phase1では、
Preview DTOや
プレビュー済みCSVを
データベースへ保存しない。

以下の方式は採用しない。

```text
CSV-005
    ↓
previewId生成
    ↓
DB保存
    ↓
CSV-006でpreviewId指定
```

CSV-006では、
CSVファイルを再送し、
同じ業務ルールを
再検証する。

---

#### 7.30.50 キャッシュ

Phase1では、
CSV-005専用の
サーバー側アプリケーションキャッシュを
使用しない。

同じCSVでも、

- 資産口座の状態
- 保有商品の状態
- 利用可能設定
- snapshotの`confirmed`
- 既存商品別月末評価額

が変更されれば、
プレビュー結果も変化する。

そのため、
毎回最新状態を参照する。

---

#### 7.30.51 ログ

CSV-005では、
必要に応じて
以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
fileName
rowCount
canImport
errorCount
```

`apiId`は、

```text
CSV-005
```

とする。

以下は、
不要にログへ出力しない。

- CSVファイル全文
- 全`value`
- 全資産口座名
- 全保有商品名

---

#### 7.30.52 例外変換

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `file`未指定・ファイル検証不正 | `VALIDATION_ERROR` |
| CSV解析不能 | `INVALID_CSV_FORMAT` |
| CSVヘッダー不正 | `INVALID_CSV_FORMAT` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

CSV行単位や
CSV全体の業務エラーは、
原則として例外へ変換しない。

Preview DTOの

```text
errors
rows[].errors
```

へ格納する。

---

#### 7.30.53 想定外例外

想定外の例外は、
API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、
以下を含めない。

- SQL
- テーブル名
- カラム名
- PostgreSQL内部エラー
- 制約名
- PHP内部エラー
- Laravel内部例外メッセージ
- スタックトレース
- サーバーファイルパス

詳細情報は、
サーバーログへ記録する。

---

#### 7.30.54 テスト実装方針

Laravel側では、
Feature Testを中心として
CSV-005のAPI契約および
プレビューフロー全体を確認する。

主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `404 Not Found`
- `422 Unprocessable Entity`
- `500 Internal Server Error`
- `file`必須
- ファイル形式
- ファイルサイズ
- CSVヘッダー
- CSVヘッダー順序
- データ行0件
- `target_year_month`必須
- `target_year_month`形式
- 1ファイル1対象年月
- `asset_account_name`必須
- 資産口座不存在
- 他利用者の同名資産口座を参照しないこと
- 残高記録単位
- `holding_asset_name`必須
- 保有商品不存在
- 他資産口座の同名保有商品を参照しないこと
- 他利用者の同名保有商品を参照しないこと
- 対象年月時点の有効性
- `value`必須
- `value`整数
- `value`0以上
- `value = 0`
- CSV内重複
- snapshot不存在
- snapshot未確定
- snapshot確定済み
- 既存商品別月末評価額
- 複数エラー収集
- `rowNumber`
- `targetYearMonth`
- `canImport`
- 業務データ非更新
- 冪等性

---

#### 7.30.55 CSV ParserのUnit Test

CSV Parserについて、
以下を確認する。

- 正しいヘッダーを取得できること
- データ行を正しく解析できること
- CSV上の行番号を保持できること
- BOMを仕様どおり処理できること
- 完全な空行を仕様どおり扱うこと
- `,,,`をデータ行として扱えること
- CSV解析不能時に適切な例外となること
- ヘッダー不正時に後続の業務検証へ進まないこと

---

#### 7.30.56 CSV ValidatorのUnit Test

CSV Validatorについて、
以下を個別に確認する。

```text
target_year_month
    required
    YYYY-MM

asset_account_name
    required

holding_asset_name
    required

value
    required
    integer
    min:0
```

特に、

```text
value = 0
```

を正常値として
必ず確認する。

---

#### 7.30.57 業務検証のUnit Test

CSV内容と
業務データを照合する
検証ロジックについて、
以下を確認する。

```text
資産口座あり
    → 正常

資産口座なし
    → ASSET_ACCOUNT_NOT_FOUND

口座単位資産口座
    → BALANCE_RECORDING_UNIT_MISMATCH

保有商品あり
    → 正常

保有商品なし
    → HOLDING_ASSET_NOT_FOUND

別資産口座にのみ同名商品あり
    → HOLDING_ASSET_NOT_FOUND

対象年月時点で無効
    → HOLDING_ASSET_NOT_AVAILABLE

snapshotなし
    → それだけではエラーにしない

snapshot未確定
    → 登録可能

snapshot確定済み
    → MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED

既存評価額あり
    → MONTH_END_HOLDING_VALUE_ALREADY_EXISTS

CSV内重複
    → DUPLICATE_HOLDING_ASSET_IN_CSV
```

---

#### 7.30.58 QueryのDatabase Test

AssetAccountQueryでは、
以下を確認する。

```text
user_id一致
+
name一致
+
deleted_at IS NULL
    ↓
取得できる
```

他利用者の
同名資産口座は
取得できないことを確認する。

HoldingAssetQueryでは、

```text
asset_account_id一致
+
name一致
+
deleted_at IS NULL
    ↓
取得できる
```

ことを確認する。

他資産口座・他利用者の
同名保有商品が
混入しないことも確認する。

MonthEndAssetSnapshotQueryでは、

```text
user_id
+
target_year_month
```

で
正しいsnapshotだけを
取得できることを確認する。

MonthEndHoldingValueQueryでは、

```text
snapshotId
+
holdingAssetIds
```

で
既存評価額を
正しく取得できることを確認する。

---

#### 7.30.59 UseCaseのUnit Test

UseCaseについては、
Parser、
Validator、
Queryの結果を組み合わせて
正しいPreview DTOを
生成できることを確認する。

概念的には、

```text
ParsedCsv
+
CsvRowValidationResult
+
AssetAccountQuery結果
+
HoldingAssetQuery結果
+
AvailableSettingQuery結果
+
SnapshotQuery結果
+
ExistingValueQuery結果
    ↓
PreviewMonthEndHoldingValueCsvUseCase
    ↓
MonthEndHoldingValueCsvPreview
```

を確認する。

特に、

```text
エラーなし
    ↓
canImport = true
```

```text
CSV全体エラーあり
    ↓
canImport = false
```

```text
行エラーあり
    ↓
canImport = false
```

```text
複数行エラーあり
    ↓
複数エラーを保持
```

となることを確認する。

また、
UseCase実行によって
業務データが
更新されないことも確認する。

---

#### 7.30.60 CSV-006との整合性テスト

CSV-005とCSV-006で、
同一利用者、
同一DB状態、
同一CSVに対する
登録可否判定が一致することを確認する。

概念的には、

```text
CSV-005
canImport = true

同一DB状態
+
同一CSV
    ↓
CSV-006
再検証
    ↓
登録可能
```

となることを確認する。

ただし、
CSV-005とCSV-006の間で
DB状態が変更された場合は、
結果が変化してよい。

例えば、

```text
CSV-005
confirmed = false
    ↓
canImport = true

その後
confirmed = true

CSV-006
    ↓
登録不可
```

となることを
正常な挙動として扱う。

---

### 7.31 React・TypeScriptでの利用

CSV-005は、
利用者が選択した
商品別月末評価額CSVを送信し、
CSV-006で登録する前に
登録予定内容とエラー内容を確認するために使用する。

フロントエンドでは、
CSVファイルを`File`として保持し、
`FormData`へ設定して
APIへ送信する。

概念的な利用フローは、
以下とする。

```text
CSV-004
テンプレート取得
    ↓
利用者がCSV編集
    ↓
CSVファイル選択
    ↓
CSV-005
プレビュー
    ↓
canImport確認
    ├─ false
    │     ↓
    │   エラー表示
    │
    └─ true
          ↓
       登録ボタン有効化
          ↓
       CSV-006
```

---

#### 7.31.1 TypeScript型

CSVプレビュー結果は、
以下のような型として扱う。

概念例：

```ts
export type CsvPreviewError = {
  code: string;
  message: string;
};

export type MonthEndHoldingValueCsvPreviewRow = {
  rowNumber: number;
  assetAccountName: string | null;
  holdingAssetName: string | null;
  value: number | null;
  errors: CsvPreviewError[];
};

export type MonthEndHoldingValueCsvPreview = {
  targetYearMonth: string | null;
  canImport: boolean;
  errors: CsvPreviewError[];
  rows: MonthEndHoldingValueCsvPreviewRow[];
};

export type PreviewMonthEndHoldingValueCsvResponse = {
  data: MonthEndHoldingValueCsvPreview;
  requestId: string;
};
```

API共通Envelopeの
正式な型定義が存在する場合は、
共通型を利用する。

概念例：

```ts
export type PreviewMonthEndHoldingValueCsvResponse =
  ApiResponse<MonthEndHoldingValueCsvPreview>;
```

---

#### 7.31.2 CSVファイルの保持

CSVファイルは、
ブラウザの`File`として保持する。

概念例：

```ts
const [file, setFile] =
  useState<File | null>(
    null,
  );
```

CSV-005実行後も、
CSV-006で
同じCSVファイルを再送するため、
登録処理が完了するまで
`File`を保持する。

---

#### 7.31.3 ファイル選択

ファイル選択UIでは、
CSVファイルを
選択できるようにする。

概念例：

```tsx
<input
  type="file"
  accept=".csv,text/csv"
  onChange={(event) => {
    const selectedFile =
      event.target.files?.[0]
      ?? null;

    setFile(
      selectedFile,
    );
  }}
/>
```

`accept`属性は、
利用者のファイル選択を
補助するために使用する。

バックエンド側の
ファイル形式検証を
代替するものではない。

---

#### 7.31.4 FormData

CSVファイルは、
`FormData`へ
`file`という項目名で設定する。

概念例：

```ts
const formData =
  new FormData();

formData.append(
  'file',
  file,
);
```

以下のような
業務データは
`FormData`へ追加しない。

```text
userId
targetYearMonth
assetAccountId
holdingAssetId
value
confirmed
```

`userId`は
`X-User-Id`で扱い、
その他の情報は
CSV内容から取得する。

---

#### 7.31.5 Content-Type

`multipart/form-data`の
`Content-Type`は、
ブラウザまたは
HTTP Clientに設定させる。

以下のように
手動設定しない。

```ts
headers: {
  'Content-Type':
    'multipart/form-data',
}
```

`FormData`を送信することで、
必要なboundaryを
HTTP Client側に生成させる。

---

#### 7.31.6 API Client

API呼び出しは、
CSVファイルを受け取る
専用関数として定義する。

概念例：

```ts
export const previewMonthEndHoldingValueCsv =
  async (
    file: File,
  ): Promise<MonthEndHoldingValueCsvPreview> => {
    const formData =
      new FormData();

    formData.append(
      'file',
      file,
    );

    const response =
      await apiClient.post<
        PreviewMonthEndHoldingValueCsvResponse
      >(
        '/api/v1/month-end-holding-values/imports/preview',
        formData,
      );

    return response.data.data;
  };
```

CSV解析や
登録可否判定を
API Client内で行わない。

---

#### 7.31.7 X-User-Id

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

CSV-005専用処理で
`userId`を
`FormData`へ追加しない。

---

#### 7.31.8 Mutationとして扱う

CSV-005は、
業務データを変更しないが、
利用者操作によって
CSVファイルを送信し、
明示的にプレビューを実行するAPIである。

TanStack Queryを使用する場合は、
Mutationとして扱ってよい。

概念例：

```ts
export const usePreviewMonthEndHoldingValueCsv =
  () =>
    useMutation({
      mutationFn:
        previewMonthEndHoldingValueCsv,
    });
```

画面表示時に
自動実行するQueryとしては
扱わない。

---

#### 7.31.9 プレビュー実行

利用者が
CSVファイルを選択した後、
プレビューボタンから
CSV-005を実行する。

概念例：

```ts
const previewMutation =
  usePreviewMonthEndHoldingValueCsv();

const handlePreview =
  (): void => {
    if (file === null) {
      return;
    }

    previewMutation.mutate(
      file,
    );
  };
```

---

#### 7.31.10 プレビューボタン

CSVファイルが
選択されていない場合、
プレビューボタンを
無効化してよい。

また、
プレビュー実行中も
二重送信防止のため
無効化してよい。

概念例：

```tsx
<button
  type="button"
  disabled={
    file === null
    || previewMutation.isPending
  }
  onClick={
    handlePreview
  }
>
  {previewMutation.isPending
    ? '確認中...'
    : 'プレビュー'}
</button>
```

ただし、
`file`必須チェックは
バックエンドでも行う。

---

#### 7.31.11 プレビュー結果の保持

CSV-005成功後は、
プレビュー結果を
画面Stateへ保持してよい。

概念例：

```ts
const [preview, setPreview] =
  useState<
    MonthEndHoldingValueCsvPreview
    | null
  >(null);
```

Mutationの
`data`をそのまま利用する場合は、
別Stateを作成しなくてもよい。

同じ情報を
複数のStateへ
不要に二重管理しない。

---

#### 7.31.12 targetYearMonth

`targetYearMonth`は、
CSV全体から特定された
対象年月を表す。

正常例：

```json
{
  "targetYearMonth": "2026-07"
}
```

対象年月が混在しているなど、
単一の対象年月として
特定できない場合は、

```json
{
  "targetYearMonth": null
}
```

となる。

そのため、
TypeScriptでは、

```ts
targetYearMonth:
  string | null;
```

として扱う。

---

#### 7.31.13 canImport

`canImport`は、
プレビュー時点で
CSV全体を登録可能かを表す。

```text
true
    → プレビュー時点では登録可能

false
    → 登録不可となるエラーあり
```

登録ボタンの制御には、
この値を使用する。

フロントエンドで
`errors`配列を独自に解析して
登録可否を再計算しない。

---

#### 7.31.14 canImportは登録成功保証ではない

`canImport = true`は、
CSV-006が
必ず成功することを意味しない。

CSV-005実行後に、

- 月末資産状況が確定される
- 商品別月末評価額が別処理で登録される
- 資産口座の状態が変更される
- 保有商品の状態が変更される

可能性がある。

そのため、
CSV-006で
業務エラーが返却される可能性を
考慮する。

---

#### 7.31.15 CSV全体エラー

`data.errors`には、
CSV全体に関係する
エラーを表示する。

例えば、

```text
CSV_DATA_REQUIRED
MULTIPLE_TARGET_YEAR_MONTHS
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

などを対象とする。

概念例：

```tsx
{preview.errors.length > 0 && (
  <ul>
    {preview.errors.map(
      (error) => (
        <li key={error.code}>
          {error.message}
        </li>
      ),
    )}
  </ul>
)}
```

---

#### 7.31.16 rows

`rows`には、
CSV各行の
プレビュー結果が返却される。

概念例：

```tsx
<tbody>
  {preview.rows.map(
    (row) => (
      <CsvPreviewRow
        key={row.rowNumber}
        row={row}
      />
    ),
  )}
</tbody>
```

バックエンドから返却された
CSV上の並び順を維持して
表示する。

---

#### 7.31.17 rowNumber

`rowNumber`は、
CSVファイル上の
実際の行番号として扱う。

ヘッダーを1行目とするため、
最初のデータ行は、

```text
rowNumber = 2
```

となる。

画面では、

```text
2行目
3行目
4行目
```

のように
利用者へ表示してよい。

---

#### 7.31.18 assetAccountName

`assetAccountName`は、
CSVへ入力された
資産口座名を表示する。

概念例：

```tsx
<td>
  {row.assetAccountName
    ?? '-'}
</td>
```

内部の

```text
asset_accounts.id
```

は
フロントエンドで扱わない。

---

#### 7.31.19 holdingAssetName

`holdingAssetName`は、
CSVへ入力された
保有商品名を表示する。

概念例：

```tsx
<td>
  {row.holdingAssetName
    ?? '-'}
</td>
```

内部の

```text
holding_assets.id
```

は
フロントエンドで扱わない。

---

#### 7.31.20 value

`value`は、
正常に整数として解析できた場合、
`number`として扱う。

```ts
value:
  number | null;
```

表示時は、
日本円として
フォーマットしてよい。

概念例：

```tsx
<td>
  {row.value === null
    ? '-'
    : formatYen(
        row.value,
      )}
</td>
```

---

#### 7.31.21 0円とnull

`value = 0`は、
正常な業務値である。

一方、

```text
value = null
```

は、
正常な整数として
扱えなかった状態などを表す。

そのため、
以下のような
truthy / falsy判定を行わない。

```tsx
{row.value
  ? formatYen(
      row.value,
    )
  : '-'}
```

上記では、
0円まで`-`となる。

以下のように
明示的に`null`を判定する。

```tsx
{row.value === null
  ? '-'
  : formatYen(
      row.value,
    )}
```

---

#### 7.31.22 行エラー

各行の`errors`には、
そのCSV行に関する
入力・業務エラーが設定される。

例えば、

```text
ASSET_ACCOUNT_NOT_FOUND
BALANCE_RECORDING_UNIT_MISMATCH
HOLDING_ASSET_NOT_FOUND
HOLDING_ASSET_NOT_AVAILABLE
INVALID_VALUE
DUPLICATE_HOLDING_ASSET_IN_CSV
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

などを対象とする。

概念例：

```tsx
{row.errors.length > 0 && (
  <ul>
    {row.errors.map(
      (error) => (
        <li key={error.code}>
          {error.message}
        </li>
      ),
    )}
  </ul>
)}
```

1行に
複数エラーが
存在する可能性があるため、
1行1エラーを前提としない。

---

#### 7.31.23 エラー行の表示

`row.errors.length > 0`
の場合は、
該当行を
視覚的に強調してよい。

例えば、

- 背景色
- アイコン
- 枠線
- エラーメッセージ

などを使用する。

具体的な表示方法は、
画面設計に従う。

---

#### 7.31.24 200 OKかつcanImport=false

CSV-005では、
CSV内容に
入力・業務エラーが存在していても、
プレビュー処理自体が
正常に完了した場合は、

```text
200 OK
```

となる。

そのため、
HTTPステータスだけで
登録可能と判断しない。

必ず、

```ts
preview.canImport
```

を確認する。

---

#### 7.31.25 HTTPエラー

以下は、
プレビュー結果ではなく
HTTPエラーとして扱う。

- `X-User-Id`不正
- `file`未指定
- ファイル形式不正
- ファイルサイズ超過
- CSV解析不能
- CSVヘッダー不正
- サーバー内部エラー

これらの場合は、
`MonthEndHoldingValueCsvPreview`として
処理しない。

---

#### 7.31.26 VALIDATION_ERROR

`VALIDATION_ERROR`の場合は、
主にアップロードファイルに関する
入力不正として扱う。

例えば、

- ファイル未指定
- 許可されないファイル形式
- ファイルサイズ超過

などを表示する。

フロントエンドで
同じ検証を完全に
再実装する必要はない。

UX向上を目的とした
事前チェックは行ってよい。

---

#### 7.31.27 INVALID_CSV_FORMAT

`INVALID_CSV_FORMAT`の場合は、
CSVをプレビュー可能な形式として
解析できなかったことを表示する。

例えば、

```text
CSVファイルの形式を確認してください。
商品別月末評価額CSVテンプレートを使用して
再度作成してください。
```

のように案内する。

必要に応じて、
CSV-004の
テンプレート取得導線を表示する。

---

#### 7.31.28 利用者関連エラー

以下のエラーは、
API共通方針に従って処理する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
```

CSV-005画面だけで
独自処理を実装しない。

必要に応じて、
利用者選択画面へ戻す。

---

#### 7.31.29 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`の場合は、
共通サーバーエラーとして扱う。

CSV登録処理へは進まない。

必要に応じて、
同じCSVを
再プレビューできる導線を表示する。

---

#### 7.31.30 エラーコードと表示文言

プレビュー内の
`error.code`は、
画面処理の判別に使用できる。

ただし、
フロントエンドで
業務判定そのものを
再実装しない。

バックエンドから
`message`が返却される場合は、
基本的にその文言を表示してよい。

フロントエンド側で
文言を管理する場合は、
エラーコードとの対応を
共通定義へ集約する。

---

#### 7.31.31 ファイル変更時

利用者が
プレビュー後に
別のCSVファイルを選択した場合は、
以前のプレビュー結果を
無効化する。

概念例：

```ts
const handleFileChange =
  (
    nextFile: File | null,
  ): void => {
    setFile(
      nextFile,
    );

    setPreview(
      null,
    );
  };
```

新しいCSVファイルに対して、
以前の

```text
canImport = true
```

を流用してはならない。

---

#### 7.31.32 再プレビュー

CSV-005は、
業務データを変更しないため、
同じCSVまたは
修正したCSVを
再度プレビューできる。

概念的な操作は、
以下とする。

```text
CSV-005
    ↓
canImport = false
    ↓
利用者がCSV修正
    ↓
ファイル再選択
    ↓
CSV-005
    ↓
再プレビュー
```

---

#### 7.31.33 CSV-006登録ボタン

CSV-006の登録ボタンは、
少なくとも以下を
すべて満たす場合に
有効化する。

```text
file != null
AND
preview != null
AND
preview.canImport = true
```

概念例：

```ts
const canImport =
  file !== null
  && preview !== null
  && preview.canImport;
```

```tsx
<button
  type="button"
  disabled={!canImport}
>
  登録する
</button>
```

---

#### 7.31.34 CSV-006へ同じFileを送信する

CSV-006では、
CSV-005で使用した
同じ`File`を再送する。

```text
CSV-005
File A
    ↓
canImport = true
    ↓
CSV-006
File A
```

プレビュー結果から
CSV内容を再構築して
CSV-006へ送信しない。

---

#### 7.31.35 CSV-006成功後

CSV-006が成功した場合は、
保持していた

- CSVファイル
- プレビュー結果

をクリアしてよい。

概念例：

```ts
setFile(
  null,
);

setPreview(
  null,
);
```

また、
関連する

- 商品別月末評価額
- 月末資産状況

のQuery Cacheを
invalidateする。

具体的なinvalidate対象は、
フロントエンド共通設計に従う。

---

#### 7.31.36 CSV-006失敗時

CSV-005で

```text
canImport = true
```

となっていても、
CSV-006で
登録不可となる可能性がある。

その場合は、
CSV-006のエラーを表示し、
必要に応じて
CSV-005を再実行する。

CSV-005の結果だけを理由に
CSV-006の業務エラーを
無視しない。

---

#### 7.31.37 CSVをReact側で正式解析しない

Phase1では、
CSV内容の正式な解析は
バックエンド側で行う。

React側で
CSV Parserライブラリを利用して、
CSV-005と同じ

- ヘッダー検証
- 対象年月検証
- 資産口座検証
- 保有商品検証
- 評価額検証
- CSV内重複検証

を二重実装しない。

フロントエンドは、
CSV-005の結果を
表示・操作制御へ使用する。

---

#### 7.31.38 canImportをReact側で再計算しない

以下のような
独自判定は行わない。

```ts
const canImport =
  preview.errors.length === 0
  && preview.rows.every(
    (row) =>
      row.errors.length === 0,
  );
```

正式な登録可否は、
バックエンドが返却する

```text
canImport
```

を使用する。

これにより、
登録可否判定を
バックエンドへ集約する。

---

### 7.32 設計上の補足

#### 7.32.1 POSTを採用する理由

CSV-005は、
業務データを更新しない。

ただし、
CSVファイルを
リクエストボディとして送信し、
サーバー側で解析・検証処理を行う。

そのため、
HTTPメソッドには
`POST`を採用する。

---

#### 7.32.2 プレビュー専用APIを分ける理由

CSV-005とCSV-006を
同じAPIへ統合し、

```text
preview = true
```

などのフラグで
処理を切り替えない。

以下のように、
副作用の有無を
エンドポイントとして分離する。

```text
CSV-005
/imports/preview
    ↓
検証のみ

CSV-006
/imports
    ↓
検証
+
登録
```

これにより、
APIの責務を明確にする。

---

#### 7.32.3 canImportを返却する理由

CSVプレビューでは、
CSV全体エラーと
複数の行エラーが
存在する可能性がある。

フロントエンドが
これらを走査して
登録可否を独自判定する必要がないよう、
バックエンドから

```text
canImport
```

を返却する。

登録可否判定の
責務をバックエンドへ集約する。

---

#### 7.32.4 業務エラーを200 OKで返す理由

CSVを正常に解析でき、
プレビュー処理が
最後まで完了している場合、
API処理そのものは成功している。

例えば、

- 資産口座不存在
- 保有商品不存在
- `value`不正
- CSV内重複
- 確定済み月末資産状況
- 既存商品別月末評価額

などは、
CSVプレビューによって
利用者へ発見・提示すること自体が
正常なユースケースである。

そのため、

```text
200 OK
canImport = false
```

として扱う。

---

#### 7.32.5 CSV構造不正を422とする理由

CSVヘッダー不正や
CSV解析不能の場合は、
各データ行を
正しく解釈できない。

そのため、
プレビュー結果ではなく
入力形式が成立していない状態として、

```text
422 Unprocessable Entity
```

を返却する。

---

#### 7.32.6 複数エラーを返却する理由

CSVでは、
複数行をまとめて
登録対象とする。

最初の1件だけを返却すると、

```text
プレビュー
    ↓
1件修正
    ↓
再プレビュー
    ↓
次の1件修正
```

という操作が
繰り返し必要になる。

可能な範囲で
複数エラーをまとめて返却し、
利用者が一度に
CSVを修正できるようにする。

---

#### 7.32.7 rowNumberを返却する理由

エラー内容だけでは、
利用者がCSV上の
該当行を特定しにくい。

そのため、
CSV上の実際の行番号を

```text
rowNumber
```

として返却する。

---

#### 7.32.8 資産口座名・保有商品名を返却する理由

商品別月末評価額CSVでは、
利用者が確認したい単位は、

```text
どの資産口座の
どの保有商品か
```

である。

そのため、

```text
assetAccountName
holdingAssetName
```

を
プレビュー結果へ返却する。

---

#### 7.32.9 内部IDを返却しない理由

CSV利用者が
確認する必要があるのは、

- CSV上の行番号
- 資産口座名
- 保有商品名
- 商品別月末評価額
- エラー内容

である。

そのため、

```text
asset_accounts.id
holding_assets.id
month_end_asset_snapshots.id
month_end_holding_values.id
```

などの内部IDは返却しない。

---

#### 7.32.10 balance_recording_unitを返却しない理由

残高記録単位は、
CSV-005の
登録可否判定に使用する内部情報である。

口座単位の資産口座が
指定された場合は、
エラーとして通知すればよい。

React側が

```text
balanceRecordingUnit
```

を参照して
独自に登録可否を判定する必要はない。

---

#### 7.32.11 confirmedを返却しない理由

確定済みのため
CSV登録できない場合は、
CSV全体エラーとして
その状態を通知する。

React側が

```text
confirmed = true
```

を参照して
登録可否を独自判定する必要はない。

そのため、
`confirmed`自体は返却しない。

---

#### 7.32.12 プレビュー結果を保存しない理由

Phase1では、
CSV-005からCSV-006までの間に
サーバー側で
プレビュー状態を保存しない。

これにより、

- `previewId`
- preview token
- 有効期限
- 一時ファイル
- 一時データ削除

などの管理を不要とする。

CSV-006では、
CSVファイルを再送して
最新状態で再検証する。

---

#### 7.32.13 CSV-006で再検証する理由

CSV-005とCSV-006の間に、
業務データの状態が
変化する可能性がある。

例えば、

```text
CSV-005
confirmed = false
    ↓
canImport = true

別処理
confirmed = true
    ↓

CSV-006
```

となり得る。

そのため、
CSV-005の判定結果を
登録権利として扱わず、
CSV-006で必ず
最新状態を再検証する。

---

#### 7.32.14 ロックを保持しない理由

CSV-005実行後、
利用者がプレビュー画面を
長時間確認する可能性がある。

CSV-005からCSV-006まで
データベースロックを保持すると、
他の月末資産関連処理を
長時間ブロックする可能性がある。

そのため、
CSV-005ではロックを保持せず、
CSV-006実行時に
再検証する。

---

#### 7.32.15 React側で業務検証を再実装しない理由

CSV登録可否に関する
正式な業務ルールは、
バックエンドで管理する。

React側でも
同じルールを実装すると、

```text
バックエンド
    → 登録不可

React
    → 登録可能
```

などの
判定差異が発生する可能性がある。

そのため、
Reactは
CSV-005の結果を表示し、
画面操作を制御する役割に限定する。

---

#### 7.32.16 0円とnullを区別する理由

`value = 0`は、
正常な商品別月末評価額である。

一方、

```text
value = null
```

は、
正常な整数として
解析できなかった場合などを表す。

そのため、
TypeScriptでも、

```ts
value:
  number | null;
```

として
明確に区別する。

---

#### 7.32.17 CSV-004との仕様共通化

CSV-004で提供する
テンプレートと、
CSV-005で受け付ける
CSV形式は同一とする。

React側では、
CSVヘッダーを
独自定義しない。

正式なCSV仕様は、
バックエンドの
共通CSV Definitionへ集約する。

---

#### 7.32.18 CSV-006との検証ロジック共通化

CSV-005とCSV-006では、
CSV解析および
登録可否判定ロジックを
可能な限り共通化する。

同一システム状態かつ
同一CSVであれば、

```text
CSV-005
canImport = true

CSV-006
再検証
    ↓
登録可能
```

となることを基本とする。

ただし、
CSV-005とCSV-006の間で
業務データが変更された場合は、
結果が変わることを許容する。

---

#### 7.32.19 Idempotency-Keyを使用しない理由

CSV-005は
POST APIだが、
業務データを変更しない。

複数回実行しても
重複登録などの副作用が
発生しないため、

```text
Idempotency-Key
```

は使用しない。

---

#### 7.32.20 キャッシュしない理由

同じCSVファイルでも、

- 資産口座の状態
- 保有商品の状態
- 対象年月時点の利用可能状態
- 月末資産状況
- `confirmed`
- 既存商品別月末評価額

が変化すれば、
プレビュー結果も変化する。

そのため、
Phase1では
CSVプレビュー結果を
サーバー側でキャッシュしない。

毎回、
最新の業務データを使用して
検証する。

---

### 7.33 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [CSVインポートAPI詳細](./csv-imports.md)
- [資産口座API詳細](./asset-accounts.md)
- [保有商品API詳細](./holding-assets.md)
- [月末資産状況API詳細](./month-end-asset-snapshots.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)

---

## 8. CSV-006 商品別月末評価額CSV登録

### 8.1 概要

操作対象となる利用者について、
アップロードされた
商品別月末評価額CSVの内容を再検証し、
検証に成功した場合に
商品別月末評価額を一括登録する。

本APIでは、
CSV-005 商品別月末評価額CSVプレビューで
登録可能と判定されたCSVであっても、
CSV-006実行時点の最新状態をもとに
再度検証する。

主に以下を確認する。

- CSVファイル形式
- CSVヘッダー
- 対象年月
- 資産口座名
- 保有商品名
- 商品別月末評価額
- 1ファイル内の対象年月統一
- 資産口座の存在
- 資産口座の利用者境界
- 残高記録単位
- 保有商品の存在
- 保有商品と資産口座の関連
- 対象年月時点での保有商品の有効性
- 月末資産状況の状態
- 既存商品別月末評価額との重複
- CSV内の重複

すべての検証に成功した場合のみ、
CSV内の商品別月末評価額を
一括登録する。

CSV内に
登録できないデータが
1件でも存在する場合は、
正常な行だけを
部分登録してはならない。

登録処理は、
トランザクション内で実行し、
途中でエラーが発生した場合は
CSV全体の登録を
ロールバックする。

概念的な処理は、
以下とする。

```text
CSVファイル受信
    ↓
ファイル検証
    ↓
CSVヘッダー検証
    ↓
CSV行解析
    ↓
入力値検証
    ↓
業務ルール再検証
    ↓
登録可否判定
    ↓
トランザクション開始
    ↓
必要に応じて
月末資産状況作成
    ↓
商品別月末評価額一括登録
    ↓
commit
    ↓
登録結果返却
```

本APIは、
CSV-005のプレビュー結果を
そのまま信頼して登録するAPIではない。

CSV-005とCSV-006の間で
業務データの状態が
変更される可能性があるため、
CSV-006では
必ず最新状態を再検証する。

---

### 8.2 ユースケース

利用者は、
CSV-005 商品別月末評価額CSVプレビューで
CSV内容を確認した後、
問題がなければ
本APIを実行して
商品別月末評価額を一括登録する。

基本的な利用フローは、
以下とする。

```text
CSV-004
商品別月末評価額CSVテンプレート取得
    ↓
利用者がCSVへデータを入力
    ↓
CSV-005
商品別月末評価額CSVプレビュー
    ↓
canImport = true
    ↓
利用者が登録を実行
    ↓
CSV-006
商品別月末評価額CSV登録
    ↓
CSV内容・業務状態を再検証
    ↓
一括登録
```

CSV-005で

```text
canImport = true
```

となっていても、
CSV-006実行時点で

- 月末資産状況が確定された
- 商品別月末評価額が別処理で登録された
- 資産口座の状態が変更された
- 保有商品の状態が変更された

などの場合は、
登録できない。

本APIでは、
CSV-005で使用した
同一のCSVファイルを
再送することを前提とする。

プレビュー結果そのものや
`previewId`などを使用して
登録する方式は採用しない。

---

### 8.3 エンドポイント

```http
POST /api/v1/month-end-holding-values/imports
```

---

### 8.4 HTTPメソッド

```http
POST
```

本APIは、
CSVファイルを受け取り、
複数の商品別月末評価額を
新規登録するAPIであるため、
`POST`を使用する。

正常に登録された場合は、
CSV内の複数行に対応する
`month_end_holding_values`が
新規作成される。

本APIは
業務データを変更するため、
CSV-005 商品別月末評価額CSVプレビューとは異なり、
副作用を持つ。

---

### 8.5 認証・利用者の扱い

Phase1では、
認証機能を実装しない。

操作対象となる利用者は、
`X-User-Id`
リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

利用者IDは、
以下では受け付けない。

- パスパラメータ
- クエリパラメータ
- CSV列
- `multipart/form-data`の業務項目

操作対象利用者は、
API共通方針に従い、
`X-User-Id`から特定する。

商品別月末評価額CSVには、

```text
user_id
userId
```

を持たせない。

CSV内では、
資産口座および保有商品を

```text
asset_account_name
holding_asset_name
```

によって指定する。

バックエンドでは、
操作対象利用者との関連を確認したうえで、
内部IDへ変換する。

---

#### 8.5.1 資産口座の利用者境界

CSVの
`asset_account_name`から
資産口座を特定する際は、
必ず操作対象利用者によって
絞り込む。

概念的な検索条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = CSVのasset_account_name

AND

asset_accounts.deleted_at
    IS NULL
```

他の利用者に
同名の資産口座が存在していても、
登録対象としてはならない。

例えば、

```text
User A
証券口座

User B
証券口座
```

が存在する状態で、
User Aを操作対象として
CSVに

```text
asset_account_name = 証券口座
```

が指定されている場合は、
User Aの資産口座のみを
対象とする。

---

#### 8.5.2 保有商品の利用者境界

CSVの
`holding_asset_name`から
登録対象となる保有商品を特定する場合は、
資産口座との関連を含めて
利用者境界を保証する。

概念的な条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = CSVのasset_account_name

AND

holding_assets.asset_account_id
    = asset_accounts.id

AND

holding_assets.name
    = CSVのholding_asset_name

AND

holding_assets.deleted_at
    IS NULL
```

保有商品名だけを条件として
登録対象を特定してはならない。

---

#### 8.5.3 同名保有商品の扱い

異なる資産口座に
同じ名前の保有商品が
存在する可能性がある。

例えば、

```text
証券口座A
    全世界株式

証券口座B
    全世界株式
```

が存在する場合、
CSVに

```text
asset_account_name = 証券口座A
holding_asset_name = 全世界株式
```

と指定されたときは、
証券口座Aに属する
`全世界株式`のみを
登録対象とする。

`holding_asset_name`だけで
保有商品を検索してはならない。

---

#### 8.5.4 他利用者の同名保有商品

操作対象利用者には
該当する保有商品が存在せず、
他の利用者にのみ
同名保有商品が存在する場合でも、
登録可能とは判定しない。

例えば、

```text
User A
証券口座あり
全世界株式なし

User B
証券口座あり
全世界株式あり
```

の場合に、
User Aとして

```text
asset_account_name = 証券口座
holding_asset_name = 全世界株式
```

を指定した場合は、

```text
HOLDING_ASSET_NOT_FOUND
```

相当の
登録エラーとして扱う。

User Bの保有商品へ
商品別月末評価額を
登録してはならない。

---

#### 8.5.5 残高記録単位

CSV-006は、
商品別月末評価額CSVの
登録APIである。

そのため、
対象資産口座の

```text
asset_accounts.balance_recording_unit
```

が
商品単位であることを確認する。

口座単位の資産口座は、
本APIの登録対象としない。

口座単位の資産口座については、
月末資産残高CSV登録APIを使用する。

他利用者の資産口座の
残高記録単位を
判定へ使用してはならない。

---

#### 8.5.6 保有商品と資産口座の関連

CSVで指定された
`holding_asset_name`が、
同じ行で指定された
`asset_account_name`の
資産口座に属していることを
必ず確認する。

例えば、

```text
証券口座A
    全世界株式

証券口座B
    S&P500
```

という状態で、

```text
asset_account_name
    = 証券口座A

holding_asset_name
    = S&P500
```

と指定されても、
証券口座Bの
`S&P500`を
登録対象として使用しない。

指定された資産口座内に
対象保有商品が存在しないものとして扱う。

---

#### 8.5.7 月末資産状況の利用者境界

CSVの`target_year_month`に対応する
月末資産状況を確認する際も、
必ず操作対象利用者によって
絞り込む。

概念的な条件は、
以下とする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = CSVのtarget_year_month
```

他の利用者に
同一対象年月の
月末資産状況が存在していても、
CSV-006の登録処理に
使用してはならない。

---

#### 8.5.8 月末資産状況が存在しない場合

対象年月の
`month_end_asset_snapshots`が
存在しない場合は、
CSV-006で
新しい月末資産状況を作成する。

作成する場合は、
必ず操作対象利用者に
紐づける。

概念的には、
以下とする。

```text
user_id
    = 操作対象利用者ID

target_year_month
    = CSVのtarget_year_month

confirmed
    = false
```

他の利用者に
同じ`target_year_month`の
月末資産状況が存在していても、
そのレコードを
流用してはならない。

月末資産状況の作成は、
CSV内容および
すべての業務ルール検証に
成功した後、
登録トランザクション内で行う。

---

#### 8.5.9 既存商品別月末評価額の利用者境界

既存の
`month_end_holding_values`との
重複確認では、
対象月末資産状況と
対象保有商品の双方について
操作対象利用者との関連を保証する。

概念的には、

```text
month_end_holding_values
    ↓
month_end_asset_snapshots
    ↓
month_end_asset_snapshots.user_id
        = 操作対象利用者ID
```

および、

```text
month_end_holding_values
    ↓
holding_assets
    ↓
asset_accounts
    ↓
asset_accounts.user_id
        = 操作対象利用者ID
```

によって
利用者境界を保証する。

---

#### 8.5.10 他利用者の既存商品別月末評価額

他の利用者に

- 同じ対象年月
- 同名資産口座
- 同名保有商品
- 商品別月末評価額

が存在していても、
CSV-006の重複判定へ
影響させてはならない。

操作対象利用者に属する
既存商品別月末評価額だけを
重複判定へ使用する。

---

#### 8.5.11 登録先保有商品の特定

Phase1では、
CSV入力項目として

```text
asset_account_id
holding_asset_id
```

を使用しない。

登録先保有商品は、

```text
操作対象利用者
+
asset_account_name
+
holding_asset_name
```

によって特定する。

CSV解析後に
サーバー側で取得した

```text
holding_assets.id
```

を使用して、

```text
month_end_holding_values.holding_asset_id
```

を設定する。

---

#### 8.5.12 登録する商品別月末評価額の利用者境界

`month_end_holding_values`自体に
`user_id`を保持しない場合でも、
登録する

```text
month_end_asset_snapshot_id
```

および

```text
holding_asset_id
```

の両方が
操作対象利用者に属することを
登録前に保証する。

概念的には、

```text
操作対象利用者
    ↓
month_end_asset_snapshot

操作対象利用者
    ↓
asset_account
    ↓
holding_asset
```

の双方を満たしたうえで、

```text
month_end_holding_value
```

を登録する。

以下のような
利用者をまたぐ組み合わせを
登録してはならない。

```text
User Aのsnapshot
+
User Bのholding_asset
    ↓
month_end_holding_value
```

---

#### 8.5.13 他利用者データを登録可否判定へ利用しない

他の利用者に属する

- 資産口座
- 保有商品
- 月末資産状況
- 商品別月末評価額

の存在を、
CSV-006の登録可否判定へ
影響させてはならない。

例えば、
操作対象利用者には
`全世界株式`が存在せず、
他利用者にのみ存在する場合は、

```text
HOLDING_ASSET_NOT_FOUND
```

相当の
登録エラーとして扱う。

---

#### 8.5.14 X-User-Idのエラー

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

として扱う。

形式が不正な場合は、

```text
INVALID_USER_ID
```

として扱う。

指定された利用者が
存在しない場合、
または論理削除されている場合は、

```text
USER_NOT_FOUND
```

として扱う。

これらのエラーが発生した場合は、

- CSVファイル解析
- 資産口座検索
- 保有商品検索
- 月末資産状況作成
- 商品別月末評価額登録

へ進まない。

---

#### 8.5.15 CSV-005の利用者コンテキストを引き継がない

CSV-006では、
CSV-005実行時の
利用者コンテキストを
サーバー側で保持・引き継がない。

CSV-006のリクエストでも、
改めて

```http
X-User-Id
```

を指定する。

CSV-005で
User AとしてプレビューしたCSVを、
CSV-006でUser Bとして送信した場合は、
CSV-006では
User Bの業務データを基準として
再検証する。

CSV-005の結果を
利用者境界の保証として
使用しない。

---

#### 8.5.16 利用者境界確認後に登録する

商品別月末評価額を
登録する前に、
少なくとも以下について
操作対象利用者との関連を確認する。

```text
asset_account
    → 操作対象利用者に属する

holding_asset
    → 操作対象利用者の
      指定asset_accountに属する

month_end_asset_snapshot
    → 操作対象利用者に属する

existing month_end_holding_value
    → 操作対象利用者のデータだけを確認
```

これらの検証に
失敗した場合は、
CSV全体を登録しない。

一部の行だけを
登録してはならない。

---

#### 8.5.17 1リクエスト1利用者

1回のCSV-006リクエストでは、
`X-User-Id`で指定された
1利用者のデータだけを扱う。

1つのCSVファイルから
複数利用者の商品別月末評価額を
登録することはできない。

CSVには
利用者を切り替えるための
列を持たせない。

これにより、
商品別月末評価額CSV登録の
利用者境界を明確にする。

---

### 8.6 パスパラメータ

本APIでは、
パスパラメータを使用しない。

エンドポイントは、
以下とする。

```http
POST /api/v1/month-end-holding-values/imports
```

以下の情報を
URLへ含めない。

- `userId`
- `targetYearMonth`
- `assetAccountId`
- `holdingAssetId`
- `snapshotId`

操作対象利用者は、
`X-User-Id`
リクエストヘッダーから特定する。

対象年月、
資産口座名、
保有商品名、
商品別月末評価額は、
アップロードされたCSVファイルから取得する。

---

### 8.7 クエリパラメータ

本APIでは、
クエリパラメータを使用しない。

以下のような
クエリパラメータは受け付けない。

- `targetYearMonth`
- `assetAccountId`
- `holdingAssetId`
- `confirmed`
- `overwrite`
- `force`
- `preview`

本APIは、
商品別月末評価額を
新規一括登録するAPIである。

上書き可否や
強制登録などを
クエリパラメータによって
切り替えない。

---

### 8.8 リクエストヘッダー

本APIでは、
以下のリクエストヘッダーを使用する。

| ヘッダー | 必須 | 内容 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
POST /api/v1/month-end-holding-values/imports
Accept: application/json
X-User-Id: 1
```

CSVファイルは、
`multipart/form-data`で送信する。

`Content-Type`は、
HTTPクライアントによって
boundary付きで設定する。

概念例：

```http
Content-Type: multipart/form-data; boundary=...
```

クライアント側で
boundaryを固定値として
手動設定しない。

---

### 8.9 リクエストボディ

本APIでは、
`multipart/form-data`形式で
CSVファイルを受け取る。

リクエスト項目は、
以下とする。

| 項目 | 型 | 必須 | 内容 |
|---|---|:---:|---|
| `file` | file | ○ | 登録対象となる商品別月末評価額CSV |

概念例：

```text
multipart/form-data

file:
  month-end-holding-values.csv
```

CSVファイル以外の
業務項目は
リクエストボディで受け付けない。

以下のような項目は、
`multipart/form-data`へ
追加しない。

- `userId`
- `targetYearMonth`
- `assetAccountId`
- `holdingAssetId`
- `value`
- `confirmed`
- `overwrite`
- `previewId`
- `canImport`

操作対象利用者は、
`X-User-Id`から取得する。

その他の業務項目は、
CSV内容から取得する。

---

### 8.10 CSVファイル仕様

商品別月末評価額CSVでは、
以下のヘッダーを使用する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

ヘッダー順序も
CSV仕様の一部とする。

CSV-004
商品別月末評価額CSVテンプレート取得で
提供する形式と同一とする。

CSV入力例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

1つのCSVファイルでは、
1つの`target_year_month`のみを扱う。

---

### 8.11 バリデーション

CSV-006では、
CSV-005と同等の
CSV解析・登録可否検証を
再度実行する。

検証は、
概念的に以下の段階で行う。

```text
リクエスト検証
    ↓
CSVファイル・構造検証
    ↓
CSV行入力値検証
    ↓
CSV全体ルール検証
    ↓
業務データ照合
    ↓
業務ルール検証
    ↓
登録可否確定
    ↓
登録処理
```

CSV-005で

```text
canImport = true
```

となっていることを、
CSV-006の
入力条件とはしない。

CSV-006単体でも、
安全に登録可否を
判定できる必要がある。

---

#### 8.11.1 file 必須

`file`は、
必須とする。

CSVファイルが
指定されていない場合は、
バリデーションエラーとする。

この場合、
CSV解析処理や
データベース登録へ進まない。

---

#### 8.11.2 アップロードファイルであること

`file`は、
HTTPアップロードファイルとして
正常に受信できていることを確認する。

通常の文字列や
フォーム値を
CSVファイルとして扱わない。

---

#### 8.11.3 ファイル拡張子

アップロードファイルは、
CSVファイルであることを確認する。

Phase1では、
拡張子として

```text
.csv
```

を受け付ける。

例えば、
以下は不正とする。

```text
.xlsx
.xls
.pdf
```

ただし、
拡張子だけを
唯一の判定根拠とはせず、
実際にCSVとして
正常に読み取れることも確認する。

---

#### 8.11.4 ファイルサイズ

CSVファイルサイズには、
CSV共通仕様で定める
上限を適用する。

上限を超える場合は、
CSV解析前に
バリデーションエラーとする。

具体的な上限値は、
CSV共通仕様の
最新定義に従う。

---

#### 8.11.5 空ファイル

CSVファイルが
0バイトの場合、
または有効なヘッダー行を
取得できない場合は、
不正なCSVとして扱う。

この場合、
登録処理へ進まない。

---

#### 8.11.6 文字コード

CSVファイルの文字コードは、
CSV共通仕様に従う。

Phase1では、
UTF-8を基本とする。

CSV-004で提供する
テンプレートと
同じ文字コードを
使用することを前提とする。

UTF-8 BOMを許可する場合は、
先頭ヘッダーへ
BOMが混入しないよう
解析時に除去する。

---

#### 8.11.7 CSVとして読み取り可能であること

アップロードされたファイルが、
CSVとして正常に
読み取れることを確認する。

例えば、
以下を不正とする。

- CSV構造が破損している
- 引用符が不正
- ヘッダーを取得できない
- 列構造を正常に解釈できない

CSV解析に失敗した場合は、
データベース登録へ進まない。

---

#### 8.11.8 ヘッダー必須

CSVの1行目には、
ヘッダー行が
存在することを必須とする。

期待するヘッダーは、
以下とする。

```text
target_year_month
asset_account_name
holding_asset_name
value
```

---

#### 8.11.9 ヘッダー名

ヘッダーは、
以下と完全一致することを
基本とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

例えば、
以下のような別名は
受け付けない。

```text
targetYearMonth
assetAccountName
holdingAssetName
amount
```

CSV-004のテンプレートを
正式な入力形式とする。

---

#### 8.11.10 ヘッダー順序

ヘッダーは、
以下の順序とする。

```text
1. target_year_month
2. asset_account_name
3. holding_asset_name
4. value
```

Phase1では、
同じヘッダー名でも
順序が異なるCSVは
不正とする。

CSV-004、
CSV-005、
CSV-006で
同一仕様を使用する。

---

#### 8.11.11 余分なヘッダー

定義されていない
余分なCSV列は
受け付けない。

例えば、

```csv
target_year_month,asset_account_name,holding_asset_name,value,memo
```

は不正とする。

以下のような
内部管理項目も
CSV列として受け付けない。

- `user_id`
- `asset_account_id`
- `holding_asset_id`
- `month_end_asset_snapshot_id`
- `month_end_holding_value_id`
- `confirmed`

---

#### 8.11.12 データ行の存在

ヘッダー行だけで、
データ行が1件も存在しない場合は、
登録対象データが存在しないため、
登録不可とする。

例えば、

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

のみのCSVは
登録しない。

CSV-006では、

```text
importedCount = 0
```

として正常終了させない。

---

#### 8.11.13 空行

CSV末尾などの
完全な空行については、
CSV共通仕様に従って
無視してよい。

ただし、

```csv
,,,
```

のように
列として存在する行は、
データ行として扱い、
各項目の必須チェックを行う。

---

#### 8.11.14 target_year_month 必須

各データ行の
`target_year_month`は
必須とする。

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,全世界株式,1500000
```

未入力の場合は、
登録不可とする。

---

#### 8.11.15 target_year_month 形式

`target_year_month`は、
以下の形式とする。

```text
YYYY-MM
```

正常例：

```text
2026-01
2026-07
2026-12
```

不正例：

```text
2026-1
26-07
2026/07
202607
2026-00
2026-13
abc
```

形式だけでなく、
年月として
有効な値であることも確認する。

---

#### 8.11.16 1ファイル1対象年月

1つのCSVファイルでは、
すべてのデータ行の
`target_year_month`が
同一であることを必須とする。

正常例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-06,証券口座,全世界株式,1400000
2026-07,証券口座,S&P500,800000
```

複数対象年月が
混在する場合は、
CSV全体を登録しない。

---

#### 8.11.17 asset_account_name 必須

各データ行の
`asset_account_name`は
必須とする。

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,,全世界株式,1500000
```

未入力の場合は、
登録不可とする。

---

#### 8.11.18 asset_account_nameの扱い

`asset_account_name`は、
資産口座を特定するための
文字列として扱う。

前後空白の扱いは、
CSV共通仕様または
資産口座名の業務ルールに従う。

入力値を
暗黙的に別名へ変換して
資産口座を特定しない。

---

#### 8.11.19 資産口座存在確認

CSVの
`asset_account_name`に対応する
資産口座が、
操作対象利用者に
存在することを確認する。

概念的な条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = asset_account_name

AND

asset_accounts.deleted_at
    IS NULL
```

該当する資産口座が
存在しない場合は、
CSV全体を登録不可とする。

---

#### 8.11.20 他利用者の同名資産口座

操作対象利用者には
該当資産口座が存在せず、
他利用者にのみ
同名資産口座が存在する場合でも、
正常とは判定しない。

概念的には、

```text
ASSET_ACCOUNT_NOT_FOUND
```

相当の
登録エラーとする。

---

#### 8.11.21 残高記録単位

CSV-006は、
商品別月末評価額CSVの
登録APIである。

そのため、
対象資産口座の

```text
asset_accounts.balance_recording_unit
```

が
商品単位であることを確認する。

口座単位の資産口座は、
本APIの登録対象としない。

---

#### 8.11.22 holding_asset_name 必須

各データ行の
`holding_asset_name`は
必須とする。

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,,1500000
```

未入力の場合は、
登録不可とする。

---

#### 8.11.23 holding_asset_nameの扱い

`holding_asset_name`は、
同一行で指定された
資産口座内の
保有商品を特定するために使用する。

保有商品名だけを
システム全体から検索して
対象商品を決定しない。

---

#### 8.11.24 保有商品存在確認

CSVで指定された

```text
asset_account_name
+
holding_asset_name
```

に対応する
保有商品が存在することを確認する。

概念的な条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = asset_account_name

AND

holding_assets.asset_account_id
    = asset_accounts.id

AND

holding_assets.name
    = holding_asset_name

AND

holding_assets.deleted_at
    IS NULL
```

存在しない場合は、
CSV全体を登録不可とする。

---

#### 8.11.25 他資産口座にのみ同名保有商品が存在する

例えば、

```text
証券口座A
    全世界株式なし

証券口座B
    全世界株式あり
```

の状態で、

```text
asset_account_name
    = 証券口座A

holding_asset_name
    = 全世界株式
```

と指定した場合は、
証券口座Bの商品を
使用しない。

```text
HOLDING_ASSET_NOT_FOUND
```

相当の
登録エラーとする。

---

#### 8.11.26 他利用者にのみ同名保有商品が存在する

操作対象利用者には
該当保有商品が存在せず、
他利用者にのみ
同名保有商品が存在する場合も、

```text
HOLDING_ASSET_NOT_FOUND
```

相当の
登録エラーとする。

他利用者の商品へ
商品別月末評価額を
登録してはならない。

---

#### 8.11.27 対象年月時点での保有商品の有効性

対象保有商品が、
CSVの`target_year_month`時点で
商品別月末評価額の
記録対象として有効であることを確認する。

現在時点の状態だけで
判定してはならない。

必ず、

```text
CSVのtarget_year_month
```

を基準とする。

対象年月時点で
無効な保有商品は、
登録不可とする。

---

#### 8.11.28 value 必須

各データ行の
`value`は
必須とする。

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,
```

未入力の場合は、
登録不可とする。

---

#### 8.11.29 value 数値形式

`value`は、
日本円の整数として
扱える値であることを確認する。

正常例：

```text
0
1000
1500000
```

不正例：

```text
1,500,000
¥1500000
1500.5
abc
```

CSV上では、
桁区切りや
通貨記号を使用しない。

---

#### 8.11.30 value 整数

`value`は、
整数であることを必須とする。

以下のような
小数値は受け付けない。

```text
1000.5
```

Phase1では、
日本円の整数として管理する。

---

#### 8.11.31 value 0円以上

`value`は、
0以上とする。

正常例：

```text
0
1
1500000
```

不正例：

```text
-1
-100000
```

0円は
有効な商品別月末評価額として扱う。

```text
0円
≠
未入力
```

とする。

---

#### 8.11.32 CSV内重複

同一CSV内で、
同じ対象年月、
同じ資産口座、
同じ保有商品が
複数回指定されていないことを確認する。

概念的な重複キーは、
以下とする。

```text
targetYearMonth
+
assetAccountName
+
holdingAssetName
```

1ファイル1対象年月が
成立している場合は、
実質的には

```text
assetAccountName
+
holdingAssetName
```

で
重複確認してよい。

---

#### 8.11.33 CSV内重複時の扱い

例えば、

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,全世界株式,1600000
```

は登録不可とする。

以下のような
自動解決は行わない。

- 先勝ち
- 後勝ち
- 評価額合算
- 平均値算出

---

#### 8.11.34 月末資産状況の確認

CSVの`target_year_month`について、
操作対象利用者に属する
`month_end_asset_snapshots`を確認する。

概念的な条件は、
以下とする。

```text
user_id
    = 操作対象利用者ID

AND

target_year_month
    = CSVのtarget_year_month
```

他利用者の
同一対象年月snapshotを
使用しない。

---

#### 8.11.35 月末資産状況が存在しない場合

対象年月の
`month_end_asset_snapshots`が
存在しない場合は、
CSV-006で
新しい月末資産状況を作成する。

ただし、
CSV解析・入力値検証・
業務ルール検証が
すべて成功した後に行う。

概念的には、
以下を登録する。

```text
user_id
    = 操作対象利用者ID

target_year_month
    = CSVのtarget_year_month

confirmed
    = false
```

月末資産状況の作成は、
商品別月末評価額の登録と
同一トランザクション内で行う。

---

#### 8.11.36 確定済み月末資産状況

対象年月の
月末資産状況が存在し、

```text
confirmed = true
```

の場合は、
商品別月末評価額を
登録できない。

この場合、
CSV全体を登録不可とする。

CSV-005で
`canImport = true`
となっていた場合でも、
CSV-006実行時点で
確定済みであれば
登録してはならない。

---

#### 8.11.37 既存商品別月末評価額

対象年月、
対象保有商品について、
既に
`month_end_holding_values`が
存在するか確認する。

概念的には、

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

の組み合わせで
既存データを確認する。

Phase1では、
CSV-006を
新規一括登録APIとして扱う。

そのため、
既存データが存在する場合は
重複エラーとする。

既存の`value`を
CSV値で上書きしない。

---

#### 8.11.38 overwriteを許可しない

CSV-006では、
既存の商品別月末評価額を
上書きするための

```text
overwrite = true
```

などのオプションを
受け付けない。

既存データを変更する場合は、
商品別月末評価額更新APIを使用する。

CSV登録APIへ
更新責務を持たせない。

---

#### 8.11.39 他利用者の既存商品別月末評価額

他利用者に

- 同じ対象年月
- 同名資産口座
- 同名保有商品
- 商品別月末評価額

が存在していても、
操作対象利用者の
重複判定へ影響させない。

操作対象利用者に属する
既存データだけを
重複判定へ使用する。

---

#### 8.11.40 複数エラー

CSV内に
複数の入力・業務エラーが
存在する場合は、
CSV-005と同様に
可能な範囲で
複数エラーを検出してよい。

ただし、
CSV-006では
エラーが1件でも存在する場合、
登録処理へ進まない。

概念的には、

```text
CSV検証
    ↓
エラー収集
    ↓
1件以上あり
    ↓
登録処理中止
```

とする。

---

#### 8.11.41 一部登録を行わない

CSV内に
1件でも登録不可データが
存在する場合は、
CSV全体を登録しない。

例えば、

```text
2行目 正常
3行目 正常
4行目 エラー
```

の場合でも、

```text
2行目 登録
3行目 登録
4行目 未登録
```

とはしない。

CSV全体を
登録失敗として扱う。

---

#### 8.11.42 登録前に全件検証する

CSV行を
読み込みながら
逐次データベースへ
INSERTする方式は採用しない。

以下のような処理は
行わない。

```text
2行目
検証
    ↓
INSERT

3行目
検証
    ↓
INSERT

4行目
エラー
```

原則として、

```text
CSV全体解析
    ↓
全件検証
    ↓
登録可能確認
    ↓
トランザクション開始
    ↓
一括登録
```

とする。

---

#### 8.11.43 CSV-005と同じ検証ロジックを使用する

CSV-006では、
CSV-005と
可能な限り同一の

- CSV Definition
- CSV Parser
- CSV入力値Validator
- 業務ルールValidator
- Query

を使用する。

同一のシステム状態、
同一のCSVであれば、

```text
CSV-005
canImport = true
```

の場合に、

```text
CSV-006
登録可能
```

となることを基本とする。

CSV-005とCSV-006で
同じ検証ルールを
別々に実装しない。

---

#### 8.11.44 CSV-005の結果は信頼しない

CSV-006では、
CSV-005で返却された

```text
canImport
errors
rows
targetYearMonth
```

などを
リクエストとして受け取らない。

また、
クライアント側で保持している
プレビュー結果を
登録可否の根拠として使用しない。

CSV-006は、
受信したCSVファイルと
実行時点のデータベース状態だけを
基準として登録可否を判定する。

---

#### 8.11.45 登録直前の再確認

CSV全体の事前検証に
成功した場合でも、
登録トランザクション内で
競合し得る状態については
必要に応じて再確認する。

特に、

- `month_end_asset_snapshots.confirmed`
- 既存`month_end_holding_values`

については、
CSV-005実行時点ではなく
CSV-006の登録時点で
正しい状態を保証する必要がある。

具体的なロック・
再確認方法は、
「Laravel実装方針」で定義する。

---

#### 8.11.46 データベース制約も最終防衛線とする

アプリケーション側で
既存商品別月末評価額の
重複を事前検証する。

加えて、
テーブル定義で

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

に一意性を保証している場合は、
データベース制約も
最終的な重複防止として利用する。

ただし、
DB制約違反を
通常の業務フローとして
発生させることを前提にはしない。

可能な限り、
登録前に業務ルールとして
検出する。

---

#### 8.11.47 バリデーション失敗時はDB更新しない

以下のいずれかの
検証に失敗した場合は、
業務データを更新しない。

- ファイル検証
- CSV構造検証
- CSV入力値検証
- 資産口座検証
- 残高記録単位検証
- 保有商品検証
- 資産口座と保有商品の関連検証
- 対象年月時点の有効性検証
- 確定状態検証
- 既存商品別月末評価額検証
- CSV内重複検証

以下を行ってはならない。

```text
month_end_asset_snapshots INSERT
month_end_holding_values INSERT
```

すべての検証に成功した場合のみ、
登録処理へ進む。

---

### 8.12 業務ルール

CSV-006では、
商品別月末評価額CSVの内容を再検証し、
すべての登録条件を満たす場合のみ、
商品別月末評価額を一括登録する。

本APIは、
CSV-005 商品別月末評価額CSVプレビューの
結果を保存・参照しない。

CSV-006実行時点の

- CSVファイル
- 操作対象利用者
- 資産口座
- 保有商品
- 月末資産状況
- 既存商品別月末評価額

をもとに、
登録可否を再判定する。

---

#### 8.12.1 CSV-004・CSV-005と同じCSV仕様を使用する

CSV-006で受け付ける
CSV形式は、
CSV-004 商品別月末評価額CSVテンプレート取得、
CSV-005 商品別月末評価額CSVプレビューと
同一とする。

ヘッダーは、
以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

以下についても、
同一のCSV共通仕様を使用する。

- 文字コード
- BOM
- 改行コード
- ヘッダー名
- ヘッダー順序
- 必須項目
- 対象年月形式
- 商品別月末評価額形式
- CSV内重複判定

CSV-005では正常、
CSV-006では
CSV仕様不正となるような
実装差異を発生させない。

---

#### 8.12.2 CSV-005と同じ登録可否ルールを使用する

CSV-005とCSV-006では、
可能な限り
同じCSV解析・業務検証ロジックを使用する。

同一DB状態、
同一利用者、
同一CSVであれば、

```text
CSV-005
canImport = true
```

となる場合は、

```text
CSV-006
登録可能
```

となることを基本とする。

ただし、
CSV-005からCSV-006までの間に
業務状態が変更された場合は、
CSV-006で登録不可となることを許容する。

---

#### 8.12.3 1ファイル1対象年月

1つのCSVファイルでは、
1つの対象年月のみを扱う。

すべてのデータ行について、

```text
target_year_month
```

が同一であることを必須とする。

正常例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-06,証券口座,全世界株式,1400000
2026-07,証券口座,S&P500,800000
```

複数対象年月が
混在する場合は、
CSV全体を登録しない。

---

#### 8.12.4 資産口座の特定

CSV上では、
資産口座IDを指定しない。

対象資産口座は、

```text
操作対象利用者
+
asset_account_name
```

によって特定する。

概念的な条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = CSVのasset_account_name

AND

asset_accounts.deleted_at
    IS NULL
```

CSVに

```text
asset_account_id
```

を持たせない。

---

#### 8.12.5 保有商品の特定

CSV上では、
保有商品IDを指定しない。

対象保有商品は、

```text
操作対象利用者
+
asset_account_name
+
holding_asset_name
```

によって特定する。

概念的な条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = CSVのasset_account_name

AND

holding_assets.asset_account_id
    = asset_accounts.id

AND

holding_assets.name
    = CSVのholding_asset_name

AND

holding_assets.deleted_at
    IS NULL
```

保有商品名だけを条件として
対象商品を特定しない。

---

#### 8.12.6 利用者境界

以下の業務データについて、
必ず操作対象利用者との
関連を保証する。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

他利用者の

- 同名資産口座
- 同名保有商品
- 同一対象年月の月末資産状況
- 商品別月末評価額

を
登録処理へ使用してはならない。

---

#### 8.12.7 残高記録単位

商品別月末評価額CSVでは、
商品単位で管理する
資産口座のみを対象とする。

対象資産口座の

```text
asset_accounts.balance_recording_unit
```

が
商品単位であることを必須とする。

口座単位の資産口座は、
CSV-006の登録対象としない。

口座単位の月末資産残高は、
CSV-003 月末資産残高CSV登録で扱う。

---

#### 8.12.8 保有商品は指定資産口座に属すること

CSVで指定された
`holding_asset_name`は、
同じ行で指定された
`asset_account_name`の
資産口座に属している必要がある。

例えば、

```text
証券口座A
    全世界株式

証券口座B
    S&P500
```

の状態で、

```text
asset_account_name
    = 証券口座A

holding_asset_name
    = S&P500
```

と指定された場合に、
証券口座Bの
`S&P500`を使用してはならない。

指定された資産口座内に
保有商品が存在しないものとして扱う。

---

#### 8.12.9 対象年月時点の保有商品

対象保有商品が、
CSVの`target_year_month`時点で
商品別月末評価額の
記録対象として有効であることを確認する。

現在日時ではなく、

```text
CSVのtarget_year_month
```

を基準として判定する。

対象年月時点で
有効でない保有商品については、
CSV全体を登録不可とする。

---

#### 8.12.10 value

`value`は、
対象年月末時点の
保有商品の評価額として扱う。

Phase1では、
日本円の整数とする。

以下を満たす必要がある。

```text
整数
AND
0以上
```

正常例：

```text
0
1000
1500000
```

不正例：

```text
-1
1000.5
abc
¥1000
1,000
```

---

#### 8.12.11 0円を有効値とする

```text
value = 0
```

は、
正常な商品別月末評価額として扱う。

0円を

- 未入力
- NULL
- 登録対象外

として扱わない。

```text
0円
≠
未入力
```

とする。

---

#### 8.12.12 CSV内重複

同一CSV内では、
以下の組み合わせを
重複させない。

```text
target_year_month
+
asset_account_name
+
holding_asset_name
```

1ファイル1対象年月であるため、
実質的には、

```text
asset_account_name
+
holding_asset_name
```

で一意であることを
必須とする。

---

#### 8.12.13 CSV内重複を自動解決しない

同一保有商品が
CSV内に複数行存在する場合は、
登録不可とする。

例えば、

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,全世界株式,1600000
```

について、

- 先勝ち
- 後勝ち
- 評価額合算
- 平均値算出

などの
自動解決を行わない。

---

#### 8.12.14 月末資産状況の取得

CSVの対象年月について、
操作対象利用者に属する
月末資産状況を取得する。

概念的な条件は、
以下とする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = CSVのtarget_year_month
```

他利用者の
同一対象年月snapshotを
登録先として使用しない。

---

#### 8.12.15 月末資産状況が存在しない場合

対象年月の
月末資産状況が
存在しない場合は、
CSV登録処理の中で
新規作成する。

概念的には、
以下を設定する。

```text
user_id
    = 操作対象利用者ID

target_year_month
    = CSVのtarget_year_month

confirmed
    = false
```

月末資産状況の作成は、
CSV全体の検証が
正常に完了した後、
登録トランザクション内で行う。

CSV検証途中で
先にsnapshotを
作成してはならない。

---

#### 8.12.16 既存の未確定月末資産状況

対象年月について
未確定の月末資産状況が
既に存在する場合は、
新しいsnapshotを作成しない。

既存snapshotを
商品別月末評価額の
登録先として使用する。

```text
snapshotあり
+
confirmed = false
    ↓
既存snapshot使用
```

とする。

---

#### 8.12.17 確定済み月末資産状況

対象年月の
月末資産状況が

```text
confirmed = true
```

の場合は、
CSV登録を許可しない。

CSV全体を登録しない。

CSV-005実行時点で
未確定であっても、
CSV-006実行時点で
確定済みの場合は、
CSV-006実行時点の
最新状態を優先する。

---

#### 8.12.18 既存商品別月末評価額

対象年月、
対象保有商品について、
既に商品別月末評価額が
登録されていないことを確認する。

概念的には、

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

の組み合わせで
重複を確認する。

既存データが存在する場合は、
CSV登録を許可しない。

---

#### 8.12.19 既存データを上書きしない

CSV-006では、
既存の商品別月末評価額を
CSV値によって
上書きしない。

以下のような処理は行わない。

```text
既存valueあり
    ↓
CSVのvalueでUPDATE
```

CSV-006は、
新規一括登録に
責務を限定する。

既存の商品別月末評価額を
変更する場合は、
専用更新APIを使用する。

---

#### 8.12.20 既存値と同じ場合も登録しない

既に、

```text
value = 1500000
```

が登録されている状態で、
CSVにも

```text
value = 1500000
```

が指定されていても、
登録済みとして扱う。

同じ値であることを理由に、
成功扱いにはしない。

---

#### 8.12.21 全件成功または全件失敗

CSV-006は、
CSVファイル単位で
原子的に登録する。

1件でも
登録不可となるデータが
存在する場合は、
正常な行を含めて
すべて登録しない。

```text
2行目
正常

3行目
正常

4行目
エラー
    ↓
全件登録しない
```

部分成功は許可しない。

---

#### 8.12.22 全件検証後に登録する

CSV行を解析しながら
逐次INSERTしてはならない。

以下のような処理は
採用しない。

```text
2行目
検証成功
    ↓
INSERT

3行目
検証成功
    ↓
INSERT

4行目
検証失敗
```

基本的には、
以下の順序とする。

```text
CSV全体解析
    ↓
全行入力値検証
    ↓
全行业務ルール検証
    ↓
登録可能確認
    ↓
トランザクション開始
    ↓
一括登録
```

---

#### 8.12.23 登録トランザクション

月末資産状況の必要時作成と
商品別月末評価額の登録は、
同一トランザクション内で行う。

概念的には、

```text
BEGIN
    ↓
snapshot取得・必要なら作成
    ↓
確定状態再確認
    ↓
既存商品別月末評価額再確認
    ↓
商品別月末評価額一括登録
    ↓
COMMIT
```

とする。

途中で
1件でも登録に失敗した場合は、

```text
ROLLBACK
```

する。

---

#### 8.12.24 snapshotだけを残さない

対象年月のsnapshotが
存在しなかったため
CSV-006で新規作成した後に、
商品別月末評価額登録で
エラーが発生した場合は、
新規作成したsnapshotも
ロールバックする。

```text
snapshotなし
    ↓
snapshot作成
    ↓
商品別月末評価額登録中にエラー
    ↓
ROLLBACK
    ↓
snapshotも残さない
```

とする。

---

#### 8.12.25 商品別月末評価額の登録

CSV各行について、
概念的に以下を登録する。

```text
month_end_asset_snapshot_id
    = 対象年月のsnapshot.id

holding_asset_id
    = asset_account_name
      +
      holding_asset_name
      から特定したholding_assets.id

value
    = CSVのvalue
```

CSV上の

```text
asset_account_name
holding_asset_name
```

自体を
`month_end_holding_values`へ
重複保存しない。

---

#### 8.12.26 CSV行番号は保存しない

CSV解析時に使用した

```text
rowNumber
```

は、
CSV上のエラー箇所を
特定するための一時情報である。

`month_end_holding_values`へ
CSV行番号を保存しない。

---

#### 8.12.27 CSVファイルを保存しない

Phase1では、
アップロードされたCSVファイル自体を
サーバーへ永続保存しない。

以下のような
CSV管理データを作成しない。

```text
csv_import_files
csv_import_histories
csv_import_sessions
```

登録結果は、
通常の

```text
month_end_asset_snapshots
month_end_holding_values
```

として保存する。

---

#### 8.12.28 プレビュー結果を保存・参照しない

CSV-005で生成した
プレビュー結果は、
CSV-006で参照しない。

以下のような
情報を使用しない。

```text
previewId
previewToken
previewResult
canImport
```

CSV-006では、
受信したCSVを
再解析・再検証する。

---

#### 8.12.29 canImportを登録権利としない

CSV-005の

```text
canImport = true
```

は、
その時点での
登録可能性を示すだけである。

CSV-006では、
以下を再確認する。

- CSV内容
- 資産口座
- 残高記録単位
- 保有商品
- 資産口座と保有商品の関連
- 対象年月時点の有効性
- 月末資産状況
- 確定状態
- 既存商品別月末評価額
- CSV内重複

CSV-005の結果を
登録予約として扱わない。

---

#### 8.12.30 同時実行

同一利用者、
同一対象年月、
同一保有商品について
複数のCSV-006が
同時実行される可能性を考慮する。

例えば、

```text
Request A
既存評価額なし確認

Request B
既存評価額なし確認

Request A
INSERT

Request B
INSERT
```

のような競合によって
重複登録が発生しないようにする。

アプリケーション側の確認に加え、
データベースのUNIQUE制約を
最終防衛線として使用する。

---

#### 8.12.31 データベース制約

`month_end_holding_values`では、
同一月末資産状況、
同一保有商品について
複数の評価額を登録できないようにする。

概念的には、

```text
UNIQUE (
    month_end_asset_snapshot_id,
    holding_asset_id
)
```

とする。

CSV-006では、
この制約を
アプリケーション側の
重複チェックの代替にはしない。

---

#### 8.12.32 snapshotの重複作成を防止する

同一利用者、
同一対象年月について
複数のsnapshotを
作成してはならない。

概念的には、

```text
UNIQUE (
    user_id,
    target_year_month
)
```

によって
データベース側でも
一意性を保証する。

---

#### 8.12.33 登録後も未確定とする

CSV-006によって
商品別月末評価額を登録しても、
月末資産状況を
自動確定しない。

```text
CSV-006
商品別月末評価額登録
    ↓
confirmed = false
```

のままとする。

月末資産状況の確定は、
月末資産状況確定APIの
責務とする。

---

#### 8.12.34 未登録商品が存在しても登録可能とする

CSV-006は、
CSVに含まれる
商品別月末評価額を
登録するAPIである。

対象年月時点で有効な
すべての保有商品が
CSVに含まれていることを、
CSV-006の登録条件とはしない。

例えば、
商品単位で管理する
保有商品が3件存在し、
CSVに2件のみ含まれている場合でも、
その2件が正常であれば
登録可能とする。

必要なデータが
すべて揃っているかどうかは、
月末資産状況確定時に判定する。

---

#### 8.12.35 月末資産残高を登録しない

CSV-006では、

```text
month_end_asset_balances
```

を登録しない。

口座単位の月末資産残高は、
CSV-003の責務とする。

CSV-006は、

```text
month_end_holding_values
```

の新規一括登録に
責務を限定する。

---

#### 8.12.36 登録件数

登録件数は、
実際に新規登録した

```text
month_end_holding_values
```

の件数とする。

例えば、
CSVに3件の正常なデータがあり、
3件すべてを登録した場合は、

```text
importedCount = 3
```

とする。

以下は、
登録件数へ含めない。

- ヘッダー行
- 完全な空行
- snapshotの作成件数

---

### 8.13 処理フロー

CSV-006の
正常系処理フローは、
以下とする。

```text
POST
/api/v1/month-end-holding-values/imports
    ↓
X-User-Id検証
    ↓
file検証
    ↓
CSV解析
    ↓
CSVヘッダー検証
    ↓
CSV行入力値検証
    ↓
対象年月特定
    ↓
CSV内重複確認
    ↓
資産口座一括取得
    ↓
保有商品一括取得
    ↓
対象年月時点の有効性確認
    ↓
snapshot確認
    ↓
既存商品別月末評価額確認
    ↓
全件登録可能確認
    ↓
トランザクション開始
    ↓
snapshot再確認
    ↓
必要時snapshot作成
    ↓
確定状態再確認
    ↓
既存商品別月末評価額再確認
    ↓
商品別月末評価額一括登録
    ↓
COMMIT
    ↓
登録結果返却
```

---

#### 8.13.1 利用者コンテキスト確認

最初に、
`X-User-Id`から
操作対象利用者を特定する。

以下の場合は、
CSV解析へ進まない。

```text
X-User-Id未指定
X-User-Id形式不正
利用者不存在
論理削除済み利用者
```

---

#### 8.13.2 CSV解析

CSVファイルを解析し、

- ヘッダー
- データ行
- CSV行番号

を取得する。

CSVとして
正常に解析できない場合は、
登録処理へ進まない。

---

#### 8.13.3 CSV入力値検証

各行について、
以下を検証する。

```text
target_year_month
asset_account_name
holding_asset_name
value
```

入力エラーが
1件でも存在する場合は、
登録処理へ進まない。

---

#### 8.13.4 対象年月特定

CSV全体から、
単一の対象年月を特定する。

複数の対象年月が
存在する場合は、
CSV全体を登録しない。

---

#### 8.13.5 資産口座・保有商品の特定

CSV内で使用されている
資産口座名・保有商品名から、
操作対象利用者に属する

```text
asset_accounts
holding_assets
```

を一括取得する。

CSV行ごとに
個別検索しない。

---

#### 8.13.6 業務ルール検証

以下を確認する。

- 資産口座存在
- 残高記録単位
- 保有商品存在
- 資産口座と保有商品の関連
- 対象年月時点の有効性
- CSV内重複
- 月末資産状況の確定状態
- 既存商品別月末評価額

1件でも
登録不可状態が存在する場合は、
トランザクションへ進まない。

---

#### 8.13.7 トランザクション開始

全件登録可能であることを
事前確認した後に、
登録トランザクションを開始する。

CSV解析処理中から
トランザクションを保持しない。

---

#### 8.13.8 snapshotの再取得

トランザクション内では、
対象年月のsnapshotを
最新状態で再取得する。

必要に応じて
排他ロックを行う。

CSV検証段階で取得した
snapshotの状態だけを
登録可否の最終判断として
使用しない。

---

#### 8.13.9 snapshotの必要時作成

対象年月のsnapshotが
存在しない場合は、
トランザクション内で
新規作成する。

```text
user_id
    = 操作対象利用者ID

target_year_month
    = CSV対象年月

confirmed
    = false
```

とする。

---

#### 8.13.10 確定状態の最終確認

既存または
新規作成したsnapshotについて、
登録直前に
確定状態を確認する。

既存snapshotが

```text
confirmed = true
```

の場合は、
商品別月末評価額を登録しない。

---

#### 8.13.11 既存商品別月末評価額の最終確認

登録トランザクション内でも、
対象保有商品について
既存の商品別月末評価額が
存在しないことを再確認する。

事前検証後に
別リクエストが
先に登録している可能性を考慮する。

---

#### 8.13.12 一括登録

すべての最終確認に
成功した場合は、
CSV各行について

```text
month_end_holding_values
```

を一括登録する。

概念的には、

```text
month_end_asset_snapshot_id
holding_asset_id
value
```

を設定する。

---

#### 8.13.13 COMMIT

すべての商品別月末評価額を
正常に登録できた場合は、
トランザクションを
COMMITする。

その後、
登録結果を返却する。

---

#### 8.13.14 ROLLBACK

トランザクション内で
1件でも登録に失敗した場合は、
すべてROLLBACKする。

以下の状態を
発生させない。

- snapshotだけ新規作成済み
- CSV前半の商品だけ登録済み
- 一部保有商品の評価額だけ登録済み

---

### 8.14 正常レスポンス

CSV内の
すべての商品別月末評価額を
正常に登録できた場合は、

```http
201 Created
```

を返却する。

正常時は、
API共通の
成功レスポンスEnvelopeを使用する。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "importedCount": 2
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

CSV内の
登録済み商品別月末評価額を
全件返却するのではなく、
登録結果の確認に必要な
最小限の情報を返却する。

---

#### 8.14.1 201 Created

CSV内の
すべてのデータについて
検証および登録が成功した場合は、

```http
201 Created
```

を返却する。

複数の

```text
month_end_holding_values
```

を新規作成するため、
`201 Created`とする。

---

#### 8.14.2 snapshotを新規作成した場合

対象年月の
月末資産状況が存在せず、
CSV-006内で
snapshotを新規作成した場合も、
レスポンス形式は変更しない。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "importedCount": 2
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

以下のような
フラグは返却しない。

```text
snapshotCreated
```

---

#### 8.14.3 既存の未確定snapshotを使用した場合

対象年月の
未確定snapshotが
既に存在する場合は、
そのsnapshotへ
商品別月末評価額を登録する。

この場合も、
レスポンス形式は
同一とする。

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "importedCount": 2
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 8.14.4 登録済み商品一覧を返却しない

CSV-006では、
登録した商品別月末評価額を
全件レスポンスへ含めない。

以下のような
レスポンスにはしない。

```json
{
  "data": {
    "values": [
      {
        "assetAccountName": "証券口座",
        "holdingAssetName": "全世界株式",
        "value": 1500000
      },
      {
        "assetAccountName": "証券口座",
        "holdingAssetName": "S&P500",
        "value": 800000
      }
    ]
  }
}
```

登録内容は、
CSV-005で
事前確認できる。

登録後に
最新の商品別月末評価額が必要な場合は、
参照APIから再取得する。

---

#### 8.14.5 エラー時

CSV内容または
業務ルールの検証に失敗した場合は、
商品別月末評価額を登録しない。

CSV-005とは異なり、
CSV-006では
登録できないCSVを

```text
200 OK
canImport = false
```

として返却しない。

CSV-006は登録APIであるため、
登録条件を満たさない場合は
適切なHTTPエラーとして扱う。

具体的なHTTPステータスと
独自エラーコードは、
「エラーレスポンス」で定義する。

---

#### 8.14.6 登録途中のエラー

トランザクション開始後に
登録処理でエラーが発生した場合は、
すべてロールバックする。

以下のような
部分成功レスポンスは返却しない。

```json
{
  "data": {
    "importedCount": 2,
    "failedCount": 1
  }
}
```

本APIは、

```text
全件成功
または
全件失敗
```

とする。

---

### 8.15 レスポンス項目

正常時の
`data`配下には、
以下の項目を返却する。

| 項目 | 型 | NULL | 内容 |
|---|---|:---:|---|
| `targetYearMonth` | string | × | 登録対象となった対象年月 |
| `importedCount` | integer | × | 新規登録した商品別月末評価額件数 |

---

#### 8.15.1 targetYearMonth

登録対象となった
CSVの対象年月を返却する。

形式は、

```text
YYYY-MM
```

とする。

例：

```json
{
  "targetYearMonth": "2026-07"
}
```

CSVは
1ファイル1対象年月であるため、
正常登録時には
必ず1つの対象年月を返却する。

---

#### 8.15.2 importedCount

実際に新規登録した

```text
month_end_holding_values
```

の件数を返却する。

例：

```json
{
  "importedCount": 2
}
```

登録件数には、
以下を含めない。

- CSVヘッダー
- 完全な空行
- `month_end_asset_snapshots`の作成件数

例えば、

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

を正常登録した場合は、

```json
{
  "importedCount": 2
}
```

となる。

---

#### 8.15.3 importedCountは0にならない

CSV-006では、
データ行が0件のCSVを
登録不可とする。

そのため、
正常レスポンスにおける

```text
importedCount
```

は、
1以上となる。

以下を
正常登録結果として返却しない。

```json
{
  "importedCount": 0
}
```

---

#### 8.15.4 返却しない情報

本APIでは、
以下の情報を返却しない。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`
- `asset_accounts.balance_recording_unit`
- `month_end_asset_snapshots.confirmed`
- `created_at`
- `updated_at`
- `asset_account_available_settings`
- CSVファイル内容
- CSV行番号
- 登録した各`asset_accounts.name`
- 登録した各`holding_assets.name`
- 登録した各`month_end_holding_values.value`

登録処理に使用した
内部情報を
そのままレスポンスへ公開しない。

---

#### 8.15.5 snapshotIdを返却しない

CSV-006では、
内部で

```text
month_end_asset_snapshots.id
```

を使用して
商品別月末評価額を登録する。

ただし、
CSV登録結果の確認に
内部snapshot IDは不要であるため、
レスポンスへ返却しない。

対象年月をキーとして
必要な参照APIを利用する。

---

#### 8.15.6 assetAccountIdを返却しない

CSV-006では、
複数の資産口座に属する
保有商品を
一括登録する可能性がある。

また、
登録結果確認に
内部資産口座IDは不要である。

そのため、

```text
assetAccountId
```

をレスポンスへ返却しない。

---

#### 8.15.7 holdingAssetIdを返却しない

CSV-006では、
複数の保有商品を
一括登録する。

登録した保有商品ID一覧を
成功レスポンスへ含めない。

登録後に
商品別月末評価額の詳細が必要な場合は、
参照APIから取得する。

---

#### 8.15.8 confirmedを返却しない

CSV-006では、
月末資産状況を
自動確定しない。

また、
確定状態の取得・表示は
月末資産状況APIの責務とする。

そのため、
CSV-006の登録結果として

```text
confirmed
```

を返却しない。

---

#### 8.15.9 登録したvalue一覧を返却しない

CSV-006では、
登録した各商品の

```text
value
```

を
成功レスポンスへ返却しない。

利用者は、
CSV-005のプレビューで
登録予定内容を確認できる。

CSV-006では、
登録成功を示す

```text
targetYearMonth
importedCount
```

だけを返却する。

---

#### 8.15.10 レスポンス例

CSV内の
2件の商品別月末評価額を
正常登録した場合の
概念例は、
以下とする。

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "importedCount": 2
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

CSV-006では、
登録対象年月と
登録件数を返却することで、
利用者が
一括登録の完了を
確認できるようにする。

---

### 8.16 エラーレスポンス

CSV-006では、
CSV-005とは異なり、
登録条件を満たさないCSVを
正常レスポンスとして返却しない。

CSV-006は
商品別月末評価額を
実際に登録するAPIであるため、
登録できない状態は
APIエラーとして扱う。

概念的には、
以下とする。

```text
CSV-005
    ↓
解析・検証
    ↓
業務エラーあり
    ↓
200 OK
canImport = false
```

```text
CSV-006
    ↓
再解析・再検証
    ↓
登録不可
    ↓
4xx Error
業務データ更新なし
```

CSV-006では、
エラーが発生した場合に
商品別月末評価額を
部分登録してはならない。

---

#### 8.16.1 エラー一覧

本APIで想定する
主なエラーは、
以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`が指定されていない |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`の形式が不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 指定された利用者が存在しない、または論理削除済み |
| `422 Unprocessable Entity` | `VALIDATION_ERROR` | `file`未指定、ファイル形式・サイズなどの入力不正 |
| `422 Unprocessable Entity` | `INVALID_CSV_FORMAT` | CSV解析不能、CSVヘッダー不正 |
| `422 Unprocessable Entity` | `CSV_DATA_REQUIRED` | CSVに登録対象のデータ行が存在しない |
| `422 Unprocessable Entity` | `MULTIPLE_TARGET_YEAR_MONTHS` | 1ファイル内に複数対象年月が存在する |
| `422 Unprocessable Entity` | `ASSET_ACCOUNT_NOT_FOUND` | CSVで指定された資産口座が操作対象利用者に存在しない |
| `422 Unprocessable Entity` | `BALANCE_RECORDING_UNIT_MISMATCH` | 口座単位の資産口座が指定されている |
| `422 Unprocessable Entity` | `HOLDING_ASSET_NOT_FOUND` | 指定資産口座にCSVで指定された保有商品が存在しない |
| `422 Unprocessable Entity` | `HOLDING_ASSET_NOT_AVAILABLE` | 対象年月時点で保有商品が記録対象として有効ではない |
| `422 Unprocessable Entity` | `INVALID_VALUE` | 商品別月末評価額が0以上の整数ではない |
| `422 Unprocessable Entity` | `DUPLICATE_HOLDING_ASSET_IN_CSV` | CSV内で同一資産口座・同一保有商品が重複している |
| `409 Conflict` | `MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED` | 対象年月の月末資産状況が確定済み |
| `409 Conflict` | `MONTH_END_HOLDING_VALUE_ALREADY_EXISTS` | 対象年月・保有商品の商品別月末評価額が既に存在する |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバー内部エラー |

実際の独自エラーコード名は、
`error-codes.md`の
既存命名規則と統一する。

---

#### 8.16.2 USER_CONTEXT_REQUIRED

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

CSV解析および
登録処理へ進まない。

---

#### 8.16.3 INVALID_USER_ID

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
0
-1
abc
1.5
```

---

#### 8.16.4 USER_NOT_FOUND

指定された利用者が
存在しない場合、
または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

他利用者の
資産口座や保有商品を使用して
登録処理を続行してはならない。

---

#### 8.16.5 file未指定

`file`が
指定されていない場合は、

```text
VALIDATION_ERROR
```

を返却する。

概念例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "file",
        "reason": "required",
        "message": "CSVファイルを指定してください。"
      }
    ]
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

---

#### 8.16.6 ファイル形式不正

許可されていない
ファイル形式の場合は、

```text
VALIDATION_ERROR
```

として扱う。

例えば、
以下を対象とする。

```text
.xlsx
.xls
.pdf
```

CSV解析処理へ進まない。

---

#### 8.16.7 ファイルサイズ超過

CSV共通仕様で定める
最大ファイルサイズを
超えている場合は、

```text
VALIDATION_ERROR
```

を返却する。

CSV解析や
業務データ検索へ進まない。

---

#### 8.16.8 空ファイル

CSVファイルが
0バイトの場合、
または有効なヘッダーを
取得できない場合は、

```text
INVALID_CSV_FORMAT
```

として扱う。

この場合、
登録処理へ進まない。

---

#### 8.16.9 CSV解析不能

CSVとして
正常に解析できない場合は、

```text
INVALID_CSV_FORMAT
```

を返却する。

例えば、
以下を対象とする。

- CSV構造が破損している
- 引用符が不正
- 列構造を正常に解釈できない
- ヘッダーを取得できない

PHPやLaravel内部の
解析エラーを
そのままレスポンスへ公開しない。

---

#### 8.16.10 CSVヘッダー不正

CSVヘッダーが
以下と一致しない場合は、

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

```text
INVALID_CSV_FORMAT
```

として扱う。

例えば、
以下は不正とする。

```csv
targetYearMonth,assetAccountName,holdingAssetName,value
```

```csv
asset_account_name,target_year_month,holding_asset_name,value
```

```csv
target_year_month,asset_account_name,holding_asset_name,value,memo
```

ヘッダー不正の場合は、
後続の業務検証へ進まない。

---

#### 8.16.11 データ行0件

CSVヘッダーは正常だが、
データ行が1件も存在しない場合は、

```text
CSV_DATA_REQUIRED
```

として登録を拒否する。

CSV-005では
プレビュー結果として

```text
canImport = false
```

を返却するが、
CSV-006では

```text
422 Unprocessable Entity
```

を返却する。

業務データは更新しない。

---

#### 8.16.12 対象年月混在

1つのCSV内に
複数の`target_year_month`が
存在する場合は、

```text
MULTIPLE_TARGET_YEAR_MONTHS
```

として登録を拒否する。

例えば、

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-06,証券口座,全世界株式,1400000
2026-07,証券口座,S&P500,800000
```

は登録しない。

---

#### 8.16.13 行単位の入力エラー

以下のような
CSV行の入力不正が存在する場合は、
CSV全体を登録しない。

- `target_year_month`未入力
- `target_year_month`形式不正
- `asset_account_name`未入力
- `holding_asset_name`未入力
- `value`未入力
- `value`整数形式不正
- `value`小数
- `value`負数

可能な範囲で
複数の行エラーを
収集してよい。

ただし、
1件でもエラーが存在すれば
登録処理へ進まない。

---

#### 8.16.14 資産口座不存在

CSVで指定された
`asset_account_name`に対応する
資産口座が
操作対象利用者に存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

他利用者に
同名資産口座が存在していても、
正常とは判定しない。

---

#### 8.16.15 残高記録単位不一致

指定された資産口座が
口座単位で管理されている場合は、

```text
BALANCE_RECORDING_UNIT_MISMATCH
```

として登録を拒否する。

口座単位の月末残高は、
CSV-003 月末資産残高CSV登録で扱う。

---

#### 8.16.16 保有商品不存在

CSVで指定された

```text
asset_account_name
+
holding_asset_name
```

に対応する
保有商品が存在しない場合は、

```text
HOLDING_ASSET_NOT_FOUND
```

として扱う。

他資産口座や
他利用者に
同名保有商品が存在していても、
正常とは判定しない。

---

#### 8.16.17 対象年月時点で保有商品が無効

指定された保有商品が
CSVの`target_year_month`時点で
商品別月末評価額の
記録対象として有効でない場合は、

```text
HOLDING_ASSET_NOT_AVAILABLE
```

として登録を拒否する。

現在時点ではなく、
対象年月時点の状態によって判定する。

---

#### 8.16.18 商品別月末評価額不正

`value`が
0以上の整数ではない場合は、

```text
INVALID_VALUE
```

として扱う。

例えば、
以下は不正とする。

```text
-1
1000.5
abc
¥1000
1,000
```

以下は正常とする。

```text
0
1
1000
```

---

#### 8.16.19 CSV内重複

同一CSV内で
同じ資産口座・保有商品が
複数回指定されている場合は、

```text
DUPLICATE_HOLDING_ASSET_IN_CSV
```

として登録を拒否する。

先勝ち、
後勝ち、
評価額合算などによる
自動解決は行わない。

---

#### 8.16.20 確定済み月末資産状況

CSV-006実行時点で、
対象年月の
月末資産状況が

```text
confirmed = true
```

の場合は、

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

を返却する。

HTTPステータスは、

```text
409 Conflict
```

とする。

CSV-005実行時点で
未確定であった場合でも、
CSV-006実行時点の
最新状態を優先する。

---

#### 8.16.21 既存商品別月末評価額

同一の

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

について、
既に商品別月末評価額が
存在する場合は、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

を返却する。

HTTPステータスは、

```text
409 Conflict
```

とする。

既存レコードを
CSV値によって上書きしない。

---

#### 8.16.22 同時実行によるUNIQUE制約違反

アプリケーション側の
重複確認後に、
別リクエストによって
同じ商品別月末評価額が
先に登録される可能性がある。

この場合、
データベースのUNIQUE制約によって
重複登録を防止する。

概念的な一意制約は、

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

とする。

この制約違反は、
内部エラーとして
そのまま返却せず、

```text
409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

へ変換する。

---

#### 8.16.23 複数エラーの扱い

登録トランザクション開始前の
CSV検証段階では、
可能な範囲で
複数エラーを収集してよい。

例えば、

```text
2行目
ASSET_ACCOUNT_NOT_FOUND

4行目
HOLDING_ASSET_NOT_FOUND

6行目
INVALID_VALUE
```

のように、
利用者が一度に
複数箇所を修正できる形とする。

ただし、
CSV-006では
エラーが1件でも存在すれば
登録処理へ進まない。

---

#### 8.16.24 エラー詳細

CSV行に関する
複数エラーを返却する場合は、
API共通エラーの
`details`を使用してよい。

概念例：

```json
{
  "error": {
    "code": "CSV_IMPORT_VALIDATION_FAILED",
    "message": "CSVの内容に誤りがあります。",
    "details": [
      {
        "rowNumber": 2,
        "field": "holding_asset_name",
        "code": "HOLDING_ASSET_NOT_FOUND",
        "message": "指定された保有商品が存在しません。"
      },
      {
        "rowNumber": 4,
        "field": "value",
        "code": "INVALID_VALUE",
        "message": "商品別月末評価額は0以上の整数で指定してください。"
      }
    ]
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

具体的な
集約エラーコードの有無や
`details`構造は、
API共通エラー仕様に従う。

---

#### 8.16.25 INTERNAL_SERVER_ERROR

CSV解析、
月末資産状況作成、
商品別月末評価額登録などで
想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスには、
以下の内部情報を含めない。

- SQL
- PostgreSQLのエラー内容
- 制約名
- テーブル名
- カラム名
- PHP内部エラー
- Laravel内部例外
- スタックトレース
- サーバーファイルパス

詳細情報は、
サーバーログへ記録する。

---

### 8.17 HTTPステータス

本APIで使用する
HTTPステータスは、
以下とする。

| HTTPステータス | 用途 |
|---|---|
| `201 Created` | CSV内の商品別月末評価額を全件登録できた |
| `400 Bad Request` | 利用者コンテキストの指定不備 |
| `404 Not Found` | 指定された利用者が存在しない |
| `409 Conflict` | 確定済み月末資産状況、既存商品別月末評価額など現在状態との競合 |
| `422 Unprocessable Entity` | CSVファイル・CSV内容・業務入力値が登録条件を満たさない |
| `500 Internal Server Error` | 想定外のサーバー内部エラー |

CSV-006は
一括登録APIであるため、
登録不可のCSVに対して

```text
200 OK
canImport = false
```

を返却しない。

---

#### 8.17.1 201 Created

CSV内の
すべての商品別月末評価額について
検証および登録が成功した場合は、

```text
201 Created
```

を返却する。

対象年月の
月末資産状況を
CSV-006内で新規作成した場合も、
同じステータスとする。

---

#### 8.17.2 400 Bad Request

以下の場合に使用する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
```

CSV内容の
入力エラーには使用しない。

---

#### 8.17.3 404 Not Found

指定された利用者が
存在しない、
または論理削除済みの場合に使用する。

```text
USER_NOT_FOUND
```

CSV内で指定された
資産口座や保有商品が存在しない場合には、
`404 Not Found`を使用しない。

CSV内容の登録不成立として
`422 Unprocessable Entity`を使用する。

---

#### 8.17.4 409 Conflict

リクエスト内容自体は
解釈可能だが、
現在の業務データ状態と
競合して登録できない場合に使用する。

主に以下を対象とする。

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

並行登録による
UNIQUE制約違反も、
既存商品別月末評価額との競合として
`409 Conflict`へ変換する。

---

#### 8.17.5 422 Unprocessable Entity

CSVファイルまたは
CSV内容が
登録条件を満たさない場合に使用する。

主に以下を対象とする。

- `file`未指定
- ファイル形式不正
- ファイルサイズ超過
- CSV解析不能
- CSVヘッダー不正
- データ行0件
- 対象年月形式不正
- 対象年月混在
- 資産口座名未入力
- 資産口座不存在
- 残高記録単位不一致
- 保有商品名未入力
- 保有商品不存在
- 対象年月時点の保有商品無効
- 商品別月末評価額不正
- CSV内重複

---

#### 8.17.6 500 Internal Server Error

想定外の
サーバー内部エラーが
発生した場合に使用する。

対象となる
独自エラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

トランザクション中の場合は、
すべてロールバックしたうえで
エラーを返却する。

---

### 8.18 副作用

本APIには、
業務データに対する
副作用がある。

正常終了した場合は、
CSV内容に基づいて

```text
month_end_holding_values
```

へ
商品別月末評価額を新規登録する。

また、
対象年月の
月末資産状況が存在しない場合は、

```text
month_end_asset_snapshots
```

を新規作成する。

---

#### 8.18.1 正常終了時の更新

対象年月の
月末資産状況が
既に存在する場合は、
概念的に以下となる。

```text
month_end_asset_snapshots
    → 更新なし

month_end_holding_values
    → CSV行数分INSERT
```

対象年月の
月末資産状況が
存在しない場合は、

```text
month_end_asset_snapshots
    → 1件INSERT

month_end_holding_values
    → CSV行数分INSERT
```

となる。

---

#### 8.18.2 更新しないデータ

CSV-006では、
以下を更新しない。

- `users`
- `asset_accounts`
- `holding_assets`
- `month_end_asset_balances`
- `asset_account_available_settings`
- `net_incomes`
- `objectives`
- `assessment_histories`

また、
既存の

```text
month_end_holding_values.value
```

も更新しない。

---

#### 8.18.3 confirmedを変更しない

CSV登録によって
月末資産状況の

```text
confirmed
```

を変更しない。

新規作成する場合は、

```text
confirmed = false
```

とする。

既存snapshotについても、
CSV-006によって
確定状態を変更しない。

---

#### 8.18.4 エラー時の副作用

CSV解析・検証段階で
エラーとなった場合は、
業務データを
一切更新しない。

トランザクション開始後に
エラーが発生した場合は、
すべてロールバックする。

そのため、

```text
一部の商品だけ登録済み
```

または、

```text
snapshotだけ新規作成済み
```

という状態を
残してはならない。

---

### 8.19 トランザクション

CSV-006では、
登録処理を
データベーストランザクション内で実行する。

CSV全体の
入力・業務検証を行った後、
登録可能であることを確認してから
トランザクションを開始する。

概念的には、
以下とする。

```text
CSV解析
    ↓
全件検証
    ↓
登録可能
    ↓
BEGIN
    ↓
月末資産状況取得
    ↓
確定状態再確認
    ↓
必要ならsnapshot作成
    ↓
既存商品別月末評価額再確認
    ↓
商品別月末評価額一括登録
    ↓
COMMIT
```

---

#### 8.19.1 同一トランザクションで行う処理

主に以下を
同一トランザクション内で行う。

- 月末資産状況の取得
- 確定状態の最終確認
- 月末資産状況の必要時作成
- 既存商品別月末評価額の最終確認
- 商品別月末評価額の一括登録

これにより、
snapshot作成と
商品別月末評価額登録を
原子的に処理する。

---

#### 8.19.2 ロールバック

トランザクション内で
業務例外または
想定外例外が発生した場合は、
すべてロールバックする。

例えば、

```text
snapshot新規作成
    ↓
1件目の商品別月末評価額登録
    ↓
2件目登録時にエラー
    ↓
ROLLBACK
```

となった場合は、

- 新規snapshot
- 1件目の商品別月末評価額

の両方を
データベースへ残さない。

---

#### 8.19.3 CSV検証処理をトランザクションへ入れすぎない

CSVファイルの

- 読み込み
- ヘッダー検証
- 入力値検証
- 基本業務検証

などは、
原則として
トランザクション開始前に行う。

CSV解析中ずっと
トランザクションを保持しない。

これにより、
トランザクション時間を
必要最小限にする。

---

### 8.20 ロック

CSV-006では、
同一利用者・同一対象年月への
並行登録を考慮する。

登録トランザクション内では、
既存の月末資産状況が存在する場合、
必要に応じて
対象snapshotを
行ロックする。

概念例：

```php
$snapshot =
    MonthEndAssetSnapshot::query()
        ->where(
            'user_id',
            $userId,
        )
        ->where(
            'target_year_month',
            $targetYearMonth,
        )
        ->lockForUpdate()
        ->first();
```

これにより、
同じsnapshotに対する

- CSV-006
- 月末資産状況確定
- その他の更新処理

との競合を制御する。

---

#### 8.20.1 snapshot不存在時の競合

対象年月のsnapshotが
存在しない状態で、
複数のCSV登録処理が
同時実行される可能性がある。

例えば、

```text
CSV-003
月末資産残高CSV登録

CSV-006
商品別月末評価額CSV登録
```

が
同一利用者・同一対象年月へ
同時実行される場合もある。

双方が

```text
snapshot不存在
```

と判定して
重複作成しないよう、
以下の組み合わせに対する
データベースの一意制約を
最終防衛線とする。

```text
month_end_asset_snapshots.user_id
+
month_end_asset_snapshots.target_year_month
```

---

#### 8.20.2 商品別月末評価額の重複防止

`month_end_holding_values`について、
以下の組み合わせに
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

アプリケーション側の
事前重複確認を
複数リクエストが同時に通過しても、
データベース制約によって
重複登録を防止する。

---

#### 8.20.3 過剰なロックを行わない

CSV-006では、
操作対象利用者の

- すべての資産口座
- すべての保有商品

を長時間ロックするような
実装は避ける。

排他制御は、
登録対象となる
月末資産状況を中心に
必要最小限とする。

CSV解析処理中には
行ロックを取得しない。

---

### 8.21 キャッシュ

Phase1では、
CSV-006専用の
サーバー側アプリケーションキャッシュを
使用しない。

CSV登録では、

- 月末資産状況
- 商品別月末評価額
- 資産口座
- 保有商品

の最新状態を
確認する必要があるためである。

古いキャッシュを利用して
登録可否を判定してはならない。

---

#### 8.21.1 登録後のキャッシュ

将来的に、
商品別月末評価額や
月末資産状況に関する
サーバー側キャッシュを
採用する場合は、
CSV-006成功時に
対象年月に関係するキャッシュを
無効化する必要がある。

Phase1では、
サーバー側キャッシュを
採用しないため、
明示的なキャッシュ削除処理も
実装しない。

---

### 8.22 冪等性

本APIは、
HTTP POSTを使用して
新しい商品別月末評価額を
一括登録するAPIである。

そのため、
HTTPメソッドとしては
冪等ではない。

同じCSVを
複数回実行した場合、
2回目に
同じ商品別月末評価額を
新たに登録してはならない。

---

#### 8.22.1 同一CSVの再実行

例えば、
以下のCSVを送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

1回目は、

```text
CSV-006
    ↓
201 Created
importedCount = 2
```

となる。

同じCSVを
再度実行した場合は、

```text
CSV-006
    ↓
409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

となる。

既存データを返して
成功扱いにはしない。

---

#### 8.22.2 値が異なる場合も上書きしない

既に、

```text
全世界株式
value = 1500000
```

が登録されている状態で、

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1600000
```

をCSV-006へ送信しても、
既存値を更新しない。

```text
409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

として扱う。

評価額を変更する場合は、
商品別月末評価額更新APIを使用する。

---

#### 8.22.3 一部だけ既存の場合

CSV内の一部の保有商品について
商品別月末評価額が
既に存在する場合も、
未登録行だけを登録しない。

例えば、

```text
全世界株式
    → 登録済み

S&P500
    → 未登録
```

の状態で
2件を含むCSVを送信した場合は、

```text
CSV全体
    ↓
409 Conflict
    ↓
新規登録0件
```

とする。

全件成功または
全件失敗のルールを維持する。

---

#### 8.22.4 Idempotency-Key

Phase1では、
`Idempotency-Key`を採用しない。

意図しない二重登録は、

- アプリケーション側の重複確認
- トランザクション
- データベースUNIQUE制約
- フロントエンドの二重送信防止

によって制御する。

---

#### 8.22.5 二重送信

フロントエンドでは、
CSV-006実行中に
登録ボタンを非活性化し、
意図しない連続送信を防止する。

ただし、
フロントエンドの制御だけを
重複登録防止の保証としてはならない。

バックエンドおよび
データベース制約によって
最終的な整合性を保証する。

---

#### 8.22.6 通信失敗後の再送

CSV-006実行後に
クライアントが
レスポンスを受信できなかった場合、
サーバー側では
登録が完了している可能性がある。

その状態で
同じCSVを再送した場合は、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

となる可能性がある。

Phase1では、
`Idempotency-Key`を
採用しないため、
再送を
初回リクエストと同じ成功結果へ
自動的に変換しない。

必要に応じて、
商品別月末評価額の参照APIから
現在状態を再取得する。

---

#### 8.22.7 同時実行

同一CSVまたは
同一対象年月・同一保有商品を含む
複数のCSV-006が
同時実行された場合でも、
同一の商品別月末評価額を
複数件登録してはならない。

最終的には、

```text
UNIQUE (
    month_end_asset_snapshot_id,
    holding_asset_id
)
```

によって
1件のみ登録可能とする。

競合したリクエストは、

```text
409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

として処理する。

本API自体は
非冪等であるが、
重複データを
許容するという意味ではない。

---

### 8.23 関連テーブル

CSV-006では、
商品別月末評価額CSVの内容を検証し、
登録可能な場合に
商品別月末評価額を一括登録するため、
以下のテーブルを使用する。

| テーブル | 用途 | 更新 |
|---|---|:---:|
| `users` | 操作対象利用者の確認 | × |
| `asset_accounts` | CSVで指定された資産口座の特定、残高記録単位の確認 | × |
| `holding_assets` | CSVで指定された保有商品の特定 | × |
| `asset_account_available_settings` | 対象年月時点での記録対象判定 | × |
| `month_end_asset_snapshots` | 対象年月の月末資産状況確認、必要時の新規作成、確定状態確認 | ○ |
| `month_end_holding_values` | 既存評価額の重複確認およびCSV内容の一括登録 | ○ |

CSV-006では、
`month_end_asset_snapshots`および
`month_end_holding_values`を
更新対象とする。

その他のテーブルは、
参照のみとする。

---

#### 8.23.1 users

`X-User-Id`で指定された
操作対象利用者の
存在確認に使用する。

概念的な条件は、
以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者コンテキストの確認は、
API共通Middlewareで行う。

CSV-006では、
`users`を更新しない。

---

#### 8.23.2 asset_accounts

CSVの

```text
asset_account_name
```

から、
操作対象利用者に属する
資産口座を特定するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 保有商品の所属資産口座確認 |
| `user_id` | 利用者境界確認 |
| `name` | CSVの`asset_account_name`との照合 |
| `balance_recording_unit` | 商品単位CSVの登録対象か判定 |
| `deleted_at` | 論理削除済み資産口座を除外 |

概念的な検索条件は、
以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    IN CSVで指定された資産口座名

AND

asset_accounts.deleted_at
    IS NULL
```

他利用者に
同名資産口座が存在していても、
登録対象へ含めない。

---

#### 8.23.3 残高記録単位

`asset_accounts.balance_recording_unit`を使用して、
商品別月末評価額CSVの
登録対象であることを確認する。

CSV-006では、

```text
商品単位
```

の資産口座のみを
登録対象とする。

口座単位の資産口座については、
`month_end_holding_values`へ
登録しない。

---

#### 8.23.4 holding_assets

CSVの

```text
asset_account_name
+
holding_asset_name
```

から、
登録対象となる
保有商品を特定するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | `month_end_holding_values.holding_asset_id`へ設定 |
| `asset_account_id` | 指定資産口座との関連確認 |
| `name` | CSVの`holding_asset_name`との照合 |
| `deleted_at` | 論理削除済み保有商品を除外 |

概念的な条件は、
以下とする。

```text
holding_assets.asset_account_id
    = CSVのasset_account_nameから特定した
      asset_accounts.id

AND

holding_assets.name
    = CSVのholding_asset_name

AND

holding_assets.deleted_at
    IS NULL
```

保有商品名だけを条件として
システム全体から検索しない。

---

#### 8.23.5 同名保有商品の識別

異なる資産口座に
同名保有商品が存在する場合でも、
以下の組み合わせによって
登録対象を識別する。

```text
asset_accounts.id
+
holding_assets.name
```

例えば、

```text
証券口座A
    全世界株式

証券口座B
    全世界株式
```

が存在する場合は、
CSVの`asset_account_name`に対応する
資産口座配下の商品だけを使用する。

---

#### 8.23.6 asset_account_available_settings

CSVの対象年月時点で、
指定された保有商品が
商品別月末評価額の
記録対象として有効かを
判定するために参照する。

判定では、

```text
target_year_month
```

を基準とする。

現在日時点の設定だけで
登録可否を判定しない。

具体的な参照条件は、
資産口座・利用可能資産設定の
テーブル定義および
機能要件に従う。

CSV-006では、
`asset_account_available_settings`を
更新しない。

---

#### 8.23.7 month_end_asset_snapshots

CSVの
`target_year_month`に対応する
月末資産状況を
取得するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 商品別月末評価額の登録先snapshot |
| `user_id` | 利用者境界確認 |
| `target_year_month` | CSVの対象年月との照合 |
| `confirmed` | CSV登録可否判定 |

概念的な検索条件は、
以下とする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = CSVのtarget_year_month
```

他利用者の
同一対象年月snapshotを
登録先として使用しない。

---

#### 8.23.8 月末資産状況が存在する場合

対象年月の
月末資産状況が存在する場合は、
既存snapshotを使用する。

ただし、

```text
confirmed = true
```

の場合は、
CSV登録を許可しない。

未確定の場合のみ、
商品別月末評価額の
登録先として使用する。

---

#### 8.23.9 月末資産状況が存在しない場合

対象年月の
月末資産状況が
存在しない場合は、
登録トランザクション内で
新規作成する。

概念的には、
以下を設定する。

```text
user_id
    = 操作対象利用者ID

target_year_month
    = CSVのtarget_year_month

confirmed
    = false
```

作成した

```text
month_end_asset_snapshots.id
```

を、
CSV各行の

```text
month_end_holding_values.month_end_asset_snapshot_id
```

として使用する。

---

#### 8.23.10 month_end_asset_snapshotsの一意性

同一利用者、
同一対象年月について
複数のsnapshotが
作成されないようにする。

概念的には、
以下の組み合わせに
一意性を保証する。

```text
user_id
+
target_year_month
```

同時実行によって
重複作成されないよう、
データベース制約を
最終防衛線として使用する。

---

#### 8.23.11 month_end_holding_values

CSV内容に基づいて、
商品別月末評価額を
新規登録するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 商品別月末評価額ID |
| `month_end_asset_snapshot_id` | 対象年月の月末資産状況 |
| `holding_asset_id` | 登録対象保有商品 |
| `value` | CSVの商品別月末評価額 |
| `created_at` | 登録日時 |
| `updated_at` | 更新日時 |

CSV各行について、
概念的に以下を登録する。

```text
month_end_asset_snapshot_id
    = 対象snapshot.id

holding_asset_id
    = asset_account_name
      +
      holding_asset_name
      から特定したholding_assets.id

value
    = CSVのvalue
```

---

#### 8.23.12 既存商品別月末評価額の確認

登録前に、
同一の

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

について、
既存の

```text
month_end_holding_values
```

が存在しないことを確認する。

既存データが存在する場合は、
新規登録しない。

CSV-006では、
既存の`value`を
更新・上書きしない。

---

#### 8.23.13 month_end_holding_valuesの一意性

同一snapshot、
同一保有商品について
複数の商品別月末評価額が
登録されないようにする。

概念的には、
以下の組み合わせに
一意性を保証する。

```text
UNIQUE (
    month_end_asset_snapshot_id,
    holding_asset_id
)
```

アプリケーション側の
重複確認後に
並行リクエストが発生した場合でも、
データベース制約によって
重複登録を防止する。

---

#### 8.23.14 更新対象テーブル

正常終了時に
更新する可能性があるテーブルは、
以下とする。

```text
month_end_asset_snapshots
month_end_holding_values
```

`month_end_asset_snapshots`は、
対象年月のsnapshotが
存在しない場合のみ
新規作成する。

`month_end_holding_values`は、
CSVの正常なデータ行数分を
新規登録する。

---

#### 8.23.15 更新しないテーブル

以下のテーブルは、
CSV-006では更新しない。

- `users`
- `asset_accounts`
- `holding_assets`
- `asset_account_available_settings`
- `month_end_asset_balances`
- `net_incomes`
- `objectives`
- `assessment_histories`

CSV登録によって、
資産口座、
保有商品、
利用可能設定などを
変更してはならない。

---

#### 8.23.16 month_end_asset_balances

CSV-006では、

```text
month_end_asset_balances
```

を参照・登録対象としない。

口座単位の月末資産残高は、
CSV-003 月末資産残高CSV登録で扱う。

商品別月末評価額CSV登録によって
口座単位残高を
自動生成しない。

---

### 8.24 関連する機能要件

CSV-006は、
商品別月末評価額の
CSV一括登録に関する
機能要件と対応する。

主な関連要件は、
以下とする。

- CSVインポート
  - CSVテンプレートを使用してデータを入力できる
  - 登録前にCSV内容をプレビューできる
  - CSV登録時に入力内容を再検証する
  - 1つのCSVファイルでは1つの対象年月のみを扱う
  - CSV内にエラーが存在する場合は一部登録しない
  - CSV登録は全件成功または全件失敗とする

- 商品別月末評価額
  - 商品単位で管理する資産口座の保有商品について月末評価額を登録できる
  - 商品別月末評価額は日本円の整数として扱う
  - 0円を有効な評価額として扱う
  - 同一対象年月・同一保有商品への重複登録を防止する
  - 既存商品別月末評価額をCSVで上書きしない

- 資産口座
  - 操作対象利用者に属する資産口座のみ登録対象とする
  - CSVでは資産口座名を使用して対象口座を特定する
  - 残高記録単位が商品単位の資産口座のみ対象とする

- 保有商品
  - CSVでは保有商品名を使用して対象商品を特定する
  - 保有商品は指定された資産口座に属している必要がある
  - 対象年月時点で記録対象として有効な保有商品のみ登録対象とする

- 月末資産状況
  - 対象年月ごとに月末資産状況を管理する
  - 対象年月の月末資産状況が存在しない場合は登録処理で作成できる
  - 新規作成する月末資産状況は未確定とする
  - 確定済み月末資産状況へ商品別月末評価額を追加できない
  - CSV登録によって月末資産状況を自動確定しない

- 利用者境界
  - 他利用者の資産口座を登録対象にしない
  - 他利用者の保有商品を登録対象にしない
  - 他利用者の月末資産状況を使用しない
  - 他利用者の商品別月末評価額を重複判定へ使用しない

- トランザクション
  - CSV内の複数行を一括して登録する
  - 登録途中でエラーが発生した場合は全件ロールバックする
  - 月末資産状況を新規作成した場合も同一トランザクションで扱う

具体的な章番号は、
`functional-requirements.md`の
最新定義に従う。

---

### 8.25 インデックス

CSV-006では、
主に以下の検索条件を使用する。

```text
asset_accounts.user_id
+
asset_accounts.name
```

```text
holding_assets.asset_account_id
+
holding_assets.name
```

```text
month_end_asset_snapshots.user_id
+
month_end_asset_snapshots.target_year_month
```

```text
month_end_holding_values.month_end_asset_snapshot_id
+
month_end_holding_values.holding_asset_id
```

必要なインデックスは、
テーブル全体の
利用状況を考慮して設定する。

CSV-006専用として
不要な重複インデックスを
追加しない。

---

#### 8.25.1 asset_accounts

利用者内の
資産口座名検索で、

```text
user_id
+
name
```

を使用する。

利用者内で
資産口座名を一意とする設計であれば、
UNIQUE制約による
インデックスを利用できる。

---

#### 8.25.2 holding_assets

資産口座内の
保有商品名検索では、

```text
asset_account_id
+
name
```

を使用する。

資産口座内で
保有商品名を一意とする設計であれば、
その制約による
インデックスを活用する。

---

#### 8.25.3 month_end_asset_snapshots

以下の組み合わせで
対象snapshotを取得する。

```text
user_id
+
target_year_month
```

同一利用者・同一対象年月の
snapshot一意性を保証する
UNIQUE制約を利用する。

---

#### 8.25.4 month_end_holding_values

以下の組み合わせで
既存商品別月末評価額を確認する。

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

同時に、
この組み合わせは
重複登録防止の
UNIQUE制約として使用する。

---

### 8.26 性能

CSV-006では、
CSV行ごとに
データベースアクセスを行わず、
必要なデータを
可能な限り一括取得する。

概念的には、
以下とする。

```text
CSV全体解析
    ↓
必要キー抽出
    ↓
asset_accounts一括取得
    ↓
holding_assets一括取得
    ↓
available_settings一括取得
    ↓
snapshot取得
    ↓
existingValues一括取得
    ↓
メモリ上で全件検証
    ↓
トランザクション
    ↓
一括INSERT
```

---

#### 8.26.1 N+1問題

以下のような
CSV行単位の検索を
行わない。

```text
1行目
    ↓
asset_account検索
holding_asset検索
existingValue検索

2行目
    ↓
asset_account検索
holding_asset検索
existingValue検索
```

CSV行数に比例して
SQL発行回数が
増加する実装を避ける。

---

#### 8.26.2 資産口座一括取得

CSVから
重複を除いた

```text
asset_account_name
```

を抽出し、
操作対象利用者に属する
資産口座を
まとめて取得する。

取得後は、

```text
assetAccountName
    → AssetAccount
```

のMapとして
利用してよい。

---

#### 8.26.3 保有商品一括取得

CSVで指定された

```text
asset_account_name
+
holding_asset_name
```

をもとに、
対象保有商品を
まとめて取得する。

取得後は、

```text
assetAccountId
+
holdingAssetName
```

をキーとして
Map化してよい。

---

#### 8.26.4 既存商品別月末評価額の一括取得

snapshotが存在する場合は、
CSVで対象となる
保有商品IDをまとめて使用し、

```text
month_end_holding_values
```

を一括取得する。

CSV行ごとに
重複確認SQLを
発行しない。

---

#### 8.26.5 一括INSERT

すべての検証に
成功した場合は、
CSV行数分の
`month_end_holding_values`を
可能な限り
一括INSERTする。

概念的には、

```text
100行CSV
    ↓
100回INSERT
```

ではなく、

```text
100行CSV
    ↓
1回または少数回のINSERT
```

を基本とする。

PostgreSQLの
パラメータ上限などを考慮し、
必要であれば
一定件数でchunkしてよい。

---

#### 8.26.6 トランザクション時間

CSVファイルの解析から
トランザクションを開始しない。

以下を
トランザクション開始前に行う。

- CSVファイル解析
- ヘッダー検証
- 行入力値検証
- 基本業務検証

その後、

```text
DB::transaction
    ↓
snapshot最終確認
    ↓
既存評価額最終確認
    ↓
登録
```

とする。

ロック・トランザクション保持時間を
必要最小限にする。

---

#### 8.26.7 非同期処理

Phase1では、
CSV-006を
同期APIとして実装する。

以下は導入しない。

```text
Queue
Job
Batch
Polling
WebSocket
```

CSV規模が
同期処理では扱えないほど
大きくなった場合に、
将来拡張として検討する。

---

### 8.27 セキュリティ

CSV-006では、
アップロードされたCSVをもとに
業務データを登録するため、
利用者境界と
入力値の信頼境界を
明確にする。

---

#### 8.27.1 利用者境界

以下の検索・登録では、
常に
`X-User-Id`から特定した
操作対象利用者との
関連を保証する。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

CSV内の名称だけを使用して、
他利用者のデータへ
登録してはならない。

---

#### 8.27.2 内部IDをCSVから受け取らない

CSV仕様には、
以下の内部IDを含めない。

- `user_id`
- `asset_account_id`
- `holding_asset_id`
- `month_end_asset_snapshot_id`
- `month_end_holding_value_id`

内部IDは、
サーバー側で
利用者境界を確認したうえで
特定する。

---

#### 8.27.3 SQLインジェクション対策

CSVの

```text
asset_account_name
holding_asset_name
```

を検索条件として使用する場合でも、
SQL文字列へ
直接連結しない。

Laravelの

- Eloquent
- Query Builder
- バインドパラメータ

を使用する。

---

#### 8.27.4 ファイルサイズ制限

過大なCSVによって
サーバー資源が
過剰消費されないよう、
CSV共通仕様の
ファイルサイズ上限を適用する。

必要に応じて
CSV行数上限も
共通仕様として設定する。

---

#### 8.27.5 CSVを実行しない

CSV内容は、
データとしてのみ扱う。

以下として
評価・実行しない。

- PHPコード
- SQL
- シェルコマンド
- テンプレートコード

---

#### 8.27.6 ファイルを永続保存しない

Phase1では、
アップロードCSVを
永続保存しない。

一時ファイルを扱う場合は、
Laravel・PHPの
通常のアップロード管理に従う。

利用者が
任意のサーバーファイルパスを
指定できない設計とする。

---

#### 8.27.7 エラー情報

エラーレスポンスへ、
以下の内部情報を含めない。

- SQL
- テーブル名
- カラム名
- UNIQUE制約名
- PostgreSQL内部エラー
- スタックトレース
- Laravel内部例外
- サーバーファイルパス

必要な詳細情報は、
サーバーログへ記録する。

---

### 8.28 ログ・監視

CSV-006では、
API共通ログ方針に従って
登録処理の結果を記録する。

商品別月末評価額は
利用者の資産情報であるため、
CSV内容全体を
ログへ出力しない。

---

#### 8.28.1 ログコンテキスト

必要に応じて、
以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
targetYearMonth
rowCount
importedCount
errorCode
```

`apiId`は、

```text
CSV-006
```

とする。

---

#### 8.28.2 正常時

正常登録時は、
必要に応じて
以下を記録する。

```text
requestId
userId
apiId = CSV-006
targetYearMonth
importedCount
httpStatus = 201
```

登録した
各`value`を
全件ログへ出力する必要はない。

---

#### 8.28.3 異常時

業務エラーまたは
想定外エラー時は、
必要に応じて
以下を記録する。

```text
requestId
userId
apiId
targetYearMonth
errorCode
httpStatus
```

想定外例外については、
調査に必要な内部情報を
サーバーログへ記録する。

---

#### 8.28.4 ログへ出力しない情報

原則として、
以下を不要にログへ記録しない。

- CSVファイル全文
- 全`value`
- 資産口座名一覧
- 保有商品名一覧
- multipartリクエストボディ全文

障害調査上
必要な場合でも、
最小限の情報に限定する。

---

### 8.29 テスト観点

CSV-006では、
CSV入力、
利用者境界、
業務ルール、
一括登録、
トランザクション、
同時実行を
重点的に確認する。

CSV-005と
共通化している検証ロジックについては、
同一入力・同一DB状態で
判定が一致することも確認する。

---

#### 8.29.1 正常系

正常なCSVを送信する。

例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

以下を確認する。

- `201 Created`となること
- `targetYearMonth = 2026-07`となること
- `importedCount = 2`となること
- 2件の`month_end_holding_values`が登録されること
- 登録された`value`がCSVと一致すること
- 各行が正しい`holding_asset_id`へ紐づくこと
- 各行が同一の`month_end_asset_snapshot_id`へ紐づくこと
- CSV登録によってsnapshotが確定されないこと

---

#### 8.29.2 CSV-004との整合性

CSV-004で取得した
正式テンプレートへ
正常なデータを入力し、
CSV-006へ送信する。

以下を確認する。

```text
CSV-004
生成ヘッダー
    =
CSV-006
受付ヘッダー
```

正式テンプレートが
ヘッダー不正にならないこと。

---

#### 8.29.3 CSV-005との整合性

同一利用者、
同一DB状態、
同一CSVについて、

```text
CSV-005
canImport = true
```

となる場合に、
CSV-006でも
登録可能となることを確認する。

CSV-005とCSV-006で
CSV解析・業務ルールの
実装差異がないことを確認する。

---

#### 8.29.4 file未指定

`file`を指定せずに
CSV-006を実行する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

以下も確認する。

- CSV解析を行わないこと
- snapshotを作成しないこと
- 商品別月末評価額を登録しないこと

---

#### 8.29.5 CSV以外のファイル

例えば、

```text
test.xlsx
```

を送信する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

業務データが
更新されないこと。

---

#### 8.29.6 ファイルサイズ超過

CSV共通仕様で定める
上限を超えるファイルを送信する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

登録処理へ
進まないこと。

---

#### 8.29.7 空ファイル

0バイトのCSVを送信する。

期待結果：

```text
422 Unprocessable Entity
INVALID_CSV_FORMAT
```

以下が
新規作成されないこと。

- `month_end_asset_snapshots`
- `month_end_holding_values`

---

#### 8.29.8 ヘッダー不正

以下を送信する。

```csv
targetYearMonth,assetAccountName,holdingAssetName,value
2026-07,証券口座,全世界株式,1500000
```

期待結果：

```text
422 Unprocessable Entity
INVALID_CSV_FORMAT
```

登録処理へ
進まないこと。

---

#### 8.29.9 ヘッダー順序不正

以下を送信する。

```csv
asset_account_name,target_year_month,holding_asset_name,value
証券口座,2026-07,全世界株式,1500000
```

期待結果：

```text
422 Unprocessable Entity
INVALID_CSV_FORMAT
```

となること。

---

#### 8.29.10 余分なヘッダー

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value,memo
2026-07,証券口座,全世界株式,1500000,test
```

期待結果：

```text
422 Unprocessable Entity
INVALID_CSV_FORMAT
```

となること。

---

#### 8.29.11 データ行0件

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

期待結果：

```text
422 Unprocessable Entity
CSV_DATA_REQUIRED
```

以下を確認する。

- `importedCount = 0`の正常レスポンスとしないこと
- snapshotを作成しないこと
- 商品別月末評価額を登録しないこと

---

#### 8.29.12 target_year_month未入力

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,全世界株式,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 8.29.13 target_year_month形式不正

以下の値を
それぞれテストする。

```text
2026-1
2026/07
202607
2026-00
2026-13
abc
```

以下を確認する。

- `422 Unprocessable Entity`となること
- 商品別月末評価額が登録されないこと

---

#### 8.29.14 対象年月混在

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-06,証券口座,全世界株式,1400000
2026-07,証券口座,S&P500,800000
```

期待結果：

```text
422 Unprocessable Entity
MULTIPLE_TARGET_YEAR_MONTHS
```

CSV全体が
登録されないこと。

---

#### 8.29.15 asset_account_name未入力

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,,全世界株式,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 8.29.16 資産口座不存在

操作対象利用者に
存在しない資産口座を指定する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,存在しない口座,全世界株式,1500000
```

期待結果：

```text
422 Unprocessable Entity
ASSET_ACCOUNT_NOT_FOUND
```

商品別月末評価額が
登録されないこと。

---

#### 8.29.17 他利用者にのみ同名資産口座が存在する

以下の状態を用意する。

```text
User A
証券口座なし

User B
証券口座あり
```

User Aとして、

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
```

を送信する。

期待結果：

```text
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの資産口座を
使用しないこと。

---

#### 8.29.18 論理削除済み資産口座

`asset_accounts.deleted_at`が
設定された資産口座を指定する。

以下を確認する。

- 有効な資産口座として扱われないこと
- CSV全体が登録されないこと

---

#### 8.29.19 残高記録単位が商品単位

対象資産口座の

```text
balance_recording_unit
    = 商品単位
```

とする。

他の条件が正常な場合は、
商品別月末評価額を
正常に登録できること。

---

#### 8.29.20 残高記録単位が口座単位

口座単位で管理する
資産口座を指定する。

期待結果：

```text
422 Unprocessable Entity
BALANCE_RECORDING_UNIT_MISMATCH
```

以下を確認する。

- `month_end_holding_values`へ登録されないこと
- `month_end_asset_balances`へも登録されないこと

---

#### 8.29.21 holding_asset_name未入力

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 8.29.22 保有商品不存在

指定資産口座に存在しない
保有商品を指定する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,存在しない商品,1500000
```

期待結果：

```text
422 Unprocessable Entity
HOLDING_ASSET_NOT_FOUND
```

となること。

---

#### 8.29.23 他資産口座にのみ同名保有商品が存在する

以下の状態を用意する。

```text
証券口座A
    全世界株式なし

証券口座B
    全世界株式あり
```

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座A,全世界株式,1500000
```

期待結果：

```text
HOLDING_ASSET_NOT_FOUND
```

となること。

証券口座Bの商品へ
登録されないこと。

---

#### 8.29.24 他利用者にのみ同名保有商品が存在する

以下の状態を用意する。

```text
User A
証券口座
    全世界株式なし

User B
証券口座
    全世界株式あり
```

User Aとして
CSV-006を実行する。

期待結果：

```text
HOLDING_ASSET_NOT_FOUND
```

となること。

User Bの保有商品へ
登録されないこと。

---

#### 8.29.25 論理削除済み保有商品

`holding_assets.deleted_at`が
設定された保有商品を指定する。

以下を確認する。

- 有効な保有商品として扱われないこと
- 商品別月末評価額が登録されないこと

---

#### 8.29.26 対象年月時点で有効

対象年月時点で
商品別月末評価額の
記録対象として有効な
保有商品を指定する。

他の条件が正常であれば、
登録できること。

---

#### 8.29.27 対象年月時点で無効

対象年月時点で
記録対象として無効な
保有商品を指定する。

期待結果：

```text
422 Unprocessable Entity
HOLDING_ASSET_NOT_AVAILABLE
```

となること。

現在時点で有効であっても、
対象年月時点で無効なら
登録できないこと。

---

#### 8.29.28 value未入力

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 8.29.29 valueが文字列

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,abc
```

期待結果：

```text
422 Unprocessable Entity
INVALID_VALUE
```

となること。

---

#### 8.29.30 valueが小数

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1000.5
```

期待結果：

```text
422 Unprocessable Entity
INVALID_VALUE
```

となること。

---

#### 8.29.31 valueが負数

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,-1
```

期待結果：

```text
422 Unprocessable Entity
INVALID_VALUE
```

となること。

---

#### 8.29.32 valueが0円

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,0
```

他の条件が正常であれば、
登録できること。

登録後に、

```text
value = 0
```

として保持されること。

0円を
未入力扱いしないこと。

---

#### 8.29.33 桁区切り付きvalue

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,"1,500,000"
```

期待結果：

```text
422 Unprocessable Entity
INVALID_VALUE
```

となること。

---

#### 8.29.34 通貨記号付きvalue

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,¥1500000
```

期待結果：

```text
422 Unprocessable Entity
INVALID_VALUE
```

となること。

---

#### 8.29.35 CSV内重複

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,全世界株式,1600000
```

期待結果：

```text
422 Unprocessable Entity
DUPLICATE_HOLDING_ASSET_IN_CSV
```

以下を確認する。

- どちらの行も登録されないこと
- 先勝ち・後勝ちにならないこと

---

#### 8.29.36 月末資産状況が存在しない

対象年月の
`month_end_asset_snapshots`が
存在しない状態で、
正常なCSVを送信する。

以下を確認する。

- `201 Created`となること
- `month_end_asset_snapshots`が1件作成されること
- `user_id`が操作対象利用者となること
- `target_year_month`がCSV対象年月となること
- `confirmed = false`となること
- CSV行数分の`month_end_holding_values`が登録されること

---

#### 8.29.37 月末資産状況が未確定

対象年月について、

```text
confirmed = false
```

のsnapshotを用意する。

正常CSVを送信し、
既存snapshotへ
商品別月末評価額が登録されること。

新しいsnapshotが
追加作成されないこと。

---

#### 8.29.38 月末資産状況が確定済み

対象年月について、

```text
confirmed = true
```

のsnapshotを用意する。

期待結果：

```text
409 Conflict
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

以下を確認する。

- 商品別月末評価額が登録されないこと
- `confirmed`が変更されないこと

---

#### 8.29.39 他利用者の同一対象年月

以下を用意する。

```text
User A
2026-07 snapshotなし

User B
2026-07 snapshotあり
```

User Aとして
正常CSVを登録する。

以下を確認する。

- User Bのsnapshotを使用しないこと
- 必要であればUser A用snapshotが新規作成されること
- User Aのデータとして登録されること

---

#### 8.29.40 他利用者の同一対象年月が確定済み

以下を用意する。

```text
User A
2026-07 snapshotなし

User B
2026-07
confirmed = true
```

User Aとして
CSV-006を実行する。

User Bの確定状態によって
User Aの登録が
拒否されないこと。

---

#### 8.29.41 既存商品別月末評価額

同一対象年月、
同一保有商品について、
既に商品別月末評価額を用意する。

期待結果：

```text
409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

以下を確認する。

- 既存`value`が変更されないこと
- CSV値で上書きされないこと
- 新規レコードが追加されないこと

---

#### 8.29.42 既存値と同じvalue

既存の商品別月末評価額と
CSVの`value`が
同じ場合でも、
成功扱いにしない。

期待結果：

```text
409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

とする。

CSV-006を
upsert APIとして扱わないこと。

---

#### 8.29.43 既存値と異なるvalue

既存値が

```text
1500000
```

で、
CSVに

```text
1600000
```

が指定されている場合も、

```text
409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

となること。

既存値が
`1600000`へ
更新されないこと。

---

#### 8.29.44 一部だけ既存

以下の状態を用意する。

```text
全世界株式
    → 登録済み

S&P500
    → 未登録
```

両方を含むCSVを送信する。

期待結果：

```text
409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

以下を確認する。

- S&P500だけを登録しないこと
- CSV全体が失敗すること
- 新規登録件数が0件であること

---

#### 8.29.45 複数エラー

以下のようなCSVを送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,存在しない口座,全世界株式,100000
2026-07,証券口座,存在しない商品,200000
2026-07,証券口座,S&P500,-1
```

可能な範囲で
複数エラーが
返却されることを確認する。

以下も確認する。

- CSV全体が登録されないこと
- snapshotが新規作成されないこと
- 商品別月末評価額が1件も登録されないこと

---

#### 8.29.46 一部登録されないこと

3行中、
2行が正常で
1行がエラーとなるCSVを送信する。

以下を確認する。

```text
正常行
    → INSERTされない

エラー行
    → INSERTされない
```

CSV全体で
新規登録0件となること。

---

#### 8.29.47 全件検証後に登録されること

CSV前半の行が正常で、
末尾行がエラーとなるCSVを送信する。

以下を確認する。

- 前半の正常行が先に登録されないこと
- エラー発見時点でDBに部分データが存在しないこと

---

#### 8.29.48 トランザクション成功

対象年月のsnapshotが
存在しない状態で、
複数行の正常CSVを送信する。

以下を確認する。

```text
snapshot INSERT
+
month_end_holding_values
複数件 INSERT
    ↓
COMMIT
```

となること。

すべてのデータが
正常に保存されること。

---

#### 8.29.49 トランザクションロールバック

snapshot作成後、
商品別月末評価額登録中に
意図的に例外を発生させる。

以下を確認する。

```text
snapshot INSERT
    ↓
holding value INSERT
    ↓
例外
    ↓
ROLLBACK
```

結果として、

- 新規snapshotが残らないこと
- 一部の商品別月末評価額が残らないこと

を確認する。

---

#### 8.29.50 既存snapshot使用時のロールバック

既存の未確定snapshotへ
複数件登録する途中で
例外を発生させる。

以下を確認する。

- 新規登録した商品別月末評価額がすべてロールバックされること
- 既存snapshot自体は残ること
- snapshotの`confirmed`が変更されないこと

---

#### 8.29.51 snapshot重複作成防止

同一利用者、
同一対象年月について
CSV-006を並行実行する。

以下を確認する。

- `month_end_asset_snapshots`が重複作成されないこと
- `user_id + target_year_month`の一意性が維持されること
- 不整合なsnapshotが残らないこと

---

#### 8.29.52 CSV-003とのsnapshot作成競合

対象年月のsnapshotが
存在しない状態で、

```text
CSV-003
月末資産残高CSV登録
```

と

```text
CSV-006
商品別月末評価額CSV登録
```

を
同一利用者・同一対象年月へ
並行実行する。

以下を確認する。

- snapshotが1件だけ作成されること
- 両APIが異なるsnapshotを作成しないこと
- 最終的に同じsnapshotへ紐づくこと
- 一意制約違反が未処理の500エラーとして露出しないこと

---

#### 8.29.53 商品別月末評価額の同時登録

同一snapshot、
同一保有商品について
複数リクエストを
並行実行する。

以下を確認する。

- 同一商品の評価額が複数件登録されないこと
- UNIQUE制約によって重複が防止されること
- 競合したリクエストが適切な`409 Conflict`となること

---

#### 8.29.54 CSV登録後も未確定

CSV登録成功後、
対象snapshotの

```text
confirmed
```

を確認する。

以下となること。

```text
confirmed = false
```

CSV-006によって
自動確定されないこと。

---

#### 8.29.55 CSVに含まれない保有商品

対象年月時点で
商品別月末評価額の
記録対象となる保有商品が
3件存在する状態で、
そのうち2件だけを
CSVへ含める。

CSVに含まれた2件が
他の条件を満たしている場合は、
登録できること。

CSVに含まれていない
残り1件を理由として
CSV-006が失敗しないこと。

必要な商品別月末評価額が
すべて登録されているかどうかは、
月末資産状況確定処理の
責務とする。

---

#### 8.29.56 月末資産残高への副作用

CSV-006実行前後で、

```text
month_end_asset_balances
```

が変更されないことを確認する。

商品単位データの登録によって
口座単位残高を
自動作成・更新しないこと。

---

#### 8.29.57 利用者コンテキスト未指定

`X-User-Id`を
指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

以下を確認する。

- CSV解析へ進まないこと
- 業務データを更新しないこと

---

#### 8.29.58 利用者ID形式不正

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

業務データが
更新されないこと。

---

#### 8.29.59 利用者不存在

存在しない利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 8.29.60 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 8.29.61 他利用者データの非更新

User Aとして
CSV-006を実行する。

実行前後で、
User Bに属する以下が
変更されないことを確認する。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

---

#### 8.29.62 正常レスポンス契約

正常登録時、
API共通の
成功Envelope形式で
返却されることを確認する。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "importedCount": 2
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

以下を確認する。

- `201 Created`であること
- `data`がobjectであること
- `targetYearMonth`がstringであること
- `importedCount`がintegerであること
- `importedCount >= 1`であること
- JSONフィールド名がcamelCaseであること
- `requestId`が設定されること

---

#### 8.29.63 返却しない情報

正常レスポンスに、
以下の情報が
含まれていないことを確認する。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`
- `asset_accounts.balance_recording_unit`
- `month_end_asset_snapshots.confirmed`
- `created_at`
- `updated_at`
- `asset_account_available_settings`
- CSVファイル内容
- CSV行番号
- 登録した各`asset_accounts.name`
- 登録した各`holding_assets.name`
- 登録した各`month_end_holding_values.value`

---

#### 8.29.64 エラーレスポンス契約

以下の代表的な異常系について、
API共通の
エラーレスポンス形式となることを確認する。

- `USER_CONTEXT_REQUIRED`
- `INVALID_USER_ID`
- `USER_NOT_FOUND`
- `VALIDATION_ERROR`
- `INVALID_CSV_FORMAT`
- `CSV_DATA_REQUIRED`
- `MULTIPLE_TARGET_YEAR_MONTHS`
- `ASSET_ACCOUNT_NOT_FOUND`
- `BALANCE_RECORDING_UNIT_MISMATCH`
- `HOLDING_ASSET_NOT_FOUND`
- `HOLDING_ASSET_NOT_AVAILABLE`
- `INVALID_VALUE`
- `DUPLICATE_HOLDING_ASSET_IN_CSV`
- `MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED`
- `MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`
- `INTERNAL_SERVER_ERROR`

以下も確認する。

- `error.code`が設定されること
- `error.message`が設定されること
- 必要に応じて`error.details`が設定されること
- `requestId`が設定されること
- SQLが含まれないこと
- PostgreSQLの制約名が含まれないこと
- スタックトレースが含まれないこと
- サーバー内部ファイルパスが含まれないこと

---

#### 8.29.65 同一CSVの再実行

正常なCSVを1回登録した後、
同じCSVを再度送信する。

1回目：

```text
201 Created
```

2回目：

```text
409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

となること。

同じレコードが
重複登録されないこと。

---

#### 8.29.66 通信失敗後の再送

サーバー側では
CSV登録が完了したが、
クライアントが
正常レスポンスを
受信できなかった状態を想定する。

同じCSVを再送した場合に、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

となり得ることを確認する。

再送によって
重複データが
作成されないこと。

---

#### 8.29.67 Idempotency-Keyなし

`Idempotency-Key`を指定せずに
正常登録できることを確認する。

また、
`Idempotency-Key`を
重複防止の前提として
実装していないことを確認する。

重複防止は、

```text
アプリケーション側重複確認
+
トランザクション
+
UNIQUE制約
```

によって保証する。

---

#### 8.29.68 INTERNAL_SERVER_ERROR

登録処理中に
想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

- トランザクションがロールバックされること
- 一部登録データが残らないこと
- 内部情報がレスポンスへ公開されないこと

---

### 8.30 Laravel実装方針

CSV-006では、
Action、
Request、
UseCase、
CSV Definition、
CSV Parser、
CSV Validator、
業務Validator、
Query、
Repository、
DTO、
API Resource、
Responderを分離して実装する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
Middleware
    ↓
Request
    ↓
Action
    ↓
UseCase
    ├─ CSV Parser
    ├─ CSV Validator
    ├─ Import Validator
    ├─ AssetAccountQuery
    ├─ HoldingAssetQuery
    ├─ AssetAccountAvailableSettingQuery
    ├─ MonthEndAssetSnapshotQuery
    ├─ MonthEndHoldingValueQuery
    ├─ MonthEndAssetSnapshotRepository
    └─ MonthEndHoldingValueRepository
    ↓
Import Result DTO
    ↓
API Resource
    ↓
Responder
```

CSV-006では、
CSV-005と同じ
CSV解析・登録可否判定ロジックを
可能な限り共通利用する。

一方、
CSV-006固有の

- トランザクション
- snapshotの必要時作成
- 排他制御
- 既存評価額の最終確認
- 商品別月末評価額の一括登録

は、
登録UseCase内で扱う。

---

#### 8.30.1 Action

Actionは、
HTTPリクエストを受け付け、
検証済みCSVファイルと
操作対象利用者を取得し、
登録UseCaseを呼び出す。

概念例：

```php
final class ImportMonthEndHoldingValueCsvAction
{
    public function __invoke(
        ImportMonthEndHoldingValueCsvRequest $request,
        ImportMonthEndHoldingValueCsvUseCase $useCase,
        MonthEndHoldingValueCsvImportResponder $responder,
        userContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                file:
                    $request->file('file'),
            );

        return $responder->created(
            $result,
        );
    }
}
```

Actionでは、
以下を行わない。

- CSV解析
- CSVヘッダー検証
- CSV入力値検証
- 対象年月特定
- CSV内重複判定
- 資産口座検索
- 保有商品検索
- 残高記録単位判定
- 対象年月時点の有効性判定
- snapshot検索
- 確定状態判定
- 既存商品別月末評価額検索
- トランザクション制御
- lock取得
- snapshot作成
- 商品別月末評価額登録
- レスポンス変換

Actionは、
UseCase呼び出しと
Responderへの受け渡しに
責務を限定する。

---

#### 8.30.2 Request

Requestでは、
HTTPリクエストとして
CSVファイルを
受け付けられる状態かを検証する。

概念例：

```php
final class ImportMonthEndHoldingValueCsvRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'file' => [
                'required',
                'file',
                'mimes:csv',
                'max:' . config(
                    'csv.max_file_size_kb',
                ),
            ],
        ];
    }
}
```

実際の

- MIME Type
- 拡張子
- 最大ファイルサイズ

は、
CSV共通仕様に従う。

---

#### 8.30.3 Requestで行うこと

Requestでは、
主に以下を検証する。

- `file`必須
- アップロードファイルであること
- 許可されたファイル形式であること
- ファイルサイズ上限以内であること

これらは、
CSV内容を解析する前に
判定できる
HTTP入力レベルの検証とする。

---

#### 8.30.4 Requestで行わないこと

Requestでは、
以下を検証しない。

- CSVヘッダー
- CSVデータ行
- `target_year_month`
- `asset_account_name`
- `holding_asset_name`
- `value`
- 1ファイル1対象年月
- CSV内重複
- 資産口座存在確認
- 残高記録単位
- 保有商品存在確認
- 保有商品と資産口座の関連
- 対象年月時点の有効性
- snapshot存在確認
- `confirmed`
- 既存商品別月末評価額

これらは、
CSV Validator、
業務Validator、
Query、
UseCaseで扱う。

---

#### 8.30.5 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、
`X-User-Id`を検証する。

概念的には、
以下とする。

```text
X-User-Id取得
    ↓
必須チェック
    ↓
形式チェック
    ↓
users存在確認
    ↓
userContext設定
    ↓
Request
    ↓
Action
```

Action以降では、
検証済み利用者コンテキストを
使用する。

---

#### 8.30.6 UseCase

CSV登録の
ユースケース全体を担当する。

主な処理は、
以下とする。

1. CSVファイルを解析する
2. CSVヘッダーを検証する
3. CSV各行の入力値を検証する
4. 対象年月を特定する
5. CSV内重複を検証する
6. 資産口座を一括取得する
7. 保有商品を一括取得する
8. 対象年月時点の有効性を検証する
9. snapshotを事前確認する
10. 既存商品別月末評価額を確認する
11. 全件登録可能であることを確認する
12. トランザクションを開始する
13. snapshotを最新状態で再取得する
14. 必要ならsnapshotを作成する
15. `confirmed`を再確認する
16. 既存商品別月末評価額を再確認する
17. 商品別月末評価額を一括登録する
18. Import Result DTOを返す

概念的には、
以下とする。

```text
CSV解析
    ↓
事前検証
    ↓
業務データ一括取得
    ↓
登録可否判定
    ↓
DB::transaction
    ↓
snapshot最終確認
    ↓
snapshot必要時作成
    ↓
既存評価額最終確認
    ↓
一括登録
    ↓
Import Result DTO
```

---

#### 8.30.7 CSV-005との共通化

CSV-005とCSV-006では、
以下を共通利用する。

```text
MonthEndHoldingValueCsvDefinition
MonthEndHoldingValueCsvParser
MonthEndHoldingValueCsvValidator
MonthEndHoldingValueCsvImportValidator
AssetAccountQuery
HoldingAssetQuery
AssetAccountAvailableSettingQuery
MonthEndAssetSnapshotQuery
MonthEndHoldingValueQuery
```

概念的には、

```text
CSV-005
    ↓
共通解析・検証
    ↓
Preview DTO
```

```text
CSV-006
    ↓
共通解析・検証
    ↓
登録処理
```

とする。

同じ登録可否ルールを
別々に実装しない。

---

#### 8.30.8 CSV Definition

CSV-004、
CSV-005、
CSV-006で使用する
商品別月末評価額CSV仕様は、
共通Definitionへ集約する。

概念例：

```php
final class MonthEndHoldingValueCsvDefinition
{
    public const HEADERS = [
        'target_year_month',
        'asset_account_name',
        'holding_asset_name',
        'value',
    ];
}
```

CSV-006だけで
独自のヘッダー定義を持たない。

---

#### 8.30.9 CSV Parser

CSVファイル解析は、
専用Parserへ分離する。

概念例：

```php
final class MonthEndHoldingValueCsvParser
{
    public function parse(
        UploadedFile $file,
    ): ParsedCsv {
        // CSV解析
    }
}
```

Parserでは、
主に以下を行う。

- ファイルオープン
- BOM処理
- ヘッダー取得
- データ行読み込み
- 行番号保持
- 完全な空行の除外
- CSV構造異常検出

業務データ検索や
登録処理は行わない。

---

#### 8.30.10 CSVヘッダー検証

CSVヘッダーは、
共通Definitionと
完全一致することを確認する。

概念例：

```php
if (
    $parsedCsv->headers
    !== MonthEndHoldingValueCsvDefinition::HEADERS
) {
    throw new InvalidCsvFormatException();
}
```

以下を不正とする。

- ヘッダー不足
- ヘッダー名不一致
- ヘッダー順序不一致
- 余分なヘッダー

ヘッダー不正時は、
データベース検索へ進まない。

---

#### 8.30.11 CSV Validator

CSV各行の
入力値検証は、
CSV Validatorへ分離する。

概念例：

```php
final class MonthEndHoldingValueCsvValidator
{
    public function validate(
        ParsedCsv $csv,
    ): CsvValidationResult {
        // CSV入力値検証
    }
}
```

主に以下を検証する。

```text
target_year_month
    required
    YYYY-MM

asset_account_name
    required

holding_asset_name
    required

value
    required
    integer
    min:0
```

データベース検索は行わない。

---

#### 8.30.12 valueの型変換

`value`は、
正常な整数形式の場合のみ
integerへ変換する。

以下を
暗黙変換によって
正常値にしてはならない。

```text
1000abc
1000.5
1,000
¥1000
```

また、

```text
0
```

は
正常値として扱う。

---

#### 8.30.13 対象年月特定

正常に解析できた
`target_year_month`を収集し、
単一対象年月であることを確認する。

概念例：

```php
$targetYearMonths =
    collect(
        $rows,
    )
        ->pluck(
            'targetYearMonth',
        )
        ->filter()
        ->unique()
        ->values();
```

2件以上存在する場合は、

```text
MULTIPLE_TARGET_YEAR_MONTHS
```

として扱う。

---

#### 8.30.14 CSV内重複判定

同一CSV内で、
同じ

```text
targetYearMonth
+
assetAccountName
+
holdingAssetName
```

が複数存在しないことを確認する。

重複時は、

```text
DUPLICATE_HOLDING_ASSET_IN_CSV
```

として扱う。

先勝ち・後勝ちにはしない。

---

#### 8.30.15 Import Validator

業務データを使用した
登録可否判定は、
専用Validatorへ分離する。

概念的には、

```text
MonthEndHoldingValueCsvImportValidator
```

が、
以下を判定する。

- 資産口座存在
- 残高記録単位
- 保有商品存在
- 保有商品と資産口座の関連
- 対象年月時点の有効性
- snapshot確定状態
- 既存商品別月末評価額

CSV-005でも
同じValidatorを使用する。

---

#### 8.30.16 AssetAccountQuery

CSVで使用される
資産口座名を収集し、
操作対象利用者に属する
資産口座を一括取得する。

概念例：

```php
$assetAccounts =
    $this->assetAccountQuery
        ->findActiveByNames(
            userId:
                $userId,

            names:
                $assetAccountNames,
        );
```

検索条件は、
概念的に以下とする。

```text
user_id = 操作対象利用者ID
AND
name IN (...)
AND
deleted_at IS NULL
```

---

#### 8.30.17 HoldingAssetQuery

対象資産口座と
保有商品名を使用し、
対象保有商品を
一括取得する。

概念例：

```php
$holdingAssets =
    $this->holdingAssetQuery
        ->findActiveByAssetAccountsAndNames(
            assetAccountIds:
                $assetAccountIds,

            names:
                $holdingAssetNames,
        );
```

保有商品名だけで
全利用者・全資産口座から
検索しない。

---

#### 8.30.18 Map化

取得結果は、
CSV行ごとの検証を
メモリ上で行えるよう、
Map化してよい。

資産口座：

```text
assetAccountName
    → AssetAccount
```

保有商品：

```text
assetAccountId
+
holdingAssetName
    → HoldingAsset
```

これにより、
CSV行ごとの
追加SQLを避ける。

---

#### 8.30.19 残高記録単位判定

対象資産口座の

```text
balance_recording_unit
```

が
商品単位であることを確認する。

概念例：

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::HOLDING
) {
    throw new BalanceRecordingUnitMismatchException();
}
```

実際のEnum・定数名は、
共通設計に従う。

---

#### 8.30.20 対象年月時点の有効性判定

対象保有商品が、
CSVの対象年月時点で
記録対象として有効かを判定する。

必要な
`asset_account_available_settings`を
一括取得し、
業務Validatorへ渡す。

現在日時ではなく、
必ず

```text
targetYearMonth
```

を基準にする。

---

#### 8.30.21 MonthEndAssetSnapshotQuery

事前検証では、
操作対象利用者・対象年月から
snapshotを取得する。

概念例：

```php
$snapshot =
    $this->snapshotQuery
        ->findByUserAndTargetYearMonth(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,
        );
```

事前検証時には、
原則として
`lockForUpdate()`を使用しない。

---

#### 8.30.22 snapshot不存在

事前検証時に
snapshotが存在しなくても、
それだけでは
登録不可としない。

CSV-006では、
トランザクション内で
必要に応じて作成する。

---

#### 8.30.23 確定済みsnapshot

事前検証時点で、

```text
confirmed = true
```

の場合は、

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

として登録不可とする。

ただし、
最終的な状態確認は
トランザクション内でも行う。

---

#### 8.30.24 MonthEndHoldingValueQuery

snapshotが存在する場合は、
対象保有商品について
既存の商品別月末評価額を
一括取得する。

概念例：

```php
$existingValues =
    $this->monthEndHoldingValueQuery
        ->findBySnapshotAndHoldingAssets(
            snapshotId:
                $snapshot->id,

            holdingAssetIds:
                $holdingAssetIds,
        );
```

CSV行ごとに
重複確認SQLを発行しない。

---

#### 8.30.25 事前検証後にトランザクションを開始する

CSV解析や
基本的な業務検証が
完了した後に、

```php
DB::transaction(
    function () {
        // DB更新処理
    },
);
```

を開始する。

CSVファイル解析開始時点から
トランザクションを
保持しない。

---

#### 8.30.26 トランザクション内の再取得

事前検証で使用した
snapshot状態を
そのまま最終判断には使用しない。

トランザクション内で
対象年月のsnapshotを
最新状態で再取得する。

概念例：

```php
$snapshot =
    $this->snapshotRepository
        ->findForUpdateByUserAndTargetYearMonth(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,
        );
```

---

#### 8.30.27 lockForUpdate

既存snapshotが存在する場合は、
必要に応じて
`lockForUpdate()`を使用する。

概念例：

```php
return MonthEndAssetSnapshot::query()
    ->where(
        'user_id',
        $userId,
    )
    ->where(
        'target_year_month',
        $targetYearMonth,
    )
    ->lockForUpdate()
    ->first();
```

これにより、
CSV登録中に
同じsnapshotの
確定処理などが
競合することを抑制する。

---

#### 8.30.28 snapshotの必要時作成

トランザクション内で
snapshotが存在しない場合は、
Repositoryを使用して
新規作成する。

概念例：

```php
$snapshot =
    $this->snapshotRepository
        ->create(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,

            confirmed:
                false,
        );
```

新規作成したsnapshotも
同一トランザクション内で扱う。

---

#### 8.30.29 snapshot重複作成への対応

同一利用者・同一対象年月について、

```text
user_id
+
target_year_month
```

にUNIQUE制約を設定する。

snapshot不存在時は、
対象行そのものがないため
`lockForUpdate()`だけでは
重複作成を完全に防げない。

そのため、
UNIQUE制約を
最終防衛線とする。

競合が発生した場合は、
API共通方針に沿って
再取得または
適切な競合処理を行う。

---

#### 8.30.30 confirmedの最終確認

既存snapshotを
トランザクション内で取得した後、
再度

```text
confirmed
```

を確認する。

概念例：

```php
if (
    $snapshot->confirmed
) {
    throw new
        MonthEndAssetSnapshotAlreadyConfirmedException();
}
```

CSV-005や
事前検証時の状態を
最終判断には使用しない。

---

#### 8.30.31 既存評価額の最終確認

トランザクション内でも、
対象保有商品について
既存の商品別月末評価額を
再確認する。

概念例：

```php
$existingValues =
    $this->monthEndHoldingValueRepository
        ->findExistingForUpdate(
            snapshotId:
                $snapshot->id,

            holdingAssetIds:
                $holdingAssetIds,
        );
```

ただし、
存在しないレコードそのものを
ロックすることはできないため、
最終的には
UNIQUE制約も利用する。

---

#### 8.30.32 UNIQUE制約

`month_end_holding_values`には、
概念的に以下の
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

アプリケーション側の
重複確認と
データベース制約の
二段構えとする。

---

#### 8.30.33 Repository

CSV-006では、
DB更新が存在するため
Repositoryを使用する。

主に以下を担当する。

```text
MonthEndAssetSnapshotRepository
    → snapshot作成
    → 更新ロック付き取得

MonthEndHoldingValueRepository
    → 既存評価額最終確認
    → 商品別月末評価額一括登録
```

Queryは
参照・判定用、
Repositoryは
更新処理用として
責務を分ける。

---

#### 8.30.34 MonthEndHoldingValueRepository

商品別月末評価額の
一括登録を担当する。

概念例：

```php
$this->monthEndHoldingValueRepository
    ->insertMany(
        $rows,
    );
```

登録データは、
概念的に以下とする。

```php
[
    [
        'month_end_asset_snapshot_id'
            => $snapshot->id,

        'holding_asset_id'
            => $holdingAssetId,

        'value'
            => $value,

        'created_at'
            => $now,

        'updated_at'
            => $now,
    ],
]
```

CSVの名称文字列を
そのまま保存しない。

---

#### 8.30.35 一括INSERT

可能な限り、
複数行を
一括INSERTする。

概念例：

```php
MonthEndHoldingValue::query()
    ->insert(
        $insertRows,
    );
```

ただし、
timestamp自動設定など
Eloquentイベントに依存する場合は、
その影響を理解したうえで
実装方式を選択する。

Phase1では、
大量データ向けの
複雑なBatch基盤は導入しない。

---

#### 8.30.36 現在時刻

`created_at`、
`updated_at`を
一括INSERTで設定する場合は、
同一処理内で取得した
同じ現在時刻を使用してよい。

概念例：

```php
$now =
    now();
```

CSV行ごとに
個別に`now()`を呼び出す必要はない。

---

#### 8.30.37 全件成功・全件失敗

CSV-006では、
部分登録を許可しない。

Repositoryで
一部だけ登録された後に
例外が発生した場合でも、
トランザクションによって
全件ロールバックする。

---

#### 8.30.38 UNIQUE制約違反の変換

同時実行などにより、

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

のUNIQUE制約違反が
発生する可能性がある。

この場合、
データベース例外を
そのまま500として返却しない。

可能な限り、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

へ変換し、

```text
409 Conflict
```

として扱う。

PostgreSQLの
制約名やSQLを
レスポンスへ公開しない。

---

#### 8.30.39 snapshot UNIQUE制約違反

snapshot新規作成時に、

```text
user_id
+
target_year_month
```

のUNIQUE制約違反が
発生した場合は、
同時実行で
別処理が先にsnapshotを
作成した可能性がある。

この場合は、
必要に応じて
既存snapshotを再取得し、
その状態を確認する。

ただし、
無制限なリトライ処理は
導入しない。

---

#### 8.30.40 Import Result DTO

正常登録結果は、
専用DTOで表現する。

概念例：

```php
final readonly class MonthEndHoldingValueCsvImportResult
{
    public function __construct(
        public string $targetYearMonth,
        public int $importedCount,
    ) {
    }
}
```

内部IDや
Eloquent Modelを
そのまま返却しない。

---

#### 8.30.41 importedCount

`importedCount`は、
実際に新規登録した

```text
month_end_holding_values
```

の件数とする。

例えば、

```php
$importedCount =
    count(
        $insertRows,
    );
```

とする。

snapshot作成件数は
含めない。

---

#### 8.30.42 API Resource

Import Result DTOを、
専用API Resourceで
レスポンス形式へ変換する。

概念例：

```php
final class MonthEndHoldingValueCsvImportResultResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'targetYearMonth'
                => $this->targetYearMonth,

            'importedCount'
                => $this->importedCount,
        ];
    }
}
```

JSONフィールド名は、
API共通方針に従って
camelCaseとする。

---

#### 8.30.43 Responder

Responderは、
Import Result DTOを
`201 Created`レスポンスへ変換する。

概念例：

```php
final class MonthEndHoldingValueCsvImportResponder
{
    public function created(
        MonthEndHoldingValueCsvImportResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new
                    MonthEndHoldingValueCsvImportResultResource(
                        $result,
                    ),
            ],
            Response::HTTP_CREATED,
        );
    }
}
```

`requestId`などの
共通項目は、
API共通レスポンス処理に従う。

---

#### 8.30.44 Responderで行わないこと

Responderでは、
以下を行わない。

- CSV解析
- CSV検証
- 登録可否判定
- DB検索
- transaction制御
- snapshot作成
- 商品別月末評価額登録
- importedCount再計算

Responderは、
生成済み結果を
HTTPレスポンスへ変換することに
責務を限定する。

---

#### 8.30.45 返却しない情報

API Resourceでは、
以下を返却しない。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`
- `asset_accounts.balance_recording_unit`
- `month_end_asset_snapshots.confirmed`
- `created_at`
- `updated_at`
- CSVファイル内容
- CSV行番号
- 登録した各`value`

成功レスポンスは、

```text
targetYearMonth
importedCount
```

に限定する。

---

#### 8.30.46 エラー変換

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `file`不正 | `VALIDATION_ERROR` |
| CSV解析不能 | `INVALID_CSV_FORMAT` |
| CSVヘッダー不正 | `INVALID_CSV_FORMAT` |
| データ行0件 | `CSV_DATA_REQUIRED` |
| 複数対象年月 | `MULTIPLE_TARGET_YEAR_MONTHS` |
| 資産口座不存在 | `ASSET_ACCOUNT_NOT_FOUND` |
| 残高記録単位不一致 | `BALANCE_RECORDING_UNIT_MISMATCH` |
| 保有商品不存在 | `HOLDING_ASSET_NOT_FOUND` |
| 対象年月時点で無効 | `HOLDING_ASSET_NOT_AVAILABLE` |
| `value`不正 | `INVALID_VALUE` |
| CSV内重複 | `DUPLICATE_HOLDING_ASSET_IN_CSV` |
| snapshot確定済み | `MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED` |
| 既存評価額あり | `MONTH_END_HOLDING_VALUE_ALREADY_EXISTS` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

実際のコード名は、
共通エラーコード定義に合わせる。

---

#### 8.30.47 複数入力エラー

トランザクション開始前の
CSV検証段階では、
可能な範囲で
複数エラーを収集してよい。

この場合は、
API共通エラー形式の

```text
error.details
```

へ格納する。

ただし、
CSV-006では
1件でもエラーが存在すれば
登録処理へ進まない。

---

#### 8.30.48 INTERNAL_SERVER_ERROR

想定外の例外は、
API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ
以下を含めない。

- SQL
- PostgreSQL内部エラー
- UNIQUE制約名
- テーブル名
- カラム名
- PHP内部エラー
- Laravel内部例外メッセージ
- スタックトレース
- サーバーファイルパス

詳細は、
サーバーログへ記録する。

---

#### 8.30.49 ログ

必要に応じて、
以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
targetYearMonth
rowCount
importedCount
errorCode
```

`apiId`は、

```text
CSV-006
```

とする。

CSVファイル全文や
全評価額を
不要にログへ出力しない。

---

#### 8.30.50 キャッシュ

Phase1では、
CSV-006専用の
サーバー側アプリケーションキャッシュを
使用しない。

登録可否は、
最新のDB状態を使用して判断する。

古い

- snapshot
- confirmed
- existing values
- asset account
- holding asset

を使用しない。

---

#### 8.30.51 Idempotency-Key

Phase1では、
`Idempotency-Key`を使用しない。

重複登録は、

- 事前重複確認
- トランザクション
- `lockForUpdate()`
- UNIQUE制約
- フロントエンドの二重送信防止

によって制御する。

---

#### 8.30.52 テスト実装方針

Laravel側では、
Feature Testを中心として
CSV-006のAPI契約と
一括登録フローを確認する。

主に以下を確認する。

- `201 Created`
- `400 Bad Request`
- `404 Not Found`
- `409 Conflict`
- `422 Unprocessable Entity`
- `500 Internal Server Error`
- `file`必須
- ファイル形式
- ファイルサイズ
- CSVヘッダー
- データ行0件
- 対象年月
- 1ファイル1対象年月
- 資産口座存在
- 利用者境界
- 残高記録単位
- 保有商品存在
- 資産口座と保有商品の関連
- 対象年月時点の有効性
- `value`
- 0円
- CSV内重複
- snapshot不存在時の作成
- snapshot未確定
- snapshot確定済み
- 既存商品別月末評価額
- 一部登録されないこと
- rollback
- snapshot重複作成防止
- 商品別月末評価額重複防止
- `targetYearMonth`
- `importedCount`

---

#### 8.30.53 CSV Parser・ValidatorのUnit Test

CSV Parserおよび
CSV Validatorは、
CSV-005と共通であるため、
同じUnit Testを使用する。

以下を重点的に確認する。

```text
ヘッダー
BOM
空行
target_year_month
asset_account_name
holding_asset_name
value
CSV内重複
```

CSV-005用と
CSV-006用で
同じテストケースを
二重管理しない。

---

#### 8.30.54 Import ValidatorのUnit Test

業務Validatorについて、
以下を確認する。

```text
資産口座なし
    → ASSET_ACCOUNT_NOT_FOUND

口座単位
    → BALANCE_RECORDING_UNIT_MISMATCH

保有商品なし
    → HOLDING_ASSET_NOT_FOUND

対象年月時点で無効
    → HOLDING_ASSET_NOT_AVAILABLE

snapshotなし
    → それだけではエラーにしない

snapshot未確定
    → 正常

snapshot確定済み
    → MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED

既存評価額あり
    → MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

CSV-005とCSV-006で
同じValidator結果になることを確認する。

---

#### 8.30.55 RepositoryのDatabase Test

`MonthEndAssetSnapshotRepository`では、
以下を確認する。

- 未確定snapshotを取得できる
- `user_id + target_year_month`で取得できる
- 他利用者のsnapshotを取得しない
- snapshotを`confirmed = false`で作成できる
- UNIQUE制約が維持される

`MonthEndHoldingValueRepository`では、
以下を確認する。

- 複数件を一括登録できる
- 正しいsnapshotへ紐づく
- 正しいholdingAssetへ紐づく
- `value = 0`を保存できる
- UNIQUE制約が維持される

---

#### 8.30.56 UseCaseのUnit Test

UseCaseでは、
Parser、
Validator、
Query、
Repositoryを組み合わせて
正しい登録結果になることを確認する。

概念的には、

```text
Parsed CSV
+
CSV Validation
+
Business Validation
+
Queries
    ↓
ImportMonthEndHoldingValueCsvUseCase
    ↓
Transaction
    ↓
Repositories
    ↓
Import Result DTO
```

を確認する。

特に、

```text
snapshotなし
    ↓
snapshot作成
    ↓
holding values登録
```

```text
snapshotあり未確定
    ↓
既存snapshot使用
```

```text
snapshot確定済み
    ↓
登録しない
```

```text
既存評価額あり
    ↓
登録しない
```

を確認する。

---

#### 8.30.57 トランザクションテスト

意図的に
商品別月末評価額登録途中で
例外を発生させ、

```text
ROLLBACK
```

されることを確認する。

特に、
CSV-006内で
snapshotを新規作成した場合は、
snapshotも
ロールバックされることを確認する。

---

#### 8.30.58 並行実行テスト

可能であれば、
Database Testまたは
Integration Testで
並行実行を確認する。

主な観点は、
以下とする。

```text
同一利用者
+
同一対象年月
+
snapshot不存在
```

で
複数登録が走っても、
snapshotが
重複作成されないこと。

また、

```text
同一snapshot
+
同一holding_asset
```

について
複数登録が走っても、
商品別月末評価額が
重複登録されないことを確認する。

---

#### 8.30.59 CSV-005との整合性テスト

同一利用者、
同一DB状態、
同一CSVで、

```text
CSV-005
canImport = true
```

となった場合に、
DB状態を変更せず
CSV-006を実行すると
登録成功することを確認する。

逆に、
CSV-005後に

```text
confirmed = true
```

へ変更した場合は、
CSV-006が

```text
409 Conflict
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

となることを確認する。

これにより、

```text
CSV-005
    → 事前確認

CSV-006
    → 最新状態で再検証
```

という責務分離を保証する。

---



---


































