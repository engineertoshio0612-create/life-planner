##  ASM-002 目的達成判定履歴一覧取得

### 1 概要

本ドキュメントでは、
ASM-002 目的達成判定履歴一覧取得APIを
Laravelで実装する際の
アーキテクチャおよび責務分離方針を定義する。

ASM-002では、
操作対象利用者の指定された目的について、
`assessment_histories`に保存されている
目的達成判定履歴を
新しい履歴から順に
ページネーションして取得する。

本APIは、
保存済みの目的達成判定履歴を
参照するための読み取り専用APIであり、
目的達成判定そのものは実行しない。

また、
現在の目的、
資産状況、
手取り収入などを使用して
保存済みの判定結果を再計算しない。

Laravel実装では、
HTTPリクエストの受付から
目的の利用者境界確認、
判定履歴の検索、
並び順、
ページネーション、
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
    └─ AssessmentHistoryQuery
    ↓
API Resource Collection
    ↓
Responder
```

Actionは、
Requestから取得した
`objectiveId`、
`page`、
`perPage`と
利用者コンテキストをUseCaseへ渡し、
取得結果をResponderへ引き渡す
HTTP層の調整役に限定する。

Requestは、
`objectiveId`および
ページネーションパラメータの
形式検証を担当する。

UseCaseは、
操作対象利用者に属する目的の確認から
判定履歴一覧の取得までの
ASM-002固有の業務フローを制御する。

目的の取得および利用者境界の確認は
`ObjectiveQuery`へ委譲し、
判定履歴の絞り込み、
並び順、
ページネーションは
`AssessmentHistoryQuery`へ集約する。

利用者境界については、

```text
UserContext.userId
    ↓
ObjectiveQuery
    ↓
操作対象利用者に属する目的を取得
    ↓
利用者境界確認済みobjectiveId
    ↓
AssessmentHistoryQuery
    ↓
assessment_histories取得
```

という流れで保証する。

他利用者の目的、
論理削除済みの目的、
存在しない目的は
いずれも取得対象とせず、
`OBJECTIVE_NOT_FOUND`として扱う。

一方、
目的が存在していて
判定履歴が0件の場合はエラーとせず、
空の一覧を正常結果として返却する。

判定履歴一覧は、
Phase1では原則として
`assessment_histories.id DESC`によって
新しい履歴から順に取得する。

ページネーションについては、
Laravelの`paginate()`を利用し、
デフォルト件数や最大件数などは
API共通方針の設定を使用する。

APIレスポンスへの変換は
API Resource Collection、
Envelopeおよびページネーション情報を含む
HTTPレスポンスの生成は
Responderへ責務を分離する。

これによりASM-002では、

```text
HTTP制御
入力値検証
業務フロー
利用者境界
データ取得
並び順
ページネーション
API表現
HTTPレスポンス生成
```

の責務を明確に分離する。

また、
本APIは参照専用であるため、
明示的なトランザクションや
`lockForUpdate()`などの行ロックは使用せず、
目的達成判定履歴を含む
業務データの登録・更新・削除を行わない。

ASM-001によって
新しい目的達成判定履歴が登録された場合は、
次回のASM-002実行時に
データベースの最新状態から取得できる構成とする。

---

### 2 Laravel実装方針

ASM-002では、
Action、
Request、
UseCase、
Query、
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
    ├─ ObjectiveQuery
    └─ AssessmentHistoryQuery
    ↓
API Resource Collection
    ↓
Responder
```
目的達成判定履歴一覧の取得条件、利用者境界、並び順、ページネーションは、UseCaseおよびQueryへ集約する。

Actionへ検索条件や業務ロジックを直接記述しない。

---

#### 2.1 Action

HTTPリクエストを受け付け、目的ID、ページネーション条件、利用者コンテキストを取得する。

一覧取得UseCaseを呼び出し、取得結果をResponderへ渡す。

概念例：

```php
final class ListAssessmentHistoriesAction
{
    public function __invoke(
        ListAssessmentHistoriesRequest $request,
        ListAssessmentHistoriesUseCase $useCase,
        AssessmentHistoryListResponder $responder,
        string $objectiveId,
    ): JsonResponse {
        $result = $useCase->execute(
            userId: $request->userId(),
            objectiveId: (int) $objectiveId,
            page: $request->integer(
                'page',
                1,
            ),
            perPage: $request->integer(
                'perPage',
                config(
                    'api.pagination.per_page',
                ),
            ),
        );

        return $responder->ok(
            $result,
        );
    }
}
```

Actionでは、以下を行わない。

* `objectiveId`の形式検証
* `page`の形式検証
* `perPage`の形式検証
* 目的の検索
* 利用者境界の判定
* 目的達成判定履歴の検索
* 並び順制御
* ページネーション処理
* 判定結果の再計算
* APIレスポンス形式への変換

---

#### 2.2 Request

パスパラメータの`objectiveId`と、クエリパラメータの`page`、`perPage`を検証する。

本APIでは、リクエストボディを使用しない。

`objectiveId`は、`prepareForValidation()`でバリデーション対象へ追加する。

概念例：

```php
final class ListAssessmentHistoriesRequest
    extends FormRequest
{
    protected function prepareForValidation(): void
    {
        $this->merge([
            'objectiveId'
                => $this->route(
                    'objectiveId',
                ),
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

            'page' => [
                'sometimes',
                'integer',
                'min:1',
            ],

            'perPage' => [
                'sometimes',
                'integer',
                'min:1',
                'max:' . config(
                    'api.pagination.max_per_page',
                ),
            ],
        ];
    }
}
```

`X-User-Id`の検証および操作対象利用者コンテキストの生成は、API共通Middlewareで行う。

Requestでは、以下を行わない。

* 目的の存在確認
* 目的が操作対象利用者に属するかの判定
* 目的達成判定履歴の存在確認
* 判定履歴の検索
* 並び順制御
* ページネーション実行

これらは、業務処理としてUseCaseおよびQueryで扱う。

---

#### 2.3 UseCase

目的達成判定履歴一覧取得のユースケース処理を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `objectiveId`を受け取る
3. `page`および`perPage`を受け取る
4. 操作対象利用者に属する目的を取得する
5. 目的が存在しない場合は業務例外を送出する
6. 指定された目的に紐づく目的達成判定履歴を取得する
7. 新しい履歴から順に並べる
8. ページネーションを適用する
9. 取得結果を返却する

概念的な処理は、以下とする。

```text
操作対象利用者
    ↓
目的取得
    ↓
目的なし
    → OBJECTIVE_NOT_FOUND

目的あり
    ↓
目的達成判定履歴取得
    ↓
新しい順
    ↓
ページネーション
    ↓
一覧結果返却
```

目的達成判定履歴が0件の場合は、例外を送出しない。

空のページネーション結果を正常結果として返却する。

UseCaseでは、目的達成判定そのものを実行しない。

また、以下のテーブルを使用して過去の判定結果を再計算しない。

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
final class ObjectiveQuery
{
    public function findForUser(
        int $userId,
        int $objectiveId,
    ): ?Objective {
        return Objective::query()
            ->whereKey(
                $objectiveId,
            )
            ->where(
                'user_id',
                $userId,
            )
            ->first();
    }
}
```

論理削除済み目的を通常の取得対象としない。

LaravelのSoftDeletesを使用している場合は、通常のQueryによって`deleted_at IS NULL`が適用されることを前提とする。

他の利用者に属する目的が存在していても、取得結果は`null`とする。

これにより、他利用者の目的の存在をレスポンスから推測できないようにする。

---

#### 2.5 目的不存在

ObjectiveQueryの結果が`null`の場合は、

```text
OBJECTIVE_NOT_FOUND
```

へ変換するための業務例外を送出する。

概念例：

```php
$objective =
    $this->objectiveQuery
        ->findForUser(
            userId: $userId,
            objectiveId: $objectiveId,
        );

if ($objective === null) {
    throw new ObjectiveNotFoundException();
}
```

以下を同じ扱いとする。

* 目的が存在しない
* 他利用者に属している
* 論理削除済みである

---

#### 2.6 AssessmentHistoryQuery

目的達成判定履歴一覧の取得は、専用Queryクラスで行う。

概念例：

```php
final class AssessmentHistoryQuery
{
    public function paginateByObjective(
        int $objectiveId,
        int $page,
        int $perPage,
    ): LengthAwarePaginator {
        return AssessmentHistory::query()
            ->where(
                'objective_id',
                $objectiveId,
            )
            ->orderByDesc(
                'id',
            )
            ->paginate(
                perPage: $perPage,
                page: $page,
            );
    }
}
```

Queryでは、指定された目的に紐づく判定履歴のみを取得する。

以下を行わない。

* 利用者の存在確認
* 目的の利用者境界判定
* 目的達成判定
* 判定結果の再計算
* 目的達成判定履歴の登録
* 目的達成判定履歴の更新
* 目的達成判定履歴の削除
* HTTPレスポンス生成

---

#### 2.7 利用者境界

目的達成判定履歴の利用者境界は、ObjectiveQueryによって取得済みの目的を経由して保証する。

```text
操作対象利用者ID
    ↓
objectives.user_id

objectiveId
    ↓
objectives.id
    ↓
assessment_histories.objective_id
```

UseCaseでは、利用者境界確認済みの`$objective->id`をAssessmentHistoryQueryへ渡す。

概念例：

```php
return $this
    ->assessmentHistoryQuery
    ->paginateByObjective(
        objectiveId: $objective->id,
        page: $page,
        perPage: $perPage,
    );
```

利用者境界を確認する前に、`objectiveId`だけを使用して判定履歴を検索しない。

---

#### 2.8 並び順

Phase1では、目的達成判定履歴を新しい順で返却する。

概念的には、

```text
assessment_histories.id DESC
```

とする。

Laravelでは、以下のように実装する。

```php
->orderByDesc(
    'id',
)
```

並び順は、AssessmentHistoryQueryへ集約する。

Action、UseCase、Resource、Responderで取得後に並べ替えない。

---

#### 2.9 created_atを使用する場合

判定実行日時を`created_at`によって厳密に表現する仕様とする場合は、

```php
->orderByDesc(
    'created_at',
)
->orderByDesc(
    'id',
)
```

としてよい。

同一時刻の履歴が存在しても順序が不定にならないよう、`id`を第二ソート条件として使用する。

正式な並び順は、API仕様とテーブル定義で統一する。

---

#### 2.10 ページネーション

Laravelの`paginate()`を使用する。

概念例：

```php
$paginator =
    AssessmentHistory::query()
        ->where(
            'objective_id',
            $objectiveId,
        )
        ->orderByDesc(
            'id',
        )
        ->paginate(
            perPage: $perPage,
            page: $page,
        );
```

以下は、API共通方針の値を使用する。

* デフォルトページ
* デフォルト`perPage`
* 最大`perPage`

ASM-002だけで独自のページネーション設定を持たない。

---

#### 2.11 pageの扱い

`page`が未指定の場合は、1ページ目として扱う。

```php
$page =
    $request->integer(
        'page',
        1,
    );
```

形式および1以上であることは、Requestで検証する。

指定されたページにデータが存在しない場合も、例外を送出しない。

```json
{
  "data": []
}
```

の正常結果として扱う。

---

#### 2.12 perPageの扱い

`perPage`が未指定の場合は、API共通設定値を使用する。

概念例：

```php
$perPage =
    $request->integer(
        'perPage',
        config(
            'api.pagination.per_page',
        ),
    );
```

最大件数も、共通設定値から取得する。

以下のようなマジックナンバーをASM-002内へ直接記述しない。

```php
'max:100'
```

---

#### 2.13 履歴0件

指定された目的に目的達成判定履歴が存在しない場合も、正常結果とする。

```text
目的あり
+
assessment_histories = 0件
    ↓
200 OK
data = []
```

以下のような一覧0件専用例外は定義しない。

```text
AssessmentHistoryNotFoundException
```

一覧APIでは、0件も正常な検索結果として扱う。

---

#### 2.14 判定結果による絞り込みを行わない

Phase1では、判定結果によるフィルタリングを行わない。

そのため、AssessmentHistoryQueryへ

```php
->where(
    'result',
    AssessmentResultType::Achievable,
)
```

などの条件を追加しない。

以下をすべて取得対象とする。

```text
achievable
difficult
unassessable
```

---

#### 2.15 過去履歴を再計算しない

`assessment_histories.result`に保存されている判定結果をそのまま使用する。

一覧取得時に、

* 現在の目的
* 現在の資産状況
* 現在の手取り収入

を使用して判定結果を再計算しない。

以下のような処理は行わない。

```text
assessment_histories取得
    ↓
現在の資産情報取得
    ↓
AssessmentCalculator
    ↓
result再計算
```

ASM-002は、保存済み履歴の参照に責務を限定する。

---

#### 2.16 取得カラム

一覧取得では、レスポンス生成と並び順に必要なカラムを中心に取得する。

概念例：

```php
return AssessmentHistory::query()
    ->select([
        'id',
        'objective_id',
        'result',
    ])
    ->where(
        'objective_id',
        $objectiveId,
    )
    ->orderByDesc(
        'id',
    )
    ->paginate(
        perPage: $perPage,
        page: $page,
    );
```

一覧レスポンスで使用しない判定内部情報を不要に取得しない。

---

#### 2.17 N+1問題

ASM-002の一覧レスポンス項目は、`assessment_histories`だけで生成できる。

そのため、一覧行ごとに`objective`を遅延ロードしない。

以下のような処理は行わない。

```php
foreach (
    $histories
    as $history
) {
    $history->objective->id;
}
```

`objectiveId`は、

```text
assessment_histories.objective_id
```

から直接取得する。

これにより、不要なN+1クエリを防止する。

---

#### 2.18 AssessmentHistory Model

`AssessmentHistory` Modelは、`assessment_histories`へ対応する。

目的との関連は、必要に応じてRelationとして定義する。

概念例：

```php
final class AssessmentHistory
    extends Model
{
    public function objective(): BelongsTo
    {
        return $this->belongsTo(
            Objective::class,
        );
    }
}
```

ただし、ASM-002の一覧取得処理そのものをModelへ記述しない。

---

#### 2.19 resultのCast

`assessment_histories.result`をEnumとして管理する場合は、Eloquent Castを使用する。

概念例：

```php
protected function casts(): array
{
    return [
        'result'
            => AssessmentResultType::class,
    ];
}
```

これにより、Laravel内部では判定結果をEnumとして扱える。

API Resourceでは、Enumの`value`をレスポンスへ変換する。

---

#### 2.20 API Resource

データベースカラムを直接返却せず、API Resourceを使用してレスポンス形式へ変換する。

1件分の概念例：

```php
final class AssessmentHistoryResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'objectiveId'
                => (string) $this->objective_id,

            'result'
                => $this->result
                    instanceof AssessmentResultType
                    ? $this->result->value
                    : $this->result,
        ];
    }
}
```

IDは、API共通方針に従ってstringへ変換する。

snake_caseのデータベースカラム名をそのまま返却しない。

---

#### 2.21 Resource Collection

一覧レスポンスでは、`AssessmentHistoryResource`をCollectionとして利用する。

概念例：

```php
AssessmentHistoryResource::collection(
    $paginator->items(),
);
```

Laravel標準のResource Collection形式をそのまま公開するかどうかは、API共通Envelope仕様に従う。

独自の`meta`形式を採用している場合は、Responderで共通形式へ変換する。

---

#### 2.22 Responder

Responderは、Paginatorを受け取り、API共通方針に従った一覧レスポンスへ変換する。

概念例：

```php
final class AssessmentHistoryListResponder
{
    public function ok(
        LengthAwarePaginator $paginator,
    ): JsonResponse {
        return response()->json(
            [
                'data' =>
                    AssessmentHistoryResource::collection(
                        $paginator->items(),
                    ),

                'meta' => [
                    'currentPage'
                        => $paginator
                            ->currentPage(),

                    'perPage'
                        => $paginator
                            ->perPage(),

                    'total'
                        => $paginator
                            ->total(),

                    'lastPage'
                        => $paginator
                            ->lastPage(),
                ],
            ],
            Response::HTTP_OK,
        );
    }
}
```

実際のEnvelope、`meta`構造、`requestId`の付与方法は、API共通方針に従う。

---

#### 2.23 Responderの責務

Responderでは、以下を行わない。

* データベース検索
* 目的の存在確認
* 利用者境界の判定
* 判定履歴の絞り込み
* 判定結果によるフィルタリング
* 並び順制御
* ページネーション検索
* 過去の判定結果の再計算

Responderは、取得済みの一覧結果をHTTPレスポンスへ変換することに責務を限定する。

---

#### 2.24 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* JSONレスポンス共通処理
* 共通例外処理
* ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

#### 2.25 トランザクション

ASM-002は、読み取り専用APIである。

そのため、明示的な`DB::transaction()`を使用しない。

以下のような実装は行わない。

```php
DB::transaction(
    function () {
        // 一覧取得のみ
    },
);
```

参照処理だけのために不要なトランザクション境界を追加しない。

---

#### 2.26 ロック

ASM-002では、行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

ASM-002の一覧取得によって、ASM-001の目的達成判定履歴登録をブロックしてはならない。

---

#### 2.27 同時登録との扱い

ASM-002の実行中に、ASM-001によって新しい目的達成判定履歴が登録される可能性がある。

Phase1では、複数ページにまたがる完全なスナップショット整合性までは保証しない。

```text
1ページ目取得
    ↓
ASM-001で履歴追加
    ↓
2ページ目取得
```

のような場合は、offsetベースページネーションの一般的な挙動として許容する。

将来的に履歴件数が増加し、ページ間整合性が重要となった場合は、カーソルベースページネーションを別途検討する。

---

#### 2.28 キャッシュ

Phase1では、ASM-002専用のサーバー側アプリケーションキャッシュを使用しない。

ASM-001実行後に新しい判定履歴を一覧へ反映できるよう、データベースから最新状態を取得する。

将来的に履歴件数やアクセス量が増加した場合は、別途キャッシュ戦略を検討する。

---

#### 2.29 例外

目的が存在しない場合は、専用業務例外を使用する。

概念例：

```php
throw new ObjectiveNotFoundException();
```

共通Exception Handlerで、

```text
ObjectiveNotFoundException
    ↓
OBJECTIVE_NOT_FOUND
    ↓
404 Not Found
```

へ変換する。

履歴0件については、例外を使用しない。

---

#### 2.30 想定外例外

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

#### 2.31 ログ

ASM-002では、必要に応じて以下の情報をログコンテキストへ設定する。

```text
requestId
userId
objectiveId
page
perPage
```

取得した目的達成判定履歴の内容そのものを大量にログへ出力しない。

---

#### 2.32 テスト実装方針

Laravel側では、Feature Testを中心としてASM-002のAPI契約および一覧取得処理を確認する。

Feature Testでは、主に以下を確認する。

* `200 OK`
* `400 Bad Request`
* `404 Not Found`
* `422 Unprocessable Entity`
* `500 Internal Server Error`
* 利用者境界
* 指定目的に紐づく履歴のみ取得されること
* 他の目的の履歴が混入しないこと
* 他利用者の履歴が混入しないこと
* 履歴0件が正常結果となること
* `achievable`が取得対象となること
* `difficult`が取得対象となること
* `unassessable`が取得対象となること
* 新しい履歴から返却されること
* `page`のデフォルト値
* `perPage`のデフォルト値
* ページネーション
* ページ範囲外で空配列となること
* API ResourceによるcamelCase変換
* IDがstringとして返却されること
* 読み取り専用であること
* 目的達成判定が再実行されないこと
* 過去の判定結果が再計算されないこと
* データベースが更新されないこと

AssessmentHistoryQueryについては、必要に応じてDatabase Testを行う。

主に以下を確認する。

```text
objective_idによる絞り込み
+
id DESC
+
ページネーション
```

ObjectiveQueryについては、以下を確認する。

```text
同一利用者の目的
    → 取得できる

他利用者の目的
    → null

論理削除済み目的
    → null
```

UseCaseについては、目的不存在の場合に

```text
ObjectiveNotFoundException
```

となること、履歴0件の場合は正常なPaginatorが返却されることを確認する。


---

#### 27.1 objectiveIdの扱い

`objectiveId`は、
API共通方針に従って
stringとして扱う。

```ts
const objectiveId: string = '5';
```

フロントエンド側で
numberへ変換して
業務計算には使用しない。

URL生成時も、
stringのまま使用する。

---

#### 27.2 pageの扱い

`page`は、
1以上のintegerとして扱う。

```ts
const page = 1;
```

未指定の場合は、
バックエンド側で
1ページ目として扱われる。

フロントエンド側では、
ページネーションUIの
現在ページとして保持してよい。

```ts
const [page, setPage] =
  useState(1);
```

---

#### 27.3 perPageの扱い

`perPage`は、
1ページあたりの取得件数として扱う。

```ts
const perPage = 20;
```

未指定の場合は、
API共通方針で定めた
デフォルト件数が使用される。

フロントエンド側で
任意の大きな値を設定せず、
API共通方針で定めた
最大件数以内で使用する。

---

#### 27.4 Queryとして扱う

ASM-002は、
読み取り専用のGET APIであるため、
React Query等では
Queryとして扱う。

概念例：

```ts
export const fetchAssessmentHistories =
  async ({
    objectiveId,
    page,
    perPage,
  }: ListAssessmentHistoriesParams &
    ListAssessmentHistoriesQuery) => {
    const response =
      await apiClient.get<ListAssessmentHistoriesResponse>(
        `/api/v1/objectives/${objectiveId}/assessments`,
        {
          params: {
            page,
            perPage,
          },
        },
      );

    return response.data;
  };
```

---

#### 27.5 Query Key

Query Keyには、
少なくとも
`objectiveId`、
`page`、
`perPage`
を含める。

概念例：

```ts
export const assessmentHistoryKeys = {
  all: [
    'assessmentHistories',
  ] as const,

  list: (
    objectiveId: string,
    page: number,
    perPage: number,
  ) =>
    [
      ...assessmentHistoryKeys.all,
      objectiveId,
      page,
      perPage,
    ] as const,
};
```

これにより、
目的ごと、
ページごとに
キャッシュを分離する。

---

#### 27.6 Hook実装例

React Queryを使用する場合の
概念例は、
以下とする。

```ts
export const useAssessmentHistories =
  (
    objectiveId: string,
    page: number,
    perPage: number,
  ) => {
    return useQuery({
      queryKey:
        assessmentHistoryKeys.list(
          objectiveId,
          page,
          perPage,
        ),

      queryFn:
        () =>
          fetchAssessmentHistories({
            objectiveId,
            page,
            perPage,
          }),

      enabled:
        objectiveId.length > 0,
    });
  };
```

実際のAPI Clientや
React Queryの利用方針は、
フロントエンド共通設計に従う。

---

#### 27.7 履歴一覧表示

取得した`data`を使用して、
目的達成判定履歴一覧を表示する。

概念例：

```tsx
{data.data.map((history) => (
  <AssessmentHistoryRow
    key={history.id}
    history={history}
  />
))}
```

各履歴には、
一覧表示に必要な

```text
id
objectiveId
result
```

のみを使用する。

---

#### 27.8 判定結果の表示

`result`に応じて、
表示内容を切り替える。

概念例：

```ts
const resultLabelMap: Record<
  AssessmentResult,
  string
> = {
  achievable: '達成可能',
  difficult: '達成困難',
  unassessable: '判定不可',
};
```

```tsx
<span>
  {resultLabelMap[history.result]}
</span>
```

バックエンドから返却された
`result`をもとに表示する。

フロントエンドで
現在の資産状況や
手取り収入を使用して
判定結果を再計算しない。

---

#### 27.9 判定不可の表示

`result = 'unassessable'`の履歴も、
通常の履歴として一覧表示する。

```ts
if (
  history.result
  === 'unassessable'
) {
  // 判定不可として表示
}
```

一覧から除外しない。

また、
APIエラーとして扱わない。

---

#### 27.10 履歴0件

`data`が空配列の場合は、
エラー表示を行わない。

```ts
if (
  response.data.length === 0
) {
  // 履歴なし表示
}
```

表示例：

```text
目的達成判定履歴はありません。
```

ASM-001を実行できる画面であれば、
必要に応じて
目的達成判定実行への
導線を表示してよい。

---

#### 27.11 ローディング表示

一覧取得中は、
ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return <Loading />;
}
```

ページ切り替え時に
既存一覧を維持するかどうかは、
フロントエンド共通設計に従う。

---

#### 27.12 ページネーションUI

レスポンスの`meta`を使用して、
ページネーションUIを構築する。

概念例：

```tsx
<Pagination
  currentPage={
    data.meta.currentPage
  }
  lastPage={
    data.meta.lastPage
  }
  onChange={setPage}
/>
```

総件数表示が必要な場合は、
`meta.total`を使用する。

```tsx
<span>
  全{data.meta.total}件
</span>
```

ページ数を
フロントエンド側で
独自計算する必要はない。

---

#### 27.13 ページ切り替え

ページ変更時は、
`page`を更新し、
新しいQuery Keyで
ASM-002を再取得する。

```ts
setPage(nextPage);
```

ページ変更によって、
目的達成判定が
実行されることはない。

---

#### 27.14 perPage変更

1ページあたりの表示件数を
利用者が変更できるUIを
採用する場合は、
`perPage`を更新する。

概念例：

```ts
const handlePerPageChange =
  (nextPerPage: number) => {
    setPerPage(nextPerPage);
    setPage(1);
  };
```

`perPage`変更時は、
1ページ目へ戻すことを推奨する。

ただし、
Phase1で
表示件数変更UIを提供しない場合は、
固定値または
APIデフォルトを使用してよい。

---

#### 27.15 ページ範囲外

ページ切り替え中に
履歴件数が変化し、
指定ページが
範囲外となった場合でも、
APIから

```text
200 OK
data = []
```

が返却される可能性がある。

この場合は、
空一覧として扱う。

必要に応じて、
`meta.lastPage`を確認し、
有効な最終ページへ
戻すことを検討できる。

---

#### 27.16 ASM-001との連携

ASM-001 目的達成判定実行APIが
成功した場合は、
ASM-002の一覧キャッシュを
invalidateする。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey: [
    'assessmentHistories',
    objectiveId,
  ],
});
```

これにより、
新しく作成された
目的達成判定履歴を
一覧へ反映する。

---

#### 27.17 最新履歴の扱い

ASM-002は
新しい履歴から返却されるため、
通常は`data[0]`が
そのページ内の
最新履歴となる。

1ページ目であれば、
概念的に

```ts
const latestAssessment =
  data.data[0] ?? null;
```

として
最新の目的達成判定履歴を
表示に利用できる。

ただし、
最新履歴専用のAPI契約として
`data[0]`へ過度に依存せず、
一覧結果の先頭として扱う。

---

#### 27.18 ASM-003への遷移

一覧の各履歴から
ASM-003 目的達成判定履歴詳細取得APIを
利用する詳細画面へ遷移できる。

概念例：

```tsx
<Link
  to={
    `/objectives/${objectiveId}/assessments/${history.id}`
  }
>
  詳細
</Link>
```

詳細画面では、
一覧データだけではなく、
ASM-003によって
対象履歴を再取得する。

---

#### 27.19 OBJECTIVE_NOT_FOUND

`OBJECTIVE_NOT_FOUND`
が返却された場合は、
対象目的を
現在参照できない状態として扱う。

例えば、

- 目的が削除された
- URLが古い
- 他利用者の目的IDが指定された

などが考えられる。

理由を推測せず、
目的一覧画面へ戻す。

---

#### 27.20 VALIDATION_ERROR

`objectiveId`、
`page`、
`perPage`が
不正な場合は、
`VALIDATION_ERROR`
が返却される。

`objectiveId`の不正は、
不正なURLまたは
画面状態として扱う。

`page`または
`perPage`の不正は、
通常のUI操作では
発生しないことを前提とする。

必要に応じて、
初期値へ戻して
再取得してよい。

---

#### 27.21 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 不正なURLまたはページネーション状態として扱う |
| `OBJECTIVE_NOT_FOUND` | 目的一覧画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

履歴0件は、
エラー表示の対象としない。

---

#### 27.22 自動リトライ

ASM-002は
読み取り専用GET APIであり、
冪等である。

そのため、
一時的な通信エラーに対して
React Query等の
標準的な自動リトライを
利用してよい。

ただし、

```text
400
404
422
```

など、
再送しても解消しない
クライアント起因エラーについては、
不要なリトライを行わないよう
共通API Clientまたは
Query設定で制御する。

---

#### 27.23 キャッシュ

ASM-002は
React Query等による
クライアントキャッシュの対象としてよい。

ただし、
ASM-001成功後は
履歴一覧が変更されるため、
対象目的の
ASM-002キャッシュを
invalidateする。

Phase1では、
サーバー側の
アプリケーションキャッシュは
前提としない。

---

#### 27.24 現在データから再計算しない

フロントエンドでは、
目的達成判定履歴の表示時に、

- 現在の目的
- 現在の資産額
- 現在の手取り収入

を使用して
`result`を再計算しない。

例えば、
以下のような処理は行わない。

```ts
const recalculatedResult =
  calculateAssessment(
    currentObjective,
    currentAssets,
    currentNetIncome,
  );
```

一覧には、
ASM-001実行時に保存された
`assessment_histories.result`を
そのまま表示する。

---

### 3 関連ドキュメント

- [ASM-002 API詳細設計](../../../api/details/assessments/asm-002-create.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [目的達成判定 Laravelアーキテクチャ設計](./README.md)
- [ASM-002 テスト設計](../../../tests/assessments/asm-002-create.md)
- [目的達成判定 テスト設計](../../../tests/assessments/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
