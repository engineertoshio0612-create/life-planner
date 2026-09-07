# HLD-003 保有商品詳細取得

## 1. 概要

本ドキュメントでは、HLD-003 保有商品詳細取得APIをLaravelで実装する際のアーキテクチャおよび責務分離方針を定義する。

本APIでは、操作対象となる利用者に帰属する保有商品の詳細情報を取得する。

保有商品の編集画面表示および詳細情報の確認に利用することを想定し、保有商品に加えて、所属する資産口座の情報を取得する。

無効化済みの保有商品も詳細表示の対象とするため、論理削除済みのデータを含めて取得する。また、所属する資産口座が無効化済みの場合についても、保有商品の詳細情報として参照できるよう取得対象とする。

Laravel実装では、HTTPリクエストの受付、利用者境界を考慮したデータ取得、レスポンス生成の責務を分離し、以下の構成を基本とする。

```text
Middleware
    ↓
Action
    ↓
UseCase
    ↓
Query
    ↓
API Resource
    ↓
Responder
```

保有商品の取得時には、操作対象利用者IDおよび保有商品IDを検索条件として利用し、他利用者に帰属する保有商品を取得できないよう利用者境界を保証する。

利用者境界外の保有商品を含め、対象となる保有商品が取得できない場合は、存在しないものとして扱い、`HOLDING_ASSET_NOT_FOUND` に対応する `404 Not Found` を返却する。

正常終了時は、取得した保有商品および所属資産口座の情報をAPI ResourceによってAPIレスポンス形式へ変換し、`200 OK`として返却する。


---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、保有商品IDおよび利用者コンテキストを取得する。

保有商品詳細取得UseCaseを呼び出し、処理結果をResponderへ渡す。

保有商品の検索条件、利用者境界の判定、所属資産口座の取得およびレスポンス生成処理は、Actionへ直接記述しない。

---

### 2.2 UseCase

保有商品詳細取得のユースケース処理を担当する。

主な処理は、以下とする。

- 操作対象利用者を受け取る
- 保有商品IDを受け取る
- 操作対象利用者に帰属する保有商品を取得する
- 所属資産口座を取得する
- 取得結果をResponderへ返却する

利用者境界外の保有商品が指定された場合は、対象が存在しないものとして扱う。

---

### 2.3 Query

操作対象利用者に帰属する保有商品を取得する。

無効化済み保有商品も詳細表示の対象とするため、Laravel SoftDeletesを使用する場合は、`withTrashed()`を利用する。

取得時は、所属資産口座を同時に取得する。

取得例：

```php
HoldingAsset::query()
    ->withTrashed()
    ->with([
        'assetAccount' => fn ($query) => $query->withTrashed(),
    ])
    ->where('id', $holdingAssetId)
    ->where('user_id', $userId)
    ->first();
```

保有商品が取得できない場合は、`HOLDING_ASSET_NOT_FOUND` として扱う。

所属資産口座についても、他利用者のデータが誤って紐づかないよう、リレーションおよび外部キー制約で整合性を保証する。

### 2.4 Responder

UseCaseから受け取った処理結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`200 OK` とともに保有商品詳細を `data` オブジェクトで返却する。

対象が存在しない場合は、共通エラーレスポンス形式で `404 Not Found` を返却する。

### 2.5 API Resource

データベースカラムを直接返却せず、API Resourceを利用してレスポンス形式へ変換する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'assetAccountId' => (string) $this->asset_account_id,
    'assetAccountName' => $this->assetAccount->name,
    'name' => $this->name,
    'productType' => $this->productType->value,
    'startYearMonth' => $this->start_year_month,
    'memo' => $this->memo,
    'isEnabled' => $this->deleted_at === null,
];
```

API Resourceは、Responderから利用する。

### 2.6 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id` を検証し、操作対象利用者を特定する。

---

## 3. 関連ドキュメント
- [HLD-003 API詳細設計](../../../api/details/holding-assets/hld-003-detail.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [保有商品 Laravelアーキテクチャ設計](./README.md)
- [HLD-003 テスト設計](../../../tests/holding-assets/hld-003-detail.md)
- [保有商品 テスト設計](../../../tests/holding-assets/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
