##  CSV-002 月末資産残高CSVプレビュー

### 1 概要

CSV-002のテストでは、アップロードされた月末資産残高CSVを正しく解析・検証し、CSV-003による登録前のプレビューとして、登録予定内容およびエラー内容を正しく返却できることを確認する。

本APIはCSV内容の検証のみを行い、月末資産残高や月末資産状況などの業務データを更新しない。そのため、CSV解析・入力値検証・業務ルール検証・利用者境界・レスポンス契約に加えて、プレビュー処理による副作用が発生しないことを主要なテスト対象とする。

```text
CSVファイル送信
    ↓
ファイル・ヘッダー検証
    ↓
CSV行解析
    ↓
入力値検証
    ↓
資産口座・対象年月検証
    ↓
既存データ・重複検証
    ↓
プレビュー結果生成
    ↓
canImport・rows・errors確認

---

### 2 テスト観点

#### 2.1 正常系

正常なCSVを送信する。

例：

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,生活用口座,800000
```

以下を確認する。

* `200 OK`となること
* `targetYearMonth = 2026-07`となること
* `canImport = true`となること
* `rows`が2件返却されること
* 各行の`rowNumber`がCSV上の行番号と一致すること
* 各行の`assetAccountName`がCSVと一致すること
* 各行の`balance`がCSVと一致すること
* 各行の`errors`が空配列となること
* CSV全体の`errors`が空配列となること
* 業務データが更新されないこと

---

#### 2.2 CSV-001との整合性

CSV-001で取得したテンプレートへ
正常なデータを入力し、
CSV-002へ送信する。

以下を確認する。

```text
CSV-001
テンプレートヘッダー
    =
CSV-002
受付ヘッダー
```

正常なテンプレートが
ヘッダー不正にならないこと。

---

#### 2.3 CSV-003との整合性

同一システム状態で、
同一CSVを
CSV-002およびCSV-003へ送信する。

CSV-002で

```text
canImport = true
```

となるCSVについて、
CSV-003でも
同じ業務ルール上
登録可能となることを確認する。

CSV-002とCSV-003で、

* CSV解析
* 入力値検証
* 資産口座判定
* 残高記録単位判定
* 対象年月時点の有効性判定
* snapshot状態判定
* 既存残高判定
* CSV内重複判定

に実装差異がないことを確認する。

---

#### 2.4 file未指定

`file`を指定せずに
CSV-002を実行する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

以下も確認する。

* CSV解析へ進まないこと
* `month_end_asset_snapshots`を更新しないこと
* `month_end_asset_balances`を更新しないこと

---

#### 2.5 CSV以外のファイル

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

以下を確認する。

* CSV行解析へ進まないこと
* 業務データが更新されないこと

---

#### 2.6 ファイルサイズ超過

CSV共通仕様で定める
上限を超えるファイルを送信する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

CSV解析へ進まず、
業務データが更新されないこと。

---

#### 2.7 空ファイル

0バイトのCSVファイルを送信する。

期待結果：

```text
422 Unprocessable Entity
INVALID_CSV_FORMAT
```

以下が更新されないこと。

* `month_end_asset_snapshots`
* `month_end_asset_balances`

---

#### 2.8 ヘッダー不正

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

後続の業務検証へ進まないこと。

---

#### 2.9 ヘッダー順序不正

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

#### 2.10 余分なヘッダー

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

#### 2.11 データ行0件

以下を送信する。

```csv
target_year_month,asset_account_name,balance
```

CSV共通仕様および
エラーコード定義に従い、
登録可能とは判定されないことを確認する。

少なくとも、

```text
canImport = true
```

として正常な登録可能CSV扱いしないこと。

業務データが更新されないこと。

---

#### 2.12 target_year_month未入力

以下を送信する。

```csv
target_year_month,asset_account_name,balance
,普通預金,1500000
```

以下を確認する。

* CSV自体を解析可能な場合はプレビュー結果を返却すること
* `canImport = false`となること
* 対象行の`errors`へエラーが設定されること
* 業務データが更新されないこと

---

#### 2.13 target_year_month形式不正

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

* `canImport = false`となること
* 対象行に入力値エラーが設定されること
* 月末資産状況が作成されないこと
* 月末資産残高が登録されないこと

---

#### 2.14 対象年月混在

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-06,普通預金,1500000
2026-07,生活用口座,800000
```

以下を確認する。

* `canImport = false`となること
* 複数対象年月に関するエラーが設定されること
* CSV全体を登録可能と判定しないこと
* 業務データが更新されないこと

---

#### 2.15 asset_account_name未入力

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,,1500000
```

以下を確認する。

* `200 OK`でプレビュー結果を返却できること
* `canImport = false`となること
* 対象行の`errors`へエラーが設定されること
* `assetAccountName`がプレビュー表示に適した値となること

---

#### 2.16 資産口座不存在

操作対象利用者に
存在しない資産口座を指定する。

```csv
target_year_month,asset_account_name,balance
2026-07,存在しない口座,1500000
```

以下を確認する。

```text
canImport = false
```

対象行に、

```text
ASSET_ACCOUNT_NOT_FOUND
```

相当のエラーが設定されること。

---

#### 2.17 他利用者にのみ同名資産口座が存在する

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

以下を確認する。

```text
canImport = false
```

かつ、

```text
ASSET_ACCOUNT_NOT_FOUND
```

相当のエラーとなること。

User Bの資産口座情報を
レスポンスへ含めないこと。

---

#### 2.18 論理削除済み資産口座

`deleted_at`が設定された
資産口座を指定する。

以下を確認する。

* 有効な資産口座として扱われないこと
* `canImport = false`となること
* 論理削除済み資産口座の内部情報を返却しないこと

---

#### 2.19 残高記録単位が口座単位

対象資産口座の

```text
balance_recording_unit
    = 口座単位
```

とする。

他の条件が正常な場合は、

```text
canImport = true
```

となること。

---

#### 2.20 残高記録単位が商品単位

商品単位で管理する
資産口座を指定する。

以下を確認する。

```text
canImport = false
```

対象行に、

```text
BALANCE_RECORDING_UNIT_MISMATCH
```

相当のエラーが設定されること。

以下も確認する。

* `month_end_asset_balances`を更新しないこと
* `month_end_holding_values`も更新しないこと

---

#### 2.21 対象年月時点で資産口座が有効

CSVの対象年月時点で
有効な資産口座を指定する。

他の条件が正常であれば、

```text
canImport = true
```

となること。

---

#### 2.22 対象年月時点で資産口座が無効

CSVの対象年月時点で
利用できない資産口座を指定する。

以下を確認する。

```text
canImport = false
```

対象行に、

```text
ASSET_ACCOUNT_NOT_AVAILABLE
```

相当のエラーが設定されること。

現在時点では有効であっても、
対象年月時点で無効なら
登録可能と判定しないこと。

---

#### 2.23 balance未入力

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,
```

以下を確認する。

* `canImport = false`となること
* 対象行へ入力値エラーが設定されること
* `balance`を0円として扱わないこと

---

#### 2.24 balanceが文字列

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,abc
```

以下を確認する。

* `canImport = false`となること
* 対象行に`INVALID_BALANCE`相当のエラーが設定されること
* レスポンス上の`balance`を`null`として表現してよいこと

---

#### 2.25 balanceが小数

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1000.5
```

以下を確認する。

```text
canImport = false
```

対象行に、

```text
INVALID_BALANCE
```

相当のエラーが設定されること。

---

#### 2.26 balanceが負数

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,-1
```

以下を確認する。

```text
canImport = false
```

対象行に、

```text
INVALID_BALANCE
```

相当のエラーが設定されること。

---

#### 2.27 balanceが0円

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,0
```

他の条件が正常であれば、

```text
canImport = true
```

となること。

レスポンスでは、

```json
"balance": 0
```

となること。

0円を未入力扱いしないこと。

---

#### 2.28 桁区切り付きbalance

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,"1,500,000"
```

以下を確認する。

```text
canImport = false
```

対象行に、

```text
INVALID_BALANCE
```

相当のエラーが設定されること。

---

#### 2.29 通貨記号付きbalance

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,¥1500000
```

以下を確認する。

```text
canImport = false
```

対象行に、

```text
INVALID_BALANCE
```

相当のエラーが設定されること。

---

#### 2.30 月末資産状況が存在しない

対象年月の
`month_end_asset_snapshots`が
存在しない状態で、
正常なCSVを送信する。

CSV-003で
snapshotを自動作成する仕様であるため、
他の条件が正常であれば、

```text
canImport = true
```

となること。

以下も確認する。

* CSV-002ではsnapshotを作成しないこと
* `month_end_asset_snapshots`の件数が変化しないこと
* `month_end_asset_balances`を登録しないこと

---

#### 2.31 月末資産状況が未確定

対象年月について、

```text
confirmed = false
```

の月末資産状況を用意する。

正常CSVを送信し、
他の条件が正常であれば、

```text
canImport = true
```

となること。

`confirmed`が変更されないこと。

---

#### 2.32 月末資産状況が確定済み

対象年月について、

```text
confirmed = true
```

の月末資産状況を用意する。

以下を確認する。

```text
canImport = false
```

月末資産状況確定済みに関する
エラーが設定されること。

以下も確認する。

* `confirmed`を変更しないこと
* 月末資産残高を登録しないこと

---

#### 2.33 他利用者の同一対象年月

以下を用意する。

```text
User A
2026-07 snapshotなし

User B
2026-07 snapshotあり
```

User Aとして
正常CSVをプレビューする。

以下を確認する。

* User Bのsnapshotを使用しないこと
* User Bのsnapshotの状態が判定へ影響しないこと
* CSV-003でUser A用snapshotを作成可能な仕様であれば、他の条件が正常な場合`canImport = true`となること

---

#### 2.34 他利用者の同一対象年月が確定済み

以下を用意する。

```text
User A
2026-07 snapshotなし

User B
2026-07
confirmed = true
```

User AとしてCSV-002を実行する。

User Bの確定状態によって
User Aの

```text
canImport
```

が`false`にならないこと。

---

#### 2.35 既存月末資産残高

同一対象年月、
同一資産口座について、
既に月末資産残高を用意する。

以下を確認する。

```text
canImport = false
```

対象行に、

```text
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

相当のエラーが設定されること。

既存`balance`が変更されないこと。

---

#### 2.36 既存値と同じbalance

既存の月末資産残高と
CSVの`balance`が同じ場合でも、
登録可能扱いにしない。

以下を確認する。

```text
canImport = false
```

および、

```text
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

相当となること。

同一値であることを理由として
upsert可能と判定しないこと。

---

#### 2.37 既存値と異なるbalance

既存残高が、

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
canImport = false
```

となること。

既存値が
`1600000`へ更新されないこと。

---

#### 2.38 一部だけ既存

以下の状態を用意する。

```text
普通預金
    → 登録済み

生活用口座
    → 未登録
```

両方を含むCSVを送信する。

以下を確認する。

* `canImport = false`となること
* 普通預金の行に重複エラーが設定されること
* 生活用口座の行が正常でもCSV全体を登録可能と判定しないこと
* 業務データが更新されないこと

---

#### 2.39 CSV内重複

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,普通預金,1600000
```

以下を確認する。

```text
canImport = false
```

重複対象行に、

```text
DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

相当のエラーが設定されること。

先勝ち・後勝ちで
正常扱いしないこと。

---

#### 2.40 複数エラー

以下のようなCSVを送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,存在しない口座,100000
2026-07,普通預金,-1
2026-07,,500000
```

可能な範囲で
複数エラーが返却されることを確認する。

以下も確認する。

* `canImport = false`となること
* 最初の1件だけで検証を終了しないこと
* 各エラーが対応する行へ設定されること
* 業務データが更新されないこと

---

#### 2.41 1行に複数エラー

例えば、

```csv
target_year_month,asset_account_name,balance
2026-07,存在しない口座,-1
```

を送信する。

検証可能な範囲で、

```text
ASSET_ACCOUNT_NOT_FOUND
INVALID_BALANCE
```

など複数エラーを
同一行の`errors`へ
格納できることを確認する。

---

#### 2.42 エラーが1件でも存在する場合

3行中、
2行が正常で
1行がエラーとなるCSVを送信する。

以下を確認する。

```text
正常行あり
+
エラー行あり
    ↓
canImport = false
```

正常行が存在することを理由として、
CSV全体を登録可能と判定しないこと。

---

#### 2.43 rowNumber

以下のCSVを送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,1500000
2026-07,生活用口座,800000
```

以下を確認する。

```text
普通預金
rowNumber = 2

生活用口座
rowNumber = 3
```

ヘッダー行を1行目として
行番号が返却されること。

内部配列インデックスを
そのまま返却しないこと。

---

#### 2.44 CSV上の並び順

複数の資産口座を
任意の順番でCSVへ記述する。

レスポンスの`rows`が
CSV上の並び順を
維持していることを確認する。

例えば、

```text
CSV

1. 普通預金
2. 証券口座
3. 生活用口座
```

なら、

```text
rows

1. 普通預金
2. 証券口座
3. 生活用口座
```

となること。

---

#### 2.45 プレビューによるDB更新がないこと

正常CSVを送信し、
CSV-002実行前後の
業務テーブルを比較する。

以下について
レコード件数および値が
変更されないことを確認する。

* `users`
* `asset_accounts`
* `month_end_asset_snapshots`
* `month_end_asset_balances`

特に、

```text
INSERT
UPDATE
DELETE
```

が発生しないこと。

---

#### 2.46 snapshot不存在時も作成しない

対象年月のsnapshotが
存在しない状態で
正常CSVを送信する。

以下を確認する。

```text
CSV-002
    ↓
canImport = true
    ↓
snapshot件数
変更なし
```

CSV-003で作成予定であっても、
CSV-002では作成しないこと。

---

#### 2.47 商品別月末評価額への副作用

CSV-002実行前後で、

```text
month_end_holding_values
```

が変更されないことを確認する。

商品単位データを
誤って登録・更新・削除しないこと。

---

#### 2.48 利用者コンテキスト未指定

`X-User-Id`を指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

以下を確認する。

* CSV解析へ進まないこと
* 業務データを更新しないこと

---

#### 2.49 利用者ID形式不正

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

CSV解析へ進まないこと。

---

#### 2.50 利用者不存在

存在しない利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

業務データを参照・更新する
後続処理へ進まないこと。

---

#### 2.51 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 2.52 他利用者データの非利用

User Aとして
CSV-002を実行する。

User Bに属する以下のデータが、
登録可否判定へ
使用されないことを確認する。

* `asset_accounts`
* `month_end_asset_snapshots`
* `month_end_asset_balances`

また、
User Bの内部情報が
レスポンスへ含まれないこと。

---

#### 2.53 正常レスポンス契約

正常なCSVを送信し、
API共通の成功Envelope形式で
返却されることを確認する。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "canImport": true,
    "errors": [],
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

以下を確認する。

* `200 OK`であること
* `data`がobjectであること
* `targetYearMonth`がstringであること
* `canImport`がbooleanであること
* `errors`がarrayであること
* `rows`がarrayであること
* `rowNumber`がintegerであること
* `assetAccountName`がstringまたは`null`であること
* `balance`がintegerまたは`null`であること
* JSONフィールド名がcamelCaseであること
* `requestId`が設定されること

---

#### 2.54 登録不可時のレスポンス契約

業務エラーを含むが、
CSV構造自体は解析可能なCSVを送信する。

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
        "assetAccountName": "存在しない口座",
        "balance": 1500000,
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

以下を確認する。

* CSV解析自体が成功している場合は`200 OK`となること
* `canImport = false`となること
* 業務エラーをHTTP例外として扱わないこと
* 対象行の`errors`へエラーが設定されること

---

#### 2.55 errorsが空配列

CSV全体エラーが
存在しない場合は、

```json
"errors": []
```

となること。

`null`を返却しないこと。

同様に、
行エラーが存在しない場合も、

```json
"errors": []
```

となること。

---

#### 2.56 返却しない情報

正常・異常いずれの
プレビュー結果にも、
以下の内部情報が
含まれていないことを確認する。

* `user_id`
* `asset_account_id`
* `month_end_asset_snapshot_id`
* `month_end_asset_balance_id`
* `balance_recording_unit`
* `confirmed`
* `created_at`
* `updated_at`
* SQL
* テーブル名
* PostgreSQL制約名
* 他利用者の業務データ

---

#### 2.57 HTTPエラーレスポンス契約

以下の代表的な
HTTPレベルの異常について、
API共通のエラーレスポンス形式となることを確認する。

* `USER_CONTEXT_REQUIRED`
* `INVALID_USER_ID`
* `USER_NOT_FOUND`
* `VALIDATION_ERROR`
* `INVALID_CSV_FORMAT`
* `INTERNAL_SERVER_ERROR`

以下も確認する。

* `error.code`が設定されること
* `error.message`が設定されること
* 必要に応じて`error.details`が設定されること
* `requestId`が設定されること
* SQLが含まれないこと
* PostgreSQLの制約名が含まれないこと
* スタックトレースが含まれないこと
* サーバー内部ファイルパスが含まれないこと

---

#### 2.58 同一CSVの再実行

同一システム状態で、
同じCSVを複数回送信する。

以下を確認する。

```text
1回目
CSV-002
    ↓
canImport = true

2回目
同一CSV
    ↓
canImport = true
```

業務状態に変化がなければ、
同等のプレビュー結果となること。

以下も確認する。

* snapshotが作成されないこと
* balanceが登録されないこと
* 再実行による副作用がないこと

---

#### 2.59 通信失敗後の再送

サーバー側では
プレビュー生成が完了したが、
クライアントが
レスポンスを受信できなかった状態を想定する。

同じCSVを再送する。

以下を確認する。

* 安全に再実行できること
* 業務データが変更されないこと
* 業務状態が同じなら同等の結果となること

---

#### 2.60 業務状態変更後の再実行

1回目は、

```text
confirmed = false
```

の状態でCSV-002を実行する。

```text
canImport = true
```

となることを確認する。

その後、
別処理で、

```text
confirmed = true
```

へ変更する。

同一CSVを
再度CSV-002へ送信する。

以下を確認する。

```text
canImport = false
```

となること。

以前のプレビュー結果を
キャッシュして返却しないこと。

---

#### 2.61 CSV-002後にCSV-003実行前の状態変更

以下の流れをテストする。

```text
CSV-002
    ↓
confirmed = false
    ↓
canImport = true

別処理
    ↓
confirmed = true

CSV-003
    ↓
再検証
    ↓
登録不可
```

以下を確認する。

* CSV-002の`canImport = true`が登録権利の予約にならないこと
* CSV-002からCSV-003までロックを保持しないこと
* CSV-003が最新状態を再検証すること

---

#### 2.62 Idempotency-Keyなし

`Idempotency-Key`を指定せずに
正常にCSVプレビューできることを確認する。

CSV-002では、

```text
Idempotency-Key
```

を必須としないこと。

---

#### 2.63 プレビュー結果を保存しない

CSV-002を実行した後、
プレビュー結果保存用の
レコードが作成されていないことを確認する。

Phase1では、

```text
csv_import_previews
csv_import_sessions
```

のような
プレビュー管理データを
作成しないこと。

---

#### 2.64 previewIdを発行しない

CSV-002の正常レスポンスに、

```text
previewId
previewToken
importSessionId
```

などが含まれていないことを確認する。

CSV-003が、
保存済みプレビューIDを
前提としていないことも確認する。

---

#### 2.65 DBロックを使用しない

CSV-002実行時に、

```php
lockForUpdate()
```

や、

```sql
SELECT ... FOR UPDATE
```

を使用しないことを確認する。

CSV-002実行中に、
他処理による

* 月末資産状況更新
* 月末資産状況確定
* 月末資産残高登録

を不要にブロックしないこと。

---

#### 2.66 トランザクションを使用しない

CSV-002では、
業務データを更新しないため、
明示的な

```php
DB::transaction()
```

を必要としないことを確認する。

プレビュー処理のためだけに
更新トランザクションを
開始しないこと。

---

#### 2.67 キャッシュを使用しない

同じCSVファイルを使用し、
業務データだけを変更して
複数回CSV-002を実行する。

例えば、

```text
1回目
既存balanceなし
    ↓
canImport = true

別処理
balance登録
    ↓

2回目
同一CSV
    ↓
canImport = false
```

となること。

古いプレビュー結果や
業務データ参照結果を
キャッシュして使用しないこと。

---

#### 2.68 N+1問題

複数行CSVを使用して、
発行されるSQLを確認する。

CSV行ごとに、

```text
asset_accounts検索
snapshot検索
existing balance検索
```

を繰り返さないこと。

基本的には、

```text
CSV全体解析
    ↓
必要な資産口座名を収集
    ↓
資産口座一括取得
    ↓
snapshot取得
    ↓
既存残高一括取得
    ↓
メモリ上で検証
```

となることを確認する。

---

#### 2.69 INTERNAL_SERVER_ERROR

CSVプレビュー処理中に
想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

* 業務データが更新されないこと
* SQLをレスポンスへ含めないこと
* PostgreSQL内部エラーを含めないこと
* スタックトレースを含めないこと
* Laravel内部例外メッセージを含めないこと
* サーバーファイルパスを含めないこと

---

### 3 関連ドキュメント

- [CSV-002 API詳細設計](../../api/details/csv-imports/csv-002-preview.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [CSV-002 Laravelアーキテクチャ設計](../../architecture/laravel/csv-imports/csv-002-preview.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [CSV-002 Reactアーキテクチャ設計](../../architecture/react/csv-imports/csv-002-preview.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
