##  ASM-003 目的達成判定履歴詳細取得

### 1 概要

操作対象となる利用者について、
指定された目的に紐づく
特定の目的達成判定履歴を取得する。

本APIでは、
ASM-001 目的達成判定実行APIによって
作成された
`assessment_histories`の
保存済みデータを参照する。

目的達成判定そのものは
実行しない。

また、
既存の目的達成判定履歴を
登録・更新・削除しない。

取得対象は、
指定された目的に紐づく
指定された目的達成判定履歴1件のみとする。

過去の目的達成判定結果について、
現在の目的、
資産状況、
手取り収入などを使用して
再計算しない。

---

### 2 ユースケース

利用者は、
過去に実行した
特定の目的達成判定履歴について、
保存されている判定結果を
詳細に確認する。

例えば、
以下のような場合に使用する。

- ASM-002 目的達成判定履歴一覧取得APIから特定の履歴を選択して詳細を確認する
- 過去の目的達成判定結果を確認する
- 達成可能・達成困難・判定不可の結果を詳細画面で確認する
- 判定実行時点で保存されている情報を確認する
- 過去の判定結果と現在の状態を区別して確認する

本APIでは、
目的達成判定結果を
現在の業務データから
再生成しない。

保存されている
目的達成判定履歴を
そのまま取得する。

---

### 3 エンドポイント

```http
GET /api/v1/objectives/{objectiveId}/assessments/{assessmentId}
```

---

### 4 HTTPメソッド

```http
GET
```

本APIは、
指定された目的に紐づく
特定の目的達成判定履歴を取得する。

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
指定された目的に紐づく
履歴のみを取得する。

取得条件は、
概念的に以下とする。

```text
assessment_histories.id = assessmentId
AND
assessment_histories.objective_id = objectiveId
```

ただし、
履歴取得前に、
指定された目的が
操作対象利用者に属していることを
確認する。

目的達成判定履歴自体に
`user_id`を保持しない場合でも、

```text
assessment_histories
    ↓
objectives
    ↓
users
```

の関連によって
利用者境界を保証する。

他の利用者に属する目的の
`assessmentId`を指定しても、
目的達成判定履歴を
取得できない。

また、
同一利用者に属する
別の目的の判定履歴についても、
`objectiveId`と
`assessmentId`の組み合わせが
一致しない場合は取得できない。

例えば、

```text
objectiveId = 5
assessmentId = 20
```

を指定した場合、

```text
assessment_histories.id = 20
AND
assessment_histories.objective_id = 5
```

を満たす履歴のみを
取得対象とする。

以下のように、
`assessmentId`だけで
目的達成判定履歴を
取得してはならない。

```text
assessment_histories.id = assessmentId
```

これにより、

- 他の利用者に属する目的の履歴
- 同一利用者の別目的に属する履歴

を誤って取得することを防止する。

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
| `objectiveId` | string | ○ | 目的達成判定履歴が紐づく目的ID |
| `assessmentId` | string | ○ | 取得対象となる目的達成判定履歴ID |

リクエスト例：

```http
GET /api/v1/objectives/5/assessments/20
```

`objectiveId`は、
目的達成判定履歴が紐づく
目的を一意に識別するIDである。

`assessmentId`は、
取得対象となる
目的達成判定履歴を
一意に識別するIDである。

本APIでは、
`assessmentId`だけで
目的達成判定履歴を特定しない。

以下の組み合わせによって、
取得対象を特定する。

```text
objectiveId
+
assessmentId
```

---

### 7 クエリパラメータ

なし。

本APIでは、
クエリパラメータを使用しない。

以下のような
取得条件は受け付けない。

- 判定結果
- 判定対象年月
- 利用者ID
- 資産額
- 手取り収入

取得対象は、
パスパラメータの
`objectiveId`および
`assessmentId`によって特定する。

---

### 8 リクエストヘッダー

#### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/objectives/5/assessments/20
Accept: application/json
X-User-Id: 1
```

本APIはGETのため、
`Content-Type`は必須としない。

---

### 9 リクエストボディ

なし。

本APIでは、
リクエストボディを使用しない。

取得対象となる目的は、
パスパラメータの
`objectiveId`から特定する。

取得対象となる
目的達成判定履歴は、
パスパラメータの
`assessmentId`から特定する。

---

### 10 リクエスト項目

本APIでは、
リクエストボディに
業務項目を持たない。

利用者IDは、
`X-User-Id`から取得する。

取得対象は、
以下の組み合わせによって特定する。

```text
X-User-Id
+
objectiveId
+
assessmentId
```

クライアントから
以下の情報は受け付けない。

- `userId`
- `result`
- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 資産口座情報
- 保有商品情報
- 手取り収入
- 判定時に使用した計算値

これらをクライアントから受け取り、
目的達成判定履歴を
再計算することはしない。

---

### 11 バリデーション

#### 11.1 objectiveId

`objectiveId`は、
必須のパスパラメータとする。

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

`objectiveId`の形式が不正な場合は、
`VALIDATION_ERROR`
として扱う。

---

#### 11.2 assessmentId

`assessmentId`は、
必須のパスパラメータとする。

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

正常例：

```text
1
20
123
```

不正例：

```text
0
-1
abc
1.5
```

`assessmentId`の形式が不正な場合は、
`VALIDATION_ERROR`
として扱う。

---

#### 11.3 目的の存在確認

指定された`objectiveId`について、
以下の条件を満たす
目的が存在することを確認する。

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
AND
objectives.deleted_at IS NULL
```

対象となる目的が
存在しない場合は、
`OBJECTIVE_NOT_FOUND`
として扱う。

以下の場合を含む。

- 指定された`objectiveId`が存在しない
- 指定された目的が他の利用者に属している
- 指定された目的が論理削除されている

他の利用者に属する
目的IDが指定された場合も、
同じエラーとして扱う。

これにより、
他の利用者に属する目的の存在を
レスポンスから判別できないようにする。

---

#### 11.4 目的達成判定履歴の存在確認

目的の存在および
利用者境界を確認した後、
指定された`assessmentId`について、
以下の条件を満たす
目的達成判定履歴が
存在することを確認する。

```text
assessment_histories.id = assessmentId
AND
assessment_histories.objective_id = objectiveId
```

対象となる
目的達成判定履歴が
存在しない場合は、
`ASSESSMENT_HISTORY_NOT_FOUND`
として扱う。

以下の場合を含む。

- 指定された`assessmentId`が存在しない
- 指定された履歴が別の目的に紐づいている

`assessmentId`だけで
目的達成判定履歴を取得した後に、
`objective_id`を比較する方式ではなく、
検索条件へ
`objective_id`を含める。

---

#### 11.5 別目的に属する履歴

同一利用者が所有する
別の目的に紐づく
`assessmentId`を指定した場合も、
取得対象とはしない。

例えば、

```text
Objective A
id = 5

Objective B
id = 6

Assessment History
id = 20
objective_id = 6
```

の状態で、

```http
GET /api/v1/objectives/5/assessments/20
```

を実行した場合、

```text
assessment_histories.id = 20
AND
assessment_histories.objective_id = 5
```

を満たさないため、
`ASSESSMENT_HISTORY_NOT_FOUND`
として扱う。

別の目的に
履歴が存在することを
レスポンスから判別できないようにする。

---

#### 11.6 他利用者に属する履歴

他の利用者に属する
目的達成判定履歴を
取得することはできない。

利用者境界は、
以下の順序で保証する。

```text
X-User-Id
    ↓
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
    ↓
assessment_histories.id = assessmentId
AND
assessment_histories.objective_id = objectiveId
```

他の利用者に属する
`objectiveId`が指定された場合は、
目的の取得段階で
`OBJECTIVE_NOT_FOUND`
として扱う。

他の利用者の
目的達成判定履歴が
返却されることはない。

---

#### 11.7 判定結果による取得可否

保存されている
目的達成判定結果によって、
取得可否を変更しない。

以下のいずれも
正常な取得対象とする。

```text
achievable
difficult
unassessable
```

`unassessable`であることを理由として、
APIエラーとはしない。

---

#### 11.8 現在の業務状態による再検証

目的達成判定履歴取得時に、
以下の現在状態を
再検証しない。

- 最新の確定済み月末資産状況が存在するか
- 月末資産残高が登録されているか
- 商品別月末評価額が登録されているか
- 資産口座が現在有効か
- 保有商品が現在有効か
- 手取り収入が現在3ヶ月分存在するか
- 現在の資産状況で目的を達成可能か

これらは、
ASM-001 目的達成判定実行時の
業務判定に関する責務である。

ASM-003では、
保存済みの
目的達成判定履歴を取得することに
責務を限定する。

---

#### 11.9 X-User-Id

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

`X-User-Id`が
指定されていない場合は、
`USER_CONTEXT_REQUIRED`
として扱う。

形式が不正な場合は、
`INVALID_USER_ID`
として扱う。

指定された利用者が
存在しない場合、
または論理削除されている場合は、
`USER_NOT_FOUND`
として扱う。

---

#### 11.10 業務状態に依存する検証

以下は、
単項目バリデーションではなく、
業務ルールとして扱う。

- 指定された目的が操作対象利用者に属していること
- 論理削除済みの目的を取得対象としないこと
- 指定された目的達成判定履歴が指定された目的に紐づいていること
- 他の目的に紐づく目的達成判定履歴を取得しないこと
- 他の利用者に属する目的達成判定履歴を取得しないこと
- 判定結果によって取得対象から除外しないこと
- 現在の業務データから過去の判定結果を再計算しないこと

入力形式の不正による
`VALIDATION_ERROR`と、
目的または
目的達成判定履歴が
存在しない状態は
明確に区別して扱う。

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
AND
objectives.deleted_at IS NULL
```

他の利用者に属する目的、
または論理削除済みの目的は、
存在しないものとして扱う。

---

#### 12.2 取得対象となる目的達成判定履歴

取得対象は、
`assessmentId`で指定された
目的達成判定履歴1件とする。

ただし、
`assessmentId`だけでは取得せず、
指定された目的との関連を
検索条件に含める。

```text
assessment_histories.id = assessmentId
AND
assessment_histories.objective_id = objectiveId
```

これにより、
指定された目的に紐づく
目的達成判定履歴のみを
取得対象とする。

---

#### 12.3 利用者境界

目的達成判定履歴の
利用者境界は、
所属する目的を経由して確認する。

```text
X-User-Id
    ↓
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
    ↓
assessment_histories.id = assessmentId
AND
assessment_histories.objective_id = objectiveId
```

`assessment_histories`に
`user_id`を保持しない場合でも、
`objectives`を経由して
利用者境界を保証する。

他の利用者に属する
目的達成判定履歴を
取得してはならない。

---

#### 12.4 別の目的に紐づく履歴を取得しない

同一利用者が
複数の目的を所有している場合でも、
指定された`objectiveId`とは
別の目的に紐づく履歴を
取得してはならない。

例えば、

```text
Objective A
id = 5

Objective B
id = 6

Assessment History
id = 20
objective_id = 6
```

の場合、

```http
GET /api/v1/objectives/5/assessments/20
```

では、
履歴ID`20`を返却しない。

```text
assessment_histories.id = 20
AND
assessment_histories.objective_id = 5
```

を満たさないため、
対象履歴は存在しないものとして扱う。

---

#### 12.5 保存済みの判定結果を取得する

本APIでは、
ASM-001 目的達成判定実行APIによって
保存された
目的達成判定履歴を取得する。

詳細取得時に、
目的達成判定を
再実行しない。

```text
assessment_histories
    ↓
保存済みデータを取得
    ↓
レスポンス
```

現在の業務データを使用して、
過去の判定結果を
再計算してはならない。

---

#### 12.6 過去の判定結果を変更しない

目的達成判定履歴は、
ASM-001実行時点の
判定結果を保持する。

その後、

- 目的
- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品
- 手取り収入

などが変更されても、
保存済みの目的達成判定履歴を
現在の状態に合わせて変更しない。

本APIでは、
保存されている履歴を
そのまま取得する。

---

#### 12.7 判定不可の履歴

`result = unassessable`の
目的達成判定履歴も、
正常な取得対象とする。

```text
achievable
difficult
unassessable
```

のいずれであっても、
指定された目的に紐づく
保存済みの履歴であれば
取得できる。

判定不可を理由として、
APIエラーとはしない。

---

#### 12.8 目的が存在しない場合

指定された目的が
操作対象利用者の取得対象として
存在しない場合は、

```text
OBJECTIVE_NOT_FOUND
```

として扱う。

以下の場合を含む。

- `objectiveId`が存在しない
- 他の利用者に属する目的である
- 論理削除済みの目的である

他の利用者に属する目的の存在を
レスポンスから推測できないようにする。

---

#### 12.9 目的達成判定履歴が存在しない場合

目的は存在するが、
以下を満たす履歴が存在しない場合は、

```text
assessment_histories.id = assessmentId
AND
assessment_histories.objective_id = objectiveId
```

`ASSESSMENT_HISTORY_NOT_FOUND`
として扱う。

以下の場合を含む。

- `assessmentId`自体が存在しない
- `assessmentId`は存在するが別の目的に紐づいている

別の目的に履歴が存在するかどうかを
レスポンスから判別できないようにする。

---

#### 12.10 現在の業務状態を再評価しない

本APIでは、
目的達成判定履歴を取得するために、
以下の現在状態を再評価しない。

- 確定済み月末資産状況が存在するか
- 月末資産残高が存在するか
- 商品別月末評価額が存在するか
- 資産口座が現在有効か
- 保有商品が現在有効か
- 手取り収入が3ヶ月分存在するか
- 現在の総資産額はいくらか
- 現在の平均手取り収入はいくらか
- 現在の状態で目的を達成可能か

これらは、
ASM-001 目的達成判定実行APIの
責務とする。

ASM-003は、
保存済み履歴の取得に
責務を限定する。

---

### 13 取得条件

目的達成判定履歴を取得する前に、
対象目的について
以下の条件を確認する。

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
AND
objectives.deleted_at IS NULL
```

目的が取得できた場合、
目的達成判定履歴を
以下の条件で取得する。

```text
assessment_histories.id = assessmentId
AND
assessment_histories.objective_id = objectiveId
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
目的達成判定履歴の取得
    ↓
レスポンス
```

他の利用者に属する目的の場合は、
目的達成判定履歴の検索へ進まない。

---

### 14 取得対象

取得対象は、
指定された目的に紐づく
`assessment_histories`の
1レコードとする。

```text
assessment_histories.id = assessmentId
AND
assessment_histories.objective_id = objectiveId
```

判定結果による
取得対象の制限は行わない。

そのため、
以下の判定結果を
すべて取得可能とする。

```text
achievable
difficult
unassessable
```

本APIでは、
現在の業務データを取得して
判定結果を再生成しない。

---

### 15 レスポンス

取得成功時は、
指定された
目的達成判定履歴を返却する。

HTTPステータスは、

```http
200 OK
```

とする。

レスポンスは、
API共通方針で定めた
Envelope形式を使用する。

例：

```json
{
  "data": {
    "id": "20",
    "objectiveId": "5",
    "result": "achievable"
  }
}
```

---

#### 15.1 達成可能の場合

保存されている判定結果が
`achievable`の場合は、
その値を返却する。

```json
{
  "data": {
    "id": "20",
    "objectiveId": "5",
    "result": "achievable"
  }
}
```

---

#### 15.2 達成困難の場合

保存されている判定結果が
`difficult`の場合は、
その値を返却する。

```json
{
  "data": {
    "id": "20",
    "objectiveId": "5",
    "result": "difficult"
  }
}
```

---

#### 15.3 判定不可の場合

保存されている判定結果が
`unassessable`の場合も、
正常レスポンスとして返却する。

```json
{
  "data": {
    "id": "20",
    "objectiveId": "5",
    "result": "unassessable"
  }
}
```

判定不可を理由として、
4xxエラーにはしない。

---

### 16 レスポンス項目

目的達成判定履歴詳細の
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
  "id": "20"
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

例：

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
目的達成判定結果である。

例：

```json
{
  "result": "achievable"
}
```

本APIの実行時に
目的達成判定を
再実行して生成する値ではない。

`assessment_histories`に
保存されている判定結果を
APIレスポンスへ変換して返却する。

---

#### 16.4 返却しない情報

本APIでは、
以下の情報は返却しない。

- `user_id`
- 目的の詳細情報
- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 資産口座情報
- 保有商品情報
- 利用可能資産設定
- 手取り収入
- 判定時の計算途中の値
- `created_at`
- `updated_at`

目的達成判定履歴詳細として
必要な情報のみを返却し、
現在の業務データを
付加して返却しない。

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
- `assessmentId`の形式が不正
- 指定された目的が存在しない
- 指定された目的達成判定履歴が存在しない
- 想定外のサーバーエラー

保存されている判定結果が
`unassessable`であることは、
APIエラーとして扱わない。

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

#### 17.5 assessmentIdのバリデーションエラー

`assessmentId`が
バリデーション条件を満たさない場合は、
`VALIDATION_ERROR`
を返却する。

例えば、
以下の場合を含む。

```text
assessmentId = 0
assessmentId = -1
assessmentId = abc
assessmentId = 1.5
```

レスポンス例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "assessmentId",
        "reason": "invalidFormat",
        "message": "目的達成判定履歴IDの形式を確認してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

#### 17.6 目的が存在しない場合

以下の条件を満たす
目的が存在しない場合は、
`OBJECTIVE_NOT_FOUND`
を返却する。

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
AND
objectives.deleted_at IS NULL
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

#### 17.7 目的達成判定履歴が存在しない場合

以下の条件を満たす
目的達成判定履歴が
存在しない場合は、
`ASSESSMENT_HISTORY_NOT_FOUND`
を返却する。

```text
assessment_histories.id = assessmentId
AND
assessment_histories.objective_id = objectiveId
```

レスポンス例：

```json
{
  "error": {
    "code": "ASSESSMENT_HISTORY_NOT_FOUND",
    "message": "指定された目的達成判定履歴が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

以下の場合を含む。

- 指定された`assessmentId`が存在しない
- 指定された`assessmentId`が別の目的に紐づいている

別の目的に
同じ`assessmentId`が存在することを
レスポンスから判別できないようにする。

---

#### 17.8 他利用者に属する履歴の場合

他の利用者に属する
目的達成判定履歴を
取得することはできない。

他の利用者に属する
`objectiveId`を指定した場合は、
目的取得の段階で

`OBJECTIVE_NOT_FOUND`

を返却する。

そのため、
他利用者の
目的達成判定履歴について
専用エラーコードは定義しない。

---

#### 17.9 判定不可の履歴の場合

保存されている
目的達成判定結果が
`unassessable`の場合は、
エラーとしない。

```text
result = unassessable
    ↓
200 OK
```

レスポンス例：

```json
{
  "data": {
    "id": "20",
    "objectiveId": "5",
    "result": "unassessable"
  }
}
```

判定不可は、
ASM-001実行時に保存された
正常な判定結果として扱う。

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
| `200 OK` | 目的達成判定履歴詳細の取得に成功した |
| `400 Bad Request` | 利用者コンテキストが未指定、または利用者IDの形式が不正である |
| `404 Not Found` | 指定された利用者、目的または目的達成判定履歴が存在しない |
| `422 Unprocessable Entity` | `objectiveId`または`assessmentId`のバリデーションエラー |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

---

#### 18.1 200の扱い

指定された
目的達成判定履歴を
正常に取得できた場合は、
`200 OK`
を返却する。

以下の判定結果すべてを
正常取得として扱う。

```text
achievable
difficult
unassessable
```

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
- 指定された目的達成判定履歴が存在しない
- 指定された目的達成判定履歴が別の目的に紐づいている

---

#### 18.4 422の扱い

以下のパスパラメータが
バリデーション条件を満たさない場合は、
`422 Unprocessable Entity`
を返却する。

- `objectiveId`
- `assessmentId`

---

### 19 エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `USER_CONTEXT_REQUIRED` | 400 | `X-User-Id`が指定されていない | × |
| `INVALID_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `USER_NOT_FOUND` | 404 | 指定された利用者が存在しない、または論理削除されている | × |
| `VALIDATION_ERROR` | 422 | `objectiveId`または`assessmentId`が不正である | × |
| `OBJECTIVE_NOT_FOUND` | 404 | 目的が存在しない、論理削除済み、または他利用者に属している | × |
| `ASSESSMENT_HISTORY_NOT_FOUND` | 404 | 目的達成判定履歴が存在しない、または指定目的に紐づいていない | × |
| `INTERNAL_SERVER_ERROR` | 500 | 想定外のサーバーエラーが発生した | ○ |

以下は、
エラーコードの対象としない。

- 判定結果が`achievable`
- 判定結果が`difficult`
- 判定結果が`unassessable`

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
- 過去の目的達成判定結果の再計算
- 現在の資産状況を使用した判定結果の更新
- 月末資産状況の作成
- 月末資産状況の確定
- 月末資産状況の確定解除
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
目的達成判定履歴取得
    ↓
レスポンス
```

詳細取得のためだけに、

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

目的達成判定履歴詳細の参照によって、
ASM-001などの更新処理を
不要にブロックしてはならない。

---

#### 21.2 取得中のデータ変更

ASM-003で
目的達成判定履歴を取得している間に、
別APIによって
新しい目的達成判定履歴が追加されても、
取得対象となる
既存履歴の内容には影響しない。

また、
本APIでは
現在の業務データを
再取得して判定結果を生成しないため、

```text
目的変更
資産変更
手取り収入変更
```

が並行して行われても、
保存済みの
`assessment_histories`を
そのまま返却する。

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
GET /api/v1/objectives/5/assessments/20
    ↓
200 OK

2回目
GET /api/v1/objectives/5/assessments/20
    ↓
200 OK
```

本APIの実行によって、

- 目的達成判定履歴が増加する
- 判定結果が再計算される
- 保存済み履歴が更新される

ことはない。

対象となる履歴自体が
別処理によって削除されるなど、
サーバー上のデータ状態が変化した場合は、
同じGETリクエストでも
異なるレスポンスとなる可能性がある。

これは、
GET APIの冪等性を
損なうものではない。

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

目的達成判定履歴詳細の
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
| `created_at` | 登録日時 |
| `updated_at` | 更新日時 |

取得条件は、
以下とする。

```text
id = assessmentId
AND
objective_id = objectiveId
```

ただし、
事前に`objectives`を参照し、
対象目的が
操作対象利用者に属することを
確認する。

`assessmentId`だけで
取得してはならない。

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
目的達成判定履歴詳細取得のために
参照しない。

目的達成判定履歴は、
ASM-001実行時点の結果として
`assessment_histories`へ保存されているため、
詳細取得時に
最新の月末資産状況を再評価しない。

本APIでは、
`month_end_asset_snapshots`を更新しない。

---

#### 23.5 month_end_asset_balances

本APIでは、
`month_end_asset_balances`を
参照・登録・更新・削除しない。

過去の目的達成判定結果を、
現在の月末資産残高から
再計算しない。

---

#### 23.6 month_end_holding_values

本APIでは、
`month_end_holding_values`を
参照・登録・更新・削除しない。

過去の目的達成判定結果を、
現在の商品別月末評価額から
再計算しない。

---

#### 23.7 asset_accounts

本APIでは、
`asset_accounts`を
参照・登録・更新・削除しない。

残高記録単位や
現在の資産口座状態を
詳細取得時に再評価しない。

---

#### 23.8 asset_account_available_settings

本APIでは、
`asset_account_available_settings`を
参照・登録・更新・削除しない。

過去の目的達成判定履歴について、
現在または過去の
資産口座利用可否を
再判定しない。

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
  - 指定された目的に紐づく特定の判定履歴を取得できる
  - 判定履歴は`assessmentId`だけでなく、`objectiveId`との組み合わせで取得する
  - 別の目的に紐づく判定履歴は取得できない
  - 判定不可の履歴も詳細取得できる
  - 保存済みの判定結果をそのまま返却する

- 利用者境界
  - 操作対象利用者に属する目的の判定履歴のみ取得できる
  - 他の利用者に属する目的達成判定履歴を取得できない
  - 他の利用者に属する目的の存在をレスポンスから推測できない

- API共通
  - IDはAPIレスポンス上stringとして扱う
  - JSONフィールド名はcamelCaseとする
  - エラー時は共通エラーレスポンス形式を使用する

具体的な章番号は、
`functional-requirements.md`の
最新定義に従う。

---

### 25 テスト観点

#### 25.1 正常系

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

#### 25.2 達成可能の履歴

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

#### 25.3 達成困難の履歴

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

#### 25.4 判定不可の履歴

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

#### 25.5 objectiveId形式不正

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

#### 25.6 assessmentId形式不正

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

#### 25.7 objectiveIdとassessmentIdの両方が不正

`objectiveId`および
`assessmentId`の両方に
不正値を指定する。

以下を確認する。

- `422 Unprocessable Entity`となること
- `VALIDATION_ERROR`となること
- `error.details`に対象項目が適切に含まれること

---

#### 25.8 目的不存在

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

#### 25.9 他利用者の目的

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

#### 25.10 論理削除済みの目的

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

#### 25.11 目的達成判定履歴不存在

目的は存在するが、
存在しない`assessmentId`を指定する。

期待結果：

```text
404 Not Found
ASSESSMENT_HISTORY_NOT_FOUND
```

---

#### 25.12 同一利用者の別目的に属する履歴

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

#### 25.13 他利用者の履歴

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

#### 25.18 現在の目的が変更されている

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

#### 25.19 現在の資産状況が変更されている

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

#### 25.20 現在の手取り収入が変更されている

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

#### 25.21 判定不可条件が解消されている

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

#### 25.22 ASM-001再実行後の過去履歴

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

#### 25.23 ASM-002からの詳細取得

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

#### 25.24 副作用

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

#### 25.25 冪等性

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

#### 25.26 レスポンス契約

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

#### 25.27 返却しない情報

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

#### 25.28 エラーレスポンス

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

#### 25.29 不要なテーブルを参照しないこと

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

### 26 Laravel実装方針

ASM-003では、
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
API Resource
    ↓
Responder
```
本APIは、保存済みの目的達成判定履歴1件を参照するだけのAPIである。

目的達成判定の再実行や、現在の資産情報・手取り収入を使用した判定結果の再計算は行わない。

---

#### 26.1 Action

HTTPリクエストを受け付け、目的ID、目的達成判定履歴IDおよび利用者コンテキストを取得する。

目的達成判定履歴詳細取得UseCaseを呼び出し、取得結果をResponderへ渡す。

概念例：

```php
final class ShowAssessmentHistoryAction
{
    public function __invoke(
        ShowAssessmentHistoryRequest $request,
        ShowAssessmentHistoryUseCase $useCase,
        AssessmentHistoryDetailResponder $responder,
        string $objectiveId,
        string $assessmentId,
    ): JsonResponse {
        $assessmentHistory = $useCase->execute(
            userId: $request->userId(),
            objectiveId: (int) $objectiveId,
            assessmentId: (int) $assessmentId,
        );

        return $responder->ok(
            $assessmentHistory,
        );
    }
}
```

Actionでは、以下の処理を行わない。

- `objectiveId`の形式検証
- `assessmentId`の形式検証
- 目的の検索
- 利用者境界の判定
- 目的達成判定履歴の検索
- 目的と判定履歴の関連確認
- 判定結果の再計算
- 現在の資産情報の取得
- 現在の手取り収入の取得
- レスポンス形式への変換

---

#### 26.2 Request

パスパラメータの以下を検証する。

- `objectiveId`
- `assessmentId`

本APIでは、クエリパラメータおよびリクエストボディを使用しない。

パスパラメータは、`prepareForValidation()`でバリデーション対象へ追加する。

概念例：

```php
final class ShowAssessmentHistoryRequest
    extends FormRequest
{
    protected function prepareForValidation(): void
    {
        $this->merge([
            'objectiveId'
                => $this->route(
                    'objectiveId',
                ),

            'assessmentId'
                => $this->route(
                    'assessmentId',
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

            'assessmentId' => [
                'required',
                'integer',
                'min:1',
            ],
        ];
    }
}
```

Requestでは、以下を行わない。

- 目的の存在確認
- 利用者境界の確認
- 目的達成判定履歴の存在確認
- 目的と判定履歴の関連確認
- 判定結果の取得
- 判定結果の再計算

これらは、データベース状態に依存するため、UseCaseおよびQueryで扱う。

---

#### 26.3 UseCase

目的達成判定履歴詳細取得のユースケース処理を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `objectiveId`を受け取る
3. `assessmentId`を受け取る
4. 操作対象利用者に属する目的を取得する
5. 目的が存在しない場合は業務例外を送出する
6. 指定された目的に紐づく目的達成判定履歴を取得する
7. 履歴が存在しない場合は業務例外を送出する
8. 保存済みの目的達成判定履歴を返却する

概念的な処理は、以下とする。

```text
操作対象利用者
    ↓
objectiveId
    ↓
目的取得
    ↓
目的なし
    → OBJECTIVE_NOT_FOUND

目的あり
    ↓
objectiveId
+
assessmentId
    ↓
目的達成判定履歴取得
    ↓
履歴なし
    → ASSESSMENT_HISTORY_NOT_FOUND

履歴あり
    ↓
保存済み履歴返却
```

UseCaseでは、目的達成判定そのものを実行しない。

---

#### 26.4 目的を先に確認する

目的達成判定履歴を取得する前に、操作対象利用者に属する目的が存在することを確認する。

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

これにより、

```text
操作対象利用者
    ↓
目的
    ↓
目的達成判定履歴
```

の順序で利用者境界を保証する。

`assessmentId`だけを使用して最初に履歴を検索しない。

---

#### 26.5 ObjectiveQuery

操作対象利用者に属する目的を1件取得する。

取得条件には、必ず目的IDおよび利用者IDを含める。

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

以下の目的は、取得対象としない。

- 他の利用者に属する目的
- 論理削除済みの目的

他利用者の目的が存在する場合も、取得結果は`null`とする。

これにより、他利用者の目的の存在をレスポンスから推測できないようにする。

---

#### 26.6 無効化済み目的

目的が

```text
enabled = false
```

であっても、論理削除されていなければ履歴参照の対象とする。

無効化は、新しい目的達成判定を実行できない状態を意味する。

過去に保存された目的達成判定履歴まで参照不可にはしない。

```text
目的
enabled = false
    ↓
過去履歴あり
    ↓
ASM-003で取得可能
```

そのため、ObjectiveQueryでは`enabled = true`による絞り込みを行わない。

---

#### 26.7 AssessmentHistoryQuery

目的達成判定履歴1件の取得は、専用Queryで行う。

検索条件には、

```text
assessment_histories.id
+
assessment_histories.objective_id
```

を含める。

概念例：

```php
final class AssessmentHistoryQuery
{
    public function findByObjectiveAndId(
        int $objectiveId,
        int $assessmentId,
    ): ?AssessmentHistory {
        return AssessmentHistory::query()
            ->whereKey(
                $assessmentId,
            )
            ->where(
                'objective_id',
                $objectiveId,
            )
            ->first();
    }
}
```

`assessmentId`だけで目的達成判定履歴を取得しない。

以下のような実装は行わない。

```php
AssessmentHistory::find(
    $assessmentId,
);
```

---

#### 26.8 objectiveIdとassessmentIdの組み合わせ

指定された`assessmentId`が存在していても、別の目的に紐づいている場合は取得しない。

例えば、

```text
objectiveId = 5

assessmentId = 10
objective_id = 6
```

の場合は、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

として扱う。

これにより、別の目的に紐づく履歴を誤って取得することを防止する。

---

#### 26.9 利用者境界

利用者境界は、以下の関連によって保証する。

```text
users
    ↓
objectives.user_id
    ↓
objectives.id
    ↓
assessment_histories.objective_id
```

まず、

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
```

で目的を確認する。

その後、

```text
assessment_histories.id = assessmentId
AND
assessment_histories.objective_id = objectiveId
```

で履歴を取得する。

これにより、`assessment_histories`に`user_id`を直接保持していない場合でも、目的を経由して利用者境界を保証できる。

---

#### 26.10 目的達成判定履歴不存在

AssessmentHistoryQueryの結果が`null`の場合は、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

へ変換するための業務例外を送出する。

概念例：

```php
$assessmentHistory =
    $this->assessmentHistoryQuery
        ->findByObjectiveAndId(
            objectiveId: $objective->id,
            assessmentId: $assessmentId,
        );

if ($assessmentHistory === null) {
    throw new AssessmentHistoryNotFoundException();
}
```

以下を同じ404として扱う。

- `assessmentId`が存在しない
- 別の目的に紐づいている

内部的な不存在理由をクライアントへ区別して公開しない。

---

#### 26.11 Repository

本APIでは、Repositoryを使用しない。

ASM-003は参照専用APIであり、以下を行わないためである。

- 登録
- 更新
- 削除

目的および目的達成判定履歴の取得は、参照専用Queryが担当する。

---

#### 26.12 保存済みresultを使用する

レスポンスの`result`は、`assessment_histories`へ保存されている値を使用する。

以下のような処理は行わない。

```text
目的達成判定履歴取得
    ↓
現在の資産情報取得
    ↓
現在の手取り収入取得
    ↓
AssessmentCalculator
    ↓
result再計算
```

ASM-003では、過去の判定実行時点で保存された結果をそのまま返却する。

---

#### 26.13 判定不可の扱い

`result = unassessable`の目的達成判定履歴も、通常の履歴として取得する。

```text
result = unassessable
    ↓
200 OK
```

とする。

以下のような処理は行わない。

```php
if (
    $assessmentHistory->result
    === AssessmentResultType::Unassessable
) {
    throw new AssessmentUnavailableException();
}
```

判定不可は、ASM-001実行時に保存された正常な履歴データとして扱う。

---

#### 26.14 現在の業務データを取得しない

ASM-003では、履歴詳細を取得するために以下の現在データを検索しない。

- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `holding_assets`
- `asset_account_available_settings`
- `net_incomes`

目的達成判定履歴に保存されていない値を、現在の業務データから復元しない。

---

#### 26.15 判定時点の詳細情報

将来的に、以下の情報などを詳細レスポンスへ追加する場合は、ASM-001実行時点で`assessment_histories`へスナップショットとして保存された値を使用する。

- 判定対象年月
- 判定時点の目的情報
- 判定時点の利用可能資産
- 判定時点の平均手取り収入
- 判定時点の必要支出額
- 判定時点の残余資金
- 判定不可理由

ASM-003実行時に現在データから再生成しない。

---

#### 26.16 AssessmentHistory Model

`AssessmentHistory` Modelは、`assessment_histories`へ対応する。

目的との関連は、必要に応じてRelationとして定義してよい。

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

ただし、ASM-003の検索ロジックそのものをModelへ記述しない。

---

#### 26.17 resultのCast

`assessment_histories.result`をEnumで管理する場合は、Eloquent Castを使用する。

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

これにより、Laravel内部では

```text
achievable
difficult
unassessable
```

をマジック文字列として散在させずに扱える。

---

#### 26.18 API Resource

データベースカラムを直接返却せず、API Resourceを使用してAPIレスポンス形式へ変換する。

概念例：

```php
final class AssessmentHistoryDetailResource
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

データベースのsnake_caseカラム名をそのまま返却しない。

---

#### 26.19 一覧Resourceとの共通化

ASM-002とASM-003でレスポンス項目が同一である場合は、共通の

```text
AssessmentHistoryResource
```

を使用してよい。

将来的にASM-003へ判定時点の詳細情報を追加する場合は、

```text
AssessmentHistoryResource
AssessmentHistoryDetailResource
```

を分離する。

一覧レスポンスを詳細APIの拡張によって不要に肥大化させない。

---

#### 26.20 Responder

Responderは、取得済みの目的達成判定履歴を受け取り、API共通方針に従ったHTTPレスポンスへ変換する。

概念例：

```php
final class AssessmentHistoryDetailResponder
{
    public function ok(
        AssessmentHistory $assessmentHistory,
    ): JsonResponse {
        return response()->json(
            [
                'data' =>
                    new AssessmentHistoryDetailResource(
                        $assessmentHistory,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

正常時は、

```text
200 OK
```

を返却する。

`requestId`などの共通Envelope項目は、API共通方針に従って付与する。

---

#### 26.21 Responderの責務

Responderでは、以下の処理を行わない。

- データベース検索
- 目的の存在確認
- 利用者境界の判定
- 目的達成判定履歴の存在確認
- 目的と履歴の関連確認
- 判定結果の再計算
- 現在の業務データの取得

Responderは、取得済みリソースをHTTPレスポンス形式へ変換することに責務を限定する。

---

#### 26.22 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONレスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

#### 26.23 トランザクション

ASM-003は、読み取り専用APIである。

そのため、明示的な`DB::transaction()`を使用しない。

以下のような実装は行わない。

```php
DB::transaction(
    function () {
        // 履歴詳細取得のみ
    },
);
```

参照処理だけのために不要なトランザクション境界を追加しない。

Phase1では、詳細取得処理中の厳密なスナップショット分離は保証しない。

---

#### 26.24 ロック

ASM-003では、行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

ASM-003による参照で、ASM-001の履歴登録などを不要にブロックしない。

---

#### 26.25 N+1問題

ASM-003は、単一レコード取得APIであるため、典型的なN+1問題は発生しにくい。

また、Phase1のレスポンス項目は`assessment_histories`単体で取得できる情報を中心とする。

`objectiveId`は、

```text
assessment_histories.objective_id
```

から取得する。

レスポンス生成のためだけに`objective` Relationを遅延ロードしない。

---

#### 26.26 取得カラム

Phase1でレスポンスに必要な項目だけを取得してよい。

概念例：

```php
return AssessmentHistory::query()
    ->select([
        'id',
        'objective_id',
        'result',
    ])
    ->whereKey(
        $assessmentId,
    )
    ->where(
        'objective_id',
        $objectiveId,
    )
    ->first();
```

将来的に判定時点のスナップショット情報をレスポンスへ追加する場合は、必要なカラムを追加する。

現在の別テーブルをJOINして過去情報を再構成しない。

---

#### 26.27 キャッシュ

Phase1では、ASM-003専用のサーバー側アプリケーションキャッシュを使用しない。

目的達成判定履歴は履歴データであり変更頻度は低いが、Phase1では実装を単純に保つ。

React Query等によるクライアント側キャッシュは使用してよい。

---

#### 26.28 例外変換

Laravel内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `objectiveId`形式不正 | `VALIDATION_ERROR` |
| `assessmentId`形式不正 | `VALIDATION_ERROR` |
| 目的不存在 | `OBJECTIVE_NOT_FOUND` |
| 利用者境界外の目的 | `OBJECTIVE_NOT_FOUND` |
| 論理削除済み目的 | `OBJECTIVE_NOT_FOUND` |
| 目的達成判定履歴不存在 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| 別目的に紐づく目的達成判定履歴 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

`result = unassessable`の場合は、例外へ変換しない。

正常な履歴として返却する。

---

#### 26.29 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- SQL
- PostgreSQLの制約名
- スタックトレース
- PHP内部エラー
- Laravel内部例外メッセージ
- サーバー内部ファイルパス

詳細情報は、サーバーログへ記録する。

---

#### 26.30 ログ

ASM-003では、必要に応じて以下の情報をログコンテキストへ設定する。

```text
requestId
userId
objectiveId
assessmentId
```

取得した履歴の判定材料や資産情報などを不要にログへ出力しない。

---

#### 26.31 テスト実装方針

Laravel側では、Feature Testを中心としてASM-003のAPI契約および詳細取得処理を確認する。

Feature Testでは、主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `404 Not Found`
- `422 Unprocessable Entity`
- `500 Internal Server Error`
- `objectiveId`の形式検証
- `assessmentId`の形式検証
- 利用者境界
- 指定目的に紐づく履歴を取得できること
- 他の目的に紐づく履歴を取得できないこと
- 他利用者の目的に紐づく履歴を取得できないこと
- 論理削除済み目的の履歴を取得できないこと
- 無効化済み目的の過去履歴を取得できること
- `achievable`を正常取得できること
- `difficult`を正常取得できること
- `unassessable`を正常取得できること
- 保存済みの`result`がそのまま返却されること
- 現在の目的情報で再計算されないこと
- 現在の資産情報で再計算されないこと
- 現在の手取り収入で再計算されないこと
- ASM-001再実行後も指定した過去履歴が返却されること
- API ResourceによってcamelCaseへ変換されること
- IDがstringとして返却されること
- データベースが更新されないこと

ObjectiveQueryについては、以下を確認する。

```text
同一利用者の目的
    → 取得できる

他利用者の目的
    → null

論理削除済み目的
    → null

enabled = false
    → 取得できる
```

AssessmentHistoryQueryについては、以下を確認する。

```text
objectiveId一致
+
assessmentId一致
    → 取得できる

assessmentId不存在
    → null

assessmentId存在
+
objectiveId不一致
    → null
```

UseCaseについては、以下を確認する。

```text
目的不存在
    ↓
ObjectiveNotFoundException
```

```text
目的あり
+
履歴不存在
    ↓
AssessmentHistoryNotFoundException
```

```text
目的あり
+
履歴あり
    ↓
保存済みAssessmentHistoryを返却
```

また、UseCase実行時に目的達成判定ロジックが呼び出されないことを確認する。

ASM-003は、保存済み目的達成判定履歴を参照することだけに責務を限定する。

---

### 27 React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type GetAssessmentHistoryParams = {
  objectiveId: string;
  assessmentId: string;
};
```

目的達成判定履歴詳細の型は、
以下とする。

```ts
export type AssessmentResult =
  | 'achievable'
  | 'difficult'
  | 'unassessable';

export type AssessmentHistoryDetail = {
  id: string;
  objectiveId: string;
  result: AssessmentResult;
};
```

レスポンス型は、
以下とする。

```ts
export type GetAssessmentHistoryResponse = {
  data: AssessmentHistoryDetail;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetAssessmentHistoryResponse>(
    `/api/v1/objectives/${objectiveId}/assessments/${assessmentId}`,
  );
```

目的達成判定履歴一覧画面などから、
特定の目的達成判定履歴を選択し、
詳細を表示する際に利用する。

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

#### 27.2 assessmentIdの扱い

`assessmentId`は、
API共通方針に従って
stringとして扱う。

```ts
const assessmentId: string = '20';
```

URL生成時は、
`objectiveId`と
`assessmentId`の両方を使用する。

```ts
const url =
  `/api/v1/objectives/${objectiveId}/assessments/${assessmentId}`;
```

フロントエンド側で、
`assessmentId`だけを使用して
履歴詳細を取得しない。

---

#### 27.3 Queryとして扱う

ASM-003は、
読み取り専用のGET APIであるため、
React Query等では
Queryとして扱う。

概念例：

```ts
export const fetchAssessmentHistory =
  async ({
    objectiveId,
    assessmentId,
  }: GetAssessmentHistoryParams) => {
    const response =
      await apiClient.get<GetAssessmentHistoryResponse>(
        `/api/v1/objectives/${objectiveId}/assessments/${assessmentId}`,
      );

    return response.data;
  };
```

---

#### 27.4 Query Key

Query Keyには、
`objectiveId`および
`assessmentId`を含める。

概念例：

```ts
export const assessmentHistoryKeys = {
  all: [
    'assessmentHistories',
  ] as const,

  detail: (
    objectiveId: string,
    assessmentId: string,
  ) =>
    [
      ...assessmentHistoryKeys.all,
      objectiveId,
      assessmentId,
    ] as const,
};
```

これにより、
目的および履歴ごとに
キャッシュを分離する。

---

#### 27.5 Hook実装例

React Queryを使用する場合の
概念例は、
以下とする。

```ts
export const useAssessmentHistory =
  (
    objectiveId: string,
    assessmentId: string,
  ) => {
    return useQuery({
      queryKey:
        assessmentHistoryKeys.detail(
          objectiveId,
          assessmentId,
        ),

      queryFn:
        () =>
          fetchAssessmentHistory({
            objectiveId,
            assessmentId,
          }),

      enabled:
        objectiveId.length > 0
        && assessmentId.length > 0,
    });
  };
```

実際のAPI Clientや
React Queryの利用方針は、
フロントエンド共通設計に従う。

---

#### 27.6 ASM-002からの遷移

ASM-002 目的達成判定履歴一覧取得APIで
取得した履歴から、
詳細画面へ遷移する。

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
一覧画面で保持している
`result`だけに依存せず、
ASM-003によって
対象履歴を再取得する。

---

#### 27.7 詳細画面で再取得する理由

一覧取得後に
画面遷移する場合でも、
詳細画面では
ASM-003を実行する。

これにより、
詳細画面のデータ取得責務を
ASM-003へ統一できる。

また、
ブラウザから
詳細URLへ直接アクセスした場合でも
同じ処理で表示できる。

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
  {resultLabelMap[data.result]}
</span>
```

バックエンドから返却された
保存済みの`result`を使用する。

---

#### 27.9 達成可能の表示

`result = 'achievable'`
の場合は、
過去の判定結果として
達成可能であったことを表示する。

表示例：

```text
判定結果：達成可能
```

現在の資産状況でも
達成可能であることを
意味するものとして
再解釈しない。

---

#### 27.10 達成困難の表示

`result = 'difficult'`
の場合は、
過去の判定結果として
達成困難であったことを表示する。

表示例：

```text
判定結果：達成困難
```

現在の状態を使用して
結果を書き換えない。

---

#### 27.11 判定不可の表示

`result = 'unassessable'`
の場合も、
正常な履歴詳細として表示する。

表示例：

```text
判定結果：判定不可
```

APIエラーとして扱わない。

```ts
if (
  data.result
  === 'unassessable'
) {
  // 判定不可として通常表示
}
```

---

#### 27.12 過去履歴として表示する

ASM-003で取得したデータは、
現在の判定結果ではなく、
過去の判定実行結果として扱う。

画面上でも、
必要に応じて

```text
過去の目的達成判定結果
```

であることが
分かる表示とする。

現在の目的達成可否を
確認したい場合は、
ASM-001を再実行する。

---

#### 27.13 現在データから再計算しない

フロントエンドでは、
ASM-003取得後に、

- 現在の目的
- 現在の資産額
- 現在の手取り収入

を使用して
`result`を再計算しない。

以下のような処理は行わない。

```ts
const currentResult =
  calculateAssessment(
    currentObjective,
    currentAssets,
    currentNetIncome,
  );
```

ASM-003から返却された
`result`をそのまま表示する。

---

#### 27.14 現在の目的情報と混同しない

詳細画面で
現在の目的情報を
別APIから取得して表示する場合でも、
ASM-003の`result`は
過去の履歴として扱う。

例えば、

```text
現在の目的金額
5,000,000円

過去の判定結果
達成困難
```

のように、
現在情報と
過去履歴を
同じ時点の情報として
誤認させないようにする。

---

#### 27.15 OBJECTIVE_NOT_FOUND

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

#### 27.16 ASSESSMENT_HISTORY_NOT_FOUND

`ASSESSMENT_HISTORY_NOT_FOUND`
が返却された場合は、
指定された目的に紐づく
対象履歴を
取得できない状態として扱う。

例えば、

- 履歴が存在しない
- URLが古い
- 別の目的に紐づく履歴IDが指定された

などが考えられる。

フロントエンドでは、
履歴が別の目的に存在するかどうかを
推測しない。

必要に応じて、
ASM-002の
目的達成判定履歴一覧画面へ戻す。

---

#### 27.17 VALIDATION_ERROR

`objectiveId`または
`assessmentId`の形式が不正な場合は、
`VALIDATION_ERROR`
が返却される。

通常の画面操作では
発生しないことを前提とし、
不正なURLまたは
画面状態として扱う。

必要に応じて、
目的一覧または
判定履歴一覧へ戻す。

---

#### 27.18 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 不正なURLまたは画面状態として扱う |
| `OBJECTIVE_NOT_FOUND` | 目的一覧画面へ戻す |
| `ASSESSMENT_HISTORY_NOT_FOUND` | 目的達成判定履歴一覧画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

`unassessable`は、
エラーコードとして扱わない。

---

#### 27.19 ローディング表示

詳細取得中は、
ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return <Loading />;
}
```

取得完了前に、
一覧画面の古い情報を
詳細情報として確定表示しない。

---

#### 27.20 自動リトライ

ASM-003は
読み取り専用GET APIであり、
冪等である。

そのため、
一時的な通信エラーについては、
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
クライアント起因エラーは、
不要にリトライしない。

---

#### 27.21 キャッシュ

ASM-003は、
React Query等による
クライアントキャッシュの対象としてよい。

目的達成判定履歴は
原則として過去の履歴であり、
ASM-003自体によって
内容が変更されることはない。

ただし、
キャッシュ保持時間などは
フロントエンド共通設計に従う。

---

#### 27.22 ASM-001との関係

ASM-001を再実行して
新しい目的達成判定履歴が作成されても、
過去の`assessmentId`に対する
ASM-003の結果は
変更されないことを前提とする。

例えば、

```text
assessmentId = 10
result = difficult

    ↓ ASM-001再実行

assessmentId = 20
result = achievable
```

となった場合でも、

```http
GET /objectives/5/assessments/10
```

では、

```text
result = difficult
```

を返却する。

最新結果を表示したい場合は、
新しい履歴を取得する。

---

#### 27.23 Query実装例

概念例：

```ts
type GetAssessmentHistoryArgs = {
  objectiveId: string;
  assessmentId: string;
};

export const getAssessmentHistory =
  async ({
    objectiveId,
    assessmentId,
  }: GetAssessmentHistoryArgs) => {
    const response =
      await apiClient.get<GetAssessmentHistoryResponse>(
        `/api/v1/objectives/${objectiveId}/assessments/${assessmentId}`,
      );

    return response.data;
  };
```

Hook例：

```ts
export const useAssessmentHistoryDetail =
  (
    objectiveId: string,
    assessmentId: string,
  ) => {
    return useQuery({
      queryKey:
        assessmentHistoryKeys.detail(
          objectiveId,
          assessmentId,
        ),

      queryFn:
        () =>
          getAssessmentHistory({
            objectiveId,
            assessmentId,
          }),

      enabled:
        objectiveId.length > 0
        && assessmentId.length > 0,
    });
  };
```

実際のAPI Client、
React Queryおよび
エラー処理の共通化方式は、
フロントエンド共通設計に従う。

---

### 28 設計上の補足

#### 28.1 GETを採用する理由

ASM-003は、
保存済みの
目的達成判定履歴1件を
取得するだけのAPIである。

データの登録、
更新または削除を行わないため、
HTTPメソッドには
`GET`を採用する。

---

#### 28.2 objectiveIdとassessmentIdをパスに含める理由

目的達成判定履歴は、
特定の目的に紐づく
サブリソースとして扱う。

そのため、

```http
/objectives/{objectiveId}/assessments/{assessmentId}
```

というURLを採用する。

これにより、

```text
どの目的の
どの判定履歴か
```

をURLから明確に表現できる。

---

#### 28.3 assessmentIdだけで取得しない理由

`assessmentId`は
データベース上では
一意であっても、
API上では
目的配下のリソースとして扱う。

そのため、

```text
objectiveId
+
assessmentId
```

の組み合わせによって
取得する。

これにより、
別の目的に紐づく履歴を
誤って取得することを防止する。

---

#### 28.4 目的の利用者境界を先に確認する理由

`assessment_histories`自体に
`user_id`を保持しない場合、
履歴単体では
利用者境界を判断できない。

そのため、
まず

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
```

を確認し、
その後に
目的達成判定履歴を取得する。

これにより、
他利用者の履歴へのアクセスを防止する。

---

#### 28.5 判定不可を404にしない理由

`unassessable`は、
ASM-001によって
正常に保存された
目的達成判定結果である。

そのため、
履歴が存在している限り、
ASM-003では正常取得する。

```text
履歴あり
result = unassessable
    ↓
200 OK
```

とする。

---

#### 28.6 現在データを再評価しない理由

目的達成判定履歴は、
ASM-001実行時点の
結果を保持する。

ASM-003取得時に
現在の資産や
手取り収入から再判定すると、
過去履歴としての意味が失われる。

そのため、
保存済みの結果を
そのまま返却する。

---

#### 28.7 詳細APIでもレスポンスを必要最小限にする理由

Phase1では、
`assessment_histories`に保存されている
主要な判定結果を
詳細APIで返却する。

現在の資産情報や
手取り収入などを
同時に返却しない。

これらを返却すると、

```text
過去の判定結果
+
現在の業務データ
```

が混在し、
どの時点の情報か
分かりにくくなるためである。

---

#### 28.8 判定時点の詳細情報が必要になった場合

将来的に、

- 判定時点の総資産額
- 判定時点の平均手取り収入
- 判定不可理由
- 判定に使用した月末資産状況
- 判定時点の必要資金

などを
詳細画面へ表示する場合は、
ASM-001実行時点で
それらを
`assessment_histories`側へ
スナップショットとして保存する。

ASM-003取得時に
現在の業務データから
復元しない。

---

#### 28.9 ASM-002とASM-003を分ける理由

ASM-002は、
一覧表示に必要な
軽量な情報を複数件取得する。

ASM-003は、
特定の履歴1件を
詳細画面で取得する。

責務を分離することで、
将来的にASM-003へ
判定時点の詳細情報を追加しても、
一覧レスポンスを
不要に肥大化させずに済む。

---

#### 28.10 フロントエンドで再判定しない理由

目的達成判定ロジックを
React側でも実装すると、
バックエンドとの
判定結果差異が生じる可能性がある。

ASM-003では、
保存済み結果を
表示することに責務を限定する。

判定ロジックは、
バックエンドへ集約する。

---

#### 28.11 GETの冪等性

ASM-003は、
同じリクエストを
複数回実行しても
業務データを変更しない。

そのため、
本APIは冪等である。

別処理によって
対象リソース自体の状態が
変化した場合は、
レスポンスが変化する可能性があるが、
ASM-003自体が
データを変更するものではない。

---

#### 28.12 Idempotency-Keyを使用しない理由

ASM-003は
読み取り専用のGET APIである。

再実行によって
履歴の重複登録などが
発生しないため、
`Idempotency-Key`は使用しない。

---

#### 28.13 サーバーキャッシュを採用しない理由

目的達成判定履歴は
変更頻度が低いデータではあるが、
Phase1では
実装を単純に保つ。

そのため、
ASM-003専用の
サーバー側アプリケーションキャッシュは
採用しない。

必要性が生じた場合は、
アクセス量や
性能要件を確認したうえで
将来的に検討する。

---

### 29 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [ASM-001 目的達成判定プレビュー](./asm-001-preview.md)
- [ASM-002 目的達成判定結果保存](./asm-002-create.md)
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
