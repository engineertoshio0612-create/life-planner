# ACC-004 資産口座更新

## 1. 概要

操作対象となる利用者に帰属する指定された資産口座の基本情報を更新する。

ACC-004では、資産口座の通常属性を更新する。

主な更新対象は、以下とする。

- 資産口座名
- 資産種別

一方、以下はACC-004では更新しない。

- 利用者
- 残高記録単位
- 利用開始年月
- 利用可能資産区分
- 利用状態

利用可能資産区分の変更は、履歴管理を伴うためACC-006 利用可能資産設定登録APIで扱う。

資産口座の無効化は、通常属性更新とは異なる業務操作としてACC-005 資産口座無効化APIで扱う。

概念的には、以下の責務分離とする。

```text
ACC-004
    ↓
資産口座の通常属性更新
    ├─ name
    └─ asset_type

ACC-005
    ↓
資産口座無効化

ACC-006
    ↓
利用可能資産区分変更
```

本APIでは、`asset_accounts`のみを更新し、`asset_account_available_settings`は更新しない。

---

## 2. ユースケース

利用者は、登録済み資産口座の基本情報に変更があった場合にACC-004を使用する。

例えば、以下の場合に使用する。

- 資産口座名を変更したい
- 登録時に誤った資産種別を設定したため修正したい

概念的な画面利用は、以下とする。

```text
ACC-001
資産口座一覧取得
    ↓
資産口座選択
    ↓
ACC-003
資産口座詳細取得
    ↓
編集画面
    ↓
利用者が編集
    ↓
ACC-004
資産口座更新
```

ACC-004成功後は、ACC-001またはACC-003を再取得することで、最新状態を確認できる。

---

### 2.1 更新対象としない操作

以下の操作にはACC-004を使用しない。

```text
残高記録単位変更
    → ACC-004では扱わない

利用開始年月変更
    → ACC-004では扱わない

利用可能資産区分変更
    → ACC-006

資産口座無効化
    → ACC-005
```

異なる業務上の意味を持つ操作を1つの更新APIへ集約しない。

---

## 3. エンドポイント

```http
PATCH /api/v1/asset-accounts/{assetAccountId}
```

`assetAccountId`には、更新対象となる資産口座IDを指定する。

例：

```http
PATCH /api/v1/asset-accounts/10
```

---

## 4. HTTPメソッド

```text
PATCH
```

資産口座リソース全体を置き換えるのではなく、更新可能な一部項目だけを変更するため、`PATCH`を使用する。

概念的には、

```text
asset_accounts
    ↓
name
asset_type
    ↓
部分更新
```

とする。

以下のようなリソース全体置換を前提としない。

```http
PUT /api/v1/asset-accounts/{assetAccountId}
```

---

### 4.1 ACC-005との分離

資産口座無効化は、ACC-004のリクエストボディへ

```json
{
  "isEnabled": false
}
```

のような値を指定して実行しない。

通常属性更新とライフサイクル変更を分離し、

```text
ACC-004
PATCH /api/v1/asset-accounts/{assetAccountId}

ACC-005
資産口座無効化用エンドポイント
```

として扱う。

ACC-005の具体的なURLは、ACC-005詳細設計に従う。

---

### 4.2 更新による副作用

ACC-004成功時は、対象となる`asset_accounts`1件を更新する。

通常のLaravel・Eloquent更新に従い、`updated_at`も更新される。

一方、以下の関連データはACC-004では更新しない。

- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `assessment_histories`

---

## 5. 利用者コンテキスト

本APIは利用者依存APIのため、`X-User-Id`を必須とする。

操作対象となる利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者コンテキストの特定、`X-User-Id`の検証および利用者境界については、[API共通方針](../api-common-policy.md)に従う。

指定された`assetAccountId`に対応する資産口座が操作対象利用者に帰属する場合のみ、更新できる。

概念的には、以下の条件を満たす必要がある。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 5.1 利用者境界

ACC-004では、`assetAccountId`だけを条件として更新対象を取得してはならない。

以下のような取得は基本としない。

```text
asset_accounts.id
    = assetAccountId
```

操作対象利用者IDも含めて、

```text
assetAccountId
+
操作対象利用者ID
+
利用中状態
```

を条件として更新対象を取得する。

これにより、他利用者の資産口座を誤って更新することを防止する。

---

### 5.2 他利用者の資産口座

指定された`assetAccountId`が他利用者に帰属する場合は、対象が存在しないものとして扱う。

概念的には、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

とする。

以下のような他利用者専用エラーは返却しない。

```text
FORBIDDEN
OTHER_USER_ASSET_ACCOUNT
```

これにより、他利用者の資産口座が存在するかどうかを外部から推測しにくくする。

---

### 5.3 論理削除済み資産口座

ACC-004では、利用中の資産口座のみを更新対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は更新対象に含めない。

論理削除済み資産口座を指定した場合も、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

再有効化はACC-004の責務に含めない。

---

### 5.4 userIdをリクエストボディから受け付けない

ACC-004では、

```text
userId
user_id
```

を更新項目として受け付けない。

資産口座の所有利用者を変更する機能は提供しない。

例えば、

```json
{
  "userId": "2"
}
```

のような指定によって資産口座を別利用者へ移動できない。

利用者境界は、

```text
X-User-Id
    ↓
UserContext
    ↓
更新対象取得条件
```

によって保証する。

---

### 5.5 利用可能資産設定の利用者境界

ACC-004では、`asset_account_available_settings`を更新しない。

利用可能資産区分を変更する場合は、ACC-006で操作対象利用者に帰属する資産口座であることを確認したうえで、設定履歴を更新する。

ACC-004から利用可能資産設定へ暗黙的な変更を加えない。

---

### 5.6 保有商品の利用者境界

対象資産口座に`holding_assets`が存在していても、ACC-004では保有商品を更新しない。

保有商品の更新は、各HLD APIの利用者境界に従って行う。

資産口座情報の更新を理由として、配下の保有商品を一括更新しない。

---

### 5.7 X-User-Idが不正な場合

以下の場合は、資産口座検索および更新処理へ進まない。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正
- 指定された利用者が存在しない
- 指定された利用者が論理削除済み

この場合、`asset_accounts`を更新しない。

利用者コンテキストに関するエラーコードおよびHTTPステータスは、API共通方針に従う。

---

### 5.8 1リクエスト1利用者

1回のACC-004リクエストでは、`X-User-Id`で指定された1利用者の資産口座だけを扱う。

URLやクエリパラメータから別利用者を指定する方式は採用しない。

例えば、

```text
/api/v1/users/{userId}/asset-accounts/{assetAccountId}
```

や、

```text
?userId=1
```

は使用しない。

利用者の指定経路は、`X-User-Id`へ統一する。

---

## 6. パスパラメータ

本APIでは、更新対象となる資産口座を指定するために以下のパスパラメータを使用する。

| パラメータ | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `assetAccountId` | string | ○ | 更新対象となる資産口座ID |

エンドポイントは、以下とする。

```http
PATCH /api/v1/asset-accounts/{assetAccountId}
```

---

### 6.1 assetAccountId

`assetAccountId`には、更新対象となる資産口座IDを指定する。

例：

```http
PATCH /api/v1/asset-accounts/10
```

APIでは主キーを文字列として扱うが、値としては正の整数形式を前提とする。

正常例：

```text
1
10
123
```

不正例：

```text
0
-1
abc
1.5
```

---

### 6.2 利用者境界

`assetAccountId`だけでは更新対象を確定しない。

実際の更新対象取得では、操作対象利用者IDと組み合わせて検索する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

他利用者に属する資産口座IDが指定された場合は、対象が存在しないものとして扱う。

---

## 7. クエリパラメータ

なし。

ACC-004では、更新対象は`assetAccountId`で指定し、更新内容はリクエストボディで指定する。

以下のような更新動作を切り替えるためのクエリパラメータは使用しない。

- `userId`
- `isEnabled`
- `force`
- `includeDeleted`
- `targetYearMonth`

---

## 8. リクエストヘッダー

以下のリクエストヘッダーを使用する。

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Content-Type` | ○ | `application/json` |
| `Accept` | ○ | `application/json` |

リクエスト例：

```http
PATCH /api/v1/asset-accounts/10
Content-Type: application/json
Accept: application/json
X-User-Id: 1
```

---

### 8.1 X-User-Id

`X-User-Id`は、API共通方針に従って検証する。

本APIの処理開始前に、共通Middlewareで以下を確認する。

- ヘッダーが指定されていること
- IDが共通方針で定めた形式であること
- 利用者が存在すること
- 利用者が論理削除されていないこと

検証済み利用者IDを利用者コンテキストへ保持し、Action以降で使用する。

---

## 9. リクエストボディ

ACC-004では、更新可能な資産口座の通常属性を指定する。

更新可能項目は、以下とする。

- `name`
- `assetType`

リクエスト例：

```json
{
  "name": "メイン証券口座",
  "assetType": "SECURITIES"
}
```

`PATCH`であるため、更新したい項目だけを指定できる。

例えば、資産口座名だけを変更する場合は、以下とする。

```json
{
  "name": "メイン証券口座"
}
```

資産種別だけを変更する場合は、以下とする。

```json
{
  "assetType": "SECURITIES"
}
```

---

### 9.1 リクエスト項目

| 項目 | 型 | 必須 | NULL | 説明 |
|---|---|:---:|:---:|---|
| `name` | string | △ | × | 資産口座名 |
| `assetType` | string | △ | × | 資産種別 |

`△`は、項目単体では任意だが、少なくとも1項目の指定を必須とすることを表す。

---

### 9.2 name

資産口座名を指定する。

例：

```json
{
  "name": "メイン証券口座"
}
```

同一利用者内では、他の資産口座と同じ名前へ変更できない。

ただし、更新対象自身が現在使用している名前と同一値を指定した場合は、重複とは扱わない。

概念的には、

```text
同一user_id
+
同一name
+
id != 更新対象assetAccountId
```

に該当する資産口座が存在する場合に重複とする。

論理削除済み資産口座も同名重複判定の対象とする。

---

### 9.3 assetType

資産種別を指定する。

指定可能な値は、ACC-002と同じく以下とする。

| 値 | 意味 |
|---|---|
| `CASH` | 現金 |
| `BANK` | 銀行 |
| `SECURITIES` | 証券 |
| `IDECO` | iDeCo |
| `CORPORATE_DC` | 企業型DC |
| `OTHER` | その他 |

リクエストでは、DB内部の数値コードを直接指定しない。

---

### 9.4 PATCHとしての部分更新

ACC-004では、指定された項目だけを更新する。

例えば、

```json
{
  "name": "メイン証券口座"
}
```

の場合は、

```text
name
```

だけを更新し、

```text
asset_type
```

は変更しない。

同様に、

```json
{
  "assetType": "SECURITIES"
}
```

の場合は、

```text
asset_type
```

だけを更新し、

```text
name
```

は変更しない。

未指定項目を

```text
NULL
空文字
デフォルト値
```

へ置き換えない。

---

### 9.5 空リクエスト

以下のような更新項目を1件も含まないリクエストは受け付けない。

```json
{}
```

ACC-004は更新APIであるため、少なくとも

```text
name
assetType
```

のいずれか1項目を指定することを必須とする。

---

### 9.6 同一値の指定

現在の値と同じ値が指定された場合は、入力自体は有効として扱ってよい。

例えば、現在の資産口座名が

```text
証券口座
```

の状態で、

```json
{
  "name": "証券口座"
}
```

を送信しても、バリデーションエラーとはしない。

実際にUPDATEを発行するかどうかは、Laravel実装方針で定義する。

---

### 9.7 リクエストで受け付けない項目

以下の項目は、ACC-004では受け付けない。

- `id`
- `userId`
- `user_id`
- `balanceRecordingUnit`
- `balance_recording_unit`
- `startYearMonth`
- `start_year_month`
- `isAvailable`
- `is_available`
- `isEnabled`
- `deletedAt`
- `deleted_at`
- `createdAt`
- `created_at`
- `updatedAt`
- `updated_at`

これらは、ACC-004の更新責務に含めない。

---

### 9.8 balanceRecordingUnitを受け付けない

残高記録単位は、資産口座登録後に変更すると、既存の月末資産データや保有商品管理方法に影響する。

そのため、Phase1ではACC-004の更新対象としない。

以下のようなリクエストは受け付けない。

```json
{
  "balanceRecordingUnit": "ACCOUNT"
}
```

---

### 9.9 startYearMonthを受け付けない

利用開始年月を後から変更すると、既存の

- 利用可能資産設定履歴
- 月末資産データ
- 対象年月判定

との整合性へ影響する。

そのため、ACC-004では`startYearMonth`を更新しない。

---

### 9.10 isAvailableを受け付けない

利用可能資産区分は、履歴として管理する。

そのため、

```json
{
  "isAvailable": false
}
```

のような指定によってACC-004から変更しない。

利用可能資産区分の変更は、ACC-006で扱う。

---

### 9.11 isEnabledを受け付けない

資産口座の無効化は、ACC-005の責務とする。

以下のような状態変更はACC-004で扱わない。

```json
{
  "isEnabled": false
}
```

---

## 10. バリデーション

ACC-004では、主に以下を検証する。

```text
X-User-Id
assetAccountId
name
assetType
更新項目件数
```

資産口座存在確認や利用者境界確認は、更新対象取得時に行う。

---

### 10.1 X-User-Id

`X-User-Id`について、以下を検証する。

- 必須であること
- 共通ID形式に一致すること
- 正の整数として扱えること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

不正な場合は、資産口座検索や更新処理へ進まない。

---

### 10.2 assetAccountId

`assetAccountId`は、正の整数形式であることを必須とする。

正常例：

```text
1
10
999
```

不正例：

```text
0
-1
abc
1.5
1e3
10abc
```

形式不正の場合は、更新対象検索へ進まない。

---

### 10.3 更新項目が1件以上指定されていること

ACC-004では、

```text
name
assetType
```

のうち、少なくとも1項目が指定されていることを必須とする。

以下は不正とする。

```json
{}
```

概念的には、

```text
name未指定
AND
assetType未指定
    ↓
VALIDATION_ERROR
```

とする。

---

### 10.4 name

`name`が指定された場合は、以下を検証する。

- 文字列であること
- `NULL`ではないこと
- 空文字ではないこと
- テーブル定義で定めた最大文字数以内であること

概念的には、FormRequestで

```text
sometimes
required
string
max
```

相当の検証を行う。

---

### 10.5 nameの重複判定

`name`が指定された場合は、同一利用者内に同名の別資産口座が存在しないことを確認する。

概念的な重複条件は、以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = request.name

AND

asset_accounts.id
    != assetAccountId
```

論理削除済み資産口座も重複判定対象とする。

---

### 10.6 自分自身の現在名

更新対象自身の現在の`name`と同じ値を指定した場合は、重複エラーとしない。

例えば、

```text
更新対象
id = 10
name = 証券口座
```

の状態で、

```json
{
  "name": "証券口座"
}
```

を送信した場合は正常な入力とする。

---

### 10.7 論理削除済み同名資産口座

同一利用者に、

```text
name = 証券口座
deleted_at IS NOT NULL
```

の別資産口座が存在する場合も、同名への変更を許可しない。

ACC-002と同じ名前一意性ルールを適用する。

---

### 10.8 他利用者の同名資産口座

他利用者にのみ同名資産口座が存在する場合は、重複エラーとしない。

例えば、

```text
User A
    証券口座

User B
    銀行口座
```

の状態で、User Bの資産口座名を

```text
証券口座
```

へ変更することは許可する。

---

### 10.9 assetType

`assetType`が指定された場合は、以下を検証する。

- 文字列であること
- `NULL`ではないこと
- 定義済みの列挙値であること

指定可能な値は、以下とする。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

未定義値は受け付けない。

例：

```json
{
  "assetType": "CRYPTO"
}
```

は不正とする。

---

### 10.10 未指定項目は検証対象外とする

PATCHであるため、指定されていない更新可能項目について必須チェックを行わない。

例えば、

```json
{
  "name": "メイン証券口座"
}
```

の場合に、

```text
assetType未指定
```

をエラーとしない。

---

### 10.11 NULL

ACC-004の更新可能項目は、`NULL`を許可しない。

以下は不正とする。

```json
{
  "name": null
}
```

```json
{
  "assetType": null
}
```

「項目を変更しない」場合は、`NULL`を送信するのではなく、その項目自体をリクエストから省略する。

---

### 10.12 空文字

`name`に空文字を指定することはできない。

例えば、

```json
{
  "name": ""
}
```

は不正とする。

必要に応じて、空白のみの文字列についてもAPI共通の文字列入力方針に従って不正とする。

---

### 10.13 更新対象資産口座の存在確認

形式検証済みの`assetAccountId`について、操作対象利用者に属する有効な資産口座が存在することを確認する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

該当しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 10.14 他利用者の資産口座

指定された`assetAccountId`が他利用者に属する場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

他利用者に属することを専用エラーで公開しない。

---

### 10.15 論理削除済み資産口座

対象資産口座が

```text
deleted_at IS NOT NULL
```

の場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

ACC-004から再有効化しない。

---

### 10.16 更新対象外項目

未定義項目を拒否するAPI共通方針を採用する場合は、以下のような項目が含まれているリクエストをバリデーションエラーとする。

```json
{
  "name": "証券口座",
  "isAvailable": false
}
```

あるいは、

```json
{
  "balanceRecordingUnit": "ACCOUNT"
}
```

これにより、クライアントがACC-004で変更できない項目を誤って送信しても黙って無視しない。

---

### 10.17 バリデーション失敗時

入力値検証に失敗した場合は、更新処理へ進まない。

特に、

```text
asset_accounts
```

の内容を変更しない。

複数の項目エラーが存在する場合は、API共通方針に従って`error.details`へ複数件返却してよい。

---

## 11. 業務ルール

ACC-004では、操作対象利用者に帰属する利用中の資産口座について、通常属性のみを更新する。

更新対象は、以下とする。

```text
name
asset_type
```

以下の項目はACC-004では更新しない。

```text
user_id
balance_recording_unit
start_year_month
deleted_at
```

また、利用可能資産区分を管理する

```text
asset_account_available_settings
```

も更新しない。

---

### 11.1 操作対象利用者に帰属する資産口座のみ更新できる

更新対象となる資産口座は、以下の条件をすべて満たすものとする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

`assetAccountId`だけを条件として資産口座を取得してはならない。

---

### 11.2 他利用者の資産口座は更新できない

指定された`assetAccountId`が他利用者に帰属する場合は、対象が存在しないものとして扱う。

概念的には、

```text
X-User-Id = 1

assetAccountId = 10

asset_accounts.id = 10
asset_accounts.user_id = 2
    ↓
更新対象外
    ↓
ASSET_ACCOUNT_NOT_FOUND
```

とする。

他利用者に属することを示す専用エラーは返却しない。

---

### 11.3 論理削除済み資産口座は更新できない

ACC-004では、利用中の資産口座のみを更新対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は更新対象に含めない。

論理削除済み資産口座を再有効化する処理も行わない。

---

### 11.4 更新可能項目

ACC-004で更新可能な項目は、以下とする。

```text
name
asset_type
```

APIでは、それぞれ

```text
name
assetType
```

として受け付ける。

---

### 11.5 nameのみの更新

`name`だけが指定された場合は、資産口座名だけを更新する。

例えば、

```json
{
  "name": "メイン証券口座"
}
```

の場合は、

```text
asset_accounts.name
```

のみを変更する。

```text
asset_accounts.asset_type
```

は変更しない。

---

### 11.6 assetTypeのみの更新

`assetType`だけが指定された場合は、資産種別だけを更新する。

例えば、

```json
{
  "assetType": "SECURITIES"
}
```

の場合は、

```text
asset_accounts.asset_type
```

のみを変更する。

```text
asset_accounts.name
```

は変更しない。

---

### 11.7 複数項目の更新

`name`と`assetType`の両方が指定された場合は、両方を同一リクエストで更新する。

例えば、

```json
{
  "name": "メイン証券口座",
  "assetType": "SECURITIES"
}
```

の場合は、

```text
name
asset_type
```

を更新する。

---

### 11.8 未指定項目は変更しない

PATCHであるため、リクエストに含まれない項目は現在値を保持する。

例えば、

```json
{
  "name": "メイン証券口座"
}
```

の場合に、

```text
asset_type
```

を

```text
NULL
デフォルト値
```

へ変更しない。

---

### 11.9 同一値の指定

現在値と同じ値が指定された場合も、入力自体は正常とする。

例えば、

```text
現在
name = 証券口座
```

の状態で、

```json
{
  "name": "証券口座"
}
```

を送信しても、業務エラーとはしない。

実際にDB UPDATEが発生するかは、EloquentのDirty判定に従ってよい。

---

### 11.10 同一利用者内で資産口座名を重複させない

`name`を変更する場合は、同一利用者内に同名の別資産口座が存在しないことを必須とする。

概念的には、

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = request.name

AND

asset_accounts.id
    != assetAccountId
```

に該当する資産口座が存在する場合は、更新不可とする。

---

### 11.11 論理削除済み資産口座も名前重複判定対象とする

ACC-002と同様に、論理削除済みの資産口座も資産口座名の重複判定対象とする。

例えば、

```text
id = 20
name = 証券口座
deleted_at IS NOT NULL
```

という別資産口座が存在する場合、利用中の資産口座を

```text
証券口座
```

へ変更することは許可しない。

---

### 11.12 更新対象自身の現在名は重複としない

更新対象自身が現在持っている名前を再度指定した場合は、重複エラーとしない。

概念的には、

```text
id != assetAccountId
```

を重複確認条件に含める。

---

### 11.13 異なる利用者間では同名を許可する

資産口座名の一意性は、システム全体ではなく利用者単位とする。

例えば、

```text
User A
    証券口座

User B
    銀行口座
```

という状態で、User Bの資産口座を

```text
証券口座
```

へ変更することは許可する。

---

### 11.14 assetTypeは定義済み値のみ更新できる

`assetType`には、APIで定義した以下の値のみ指定できる。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

DB内部で`smallint`などを使用していても、クライアントから数値コードを直接指定させない。

---

### 11.15 assetType変更によって他データを自動変更しない

資産種別を変更しても、以下の関連データを自動変更しない。

- 保有商品
- 月末資産残高
- 商品別月末評価額
- 利用可能資産設定
- 月末資産状況
- 目的達成判定履歴

ACC-004では、`asset_accounts.asset_type`だけを更新する。

---

### 11.16 残高記録単位は更新しない

```text
balance_recording_unit
```

は、ACC-004の更新対象外とする。

残高記録単位を変更すると、

```text
ACCOUNT
    ↕
HOLDING
```

に伴って、

- 月末資産残高
- 保有商品
- 商品別月末評価額

などの既存データとの整合性ルールが必要になるためである。

Phase1では、登録後の変更を許可しない。

---

### 11.17 利用開始年月は更新しない

```text
start_year_month
```

も、ACC-004の更新対象外とする。

利用開始年月を変更すると、

- 利用可能資産設定履歴
- 月末資産対象判定
- 過去月の資産記録

との整合性に影響するためである。

---

### 11.18 利用可能資産区分は更新しない

```text
isAvailable
```

は、ACC-004では受け付けない。

利用可能資産区分は、

```text
asset_account_available_settings
```

によって履歴管理する。

そのため、変更はACC-006で行う。

---

### 11.19 利用状態は更新しない

ACC-004では、

```text
isEnabled
deleted_at
```

を変更しない。

資産口座無効化は、ACC-005で扱う。

通常属性更新とライフサイクル変更を同じAPIへ混在させない。

---

### 11.20 user_idは変更しない

資産口座の所有利用者を変更する機能は提供しない。

以下のような変更はACC-004の責務に含めない。

```text
User Aの資産口座
    ↓
User Bへ移管
```

`asset_accounts.user_id`は変更しない。

---

### 11.21 関連データを更新しない

ACC-004では、以下の関連テーブルを更新しない。

```text
asset_account_available_settings
holding_assets
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
assessment_histories
```

資産口座の通常属性変更だけに責務を限定する。

---

## 12. 処理フロー

概念的な処理フローは、以下とする。

```text
リクエスト受付
    ↓
リクエストID生成
    ↓
X-User-Id検証
    ↓
操作対象利用者の特定
    ↓
assetAccountId形式検証
    ↓
リクエストボディ検証
    ↓
操作対象利用者に帰属する
有効な資産口座を取得
    ↓
存在しない
    → ASSET_ACCOUNT_NOT_FOUND
    ↓
name指定あり？
    ├─ Yes
    │    ↓
    │  同名重複確認
    │    ↓
    │  重複あり
    │    → ASSET_ACCOUNT_NAME_ALREADY_EXISTS
    │
    └─ No
    ↓
指定された更新項目のみ反映
    ↓
asset_accounts更新
    ↓
更新結果DTO生成
    ↓
APIレスポンス変換
    ↓
正常レスポンス返却
```

---

### 12.1 利用者コンテキスト確認

共通Middlewareで`X-User-Id`を検証し、操作対象利用者を特定する。

利用者コンテキストが不正な場合は、更新対象検索へ進まない。

---

### 12.2 assetAccountId検証

パスパラメータの

```text
assetAccountId
```

が正の整数形式であることを確認する。

形式不正の場合は、DB検索へ進まない。

---

### 12.3 リクエストボディ検証

以下について検証する。

```text
name
assetType
```

また、少なくとも1項目が指定されていることを確認する。

空リクエストの場合は、更新処理へ進まない。

---

### 12.4 更新対象取得

操作対象利用者IDと`assetAccountId`を条件として、利用中の資産口座を取得する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

---

### 12.5 対象不存在

該当する資産口座が存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下を同じ扱いとする。

- 存在しない資産口座ID
- 他利用者に属する資産口座
- 論理削除済み資産口座

---

### 12.6 資産口座名重複確認

`name`が指定されている場合のみ、資産口座名の重複確認を行う。

概念的には、

```text
user_id
    = 操作対象利用者ID

AND

name
    = request.name

AND

id
    != assetAccountId
```

とする。

論理削除済み資産口座も検索対象に含める。

---

### 12.7 nameが未指定の場合

`name`がリクエストに含まれていない場合は、資産口座名重複確認を行わない。

不要なDB検索を実行しない。

---

### 12.8 更新値生成

リクエストに指定された項目だけを更新対象として生成する。

概念的には、

```text
name指定あり
    → name更新

assetType指定あり
    → asset_type更新
```

とする。

未指定項目は現在値を保持する。

---

### 12.9 資産口座更新

対象となる

```text
asset_accounts
```

1件を更新する。

概念的には、

```sql
UPDATE asset_accounts
SET
    指定された項目
WHERE
    id = assetAccountId
```

となる。

実際には、Eloquent Modelを通して更新する。

---

### 12.10 updated_at

通常のEloquent更新によって変更が発生した場合は、

```text
updated_at
```

も更新される。

リクエストから`updatedAt`を指定させない。

---

### 12.11 関連テーブル非更新

ACC-004成功時も、以下を更新しない。

```text
asset_account_available_settings
holding_assets
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
assessment_histories
```

---

### 12.12 更新結果生成

更新後の資産口座から、APIレスポンスに必要な結果DTOを生成する。

DB Modelをそのままレスポンスへ返却しない。

必要な項目だけをDTOおよびAPI Resourceへ渡す。

---

## 13. トランザクション境界

ACC-004では、Phase1において明示的なDBトランザクションを必須としない。

更新対象が、

```text
asset_accounts
```

の1レコードだけであり、関連テーブルを同時更新しないためである。

---

### 13.1 更新対象

正常時に更新する業務データは、

```text
asset_accounts 1件
```

のみとする。

主な更新対象カラムは、

```text
name
asset_type
updated_at
```

となる。

---

### 13.2 単一UPDATEで完結する

ACC-004の更新処理は、概念的には

```text
対象取得
    ↓
重複確認
    ↓
asset_accounts UPDATE
```

で完結する。

複数テーブルを原子的に更新する必要がないため、

```php
DB::transaction()
```

を必須としない。

---

### 13.3 重複確認とUPDATE

`name`を更新する場合は、

```text
重複確認
    ↓
UPDATE
```

という複数SQLになる可能性がある。

ただし、重複確認だけで名前一意性を保証しない。

データベースの

```text
user_id
+
name
```

に対するUNIQUE制約を最終防衛線として使用する。

そのため、単純な重複確認とUPDATEを1トランザクションへ入れるだけで競合を防止しようとはしない。

---

### 13.4 同時更新による名前競合

例えば、同一利用者の別々の資産口座を同じ名前へ変更する2リクエストが同時実行された場合、

```text
Request A
重複なし

Request B
重複なし

Request A
UPDATE

Request B
UPDATE
```

となる可能性がある。

この場合は、DBのUNIQUE制約によって一方の更新を失敗させる。

具体的な例外変換は、後続の「排他制御」「エラーレスポンス」「Laravel実装方針」で定義する。

---

### 13.5 Repository単位でトランザクションを開始しない

ACC-004では、Repository内で独自に

```php
DB::transaction()
```

を開始しない。

将来的にACC-004の業務要件が拡張され、

```text
asset_accounts更新
+
別テーブル更新
```

を原子的に扱う必要が生じた場合は、UseCaseをトランザクション境界とする。

---

### 13.6 将来的にトランザクションを検討するケース

将来的に、資産口座更新と同時に

- 変更履歴の登録
- 監査用テーブルへの書き込み
- 関連設定の更新

などを行う場合は、明示的なトランザクションを導入する。

Phase1では、通常属性1レコード更新という現在の責務に合わせて不要なトランザクションを追加しない。

---

## 14. 排他制御

ACC-004では、同一資産口座に対する同時更新と、同一利用者内の資産口座名競合を考慮する。

Phase1では、明示的な行ロックや楽観ロックは導入しない。

資産口座名の一意性については、

```text
アプリケーション側の事前重複確認
+
データベースのUNIQUE制約
```

によって保証する。

---

### 14.1 同一資産口座への同時更新

同じ`assetAccountId`に対して、複数のACC-004が同時実行される可能性がある。

例えば、

```text
Request A
nameを更新

Request B
assetTypeを更新
```

のようなケースが考えられる。

Phase1では、以下のような明示的な排他制御は使用しない。

```php
lockForUpdate()
```

また、

```text
version
updated_at比較
ETag
If-Match
```

などによる楽観ロックも導入しない。

---

### 14.2 Last Write Wins

同一項目に対する同時更新が発生した場合は、Phase1では後から完了した更新内容が最終状態となる可能性を許容する。

概念的には、

```text
Request A
name = 証券口座A

Request B
name = 証券口座B

        ↓

最終的に
後から反映された値が残る
```

となる。

ACC-004は高頻度な競合更新を前提としたAPIではないため、Phase1では複雑な競合制御を追加しない。

---

### 14.3 資産口座名の同時競合

同一利用者に属する別々の資産口座を、同じ名前へ変更する複数リクエストが同時実行される可能性がある。

例えば、

```text
資産口座A
name = A

資産口座B
name = B
```

の状態から、

```text
Request A
資産口座A
name = 証券口座

Request B
資産口座B
name = 証券口座
```

を同時実行するケースである。

---

### 14.4 事前重複確認だけでは保証しない

以下のように、両リクエストが事前重複確認を通過する可能性がある。

```text
Request A
重複なし確認

Request B
重複なし確認

Request A
UPDATE

Request B
UPDATE
```

そのため、アプリケーション側の重複確認だけを一意性保証とはしない。

---

### 14.5 UNIQUE制約

同一利用者内の資産口座名については、データベース側でも

```text
user_id
+
name
```

の組み合わせに一意性を保証する。

これにより、並行更新時にも同名資産口座が複数存在する状態を防止する。

---

### 14.6 論理削除済み資産口座も一意性対象とする

ACC-002およびACC-004の業務ルールでは、論理削除済み資産口座も同名重複判定対象とする。

そのため、一意性制約も

```text
deleted_at IS NULL
```

だけを対象とする部分UNIQUE制約にはしない。

利用中・論理削除済みを問わず、

```text
user_id
+
name
```

の重複を防止する。

---

### 14.7 UNIQUE制約違反

同時更新によってUNIQUE制約違反が発生した場合は、PostgreSQLの例外をそのまま返却しない。

業務上の

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

クライアントから見た結果は、事前重複確認で検出した場合と同じものとする。

---

### 14.8 lockForUpdateを使用しない

ACC-004では、Phase1で

```php
lockForUpdate()
```

を使用しない。

行ロックを使用しても、別々の資産口座を同じ名前へ変更する競合については、ロック対象が異なるため名前一意性を保証できない。

資産口座名の競合は、DBのUNIQUE制約で最終的に保証する。

---

### 14.9 利用者行をロックしない

同一利用者内の資産口座更新を直列化するために、

```text
users
```

の行をロックする方式は採用しない。

利用者行をロックすると、同一利用者に関する他の処理まで不要に待機させる可能性があるためである。

---

### 14.10 将来的な競合制御

将来的に、

- 複数端末からの編集
- 複数ユーザーによる共同編集
- 更新競合の検出
- 編集内容の上書き防止

が必要になった場合は、

```text
versionカラム
updated_at比較
ETag
If-Match
```

などによる楽観ロックの導入を検討する。

Phase1では対象外とする。

---

## 15. 成功レスポンス

資産口座更新に成功した場合は、

```text
200 OK
```

を返却する。

正常時は、API共通方針に従った成功レスポンス形式を使用する。

概念例：

```json
{
  "data": {
    "id": "10",
    "name": "メイン証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

ACC-004で直接更新する項目は

```text
name
assetType
```

だけだが、更新後の資産口座状態をクライアントが確認できるよう、資産口座詳細として必要な情報を返却する。

---

### 15.1 レスポンス項目

正常時の`data`配下には、以下を返却する。

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `id` | string | × | 更新した資産口座ID |
| `name` | string | × | 更新後の資産口座名 |
| `assetType` | string | × | 更新後の資産種別 |
| `balanceRecordingUnit` | string | × | 残高記録単位 |
| `isAvailable` | boolean | × | 現在の利用可能資産区分 |
| `startYearMonth` | string | × | 利用開始年月。`YYYY-MM`形式 |
| `isEnabled` | boolean | × | 利用状態。正常時は`true` |

---

### 15.2 id

更新した資産口座IDを文字列として返却する。

例：

```json
{
  "id": "10"
}
```

データベース上で`bigint`として保持していても、API共通方針に従ってstringで返却する。

---

### 15.3 name

更新後の資産口座名を返却する。

`name`がリクエストで未指定だった場合は、更新前から保持している現在値を返却する。

---

### 15.4 assetType

更新後の資産種別をAPI用文字列として返却する。

例えば、

```json
{
  "assetType": "SECURITIES"
}
```

とする。

DB内部の数値コードをそのまま返却しない。

---

### 15.5 balanceRecordingUnit

ACC-004では残高記録単位を更新しない。

ただし、更新後の資産口座状態として現在値を返却する。

想定値は、

```text
ACCOUNT
HOLDING
```

とする。

---

### 15.6 isAvailable

ACC-004では利用可能資産区分を更新しない。

レスポンスでは、現在有効な

```text
asset_account_available_settings.is_available
```

を

```text
isAvailable
```

として返却する。

これにより、ACC-003と同じ資産口座詳細表現を使用できる。

---

### 15.7 startYearMonth

ACC-004では利用開始年月を更新しない。

現在保持している

```text
asset_accounts.start_year_month
```

を

```text
YYYY-MM
```

形式で返却する。

---

### 15.8 isEnabled

ACC-004では、利用中の資産口座だけを更新対象とする。

そのため、正常レスポンスでは

```json
{
  "isEnabled": true
}
```

となる。

`deleted_at`はレスポンスへ返却しない。

---

### 15.9 返却しない情報

ACC-004では、以下の内部情報を返却しない。

- `user_id`
- `deleted_at`
- `created_at`
- `updated_at`
- `asset_account_available_settings.id`
- `asset_account_available_settings.asset_account_id`
- `asset_account_available_settings.start_year_month`
- `asset_account_available_settings.end_year_month`
- DB内部の数値コード

また、以下の関連データも返却しない。

- 保有商品一覧
- 月末資産残高
- 商品別月末評価額
- 月末資産状況
- 資産推移

---

## 16. エラーレスポンス

異常時は、API共通方針に従った共通エラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_NAME_ALREADY_EXISTS",
    "message": "同じ名前の資産口座がすでに存在します。"
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

---

### 16.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

この場合、資産口座検索や更新処理へ進まない。

---

### 16.2 INVALID_USER_ID

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

例えば、以下を不正とする。

```text
0
-1
abc
1.5
```

---

### 16.3 USER_NOT_FOUND

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

この場合、資産口座を更新しない。

---

### 16.4 INVALID_ASSET_ACCOUNT_ID

`assetAccountId`の形式が不正な場合は、

```text
INVALID_ASSET_ACCOUNT_ID
```

を返却する。

例えば、

```text
0
-1
abc
1.5
1e3
10abc
```

などを不正とする。

---

### 16.5 VALIDATION_ERROR

リクエストボディの入力値検証に失敗した場合は、

```text
VALIDATION_ERROR
```

を返却する。

主な対象は、以下とする。

- `name`
- `assetType`
- 更新項目未指定
- 更新対象外項目の指定

---

### 16.6 更新項目未指定

以下のように更新可能項目が1件も指定されていない場合は、

```json
{}
```

`VALIDATION_ERROR`として扱う。

概念的なエラー詳細は、

```text
少なくとも1つの更新項目を指定してください。
```

とする。

---

### 16.7 name不正

`name`が指定されている場合に、以下を不正とする。

- `NULL`
- 文字列以外
- 空文字
- 最大文字数超過
- API共通方針上不正となる空白文字列

これらは、

```text
VALIDATION_ERROR
```

として扱う。

---

### 16.8 assetType不正

`assetType`が定義済みの値ではない場合は、

```text
VALIDATION_ERROR
```

とする。

例えば、

```json
{
  "assetType": "CRYPTO"
}
```

は不正とする。

---

### 16.9 更新対象外項目

ACC-004で更新を許可していない項目が送信された場合、未定義項目を拒否するAPI共通方針に従って`VALIDATION_ERROR`とする。

例えば、

```json
{
  "isAvailable": false
}
```

```json
{
  "balanceRecordingUnit": "ACCOUNT"
}
```

```json
{
  "startYearMonth": "2026-09"
}
```

```json
{
  "isEnabled": false
}
```

などを対象とする。

---

### 16.10 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

- 指定された資産口座が存在しない
- 指定された資産口座が他利用者に属する
- 指定された資産口座が論理削除済み

他利用者に属することや論理削除済みであることを個別のエラーコードで公開しない。

---

### 16.11 他利用者の資産口座

例えば、

```text
User A
    assetAccountId = 10

User B
    X-User-Id = 2
```

の状態で、User Bが

```http
PATCH /api/v1/asset-accounts/10
```

を実行した場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

他利用者の資産口座を更新しない。

---

### 16.12 論理削除済み資産口座

対象資産口座が

```text
deleted_at IS NOT NULL
```

の場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

ACC-004から再有効化しない。

---

### 16.13 ASSET_ACCOUNT_NAME_ALREADY_EXISTS

`name`を更新する際、同一利用者内に同名の別資産口座が存在する場合は、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

を返却する。

論理削除済みの別資産口座も重複判定対象とする。

---

### 16.14 更新対象自身の名前

更新対象自身の現在の名前と同じ値を指定した場合は、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

としない。

例えば、

```text
id = 10
name = 証券口座
```

に対して、

```json
{
  "name": "証券口座"
}
```

を送信することは許可する。

---

### 16.15 他利用者の同名資産口座

他利用者にのみ同名資産口座が存在する場合は、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

としない。

名前の一意性は利用者単位で判定する。

---

### 16.16 同時更新による名前競合

事前重複確認を通過した後、別リクエストが同じ名前を先に確定した結果、DBのUNIQUE制約違反が発生する場合がある。

この場合も、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

PostgreSQLの

- SQLSTATE
- 制約名
- SQL
- テーブル名

などをクライアントへ公開しない。

---

### 16.17 利用可能資産設定の不整合

ACC-004の成功レスポンスで現在の`isAvailable`を返却する設計であるため、更新後に現在有効な

```text
asset_account_available_settings
```

を取得する必要がある。

現在有効な設定が0件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

とする。

現在有効な設定が複数件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

とする。

これらは、サーバー側の業務データ不整合として扱う。

---

### 16.18 データ不整合時に補完しない

利用可能資産設定が取得できない場合に、

```text
isAvailable = false
```

へ補完しない。

また、複数件存在する場合に最新1件を任意採用しない。

データ不整合を正常レスポンスで隠蔽しない。

---

### 16.19 更新後レスポンス生成時のエラー

資産口座自体のUPDATEが成功した後に、レスポンス生成に必要な利用可能資産設定取得でデータ不整合が判明する可能性がある。

Phase1ではACC-004を単一レコード更新として明示的なトランザクションを必須としていないため、このケースでは資産口座の更新自体が完了している可能性がある。

この状態を避けたい場合は、Laravel実装方針において、

```text
更新前に
現在有効な利用可能資産設定を検証する
```

構成とする。

概念的には、

```text
資産口座取得
    ↓
現在設定1件確認
    ↓
name重複確認
    ↓
UPDATE
    ↓
成功レスポンス
```

とし、UPDATE後に初めて設定不整合を検出する構成を避ける。

---

### 16.20 INTERNAL_SERVER_ERROR

資産口座更新処理で想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスへ、以下の内部情報を含めない。

- SQL
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 16.21 主なエラー一覧

ACC-004で想定する主なエラーは、以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`未指定 |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`形式不正 |
| `400 Bad Request` | `INVALID_ASSET_ACCOUNT_ID` | `assetAccountId`形式不正 |
| `400 Bad Request` | `VALIDATION_ERROR` | リクエストボディ不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 利用者不存在、または論理削除済み |
| `404 Not Found` | `ASSET_ACCOUNT_NOT_FOUND` | 資産口座不存在、他利用者所属、または論理削除済み |
| `409 Conflict` | `ASSET_ACCOUNT_NAME_ALREADY_EXISTS` | 同一利用者内の資産口座名重複 |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND` | 現在有効な利用可能資産設定が存在しない |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT` | 現在有効な利用可能資産設定が複数存在する |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバー内部エラー |

具体的なHTTPステータスとエラーコードの最終定義は、API共通方針を優先する。

---

### 16.22 エラー時の更新

以下のエラーをUPDATE前に検出した場合は、`asset_accounts`を更新しない。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

特に、利用可能資産設定の不整合確認をUPDATE前に行うことで、

```text
更新だけ成功
+
レスポンス生成失敗
```

という状態を可能な限り避ける。

---

## 17. HTTPステータス

ACC-004では、処理結果に応じて以下のHTTPステータスを返却する。

| HTTPステータス | 用途 |
|---|---|
| `200 OK` | 資産口座更新成功 |
| `400 Bad Request` | 利用者コンテキスト、`assetAccountId`、またはリクエストボディの不正 |
| `404 Not Found` | 利用者または資産口座が存在しない |
| `409 Conflict` | 同一利用者内の資産口座名重複 |
| `500 Internal Server Error` | 利用可能資産設定の不整合、または想定外のサーバー内部エラー |

具体的な独自エラーコードは、「エラーレスポンス」およびAPI共通方針に従う。

---

### 17.1 200 OK

指定された資産口座の更新に成功した場合は、

```http
200 OK
```

を返却する。

概念的には、

```text
更新対象資産口座取得
    ↓
入力値・業務ルール確認
    ↓
必要に応じて資産口座名重複確認
    ↓
asset_accounts更新
    ↓
200 OK
```

とする。

---

### 17.2 400 Bad Request

以下の場合は、

```http
400 Bad Request
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
```

例えば、

- `X-User-Id`未指定
- `X-User-Id`形式不正
- `assetAccountId`形式不正
- 更新項目未指定
- `name`不正
- `assetType`不正
- 更新対象外項目の指定

などを対象とする。

---

### 17.3 404 Not Found

以下の場合は、

```http
404 Not Found
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_NOT_FOUND
ASSET_ACCOUNT_NOT_FOUND
```

`ASSET_ACCOUNT_NOT_FOUND`には、以下を含む。

- 指定された資産口座が存在しない
- 指定された資産口座が他利用者に属する
- 指定された資産口座が論理削除済み

他利用者所属や論理削除済みであることを個別のHTTPステータスで公開しない。

---

### 17.4 409 Conflict

同一利用者内に、更新後の`name`と同じ名前を持つ別の資産口座が存在する場合は、

```http
409 Conflict
```

を返却する。

エラーコードは、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

とする。

アプリケーション側の事前重複確認で検出した場合だけでなく、並行更新によってDBのUNIQUE制約違反が発生した場合も、同じHTTPステータスおよびエラーコードへ変換する。

---

### 17.5 500 Internal Server Error

以下の場合は、

```http
500 Internal Server Error
```

として扱う。

主なエラーコードは、以下とする。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
INTERNAL_SERVER_ERROR
```

利用可能資産設定の不存在または重複は、クライアント入力ではなくサーバー側の業務データ不整合として扱う。

---

### 17.6 ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND

ACC-004の成功レスポンスで現在の`isAvailable`を返却するため、現在有効な利用可能資産設定が1件存在することを前提とする。

対象設定が存在しない場合は、

```http
500 Internal Server Error
```

とし、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

を返却する。

---

### 17.7 ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT

現在年月において有効な利用可能資産設定が複数件存在する場合は、

```http
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

とする。

任意の1件を採用して正常レスポンスを返却しない。

---

### 17.8 INTERNAL_SERVER_ERROR

想定外のサーバー内部エラーが発生した場合は、

```http
500 Internal Server Error
```

を返却する。

エラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

レスポンスへ内部例外やSQLなどを公開しない。

---

## 18. 冪等性

ACC-004は、同じ更新内容を同じ資産口座へ複数回適用した場合、最終的な業務状態が同一となるため、データ状態としては冪等な更新として扱う。

例えば、

```json
{
  "name": "メイン証券口座"
}
```

を複数回送信した場合、最終的な

```text
asset_accounts.name
```

は、

```text
メイン証券口座
```

となる。

---

### 18.1 同一リクエストの再実行

以下のリクエストを同じ資産口座へ複数回送信した場合を考える。

```json
{
  "name": "メイン証券口座",
  "assetType": "SECURITIES"
}
```

1回目の実行後に、同じ内容で再度実行しても、業務上の最終状態は変化しない。

概念的には、

```text
1回目
name = メイン証券口座
asset_type = SECURITIES

2回目
name = メイン証券口座
asset_type = SECURITIES

    ↓

最終状態は同一
```

となる。

---

### 18.2 PATCHと冪等性

HTTPメソッドとして`PATCH`を使用しているが、PATCH自体が必ず非冪等であることを意味しない。

ACC-004では、

```text
指定された属性を
指定された値へ更新する
```

という処理であり、加算や履歴追加のような実行回数依存の処理を行わない。

そのため、同じ入力を繰り返した場合の最終データ状態は同一となる。

---

### 18.3 updated_at

同一内容の再送時に、Laravel/EloquentのDirty判定によって実際にUPDATEが発行されない場合は、

```text
updated_at
```

も変更されない可能性がある。

一方、実装方法によっては同一値更新でも`updated_at`が変化する可能性がある。

ACC-004の冪等性は、主に業務属性である

```text
name
asset_type
```

の最終状態について評価する。

---

### 18.4 同一値更新をエラーにしない

現在値と同じ値を指定した場合も、以下のような専用エラーにはしない。

```text
NO_CHANGES
ALREADY_UPDATED
```

正常な更新リクエストとして扱う。

---

### 18.5 Idempotency-Key

Phase1では、ACC-004専用の

```text
Idempotency-Key
```

は使用しない。

ACC-004は履歴レコードを追加するAPIではなく、指定資産口座の属性を指定値へ更新するAPIであるため、Phase1では追加の冪等性キー管理を必要としない。

---

### 18.6 同時更新と冪等性は区別する

ACC-004が同一入力に対して同じ最終状態へ収束することと、同時更新時の競合を検出できることは別である。

例えば、

```text
Request A
name = A

Request B
name = B
```

が同時実行された場合、Phase1ではLast Write Winsとなる可能性がある。

これは冪等性の問題ではなく、排他制御・競合制御の問題として扱う。

---

## 19. キャッシュ

Phase1では、ACC-004専用のサーバー側アプリケーションキャッシュを使用しない。

ACC-004成功時は、PostgreSQL上の

```text
asset_accounts
```

が最新状態となる。

---

### 19.1 更新API自体をキャッシュしない

ACC-004は更新APIであるため、

```http
PATCH /api/v1/asset-accounts/{assetAccountId}
```

のレスポンスをHTTPキャッシュによって後続リクエストへ再利用することを前提としない。

---

### 19.2 ACC-001への影響

ACC-004で資産口座名または資産種別を変更すると、ACC-001 資産口座一覧取得のレスポンス内容も変化する。

概念的には、

```text
ACC-004
    ↓
name / assetType更新
    ↓
ACC-001の取得結果が変化
```

となる。

フロントエンドでTanStack Queryなどを使用する場合は、ACC-004成功後に資産口座一覧Queryを無効化する。

---

### 19.3 ACC-003への影響

ACC-004成功後は、対象資産口座のACC-003 資産口座詳細取得結果も変化する。

そのため、対象`assetAccountId`の詳細Query Cacheを無効化する。

概念的には、

```text
ACC-004成功
    ↓
ACC-003 detail cache invalidate
    ↓
必要に応じて再取得
```

とする。

---

### 19.4 利用可能資産設定キャッシュ

ACC-004では、

```text
asset_account_available_settings
```

を更新しない。

そのため、ACC-004単独を理由として利用可能資産設定そのもののサーバー側キャッシュを無効化する必要はない。

ただし、ACC-003相当の詳細レスポンスをフロントエンド側でキャッシュしている場合は、`name`や`assetType`が変わるため詳細キャッシュ全体を無効化する。

---

### 19.5 保有商品キャッシュ

ACC-004では、

```text
holding_assets
```

を更新しない。

そのため、保有商品一覧そのもののキャッシュを必ず無効化する必要はない。

ただし、画面側で資産口座名を保有商品表示に合成している場合は、UI全体のキャッシュ設計に応じて関連Queryの再取得を検討する。

---

### 19.6 月末資産キャッシュ

ACC-004では、

```text
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
```

を更新しない。

そのため、月末資産データそのものはACC-004によって変更されない。

ただし、画面表示上資産口座名や資産種別を月末資産データと組み合わせて表示している場合は、表示用Queryの構成に応じてキャッシュ無効化を検討する。

---

### 19.7 Query Cacheの基本方針

React側では、ACC-004成功後に最低限、

```text
資産口座一覧
+
対象資産口座詳細
```

のQuery Cacheを無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey: assetAccountKeys.all,
});

await queryClient.invalidateQueries({
  queryKey: assetAccountKeys.detail(
    assetAccountId,
  ),
});
```

正式なQuery Keyは、React共通設計に従う。

---

### 19.8 Optimistic Update

Phase1では、ACC-004に対するOptimistic Updateは必須としない。

ACC-004では、

- 利用者境界確認
- 資産口座存在確認
- 資産口座名重複確認
- DBのUNIQUE制約確認

など、サーバー側でしか最終判断できない処理が存在する。

そのため、

```text
API成功
    ↓
Query invalidate
    ↓
再取得
```

という単純な方式を基本とする。

---

### 19.9 サーバー状態を正とする

ACC-004成功後の最終状態は、PostgreSQLへ保存されたデータを正とする。

React側で送信したRequestだけを基準に永続的な表示状態を確定しない。

必要に応じてACC-001またはACC-003を再取得して最新状態へ同期する。

---

### 19.10 HTTPキャッシュ

ACC-004自体はHTTPキャッシュ対象としない。

また、ACC-004成功後にACC-003などのGETレスポンスが古い状態を返し続けないよう、将来的にHTTPキャッシュを導入する場合は、関連リソースの失効方針を合わせて設計する。

Phase1では、ACC系API独自の

```text
ETag
Last-Modified
If-Match
If-None-Match
```

などは導入しない。

---

### 19.11 利用者境界とキャッシュ

フロントエンドで資産口座一覧・詳細をキャッシュする場合は、利用者切替時に前利用者のデータを誤表示しないようにする。

概念的には、

```text
userId
+
assetAccountId
```

または利用者切替時のQuery無効化によって、キャッシュ上でも利用者境界を維持する。

---

### 19.12 ACC-005・ACC-006との関係

ACC-004成功後は、主に

```text
name
assetType
```

が変化する。

一方、

```text
isEnabled
    → ACC-005

isAvailable
    → ACC-006
```

によって変更される。

そのため、各Mutation成功後に対応する資産口座一覧・詳細Queryを無効化する方針を共通化するとよい。

---

## 20. 関連テーブル

ACC-004では、指定された資産口座の通常属性を更新するため、以下のテーブルを使用する。

| テーブル | 用途 | 更新 |
|---|---|:---:|
| `users` | 操作対象利用者の確認 | × |
| `asset_accounts` | 更新対象資産口座の取得、利用者境界確認、通常属性更新、同名重複確認 | ○ |
| `asset_account_available_settings` | 現在の利用可能資産区分の取得 | × |

ACC-004で直接更新する業務テーブルは、

```text
asset_accounts
```

のみとする。

---

### 20.1 users

`X-User-Id`で指定された操作対象利用者の存在確認に使用する。

概念的な条件は、以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者コンテキストの確認は、API共通Middlewareで行う。

ACC-004では、`users`を更新しない。

---

### 20.2 asset_accounts

更新対象となる資産口座の取得、利用者境界確認、資産口座名重複確認、通常属性更新に使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | `assetAccountId`との照合 |
| `user_id` | 操作対象利用者との利用者境界確認 |
| `name` | 資産口座名、同名重複確認、更新 |
| `asset_type` | 資産種別、更新 |
| `balance_recording_unit` | 現在値の返却 |
| `start_year_month` | 現在値の返却 |
| `updated_at` | 更新日時 |
| `deleted_at` | 利用状態の判定 |

---

### 20.3 更新対象取得

更新対象資産口座は、概念的に以下の条件で取得する。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

他利用者に属する資産口座、論理削除済み資産口座は更新対象として取得しない。

---

### 20.4 資産口座名の重複確認

`name`が指定された場合は、同一利用者内に同名の別資産口座が存在しないことを確認する。

概念的な条件は、以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = request.name

AND

asset_accounts.id
    != assetAccountId
```

論理削除済み資産口座も重複判定対象に含める。

Laravel SoftDeletesを使用する場合は、重複確認時に

```php
withTrashed()
```

を使用する。

---

### 20.5 asset_accountsの一意性

同一利用者内の資産口座名については、データベース側でも

```text
user_id
+
name
```

の組み合わせに一意性を保証する。

これにより、並行更新時にアプリケーション側の事前重複確認を複数リクエストが通過しても、同名資産口座が複数存在する状態を防止する。

---

### 20.6 name

リクエストに`name`が指定された場合のみ、

```text
asset_accounts.name
```

を更新する。

未指定の場合は、現在値を保持する。

現在値と同一の名前が指定された場合は、更新対象自身を重複判定から除外する。

---

### 20.7 asset_type

リクエストに`assetType`が指定された場合のみ、

```text
asset_accounts.asset_type
```

を更新する。

APIで受け取った

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

を、DB内部の保存形式へ変換して保存する。

---

### 20.8 balance_recording_unit

```text
asset_accounts.balance_recording_unit
```

は、ACC-004では更新しない。

成功レスポンス生成のために現在値を使用するのみとする。

---

### 20.9 start_year_month

```text
asset_accounts.start_year_month
```

も、ACC-004では更新しない。

成功レスポンスでは現在値を

```text
startYearMonth
```

として返却する。

---

### 20.10 deleted_at

```text
asset_accounts.deleted_at
```

は、更新対象の有効状態確認に使用する。

ACC-004では変更しない。

正常レスポンスでは対象資産口座が利用中であるため、

```text
isEnabled = true
```

として扱う。

---

### 20.11 updated_at

実際に資産口座の属性が更新された場合は、Eloquentの通常動作に従って

```text
updated_at
```

が更新される。

クライアントから`updatedAt`を指定させない。

---

### 20.12 asset_account_available_settings

ACC-004の成功レスポンスで現在の

```text
isAvailable
```

を返却するために参照する。

ACC-004では、このテーブルを更新しない。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `asset_account_id` | 対象資産口座との関連 |
| `start_year_month` | 設定の適用開始年月 |
| `end_year_month` | 設定の適用終了年月 |
| `is_available` | 現在の利用可能資産区分 |

---

### 20.13 現在有効な利用可能資産設定

現在年月を基準として、概念的に以下の条件を満たす利用可能資産設定を取得する。

```text
asset_account_available_settings.asset_account_id
    = assetAccountId

AND

asset_account_available_settings.start_year_month
    <= 現在年月

AND

(
    asset_account_available_settings.end_year_month
        IS NULL

    OR

    asset_account_available_settings.end_year_month
        >= 現在年月
)
```

正常状態では、該当する設定が1件だけ存在することを前提とする。

---

### 20.14 利用可能資産設定を更新しない

ACC-004では、

```text
asset_account_available_settings
```

への

```text
INSERT
UPDATE
DELETE
```

を行わない。

利用可能資産区分の変更は、ACC-006の責務とする。

---

### 20.15 更新しない関連テーブル

ACC-004では、以下のテーブルを更新しない。

- `users`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

資産口座の通常属性変更だけに責務を限定する。

---

## 21. 関連する機能要件

ACC-004は、資産口座の通常属性更新に関する機能要件と対応する。

主な関連要件は、以下とする。

- 資産口座管理
  - 登録済み資産口座を編集できる
  - 資産口座名を変更できる
  - 資産種別を変更できる
- 資産口座名
  - 同一利用者内では重複させない
  - 更新対象自身の現在名は重複としない
  - 異なる利用者間では同名を許可する
  - 論理削除済み資産口座も重複判定対象とする
- 残高記録単位
  - Phase1では登録後に変更しない
- 利用開始年月
  - Phase1では登録後に変更しない
- 利用可能資産区分
  - ACC-004では変更しない
  - 利用可能資産区分の変更はACC-006で扱う
- 利用状態
  - ACC-004では変更しない
  - 資産口座無効化はACC-005で扱う
- 利用者境界
  - 操作対象利用者に帰属する資産口座だけを更新できる
  - 他利用者の資産口座は対象不存在として扱う

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

## 22. インデックス

ACC-004では、主に以下の条件を使用する。

```text
asset_accounts.id
```

```text
asset_accounts.user_id
```

```text
asset_accounts.name
```

また、成功レスポンス生成時には

```text
asset_account_available_settings.asset_account_id
+
start_year_month
+
end_year_month
```

を使用する。

---

### 22.1 asset_accounts.id

`asset_accounts.id`は主キーであるため、主キーインデックスを使用する。

ACC-004専用として追加インデックスを作成しない。

---

### 22.2 user_id + name

同一利用者内の資産口座名一意性を保証するため、

```text
user_id
+
name
```

のUNIQUE制約を使用する。

この制約は、ACC-002の登録時とACC-004の更新時で共通して利用する。

---

### 22.3 論理削除済み資産口座

論理削除済み資産口座も名前一意性の対象とするため、Phase1では

```text
WHERE deleted_at IS NULL
```

を条件とする部分UNIQUEインデックスを使用しない。

---

### 22.4 asset_account_available_settings

現在有効な設定取得では、

```text
asset_account_id
```

を必須条件として使用する。

必要に応じて、

```text
asset_account_id
+
start_year_month
```

などの複合インデックスを検討する。

ACC-003やACC-006など他APIでも利用する検索条件を踏まえて設計する。

---

### 22.5 不要なインデックスを追加しない

ACC-004だけを理由として、主キーやUNIQUE制約と役割が重複するインデックスを追加しない。

実際のデータ件数や実行計画を確認したうえで追加を判断する。

---

## 23. 性能

ACC-004は、単一の資産口座を更新するAPIであるため、処理負荷は小さい。

概念的には、

```text
資産口座1件取得
    ↓
必要に応じて名前重複確認
    ↓
現在の利用可能資産設定確認
    ↓
資産口座1件更新
```

で完結する。

---

### 23.1 一覧取得を行わない

以下のように、利用者の資産口座一覧を全件取得してから対象IDを探す実装は行わない。

```text
資産口座一覧取得
    ↓
PHP側でassetAccountId検索
```

対象IDと利用者IDを条件として直接1件取得する。

---

### 23.2 name未指定時は重複確認しない

`name`がリクエストに含まれない場合は、名前重複確認Queryを実行しない。

例えば、

```json
{
  "assetType": "BANK"
}
```

の場合は、資産口座名の重複検索を省略する。

---

### 23.3 利用可能資産設定を全履歴取得しない

成功レスポンスに必要なのは、現在有効な

```text
isAvailable
```

だけである。

そのため、利用可能資産設定の全履歴を取得してPHP側で判定しない。

DB側で現在年月条件を指定して取得する。

---

### 23.4 取得件数

通常は、

```text
asset_accounts
    1件

asset_account_available_settings
    1件
```

だけを扱う。

大量データをアプリケーションメモリへ読み込まない。

---

### 23.5 保有商品を取得しない

対象資産口座の`balance_recording_unit`が

```text
HOLDING
```

であっても、

```text
holding_assets
```

をEager Loadしない。

ACC-004では保有商品を更新・返却しないためである。

---

### 23.6 月末資産データを取得しない

以下のデータをACC-004の更新処理のために取得しない。

- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`

通常属性更新に不要な関連データを読み込まない。

---

### 23.7 N+1問題

ACC-004は、単一資産口座を対象とするため、通常の意味でのN+1問題は発生しない。

不要な関連リレーションを読み込まない構成とする。

---

## 24. セキュリティ

ACC-004では、利用者境界と更新可能項目の限定を重要なセキュリティ要件とする。

---

### 24.1 assetAccountIdだけで取得しない

以下のような更新対象取得は避ける。

```php
AssetAccount::find(
    $assetAccountId,
);
```

必ず、

```text
assetAccountId
+
userId
+
deleted_at IS NULL
```

を条件とする。

---

### 24.2 他利用者の資産口座を更新しない

他利用者に属する`assetAccountId`が指定された場合でも、その資産口座を取得・更新しない。

レスポンスは、

```text
ASSET_ACCOUNT_NOT_FOUND
```

とし、他利用者のリソース存在有無を外部へ公開しない。

---

### 24.3 論理削除済み資産口座を更新しない

論理削除済み資産口座を通常更新対象に含めない。

ACC-004から`deleted_at`を`NULL`へ戻して再有効化することもできない。

---

### 24.4 Mass Assignmentを避ける

以下のようにRequest全体をそのままModelへ渡さない。

```php
$assetAccount->update(
    $request->all(),
);
```

更新対象は、

```text
name
asset_type
```

だけに限定する。

---

### 24.5 更新対象外項目を変更させない

以下をクライアント指定によって更新できないようにする。

- `user_id`
- `balance_recording_unit`
- `start_year_month`
- `deleted_at`
- `created_at`
- `updated_at`

また、

```text
isAvailable
```

から利用可能資産設定を変更させない。

---

### 24.6 user_idを変更しない

クライアントが

```json
{
  "userId": "999"
}
```

などを送信しても、資産口座の所有利用者を変更できない。

所有者変更はACC-004の責務に含めない。

---

### 24.7 DB内部コードを受け付けない

資産種別をDB内部の数値コードで指定させない。

APIで定義した

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

だけを受け付ける。

---

### 24.8 SQLインジェクション対策

`assetAccountId`、利用者ID、資産口座名などを検索条件へ使用する場合は、EloquentまたはQuery Builderのバインド機構を使用する。

SQL文字列へ入力値を直接連結しない。

---

### 24.9 内部情報を公開しない

エラー時に、以下の情報をレスポンスへ含めない。

- SQL
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- Laravel内部例外
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

---

## 25. ログ・監視

ACC-004では、API共通ログ方針に従って更新処理の結果を記録する。

---

### 25.1 ログコンテキスト

必要に応じて、以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assetAccountId
httpStatus
errorCode
```

`apiId`は、

```text
ACC-004
```

とする。

---

### 25.2 正常時

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = ACC-004
assetAccountId
httpStatus = 200
```

更新後の資産口座名や資産種別を通常ログへ不要に記録しない。

---

### 25.3 異常時

異常時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId
assetAccountId
errorCode
httpStatus
```

---

### 25.4 資産口座名競合

以下のエラーが発生した場合は、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

必要に応じて競合発生をログへ記録する。

ただし、PostgreSQLの制約名やSQLをレスポンスへ公開しない。

---

### 25.5 利用可能資産設定の不整合

以下の場合は、データ不整合として調査可能なログを記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text
assetAccountId
currentYearMonth
matchedSettingCount
```

を内部ログへ記録してよい。

---

### 25.6 requestId

クライアントへ返却する`requestId`とサーバーログを関連付けられるようにする。

障害調査時に、

```text
requestId
    ↓
対象ログ
```

を追跡できる状態とする。

---

## 26. テスト観点

ACC-004では、通常属性更新だけでなく、

- PATCHによる部分更新
- 利用者境界
- 論理削除
- 資産口座名重複
- 更新対象外項目
- 同時更新
- 関連データ非更新
- レスポンス契約

を重点的に確認する。

---

### 26.1 nameのみ更新

以下のリクエストを送信する。

```json
{
  "name": "メイン証券口座"
}
```

以下を確認する。

- `200 OK`となること
- `asset_accounts.name`が更新されること
- `asset_accounts.asset_type`が変更されないこと
- `balance_recording_unit`が変更されないこと
- `start_year_month`が変更されないこと
- `deleted_at`が変更されないこと

---

### 26.2 assetTypeのみ更新

以下のリクエストを送信する。

```json
{
  "assetType": "SECURITIES"
}
```

以下を確認する。

- `200 OK`となること
- `asset_accounts.asset_type`が更新されること
- `asset_accounts.name`が変更されないこと
- その他の更新対象外項目が変更されないこと

---

### 26.3 nameとassetTypeを同時更新

以下を指定する。

```json
{
  "name": "メイン証券口座",
  "assetType": "SECURITIES"
}
```

両方が正しく更新されることを確認する。

---

### 26.4 空リクエスト

以下を送信する。

```json
{}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

`asset_accounts`が更新されないことも確認する。

---

### 26.5 name未指定

`assetType`だけを指定し、`name`未指定がエラーにならないことを確認する。

PATCHとして、未指定項目が必須扱いされないこと。

---

### 26.6 assetType未指定

`name`だけを指定し、`assetType`未指定がエラーにならないことを確認する。

---

### 26.7 name = null

以下を送信する。

```json
{
  "name": null
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.8 name = 空文字

以下を送信する。

```json
{
  "name": ""
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.9 name最大文字数

テーブル定義で定めた最大文字数ちょうどの値を更新できることを確認する。

最大文字数を超えた場合は、

```text
VALIDATION_ERROR
```

となること。

---

### 26.10 assetType正常値

以下をそれぞれ確認する。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

API値が正しいDB保存形式へ変換されること。

---

### 26.11 assetType不正

例えば、

```json
{
  "assetType": "CRYPTO"
}
```

を送信する。

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.12 assetType = null

以下を送信する。

```json
{
  "assetType": null
}
```

バリデーションエラーとなり、業務データが更新されないこと。

---

### 26.13 自分自身と同じname

現在、

```text
id = 10
name = 証券口座
```

の資産口座に対して、

```json
{
  "name": "証券口座"
}
```

を送信する。

重複エラーにならず、正常に処理できることを確認する。

---

### 26.14 同一利用者の別資産口座とのname重複

以下の状態を用意する。

```text
User A
├─ id = 10 / 証券口座
└─ id = 20 / 銀行口座
```

`id = 20`を

```json
{
  "name": "証券口座"
}
```

へ変更する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となること。

---

### 26.15 論理削除済み資産口座とのname重複

同一利用者に、

```text
name = 証券口座
deleted_at IS NOT NULL
```

の別資産口座を用意する。

利用中資産口座を同じ名前へ変更する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となること。

---

### 26.16 他利用者の同名資産口座

以下の状態を用意する。

```text
User A
    証券口座

User B
    銀行口座
```

User Bの`銀行口座`を

```text
証券口座
```

へ変更する。

正常に更新できることを確認する。

---

### 26.17 X-User-Id未指定

`X-User-Id`を指定しない。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

資産口座が更新されないこと。

---

### 26.18 X-User-Id形式不正

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

### 26.19 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 26.20 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 26.21 assetAccountId形式不正

以下の値を確認する。

```text
0
-1
abc
1.5
1e3
10abc
```

期待結果：

```text
400 Bad Request
INVALID_ASSET_ACCOUNT_ID
```

となること。

---

### 26.22 資産口座不存在

存在しない`assetAccountId`を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 26.23 他利用者の資産口座

以下の状態を用意する。

```text
User A
    Asset Account A

User B
    Asset Account B
```

User AとしてAsset Account Bを更新する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの資産口座が変更されないことも確認する。

---

### 26.24 論理削除済み資産口座

対象資産口座を

```text
deleted_at IS NOT NULL
```

としてACC-004を実行する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 26.25 balanceRecordingUnitを送信

以下を送信する。

```json
{
  "balanceRecordingUnit": "ACCOUNT"
}
```

更新対象外項目を拒否する共通方針の場合は、

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

`balance_recording_unit`が変更されないことを確認する。

---

### 26.26 startYearMonthを送信

以下を送信する。

```json
{
  "startYearMonth": "2026-09"
}
```

ACC-004では変更できないことを確認する。

`asset_accounts.start_year_month`が変更されないこと。

---

### 26.27 isAvailableを送信

以下を送信する。

```json
{
  "isAvailable": false
}
```

ACC-004では利用可能資産設定を変更できないことを確認する。

```text
asset_account_available_settings
```

に変更がないこと。

---

### 26.28 isEnabledを送信

以下を送信する。

```json
{
  "isEnabled": false
}
```

ACC-004から無効化できないことを確認する。

```text
deleted_at
```

が変更されないこと。

---

### 26.29 userIdを送信

以下を送信する。

```json
{
  "name": "証券口座",
  "userId": "999"
}
```

未定義項目を拒否する共通方針の場合は、バリデーションエラーとなること。

少なくとも、

```text
asset_accounts.user_id
```

が変更されないことを確認する。

---

### 26.30 isAvailable = true

現在有効な利用可能資産設定が

```text
is_available = true
```

の場合、成功レスポンスで

```json
{
  "isAvailable": true
}
```

となることを確認する。

---

### 26.31 isAvailable = false

現在有効な設定が

```text
is_available = false
```

の場合、

```json
{
  "isAvailable": false
}
```

となることを確認する。

---

### 26.32 現在利用可能資産設定が0件

現在有効な利用可能資産設定を0件とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

となること。

資産口座更新前に設定不整合を検出する実装の場合は、`asset_accounts`が変更されないことも確認する。

---

### 26.33 現在利用可能資産設定が複数件

現在有効な設定を2件以上用意する。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

となること。

任意の1件を成功レスポンスへ使用しないこと。

---

### 26.34 同一値更新

現在値と同じ

```text
name
assetType
```

を指定する。

業務エラーにならないことを確認する。

EloquentのDirty判定によってUPDATEが発行されない場合も正常とする。

---

### 26.35 updated_at

実際に値が変更された場合は、

```text
updated_at
```

が更新されることを確認する。

同一値更新時の`updated_at`の扱いは、実装方針に従う。

---

### 26.36 同時名前更新

同一利用者に属する2つの資産口座を、並行して同じ名前へ変更する。

以下を確認する。

- 同名資産口座が2件存在する状態にならないこと
- DBのUNIQUE制約が機能すること
- 一方が成功すること
- 競合側が`409 Conflict`相当となること
- `ASSET_ACCOUNT_NAME_ALREADY_EXISTS`へ変換されること

---

### 26.37 同一資産口座への同時更新

同じ資産口座に対して異なる値を並行更新する。

Phase1では楽観ロックを行わないため、Last Write Winsとなり得ることを確認する。

想定外の500エラーやデッドロックを通常発生させないことも確認する。

---

### 26.38 関連データ非更新

ACC-004実行前後で、以下が変更されないことを確認する。

- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

---

### 26.39 正常レスポンス契約

正常時に、概念的に以下の形式となることを確認する。

```json
{
  "data": {
    "id": "10",
    "name": "メイン証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

以下を確認する。

- HTTPステータスが`200 OK`
- `id`がstring
- `name`がstring
- `assetType`がAPI用文字列
- `balanceRecordingUnit`がAPI用文字列
- `isAvailable`がboolean
- `startYearMonth`が`YYYY-MM`
- `isEnabled = true`

---

### 26.40 返却しない情報

正常レスポンスへ、以下が含まれないことを確認する。

- `user_id`
- `userId`
- `deleted_at`
- `deletedAt`
- `created_at`
- `createdAt`
- `updated_at`
- `updatedAt`
- `asset_account_available_settings.id`
- `assetAccountAvailableSettingId`
- `asset_account_id`
- `endYearMonth`
- DB内部の数値コード
- 保有商品一覧
- 月末資産データ

---

### 26.41 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
INTERNAL_SERVER_ERROR
```

以下も確認する。

- `error.code`が設定されること
- `error.message`が設定されること
- 必要に応じて`error.details`が設定されること
- `requestId`が設定されること
- SQLが含まれないこと
- PostgreSQL内部エラーが含まれないこと
- 制約名が含まれないこと
- スタックトレースが含まれないこと
- サーバーファイルパスが含まれないこと

---

### 26.42 副作用範囲

正常終了時に変更される業務データが、

```text
対象asset_accounts 1件
```

だけであることを確認する。

更新対象カラムも、リクエストで指定された

```text
name
asset_type
```

に限定されることを確認する。

---

### 26.43 INTERNAL_SERVER_ERROR

更新処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

- 内部情報がレスポンスへ公開されないこと
- サーバーログに調査情報が記録されること
- レスポンスの`requestId`からログを追跡できること

---

## 27. Laravel実装方針

ACC-004では、Action、FormRequest、UseCase、Query、Repository、DTO、API Resource、Responderを分離して実装する。

概念的な構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
FormRequest
    ↓
Action
    ↓
UseCase
    ├─ AssetAccountQuery
    ├─ AssetAccountAvailableSettingQuery
    └─ AssetAccountRepository
    ↓
Update Result DTO
    ↓
API Resource
    ↓
Responder
```

ACC-004では、`asset_accounts`の通常属性だけを更新する。

更新対象は、

```text
name
asset_type
```

に限定する。

一方、

```text
balance_recording_unit
start_year_month
deleted_at
```

および

```text
asset_account_available_settings
```

は更新しない。

---

### 27.1 Route

ACC-004は、以下のルートとして定義する。

概念例：

```php
Route::patch(
    '/api/v1/asset-accounts/{assetAccountId}',
    UpdateAssetAccountAction::class,
);
```

ACC-003とは同じパスを使用するが、HTTPメソッドによって責務を分離する。

```text
GET
/api/v1/asset-accounts/{assetAccountId}
    → ACC-003 資産口座詳細取得

PATCH
/api/v1/asset-accounts/{assetAccountId}
    → ACC-004 資産口座更新
```

---

### 27.2 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、以下とする。

```text
X-User-Id取得
    ↓
必須チェック
    ↓
形式チェック
    ↓
users存在確認
    ↓
UserContext設定
    ↓
FormRequest
    ↓
Action
```

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 27.3 FormRequest

リクエストボディの入力値検証には、ACC-004専用のFormRequestを使用する。

概念例：

```php
final class UpdateAssetAccountRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => [
                'sometimes',
                'required',
                'string',
                'max:100',
            ],

            'assetType' => [
                'sometimes',
                'required',
                'string',
                Rule::enum(
                    AssetType::class,
                ),
            ],
        ];
    }
}
```

ただし、`name`と`assetType`の両方が未指定の場合はバリデーションエラーとする。

---

### 27.4 空リクエストの検証

以下のような空リクエストは許可しない。

```json
{}
```

FormRequestまたは専用Validatorで、

```text
name未指定
AND
assetType未指定
```

の場合は、

```text
VALIDATION_ERROR
```

とする。

概念例：

```php
public function after(): array
{
    return [
        function (
            Validator $validator,
        ): void {
            if (
                ! $this->has('name')
                && ! $this->has('assetType')
            ) {
                $validator->errors()->add(
                    'request',
                    '少なくとも1つの更新項目を指定してください。',
                );
            }
        },
    ];
}
```

---

### 27.5 FormRequestで行うこと

FormRequestでは、以下の入力形式を検証する。

```text
name
assetType
```

主に以下を確認する。

- 項目指定有無
- 文字列型
- NULL禁止
- 空文字禁止
- 最大文字数
- Enum値
- 少なくとも1項目指定

HTTP入力として不正な値をUseCaseへ渡さない。

---

### 27.6 FormRequestで行わないこと

FormRequestでは、以下の業務ルールを扱わない。

- 資産口座存在確認
- 利用者境界確認
- 論理削除判定
- 資産口座名重複確認
- 現在利用可能資産設定確認
- DB更新
- 同時更新競合判定

これらは、UseCase、Query、Repositoryで扱う。

---

### 27.7 更新対象外項目

ACC-004では、以下を更新対象として定義しない。

```text
userId
balanceRecordingUnit
startYearMonth
isAvailable
isEnabled
deletedAt
```

API共通方針として未定義項目を拒否する場合は、これらが送信された時点で`VALIDATION_ERROR`とする。

Request全体をそのままModelへ渡す実装は行わない。

---

### 27.8 assetAccountIdの形式検証

`assetAccountId`は、ルート制約またはAPI共通のパスパラメータ検証方式で検証する。

概念例：

```php
Route::patch(
    '/api/v1/asset-accounts/{assetAccountId}',
    UpdateAssetAccountAction::class,
)
    ->where(
        'assetAccountId',
        '[1-9][0-9]*',
    );
```

正の整数形式のみを許可する。

形式不正の場合は、

```text
INVALID_ASSET_ACCOUNT_ID
```

へ変換する。

---

### 27.9 Action

Actionは、検証済みリクエスト、パスパラメータ、利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php
final class UpdateAssetAccountAction
{
    public function __invoke(
        string $assetAccountId,
        UpdateAssetAccountRequest $request,
        UpdateAssetAccountUseCase $useCase,
        UpdateAssetAccountResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $input =
            new UpdateAssetAccountInput(
                userId:
                    $userContext->userId,

                assetAccountId:
                    (int) $assetAccountId,

                name:
                    $request->has('name')
                        ? $request->string(
                            'name',
                        )->toString()
                        : null,

                assetType:
                    $request->has('assetType')
                        ? AssetType::from(
                            $request->string(
                                'assetType',
                            )->toString(),
                        )
                        : null,
            );

        $result =
            $useCase->execute(
                $input,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

---

### 27.10 Input DTO

PATCHでは、「未指定」と「指定済み」を区別する必要がある。

そのため、Input DTOでは更新可能項目をnullableとして表現してよい。

概念例：

```php
final readonly class UpdateAssetAccountInput
{
    public function __construct(
        public int $userId,
        public int $assetAccountId,
        public ?string $name,
        public ?AssetType $assetType,
    ) {
    }
}
```

ただし、ACC-004では`NULL`値そのものを更新値として許可しないため、

```text
null
    = Requestで未指定
```

という意味に限定する。

---

### 27.11 未指定とNULL入力を混同しない

例えば、

```json
{
  "name": null
}
```

はバリデーションエラーとする。

一方、

```json
{
  "assetType": "SECURITIES"
}
```

の場合の

```text
name = null
```

は、Input DTO上で「name未指定」を意味する。

Actionでは`has()`や`array_key_exists()`相当の判定を使用し、未指定とNULL入力を区別する。

---

### 27.12 Actionで行わないこと

Actionでは、以下を行わない。

- 資産口座検索
- 利用者境界判定
- 同名重複確認
- 現在年月取得
- 利用可能資産設定検索
- UPDATE
- UNIQUE制約例外処理
- レスポンス配列生成

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 27.13 UseCase

ACC-004の業務処理全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `assetAccountId`を受け取る
3. 更新対象資産口座を取得する
4. 対象不存在の場合は業務例外を送出する
5. 現在年月を取得する
6. 現在有効な利用可能資産設定が1件であることを確認する
7. `name`指定時のみ同名重複を確認する
8. 指定された項目のみ更新する
9. 更新結果DTOを生成する
10. DTOを返却する

概念的には、以下とする。

```text
UpdateAssetAccountInput
    ↓
AssetAccountQuery
    ↓
更新対象取得
    ↓
CurrentYearMonth取得
    ↓
AssetAccountAvailableSettingQuery
    ↓
現在設定1件確認
    ↓
name指定あり？
    ├─ Yes
    │    ↓
    │ AssetAccountQuery
    │ 同名重複確認
    └─ No
    ↓
AssetAccountRepository
    ↓
部分更新
    ↓
Update Result DTO
```

---

### 27.14 AssetAccountQuery

更新対象資産口座の取得は、Queryへ委譲する。

概念例：

```php
$assetAccount =
    $this->assetAccountQuery
        ->findActiveByIdAndUser(
            assetAccountId:
                $input->assetAccountId,

            userId:
                $input->userId,
        );
```

概念的な条件は、以下とする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 27.15 利用者境界をQueryへ含める

以下のように、IDだけで取得してから所有利用者を判定する方式は基本としない。

```php
$assetAccount =
    AssetAccount::find(
        $assetAccountId,
    );
```

対象取得時点で、

```text
assetAccountId
+
userId
+
有効状態
```

を条件へ含める。

---

### 27.16 SoftDeletes

`AssetAccount` Modelでは、Laravelの

```php
use SoftDeletes;
```

を使用する。

ACC-004の更新対象取得では、

```php
withTrashed()
```

を使用しない。

論理削除済み資産口座は更新対象外とする。

---

### 27.17 ASSET_ACCOUNT_NOT_FOUND

対象資産口座を取得できない場合は、業務例外を送出する。

概念例：

```php
if ($assetAccount === null) {
    throw new
        AssetAccountNotFoundException();
}
```

以下を同じ例外へ集約する。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

最終的に、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

へ変換する。

---

### 27.18 現在利用可能資産設定をUPDATE前に確認する

ACC-004の成功レスポンスでは、現在の

```text
isAvailable
```

を返却する。

そのため、資産口座UPDATE後に初めて利用可能資産設定不整合を検出する構成は避ける。

概念的には、

```text
更新対象取得
    ↓
現在設定確認
    ↓
設定1件
    ↓
更新
```

とする。

これにより、

```text
asset_accounts更新成功
    ↓
現在設定取得失敗
    ↓
500レスポンス
```

という分かりにくい状態を避ける。

---

### 27.19 CurrentYearMonth

現在利用可能資産設定判定には、サーバー側の現在年月を使用する。

概念例：

```php
$currentYearMonth =
    now()->format(
        'Y-m',
    );
```

ACC-003と同様に、必要に応じてClockやDateProviderへ時刻依存を分離してよい。

---

### 27.20 AssetAccountAvailableSettingQuery

現在有効な利用可能資産設定の取得は、専用Queryへ委譲する。

概念例：

```php
$settings =
    $this->availableSettingQuery
        ->findCurrentByAssetAccount(
            assetAccountId:
                $assetAccount->id,

            yearMonth:
                $currentYearMonth,
        );
```

概念的な条件は、以下とする。

```text
asset_account_id
    = assetAccountId

AND

start_year_month
    <= currentYearMonth

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= currentYearMonth
)
```

---

### 27.21 利用可能資産設定件数

現在有効な設定は、1件であることを必須とする。

概念例：

```php
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingNotFoundException();
}

if ($settings->count() > 1) {
    throw new
        AssetAccountAvailableSettingConflictException();
}
```

0件の場合に`false`へ補完せず、複数件の場合に任意の1件を採用しない。

---

### 27.22 利用可能資産設定は更新しない

ACC-004では、

```text
asset_account_available_settings
```

に対する

```text
INSERT
UPDATE
DELETE
```

を行わない。

`isAvailable`変更はACC-006へ委譲する。

---

### 27.23 name指定時のみ重複確認する

`name`がRequestで指定された場合のみ、同名重複確認を行う。

概念例：

```php
if ($input->name !== null) {
    $exists =
        $this->assetAccountQuery
            ->existsDuplicateNameIncludingDeleted(
                userId:
                    $input->userId,

                name:
                    $input->name,

                exceptAssetAccountId:
                    $input->assetAccountId,
            );

    if ($exists) {
        throw new
            AssetAccountNameAlreadyExistsException();
    }
}
```

`assetType`だけの更新では、不要な名前重複Queryを実行しない。

---

### 27.24 論理削除済みも重複確認対象とする

重複確認では、SoftDeletesの通常スコープを外し、論理削除済み資産口座も検索対象に含める。

概念例：

```php
return AssetAccount::query()
    ->withTrashed()
    ->where(
        'user_id',
        $userId,
    )
    ->where(
        'name',
        $name,
    )
    ->whereKeyNot(
        $exceptAssetAccountId,
    )
    ->exists();
```

実際のメソッドは、Laravelバージョンに合わせて適切な記述とする。

---

### 27.25 更新対象自身を除外する

名前重複確認では、更新対象自身を除外する。

概念的には、

```text
id != assetAccountId
```

を条件へ含める。

これにより、現在の名前と同じ名前を再指定しても重複エラーとならない。

---

### 27.26 AssetAccountRepository

資産口座更新は、Repositoryへ委譲する。

概念的なインターフェースは、以下とする。

```php
interface AssetAccountRepository
{
    public function update(
        AssetAccount $assetAccount,
        ?string $name,
        ?AssetType $assetType,
    ): AssetAccount;
}
```

ただし、引数が増える場合は更新専用DTOを使用してよい。

---

### 27.27 Repositoryで部分更新する

Repositoryでは、指定された項目だけを更新する。

概念例：

```php
public function update(
    AssetAccount $assetAccount,
    ?string $name,
    ?AssetType $assetType,
): AssetAccount {
    if ($name !== null) {
        $assetAccount->name =
            $name;
    }

    if ($assetType !== null) {
        $assetAccount->asset_type =
            $assetType;
    }

    $assetAccount->save();

    return $assetAccount;
}
```

未指定項目をNULLやデフォルト値へ置き換えない。

---

### 27.28 Mass Assignmentを使用しない

以下のような処理は行わない。

```php
$assetAccount->update(
    $request->all(),
);
```

更新対象を

```text
name
asset_type
```

へ限定するため、明示的に値を設定する。

---

### 27.29 更新対象外カラムを変更しない

Repositoryでは、以下を変更しない。

```text
user_id
balance_recording_unit
start_year_month
deleted_at
```

また、別Repositoryを呼び出して

```text
asset_account_available_settings
```

を更新しない。

---

### 27.30 Enum Cast

`asset_type`は、PHP EnumまたはEloquent Castとして扱う。

概念例：

```php
protected function casts(): array
{
    return [
        'asset_type'
            => AssetType::class,

        'balance_recording_unit'
            => BalanceRecordingUnit::class,
    ];
}
```

UseCaseやRepositoryへDB内部の数値コードを直接持ち込まない。

---

### 27.31 同一値更新

現在値と同一値が指定された場合は、業務エラーとしない。

Eloquentでは、Dirty判定により実際の変更がない場合にUPDATEが発行されないことがある。

概念的には、

```php
$assetAccount->save();
```

を使用し、Laravel標準のDirty判定に従ってよい。

---

### 27.32 updated_at

実際にEloquent上で変更が発生した場合は、

```text
updated_at
```

が通常どおり更新される。

クライアントから`updatedAt`を指定させない。

同一値更新でDirtyでない場合は、`updated_at`が変わらなくてもよい。

---

### 27.33 トランザクション

Phase1では、ACC-004専用の

```php
DB::transaction()
```

は必須としない。

更新対象が

```text
asset_accounts 1件
```

だけであり、関連テーブルを更新しないためである。

現在設定確認や名前重複確認はUPDATE前に行い、単一UPDATEで完結させる。

---

### 27.34 lockForUpdateを使用しない

ACC-004では、Phase1で

```php
lockForUpdate()
```

を使用しない。

同一資産口座への更新競合はLast Write Winsとなる可能性を許容する。

また、別資産口座間の同名競合はUNIQUE制約で保証する。

---

### 27.35 UNIQUE制約

同一利用者内の資産口座名は、DB側でも

```text
user_id
+
name
```

で一意性を保証する。

アプリケーション側の事前重複確認を同時実行した複数リクエストが通過しても、最終的にDB制約で重複を防止する。

---

### 27.36 UNIQUE制約違反の変換

並行更新などによってUNIQUE制約違反が発生した場合は、PostgreSQL例外をそのまま返却しない。

概念的には、

```text
unique violation
    ↓
AssetAccountNameAlreadyExistsException
    ↓
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

制約名やSQLSTATEはレスポンスへ公開しない。

---

### 27.37 Update Result DTO

更新結果は、専用DTOとして表現する。

概念例：

```php
final readonly class UpdateAssetAccountResult
{
    public function __construct(
        public int $id,
        public string $name,
        public AssetType $assetType,
        public BalanceRecordingUnit $balanceRecordingUnit,
        public bool $isAvailable,
        public string $startYearMonth,
    ) {
    }
}
```

`isEnabled`は、ACC-004成功時には必ず`true`となるため、Resource側で固定値として設定してよい。

---

### 27.38 DTOへEloquent Modelを保持しない

以下のようなDTOは基本としない。

```php
final readonly class UpdateAssetAccountResult
{
    public function __construct(
        public AssetAccount $assetAccount,
        public AssetAccountAvailableSetting $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

---

### 27.39 UseCaseの戻り値

概念例：

```php
return new UpdateAssetAccountResult(
    id:
        $assetAccount->id,

    name:
        $assetAccount->name,

    assetType:
        $assetAccount->asset_type,

    balanceRecordingUnit:
        $assetAccount->balance_recording_unit,

    isAvailable:
        $setting->is_available,

    startYearMonth:
        $assetAccount->start_year_month,
);
```

Eloquent ModelをActionへ直接返却しない。

---

### 27.40 API Resource

Update Result DTOを、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php
final class UpdateAssetAccountResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'name'
                => $this->name,

            'assetType'
                => $this->assetType->value,

            'balanceRecordingUnit'
                => $this->balanceRecordingUnit->value,

            'isAvailable'
                => $this->isAvailable,

            'startYearMonth'
                => $this->startYearMonth,

            'isEnabled'
                => true,
        ];
    }
}
```

APIフィールド名は、API共通方針に従ってcamelCaseとする。

---

### 27.41 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

- `user_id`
- `deleted_at`
- `created_at`
- `updated_at`
- `asset_account_available_settings.id`
- `asset_account_available_settings.asset_account_id`
- `asset_account_available_settings.start_year_month`
- `asset_account_available_settings.end_year_month`
- DB内部コード

また、以下も返却しない。

- 保有商品一覧
- 月末資産残高
- 商品別月末評価額
- 月末資産状況
- 資産推移

---

### 27.42 Responder

Responderは、Update Result DTOを`200 OK`レスポンスへ変換する。

概念例：

```php
final class UpdateAssetAccountResponder
{
    public function ok(
        UpdateAssetAccountResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new UpdateAssetAccountResource(
                        $result,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの共通Envelope項目は、API共通レスポンス処理に従う。

---

### 27.43 Responderで行わないこと

Responderでは、以下を行わない。

- 資産口座検索
- 利用者境界判定
- 資産口座名重複確認
- 現在年月取得
- 利用可能資産設定取得
- DB更新
- UNIQUE制約処理
- 業務例外判定

Responderは、生成済みDTOをHTTPレスポンスへ変換することに責務を限定する。

---

### 27.44 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `assetAccountId`形式不正 | `INVALID_ASSET_ACCOUNT_ID` |
| Request入力値不正 | `VALIDATION_ERROR` |
| 資産口座不存在 | `ASSET_ACCOUNT_NOT_FOUND` |
| 他利用者の資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 論理削除済み資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 同名資産口座存在 | `ASSET_ACCOUNT_NAME_ALREADY_EXISTS` |
| UNIQUE制約による同名競合 | `ASSET_ACCOUNT_NAME_ALREADY_EXISTS` |
| 現在設定0件 | `ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND` |
| 現在設定複数件 | `ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

---

### 27.45 AssetAccountNameAlreadyExistsException

同一利用者内に同名の別資産口座が存在する場合は、業務例外を送出する。

概念例：

```php
if ($exists) {
    throw new
        AssetAccountNameAlreadyExistsException();
}
```

最終的に、

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

---

### 27.46 利用可能資産設定不整合例外

現在設定が0件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

へ変換する。

現在設定が複数件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

へ変換する。

どちらも500系のデータ不整合として扱う。

---

### 27.47 ValidationException

FormRequestによる入力値不正は、API共通形式の

```text
400 Bad Request
VALIDATION_ERROR
```

へ変換する。

複数のフィールドエラーがある場合は、`error.details`へ複数件返却してよい。

---

### 27.48 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- SQL
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

---

### 27.49 ログ

ACC-004では、必要に応じて以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assetAccountId
httpStatus
errorCode
```

`apiId`は、

```text
ACC-004
```

とする。

正常時には、更新対象IDを記録してよい。

ただし、資産口座名などの不要な業務情報を通常ログへ出力しない。

---

### 27.50 データ不整合ログ

以下の場合は、調査に必要な情報を内部ログへ記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text
assetAccountId
currentYearMonth
matchedSettingCount
```

を記録してよい。

---

### 27.51 キャッシュ

Phase1では、ACC-004専用のサーバー側キャッシュを使用しない。

React側でTanStack Queryを使用する場合は、ACC-004成功後に

```text
ACC-001
資産口座一覧

ACC-003
対象資産口座詳細
```

のQuery Cacheを無効化する。

---

### 27.52 テスト実装方針

Laravel側では、Feature Testを中心としてACC-004のAPI契約と部分更新処理を確認する。

主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `404 Not Found`
- `409 Conflict`
- `500 Internal Server Error`
- `X-User-Id`必須
- `assetAccountId`形式
- `name`のみ更新
- `assetType`のみ更新
- 複数項目更新
- 空リクエスト
- NULL
- 未定義項目
- 利用者境界
- SoftDeletes
- 資産口座名重複
- 論理削除済み資産口座との名前重複
- 現在利用可能資産設定確認
- UNIQUE制約
- 同時更新
- 関連データ非更新
- レスポンス契約

---

### 27.53 FormRequestのTest

FormRequestについて、以下を確認する。

```text
name
    sometimes
    required
    string
    max length

assetType
    sometimes
    required
    enum

name未指定
+
assetType未指定
    → error
```

以下も確認する。

- `name = null`はエラー
- `assetType = null`はエラー
- 空リクエストはエラー
- `name`だけは正常
- `assetType`だけは正常

---

### 27.54 AssetAccountQueryのDatabase Test

更新対象取得について、以下を確認する。

```text
id一致
+
user_id一致
+
deleted_at IS NULL
    ↓
取得できる
```

以下は取得できないこと。

- 存在しないID
- 他利用者の資産口座
- 論理削除済み資産口座

---

### 27.55 名前重複QueryのDatabase Test

以下を確認する。

```text
同一user_id
+
同一name
+
別id
    ↓
重複
```

```text
同一user_id
+
同一name
+
論理削除済み別id
    ↓
重複
```

```text
同一user_id
+
同一name
+
同じid
    ↓
重複ではない
```

```text
別user_id
+
同一name
    ↓
重複ではない
```

---

### 27.56 AssetAccountAvailableSettingQueryのDatabase Test

現在年月を固定し、現在有効な設定が正しく取得できることを確認する。

境界条件として、以下を確認する。

- `start_year_month = currentYearMonth`
- `end_year_month = currentYearMonth`
- `end_year_month = 前月`
- `start_year_month = 翌月`
- `end_year_month = NULL`

---

### 27.57 AssetAccountRepositoryのDatabase Test

`name`だけを渡した場合に、

```text
name
    → 更新

asset_type
    → 維持
```

となることを確認する。

`assetType`だけの場合は、

```text
asset_type
    → 更新

name
    → 維持
```

となることを確認する。

また、以下が変更されないことも確認する。

```text
user_id
balance_recording_unit
start_year_month
deleted_at
```

---

### 27.58 同一値更新Test

現在値と同じ値を指定する。

以下を確認する。

- 業務エラーにならない
- 最終業務状態が変わらない
- EloquentがDirtyでない場合は正常
- `updated_at`が変わらなくても正常

---

### 27.59 UseCaseのUnit Test

QueryとRepositoryをMockし、以下を確認する。

正常系：

```text
AssetAccountQuery
    ↓
資産口座取得
    ↓
AvailableSettingQuery
    ↓
現在設定1件
    ↓
必要なら名前重複確認
    ↓
Repository::update()
    ↓
Update Result DTO
```

対象不存在：

```text
AssetAccountQuery
    ↓
null
    ↓
AssetAccountNotFoundException
```

名前重複：

```text
DuplicateQuery
    ↓
true
    ↓
AssetAccountNameAlreadyExistsException
```

設定0件：

```text
AvailableSettingQuery
    ↓
0件
    ↓
AssetAccountAvailableSettingNotFoundException
```

設定複数件：

```text
AvailableSettingQuery
    ↓
2件以上
    ↓
AssetAccountAvailableSettingConflictException
```

---

### 27.60 name未指定時のTest

`assetType`だけを更新する場合に、名前重複確認Queryが呼び出されないことを確認する。

これにより、不要なDBアクセスを防止する。

---

### 27.61 UNIQUE制約競合Test

可能であれば、Integration TestまたはDatabase Testで、同一利用者の別資産口座を同じ名前へ並行更新する。

以下を確認する。

- 同名資産口座が2件存在しないこと
- 競合側がDB制約で失敗すること
- PostgreSQL例外がそのまま公開されないこと
- `ASSET_ACCOUNT_NAME_ALREADY_EXISTS`へ変換されること

---

### 27.62 API ResourceのTest

正常時に、以下の項目だけが`data`へ含まれることを確認する。

```json
{
  "id": "10",
  "name": "メイン証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "isAvailable": true,
  "startYearMonth": "2026-08",
  "isEnabled": true
}
```

以下が含まれないことを確認する。

- `userId`
- `user_id`
- `deletedAt`
- `createdAt`
- `updatedAt`
- `assetAccountAvailableSettingId`
- `asset_account_id`
- `endYearMonth`
- DB内部コード

---

### 27.63 関連データ非更新Test

ACC-004実行前後で、以下に変更がないことを確認する。

```text
asset_account_available_settings
holding_assets
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
net_incomes
objectives
assessment_histories
```

ACC-004の副作用が対象`asset_accounts`1件に限定されていることを保証する。

---

## 28. React・TypeScriptでの利用

ACC-004は、利用者が登録済み資産口座の通常属性を編集する際に使用する。

フロントエンドでは、ACC-003 資産口座詳細取得で現在値を取得し、編集フォームへ反映したうえでACC-004を実行する。

概念的な利用フローは、以下とする。

```text
ACC-003
資産口座詳細取得
    ↓
編集フォーム初期表示
    ↓
利用者が編集
    ↓
変更内容をRequestへ変換
    ↓
ACC-004
PATCH
/api/v1/asset-accounts/{assetAccountId}
    ↓
成功
    ↓
関連Query Cache無効化
    ↓
最新状態を再取得
```

ACC-004は、サーバー状態を変更するため、TanStack Queryを使用する場合はMutationとして扱う。

---

### 28.1 TypeScript型

ACC-004のリクエスト型は、以下のように定義する。

概念例：

```typescript
export type UpdateAssetAccountRequest = {
  name?: string;
  assetType?: AssetType;
};
```

`PATCH`であるため、各項目は任意とする。

ただし、実際にAPIを呼び出す際は、

```text
name
assetType
```

の少なくとも1項目を指定する必要がある。

---

### 28.2 正常レスポンス型

正常レスポンスは、ACC-003と同じ資産口座詳細表現を使用する。

概念例：

```typescript
export type UpdateAssetAccountResult = {
  id: string;
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  isAvailable: boolean;
  startYearMonth: string;
  isEnabled: boolean;
};
```

API共通Envelopeを使用する場合は、以下のように定義する。

```typescript
export type UpdateAssetAccountResponse =
  ApiResponse<UpdateAssetAccountResult>;
```

---

### 28.3 ACC-003と共通型を使用してよい

ACC-003とACC-004の正常レスポンス項目が同一である場合は、専用の重複型を作成せず、共通型を使用してよい。

概念例：

```typescript
export type AssetAccountDetail = {
  id: string;
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  isAvailable: boolean;
  startYearMonth: string;
  isEnabled: boolean;
};

export type UpdateAssetAccountResponse =
  ApiResponse<AssetAccountDetail>;
```

同じ意味のデータにAPIごとに異なる型名を増やしすぎない。

---

### 28.4 編集可能な項目

ACC-004で編集できる項目は、以下のみとする。

```text
name
assetType
```

編集画面では、これらを入力可能な状態とする。

---

### 28.5 編集できない項目

以下は、ACC-004では更新できない。

- `balanceRecordingUnit`
- `startYearMonth`
- `isAvailable`
- `isEnabled`

ACC-003から取得したこれらの情報を画面へ表示してもよいが、ACC-004用フォームでは編集可能な入力欄にしない。

---

### 28.6 balanceRecordingUnit

残高記録単位は、Phase1では登録後変更不可とする。

そのため、編集画面で表示する場合は読み取り専用とする。

概念例：

```tsx
<ReadonlyField
  label="残高記録単位"
  value={
    balanceRecordingUnitLabels[
      assetAccount.balanceRecordingUnit
    ]
  }
/>
```

以下のような編集可能なセレクトボックスにはしない。

```tsx
<select
  value={form.balanceRecordingUnit}
>
  ...
</select>
```

---

### 28.7 startYearMonth

利用開始年月も、ACC-004では変更不可とする。

画面へ表示する場合は、読み取り専用とする。

例えば、

```text
利用開始年月
2026年8月
```

のように表示する。

---

### 28.8 isAvailable

利用可能資産区分の変更は、ACC-006の責務である。

そのため、ACC-004の編集フォームへ

```typescript
isAvailable: boolean;
```

を更新項目として含めない。

利用可能資産区分を変更するUIは、ACC-006を実行する別操作として設計する。

---

### 28.9 isEnabled

資産口座無効化は、ACC-005の責務とする。

そのため、ACC-004のRequestへ

```typescript
isEnabled?: boolean;
```

を追加しない。

以下のような汎用的な状態更新は行わない。

```typescript
updateAssetAccount(
  assetAccountId,
  {
    isEnabled: false,
  },
);
```

---

### 28.10 編集フォーム型

画面上の編集フォームStateは、以下のように定義できる。

概念例：

```typescript
export type UpdateAssetAccountFormValues = {
  name: string;
  assetType: AssetType;
};
```

ACC-003から取得した編集不可項目までフォームStateへ含める必要はない。

---

### 28.11 ACC-003から初期値を設定する

編集画面では、ACC-003で取得した

```text
name
assetType
```

を初期値として使用する。

概念例：

```typescript
const initialValues:
  UpdateAssetAccountFormValues = {
    name:
      assetAccount.name,

    assetType:
      assetAccount.assetType,
  };
```

---

### 28.12 Queryデータを直接編集しない

ACC-003で取得したQuery Cache上の

```text
AssetAccountDetail
```

を、フォーム入力によって直接書き換えない。

例えば、

```typescript
assetAccount.name =
  inputValue;
```

のような変更は行わない。

編集用Stateへ必要な値をコピーする。

---

### 28.13 変更項目だけを送信する

ACC-004はPATCHであるため、実際に変更された項目だけをRequestへ含めてよい。

例えば、資産口座名だけを変更した場合は、

```json
{
  "name": "メイン証券口座"
}
```

とする。

---

### 28.14 assetTypeだけ変更した場合

資産種別だけを変更した場合は、

```json
{
  "assetType": "BANK"
}
```

とする。

未変更の`name`を必ず送信する必要はない。

---

### 28.15 両方変更した場合

両方変更した場合は、

```json
{
  "name": "メイン証券口座",
  "assetType": "SECURITIES"
}
```

とする。

---

### 28.16 Request生成

現在値とフォーム値を比較し、変更された項目だけをRequestへ設定してよい。

概念例：

```typescript
export const buildUpdateAssetAccountRequest =
  (
    current: AssetAccountDetail,
    form: UpdateAssetAccountFormValues,
  ): UpdateAssetAccountRequest => {
    const request:
      UpdateAssetAccountRequest = {};

    if (
      form.name !== current.name
    ) {
      request.name =
        form.name;
    }

    if (
      form.assetType
        !== current.assetType
    ) {
      request.assetType =
        form.assetType;
    }

    return request;
  };
```

---

### 28.17 空Requestを送信しない

変更がない場合は、

```json
{}
```

をACC-004へ送信しない。

概念例：

```typescript
const request =
  buildUpdateAssetAccountRequest(
    assetAccount,
    form,
  );

if (
  Object.keys(request).length === 0
) {
  return;
}
```

画面上では、例えば

```text
変更内容がありません。
```

と表示してよい。

または、変更がない間は更新ボタンを非活性化してもよい。

---

### 28.18 更新ボタンの活性制御

現在値とフォーム値が同一の場合は、更新ボタンを非活性化してよい。

概念例：

```typescript
const hasChanges =
  form.name !==
    assetAccount.name
  ||
  form.assetType !==
    assetAccount.assetType;
```

```tsx
<button
  type="submit"
  disabled={
    !hasChanges
    || updateMutation.isPending
  }
>
  更新
</button>
```

ただし、サーバー側でも空Requestを拒否する。

---

### 28.19 nameの前処理

資産口座名について、API共通の文字列入力方針で前後空白を除去する場合は、Request生成時に統一して処理してよい。

概念例：

```typescript
const name =
  form.name.trim();
```

ただし、フロントエンドとバックエンドで異なる正規化ルールを持たせない。

---

### 28.20 AssetType

`assetType`は、ACC-002・ACC-003と共通の型を使用する。

概念例：

```typescript
export type AssetType =
  | 'CASH'
  | 'BANK'
  | 'SECURITIES'
  | 'IDECO'
  | 'CORPORATE_DC'
  | 'OTHER';
```

画面表示用ラベルは、共通定義を使用する。

---

### 28.21 API Client

ACC-004を呼び出す専用API Client関数を定義する。

概念例：

```typescript
export const updateAssetAccount =
  async (
    assetAccountId: string,
    request: UpdateAssetAccountRequest,
  ): Promise<AssetAccountDetail> => {
    const response =
      await apiClient.patch<
        UpdateAssetAccountResponse
      >(
        `/api/v1/asset-accounts/${assetAccountId}`,
        request,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

### 28.22 userIdをAPI Client引数へ含めない

以下のような関数にはしない。

```typescript
updateAssetAccount(
  userId,
  assetAccountId,
  request,
);
```

利用者IDは、共通API Clientから

```text
X-User-Id
```

として付与する。

---

### 28.23 X-User-Id

概念例：

```typescript
apiClient.interceptors.request.use(
  (config) => {
    config.headers['X-User-Id'] =
      currentUserId;

    return config;
  },
);
```

ACC-004専用コンポーネントでヘッダーを直接生成しない。

---

### 28.24 Mutationとして扱う

ACC-004は、資産口座の状態を変更するため、TanStack QueryではMutationとして扱う。

概念例：

```typescript
export type UpdateAssetAccountVariables = {
  assetAccountId: string;
  request: UpdateAssetAccountRequest;
};
```

```typescript
export const useUpdateAssetAccount =
  () => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn: (
        variables:
          UpdateAssetAccountVariables,
      ) =>
        updateAssetAccount(
          variables.assetAccountId,
          variables.request,
        ),

      onSuccess: async (
        result,
      ) => {
        await queryClient
          .invalidateQueries({
            queryKey:
              assetAccountKeys.all,
          });

        await queryClient
          .invalidateQueries({
            queryKey:
              assetAccountKeys.detail(
                result.id,
              ),
          });
      },
    });
  };
```

---

### 28.25 更新処理

概念例：

```typescript
const updateMutation =
  useUpdateAssetAccount();

const handleSubmit = (): void => {
  const request =
    buildUpdateAssetAccountRequest(
      assetAccount,
      form,
    );

  if (
    Object.keys(request).length === 0
  ) {
    return;
  }

  updateMutation.mutate({
    assetAccountId:
      assetAccount.id,

    request,
  });
};
```

---

### 28.26 二重送信防止

Mutation実行中は、更新ボタンを非活性化する。

概念例：

```tsx
<button
  type="submit"
  disabled={
    updateMutation.isPending
  }
>
  {updateMutation.isPending
    ? '更新中...'
    : '更新'}
</button>
```

フロントエンド側の二重送信防止だけをサーバー側整合性保証とはしない。

---

### 28.27 自動Retry

ACC-004のMutationでは、原則として自動Retryを行わない。

概念例：

```typescript
useMutation({
  mutationFn:
    updateAssetAccount,
  retry: false,
});
```

更新処理であり、通信結果が不明な状態で自動再送するより、利用者へ状態を通知したうえで必要に応じて再取得する方針とする。

---

### 28.28 同一Requestの再送

ACC-004は、同じ値を再度設定しても最終的な業務状態は同じとなる。

そのため、利用者が手動で再実行すること自体は可能である。

ただし、ネットワークエラー時にフロントエンドが自動的に繰り返し送信する設計はPhase1では採用しない。

---

### 28.29 成功時

ACC-004成功時は、

```http
200 OK
```

とともに、更新後の資産口座詳細が返却される。

概念例：

```json
{
  "data": {
    "id": "10",
    "name": "メイン証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

---

### 28.30 成功メッセージ

正常終了後は、必要に応じて

```text
資産口座を更新しました。
```

などの完了メッセージを表示する。

---

### 28.31 成功後の画面遷移

更新成功後は、画面設計に応じて

- 資産口座詳細画面へ戻る
- 資産口座一覧画面へ戻る
- 編集画面に留まり最新値を表示する

などの動作とする。

Phase1では、詳細画面または一覧画面へ戻るシンプルな導線としてよい。

---

### 28.32 ACC-001のQuery Cache

ACC-004によって、

```text
name
assetType
```

が変更されるため、資産口座一覧のQuery Cacheを無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.all,
});
```

---

### 28.33 ACC-003のQuery Cache

対象資産口座の詳細Queryも無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

これにより、ACC-003から更新後の最新状態を取得する。

---

### 28.34 成功レスポンスでCacheを更新する方式

ACC-004の成功レスポンスは、ACC-003と同じ資産口座詳細表現であるため、成功結果を直接詳細Query Cacheへ設定することもできる。

概念例：

```typescript
queryClient.setQueryData(
  assetAccountKeys.detail(
    result.id,
  ),
  result,
);
```

ただし、Phase1では

```text
Mutation成功
    ↓
invalidate
    ↓
再取得
```

の単純な方式を基本としてよい。

---

### 28.35 Optimistic Update

Phase1では、ACC-004に対するOptimistic Updateを必須としない。

資産口座名については、

- 同名重複
- 他利用者境界
- 論理削除状態
- UNIQUE制約競合

など、サーバー側で最終判定する条件がある。

そのため、サーバー成功後に画面状態を更新する方式を基本とする。

---

### 28.36 VALIDATION_ERROR

Laravel側から

```text
VALIDATION_ERROR
```

が返却された場合は、`error.details`を利用して対応するフォーム項目へエラー表示する。

主な対象は、

```text
name
assetType
```

とする。

---

### 28.37 nameのフィールドエラー

例えば、

```text
資産口座名を入力してください。
```

や、

```text
資産口座名は100文字以内で入力してください。
```

などを、資産口座名入力欄の近くへ表示する。

具体的な文言は、APIレスポンスまたはフロントエンド共通エラー表示方針に従う。

---

### 28.38 assetTypeのフィールドエラー

`assetType`に未定義値が返された場合は、資産種別入力欄のエラーとして扱う。

通常のUIでは定義済みセレクト値しか送信しないため、発生頻度は低い。

---

### 28.39 ASSET_ACCOUNT_NAME_ALREADY_EXISTS

同一利用者内に同名資産口座が存在する場合は、

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となる。

フロントエンドでは、例えば

```text
同じ名前の資産口座が
すでに登録されています。
```

と表示する。

`name`に関連する業務エラーとして、資産口座名入力欄の近くへ表示してよい。

---

### 28.40 同時更新による重複も同じ扱いとする

DBのUNIQUE制約によって名前競合が検出された場合も、フロントエンドでは

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

として扱う。

アプリケーション事前判定か、DB制約による判定かをクライアントから区別しない。

---

### 28.41 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

となる。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

フロントエンドでは、原因を推測せず、

```text
指定された資産口座が見つかりません。
```

などの共通表示とする。

---

### 28.42 更新中に無効化された場合

編集画面表示後、別処理によって対象資産口座が無効化される可能性がある。

その後ACC-004を実行すると、

```text
ASSET_ACCOUNT_NOT_FOUND
```

となり得る。

この場合は、編集画面へ残して再送を繰り返さず、資産口座一覧へ戻る導線を表示してよい。

---

### 28.43 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-004専用のフォームエラーとして表示しない。

---

### 28.44 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、API共通の利用者コンテキストエラーとして扱う。

---

### 28.45 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択されている利用者が有効ではない状態として共通処理する。

---

### 28.46 INVALID_ASSET_ACCOUNT_ID

通常の画面遷移では、ACC-001やACC-003から取得した有効なIDを使用するため、発生頻度は低い。

発生した場合は、不正なURLまたは画面状態として扱う。

---

### 28.47 ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND

現在の利用可能資産設定が存在しない場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

となる。

これは、フォーム入力による問題ではなくサーバー側データ不整合である。

画面では、一般的な更新失敗として扱う。

例えば、

```text
資産口座を更新できませんでした。
```

と表示する。

---

### 28.48 ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT

現在有効な利用可能資産設定が複数存在する場合も、サーバー側データ不整合として扱う。

フロントエンドで利用可能資産設定を独自に選択して更新処理を継続しない。

---

### 28.49 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
資産口座を更新できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

### 28.50 エラー時のフォーム値

ACC-004が失敗した場合は、原則として利用者が入力したフォーム値を保持する。

入力内容を自動的に初期値へ戻さない。

これにより、入力内容を修正して再実行できるようにする。

---

### 28.51 ASSET_ACCOUNT_NOT_FOUND時のフォーム値

ただし、

```text
ASSET_ACCOUNT_NOT_FOUND
```

の場合は、対象資産口座自体が編集不能な可能性がある。

そのため、一覧へ戻すなど画面遷移を優先してよい。

---

### 28.52 サーバー状態との競合

Phase1では`version`や`If-Match`を使用しないため、編集画面表示後に別リクエストで同じ資産口座が更新されていても、ACC-004は更新競合エラーを返さない。

同一項目を更新した場合は、Last Write Winsとなる可能性がある。

フロントエンドでも独自の競合判定を実装しない。

---

### 28.53 updatedAtを送信しない

競合判定用として

```typescript
updatedAt: string;
```

をACC-004のRequestへ追加しない。

Phase1では、更新時刻比較による楽観ロックを採用しないためである。

---

### 28.54 balanceRecordingUnit変更UIを設けない

ACC-004では残高記録単位を変更できないため、編集画面に

```text
ACCOUNT
HOLDING
```

を切り替える入力UIを設けない。

表示する場合は読み取り専用とする。

---

### 28.55 利用可能資産変更UIとの分離

資産口座編集画面に利用可能資産変更機能を同時に配置する場合でも、内部的にはACC-004とは別操作として扱う。

概念的には、

```text
基本情報保存
    ↓
ACC-004

利用可能資産区分変更
    ↓
ACC-006
```

とする。

1つのACC-004 Requestへまとめない。

---

### 28.56 無効化UIとの分離

無効化ボタンを同じ画面へ配置する場合でも、

```text
通常属性更新
    → ACC-004

無効化
    → ACC-005
```

とAPIを分離する。

例えば、「保存」ボタンと「無効化」ボタンを別操作として扱う。

---

### 28.57 Pageの責務

編集Pageでは、主に以下を担当する。

- URLから`assetAccountId`取得
- ACC-003による初期データ取得
- 編集フォーム表示
- ACC-004成功後の画面遷移
- ページ単位のエラー表示

HTTP通信の詳細を直接記述しない。

---

### 28.58 Form Componentの責務

Form Componentでは、主に以下を担当する。

- `name`入力
- `assetType`入力
- クライアント側入力チェック
- フィールドエラー表示
- submitイベント通知

ACC-004のHTTP通信を直接実行しない構成としてよい。

---

### 28.59 Mutation Hookの責務

Mutation Hookでは、主に以下を担当する。

```text
ACC-004実行
成功処理
Query Cache無効化
```

画面固有の表示レイアウトをMutation Hookへ持たせない。

---

### 28.60 API Clientの責務

API Clientでは、

```http
PATCH /api/v1/asset-accounts/{assetAccountId}
```

のHTTP通信と型付きレスポンス取得を担当する。

画面遷移やToast表示などはAPI Clientへ含めない。

---

### 28.61 概念的なディレクトリ構成

例えば、以下のように機能単位で整理できる。

```text
features/
└── asset-accounts/
    ├── api/
    │   ├── getAssetAccountDetail.ts
    │   └── updateAssetAccount.ts
    ├── components/
    │   └── AssetAccountForm.tsx
    ├── hooks/
    │   ├── useAssetAccountDetail.ts
    │   └── useUpdateAssetAccount.ts
    ├── types/
    │   └── assetAccount.ts
    └── pages/
        └── EditAssetAccountPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

### 28.62 フロントエンドで行わないこと

ACC-004のReact・TypeScript実装では、以下をフロントエンドの責務としない。

- 利用者境界の最終保証
- 資産口座存在確認の最終保証
- 論理削除判定
- 資産口座名一意性の最終保証
- DBのUNIQUE競合判定
- DB内部コードへの変換
- 現在利用可能資産設定の期間判定
- データ不整合の補正
- `balanceRecordingUnit`変更
- `startYearMonth`変更
- `isAvailable`変更
- `isEnabled`変更
- 楽観ロック判定

フロントエンドは、

```text
現在値取得
    ↓
編集可能項目の入力
    ↓
変更Request生成
    ↓
ACC-004実行
    ↓
結果表示
```

に責務を限定する。

---

## 29. 設計上の補足

### 29.1 通常属性更新に責務を限定する理由

ACC-004は、資産口座の通常属性を更新するAPIとする。

更新対象は、

```text
name
assetType
```

に限定する。

一方、

```text
balanceRecordingUnit
startYearMonth
isAvailable
isEnabled
```

は扱わない。

概念的には、

```text
ACC-004
    ↓
通常属性更新
    ├─ name
    └─ assetType
```

とする。

資産口座のライフサイクルや履歴管理を伴う操作まで同じAPIへ含めない。

---

### 29.2 PATCHを採用する理由

ACC-004では、資産口座リソース全体を置き換えるのではなく、指定された一部属性だけを更新する。

そのため、

```http
PATCH /api/v1/asset-accounts/{assetAccountId}
```

を使用する。

例えば、

```json
{
  "name": "メイン証券口座"
}
```

の場合は、`name`だけを更新し、

```text
asset_type
balance_recording_unit
start_year_month
```

などの未指定項目を変更しない。

---

### 29.3 PUTを採用しない理由

`PUT`は、リソース全体の置換として解釈されることがある。

ACC-004では、

```text
更新したい項目だけを送信する
```

という部分更新を前提とするため、`PUT`ではなく`PATCH`を採用する。

---

### 29.4 残高記録単位を更新対象外とする理由

`balanceRecordingUnit`は、資産口座登録後の月末資産管理方法を決定する重要な属性である。

例えば、

```text
ACCOUNT
    ↓
HOLDING
```

へ変更すると、

- 既存の月末資産残高
- 保有商品
- 商品別月末評価額

との整合性ルールが必要になる。

そのため、Phase1では登録後の変更を許可しない。

---

### 29.5 利用開始年月を更新対象外とする理由

`startYearMonth`を変更すると、

- 利用可能資産設定履歴
- 月末資産管理対象
- 過去の資産記録
- 対象年月判定

へ影響する。

単純な属性変更ではなく、既存履歴の意味まで変化する可能性があるため、ACC-004では更新しない。

---

### 29.6 利用可能資産区分を更新対象外とする理由

利用可能資産区分は、対象年月によって変化する履歴情報として管理する。

そのため、

```text
asset_accounts.is_available
```

のような通常属性として更新しない。

変更は、

```text
ACC-006
利用可能資産設定登録
```

で扱う。

概念的には、

```text
ACC-004
    → 通常属性

ACC-006
    → 時系列設定
```

と責務を分離する。

---

### 29.7 無効化を更新対象外とする理由

資産口座無効化は、

```text
現在利用中
    ↓
今後の管理対象外
```

というライフサイクル変更である。

単なる通常属性更新とは業務上の意味が異なるため、

```text
ACC-005
資産口座無効化
```

へ分離する。

ACC-004のRequestへ

```json
{
  "isEnabled": false
}
```

のような状態指定を追加しない。

---

### 29.8 userIdを更新対象にしない理由

資産口座の所有利用者は、登録時に確定する。

ACC-004から

```text
User A
    ↓
User B
```

へ資産口座を移管する機能は提供しない。

そのため、

```text
userId
user_id
```

は更新対象に含めない。

---

### 29.9 更新対象項目を明示的に限定する理由

ACC-004では、Request全体をそのままEloquentへ渡さない。

例えば、

```php
$assetAccount->update(
    $request->all(),
);
```

のような実装は避ける。

更新対象を

```text
name
asset_type
```

へ明示的に限定することで、

- `user_id`
- `deleted_at`
- `balance_recording_unit`
- `start_year_month`

などの意図しない変更を防止する。

---

### 29.10 空PATCHを許可しない理由

以下のような更新項目が存在しないRequestは受け付けない。

```json
{}
```

ACC-004は資産口座を更新するためのAPIであり、変更対象が1件もないRequestは業務上意味を持たないためである。

そのため、

```text
name
または
assetType
```

の少なくとも1項目を必須とする。

---

### 29.11 未指定項目とNULLを区別する理由

PATCHでは、

```text
項目未指定
```

と

```text
項目にNULLを指定
```

は異なる意味を持つ。

ACC-004では、

```text
未指定
    → 変更しない

NULL指定
    → バリデーションエラー
```

とする。

例えば、

```json
{
  "assetType": "SECURITIES"
}
```

では、`name`を変更しない。

一方、

```json
{
  "name": null
}
```

は不正とする。

---

### 29.12 同一値更新を許可する理由

現在値と同じ値が指定された場合でも、業務エラーとはしない。

例えば、

```text
現在
name = 証券口座
```

に対して、

```json
{
  "name": "証券口座"
}
```

を送信しても正常なRequestとして扱う。

フロントエンド側では変更なしとして送信を省略してよいが、サーバー側でも同一値指定をエラーにしない。

---

### 29.13 NO_CHANGESエラーを設けない理由

同一値が指定された場合に、

```text
NO_CHANGES
ALREADY_UPDATED
```

などの専用業務エラーは定義しない。

同一値更新は有害な操作ではなく、最終業務状態も変化しないためである。

---

### 29.14 同一利用者内で資産口座名を一意とする理由

資産口座名は、画面やCSVなどで利用者が資産口座を識別するために使う。

同一利用者に

```text
証券口座
証券口座
```

が存在すると、利用者から見て識別しにくくなる。

そのため、

```text
user_id
+
name
```

を一意とする。

---

### 29.15 更新対象自身を重複判定から除外する理由

更新対象自身の現在の名前と同じ名前を指定した場合まで重複とすると、正常な同一値更新ができない。

そのため、重複確認では、

```text
id != assetAccountId
```

を条件へ含める。

---

### 29.16 論理削除済み資産口座も重複対象とする理由

ACC-002と同じく、論理削除済み資産口座も同名重複対象とする。

これにより、過去に存在した資産口座と現在の資産口座が同じ名前になることを防止する。

履歴参照やCSVなどで名称による識別が曖昧になることを避ける。

---

### 29.17 異なる利用者間では同名を許可する理由

資産口座名の一意性は、利用者ごとに保証すればよい。

例えば、

```text
User A
    証券口座

User B
    証券口座
```

は問題ない。

そのため、システム全体で資産口座名を一意にはしない。

---

### 29.18 DBのUNIQUE制約も使用する理由

アプリケーション側で更新前に重複確認をしても、並行更新では競合を完全に防止できない。

例えば、

```text
Request A
重複なし確認

Request B
重複なし確認

Request A
UPDATE

Request B
UPDATE
```

となる可能性がある。

そのため、

```text
user_id
+
name
```

のUNIQUE制約をDB側にも設定する。

---

### 29.19 アプリケーション側でも重複確認する理由

DB制約だけに依存すると、すべての同名エラーがDB例外として発生する。

通常ケースでは、事前に業務上の重複を判定し、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

として分かりやすく返却する。

DB制約は、同時更新時などの最終防衛線とする。

---

### 29.20 `lockForUpdate()`を使用しない理由

同一資産口座を行ロックしても、別々の資産口座を同じ名前へ変更する競合は防止できない。

例えば、

```text
Asset Account A
    lock

Asset Account B
    lock
```

は、別行なので双方取得できる。

名前一意性はUNIQUE制約で保証するため、Phase1では`lockForUpdate()`を使用しない。

---

### 29.21 利用者行をロックしない理由

同一利用者の更新をすべて直列化するために

```text
users
```

の行をロックする方式は採用しない。

資産口座更新以外の同一利用者処理まで不要に待機させるためである。

---

### 29.22 Last Write Winsを許容する理由

Phase1では、同一資産口座への同時編集を高頻度利用ケースとして想定しない。

そのため、同じ属性に対する競合更新についてはLast Write Winsとなる可能性を許容する。

例えば、

```text
Request A
name = A

Request B
name = B
```

では、最終的に後から反映された値が残る。

---

### 29.23 楽観ロックを採用しない理由

Phase1では、

```text
version
updatedAt
ETag
If-Match
```

などの競合検出機構を導入しない。

これらを導入すると、

- version管理
- Requestへの競合情報追加
- 409等の競合エラー
- React側の競合解決UI

が必要になる。

Phase1の利用規模に対して複雑性が大きいため、将来課題とする。

---

### 29.24 明示的なトランザクションを必須としない理由

ACC-004で更新する業務データは、

```text
asset_accounts 1件
```

だけである。

関連テーブルとの複数更新を行わないため、Phase1では

```php
DB::transaction()
```

を必須としない。

単一UPDATEの原子性はDBが保証する。

---

### 29.25 利用可能資産設定を更新前に確認する理由

ACC-004の成功レスポンスでは、

```text
isAvailable
```

も返却する。

そのため、現在有効な利用可能資産設定に不整合がある状態で資産口座だけ更新してから500エラーとなる構成は避ける。

概念的には、

```text
現在設定確認
    ↓
正常
    ↓
asset_accounts更新
```

とする。

---

### 29.26 利用可能資産設定の0件を補完しない理由

現在有効な設定が存在しない場合に、

```text
isAvailable = false
```

として正常レスポンスを返すと、データ不整合を隠蔽する。

ACC-002で初期設定を必ず作成する設計のため、0件は正常状態とはしない。

---

### 29.27 利用可能資産設定の複数件を補正しない理由

現在有効な設定が複数存在する場合に、

```text
最新1件を採用
```

すると、期間重複を隠蔽する。

ACC-004ではデータ修復を行わず、データ不整合として扱う。

---

### 29.28 ACC-004で利用可能資産設定を更新しない理由

成功レスポンス生成のために

```text
asset_account_available_settings
```

を参照するが、更新は行わない。

参照することと変更責務を持つことは別である。

利用可能資産設定の変更は、ACC-006へ限定する。

---

### 29.29 assetType変更時に関連データを自動変更しない理由

資産種別を変更しても、

- 保有商品
- 月末残高
- 商品別月末評価額
- 利用可能資産設定

を自動変更しない。

`assetType`は資産口座の分類情報であり、ACC-004では関連データの構造変更まで行わない。

---

### 29.30 保有商品を更新しない理由

資産口座と保有商品は、別リソースとして扱う。

そのため、資産口座の名前や資産種別を変更しても、

```text
holding_assets
```

を更新しない。

保有商品の変更はHLD APIの責務とする。

---

### 29.31 月末資産データを更新しない理由

資産口座の通常属性変更によって、過去に記録した

```text
month_end_asset_balances
month_end_holding_values
month_end_asset_snapshots
```

を変更しない。

過去時点の資産記録そのものは維持する。

---

### 29.32 Repositoryを使用する理由

ACC-004では、実際に

```text
asset_accounts
```

を更新する。

そのため、

```text
Query
    → 読み取り

Repository
    → 書き込み
```

という責務分離方針に従い、更新処理をRepositoryへ置く。

---

### 29.33 QueryとRepositoryを分ける理由

ACC-004では、

```text
AssetAccountQuery
    → 更新対象取得
    → 名前重複確認

AssetAccountRepository
    → asset_accounts更新
```

と分離する。

これにより、検索条件と状態変更処理が混在することを防止する。

---

### 29.34 UseCaseを設ける理由

ACC-004では、

```text
資産口座取得
    ↓
現在設定確認
    ↓
名前重複確認
    ↓
部分更新
    ↓
結果生成
```

という業務処理の流れを持つ。

Actionへこれらを直接記述せず、UseCaseへオーケストレーションを集約する。

---

### 29.35 FormRequestを使用する理由

ACC-004には、

```text
name
assetType
```

というRequest Bodyが存在する。

さらに、

```text
PATCHの未指定項目
NULL禁止
Enum
最大文字数
最低1項目
```

などのHTTP入力検証が必要になる。

そのため、専用FormRequestを使用する。

---

### 29.36 PATCHのInput DTOを使う理由

PATCHでは、

```text
未指定
```

と

```text
入力済み
```

の区別が必要になる。

そのため、ActionからUseCaseへRequestオブジェクトそのものを渡さず、更新対象を明確にしたInput DTOへ変換してよい。

---

### 29.37 DTOでNULLを未指定として使用する際の注意

ACC-004では、APIとしてNULL更新を許可しない。

そのため、Input DTO上の

```text
?string
?AssetType
```

の`null`は、

```text
Request項目未指定
```

という意味だけに使用する。

入力値として送信されたJSONの`null`とはFormRequestで区別する。

---

### 29.38 Mass Assignmentを避ける理由

ACC-004では更新対象が明確に限定されている。

Request配列をそのままEloquentへ渡すより、

```text
name指定あり
    → name変更

assetType指定あり
    → asset_type変更
```

と明示的に処理することで、更新範囲をコード上でも明確にできる。

---

### 29.39 EloquentのDirty判定を利用してよい理由

現在値と同一値の場合、Eloquentは変更なしと判定できる。

この場合に、無理にUPDATEを発行する必要はない。

そのため、

```text
同一値
    ↓
save()
    ↓
Dirtyなし
```

というLaravel標準動作を利用してよい。

---

### 29.40 `updated_at`をAPIへ返さない理由

Phase1では、`updated_at`を楽観ロックや画面表示要件に使用しない。

そのため、ACC-004成功レスポンスへ`updatedAt`を含めない。

DB内部の管理日時として扱う。

---

### 29.41 成功レスポンスをACC-003と揃える理由

ACC-004成功後にクライアントが必要とするのは、更新後の資産口座状態である。

そのため、ACC-003と同じ意味を持つ

```text
id
name
assetType
balanceRecordingUnit
isAvailable
startYearMonth
isEnabled
```

を返却する。

これにより、React側で共通型を利用しやすくする。

---

### 29.42 ACC-004で更新していない値も返す理由

ACC-004で直接変更するのは、

```text
name
assetType
```

だけである。

ただし、成功レスポンスを資産口座詳細として完結させるため、

```text
balanceRecordingUnit
isAvailable
startYearMonth
isEnabled
```

も返却する。

これにより、クライアントが更新後の資産口座状態を1つのレスポンスで確認できる。

---

### 29.43 `isEnabled = true`のみ返る理由

ACC-004では、論理削除済み資産口座を更新できない。

そのため、正常レスポンスでは

```text
isEnabled = true
```

となる。

`false`を返すケースはACC-004の正常系には存在しない。

---

### 29.44 `deletedAt`を返さない理由

SoftDeletesの

```text
deleted_at
```

はDB内部実装である。

APIでは業務上の状態として

```text
isEnabled
```

を返却する。

---

### 29.45 Optimistic Updateを必須としない理由

ACC-004では、

- 名前重複
- UNIQUE制約競合
- 利用者境界
- 論理削除
- 利用可能資産設定不整合

など、サーバー側で最終判定する条件がある。

そのため、Phase1では先に画面を更新するOptimistic Updateを必須としない。

---

### 29.46 Query invalidateを基本とする理由

ACC-004成功後は、

```text
ACC-001
資産口座一覧

ACC-003
資産口座詳細
```

の表示内容が変化する可能性がある。

そのため、

```text
Mutation成功
    ↓
Query invalidate
    ↓
最新状態再取得
```

を基本とする。

---

### 29.47 成功レスポンスをQuery Cacheへ直接設定できる理由

ACC-004成功レスポンスは、ACC-003と同じ詳細表現を返す。

そのため、

```text
ACC-004 response
    ↓
ACC-003 detail cache
```

へ直接設定することも可能である。

ただし、Phase1では実装の単純さを優先してinvalidate方式を基本としてよい。

---

### 29.48 Mutationの自動Retryを行わない理由

ACC-004は更新APIである。

同一入力は最終状態として冪等だが、通信結果不明時に自動再送すると、ユーザーから見た処理状況が分かりにくくなる。

Phase1ではMutationの自動Retryを原則無効とする。

---

### 29.49 Idempotency-Keyを使用しない理由

ACC-004は、

```text
指定属性を
指定値へ更新する
```

処理であり、履歴レコード追加や加算処理ではない。

同一入力の再実行で業務状態が増殖しないため、Phase1ではIdempotency-Keyを使用しない。

---

### 29.50 ACC-003を編集画面初期値に使用する理由

ACC-004専用に「編集用データ取得API」を追加しない。

ACC-003で現在の資産口座詳細を取得し、その中から

```text
name
assetType
```

を編集フォーム初期値へ使用する。

API数を不要に増やさない。

---

### 29.51 編集不可項目を画面で読み取り専用にする理由

ACC-003では、

```text
balanceRecordingUnit
startYearMonth
isAvailable
isEnabled
```

も取得できる。

ただし、これらをACC-004で変更できるようにはしない。

同じ画面に表示する場合でも、

```text
表示
    ≠
編集可能
```

として扱う。

---

### 29.52 ACC-005との責務分離

資産口座無効化は、ACC-004に含めない。

概念的には、

```text
保存ボタン
    ↓
ACC-004

無効化ボタン
    ↓
ACC-005
```

とする。

同じ編集画面に両操作を配置することはできるが、APIとしては分離する。

---

### 29.53 ACC-006との責務分離

利用可能資産区分の変更も、ACC-004へ含めない。

画面上で基本情報と利用可能資産区分を同時に見せる場合でも、

```text
基本情報変更
    → ACC-004

利用可能資産設定変更
    → ACC-006
```

として別Mutationにする。

---

### 29.54 HLD APIとの責務分離

資産口座が`HOLDING`管理であっても、ACC-004で保有商品を更新しない。

保有商品の

- 名前
- 商品種別
- 備考
- 無効化

などはHLD APIで扱う。

---

### 29.55 Phase1では更新履歴を保存しない

ACC-004による

```text
name
assetType
```

の変更履歴を専用履歴テーブルへ保存しない。

Phase1では、現在値のみを`asset_accounts`へ保持する。

将来的に監査要件が生じた場合は、変更履歴テーブルや監査ログの導入を検討する。

---

### 29.56 Phase1では設計を広げすぎない

ACC-004では、以下の機能を追加しない。

- 残高記録単位変更
- 利用開始年月変更
- 利用可能資産区分変更
- 資産口座無効化
- 資産口座再有効化
- 所有利用者変更
- 更新履歴保存
- 楽観ロック
- ETag
- Idempotency-Key
- 複数資産口座一括更新

Phase1では、

```text
name
+
assetType
```

の通常属性更新へ責務を限定する。

---

## 30. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)