##  CSV-003 月末資産残高CSV登録

### 1 概要

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

### 2 ユースケース

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

### 3 エンドポイント

```http
POST /api/v1/month-end-asset-balances/imports
```

---

### 4 HTTPメソッド

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

#### 5.1 月末資産状況の利用者境界

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

#### 5.2 月末資産状況が存在しない場合

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

#### 5.3 既存月末資産残高の利用者境界

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

#### 5.4 登録先資産口座の特定

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

#### 5.5 登録する月末資産残高の利用者境界

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

#### 5.6 他利用者データを登録可否判定へ利用しない

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

#### 5.7 X-User-Idのエラー

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

#### 5.8 CSV-002の利用者コンテキストを引き継がない

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

#### 5.9 利用者境界確認後に登録する

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

#### 5.10 登録処理の利用者単位

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

### 6 パスパラメータ

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

### 7 クエリパラメータ

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

### 8 リクエストヘッダー

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

### 9 リクエストボディ

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

### 10 CSVファイル仕様

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

### 11 バリデーション

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

#### 11.1 file 必須

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
.txt
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

上限を超えるファイルは、
CSV解析前に
バリデーションエラーとする。

具体的な上限値は、
CSV共通仕様の
最新定義に従う。

---

#### 11.5 空ファイル

CSVファイルが
0バイトの場合、
またはCSVとして
有効なヘッダー行を
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

CSV-001で提供する
テンプレートと
同じ文字コードを
使用することを前提とする。

UTF-8 BOMを許可する場合は、
先頭ヘッダーへ
BOMが混入しないよう
解析時に適切に除去する。

---

#### 11.7 CSVとして読み取り可能であること

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

#### 11.8 ヘッダー必須

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

#### 11.9 ヘッダー名

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

#### 11.10 ヘッダー順序

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

#### 11.11 余分なヘッダー

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

#### 11.12 データ行の存在

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

#### 11.13 空行

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

#### 11.14 target_year_month 必須

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
年月として有効な値であることも確認する。

---

#### 11.16 1ファイル1対象年月

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

#### 11.17 asset_account_name 必須

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

#### 11.19 資産口座の存在確認

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

#### 11.20 他利用者の同名資産口座

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

#### 11.21 残高記録単位

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

#### 11.22 対象年月時点での資産口座の有効性

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

#### 11.23 balance 必須

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

#### 11.24 balance 数値形式

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

#### 11.25 balance 整数

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

#### 11.26 balance 0円以上

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

#### 11.27 月末資産状況の確認

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

#### 11.28 月末資産状況が存在しない場合

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

#### 11.29 確定済み月末資産状況

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

#### 11.30 既存月末資産残高との重複

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

#### 11.31 overwriteを許可しない

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

#### 11.32 CSV内重複

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

#### 11.33 複数エラー

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

#### 11.34 一部登録を行わない

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

#### 11.35 登録前に全件検証する

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

#### 11.36 CSV-002と同じ検証ロジックを使用する

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

#### 11.37 CSV-002の結果は信頼しない

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

#### 11.38 登録直前の再確認

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

#### 11.39 データベース制約も最終防衛線とする

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

#### 11.40 バリデーション失敗時はDB更新しない

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

### 12 業務ルール

#### 12.1 CSV登録の目的

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

#### 12.2 CSV-002と同じCSV仕様を使用する

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

#### 12.3 1ファイル1対象年月

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

#### 12.4 資産口座の特定

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

#### 12.5 利用者境界

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

#### 12.6 残高記録単位

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

#### 12.7 対象年月時点の資産口座

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

#### 12.8 月末残高

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

#### 12.9 月末資産状況の取得

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

#### 12.10 月末資産状況が存在しない場合

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

#### 12.11 既存の未確定月末資産状況

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

#### 12.12 確定済み月末資産状況

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

#### 12.13 既存月末資産残高

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

#### 12.14 既存データを上書きしない

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

#### 12.15 CSV内重複

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

#### 12.16 全件成功または全件失敗

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

#### 12.17 全件検証後に登録する

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

#### 12.18 登録トランザクション

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

#### 12.19 月末資産状況だけを残さない

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

#### 12.20 一括登録

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

#### 12.21 CSV行番号は保存しない

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

#### 12.22 CSVファイルを保存しない

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

#### 12.23 プレビュー結果を保存しない

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

#### 12.24 CSV-002での正常判定を登録権利としない

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

#### 12.25 同時実行

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

#### 12.26 データベース制約

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

#### 12.27 登録後の確定状態

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

#### 12.28 未登録資産の存在

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

#### 12.29 商品単位データを登録しない

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

#### 12.30 登録件数

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

### 13 レスポンス

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

#### 13.1 正常レスポンス

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

#### 13.2 新しい月末資産状況を作成した場合

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

#### 13.3 既存の未確定月末資産状況を使用した場合

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

#### 13.4 登録した残高一覧を返却しない

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

#### 13.5 エラー時

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

#### 13.6 登録途中のエラー

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

### 14 レスポンス項目

正常時の
`data`配下には、
以下の項目を返却する。

| 項目 | 型 | NULL | 内容 |
|---|---|:---:|---|
| `targetYearMonth` | string | × | 登録対象となった対象年月 |
| `importedCount` | integer | × | 新規登録した月末資産残高件数 |

---

#### 14.1 targetYearMonth

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

#### 14.2 importedCount

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

#### 14.3 importedCountは0にならない

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

#### 14.4 返却しない情報

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

#### 14.5 snapshotIdを返却しない

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

#### 14.6 assetAccountIdを返却しない

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

#### 14.7 confirmedを返却しない

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

#### 14.8 レスポンス例

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

### 15 エラーレスポンス

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

#### 15.1 エラー一覧

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

#### 15.2 USER_CONTEXT_REQUIRED

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

#### 15.3 INVALID_USER_ID

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

#### 15.4 USER_NOT_FOUND

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

#### 15.5 file未指定

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

#### 15.6 ファイル形式不正

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

#### 15.7 ファイルサイズ超過

CSV共通仕様で定める
最大ファイルサイズを超えている場合は、

```text
VALIDATION_ERROR
```

を返却する。

CSV解析や
月末資産状況検索へ進まない。

---

#### 15.8 空ファイル

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

#### 15.9 CSV解析不能

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

#### 15.10 CSVヘッダー不正

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

#### 15.11 データ行0件

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

#### 15.12 対象年月混在

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

#### 15.13 行単位の入力エラー

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

#### 15.14 資産口座不存在

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

#### 15.15 残高記録単位不一致

指定された資産口座が
商品単位で管理されている場合は、

```text
BALANCE_RECORDING_UNIT_MISMATCH
```

として登録を拒否する。

商品単位のデータは、
商品別月末評価額CSV登録APIで扱う。

---

#### 15.16 対象年月時点で資産口座が無効

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

#### 15.17 月末残高不正

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

#### 15.18 CSV内重複

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

#### 15.19 確定済み月末資産状況

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

#### 15.20 既存月末資産残高

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

#### 15.21 同時実行によるUNIQUE制約違反

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

#### 15.22 複数エラーの扱い

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

#### 15.23 エラー詳細

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

#### 15.24 INTERNAL_SERVER_ERROR

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

### 16 HTTPステータス

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

#### 16.1 201 Created

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

#### 16.2 400 Bad Request

以下の場合に使用する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
```

CSV内容の
入力エラーには使用しない。

---

#### 16.3 404 Not Found

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

#### 16.4 409 Conflict

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

#### 16.5 422 Unprocessable Entity

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

#### 16.6 500 Internal Server Error

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

### 17 副作用

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

#### 17.1 正常終了時の更新

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

#### 17.2 更新しないデータ

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

#### 17.3 confirmedを変更しない

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

#### 17.4 エラー時の副作用

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

### 18 トランザクション

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

#### 18.1 同一トランザクションで行う処理

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

#### 18.2 ロールバック

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

#### 18.3 CSV検証処理をトランザクションへ入れすぎない

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

### 19 ロック

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

#### 19.1 snapshot不存在時の競合

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

#### 19.2 月末資産残高の重複防止

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

#### 19.3 過剰なロックを行わない

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

### 20 キャッシュ

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

#### 20.1 登録後のキャッシュ

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

### 21 冪等性

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

#### 21.1 同一CSVの再実行

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

#### 21.2 値が異なる場合も上書きしない

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

#### 21.3 一部だけ既存の場合

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

#### 21.4 Idempotency-Key

Phase1では、
`Idempotency-Key`を採用しない。

意図しない二重登録は、

- アプリケーション側の重複確認
- トランザクション
- データベースUNIQUE制約
- フロントエンドの二重送信防止

によって制御する。

---

#### 21.5 二重送信

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

#### 21.6 通信失敗後の再送

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

#### 21.7 同時実行

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

### 22 関連テーブル

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

#### 22.1 users

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

#### 22.2 asset_accounts

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

#### 22.3 残高記録単位

`asset_accounts.balance_recording_unit`を使用して、対象資産口座が口座単位であることを確認する。

本APIでは、

```text
口座単位
```

の資産口座のみを登録対象とする。

商品単位の資産口座については、`month_end_asset_balances`へ登録しない。

---

#### 22.4 対象年月時点の資産口座

対象年月時点で資産口座が月末資産管理対象として有効であることを確認する。

対象年月時点の有効性判定に追加テーブルを使用する設計の場合は、そのテーブルも関連テーブルへ含める。

具体的な判定方法は、資産口座管理の機能要件およびテーブル定義に従う。

現在時点の状態だけを基準に登録可否を判定しない。

---

#### 22.5 month_end_asset_snapshots

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

#### 22.6 月末資産状況が存在する場合

対象年月の月末資産状況が存在する場合は、既存レコードを使用する。

ただし、

```text
confirmed = true
```

の場合は、CSV登録を許可しない。

未確定の場合のみ、月末資産残高の登録先として使用する。

---

#### 22.7 月末資産状況が存在しない場合

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

#### 22.8 month_end_asset_snapshotsの一意性

同一利用者、同一対象年月について複数の月末資産状況が作成されないようにする。

概念的には、以下の組み合わせに一意性を保証する。

```text
user_id
+
target_year_month
```

同時実行によって月末資産状況が重複作成されないよう、データベース制約を最終防衛線として使用する。

---

#### 22.9 month_end_asset_balances

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

#### 22.10 既存月末資産残高の確認

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

#### 22.11 month_end_asset_balancesの一意性

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

#### 22.12 参照しないテーブル

月末資産残高CSV登録では、以下のテーブルを直接登録対象としない。

- `holding_assets`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

商品単位の商品別月末評価額は、CSV-006 商品別月末評価額CSV登録で扱う。

CSV-003では、`month_end_holding_values`へデータを登録しない。

---

#### 22.13 更新対象テーブル

正常終了時に更新する可能性があるテーブルは、以下とする。

```text
month_end_asset_snapshots
month_end_asset_balances
```

`month_end_asset_snapshots`は、対象年月のsnapshotが存在しない場合のみ新規作成する。

`month_end_asset_balances`は、CSVのデータ行数分を新規登録する。

---

#### 22.14 更新しないテーブル

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

### 23 関連する機能要件

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

### 24 テスト観点

#### 24.1 正常系

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

#### 24.2 CSV-001との整合性

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

#### 24.3 CSV-002との整合性

同一システム状態で、同一CSVをCSV-002およびCSV-003へ送信する。

CSV-002で

```text
canImport = true
```

の場合に、CSV-003でも登録可能となることを確認する。

CSV-002とCSV-003でCSV解析・業務ルールの実装差異がないことを確認する。

---

#### 24.4 file未指定

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

#### 24.5 CSV以外のファイル

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

#### 24.6 ファイルサイズ超過

CSV共通仕様で定める上限を超えるファイルを送信する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

登録処理へ進まないこと。

---

#### 24.7 空ファイル

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

#### 24.8 ヘッダー不正

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

#### 24.9 ヘッダー順序不正

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

#### 24.10 余分なヘッダー

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

#### 24.11 データ行0件

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

#### 24.12 target_year_month未入力

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

#### 24.13 target_year_month形式不正

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

#### 24.14 対象年月混在

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

#### 24.15 asset_account_name未入力

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 24.16 資産口座不存在

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

#### 24.17 他利用者にのみ同名資産口座が存在する

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

#### 24.18 論理削除済み資産口座

`deleted_at`が設定された資産口座を指定する。

以下を確認する。

- 有効な資産口座として扱われないこと
- CSV全体が登録されないこと
- 論理削除済み資産口座へ残高が登録されないこと

---

#### 24.19 残高記録単位が口座単位

対象資産口座の

```text
balance_recording_unit
    = 口座単位
```

とする。

他の条件が正常な場合は、月末資産残高を正常に登録できること。

---

#### 24.20 残高記録単位が商品単位

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

#### 24.21 対象年月時点で資産口座が有効

CSVの対象年月時点で有効な資産口座を指定する。

他の条件が正常であれば、登録できること。

---

#### 24.22 対象年月時点で資産口座が無効

CSVの対象年月時点で利用できない資産口座を指定する。

期待結果：

```text
422 Unprocessable Entity
ASSET_ACCOUNT_NOT_AVAILABLE
```

現在時点で有効であっても、対象年月時点で無効なら登録できないこと。

---

#### 24.23 balance未入力

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 24.24 balanceが文字列

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

#### 24.25 balanceが小数

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

#### 24.26 balanceが負数

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

#### 24.27 balanceが0円

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

#### 24.28 桁区切り付きbalance

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

#### 24.29 通貨記号付きbalance

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

#### 24.30 月末資産状況が存在しない

対象年月の`month_end_asset_snapshots`が存在しない状態で、正常なCSVを送信する。

以下を確認する。

- `201 Created`となること
- `month_end_asset_snapshots`が1件作成されること
- `user_id`が操作対象利用者となること
- `target_year_month`がCSV対象年月となること
- `confirmed = false`となること
- CSV行数分の月末資産残高が登録されること

---

#### 24.31 月末資産状況が未確定

対象年月について、

```text
confirmed = false
```

の月末資産状況を用意する。

正常CSVを送信し、既存snapshotへ月末資産残高が登録されること。

新しいsnapshotが追加作成されないこと。

---

#### 24.32 月末資産状況が確定済み

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

#### 24.33 他利用者の同一対象年月

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

#### 24.34 他利用者の同一対象年月が確定済み

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

#### 24.35 既存月末資産残高

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

#### 24.36 既存値と同じbalance

既存の月末資産残高とCSVの`balance`が同じ場合でも、成功扱いにしない。

期待結果：

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

とする。

CSV-003を疑似的なupsert APIとして扱わないこと。

---

#### 24.37 既存値と異なるbalance

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

#### 24.38 一部だけ既存

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

#### 24.39 CSV内重複

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

#### 24.40 複数エラー

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

#### 24.41 一部登録されないこと

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

#### 24.42 全件検証後に登録されること

CSV前半の行が正常で、末尾行がエラーとなるCSVを送信する。

以下を確認する。

- 前半の正常行が先に登録されないこと
- エラー発見時点でDBに部分データが存在しないこと

---

#### 24.43 トランザクション成功

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

#### 24.44 トランザクションロールバック

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

#### 24.45 既存snapshot使用時のロールバック

既存の未確定snapshotへ複数件登録する途中で例外を発生させる。

以下を確認する。

- 新規登録したbalanceがすべてロールバックされること
- 既存snapshot自体は残ること
- snapshotの`confirmed`が変更されないこと

---

#### 24.46 snapshot重複作成防止

同一利用者、同一対象年月についてCSV-003を並行実行する。

以下を確認する。

- `month_end_asset_snapshots`が重複作成されないこと
- `user_id + target_year_month`の一意性が維持されること
- 不整合なsnapshotが残らないこと

---

#### 24.47 月末資産残高の同時登録

同一snapshot、同一資産口座について複数リクエストを並行実行する。

以下を確認する。

- 同一資産口座の残高が複数件登録されないこと
- UNIQUE制約によって重複が防止されること
- 競合したリクエストが適切な`409 Conflict`となること

---

#### 24.48 CSV登録後も未確定

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

#### 24.49 CSVに含まれない資産口座

対象年月時点で口座単位の資産口座が3件存在する状態で、そのうち2件だけをCSVへ含める。

CSVに含まれた2件が他の条件を満たしている場合は、登録できること。

CSVに含まれていない残り1件を理由としてCSV-003が失敗しないこと。

未登録データの確認や確定可否判定は、月末資産状況側の責務とする。

---

#### 24.50 商品別月末評価額への副作用

CSV-003実行前後で、

```text
month_end_holding_values
```

が変更されないことを確認する。

商品単位データを誤って登録・更新・削除しないこと。

---

#### 24.51 利用者コンテキスト未指定

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

#### 24.52 利用者ID形式不正

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

#### 24.53 利用者不存在

存在しない利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 24.54 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 24.55 他利用者データの非更新

User AとしてCSV-003を実行する。

実行前後で、User Bに属する以下が変更されないことを確認する。

- `asset_accounts`
- `month_end_asset_snapshots`
- `month_end_asset_balances`

---

#### 24.56 正常レスポンス契約

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

#### 24.57 返却しない情報

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

#### 24.58 エラーレスポンス契約

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

#### 24.59 同一CSVの再実行

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

#### 24.60 通信失敗後の再送

サーバー側ではCSV登録が完了したが、クライアントが正常レスポンスを受信できなかった状態を想定する。

同じCSVを再送した場合に、

```text
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

となり得ることを確認する。

再送によって重複データが作成されないこと。

---

#### 24.61 Idempotency-Keyなし

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

#### 24.62 INTERNAL_SERVER_ERROR

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

### 25 Laravel実装方針

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

#### 25.1 Action

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

#### 25.2 Request

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

#### 25.3 Requestで行うこと

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

#### 25.4 Requestで行わないこと

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

#### 25.5 Middleware

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

#### 25.6 UseCase

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

#### 25.7 CSV-002との共通化

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

#### 25.8 CSV Definition

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

#### 25.9 CSV Parser

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

#### 25.10 CSVヘッダー検証

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

#### 25.11 CSV Validator

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

#### 25.12 0円の扱い

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

#### 25.13 データ行0件

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

#### 25.14 対象年月の特定

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

#### 25.15 CSV内重複

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

#### 25.16 AssetAccountQuery

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

#### 25.17 資産口座Map

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

#### 25.18 資産口座不存在

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

#### 25.19 残高記録単位判定

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

#### 25.20 対象年月時点の有効性判定

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

#### 25.21 MonthEndAssetSnapshotQuery

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

#### 25.22 確定状態の事前確認

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

#### 25.23 MonthEndAssetBalanceQuery

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

#### 25.24 全件検証後に登録する

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

#### 25.25 Import Input DTO

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

#### 25.26 Import Result DTO

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

#### 25.27 トランザクション

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

#### 25.28 トランザクション内での再取得

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

#### 25.29 snapshotのロック

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

#### 25.30 snapshot不存在時

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

#### 25.31 MonthEndAssetSnapshotRepository

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

#### 25.32 snapshotの重複作成防止

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

#### 25.33 確定状態の最終確認

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

#### 25.34 既存月末資産残高の最終確認

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

#### 25.35 MonthEndAssetBalanceRepository

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

#### 25.36 一括INSERT

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

#### 25.37 Repositoryの責務

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

#### 25.38 UNIQUE制約

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

#### 25.39 UNIQUE制約違反

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

#### 25.40 snapshot UNIQUE制約違反

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

#### 25.41 部分登録を行わない

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

#### 25.42 confirmedを更新しない

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

#### 25.43 全資産口座の登録完了を要求しない

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

#### 25.44 API Resource

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

#### 25.45 Responder

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

#### 25.46 Responderの責務

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

#### 25.47 返却しない情報

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

#### 25.48 例外

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

#### 25.49 CSV複数エラー

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

#### 25.50 想定外例外

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

#### 25.51 N+1問題

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

#### 25.52 ロック範囲

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

#### 25.53 キャッシュ

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

#### 25.54 ログ

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

#### 25.55 テスト実装方針

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

#### 25.56 CSV ParserのUnit Test

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

#### 25.57 CSV ValidatorのUnit Test

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

#### 25.58 QueryのDatabase Test

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

#### 25.59 RepositoryのDatabase Test

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

#### 25.60 UseCaseのUnit Test

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

#### 25.61 トランザクションのFeature Test

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

#### 25.62 UNIQUE制約のDatabase Test

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

#### 25.63 CSV-002との整合性テスト

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

#### 25.64 同時実行テスト

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

### 26 React・TypeScriptでの利用

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

#### 26.1 TypeScript型

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

#### 26.2 Fileの保持

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

#### 26.3 同じFileを使用する

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

#### 26.4 FormData

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

#### 26.5 Content-Type

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

#### 26.6 X-User-Id

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

#### 26.7 API Client

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

#### 26.8 Mutationとして扱う

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

#### 26.9 登録可能状態

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

#### 26.10 登録ボタン

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

#### 26.11 登録処理

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

#### 26.12 登録確認

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

#### 26.13 CSV-002結果を登録条件としてのみ利用する

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

#### 26.14 登録成功

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

#### 26.15 importedCount

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

#### 26.16 targetYearMonth

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

#### 26.17 登録成功後の状態クリア

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

#### 26.18 file inputのリセット

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

#### 26.19 Query Cacheの無効化

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

#### 26.20 資産状況系Queryの扱い

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

#### 26.21 登録成功後の画面遷移

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

#### 26.22 409 Conflict

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

#### 26.23 確定済みエラー

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

#### 26.24 既存月末資産残高エラー

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

#### 26.25 422 Unprocessable Entity

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

#### 26.26 CSV複数エラー

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

#### 26.27 CSV-003失敗後のプレビュー

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

#### 26.28 VALIDATION_ERROR

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

#### 26.29 INVALID_CSV_FORMAT

`INVALID_CSV_FORMAT`の場合は、
CSV自体を
正常に解析できないことを表示する。

利用者へ
CSV-001のテンプレートを使用して
再作成するよう案内してよい。

---

#### 26.30 利用者関連エラー

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

#### 26.31 INTERNAL_SERVER_ERROR

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

#### 26.32 通信エラー

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

#### 26.33 同一CSVの再送

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

#### 26.34 二重送信防止

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

#### 26.35 Idempotency-Key

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

#### 26.36 登録済みかをフロントエンドだけで判断しない

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

#### 26.37 CSVをReact側で再解析しない

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

#### 26.38 成功後に登録済みデータをレスポンスから構築しない

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

### 27 設計上の補足

#### 27.1 POSTを採用する理由

CSV-003は、
CSVファイルの内容に基づき、
複数の月末資産残高を
新規登録する。

そのため、
HTTPメソッドには
`POST`を採用する。

---

#### 27.2 CSV-002とCSV-003を分離する理由

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

#### 27.3 CSV-003で再検証する理由

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

#### 27.4 プレビュー結果を送信しない理由

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

#### 27.5 Fileを再送する理由

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

#### 27.6 全件成功・全件失敗とする理由

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

#### 27.7 トランザクションを使用する理由

CSV-003では、

- 必要に応じたsnapshot作成
- 複数の月末資産残高登録

を行う。

途中でエラーが発生した場合に
一部だけデータを残さないため、
同一トランザクションで処理する。

---

#### 27.8 snapshotを必要時に作成する理由

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

#### 27.9 snapshotを自動確定しない理由

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

#### 27.10 既存残高を上書きしない理由

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

#### 27.11 importedCountだけを返却する理由

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

#### 27.12 snapshotIdを返却しない理由

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

#### 27.13 409 Conflictを使用する理由

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

#### 27.14 CSV入力不正を422とする理由

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

#### 27.15 UNIQUE制約を使用する理由

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

#### 27.16 Idempotency-Keyを採用しない理由

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

#### 27.17 通信エラー後に成功判定できない理由

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

#### 27.18 React Query Cacheを再取得する理由

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

#### 27.19 資産推移系を即時更新しなくてよい理由

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

#### 27.20 React側で登録可否を再計算しない理由

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

### 28 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [エラーコード一覧](../../error-codes.md)
- [CSV-001 月末資産残高CSVテンプレート取得](./csv-001-create.md)
- [CSV-002 月末資産残高CSVプレビュー](./csv-002-preview.md)
- [CSV-004 商品別月末評価額CSVテンプレート取得](./csv-004-template.md)
- [CSV-005 商品別月末評価額CSVプレビュー](./csv-005-preview.md)
- [CSV-006 商品別月末評価額CSV登録](./csv-006-create.md)
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