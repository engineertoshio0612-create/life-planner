# ACC-001 資産口座一覧取得

## 1. 概要

本ドキュメントでは、
ACC-001 資産口座一覧取得APIを
Laravelで実装する際の
アーキテクチャおよび責務分離方針を定義する。

ACC-001では、
操作対象利用者に帰属する資産口座を取得し、
利用中の資産口座だけでなく、
無効化された資産口座も含めて一覧として返却する。

各資産口座について、
資産種別、残高記録単位、
現在有効な利用可能資産設定、
および利用状態を取得する。

Laravel実装では、
HTTPリクエストの受付から
利用者コンテキストの取得、
資産口座および利用可能資産設定の検索、
並び替え、APIレスポンスの生成までを
単一のクラスへ集約せず、
各責務を分離する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ↓
Query
    ↓
Model / Database
    ↓
UseCase
    ↓
Responder
    ↓
API Resource
    ↓
HTTP Response
```

Actionは、
HTTPリクエストおよび利用者コンテキストを受け取り、
UseCaseの呼び出しを担当する。

UseCaseは、
資産口座一覧取得に必要な処理を統括し、
Queryを利用して操作対象利用者に帰属する
資産口座および現在有効な利用可能資産設定を取得する。

Queryは、
利用者境界を考慮した検索処理を担当し、
無効化された資産口座についても
取得対象に含める。

ResponderおよびAPI Resourceは、
UseCaseから受け取った処理結果を
API共通方針に従ったレスポンス形式へ変換する。

これにより、
HTTP層、ユースケース、データ取得、
レスポンス生成の責務を明確に分離し、
各処理の変更影響を局所化するとともに、
テスト容易性および保守性を確保する。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
入力値および利用者コンテキストを取得する。

必要なUseCaseを呼び出し、
処理結果をResponderへ渡す。

業務ルール、
検索条件、
並び替え、
利用可能資産区分の判定および
レスポンス生成処理は、
Actionへ直接記述しない。

### 2.2 UseCase

資産口座一覧取得の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者の資産口座を取得する
- API実行時点で有効な利用可能資産設定を取得する
- 利用中・無効化済み・資産口座名の順で並び替える
- Responderへ返却するデータを生成する

### 2.3 Query

操作対象利用者に帰属する
資産口座一覧を取得する。

利用可能資産設定は、
API実行時点で有効な設定のみ取得する。

無効化済み資産口座も一覧表示するため、
Laravel SoftDeletesを使用する場合は、
`withTrashed()`を利用する。

取得例：

```php
AssetAccount::query()
    ->withTrashed()
    ->where('user_id', $userId)
    ->with('availableSettings')
    ->orderByRaw('deleted_at IS NULL DESC')
    ->orderBy('name')
    ->get();
```

実際の実装では、
対象年月時点で有効な利用可能資産設定のみ取得し、
不要な履歴は読み込まない。

### 2.4 Responder

UseCaseから受け取った処理結果を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`200 OK`とともに
資産口座一覧を`data`配列で返却する。

資産口座が存在しない場合も、
`200 OK`と空配列を返却する。

```json
{
  "data": []
}
```

エラー発生時は、
共通エラーレスポンス形式に従って返却する。

### 2.5 API Resource

データベースカラムを直接返却せず、
API Resourceを利用して
APIレスポンス形式へ変換する。

資産口座IDは、
API共通方針に従い
文字列として返却する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'name' => $this->name,
    'assetType' => $this->assetType,
    'balanceRecordingUnit' => $this->balanceRecordingUnit,
    'isAvailable' => $this->isAvailable,
    'startYearMonth' => $this->startYearMonth,
    'isEnabled' => $this->deleted_at === null,
];
```

API Resourceは、
Responderから利用する。

### 2.6 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキストは、
`X-User-Id`を検証し、
操作対象利用者を特定する。

存在しない利用者、
または論理削除済み利用者が指定された場合は、
処理を終了し、
共通エラーレスポンスを返却する。

---

### 3 関連ドキュメント

- [ACC-001 API詳細設計](../../../api/details/asset-accounts/acc-001-list.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [資産口座 Laravelアーキテクチャ設計](./README.md)
- [ACC-001 テスト設計](../../../tests/asset-accounts/acc-001-list.md)
- [資産口座 テスト設計](../../../tests/asset-accounts/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
