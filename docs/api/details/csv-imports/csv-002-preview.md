##  CSV-002 月末資産残高CSVプレビュー

### 1 概要

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

### 6 パスパラメータ

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

### 7 クエリパラメータ

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

### 8 リクエストヘッダー

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

### 9 リクエストボディ

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

### 10 CSVファイル仕様

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

### 11 バリデーション

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

#### 11.1 file 必須チェック

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

#### 11.2 ファイルであること

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
.txt
.pdf
```

拡張子のみを
唯一の判定根拠とはせず、
実際にCSVとして
読み取り可能であることも確認する。

---

#### 11.4 ファイルサイズ

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

#### 11.5 空ファイル

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

#### 11.6 文字コード

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

#### 11.7 CSVとして読み取り可能であること

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
不正として扱う。

これにより、
CSV-001、
CSV-002、
CSV-003で
CSV仕様を統一する。

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
- `confirmed`

---

#### 11.12 データ行の存在

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

#### 11.13 空行

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

#### 11.14 target_year_month 必須

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

#### 11.17 asset_account_name 必須

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

#### 11.18 asset_account_nameの文字列

`asset_account_name`は、
文字列として扱う。

前後空白の扱いは、
CSV共通仕様または
資産口座名の入力ルールに従う。

入力値を
暗黙的に別名へ変換して
資産口座を特定しない。

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
その行を登録不可とする。

---

#### 11.20 他利用者の同名資産口座

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

#### 11.21 残高記録単位

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

#### 11.22 対象年月時点での資産口座の有効性

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

#### 11.23 balance 必須

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
正常な業務値として扱う。

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
判定へ使用しない。

---

#### 11.28 確定済み月末資産状況

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

#### 11.29 月末資産状況が存在しない場合

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

#### 11.31 CSV内重複

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

#### 11.32 行番号

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

#### 11.33 複数エラー

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

#### 11.34 エラーが1件でも存在する場合

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

#### 11.35 プレビュー時のDB更新禁止

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

#### 11.36 CSV-003との再検証

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

### 12 業務ルール

#### 12.1 プレビューの目的

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

#### 12.2 1ファイル1対象年月

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

#### 12.3 資産口座の特定

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

#### 12.4 利用者境界

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

#### 12.5 残高記録単位

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

#### 12.6 対象年月時点の資産口座

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

#### 12.7 月末資産状況

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

#### 12.8 月末資産状況が存在しない場合

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

#### 12.9 確定済み月末資産状況

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

#### 12.10 月末残高

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

#### 12.11 既存月末資産残高

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

#### 12.12 CSV内重複

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

#### 12.13 部分的な登録可能判定を行わない

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

#### 12.14 複数エラーの返却

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

#### 12.15 行番号

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

#### 12.16 プレビュー成功

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

#### 12.17 プレビューエラー

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

#### 12.18 CSV-003での再検証

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

#### 12.19 プレビュー結果を保存しない

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

#### 12.20 業務データを更新しない

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

### 13 レスポンス

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

#### 13.1 登録可能な場合

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

#### 13.2 登録不可行が存在する場合

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

#### 13.3 複数の行エラー

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

#### 13.4 CSV全体に関するエラー

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

#### 13.5 HTTPエラーとの区別

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

### 14 レスポンス項目

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

#### 14.1 targetYearMonth

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

#### 14.2 canImport

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

#### 14.3 errors

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

#### 14.4 rows

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

#### 14.5 rowNumber

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

#### 14.6 assetAccountName

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

#### 14.7 balance

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

#### 14.8 rows.errors

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

#### 14.9 返却しない情報

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

### 15 エラーレスポンス

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

#### 15.1 エラー一覧

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

#### 15.2 USER_CONTEXT_REQUIRED

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

形式不正の場合、
CSV解析や
業務データ検索は行わない。

---

#### 15.4 USER_NOT_FOUND

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

#### 15.5 file未指定

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

#### 15.6 ファイル形式不正

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

#### 15.7 ファイルサイズ超過

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
行単位のプレビュー結果は
生成しない。

---

#### 15.9 CSV解析不能

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

#### 15.10 CSVヘッダー不正

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

#### 15.11 データ行0件

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

#### 15.12 CSV全体に関する業務エラー

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

#### 15.13 行単位の入力エラー

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

#### 15.14 資産口座不存在

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

#### 15.15 残高記録単位不一致

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

#### 15.16 対象年月時点で資産口座が無効

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

#### 15.17 確定済み月末資産状況

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

#### 15.18 既存月末資産残高

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

#### 15.19 CSV内重複

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

#### 15.20 複数エラー

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

#### 15.21 INTERNAL_SERVER_ERROR

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

### 16 HTTPステータス

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

#### 16.1 200 OK

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

#### 16.2 400 Bad Request

以下の場合に使用する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
```

CSVファイルの
業務内容には使用しない。

---

#### 16.3 404 Not Found

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

#### 16.4 422 Unprocessable Entity

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

#### 16.5 500 Internal Server Error

想定外のサーバー内部エラーが
発生した場合に使用する。

対象となるエラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

---

### 17 副作用

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

#### 17.1 プレビュー結果を保存しない

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

#### 17.2 月末資産状況を作成しない

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

### 18 トランザクション

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

#### 18.1 複数SELECT

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

#### 18.2 CSV-003との差異

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

### 19 ロック

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

#### 19.1 プレビュー後の状態変更

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

### 20 キャッシュ

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

#### 20.1 プレビュー結果のキャッシュ

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

### 21 冪等性

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

#### 21.1 POSTと冪等性

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

#### 21.2 Idempotency-Key

本APIでは、
`Idempotency-Key`を使用しない。

CSV-002は
登録APIではなく、
重複実行によって
業務データの重複登録が
発生しないためである。

---

#### 21.3 再実行

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

#### 21.4 データ状態変更後の再実行

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

### 22 関連テーブル

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

#### 22.1 users

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

### 25 Laravel実装方針

### 25 Laravel実装方針

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

#### 25.1 Action

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

#### 25.2 Request

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

#### 25.3 Requestで行うこと

Requestでは、主に以下を検証する。

- `file`が指定されていること
- HTTPアップロードファイルであること
- 許可されたファイル形式であること
- ファイルサイズ上限以内であること

これらは、CSV内容を解析する前に判定できるHTTPリクエストレベルのバリデーションとする。

---

#### 25.4 Requestで行わないこと

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

#### 25.5 Middleware

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

#### 25.6 UseCase

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

#### 25.8 CSV Parser

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

#### 25.9 CSV Parserの責務

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

#### 25.10 CSV構造異常

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

#### 25.11 CSVヘッダー検証

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

#### 25.12 CSV Validator

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

#### 25.13 行入力値の正規化

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

#### 25.14 0円の扱い

`balance = 0`は、有効な月末残高として扱う。

以下のようなtruthy / falsy判定を使用しない。

```php
if (! $balance) {
    // 0円まで未入力扱いになるため使用しない
}
```

必須判定と数値判定を明確に分離する。

---

#### 25.15 データ行0件

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

#### 25.16 対象年月の特定

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

#### 25.17 CSV内重複判定

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

#### 25.18 AssetAccountQuery

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

#### 25.19 資産口座のMap化

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

#### 25.20 資産口座不存在

操作対象利用者に該当する資産口座が存在しない場合は、HTTP 404にはしない。

行単位のプレビューエラーとして、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を生成する。

他の利用者にのみ同名資産口座が存在する場合も、同じ扱いとする。

---

#### 25.21 残高記録単位の判定

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

#### 25.22 対象年月時点の資産口座有効性

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

#### 25.23 MonthEndAssetSnapshotQuery

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

#### 25.24 月末資産状況不存在

対象年月の月末資産状況が存在しない場合は、CSV-003と同一の業務ルールで判定する。

CSV-003で登録時に月末資産状況を作成する仕様であれば、CSV-002では不存在のみを理由としてプレビューエラーを生成しない。

ただし、CSV-002では実際に

```text
month_end_asset_snapshots
```

を作成しない。

---

#### 25.25 確定状態判定

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

#### 25.26 MonthEndAssetBalanceQuery

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

#### 25.27 既存残高のMap化

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

#### 25.28 既存月末資産残高

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

#### 25.29 N+1問題

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

#### 25.30 CsvPreviewError DTO

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

#### 25.31 Preview Row DTO

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

#### 25.32 Preview DTO

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

#### 25.33 canImportの判定

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

#### 25.34 複数エラー収集

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

#### 25.35 HTTPエラーとプレビューエラーの分離

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

#### 25.36 API Resource

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

#### 25.37 Row Resource

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

#### 25.38 Error Resource

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

#### 25.39 返却しない情報

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

#### 25.40 Responder

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

#### 25.41 Responderの責務

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

#### 25.42 Repository

CSV-002では、Repositoryを使用しない。

本APIは業務データを更新しない参照・検証専用APIであるため、以下はQueryクラスが担当する。

- 資産口座取得
- 月末資産状況取得
- 既存月末資産残高取得

登録・更新・削除処理は存在しないため、CSV-002専用Repositoryは作成しない。

---

#### 25.43 トランザクション

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

#### 25.44 ロック

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

#### 25.45 プレビュー結果の保存

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

#### 25.46 CSV-003との共通化

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

#### 25.47 CSV-003との責務差

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

#### 25.48 キャッシュ

Phase1では、CSV-002専用のサーバー側アプリケーションキャッシュを使用しない。

同一CSVファイルであっても、以下が変更されればプレビュー結果も変化する。

- 資産口座の状態
- 月末資産状況の存在
- `confirmed`
- 既存月末資産残高

そのため、プレビュー実行ごとに最新の業務データを参照する。

---

#### 25.49 ログ

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

#### 25.50 例外変換

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

#### 25.51 想定外例外

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

#### 25.52 テスト実装方針

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

#### 25.53 CSV ParserのUnit Test

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

#### 25.54 CSV ValidatorのUnit Test

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

#### 25.55 業務検証のUnit Test

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

#### 25.56 QueryのDatabase Test

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

#### 25.57 UseCaseのUnit Test

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

#### 25.58 CSV-003との共通検証テスト

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

### 26 React・TypeScriptでの利用

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

#### 26.1 TypeScript型

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

#### 26.2 CSVファイルの保持

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

#### 26.3 ファイル選択

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

#### 26.4 FormData

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

#### 26.5 Content-Type

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

#### 26.6 API Client

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

#### 26.7 X-User-Id

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

#### 26.8 Mutationとして扱う

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

#### 26.9 プレビュー実行

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

#### 26.10 プレビューボタン

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

#### 26.11 プレビュー結果の保持

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

#### 26.12 targetYearMonth

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

#### 26.13 canImport

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

#### 26.14 canImportは登録成功保証ではない

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

#### 26.15 CSV全体エラー

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

#### 26.16 rows

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

#### 26.17 rowNumber

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

#### 26.18 assetAccountName

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

#### 26.19 balance

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

#### 26.20 0円とnull

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

#### 26.21 行エラー

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

#### 26.22 エラー行の表示

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

#### 26.23 200 OKかつcanImport=false

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

#### 26.24 HTTPエラー

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

#### 26.25 VALIDATION_ERROR

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

#### 26.26 INVALID_CSV_FORMAT

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

#### 26.27 利用者関連エラー

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

#### 26.28 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`の場合は、
共通サーバーエラーとして扱う。

CSV登録処理へは進まない。

必要に応じて、
同じCSVを
再プレビューできる導線を表示する。

---

#### 26.29 エラーコードと表示文言

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

#### 26.30 ファイル変更時

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

#### 26.31 再プレビュー

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

#### 26.32 CSV-003登録ボタン

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

#### 26.33 CSV-003へ同じFileを送信する

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

#### 26.34 CSV-003成功後

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

#### 26.35 CSV-003失敗時

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

#### 26.36 CSVをReact側で正式解析しない

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

#### 26.37 canImportをReact側で再計算しない

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

### 27 設計上の補足

#### 27.1 POSTを採用する理由

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

#### 27.2 プレビュー専用APIを分ける理由

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

#### 27.3 canImportを返却する理由

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

#### 27.4 業務エラーを200 OKで返す理由

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

#### 27.5 CSV構造不正を422とする理由

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

#### 27.6 複数エラーを返却する理由

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

#### 27.7 rowNumberを返却する理由

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

#### 27.8 内部IDを返却しない理由

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

#### 27.9 confirmedを返却しない理由

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

#### 27.10 プレビュー結果を保存しない理由

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

#### 27.11 CSV-003で再検証する理由

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

#### 27.12 ロックを保持しない理由

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

#### 27.13 React側で業務検証を再実装しない理由

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

#### 27.14 0円とnullを区別する理由

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

#### 27.15 CSV-001との仕様共通化

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

#### 27.16 CSV-003との検証ロジック共通化

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

#### 27.17 Idempotency-Keyを使用しない理由

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

#### 27.18 キャッシュしない理由

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

### 28 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [エラーコード一覧](../../error-codes.md)
- [CSV-001 月末資産残高CSVテンプレート取得](./csv-001-create.md)
- [CSV-003 月末資産残高CSV登録](./csv-003-create.md)
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
