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

### 29 テスト観点

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

#### 29.1 正常系

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

#### 29.2 CSV-004との整合性

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

#### 29.3 CSV-005との整合性

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

#### 29.4 file未指定

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

#### 29.5 CSV以外のファイル

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

#### 29.6 ファイルサイズ超過

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

#### 29.7 空ファイル

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

#### 29.8 ヘッダー不正

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

#### 29.9 ヘッダー順序不正

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

#### 29.10 余分なヘッダー

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

#### 29.11 データ行0件

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

#### 29.12 target_year_month未入力

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,全世界株式,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 29.13 target_year_month形式不正

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

#### 29.14 対象年月混在

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

#### 29.15 asset_account_name未入力

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,,全世界株式,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 29.16 資産口座不存在

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

#### 29.17 他利用者にのみ同名資産口座が存在する

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

#### 29.18 論理削除済み資産口座

`asset_accounts.deleted_at`が
設定された資産口座を指定する。

以下を確認する。

- 有効な資産口座として扱われないこと
- CSV全体が登録されないこと

---

#### 29.19 残高記録単位が商品単位

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

#### 29.20 残高記録単位が口座単位

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

#### 29.21 holding_asset_name未入力

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 29.22 保有商品不存在

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

#### 29.23 他資産口座にのみ同名保有商品が存在する

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

#### 29.24 他利用者にのみ同名保有商品が存在する

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

#### 29.25 論理削除済み保有商品

`holding_assets.deleted_at`が
設定された保有商品を指定する。

以下を確認する。

- 有効な保有商品として扱われないこと
- 商品別月末評価額が登録されないこと

---

#### 29.26 対象年月時点で有効

対象年月時点で
商品別月末評価額の
記録対象として有効な
保有商品を指定する。

他の条件が正常であれば、
登録できること。

---

#### 29.27 対象年月時点で無効

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

#### 29.28 value未入力

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 29.29 valueが文字列

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

#### 29.30 valueが小数

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

#### 29.31 valueが負数

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

#### 29.32 valueが0円

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

#### 29.33 桁区切り付きvalue

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

#### 29.34 通貨記号付きvalue

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

#### 29.35 CSV内重複

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

#### 29.36 月末資産状況が存在しない

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

#### 29.37 月末資産状況が未確定

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

#### 29.38 月末資産状況が確定済み

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

#### 29.39 他利用者の同一対象年月

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

#### 29.40 他利用者の同一対象年月が確定済み

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

#### 29.41 既存商品別月末評価額

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

#### 29.42 既存値と同じvalue

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

#### 29.43 既存値と異なるvalue

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

#### 29.44 一部だけ既存

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

#### 29.45 複数エラー

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

#### 29.46 一部登録されないこと

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

#### 29.47 全件検証後に登録されること

CSV前半の行が正常で、
末尾行がエラーとなるCSVを送信する。

以下を確認する。

- 前半の正常行が先に登録されないこと
- エラー発見時点でDBに部分データが存在しないこと

---

#### 29.48 トランザクション成功

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

#### 29.49 トランザクションロールバック

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

#### 29.50 既存snapshot使用時のロールバック

既存の未確定snapshotへ
複数件登録する途中で
例外を発生させる。

以下を確認する。

- 新規登録した商品別月末評価額がすべてロールバックされること
- 既存snapshot自体は残ること
- snapshotの`confirmed`が変更されないこと

---

#### 29.51 snapshot重複作成防止

同一利用者、
同一対象年月について
CSV-006を並行実行する。

以下を確認する。

- `month_end_asset_snapshots`が重複作成されないこと
- `user_id + target_year_month`の一意性が維持されること
- 不整合なsnapshotが残らないこと

---

#### 29.52 CSV-003とのsnapshot作成競合

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

#### 29.53 商品別月末評価額の同時登録

同一snapshot、
同一保有商品について
複数リクエストを
並行実行する。

以下を確認する。

- 同一商品の評価額が複数件登録されないこと
- UNIQUE制約によって重複が防止されること
- 競合したリクエストが適切な`409 Conflict`となること

---

#### 29.54 CSV登録後も未確定

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

#### 29.55 CSVに含まれない保有商品

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

#### 29.56 月末資産残高への副作用

CSV-006実行前後で、

```text
month_end_asset_balances
```

が変更されないことを確認する。

商品単位データの登録によって
口座単位残高を
自動作成・更新しないこと。

---

#### 29.57 利用者コンテキスト未指定

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

#### 29.58 利用者ID形式不正

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

#### 29.59 利用者不存在

存在しない利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 29.60 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 29.61 他利用者データの非更新

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

#### 29.62 正常レスポンス契約

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

#### 29.63 返却しない情報

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

#### 29.64 エラーレスポンス契約

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

#### 29.65 同一CSVの再実行

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

#### 29.66 通信失敗後の再送

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

#### 29.67 Idempotency-Keyなし

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

#### 29.68 INTERNAL_SERVER_ERROR

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

### 30 Laravel実装方針

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

#### 30.1 Action

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

#### 30.2 Request

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

#### 30.3 Requestで行うこと

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

#### 30.4 Requestで行わないこと

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

#### 30.5 Middleware

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

#### 30.6 UseCase

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

#### 30.7 CSV-005との共通化

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

#### 30.8 CSV Definition

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

#### 30.9 CSV Parser

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

#### 30.10 CSVヘッダー検証

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

#### 30.11 CSV Validator

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

#### 30.12 valueの型変換

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

#### 30.13 対象年月特定

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

#### 30.14 CSV内重複判定

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

#### 30.15 Import Validator

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

#### 30.16 AssetAccountQuery

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

#### 30.17 HoldingAssetQuery

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

#### 30.18 Map化

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

#### 30.19 残高記録単位判定

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

#### 30.20 対象年月時点の有効性判定

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

#### 30.21 MonthEndAssetSnapshotQuery

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

#### 30.22 snapshot不存在

事前検証時に
snapshotが存在しなくても、
それだけでは
登録不可としない。

CSV-006では、
トランザクション内で
必要に応じて作成する。

---

#### 30.23 確定済みsnapshot

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

#### 30.24 MonthEndHoldingValueQuery

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

#### 30.25 事前検証後にトランザクションを開始する

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

#### 30.26 トランザクション内の再取得

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

#### 30.27 lockForUpdate

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

#### 30.28 snapshotの必要時作成

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

#### 30.29 snapshot重複作成への対応

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

#### 30.30 confirmedの最終確認

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

#### 30.31 既存評価額の最終確認

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

#### 30.32 UNIQUE制約

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

#### 30.33 Repository

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

#### 30.34 MonthEndHoldingValueRepository

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

#### 30.35 一括INSERT

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

#### 30.36 現在時刻

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

#### 30.37 全件成功・全件失敗

CSV-006では、
部分登録を許可しない。

Repositoryで
一部だけ登録された後に
例外が発生した場合でも、
トランザクションによって
全件ロールバックする。

---

#### 30.38 UNIQUE制約違反の変換

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

#### 30.39 snapshot UNIQUE制約違反

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

#### 30.40 Import Result DTO

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

#### 30.41 importedCount

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

#### 30.42 API Resource

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

#### 30.43 Responder

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

#### 30.44 Responderで行わないこと

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

#### 30.45 返却しない情報

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

#### 30.46 エラー変換

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

#### 30.47 複数入力エラー

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

#### 30.48 INTERNAL_SERVER_ERROR

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

#### 30.49 ログ

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

#### 30.50 キャッシュ

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

#### 30.51 Idempotency-Key

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

#### 30.52 テスト実装方針

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

#### 30.53 CSV Parser・ValidatorのUnit Test

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

#### 30.54 Import ValidatorのUnit Test

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

#### 30.55 RepositoryのDatabase Test

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

#### 30.56 UseCaseのUnit Test

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

#### 30.57 トランザクションテスト

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

#### 30.58 並行実行テスト

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

#### 30.59 CSV-005との整合性テスト

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

### 31. React・TypeScriptでの利用

CSV-006は、CSV-005 商品別月末評価額CSVプレビューで内容を確認した後に、同じCSVファイルを再送して商品別月末評価額を一括登録する際に使用する。 :contentReference[oaicite:0]{index=0}

CSV-006は業務データを変更するPOST APIであるため、TanStack Queryを使用する場合はMutationとして扱う。

概念的な利用フローは、以下とする。

```text
CSVファイル選択
    ↓
CSV-005
商品別月末評価額CSVプレビュー
    ↓
canImport = true
    ↓
プレビュー内容表示
    ↓
利用者が登録実行
    ↓
CSV-006
同じCSVファイルを再送
    ↓
サーバー側で再検証
    ↓
201 Created
    ↓
関連Query Cache無効化
    ↓
最新データ再取得
```

---

#### 31.1 TypeScript型

CSV-006では、`multipart/form-data`でCSVファイルを送信する。

API呼び出しに必要な値は、

```text
file
```

のみとする。

概念例：

```ts
export type ImportMonthEndHoldingValueCsvVariables = {
  file: File;
};
```

`userId`、`targetYearMonth`、`assetAccountId`、`holdingAssetId`などをMutation引数へ含めない。

---

#### 31.2 正常レスポンス型

CSV-006の正常レスポンスは、以下の型として定義する。

概念例：

```ts
export type MonthEndHoldingValueCsvImportResult = {
  targetYearMonth: string;
  importedCount: number;
};
```

API共通Envelopeを使用する場合は、以下のように定義する。

```ts
export type ImportMonthEndHoldingValueCsvResponse =
  ApiResponse<MonthEndHoldingValueCsvImportResult>;
```

レスポンス例：

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "importedCount": 2
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

---

#### 31.3 importedCount

`importedCount`は、実際に新規登録された

```text
month_end_holding_values
```

の件数として扱う。

型は、

```text
number
```

とする。

正常レスポンスでは1以上となる。

---

#### 31.4 targetYearMonth

`targetYearMonth`は、

```text
YYYY-MM
```

形式のstringとして扱う。

概念例：

```ts
targetYearMonth: string;
```

必要に応じて、画面表示時に

```text
2026-07
    ↓
2026年7月
```

などへ変換する。

---

#### 31.5 API Client

CSV-006を呼び出す専用API Client関数を定義する。

概念例：

```ts
export const importMonthEndHoldingValueCsv =
  async (
    file: File,
  ): Promise<MonthEndHoldingValueCsvImportResult> => {
    const formData =
      new FormData();

    formData.append(
      'file',
      file,
    );

    const response =
      await apiClient.post<
        ImportMonthEndHoldingValueCsvResponse
      >(
        '/api/v1/month-end-holding-values/imports',
        formData,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

#### 31.6 Content-Typeを手動設定しない

CSV-006では、`FormData`を使用する。

そのため、以下のように

```ts
headers: {
  'Content-Type': 'multipart/form-data',
}
```

を手動設定しないことを基本とする。

ブラウザまたはHTTPクライアントに

```text
boundary
```

を含む`Content-Type`生成を任せる。

概念的には、

```ts
await apiClient.post(
  '/api/v1/month-end-holding-values/imports',
  formData,
);
```

とする。

---

#### 31.7 X-User-Id

`X-User-Id`は、CSV-006専用処理ではなく、共通API Clientから付与する。

概念例：

```ts
apiClient.interceptors.request.use(
  (config) => {
    config.headers['X-User-Id'] =
      currentUserId;

    return config;
  },
);
```

CSVアップロードComponentやMutation Hookから直接設定しない。

---

#### 31.8 userIdをMutation引数へ含めない

以下のようなMutation関数にはしない。

```ts
importMonthEndHoldingValueCsv(
  userId,
  file,
);
```

利用者IDは、共通API Clientが

```text
X-User-Id
```

として付与する。

CSV-006固有の入力は、

```text
file
```

だけとする。

---

#### 31.9 Mutationとして扱う

CSV-006は商品別月末評価額を新規登録するため、TanStack QueryではMutationとして扱う。

概念例：

```ts
export const useImportMonthEndHoldingValueCsv =
  () => {
    return useMutation({
      mutationFn:
        ({
          file,
        }: ImportMonthEndHoldingValueCsvVariables) =>
          importMonthEndHoldingValueCsv(
            file,
          ),
    });
  };
```

Queryとして実装しない。

---

#### 31.10 CSV-005との責務分離

CSV-005は、CSV内容のプレビューおよび登録可否確認を行う。

CSV-006は、実際の登録処理を行う。

概念的には、

```text
CSV-005
    → Queryではなく
      プレビュー用POST
    → 業務データ更新なし

CSV-006
    → Mutation
    → 業務データ更新あり
```

とする。

HTTPメソッドがどちらもPOSTであっても、フロントエンド上の責務は明確に分離する。

---

#### 31.11 CSV-005と同じFileを保持する

CSV-005成功後にCSV-006を実行するため、選択された

```text
File
```

を登録完了またはファイル再選択まで保持する。

概念的には、

```ts
const [
  selectedFile,
  setSelectedFile,
] = useState<File | null>(
  null,
);
```

とする。

CSV-005のプレビュー結果だけを保持し、元のFileを破棄しない。

---

#### 31.12 previewIdを保持しない

CSV-006では、

```text
previewId
previewToken
```

を使用しない。

そのため、React側でもCSV-005実行後に登録用IDを保持する設計にはしない。

概念的には、

```text
CSV-005
    ↓
Preview Result
+
元のFile
    ↓
CSV-006
元のFileを再送
```

とする。

---

#### 31.13 canImportを登録保証として扱わない

CSV-005で

```text
canImport === true
```

の場合に登録ボタンを表示・活性化してよい。

ただし、

```text
canImport = true
    =
CSV-006も必ず成功する
```

とは扱わない。

CSV-005とCSV-006の間に業務状態が変更される可能性があるためである。

---

#### 31.14 登録ボタン

CSV-005の結果が

```text
canImport = true
```

の場合にのみ、登録ボタンを活性化してよい。

概念例：

```tsx
<button
  type="button"
  disabled={
    !preview.data?.canImport ||
    importMutation.isPending
  }
  onClick={
    handleImport
  }
>
  登録する
</button>
```

ただし、最終的な登録可否はCSV-006のバックエンド処理で保証する。

---

#### 31.15 CSV-005未実行でのCSV-006送信をフロントでは防いでよい

通常の画面フローでは、

```text
ファイル選択
    ↓
CSV-005
    ↓
内容確認
    ↓
CSV-006
```

とする。

そのため、CSV-005未実行状態では登録ボタンを表示しない、または非活性にしてよい。

ただし、CSV-006自体はCSV-005の実行履歴に依存せず単体で安全に検証できるAPIとする。

---

#### 31.16 handleImport

概念例：

```ts
const handleImport =
  async (): Promise<void> => {
    if (
      selectedFile === null
    ) {
      return;
    }

    await importMutation.mutateAsync({
      file:
        selectedFile,
    });
  };
```

CSV内容をReact側で再構築して送信しない。

選択済みの元CSVファイルを送信する。

---

#### 31.17 FormDataへ余分な値を追加しない

以下のような実装にはしない。

```ts
formData.append(
  'targetYearMonth',
  preview.targetYearMonth,
);

formData.append(
  'canImport',
  'true',
);

formData.append(
  'previewId',
  preview.id,
);
```

CSV-006へ送信する業務項目は、

```text
file
```

だけとする。

---

#### 31.18 CSV内容をクライアントで書き換えない

CSV-005のプレビュー結果をもとに、React側で新しいCSVを生成し直してCSV-006へ送信しない。

概念的には、

```text
利用者が選択したCSV
    ↓
CSV-005

同じFile
    ↓
CSV-006
```

とする。

---

#### 31.19 二重送信防止

CSV-006実行中は、登録ボタンを非活性化する。

概念例：

```tsx
<button
  disabled={
    importMutation.isPending
  }
>
  {importMutation.isPending
    ? '登録中...'
    : '登録する'}
</button>
```

これにより、意図しない連続クリックを抑制する。

---

#### 31.20 フロントエンドの二重送信防止だけに依存しない

ボタン非活性化は、UX上の重複送信防止である。

バックエンドでは、

```text
重複確認
トランザクション
UNIQUE制約
```

によって最終的な整合性を保証する。

React側の制御をデータ整合性保証とはしない。

---

#### 31.21 Mutation中の画面操作

CSV-006実行中は、少なくとも以下を制御してよい。

- 登録ボタンを非活性化
- CSVファイル再選択を非活性化
- 再プレビューボタンを非活性化

登録中に対象Fileが切り替わらないようにする。

---

#### 31.22 成功時

CSV-006が成功した場合は、

```http
201 Created
```

として扱う。

Mutation成功時に、例えば

```text
2026年7月の商品別月末評価額を
2件登録しました。
```

などの完了表示を行ってよい。

表示値には、

```text
targetYearMonth
importedCount
```

を使用する。

---

#### 31.23 成功メッセージ

概念例：

```ts
const message =
  `${formatYearMonth(
    result.targetYearMonth,
  )}の商品別月末評価額を`
  + `${result.importedCount}件登録しました。`;
```

文言は、画面設計を正とする。

---

#### 31.24 登録後にプレビュー状態を破棄する

CSV-006成功後は、同じCSVを誤って再登録しないよう、

```text
selectedFile
previewResult
```

をクリアしてよい。

概念例：

```ts
setSelectedFile(
  null,
);

setPreviewResult(
  null,
);
```

画面遷移する場合は、遷移によって状態が破棄されてもよい。

---

#### 31.25 成功後の画面遷移

CSV登録成功後は、例えば

```text
月末資産状況詳細画面
```

へ遷移してよい。

レスポンスには`snapshotId`が含まれないため、遷移方法は画面設計に応じて決定する。

例えば、`targetYearMonth`を使って対象年月の一覧・詳細へ戻る設計としてよい。

---

#### 31.26 成功後に登録内容をレスポンスから再構築しない

CSV-006成功レスポンスには、

```text
targetYearMonth
importedCount
```

しか含まれない。

そのため、成功レスポンスだけから商品別評価額一覧をローカルCacheへ手動追加しない。

登録後は参照APIを再取得することを基本とする。

---

#### 31.27 VAL-001のCache無効化

CSV-006成功後は、商品別月末評価額一覧が変更されている。

対象Snapshotをフロント側で特定できる場合は、VAL-001のQuery Cacheを無効化する。

概念的には、

```ts
await queryClient.invalidateQueries({
  queryKey:
    monthEndHoldingValueKeys.all,
});
```

または、対象Snapshotが分かる場合はより限定したQuery Keyを無効化する。

---

#### 31.28 snapshotIdがレスポンスにない場合

CSV-006レスポンスには、

```text
snapshotId
```

を含めない。

そのため、対象SnapshotのIDを画面コンテキストとして既に保持していない場合は、`targetYearMonth`に関連する月末資産状況Queryを無効化して再取得する。

概念的には、

```text
CSV-006成功
    ↓
SNP系Query invalidate
    ↓
対象年月のSnapshot再取得
    ↓
VAL-001再取得
```

としてよい。

---

#### 31.29 月末資産状況Cacheも無効化する

CSV-006によって対象年月のsnapshotが新規作成される可能性がある。

そのため、月末資産状況一覧などの関連Query Cacheも無効化する。

例えば、

```ts
await queryClient.invalidateQueries({
  queryKey:
    monthEndAssetSnapshotKeys.all,
});
```

とする。

---

#### 31.30 資産推移系Cache

CSV-006による登録結果が資産状況・資産推移表示に影響する場合は、対象となる参照Queryも無効化してよい。

ただし、実際にどのQueryへ影響するかは各参照APIの仕様を正とする。

不要な全Query無効化は避ける。

---

#### 31.31 invalidateの基本方針

Phase1では、成功レスポンスから複雑にCacheを手動更新するより、

```text
CSV-006成功
    ↓
関連Query invalidate
    ↓
サーバーから最新状態再取得
```

を基本とする。

---

#### 31.32 CSV-005のPreview Cache

CSV-005のプレビュー結果をTanStack QueryのCacheとして保持している場合は、CSV-006成功後にそのPreview状態を破棄または無効化する。

登録後も

```text
canImport = true
```

の古いPreviewを表示し続けないようにする。

---

#### 31.33 CSV-005後にCSV-006が409になる場合

CSV-005で

```text
canImport = true
```

だったとしても、CSV-006で

```http
409 Conflict
```

となる場合がある。

主に以下である。

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED

MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

これは、CSV-005後にサーバー状態が変化した正常な競合ケースとして扱う。

---

#### 31.34 MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED

以下のエラーを受信した場合は、

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

例えば、

```text
対象年月の月末資産状況が
既に確定されているため、
登録できません。
```

などを表示する。

CSV-005のPreview結果を登録可能状態として表示し続けない。

---

#### 31.35 MONTH_END_HOLDING_VALUE_ALREADY_EXISTS

以下のエラーを受信した場合は、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

既に商品別月末評価額が登録されていることを利用者へ通知する。

既存値をCSV値で上書きする確認画面などは表示しない。

CSV-006には上書き機能がないためである。

---

#### 31.36 409発生後の再取得

競合エラーが発生した場合は、現在状態がプレビュー時点から変化している可能性が高い。

そのため、必要に応じて

```text
関連Query invalidate
+
CSV-005再実行を促す
```

としてよい。

---

#### 31.37 422エラー

CSVファイルやCSV内容に問題がある場合は、

```http
422 Unprocessable Entity
```

として扱う。

主なエラーコードは、以下とする。

```text
VALIDATION_ERROR
INVALID_CSV_FORMAT
CSV_DATA_REQUIRED
MULTIPLE_TARGET_YEAR_MONTHS
ASSET_ACCOUNT_NOT_FOUND
BALANCE_RECORDING_UNIT_MISMATCH
HOLDING_ASSET_NOT_FOUND
HOLDING_ASSET_NOT_AVAILABLE
INVALID_VALUE
DUPLICATE_HOLDING_ASSET_IN_CSV
```

---

#### 31.38 行単位エラー表示

`error.details`にCSV行番号が含まれる場合は、利用者が修正箇所を確認できる形で表示する。

概念的な型：

```ts
export type CsvImportErrorDetail = {
  rowNumber?: number;
  field?: string;
  code?: string;
  message: string;
};
```

---

#### 31.39 複数エラー表示

複数のCSVエラーが返却された場合は、最初の1件だけでなく可能な範囲で一覧表示する。

概念例：

```tsx
<ul>
  {error.details?.map(
    (
      detail,
      index,
    ) => (
      <li
        key={
          `${detail.rowNumber ?? 'general'}-${index}`
        }
      >
        {detail.message}
      </li>
    ),
  )}
</ul>
```

---

#### 31.40 rowNumber

`rowNumber`は、CSVファイル上の修正箇所を示すために使用する。

React側でDB上の行番号などとして扱わない。

---

#### 31.41 field

`field`が存在する場合は、例えば、

```text
target_year_month
asset_account_name
holding_asset_name
value
```

に対応する表示名へ変換してよい。

概念例：

```ts
const csvFieldLabels = {
  target_year_month:
    '対象年月',

  asset_account_name:
    '資産口座名',

  holding_asset_name:
    '保有商品名',

  value:
    '商品別月末評価額',
} as const;
```

---

#### 31.42 error.codeで処理を分岐する

フロントエンドでは、`error.message`ではなく、

```text
error.code
```

を基準としてエラー処理を分岐する。

概念例：

```ts
switch (error.code) {
  case 'MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED':
    // 確定済み
    break;

  case 'MONTH_END_HOLDING_VALUE_ALREADY_EXISTS':
    // 既存評価額あり
    break;

  case 'INVALID_CSV_FORMAT':
    // CSV形式不正
    break;

  default:
    // 共通エラー
    break;
}
```

---

#### 31.43 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

CSV-006専用Componentに同じ処理を重複実装しない。

---

#### 31.44 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、利用者コンテキストに関する共通エラーとして扱う。

---

#### 31.45 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択されている利用者が有効ではない状態として共通処理する。

---

#### 31.46 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
商品別月末評価額を
登録できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

#### 31.47 CSV-006は自動Retryしない

CSV-006は非冪等なPOST APIであり、サーバー側で登録済みかどうかをクライアントが判断できないケースがある。

そのため、TanStack QueryのMutationで自動Retryを原則として行わない。

概念例：

```ts
useMutation({
  mutationFn:
    importMonthEndHoldingValueCsv,

  retry: false,
});
```

---

#### 31.48 通信失敗時に安易に再送しない

CSV-006実行後に通信が切断された場合、

```text
サーバーでは登録成功
+
クライアントでは結果不明
```

となる可能性がある。

この状態で自動的に同じCSVを再送しない。

---

#### 31.49 通信結果不明時

レスポンスを受け取れなかった場合は、必要に応じて

```text
商品別月末評価額一覧
月末資産状況
```

を再取得し、現在状態を確認する導線を提供してよい。

Phase1では`Idempotency-Key`を使用しないため、同一Mutationの自動再送で初回結果を再現する設計にはしない。

---

#### 31.50 同じCSVを利用者が再送した場合

利用者が手動で同じCSVを再登録した場合は、バックエンドから

```text
409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

となる可能性がある。

React側で同じFileかどうかを比較して登録成功扱いにしない。

---

#### 31.51 フロントエンドで重複判定しない

React側で、

```text
このCSVは以前登録した
```

という履歴管理を行ってCSV-006の重複判定を代替しない。

正式な重複判定は、サーバー側の

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

に基づいて行う。

---

#### 31.52 CSVファイル選択

CSVファイル選択Componentでは、`accept`属性を設定してよい。

概念例：

```tsx
<input
  type="file"
  accept=".csv,text/csv"
  onChange={
    handleFileChange
  }
/>
```

ただし、`accept`はブラウザUI上の補助であり、ファイル検証の保証ではない。

最終検証はバックエンドで行う。

---

#### 31.53 Fileのnull確認

ファイル未選択状態では、CSV-005・CSV-006を実行しない。

概念例：

```ts
if (
  selectedFile === null
) {
  return;
}
```

ただし、バックエンド側の`file`必須検証も維持する。

---

#### 31.54 ファイルサイズのクライアント事前確認

CSV共通仕様の最大ファイルサイズがフロントエンドでも共有されている場合は、送信前に簡易チェックを行ってよい。

ただし、フロントエンドの検証だけを正式な制約にはしない。

バックエンド側でも必ず検証する。

---

#### 31.55 CSV内容をブラウザ側で正式検証しない

React側でCSVを読み込んで、

```text
ヘッダー
対象年月
資産口座
保有商品
value
```

などをバックエンドと同等に正式検証する必要はない。

登録可否の正は、CSV-005・CSV-006のバックエンド検証とする。

---

#### 31.56 フロント側で簡易表示してもよい

UX向上のため、選択したFileについて

```text
ファイル名
ファイルサイズ
```

などを表示してよい。

概念例：

```tsx
<p>
  {selectedFile.name}
</p>
```

ただし、CSV解析結果としてはCSV-005のレスポンスを正とする。

---

#### 31.57 プレビュー結果と選択Fileを紐づける

CSV-005実行後に利用者が別のCSVを選択した場合は、以前のPreview結果を破棄する。

概念的には、

```text
File A選択
    ↓
CSV-005 Preview A
    ↓
File B選択
    ↓
Preview A破棄
    ↓
CSV-005 Preview B
```

とする。

File Bを選択しているのにFile Aの

```text
canImport = true
```

を使って登録ボタンを活性化しない。

---

#### 31.58 CSV-006実行対象とPreview対象を一致させる

登録時には、CSV-005で確認したFileと現在選択されているFileが同じ状態であることを画面状態管理上保証する。

利用者がFileを変更した場合は、再プレビューを必須とするUIにしてよい。

---

#### 31.59 プレビュー後のFile内容変更

ブラウザの`File`オブジェクトは、選択時点のファイルを表す。

利用者がローカルファイルを編集した場合にブラウザ内のFileが自動更新されるとは限らない。

変更内容を登録したい場合は、再選択・再プレビューを行うUIとする。

---

#### 31.60 確認画面

CSV-005で取得したプレビュー内容を表示し、登録前に利用者が確認できるようにする。

例えば、

```text
対象年月
資産口座名
保有商品名
商品別月末評価額
```

を一覧表示してよい。

CSV-006成功レスポンスには明細が含まれないため、登録前確認はCSV-005の責務とする。

---

#### 31.61 登録確認ダイアログ

必要に応じて、CSV-006実行前に

```text
この内容で登録しますか？
```

という確認ダイアログを表示してよい。

ただし、バックエンドの再検証・競合検出は引き続き必要とする。

---

#### 31.62 登録中表示

CSV-006は同期APIであるため、Mutation中は

```text
登録中...
```

などの状態を表示する。

QueueやPolling前提の進捗UIはPhase1では不要とする。

---

#### 31.63 進捗率を表示しない

Phase1ではCSV-006を同期APIとして実装し、バックエンドから進捗情報を返さない。

そのため、

```text
45%
80%
```

などの登録進捗率を疑似的に表示しない。

単純なLoading状態とする。

---

#### 31.64 importedCountを利用した完了表示

成功時には、

```text
result.importedCount
```

を使用して、実際に登録された件数を表示する。

CSV行数をReact側で数えて登録件数として表示しない。

ヘッダーや空行等の扱いとずれる可能性があるためである。

---

#### 31.65 targetYearMonthも成功レスポンスを正とする

完了表示に使用する対象年月は、可能であれば

```text
result.targetYearMonth
```

を正とする。

CSVプレビュー結果の値だけに依存しない。

---

#### 31.66 Mutation Hookの責務

Mutation Hookでは、主に以下を担当する。

```text
CSV-006実行
Mutation状態管理
成功時Cache無効化
```

画面固有のダイアログやレイアウトをHookへ持たせない。

---

#### 31.67 onSuccess

概念例：

```ts
export const useImportMonthEndHoldingValueCsv =
  () => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn:
        ({
          file,
        }: ImportMonthEndHoldingValueCsvVariables) =>
          importMonthEndHoldingValueCsv(
            file,
          ),

      retry:
        false,

      onSuccess:
        async () => {
          await Promise.all([
            queryClient.invalidateQueries({
              queryKey:
                monthEndAssetSnapshotKeys.all,
            }),

            queryClient.invalidateQueries({
              queryKey:
                monthEndHoldingValueKeys.all,
            }),
          ]);
        },
    });
  };
```

実際のQuery Keyは、React共通設計を正とする。

---

#### 31.68 onErrorでToastを固定しない

共通Mutation Hookですべてのエラーを同一Toastへ変換すると、

```text
CSV行エラー
409競合
500エラー
```

の表示を分けにくくなる。

エラーオブジェクトをPageまたはエラー表示Componentへ渡し、必要な表示分岐を行ってよい。

---

#### 31.69 API Clientの責務

API Clientでは、以下を担当する。

```text
FormData生成
POST送信
型付きレスポンス取得
```

以下は担当しない。

- ファイル選択
- Preview表示
- 登録確認
- Toast
- 画面遷移
- Query Cache無効化
- CSV行エラー表示
- 登録ボタン制御

---

#### 31.70 Pageの責務

商品別月末評価額CSV登録Pageでは、主に以下を担当する。

- CSVファイル選択状態管理
- CSV-005プレビュー実行
- プレビュー結果表示
- `canImport`による登録ボタン制御
- CSV-006実行
- 登録中表示
- 登録成功表示
- CSVエラー表示
- 登録後の画面遷移

---

#### 31.71 FileInput Componentの責務

CSV FileInput Componentでは、主に以下を担当する。

- CSVファイル選択
- 選択ファイル名表示
- ファイル変更通知

以下は行わない。

- CSV-005実行
- CSV-006実行
- 業務ルール判定
- 資産口座確認
- 保有商品確認

---

#### 31.72 Preview Componentの責務

CSV-005で取得したプレビュー結果を表示する。

主に、

- 対象年月
- 資産口座名
- 保有商品名
- 商品別月末評価額
- CSVエラー
- `canImport`

などを表示する。

CSV-006の登録処理は行わない。

---

#### 31.73 Import Button Component

登録ボタンを独立Componentとする場合は、

```ts
type Props = {
  disabled: boolean;
  isPending: boolean;
  onClick: () => void;
};
```

など、表示・操作に必要な値だけを受け取る。

API ClientをButton Componentから直接呼び出さない。

---

#### 31.74 Error List Component

CSV行単位のエラー表示が複雑になる場合は、専用Componentへ分離してよい。

概念例：

```tsx
<CsvImportErrorList
  details={
    error.details
  }
/>
```

---

#### 31.75 エラー表示ではコードをそのまま見せない

例えば、

```text
HOLDING_ASSET_NOT_FOUND
```

をそのまま利用者向け画面へ表示するのではなく、必要に応じて分かりやすい文言へ変換する。

ただし、デバッグ・開発環境でコードを補助表示する方針がある場合は、共通設計に従う。

---

#### 31.76 CSV-005のエラーとCSV-006のエラーを区別する

CSV-005では、業務エラーがあっても

```text
200 OK
canImport = false
```

となる。

CSV-006では、登録不可の場合は

```text
4xx Error
```

となる。

React側で同じHTTP状態管理として扱わないよう注意する。

---

#### 31.77 CSV-005のcanImport = false

CSV-005で

```text
canImport = false
```

の場合は、CSV-006を実行しないUIとする。

利用者には、プレビューで返却されたエラー内容を修正してCSVを再選択・再プレビューするよう促す。

---

#### 31.78 CSV-005のcanImport = true後の409

CSV-006で409になった場合は、

```text
CSVファイル自体の入力不正
```

とは限らない。

プレビュー後に業務状態が変わった可能性があるため、

```text
最新状態が変更されたため、
再度プレビューしてください。
```

などの導線を設けてよい。

---

#### 31.79 422の場合はCSV修正を促す

422の場合は、CSV内容またはCSVが参照している業務データが登録条件を満たしていない。

可能であれば`error.details`を表示し、CSVの修正箇所を確認できるようにする。

---

#### 31.80 ファイルをサーバー保存済みとみなさない

CSV-005実行後も、サーバー側にCSVファイルが保持されているとは考えない。

CSV-006実行時には必ずブラウザ側のFileを再送する。

---

#### 31.81 ページ再読み込み

CSVファイルは、通常のブラウザ状態ではページ再読み込み後に保持できない。

そのため、ページ再読み込み後は、

```text
CSV再選択
    ↓
CSV-005再実行
    ↓
CSV-006
```

を基本とする。

FileをLocalStorage等へ保存しない。

---

#### 31.82 FileをLocalStorageへ保存しない

アップロードCSVをBase64等へ変換してLocalStorageへ永続保存しない。

Phase1では、選択中のFileを画面メモリ上でのみ保持する。

---

#### 31.83 CSV内容をログ出力しない

フロントエンド側でも、

```ts
console.log(
  file,
);

console.log(
  preview.rows,
);
```

などによって本番環境のConsoleへ資産情報を不要に出力しない。

---

#### 31.84 エラーオブジェクトにも注意する

HTTPクライアントのエラーオブジェクト全体を

```ts
console.error(
  error,
);
```

として本番環境へ常時出力すると、Request情報等が含まれる可能性がある。

ログ方針は、React共通設計に従う。

---

#### 31.85 概念的なディレクトリ構成

例えば、以下のように整理できる。

```text
features/
└── csv-imports/
    ├── api/
    │   ├── previewMonthEndHoldingValueCsv.ts
    │   └── importMonthEndHoldingValueCsv.ts
    ├── components/
    │   ├── CsvFileInput.tsx
    │   ├── MonthEndHoldingValueCsvPreview.tsx
    │   ├── CsvImportErrorList.tsx
    │   └── CsvImportButton.tsx
    ├── hooks/
    │   ├── usePreviewMonthEndHoldingValueCsv.ts
    │   └── useImportMonthEndHoldingValueCsv.ts
    ├── types/
    │   └── monthEndHoldingValueCsv.ts
    └── pages/
        └── MonthEndHoldingValueCsvImportPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

#### 31.86 CSV-005と型を共通化する

CSV-005とCSV-006で共通する概念については、型を共通化してよい。

例えば、

```text
CSV行エラー
対象年月
CSVフィールド名
```

などである。

一方、

```text
Preview Response
Import Response
```

は責務が異なるため、別の型として定義する。

---

#### 31.87 Import ResponseをPreview Responseから流用しない

以下のように、CSV-005レスポンス型をCSV-006へそのまま流用しない。

```ts
type ImportResponse =
  PreviewResponse;
```

CSV-006成功レスポンスは、

```text
targetYearMonth
importedCount
```

に限定されているため、専用型を定義する。

---

#### 31.88 CSV登録結果をView Modelへ変換してもよい

Phase1では、APIレスポンスをそのまま完了表示へ使用してよい。

将来的に表示要件が複雑になった場合は、

```text
Import Result
    ↓
View Model
    ↓
Completion Component
```

へ分離してよい。

現時点では不要な変換層を追加しない。

---

#### 31.89 フロントエンドで行わないこと

CSV-006のReact・TypeScript実装では、以下をフロントエンドの責務としない。

- 利用者境界の最終保証
- CSV仕様の最終検証
- CSVヘッダーの最終検証
- 資産口座存在確認
- 残高記録単位判定
- 保有商品存在確認
- 資産口座と保有商品の関連保証
- 対象年月時点の有効性判定
- snapshot存在確認
- snapshot作成
- snapshot確定状態の最終判定
- 既存商品別月末評価額の重複判定
- トランザクション制御
- ロック制御
- UNIQUE制約による重複防止
- importedCountの算出
- CSV-005の`canImport`を登録保証として扱うこと

フロントエンドは、

```text
CSV選択
    ↓
CSV-005でプレビュー
    ↓
利用者が内容確認
    ↓
同じFileをCSV-006へ送信
    ↓
Mutation状態管理
    ↓
成功・エラー表示
    ↓
関連Query再取得
```

という責務を基本とする。

---



---

### 33 関連ドキュメント

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