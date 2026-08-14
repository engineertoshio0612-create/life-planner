# CSVインポートAPI詳細設計

## 1. 概要

CSVインポートに関するAPIの詳細設計を管理する。

CSVインポートAPIでは、主に以下を扱う。

- 月末資産残高CSVテンプレート取得
- 月末資産残高CSVプレビュー
- 月末資産残高CSV登録
- 商品別月末評価額CSVテンプレート取得
- 商品別月末評価額CSVプレビュー
- 商品別月末評価額CSV登録

各API固有の仕様は、
それぞれのAPI詳細設計を参照する。

API全体に共通する仕様については、
[API共通方針](../../api-common-policy.md)
を参照する。

---

## 2. API一覧

| API ID | API名 | HTTPメソッド | エンドポイント | 詳細設計 |
|---|---|---|---|---|
| CSV-001 | 月末資産残高CSVテンプレート取得 | GET | `/api/v1/month-end-asset-balances/csv-template` | [CSV-001](./csv-001-template.md) |
| CSV-002 | 月末資産残高CSVプレビュー | POST | `/api/v1/month-end-asset-balances/imports/preview` | [CSV-002](./csv-002-preview.md) |
| CSV-003 | 月末資産残高CSV登録 | POST | `/api/v1/month-end-asset-balances/imports` | [CSV-003](./csv-003-create.md) |
| CSV-004 | 商品別月末評価額CSVテンプレート取得 | PATCH | `/api/v1/month-end-holding-values/csv-template` | [CSV-004](./csv-004-template.md) |
| CSV-005 | 商品別月末評価額CSVプレビュー | PATCH | `/api/v1/month-end-holding-values/imports/preview` | [CSV-005](./csv-005-preview.md) |
| CSV-006 | 商品別月末評価額CSV登録 | PATCH | `/api/v1/month-end-holding-values/imports` | [CSV-006](./csv-006-create.md) |

---

## 3. APIの責務

### CSV-001 月末資産残高CSVテンプレート取得

口座単位の月末資産残高CSVテンプレートを取得する。

### CSV-002 月末資産残高CSVプレビュー

CSVを検証し、保存せずに登録予定内容とエラーを返却する。

### CSV-003 月末資産残高CSV登録

検証済みCSVの月末資産残高をトランザクションで一括登録する。

### CSV-004 商品別月末評価額CSVテンプレート取得

商品単位の商品別月末評価額CSVテンプレートを取得する。

### CSV-005 商品別月末評価額CSVプレビュー

CSVを検証し、保存せずに登録予定内容とエラーを返却する。

### CSV-006 商品別月末評価額CSV登録

検証済みCSVの商品別月末評価額をトランザクションで一括登録する。

---

## 4. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [Laravel共通設計](../../../architecture/laravel-architecture.md)
- [React共通設計](../../../architecture/react-architecture.md)