# 資産状況 Laravelアーキテクチャ設計

## 1. 概要

本ディレクトリでは、資産状況に関するAPIをLaravelで実装する際のアーキテクチャおよび責務分離方針を定義する。

対象APIは以下とする。

- AST-001 現在資産状況取得
- AST-002 指定年月資産状況取得
- AST-003 資産推移取得

資産状況APIでは、`month_end_asset_snapshots`を基点として、月末資産残高、商品別月末評価額および利用可能資産設定から資産状況を算出する。

資産額は、資産口座の残高記録単位に応じて算出元を切り替える。

```text
口座単位
    → month_end_asset_balances.balance

商品単位
    → month_end_holding_values.value
```

月末資産残高と商品別月末評価額を単純に合算せず、同一資産の二重計上を防止する。

また、利用可能資産は対象年月時点で有効な`asset_account_available_settings`を使用して算出し、過去の資産状況に現在時点の設定を適用しない。

Laravel実装では、主に以下の責務へ分離する。

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
    ↓
Query
    ↓
DTO
    ↓
API Resource
    ↓
Responder
```

ActionはHTTPリクエストの受付とUseCaseの呼び出しに限定し、資産状況の取得・集計に関する業務フローはUseCaseが制御する。

データベース参照はQueryへ分離し、Eloquent ModelやQuery結果をそのままAPIレスポンスとして使用しない。

AST-001・AST-002・AST-003で共通する1ヶ月分の資産状況算出ロジックは可能な限り共通化し、資産口座別集計、総資産集計、利用可能資産集計などを重複実装しない。

AST-003では複数年月を扱うため、対象データを一括取得してアプリケーション上で年月単位に集計し、対象年月数に比例してSQL発行回数が増加するN+1構成を避ける。

各APIは読み取り専用とし、資産状況の算出結果を専用テーブルへ保存しない。

## 2. API別設計

- AST-001 現在資産状況取得
- AST-002 指定年月資産状況取得
- AST-003 資産推移取得

## 3. 関連ドキュメント

- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)