# ACC-006 利用可能資産設定登録

## 1. 概要

操作対象となる利用者に帰属する指定された資産口座について、新しい利用可能資産設定を登録する。

ACC-006では、現在有効な利用可能資産設定を終了し、新しい設定を追加することで、利用可能資産区分の変更履歴を保持する。

概念的には、以下のように処理する。

```text
現在設定
2026-01 ～ NULL
isAvailable = true
    ↓
ACC-006
startYearMonth = 2026-08
isAvailable = false
    ↓
旧設定
2026-01 ～ 2026-07
isAvailable = true

新設定
2026-08 ～ NULL
isAvailable = false
```

ACC-006は、単純に現在設定の

```text
is_available
```

を上書きするAPIではない。

利用可能資産区分を年月単位の履歴として保持するため、

```text
旧設定の終了
+
新設定の登録
```

を1つの業務操作として扱う。

---

### 1.1 更新対象

ACC-006では、以下のテーブルを更新する。

```text
asset_account_available_settings
```

主な更新内容は、以下とする。

```text
現在設定
    end_year_month更新

新設定
    INSERT
```

一方、以下はACC-006では更新しない。

```text
asset_accounts.name
asset_accounts.asset_type
asset_accounts.balance_recording_unit
asset_accounts.start_year_month
asset_accounts.deleted_at
```

---

### 1.2 ACC-004との違い

ACC-004 資産口座更新は、資産口座の通常属性を更新する。

更新対象は、

```text
name
assetType
```

とする。

一方、ACC-006では、

```text
isAvailable
```

に相当する利用可能資産区分を履歴として変更する。

概念的には、

```text
ACC-004
    ↓
通常属性更新

ACC-006
    ↓
利用可能資産設定履歴変更
```

と責務を分離する。

---

### 1.3 ACC-005との違い

ACC-005は、利用可能資産設定履歴を取得する参照APIである。

ACC-006は、新しい利用可能資産設定を登録する更新APIである。

概念的には、

```text
ACC-005
    → 履歴取得

ACC-006
    → 履歴追加
```

とする。

---

### 1.4 現在設定を直接UPDATEしない理由

例えば、現在の設定が

```text
2026-01 ～ NULL
isAvailable = true
```

の場合に、

```text
is_available = false
```

へ直接変更すると、2026-01以降がすべて`false`だったことになり、過去状態を再現できなくなる。

そのため、

```text
旧設定
    ↓
終了年月を設定

新設定
    ↓
新しい状態を登録
```

とする。

---

### 1.5 新規設定の意味

ACC-006で登録する新しい設定は、

```text
startYearMonth
```

以降に適用される利用可能資産区分を表す。

例えば、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

を登録した場合は、

```text
2026-08以降
利用可能資産として扱わない
```

という意味になる。

---

## 2. ユースケース

利用者は、資産口座を目的達成判定などで利用可能資産として扱うかどうかを変更したい場合にACC-006を使用する。

主な利用例は、以下とする。

- 現在利用可能な資産口座を利用対象外へ変更する
- 現在利用対象外の資産口座を利用可能へ変更する
- 将来の年月から利用可能資産区分を変更する
- 資産口座詳細画面から設定変更する

概念的な画面利用は、以下とする。

```text
ACC-003
資産口座詳細取得
    ↓
現在のisAvailable確認
    ↓
ACC-005
利用可能資産設定履歴確認
    ↓
利用者が設定変更
    ↓
ACC-006
利用可能資産設定登録
    ↓
ACC-003 / ACC-005再取得
```

---

### 2.1 利用可能から利用対象外へ変更

例えば、現在の設定が

```text
2026-01 ～ NULL
isAvailable = true
```

で、2026-08から利用対象外へ変更する場合は、

```text
旧設定
2026-01 ～ 2026-07
true

新設定
2026-08 ～ NULL
false
```

となる。

---

### 2.2 利用対象外から利用可能へ変更

現在、

```text
2026-01 ～ NULL
isAvailable = false
```

の場合に、2026-08から利用可能へ変更すると、

```text
旧設定
2026-01 ～ 2026-07
false

新設定
2026-08 ～ NULL
true
```

となる。

---

### 2.3 過去履歴を直接編集しない

ACC-006では、既存の過去履歴を指定して

```text
過去設定のisAvailable変更
```

を行わない。

設定変更は、

```text
現在設定終了
+
新設定登録
```

として行う。

---

## 3. エンドポイント

```http
POST /api/v1/asset-accounts/{assetAccountId}/available-settings
```

`assetAccountId`には、利用可能資産設定を追加する資産口座IDを指定する。

例：

```http
POST /api/v1/asset-accounts/10/available-settings
```

---

### 3.1 資産口座配下のリソースとする理由

利用可能資産設定は、特定の資産口座に紐づく履歴リソースである。

そのため、

```text
asset-accounts
    ↓
available-settings
```

という親子関係をURLへ表現する。

トップレベルの

```http
POST /api/v1/asset-account-available-settings
```

とはしない。

---

### 3.2 ACC-005とのURL関係

ACC-005とACC-006は、同じリソースURLを使用し、HTTPメソッドによって責務を分離する。

```text
GET
/api/v1/asset-accounts/{assetAccountId}/available-settings
    → ACC-005
      利用可能資産設定履歴取得

POST
/api/v1/asset-accounts/{assetAccountId}/available-settings
    → ACC-006
      利用可能資産設定登録
```

---

## 4. HTTPメソッド

```text
POST
```

ACC-006では、新しい

```text
asset_account_available_settings
```

レコードを追加するため、`POST`を使用する。

---

### 4.1 PATCHを使用しない理由

ACC-006は、既存の1レコードを単純更新する操作ではない。

概念的には、

```text
旧設定
    UPDATE

+
新設定
    INSERT
```

となる。

業務上は、新しい利用可能資産設定を登録する操作であるため、`POST`とする。

---

### 4.2 既存設定IDをURLに含めない理由

ACC-006では、現在設定のIDをクライアントに指定させない。

以下のようなURLにはしない。

```http
PATCH /api/v1/asset-account-available-settings/{settingId}
```

現在有効な設定は、サーバー側で対象資産口座の履歴から特定する。

これにより、クライアントが誤った履歴レコードを直接更新することを防止する。

---

### 4.3 副作用

ACC-006成功時は、通常、

```text
asset_account_available_settings

旧設定
    1件UPDATE

新設定
    1件INSERT
```

という2つの状態変更が発生する。

そのため、これらは同一トランザクション内で処理する。

具体的なトランザクション境界は、後続で定義する。

---

## 5. 利用者コンテキスト

本APIは利用者依存APIのため、`X-User-Id`を必須とする。

操作対象となる利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者コンテキストの特定、`X-User-Id`の検証および利用者境界については、[API共通方針](../api-common-policy.md)に従う。

指定された`assetAccountId`に対応する資産口座が操作対象利用者に帰属する場合のみ、利用可能資産設定を登録できる。

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

ACC-006では、`assetAccountId`だけを条件として資産口座を取得してはならない。

まず、

```text
assetAccountId
+
操作対象利用者ID
+
利用中状態
```

によって対象資産口座を確定する。

その後、対象資産口座に紐づく

```text
asset_account_available_settings
```

を更新する。

概念的には、

```text
X-User-Id
    ↓
UserContext
    ↓
asset_accounts
    ↓
利用者境界確認
    ↓
asset_account_available_settings
    ↓
設定変更
```

とする。

---

### 5.2 他利用者の資産口座

指定された`assetAccountId`が他利用者に帰属する場合は、対象資産口座が存在しないものとして扱う。

概念的には、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

とする。

以下のような専用エラーは返却しない。

```text
FORBIDDEN
OTHER_USER_ASSET_ACCOUNT
```

他利用者の資産口座存在有無を外部から推測しにくくするためである。

---

### 5.3 論理削除済み資産口座

ACC-006では、利用中の資産口座だけを設定変更対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座には、新しい利用可能資産設定を登録しない。

論理削除済み資産口座を指定した場合も、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 5.4 利用可能資産設定にはuser_idを持たせない

```text
asset_account_available_settings
```

には、直接

```text
user_id
```

を保持しない。

利用者境界は、親となる

```text
asset_accounts.user_id
```

を通して保証する。

概念的には、

```text
users
    ↓
asset_accounts.user_id
    ↓
asset_accounts.id
    ↓
asset_account_available_settings.asset_account_id
```

という関係とする。

---

### 5.5 settingIdをクライアントから受け付けない

ACC-006では、現在有効な利用可能資産設定IDをリクエストから受け付けない。

例えば、

```json
{
  "currentSettingId": "15",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

のような形式にはしない。

現在設定は、対象資産口座の履歴からサーバー側で特定する。

---

### 5.6 assetAccountIdをリクエストボディから受け付けない

対象資産口座は、URLの

```text
assetAccountId
```

で指定する。

そのため、Request Bodyへ

```json
{
  "assetAccountId": "10"
}
```

のような値を持たせない。

資産口座指定経路を複数にしない。

---

### 5.7 userIdをリクエストボディから受け付けない

以下のような指定によって別利用者の資産口座へ設定を追加できない。

```json
{
  "userId": "999",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

利用者は、

```text
X-User-Id
    ↓
UserContext
```

によってのみ確定する。

---

### 5.8 他利用者の設定を更新しない

例えば、

```text
User A
    Asset Account A
        Setting A

User B
    Asset Account B
        Setting B
```

という状態で、User AとしてAsset Account Bを指定した場合は、

```text
Setting B
```

を取得・更新しない。

資産口座の利用者境界確認時点で、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として処理を終了する。

---

### 5.9 X-User-Idが不正な場合

以下の場合は、資産口座検索や利用可能資産設定更新へ進まない。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正
- 指定された利用者が存在しない
- 指定された利用者が論理削除済み

この場合、

```text
asset_account_available_settings
```

を更新しない。

利用者コンテキストに関するエラーコードおよびHTTPステータスは、API共通方針に従う。

---

### 5.10 1リクエスト1利用者

1回のACC-006リクエストでは、`X-User-Id`で指定された1利用者の資産口座だけを扱う。

以下のように、URLやクエリパラメータから別利用者を指定する方式は採用しない。

```text
/api/v1/users/{userId}/asset-accounts/{assetAccountId}/available-settings
```

または、

```text
?userId=1
```

利用者の指定経路は、`X-User-Id`へ統一する。

---

## 6. パスパラメータ

本APIでは、利用可能資産設定を登録する対象資産口座を指定するために、以下のパスパラメータを使用する。

| パラメータ            | 型      |  必須 | 説明                  |
| ---------------- | ------ | :-: | ------------------- |
| `assetAccountId` | string |  ○  | 利用可能資産設定を登録する資産口座ID |

エンドポイントは、以下とする。

```http
POST /api/v1/asset-accounts/{assetAccountId}/available-settings
```

---

### 6.1 assetAccountId

`assetAccountId`には、利用可能資産設定を変更する資産口座IDを指定する。

例：

```http
POST /api/v1/asset-accounts/10/available-settings
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

実際の資産口座取得では、操作対象利用者IDと組み合わせて検索する。

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

この条件を満たす資産口座に対してのみ、利用可能資産設定を登録できる。

---

## 7. クエリパラメータ

なし。

ACC-006では、対象資産口座は`assetAccountId`で指定し、新しい利用可能資産設定はリクエストボディで指定する。

以下のようなクエリパラメータは使用しない。

* `userId`
* `settingId`
* `currentSettingId`
* `isAvailable`
* `startYearMonth`
* `force`

---

## 8. リクエストヘッダー

以下のリクエストヘッダーを使用する。

| ヘッダー名          |  必須 | 説明                 |
| -------------- | :-: | ------------------ |
| `X-User-Id`    |  ○  | 操作対象となる利用者ID       |
| `Content-Type` |  ○  | `application/json` |
| `Accept`       |  ○  | `application/json` |

リクエスト例：

```http
POST /api/v1/asset-accounts/10/available-settings
Content-Type: application/json
Accept: application/json
X-User-Id: 1
```

---

### 8.1 X-User-Id

`X-User-Id`は、API共通方針に従って検証する。

本APIの処理開始前に、共通Middlewareで以下を確認する。

* ヘッダーが指定されていること
* IDが共通方針で定めた形式であること
* 利用者が存在すること
* 利用者が論理削除されていないこと

検証済み利用者IDを利用者コンテキストへ保持し、Action以降で使用する。

---

## 9. リクエストボディ

ACC-006では、新しく適用する利用可能資産設定を指定する。

リクエスト項目は、以下とする。

* `startYearMonth`
* `isAvailable`

リクエスト例：

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

このリクエストは、

```text
2026-08から
利用可能資産として扱わない
```

という意味になる。

---

### 9.1 リクエスト項目

| 項目               | 型       |  必須 | NULL | 説明                       |
| ---------------- | ------- | :-: | :--: | ------------------------ |
| `startYearMonth` | string  |  ○  |   ×  | 新しい設定の適用開始年月。`YYYY-MM`形式 |
| `isAvailable`    | boolean |  ○  |   ×  | 利用可能資産として扱うか             |

---

### 9.2 startYearMonth

新しい利用可能資産設定を適用開始する年月を指定する。

形式は、

```text
YYYY-MM
```

とする。

正常例：

```text
2026-08
2027-01
2030-12
```

不正例：

```text
2026-8
2026/08
202608
2026-13
2026-00
abc
```

---

### 9.3 isAvailable

新しい設定期間で、対象資産口座を利用可能資産として扱うかをbooleanで指定する。

利用可能にする場合：

```json
{
  "isAvailable": true
}
```

利用対象外にする場合：

```json
{
  "isAvailable": false
}
```

以下のような文字列や数値は受け付けない。

```json
{
  "isAvailable": "true"
}
```

```json
{
  "isAvailable": 1
}
```

---

### 9.4 endYearMonthを受け付けない

新しい設定の

```text
endYearMonth
```

は、クライアントから指定させない。

ACC-006で登録する新設定は、現在継続中の設定として

```text
end_year_month = NULL
```

で登録する。

終了年月は、次回ACC-006で新しい設定を登録する際にサーバー側で確定する。

---

### 9.5 現在設定IDを受け付けない

以下のような`settingId`は受け付けない。

```json
{
  "currentSettingId": "15",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

現在有効な設定は、サーバー側で対象資産口座の履歴から特定する。

---

### 9.6 assetAccountIdを受け付けない

Request Bodyへ、

```json
{
  "assetAccountId": "10"
}
```

を含めない。

対象資産口座は、URLの

```text
assetAccountId
```

によって指定する。

---

### 9.7 userIdを受け付けない

Request Bodyへ、

```json
{
  "userId": "1"
}
```

を含めない。

操作対象利用者は、`X-User-Id`から特定する。

---

### 9.8 更新対象外項目

以下の項目は、ACC-006では受け付けない。

* `id`
* `userId`
* `user_id`
* `assetAccountId`
* `asset_account_id`
* `settingId`
* `currentSettingId`
* `endYearMonth`
* `end_year_month`
* `createdAt`
* `created_at`
* `updatedAt`
* `updated_at`

未定義項目を拒否するAPI共通方針を採用する場合は、これらを含むRequestをバリデーションエラーとする。

---

## 10. バリデーション

ACC-006では、主に以下を検証する。

```text
X-User-Id
assetAccountId
startYearMonth
isAvailable
```

加えて、入力形式検証後に以下の業務条件を確認する。

```text
対象資産口座存在
現在設定存在
履歴整合性
startYearMonthの妥当性
現在設定との差分
```

業務上の詳細判定は、後続の「業務ルール」で定義する。

---

### 10.1 X-User-Id

`X-User-Id`について、以下を検証する。

* 必須であること
* 共通ID形式に一致すること
* 正の整数として扱えること
* 指定された利用者が存在すること
* 指定された利用者が論理削除されていないこと

不正な場合は、資産口座検索や設定更新へ進まない。

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

形式不正の場合は、資産口座検索へ進まない。

---

### 10.3 startYearMonth 必須

`startYearMonth`は必須とする。

以下のように未指定の場合は不正とする。

```json
{
  "isAvailable": false
}
```

---

### 10.4 startYearMonth NULL

`startYearMonth`に`null`を指定することはできない。

```json
{
  "startYearMonth": null,
  "isAvailable": false
}
```

は不正とする。

---

### 10.5 startYearMonth形式

`startYearMonth`は、厳密な

```text
YYYY-MM
```

形式であることを確認する。

例えば、

```text
2026-08
```

は正常とする。

以下は不正とする。

```text
2026-8
26-08
2026/08
202608
2026-13
2026-00
```

---

### 10.6 実在する年月

正規表現で見た目だけを確認するのではなく、実在する年月であることを確認する。

例えば、

```text
2026-13
2026-00
```

は不正とする。

---

### 10.7 isAvailable 必須

`isAvailable`は必須とする。

以下のように未指定の場合は不正とする。

```json
{
  "startYearMonth": "2026-08"
}
```

---

### 10.8 isAvailable NULL

`isAvailable`に`null`を指定することはできない。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": null
}
```

は不正とする。

---

### 10.9 isAvailable型

`isAvailable`は、JSON booleanのみを受け付ける。

正常：

```json
{
  "isAvailable": true
}
```

```json
{
  "isAvailable": false
}
```

不正：

```json
{
  "isAvailable": "true"
}
```

```json
{
  "isAvailable": "false"
}
```

```json
{
  "isAvailable": 1
}
```

```json
{
  "isAvailable": 0
}
```

---

### 10.10 資産口座存在確認

形式検証済みの`assetAccountId`について、操作対象利用者に属する利用中の資産口座が存在することを確認する。

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

### 10.11 他利用者の資産口座

指定された`assetAccountId`が他利用者に属する場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

他利用者に属することを専用エラーで公開しない。

---

### 10.12 論理削除済み資産口座

対象資産口座が

```text
deleted_at IS NOT NULL
```

の場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

ACC-006によって論理削除済み資産口座へ新しい設定を追加しない。

---

### 10.13 利用開始年月より前を許可しない

`startYearMonth`は、対象資産口座の

```text
asset_accounts.start_year_month
```

より前には指定できない。

例えば、

```text
資産口座利用開始年月
2026-08

Request
startYearMonth = 2026-07
```

は不正とする。

---

### 10.14 現在設定開始年月より後を必須とする

ACC-006は、現在有効な設定を終了して新しい設定を追加する操作である。

そのため、新しい`startYearMonth`は、現在設定の

```text
start_year_month
```

より後であることを必須とする。

例えば、

```text
現在設定
startYearMonth = 2026-01
endYearMonth   = NULL
```

の場合、

```text
2026-02
2026-08
2027-01
```

などは候補となる。

一方、

```text
2026-01
2025-12
```

は許可しない。

---

### 10.15 現在設定と同じ開始年月を許可しない

例えば、現在設定が

```text
2026-08 ～ NULL
```

の場合に、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

を送信すると、旧設定の有効期間を正しく終了できない。

そのため、現在設定と同じ`startYearMonth`は不正とする。

---

### 10.16 新設定開始年月の前月を算出できること

ACC-006では、旧設定の終了年月を

```text
新設定.startYearMonthの前月
```

として設定する。

例えば、

```text
新設定
2026-08
```

の場合、

```text
旧設定.endYearMonth
    = 2026-07
```

となる。

そのため、`startYearMonth`は安全に前月計算できる正しい年月値であることを前提とする。

---

### 10.17 年跨ぎ

以下のような年跨ぎも正しく扱う。

```text
新設定.startYearMonth
    = 2027-01

旧設定.endYearMonth
    = 2026-12
```

年月計算を文字列操作だけで実装しない。

---

### 10.18 現在設定の存在確認

ACC-006では、現在継続中の設定として

```text
end_year_month = NULL
```

の履歴が1件存在することを前提とする。

0件の場合は、正常な設定変更を実行できない。

この場合は、業務データ不整合として扱う。

---

### 10.19 現在設定が複数件の場合

以下のように、

```text
end_year_month = NULL
```

の履歴が複数存在する場合も正常状態ではない。

例えば、

```text
2026-01 ～ NULL
true

2026-08 ～ NULL
false
```

のような状態では、どちらを終了させるべきか一意に判断できない。

そのため、ACC-006を実行しない。

---

### 10.20 履歴全体の整合性

ACC-006実行前に、必要に応じて既存履歴全体が正常な状態であることを確認する。

主に以下を確認する。

* 履歴が1件以上存在する
* 初期設定開始年月が資産口座利用開始年月と一致する
* 設定期間が重複していない
* 設定期間に欠落がない
* 継続中設定が1件だけ存在する
* 最新設定が継続中である

不整合状態に新しい履歴を追加して、さらに状態を複雑化しない。

---

### 10.21 同じisAvailableを指定した場合

現在設定と同じ`isAvailable`を指定する場合は、新しい履歴を追加する必要がない。

例えば、

```text
現在設定
isAvailable = true
```

に対して、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

を送信するケースである。

これは状態変更を伴わないため、ACC-006では業務エラーとして扱う。

具体的な独自エラーコードは、後続の「エラーレスポンス」で定義する。

---

### 10.22 同一状態の履歴を追加しない理由

以下のような履歴を作成しない。

```text
2026-01 ～ 2026-07
true

2026-08 ～ NULL
true
```

業務状態が変化していないにもかかわらず履歴を分割すると、設定履歴に不要なレコードが増えるためである。

ACC-006は、利用可能資産区分が実際に変化する場合だけ実行する。

---

### 10.23 過去設定のstartYearMonthを指定しない

ACC-006は、過去履歴を修正するAPIではない。

例えば、

```text
現在設定
2026-08 ～ NULL

Request
startYearMonth = 2026-05
```

のように、現在設定より前の年月を指定して過去履歴を差し替えることはできない。

---

### 10.24 将来年月

Phase1では、現在より将来の年月を`startYearMonth`として指定することを許可してよい。

例えば、現在年月が

```text
2026-08
```

であっても、

```json
{
  "startYearMonth": "2026-10",
  "isAvailable": false
}
```

を登録できる。

この場合、

```text
旧設定
～ 2026-09

新設定
2026-10 ～ NULL
```

となる。

ただし、将来予約を許可するかどうかの最終業務方針は、機能要件に従う。

---

### 10.25 将来設定が既に存在する場合

既存履歴に現在より将来開始の継続中設定が登録されている場合、さらにその前後へ新設定を挿入すると履歴再構成が必要になる可能性がある。

Phase1では、ACC-006を

```text
現在の最新設定を終了し
その後ろへ新設定を追加する
```

操作に限定する。

既存履歴の途中へ設定を挿入する機能は提供しない。

---

### 10.26 更新対象外項目

未定義項目を拒否するAPI共通方針を採用する場合は、以下のようなRequestをバリデーションエラーとする。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false,
  "endYearMonth": "2026-12"
}
```

または、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false,
  "currentSettingId": "10"
}
```

サーバー側で管理する値をクライアントから指定させない。

---

### 10.27 バリデーション失敗時

入力値検証または業務前提条件の確認に失敗した場合は、利用可能資産設定を変更しない。

特に、

```text
asset_account_available_settings
```

に対する

```text
UPDATE
INSERT
```

を途中まで実行しない。

実際の更新は、すべての事前条件を確認した後にトランザクション内で行う。

---

## 11. 業務ルール

ACC-006では、操作対象利用者に帰属する指定された資産口座について、現在の利用可能資産設定を終了し、新しい利用可能資産設定を登録する。

概念的には、

```text
現在設定
2026-01 ～ NULL
isAvailable = true
    ↓
ACC-006
startYearMonth = 2026-08
isAvailable = false
    ↓
旧設定
2026-01 ～ 2026-07
true

新設定
2026-08 ～ NULL
false
```

とする。

ACC-006は、現在設定の`is_available`を直接上書きするAPIではない。

---

### 11.1 操作対象利用者に帰属する資産口座のみ変更できる

対象資産口座は、以下の条件をすべて満たす必要がある。

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

`assetAccountId`だけを条件として対象資産口座を取得しない。

---

### 11.2 他利用者の資産口座は変更できない

指定された`assetAccountId`が他利用者に帰属する場合は、対象不存在として扱う。

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

---

### 11.3 論理削除済み資産口座は変更できない

ACC-006では、利用中の資産口座のみを設定変更対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座には、新しい利用可能資産設定を登録しない。

---

### 11.4 利用可能資産設定は履歴として管理する

利用可能資産区分は、単一の現在値として上書きしない。

以下のような処理は行わない。

```text
現在設定
is_available = true
    ↓
UPDATE
is_available = false
```

これでは、過去時点の状態を再現できなくなるためである。

---

### 11.5 旧設定を終了する

新しい設定を登録する前に、現在継続中の設定の

```text
end_year_month
```

を更新する。

終了年月は、

```text
新設定.startYearMonthの前月
```

とする。

例えば、

```text
新設定.startYearMonth
    = 2026-08
```

の場合、

```text
旧設定.endYearMonth
    = 2026-07
```

とする。

---

### 11.6 新設定を登録する

旧設定終了後、新しい利用可能資産設定を追加する。

新設定は、

```text
start_year_month
    = request.startYearMonth

end_year_month
    = NULL

is_available
    = request.isAvailable
```

とする。

---

### 11.7 新設定は必ず継続中として登録する

ACC-006で登録する新しい設定には、

```text
end_year_month = NULL
```

を設定する。

クライアントから終了年月を指定させない。

終了年月は、次回ACC-006実行時にサーバー側で設定する。

---

### 11.8 現在設定は1件のみ存在する

ACC-006実行前には、現在継続中の設定として

```text
end_year_month = NULL
```

の履歴が1件だけ存在することを正常状態とする。

---

### 11.9 現在設定0件は不整合とする

現在継続中設定が0件の場合は、正常な設定変更を行えない。

例えば、

```text
2026-01 ～ 2026-06
true

2026-07 ～ 2026-12
false
```

しか存在せず、`end_year_month = NULL`の履歴がない状態は不整合とする。

---

### 11.10 現在設定複数件は不整合とする

以下のように、

```text
2026-01 ～ NULL
true

2026-08 ～ NULL
false
```

という状態では、現在設定を一意に判断できない。

そのため、ACC-006を実行しない。

---

### 11.11 既存履歴全体が正常であることを前提とする

ACC-006実行前に、既存の利用可能資産設定履歴が正常であることを確認する。

主に以下を確認する。

- 履歴が1件以上存在する
- 最初の設定開始年月が資産口座利用開始年月と一致する
- 各設定で`startYearMonth <= endYearMonth`
- 設定期間が重複していない
- 設定期間に欠落がない
- `endYearMonth = null`の設定が1件のみ
- 最新設定が継続中である

不整合履歴へ新しい履歴を追加しない。

---

### 11.12 新設定開始年月は現在設定開始年月より後とする

`startYearMonth`は、現在設定の

```text
start_year_month
```

より後であることを必須とする。

例えば、

```text
現在設定
2026-01 ～ NULL
```

の場合、

```text
2026-02
2026-08
2027-01
```

は指定できる。

一方、

```text
2026-01
2025-12
```

は指定できない。

---

### 11.13 現在設定と同じ開始年月は指定できない

例えば、

```text
現在設定
2026-08 ～ NULL
```

に対して、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

を指定することはできない。

旧設定の終了年月を

```text
2026-07
```

とした場合、

```text
startYearMonth > endYearMonth
```

となってしまうためである。

---

### 11.14 過去履歴の途中へ設定を挿入しない

ACC-006は、履歴の末尾へ新しい設定を追加するAPIとする。

例えば、

```text
既存履歴

2026-01 ～ 2026-06
true

2026-07 ～ NULL
false
```

に対して、

```text
startYearMonth = 2026-04
```

の設定を挿入しない。

既存履歴の途中編集や再構成は、Phase1では対象外とする。

---

### 11.15 利用開始年月より前には設定できない

新しい`startYearMonth`は、資産口座の

```text
asset_accounts.start_year_month
```

より前に指定できない。

例えば、

```text
asset_accounts.start_year_month
    = 2026-08
```

の場合、

```text
2026-07
```

から始まる設定を追加できない。

---

### 11.16 同じisAvailableでは新しい履歴を作成しない

現在設定と同じ`isAvailable`を指定した場合は、新しい履歴を追加しない。

例えば、

```text
現在設定
2026-01 ～ NULL
isAvailable = true
```

に対して、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

を指定しても、業務状態は変化しない。

そのため、ACC-006では設定変更なしとして業務エラーとする。

---

### 11.17 不要な履歴分割を防止する

以下のような履歴は作成しない。

```text
2026-01 ～ 2026-07
true

2026-08 ～ NULL
true
```

状態が同一にもかかわらず期間だけ分割すると、履歴件数が不要に増えるためである。

ACC-006は、利用可能資産区分が実際に変化する場合だけ新しい履歴を追加する。

---

### 11.18 利用可能から利用対象外への変更

現在、

```text
2026-01 ～ NULL
isAvailable = true
```

の場合に、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

を登録すると、

```text
旧設定
2026-01 ～ 2026-07
true

新設定
2026-08 ～ NULL
false
```

となる。

---

### 11.19 利用対象外から利用可能への変更

現在、

```text
2026-01 ～ NULL
isAvailable = false
```

の場合に、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

を登録すると、

```text
旧設定
2026-01 ～ 2026-07
false

新設定
2026-08 ～ NULL
true
```

となる。

---

### 11.20 年跨ぎ

年月の前月計算では、年跨ぎを正しく扱う。

例えば、

```text
新設定.startYearMonth
    = 2027-01
```

の場合、

```text
旧設定.endYearMonth
    = 2026-12
```

となる。

---

### 11.21 現在より将来の年月を指定できる

Phase1で将来予約を許可する場合は、現在年月より後の

```text
startYearMonth
```

を指定できる。

例えば、現在年月が

```text
2026-08
```

で、

```json
{
  "startYearMonth": "2026-10",
  "isAvailable": false
}
```

を登録した場合、

```text
旧設定
～ 2026-09

新設定
2026-10 ～ NULL
```

となる。

---

### 11.22 将来設定登録後の意味

将来開始設定を登録すると、旧設定には将来の終了年月が設定される。

例えば、

```text
現在年月
2026-08

旧設定
2026-01 ～ NULL
true
```

に対して、

```text
新設定開始
2026-10
false
```

を登録すると、

```text
旧設定
2026-01 ～ 2026-09
true

新設定
2026-10 ～ NULL
false
```

となる。

---

### 11.23 将来予約が存在する状態でさらに追加しない

Phase1では、既に将来開始の最新設定が存在する状態で、その前後へさらに設定を追加して履歴を再構成する処理は扱わない。

ACC-006は、

```text
現在の最新履歴
    ↓
その後ろへ
新しい設定を追加
```

する操作に限定する。

---

### 11.24 過去設定を直接変更しない

ACC-006では、過去履歴の

```text
start_year_month
end_year_month
is_available
```

を自由に編集しない。

変更する既存レコードは、現在継続中の最新設定の

```text
end_year_month
```

だけとする。

---

### 11.25 asset_accountsは更新しない

ACC-006では、

```text
asset_accounts
```

を更新しない。

以下を変更しない。

```text
name
asset_type
balance_recording_unit
start_year_month
deleted_at
```

利用可能資産区分は、`asset_account_available_settings`だけで管理する。

---

### 11.26 他の関連テーブルを更新しない

ACC-006では、以下を更新しない。

```text
holding_assets
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
net_incomes
objectives
assessment_histories
```

利用可能資産設定履歴変更だけに責務を限定する。

---

### 11.27 過去の月末資産データを変更しない

利用可能資産設定を変更しても、既に登録されている

```text
month_end_asset_balances
month_end_holding_values
month_end_asset_snapshots
```

を変更しない。

設定履歴は、各対象年月時点でどの状態だったかを判定するために使用する。

---

### 11.28 過去の目的達成判定履歴を変更しない

ACC-006によって新しい利用可能資産区分を登録しても、

```text
assessment_histories
```

に保存済みの過去判定結果を自動更新しない。

既存履歴は、判定実行時点の結果として保持する。

---

## 12. 処理フロー

ACC-006の基本処理フローは、以下とする。

```text
Request
    ↓
X-User-Id検証
    ↓
UserContext取得
    ↓
assetAccountId形式検証
    ↓
Request Body検証
    ↓
トランザクション開始
    ↓
対象資産口座取得
    ↓
ASSET_ACCOUNT_NOT_FOUND？
    ↓ No
既存利用可能資産設定履歴取得
    ↓
履歴0件？
    ↓ No
履歴整合性確認
    ↓
現在設定1件を特定
    ↓
startYearMonth妥当性確認
    ↓
isAvailable変更あり？
    ↓ Yes
現在設定をロック
    ↓
旧設定.endYearMonth
    = 新設定.startYearMonthの前月
    ↓
新設定INSERT
    ↓
Result DTO生成
    ↓
コミット
    ↓
201 Created
```

---

### 12.1 利用者コンテキスト確認

共通Middlewareで`X-User-Id`を検証し、操作対象利用者を特定する。

利用者コンテキストが不正な場合は、トランザクションやDB更新へ進まない。

---

### 12.2 assetAccountId検証

`assetAccountId`が正の整数形式であることを確認する。

形式不正の場合は、資産口座検索や更新処理へ進まない。

---

### 12.3 Request Body検証

以下を検証する。

```text
startYearMonth
isAvailable
```

主に、

- 必須
- NULL禁止
- 型
- `YYYY-MM`形式
- 実在年月

を確認する。

---

### 12.4 トランザクション開始

入力形式の検証完了後、業務更新処理を同一トランザクション内で実行する。

概念的には、

```php
DB::transaction(function () {
    // 対象取得
    // 履歴確認
    // 旧設定更新
    // 新設定登録
});
```

とする。

---

### 12.5 資産口座取得

操作対象利用者IDと`assetAccountId`を条件として、利用中の資産口座を取得する。

概念的には、

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

とする。

---

### 12.6 資産口座不存在

対象資産口座を取得できない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

としてトランザクションを終了する。

以下を同じ扱いとする。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

---

### 12.7 既存履歴取得

対象資産口座に紐づく

```text
asset_account_available_settings
```

の履歴を取得する。

概念的には、

```text
asset_account_id
    = assetAccountId
```

を条件とする。

---

### 12.8 履歴0件確認

既存履歴が0件の場合は、ACC-002の初期登録ルールに反する。

そのため、新設定登録へ進まずデータ不整合として扱う。

---

### 12.9 履歴整合性確認

既存履歴について、ACC-005と同じ基本的な整合性ルールを確認する。

主に以下とする。

```text
初期開始年月一致
期間大小正常
期間重複なし
期間欠落なし
継続中設定1件
最新設定が継続中
```

不整合状態へ新しい履歴を追加しない。

---

### 12.10 現在設定取得

整合性確認後、最新かつ継続中の

```text
end_year_month = NULL
```

の設定を現在設定として扱う。

現在設定をクライアント指定のIDから取得しない。

---

### 12.11 startYearMonth業務検証

新設定の`startYearMonth`について、少なくとも以下を確認する。

```text
assetAccount.startYearMonth
    <=
request.startYearMonth

AND

currentSetting.startYearMonth
    <
request.startYearMonth
```

Phase1では、履歴途中への挿入を許可しない。

---

### 12.12 isAvailable差分確認

現在設定の

```text
is_available
```

とRequestの

```text
isAvailable
```

が異なることを確認する。

同一の場合は、不要な履歴追加となるため登録しない。

---

### 12.13 旧設定終了年月算出

Requestの

```text
startYearMonth
```

から1か月減算し、旧設定の終了年月を算出する。

概念的には、

```text
request.startYearMonth
    = 2026-08
        ↓
previousYearMonth
    = 2026-07
```

とする。

---

### 12.14 旧設定更新

現在設定の

```text
end_year_month
```

へ、算出した前月を設定する。

例えば、

```text
Before

2026-01 ～ NULL
true
```

を、

```text
After

2026-01 ～ 2026-07
true
```

へ更新する。

---

### 12.15 新設定INSERT

新しい履歴を以下の内容で登録する。

```text
asset_account_id
    = assetAccountId

start_year_month
    = request.startYearMonth

end_year_month
    = NULL

is_available
    = request.isAvailable
```

---

### 12.16 更新順序

概念的には、

```text
旧設定UPDATE
    ↓
新設定INSERT
```

とする。

ただし、両方を同一トランザクション内で実行するため、途中状態を外部へ確定させない。

---

### 12.17 新設定登録失敗時

旧設定のUPDATE後に新設定INSERTが失敗した場合は、トランザクション全体をロールバックする。

以下の状態を残してはならない。

```text
旧設定
終了済み

+
新設定
存在しない
```

---

### 12.18 旧設定更新失敗時

旧設定の更新に失敗した場合も、新設定を登録しない。

トランザクションをロールバックする。

---

### 12.19 Result DTO生成

新設定登録成功後、APIレスポンスに必要なResult DTOを生成する。

必要に応じて、

```text
新規設定ID
startYearMonth
endYearMonth
isAvailable
```

を保持する。

---

### 12.20 コミット

旧設定更新と新設定登録の両方が正常に完了した場合のみ、トランザクションをコミットする。

---

## 13. 成功レスポンス

ACC-006で新しい利用可能資産設定の登録に成功した場合は、

```http
201 Created
```

を返却する。

レスポンス例：

```json
{
  "data": {
    "id": "15",
    "startYearMonth": "2026-08",
    "endYearMonth": null,
    "isAvailable": false
  }
}
```

新規作成された利用可能資産設定を返却する。

---

### 13.1 レスポンス項目

正常時の`data`配下には、以下を返却する。

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `id` | string | × | 新しく登録した利用可能資産設定ID |
| `startYearMonth` | string | × | 新設定の適用開始年月。`YYYY-MM`形式 |
| `endYearMonth` | string | ○ | 新設定の終了年月。登録直後は`null` |
| `isAvailable` | boolean | × | 新設定の利用可能資産区分 |

---

### 13.2 id

新しく作成された

```text
asset_account_available_settings.id
```

をstringとして返却する。

---

### 13.3 startYearMonth

Requestで指定された

```text
startYearMonth
```

を、新規設定の適用開始年月として返却する。

---

### 13.4 endYearMonth

新規設定は継続中設定として登録されるため、正常レスポンスでは

```json
{
  "endYearMonth": null
}
```

となる。

---

### 13.5 isAvailable

Requestで指定された新しい利用可能資産区分をbooleanとして返却する。

---

### 13.6 旧設定をレスポンスへ含めない

ACC-006成功時に旧設定の

```text
end_year_month
```

も更新されるが、レスポンスへ旧設定を含めない。

必要な場合は、ACC-005を再取得して履歴全体を確認する。

---

### 13.7 資産口座情報を返却しない

ACC-006では、以下の資産口座情報をレスポンスへ含めない。

- `assetAccountId`
- `name`
- `assetType`
- `balanceRecordingUnit`
- `startYearMonth`（資産口座側）
- `isEnabled`

資産口座詳細はACC-003の責務とする。

---

## 14. トランザクション境界

ACC-006では、明示的なDBトランザクションを必須とする。

ACC-006は、

```text
旧設定UPDATE
+
新設定INSERT
```

という複数の状態変更を1つの業務操作として扱うためである。

---

### 14.1 UseCaseをトランザクション境界とする

トランザクションは、Repository単位ではなく、ACC-006のUseCase全体を境界とする。

概念的には、

```text
UseCase
    ↓
DB::transaction
    ├─ 資産口座確認
    ├─ 履歴確認
    ├─ 現在設定ロック
    ├─ 旧設定更新
    └─ 新設定登録
```

とする。

---

### 14.2 原子的に処理する

以下の2処理は、必ず同時に成功または失敗させる。

```text
1.
現在設定.end_year_month更新

2.
新しい利用可能資産設定INSERT
```

片方だけを確定させない。

---

### 14.3 旧設定更新後にINSERT失敗

例えば、

```text
旧設定UPDATE
    成功

新設定INSERT
    失敗
```

となった場合は、旧設定UPDATEもロールバックする。

結果として、実行前の

```text
旧設定
end_year_month = NULL
```

の状態へ戻す。

---

### 14.4 新設定だけ登録しない

以下の状態も発生させない。

```text
旧設定
end_year_month = NULL

新設定
start_year_month = 2026-08
end_year_month = NULL
```

これは継続中設定が複数存在する状態になるためである。

旧設定終了と新設定登録を同一トランザクションで扱う。

---

### 14.5 現在設定取得と更新

ACC-006では、同時実行による競合を防止するため、トランザクション内で現在設定を取得し、必要に応じて行ロックを取得する。

具体的な

```php
lockForUpdate()
```

の使用方針は、後続の「排他制御」で定義する。

---

### 14.6 履歴検証と更新の関係

履歴整合性を確認してから更新する。

ただし、確認から更新までの間に別ACC-006が割り込む可能性があるため、更新対象となる現在設定についてはトランザクション内で排他制御する。

---

### 14.7 Repository内で独自トランザクションを開始しない

以下のように、各Repositoryが個別トランザクションを開始しない。

```text
AvailableSettingRepository
    ↓
transaction

別処理
    ↓
別transaction
```

ACC-006という1つの業務操作全体でトランザクションを管理する。

---

### 14.8 ロールバック対象

以下の場合は、トランザクション全体をロールバックする。

- 現在設定更新失敗
- 新設定INSERT失敗
- UNIQUE制約違反
- DB例外
- 更新対象消失
- 同時更新競合
- その他想定外例外

---

### 14.9 バリデーションエラーはトランザクション開始前に検出する

HTTP入力として判定できる

```text
startYearMonth形式不正
isAvailable型不正
必須項目不足
```

などは、原則としてトランザクション開始前に検出する。

不要なトランザクションを開始しない。

---

### 14.10 業務条件はトランザクション内で再確認する

以下のようなDB状態に依存する条件は、トランザクション内で確認する。

```text
対象資産口座存在
現在設定存在
現在設定件数
現在設定startYearMonth
現在設定isAvailable
既存履歴状態
```

特に、更新直前のDB状態を基準として判定する。

---

### 14.11 トランザクション中に外部I/Oを行わない

ACC-006のトランザクション内では、以下のようなDB以外の外部I/Oを行わない。

```text
外部API
メール送信
ファイル操作
長時間処理
```

Phase1ではDB更新だけにトランザクションを限定する。

---

### 14.12 トランザクションを短く保つ

ACC-006では、必要な

```text
SELECT
LOCK
UPDATE
INSERT
```

だけをトランザクション内で実行する。

レスポンス整形や不要な関連データ取得をトランザクション内へ含めない。

---

## 15. 排他制御

ACC-006では、同一資産口座に対する利用可能資産設定登録の同時実行を考慮する。

ACC-006は、

```text
現在設定を終了
+
新設定を登録
```

という複数レコード更新を伴うため、Phase1では現在設定に対して明示的な行ロックを使用する。

概念的には、

```text
トランザクション開始
    ↓
現在設定取得
    ↓
lockForUpdate()
    ↓
業務条件再確認
    ↓
旧設定UPDATE
    ↓
新設定INSERT
    ↓
コミット
```

とする。

---

### 15.1 lockForUpdateを使用する

現在継続中の

```text
end_year_month = NULL
```

の利用可能資産設定を取得する際は、トランザクション内で

```php
lockForUpdate()
```

を使用する。

これにより、同一の現在設定を複数リクエストが同時に終了させることを防止する。

---

### 15.2 同一資産口座への同時実行

例えば、以下の2リクエストが同時に実行される場合を考える。

```text
Request A
2026-08から
isAvailable = false

Request B
2026-09から
isAvailable = false
```

排他制御を行わない場合、

```text
Request A
現在設定取得

Request B
現在設定取得

Request A
旧設定終了
新設定登録

Request B
同じ旧設定を終了
新設定登録
```

となり、履歴が不整合になる可能性がある。

---

### 15.3 現在設定をロックする理由

ACC-006で変更する既存レコードは、

```text
現在継続中設定
```

である。

そのため、対象となる現在設定行をロックすることで、同じ設定を基点とした同時履歴追加を直列化する。

---

### 15.4 資産口座行は原則ロックしない

Phase1では、以下の

```text
asset_accounts
```

行自体を原則としてロックしない。

利用者境界確認と対象存在確認のために参照するだけであり、ACC-006で更新するのは

```text
asset_account_available_settings
```

だからである。

ただし、将来的に資産口座無効化との厳密な競合制御が必要になった場合は、資産口座行のロックも検討する。

---

### 15.5 ロック取得後に業務条件を再確認する

現在設定を`lockForUpdate()`で取得した後は、そのロック済み状態を基準として業務条件を再確認する。

主に以下を確認する。

- 現在設定が1件だけ存在する
- 現在設定の`startYearMonth`より新設定開始年月が後である
- `isAvailable`が現在設定と異なる
- 現在設定がまだ`endYearMonth = null`である

ロック前に取得した古い状態だけを基準に更新しない。

---

### 15.6 同時実行時の待機

Request Aが現在設定をロックしている間、Request Bが同じ現在設定を取得しようとした場合は、Request Aのトランザクション終了まで待機する。

概念的には、

```text
Request A
lock取得
    ↓
更新
    ↓
commit
    ↓
Request B
lock取得
```

となる。

---

### 15.7 待機後は最新状態を基準に判定する

Request Bは、Request Aのコミット後に最新の現在設定を取得する。

その結果、Request Bの

```text
startYearMonth
isAvailable
```

が最新状態に対して業務条件を満たさない可能性がある。

この場合は、業務エラーとして処理し、履歴を無理に追加しない。

---

### 15.8 UNIQUE制約も併用する

行ロックだけに依存せず、DB側でも

```text
asset_account_id
+
start_year_month
```

の組み合わせにUNIQUE制約を設定する。

これにより、同一資産口座に同じ開始年月の設定が複数作成されることを防止する。

---

### 15.9 UNIQUE制約違反

並行実行などによってUNIQUE制約違反が発生した場合は、PostgreSQL例外をそのまま返却しない。

業務上の競合エラーへ変換する。

具体的な独自エラーコードは、後続の「エラーレスポンス」で定義する。

---

### 15.10 デッドロックを避ける

ACC-006では、ロック取得順を一定にする。

概念的には、

```text
1. assetAccountIdで対象確認
2. 現在設定を取得・ロック
3. 旧設定更新
4. 新設定登録
```

の順序を統一する。

複数テーブル・複数行を無秩序にロックしない。

---

### 15.11 ロック保持時間を短くする

`lockForUpdate()`取得後は、必要な業務判定とDB更新だけを行う。

以下をロック保持中に行わない。

```text
外部API呼び出し
メール送信
長時間計算
画面用レスポンス整形
```

トランザクションとロック保持時間を短く保つ。

---

### 15.12 将来的な楽観ロック

Phase1では、ACC-006について`version`や`updated_at`比較による楽観ロックは使用しない。

現在設定への行ロックによって同一資産口座の設定変更を直列化する。

将来的に高い同時実行性が必要になった場合は、楽観ロックへの変更を検討してよい。

---

## 16. エラーレスポンス

異常時は、API共通方針に従った共通エラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE",
    "message": "利用可能資産区分が現在の設定と同じです。"
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

この場合、資産口座検索や設定更新へ進まない。

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

リクエストボディの形式検証に失敗した場合は、

```text
VALIDATION_ERROR
```

を返却する。

主な対象は、

- `startYearMonth`未指定
- `startYearMonth = null`
- `startYearMonth`形式不正
- 実在しない年月
- `isAvailable`未指定
- `isAvailable = null`
- `isAvailable`がboolean以外
- 更新対象外項目の指定

とする。

---

### 16.6 startYearMonth形式不正

例えば、

```json
{
  "startYearMonth": "2026/08",
  "isAvailable": false
}
```

や、

```json
{
  "startYearMonth": "2026-13",
  "isAvailable": false
}
```

は、

```text
VALIDATION_ERROR
```

として扱う。

---

### 16.7 isAvailable型不正

以下のような値は受け付けない。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": "false"
}
```

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": 0
}
```

JSON booleanのみを許可する。

---

### 16.8 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

- 指定された資産口座が存在しない
- 指定された資産口座が他利用者に属する
- 指定された資産口座が論理削除済み

他利用者所属や論理削除済みであることを個別のエラーコードで公開しない。

---

### 16.9 他利用者の資産口座

例えば、

```text
User A
    assetAccountId = 10

User B
    X-User-Id = 2
```

の状態で、User Bが

```http
POST /api/v1/asset-accounts/10/available-settings
```

を実行した場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

User Aの利用可能資産設定を変更しない。

---

### 16.10 論理削除済み資産口座

対象資産口座が

```text
deleted_at IS NOT NULL
```

の場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

ACC-006から論理削除済み資産口座へ新しい設定を追加しない。

---

### 16.11 ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND

対象資産口座に利用可能資産設定履歴が1件も存在しない場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

を返却する。

ACC-002で初期設定を必ず登録する設計に対するデータ不整合として扱う。

---

### 16.12 ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID

既存履歴に以下のような整合性不正が存在する場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

を返却する。

主な対象は、

- 初期設定開始年月不一致
- `startYearMonth > endYearMonth`
- 期間重複
- 期間欠落
- 継続中設定が0件
- 継続中設定が複数件
- 最新設定が終了済み
- 同一開始年月の重複

とする。

不整合履歴へ新しい設定を追加しない。

---

### 16.13 現在設定0件

既存履歴は存在していても、

```text
end_year_month = NULL
```

の設定が0件の場合は、正常な設定変更を実行できない。

この場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

として扱う。

---

### 16.14 現在設定複数件

```text
end_year_month = NULL
```

の設定が複数存在する場合も、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

とする。

どの設定を終了させるか任意に決定しない。

---

### 16.15 ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID

Requestの

```text
startYearMonth
```

が業務上登録できない年月の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

とする。

主な対象は、

- 資産口座利用開始年月より前
- 現在設定開始年月より前
- 現在設定開始年月と同じ
- 履歴途中への挿入となる年月

とする。

---

### 16.16 現在設定と同じ開始年月

例えば、

```text
現在設定
2026-08 ～ NULL
```

に対して、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

を指定した場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

として扱う。

---

### 16.17 過去開始年月

例えば、

```text
現在設定
2026-08 ～ NULL
```

に対して、

```json
{
  "startYearMonth": "2026-05",
  "isAvailable": true
}
```

を指定した場合も、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

とする。

ACC-006は過去履歴を再構成するAPIではない。

---

### 16.18 ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE

Requestの`isAvailable`が現在設定と同じ場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

を返却する。

例えば、

```text
現在設定
isAvailable = true
```

に対して、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

を指定した場合である。

---

### 16.19 同一状態では履歴を作成しない

`ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE`の場合は、

```text
旧設定UPDATE
新設定INSERT
```

のどちらも行わない。

以下のような不要な履歴分割を防止する。

```text
2026-01 ～ 2026-07
true

2026-08 ～ NULL
true
```

---

### 16.20 ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS

同一資産口座に同じ`startYearMonth`の設定が既に存在する場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

として扱う。

これは、アプリケーション側の事前確認またはDBのUNIQUE制約違反から検出できる。

---

### 16.21 UNIQUE制約違反

並行実行等によって、

```text
asset_account_id
+
start_year_month
```

のUNIQUE制約違反が発生した場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

へ変換する。

PostgreSQLの内部例外をそのまま返却しない。

---

### 16.22 同時更新競合

同一資産口座に対して複数のACC-006が同時実行された場合、`lockForUpdate()`によって直列化する。

待機後に最新状態を再確認した結果、Request内容が最新履歴に対して不正となった場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

または、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

など、実際の最新状態に応じた業務エラーを返却する。

---

### 16.23 LOCK_TIMEOUT

DB側のロック待機が許容時間を超えた場合は、API共通方針に専用競合エラーを定義するなら、

```text
RESOURCE_BUSY
```

などへ変換してよい。

ただし、Phase1で専用コードを設けない場合は、

```text
INTERNAL_SERVER_ERROR
```

として扱う。

具体的な方針はAPI共通方針を優先する。

---

### 16.24 トランザクション失敗

以下のいずれかが失敗した場合は、トランザクション全体をロールバックする。

- 現在設定UPDATE
- 新設定INSERT
- DB制約確認
- コミット

旧設定だけを終了させた状態や、新設定だけを追加した状態を残さない。

---

### 16.25 UPDATE成功後にINSERT失敗

例えば、

```text
旧設定UPDATE
    成功

新設定INSERT
    失敗
```

となった場合でも、レスポンス上だけエラーにするのではなく、旧設定UPDATEもロールバックする。

---

### 16.26 INTERNAL_SERVER_ERROR

想定外のサーバー内部エラーが発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスへ、以下の内部情報を含めない。

- SQL
- PostgreSQL内部エラー
- SQLSTATE
- 制約名
- テーブル名
- カラム名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 16.27 主なエラー一覧

ACC-006で想定する主なエラーは、以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`未指定 |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`形式不正 |
| `400 Bad Request` | `INVALID_ASSET_ACCOUNT_ID` | `assetAccountId`形式不正 |
| `400 Bad Request` | `VALIDATION_ERROR` | Request Body形式不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 利用者不存在、または論理削除済み |
| `404 Not Found` | `ASSET_ACCOUNT_NOT_FOUND` | 資産口座不存在、他利用者所属、または論理削除済み |
| `409 Conflict` | `ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID` | 新設定開始年月が業務上不正 |
| `409 Conflict` | `ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE` | 現在設定と同じ`isAvailable`を指定 |
| `409 Conflict` | `ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS` | 同一開始年月の設定が既に存在する |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND` | 利用可能資産設定履歴が0件 |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID` | 既存履歴が不整合 |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバー内部エラー |

具体的なHTTPステータスや独自エラーコードの最終定義は、API共通方針を優先する。

---

### 16.28 エラー時の更新

以下のエラーをUPDATE前に検出した場合は、

```text
asset_account_available_settings
```

を一切変更しない。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

---

### 16.29 DB更新中のエラー

旧設定UPDATEまたは新設定INSERTの途中でエラーが発生した場合は、トランザクションをロールバックする。

そのため、エラーレスポンス返却時に

```text
旧設定だけ変更済み
```

または、

```text
新設定だけ登録済み
```

という中途半端な状態を残さない。

---

## 17. HTTPステータス

ACC-006では、処理結果に応じて以下のHTTPステータスを返却する。

| HTTPステータス | 用途 |
|---|---|
| `201 Created` | 利用可能資産設定登録成功 |
| `400 Bad Request` | 利用者コンテキスト、`assetAccountId`、またはRequest Bodyの形式不正 |
| `404 Not Found` | 利用者または資産口座が存在しない |
| `409 Conflict` | 新設定開始年月不正、状態変更なし、同一開始年月競合 |
| `500 Internal Server Error` | 利用可能資産設定履歴の不整合、または想定外のサーバー内部エラー |

具体的な独自エラーコードは、「エラーレスポンス」およびAPI共通方針に従う。

---

### 17.1 201 Created

新しい利用可能資産設定の登録に成功した場合は、

```http
201 Created
```

を返却する。

概念的には、

```text
対象資産口座確認
    ↓
既存履歴確認
    ↓
現在設定ロック
    ↓
旧設定終了
    ↓
新設定登録
    ↓
コミット
    ↓
201 Created
```

とする。

---

### 17.2 200 OKではなく201 Createdとする理由

ACC-006では、既存設定を更新するだけでなく、

```text
asset_account_available_settings
```

へ新しい履歴レコードを作成する。

そのため、新規リソース作成を表す

```text
201 Created
```

を使用する。

---

### 17.3 400 Bad Request

以下の場合は、

```text
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
- `startYearMonth`未指定
- `startYearMonth`形式不正
- 実在しない年月
- `isAvailable`未指定
- `isAvailable`がboolean以外
- 更新対象外項目の指定

などを対象とする。

---

### 17.4 404 Not Found

以下の場合は、

```text
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

### 17.5 409 Conflict

現在のDB状態に対してRequest内容を適用できない場合は、

```text
409 Conflict
```

を返却する。

主なエラーコードは、以下とする。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

---

### 17.6 START_YEAR_MONTH_INVALID

新しい設定の`startYearMonth`が現在の履歴状態に対して登録できない場合は、

```text
409 Conflict
```

とし、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

を返却する。

主な対象は、

- 資産口座利用開始年月より前
- 現在設定開始年月より前
- 現在設定開始年月と同じ
- 履歴途中への挿入となる年月

とする。

---

### 17.7 NO_CHANGE

現在設定と同じ`isAvailable`を指定した場合は、

```text
409 Conflict
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

とする。

この場合、旧設定UPDATEも新設定INSERTも行わない。

---

### 17.8 ALREADY_EXISTS

同一資産口座に同じ`startYearMonth`の設定が既に存在する場合は、

```text
409 Conflict
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

とする。

アプリケーション側の事前確認で検出した場合も、DBのUNIQUE制約違反で検出した場合も、同じHTTPステータスおよび独自エラーコードへ変換する。

---

### 17.9 500 Internal Server Error

以下の場合は、

```text
500 Internal Server Error
```

として扱う。

主なエラーコードは、以下とする。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
INTERNAL_SERVER_ERROR
```

既存履歴の不存在や不整合は、クライアント入力によるものではなく、サーバー側の業務データ不整合として扱う。

---

### 17.10 HISTORY_NOT_FOUND

対象資産口座について、

```text
asset_account_available_settings
```

が1件も存在しない場合は、

```text
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

とする。

ACC-002で初期設定を必ず作成する設計に対するデータ不整合として扱う。

---

### 17.11 HISTORY_INVALID

既存履歴に以下のような不整合が存在する場合は、

```text
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

とする。

主な対象は、

- 初期設定開始年月不一致
- `startYearMonth > endYearMonth`
- 期間重複
- 期間欠落
- 継続中設定0件
- 継続中設定複数件
- 最新設定が終了済み
- 同一開始年月の重複

とする。

---

### 17.12 INTERNAL_SERVER_ERROR

想定外のサーバー内部エラーが発生した場合は、

```text
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

ACC-006は、新しい利用可能資産設定履歴を追加するPOST APIであるため、HTTP操作としては冪等ではない。

同一Requestを無条件に複数回実行して毎回新しい履歴を追加する設計にはしないが、POST自体を冪等操作として定義しない。

---

### 18.1 同一Requestの再実行

例えば、現在設定が

```text
2026-01 ～ NULL
isAvailable = true
```

の状態で、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

を実行すると、

```text
2026-01 ～ 2026-07
true

2026-08 ～ NULL
false
```

となる。

その後、同一Requestを再実行した場合は、最新状態に対して再判定する。

---

### 18.2 再実行時に同じ履歴を追加しない

1回目の実行後は、現在設定が

```text
2026-08 ～ NULL
isAvailable = false
```

となっている。

そのため、同一Requestを再実行すると、

```text
startYearMonth
    = 現在設定.startYearMonth

AND

isAvailable
    = 現在設定.isAvailable
```

となる。

この場合は、新しい履歴を追加しない。

---

### 18.3 再実行時に返り得るエラー

同一Requestの再実行では、最新状態に応じて、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

または、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

となり得る。

どちらを優先するかは、業務チェック順序をLaravel実装方針で統一する。

---

### 18.4 重複防止

以下のDB制約によっても、同じ開始年月の設定を複数登録できないようにする。

```text
asset_account_id
+
start_year_month
```

UNIQUE制約違反は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

へ変換する。

---

### 18.5 同一状態の履歴増殖を防止する

現在設定と同じ`isAvailable`で新規履歴を追加しない。

これにより、

```text
2026-01 ～ 2026-07
true

2026-08 ～ 2026-09
true

2026-10 ～ NULL
true
```

のような意味のない履歴増殖を防止する。

---

### 18.6 Idempotency-Key

Phase1では、ACC-006専用の

```text
Idempotency-Key
```

は使用しない。

理由は、以下の仕組みによって重複した履歴作成を防止できるためである。

```text
業務状態再確認
+
現在設定への行ロック
+
asset_account_id / start_year_month
UNIQUE制約
```

---

### 18.7 通信結果不明時

クライアントがACC-006送信後にネットワークエラーとなり、

```text
サーバーでは成功したのか
失敗したのか
```

を判断できない場合がある。

Phase1では、即座に同じPOSTを自動再送するのではなく、

```text
ACC-005
利用可能資産設定履歴取得

または

ACC-003
資産口座詳細取得
```

によって現在状態を再取得したうえで利用者へ表示する方針とする。

---

### 18.8 自動Retryを前提としない

ACC-006は更新APIであるため、クライアント側の無条件な自動Retryを前提としない。

TanStack Query等では、Mutationの自動Retryを原則無効とする。

---

### 18.9 排他制御と冪等性は別とする

現在設定への`lockForUpdate()`は、同時実行による履歴破壊を防止するための排他制御である。

これは、HTTPリクエスト自体を冪等にする仕組みではない。

概念的には、

```text
lockForUpdate()
    → 同時更新制御

UNIQUE制約
    → 重複データ防止

業務チェック
    → 不要な履歴追加防止

Idempotency
    → 別概念
```

として区別する。

---

## 19. キャッシュ

Phase1では、ACC-006専用のサーバー側アプリケーションキャッシュを使用しない。

ACC-006成功後は、PostgreSQL上の

```text
asset_account_available_settings
```

が最新状態となる。

---

### 19.1 更新API自体をキャッシュしない

ACC-006は状態変更を行うPOST APIであるため、

```http
POST /api/v1/asset-accounts/{assetAccountId}/available-settings
```

のレスポンスをHTTPキャッシュによって後続Requestへ再利用することを前提としない。

---

### 19.2 ACC-005への影響

ACC-006成功時は、利用可能資産設定履歴が必ず変化する。

概念的には、

```text
ACC-006
    ↓
旧設定.endYearMonth更新
+
新設定追加
    ↓
ACC-005の取得結果が変化
```

となる。

そのため、React側でACC-005をQuery Cacheしている場合は、ACC-006成功後に対象資産口座の履歴Queryを無効化する。

---

### 19.3 ACC-003への影響

ACC-003では、現在年月に有効な

```text
isAvailable
```

を返却する。

ACC-006によって現在年月に適用される設定が変化した場合は、ACC-003のレスポンスも変化する。

そのため、ACC-006成功後は、対象資産口座の詳細Queryも無効化する。

---

### 19.4 ACC-005 Query Cacheの無効化

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.availableSettings(
      assetAccountId,
    ),
});
```

これにより、ACC-005を再実行して最新履歴を取得する。

---

### 19.5 ACC-003 Query Cacheの無効化

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

これにより、最新の`isAvailable`を再取得する。

---

### 19.6 ACC-001への影響

ACC-001 資産口座一覧取得が`isAvailable`を返却する設計の場合は、ACC-006成功後に資産口座一覧Queryも無効化する。

概念的には、

```text
ACC-006成功
    ↓
ACC-001
ACC-003
ACC-005
```

のうち、`isAvailable`を含むQueryを再取得対象とする。

ACC-001が`isAvailable`を返却しない設計であれば、ACC-006だけを理由として必ず無効化する必要はない。

---

### 19.7 将来開始設定の場合

ACC-006で現在より将来の

```text
startYearMonth
```

を登録した場合、ACC-005の履歴は即座に変化する。

一方、ACC-003の現在の

```text
isAvailable
```

はまだ変化しない場合がある。

例えば、

```text
現在年月
2026-08

現在設定
2026-01 ～ 2026-09
true

新設定
2026-10 ～ NULL
false
```

の場合、2026-08時点の`isAvailable`は引き続き`true`となる。

---

### 19.8 将来設定でもACC-003を再取得してよい

将来開始設定の場合でも、React側で適用年月判定を独自に行うより、

```text
ACC-006成功
    ↓
ACC-003再取得
```

としてよい。

サーバー側の現在年月判定を正とし、フロントエンドに不要な業務ロジックを持たせない。

---

### 19.9 ACC-005を必ず再取得する

ACC-006成功時は、新しい履歴が追加され、旧設定の終了年月も変更される。

そのため、ACC-005のキャッシュは必ず古くなる。

Phase1では、

```text
ACC-006成功
    ↓
ACC-005 invalidate
    ↓
再取得
```

を基本とする。

---

### 19.10 成功レスポンスだけで履歴Cacheを再構築しない

ACC-006の成功レスポンスでは、新しく作成した設定だけを返却する。

旧設定については、レスポンスへ含めない。

そのため、React側で

```text
新設定を配列へ追加
+
旧設定.endYearMonthを手動計算
```

して履歴Cacheを再構築することは、Phase1では基本としない。

---

### 19.11 サーバー状態を正とする

利用可能資産設定履歴の最終状態は、PostgreSQL上のデータを正とする。

ACC-006成功後は、ACC-005を再取得し、

```text
旧設定のendYearMonth
新設定ID
新設定startYearMonth
新設定isAvailable
```

をサーバー確定状態から取得する。

---

### 19.12 Optimistic Update

Phase1では、ACC-006に対するOptimistic Updateを必須としない。

ACC-006では、

- 既存履歴整合性確認
- 現在設定確認
- 開始年月判定
- 同一状態判定
- 行ロック
- UNIQUE制約
- トランザクション

など、サーバー側で最終判断する条件が多いためである。

---

### 19.13 Mutation成功後に更新する

React側では、

```text
ACC-006成功
    ↓
Query invalidate
    ↓
最新状態再取得
```

を基本とする。

サーバー成功前に履歴表示を先に確定しない。

---

### 19.14 サーバー側キャッシュ

Phase1では、以下のようなACC-006専用キャッシュを使用しない。

```text
Redis
Laravel Cache
Application Memory Cache
```

利用可能資産設定変更は高頻度処理ではなく、DB状態との整合性を優先する。

---

### 19.15 ACC-004への影響

ACC-006では、

```text
asset_accounts.name
asset_accounts.asset_type
```

を変更しない。

そのため、ACC-004で編集する通常属性自体はACC-006によって変化しない。

ただし、ACC-003相当の詳細表示を同一Queryで管理している場合は、`isAvailable`変更のため詳細Queryを無効化する。

---

### 19.16 月末資産関連Cache

ACC-006では、以下のテーブルを直接更新しない。

```text
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
```

そのため、月末資産データ自体のQuery Cacheを必ず無効化する必要はない。

ただし、表示時に対象年月の利用可能資産区分を組み合わせているQueryがある場合は、そのQuery設計に応じて失効を検討する。

---

### 19.17 目的達成判定関連Cache

ACC-006は、

```text
assessment_histories
```

を更新しない。

既存の目的達成判定履歴も変更しない。

ただし、将来新たに判定を実行する際には、対象年月の利用可能資産設定が判定条件へ影響する可能性がある。

過去の判定履歴CacheをACC-006成功だけで書き換えない。

---

### 19.18 利用者境界とQuery Cache

ACC-006成功後にQuery Cacheを無効化する場合も、利用者境界を考慮する。

Phase1では`X-User-Id`で操作対象利用者を切り替えるため、前利用者のQuery Cacheを別利用者へ誤表示しないようにする。

---

### 19.19 Query KeyへuserIdを含める方式

必要に応じて、

```typescript
[
  'assetAccounts',
  userId,
  assetAccountId,
  'availableSettings',
]
```

のように、Query Keyへ`userId`を含めてよい。

または、利用者切替時に資産口座関連Queryを一括無効化する。

正式な方式は、React共通設計に従う。

---

### 19.20 HTTPキャッシュ

ACC-006自体はHTTPキャッシュ対象としない。

また、ACC-006成功後にACC-003やACC-005の古いGETレスポンスが長期間返却されないよう、将来的にHTTPキャッシュを導入する場合は、関連リソースの失効設計を合わせて行う。

---

### 19.21 ETag等

Phase1では、

```text
ETag
Last-Modified
If-Match
If-None-Match
```

などのHTTP条件付きRequestをACC-006へ導入しない。

同時更新制御は、現在設定への`lockForUpdate()`とDB制約によって行う。

---

### 19.22 キャッシュ方針まとめ

ACC-006成功後の基本的なキャッシュ更新方針は、以下とする。

```text
ACC-006成功
    ↓
ACC-005
利用可能資産設定履歴
    → invalidate

ACC-003
資産口座詳細
    → invalidate

ACC-001
資産口座一覧
    → isAvailableを含む場合はinvalidate
```

Phase1では、クライアント側で履歴を複雑に手動再構築せず、サーバーからの再取得を基本とする。

---

## 20. 関連テーブル

ACC-006では、指定された資産口座の利用可能資産設定を変更するため、以下のテーブルを使用する。

| テーブル                               | 用途                         |  更新 |
| ---------------------------------- | -------------------------- | :-: |
| `users`                            | 操作対象利用者の確認                 |  ×  |
| `asset_accounts`                   | 対象資産口座の取得、利用者境界確認、利用開始年月確認 |  ×  |
| `asset_account_available_settings` | 既存履歴取得、現在設定ロック、旧設定終了、新設定登録 |  ○  |

ACC-006で直接更新する業務テーブルは、

```text
asset_account_available_settings
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

ACC-006では、`users`を更新しない。

---

### 20.2 asset_accounts

指定された`assetAccountId`に対応する資産口座を取得し、利用者境界および利用開始年月を確認するために使用する。

主に以下のカラムを使用する。

| カラム                | 用途                      |
| ------------------ | ----------------------- |
| `id`               | `assetAccountId`との照合    |
| `user_id`          | 操作対象利用者との利用者境界確認        |
| `start_year_month` | 新設定開始年月および履歴初期年月との整合性確認 |
| `deleted_at`       | 利用状態の判定                 |

---

### 20.3 対象資産口座取得

対象資産口座は、概念的に以下の条件で取得する。

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

他利用者に属する資産口座、論理削除済み資産口座は設定変更対象として取得しない。

---

### 20.4 asset_accountsは更新しない

ACC-006では、以下の資産口座属性を変更しない。

```text
name
asset_type
balance_recording_unit
start_year_month
deleted_at
```

利用可能資産区分は、

```text
asset_account_available_settings
```

によって履歴管理する。

---

### 20.5 asset_account_available_settings

ACC-006の主たる更新対象テーブルとする。

主に以下のカラムを使用する。

| カラム                | 用途          |
| ------------------ | ----------- |
| `id`               | 利用可能資産設定ID  |
| `asset_account_id` | 対象資産口座との関連  |
| `start_year_month` | 設定適用開始年月    |
| `end_year_month`   | 設定適用終了年月    |
| `is_available`     | 利用可能資産区分    |
| `created_at`       | 新設定登録日時     |
| `updated_at`       | 旧設定終了時の更新日時 |

---

### 20.6 既存履歴取得

対象資産口座に紐づく既存履歴を、

```text
asset_account_available_settings.asset_account_id
    = assetAccountId
```

で取得する。

履歴整合性確認では、必要に応じて

```text
start_year_month ASC
```

で取得する。

---

### 20.7 現在設定取得

現在設定は、

```text
asset_account_id
    = assetAccountId

AND

end_year_month
    IS NULL
```

の条件で取得する。

正常状態では、該当する設定が1件だけ存在する。

---

### 20.8 現在設定をロックする

ACC-006では、現在設定をトランザクション内で取得し、

```php
lockForUpdate()
```

を使用する。

これにより、同一資産口座への設定変更を直列化する。

---

### 20.9 旧設定の更新

現在設定の

```text
end_year_month
```

を、

```text
request.startYearMonth
の前月
```

へ更新する。

例えば、

```text
request.startYearMonth
    = 2026-08
```

の場合、

```text
end_year_month
    = 2026-07
```

とする。

---

### 20.10 新設定の登録

新しい設定は、以下の内容で登録する。

```text
asset_account_id
    = assetAccountId

start_year_month
    = request.startYearMonth

end_year_month
    = NULL

is_available
    = request.isAvailable
```

新設定登録時に、旧設定の値をコピーしてから一部変更する方式にはしない。

---

### 20.11 asset_account_id

新設定の

```text
asset_account_id
```

は、URLで指定され、利用者境界確認済みの`assetAccountId`を使用する。

Request Bodyから指定された値は使用しない。

---

### 20.12 start_year_month

新設定の

```text
start_year_month
```

には、Requestの

```text
startYearMonth
```

を保存する。

形式は、

```text
YYYY-MM
```

とする。

---

### 20.13 end_year_month

新設定は継続中設定として登録するため、

```text
end_year_month
    = NULL
```

とする。

クライアント指定値を保存しない。

---

### 20.14 is_available

Requestの

```text
isAvailable
```

を、

```text
asset_account_available_settings.is_available
```

へ保存する。

APIではboolean、DBではテーブル定義に従ったboolean表現を使用する。

---

### 20.15 一意性

同一資産口座内では、

```text
asset_account_id
+
start_year_month
```

を一意とする。

これにより、以下のような状態を防止する。

```text
assetAccountId = 10

2026-08
true

2026-08
false
```

---

### 20.16 現在設定の一意性

業務ルール上は、同一資産口座について

```text
end_year_month IS NULL
```

の設定は1件のみとする。

PostgreSQLを使用する場合は、必要に応じて

```text
asset_account_id
WHERE end_year_month IS NULL
```

に対する部分UNIQUEインデックスを検討してよい。

これにより、アプリケーション側の`lockForUpdate()`に加えて、DB側でも継続中設定複数件を防止できる。

---

### 20.17 履歴期間の連続性

正常な履歴では、前設定と次設定について、

```text
前設定.end_year_month
    の翌月
        =
次設定.start_year_month
```

となる。

例えば、

```text
2026-01 ～ 2026-06
2026-07 ～ NULL
```

は正常とする。

---

### 20.18 初期設定との整合性

最古の利用可能資産設定の

```text
start_year_month
```

は、

```text
asset_accounts.start_year_month
```

と一致することを正常状態とする。

概念的には、

```text
asset_accounts.start_year_month
    =
MIN(
    asset_account_available_settings.start_year_month
)
```

となる。

---

### 20.19 更新しない関連テーブル

ACC-006では、以下のテーブルを更新しない。

* `users`
* `asset_accounts`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

既存の月末資産データや目的達成判定履歴へ副作用を与えない。

---

## 21. 関連する機能要件

ACC-006は、利用可能資産区分の履歴管理および変更に関する機能要件と対応する。

主な関連要件は、以下とする。

* 資産口座管理

  * 資産口座ごとに利用可能資産区分を設定できる
  * 利用可能資産区分を変更できる
  * 利用可能資産区分の変更履歴を保持する
* 利用可能資産設定

  * 資産口座登録時に初期設定を作成する
  * 設定変更時は旧設定を終了する
  * 新設定を履歴として追加する
  * 同一状態の不要な履歴を追加しない
  * 設定期間を重複させない
  * 設定期間に欠落を作らない
  * 継続中設定は1件のみとする
* 利用者境界

  * 操作対象利用者に帰属する資産口座だけを変更できる
  * 他利用者の資産口座は対象不存在として扱う
* 論理削除

  * 論理削除済み資産口座へ新しい設定を追加しない

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

## 22. インデックス

ACC-006では、主に以下の検索条件を使用する。

```text
asset_accounts.id
```

```text
asset_account_available_settings.asset_account_id
```

```text
asset_account_available_settings.start_year_month
```

```text
asset_account_available_settings.end_year_month
```

---

### 22.1 asset_accounts.id

`asset_accounts.id`は主キーであるため、主キーインデックスを使用する。

ACC-006専用として追加インデックスを作成しない。

---

### 22.2 asset_account_id

利用可能資産設定履歴は、

```text
asset_account_id
```

を条件として取得する。

FK列としてインデックスを設定する、または既存インデックスを利用する。

---

### 22.3 asset_account_id + start_year_month

同一資産口座内で開始年月を一意とするため、

```text
asset_account_id
+
start_year_month
```

にUNIQUE制約を設定する。

このインデックスは、

* 履歴取得
* 開始年月重複確認
* 並行登録時の競合防止

にも利用できる。

---

### 22.4 継続中設定の部分UNIQUEインデックス

PostgreSQLでは、必要に応じて

```text
asset_account_id
WHERE end_year_month IS NULL
```

の部分UNIQUEインデックスを検討する。

概念例：

```sql
CREATE UNIQUE INDEX
    uq_asset_account_available_settings_current
ON
    asset_account_available_settings (
        asset_account_id
    )
WHERE
    end_year_month IS NULL;
```

これにより、同一資産口座に継続中設定が複数存在する状態をDB側でも防止できる。

---

### 22.5 lockForUpdateとDB制約を併用する

ACC-006では、

```text
lockForUpdate
    → 同時更新を直列化

UNIQUE制約
    → 重複開始年月防止

部分UNIQUE制約
    → 継続中設定複数件防止
```

という複数層の整合性保証を組み合わせてよい。

---

### 22.6 不要なインデックスを追加しない

ACC-006だけを理由として、既存の

* 主キー
* FKインデックス
* UNIQUE制約

と役割が重複するインデックスを追加しない。

実際のSQLと実行計画を確認したうえで必要性を判断する。

---

## 23. 性能

ACC-006は、単一資産口座の利用可能資産設定履歴を変更するAPIである。

通常は、

```text
資産口座1件取得
    ↓
設定履歴取得
    ↓
現在設定1件ロック
    ↓
旧設定1件UPDATE
    ↓
新設定1件INSERT
```

で完結する。

---

### 23.1 資産口座一覧を取得しない

操作対象利用者の資産口座一覧を全件取得してから対象資産口座を探さない。

```text
assetAccountId
+
userId
```

を条件として直接1件取得する。

---

### 23.2 他資産口座の履歴を取得しない

利用可能資産設定履歴は、

```text
asset_account_id
    = assetAccountId
```

で絞り込む。

全資産口座分の設定履歴を取得してPHP側で絞り込まない。

---

### 23.3 履歴件数

利用可能資産設定は年月単位の変更履歴であるため、1資産口座あたりの件数は比較的少ないことを想定する。

Phase1では、履歴全件を取得して整合性確認してよい。

---

### 23.4 履歴整合性確認

履歴整合性確認は、取得済み履歴を開始年月順に1回走査する程度とする。

各履歴レコードごとに追加SQLを発行しない。

概念的には、

```text
履歴SELECT 1回
    ↓
PHP側で前後比較
```

とする。

---

### 23.5 現在設定取得

排他制御時は、現在設定のみを

```text
end_year_month IS NULL
```

で取得し、

```php
lockForUpdate()
```

する。

必要以上の履歴行をロックしない。

---

### 23.6 ロック範囲を最小化する

ACC-006では、対象資産口座の現在設定行だけを原則ロックする。

履歴全件や同一利用者の全資産口座をロックしない。

---

### 23.7 トランザクションを短く保つ

ロック取得後は、

```text
業務条件再確認
旧設定UPDATE
新設定INSERT
```

を速やかに実行する。

以下をトランザクション中に行わない。

* 外部API呼び出し
* メール送信
* ファイル処理
* 不要なEager Load
* 大量データ取得

---

### 23.8 N+1問題

ACC-006は単一資産口座を対象とし、履歴を一括取得するため、通常の意味でのN+1問題は発生しない。

---

## 24. セキュリティ

ACC-006では、利用者境界、更新可能項目の限定、履歴操作のサーバー側制御を重要なセキュリティ要件とする。

---

### 24.1 assetAccountIdだけで取得しない

以下のような対象取得は避ける。

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

### 24.2 他利用者の設定を変更しない

他利用者に属する資産口座IDが指定された場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

他利用者の利用可能資産設定を検索・ロック・更新しない。

---

### 24.3 settingIdをクライアントに指定させない

現在設定の

```text
id
```

をRequestから指定させない。

現在設定は、サーバー側で

```text
asset_account_id
+
end_year_month IS NULL
```

から特定する。

これにより、任意の過去履歴を直接更新することを防止する。

---

### 24.4 Mass Assignmentを避ける

以下のようにRequest全体をそのままModelへ渡さない。

```php
$setting->update(
    $request->all(),
);
```

更新・登録対象を明示的に限定する。

---

### 24.5 endYearMonthをクライアント指定させない

旧設定の終了年月は、

```text
新設定開始年月の前月
```

としてサーバー側で計算する。

Requestから

```text
endYearMonth
```

を指定させない。

---

### 24.6 assetAccountIdをRequest Bodyから受け付けない

対象資産口座はURLだけから取得する。

Request Body中の`assetAccountId`によって対象を変更できないようにする。

---

### 24.7 userIdをRequest Bodyから受け付けない

利用者IDは`X-User-Id`からのみ取得する。

Request Body中の`userId`によって所有者境界を変更できないようにする。

---

### 24.8 過去履歴を直接編集させない

ACC-006では、利用可能資産設定IDを指定した任意UPDATEを許可しない。

これにより、クライアント側から

```text
過去履歴期間改ざん
```

を行えないようにする。

---

### 24.9 SQLインジェクション対策

以下の入力値を検索条件へ使用する場合は、EloquentまたはQuery Builderのバインド機構を使用する。

* `assetAccountId`
* 利用者ID
* `startYearMonth`

SQL文字列へ入力値を直接連結しない。

---

### 24.10 内部情報を公開しない

エラー時に、以下をレスポンスへ含めない。

* SQL
* SQLSTATE
* PostgreSQL内部エラー
* 制約名
* テーブル名
* カラム名
* Laravel内部例外
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

---

## 25. ログ・監視

ACC-006では、API共通ログ方針に従って設定変更結果を記録する。

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
ACC-006
```

とする。

---

### 25.2 正常時

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = ACC-006
assetAccountId
newSettingId
httpStatus = 201
```

設定変更の追跡が必要な場合は、

```text
startYearMonth
```

を内部ログへ記録してもよい。

---

### 25.3 isAvailableのログ

`isAvailable`は機密情報ではないが、通常ログへ不要に大量出力しない。

設定変更調査上必要であれば、

```text
previousIsAvailable
newIsAvailable
```

を構造化ログとして記録してよい。

---

### 25.4 異常時

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

### 25.5 履歴不整合

以下のエラーでは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

調査可能な情報を内部ログへ記録する。

必要に応じて、

```text
assetAccountId
historyCount
invalidReason
```

を記録してよい。

---

### 25.6 同時更新競合

以下のような競合が発生した場合は、必要に応じて内部ログへ記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

ロック待機時間や競合回数を監視対象にしてもよい。

---

### 25.7 DB制約違反

UNIQUE制約違反を業務エラーへ変換した場合、内部ログでは原因調査のためにSQLSTATEや制約識別情報を記録してよい。

ただし、APIレスポンスへは公開しない。

---

### 25.8 トランザクションロールバック

旧設定UPDATE後に新設定INSERTが失敗した場合など、トランザクションがロールバックされたことを障害調査可能な状態にする。

必要に応じて、

```text
transactionRolledBack = true
```

相当の内部ログを記録してよい。

---

### 25.9 requestId

クライアントへ返却する`requestId`とサーバーログを関連付けられるようにする。

概念的には、

```text
requestId
    ↓
ACC-006ログ
    ↓
トランザクション
    ↓
競合・不整合原因調査
```

を可能とする。

---

## 26. テスト観点

ACC-006では、入力値検証だけでなく、

* 利用者境界
* 履歴整合性
* 開始年月
* 状態変更有無
* 旧設定更新
* 新設定登録
* トランザクション
* 排他制御
* UNIQUE制約
* 副作用範囲

を重点的に確認する。

---

### 26.1 正常系：trueからfalse

現在設定を、

```text
2026-01 ～ NULL
is_available = true
```

とする。

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

実行後、

```text
旧設定
2026-01 ～ 2026-07
true

新設定
2026-08 ～ NULL
false
```

となることを確認する。

---

### 26.2 正常系：falseからtrue

現在設定を、

```text
2026-01 ～ NULL
false
```

とする。

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

実行後、

```text
2026-01 ～ 2026-07
false

2026-08 ～ NULL
true
```

となることを確認する。

---

### 26.3 成功レスポンス

正常時に、

```text
201 Created
```

となることを確認する。

概念的なレスポンス：

```json
{
  "data": {
    "id": "15",
    "startYearMonth": "2026-08",
    "endYearMonth": null,
    "isAvailable": false
  }
}
```

---

### 26.4 idの型

新規設定IDがDB上で`bigint`でも、

```json
{
  "id": "15"
}
```

のようにstringとして返却されることを確認する。

---

### 26.5 startYearMonth未指定

以下を送信する。

```json
{
  "isAvailable": false
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.6 startYearMonth = null

以下を送信する。

```json
{
  "startYearMonth": null,
  "isAvailable": false
}
```

バリデーションエラーとなり、DB更新されないことを確認する。

---

### 26.7 startYearMonth形式不正

以下を確認する。

```text
2026-8
2026/08
202608
2026-00
2026-13
abc
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.8 isAvailable未指定

以下を送信する。

```json
{
  "startYearMonth": "2026-08"
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.9 isAvailable = null

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": null
}
```

バリデーションエラーとなること。

---

### 26.10 isAvailableが文字列

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": "false"
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.11 isAvailableが数値

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": 0
}
```

boolean以外として拒否されることを確認する。

---

### 26.12 X-User-Id未指定

`X-User-Id`を指定しない。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

となること。

---

### 26.13 X-User-Id形式不正

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

### 26.14 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 26.15 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 26.16 assetAccountId形式不正

以下を確認する。

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

### 26.17 資産口座不存在

存在しない`assetAccountId`を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 26.18 他利用者の資産口座

User AとしてUser Bの資産口座を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの`asset_account_available_settings`が一切変更されないことを確認する。

---

### 26.19 論理削除済み資産口座

対象資産口座を

```text
deleted_at IS NOT NULL
```

とする。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

新しい設定が追加されないこと。

---

### 26.20 履歴0件

利用中資産口座に対して利用可能資産設定履歴を0件とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

となること。

設定を自動作成しないこと。

---

### 26.21 初期開始年月不整合

以下の状態を用意する。

```text
asset_accounts.start_year_month
    = 2026-01

最古設定.start_year_month
    = 2026-02
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 26.22 既存履歴の期間重複

以下を用意する。

```text
2026-01 ～ 2026-08
true

2026-08 ～ NULL
false
```

ACC-006を実行しても新設定を追加せず、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 26.23 既存履歴の期間欠落

以下を用意する。

```text
2026-01 ～ 2026-05
true

2026-07 ～ NULL
false
```

履歴不整合となること。

---

### 26.24 継続中設定0件

すべての設定に`end_year_month`が存在する状態を用意する。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 26.25 継続中設定複数件

以下を用意する。

```text
2026-01 ～ NULL
true

2026-08 ～ NULL
false
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

どちらか一方を任意に更新しないこと。

---

### 26.26 startYearMonthが資産口座利用開始年月より前

例えば、

```text
assetAccount.startYearMonth
    = 2026-08
```

に対して、

```json
{
  "startYearMonth": "2026-07",
  "isAvailable": false
}
```

を送信する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

となること。

---

### 26.27 startYearMonthが現在設定開始年月と同じ

現在設定：

```text
2026-08 ～ NULL
true
```

Request：

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

となること。

---

### 26.28 startYearMonthが現在設定より前

現在設定：

```text
2026-08 ～ NULL
false
```

Request：

```json
{
  "startYearMonth": "2026-05",
  "isAvailable": true
}
```

履歴途中への挿入として拒否されること。

---

### 26.29 1か月後から変更

現在設定：

```text
2026-08 ～ NULL
true
```

Request：

```json
{
  "startYearMonth": "2026-09",
  "isAvailable": false
}
```

正常に、

```text
2026-08 ～ 2026-08
true

2026-09 ～ NULL
false
```

となること。

---

### 26.30 年跨ぎ

現在設定：

```text
2026-01 ～ NULL
true
```

Request：

```json
{
  "startYearMonth": "2027-01",
  "isAvailable": false
}
```

旧設定終了年月が

```text
2026-12
```

となること。

---

### 26.31 同じisAvailable

現在設定：

```text
2026-01 ～ NULL
true
```

Request：

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

となること。

以下を確認する。

* 旧設定を更新しない
* 新設定を追加しない
* 履歴件数が変化しない

---

### 26.32 同一startYearMonth重複

同一資産口座に

```text
start_year_month = 2026-08
```

の設定が既に存在する状態で、同じ開始年月の登録を試みる。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

または、より前の業務チェックで`START_YEAR_MONTH_INVALID`となる場合は、チェック順序に従った結果となること。

---

### 26.33 UNIQUE制約

DBレベルで、

```text
asset_account_id
+
start_year_month
```

が重複できないことを確認する。

並行実行などで制約違反が発生した場合は、PostgreSQL例外がAPIへそのまま露出しないこと。

---

### 26.34 継続中設定部分UNIQUE制約

部分UNIQUEインデックスを採用する場合は、同一資産口座に

```text
end_year_month IS NULL
```

の設定を2件作成できないことを確認する。

---

### 26.35 旧設定UPDATE

正常系で、現在設定の

```text
end_year_month
```

だけが新設定開始年月の前月へ更新されることを確認する。

以下は変更されないこと。

```text
start_year_month
is_available
asset_account_id
```

---

### 26.36 新設定INSERT

正常系で、新設定が以下の値で登録されることを確認する。

```text
asset_account_id
    = 対象資産口座ID

start_year_month
    = Request値

end_year_month
    = NULL

is_available
    = Request値
```

---

### 26.37 旧設定更新後に新設定INSERT失敗

新設定INSERTで意図的に例外を発生させる。

以下を確認する。

* APIはエラーとなる
* トランザクションがロールバックされる
* 旧設定の`end_year_month`が元の`NULL`へ戻る
* 新設定が存在しない

---

### 26.38 旧設定UPDATE失敗

旧設定UPDATEで例外を発生させる。

以下を確認する。

* 新設定をINSERTしない
* トランザクションがロールバックされる
* 履歴状態が実行前と同じ

---

### 26.39 Transaction全体

正常時は、

```text
旧設定UPDATE
+
新設定INSERT
```

の両方がコミットされること。

異常時は、両方とも確定しないことを確認する。

---

### 26.40 同時実行

同一資産口座に対して複数のACC-006を並行実行する。

以下を確認する。

* 現在設定への`lockForUpdate()`が機能する
* 同時に同じ旧設定を更新しない
* 継続中設定が複数件にならない
* 履歴期間が重複しない
* 2件目は最新状態で業務条件を再評価する

---

### 26.41 同時に異なる開始年月

例えば、

```text
Request A
2026-08 / false

Request B
2026-09 / false
```

を並行実行する。

Request A成功後、Request Bが最新状態を基準に再判定されることを確認する。

不要な同一状態履歴が作成されないこと。

---

### 26.42 同時に同じ開始年月

以下を並行実行する。

```text
Request A
2026-08 / false

Request B
2026-08 / false
```

以下を確認する。

* 同じ開始年月の設定が2件作成されない
* 一方のみ成功すること
* 競合側が業務エラーへ変換されること
* DB制約違反が外部へ露出しないこと

---

### 26.43 他の関連テーブル非更新

ACC-006実行前後で、以下が変更されないことを確認する。

* `asset_accounts`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

---

### 26.44 過去の月末資産非更新

利用可能資産設定変更によって、既存の

```text
month_end_asset_balances
month_end_holding_values
month_end_asset_snapshots
```

が更新されないことを確認する。

---

### 26.45 過去判定履歴非更新

ACC-006実行によって、

```text
assessment_histories
```

の既存レコードが変更されないことを確認する。

---

### 26.46 更新対象外項目

以下のようなRequestを送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false,
  "endYearMonth": "2026-12"
}
```

未定義項目を拒否する共通方針の場合は、

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.47 currentSettingIdを送信

以下を送信する。

```json
{
  "currentSettingId": "10",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

Request指定のIDによって更新対象を変更できないことを確認する。

---

### 26.48 userIdを送信

以下を送信する。

```json
{
  "userId": "999",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

`user_id`が変更されず、別利用者の資産口座を操作できないことを確認する。

---

### 26.49 assetAccountIdをBodyへ送信

以下を送信する。

```json
{
  "assetAccountId": "999",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

URLの`assetAccountId`以外を対象決定に使用しないことを確認する。

---

### 26.50 正常レスポンス契約

正常時に、以下の形式となることを確認する。

```json
{
  "data": {
    "id": "15",
    "startYearMonth": "2026-08",
    "endYearMonth": null,
    "isAvailable": false
  }
}
```

以下を確認する。

* HTTPステータスが`201 Created`
* `id`がstring
* `startYearMonth`が`YYYY-MM`
* `endYearMonth = null`
* `isAvailable`がboolean

---

### 26.51 返却しない情報

正常レスポンスへ、以下が含まれないことを確認する。

* `assetAccountId`
* `asset_account_id`
* `userId`
* `user_id`
* `createdAt`
* `created_at`
* `updatedAt`
* `updated_at`
* 旧設定情報
* 資産口座名
* 資産種別
* 残高記録単位
* `isEnabled`

---

### 26.52 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
INTERNAL_SERVER_ERROR
```

以下も確認する。

* `error.code`が設定されること
* `error.message`が設定されること
* 必要に応じて`error.details`が設定されること
* `requestId`が設定されること
* SQLが含まれないこと
* SQLSTATEが含まれないこと
* PostgreSQL内部エラーが含まれないこと
* 制約名が含まれないこと
* スタックトレースが含まれないこと

---

### 26.53 エラー時の副作用

業務エラー発生時に、

```text
asset_account_available_settings
```

が中途半端に変更されないことを確認する。

特に、

```text
旧設定終了済み
+
新設定なし
```

や、

```text
旧設定継続中
+
新設定追加済み
```

という状態を残さないこと。

---

### 26.54 INTERNAL_SERVER_ERROR

更新処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

* トランザクションがロールバックされること
* 内部情報がレスポンスへ公開されないこと
* サーバーログに調査情報が記録されること
* レスポンスの`requestId`からログを追跡できること

---

## 27. Laravel実装方針

ACC-006では、Action、FormRequest、UseCase、Query、Repository、Validator、DTO、API Resource、Responderを分離して実装する。

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
    ├─ AvailableSettingHistoryValidator
    └─ AssetAccountAvailableSettingRepository
    ↓
Create Result DTO
    ↓
API Resource
    ↓
Responder
```

ACC-006は、

```text
旧設定.end_year_month更新
+
新設定INSERT
```

という複数レコード更新を1つの業務操作として扱うため、UseCase全体をトランザクション境界とする。

---

### 27.1 Route

ACC-006は、以下のルートとして定義する。

概念例：

```php
Route::post(
    '/api/v1/asset-accounts/{assetAccountId}/available-settings',
    CreateAssetAccountAvailableSettingAction::class,
);
```

ACC-005とは同じURLを使用し、HTTPメソッドによって責務を分離する。

```text
GET
/api/v1/asset-accounts/{assetAccountId}/available-settings
    → ACC-005
      利用可能資産設定履歴取得

POST
/api/v1/asset-accounts/{assetAccountId}/available-settings
    → ACC-006
      利用可能資産設定登録
```

---

### 27.2 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、

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

とする。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 27.3 FormRequest

ACC-006では、リクエストボディの入力値検証に専用FormRequestを使用する。

概念例：

```php
final class CreateAssetAccountAvailableSettingRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'startYearMonth' => [
                'required',
                'string',
                'date_format:Y-m',
            ],

            'isAvailable' => [
                'required',
                'boolean',
            ],
        ];
    }
}
```

ただし、Laravelの`boolean`ルールで許容される値がAPI契約より広い場合は、JSON booleanのみを許容する専用Ruleまたは追加検証を使用する。

---

### 27.4 JSON booleanを厳密に扱う

ACC-006では、

```text
true
false
```

だけを`isAvailable`として許可する。

以下は受け付けない。

```text
"true"
```

```text
"false"
```

```text
1
```

```text
0
```

Laravel標準の`boolean`ルールがこれらを許容する場合は、例えば専用Ruleを使用する。

概念例：

```php
final class StrictBoolean implements ValidationRule
{
    public function validate(
        string $attribute,
        mixed $value,
        Closure $fail,
    ): void {
        if (! is_bool($value)) {
            $fail(
                'booleanで指定してください。',
            );
        }
    }
}
```

---

### 27.5 startYearMonthの形式検証

`startYearMonth`は、厳密な

```text
YYYY-MM
```

形式として検証する。

例えば、

```text
2026-08
```

は正常とする。

以下は不正とする。

```text
2026-8
2026/08
202608
2026-00
2026-13
```

見た目だけでなく、実在する年月であることも確認する。

---

### 27.6 FormRequestで行うこと

FormRequestでは、HTTP入力として判断できる以下を検証する。

```text
startYearMonth
isAvailable
```

主に以下とする。

- 必須
- NULL禁止
- 型
- `YYYY-MM`形式
- 実在年月
- JSON boolean
- 未定義項目

---

### 27.7 FormRequestで行わないこと

FormRequestでは、以下のDB状態に依存する業務ルールを扱わない。

- 資産口座存在確認
- 利用者境界確認
- 論理削除判定
- 既存履歴0件判定
- 履歴整合性判定
- 現在設定特定
- 開始年月の業務妥当性
- 現在設定との`isAvailable`比較
- 行ロック
- DB更新

これらは、UseCase、Query、Validator、Repositoryで扱う。

---

### 27.8 更新対象外項目

ACC-006では、以下をRequest項目として定義しない。

```text
id
userId
assetAccountId
settingId
currentSettingId
endYearMonth
createdAt
updatedAt
```

API共通方針として未定義項目を拒否する場合は、これらが送信された時点で

```text
VALIDATION_ERROR
```

とする。

---

### 27.9 assetAccountIdの形式検証

`assetAccountId`は、ルート制約またはAPI共通のパスパラメータ検証方式で検証する。

概念例：

```php
Route::post(
    '/api/v1/asset-accounts/{assetAccountId}/available-settings',
    CreateAssetAccountAvailableSettingAction::class,
)
    ->where(
        'assetAccountId',
        '[1-9][0-9]*',
    );
```

形式不正は、

```text
INVALID_ASSET_ACCOUNT_ID
```

へ変換する。

---

### 27.10 Action

Actionは、検証済みRequest、`assetAccountId`、利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php
final class CreateAssetAccountAvailableSettingAction
{
    public function __invoke(
        string $assetAccountId,
        CreateAssetAccountAvailableSettingRequest $request,
        CreateAssetAccountAvailableSettingUseCase $useCase,
        CreateAssetAccountAvailableSettingResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $input =
            new CreateAssetAccountAvailableSettingInput(
                userId:
                    $userContext->userId,

                assetAccountId:
                    (int) $assetAccountId,

                startYearMonth:
                    $request->string(
                        'startYearMonth',
                    )->toString(),

                isAvailable:
                    $request->boolean(
                        'isAvailable',
                    ),
            );

        $result =
            $useCase->execute(
                $input,
            );

        return $responder->created(
            $result,
        );
    }
}
```

---

### 27.11 Input DTO

ActionからUseCaseへは、専用Input DTOを渡す。

概念例：

```php
final readonly class
    CreateAssetAccountAvailableSettingInput
{
    public function __construct(
        public int $userId,
        public int $assetAccountId,
        public string $startYearMonth,
        public bool $isAvailable,
    ) {
    }
}
```

RequestオブジェクトそのものをUseCaseへ渡さない。

---

### 27.12 Actionで行わないこと

Actionでは、以下を行わない。

- 資産口座検索
- 利用者境界判定
- 履歴取得
- 履歴整合性判定
- 現在設定取得
- 前月計算
- `lockForUpdate()`
- UPDATE
- INSERT
- トランザクション制御
- レスポンス配列生成

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 27.13 UseCase

ACC-006の業務処理全体を担当する。

主な処理は、以下とする。

1. Input DTOを受け取る
2. トランザクションを開始する
3. 対象資産口座を取得する
4. 対象不存在の場合は業務例外を送出する
5. 既存履歴を取得する
6. 履歴0件を確認する
7. 既存履歴整合性を検証する
8. 現在設定を`lockForUpdate()`で取得する
9. ロック後の最新状態を基準に業務条件を再確認する
10. 新設定開始年月の妥当性を確認する
11. `isAvailable`が現在設定と異なることを確認する
12. 旧設定終了年月を算出する
13. 旧設定を更新する
14. 新設定を登録する
15. Result DTOを生成する
16. コミットする
17. Result DTOを返却する

---

### 27.14 UseCaseの概念フロー

概念的には、以下とする。

```text
Input DTO
    ↓
DB::transaction
    ↓
AssetAccountQuery
    ↓
資産口座取得
    ↓
AvailableSettingQuery
    ↓
既存履歴取得
    ↓
HistoryValidator
    ↓
履歴整合性確認
    ↓
AvailableSettingQuery
    ↓
現在設定lockForUpdate
    ↓
最新状態再確認
    ↓
開始年月チェック
    ↓
isAvailable差分チェック
    ↓
前月算出
    ↓
Repository
    ├─ 旧設定終了
    └─ 新設定登録
    ↓
Result DTO
```

---

### 27.15 DB::transaction

ACC-006では、UseCase全体を

```php
DB::transaction()
```

で囲む。

概念例：

```php
return DB::transaction(
    function () use (
        $input,
    ): CreateAssetAccountAvailableSettingResult {
        // 業務処理
    },
);
```

旧設定UPDATEと新設定INSERTを原子的に扱う。

---

### 27.16 AssetAccountQuery

対象資産口座取得は、Queryへ委譲する。

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

取得条件は、

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

とする。

---

### 27.17 利用者境界をQueryへ含める

以下のようなID単独取得を基本としない。

```php
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
利用中状態
```

を条件へ含める。

---

### 27.18 SoftDeletes

`AssetAccount` Modelでは、

```php
use SoftDeletes;
```

を使用する。

ACC-006では論理削除済み資産口座を対象としないため、

```php
withTrashed()
```

を使用しない。

---

### 27.19 ASSET_ACCOUNT_NOT_FOUND

対象資産口座を取得できない場合は、

```php
throw new
    AssetAccountNotFoundException();
```

とする。

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

### 27.20 AssetAccountAvailableSettingQuery

既存履歴取得は、専用Queryへ委譲する。

概念例：

```php
$settings =
    $this->availableSettingQuery
        ->findHistoryByAssetAccount(
            assetAccountId:
                $assetAccount->id,
        );
```

履歴検証しやすいように、

```text
start_year_month ASC
```

で取得してよい。

---

### 27.21 履歴0件

履歴が0件の場合は、正常状態としない。

概念例：

```php
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingHistoryNotFoundException();
}
```

最終的に、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

へ変換する。

---

### 27.22 AvailableSettingHistoryValidator

既存履歴の整合性確認には、ACC-005と同じValidatorを再利用する。

概念的には、

```php
$this->historyValidator
    ->validate(
        assetAccountStartYearMonth:
            $assetAccount->start_year_month,

        settings:
            $settings,
    );
```

とする。

同じ業務ルールをACC-005とACC-006で重複実装しない。

---

### 27.23 Validatorの確認内容

主に以下を確認する。

```text
最古設定開始年月
    =
assetAccount.startYearMonth

各設定
startYearMonth <= endYearMonth

期間重複なし

期間欠落なし

継続中設定1件

最新設定
endYearMonth = null
```

---

### 27.24 ValidatorでDB更新しない

`AvailableSettingHistoryValidator`は、整合性判定だけを行う。

以下を行わない。

- DB検索
- UPDATE
- INSERT
- 自動修復
- ロック

QueryとRepositoryの責務を持たせない。

---

### 27.25 現在設定をロック付きで取得する

履歴整合性確認後、現在設定をトランザクション内で再取得する。

概念例：

```php
$currentSetting =
    $this->availableSettingQuery
        ->findCurrentForUpdate(
            assetAccountId:
                $assetAccount->id,
        );
```

Query内部では、

```php
->where(
    'asset_account_id',
    $assetAccountId,
)
->whereNull(
    'end_year_month',
)
->lockForUpdate()
```

を使用する。

---

### 27.26 現在設定は1件を前提とする

`findCurrentForUpdate()`では、継続中設定が1件のみであることを前提とする。

0件または複数件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ変換する。

単純に

```php
->first()
```

だけで複数件を黙って無視しない。

---

### 27.27 ロック後に再確認する

履歴整合性確認後からロック取得までの間に、別ACC-006が更新している可能性がある。

そのため、ロック取得後の現在設定を基準に、

```text
startYearMonth
isAvailable
```

の業務条件を再確認する。

---

### 27.28 startYearMonthの業務検証

新設定開始年月は、少なくとも

```text
request.startYearMonth
    >
currentSetting.startYearMonth
```

であることを確認する。

また、

```text
request.startYearMonth
    >=
assetAccount.startYearMonth
```

も満たすことを確認する。

---

### 27.29 開始年月不正

条件を満たさない場合は、

```php
throw new
    AssetAccountAvailableSettingStartYearMonthInvalidException();
```

とする。

最終的に、

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

へ変換する。

---

### 27.30 現在設定との差分確認

現在設定の

```text
is_available
```

とRequestの

```text
isAvailable
```

が異なることを確認する。

概念例：

```php
if (
    $currentSetting->is_available
    === $input->isAvailable
) {
    throw new
        AssetAccountAvailableSettingNoChangeException();
}
```

---

### 27.31 NO_CHANGE

現在設定と同じ状態の場合は、

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

へ変換する。

この場合、Repositoryを呼び出して旧設定を更新しない。

---

### 27.32 YearMonthの扱い

`startYearMonth`から前月を算出する必要がある。

年月計算を文字列操作で行わず、`CarbonImmutable`または`YearMonth` Value Objectを使用する。

概念例：

```php
$previousYearMonth =
    CarbonImmutable::createFromFormat(
        '!Y-m',
        $input->startYearMonth,
    )
        ->subMonth()
        ->format('Y-m');
```

---

### 27.33 年跨ぎを考慮する

例えば、

```text
2027-01
```

の前月は、

```text
2026-12
```

となる。

以下のような独自文字列演算は行わない。

```text
month - 1
```

---

### 27.34 YearMonth Value Object

年月処理が複数ユースケースへ広がる場合は、Value Objectを導入してよい。

概念例：

```php
final readonly class YearMonth
{
    public function __construct(
        public int $year,
        public int $month,
    ) {
    }

    public function previous(): self
    {
        // 前月返却
    }

    public function isAfter(
        self $other,
    ): bool {
        // 比較
    }

    public function toString(): string
    {
        return sprintf(
            '%04d-%02d',
            $this->year,
            $this->month,
        );
    }
}
```

Phase1では、過剰にならない範囲で採用する。

---

### 27.35 Repository

`asset_account_available_settings`の更新・登録は、Repositoryへ委譲する。

概念的なインターフェースは、以下とする。

```php
interface
    AssetAccountAvailableSettingRepository
{
    public function close(
        AssetAccountAvailableSetting $setting,
        string $endYearMonth,
    ): void;

    public function create(
        int $assetAccountId,
        string $startYearMonth,
        bool $isAvailable,
    ): AssetAccountAvailableSetting;
}
```

---

### 27.36 旧設定終了

Repositoryでは、現在設定の

```text
end_year_month
```

だけを更新する。

概念例：

```php
public function close(
    AssetAccountAvailableSetting $setting,
    string $endYearMonth,
): void {
    $setting->end_year_month =
        $endYearMonth;

    $setting->save();
}
```

以下を変更しない。

```text
asset_account_id
start_year_month
is_available
```

---

### 27.37 新設定登録

新設定は、明示的な値で登録する。

概念例：

```php
public function create(
    int $assetAccountId,
    string $startYearMonth,
    bool $isAvailable,
): AssetAccountAvailableSetting {
    $setting =
        new AssetAccountAvailableSetting();

    $setting->asset_account_id =
        $assetAccountId;

    $setting->start_year_month =
        $startYearMonth;

    $setting->end_year_month =
        null;

    $setting->is_available =
        $isAvailable;

    $setting->save();

    return $setting;
}
```

---

### 27.38 既存Modelを複製しない

以下のような現在設定Modelの複製をそのまま保存する方式は避ける。

```php
$newSetting =
    $currentSetting->replicate();
```

新設定の意味を明確にするため、必要項目を明示的に設定する。

---

### 27.39 Mass Assignmentを避ける

以下のような処理は行わない。

```php
AssetAccountAvailableSetting::create(
    $request->all(),
);
```

クライアントから指定可能な値とサーバー管理値を明確に分離する。

---

### 27.40 end_year_monthはサーバー管理とする

新設定では、

```text
end_year_month = null
```

とする。

旧設定の終了年月は、

```text
新設定開始年月の前月
```

としてサーバー側で計算する。

Request Body中の`endYearMonth`は使用しない。

---

### 27.41 asset_account_idはサーバー管理とする

新設定の

```text
asset_account_id
```

は、利用者境界確認済みの資産口座IDを使用する。

Request Bodyから受け取らない。

---

### 27.42 トランザクション内の更新順

概念的には、

```text
1. 現在設定lockForUpdate
2. 業務条件再確認
3. 旧設定close
4. 新設定create
```

の順とする。

旧設定終了前に新設定を作成して、一時的に継続中設定を複数作る構成は避ける。

---

### 27.43 UPDATE後にINSERT失敗した場合

新設定INSERTで例外が発生した場合は、

```text
旧設定.end_year_month更新
```

もトランザクションによってロールバックする。

UseCase外で別々にコミットしない。

---

### 27.44 UNIQUE制約

同一資産口座内では、

```text
asset_account_id
+
start_year_month
```

をUNIQUE制約とする。

アプリケーション側の業務チェックだけに依存しない。

---

### 27.45 継続中設定の部分UNIQUE制約

PostgreSQLでは、必要に応じて

```text
asset_account_id
WHERE end_year_month IS NULL
```

の部分UNIQUEインデックスを使用する。

概念例：

```sql
CREATE UNIQUE INDEX
    uq_asset_account_available_settings_current
ON
    asset_account_available_settings (
        asset_account_id
    )
WHERE
    end_year_month IS NULL;
```

ACC-006の`lockForUpdate()`と合わせて、DB側でも整合性を保証する。

---

### 27.46 UNIQUE制約違反の変換

同一開始年月のUNIQUE制約違反が発生した場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

へ変換する。

概念的には、

```text
PostgreSQL unique violation
    ↓
AssetAccountAvailableSettingAlreadyExistsException
    ↓
409 Conflict
```

とする。

SQLSTATEや制約名はクライアントへ公開しない。

---

### 27.47 部分UNIQUE制約違反

継続中設定の部分UNIQUE制約違反が発生した場合は、同時更新競合または履歴不整合として扱う。

API共通方針に応じて、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

または競合系エラーへ変換する。

Phase1では、内部DB例外をそのまま返却しないことを優先する。

---

### 27.48 lockForUpdate

現在設定取得には、

```php
lockForUpdate()
```

を使用する。

ただし、必ず

```php
DB::transaction()
```

内で使用する。

トランザクション外で形式的に呼び出すだけの実装にはしない。

---

### 27.49 ロック対象

原則として、現在継続中の利用可能資産設定1件だけをロックする。

履歴全件を`lockForUpdate()`しない。

また、利用者の全資産口座をロックしない。

---

### 27.50 ロック後の処理を短く保つ

ロック取得後は、

```text
最新状態確認
前月算出
旧設定UPDATE
新設定INSERT
```

を速やかに実行する。

以下をロック保持中に行わない。

- 外部API
- メール送信
- ファイル処理
- 大量データ取得
- レスポンス整形

---

### 27.51 Result DTO

新規登録した利用可能資産設定を、専用Result DTOで表現する。

概念例：

```php
final readonly class
    CreateAssetAccountAvailableSettingResult
{
    public function __construct(
        public int $id,
        public string $startYearMonth,
        public ?string $endYearMonth,
        public bool $isAvailable,
    ) {
    }
}
```

正常登録直後の`endYearMonth`は`null`となる。

---

### 27.52 DTOへEloquent Modelを保持しない

以下のようなResult DTOは基本としない。

```php
final readonly class
    CreateAssetAccountAvailableSettingResult
{
    public function __construct(
        public AssetAccountAvailableSetting $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

---

### 27.53 UseCaseの戻り値

概念例：

```php
return new
    CreateAssetAccountAvailableSettingResult(
        id:
            $newSetting->id,

        startYearMonth:
            $newSetting->start_year_month,

        endYearMonth:
            $newSetting->end_year_month,

        isAvailable:
            (bool) $newSetting->is_available,
    );
```

Eloquent ModelをActionへ直接返却しない。

---

### 27.54 API Resource

Result DTOを、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php
final class
    CreateAssetAccountAvailableSettingResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'startYearMonth'
                => $this->startYearMonth,

            'endYearMonth'
                => $this->endYearMonth,

            'isAvailable'
                => $this->isAvailable,
        ];
    }
}
```

API共通方針に従ってcamelCaseで返却する。

---

### 27.55 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

- `asset_account_id`
- `user_id`
- `created_at`
- `updated_at`
- 旧設定ID
- 旧設定`start_year_month`
- 旧設定`end_year_month`
- 旧設定`is_available`
- 資産口座名
- 資産種別
- 残高記録単位
- `deleted_at`

旧履歴を確認する場合は、ACC-005を使用する。

---

### 27.56 Responder

Responderは、Result DTOを`201 Created`へ変換する。

概念例：

```php
final class
    CreateAssetAccountAvailableSettingResponder
{
    public function created(
        CreateAssetAccountAvailableSettingResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    =>
                    new CreateAssetAccountAvailableSettingResource(
                        $result,
                    ),
            ],
            Response::HTTP_CREATED,
        );
    }
}
```

`requestId`等の共通Envelope項目は、API共通レスポンス処理に従う。

---

### 27.57 Locationヘッダー

`201 Created`では、新規リソースを示す`Location`ヘッダーを設定することもできる。

ただし、Phase1では利用可能資産設定の個別取得APIを用意していないため、ACC-006専用の`Location`ヘッダーは必須としない。

---

### 27.58 Responderで行わないこと

Responderでは、以下を行わない。

- 資産口座検索
- 履歴取得
- 履歴整合性判定
- 現在設定取得
- 行ロック
- 前月計算
- UPDATE
- INSERT
- トランザクション制御
- 業務例外判定

HTTPレスポンス生成だけに責務を限定する。

---

### 27.59 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `assetAccountId`形式不正 | `INVALID_ASSET_ACCOUNT_ID` |
| Request Body不正 | `VALIDATION_ERROR` |
| 資産口座不存在 | `ASSET_ACCOUNT_NOT_FOUND` |
| 他利用者の資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 論理削除済み資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 履歴0件 | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND` |
| 履歴不整合 | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID` |
| 開始年月不正 | `ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID` |
| 現在設定と同一状態 | `ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE` |
| 同一開始年月重複 | `ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

---

### 27.60 START_YEAR_MONTH_INVALID

開始年月の業務妥当性に違反した場合は、専用例外を送出する。

概念例：

```php
if (
    ! $newStartYearMonth
        ->isAfter(
            $currentStartYearMonth,
        )
) {
    throw new
        AssetAccountAvailableSettingStartYearMonthInvalidException();
}
```

最終的に、

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

へ変換する。

---

### 27.61 NO_CHANGE

現在設定と同じ`isAvailable`の場合は、

```php
throw new
    AssetAccountAvailableSettingNoChangeException();
```

とする。

最終的に、

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

へ変換する。

---

### 27.62 HISTORY_NOT_FOUND

履歴が0件の場合は、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

へ変換する。

ACC-002の初期設定登録ルールに反する状態として扱う。

---

### 27.63 HISTORY_INVALID

既存履歴整合性に違反した場合は、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ変換する。

内部的には、ACC-005と同じ`invalidReason`を保持してよい。

---

### 27.64 DB例外変換

PostgreSQLのDB例外は、制約名やSQLSTATEを判定材料として内部で利用してよい。

ただし、レスポンスへそのまま公開しない。

例えば、

```text
unique violation
    ↓
業務例外へ変換
```

とする。

---

### 27.65 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- SQL
- SQLSTATE
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

---

### 27.66 ログ

ACC-006では、必要に応じて以下をログコンテキストへ設定する。

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
ACC-006
```

とする。

---

### 27.67 正常時ログ

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = ACC-006
assetAccountId
newSettingId
startYearMonth
httpStatus = 201
```

必要であれば、

```text
previousIsAvailable
newIsAvailable
```

も内部構造化ログへ記録してよい。

---

### 27.68 履歴不整合ログ

以下の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

調査に必要な情報を記録する。

概念例：

```text
assetAccountId
historyCount
invalidReason
```

---

### 27.69 競合ログ

以下の場合は、必要に応じて競合として記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

同時実行が多発する場合に監視できる状態としてよい。

---

### 27.70 トランザクションロールバックログ

旧設定UPDATE後に新設定INSERTが失敗した場合などは、必要に応じて

```text
transactionRolledBack = true
```

相当の内部ログを記録してよい。

---

### 27.71 キャッシュ

Phase1では、ACC-006専用のサーバー側キャッシュを使用しない。

React側では、ACC-006成功後に少なくとも以下をinvalidateする。

```text
ACC-005
利用可能資産設定履歴

ACC-003
資産口座詳細
```

ACC-001が`isAvailable`を返却する場合は、一覧Queryもinvalidate対象とする。

---

### 27.72 テスト実装方針

Laravel側では、Feature Testを中心にAPI契約とトランザクション動作を確認する。

また、以下についてはUnit TestまたはDatabase Testも行う。

- FormRequest
- AssetAccountQuery
- AvailableSettingQuery
- HistoryValidator
- Repository
- UseCase
- API Resource

---

### 27.73 FormRequestのTest

以下を確認する。

```text
startYearMonth
    required
    string
    YYYY-MM
    実在年月

isAvailable
    required
    strict boolean
```

主な異常系：

```text
startYearMonth未指定
startYearMonth = null
2026-8
2026/08
2026-13
isAvailable未指定
isAvailable = null
"true"
1
0
```

---

### 27.74 AssetAccountQueryのDatabase Test

以下を確認する。

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

### 27.75 AvailableSettingQueryの履歴取得Test

対象資産口座について、履歴全件が取得できることを確認する。

```text
asset_account_id
    = assetAccountId
```

他資産口座の履歴が混在しないこと。

---

### 27.76 findCurrentForUpdateのTest

現在設定取得について、

```text
asset_account_id
    = assetAccountId

AND

end_year_month IS NULL
```

となる1件を取得できることを確認する。

Integration Testでは、可能であれば`FOR UPDATE`が発行されることも確認する。

---

### 27.77 HistoryValidatorの再利用Test

ACC-005とACC-006で同じ履歴整合性Validatorを利用する。

以下の同じ履歴に対して、両UseCaseで判定結果が一致することを確認する。

```text
初期開始年月不一致
期間重複
期間欠落
継続中設定0件
継続中設定複数件
最新設定終了済み
```

---

### 27.78 開始年月のUnit Test

現在設定が

```text
2026-08 ～ NULL
```

の場合を考える。

正常：

```text
2026-09
2026-12
2027-01
```

異常：

```text
2026-08
2026-07
2025-12
```

異常時に`StartYearMonthInvalidException`となることを確認する。

---

### 27.79 同一状態のUnit Test

現在設定：

```text
isAvailable = true
```

Request：

```text
isAvailable = true
```

の場合、

```text
AssetAccountAvailableSettingNoChangeException
```

となること。

Repositoryが呼ばれないことも確認する。

---

### 27.80 前月計算のUnit Test

以下を確認する。

```text
2026-08
    ↓
2026-07
```

```text
2027-01
    ↓
2026-12
```

```text
2026-01
    ↓
2025-12
```

年跨ぎを正しく扱えること。

---

### 27.81 RepositoryのDatabase Test

`close()`について、以下を確認する。

```text
end_year_month
    → 指定年月へ更新
```

以下は変更されないこと。

```text
asset_account_id
start_year_month
is_available
```

---

### 27.82 create()のDatabase Test

新設定が、以下で作成されることを確認する。

```text
asset_account_id
    = 指定ID

start_year_month
    = 指定年月

end_year_month
    = NULL

is_available
    = 指定boolean
```

---

### 27.83 UseCase正常系Test

Query、Validator、Repositoryを使用して、以下の順で処理されることを確認する。

```text
資産口座取得
    ↓
履歴取得
    ↓
履歴検証
    ↓
現在設定ロック
    ↓
最新状態検証
    ↓
旧設定close
    ↓
新設定create
    ↓
Result DTO
```

---

### 27.84 UseCaseのNO_CHANGE Test

現在設定とRequestの`isAvailable`が同一の場合は、

```text
NO_CHANGE
```

となり、

```text
Repository::close()
Repository::create()
```

が呼ばれないことを確認する。

---

### 27.85 UseCaseの開始年月不正Test

`startYearMonth`が現在設定開始年月以下の場合は、

```text
START_YEAR_MONTH_INVALID
```

となり、DB更新されないことを確認する。

---

### 27.86 トランザクションロールバックTest

旧設定更新後、新設定作成時に例外を発生させる。

以下を確認する。

```text
旧設定.end_year_month
    → 元のNULLへ戻る

新設定
    → 存在しない
```

---

### 27.87 UNIQUE制約Test

同一

```text
asset_account_id
+
start_year_month
```

で複数設定を登録できないことを確認する。

制約違反は、APIでは

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

へ変換されること。

---

### 27.88 継続中設定部分UNIQUE Test

部分UNIQUE制約を採用する場合は、同一資産口座に

```text
end_year_month IS NULL
```

の設定を2件作れないことを確認する。

---

### 27.89 同時実行Integration Test

可能であれば、同一資産口座に対するACC-006を並行実行する。

以下を確認する。

- `lockForUpdate()`によって直列化される
- 同じ現在設定を二重終了しない
- 継続中設定が複数にならない
- 同一開始年月が重複しない
- 後続Requestが最新状態を基準に再評価される

---

### 27.90 同時同一Request Test

以下を同時送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

以下を確認する。

- 1つの履歴だけが登録される
- 2件目は業務エラーとなる
- DB例外がそのまま返らない
- 履歴期間が壊れない

---

### 27.91 API ResourceのTest

正常時に、以下の項目だけを返却することを確認する。

```json
{
  "id": "15",
  "startYearMonth": "2026-08",
  "endYearMonth": null,
  "isAvailable": false
}
```

以下を返さないこと。

- `assetAccountId`
- `asset_account_id`
- `userId`
- `createdAt`
- `updatedAt`
- 旧設定情報
- DB内部情報

---

### 27.92 Feature Test

Feature Testでは、少なくとも以下を確認する。

```text
201 Created
400 Bad Request
404 Not Found
409 Conflict
500 Internal Server Error
```

主なケースは、

- 正常登録
- 利用者コンテキスト不正
- `assetAccountId`不正
- Request Body不正
- 他利用者資産口座
- 論理削除済み資産口座
- 履歴0件
- 履歴不整合
- 開始年月不正
- `NO_CHANGE`
- 同一開始年月重複
- トランザクションロールバック
- レスポンス契約

とする。

---

### 27.93 副作用範囲Test

ACC-006正常終了時に変更される業務データが、

```text
asset_account_available_settings

旧設定1件
    UPDATE

新設定1件
    INSERT
```

だけであることを確認する。

以下が変更されないことも確認する。

```text
asset_accounts
holding_assets
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
net_incomes
objectives
assessment_histories
```

---

## 28. React・TypeScriptでの利用

ACC-006は、指定した資産口座の利用可能資産区分を変更する際に使用する。

フロントエンドでは、ACC-003で現在状態を取得し、必要に応じてACC-005で利用可能資産設定履歴を確認したうえで、ACC-006をMutationとして実行する。

概念的な利用フローは、以下とする。

```text
ACC-003
資産口座詳細取得
    ↓
現在のisAvailable確認
    ↓
必要に応じてACC-005
利用可能資産設定履歴取得
    ↓
利用者が
startYearMonth
isAvailable
を入力
    ↓
ACC-006
POST
/api/v1/asset-accounts/{assetAccountId}/available-settings
    ↓
成功
    ↓
関連Query Cache無効化
    ↓
ACC-003 / ACC-005再取得
```

ACC-006は、サーバー状態を変更するため、TanStack Queryを使用する場合はMutationとして扱う。

---

### 28.1 TypeScript型

ACC-006のリクエスト型は、以下のように定義する。

概念例：

```typescript
export type CreateAssetAccountAvailableSettingRequest = {
  startYearMonth: string;
  isAvailable: boolean;
};
```

両項目とも必須とする。

---

### 28.2 正常レスポンス型

ACC-006成功時は、新しく登録された利用可能資産設定を返却する。

概念例：

```typescript
export type AssetAccountAvailableSetting = {
  id: string;
  startYearMonth: string;
  endYearMonth: string | null;
  isAvailable: boolean;
};
```

API共通Envelopeを使用する場合は、以下のように定義する。

```typescript
export type CreateAssetAccountAvailableSettingResponse =
  ApiResponse<AssetAccountAvailableSetting>;
```

---

### 28.3 ACC-005と共通型を使用してよい

ACC-005で返却する利用可能資産設定履歴1件と、ACC-006成功時に返却する新規設定の項目が同一である場合は、共通型を使用してよい。

概念例：

```typescript
export type AssetAccountAvailableSetting = {
  id: string;
  startYearMonth: string;
  endYearMonth: string | null;
  isAvailable: boolean;
};
```

ACC-005：

```typescript
export type AssetAccountAvailableSettingHistoryResponse =
  ApiResponse<
    AssetAccountAvailableSetting[]
  >;
```

ACC-006：

```typescript
export type CreateAssetAccountAvailableSettingResponse =
  ApiResponse<
    AssetAccountAvailableSetting
  >;
```

同じ意味のデータに対してAPIごとに不要な重複型を増やさない。

---

### 28.4 assetAccountId

対象資産口座IDは、API Client関数の引数として渡す。

概念例：

```typescript
createAssetAccountAvailableSetting(
  assetAccountId,
  request,
);
```

`assetAccountId`はAPI契約に合わせて`string`として扱う。

---

### 28.5 userIdをAPI Client引数へ含めない

以下のような関数にはしない。

```typescript
createAssetAccountAvailableSetting(
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

### 28.6 X-User-Id

`X-User-Id`は、ACC-006専用処理ではなく、共通API Clientから付与する。

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

各コンポーネントやHookから直接ヘッダーを生成しない。

---

### 28.7 フォーム型

利用可能資産設定登録フォームは、以下のように定義できる。

概念例：

```typescript
export type CreateAssetAccountAvailableSettingFormValues = {
  startYearMonth: string;
  isAvailable: boolean;
};
```

Request型とフォーム型が同一であれば、共通利用してもよい。

ただし、画面固有の入力状態が増える場合は分離する。

---

### 28.8 startYearMonth

`startYearMonth`は、

```text
YYYY-MM
```

形式の文字列として扱う。

HTMLの

```html
<input type="month">
```

を使用する場合は、通常

```text
YYYY-MM
```

形式で取得できる。

概念例：

```tsx
<input
  type="month"
  value={form.startYearMonth}
  onChange={(event) => {
    setForm({
      ...form,
      startYearMonth:
        event.target.value,
    });
  }}
/>
```

---

### 28.9 startYearMonthをDateへ変換して保持しなくてよい

API契約は

```text
YYYY-MM
```

であるため、フォームStateをJavaScriptの`Date`へ必ず変換する必要はない。

概念的には、

```typescript
startYearMonth: string;
```

として保持してよい。

タイムゾーンの影響を不要に持ち込まない。

---

### 28.10 isAvailable

`isAvailable`は、booleanとして保持する。

概念例：

```tsx
<select
  value={
    form.isAvailable
      ? 'true'
      : 'false'
  }
  onChange={(event) => {
    setForm({
      ...form,
      isAvailable:
        event.target.value === 'true',
    });
  }}
>
  <option value="true">
    利用可能
  </option>

  <option value="false">
    利用対象外
  </option>
</select>
```

Request送信時には、文字列ではなくbooleanへ変換する。

---

### 28.11 booleanを文字列のまま送信しない

HTMLフォームでは、`select`や`radio`から文字列として値を取得する場合がある。

以下のように、

```json
{
  "isAvailable": "false"
}
```

を送信しない。

送信時には、

```json
{
  "isAvailable": false
}
```

とする。

---

### 28.12 現在状態を初期表示する

ACC-003で取得した

```text
isAvailable
```

を使用して、現在の利用可能資産区分を表示する。

例えば、

```text
現在の設定
利用可能
```

または、

```text
現在の設定
利用対象外
```

とする。

ただし、ACC-006の新設定入力値を現在値でそのまま確定しない。

---

### 28.13 現在状態と異なる値だけ選択させてもよい

ACC-006では、現在設定と同じ`isAvailable`を指定すると

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

となる。

そのため、フロントエンドでは現在値と反対の選択肢だけを提示することもできる。

例えば、現在が

```text
isAvailable = true
```

なら、

```text
利用対象外へ変更
```

という操作ボタンとして表現してよい。

---

### 28.14 ただしサーバー側判定を残す

フロントエンドで同一状態を選択できないようにしても、サーバー側の

```text
NO_CHANGE
```

判定は必要とする。

別タブや別リクエストによって画面表示後に現在状態が変更される可能性があるためである。

---

### 28.15 現在設定開始年月

ACC-006の`startYearMonth`は、現在設定の開始年月より後である必要がある。

ただし、ACC-003では現在の利用可能資産設定の開始年月を返却しない設計であるため、必要な場合はACC-005の履歴を利用する。

概念的には、

```text
ACC-005
履歴先頭
    ↓
endYearMonth = null
    ↓
現在設定
    ↓
startYearMonth
```

とする。

---

### 28.16 ACC-005から現在設定を取得する

ACC-005は、

```text
startYearMonth DESC
```

で返却するため、正常状態では先頭レコードが継続中設定となる。

概念例：

```typescript
const currentSetting =
  settings?.find(
    (setting) =>
      setting.endYearMonth === null,
  );
```

API契約上、継続中設定は1件のみである。

---

### 28.17 現在設定をフロントエンドで最終保証しない

ACC-005の正常レスポンスから現在設定を表示できるが、ACC-006実行時の最終判定はバックエンド側で行う。

フロントエンドで取得した`currentSetting`を更新対象IDとしてRequestへ送信しない。

---

### 28.18 settingIdを送信しない

以下のようなRequestにはしない。

```typescript
type CreateRequest = {
  currentSettingId: string;
  startYearMonth: string;
  isAvailable: boolean;
};
```

ACC-006では、現在設定をサーバー側で特定する。

---

### 28.19 endYearMonthを送信しない

フォームに

```text
終了年月
```

を入力するUIは設けない。

旧設定の終了年月は、

```text
新設定開始年月の前月
```

としてサーバー側で算出する。

---

### 28.20 旧設定を編集させない

ACC-005で表示した過去の利用可能資産設定に対して、

```text
編集
削除
```

ボタンをPhase1では設けない。

ACC-006は、現在履歴の後ろへ新しい設定を追加する操作に限定する。

---

### 28.21 API Client

ACC-006を呼び出す専用API Client関数を定義する。

概念例：

```typescript
export const createAssetAccountAvailableSetting =
  async (
    assetAccountId: string,
    request:
      CreateAssetAccountAvailableSettingRequest,
  ): Promise<
    AssetAccountAvailableSetting
  > => {
    const response =
      await apiClient.post<
        CreateAssetAccountAvailableSettingResponse
      >(
        `/api/v1/asset-accounts/${assetAccountId}/available-settings`,
        request,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

### 28.22 Mutationとして扱う

ACC-006はサーバー状態を変更するため、TanStack QueryではMutationとして扱う。

概念例：

```typescript
export type CreateAssetAccountAvailableSettingVariables = {
  assetAccountId: string;
  request:
    CreateAssetAccountAvailableSettingRequest;
};
```

---

### 28.23 Mutation Hook

概念例：

```typescript
export const useCreateAssetAccountAvailableSetting =
  () => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn: (
        variables:
          CreateAssetAccountAvailableSettingVariables,
      ) =>
        createAssetAccountAvailableSetting(
          variables.assetAccountId,
          variables.request,
        ),

      onSuccess: async (
        _result,
        variables,
      ) => {
        await queryClient
          .invalidateQueries({
            queryKey:
              assetAccountKeys
                .availableSettings(
                  variables.assetAccountId,
                ),
          });

        await queryClient
          .invalidateQueries({
            queryKey:
              assetAccountKeys.detail(
                variables.assetAccountId,
              ),
          });
      },

      retry: false,
    });
  };
```

---

### 28.24 ACC-005のQuery Cacheを無効化する

ACC-006成功時は、

```text
旧設定.endYearMonth更新
+
新設定追加
```

が発生するため、ACC-005の履歴Cacheは必ず古くなる。

そのため、

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys
      .availableSettings(
        assetAccountId,
      ),
});
```

を実行する。

---

### 28.25 ACC-003のQuery Cacheを無効化する

ACC-006によって現在の

```text
isAvailable
```

が変化する可能性があるため、ACC-003の詳細Queryも無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

---

### 28.26 ACC-001のQuery Cache

ACC-001 資産口座一覧取得が`isAvailable`を返却する設計の場合は、一覧Queryも無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.all,
});
```

ACC-001が利用可能資産区分を返却しない場合は、必須ではない。

---

### 28.27 将来開始設定でも履歴Cacheは無効化する

例えば、現在年月が

```text
2026-08
```

で、

```json
{
  "startYearMonth": "2026-10",
  "isAvailable": false
}
```

を登録した場合でも、履歴は即座に変化する。

そのため、ACC-005は必ず再取得する。

---

### 28.28 将来開始設定とACC-003

将来開始設定の場合、現在時点の`isAvailable`は変化しないことがある。

例えば、

```text
現在年月
2026-08

旧設定
2026-01 ～ 2026-09
true

新設定
2026-10 ～ NULL
false
```

では、ACC-003の現在値は引き続き`true`となる。

React側で独自に現在値を変更せず、ACC-003再取得結果をそのまま使用する。

---

### 28.29 成功レスポンスだけで履歴を再構築しない

ACC-006成功レスポンスでは、新規設定だけが返却される。

旧設定の更新後状態は返却されない。

そのため、以下のような履歴Cacheの手動再構築はPhase1では基本としない。

```typescript
queryClient.setQueryData(
  key,
  (old) => {
    // 旧設定endYearMonthを計算
    // 新設定を追加
  },
);
```

ACC-005を再取得してサーバー確定状態を使用する。

---

### 28.30 Optimistic Update

Phase1では、ACC-006に対するOptimistic Updateを必須としない。

ACC-006は、

- 履歴整合性
- 開始年月
- 現在設定
- `NO_CHANGE`
- 同時更新
- UNIQUE制約
- 行ロック

などのサーバー側判定を伴うためである。

---

### 28.31 Mutationの自動Retry

ACC-006では、Mutationの無条件な自動Retryを行わない。

概念例：

```typescript
useMutation({
  mutationFn:
    createAssetAccountAvailableSetting,
  retry: false,
});
```

ACC-006はPOSTであり、通信結果不明時に同じRequestを自動的に再送しない。

---

### 28.32 通信結果不明時

ACC-006送信後に通信エラーとなり、サーバー側で成功したか判断できない場合は、

```text
ACC-005
履歴再取得

+
ACC-003
現在状態再取得
```

によって現在状態を確認する。

同一Requestを即座に自動再送しない。

---

### 28.33 二重送信防止

Mutation実行中は、登録ボタンを非活性化する。

概念例：

```tsx
<button
  type="submit"
  disabled={
    mutation.isPending
  }
>
  {mutation.isPending
    ? '変更中...'
    : '設定を変更'}
</button>
```

ただし、フロントエンド側の二重送信防止だけを整合性保証にはしない。

サーバー側でも

```text
lockForUpdate
UNIQUE制約
業務状態再確認
```

を行う。

---

### 28.34 フォーム送信

概念例：

```typescript
const mutation =
  useCreateAssetAccountAvailableSetting();

const handleSubmit =
  (
    values:
      CreateAssetAccountAvailableSettingFormValues,
  ): void => {
    mutation.mutate({
      assetAccountId,
      request: {
        startYearMonth:
          values.startYearMonth,

        isAvailable:
          values.isAvailable,
      },
    });
  };
```

---

### 28.35 クライアント側バリデーション

送信前に、最低限以下を確認してよい。

- `startYearMonth`が入力されている
- `YYYY-MM`形式である
- `isAvailable`がbooleanとして確定している

ただし、サーバー側バリデーションを省略しない。

---

### 28.36 現在設定開始年月との簡易チェック

ACC-005から現在設定を取得済みであれば、

```text
request.startYearMonth
    >
currentSetting.startYearMonth
```

をフロントエンドでも事前確認してよい。

これにより、明らかに不正な入力を送信前に防げる。

---

### 28.37 業務判定の最終保証はサーバー側とする

フロントエンドで開始年月をチェックしても、ACC-006実行直前に別リクエストで現在設定が変わる可能性がある。

そのため、

```text
START_YEAR_MONTH_INVALID
```

の最終判定はLaravel側で行う。

---

### 28.38 NO_CHANGEの事前抑止

現在の`isAvailable`が取得できている場合は、同じ状態を選択できないようにしてよい。

例えば、

```tsx
<option
  value="true"
  disabled={
    currentIsAvailable === true
  }
>
  利用可能
</option>
```

ただし、バックエンドの`NO_CHANGE`判定は残す。

---

### 28.39 成功時

ACC-006成功時は、

```text
201 Created
```

となる。

例えば、

```json
{
  "data": {
    "id": "15",
    "startYearMonth": "2026-08",
    "endYearMonth": null,
    "isAvailable": false
  }
}
```

が返却される。

---

### 28.40 成功メッセージ

正常終了後は、必要に応じて

```text
利用可能資産設定を変更しました。
```

などの完了メッセージを表示する。

---

### 28.41 成功後の画面

成功後は、画面設計に応じて

- 設定画面に留まる
- 資産口座詳細画面へ戻る
- 履歴一覧を再表示する

などの動作とする。

Phase1では、

```text
Mutation成功
    ↓
ACC-003 / ACC-005再取得
    ↓
最新状態表示
```

という構成としてよい。

---

### 28.42 VALIDATION_ERROR

Laravel側から

```text
VALIDATION_ERROR
```

が返却された場合は、`error.details`を利用して対応するフォーム項目へエラーを表示する。

主な対象は、

```text
startYearMonth
isAvailable
```

とする。

---

### 28.43 startYearMonthのフィールドエラー

例えば、

```text
適用開始年月を入力してください。
```

または、

```text
適用開始年月はYYYY-MM形式で入力してください。
```

などを入力欄付近へ表示する。

具体的な文言は、API共通エラー表示方針に従う。

---

### 28.44 isAvailableのフィールドエラー

通常のUIでは、boolean以外を送信しない構成にする。

それでも

```text
VALIDATION_ERROR
```

となった場合は、設定フォーム全体のエラーとして扱ってもよい。

---

### 28.45 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

となる。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

フロントエンドでは理由を推測せず、

```text
指定された資産口座が見つかりません。
```

などの共通表示とする。

---

### 28.46 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-006専用フォームエラーにはしない。

---

### 28.47 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、利用者コンテキストに関する共通エラーとして扱う。

---

### 28.48 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択中の利用者が有効ではない状態として共通処理する。

---

### 28.49 INVALID_ASSET_ACCOUNT_ID

通常画面ではACC-001やACC-003から取得した有効なIDを使用するため、発生頻度は低い。

発生した場合は、不正なURLまたは画面状態として扱う。

---

### 28.50 HISTORY_NOT_FOUND

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

は、サーバー側の業務データ不整合として扱う。

フロントエンドで初期設定を補完しない。

例えば、

```text
利用可能資産設定を
変更できませんでした。
```

などを表示する。

---

### 28.51 HISTORY_INVALID

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

の場合も、フロントエンドで履歴を補正しない。

以下を行わない。

- 期間重複の解消
- 欠落期間の補完
- 継続中設定の選択
- 終了年月の自動修正

一般的な設定変更失敗として扱う。

---

### 28.52 START_YEAR_MONTH_INVALID

以下の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

となる。

主に、

- 現在設定開始年月と同じ
- 現在設定開始年月より前
- 資産口座利用開始年月より前
- 履歴途中への挿入

などである。

フロントエンドでは、

```text
適用開始年月を確認してください。
```

などを表示し、`startYearMonth`入力欄へ関連付けてよい。

---

### 28.53 NO_CHANGE

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

の場合は、現在設定と同じ状態が指定されている。

例えば、

```text
現在の利用可能資産区分と
同じ設定です。
```

などを表示してよい。

---

### 28.54 NO_CHANGE受信後は再取得してよい

画面上では異なる状態を選択したつもりでも、別リクエストによって現在状態が変化していた可能性がある。

そのため、`NO_CHANGE`受信後に

```text
ACC-003
ACC-005
```

を再取得して最新状態を表示してよい。

---

### 28.55 ALREADY_EXISTS

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

の場合は、同じ開始年月の設定が既に存在する。

例えば、

```text
指定した適用開始年月の設定は
すでに登録されています。
```

などを表示する。

ただし、同時更新によって発生した可能性もあるため、履歴を再取得してよい。

---

### 28.56 409 Conflict受信後

以下の409系エラーでは、

```text
START_YEAR_MONTH_INVALID
NO_CHANGE
ALREADY_EXISTS
```

必要に応じてACC-005を再取得する。

現在の履歴状態が画面表示時から変化している可能性があるためである。

---

### 28.57 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
利用可能資産設定を
変更できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

### 28.58 エラー時のフォーム値

ACC-006が失敗した場合は、原則として利用者が入力した

```text
startYearMonth
isAvailable
```

を保持する。

入力内容を自動的に初期化しない。

---

### 28.59 再取得後にフォーム値を見直す

409系エラーなどでACC-005を再取得した場合、現在設定が変化している可能性がある。

その場合は、現在のフォーム値が最新状態に対して有効かを再評価する。

---

### 28.60 過去履歴編集UIを設けない

ACC-005で表示した過去履歴に対して、

```text
編集
削除
```

の操作をACC-006へ結び付けない。

ACC-006は新しい設定追加専用とする。

---

### 28.61 資産口座編集UIとの分離

資産口座詳細画面で通常属性編集と利用可能資産設定変更を同時に表示する場合でも、内部的には別Mutationとする。

```text
name / assetType
    ↓
ACC-004

isAvailable変更
    ↓
ACC-006
```

1つのRequestへ統合しない。

---

### 28.62 資産口座無効化との分離

同じ画面に資産口座無効化操作があっても、

```text
通常属性更新
    → ACC-004

利用可能資産設定変更
    → ACC-006

資産口座無効化
    → 無効化API
```

として別操作とする。

---

### 28.63 Pageの責務

資産口座詳細または利用可能資産設定Pageでは、主に以下を担当する。

- URLから`assetAccountId`取得
- ACC-003による現在状態取得
- ACC-005による履歴取得
- ACC-006成功後の画面更新
- ページ単位のエラー表示

HTTP通信の詳細はAPI ClientやHookへ委譲する。

---

### 28.64 Form Componentの責務

利用可能資産設定フォームでは、主に以下を担当する。

- `startYearMonth`入力
- `isAvailable`選択
- クライアント側入力チェック
- フィールドエラー表示
- submitイベント通知

API Clientを直接呼び出さない構成としてよい。

---

### 28.65 Mutation Hookの責務

Mutation Hookでは、主に以下を担当する。

```text
ACC-006実行
Mutation状態管理
成功時Query invalidate
```

画面固有の表示レイアウトや業務文言をMutation Hookへ持たせない。

---

### 28.66 API Clientの責務

API Clientでは、

```http
POST /api/v1/asset-accounts/{assetAccountId}/available-settings
```

のHTTP通信と型付きレスポンス取得を担当する。

以下をAPI Clientへ含めない。

- Toast表示
- 画面遷移
- 現在設定表示
- 履歴期間表示
- 業務エラー文言生成

---

### 28.67 QueryとMutationを分離する

利用可能資産設定では、

```text
ACC-005
    → Query

ACC-006
    → Mutation
```

とする。

同じHookで取得と更新をまとめすぎない。

---

### 28.68 Query Key

例えば、以下のように共通定義する。

```typescript
export const assetAccountKeys = {
  all: [
    'assetAccounts',
  ] as const,

  detail: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
    ] as const,

  availableSettings: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
      'availableSettings',
    ] as const,
};
```

---

### 28.69 利用者切替時

Phase1では、`X-User-Id`によって操作対象利用者を切り替える。

利用者切替時に、前利用者の

```text
ACC-003
ACC-005
```

のQuery Cacheを新しい利用者へ誤表示しないようにする。

---

### 28.70 Query KeyへuserIdを含めてもよい

利用者境界をQuery Cache上でも明確にする場合は、

```typescript
[
  'assetAccounts',
  userId,
  assetAccountId,
  'availableSettings',
]
```

のように`userId`を含めてよい。

正式なQuery Key方針は、Reactアーキテクチャ設計に従う。

---

### 28.71 概念的なディレクトリ構成

例えば、以下のように整理できる。

```text
features/
└── asset-accounts/
    ├── api/
    │   ├── getAssetAccountDetail.ts
    │   ├── getAssetAccountAvailableSettings.ts
    │   └── createAssetAccountAvailableSetting.ts
    ├── components/
    │   ├── AssetAccountDetail.tsx
    │   ├── AvailableSettingHistory.tsx
    │   └── AvailableSettingForm.tsx
    ├── hooks/
    │   ├── useAssetAccountDetail.ts
    │   ├── useAssetAccountAvailableSettings.ts
    │   └── useCreateAssetAccountAvailableSetting.ts
    ├── types/
    │   └── assetAccount.ts
    └── pages/
        └── AssetAccountDetailPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

### 28.72 フロントエンドで履歴を直接更新しない

ACC-006成功後に、React側で

```text
旧設定.endYearMonth更新
+
新設定追加
```

を正式な履歴状態として手動確定しない。

サーバー側でトランザクション・ロック・制約を経て確定した状態をACC-005から再取得する。

---

### 28.73 フロントエンドで前月を業務確定しない

画面表示のために、

```text
2026-08
    ↓
2026-07
```

を計算することはできる。

ただし、旧設定の正式な`endYearMonth`をフロントエンド側で確定しない。

正式状態はサーバー側の処理結果を正とする。

---

### 28.74 フロントエンドで行わないこと

ACC-006のReact・TypeScript実装では、以下をフロントエンドの責務としない。

- 利用者境界の最終保証
- 資産口座存在確認の最終保証
- 論理削除判定
- 履歴0件判定
- 履歴整合性の最終保証
- 期間重複判定
- 期間欠落判定
- 継続中設定の最終特定
- 行ロック
- 同時更新制御
- UNIQUE制約判定
- 旧設定のDB更新
- 新設定のDB登録
- トランザクション制御
- 履歴の自動修復
- 過去履歴の直接編集

フロントエンドは、

```text
現在状態・履歴取得
    ↓
利用者入力
    ↓
ACC-006実行
    ↓
結果表示
    ↓
ACC-003 / ACC-005再取得
```

に責務を限定する。

---

## 29. 設計上の補足

### 29.1 利用可能資産設定を履歴管理する理由

利用可能資産区分は、単なる現在値ではなく、対象年月によって変化する業務状態として扱う。

例えば、

```text
2026-01 ～ 2026-07
isAvailable = true

2026-08 ～ NULL
isAvailable = false
```

のように、どの年月でどの状態だったかを後から判定できる必要がある。

そのため、

```text
asset_accounts.is_available
```

のような固定属性として保持せず、

```text
asset_account_available_settings
```

で履歴管理する。

---

### 29.2 現在設定を直接上書きしない理由

例えば、現在設定が

```text
2026-01 ～ NULL
isAvailable = true
```

の場合に、

```text
is_available = false
```

へ直接UPDATEすると、

```text
2026-01以降
常にfalseだった
```

という意味に変わってしまう。

過去状態を保持するため、

```text
旧設定を終了
+
新設定を追加
```

する方式とする。

---

### 29.3 ACC-006を登録APIとする理由

ACC-006では、新しい利用可能資産設定を1件追加する。

概念的には、

```text
旧設定
    UPDATE

新設定
    INSERT
```

となるが、業務上の主目的は

```text
新しい設定を登録する
```

ことである。

そのため、

```http
POST /api/v1/asset-accounts/{assetAccountId}/available-settings
```

を使用する。

---

### 29.4 PATCHを採用しない理由

ACC-006は、既存の現在設定1件だけを部分更新する操作ではない。

実際には、

```text
現在設定を終了する
+
新設定を追加する
```

という履歴変更を行う。

そのため、

```http
PATCH /api/v1/asset-account-available-settings/{settingId}
```

のような既存履歴直接更新APIとはしない。

---

### 29.5 settingIdをURLへ含めない理由

ACC-006では、どの現在設定を終了するかをクライアントに指定させない。

以下のようなURLにはしない。

```http
POST /api/v1/asset-account-available-settings/{settingId}/next
```

現在設定は、サーバー側で

```text
asset_account_id
+
end_year_month IS NULL
```

から特定する。

これにより、クライアントが過去設定や誤った設定を直接変更することを防止する。

---

### 29.6 資産口座配下のURLとする理由

利用可能資産設定は、単独で存在するものではなく、必ず資産口座に帰属する。

そのため、

```text
asset-accounts
    ↓
available-settings
```

という親子関係をURLへ表現する。

```http
POST /api/v1/asset-accounts/{assetAccountId}/available-settings
```

とする。

---

### 29.7 ACC-005と同じURLを使用する理由

ACC-005とACC-006は、同じ利用可能資産設定リソースを扱う。

そのため、

```text
GET
    → 履歴取得

POST
    → 新設定登録
```

として、HTTPメソッドによって操作を表現する。

```text
ACC-005
GET
/api/v1/asset-accounts/{assetAccountId}/available-settings

ACC-006
POST
/api/v1/asset-accounts/{assetAccountId}/available-settings
```

とする。

---

### 29.8 ACC-004へisAvailableを含めない理由

ACC-004は、資産口座の通常属性更新APIである。

更新対象は、

```text
name
assetType
```

とする。

一方、

```text
isAvailable
```

は、履歴を伴う状態変更である。

そのため、

```text
通常属性
    → ACC-004

利用可能資産区分
    → ACC-006
```

と責務を分離する。

---

### 29.9 ACC-005との責務分離

ACC-005は、履歴を参照する。

ACC-006は、履歴を変更する。

概念的には、

```text
ACC-005
    → Query

ACC-006
    → Command
```

とする。

履歴取得処理の中で設定変更を行わない。

---

### 29.10 旧設定の終了年月をクライアント指定させない理由

ACC-006では、旧設定の

```text
endYearMonth
```

をRequestから受け付けない。

旧設定終了年月は、

```text
新設定.startYearMonthの前月
```

として一意に決定できるためである。

例えば、

```text
新設定開始
2026-08
```

なら、

```text
旧設定終了
2026-07
```

となる。

---

### 29.11 新設定のendYearMonthをNULLとする理由

ACC-006で登録する新しい設定は、その時点で最新の設定となる。

そのため、

```text
end_year_month = NULL
```

として登録する。

終了年月は、次回ACC-006でさらに新しい設定を追加するときに確定する。

---

### 29.12 9999-12などを使わない理由

継続中設定を表すために、

```text
9999-12
2099-12
```

などの特殊年月を使用しない。

終了年月未確定という状態を自然に表現するため、

```text
NULL
```

を使用する。

---

### 29.13 新設定開始年月を現在設定開始年月より後にする理由

現在設定が

```text
2026-08 ～ NULL
```

の場合に、新設定を

```text
2026-08 ～
```

から開始すると、旧設定の終了年月が

```text
2026-07
```

となり、

```text
startYearMonth
    >
endYearMonth
```

になる。

そのため、

```text
new.startYearMonth
    >
current.startYearMonth
```

を必須とする。

---

### 29.14 過去履歴への挿入を許可しない理由

例えば、

```text
2026-01 ～ 2026-06
true

2026-07 ～ NULL
false
```

という履歴に対して、

```text
2026-04
```

から新設定を入れる場合、既存履歴を複数件再構成する必要がある。

Phase1ではその複雑性を扱わず、

```text
最新履歴の後ろへ
新設定を追加する
```

操作に限定する。

---

### 29.15 同じisAvailableを登録できない理由

現在設定が

```text
isAvailable = true
```

の場合に、

```text
isAvailable = true
```

の新設定を登録しても、業務状態は変わらない。

例えば、

```text
2026-01 ～ 2026-07
true

2026-08 ～ NULL
true
```

のような意味のない履歴分割となる。

そのため、

```text
current.isAvailable
    !=
request.isAvailable
```

を必須とする。

---

### 29.16 NO_CHANGEを正常扱いしない理由

現在状態と同じ設定を送信した場合に、

```text
201 Created
```

として新しい履歴を作成しない。

また、何もせず

```text
200 OK
```

として成功扱いすることもしない。

ACC-006は新しい業務状態を履歴として登録するAPIであるため、変更がない場合は

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

として扱う。

---

### 29.17 履歴が0件の場合に自動作成しない理由

ACC-002では、資産口座登録時に初期設定を必ず作成する。

そのため、利用中資産口座で

```text
利用可能資産設定履歴
0件
```

は通常状態ではない。

ACC-006で初期設定を自動補完すると、既存データ不整合を隠蔽することになる。

そのため、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

として扱う。

---

### 29.18 既存履歴不整合時に追加しない理由

既に、

```text
期間重複
期間欠落
複数継続中設定
```

などの不整合が存在する場合に新しい履歴を追加すると、状態がさらに複雑になる。

そのため、ACC-006実行前に既存履歴を検証し、不整合があれば更新を行わない。

---

### 29.19 ACC-005と同じValidatorを使用する理由

ACC-005とACC-006で利用可能資産設定履歴の正常条件が異なると、APIごとに履歴の解釈が変わってしまう。

そのため、

```text
AvailableSettingHistoryValidator
```

を共通利用し、

```text
ACC-005
    → 参照前に検証

ACC-006
    → 更新前に検証
```

とする。

---

### 29.20 Validatorを共通化することで得られる効果

履歴整合性ルールを1か所へ集約することで、

- 初期開始年月
- 期間大小
- 期間重複
- 期間欠落
- 継続中設定数
- 最新設定

について、APIごとの判定差異を防止できる。

---

### 29.21 Validatorで履歴を修復しない理由

Validatorは、

```text
正常
または
不正
```

を判定する責務とする。

以下のような処理は行わない。

```text
期間欠落
    → 前設定を延長

複数継続設定
    → 古い設定を終了

最新設定終了済み
    → NULLへ戻す
```

検証と修復を同じ責務にしない。

---

### 29.22 トランザクションを必須とする理由

ACC-006は、

```text
旧設定UPDATE
+
新設定INSERT
```

という複数DB更新を伴う。

片方だけ成功すると、履歴が不整合になる。

そのため、両処理を

```php
DB::transaction()
```

内で実行する。

---

### 29.23 Repository単位でトランザクションを張らない理由

ACC-006では、

```text
close()
create()
```

の両方が1つの業務操作である。

各Repositoryメソッド単位で別トランザクションにすると、

```text
close()
    commit

create()
    rollback
```

という中途半端な状態が発生し得る。

そのため、UseCase全体をトランザクション境界とする。

---

### 29.24 lockForUpdateを使用する理由

同一資産口座に対してACC-006が同時実行されると、複数Requestが同じ現在設定を取得する可能性がある。

そのため、現在設定を

```php
lockForUpdate()
```

でロックし、同一履歴への更新を直列化する。

---

### 29.25 履歴全件をロックしない理由

更新対象となる既存レコードは、現在継続中設定1件である。

そのため、過去履歴全件をロックする必要はない。

必要最小限の

```text
現在設定
```

だけをロックし、ロック範囲を小さくする。

---

### 29.26 資産口座行を原則ロックしない理由

ACC-006では、

```text
asset_accounts
```

を更新しない。

利用者境界や利用開始年月確認のために参照するだけである。

そのため、Phase1では資産口座行自体を原則ロックしない。

---

### 29.27 ロック後に再確認する理由

ロック取得前に履歴を確認していても、別Requestが先に変更している可能性がある。

そのため、`lockForUpdate()`取得後の現在設定を基準として、

```text
startYearMonth
isAvailable
```

を再確認する。

古い画面状態や古いQuery結果だけを信頼しない。

---

### 29.28 DB制約も併用する理由

アプリケーション側のロックと業務チェックだけではなく、DB側でも最終整合性を保証する。

少なくとも、

```text
asset_account_id
+
start_year_month
```

を一意とする。

必要に応じて、

```text
asset_account_id
WHERE end_year_month IS NULL
```

の部分UNIQUEインデックスも使用する。

---

### 29.29 部分UNIQUEインデックスを検討する理由

業務ルールでは、同一資産口座に

```text
end_year_month = NULL
```

の設定は1件のみ許可する。

PostgreSQLの部分UNIQUEインデックスを使用すれば、このルールをDBレベルでも保証できる。

概念的には、

```sql
CREATE UNIQUE INDEX
    uq_asset_account_available_settings_current
ON
    asset_account_available_settings (
        asset_account_id
    )
WHERE
    end_year_month IS NULL;
```

とする。

---

### 29.30 ApplicationとDBの二重防御とする理由

概念的には、

```text
UseCase
    → 業務条件

lockForUpdate
    → 同時更新制御

UNIQUE制約
    → 同一開始年月防止

部分UNIQUE
    → 現在設定複数件防止
```

という複数層の整合性保証とする。

DB制約だけ、またはアプリケーションだけに責務を偏らせない。

---

### 29.31 Repositoryを使用する理由

ACC-006では、

```text
UPDATE
INSERT
```

を行う。

そのため、

```text
Query
    → 読み取り

Repository
    → 書き込み
```

という共通方針に従い、状態変更をRepositoryへ集約する。

---

### 29.32 QueryとRepositoryを分離する理由

例えば、

```text
AssetAccountAvailableSettingQuery
    → 履歴取得
    → 現在設定取得

AssetAccountAvailableSettingRepository
    → 現在設定終了
    → 新設定作成
```

とする。

検索条件と状態変更処理を同じクラスへ混在させない。

---

### 29.33 UseCaseを設ける理由

ACC-006には、

```text
資産口座確認
履歴取得
履歴検証
現在設定ロック
開始年月判定
状態差分判定
旧設定終了
新設定登録
```

という複数の処理が存在する。

これらのオーケストレーションをActionへ直接書かず、UseCaseへ集約する。

---

### 29.34 Actionを薄くする理由

Actionは、

```text
HTTP入力
    ↓
Input DTO
    ↓
UseCase
    ↓
Responder
```

の橋渡しに限定する。

業務ルールやDB更新をActionへ持たせない。

ADRパターンとしても、Actionの責務をHTTP境界へ限定する。

---

### 29.35 FormRequestを使用する理由

ACC-006には、

```text
startYearMonth
isAvailable
```

という明確なRequest Bodyがある。

そのため、

- 必須
- NULL禁止
- 型
- `YYYY-MM`
- strict boolean

などのHTTP入力検証をFormRequestへ集約する。

---

### 29.36 DB状態依存の判定をFormRequestへ持たせない理由

例えば、

```text
現在設定開始年月より後か
現在とisAvailableが異なるか
```

は、現在のDB状態に依存する。

これらをFormRequestで判定すると、HTTP入力検証と業務状態判定が混在する。

そのため、UseCase側で扱う。

---

### 29.37 Input DTOを使用する理由

UseCaseがLaravelのRequestへ直接依存しないように、

```text
FormRequest
    ↓
Input DTO
    ↓
UseCase
```

とする。

これにより、HTTP層とアプリケーション層の依存関係を整理する。

---

### 29.38 Mass Assignmentを避ける理由

ACC-006では、クライアント指定可能項目とサーバー管理項目が明確に分かれている。

クライアント指定：

```text
startYearMonth
isAvailable
```

サーバー管理：

```text
asset_account_id
end_year_month
```

そのため、

```php
Model::create(
    $request->all(),
);
```

のようなMass Assignmentは避ける。

---

### 29.39 replicateを使用しない理由

現在設定を

```php
$currentSetting->replicate();
```

して新設定を作る方式は、意図しない属性まで引き継ぐ可能性がある。

新設定は、

```text
asset_account_id
start_year_month
end_year_month
is_available
```

を明示的に指定して作成する。

---

### 29.40 YearMonth処理を文字列演算しない理由

ACC-006では、

```text
開始年月比較
前月計算
年跨ぎ
```

が必要になる。

例えば、

```text
2027-01
    ↓
2026-12
```

を扱う。

そのため、

```text
month - 1
```

のような単純文字列・数値処理に依存しない。

---

### 29.41 YearMonth Value Objectを検討する理由

利用可能資産設定だけでなく、Life Plannerでは月単位の概念を多く扱う。

そのため、将来的に

```text
YearMonth
```

をValue Objectとして導入すると、

- 比較
- 前月
- 翌月
- 表示形式
- 妥当性

を共通化できる。

---

### 29.42 Phase1でYearMonth Value Objectを必須としない理由

Value Objectの導入は設計上有効だが、利用箇所が少ない段階ではクラス数だけが増える可能性がある。

Phase1では、Laravel標準機能等で十分安全に扱える場合は必須としない。

必要性が高まった段階でリファクタリングしてよい。

---

### 29.43 成功レスポンスで新設定だけ返す理由

ACC-006では、新しく作成された設定を返却する。

旧設定も更新されるが、旧設定まで含めるとレスポンス責務が履歴全体へ広がる。

履歴全体が必要な場合は、

```text
ACC-005
```

を再取得する。

---

### 29.44 旧設定をレスポンスへ含めない理由

ACC-006成功時に、

```json
{
  "previousSetting": {},
  "newSetting": {}
}
```

のような複合レスポンスにはしない。

ACC-006は新規登録された利用可能資産設定リソースを返す。

履歴表示はACC-005へ委譲する。

---

### 29.45 201 Createdを返す理由

ACC-006では、新しい

```text
asset_account_available_settings
```

レコードを作成する。

そのため、

```text
201 Created
```

を返却する。

旧設定UPDATEを含むことだけを理由に`200 OK`とはしない。

---

### 29.46 Locationヘッダーを必須としない理由

`201 Created`では`Location`ヘッダーを利用できる。

ただし、Phase1では利用可能資産設定の個別取得APIを持たない。

そのため、新規設定を指す明確な個別URLが存在せず、`Location`を必須とはしない。

---

### 29.47 Result DTOを使用する理由

Eloquent ModelをそのままResponderへ渡すと、HTTP層がDB構造に依存しやすい。

そのため、

```text
UseCase
    ↓
Result DTO
    ↓
Resource
```

とし、API返却に必要な値だけを渡す。

---

### 29.48 API Resourceを使用する理由

DBでは、

```text
start_year_month
end_year_month
is_available
```

だが、APIでは、

```text
startYearMonth
endYearMonth
isAvailable
```

とする。

この変換責務をAPI Resourceへ集約する。

---

### 29.49 Responderを使用する理由

Responderは、

```text
Result DTO
    ↓
Resource
    ↓
201 Created
```

へ変換する。

UseCaseにHTTPステータスやJSONレスポンス生成を持たせない。

---

### 29.50 ACC-006成功後にACC-005を再取得する理由

ACC-006によって、

```text
旧設定.endYearMonth
+
新設定
```

の両方が変化する。

成功レスポンスでは新設定だけを返すため、履歴全体の最新状態は

```text
ACC-005
```

から再取得する。

---

### 29.51 ACC-006成功後にACC-003を再取得する理由

ACC-003は、現在年月における

```text
isAvailable
```

を返却する。

ACC-006によって現在設定が変わった可能性があるため、ACC-003も再取得対象とする。

---

### 29.52 将来開始設定でもACC-005を再取得する理由

新設定の適用開始が将来であっても、履歴自体は登録直後に変化している。

そのため、ACC-005は必ず最新化する。

---

### 29.53 将来開始設定では現在状態が変わらない場合がある

例えば、

```text
現在年月
2026-08

旧設定
2026-01 ～ 2026-09
true

新設定
2026-10 ～ NULL
false
```

では、現在年月の

```text
isAvailable
```

はまだ`true`である。

そのため、React側でACC-006 Request値をそのまま現在状態へ反映しない。

---

### 29.54 Optimistic Updateを必須としない理由

ACC-006には、

- 履歴整合性
- 現在設定確認
- 開始年月判定
- `NO_CHANGE`
- 行ロック
- UNIQUE制約
- トランザクション

が関係する。

サーバー側で拒否される可能性があるため、Phase1ではOptimistic Updateを必須としない。

---

### 29.55 Mutationの自動Retryを行わない理由

ACC-006はPOSTによる更新APIである。

通信切断時に、サーバー側で

```text
成功済み
```

なのか

```text
未実行
```

なのか判断できない可能性がある。

そのため、同一POSTを無条件に自動再送しない。

---

### 29.56 Idempotency-Keyを採用しない理由

Phase1では、専用の

```text
Idempotency-Key
```

を導入しない。

重複や競合については、

```text
現在状態再確認
lockForUpdate
UNIQUE制約
NO_CHANGE判定
```

で防止する。

必要性が高まった場合は、将来的に検討する。

---

### 29.57 通信結果不明時に再取得する理由

ACC-006送信後に通信結果が不明となった場合は、

```text
ACC-005
+
ACC-003
```

を再取得する。

これにより、同じPOSTを無条件再送せずに実際のサーバー状態を確認できる。

---

### 29.58 Query Cacheを手動再構築しない理由

ACC-006成功レスポンスだけでは、旧設定の更新後状態を完全には取得できない。

そのため、

```text
旧設定.endYearMonthを
Reactで計算

+
新設定を配列へ追加
```

する方式を基本としない。

サーバーからACC-005を再取得する。

---

### 29.59 利用者境界をサーバーで最終保証する理由

React側で現在利用者と資産口座の対応を保持していても、Requestは改変できる。

そのため、

```text
userId
+
assetAccountId
```

による利用者境界はLaravel側で必ず確認する。

---

### 29.60 userIdをRequest Bodyへ含めない理由

利用者指定経路を

```text
X-User-Id
```

へ統一する。

Request Bodyにも`userId`を持たせると、

```text
Header user
Body user
```

という2つの値が存在し、不整合時のルールが必要になる。

そのため、Bodyへ含めない。

---

### 29.61 assetAccountIdをRequest Bodyへ含めない理由

対象資産口座はURLで既に表現されている。

そのため、

```text
URL assetAccountId
+
Body assetAccountId
```

という重複指定を避ける。

---

### 29.62 isAvailableを資産口座テーブルへ複製しない理由

現在状態を高速取得する目的で、

```text
asset_accounts.is_available
```

にも値を複製すると、

```text
asset_accounts
+
asset_account_available_settings
```

で同じ意味のデータを二重管理することになる。

Phase1では履歴テーブルを正とし、現在状態も履歴から判定する。

---

### 29.63 過去の月末資産データを更新しない理由

利用可能資産設定を新しく登録しても、既存の

```text
month_end_asset_balances
month_end_holding_values
month_end_asset_snapshots
```

を変更しない。

過去記録そのものと、その年月における利用可能資産区分は別概念として扱う。

---

### 29.64 過去の目的達成判定履歴を更新しない理由

`assessment_histories`は、判定実行時点の結果を履歴として保存する。

ACC-006によって後から設定が変わっても、過去判定結果を自動で書き換えない。

---

### 29.65 利用可能資産設定変更をイベントとして扱わない理由

Phase1では、

```text
AvailableSettingChanged
```

のようなドメインイベントを発行して関連処理を非同期実行する設計は必須としない。

ACC-006の責務は、

```text
履歴整合性を保って
設定を変更する
```

ことに限定する。

---

### 29.66 監査履歴を別途作らない理由

利用可能資産設定自体が時系列履歴となっている。

そのため、Phase1では同じ変更内容を記録する別監査テーブルを追加しない。

運用上の追跡は、通常ログと設定履歴を利用する。

---

### 29.67 履歴削除APIを作らない理由

利用可能資産設定履歴は、過去状態を再現するための業務データである。

そのため、Phase1では

```text
DELETE available-setting
```

のような履歴削除APIを提供しない。

誤登録修正が業務要件として必要になった場合は、履歴訂正方式を別途設計する。

---

### 29.68 過去履歴編集APIを作らない理由

過去履歴を自由に編集すると、

```text
期間重複
期間欠落
初期年月不整合
```

を生みやすい。

Phase1では履歴末尾への追加だけに限定し、履歴全体の整合性を単純に保つ。

---

### 29.69 再有効化専用APIを作らない理由

利用可能資産区分はbooleanであり、

```text
true
false
```

の双方をACC-006で登録できる。

そのため、

```text
enable
disable
```

の専用APIへ分割しない。

---

### 29.70 ACC-006と資産口座無効化を分離する理由

```text
isAvailable = false
```

と

```text
資産口座無効化
```

は意味が異なる。

`isAvailable = false`は、

```text
資産口座は利用中
ただし利用可能資産として扱わない
```

という状態である。

一方、資産口座無効化は、

```text
資産口座自体を
通常の管理対象から外す
```

操作である。

そのため、別APIとして扱う。

---

### 29.71 Phase1では複数設定一括登録を行わない

ACC-006では、1回のRequestで1つの新設定だけを登録する。

以下のような一括登録は行わない。

```json
{
  "settings": [
    {
      "startYearMonth": "2026-08",
      "isAvailable": false
    },
    {
      "startYearMonth": "2027-01",
      "isAvailable": true
    }
  ]
}
```

履歴追加ルールと排他制御を単純に保つためである。

---

### 29.72 将来予約を許可する場合の注意

将来年月からの設定変更を許可する場合、登録直後には

```text
最新履歴
```

と

```text
現在年月に有効な設定
```

が一致しないことがある。

例えば、

```text
現在年月
2026-08

設定A
2026-01 ～ 2026-09
true

設定B
2026-10 ～ NULL
false
```

では、

```text
最新履歴
    → 設定B

現在有効
    → 設定A
```

となる。

---

### 29.73 最新設定と現在設定を同一視しない

将来予約を許可する場合は、

```text
endYearMonth = null
```

だからといって現在年月にその設定が有効とは限らない。

現在年月での有効設定は、

```text
start_year_month
    <= currentYearMonth

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= currentYearMonth
)
```

で判定する。

---

### 29.74 ACC-003は現在年月基準で判定する

ACC-003の

```text
isAvailable
```

は、最新履歴ではなく現在年月に有効な履歴から取得する。

これにより、将来開始設定をACC-006で登録しても、適用開始前に現在状態が切り替わらない。

---

### 29.75 ACC-005は履歴全体を返す

ACC-005では、将来開始設定も含めて履歴全体を返却する。

これにより、利用者は

```text
現在設定
+
将来予約済み設定
```

を履歴として確認できる。

---

### 29.76 将来予約が存在する場合の追加登録

Phase1では、既に将来予約設定が存在する状態でさらに履歴途中へ新しい設定を挿入することは扱わない。

例えば、

```text
現在
2026-08

設定A
2026-01 ～ 2026-09

設定B
2026-10 ～ NULL
```

が存在する状態で、

```text
2026-09から別設定
```

を追加するような操作は対象外とする。

---

### 29.77 将来予約機能を広げすぎない

Phase1では、以下のような高度な予約管理機能を追加しない。

- 複数将来設定予約
- 将来予約変更
- 将来予約取消
- 履歴途中挿入
- 予約優先順位
- 日単位適用
- 時刻単位適用

必要になった段階で別ユースケースとして設計する。

---

### 29.78 Phase1では設計を広げすぎない

ACC-006では、以下を対象外とする。

- 過去履歴編集
- 過去履歴削除
- 履歴途中挿入
- 複数設定一括登録
- 履歴自動修復
- 資産口座通常属性更新
- 資産口座無効化
- 月末資産データ更新
- 過去判定履歴再計算
- Idempotency-Key
- 楽観ロック
- ETag
- ドメインイベント
- 専用監査履歴

Phase1では、

```text
現在の履歴末尾を終了し
新しい利用可能資産設定を
安全に1件追加する
```

ことへ責務を限定する。

---

## 30. 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)
