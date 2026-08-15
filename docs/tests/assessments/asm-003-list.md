### 1 概要

本ドキュメントでは、
ASM-003 目的達成判定履歴詳細取得APIに対する
バックエンドのテスト観点を定義する。

ASM-003では、
操作対象利用者の指定された目的に紐づく
`assessment_histories`から、
指定された目的達成判定履歴1件を取得する。

テストでは、
単にHTTPステータスやレスポンス形式を確認するだけではなく、
主に以下を確認する。

* 指定された目的に紐づく指定された判定履歴1件のみ取得されること
* 達成可能・達成困難・判定不可の保存済み判定結果がそのまま返却されること
* 指定された目的とは異なる目的の判定履歴を取得できないこと
* 操作対象利用者以外の目的および判定履歴を取得できないこと
* 論理削除済みの目的を通じて判定履歴を取得できないこと
* ASM-001を複数回実行した場合でも、最新履歴ではなく指定された履歴が取得されること
* ASM-002の一覧取得結果から指定した履歴を正しく詳細取得できること
* 現在の目的、資産状況、手取り収入などを使用して過去の判定結果を再計算しないこと
* 詳細取得によって目的達成判定やデータ更新などの副作用が発生しないこと
* 同一条件で複数回取得してもデータ状態が変化しないこと
* 目的達成判定履歴の取得に不要な業務テーブルを参照しないこと
* API共通方針に従った正常レスポンス・エラーレスポンスとなること

特に、
ASM-003は現在の状態から
目的達成可否を再判定するAPIではなく、
ASM-001によって作成された
保存済みの目的達成判定履歴を
参照するためのAPIである。

そのため、
履歴作成後に目的、
月末資産状況、
月末資産残高、
商品別月末評価額、
手取り収入などが変更されていた場合でも、
現在の業務データを使用して再計算せず、
`assessment_histories`に保存されている
判定結果をそのまま返却することを確認する。

また、
本APIは参照のみを行う
冪等なAPIであるため、
実行によって新しい判定履歴が作成されたり、
既存の判定履歴やその他の業務データが
更新・削除されたりしないことを確認する。

---

### 2 テスト観点

#### 2.1 正常系

指定された目的に紐づく
目的達成判定履歴が存在する状態で、
ASM-003を実行する。

以下を確認する。

- `200 OK`となること
- 指定された目的達成判定履歴1件が返却されること
- `id`がstringであること
- `objectiveId`がstringであること
- `result`が正しく返却されること
- データベースが更新されないこと
- 目的達成判定が再実行されないこと

---

#### 2.2 達成可能の履歴

`result = achievable`の
目的達成判定履歴を用意する。

期待結果：

```text
200 OK
result = achievable
```

保存済みの判定結果が
そのまま返却されること。

---

#### 2.3 達成困難の履歴

`result = difficult`の
目的達成判定履歴を用意する。

期待結果：

```text
200 OK
result = difficult
```

保存済みの判定結果が
そのまま返却されること。

---

#### 2.4 判定不可の履歴

`result = unassessable`の
目的達成判定履歴を用意する。

期待結果：

```text
200 OK
result = unassessable
```

以下を確認する。

- APIエラーとならないこと
- 判定不可の履歴を正常取得できること
- 現在の業務データを使用して再判定しないこと

---

#### 2.5 objectiveId形式不正

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

#### 2.6 assessmentId形式不正

以下のような
不正な`assessmentId`を指定する。

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

#### 2.7 objectiveIdとassessmentIdの両方が不正

`objectiveId`および
`assessmentId`の両方に
不正値を指定する。

以下を確認する。

- `422 Unprocessable Entity`となること
- `VALIDATION_ERROR`となること
- `error.details`に対象項目が適切に含まれること

---

#### 2.8 目的不存在

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

#### 2.9 他利用者の目的

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

#### 2.10 論理削除済みの目的

論理削除済みの目的を
`objectiveId`へ指定する。

期待結果：

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

その目的に紐づく
目的達成判定履歴が存在していても、
通常のASM-003では返却されないこと。

---

#### 2.11 目的達成判定履歴不存在

目的は存在するが、
存在しない`assessmentId`を指定する。

期待結果：

```text
404 Not Found
ASSESSMENT_HISTORY_NOT_FOUND
```

---

#### 2.12 同一利用者の別目的に属する履歴

以下の状態を用意する。

```text
User A
├─ Objective A
│   └─ Assessment History 10
│
└─ Objective B
    └─ Assessment History 20
```

以下を実行する。

```http
GET /api/v1/objectives/{ObjectiveA}/assessments/20
```

期待結果：

```text
404 Not Found
ASSESSMENT_HISTORY_NOT_FOUND
```

以下を確認する。

- Objective Bの履歴が返却されないこと
- 履歴ID`20`が存在することをレスポンスから判別できないこと

---

#### 2.13 他利用者の履歴

User AとUser Bについて、
それぞれ目的および
目的達成判定履歴を用意する。

User Aを操作対象として、
User Bの`objectiveId`を指定する。

期待結果：

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

User Bの履歴が
返却されないこと。

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

#### 2.18 現在の目的が変更されている

ASM-001で
目的達成判定履歴を作成した後に、
目的の内容を変更する。

その後、
ASM-003で
過去の履歴を取得する。

以下を確認する。

- 保存済みの`result`がそのまま返却されること
- 現在の目的内容を使用して再計算しないこと
- `assessment_histories`が更新されないこと

---

#### 2.19 現在の資産状況が変更されている

ASM-001で
目的達成判定履歴を作成した後に、
以下のいずれかを変更する。

- 月末資産状況
- 月末資産残高
- 商品別月末評価額

その後、
ASM-003を実行する。

以下を確認する。

- 保存済みの`result`がそのまま返却されること
- 現在の資産額を使用して再判定しないこと

---

#### 2.20 現在の手取り収入が変更されている

ASM-001で
目的達成判定履歴を作成した後に、
`net_incomes`を変更する。

その後、
ASM-003を実行する。

以下を確認する。

- 保存済みの`result`がそのまま返却されること
- 現在の手取り収入を使用して再判定しないこと
- `net_incomes`を本APIのために参照しないこと

---

#### 2.21 判定不可条件が解消されている

過去に
`result = unassessable`として
保存された履歴を用意する。

その後、
判定材料を追加し、
現在は判定可能な状態にする。

ASM-003を実行する。

期待結果：

```text
200 OK
result = unassessable
```

過去の履歴を
現在の状態に合わせて
`achievable`や`difficult`へ
変更しないこと。

---

#### 2.22 ASM-001再実行後の過去履歴

同一目的について、
ASM-001を複数回実行し、
複数の履歴を作成する。

例えば、

```text
id = 10
result = difficult

id = 20
result = achievable
```

ASM-003で
`assessmentId = 10`を指定する。

期待結果：

```text
result = difficult
```

最新履歴ではなく、
指定された履歴が
正しく返却されること。

---

#### 2.23 ASM-002からの詳細取得

ASM-002で取得した
履歴一覧の1件について、

```text
objectiveId
assessmentId
```

を使用して
ASM-003を実行する。

以下を確認する。

- ASM-002の対象履歴と同じ`id`が返却されること
- `objectiveId`が一致すること
- `result`が一致すること

---

#### 2.24 副作用

ASM-003実行前後で、
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

以下も確認する。

- 目的達成判定が自動実行されていないこと
- 新しい目的達成判定履歴が作成されていないこと
- 保存済み履歴が更新されていないこと

---

#### 2.25 冪等性

同一のデータ状態で
同じリクエストを
複数回実行する。

```text
1回目
GET /api/v1/objectives/5/assessments/20
    ↓
200 OK

2回目
GET /api/v1/objectives/5/assessments/20
    ↓
200 OK
```

以下を確認する。

- データベースが変更されないこと
- 目的達成判定履歴が増加しないこと
- 目的達成判定が再実行されないこと
- 同一状態であれば同じ取得結果となること

---

#### 2.26 レスポンス契約

正常取得時に、
API共通方針で定めた
Envelope形式で返却されること。

概念例：

```json
{
  "data": {
    "id": "20",
    "objectiveId": "5",
    "result": "achievable"
  }
}
```

以下を確認する。

- `data`がobjectであること
- `id`がstringであること
- `objectiveId`がstringであること
- `result`が定義された値であること
- JSONフィールド名がcamelCaseであること
- ページネーション用の`meta`が含まれないこと

---

#### 2.27 返却しない情報

本APIでは、
以下の情報が
レスポンスへ含まれていないことを確認する。

- `user_id`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `holding_assets`
- `asset_account_available_settings`
- `net_incomes`
- `created_at`
- `updated_at`

現在の業務データや
判定内部データを
付加して返却しないこと。

---

#### 2.28 エラーレスポンス

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

#### 2.29 不要なテーブルを参照しないこと

ASM-003実行時に、
必要に応じてSQLログなどを確認し、
以下のテーブルが
目的達成判定履歴詳細取得のために
不要に参照されていないことを確認する。

- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `net_incomes`

ASM-003の取得対象は、
原則として

```text
users
objectives
assessment_histories
```

の範囲に限定する。

---

### 3 関連ドキュメント

- [ASM-003 API詳細設計](../../api/details/assessments/asm-003-list.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [ASM-003 Laravelアーキテクチャ設計](../../architecture/laravel/assessments/asm-003-list.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [ASM-003 Reactアーキテクチャ設計](../../architecture/react/assessments/asm-003-list.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [目的達成判定 テスト設計](./README.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
