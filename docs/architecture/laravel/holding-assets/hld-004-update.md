# HLD-004 保有商品更新

## 1. 概要

本ドキュメントでは、HLD-004 保有商品更新APIをLaravelで実装する際のアーキテクチャおよび責務分離方針を定義する。

本APIでは、操作対象となる利用者に帰属する保有商品の更新可能な情報を変更する。

更新対象は、保有商品名、商品種別および備考とし、所属資産口座、利用開始年月および利用状態は変更対象としない。利用状態の変更については、HLD-005 保有商品無効化APIの責務とする。

更新時には、操作対象利用者に帰属する保有商品であること、対象が利用可能な状態であること、および同一資産口座内に同名の保有商品が存在しないことを確認したうえで、リクエストで指定された項目のみを更新する。

Laravel実装では、HTTPリクエストの受付、入力値検証、業務ルール判定、データアクセス、更新処理およびレスポンス生成の責務を分離し、以下の構成を基本とする。

```text
Middleware
    ↓
Action
    ↓
Form Request / DTO
    ↓
UseCase
    ↓
Query / Repository
    ↓
API Resource
    ↓
Responder
```

更新可能な項目および入力形式の検証はForm RequestまたはDTOで行い、保有商品の利用状態や同名商品の重複など、データベースの状態に依存する業務ルールはUseCaseを中心に制御する。

また、更新対象となる保有商品の取得時には操作対象利用者IDを検索条件へ含め、他利用者に帰属する保有商品を更新できないよう利用者境界を保証する。

更新処理はデータベーストランザクション内で実行し、正常終了時は更新後の保有商品をAPI ResourceによってAPIレスポンス形式へ変換し、`200 OK`として返却する。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
保有商品ID、
更新内容および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOを利用して
更新内容を受け取り、
保有商品更新UseCaseを呼び出す。

UseCaseから受け取った処理結果を、
Responderへ渡す。

業務ルール、
利用者境界の判定、
重複判定、
更新処理および
レスポンス生成処理は、
Actionへ直接記述しない。

---

### 2.2 UseCase

保有商品更新の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者の受け取り
- 保有商品IDの受け取り
- 更新内容の受け取り
- 更新対象保有商品の取得
- 利用者境界の確認
- 保有商品の利用状態確認
- 更新対象外項目の確認
- 保有商品名の重複確認
- 指定された項目のみ更新
- 更新結果の返却

更新処理は、
データベーストランザクション内で実行する。

---

### 2.3 Form Request / DTO

入力値の形式および
単項目バリデーションを担当する。

主な検証対象は、
以下とする。

- `name`
- `productType`
- `memo`

更新可能な項目が
1つ以上指定されていることを検証する。

以下の更新対象外項目が含まれている場合は、
バリデーションエラーとする。

- `assetAccountId`
- `startYearMonth`
- `isEnabled`
- `id`
- `userId`
- `createdAt`
- `updatedAt`
- `deletedAt`

保有商品の利用状態や
商品名の重複など、
データベースの状態に依存する業務ルールは、
Form Requestへ記述しない。

---

### 2.4 Query / Repository

更新対象保有商品の取得、
同名保有商品の存在確認および
保有商品の更新を担当する。

保有商品を取得する場合は、
必ず操作対象利用者IDを検索条件へ含める。

```php
HoldingAsset::query()
    ->where('id', $holdingAssetId)
    ->where('user_id', $userId)
    ->first();
```

通常の取得では、論理削除済み保有商品を対象外とする。

無効化済み保有商品かどうかを独自エラーとして判定する必要がある場合は、`withTrashed()` で取得した上で利用状態を確認する方法を採用できる。

重複確認では、以下の条件を使用する。

```php
HoldingAsset::query()
    ->where('asset_account_id', $holdingAsset->asset_account_id)
    ->where('name', $newName)
    ->whereKeyNot($holdingAsset->id)
    ->exists();
```

Laravel SoftDeletesを使用する場合、通常のクエリでは論理削除済み保有商品が重複確認から除外される。

更新時は、リクエストで指定された項目のみ変更する。

### 2.5 Responder

UseCaseから受け取った更新結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`200 OK` とともに更新後の保有商品を返却する。

対象が存在しない場合、無効化済みの場合、重複している場合またはバリデーションエラーの場合は、共通エラーレスポンス形式へ変換する。

### 2.6 API Resource

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

更新後のレスポンス生成時は、所属資産口座を取得済みであることを保証する。

### 2.7 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id` を検証し、操作対象利用者を特定する。

---

## 3. 関連ドキュメント

- [HLD-004 API詳細設計](../../../api/details/holding-assets/hld-004-update.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [保有商品 Laravelアーキテクチャ設計](./README.md)
- [HLD-004 テスト設計](../../../tests/holding-assets/hld-004-update.md)
- [保有商品 テスト設計](../../../tests/holding-assets/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
