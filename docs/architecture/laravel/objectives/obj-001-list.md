# OBJ-001 目的一覧取得

## 1. 概要

操作対象となる利用者に登録された
目的一覧を取得する。

目的は、
達成したい内容、
実施予定年月、
必要支出額および
利用状態を管理する。

本APIでは、
利用者に帰属する目的を取得し、
目的一覧画面の表示に必要な情報として返却する。

利用中の目的だけでなく、
無効化された目的も取得対象とする。

一覧取得では、主に以下の情報を返却する。

* 目的ID
* 目的名
* 実施予定年月
* 必要支出額
* メモ
* 利用状態

目的は操作対象利用者単位で管理し、
他利用者に登録された目的は取得しない。

取得結果が0件の場合は、
正常終了として空配列を返却する。

本APIでは目的情報の参照のみを行い、
目的の登録、更新、無効化および
達成判定は行わない。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
入力値および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの入力値を受け取り、
目的登録UseCaseを呼び出す。

UseCaseから受け取った処理結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 入力値の単項目バリデーション
- 利用者境界の判定
- 目的名重複確認
- 登録処理
- トランザクション制御
- レスポンス生成処理

---

### 2.2 UseCase

目的登録の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 入力内容を受け取る
- 同一利用者内で目的名の重複を確認する
- 新しい目的を登録する
- 登録結果を返却する

目的名が重複する場合は、
`OBJECTIVE_ALREADY_EXISTS`
として扱う。

登録処理は、
データベーストランザクション内で実行する。

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

- 必須項目
- データ型
- NULL可否
- 文字数
- 対象年月形式
- 数値範囲
- 未定義項目
- 更新不可項目

目的名の重複確認など、
データベース状態に依存する業務ルールは、
Form Requestへ記述しない。

検証済みの入力値は、
登録用DTOへ変換してUseCaseへ渡す。

DTOの例：

```php
final readonly class CreateObjectiveInput
{
    public function __construct(
        public string $name,
        public ?string $plannedYearMonth,
        public int $requiredExpense,
        public ?string $memo,
    ) {
    }
}
```

---

### 2.4 Query

同一利用者に
同じ目的名が存在するか確認する。

検索条件には、
必ず利用者IDを含める。

```php
$exists = Objective::query()
    ->where('user_id', $userId)
    ->where('name', $input->name)
    ->exists();
```

存在する場合は、
`OBJECTIVE_ALREADY_EXISTS`
として扱う。

---

### 2.5 Repository

目的の登録を担当する。

登録対象は、
以下とする。

- `user_id`
- `name`
- `planned_year_month`
- `required_expense`
- `memo`
- `enabled`

登録時には、
以下の初期値を設定する。

- `enabled = true`
- `created_at = 現在日時`
- `updated_at = 現在日時`

`deleted_at`は設定しない。

登録例：

```php
return Objective::create([
    'user_id' => $userId,
    'name' => $input->name,
    'planned_year_month' => $input->plannedYearMonth,
    'required_expense' => $input->requiredExpense,
    'memo' => $input->memo,
    'enabled' => true,
]);
```

---

### 2.6 トランザクション

以下の処理を、
1つのデータベーストランザクション内で実行する。

- 目的名重複確認
- 目的登録

実装例：

```php
$objective = DB::transaction(
    function () use ($userId, $input): Objective {
        if ($this->query->existsByUserAndName(
            $userId,
            $input->name,
        )) {
            throw new ObjectiveAlreadyExistsException();
        }

        return $this->repository->create(
            $userId,
            $input,
        );
    },
);
```

処理途中で例外が発生した場合は、
登録内容をロールバックする。

同時登録によって
UNIQUE制約違反が発生した場合は、
`OBJECTIVE_ALREADY_EXISTS`
へ変換する。

---

### 2.7 Responder

UseCaseから受け取った登録結果を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`201 Created`
とともに
登録した目的を
`data`オブジェクトで返却する。

以下の場合は、
共通エラーレスポンス形式へ変換する。

- 目的名重複
- 入力値不正
- 利用者不存在
- 想定外例外

Responderは、
業務ルールの判定や
データベース操作を行わない。

---

### 2.8 API Resource

データベースカラムを直接返却せず、
API Resourceを利用して
APIレスポンス形式へ変換する。

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

以下の項目は、
レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`
- `deletedAt`

---

### 2.9 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、
`X-User-Id`を検証し、
操作対象利用者を特定する。

Action以降では、
検証済みの利用者コンテキストを使用する。

---

### 2.10 例外変換

LaravelおよびPostgreSQLの内部例外は、
そのままAPIレスポンスへ公開しない。

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 目的名重複 | `OBJECTIVE_ALREADY_EXISTS` |
| 入力値不正 | `VALIDATION_ERROR` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

UNIQUE制約違反については、
対象となった制約を判別し、
`OBJECTIVE_ALREADY_EXISTS`
へ変換する。

SQL、
スタックトレースおよび
内部例外メッセージは、
APIレスポンスへ含めない。

---

## 3. 関連ドキュメント

- [CSV-001 API詳細設計](../../../api/details/csv-imports/csv-001-template.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [CSVインポート Laravelアーキテクチャ設計](./README.md)
- [CSV-001 テスト設計](../../../tests/csv-imports/csv-001-template.md)
- [CSVインポート テスト設計](../../../tests/csv-imports/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
