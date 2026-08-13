# 保有商品API

## 1. 概要

保有商品に関するAPIの詳細設計を管理する。

保有商品APIでは、主に以下を扱う。

- 保有商品一覧取得
- 保有商品登録
- 保有商品詳細取得
- 保有商品更新
- 保有商品無効化

各API固有の仕様は、
それぞれのAPI詳細設計を参照する。

API全体に共通する仕様については、
[API共通方針](../../api-common-policy.md)
を参照する。

---

## 2. API一覧

| API ID | API名 | HTTPメソッド | エンドポイント | 詳細設計 |
|---|---|---|---|---|
| HLD-001 | 保有商品一覧取得 | GET | `/api/v1/holding-assets` | [HLD-001](./hld-001-list.md) |
| HLD-002 | 保有商品登録 | POST | `/api/v1/holding-assets` | [HLD-002](./hld-002-create.md) |
| HLD-003 | 保有商品詳細取得 | GET | `/api/v1/holding-assets/{holdingAssetId}` | [HLD-003](./hld-003-detail.md) |
| HLD-004 | 保有商品更新 | PATCH | `/api/v1/holding-assets/{holdingAssetId}` | [HLD-004](./hld-004-update.md) |
| HLD-005 | 保有商品無効化 | GET | `/api/v1/holding-assets/{holdingAssetId}/disable` | [HLD-005](./hld-005-disable.md) |

---

## 3. APIの責務

### HLD-001 保有商品一覧取得

操作対象利用者の保有商品一覧を取得する。

### HLD-002 保有商品登録

商品単位で管理する資産口座へ保有商品を登録する。

### HLD-003 保有商品詳細取得

指定した保有商品の詳細を取得する。

### HLD-004 保有商品更新

保有商品名、商品種別および備考を更新する。


### HLD-005 保有商品無効化

指定した保有商品を無効化する。

---

## 4. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [Laravel共通設計](../../../architecture/laravel-architecture.md)
- [React共通設計](../../../architecture/react-architecture.md)