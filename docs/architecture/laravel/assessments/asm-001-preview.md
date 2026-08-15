##  ASM-001 目的達成判定実行

### 1 概要

本ドキュメントでは、
ASM-001 目的達成判定実行APIを
Laravelで実装する際の
アーキテクチャおよび責務分離方針を定義する。

ASM-001では、
操作対象利用者の指定された目的について、
確定済みの月末資産状況、
月末資産残高、
商品別月末評価額、
手取り収入などの
判定に必要な情報を取得し、
目的達成判定を実行する。

判定結果は、
`assessment_histories`へ
目的達成判定履歴として保存する。

判定に必要な情報が不足している場合は、
例外やAPIエラーとして扱うのではなく、
業務上の判定結果である
「判定不可」として履歴を保存する。

Laravel実装では、
HTTPリクエストの受付から
目的達成判定、
判定履歴の保存、
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
    ├─ MonthEndAssetSnapshotQuery
    ├─ AssetAssessmentQuery
    ├─ NetIncomeQuery
    ├─ AssessmentCalculator
    └─ AssessmentHistoryRepository
    ↓
API Resource
    ↓
Responder
```

Actionは、
Requestから受け取った入力と
利用者コンテキストをUseCaseへ引き渡し、
処理結果をResponderへ渡す
HTTP層の調整役に限定する。

UseCaseは、
目的の取得、
最新の確定済み月末資産状況の取得、
判定対象となる資産情報の取得、
手取り収入の取得、
目的達成判定の実行、
判定履歴の保存といった
ASM-001固有の業務フローを制御する。

データ取得については、
用途に応じたQueryへ責務を分離し、
目的達成判定の具体的な計算および
判定ロジックについては
`AssessmentCalculator`へ分離する。

判定結果の永続化は
`AssessmentHistoryRepository`へ委譲し、
UseCaseやActionへ
Eloquentによる保存処理を直接記述しない。

また、
APIレスポンスへの変換は
API Resource、
正常系・異常系を含む
HTTPレスポンスの生成は
Responderへ責務を分離する。

これによりASM-001では、

```text
HTTP制御
業務フロー
データ取得
目的達成判定
永続化
API表現
HTTPレスポンス生成
```

の責務を明確に分離し、
目的達成判定ロジックを
LaravelのHTTP層や
データアクセスの実装詳細へ
依存させない構成とする。

また、
目的達成判定によって変更するデータは、
今回の判定結果として新規作成する
`assessment_histories`に限定する。

目的、
月末資産状況、
月末資産残高、
商品別月末評価額、
手取り収入などの
判定元となる業務データは変更しない。

---

### 27 Laravel実装方針

ASM-001では、Action、Request、UseCase、Query、Repository、目的達成判定ロジック、Model、API Resource、Responderを分離して実装する。

概念的な処理構成は、以下とする。

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
    ├─ MonthEndAssetSnapshotQuery
    ├─ AssetAssessmentQuery
    ├─ NetIncomeQuery
    ├─ AssessmentCalculator
    └─ AssessmentHistoryRepository
    ↓
API Resource
    ↓
Responder
```

目的達成判定に必要な業務フローの制御は、UseCaseへ集約する。

目的達成判定の具体的な計算ロジックは、`AssessmentCalculator`へ分離する。

Actionへ目的達成判定の業務ロジックを直接記述しない。

---

### 3 関連ドキュメント

- [ASM-001 API詳細設計](../../../api/details/assessments/asm-001-preview.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [目的達成判定 Laravelアーキテクチャ設計](./README.md)
- [ASM-001 テスト設計](../../../tests/assessments/asm-001-preview.md)
- [目的達成判定 テスト設計](../../../tests/assessments/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
