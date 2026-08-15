##  ASM-003 目的達成判定履歴詳細取得

### 1 概要

本ドキュメントでは、
ASM-003 目的達成判定履歴詳細取得APIを
Laravelで実装する際の
アーキテクチャおよび責務分離方針を定義する。

ASM-003では、
操作対象利用者の指定された目的に紐づく
`assessment_histories`から、
指定された目的達成判定履歴1件を取得する。

本APIは、
ASM-001によって保存された
過去の目的達成判定履歴を
参照するための読み取り専用APIであり、
目的達成判定そのものは実行しない。

Laravel実装では、
HTTPリクエストの受付から
目的の利用者境界確認、
判定履歴の取得、
APIレスポンスの生成までを
単一のクラスへ集約せず、
各責務を分離する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
Middleware
    ↓
Request
    ↓
Action
    ↓
UseCase
    ├─ ObjectiveQuery
    └─ AssessmentHistoryQuery
    ↓
API Resource
    ↓
Responder
```

---

### 2 Laravel実装方針

ASM-003では、
Action、
Request、
UseCase、
Query、
API Resource、
Responderを分離して実装する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
Middleware
    ↓
Request
    ↓
Action
    ↓
UseCase
    ├─ ObjectiveQuery
    └─ AssessmentHistoryQuery
    ↓
API Resource
    ↓
Responder
```
本APIは、保存済みの目的達成判定履歴1件を参照するだけのAPIである。

目的達成判定の再実行や、現在の資産情報・手取り収入を使用した判定結果の再計算は行わない。

---

### 3 関連ドキュメント

- [ASM-003 API詳細設計](../../../api/details/assessments/asm-003-list.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [目的達成判定 Laravelアーキテクチャ設計](./README.md)
- [ASM-003 Reactアーキテクチャ設計](../../../architecture/react/assessments/asm-003-list.md)
- [ASM-003 テスト設計](../../../tests/assessments/asm-003-list.md)
- [目的達成判定 テスト設計](../../../tests/assessments/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
