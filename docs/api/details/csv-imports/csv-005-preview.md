##  CSV-005 商品別月末評価額CSVプレビュー

### 1 概要

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

### 2 ユースケース

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

### 3 エンドポイント

```http
POST /api/v1/month-end-holding-values/imports/preview
```

---

### 4 HTTPメソッド

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

### 5 認証・利用者の扱い

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

#### 5.1 X-User-Id

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

#### 5.2 利用者存在確認

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

#### 5.3 資産口座の利用者境界

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

#### 5.4 保有商品の利用者境界

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

#### 5.5 同名保有商品の扱い

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

#### 5.6 他利用者の同名保有商品

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

#### 5.7 残高記録単位

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

#### 5.8 保有商品と資産口座の関連

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

#### 5.9 月末資産状況の利用者境界

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

#### 5.10 月末資産状況が存在しない場合

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

#### 5.11 確定済み月末資産状況

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

#### 5.12 既存商品別月末評価額の利用者境界

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

#### 5.13 他利用者の既存評価額

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

#### 5.14 CSV-004の利用者コンテキストを引き継がない

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

#### 5.15 CSV-006への利用者コンテキスト引き継ぎを前提としない

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

#### 5.16 1リクエスト1利用者

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

#### 5.17 X-User-Id未指定

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

として扱う。

CSVファイルの解析へ
進まない。

---

#### 5.18 X-User-Id形式不正

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

#### 5.19 利用者不存在

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

#### 5.20 プレビューでは業務データを更新しない

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

### 6 パスパラメータ

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

### 7 クエリパラメータ

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

### 8 リクエストヘッダー

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

### 9 リクエストボディ

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

### 10 CSVファイル仕様

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

### 11 バリデーション

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

#### 11.1 file 必須

`file`は、
必須とする。

CSVファイルが
指定されていない場合は、
バリデーションエラーとする。

この場合、
CSV解析処理や
業務データ検索へ進まない。

---

#### 11.2 アップロードファイルであること

`file`は、
HTTPアップロードファイルとして
正常に受信できていることを確認する。

文字列や
通常のフォーム値を
CSVファイルとして扱わない。

---

#### 11.3 ファイル拡張子

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

#### 11.4 ファイルサイズ

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

#### 11.5 空ファイル

CSVファイルが
0バイトの場合、
または有効なヘッダー行を
取得できない場合は、
不正なCSVとして扱う。

プレビュー結果を
生成できるCSVではないため、
後続の業務検証へ進まない。

---

#### 11.6 文字コード

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

#### 11.7 CSVとして読み取り可能であること

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

#### 11.8 ヘッダー必須

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

#### 11.9 ヘッダー名

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

#### 11.10 ヘッダー順序

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

#### 11.11 余分なヘッダー

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

#### 11.12 データ行0件

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

#### 11.13 空行

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

#### 11.14 target_year_month 必須

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

#### 11.15 target_year_month 形式

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

#### 11.16 1ファイル1対象年月

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

#### 11.17 asset_account_name 必須

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

#### 11.18 asset_account_nameの扱い

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

#### 11.19 資産口座存在確認

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

#### 11.20 他利用者の同名資産口座

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

#### 11.21 残高記録単位

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

#### 11.22 holding_asset_name 必須

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

#### 11.23 holding_asset_nameの扱い

`holding_asset_name`は、
同一行で指定された
資産口座内の保有商品を
特定するために使用する。

保有商品名だけを
システム全体から検索して
対象商品を決定しない。

---

#### 11.24 保有商品存在確認

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

#### 11.25 他資産口座にのみ同名保有商品が存在する

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

#### 11.26 他利用者にのみ同名保有商品が存在する

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

#### 11.27 対象年月時点での保有商品の有効性

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

#### 11.28 value 必須

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

#### 11.29 value 数値形式

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

#### 11.30 value 整数

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

#### 11.31 value 0円以上

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

#### 11.32 CSV内重複

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

#### 11.33 CSV内重複時の扱い

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

#### 11.34 月末資産状況の確認

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

#### 11.35 月末資産状況が存在しない場合

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

#### 11.36 確定済み月末資産状況

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

#### 11.37 既存商品別月末評価額

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

#### 11.38 他利用者の既存商品別月末評価額

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

#### 11.39 複数エラーの収集

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

#### 11.40 canImport

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

#### 11.41 プレビュー時は登録しない

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

#### 11.42 CSV-006で再検証する

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

### 12 業務ルール

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

#### 12.1 CSV-004と同じCSV仕様を使用する

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

#### 12.2 CSV-006と同じ登録可否ルールを使用する

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

#### 12.3 1ファイル1対象年月

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

#### 12.4 資産口座の特定

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

#### 12.5 保有商品の特定

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

#### 12.6 利用者境界

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

#### 12.7 資産口座の残高記録単位

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

#### 12.8 保有商品は指定資産口座に属すること

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

#### 12.9 対象年月時点での保有商品

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

#### 12.10 value

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

#### 12.11 0円

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

#### 12.12 CSV内重複

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

#### 12.13 CSV内重複を自動解決しない

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

#### 12.14 月末資産状況

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

#### 12.15 月末資産状況が存在しない場合

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

#### 12.16 既存の未確定月末資産状況

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

#### 12.17 確定済み月末資産状況

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

#### 12.18 既存商品別月末評価額

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

#### 12.19 既存値と同じ場合も重複とする

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

#### 12.20 既存値と異なる場合も上書きしない

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

#### 12.21 データ行0件

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

#### 12.22 複数エラーを収集する

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

#### 12.23 CSV全体エラー

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

#### 12.24 行エラー

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

#### 12.25 canImport

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

#### 12.26 正常行だけを登録対象としない

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

#### 12.27 プレビュー結果を保存しない

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

#### 12.28 CSVファイルを保存しない

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

#### 12.29 CSV-005では副作用を発生させない

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

#### 12.30 CSV-006で再検証する

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

### 13 処理フロー

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

#### 13.1 リクエスト検証

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

#### 13.2 CSV解析

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

#### 13.3 ヘッダー検証

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

#### 13.4 行入力値検証

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

#### 13.5 対象年月特定

正常に取得できた
`target_year_month`から
CSV全体の対象年月を特定する。

単一年月に
特定できない場合は、
CSV全体エラーを生成する。

---

#### 13.6 資産口座取得

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

#### 13.7 保有商品取得

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

#### 13.8 月末資産状況取得

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

#### 13.9 既存商品別月末評価額取得

snapshotが存在する場合は、
CSVで対象となる
保有商品について、
既存商品別月末評価額を
まとめて取得する。

既存データがある行は、
行エラーとする。

---

#### 13.10 エラー集約

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

#### 13.11 canImport判定

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

#### 13.12 Preview DTO生成

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

### 14 正常レスポンス

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

#### 14.1 登録可能な場合

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

#### 14.2 登録不可の場合

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

#### 14.3 CSV全体エラーがある場合

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

#### 14.4 データ行0件の場合

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

#### 14.5 対象年月を特定できない場合

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

#### 14.6 HTTPエラーとなる場合

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

#### 14.7 レスポンス順序

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

### 15 レスポンス項目

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

#### 15.1 targetYearMonth

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

#### 15.2 canImport

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

#### 15.3 errors

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

#### 15.4 rows

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

#### 15.5 rowNumber

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

#### 15.6 assetAccountName

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

#### 15.7 holdingAssetName

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

#### 15.8 value

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

#### 15.9 行エラー

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

#### 15.10 CSV全体エラーと行エラーを分離する

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

#### 15.11 内部IDを返却しない

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

#### 15.12 返却しない情報

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

### 16 エラーレスポンス

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

#### 16.1 HTTPエラー一覧

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

#### 16.2 プレビュー内で扱うエラー

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

#### 16.3 USER_CONTEXT_REQUIRED

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

#### 16.4 INVALID_USER_ID

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

#### 16.5 USER_NOT_FOUND

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

#### 16.6 file未指定

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

#### 16.7 ファイル形式不正

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

#### 16.8 ファイルサイズ超過

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

#### 16.9 空ファイル

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

#### 16.10 CSV解析不能

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

#### 16.11 CSVヘッダー不正

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

#### 16.12 CSV_DATA_REQUIRED

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

#### 16.13 MULTIPLE_TARGET_YEAR_MONTHS

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

#### 16.14 ASSET_ACCOUNT_NOT_FOUND

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

#### 16.15 BALANCE_RECORDING_UNIT_MISMATCH

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

#### 16.16 HOLDING_ASSET_NOT_FOUND

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

#### 16.17 HOLDING_ASSET_NOT_AVAILABLE

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

#### 16.18 INVALID_VALUE

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

#### 16.19 DUPLICATE_HOLDING_ASSET_IN_CSV

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

#### 16.20 MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED

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

#### 16.21 MONTH_END_HOLDING_VALUE_ALREADY_EXISTS

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

#### 16.22 複数エラー

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

#### 16.23 INTERNAL_SERVER_ERROR

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

### 17 HTTPステータス

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

#### 17.1 200 OK

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

#### 17.2 400 Bad Request

以下の場合に使用する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
```

CSV内容に関する
入力不正には使用しない。

---

#### 17.3 404 Not Found

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

#### 17.4 422 Unprocessable Entity

プレビュー処理を成立させられない
入力・構造不正に使用する。

主に以下を対象とする。

- `file`未指定
- ファイル形式不正
- ファイルサイズ超過
- CSV解析不能
- CSVヘッダー不正

---

#### 17.5 500 Internal Server Error

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

### 18 副作用

CSV-005には、
業務データに対する
副作用はない。

本APIでは、
CSV内容の解析・検証のみを行う。

---

#### 18.1 更新しないテーブル

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

#### 18.2 snapshotを作成しない

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

#### 18.3 商品別月末評価額を作成しない

`canImport = true`の場合でも、

```text
month_end_holding_values
```

へデータを登録しない。

CSV-005は、
登録前確認に責務を限定する。

---

#### 18.4 CSVファイルを永続保存しない

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

#### 18.5 プレビュー結果を保存しない

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

### 19 トランザクション

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

#### 19.1 複数SELECT

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

### 20 ロック

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

#### 20.1 CSV-005とCSV-006の間をロックしない

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

### 21 キャッシュ

Phase1では、
CSV-005専用の
サーバー側アプリケーションキャッシュを
使用しない。

同じCSVファイルでも、
業務データの状態が変われば
プレビュー結果も変化するためである。

---

#### 21.1 キャッシュしない対象

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

#### 21.2 同一CSVの再プレビュー

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

### 22 冪等性

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

#### 22.1 同一CSVの複数回実行

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

#### 22.2 業務状態が変わった場合

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

#### 22.3 Idempotency-Key

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

#### 22.4 二重送信

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

#### 22.5 CSV-006との違い

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

### 23 関連テーブル

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

#### 23.1 users

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

#### 23.2 asset_accounts

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

#### 23.3 holding_assets

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

#### 23.4 asset_account_available_settings

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

#### 23.5 month_end_asset_snapshots

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

#### 23.6 month_end_holding_values

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

### 24 Query方針

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

#### 24.1 N+1を発生させない

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

#### 24.2 資産口座の一括取得

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

#### 24.3 保有商品の一括取得

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

#### 24.4 利用可能資産設定の取得

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

#### 24.5 snapshotの取得

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

#### 24.6 既存商品別月末評価額の一括取得

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

#### 24.7 不要なQueryを実行しない

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

### 25 レスポンス変換方針

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

#### 25.1 Preview DTO

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

#### 25.2 CSV入力名とAPIレスポンス名

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

#### 25.3 エラー変換

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

#### 25.4 行順序

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

### 26 ログ・監視

CSV-005では、
通常のプレビュー成功時に
CSV内容全体を
アプリケーションログへ出力しない。

商品別月末評価額は
利用者の資産情報であるため、
必要以上にログへ残さない。

---

#### 26.1 通常ログ

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

#### 26.2 エラーログ

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

#### 26.3 ログへ出力しない情報

原則として、
以下をログへ直接出力しない。

- CSVファイル全文
- 全行の`value`
- 資産情報一覧
- multipartリクエストボディ全体

障害調査で必要な場合でも、
最小限の情報に限定する。

---

#### 26.4 requestId

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

### 27 セキュリティ

CSV-005では、
アップロードファイルを扱うため、
通常のJSON APIに加えて
ファイル入力を前提とした
防御を行う。

---

#### 27.1 利用者境界

すべての業務データ検索で、
`X-User-Id`から特定した
操作対象利用者との
利用者境界を保証する。

CSV内の名称だけを使用して、
他利用者のデータへ
アクセスしない。

---

#### 27.2 内部IDを信用しない

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

#### 27.3 ファイルサイズ制限

CSV共通仕様で定める
最大ファイルサイズを適用する。

過大なCSVファイルによって
メモリやCPUを
過剰消費しないようにする。

---

#### 27.4 CSVを実行しない

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

#### 27.5 SQLインジェクション対策

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

#### 27.6 CSV内容のレスポンス反映

CSVに含まれる名称を
レスポンスへ返却する場合は、
JSON文字列として
安全にエンコードする。

フロントエンドでは、
レスポンス値を
`dangerouslySetInnerHTML`などで
直接HTMLとして描画しない。

---

#### 27.7 一時ファイル

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

#### 27.8 エラー情報

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

### 28 性能・スケーラビリティ

CSV-005では、
CSV行数に比例して
SQL発行回数が増加しないようにする。

Phase1では、
大規模CSV処理基盤を
構築することよりも、
一括取得による
単純なN+1回避を優先する。

---

#### 28.1 SQL発行回数

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

#### 28.2 CSV行数

CSV共通仕様で
現実的な最大行数を設定する場合は、
CSV-005にも
同じ上限を適用する。

CSV-005とCSV-006で
異なる行数制限を
持たせない。

---

#### 28.3 非同期処理

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

#### 28.4 キャッシュ

プレビュー結果は、
最新の業務状態に依存する。

そのため、
プレビュー結果や
`canImport`を
キャッシュしない。

---

### 29 設計上の補足

#### 29.1 POSTを採用する理由

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

#### 29.2 プレビュー専用APIを分ける理由

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

#### 29.3 canImportを返却する理由

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

#### 29.4 業務エラーを200 OKで返す理由

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

#### 29.5 CSV構造不正を422とする理由

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

#### 29.6 複数エラーを返却する理由

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

#### 29.7 rowNumberを返却する理由

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

#### 29.8 資産口座名・保有商品名を返却する理由

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

#### 29.9 内部IDを返却しない理由

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

#### 29.10 balance_recording_unitを返却しない理由

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

#### 29.11 confirmedを返却しない理由

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

#### 29.12 プレビュー結果を保存しない理由

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

#### 29.13 CSV-006で再検証する理由

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

#### 29.14 ロックを保持しない理由

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

#### 29.15 React側で業務検証を再実装しない理由

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

#### 29.16 0円とnullを区別する理由

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

#### 29.17 CSV-004との仕様共通化

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

#### 29.18 CSV-006との検証ロジック共通化

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

#### 29.19 Idempotency-Keyを使用しない理由

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

#### 29.20 キャッシュしない理由

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

### 30 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [エラーコード一覧](../../error-codes.md)
- [CSV-004 商品別月末評価額CSVテンプレート取得](./csv-004-template.md)
- [CSV-006 商品別月末評価額CSV登録](./csv-006-create.md)
- [CSV-001 月末資産残高CSVテンプレート取得](./csv-001-create.md)
- [CSV-002 月末資産残高CSVプレビュー](./csv-002-preview.md)
- [CSV-003 月末資産残高CSV登録](./csv-003-create.md)
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