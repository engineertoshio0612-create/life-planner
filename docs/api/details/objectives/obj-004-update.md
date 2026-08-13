# OBJ-004 目的更新

## 1. 概要

操作対象となる利用者に登録された
有効な目的を更新する。

本APIでは、
以下の項目を更新できる。

- 目的名
- 実施予定年月
- 必要支出額
- メモ

利用状態は、
本APIの更新対象としない。

目的の無効化は、
OBJ-005 目的無効化APIで行う。

目的を更新しても、
過去に保存された目的達成判定履歴は変更しない。

---

## 2. ユースケース

利用者は、
登録済みの目的について、
内容の誤りや計画変更を反映する。

例えば、
以下のような場合に使用する。

- 目的名を変更する
- 実施予定年月を設定または変更する
- 実施予定年月を未設定へ戻す
- 必要支出額を変更する
- メモを追加、変更または削除する

更新後の目的情報は、
今後実行する目的達成判定で利用する。

過去の判定結果を更新後の内容で確認したい場合は、
目的達成判定を再実行する。

---

## 3. エンドポイント

```http
PATCH /api/v1/objectives/{objectiveId}
```

---

## 4. HTTPメソッド

`PATCH`

本APIは、既存の目的の更新可能な項目を変更する。

更新対象として指定されなかった項目は、既存の値を維持する。

登録、無効化および目的達成判定は行わない。

---

## 5. 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、`X-User-Id` リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

指定された利用者に登録された目的のみ更新できる。

他の利用者に登録された目的は、更新できない。

他の利用者に帰属する目的を指定した場合は、対象が存在しないものとして扱う。

無効化された目的は、本APIでは更新できない。

論理削除された目的も、更新対象としない。

利用者IDは、リクエストボディ、クエリパラメータまたはパスパラメータでは受け付けない。

利用状態を表す `enabled` も、リクエストから指定できない。

目的の無効化は、OBJ-005 目的無効化APIで行う。

---
## 6. パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `objectiveId` | string | ○ | 更新対象の目的ID |

リクエスト例

```http
PATCH /api/v1/objectives/1
```

---

## 7. クエリパラメータ

なし。

---

## 8. リクエストヘッダー

### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Content-Type` | ○ | `application/json`を指定する |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例

```http
PATCH /api/v1/objectives/1
Content-Type: application/json
Accept: application/json
X-User-Id: 1
```

---

## 9. リクエストボディ

```json
{
  "name": "マイホーム購入（頭金）",
  "plannedYearMonth": "2031-03",
  "requiredExpense": 6000000,
  "memo": "物価上昇を考慮して金額を見直す"
}
```

更新したい項目のみ指定する。

---

## 10. リクエスト項目

| 項目 | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `name` | string | × | 目的名 |
| `plannedYearMonth` | string | × | 実施予定年月（YYYY-MM） |
| `requiredExpense` | integer | × | 必要支出額（円） |
| `memo` | string | × | メモ |

---

### 10.1 name

目的名を更新する。

未指定の場合は、
現在の値を保持する。

---

### 10.2 plannedYearMonth

実施予定年月を更新する。

`YYYY-MM`形式で指定する。

未設定にする場合は、
`null`を指定する。

未指定の場合は、
現在の値を保持する。

---

### 10.3 requiredExpense

必要支出額を更新する。

日本円の整数値で指定する。

未指定の場合は、
現在の値を保持する。

---

### 10.4 memo

メモを更新する。

未設定にする場合は、
`null`を指定する。

未指定の場合は、
現在の値を保持する。

---

## 11. バリデーション

### 11.1 X-User-Id

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

---

### 11.2 objectiveId

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された目的が存在すること
- 指定された目的が論理削除されていないこと
- 指定された目的が操作対象利用者に属していること
- 指定された目的が有効であること

---

### 11.3 name

指定された場合は、
以下を検証する。

- 文字列であること
- `null`でないこと
- 1文字以上100文字以下であること
- 前後の空白を除去した結果が空文字でないこと
- 同一利用者内で重複しないこと

---

### 11.4 plannedYearMonth

指定された場合は、
以下を検証する。

- 文字列であること
- `null`を許可すること
- `YYYY-MM`形式であること
- 実在する年月であること

以下は不正な値とする。

```text
2030-3
2030/03
2030-13
203003
```

---

### 11.5 requiredExpense

指定された場合は、
以下を検証する。

- 整数であること
- `null`でないこと
- 0以上であること
- int型の範囲内であること

小数および負数は受け付けない。

---

### 11.6 memo

指定された場合は、
以下を検証する。

- 文字列であること
- `null`を許可すること
- 最大文字数以内であること

空文字列は、
`null`へ正規化してよい。

---

### 11.7 更新不可項目

以下の項目は、
リクエストで受け付けない。

- `id`
- `userId`
- `enabled`
- `createdAt`
- `updatedAt`
- `deletedAt`

指定された場合は、
バリデーションエラーとする。

---

### 11.8 未定義項目

定義されていない項目を
リクエストへ含めてはならない。

未定義項目が指定された場合は、
バリデーションエラーとする。

---

## 18. エラーレスポンス

異常時は、
API共通方針で定めた
共通エラーレスポンス形式を使用する。

---

### 18.1 指定した目的が存在しない場合

指定した目的が存在しない場合は、
`OBJECTIVE_NOT_FOUND`
を返却する。

```json
{
  "error": {
    "code": "OBJECTIVE_NOT_FOUND",
    "message": "指定された目的が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 18.2 目的名が重複する場合

同一利用者において、
同じ目的名がすでに登録されている場合は、
`OBJECTIVE_ALREADY_EXISTS`
を返却する。

```json
{
  "error": {
    "code": "OBJECTIVE_ALREADY_EXISTS",
    "message": "同じ目的名がすでに登録されています。",
    "details": [
      {
        "field": "name",
        "reason": "duplicated",
        "message": "目的名が重複しています。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 18.3 無効化された目的を更新した場合

無効化された目的を更新しようとした場合は、
`OBJECTIVE_DISABLED`
を返却する。

```json
{
  "error": {
    "code": "OBJECTIVE_DISABLED",
    "message": "無効化された目的は更新できません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 18.4 バリデーションエラー

入力値が不正な場合は、
`VALIDATION_ERROR`
を返却する。

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "requiredExpense",
        "reason": "min",
        "message": "必要支出額を確認してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

## 19. HTTPステータス

| HTTPステータス | 条件 |
|---:|---|
| `200 OK` | 目的更新に成功した |
| `400 Bad Request` | 利用者IDまたは目的IDの形式が不正である |
| `404 Not Found` | 指定された利用者または目的が存在しない |
| `409 Conflict` | 同一利用者で目的名が重複している |
| `422 Unprocessable Entity` | 入力値が不正、または無効化された目的である |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

---

### 19.1 404の扱い

以下の場合は、
`404 Not Found`
を返却する。

- 指定された利用者が存在しない
- 指定された利用者が論理削除されている
- 指定された目的が存在しない
- 指定された目的が論理削除されている
- 指定された目的が操作対象利用者に属していない

---

### 19.2 409の扱い

以下の場合は、
`409 Conflict`
を返却する。

- 同一利用者に同じ目的名が登録されている

入力値は正しいが、
現在のリソース状態と競合しているため、
`409 Conflict`
を返却する。

---

### 19.3 422の扱い

以下の場合は、
`422 Unprocessable Entity`
を返却する。

- 入力値が不正である
- 無効化された目的を更新しようとした

---

## 20. エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `USER_CONTEXT_REQUIRED` | 400 | `X-User-Id`が指定されていない | × |
| `INVALID_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `INVALID_OBJECTIVE_ID` | 400 | 目的IDの形式が不正である | × |
| `USER_NOT_FOUND` | 404 | 指定された利用者が存在しない | × |
| `OBJECTIVE_NOT_FOUND` | 404 | 指定された目的が存在しない、または操作対象利用者に属していない | × |
| `OBJECTIVE_ALREADY_EXISTS` | 409 | 同一利用者で目的名が重複している | × |
| `OBJECTIVE_DISABLED` | 422 | 無効化された目的を更新しようとした | × |
| `VALIDATION_ERROR` | 422 | 入力値が不正である | × |
| `INTERNAL_SERVER_ERROR` | 500 | 想定外のサーバーエラーが発生した | ○ |

エラーコードの正式な定義は、
`error-codes.md`
に従う。

---

## 21. 冪等性

本APIは、
HTTP PATCHを使用する。

同一内容で複数回実行した場合、
1回目の更新後は
同じ状態が維持される。

そのため、
本APIは冪等である。

ただし、
更新対象の目的が途中で変更された場合は、
レスポンス内容が異なる場合がある。

Phase1では、
ETagやIf-Matchによる楽観ロックは採用しない。

---

## 22. 関連テーブル

### 22.1 objectives

目的の情報を保持する。

使用する主なカラムは、
以下とする。

| カラム | 用途 |
|---|---|
| `id` | 目的ID |
| `user_id` | 利用者境界 |
| `name` | 目的名 |
| `planned_year_month` | 実施予定年月 |
| `required_expense` | 必要支出額 |
| `memo` | メモ |
| `enabled` | 利用状態 |
| `updated_at` | 更新日時 |

本APIでは、
指定した目的の情報を更新する。

更新対象は、
以下とする。

- `name`
- `planned_year_month`
- `required_expense`
- `memo`

以下の項目は更新しない。

- `id`
- `user_id`
- `enabled`
- `created_at`
- `deleted_at`

更新時は、
`updated_at`を現在日時へ更新する。

---

## 23. 関連する機能要件

- `10.3 編集`
  - 登録済みの目的を編集できる
- `10.9 業務ルール`
  - 目的名は利用者ごとに一意とする
  - 実施予定年月は任意入力とする
  - 必要支出額は必須項目とする
  - 保存済みの判定結果は、目的を編集しても変更しない

---

## 24. テスト観点

### 24.1 正常系

- 目的名を更新できること
- 実施予定年月を更新できること
- 実施予定年月を`null`へ更新できること
- 必要支出額を更新できること
- メモを更新できること
- メモを`null`へ更新できること
- 複数項目を同時に更新できること
- 変更対象以外の項目は保持されること
- `200 OK`で返却されること

---

### 24.2 利用者境界

- 操作対象利用者の目的のみ更新できること
- 他利用者の目的を更新できないこと
- 他利用者の目的IDを指定した場合は`404 Not Found`となること

---

### 24.3 利用状態

- 有効な目的を更新できること
- 無効化された目的は更新できず`422 Unprocessable Entity`となること
- 論理削除済みの目的は更新できないこと

---

### 24.4 目的名

- 同一利用者内で重複しないこと
- 他利用者では同じ目的名へ更新できること
- 最大文字数以内で更新できること
- 最大文字数超過で`422 Unprocessable Entity`となること
- 空文字で`422 Unprocessable Entity`となること

---

### 24.5 実施予定年月

- `YYYY-MM`形式で更新できること
- `null`へ更新できること
- 不正な年月形式で`422 Unprocessable Entity`となること
- 存在しない年月で`422 Unprocessable Entity`となること

---

### 24.6 必要支出額

- 正常な整数値で更新できること
- 最小値で更新できること
- 最大値で更新できること
- 負数で`422 Unprocessable Entity`となること
- 小数で`422 Unprocessable Entity`となること
- 最大値超過で`422 Unprocessable Entity`となること

---

### 24.7 メモ

- メモを更新できること
- `null`へ更新できること
- 空文字が`null`へ正規化されること
- 最大文字数以内で更新できること
- 最大文字数超過で`422 Unprocessable Entity`となること

---

### 24.8 更新対象外項目

- `id`を更新できないこと
- `userId`を更新できないこと
- `enabled`を更新できないこと
- `createdAt`を更新できないこと
- `updatedAt`を更新できないこと
- `deletedAt`を更新できないこと

---

### 24.9 判定履歴

- 目的更新後も過去の判定履歴が変更されないこと
- 実施予定年月を変更しても過去の判定履歴が再計算されないこと
- 必要支出額を変更しても過去の判定履歴が変更されないこと

---

### 24.10 更新日時

- 更新時に`updated_at`が更新されること
- `created_at`が変更されないこと

---

### 24.11 トランザクション

- 更新途中で例外が発生した場合はロールバックされること
- 重複更新時は更新されないこと

---

### 24.12 同時更新

- 同時更新時にデータ整合性が維持されること
- UNIQUE制約違反時は`409 Conflict`となること
- 想定外例外へ変換されないこと

---

### 24.13 レスポンス契約

- JSONフィールド名がcamelCaseであること
- 目的IDが文字列で返却されること
- 実施予定年月が`YYYY-MM`形式または`null`で返却されること
- 必要支出額が整数で返却されること
- メモが文字列または`null`で返却されること
- 利用状態がbooleanで返却されること
- `userId`がレスポンスへ含まれないこと
- `createdAt`がレスポンスへ含まれないこと
- `updatedAt`がレスポンスへ含まれないこと
- `deletedAt`がレスポンスへ含まれないこと
- `data`オブジェクトで返却されること

---

### 24.14 異常系

- 想定外の例外で`500 Internal Server Error`となること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログで同じリクエストIDが記録されること
- SQLおよびスタックトレースなどの内部情報がレスポンスへ含まれないこと

---

## 25. Laravel実装方針

### 25.1 Action

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

### 25.2 UseCase

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

### 25.3 Form Request / DTO

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

### 25.4 パスパラメータ検証

`objectiveId` の形式は、API共通方針に従って検証する。

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること

形式が不正な場合は、`INVALID_OBJECTIVE_ID` として扱う。

目的の存在確認および利用者境界の確認は、Queryで行う。

### 25.5 Query

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

### 25.6 Repository

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

### 25.7 トランザクション

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

### 25.8 Responder

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

### 25.9 API Resource

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

### 25.10 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id` を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

### 25.11 例外変換

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

---## 26. React・TypeScriptでの利用

更新リクエスト型は、
以下とする。

```ts
export type UpdateObjectiveRequest = {
  name?: string;
  plannedYearMonth?: string | null;
  requiredExpense?: number;
  memo?: string | null;
};
```

レスポンス型は、
以下とする。

```ts
export type Objective = {
  id: string;
  name: string;
  plannedYearMonth: string | null;
  requiredExpense: number;
  memo: string | null;
  enabled: boolean;
};

export type UpdateObjectiveResponse = {
  data: Objective;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.patch<
    UpdateObjectiveResponse
  >(
    `/api/v1/objectives/${objectiveId}`,
    {
      name: 'マイホーム購入（頭金）',
      plannedYearMonth: '2031-03',
      requiredExpense: 6000000,
      memo: '物価上昇を考慮して見直し',
    },
  );
```

更新成功後は、
目的詳細画面または
目的一覧画面へ反映する。

---

### 26.1 PATCHの扱い

本APIは、
PATCHを使用する。

更新する項目のみ送信する。

未指定の項目は、
現在の値を保持する。

---

### 26.2 plannedYearMonthの扱い

`plannedYearMonth`は、
`YYYY-MM`形式で送信する。

未設定にする場合は、
`null`を送信する。

未変更の場合は、
送信しない。

```ts
{
  plannedYearMonth: null
}
```

---

### 26.3 requiredExpenseの扱い

`requiredExpense`は、
日本円の整数値として扱う。

送信前に、
数値へ変換する。

```ts
requiredExpense:
  Number(form.requiredExpense);
```

小数点は送信しない。

---

### 26.4 memoの扱い

`memo`は、
任意入力とする。

削除する場合は、
`null`を送信する。

未変更の場合は、
送信しない。

```ts
{
  memo: null
}
```

---

### 26.5 enabledの扱い

`enabled`は、
レスポンスでのみ取得する。

更新APIでは、
送信しない。

利用状態の変更は、
目的無効化APIで行う。

---

### 26.6 ローディング表示

更新処理中は、
更新ボタンを非活性化する。

更新完了または
エラーになるまで、
再送信できないようにする。

---

### 26.7 エラー表示

入力項目ごとのエラーは、
`error.details.field`
を利用して表示する。

対象となる項目は、
以下とする。

- `name`
- `plannedYearMonth`
- `requiredExpense`
- `memo`

エラーコードごとの基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `VALIDATION_ERROR` | 入力欄へエラー表示 |
| `OBJECTIVE_ALREADY_EXISTS` | 「同じ目的名が登録されています」を表示 |
| `OBJECTIVE_DISABLED` | 「無効化された目的は更新できません」を表示 |
| `OBJECTIVE_NOT_FOUND` | 一覧画面へ戻し、対象が存在しないことを表示する |
| `INVALID_OBJECTIVE_ID` | 不正なURLとしてエラー表示する |
| `INVALID_USER_ID` | 共通エラー表示 |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示 |

---

## 27. 設計上の補足

### 27.1 PATCHを採用する理由

目的更新では、
変更した項目のみ更新できればよい。

そのため、
PUTではなく
PATCHを採用する。

---

### 27.2 利用状態を更新対象にしない理由

利用状態の変更は、
業務上独立した操作である。

そのため、
目的更新APIでは扱わず、
目的無効化APIへ責務を分離する。

---

### 27.3 判定履歴を更新しない理由

保存済みの目的達成判定履歴は、
実行当時の結果として保持する。

目的を更新しても、
過去の判定履歴は変更しない。

新しい内容で判定したい場合は、
目的達成判定APIを再実行する。

---

### 27.4 実施予定年月を未設定にできる理由

実施予定年月は、
任意入力項目である。

そのため、
`null`を指定することで
未設定へ戻すことができる。

---

### 27.5 必要支出額を必須とする理由

実施予定年月が未設定であっても、
目的達成判定では
必要支出額を利用する。

そのため、
必要支出額は
常に必須項目とする。

---

### 27.6 PATCHで未指定項目を保持する理由

利用者が変更したい項目のみを
送信できるようにするためである。

未指定項目は、
現在の値を保持する。

---

### 27.7 enabledをレスポンスへ含める理由

更新後の利用状態を
画面側で判定できるようにするためである。

更新ボタンや
目的達成判定ボタンの表示制御に利用する。

---

### 27.8 冪等性キーを採用しない理由

PATCHは冪等であり、
同じ更新内容を繰り返し送信しても
最終状態は変化しない。

また、
Phase1では
個人利用を前提とするため、
送信ボタンの非活性化による
二重送信防止で十分と判断する。

---

## 28. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)