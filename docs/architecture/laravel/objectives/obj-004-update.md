# OBJ-004 目的更新

## 1. 概要

操作対象となる利用者に登録された
指定した目的の詳細情報を取得する。

本APIでは、
目的IDを指定して対象の目的を特定し、
目的編集画面および
目的達成判定画面の表示に必要な情報を返却する。

主に以下の情報を取得する。

* 目的ID
* 目的名
* 実施予定年月
* 必要支出額
* メモ
* 利用状態

目的は操作対象利用者単位で管理し、
他利用者に登録された目的は取得できない。

無効化された目的は取得対象とするが、
論理削除された目的は取得対象としない。

指定した目的が存在しない場合、
論理削除されている場合、
または操作対象利用者に帰属しない場合は、
目的を取得できないものとして扱う。

本APIでは目的情報の参照のみを行い、
目的の登録、更新、無効化および
目的達成判定は行わない。

また、目的達成判定結果および
過去の判定履歴は返却しない。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
目的ID、
更新内容および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの更新内容を受け取り、
目的更新UseCaseを呼び出す。

UseCaseから受け取った処理結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 入力値の単項目バリデーション
- 利用者境界の判定
- 更新対象の存在確認
- 目的の利用状態確認
- 目的名の重複確認
- 目的の更新処理
- トランザクション制御
- レスポンス生成処理

---

### 2.2 UseCase

目的更新の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 目的IDを受け取る
- 更新内容を受け取る
- 更新対象の目的を取得する
- 目的の利用状態を確認する
- 更新後の目的名について重複を確認する
- 指定された項目のみ更新する
- 更新結果を返却する

指定された目的が存在しない場合、
論理削除されている場合、
または操作対象利用者に帰属しない場合は、
`OBJECTIVE_NOT_FOUND`として扱う。

無効化された目的の場合は、
`OBJECTIVE_DISABLED`として扱う。

更新後の目的名が
同一利用者の他の目的と重複する場合は、
`OBJECTIVE_ALREADY_EXISTS`として扱う。

更新処理は、
データベーストランザクション内で実行する。

目的を更新しても、
保存済みの目的達成判定履歴は変更しない。

---

### 2.3 Form Request / DTO

入力値の形式および
単項目バリデーションを担当する。

主な検証対象は、
以下とする。

- `name`
- `plannedYearMonth`
- `requiredExpense`
- `memo`

Form Requestでは、
以下を検証する。

- データ型
- NULL可否
- 文字数
- 対象年月の形式
- 数値範囲
- 未定義項目の有無
- 更新対象外項目の有無
- 更新可能な項目が1つ以上指定されていること

目的の存在確認、
利用状態の確認および
目的名の重複確認など、
データベースの状態に依存する業務ルールは、
Form Requestへ記述しない。

検証済みの入力値は、
更新用DTOへ変換してUseCaseへ渡す。

DTOの例：

```php
final readonly class UpdateObjectiveInput
{
    public function __construct(
        public ?string $name,
        public ?string $plannedYearMonth,
        public ?int $requiredExpense,
        public ?string $memo,
        public bool $hasName,
        public bool $hasPlannedYearMonth,
        public bool $hasRequiredExpense,
        public bool $hasMemo,
    ) {
    }
}
```
PATCHでは、  
「未指定」と「`null` 指定」を区別する必要がある。

例えば、`plannedYearMonth` および `memo` では、以下を区別する。

```text
項目未指定
    → 現在値を維持する

項目へnullを指定
    → 未設定へ更新する
```

単純なnullableプロパティだけでは両者を区別できないため、DTOでは項目の指定有無を保持する。

### 2.4 パスパラメータ検証

`objectiveId` の形式は、API共通方針に従って検証する。

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること

形式が不正な場合は、`INVALID_OBJECTIVE_ID` として扱う。

目的の存在確認および利用者境界の確認は、Queryで行う。

### 2.5 Query

更新対象となる目的を取得する。

取得条件には、必ず目的IDおよび操作対象利用者IDを含める。

```php
$objective = Objective::query()
    ->where('id', $objectiveId)
    ->where('user_id', $userId)
    ->first();
```

以下のように、目的IDだけで取得してはならない。

```php
Objective::find($objectiveId);
```

LaravelのSoftDeletesを利用する場合、通常のEloquentクエリでは論理削除済みの目的を取得対象から除外する。

取得できなかった場合は、`OBJECTIVE_NOT_FOUND` として扱う。

取得した目的の `enabled` が `false` の場合は、`OBJECTIVE_DISABLED` として扱う。

また、更新後の目的名について、同一利用者の他の目的が存在するかを確認する。

```php
$exists = Objective::query()
    ->where('user_id', $userId)
    ->where('name', $newName)
    ->whereKeyNot($objectiveId)
    ->exists();
```

更新対象自身は、重複確認から除外する。

目的名がリクエストに含まれていない場合は、目的名の重複確認を行わない。

### 2.6 Repository

目的の更新を担当する。

更新対象は、以下のカラムとする。

- `name`
- `planned_year_month`
- `required_expense`
- `memo`
- `updated_at`

以下のカラムは更新しない。

- `id`
- `user_id`
- `enabled`
- `created_at`
- `deleted_at`

リクエストで指定された項目のみ更新する。

更新例：

```php
$attributes = [];

if ($input->hasName) {
    $attributes['name'] = $input->name;
}

if ($input->hasPlannedYearMonth) {
    $attributes['planned_year_month']
        = $input->plannedYearMonth;
}

if ($input->hasRequiredExpense) {
    $attributes['required_expense']
        = $input->requiredExpense;
}

if ($input->hasMemo) {
    $attributes['memo'] = $input->memo;
}

$objective->fill($attributes);
$objective->save();

return $objective;
```

更新後も、同じ目的IDを継続して使用する。

目的更新によって、`assessment_histories` は更新しない。

### 2.7 トランザクション

以下の処理を、1つのデータベーストランザクション内で実行する。

- 更新対象の取得
- 利用状態の確認
- 更新後目的名の重複確認
- 目的の更新

実装例：

```php
$objective = DB::transaction(
    function () use (
        $userId,
        $objectiveId,
        $input,
    ): Objective {
        $objective = $this->query->findByUserAndId(
            $userId,
            $objectiveId,
        );

        if ($objective === null) {
            throw new ObjectiveNotFoundException();
        }

        if (! $objective->enabled) {
            throw new ObjectiveDisabledException();
        }

        if (
            $input->hasName
            && $this->query->existsByUserAndNameExcludingId(
                $userId,
                $input->name,
                $objectiveId,
            )
        ) {
            throw new ObjectiveAlreadyExistsException();
        }

        return $this->repository->update(
            $objective,
            $input,
        );
    },
);
```

処理途中で例外が発生した場合は、更新内容をロールバックする。

同時更新によってUNIQUE制約違反が発生した場合は、内部例外をそのまま返却せず、`OBJECTIVE_ALREADY_EXISTS` へ変換する。

### 2.8 Responder

UseCaseから受け取った更新結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`200 OK` とともに更新後の目的を `data` オブジェクトで返却する。

以下の場合は、共通エラーレスポンス形式へ変換する。

- 更新対象が存在しない
- 目的が無効化されている
- 目的名が重複している
- 入力値が不正である
- 想定外の例外が発生した

Responderは、以下を行わない。

- 業務ルールの判定
- 利用者境界の判定
- データベース操作
- 判定履歴の更新

### 2.9 API Resource

データベースカラムを直接返却せず、API Resourceを利用してAPIレスポンス形式へ変換する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'name' => $this->name,
    'plannedYearMonth' => $this->planned_year_month,
    'requiredExpense' => $this->required_expense,
    'memo' => $this->memo,
    'enabled' => $this->enabled,
];
```

`planned_year_month` が未設定の場合は、`plannedYearMonth` へ `null` を設定する。

`memo` が未設定の場合は、`memo` へ `null` を設定する。

以下の項目は、レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`
- `deletedAt`

目的達成判定結果および判定履歴も、本APIでは返却しない。

### 2.10 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id` を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

### 2.11 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 目的ID形式不正 | `INVALID_OBJECTIVE_ID` |
| 目的不存在 | `OBJECTIVE_NOT_FOUND` |
| 論理削除済み目的 | `OBJECTIVE_NOT_FOUND` |
| 利用者境界外の目的 | `OBJECTIVE_NOT_FOUND` |
| 無効化済み目的 | `OBJECTIVE_DISABLED` |
| 目的名重複 | `OBJECTIVE_ALREADY_EXISTS` |
| 入力値不正 | `VALIDATION_ERROR` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

UNIQUE制約違反については、対象となった制約を判別し、`OBJECTIVE_ALREADY_EXISTS` へ変換する。

SQL、スタックトレースおよび内部例外メッセージは、APIレスポンスへ含めない。

ログには、調査に必要な範囲で以下を記録する。

- 操作対象利用者ID
- 目的ID
- 独自エラーコード
- リクエストID

目的名、必要支出額およびメモなどの業務データを不要にエラーログへ出力しない。

---

## 3. 関連ドキュメント

- [OBJ-004 API詳細設計](../../../api/details/objectives/obj-004-update.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [目的 Laravelアーキテクチャ設計](./README.md)
- [OBJ-004 テスト設計](../../../tests/objectives/obj-004-update.md)
- [目的 テスト設計](../../../tests/objectives/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)