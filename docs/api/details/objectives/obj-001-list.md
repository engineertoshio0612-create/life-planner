# OBJ-001 目的一覧取得

## 1. 概要

操作対象となる利用者に登録された
目的一覧を取得する。

目的は、
達成したい内容、
実施予定年月、
必要支出額および
利用状態を管理する。

一覧取得では、
目的一覧画面の表示に必要な情報を取得する。

---

## 2. ユースケース

利用者は、
登録済みの目的を一覧で確認する。

例えば、
以下のような場合に使用する。

- 目的一覧画面を表示する
- 登録済みの目的を確認する
- 編集対象となる目的を選択する
- 目的達成判定を実行する目的を選択する

登録済みの目的が存在しない場合は、
空一覧を返却する。

---

## 3. エンドポイント

```http
GET /api/v1/objectives
```

---

## 4. HTTPメソッド

```http
GET
```

本APIは、
操作対象利用者に登録された
目的一覧を取得する。

登録、
更新、
無効化および
目的達成判定は行わない。

---

## 5. 認証・利用者の扱い

Phase1では、
認証機能を実装しない。

操作対象となる利用者は、
`X-User-Id`リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

指定された利用者に登録された
目的のみ取得する。

無効化された目的も、
一覧表示対象とする。

他の利用者に登録された目的は、
取得しない。

利用者IDは、
リクエストボディ、
クエリパラメータまたは
パスパラメータでは受け付けない。

---

## 6. パスパラメータ

なし。

---

## 7. クエリパラメータ

なし。

Phase1では、
一覧の検索、
並び替えおよび
ページネーションは提供しない。

登録順で一覧を取得する。

---

## 8. リクエストヘッダー

### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/objectives
Accept: application/json
X-User-Id: 1
```

---

## 9. リクエストボディ

本APIはGETメソッドのため、
リクエストボディを使用しない。

---

## 10. リクエスト項目

本APIは、
リクエストボディおよび
クエリパラメータを受け付けない。

操作対象利用者は、
`X-User-Id`リクエストヘッダーで指定する。

---

## 11. バリデーション

### 11.1 X-User-Id

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

---

### 11.2 データ取得条件

目的を取得する際は、
以下を検索条件に含める。

```text
user_id = 操作対象利用者ID
```

操作対象利用者以外の
目的は取得しない。

無効化された目的も、
一覧取得の対象とする。

---

### 11.3 並び順

一覧は、
以下の順序で返却する。

1. 有効な目的
2. 無効な目的

それぞれのグループ内では、
実施予定年月の昇順、
実施予定年月が同じ場合は
登録日時の昇順とする。

```text
enabled DESC
target_year_month ASC
created_at ASC
```

---

### 11.4 データなし

操作対象利用者に
目的が登録されていない場合は、
空配列を返却する。

データが存在しないことを理由に、
エラーは返却しない。

```json
{
  "data": []
}
```

---

## 12. 業務ルール

- 操作対象利用者に登録された目的のみ取得する。
- 他の利用者の目的は取得しない。
- 有効な目的および無効化された目的を取得対象とする。
- 一覧は実施予定年月の昇順で返却する。
- 実施予定年月が同一の場合は登録日時の昇順で返却する。
- 目的達成判定結果は一覧へ含めない。
- 目的の達成可否は本APIでは判定しない。

---

## 13. 処理フロー

```text
リクエスト受付
    ↓
リクエストID生成
    ↓
X-User-Id検証
    ↓
操作対象利用者の目的一覧取得
    ↓
一覧並び替え
    ↓
APIレスポンス形式へ変換
    ↓
200 OK返却
```

---

## 14. トランザクション境界

本APIは参照系APIであるため、
データベーストランザクションは開始しない。

目的一覧の取得のみ実行する。

---

## 15. 排他制御

本APIはデータを更新しないため、
排他制御は行わない。

取得中に他のリクエストで
目的が更新または無効化された場合は、
取得時点のデータを返却する。

---

## 16. 成功レスポンス

### 16.1 HTTPステータス

```http
200 OK
```

### 16.2 レスポンスボディ

```json
{
  "data": [
    {
      "id": "1",
      "name": "マイホーム購入",
      "targetYearMonth": "2030-03",
      "requiredAmount": 5000000,
      "enabled": true
    },
    {
      "id": "2",
      "name": "新車購入",
      "targetYearMonth": "2028-10",
      "requiredAmount": 3500000,
      "enabled": false
    }
  ]
}
```

---

## 17. レスポンス項目

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `data[].id` | string | × | 目的ID |
| `data[].name` | string | × | 目的名 |
| `data[].targetYearMonth` | string | × | 実施予定年月（YYYY-MM） |
| `data[].requiredAmount` | integer | × | 必要支出額（円） |
| `data[].enabled` | boolean | × | 利用状態（`true`：有効、`false`：無効） |

一覧表示に不要な情報は、
レスポンスへ含めない。

以下の項目は返却しない。

- `userId`
- `memo`
- `createdAt`
- `updatedAt`
- `deletedAt`

目的達成判定結果および
判定履歴は、
本APIのレスポンスへ含めない。

---

## 18. エラーレスポンス

異常時は、
API共通方針で定めた
共通エラーレスポンス形式を使用する。

---

### 18.1 利用者が存在しない場合

指定された利用者が存在しない場合は、
`USER_NOT_FOUND`
を返却する。

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "指定された利用者が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 18.2 利用者ID形式が不正な場合

`X-User-Id`が
API共通方針で定めたID形式に一致しない場合は、
`INVALID_USER_ID`
を返却する。

```json
{
  "error": {
    "code": "INVALID_USER_ID",
    "message": "利用者IDの形式が不正です。",
    "details": [
      {
        "field": "X-User-Id",
        "reason": "invalidFormat",
        "message": "利用者IDの形式を確認してください。"
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
| `200 OK` | 目的一覧の取得に成功した |
| `400 Bad Request` | 利用者IDの形式が不正である |
| `404 Not Found` | 指定された利用者が存在しない |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

---

### 19.1 404の扱い

以下の場合は、
`404 Not Found`
を返却する。

- 指定された利用者が存在しない
- 指定された利用者が論理削除されている

---

## 20. エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `USER_CONTEXT_REQUIRED` | 400 | `X-User-Id`が指定されていない | × |
| `INVALID_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `USER_NOT_FOUND` | 404 | 指定された利用者が存在しない | × |
| `INTERNAL_SERVER_ERROR` | 500 | 想定外のサーバーエラーが発生した | ○ |

エラーコードの正式な定義は、
`error-codes.md`
に従う。

---

## 21. 冪等性

本APIは、
HTTP GETを使用する。

同一条件で複数回実行しても、
サーバー側の状態は変更しない。

目的データが変更されない限り、
同一のレスポンスを返却する。

そのため、
本APIは冪等である。

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
| `enabled` | 利用状態 |
| `created_at` | 登録日時 |

本APIでは、
操作対象利用者に登録された
目的一覧を取得する。

取得対象は、
有効な目的および
無効化された目的とする。

論理削除された目的は、
取得対象としない。

---

## 23. 関連する機能要件

- `10.5 一覧表示`
  - 利用者に登録された目的一覧を表示する
- `10.4 無効化`
  - 無効化された目的も一覧で確認できる
- `11.1 判定対象`
  - 目的達成判定を実行する目的を選択できる

---

## 24. テスト観点

### 24.1 正常系

- 操作対象利用者の目的一覧を取得できること
- 有効な目的を取得できること
- 無効化された目的を取得できること
- 目的が1件の場合でも取得できること
- 複数件取得できること
- `200 OK`で返却されること

### 24.2 利用者境界

- 操作対象利用者の目的のみ取得できること
- 他利用者の目的を取得しないこと
- 他利用者の目的がレスポンスへ含まれないこと

### 24.3 一覧順

- 実施予定年月の昇順で取得できること
- 実施予定年月が同一の場合は登録日時昇順となること
- 有効な目的が無効化された目的より先に取得されること

### 24.4 利用状態

- `enabled=true`が返却されること
- `enabled=false`が返却されること
- 論理削除済みの目的は取得されないこと

### 24.5 データなし

- 登録データが存在しない場合は空配列を返却すること
- 空配列でも`200 OK`となること

### 24.6 ヘッダー

- `X-User-Id`未指定で400となること
- 利用者ID形式不正で400となること
- 存在しない利用者IDで404となること
- 論理削除済み利用者で404となること

### 24.7 レスポンス契約

- JSONフィールド名がcamelCaseであること
- 目的IDが文字列で返却されること
- 実施予定年月が`YYYY-MM`形式で返却されること
- 必要支出額が整数で返却されること
- 利用状態がbooleanで返却されること
- `userId`がレスポンスへ含まれないこと
- `memo`がレスポンスへ含まれないこと
- `createdAt`がレスポンスへ含まれないこと
- `updatedAt`がレスポンスへ含まれないこと
- `deletedAt`がレスポンスへ含まれないこと
- `data`オブジェクトで返却されること

### 24.8 異常系

- 想定外の例外で`500 Internal Server Error`となること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログで同じリクエストIDが記録されること
- SQLやスタックトレースなどの内部情報がレスポンスへ含まれないこと

---

## 25. Laravel実装方針

### 25.1 Action

HTTPリクエストを受け付け、
利用者コンテキストを取得する。

目的一覧取得UseCaseを呼び出し、
処理結果をResponderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 利用者境界の判定
- 一覧取得処理
- 並び順制御
- レスポンス生成処理

---

### 25.2 UseCase

目的一覧取得の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- Queryを呼び出して目的一覧を取得する
- 取得結果を返却する

一覧が0件の場合も、
空配列を返却する。

---

### 25.3 Query

操作対象利用者の
目的一覧を取得する。

取得条件には、
必ず操作対象利用者IDを含める。

取得例：

```php
$objectives = Objective::query()
    ->where('user_id', $userId)
    ->orderByDesc('enabled')
    ->orderBy('planned_year_month')
    ->orderBy('created_at')
    ->get();
```

取得対象は、
論理削除されていない目的とする。

無効化された目的も、
一覧取得対象とする。

以下のように、
利用者条件を指定しない取得は禁止する。

```php
Objective::all();
```

---

### 25.4 Repository

本APIでは、
Repositoryによる更新処理は行わない。

目的一覧取得のみを行うため、
Repositoryは使用しない。

---

### 25.5 トランザクション

本APIは参照系APIであるため、
データベーストランザクションは開始しない。

目的一覧の取得のみ実行する。

---

### 25.6 Responder

UseCaseから受け取った一覧を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`200 OK`とともに
`data`配列として返却する。

一覧が存在しない場合も、
空配列を返却する。

Responderは、
並び順制御や
データベース操作を行わない。

---

### 25.7 API Resource

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
    'enabled' => $this->enabled,
];
```

以下の項目は、
レスポンスへ含めない。

- `userId`
- `memo`
- `createdAt`
- `updatedAt`
- `deletedAt`

---

### 25.8 Middleware

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

### 25.9 例外変換

LaravelおよびPostgreSQLの内部例外は、
そのままAPIレスポンスへ公開しない。

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

SQL、
スタックトレースおよび
内部例外メッセージは、
APIレスポンスへ含めない。

---

## 26. React・TypeScriptでの利用

レスポンス型は、
以下とする。

```ts
export type Objective = {
  id: string;
  name: string;
  plannedYearMonth: string | null;
  requiredExpense: number;
  enabled: boolean;
};

export type GetObjectivesResponse = {
  data: Objective[];
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetObjectivesResponse>(
    '/api/v1/objectives',
  );
```

取得した一覧は、
目的一覧画面へ表示する。

一覧の各行から、
以下の画面へ遷移できる。

- 目的詳細画面
- 目的編集画面
- 目的達成判定画面

---

### 26.1 実施予定年月の扱い

`plannedYearMonth`は、
API共通方針に従い
`YYYY-MM`形式で返却される。

設定されていない場合は、
`null`を返却する。

```ts
if (objective.plannedYearMonth !== null) {
  console.log(objective.plannedYearMonth);
}
```

JavaScriptの`Date`へ
変換せず、
年月を表す文字列として扱う。

---

### 26.2 必要支出額の扱い

`requiredExpense`は、
日本円の整数値として扱う。

画面表示時は、
必要に応じて桁区切りを行う。

```ts
const formatted =
  new Intl.NumberFormat('ja-JP').format(
    objective.requiredExpense,
  );
```

フロントエンド側で、
小数点処理は行わない。

---

### 26.3 利用状態の扱い

`enabled`を利用して、
利用状態を表示する。

表示例

| enabled | 表示 |
|:---:|---|
| `true` | 有効 |
| `false` | 無効 |

無効な目的は、
グレーアウト表示または
「無効」バッジを表示してもよい。

---

### 26.4 データなしの扱い

目的が登録されていない場合は、
以下のレスポンスが返却される。

```json
{
  "data": []
}
```

フロントエンドでは、
エラー画面ではなく、
「登録された目的はありません。」
などのメッセージを表示する。

---

### 26.5 ローディング表示

一覧取得中は、
ローディング表示を行う。

取得完了後に、
一覧または空一覧メッセージを表示する。

---

### 26.6 エラー表示

エラーコードごとの基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `INVALID_USER_ID` | 共通エラー表示 |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示 |

---

## 27. 設計上の補足

### 27.1 一覧取得APIである理由

本APIは、
操作対象利用者に登録された
目的一覧を取得する責務のみを持つ。

目的の登録、
更新、
無効化および
目的達成判定は、
それぞれ専用APIで実施する。

---

### 27.2 無効な目的も取得する理由

無効化された目的も、
過去に登録した目的として
一覧表示できるようにする。

これにより、
利用者は過去の目的を確認できる。

無効な目的は、
編集や再利用の判断材料として扱う。

---

### 27.3 利用状態を返却する理由

一覧画面で、
有効・無効を判別できるようにするため、
`enabled`を返却する。

論理削除日時である
`deletedAt`は、
データベース実装の詳細であるため、
APIレスポンスへ含めない。

---

### 27.4 実施予定年月を返却する理由

一覧画面で、
各目的の実施予定時期を確認できるようにする。

実施予定年月が未設定の場合は、
`null`を返却する。

---

### 27.5 必要支出額を返却する理由

一覧画面で、
目的達成に必要な金額を確認できるようにする。

詳細画面へ遷移しなくても、
概要を把握できるようにする。

---

### 27.6 メモを返却しない理由

メモは、
一覧画面では使用しない。

レスポンスサイズを抑えるため、
本APIでは返却しない。

必要な場合は、
目的詳細取得APIを利用する。

---

### 27.7 目的達成判定結果を返却しない理由

本APIの責務は、
目的一覧の取得である。

目的達成判定結果は、
目的達成判定APIおよび
判定履歴APIで取得する。

責務を分離することで、
一覧取得APIを単純に保つ。

---

### 27.8 ページネーションを採用しない理由

Phase1では、
管理する目的数は多くないことを前提とする。

そのため、
ページネーションは採用しない。

将来的に件数が増加した場合は、
API共通方針に従って
ページネーション対応を検討する。

---

## 28. 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)

---# OBJ-002 目的登録

## 1. 概要

操作対象となる利用者の
目的を登録する。

目的には、
目的名、
実施予定年月、
必要支出額および
メモを登録できる。

登録した目的は、
目的達成判定の対象となる。

登録直後の利用状態は、
有効とする。

---

## 2. ユースケース

利用者は、
新しい目的を登録する。

例えば、
以下のような場合に使用する。

- マイホーム購入資金を登録する
- 車の購入資金を登録する
- 結婚資金を登録する
- 教育資金を登録する
- 老後資金を登録する

登録完了後は、
目的一覧および
目的達成判定で利用できる。

---

## 3. エンドポイント

```http
POST /api/v1/objectives
```

---

## 4. HTTPメソッド

```http
POST
```

本APIは、
新しい目的を登録する。

一覧取得、
更新、
無効化および
目的達成判定は行わない。

---

## 5. 認証・利用者の扱い

Phase1では、
認証機能を実装しない。

操作対象となる利用者は、
`X-User-Id`
リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

登録する目的は、
指定された利用者へ紐付ける。

他の利用者の目的として
登録することはできない。

利用者IDは、
リクエストボディでは受け付けない。

利用者IDは、
ミドルウェアで設定された
利用者コンテキストから取得する。

登録直後の利用状態は、
`enabled = true`
とする。

利用状態は、
リクエストから指定できない。

---

## 6. パスパラメータ

なし。

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
POST /api/v1/objectives
Content-Type: application/json
Accept: application/json
X-User-Id: 1
```

---

## 9. リクエストボディ

```json
{
  "name": "マイホーム購入",
  "plannedYearMonth": "2030-03",
  "requiredExpense": 5000000,
  "memo": "頭金として利用する"
}
```

---

## 10. リクエスト項目

| 項目 | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `name` | string | ○ | 目的名 |
| `plannedYearMonth` | string | × | 実施予定年月（YYYY-MM） |
| `requiredExpense` | integer | ○ | 必要支出額（円） |
| `memo` | string | × | メモ |

---

### 10.1 name

目的名を指定する。

利用者が目的を識別できる名称とする。

例

```text
マイホーム購入
老後資金
教育資金
```

---

### 10.2 plannedYearMonth

実施予定年月を指定する。

形式は、
`YYYY-MM`とする。

未定の場合は、
`null`を指定できる。

例

```text
2030-03
```

---

### 10.3 requiredExpense

目的達成に必要な支出額を指定する。

日本円の整数値とする。

例

```text
5000000
```

---

### 10.4 memo

目的に関する補足情報を指定する。

未入力の場合は、
`null`を指定できる。

---

## 11. バリデーション

### 11.1 X-User-Id

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

---

### 11.2 name

以下を検証する。

- 指定されていること
- 文字列であること
- `null`でないこと
- 1文字以上100文字以下であること
- 前後の空白を除去した結果が空文字でないこと

---

### 11.3 plannedYearMonth

以下を検証する。

- 指定された場合は文字列であること
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

### 11.4 requiredExpense

以下を検証する。

- 指定されていること
- 整数であること
- `null`でないこと
- 0以上であること
- int型の範囲内であること

小数および負数は受け付けない。

---

### 11.5 memo

以下を検証する。

- 指定された場合は文字列であること
- `null`を許可すること
- 最大文字数以内であること

空文字列は、
`null`へ正規化してよい。

---

### 11.6 更新不可項目

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

### 11.7 未定義項目

定義されていない項目を
リクエストへ含めてはならない。

未定義項目が指定された場合は、
バリデーションエラーとする。

---

## 12. 業務ルール

- 操作対象利用者の目的のみ取得する。
- 他の利用者の目的は取得しない。
- 有効な目的および無効化された目的を取得対象とする。
- 論理削除された目的は取得しない。
- 目的達成判定結果は取得しない。
- 判定履歴は取得しない。
- 実施予定年月が未設定の場合は`null`を返却する。

---

## 13. 処理フロー

```text
リクエスト受付
    ↓
リクエストID生成
    ↓
X-User-Id検証
    ↓
目的ID検証
    ↓
目的取得
    ↓
利用者境界確認
    ↓
APIレスポンス形式へ変換
    ↓
200 OK返却
```

---

## 14. トランザクション境界

本APIは参照系APIであるため、
データベーストランザクションは開始しない。

目的情報の取得のみ実行する。

---

## 15. 排他制御

本APIはデータを更新しないため、
排他制御は行わない。

取得中に他のリクエストで
目的が更新または無効化された場合は、
取得時点のデータを返却する。

---

## 16. 成功レスポンス

### 16.1 HTTPステータス

```http
200 OK
```

### 16.2 レスポンスボディ

```json
{
  "data": {
    "id": "1",
    "name": "マイホーム購入",
    "plannedYearMonth": "2030-03",
    "requiredExpense": 5000000,
    "memo": "頭金として利用する",
    "enabled": true
  }
}
```

---

## 17. レスポンス項目

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `data.id` | string | × | 目的ID |
| `data.name` | string | × | 目的名 |
| `data.plannedYearMonth` | string | ○ | 実施予定年月（YYYY-MM） |
| `data.requiredExpense` | integer | × | 必要支出額（円） |
| `data.memo` | string | ○ | メモ |
| `data.enabled` | boolean | × | 利用状態（`true`：有効、`false`：無効） |

以下の項目は、
レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`
- `deletedAt`

本APIでは、
目的達成判定結果および
判定履歴は返却しない。

---

## 18. エラーレスポンス

異常時は、
API共通方針で定めた
共通エラーレスポンス形式を使用する。

---

### 18.1 目的名が重複する場合

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

### 18.2 バリデーションエラー

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
        "message": "必要支出額を入力してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 18.3 利用者が存在しない場合

指定された利用者が存在しない場合は、
`USER_NOT_FOUND`
を返却する。

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "指定された利用者が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

## 19. HTTPステータス

| HTTPステータス | 条件 |
|---:|---|
| `201 Created` | 目的登録に成功した |
| `400 Bad Request` | 利用者IDの形式が不正である |
| `404 Not Found` | 指定された利用者が存在しない |
| `409 Conflict` | 同一利用者で目的名が重複している |
| `422 Unprocessable Entity` | 入力値が不正である |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

---

### 19.1 404の扱い

以下の場合は、
`404 Not Found`
を返却する。

- 指定された利用者が存在しない
- 指定された利用者が論理削除されている

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

## 20. エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `USER_CONTEXT_REQUIRED` | 400 | `X-User-Id`が指定されていない | × |
| `INVALID_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `USER_NOT_FOUND` | 404 | 指定された利用者が存在しない | × |
| `OBJECTIVE_ALREADY_EXISTS` | 409 | 同一利用者で目的名が重複している | × |
| `VALIDATION_ERROR` | 422 | 入力値が不正である | × |
| `INTERNAL_SERVER_ERROR` | 500 | 想定外のサーバーエラーが発生した | ○ |

エラーコードの正式な定義は、
`error-codes.md`
に従う。

---

## 21. 冪等性

本APIは、
HTTP POSTを使用する。

POSTは新しいリソースを生成するため、
HTTP仕様上、
冪等ではない。

同一リクエストを複数回送信した場合は、
複数の登録処理が実行される可能性がある。

ただし、
同一利用者に同じ目的名を登録することはできないため、
目的名が重複した場合は、
`409 Conflict`
を返却する。

Phase1では、
冪等性キー（Idempotency-Key）は採用しない。

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
| `created_at` | 登録日時 |
| `updated_at` | 更新日時 |

本APIでは、
新しい目的を登録する。

登録時の初期値は、
以下とする。

- `enabled = true`
- `created_at = 現在日時`
- `updated_at = 現在日時`

`deleted_at`は、
登録時には設定しない。

---

## 23. 関連する機能要件

- `10.2 登録`
  - 新しい目的を登録できる
- `10.9 業務ルール`
  - 目的名は利用者ごとに一意とする
  - 実施予定年月は任意入力とする
  - 必要支出額は必須項目とする

---

## 24. テスト観点

### 24.1 正常系

- 新しい目的を登録できること
- 実施予定年月ありで登録できること
- 実施予定年月なしで登録できること
- メモありで登録できること
- メモなしで登録できること
- `201 Created`で返却されること

### 24.2 利用者境界

- 操作対象利用者へ登録されること
- 他利用者へ登録されないこと
- `userId`をリクエストから指定できないこと

### 24.3 目的名

- 100文字以内で登録できること
- 同一利用者で重複登録すると409となること
- 他利用者では同じ目的名を登録できること
- 空文字で422となること
- 最大文字数超過で422となること

### 24.4 実施予定年月

- `YYYY-MM`形式で登録できること
- `null`で登録できること
- 不正な形式で422となること
- 存在しない年月で422となること

### 24.5 必要支出額

- 正常な整数値で登録できること
- 最小値で登録できること
- 最大値で登録できること
- 負数で422となること
- 小数で422となること
- 最大値超過で422となること

### 24.6 メモ

- メモを登録できること
- `null`で登録できること
- 空文字が`null`へ正規化されること
- 最大文字数以内で登録できること
- 最大文字数超過で422となること

### 24.7 初期値

- `enabled=true`で登録されること
- `created_at`が設定されること
- `updated_at`が設定されること
- `deleted_at`が設定されないこと

### 24.8 リクエストボディ

- 必須項目不足で422となること
- 未定義項目指定で422となること
- `id`指定で422となること
- `userId`指定で422となること
- `enabled`指定で422となること
- `createdAt`指定で422となること
- `updatedAt`指定で422となること
- `deletedAt`指定で422となること

### 24.9 ヘッダー

- `X-User-Id`未指定で400となること
- 利用者ID形式不正で400となること
- 存在しない利用者IDで404となること
- 論理削除済み利用者で404となること

### 24.10 トランザクション

- 登録途中で例外が発生した場合はロールバックされること
- 重複登録時は登録されないこと

### 24.11 同時登録

- 同時登録時にデータ整合性が維持されること
- UNIQUE制約違反時は409となること
- 想定外例外へ変換されないこと

### 24.12 レスポンス契約

- JSONフィールド名がcamelCaseであること
- 目的IDが文字列で返却されること
- 実施予定年月が`YYYY-MM`形式または`null`で返却されること
- 必要支出額が整数で返却されること
- 利用状態がbooleanで返却されること
- `memo`が`null`を許容すること
- `userId`がレスポンスへ含まれないこと
- `createdAt`がレスポンスへ含まれないこと
- `updatedAt`がレスポンスへ含まれないこと
- `deletedAt`がレスポンスへ含まれないこと
- `data`オブジェクトで返却されること

### 24.13 異常系

- 想定外の例外で`500 Internal Server Error`となること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログで同じリクエストIDが記録されること
- SQLやスタックトレースなどの内部情報がレスポンスへ含まれないこと

---

## 25. Laravel実装方針

### 25.1 Action

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

### 25.2 UseCase

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

### 25.4 Query

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

### 25.5 Repository

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

### 25.6 トランザクション

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

### 25.7 Responder

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

### 25.8 API Resource

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

### 25.9 Middleware

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

### 25.10 例外変換

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

## 26. React・TypeScriptでの利用

登録リクエスト型は、
以下とする。

```ts
export type CreateObjectiveRequest = {
  name: string;
  plannedYearMonth: string | null;
  requiredExpense: number;
  memo: string | null;
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

export type CreateObjectiveResponse = {
  data: Objective;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.post<
    CreateObjectiveResponse
  >(
    '/api/v1/objectives',
    {
      name: 'マイホーム購入',
      plannedYearMonth: '2030-03',
      requiredExpense: 5000000,
      memo: '頭金として利用する',
    },
  );
```

登録成功後は、
目的一覧画面または
目的詳細画面へ遷移する。

---

### 26.1 nameの扱い

`name`は、
入力必須項目とする。

入力前後の空白は除去して送信する。

```ts
const request = {
  ...form,
  name: form.name.trim(),
};
```

---

### 26.2 plannedYearMonthの扱い

`plannedYearMonth`は、
`YYYY-MM`形式で送信する。

未設定の場合は、
`null`を送信する。

```ts
plannedYearMonth: null
```

JavaScriptの`Date`オブジェクトへ
変換せず、
年月を表す文字列として扱う。

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

未入力の場合は、
`null`を送信する。

```ts
memo:
  form.memo === ''
    ? null
    : form.memo;
```

---

### 26.5 enabledの扱い

`enabled`は、
レスポンスでのみ取得する。

登録時は、
フロントエンドから送信しない。

登録直後は、
`true`で返却される。

---

### 26.6 ローディング表示

登録処理中は、
送信ボタンを非活性化する。

二重送信を防止するため、
登録完了またはエラーになるまで
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
| `INVALID_USER_ID` | 共通エラー表示 |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示 |

---

## 27. 設計上の補足

### 27.1 enabledをリクエストで受け付けない理由

登録直後の目的は、
必ず有効状態で登録する。

そのため、
利用状態はクライアントから指定できない。

無効化は、
目的無効化APIでのみ実施する。

---

### 27.2 userIdを受け付けない理由

操作対象利用者は、
利用者コンテキストから決定する。

他利用者への登録を防止するため、
`userId`はリクエストで受け付けない。

---

### 27.3 実施予定年月を任意入力とする理由

目的によっては、
具体的な実施時期が未定の場合がある。

そのため、
`plannedYearMonth`は
任意入力とする。

---

### 27.4 必要支出額を必須とする理由

目的達成判定では、
必要支出額を利用する。

そのため、
実施予定年月が未設定であっても、
必要支出額は必須とする。

---

### 27.5 登録時に判定を実行しない理由

目的登録の責務は、
目的情報の登録である。

目的達成判定は、
専用APIで実行する。

登録時に判定を実行すると、
責務が複雑になるため採用しない。

---

### 27.6 判定履歴を登録しない理由

判定履歴は、
目的達成判定APIの実行時のみ作成する。

目的登録だけでは、
判定履歴を作成しない。

---

### 27.7 メモを登録対象とする理由

メモには、
目的に関する補足情報を保存できる。

目的達成判定では利用しないため、
参照情報として扱う。

---

### 27.8 レスポンスへ登録結果を返却する理由

登録完了後に、
画面遷移や状態更新を容易にするため、
登録した目的を返却する。

フロントエンドは、
再度詳細取得APIを呼び出す必要がない。

---

### 27.9 冪等性キーを採用しない理由

Phase1では、
個人利用を前提とする。

二重送信については、
送信ボタンの非活性化により防止する。

そのため、
Idempotency-Keyは採用しない。

---

## 28. 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)