# ACC-006 利用可能資産設定登録

## 1. 概要

本ドキュメントでは、  
ACC-006 利用可能資産設定登録APIをLaravelで実装する際の、アーキテクチャおよび責務分離方針を定義する。

ACC-006では、操作対象利用者に帰属する指定された資産口座について、新しい利用可能資産設定を登録する。

利用可能資産区分は現在値を直接上書きせず、`asset_account_available_settings`によって年月単位の履歴として管理する。

そのため、本APIでは現在有効な利用可能資産設定の終了年月を新しい設定開始年月の前月へ更新したうえで、新しい利用可能資産設定を登録する。

旧設定の終了と新設定の登録は1つの業務操作として扱い、トランザクションによって両処理の整合性を保証する。

ACC-004 資産口座更新が資産口座名や資産種別などの通常属性を更新するのに対し、ACC-006では利用可能資産区分の変更履歴を管理する。また、ACC-005 利用可能資産設定履歴取得が履歴を参照するAPIであるのに対し、本APIは新しい履歴を追加する更新APIとして責務を分離する。

Laravel実装では、HTTPリクエストの受付から入力値検証、利用者コンテキストの取得、対象資産口座の確認、既存履歴の取得・整合性検証、現在設定のロック、新設定の業務条件確認、旧設定の終了、新設定の登録、APIレスポンスの生成までを単一のクラスへ集約せず、各責務を分離する。

概念的な処理構成は、以下とする。

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
    ├─ AvailableSettingHistoryValidator
    └─ AssetAccountAvailableSettingRepository
    ↓
Result DTO
    ↓
Responder
    ↓
API Resource
    ↓
HTTP Response

---

## 2. Laravel実装方針

ACC-006では、Action、FormRequest、UseCase、Query、Repository、Validator、DTO、API Resource、Responderを分離して実装する。

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
    ├─ AvailableSettingHistoryValidator
    └─ AssetAccountAvailableSettingRepository
    ↓
Create Result DTO
    ↓
API Resource
    ↓
Responder
```

ACC-006は、

```text
旧設定.end_year_month更新
+
新設定INSERT
```

という複数レコード更新を1つの業務操作として扱うため、UseCase全体をトランザクション境界とする。

---

### 2.1 Route

ACC-006は、以下のルートとして定義する。

概念例：

```php
Route::post(
    '/api/v1/asset-accounts/{assetAccountId}/available-settings',
    CreateAssetAccountAvailableSettingAction::class,
);
```

ACC-005とは同じURLを使用し、HTTPメソッドによって責務を分離する。

```text
GET
/api/v1/asset-accounts/{assetAccountId}/available-settings
    → ACC-005
      利用可能資産設定履歴取得

POST
/api/v1/asset-accounts/{assetAccountId}/available-settings
    → ACC-006
      利用可能資産設定登録
```

---

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

とする。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 2.3 FormRequest

ACC-006では、リクエストボディの入力値検証に専用FormRequestを使用する。

概念例：

```php
final class CreateAssetAccountAvailableSettingRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'startYearMonth' => [
                'required',
                'string',
                'date_format:Y-m',
            ],

            'isAvailable' => [
                'required',
                'boolean',
            ],
        ];
    }
}
```

ただし、Laravelの`boolean`ルールで許容される値がAPI契約より広い場合は、JSON booleanのみを許容する専用Ruleまたは追加検証を使用する。

---

### 2.4 JSON booleanを厳密に扱う

ACC-006では、

```text
true
false
```

だけを`isAvailable`として許可する。

以下は受け付けない。

```text
"true"
```

```text
"false"
```

```text
1
```

```text
0
```

Laravel標準の`boolean`ルールがこれらを許容する場合は、例えば専用Ruleを使用する。

概念例：

```php
final class StrictBoolean implements ValidationRule
{
    public function validate(
        string $attribute,
        mixed $value,
        Closure $fail,
    ): void {
        if (! is_bool($value)) {
            $fail(
                'booleanで指定してください。',
            );
        }
    }
}
```

---

### 2.5 startYearMonthの形式検証

`startYearMonth`は、厳密な

```text
YYYY-MM
```

形式として検証する。

例えば、

```text
2026-08
```

は正常とする。

以下は不正とする。

```text
2026-8
2026/08
202608
2026-00
2026-13
```

見た目だけでなく、実在する年月であることも確認する。

---

### 2.6 FormRequestで行うこと

FormRequestでは、HTTP入力として判断できる以下を検証する。

```text
startYearMonth
isAvailable
```

主に以下とする。

- 必須
- NULL禁止
- 型
- `YYYY-MM`形式
- 実在年月
- JSON boolean
- 未定義項目

---

### 2.7 FormRequestで行わないこと

FormRequestでは、以下のDB状態に依存する業務ルールを扱わない。

- 資産口座存在確認
- 利用者境界確認
- 論理削除判定
- 既存履歴0件判定
- 履歴整合性判定
- 現在設定特定
- 開始年月の業務妥当性
- 現在設定との`isAvailable`比較
- 行ロック
- DB更新

これらは、UseCase、Query、Validator、Repositoryで扱う。

---

### 2.8 更新対象外項目

ACC-006では、以下をRequest項目として定義しない。

```text
id
userId
assetAccountId
settingId
currentSettingId
endYearMonth
createdAt
updatedAt
```

API共通方針として未定義項目を拒否する場合は、これらが送信された時点で

```text
VALIDATION_ERROR
```

とする。

---

### 2.9 assetAccountIdの形式検証

`assetAccountId`は、ルート制約またはAPI共通のパスパラメータ検証方式で検証する。

概念例：

```php
Route::post(
    '/api/v1/asset-accounts/{assetAccountId}/available-settings',
    CreateAssetAccountAvailableSettingAction::class,
)
    ->where(
        'assetAccountId',
        '[1-9][0-9]*',
    );
```

形式不正は、

```text
INVALID_ASSET_ACCOUNT_ID
```

へ変換する。

---

### 2.10 Action

Actionは、検証済みRequest、`assetAccountId`、利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php
final class CreateAssetAccountAvailableSettingAction
{
    public function __invoke(
        string $assetAccountId,
        CreateAssetAccountAvailableSettingRequest $request,
        CreateAssetAccountAvailableSettingUseCase $useCase,
        CreateAssetAccountAvailableSettingResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $input =
            new CreateAssetAccountAvailableSettingInput(
                userId:
                    $userContext->userId,

                assetAccountId:
                    (int) $assetAccountId,

                startYearMonth:
                    $request->string(
                        'startYearMonth',
                    )->toString(),

                isAvailable:
                    $request->boolean(
                        'isAvailable',
                    ),
            );

        $result =
            $useCase->execute(
                $input,
            );

        return $responder->created(
            $result,
        );
    }
}
```

---

### 2.11 Input DTO

ActionからUseCaseへは、専用Input DTOを渡す。

概念例：

```php
final readonly class
    CreateAssetAccountAvailableSettingInput
{
    public function __construct(
        public int $userId,
        public int $assetAccountId,
        public string $startYearMonth,
        public bool $isAvailable,
    ) {
    }
}
```

RequestオブジェクトそのものをUseCaseへ渡さない。

---

### 2.12 Actionで行わないこと

Actionでは、以下を行わない。

- 資産口座検索
- 利用者境界判定
- 履歴取得
- 履歴整合性判定
- 現在設定取得
- 前月計算
- `lockForUpdate()`
- UPDATE
- INSERT
- トランザクション制御
- レスポンス配列生成

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 2.13 UseCase

ACC-006の業務処理全体を担当する。

主な処理は、以下とする。

1. Input DTOを受け取る
2. トランザクションを開始する
3. 対象資産口座を取得する
4. 対象不存在の場合は業務例外を送出する
5. 既存履歴を取得する
6. 履歴0件を確認する
7. 既存履歴整合性を検証する
8. 現在設定を`lockForUpdate()`で取得する
9. ロック後の最新状態を基準に業務条件を再確認する
10. 新設定開始年月の妥当性を確認する
11. `isAvailable`が現在設定と異なることを確認する
12. 旧設定終了年月を算出する
13. 旧設定を更新する
14. 新設定を登録する
15. Result DTOを生成する
16. コミットする
17. Result DTOを返却する

---

### 2.14 UseCaseの概念フロー

概念的には、以下とする。

```text
Input DTO
    ↓
DB::transaction
    ↓
AssetAccountQuery
    ↓
資産口座取得
    ↓
AvailableSettingQuery
    ↓
既存履歴取得
    ↓
HistoryValidator
    ↓
履歴整合性確認
    ↓
AvailableSettingQuery
    ↓
現在設定lockForUpdate
    ↓
最新状態再確認
    ↓
開始年月チェック
    ↓
isAvailable差分チェック
    ↓
前月算出
    ↓
Repository
    ├─ 旧設定終了
    └─ 新設定登録
    ↓
Result DTO
```

---

### 2.15 DB::transaction

ACC-006では、UseCase全体を

```php
DB::transaction()
```

で囲む。

概念例：

```php
return DB::transaction(
    function () use (
        $input,
    ): CreateAssetAccountAvailableSettingResult {
        // 業務処理
    },
);
```

旧設定UPDATEと新設定INSERTを原子的に扱う。

---

### 2.16 AssetAccountQuery

対象資産口座取得は、Queryへ委譲する。

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

取得条件は、

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

とする。

---

### 2.17 利用者境界をQueryへ含める

以下のようなID単独取得を基本としない。

```php
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
利用中状態
```

を条件へ含める。

---

### 2.18 SoftDeletes

`AssetAccount` Modelでは、

```php
use SoftDeletes;
```

を使用する。

ACC-006では論理削除済み資産口座を対象としないため、

```php
withTrashed()
```

を使用しない。

---

### 2.19 ASSET_ACCOUNT_NOT_FOUND

対象資産口座を取得できない場合は、

```php
throw new
    AssetAccountNotFoundException();
```

とする。

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

### 2.20 AssetAccountAvailableSettingQuery

既存履歴取得は、専用Queryへ委譲する。

概念例：

```php
$settings =
    $this->availableSettingQuery
        ->findHistoryByAssetAccount(
            assetAccountId:
                $assetAccount->id,
        );
```

履歴検証しやすいように、

```text
start_year_month ASC
```

で取得してよい。

---

### 2.21 履歴0件

履歴が0件の場合は、正常状態としない。

概念例：

```php
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingHistoryNotFoundException();
}
```

最終的に、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

へ変換する。

---

### 2.22 AvailableSettingHistoryValidator

既存履歴の整合性確認には、ACC-005と同じValidatorを再利用する。

概念的には、

```php
$this->historyValidator
    ->validate(
        assetAccountStartYearMonth:
            $assetAccount->start_year_month,

        settings:
            $settings,
    );
```

とする。

同じ業務ルールをACC-005とACC-006で重複実装しない。

---

### 2.23 Validatorの確認内容

主に以下を確認する。

```text
最古設定開始年月
    =
assetAccount.startYearMonth

各設定
startYearMonth <= endYearMonth

期間重複なし

期間欠落なし

継続中設定1件

最新設定
endYearMonth = null
```

---

### 2.24 ValidatorでDB更新しない

`AvailableSettingHistoryValidator`は、整合性判定だけを行う。

以下を行わない。

- DB検索
- UPDATE
- INSERT
- 自動修復
- ロック

QueryとRepositoryの責務を持たせない。

---

### 2.25 現在設定をロック付きで取得する

履歴整合性確認後、現在設定をトランザクション内で再取得する。

概念例：

```php
$currentSetting =
    $this->availableSettingQuery
        ->findCurrentForUpdate(
            assetAccountId:
                $assetAccount->id,
        );
```

Query内部では、

```php
->where(
    'asset_account_id',
    $assetAccountId,
)
->whereNull(
    'end_year_month',
)
->lockForUpdate()
```

を使用する。

---

### 2.26 現在設定は1件を前提とする

`findCurrentForUpdate()`では、継続中設定が1件のみであることを前提とする。

0件または複数件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ変換する。

単純に

```php
->first()
```

だけで複数件を黙って無視しない。

---

### 2.27 ロック後に再確認する

履歴整合性確認後からロック取得までの間に、別ACC-006が更新している可能性がある。

そのため、ロック取得後の現在設定を基準に、

```text
startYearMonth
isAvailable
```

の業務条件を再確認する。

---

### 2.28 startYearMonthの業務検証

新設定開始年月は、少なくとも

```text
request.startYearMonth
    >
currentSetting.startYearMonth
```

であることを確認する。

また、

```text
request.startYearMonth
    >=
assetAccount.startYearMonth
```

も満たすことを確認する。

---

### 2.29 開始年月不正

条件を満たさない場合は、

```php
throw new
    AssetAccountAvailableSettingStartYearMonthInvalidException();
```

とする。

最終的に、

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

へ変換する。

---

### 2.30 現在設定との差分確認

現在設定の

```text
is_available
```

とRequestの

```text
isAvailable
```

が異なることを確認する。

概念例：

```php
if (
    $currentSetting->is_available
    === $input->isAvailable
) {
    throw new
        AssetAccountAvailableSettingNoChangeException();
}
```

---

### 2.31 NO_CHANGE

現在設定と同じ状態の場合は、

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

へ変換する。

この場合、Repositoryを呼び出して旧設定を更新しない。

---

### 2.32 YearMonthの扱い

`startYearMonth`から前月を算出する必要がある。

年月計算を文字列操作で行わず、`CarbonImmutable`または`YearMonth` Value Objectを使用する。

概念例：

```php
$previousYearMonth =
    CarbonImmutable::createFromFormat(
        '!Y-m',
        $input->startYearMonth,
    )
        ->subMonth()
        ->format('Y-m');
```

---

### 2.33 年跨ぎを考慮する

例えば、

```text
2027-01
```

の前月は、

```text
2026-12
```

となる。

以下のような独自文字列演算は行わない。

```text
month - 1
```

---

### 2.34 YearMonth Value Object

年月処理が複数ユースケースへ広がる場合は、Value Objectを導入してよい。

概念例：

```php
final readonly class YearMonth
{
    public function __construct(
        public int $year,
        public int $month,
    ) {
    }

    public function previous(): self
    {
        // 前月返却
    }

    public function isAfter(
        self $other,
    ): bool {
        // 比較
    }

    public function toString(): string
    {
        return sprintf(
            '%04d-%02d',
            $this->year,
            $this->month,
        );
    }
}
```

Phase1では、過剰にならない範囲で採用する。

---

### 2.35 Repository

`asset_account_available_settings`の更新・登録は、Repositoryへ委譲する。

概念的なインターフェースは、以下とする。

```php
interface
    AssetAccountAvailableSettingRepository
{
    public function close(
        AssetAccountAvailableSetting $setting,
        string $endYearMonth,
    ): void;

    public function create(
        int $assetAccountId,
        string $startYearMonth,
        bool $isAvailable,
    ): AssetAccountAvailableSetting;
}
```

---

### 2.36 旧設定終了

Repositoryでは、現在設定の

```text
end_year_month
```

だけを更新する。

概念例：

```php
public function close(
    AssetAccountAvailableSetting $setting,
    string $endYearMonth,
): void {
    $setting->end_year_month =
        $endYearMonth;

    $setting->save();
}
```

以下を変更しない。

```text
asset_account_id
start_year_month
is_available
```

---

### 2.37 新設定登録

新設定は、明示的な値で登録する。

概念例：

```php
public function create(
    int $assetAccountId,
    string $startYearMonth,
    bool $isAvailable,
): AssetAccountAvailableSetting {
    $setting =
        new AssetAccountAvailableSetting();

    $setting->asset_account_id =
        $assetAccountId;

    $setting->start_year_month =
        $startYearMonth;

    $setting->end_year_month =
        null;

    $setting->is_available =
        $isAvailable;

    $setting->save();

    return $setting;
}
```

---

### 2.38 既存Modelを複製しない

以下のような現在設定Modelの複製をそのまま保存する方式は避ける。

```php
$newSetting =
    $currentSetting->replicate();
```

新設定の意味を明確にするため、必要項目を明示的に設定する。

---

### 2.39 Mass Assignmentを避ける

以下のような処理は行わない。

```php
AssetAccountAvailableSetting::create(
    $request->all(),
);
```

クライアントから指定可能な値とサーバー管理値を明確に分離する。

---

### 2.40 end_year_monthはサーバー管理とする

新設定では、

```text
end_year_month = null
```

とする。

旧設定の終了年月は、

```text
新設定開始年月の前月
```

としてサーバー側で計算する。

Request Body中の`endYearMonth`は使用しない。

---

### 2.41 asset_account_idはサーバー管理とする

新設定の

```text
asset_account_id
```

は、利用者境界確認済みの資産口座IDを使用する。

Request Bodyから受け取らない。

---

### 2.42 トランザクション内の更新順

概念的には、

```text
1. 現在設定lockForUpdate
2. 業務条件再確認
3. 旧設定close
4. 新設定create
```

の順とする。

旧設定終了前に新設定を作成して、一時的に継続中設定を複数作る構成は避ける。

---

### 2.43 UPDATE後にINSERT失敗した場合

新設定INSERTで例外が発生した場合は、

```text
旧設定.end_year_month更新
```

もトランザクションによってロールバックする。

UseCase外で別々にコミットしない。

---

### 2.44 UNIQUE制約

同一資産口座内では、

```text
asset_account_id
+
start_year_month
```

をUNIQUE制約とする。

アプリケーション側の業務チェックだけに依存しない。

---

### 2.45 継続中設定の部分UNIQUE制約

PostgreSQLでは、必要に応じて

```text
asset_account_id
WHERE end_year_month IS NULL
```

の部分UNIQUEインデックスを使用する。

概念例：

```sql
CREATE UNIQUE INDEX
    uq_asset_account_available_settings_current
ON
    asset_account_available_settings (
        asset_account_id
    )
WHERE
    end_year_month IS NULL;
```

ACC-006の`lockForUpdate()`と合わせて、DB側でも整合性を保証する。

---

### 2.46 UNIQUE制約違反の変換

同一開始年月のUNIQUE制約違反が発生した場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

へ変換する。

概念的には、

```text
PostgreSQL unique violation
    ↓
AssetAccountAvailableSettingAlreadyExistsException
    ↓
409 Conflict
```

とする。

SQLSTATEや制約名はクライアントへ公開しない。

---

### 2.47 部分UNIQUE制約違反

継続中設定の部分UNIQUE制約違反が発生した場合は、同時更新競合または履歴不整合として扱う。

API共通方針に応じて、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

または競合系エラーへ変換する。

Phase1では、内部DB例外をそのまま返却しないことを優先する。

---

### 2.48 lockForUpdate

現在設定取得には、

```php
lockForUpdate()
```

を使用する。

ただし、必ず

```php
DB::transaction()
```

内で使用する。

トランザクション外で形式的に呼び出すだけの実装にはしない。

---

### 2.49 ロック対象

原則として、現在継続中の利用可能資産設定1件だけをロックする。

履歴全件を`lockForUpdate()`しない。

また、利用者の全資産口座をロックしない。

---

### 2.50 ロック後の処理を短く保つ

ロック取得後は、

```text
最新状態確認
前月算出
旧設定UPDATE
新設定INSERT
```

を速やかに実行する。

以下をロック保持中に行わない。

- 外部API
- メール送信
- ファイル処理
- 大量データ取得
- レスポンス整形

---

### 2.51 Result DTO

新規登録した利用可能資産設定を、専用Result DTOで表現する。

概念例：

```php
final readonly class
    CreateAssetAccountAvailableSettingResult
{
    public function __construct(
        public int $id,
        public string $startYearMonth,
        public ?string $endYearMonth,
        public bool $isAvailable,
    ) {
    }
}
```

正常登録直後の`endYearMonth`は`null`となる。

---

### 2.52 DTOへEloquent Modelを保持しない

以下のようなResult DTOは基本としない。

```php
final readonly class
    CreateAssetAccountAvailableSettingResult
{
    public function __construct(
        public AssetAccountAvailableSetting $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

---

### 2.53 UseCaseの戻り値

概念例：

```php
return new
    CreateAssetAccountAvailableSettingResult(
        id:
            $newSetting->id,

        startYearMonth:
            $newSetting->start_year_month,

        endYearMonth:
            $newSetting->end_year_month,

        isAvailable:
            (bool) $newSetting->is_available,
    );
```

Eloquent ModelをActionへ直接返却しない。

---

### 2.54 API Resource

Result DTOを、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php
final class
    CreateAssetAccountAvailableSettingResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'startYearMonth'
                => $this->startYearMonth,

            'endYearMonth'
                => $this->endYearMonth,

            'isAvailable'
                => $this->isAvailable,
        ];
    }
}
```

API共通方針に従ってcamelCaseで返却する。

---

### 2.55 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

- `asset_account_id`
- `user_id`
- `created_at`
- `updated_at`
- 旧設定ID
- 旧設定`start_year_month`
- 旧設定`end_year_month`
- 旧設定`is_available`
- 資産口座名
- 資産種別
- 残高記録単位
- `deleted_at`

旧履歴を確認する場合は、ACC-005を使用する。

---

### 2.56 Responder

Responderは、Result DTOを`201 Created`へ変換する。

概念例：

```php
final class
    CreateAssetAccountAvailableSettingResponder
{
    public function created(
        CreateAssetAccountAvailableSettingResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    =>
                    new CreateAssetAccountAvailableSettingResource(
                        $result,
                    ),
            ],
            Response::HTTP_CREATED,
        );
    }
}
```

`requestId`等の共通Envelope項目は、API共通レスポンス処理に従う。

---

### 2.57 Locationヘッダー

`201 Created`では、新規リソースを示す`Location`ヘッダーを設定することもできる。

ただし、Phase1では利用可能資産設定の個別取得APIを用意していないため、ACC-006専用の`Location`ヘッダーは必須としない。

---

### 2.58 Responderで行わないこと

Responderでは、以下を行わない。

- 資産口座検索
- 履歴取得
- 履歴整合性判定
- 現在設定取得
- 行ロック
- 前月計算
- UPDATE
- INSERT
- トランザクション制御
- 業務例外判定

HTTPレスポンス生成だけに責務を限定する。

---

### 2.59 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `assetAccountId`形式不正 | `INVALID_ASSET_ACCOUNT_ID` |
| Request Body不正 | `VALIDATION_ERROR` |
| 資産口座不存在 | `ASSET_ACCOUNT_NOT_FOUND` |
| 他利用者の資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 論理削除済み資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 履歴0件 | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND` |
| 履歴不整合 | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID` |
| 開始年月不正 | `ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID` |
| 現在設定と同一状態 | `ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE` |
| 同一開始年月重複 | `ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

---

### 2.60 START_YEAR_MONTH_INVALID

開始年月の業務妥当性に違反した場合は、専用例外を送出する。

概念例：

```php
if (
    ! $newStartYearMonth
        ->isAfter(
            $currentStartYearMonth,
        )
) {
    throw new
        AssetAccountAvailableSettingStartYearMonthInvalidException();
}
```

最終的に、

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

へ変換する。

---

### 2.61 NO_CHANGE

現在設定と同じ`isAvailable`の場合は、

```php
throw new
    AssetAccountAvailableSettingNoChangeException();
```

とする。

最終的に、

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

へ変換する。

---

### 2.62 HISTORY_NOT_FOUND

履歴が0件の場合は、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

へ変換する。

ACC-002の初期設定登録ルールに反する状態として扱う。

---

### 2.63 HISTORY_INVALID

既存履歴整合性に違反した場合は、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ変換する。

内部的には、ACC-005と同じ`invalidReason`を保持してよい。

---

### 2.64 DB例外変換

PostgreSQLのDB例外は、制約名やSQLSTATEを判定材料として内部で利用してよい。

ただし、レスポンスへそのまま公開しない。

例えば、

```text
unique violation
    ↓
業務例外へ変換
```

とする。

---

### 2.65 想定外例外

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

---

### 2.66 ログ

ACC-006では、必要に応じて以下をログコンテキストへ設定する。

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
ACC-006
```

とする。

---

### 2.67 正常時ログ

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = ACC-006
assetAccountId
newSettingId
startYearMonth
httpStatus = 201
```

必要であれば、

```text
previousIsAvailable
newIsAvailable
```

も内部構造化ログへ記録してよい。

---

### 2.68 履歴不整合ログ

以下の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

調査に必要な情報を記録する。

概念例：

```text
assetAccountId
historyCount
invalidReason
```

---

### 2.69 競合ログ

以下の場合は、必要に応じて競合として記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

同時実行が多発する場合に監視できる状態としてよい。

---

### 2.70 トランザクションロールバックログ

旧設定UPDATE後に新設定INSERTが失敗した場合などは、必要に応じて

```text
transactionRolledBack = true
```

相当の内部ログを記録してよい。

---

### 2.71 キャッシュ

Phase1では、ACC-006専用のサーバー側キャッシュを使用しない。

React側では、ACC-006成功後に少なくとも以下をinvalidateする。

```text
ACC-005
利用可能資産設定履歴

ACC-003
資産口座詳細
```

ACC-001が`isAvailable`を返却する場合は、一覧Queryもinvalidate対象とする。

---

### 2.72 テスト実装方針

Laravel側では、Feature Testを中心にAPI契約とトランザクション動作を確認する。

また、以下についてはUnit TestまたはDatabase Testも行う。

- FormRequest
- AssetAccountQuery
- AvailableSettingQuery
- HistoryValidator
- Repository
- UseCase
- API Resource

---

### 2.73 FormRequestのTest

以下を確認する。

```text
startYearMonth
    required
    string
    YYYY-MM
    実在年月

isAvailable
    required
    strict boolean
```

主な異常系：

```text
startYearMonth未指定
startYearMonth = null
2026-8
2026/08
2026-13
isAvailable未指定
isAvailable = null
"true"
1
0
```

---

### 2.74 AssetAccountQueryのDatabase Test

以下を確認する。

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

### 2.75 AvailableSettingQueryの履歴取得Test

対象資産口座について、履歴全件が取得できることを確認する。

```text
asset_account_id
    = assetAccountId
```

他資産口座の履歴が混在しないこと。

---

### 2.76 findCurrentForUpdateのTest

現在設定取得について、

```text
asset_account_id
    = assetAccountId

AND

end_year_month IS NULL
```

となる1件を取得できることを確認する。

Integration Testでは、可能であれば`FOR UPDATE`が発行されることも確認する。

---

### 2.77 HistoryValidatorの再利用Test

ACC-005とACC-006で同じ履歴整合性Validatorを利用する。

以下の同じ履歴に対して、両UseCaseで判定結果が一致することを確認する。

```text
初期開始年月不一致
期間重複
期間欠落
継続中設定0件
継続中設定複数件
最新設定終了済み
```

---

### 2.78 開始年月のUnit Test

現在設定が

```text
2026-08 ～ NULL
```

の場合を考える。

正常：

```text
2026-09
2026-12
2027-01
```

異常：

```text
2026-08
2026-07
2025-12
```

異常時に`StartYearMonthInvalidException`となることを確認する。

---

### 2.79 同一状態のUnit Test

現在設定：

```text
isAvailable = true
```

Request：

```text
isAvailable = true
```

の場合、

```text
AssetAccountAvailableSettingNoChangeException
```

となること。

Repositoryが呼ばれないことも確認する。

---

### 2.80 前月計算のUnit Test

以下を確認する。

```text
2026-08
    ↓
2026-07
```

```text
2027-01
    ↓
2026-12
```

```text
2026-01
    ↓
2025-12
```

年跨ぎを正しく扱えること。

---

### 2.81 RepositoryのDatabase Test

`close()`について、以下を確認する。

```text
end_year_month
    → 指定年月へ更新
```

以下は変更されないこと。

```text
asset_account_id
start_year_month
is_available
```

---

### 2.82 create()のDatabase Test

新設定が、以下で作成されることを確認する。

```text
asset_account_id
    = 指定ID

start_year_month
    = 指定年月

end_year_month
    = NULL

is_available
    = 指定boolean
```

---

### 2.83 UseCase正常系Test

Query、Validator、Repositoryを使用して、以下の順で処理されることを確認する。

```text
資産口座取得
    ↓
履歴取得
    ↓
履歴検証
    ↓
現在設定ロック
    ↓
最新状態検証
    ↓
旧設定close
    ↓
新設定create
    ↓
Result DTO
```

---

### 2.84 UseCaseのNO_CHANGE Test

現在設定とRequestの`isAvailable`が同一の場合は、

```text
NO_CHANGE
```

となり、

```text
Repository::close()
Repository::create()
```

が呼ばれないことを確認する。

---

### 2.85 UseCaseの開始年月不正Test

`startYearMonth`が現在設定開始年月以下の場合は、

```text
START_YEAR_MONTH_INVALID
```

となり、DB更新されないことを確認する。

---

### 2.86 トランザクションロールバックTest

旧設定更新後、新設定作成時に例外を発生させる。

以下を確認する。

```text
旧設定.end_year_month
    → 元のNULLへ戻る

新設定
    → 存在しない
```

---

### 2.87 UNIQUE制約Test

同一

```text
asset_account_id
+
start_year_month
```

で複数設定を登録できないことを確認する。

制約違反は、APIでは

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

へ変換されること。

---

### 2.88 継続中設定部分UNIQUE Test

部分UNIQUE制約を採用する場合は、同一資産口座に

```text
end_year_month IS NULL
```

の設定を2件作れないことを確認する。

---

### 2.89 同時実行Integration Test

可能であれば、同一資産口座に対するACC-006を並行実行する。

以下を確認する。

- `lockForUpdate()`によって直列化される
- 同じ現在設定を二重終了しない
- 継続中設定が複数にならない
- 同一開始年月が重複しない
- 後続Requestが最新状態を基準に再評価される

---

### 2.90 同時同一Request Test

以下を同時送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

以下を確認する。

- 1つの履歴だけが登録される
- 2件目は業務エラーとなる
- DB例外がそのまま返らない
- 履歴期間が壊れない

---

### 2.91 API ResourceのTest

正常時に、以下の項目だけを返却することを確認する。

```json
{
  "id": "15",
  "startYearMonth": "2026-08",
  "endYearMonth": null,
  "isAvailable": false
}
```

以下を返さないこと。

- `assetAccountId`
- `asset_account_id`
- `userId`
- `createdAt`
- `updatedAt`
- 旧設定情報
- DB内部情報

---

### 2.92 Feature Test

Feature Testでは、少なくとも以下を確認する。

```text
201 Created
400 Bad Request
404 Not Found
409 Conflict
500 Internal Server Error
```

主なケースは、

- 正常登録
- 利用者コンテキスト不正
- `assetAccountId`不正
- Request Body不正
- 他利用者資産口座
- 論理削除済み資産口座
- 履歴0件
- 履歴不整合
- 開始年月不正
- `NO_CHANGE`
- 同一開始年月重複
- トランザクションロールバック
- レスポンス契約

とする。

---

### 2.93 副作用範囲Test

ACC-006正常終了時に変更される業務データが、

```text
asset_account_available_settings

旧設定1件
    UPDATE

新設定1件
    INSERT
```

だけであることを確認する。

以下が変更されないことも確認する。

```text
asset_accounts
holding_assets
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
net_incomes
objectives
assessment_histories
```

---

## 3. 関連ドキュメント

- [ACC-006 API詳細設計](../../../api/details/asset-accounts/acc-006-create-available-setting.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [資産口座 Laravelアーキテクチャ設計](./README.md)
- [ACC-006 テスト設計](../../../tests/asset-accounts/acc-006-create-available-setting.md)
- [資産口座 テスト設計](../../../tests/asset-accounts/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
