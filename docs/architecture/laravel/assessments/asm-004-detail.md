# ASM-004 判定履歴詳細取得

### 1 概要

本ドキュメントでは、
ASM-004 判定履歴詳細取得APIを
Laravelで実装する際の
アーキテクチャおよび責務分離方針を定義する。

ASM-004では、
操作対象利用者に紐づく
指定された目的達成判定履歴1件について、
`assessment_histories`に保存された判定結果および
判定実行時点の計算根拠を取得する。

本APIは、
ASM-002によって保存された
過去の目的達成判定履歴を参照するための
読み取り専用APIであり、
目的達成判定そのものは実行しない。

また、
現在の目的、資産状況、手取り収入、
利用可能資産設定などを使用して
判定結果や計算根拠を再計算しない。

判定履歴詳細の正は、
ASM-002実行時に保存された
`assessment_histories`の情報とする。

必要に応じて、
目的情報や判定対象となった
`month_end_asset_snapshots`の情報を参照するが、
現在状態から過去の判定結果を再構築しない。

Laravel実装では、
HTTPリクエストの受付から
利用者境界を含む判定履歴の取得、
関連情報の取得、
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
Action
    ↓
UseCase
    ↓
Query
    ↓
Result DTO
    ↓
API Resource
    ↓
Responder
```

ASM-004は参照専用APIであるため、
データ取得にはQueryを使用し、
Repositoryは原則として使用しない。

また、
判定履歴の取得時点から
操作対象利用者の境界を検索条件へ含め、
他利用者に属する判定履歴の存在を
クライアントへ公開しない。

目的が現在無効化されている場合や、
判定対象となった月末資産状況が
後から確定解除されている場合でも、
保存済み判定履歴は
過去の記録として参照可能とする。

ASM-004のLaravel実装責務は、

```text
保存済み判定履歴を
利用者境界を守って取得し、
判定時点の結果と計算根拠を
変更せず返却する
```

ことに限定する。

---

## 2. Laravel実装方針

ASM-004では、Action、UseCase、Query、DTO、API Resource、Responderを分離して実装する。

参照専用GET APIであるため、Repositoryは原則として使用しない。

概念的な構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ AssessmentHistoryQuery
    ├─ ObjectiveQuery
    └─ MonthEndAssetSnapshotQuery
    ↓
AssessmentHistoryDetailResult DTO
    ↓
API Resource
    ↓
Responder
```

ASM-004では、保存済みの判定履歴を正として取得し、現在の資産状況から目的達成判定を再計算しない。

---

### 2.1 Route

ASM-004は、以下のルートとして定義する。

概念例：

```php
Route::get(
    '/api/v1/assessment-histories/{assessmentHistoryId}',
    ShowAssessmentHistoryAction::class,
);
```

`assessmentHistoryId`は、正の整数形式のみ許可する。

概念例：

```php
Route::get(
    '/api/v1/assessment-histories/{assessmentHistoryId}',
    ShowAssessmentHistoryAction::class,
)
    ->where(
        'assessmentHistoryId',
        '[1-9][0-9]*',
    );
```

ただし、形式不正時に

```text
INVALID_ASSESSMENT_HISTORY_ID
```

を返却する共通方針がある場合は、Route制約だけで完結させず、共通のパスパラメータ検証方式に従う。

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

---

### 2.3 FormRequest

ASM-004では、

```text
Request Bodyなし
Query Parameterなし
```

であるため、専用FormRequestは原則として作成しない。

以下のような空FormRequestを形式的に追加しない。

```php
final class ShowAssessmentHistoryRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [];
    }
}
```

`assessmentHistoryId`は、RouteまたはAPI共通のパスパラメータ検証方式で扱う。

---

### 2.4 assessmentHistoryIdの形式検証

`assessmentHistoryId`は、正の整数形式を必須とする。

正常例：

```text
1
100
999
```

不正例：

```text
0
-1
abc
1.5
1e3
100abc
```

形式不正の場合は、

```text
INVALID_ASSESSMENT_HISTORY_ID
```

へ変換する。

---

### 2.5 Action

Actionは、

```text
assessmentHistoryId
+
UserContext
```

を受け取り、UseCaseを呼び出す。

概念例：

```php
final class ShowAssessmentHistoryAction
{
    public function __invoke(
        string $assessmentHistoryId,
        ShowAssessmentHistoryUseCase $useCase,
        ShowAssessmentHistoryResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                assessmentHistoryId:
                    (int) $assessmentHistoryId,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

---

### 2.6 Actionで行わないこと

Actionでは、以下を行わない。

- 判定履歴検索
- 利用者境界判定
- 目的検索
- 月末資産状況検索
- 判定結果の再計算
- 計算根拠の再生成
- DBアクセス
- レスポンス配列生成
- エラーコード判定

Actionは、

```text
HTTP入力
    ↓
UseCase
    ↓
Responder
```

の橋渡しに責務を限定する。

---

### 2.7 UseCase

ASM-004のアプリケーション処理全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `assessmentHistoryId`を受け取る
3. 利用者境界を含めて判定履歴を取得する
4. 対象不存在の場合は業務例外を送出する
5. 必要に応じて目的情報を取得する
6. 必要に応じて月末資産状況を取得する
7. 保存済みの判定結果と計算根拠を取得する
8. Result DTOへ変換する
9. Result DTOを返却する

---

### 2.8 UseCaseの概念フロー

```text
userId
+
assessmentHistoryId
    ↓
AssessmentHistoryQuery
    ↓
判定履歴取得
    ↓
存在する？
    ├─ No
    │    ↓
    │ ASSESSMENT_HISTORY_NOT_FOUND
    │
    └─ Yes
         ↓
      必要な関連情報取得
         ↓
      保存済み判定結果取得
         ↓
      保存済み計算根拠取得
         ↓
      Result DTO
```

---

### 2.9 AssessmentHistoryQuery

判定履歴の取得は、専用Queryへ委譲する。

概念例：

```php
$assessmentHistory =
    $this->assessmentHistoryQuery
        ->findDetailByIdAndUser(
            assessmentHistoryId:
                $assessmentHistoryId,

            userId:
                $userId,
        );
```

ASM-004ではQueryが読み取り責務を担う。

---

### 2.10 利用者境界をQueryへ含める

判定履歴を、

```php
AssessmentHistory::find(
    $assessmentHistoryId,
);
```

のようにIDだけで取得した後に利用者を確認する方式を基本としない。

対象取得時点から利用者境界を含める。

`assessment_histories`に`user_id`を持つ場合は、概念的に、

```text
assessment_histories.id
    = assessmentHistoryId

AND

assessment_histories.user_id
    = userId
```

とする。

---

### 2.11 assessment_historiesにuser_idを持たない場合

`assessment_histories`に直接`user_id`を持たない場合は、関連する目的を介して利用者境界を保証する。

概念的には、

```text
assessment_histories
    ↓ objective_id
objectives
    ↓ user_id
```

として、

```text
objectives.user_id
    = userId
```

をQuery条件へ含める。

実際の方式は、テーブル定義を正とする。

---

### 2.12 ASSESSMENT_HISTORY_NOT_FOUND

対象判定履歴を取得できない場合は、

```php
throw new
    AssessmentHistoryNotFoundException();
```

とする。

以下を同じ例外へ集約する。

- 判定履歴不存在
- 他利用者所属
- SoftDeletes採用時の論理削除済み

最終的に、

```text
404 Not Found
ASSESSMENT_HISTORY_NOT_FOUND
```

へ変換する。

---

### 2.13 他利用者所属専用例外を作らない

以下のような専用例外は作成しない。

```text
AssessmentHistoryForbiddenException
```

他利用者所属も`AssessmentHistoryNotFoundException`として扱う。

他利用者の履歴存在をクライアントへ公開しない。

---

### 2.14 判定履歴を取得の起点とする

ASM-004では、取得の起点を

```text
assessment_histories
```

とする。

以下のように現在の目的や現在の月末資産状況から判定結果を再構築しない。

```text
objectives
+
month_end_asset_snapshots
+
net_incomes
+
asset_account_available_settings
    ↓
再判定
```

ASM-004ではこの処理を行わない。

---

### 2.15 判定結果は保存済み値を使用する

例えば、

```text
assessment_result
available_asset_amount
required_amount
average_net_income
required_emergency_fund
remaining_amount
not_assessable_reason
```

などが`assessment_histories`へ保存されている場合は、その値をそのまま使用する。

---

### 2.16 現在の資産情報を参照しない

判定結果・計算根拠の取得目的で、

```text
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
net_incomes
```

を再参照しない。

これらはASM-001・ASM-002の判定計算処理で使用する。

---

### 2.17 現在値から再計算しない

UseCase内で、

```php
$availableAssets =
    $calculator->calculate(...);

$result =
    $assessmentService->assess(...);
```

のような判定ロジックを呼び出さない。

ASM-004は、保存済み判定履歴取得に責務を限定する。

---

### 2.18 ObjectiveQuery

目的情報を別Queryで取得する必要がある場合は、`ObjectiveQuery`を使用する。

概念例：

```php
$objective =
    $this->objectiveQuery
        ->findForAssessmentHistory(
            objectiveId:
                $assessmentHistory->objectiveId,
        );
```

ただし、判定時点の目的情報を`assessment_histories`へスナップショット保存している場合は、不要な目的Queryを行わない。

---

### 2.19 目的の現在のenabledを取得条件にしない

目的を参照する場合でも、

```text
enabled = true
```

を取得条件へ含めない。

現在無効化済みの目的でも、過去の判定履歴は参照可能とする。

---

### 2.20 目的のSoftDeletes

目的が論理削除済みでも過去判定履歴を参照可能とする仕様の場合は、EloquentのSoftDeletes Global Scopeに注意する。

必要に応じて、

```php
withTrashed()
```

を使用する、または判定時点の目的名等を履歴へ保存して現在の`objectives`への依存を減らす。

---

### 2.21 objectiveNameの取得方針

判定時点の目的名を`assessment_histories`へ保存している場合は、保存済み値を使用する。

概念例：

```php
$objectiveName =
    $assessmentHistory
        ->objective_name;
```

保存していない場合だけ、`objectives.name`を参照する。

---

### 2.22 判定時点情報はスナップショット保存を優先する

ASM-004で過去時点の値を正確に再現する必要がある情報は、

```text
assessment_histories
```

へ保存する設計を優先する。

例えば、

```text
objectiveName
requiredAmount
averageNetIncome
requiredEmergencyFund
availableAssetAmount
remainingAmount
```

などである。

現在テーブルを参照して過去値を推測しない。

---

### 2.23 MonthEndAssetSnapshotQuery

判定対象年月などを`month_end_asset_snapshots`から取得する場合は、専用Queryを使用する。

概念例：

```php
$snapshot =
    $this->monthEndAssetSnapshotQuery
        ->findForAssessmentHistory(
            snapshotId:
                $assessmentHistory
                    ->monthEndAssetSnapshotId,
        );
```

---

### 2.24 confirmedを取得条件にしない

月末資産状況を取得する場合でも、

```text
confirmed = true
```

を条件へ含めない。

判定履歴保存後に確定解除されていても、過去履歴は参照可能とする。

---

### 2.25 targetYearMonthの取得方針

判定時点の対象年月を`assessment_histories`へ保存している場合は、保存値を使用する。

保存していない場合は、

```text
month_end_asset_snapshots.target_year_month
```

を参照する。

過去時点の表示内容を完全に固定したい場合は、履歴側への保存を優先する。

---

### 2.26 JOINでまとめて取得してよい

ASM-004では、必要な情報が明確であるため、`AssessmentHistoryQuery`内で

```text
assessment_histories
+
objectives
+
month_end_asset_snapshots
```

をJOINして1回のQueryで取得してもよい。

概念例：

```php
AssessmentHistory::query()
    ->select([
        'assessment_histories.id',
        'assessment_histories.objective_id',
        'assessment_histories.assessment_result',
        'assessment_histories.available_asset_amount',
        'assessment_histories.required_amount',
        'assessment_histories.average_net_income',
        'assessment_histories.required_emergency_fund',
        'assessment_histories.remaining_amount',
        'assessment_histories.not_assessable_reason',
        'assessment_histories.assessed_at',
        'objectives.name as objective_name',
        'month_end_asset_snapshots.target_year_month',
    ])
    ->join(
        'objectives',
        'objectives.id',
        '=',
        'assessment_histories.objective_id',
    )
    ->join(
        'month_end_asset_snapshots',
        'month_end_asset_snapshots.id',
        '=',
        'assessment_histories.month_end_asset_snapshot_id',
    );
```

実際のカラム名は、テーブル定義を正とする。

---

### 2.27 JOIN時の利用者境界

JOINで取得する場合も、利用者境界を必ずQuery条件へ含める。

例えば、`objectives.user_id`を利用者境界とする場合は、

```php
->where(
    'objectives.user_id',
    $userId,
)
```

を含める。

---

### 2.28 JOIN時のSoftDeletesに注意する

目的や月末資産状況にSoftDeletesを採用している場合、通常のJOINではEloquent Global Scopeが期待どおり適用されない場合がある。

ASM-004の

```text
過去履歴を現在状態に依存せず参照する
```

という仕様に合わせて、JOIN条件を明示する。

---

### 2.29 N+1を発生させない

ASM-004は単一履歴取得APIなので大量N+1は発生しにくいが、関連情報を1項目ずつ不要にQueryしない。

例えば、

```text
履歴取得
    ↓
目的取得
    ↓
月末資産状況取得
```

の3Queryが明確で十分軽量なら許容できる。

一方、必要情報をJOINで自然に取得できる場合は、Queryをまとめてもよい。

過剰な最適化も避ける。

---

### 2.30 Repositoryを使用しない

ASM-004ではデータ更新を行わないため、Repositoryは原則として使用しない。

概念的には、

```text
Query
    → SELECT

Repository
    → INSERT / UPDATE / DELETE
```

というプロジェクト共通方針に従う。

---

### 2.31 Queryで更新しない

`AssessmentHistoryQuery`では、以下を行わない。

- 判定履歴更新
- 判定履歴削除
- 計算根拠補完
- 目的更新
- 月末資産状況更新
- 再判定

Queryは読み取りだけに責務を限定する。

---

### 2.32 Result DTO

取得した判定履歴詳細を、専用Result DTOとして表現する。

概念例：

```php
final readonly class AssessmentHistoryDetailResult
{
    public function __construct(
        public int $id,
        public int $objectiveId,
        public string $objectiveName,
        public string $targetYearMonth,
        public string $assessmentResult,
        public ?int $availableAssetAmount,
        public ?int $requiredAmount,
        public ?int $averageNetIncome,
        public ?int $requiredEmergencyFund,
        public ?int $remainingAmount,
        public ?string $notAssessableReason,
        public string $assessedAt,
    ) {
    }
}
```

具体的な項目・NULL可否は、ASM-002の保存仕様を正とする。

---

### 2.33 DTOへEloquent Modelを保持しない

以下のようなResult DTOは基本としない。

```php
final readonly class AssessmentHistoryDetailResult
{
    public function __construct(
        public AssessmentHistory $model,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

---

### 2.34 Enumを使用してよい

`assessmentResult`や`notAssessableReason`をEnumとして管理している場合は、UseCaseまたはQuery結果変換時に共通Enumを利用する。

例えば、

```php
AssessmentResult::from(
    $row->assessment_result,
);
```

とする。

ASM-004専用の異なるコード定義を作らない。

---

### 2.35 ASM-001・ASM-002とEnumを共通化する

以下の値は、ASM系APIで共通化する。

```text
ACHIEVABLE
NOT_ACHIEVABLE
NOT_ASSESSABLE
```

判定不可理由についても、ASM-001・ASM-002・ASM-004で同じEnumまたはValue Objectを使用する。

---

### 2.36 API Resource

Result DTOを、専用API ResourceでJSON形式へ変換する。

概念例：

```php
final class AssessmentHistoryDetailResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'objectiveId'
                => (string) $this->objectiveId,

            'objectiveName'
                => $this->objectiveName,

            'targetYearMonth'
                => $this->targetYearMonth,

            'assessmentResult'
                => $this->assessmentResult,

            'availableAssetAmount'
                => $this->availableAssetAmount,

            'requiredAmount'
                => $this->requiredAmount,

            'averageNetIncome'
                => $this->averageNetIncome,

            'requiredEmergencyFund'
                => $this->requiredEmergencyFund,

            'remainingAmount'
                => $this->remainingAmount,

            'notAssessableReason'
                => $this->notAssessableReason,

            'assessedAt'
                => $this->assessedAt,
        ];
    }
}
```

---

### 2.37 IDをstringへ変換する

API Resourceでは、

```php
(string) $this->id
```

のように、`bigint`のIDをstringへ変換する。

主に、

```text
id
objectiveId
```

が対象となる。

---

### 2.38 金額を文字列化しない

以下の金額項目は、

```text
availableAssetAmount
requiredAmount
averageNetIncome
requiredEmergencyFund
remainingAmount
```

integerまたはnullとして返却する。

以下のような表示用文字列へ変換しない。

```php
number_format(
    $this->availableAssetAmount,
) . '円';
```

表示フォーマットはReact側で行う。

---

### 2.39 日時変換

`assessedAt`は、API共通方針の日時形式へ変換する。

例えば、Carbonを使用する場合は、

```php
$this->assessedAt
    ->toIso8601String();
```

などを利用してよい。

ただし、タイムゾーン・形式はAPI共通方針を正とする。

---

### 2.40 Resourceで再計算しない

API Resourceでは、

```text
availableAssetAmount
-
requiredAmount
-
requiredEmergencyFund
```

などを行って`remainingAmount`を再計算しない。

Resourceは表現変換だけを担当する。

---

### 2.41 Responder

Responderは、Result DTOを`200 OK`レスポンスへ変換する。

概念例：

```php
final class ShowAssessmentHistoryResponder
{
    public function ok(
        AssessmentHistoryDetailResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    =>
                    new AssessmentHistoryDetailResource(
                        $result,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

実際のEnvelope生成は、API共通方針に従う。

---

### 2.42 Responderで行わないこと

Responderでは、以下を行わない。

- 判定履歴検索
- 利用者境界判定
- 関連目的検索
- 月末資産状況検索
- 再判定
- 計算根拠生成
- DBアクセス
- 業務例外判定

HTTPレスポンス生成だけに責務を限定する。

---

### 2.43 トランザクションを使用しない

ASM-004は参照専用GET APIであるため、

```php
DB::transaction(...)
```

を原則として使用しない。

複数更新を原子的に保証する必要がないためである。

---

### 2.44 lockForUpdateを使用しない

ASM-004では、

```php
lockForUpdate()
```

を使用しない。

判定履歴、目的、月末資産状況を参照するだけであり、更新APIを不要にブロックしない。

---

### 2.45 キャッシュ

Phase1では、ASM-004専用のサーバー側キャッシュを原則として導入しない。

React側では、TanStack QueryによるQuery Cacheを使用してよい。

---

### 2.46 例外変換

主な例外変換は、以下とする。

| 内部状態 | エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `assessmentHistoryId`形式不正 | `INVALID_ASSESSMENT_HISTORY_ID` |
| 判定履歴不存在 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| 他利用者所属 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| 論理削除済み判定履歴 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| 関連データの想定外不整合 | `INTERNAL_SERVER_ERROR` |
| その他想定外例外 | `INTERNAL_SERVER_ERROR` |

---

### 2.47 AssessmentHistoryNotFoundException

対象履歴を取得できない場合は、

```php
throw new
    AssessmentHistoryNotFoundException();
```

とする。

最終的に、

```text
404 Not Found
ASSESSMENT_HISTORY_NOT_FOUND
```

へ変換する。

---

### 2.48 関連データ不整合

例えば、

```text
assessment_histories.objective_id
    ↓
objectives不存在
```

または、

```text
assessment_histories.month_end_asset_snapshot_id
    ↓
month_end_asset_snapshots不存在
```

などの状態は、利用者入力エラーとして扱わない。

外部キー制約で原則として防止する。

万一発生した場合は、

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

として扱う。

---

### 2.49 保存済み判定根拠の欠落

テーブル定義上必須となっている判定根拠が欠落している場合も、現在値から補完しない。

例えば、

```text
assessmentResult = ACHIEVABLE

しかし
availableAssetAmount = NULL
```

が仕様上あり得ない場合は、内部データ不整合として扱う。

---

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
- テーブル名
- カラム名
- 制約名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

---

### 2.51 ログ

ASM-004では、必要に応じて以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assessmentHistoryId
httpStatus
errorCode
```

`apiId`は、

```text
ASM-004
```

とする。

---

### 2.52 正常時ログ

正常終了時は、必要に応じて

```text
requestId
userId
apiId = ASM-004
assessmentHistoryId
httpStatus = 200
```

を記録する。

判定根拠となる詳細な金額を通常のアクセスログへ不要に出力しない。

---

### 2.53 Not Found時ログ

`ASSESSMENT_HISTORY_NOT_FOUND`の場合は、必要に応じて

```text
requestId
userId
assessmentHistoryId
errorCode
httpStatus
```

を記録する。

ただし、

```text
本当に不存在だった
他利用者所属だった
```

という内部判定理由をAPIレスポンスへは公開しない。

---

### 2.54 Feature Test

LaravelのFeature Testでは、主に以下を確認する。

- 正常取得
- `200 OK`
- IDのstring変換
- 金額のinteger変換
- 判定結果コード
- 判定不可理由
- `assessedAt`の日時形式
- `X-User-Id`未指定
- `X-User-Id`形式不正
- 利用者不存在
- `assessmentHistoryId`形式不正
- 判定履歴不存在
- 他利用者境界
- 目的無効化後も参照可能
- 月末資産状況確定解除後も参照可能
- 現在資産から再計算しない
- 現在手取り収入から再計算しない
- 副作用なし

---

### 2.55 AssessmentHistoryQueryのDatabase Test

以下の条件で正しく取得できることを確認する。

```text
assessmentHistoryId一致
+
利用者境界一致
```

また、以下を取得できないことを確認する。

- 存在しない判定履歴
- 他利用者の判定履歴
- 論理削除済み判定履歴
  - SoftDeletes採用時のみ

---

### 2.56 利用者境界Test

例えば、

```text
User 1
    History 100

User 2
    History 200
```

という状態で、

```text
userId = 1
assessmentHistoryId = 200
```

を指定した場合、Query結果が`null`となることを確認する。

---

### 2.57 保存済み値取得Test

判定履歴に、

```text
assessmentResult
availableAssetAmount
requiredAmount
averageNetIncome
requiredEmergencyFund
remainingAmount
notAssessableReason
```

を設定しておき、Result DTOへ同じ値が設定されることを確認する。

---

### 2.58 現在値変更後のTest

判定履歴保存後に、

```text
net_incomes
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
```

を変更する。

その後ASM-004を実行しても、保存済みの判定根拠が変更されないことを確認する。

---

### 2.59 目的無効化後のTest

判定履歴保存後に対象目的を

```text
enabled = false
```

へ変更する。

その後もASM-004で履歴詳細を取得できることを確認する。

---

### 2.60 月末資産状況確定解除後のTest

判定履歴保存後に対象月末資産状況を

```text
confirmed = false
```

へ変更する。

その後もASM-004で履歴詳細を取得できることを確認する。

---

### 2.61 API Resource Test

API Resourceでは、主に以下を確認する。

```text
id
    → string

objectiveId
    → string

snake_case
    → camelCase

金額
    → integer / null

日時
    → API共通形式
```

---

### 2.62 判定不可Resource Test

判定結果が

```text
NOT_ASSESSABLE
```

の場合に、

```json
{
  "assessmentResult": "NOT_ASSESSABLE",
  "notAssessableReason": "..."
}
```

として正しく変換されることを確認する。

---

### 2.63 Resourceで再計算しないこと

Resource Testでは、`remainingAmount`などがResource内部で再計算されず、Result DTOの保存済み値をそのまま返却することを確認する。

---

### 2.64 副作用なしTest

ASM-004実行前後で、

```text
assessment_histories
objectives
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
net_incomes
```

の業務データが変更されないことを確認する。

---

### 2.65 ASM-002とのIntegration Test

ASM-002で判定結果を保存した後、返却された

```text
assessmentHistoryId
```

を使用してASM-004を実行する。

ASM-002で保存した

```text
判定結果
+
計算根拠
```

と、ASM-004で取得した内容が一致することを確認する。

---

### 2.66 ASM-003とのIntegration Test

ASM-003で判定履歴一覧を取得し、一覧に含まれる

```text
assessmentHistoryId
```

をASM-004へ渡した場合に、対応する詳細を取得できることを確認する。

---

### 2.67 ASM-001とは履歴連携しない

ASM-001はプレビュー専用であり、判定履歴を保存しない。

そのため、ASM-001実行だけではASM-004で取得可能な新しい`assessmentHistoryId`が生成されないことを確認する。

---

### 2.68 実装上の責務分離

ASM-004では、最終的に以下の責務分離を維持する。

```text
Middleware
    → 利用者コンテキスト

Action
    → HTTP入力とUseCaseの橋渡し

UseCase
    → アプリケーション処理

Query
    → 判定履歴・関連情報取得

Result DTO
    → 取得結果表現

API Resource
    → API形式への変換

Responder
    → HTTPレスポンス生成
```

ASM-004では、

```text
保存済み判定履歴を
利用者境界を守って取得し、
判定時点の結果と計算根拠を
変更せず返却する
```

ことへLaravel実装の責務を限定する。

---

## 3. 関連ドキュメント

- [ASM-004 API詳細設計](../../../api/details/assessments/asm-004-detail.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [目的達成判定 Laravelアーキテクチャ設計](./README.md)
- [ASM-004 Reactアーキテクチャ設計](../../../architecture/react/assessments/asm-004-detail.md)
- [ASM-004 テスト設計](../../../tests/assessments/asm-004-detail.md)
- [目的達成判定 テスト設計](../../../tests/assessments/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
