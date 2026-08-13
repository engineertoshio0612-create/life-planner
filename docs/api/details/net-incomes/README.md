# 手取り収入API

## 1. 概要

手取り収入に関するAPIの詳細設計を管理する。

手取り収入APIでは、主に以下を扱う。

- 手取り収入一覧取得
- 手取り収入登録
- 手取り収入詳細取得
- 手取り収入更新
- 平均手取り収入取得

各API固有の仕様は、
それぞれのAPI詳細設計を参照する。

API全体に共通する仕様については、
[API共通方針](../../api-common-policy.md)
を参照する。

---

## 2. API一覧

| API ID | API名 | HTTPメソッド | エンドポイント | 詳細設計 |
|---|---|---|---|---|
| INC-001 | 手取り収入一覧取得 | GET | `/api/v1/net-incomes` | [INC-001](./inc-001-list.md) |
| INC-002 | 手取り収入登録 | POST | `/api/v1/net-incomes` | [INC-002](./inc-002-create.md) |
| INC-003 | 手取り収入詳細取得 | GET | `/api/v1/net-incomes/{netIncomeId}` | [INC-003](./inc-003-detail.md) |
| INC-004 | 手取り収入更新 | PATCH | `/api/v1/net-incomes/{netIncomeId}` | [INC-004](./inc-004-update.md) |
| INC-005 | 平均手取り収入取得 | GET | `/api/v1/net-incomes/average` | [INC-005](./inc-005-average.md) |

---

## 3. APIの責務

### INC-001 手取り収入一覧取得

操作対象利用者の対象年月ごとの手取り収入一覧を取得する。

### INC-002 手取り収入登録

対象年月の手取り収入を登録する。

### INC-003 手取り収入詳細取得

指定した手取り収入の詳細を取得する。

### INC-004 手取り収入更新

手取り収入および備考を更新する。


### INC-005 平均手取り収入取得

指定した判定対象年月以前の連続する3か月の平均手取り収入を取得する。

---

## 4. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [Laravel共通設計](../../../architecture/laravel-architecture.md)
- [React共通設計](../../../architecture/react-architecture.md)