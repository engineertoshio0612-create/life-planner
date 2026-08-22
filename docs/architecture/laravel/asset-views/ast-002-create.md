##  AST-002 指定年月資産状況取得

## 1. 概要

本ドキュメントでは、
AST-002 指定年月資産状況取得APIをLaravelで実装する際の、アーキテクチャおよび責務分離方針を定義する。

AST-002では、操作対象利用者について、パスパラメータ`targetYearMonth`で指定された対象年月に一致する確定済み月末資産状況を取得し、その時点における資産状況を算出する。

対象となる月末資産状況は、以下の条件をすべて満たすものとする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = targetYearMonth

AND

month_end_asset_snapshots.confirmed
    = true
```

取得する資産状況には、主に以下を含む。

* 対象年月
* 総資産
* 利用可能資産
* 資産口座別資産状況
* 保有商品別資産状況

資産額の算出では、資産口座の`balance_recording_unit`に応じて使用するデータを切り替える。

```text
口座単位で残高を記録する資産口座
    ↓
month_end_asset_balances.balance

商品単位で評価額を記録する資産口座
    ↓
month_end_holding_values.value
```

月末資産残高と商品別月末評価額を単純に合算せず、残高記録単位に対応する値のみを採用することで、同一資産の二重計上を防止する。

また、利用可能資産については現在日時点の設定ではなく、指定された`targetYearMonth`時点で有効な`asset_account_available_settings`をもとに算出する。これにより、過去の対象年月を指定した場合でも、その年月時点の利用可能資産状態を再現する。

Laravel実装では、指定年月資産状況取得に関する処理を単一のクラスへ集約せず、以下の責務へ分離する。

```text
Route
    ↓
Middleware
    ↓
Request
    ↓
Action
    ↓
UseCase
    ├─ MonthEndAssetSnapshotQuery
    ├─ AssetSummaryQuery
    └─ AssetAvailabilityQuery
    ↓
AssetSummary DTO
    ↓
API Resource
    ↓
Responder
```

Requestでは`targetYearMonth`の形式検証を行い、Actionは検証済みの入力値および利用者コンテキストを受け取ってUseCaseを呼び出すことに責務を限定する。

指定年月の確定済み月末資産状況の特定、資産データの取得、資産口座別・保有商品別の資産状況構成、総資産および利用可能資産の集計といった業務フローはUseCaseで制御し、データベースへの参照処理はQueryへ委譲する。

指定された年月について確定済み月末資産状況が存在しない場合は、前月・翌月や最新の確定済み年月へフォールバックせず、`CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND`として扱う。AST-002は、指定された年月の資産状況を取得することに責務を限定する。

AST-001 現在資産状況取得との主な違いは、対象となる確定済み月末資産状況の特定方法である。

```text
AST-001
    ↓
最新の確定済みsnapshot

AST-002
    ↓
指定年月の確定済みsnapshot
```

snapshot特定後の資産データ取得、残高記録単位による算出元の切り替え、資産口座別・保有商品別集計、利用可能資産判定、総資産集計およびDTO生成については、AST-001と可能な限り共通化し、同一の資産集計ロジックを重複実装しない。

AST-002は資産状況を参照するための読み取り専用APIであり、資産データの登録・更新・削除は行わない。

そのため、Phase1ではRepositoryによる更新処理、明示的なトランザクション、行ロックおよびAST-002専用のサーバー側アプリケーションキャッシュは使用せず、参照処理に適したQuery中心の構成とする。

---

### 2 Laravel実装方針

AST-002では、
Action、
Request、
UseCase、
Query、
集計用DTO、
API Resource、
Responderを分離して実装する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
Middleware
    ↓
Request
    ↓
Action
    ↓
UseCase
    ├─ MonthEndAssetSnapshotQuery
    ├─ AssetSummaryQuery
    └─ AssetAvailabilityQuery
    ↓
AssetSummary DTO
    ↓
API Resource
    ↓
Responder
```

AST-002の業務フロー制御は、UseCaseへ集約する。

Actionへ、指定年月の月末資産状況検索、資産集計、利用可能資産判定、利用者境界確認などの業務ロジックを直接記述しない。

---

#### 2.1 Action

HTTPリクエストを受け付け、検証済みの`targetYearMonth`および利用者コンテキストを取得する。

指定年月資産状況取得UseCaseを呼び出し、取得結果をResponderへ渡す。

概念例：

```php
final class GetAssetSummaryByMonthAction
{
    public function __invoke(
        GetAssetSummaryByMonthRequest $request,
        GetAssetSummaryByMonthUseCase $useCase,
        AssetSummaryByMonthResponder $responder,
        userContext $userContext,
    ): JsonResponse {
        $summary = $useCase->execute(
            userId: $userContext->userId,
            targetYearMonth:
                $request->validated(
                    'targetYearMonth',
                ),
        );

        return $responder->ok(
            $summary,
        );
    }
}
```

Actionでは、以下を行わない。

* `targetYearMonth`の形式検証
* 指定年月の月末資産状況検索
* 月末資産残高の取得
* 商品別月末評価額の取得
* 残高記録単位の判定
* 総資産の集計
* 利用可能資産の集計
* 利用可能資産設定の判定
* 資産口座別資産状況の組み立て
* 保有商品別資産状況の組み立て
* 利用者境界の判定
* APIレスポンス形式への変換

---

#### 2.2 Request

パスパラメータの`targetYearMonth`を検証する。

本APIでは、クエリパラメータおよびリクエストボディを使用しない。

`targetYearMonth`は、`prepareForValidation()`でバリデーション対象へ追加する。

概念例：

```php
final class GetAssetSummaryByMonthRequest
    extends FormRequest
{
    protected function prepareForValidation(): void
    {
        $this->merge([
            'targetYearMonth'
                => $this->route(
                    'targetYearMonth',
                ),
        ]);
    }

    public function rules(): array
    {
        return [
            'targetYearMonth' => [
                'required',
                'regex:/^\d{4}-(0[1-9]|1[0-2])$/',
            ],
        ];
    }
}
```

年月形式の検証を複数APIで使用する場合は、独自Ruleへ切り出してよい。

概念例：

```php
'targetYearMonth' => [
    'required',
    new YearMonthRule(),
],
```

---

#### 2.3 Requestで行わないこと

Requestでは、以下の業務判定を行わない。

* 指定年月の月末資産状況が存在するか
* 指定年月の月末資産状況が確定済みか
* 指定年月の月末資産状況が操作対象利用者に属するか
* 月末資産残高が存在するか
* 商品別月末評価額が存在するか
* 利用可能資産設定が存在するか
* 資産状況を正常に算出できるか

これらは、入力値の形式検証ではなく、データベース状態に依存する業務ルールとしてUseCaseおよびQueryで扱う。

---

#### 2.4 UseCase

指定年月資産状況取得のユースケース処理を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `targetYearMonth`を受け取る
3. 指定年月の確定済み月末資産状況を取得する
 対象が存在しない場合は業務例外を送出する
5. 対象snapshotに紐づく資産データを取得する
6. 指定年月時点の利用可能資産設定を取得する
7. 資産口座ごとの資産額を算出する
8. 商品単位の資産口座について保有商品別資産状況を構成する
9. 総資産を算出する
10. 利用可能資産を算出する
11. `AssetSummary` DTOを生成して返却する

概念的な処理は、以下とする。

```text
操作対象利用者
+
targetYearMonth
    ↓
指定年月の
確定済み月末資産状況取得
    ↓
存在しない
    → CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND

存在する
    ↓
snapshotId確定
    ↓
資産データ取得
    ↓
targetYearMonth時点の
利用可能資産設定取得
    ↓
資産口座別集計
    ↓
総資産集計
    ↓
利用可能資産集計
    ↓
AssetSummary生成
```

UseCaseでは、SQLやEloquent Query Builderを直接組み立てない。

データベースアクセスは、Queryへ委譲する。

---

#### 2.5 MonthEndAssetSnapshotQuery

操作対象利用者について、指定年月と一致する確定済み月末資産状況を取得する。

検索条件には、必ず以下を含める。

```text
user_id
target_year_month
confirmed
```

概念例：

```php
final class MonthEndAssetSnapshotQuery
{
    public function findConfirmedForUserByTargetYearMonth(
        int $userId,
        string $targetYearMonth,
    ): ?MonthEndAssetSnapshot {
        return MonthEndAssetSnapshot::query()
            ->where(
                'user_id',
                $userId,
            )
            ->where(
                'target_year_month',
                $targetYearMonth,
            )
            ->where(
                'confirmed',
                true,
            )
            ->first([
                'id',
                'target_year_month',
            ]);
    }
}
```

`targetYearMonth`だけを条件として月末資産状況を取得しない。

---

#### 2.6 指定年月の確定済み月末資産状況不存在

MonthEndAssetSnapshotQueryの取得結果が`null`の場合は、

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

へ変換するための業務例外を送出する。

概念例：

```php
$snapshot =
    $this->monthEndAssetSnapshotQuery
        ->findConfirmedForUserByTargetYearMonth(
            userId: $userId,
            targetYearMonth:
                $targetYearMonth,
        );

if ($snapshot === null) {
    throw new
        ConfirmedAssetSnapshotNotFoundException();
}
```

以下の場合は、クライアントへ区別して公開しない。

* 指定年月の月末資産状況が存在しない
* 指定年月の月末資産状況が未確定
* 他利用者にのみ同年月の確定済み月末資産状況が存在する

すべて、

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

として扱う。

---

#### 2.7 別年月へフォールバックしない

AST-002では、指定した`targetYearMonth`について確定済み月末資産状況が存在しない場合でも、前月または翌月を代替利用しない。

以下のような処理は行わない。

```text
2026-06
確定済みデータなし
    ↓
2026-05を取得
```

または、

```text
2026-06
確定済みデータなし
    ↓
2026-07を取得
```

AST-002は、指定年月の資産状況取得に責務を限定する。

---

#### 2.8 AssetSummaryQuery

対象snapshotに紐づく資産状況算出用データを取得する。

概念例：

```php
$assetData =
    $this->assetSummaryQuery
        ->findBySnapshot(
            userId: $userId,
            snapshotId: $snapshot->id,
        );
```

Queryでは、主に以下のデータを取得する。

```text
asset_accounts
month_end_asset_balances
holding_assets
month_end_holding_values
```

資産口座の`balance_recording_unit`に応じて、資産額算出に必要となる情報を取得する。

---

#### 2.9 AST-001との共通化

AST-001とAST-002の主な違いは、対象となる確定済み月末資産状況の特定方法である。

```text
AST-001
    ↓
最新の確定済みsnapshot

AST-002
    ↓
指定年月の確定済みsnapshot
```

snapshotが特定された後の以下の処理については、AST-001と共通化する。

* 資産データ取得
* 残高記録単位による算出元切り替え
* 資産口座別集計
* 保有商品別集計
* 利用可能資産判定
* 総資産集計
* 利用可能資産集計
* DTO生成

同じ資産集計ロジックをAST-001とAST-002へ重複実装しない。

---

#### 2.10 利用者境界

資産情報取得時は、必ず操作対象利用者との利用者境界を保証する。

月末資産残高については、

```text
month_end_asset_balances
    ↓
asset_accounts
    ↓
asset_accounts.user_id
        = 操作対象利用者ID
```

によって保証する。

商品別月末評価額については、

```text
month_end_holding_values
    ↓
holding_assets
    ↓
asset_accounts
    ↓
asset_accounts.user_id
        = 操作対象利用者ID
```

によって保証する。

`snapshotId`だけを条件として資産情報を取得しない。

---

#### 2.11 口座単位の資産額

口座単位で残高を記録する資産口座では、

```text
month_end_asset_balances.balance
```

を資産額として使用する。

概念的な取得条件は、以下とする。

```text
month_end_asset_snapshot_id
    = snapshotId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.balance_recording_unit
    = 口座単位
```

商品別月末評価額が何らかの理由で存在していても、資産額へ加算しない。

---

#### 2.12 商品単位の資産額

商品単位で評価額を記録する資産口座では、

```text
month_end_holding_values.value
```

を資産額として使用する。

保有商品ごとの`value`を取得し、同一資産口座に属する保有商品の評価額を合計して資産口座の資産額とする。

```text
assetAccount.assetAmount
=
SUM(holdingAssets.assetAmount)
```

概念的な取得条件は、以下とする。

```text
month_end_asset_snapshot_id
    = snapshotId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.balance_recording_unit
    = 商品単位
```

---

#### 2.13 二重計上の防止

口座単位と商品単位の資産額を明確に分離して扱う。

以下のような単純合計は行わない。

```text
SUM(month_end_asset_balances.balance)
+
SUM(month_end_holding_values.value)
```

残高記録単位に応じて、

```text
口座単位
    → balanceのみ

商品単位
    → valueのみ
```

を使用する。

これにより、同一資産の二重計上を防止する。

---

#### 2.14 AssetAvailabilityQuery

指定年月時点の利用可能資産設定を取得する。

概念例：

```php
$availabilityMap =
    $this->assetAvailabilityQuery
        ->findForTargetYearMonth(
            userId: $userId,
            targetYearMonth:
                $targetYearMonth,
        );
```

現在日時点の設定ではなく、リクエストで指定された`targetYearMonth`を基準とする。

---

#### 2.15 利用可能資産設定の利用者境界

利用可能資産設定についても、操作対象利用者に属する資産口座の設定のみを取得する。

概念的には、

```text
asset_account_available_settings
    ↓
asset_accounts
    ↓
asset_accounts.user_id
        = 操作対象利用者ID
```

によって利用者境界を保証する。

他の利用者に属する利用可能資産設定を使用してはならない。

---

#### 2.16 指定年月時点の設定判定

利用可能資産設定は、`targetYearMonth`時点で有効な設定を使用する。

現在の最新設定だけを使用しない。

概念的には、

```text
targetYearMonth
    ↓
指定年月時点で有効な
asset_account_available_settings
    ↓
available判定
```

とする。

これにより、過去月を指定した場合でも、その年月時点の利用可能資産状態を再現する。

具体的な有効期間判定は、`asset_account_available_settings`のテーブル定義および業務ルールに従う。

---

#### 2.17 AssetSummary DTO

指定年月資産状況は、Eloquent ModelをそのままResourceへ渡すのではなく、表示用DTOとして構成する。

概念例：

```php
final readonly class AssetSummary
{
    public function __construct(
        public string $targetYearMonth,
        public int $totalAssets,
        public int $availableAssets,
        /** @var AssetAccountSummary[] */
        public array $assetAccounts,
    ) {
    }
}
```

資産口座単位は、以下のようなDTOとする。

```php
final readonly class AssetAccountSummary
{
    public function __construct(
        public int $assetAccountId,
        public string $name,
        public int $assetAmount,
        public bool $available,
        /** @var HoldingAssetSummary[] */
        public array $holdingAssets,
    ) {
    }
}
```

保有商品単位は、以下のようなDTOとする。

```php
final readonly class HoldingAssetSummary
{
    public function __construct(
        public int $holdingAssetId,
        public string $name,
        public int $assetAmount,
    ) {
    }
}
```

AST-001と同じレスポンス構造であるため、同じDTOを共通利用してよい。

---

#### 2.18 資産口座別資産状況の生成

UseCaseでは、Queryで取得したデータをもとに`AssetAccountSummary`を生成する。

口座単位の場合は、

```text
assetAmount
    = month_end_asset_balances.balance

holdingAssets
    = []
```

とする。

商品単位の場合は、

```text
holdingAssets
    = 商品別月末評価額一覧

assetAmount
    = SUM(holdingAssets.assetAmount)
```

とする。

---

#### 2.19 総資産の集計

総資産は、生成した各資産口座の`assetAmount`を合計する。

概念例：

```php
$totalAssets =
    array_sum(
        array_map(
            static fn (
                AssetAccountSummary $account,
            ): int =>
                $account->assetAmount,
            $assetAccounts,
        ),
    );
```

正常なデータ状態では、

```text
totalAssets
=
SUM(assetAccounts.assetAmount)
```

となることを保証する。

---

#### 2.20 利用可能資産の集計

利用可能資産は、指定年月時点で

```text
available = true
```

となる資産口座だけを合計する。

概念例：

```php
$availableAssets =
    array_sum(
        array_map(
            static fn (
                AssetAccountSummary $account,
            ): int =>
                $account->available
                    ? $account->assetAmount
                    : 0,
            $assetAccounts,
        ),
    );
```

利用可能資産の計算のために資産額を再取得しない。

資産口座別集計結果をそのまま利用する。

---

#### 2.21 0円の扱い

以下は、正常な資産額として扱う。

```text
balance = 0
value = 0
```

以下のようなtruthy / falsy判定を使用しない。

```php
if (! $balance) {
    // 0円まで不存在扱いになるため使用しない
}
```

0円の資産口座や0円の保有商品も、対象データが存在する限りレスポンスへ含める。

---

#### 2.22 QueryとUseCaseの責務分離

Queryは、データベースから必要なデータを取得することに責務を限定する。

UseCaseは、複数Queryの結果を組み合わせて指定年月資産状況を構成する。

概念的には、

```text
Query
    → データ取得

UseCase
    → 業務フロー制御
    → 資産状況構成
    → 集計

Resource
    → API形式へ変換
```

とする。

QueryでHTTPレスポンスを生成しない。

UseCaseでSQLやEloquent Query Builderを直接組み立てない。

---

#### 2.23 API Resource

`AssetSummary` DTOを、API ResourceによってAPIレスポンス形式へ変換する。

概念例：

```php
final class AssetSummaryByMonthResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'targetYearMonth'
                => $this->targetYearMonth,

            'totalAssets'
                => $this->totalAssets,

            'availableAssets'
                => $this->availableAssets,

            'assetAccounts'
                => AssetAccountSummaryResource::collection(
                    $this->assetAccounts,
                ),
        ];
    }
}
```

JSONフィールド名は、API共通方針に従ってcamelCaseとする。

AST-001とレスポンス項目が同一である場合は、共通の`AssetSummaryResource`を使用してよい。

---

#### 2.24 AssetAccountSummaryResource

資産口座別資産状況は、専用Resourceへ変換する。

概念例：

```php
final class AssetAccountSummaryResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'assetAccountId'
                => (string)
                    $this->assetAccountId,

            'name'
                => $this->name,

            'assetAmount'
                => $this->assetAmount,

            'available'
                => $this->available,

            'holdingAssets'
                => HoldingAssetSummaryResource::collection(
                    $this->holdingAssets,
                ),
        ];
    }
}
```

`assetAccountId`は、API共通方針に従ってstringとして返却する。

---

#### 2.25 HoldingAssetSummaryResource

保有商品別資産状況についても、専用Resourceへ変換する。

概念例：

```php
final class HoldingAssetSummaryResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'holdingAssetId'
                => (string)
                    $this->holdingAssetId,

            'name'
                => $this->name,

            'assetAmount'
                => $this->assetAmount,
        ];
    }
}
```

`holdingAssetId`は、API共通方針に従ってstringとして返却する。

---

#### 2.26 返却しない情報

API Resourceでは、指定年月資産状況の表示に必要な情報だけを返却する。

以下の情報は返却しない。

* `user_id`
* `month_end_asset_snapshot_id`
* `confirmed`
* `month_end_asset_balances.id`
* `month_end_holding_values.id`
* `balance_recording_unit`
* `asset_account_available_settings`
* `created_at`
* `updated_at`

Eloquent ModelをそのままJSON化してはならない。

---

#### 2.27 Responder

Responderは、生成済みの`AssetSummary` DTOを受け取り、API共通方針に従ったHTTPレスポンスへ変換する。

概念例：

```php
final class AssetSummaryByMonthResponder
{
    public function ok(
        AssetSummary $summary,
    ): JsonResponse {
        return response()->json(
            [
                'data' =>
                    new AssetSummaryByMonthResource(
                        $summary,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

正常時は、

```text
200 OK
```

を返却する。

`requestId`などの共通Envelope項目は、API共通レスポンス処理に従う。

---

#### 2.28 Responderの責務

Responderでは、以下を行わない。

* データベース検索
* `targetYearMonth`の形式検証
* 指定年月のsnapshot検索
* 残高記録単位の判定
* 資産口座別集計
* 保有商品別集計
* 総資産集計
* 利用可能資産集計
* 利用可能資産設定の判定
* 利用者境界の判定

Responderは、取得済みのAssetSummaryをHTTPレスポンスへ変換することに責務を限定する。

---

#### 2.29 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* JSONレスポンス共通処理
* 共通例外処理
* ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

#### 2.30 Repository

AST-002では、Repositoryを使用しない。

本APIは読み取り専用APIであり、以下を行わないためである。

* 登録
* 更新
* 削除

参照処理は、Queryクラスへ集約する。

---

#### 2.31 トランザクション

AST-002は、読み取り専用APIであるため、Phase1では明示的な`DB::transaction()`を使用しない。

以下のような実装は行わない。

```php
DB::transaction(
    function () {
        // 指定年月資産状況取得のみ
    },
);
```

複数SELECT間で厳密な読み取り一貫性が必要となる要件が将来的に追加された場合は、トランザクション分離レベルを含めて別途検討する。

---

#### 2.32 ロック

AST-002では、行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

指定年月資産状況の取得によって、他の月末資産関連処理を不要にブロックしない。

---

#### 2.33 N+1問題

資産口座ごと、保有商品ごとに個別SQLを発行する実装は避ける。

以下のような取得方法は採用しない。

```text
資産口座一覧取得
    ↓
各資産口座ごとに
月末資産残高取得
    ↓
各資産口座ごとに
保有商品取得
    ↓
各保有商品ごとに
商品別月末評価額取得
```

専用Queryによって、以下を必要な単位でまとめて取得する。

* 口座単位資産データ
* 商品単位資産データ
* 利用可能資産設定

その後、UseCaseでMap化・グルーピングしてDTOを生成する。

---

#### 2.34 取得カラム

Queryでは、指定年月資産状況の構築に必要なカラムを中心に取得する。

例えば、資産口座では、

```text
id
name
balance_recording_unit
```

月末資産残高では、

```text
asset_account_id
balance
```

保有商品では、

```text
id
asset_account_id
name
```

商品別月末評価額では、

```text
holding_asset_id
value
```

を使用する。

レスポンス生成や業務判定に不要なカラムを過剰に取得しない。

---

#### 2.35 キャッシュ

Phase1では、AST-002専用のサーバー側アプリケーションキャッシュを使用しない。

指定年月の資産状況は、確定済み月末資産データおよび対象年月時点の利用可能資産設定から算出する。

Phase1では、キャッシュ管理を追加せず、実装を単純に保つ。

性能上の必要性が生じた場合は、将来的に対象年月単位のキャッシュを検討する。

---

#### 2.36 例外変換

Laravel内部例外をそのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態                  | 独自エラーコード                             |
| --------------------- | ------------------------------------ |
| 利用者未指定                | `USER_CONTEXT_REQUIRED`         |
| 利用者ID形式不正             | `INVALID_USER_ID`               |
| 利用者不存在                | `USER_NOT_FOUND`                |
| `targetYearMonth`形式不正 | `VALIDATION_ERROR`                   |
| 指定年月の確定済み月末資産状況不存在    | `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` |
| 想定外例外                 | `INTERNAL_SERVER_ERROR`              |

指定年月の月末資産状況が未確定の場合も、

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

として扱う。

他利用者にのみ指定年月の確定済みデータが存在する場合も、同じエラーとして扱う。

---

#### 2.37 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

* SQL
* PostgreSQLの制約名
* スタックトレース
* PHP内部エラー
* Laravel内部例外メッセージ
* サーバー内部ファイルパス

詳細情報は、サーバーログへ記録する。

---

#### 2.38 ログ

AST-002では、必要に応じて以下の情報をログコンテキストへ設定する。

```text
requestId
userId
targetYearMonth
snapshotId
```

以下の具体的な金額情報は、不要にアクセスログへ出力しない。

* 総資産
* 利用可能資産
* 資産口座別残高
* 商品別月末評価額

---

#### 2.39 テスト実装方針

Laravel側では、Feature Testを中心としてAST-002のAPI契約および指定年月資産状況取得処理を確認する。

Feature Testでは、主に以下を確認する。

* `200 OK`
* `400 Bad Request`
* `404 Not Found`
* `422 Unprocessable Entity`
* `500 Internal Server Error`
* `targetYearMonth`の形式検証
* 指定した対象年月がそのまま使用されること
* 最新年月へ置き換えないこと
* 指定年月が未確定の場合に取得できないこと
* 指定年月が存在しない場合に取得できないこと
* 別年月へフォールバックしないこと
* 口座単位資産の集計
* 商品単位資産の集計
* 月末資産残高と商品別月末評価額の二重計上防止
* 複数資産口座の総資産集計
* `totalAssets`と資産口座別合計の一致
* 指定年月時点の利用可能資産設定を使用すること
* 過去月の利用可能資産状態を正しく再現すること
* 利用可能資産が0円の場合
* `balance = 0`の扱い
* `value = 0`の扱い
* 口座単位では`holdingAssets = []`となること
* 商品単位では保有商品別資産状況が返却されること
* 他利用者の同一年月snapshotを使用しないこと
* 他利用者の月末資産残高が混入しないこと
* 他利用者の商品別月末評価額が混入しないこと
* 他利用者の利用可能資産設定を使用しないこと
* AST-001と同一対象年月なら同じ集計結果となること
* IDがstringとして返却されること
* JSONフィールド名がcamelCaseであること
* 返却対象外項目が含まれないこと
* データベースが更新されないこと
* 冪等性

Requestについては、以下を確認する。

```text
2026-01
2026-12
    → 正常
```

```text
2026-1
2026/01
202601
2026-00
2026-13
abc
    → VALIDATION_ERROR
```

MonthEndAssetSnapshotQueryについては、Database Testで以下を確認する。

```text
user_id一致
+
target_year_month一致
+
confirmed = true
    ↓
取得できる
```

```text
target_year_month一致
+
confirmed = false
    ↓
null
```

```text
他利用者
+
target_year_month一致
+
confirmed = true
    ↓
null
```

AssetSummaryQueryについては、以下を確認する。

```text
口座単位
    → balanceのみ使用

商品単位
    → valueのみ使用

他利用者データ
    → 取得対象外
```

AssetAvailabilityQueryについては、以下を確認する。

```text
操作対象利用者
+
targetYearMonth
    ↓
指定年月時点の設定を取得
```

UseCaseについては、Queryの結果から期待するAssetSummaryが生成されることを確認する。

概念的には、

```text
snapshot
+
口座単位資産
+
商品単位資産
+
指定年月時点の利用可能資産設定
    ↓
GetAssetSummaryByMonthUseCase
    ↓
AssetSummary
```

をUnit Testする。

特に、

```text
totalAssets
=
SUM(assetAccounts.assetAmount)
```

および、

```text
availableAssets
=
SUM(
    available = true
    のassetAccounts.assetAmount
)
```

となることを確認する。

また、商品単位の資産口座について、

```text
assetAccount.assetAmount
=
SUM(holdingAssets.assetAmount)
```

となることを確認する。

---

### 3 関連ドキュメント
- [AST-002 API詳細設計](../../../api/details/asset-views/ast-002-create.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [資産状況 Laravelアーキテクチャ設計](./README.md)
- [AST-002 テスト設計](../../../tests/asset-views/ast-002-create.md)
- [資産状況 テスト設計](../../../tests/asset-views/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)