# INC-002 手取り収入登録

## 1. 概要

本ドキュメントでは、
INC-002 手取り収入登録APIを
Laravelで実装する際の
アーキテクチャおよび責務分離方針を定義する。

本APIでは、
操作対象となる利用者に対して、
指定された対象年月の手取り収入を
新規登録する。

手取り収入は、
利用者ごとに対象年月単位で管理し、
同一利用者について
同一対象年月の手取り収入を
複数登録することはできない。

対象年月に手取り収入が発生しなかった場合も、
未登録とはせず、
金額を0円として登録する。
Laravel実装では、
以下の責務に分離する。

- Action
  - HTTPリクエストの受付
  - 利用者コンテキストの取得
  - UseCaseの呼び出し
  - Responderへの処理結果の受け渡し

- UseCase
  - 手取り収入登録処理の制御
  - 対象年月の重複確認
  - Repositoryによる登録処理の実行
  - トランザクション制御

- Form Request / DTO
  - 登録内容の受け取り
  - 入力値のバリデーション
  - 登録用DTOへの変換

- Query
  - 同一利用者・同一対象年月の登録有無の確認

- Repository
  - 手取り収入の永続化

- Responder
  - API共通方針に従ったHTTPレスポンスへの変換
  - エラーレスポンスへの変換

- API Resource
  - データベースモデルからAPIレスポンス形式への変換
  - snake_caseからcamelCaseへの変換
  - IDの文字列化

登録処理は、
データベーストランザクション内で実行し、
対象年月の重複確認と
手取り収入の登録を一連の処理として扱う。

また、アプリケーションによる事前の重複確認に加えて、
データベースのUNIQUE制約を
最終的な重複防止手段として使用する。
同時実行などによって
UNIQUE制約違反が発生した場合は、
データベース内部の例外を直接公開せず、
`NET_INCOME_ALREADY_EXISTS`へ変換する。
登録時の利用者IDは、
リクエストボディから受け取らず、
利用者コンテキストから取得することで、
他利用者の手取り収入として
登録できないようにする。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
入力値および利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
登録内容を受け取り、
手取り収入登録UseCaseを呼び出す。

UseCaseから受け取った処理結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 入力値の単項目バリデーション
- 対象年月の重複確認
- 手取り収入の登録処理
- トランザクション制御
- レスポンス生成処理

---

### 2.2 UseCase

手取り収入登録の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 登録内容を受け取る
- 同一対象年月の登録有無を確認する
- 手取り収入を登録する
- 登録結果を返却する

登録処理は、
データベーストランザクション内で実行する。

同一利用者について、
同一対象年月の手取り収入がすでに存在する場合は、
`NET_INCOME_ALREADY_EXISTS`として扱う。

---

### 2.3 Form Request / DTO

入力値の形式および
単項目バリデーションを担当する。

主な検証対象は、
以下とする。

- `targetYearMonth`
- `amount`
- `memo`

Form Requestでは、
以下を検証する。

- 必須項目
- データ型
- NULL可否
- 文字数
- 数値範囲
- 対象年月の形式
- 未定義項目の有無

同一対象年月の登録有無など、
データベースの状態に依存する業務ルールは、
Form Requestへ記述しない。

検証済みの入力値は、
登録用DTOへ変換してUseCaseへ渡す。

DTOの例：

```php
final readonly class CreateNetIncomeInput
{
    public function __construct(
        public string $targetYearMonth,
        public int $amount,
        public ?string $memo,
    ) {
    }
}
```

### 2.4 Query

同一利用者について、指定された対象年月の手取り収入がすでに存在するかを確認する。

検索条件には、必ず操作対象利用者IDを含める。

確認例：

```php
$exists = NetIncome::query()
    ->where('user_id', $userId)
    ->where('target_year_month', $targetYearMonth)
    ->exists();
```

Queryによる事前確認だけでなく、データベースのUNIQUE制約を最終的な重複防止手段として使用する。

---

### 2.5 Repository

手取り収入の永続化を担当する。

Repositoryは、UseCaseから受け取った利用者IDおよび登録内容を使用して、`net_incomes` へ新しいレコードを登録する。

登録例：

```php
$netIncome = NetIncome::query()->create([
    'user_id' => $userId,
    'target_year_month' => $input->targetYearMonth,
    'amount' => $input->amount,
    'memo' => $input->memo,
]);
```

以下の項目は、クライアントから受け取らない。

- `id`
- `user_id`
- `created_at`
- `updated_at`

`user_id` は、利用者コンテキストから設定する。

`id`、`created_at` および `updated_at` は、データベースまたはLaravelによって自動設定する。

---

### 2.6 トランザクション

以下の処理を、1つのデータベーストランザクション内で実行する。

- 対象年月の重複確認
- 手取り収入の登録

実装例：

```php
$netIncome = DB::transaction(
    function () use ($userId, $input): NetIncome {
        if ($this->query->existsByUserAndYearMonth(
            $userId,
            $input->targetYearMonth,
        )) {
            throw new NetIncomeAlreadyExistsException();
        }

        return $this->repository->create(
            $userId,
            $input,
        );
    },
);
```

処理途中で例外が発生した場合は、登録内容をロールバックする。

同時登録によってUNIQUE制約違反が発生した場合は、内部例外をそのまま返却せず、`NET_INCOME_ALREADY_EXISTS` へ変換する。

---

### 2.7 Responder

UseCaseから受け取った登録結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`201 Created` とともに登録した手取り収入を `data` オブジェクトで返却する。

対象年月の重複、バリデーションエラーまたは予期しない例外が発生した場合は、共通エラーレスポンス形式へ変換する。

Responderは、業務ルールの判定やデータベース操作を行わない。

---

### 2.8 API Resource

データベースカラムを直接返却せず、API Resourceを利用してAPIレスポンス形式へ変換する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'targetYearMonth' => $this->target_year_month,
    'amount' => $this->amount,
    'memo' => $this->memo,
];
```

手取り収入IDは、API共通方針に従って文字列として返却する。

金額は、日本円の整数値として返却する。

以下の項目は、レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`

API Resourceは、Responderから利用する。

---

### 2.9 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id` を検証し、操作対象利用者を特定する。

Action以降の処理では、検証済みの利用者コンテキストを使用する。

---

### 2.10 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 同一対象年月が登録済み | `NET_INCOME_ALREADY_EXISTS` |
| 入力値不正 | `VALIDATION_ERROR` |
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

UNIQUE制約違反については、対象となった制約を判別し、`NET_INCOME_ALREADY_EXISTS` へ変換する。

SQL、スタックトレースおよび内部例外メッセージは、APIレスポンスへ含めない。

---

---

## 3. 関連ドキュメント

- [INC-002 API詳細設計](../../../api/details/net-incomes/inc-002-create.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [手取り収入 Laravelアーキテクチャ設計](./README.md)
- [INC-002 テスト設計](../../../tests/net-incomes/inc-002-create.md)
- [手取り収入 テスト設計](../../../tests/net-incomes/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)