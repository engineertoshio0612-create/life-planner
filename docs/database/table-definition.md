# テーブル定義書

## 1. 命名規約

- テーブル名：複数形（スネークケース）
- カラム名：スネークケース
- 主キー：id
- 外部キー：{table}_id
- 論理削除：deleted_at（Laravel SoftDeletes）
- 日時：created_at / updated_at

---

## 2. users

### 概要

Life Plannerを利用する利用者。

### カラム一覧

|論理名|物理名|型|長さ|NULL|PK|FK|UNIQUE|DEFAULT|備考|
|---|---|---|:-:|:-:|:-:|:-:|:-:|---|---|
|利用者ID|id|bigint||×|○|×|×|||IDENTITY（Auto Increment）|
|名前|name|varchar|100|×|×|×|×|||
|メールアドレス|email|varchar|255|×|×|×|○|||
|登録日時|created_at|timestamp||×|×|×|×|||
|更新日時|updated_at|timestamp||×|×|×|×|||
|削除日時|deleted_at|timestamp||○|×|×|×||論理削除|

### 制約

- email は UNIQUE

### インデックス

|インデックス名|対象|
|---|---|
|users_email_unique|email|

### リレーション

- users 1 --- N asset_accounts
- users 1 --- N objectives
- users 1 --- N net_incomes
- users 1 --- N month_end_asset_snapshots

### 設計方針

Laravelの標準命名規約を採用する。

- 主キーは id
- 外部キーは {table}_id
- created_at / updated_at を使用
- deleted_at による論理削除を採用

---

## 3. asset_accounts

### 概要

利用者が所有する資産口座。

### カラム一覧

|論理名|物理名|型|長さ|NULL|PK|FK|UNIQUE|DEFAULT|備考|
|---|---|---|:-:|:-:|:-:|:-:|:-:|---|---|
|資産口座ID|id|bigint||×|○|×|×|||IDENTITY（Auto Increment）|
|利用者ID|user_id|bigint||×|×|○|×||||
|名前|name|varchar|100|×|×|×|×|||
|残高記録単位|balance_recording_unit|smallint||×|×|×|×||CHECK制約|
|登録日時|created_at|timestamp||×|×|×|×|||
|更新日時|updated_at|timestamp||×|×|×|×|||
|削除日時|deleted_at|timestamp||○|×|×|×||論理削除|

### 制約

* user_id は users.id を参照する外部キー
* 未削除の資産口座について、user_id と name の組み合わせは一意とする
* balance_recording_unit は以下の値のみ許可する
  * `1`：口座単位
  * `2`：商品単位
* CHECK：`balance_recording_unit IN (1, 2)`

| 制約名                                         | 対象                     | 内容                |
| ------------------------------------------- | ---------------------- | ----------------- |
| asset_accounts_user_id_foreign              | user_id                | users.idを参照する外部キー |
| asset_accounts_balance_recording_unit_check | balance_recording_unit | 1または2のみ許可         |

### インデックス

| インデックス名                            | 対象            | 条件                 | 目的                 |
| ---------------------------------- | ------------- | ------------------ | ------------------ |
| asset_accounts_user_id_index       | user_id       | -                  | 利用者ごとの資産口座検索       |
| asset_accounts_user_id_name_unique | user_id, name | deleted_at IS NULL | 同一利用者内の未削除口座名の重複防止 |


### リレーション

- users 1 --- N asset_accounts
- asset_accounts 1 --- N holding_assets
- asset_accounts 1 --- N asset_account_available_settings
- asset_accounts 1 --- N month_end_asset_balances

### 設計方針

- 資産口座は利用者に帰属する
- 残高の記録方法は、口座単位または商品単位から選択する
- 登録済みの履歴を保持するため、削除は論理削除とする

---

## 4. holding_assets

### 概要

資産口座で保有する金融商品。

### カラム一覧

| 論理名    | 物理名              | 型         |  長さ | NULL |  PK |  FK | UNIQUE | DEFAULT | 備考   |                          |
| ------ | ---------------- | --------- | :-: | :--: | :-: | :-: | :----: | ------- | ---- | ------------------------ |
| 保有商品ID | id               | bigint    |     |   ×  |  ○  |  ×  |    ×   |         |      | IDENTITY（Auto Increment） |
| 資産口座ID | asset_account_id | bigint    |     |   ×  |  ×  |  ○  |    ×   |         |      |                          |
| 名前     | name             | varchar   | 100 |   ×  |  ×  |  ×  |    ×   |         |      |                          |
| 登録日時   | created_at       | timestamp |     |   ×  |  ×  |  ×  |    ×   |         |      |                          |
| 更新日時   | updated_at       | timestamp |     |   ×  |  ×  |  ×  |    ×   |         |      |                          |
| 削除日時   | deleted_at       | timestamp |     |   ○  |  ×  |  ×  |    ×   |         | 論理削除 |                          |

### 制約

* asset_account_id は asset_accounts.id を参照する外部キー
* 未削除の保有商品について、asset_account_id と name の組み合わせは一意とする

| 制約名                                     | 対象               | 内容                         |
| --------------------------------------- | ---------------- | -------------------------- |
| holding_assets_asset_account_id_foreign | asset_account_id | asset_accounts.idを参照する外部キー |

### インデックス

| インデックス名                                     | 対象                     | 条件                 | 目的                  |
| ------------------------------------------- | ---------------------- | ------------------ | ------------------- |
| holding_assets_asset_account_id_index       | asset_account_id       | -                  | 資産口座ごとの保有商品検索       |
| holding_assets_asset_account_id_name_unique | asset_account_id, name | deleted_at IS NULL | 同一資産口座内の未削除商品名の重複防止 |

### リレーション

* asset_accounts 1 --- N holding_assets
* holding_assets 1 --- N month_end_holding_values

### 設計方針

* 保有商品は資産口座に帰属する
* 商品単位で残高を記録する資産口座に対して登録する
* 過去の商品別月末評価額との関連を保持するため、削除は論理削除とする

---

## 5. asset_account_available_settings

### 概要

資産口座を利用可能資産に含めるかどうかを、適用期間ごとに管理する。

### カラム一覧

| 論理名          | 物理名              | 型         |  長さ | NULL |  PK |  FK | UNIQUE | DEFAULT | 備考 |                          |
| ------------ | ---------------- | --------- | :-: | :--: | :-: | :-: | :----: | ------- | -- | ------------------------ |
| 資産口座利用可能設定ID | id               | bigint    |     |   ×  |  ○  |  ×  |    ×   |         |    | IDENTITY（Auto Increment） |
| 資産口座ID       | asset_account_id | bigint    |     |   ×  |  ×  |  ○  |    ×   |         |    |                          |
| 適用開始年月       | start_year_month | char      |  7  |   ×  |  ×  |  ×  |    ×   |         |    | YYYY-MM形式                |
| 適用終了年月       | end_year_month   | char      |  7  |   ○  |  ×  |  ×  |    ×   |         |    | NULLの場合は終了年月なし           |
| 利用可能資産区分     | is_available     | boolean   |     |   ×  |  ×  |  ×  |    ×   |         |    | true：含める、false：含めない      |
| 登録日時         | created_at       | timestamp |     |   ×  |  ×  |  ×  |    ×   |         |    |                          |
| 更新日時         | updated_at       | timestamp |     |   ×  |  ×  |  ×  |    ×   |         |    |                          |

### 制約

* asset_account_id は asset_accounts.id を参照する外部キー
* start_year_month は `YYYY-MM` 形式とする
* end_year_month はNULL、または `YYYY-MM` 形式とする
* end_year_month が設定されている場合、start_year_month 以降とする
* 同一資産口座について、適用期間を重複させない
* CHECK：`end_year_month IS NULL OR start_year_month <= end_year_month`

| 制約名                                                       | 対象                               | 内容                         |
| --------------------------------------------------------- | -------------------------------- | -------------------------- |
| asset_account_available_settings_asset_account_id_foreign | asset_account_id                 | asset_accounts.idを参照する外部キー |
| asset_account_available_settings_start_year_month_check   | start_year_month                 | YYYY-MM形式のみ許可              |
| asset_account_available_settings_end_year_month_check     | end_year_month                   | NULLまたはYYYY-MM形式のみ許可       |
| asset_account_available_settings_period_check             | start_year_month, end_year_month | 終了年月は開始年月以降とする             |

### インデックス

| インデックス名                                                 | 対象                                                 | 条件 | 目的                |
| ------------------------------------------------------- | -------------------------------------------------- | -- | ----------------- |
| asset_account_available_settings_asset_account_id_index | asset_account_id                                   | -  | 資産口座ごとの利用可能資産設定検索 |
| asset_account_available_settings_account_period_index   | asset_account_id, start_year_month, end_year_month | -  | 対象年月に適用される設定の検索   |

### リレーション

* asset_accounts 1 --- N asset_account_available_settings

### 設計方針

* 利用可能資産設定は資産口座に帰属する
* 利用可能資産に含めるかどうかを適用期間ごとに記録する
* 過去の設定を上書きせず、期間履歴として保持する
* 同一資産口座の適用期間は重複させない

---

## 6. month_end_asset_balances

### 概要

口座単位で残高を管理する資産口座について、対象年月の月末時点における残高を記録する。

### カラム一覧

| 論理名      | 物理名                         | 型         |  長さ | NULL |  PK |  FK | UNIQUE | DEFAULT | 備考 |                          |
| -------- | --------------------------- | --------- | :-: | :--: | :-: | :-: | :----: | ------- | -- | ------------------------ |
| 月末資産残高ID | id                          | bigint    |     |   ×  |  ○  |  ×  |    ×   |         |    | IDENTITY（Auto Increment） |
| 月末資産状況ID | month_end_asset_snapshot_id | bigint    |     |   ×  |  ×  |  ○  |    ×   |         |    |                          |
| 資産口座ID   | asset_account_id            | bigint    |     |   ×  |  ×  |  ○  |    ×   |         |    |                          |
| 資産口座名    | asset_account_name          | varchar   | 100 |   ×  |  ×  |  ×  |    ×   |         |    | 記録時点の名称を保持               |
| 資産残高     | asset_balance               | int       |     |   ×  |  ×  |  ×  |    ×   |         |    | 日本円の整数値                  |
| 登録日時     | created_at                  | timestamp |     |   ×  |  ×  |  ×  |    ×   |         |    |                          |
| 更新日時     | updated_at                  | timestamp |     |   ×  |  ×  |  ×  |    ×   |         |    |                          |

### 制約

* month_end_asset_snapshot_id は month_end_asset_snapshots.id を参照する外部キー
* asset_account_id は asset_accounts.id を参照する外部キー
* 同一の月末資産状況について、同じ資産口座の残高は1件のみ登録できる
* asset_balance は0以上とする
* CHECK：`asset_balance >= 0`

| 制約名                                               | 対象                          | 内容                                    |
| ------------------------------------------------- | --------------------------- | ------------------------------------- |
| month_end_asset_balances_snapshot_id_foreign      | month_end_asset_snapshot_id | month_end_asset_snapshots.idを参照する外部キー |
| month_end_asset_balances_asset_account_id_foreign | asset_account_id            | asset_accounts.idを参照する外部キー            |
| month_end_asset_balances_asset_balance_check      | asset_balance               | 0以上のみ許可                               |

### インデックス

| インデックス名                                          | 対象                                            | 条件 | 目的                     |
| ------------------------------------------------ | --------------------------------------------- | -- | ---------------------- |
| month_end_asset_balances_snapshot_account_unique | month_end_asset_snapshot_id, asset_account_id | -  | 同一月末資産状況内での資産口座残高の重複防止 |
| month_end_asset_balances_asset_account_id_index  | asset_account_id                              | -  | 資産口座ごとの月末残高履歴検索        |

### リレーション

* month_end_asset_snapshots 1 --- N month_end_asset_balances
* asset_accounts 1 --- N month_end_asset_balances

### 設計方針

* 月末資産残高は月末資産状況に属する
* 口座単位で残高を記録する資産口座のみを対象とする
* 対象年月は親テーブルである month_end_asset_snapshots が保持する
* 資産口座名の変更後も過去時点の表示を再現できるよう、記録時点の資産口座名を保持する
* 同一の月末資産状況に対して、資産口座ごとの残高は1件のみ保持する

---

## 7. net_incomes

### 概要

利用者の対象年月ごとの手取り収入を記録する。

### カラム一覧

| 論理名     | 物理名               | 型         |  長さ | NULL |  PK |  FK | UNIQUE | DEFAULT | 備考 |                          |
| ------- | ----------------- | --------- | :-: | :--: | :-: | :-: | :----: | ------- | -- | ------------------------ |
| 手取り収入ID | id                | bigint    |     |   ×  |  ○  |  ×  |    ×   |         |    | IDENTITY（Auto Increment） |
| 利用者ID   | user_id           | bigint    |     |   ×  |  ×  |  ○  |    ×   |         |    |                          |
| 対象年月    | target_year_month | char      |  7  |   ×  |  ×  |  ×  |    ×   |         |    | YYYY-MM形式                |
| 手取り収入額  | amount            | int       |     |   ×  |  ×  |  ×  |    ×   |         |    | 日本円の整数値                  |
| メモ      | memo              | text      |     |   ○  |  ×  |  ×  |    ×   |         |    | 任意入力                     |
| 登録日時    | created_at        | timestamp |     |   ×  |  ×  |  ×  |    ×   |         |    |                          |
| 更新日時    | updated_at        | timestamp |     |   ×  |  ×  |  ×  |    ×   |         |    |                          |

### 制約

* user_id は users.id を参照する外部キー
* 同一利用者について、対象年月ごとの手取り収入は1件のみ登録できる
* target_year_month は `YYYY-MM` 形式とする
* amount は0以上とする
* CHECK：`target_year_month ~ '^[0-9]{4}-(0[1-9]|1[0-2])$'`
* CHECK：`amount >= 0`

| 制約名                                          | 対象                         | 内容                                  |
| ----------------------------------------------- | ---------------------------- | ------------------------------------- |
| net_incomes_user_id_foreign                     | user_id                      | users.idを参照する外部キー            |
| net_incomes_user_id_target_year_month_unique    | user_id, target_year_month   | 同一利用者の対象年月ごとに1件のみ許可 |
| net_incomes_target_year_month_check             | target_year_month            | YYYY-MM形式のみ許可                   |
| net_incomes_amount_check                        | amount                       | 0以上のみ許可                         |

### インデックス

| インデックス名                                      | 対象                         | 条件 | 目的                     |
| -------------------------------------------- | -------------------------- | -- | ---------------------- |
| net_incomes_user_id_target_year_month_unique | user_id, target_year_month | -  | 同一利用者における対象年月ごとの重複登録防止 |

### リレーション

* users 1 --- N net_incomes

### 設計方針

* 手取り収入は利用者に帰属する
* 手取り収入は利用者ごとに対象年月単位で1件のみ保持する
* 金額は日本円の整数値として保持する
* 直近3か月の平均手取り収入は、本テーブルの記録を基に算出する
* メモは手取り収入に関する補足情報を任意で記録するために保持する

