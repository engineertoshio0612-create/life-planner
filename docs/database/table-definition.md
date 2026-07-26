# テーブル定義書

## 1. 命名規約

- テーブル名：複数形（snake_case）
- カラム名：snake_case
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

...