# OBJ-005 目的無効化

## 1. 概要

操作対象となる利用者に登録された
有効な目的を無効化する。

本APIでは、
目的の利用状態を
`enabled = true`から
`enabled = false`へ変更する。

目的名、
実施予定年月、
必要支出額および
メモなどの目的情報は変更しない。

目的の無効化は論理削除とは異なる
業務上の状態変更として扱い、
`deleted_at`は更新しない。

そのため、
無効化された目的は
新しい目的達成判定の対象外となるが、
目的一覧および目的詳細から
引き続き参照できる。

また、
無効化以前に保存された
目的達成判定履歴については、
更新、削除および再計算を行わない。

主に以下を確認する。

- 有効な目的を無効化できること
- `enabled`のみが`false`へ変更されること
- 無効化に伴って`updated_at`が更新されること
- 目的の通常属性および`deleted_at`が変更されないこと
- 操作対象利用者に帰属する目的のみ無効化できること
- 論理削除済みの目的を無効化できないこと
- 無効化済みの目的を再度更新しないこと
- Request Bodyから利用状態や目的情報を変更できないこと
- 同時無効化時にもデータ整合性が維持されること
- 他の目的へ副作用が発生しないこと
- 過去の目的達成判定履歴が変更されないこと
- 無効化後の目的が新しい目的達成判定の対象外となること
- 無効化後も目的一覧および目的詳細から参照できること
- 正常時および異常時のレスポンスがAPI仕様に従っていること

---

## 2. テスト観点

OBJ-005では、正常な無効化だけでなく、

* 利用者境界
* 論理削除
* 無効化済み状態
* 更新対象
* 同時実行
* 副作用
* レスポンス契約

を重点的に確認する。

---

### 2.1 正常系

以下の状態を用意する。

```text
objective.id = 1
objective.user_id = 1
objective.enabled = true
objective.deleted_at = NULL
```

以下を実行する。

```http
PATCH /api/v1/objectives/1/disabled
X-User-Id: 1
```

期待結果：

```http
200 OK
```

となること。

---

### 2.2 enabledがfalseになること

実行後、

```text
objectives.enabled
    = false
```

となることを確認する。

---

### 2.3 updated_atが更新されること

正常な無効化によって、

```text
updated_at
```

が更新されることを確認する。

---

### 2.4 通常属性が変更されないこと

無効化前後で、以下が変更されないことを確認する。

* `name`
* `planned_year_month`
* `required_expense`
* `memo`
* `user_id`

---

### 2.5 deleted_atが変更されないこと

無効化後も、

```text
deleted_at = NULL
```

であることを確認する。

目的無効化が論理削除として実装されていないことを確認する。

---

### 2.6 正常レスポンス

正常時に、概念的に以下の形式となることを確認する。

```json
{
  "data": {
    "id": "1",
    "name": "一人暮らし",
    "plannedYearMonth": "2027-04",
    "requiredExpense": 500000,
    "memo": "引っ越し費用を含む",
    "enabled": false
  }
}
```

---

### 2.7 idの型

DB上のIDが`bigint`であっても、

```json
{
  "id": "1"
}
```

のようにstringで返却されることを確認する。

---

### 2.8 enabledの型

レスポンスの

```text
enabled
```

がbooleanであることを確認する。

```json
{
  "enabled": false
}
```

となり、

```json
{
  "enabled": "false"
}
```

とはならないこと。

---

### 2.9 memoがnullの場合

目的の`memo`が`NULL`の場合に、

```json
{
  "memo": null
}
```

として返却されることを確認する。

---

### 2.10 X-User-Id未指定

`X-User-Id`を指定せずOBJ-005を実行する。

期待結果：

```text
USER_CONTEXT_REQUIRED
```

となること。

目的が更新されないことも確認する。

---

### 2.11 X-User-Id形式不正

例えば、

```http
X-User-Id: abc
```

を指定する。

期待結果：

```text
400 Bad Request
INVALID_USER_ID
```

となること。

---

### 2.12 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 2.13 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

目的無効化処理へ進まないこと。

---

### 2.14 objectiveId形式不正

API共通方針で正の整数IDを前提とする場合は、例えば以下を確認する。

```text
0
-1
abc
1.5
10abc
```

期待結果：

```text
400 Bad Request
INVALID_OBJECTIVE_ID
```

となること。

---

### 2.15 目的不存在

存在しない`objectiveId`を指定する。

期待結果：

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

となること。

---

### 2.16 他利用者の目的

以下の状態を用意する。

```text
User A
    Objective A

User B
    Objective B
```

User AとしてObjective Bを指定する。

期待結果：

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

となること。

Objective Bの

```text
enabled
```

が変更されないことを確認する。

---

### 2.17 論理削除済み目的

以下の目的を用意する。

```text
deleted_at IS NOT NULL
```

OBJ-005を実行する。

期待結果：

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

となること。

---

### 2.18 無効化済み目的

以下の目的を用意する。

```text
enabled = false
deleted_at = NULL
```

OBJ-005を実行する。

期待結果：

```text
409 Conflict
OBJECTIVE_DISABLED
```

となること。

---

### 2.19 無効化済み目的を再更新しない

`OBJECTIVE_DISABLED`の場合に、`updated_at`が不要に更新されないことを確認する。

つまり、

```text
enabled = false
```

のレコードへ再度UPDATEを発行しないことを確認する。

---

### 2.20 Request Bodyなし

以下のようにRequest Bodyを送信せずに正常実行できることを確認する。

```http
PATCH /api/v1/objectives/1/disabled
X-User-Id: 1
```

---

### 2.21 enabledをRequest Bodyから指定しない

以下のようなRequestをAPI仕様として使用しないことを確認する。

```json
{
  "enabled": false
}
```

OBJ-005はエンドポイント自体によって無効化を表現する。

未定義項目を拒否するAPI共通方針の場合は、Bodyを送信した場合にバリデーションエラーとなることも確認する。

---

### 2.22 nameを変更できない

例えば、不正に以下を送信しても、

```json
{
  "name": "変更後名称"
}
```

`name`が変更されないことを確認する。

---

### 2.23 userIdを変更できない

Requestから任意の`userId`を指定しても、目的の

```text
user_id
```

が変更されないことを確認する。

---

### 2.24 同時無効化

同一目的に対して2つのOBJ-005をほぼ同時に実行する。

初期状態：

```text
enabled = true
```

期待結果：

```text
Request A
    → 200 OK

Request B
    → 409 Conflict
       OBJECTIVE_DISABLED
```

または実行順序が逆になっても、一方のみが更新成功すること。

---

### 2.25 同時実行後の状態

同時実行後も、

```text
enabled = false
```

となっていることを確認する。

中間状態や不正な値にならないこと。

---

### 2.26 更新件数

正常系では、UPDATE対象が1件だけであることを確認する。

他の目的レコードへ副作用がないこと。

---

### 2.27 他目的非更新

同一利用者に複数目的が存在する場合でも、指定した`objectiveId`以外の

```text
enabled
updated_at
```

が変更されないことを確認する。

---

### 2.28 assessment_histories非更新

対象目的に過去の

```text
assessment_histories
```

が存在する状態でOBJ-005を実行する。

以下を確認する。

* レコード件数が変化しない
* 過去判定結果が変更されない
* 削除されない
* 新規判定履歴が作成されない

---

### 2.29 新しい目的達成判定の対象外

OBJ-005成功後、目的達成判定APIを実行した場合に、無効化済み目的が新規判定対象にならないことを関連API側のテストで確認する。

OBJ-005自身が判定処理を実行しないことも確認する。

---

### 2.30 目的一覧との連携

OBJ-005成功後にOBJ-001を再取得し、OBJ-001の仕様に従って

```text
enabled = false
```

が反映されることを確認する。

無効化済みを含む一覧の場合は無効状態として表示され、有効目的だけの一覧の場合は対象外となることを確認する。

---

### 2.31 目的詳細との連携

OBJ-005成功後にOBJ-003で対象目的を取得し、

```text
enabled = false
```

となっていることを確認する。

無効化後も詳細参照可能というOBJ-003の仕様と整合していることを確認する。

---

### 2.32 再有効化されないこと

OBJ-005を何度実行しても、

```text
enabled = true
```

へ戻らないことを確認する。

また、OBJ-005のRequestから`enabled = true`を指定して再有効化できないことを確認する。

---

### 2.33 副作用範囲

OBJ-005実行前後で、業務上変更されるのが

```text
objectives.enabled
objectives.updated_at
```

だけであることを確認する。

以下は変更されないこと。

* `objectives.user_id`
* `objectives.name`
* `objectives.planned_year_month`
* `objectives.required_expense`
* `objectives.memo`
* `objectives.created_at`
* `objectives.deleted_at`
* `assessment_histories`

---

### 2.34 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_OBJECTIVE_ID
OBJECTIVE_NOT_FOUND
OBJECTIVE_DISABLED
INTERNAL_SERVER_ERROR
```

以下も確認する。

* `error.code`が設定される
* `error.message`が設定される
* `requestId`が共通仕様に従って設定される
* SQLが含まれない
* DB内部エラーが含まれない
* スタックトレースが含まれない
* サーバーファイルパスが含まれない

---

### 2.35 INTERNAL_SERVER_ERROR

目的無効化処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

となること。

以下も確認する。

* 他の目的が変更されないこと
* 内部情報がレスポンスへ公開されないこと
* サーバーログに調査情報が記録されること

---

### 2.36 冪等的な最終状態

同じ目的に対してOBJ-005を複数回呼び出しても、最終的な業務状態が

```text
enabled = false
```

から変化しないことを確認する。

2回目以降に新たな副作用が発生しないことも確認する。

---

## 3. 関連ドキュメント

- [OBJ-005 API詳細設計](../../api/details/objectives/obj-005-assessments.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [OBJ-005 Laravelアーキテクチャ設計](../../architecture/laravel/objectives/obj-005-assessments.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [OBJ-005 Reactアーキテクチャ設計](../../architecture/react/objectives/obj-005-assessments.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
