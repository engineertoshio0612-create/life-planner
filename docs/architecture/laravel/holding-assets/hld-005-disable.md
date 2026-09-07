# HLD-005 保有商品無効化

## 1. 概要

本ドキュメントでは、HLD-005 保有商品無効化APIをLaravelで実装する際のアーキテクチャおよび責務分離方針を定義する。

本APIでは、操作対象となる利用者に帰属する利用中の保有商品を無効化する。

保有商品の無効化は物理削除ではなく、LaravelのSoftDeletesを利用して`holding_assets.deleted_at`を設定する論理削除として実現する。

無効化された保有商品は、無効化後の対象年月における商品別月末評価額の登録対象から除外する。一方で、過去の商品別月末評価額、月末資産状況および目的達成判定履歴は履歴情報として保持し、本APIによる更新・削除の対象としない。

また、保有商品の通常属性を更新するHLD-004 保有商品更新とは異なる業務操作として扱い、ルート、ActionおよびUseCaseを分離する。

Laravel実装では、HTTPリクエストの受付、利用者境界を考慮した対象取得、無効化処理、結果生成およびレスポンス生成の責務を分離し、以下の構成を基本とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ HoldingAssetQuery
    └─ HoldingAssetRepository
    ↓
Disable Result DTO
    ↓
API Resource
    ↓
Responder
```

無効化対象の取得時には、保有商品ID、操作対象利用者および有効状態を検索条件へ含め、他利用者に帰属する保有商品や無効化済み保有商品を操作できないよう利用者境界と状態条件を保証する。

無効化処理は`HoldingAssetRepository`を通じてLaravel標準の`delete()`を実行し、`forceDelete()`による物理削除や`deleted_at`の直接更新は行わない。

Phase1では更新対象が`holding_assets`の1レコードのみであるため、HLD-005専用の明示的なトランザクションおよび行ロックは使用しない。

正常終了時は、無効化結果を専用DTOおよびAPI ResourceによってAPIレスポンス形式へ変換し、`200 OK`として返却する。

---

## 2. Laravel実装方針

HLD-005では、Action、UseCase、Query、Repository、DTO、API Resource、Responderを分離して実装する。

概念的な処理構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ HoldingAssetQuery
    └─ HoldingAssetRepository
    ↓
Disable Result DTO
    ↓
API Resource
    ↓
Responder
```

本APIは、リクエストボディを使用しないため、HLD-005専用のFormRequestは作成しない。

また、更新対象は

```text
holding_assets.deleted_at
```

のみとし、HLD-004で扱う通常更新処理とは分離する。

---

### 2.1 Route

HLD-005は、以下のルートとして定義する。

概念例：

```php
Route::patch(
    '/api/v1/holding-assets/{holdingAssetId}/disable',
    DisableHoldingAssetAction::class,
);
```

HLD-004は、

```php
Route::patch(
    '/api/v1/holding-assets/{holdingAssetId}',
    UpdateHoldingAssetAction::class,
);
```

とし、同一HTTPメソッドでもURLを分離する。

これにより、

```text
通常更新
    → HLD-004

無効化
    → HLD-005
```

という責務をルーティングレベルでも明確にする。

---

### 2.2 Action

Actionは、パスパラメータの

```text
holdingAssetId
```

と、検証済みの利用者コンテキストを取得し、UseCaseを呼び出す。

概念例：

```php
final class DisableHoldingAssetAction
{
    public function __invoke(
        string $holdingAssetId,
        DisableHoldingAssetUseCase $useCase,
        DisableHoldingAssetResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                holdingAssetId:
                    $holdingAssetId,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

Actionでは、以下を行わない。

* 利用者存在確認
* `holdingAssetId`の業務的な存在確認
* 保有商品検索SQLの組み立て
* 利用者境界判定
* `deleted_at`更新
* SoftDeletes実行
* レスポンス配列生成
* JSON変換

Actionは、UseCase呼び出しとResponderへの受け渡しに責務を限定する。

---

### 2.3 FormRequest

HLD-005専用のFormRequestは作成しない。

本APIでは、

```text
パスパラメータ
    holdingAssetId

リクエストボディ
    なし
```

であり、HLD-005固有のボディ入力値が存在しないためである。

以下のような空のFormRequestは作成しない。

```php
final class DisableHoldingAssetRequest
    extends FormRequest
{
}
```

不要なクラスを追加しない。

---

### 2.4 パスパラメータ検証

`holdingAssetId`の形式検証は、ルート制約または共通のパスパラメータ検証方式で行う。

概念例：

```php
->whereNumber(
    'holdingAssetId',
);
```

ただし、`0`や負数を有効なIDとして扱わない。

共通方針として正の整数IDのみを許可する場合は、そのルールへ統一する。

形式不正の場合は、

```text
INVALID_HOLDING_ASSET_ID
```

へ変換する。

---

### 2.5 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* 共通例外処理
* ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、以下とする。

```text
X-User-Id取得
    ↓
必須チェック
    ↓
形式チェック
    ↓
users存在確認
    ↓
UserContext設定
    ↓
Action
```

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 2.6 UseCase

HLD-005の業務処理全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `holdingAssetId`を受け取る
3. 操作対象利用者に属する有効な保有商品を取得する
4. 存在しない場合は業務例外とする
5. Repositoryへ無効化処理を委譲する
6. Disable Result DTOを返す

概念的には、以下とする。

```text
userId
+
holdingAssetId
    ↓
HoldingAssetQuery
    ↓
対象保有商品取得
    ↓
存在確認
    ↓
HoldingAssetRepository
    ↓
論理削除
    ↓
Disable Result DTO
```

UseCaseでは、HTTPレスポンスを生成しない。

---

### 2.7 HoldingAssetQuery

無効化対象となる有効な保有商品を取得するために使用する。

概念例：

```php
$holdingAsset =
    $this->holdingAssetQuery
        ->findActiveByIdAndUser(
            holdingAssetId:
                $holdingAssetId,

            userId:
                $userId,
        );
```

概念的な検索条件は、以下とする。

```text
holding_assets.id
    = holdingAssetId

AND

holding_assets.deleted_at
    IS NULL

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 2.8 利用者境界をQueryへ含める

以下のように、保有商品だけをIDで取得した後に別処理で利用者判定する方式は基本としない。

```php
$holdingAsset =
    HoldingAsset::find(
        $holdingAssetId,
    );
```

対象取得時点から、

```text
holdingAssetId
+
userId
+
有効状態
```

を条件へ含める。

これにより、他利用者の保有商品を誤って更新することを防止する。

---

### 2.9 EloquentのSoftDeletes

`HoldingAsset` Modelでは、Laravelの

```php
use SoftDeletes;
```

を使用する。

概念例：

```php
final class HoldingAsset
    extends Model
{
    use SoftDeletes;
}
```

HLD-005では、物理DELETEではなくSoftDeletesによって無効化する。

---

### 2.10 QueryではwithTrashedを使用しない

HLD-005の通常取得では、

```php
withTrashed()
```

を使用しない。

論理削除済み保有商品は、無効化対象として取得しない。

そのため、

```text
deleted_at IS NULL
```

の通常スコープを利用する。

---

### 2.11 無効化済み保有商品の扱い

論理削除済みの保有商品を指定した場合は、Query結果が`null`となる。

UseCaseでは、

```php
if ($holdingAsset === null) {
    throw new HoldingAssetNotFoundException();
}
```

のように扱う。

再度`delete()`を実行しない。

これにより、最初に設定された

```text
deleted_at
```

を保持する。

---

### 2.12 HoldingAssetRepository

保有商品の更新処理はRepositoryへ委譲する。

HLD-005では、無効化専用メソッドを用意する。

概念例：

```php
interface HoldingAssetRepository
{
    public function disable(
        HoldingAsset $holdingAsset,
    ): void;
}
```

実装例：

```php
final class EloquentHoldingAssetRepository
    implements HoldingAssetRepository
{
    public function disable(
        HoldingAsset $holdingAsset,
    ): void {
        $holdingAsset->delete();
    }
}
```

Repositoryでは、業務上の対象判定は行わない。

対象取得・利用者境界確認は、UseCaseおよびQuery側で完了していることを前提とする。

---

### 2.13 delete()を使用する

SoftDeletesを利用するため、HLD-005では概念的に以下を使用する。

```php
$holdingAsset->delete();
```

以下のように`deleted_at`を直接更新する実装は、基本としない。

```php
$holdingAsset->update([
    'deleted_at' => now(),
]);
```

Laravel標準のSoftDeletes動作へ統一する。

---

### 2.14 forceDelete()を使用しない

HLD-005では、以下を使用しない。

```php
$holdingAsset->forceDelete();
```

物理削除すると、過去の

```text
month_end_holding_values
```

との関連が失われる可能性があるためである。

---

### 2.15 HLD-004の更新処理を流用しない

HLD-005では、HLD-004の更新UseCaseへ

```text
isEnabled = false
```

のような値を渡して無効化する方式は採用しない。

概念的には、

```text
HLD-004
UpdateHoldingAssetUseCase

HLD-005
DisableHoldingAssetUseCase
```

として分離する。

これにより、

```text
通常属性更新
無効化
```

の業務操作を明確に区別する。

---

### 2.16 Mass Assignmentを使用しない

HLD-005ではリクエストボディが存在しないため、以下のような処理を行わない。

```php
$holdingAsset->update(
    $request->all(),
);
```

無効化処理は、

```php
$holdingAsset->delete();
```

と明示する。

---

### 2.17 過去の商品別月末評価額を操作しない

RepositoryまたはUseCaseから、

```text
month_end_holding_values
```

を更新・削除しない。

以下のようなcascade的な業務処理は行わない。

```text
HoldingAsset無効化
    ↓
過去MonthEndHoldingValue削除
```

過去データは保持する。

---

### 2.18 snapshotを操作しない

HLD-005では、

```text
month_end_asset_snapshots
```

を更新しない。

特に、

```text
confirmed
confirmed_at
```

を変更しない。

保有商品の無効化を理由として、確定済みsnapshotを未確定へ戻さない。

---

### 2.19 トランザクション

Phase1では、HLD-005専用の明示的な

```php
DB::transaction()
```

は使用しない。

更新対象が

```text
holding_assets 1件
```

だけであり、SoftDeletesの単一UPDATEで完結するためである。

概念的には、

```text
SELECT
    ↓
UPDATE deleted_at
```

となる。

---

### 2.20 トランザクションを導入しない理由

HLD-005でトランザクションを導入しても、

```text
複数テーブル更新
複数レコード更新
```

が存在しないため、Phase1では得られる利点が小さい。

不要な以下を増やさない。

* トランザクション境界
* ロック保持
* 実装複雑性

将来的に複数テーブルを更新する場合は、改めて導入を検討する。

---

### 2.21 ロック

Phase1では、HLD-005専用の`lockForUpdate()`は使用しない。

以下のような明示的な行ロックは基本としない。

```php
HoldingAsset::query()
    ->whereKey(
        $holdingAssetId,
    )
    ->lockForUpdate()
    ->first();
```

単一レコードの通常の論理削除として扱う。

---

### 2.22 HLD-004との同時実行

HLD-004とHLD-005が同じ保有商品へ同時実行される可能性はある。

Phase1では、以下は導入しない。

* `version`カラム
* 楽観ロック
* ETag
* If-Match

必要になった時点で排他制御方針を別途検討する。

---

### 2.23 Disable Result DTO

無効化結果は、専用DTOとして表現する。

概念例：

```php
final readonly class DisableHoldingAssetResult
{
    public function __construct(
        public int $id,
    ) {
    }
}
```

`disabled`はHLD-005成功時に必ず`true`となるため、DTOへ保持せずResource側で固定値として設定してもよい。

または、レスポンス契約をDTOへ明示的に持たせる場合は、

```php
final readonly class DisableHoldingAssetResult
{
    public function __construct(
        public int $id,
        public bool $disabled,
    ) {
    }
}
```

としてもよい。

Phase1では、どちらかに統一する。

---

### 2.24 UseCaseの戻り値

概念例：

```php
return new DisableHoldingAssetResult(
    id:
        $holdingAsset->id,
);
```

Eloquent ModelそのものをActionやResponderへ返却しない。

---

### 2.25 API Resource

Disable Result DTOを、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php
final class DisableHoldingAssetResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'disabled'
                => true,
        ];
    }
}
```

主キーは、API共通方針に従ってstringへ変換する。

---

### 2.26 Resourceで返却しない情報

HLD-005のResourceでは、以下を返却しない。

* `asset_account_id`
* `name`
* `product_type`
* `start_year_month`
* `memo`
* `created_at`
* `updated_at`
* `deleted_at`

Eloquent Modelを

```php
return $holdingAsset->toArray();
```

のようにそのまま返却しない。

---

### 2.27 Responder

Responderは、Disable Result DTOをAPI共通の成功Envelopeへ変換する。

概念例：

```php
final class DisableHoldingAssetResponder
{
    public function ok(
        DisableHoldingAssetResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new DisableHoldingAssetResource(
                        $result,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの共通項目は、API共通レスポンス処理に従う。

---

### 2.28 Responderで行わないこと

Responderでは、以下を行わない。

* 保有商品検索
* 利用者境界判定
* 無効化可否判定
* SoftDeletes
* `deleted_at`更新
* DBアクセス

Responderは、生成済み結果をHTTPレスポンスへ変換することに責務を限定する。

---

### 2.29 例外変換

主な例外変換は、以下とする。

| 内部状態                 | 独自エラーコード                   |
| -------------------- | -------------------------- |
| `X-User-Id`未指定       | `USER_CONTEXT_REQUIRED`    |
| `X-User-Id`形式不正      | `INVALID_USER_ID`          |
| 利用者不存在               | `USER_NOT_FOUND`           |
| `holdingAssetId`形式不正 | `INVALID_HOLDING_ASSET_ID` |
| 対象保有商品不存在            | `HOLDING_ASSET_NOT_FOUND`  |
| 他利用者の保有商品            | `HOLDING_ASSET_NOT_FOUND`  |
| 無効化済み保有商品            | `HOLDING_ASSET_NOT_FOUND`  |
| 想定外例外                | `INTERNAL_SERVER_ERROR`    |

他利用者のリソースであることを示す専用エラーコードは返却しない。

---

### 2.30 HoldingAssetNotFoundException

対象保有商品を取得できない場合は、共通またはHLD系の業務例外を送出する。

概念例：

```php
if ($holdingAsset === null) {
    throw new HoldingAssetNotFoundException();
}
```

最終的に、

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

へ変換する。

---

### 2.31 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

* SQL
* テーブル名
* カラム名
* PostgreSQL内部エラー
* PHP内部エラー
* Laravel内部例外メッセージ
* スタックトレース
* サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 2.32 ログ

HLD-005では、必要に応じて以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
holdingAssetId
httpStatus
errorCode
```

`apiId`は、

```text
HLD-005
```

とする。

保有商品名や過去の商品別月末評価額などを不要にログへ出力しない。

---

### 2.33 キャッシュ

Phase1では、HLD-005専用のサーバー側アプリケーションキャッシュを使用しない。

無効化後は、SoftDeletesの通常スコープによって対象保有商品が有効一覧・詳細取得対象から除外される。

---

### 2.34 テスト実装方針

Laravel側では、Feature Testを中心としてHLD-005のAPI契約と無効化処理を確認する。

主に以下を確認する。

* `200 OK`
* `400 Bad Request`
* `404 Not Found`
* `500 Internal Server Error`
* `X-User-Id`必須
* `X-User-Id`形式検証
* 利用者存在確認
* `holdingAssetId`形式検証
* 保有商品存在確認
* 利用者境界
* SoftDeletes
* 過去データ非更新
* snapshot非更新
* 再実行
* HLD-004とのルート分離
* リクエストボディ不要
* レスポンス契約

---

### 2.35 QueryのDatabase Test

`HoldingAssetQuery`について、以下を確認する。

```text
holdingAssetId一致
+
操作対象利用者一致
+
holding_assets.deleted_at IS NULL
+
asset_accounts.deleted_at IS NULL
    ↓
取得できる
```

また、以下を取得できないことを確認する。

* 他利用者の保有商品
* 論理削除済み保有商品
* 論理削除済み資産口座配下の保有商品
* 存在しない保有商品

---

### 2.36 RepositoryのDatabase Test

`HoldingAssetRepository::disable()`について、以下を確認する。

* `deleted_at`が設定されること
* レコードが物理削除されないこと
* `updated_at`が更新されること
* 他カラムが変更されないこと

特に、以下が保持されることを確認する。

```text
asset_account_id
name
product_type
start_year_month
memo
```

---

### 2.37 SoftDeletesのTest

無効化実行後に、通常Queryでは対象保有商品が取得できないことを確認する。

一方、

```php
HoldingAsset::withTrashed()
```

を使用すれば、レコード自体は存在することを確認する。

これにより、物理削除されていないことを保証する。

---

### 2.38 UseCaseのUnit Test

UseCaseについて、QueryとRepositoryをMockし、以下を確認する。

正常系：

```text
Query
    ↓
HoldingAsset返却
    ↓
Repository::disable()
    ↓
Disable Result DTO
```

異常系：

```text
Query
    ↓
null
    ↓
HoldingAssetNotFoundException
```

また、対象不存在時に

```text
Repository::disable()
```

が呼び出されないことを確認する。

---

### 2.39 関連データ非更新テスト

Feature TestまたはDatabase Testで、HLD-005実行前後の

```text
month_end_holding_values
month_end_asset_snapshots
```

を比較する。

以下が変更されないことを確認する。

* レコード数
* 商品別月末評価額
* snapshotの`confirmed`
* snapshotの`confirmed_at`

---

### 2.40 再実行テスト

同じ保有商品に対して2回HLD-005を実行する。

1回目：

```text
200 OK
```

2回目：

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

となることを確認する。

また、1回目に設定された

```text
deleted_at
```

が2回目によって更新されないことを確認する。

---

### 2.41 HLD-004とのルートテスト

以下の2ルートが独立して登録されていることを確認する。

```http
PATCH /api/v1/holding-assets/{holdingAssetId}
```

```http
PATCH /api/v1/holding-assets/{holdingAssetId}/disable
```

HLD-005リクエストがHLD-004のActionへルーティングされないことを確認する。

---

### 2.42 レスポンスResourceのTest

正常時に、以下だけが`data`へ含まれることを確認する。

```json
{
  "id": "15",
  "disabled": true
}
```

以下が含まれないことを確認する。

* `assetAccountId`
* `name`
* `productType`
* `startYearMonth`
* `memo`
* `createdAt`
* `updatedAt`
* `deletedAt`

これにより、HLD-005のレスポンスを無効化結果の確認に必要な最小限の情報へ限定する。

---

## 3. 関連ドキュメント

- [HLD-005 API詳細設計](../../../api/details/holding-assets/hld-005-disable.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [保有商品 Laravelアーキテクチャ設計](./README.md)
- [HLD-005 テスト設計](../../../tests/holding-assets/hld-005-disable.md)
- [保有商品 テスト設計](../../../tests/holding-assets/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
