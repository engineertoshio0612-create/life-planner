##  CSV-003 月末資産残高CSV登録

### 1 概要

CSV-003のテストでは、アップロードされた月末資産残高CSVを実行時点の最新状態で再検証し、すべての検証に成功した場合のみ、月末資産残高を一括登録できることを確認する。

本APIはCSV-002 月末資産残高CSVプレビューの結果をそのまま信頼して登録するのではなく、CSVファイルの形式、入力値、資産口座、対象年月時点の有効性、月末資産状況、既存月末資産残高、CSV内重複などを再検証する。

```text
CSVファイル受信
    ↓
ファイル・ヘッダー検証
    ↓
CSV行解析
    ↓
入力値検証
    ↓
業務ルール再検証
    ↓
全件登録可能
    ↓
トランザクション開始
    ↓
必要に応じてsnapshot作成
    ↓
月末資産残高一括登録
    ↓
COMMIT
```

正常系では、CSVの全行が登録条件を満たした場合に`201 Created`となり、CSV行数分の`month_end_asset_balances`が正しい資産口座および月末資産状況へ紐づいて登録されることを確認する。

対象年月の月末資産状況が存在しない場合は未確定状態で新規作成し、既存の未確定月末資産状況が存在する場合はそれを利用する。CSV-003によって月末資産状況が自動的に確定されないことも確認する。

CSV内に登録できない行が1件でも存在する場合は、正常な行を含めてCSV全体を登録しない。

```text
正常行
正常行
エラー行
    ↓
CSV全体を失敗
    ↓
新規登録 0件
```

そのため、全行の検証が完了する前にINSERTしないこと、および登録処理中に例外が発生した場合はトランザクションによってsnapshot作成と月末資産残高登録がすべてロールバックされることを重点的に確認する。

業務ルールでは、資産口座の存在と利用者境界、論理削除、残高記録単位、対象年月時点の有効性、確定済み月末資産状況、既存月末資産残高、CSV内重複などを検証する。特に既存データが存在する場合は上書きやupsertを行わず、CSV全体を競合として扱うことを確認する。

CSV-001およびCSV-002との整合性もテスト対象とする。

```text
CSV-001
テンプレート
    ↓
CSV-002
プレビュー
    ↓
canImport = true
    ↓
CSV-003
最新状態で再検証
    ↓
一括登録
```

同一システム状態ではCSV-002で`canImport = true`となったCSVをCSV-003でも登録できることを確認する。一方、CSV-002実行後に月末資産状況や既存残高などの状態が変更された場合は、CSV-003が最新状態を再検証して登録を拒否できることを確認する。

利用者境界については、他利用者の同名資産口座、同一対象年月のsnapshot、確定状態などが操作対象利用者の登録へ影響せず、他利用者の業務データを参照・更新しないことを確認する。

また、同一利用者・同一対象年月への並行登録や同一資産口座への同時登録を検証し、アプリケーション側の重複確認、トランザクション、UNIQUE制約によってsnapshotおよび月末資産残高の重複が防止されることを確認する。

CSV-003は非冪等な登録APIであるため、同一CSVを再実行した場合は既存データとの競合となり、重複登録されないことを確認する。通信失敗後の再送についても同様に、既に登録済みの場合は競合として扱われることを確認する。

これらのテストにより、CSV-003が利用者境界と業務整合性を維持しながら、月末資産残高CSVを部分登録することなく、安全かつ原子的に一括登録できることを保証する。

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

- `201 Created`となること
- `targetYearMonth = 2026-07`となること
- `importedCount = 2`となること
- 2件の`month_end_asset_balances`が登録されること
- 登録された`balance`がCSVと一致すること
- 各行が正しい`asset_account_id`へ紐づくこと
- 各行が同一の`month_end_asset_snapshot_id`へ紐づくこと
- CSV登録によって月末資産状況が確定されないこと

---

#### 2.2 CSV-001との整合性

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

#### 2.3 CSV-002との整合性

同一システム状態で、同一CSVをCSV-002およびCSV-003へ送信する。

CSV-002で

```text
canImport = true
```

の場合に、CSV-003でも登録可能となることを確認する。

CSV-002とCSV-003でCSV解析・業務ルールの実装差異がないことを確認する。

---

#### 2.4 file未指定

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

業務データが更新されないこと。

---

#### 2.6 ファイルサイズ超過

CSV共通仕様で定める上限を超えるファイルを送信する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

登録処理へ進まないこと。

---

#### 2.7 空ファイル

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

登録処理へ進まないこと。

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

#### 2.12 target_year_month未入力

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

- `422 Unprocessable Entity`となること
- 月末資産残高が登録されないこと

---

#### 2.14 対象年月混在

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

#### 2.15 asset_account_name未入力

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 2.16 資産口座不存在

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

期待結果：

```text
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの資産口座へ月末資産残高が登録されないこと。

---

#### 2.18 論理削除済み資産口座

`deleted_at`が設定された資産口座を指定する。

以下を確認する。

- 有効な資産口座として扱われないこと
- CSV全体が登録されないこと
- 論理削除済み資産口座へ残高が登録されないこと

---

#### 2.19 残高記録単位が口座単位

対象資産口座の

```text
balance_recording_unit
    = 口座単位
```

とする。

他の条件が正常な場合は、月末資産残高を正常に登録できること。

---

#### 2.20 残高記録単位が商品単位

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

#### 2.21 対象年月時点で資産口座が有効

CSVの対象年月時点で有効な資産口座を指定する。

他の条件が正常であれば、登録できること。

---

#### 2.22 対象年月時点で資産口座が無効

CSVの対象年月時点で利用できない資産口座を指定する。

期待結果：

```text
422 Unprocessable Entity
ASSET_ACCOUNT_NOT_AVAILABLE
```

現在時点で有効であっても、対象年月時点で無効なら登録できないこと。

---

#### 2.23 balance未入力

以下を送信する。

```csv
target_year_month,asset_account_name,balance
2026-07,普通預金,
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 2.24 balanceが文字列

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

#### 2.25 balanceが小数

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

#### 2.26 balanceが負数

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

#### 2.27 balanceが0円

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

#### 2.28 桁区切り付きbalance

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

#### 2.29 通貨記号付きbalance

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

#### 2.30 月末資産状況が存在しない

対象年月の`month_end_asset_snapshots`が存在しない状態で、正常なCSVを送信する。

以下を確認する。

- `201 Created`となること
- `month_end_asset_snapshots`が1件作成されること
- `user_id`が操作対象利用者となること
- `target_year_month`がCSV対象年月となること
- `confirmed = false`となること
- CSV行数分の月末資産残高が登録されること

---

#### 2.31 月末資産状況が未確定

対象年月について、

```text
confirmed = false
```

の月末資産状況を用意する。

正常CSVを送信し、既存snapshotへ月末資産残高が登録されること。

新しいsnapshotが追加作成されないこと。

---

#### 2.32 月末資産状況が確定済み

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

#### 2.33 他利用者の同一対象年月

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

#### 2.34 他利用者の同一対象年月が確定済み

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

#### 2.35 既存月末資産残高

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

#### 2.36 既存値と同じbalance

既存の月末資産残高とCSVの`balance`が同じ場合でも、成功扱いにしない。

期待結果：

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

とする。

CSV-003を疑似的なupsert APIとして扱わないこと。

---

#### 2.37 既存値と異なるbalance

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

#### 2.38 一部だけ既存

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

#### 2.39 CSV内重複

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

#### 2.40 複数エラー

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

#### 2.41 一部登録されないこと

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

#### 2.42 全件検証後に登録されること

CSV前半の行が正常で、末尾行がエラーとなるCSVを送信する。

以下を確認する。

- 前半の正常行が先に登録されないこと
- エラー発見時点でDBに部分データが存在しないこと

---

#### 2.43 トランザクション成功

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

#### 2.44 トランザクションロールバック

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

#### 2.45 既存snapshot使用時のロールバック

既存の未確定snapshotへ複数件登録する途中で例外を発生させる。

以下を確認する。

- 新規登録したbalanceがすべてロールバックされること
- 既存snapshot自体は残ること
- snapshotの`confirmed`が変更されないこと

---

#### 2.46 snapshot重複作成防止

同一利用者、同一対象年月についてCSV-003を並行実行する。

以下を確認する。

- `month_end_asset_snapshots`が重複作成されないこと
- `user_id + target_year_month`の一意性が維持されること
- 不整合なsnapshotが残らないこと

---

#### 2.47 月末資産残高の同時登録

同一snapshot、同一資産口座について複数リクエストを並行実行する。

以下を確認する。

- 同一資産口座の残高が複数件登録されないこと
- UNIQUE制約によって重複が防止されること
- 競合したリクエストが適切な`409 Conflict`となること

---

#### 2.48 CSV登録後も未確定

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

#### 2.49 CSVに含まれない資産口座

対象年月時点で口座単位の資産口座が3件存在する状態で、そのうち2件だけをCSVへ含める。

CSVに含まれた2件が他の条件を満たしている場合は、登録できること。

CSVに含まれていない残り1件を理由としてCSV-003が失敗しないこと。

未登録データの確認や確定可否判定は、月末資産状況側の責務とする。

---

#### 2.50 商品別月末評価額への副作用

CSV-003実行前後で、

```text
month_end_holding_values
```

が変更されないことを確認する。

商品単位データを誤って登録・更新・削除しないこと。

---

#### 2.51 利用者コンテキスト未指定

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

#### 2.52 利用者ID形式不正

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

#### 2.53 利用者不存在

存在しない利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 2.54 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 2.55 他利用者データの非更新

User AとしてCSV-003を実行する。

実行前後で、User Bに属する以下が変更されないことを確認する。

- `asset_accounts`
- `month_end_asset_snapshots`
- `month_end_asset_balances`

---

#### 2.56 正常レスポンス契約

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

#### 2.57 返却しない情報

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

#### 2.58 エラーレスポンス契約

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

#### 2.59 同一CSVの再実行

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

#### 2.60 通信失敗後の再送

サーバー側ではCSV登録が完了したが、クライアントが正常レスポンスを受信できなかった状態を想定する。

同じCSVを再送した場合に、

```text
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

となり得ることを確認する。

再送によって重複データが作成されないこと。

---

#### 2.61 Idempotency-Keyなし

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

#### 2.62 INTERNAL_SERVER_ERROR

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

### 3 関連ドキュメント

- [CSV-003 API詳細設計](../../api/details/csv-imports/csv-003-create.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [CSV-003 Laravelアーキテクチャ設計](../../architecture/laravel/csv-imports/csv-003-create.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [CSV-003 Reactアーキテクチャ設計](../../architecture/react/csv-imports/csv-003-create.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)