##  ASM-001 目的達成判定実行

### 1 概要

本ドキュメントでは、
ASM-001 目的達成判定実行APIを
Laravelで実装する際の
アーキテクチャおよび責務分離方針を定義する。

ASM-001では、
操作対象利用者の指定された目的について、
確定済みの月末資産状況、
月末資産残高、
商品別月末評価額、
手取り収入などの
判定に必要な情報を取得し、
目的達成判定を実行する。

判定結果は、
`assessment_histories`へ
目的達成判定履歴として保存する。

判定に必要な情報が不足している場合は、
例外やAPIエラーとして扱うのではなく、
業務上の判定結果である
「判定不可」として履歴を保存する。

Laravel実装では、
HTTPリクエストの受付から
目的達成判定、
判定履歴の保存、
APIレスポンスの生成までを
単一のクラスへ集約せず、
各責務を分離する。

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
    ├─ ObjectiveQuery
    ├─ MonthEndAssetSnapshotQuery
    ├─ AssetAssessmentQuery
    ├─ NetIncomeQuery
    ├─ AssessmentCalculator
    └─ AssessmentHistoryRepository
    ↓
API Resource
    ↓
Responder
```

Actionは、
Requestから受け取った入力と
利用者コンテキストをUseCaseへ引き渡し、
処理結果をResponderへ渡す
HTTP層の調整役に限定する。

UseCaseは、
目的の取得、
最新の確定済み月末資産状況の取得、
判定対象となる資産情報の取得、
手取り収入の取得、
目的達成判定の実行、
判定履歴の保存といった
ASM-001固有の業務フローを制御する。

データ取得については、
用途に応じたQueryへ責務を分離し、
目的達成判定の具体的な計算および
判定ロジックについては
`AssessmentCalculator`へ分離する。

判定結果の永続化は
`AssessmentHistoryRepository`へ委譲し、
UseCaseやActionへ
Eloquentによる保存処理を直接記述しない。

また、
APIレスポンスへの変換は
API Resource、
正常系・異常系を含む
HTTPレスポンスの生成は
Responderへ責務を分離する。

これによりASM-001では、

```text
HTTP制御
業務フロー
データ取得
目的達成判定
永続化
API表現
HTTPレスポンス生成
```

の責務を明確に分離し、
目的達成判定ロジックを
LaravelのHTTP層や
データアクセスの実装詳細へ
依存させない構成とする。

また、
目的達成判定によって変更するデータは、
今回の判定結果として新規作成する
`assessment_histories`に限定する。

目的、
月末資産状況、
月末資産残高、
商品別月末評価額、
手取り収入などの
判定元となる業務データは変更しない。

---

### 2 Laravel実装方針

ASM-001では、Action、Request、UseCase、Query、Repository、目的達成判定ロジック、Model、API Resource、Responderを分離して実装する。

概念的な処理構成は、以下とする。

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
    ├─ ObjectiveQuery
    ├─ MonthEndAssetSnapshotQuery
    ├─ AssetAssessmentQuery
    ├─ NetIncomeQuery
    ├─ AssessmentCalculator
    └─ AssessmentHistoryRepository
    ↓
API Resource
    ↓
Responder
```

目的達成判定に必要な業務フローの制御は、UseCaseへ集約する。

目的達成判定の具体的な計算ロジックは、`AssessmentCalculator`へ分離する。

Actionへ目的達成判定の業務ロジックを直接記述しない。

---

#### 2.1 Action

HTTPリクエストを受け付け、目的IDおよび利用者コンテキストを取得する。

目的達成判定UseCaseを呼び出し、処理結果をResponderへ渡す。

概念例：

```php
final class ExecuteAssessmentAction
{
    public function __invoke(
        ExecuteAssessmentRequest $request,
        ExecuteAssessmentUseCase $useCase,
        AssessmentHistoryResponder $responder,
        string $objectiveId,
    ): JsonResponse {
        $assessmentHistory = $useCase->execute(
            userId: $request->userId(),
            objectiveId: $objectiveId,
        );

        return $responder->created(
            $assessmentHistory,
        );
    }
}
```

Actionでは、以下の処理を行わない。

* 目的IDの形式検証
* 目的の検索
* 利用者境界の判定
* 目的の判定対象状態の判定
* 最新の確定済み月末資産状況の検索
* 総資産額の集計
* 直近3ヶ月の手取り収入取得
* 平均手取り収入の算出
* 判定不可条件の判定
* 目的達成判定の計算
* 目的達成判定履歴の登録
* トランザクション制御
* レスポンス形式への変換

---

#### 2.2 Request

パスパラメータの`objectiveId`を検証する。

本APIでは、リクエストボディを使用しない。

`objectiveId`は、`prepareForValidation()`でバリデーション対象へ追加する。

概念例：

```php
final class ExecuteAssessmentRequest extends FormRequest
{
    protected function prepareForValidation(): void
    {
        $this->merge([
            'objectiveId'
                => $this->route('objectiveId'),
        ]);
    }

    public function rules(): array
    {
        return [
            'objectiveId' => [
                'required',
                'integer',
                'min:1',
            ],
        ];
    }
}
```

Requestでは、以下の処理を行わない。

* 目的の存在確認
* 利用者境界の判定
* 目的の判定対象状態の判定
* 月末資産状況の取得
* 資産額の集計
* 手取り収入の取得
* 判定不可条件の判定
* 目的達成判定

これらは、入力形式の検証ではなく業務ルールとして扱う。

---

#### 2.3 UseCase

目的達成判定実行のユースケース処理を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. 目的IDを受け取る
3. 操作対象利用者に属する目的を取得する
4. 目的が判定対象となる状態であることを確認する
5. 最新の確定済み月末資産状況を取得する
6. 判定対象となる資産額を取得・集計する
7. 直近3ヶ月の手取り収入を取得する
8. 平均手取り収入を算出する
9. 判定に必要な情報が揃っているか確認する
10. 判定可能な場合は目的達成判定を実行する
11. 判定できない場合は判定不可結果を生成する
12. 目的達成判定履歴を新規登録する
13. 登録した目的達成判定履歴を返却する

概念的には、以下の流れとする。

```text
目的取得
    ↓
判定対象状態確認
    ↓
最新確定済み月末資産状況取得
    ↓
資産額取得・集計
    ↓
直近3ヶ月の手取り収入取得
    ↓
平均手取り収入算出
    ↓
判定材料確認
    ├─ 不足あり
    │      ↓
    │   判定不可結果生成
    │
    └─ 不足なし
           ↓
       AssessmentCalculator
           ↓
       判定結果生成
    ↓
AssessmentHistoryRepository
    ↓
目的達成判定履歴登録
```

判定材料が不足している場合は、例外によって処理を終了しない。

```text
判定材料不足
    ↓
AssessmentResult::unassessable()
    ↓
目的達成判定履歴登録
    ↓
201 Created
```

一方、目的が存在しないなどAPIとして処理できない状態については、業務例外を送出する。

APIエラーが発生した場合は、目的達成判定履歴を登録しない。

UseCaseでは、目的達成判定履歴以外の業務データを更新しない。

以下のデータは参照のみとする。

* `objectives`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `asset_accounts`
* `holding_assets`
* `net_incomes`

---

#### 2.4 ObjectiveQuery

操作対象利用者に属する目的を取得する。

検索条件には、必ず利用者IDを含める。

概念例：

```php
$objective =
    Objective::query()
        ->whereKey($objectiveId)
        ->where('user_id', $userId)
        ->first();
```

他の利用者に属する目的が存在する場合も、取得結果は`null`とする。

これにより、他の利用者に属する目的の存在をAPIから推測できないようにする。

Queryでは、目的達成判定を実行しない。

---

#### 2.5 MonthEndAssetSnapshotQuery

操作対象利用者に属する最新の確定済み月末資産状況を取得する。

概念例：

```php
return MonthEndAssetSnapshot::query()
    ->where('user_id', $userId)
    ->where('confirmed', true)
    ->orderByDesc('target_year_month')
    ->first();
```

未確定の月末資産状況は、判定材料として取得しない。

確定済み月末資産状況が存在しない場合は、Queryから例外を送出せず、`null`を返却する。

`null`の場合に判定不可とする判断は、UseCase側で行う。

---

#### 2.6 AssetAssessmentQuery

目的達成判定に使用する資産額を取得・集計する。

最新の確定済み月末資産状況に紐づく資産データのみを対象とする。

資産口座の残高記録単位に応じて、

```text
口座単位
    ↓
month_end_asset_balances.balance

商品単位
    ↓
month_end_holding_values.value
```

を使用する。

口座単位と商品単位の金額を重複して加算しない。

Queryは、総資産額など目的達成判定に必要な参照結果を返却する。

目的達成判定そのものは実行しない。

---

#### 2.7 NetIncomeQuery

操作対象利用者に属する直近3ヶ月分の手取り収入を取得する。

概念例：

```php
return NetIncome::query()
    ->where('user_id', $userId)
    ->orderByDesc('target_year_month')
    ->limit(3)
    ->get();
```

他の利用者の手取り収入を取得しない。

3件未満の場合も、Queryでは例外を送出しない。

取得件数が不足している場合に判定不可とする判断は、UseCase側で行う。

---

#### 2.8 平均手取り収入

直近3ヶ月分の手取り収入が取得できた場合は、平均手取り収入を算出する。

```text
直近3ヶ月の手取り収入合計
÷
3
=
平均手取り収入
```

1ヶ月または2ヶ月分のみで平均値を算出して目的達成判定へ使用しない。

3ヶ月分が揃っていない場合は、判定不可とする。

平均値の算出結果は、目的達成判定用の入力値として`AssessmentCalculator`へ渡す。

---

#### 2.9 AssessmentInput

目的達成判定ロジックへ渡す入力値は、専用のDTOまたはValue Objectとして表現する。

概念例：

```php
final readonly class AssessmentInput
{
    public function __construct(
        public Objective $objective,
        public int $totalAssets,
        public int $averageNetIncome,
    ) {
    }
}
```

`AssessmentCalculator`へEloquent ModelやQuery Builderを直接依存させることは避ける。

判定ロジックに必要な値を`AssessmentInput`へまとめて渡す。

---

#### 2.10 AssessmentCalculator

目的達成判定の具体的な計算処理を担当する。

概念例：

```php
final class AssessmentCalculator
{
    public function calculate(
        AssessmentInput $input,
    ): AssessmentResult {
        // 目的達成判定計算

        return AssessmentResult::achievable();
    }
}
```

`AssessmentCalculator`では、以下を行わない。

* データベース検索
* Eloquentによるデータ取得
* 利用者境界の判定
* 目的達成判定履歴の登録
* HTTPレスポンス生成
* ログ出力

これにより、目的達成判定ロジックをデータベースやHTTPから独立させ、Unit Test可能な構成とする。

---

#### 2.11 AssessmentResult

目的達成判定結果は、単純な文字列ではなく、EnumまたはValue Objectで表現する。

概念例：

```php
enum AssessmentResultType: string
{
    case Achievable = 'achievable';
    case Difficult = 'difficult';
    case Unassessable = 'unassessable';
}
```

判定結果に追加情報が必要な場合は、Value Objectとして表現する。

概念例：

```php
final readonly class AssessmentResult
{
    public function __construct(
        public AssessmentResultType $type,
    ) {
    }
}
```

これにより、

```text
'achievable'
'difficult'
'unassessable'
```

などのマジック文字列をUseCaseやRepositoryへ散在させない。

---

#### 2.12 判定不可

判定不可となる条件は、UseCaseで判定材料を確認し、`AssessmentResult`として表現する。

例えば、

```text
確定済み月末資産状況なし
直近3ヶ月の手取り収入不足
目的の判定材料不足
必要な資産情報を取得できない
```

などの場合は、

```php
$result =
    AssessmentResult::unassessable();
```

のように判定不可結果を生成する。

判定不可はAPIエラーではないため、業務例外を送出しない。

判定不可の場合も、目的達成判定履歴を新規登録する。

---

#### 2.13 AssessmentHistoryRepository

目的達成判定結果を、`assessment_histories`へ新規登録する。

概念例：

```php
return AssessmentHistory::create([
    'objective_id' => $objective->id,
    'result' => $result->type->value,
]);
```

ASM-001を実行するたびに、新しいレコードを登録する。

```text
ASM-001実行
    ↓
常にINSERT
```

過去の目的達成判定履歴を検索して更新しない。

以下のような実装は行わない。

```php
updateOrCreate()
```

```php
firstOrCreate()
```

同一目的について同一条件で再判定した場合も、新しい履歴を登録する。

---

#### 2.14 トランザクション

目的達成判定結果の確定から目的達成判定履歴の登録までを、トランザクション内で実行する。

概念例：

```php
$assessmentHistory =
    DB::transaction(
        function () use (
            $userId,
            $objectiveId,
        ): AssessmentHistory {
            // 判定に必要なデータ取得
            // 判定材料確認
            // 目的達成判定
            // assessment_histories登録
        },
    );
```

処理途中でAPIエラーまたは想定外例外が発生した場合は、目的達成判定履歴を登録しない。

履歴登録後に例外が発生した場合も、トランザクションをロールバックする。

ASM-001では、`assessment_histories`以外の業務データを更新しない。

---

#### 2.15 排他制御

ASM-001では、同一目的に対して複数の判定要求が同時に実行された場合でも、それぞれを独立した目的達成判定履歴として保存する。

同一目的について複数の履歴が存在することは、正常な状態である。

そのため、履歴の重複登録を防止するためのUNIQUE制約や`lockForUpdate()`は使用しない。

```text
同時実行A
    ↓
assessment_histories A

同時実行B
    ↓
assessment_histories B
```

ただし、各リクエスト内で不完全な履歴が残らないことは、トランザクションによって保証する。

---

#### 2.16 Model

目的達成判定履歴には、`AssessmentHistory` Modelを使用する。

概念例：

```php
final class AssessmentHistory extends Model
{
    protected $fillable = [
        'objective_id',
        'result',
    ];
}
```

実際の`fillable`、`casts`、リレーションについては、`assessment_histories`のテーブル定義に従う。

目的との関連は、Eloquent Relationとして定義してよい。

---

#### 2.17 API Resource

データベースカラムを直接レスポンスへ返却せず、API Resourceを使用してAPIレスポンス形式へ変換する。

概念例：

```php
final class AssessmentHistoryResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id' => (string) $this->id,
            'objectiveId'
                => (string) $this->objective_id,
            'result' => $this->result,
        ];
    }
}
```

IDは、API共通方針に従ってstringへ変換する。

データベースのsnake_caseカラムをそのまま返却しない。

---

#### 2.18 Responder

Responderは、登録済みの目的達成判定履歴を受け取り、API共通方針に従ったHTTPレスポンスへ変換する。

概念例：

```php
final class AssessmentHistoryResponder
{
    public function created(
        AssessmentHistory $history,
    ): JsonResponse {
        return response()->json(
            [
                'data' =>
                    new AssessmentHistoryResource(
                        $history,
                    ),
            ],
            Response::HTTP_CREATED,
        );
    }
}
```

正常終了時は、

```text
201 Created
```

を返却する。

判定結果が`unassessable`の場合も、正常終了として`201 Created`を返却する。

Responderでは、以下を行わない。

* データベース検索
* 目的達成判定
* 判定不可判定
* 利用者境界の判定
* 目的達成判定履歴の登録

---

#### 2.19 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* JSONレスポンス共通処理
* 共通例外処理
* ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

#### 2.20 例外

APIとして処理できない業務状態は、専用例外として表現する。

例えば、以下を想定する。

```text
ObjectiveNotFoundException
ObjectiveNotAssessableException
```

これらを共通Exception HandlerでAPIエラーコードへ変換する。

概念例：

```text
ObjectiveNotFoundException
    ↓
OBJECTIVE_NOT_FOUND
    ↓
404 Not Found
```

判定不可については、例外を使用しない。

```text
判定材料不足
    ↓
AssessmentResult::unassessable()
    ↓
assessment_historiesへ保存
    ↓
201 Created
```

---

#### 2.21 想定外例外

想定外の例外は、API共通Exception Handlerで`INTERNAL_SERVER_ERROR`へ変換する。

レスポンスへ、以下を含めない。

* SQL
* PostgreSQLの制約名
* スタックトレース
* PHP内部エラー
* Laravel内部例外メッセージ

詳細情報は、サーバーログへ記録する。

---

#### 2.22 ログ

ASM-001では、必要に応じて以下の情報をログへ記録する。

```text
requestId
userId
objectiveId
assessmentHistoryId
result
```

判定不可理由を保存する設計の場合は、判定不可理由もログへ記録してよい。

ただし、以下の情報を不要にログへ出力しない。

* 目的の詳細情報
* 総資産額
* 個別資産額
* 手取り収入額

---

#### 2.23 テスト実装方針

Laravel側では、Feature Testを中心としてASM-001のAPI契約およびユースケース全体を確認する。

Feature Testでは、主に以下を確認する。

* `201 Created`
* `400 Bad Request`
* `404 Not Found`
* `409 Conflict`
* `422 Unprocessable Entity`
* `500 Internal Server Error`
* 利用者境界
* 最新確定済み月末資産状況の選択
* 未確定月末資産状況を使用しないこと
* 総資産額の集計
* 口座単位・商品単位の重複加算防止
* 直近3ヶ月の手取り収入取得
* 平均手取り収入の算出
* 判定不可
* 判定不可の場合も履歴が登録されること
* 目的達成判定履歴の新規登録
* 同一条件で複数回実行した場合の履歴追加
* APIエラー時に履歴が登録されないこと
* 想定外例外時に不完全な履歴が残らないこと
* 他の業務データへ副作用がないこと

目的達成判定の具体的な計算ロジックについては、`AssessmentCalculator`をUnit Testの対象とする。

Unit Testでは、データベースやHTTPへ依存せず、

```text
AssessmentInput
    ↓
AssessmentCalculator
    ↓
AssessmentResult
```

を確認する。

概念例：

```php
$result =
    $calculator->calculate(
        new AssessmentInput(
            objective: $objective,
            totalAssets: 3_000_000,
            averageNetIncome: 350_000,
        ),
    );

$this->assertSame(
    AssessmentResultType::Achievable,
    $result->type,
);
```

これにより、目的達成判定の計算ロジックをLaravelのHTTP層およびデータベースアクセスから分離して独立してテストできる構成とする。

---

### 3 関連ドキュメント

- [ASM-001 API詳細設計](../../../api/details/assessments/asm-001-preview.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [目的達成判定 Laravelアーキテクチャ設計](./README.md)
- [ASM-001 テスト設計](../../../tests/assessments/asm-001-preview.md)
- [目的達成判定 テスト設計](../../../tests/assessments/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
