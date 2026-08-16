# ACC-005 利用可能資産設定履歴取得

## 1. 概要

本ドキュメントでは、
ACC-005 利用可能資産設定履歴取得APIをLaravelで実装する際の、アーキテクチャおよび責務分離方針を定義する。

ACC-005では、操作対象利用者に帰属する指定された資産口座について、利用可能資産設定の履歴一覧を取得する。

利用可能資産区分は資産口座の固定属性として保持せず、`asset_account_available_settings`によって年月単位の履歴として管理する。本APIでは、指定された資産口座に紐づく過去から現在までの設定履歴を取得し、適用開始年月、適用終了年月および利用可能資産区分を返却する。

ACC-003 資産口座詳細取得が現在有効な利用可能資産設定のみを取得するのに対し、ACC-005では設定履歴全体を取得する。また、利用可能資産区分の変更はACC-006 利用可能資産設定登録で扱い、本APIから設定履歴を変更しない。

Laravel実装では、HTTPリクエストの受付から利用者コンテキストの取得、対象資産口座の確認、利用可能資産設定履歴の取得、履歴整合性の検証、並び替え、APIレスポンスの生成までを単一のクラスへ集約せず、各責務を分離する。

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
    ├─ AssetAccountAvailableSettingQuery
    └─ AvailableSettingHistoryValidator
    ↓
Result DTO
    ↓
Responder
    ↓
API Resource Collection
    ↓
HTTP Response
```

Actionは、パスパラメータおよび利用者コンテキストを受け取り、UseCaseの呼び出しを担当する。

UseCaseは、利用可能資産設定履歴取得に必要な処理を統括し、Queryを利用して操作対象利用者に帰属する有効な資産口座、およびその資産口座に紐づく利用可能資産設定履歴を取得する。

Queryは、利用者境界を含めた資産口座の検索、および指定された資産口座に紐づく設定履歴全体の取得を担当する。

AvailableSettingHistoryValidatorは、初期設定の開始年月、設定期間の連続性、期間の重複・欠落、継続中設定の件数などを検証し、履歴が一貫した状態であることを確認する。

履歴が存在しない場合や履歴に不整合が存在する場合は、正常な一覧として補完せず、データ不整合として扱う。また、本APIは参照専用であるため、不整合な履歴を自動的に修復しない。

ResponderおよびAPI Resource Collectionは、UseCaseから受け取った履歴をAPI共通方針に従ったレスポンス形式へ変換し、適用開始年月の降順で返却する。

これにより、HTTP層、ユースケース、データ取得、履歴整合性検証、レスポンス生成の責務を明確に分離するとともに、GET処理の副作用を排除し、履歴データの信頼性、テスト容易性および保守性を確保する。

---

## 2. Laravel実装方針

ACC-005では、Action、UseCase、Query、Validator、DTO、API Resource、Responderを分離して実装する。

概念的な構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ AssetAccountQuery
    ├─ AssetAccountAvailableSettingQuery
    └─ AvailableSettingHistoryValidator
    ↓
History Result DTO
    ↓
API Resource Collection
    ↓
Responder
```

ACC-005は参照専用APIであるため、Repositoryは使用しない。

また、リクエストボディを使用しないため、ACC-005専用のFormRequestは作成しない。

---

### 2.1 Route

ACC-005は、以下のルートとして定義する。

概念例：

```php
Route::get(
    '/api/v1/asset-accounts/{assetAccountId}/available-settings',
    ListAssetAccountAvailableSettingsAction::class,
);
```

同じ資産口座配下であっても、ACC-003とはURLによって責務を分離する。

```text
GET
/api/v1/asset-accounts/{assetAccountId}
    → ACC-003 資産口座詳細取得

GET
/api/v1/asset-accounts/{assetAccountId}/available-settings
    → ACC-005 利用可能資産設定履歴取得
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
Action
```

とする。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 2.3 FormRequest

ACC-005専用のFormRequestは作成しない。

本APIでは、

```text
リクエストボディ
    なし

クエリパラメータ
    なし
```

であり、ACC-005固有のRequest Bodyバリデーションが存在しないためである。

以下のような空のFormRequestは作成しない。

```php
final class ListAssetAccountAvailableSettingsRequest
    extends FormRequest
{
}
```

不要なクラスを追加しない。

---

### 2.4 assetAccountIdの形式検証

`assetAccountId`は、ルート制約またはAPI共通のパスパラメータ検証方式で検証する。

概念例：

```php
Route::get(
    '/api/v1/asset-accounts/{assetAccountId}/available-settings',
    ListAssetAccountAvailableSettingsAction::class,
)
    ->where(
        'assetAccountId',
        '[1-9][0-9]*',
    );
```

正の整数形式のみを許可する。

以下を有効なIDとして扱わない。

```text
0
-1
abc
1.5
1e3
10abc
```

形式不正は、

```text
INVALID_ASSET_ACCOUNT_ID
```

へ変換する。

---

### 2.5 Action

Actionは、パスパラメータと利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php
final class ListAssetAccountAvailableSettingsAction
{
    public function __invoke(
        string $assetAccountId,
        ListAssetAccountAvailableSettingsUseCase $useCase,
        ListAssetAccountAvailableSettingsResponder $responder,
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

- 資産口座検索
- 利用者境界判定
- 論理削除判定
- 利用可能資産設定履歴検索
- 履歴0件判定
- 履歴整合性判定
- 並び順制御
- DBアクセス
- レスポンス配列生成

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 2.7 UseCase

ACC-005の参照ユースケース全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `assetAccountId`を受け取る
3. 操作対象利用者に属する有効な資産口座を取得する
4. 対象不存在の場合は業務例外を送出する
5. 利用可能資産設定履歴を取得する
6. 履歴0件を確認する
7. 履歴整合性を検証する
8. API返却順へ整列する
9. Result DTOへ変換する
10. DTO一覧を返却する

概念的には、

```text
userId
+
assetAccountId
    ↓
AssetAccountQuery
    ↓
資産口座取得
    ↓
AssetAccountAvailableSettingQuery
    ↓
履歴取得
    ↓
0件確認
    ↓
AvailableSettingHistoryValidator
    ↓
履歴整合性確認
    ↓
Result DTO一覧
```

とする。

---

### 2.8 AssetAccountQuery

対象資産口座の取得は、専用Queryへ委譲する。

概念例：

```php
$assetAccount =
    $this->assetAccountQuery
        ->findActiveByIdAndUser(
            assetAccountId:
                $assetAccountId,

            userId:
                $userId,
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

### 2.9 利用者境界をQueryへ含める

以下のように、IDだけで資産口座を取得してからUseCase側で所有利用者を判定する方式を基本としない。

```php
$assetAccount =
    AssetAccount::find(
        $assetAccountId,
    );
```

対象取得時点から、

```text
assetAccountId
+
userId
+
利用中状態
```

を条件へ含める。

これにより、他利用者の資産口座を誤って取得することを防止する。

---

### 2.10 SoftDeletes

`AssetAccount` Modelでは、Laravelの

```php
use SoftDeletes;
```

を使用する。

ACC-005の資産口座取得では、

```php
withTrashed()
```

を使用しない。

論理削除済み資産口座は、履歴取得対象外とする。

---

### 2.11 ASSET_ACCOUNT_NOT_FOUND

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

### 2.12 AssetAccountAvailableSettingQuery

利用可能資産設定履歴の取得は、専用Queryへ委譲する。

概念例：

```php
$settings =
    $this->availableSettingQuery
        ->findHistoryByAssetAccount(
            assetAccountId:
                $assetAccount->id,
        );
```

概念的な条件は、以下とする。

```text
asset_account_id
    = assetAccountId
```

ACC-005では、現在年月条件を付与しない。

対象資産口座の履歴全体を取得する。

---

### 2.13 履歴の並び順

DB取得時は、履歴整合性検証を行いやすいように

```text
start_year_month ASC
```

で取得してよい。

概念例：

```php
return AssetAccountAvailableSetting::query()
    ->where(
        'asset_account_id',
        $assetAccountId,
    )
    ->orderBy(
        'start_year_month',
    )
    ->get();
```

履歴整合性確認では、古い設定から新しい設定へ順番に比較する方が実装しやすいためである。

---

### 2.14 API返却順と検証順を分けてよい

ACC-005のAPI返却順は、

```text
startYearMonth DESC
```

とする。

一方、履歴整合性検証では、

```text
startYearMonth ASC
```

で処理してよい。

概念的には、

```text
DB取得
    ASC
    ↓
履歴整合性検証
    ↓
Result生成
    ↓
DESCへ変換
    ↓
API返却
```

とする。

または、DBからDESCで取得し、Validator内部でASCへ並び替えてもよい。

実装上、責務が分かりやすい方式を採用する。

---

### 2.15 履歴0件

取得した利用可能資産設定履歴が0件の場合は、正常な空配列として扱わない。

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

### 2.16 AvailableSettingHistoryValidator

利用可能資産設定履歴の整合性確認は、専用Validatorへ分離する。

概念的な構成は、以下とする。

```text
AvailableSettingHistoryValidator
    ├─ 初期開始年月確認
    ├─ 各期間の大小確認
    ├─ 期間連続性確認
    ├─ 期間重複確認
    ├─ 継続中設定件数確認
    └─ 最新設定確認
```

UseCaseへ複雑な期間検証ロジックを直接書き並べない。

---

### 2.17 Validatorへ渡す値

Validatorには、少なくとも以下を渡す。

```text
assetAccount.startYearMonth
+
利用可能資産設定履歴
```

概念例：

```php
$this->historyValidator
    ->validate(
        assetAccountStartYearMonth:
            $assetAccount->start_year_month,

        settings:
            $settings,
    );
```

---

### 2.18 初期設定開始年月の検証

最古の設定の

```text
start_year_month
```

が、

```text
asset_accounts.start_year_month
```

と一致することを確認する。

概念例：

```php
$first =
    $settings->first();

if (
    $first->start_year_month
    !== $assetAccountStartYearMonth
) {
    throw new
        AssetAccountAvailableSettingHistoryInvalidException(
            reason:
                AvailableSettingHistoryInvalidReason::INITIAL_START_MISMATCH,
        );
}
```

---

### 2.19 各設定期間の大小関係

`end_year_month`が`NULL`ではない設定について、

```text
start_year_month
    <= end_year_month
```

であることを確認する。

例えば、

```text
2026-08 ～ 2026-07
```

のような設定は不正とする。

---

### 2.20 年月比較

年月比較は、文字列の大小比較へ無造作に依存せず、年月として意味のある型へ変換して扱ってよい。

例えば、

```text
YearMonth
```

のValue Objectを導入してもよい。

概念例：

```php
final readonly class YearMonth
{
    public function __construct(
        public int $year,
        public int $month,
    ) {
    }
}
```

ただし、Phase1で過剰になる場合は、`YYYY-MM`形式が保証されていることを前提にCarbon等へ変換して比較してよい。

---

### 2.21 期間連続性の検証

古い設定から順に前後レコードを比較する。

概念的には、

```text
前設定.endYearMonth
    の翌月
        =
次設定.startYearMonth
```

であることを確認する。

例えば、

```text
前設定
2026-01 ～ 2026-06

次設定
2026-07 ～ 2026-12
```

は正常とする。

---

### 2.22 翌月計算

年月単位で翌月を求める処理は、文字列連結などで独自実装しない。

例えば、

```php
$expectedNextStart =
    CarbonImmutable::createFromFormat(
        'Y-m',
        $previous->end_year_month,
    )
        ->addMonth()
        ->format('Y-m');
```

のように、日付ライブラリまたはYearMonth Value Objectを使用する。

12月から翌年1月への年跨ぎも正しく扱う。

---

### 2.23 期間重複の検出

例えば、

```text
前設定
2026-01 ～ 2026-08

次設定
2026-08 ～ NULL
```

の場合は、

```text
前設定.endYearMonth
    >=
次設定.startYearMonth
```

となるため、期間重複として扱う。

ただし、期間連続性チェックを

```text
前設定終了年月の翌月
    =
次設定開始年月
```

として実装すれば、重複と欠落の双方を判定できる。

---

### 2.24 期間欠落の検出

例えば、

```text
前設定
2026-01 ～ 2026-05

次設定
2026-07 ～ NULL
```

では、

```text
前設定終了年月の翌月
    = 2026-06

次設定開始年月
    = 2026-07
```

となるため、期間欠落として扱う。

---

### 2.25 end_year_month = NULLの位置

`end_year_month = NULL`は、最新設定だけに許可する。

古い設定に`NULL`が存在した状態で後続設定が存在する場合は、履歴不整合とする。

例えば、

```text
設定A
2026-01 ～ NULL

設定B
2026-08 ～ NULL
```

は不正とする。

---

### 2.26 継続中設定件数

利用中資産口座では、

```text
end_year_month = NULL
```

の履歴が1件だけ存在することを正常状態とする。

概念例：

```php
$openCount =
    $settings
        ->filter(
            fn ($setting) =>
                $setting->end_year_month
                    === null,
        )
        ->count();

if ($openCount !== 1) {
    throw new
        AssetAccountAvailableSettingHistoryInvalidException(
            reason:
                AvailableSettingHistoryInvalidReason::INVALID_OPEN_PERIOD_COUNT,
        );
}
```

---

### 2.27 最新設定の確認

開始年月が最も新しい設定の

```text
end_year_month
```

が`NULL`であることを確認する。

最新設定に終了年月が存在する場合は、利用中資産口座として現在状態を一意に判定できないため不整合とする。

---

### 2.28 同一開始年月

同一資産口座について、

```text
asset_account_id
+
start_year_month
```

は一意であることを前提とする。

通常はDBのUNIQUE制約によって防止する。

Validatorでも検出可能な場合は、不整合として扱ってよい。

---

### 2.29 不整合理由の内部表現

履歴不整合は、API外部向けには

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ統一する。

一方、内部的には原因を区別してよい。

概念例：

```php
enum AvailableSettingHistoryInvalidReason: string
{
    case INITIAL_START_MISMATCH =
        'INITIAL_START_MISMATCH';

    case INVALID_PERIOD =
        'INVALID_PERIOD';

    case PERIOD_OVERLAP =
        'PERIOD_OVERLAP';

    case PERIOD_GAP =
        'PERIOD_GAP';

    case INVALID_OPEN_PERIOD_COUNT =
        'INVALID_OPEN_PERIOD_COUNT';

    case LATEST_PERIOD_CLOSED =
        'LATEST_PERIOD_CLOSED';

    case DUPLICATE_START_YEAR_MONTH =
        'DUPLICATE_START_YEAR_MONTH';
}
```

これは、ログ・テスト・障害調査用として使用する。

---

### 2.30 内部理由をレスポンスへ公開しない

例えば、

```text
PERIOD_GAP
MULTIPLE_OPEN_PERIODS
```

などの内部診断情報をそのままAPIの`error.code`へ使用しない。

外部向けには、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ集約する。

これにより、API契約を過度に細分化しない。

---

### 2.31 履歴を自動修復しない

Validatorは履歴を検証するだけとする。

以下を行わない。

```text
期間欠落
    → 前設定を自動延長

複数NULL
    → 古い設定を自動終了

最新設定終了済み
    → end_year_monthをNULLへ変更
```

ACC-005はGET APIであり、参照処理に副作用を持たせない。

---

### 2.32 Repositoryを使用しない

ACC-005では、

```text
INSERT
UPDATE
DELETE
```

を行わない。

そのため、ACC-005専用Repositoryは使用しない。

概念的には、

```text
Query
    → 読み取り

Repository
    → 書き込み
```

という共通責務分離方針に従う。

---

### 2.33 トランザクション

ACC-005では、明示的な

```php
DB::transaction()
```

を使用しない。

参照専用であり、履歴表示用途として厳密な同一時点スナップショットを要件としないためである。

---

### 2.34 lockForUpdateを使用しない

ACC-005では、

```php
lockForUpdate()
```

を使用しない。

履歴取得中にACC-006の更新を不必要にブロックしない。

---

### 2.35 Result DTO

利用可能資産設定履歴1件を、専用DTOとして表現する。

概念例：

```php
final readonly class
    AssetAccountAvailableSettingHistoryResult
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

---

### 2.36 Collection Result

UseCaseの戻り値として、DTO配列または専用Collection DTOを使用してよい。

概念例：

```php
final readonly class
    AssetAccountAvailableSettingHistoryListResult
{
    /**
     * @param array<
     *   AssetAccountAvailableSettingHistoryResult
     * > $items
     */
    public function __construct(
        public array $items,
    ) {
    }
}
```

Phase1では、単純なDTO配列でもよい。

過剰にWrapper DTOを増やさない。

---

### 2.37 DTOへEloquent Modelを保持しない

以下のようなDTOは基本としない。

```php
final readonly class
    AssetAccountAvailableSettingHistoryResult
{
    public function __construct(
        public AssetAccountAvailableSetting
            $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

---

### 2.38 DTO生成

履歴整合性確認後に、各Eloquent ModelからDTOを生成する。

概念例：

```php
$results =
    $settings
        ->sortByDesc(
            'start_year_month',
        )
        ->map(
            fn (
                AssetAccountAvailableSetting $setting,
            ) =>
                new AssetAccountAvailableSettingHistoryResult(
                    id:
                        $setting->id,

                    startYearMonth:
                        $setting->start_year_month,

                    endYearMonth:
                        $setting->end_year_month,

                    isAvailable:
                        (bool) $setting->is_available,
                ),
        )
        ->values()
        ->all();
```

---

### 2.39 API Resource

利用可能資産設定履歴1件を、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php
final class
    AssetAccountAvailableSettingHistoryResource
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

---

### 2.40 Resource Collection

一覧レスポンスでは、Resource Collectionを使用する。

概念例：

```php
return
    AssetAccountAvailableSettingHistoryResource
        ::collection(
            $results,
        );
```

または、Responder内でResource Collectionを組み立てる。

---

### 2.41 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

- `asset_account_id`
- `created_at`
- `updated_at`
- `user_id`
- 資産口座名
- 資産種別
- 残高記録単位
- 資産口座の利用開始年月
- `deleted_at`

また、派生値として

```text
isCurrent
```

もPhase1では返却しない。

---

### 2.42 snake_caseをそのまま返さない

DBカラム名をそのままAPIへ公開しない。

例えば、

```text
start_year_month
end_year_month
is_available
```

は、

```text
startYearMonth
endYearMonth
isAvailable
```

へ変換する。

---

### 2.43 Responder

Responderは、履歴DTO一覧を`200 OK`レスポンスへ変換する。

概念例：

```php
final class
    ListAssetAccountAvailableSettingsResponder
{
    /**
     * @param array<
     *   AssetAccountAvailableSettingHistoryResult
     * > $results
     */
    public function ok(
        array $results,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    =>
                    AssetAccountAvailableSettingHistoryResource
                        ::collection(
                            $results,
                        ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの共通Envelope項目は、API共通レスポンス処理に従う。

---

### 2.44 Responderで行わないこと

Responderでは、以下を行わない。

- 資産口座検索
- 利用者境界確認
- 履歴取得
- 並び順判定
- 履歴0件判定
- 期間重複判定
- 期間欠落判定
- 継続中設定判定
- DBアクセス

Responderは、生成済みDTOをHTTPレスポンスへ変換することに責務を限定する。

---

### 2.45 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `assetAccountId`形式不正 | `INVALID_ASSET_ACCOUNT_ID` |
| 資産口座不存在 | `ASSET_ACCOUNT_NOT_FOUND` |
| 他利用者の資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 論理削除済み資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 利用可能資産設定履歴0件 | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND` |
| 履歴整合性不正 | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

---

### 2.46 HistoryNotFoundException

履歴が0件の場合は、専用例外を送出する。

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

### 2.47 HistoryInvalidException

履歴整合性に違反した場合は、専用例外を送出する。

概念例：

```php
throw new
    AssetAccountAvailableSettingHistoryInvalidException(
        reason:
            AvailableSettingHistoryInvalidReason::PERIOD_GAP,
    );
```

外部向けには、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ変換する。

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

ACC-005では、必要に応じて以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assetAccountId
historyCount
httpStatus
errorCode
```

`apiId`は、

```text
ACC-005
```

とする。

---

### 2.50 正常時ログ

正常時には、必要に応じて

```text
requestId
userId
apiId = ACC-005
assetAccountId
historyCount
httpStatus = 200
```

を記録する。

履歴の具体的な

```text
startYearMonth
endYearMonth
isAvailable
```

は、通常ログへ不要に出力しない。

---

### 2.51 履歴不整合ログ

以下の場合は、調査に必要な情報を内部ログへ記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

必要に応じて、

```text
requestId
userId
assetAccountId
historyCount
invalidReason
```

を記録する。

---

### 2.52 invalidReason

`ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID`の場合は、内部的に

```text
INITIAL_START_MISMATCH
INVALID_PERIOD
PERIOD_OVERLAP
PERIOD_GAP
INVALID_OPEN_PERIOD_COUNT
LATEST_PERIOD_CLOSED
DUPLICATE_START_YEAR_MONTH
```

などを記録してよい。

クライアントには公開しない。

---

### 2.53 キャッシュ

Phase1では、ACC-005専用のサーバー側キャッシュを使用しない。

毎回、

```text
asset_accounts
+
asset_account_available_settings
```

の最新DB状態を参照する。

React側では、TanStack Query等によるQuery Cacheを使用してよい。

---

### 2.54 テスト実装方針

Laravel側では、Feature Testを中心としてACC-005のAPI契約を確認する。

また、履歴整合性ロジックについてはValidatorのUnit Testを重点的に作成する。

主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `404 Not Found`
- `500 Internal Server Error`
- `X-User-Id`必須
- `assetAccountId`形式
- 利用者境界
- SoftDeletes
- 履歴1件
- 履歴複数件
- 並び順
- 履歴0件
- 初期開始年月
- 期間大小
- 期間重複
- 期間欠落
- 継続中設定件数
- 最新設定
- レスポンス契約
- 副作用なし

---

### 2.55 AssetAccountQueryのDatabase Test

対象資産口座取得について、以下を確認する。

```text
id一致
+
user_id一致
+
deleted_at IS NULL
    ↓
取得できる
```

以下の場合は取得できないことを確認する。

- 存在しないID
- 他利用者の資産口座
- 論理削除済み資産口座

---

### 2.56 AssetAccountAvailableSettingQueryのDatabase Test

以下の条件で対象資産口座の履歴だけを取得できることを確認する。

```text
asset_account_id
    = assetAccountId
```

以下も確認する。

- 他資産口座の設定を取得しない
- 過去設定も取得する
- 現在設定も取得する
- 現在年月で絞り込まない
- 並び順が想定どおりである

---

### 2.57 HistoryValidatorのUnit Test

履歴Validatorでは、年月境界を含めて細かくUnit Testを作成する。

正常例：

```text
2026-01 ～ 2026-06
2026-07 ～ 2026-12
2027-01 ～ NULL
```

正常として通過することを確認する。

---

### 2.58 初期開始年月のUnit Test

以下を確認する。

正常：

```text
assetAccount.startYearMonth
    = 2026-01

最古設定.startYearMonth
    = 2026-01
```

異常：

```text
assetAccount.startYearMonth
    = 2026-01

最古設定.startYearMonth
    = 2026-02
```

異常時は、

```text
INITIAL_START_MISMATCH
```

として検出できることを確認する。

---

### 2.59 期間大小のUnit Test

以下を確認する。

正常：

```text
2026-08 ～ 2026-08
```

異常：

```text
2026-08 ～ 2026-07
```

異常時は、

```text
INVALID_PERIOD
```

として検出できることを確認する。

---

### 2.60 期間連続性のUnit Test

正常例：

```text
2026-01 ～ 2026-06
2026-07 ～ NULL
```

異常例：

```text
2026-01 ～ 2026-05
2026-07 ～ NULL
```

後者は、

```text
PERIOD_GAP
```

として検出できることを確認する。

---

### 2.61 期間重複のUnit Test

以下を用意する。

```text
2026-01 ～ 2026-08
2026-08 ～ NULL
```

重複として検出できることを確認する。

---

### 2.62 年跨ぎのUnit Test

以下のような年跨ぎの連続性を確認する。

```text
設定A
2026-01 ～ 2026-12

設定B
2027-01 ～ NULL
```

正常に連続期間として扱われること。

---

### 2.63 12月から1月への翌月計算

翌月計算が、

```text
2026-12
    ↓
2027-01
```

となることを確認する。

単純な月数加算によって

```text
2026-13
```

などにならないこと。

---

### 2.64 継続中設定件数のUnit Test

以下を確認する。

正常：

```text
end_year_month = NULL
    1件
```

異常：

```text
end_year_month = NULL
    0件
```

異常：

```text
end_year_month = NULL
    2件以上
```

正常以外では履歴不整合となること。

---

### 2.65 最新設定のUnit Test

最新設定が、

```text
end_year_month = NULL
```

の場合は正常とする。

最新設定に

```text
end_year_month = 2026-12
```

などが設定されている場合は、

```text
LATEST_PERIOD_CLOSED
```

として検出できることを確認する。

---

### 2.66 UseCaseのUnit Test

QueryとValidatorをMockして、業務フローを確認する。

正常系：

```text
AssetAccountQuery
    ↓
資産口座取得
    ↓
AvailableSettingQuery
    ↓
履歴1件以上
    ↓
HistoryValidator
    ↓
正常
    ↓
Result DTO一覧
```

資産口座不存在：

```text
AssetAccountQuery
    ↓
null
    ↓
AssetAccountNotFoundException
```

履歴0件：

```text
AvailableSettingQuery
    ↓
0件
    ↓
AssetAccountAvailableSettingHistoryNotFoundException
```

履歴不整合：

```text
HistoryValidator
    ↓
invalid
    ↓
AssetAccountAvailableSettingHistoryInvalidException
```

---

### 2.67 API ResourceのTest

正常時に、各履歴が以下の形式となることを確認する。

```json
{
  "id": "3",
  "startYearMonth": "2027-01",
  "endYearMonth": null,
  "isAvailable": true
}
```

以下が含まれないことを確認する。

- `assetAccountId`
- `asset_account_id`
- `userId`
- `user_id`
- `createdAt`
- `created_at`
- `updatedAt`
- `updated_at`
- `isCurrent`

---

### 2.68 Resource CollectionのTest

複数履歴の場合に、

```text
startYearMonth DESC
```

で返却されることを確認する。

例えば、

```text
2027-01
2026-07
2026-01
```

の順となること。

---

### 2.69 副作用なしのTest

ACC-005実行前後で、以下のテーブルに変更がないことを確認する。

```text
asset_accounts
asset_account_available_settings
```

特に、

```text
created_at
updated_at
deleted_at
```

が履歴取得によって変更されないことを確認する。

---

### 2.70 GET処理で修復しないことのTest

履歴不整合データを用意してACC-005を実行する。

以下を確認する。

```text
履歴不整合
    ↓
500エラー
```

となり、

```text
asset_account_available_settings
```

に対する

```text
INSERT
UPDATE
DELETE
```

が一切行われないこと。

---

## 3. 関連ドキュメント

- [ACC-005 API詳細設計](../../../api/details/asset-accounts/acc-005-available-setting-history.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [資産口座 Laravelアーキテクチャ設計](./README.md)
- [ACC-005 テスト設計](../../../tests/asset-accounts/acc-005-available-setting-history.md)
- [資産口座 テスト設計](../../../tests/asset-accounts/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
