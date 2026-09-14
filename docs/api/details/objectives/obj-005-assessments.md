# OBJ-005 目的無効化

## 1. 概要

操作対象となる利用者に登録された
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

概念的には、
以下の状態変更を行う。

```text
enabled = true
    ↓
OBJ-005
    ↓
enabled = false
```

本APIでは、
`deleted_at`を更新しない。

目的の無効化は
論理削除とは別の業務操作として扱い、
無効化後も
目的一覧および目的詳細から
参照できる状態を維持する。

---

### 1.1 更新対象

OBJ-005では、
以下を更新する。

```text
objectives.enabled
    true
        ↓
    false
```

更新に伴い、

```text
objectives.updated_at
```

もサーバー側で更新する。

---

### 1.2 更新対象外

OBJ-005では、
以下の目的情報を変更しない。

* `name`
* `planned_year_month`
* `required_expense`
* `memo`
* `user_id`
* `created_at`
* `deleted_at`

目的内容の変更は、
OBJ-004 目的更新APIで行う。

---

### 1.3 論理削除との違い

OBJ-005による無効化は、

```text
enabled = false
```

によって表現する。

以下のような
論理削除は行わない。

```text
deleted_at
    = 現在日時
```

無効化済みの目的は、
通常利用中の目的とは区別するが、
過去情報として
引き続き参照可能とする。

---

### 1.4 目的達成判定への影響

無効化された目的は、
新しく実行する
目的達成判定の対象外とする。

一方、
無効化以前に保存された

```text
assessment_histories
```

は変更しない。

目的無効化を理由として、
過去の目的達成判定履歴を

* 更新
* 削除
* 再計算

しない。

---

## 2. ユースケース

利用者は、
今後利用しない目的を
管理対象から除外するために
OBJ-005を使用する。

例えば、
以下のような場合に使用する。

* 目的を取りやめた
* 目的をすでに達成した
* 今後の目的達成判定対象から外したい
* 現在利用する目的と過去の目的を区別したい

無効化後も、
目的そのものは削除しない。

そのため、
目的一覧および目的詳細では、
無効化済みの目的として
確認できる。

概念的な利用フローは、
以下とする。

```text
OBJ-003
目的詳細取得
    ↓
利用中の目的を表示
    ↓
利用者が無効化を選択
    ↓
OBJ-005
目的無効化
    ↓
enabled = false
    ↓
無効化済みとして表示
```

---

### 2.1 目的を取りやめた場合

利用者が
登録済みの目的を
今後実施しないと判断した場合は、
目的を物理削除せず
無効化する。

これにより、
過去に登録した目的情報を
保持したまま、
今後の利用対象から外す。

---

### 2.2 目的を達成した場合

目的を達成した場合も、
目的そのものを削除せず、
必要に応じて無効化する。

これにより、

```text
過去に設定していた目的
+
過去の目的達成判定履歴
```

を保持できる。

---

### 2.3 過去の判定履歴を保持する

目的を無効化しても、
過去に保存された
目的達成判定履歴は保持する。

概念的には、

```text
Objective
enabled = true
    ↓
目的達成判定
    ↓
assessment_histories登録
    ↓
OBJ-005
    ↓
Objective
enabled = false

assessment_histories
    → 変更なし
```

とする。

---

### 2.4 Phase1では再有効化しない

Phase1では、
無効化済みの目的を
再び

```text
enabled = true
```

へ戻すAPIは提供しない。

再有効化が必要になった場合は、
将来要件として
別途設計する。

---

## 3. エンドポイント

```http
PATCH /api/v1/objectives/{objectiveId}/disabled
```

`objectiveId`には、
無効化対象となる
目的IDを指定する。

例：

```http
PATCH /api/v1/objectives/1/disabled
```

---

### 3.1 OBJ-004とのURL分離

OBJ-004 目的更新は、
以下のURLを使用する。

```http
PATCH /api/v1/objectives/{objectiveId}
```

OBJ-005では、
目的の通常属性更新とは異なる
「無効化」という業務操作を
明示するため、

```http
PATCH /api/v1/objectives/{objectiveId}/disabled
```

とする。

概念的には、

```text
OBJ-004
PATCH /api/v1/objectives/{objectiveId}
    ↓
目的情報更新

OBJ-005
PATCH /api/v1/objectives/{objectiveId}/disabled
    ↓
目的無効化
```

と責務を分離する。

---

### 3.2 通常更新APIと分離する理由

目的無効化を
OBJ-004のRequest Bodyで、

```json
{
  "enabled": false
}
```

のように扱わない。

目的の通常属性変更と
利用状態の変更は、
業務上異なる操作であるためである。

これにより、

```text
name
plannedYearMonth
requiredExpense
memo
    ↓
OBJ-004

enabled
    ↓
OBJ-005
```

という責務分離を明確にする。

---

### 3.3 `/disabled`を使用する理由

OBJ-004とOBJ-005は、
どちらも`PATCH`を使用する。

そのため、
同じ

```http
PATCH /api/v1/objectives/{objectiveId}
```

へ割り当てず、
OBJ-005では

```text
/disabled
```

を付与することで、
目的無効化という
業務操作をURL上でも区別する。

---

## 4. HTTPメソッド

```http
PATCH
```

本APIでは、
既存の目的について

```text
enabled
```

だけを変更するため、
`PATCH`を使用する。

---

### 4.1 PATCHを使用する理由

目的リソース全体を
置き換えるのではなく、
利用状態だけを変更する。

概念的には、

```text
Objective

name
plannedYearMonth
requiredExpense
memo
enabled
    ↓
enabledのみ変更
```

となる。

そのため、
`PUT`ではなく
`PATCH`を使用する。

---

### 4.2 本APIで行わないこと

OBJ-005では、
以下を行わない。

* 目的登録
* 目的名更新
* 実施予定年月更新
* 必要支出額更新
* メモ更新
* 目的の論理削除
* 目的の物理削除
* 目的の再有効化
* 目的達成判定
* 過去の目的達成判定履歴更新

目的無効化だけに
責務を限定する。

---

### 4.3 リクエストボディでenabledを指定させない

OBJ-005は、

```text
目的を無効化する
```

こと自体を表す
専用エンドポイントである。

そのため、
クライアントから

```json
{
  "enabled": false
}
```

を送信させる必要はない。

`PATCH /disabled`が
呼び出されたこと自体を、

```text
enabled = false
```

への状態遷移要求として扱う。

また、

```json
{
  "enabled": true
}
```

のような指定によって
再有効化することもできない。

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

### 5.1 利用者コンテキスト

`X-User-Id`は、
API共通方針に従って検証する。

概念的には、

```text
X-User-Id
    ↓
必須確認
    ↓
形式確認
    ↓
利用者存在確認
    ↓
利用者コンテキスト設定
    ↓
OBJ-005
```

とする。

Action以降では、
検証済みの
利用者コンテキストを使用する。

---

### 5.2 利用者境界

無効化対象となる目的は、
少なくとも以下の条件で取得する。

```text
objectives.id
    = objectiveId

AND

objectives.user_id
    = 操作対象利用者ID

AND

objectives.deleted_at
    IS NULL
```

`objectiveId`だけで
対象目的を取得しない。

---

### 5.3 他利用者の目的

例えば、

```text
User A
    Objective ID = 10

User B
    X-User-Id = 2
```

という状態で、
User Bが

```http
PATCH /api/v1/objectives/10/disabled
```

を実行した場合は、
User Aの目的を
無効化しない。

対象不存在として、

```text
OBJECTIVE_NOT_FOUND
```

を返却する。

他利用者に属することを
クライアントへ公開しない。

---

### 5.4 論理削除済み目的

以下の目的は、
OBJ-005の無効化対象としない。

```text
objectives.deleted_at
    IS NOT NULL
```

論理削除済み目的を指定した場合も、
通常の対象不存在として扱う。

概念的には、

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

とする。

---

### 5.5 無効化済み目的

以下の状態の目的は、

```text
enabled = false
```

すでに無効化済みである。

この目的に対して
OBJ-005を再実行した場合は、
再度UPDATEしない。

概念的には、

```text
OBJECTIVE_DISABLED
```

として扱う。

---

### 5.6 userIdをリクエストから受け付けない

Request Bodyへ、

```json
{
  "userId": "2"
}
```

のような
利用者指定を受け付けない。

また、

```text
?userId=2
```

のような
クエリパラメータも使用しない。

利用者の指定経路は、

```text
X-User-Id
```

へ統一する。

---

### 5.7 enabledをリクエストから受け付けない

本APIでは、
`enabled`も
Request Bodyから受け付けない。

以下のようなRequestは使用しない。

```json
{
  "enabled": false
}
```

エンドポイント

```http
PATCH /api/v1/objectives/{objectiveId}/disabled
```

を呼び出すこと自体が、
目的を

```text
enabled = false
```

へ変更する意思を表す。

---

### 5.8 他利用者の目的を取得してから判定しない

以下のように、
目的IDだけで取得した後に

```text
user_id
```

を比較する方式を
基本としない。

```text
objectiveIdで取得
    ↓
user_id比較
```

対象取得時点から、

```text
objectiveId
+
userId
+
deleted_at IS NULL
```

を検索条件に含める。

これにより、
利用者境界外の目的を
不要に取得しない。

---

### 5.9 無効化後も所有利用者は変更しない

OBJ-005成功後も、

```text
objectives.user_id
```

は変更しない。

目的の所有者は
無効化前後で同一とする。

概念的には、

```text
Before

user_id = 1
enabled = true

After

user_id = 1
enabled = false
```

とする。

---

### 5.10 利用者コンテキスト不正時

以下の場合は、
目的検索および
無効化処理へ進まない。

* `X-User-Id`未指定
* `X-User-Id`形式不正
* 指定利用者が存在しない
* 指定利用者が論理削除済み

この場合、

```text
objectives.enabled
```

を変更しない。


---

## 6. パスパラメータ

| パラメータ         | 型      |  必須 | 説明         |
| ------------- | ------ | :-: | ---------- |
| `objectiveId` | string |  ○  | 無効化対象の目的ID |

リクエスト例

```http
PATCH /api/v1/objectives/1/disabled
```

`objectiveId`には、
無効化対象となる目的のIDを指定する。

---

### 6.1 objectiveId

`objectiveId`は、
API共通方針で定めた
ID形式で指定する。

指定された目的について、
以下を確認する。

* 目的が存在すること
* 論理削除されていないこと
* 操作対象利用者に帰属していること
* 有効な目的であること

他の利用者に帰属する目的を指定した場合は、
対象が存在しないものとして扱う。

論理削除済みの目的についても、
対象が存在しないものとして扱う。

無効化済みの目的を指定した場合は、
すでに無効化されているものとして扱う。

---

### 6.2 利用者IDをパスへ含めない

利用者IDは、
パスパラメータとして指定しない。

以下のようなURLは使用しない。

```http
PATCH /api/v1/users/{userId}/objectives/{objectiveId}/disabled
```

操作対象利用者は、
`X-User-Id`から取得する。

そのため、
エンドポイントは以下へ統一する。

```http
PATCH /api/v1/objectives/{objectiveId}/disabled
```

---

## 7. クエリパラメータ

なし。

本APIでは、
クエリパラメータを使用しない。

以下のような指定は受け付けない。

```text
?userId=1
?enabled=false
```

操作対象利用者は、
`X-User-Id`で指定する。

無効化する利用状態については、
エンドポイント自体が

```text
enabled = false
```

への状態遷移を表すため、
クエリパラメータでは指定しない。

---

## 8. リクエストヘッダー

### 8.1 必須ヘッダー

| ヘッダー名          |  必須 | 説明                      |
| -------------- | :-: | ----------------------- |
| `X-User-Id`    |  ○  | 操作対象となる利用者ID            |
| `Content-Type` |  ○  | `application/json`を指定する |
| `Accept`       |  ○  | `application/json`を指定する |

リクエスト例

```http
PATCH /api/v1/objectives/1/disabled
Content-Type: application/json
Accept: application/json
X-User-Id: 1
```

---

### 8.2 X-User-Id

`X-User-Id`には、
操作対象となる利用者IDを指定する。

```http
X-User-Id: 1
```

利用者IDは、
API共通方針で定めた
ID形式で指定する。

以下の場合は、
目的無効化処理へ進まない。

* `X-User-Id`が指定されていない
* 利用者IDの形式が不正である
* 指定された利用者が存在しない
* 指定された利用者が論理削除されている

---

### 8.3 Content-Type

`Content-Type`には、
以下を指定する。

```http
Content-Type: application/json
```

本APIでは
リクエストボディを使用しないが、
API全体のHTTPヘッダー方針を統一するため、
共通方針に従って
`application/json`を指定する。

---

### 8.4 Accept

`Accept`には、
以下を指定する。

```http
Accept: application/json
```

正常時および異常時ともに、
レスポンスはJSON形式で返却する。

---

## 9. リクエストボディ

本APIでは、
リクエストボディを使用しない。

目的を無効化すること自体は、

```http
PATCH /api/v1/objectives/{objectiveId}/disabled
```

というエンドポイントによって表現する。

そのため、
以下のようなリクエストボディは送信しない。

```json
{
  "enabled": false
}
```

本APIを呼び出すこと自体を、

```text
enabled = true
    ↓
enabled = false
```

への状態遷移要求として扱う。

---

### 9.1 enabledを受け付けない理由

本APIは、
目的無効化専用のエンドポイントである。

そのため、
クライアントに

```json
{
  "enabled": false
}
```

を指定させない。

これにより、
以下のような不正な指定も発生しない。

```json
{
  "enabled": true
}
```

Phase1では、
目的の再有効化を提供しない。

---

### 9.2 目的情報を受け付けない

本APIでは、
目的の通常属性も
リクエストボディから受け付けない。

例えば、
以下のようなリクエストは使用しない。

```json
{
  "name": "マイホーム購入",
  "plannedYearMonth": "2031-03",
  "requiredExpense": 6000000,
  "memo": "住宅購入用",
  "enabled": false
}
```

目的情報の更新は、
OBJ-004 目的更新APIで行う。

OBJ-005では、
目的無効化だけを行う。

---

## 10. リクエスト項目

本APIでは、
リクエストボディおよび
クエリパラメータによる
リクエスト項目を受け付けない。

無効化対象の目的は、
パスパラメータ

```text
objectiveId
```

で指定する。

操作対象利用者は、
リクエストヘッダー

```text
X-User-Id
```

で指定する。

無効化後の利用状態は、
API側で

```text
enabled = false
```

へ変更する。

---

### 10.1 入力情報

本APIで使用する入力情報を整理すると、
以下となる。

| 入力元       | 項目            |  必須 | 用途            |
| --------- | ------------- | :-: | ------------- |
| パスパラメータ   | `objectiveId` |  ○  | 無効化対象の目的を特定する |
| リクエストヘッダー | `X-User-Id`   |  ○  | 操作対象利用者を特定する  |
| リクエストボディ  | なし            |  -  | 使用しない         |
| クエリパラメータ  | なし            |  -  | 使用しない         |

---

### 10.2 受け付けない項目

以下の項目は、
リクエストボディまたは
クエリパラメータから受け付けない。

* `id`
* `userId`
* `name`
* `plannedYearMonth`
* `requiredExpense`
* `memo`
* `enabled`
* `createdAt`
* `updatedAt`
* `deletedAt`

OBJ-005では、
これらの値をクライアントから指定して
目的を変更することはできない。

---

### 10.3 enabledの決定主体

`enabled`の値は、
クライアントが指定するのではなく、
OBJ-005の業務処理によって決定する。

```text
OBJ-005実行
    ↓
対象目的を取得
    ↓
現在のenabledを確認
    ↓
enabled = true
    ↓
enabled = falseへ更新
```

すでに

```text
enabled = false
```

の場合は、
再度更新せず、
無効化済みとして扱う。

---

### 10.4 OBJ-004との責務分離

目的情報の変更は、
OBJ-004で行う。

```text
OBJ-004 目的更新

name
plannedYearMonth
requiredExpense
memo
```

目的の利用状態変更は、
OBJ-005で行う。

```text
OBJ-005 目的無効化

enabled
    true → false
```

これにより、
通常属性の更新と
業務上の状態遷移を
別APIとして明確に分離する。

---

## 11. バリデーション

OBJ-005では、
リクエストボディを使用しないため、
Request Bodyに対する
項目バリデーションは存在しない。

一方、
以下について検証する。

* `X-User-Id`
* `objectiveId`
* 操作対象利用者の存在
* 対象目的の存在
* 利用者境界
* 論理削除状態
* 現在の利用状態

概念的には、
以下の順序で確認する。

```text
X-User-Id
    ↓
利用者コンテキスト検証
    ↓
objectiveId
    ↓
形式検証
    ↓
目的取得
    ↓
利用者境界確認
    ↓
論理削除状態確認
    ↓
enabled確認
    ↓
無効化処理
```

---

### 11.1 X-User-Id

`X-User-Id`は、
API共通方針に従って検証する。

主に以下を確認する。

* 必須であること
* API共通方針で定めたID形式であること
* 対象利用者が存在すること
* 対象利用者が論理削除されていないこと

利用者コンテキストが
確立できない場合は、
目的取得処理へ進まない。

---

### 11.2 objectiveId

`objectiveId`は、
API共通方針で定めた
ID形式であることを確認する。

例えば、
数値IDを使用する場合は、
正の整数として解釈できることを確認する。

不正な形式の場合は、

```text
INVALID_OBJECTIVE_ID
```

として扱う。

---

### 11.3 Request Bodyのバリデーション

本APIでは、
Request Bodyを使用しない。

そのため、

```text
name
plannedYearMonth
requiredExpense
memo
enabled
```

に対する
FormRequestの項目バリデーションは
行わない。

---

### 11.4 業務状態の検証

以下は、
単純な入力形式ではなく、
現在のDB状態に依存する。

* 目的が存在するか
* 操作対象利用者に帰属しているか
* 論理削除されていないか
* すでに無効化されていないか

これらは、
入力値の形式検証とは分離して
業務処理内で判定する。

---

### 11.5 他利用者の目的

以下の場合、

```text
objective.id = objectiveId

AND

objective.user_id != currentUserId
```

対象目的を
取得できないものとして扱う。

他利用者の目的であることを
クライアントへ通知しない。

```text
OBJECTIVE_NOT_FOUND
```

として扱う。

---

### 11.6 論理削除済み目的

以下の場合、

```text
deleted_at IS NOT NULL
```

OBJ-005の対象外とする。

論理削除済みであることを
個別に公開せず、

```text
OBJECTIVE_NOT_FOUND
```

として扱う。

---

### 11.7 無効化済み目的

対象目的が、

```text
enabled = false
```

の場合は、
すでに無効化済みである。

この場合、
再度

```text
enabled = false
```

へUPDATEしない。

業務エラーとして、

```text
OBJECTIVE_DISABLED
```

を返却する。

---

### 11.8 バリデーションと業務ルールを分離する

概念的には、
以下のように責務を分離する。

```text
入力形式検証
    ↓
X-User-Id
objectiveId

業務状態検証
    ↓
利用者存在
目的存在
利用者境界
論理削除状態
enabled状態
```

HTTP入力値の形式と、
現在の業務状態を
同一のバリデーションとして
扱わない。

---

## 12. 業務ルール

OBJ-005では、
以下の業務ルールに従って
目的を無効化する。

---

### 12.1 操作対象利用者の目的のみ無効化できる

無効化できる目的は、

```text
objective.user_id
    =
currentUserId
```

を満たすものに限定する。

他利用者の目的は
無効化できない。

---

### 12.2 論理削除されていない目的のみ対象とする

対象目的は、

```text
deleted_at IS NULL
```

である必要がある。

論理削除済み目的は、
OBJ-005の対象外とする。

---

### 12.3 有効な目的のみ無効化できる

対象目的は、

```text
enabled = true
```

である必要がある。

OBJ-005成功時に、

```text
enabled = true
    ↓
enabled = false
```

へ変更する。

---

### 12.4 無効化済み目的を再度無効化しない

対象目的がすでに、

```text
enabled = false
```

の場合は、
UPDATEを実行しない。

```text
OBJECTIVE_DISABLED
```

として扱う。

---

### 12.5 目的の通常属性を変更しない

OBJ-005では、
以下を変更しない。

* `name`
* `planned_year_month`
* `required_expense`
* `memo`
* `user_id`

目的内容を変更する場合は、
OBJ-004を使用する。

---

### 12.6 deleted_atを変更しない

目的無効化では、

```text
deleted_at
```

を更新しない。

OBJ-005は
論理削除APIではない。

成功後も、

```text
deleted_at = NULL
```

を維持する。

---

### 12.7 updated_atは更新する

`enabled`の変更に伴い、

```text
updated_at
```

は更新する。

Laravelの
Eloquentによる通常更新を使用する場合は、
フレームワークによる
自動更新を利用してよい。

---

### 12.8 無効化後も目的を参照可能とする

OBJ-005成功後も、
目的レコード自体は保持する。

そのため、
OBJ-001 目的一覧取得および
OBJ-003 目的詳細取得では、
API仕様に従って
無効化済み目的を参照できる。

概念的には、

```text
Objective

enabled = false
deleted_at = NULL

    ↓

過去情報として参照可能
```

とする。

---

### 12.9 新しい目的達成判定の対象外とする

無効化済み目的は、
新しく実行する
目的達成判定の対象外とする。

概念的には、

```text
enabled = true
    ↓
判定対象

enabled = false
    ↓
判定対象外
```

とする。

---

### 12.10 過去の目的達成判定履歴を変更しない

OBJ-005を実行しても、

```text
assessment_histories
```

は更新しない。

以下を行わない。

* 過去判定結果の削除
* 過去判定結果の更新
* 過去判定結果の再計算
* 目的無効化時点での新規判定

過去の判定結果は、
判定実行時点の履歴として保持する。

---

### 12.11 再有効化しない

Phase1では、
OBJ-005によって無効化した目的を

```text
enabled = false
    ↓
enabled = true
```

へ戻す処理は提供しない。

再有効化が必要になった場合は、
別ユースケースとして設計する。

---

### 12.12 OBJ-004からenabledを変更しない

OBJ-004 目的更新では、
`enabled`を更新対象としない。

責務を以下のように分離する。

```text
OBJ-004
    ↓
目的通常属性更新

name
plannedYearMonth
requiredExpense
memo

OBJ-005
    ↓
目的無効化

enabled
true → false
```

---

### 12.13 無効化と削除を同一視しない

目的の無効化は、

```text
今後利用しない
```

という業務状態を表す。

一方、
論理削除は、

```text
通常の参照対象から除外する
```

というデータ管理上の状態である。

そのため、

```text
enabled
```

と

```text
deleted_at
```

を別の状態として管理する。

---

### 12.14 状態遷移

Phase1における
目的の利用状態遷移は、
以下とする。

```text
目的登録
    ↓
enabled = true
    ↓
OBJ-005
    ↓
enabled = false
```

OBJ-005では、
逆方向の状態遷移は行わない。

---

## 13. 更新処理

OBJ-005では、
対象目的の

```text
enabled
```

を更新する。

概念的な処理は、
以下とする。

```text
利用者コンテキスト取得
    ↓
objectiveId検証
    ↓
対象目的取得
    ↓
enabled確認
    ↓
enabled = false
    ↓
保存
    ↓
更新結果返却
```

---

### 13.1 更新前

更新対象となる目的は、
以下の状態である。

```text
user_id
    = currentUserId

deleted_at
    = NULL

enabled
    = true
```

---

### 13.2 更新内容

更新する項目は、
以下とする。

```text
enabled = false
```

これに伴い、

```text
updated_at
```

も更新される。

---

### 13.3 更新後

更新後は、
概念的に以下の状態となる。

```text
Before

id = 1
user_id = 1
name = "一人暮らし"
enabled = true
deleted_at = NULL

    ↓
OBJ-005

After

id = 1
user_id = 1
name = "一人暮らし"
enabled = false
deleted_at = NULL
```

---

### 13.4 更新対象を限定する

Repositoryでは、
クライアント入力を
そのままUPDATEしない。

例えば、

```php
$objective->update([
    'enabled' => false,
]);
```

のように、
更新対象を明示する。

以下のような処理は行わない。

```php
$objective->update(
    $request->all(),
);
```

---

### 13.5 更新件数

正常時に更新する
`objectives`レコードは
1件のみとする。

複数目的を
一括無効化するAPIではない。

---

### 13.6 関連テーブルを更新しない

OBJ-005では、
原則として

```text
objectives
```

だけを更新する。

以下の関連データを
更新しない。

```text
assessment_histories
```

また、
目的無効化を契機として
他の資産関連データを
更新しない。

---

## 14. トランザクション境界

OBJ-005で更新する
業務データは、

```text
objectives
```

の1レコードのみである。

そのため、
複数テーブル更新を
原子的にまとめる必要はない。

Phase1では、
明示的なDBトランザクションを
必須としない。

概念的には、

```text
目的取得
    ↓
状態確認
    ↓
objectives.enabled更新
```

という
単一レコード更新として扱う。

---

### 14.1 assessment_historiesをトランザクションへ含めない

目的無効化時に、

```text
assessment_histories
```

を更新しない。

そのため、

```text
objectives更新
+
assessment_histories更新
```

のような
複数テーブル更新を
同一トランザクションで扱う必要はない。

---

### 14.2 将来トランザクションが必要になる場合

将来的に、
目的無効化と同時に

* 監査履歴登録
* ドメインイベント保存
* 関連状態更新
* その他の業務データ更新

などを
同期的に行う場合は、
トランザクション境界を
再検討する。

その場合は、

```text
目的無効化
+
関連更新
```

を1つの業務操作として
原子的に扱う。

---

### 14.3 Repository単位で不要なトランザクションを張らない

単一UPDATEしか行わないため、
Repositoryメソッド内部で

```php
DB::transaction(...)
```

を必須とはしない。

トランザクションは、
複数更新を
1つの業務操作として
保証する必要がある場合に使用する。

---

## 15. 排他制御

OBJ-005では、
同一目的に対して
複数の無効化リクエストが
同時実行される可能性がある。

そのため、
更新時には
現在状態を条件へ含め、
有効状態から無効状態への
遷移だけを成立させる。

概念的には、

```text
WHERE
    id = objectiveId
AND
    user_id = currentUserId
AND
    deleted_at IS NULL
AND
    enabled = true
```

を満たす場合のみ、

```text
enabled = false
```

へ更新する。

---

### 15.1 同時無効化

例えば、
同一目的に対して

```text
Request A
PATCH /objectives/1/disabled

Request B
PATCH /objectives/1/disabled
```

が
ほぼ同時に実行された場合を考える。

概念的には、

```text
初期状態
enabled = true

        ┌─ Request A
        │      ↓
        │  enabled = false
        │
        └─ Request B
               ↓
           enabled = true
           を満たさない
```

となるようにする。

---

### 15.2 二重更新を正常成功させない

先行Requestによって
すでに

```text
enabled = false
```

となった場合、
後続Requestでは
再度UPDATEしない。

後続Requestは、
最新状態を確認したうえで

```text
OBJECTIVE_DISABLED
```

として扱う。

---

### 15.3 lockForUpdateを必須としない

OBJ-005では、

```text
enabled = true
```

を更新条件へ含めた
条件付きUPDATEによって、
状態遷移を原子的に制御できる。

そのため、
Phase1では

```php
lockForUpdate()
```

による
悲観ロックを必須としない。

---

### 15.4 条件付きUPDATEを使用する理由

以下のように、

```text
SELECT
    ↓
enabled確認
    ↓
UPDATE
```

だけで処理すると、
SELECTとUPDATEの間に
別Requestが更新する可能性がある。

そのため、
最終的なUPDATE条件にも

```text
enabled = true
```

を含める。

---

### 15.5 更新件数を確認する

条件付きUPDATEの結果として、
更新件数を確認する。

概念的には、

```text
更新件数 = 1
    ↓
無効化成功

更新件数 = 0
    ↓
最新状態確認
```

とする。

更新件数が0件の場合は、
対象不存在なのか、
すでに無効化済みなのかを
必要に応じて判定する。

---

### 15.6 更新件数0件の場合

更新件数が0件の場合は、
対象目的の最新状態を確認する。

概念的には、

```text
UPDATE 0件
    ↓
目的再確認
    ├─ 存在しない
    │      ↓
    │  OBJECTIVE_NOT_FOUND
    │
    └─ enabled = false
           ↓
       OBJECTIVE_DISABLED
```

とする。

他利用者所属または
論理削除済みの場合も、

```text
OBJECTIVE_NOT_FOUND
```

として扱う。

---

### 15.7 楽観ロックを必須としない

Phase1では、

```text
version
lock_version
```

などの
専用バージョンカラムを
追加しない。

OBJ-005では、

```text
enabled = true
```

という現在状態を
更新条件として利用することで、
必要な競合制御を行う。

---

### 15.8 ETagを使用しない

Phase1では、

```text
ETag
If-Match
```

による
HTTPレベルの
楽観的排他制御は使用しない。

目的無効化に必要な
競合制御は、
DB更新条件によって行う。

---

### 15.9 Phase1の排他制御方針

Phase1では、
以下の構成とする。

```text
利用者境界付き対象確認
    ↓
現在状態確認
    ↓
enabled = true
を条件としたUPDATE
    ↓
更新件数確認
    ↓
0件なら最新状態再確認
```

これにより、
不要にロック範囲を広げず、
目的の

```text
有効
    ↓
無効
```

という
一方向の状態遷移を保証する。

---

## 16. 成功レスポンス

目的の無効化に成功した場合は、
更新後の目的情報を返却する。

HTTPステータスは、

```http
200 OK
```

とする。

レスポンス例：

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

OBJ-005成功後は、

```text
enabled = false
```

となった目的を返却する。

---

### 16.1 200 OKを使用する理由

OBJ-005では、
既存の目的リソースを更新する。

新しいリソースを
作成する処理ではないため、

```http
201 Created
```

は使用しない。

また、
更新後の目的情報を
レスポンスとして返却するため、

```http
204 No Content
```

ではなく、

```http
200 OK
```

を使用する。

---

### 16.2 更新後の状態を返却する

成功レスポンスでは、
無効化前ではなく、
無効化後の状態を返却する。

```text
Before

enabled = true

    ↓
OBJ-005

After

enabled = false
```

React側では、
このレスポンスによって
無効化処理が成功したことを
確認できる。

---

### 16.3 Envelope形式

API共通方針に従い、
正常レスポンスは

```text
data
```

でラップする。

概念的には、

```json
{
  "data": {
    "...": "..."
  }
}
```

とする。

OBJ-005独自の
Envelope形式は定義しない。

---

## 17. レスポンス項目

`data`配下には、
更新後の目的情報を返却する。

| 項目                 | 型       | NULL | 説明                      |
| ------------------ | ------- | :--: | ----------------------- |
| `id`               | string  |   ×  | 目的ID                    |
| `name`             | string  |   ×  | 目的名                     |
| `plannedYearMonth` | string  |   ×  | 実施予定年月。`YYYY-MM`形式      |
| `requiredExpense`  | integer |   ×  | 目的達成に必要な支出額             |
| `memo`             | string  |   ○  | 目的に関するメモ                |
| `enabled`          | boolean |   ×  | 利用状態。OBJ-005成功時は`false` |

---

### 17.1 id

目的IDを
文字列として返却する。

```json
{
  "id": "1"
}
```

データベース上の
`bigint`を
そのままJSON Numberとして扱わず、
API共通方針に従って
文字列として返却する。

---

### 17.2 name

目的名を返却する。

```json
{
  "name": "一人暮らし"
}
```

OBJ-005では、
`name`を変更しない。

そのため、
無効化前と同じ値を返却する。

---

### 17.3 plannedYearMonth

実施予定年月を、

```text
YYYY-MM
```

形式で返却する。

例：

```json
{
  "plannedYearMonth": "2027-04"
}
```

OBJ-005では、
実施予定年月を変更しない。

---

### 17.4 requiredExpense

目的達成に必要な
支出額を返却する。

```json
{
  "requiredExpense": 500000
}
```

日本円の整数値として扱う。

OBJ-005では、
必要支出額を変更しない。

---

### 17.5 memo

目的に登録された
メモを返却する。

```json
{
  "memo": "引っ越し費用を含む"
}
```

メモが登録されていない場合は、

```json
{
  "memo": null
}
```

とする。

OBJ-005では、
メモを変更しない。

---

### 17.6 enabled

目的の利用状態を
booleanで返却する。

OBJ-005成功時は、
必ず

```json
{
  "enabled": false
}
```

となる。

文字列の

```json
{
  "enabled": "false"
}
```

として返却しない。

---

### 17.7 deletedAtを返却しない

OBJ-005は、
目的の論理削除APIではない。

そのため、
通常のAPIレスポンスとして

```text
deletedAt
```

を公開する必要はない。

論理削除状態は
サーバー内部の
データ管理情報として扱う。

---

### 17.8 userIdを返却しない

操作対象利用者は、

```text
X-User-Id
```

によって確定している。

そのため、
OBJ-005のレスポンスで
`userId`を返却する必要はない。

利用者境界は
サーバー側で保証する。

---

### 17.9 createdAt・updatedAt

OBJ-005の
主要な業務レスポンスとしては、

```text
createdAt
updatedAt
```

を必須項目としない。

目的API全体で
日時情報を返却する方針とする場合は、
OBJ-001からOBJ-005まで
統一して扱う。

OBJ-005だけで
独自に日時項目を追加しない。

---

## 18. エラーレスポンス

エラー時は、
API共通方針で定めた
エラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "OBJECTIVE_NOT_FOUND",
    "message": "指定された目的が見つかりません。"
  }
}
```

OBJ-005独自の
エラーレスポンスEnvelopeは
定義しない。

---

### 18.1 利用者コンテキストエラー

`X-User-Id`に問題がある場合は、
API共通の
利用者コンテキストエラーを返却する。

主なエラーは、
以下とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
```

この場合、
目的の取得および
無効化処理は行わない。

---

### 18.2 objectiveId形式不正

`objectiveId`が
API共通方針で定めた
ID形式として不正な場合は、

```text
INVALID_OBJECTIVE_ID
```

を返却する。

例えば、
数値IDを前提とする場合に、
数値として解釈できない値などを
指定した場合である。

---

### 18.3 目的不存在

以下の場合は、

```text
OBJECTIVE_NOT_FOUND
```

を返却する。

* 指定された目的が存在しない
* 他利用者に帰属している
* 論理削除済みである

これらを
クライアント側で区別できる
エラーにはしない。

---

### 18.4 他利用者所属

他利用者の目的を指定した場合も、

```text
OBJECTIVE_NOT_FOUND
```

とする。

例えば、

```text
X-User-Id = 2

objectiveId = 10
objective.user_id = 1
```

の場合でも、

```text
OBJECTIVE_FORBIDDEN
```

のような
エラーは返却しない。

他利用者に属する目的の存在を
公開しないためである。

---

### 18.5 論理削除済み

対象目的が、

```text
deleted_at IS NOT NULL
```

の場合も、

```text
OBJECTIVE_NOT_FOUND
```

として扱う。

論理削除済みであることを
クライアントへ公開しない。

---

### 18.6 無効化済み

対象目的が、

```text
enabled = false
```

の場合は、

```text
OBJECTIVE_DISABLED
```

を返却する。

レスポンス例：

```json
{
  "error": {
    "code": "OBJECTIVE_DISABLED",
    "message": "指定された目的はすでに無効化されています。"
  }
}
```

この場合、
目的を再度UPDATEしない。

---

### 18.7 サーバー内部エラー

予期しないエラーが
発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

として扱う。

内部例外、
SQL、
スタックトレースなどの
詳細情報を
APIレスポンスへ公開しない。

詳細は
サーバーログへ記録する。

---

### 18.8 エラーメッセージ

クライアント側では、
原則として

```text
error.code
```

を基準として
エラー種別を判定する。

`message`は、
利用者向け表示または
補助情報として扱う。

フロントエンドの
業務分岐を
エラーメッセージ文字列へ
依存させない。

---

## 19. HTTPステータス

OBJ-005では、
以下のHTTPステータスを使用する。

| HTTPステータス                   | 用途           | 主なエラーコード                                 |
| --------------------------- | ------------ | ---------------------------------------- |
| `200 OK`                    | 目的無効化成功      | -                                        |
| `400 Bad Request`           | ID形式不正など     | `INVALID_USER_ID`、`INVALID_OBJECTIVE_ID` |
| `404 Not Found`             | 利用者または目的不存在  | `USER_NOT_FOUND`、`OBJECTIVE_NOT_FOUND`   |
| `409 Conflict`              | 現在の業務状態と競合   | `OBJECTIVE_DISABLED`                     |
| `500 Internal Server Error` | 予期しないサーバーエラー | `INTERNAL_SERVER_ERROR`                  |

`X-User-Id`未指定時の
HTTPステータスについては、
API共通方針に従う。

---

### 19.1 200 OK

目的の無効化に成功した場合は、

```http
200 OK
```

を返却する。

レスポンスには、
無効化後の目的情報を含める。

---

### 19.2 400 Bad Request

入力値の形式が
API仕様に適合しない場合に使用する。

主な対象は、

```text
INVALID_USER_ID
INVALID_OBJECTIVE_ID
```

とする。

---

### 19.3 404 Not Found

対象リソースを
取得できない場合に使用する。

主な対象は、

```text
USER_NOT_FOUND
OBJECTIVE_NOT_FOUND
```

とする。

他利用者所属や
論理削除済み目的も、
`OBJECTIVE_NOT_FOUND`として
404を返却する。

---

### 19.4 409 Conflict

Request自体は
形式上正しいが、
現在の業務状態によって
処理できない場合に使用する。

OBJ-005では、

```text
OBJECTIVE_DISABLED
```

を対象とする。

すでに

```text
enabled = false
```

となっている目的に対する
再無効化要求は、
現在状態との競合として扱う。

---

### 19.5 500 Internal Server Error

予期しない
サーバー内部エラーの場合は、

```http
500 Internal Server Error
```

を返却する。

内部実装の詳細は
レスポンスへ公開しない。

---

## 20. エラーコード

OBJ-005で使用する
主なエラーコードは、
以下とする。

| エラーコード                  |  HTTPステータス | 発生条件                 |
| ----------------------- | ---------: | -------------------- |
| `USER_CONTEXT_REQUIRED` | API共通方針に従う | `X-User-Id`が指定されていない |
| `INVALID_USER_ID`       |      `400` | `X-User-Id`の形式が不正    |
| `USER_NOT_FOUND`        |      `404` | 指定利用者が存在しない、または利用対象外 |
| `INVALID_OBJECTIVE_ID`  |      `400` | `objectiveId`の形式が不正  |
| `OBJECTIVE_NOT_FOUND`   |      `404` | 目的不存在、他利用者所属、論理削除済み  |
| `OBJECTIVE_DISABLED`    |      `409` | 指定された目的がすでに無効化済み     |
| `INTERNAL_SERVER_ERROR` |      `500` | 予期しないサーバー内部エラー       |

---

### 20.1 USER_CONTEXT_REQUIRED

`X-User-Id`が
指定されていない場合に使用する。

目的無効化処理へは進まない。

---

### 20.2 INVALID_USER_ID

`X-User-Id`が
API共通方針で定めた
ID形式として不正な場合に使用する。

---

### 20.3 USER_NOT_FOUND

指定された利用者が
存在しない、
または利用対象として
扱えない場合に使用する。

目的の検索処理へは進まない。

---

### 20.4 INVALID_OBJECTIVE_ID

`objectiveId`が
API共通方針で定めた
ID形式として不正な場合に使用する。

---

### 20.5 OBJECTIVE_NOT_FOUND

以下の場合に使用する。

```text
目的不存在
他利用者所属
論理削除済み
```

これらを
同一エラーコードとして扱うことで、
利用者境界外の目的や
論理削除済み目的の存在を
クライアントへ公開しない。

---

### 20.6 OBJECTIVE_DISABLED

対象目的が
すでに

```text
enabled = false
```

の場合に使用する。

この場合、
UPDATEを行わない。

同時無効化によって
先行Requestが成功し、
後続Requestで
無効化済みとなっていた場合も、
同じエラーコードを使用する。

---

### 20.7 INTERNAL_SERVER_ERROR

想定外の例外や
データベースエラーなどによって
正常に処理できない場合に使用する。

クライアントには
内部例外の詳細を公開しない。

---

### 20.8 エラーコードの責務

エラーコードは、
クライアントが
エラー種別を安定して判定するために使用する。

React側では、

```ts
switch (error.code) {
  case 'OBJECTIVE_NOT_FOUND':
    // 対象不存在として処理
    break;

  case 'OBJECTIVE_DISABLED':
    // 無効化済みとして処理
    break;
}
```

のように、
`message`ではなく
`code`を基準として
分岐できるようにする。

---

## 21. 冪等性

OBJ-005は、同一の有効な目的に対して最初の1回だけ状態変更を行う。

概念的には、

```text
enabled = true
    ↓
OBJ-005
    ↓
enabled = false
```

となる。

2回目以降は、すでに無効化済みのため、再度更新しない。

---

### 21.1 同一Requestの再実行

例えば、以下のRequestを実行する。

```http
PATCH /api/v1/objectives/1/disabled
X-User-Id: 1
```

1回目は、対象目的が

```text
enabled = true
```

であれば、

```text
enabled = false
```

へ更新する。

レスポンスは、

```http
200 OK
```

とする。

---

### 21.2 2回目以降

1回目の処理後は、

```text
enabled = false
```

となっている。

同じRequestを再実行した場合は、再度UPDATEしない。

概念的には、

```text
2回目
    ↓
enabled = false
    ↓
OBJECTIVE_DISABLED
```

とする。

---

### 21.3 HTTP操作としての扱い

OBJ-005は、同一Requestを複数回実行しても最終的な業務状態は、

```text
enabled = false
```

から変化しない。

その意味では、状態遷移自体は冪等な性質を持つ。

ただし、APIレスポンスとしては、

```text
1回目
    → 200 OK

2回目以降
    → 409 Conflict
       OBJECTIVE_DISABLED
```

となる。

そのため、「常に同じレスポンスを返す」という意味での冪等性は保証しない。

---

### 21.4 再無効化を成功扱いしない理由

すでに無効化済みの目的に対して、

```http
200 OK
```

を返却する設計も可能である。

ただし、Phase1では、

```text
今回のRequestによって
実際に状態変更されたか
```

を明確にするため、無効化済みの場合は

```text
OBJECTIVE_DISABLED
```

として扱う。

---

### 21.5 二重送信

画面上で無効化ボタンが二重送信された場合でも、2件目のRequestによって新たな状態変更は発生しない。

概念的には、

```text
Request A
    ↓
enabled = true
    ↓
enabled = false
    ↓
200 OK

Request B
    ↓
enabled = false
    ↓
UPDATEなし
    ↓
OBJECTIVE_DISABLED
```

となる。

---

### 21.6 Idempotency-Key

Phase1では、OBJ-005専用の

```text
Idempotency-Key
```

を使用しない。

理由は、目的の状態が

```text
enabled = true
    ↓
enabled = false
```

という一方向の状態遷移であり、

```text
enabled = true
```

を条件とした更新によって二重更新を防止できるためである。

---

### 21.7 自動Retry

OBJ-005は更新APIであるため、フロントエンドで無条件の自動Retryを前提としない。

通信結果が不明な場合は、OBJ-003 目的詳細取得などで最新状態を再取得し、

```text
enabled
```

を確認してから次の操作を判断する。

---

## 22. キャッシュ

OBJ-005自体は状態変更を行うPATCH APIであるため、レスポンスをHTTPキャッシュ対象として再利用することは想定しない。

Phase1では、OBJ-005専用のサーバー側アプリケーションキャッシュも使用しない。

---

### 22.1 サーバー側キャッシュ

Phase1では、以下のようなOBJ-005専用キャッシュを導入しない。

```text
Redis
Laravel Cache
Application Memory Cache
```

目的無効化は高頻度に実行される処理ではなく、DB上の最新状態との整合性を優先する。

---

### 22.2 OBJ-001への影響

OBJ-001 目的一覧取得が、目的の

```text
enabled
```

を返却する場合は、OBJ-005成功後に一覧Query Cacheを無効化する。

概念的には、

```text
OBJ-005成功
    ↓
OBJ-001 Query invalidate
    ↓
目的一覧再取得
```

とする。

---

### 22.3 OBJ-003への影響

OBJ-003 目的詳細取得では、無効化後の

```text
enabled = false
```

を反映する必要がある。

そのため、OBJ-005成功後は、対象目的の詳細Query Cacheも無効化する。

---

### 22.4 Query Cacheの無効化

TanStack Queryを使用する場合は、概念的に以下のようにする。

```typescript
await queryClient.invalidateQueries({
  queryKey: objectiveKeys.list(),
});

await queryClient.invalidateQueries({
  queryKey: objectiveKeys.detail(
    objectiveId,
  ),
});
```

具体的なQuery Keyは、React共通設計に従う。

---

### 22.5 成功レスポンスで直接Cache更新してもよい

OBJ-005成功レスポンスでは、更新後の目的情報が返却される。

そのため、技術的には

```text
enabled = false
```

となったレスポンスを使用して対象詳細Cacheを直接更新してもよい。

ただし、Phase1では実装を単純にするため、

```text
OBJ-005成功
    ↓
invalidate
    ↓
再取得
```

を基本としてよい。

---

### 22.6 目的一覧Cache

目的一覧に無効化済み目的を含める場合は、再取得後も対象目的は一覧へ残り、

```text
enabled = false
```

として表示される。

一覧が有効目的だけを返却する設計の場合は、OBJ-005成功後に対象目的が一覧から除外される可能性がある。

具体的な挙動は、OBJ-001の仕様を正とする。

---

### 22.7 目的達成判定関連Cache

OBJ-005成功後、新しい目的達成判定では無効化済み目的を対象外とする。

そのため、有効目的一覧を前提にした判定画面やQuery Cacheが存在する場合は、必要に応じて無効化する。

一方、

```text
assessment_histories
```

に保存済みの過去判定履歴は変更されないため、過去判定履歴CacheをOBJ-005成功だけで書き換えない。

---

### 22.8 利用者境界とCache

Phase1では、`X-User-Id`によって操作対象利用者を切り替える。

そのため、目的関連のQuery Cacheでも利用者境界を考慮する。

必要に応じて、

```text
userId
+
objectiveId
```

をQuery Keyへ含める、または利用者切替時に目的関連Cacheを無効化する。

---

### 22.9 HTTPキャッシュ

Phase1では、OBJ-005について

```text
ETag
Last-Modified
If-Match
```

などのHTTPキャッシュ・条件付き更新機構を導入しない。

排他制御は、`enabled = true`を条件としたDB更新によって行う。

---

## 23. 関連テーブル

OBJ-005では、以下のテーブルを使用する。

| テーブル                   | 用途                  |  更新 |
| ---------------------- | ------------------- | :-: |
| `users`                | 操作対象利用者の確認          |  ×  |
| `objectives`           | 無効化対象の目的取得、利用状態更新   |  ○  |
| `assessment_histories` | 過去判定履歴が存在していても変更しない |  ×  |

---

### 23.1 users

`X-User-Id`で指定された操作対象利用者の存在確認に使用する。

概念的には、

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

を確認する。

利用者コンテキスト確認は、API共通Middlewareで行う。

OBJ-005では、`users`を更新しない。

---

### 23.2 objectives

OBJ-005の主たる対象テーブルである。

主に以下のカラムを使用する。

| カラム          | 用途                |
| ------------ | ----------------- |
| `id`         | `objectiveId`との照合 |
| `user_id`    | 操作対象利用者との利用者境界確認  |
| `enabled`    | 現在の利用状態確認および無効化   |
| `deleted_at` | 論理削除状態確認          |
| `updated_at` | 無効化時に更新           |

---

### 23.3 対象目的取得条件

対象目的は、概念的に以下の条件で取得する。

```text
objectives.id
    = objectiveId

AND

objectives.user_id
    = 操作対象利用者ID

AND

objectives.deleted_at
    IS NULL
```

他利用者所属、論理削除済み目的は取得対象外とする。

---

### 23.4 更新条件

最終的な無効化UPDATEでは、少なくとも

```text
objectives.id
    = objectiveId

AND

objectives.user_id
    = 操作対象利用者ID

AND

objectives.deleted_at
    IS NULL

AND

objectives.enabled
    = true
```

を条件とする。

この条件を満たした場合のみ、

```text
enabled = false
```

へ更新する。

---

### 23.5 更新対象カラム

OBJ-005で業務上明示的に変更するカラムは、

```text
enabled
```

のみとする。

EloquentのTimestamp機能により、

```text
updated_at
```

も更新される。

---

### 23.6 更新しないカラム

以下は変更しない。

```text
user_id
name
planned_year_month
required_expense
memo
created_at
deleted_at
```

---

### 23.7 assessment_histories

過去に対象目的に対する目的達成判定履歴が存在していても、OBJ-005では更新しない。

概念的には、

```text
objectives
    enabled
        true → false

assessment_histories
    変更なし
```

とする。

---

### 23.8 過去判定履歴を削除しない

目的が無効化されても、

```text
assessment_histories
```

を削除しない。

過去判定結果は、判定実行時点の履歴として保持する。

---

### 23.9 他テーブルへの副作用

OBJ-005では、以下のような資産関連テーブルを更新しない。

* `asset_accounts`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`

目的無効化の責務を`objectives.enabled`の変更へ限定する。

---

## 24. 設計上の注意

### 24.1 OBJ-004とOBJ-005の責務を混在させない

OBJ-004は、目的の通常属性を更新するAPIとする。

```text
OBJ-004

name
plannedYearMonth
requiredExpense
memo
```

一方、OBJ-005は、目的の利用状態を変更するAPIとする。

```text
OBJ-005

enabled
true → false
```

そのため、OBJ-004のRequest Bodyへ

```json
{
  "enabled": false
}
```

を追加して目的無効化を実現しない。

---

### 24.2 `/disabled`を業務操作として扱う

OBJ-005では、

```http
PATCH /api/v1/objectives/{objectiveId}/disabled
```

を使用する。

このURLは、単なる属性名の更新というより、

```text
目的を無効化する
```

という明示的な業務操作として扱う。

OBJ-004と同一の

```http
PATCH /api/v1/objectives/{objectiveId}
```

へ割り当てない。

---

### 24.3 Request Bodyを持たせない

OBJ-005では、無効化すること自体がエンドポイントで表現されている。

そのため、以下のようなRequest Bodyを必要としない。

```json
{
  "enabled": false
}
```

また、

```json
{
  "enabled": true
}
```

による再有効化もできない。

---

### 24.4 enabledを通常編集項目にしない

`enabled`は、目的編集フォームで自由に切り替える属性として扱わない。

概念的には、

```text
通常属性
    → 編集フォーム

enabled
    → 専用の無効化操作
```

とする。

これにより、業務上重要な状態遷移を通常編集と区別する。

---

### 24.5 無効化と論理削除を区別する

OBJ-005では、

```text
enabled = false
```

へ変更する。

一方、

```text
deleted_at
```

は更新しない。

概念的には、

```text
enabled = false
    ↓
今後利用しないが
過去情報として参照可能

deleted_at IS NOT NULL
    ↓
通常参照対象から除外
```

という別の状態として扱う。

---

### 24.6 SoftDeletesを無効化用途に使用しない

Laravelの

```php
$objective->delete();
```

をOBJ-005で使用しない。

これを使用すると、

```text
deleted_at
```

が更新され、目的無効化ではなく論理削除になってしまう。

OBJ-005では、明示的に

```text
enabled = false
```

だけを変更する。

---

### 24.7 無効化後も目的を保持する

OBJ-005成功後も、`objectives`レコードを保持する。

これにより、

* 過去に設定していた目的
* 無効化時点の目的情報
* 過去の目的達成判定との関係

を後から確認できる。

---

### 24.8 過去判定履歴を無効化に追従させない

目的を無効化しても、

```text
assessment_histories
```

を更新しない。

過去判定履歴は、

```text
判定実行時点の結果
```

として保持する。

現在の目的状態に合わせて過去の結果を書き換えない。

---

### 24.9 新しい判定だけ対象外とする

`enabled = false`となった目的は、今後新しく実行される目的達成判定の対象外とする。

概念的には、

```text
過去の判定履歴
    → 保持

将来の判定
    → 対象外
```

とする。

---

### 24.10 無効化済み目的を再無効化しない

すでに

```text
enabled = false
```

の目的に対してOBJ-005を実行した場合は、再度UPDATEしない。

```text
OBJECTIVE_DISABLED
```

として扱う。

これにより、

```text
updated_atだけが
無意味に更新される
```

ことも防止する。

---

### 24.11 再無効化を200 OKにしない

OBJ-005は、実際に

```text
enabled = true
    ↓
enabled = false
```

という状態変更が行われた場合だけ成功とする。

すでに無効化済みの場合は、

```text
409 Conflict
OBJECTIVE_DISABLED
```

とする。

Phase1では、無効化操作が実際に成立したかをレスポンスから明確に判断できる設計とする。

---

### 24.12 条件付きUPDATEを使用する

OBJ-005では、最終UPDATE条件へ

```text
enabled = true
```

を含める。

概念的には、

```text
WHERE
    id = objectiveId
AND
    user_id = userId
AND
    deleted_at IS NULL
AND
    enabled = true
```

として、

```text
enabled = false
```

へ更新する。

これにより、同時実行時の二重更新を防止する。

---

### 24.13 SELECT時の状態だけを信用しない

以下のような処理だけでは、競合が発生する可能性がある。

```text
SELECT
    ↓
enabled = true確認
    ↓
別Requestが無効化
    ↓
UPDATE
```

そのため、SELECT時の確認に加えて、UPDATE条件にも

```text
enabled = true
```

を含める。

---

### 24.14 lockForUpdateを必須にしない

OBJ-005は、単一レコードの一方向状態遷移である。

そのため、Phase1では

```php
lockForUpdate()
```

による悲観ロックではなく、

```text
状態条件付きUPDATE
+
更新件数確認
```

で競合を制御する。

不要にロック範囲を広げない。

---

### 24.15 更新件数0件を単純にOBJECTIVE_DISABLEDとしない

条件付きUPDATE結果が0件だった場合は、

```text
すでに無効化済み
```

とは限らない。

例えば、更新直前に

* 目的が論理削除された
* 対象状態が変更された

可能性がある。

そのため、必要に応じて最新状態を再確認する。

---

### 24.16 利用者境界を取得条件へ含める

以下のように、

```php
Objective::find(
    $objectiveId,
);
```

で取得した後に`user_id`を確認する方式を基本としない。

取得時点から、

```text
objectiveId
+
userId
+
deleted_at IS NULL
```

を条件へ含める。

---

### 24.17 他利用者所属を公開しない

他利用者の目的を指定した場合は、

```text
OBJECTIVE_NOT_FOUND
```

として扱う。

以下のような専用エラーは返却しない。

```text
OBJECTIVE_FORBIDDEN
OTHER_USER_OBJECTIVE
```

対象目的が他利用者に存在することを外部から推測しにくくする。

---

### 24.18 論理削除済みもOBJECTIVE_NOT_FOUNDへ集約する

論理削除済み目的についても、

```text
OBJECTIVE_NOT_FOUND
```

として扱う。

クライアントから見て、

```text
不存在
他利用者所属
論理削除済み
```

を区別しない。

---

### 24.19 無効化済みだけは区別する

一方、同一利用者に属する通常参照可能な目的で、

```text
enabled = false
```

の場合は、

```text
OBJECTIVE_DISABLED
```

として区別する。

これは、対象目的自体は存在しており、現在の業務状態によって操作できないためである。

---

### 24.20 明示的トランザクションを必須にしない

OBJ-005で業務上更新するのは、`objectives`の1レコードのみである。

そのため、Phase1では

```php
DB::transaction()
```

を必須としない。

単一UPDATEをDBの原子性に任せる。

---

### 24.21 将来複数更新になった場合は再検討する

将来的に、

```text
目的無効化
+
監査履歴INSERT
+
別テーブル更新
```

などを同期的に行う場合は、UseCase全体をトランザクション境界とする。

現在の設計を固定的なものとしない。

---

### 24.22 Repositoryは状態変更に限定する

ObjectiveRepositoryでは、

```text
disable
```

という状態変更を担当する。

以下をRepositoryへ詰め込みすぎない。

* HTTPレスポンス生成
* エラー文言生成
* React向け項目変換
* 利用者向け表示判断

Repositoryは、DB状態変更へ責務を限定する。

---

### 24.23 QueryとRepositoryを分離する

概念的には、

```text
ObjectiveQuery
    → 読み取り

ObjectiveRepository
    → 書き込み
```

とする。

OBJ-005では、

```text
対象取得
    → Query

無効化
    → Repository
```

として責務を分離する。

---

### 24.24 Actionを薄く保つ

Actionでは、

```text
objectiveId
+
UserContext
    ↓
UseCase
    ↓
Responder
```

の橋渡しだけを行う。

以下をActionへ直接書かない。

* DB検索
* enabled判定
* UPDATE
* 更新件数判定
* 利用者境界判定

---

### 24.25 FormRequestを無理に作らない

OBJ-005では、Request Bodyを使用しない。

そのため、形式的な統一だけを目的に

```text
DisableObjectiveRequest
```

という空FormRequestを作らない。

クラス数を増やすこと自体を設計品質としない。

---

### 24.26 objectiveId検証は共通化する

`objectiveId`のようなパスID形式検証は、OBJ-005だけで独自実装しない。

API共通方針または共通ルート制約に従う。

例えば、

```text
正の整数形式
```

というルールを目的API全体で統一する。

---

### 24.27 Result DTOを使用する

UseCaseからEloquent Modelを直接HTTP層へ返さない。

概念的には、

```text
Eloquent Model
    ↓
UseCase
    ↓
Result DTO
    ↓
API Resource
```

とする。

これにより、アプリケーション層とDB実装詳細の依存を整理する。

---

### 24.28 API Resourceを使用する

DBカラムの

```text
planned_year_month
required_expense
```

を、APIでは

```text
plannedYearMonth
requiredExpense
```

として返却する。

この変換をResourceへ集約する。

---

### 24.29 無効化後の目的情報を返す

OBJ-005成功時は、

```text
enabled = false
```

となった目的情報を返却する。

React側は、更新後状態を確認できる。

一方、無効化処理そのものの結果だけを

```json
{
  "success": true
}
```

と返す形式にはしない。

---

### 24.30 204 No Contentを採用しない

OBJ-005では、更新後の目的情報を返却するため、

```text
204 No Content
```

ではなく、

```text
200 OK
```

を使用する。

目的API全体のレスポンス形式との一貫性も考慮する。

---

### 24.31 ResponseへDB内部情報を出さない

以下をOBJ-005レスポンスへ不要に含めない。

* `user_id`
* `deleted_at`
* DB制約情報
* SQL情報
* Eloquent内部情報

API利用者に必要な業務データだけを返却する。

---

### 24.32 Reactでは専用Mutationとする

React側では、

```text
OBJ-004
    → useUpdateObjective

OBJ-005
    → useDisableObjective
```

と分離する。

目的編集Mutationに

```text
enabled = false
```

を混在させない。

---

### 24.33 無効化操作は確認UIを設けてよい

目的無効化は、今後の目的達成判定から対象を外す操作である。

そのため、誤操作防止として確認ダイアログを設けてよい。

ただし、確認UIの存在をバックエンドの安全性保証にはしない。

---

### 24.34 Mutation実行中は二重操作を防ぐ

Reactでは、Mutation実行中に無効化ボタンを非活性化してよい。

ただし、二重送信防止の最終保証はバックエンド側の条件付きUPDATEで行う。

---

### 24.35 Mutationを自動Retryしない

OBJ-005は更新APIであるため、通信失敗時に同じPATCHを無条件で自動Retryしない。

通信結果不明の場合は、OBJ-003を再取得して

```text
enabled
```

を確認する。

---

### 24.36 成功後は関連Queryを再取得する

OBJ-005成功後は、少なくとも

```text
OBJ-001
目的一覧

OBJ-003
目的詳細
```

のキャッシュを最新化する。

有効目的を使用する目的達成判定画面がある場合は、必要に応じて関連Queryもinvalidateする。

---

### 24.37 過去判定履歴Cacheを削除しない

目的が無効化されても、保存済み判定履歴は有効な過去情報である。

そのため、OBJ-005成功だけを理由として

```text
assessmentHistories
```

のCacheを削除または改変しない。

---

### 24.38 再有効化UIを作らない

Phase1では、無効化済み目的を再び有効化するAPIを提供しない。

そのため、React側にも

```text
再有効化
```

ボタンを設けない。

APIに存在しない業務操作をフロントエンドだけで表現しない。

---

### 24.39 無効化と達成済みを同一視しない

目的を無効化する理由として、

```text
目的を達成した
```

場合もあり得るが、

```text
enabled = false
```

自体が

```text
達成済み
```

を意味するわけではない。

無効化は、

```text
今後の利用対象から外す
```

という状態である。

達成状態を将来的に明示管理する場合は、別の業務概念として設計する。

---

### 24.40 enabledに複数の意味を持たせすぎない

`enabled = false`へ、

* 達成済み
* 中止
* 期限切れ
* 削除
* 一時停止

などの詳細理由まで持たせない。

Phase1では単純に、

```text
現在利用する目的かどうか
```

だけを表す。

無効化理由が必要になった場合は、別カラムや履歴設計を検討する。

---

### 24.41 無効化日時を別カラムで持たない

Phase1では、

```text
disabled_at
```

のような専用カラムを追加しない。

無効化状態は

```text
enabled = false
```

で管理する。

無効化日時が業務要件として必要になった場合は、将来設計として検討する。

---

### 24.42 updated_atを無効化日時として厳密利用しない

OBJ-005実行時には`updated_at`が更新されるが、

```text
updated_at
    =
無効化日時
```

という業務意味を持たせない。

将来、別の更新が可能になると意味が変わるためである。

無効化日時が必要なら、専用項目を検討する。

---

### 24.43 無効化理由をPhase1では保持しない

Phase1では、

```text
なぜ無効化したか
```

をDBへ保存しない。

例えば、

* 達成
* 中止
* 予定変更

などをOBJ-005 Requestへ含めない。

必要性が明確になった場合に別途設計する。

---

### 24.44 目的の状態モデルを複雑化しすぎない

Phase1では、目的の利用状態は基本的に

```text
enabled = true
enabled = false
```

の2状態とする。

以下のような複雑な状態Enumは現時点では導入しない。

```text
DRAFT
ACTIVE
ACHIEVED
CANCELLED
ARCHIVED
DELETED
```

機能要件が必要とする範囲に設計を限定する。

---

### 24.45 enabledとdeleted_atの組み合わせを明確にする

Phase1で通常想定する状態は、概念的に以下とする。

```text
利用中

enabled = true
deleted_at = NULL
```

```text
無効化済み

enabled = false
deleted_at = NULL
```

論理削除状態については、

```text
deleted_at IS NOT NULL
```

となり、`enabled`の値に関係なく通常APIの対象外とする。

---

### 24.46 論理削除済みのenabledを変更しない

論理削除済み目的に対してOBJ-005を実行しても、

```text
enabled
```

を変更しない。

論理削除済みデータへ追加の業務状態変更を行わない。

---

### 24.47 無効化済み目的をOBJ-004で更新できるかはOBJ-004を正とする

OBJ-005では、目的を無効化することだけを定義する。

無効化後の目的について、

```text
name
plannedYearMonth
requiredExpense
memo
```

をOBJ-004で変更できるかどうかは、OBJ-004の仕様を正とする。

OBJ-005側で勝手に制約を追加しない。

---

### 24.48 OBJ-001・OBJ-003との整合性を維持する

OBJ-005成功後の

```text
enabled = false
```

を、OBJ-001とOBJ-003がどのように扱うかを仕様上統一する。

特に、

```text
OBJ-001
無効化済みを一覧へ含めるか

OBJ-003
無効化済みを詳細取得できるか
```

との整合性を確認する。

---

### 24.49 目的達成判定APIとの整合性を維持する

目的達成判定では、

```text
enabled = true
```

の目的のみを新規判定対象とする。

OBJ-005側で判定処理を持つのではなく、判定API側でもこの条件を明示的に保証する。

---

### 24.50 Phase1ではイベント駆動にしない

OBJ-005成功時に、

```text
ObjectiveDisabled
```

のようなドメインイベントを必須とはしない。

目的無効化に伴う同期・非同期の関連処理が増えた段階で検討する。

---

### 24.51 Phase1では監査テーブルを追加しない

OBJ-005専用の

```text
objective_disable_histories
```

のような監査テーブルは作成しない。

現在の要件では、

```text
objectives.enabled
+
通常ログ
```

で十分とする。

無効化履歴そのものが業務要件になった場合に改めて設計する。

---

### 24.52 Phase1では一括無効化を行わない

OBJ-005は、1回のRequestで1目的だけを無効化する。

以下のような一括APIは提供しない。

```json
{
  "objectiveIds": [
    "1",
    "2",
    "3"
  ]
}
```

単一リソース操作として責務を単純に保つ。

---

### 24.53 Phase1では復元APIを設けない

無効化済み目的を

```text
enabled = true
```

へ戻すAPIは、Phase1では提供しない。

復元が必要になった場合は、

```text
OBJ-006 目的再有効化
```

のような別ユースケースとして設計することを検討する。

---

### 24.54 Phase1で扱わないもの

OBJ-005では、以下を対象外とする。

* 目的通常属性更新
* 目的論理削除
* 目的物理削除
* 目的再有効化
* 無効化理由保存
* 無効化日時専用管理
* 過去判定履歴更新
* 過去判定再計算
* 一括無効化
* Idempotency-Key
* `lockForUpdate()`
* versionカラムによる楽観ロック
* ETag / If-Match
* ドメインイベント
* 専用監査テーブル

Phase1では、

```text
操作対象利用者の
有効な目的1件を
安全に無効化する
```

ことへ責務を限定する。

---

## 25. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)