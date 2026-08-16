# ACC-002 資産口座登録

1. 概要

本ドキュメントでは、
ACC-002 資産口座登録APIをLaravelで実装する際の、アーキテクチャおよび責務分離方針を定義する。

ACC-002では、操作対象利用者に帰属する資産口座を新規登録する。

資産口座登録時には、資産口座名、資産種別、残高記録単位、利用開始年月を登録するとともに、登録時点の利用可能資産区分を初期設定として登録する。

そのため、本APIではasset_accountsへの資産口座登録と、asset_account_available_settingsへの初期利用可能資産設定登録を同一ユースケースとして扱う。

両テーブルへの登録は一連の処理として実行し、途中で一方のみが登録された状態を残さないよう、トランザクションによって整合性を保証する。

Laravel実装では、HTTPリクエストの受付から入力値検証、利用者コンテキストの取得、資産口座名の重複確認、資産口座および初期利用可能資産設定の登録、APIレスポンスの生成までを単一のクラスへ集約せず、各責務を分離する。

概念的な処理構成は、以下とする。

Route
    ↓
Middleware
    ↓
FormRequest
    ↓
Action
    ↓
UseCase
    ├─ Query
    └─ Repository
         ├─ asset_accounts
         └─ asset_account_available_settings
    ↓
Result DTO
    ↓
Responder
    ↓
API Resource
    ↓
HTTP Response

FormRequestは、資産口座登録に必要な入力値の形式検証を担当する。

Actionは、検証済みのリクエストおよび利用者コンテキストを受け取り、UseCaseの呼び出しを担当する。

UseCaseは、資産口座登録に必要な処理を統括し、Queryを利用して同一利用者内の資産口座名重複を確認したうえで、トランザクション内でRepositoryを利用して資産口座および初期利用可能資産設定を登録する。

Queryは、論理削除済み資産口座を含めた同名資産口座の存在確認を担当する。

Repositoryは、asset_accountsおよびasset_account_available_settingsへの登録処理を担当し、検索やHTTPレスポンス生成の責務を持たない。

ResponderおよびAPI Resourceは、UseCaseから受け取った登録結果をAPI共通方針に従ったレスポンス形式へ変換する。

これにより、HTTP層、入力値検証、ユースケース、データ参照、データ更新、トランザクション制御、レスポンス生成の責務を明確に分離し、登録処理の整合性を保証するとともに、テスト容易性および保守性を確保する。

---

## 2. Laravel実装方針

ACC-002では、Action、FormRequest、UseCase、Query、Repository、DTO、API Resource、Responderを分離して実装する。

概念的な構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
FormRequest
    ↓
Action
    ↓
UseCase
    ├─ AssetAccountQuery
    ├─ AssetAccountRepository
    └─ AssetAccountAvailableSettingRepository
    ↓
Create Result DTO
    ↓
API Resource
    ↓
Responder
```

ACC-002では、

```text
asset_accounts
+
asset_account_available_settings
```

の2テーブルを同一ユースケースで登録する。

Actionへ、以下を直接記述しない。

* 入力値検証
* 利用者境界判定
* 資産口座名重複確認
* Enum変換
* トランザクション制御
* 資産口座登録
* 初期利用可能資産設定登録
* レスポンス変換

---

### 2.1 Route

ACC-002は、以下のルートとして定義する。

概念例：

```php
Route::post(
    '/api/v1/asset-accounts',
    CreateAssetAccountAction::class,
);
```

ACC-001と同じURLを使用するが、HTTPメソッドによって責務を分離する。

```text
GET
/api/v1/asset-accounts
    → ACC-001 資産口座一覧取得

POST
/api/v1/asset-accounts
    → ACC-002 資産口座登録
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
FormRequest
    ↓
Action
```

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 2.3 FormRequest

リクエストボディの入力値検証には、専用FormRequestを使用する。

概念例：

```php
final class CreateAssetAccountRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => [
                'required',
                'string',
                'max:100',
            ],

            'assetType' => [
                'required',
                'string',
                Rule::enum(
                    AssetType::class,
                ),
            ],

            'balanceRecordingUnit' => [
                'required',
                'string',
                Rule::enum(
                    BalanceRecordingUnit::class,
                ),
            ],

            'startYearMonth' => [
                'required',
                'string',
                new YearMonthRule(),
            ],

            'isAvailable' => [
                'required',
                'boolean',
            ],
        ];
    }
}
```

最大文字数やEnum定義については、テーブル定義およびAPI共通定義に従う。

---

### 2.4 FormRequestで行うこと

FormRequestでは、以下の入力形式を検証する。

```text
name
assetType
balanceRecordingUnit
startYearMonth
isAvailable
```

主に以下を確認する。

* 必須
* 型
* 最大文字数
* Enum値
* `YYYY-MM`形式
* boolean

HTTPリクエストとして不正な入力を、UseCaseへ渡さない。

---

### 2.5 FormRequestで行わないこと

FormRequestでは、以下の業務ルールを判定しない。

* 同一利用者内の資産口座名重複
* 論理削除済み資産口座との重複
* 資産口座登録
* 初期利用可能資産設定登録
* トランザクション制御
* 利用者IDの決定

これらは、UseCase、Query、Repositoryで扱う。

---

### 2.6 userIdをRequestから取得しない

ACC-002では、リクエストボディに

```text
userId
user_id
```

を定義しない。

操作対象利用者IDは、共通Middlewareによって設定された

```text
UserContext
```

から取得する。

概念例：

```php
$userId =
    $userContext->userId;
```

これにより、クライアントが任意の利用者IDを指定して資産口座を登録することを防止する。

---

### 2.7 Action

Actionは、検証済みリクエストと利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php
final class CreateAssetAccountAction
{
    public function __invoke(
        CreateAssetAccountRequest $request,
        CreateAssetAccountUseCase $useCase,
        CreateAssetAccountResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                name:
                    $request->string(
                        'name',
                    )->toString(),

                assetType:
                    AssetType::from(
                        $request->string(
                            'assetType',
                        )->toString(),
                    ),

                balanceRecordingUnit:
                    BalanceRecordingUnit::from(
                        $request->string(
                            'balanceRecordingUnit',
                        )->toString(),
                    ),

                startYearMonth:
                    $request->string(
                        'startYearMonth',
                    )->toString(),

                isAvailable:
                    $request->boolean(
                        'isAvailable',
                    ),
            );

        return $responder->created(
            $result,
        );
    }
}
```

実際には、引数が増える場合はInput DTOを使用してよい。

---

### 2.8 Actionで行わないこと

Actionでは、以下を行わない。

* 資産口座名重複確認
* `asset_accounts`検索
* `asset_accounts`登録
* `asset_account_available_settings`登録
* transaction制御
* SQL生成
* レスポンス配列生成
* DB保存用数値コードの直接操作

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 2.9 Input DTO

ActionからUseCaseへ複数の入力値を渡す場合は、専用Input DTOを使用してよい。

概念例：

```php
final readonly class CreateAssetAccountInput
{
    public function __construct(
        public int $userId,
        public string $name,
        public AssetType $assetType,
        public BalanceRecordingUnit $balanceRecordingUnit,
        public string $startYearMonth,
        public bool $isAvailable,
    ) {
    }
}
```

概念的には、

```php
$input =
    new CreateAssetAccountInput(
        userId:
            $userContext->userId,

        name:
            $request->string(
                'name',
            )->toString(),

        assetType:
            AssetType::from(
                $request->string(
                    'assetType',
                )->toString(),
            ),

        balanceRecordingUnit:
            BalanceRecordingUnit::from(
                $request->string(
                    'balanceRecordingUnit',
                )->toString(),
            ),

        startYearMonth:
            $request->string(
                'startYearMonth',
            )->toString(),

        isAvailable:
            $request->boolean(
                'isAvailable',
            ),
    );
```

とする。

---

### 2.10 Enum

`assetType`および`balanceRecordingUnit`は、PHP Enumとして表現する。

概念例：

```php
enum AssetType: string
{
    case CASH = 'CASH';
    case BANK = 'BANK';
    case SECURITIES = 'SECURITIES';
    case IDECO = 'IDECO';
    case CORPORATE_DC = 'CORPORATE_DC';
    case OTHER = 'OTHER';
}
```

```php
enum BalanceRecordingUnit: string
{
    case ACCOUNT = 'ACCOUNT';
    case HOLDING = 'HOLDING';
}
```

データベースへ`smallint`などで保存する場合は、保存用コードへの変換責務をModel Cast、Mapper、またはEnum側へ集約する。

ActionやUseCase内にマジックナンバーを直接記述しない。

---

### 2.11 YearMonth

`startYearMonth`は、単なる自由文字列としてUseCase内で扱い続けず、必要に応じて年月を表すValue Objectへ変換してよい。

概念例：

```php
$startYearMonth =
    YearMonth::fromString(
        $input->startYearMonth,
    );
```

Value Objectを採用する場合は、以下を集約できる。

* `YYYY-MM`形式保証
* 年月比較
* DB保存形式への変換

Phase1で過剰な抽象化になる場合は、FormRequestで形式保証したstringとして扱ってもよい。

---

### 2.12 UseCase

ACC-002の業務処理全体を担当する。

主な処理は、以下とする。

1. 入力値を受け取る
2. 同一利用者内の資産口座名重複を確認する
3. 重複している場合は業務例外を送出する
4. トランザクションを開始する
5. 資産口座を登録する
6. 新規資産口座IDを取得する
7. 初期利用可能資産設定を登録する
8. Create Result DTOを生成する
9. トランザクションをコミットする
10. DTOを返却する

概念的には、以下とする。

```text
CreateAssetAccountInput
    ↓
AssetAccountQuery
    ↓
同名重複確認
    ↓
DB::transaction
    ↓
AssetAccountRepository
    ↓
asset_accounts INSERT
    ↓
AssetAccountAvailableSettingRepository
    ↓
asset_account_available_settings INSERT
    ↓
Create Result DTO
```

---

### 2.13 資産口座名の重複確認

UseCaseでは、Queryを使用して同一利用者内に同名資産口座が存在しないことを確認する。

概念例：

```php
$exists =
    $this->assetAccountQuery
        ->existsByUserAndNameIncludingDeleted(
            userId:
                $input->userId,

            name:
                $input->name,
        );

if ($exists) {
    throw new
        AssetAccountNameAlreadyExistsException();
}
```

論理削除済み資産口座も重複確認対象とする。

---

### 2.14 AssetAccountQuery

資産口座名の重複確認を担当する。

概念的な検索条件は、以下とする。

```text
user_id
    = 操作対象利用者ID

AND

name
    = 登録する資産口座名
```

SoftDeletesを使用している場合でも、重複確認では論理削除済み資産口座を含める。

概念例：

```php
return AssetAccount::query()
    ->withTrashed()
    ->where(
        'user_id',
        $userId,
    )
    ->where(
        'name',
        $name,
    )
    ->exists();
```

---

### 2.15 QueryとRepositoryを分離する

ACC-002では、

```text
AssetAccountQuery
    → 重複確認・参照

AssetAccountRepository
    → 資産口座登録

AssetAccountAvailableSettingRepository
    → 初期利用可能資産設定登録
```

と責務を分離する。

QueryへINSERT処理を持たせず、Repositoryへ検索条件の組み立てを集中させない。

---

### 2.16 トランザクション

ACC-002では、UseCaseがトランザクション境界を管理する。

概念例：

```php
$result =
    DB::transaction(
        function () use (
            $input,
        ): CreateAssetAccountResult {
            // asset account登録
            // 初期available setting登録
            // DTO生成
        },
    );
```

Repositoryごとに個別のtransactionを開始しない。

---

### 2.17 AssetAccountRepository

資産口座の新規登録を担当する。

概念的なインターフェースは、以下とする。

```php
interface AssetAccountRepository
{
    public function create(
        int $userId,
        string $name,
        AssetType $assetType,
        BalanceRecordingUnit $balanceRecordingUnit,
        string $startYearMonth,
    ): AssetAccount;
}
```

実装では、`asset_accounts`へ新規レコードを登録する。

---

### 2.18 asset_accounts登録

概念例：

```php
$assetAccount =
    new AssetAccount();

$assetAccount->user_id =
    $userId;

$assetAccount->name =
    $name;

$assetAccount->asset_type =
    $assetType;

$assetAccount->balance_recording_unit =
    $balanceRecordingUnit;

$assetAccount->start_year_month =
    $startYearMonth;

$assetAccount->save();
```

実際のEnum Cast方法は、Model定義に従う。

---

### 2.19 Mass Assignment

ACC-002では、以下のようにHTTPリクエスト配列をそのままModelへ渡さない。

```php
AssetAccount::create(
    $request->all(),
);
```

特に、

```text
user_id
deleted_at
```

などのクライアント指定を防止するため、登録値は明示的に設定する。

---

### 2.20 user_id

`asset_accounts.user_id`には、必ず

```text
UserContext.userId
```

を設定する。

Requestに含まれる未知の`userId`や`user_id`を参照しない。

---

### 2.21 deleted_at

新規登録時は、SoftDeletesの通常動作に従い、

```text
deleted_at = NULL
```

とする。

Repositoryから明示的に`deleted_at`を設定する必要はない。

---

### 2.22 AssetAccountAvailableSettingRepository

初期利用可能資産設定の登録を担当する。

概念的なインターフェースは、以下とする。

```php
interface AssetAccountAvailableSettingRepository
{
    public function createInitial(
        int $assetAccountId,
        string $startYearMonth,
        bool $isAvailable,
    ): AssetAccountAvailableSetting;
}
```

---

### 2.23 初期利用可能資産設定登録

概念例：

```php
$setting =
    new AssetAccountAvailableSetting();

$setting->asset_account_id =
    $assetAccount->id;

$setting->start_year_month =
    $input->startYearMonth;

$setting->end_year_month =
    null;

$setting->is_available =
    $input->isAvailable;

$setting->save();
```

資産口座登録時は、必ず1件の初期設定を作成する。

---

### 2.24 start_year_month

初期利用可能資産設定の

```text
start_year_month
```

には、必ず資産口座の

```text
start_year_month
```

と同じ値を使用する。

概念的には、

```php
$setting->start_year_month =
    $assetAccount->start_year_month;
```

としてもよい。

クライアントから初期設定専用の開始年月を受け取らない。

---

### 2.25 end_year_month

初期利用可能資産設定の

```text
end_year_month
```

は、

```text
null
```

とする。

ACC-002で終了年月を指定させない。

---

### 2.26 is_available

Requestの

```text
isAvailable
```

は、

```text
asset_account_available_settings.is_available
```

へ保存する。

`asset_accounts`へ同じフラグを保存しない。

---

### 2.27 登録順序

トランザクション内では、以下の順序で登録する。

```text
asset_accounts
    ↓
生成されたid取得
    ↓
asset_account_available_settings
```

初期利用可能資産設定は、新規資産口座IDが必要であるため、順序を逆にしない。

---

### 2.28 初期設定登録失敗時

初期利用可能資産設定の登録に失敗した場合は、例外を送出してトランザクションをロールバックする。

その結果、

```text
asset_accounts
    → 登録されない

asset_account_available_settings
    → 登録されない
```

状態へ戻す。

---

### 2.29 同名資産口座のUNIQUE制約

アプリケーション側の事前重複確認だけでなく、データベース側でも

```text
user_id
+
name
```

の一意性を保証する。

これにより、同時リクエストによる重複登録を防止する。

---

### 2.30 UNIQUE制約違反の変換

同時登録によってUNIQUE制約違反が発生した場合は、PostgreSQL例外をそのまま返却しない。

概念的には、

```text
unique violation
    ↓
AssetAccountNameAlreadyExistsException
    ↓
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

制約名やSQLをクライアントへ公開しない。

---

### 2.31 ロック

ACC-002では、新規レコードを作成するため、対象となる資産口座行が事前に存在しない。

そのため、同名登録防止のための

```text
lockForUpdate()
```

は使用しない。

また、利用者行をロックして同一利用者の登録処理を直列化する方式も採用しない。

---

### 2.32 Create Result DTO

登録結果は、専用DTOとして表現する。

概念例：

```php
final readonly class CreateAssetAccountResult
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

`isEnabled`は、ACC-002成功時に必ず`true`となるため、Resource側で固定値として設定してよい。

---

### 2.33 DTOへ内部Modelを持たせない

以下のように、Eloquent ModelをそのままDTOへ保持しない。

```php
final readonly class CreateAssetAccountResult
{
    public function __construct(
        public AssetAccount $assetAccount,
        public AssetAccountAvailableSetting $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけをDTOへ保持する。

---

### 2.34 UseCaseの戻り値

概念例：

```php
return new CreateAssetAccountResult(
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

ActionへEloquent Modelをそのまま返却しない。

---

### 2.35 API Resource

Create Result DTOを、専用API Resourceによってレスポンス形式へ変換する。

概念例：

```php
final class CreateAssetAccountResource
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

APIフィールド名は、API共通方針に従ってcamelCaseとする。

---

### 2.36 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

* `user_id`
* `deleted_at`
* `created_at`
* `updated_at`
* `asset_account_available_settings.id`
* `asset_account_available_settings.asset_account_id`
* `asset_account_available_settings.start_year_month`
* `asset_account_available_settings.end_year_month`
* DB内部の数値コード

Eloquent Modelを

```php
return $assetAccount->toArray();
```

のように直接返却しない。

---

### 2.37 Responder

Responderは、Create Result DTOを`201 Created`レスポンスへ変換する。

概念例：

```php
final class CreateAssetAccountResponder
{
    public function created(
        CreateAssetAccountResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new CreateAssetAccountResource(
                        $result,
                    ),
            ],
            Response::HTTP_CREATED,
        );
    }
}
```

`requestId`などの共通Envelope項目は、API共通レスポンス処理に従う。

---

### 2.38 Responderで行わないこと

Responderでは、以下を行わない。

* 入力値検証
* 資産口座名重複確認
* 利用者境界判定
* transaction制御
* 資産口座登録
* 初期利用可能資産設定登録
* Enum変換
* DB検索

Responderは、生成済みDTOをHTTPレスポンスへ変換することに責務を限定する。

---

### 2.39 例外変換

主な例外変換は、以下とする。

| 内部状態            | 独自エラーコード                            |
| --------------- | ----------------------------------- |
| `X-User-Id`未指定  | `USER_CONTEXT_REQUIRED`             |
| `X-User-Id`形式不正 | `INVALID_USER_ID`                   |
| 利用者不存在          | `USER_NOT_FOUND`                    |
| Request入力値不正    | `VALIDATION_ERROR`                  |
| 同名資産口座存在        | `ASSET_ACCOUNT_NAME_ALREADY_EXISTS` |
| UNIQUE制約による同名競合 | `ASSET_ACCOUNT_NAME_ALREADY_EXISTS` |
| 想定外例外           | `INTERNAL_SERVER_ERROR`             |

---

### 2.40 AssetAccountNameAlreadyExistsException

同一利用者内に同名資産口座が存在する場合は、業務例外を送出する。

概念例：

```php
if ($exists) {
    throw new
        AssetAccountNameAlreadyExistsException();
}
```

API共通Exception Handlerで、

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

---

### 2.41 ValidationException

FormRequestによる入力値不正は、Laravelのバリデーション例外をAPI共通形式へ変換する。

概念的には、

```text
ValidationException
    ↓
400 Bad Request
VALIDATION_ERROR
```

とする。

HTTPステータスは、API共通方針の定義を優先する。

---

### 2.42 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

トランザクション中に例外が発生した場合は、Laravelのtransaction機構によってロールバックする。

レスポンスへ、以下を含めない。

* SQL
* PostgreSQL内部エラー
* UNIQUE制約名
* テーブル名
* カラム名
* Laravel内部例外メッセージ
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

---

### 2.43 ログ

ACC-002では、必要に応じて以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
httpStatus
errorCode
```

`apiId`は、

```text
ACC-002
```

とする。

正常時には、必要に応じて新規登録した

```text
assetAccountId
```

を記録してよい。

ただし、不要な資産情報をログへ出力しない。

---

### 2.44 キャッシュ

Phase1では、ACC-002専用のサーバー側アプリケーションキャッシュを使用しない。

資産口座登録成功後は、DBが最新状態となる。

React側でTanStack Queryなどを使用する場合は、ACC-001等の関連Query Cacheを無効化する。

---

### 2.45 テスト実装方針

Laravel側では、Feature Testを中心としてACC-002のAPI契約と登録フローを確認する。

主に以下を確認する。

* `201 Created`
* `400 Bad Request`
* `404 Not Found`
* `409 Conflict`
* `500 Internal Server Error`
* `X-User-Id`必須
* 利用者存在確認
* `name`
* `assetType`
* `balanceRecordingUnit`
* `startYearMonth`
* `isAvailable`
* 資産口座名重複
* 論理削除済み同名資産口座
* 他利用者の同名資産口座
* 資産口座登録
* 初期利用可能資産設定登録
* transaction
* rollback
* 同時登録
* UNIQUE制約
* レスポンス契約

---

### 2.46 FormRequestのTest

FormRequestについて、以下を確認する。

```text
name
    required
    string
    max length

assetType
    required
    enum

balanceRecordingUnit
    required
    enum

startYearMonth
    required
    YYYY-MM

isAvailable
    required
    boolean
```

特に、

```text
isAvailable = false
```

を正常値として確認する。

---

### 2.47 AssetAccountQueryのDatabase Test

重複確認Queryについて、以下を確認する。

```text
同一user_id
+
同一name
+
有効
    ↓
存在あり
```

```text
同一user_id
+
同一name
+
論理削除済み
    ↓
存在あり
```

```text
別user_id
+
同一name
    ↓
操作対象利用者については存在なし
```

論理削除済みデータを正しく重複対象へ含めることを確認する。

---

### 2.48 AssetAccountRepositoryのDatabase Test

資産口座登録について、以下を確認する。

* `user_id`が正しく設定される
* `name`が正しく設定される
* `asset_type`が正しく保存される
* `balance_recording_unit`が正しく保存される
* `start_year_month`が正しく保存される
* `deleted_at = NULL`となる
* `created_at`が設定される
* `updated_at`が設定される

---

### 2.49 AssetAccountAvailableSettingRepositoryのDatabase Test

初期利用可能資産設定について、以下を確認する。

* `asset_account_id`が新規資産口座IDと一致する
* `start_year_month`が資産口座の利用開始年月と一致する
* `end_year_month = NULL`となる
* `is_available = true`を保存できる
* `is_available = false`を保存できる

---

### 2.50 UseCaseのUnit Test

UseCaseでは、QueryとRepositoryをMockし、業務フローを確認する。

正常系：

```text
重複なし
    ↓
AssetAccountRepository::create()
    ↓
AvailableSettingRepository::createInitial()
    ↓
Create Result DTO
```

異常系：

```text
重複あり
    ↓
AssetAccountNameAlreadyExistsException
```

重複時に、以下が呼び出されないことを確認する。

```text
AssetAccountRepository::create()
AssetAccountAvailableSettingRepository::createInitial()
```

---

### 2.51 トランザクションテスト

初期利用可能資産設定登録時に意図的に例外を発生させる。

以下を確認する。

```text
asset_accounts INSERT
    ↓
available_settings INSERT失敗
    ↓
ROLLBACK
```

最終的に、

```text
asset_accounts
    → 新規登録なし

asset_account_available_settings
    → 新規登録なし
```

となることを確認する。

---

### 2.52 同時登録テスト

可能であれば、Integration TestまたはDatabase Testで同一利用者・同一名称の並行登録を確認する。

以下を確認する。

* 同名資産口座が2件登録されないこと
* 1件のみ成功すること
* 競合側が`409 Conflict`相当になること
* `ASSET_ACCOUNT_NAME_ALREADY_EXISTS`へ変換されること
* 孤立した初期利用可能資産設定が残らないこと

---

### 2.53 API ResourceのTest

正常時に、以下の項目だけが`data`へ含まれることを確認する。

```json
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
* `endYearMonth`
* DB内部の数値コード

---

### 3 関連ドキュメント

- [ACC-002 API詳細設計](../../../api/details/asset-accounts/acc-002-create.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [資産口座 Laravelアーキテクチャ設計](./README.md)
- [ACC-002 テスト設計](../../../tests/asset-accounts/acc-002-create.md)
- [資産口座 テスト設計](../../../tests/asset-accounts/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
