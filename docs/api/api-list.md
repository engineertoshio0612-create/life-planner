# API一覧

## 1. 目的

本書では、
Phase1で提供するAPIの一覧を定義する。

各APIの詳細なリクエスト、
レスポンス、
バリデーション、
業務ルールおよびエラー仕様は、
各詳細設計書で定義する。

---

## 2. 共通事項

### 2.1 ベースURL

```text
/api/v1
```

### 2.2 API共通方針

すべてのAPIは、
[API共通方針](./api-common-policy.md)に従う。

### 2.3 エラーコード

各APIで使用する独自エラーコードは、
[エラーコード一覧](./error-codes.md)で管理する。

### 2.4 利用者コンテキスト

利用者一覧取得APIを除く利用者依存APIでは、
原則として以下のリクエストヘッダーを使用する。

```http
X-User-Id: 1
```

---

## 3. API IDの命名規則

API IDは、
機能分類を表す英字と3桁の連番で構成する。

```text
{機能分類}-{連番}
```

例：

```text
USR-001
ACC-001
SNP-001
```

API IDは、
API一覧、
API詳細設計書、
テスト仕様書およびIssueで共通して使用する。

---

## 4. 利用者API

| API ID | 機能分類 | API名 | HTTPメソッド | URL | 概要 | 主な関連テーブル | 対応する機能要件 | 詳細設計書 |
|---|---|---|---|---|---|---|---|---|
| USR-001 | 利用者 | 利用者一覧取得 | GET | `/api/v1/users` | 選択可能な利用者の一覧を取得する | `users` | 3. 利用者 | [USR-001](./details/users.md#usr-001-利用者一覧取得) |

---

## 5. 資産口座API

| API ID | 機能分類 | API名 | HTTPメソッド | URL | 概要 | 主な関連テーブル | 対応する機能要件 | 詳細設計書 |
|---|---|---|---|---|---|---|---|---|
| ACC-001 | 資産口座 | 資産口座一覧取得 | GET | `/api/v1/asset-accounts` | 操作対象利用者の資産口座一覧を取得する | `asset_accounts`、`asset_account_available_settings` | 4.9 一覧表示、4.10 表示順 | [ACC-001](./details/asset-accounts.md#acc-001-資産口座一覧取得) |
| ACC-002 | 資産口座 | 資産口座登録 | POST | `/api/v1/asset-accounts` | 資産口座と初期の利用可能資産設定を登録する | `asset_accounts`、`asset_account_available_settings` | 4.2 登録 | [ACC-002](./details/asset-accounts.md#acc-002-資産口座登録) |
| ACC-003 | 資産口座 | 資産口座詳細取得 | GET | `/api/v1/asset-accounts/{assetAccountId}` | 指定した資産口座の詳細を取得する | `asset_accounts`、`asset_account_available_settings` | 4. 資産口座管理 | [ACC-003](./details/asset-accounts.md#acc-003-資産口座詳細取得) |
| ACC-004 | 資産口座 | 資産口座更新 | PATCH | `/api/v1/asset-accounts/{assetAccountId}` | 資産口座名、備考、利用状態などの更新可能項目を変更する | `asset_accounts` | 4.3 編集、4.8 無効化 | [ACC-004](./details/asset-accounts.md#acc-004-資産口座更新) |
| ACC-005 | 資産口座 | 利用可能資産設定履歴取得 | GET | `/api/v1/asset-accounts/{assetAccountId}/available-settings` | 資産口座の利用可能資産区分の期間履歴を取得する | `asset_accounts`、`asset_account_available_settings` | 4.6 利用可能資産区分、4.7 変更 | [ACC-005](./details/asset-accounts.md#acc-005-利用可能資産設定履歴取得) |
| ACC-006 | 資産口座 | 利用可能資産設定登録 | POST | `/api/v1/asset-accounts/{assetAccountId}/available-settings` | 適用開始年月を指定して新しい利用可能資産設定を登録する | `asset_accounts`、`asset_account_available_settings` | 4.7 利用可能資産区分の変更 | [ACC-006](./details/asset-accounts.md#acc-006-利用可能資産設定登録) |

---

## 6. 保有商品API

| API ID | 機能分類 | API名 | HTTPメソッド | URL | 概要 | 主な関連テーブル | 対応する機能要件 | 詳細設計書 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| HLD-001 | 保有商品 | 保有商品一覧取得 | GET | `/api/v1/holding-assets` | 操作対象利用者の保有商品一覧を取得する | `holding_assets`、`asset_accounts` | 5.8 一覧表示、5.9 表示順 | [HLD-001](./details/holding-assets.md#has-001-保有商品一覧取得) |
| HLD-002 | 保有商品 | 保有商品登録 | POST | `/api/v1/holding-assets` | 商品単位で管理する資産口座へ保有商品を登録する | `holding_assets`、`asset_accounts` | 5.2 登録 | [HLD-002](./details/holding-assets.md#has-002-保有商品登録) |
| HLD-003 | 保有商品 | 保有商品詳細取得 | GET | `/api/v1/holding-assets/{holdingAssetId}` | 指定した保有商品の詳細を取得する | `holding_assets`、`asset_accounts` | 5.3 編集 | [HLD-003](./details/holding-assets.md#has-003-保有商品詳細取得) |
| HLD-004 | 保有商品 | 保有商品更新 | PATCH | `/api/v1/holding-assets/{holdingAssetId}` | 保有商品名、商品種別および備考を更新する | `holding_assets`、`asset_accounts` | 5.3 編集 | [HLD-004](./details/holding-assets.md#has-004-保有商品更新) |
| HLD-005 | 保有商品 | 保有商品無効化 | PATCH | `/api/v1/holding-assets/{holdingAssetId}/disable` | 指定した保有商品を無効化する | `holding_assets` | 5.7 無効化 | [HLD-005](./details/holding-assets.md#has-005-保有商品無効化) |
---

## 7. 手取り収入API

| API ID | 機能分類 | API名 | HTTPメソッド | URL | 概要 | 主な関連テーブル | 対応する機能要件 | 詳細設計書 |
|---|---|---|---|---|---|---|---|---|
| INC-001 | 手取り収入 | 手取り収入一覧取得 | GET | `/api/v1/net-incomes` | 対象年月ごとの手取り収入一覧を取得する | `net_incomes` | 9.5 一覧表示 | [INC-001](./details/net-incomes.md#inc-001-手取り収入一覧取得) |
| INC-002 | 手取り収入 | 手取り収入登録 | POST | `/api/v1/net-incomes` | 対象年月の手取り収入を登録する | `net_incomes` | 9.2 登録 | [INC-002](./details/net-incomes.md#inc-002-手取り収入登録) |
| INC-003 | 手取り収入 | 手取り収入詳細取得 | GET | `/api/v1/net-incomes/{netIncomeId}` | 指定した手取り収入の詳細を取得する | `net_incomes` | 9. 手取り収入管理 | [INC-003](./details/net-incomes.md#inc-003-手取り収入詳細取得) |
| INC-004 | 手取り収入 | 手取り収入更新 | PATCH | `/api/v1/net-incomes/{netIncomeId}` | 手取り収入および備考を更新する | `net_incomes` | 9.3 編集 | [INC-004](./details/net-incomes.md#inc-004-手取り収入更新) |
| INC-005 | 手取り収入 | 平均手取り収入取得 | GET | `/api/v1/net-incomes/average` | 指定した判定対象年月以前の連続する3か月の平均手取り収入を取得する | `net_incomes` | 9.6 平均手取り収入、9.7 判定対象、9.8 データ不足 | [INC-005](./details/net-incomes.md#inc-005-平均手取り収入取得) |

---

## 8. 目的API

| API ID | 機能分類 | API名 | HTTPメソッド | URL | 概要 | 主な関連テーブル | 対応する機能要件 | 詳細設計書 |
|---|---|---|---|---|---|---|---|---|
| OBJ-001 | 目的 | 目的一覧取得 | GET | `/api/v1/objectives` | 操作対象利用者の目的一覧を取得する | `objectives` | 10.5 一覧表示 | [OBJ-001](./details/objectives.md#obj-001-目的一覧取得) |
| OBJ-002 | 目的 | 目的登録 | POST | `/api/v1/objectives` | 目的名、実施予定年月、必要支出額などを登録する | `objectives` | 10.2 登録 | [OBJ-002](./details/objectives.md#obj-002-目的登録) |
| OBJ-003 | 目的 | 目的詳細取得 | GET | `/api/v1/objectives/{objectiveId}` | 指定した目的の詳細を取得する | `objectives` | 10. 目的管理 | [OBJ-003](./details/objectives.md#obj-003-目的詳細取得) |
| OBJ-004 | 目的 | 目的更新 | PATCH | `/api/v1/objectives/{objectiveId}` | 目的情報を更新する | `objectives` | 10.3 編集 | [OBJ-004](./details/objectives.md#obj-004-目的更新) |
| OBJ-005 | 目的 | 目的無効化 | PATCH | `/api/v1/objectives/{objectiveId}` | 指定した目的を無効化する | `objectives` | 10.4 無効化 | [OBJ-005](./details/objectives.md#obj-005-目的無効化) |
| OBJ-006 | 目的 | 目的達成判定 | POST | `/api/v1/objectives/{objectiveId}/assessments` | 指定した目的の達成可否を判定し、判定履歴を登録する | `objectives`、`assessment_histories`、`month_end_asset_snapshots`、`net_incomes` | 11. 目的達成判定 | [OBJ-006](./details/objectives.md#obj-006-目的達成判定) |

---

## 9. 月末資産状況API

| API ID | 機能分類 | API名 | HTTPメソッド | URL | 概要 | 主な関連テーブル | 対応する機能要件 | 詳細設計書 |
|---|---|---|---|---|---|---|---|---|
| SNP-001 | 月末資産状況 | 月末資産状況一覧取得 | GET | `/api/v1/month-end-asset-snapshots` | 対象年月ごとの確定状態、登録済み件数、未登録件数などを取得する | `month_end_asset_snapshots`、`month_end_asset_balances`、`month_end_holding_values` | 7.11 一覧表示 | [SNP-001](./details/month-end-assets.md#snp-001-月末資産状況一覧取得) |
| SNP-002 | 月末資産状況 | 月末資産状況作成 | POST | `/api/v1/month-end-asset-snapshots` | 指定した対象年月の未確定な月末資産状況を作成する | `month_end_asset_snapshots` | 6.3 対象年月、7.2 確定対象 | [SNP-002](./details/month-end-assets.md#snp-002-月末資産状況作成) |
| SNP-003 | 月末資産状況 | 月末資産状況詳細取得 | GET | `/api/v1/month-end-asset-snapshots/{snapshotId}` | 月末資産状況、登録済み残高、未登録資産などを取得する | `month_end_asset_snapshots`、`month_end_asset_balances`、`month_end_holding_values`、`asset_accounts`、`holding_assets` | 7.4 確定順序、7.5 未登録資産の表示、7.11 一覧表示 | [SNP-003](./details/month-end-assets.md#snp-003-月末資産状況詳細取得) |
| SNP-004 | 月末資産状況 | 月末資産状況確定 | POST | `/api/v1/month-end-asset-snapshots/{snapshotId}/confirm` | 確定条件を検証し、対象年月の月末資産状況を確定する | `month_end_asset_snapshots`、`month_end_asset_balances`、`month_end_holding_values` | 7.3 確定条件、7.4 確定順序、7.6 確定 | [SNP-004](./details/month-end-assets.md#snp-004-月末資産状況確定) |
| SNP-005 | 月末資産状況 | 月末資産状況確定解除 | POST | `/api/v1/month-end-asset-snapshots/{snapshotId}/unconfirm` | 最新の確定済み月末資産状況を未確定へ戻す | `month_end_asset_snapshots` | 7.7 確定解除 | [SNP-005](./details/month-end-assets.md#snp-005-月末資産状況確定解除) |

---

## 10. 月末資産残高API

| API ID | 機能分類 | API名 | HTTPメソッド | URL | 概要 | 主な関連テーブル | 対応する機能要件 | 詳細設計書 |
|---|---|---|---|---|---|---|---|---|
| BAL-001 | 月末資産残高 | 月末資産残高一覧取得 | GET | `/api/v1/month-end-asset-snapshots/{snapshotId}/asset-balances` | 指定した月末資産状況に属する口座単位の残高一覧を取得する | `month_end_asset_snapshots`、`month_end_asset_balances`、`asset_accounts` | 6.4 口座単位での登録、6.10 一覧表示 | [BAL-001](./details/month-end-assets.md#bal-001-月末資産残高一覧取得) |
| BAL-002 | 月末資産残高 | 月末資産残高登録 | POST | `/api/v1/month-end-asset-snapshots/{snapshotId}/asset-balances` | 指定した資産口座の月末残高を登録する | `month_end_asset_snapshots`、`month_end_asset_balances`、`asset_accounts` | 6.4 口座単位での登録 | [BAL-002](./details/month-end-assets.md#bal-002-月末資産残高登録) |
| BAL-003 | 月末資産残高 | 月末資産残高更新 | PATCH | `/api/v1/month-end-asset-balances/{balanceId}` | 未確定の対象年月に登録された月末資産残高を更新する | `month_end_asset_snapshots`、`month_end_asset_balances` | 6.8 修正、7.9 修正 | [BAL-003](./details/month-end-assets.md#bal-003-月末資産残高更新) |

---

## 11. 商品別月末評価額API

| API ID | 機能分類 | API名 | HTTPメソッド | URL | 概要 | 主な関連テーブル | 対応する機能要件 | 詳細設計書 |
|---|---|---|---|---|---|---|---|---|
| VAL-001 | 商品別月末評価額 | 商品別月末評価額一覧取得 | GET | `/api/v1/month-end-asset-snapshots/{snapshotId}/holding-values` | 指定した月末資産状況に属する商品別評価額一覧を取得する | `month_end_asset_snapshots`、`month_end_holding_values`、`holding_assets` | 6.5 商品単位での登録、6.10 一覧表示 | [VAL-001](./details/month-end-assets.md#val-001-商品別月末評価額一覧取得) |
| VAL-002 | 商品別月末評価額 | 商品別月末評価額登録 | POST | `/api/v1/month-end-asset-snapshots/{snapshotId}/holding-values` | 指定した保有商品の月末評価額を登録する | `month_end_asset_snapshots`、`month_end_holding_values`、`holding_assets` | 6.5 商品単位での登録 | [VAL-002](./details/month-end-assets.md#val-002-商品別月末評価額登録) |
| VAL-003 | 商品別月末評価額 | 商品別月末評価額更新 | PATCH | `/api/v1/month-end-holding-values/{holdingValueId}` | 未確定の対象年月に登録された商品別月末評価額を更新する | `month_end_asset_snapshots`、`month_end_holding_values` | 6.8 修正、7.9 修正 | [VAL-003](./details/month-end-assets.md#val-003-商品別月末評価額更新) |

---

## 12. 資産状況・資産推移API

| API ID | 機能分類 | API名 | HTTPメソッド | URL | 概要 | 主な関連テーブル | 対応する機能要件 | 詳細設計書 |
|---|---|---|---|---|---|---|---|---|
| AST-001 | 資産状況 | 現在資産状況取得 | GET | `/api/v1/asset-summaries/current` | 最新の確定済み対象年月における総資産と内訳を取得する | `month_end_asset_snapshots`、`month_end_asset_balances`、`month_end_holding_values`、`asset_account_available_settings` | 8.2 現在の資産状況、8.3 資産口座別資産状況、8.4 保有商品別資産状況 | [AST-001](./details/asset-views.md#ast-001-現在資産状況取得) |
| AST-002 | 資産状況 | 指定年月資産状況取得 | GET | `/api/v1/asset-summaries/{targetYearMonth}` | 指定した確定済み対象年月の総資産と内訳を取得する | `month_end_asset_snapshots`、`month_end_asset_balances`、`month_end_holding_values`、`asset_account_available_settings` | 8.2〜8.5 | [AST-002](./details/asset-views.md#ast-002-指定年月資産状況取得) |
| AST-003 | 資産推移 | 資産推移取得 | GET | `/api/v1/asset-trends` | 指定期間の総資産、利用可能資産、資産口座別または保有商品別の推移を取得する | `month_end_asset_snapshots`、`month_end_asset_balances`、`month_end_holding_values`、`asset_account_available_settings` | 8.6 前月比較、8.7 資産推移、8.8 表示対象期間 | [AST-003](./details/asset-views.md#ast-003-資産推移取得) |

---

## 13. 目的達成判定API

| API ID | 機能分類 | API名 | HTTPメソッド | URL | 概要 | 主な関連テーブル | 対応する機能要件 | 詳細設計書 |
|---|---|---|---|---|---|---|---|---|
| ASM-001 | 目的達成判定 | 目的達成判定プレビュー | POST | `/api/v1/objectives/{objectiveId}/assessment-previews` | 判定結果を保存せずに目的達成可否と計算根拠を算出する | `objectives`、`month_end_asset_snapshots`、`month_end_asset_balances`、`month_end_holding_values`、`asset_account_available_settings`、`net_incomes` | 11.2〜11.8 | [ASM-001](./details/assessments.md#asm-001-目的達成判定プレビュー) |
| ASM-002 | 目的達成判定 | 目的達成判定結果保存 | POST | `/api/v1/objectives/{objectiveId}/assessments` | 目的達成判定を実行し、判定結果と計算根拠を履歴として保存する | `objectives`、`month_end_asset_snapshots`、`assessment_histories`、`net_incomes` | 11.9 判定結果の保存、11.11 再判定 | [ASM-002](./details/assessments.md#asm-002-目的達成判定結果保存) |
| ASM-003 | 目的達成判定 | 判定履歴一覧取得 | GET | `/api/v1/assessment-histories` | 操作対象利用者の判定履歴を取得する | `assessment_histories`、`objectives`、`month_end_asset_snapshots` | 11.10 判定履歴 | [ASM-003](./details/assessments.md#asm-003-判定履歴一覧取得) |
| ASM-004 | 目的達成判定 | 判定履歴詳細取得 | GET | `/api/v1/assessment-histories/{assessmentHistoryId}` | 指定した判定履歴と判定時点の計算根拠を取得する | `assessment_histories`、`objectives`、`month_end_asset_snapshots` | 11.8 判定結果の表示、11.10 判定履歴 | [ASM-004](./details/assessments.md#asm-004-判定履歴詳細取得) |

---

## 14. CSVインポートAPI

| API ID | 機能分類 | API名 | HTTPメソッド | URL | 概要 | 主な関連テーブル | 対応する機能要件 | 詳細設計書 |
|---|---|---|---|---|---|---|---|---|
| CSV-001 | CSVインポート | 月末資産残高CSVテンプレート取得 | GET | `/api/v1/month-end-asset-balances/csv-template` | 口座単位の月末資産残高CSVテンプレートを取得する | `asset_accounts` | 12.3 CSVテンプレート、12.6 CSVフォーマット | [CSV-001](./details/csv-imports.md#csv-001-月末資産残高csvテンプレート取得) |
| CSV-002 | CSVインポート | 月末資産残高CSVプレビュー | POST | `/api/v1/month-end-asset-balances/imports/preview` | CSVを検証し、保存せずに登録予定内容とエラーを返却する | `asset_accounts`、`month_end_asset_snapshots`、`month_end_asset_balances` | 12.7 入力チェック、12.8 プレビュー | [CSV-002](./details/csv-imports.md#csv-002-月末資産残高csvプレビュー) |
| CSV-003 | CSVインポート | 月末資産残高CSV登録 | POST | `/api/v1/month-end-asset-balances/imports` | 検証済みCSVの月末資産残高をトランザクションで一括登録する | `asset_accounts`、`month_end_asset_snapshots`、`month_end_asset_balances` | 12.9 登録、12.10 エラー、12.13 確定済みデータ | [CSV-003](./details/csv-imports.md#csv-003-月末資産残高csv登録) |
| CSV-004 | CSVインポート | 商品別月末評価額CSVテンプレート取得 | GET | `/api/v1/month-end-holding-values/csv-template` | 商品単位の商品別月末評価額CSVテンプレートを取得する | `asset_accounts`、`holding_assets` | 12.3 CSVテンプレート、12.6 CSVフォーマット | [CSV-004](./details/csv-imports.md#csv-004-商品別月末評価額csvテンプレート取得) |
| CSV-005 | CSVインポート | 商品別月末評価額CSVプレビュー | POST | `/api/v1/month-end-holding-values/imports/preview` | CSVを検証し、保存せずに登録予定内容とエラーを返却する | `asset_accounts`、`holding_assets`、`month_end_asset_snapshots`、`month_end_holding_values` | 12.7 入力チェック、12.8 プレビュー | [CSV-005](./details/csv-imports.md#csv-005-商品別月末評価額csvプレビュー) |
| CSV-006 | CSVインポート | 商品別月末評価額CSV登録 | POST | `/api/v1/month-end-holding-values/imports` | 検証済みCSVの商品別月末評価額をトランザクションで一括登録する | `asset_accounts`、`holding_assets`、`month_end_asset_snapshots`、`month_end_holding_values` | 12.9 登録、12.10 エラー、12.13 確定済みデータ | [CSV-006](./details/csv-imports.md#csv-006-商品別月末評価額csv登録) |

---

## 15. API件数

Phase1で設計対象とするAPIは、
以下のとおりとする。

| 機能分類 | API数 |
|---|---:|
| 利用者 | 1 |
| 資産口座 | 6 |
| 保有商品 | 5 |
| 手取り収入 | 5 |
| 目的 | 6 |
| 月末資産状況 | 5 |
| 月末資産残高 | 3 |
| 商品別月末評価額 | 3 |
| 資産状況・資産推移 | 3 |
| 目的達成判定 | 4 |
| CSVインポート | 6 |
| **合計** | **47** |

---

## 16. 対象外API

Phase1では、
以下のAPIは提供しない。

- 利用者登録API
- ログイン・ログアウトAPI
- パスワード管理API
- 権限管理API
- 物理削除API
- 資産移動履歴API
- クレジットカード管理API
- 外貨換算API
- 金融機関連携API
- 将来資産予測API
- 目的達成可能年月試算API

---

## 17. 設計上の補足

### 17.1 利用者切替

利用者切替専用APIは提供しない。

フロントエンドが選択中の利用者IDを保持し、
利用者依存APIへ`X-User-Id`を付与する。

### 17.2 無効化

資産口座、
保有商品および目的の無効化は、
各リソースの更新APIで状態を変更する。

```http
PATCH /api/v1/asset-accounts/{assetAccountId}
```

```json
{
  "isEnabled": false
}
```

### 17.3 確定・確定解除

月末資産状況の確定および確定解除は、
単純なCRUDでは表現しにくい業務操作であるため、
ユースケースを表すURLを採用する。

```http
POST /api/v1/month-end-asset-snapshots/{snapshotId}/confirm
POST /api/v1/month-end-asset-snapshots/{snapshotId}/unconfirm
```

### 17.4 判定不可

判定に必要な情報が不足している場合は、
判定処理を実行せず、
判定履歴も作成しない。

### 17.5 詳細設計書

詳細設計書の作成前は、
リンク先ファイルまたはアンカーが存在しない場合がある。

API詳細設計の作成時に、
API IDと見出しを一致させる。