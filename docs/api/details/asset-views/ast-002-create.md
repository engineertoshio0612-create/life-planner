##  AST-002 指定年月資産状況取得

### 1 概要

操作対象となる利用者について、
指定された対象年月における
資産状況を取得する。

本APIでは、
パスパラメータの
`targetYearMonth`で指定された年月について、
操作対象利用者に属する
確定済み月末資産状況を取得する。

対象となる月末資産状況は、
以下の条件を満たすものとする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = targetYearMonth

AND

month_end_asset_snapshots.confirmed
    = true
```

対象となる月末資産状況に紐づく
月末資産残高、
商品別月末評価額および
利用可能資産設定をもとに、
指定年月の資産状況を算出する。

主に以下を取得する。

- 対象年月
- 総資産
- 利用可能資産
- 資産口座別資産状況
- 保有商品別資産状況

資産額の算出では、
資産口座の残高記録単位に応じて、
使用するデータを切り替える。

```text
口座単位で残高を記録する資産口座
    ↓
month_end_asset_balances.balance

商品単位で評価額を記録する資産口座
    ↓
month_end_holding_values.value
```

同一資産について、
月末資産残高と
商品別月末評価額を
重複して総資産へ加算しない。

利用可能資産は、
指定された`targetYearMonth`時点の
`asset_account_available_settings`をもとに
算出する。

本APIは、
資産状況を参照するための
読み取り専用APIである。

資産データの
登録・更新・削除は行わない。

---

### 2 ユースケース

利用者は、
過去または現在の
特定年月について、
確定済みの資産状況を確認する。

例えば、
以下のような場合に使用する。

- 2026年5月時点の総資産を確認する
- 過去の特定月における利用可能資産を確認する
- 特定月の資産口座別資産状況を確認する
- 特定月の商品別資産状況を確認する
- AST-003 資産推移取得APIのグラフから特定月を選択して詳細を確認する
- AST-001で表示している現在資産状況と過去月を比較する

例えば、

```text
2026-05
confirmed = true

2026-06
confirmed = true

2026-07
confirmed = false
```

の状態で、

```http
GET /api/v1/asset-summaries/2026-05
```

を実行した場合は、
`2026-05`の
確定済み月末資産状況を使用する。

最新年月が
`2026-06`であっても、
AST-002では
指定された`2026-05`を取得する。

また、

```http
GET /api/v1/asset-summaries/2026-07
```

のように、
指定年月の月末資産状況が
未確定の場合は、
資産状況を取得対象としない。

---

### 3 エンドポイント

```http
GET /api/v1/asset-summaries/{targetYearMonth}
```

例：

```http
GET /api/v1/asset-summaries/2026-05
```

---

### 4 HTTPメソッド

```http
GET
```

本APIは、
指定年月の資産状況を取得する
読み取り専用APIである。

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

本APIでは、
指定された利用者に属する
資産情報のみを
取得対象とする。

利用者IDは、
クエリパラメータ、
パスパラメータまたは
リクエストボディでは受け付けない。

利用者IDは、
ミドルウェアで設定された
利用者コンテキストから取得する。

指定年月の月末資産状況を取得する際は、
必ず以下の条件によって
利用者境界を保証する。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = targetYearMonth

AND

month_end_asset_snapshots.confirmed
    = true
```

`targetYearMonth`だけで
月末資産状況を取得してはならない。

例えば、
以下の状態とする。

```text
User A
2026-05
confirmed = true

User B
2026-06
confirmed = true
```

User Aを操作対象として、

```http
GET /api/v1/asset-summaries/2026-06
```

を実行した場合、
User Bの`2026-06`を
取得してはならない。

User Aについて
`2026-06`の確定済み月末資産状況が
存在しないものとして扱う。

月末資産残高についても、
操作対象利用者に属する
月末資産状況および
資産口座に紐づくデータのみを
取得対象とする。

概念的には、

```text
month_end_asset_balances.month_end_asset_snapshot_id
    = month_end_asset_snapshots.id

AND

asset_accounts.user_id
    = 操作対象利用者ID
```

によって
利用者境界を保証する。

商品別月末評価額についても、
操作対象利用者に属する
月末資産状況、
資産口座および
保有商品に紐づくデータのみを
取得対象とする。

概念的には、

```text
month_end_holding_values.month_end_asset_snapshot_id
    = month_end_asset_snapshots.id

AND

month_end_holding_values.holding_asset_id
    = holding_assets.id

AND

holding_assets.asset_account_id
    = asset_accounts.id

AND

asset_accounts.user_id
    = 操作対象利用者ID
```

によって
利用者境界を保証する。

利用可能資産設定についても、
操作対象利用者に属する
資産口座の設定のみを参照する。

概念的には、

```text
asset_account_available_settings.asset_account_id
    = asset_accounts.id

AND

asset_accounts.user_id
    = 操作対象利用者ID
```

によって
利用者境界を保証する。

本APIでは、
他の利用者に属する

- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品
- 利用可能資産設定

を資産状況の算出へ
含めてはならない。

`X-User-Id`が
指定されていない場合、
形式が不正な場合、
または指定された利用者が
存在しない場合は、
API共通方針に従って
エラーを返却する。

指定された`targetYearMonth`について、
他の利用者には
確定済み月末資産状況が存在していても、
その存在を
レスポンスから推測できないようにする。

---

### 6 パスパラメータ

本APIでは、
取得対象となる年月を
`targetYearMonth`で指定する。

| パラメータ名 | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `targetYearMonth` | string | ○ | 資産状況を取得する対象年月 |

形式は、

```text
YYYY-MM
```

とする。

例：

```http
GET /api/v1/asset-summaries/2026-05
```

`targetYearMonth`は、
操作対象利用者に属する
確定済み月末資産状況の
`target_year_month`を
特定するために使用する。

---

### 7 クエリパラメータ

なし。

本APIでは、
クエリパラメータを使用しない。

以下のような条件は、
クライアントから指定しない。

- 利用者ID
- 資産口座ID
- 保有商品ID
- 残高記録単位
- 確定状態
- 利用可能資産のみを取得するか
- 表示対象となる資産種別

対象年月のみを
パスパラメータの
`targetYearMonth`で指定する。

---

### 8 リクエストヘッダー

#### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/asset-summaries/2026-05
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

指定年月の資産状況を
算出するために必要な情報は、

- `targetYearMonth`
- `X-User-Id`

をもとに
サーバー側で取得する。

クライアントから、
資産額や確定状態などの
業務データを受け取らない。

---

### 10 リクエスト項目

本APIでは、
リクエストボディに
業務項目を持たない。

取得条件として使用する
クライアント指定値は、
以下とする。

| 項目 | 取得元 | 型 | 必須 | 説明 |
|---|---|---|:---:|---|
| `targetYearMonth` | パスパラメータ | string | ○ | 資産状況を取得する対象年月 |
| `X-User-Id` | リクエストヘッダー | string | ○ | 操作対象となる利用者ID |

以下の情報は、
クライアントから受け付けない。

- `userId`
- `snapshotId`
- `confirmed`
- `assetAccountId`
- `holdingAssetId`
- `balance`
- `value`
- 利用可能資産設定

これらは、
操作対象利用者および
`targetYearMonth`をもとに、
サーバー側で取得する。

---

### 11 バリデーション

#### 11.1 targetYearMonth 必須チェック

`targetYearMonth`は、
必須のパスパラメータとする。

エンドポイントは、

```http
GET /api/v1/asset-summaries/{targetYearMonth}
```

であるため、
対象年月を指定せずに
AST-002を実行することはできない。

---

#### 11.2 targetYearMonth 形式チェック

`targetYearMonth`は、
以下の形式であることを検証する。

```text
YYYY-MM
```

正常例：

```text
2026-01
2026-05
2026-12
```

不正例：

```text
2026-1
26-05
2026/05
202605
2026-13
2026-00
abc
```

年4桁、
ハイフン、
月2桁の形式とし、
月は`01`から`12`の
有効な値であることを確認する。

Laravelでは、
単純な文字列形式だけではなく、
年月として有効であることを検証する。

---

#### 11.3 targetYearMonthの存在確認

`targetYearMonth`の
形式が正常であっても、
その年月の
月末資産状況が存在することを
入力値バリデーションでは
直接検証しない。

例えば、

```text
targetYearMonth = 2026-05
```

が形式上正常であれば、
パスパラメータの
バリデーションは成功とする。

その後、
ServiceまたはQueryで、

```text
user_id = 操作対象利用者ID
AND
target_year_month = targetYearMonth
AND
confirmed = true
```

を満たす
月末資産状況を検索する。

存在しない場合の扱いは、
業務状態として
「エラーレスポンス」で定義する。

---

#### 11.4 未確定月の扱い

指定された`targetYearMonth`について、
月末資産状況が存在していても、

```text
confirmed = false
```

の場合は、
資産状況の取得対象としない。

例えば、

```text
2026-07
confirmed = false
```

の状態で、

```http
GET /api/v1/asset-summaries/2026-07
```

を実行しても、
未確定データを使用して
資産状況を返却しない。

AST-001と異なり、
1つ前の確定済み年月へ
自動的にフォールバックしない。

```text
AST-001
指定年月なし
    ↓
最新の確定済み年月を検索

AST-002
2026-07を指定
    ↓
2026-07が未確定
    ↓
別年月へフォールバックしない
```

具体的なエラーコードおよび
HTTPステータスは、
「エラーレスポンス」で定義する。

---

#### 11.5 別年月へのフォールバック禁止

指定された`targetYearMonth`の
確定済み月末資産状況が
存在しない場合、
それ以前またはそれ以降の
月末資産状況を
代替して使用してはならない。

例えば、

```text
2026-05
confirmed = true

2026-06
データなし

2026-07
confirmed = true
```

の状態で、

```http
GET /api/v1/asset-summaries/2026-06
```

を実行した場合、

```text
2026-05
```

または、

```text
2026-07
```

の資産状況を
返却してはならない。

---

#### 11.6 X-User-Id

`X-User-Id`について、
以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 正の整数として扱えること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

正常例：

```http
X-User-Id: 1
```

不正例：

```text
未指定
0
-1
abc
1.5
```

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

として扱う。

形式が不正な場合は、

```text
INVALID_USER_ID
```

として扱う。

指定された利用者が
存在しない場合、
または論理削除されている場合は、

```text
USER_NOT_FOUND
```

として扱う。

---

#### 11.7 指定年月の確定済み月末資産状況

操作対象利用者について、
以下の条件をすべて満たす
月末資産状況を検索する。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = targetYearMonth

AND

month_end_asset_snapshots.confirmed
    = true
```

概念的には、
以下の条件で取得する。

```text
user_id = 操作対象利用者ID
AND
target_year_month = targetYearMonth
AND
confirmed = true
```

指定年月以外の
月末資産状況は使用しない。

---

#### 11.8 他利用者の同一対象年月

指定された`targetYearMonth`について、
他の利用者に
確定済み月末資産状況が
存在していても、
取得対象としてはならない。

例えば、

```text
User A
2026-05
データなし

User B
2026-05
confirmed = true
```

の状態で、
User Aを操作対象として、

```http
GET /api/v1/asset-summaries/2026-05
```

を実行した場合、
User Bの資産状況を
返却してはならない。

User Aについて
指定年月の確定済み
月末資産状況が存在しないものとして扱う。

---

#### 11.9 月末資産残高の扱い

指定年月の
確定済み月末資産状況に紐づく
月末資産残高については、
資産口座の残高記録単位に従って
集計対象を決定する。

口座単位で
残高を記録する資産口座については、

```text
month_end_asset_balances.balance
```

を資産額として使用する。

商品単位で
評価額を記録する資産口座については、
月末資産残高を
総資産へ重複加算しない。

---

#### 11.10 商品別月末評価額の扱い

商品単位で
評価額を記録する資産口座については、

```text
month_end_holding_values.value
```

を資産額として使用する。

同一資産口座について、

```text
month_end_asset_balances.balance
+
month_end_holding_values.value
```

のように
両方を総資産へ加算してはならない。

残高記録単位に応じて、
どちらか一方を
資産額算出の基礎とする。

---

#### 11.11 利用可能資産設定の扱い

利用可能資産の算出では、
指定された`targetYearMonth`時点で
有効な
`asset_account_available_settings`を
参照する。

現在時点の設定ではなく、
指定年月に対応する
利用可能資産設定を使用する。

概念的には、

```text
targetYearMonth
    ↓
対象年月時点の
asset_account_available_settings
    ↓
利用可能資産を算出
```

とする。

これにより、
過去の対象年月を指定した場合でも、
その年月時点の設定に基づいて
利用可能資産を再現する。

---

#### 11.12 他利用者データの除外

指定年月の資産状況算出では、
他の利用者に属するデータを
集計対象としてはならない。

以下について、
操作対象利用者との関連を保証する。

- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `holding_assets`
- `asset_account_available_settings`

同一の`targetYearMonth`を持つ
他利用者のデータが存在していても、
資産状況へ混入させない。

---

#### 11.13 0円の扱い

月末資産残高の

```text
balance = 0
```

および、
商品別月末評価額の

```text
value = 0
```

は、
正常な業務値として扱う。

0円であることを理由として、
データ不存在とは判定しない。

```text
0円
≠
データ不存在
```

とする。

---

#### 11.14 業務状態に依存する検証

以下は、
単項目バリデーションではなく、
指定年月資産状況を取得するための
業務ルールとして扱う。

- 指定年月の月末資産状況が操作対象利用者に属すること
- 指定年月の月末資産状況が確定済みであること
- 指定年月以外へフォールバックしないこと
- 残高記録単位に応じて集計対象を切り替えること
- 月末資産残高と商品別月末評価額を重複加算しないこと
- 指定年月時点の利用可能資産設定を使用すること
- 他の利用者に属する資産情報を集計しないこと

これらの業務ルールは、
FormRequestなどの
入力値バリデーションではなく、
Serviceおよび
資産集計処理で保証する。

---

### 12 業務ルール

#### 12.1 指定年月の資産状況を取得する

本APIでは、
パスパラメータの
`targetYearMonth`で指定された年月について、
資産状況を取得する。

対象となる月末資産状況は、
以下の条件をすべて満たすものとする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = targetYearMonth

AND

month_end_asset_snapshots.confirmed
    = true
```

指定年月以外の
月末資産状況は使用しない。

---

#### 12.2 指定年月が最新である必要はない

AST-002では、
`targetYearMonth`が
最新の確定済み対象年月である必要はない。

例えば、

```text
2026-05
confirmed = true

2026-06
confirmed = true

2026-07
confirmed = true
```

の状態で、

```http
GET /api/v1/asset-summaries/2026-05
```

を実行した場合は、
`2026-05`の資産状況を取得する。

最新の`2026-07`へ
置き換えてはならない。

---

#### 12.3 未確定の指定年月は取得対象としない

指定された`targetYearMonth`について、
月末資産状況が存在していても、

```text
confirmed = false
```

の場合は、
資産状況を取得しない。

AST-002では、
未確定データを
表示用の正式な資産状況として扱わない。

---

#### 12.4 別年月へフォールバックしない

指定された`targetYearMonth`に
確定済み月末資産状況が存在しない場合、
別の確定済み年月を
代替して使用してはならない。

例えば、

```text
2026-05
confirmed = true

2026-06
データなし

2026-07
confirmed = true
```

の状態で、

```http
GET /api/v1/asset-summaries/2026-06
```

を実行した場合、

```text
2026-05
```

または

```text
2026-07
```

を返却しない。

---

#### 12.5 資産額の算出単位

資産額は、
資産口座の
残高記録単位に応じて算出する。

口座単位で
残高を記録する資産口座では、

```text
month_end_asset_balances.balance
```

を使用する。

商品単位で
評価額を記録する資産口座では、

```text
month_end_holding_values.value
```

を使用する。

概念的には、
以下とする。

```text
資産口座
    ↓
balance_recording_unit
    ├─ 口座単位
    │      ↓
    │  month_end_asset_balances.balance
    │
    └─ 商品単位
           ↓
       month_end_holding_values.value
```

---

#### 12.6 二重計上の防止

同一資産について、
月末資産残高と
商品別月末評価額を
重複して加算してはならない。

例えば、
商品単位で管理する資産口座について、

```text
month_end_asset_balances.balance
+
SUM(month_end_holding_values.value)
```

とはしない。

残高記録単位に応じて、
資産額の算出元を
一意に決定する。

---

#### 12.7 口座単位の資産額

口座単位で
残高を記録する資産口座では、
指定年月の月末資産状況に紐づく
`month_end_asset_balances.balance`を
資産口座の資産額として扱う。

概念的には、

```text
asset_account
    ↓
balance_recording_unit = 口座単位
    ↓
month_end_asset_balances.balance
```

とする。

---

#### 12.8 商品単位の資産額

商品単位で
評価額を記録する資産口座では、
指定年月の月末資産状況に紐づく
商品別月末評価額を合計し、
資産口座の資産額として扱う。

概念的には、

```text
asset_account
    ↓
balance_recording_unit = 商品単位
    ↓
month_end_holding_values.value
    ↓
資産口座単位で合計
```

とする。

資産口座の資産額は、
概念的に以下となる。

```text
SUM(month_end_holding_values.value)
```

---

#### 12.9 総資産の算出

総資産は、
指定年月における
各資産口座の資産額を
合計して算出する。

概念的には、

```text
総資産
=
口座単位で管理する資産口座の残高合計
+
商品単位で管理する資産口座の商品別評価額合計
```

とする。

```text
totalAssets
=
SUM(accountUnitBalances)
+
SUM(holdingUnitValues)
```

---

#### 12.10 利用可能資産の算出

利用可能資産は、
指定された`targetYearMonth`時点の
`asset_account_available_settings`を
もとに算出する。

現在時点の設定ではなく、
指定年月に対応する設定を使用する。

概念的には、

```text
targetYearMonth
    ↓
対象年月時点で
利用可能な資産口座を特定
    ↓
該当する資産口座の資産額を合計
    ↓
利用可能資産
```

とする。

---

#### 12.11 過去月の利用可能資産を再現する

過去の`targetYearMonth`を指定した場合は、
その年月時点の
利用可能資産設定を使用する。

例えば、

```text
2026-05
口座A available = true

2026-06
口座A available = false
```

の場合、

```http
GET /api/v1/asset-summaries/2026-05
```

では、

```text
available = true
```

として扱う。

現在の設定が
`false`であることを理由として、
過去月の結果を書き換えてはならない。

---

#### 12.12 資産口座別資産状況

資産口座別資産状況では、
指定年月における
各資産口座の資産額を返却する。

各資産口座について、
残高記録単位に応じて
資産額を算出する。

```text
口座単位
    ↓
balance

商品単位
    ↓
SUM(value)
```

---

#### 12.13 保有商品別資産状況

商品単位で
評価額を記録する資産口座については、
保有商品別の資産状況を返却する。

各保有商品の資産額は、

```text
month_end_holding_values.value
```

を使用する。

口座単位で
残高を記録する資産口座について、
架空の商品別内訳を生成しない。

---

#### 12.14 0円の資産

以下は、
正常な業務値として扱う。

```text
month_end_asset_balances.balance = 0
```

```text
month_end_holding_values.value = 0
```

0円であることを理由として、
資産口座や
保有商品を
取得対象から除外しない。

---

#### 12.15 利用者境界

指定年月の資産状況算出では、
操作対象利用者に属する
資産情報のみを使用する。

他の利用者に属する

- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品
- 利用可能資産設定

を集計対象へ含めてはならない。

同一の`targetYearMonth`を持つ
他利用者のデータが存在していても、
取得結果へ混入させない。

---

#### 12.16 現在状態で過去月を再構成しない

AST-002では、
指定年月の資産状況を
現在の資産口座状態や
現在の利用可能資産設定だけで
再構成してはならない。

指定年月に対応する

- 月末資産残高
- 商品別月末評価額
- 利用可能資産設定

を使用して
資産状況を算出する。

---

#### 12.17 集計結果を保存しない

本APIで算出した

- 総資産
- 利用可能資産
- 資産口座別資産額
- 保有商品別資産額

を、
専用の集計テーブルへ保存しない。

既存の月末資産データから
取得時に算出する。

---

### 13 取得条件

最初に、
指定年月の
確定済み月末資産状況を取得する。

取得条件は、
以下とする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.target_year_month
    = targetYearMonth

AND

month_end_asset_snapshots.confirmed
    = true
```

対象となる
`month_end_asset_snapshots`を
特定した後、
そのスナップショットに紐づく
月末資産データを取得する。

```text
snapshotId
    ↓
month_end_asset_balances
```

および、

```text
snapshotId
    ↓
month_end_holding_values
```

とする。

さらに、
資産口座および
保有商品との関連をもとに、
資産口座別・保有商品別の
表示情報を構成する。

利用可能資産については、
指定された`targetYearMonth`時点の
`asset_account_available_settings`を
参照する。

---

### 14 集計方法

#### 11 総資産

総資産は、
指定年月における
全資産口座の
資産額を合計する。

```text
totalAssets
=
口座単位資産額の合計
+
商品単位資産額の合計
```

---

#### 12 利用可能資産

利用可能資産は、
指定年月時点で
利用可能資産として扱う
資産口座の資産額のみを
合計する。

```text
availableAssets
=
利用可能な資産口座の
assetAmount合計
```

---

#### 13 資産口座別

資産口座ごとに、
`balance_recording_unit`を判定する。

口座単位の場合：

```text
assetAmount
=
month_end_asset_balances.balance
```

商品単位の場合：

```text
assetAmount
=
SUM(month_end_holding_values.value)
```

---

#### 14 保有商品別

商品単位で管理する
資産口座について、
保有商品ごとの

```text
month_end_holding_values.value
```

を資産額として使用する。

```text
holdingAsset.assetAmount
=
month_end_holding_values.value
```

---

#### 15 集計値の整合性

正常なデータ状態では、
以下が成立することを前提とする。

```text
totalAssets
=
SUM(assetAccounts.assetAmount)
```

また、
商品単位で管理する
各資産口座については、

```text
assetAccount.assetAmount
=
SUM(holdingAssets.assetAmount)
```

となる。

---

### 15 レスポンス

取得成功時は、
指定された対象年月における
資産状況を返却する。

HTTPステータスは、

```http
200 OK
```

とする。

レスポンスは、
API共通方針で定めた
Envelope形式を使用する。

概念例：

```json
{
  "data": {
    "targetYearMonth": "2026-05",
    "totalAssets": 3200000,
    "availableAssets": 1400000,
    "assetAccounts": [
      {
        "assetAccountId": "1",
        "name": "普通預金",
        "assetAmount": 900000,
        "available": true,
        "holdingAssets": []
      },
      {
        "assetAccountId": "2",
        "name": "証券口座",
        "assetAmount": 2300000,
        "available": false,
        "holdingAssets": [
          {
            "holdingAssetId": "10",
            "name": "投資信託A",
            "assetAmount": 1400000
          },
          {
            "holdingAssetId": "11",
            "name": "投資信託B",
            "assetAmount": 900000
          }
        ]
      }
    ]
  }
}
```

---

#### 15.1 targetYearMonth

レスポンスの
`targetYearMonth`には、
リクエストで指定された
対象年月を返却する。

```http
GET /api/v1/asset-summaries/2026-05
```

の場合、

```json
{
  "targetYearMonth": "2026-05"
}
```

となる。

別年月へ
置き換えて返却しない。

---

#### 15.2 口座単位のレスポンス

口座単位で
残高を記録する資産口座では、
`holdingAssets`を
空配列として返却する。

```json
{
  "assetAccountId": "1",
  "name": "普通預金",
  "assetAmount": 900000,
  "available": true,
  "holdingAssets": []
}
```

---

#### 15.3 商品単位のレスポンス

商品単位で
評価額を記録する資産口座では、
資産口座の資産額と
保有商品別資産状況を返却する。

```json
{
  "assetAccountId": "2",
  "name": "証券口座",
  "assetAmount": 2300000,
  "available": false,
  "holdingAssets": [
    {
      "holdingAssetId": "10",
      "name": "投資信託A",
      "assetAmount": 1400000
    },
    {
      "holdingAssetId": "11",
      "name": "投資信託B",
      "assetAmount": 900000
    }
  ]
}
```

---

### 16 レスポンス項目

#### 16.1 data

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `targetYearMonth` | string | × | 指定された確定済み対象年月 |
| `totalAssets` | integer | × | 指定年月における総資産 |
| `availableAssets` | integer | × | 指定年月における利用可能資産 |
| `assetAccounts` | array | × | 資産口座別資産状況 |

金額は、
日本円の整数値として返却する。

---

#### 16.2 assetAccounts

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `assetAccountId` | string | × | 資産口座ID |
| `name` | string | × | 指定年月時点の表示対象となる資産口座名 |
| `assetAmount` | integer | × | 指定年月における資産口座の資産額 |
| `available` | boolean | × | 指定年月時点で利用可能資産として扱うか |
| `holdingAssets` | array | × | 保有商品別資産状況 |

`assetAccountId`は、
API共通方針に従って
stringとして返却する。

---

#### 16.3 holdingAssets

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `holdingAssetId` | string | × | 保有商品ID |
| `name` | string | × | 保有商品名 |
| `assetAmount` | integer | × | 指定年月における商品別月末評価額 |

`holdingAssetId`は、
API共通方針に従って
stringとして返却する。

口座単位で管理する
資産口座の場合は、

```json
"holdingAssets": []
```

とする。

---

#### 16.4 totalAssets

`totalAssets`は、
指定年月における
総資産額を返却する。

```text
totalAssets
=
口座単位資産額の合計
+
商品単位資産額の合計
```

---

#### 16.5 availableAssets

`availableAssets`は、
指定年月時点で
利用可能資産として扱われる
資産口座の資産額合計を返却する。

現在の利用可能資産設定ではなく、
`targetYearMonth`時点の設定を
反映した値とする。

---

#### 16.6 assetAmount

`assetAmount`は、
資産口座または
保有商品の資産額を表す。

資産口座の場合は、
残高記録単位に応じて
算出方法が異なる。

```text
口座単位
    ↓
month_end_asset_balances.balance

商品単位
    ↓
SUM(month_end_holding_values.value)
```

保有商品の場合は、

```text
month_end_holding_values.value
```

を使用する。

---

#### 16.7 available

`available`は、
指定された`targetYearMonth`時点で
その資産口座を
利用可能資産として扱うかを表す。

```json
{
  "available": true
}
```

または、

```json
{
  "available": false
}
```

とする。

---

#### 16.8 返却しない情報

本APIでは、
以下の情報は返却しない。

- `user_id`
- `month_end_asset_snapshot_id`
- `confirmed`
- `month_end_asset_balances.id`
- `month_end_holding_values.id`
- `balance_recording_unit`
- `asset_account_available_settings`
- `created_at`
- `updated_at`

指定年月資産状況の表示に
必要な情報のみを返却する。

月末資産データそのものではなく、
それらをもとに算出した
表示用の資産状況を返却する。

---

### 17 エラーレスポンス

本APIで発生する
主なエラーは、
以下とする。

| HTTPステータス | エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`が指定されていない |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`の形式が不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 指定された利用者が存在しない、または論理削除されている |
| `404 Not Found` | `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` | 指定年月について、操作対象利用者に属する確定済み月末資産状況が存在しない |
| `422 Unprocessable Entity` | `VALIDATION_ERROR` | `targetYearMonth`の形式が不正 |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバーエラーが発生した |

エラーレスポンス形式は、
API共通方針に従う。

概念例：

```json
{
  "error": {
    "code": "CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND",
    "message": "指定年月の確定済み月末資産状況が存在しません。",
    "details": []
  },
  "requestId": "01HXXXXXXXXXXXXXXX"
}
```

---

#### 17.1 USER_CONTEXT_REQUIRED

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

指定年月資産状況の
取得処理は実行しない。

---

#### 17.2 INVALID_USER_ID

`X-User-Id`が
API共通方針で定める
利用者ID形式を満たさない場合は、

```text
INVALID_USER_ID
```

を返却する。

例えば、
以下を不正とする。

```text
0
-1
abc
1.5
```

---

#### 17.3 USER_NOT_FOUND

`X-User-Id`で
指定された利用者が
存在しない場合、
または論理削除されている場合は、

```text
USER_NOT_FOUND
```

を返却する。

他の利用者の
同一対象年月の資産情報を
代替して返却してはならない。

---

#### 17.4 targetYearMonthのバリデーションエラー

`targetYearMonth`が
指定された形式を満たさない場合は、

```text
VALIDATION_ERROR
```

を返却する。

例えば、
以下の場合を含む。

```text
2026-1
26-05
2026/05
202605
2026-00
2026-13
abc
```

レスポンス例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "targetYearMonth",
        "reason": "invalidFormat",
        "message": "対象年月の形式を確認してください。"
      }
    ]
  },
  "requestId": "01HXXXXXXXXXXXXXXX"
}
```

---

#### 17.5 指定年月の月末資産状況が存在しない場合

指定された`targetYearMonth`について、
操作対象利用者に属する
月末資産状況自体が
存在しない場合は、

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

を返却する。

例えば、

```text
User A
2026-05
データなし
```

の状態で、

```http
GET /api/v1/asset-summaries/2026-05
```

を実行した場合が該当する。

別年月の資産状況を
代替して返却しない。

---

#### 17.6 指定年月が未確定の場合

指定された`targetYearMonth`について、
月末資産状況が存在していても、

```text
confirmed = false
```

の場合は、
確定済み月末資産状況が
存在しないものとして扱う。

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

を返却する。

例えば、

```text
2026-07
confirmed = false
```

の状態で、

```http
GET /api/v1/asset-summaries/2026-07
```

を実行した場合が該当する。

未確定データを
表示用の正式な資産状況として
返却してはならない。

---

#### 17.7 他利用者にのみ指定年月のデータが存在する場合

以下の状態を用意する。

```text
User A
2026-05
データなし

User B
2026-05
confirmed = true
```

User Aを操作対象として、

```http
GET /api/v1/asset-summaries/2026-05
```

を実行した場合は、

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

として扱う。

User Bのデータを
取得してはならない。

また、
User Bに
指定年月のデータが存在することを
レスポンスから推測できないようにする。

---

#### 17.8 別年月へのフォールバック禁止

指定された年月に
確定済み月末資産状況が
存在しない場合でも、
前月または翌月のデータを
代替して使用しない。

例えば、

```text
2026-05
confirmed = true

2026-06
データなし

2026-07
confirmed = true
```

の状態で、

```http
GET /api/v1/asset-summaries/2026-06
```

を実行した場合は、

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

を返却する。

---

#### 17.9 集計対象データの不整合

指定年月の
確定済み月末資産状況が
存在しているにもかかわらず、
データ不整合によって
正常に資産状況を算出できない場合は、
クライアント起因のエラーとして扱わない。

例えば、
アプリケーションが前提とする
月末資産データの整合性が
失われている場合は、
想定外エラーとして扱う。

クライアントへ
データベース内部の状態や
不整合内容を
直接公開しない。

---

#### 17.10 INTERNAL_SERVER_ERROR

データベース接続エラーなど、
想定外のサーバーエラーが
発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスには、
以下の内部情報を含めない。

- SQL
- テーブル名
- カラム名
- PostgreSQLのエラー内容
- 制約名
- PHP内部エラー
- Laravel内部例外
- スタックトレース

詳細は、
サーバーログへ記録する。

---

### 18 HTTPステータス

本APIで使用する
HTTPステータスは、
以下とする。

| HTTPステータス | 用途 |
|---|---|
| `200 OK` | 指定年月資産状況の取得成功 |
| `400 Bad Request` | 利用者コンテキストの指定不備 |
| `404 Not Found` | 利用者、または指定年月の確定済み月末資産状況が存在しない |
| `422 Unprocessable Entity` | `targetYearMonth`のバリデーションエラー |
| `500 Internal Server Error` | 想定外のサーバーエラー |

---

#### 18.1 200の扱い

指定された`targetYearMonth`について、
操作対象利用者に属する
確定済み月末資産状況が存在し、
資産状況を正常に算出できた場合は、

```text
200 OK
```

を返却する。

資産額が0円であっても、
正常なデータとして扱う。

---

#### 18.2 400の扱い

以下の場合は、

```text
400 Bad Request
```

を返却する。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正

---

#### 18.3 404の扱い

以下の場合は、

```text
404 Not Found
```

を返却する。

- 指定された利用者が存在しない
- 指定された利用者が論理削除されている
- 指定年月の月末資産状況が存在しない
- 指定年月の月末資産状況が未確定
- 指定年月の確定済み月末資産状況が他利用者にのみ存在する

---

#### 18.4 422の扱い

`targetYearMonth`が
`YYYY-MM`形式ではない場合、
または有効な年月として扱えない場合は、

```text
422 Unprocessable Entity
```

を返却する。

例えば、

```text
2026-13
```

は、
文字列構造としては年月に見えるが、
有効な月ではないため
バリデーションエラーとする。

---

### 19 副作用

本APIは、
読み取り専用APIである。

以下のデータに対する
登録・更新・削除を行わない。

- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `holding_assets`
- `asset_account_available_settings`

また、
指定年月資産状況として算出した

- `totalAssets`
- `availableAssets`
- 資産口座別資産額
- 保有商品別資産額

を、
データベースへ保存しない。

本APIの実行によって、
月末資産状況の
`confirmed`を変更してはならない。

利用可能資産設定についても、
参照のみとし、
設定を変更しない。

---

### 20 トランザクション

本APIは、
読み取り専用APIであるため、
Phase1では
明示的な更新トランザクションを
使用しない。

概念的には、
以下の処理を行う。

```text
指定年月の確定済み月末資産状況を取得
    ↓
月末資産残高を取得
    ↓
商品別月末評価額を取得
    ↓
指定年月時点の利用可能資産設定を取得
    ↓
資産状況を集計
    ↓
レスポンス
```

データの更新を伴わないため、
単純な参照処理のためだけに

```php
DB::transaction(...)
```

を使用しない。

ただし、
複数のSELECT間で
厳密な読み取り一貫性が
必要となる要件が
将来的に追加された場合は、
トランザクション分離レベルを含めて
別途検討する。

---

### 21 ロック

本APIでは、
行ロックを使用しない。

以下のような
排他ロックは行わない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

本APIは、
指定年月の資産状況を
参照するだけであるため、
月末資産状況、
月末資産残高、
商品別月末評価額などの
更新処理を不要にブロックしない。

---

### 22 キャッシュ

Phase1では、
AST-002専用の
サーバー側アプリケーションキャッシュは
使用しない。

指定年月の資産状況は、
確定済み月末資産データおよび
対象年月時点の
利用可能資産設定から算出する。

Phase1では、
キャッシュ無効化処理を
追加するよりも、
リクエストごとに
データベースから取得して
算出することを優先する。

React Query等による
クライアント側キャッシュについては、
フロントエンド共通設計に従って
利用してよい。

---

### 23 冪等性

本APIは、
読み取り専用のGET APIであるため、
冪等である。

同一のデータ状態に対して、
同一利用者が
同一の`targetYearMonth`を指定して
複数回実行しても、
業務データの状態は変化しない。

例えば、

```text
1回目
GET /api/v1/asset-summaries/2026-05
    ↓
200 OK

2回目
GET /api/v1/asset-summaries/2026-05
    ↓
200 OK
```

本APIの実行回数によって、

- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品
- 利用可能資産設定

が変更されることはない。

ただし、
リクエスト間に
別の処理によって
参照対象データが変更された場合は、
同じ`targetYearMonth`でも
レスポンス内容が
変化する可能性がある。

これは、
AST-002自体の副作用によるものではなく、
参照対象データが
変更されたためである。

本APIはGETであり、
重複登録などの副作用がないため、
`Idempotency-Key`は使用しない。

---

### 24 関連テーブル

#### 21 month_end_asset_snapshots

指定年月資産状況の
対象となる月末資産状況を
特定するために使用する。

本APIでは、
操作対象利用者について、
指定された`targetYearMonth`と一致する
確定済み月末資産状況を取得する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 月末資産状況ID |
| `user_id` | 利用者境界確認 |
| `target_year_month` | 指定年月との一致確認 |
| `confirmed` | 確定済み月末資産状況のみを対象とする判定 |

取得条件は、
以下とする。

```text
user_id = 操作対象利用者ID
AND
target_year_month = targetYearMonth
AND
confirmed = true
```

指定年月以外の
月末資産状況は使用しない。

本APIでは、
`month_end_asset_snapshots`を更新しない。

---

#### 22 month_end_asset_balances

口座単位で
残高を記録する資産口座について、
指定年月の月末資産残高を
取得するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `month_end_asset_snapshot_id` | 対象となる月末資産状況との関連 |
| `asset_account_id` | 資産口座との関連 |
| `balance` | 口座単位の資産額 |

取得対象は、
AST-002で特定した
指定年月の確定済み月末資産状況に
紐づくレコードとする。

```text
month_end_asset_snapshot_id
    = 指定年月の確定済みsnapshotId
```

口座単位の資産口座についてのみ、
`balance`を資産額として使用する。

商品単位の資産口座については、
`balance`を総資産へ
重複加算しない。

本APIでは、
`month_end_asset_balances`を更新しない。

---

#### 23 month_end_holding_values

商品単位で
評価額を記録する資産口座について、
指定年月の商品別月末評価額を
取得するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `month_end_asset_snapshot_id` | 対象となる月末資産状況との関連 |
| `holding_asset_id` | 保有商品との関連 |
| `value` | 商品単位の資産額 |

取得対象は、
AST-002で特定した
指定年月の確定済み月末資産状況に
紐づくレコードとする。

```text
month_end_asset_snapshot_id
    = 指定年月の確定済みsnapshotId
```

商品単位の資産口座では、
同一資産口座に属する
`value`を合計して、
資産口座の資産額を算出する。

```text
SUM(month_end_holding_values.value)
```

本APIでは、
`month_end_holding_values`を更新しない。

---

#### 24 asset_accounts

資産口座の
表示情報および
残高記録単位を取得するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 資産口座ID |
| `user_id` | 利用者境界確認 |
| `name` | 資産口座名 |
| `balance_recording_unit` | 資産額の算出元判定 |

資産口座の
`balance_recording_unit`に応じて、
資産額の算出元を切り替える。

```text
口座単位
    → month_end_asset_balances.balance

商品単位
    → SUM(month_end_holding_values.value)
```

他の利用者に属する
資産口座を
集計対象へ含めない。

本APIでは、
`asset_accounts`を更新しない。

---

#### 25 holding_assets

商品単位で
評価額を記録する資産口座について、
保有商品の表示情報を
取得するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 保有商品ID |
| `asset_account_id` | 所属する資産口座との関連 |
| `name` | 保有商品名 |

`month_end_holding_values`との関連は、
以下とする。

```text
month_end_holding_values.holding_asset_id
    = holding_assets.id
```

資産口座との関連は、
以下とする。

```text
holding_assets.asset_account_id
    = asset_accounts.id
```

本APIでは、
`holding_assets`を更新しない。

---

#### 26 asset_account_available_settings

指定された`targetYearMonth`時点で、
各資産口座を
利用可能資産として扱うかを
判定するために使用する。

判定対象年月は、
リクエストで指定された
`targetYearMonth`とする。

概念的には、
以下を判定する。

```text
asset_account_id
+
targetYearMonth
    ↓
指定年月時点で
利用可能資産として扱うか
```

現在時点の設定ではなく、
指定年月に対応する
利用可能資産設定を使用する。

本APIでは、
`asset_account_available_settings`を更新しない。

---

#### 27 users

`X-User-Id`で指定された
操作対象利用者の
存在確認に使用する。

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

#### 28 関連しないテーブル

本APIでは、
以下のテーブルを
指定年月資産状況取得のために
参照しない。

- `net_incomes`
- `objectives`
- `assessment_histories`

AST-002は、
指定年月における
資産状況表示に
責務を限定する。

---

### 25 関連する機能要件

- 指定年月の資産状況
  - 指定された対象年月の確定済み月末資産状況を使用する
  - 指定年月が最新である必要はない
  - 指定年月が未確定の場合は表示対象としない
  - 指定年月に確定済みデータが存在しない場合、別年月へフォールバックしない

- 総資産
  - 口座単位の資産口座は月末資産残高を使用する
  - 商品単位の資産口座は商品別月末評価額を使用する
  - 月末資産残高と商品別月末評価額を重複加算しない
  - 各資産口座の資産額合計を総資産とする

- 利用可能資産
  - 指定年月時点の利用可能資産設定を使用する
  - 利用可能資産として扱う資産口座のみを合計する
  - 現在の利用可能資産設定だけで過去月を再構成しない

- 資産口座別資産状況
  - 指定年月の資産口座ごとの資産額を取得できる
  - 口座単位と商品単位で算出方法を切り替える

- 保有商品別資産状況
  - 商品単位で管理する資産口座について保有商品別評価額を取得できる
  - 口座単位の資産口座について架空の商品別内訳を生成しない

- 利用者境界
  - 操作対象利用者に属する資産情報のみを取得する
  - 他の利用者に属する同一対象年月の資産情報を取得しない
  - 他の利用者に属する資産情報を集計しない

- API共通
  - IDはAPIレスポンス上stringとして扱う
  - JSONフィールド名はcamelCaseとする
  - エラー時は共通エラーレスポンス形式を使用する

具体的な章番号は、
`functional-requirements.md`の
最新定義に従う。

---

### 26 設計上の補足

#### 26.1 targetYearMonthをURLに含める理由

AST-002は、
特定の対象年月という
リソースを明示して取得する。

そのため、

```http
/api/v1/asset-summaries/{targetYearMonth}
```

として、
対象年月をパスで表現する。

---

#### 26.2 クエリパラメータではなくパスにする理由

`targetYearMonth`は、
一覧を絞り込むための
任意条件ではない。

取得対象となる
資産状況そのものを
一意に特定するための
主要識別条件である。

そのため、

```http
/asset-summaries?targetYearMonth=2026-05
```

ではなく、

```http
/asset-summaries/2026-05
```

を採用する。

---

#### 26.3 AST-001とAST-002を分ける理由

AST-001とAST-002では、
返却する資産状況の構造は
同一である。

ただし、
対象年月の決定方法が異なる。

```text
AST-001
    ↓
サーバーが最新確定月を決定

AST-002
    ↓
クライアントが対象年月を指定
```

この違いを
APIとして明確に分離する。

---

#### 26.4 未確定月を取得しない理由

未確定の月末資産状況は、
入力途中または
修正途中である可能性がある。

そのため、
正式な資産状況表示には使用しない。

AST-002でも、
AST-001と同様に
確定済みデータのみを対象とする。

---

#### 26.5 別年月へフォールバックしない理由

AST-002では、
利用者が明示的に
対象年月を指定している。

そのため、
指定年月が存在しない場合に
別年月を返却すると、

```text
利用者が指定した年月
```

と

```text
実際に返却した年月
```

が異なってしまう。

そのため、
指定年月に確定済みデータが
存在しない場合は、
エラーとして扱う。

---

#### 26.6 AST-001と集計ロジックを共通化する理由

AST-001とAST-002は、
対象となる`snapshotId`を
決定した後の処理が同一である。

```text
snapshotId
    ↓
資産口座別集計
    ↓
保有商品別集計
    ↓
利用可能資産判定
    ↓
総資産集計
```

そのため、
共通のQuery、
DTO、
集計処理、
API Resourceを使用する。

これにより、
AST-001とAST-002で
集計結果の差異が生じることを防止する。

---

#### 26.7 指定年月時点の利用可能資産設定を使用する理由

AST-002では、
過去月の資産状況を
表示する場合がある。

現在の利用可能資産設定を
使用すると、
過去月の資産状況を
正しく再現できない。

そのため、
`targetYearMonth`時点の
設定を使用する。

---

#### 26.8 集計結果を保存しない理由

`totalAssets`や
`availableAssets`は、
既存の月末資産データから
導出可能である。

Phase1では、
集計結果を
別テーブルへ保存しない。

これにより、
元データとの
同期問題を避ける。

---

#### 26.9 balance_recording_unitを返却しない理由

資産額の算出方法は、
バックエンド側で
残高記録単位に応じて
解決済みである。

フロントエンドは、
`assetAmount`を
表示すればよい。

そのため、
Phase1のAST-002では
`balance_recording_unit`を
レスポンスへ返却しない。

---

#### 26.10 holdingAssetsを空配列で返す理由

口座単位の資産口座では、
保有商品別の資産状況を持たない。

この場合も、

```json
{
  "holdingAssets": []
}
```

として返却する。

`null`と配列を混在させず、
React・TypeScript側で
一貫した型として扱えるようにする。

---

#### 26.11 GETを採用する理由

AST-002は、
指定年月の資産状況を
参照するだけのAPIである。

データの登録、
更新、
削除を行わないため、
HTTPメソッドには
`GET`を採用する。

---

#### 26.12 AST-002が冪等である理由

同一のデータ状態で
同じ`targetYearMonth`を指定して
複数回実行しても、
業務データを変更しない。

そのため、
本APIは冪等である。

参照対象データが
別処理によって変更された場合に
レスポンスが変化しても、
AST-002自体による
副作用ではない。

---

#### 26.13 Idempotency-Keyを使用しない理由

AST-002は、
読み取り専用のGET APIである。

再実行によって
データの重複登録などが
発生しないため、
`Idempotency-Key`は使用しない。

---

#### 26.14 サーバーキャッシュを採用しない理由

指定年月の資産状況は、
確定済みデータをもとに
算出するため、
比較的変更頻度は低い。

ただし、
Phase1では
キャッシュ管理を追加せず、
実装を単純に保つ。

性能上の必要性が生じた場合は、
将来的に
対象年月単位のキャッシュを
検討する。

---

### 27 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)
