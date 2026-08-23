##  CSV-006 商品別月末評価額CSV登録

### 1 概要

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

### 2 ユースケース

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

### 3 エンドポイント

```http
POST /api/v1/month-end-holding-values/imports
```

---

### 4 HTTPメソッド

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

### 5 認証・利用者の扱い

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

#### 5.1 資産口座の利用者境界

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

#### 5.2 保有商品の利用者境界

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

#### 5.3 同名保有商品の扱い

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

#### 5.4 他利用者の同名保有商品

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

#### 5.5 残高記録単位

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

#### 5.6 保有商品と資産口座の関連

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

#### 5.7 月末資産状況の利用者境界

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

#### 5.8 月末資産状況が存在しない場合

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

#### 5.9 既存商品別月末評価額の利用者境界

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

#### 5.10 他利用者の既存商品別月末評価額

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

#### 5.11 登録先保有商品の特定

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

#### 5.12 登録する商品別月末評価額の利用者境界

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

#### 5.13 他利用者データを登録可否判定へ利用しない

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

#### 5.14 X-User-Idのエラー

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

#### 5.15 CSV-005の利用者コンテキストを引き継がない

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

#### 5.16 利用者境界確認後に登録する

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

#### 5.17 1リクエスト1利用者

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

### 6 パスパラメータ

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
商品別月末評価額を
新規一括登録するAPIである。

上書き可否や
強制登録などを
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

### 9 リクエストボディ

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

#### 11.1 file 必須

`file`は、
必須とする。

CSVファイルが
指定されていない場合は、
バリデーションエラーとする。

この場合、
CSV解析処理や
データベース登録へ進まない。

---

#### 11.2 アップロードファイルであること

`file`は、
HTTPアップロードファイルとして
正常に受信できていることを確認する。

通常の文字列や
フォーム値を
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

ただし、
拡張子だけを
唯一の判定根拠とはせず、
実際にCSVとして
正常に読み取れることも確認する。

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

この場合、
登録処理へ進まない。

---

#### 11.6 文字コード

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

#### 11.7 CSVとして読み取り可能であること

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

#### 11.12 データ行の存在

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

#### 11.13 空行

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
登録不可とする。

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
有効な値であることも確認する。

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

複数対象年月が
混在する場合は、
CSV全体を登録しない。

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
登録不可とする。

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
CSV全体を登録不可とする。

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
登録エラーとする。

---

#### 11.21 残高記録単位

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
登録不可とする。

---

#### 11.23 holding_asset_nameの扱い

`holding_asset_name`は、
同一行で指定された
資産口座内の
保有商品を特定するために使用する。

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
CSV全体を登録不可とする。

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

```text
HOLDING_ASSET_NOT_FOUND
```

相当の
登録エラーとする。

---

#### 11.26 他利用者にのみ同名保有商品が存在する

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

#### 11.27 対象年月時点での保有商品の有効性

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
登録不可とする。

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

で
重複確認してよい。

---

#### 11.33 CSV内重複時の扱い

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

#### 11.36 確定済み月末資産状況

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

Phase1では、
CSV-006を
新規一括登録APIとして扱う。

そのため、
既存データが存在する場合は
重複エラーとする。

既存の`value`を
CSV値で上書きしない。

---

#### 11.38 overwriteを許可しない

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

#### 11.39 他利用者の既存商品別月末評価額

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

#### 11.40 複数エラー

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

#### 11.41 一部登録を行わない

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

#### 11.42 登録前に全件検証する

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

#### 11.43 CSV-005と同じ検証ロジックを使用する

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

#### 11.44 CSV-005の結果は信頼しない

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

#### 11.45 登録直前の再確認

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

#### 11.46 データベース制約も最終防衛線とする

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

#### 11.47 バリデーション失敗時はDB更新しない

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

### 12 業務ルール

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

#### 12.1 CSV-004・CSV-005と同じCSV仕様を使用する

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

#### 12.2 CSV-005と同じ登録可否ルールを使用する

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

#### 12.3 1ファイル1対象年月

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

#### 12.6 利用者境界

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

#### 12.7 残高記録単位

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

#### 12.8 保有商品は指定資産口座に属すること

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

#### 12.9 対象年月時点の保有商品

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

#### 12.11 0円を有効値とする

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

で一意であることを
必須とする。

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

について、

- 先勝ち
- 後勝ち
- 評価額合算
- 平均値算出

などの
自動解決を行わない。

---

#### 12.14 月末資産状況の取得

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

#### 12.15 月末資産状況が存在しない場合

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

#### 12.16 既存の未確定月末資産状況

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

#### 12.17 確定済み月末資産状況

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

#### 12.18 既存商品別月末評価額

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

#### 12.19 既存データを上書きしない

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

#### 12.20 既存値と同じ場合も登録しない

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

#### 12.21 全件成功または全件失敗

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

#### 12.22 全件検証後に登録する

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

#### 12.23 登録トランザクション

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

#### 12.24 snapshotだけを残さない

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

#### 12.25 商品別月末評価額の登録

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

#### 12.26 CSV行番号は保存しない

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

#### 12.27 CSVファイルを保存しない

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

#### 12.28 プレビュー結果を保存・参照しない

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

#### 12.29 canImportを登録権利としない

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

#### 12.30 同時実行

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

#### 12.31 データベース制約

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

#### 12.32 snapshotの重複作成を防止する

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

#### 12.33 登録後も未確定とする

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

#### 12.34 未登録商品が存在しても登録可能とする

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

#### 12.35 月末資産残高を登録しない

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

#### 12.36 登録件数

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

### 13 処理フロー

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

#### 13.1 利用者コンテキスト確認

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

#### 13.2 CSV解析

CSVファイルを解析し、

- ヘッダー
- データ行
- CSV行番号

を取得する。

CSVとして
正常に解析できない場合は、
登録処理へ進まない。

---

#### 13.3 CSV入力値検証

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

#### 13.4 対象年月特定

CSV全体から、
単一の対象年月を特定する。

複数の対象年月が
存在する場合は、
CSV全体を登録しない。

---

#### 13.5 資産口座・保有商品の特定

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

#### 13.6 業務ルール検証

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

#### 13.7 トランザクション開始

全件登録可能であることを
事前確認した後に、
登録トランザクションを開始する。

CSV解析処理中から
トランザクションを保持しない。

---

#### 13.8 snapshotの再取得

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

#### 13.9 snapshotの必要時作成

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

#### 13.10 確定状態の最終確認

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

#### 13.11 既存商品別月末評価額の最終確認

登録トランザクション内でも、
対象保有商品について
既存の商品別月末評価額が
存在しないことを再確認する。

事前検証後に
別リクエストが
先に登録している可能性を考慮する。

---

#### 13.12 一括登録

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

#### 13.13 COMMIT

すべての商品別月末評価額を
正常に登録できた場合は、
トランザクションを
COMMITする。

その後、
登録結果を返却する。

---

#### 13.14 ROLLBACK

トランザクション内で
1件でも登録に失敗した場合は、
すべてROLLBACKする。

以下の状態を
発生させない。

- snapshotだけ新規作成済み
- CSV前半の商品だけ登録済み
- 一部保有商品の評価額だけ登録済み

---

### 14 正常レスポンス

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

#### 14.1 201 Created

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

#### 14.2 snapshotを新規作成した場合

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

#### 14.3 既存の未確定snapshotを使用した場合

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

#### 14.4 登録済み商品一覧を返却しない

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

#### 14.5 エラー時

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

#### 14.6 登録途中のエラー

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

### 15 レスポンス項目

正常時の
`data`配下には、
以下の項目を返却する。

| 項目 | 型 | NULL | 内容 |
|---|---|:---:|---|
| `targetYearMonth` | string | × | 登録対象となった対象年月 |
| `importedCount` | integer | × | 新規登録した商品別月末評価額件数 |

---

#### 15.1 targetYearMonth

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

#### 15.2 importedCount

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

#### 15.3 importedCountは0にならない

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

#### 15.4 返却しない情報

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

#### 15.5 snapshotIdを返却しない

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

#### 15.6 assetAccountIdを返却しない

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

#### 15.7 holdingAssetIdを返却しない

CSV-006では、
複数の保有商品を
一括登録する。

登録した保有商品ID一覧を
成功レスポンスへ含めない。

登録後に
商品別月末評価額の詳細が必要な場合は、
参照APIから取得する。

---

#### 15.8 confirmedを返却しない

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

#### 15.9 登録したvalue一覧を返却しない

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

#### 15.10 レスポンス例

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

### 16 エラーレスポンス

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

#### 16.1 エラー一覧

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

#### 16.2 USER_CONTEXT_REQUIRED

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

CSV解析および
登録処理へ進まない。

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
0
-1
abc
1.5
```

---

#### 16.4 USER_NOT_FOUND

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

#### 16.5 file未指定

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

#### 16.6 ファイル形式不正

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

#### 16.7 ファイルサイズ超過

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

#### 16.8 空ファイル

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

#### 16.9 CSV解析不能

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

#### 16.10 CSVヘッダー不正

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

#### 16.11 データ行0件

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

#### 16.12 対象年月混在

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

#### 16.13 行単位の入力エラー

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

#### 16.14 資産口座不存在

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

#### 16.15 残高記録単位不一致

指定された資産口座が
口座単位で管理されている場合は、

```text
BALANCE_RECORDING_UNIT_MISMATCH
```

として登録を拒否する。

口座単位の月末残高は、
CSV-003 月末資産残高CSV登録で扱う。

---

#### 16.16 保有商品不存在

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

#### 16.17 対象年月時点で保有商品が無効

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

#### 16.18 商品別月末評価額不正

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

#### 16.19 CSV内重複

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

#### 16.20 確定済み月末資産状況

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

#### 16.21 既存商品別月末評価額

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

#### 16.22 同時実行によるUNIQUE制約違反

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

#### 16.23 複数エラーの扱い

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

#### 16.24 エラー詳細

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

#### 16.25 INTERNAL_SERVER_ERROR

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

### 17 HTTPステータス

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

#### 17.1 201 Created

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

#### 17.2 400 Bad Request

以下の場合に使用する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
```

CSV内容の
入力エラーには使用しない。

---

#### 17.3 404 Not Found

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

#### 17.4 409 Conflict

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

#### 17.5 422 Unprocessable Entity

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

#### 17.6 500 Internal Server Error

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

### 18 副作用

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

#### 18.1 正常終了時の更新

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

#### 18.2 更新しないデータ

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

#### 18.3 confirmedを変更しない

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

#### 18.4 エラー時の副作用

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

### 19 トランザクション

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

#### 19.1 同一トランザクションで行う処理

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

#### 19.2 ロールバック

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

#### 19.3 CSV検証処理をトランザクションへ入れすぎない

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

### 20 ロック

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

#### 20.1 snapshot不存在時の競合

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

#### 20.2 商品別月末評価額の重複防止

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

#### 20.3 過剰なロックを行わない

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

### 21 キャッシュ

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

#### 21.1 登録後のキャッシュ

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

### 22 冪等性

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

#### 22.1 同一CSVの再実行

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

#### 22.2 値が異なる場合も上書きしない

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

#### 22.3 一部だけ既存の場合

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

#### 22.4 Idempotency-Key

Phase1では、
`Idempotency-Key`を採用しない。

意図しない二重登録は、

- アプリケーション側の重複確認
- トランザクション
- データベースUNIQUE制約
- フロントエンドの二重送信防止

によって制御する。

---

#### 22.5 二重送信

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

#### 22.6 通信失敗後の再送

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

#### 22.7 同時実行

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

### 23 関連テーブル

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

#### 23.1 users

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

#### 23.2 asset_accounts

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

#### 23.3 残高記録単位

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

#### 23.4 holding_assets

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

#### 23.5 同名保有商品の識別

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

#### 23.6 asset_account_available_settings

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

#### 23.7 month_end_asset_snapshots

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

#### 23.8 月末資産状況が存在する場合

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

#### 23.9 月末資産状況が存在しない場合

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

#### 23.10 month_end_asset_snapshotsの一意性

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

#### 23.11 month_end_holding_values

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

#### 23.12 既存商品別月末評価額の確認

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

#### 23.13 month_end_holding_valuesの一意性

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

#### 23.14 更新対象テーブル

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

#### 23.15 更新しないテーブル

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

#### 23.16 month_end_asset_balances

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

### 24 関連する機能要件

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

### 25 インデックス

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

#### 25.1 asset_accounts

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

#### 25.2 holding_assets

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

#### 25.3 month_end_asset_snapshots

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

#### 25.4 month_end_holding_values

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

### 26 性能

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

#### 26.1 N+1問題

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

#### 26.2 資産口座一括取得

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

#### 26.3 保有商品一括取得

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

#### 26.4 既存商品別月末評価額の一括取得

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

#### 26.5 一括INSERT

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

#### 26.6 トランザクション時間

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

#### 26.7 非同期処理

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

### 27 セキュリティ

CSV-006では、
アップロードされたCSVをもとに
業務データを登録するため、
利用者境界と
入力値の信頼境界を
明確にする。

---

#### 27.1 利用者境界

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

#### 27.2 内部IDをCSVから受け取らない

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

#### 27.3 SQLインジェクション対策

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

#### 27.4 ファイルサイズ制限

過大なCSVによって
サーバー資源が
過剰消費されないよう、
CSV共通仕様の
ファイルサイズ上限を適用する。

必要に応じて
CSV行数上限も
共通仕様として設定する。

---

#### 27.5 CSVを実行しない

CSV内容は、
データとしてのみ扱う。

以下として
評価・実行しない。

- PHPコード
- SQL
- シェルコマンド
- テンプレートコード

---

#### 27.6 ファイルを永続保存しない

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

#### 27.7 エラー情報

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

### 28 ログ・監視

CSV-006では、
API共通ログ方針に従って
登録処理の結果を記録する。

商品別月末評価額は
利用者の資産情報であるため、
CSV内容全体を
ログへ出力しない。

---

#### 28.1 ログコンテキスト

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

#### 28.2 正常時

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

#### 28.3 異常時

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

#### 28.4 ログへ出力しない情報

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

### 29 設計上の補足

#### 29.1 POSTを採用する理由

CSV-006は、
CSVファイルの内容に基づき、
複数の商品別月末評価額を
新規登録する。

そのため、
HTTPメソッドには
`POST`を採用する。

本APIは、
単純な参照処理ではなく、

```text
CSV受信
    ↓
CSV解析
    ↓
入力・業務検証
    ↓
商品別月末評価額登録
```

という副作用を持つ処理である。

---

#### 29.2 CSV-005とCSV-006を分離する理由

CSVプレビューと
CSV登録では、
副作用の有無が異なる。

```text
CSV-005
    ↓
解析・検証
    ↓
プレビュー生成
    ↓
DB更新なし
```

```text
CSV-006
    ↓
解析・再検証
    ↓
商品別月末評価額登録
    ↓
DB更新あり
```

同一APIに

```text
preview = true
```

などを指定して
処理を切り替える方式は採用しない。

プレビューと登録を
別APIとして明確に分離することで、

* 副作用の有無
* レスポンス形式
* トランザクション
* 排他制御
* エラー処理

の責務を明確にする。

---

#### 29.3 CSV-006で再検証する理由

CSV-005とCSV-006の間で、
業務データの状態が
変更される可能性がある。

例えば、

```text
CSV-005

snapshot
confirmed = false

既存評価額なし
    ↓
canImport = true
```

となった後に、
別処理によって、

```text
snapshot確定

または

商品別月末評価額登録
```

が行われる可能性がある。

そのため、

```text
CSV-005
canImport = true
```

であっても、
CSV-006では

* CSV内容
* 資産口座
* 残高記録単位
* 保有商品
* 資産口座と保有商品の関連
* 対象年月時点での保有商品の有効性
* 月末資産状況
* 既存商品別月末評価額

を最新状態で再検証する。

CSV-005の結果を
登録可否の最終保証として
使用しない。

---

#### 29.4 プレビュー結果を送信しない理由

CSV-006では、
CSV-005で返却された

```text
canImport
errors
rows
targetYearMonth
previewId
```

などを
登録リクエストとして受け取らない。

クライアント側で保持された
プレビュー結果は、
サーバーが保証できる情報ではないためである。

また、
プレビュー後に
データベース状態が
変更される可能性もある。

CSV-006では、

```text
CSVファイル
+
X-User-Id
+
CSV-006実行時点の最新DB状態
```

を基準として
登録可否を判定する。

---

#### 29.5 CSVファイルを再送する理由

Phase1では、
CSV-005のプレビュー結果や
アップロードされたCSVファイルを
登録用データとして
サーバー側へ保持しない。

そのため、
CSV-006では
CSV-005で使用した
同一CSVファイルを再送する。

これにより、

* 一時CSVファイル保存
* `previewId`
* preview token
* preview session
* 一時データの有効期限
* 一時データの削除処理
* プレビューと利用者の関連管理

などを不要とする。

Phase1では、
プレビューセッション管理を導入せず、
APIをステートレスに近い形で扱う。

---

#### 29.6 資産口座名と保有商品名を使用する理由

CSVでは、

```text
asset_account_id
holding_asset_id
```

のような
内部IDを入力させない。

利用者がCSV上で扱う値は、

```text
asset_account_name
holding_asset_name
```

とする。

内部IDは
CSV-006実行時に
バックエンド側で特定する。

概念的には、

```text
X-User-Id
+
asset_account_name
    ↓
asset_account特定
    ↓
asset_account.id
+
holding_asset_name
    ↓
holding_asset特定
```

とする。

これにより、
データベース内部IDを
CSV仕様へ露出させない。

---

#### 29.7 保有商品名だけで特定しない理由

異なる資産口座に、
同名の保有商品が
存在する可能性がある。

例えば、

```text
証券口座A
    全世界株式

証券口座B
    全世界株式
```

という状態があり得る。

そのため、
CSV-006では

```text
holding_asset_name
```

だけを使用して
保有商品を特定しない。

登録対象は、

```text
操作対象利用者
+
asset_account_name
+
holding_asset_name
```

によって特定する。

これにより、
別資産口座に属する
同名保有商品へ
誤って評価額を登録することを防止する。

---

#### 29.8 商品単位の資産口座だけを対象とする理由

Life Plannerでは、
資産口座ごとに
残高の記録単位が異なる。

概念的には、

```text
口座単位
    ↓
month_end_asset_balances
```

```text
商品単位
    ↓
month_end_holding_values
```

とする。

CSV-006は、
`month_end_holding_values`を
登録するAPIである。

そのため、
対象資産口座の

```text
balance_recording_unit
```

が商品単位であることを
必須とする。

口座単位の資産口座へ
商品別月末評価額を
登録してはならない。

---

#### 29.9 対象年月時点の保有商品状態を使用する理由

商品別月末評価額は、
現在時点の保有状態ではなく、
対象年月時点の
資産状態を記録するデータである。

そのため、
保有商品の有効性についても、

```text
現在有効か
```

ではなく、

```text
target_year_month時点で
評価額記録対象として有効か
```

を基準として判定する。

例えば、
現在は利用されている保有商品でも、
対象年月時点では
まだ存在していない場合は、
その対象年月の評価額として
登録可能とは判定しない。

---

#### 29.10 全件成功・全件失敗とする理由

CSVは、
複数の商品別月末評価額を
まとめて登録するための入力手段である。

一部の行だけを登録すると、
利用者から見て

```text
どの商品まで登録されたのか
```

が分かりにくくなる。

例えば、

```text
2行目
正常

3行目
正常

4行目
エラー
```

の場合に、

```text
2行目 登録済み
3行目 登録済み
4行目 未登録
```

とはしない。

CSV-006では、

```text
全件成功
または
全件失敗
```

とする。

---

#### 29.11 登録前に全件検証する理由

CSVを読み込みながら、
1行ずつ

```text
検証
    ↓
INSERT
```

する方式は採用しない。

途中の行で
エラーが発生した場合に、
前半だけ登録される可能性があるためである。

基本的には、

```text
CSV全体解析
    ↓
入力値検証
    ↓
業務ルール検証
    ↓
全件登録可能
    ↓
トランザクション開始
    ↓
一括登録
```

とする。

---

#### 29.12 トランザクションを使用する理由

CSV-006では、

* 必要に応じた月末資産状況の作成
* 複数の商品別月末評価額登録

を行う。

途中でエラーが発生した場合に、
一部の評価額だけを
データベースへ残してはならない。

そのため、

```text
BEGIN
    ↓
必要に応じてsnapshot作成
    ↓
month_end_holding_values登録
    ↓
すべて成功
    ↓
COMMIT
```

とする。

途中で失敗した場合は、

```text
ROLLBACK
```

し、
CSV-006実行前の状態へ戻す。

---

#### 29.13 snapshotを必要時に作成する理由

商品別月末評価額を
登録するためには、
対象年月の
月末資産状況が必要となる。

利用者に、

```text
CSV登録前に
対象年月のsnapshotを作成する
```

という操作を
必須とすると、
操作手順が増える。

そのため、
対象年月のsnapshotが
存在しない場合は、
CSV-006内で
未確定状態として作成する。

概念的には、

```text
user_id
    = 操作対象利用者ID

target_year_month
    = CSVのtarget_year_month

confirmed
    = false
```

とする。

ただし、
CSV検証に失敗した状態で
snapshotだけを作成してはならない。

---

#### 29.14 snapshotを自動確定しない理由

CSV-006は、
商品単位の
商品別月末評価額だけを
登録するAPIである。

対象年月には、

* 別の保有商品の評価額
* 別の資産口座の商品別月末評価額
* 口座単位で管理する月末資産残高

などが
まだ登録されていない可能性がある。

そのため、

```text
CSV-006成功
    ↓
snapshot自動確定
```

とはしない。

CSV登録と
月末資産状況の確定は、
別の業務操作として扱う。

---

#### 29.15 既存商品別月末評価額を上書きしない理由

CSV-006へ

```text
新規登録
+
既存データ更新
```

の両方の責務を持たせると、

```text
既存値がある場合はどうするか
同じ値なら成功扱いするか
異なる値なら上書きするか
一部だけ更新するか
```

など、
挙動が複雑になる。

Phase1では、
CSV-006を

```text
新規一括登録API
```

に限定する。

既存の
`month_end_holding_values`が
存在する場合は
重複エラーとする。

既存評価額の変更は、
商品別月末評価額の
更新責務を持つ処理へ委譲する。

---

#### 29.16 同じvalueでも重複とする理由

既存の商品別月末評価額と、
CSVに指定された`value`が
同じ場合でも、
登録成功とはしない。

例えば、

```text
既存value
    = 1500000

CSV value
    = 1500000
```

であっても、

```text
既存商品別月末評価額あり
    ↓
重複
```

とする。

CSV-006を
疑似的なupsert APIとして
扱わないためである。

---

#### 29.17 valueを合算しない理由

同一CSV内に、

```text
証券口座
全世界株式
1500000
```

と、

```text
証券口座
全世界株式
500000
```

が存在していても、

```text
1500000 + 500000
    =
2000000
```

として自動的に
合算しない。

商品別月末評価額は、
対象年月・保有商品に対する
1つの評価額として扱う。

同じ

```text
asset_account_name
+
holding_asset_name
```

が複数行存在する場合は、
CSV内重複として扱う。

---

#### 29.18 importedCountだけを返却する理由

CSV-006実行前には、
CSV-005で
登録予定内容を確認できる。

そのため、
CSV-006の成功レスポンスで
登録した商品別月末評価額を
すべて再返却する必要はない。

登録成功の確認に必要な

```text
targetYearMonth
importedCount
```

を中心とした
最小限の結果を返却する。

これにより、
登録APIのレスポンスを
簡潔に保つ。

---

#### 29.19 内部IDをレスポンスへ返却しない理由

CSV-006の処理では、

```text
asset_account_id
holding_asset_id
month_end_asset_snapshot_id
month_end_holding_value_id
```

などの
内部IDを使用する。

ただし、
これらはバックエンド内部の
関連付けに必要な情報であり、
CSV登録結果として
フロントエンドへ公開する必要はない。

CSV-006の成功レスポンスでは、
業務上必要な情報だけを返却する。

---

#### 29.20 409 Conflictを使用する理由

確定済み月末資産状況や
既存商品別月末評価額は、
CSVファイル自体が
解析不能なわけではない。

CSV-006実行時点の
サーバー側業務状態と競合し、
登録できない状態である。

そのため、
これらの競合については、

```text
409 Conflict
```

として扱う。

代表例：

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

---

#### 29.21 CSV入力不正を422とする理由

以下のような
CSV入力自体の問題については、

* CSV構造不正
* 必須値不足
* 対象年月形式不正
* 複数対象年月
* value形式不正
* CSV内重複
* 資産口座不存在
* 保有商品不存在
* 残高記録単位不一致
* 対象年月時点で保有商品が無効

HTTPリクエスト自体は
受信できているが、
業務処理可能な入力ではない。

そのため、

```text
422 Unprocessable Entity
```

として扱う。

---

#### 29.22 UNIQUE制約を最終防衛線とする理由

アプリケーション側で、

```text
既存商品別月末評価額なし
```

と判定した直後に、
別リクエストが
同じ評価額を登録する可能性がある。

そのため、
テーブル定義で一意性を保証する場合は、

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

のUNIQUE制約を
最終防衛線として使用する。

概念的には、

```text
アプリケーション
    ↓
事前重複確認

+

データベース
    ↓
UNIQUE制約
```

の二重防御とする。

フロントエンドの
二重送信防止だけで
データ整合性を保証しない。

---

#### 29.23 Idempotency-Keyを採用しない理由

Phase1では、
CSV登録の重複防止を、

* アプリケーション側の重複確認
* 登録直前の再確認
* トランザクション
* UNIQUE制約
* フロントエンドの二重送信防止

によって実現する。

`Idempotency-Key`を導入すると、

* Key保存
* Keyの有効期限管理
* 利用者との関連管理
* リクエスト内容との関連管理
* レスポンス再現
* Key重複時の挙動

などの追加設計が必要になる。

そのため、
Phase1では採用しない。

---

#### 29.24 通信エラー後に登録結果を断定できない理由

Phase1では
`Idempotency-Key`を採用しない。

そのため、
CSV-006送信後に
通信が切断された場合、

```text
登録処理前に失敗した
```

のか、

```text
登録処理は成功したが
レスポンスだけ受信できなかった
```

のかを、
クライアント側から
完全には判定できない。

同じCSVを再送した場合、
既に登録済みであれば
重複エラーとなる可能性がある。

必要に応じて、
商品別月末評価額の
参照APIから
現在状態を再取得する。

---

#### 29.25 CSV-005とCSV-006で検証ロジックを共通化する理由

CSV-005とCSV-006で
同じ検証ルールを
別々に実装すると、
将来的に仕様差異が
発生する可能性がある。

例えば、

```text
CSV-005
    ↓
登録可能

CSV-006
    ↓
同じCSVなのに入力エラー
```

という状態は避ける。

そのため、
可能な限り、

* CSV Definition
* CSV Parser
* CSV入力値Validator
* 業務ルールValidator
* Query

を共通利用する。

CSV-006固有の責務は、

```text
最新状態の再確認
    ↓
トランザクション
    ↓
必要に応じたsnapshot作成
    ↓
商品別月末評価額登録
```

とする。

---

#### 29.26 CSV行ごとにDB検索しない理由

CSVには、
複数の商品別月末評価額が
含まれる可能性がある。

各行について、

```text
asset_account検索
    ↓
holding_asset検索
    ↓
snapshot検索
    ↓
existing value検索
```

を繰り返すと、
CSV行数に応じて
SQL発行回数が増加する。

そのため、
基本的には、

```text
CSV全体解析
    ↓
必要な資産口座名抽出
    ↓
asset_accounts一括取得
    ↓
必要な保有商品特定
    ↓
holding_assets一括取得
    ↓
snapshot取得
    ↓
existing values一括取得
    ↓
メモリ上で照合
```

とする。

---

#### 29.27 snapshot確定状態を登録時点で確認する理由

CSV-006で
商品別月末評価額を登録する直前に、
対象snapshotが
別処理で確定される可能性がある。

例えば、

```text
CSV-006
事前検証
confirmed = false
    ↓

別処理
SNP確定
confirmed = true
    ↓

CSV-006
INSERT
```

となると、
確定済みsnapshotへ
新しい評価額が追加される可能性がある。

そのため、
登録時点で
必要な排他制御または再確認を行い、

```text
confirmed = false
```

であることを保証したうえで
登録する。

---

#### 29.28 既存評価額を登録直前にも保証する理由

CSV-006の事前検証で

```text
既存month_end_holding_valueなし
```

と確認した直後に、
別リクエストによって
同じ評価額が登録される可能性がある。

そのため、

```text
アプリケーション側重複確認
+
登録時点の競合対策
+
UNIQUE制約
```

によって
重複登録を防止する。

事前SELECTだけに
整合性保証を依存しない。

---

#### 29.29 CSV登録成功だけで月末資産全体が完成したと判断しない理由

CSV-006で登録されるのは、
CSVに含まれる
商品別月末評価額だけである。

CSVに含まれていない、

* 別の資産口座
* 別の保有商品
* 口座単位で管理する月末資産残高

が存在する可能性がある。

そのため、

```text
CSV-006成功
    ≠
対象年月の月末資産入力完了
```

とする。

月末資産全体の
入力完了・確定可否については、
月末資産状況側の責務とする。

---

#### 29.30 CSVに含まれない保有商品を自動補完しない理由

対象年月時点で
有効な保有商品が複数存在していても、
CSVに含まれていない商品について

```text
value = 0
```

などを
自動登録しない。

例えば、

```text
証券口座

全世界株式
S&P500
国内株式
```

が存在し、
CSVに

```text
全世界株式
S&P500
```

だけが含まれていても、

```text
国内株式
value = 0
```

を自動生成しない。

```text
0円
```

と

```text
未登録
```

は、
異なる状態として扱うためである。

---

#### 29.31 口座単位残高を自動生成しない理由

CSV-006は、
商品単位で管理する資産口座の
`month_end_holding_values`を
登録するAPIである。

商品別月末評価額の合計から、

```text
month_end_asset_balances.balance
```

を自動生成しない。

概念的には、

```text
商品単位
    ↓
month_end_holding_values.value
```

と、

```text
口座単位
    ↓
month_end_asset_balances.balance
```

を
別の記録方式として扱う。

両者を
同じ資産口座について
重複して作成しない。

---

#### 29.32 フロントエンドで最終登録可否を再計算しない理由

登録可否の判定には、

* 最新のsnapshot確定状態
* 最新の既存商品別月末評価額
* 資産口座の状態
* 残高記録単位
* 保有商品の状態
* 資産口座と保有商品の関連
* 対象年月時点の有効性

など、
フロントエンドだけでは
正確に保証できない情報が含まれる。

そのため、
React側では
CSV-005の

```text
canImport
```

を
登録ボタンなどの
画面制御に利用してよいが、
最終的な登録可否は
CSV-006へ委ねる。

```text
React
    ↓
canImportによるUI制御

CSV-006
    ↓
最新状態で最終判定
```

とする。

---

#### 29.33 登録成功後に関連データを再取得する理由

CSV-006によって
商品別月末評価額が
新規登録されるため、
フロントエンドが保持している
商品別月末評価額のキャッシュは
最新状態ではなくなる。

そのため、
登録成功後は必要に応じて、
関連する参照Queryを
invalidateし、
サーバーから最新状態を取得する。

少なくとも、
商品別月末評価額を表示する画面では、
CSV登録成功後に
最新データへ更新できるようにする。

---

### 30 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [エラーコード一覧](../../error-codes.md)
- [CSV-004 商品別月末評価額CSVテンプレート取得](./csv-004-template.md)
- [CSV-005 商品別月末評価額CSVプレビュー](./csv-005-preview.md)
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