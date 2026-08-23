# 利用者API詳細設計

## 1. 概要

本書では、
利用者APIに関する詳細仕様を定義する。

Phase1では、
あらかじめ登録された利用者の一覧取得のみを提供する。

利用者の切替は、
フロントエンドが選択中の利用者IDを保持し、
利用者依存APIのリクエストヘッダーへ付与することで実現する。

利用者切替専用APIは提供しない。

---

## 2. 対象API

| API ID | API名 | HTTPメソッド | URL |
|---|---|---|---|
| USR-001 | 利用者一覧取得 | GET | `/api/v1/users` |

---

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)
- [Laravel CSVインポート設計](../../../architecture/laravel/csv-imports.md)
- [React CSVインポート設計](../../../architecture/react/csv-imports.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)