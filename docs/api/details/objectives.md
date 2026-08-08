# 目的API

## 1. 概要

本書は、
目的管理に関するAPIの詳細設計を定義する。

目的管理では、
利用者が達成したい目的を登録し、
編集、
無効化および参照できる。

登録された目的は、
目的達成判定で利用する。

本書では、
各APIについて以下を定義する。

- エンドポイント
- リクエスト
- レスポンス
- バリデーション
- 業務ルール
- エラー処理
- Laravel実装方針
- React・TypeScriptでの利用

---

## 2. 対象API

| API ID | API名 |
|---|---|
| OBJ-001 | 目的一覧取得 |
| OBJ-002 | 目的登録 |
| OBJ-003 | 目的詳細取得 |
| OBJ-004 | 目的更新 |
| OBJ-005 | 目的無効化 |
| OBJ-006 | 目的達成判定 |

---

# OBJ-001 目的一覧取得

## 1. 概要

操作対象となるデモ利用者に登録された
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

操作対象となるデモ利用者は、
`X-Demo-User-Id`リクエストヘッダーで指定する。

```http
X-Demo-User-Id: 1
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
| `X-Demo-User-Id` | ○ | 操作対象となるデモ利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/objectives
Accept: application/json
X-Demo-User-Id: 1
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
`X-Demo-User-Id`リクエストヘッダーで指定する。

---

## 11. バリデーション

### 11.1 X-Demo-User-Id

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
X-Demo-User-Id検証
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
`DEMO_USER_NOT_FOUND`
を返却する。

```json
{
  "error": {
    "code": "DEMO_USER_NOT_FOUND",
    "message": "指定された利用者が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 18.2 利用者ID形式が不正な場合

`X-Demo-User-Id`が
API共通方針で定めたID形式に一致しない場合は、
`INVALID_DEMO_USER_ID`
を返却する。

```json
{
  "error": {
    "code": "INVALID_DEMO_USER_ID",
    "message": "利用者IDの形式が不正です。",
    "details": [
      {
        "field": "X-Demo-User-Id",
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
| `DEMO_USER_CONTEXT_REQUIRED` | 400 | `X-Demo-User-Id`が指定されていない | × |
| `INVALID_DEMO_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `DEMO_USER_NOT_FOUND` | 404 | 指定された利用者が存在しない | × |
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

- `X-Demo-User-Id`未指定で400となること
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
デモ利用者コンテキストを取得する。

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
    ->where('user_id', $demoUserId)
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

- デモ利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

デモ利用者コンテキスト設定ミドルウェアでは、
`X-Demo-User-Id`を検証し、
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
| 利用者未指定 | `DEMO_USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_DEMO_USER_ID` |
| 利用者不存在 | `DEMO_USER_NOT_FOUND` |
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
| `INVALID_DEMO_USER_ID` | 共通エラー表示 |
| `DEMO_USER_NOT_FOUND` | デモ利用者選択画面へ戻す |
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

操作対象となるデモ利用者の
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

操作対象となるデモ利用者は、
`X-Demo-User-Id`
リクエストヘッダーで指定する。

```http
X-Demo-User-Id: 1
```

登録する目的は、
指定された利用者へ紐付ける。

他の利用者の目的として
登録することはできない。

利用者IDは、
リクエストボディでは受け付けない。

利用者IDは、
ミドルウェアで設定された
デモ利用者コンテキストから取得する。

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
| `X-Demo-User-Id` | ○ | 操作対象となるデモ利用者ID |
| `Content-Type` | ○ | `application/json`を指定する |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例

```http
POST /api/v1/objectives
Content-Type: application/json
Accept: application/json
X-Demo-User-Id: 1
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

### 11.1 X-Demo-User-Id

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
X-Demo-User-Id検証
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
`DEMO_USER_NOT_FOUND`
を返却する。

```json
{
  "error": {
    "code": "DEMO_USER_NOT_FOUND",
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
| `DEMO_USER_CONTEXT_REQUIRED` | 400 | `X-Demo-User-Id`が指定されていない | × |
| `INVALID_DEMO_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `DEMO_USER_NOT_FOUND` | 404 | 指定された利用者が存在しない | × |
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

- `X-Demo-User-Id`未指定で400となること
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
デモ利用者コンテキストを取得する。

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
    ->where('user_id', $demoUserId)
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
    'user_id' => $demoUserId,
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
    function () use ($demoUserId, $input): Objective {
        if ($this->query->existsByUserAndName(
            $demoUserId,
            $input->name,
        )) {
            throw new ObjectiveAlreadyExistsException();
        }

        return $this->repository->create(
            $demoUserId,
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

- デモ利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

デモ利用者コンテキスト設定ミドルウェアでは、
`X-Demo-User-Id`を検証し、
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
| 利用者未指定 | `DEMO_USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_DEMO_USER_ID` |
| 利用者不存在 | `DEMO_USER_NOT_FOUND` |
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
| `INVALID_DEMO_USER_ID` | 共通エラー表示 |
| `DEMO_USER_NOT_FOUND` | デモ利用者選択画面へ戻す |
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
デモ利用者コンテキストから決定する。

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

---

# OBJ-003 目的詳細取得

## 1. 概要

操作対象となるデモ利用者の
指定した目的の詳細情報を取得する。

取得した目的は、
目的編集画面および
目的達成判定画面で利用する。

---

## 2. ユースケース

利用者は、
登録済みの目的の詳細を確認する。

例えば、
以下のような場合に使用する。

- 目的詳細画面を表示する
- 目的編集画面を表示する
- 目的達成判定前に内容を確認する

無効化された目的も、
詳細を取得できる。

---

## 3. エンドポイント

```http
GET /api/v1/objectives/{objectiveId}
```

---

## 4. HTTPメソッド

```http
GET
```

本APIは、
指定した目的の詳細情報を取得する。

登録、
更新、
無効化および
目的達成判定は行わない。

---

## 5. 認証・利用者の扱い

Phase1では、
認証機能を実装しない。

操作対象となるデモ利用者は、
`X-Demo-User-Id`
リクエストヘッダーで指定する。

```http
X-Demo-User-Id: 1
```

指定された利用者に登録された
目的のみ取得する。

無効化された目的も、
取得対象とする。

論理削除された目的は、
取得対象としない。

他の利用者の目的は、
取得できない。

利用者IDは、
リクエストボディ、
クエリパラメータまたは
パスパラメータでは受け付けない。

---

## 6. パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `objectiveId` | string | ○ | 取得対象の目的ID |

リクエスト例

```http
GET /api/v1/objectives/1
```

---

## 7. クエリパラメータ

なし。

---

## 8. リクエストヘッダー

### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-Demo-User-Id` | ○ | 操作対象となるデモ利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例

```http
GET /api/v1/objectives/1
Accept: application/json
X-Demo-User-Id: 1
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

取得対象の目的は、
パスパラメータで指定する。

操作対象利用者は、
`X-Demo-User-Id`
リクエストヘッダーで指定する。

---

## 11. バリデーション

### 11.1 X-Demo-User-Id

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

---

### 11.3 利用者境界

目的を取得する際は、
以下を検索条件に含める。

```text
id = objectiveId
AND user_id = 操作対象利用者ID
```

他の利用者の目的は、
取得できない。

---

### 11.4 無効化された目的

無効化された目的も、
詳細取得の対象とする。

利用状態は、
`enabled`で判定する。

---

### 11.5 リクエスト項目

本APIでは、
リクエストボディおよび
クエリパラメータを受け付けない。

未定義のクエリパラメータが指定された場合は、
バリデーションエラーとする。

---

## 12. 業務ルール

- 操作対象利用者に登録された目的のみ取得する。
- 他の利用者の目的は取得しない。
- 有効な目的および無効化された目的を取得対象とする。
- 論理削除された目的は取得対象としない。
- 実施予定年月が未設定の場合は`null`を返却する。
- 目的達成判定結果は取得しない。
- 判定履歴は取得しない。

---

## 13. 処理フロー

```text
リクエスト受付
    ↓
リクエストID生成
    ↓
X-Demo-User-Id検証
    ↓
目的ID検証
    ↓
目的取得
    ↓
利用者境界確認
    ↓
API Resourceへ変換
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

---## 18. エラーレスポンス

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

### 18.2 利用者が存在しない場合

指定された利用者が存在しない場合は、
`DEMO_USER_NOT_FOUND`
を返却する。

```json
{
  "error": {
    "code": "DEMO_USER_NOT_FOUND",
    "message": "指定された利用者が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 18.3 利用者ID形式が不正な場合

`X-Demo-User-Id`が
API共通方針で定めたID形式に一致しない場合は、
`INVALID_DEMO_USER_ID`
を返却する。

```json
{
  "error": {
    "code": "INVALID_DEMO_USER_ID",
    "message": "利用者IDの形式が不正です。",
    "details": [
      {
        "field": "X-Demo-User-Id",
        "reason": "invalidFormat",
        "message": "利用者IDの形式を確認してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 18.4 目的ID形式が不正な場合

`objectiveId`が
API共通方針で定めたID形式に一致しない場合は、
`INVALID_OBJECTIVE_ID`
を返却する。

```json
{
  "error": {
    "code": "INVALID_OBJECTIVE_ID",
    "message": "目的IDの形式が不正です。",
    "details": [
      {
        "field": "objectiveId",
        "reason": "invalidFormat",
        "message": "目的IDの形式を確認してください。"
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
| `200 OK` | 目的詳細の取得に成功した |
| `400 Bad Request` | 利用者IDまたは目的IDの形式が不正である |
| `404 Not Found` | 指定した利用者または目的が存在しない |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

---

### 19.1 404の扱い

以下の場合は、
`404 Not Found`
を返却する。

- 指定した利用者が存在しない
- 指定した利用者が論理削除されている
- 指定した目的が存在しない
- 指定した目的が論理削除されている
- 指定した目的が操作対象利用者に属していない

---

## 20. エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `DEMO_USER_CONTEXT_REQUIRED` | 400 | `X-Demo-User-Id`が指定されていない | × |
| `INVALID_DEMO_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `INVALID_OBJECTIVE_ID` | 400 | 目的IDの形式が不正である | × |
| `DEMO_USER_NOT_FOUND` | 404 | 指定された利用者が存在しない | × |
| `OBJECTIVE_NOT_FOUND` | 404 | 指定した目的が存在しない、または操作対象利用者に属していない | × |
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

目的情報が変更されない限り、
同一のレスポンスを返却する。

そのため、
本APIは冪等である。

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

本APIでは、
操作対象利用者に登録された
目的を1件取得する。

取得対象は、
有効な目的および
無効化された目的とする。

論理削除された目的は、
取得対象としない。

---

## 23. 関連する機能要件

- `10.6 詳細表示`
  - 指定した目的の詳細を表示できる
- `10.9 業務ルール`
  - 実施予定年月は任意入力とする
  - 必要支出額は必須項目とする
  - 無効化した目的は新規判定対象としない

---

## 24. テスト観点

### 24.1 正常系

- 指定した目的を取得できること
- 実施予定年月ありの目的を取得できること
- 実施予定年月未設定（`null`）の目的を取得できること
- メモありの目的を取得できること
- メモ未設定（`null`）の目的を取得できること
- 有効な目的を取得できること
- 無効化された目的を取得できること
- `200 OK`で返却されること

---

### 24.2 利用者境界

- 操作対象利用者の目的のみ取得できること
- 他利用者の目的を取得できないこと
- 他利用者の目的IDを指定した場合は`404 Not Found`となること

---

### 24.3 利用状態

- `enabled=true`が返却されること
- `enabled=false`が返却されること
- 論理削除済みの目的は取得できないこと

---

### 24.4 パスパラメータ

- `objectiveId`未指定でエラーとなること
- `objectiveId`形式不正で`400 Bad Request`となること
- 存在しない目的IDで`404 Not Found`となること

---

### 24.5 ヘッダー

- `X-Demo-User-Id`未指定で`400 Bad Request`となること
- 利用者ID形式不正で`400 Bad Request`となること
- 存在しない利用者IDで`404 Not Found`となること
- 論理削除済み利用者で`404 Not Found`となること

---

### 24.6 レスポンス契約

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

### 24.7 異常系

- 想定外の例外で`500 Internal Server Error`となること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログで同じリクエストIDが記録されること
- SQLおよびスタックトレースなどの内部情報がレスポンスへ含まれないこと

---

## 25. Laravel実装方針

### 25.1 Action

HTTPリクエストを受け付け、
目的IDおよび
デモ利用者コンテキストを取得する。

目的詳細取得UseCaseを呼び出し、
取得結果をResponderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 目的IDの形式検証
- 利用者境界の判定
- 目的取得処理
- 論理削除状態の判定
- レスポンス生成処理

---

### 25.2 UseCase

目的詳細取得の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 目的IDを受け取る
- Queryを呼び出して目的を取得する
- 取得結果を返却する

指定された目的が存在しない場合、
論理削除されている場合、
または操作対象利用者に帰属しない場合は、
`OBJECTIVE_NOT_FOUND`として扱う。

無効化された目的は、
詳細取得の対象とする。

---

### 25.3 パスパラメータ検証

`objectiveId`の形式検証は、
Route Model Bindingへ全面的に依存せず、
API共通方針に従って実施する。

以下を検証する。

- 必須であること
- API共通方針で定めたID形式であること

形式が不正な場合は、
`INVALID_OBJECTIVE_ID`として扱う。

目的の存在確認および
利用者境界の確認は、
Queryで行う。

---

### 25.4 Query

操作対象利用者に帰属する
目的を1件取得する。

取得条件には、
必ず目的IDおよび
操作対象利用者IDを含める。

取得例：

```php
$objective = Objective::query()
    ->where('id', $objectiveId)
    ->where('user_id', $demoUserId)
    ->first();
```

LaravelのSoftDeletesを利用する場合、
通常のEloquentクエリによって
論理削除済みの目的を取得対象から除外する。

`enabled = false` の目的は
論理削除済みではないため、
通常どおり取得対象とする。

以下のように、
目的IDだけで取得してはならない。

```php
Objective::find($objectiveId);
```

取得できなかった場合は、
原因を外部へ区別して公開せず、
`OBJECTIVE_NOT_FOUND` として扱う。

### 25.5 Repository

本APIは参照系APIであるため、
目的の登録、
更新および無効化は行わない。

永続化処理を必要としないため、
Repositoryは使用しない。

目的の取得は、
参照専用のQueryが担当する。

### 25.6 トランザクション

本APIは参照処理のみであるため、
明示的なデータベーストランザクションは使用しない。

目的情報の取得によって、
データベースの状態を変更しない。

Phase1では、
取得処理中の厳密なスナップショット分離は保証しない。

### 25.7 Responder

UseCaseから受け取った目的を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`200 OK` とともに
目的詳細を `data` オブジェクトで返却する。

目的が取得できない場合は、
`404 Not Found` および
`OBJECTIVE_NOT_FOUND` を返却する。

Responderは、
以下を行わない。

- 目的の取得
- 利用者境界の判定
- 利用状態の判定
- 業務ルールの判定

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

`planned_year_month` が未設定の場合は、
`plannedYearMonth` へ `null` を設定する。

`memo` が未設定の場合は、
`memo` へ `null` を設定する。

以下の項目は、
レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`
- `deletedAt`

目的達成判定結果および
判定履歴も、
本APIでは返却しない。

### 25.9 Middleware

以下の共通ミドルウェアを適用する。

- デモ利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

デモ利用者コンテキスト設定ミドルウェアでは、
`X-Demo-User-Id` を検証し、
操作対象利用者を特定する。

Action以降では、
検証済みの利用者コンテキストを使用する。

### 25.10 例外変換

LaravelおよびPostgreSQLの内部例外は、
そのままAPIレスポンスへ公開しない。

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `DEMO_USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_DEMO_USER_ID` |
| 利用者不存在 | `DEMO_USER_NOT_FOUND` |
| 目的ID形式不正 | `INVALID_OBJECTIVE_ID` |
| 目的不存在 | `OBJECTIVE_NOT_FOUND` |
| 論理削除済み目的 | `OBJECTIVE_NOT_FOUND` |
| 利用者境界外の目的 | `OBJECTIVE_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

目的が存在しない場合、
論理削除されている場合、
または他利用者に帰属する場合を、
外部レスポンスでは区別しない。

SQL、
スタックトレースおよび
内部例外メッセージは、
APIレスポンスへ含めない。

ログには、
調査に必要な範囲で
以下を記録する。

- 操作対象利用者ID
- 指定された目的ID
- 独自エラーコード
- リクエストID

---

## 26. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type GetObjectiveDetailParams = {
  objectiveId: string;
};
```

目的詳細の型は、
以下とする。

```typescript
export type ObjectiveDetail = {
  id: string;
  name: string;
  plannedYearMonth: string | null;
  requiredExpense: number;
  memo: string | null;
  enabled: boolean;
};
```

レスポンス型は、
以下とする。

```typescript
export type GetObjectiveDetailResponse = {
  data: ObjectiveDetail;
};
```

API呼び出し例は、
以下とする。

```typescript
const response =
  await apiClient.get<GetObjectiveDetailResponse>(
    `/api/v1/objectives/${objectiveId}`,
  );
```

取得したデータは、
以下の画面で利用する。

- 目的詳細画面
- 目的編集画面の初期表示
- 目的達成判定前の目的情報確認

### 26.1 plannedYearMonthの扱い

`plannedYearMonth` は、
`YYYY-MM`形式または
`null`で返却される。

```typescript
if (objective.plannedYearMonth === null) {
  // 実施予定年月は未設定
}
```

実施予定年月は、
日付ではなく年月を表す業務値である。

そのため、
JavaScriptの`Date`オブジェクトへ
不必要に変換せず、
文字列として扱う。

未設定の場合は、
画面上で以下のように表示する。

```text
未設定
```

### 26.2 requiredExpenseの扱い

`requiredExpense`は、
日本円の整数値として扱う。

画面表示時は、
必要に応じて桁区切りを行う。

```typescript
const formattedRequiredExpense =
  new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY',
    maximumFractionDigits: 0,
  }).format(objective.requiredExpense);
```

フロントエンド側で、
小数への変換や
独自の端数処理は行わない。

### 26.3 memoの扱い

`memo`は、
文字列または`null`で返却される。

`null`の場合は、
空欄または未入力として表示する。

```typescript
const memo = objective.memo ?? '';
```

編集画面では、
`null`を空文字へ変換して
フォーム初期値へ設定してよい。

### 26.4 enabledの扱い

`enabled`を利用して、
目的の利用状態を判定する。

| enabled | 状態 |
|---|---|
| `true` | 有効 |
| `false` | 無効 |

無効化された目的は、
詳細表示できる。

ただし、
以下の操作は許可しない。

- 目的情報の更新
- 再無効化
- 新しい目的達成判定の実行

フロントエンドでは、
`enabled = false`の場合に
編集ボタン、
無効化ボタンおよび
判定実行ボタンを非表示または非活性にする。

実際の更新・無効化・判定APIでも、
バックエンド側で利用状態を再検証する。

### 26.5 ローディング表示

目的詳細の取得中は、
ローディング表示を行う。

取得完了前に、
以前表示していた別の目的情報を
現在の目的として表示しない。

目的IDが変更された場合は、
以前の取得結果を破棄するか、
新しい取得が完了するまで
ローディング状態を表示する。

### 26.6 エラー表示

エラーコードごとの基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `INVALID_OBJECTIVE_ID` | 不正なURLとしてエラー表示する |
| `OBJECTIVE_NOT_FOUND` | 目的一覧画面へ戻し、対象が存在しないことを表示する |
| `INVALID_DEMO_USER_ID` | 共通エラー表示を行う |
| `DEMO_USER_NOT_FOUND` | デモ利用者選択画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

`OBJECTIVE_NOT_FOUND`の場合は、
以下の原因をフロントエンドで区別しない。

- 目的が存在しない
- 論理削除されている
- 他の利用者に帰属している

---

## 27. 設計上の補足

### 27.1 詳細取得APIを分離する理由

目的一覧取得APIでは、
一覧表示に必要な情報のみ返却する。

目的詳細画面および
目的編集画面では、
メモを含む目的の全項目が必要となる。

そのため、
一覧取得APIとは別に
詳細取得APIを提供する。

### 27.2 無効化された目的も取得する理由

無効化された目的も、
過去に登録した目的として
内容を確認できるようにする。

また、
保存済みの判定履歴を確認する際に、
目的名、
必要支出額および
実施予定年月を参照できるようにする。

ただし、
無効化された目的は
新しい目的達成判定の対象としない。

### 27.3 論理削除された目的を取得しない理由

論理削除済みの目的は、
通常の利用対象から除外された内部データとして扱う。

そのため、
通常の詳細取得APIでは返却しない。

論理削除済みデータを確認する
管理者向けAPIは、
Phase1では提供しない。

### 27.4 利用者境界外の目的を404とする理由

他の利用者に帰属する目的を指定した場合に、
権限エラーとして返却すると、
指定した目的が存在することを推測できる。

そのため、
利用者境界外の目的は
存在しないものとして扱い、
`OBJECTIVE_NOT_FOUND`を返却する。

### 27.5 目的達成判定結果を返却しない理由

本APIの責務は、
目的そのものの詳細情報を返却することである。

目的達成判定結果は、
目的とは別の業務データであり、
判定実行APIまたは
判定履歴取得APIで扱う。

詳細取得APIへ判定結果を含めると、
目的情報の取得と判定情報の取得が混在するため、
レスポンスへ含めない。

### 27.6 判定履歴を返却しない理由

1つの目的には、
複数回の判定履歴が存在する可能性がある。

判定履歴を目的詳細へ含めると、
レスポンス件数や
ページネーション方針が複雑になる。

そのため、
判定履歴は専用APIで取得する。

### 27.7 plannedYearMonthをnullで返却する理由

実施予定年月は、
目的登録時の任意項目である。

未設定状態を
空文字や特別な年月で表現すると、
業務上の意味が曖昧になる。

そのため、
未設定の場合は`null`を返却する。

### 27.8 enabledを返却する理由

目的詳細画面では、
目的が有効か無効かによって
利用可能な操作が異なる。

そのため、
業務上の利用状態を表す
`enabled`を返却する。

データベース実装上の
`deletedAt`は返却しない。

### 27.9 更新可否を返却しない理由

Phase1では、
更新可否は`enabled`から判断できる。

そのため、
`editable`のような
派生フラグは返却しない。

将来、
利用状態以外にも
更新可否へ影響する業務ルールが追加された場合は、
専用フラグの追加を再検討する。

### 27.10 キャッシュを採用しない理由

Phase1では、
目的の件数および
詳細取得頻度は多くない。

キャッシュを導入すると、
目的更新時および無効化時に
キャッシュ破棄処理が必要となる。

性能上の効果よりも
実装の複雑さが大きいため、
Phase1ではキャッシュを採用しない。

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

---

# OBJ-004 目的更新

## 1. 概要

操作対象となるデモ利用者に登録された
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

操作対象となるデモ利用者は、`X-Demo-User-Id` リクエストヘッダーで指定する。

```http
X-Demo-User-Id: 1
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
| `X-Demo-User-Id` | ○ | 操作対象となるデモ利用者ID |
| `Content-Type` | ○ | `application/json`を指定する |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例

```http
PATCH /api/v1/objectives/1
Content-Type: application/json
Accept: application/json
X-Demo-User-Id: 1
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

### 11.1 X-Demo-User-Id

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
| `DEMO_USER_CONTEXT_REQUIRED` | 400 | `X-Demo-User-Id`が指定されていない | × |
| `INVALID_DEMO_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `INVALID_OBJECTIVE_ID` | 400 | 目的IDの形式が不正である | × |
| `DEMO_USER_NOT_FOUND` | 404 | 指定された利用者が存在しない | × |
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
デモ利用者コンテキストを取得する。

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
    ->where('user_id', $demoUserId)
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
    ->where('user_id', $demoUserId)
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
        $demoUserId,
        $objectiveId,
        $input,
    ): Objective {
        $objective = $this->query->findByUserAndId(
            $demoUserId,
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
                $demoUserId,
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

- デモ利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

デモ利用者コンテキスト設定ミドルウェアでは、`X-Demo-User-Id` を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

### 25.11 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `DEMO_USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_DEMO_USER_ID` |
| 利用者不存在 | `DEMO_USER_NOT_FOUND` |
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
| `INVALID_DEMO_USER_ID` | 共通エラー表示 |
| `DEMO_USER_NOT_FOUND` | デモ利用者選択画面へ戻す |
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

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)

---
# OBJ-005 目的無効化

## 1. 概要

操作対象となるデモ利用者に登録された
有効な目的を無効化する。

本APIでは、
目的の利用状態のみ変更する。

目的名、
実施予定年月、
必要支出額および
メモは変更しない。

無効化された目的は、
新しい目的達成判定の対象外とする。

過去に保存された
目的達成判定履歴は変更しない。

---

## 2. ユースケース

利用者は、
今後利用しない目的を
管理対象から除外する。

例えば、
以下のような場合に使用する。

- 目的を取りやめた
- すでに達成済みとなった
- 今後判定対象としたくない

無効化後も、
目的一覧および
目的詳細では確認できる。

---

## 3. エンドポイント

```http
PATCH /api/v1/objectives/{objectiveId}/disable
```

---

## 4. HTTPメソッド

```http
PATCH
```

本APIは、
目的の利用状態のみ更新する。

目的情報の更新、
登録および
目的達成判定は行わない。

---

## 5. 認証・利用者の扱い

Phase1では、
認証機能を実装しない。

操作対象となるデモ利用者は、
`X-Demo-User-Id`
リクエストヘッダーで指定する。

```http
X-Demo-User-Id: 1
```

指定された利用者に登録された
目的のみ無効化できる。

他の利用者に登録された目的は、
無効化できない。

他の利用者に帰属する目的を指定した場合は、
対象が存在しないものとして扱う。

無効化済みの目的は、
再度無効化できない。

論理削除された目的も、
無効化対象としない。

利用者IDは、
リクエストボディ、
クエリパラメータまたは
パスパラメータでは受け付けない。

本APIでは、
利用状態のみ変更する。

目的名、
実施予定年月、
必要支出額および
メモは変更できない。

無効化後も、
目的データおよび
保存済みの目的達成判定履歴は保持する。

---

## 6. パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `objectiveId` | string | ○ | 無効化対象の目的ID |

リクエスト例

```http
PATCH /api/v1/objectives/1/disable
```

---

## 7. クエリパラメータ

なし。

---

## 8. リクエストヘッダー

### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-Demo-User-Id` | ○ | 操作対象となるデモ利用者ID |
| `Content-Type` | ○ | `application/json`を指定する |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例

```http
PATCH /api/v1/objectives/1/disable
Content-Type: application/json
Accept: application/json
X-Demo-User-Id: 1
```

---

## 9. リクエストボディ

本APIでは、
リクエストボディを使用しない。

---

## 10. リクエスト項目

本APIでは、
リクエストボディおよび
クエリパラメータを受け付けない。

無効化対象の目的は、
パスパラメータで指定する。

操作対象利用者は、
`X-Demo-User-Id`
リクエストヘッダーで指定する。

---

## 11. バリデーション

### 11.1 X-Demo-User-Id

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

### 11.3 利用者境界

目的を取得する際は、
以下を検索条件に含める。

```text
id = objectiveId
AND user_id = 操作対象利用者ID
```

他の利用者の目的は、
無効化できない。

---

### 11.4 利用状態

無効化済みの目的は、
再度無効化できない。

無効化済みの目的を指定した場合は、
`OBJECTIVE_DISABLED`
を返却する。

---

### 11.5 リクエスト項目

本APIでは、
リクエストボディおよび
クエリパラメータを受け付けない。

以下の項目は、
リクエストで受け付けない。

- `name`
- `plannedYearMonth`
- `requiredExpense`
- `memo`
- `enabled`
- `createdAt`
- `updatedAt`
- `deletedAt`

指定された場合は、
バリデーションエラーとする。

---

### 11.6 未定義項目

定義されていない項目を
リクエストへ含めてはならない。

未定義項目が指定された場合は、
バリデーションエラーとする。

---

## 12. 業務ルール

- 操作対象利用者に登録された有効な目的のみ無効化できる。
- 他の利用者の目的は無効化できない。
- 無効化済みの目的は再度無効化できない。
- 論理削除された目的は無効化できない。
- 無効化では利用状態のみ変更する。
- 目的名、実施予定年月、必要支出額およびメモは変更しない。
- 無効化した目的は新しい目的達成判定の対象としない。
- 保存済みの目的達成判定履歴は変更しない。
- 更新日時はサーバー側で更新する。

---

## 13. 処理フロー

```text
リクエスト受付
    ↓
リクエストID生成
    ↓
X-Demo-User-Id検証
    ↓
目的ID検証
    ↓
目的取得
    ↓
利用者境界確認
    ↓
利用状態確認
    ↓
目的無効化
    ↓
APIレスポンス生成
    ↓
200 OK返却
```

---

## 14. トランザクション境界

以下の処理を、
1つのデータベーストランザクション内で実行する。

- 目的取得
- 利用状態確認
- 利用状態更新

処理途中で例外が発生した場合は、
無効化処理をロールバックする。

---

## 15. 排他制御

Phase1では、
楽観ロックおよび悲観ロックは採用しない。

同時に無効化された場合でも、
最終状態は
`enabled = false`
となる。

無効化済みの目的に対して
再度無効化を実行した場合は、
`OBJECTIVE_DISABLED`
を返却する。

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
    "plannedYearMonth": "2031-03",
    "requiredExpense": 6000000,
    "memo": "物価上昇を考慮して見直し",
    "enabled": false
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
| `data.enabled` | boolean | × | 利用状態（`false`：無効） |

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

### 18.2 無効化済みの目的を指定した場合

すでに無効化されている目的を指定した場合は、
`OBJECTIVE_DISABLED`
を返却する。

```json
{
  "error": {
    "code": "OBJECTIVE_DISABLED",
    "message": "指定された目的はすでに無効化されています。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 18.3 利用者が存在しない場合

指定された利用者が存在しない場合は、
`DEMO_USER_NOT_FOUND`
を返却する。

```json
{
  "error": {
    "code": "DEMO_USER_NOT_FOUND",
    "message": "指定された利用者が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 18.4 利用者ID形式が不正な場合

`X-Demo-User-Id`が
API共通方針で定めたID形式に一致しない場合は、
`INVALID_DEMO_USER_ID`
を返却する。

```json
{
  "error": {
    "code": "INVALID_DEMO_USER_ID",
    "message": "利用者IDの形式が不正です。",
    "details": [
      {
        "field": "X-Demo-User-Id",
        "reason": "invalidFormat",
        "message": "利用者IDの形式を確認してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 18.5 目的ID形式が不正な場合

`objectiveId`が
API共通方針で定めたID形式に一致しない場合は、
`INVALID_OBJECTIVE_ID`
を返却する。

```json
{
  "error": {
    "code": "INVALID_OBJECTIVE_ID",
    "message": "目的IDの形式が不正です。",
    "details": [
      {
        "field": "objectiveId",
        "reason": "invalidFormat",
        "message": "目的IDの形式を確認してください。"
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
| `200 OK` | 目的無効化に成功した |
| `400 Bad Request` | 利用者IDまたは目的IDの形式が不正である |
| `404 Not Found` | 指定された利用者または目的が存在しない |
| `422 Unprocessable Entity` | すでに無効化されている目的である |
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

### 19.2 422の扱い

以下の場合は、
`422 Unprocessable Entity`
を返却する。

- すでに無効化されている目的を指定した

---

## 20. エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `DEMO_USER_CONTEXT_REQUIRED` | 400 | `X-Demo-User-Id`が指定されていない | × |
| `INVALID_DEMO_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `INVALID_OBJECTIVE_ID` | 400 | 目的IDの形式が不正である | × |
| `DEMO_USER_NOT_FOUND` | 404 | 指定された利用者が存在しない | × |
| `OBJECTIVE_NOT_FOUND` | 404 | 指定された目的が存在しない、または操作対象利用者に属していない | × |
| `OBJECTIVE_DISABLED` | 422 | 指定された目的がすでに無効化されている | × |
| `INTERNAL_SERVER_ERROR` | 500 | 想定外のサーバーエラーが発生した | ○ |

エラーコードの正式な定義は、
`error-codes.md`
に従う。

---

## 21. 冪等性

本APIは、
HTTP PATCHを使用する。

同じ目的に対して
無効化を複数回実行した場合でも、
目的の最終状態は
`enabled = false`
となる。

ただし、
本APIでは
すでに無効化済みの目的に対する再実行を
業務エラーとして扱う。

そのため、
2回目以降の実行では
`OBJECTIVE_DISABLED`
を返却する。

Phase1では、
ETagやIf-Matchによる
楽観ロックは採用しない。

また、
Idempotency-Keyも採用しない。

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
| `enabled` | 利用状態 |
| `updated_at` | 更新日時 |

本APIでは、
目的の利用状態のみ更新する。

更新対象は、
以下とする。

- `enabled`
- `updated_at`

以下の項目は更新しない。

- `id`
- `user_id`
- `name`
- `planned_year_month`
- `required_expense`
- `memo`
- `created_at`
- `deleted_at`

---

### 22.2 assessment_histories

目的達成判定履歴を保持する。

本APIでは、
`assessment_histories`を更新しない。

無効化後も、
保存済みの判定履歴は保持する。

---

## 23. 関連する機能要件

- `10.4 無効化`
  - 利用者は目的を無効化できる
- `10.5 一覧表示`
  - 無効化された目的も一覧表示できる
- `10.6 詳細表示`
  - 無効化された目的も詳細表示できる
- `10.9 業務ルール`
  - 無効化した目的は新規判定対象としない
  - 保存済みの判定結果は、目的を無効化しても変更しない

---

## 24. テスト観点

### 24.1 正常系

- 有効な目的を無効化できること
- `enabled`が`false`へ更新されること
- `updated_at`が更新されること
- `200 OK`で返却されること

---

### 24.2 利用者境界

- 操作対象利用者の目的のみ無効化できること
- 他利用者の目的を無効化できないこと
- 他利用者の目的IDを指定した場合は`404 Not Found`となること

---

### 24.3 利用状態

- 無効化済みの目的は再度無効化できないこと
- 無効化済みの目的を指定した場合は`422 Unprocessable Entity`となること
- 論理削除済みの目的は無効化できないこと

---

### 24.4 更新対象

- `enabled`のみ更新されること
- `name`が変更されないこと
- `planned_year_month`が変更されないこと
- `required_expense`が変更されないこと
- `memo`が変更されないこと
- `created_at`が変更されないこと
- `deleted_at`が変更されないこと

---

### 24.5 判定履歴

- 無効化後も過去の判定履歴が変更されないこと
- 無効化後も判定履歴が削除されないこと
- 無効化後は新しい目的達成判定の対象外となること

---

### 24.6 トランザクション

- 無効化途中で例外が発生した場合はロールバックされること
- 更新途中のデータが保存されないこと

---

### 24.7 同時更新

- 同時に無効化要求が実行されても整合性が保たれること
- 二重無効化時は`OBJECTIVE_DISABLED`となること

---

### 24.8 レスポンス契約

- JSONフィールド名がcamelCaseであること
- 目的IDが文字列で返却されること
- 実施予定年月が`YYYY-MM`形式または`null`で返却されること
- 必要支出額が整数で返却されること
- メモが文字列または`null`で返却されること
- `enabled`が`false`で返却されること
- `userId`がレスポンスへ含まれないこと
- `createdAt`がレスポンスへ含まれないこと
- `updatedAt`がレスポンスへ含まれないこと
- `deletedAt`がレスポンスへ含まれないこと
- `data`オブジェクトで返却されること

---

### 24.9 異常系

- 想定外の例外で`500 Internal Server Error`となること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログで同じリクエストIDが記録されること
- SQLおよびスタックトレースなどの内部情報がレスポンスへ含まれないこと

---

## 25. Laravel実装方針

### 25.1 Action

HTTPリクエストを受け付け、
目的IDおよび
デモ利用者コンテキストを取得する。

目的無効化UseCaseを呼び出し、
処理結果をResponderへ渡す。

本APIはリクエストボディを使用しない。

以下の処理は、
Actionへ直接記述しない。

- 目的IDの形式検証
- 利用者境界の判定
- 無効化対象の存在確認
- 目的の利用状態確認
- 無効化処理
- トランザクション制御
- レスポンス生成処理

---

### 25.2 UseCase

目的無効化の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 目的IDを受け取る
- 無効化対象の目的を取得する
- 現在の利用状態を確認する
- 目的を無効化する
- 無効化後の目的を返却する

指定された目的が存在しない場合、
論理削除されている場合、
または操作対象利用者に帰属しない場合は、
`OBJECTIVE_NOT_FOUND`として扱う。

すでに無効化されている場合は、
`OBJECTIVE_DISABLED`として扱う。

無効化処理は、
データベーストランザクション内で実行する。

目的を無効化しても、
保存済みの目的達成判定履歴は変更しない。

---

### 25.3 パスパラメータ検証

`objectiveId`の形式は、
API共通方針に従って検証する。

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること

形式が不正な場合は、
`INVALID_OBJECTIVE_ID`として扱う。

目的の存在確認、
利用者境界の確認および
利用状態の確認は、
UseCaseおよびQueryで行う。

---

### 25.4 Query

無効化対象となる目的を取得する。

取得条件には、
必ず目的IDおよび
操作対象利用者IDを含める。

```php
$objective = Objective::query()
    ->where('id', $objectiveId)
    ->where('user_id', $demoUserId)
    ->first();
```

以下のように、目的IDだけで取得してはならない。

```php
Objective::find($objectiveId);
```

LaravelのSoftDeletesを利用する場合、通常のEloquentクエリでは論理削除済みの目的を取得対象から除外する。

取得できなかった場合は、`OBJECTIVE_NOT_FOUND` として扱う。

取得した目的の `enabled` が `false` の場合は、`OBJECTIVE_DISABLED` として扱う。

### 25.5 Repository

目的の無効化を担当する。

本APIでは、以下のカラムを更新する。

- `enabled`
- `updated_at`

無効化時は、`enabled` を `false` へ更新する。

以下のカラムは変更しない。

- `id`
- `user_id`
- `name`
- `planned_year_month`
- `required_expense`
- `memo`
- `created_at`
- `deleted_at`

更新例：

```php
$objective->enabled = false;
$objective->save();

return $objective;
```

本APIでは、`delete()` および `forceDelete()` を使用しない。

`assessment_histories` も更新または削除しない。

### 25.6 トランザクション

以下の処理を、1つのデータベーストランザクション内で実行する。

- 無効化対象の取得
- 利用状態の確認
- 目的の無効化

実装例：

```php
$objective = DB::transaction(
    function () use (
        $demoUserId,
        $objectiveId,
    ): Objective {
        $objective = $this->query->findByUserAndId(
            $demoUserId,
            $objectiveId,
        );

        if ($objective === null) {
            throw new ObjectiveNotFoundException();
        }

        if (! $objective->enabled) {
            throw new ObjectiveDisabledException();
        }

        return $this->repository->disable(
            $objective,
        );
    },
);
```

処理途中で例外が発生した場合は、無効化処理をロールバックする。

### 25.7 同時実行時の扱い

Phase1では、明示的な行ロックおよび楽観ロックは採用しない。

同じ目的に対して複数の無効化要求が同時実行された場合は、少なくとも1件が無効化に成功する。

後続処理では、目的がすでに無効化されていることを検知し、`OBJECTIVE_DISABLED` として扱う。

ただし、単純な取得後更新では、同時実行した複数処理がともに `enabled = true` を読み取る可能性がある。

厳密に1件だけを成功させる必要がある場合は、次のような条件付き更新を利用する。

```php
$updatedCount = Objective::query()
    ->where('id', $objectiveId)
    ->where('user_id', $demoUserId)
    ->where('enabled', true)
    ->update([
        'enabled' => false,
        'updated_at' => now(),
    ]);
```

`$updatedCount` が `0` の場合は、再取得して以下を判定する。

- 対象が存在しない
- 論理削除されている
- 他利用者に帰属している
- すでに無効化されている

Phase1では、実装の単純さと同時実行時の整合性を両立できるため、条件付き更新を推奨する。

### 25.8 Responder

UseCaseから受け取った無効化結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`200 OK` とともに無効化後の目的を `data` オブジェクトで返却する。

以下の場合は、共通エラーレスポンス形式へ変換する。

- 無効化対象が存在しない
- 目的がすでに無効化されている
- 目的IDの形式が不正である
- 利用者が存在しない
- 想定外の例外が発生した

Responderは、以下を行わない。

- 利用者境界の判定
- 利用状態の判定
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

無効化成功後の `enabled` は `false` となる。

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

- デモ利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

デモ利用者コンテキスト設定ミドルウェアでは、`X-Demo-User-Id` を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

### 25.11 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `DEMO_USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_DEMO_USER_ID` |
| 利用者不存在 | `DEMO_USER_NOT_FOUND` |
| 目的ID形式不正 | `INVALID_OBJECTIVE_ID` |
| 目的不存在 | `OBJECTIVE_NOT_FOUND` |
| 論理削除済み目的 | `OBJECTIVE_NOT_FOUND` |
| 利用者境界外の目的 | `OBJECTIVE_NOT_FOUND` |
| 無効化済み目的 | `OBJECTIVE_DISABLED` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

目的が存在しない場合、論理削除されている場合、または他利用者に帰属する場合を、外部レスポンスでは区別しない。

SQL、スタックトレースおよび内部例外メッセージは、APIレスポンスへ含めない。

ログには、調査に必要な範囲で以下を記録する。

- 操作対象利用者ID
- 目的ID
- 独自エラーコード
- リクエストID

目的名、必要支出額およびメモなどの業務データを不要にエラーログへ出力しない。

---

## 設計上の注意

今回の設計では、目的の無効化は **`enabled = false` のみ**で表現する。

したがって、`deleted_at` は設定しない。

これは、直前までの以下の設計方針と整合する。

- 無効化された目的も一覧・詳細で参照する
- 論理削除された目的は通常APIから取得しない
- OBJ-005では `deleted_at` を変更しない

以前の「無効化時に `enabled = false` と `deleted_at` の両方を設定する」という案を残している箇所がある場合は、`enabled` のみを変更する内容へ統一する必要がある。

---
## 26. React・TypeScriptでの利用

無効化リクエスト型は、以下とする。

```ts
export type DisableObjectiveRequest = {
  enabled: false;
};
```

`false` のリテラル型を使用することで、フロントエンドから `true` を誤送信できないようにする。

目的詳細の型は、以下とする。

```ts
export type ObjectiveDetail = {
  id: string;
  name: string;
  plannedYearMonth: string | null;
  requiredExpense: number;
  memo: string | null;
  enabled: boolean;
};
```

レスポンス型は、以下とする。

```ts
export type DisableObjectiveResponse = {
  data: ObjectiveDetail;
};
```

API呼び出し例は、以下とする。

```ts
const response =
  await apiClient.patch<DisableObjectiveResponse>(
    `/api/v1/objectives/${objectiveId}`,
    {
      enabled: false,
    },
  );
```

無効化処理の実行前には、確認ダイアログを表示する。

確認内容には、少なくとも以下を含める。

- 無効化後は目的達成判定の対象にならないこと
- 過去の目的達成判定履歴は保持されること
- Phase1では再有効化できないこと
- 無効化後も目的情報は一覧および詳細から参照できること

無効化成功後は、以下のいずれかを行う。

- 一覧APIを再取得する
- フロントエンドの対象データの `enabled` を `false` へ更新する

無効化済みの目的は、利用中の目的と区別して表示する。

無効化済みの目的に対しては、以下の操作を表示しない。

- 更新
- 無効化
- 目的達成判定

### 26.1 enabledの扱い

`enabled` は、目的の利用状態を表す。

```ts
enabled === true
```

- 利用中

```ts
enabled === false
```

- 無効化済み

フロントエンドでは、`enabled` を直接編集しない。

無効化を行う場合のみ、以下を送信する。

```json
{
  "enabled": false
}
```

### 26.2 確認ダイアログ

誤操作を防止するため、無効化前に確認ダイアログを表示する。

表示例：

```text
この目的を無効化しますか？

・目的達成判定では利用できなくなります
・過去の判定履歴は保持されます
・Phase1では再有効化できません
```

利用者がキャンセルした場合は、APIを呼び出さない。

### 26.3 無効化成功後の画面制御

無効化成功後は、返却されたレスポンスを利用して画面状態を更新する。

必要に応じて、以下のいずれかを行う。

- 目的一覧画面へ戻る
- 目的詳細画面を再表示する
- 一覧データの `enabled` を `false` へ更新する

### 26.4 ローディング表示

無効化リクエスト送信中は、二重送信を防止するため、無効化ボタンを非活性にする。

処理完了後またはエラー発生後に操作可能な状態へ戻す。

Phase1では、冪等性キーは使用しない。

### 26.5 エラー表示

エラーコードごとの基本的な扱いは、以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `INVALID_OBJECTIVE_ID` | 不正なURLとしてエラー表示する |
| `OBJECTIVE_NOT_FOUND` | 目的一覧画面へ戻し、対象が存在しないことを表示する |
| `OBJECTIVE_DISABLED` | すでに無効化済みであることを表示する |
| `INVALID_DEMO_USER_ID` | 共通エラー表示を行う |
| `DEMO_USER_NOT_FOUND` | デモ利用者選択画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

# 27. 設計上の補足

## 27.1 enabledによる無効化を採用する理由

目的は過去の目的達成判定履歴から参照される。

物理削除すると、過去データとの関連が失われる可能性がある。

そのため、Phase1では `enabled = false` によって利用停止を表現する。

---

## 27.2 論理削除を利用しない理由

本APIでは、`deleted_at` を変更しない。

論理削除は、通常利用対象から完全に除外するための内部管理用途とし、利用停止とは役割を分離する。

無効化された目的も一覧・詳細表示では取得対象とする。

---

## 27.3 再有効化を提供しない理由

Phase1では、再有効化APIは提供しない。

一度無効化した目的を再利用する要件が存在しないためである。

将来要件が追加された場合に再検討する。

---

## 27.4 判定履歴を変更しない理由

目的を無効化しても、過去の目的達成判定履歴は変更しない。

判定履歴は、判定実行時点のスナップショットとして保持する。

---

## 27.5 目的達成判定対象から除外する理由

無効化された目的は、現在の利用対象ではない。

そのため、新しい目的達成判定では対象外とする。

---

## 27.6 詳細取得可能とする理由

無効化後も、利用者は目的内容や過去の判定履歴を確認する必要がある。

そのため、一覧取得APIおよび詳細取得APIでは取得対象とする。

---

## 27.7 更新不可とする理由

無効化済み目的は、利用終了した業務データとして扱う。

そのため、更新APIでは `OBJECTIVE_DISABLED` を返却する。

---

## 27.8 200 OKでリソースを返却する理由

無効化成功時は、`204 No Content` ではなく `200 OK` を返却する。

フロントエンドは返却された `enabled = false` を利用して画面状態を更新できる。

---

## 27.9 Last Write Winsを採用しない理由

無効化は状態遷移を伴う操作である。

単純な Last Write Wins よりも、条件付き更新によって利用中の目的だけを無効化する方が整合性を保ちやすい。

---

## 27.10 キャッシュを採用しない理由

Phase1では目的件数が少ない。

キャッシュを導入すると、無効化時のキャッシュ破棄処理が必要となり、実装が複雑になる。

そのため、キャッシュは採用しない。

---

## 27.11 delete()を使用しない理由

本APIは利用停止を目的とする。

Laravelの `delete()` を使用すると `deleted_at` が更新され、通常APIから取得できなくなる。

本APIでは一覧および詳細取得を継続する必要があるため、`delete()` は使用しない。

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

---

# OBJ-006 目的達成判定

## 1. 概要

操作対象となるデモ利用者に帰属する目的について、
指定した判定対象年月時点での目的達成可否を判定する。

判定では、
以下の情報を利用する。

- 目的情報
- 平均手取り収入
- 判定対象年月の月末資産状況

判定結果は、
達成・未達成・判定不可のいずれかとする。

判定結果は、
利用者へ返却するとともに、
判定履歴として保存する。

---

## 2. ユースケース

利用者は、
現在の資産状況および平均手取り収入をもとに、
目的を達成できるか確認する。

例えば、
以下のような場合に使用する。

- 住宅購入資金を準備できるか確認する
- 車の購入資金を準備できるか確認する
- 結婚資金を準備できるか確認する
- 留学資金を準備できるか確認する

判定結果は、
目的達成判定画面へ表示するとともに、
判定履歴として保存する。

---

## 3. エンドポイント

```http
POST /api/v1/objectives/{objectiveId}/assessments
```

---

## 4. HTTPメソッド

`POST`

本APIは、
目的達成判定を新たに実行し、
判定履歴を登録する。

同じ目的、
同じ判定対象年月に対して
複数回判定を実行できる。

判定結果は、
過去の判定履歴として保持する。

---

## 5. 認証・利用者の扱い

Phase1では、
認証機能を実装しない。

操作対象となるデモ利用者は、
`X-Demo-User-Id`
リクエストヘッダーで指定する。

```http
X-Demo-User-Id: 1
```

指定された利用者に帰属する
目的のみ判定できる。

他の利用者に帰属する目的は、
判定できない。

他の利用者に帰属する目的を指定した場合は、
対象が存在しないものとして扱う。

無効化された目的は、
判定対象としない。

論理削除された目的も、
判定対象としない。

利用者IDは、
リクエストボディ、
クエリパラメータまたは
パスパラメータでは受け付けない。

---

## 6. パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `objectiveId` | string | ○ | 判定対象となる目的ID |

---

## 7. クエリパラメータ

なし。

---

## 8. リクエストヘッダー

### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-Demo-User-Id` | ○ | 操作対象となるデモ利用者ID |
| `Content-Type` | ○ | `application/json` を指定する |
| `Accept` | ○ | `application/json` を指定する |

リクエスト例：

```http
POST /api/v1/objectives/15/assessments
Content-Type: application/json
Accept: application/json
X-Demo-User-Id: 1
```

---

## 9. リクエストボディ

```json
{
  "targetYearMonth": "2026-08"
}
```

判定対象年月を指定して、
目的達成判定を実行する。

---

## 10. リクエスト項目

| 項目 | 型 | 必須 | NULL | 説明 |
|---|---|:---:|:---:|---|
| `targetYearMonth` | string | ○ | × | 判定対象年月（YYYY-MM形式） |

---

### 10.1 targetYearMonth

目的達成判定を実行する対象年月を指定する。

形式は、
`YYYY-MM` とする。

```json
{
  "targetYearMonth": "2026-08"
}
```

判定対象年月は、
以下の処理で利用する。

- 平均手取り収入の算出
- 判定対象年月の月末資産状況の取得
- 判定履歴への保存

---

## 11. 更新対象外項目

本APIでは、
以下の項目をリクエストで受け付けない。

- `id`
- `userId`
- `objectiveId`
- `assessmentResult`
- `averageAmount`
- `currentAssetAmount`
- `remainingAmount`
- `createdAt`
- `updatedAt`

判定結果および判定に使用する値は、
すべてサーバー側で算出する。

クライアントから送信された値は、
信頼しない。

更新対象外項目が含まれている場合は、
バリデーションエラーとする。

---

## 12. バリデーション

### 12.1 X-Demo-User-Id

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

---

### 12.2 objectiveId

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された目的が存在すること
- 指定された目的が操作対象利用者に帰属すること
- 指定された目的が有効であること

他の利用者に帰属する目的を指定した場合は、
存在しないものとして扱う。

無効化された目的は、
判定対象としない。

論理削除された目的も、
判定対象としない。

---

### 12.3 リクエストボディ

以下を検証する。

- JSONオブジェクトであること
- `targetYearMonth` が指定されていること
- 定義されていない項目が含まれていないこと
- 更新対象外項目が含まれていないこと

空のJSONオブジェクトは、
バリデーションエラーとする。

```json
{}
```

---

### 12.4 targetYearMonth

以下を検証する。

- 必須であること
- 文字列であること
- `null` でないこと
- `YYYY-MM` 形式であること
- 実在する年月であること

以下は、
不正な値として扱う。

```text
2026-8
2026/08
2026-13
202608
```

---

### 12.5 判定実行条件

判定対象年月について、
以下を検証する。

- 判定対象年月の月末資産状況が存在すること
- 判定対象年月より前の連続する3か月分の手取り収入が存在すること
- 平均手取り収入を算出できること

以下の場合は、
目的達成判定を実行できない。

- 判定対象年月の月末資産状況が未登録である
- 平均手取り収入の算出に必要な3か月分の手取り収入が不足している

これらは、
入力形式ではなく業務上の成立条件を満たさないため、
業務ルール違反として扱う。

---

## 6. パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `objectiveId` | string | ○ | 判定対象となる目的ID |

---

## 7. クエリパラメータ

なし。

---

## 8. リクエストヘッダー

### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-Demo-User-Id` | ○ | 操作対象となるデモ利用者ID |
| `Content-Type` | ○ | `application/json` を指定する |
| `Accept` | ○ | `application/json` を指定する |

リクエスト例：

```http
POST /api/v1/objectives/15/assessments
Content-Type: application/json
Accept: application/json
X-Demo-User-Id: 1
```

---

## 9. リクエストボディ

```json
{
  "targetYearMonth": "2026-08"
}
```

判定対象年月を指定して、
目的達成判定を実行する。

---

## 10. リクエスト項目

| 項目 | 型 | 必須 | NULL | 説明 |
|---|---|:---:|:---:|---|
| `targetYearMonth` | string | ○ | × | 判定対象年月（YYYY-MM形式） |

---

### 10.1 targetYearMonth

目的達成判定を実行する対象年月を指定する。

形式は、
`YYYY-MM` とする。

```json
{
  "targetYearMonth": "2026-08"
}
```

判定対象年月は、
以下の処理で利用する。

- 平均手取り収入の算出
- 判定対象年月の月末資産状況の取得
- 判定履歴への保存

---

## 11. 更新対象外項目

本APIでは、
以下の項目をリクエストで受け付けない。

- `id`
- `userId`
- `objectiveId`
- `assessmentResult`
- `averageAmount`
- `currentAssetAmount`
- `remainingAmount`
- `createdAt`
- `updatedAt`

判定結果および判定に使用する値は、
すべてサーバー側で算出する。

クライアントから送信された値は、
信頼しない。

更新対象外項目が含まれている場合は、
バリデーションエラーとする。

---

## 12. バリデーション

### 12.1 X-Demo-User-Id

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

---

### 12.2 objectiveId

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された目的が存在すること
- 指定された目的が操作対象利用者に帰属すること
- 指定された目的が有効であること

他の利用者に帰属する目的を指定した場合は、
存在しないものとして扱う。

無効化された目的は、
判定対象としない。

論理削除された目的も、
判定対象としない。

---

### 12.3 リクエストボディ

以下を検証する。

- JSONオブジェクトであること
- `targetYearMonth` が指定されていること
- 定義されていない項目が含まれていないこと
- 更新対象外項目が含まれていないこと

空のJSONオブジェクトは、
バリデーションエラーとする。

```json
{}
```

---

### 12.4 targetYearMonth

以下を検証する。

- 必須であること
- 文字列であること
- `null` でないこと
- `YYYY-MM` 形式であること
- 実在する年月であること

以下は、
不正な値として扱う。

```text
2026-8
2026/08
2026-13
202608
```

---

### 12.5 判定実行条件

判定対象年月について、
以下を検証する。

- 判定対象年月の月末資産状況が存在すること
- 判定対象年月より前の連続する3か月分の手取り収入が存在すること
- 平均手取り収入を算出できること

以下の場合は、
目的達成判定を実行できない。

- 判定対象年月の月末資産状況が未登録である
- 平均手取り収入の算出に必要な3か月分の手取り収入が不足している

これらは、
入力形式ではなく業務上の成立条件を満たさないため、
業務ルール違反として扱う。

---

## 13. 業務ルール

- 操作対象利用者に帰属する目的のみ判定できる。
- 他の利用者に帰属する目的は判定できない。
- 無効化された目的は判定できない。
- 論理削除された目的は判定できない。
- 判定対象年月の月末資産状況を使用して判定する。
- 平均手取り収入は、判定対象年月より前の連続する3か月から算出する。
- 判定対象年月自身の手取り収入は平均値の算出対象に含めない。
- 手取り収入が0円の対象年月も平均値の算出対象とする。
- 3か月分の手取り収入が揃っていない場合は判定不可とする。
- 判定対象年月の月末資産状況が存在しない場合は判定不可とする。
- 判定結果は、達成・未達成・判定不可のいずれかとする。
- 判定結果は判定履歴として保存する。
- 判定履歴は過去の結果を変更しない。
- 同じ目的および同じ判定対象年月に対して複数回判定を実行できる。
- 判定実行のたびに新しい判定履歴を登録する。
- クライアントから送信された判定結果、平均手取り収入および資産額は使用しない。

---

## 14. 判定ロジック

判定には、
以下の情報を使用する。

- 必要支出額
- 判定対象年月の月末資産状況
- 判定対象年月直前3か月の平均手取り収入

判定結果は、
以下のいずれかとする。

| 判定結果 | 条件 |
|---|---|
| `ACHIEVED` | 必要支出額以下の条件を満たす |
| `NOT_ACHIEVED` | 必要支出額を満たさない |
| `UNASSESSABLE` | 判定に必要なデータが不足している |

具体的な計算式は、
ドメインサービスで一元管理する。

---

## 15. 処理フロー

```text
リクエスト受付
    ↓
リクエストID生成
    ↓
X-Demo-User-Id検証
    ↓
操作対象利用者の特定
    ↓
objectiveId検証
    ↓
リクエストボディのバリデーション
    ↓
目的取得
    ↓
目的の利用状態確認
    ↓
平均手取り収入算出
    ↓
月末資産状況取得
    ↓
目的達成判定
    ↓
判定履歴登録
    ↓
APIレスポンス形式へ変換
    ↓
正常レスポンス返却
```

---

## 16. トランザクション境界

以下の処理を、
1つのデータベーストランザクション内で実行する。

- 目的達成判定
- 判定履歴の登録

処理途中で例外が発生した場合は、
判定履歴の登録をロールバックする。

判定に利用する参照データは、
更新しない。

---

## 17. 排他制御

Phase1では、
明示的な行ロックおよび
楽観ロックは採用しない。

同一目的について
複数の判定要求が同時に実行された場合でも、
各判定結果を独立した判定履歴として保存する。

判定履歴は追加のみであり、
既存データを更新しない。

---

## 18. 成功レスポンス

### 18.1 HTTPステータス

```http
201 Created
```

---

### 18.2 レスポンスボディ

```json
{
  "data": {
    "assessmentHistoryId": "120",
    "objectiveId": "15",
    "targetYearMonth": "2026-08",
    "assessmentResult": "ACHIEVED",
    "requiredExpense": 3000000,
    "currentAssetAmount": 3250000,
    "averageNetIncome": 310000,
    "remainingAmount": 250000,
    "assessedAt": "2026-08-31T10:15:30+09:00"
  }
}
```

---

## 19. レスポンス項目

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `data` | object | × | 判定結果 |
| `data.assessmentHistoryId` | string | × | 判定履歴ID |
| `data.objectiveId` | string | × | 目的ID |
| `data.targetYearMonth` | string | × | 判定対象年月 |
| `data.assessmentResult` | string | × | 判定結果 |
| `data.requiredExpense` | integer | × | 必要支出額 |
| `data.currentAssetAmount` | integer | × | 判定対象年月の月末資産額 |
| `data.averageNetIncome` | integer | × | 平均手取り収入 |
| `data.remainingAmount` | integer | × | 判定後残額 |
| `data.assessedAt` | string | × | 判定実行日時（ISO 8601形式） |

以下の項目は、
レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`
- `deletedAt`

判定結果は、
保存された判定履歴の内容を返却する。

---

## 20. エラーレスポンス

異常時は、  
API共通方針で定めた共通エラーレスポンス形式を使用する。

判定に必要なデータが不足している場合は、  
判定結果を返却せず、判定履歴も登録しない。

---

### 20.1 指定した目的が存在しない場合

指定した目的が存在しない場合、  
論理削除されている場合、  
または操作対象利用者に帰属しない場合は、  
`OBJECTIVE_NOT_FOUND` を返却する。

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

### 20.2 目的が無効化されている場合

無効化された目的を判定対象として指定した場合は、  
`OBJECTIVE_DISABLED` を返却する。

```json
{
  "error": {
    "code": "OBJECTIVE_DISABLED",
    "message": "無効化された目的は達成判定できません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 20.3 確定済みの月末資産状況が存在しない場合

判定対象年月に利用する確定済みの月末資産状況が存在しない場合は、  
`CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` を返却する。

```json
{
  "error": {
    "code": "CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND",
    "message": "目的達成判定に利用できる確定済みの月末資産状況がありません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

未確定の月末資産状況は、目的達成判定に使用しない。

---

### 20.4 平均手取り収入を算出できない場合

判定対象年月より前の連続する3か月の手取り収入が不足している場合は、  
`NET_INCOME_DATA_INSUFFICIENT` を返却する。

```json
{
  "error": {
    "code": "NET_INCOME_DATA_INSUFFICIENT",
    "message": "平均手取り収入を算出するためのデータが不足しています。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

金額が0円で登録された手取り収入は、有効な実績値として扱う。

レコードが存在しない対象年月のみ、データ不足として扱う。

---

### 20.5 利用可能資産を算出できない場合

判定対象年月時点の利用可能資産設定を特定できない場合、または利用可能資産を算出するための必要なデータが不足している場合は、  
`AVAILABLE_ASSETS_CALCULATION_NOT_AVAILABLE` を返却する。

```json
{
  "error": {
    "code": "AVAILABLE_ASSETS_CALCULATION_NOT_AVAILABLE",
    "message": "利用可能資産を算出できないため、目的達成判定を実行できません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 20.6 バリデーションエラー

リクエスト内容が不正な場合は、  
`VALIDATION_ERROR` を返却する。

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "nextMonthCreditCardPayment",
        "reason": "min",
        "message": "翌月クレジットカード支払予定額は0以上で指定してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 20.7 判定を実行できない場合

前述した個別のエラーへ分類できないものの、目的達成判定の成立条件を満たさない場合は、  
`ASSESSMENT_NOT_AVAILABLE` を返却する。

```json
{
  "error": {
    "code": "ASSESSMENT_NOT_AVAILABLE",
    "message": "必要な情報が不足しているため、目的達成判定を実行できません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

`ASSESSMENT_NOT_AVAILABLE` は、想定外のシステムエラーではなく、判定を成立させられない業務状態を表す。

可能な限り、不足原因を表す具体的なエラーコードを優先する。

---

## 21. HTTPステータス

| HTTPステータス | 条件 |
|---|---|
| `201 Created` | 目的達成判定に成功し、判定履歴を登録した |
| `400 Bad Request` | 利用者ID、目的IDまたはリクエスト形式が不正である |
| `404 Not Found` | 指定された利用者、目的または判定に必要な対象データが存在しない |
| `422 Unprocessable Entity` | 入力値は解釈できるが、業務上の条件を満たさず判定できない |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

### 21.1 201の扱い

本APIでは、目的達成判定結果を返却するだけでなく、`assessment_histories` へ新しい判定履歴を登録する。

そのため、正常終了時は `200 OK` ではなく `201 Created` を返却する。

### 21.2 404の扱い

以下の場合は、`404 Not Found` を返却する。

- 指定された利用者が存在しない
- 指定された利用者が論理削除されている
- 指定された目的が存在しない
- 指定された目的が論理削除されている
- 指定された目的が操作対象利用者に属していない
- 判定に使用する確定済み月末資産状況が存在しない

他の利用者に帰属する目的を指定した場合も、対象の存在を推測できないようにするため、存在しないものとして扱う。

### 21.3 422の扱い

以下の場合は、`422 Unprocessable Entity` を返却する。

- 目的が無効化されている
- 入力値が業務上許容されない
- 平均手取り収入を算出するためのデータが不足している
- 利用可能資産を算出できない
- その他、目的達成判定の成立条件を満たしていない

リクエスト形式は正しいが、業務上の理由によって判定できないため、`422 Unprocessable Entity` を返却する。

### 21.4 判定不可時の履歴登録

4xxまたは5xxを返却した場合は、`assessment_histories` へ判定履歴を登録しない。

判定処理が正常に完了し、判定結果を確定できた場合のみ、判定履歴を登録する。

---

## 22. エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `DEMO_USER_CONTEXT_REQUIRED` | 400 | `X-Demo-User-Id` が指定されていない | × |
| `INVALID_DEMO_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `INVALID_OBJECTIVE_ID` | 400 | 目的IDの形式が不正である | × |
| `DEMO_USER_NOT_FOUND` | 404 | 指定された利用者が存在しない | × |
| `OBJECTIVE_NOT_FOUND` | 404 | 指定された目的が存在しない、または操作対象利用者に属していない | × |
| `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` | 404 | 判定に使用できる確定済み月末資産状況が存在しない | ○ |
| `OBJECTIVE_DISABLED` | 422 | 無効化された目的を判定しようとした | × |
| `NET_INCOME_DATA_INSUFFICIENT` | 422 | 平均手取り収入の算出に必要な連続3か月のデータが不足している | ○ |
| `AVAILABLE_ASSETS_CALCULATION_NOT_AVAILABLE` | 422 | 利用可能資産を算出できない | ○ |
| `ASSESSMENT_NOT_AVAILABLE` | 422 | その他の業務上の理由により判定できない | ○ |
| `VALIDATION_ERROR` | 422 | 入力値が不正である | × |
| `INTERNAL_SERVER_ERROR` | 500 | 想定外のサーバーエラーが発生した | ○ |

エラーコードの正式な定義は、`error-codes.md` に従う。

再試行可能となっている業務エラーも、同じリクエストをそのまま再送するだけでは解消しない場合がある。

不足データの登録、月末資産状況の確定または利用可能資産設定の修正後に、再度判定を実行する。

---

## 23. 冪等性

本APIは、HTTP `POST` を使用する。

判定実行に成功するたびに、新しい判定履歴を `assessment_histories` へ登録する。

そのため、同一内容のリクエストを複数回実行した場合でも、実行回数分の判定履歴が作成される。

本APIは冪等ではない。

同一目的について、同じ入力値および同じ基礎データで複数回判定することを許可する。

これは、判定が実行された事実を時系列の履歴として保持するためである。

Phase1では、冪等性キー（`Idempotency-Key`）を採用しない。

フロントエンドでは、判定処理中に実行ボタンを非活性化し、意図しない二重送信を防止する。

ただし、通信再送などによる完全な重複登録までは防止しない。

将来、同一操作による重複履歴の防止が必要になった場合は、以下を再検討する。

- `Idempotency-Key` の採用
- クライアント生成リクエストIDの保存
- 同一条件・短時間での重複判定制御

 **設計メモ**  
`CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` と `AVAILABLE_ASSETS_CALCULATION_NOT_AVAILABLE` は、既存の `error-codes.md` に未登録であれば追記する。既存コードとして `SNAPSHOT_NOT_FOUND` など別名を採用済みの場合は、その名称へ統一する。

## 24. 関連テーブル

### 24.1 objectives

判定対象となる目的を保持する。

使用する主なカラムは、以下とする。

| カラム | 用途 |
|---|---|
| `id` | 判定対象となる目的ID |
| `user_id` | 利用者境界 |
| `name` | 判定履歴へ保存する目的名 |
| `required_expense` | 目的達成に必要な支出額 |
| `enabled` | 新規判定の実行可否 |
| `deleted_at` | 論理削除状態の判定 |

本APIでは、操作対象利用者に帰属する有効な目的のみ判定対象とする。

以下の目的は、判定対象としない。

- 他の利用者に帰属する目的
- 無効化された目的
- 論理削除された目的

目的の実施予定年月は、Phase1の目的達成判定計算には使用しない。

---

### 24.2 assessment_histories

目的達成判定の実行結果を保持する。

本APIでは、判定が正常に完了した場合のみ、新しい判定履歴を登録する。

保存する内容は、テーブル定義書で定めた判定時点の入力値、計算根拠および判定結果とする。

主な用途は、以下とする。

| 用途 | 説明 |
|---|---|
| 目的との関連 | 判定対象となった目的を識別する |
| 判定対象年月 | どの年月を基準に判定したかを保持する |
| 必要支出額 | 判定時点の目的の必要支出額を保持する |
| 利用可能資産 | 判定時点で算出した利用可能資産を保持する |
| 平均手取り収入 | 判定時点で算出した平均手取り収入を保持する |
| 翌月クレジットカード支払予定額 | 判定実行時に入力された金額を保持する |
| 計算結果 | 判定に使用した計算結果を保持する |
| 判定結果 | 達成可能または達成困難などの結果を保持する |
| 判定日時 | 判定を実行した日時を保持する |

目的、手取り収入または月末資産情報が後から変更されても、保存済みの判定履歴は更新しない。

判定不可となった場合は、`assessment_histories` へレコードを登録しない。

具体的な保存カラム名は、`table-definition.md` の `assessment_histories` 定義に従う。

---

### 24.3 month_end_asset_snapshots

対象年月ごとの月末資産状況および確定状態を保持する。

本APIでは、判定に使用できる確定済みの月末資産状況を取得する。

未確定の月末資産状況は、目的達成判定に使用しない。

判定に使用した月末資産状況は、判定履歴から追跡できるようにする。

---

### 24.4 month_end_asset_balances

口座単位で記録された月末資産残高を保持する。

残高記録単位が口座単位の資産口座について、利用可能資産の算出に使用する。

対象となる月末資産状況に紐づく残高のみを取得する。

---

### 24.5 month_end_holding_values

商品単位で記録された月末評価額を保持する。

残高記録単位が商品単位の資産口座について、利用可能資産の算出に使用する。

対象となる月末資産状況に紐づく商品別月末評価額のみを取得する。

---

### 24.6 asset_account_available_settings

資産口座が目的達成判定の利用可能資産へ含まれるかを、対象年月単位で判定するために使用する。

本APIでは、判定対象年月時点で有効な設定を使用する。

現在の設定だけを参照して、過去の判定対象年月へ適用してはならない。

対象年月時点の設定を特定できない場合は、利用可能資産を算出せず、判定不可として扱う。

---

### 24.7 asset_accounts

資産口座の情報を保持する。

主に以下の判定に使用する。

- 月末資産残高がどの資産口座に属するか
- 残高記録単位が口座単位か商品単位か
- 利用可能資産設定の対象となる資産口座か

論理削除または無効化後であっても、判定対象年月時点の記録および確定済みスナップショットとの整合性を考慮して扱う。

---

### 24.8 holding_assets

商品単位で管理する保有商品の情報を保持する。

商品別月末評価額と所属資産口座の関連を特定するために使用する。

現在の利用状態だけを理由として、過去の対象年月に記録された商品別月末評価額を除外してはならない。

---

### 24.9 net_incomes

対象年月ごとの手取り収入を保持する。

判定対象年月より前の連続する3か月の手取り収入から、平均手取り収入を算出する。

金額が `0円` のレコードは、有効な実績値として算出対象へ含める。

対象となる3か月のうち、1か月以上のレコードが存在しない場合は、平均手取り収入を算出せず、判定不可として扱う。

---

## 25. 関連する機能要件

- `10.9 業務ルール`
  - 目的達成判定に必要な変動値は目的へ保持しない
  - 翌月クレジットカード支払予定額は判定実行時に入力する
  - 実施予定年月はPhase1の将来予測に使用しない
  - 無効化した目的は新規判定対象としない
  - 保存済みの判定結果は目的の編集または無効化によって変更しない

- `11. 目的達成判定`
  - 指定した目的について達成可否を判定する
  - 確定済みの月末資産状況を使用する
  - 判定対象年月時点の利用可能資産設定を使用する
  - 判定対象年月より前の連続する3か月から平均手取り収入を算出する
  - 翌月クレジットカード支払予定額を判定計算へ使用する
  - 判定時点の入力値、計算根拠および判定結果を履歴へ保存する
  - 判定条件を満たさない場合は判定を実行せず、履歴を保存しない

- `7. 月末資産状況の確定`
  - 確定済みの月末資産状況のみ正式な資産状況として使用する
  - 未確定の月末資産状況は目的達成判定に使用しない

- `9.6 平均手取り収入`
  - 判定対象年月より前の連続する3か月から平均手取り収入を算出する

- `9.7 判定対象`
  - 判定対象年月自身の手取り収入は平均値へ含めない

- `9.8 データ不足`
  - 算出に必要な手取り収入が不足する場合は平均値を算出しない

---

## 26. テスト観点

### 26.1 正常系

- 有効な目的について目的達成判定を実行できること
- 判定成功時に `201 Created` が返却されること
- 判定結果がレスポンスへ返却されること
- 判定成功時に判定履歴が1件登録されること
- 同一目的について複数回判定できること
- 判定実行ごとに新しい判定履歴が登録されること

---

### 26.2 利用者境界

- 操作対象利用者に帰属する目的のみ判定できること
- 他利用者の目的を判定できないこと
- 他利用者の目的IDを指定した場合は `404 Not Found` となること
- 他利用者の資産情報を判定計算へ含めないこと
- 他利用者の手取り収入を平均値へ含めないこと
- 他利用者の利用可能資産設定を参照しないこと

---

### 26.3 目的

- 存在する有効な目的を判定できること
- 存在しない目的IDで `404 Not Found` となること
- 論理削除済みの目的で `404 Not Found` となること
- 無効化済みの目的で `422 Unprocessable Entity` となること
- 目的ID形式不正で `400 Bad Request` となること
- 実施予定年月が未設定でも判定できること
- 実施予定年月が判定計算へ使用されないこと
- 目的の必要支出額が判定計算へ使用されること

---

### 26.4 月末資産状況

- 判定に使用する確定済み月末資産状況を正しく取得できること
- 未確定の月末資産状況を判定へ使用しないこと
- 確定済み月末資産状況が存在しない場合は判定できないこと
- 確定済み月末資産状況が存在しない場合は `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` となること
- 判定に使用した月末資産状況を判定履歴から追跡できること

---

### 26.5 利用可能資産

- 判定対象年月時点の利用可能資産設定を使用すること
- 現在の設定を過去の判定対象年月へ誤って適用しないこと
- 利用可能資産へ含める設定の資産だけを合算すること
- 利用可能資産へ含めない設定の資産を合算しないこと
- 口座単位の月末資産残高を正しく合算すること
- 商品単位の月末評価額を正しく合算すること
- 同じ資産を口座単位と商品単位で二重計上しないこと
- 利用可能資産設定を特定できない場合は判定できないこと
- 利用可能資産を算出できない場合は `AVAILABLE_ASSETS_CALCULATION_NOT_AVAILABLE` となること

---

### 26.6 平均手取り収入

- 判定対象年月より前の連続する3か月を使用すること
- 判定対象年月自身の手取り収入を含めないこと
- 3か月分の手取り収入から平均値を算出できること
- 金額が `0円` の月も平均値へ含めること
- レコードが存在しない月は `0円` として補完しないこと
- 1か月不足する場合は判定できないこと
- 2か月不足する場合は判定できないこと
- 3か月すべて不足する場合は判定できないこと
- データ不足時は `NET_INCOME_DATA_INSUFFICIENT` となること
- 平均値の端数処理が確定した業務ルールどおりであること

---

### 26.7 翌月クレジットカード支払予定額

- 正常な整数値を指定して判定できること
- `0円` を指定して判定できること
- 負数で `422 Unprocessable Entity` となること
- 小数で `422 Unprocessable Entity` となること
- 最大許容値を超えた場合は `422 Unprocessable Entity` となること
- 入力値が判定計算へ正しく反映されること
- 入力値が判定履歴へ保存されること

---

### 26.8 判定計算

- 定義した計算式どおりに判定されること
- 利用可能資産が必要支出額および必要な控除額を上回る場合の結果が正しいこと
- 利用可能資産が必要額と等しい境界値で結果が正しいこと
- 利用可能資産が必要額を下回る場合の結果が正しいこと
- 計算途中の端数処理が業務ルールどおりであること
- 金額計算で浮動小数点数を使用しないこと
- 判定結果と計算根拠に矛盾がないこと

具体的な計算式および境界値は、機能要件と `assessment_histories` のテーブル定義に従ってテストケースへ落とし込む。

---

### 26.9 判定履歴

- 判定成功時のみ履歴が登録されること
- 判定不可時は履歴が登録されないこと
- バリデーションエラー時は履歴が登録されないこと
- システムエラー時は履歴が登録されないこと
- 判定対象となった目的IDが保存されること
- 判定時点の目的名が保存されること
- 判定時点の必要支出額が保存されること
- 判定時点の利用可能資産が保存されること
- 判定時点の平均手取り収入が保存されること
- 翌月クレジットカード支払予定額が保存されること
- 計算根拠および判定結果が保存されること
- 判定日時が保存されること

保存項目の具体的な名称は、`assessment_histories` のテーブル定義に合わせる。

---

### 26.10 履歴の不変性

- 判定後に目的名を変更しても保存済み履歴が変更されないこと
- 判定後に必要支出額を変更しても保存済み履歴が変更されないこと
- 判定後に目的を無効化しても保存済み履歴が変更されないこと
- 判定後に手取り収入を変更しても保存済み履歴が変更されないこと
- 判定後に月末資産情報を変更しても保存済み履歴が変更されないこと
- 判定後に利用可能資産設定を変更しても保存済み履歴が変更されないこと

---

### 26.11 トランザクション

- 判定計算と判定履歴登録が同一トランザクションで実行されること
- 履歴登録中に例外が発生した場合はロールバックされること
- 判定履歴の一部だけが保存されないこと
- 判定に失敗した場合は履歴が保存されないこと
- 判定成功レスポンス返却前に履歴登録が完了していること

---

### 26.12 同時実行

- 同一目的へ同時に判定を実行してもデータ整合性が維持されること
- 同時実行した判定ごとに独立した履歴が登録されること
- 判定処理中に目的が無効化された場合の挙動が一貫していること
- 判定処理中に基礎データが更新された場合でも、1回の判定内で計算根拠に矛盾が生じないこと
- デッドロックなどのデータベース例外が共通エラーへ変換されること

Phase1で厳密なスナップショット分離を保証しない場合は、その制約を設計上の補足へ明記する。

---

### 26.13 二重送信

- 判定ボタンを二重送信できないよう画面制御されること
- 同一リクエストを再実行すると新しい判定履歴が登録されること
- 同一条件の判定履歴が複数存在しても整合性が崩れないこと
- Phase1では `Idempotency-Key` を要求しないこと

---

### 26.14 レスポンス契約

- JSONフィールド名がcamelCaseであること
- IDがAPI共通方針に従って文字列で返却されること
- 対象年月が `YYYY-MM` 形式で返却されること
- 金額項目が整数で返却されること
- 判定結果が定義済みの値で返却されること
- 判定に使用した入力値と計算根拠が返却されること
- `userId` がレスポンスへ含まれないこと
- DB内部のカラム名がそのまま公開されないこと
- `data` オブジェクトで返却されること

---

### 26.15 エラー時の履歴

- `OBJECTIVE_NOT_FOUND` 時に履歴が登録されないこと
- `OBJECTIVE_DISABLED` 時に履歴が登録されないこと
- `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` 時に履歴が登録されないこと
- `NET_INCOME_DATA_INSUFFICIENT` 時に履歴が登録されないこと
- `AVAILABLE_ASSETS_CALCULATION_NOT_AVAILABLE` 時に履歴が登録されないこと
- `ASSESSMENT_NOT_AVAILABLE` 時に履歴が登録されないこと
- `VALIDATION_ERROR` 時に履歴が登録されないこと
- `INTERNAL_SERVER_ERROR` 時に不完全な履歴が残らないこと

---

### 26.16 異常系

- 想定外の例外で `500 Internal Server Error` となること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログに同じリクエストIDが記録されること
- SQLやスタックトレースなどの内部情報がレスポンスへ含まれないこと
- エラーログへ不要な金額情報やメモが出力されないこと

---

## 27. Laravel実装方針

### 27.1 Action

HTTPリクエストを受け付け、目的ID、判定対象年月、翌月クレジットカード支払予定額およびデモ利用者コンテキストを取得する。

Form Requestまたは入力用DTOから検証済みの入力値を受け取り、目的達成判定UseCaseを呼び出す。

UseCaseから受け取った判定結果を、Responderへ渡す。

以下の処理は、Actionへ直接記述しない。

- 入力値の単項目バリデーション
- 利用者境界の判定
- 判定対象データの取得
- 判定計算
- 判定履歴登録
- トランザクション制御
- レスポンス生成処理

---

### 27.2 UseCase

目的達成判定のユースケース処理を担当する。

主な処理は、以下とする。

- 操作対象利用者を受け取る
- 判定対象となる目的を取得する
- 確定済み月末資産状況を取得する
- 判定対象年月時点の利用可能資産設定を取得する
- 利用可能資産を算出する
- 平均手取り収入を算出する
- 判定計算を実行する
- 判定履歴を登録する
- 判定結果を返却する

判定条件を満たさない場合は、業務エラーとして扱う。

判定履歴は、判定成功時のみ登録する。

---

### 27.3 Form Request / DTO

入力値の形式および単項目バリデーションを担当する。

主な検証対象は、以下とする。

- `targetYearMonth`
- `nextMonthCreditCardPayment`

Form Requestでは、以下を検証する。

- 必須項目
- データ型
- NULL可否
- `YYYY-MM` 形式
- 数値範囲
- 未定義項目

データベース状態に依存する以下の業務ルールは、UseCaseで実施する。

- 目的の存在確認
- 利用者境界確認
- 利用状態確認
- 確定済み月末資産状況の存在確認
- 平均手取り収入算出可否
- 利用可能資産算出可否

DTO例：

```php
final readonly class AssessObjectiveInput
{
    public function __construct(
        public string $targetYearMonth,
        public int $nextMonthCreditCardPayment,
    ) {
    }
}
```

---

### 27.4 Query

判定対象データ取得を担当する。

取得対象は、以下とする。

- `objectives`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_account_available_settings`
- `asset_accounts`
- `holding_assets`
- `net_incomes`

目的取得時は、必ず利用者境界を条件へ含める。

```php
$objective = Objective::query()
    ->where('id', $objectiveId)
    ->where('user_id', $demoUserId)
    ->first();
```

論理削除済み目的は、取得対象としない。

無効化された目的は取得後、`OBJECTIVE_DISABLED` として扱う。

---

### 27.5 Repository

判定履歴登録のみ担当する。

Repositoryでは、`assessment_histories` への登録を実施する。

保存対象は、以下とする。

- `objective_id`
- 判定対象年月
- 判定時点の目的情報
- 利用可能資産
- 平均手取り収入
- 翌月クレジットカード支払予定額
- 判定結果
- 判定根拠

目的、月末資産情報、手取り収入および利用可能資産設定は更新しない。

---

### 27.6 判定サービス

判定計算は、UseCaseへ直接記述せず、専用ドメインサービスへ委譲する。

例：

```php
$result = $assessmentService->assess(
    objective: $objective,
    availableAssets: $availableAssets,
    averageNetIncome: $averageNetIncome,
    nextMonthCreditCardPayment: $input->nextMonthCreditCardPayment,
);
```

判定ロジックは、Controller、Repository、Queryへ記述しない。

---

### 27.7 トランザクション

以下の処理を、1つのデータベーストランザクション内で実行する。

- 判定対象取得
- 判定計算
- 判定履歴登録

実装例：

```php
$result = DB::transaction(
    function () use (
        $demoUserId,
        $objectiveId,
        $input,
    ) {
        $result =
            $this->useCase->execute(
                $demoUserId,
                $objectiveId,
                $input,
            );

        return $result;
    },
);
```

判定途中で例外が発生した場合は、判定履歴を登録しない。

---

### 27.8 Responder

UseCaseから受け取った判定結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、以下のHTTPステータスで返却する。

```text
201 Created
```

判定履歴が登録されたことを、レスポンスから判別できる。

Responderは、以下を行わない。

- 判定計算
- 判定履歴登録
- データ取得
- 業務ルール判定

---

### 27.9 API Resource

判定結果を、APIレスポンス形式へ変換する。

Resourceでは、フィールド名をcamelCaseへ変換する。

内部DBカラムは公開しない。

例：

```php
return [
    'assessmentResult' => $this->assessment_result,
    'availableAssets' => $this->available_assets,
    'requiredExpense' => $this->required_expense,
    'averageNetIncome' => $this->average_net_income,
    'remainingAmount' => $this->remaining_amount,
];
```

---

### 27.10 Middleware

以下の共通ミドルウェアを適用する。

- デモ利用者コンテキスト設定
- リクエストID生成
- JSON共通処理
- 共通例外処理
- ログコンテキスト設定

---

### 27.11 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `DEMO_USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_DEMO_USER_ID` |
| 利用者不存在 | `DEMO_USER_NOT_FOUND` |
| 目的ID形式不正 | `INVALID_OBJECTIVE_ID` |
| 目的不存在 | `OBJECTIVE_NOT_FOUND` |
| 無効化済み目的 | `OBJECTIVE_DISABLED` |
| 確定済み月末資産状況不存在 | `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` |
| 平均手取り収入不足 | `NET_INCOME_DATA_INSUFFICIENT` |
| 利用可能資産算出不可 | `AVAILABLE_ASSETS_CALCULATION_NOT_AVAILABLE` |
| 判定不可 | `ASSESSMENT_NOT_AVAILABLE` |
| 入力値不正 | `VALIDATION_ERROR` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

SQL、スタックトレースおよび内部例外メッセージは、APIレスポンスへ含めない。

ログには、調査に必要な範囲で以下を記録する。

- 利用者ID
- 目的ID
- 判定対象年月
- リクエストID
- 独自エラーコード

判定に利用した金額内訳や計算途中の情報は、通常ログへ出力しない。


## 28. React・TypeScriptでの利用

リクエスト型は、
以下とする。

```ts
export type AssessObjectiveRequest = {
  targetYearMonth: string;
  nextMonthCreditCardPayment: number;
};
```

判定結果の型は、
以下とする。

```ts
export type ObjectiveAssessmentResult = {
  assessmentResult: string;
  availableAssets: number;
  requiredExpense: number;
  averageNetIncome: number;
  remainingAmount: number;
};
```

レスポンス型は、
以下とする。

```ts
export type AssessObjectiveResponse = {
  data: ObjectiveAssessmentResult;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.post<AssessObjectiveResponse>(
    `/api/v1/objectives/${objectiveId}/assessments`,
    {
      targetYearMonth: '2026-08',
      nextMonthCreditCardPayment: 120000,
    },
  );
```

取得した判定結果は、
以下の表示に利用する。

- 目的達成判定結果
- 判定に使用した利用可能資産
- 判定時点の必要支出額
- 判定に使用した平均手取り収入
- 判定後の残額

---

### 26.1 targetYearMonthの扱い

`targetYearMonth`は、
判定の基準となる対象年月を
`YYYY-MM`形式で送信する。

```ts
const targetYearMonth = '2026-08';
```

対象年月は、
日付ではなく年月を表す業務値として扱う。

JavaScriptの`Date`オブジェクトへ
不必要に変換せず、
原則として文字列で保持する。

---

### 28.2 nextMonthCreditCardPaymentの扱い

`nextMonthCreditCardPayment`は、
翌月クレジットカード支払予定額を
日本円の整数値で指定する。

```ts
const nextMonthCreditCardPayment = 120000;
```

支払予定額がない場合は、
`0`を送信する。

```ts
const request: AssessObjectiveRequest = {
  targetYearMonth: '2026-08',
  nextMonthCreditCardPayment: 0,
};
```

`0`は有効な入力値であり、
未入力として扱ってはならない。

例えば、
以下の判定は行わない。

```ts
if (!request.nextMonthCreditCardPayment) {
  // 0円まで未入力として扱われるため使用しない
}
```

未入力判定は、
`undefined`などと明示的に比較する。

---

### 28.3 averageNetIncomeの扱い

`averageNetIncome`は、
バックエンドが判定対象年月より前の
連続する3か月の手取り収入から算出する。

フロントエンドから
平均手取り収入を送信しない。

また、
フロントエンドで独自に再計算せず、
APIから返却された値を
判定根拠として表示する。

これにより、
フロントエンドとバックエンドで
平均値や端数処理が異なることを防止する。

---

### 28.4 availableAssetsの扱い

`availableAssets`は、
判定対象年月時点の
利用可能資産設定および
確定済み月末資産状況から
バックエンドが算出する。

フロントエンドから
利用可能資産額を送信しない。

取得した値は、
判定結果の根拠として表示する。

```ts
const formattedAvailableAssets =
  new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY',
    maximumFractionDigits: 0,
  }).format(response.data.availableAssets);
```

---

### 28.5 assessmentResultの扱い

`assessmentResult`は、
API仕様で定義した判定結果の値として扱う。

フロントエンドでは、
文字列を直接画面へ表示するのではなく、
定義済みの値に応じて表示内容を切り替える。

例：

```ts
export type AssessmentResult =
  | 'achievable'
  | 'notAchievable';
```

```ts
const assessmentResultLabels: Record<
  AssessmentResult,
  string
> = {
  achievable: '達成可能',
  notAchievable: '達成困難',
};
```

実際に使用する値は、
OBJ-006のレスポンス項目および
`assessment_histories`の定義と一致させる。

未定義の値を受信した場合は、
正常な判定結果として扱わず、
共通エラー表示または
フォールバック表示を行う。

---

### 28.6 remainingAmountの扱い

`remainingAmount`は、
判定計算後に残る金額を表す。

日本円の整数値として扱い、
フロントエンドで再計算しない。

負数を許容する設計の場合は、
マイナス値もそのまま表示する。

```ts
const formattedRemainingAmount =
  new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY',
    maximumFractionDigits: 0,
  }).format(response.data.remainingAmount);
```

`remainingAmount`の正式名称および計算式は、
機能要件、
ユビキタス言語および
`assessment_histories`の定義と統一する。

---

### 28.7 判定実行前の画面制御

判定実行前に、
以下を確認できるようにする。

- 判定対象となる目的
- 判定対象年月
- 必要支出額
- 翌月クレジットカード支払予定額

無効化された目的では、
判定実行ボタンを非表示または非活性にする。

ただし、
フロントエンドの状態だけを信頼せず、
OBJ-006でも目的の利用状態を再検証する。

---

### 28.8 判定実行中の画面制御

判定処理中は、
実行ボタンを非活性にする。

判定が完了するまで、
同じリクエストを再送信できないようにする。

```ts
const [isAssessing, setIsAssessing] =
  useState(false);
```

```ts
if (isAssessing) {
  return;
}
```

本APIは判定成功ごとに
新しい判定履歴を登録するため、
二重送信によって
同じ内容の履歴が複数作成される可能性がある。

---

### 28.9 判定成功時の扱い

判定成功時は、
APIから返却された結果を使用して
判定結果画面を表示する。

判定結果をフロントエンドで再構築せず、
レスポンスを表示用データとして使用する。

必要に応じて、
以下の画面へ遷移できるようにする。

- 判定結果画面
- 判定履歴一覧画面
- 目的詳細画面

判定成功時には、
判定履歴も登録済みである。

---

### 28.10 判定不可時の扱い

判定条件を満たさない場合は、
システムエラーとして扱わず、
判定を実行できない業務状態として表示する。

エラーコードごとの基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `OBJECTIVE_DISABLED` | 無効化された目的は判定できないことを表示する |
| `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` | 月末資産状況の登録・確定を案内する |
| `NET_INCOME_DATA_INSUFFICIENT` | 必要な3か月分の手取り収入登録を案内する |
| `AVAILABLE_ASSETS_CALCULATION_NOT_AVAILABLE` | 利用可能資産設定の確認を案内する |
| `ASSESSMENT_NOT_AVAILABLE` | 判定条件を満たしていないことを表示する |
| `VALIDATION_ERROR` | 入力項目ごとにエラーを表示する |
| `OBJECTIVE_NOT_FOUND` | 目的一覧画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

判定不可の場合は、
判定結果画面へ遷移しない。

---

### 28.11 バリデーションエラーの表示

バリデーションエラーでは、
`error.details.field`を利用して
対象項目へエラーメッセージを表示する。

想定するフィールドは、
以下とする。

- `targetYearMonth`
- `nextMonthCreditCardPayment`

```ts
if (
  detail.field ===
  'nextMonthCreditCardPayment'
) {
  setFieldError(
    'nextMonthCreditCardPayment',
    detail.message,
  );
}
```

---

### 28.12 金額表示

以下の金額項目は、
すべて日本円の整数値として扱う。

- `availableAssets`
- `requiredExpense`
- `averageNetIncome`
- `nextMonthCreditCardPayment`
- `remainingAmount`

画面表示時は、
`Intl.NumberFormat`を使用して
桁区切りを行う。

フロントエンドで
独自の端数処理や
小数計算を行わない。

---

## 29. 設計上の補足

### 29.1 POSTを採用する理由

本APIは、
判定結果を返却するだけでなく、
判定成功時に
新しい判定履歴を登録する。

サーバー側の状態を変更するため、
GETではなくPOSTを採用する。

```http
POST /api/v1/objectives/{objectiveId}/assessments
```

---

### 29.2 assessmentsをサブリソースとする理由

判定履歴は、
特定の目的に対して作成される。

そのため、
目的に対する判定結果の作成として、
以下のURLを採用する。

```http
/objectives/{objectiveId}/assessments
```

単なる動詞URLである
`/assess-objective`などは採用しない。

---

### 29.3 判定入力をクライアントへ限定しない理由

クライアントから受け付ける変動値は、
判定対象年月および
翌月クレジットカード支払予定額とする。

以下の値は、
クライアントから受け付けない。

- 必要支出額
- 利用可能資産
- 平均手取り収入
- 判定結果
- 判定後の残額

これらは、
バックエンドが最新の基礎データから取得または算出する。

クライアントから送信された計算済みの値を信頼すると、
値の改変やデータ不整合が発生するためである。

---

### 29.4 確定済み月末資産状況のみ使用する理由

未確定データは、
入力途中または修正途中である可能性がある。

正式な資産状況を使用して判定するため、
確定済みの月末資産状況のみ利用する。

未確定データを推測したり、
過去月の値で補完したりしない。

---

### 29.5 判定対象年月時点の設定を使用する理由

利用可能資産へ含めるかどうかは、
対象年月によって異なる可能性がある。

現在の設定だけを参照すると、
過去の判定を正しく再現できない。

そのため、
判定対象年月時点で有効な
利用可能資産設定を使用する。

---

### 29.6 平均手取り収入を再計算する理由

INC-005で表示した平均手取り収入を、
クライアントから判定APIへ渡さない。

判定実行時に、
バックエンドで同じ計算ロジックを使用して
再計算する。

これにより、
以下の不整合を防止する。

- 表示後に手取り収入が更新された
- クライアントが値を改変した
- フロントエンドとバックエンドで端数処理が異なる

---

### 29.7 判定不可時に履歴を保存しない理由

判定に必要な情報が不足している場合は、
正式な判定結果を確定できない。

そのため、
判定不可という結果を
`assessment_histories`へ保存しない。

不足情報を登録または修正した後に、
改めて判定を実行する。

---

### 29.8 判定履歴をスナップショットとして保存する理由

判定履歴は、
判定実行時点の入力値、
計算根拠および
判定結果を保持する。

判定後に以下の情報が変更されても、
保存済み履歴は変更しない。

- 目的名
- 必要支出額
- 手取り収入
- 月末資産状況
- 利用可能資産設定
- 目的の利用状態

これにより、
過去の判定結果を
判定当時の内容で再現できる。

---

### 29.9 201 Createdを返却する理由

本APIの正常終了時には、
新しい判定履歴が作成される。

そのため、
`200 OK`ではなく
`201 Created`を返却する。

判定履歴の作成を伴わない
プレビューAPIを将来追加する場合は、
別APIとして設計する。

---

### 29.10 同一条件で複数回判定できる理由

利用者は、
同じ目的および同じ条件で
複数回判定できる。

判定履歴は、
判定が実行された事実を
時系列で保持するため、
同一条件の履歴が複数存在することを許可する。

---

### 29.11 冪等性キーを採用しない理由

Phase1では、
個人利用を前提とし、
判定処理中のボタン非活性化によって
意図しない二重送信を抑止する。

Idempotency-Keyを導入すると、
キーの保存、
有効期限、
同一キーのレスポンス再現などの
追加設計が必要になる。

Phase1では
実装コストに対する効果が小さいため、
採用しない。

---

### 29.12 フロントエンドで判定計算を行わない理由

目的達成判定は、
業務ルールとして
バックエンドへ集約する。

フロントエンドで同じ計算を実装すると、
計算式や端数処理の変更時に
複数箇所の修正が必要になる。

そのため、
Reactでは判定結果を表示するだけとし、
正式な判定計算は行わない。

---

### 29.13 実施予定年月を判定に使用しない理由

実施予定年月は、
Phase1では目的を管理・表示するための項目である。

将来時点の資産予測や
実施予定年月までの積立予測は行わない。

そのため、
目的達成判定の計算には使用しない。

---

### 29.14 キャッシュを採用しない理由

判定結果は、
手取り収入、
月末資産状況、
利用可能資産設定および
入力値によって変化する。

キャッシュを導入すると、
関連データ更新時の
無効化処理が複雑になる。

Phase1では判定頻度も高くないため、
キャッシュを採用しない。

---

### 29.15 同時更新に対する制約

Phase1では、
判定中に基礎データが更新される可能性を
完全には排除しない。

判定処理内では、
取得した入力値と計算結果を
同一の判定履歴として保存する。

厳密なスナップショット分離や
基礎データのロックが必要になった場合は、
トランザクション分離レベルや
楽観ロックの導入を再検討する。

---

## 30. 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [手取り収入API](./net-incomes.md)
- [月末資産API](./month-end-assets.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)
