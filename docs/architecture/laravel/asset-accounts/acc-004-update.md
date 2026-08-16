# ACC-004 資産口座更新

## 1. 概要

本ドキュメントでは、
ACC-004 資産口座更新APIをLaravelで実装する際の、アーキテクチャおよび責務分離方針を定義する。

ACC-004では、操作対象利用者に帰属する指定された資産口座の通常属性を部分更新する。

更新対象は、資産口座名および資産種別に限定し、利用者、残高記録単位、利用開始年月、利用可能資産区分、利用状態は更新しない。

利用可能資産区分の変更は履歴管理を伴う別の業務操作としてACC-006で扱い、資産口座の無効化についても通常属性の更新とは分離してACC-005で扱う。

そのため、本APIによる更新対象は`asset_accounts`の指定された1件に限定し、`asset_account_available_settings`を含む関連データは更新しない。

Laravel実装では、HTTPリクエストの受付から入力値検証、利用者コンテキストの取得、更新対象資産口座の確認、現在有効な利用可能資産設定の確認、資産口座名の重複確認、部分更新、APIレスポンスの生成までを単一のクラスへ集約せず、各責務を分離する。

概念的な処理構成は、以下とする。

```text id="x51w4k"
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
    ├─ AssetAccountAvailableSettingQuery
    └─ AssetAccountRepository
    ↓
Result DTO
    ↓
Responder
    ↓
API Resource
    ↓
HTTP Response
```

FormRequestは、更新対象項目の入力形式、および少なくとも1項目が指定されていることの検証を担当する。

Actionは、検証済みのリクエスト、パスパラメータおよび利用者コンテキストを受け取り、UseCaseの呼び出しを担当する。

UseCaseは、資産口座更新に必要な処理を統括し、Queryを利用して操作対象利用者に帰属する有効な資産口座および現在有効な利用可能資産設定を確認する。

資産口座名が指定された場合は、論理削除済み資産口座を含めて同一利用者内の名前重複を確認し、問題がなければRepositoryを利用して指定された項目のみを更新する。

Repositoryは、`asset_accounts`の更新処理を担当し、未指定項目や更新対象外のカラムを変更しない。

ResponderおよびAPI Resourceは、UseCaseから受け取った更新結果をAPI共通方針に従ったレスポンス形式へ変換する。

これにより、HTTP層、入力値検証、ユースケース、データ参照、データ更新、レスポンス生成の責務を明確に分離するとともに、更新範囲を資産口座の通常属性へ限定し、関連データへの意図しない副作用を防止する。


---

## 2. Laravel実装方針

ACC-004では、Action、FormRequest、UseCase、Query、Repository、DTO、API Resource、Responderを分離して実装する。

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
    ├─ AssetAccountAvailableSettingQuery
    └─ AssetAccountRepository
    ↓
Update Result DTO
    ↓
API Resource
    ↓
Responder
```

ACC-004では、`asset_accounts`の通常属性だけを更新する。

更新対象は、

```text
name
asset_type
```

に限定する。

一方、

```text
balance_recording_unit
start_year_month
deleted_at
```

および

```text
asset_account_available_settings
```

は更新しない。

---

### 2.1 Route

ACC-004は、以下のルートとして定義する。

概念例：

```php
Route::patch(
    '/api/v1/asset-accounts/{assetAccountId}',
    UpdateAssetAccountAction::class,
);
```

ACC-003とは同じパスを使用するが、HTTPメソッドによって責務を分離する。

```text
GET
/api/v1/asset-accounts/{assetAccountId}
    → ACC-003 資産口座詳細取得

PATCH
/api/v1/asset-accounts/{assetAccountId}
    → ACC-004 資産口座更新
```

---

### 2.2 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- 共通例外処理
- ログコンテキスト設定

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

リクエストボディの入力値検証には、ACC-004専用のFormRequestを使用する。

概念例：

```php
final class UpdateAssetAccountRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => [
                'sometimes',
                'required',
                'string',
                'max:100',
            ],

            'assetType' => [
                'sometimes',
                'required',
                'string',
                Rule::enum(
                    AssetType::class,
                ),
            ],
        ];
    }
}
```

ただし、`name`と`assetType`の両方が未指定の場合はバリデーションエラーとする。

---

### 2.4 空リクエストの検証

以下のような空リクエストは許可しない。

```json
{}
```

FormRequestまたは専用Validatorで、

```text
name未指定
AND
assetType未指定
```

の場合は、

```text
VALIDATION_ERROR
```

とする。

概念例：

```php
public function after(): array
{
    return [
        function (
            Validator $validator,
        ): void {
            if (
                ! $this->has('name')
                && ! $this->has('assetType')
            ) {
                $validator->errors()->add(
                    'request',
                    '少なくとも1つの更新項目を指定してください。',
                );
            }
        },
    ];
}
```

---

### 2.5 FormRequestで行うこと

FormRequestでは、以下の入力形式を検証する。

```text
name
assetType
```

主に以下を確認する。

- 項目指定有無
- 文字列型
- NULL禁止
- 空文字禁止
- 最大文字数
- Enum値
- 少なくとも1項目指定

HTTP入力として不正な値をUseCaseへ渡さない。

---

### 2.6 FormRequestで行わないこと

FormRequestでは、以下の業務ルールを扱わない。

- 資産口座存在確認
- 利用者境界確認
- 論理削除判定
- 資産口座名重複確認
- 現在利用可能資産設定確認
- DB更新
- 同時更新競合判定

これらは、UseCase、Query、Repositoryで扱う。

---

### 2.7 更新対象外項目

ACC-004では、以下を更新対象として定義しない。

```text
userId
balanceRecordingUnit
startYearMonth
isAvailable
isEnabled
deletedAt
```

API共通方針として未定義項目を拒否する場合は、これらが送信された時点で`VALIDATION_ERROR`とする。

Request全体をそのままModelへ渡す実装は行わない。

---

### 2.8 assetAccountIdの形式検証

`assetAccountId`は、ルート制約またはAPI共通のパスパラメータ検証方式で検証する。

概念例：

```php
Route::patch(
    '/api/v1/asset-accounts/{assetAccountId}',
    UpdateAssetAccountAction::class,
)
    ->where(
        'assetAccountId',
        '[1-9][0-9]*',
    );
```

正の整数形式のみを許可する。

形式不正の場合は、

```text
INVALID_ASSET_ACCOUNT_ID
```

へ変換する。

---

### 2.9 Action

Actionは、検証済みリクエスト、パスパラメータ、利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php
final class UpdateAssetAccountAction
{
    public function __invoke(
        string $assetAccountId,
        UpdateAssetAccountRequest $request,
        UpdateAssetAccountUseCase $useCase,
        UpdateAssetAccountResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $input =
            new UpdateAssetAccountInput(
                userId:
                    $userContext->userId,

                assetAccountId:
                    (int) $assetAccountId,

                name:
                    $request->has('name')
                        ? $request->string(
                            'name',
                        )->toString()
                        : null,

                assetType:
                    $request->has('assetType')
                        ? AssetType::from(
                            $request->string(
                                'assetType',
                            )->toString(),
                        )
                        : null,
            );

        $result =
            $useCase->execute(
                $input,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

---

### 2.10 Input DTO

PATCHでは、「未指定」と「指定済み」を区別する必要がある。

そのため、Input DTOでは更新可能項目をnullableとして表現してよい。

概念例：

```php
final readonly class UpdateAssetAccountInput
{
    public function __construct(
        public int $userId,
        public int $assetAccountId,
        public ?string $name,
        public ?AssetType $assetType,
    ) {
    }
}
```

ただし、ACC-004では`NULL`値そのものを更新値として許可しないため、

```text
null
    = Requestで未指定
```

という意味に限定する。

---

### 2.11 未指定とNULL入力を混同しない

例えば、

```json
{
  "name": null
}
```

はバリデーションエラーとする。

一方、

```json
{
  "assetType": "SECURITIES"
}
```

の場合の

```text
name = null
```

は、Input DTO上で「name未指定」を意味する。

Actionでは`has()`や`array_key_exists()`相当の判定を使用し、未指定とNULL入力を区別する。

---

### 2.12 Actionで行わないこと

Actionでは、以下を行わない。

- 資産口座検索
- 利用者境界判定
- 同名重複確認
- 現在年月取得
- 利用可能資産設定検索
- UPDATE
- UNIQUE制約例外処理
- レスポンス配列生成

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 2.13 UseCase

ACC-004の業務処理全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `assetAccountId`を受け取る
3. 更新対象資産口座を取得する
4. 対象不存在の場合は業務例外を送出する
5. 現在年月を取得する
6. 現在有効な利用可能資産設定が1件であることを確認する
7. `name`指定時のみ同名重複を確認する
8. 指定された項目のみ更新する
9. 更新結果DTOを生成する
10. DTOを返却する

概念的には、以下とする。

```text
UpdateAssetAccountInput
    ↓
AssetAccountQuery
    ↓
更新対象取得
    ↓
CurrentYearMonth取得
    ↓
AssetAccountAvailableSettingQuery
    ↓
現在設定1件確認
    ↓
name指定あり？
    ├─ Yes
    │    ↓
    │ AssetAccountQuery
    │ 同名重複確認
    └─ No
    ↓
AssetAccountRepository
    ↓
部分更新
    ↓
Update Result DTO
```

---

### 2.14 AssetAccountQuery

更新対象資産口座の取得は、Queryへ委譲する。

概念例：

```php
$assetAccount =
    $this->assetAccountQuery
        ->findActiveByIdAndUser(
            assetAccountId:
                $input->assetAccountId,

            userId:
                $input->userId,
        );
```

概念的な条件は、以下とする。

```text
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

### 2.15 利用者境界をQueryへ含める

以下のように、IDだけで取得してから所有利用者を判定する方式は基本としない。

```php
$assetAccount =
    AssetAccount::find(
        $assetAccountId,
    );
```

対象取得時点で、

```text
assetAccountId
+
userId
+
有効状態
```

を条件へ含める。

---

### 2.16 SoftDeletes

`AssetAccount` Modelでは、Laravelの

```php
use SoftDeletes;
```

を使用する。

ACC-004の更新対象取得では、

```php
withTrashed()
```

を使用しない。

論理削除済み資産口座は更新対象外とする。

---

### 2.17 ASSET_ACCOUNT_NOT_FOUND

対象資産口座を取得できない場合は、業務例外を送出する。

概念例：

```php
if ($assetAccount === null) {
    throw new
        AssetAccountNotFoundException();
}
```

以下を同じ例外へ集約する。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

最終的に、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

へ変換する。

---

### 2.18 現在利用可能資産設定をUPDATE前に確認する

ACC-004の成功レスポンスでは、現在の

```text
isAvailable
```

を返却する。

そのため、資産口座UPDATE後に初めて利用可能資産設定不整合を検出する構成は避ける。

概念的には、

```text
更新対象取得
    ↓
現在設定確認
    ↓
設定1件
    ↓
更新
```

とする。

これにより、

```text
asset_accounts更新成功
    ↓
現在設定取得失敗
    ↓
500レスポンス
```

という分かりにくい状態を避ける。

---

### 2.19 CurrentYearMonth

現在利用可能資産設定判定には、サーバー側の現在年月を使用する。

概念例：

```php
$currentYearMonth =
    now()->format(
        'Y-m',
    );
```

ACC-003と同様に、必要に応じてClockやDateProviderへ時刻依存を分離してよい。

---

### 2.20 AssetAccountAvailableSettingQuery

現在有効な利用可能資産設定の取得は、専用Queryへ委譲する。

概念例：

```php
$settings =
    $this->availableSettingQuery
        ->findCurrentByAssetAccount(
            assetAccountId:
                $assetAccount->id,

            yearMonth:
                $currentYearMonth,
        );
```

概念的な条件は、以下とする。

```text
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

### 2.21 利用可能資産設定件数

現在有効な設定は、1件であることを必須とする。

概念例：

```php
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingNotFoundException();
}

if ($settings->count() > 1) {
    throw new
        AssetAccountAvailableSettingConflictException();
}
```

0件の場合に`false`へ補完せず、複数件の場合に任意の1件を採用しない。

---

### 2.22 利用可能資産設定は更新しない

ACC-004では、

```text
asset_account_available_settings
```

に対する

```text
INSERT
UPDATE
DELETE
```

を行わない。

`isAvailable`変更はACC-006へ委譲する。

---

### 2.23 name指定時のみ重複確認する

`name`がRequestで指定された場合のみ、同名重複確認を行う。

概念例：

```php
if ($input->name !== null) {
    $exists =
        $this->assetAccountQuery
            ->existsDuplicateNameIncludingDeleted(
                userId:
                    $input->userId,

                name:
                    $input->name,

                exceptAssetAccountId:
                    $input->assetAccountId,
            );

    if ($exists) {
        throw new
            AssetAccountNameAlreadyExistsException();
    }
}
```

`assetType`だけの更新では、不要な名前重複Queryを実行しない。

---

### 2.24 論理削除済みも重複確認対象とする

重複確認では、SoftDeletesの通常スコープを外し、論理削除済み資産口座も検索対象に含める。

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
    ->whereKeyNot(
        $exceptAssetAccountId,
    )
    ->exists();
```

実際のメソッドは、Laravelバージョンに合わせて適切な記述とする。

---

### 2.25 更新対象自身を除外する

名前重複確認では、更新対象自身を除外する。

概念的には、

```text
id != assetAccountId
```

を条件へ含める。

これにより、現在の名前と同じ名前を再指定しても重複エラーとならない。

---

### 2.26 AssetAccountRepository

資産口座更新は、Repositoryへ委譲する。

概念的なインターフェースは、以下とする。

```php
interface AssetAccountRepository
{
    public function update(
        AssetAccount $assetAccount,
        ?string $name,
        ?AssetType $assetType,
    ): AssetAccount;
}
```

ただし、引数が増える場合は更新専用DTOを使用してよい。

---

### 2.27 Repositoryで部分更新する

Repositoryでは、指定された項目だけを更新する。

概念例：

```php
public function update(
    AssetAccount $assetAccount,
    ?string $name,
    ?AssetType $assetType,
): AssetAccount {
    if ($name !== null) {
        $assetAccount->name =
            $name;
    }

    if ($assetType !== null) {
        $assetAccount->asset_type =
            $assetType;
    }

    $assetAccount->save();

    return $assetAccount;
}
```

未指定項目をNULLやデフォルト値へ置き換えない。

---

### 2.28 Mass Assignmentを使用しない

以下のような処理は行わない。

```php
$assetAccount->update(
    $request->all(),
);
```

更新対象を

```text
name
asset_type
```

へ限定するため、明示的に値を設定する。

---

### 2.29 更新対象外カラムを変更しない

Repositoryでは、以下を変更しない。

```text
user_id
balance_recording_unit
start_year_month
deleted_at
```

また、別Repositoryを呼び出して

```text
asset_account_available_settings
```

を更新しない。

---

### 2.30 Enum Cast

`asset_type`は、PHP EnumまたはEloquent Castとして扱う。

概念例：

```php
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

UseCaseやRepositoryへDB内部の数値コードを直接持ち込まない。

---

### 2.31 同一値更新

現在値と同一値が指定された場合は、業務エラーとしない。

Eloquentでは、Dirty判定により実際の変更がない場合にUPDATEが発行されないことがある。

概念的には、

```php
$assetAccount->save();
```

を使用し、Laravel標準のDirty判定に従ってよい。

---

### 2.32 updated_at

実際にEloquent上で変更が発生した場合は、

```text
updated_at
```

が通常どおり更新される。

クライアントから`updatedAt`を指定させない。

同一値更新でDirtyでない場合は、`updated_at`が変わらなくてもよい。

---

### 2.33 トランザクション

Phase1では、ACC-004専用の

```php
DB::transaction()
```

は必須としない。

更新対象が

```text
asset_accounts 1件
```

だけであり、関連テーブルを更新しないためである。

現在設定確認や名前重複確認はUPDATE前に行い、単一UPDATEで完結させる。

---

### 2.34 lockForUpdateを使用しない

ACC-004では、Phase1で

```php
lockForUpdate()
```

を使用しない。

同一資産口座への更新競合はLast Write Winsとなる可能性を許容する。

また、別資産口座間の同名競合はUNIQUE制約で保証する。

---

### 2.35 UNIQUE制約

同一利用者内の資産口座名は、DB側でも

```text
user_id
+
name
```

で一意性を保証する。

アプリケーション側の事前重複確認を同時実行した複数リクエストが通過しても、最終的にDB制約で重複を防止する。

---

### 2.36 UNIQUE制約違反の変換

並行更新などによってUNIQUE制約違反が発生した場合は、PostgreSQL例外をそのまま返却しない。

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

制約名やSQLSTATEはレスポンスへ公開しない。

---

### 2.37 Update Result DTO

更新結果は、専用DTOとして表現する。

概念例：

```php
final readonly class UpdateAssetAccountResult
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

`isEnabled`は、ACC-004成功時には必ず`true`となるため、Resource側で固定値として設定してよい。

---

### 2.38 DTOへEloquent Modelを保持しない

以下のようなDTOは基本としない。

```php
final readonly class UpdateAssetAccountResult
{
    public function __construct(
        public AssetAccount $assetAccount,
        public AssetAccountAvailableSetting $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

---

### 2.39 UseCaseの戻り値

概念例：

```php
return new UpdateAssetAccountResult(
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

Eloquent ModelをActionへ直接返却しない。

---

### 2.40 API Resource

Update Result DTOを、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php
final class UpdateAssetAccountResource
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

### 2.41 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

- `user_id`
- `deleted_at`
- `created_at`
- `updated_at`
- `asset_account_available_settings.id`
- `asset_account_available_settings.asset_account_id`
- `asset_account_available_settings.start_year_month`
- `asset_account_available_settings.end_year_month`
- DB内部コード

また、以下も返却しない。

- 保有商品一覧
- 月末資産残高
- 商品別月末評価額
- 月末資産状況
- 資産推移

---

### 2.42 Responder

Responderは、Update Result DTOを`200 OK`レスポンスへ変換する。

概念例：

```php
final class UpdateAssetAccountResponder
{
    public function ok(
        UpdateAssetAccountResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new UpdateAssetAccountResource(
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

### 2.43 Responderで行わないこと

Responderでは、以下を行わない。

- 資産口座検索
- 利用者境界判定
- 資産口座名重複確認
- 現在年月取得
- 利用可能資産設定取得
- DB更新
- UNIQUE制約処理
- 業務例外判定

Responderは、生成済みDTOをHTTPレスポンスへ変換することに責務を限定する。

---

### 2.44 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `assetAccountId`形式不正 | `INVALID_ASSET_ACCOUNT_ID` |
| Request入力値不正 | `VALIDATION_ERROR` |
| 資産口座不存在 | `ASSET_ACCOUNT_NOT_FOUND` |
| 他利用者の資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 論理削除済み資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 同名資産口座存在 | `ASSET_ACCOUNT_NAME_ALREADY_EXISTS` |
| UNIQUE制約による同名競合 | `ASSET_ACCOUNT_NAME_ALREADY_EXISTS` |
| 現在設定0件 | `ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND` |
| 現在設定複数件 | `ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

---

### 2.45 AssetAccountNameAlreadyExistsException

同一利用者内に同名の別資産口座が存在する場合は、業務例外を送出する。

概念例：

```php
if ($exists) {
    throw new
        AssetAccountNameAlreadyExistsException();
}
```

最終的に、

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

---

### 2.46 利用可能資産設定不整合例外

現在設定が0件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

へ変換する。

現在設定が複数件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

へ変換する。

どちらも500系のデータ不整合として扱う。

---

### 2.47 ValidationException

FormRequestによる入力値不正は、API共通形式の

```text
400 Bad Request
VALIDATION_ERROR
```

へ変換する。

複数のフィールドエラーがある場合は、`error.details`へ複数件返却してよい。

---

### 2.48 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- SQL
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

---

### 2.49 ログ

ACC-004では、必要に応じて以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assetAccountId
httpStatus
errorCode
```

`apiId`は、

```text
ACC-004
```

とする。

正常時には、更新対象IDを記録してよい。

ただし、資産口座名などの不要な業務情報を通常ログへ出力しない。

---

### 2.50 データ不整合ログ

以下の場合は、調査に必要な情報を内部ログへ記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text
assetAccountId
currentYearMonth
matchedSettingCount
```

を記録してよい。

---

### 2.51 キャッシュ

Phase1では、ACC-004専用のサーバー側キャッシュを使用しない。

React側でTanStack Queryを使用する場合は、ACC-004成功後に

```text
ACC-001
資産口座一覧

ACC-003
対象資産口座詳細
```

のQuery Cacheを無効化する。

---

### 2.52 テスト実装方針

Laravel側では、Feature Testを中心としてACC-004のAPI契約と部分更新処理を確認する。

主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `404 Not Found`
- `409 Conflict`
- `500 Internal Server Error`
- `X-User-Id`必須
- `assetAccountId`形式
- `name`のみ更新
- `assetType`のみ更新
- 複数項目更新
- 空リクエスト
- NULL
- 未定義項目
- 利用者境界
- SoftDeletes
- 資産口座名重複
- 論理削除済み資産口座との名前重複
- 現在利用可能資産設定確認
- UNIQUE制約
- 同時更新
- 関連データ非更新
- レスポンス契約

---

### 2.53 FormRequestのTest

FormRequestについて、以下を確認する。

```text
name
    sometimes
    required
    string
    max length

assetType
    sometimes
    required
    enum

name未指定
+
assetType未指定
    → error
```

以下も確認する。

- `name = null`はエラー
- `assetType = null`はエラー
- 空リクエストはエラー
- `name`だけは正常
- `assetType`だけは正常

---

### 2.54 AssetAccountQueryのDatabase Test

更新対象取得について、以下を確認する。

```text
id一致
+
user_id一致
+
deleted_at IS NULL
    ↓
取得できる
```

以下は取得できないこと。

- 存在しないID
- 他利用者の資産口座
- 論理削除済み資産口座

---

### 2.55 名前重複QueryのDatabase Test

以下を確認する。

```text
同一user_id
+
同一name
+
別id
    ↓
重複
```

```text
同一user_id
+
同一name
+
論理削除済み別id
    ↓
重複
```

```text
同一user_id
+
同一name
+
同じid
    ↓
重複ではない
```

```text
別user_id
+
同一name
    ↓
重複ではない
```

---

### 2.56 AssetAccountAvailableSettingQueryのDatabase Test

現在年月を固定し、現在有効な設定が正しく取得できることを確認する。

境界条件として、以下を確認する。

- `start_year_month = currentYearMonth`
- `end_year_month = currentYearMonth`
- `end_year_month = 前月`
- `start_year_month = 翌月`
- `end_year_month = NULL`

---

### 2.57 AssetAccountRepositoryのDatabase Test

`name`だけを渡した場合に、

```text
name
    → 更新

asset_type
    → 維持
```

となることを確認する。

`assetType`だけの場合は、

```text
asset_type
    → 更新

name
    → 維持
```

となることを確認する。

また、以下が変更されないことも確認する。

```text
user_id
balance_recording_unit
start_year_month
deleted_at
```

---

### 2.58 同一値更新Test

現在値と同じ値を指定する。

以下を確認する。

- 業務エラーにならない
- 最終業務状態が変わらない
- EloquentがDirtyでない場合は正常
- `updated_at`が変わらなくても正常

---

### 2.59 UseCaseのUnit Test

QueryとRepositoryをMockし、以下を確認する。

正常系：

```text
AssetAccountQuery
    ↓
資産口座取得
    ↓
AvailableSettingQuery
    ↓
現在設定1件
    ↓
必要なら名前重複確認
    ↓
Repository::update()
    ↓
Update Result DTO
```

対象不存在：

```text
AssetAccountQuery
    ↓
null
    ↓
AssetAccountNotFoundException
```

名前重複：

```text
DuplicateQuery
    ↓
true
    ↓
AssetAccountNameAlreadyExistsException
```

設定0件：

```text
AvailableSettingQuery
    ↓
0件
    ↓
AssetAccountAvailableSettingNotFoundException
```

設定複数件：

```text
AvailableSettingQuery
    ↓
2件以上
    ↓
AssetAccountAvailableSettingConflictException
```

---

### 2.60 name未指定時のTest

`assetType`だけを更新する場合に、名前重複確認Queryが呼び出されないことを確認する。

これにより、不要なDBアクセスを防止する。

---

### 2.61 UNIQUE制約競合Test

可能であれば、Integration TestまたはDatabase Testで、同一利用者の別資産口座を同じ名前へ並行更新する。

以下を確認する。

- 同名資産口座が2件存在しないこと
- 競合側がDB制約で失敗すること
- PostgreSQL例外がそのまま公開されないこと
- `ASSET_ACCOUNT_NAME_ALREADY_EXISTS`へ変換されること

---

### 2.62 API ResourceのTest

正常時に、以下の項目だけが`data`へ含まれることを確認する。

```json
{
  "id": "10",
  "name": "メイン証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "isAvailable": true,
  "startYearMonth": "2026-08",
  "isEnabled": true
}
```

以下が含まれないことを確認する。

- `userId`
- `user_id`
- `deletedAt`
- `createdAt`
- `updatedAt`
- `assetAccountAvailableSettingId`
- `asset_account_id`
- `endYearMonth`
- DB内部コード

---

### 2.63 関連データ非更新Test

ACC-004実行前後で、以下に変更がないことを確認する。

```text
asset_account_available_settings
holding_assets
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
net_incomes
objectives
assessment_histories
```

ACC-004の副作用が対象`asset_accounts`1件に限定されていることを保証する。

---

### 3 関連ドキュメント

- [ACC-004 API詳細設計](../../../api/details/asset-accounts/acc-004-update.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [資産口座 Laravelアーキテクチャ設計](./README.md)
- [ACC-004 テスト設計](../../../tests/asset-accounts/acc-004-update.md)
- [資産口座 テスト設計](../../../tests/asset-accounts/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
