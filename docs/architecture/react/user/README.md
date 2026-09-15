# 利用者API詳細設計

## 1. 概要

利用者に関するAPIの詳細設計を管理する。

利用者APIでは、主に以下を扱う。

- 利用者一覧取得

各API固有の仕様は、
それぞれのAPI詳細設計を参照する。

API全体に共通する仕様については、
[API共通方針](../../api-common-policy.md)
を参照する。

---

## 2. API一覧

| API ID | API名 | HTTPメソッド | エンドポイント | 詳細設計 |
|---|---|---|---|---|
| USR-001 | 利用者一覧取得 | GET | `/api/v1/users` | [AST-001](./usr-001-list.md) |

---

## 3. APIの責務

### AST-001 利用者一覧取得

選択可能な利用者の一覧を取得する。

---

## 4. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [Laravel共通設計](../../../architecture/laravel-architecture.md)
- [React共通設計](../../../architecture/react-architecture.md)