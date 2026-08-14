##  ASM-002 目的達成判定履歴一覧取得

### 1 概要

操作対象となる利用者について、
指定された目的に紐づく
目的達成判定履歴の一覧を取得する。

本APIでは、
ASM-001 目的達成判定実行APIによって
作成された
`assessment_histories`を参照する。

目的達成判定そのものは
実行しない。

また、
既存の目的達成判定履歴を
登録・更新・削除しない。

取得対象は、
指定された目的に紐づく
目的達成判定履歴のみとする。

---

### 2 ユースケース

利用者は、
指定した目的について、
過去に実行した
目的達成判定の履歴を確認する。

例えば、
以下のような場合に使用する。

- 過去の目的達成判定結果を一覧で確認する
- 判定結果の推移を確認する
- 達成可能・達成困難・判定不可の履歴を確認する
- 特定の判定履歴詳細を開く前に一覧を表示する
- ASM-001実行後に最新の判定履歴を確認する

本APIでは、
過去の判定結果を
現在の資産状況や
手取り収入を使用して
再計算しない。

保存されている
目的達成判定履歴を
そのまま取得する。

---

### 3 エンドポイント

```http
GET /api/v1/objectives/{objectiveId}/assessments
```

---

### 4 HTTPメソッド

```http
GET
```

本APIは、
指定された目的に紐づく
目的達成判定履歴を取得する。

データの登録、
更新および削除は行わない。

---

### 5 認証・利用者の扱い

Phase1では、
認証機能を実装しない。

操作対象となる利用者は、
`X-User-Id`
リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

指定された利用者に属する
目的についてのみ、
目的達成判定履歴を取得できる。

他の利用者に属する目的について、
目的達成判定履歴を
取得することはできない。

目的を取得する際は、
必ず以下を検索条件に含める。

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
```

指定された`objectiveId`が
他の利用者に属する場合は、
対象となる目的が
存在しないものとして扱う。

目的達成判定履歴は、
指定された目的を経由して
利用者境界を保証する。

```text
assessment_histories.objective_id
    = objectives.id

AND

objectives.user_id
    = 操作対象利用者ID
```

`assessment_histories`自体に
`user_id`を保持しない場合でも、

```text
assessment_histories
    ↓
objectives
    ↓
users
```

の関連によって
操作対象利用者との
利用者境界を確認する。

目的達成判定履歴を
以下のように
`objective_id`だけで取得してはならない。

```text
assessment_histories.objective_id
    = objectiveId
```

事前に、
指定された目的が
操作対象利用者に属していることを
確認したうえで、
その目的に紐づく履歴のみを取得する。

利用者IDは、
クエリパラメータまたは
パスパラメータでは受け付けない。

利用者IDは、
ミドルウェアで設定された
利用者コンテキストから取得する。

`X-User-Id`が指定されていない場合、
形式が不正な場合、
または指定された利用者が存在しない場合は、
API共通方針に従って
エラーを返却する。

本APIでは、
他の利用者に属する

- 目的
- 目的達成判定履歴

の存在を
レスポンスから推測できないようにする。

---

### 6 パスパラメータ

| パラメータ名 | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `objectiveId` | string | ○ | 目的達成判定履歴を取得する対象の目的ID |

リクエスト例：

```http
GET /api/v1/objectives/5/assessments
```

`objectiveId`は、目的達成判定履歴の取得対象となる目的を一意に識別するIDである。

目的達成判定履歴は、指定された目的に紐づく履歴のみを取得する。

---

### 7 クエリパラメータ

本APIでは、目的達成判定履歴一覧のページネーションに使用するクエリパラメータを受け付ける。

| パラメータ名 | 型 | 必須 | デフォルト | 説明 |
| --- | --- | :---: | --- | --- |
| `page` | integer | × | `1` | 取得するページ番号 |
| `perPage` | integer | × | API共通方針に従う | 1ページあたりの取得件数 |

リクエスト例：

```http
GET /api/v1/objectives/5/assessments?page=1&perPage=20
```

並び順は、サーバー側で固定する。

クライアントから任意のソート条件は受け付けない。

目的達成判定履歴は、新しい判定履歴から確認できるよう、判定実行順の降順で返却する。

具体的な並び順に使用するカラムは、`assessment_histories`のテーブル定義に従う。

---

### 8 リクエストヘッダー

#### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
| --- | :---: | --- |
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/objectives/5/assessments?page=1&perPage=20
Accept: application/json
X-User-Id: 1
```

本APIはGETのため、`Content-Type`は必須としない。

---

### 9 リクエストボディ

なし。

本APIでは、リクエストボディを使用しない。

取得対象となる目的は、パスパラメータの`objectiveId`から特定する。

ページネーション条件は、クエリパラメータから取得する。

---

### 10 リクエスト項目

本APIでは、リクエストボディに業務項目を持たない。

利用者IDは、`X-User-Id`から取得する。

取得対象となる目的は、`objectiveId`から特定する。

一覧取得条件としてクライアントから以下を受け付けない。

- `userId`
- `result`
- 判定対象年月
- 資産額
- 手取り収入
- 任意の並び順

Phase1では、指定された目的に紐づく目的達成判定履歴をページネーションして取得する。

---

### 11 バリデーション

#### 11.1 objectiveId

`objectiveId`は、必須のパスパラメータとする。

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

正常例：

```text
1
5
123
```

不正例：

```text
0
-1
abc
1.5
```

`objectiveId`の形式が不正な場合は、`VALIDATION_ERROR`として扱う。

---

#### 11.2 page

`page`が指定された場合は、以下を検証する。

- integerであること
- 1以上であること

正常例：

```text
1
2
10
```

不正例：

```text
0
-1
abc
1.5
```

`page`が未指定の場合は、`1`として扱う。

形式または値が不正な場合は、`VALIDATION_ERROR`として扱う。

---

#### 11.3 perPage

`perPage`が指定された場合は、API共通方針で定めたページサイズのバリデーションを適用する。

少なくとも、以下を検証する。

- integerであること
- 1以上であること
- API共通方針で定めた最大件数以下であること

正常例：

```text
10
20
50
```

不正例：

```text
0
-1
abc
1.5
```

API共通方針で最大件数が`100`の場合の例：

```text
101
```

`perPage`が未指定の場合は、API共通方針で定めたデフォルト件数を使用する。

形式または値が不正な場合は、`VALIDATION_ERROR`として扱う。

---

#### 11.4 目的の存在確認

指定された`objectiveId`について、以下の条件を満たす目的が存在することを確認する。

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
```

対象となる目的が存在しない場合は、`OBJECTIVE_NOT_FOUND`として扱う。

他の利用者に属する目的IDが指定された場合も、同じエラーとして扱う。

これにより、他の利用者に属する目的の存在をレスポンスから判別できないようにする。

論理削除済みの目的を通常の取得対象としない場合も、`OBJECTIVE_NOT_FOUND`として扱う。

---

#### 11.5 目的達成判定履歴の存在

指定された目的について、目的達成判定履歴が1件も存在しない場合は、エラーとしない。

```text
目的あり
+
assessment_histories 0件
    ↓
200 OK
空配列
```

存在しない履歴を理由として、`404 Not Found`を返却しない。

一覧取得APIでは、対象リソースである目的が存在すれば、履歴0件も正常な状態として扱う。

---

#### 11.6 ページ範囲外

指定された`page`が総ページ数を超えている場合の扱いは、API共通ページネーション方針に従う。

Phase1では、対象ページにデータが存在しない場合も、一覧取得自体は正常終了とする。

概念的には、以下とする。

```text
page = 総ページ数を超える
    ↓
200 OK
data = []
```

ページネーション情報については、共通レスポンス形式に従う。

---

#### 11.7 並び順

目的達成判定履歴の並び順は、サーバー側で固定する。

クライアントからソート項目や昇順・降順を指定させない。

一覧では、原則として新しい目的達成判定履歴を先頭に返却する。

概念的には、以下のいずれかを使用する。

```text
created_at DESC
```

または、

```text
id DESC
```

正式な並び順は、`assessment_histories`のテーブル定義およびAPI共通方針に従う。

---

#### 11.8 X-User-Id

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

`X-User-Id`が指定されていない場合は、`USER_CONTEXT_REQUIRED`として扱う。

形式が不正な場合は、`INVALID_USER_ID`として扱う。

指定された利用者が存在しない場合、または論理削除されている場合は、`USER_NOT_FOUND`として扱う。

---

#### 11.9 業務状態に依存する検証

以下は、単項目バリデーションではなく、業務ルールとして扱う。

- 指定された目的が操作対象利用者に属していること
- 論理削除済みの目的を取得対象としないこと
- 指定された目的に紐づく履歴のみを取得すること
- 他の利用者に属する目的達成判定履歴を取得しないこと
- 履歴0件を正常な一覧結果として扱うこと
- 並び順をサーバー側で固定すること
- ページネーションをAPI共通方針に従って適用すること

入力形式の不正による`VALIDATION_ERROR`と、履歴が存在しない状態は明確に区別して扱う。

---

### 12 業務ルール

#### 12.1 取得対象となる目的

目的達成判定履歴は、
`objectiveId`で指定された
目的に対して取得する。

対象となる目的は、
必ず操作対象利用者に
属していなければならない。

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
```

他の利用者に属する目的を
取得対象としてはならない。

---

#### 12.2 目的達成判定履歴の利用者境界

目的達成判定履歴の
利用者境界は、
所属する目的を経由して確認する。

```text
assessment_histories.objective_id
    = objectives.id

AND

objectives.id
    = objectiveId

AND

objectives.user_id
    = 操作対象利用者ID
```

`assessment_histories`に
`user_id`を保持しない場合でも、
`objectives`を経由して
利用者境界を保証する。

他の利用者に属する
目的達成判定履歴を
取得してはならない。

---

#### 12.3 保存済みの判定履歴を取得する

本APIでは、
ASM-001 目的達成判定実行APIによって
保存された
目的達成判定履歴を取得する。

一覧取得時に、
目的達成判定を
再実行しない。

```text
assessment_histories
    ↓
保存済みデータを取得
    ↓
レスポンス
```

現在の

- 目的
- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 手取り収入

を使用して、
過去の判定結果を
再計算してはならない。

---

#### 12.4 過去の判定結果をそのまま扱う

目的達成判定履歴は、
判定実行時点の結果を保持する。

その後、
判定元となったデータが
変更されていても、
保存済みの判定結果を
変更しない。

例えば、

```text
1回目
result = difficult

    ↓
資産状況変更

2回目
result = achievable
```

となった場合でも、
1回目の履歴は
`difficult`のまま保持する。

本APIでは、
それぞれ独立した履歴として取得する。

---

#### 12.5 判定不可の履歴

ASM-001によって
「判定不可」として保存された履歴も、
通常の目的達成判定履歴として
取得対象に含める。

```text
achievable
difficult
unassessable
```

のいずれであっても、
保存済みの履歴であれば
一覧取得対象とする。

判定不可の履歴を
一覧から除外しない。

---

#### 12.6 履歴が存在しない場合

指定された目的は存在するが、
目的達成判定履歴が
1件も存在しない場合は、
エラーとしない。

```text
目的あり
+
assessment_histories = 0件
    ↓
200 OK
data = []
```

`OBJECTIVE_NOT_FOUND`は、
目的自体が存在しない場合に使用する。

目的達成判定履歴が
存在しないことを理由として、
`404 Not Found`を返却しない。

---

#### 12.7 並び順

目的達成判定履歴は、
新しい履歴から
確認できる順序で返却する。

Phase1では、
判定履歴の登録順を基準として
降順で取得する。

概念的には、
以下とする。

```text
assessment_histories.id DESC
```

これにより、
直近に実行された
目的達成判定履歴を
一覧の先頭に表示する。

クライアントから
任意の並び順は受け付けない。

---

#### 12.8 ページネーション

目的達成判定履歴一覧には、
API共通方針で定めた
ページネーションを適用する。

クライアントは、
以下を指定できる。

```text
page
perPage
```

`page`が未指定の場合は、
1ページ目を取得する。

`perPage`が未指定の場合は、
API共通方針で定めた
デフォルト件数を使用する。

1ページあたりの最大件数も、
API共通方針に従う。

---

#### 12.9 ページ範囲外

指定された`page`に
目的達成判定履歴が
存在しない場合も、
エラーとしない。

```text
指定ページに履歴なし
    ↓
200 OK
data = []
```

ページネーション情報は、
実際の検索結果に基づいて返却する。

---

#### 12.10 目的の現在状態による除外を行わない

目的が取得可能な状態である限り、
保存済みの目的達成判定履歴は
取得対象とする。

ASM-001実行時点と
現在とで、
目的の内容が変更されていても、
過去の履歴を
一覧から除外しない。

目的達成判定履歴は、
過去に判定を実行した事実として扱う。

---

#### 12.11 一覧取得時に他テーブルを再評価しない

本APIでは、
目的達成判定履歴を取得するために、
以下の業務情報を
再評価しない。

- 月末資産状況が現在も確定済みか
- 月末資産残高が現在も存在するか
- 商品別月末評価額が現在も存在するか
- 資産口座が現在も有効か
- 保有商品が現在も有効か
- 手取り収入が現在も3ヶ月分存在するか
- 現在の資産額で目的を達成可能か

これらは、
ASM-001実行時の
判定処理に関する責務である。

ASM-002では、
保存済みの判定履歴を
取得することに責務を限定する。

---

### 13 取得条件

目的達成判定履歴の
取得条件は、
以下とする。

```text
assessment_histories.objective_id
    = objectiveId
```

ただし、
履歴取得前に、
対象目的について
以下の利用者境界を確認する。

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
```

概念的な取得順序は、
以下とする。

```text
操作対象利用者の確認
    ↓
目的の取得
    ↓
目的の利用者境界確認
    ↓
目的達成判定履歴を取得
    ↓
id DESC
    ↓
ページネーション
    ↓
レスポンス
```

他の利用者に属する目的については、
目的達成判定履歴の検索を行わない。

---

### 14 取得対象

取得対象は、
指定された目的に紐づく
`assessment_histories`とする。

```text
assessment_histories.objective_id
    = objectiveId
```

判定結果による
絞り込みは行わない。

そのため、
以下の判定結果を
すべて取得対象とする。

```text
achievable
difficult
unassessable
```

Phase1では、
クエリパラメータによる

```text
result = achievable
```

などの
判定結果絞り込みは提供しない。

---

### 15 レスポンス

取得成功時は、
目的達成判定履歴の一覧を
返却する。

HTTPステータスは、

```http
200 OK
```

とする。

レスポンスは、
API共通方針で定めた
Envelope形式および
ページネーション形式を使用する。

例：

```json
{
  "data": [
    {
      "id": "15",
      "objectiveId": "5",
      "result": "achievable"
    },
    {
      "id": "12",
      "objectiveId": "5",
      "result": "difficult"
    },
    {
      "id": "8",
      "objectiveId": "5",
      "result": "unassessable"
    }
  ],
  "meta": {
    "currentPage": 1,
    "perPage": 20,
    "total": 3,
    "lastPage": 1
  }
}
```

実際のページネーション項目は、
API共通方針で定めた
共通レスポンス形式に従う。

---

#### 15.1 履歴0件

目的達成判定履歴が
存在しない場合は、
`data`を空配列として返却する。

例：

```json
{
  "data": [],
  "meta": {
    "currentPage": 1,
    "perPage": 20,
    "total": 0,
    "lastPage": 1
  }
}
```

履歴0件は、
正常な一覧取得結果として扱う。

---

#### 15.2 ページ範囲外

指定されたページに
履歴が存在しない場合も、
`data`を空配列として返却する。

```json
{
  "data": [],
  "meta": {
    "currentPage": 10,
    "perPage": 20,
    "total": 3,
    "lastPage": 1
  }
}
```

実際の`meta`の値は、
API共通ページネーション方針および
LaravelのPaginatorの扱いに従う。

---

### 16 レスポンス項目

目的達成判定履歴1件あたりの
レスポンス項目は、
以下とする。

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `id` | string | × | 目的達成判定履歴ID |
| `objectiveId` | string | × | 判定対象となった目的ID |
| `result` | string | × | 目的達成判定結果 |

IDは、
API共通方針に従って
stringとして返却する。

`result`は、
目的達成判定結果として
定義された値を返却する。

Phase1では、
概念的に以下を使用する。

```text
achievable
difficult
unassessable
```

実際の値は、
バックエンド側のEnum、
テーブル定義書および
API共通定義に従う。

---

#### 16.1 id

`id`は、
目的達成判定履歴を
一意に識別するIDである。

データベース上で
`bigint`として保持している場合でも、
APIレスポンスでは
stringとして返却する。

例：

```json
{
  "id": "15"
}
```

---

#### 16.2 objectiveId

`objectiveId`は、
目的達成判定履歴が紐づく
目的IDである。

```text
assessment_histories.objective_id
```

を、
APIレスポンスでは
camelCaseへ変換して返却する。

```json
{
  "objectiveId": "5"
}
```

IDは、
stringとして返却する。

---

#### 16.3 result

`result`は、
ASM-001実行時に保存された
目的達成判定結果を返却する。

例：

```json
{
  "result": "achievable"
}
```

一覧取得時に
現在の業務データを使用して
再計算しない。

保存されている判定結果を
そのままAPIレスポンスへ変換する。

---

#### 16.4 返却しない情報

本APIでは、
目的達成判定履歴一覧として
必要な情報のみを返却する。

以下の情報は返却しない。

- `user_id`
- 目的の詳細情報
- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 資産口座情報
- 保有商品情報
- 手取り収入
- 判定時に使用した計算途中の値
- `created_at`
- `updated_at`

目的達成判定履歴の詳細情報が
別途必要となる場合は、
目的達成判定履歴詳細取得APIの
責務として扱う。

一覧APIでは、
一覧表示に必要な情報へ
レスポンス項目を限定する。

---

### 17 エラーレスポンス

異常時は、
API共通方針で定めた
共通エラーレスポンス形式を使用する。

本APIでは、
主に以下のエラーを扱う。

- 利用者コンテキストが指定されていない
- 利用者IDの形式が不正
- 指定された利用者が存在しない
- `objectiveId`の形式が不正
- 指定された目的が存在しない
- ページネーションパラメータが不正
- 想定外のサーバーエラー

目的自体は存在するが、
目的達成判定履歴が
1件も存在しない場合は、
APIエラーとはしない。

```text
目的あり
+
目的達成判定履歴0件
    ↓
200 OK
data = []
```

---

#### 17.1 利用者コンテキストが指定されていない場合

`X-User-Id`が
指定されていない場合は、
`USER_CONTEXT_REQUIRED`
を返却する。

```json
{
  "error": {
    "code": "USER_CONTEXT_REQUIRED",
    "message": "操作対象の利用者を指定してください。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

目的および
目的達成判定履歴の取得は行わない。

---

#### 17.2 利用者ID形式が不正な場合

`X-User-Id`が
API共通方針で定めたID形式に
一致しない場合は、
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

#### 17.3 利用者が存在しない場合

指定された利用者が存在しない場合、
または論理削除されている場合は、
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

#### 17.4 objectiveIdのバリデーションエラー

`objectiveId`が
バリデーション条件を満たさない場合は、
`VALIDATION_ERROR`
を返却する。

例えば、
以下の場合を含む。

```text
objectiveId = 0
objectiveId = -1
objectiveId = abc
objectiveId = 1.5
```

レスポンス例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
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

#### 17.5 pageのバリデーションエラー

`page`が
バリデーション条件を満たさない場合は、
`VALIDATION_ERROR`
を返却する。

例えば、
以下の場合を含む。

```text
page = 0
page = -1
page = abc
page = 1.5
```

レスポンス例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "page",
        "reason": "invalidFormat",
        "message": "ページ番号を確認してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

#### 17.6 perPageのバリデーションエラー

`perPage`が
API共通方針で定めた
バリデーション条件を満たさない場合は、
`VALIDATION_ERROR`
を返却する。

例えば、
以下の場合を含む。

```text
perPage = 0
perPage = -1
perPage = abc
perPage = 1.5
perPage = 最大件数超過
```

レスポンス例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "perPage",
        "reason": "invalidValue",
        "message": "1ページあたりの取得件数を確認してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

#### 17.7 目的が存在しない場合

以下の条件を満たす
目的が存在しない場合は、
`OBJECTIVE_NOT_FOUND`
を返却する。

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
```

レスポンス例：

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

以下の場合を含む。

- 指定された`objectiveId`が存在しない
- 指定された目的が他の利用者に属している
- 指定された目的が論理削除されている

他利用者の目的が
存在すること自体を
レスポンスから判別できないようにする。

---

#### 17.8 目的達成判定履歴が存在しない場合

指定された目的が存在し、
その目的に紐づく
目的達成判定履歴が
1件も存在しない場合は、
APIエラーとはしない。

```text
目的あり
+
assessment_histories = 0件
    ↓
200 OK
```

レスポンス例：

```json
{
  "data": [],
  "meta": {
    "currentPage": 1,
    "perPage": 20,
    "total": 0,
    "lastPage": 1
  }
}
```

目的達成判定履歴が
存在しないことを理由として、
`OBJECTIVE_NOT_FOUND`や
その他のエラーコードを返却しない。

---

#### 17.9 指定ページに履歴が存在しない場合

指定された`page`が
総ページ数を超えている場合など、
指定ページに
目的達成判定履歴が存在しなくても
APIエラーとはしない。

```text
指定ページに履歴なし
    ↓
200 OK
data = []
```

ページ範囲外を理由として
`404 Not Found`を返却しない。

---

#### 17.10 想定外のエラーが発生した場合

データベース接続エラーなど、
想定外のサーバーエラーが発生した場合は、
`INTERNAL_SERVER_ERROR`
を返却する。

```json
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "サーバー内部でエラーが発生しました。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

SQL、
スタックトレース、
PostgreSQLの制約名および
内部例外メッセージは、
レスポンスへ含めない。

---

### 18 HTTPステータス

| HTTPステータス | 条件 |
|---:|---|
| `200 OK` | 目的達成判定履歴一覧の取得に成功した |
| `400 Bad Request` | 利用者コンテキストが未指定、または利用者IDの形式が不正である |
| `404 Not Found` | 指定された利用者または目的が存在しない |
| `422 Unprocessable Entity` | `objectiveId`、`page`、`perPage`のバリデーションエラー |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

---

#### 18.1 200の扱い

目的達成判定履歴一覧を
正常に取得できた場合は、
`200 OK`
を返却する。

以下の場合も
`200 OK`とする。

- 目的達成判定履歴が0件
- 指定ページに目的達成判定履歴が0件
- 判定不可の履歴のみが存在する
- 達成可能・達成困難・判定不可が混在している

一覧にデータが存在するかどうかと、
API処理の成否は
分けて扱う。

---

#### 18.2 400の扱い

以下の場合は、
`400 Bad Request`
を返却する。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正である

---

#### 18.3 404の扱い

以下の場合は、
`404 Not Found`
を返却する。

- 指定された利用者が存在しない
- 指定された利用者が論理削除されている
- 指定された目的が存在しない
- 指定された目的が他の利用者に属している
- 指定された目的が論理削除されている

目的達成判定履歴が
0件であることを理由として、
`404 Not Found`を返却しない。

---

#### 18.4 422の扱い

以下の入力値が
バリデーション条件を満たさない場合は、
`422 Unprocessable Entity`
を返却する。

- `objectiveId`
- `page`
- `perPage`

---

### 19 エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `USER_CONTEXT_REQUIRED` | 400 | `X-User-Id`が指定されていない | × |
| `INVALID_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `USER_NOT_FOUND` | 404 | 指定された利用者が存在しない、または論理削除されている | × |
| `VALIDATION_ERROR` | 422 | `objectiveId`、`page`、`perPage`が不正である | × |
| `OBJECTIVE_NOT_FOUND` | 404 | 目的が存在しない、論理削除済み、または他利用者に属している | × |
| `INTERNAL_SERVER_ERROR` | 500 | 想定外のサーバーエラーが発生した | ○ |

以下は、
エラーコードの対象としない。

- 目的達成判定履歴が0件
- 指定ページの履歴が0件
- 判定結果が`unassessable`である履歴が存在する

エラーコードの正式な定義は、
`error-codes.md`
に従う。

---

### 20 副作用

本APIは、
目的達成判定履歴を
参照するだけの
読み取り専用APIである。

データベースへの
登録・更新・削除は行わない。

本APIの実行によって、
以下の業務データを変更してはならない。

- `users`
- `objectives`
- `assessment_histories`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `net_incomes`

また、
以下の処理を自動実行しない。

- 目的達成判定
- 目的達成判定履歴の新規登録
- 目的達成判定履歴の更新
- 目的達成判定履歴の削除
- 過去の判定結果の再計算
- 現在の資産状況を使用した判定結果の更新
- 月末資産状況の作成・確定・確定解除
- 月末資産残高の登録・更新
- 商品別月末評価額の登録・更新
- 手取り収入の登録・更新

---

### 21 トランザクション

本APIは、
読み取り専用APIであり、
データベースへの
登録・更新・削除を行わない。

そのため、
Phase1では
明示的なトランザクションを
使用しない。

概念的には、
以下の処理のみを行う。

```text
操作対象利用者確認
    ↓
目的取得
    ↓
目的達成判定履歴検索
    ↓
ページネーション
    ↓
レスポンス
```

一覧取得のためだけに、

```php
DB::transaction(...)
```

を使用しない。

---

#### 21.1 ロック

本APIでは、
データベースレコードを更新しないため、
行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

目的達成判定履歴一覧の
参照によって、
ASM-001などの更新処理を
不要にブロックしてはならない。

---

#### 21.2 一覧取得中のデータ変更

一覧取得と同時に、
ASM-001によって
新しい目的達成判定履歴が
登録される可能性がある。

Phase1では、
ページネーション中の
完全なスナップショット整合性までは
保証しない。

例えば、

```text
1ページ目取得
    ↓
ASM-001で新しい履歴追加
    ↓
2ページ目取得
```

となった場合、
offsetベースのページネーションでは
取得結果が前後する可能性がある。

Phase1では、
API共通方針で採用している
ページネーション方式に従い、
この挙動を許容する。

将来的に、
大量の目的達成判定履歴を扱い、
ページ間の一貫性が
重要となった場合は、
カーソルベースページネーションなどを
別途検討する。

---

### 22 冪等性

本APIは、
読み取り専用のGET APIである。

同一のデータ状態に対して
同一のリクエストを複数回実行しても、
サーバー上の業務データを変更しない。

そのため、
本APIは冪等である。

例えば、

```text
1回目
GET /api/v1/objectives/5/assessments
    ↓
200 OK

2回目
GET /api/v1/objectives/5/assessments
    ↓
200 OK
```

本APIの実行によって、
目的達成判定履歴が
追加・更新・削除されることはない。

ただし、
ASM-001など別のAPIによって
目的達成判定履歴が変更された場合は、
同じGETリクエストでも
返却内容が変化する可能性がある。

これは、
GET APIの冪等性を損なうものではない。

Phase1では、
本APIに
`Idempotency-Key`は使用しない。

GET APIであり、
リクエストの再実行による
業務データの重複登録が
発生しないためである。

---

### 23 関連テーブル

#### 23.1 objectives

目的達成判定履歴一覧の
取得対象となる目的を保持する。

本APIでは、
`objectiveId`で指定された目的について、
操作対象利用者との
利用者境界を確認するために参照する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 目的ID |
| `user_id` | 利用者境界確認 |
| `deleted_at` | 論理削除状態の確認 |

取得条件は、
以下とする。

```text
id = objectiveId
AND
user_id = 操作対象利用者ID
AND
deleted_at IS NULL
```

他の利用者に属する目的は、
存在しないものとして扱う。

本APIでは、
`objectives`を更新しない。

---

#### 23.2 assessment_histories

目的達成判定の
実行結果を履歴として保持する。

本APIで取得する
主要テーブルである。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 目的達成判定履歴ID |
| `objective_id` | 目的との関連 |
| `result` | 目的達成判定結果 |
| `created_at` | 必要に応じた登録日時管理 |
| `updated_at` | 更新日時管理 |

取得条件は、
以下とする。

```text
objective_id = objectiveId
```

ただし、
事前に`objectives`を参照し、
対象目的が
操作対象利用者に属することを
確認する。

並び順は、
Phase1では
新しい履歴を先頭とする。

概念的には、
以下とする。

```text
ORDER BY id DESC
```

本APIでは、
`assessment_histories`を
登録・更新・削除しない。

---

#### 23.3 users

操作対象となる
利用者を保持する。

`X-User-Id`で指定された利用者が
存在することを確認するために参照する。

概念的には、
以下を確認する。

```text
id = X-User-Id
AND
deleted_at IS NULL
```

本APIでは、
`users`を更新しない。

---

#### 23.4 month_end_asset_snapshots

本APIでは、
`month_end_asset_snapshots`を
目的達成判定履歴一覧取得のために
参照しない。

目的達成判定履歴は、
ASM-001実行時点の結果として
`assessment_histories`へ保存されているため、
一覧取得時に
月末資産状況を再評価しない。

本APIでは、
`month_end_asset_snapshots`を更新しない。

---

#### 23.5 month_end_asset_balances

本APIでは、
`month_end_asset_balances`を
参照・登録・更新・削除しない。

過去の目的達成判定結果を
現在の月末資産残高から
再計算しない。

---

#### 23.6 month_end_holding_values

本APIでは、
`month_end_holding_values`を
参照・登録・更新・削除しない。

過去の目的達成判定結果を
現在の商品別月末評価額から
再計算しない。

---

#### 23.7 asset_accounts

本APIでは、
`asset_accounts`を
参照・登録・更新・削除しない。

残高記録単位などを
一覧取得時に再評価しない。

---

#### 23.8 asset_account_available_settings

本APIでは、
`asset_account_available_settings`を
参照・登録・更新・削除しない。

過去の目的達成判定履歴について、
現在または過去の
資産口座利用可否を
一覧取得時に再判定しない。

---

#### 23.9 holding_assets

本APIでは、
`holding_assets`を
参照・登録・更新・削除しない。

過去の目的達成判定結果について、
現在の保有商品状態を
再評価しない。

---

#### 23.10 net_incomes

本APIでは、
`net_incomes`を
参照・登録・更新・削除しない。

過去の目的達成判定結果について、
現在の手取り収入を使用して
再計算しない。

---

### 24 関連する機能要件

- 目的管理
  - 利用者は自身に属する目的を参照できる
  - 他の利用者に属する目的は参照できない
  - 論理削除済みの目的は通常の取得対象としない

- 目的達成判定
  - ASM-001実行時に目的達成判定履歴を保存する
  - 判定結果として達成可能、達成困難、判定不可を保持できる
  - 判定不可の履歴も通常の目的達成判定履歴として扱う
  - 過去の目的達成判定履歴は上書きしない
  - 現在の資産状況や手取り収入を使用して過去履歴を再計算しない

- 目的達成判定履歴
  - 指定した目的に紐づく判定履歴を一覧取得できる
  - 判定履歴が0件の場合も正常な一覧結果として扱う
  - 新しい履歴から確認できる順序で返却する
  - ページネーションを適用する

- 利用者境界
  - 操作対象利用者に属する目的の判定履歴のみ取得できる
  - 他の利用者に属する目的達成判定履歴を取得できない
  - 他の利用者に属する目的の存在をレスポンスから推測できない

- API共通
  - IDはAPIレスポンス上stringとして扱う
  - JSONフィールド名はcamelCaseとする
  - 一覧取得には共通ページネーション形式を使用する
  - エラー時は共通エラーレスポンス形式を使用する

具体的な章番号は、
`functional-requirements.md`の
最新定義に従う。

---

### 25 テスト観点

#### 25.1 正常系

指定された目的に
目的達成判定履歴が複数存在する状態で、
ASM-002を実行する。

以下を確認する。

- `200 OK`となること
- 指定された目的の履歴のみ取得されること
- `data`が配列で返却されること
- ページネーション情報が`meta`に返却されること
- 各履歴の`id`がstringであること
- 各履歴の`objectiveId`がstringであること
- 各履歴の`result`が正しく返却されること
- データベースが更新されないこと

---

#### 25.2 達成可能の履歴

`result = achievable`の
目的達成判定履歴を用意する。

以下を確認する。

- 一覧取得対象に含まれること
- `result`が`achievable`として返却されること

---

#### 25.3 達成困難の履歴

`result = difficult`の
目的達成判定履歴を用意する。

以下を確認する。

- 一覧取得対象に含まれること
- `result`が`difficult`として返却されること

---

#### 25.4 判定不可の履歴

`result = unassessable`の
目的達成判定履歴を用意する。

以下を確認する。

- 一覧取得対象に含まれること
- 一覧から除外されないこと
- `result`が`unassessable`として返却されること

---

#### 25.5 複数種類の判定結果

以下の履歴を
同一目的に用意する。

```text
achievable
difficult
unassessable
```

以下を確認する。

- すべて取得対象となること
- 判定結果による暗黙の絞り込みが行われないこと

---

#### 25.6 履歴0件

目的は存在するが、
`assessment_histories`が
0件の状態で実行する。

期待結果：

```text
200 OK
data = []
```

以下を確認する。

- `404 Not Found`にならないこと
- `OBJECTIVE_NOT_FOUND`にならないこと
- `data`が空配列であること
- `meta.total = 0`であること

---

#### 25.7 並び順

同一目的について、
複数の履歴を作成する。

例えば、

```text
id = 10
id = 11
id = 12
```

期待結果：

```text
12
11
10
```

の順序で返却されること。

新しい目的達成判定履歴が
先頭となること。

---

#### 25.8 ASM-001実行後の一覧

ASM-001を実行して
新しい目的達成判定履歴を
1件追加する。

その後、
ASM-002を実行する。

以下を確認する。

- 新しく作成された履歴が取得されること
- 最新履歴が一覧の先頭となること
- 過去履歴も保持されていること

---

#### 25.9 過去履歴を再計算しないこと

目的達成判定履歴を作成した後に、
以下のいずれかを変更する。

- 目的
- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 手取り収入

その後、
ASM-002を実行する。

以下を確認する。

- 保存済みの`result`がそのまま返却されること
- 現在の業務データを使用して過去履歴を再計算しないこと
- `assessment_histories`が更新されないこと

---

#### 25.10 objectiveId形式不正

以下のような
不正な`objectiveId`を指定する。

```text
0
-1
abc
1.5
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

目的達成判定履歴を
取得しないこと。

---

#### 25.11 目的不存在

存在しない
`objectiveId`を指定する。

期待結果：

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

目的達成判定履歴を
返却しないこと。

---

#### 25.12 他利用者の目的

User Aを
`X-User-Id`として指定し、
User Bに属する
`objectiveId`を指定する。

期待結果：

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

以下を確認する。

- User Bの目的が存在することをレスポンスから判別できないこと
- User Bの目的達成判定履歴が返却されないこと

---

#### 25.13 論理削除済みの目的

論理削除済みの目的を
`objectiveId`へ指定する。

期待結果：

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

その目的に紐づく
目的達成判定履歴が存在していても、
通常のASM-002では返却されないこと。

---

#### 25.14 利用者コンテキスト未指定

`X-User-Id`を
指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

---

#### 25.15 利用者ID形式不正

不正な
`X-User-Id`を指定する。

例：

```http
X-User-Id: abc
```

期待結果：

```text
400 Bad Request
INVALID_USER_ID
```

---

#### 25.16 利用者不存在

存在しない利用者IDを
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 25.17 論理削除済み利用者

論理削除済み利用者を
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 25.18 page未指定

`page`を指定せずに
リクエストする。

以下を確認する。

- 1ページ目として扱われること
- `200 OK`となること
- `meta.currentPage = 1`となること

---

#### 25.19 page正常値

以下のような
正常な`page`を指定する。

```text
1
2
3
```

指定したページに応じた
データが返却されること。

---

#### 25.20 page形式不正

以下を指定する。

```text
0
-1
abc
1.5
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 25.21 perPage未指定

`perPage`を指定しない場合、
API共通方針で定めた
デフォルト件数が使用されること。

---

#### 25.22 perPage正常値

API共通方針の
許容範囲内の値を指定する。

例えば、

```text
10
20
50
```

以下を確認する。

- 指定した件数以下の履歴が返却されること
- `meta.perPage`へ指定値が反映されること

---

#### 25.23 perPageが0

```text
perPage = 0
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 25.24 perPageが負数

```text
perPage = -1
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 25.25 perPageが小数

```text
perPage = 1.5
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 25.26 perPageが文字列

```text
perPage = abc
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 25.27 perPageが最大件数を超える

API共通方針で定めた
最大件数を超える値を指定する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 25.28 ページネーション1ページ目

目的達成判定履歴を、
`perPage`を超える件数用意する。

1ページ目を取得し、
以下を確認する。

- `data`件数が`perPage`以下であること
- `meta.currentPage`が正しいこと
- `meta.total`が全件数と一致すること
- `meta.lastPage`が正しいこと

---

#### 25.29 ページネーション2ページ目

複数ページ分の
目的達成判定履歴を用意する。

2ページ目を取得し、
以下を確認する。

- 1ページ目と異なる履歴が返却されること
- 並び順が維持されていること
- ページネーション情報が正しいこと

---

#### 25.30 ページ範囲外

総ページ数を超える
`page`を指定する。

期待結果：

```text
200 OK
data = []
```

以下を確認する。

- `404 Not Found`にならないこと
- 空配列として正常終了すること

---

#### 25.31 他の目的の履歴を取得しないこと

同一利用者に、
Objective Aと
Objective Bを用意する。

それぞれに
目的達成判定履歴を登録する。

Objective Aの
ASM-002を実行する。

以下を確認する。

- Objective Aの履歴のみ返却されること
- Objective Bの履歴が混入しないこと

---

#### 25.32 他利用者の履歴を取得しないこと

User AとUser Bについて、
それぞれ目的と
目的達成判定履歴を用意する。

User Aを操作対象として
ASM-002を実行する。

以下を確認する。

- User Aの目的に紐づく履歴のみ取得されること
- User Bの履歴が取得結果へ混入しないこと

---

#### 25.33 副作用

ASM-002実行前後で、
以下のテーブルが
変更されていないことを確認する。

- `users`
- `objectives`
- `assessment_histories`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `net_incomes`

目的達成判定が
自動実行されていないことも確認する。

---

#### 25.34 冪等性

同一のデータ状態で
同じリクエストを
複数回実行する。

```text
1回目
GET /api/v1/objectives/5/assessments
    ↓
200 OK

2回目
GET /api/v1/objectives/5/assessments
    ↓
200 OK
```

以下を確認する。

- データベースが変更されないこと
- 目的達成判定履歴が増加しないこと
- 目的達成判定が再実行されないこと
- 同一状態であれば同じ取得結果となること

---

#### 25.35 レスポンス契約

正常取得時に、
API共通方針で定めた
Envelope形式および
ページネーション形式で
返却されること。

概念例：

```json
{
  "data": [
    {
      "id": "15",
      "objectiveId": "5",
      "result": "achievable"
    }
  ],
  "meta": {
    "currentPage": 1,
    "perPage": 20,
    "total": 1,
    "lastPage": 1
  }
}
```

以下を確認する。

- `data`がarrayであること
- `meta`がobjectであること
- `id`がstringであること
- `objectiveId`がstringであること
- `result`が定義された値であること
- JSONフィールド名がcamelCaseであること

---

#### 25.36 返却しない情報

本APIでは、
以下の情報が
レスポンスへ含まれていないことを確認する。

- `user_id`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `holding_assets`
- `net_incomes`
- `created_at`
- `updated_at`

一覧表示に不要な
判定内部データを返却しないこと。

---

#### 25.37 エラーレスポンス

各異常系について、
API共通方針で定めた
共通エラーレスポンス形式で
返却されることを確認する。

以下を確認する。

- `error.code`が期待するエラーコードであること
- `error.message`が設定されていること
- `error.details`が配列であること
- `error.requestId`が設定されていること
- SQLが含まれていないこと
- PostgreSQLの制約名が含まれていないこと
- スタックトレースが含まれていないこと
- 内部例外メッセージが含まれていないこと

---

### 26 Laravel実装方針

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

#### 26.1 Action

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

#### 26.2 Request

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

#### 26.3 UseCase

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

#### 26.4 ObjectiveQuery

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

#### 26.5 目的不存在

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

#### 26.6 AssessmentHistoryQuery

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

#### 26.7 利用者境界

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

#### 26.8 並び順

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

#### 26.9 created_atを使用する場合

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

#### 26.10 ページネーション

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

#### 26.11 pageの扱い

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

#### 26.12 perPageの扱い

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

#### 26.13 履歴0件

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

#### 26.14 判定結果による絞り込みを行わない

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

#### 26.15 過去履歴を再計算しない

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

#### 26.16 取得カラム

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

#### 26.17 N+1問題

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

#### 26.18 AssessmentHistory Model

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

#### 26.19 resultのCast

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

#### 26.20 API Resource

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

#### 26.21 Resource Collection

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

#### 26.22 Responder

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

#### 26.23 Responderの責務

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

#### 26.24 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* JSONレスポンス共通処理
* 共通例外処理
* ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

#### 26.25 トランザクション

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

#### 26.26 ロック

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

#### 26.27 同時登録との扱い

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

#### 26.28 キャッシュ

Phase1では、ASM-002専用のサーバー側アプリケーションキャッシュを使用しない。

ASM-001実行後に新しい判定履歴を一覧へ反映できるよう、データベースから最新状態を取得する。

将来的に履歴件数やアクセス量が増加した場合は、別途キャッシュ戦略を検討する。

---

#### 26.29 例外

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

#### 26.30 想定外例外

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

#### 26.31 ログ

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

#### 26.32 テスト実装方針

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

### 28 設計上の補足

#### 28.1 GETを採用する理由

ASM-002は、
保存済みの
目的達成判定履歴一覧を
取得するだけのAPIである。

データの登録、
更新または削除を行わないため、
HTTPメソッドには
`GET`を採用する。

---

#### 28.2 目的配下のリソースとする理由

目的達成判定履歴は、
特定の目的に紐づく。

そのため、

```http
/objectives/{objectiveId}/assessments
```

として、
目的配下のサブリソースとして表現する。

ASM-001と同じURLを使用し、
HTTPメソッドによって
責務を分ける。

```text
POST
    → 目的達成判定実行

GET
    → 目的達成判定履歴一覧取得
```

---

#### 28.3 履歴0件を404としない理由

一覧取得において、
対象となる目的が存在していれば、
目的達成判定履歴が
0件である状態は正常である。

そのため、

```text
目的あり
+
履歴なし
```

を

```text
404 Not Found
```

とはしない。

空配列によって
0件を表現する。

---

#### 28.4 判定結果による絞り込みを提供しない理由

Phase1では、
目的達成判定履歴の件数は
大規模にならないことを想定する。

そのため、

```text
result
```

による検索条件は
提供しない。

すべての判定結果を
新しい順に確認できる
シンプルな一覧APIとする。

必要性が生じた場合は、
将来拡張として
フィルタ条件を追加する。

---

#### 28.5 並び順を固定する理由

目的達成判定履歴では、
利用者が直近の判定結果を
確認するケースが多い。

そのため、
新しい履歴を先頭にする。

クライアントから
任意のソート条件を受け付けず、
API側で順序を固定することで、
画面ごとの並び順差異を防止する。

---

#### 28.6 ページネーションを採用する理由

目的達成判定を
繰り返し実行すると、
`assessment_histories`は
継続的に増加する。

すべての履歴を
毎回一括取得すると、
将来的にレスポンスサイズが
増加する。

そのため、
Phase1から
API共通ページネーションを
適用する。

---

#### 28.7 過去履歴を再計算しない理由

目的達成判定履歴は、
ASM-001実行時点の
判定結果を保持する。

一覧取得時に
現在データから再計算すると、
過去の履歴としての意味が
失われる。

そのため、
ASM-002は
保存済みの結果を
そのまま返却する。

---

#### 28.8 判定不可も一覧に含める理由

判定不可も、
ASM-001を実行した結果である。

例えば、

```text
当時は手取り収入が不足していた
```

という状態も、
目的達成判定履歴として意味がある。

そのため、
判定不可の履歴も
通常の履歴として取得する。

---

#### 28.9 一覧レスポンスを軽量化する理由

ASM-002は、
目的達成判定履歴を
一覧表示するためのAPIである。

そのため、
各履歴について
必要最低限の情報だけを返却する。

詳細な判定情報まで
一覧レスポンスへ含めると、
レスポンスサイズが増加し、
一覧と詳細の責務も曖昧になる。

詳細情報は、
ASM-003 目的達成判定履歴詳細取得APIで
取得する。

---

#### 28.10 フロントエンドで判定を再計算しない理由

目的達成判定ロジックを
React側でも実装すると、
バックエンドとの
判定結果の差異が発生する可能性がある。

ASM-002では、
保存済みの`result`を
表示するだけとする。

目的達成判定の
業務ロジックは、
バックエンドへ集約する。

---

#### 28.11 GETの冪等性

ASM-002は、
同じリクエストを
複数回実行しても
業務データを変更しない。

そのため、
本APIは冪等である。

別APIによって
目的達成判定履歴が追加された場合は、
返却結果が変化する可能性があるが、
ASM-002自体が
データを変更したものではない。

---

#### 28.12 Idempotency-Keyを使用しない理由

ASM-002は
読み取り専用のGET APIである。

再実行しても
データの重複登録などが
発生しないため、
`Idempotency-Key`は使用しない。

---

#### 28.13 サーバーキャッシュを採用しない理由

ASM-001実行後は、
新しい目的達成判定履歴を
即時確認できる必要がある。

Phase1では、
実装を単純に保つため、
ASM-002専用の
サーバーキャッシュは採用しない。

必要になった場合は、
将来的に
アクセス量や履歴件数を確認したうえで
検討する。

---

### 29 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [ASM-001 目的達成判定プレビュー](./asm-001-preview.md)
- [ASM-003 判定履歴一覧取得](./asm-003-list.md)
- [ASM-003 判定履歴詳細取得](./asm-004-detail.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/assessments.md)
- [Reactアーキテクチャ設計](../../../architecture/react/assessments.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)