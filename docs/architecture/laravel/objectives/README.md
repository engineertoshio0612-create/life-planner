# 手取り収入API

## 1. 概要

手取り収入に関するAPIの詳細設計を管理する。

手取り収入APIでは、主に以下を扱う。

- 目的一覧取得
- 目的登録
- 目的詳細取得
- 目的更新
- 目的無効化
- 目的達成判定

各API固有の仕様は、
それぞれのAPI詳細設計を参照する。

API全体に共通する仕様については、
[API共通方針](../../api-common-policy.md)
を参照する。

---

## 2. API一覧

| API ID | API名 | HTTPメソッド | エンドポイント | 詳細設計 |
|---|---|---|---|---|
| OBJ-001 | 目的一覧取得 | GET | `/api/v1/objectives` | [OBJ-001](./obj-001-list.md) |
| OBJ-002 | 目的登録 | POST | `/api/v1/objectives` | [OBJ-002](./obj-002-create.md) |
| OBJ-003 | 目的詳細取得 | GET | `/api/v1/objectives/{netIncomeId}` | [OBJ-003](./obj-003-detail.md) |
| OBJ-004 | 目的更新 | PATCH | `/api/v1/objectives/{netIncomeId}` | [OBJ-004](./obj-004-update.md) |
| OBJ-005 | 目的無効化 | PATCH | `/api/v1/objectives/disabled` | [OBJ-005](./obj-005-disabled.md) |
| OBJ-006 | 目的達成判定 | GET | `/api/v1/objectives/assessments` | [OBJ-006](./obj-006-assessments.md) |


---

## 3. APIの責務

### OBJ-001 目的一覧取得

操作対象利用者の目的一覧を取得する。

### OBJ-002 目的登録

目的名、実施予定年月、必要支出額などを登録する。

### OBJ-003 目的詳細取得

指定した目的の詳細を取得する。

### OBJ-004 目的更新

目的情報を更新する。

### OBJ-005 目的無効化

指定した目的を無効化する。

### OBJ-006 目的達成判定

指定した目的の達成可否を判定し、判定履歴を登録する。

---

## 4. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [Laravel共通設計](../../../architecture/laravel-architecture.md)
- [React共通設計](../../../architecture/react-architecture.md)