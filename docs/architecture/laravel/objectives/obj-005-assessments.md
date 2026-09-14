# OBJ-005 目的無効化

## 1. 概要

操作対象となる利用者に登録された
有効な目的を無効化する。

本APIでは、
目的の利用状態を有効から無効へ変更する。

```text
enabled = true
    ↓
OBJ-005
    ↓
enabled = false
```

無効化によって変更するのは
目的の利用状態のみとし、
目的名、実施予定年月、必要支出額および
メモなどの目的情報は変更しない。

無効化された目的は、
以降の新しい目的達成判定の対象外とする。

一方で、
無効化以前に保存された目的達成判定履歴は
過去の記録として保持し、
更新、削除および再計算は行わない。

目的の無効化は論理削除とは異なる
業務上の状態変更として扱うため、
`deleted_at`は更新しない。

そのため、無効化後も
目的一覧および目的詳細から
過去の目的情報として参照できる。

すでに無効化されている目的に対して
再度無効化を実行した場合は、
目的の状態を変更せず業務エラーとして扱う。

本APIでは目的の無効化のみを行い、
目的内容の更新、論理削除および
目的達成判定は行わない。


---

## 2. Laravel実装方針

OBJ-005では、Action、UseCase、Query、Repository、DTO、API Resource、Responderを分離して実装する。

概念的な構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ ObjectiveQuery
    └─ ObjectiveRepository
    ↓
Disable Result DTO
    ↓
API Resource
    ↓
Responder
```

OBJ-005ではRequest Bodyを使用しないため、OBJ-005専用のFormRequestは原則として作成しない。

また、目的の無効化は、

```text
enabled = true
    ↓
enabled = false
```

という単一方向の状態変更として扱い、最終更新時には、

```text
enabled = true
```

をUPDATE条件へ含める。

---

### 2.1 Route

OBJ-005は、以下のルートとして定義する。

概念例：

```php
Route::patch(
    '/api/v1/objectives/{objectiveId}/disabled',
    DisableObjectiveAction::class,
);
```

OBJ-004とは、URLによって責務を分離する。

```text
PATCH
/api/v1/objectives/{objectiveId}
    → OBJ-004
      目的更新

PATCH
/api/v1/objectives/{objectiveId}/disabled
    → OBJ-005
      目的無効化
```

---

### 2.2 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* 共通例外処理
* ログコンテキスト設定

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

OBJ-005では、Request Bodyを使用しないため、専用FormRequestは原則として作成しない。

以下のような空のFormRequestは作成しない。

```php
final class DisableObjectiveRequest
    extends FormRequest
{
}
```

不要なクラスを追加しない。

---

### 2.4 objectiveIdの形式検証

`objectiveId`は、ルート制約またはAPI共通のパスパラメータ検証方式で検証する。

概念例：

```php
Route::patch(
    '/api/v1/objectives/{objectiveId}/disabled',
    DisableObjectiveAction::class,
)
    ->where(
        'objectiveId',
        '[1-9][0-9]*',
    );
```

正の整数形式だけを許可する。

例えば、以下を不正とする。

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
INVALID_OBJECTIVE_ID
```

へ変換する。

---

### 2.5 Action

Actionは、`objectiveId`と利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php
final class DisableObjectiveAction
{
    public function __invoke(
        string $objectiveId,
        DisableObjectiveUseCase $useCase,
        DisableObjectiveResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                objectiveId:
                    (int) $objectiveId,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

`objectiveId`は、形式検証済みであることを前提に整数へ変換する。

---

### 2.6 Actionで行わないこと

Actionでは、以下を行わない。

* 目的検索
* 利用者境界判定
* 論理削除判定
* `enabled`判定
* UPDATE
* 更新件数判定
* エラーコード判定
* DBアクセス
* レスポンス配列生成

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 2.7 UseCase

OBJ-005の業務処理全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `objectiveId`を受け取る
3. 対象目的を取得する
4. 対象不存在の場合は業務例外を送出する
5. 現在の`enabled`を確認する
6. すでに無効化済みの場合は業務例外を送出する
7. `enabled = true`を条件として無効化更新する
8. 更新件数を確認する
9. 更新件数0件の場合は最新状態を再確認する
10. 更新後の目的を取得またはResult DTOへ変換する
11. Result DTOを返却する

---

### 2.8 UseCaseの概念フロー

概念的には、以下とする。

```text
userId
+
objectiveId
    ↓
ObjectiveQuery
    ↓
対象目的取得
    ↓
OBJECTIVE_NOT_FOUND？
    ↓ No
enabled確認
    ↓
false？
    ↓ Yes
OBJECTIVE_DISABLED
    ↓ No
ObjectiveRepository
    ↓
enabled = true
を条件としてUPDATE
    ↓
更新件数
    ├─ 1
    │   ↓
    │ 更新成功
    │
    └─ 0
        ↓
      最新状態再確認
        ├─ 不存在
        │   ↓
        │ OBJECTIVE_NOT_FOUND
        │
        └─ enabled = false
            ↓
          OBJECTIVE_DISABLED
```

---

### 2.9 ObjectiveQuery

対象目的の取得は、専用Queryへ委譲する。

概念例：

```php
$objective =
    $this->objectiveQuery
        ->findActiveRecordByIdAndUser(
            objectiveId:
                $objectiveId,

            userId:
                $userId,
        );
```

ここでいう`ActiveRecord`は、`enabled = true`という意味ではなく、

```text
deleted_at IS NULL
```

の通常参照可能な目的を意味する。

命名が紛らわしい場合は、

```text
findNotDeletedByIdAndUser
```

など、プロジェクト内で意味が明確になる名称を使用する。

---

### 2.10 対象目的の取得条件

概念的な取得条件は、以下とする。

```text
objectives.id
    = objectiveId

AND

objectives.user_id
    = userId

AND

objectives.deleted_at
    IS NULL
```

`enabled`は取得条件に含めず、取得後に現在状態として確認してもよい。

これにより、

```text
目的不存在
```

と

```text
無効化済み目的
```

を区別できる。

---

### 2.11 利用者境界をQueryへ含める

以下のようなIDだけでの取得は基本としない。

```php
$objective =
    Objective::find(
        $objectiveId,
    );
```

対象取得時点から、

```text
objectiveId
+
userId
+
deleted_at IS NULL
```

を条件へ含める。

これにより、他利用者の目的を不要に取得しない。

---

### 2.12 SoftDeletes

`Objective` ModelでLaravelのSoftDeletesを使用する場合は、

```php
use SoftDeletes;
```

を利用する。

OBJ-005では論理削除済み目的を対象としないため、

```php
withTrashed()
```

を使用しない。

---

### 2.13 OBJECTIVE_NOT_FOUND

対象目的を取得できない場合は、

```php
throw new
    ObjectiveNotFoundException();
```

とする。

以下を同じ例外へ集約する。

* 目的不存在
* 他利用者所属
* 論理削除済み

最終的に、

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

へ変換する。

---

### 2.14 enabled状態確認

対象目的を取得後、現在の

```text
enabled
```

を確認する。

概念例：

```php
if (! $objective->enabled) {
    throw new
        ObjectiveDisabledException();
}
```

すでに無効化済みの場合は、Repositoryへ更新依頼を行わない。

---

### 2.15 OBJECTIVE_DISABLED

対象目的が、

```text
enabled = false
```

の場合は、

```php
throw new
    ObjectiveDisabledException();
```

とする。

最終的に、

```text
409 Conflict
OBJECTIVE_DISABLED
```

へ変換する。

---

### 2.16 ObjectiveRepository

目的の状態変更は、Repositoryへ委譲する。

概念的なインターフェースは、以下とする。

```php
interface ObjectiveRepository
{
    public function disable(
        int $objectiveId,
        int $userId,
    ): int;
}
```

戻り値には、更新件数を返してよい。

---

### 2.17 条件付きUPDATE

Repositoryでは、最終UPDATE条件へ、

```text
enabled = true
```

を含める。

概念例：

```php
public function disable(
    int $objectiveId,
    int $userId,
): int {
    return Objective::query()
        ->where(
            'id',
            $objectiveId,
        )
        ->where(
            'user_id',
            $userId,
        )
        ->whereNull(
            'deleted_at',
        )
        ->where(
            'enabled',
            true,
        )
        ->update([
            'enabled' => false,
            'updated_at' => now(),
        ]);
}
```

これにより、SELECT後に別Requestが先に無効化した場合でも、後続Requestによる二重更新を防止する。

---

### 2.18 Eloquent Modelを使用した更新

Eloquent Model経由で更新する場合でも、排他制御上は更新直前の状態を条件へ含めることを優先する。

単純に、

```php
$objective->enabled = false;
$objective->save();
```

だけとすると、SELECTからUPDATEまでの競合を検知しにくい。

そのため、OBJ-005では条件付きUPDATE方式を基本とする。

---

### 2.19 更新件数1件

Repositoryの戻り値が、

```text
1
```

の場合は、目的無効化成功とする。

概念的には、

```text
enabled = true
    ↓
UPDATE
    ↓
1 row affected
    ↓
成功
```

とする。

---

### 2.20 更新件数0件

Repositoryの戻り値が、

```text
0
```

の場合は、無条件に

```text
OBJECTIVE_DISABLED
```

とはしない。

最新状態を再確認する。

理由は、以下の可能性があるためである。

* 目的が削除された
* 利用者境界外になっている
* すでに無効化された

---

### 2.21 更新件数0件時の再取得

概念例：

```php
if ($affectedRows === 0) {
    $latest =
        $this->objectiveQuery
            ->findNotDeletedByIdAndUser(
                objectiveId:
                    $objectiveId,

                userId:
                    $userId,
            );

    if ($latest === null) {
        throw new
            ObjectiveNotFoundException();
    }

    if (! $latest->enabled) {
        throw new
            ObjectiveDisabledException();
    }

    throw new
        RuntimeException(
            'Unexpected objective disable state.',
        );
}
```

---

### 2.22 想定外の更新件数

OBJ-005は単一目的を対象とするため、更新件数が、

```text
2件以上
```

になることは正常ではない。

主キー条件を使用するため通常発生しないが、想定外状態として内部エラーへ変換する。

---

### 2.23 明示的トランザクション

Phase1では、単一テーブル・単一UPDATEのため、

```php
DB::transaction()
```

を必須としない。

ただし、

```text
更新
+
監査履歴INSERT
```

など複数更新が追加された場合は、UseCase全体をトランザクション境界へ変更する。

---

### 2.24 lockForUpdate

OBJ-005では、Phase1では

```php
lockForUpdate()
```

を必須としない。

競合制御は、

```text
enabled = true
```

を条件としたUPDATEによって行う。

---

### 2.25 楽観ロック用versionカラム

Phase1では、

```text
version
lock_version
```

などの専用カラムを追加しない。

OBJ-005の状態遷移は一方向であり、現在状態を更新条件へ含めることで必要な競合制御を行えるためである。

---

### 2.26 Mass Assignmentを避ける

以下のような処理は行わない。

```php
$objective->update(
    $request->all(),
);
```

OBJ-005では、Request Body自体を使用しない。

更新対象は、

```text
enabled
```

だけに限定する。

---

### 2.27 deleted_atを変更しない

Repositoryでは、

```text
deleted_at
```

を変更しない。

OBJ-005はSoftDeleteを実行するAPIではない。

以下のような処理は行わない。

```php
$objective->delete();
```

---

### 2.28 assessment_historiesを更新しない

OBJ-005では、

```text
assessment_histories
```

へのRepository処理を呼び出さない。

目的無効化時に、

```text
判定履歴削除
判定履歴更新
判定再実行
```

を行わない。

---

### 2.29 更新後の目的取得

条件付きUPDATE成功後、成功レスポンス用として更新後の目的情報を取得する。

概念例：

```php
$updatedObjective =
    $this->objectiveQuery
        ->findNotDeletedByIdAndUser(
            objectiveId:
                $objectiveId,

            userId:
                $userId,
        );
```

または、更新前に取得済みのModelへ

```text
enabled = false
```

を反映したResult DTOを生成してもよい。

---

### 2.30 更新後に再取得する方式

DB上の確定状態をレスポンスへ反映することを優先する場合は、

```text
UPDATE
    ↓
SELECT
    ↓
Result DTO
```

としてよい。

ただし、単純な1項目更新であり性能上の必要性も踏まえ、プロジェクト全体の更新API方針に合わせる。

---

### 2.31 Result DTO

無効化後の目的を専用Result DTOとして表現する。

概念例：

```php
final readonly class DisableObjectiveResult
{
    public function __construct(
        public int $id,
        public string $name,
        public string $plannedYearMonth,
        public int $requiredExpense,
        public ?string $memo,
        public bool $enabled,
    ) {
    }
}
```

OBJ-005成功時の`enabled`は`false`となる。

---

### 2.32 DTOへEloquent Modelを保持しない

以下のようなResult DTOは基本としない。

```php
final readonly class DisableObjectiveResult
{
    public function __construct(
        public Objective $objective,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

---

### 2.33 UseCaseの戻り値

概念例：

```php
return new DisableObjectiveResult(
    id:
        $objective->id,

    name:
        $objective->name,

    plannedYearMonth:
        $objective->planned_year_month,

    requiredExpense:
        $objective->required_expense,

    memo:
        $objective->memo,

    enabled:
        false,
);
```

Eloquent ModelをActionへ直接返却しない。

---

### 2.34 API Resource

Result DTOを、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php
final class DisableObjectiveResource
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

            'plannedYearMonth'
                => $this->plannedYearMonth,

            'requiredExpense'
                => $this->requiredExpense,

            'memo'
                => $this->memo,

            'enabled'
                => $this->enabled,
        ];
    }
}
```

---

### 2.35 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

* `user_id`
* `deleted_at`
* `created_at`
* `updated_at`
* DB内部管理情報
* 過去の目的達成判定履歴

API共通方針または目的API全体で日時項目を返す方針がある場合は、そちらを優先する。

---

### 2.36 snake_caseを直接返さない

DBカラム名の、

```text
planned_year_month
required_expense
```

は、APIでは、

```text
plannedYearMonth
requiredExpense
```

へ変換する。

DB構造をそのままAPI契約へ公開しない。

---

### 2.37 Responder

Responderは、Result DTOを`200 OK`レスポンスへ変換する。

概念例：

```php
final class DisableObjectiveResponder
{
    public function ok(
        DisableObjectiveResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    =>
                    new DisableObjectiveResource(
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

### 2.38 Responderで行わないこと

Responderでは、以下を行わない。

* 目的検索
* 利用者境界確認
* 論理削除判定
* `enabled`判定
* UPDATE
* 更新件数判定
* DBアクセス
* 業務例外判定

HTTPレスポンス生成だけに責務を限定する。

---

### 2.39 例外変換

主な例外変換は、以下とする。

| 内部状態              | 独自エラーコード                |
| ----------------- | ----------------------- |
| `X-User-Id`未指定    | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正   | `INVALID_USER_ID`       |
| 利用者不存在            | `USER_NOT_FOUND`        |
| `objectiveId`形式不正 | `INVALID_OBJECTIVE_ID`  |
| 目的不存在             | `OBJECTIVE_NOT_FOUND`   |
| 他利用者の目的           | `OBJECTIVE_NOT_FOUND`   |
| 論理削除済み目的          | `OBJECTIVE_NOT_FOUND`   |
| 無効化済み目的           | `OBJECTIVE_DISABLED`    |
| 想定外例外             | `INTERNAL_SERVER_ERROR` |

---

### 2.40 ObjectiveNotFoundException

対象目的を取得できない場合は、

```php
throw new
    ObjectiveNotFoundException();
```

とする。

最終的に、

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

へ変換する。

---

### 2.41 ObjectiveDisabledException

対象目的がすでに無効化済みの場合は、

```php
throw new
    ObjectiveDisabledException();
```

とする。

最終的に、

```text
409 Conflict
OBJECTIVE_DISABLED
```

へ変換する。

---

### 2.42 想定外例外

想定外の例外は、API共通Exception Handlerで、

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

* SQL
* SQLSTATE
* PostgreSQL内部エラー
* 制約名
* テーブル名
* カラム名
* Laravel内部例外メッセージ
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

---

### 2.43 ログ

OBJ-005では、必要に応じて以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
objectiveId
httpStatus
errorCode
```

`apiId`は、

```text
OBJ-005
```

とする。

---

### 2.44 正常時ログ

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = OBJ-005
objectiveId
previousEnabled = true
newEnabled = false
httpStatus = 200
```

目的名やメモなどを通常ログへ不要に出力しない。

---

### 2.45 無効化済みログ

`OBJECTIVE_DISABLED`が発生した場合は、必要に応じて、

```text
requestId
userId
objectiveId
errorCode = OBJECTIVE_DISABLED
httpStatus = 409
```

を記録する。

二重送信や同時実行の調査に利用できる。

---

### 2.46 更新件数0件ログ

条件付きUPDATEが0件だった場合は、必要に応じて内部ログへ記録する。

概念的には、

```text
objectiveId
affectedRows = 0
```

を記録し、その後の最新状態確認結果も追跡可能にしてよい。

---

### 2.47 キャッシュ

Phase1では、OBJ-005専用のサーバー側キャッシュを使用しない。

React側では、OBJ-005成功後に少なくとも以下をinvalidateする。

```text
OBJ-001
目的一覧

OBJ-003
目的詳細
```

目的達成判定画面が有効目的のQuery Cacheを持つ場合は、必要に応じて関連Queryも無効化する。

---

### 2.48 テスト実装方針

Laravel側では、Feature Testを中心としてOBJ-005のAPI契約を確認する。

また、Query、Repository、UseCase、API Resourceについて必要に応じてUnit TestまたはDatabase Testを行う。

主に以下を確認する。

* `200 OK`
* `400 Bad Request`
* `404 Not Found`
* `409 Conflict`
* `500 Internal Server Error`
* 利用者コンテキスト
* `objectiveId`形式
* 利用者境界
* SoftDeletes
* 有効目的の無効化
* 無効化済み目的
* 条件付きUPDATE
* 同時実行
* 更新対象限定
* レスポンス契約
* 副作用なし

---

### 2.49 ObjectiveQueryのDatabase Test

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

* 存在しないID
* 他利用者の目的
* 論理削除済み目的

無効化済み目的は、`enabled`状態確認のため取得できる構成としてよい。

---

### 2.50 Repository正常系Test

以下の状態を用意する。

```text
id = 1
user_id = 1
enabled = true
deleted_at = NULL
```

`disable()`実行後に、

```text
enabled = false
```

となり、更新件数が、

```text
1
```

となることを確認する。

---

### 2.51 Repository無効化済みTest

以下の状態を用意する。

```text
enabled = false
```

`disable()`を実行しても、更新件数が、

```text
0
```

となることを確認する。

`updated_at`も不要に更新されないことを確認する。

---

### 2.52 Repository利用者境界Test

他利用者の目的に対して`disable()`を実行しても、

```text
affectedRows = 0
```

となり、対象目的が変更されないことを確認する。

---

### 2.53 Repository論理削除Test

論理削除済み目的に対して`disable()`を実行しても、

```text
affectedRows = 0
```

となることを確認する。

---

### 2.54 UseCase正常系Test

QueryとRepositoryを使用して、以下の流れになることを確認する。

```text
目的取得
    ↓
enabled = true確認
    ↓
Repository::disable()
    ↓
affectedRows = 1
    ↓
Result DTO
```

---

### 2.55 UseCase OBJECTIVE_NOT_FOUND Test

ObjectiveQueryが`null`を返した場合は、

```text
ObjectiveNotFoundException
```

となることを確認する。

Repositoryが呼ばれないことも確認する。

---

### 2.56 UseCase OBJECTIVE_DISABLED Test

取得した目的が、

```text
enabled = false
```

の場合は、

```text
ObjectiveDisabledException
```

となることを確認する。

Repositoryが呼ばれないことも確認する。

---

### 2.57 更新件数0件時のUseCase Test

最初の取得では、

```text
enabled = true
```

だったが、Repository更新結果が、

```text
0
```

となるケースを用意する。

その後、最新状態を再取得し、

```text
enabled = false
```

であれば、

```text
ObjectiveDisabledException
```

になることを確認する。

---

### 2.58 同時実行Integration Test

可能であれば、同一目的に対してOBJ-005を並行実行する。

以下を確認する。

* 一方だけが更新成功する
* 最終状態が`enabled = false`
* 後続Requestが`OBJECTIVE_DISABLED`になる
* `updated_at`が不要に複数回更新されない
* 他レコードに副作用がない

---

### 2.59 API ResourceのTest

正常時に、以下の形式となることを確認する。

```json
{
  "id": "1",
  "name": "一人暮らし",
  "plannedYearMonth": "2027-04",
  "requiredExpense": 500000,
  "memo": "引っ越し費用を含む",
  "enabled": false
}
```

---

### 2.60 API Resourceで返却しない項目

以下がレスポンスへ含まれないことを確認する。

* `userId`
* `user_id`
* `deletedAt`
* `deleted_at`
* DB内部管理情報
* `assessmentHistories`

日時項目については、目的API全体の共通仕様に従う。

---

### 2.61 Feature Test正常系

以下を実行する。

```http
PATCH /api/v1/objectives/1/disabled
X-User-Id: 1
```

期待結果：

```http
200 OK
```

かつ、

```text
objectives.enabled = false
```

となること。

---

### 2.62 Feature Test無効化済み

すでに、

```text
enabled = false
```

の目的へOBJ-005を実行する。

期待結果：

```text
409 Conflict
OBJECTIVE_DISABLED
```

となること。

---

### 2.63 副作用範囲Test

OBJ-005正常終了時に業務上変更されるのが、

```text
objectives.enabled
objectives.updated_at
```

だけであることを確認する。

以下が変更されないこと。

```text
objectives.user_id
objectives.name
objectives.planned_year_month
objectives.required_expense
objectives.memo
objectives.created_at
objectives.deleted_at
assessment_histories
```

---

## 3. 関連ドキュメント

- [OBJ-005 API詳細設計](../../../api/details/objectives/obj-005-assessments.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [目的 Laravelアーキテクチャ設計](./README.md)
- [OBJ-005 テスト設計](../../../tests/objectives/obj-005-assessments.md)
- [目的 テスト設計](../../../tests/objectives/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
