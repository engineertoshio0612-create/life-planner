# 資産口座API

## 1. 概要

資産口座管理に関するAPIの詳細設計を管理する。

資産口座APIでは、主に以下を扱う。

- 資産口座の一覧取得
- 資産口座の登録
- 資産口座の詳細取得
- 資産口座の更新
- 利用可能資産設定履歴の取得
- 利用可能資産設定の登録

各API固有の仕様は、
それぞれのAPI詳細設計を参照する。

API全体に共通する仕様については、
[API共通方針](../../api-common-policy.md)
を参照する。

---

## 2. API一覧

| API ID | API名 | HTTPメソッド | エンドポイント | 詳細設計 |
|---|---|---|---|---|
| ACC-001 | 資産口座一覧取得 | GET | `/api/v1/asset-accounts` | [ACC-001](./acc-001-list.md) |
| ACC-002 | 資産口座登録 | POST | `/api/v1/asset-accounts` | [ACC-002](./acc-002-create.md) |
| ACC-003 | 資産口座詳細取得 | GET | `/api/v1/asset-accounts/{assetAccountId}` | [ACC-003](./acc-003-detail.md) |
| ACC-004 | 資産口座更新 | PATCH | `/api/v1/asset-accounts/{assetAccountId}` | [ACC-004](./acc-004-update.md) |
| ACC-005 | 利用可能資産設定履歴取得 | GET | `/api/v1/asset-accounts/{assetAccountId}/available-settings` | [ACC-005](./acc-005-available-setting-history.md) |
| ACC-006 | 利用可能資産設定登録 | POST | `/api/v1/asset-accounts/{assetAccountId}/available-settings` | [ACC-006](./acc-006-create-available-setting.md) |

---

## 3. APIの責務

### ACC-001 資産口座一覧取得

利用者に帰属する資産口座の一覧を取得する。

### ACC-002 資産口座登録

利用者に帰属する資産口座を新規登録する。

### ACC-003 資産口座詳細取得

指定された資産口座の詳細情報を取得する。

### ACC-004 資産口座更新

指定された資産口座の通常属性を更新する。

利用可能資産区分の変更は扱わない。

### ACC-005 利用可能資産設定履歴取得

指定された資産口座の
利用可能資産設定履歴を取得する。

### ACC-006 利用可能資産設定登録

指定された資産口座に対して、
新しい利用可能資産設定を登録する。

---

## 4. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [Laravel共通設計](../../../architecture/laravel-architecture.md)
- [React共通設計](../../../architecture/react-architecture.md)