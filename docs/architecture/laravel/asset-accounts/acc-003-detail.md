# ACC-003 資産口座詳細取得

## 1. 概要

本ドキュメントでは、
ACC-003 資産口座詳細取得APIをLaravelで実装する際の、アーキテクチャおよび責務分離方針を定義する。

ACC-003では、操作対象利用者に帰属する指定された資産口座の詳細情報を取得する。

資産口座の基本情報に加えて、API実行時点で有効な利用可能資産設定を取得し、現在の利用可能資産区分として返却する。

本APIは参照専用であり、資産口座、利用可能資産設定、および関連する業務データの登録・更新は行わない。

Laravel実装では、HTTPリクエストの受付から利用者コンテキストの取得、資産口座の取得、現在有効な利用可能資産設定の取得、データ整合性の確認、APIレスポンスの生成までを単一のクラスへ集約せず、各責務を分離する。

概念的な処理構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ AssetAccountQuery
    └─ AssetAccountAvailableSettingQuery
    ↓
Result DTO
    ↓
Responder
    ↓
API Resource
    ↓
HTTP Response
```

Actionは、パスパラメータおよび利用者コンテキストを受け取り、UseCaseの呼び出しを担当する。

UseCaseは、資産口座詳細取得に必要な処理を統括し、Queryを利用して操作対象利用者に帰属する有効な資産口座、および現在有効な利用可能資産設定を取得する。

Queryは、利用者境界を含めた資産口座の検索、および現在年月を基準とした利用可能資産設定の検索を担当する。

現在有効な利用可能資産設定は正常状態では1件のみ存在することを前提とし、0件または複数件の場合はデータ不整合として扱う。

本APIは参照専用であるため、Repositoryによるデータ更新や明示的なトランザクション、ロックは使用しない。

ResponderおよびAPI Resourceは、UseCaseから受け取った取得結果をAPI共通方針に従ったレスポンス形式へ変換する。

これにより、HTTP層、ユースケース、データ取得、データ整合性確認、レスポンス生成の責務を明確に分離し、参照処理の安全性、テスト容易性および保守性を確保する。

---

## 2. Laravel実装方針

ACC-003では、Action、Query、DTO、API Resource、Responderを分離して実装する。

概念的な構成は、以下とする。

```text id="f3z6rq"
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ AssetAccountQuery
    └─ AssetAccountAvailableSettingQuery
    ↓
Detail Result DTO
    ↓
API Resource
    ↓
Responder
```

本APIはリクエストボディを使用しないため、ACC-003専用のFormRequestは作成しない。

また、参照専用APIであるため、Repositoryは使用せず、取得処理はQueryへ集約する。

---

### 2.1 Route

ACC-003は、以下のルートとして定義する。

概念例：

```php id="6jn9va"
Route::get(
    '/api/v1/asset-accounts/{assetAccountId}',
    ShowAssetAccountAction::class,
);
```

ACC-001とは、HTTPメソッドおよびパスによって区別する。

```text id="xquw2f"
GET
/api/v1/asset-accounts
    → ACC-001 資産口座一覧取得

GET
/api/v1/asset-accounts/{assetAccountId}
    → ACC-003 資産口座詳細取得
```

---

### 2.2 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* 共通例外処理
* ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、以下とする。

```text id="ah9d3g"
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

### 2.3 FormRequest

ACC-003専用のFormRequestは作成しない。

本APIでは、

```text id="2c8xiy"
リクエストボディ
    なし

クエリパラメータ
    なし
```

であり、ACC-003固有のRequest Bodyバリデーションが存在しないためである。

以下のような空のFormRequestは作成しない。

```php id="yx5blc"
final class ShowAssetAccountRequest
    extends FormRequest
{
}
```

不要なクラスを追加しない。

---

### 2.4 assetAccountIdの形式検証

`assetAccountId`は、ルート制約またはAPI共通のパスパラメータ検証方式で検証する。

概念例：

```php id="3j6h5m"
Route::get(
    '/api/v1/asset-accounts/{assetAccountId}',
    ShowAssetAccountAction::class,
)
    ->where(
        'assetAccountId',
        '[1-9][0-9]*',
    );
```

正の整数形式のみを許可する。

以下を有効なIDとして扱わない。

```text id="26c1kl"
0
-1
abc
1.5
1e3
10abc
```

形式不正は、

```text id="l8n0vg"
INVALID_ASSET_ACCOUNT_ID
```

へ変換する。

---

### 2.5 Action

Actionは、パスパラメータと利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php id="vg1yuc"
final class ShowAssetAccountAction
{
    public function __invoke(
        string $assetAccountId,
        ShowAssetAccountUseCase $useCase,
        ShowAssetAccountResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                assetAccountId:
                    (int) $assetAccountId,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

`assetAccountId`は、形式検証済みであることを前提に整数へ変換する。

---

### 2.6 Actionで行わないこと

Actionでは、以下を行わない。

* 資産口座検索
* 利用者境界判定
* 論理削除判定
* 現在年月算出
* 利用可能資産設定検索
* 設定件数判定
* データ不整合判定
* Enum変換
* レスポンス配列生成
* SQL生成

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 2.7 UseCase

ACC-003の参照ユースケース全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `assetAccountId`を受け取る
3. 操作対象利用者に属する有効な資産口座を取得する
4. 存在しない場合は業務例外を送出する
5. 現在年月を取得する
6. 現在有効な利用可能資産設定を取得する
7. 設定件数が1件であることを確認する
8. Detail Result DTOを生成する
9. DTOを返却する

概念的には、以下とする。

```text id="ue38c9"
userId
+
assetAccountId
    ↓
AssetAccountQuery
    ↓
資産口座取得
    ↓
存在確認
    ↓
CurrentYearMonth取得
    ↓
AssetAccountAvailableSettingQuery
    ↓
現在有効設定取得
    ↓
件数確認
    ↓
Detail Result DTO
```

---

### 2.8 AssetAccountQuery

資産口座取得は、専用Queryへ委譲する。

概念例：

```php id="jm6wga"
$assetAccount =
    $this->assetAccountQuery
        ->findActiveByIdAndUser(
            assetAccountId:
                $assetAccountId,

            userId:
                $userId,
        );
```

概念的な検索条件は、以下とする。

```text id="lpf1z9"
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 2.9 利用者境界をQueryへ含める

以下のように、まずIDだけで取得してからUseCase側で利用者IDを比較する方式を基本としない。

```php id="3nnwlq"
$assetAccount =
    AssetAccount::find(
        $assetAccountId,
    );
```

対象取得時点から、

```text id="nszbf2"
assetAccountId
+
userId
+
有効状態
```

を条件へ含める。

これにより、他利用者の資産口座を誤って取得することを防止する。

---

### 2.10 SoftDeletes

`AssetAccount` Modelでは、Laravelの

```php id="p3tf0c"
use SoftDeletes;
```

を使用する。

ACC-003の通常取得では、

```php id="xv7olh"
withTrashed()
```

を使用しない。

論理削除済み資産口座は、詳細取得対象外とする。

---

### 2.11 ASSET_ACCOUNT_NOT_FOUND

資産口座を取得できない場合は、

```php id="dfwx7a"
if ($assetAccount === null) {
    throw new
        AssetAccountNotFoundException();
}
```

のように業務例外を送出する。

以下を同じ例外へ集約する。

* 資産口座不存在
* 他利用者に属する
* 論理削除済み

最終的に、

```text id="eizmy1"
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

へ変換する。

---

### 2.12 CurrentYearMonth

利用可能資産設定の判定に使用する現在年月は、UseCase内でサーバー時刻から取得する。

概念例：

```php id="a8fwmr"
$currentYearMonth =
    now()->format(
        'Y-m',
    );
```

ただし、テスト容易性を高めるため、現在時刻取得を直接`now()`へ依存させず、ClockやDateProviderを利用してもよい。

概念例：

```php id="2o8hvr"
$currentYearMonth =
    $this->clock
        ->now()
        ->format(
            'Y-m',
        );
```

---

### 2.13 Clockを分離してよい理由

ACC-003の

```text id="sge4w7"
isAvailable
```

は、現在年月によって結果が変化する。

そのため、時刻依存処理をテストで固定しやすくするために、現在時刻取得を専用インターフェースへ分離してよい。

例えば、

```php id="t67d9c"
interface Clock
{
    public function now(): CarbonImmutable;
}
```

とする。

Phase1で過剰になる場合は、Laravelの時刻固定機能を使い、`now()`を直接使用してもよい。

---

### 2.14 AssetAccountAvailableSettingQuery

現在有効な利用可能資産設定の取得は、専用Queryへ委譲する。

概念例：

```php id="k0zmf2"
$settings =
    $this->availableSettingQuery
        ->findCurrentByAssetAccount(
            assetAccountId:
                $assetAccount->id,

            yearMonth:
                $currentYearMonth,
        );
```

概念的な検索条件は、以下とする。

```text id="fyso03"
asset_account_id
    = assetAccountId

AND

start_year_month
    <= currentYearMonth

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= currentYearMonth
)
```

---

### 2.15 設定を全件取得しない

以下のように、利用可能資産設定の全履歴を取得してからPHP側で現在設定を探す方式は基本としない。

```php id="mw4e40"
$settings =
    AssetAccountAvailableSetting::query()
        ->where(
            'asset_account_id',
            $assetAccountId,
        )
        ->get();
```

データベース側で現在年月条件を指定する。

---

### 2.16 件数判定

現在有効な設定は、正常状態では1件だけ存在することを前提とする。

概念例：

```php id="l2i3vk"
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingNotFoundException();
}

if ($settings->count() > 1) {
    throw new
        AssetAccountAvailableSettingConflictException();
}
```

0件または複数件を正常状態として扱わない。

---

### 2.17 `first()`だけで済ませない

以下のような実装は基本としない。

```php id="y3pfwy"
$setting =
    $query
        ->orderByDesc(
            'start_year_month',
        )
        ->first();
```

これでは、設定期間が重複していても最新1件を返してデータ不整合を隠蔽する可能性がある。

ACC-003では、該当件数を確認し、1件であることを保証する。

---

### 2.18 取得上限

複数件存在するかを検出するだけであれば、全件を取得する必要はない。

概念的には、最大2件まで取得してよい。

```php id="87fmxa"
$settings =
    AssetAccountAvailableSetting::query()
        ->where(...)
        ->limit(2)
        ->get();
```

これにより、

```text id="v9j3fp"
0件
1件
2件以上
```

を判定できる。

---

### 2.19 Repositoryを使用しない

ACC-003は参照専用APIであり、以下のDB更新を行わない。

```text id="mjkqun"
INSERT
UPDATE
DELETE
```

そのため、ACC-003専用のRepositoryは使用しない。

概念的には、

```text id="py7f2t"
Query
    → 読み取り

Repository
    → 書き込み
```

という責務分離方針に従う。

---

### 2.20 トランザクション

ACC-003では、明示的な

```php id="0g6kuw"
DB::transaction()
```

を使用しない。

参照専用であり、表示用途として厳密な同一時点スナップショットを要求しないためである。

---

### 2.21 lockForUpdateを使用しない

ACC-003では、以下を使用しない。

```php id="al5fse"
lockForUpdate()
```

資産口座や利用可能資産設定を参照するだけであり、詳細表示のために更新処理をブロックしない。

---

### 2.22 Enum Cast

`asset_type`と`balance_recording_unit`は、可能であればEloquent CastまたはPHP Enumとして扱う。

概念例：

```php id="e3d4u0"
protected function casts(): array
{
    return [
        'asset_type'
            => AssetType::class,

        'balance_recording_unit'
            => BalanceRecordingUnit::class,
    ];
}
```

DB保存形式がAPI公開値と異なる場合は、Mapper等で変換する。

---

### 2.23 DB内部コードをUseCaseへ漏らさない

例えば、

```text id="gtq6xq"
1 = CASH
2 = BANK
3 = SECURITIES
```

のようなDB内部コードを、UseCaseやResourceへ直接持ち込まない。

UseCase以降では、

```text id="8prwli"
AssetType
BalanceRecordingUnit
```

という意味のある型として扱う。

---

### 2.24 Detail Result DTO

取得結果は、専用DTOとして表現する。

概念例：

```php id="8qey2m"
final readonly class AssetAccountDetailResult
{
    public function __construct(
        public int $id,
        public string $name,
        public AssetType $assetType,
        public BalanceRecordingUnit $balanceRecordingUnit,
        public bool $isAvailable,
        public string $startYearMonth,
    ) {
    }
}
```

`isEnabled`は、ACC-003正常時に必ず`true`となるため、Resource側で固定値として設定してよい。

---

### 2.25 DTOへEloquent Modelを保持しない

以下のようなDTOは基本としない。

```php id="0rbp13"
final readonly class AssetAccountDetailResult
{
    public function __construct(
        public AssetAccount $assetAccount,
        public AssetAccountAvailableSetting $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

これにより、HTTP層へEloquent Modelを直接渡さない。

---

### 2.26 UseCaseの戻り値

概念例：

```php id="m1zp84"
return new AssetAccountDetailResult(
    id:
        $assetAccount->id,

    name:
        $assetAccount->name,

    assetType:
        $assetAccount->asset_type,

    balanceRecordingUnit:
        $assetAccount->balance_recording_unit,

    isAvailable:
        $setting->is_available,

    startYearMonth:
        $assetAccount->start_year_month,
);
```

---

### 2.27 API Resource

Detail Result DTOを、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php id="d61q3c"
final class AssetAccountDetailResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'name'
                => $this->name,

            'assetType'
                => $this->assetType->value,

            'balanceRecordingUnit'
                => $this->balanceRecordingUnit->value,

            'isAvailable'
                => $this->isAvailable,

            'startYearMonth'
                => $this->startYearMonth,

            'isEnabled'
                => true,
        ];
    }
}
```

APIフィールド名は、共通方針に従ってcamelCaseとする。

---

### 2.28 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

* `user_id`
* `deleted_at`
* `created_at`
* `updated_at`
* `asset_account_available_settings.id`
* `asset_account_available_settings.asset_account_id`
* `asset_account_available_settings.start_year_month`
* `asset_account_available_settings.end_year_month`
* DB内部コード

また、以下の関連データも返却しない。

* 保有商品一覧
* 月末資産残高
* 商品別月末評価額
* 月末資産状況

---

### 2.29 Responder

Responderは、Detail Result DTOを`200 OK`レスポンスへ変換する。

概念例：

```php id="os6g3n"
final class ShowAssetAccountResponder
{
    public function ok(
        AssetAccountDetailResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new AssetAccountDetailResource(
                        $result,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの共通Envelope項目は、API共通レスポンス処理に従う。

---

### 2.30 Responderで行わないこと

Responderでは、以下を行わない。

* 資産口座検索
* 利用者境界判定
* 現在年月取得
* 利用可能資産設定検索
* 設定件数判定
* データ不整合判定
* DBアクセス
* Enum変換ロジックの組み立て

Responderは、生成済みDTOをHTTPレスポンスへ変換することに責務を限定する。

---

### 2.31 例外変換

主な例外変換は、以下とする。

| 内部状態                 | 独自エラーコード                                    |
| -------------------- | ------------------------------------------- |
| `X-User-Id`未指定       | `USER_CONTEXT_REQUIRED`                     |
| `X-User-Id`形式不正      | `INVALID_USER_ID`                           |
| 利用者不存在               | `USER_NOT_FOUND`                            |
| `assetAccountId`形式不正 | `INVALID_ASSET_ACCOUNT_ID`                  |
| 資産口座不存在              | `ASSET_ACCOUNT_NOT_FOUND`                   |
| 他利用者の資産口座            | `ASSET_ACCOUNT_NOT_FOUND`                   |
| 論理削除済み資産口座           | `ASSET_ACCOUNT_NOT_FOUND`                   |
| 現在設定0件               | `ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND` |
| 現在設定複数件              | `ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT`  |
| 想定外例外                | `INTERNAL_SERVER_ERROR`                     |

---

### 2.32 AssetAccountNotFoundException

対象資産口座を取得できない場合は、業務例外を送出する。

概念例：

```php id="m9svb7"
if ($assetAccount === null) {
    throw new
        AssetAccountNotFoundException();
}
```

最終的に、

```text id="k3j57n"
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

へ変換する。

---

### 2.33 AssetAccountAvailableSettingNotFoundException

現在有効な利用可能資産設定が0件の場合は、データ不整合として専用例外を送出する。

概念例：

```php id="pijztt"
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingNotFoundException();
}
```

最終的に、

```text id="d7a8t1"
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

へ変換する。

---

### 2.34 AssetAccountAvailableSettingConflictException

現在有効な設定が複数件存在する場合は、専用例外を送出する。

概念例：

```php id="hmzqvi"
if ($settings->count() > 1) {
    throw new
        AssetAccountAvailableSettingConflictException();
}
```

最終的に、

```text id="y0ne3a"
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

へ変換する。

---

### 2.35 データ不整合時に自動修復しない

以下のような処理は行わない。

```text id="p5r4n6"
現在設定0件
    ↓
isAvailable = false
```

または、

```text id="5pf1x4"
現在設定複数件
    ↓
最新1件採用
```

ACC-003は参照APIであるため、不整合状態を暗黙的に修復しない。

---

### 2.36 想定外例外

想定外の例外は、API共通Exception Handlerで

```text id="zps3jg"
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

* SQL
* PostgreSQL内部エラー
* 制約名
* テーブル名
* カラム名
* Laravel内部例外メッセージ
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

---

### 2.37 ログ

ACC-003では、必要に応じて以下をログコンテキストへ設定する。

```text id="03ox3h"
requestId
userId
apiId
assetAccountId
currentYearMonth
httpStatus
errorCode
```

`apiId`は、

```text id="cd29r4"
ACC-003
```

とする。

正常時に、資産口座名や`isAvailable`などの不要な業務情報をログへ出力しない。

---

### 2.38 データ不整合ログ

以下のエラー時は、調査に必要な情報を内部ログへ記録する。

```text id="316ihd"
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text id="ghaq9s"
requestId
userId
assetAccountId
currentYearMonth
matchedSettingCount
```

を記録してよい。

レスポンスへは内部状態を公開しない。

---

### 2.39 キャッシュ

Phase1では、ACC-003専用のサーバー側キャッシュを使用しない。

毎回、最新の

```text id="aq1e79"
asset_accounts
asset_account_available_settings
```

を参照する。

フロントエンド側では、TanStack Query等によるQuery Cacheを使用してよい。

---

### 2.40 テスト実装方針

Laravel側では、Feature Testを中心としてACC-003のAPI契約を確認する。

主に以下を確認する。

* `200 OK`
* `400 Bad Request`
* `404 Not Found`
* `500 Internal Server Error`
* `X-User-Id`必須
* 利用者存在確認
* `assetAccountId`形式
* 資産口座存在確認
* 利用者境界
* SoftDeletes
* 現在年月判定
* 利用可能資産設定の期間境界
* 設定0件
* 設定複数件
* API用Enum変換
* 副作用なし
* レスポンス契約

---

### 2.41 AssetAccountQueryのDatabase Test

`AssetAccountQuery`について、以下を確認する。

```text id="i6yq2u"
id一致
+
user_id一致
+
deleted_at IS NULL
    ↓
取得できる
```

以下の場合は取得できないことを確認する。

* 存在しないID
* 他利用者の資産口座
* 論理削除済み資産口座

---

### 2.42 AssetAccountAvailableSettingQueryのDatabase Test

現在年月を固定し、以下を確認する。

```text id="cdg71e"
start_year_month
    <= currentYearMonth

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= currentYearMonth
)
```

境界条件として、以下を確認する。

* `start_year_month = currentYearMonth`
* `end_year_month = currentYearMonth`
* `end_year_month = 前月`
* `start_year_month = 翌月`
* `end_year_month = NULL`

---

### 2.43 設定0件のTest

現在有効な利用可能資産設定を0件にする。

期待結果：

```text id="2qw7wm"
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

となること。

また、

```text id="ksgt0x"
isAvailable = false
```

へ補完されないことを確認する。

---

### 2.44 設定複数件のTest

現在年月に有効な設定を2件以上用意する。

期待結果：

```text id="8way71"
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

となること。

最新設定を任意に返却しないことを確認する。

---

### 2.45 UseCaseのUnit Test

UseCaseでは、QueryをMockして業務フローを確認する。

正常系：

```text id="qpsj7q"
AssetAccountQuery
    ↓
資産口座取得
    ↓
AvailableSettingQuery
    ↓
現在設定1件
    ↓
Detail Result DTO
```

資産口座不存在：

```text id="yhnojt"
AssetAccountQuery
    ↓
null
    ↓
AssetAccountNotFoundException
```

設定0件：

```text id="s6a43f"
AvailableSettingQuery
    ↓
0件
    ↓
AssetAccountAvailableSettingNotFoundException
```

設定複数件：

```text id="7kzzp1"
AvailableSettingQuery
    ↓
2件
    ↓
AssetAccountAvailableSettingConflictException
```

---

### 2.46 時刻依存テスト

現在年月判定では、テスト実行時刻を固定する。

例えば、

```text id="udf7ie"
2026-08-15
```

に固定し、

```text id="uq5jrm"
currentYearMonth = 2026-08
```

として利用可能資産設定を判定する。

Laravelの時刻固定機能またはClockのTest Doubleを使用し、実行日によってテスト結果が変わらないようにする。

---

### 2.47 API ResourceのTest

正常時に、以下の項目だけが`data`へ含まれることを確認する。

```json id="oj1k4l"
{
  "id": "10",
  "name": "証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "isAvailable": true,
  "startYearMonth": "2026-08",
  "isEnabled": true
}
```

以下が含まれないことを確認する。

* `userId`
* `user_id`
* `deletedAt`
* `createdAt`
* `updatedAt`
* `assetAccountAvailableSettingId`
* `asset_account_id`
* `start_year_month`
* `end_year_month`
* DB内部の数値コード

---

### 2.48 副作用なしのTest

ACC-003実行前後で、以下のテーブルが変更されないことを確認する。

```text id="1pmhv3"
asset_accounts
asset_account_available_settings
```

特に、

```text id="slz8dh"
updated_at
```

が詳細取得によって更新されないことを確認する。

---

### 3 関連ドキュメント

- [ACC-003 API詳細設計](../../../api/details/asset-accounts/acc-003-detail.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [資産口座 Laravelアーキテクチャ設計](./README.md)
- [ACC-003 テスト設計](../../../tests/asset-accounts/acc-003-detail.md)
- [資産口座 テスト設計](../../../tests/asset-accounts/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
