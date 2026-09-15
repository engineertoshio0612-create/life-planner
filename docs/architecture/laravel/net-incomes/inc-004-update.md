# INC-004 手取り収入更新

## 1. 概要

本ドキュメントでは、
INC-004 手取り収入更新APIを
Laravelで実装する際の
アーキテクチャおよび責務分離方針を定義する。

本APIでは、
操作対象となる利用者に登録された
指定の手取り収入について、
対象年月、
手取り収入金額および
備考を更新する。

手取り収入は、
利用者ごとに対象年月単位で管理し、
同一利用者について
同一対象年月の手取り収入を
複数登録することはできない。
Laravel実装では、
以下の責務に分離する。

- Action
  - HTTPリクエストの受付
  - パスパラメータの取得
  - 利用者コンテキストの取得
  - 更新内容の受け取り
  - UseCaseの呼び出し
  - Responderへの処理結果の受け渡し

- UseCase
  - 手取り収入更新処理の制御
  - 更新対象の取得
  - 更新後対象年月の重複確認
  - Repositoryによる更新処理の実行
  - トランザクション制御

- Form Request / DTO
  - 更新内容の受け取り
  - 入力値のバリデーション
  - 更新用DTOへの変換

- Query
  - 操作対象利用者に帰属する更新対象の取得
  - 更新後対象年月の重複確認

- Repository
  - 手取り収入の永続化
  - 更新可能項目のみの更新

- Responder
  - API共通方針に従ったHTTPレスポンスへの変換
  - エラーレスポンスへの変換

- API Resource
  - データベースモデルからAPIレスポンス形式への変換
  - snake_caseからcamelCaseへの変換
  - IDの文字列化

更新対象の取得では、
手取り収入IDだけで検索せず、
利用者コンテキストから取得した利用者IDを
必ず検索条件に含める。

これにより、
指定された手取り収入が存在しない場合と、
他利用者に帰属する場合を区別せず、
`NET_INCOME_NOT_FOUND`として扱うことで、
利用者境界を保証する。

更新処理は、
データベーストランザクション内で実行し、
更新対象の取得、
更新後対象年月の重複確認および
手取り収入の更新を
一連の処理として扱う。

対象年月の重複については、
アプリケーションによる事前確認に加えて、
データベースのUNIQUE制約を
最終的な重複防止手段として使用する。

同時実行などによって
UNIQUE制約違反が発生した場合は、
データベース内部の例外を直接公開せず、
`NET_INCOME_ALREADY_EXISTS`へ変換する。

更新後の手取り収入は、
平均手取り収入の算出および
目的達成判定に利用する。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
手取り収入ID、
更新内容および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの更新内容を受け取り、
手取り収入更新UseCaseを呼び出す。

UseCaseから受け取った処理結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 入力値の単項目バリデーション
- 利用者境界の判定
- 更新対象の存在確認
- 対象年月の重複確認
- 手取り収入の更新処理
- トランザクション制御
- レスポンス生成処理

---

### 2.2 UseCase

手取り収入更新の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 手取り収入IDを受け取る
- 更新内容を受け取る
- 更新対象の手取り収入を取得する
- 更新後の対象年月について重複を確認する
- 手取り収入を更新する
- 更新結果を返却する

指定された手取り収入が存在しない場合、
または操作対象利用者に帰属しない場合は、
`NET_INCOME_NOT_FOUND`として扱う。

更新後の対象年月が
同一利用者の他の手取り収入と重複する場合は、
`NET_INCOME_ALREADY_EXISTS`として扱う。

更新処理は、
データベーストランザクション内で実行する。

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
- 対象年月の形式
- 数値範囲
- 文字数
- 未定義項目の有無
- 更新対象外項目の有無

対象年月の重複や
更新対象の存在確認など、
データベースの状態に依存する業務ルールは、
Form Requestへ記述しない。

検証済みの入力値は、
更新用DTOへ変換してUseCaseへ渡す。

DTOの例：

```php
final readonly class UpdateNetIncomeInput
{
    public function __construct(
        public string $targetYearMonth,
        public int $amount,
        public ?string $memo,
    ) {
    }
}
```

## 2.4 Query

更新対象となる手取り収入を取得する。

取得条件には、必ず手取り収入IDおよび操作対象利用者IDを含める。

```php
$netIncome = NetIncome::query()
    ->where('id', $netIncomeId)
    ->where('user_id', $userId)
    ->first();
```

以下のように、手取り収入IDだけで取得してはならない。

```php
NetIncome::find($netIncomeId);
```

取得できなかった場合は、`NET_INCOME_NOT_FOUND` として扱う。

また、更新後の対象年月について、同一利用者の他の手取り収入が存在するかを確認する。

```php
$exists = NetIncome::query()
    ->where('user_id', $userId)
    ->where('target_year_month', $input->targetYearMonth)
    ->whereKeyNot($netIncomeId)
    ->exists();
```

更新対象自身は、重複確認から除外する。

---

## 2.5 Repository

手取り収入の更新を担当する。

更新対象は、以下のカラムとする。

- `target_year_month`
- `amount`
- `memo`
- `updated_at`

以下のカラムは更新しない。

- `id`
- `user_id`
- `created_at`

更新例：

```php
$netIncome->fill([
    'target_year_month' => $input->targetYearMonth,
    'amount' => $input->amount,
    'memo' => $input->memo,
]);

$netIncome->save();
```

更新後も、同じ手取り収入IDを継続して使用する。

---

## 2.6 トランザクション

以下の処理を、1つのデータベーストランザクション内で実行する。

- 更新対象の取得
- 更新後対象年月の重複確認
- 手取り収入の更新

実装例：

```php
$netIncome = DB::transaction(
    function () use (
        $userId,
        $netIncomeId,
        $input,
    ): NetIncome {
        $netIncome = $this->query->findByUserAndId(
            $userId,
            $netIncomeId,
        );

        if ($netIncome === null) {
            throw new NetIncomeNotFoundException();
        }

        if ($this->query->existsByUserAndYearMonthExcludingId(
            $userId,
            $input->targetYearMonth,
            $netIncomeId,
        )) {
            throw new NetIncomeAlreadyExistsException();
        }

        return $this->repository->update(
            $netIncome,
            $input,
        );
    },
);
```

処理途中で例外が発生した場合は、更新内容をロールバックする。

同時更新によってUNIQUE制約違反が発生した場合は、内部例外をそのまま返却せず、`NET_INCOME_ALREADY_EXISTS` へ変換する。

---

## 2.7 Responder

UseCaseから受け取った更新結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`200 OK` とともに更新後の手取り収入を `data` オブジェクトで返却する。

以下の場合は、共通エラーレスポンス形式へ変換する。

- 更新対象が存在しない
- 更新後の対象年月が重複している
- 入力値が不正である
- 想定外の例外が発生した

Responderは、業務ルールの判定やデータベース操作を行わない。

---

## 2.8 API Resource

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

## 2.9 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id` を検証し、操作対象利用者を特定する。

Action以降の処理では、検証済みの利用者コンテキストを使用する。

---

## 2.10 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 手取り収入ID形式不正 | `INVALID_NET_INCOME_ID` |
| 手取り収入不存在 | `NET_INCOME_NOT_FOUND` |
| 更新後対象年月の重複 | `NET_INCOME_ALREADY_EXISTS` |
| 入力値不正 | `VALIDATION_ERROR` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

UNIQUE制約違反については、対象となった制約を判別し、`NET_INCOME_ALREADY_EXISTS` へ変換する。

SQL、スタックトレースおよび内部例外メッセージは、APIレスポンスへ含めない。

---

## 3. 関連ドキュメント

- [INC-004 API詳細設計](../../../api/details/net-incomes/inc-004-update.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [手取り収入 Laravelアーキテクチャ設計](./README.md)
- [INC-004 テスト設計](../../../tests/net-incomes/inc-004-update.md)
- [手取り収入 テスト設計](../../../tests/net-incomes/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)