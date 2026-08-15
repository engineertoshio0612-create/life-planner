### 1 概要

本ドキュメントでは、
ASM-002 目的達成判定履歴一覧取得APIに対する
バックエンドのテスト観点を定義する。

ASM-002では、
操作対象利用者の指定された目的に紐づく
`assessment_histories`を参照し、
保存済みの目的達成判定履歴を
一覧として取得する。

テストでは、
単にHTTPステータスやレスポンス形式を確認するだけではなく、
主に以下を確認する。

* 指定された目的に紐づく判定履歴のみ取得されること
* 達成可能・達成困難・判定不可のすべての履歴が取得対象となること
* 履歴が存在しない場合は`404`ではなく空配列として正常終了すること
* 新しい目的達成判定履歴から順に返却されること
* ASM-001で新たに作成された履歴が一覧へ反映されること
* 保存済みの判定結果をそのまま返却し、現在の業務データを使用して再判定しないこと
* 操作対象利用者以外の目的および判定履歴を取得しないこと
* 指定された目的以外の判定履歴が取得結果へ混入しないこと
* `page`および`perPage`によるページネーションが正しく機能すること
* 一覧取得によって目的達成判定やデータ更新などの副作用が発生しないこと
* 同一条件で複数回取得してもデータ状態が変化しないこと
* API共通方針に従った正常レスポンス・エラーレスポンスとなること

特に、
ASM-002は目的達成判定を実行するAPIではなく、
ASM-001によって保存された
目的達成判定履歴を参照するためのAPIである。

そのため、
目的、
月末資産状況、
月末資産残高、
商品別月末評価額、
手取り収入などの
現在の業務データが変更されていた場合でも、
保存済みの判定結果を再計算せず、
`assessment_histories`に保存されている内容を
そのまま取得することを確認する。

また、
本APIは参照のみを行う
冪等なAPIであるため、
実行によって`assessment_histories`を含む
業務データが登録・更新・削除されないことを確認する。

---

### 2 テスト観点

#### 2.1 正常系

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

#### 2.2 達成可能の履歴

`result = achievable`の
目的達成判定履歴を用意する。

以下を確認する。

- 一覧取得対象に含まれること
- `result`が`achievable`として返却されること

---

#### 2.3 達成困難の履歴

`result = difficult`の
目的達成判定履歴を用意する。

以下を確認する。

- 一覧取得対象に含まれること
- `result`が`difficult`として返却されること

---

#### 2.4 判定不可の履歴

`result = unassessable`の
目的達成判定履歴を用意する。

以下を確認する。

- 一覧取得対象に含まれること
- 一覧から除外されないこと
- `result`が`unassessable`として返却されること

---

#### 2.5 複数種類の判定結果

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

#### 2.6 履歴0件

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

#### 2.7 並び順

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

#### 2.8 ASM-001実行後の一覧

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

#### 2.9 過去履歴を再計算しないこと

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

#### 2.10 objectiveId形式不正

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

#### 2.11 目的不存在

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

#### 2.12 他利用者の目的

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

#### 2.13 論理削除済みの目的

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

#### 2.14 利用者コンテキスト未指定

`X-User-Id`を
指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

---

#### 2.15 利用者ID形式不正

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

#### 2.16 利用者不存在

存在しない利用者IDを
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 2.17 論理削除済み利用者

論理削除済み利用者を
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 2.18 page未指定

`page`を指定せずに
リクエストする。

以下を確認する。

- 1ページ目として扱われること
- `200 OK`となること
- `meta.currentPage = 1`となること

---

#### 2.19 page正常値

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

#### 2.20 page形式不正

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

#### 2.21 perPage未指定

`perPage`を指定しない場合、
API共通方針で定めた
デフォルト件数が使用されること。

---

#### 2.22 perPage正常値

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

#### 2.23 perPageが0

```text
perPage = 0
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 2.24 perPageが負数

```text
perPage = -1
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 2.25 perPageが小数

```text
perPage = 1.5
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 2.26 perPageが文字列

```text
perPage = abc
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 2.27 perPageが最大件数を超える

API共通方針で定めた
最大件数を超える値を指定する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 2.28 ページネーション1ページ目

目的達成判定履歴を、
`perPage`を超える件数用意する。

1ページ目を取得し、
以下を確認する。

- `data`件数が`perPage`以下であること
- `meta.currentPage`が正しいこと
- `meta.total`が全件数と一致すること
- `meta.lastPage`が正しいこと

---

#### 2.29 ページネーション2ページ目

複数ページ分の
目的達成判定履歴を用意する。

2ページ目を取得し、
以下を確認する。

- 1ページ目と異なる履歴が返却されること
- 並び順が維持されていること
- ページネーション情報が正しいこと

---

#### 2.30 ページ範囲外

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

#### 2.31 他の目的の履歴を取得しないこと

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

#### 2.32 他利用者の履歴を取得しないこと

User AとUser Bについて、
それぞれ目的と
目的達成判定履歴を用意する。

User Aを操作対象として
ASM-002を実行する。

以下を確認する。

- User Aの目的に紐づく履歴のみ取得されること
- User Bの履歴が取得結果へ混入しないこと

---

#### 2.33 副作用

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

#### 2.34 冪等性

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

#### 2.35 レスポンス契約

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

#### 2.36 返却しない情報

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

#### 2.37 エラーレスポンス

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

### 3 関連ドキュメント

- [ASM-002 API詳細設計](../../api/details/assessments/asm-002-create.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [ASM-002 Laravelアーキテクチャ設計](../../architecture/laravel/assessments/asm-002-create.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [ASM-002 Reactアーキテクチャ設計](../../architecture/react/assessments/asm-002-create.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
