##  CSV-006 商品別月末評価額CSV登録

### 1 概要

CSV-006のテストでは、アップロードされた商品別月末評価額CSVを実行時点の最新状態で再検証し、すべての検証に成功した場合のみ、商品別月末評価額を一括登録できることを確認する。

本APIは、CSV-005 商品別月末評価額CSVプレビューで`canImport = true`となった結果をそのまま信頼して登録するのではなく、CSV-006実行時点の業務データをもとに、CSV構造、入力値、利用者境界、資産口座、保有商品、対象年月時点の有効性、月末資産状況、既存商品別月末評価額、CSV内重複などを再検証する。

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
商品別月末評価額一括登録
    ↓
COMMIT
```

正常系では、CSVの全行が登録条件を満たした場合に`201 Created`となり、CSV行数分の`month_end_holding_values`が正しい保有商品および月末資産状況へ紐づいて登録されることを確認する。

対象年月の月末資産状況が存在しない場合は、操作対象利用者の未確定snapshotを新規作成し、既存の未確定snapshotが存在する場合はそれを利用する。CSV-006による登録成功後も`confirmed = false`が維持され、自動的に月末資産状況を確定しないことを確認する。

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

そのため、CSV全行の検証が完了する前にINSERTしないこと、および登録処理中に例外が発生した場合は、トランザクションによって新規snapshotと商品別月末評価額の登録がすべてロールバックされることを重点的に確認する。

業務ルールでは、資産口座の存在と利用者境界、論理削除、残高記録単位、保有商品の存在と資産口座との関連、対象年月時点の有効性、確定済み月末資産状況、既存商品別月末評価額、CSV内重複などを検証する。

特に既存商品別月末評価額が存在する場合は、CSVの値が既存値と同一か異なるかにかかわらず競合として扱い、既存値の更新やupsertを行わないことを確認する。一部の保有商品のみ登録済みの場合も、未登録の商品だけを部分登録せずCSV全体を失敗させる。

CSV-004およびCSV-005との整合性もテスト対象とする。

```text
CSV-004
テンプレート取得
    ↓
CSV-005
プレビュー
    ↓
canImport = true
    ↓
CSV-006
最新状態で再検証
    ↓
一括登録
```

同一利用者・同一DB状態・同一CSVでは、CSV-005とCSV-006で共通のCSV解析および登録可否ルールが適用されることを確認する。一方、CSV-005実行後にsnapshotの確定や商品別月末評価額の登録などの状態変更が発生した場合は、CSV-006が最新状態を再検証して登録を拒否できることを確認する。

利用者境界については、他利用者の同名資産口座、同名保有商品、同一対象年月のsnapshot、商品別月末評価額などが操作対象利用者の判定や登録へ使用されないことを確認する。

また、CSV-003との同時実行を含む同一利用者・同一対象年月へのsnapshot作成競合や、同一snapshot・同一保有商品への並行登録を検証する。アプリケーション側の重複確認、トランザクション、UNIQUE制約によってsnapshotおよび商品別月末評価額の重複が防止されることを確認する。

CSV-006は非冪等な登録APIであるため、同一CSVを再実行した場合は既存商品別月末評価額との競合となり、重複登録されないことを確認する。通信失敗後に同じCSVが再送された場合についても、登録済みデータが存在すれば競合として扱われることを確認する。

また、CSVに対象年月時点で有効なすべての保有商品を含めることはCSV-006の責務としない。CSVに含まれた商品の登録条件のみを検証し、不足している商品別月末評価額の確認は月末資産状況確定処理の責務とする。

これらのテストにより、CSV-006が利用者境界と業務整合性を維持し、部分登録や重複登録を防止しながら、商品別月末評価額CSVを安全かつ原子的に一括登録できることを保証する。

---

### 2 テスト観点

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

#### 2.1 正常系

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

#### 2.2 CSV-004との整合性

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

#### 2.3 CSV-005との整合性

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

#### 2.4 file未指定

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

#### 2.5 CSV以外のファイル

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

#### 2.6 ファイルサイズ超過

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

#### 2.7 空ファイル

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

#### 2.8 ヘッダー不正

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

#### 2.9 ヘッダー順序不正

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

#### 2.10 余分なヘッダー

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

#### 2.11 データ行0件

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

#### 2.12 target_year_month未入力

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,全世界株式,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 2.13 target_year_month形式不正

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

#### 2.14 対象年月混在

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

#### 2.15 asset_account_name未入力

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,,全世界株式,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 2.16 資産口座不存在

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

#### 2.17 他利用者にのみ同名資産口座が存在する

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

#### 2.18 論理削除済み資産口座

`asset_accounts.deleted_at`が
設定された資産口座を指定する。

以下を確認する。

- 有効な資産口座として扱われないこと
- CSV全体が登録されないこと

---

#### 2.19 残高記録単位が商品単位

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

#### 2.20 残高記録単位が口座単位

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

#### 2.21 holding_asset_name未入力

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,,1500000
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 2.22 保有商品不存在

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

#### 2.23 他資産口座にのみ同名保有商品が存在する

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

#### 2.24 他利用者にのみ同名保有商品が存在する

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

#### 2.25 論理削除済み保有商品

`holding_assets.deleted_at`が
設定された保有商品を指定する。

以下を確認する。

- 有効な保有商品として扱われないこと
- 商品別月末評価額が登録されないこと

---

#### 2.26 対象年月時点で有効

対象年月時点で
商品別月末評価額の
記録対象として有効な
保有商品を指定する。

他の条件が正常であれば、
登録できること。

---

#### 2.27 対象年月時点で無効

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

#### 2.28 value未入力

以下を送信する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,
```

以下を確認する。

- `422 Unprocessable Entity`となること
- CSV全体が登録されないこと

---

#### 2.29 valueが文字列

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

#### 2.30 valueが小数

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

#### 2.31 valueが負数

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

#### 2.32 valueが0円

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

#### 2.33 桁区切り付きvalue

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

#### 2.34 通貨記号付きvalue

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

#### 2.35 CSV内重複

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

#### 2.36 月末資産状況が存在しない

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

#### 2.37 月末資産状況が未確定

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

#### 2.38 月末資産状況が確定済み

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

#### 2.39 他利用者の同一対象年月

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

#### 2.40 他利用者の同一対象年月が確定済み

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

#### 2.41 既存商品別月末評価額

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

#### 2.42 既存値と同じvalue

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

#### 2.43 既存値と異なるvalue

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

#### 2.44 一部だけ既存

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

#### 2.45 複数エラー

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

#### 2.46 一部登録されないこと

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

#### 2.47 全件検証後に登録されること

CSV前半の行が正常で、
末尾行がエラーとなるCSVを送信する。

以下を確認する。

- 前半の正常行が先に登録されないこと
- エラー発見時点でDBに部分データが存在しないこと

---

#### 2.48 トランザクション成功

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

#### 2.49 トランザクションロールバック

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

#### 2.50 既存snapshot使用時のロールバック

既存の未確定snapshotへ
複数件登録する途中で
例外を発生させる。

以下を確認する。

- 新規登録した商品別月末評価額がすべてロールバックされること
- 既存snapshot自体は残ること
- snapshotの`confirmed`が変更されないこと

---

#### 2.51 snapshot重複作成防止

同一利用者、
同一対象年月について
CSV-006を並行実行する。

以下を確認する。

- `month_end_asset_snapshots`が重複作成されないこと
- `user_id + target_year_month`の一意性が維持されること
- 不整合なsnapshotが残らないこと

---

#### 2.52 CSV-003とのsnapshot作成競合

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

#### 2.53 商品別月末評価額の同時登録

同一snapshot、
同一保有商品について
複数リクエストを
並行実行する。

以下を確認する。

- 同一商品の評価額が複数件登録されないこと
- UNIQUE制約によって重複が防止されること
- 競合したリクエストが適切な`409 Conflict`となること

---

#### 2.54 CSV登録後も未確定

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

#### 2.55 CSVに含まれない保有商品

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

#### 2.56 月末資産残高への副作用

CSV-006実行前後で、

```text
month_end_asset_balances
```

が変更されないことを確認する。

商品単位データの登録によって
口座単位残高を
自動作成・更新しないこと。

---

#### 2.57 利用者コンテキスト未指定

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

#### 2.58 利用者ID形式不正

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

#### 2.59 利用者不存在

存在しない利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 2.60 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

#### 2.61 他利用者データの非更新

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

#### 2.62 正常レスポンス契約

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

#### 2.63 返却しない情報

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

#### 2.64 エラーレスポンス契約

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

#### 2.65 同一CSVの再実行

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

#### 2.66 通信失敗後の再送

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

#### 2.67 Idempotency-Keyなし

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

#### 2.68 INTERNAL_SERVER_ERROR

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

### 3 関連ドキュメント

- [CSV-006 API詳細設計](../../api/details/csv-imports/csv-006-create.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [CSV-006 Laravelアーキテクチャ設計](../../architecture/laravel/csv-imports/csv-006-create.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [CSV-006 Reactアーキテクチャ設計](../../architecture/react/csv-imports/csv-006-create.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)