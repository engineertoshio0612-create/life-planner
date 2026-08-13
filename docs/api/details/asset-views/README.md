# 資産状況・資産推移API詳細設計

## 1. 概要

資産状況・資産推移に関するAPIの詳細設計を管理する。

資産状況・資産推移APIでは、主に以下を扱う。

- 現在資産状況取得
- 指定年月資産状況取得
- 資産推移取得

各API固有の仕様は、
それぞれのAPI詳細設計を参照する。

API全体に共通する仕様については、
[API共通方針](../../api-common-policy.md)
を参照する。

---

## 2. API一覧

| API ID | API名 | HTTPメソッド | エンドポイント | 詳細設計 |
|---|---|---|---|---|
| AST-001 | 現在資産状況取得 | GET | `/api/v1/asset-summaries/current` | [AST-001](./ast-001-list.md) |
| AST-002 | 指定年月資産状況取得 | POST | `/api/v1/asset-summaries/{targetYearMonth}` | [AST-002](./ast-002-create.md) |
| AST-003 | 資産推移取得 | PATCH | `/api/v1/asset-trends` | [AST-003](./ast-003-update.md) |

---

## 3. APIの責務

### AST-001 現在資産状況取得

判定結果を保存せずに目的達成可否と計算根拠を算出する。

### AST-002 指定年月資産状況取得

指定した確定済み対象年月の総資産と内訳を取得する。

### AST-003 資産推移取得

最新の確定済み月末資産状況を未確定へ戻す。

---

## 4. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [Laravel共通設計](../../../architecture/laravel-architecture.md)
- [React共通設計](../../../architecture/react-architecture.md)