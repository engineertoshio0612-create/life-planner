# HLD-002 保有商品登録

## 1. 概要

本ドキュメントでは、HLD-002 保有商品登録APIをLaravelで実装する際のアーキテクチャおよび責務分離方針を定義する。

本APIでは、操作対象となる利用者に帰属する資産口座へ、保有商品を新規登録する。

登録対象となる資産口座は、残高記録単位が「商品単位」であり、かつ利用開始年月時点で有効な資産口座に限定する。

また、利用者境界、資産口座の利用状態、残高記録単位、利用開始年月の整合性および同一資産口座内における保有商品名の重複を確認したうえで、保有商品を登録する。

Laravel実装では、HTTPリクエストの受付、入力値検証、業務ルール判定、データアクセス、レスポンス生成の責務を分離し、以下の構成を基本とする。

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

登録処理およびデータベースの状態に依存する業務ルールはUseCaseを中心に制御し、ActionやForm Requestへ業務ロジックを持たせない。
また、資産口座の取得時には操作対象利用者IDを検索条件へ含め、他利用者に帰属する資産口座へ保有商品を登録できないよう利用者境界を保証する。
正常終了時は、登録した保有商品をAPI ResourceによってAPIレスポンス形式へ変換し、`201 Created`として返却する。


---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、入力値および利用者コンテキストを取得する。

Form Requestまたは入力用DTOを利用して入力値を受け取り、保有商品登録UseCaseを呼び出す。

UseCaseから受け取った処理結果をResponderへ渡す。

業務ルール、重複判定、データ登録処理およびレスポンス生成処理は、Actionへ直接記述しない。

### 2.2 UseCase

保有商品登録のユースケース処理を担当する。

主な処理は以下とする。

- 操作対象利用者の受け取り
- 所属資産口座の取得
- 利用者境界の確認
- 資産口座の利用状態確認
- 残高記録単位の確認
- 利用開始年月の整合性確認
- 同名保有商品の重複確認
- 保有商品の登録
- 登録結果の返却

登録処理は、データベーストランザクション内で実行する。

### 2.3 Form Request / DTO

入力値の形式および単項目バリデーションを担当する。

主な検証対象は以下とする。

- `assetAccountId`
- `name`
- `productType`
- `startYearMonth`
- `memo`

資産口座の利用状態や残高記録単位など、データベースの状態に依存する業務ルールはForm Requestへ記述しない。

### 2.4 Query / Repository

所属資産口座の取得、同名保有商品の存在確認および保有商品の永続化を担当する。

資産口座を取得する場合は、必ず操作対象利用者IDを検索条件へ含める。

```php
AssetAccount::query()
    ->where('id', $assetAccountId)
    ->where('user_id', $userId)
    ->first();
```

論理削除済み資産口座は、通常の取得対象としない。

重複確認では、同一資産口座内の未削除の保有商品のみを対象とする。

### 2.5 Responder

UseCaseから受け取った登録結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`201 Created` とともに登録した保有商品を返却する。

エラー発生時は、共通エラーレスポンス形式へ変換する。

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

### 2.7 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

---

## 3. 関連ドキュメント

- [CSV-001 API詳細設計](../../../api/details/csv-imports/csv-001-template.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [CSVインポート Laravelアーキテクチャ設計](./README.md)
- [CSV-001 テスト設計](../../../tests/csv-imports/csv-001-template.md)
- [CSVインポート テスト設計](../../../tests/csv-imports/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
