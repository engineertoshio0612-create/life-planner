# 月末資産残高API

## 1. 概要

月末資産残高に関するAPIの詳細設計を管理する。

月末資産残高APIでは、主に以下を扱う。

- 商品別月末評価額一覧取得
- 商品別月末評価額登録
- 商品別月末評価額更新

各API固有の仕様は、
それぞれのAPI詳細設計を参照する。

API全体に共通する仕様については、
[API共通方針](../../api-common-policy.md)
を参照する。

---

## 2. API一覧

| API ID | API名 | HTTPメソッド | エンドポイント | 詳細設計 |
|---|---|---|---|---|
| BAL-001 | 商品別月末評価額一覧取得 | GET | `/api/v1/month-end-asset-snapshots/{snapshotId}/holding-values` | [BAL-001](./bal-001-list.md) |
| BAL-002 | 商品別月末評価額登録 | POST | `/api/v1/month-end-asset-snapshots/{snapshotId}/holding-values` | [BAL-002](./bal-002-create.md) |
| BAL-003 | 月末資産残高更新 | PATCH | `/api/v1/month-end-asset-snapshots/{snapshotId}/holding-values/{holdingAssetId}` | [BAL-003](./bal-003-update.md) |

---

## 3. APIの責務

### BAL-001 商品別月末評価額一覧取得

操作対象利用者の指定した月末資産状況に属する口座単位の残高一覧を取得する。

### BAL-002 商品別月末評価額

指定した資産口座の月末残高を登録する。

### BAL-003 商品別月末評価額

未確定の対象年月に登録された月末資産残高を更新する。

---

## 4. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [Laravel共通設計](../../../architecture/laravel-architecture.md)
- [React共通設計](../../../architecture/react-architecture.md)