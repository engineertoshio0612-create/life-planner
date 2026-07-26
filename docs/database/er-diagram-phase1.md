# ER図

## Phase1

```mermaid
erDiagram
    users ||--o{ asset_accounts : 所有する
    users ||--o{ net_incomes : 記録する
    users ||--o{ objectives : 設定する
    users ||--o{ month_end_asset_snapshots : 記録する
    asset_accounts ||--o{ holding_assets : 保有する
    asset_accounts ||--o{ asset_account_available_settings : 設定する
    asset_accounts ||--o{ month_end_asset_balances : 残高を記録する
    holding_assets ||--o{ month_end_holding_values : 評価額を記録する
    month_end_asset_snapshots ||--o{ month_end_asset_balances : 口座残高を含む
    month_end_asset_snapshots ||--o{ month_end_holding_values : 商品評価額を含む
    objectives ||--o{ assessment_histories : 判定履歴を持つ

    users {
        bigint id PK
        varchar name
    }

    asset_accounts {
        bigint id PK
        int user_id FK
        varchar name
        smallint balance_recording_unit
    }

    net_incomes {
        bigint id PK
        int user_id FK
        char target_year_month
        int amount
        text memo
    }

    objectives {
        int id PK
        int user_id FK
        varchar name
        char planned_year_month
        int required_expense
        text memo
    }

    assessment_histories {
        bigint id PK
        int objective_id FK
        char target_year_month
        varchar objective_name
        char planned_year_month
        int required_expense
        int available_assets
        int next_month_credit_card_payment
        int average_net_income
        int remaining_funds
        int required_remaining_funds
        smallint assessment_result
        timestamp assessed_at
    }

    holding_assets {
        bigint id PK
        int asset_account_id FK
        varchar name
    }

    asset_account_available_settings {
        bigint id PK
        int asset_account_id FK
        char start_year_month
        char end_year_month
        boolean is_available
    }

    month_end_asset_snapshots {
        bigint id PK
        int user_id FK
        char target_year_month
        timestamp confirmed_at
        timestamp created_at
        timestamp updated_at
    }

    month_end_asset_balances {
        bigint id PK
        int month_end_asset_snapshot_id FK
        int asset_account_id FK
        varchar asset_account_name
        int asset_balance
    }

    month_end_holding_values {
        bigint id PK
        int month_end_asset_snapshot_id FK
        int holding_asset_id FK
        varchar holding_asset_name
        int valuation_amount
    }

```

## 補足

- Users は AssetAccount（資産口座）  を複数所有できる
- Users は NetIncome（手取り収入） を複数所有できる
- Users は Objective(目的)  を複数所有できる
- Users は TargetYearMonth（対象年月） の MonthEndAssetSnapshot(月末資産状況) を記録できる
- AssetAccount（資産口座）は、複数のHoldingAsset（保有商品）を保有できる
- AssetAccount（資産口座）は、適用期間ごとに利用可能資産へ含めるかどうかの設定を持つ
- MonthEndAssetSnapshot（月末資産状況）は、複数のMonthEndAssetBalance（月末資産残高）を含む
- MonthEndAssetSnapshot（月末資産状況）は、複数のMonthEndHoldingValue（商品別月末評価額）を含む
- MonthEndAssetSnapshot（月末資産状況）の状態はconfirmed_atで表現する
  - confirmed_atがNULLの場合は未確定
  - confirmed_atがNULL以外の場合は確定済み
  - 確定後に残高情報を変更した場合はconfirmed_atをNULLに戻す
- MonthEndAssetBalance（月末資産残高）は、AssetAccount（資産口座）ごとの月末残高を記録する
- MonthEndHoldingValue（商品別月末評価額）は、HoldingAsset（保有商品） ごとの月末評価額を記録する
- Objective(目的) は判定時点のAssessmentHistory（判定履歴） を持つことができる
