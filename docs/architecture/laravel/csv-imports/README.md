# CSVインポート Laravelアーキテクチャ設計

## 1. 概要

本ディレクトリでは、CSVインポート関連APIをLaravelで実装する際のアーキテクチャおよび責務分離方針を定義する。

対象APIは以下とする。

- CSV-001 月末資産残高CSVテンプレート取得
- CSV-002 月末資産残高CSVプレビュー
- CSV-003 月末資産残高CSV登録
- CSV-004 商品別月末評価額CSVテンプレート取得
- CSV-005 商品別月末評価額CSVプレビュー
- CSV-006 商品別月末評価額CSV登録

CSV関連APIでは、テンプレート取得、プレビュー、登録の責務を明確に分離する。

```text
テンプレート取得
    → CSV入力形式の提供

プレビュー
    → CSV解析・入力値検証・業務ルール検証
    → DB更新なし

登録
    → 最新状態で再検証
    → トランザクション内で一括登録
```

Laravel側では、APIの責務に応じて以下を組み合わせる。

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
    ├─ CSV Definition
    ├─ CSV Parser
    ├─ CSV Validator
    ├─ Import Validator
    ├─ Query
    └─ Repository
    ↓
DTO
    ↓
API Resource
    ↓
Responder
```

CSV DefinitionではヘッダーなどのCSV仕様を定義し、テンプレート・プレビュー・登録で共通利用する。

CSV ParserはCSV構造の解析、CSV ValidatorはCSV各行の入力値検証、Import Validatorは業務データを使用した登録可否判定を担当する。

プレビューAPIと登録APIでは、Parser、Validator、Queryおよび業務ルールを可能な限り共通化し、判定ルールの乖離を防止する。ただし、プレビュー結果は登録可否を保証するものではなく、登録APIでは実行時点の最新DB状態を必ず再検証する。

登録APIでは、すべての検証に成功した場合のみトランザクションを開始し、必要に応じて行ロックを取得する。部分登録は許可せず、途中でエラーが発生した場合はCSV全体をロールバックする。

また、アプリケーション側の重複確認に加えてUNIQUE制約を最終防衛線とし、並行実行時の重複登録を防止する。

## 2. API別設計

- CSV-001 月末資産残高CSVテンプレート取得
- CSV-002 月末資産残高CSVプレビュー
- CSV-003 月末資産残高CSV登録
- CSV-004 商品別月末評価額CSVテンプレート取得
- CSV-005 商品別月末評価額CSVプレビュー
- CSV-006 商品別月末評価額CSV登録

## 3. 関連ドキュメント

- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- CSVインポート テスト設計
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)