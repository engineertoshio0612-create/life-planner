# 目的達成判定API詳細設計

## 1. 概要

目的達成判定に関するAPIの詳細設計を管理する。

目的達成判定APIでは、主に以下を扱う。

- 目的達成判定プレビュー
- 目的達成判定結果保存
- 判定履歴一覧取得
- 判定履歴詳細取得

各API固有の仕様は、
それぞれのAPI詳細設計を参照する。

API全体に共通する仕様については、
[API共通方針](../../api-common-policy.md)
を参照する。

---

## 2. API一覧

| API ID | API名 | HTTPメソッド | エンドポイント | 詳細設計 |
|---|---|---|---|---|
| ASM-001 | 目的達成判定プレビュー | GET | `/api/v1/objectives/{objectiveId}/assessment-previews` | [ASM-001](./asm-001-preview.md) |
| ASM-002 | 目的達成判定結果保存 | POST | `/api/v1/objectives/{objectiveId}/assessments` | [ASM-002](./asm-002-create.md) |
| ASM-003 | 判定履歴一覧取得 | PATCH | `/api/v1/assessment-histories` | [ASM-003](./asm-003-list.md) |
| ASM-004 | 判定履歴詳細取得 | PATCH | `/api/v1/assessment-histories/{assessmentHistoryId}` | [ASM-004](./asm-004-detail.md) |

---

## 3. APIの責務

### ASM-001 現在資産状況取得

判定結果を保存せずに目的達成可否と計算根拠を算出する。

### ASM-002 指定年月資産状況取得

指定した確定済み対象年月の総資産と内訳を取得する。

### ASM-003 資産推移取得

最新の確定済み月末資産状況を未確定へ戻す。

### ASM-004 資産推移取得

最新の確定済み月末資産状況を未確定へ戻す。

---

## 4. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [Laravel共通設計](../../../architecture/laravel-architecture.md)
- [React共通設計](../../../architecture/react-architecture.md)