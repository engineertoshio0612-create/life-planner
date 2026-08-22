##  AST-003 資産推移取得

### 1. 概要

本ドキュメントでは、  
AST-003 資産推移取得APIをLaravelで実装する際の、アーキテクチャおよび責務分離方針を定義する。

AST-003では、操作対象利用者について、クエリパラメータ`from`および`to`で指定された期間内に存在する確定済み月末資産状況をもとに、対象年月ごとの資産推移を時系列で取得する。

取得対象となる月末資産状況は、操作対象利用者に属し、`confirmed = true`であり、かつ指定期間内の`target_year_month`を持つものに限定する。 :contentReference[oaicite:0]{index=0}

主に以下の情報を対象年月ごとに取得する。

- 総資産
- 利用可能資産
- 資産口座別資産額
- 保有商品別資産額
- 総資産の前月差分
- 利用可能資産の前月差分

資産額の算出では、AST-001・AST-002と同様に、資産口座の`balance_recording_unit`に応じて使用するデータを切り替える。

```text
口座単位で残高を記録する資産口座
    ↓
month_end_asset_balances.balance

商品単位で評価額を記録する資産口座
    ↓
month_end_holding_values.value

---

### 2 Laravel実装方針

AST-003では、
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
    ├─ AssetTrendQuery
    └─ AssetAvailabilityQuery
    ↓
AssetTrendResult DTO
    ↓
API Resource
    ↓
Responder
```

AST-003の業務フロー制御は、UseCaseへ集約する。

Actionへ、期間条件の判定、対象年月の抽出、資産集計、利用可能資産判定、前月差分計算、利用者境界確認などの業務ロジックを直接記述しない。

---

#### 2.1 Action

HTTPリクエストを受け付け、検証済みの以下の値および利用者コンテキストを取得する。

- `from`
- `to`

資産推移取得UseCaseを呼び出し、取得結果をResponderへ渡す。

概念例：

```php
final class GetAssetTrendAction
{
    public function __invoke(
        GetAssetTrendRequest $request,
        GetAssetTrendUseCase $useCase,
        AssetTrendResponder $responder,
        userContext $userContext,
    ): JsonResponse {
        $result = $useCase->execute(
            userId: $userContext->userId,
            from: $request->validated('from'),
            to: $request->validated('to'),
        );

        return $responder->ok(
            $result,
        );
    }
}
```

Actionでは、以下を行わない。

- `from`の形式検証
- `to`の形式検証
- `from <= to`の判定
- 確定済み月末資産状況の検索
- 月末資産残高の取得
- 商品別月末評価額の取得
- 残高記録単位の判定
- 利用可能資産設定の取得
- 月ごとの総資産集計
- 月ごとの利用可能資産集計
- 前月差分の算出
- 利用者境界の判定
- APIレスポンス形式への変換

---

#### 2.2 Request

クエリパラメータの以下を検証する。

- `from`
- `to`

本APIでは、パスパラメータおよびリクエストボディを使用しない。

概念例：

```php
final class GetAssetTrendRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'from' => [
                'required',
                new YearMonthRule(),
            ],

            'to' => [
                'required',
                new YearMonthRule(),
            ],
        ];
    }

    public function after(): array
    {
        return [
            function (
                Validator $validator,
            ): void {
                $from =
                    $this->input('from');

                $to =
                    $this->input('to');

                if (
                    is_string($from)
                    && is_string($to)
                    && $from > $to
                ) {
                    $validator->errors()
                        ->add(
                            'to',
                            '終了年月は開始年月以降を指定してください。',
                        );
                }
            },
        ];
    }
}
```

`YYYY-MM`形式では、文字列の大小比較でも年月順を比較できる。

ただし、年月比較処理を複数箇所で使用する場合は、Value Objectまたは専用Ruleへ分離してよい。

---

#### 2.3 Requestで行わないこと

Requestでは、以下の業務判定を行わない。

- 指定期間内に確定済み月末資産状況が存在するか
- 各対象年月に月末資産残高が存在するか
- 各対象年月に商品別月末評価額が存在するか
- 利用可能資産設定が存在するか
- 前月比較が可能か
- 資産推移を正常に算出できるか

これらは、入力値の形式検証ではなく、取得対象データの状態に依存するため、UseCaseおよびQueryで扱う。

---

#### 2.4 UseCase

資産推移取得のユースケース処理を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `from`を受け取る
3. `to`を受け取る
4. 指定期間内の確定済み月末資産状況を取得する
 対象snapshotが0件の場合は空の推移結果を生成する
6. snapshotId一覧を取得する
7. 対象期間分の資産データを一括取得する
8. 対象期間分の利用可能資産設定を取得する
9. snapshot単位に資産データをグルーピングする
10. 各年月の資産口座別資産状況を生成する
11. 各年月の総資産を算出する
12. 各年月の利用可能資産を算出する
13. 暦上の前月が取得結果内に存在する場合のみ前月差分を算出する
14. `AssetTrendResult` DTOを生成して返却する

概念的な処理は、以下とする。

```text
操作対象利用者
+
from
+
to
    ↓
確定済みsnapshot一覧取得
    ↓
0件
    → trends = []

1件以上
    ↓
snapshotId一覧
    ↓
対象期間の資産データ一括取得
    ↓
利用可能資産設定一括取得
    ↓
snapshot単位にグルーピング
    ↓
各年月の資産状況生成
    ↓
前月差分算出
    ↓
AssetTrendResult生成
```

UseCaseでは、SQLやEloquent Query Builderを直接組み立てない。

データベースアクセスは、Queryへ委譲する。

---

#### 2.5 MonthEndAssetSnapshotQuery

指定期間内に存在する操作対象利用者の確定済み月末資産状況を取得する。

概念例：

```php
final class MonthEndAssetSnapshotQuery
{
    public function findConfirmedBetween(
        int $userId,
        string $from,
        string $to,
    ): Collection {
        return MonthEndAssetSnapshot::query()
            ->where(
                'user_id',
                $userId,
            )
            ->where(
                'confirmed',
                true,
            )
            ->whereBetween(
                'target_year_month',
                [
                    $from,
                    $to,
                ],
            )
            ->orderBy(
                'target_year_month',
            )
            ->get([
                'id',
                'target_year_month',
            ]);
    }
}
```

検索条件には、必ず操作対象利用者IDを含める。

他の利用者に属する確定済み月末資産状況を取得対象へ含めない。

未確定の月末資産状況も資産推移へ含めない。

---

#### 2.6 対象snapshotが0件の場合

指定期間内に確定済み月末資産状況が1件も存在しない場合は、例外を送出しない。

空の資産推移を正常結果として返却する。

概念例：

```php
if ($snapshots->isEmpty()) {
    return new AssetTrendResult(
        from: $from,
        to: $to,
        trends: [],
    );
}
```

以下のような不存在例外には変換しない。

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

AST-003は一覧取得APIであるため、0件も正常な取得結果として扱う。

---

#### 2.7 snapshotId一覧

対象snapshotが存在する場合は、資産データを一括取得するため、snapshotId一覧を生成する。

概念例：

```php
$snapshotIds =
    $snapshots
        ->pluck('id')
        ->all();
```

このID一覧を、月末資産残高および商品別月末評価額の一括取得条件として使用する。

---

#### 2.8 AssetTrendQuery

対象期間に必要となる資産データをまとめて取得する。

概念例：

```php
$assetData =
    $this->assetTrendQuery
        ->findBySnapshots(
            userId: $userId,
            snapshotIds: $snapshotIds,
        );
```

Queryでは、主に以下のデータを扱う。

```text
asset_accounts
month_end_asset_balances
holding_assets
month_end_holding_values
```

対象年月ごと、資産口座ごと、保有商品ごとに個別SQLを発行しない。

---

#### 2.9 月末資産残高の一括取得

口座単位で管理する資産口座について、対象snapshot分の月末資産残高をまとめて取得する。

概念例：

```php
$balances =
    MonthEndAssetBalance::query()
        ->join(
            'asset_accounts',
            'asset_accounts.id',
            '=',
            'month_end_asset_balances.asset_account_id',
        )
        ->whereIn(
            'month_end_asset_balances.month_end_asset_snapshot_id',
            $snapshotIds,
        )
        ->where(
            'asset_accounts.user_id',
            $userId,
        )
        ->where(
            'asset_accounts.balance_recording_unit',
            BalanceRecordingUnit::ACCOUNT,
        )
        ->get([
            'month_end_asset_balances.month_end_asset_snapshot_id',
            'asset_accounts.id as asset_account_id',
            'asset_accounts.name as asset_account_name',
            'month_end_asset_balances.balance',
        ]);
```

他の利用者に属する資産口座の月末資産残高を取得しない。

---

#### 2.10 商品別月末評価額の一括取得

商品単位で管理する資産口座について、対象snapshot分の商品別月末評価額をまとめて取得する。

概念例：

```php
$holdingValues =
    MonthEndHoldingValue::query()
        ->join(
            'holding_assets',
            'holding_assets.id',
            '=',
            'month_end_holding_values.holding_asset_id',
        )
        ->join(
            'asset_accounts',
            'asset_accounts.id',
            '=',
            'holding_assets.asset_account_id',
        )
        ->whereIn(
            'month_end_holding_values.month_end_asset_snapshot_id',
            $snapshotIds,
        )
        ->where(
            'asset_accounts.user_id',
            $userId,
        )
        ->where(
            'asset_accounts.balance_recording_unit',
            BalanceRecordingUnit::HOLDING,
        )
        ->get([
            'month_end_holding_values.month_end_asset_snapshot_id',
            'asset_accounts.id as asset_account_id',
            'asset_accounts.name as asset_account_name',
            'holding_assets.id as holding_asset_id',
            'holding_assets.name as holding_asset_name',
            'month_end_holding_values.value',
        ]);
```

他の利用者に属する資産口座および保有商品を取得対象へ含めない。

---

#### 2.11 利用者境界

資産情報の取得では、必ず操作対象利用者との利用者境界を保証する。

月末資産残高では、

```text
month_end_asset_balances
    ↓
asset_accounts
    ↓
asset_accounts.user_id
        = 操作対象利用者ID
```

商品別月末評価額では、

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

#### 2.12 二重計上の防止

AST-003でも、AST-001・AST-002と同じ残高記録単位のルールを使用する。

```text
口座単位
    → month_end_asset_balances.balance

商品単位
    → month_end_holding_values.value
```

以下のような単純合計は行わない。

```text
SUM(balance)
+
SUM(value)
```

同一資産について月末資産残高と商品別月末評価額を二重計上しない。

---

#### 2.13 AssetAvailabilityQuery

利用可能資産設定は、指定期間についてまとめて取得する。

概念例：

```php
$availabilityRows =
    $this->assetAvailabilityQuery
        ->findBetween(
            userId: $userId,
            from: $from,
            to: $to,
        );
```

各対象年月について、その年月時点で有効な`asset_account_available_settings`を判定できる形で取得する。

現在時点の利用可能資産設定を全対象年月へ使い回さない。

---

#### 2.14 利用可能資産設定の利用者境界

利用可能資産設定についても、操作対象利用者に属する資産口座の設定のみを対象とする。

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

他の利用者に属する設定を資産推移の算出へ使用してはならない。

---

#### 2.15 対象年月時点の利用可能資産判定

各推移データでは、その`targetYearMonth`時点で有効な利用可能資産設定を使用する。

例えば、

```text
2026-01
    ↓
2026-01時点の設定

2026-02
    ↓
2026-02時点の設定

2026-03
    ↓
2026-03時点の設定
```

とする。

現在の最新設定だけをすべての年月へ適用してはならない。

---

#### 2.16 AST-001・AST-002との集計ロジック共通化

AST-003における1ヶ月分の資産状況算出ルールは、AST-001・AST-002と同一とする。

```text
口座単位
    → balance

商品単位
    → SUM(value)

総資産
    → assetAccounts.assetAmount合計

利用可能資産
    → available = trueの資産口座合計
```

そのため、資産口座別資産状況を組み立てる処理は、共通のBuilderなどへ切り出してよい。

概念例：

```php
final class AssetSummaryBuilder
{
    public function buildFromLoadedData(
        MonthEndAssetSnapshot $snapshot,
        Collection $balances,
        Collection $holdingValues,
        Collection $availabilityRows,
    ): AssetSummary {
        // DBアクセスを行わず、
        // 渡されたデータから集計する
    }
}
```

AST-003からこのBuilderを使用する場合、Builder内部ではSQLを発行しない。

対象年月ごとにDBアクセスが発生する設計を避ける。

---

#### 2.17 一括取得を優先する

AST-003では、対象年月数に応じてSQL発行回数が増加しない構成を基本とする。

以下のような実装は避ける。

```php
foreach ($snapshots as $snapshot) {
    $summary =
        $this->assetSummaryBuilder
            ->build(
                userId: $userId,
                snapshotId: $snapshot->id,
                targetYearMonth:
                    $snapshot->target_year_month,
            );
}
```

上記Builder内部で毎回Queryを実行すると、対象年月数に比例してSQL発行回数が増加する。

基本的には、

```text
snapshot一覧取得
    ↓
balance一括取得
    ↓
value一括取得
    ↓
利用可能資産設定一括取得
    ↓
メモリ上で年月ごとに集計
```

とする。

---

#### 2.18 月単位へのグルーピング

一括取得した資産データは、`snapshotId`をキーとしてグルーピングする。

概念例：

```php
$balancesBySnapshot =
    $balances->groupBy(
        'month_end_asset_snapshot_id',
    );

$holdingValuesBySnapshot =
    $holdingValues->groupBy(
        'month_end_asset_snapshot_id',
    );
```

利用可能資産設定についても、各`targetYearMonth`で参照しやすい形に事前整理してよい。

その後、`snapshots`の昇順を基準として月ごとの資産状況を生成する。

---

#### 2.19 AssetTrendResult DTO

資産推移全体は、Eloquent ModelやCollectionをそのままResponderへ渡さず、専用DTOとして表現する。

概念例：

```php
final readonly class AssetTrendResult
{
    public function __construct(
        public string $from,
        public string $to,
        /** @var AssetTrendItem[] */
        public array $trends,
    ) {
    }
}
```

---

#### 2.20 AssetTrendItem DTO

1ヶ月分の資産推移は、以下のようなDTOとして表現する。

概念例：

```php
final readonly class AssetTrendItem
{
    public function __construct(
        public string $targetYearMonth,
        public int $totalAssets,
        public int $availableAssets,
        public ?int $totalAssetsDifference,
        public ?int $availableAssetsDifference,
        /** @var AssetAccountSummary[] */
        public array $assetAccounts,
    ) {
    }
}
```

資産口座単位および保有商品単位のDTOは、AST-001・AST-002と共通利用してよい。

---

#### 2.21 月ごとの資産状況生成

各snapshotについて、一括取得済みのデータだけを使用して月ごとの資産状況を生成する。

概念的には、以下とする。

```text
snapshot
    ↓
口座単位データ抽出
    ↓
商品単位データ抽出
    ↓
対象年月のavailable判定
    ↓
assetAccounts生成
    ↓
totalAssets算出
    ↓
availableAssets算出
```

月ごとのDTO生成処理中に追加SQLを発行しない。

---

#### 2.22 総資産の集計

各対象年月の総資産は、その年月の各資産口座の`assetAmount`を合計する。

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

#### 2.23 利用可能資産の集計

各対象年月の利用可能資産は、

```text
available = true
```

となる資産口座の`assetAmount`のみを合計する。

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

利用可能資産集計のためだけに資産額を再取得しない。

---

#### 2.24 前月差分算出

前月差分は、単純に`trends`配列の直前要素との差分として算出してはならない。

「前月」は、暦上の前月を意味する。

例えば、

```text
2026-01
2026-03
```

のみが取得されている場合、`2026-03`の前月として`2026-01`を使用しない。

`2026-02`が存在しないため、`2026-03`の前月差分は`null`とする。

---

#### 2.25 暦上の前月の特定

各`AssetTrendItem`について、`targetYearMonth`から暦上の前月を算出する。

概念例：

```php
$previousYearMonth =
    CarbonImmutable::createFromFormat(
        'Y-m',
        $current->targetYearMonth,
    )
        ->subMonth()
        ->format('Y-m');
```

取得済みの推移データを、

```text
targetYearMonth
    → AssetTrendItem
```

のMapへ変換しておき、暦上の前月が存在するか確認してよい。

---

#### 2.26 前月差分の計算

暦上の前月が取得結果内に存在する場合のみ、差分を算出する。

概念例：

```php
$totalAssetsDifference =
    $previous === null
        ? null
        : $current->totalAssets
            - $previous->totalAssets;

$availableAssetsDifference =
    $previous === null
        ? null
        : $current->availableAssets
            - $previous->availableAssets;
```

前月データが存在しない場合は、`0`ではなく`null`とする。

---

#### 2.27 fromより前のデータを使用しない

前月差分の算出では、AST-003の取得結果内に存在するデータだけを使用する。

例えば、

```text
from = 2026-02
to = 2026-04
```

で、DBに`2026-01`の確定済みデータが存在していても、差分算出のためだけに追加取得しない。

そのため、`2026-02`については、

```text
totalAssetsDifference = null
availableAssetsDifference = null
```

とする。

---

#### 2.28 0円の扱い

0円は、有効な資産額として扱う。

例えば、

```text
2026-01
totalAssets = 0

2026-02
totalAssets = 100000
```

の場合、

```text
totalAssetsDifference = 100000
```

とする。

以下のようなtruthy / falsy判定を使用しない。

```php
if (! $previous->totalAssets) {
    // 0円を比較不可扱いしてしまうため使用しない
}
```

---

#### 2.29 欠損月を補完しない

UseCaseでは、`from`から`to`までの全年月を生成して0円データで補完しない。

以下のような処理は行わない。

```text
2026-01
2026-02
2026-03

2026-02 snapshotなし
    ↓
2026-02を0円で生成
```

確定済みsnapshotとして実際に取得できた年月のみを`trends`へ含める。

---

#### 2.30 並び順

資産推移は、

```text
target_year_month ASC
```

の順序で返却する。

MonthEndAssetSnapshotQueryの取得時点で昇順を保証する。

UseCaseでは、その順序を維持して`AssetTrendItem`を生成する。

Responderやフロントエンドでの再ソートを前提としない。

---

#### 2.31 API Resource

資産推移全体は、専用API Resourceへ変換する。

概念例：

```php
final class AssetTrendResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'from'
                => $this->from,

            'to'
                => $this->to,

            'trends'
                => AssetTrendItemResource::collection(
                    $this->trends,
                ),
        ];
    }
}
```

データベースカラムを直接JSONへ返却しない。

---

#### 2.32 AssetTrendItemResource

1ヶ月分の資産推移は、専用Resourceへ変換する。

概念例：

```php
final class AssetTrendItemResource
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

            'totalAssetsDifference'
                => $this->totalAssetsDifference,

            'availableAssetsDifference'
                => $this->availableAssetsDifference,

            'assetAccounts'
                => AssetAccountSummaryResource::collection(
                    $this->assetAccounts,
                ),
        ];
    }
}
```

資産口座別および保有商品別のResourceは、AST-001・AST-002と共通利用してよい。

---

#### 2.33 返却しない情報

API Resourceでは、資産推移表示に必要な情報だけを返却する。

以下の情報は返却しない。

- `user_id`
- `month_end_asset_snapshot_id`
- `confirmed`
- `month_end_asset_balances.id`
- `month_end_holding_values.id`
- `balance_recording_unit`
- `asset_account_available_settings`
- `created_at`
- `updated_at`

Eloquent ModelやQuery結果をそのままJSON化してはならない。

---

#### 2.34 Responder

Responderは、生成済みの`AssetTrendResult`を受け取り、API共通方針に従ったHTTPレスポンスへ変換する。

概念例：

```php
final class AssetTrendResponder
{
    public function ok(
        AssetTrendResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data' =>
                    new AssetTrendResource(
                        $result,
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

対象snapshotが0件の場合も、

```json
{
  "data": {
    "from": "2026-01",
    "to": "2026-03",
    "trends": []
  }
}
```

として`200 OK`を返却する。

---

#### 2.35 Responderの責務

Responderでは、以下を行わない。

- データベース検索
- `from`、`to`の検証
- 対象snapshot抽出
- 残高記録単位の判定
- 資産口座別集計
- 総資産集計
- 利用可能資産集計
- 前月差分計算
- 利用者境界判定
- 並び順制御

Responderは、生成済みの`AssetTrendResult`をHTTPレスポンスへ変換することに責務を限定する。

---

#### 2.36 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONレスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

#### 2.37 Repository

AST-003では、Repositoryを使用しない。

本APIは読み取り専用APIであり、以下を行わないためである。

- 登録
- 更新
- 削除

データベース参照は、Queryクラスへ集約する。

---

#### 2.38 トランザクション

AST-003は、読み取り専用APIであるため、Phase1では明示的な`DB::transaction()`を使用しない。

以下のような実装は行わない。

```php
DB::transaction(
    function () {
        // 資産推移取得のみ
    },
);
```

複数SELECT間で厳密な読み取り一貫性が必要となる要件が将来的に追加された場合は、トランザクション分離レベルを含めて別途検討する。

---

#### 2.39 ロック

AST-003では、行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

資産推移取得によって、月末資産状況の確定・確定解除などの更新処理を不要にブロックしない。

---

#### 2.40 N+1問題

AST-003では、対象期間に含まれる月数に比例してSQL発行回数が増加しないようにする。

以下のような実装は行わない。

```php
foreach ($snapshots as $snapshot) {
    $balances =
        MonthEndAssetBalance::query()
            ->where(
                'month_end_asset_snapshot_id',
                $snapshot->id,
            )
            ->get();

    $holdingValues =
        MonthEndHoldingValue::query()
            ->where(
                'month_end_asset_snapshot_id',
                $snapshot->id,
            )
            ->get();
}
```

基本的には、

```text
snapshot一覧
    ↓
whereIn(snapshotIds)
    ↓
月末資産残高一括取得
    ↓
商品別月末評価額一括取得
    ↓
利用可能資産設定一括取得
```

とする。

取得後に、アプリケーション上で年月ごとにグルーピングする。

---

#### 2.41 取得カラム

Queryでは、資産推移構築に必要なカラムを中心に取得する。

例えば、月末資産状況では、

```text
id
target_year_month
```

月末資産残高では、

```text
month_end_asset_snapshot_id
asset_account_id
asset_account_name
balance
```

商品別月末評価額では、

```text
month_end_asset_snapshot_id
asset_account_id
asset_account_name
holding_asset_id
holding_asset_name
value
```

を使用する。

レスポンス生成や業務判定に不要なカラムを過剰に取得しない。

---

#### 2.42 キャッシュ

Phase1では、AST-003専用のサーバー側アプリケーションキャッシュを使用しない。

資産推移は、以下によって対象となる年月が変化する可能性がある。

- 月末資産状況の確定
- 月末資産状況の確定解除

そのため、リクエストごとにデータベースから最新の確定済みデータを取得して資産推移を算出する。

---

#### 2.43 例外変換

Laravel内部例外をそのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `from`未指定・形式不正 | `VALIDATION_ERROR` |
| `to`未指定・形式不正 | `VALIDATION_ERROR` |
| `from > to` | `VALIDATION_ERROR` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

指定期間に確定済み月末資産状況が存在しないことは、エラーへ変換しない。

---

#### 2.44 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- SQL
- PostgreSQLの制約名
- スタックトレース
- PHP内部エラー
- Laravel内部例外メッセージ
- サーバー内部ファイルパス

詳細情報は、サーバーログへ記録する。

---

#### 2.45 ログ

AST-003では、必要に応じて以下の情報をログコンテキストへ設定する。

```text
requestId
userId
from
to
targetMonthCount
```

以下の具体的な金額情報は、不要にアクセスログへ出力しない。

- 総資産
- 利用可能資産
- 資産口座別残高
- 商品別月末評価額
- 前月差分

---

#### 2.46 テスト実装方針

Laravel側では、Feature Testを中心としてAST-003のAPI契約および資産推移取得処理を確認する。

Feature Testでは、主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `422 Unprocessable Entity`
- `500 Internal Server Error`
- `from`必須
- `to`必須
- `from`の`YYYY-MM`形式
- `to`の`YYYY-MM`形式
- `from <= to`
- 指定期間内の確定済みsnapshotのみ取得されること
- 未確定snapshotが除外されること
- 他利用者のsnapshotが除外されること
- 対象snapshotが0件の場合に`trends = []`となること
- `targetYearMonth`昇順となること
- 欠損月を補完しないこと
- 口座単位資産の集計
- 商品単位資産の集計
- 月末資産残高と商品別月末評価額の二重計上防止
- 対象年月時点の利用可能資産設定を使用すること
- 各年月の`totalAssets`が正しいこと
- 各年月の`availableAssets`が正しいこと
- 暦上の前月が存在する場合に差分が算出されること
- 暦上の前月が存在しない場合に差分が`null`となること
- 欠損月を飛ばして前月比較しないこと
- `from`より前のデータを前月比較へ使用しないこと
- 0円からの差分が正しく算出されること
- AST-001・AST-002と同一対象年月で集計結果が一致すること
- IDがstringとして返却されること
- JSONフィールド名がcamelCaseであること
- 返却対象外項目が含まれないこと
- データベースが更新されないこと
- 冪等性

Requestについては、以下を確認する。

```text
from = 2026-01
to = 2026-12
    → 正常
```

```text
from = 2026-1
from = 2026/01
from = 2026-13

to = 2026-1
to = 2026/01
to = 2026-13
    → VALIDATION_ERROR
```

```text
from = 2026-12
to = 2026-01
    → VALIDATION_ERROR
```

MonthEndAssetSnapshotQueryについては、Database Testで以下を確認する。

```text
user_id一致
+
confirmed = true
+
from <= target_year_month <= to
    ↓
target_year_month ASC
```

AssetTrendQueryについては、以下を確認する。

```text
snapshotId一覧
+
操作対象利用者
    ↓
対象期間の資産情報を
一括取得
```

他利用者の以下のデータが取得結果へ混入しないことを確認する。

- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品

AssetAvailabilityQueryについては、以下を確認する。

```text
操作対象利用者
+
from〜to
    ↓
各対象年月で判定可能な
利用可能資産設定を取得
```

UseCaseについては、一括取得済みデータから期待する`AssetTrendResult`が生成されることをUnit Testする。

概念的には、

```text
snapshot一覧
+
月末資産残高一覧
+
商品別月末評価額一覧
+
利用可能資産設定
    ↓
GetAssetTrendUseCase
    ↓
AssetTrendResult
```

を確認する。

特に、以下をUnit Testする。

```text
totalAssets
availableAssets
totalAssetsDifference
availableAssetsDifference
```

前月差分については、以下を明示的に確認する。

```text
2026-01あり
2026-02あり
    ↓
2026-02は2026-01と比較
```

```text
2026-01あり
2026-02なし
2026-03あり
    ↓
2026-03の差分はnull
```

```text
from = 2026-02
DBには2026-01あり
取得結果は2026-02から
    ↓
2026-02の差分はnull
```

---

## 3. 関連ドキュメント

- [AST-003 API詳細設計](../../../api/details/asset-views/ast-003-update.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [資産状況 Laravelアーキテクチャ設計](./README.md)
- [AST-003 テスト設計](../../../tests/asset-views/ast-003-update.md)
- [資産状況 テスト設計](../../../tests/asset-views/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
