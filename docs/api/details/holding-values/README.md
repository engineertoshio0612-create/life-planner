# 商品別月末評価額API

## 1. 概要

商品別月末評価額に関するAPIの詳細設計を管理する。

商品別月末評価額APIでは、主に以下を扱う。

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
| VAL-001 | 商品別月末評価額一覧取得 | GET | `/api/v1/month-end-asset-snapshots/{snapshotId}/holding-values` | [VAL-001](./val-001-list.md) |
| VAL-002 | 商品別月末評価額登録 | POST | `/api/v1/month-end-asset-snapshots/{snapshotId}/holding-values` | [VAL-002](./val-002-create.md) |
| VAL-003 | 商品別月末評価額更新 | PATCH | `/api/v1/month-end-asset-snapshots/{snapshotId}/holding-values/{holdingAssetId}` | [VAL-003](./val-003-update.md) |

---

## 3. APIの責務

### VAL-001 商品別月末評価額一覧取得

操作対象利用者の指定した月末資産状況に属する商品別評価額一覧を取得する。

### VAL-002 商品別月末評価額

指定した保有商品の月末評価額を登録する。

### VAL-003 商品別月末評価額

未確定の対象年月に登録された商品別月末評価額を更新する。

---

## 4. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [Laravel共通設計](../../../architecture/laravel-architecture.md)
- [React共通設計](../../../architecture/react-architecture.md)