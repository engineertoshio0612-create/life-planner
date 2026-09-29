## 1. 概要

操作対象となる利用者について、
指定した月末資産状況に紐づく
商品別月末評価額の一覧を取得する。

商品別月末評価額は、
商品単位で残高を管理する資産口座に属する
保有商品について記録された月末時点の評価額を表す。

本APIでは、
保存済みの商品別月末評価額を起点として、
対応する保有商品情報と組み合わせて返却する。

主な取得項目は以下とする。

- 保有商品ID
- 保有商品名
- 商品種別
- 月末評価額

対象となる月末資産状況は、
URLで指定された `snapshotId` によって特定する。

未確定・確定済みにかかわらず取得可能とし、
対象月末資産状況が存在しない場合、
他利用者に属する場合、
または論理削除済みの場合は、
対象不存在として扱う。

商品別月末評価額が0件の場合は、
正常系として空配列を返却する。

本APIは参照専用とし、
商品別月末評価額、月末資産状況、
保有商品の登録・更新・削除は行わない。

---

## 2. Laravel実装方針

VAL-001では、Action、Query、DTO、API Resource、Responderを分離して実装する。

参照専用APIであるため、Repositoryは使用しない。

概念的な構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ MonthEndAssetSnapshotQuery
    └─ MonthEndHoldingValueQuery
    ↓
List Result DTO
    ↓
API Resource Collection
    ↓
Responder
```

VAL-001はGETによる参照処理であり、

```text
INSERT
UPDATE
DELETE
```

を行わない。

### 2.1 Route

VAL-001は、以下のルートとして定義する。

概念例：

```php
Route::get(
    '/api/v1/month-end-asset-snapshots/{snapshotId}/holding-values',
    ListMonthEndHoldingValuesAction::class,
);
```

`snapshotId`は、正の整数形式だけを許可する。

概念例：

```php
Route::get(
    '/api/v1/month-end-asset-snapshots/{snapshotId}/holding-values',
    ListMonthEndHoldingValuesAction::class,
)
    ->where(
        'snapshotId',
        '[1-9][0-9]*',
    );
```

### 2.2 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、

```text
X-User-Id取得
    ↓
必須確認
    ↓
形式確認
    ↓
users存在確認
    ↓
UserContext設定
    ↓
Action
```

とする。

Action以降では、検証済みの利用者コンテキストを使用する。

### 2.3 FormRequest

VAL-001では、

```text
Request Bodyなし
クエリパラメータなし
```

であるため、専用FormRequestは原則として作成しない。

以下のような空FormRequestを形式的に作成しない。

```php
final class ListMonthEndHoldingValuesRequest
    extends FormRequest
{
}
```

`snapshotId`の形式検証は、RouteまたはAPI共通のパスパラメータ検証方式で行う。

### 2.4 snapshotIdの形式検証

`snapshotId`は、正の整数形式を必須とする。

正常例：

```text
1
20
999
```

不正例：

```text
0
-1
abc
1.5
1e3
20abc
```

形式不正は、

```text
INVALID_SNAPSHOT_ID
```

へ変換する。

### 2.5 Action

Actionは、`snapshotId`と利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php
final class ListMonthEndHoldingValuesAction
{
    public function __invoke(
        string $snapshotId,
        ListMonthEndHoldingValuesUseCase $useCase,
        ListMonthEndHoldingValuesResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                snapshotId:
                    (int) $snapshotId,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

### 2.6 Actionで行わないこと

Actionでは、以下を行わない。

- 月末資産状況検索
- 利用者境界判定
- 商品別月末評価額検索
- 保有商品検索
- 並び順制御
- JOIN条件組み立て
- DTO変換
- レスポンス配列生成
- DB更新

Actionは、

```text
HTTP入力
    ↓
UseCase
    ↓
Responder
```

の橋渡しに責務を限定する。

### 2.7 UseCase

VAL-001のアプリケーション処理全体を担当する。

主な処理は、以下とする。

- 操作対象利用者IDを受け取る
- `snapshotId`を受け取る
- 対象月末資産状況を取得する
- 対象不存在の場合は業務例外を送出する
- 商品別月末評価額一覧を取得する
- 保有商品情報を組み合わせる
- 固定の並び順を適用する
- List Result DTOへ変換する
- Result DTO一覧を返却する

### 2.8 UseCaseの概念フロー

```text
userId
+
snapshotId
    ↓
MonthEndAssetSnapshotQuery
    ↓
月末資産状況取得
    ↓
NOT_FOUND？
    ↓ No
MonthEndHoldingValueQuery
    ↓
商品別月末評価額一覧取得
    ↓
holding_assets情報取得
    ↓
並び順適用
    ↓
Result DTO一覧
```

### 2.9 MonthEndAssetSnapshotQuery

対象月末資産状況の取得は、専用Queryへ委譲する。

概念例：

```php
$snapshot =
    $this->monthEndAssetSnapshotQuery
        ->findByIdAndUser(
            snapshotId:
                $snapshotId,

            userId:
                $userId,
        );
```

取得条件は、

```text
month_end_asset_snapshots.id
    = snapshotId

AND

month_end_asset_snapshots.user_id
    = userId
```

とする。

### 2.10 利用者境界をQueryへ含める

以下のようなID単独取得は基本としない。

```php
MonthEndAssetSnapshot::find(
    $snapshotId,
);
```

対象取得時点から、

```text
snapshotId
+
userId
```

を条件へ含める。

これにより、他利用者の月末資産状況を不要に取得しない。

### 2.11 SoftDeletes

`month_end_asset_snapshots`でSoftDeletesを採用する場合は、通常取得で論理削除済みを除外する。

例えば、Modelで

```php
use SoftDeletes;
```

を使用している場合は、VAL-001では

```php
withTrashed()
```

を使用しない。

### 2.12 MONTH_END_ASSET_SNAPSHOT_NOT_FOUND

対象月末資産状況を取得できない場合は、

```php
throw new
    MonthEndAssetSnapshotNotFoundException();
```

とする。

以下を同じ例外へ集約する。

- 月末資産状況不存在
- 他利用者所属
- 論理削除済み

最終的に、

```text
404 Not Found
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND
```

へ変換する。

### 2.13 confirmedは取得可否条件に含めない

対象月末資産状況の

```text
confirmed
```

は、VAL-001の取得条件に含めない。

以下のどちらでも商品別月末評価額を取得できる。

```text
confirmed = false
confirmed = true
```

### 2.14 MonthEndHoldingValueQuery

商品別月末評価額一覧の取得は、専用Queryへ委譲する。

概念例：

```php
$holdingValues =
    $this->monthEndHoldingValueQuery
        ->findListBySnapshot(
            snapshotId:
                $snapshot->id,
        );
```

主な取得条件は、

```text
month_end_holding_values.snapshot_id
    = snapshotId
```

とする。

### 2.15 Queryを起点にするテーブル

VAL-001では、保存済みの商品別月末評価額を取得することが目的である。

そのため、取得の起点は

```text
month_end_holding_values
```

とする。

現在有効な

```text
holding_assets
```

一覧を起点として評価額を探す方式にはしない。

### 2.16 holding_assetsをJOINする

レスポンスに必要な保有商品情報を取得するため、

```text
month_end_holding_values
    ↓
holding_assets
```

をJOINする。

概念例：

```php
MonthEndHoldingValue::query()
    ->join(
        'holding_assets',
        'holding_assets.id',
        '=',
        'month_end_holding_values.holding_asset_id',
    );
```

### 2.17 N+1を避ける

以下のように、評価額1件ごとに保有商品を取得しない。

```text
Holding Value 1
    ↓
SELECT holding_assets

Holding Value 2
    ↓
SELECT holding_assets

Holding Value 3
    ↓
SELECT holding_assets
```

JOINまたはEager Loadを使用し、必要情報をまとめて取得する。

### 2.18 JOIN方式

VAL-001では、表示に必要な項目が明確であるため、Query BuilderによるJOINを使用してよい。

概念例：

```php
return MonthEndHoldingValue::query()
    ->select([
        'month_end_holding_values.id',
        'month_end_holding_values.holding_asset_id',
        'month_end_holding_values.value',
        'holding_assets.name as holding_asset_name',
        'holding_assets.asset_type',
    ])
    ->join(
        'holding_assets',
        'holding_assets.id',
        '=',
        'month_end_holding_values.holding_asset_id',
    )
    ->where(
        'month_end_holding_values.snapshot_id',
        $snapshotId,
    )
    ->orderBy(
        'month_end_holding_values.holding_asset_id',
    )
    ->get();
```

実際のカラム名は、テーブル定義を正とする。

### 2.19 INNER JOINを基本とする

外部キー制約によって、

```text
month_end_holding_values.holding_asset_id
    ↓
holding_assets.id
```

の参照整合性が保証されている場合は、INNER JOINを基本とする。

対応する保有商品が存在しない状態は正常データとして扱わない。

### 2.20 SoftDeletesされたholding_assetsの扱い

保存済みの商品別月末評価額は、現在の保有商品状態だけを理由として除外しない。

そのため、`holding_assets`の

```text
deleted_at IS NULL
```

をVAL-001のJOIN条件へ無条件に追加しない。

例えば、Eloquent RelationshipのSoftDeletes Global Scopeによって過去の保有商品が自動除外されないよう注意する。

### 2.21 過去データ参照でGlobal Scopeに注意する

HoldingAsset ModelでSoftDeletesを使用している場合、Eager Loadすると

```text
deleted_at IS NULL
```

が自動適用される可能性がある。

その結果、

```text
評価額は存在する
+
保有商品は論理削除済み
    ↓
商品情報が取得できない
```

となる可能性がある。

過去評価額を参照可能とする仕様に合わせて、必要に応じて

```php
withTrashed()
```

またはQuery Builder JOINを使用する。

### 2.22 現在のenabled状態で除外しない

保有商品に利用状態カラムが存在する場合でも、

```text
enabled = true
```

だけをVAL-001の取得条件にしない。

保存済みの過去評価額が現在状態の変更によって消えて見えないようにする。

### 2.23 月末資産状況と評価額の利用者境界

`month_end_holding_values`に直接`user_id`を持たない場合は、親となる

```text
month_end_asset_snapshots
```

で利用者境界を保証する。

概念的には、

```text
UserContext
    ↓
month_end_asset_snapshots
    ↓
利用者境界確認済みsnapshotId
    ↓
month_end_holding_values
```

とする。

### 2.24 holding_assets側で別利用者データが紐づかないようDB制約を前提とする

正常なデータでは、

```text
month_end_holding_values
    ↓
holding_assets
    ↓
asset_accounts
    ↓
users
```

の所有関係が月末資産状況の利用者と整合していることを前提とする。

VAL-001で各レコードごとに利用者所有関係を再計算する構成にはしない。

ただし、登録API側およびDB制約でこの整合性を保証する。

### 2.25 並び順

一覧の並び順は、Query側で明示する。

Phase1では、例えば、

```text
holding_asset_id ASC
```

を基本としてよい。

資産口座や保有商品に明示的な表示順がある場合は、その仕様を優先する。

### 2.26 ORDER BYなしにしない

以下のようにDBの自然順へ依存しない。

```php
->where(
    'snapshot_id',
    $snapshotId,
)
->get();
```

一覧APIとして安定した表示順を保証するため、明示的に`orderBy()`を設定する。

### 2.27 Queryで業務データを更新しない

MonthEndHoldingValueQueryでは、以下を行わない。

- 未登録評価額作成
- 評価額更新
- 評価額削除
- 保有商品更新
- 月末資産状況更新

Queryは読み取りに責務を限定する。

### 2.28 Repositoryを使用しない

VAL-001では業務データを変更しないため、Repositoryは使用しない。

概念的には、

```text
Query
    → SELECT

Repository
    → INSERT / UPDATE / DELETE
```

という共通方針に従う。

VAL-001で書き込み責務を持つクラスを追加しない。

### 2.29 0件

MonthEndHoldingValueQueryの取得結果が0件の場合は、例外にしない。

概念例：

```php
if ($holdingValues->isEmpty()) {
    return [];
}
```

ただし、特別な分岐すら不要であれば、空CollectionをそのままDTO変換処理へ渡してよい。

### 2.30 0件専用例外を作らない

以下のような専用例外は作成しない。

```text
MonthEndHoldingValuesNotFoundException
```

一覧0件は正常状態として扱う。

### 2.31 Result DTO

商品別月末評価額1件を、専用Result DTOとして表現する。

概念例：

```php
final readonly class
    MonthEndHoldingValueListItem
{
    public function __construct(
        public int $id,
        public int $holdingAssetId,
        public string $holdingAssetName,
        public string $assetType,
        public int $value,
    ) {
    }
}
```

### 2.32 List Result DTO

必要に応じて、一覧全体を表現するResult DTOを用意してもよい。

概念例：

```php
final readonly class
    ListMonthEndHoldingValuesResult
{
    /**
     * @param list<MonthEndHoldingValueListItem> $items
     */
    public function __construct(
        public array $items,
    ) {
    }
}
```

一覧だけを返却する単純なAPIであれば、Item DTOのCollectionとして扱ってもよい。

### 2.33 DTOへEloquent Modelを保持しない

以下のようなResult DTOは基本としない。

```php
final readonly class
    MonthEndHoldingValueListItem
{
    public function __construct(
        public MonthEndHoldingValue $model,
    ) {
    }
}
```

APIレスポンスに必要な値だけをDTOへ保持する。

### 2.34 Query結果からDTOへ変換する

概念例：

```php
$items =
    $holdingValues
        ->map(
            static fn ($row) =>
                new MonthEndHoldingValueListItem(
                    id:
                        (int) $row->id,

                    holdingAssetId:
                        (int) $row->holding_asset_id,

                    holdingAssetName:
                        $row->holding_asset_name,

                    assetType:
                        $row->asset_type,

                    value:
                        (int) $row->value,
                ),
        )
        ->all();
```

### 2.35 assetTypeの変換

DBで`asset_type`を`smallint`等で保持している場合は、API用の文字列表現へ変換する。

例えば、

```text
1
    ↓
INVESTMENT_TRUST
```

のような変換を行う場合は、HLD系APIと同じEnumまたは変換処理を再利用する。

VAL-001専用の異なるマッピングを作成しない。

### 2.36 Enumを共通利用する

Laravel Enumを使用する場合は、例えば、

```php
AssetType::from(
    $row->asset_type,
)->name;
```

など、保有商品APIと同じ変換方針を使用する。

実際のEnum定義は、プロジェクト共通設計に従う。

### 2.37 API Resource

Result DTOを、専用API ResourceでAPIレスポンス形式へ変換する。

概念例：

```php
final class MonthEndHoldingValueResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'holdingAssetId'
                => (string) $this->holdingAssetId,

            'holdingAssetName'
                => $this->holdingAssetName,

            'assetType'
                => $this->assetType,

            'value'
                => $this->value,
        ];
    }
}
```

### 2.38 Resource Collection

一覧レスポンスでは、Resource Collectionを使用する。

概念例：

```php
MonthEndHoldingValueResource::collection(
    $result->items,
);
```

または、DTOのCollectionを直接渡せる構成としてよい。

### 2.39 API Resourceで返却しない情報

以下をVAL-001レスポンスへ含めない。

- `snapshot_id`
- `user_id`
- `asset_account_id`
- `created_at`
- `updated_at`
- `deleted_at`
- DB内部管理情報

必要な業務情報だけを返却する。

### 2.40 snake_caseを直接返さない

DBの

```text
holding_asset_id
asset_type
```

は、APIでは

```text
holdingAssetId
assetType
```

として返却する。

DB構造をそのままAPI契約へ公開しない。

### 2.41 Responder

Responderは、Result DTO一覧を`200 OK`レスポンスへ変換する。

概念例：

```php
final class ListMonthEndHoldingValuesResponder
{
    public function ok(
        ListMonthEndHoldingValuesResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    =>
                    MonthEndHoldingValueResource::collection(
                        $result->items,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

実際のEnvelope生成方式は、API共通方針に従う。

### 2.42 0件レスポンス

0件の場合は、

```json
{
  "data": []
}
```

を返却する。

Responderで0件専用の別レスポンス形式を作らない。

### 2.43 Responderで行わないこと

Responderでは、以下を行わない。

- 月末資産状況検索
- 利用者境界判定
- 評価額検索
- 保有商品検索
- 並び順制御
- `assetType`業務判定
- DBアクセス

HTTPレスポンス生成だけに責務を限定する。

### 2.44 明示的トランザクション

VAL-001では、参照専用であるため、

```php
DB::transaction()
```

を原則として使用しない。

複数SELECTを厳密な同一時点で読み取る必要がある業務要件もPhase1では設けない。

### 2.45 lockForUpdateを使用しない

VAL-001では、

```php
lockForUpdate()
```

を使用しない。

参照APIが更新APIの処理を不要に待機させない。

### 2.46 キャッシュ

Phase1では、VAL-001専用のサーバー側キャッシュを使用しない。

商品別月末評価額は更新可能なデータであるため、DB上の最新状態をそのまま取得する。

React側のQuery Cacheについては、VAL登録・更新API成功後にVAL-001のQueryをinvalidateする。

### 2.47 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `snapshotId`形式不正 | `INVALID_SNAPSHOT_ID` |
| 月末資産状況不存在 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 他利用者の月末資産状況 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 論理削除済み月末資産状況 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

### 2.48 MonthEndAssetSnapshotNotFoundException

対象月末資産状況が取得できない場合は、

```php
throw new
    MonthEndAssetSnapshotNotFoundException();
```

とする。

最終的に、

```text
404 Not Found
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND
```

へ変換する。

### 2.49 内部データ不整合

例えば、

```text
month_end_holding_values
    ↓
holding_asset_id
    ↓
holding_assets不存在
```

のような通常発生しない状態を検出した場合は、利用者入力エラーとはしない。

外部キー制約によって原則として防止する。

万一発生した場合は、共通Exception Handlerで

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

へ変換する。

### 2.50 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- SQL
- SQLSTATE
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

### 2.51 ログ

VAL-001では、必要に応じて以下をログコンテキストへ設定する。

- `requestId`
- `userId`
- `apiId`
- `snapshotId`
- `httpStatus`
- `errorCode`

`apiId`は、

```text
VAL-001
```

とする。

### 2.52 正常時ログ

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = VAL-001
snapshotId
resultCount
httpStatus = 200
```

商品名や評価額を通常ログへ不要に全件出力しない。

### 2.53 0件時ログ

商品別月末評価額が0件であっても、正常レスポンスである。

そのため、通常は

```text
error
warning
```

として扱わない。

必要に応じて、

```text
resultCount = 0
```

を通常のアクセスログへ記録するだけとする。

### 2.54 エラー時ログ

異常時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = VAL-001
snapshotId
errorCode
httpStatus
```

他利用者所属などの内部判定理由をAPIレスポンスへは公開しない。

### 2.55 テスト実装方針

Laravel側では、Feature Testを中心にVAL-001のAPI契約を確認する。

また、Query、UseCase、API Resourceについて必要に応じてUnit TestまたはDatabase Testを行う。

### 2.56 MonthEndAssetSnapshotQueryのDatabase Test

以下を確認する。

```text
id一致
+
user_id一致
    ↓
取得できる
```

以下は取得できないこと。

- 存在しない`snapshotId`
- 他利用者の月末資産状況
- 論理削除済み月末資産状況

### 2.57 MonthEndHoldingValueQueryのDatabase Test

対象`snapshotId`について、該当する商品別月末評価額だけを取得できることを確認する。

例えば、

```text
Snapshot A
    Holding Value 1
    Holding Value 2

Snapshot B
    Holding Value 3
```

の場合に、Snapshot Aを指定すると、

```text
Holding Value 1
Holding Value 2
```

だけが返却されること。

### 2.58 JOINのTest

商品別月末評価額と保有商品情報が正しく結合されることを確認する。

主に、

```text
holdingAssetId
holdingAssetName
assetType
value
```

が期待値と一致すること。

### 2.59 無効化済み保有商品のTest

保存済みの商品別月末評価額に紐づく保有商品を現在無効化済み状態にする。

その状態でも、VAL-001で保存済み評価額が取得できることを確認する。

### 2.60 論理削除済み保有商品のTest

仕様上、論理削除済み保有商品に紐づく過去評価額も参照可能とする場合は、その商品情報がSoftDeletes Global Scopeによって欠落しないことを確認する。

### 2.61 0件Test

対象月末資産状況は存在するが、商品別月末評価額が0件の状態を用意する。

期待結果：

```http
200 OK
```

```json
{
  "data": []
}
```

となること。

### 2.62 他月データ非混在Test

同じ保有商品について複数月の商品別評価額を登録する。

指定した`snapshotId`に紐づく評価額だけが返却されることを確認する。

### 2.63 並び順Test

複数の商品別月末評価額を意図的に異なる登録順で作成する。

VAL-001では、DB登録順ではなくAPI仕様で定めた固定順となることを確認する。

### 2.64 API ResourceのTest

1件について、以下の形式となることを確認する。

```json
{
  "id": "101",
  "holdingAssetId": "10",
  "holdingAssetName": "eMAXIS Slim 全世界株式",
  "assetType": "INVESTMENT_TRUST",
  "value": 350000
}
```

### 2.65 API Resourceの型Test

以下を確認する。

```text
id
    → string

holdingAssetId
    → string

holdingAssetName
    → string

assetType
    → string

value
    → integer
```

### 2.66 API Resourceで返却しない項目

以下がレスポンスへ含まれないことを確認する。

- `snapshotId`
- `snapshot_id`
- `userId`
- `user_id`
- `assetAccountId`
- `createdAt`
- `updatedAt`
- `deletedAt`
- DB内部管理情報

### 2.67 Feature Test正常系

以下を実行する。

```http
GET /api/v1/month-end-asset-snapshots/20/holding-values
X-User-Id: 1
Accept: application/json
```

期待結果：

```http
200 OK
```

かつ、指定Snapshotに属する商品別月末評価額一覧が返却されること。

### 2.68 Feature Test他利用者

他利用者に属する`snapshotId`を指定する。

期待結果：

```text
404 Not Found
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND
```

となること。

他利用者の商品別月末評価額がレスポンスへ含まれないこと。

### 2.69 Feature Test確定済み

対象Snapshotを

```text
confirmed = true
```

とする。

VAL-001を実行しても、

```http
200 OK
```

で商品別月末評価額一覧を取得できることを確認する。

### 2.70 副作用なしTest

VAL-001実行前後で、以下が変更されないことを確認する。

- `month_end_asset_snapshots`
- `month_end_holding_values`
- `holding_assets`

特に、

```text
INSERT
UPDATE
DELETE
```

が発生しないことを確認する。

---

## 3. 関連ドキュメント

- [VAL-001 API詳細設計](../../../api/details/holding-values/val-001-list.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [商品別月末評価額 Laravelアーキテクチャ設計](./README.md)
- [VAL-001 テスト設計](../../../tests/holding-values/val-001-list.md)
- [商品別月末評価額 テスト設計](../../../tests/holding-values/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)