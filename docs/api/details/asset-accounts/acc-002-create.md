# ACC-002 資産口座登録

## 1. 概要

操作対象となる利用者に帰属する資産口座を新規登録する。

資産口座登録時には、以下の基本情報を登録する。

* 資産口座名
* 資産種別
* 残高記録単位
* 利用開始年月

また、資産口座登録時点の利用可能資産区分もあわせて登録する。

そのため、ACC-002では`asset_accounts`だけでなく、初期の利用可能資産設定として

```text
asset_account_available_settings
```

にも1件登録する。

概念的には、以下の処理となる。

```text
資産口座登録
    ↓
asset_accounts
1件登録
    ↓
初期利用可能資産設定
1件登録
```

両方の登録は同一処理として扱い、途中で一方のみが登録された状態を残さない。

---

## 2. ユースケース

利用者は、新しく管理したい資産口座をLife Plannerへ登録する。

例えば、以下のような資産口座を登録する。

* 現金
* 銀行口座
* 証券口座
* iDeCo
* 企業型DC
* その他の資産口座

登録時には、その資産口座を

```text
口座単位
```

で残高管理するか、

```text
商品単位
```

で管理するかを指定する。

また、目的達成判定に利用する資産として扱うかどうかを、初期の利用可能資産区分として指定する。

登録後は、ACC-001 資産口座一覧取得やACC-003 資産口座詳細取得で登録内容を確認できる。

---

## 3. エンドポイント

```http
POST /api/v1/asset-accounts
```

---

## 4. HTTPメソッド

```text
POST
```

本APIは、新しい資産口座を作成するため、`POST`を使用する。

正常終了時には、

```text
asset_accounts
```

へ新規レコードを登録する。

また、資産口座登録時点の初期利用可能資産設定として、

```text
asset_account_available_settings
```

にも新規レコードを登録する。

そのため、本APIは業務データに対する副作用を持つ。

---

## 5. 利用者コンテキスト

本APIは利用者依存APIのため、`X-User-Id`を必須とする。

操作対象となる利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者コンテキストの特定、`X-User-Id`の検証および利用者境界については、[API共通方針](../api-common-policy.md)に従う。

資産口座登録時には、サーバー側で特定した操作対象利用者IDを

```text
asset_accounts.user_id
```

へ設定する。

クライアントから`userId`または`user_id`をリクエストボディとして受け付けない。

概念的には、以下とする。

```text
X-User-Id
    ↓
利用者コンテキスト
    ↓
asset_accounts.user_id
```

これにより、登録先利用者をクライアント側から任意に差し替えられないようにする。

---

### 5.1 利用者境界

ACC-002で登録する資産口座は、必ず操作対象利用者に帰属させる。

以下のような他利用者IDを指定して資産口座を登録することはできない。

```json
{
  "userId": "2"
}
```

`userId`はACC-002のリクエスト項目として定義しない。

---

### 5.2 資産口座名の一意性と利用者境界

資産口座名の重複判定は、システム全体ではなく、操作対象利用者の範囲で行う。

概念的には、

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = 登録する資産口座名
```

によって重複を確認する。

例えば、

```text
User A
    証券口座

User B
    証券口座
```

のように、異なる利用者が同じ資産口座名を持つことは許可する。

一方、同一利用者内で

```text
証券口座
証券口座
```

のように同名の資産口座を複数登録することは許可しない。

論理削除済み資産口座を重複判定へ含めるかどうかは、資産口座の再登録ルールに従う。

---

### 5.3 初期利用可能資産設定の利用者境界

ACC-002では、資産口座登録と同時に初期の

```text
asset_account_available_settings
```

を登録する。

この設定には直接`user_id`を持たせず、登録した

```text
asset_accounts.id
```

を通して操作対象利用者に帰属させる。

概念的には、

```text
操作対象利用者
    ↓
asset_accounts.user_id
    ↓
asset_accounts.id
    ↓
asset_account_available_settings.asset_account_id
```

となる。

他利用者の資産口座IDを利用可能資産設定の登録先として使用してはならない。

---

### 5.4 X-User-Idが不正な場合

以下の場合は、資産口座登録処理へ進まない。

* `X-User-Id`が指定されていない
* `X-User-Id`の形式が不正
* 指定された利用者が存在しない
* 指定された利用者が論理削除済み

この場合、

```text
asset_accounts
asset_account_available_settings
```

のどちらにもレコードを登録しない。

利用者コンテキストに関するエラーコードおよびHTTPステータスは、API共通方針に従う。

---

## 6. パスパラメータ

なし。

本APIでは、新しい資産口座を登録するため、既存の資産口座IDをパスパラメータとして指定しない。

---

## 7. クエリパラメータ

なし。

Phase1では、資産口座登録時の動作を変更するためのクエリパラメータは提供しない。

登録する資産口座の情報は、リクエストボディで指定する。

---

## 8. リクエストヘッダー

| ヘッダー名          |  必須 | 説明                 |
| -------------- | :-: | ------------------ |
| `X-User-Id`    |  ○  | 操作対象となる利用者ID       |
| `Content-Type` |  ○  | `application/json` |
| `Accept`       |  ○  | `application/json` |

`X-User-Id`の検証および利用者境界は、API共通方針に従う。

リクエスト例：

```http
POST /api/v1/asset-accounts
Content-Type: application/json
Accept: application/json
X-User-Id: 1
```

---

## 9. リクエストボディ

資産口座の基本情報および登録時点の利用可能資産区分を指定する。

例：

```json
{
  "name": "証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

---

### 9.1 リクエスト項目

| 項目                     | 型       |  必須 | NULL | 説明                 |
| ---------------------- | ------- | :-: | :--: | ------------------ |
| `name`                 | string  |  ○  |   ×  | 資産口座名              |
| `assetType`            | string  |  ○  |   ×  | 資産種別               |
| `balanceRecordingUnit` | string  |  ○  |   ×  | 残高記録単位             |
| `startYearMonth`       | string  |  ○  |   ×  | 利用開始年月。`YYYY-MM`形式 |
| `isAvailable`          | boolean |  ○  |   ×  | 登録時点における利用可能資産区分   |

---

### 9.2 name

資産口座名を指定する。

例：

```json
{
  "name": "証券口座"
}
```

同一利用者内では、同じ資産口座名を重複して登録できない。

資産口座名の長さは、`asset_accounts.name`のテーブル定義に従う。

---

### 9.3 assetType

資産種別を指定する。

指定可能な値は、以下とする。

| 値              | 意味    |
| -------------- | ----- |
| `CASH`         | 現金    |
| `BANK`         | 銀行    |
| `SECURITIES`   | 証券    |
| `IDECO`        | iDeCo |
| `CORPORATE_DC` | 企業型DC |
| `OTHER`        | その他   |

リクエストでは、データベースで保持する数値コードを直接指定せず、APIで定義した文字列を使用する。

---

### 9.4 balanceRecordingUnit

月末資産をどの単位で記録するかを指定する。

指定可能な値は、以下とする。

| 値         | 意味   |
| --------- | ---- |
| `ACCOUNT` | 口座単位 |
| `HOLDING` | 商品単位 |

`ACCOUNT`の場合は、資産口座単位で月末残高を記録する。

`HOLDING`の場合は、その資産口座に属する保有商品単位で月末評価額を記録する。

---

### 9.5 startYearMonth

資産口座の利用開始年月を指定する。

形式は、

```text
YYYY-MM
```

とする。

例：

```json
{
  "startYearMonth": "2026-08"
}
```

日付単位ではなく、年月単位で管理する。

---

### 9.6 isAvailable

登録時点において、目的達成判定へ利用する資産として扱うかどうかを指定する。

```text
true
    → 利用可能資産として扱う

false
    → 利用可能資産として扱わない
```

例：

```json
{
  "isAvailable": true
}
```

`isAvailable`は、`asset_accounts`へ直接保持しない。

ACC-002では、初期利用可能資産設定として

```text
asset_account_available_settings
```

へ登録する。

初期設定の

```text
start_year_month
```

には、資産口座の

```text
asset_accounts.start_year_month
```

と同じ年月を設定する。

初期設定であるため、

```text
end_year_month
```

は`NULL`とする。

概念的には、

```text
asset_accounts.start_year_month
    = 2026-08

asset_account_available_settings.start_year_month
    = 2026-08

asset_account_available_settings.end_year_month
    = NULL

asset_account_available_settings.is_available
    = true
```

となる。

---

### 9.7 リクエストで受け付けない項目

以下の項目は、クライアントから受け付けない。

* `id`
* `userId`
* `user_id`
* `isEnabled`
* `deletedAt`
* `deleted_at`
* `createdAt`
* `created_at`
* `updatedAt`
* `updated_at`
* `assetAccountId`
* `asset_account_id`
* `endYearMonth`
* `end_year_month`

これらは、サーバー側で設定するか、ACC-002では変更対象としない。

特に、

```text
user_id
```

は、`X-User-Id`から特定した利用者コンテキストを使用してサーバー側で設定する。

また、

```text
asset_account_id
```

は、新規登録した`asset_accounts.id`を使用して初期利用可能資産設定へ設定する。

---

## 10. バリデーション

リクエストヘッダーおよびリクエストボディについて、以下の検証を行う。

---

### 10.1 リクエストヘッダー

以下を検証する。

* `X-User-Id`が指定されていること
* `X-User-Id`がIDの共通形式に一致すること
* 指定された利用者が存在すること
* 指定された利用者が論理削除されていないこと
* `Content-Type`が`application/json`であること
* `Accept`が`application/json`であること

利用者コンテキストに関する検証は、API共通方針に従う。

---

### 10.2 name

以下を検証する。

* 必須であること
* 文字列であること
* 空文字ではないこと
* テーブル定義で定めた最大文字数以内であること
* 同一利用者内で同名の資産口座が存在しないこと

概念的な重複判定は、以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = name
```

異なる利用者に同名の資産口座が存在することは許可する。

---

### 10.3 assetType

以下を検証する。

* 必須であること
* 文字列であること
* 定義済みの列挙値であること

指定可能な値は、以下とする。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

以下のような未定義値は受け付けない。

```json
{
  "assetType": "CRYPTO"
}
```

---

### 10.4 balanceRecordingUnit

以下を検証する。

* 必須であること
* 文字列であること
* 定義済みの列挙値であること

指定可能な値は、以下とする。

```text
ACCOUNT
HOLDING
```

以下のような未定義値は受け付けない。

```json
{
  "balanceRecordingUnit": "UNKNOWN"
}
```

---

### 10.5 startYearMonth

以下を検証する。

* 必須であること
* 文字列であること
* `YYYY-MM`形式であること
* 実在する年月であること

正常例：

```text
2026-01
2026-08
2027-12
```

異常例：

```text
2026-1
2026/08
202608
2026-00
2026-13
abcd-ef
```

Phase1では、年月単位で管理するため、日付を含む値は受け付けない。

---

### 10.6 isAvailable

以下を検証する。

* 必須であること
* booleanであること
* `NULL`ではないこと

正常例：

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

以下のような文字列や数値による指定は受け付けない。

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

### 10.7 項目間バリデーション

ACC-002では、`balanceRecordingUnit`によって登録時に別の入力項目を追加要求しない。

例えば、

```text
balanceRecordingUnit = HOLDING
```

の場合でも、ACC-002で保有商品の同時登録は行わない。

資産口座登録後に、HLD-002 保有商品登録APIを使用して保有商品を登録する。

---

### 10.8 利用開始年月と初期利用可能資産設定

初期利用可能資産設定の適用開始年月は、クライアントから個別に指定させない。

必ず、

```text
asset_account_available_settings.start_year_month
    =
asset_accounts.start_year_month
```

とする。

これにより、資産口座の利用開始前から利用可能資産設定だけが存在する状態を防止する。

---

### 10.9 バリデーションエラー

リクエストボディの項目バリデーションに失敗した場合は、API共通方針で定めたバリデーションエラー形式を使用する。

バリデーションエラーとなった場合は、

```text
asset_accounts
asset_account_available_settings
```

のどちらにもレコードを登録しない。

---

## 11. 業務ルール

ACC-002では、操作対象利用者に帰属する資産口座と、その資産口座に対する初期利用可能資産設定を登録する。

---

### 11.1 操作対象利用者に帰属する資産口座として登録する

新規資産口座の

```text
asset_accounts.user_id
```

には、`X-User-Id`から特定した操作対象利用者IDを設定する。

クライアントから`user_id`を指定させない。

概念的には、

```text
X-User-Id
    ↓
利用者コンテキスト
    ↓
asset_accounts.user_id
```

とする。

---

### 11.2 同一利用者内で資産口座名を重複させない

同一利用者内では、同じ資産口座名を複数登録できない。

概念的には、

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = 登録する資産口座名
```

に該当する資産口座が存在する場合、登録不可とする。

異なる利用者間では、同じ資産口座名を登録できる。

例えば、

```text
User A
    └─ 証券口座

User B
    └─ 証券口座
```

は許可する。

---

### 11.3 論理削除済み資産口座との重複

資産口座一覧では、論理削除済み資産口座も履歴として保持する。

そのため、同一利用者に論理削除済みの同名資産口座が存在する場合も、同名の新規登録を許可しない。

これにより、同一利用者内で資産口座名による識別が曖昧になることを防止する。

重複判定では、Laravel SoftDeletesの通常スコープだけに依存せず、論理削除済みレコードも含めて確認する。

概念的には、

```text
同一user_id
+
同一name
+
利用中・論理削除済みの両方
    ↓
重複あり
```

とする。

---

### 11.4 資産種別

`assetType`には、APIで定義した資産種別のみ指定できる。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

APIで受け取った文字列は、Laravel側でデータベース保存形式へ変換する。

データベース内部の数値コードをクライアントへ意識させない。

---

### 11.5 残高記録単位

`balanceRecordingUnit`には、以下のいずれかを指定する。

```text
ACCOUNT
HOLDING
```

`ACCOUNT`の場合は、資産口座単位で月末残高を管理する。

`HOLDING`の場合は、保有商品単位で月末評価額を管理する。

ACC-002では、残高記録単位を登録するだけとし、月末残高または保有商品を同時登録しない。

---

### 11.6 商品単位の場合でも保有商品を自動登録しない

```text
balanceRecordingUnit = HOLDING
```

で資産口座を登録した場合でも、ACC-002では保有商品を登録しない。

資産口座登録後に、HLD-002 保有商品登録APIを使用して必要な保有商品を登録する。

概念的には、

```text
ACC-002
資産口座登録
    ↓
balanceRecordingUnit = HOLDING
    ↓
必要に応じて
HLD-002
保有商品登録
```

とする。

---

### 11.7 利用開始年月

`startYearMonth`は、資産口座をLife Planner上で管理し始める年月を表す。

年月単位で管理し、

```text
YYYY-MM
```

形式とする。

資産口座の利用開始年月より前の月末資産管理において、当該資産口座を記録対象として扱わない。

具体的な対象年月判定は、月末資産API側の業務ルールに従う。

---

### 11.8 登録時は利用中として扱う

ACC-002で登録した資産口座は、新規登録時点では利用中として扱う。

そのため、

```text
asset_accounts.deleted_at
```

は`NULL`とする。

クライアントから`isEnabled`または`deletedAt`を指定させない。

概念的には、

```text
新規登録
    ↓
deleted_at = NULL
    ↓
isEnabled = true
```

となる。

---

### 11.9 初期利用可能資産設定を必ず登録する

ACC-002では、資産口座だけを登録して利用可能資産設定が存在しない状態を作らない。

資産口座登録と同時に、

```text
asset_account_available_settings
```

へ初期設定を1件登録する。

これにより、ACC-001などで対象年月における利用可能資産区分を必ず判定できる状態とする。

---

### 11.10 初期利用可能資産設定の開始年月

初期利用可能資産設定の

```text
asset_account_available_settings.start_year_month
```

には、

```text
asset_accounts.start_year_month
```

と同じ年月を設定する。

概念例：

```text
asset_accounts
start_year_month = 2026-08

        ↓

asset_account_available_settings
start_year_month = 2026-08
```

クライアントから利用可能資産設定の開始年月を別途指定させない。

---

### 11.11 初期利用可能資産設定の終了年月

初期利用可能資産設定の

```text
asset_account_available_settings.end_year_month
```

は、

```text
NULL
```

とする。

登録時点では、利用可能資産区分の終了年月は決まっていないためである。

将来、利用可能資産区分を変更する場合は、ACC-006 利用可能資産設定登録APIで履歴を追加する。

---

### 11.12 isAvailable

リクエストの

```text
isAvailable
```

は、初期利用可能資産設定の

```text
asset_account_available_settings.is_available
```

へ保存する。

概念的には、

```text
Request
isAvailable = true

        ↓

asset_account_available_settings
is_available = true
```

とする。

`asset_accounts`へ`is_available`を保持しない。

---

### 11.13 利用可能資産区分を履歴として管理する

利用可能資産区分は、資産口座の固定属性としてではなく、対象年月によって変化する履歴情報として扱う。

そのため、

```text
asset_accounts
```

ではなく、

```text
asset_account_available_settings
```

で管理する。

ACC-002では、その履歴の最初の1件を資産口座と同時に登録する。

---

### 11.14 初期利用可能資産設定の関連先

初期利用可能資産設定の

```text
asset_account_id
```

には、ACC-002で新規登録した

```text
asset_accounts.id
```

を設定する。

クライアントから`assetAccountId`を指定させない。

概念的には、

```text
asset_accounts
INSERT
    ↓
生成されたid
    ↓
asset_account_available_settings.asset_account_id
```

とする。

---

### 11.15 資産口座と初期設定を一体として扱う

ACC-002において、

```text
asset_accounts
```

の登録だけ成功し、

```text
asset_account_available_settings
```

の登録だけ失敗した状態は許容しない。

逆に、資産口座が存在しない状態で初期利用可能資産設定だけが登録される状態も許容しない。

両方の登録を1つの業務処理として扱う。

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
リクエストボディ検証
    ↓
同一利用者内の資産口座名重複確認
    ↓
トランザクション開始
    ↓
asset_accounts登録
    ↓
生成されたasset_accounts.idを取得
    ↓
asset_account_available_settings登録
    ↓
トランザクションコミット
    ↓
APIレスポンス用の形式へ変換
    ↓
正常レスポンス返却
```

---

### 12.1 利用者コンテキスト確認

共通Middlewareで`X-User-Id`を検証し、操作対象利用者を特定する。

利用者コンテキストが不正な場合は、リクエストボディの業務処理へ進まない。

---

### 12.2 リクエストボディ検証

以下の入力値を検証する。

```text
name
assetType
balanceRecordingUnit
startYearMonth
isAvailable
```

バリデーションに失敗した場合は、データベース登録処理へ進まない。

---

### 12.3 資産口座名重複確認

操作対象利用者について、同名の資産口座が存在しないことを確認する。

論理削除済み資産口座も重複確認対象とする。

概念的には、

```text
userId
+
name
    ↓
利用中・論理削除済みを含めて検索
    ↓
存在する
    → 登録不可

存在しない
    → 登録処理へ進む
```

とする。

---

### 12.4 資産口座登録

`asset_accounts`へ資産口座を登録する。

概念的な登録内容は、以下とする。

```text
user_id
    = 操作対象利用者ID

name
    = request.name

asset_type
    = request.assetTypeをDB保存形式へ変換

balance_recording_unit
    = request.balanceRecordingUnitをDB保存形式へ変換

start_year_month
    = request.startYearMonth

deleted_at
    = NULL
```

`id`、`created_at`、`updated_at`は、Laravelおよびデータベースの通常の登録処理に従う。

---

### 12.5 初期利用可能資産設定登録

資産口座登録後、生成された資産口座IDを使用して

```text
asset_account_available_settings
```

へ初期設定を登録する。

概念的な登録内容は、以下とする。

```text
asset_account_id
    = 新規登録したasset_accounts.id

start_year_month
    = request.startYearMonth

end_year_month
    = NULL

is_available
    = request.isAvailable
```

---

### 12.6 登録結果生成

資産口座と初期利用可能資産設定の登録完了後、APIレスポンスに必要な登録結果を生成する。

データベースカラムをそのままレスポンスへ返却しない。

API用のDTOおよびAPI Resourceを通してレスポンス形式へ変換する。

---

## 13. トランザクション境界

ACC-002では、明示的なデータベーストランザクションを使用する。

理由は、1回のAPI実行で

```text
asset_accounts
```

と

```text
asset_account_available_settings
```

の2テーブルへ関連するデータを登録するためである。

---

### 13.1 トランザクション対象

以下の処理を同一トランザクション内で実行する。

```text
BEGIN
    ↓
asset_accounts INSERT
    ↓
asset_account_available_settings INSERT
    ↓
COMMIT
```

どちらかの登録に失敗した場合は、

```text
ROLLBACK
```

する。

---

### 13.2 資産口座登録失敗

`asset_accounts`の登録に失敗した場合は、初期利用可能資産設定を登録しない。

概念的には、

```text
asset_accounts
INSERT失敗
    ↓
ROLLBACK
    ↓
asset_account_available_settings
INSERTしない
```

となる。

---

### 13.3 初期利用可能資産設定登録失敗

`asset_accounts`の登録成功後に、

```text
asset_account_available_settings
```

の登録に失敗した場合は、先に登録した`asset_accounts`もロールバックする。

概念的には、

```text
asset_accounts
INSERT成功
    ↓
asset_account_available_settings
INSERT失敗
    ↓
ROLLBACK
    ↓
asset_accountsのINSERTも取消
```

とする。

---

### 13.4 トランザクション外で行う処理

以下は、原則としてトランザクション開始前に行う。

* `X-User-Id`の検証
* 操作対象利用者の特定
* リクエスト形式のバリデーション

不要な時間、トランザクションを保持しないためである。

資産口座名の重複については、事前確認だけに依存せず、データベースの一意性制約と組み合わせて整合性を保証する。

---

### 13.5 トランザクションの実装単位

Laravelでは、UseCaseからトランザクション境界を制御する。

概念的には、

```php
DB::transaction(function () {
    // 資産口座登録
    // 初期利用可能資産設定登録
});
```

とする。

Repository単位で個別にトランザクションを開始せず、

```text
資産口座登録
+
初期利用可能資産設定登録
```

というユースケース全体を1つのトランザクション境界とする。


---

## 14. 排他制御

ACC-002では、同一利用者に対して同名の資産口座が同時登録される可能性を考慮する。

アプリケーション側では登録前に重複確認を行うが、重複確認だけでは同時実行時の競合を完全には防止できない。

そのため、同一利用者内の資産口座名についてデータベースの一意性制約を最終防衛線として使用する。

概念的には、以下の組み合わせに一意性を保証する。

```text
user_id
+
name
```

論理削除済み資産口座も同名再登録不可とする設計のため、一意性制約は論理削除状態にかかわらず同一利用者・同一名称を重複させない構成とする。

---

### 14.1 登録前の重複確認

ACC-002では、登録処理前に

```text
user_id
+
name
```

で既存資産口座を確認する。

概念的には、

```text
同一利用者
+
同一資産口座名
    ↓
存在する
    → 登録不可

存在しない
    → 登録処理へ進む
```

とする。

ただし、この確認だけを一意性保証としてはならない。

---

### 14.2 同時実行

例えば、同一利用者について同じ資産口座名を2つのリクエストが同時に登録する場合を考える。

```text
Request A
重複なし確認

Request B
重複なし確認

Request A
INSERT

Request B
INSERT
```

この場合、アプリケーション側の事前確認だけでは両方が登録処理へ進む可能性がある。

そのため、データベースのUNIQUE制約によって2件目の登録を防止する。

---

### 14.3 行ロック

ACC-002では、新規レコードを登録する処理であり、登録前には対象となる`asset_accounts`行が存在しない。

そのため、

```php
lockForUpdate()
```

による既存行ロックは、同名新規登録の競合防止には適していない。

Phase1では、同名資産口座の重複防止を

```text
事前重複確認
+
UNIQUE制約
```

によって行う。

---

### 14.4 利用者行をロックしない

同一利用者の資産口座登録を直列化するために、

```text
users
```

の行を`lockForUpdate()`する方式は採用しない。

資産口座登録のたびに利用者行をロックすると、同一利用者に関する他の処理まで不要にブロックする可能性があるためである。

---

### 14.5 初期利用可能資産設定

ACC-002では、新規作成した資産口座に対して初期利用可能資産設定を1件登録する。

資産口座IDはACC-002内で新規生成されたものであるため、通常は同じ`asset_account_id`へ別リクエストが同時に初期設定を登録することはない。

初期設定の整合性は、資産口座登録と同一トランザクションで登録することによって保証する。

---

### 14.6 UNIQUE制約違反

同時実行によって`asset_accounts`の一意性制約違反が発生した場合は、PostgreSQLの例外をそのままクライアントへ返却しない。

業務上の

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換し、適切なHTTPステータスで返却する。

制約名やSQLはレスポンスへ公開しない。

---

## 15. 成功レスポンス

資産口座および初期利用可能資産設定の登録に成功した場合は、

```text
201 Created
```

を返却する。

正常時は、API共通方針に従った成功レスポンス形式を使用する。

概念例：

```json
{
  "data": {
    "id": "10",
    "name": "証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

---

### 15.1 レスポンス項目

| 項目                          | 型       | NULL | 説明                  |
| --------------------------- | ------- | :--: | ------------------- |
| `data.id`                   | string  |   ×  | 登録した資産口座ID          |
| `data.name`                 | string  |   ×  | 資産口座名               |
| `data.assetType`            | string  |   ×  | 資産種別                |
| `data.balanceRecordingUnit` | string  |   ×  | 残高記録単位              |
| `data.isAvailable`          | boolean |   ×  | 登録した初期利用可能資産区分      |
| `data.startYearMonth`       | string  |   ×  | 利用開始年月。`YYYY-MM`形式  |
| `data.isEnabled`            | boolean |   ×  | 利用状態。新規登録時は常に`true` |

---

### 15.2 id

新規登録した資産口座IDを返却する。

API共通方針に従い、データベースでは`bigint`であってもAPIレスポンスでは文字列として返却する。

例：

```json
{
  "id": "10"
}
```

---

### 15.3 name

登録した資産口座名を返却する。

例：

```json
{
  "name": "証券口座"
}
```

---

### 15.4 assetType

登録した資産種別を、API用の文字列で返却する。

例：

```json
{
  "assetType": "SECURITIES"
}
```

データベース内部の数値コードをそのまま返却しない。

---

### 15.5 balanceRecordingUnit

登録した残高記録単位を、API用の文字列で返却する。

例：

```json
{
  "balanceRecordingUnit": "HOLDING"
}
```

想定する値は、以下とする。

```text
ACCOUNT
HOLDING
```

---

### 15.6 isAvailable

ACC-002で登録した初期利用可能資産設定の`is_available`を返却する。

例：

```json
{
  "isAvailable": true
}
```

`asset_accounts`の直接カラムとして保持している値ではない。

---

### 15.7 startYearMonth

資産口座の利用開始年月を返却する。

形式は、

```text
YYYY-MM
```

とする。

例：

```json
{
  "startYearMonth": "2026-08"
}
```

---

### 15.8 isEnabled

新規登録直後の資産口座は利用中であるため、

```json
{
  "isEnabled": true
}
```

を返却する。

データベース上では、

```text
deleted_at = NULL
```

に対応する。

`deleted_at`自体はレスポンスへ返却しない。

---

### 15.9 返却しない情報

ACC-002では、以下の内部情報をレスポンスへ返却しない。

* `user_id`
* `deleted_at`
* `created_at`
* `updated_at`
* `asset_account_available_settings.id`
* `asset_account_available_settings.asset_account_id`
* `asset_account_available_settings.start_year_month`
* `asset_account_available_settings.end_year_month`
* データベース内部の数値コード

利用可能資産設定については、登録結果として必要な

```text
isAvailable
```

のみ返却する。

---

## 16. エラーレスポンス

異常時は、API共通方針で定めた共通エラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "name",
        "reason": "required",
        "message": "資産口座名を指定してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 16.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

この場合、資産口座登録処理へ進まない。

---

### 16.2 INVALID_USER_ID

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

例えば、以下のような値を不正とする。

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

この場合、

```text
asset_accounts
asset_account_available_settings
```

のどちらにも登録しない。

---

### 16.4 VALIDATION_ERROR

リクエストボディの項目バリデーションに失敗した場合は、

```text
VALIDATION_ERROR
```

を返却する。

対象となる主な項目は、以下とする。

* `name`
* `assetType`
* `balanceRecordingUnit`
* `startYearMonth`
* `isAvailable`

複数の入力エラーがある場合は、`details`へ複数件返却してよい。

---

### 16.5 name未指定

`name`が指定されていない場合は、`VALIDATION_ERROR`として扱う。

概念例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "name",
        "reason": "required",
        "message": "資産口座名を指定してください。"
      }
    ]
  }
}
```

---

### 16.6 assetType不正

`assetType`が定義済みの値ではない場合は、`VALIDATION_ERROR`として扱う。

例えば、

```json
{
  "assetType": "CRYPTO"
}
```

のような未定義値は受け付けない。

---

### 16.7 balanceRecordingUnit不正

`balanceRecordingUnit`が

```text
ACCOUNT
HOLDING
```

以外の場合は、`VALIDATION_ERROR`として扱う。

---

### 16.8 startYearMonth不正

`startYearMonth`が

```text
YYYY-MM
```

形式ではない、または実在しない年月の場合は、`VALIDATION_ERROR`として扱う。

例えば、

```text
2026-13
2026/08
202608
```

などを不正とする。

---

### 16.9 isAvailable不正

`isAvailable`がbooleanではない場合は、`VALIDATION_ERROR`として扱う。

例えば、

```json
{
  "isAvailable": "true"
}
```

や、

```json
{
  "isAvailable": 1
}
```

は不正とする。

---

### 16.10 ASSET_ACCOUNT_NAME_ALREADY_EXISTS

操作対象利用者について、同じ資産口座名が既に存在する場合は、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

を返却する。

論理削除済み資産口座も重複判定対象とする。

概念的には、

```text
User A
    証券口座
```

が存在する状態で、User Aとして再度

```json
{
  "name": "証券口座"
}
```

を登録した場合にエラーとする。

---

### 16.11 他利用者の同名資産口座

他利用者にのみ同名資産口座が存在する場合は、重複エラーとしない。

例えば、

```text
User A
    証券口座

User B
    資産口座なし
```

の状態で、User Bとして

```json
{
  "name": "証券口座"
}
```

を登録することは許可する。

---

### 16.12 同時登録による重複

事前重複確認を通過した後に、並行リクエストによって同一利用者・同一名称の資産口座が先に登録された場合は、データベースのUNIQUE制約によって重複を防止する。

この場合も、クライアントへは

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

として返却する。

PostgreSQLの一意制約違反をそのまま公開しない。

---

### 16.13 初期利用可能資産設定登録失敗

`asset_accounts`の登録後に、初期

```text
asset_account_available_settings
```

の登録に失敗した場合は、トランザクションをロールバックする。

クライアントへ初期設定だけの専用内部エラーを公開することは基本としない。

想定外の登録失敗である場合は、

```text
INTERNAL_SERVER_ERROR
```

として扱う。

---

### 16.14 INTERNAL_SERVER_ERROR

資産口座登録処理で想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

トランザクション中の場合は、すべてロールバックする。

レスポンスへ、以下の内部情報を含めない。

* SQL
* PostgreSQL内部エラー
* 制約名
* テーブル名
* カラム名
* Laravel内部例外メッセージ
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 16.15 エラー時のデータ更新

いずれのエラーの場合も、ACC-002の登録処理が正常完了していない限り、

```text
asset_accountsだけ登録済み
```

または、

```text
asset_account_available_settingsだけ登録済み
```

という状態を残してはならない。

トランザクションによって、

```text
両方成功
または
両方未登録
```

を保証する。

---

## 17. HTTPステータス

ACC-002では、処理結果に応じて以下のHTTPステータスを返却する。

| HTTPステータス                   | 用途                |
| --------------------------- | ----------------- |
| `201 Created`               | 資産口座登録成功          |
| `400 Bad Request`           | 利用者コンテキスト不正、入力値不正 |
| `404 Not Found`             | 操作対象利用者不存在        |
| `409 Conflict`              | 同一利用者内の資産口座名重複    |
| `500 Internal Server Error` | 想定外のサーバー内部エラー     |

具体的なエラーコードは、API共通方針およびACC-002で定義したエラーレスポンスに従う。

---

### 17.1 201 Created

資産口座および初期利用可能資産設定の登録が正常に完了した場合は、

```http
201 Created
```

を返却する。

以下の両方が正常に登録された時点で成功とする。

```text
asset_accounts
    +
asset_account_available_settings
```

どちらか一方だけが登録された状態では、`201 Created`を返却しない。

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
VALIDATION_ERROR
```

例えば、以下を対象とする。

* `X-User-Id`未指定
* `X-User-Id`形式不正
* `name`未指定
* `assetType`不正
* `balanceRecordingUnit`不正
* `startYearMonth`不正
* `isAvailable`不正

---

### 17.3 404 Not Found

指定された利用者が存在しない場合、または論理削除済みの場合は、

```http
404 Not Found
```

を返却する。

エラーコードは、

```text
USER_NOT_FOUND
```

とする。

---

### 17.4 409 Conflict

操作対象利用者について、同名の資産口座が既に存在する場合は、

```http
409 Conflict
```

を返却する。

エラーコードは、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

とする。

アプリケーション側の事前重複確認によって検出した場合だけでなく、同時実行によってデータベースのUNIQUE制約違反が発生した場合も、同じHTTPステータスおよびエラーコードへ変換する。

---

### 17.5 500 Internal Server Error

資産口座登録処理中に想定外の例外が発生した場合は、

```http
500 Internal Server Error
```

を返却する。

エラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

トランザクション開始後に発生した場合は、登録処理をロールバックする。

---

## 18. 冪等性

ACC-002は、新しい資産口座を登録するAPIであるため、冪等ではない。

同一リクエストを複数回送信した場合、毎回同じ結果になることを保証しない。

---

### 18.1 同一リクエストの再送

例えば、以下のリクエストを送信する。

```json
{
  "name": "証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

1回目の登録に成功した場合は、

```http
201 Created
```

となる。

同じ利用者が同じ`name`で再度登録した場合は、同一利用者内の資産口座名重複となるため、

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となる。

---

### 18.2 重複防止と冪等性は区別する

同一利用者内の資産口座名重複を防止することは、ACC-002自体を冪等にすることを意味しない。

ACC-002は、

```http
POST /api/v1/asset-accounts
```

によって新しいリソースを作成するAPIであり、同一リクエストの再送に対して最初の成功レスポンスを再現することは保証しない。

---

### 18.3 Idempotency-Key

Phase1では、ACC-002専用の

```text
Idempotency-Key
```

は導入しない。

同一利用者・同一資産口座名の重複登録については、

```text
アプリケーション側の事前重複確認
    +
データベースのUNIQUE制約
```

によって防止する。

将来的に、通信再送などに対して同一登録結果を保証する必要が生じた場合は、Idempotency-Keyの導入を検討する。

---

## 19. キャッシュ

Phase1では、ACC-002専用のサーバー側アプリケーションキャッシュを使用しない。

資産口座および初期利用可能資産設定は、PostgreSQLへ直接登録する。

---

### 19.1 登録処理のキャッシュ

ACC-002は登録APIであるため、リクエスト結果をHTTPキャッシュの対象としない。

```http
POST /api/v1/asset-accounts
```

のレスポンスを利用して、後続の同一リクエストをキャッシュから処理する方式は採用しない。

---

### 19.2 資産口座一覧への影響

ACC-002成功後は、ACC-001 資産口座一覧取得の取得結果が変化する。

概念的には、

```text
ACC-002
資産口座登録成功
    ↓
asset_accounts追加
    ↓
ACC-001の取得結果が変化
```

となる。

フロントエンドでTanStack QueryなどのQuery Cacheを使用する場合は、ACC-002成功後に資産口座一覧のQuery Cacheを無効化する。

具体的な処理は、「React・TypeScriptでの利用」で定義する。

---

### 19.3 資産口座詳細への影響

ACC-002成功後は、新しく生成された

```text
data.id
```

を使用して、ACC-003 資産口座詳細取得を実行できる。

Phase1では、ACC-002成功レスポンスをサーバー側キャッシュへ保存してACC-003へ流用する方式は採用しない。

必要な場合は、ACC-003から最新状態を取得する。

---

### 19.4 利用可能資産設定への影響

ACC-002では、資産口座と同時に

```text
asset_account_available_settings
```

へ初期利用可能資産設定を登録する。

そのため、対象年月における利用可能資産区分を参照する処理についても、ACC-002成功後は最新のデータベース状態を参照する必要がある。

Phase1では、利用可能資産設定についてもサーバー側アプリケーションキャッシュを使用しない。

---

### 19.5 Cache-Control

ACC-002の成功レスポンスは、登録操作の結果を含むため、共有キャッシュによる再利用を前提としない。

HTTPキャッシュに関する具体的なヘッダー設定は、API共通方針に従う。

ACC-002独自のキャッシュ制御ルールは設けない。


---

## 20. 関連テーブル

ACC-002では、資産口座および初期利用可能資産設定を登録するため、以下のテーブルを使用する。

| テーブル                               | 用途               |  更新 |
| ---------------------------------- | ---------------- | :-: |
| `users`                            | 操作対象利用者の確認       |  ×  |
| `asset_accounts`                   | 資産口座の新規登録、同名重複確認 |  ○  |
| `asset_account_available_settings` | 初期利用可能資産設定の新規登録  |  ○  |

ACC-002では、

```text
asset_accounts
asset_account_available_settings
```

の2テーブルを更新する。

`users`は、利用者コンテキスト確認のために参照するのみとする。

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

利用者コンテキストの検証は、API共通Middlewareで行う。

ACC-002では、`users`を更新しない。

---

### 20.2 asset_accounts

資産口座の基本情報を新規登録するために使用する。

また、同一利用者内の資産口座名重複確認にも使用する。

主に以下のカラムを使用する。

| カラム                      | 用途                         |
| ------------------------ | -------------------------- |
| `id`                     | 資産口座ID。初期利用可能資産設定の関連先として使用 |
| `user_id`                | 操作対象利用者との関連                |
| `name`                   | 資産口座名、同名重複確認               |
| `asset_type`             | 資産種別                       |
| `balance_recording_unit` | 残高記録単位                     |
| `start_year_month`       | 利用開始年月                     |
| `created_at`             | 登録日時                       |
| `updated_at`             | 更新日時                       |
| `deleted_at`             | 利用状態。新規登録時は`NULL`          |

---

### 20.3 asset_accountsへの登録

リクエスト内容と利用者コンテキストから、概念的に以下を登録する。

```text
user_id
    = 操作対象利用者ID

name
    = request.name

asset_type
    = request.assetTypeを
      DB保存形式へ変換した値

balance_recording_unit
    = request.balanceRecordingUnitを
      DB保存形式へ変換した値

start_year_month
    = request.startYearMonth

deleted_at
    = NULL
```

`id`、`created_at`、`updated_at`は、Laravelおよびデータベースの通常の登録処理に従う。

---

### 20.4 資産口座名重複確認

同一利用者内では、資産口座名を重複させない。

概念的な検索条件は、以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = request.name
```

論理削除済み資産口座も重複確認対象に含める。

Laravel SoftDeletesを使用する場合は、重複確認時に`withTrashed()`を利用する。

---

### 20.5 asset_accountsの一意性

同時登録による重複を防止するため、概念的には

```text
user_id
+
name
```

の組み合わせに一意性を保証する。

これにより、アプリケーション側の重複確認を同時に通過した場合でも、同一利用者に同名資産口座が複数作成されることを防止する。

---

### 20.6 asset_account_available_settings

資産口座登録時の初期利用可能資産設定を新規登録するために使用する。

主に以下のカラムを使用する。

| カラム                | 用途                     |
| ------------------ | ---------------------- |
| `id`               | 利用可能資産設定ID             |
| `asset_account_id` | 登録した資産口座との関連           |
| `start_year_month` | 初期設定の適用開始年月            |
| `end_year_month`   | 初期設定の適用終了年月。登録時は`NULL` |
| `is_available`     | 利用可能資産区分               |
| `created_at`       | 登録日時                   |
| `updated_at`       | 更新日時                   |

---

### 20.7 初期利用可能資産設定への登録

`asset_accounts`の登録後、生成された資産口座IDを使用して概念的に以下を登録する。

```text
asset_account_id
    = 新規登録したasset_accounts.id

start_year_month
    = request.startYearMonth

end_year_month
    = NULL

is_available
    = request.isAvailable
```

これにより、資産口座の利用開始年月から有効な初期利用可能資産設定が必ず1件存在する状態とする。

---

### 20.8 start_year_monthの整合性

初期利用可能資産設定の

```text
start_year_month
```

は、資産口座の

```text
asset_accounts.start_year_month
```

と一致させる。

以下のような状態をACC-002から作らない。

```text
asset_accounts.start_year_month
    = 2026-08

asset_account_available_settings.start_year_month
    = 2026-09
```

または、

```text
asset_accounts.start_year_month
    = 2026-08

asset_account_available_settings.start_year_month
    = 2026-07
```

初期設定の開始年月は、クライアントから別項目として受け付けない。

---

### 20.9 end_year_month

初期利用可能資産設定の

```text
end_year_month
```

は、新規登録時は

```text
NULL
```

とする。

利用可能資産区分の変更時に、ACC-006の業務ルールに従って現在の設定期間を終了させ、新しい設定履歴を追加する。

---

### 20.10 is_available

リクエストの

```text
isAvailable
```

を、

```text
asset_account_available_settings.is_available
```

へ保存する。

`asset_accounts`へ同じ値を重複保持しない。

---

### 20.11 更新しないテーブル

ACC-002では、以下のテーブルを更新しない。

* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

資産口座登録によって、保有商品や月末資産データを自動作成しない。

---

## 21. 関連する機能要件

ACC-002は、資産口座登録および初期利用可能資産設定に関する機能要件と対応する。

主な関連要件は、以下とする。

* `4.1 概要`

  * 資産口座は月末資産管理および目的達成判定の基礎情報となる
* `4.2 登録`

  * 資産口座を新規登録できる
  * 資産口座名、資産種別、残高記録単位、利用開始年月を登録する
* `4.3 資産種別`

  * 定義済みの資産種別から選択する
* `4.4 残高記録単位`

  * 口座単位または商品単位を指定する
  * 商品単位の場合でも資産口座登録時に保有商品を自動登録しない
* `4.5 利用開始年月`

  * 資産口座を管理対象とする開始年月を設定する
* `4.6 利用可能資産区分`

  * 目的達成判定へ利用する資産かどうかを管理する
  * 資産口座登録時に初期設定を登録する
* `4.7 利用可能資産区分の変更`

  * 利用可能資産区分は履歴として管理する
  * 初期設定以降の変更は設定履歴を追加する
* 利用者境界

  * 操作対象利用者に帰属する資産口座として登録する
  * 他利用者のIDをリクエストボディから指定できない
* 資産口座名

  * 同一利用者内で重複する資産口座名を登録しない
  * 異なる利用者間では同名を許可する

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

## 22. 設計上の補足

### 22.1 資産口座登録と初期利用可能資産設定を同時登録する理由

ACC-002では、資産口座を登録すると同時に、

```text
asset_account_available_settings
```

へ初期利用可能資産設定を1件登録する。

これは、資産口座が存在するにもかかわらず、利用可能資産区分を判定できない状態を作らないためである。

概念的には、

```text
asset_accounts
だけ存在
    ↓
対象年月の
利用可能資産区分が不明
```

という状態を避ける。

そのため、資産口座登録と初期利用可能資産設定登録を1つの業務操作として扱う。

---

### 22.2 `isAvailable`を`asset_accounts`へ持たせない理由

利用可能資産区分は、資産口座の固定属性ではなく、対象年月によって変化する可能性がある。

例えば、

```text
2026-08～
利用可能

2027-01～
利用不可
```

のように、時系列で状態が変化し得る。

そのため、

```text
asset_accounts.is_available
```

のような単一フラグでは管理せず、

```text
asset_account_available_settings
```

によって履歴として管理する。

ACC-002では、その履歴の初回レコードを登録する。

---

### 22.3 初期設定の開始年月を資産口座の利用開始年月と一致させる理由

初期利用可能資産設定の

```text
start_year_month
```

は、資産口座の

```text
asset_accounts.start_year_month
```

と同じ年月とする。

これにより、

```text
資産口座の利用開始前から
利用可能資産設定が存在する
```

または、

```text
資産口座利用開始後しばらく
利用可能資産設定が存在しない
```

という不整合を防止する。

初期設定の開始年月をクライアントから別途入力させないことで、整合性ルールをバックエンドへ集約する。

---

### 22.4 初期設定の`end_year_month`をNULLとする理由

ACC-002で登録する利用可能資産設定は、登録時点における最新の設定である。

そのため、

```text
end_year_month = NULL
```

とする。

将来、利用可能資産区分が変更された場合に、現在の設定の終了年月を確定し、新しい設定履歴を追加する。

これにより、利用可能資産区分を期間履歴として管理できる。

---

### 22.5 `userId`をリクエストボディへ含めない理由

ACC-002では、操作対象利用者を

```text
X-User-Id
```

から特定する。

リクエストボディにも

```text
userId
```

を持たせると、

```text
X-User-Id = 1

Request.userId = 2
```

のような矛盾した入力を許容することになる。

そのため、利用者IDの入力経路を`X-User-Id`へ統一する。

---

### 22.6 同一利用者内で資産口座名を一意とする理由

資産口座名は、画面表示やCSVなどで利用者が資産口座を識別するために使用する。

同一利用者内で同名資産口座を許可すると、

```text
証券口座
証券口座
```

のように、どちらを指しているのか判別しにくくなる。

そのため、同一利用者内では資産口座名を一意とする。

一方、異なる利用者間では同名を許可する。

---

### 22.7 論理削除済み資産口座も重複対象とする理由

Phase1では、論理削除済み資産口座と同じ名称の再登録を許可しない。

これは、過去データ上の

```text
資産口座名
```

と現在の新しい資産口座名が同一になることで、履歴参照やCSV上の識別が曖昧になることを避けるためである。

そのため、重複確認では

```text
withTrashed()
```

を使用し、論理削除済み資産口座も対象に含める。

---

### 22.8 DBのUNIQUE制約も使用する理由

アプリケーション側で登録前に重複確認を行っても、同時実行では競合を完全には防止できない。

例えば、

```text
Request A
重複なし

Request B
重複なし

Request A
INSERT

Request B
INSERT
```

となる可能性がある。

そのため、

```text
user_id
+
name
```

のUNIQUE制約をデータベース側にも設定する。

アプリケーション側の事前確認は、利用者へ分かりやすい業務エラーを返すために使用し、DB制約は最終防衛線として使用する。

---

### 22.9 `lockForUpdate()`を使用しない理由

ACC-002は、存在しない資産口座を新規登録する処理である。

登録前にはロック対象となる資産口座行が存在しないため、

```text
lockForUpdate()
```

だけでは同名新規登録の競合を防止できない。

そのため、Phase1では

```text
事前重複確認
+
UNIQUE制約
```

によって重複登録を防止する。

---

### 22.10 利用者行をロックしない理由

同一利用者の資産口座登録を直列化するために、

```text
users
```

をロックする方式は採用しない。

利用者行をロックすると、資産口座登録とは直接関係のない同一利用者の処理まで待機させる可能性がある。

ACC-002では、必要な一意性だけをDB制約によって保証する。

---

### 22.11 資産種別を文字列でAPI公開する理由

データベースでは、`asset_type`を`smallint`などの内部コードで保持する場合がある。

ただし、クライアントへ

```text
1
2
3
```

のような内部コードを公開すると、意味が分かりにくく、DB実装にも依存する。

そのため、APIでは

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

のような意味のある文字列を使用する。

---

### 22.12 残高記録単位を文字列でAPI公開する理由

`balance_recording_unit`も、DB内部コードではなく、

```text
ACCOUNT
HOLDING
```

としてAPI公開する。

これにより、フロントエンドからも業務上の意味を理解しやすくする。

---

### 22.13 `startYearMonth`を年月単位で扱う理由

資産口座の管理開始は、日単位ではなく月末資産管理の対象年月単位で意味を持つ。

そのため、

```text
YYYY-MM-DD
```

ではなく、

```text
YYYY-MM
```

で管理する。

不要な日付精度を持ち込まない。

---

### 22.14 `balanceRecordingUnit = HOLDING`でも保有商品を同時登録しない理由

資産口座登録と保有商品登録は、異なる業務操作である。

ACC-002で保有商品まで登録すると、

* 資産口座情報
* 初期利用可能資産設定
* 複数の保有商品

を1回のAPIで扱うことになり、責務が大きくなる。

そのため、

```text
ACC-002
資産口座登録

HLD-002
保有商品登録
```

として分離する。

---

### 22.15 `balanceRecordingUnit = ACCOUNT`でも月末残高を登録しない理由

資産口座登録と月末資産残高登録は、異なるライフサイクルを持つ。

資産口座を登録した時点では、必ずしも月末残高を登録する必要はない。

そのため、ACC-002では資産口座のマスタ情報だけを登録し、月末資産残高は月末資産管理APIで扱う。

---

### 22.16 2テーブル登録を同一トランザクションにする理由

ACC-002では、

```text
asset_accounts
```

と

```text
asset_account_available_settings
```

の両方が揃って初めて正常な資産口座として扱える。

どちらか一方だけが登録される状態を防ぐため、同一トランザクションで登録する。

---

### 22.17 Repositoryごとにトランザクションを持たせない理由

トランザクション境界は、

```text
資産口座登録
+
初期利用可能資産設定登録
```

というユースケース単位で決まる。

Repository単位でトランザクションを開始すると、

```text
AssetAccountRepository
    → commit

AvailableSettingRepository
    → 失敗
```

のように、ユースケース全体の原子性を保証しにくくなる。

そのため、UseCaseでトランザクション境界を管理する。

---

### 22.18 QueryとRepositoryを分ける理由

ACC-002では、

```text
Query
    → 既存データ確認

Repository
    → 新規登録
```

と責務を分離する。

これにより、参照条件と更新処理を明確に分けられる。

また、他のACC系APIでも同じ責務分離方針を再利用しやすくなる。

---

### 22.19 Input DTOを使用してよい理由

ACC-002では、UseCaseへ渡す入力項目が複数存在する。

```text
userId
name
assetType
balanceRecordingUnit
startYearMonth
isAvailable
```

これらを個別引数として増やし続けるより、Input DTOへまとめることで、UseCaseの入力契約を明確にできる。

ただし、Phase1でかえって複雑になる場合は、過剰にDTOを増やさない。

---

### 22.20 Enumを使用する理由

`assetType`や`balanceRecordingUnit`を自由文字列として扱うと、

```text
SECURITIES
securities
Security
```

などの不整合が発生しやすい。

PHP Enumを利用することで、許可された値だけをUseCase以降で扱えるようにする。

---

### 22.21 Mass Assignmentを避ける理由

ACC-002では、クライアントから送信された入力値をそのままModelへ設定しない。

例えば、

```php
AssetAccount::create(
    $request->all(),
);
```

とすると、将来的なRequest変更によって

```text
user_id
deleted_at
```

などの意図しない項目まで更新対象へ含まれるリスクがある。

登録項目を明示的に設定することで、更新可能範囲を明確にする。

---

### 22.22 `isEnabled`をリクエストで受け取らない理由

新規登録された資産口座は、必ず利用中状態から開始する。

そのため、クライアントに

```text
isEnabled
```

を指定させる必要はない。

サーバー側で

```text
deleted_at = NULL
```

となることで、利用中状態を表現する。

---

### 22.23 `isEnabled`をレスポンスへ返す理由

新規登録直後の利用状態をクライアントが明示的に確認できるよう、

```text
isEnabled = true
```

を返却する。

一方で、

```text
deletedAt
```

のようなDB実装詳細はAPIへ公開しない。

---

### 22.24 `isAvailable`を成功レスポンスへ返す理由

ACC-002では、資産口座だけでなく初期利用可能資産設定も登録する。

利用者が指定した

```text
isAvailable
```

が正しく反映されたことを登録結果として確認できるよう、成功レスポンスへ返却する。

---

### 22.25 初期設定IDを返さない理由

ACC-002の主目的は、資産口座登録である。

初期利用可能資産設定の

```text
id
```

は、内部履歴管理用のIDであり、フロントエンドが登録直後に使用する必要はない。

そのため、

```text
assetAccountAvailableSettingId
```

は返却しない。

---

### 22.26 `endYearMonth`を返さない理由

ACC-002で作成する初期利用可能資産設定の

```text
end_year_month
```

は必ず`NULL`である。

登録結果として固定的な内部状態を返却する必要はないため、レスポンスには含めない。

---

### 22.27 Optimistic Updateを必須としない理由

ACC-002では、

* 同名重複確認
* 資産口座登録
* 初期利用可能資産設定登録

がサーバー側で実行される。

クライアントだけで成功を先取りすると、失敗時のロールバック処理が複雑になる。

そのため、Phase1では

```text
API成功
    ↓
Query invalidate
    ↓
ACC-001再取得
```

を基本とする。

---

### 22.28 Mutationの自動Retryを行わない理由

ACC-002は、新規登録APIである。

ネットワークエラー時に自動Retryすると、最初のリクエストがサーバー側で成功していた場合に同じ登録処理が再送される可能性がある。

Phase1ではIdempotency-Keyを採用しないため、Mutationの自動Retryを基本的に無効化する。

---

### 22.29 Idempotency-Keyを採用しない理由

ACC-002では、同一利用者内の同名資産口座登録を

```text
事前重複確認
+
UNIQUE制約
```

で防止する。

Idempotency-Keyを導入すると、

* Key保存
* 有効期限管理
* 同一レスポンスの再現
* 利用者との関連付け

などの追加設計が必要になる。

Phase1では、必要性に対して複雑性が大きいため採用しない。

---

### 22.30 サーバー状態を正とする理由

ACC-002成功後の最終状態は、PostgreSQLに保存されたデータを正とする。

React側のフォーム入力値だけでは、

* 採番されたID
* 実際に登録されたEnum値
* 初期利用可能資産設定
* 利用状態

を完全には保証できない。

そのため、一覧表示ではACC-001を再取得して最新状態へ同期する。

---

### 22.31 ACC-001との整合性

ACC-002で登録した資産口座は、ACC-001の一覧取得対象となる。

ACC-002成功レスポンスとACC-001の項目定義は、可能な限り同じ意味の項目について同じAPI表現を使用する。

例えば、

```text
assetType
balanceRecordingUnit
startYearMonth
isEnabled
```

について、APIごとに異なる表現を使用しない。

---

### 22.32 ACC-003との整合性

ACC-002成功後、返却された

```text
id
```

を使用してACC-003 資産口座詳細取得を実行できる。

ACC-002で登録した値が、ACC-003でも同じ意味・形式で取得できることを前提とする。

---

### 22.33 ACC-006との責務分離

ACC-002では、利用可能資産設定の初期値だけを登録する。

その後の利用可能資産区分変更は、ACC-006で扱う。

概念的には、

```text
ACC-002
    ↓
初期設定作成

ACC-006
    ↓
設定履歴変更
```

とする。

ACC-002へ将来期間の設定変更まで持たせない。

---

### 22.34 Phase1では設計を広げすぎない

ACC-002では、以下のような機能は追加しない。

* 複数資産口座の一括登録
* 保有商品の同時登録
* 月末残高の同時登録
* CSVによる資産口座登録
* 将来開始予定の利用可能設定
* 複数期間の利用可能設定同時登録
* Idempotency-Key
* 利用者行ロック

Phase1では、

```text
資産口座1件
+
初期利用可能資産設定1件
```

を安全に登録することへ責務を限定する。

---

## 22. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)