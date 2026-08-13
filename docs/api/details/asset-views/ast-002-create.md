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

### 26 テスト観点

#### 26.1 正常系

操作対象利用者について、
指定年月の
確定済み月末資産状況および
必要な資産データが存在する状態で
AST-002を実行する。

以下を確認する。

- `200 OK`となること
- 指定した`targetYearMonth`が返却されること
- 総資産が正しく算出されること
- 利用可能資産が正しく算出されること
- 資産口座別資産状況が返却されること
- 商品単位の資産口座について保有商品別資産状況が返却されること
- データベースが更新されないこと

---

#### 26.2 最新ではない確定済み年月

以下の状態を用意する。

```text
2026-05
confirmed = true

2026-06
confirmed = true

2026-07
confirmed = true
```

以下を実行する。

```http
GET /api/v1/asset-summaries/2026-05
```

期待結果：

```text
targetYearMonth = 2026-05
```

となること。

最新の`2026-07`へ
置き換わらないこと。

---

#### 26.3 指定年月が未確定

以下の状態を用意する。

```text
2026-06
confirmed = true

2026-07
confirmed = false
```

以下を実行する。

```http
GET /api/v1/asset-summaries/2026-07
```

期待結果：

```text
404 Not Found
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

未確定の`2026-07`を
資産状況へ使用しないこと。

---

#### 26.4 指定年月が存在しない

以下の状態を用意する。

```text
2026-05
confirmed = true

2026-07
confirmed = true
```

以下を実行する。

```http
GET /api/v1/asset-summaries/2026-06
```

期待結果：

```text
404 Not Found
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

---

#### 26.5 別年月へフォールバックしないこと

以下の状態を用意する。

```text
2026-05
confirmed = true

2026-06
データなし

2026-07
confirmed = true
```

`2026-06`を指定する。

以下を確認する。

- `2026-05`を返却しないこと
- `2026-07`を返却しないこと
- `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND`となること

---

#### 26.6 targetYearMonth正常値

以下の値を指定する。

```text
2026-01
2026-06
2026-12
```

形式バリデーションを通過すること。

---

#### 26.7 targetYearMonth形式不正

以下の値を指定する。

```text
2026-1
26-05
2026/05
202605
abc
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 26.8 targetYearMonthの月が0

```text
2026-00
```

を指定する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 26.9 targetYearMonthの月が13

```text
2026-13
```

を指定する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 26.10 口座単位の資産口座

口座単位で
残高を記録する資産口座を用意する。

```text
balance_recording_unit = 口座単位

month_end_asset_balances.balance
    = 1000000
```

期待結果：

```text
assetAmount = 1000000
```

となること。

---

#### 26.11 商品単位の資産口座

商品単位で
評価額を記録する資産口座を用意する。

```text
商品A
value = 500000

商品B
value = 300000
```

期待結果：

```text
assetAmount = 800000
```

となること。

---

#### 26.12 商品単位の複数保有商品

商品単位の資産口座に
複数の商品別月末評価額を用意する。

以下を確認する。

- すべての`value`が資産口座単位で合計されること
- 各保有商品の`assetAmount`に個別の`value`が返却されること
- 資産口座の`assetAmount`と保有商品別合計が一致すること

---

#### 26.13 二重計上しないこと

商品単位の資産口座について、
同じ指定年月に

```text
month_end_asset_balances.balance
    = 900000

month_end_holding_values
    = 500000 + 300000
```

のようなデータを用意する。

期待結果：

```text
assetAmount = 800000
```

とし、

```text
1700000
```

とならないこと。

---

#### 26.14 口座単位で商品別評価額を加算しないこと

口座単位の資産口座について、
同じ指定年月に
商品別月末評価額が存在する状態を用意する。

以下を確認する。

- `month_end_asset_balances.balance`のみを使用すること
- 商品別月末評価額を`totalAssets`へ加算しないこと

---

#### 26.15 複数資産口座の総資産

以下の資産口座を用意する。

```text
口座A
口座単位
assetAmount = 1000000

口座B
口座単位
assetAmount = 500000

口座C
商品単位
商品1 = 800000
商品2 = 700000
```

期待結果：

```text
totalAssets
=
1000000
+
500000
+
800000
+
700000
=
3000000
```

となること。

---

#### 26.16 totalAssetsと資産口座別合計

正常なデータについて、

```text
totalAssets
=
SUM(assetAccounts.assetAmount)
```

となることを確認する。

---

#### 26.17 0円の月末資産残高

口座単位の資産口座について、

```text
balance = 0
```

を用意する。

以下を確認する。

- 未登録扱いにならないこと
- `assetAmount = 0`として返却されること
- 0円として正常に集計されること

---

#### 26.18 0円の商品別月末評価額

商品単位の保有商品について、

```text
value = 0
```

を用意する。

以下を確認する。

- 保有商品が一覧から除外されないこと
- `assetAmount = 0`として返却されること
- 未登録扱いにならないこと

---

#### 26.19 指定年月の利用可能資産

指定年月について、
以下の状態を用意する。

```text
口座A
assetAmount = 1000000
available = true

口座B
assetAmount = 2000000
available = false

口座C
assetAmount = 500000
available = true
```

期待結果：

```text
totalAssets = 3500000

availableAssets
=
1000000
+
500000
=
1500000
```

となること。

---

#### 26.20 過去月の利用可能資産設定

同一資産口座について、
以下の設定を用意する。

```text
2026-05
available = true

2026-06
available = false
```

`2026-05`を指定する。

期待結果：

```text
available = true
```

となること。

現在または
`2026-06`の設定を
誤って使用しないこと。

---

#### 26.21 すべて利用可能

指定年月時点で、
すべての資産口座が
利用可能資産として扱われる状態とする。

期待結果：

```text
availableAssets
=
totalAssets
```

となること。

---

#### 26.22 利用可能資産が0円

指定年月時点で、
すべての資産口座が
利用可能資産として
扱われない状態とする。

期待結果：

```text
availableAssets = 0
```

となること。

---

#### 26.23 口座単位のholdingAssets

口座単位の資産口座について、
レスポンスを確認する。

期待結果：

```json
{
  "holdingAssets": []
}
```

となること。

架空の商品別内訳を
生成しないこと。

---

#### 26.24 商品単位のholdingAssets

商品単位の資産口座について、
指定年月に紐づく
保有商品別資産状況が
返却されることを確認する。

各項目について、

- `holdingAssetId`
- `name`
- `assetAmount`

が正しく返却されること。

---

#### 26.25 他利用者の同一年月を使用しないこと

以下の状態を用意する。

```text
User A
2026-05
データなし

User B
2026-05
confirmed = true
```

User Aとして
`2026-05`を指定する。

期待結果：

```text
404 Not Found
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

となること。

User Bの資産状況を
返却しないこと。

---

#### 26.26 他利用者の月末資産残高を含めないこと

User AとUser Bについて、
同一対象年月に
月末資産残高を用意する。

User Aとして
AST-002を実行する。

以下を確認する。

- User Aの月末資産残高のみ集計されること
- User Bの残高が`totalAssets`へ加算されないこと

---

#### 26.27 他利用者の商品別月末評価額を含めないこと

User AとUser Bについて、
同一対象年月に
商品別月末評価額を用意する。

User Aとして
AST-002を実行する。

以下を確認する。

- User Aの商品別月末評価額のみ集計されること
- User Bの商品別月末評価額が混入しないこと

---

#### 26.28 他利用者の利用可能資産設定を使用しないこと

User AとUser Bについて、
同一対象年月に
利用可能資産設定を用意する。

User Aとして
AST-002を実行する。

以下を確認する。

- User Aに属する資産口座の設定のみ使用されること
- User Bの設定によって`availableAssets`が変化しないこと

---

#### 26.29 利用者コンテキスト未指定

`X-User-Id`を
指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

---

#### 26.30 利用者ID形式不正

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

#### 26.31 利用者不存在

存在しない利用者IDを
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 26.32 論理削除済み利用者

論理削除済み利用者を
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 26.33 AST-001との取得結果比較

AST-001の
最新確定済み対象年月が
`2026-06`である状態とする。

AST-001を実行し、
取得した資産状況を保持する。

続いて、

```http
GET /api/v1/asset-summaries/2026-06
```

を実行する。

同一時点のデータ状態であれば、
以下が一致することを確認する。

- `targetYearMonth`
- `totalAssets`
- `availableAssets`
- 資産口座別資産状況
- 保有商品別資産状況

---

#### 26.34 副作用

AST-002実行前後で、
以下のテーブルが
変更されていないことを確認する。

- `users`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `holding_assets`
- `asset_account_available_settings`

以下も確認する。

- 月末資産状況の`confirmed`が変更されないこと
- 集計結果をデータベースへ保存しないこと
- 利用可能資産設定が変更されないこと

---

#### 26.35 冪等性

同一のデータ状態で
同じリクエストを
複数回実行する。

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

以下を確認する。

- データベースが変更されないこと
- 同一状態であれば同じ`targetYearMonth`となること
- 同一状態であれば同じ集計結果となること

---

#### 26.36 レスポンス契約

正常取得時に、
API共通方針で定めた
Envelope形式で返却されること。

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
      }
    ]
  }
}
```

以下を確認する。

- `data`がobjectであること
- `targetYearMonth`がリクエスト指定値と一致すること
- `targetYearMonth`が`YYYY-MM`形式であること
- `totalAssets`がintegerであること
- `availableAssets`がintegerであること
- `assetAccounts`がarrayであること
- `assetAccountId`がstringであること
- `available`がbooleanであること
- `holdingAssets`がarrayであること
- `holdingAssetId`がstringであること
- JSONフィールド名がcamelCaseであること

---

#### 26.37 返却しない情報

本APIでは、
以下の情報が
レスポンスへ含まれていないことを確認する。

- `user_id`
- `month_end_asset_snapshot_id`
- `confirmed`
- `month_end_asset_balances.id`
- `month_end_holding_values.id`
- `balance_recording_unit`
- `asset_account_available_settings`
- `created_at`
- `updated_at`

---

#### 26.38 エラーレスポンス

各異常系について、
API共通方針で定めた
共通エラーレスポンス形式で
返却されることを確認する。

以下を確認する。

- `error.code`が期待するエラーコードであること
- `error.message`が設定されていること
- `error.details`が配列であること
- `requestId`が設定されていること
- SQLが含まれていないこと
- PostgreSQLの制約名が含まれていないこと
- スタックトレースが含まれていないこと
- 内部例外メッセージが含まれていないこと

---

### 27 Laravel実装方針

AST-002では、
Action、
Request、
UseCase、
Query、
集計用DTO、
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
    ├─ MonthEndAssetSnapshotQuery
    ├─ AssetSummaryQuery
    └─ AssetAvailabilityQuery
    ↓
AssetSummary DTO
    ↓
API Resource
    ↓
Responder
```

AST-002の業務フロー制御は、UseCaseへ集約する。

Actionへ、指定年月の月末資産状況検索、資産集計、利用可能資産判定、利用者境界確認などの業務ロジックを直接記述しない。

---

#### 27.1 Action

HTTPリクエストを受け付け、検証済みの`targetYearMonth`および利用者コンテキストを取得する。

指定年月資産状況取得UseCaseを呼び出し、取得結果をResponderへ渡す。

概念例：

```php
final class GetAssetSummaryByMonthAction
{
    public function __invoke(
        GetAssetSummaryByMonthRequest $request,
        GetAssetSummaryByMonthUseCase $useCase,
        AssetSummaryByMonthResponder $responder,
        userContext $userContext,
    ): JsonResponse {
        $summary = $useCase->execute(
            userId: $userContext->userId,
            targetYearMonth:
                $request->validated(
                    'targetYearMonth',
                ),
        );

        return $responder->ok(
            $summary,
        );
    }
}
```

Actionでは、以下を行わない。

* `targetYearMonth`の形式検証
* 指定年月の月末資産状況検索
* 月末資産残高の取得
* 商品別月末評価額の取得
* 残高記録単位の判定
* 総資産の集計
* 利用可能資産の集計
* 利用可能資産設定の判定
* 資産口座別資産状況の組み立て
* 保有商品別資産状況の組み立て
* 利用者境界の判定
* APIレスポンス形式への変換

---

#### 27.2 Request

パスパラメータの`targetYearMonth`を検証する。

本APIでは、クエリパラメータおよびリクエストボディを使用しない。

`targetYearMonth`は、`prepareForValidation()`でバリデーション対象へ追加する。

概念例：

```php
final class GetAssetSummaryByMonthRequest
    extends FormRequest
{
    protected function prepareForValidation(): void
    {
        $this->merge([
            'targetYearMonth'
                => $this->route(
                    'targetYearMonth',
                ),
        ]);
    }

    public function rules(): array
    {
        return [
            'targetYearMonth' => [
                'required',
                'regex:/^\d{4}-(0[1-9]|1[0-2])$/',
            ],
        ];
    }
}
```

年月形式の検証を複数APIで使用する場合は、独自Ruleへ切り出してよい。

概念例：

```php
'targetYearMonth' => [
    'required',
    new YearMonthRule(),
],
```

---

#### 27.3 Requestで行わないこと

Requestでは、以下の業務判定を行わない。

* 指定年月の月末資産状況が存在するか
* 指定年月の月末資産状況が確定済みか
* 指定年月の月末資産状況が操作対象利用者に属するか
* 月末資産残高が存在するか
* 商品別月末評価額が存在するか
* 利用可能資産設定が存在するか
* 資産状況を正常に算出できるか

これらは、入力値の形式検証ではなく、データベース状態に依存する業務ルールとしてUseCaseおよびQueryで扱う。

---

#### 27.4 UseCase

指定年月資産状況取得のユースケース処理を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `targetYearMonth`を受け取る
3. 指定年月の確定済み月末資産状況を取得する
 対象が存在しない場合は業務例外を送出する
5. 対象snapshotに紐づく資産データを取得する
6. 指定年月時点の利用可能資産設定を取得する
7. 資産口座ごとの資産額を算出する
8. 商品単位の資産口座について保有商品別資産状況を構成する
9. 総資産を算出する
10. 利用可能資産を算出する
11. `AssetSummary` DTOを生成して返却する

概念的な処理は、以下とする。

```text
操作対象利用者
+
targetYearMonth
    ↓
指定年月の
確定済み月末資産状況取得
    ↓
存在しない
    → CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND

存在する
    ↓
snapshotId確定
    ↓
資産データ取得
    ↓
targetYearMonth時点の
利用可能資産設定取得
    ↓
資産口座別集計
    ↓
総資産集計
    ↓
利用可能資産集計
    ↓
AssetSummary生成
```

UseCaseでは、SQLやEloquent Query Builderを直接組み立てない。

データベースアクセスは、Queryへ委譲する。

---

#### 27.5 MonthEndAssetSnapshotQuery

操作対象利用者について、指定年月と一致する確定済み月末資産状況を取得する。

検索条件には、必ず以下を含める。

```text
user_id
target_year_month
confirmed
```

概念例：

```php
final class MonthEndAssetSnapshotQuery
{
    public function findConfirmedForUserByTargetYearMonth(
        int $userId,
        string $targetYearMonth,
    ): ?MonthEndAssetSnapshot {
        return MonthEndAssetSnapshot::query()
            ->where(
                'user_id',
                $userId,
            )
            ->where(
                'target_year_month',
                $targetYearMonth,
            )
            ->where(
                'confirmed',
                true,
            )
            ->first([
                'id',
                'target_year_month',
            ]);
    }
}
```

`targetYearMonth`だけを条件として月末資産状況を取得しない。

---

#### 27.6 指定年月の確定済み月末資産状況不存在

MonthEndAssetSnapshotQueryの取得結果が`null`の場合は、

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

へ変換するための業務例外を送出する。

概念例：

```php
$snapshot =
    $this->monthEndAssetSnapshotQuery
        ->findConfirmedForUserByTargetYearMonth(
            userId: $userId,
            targetYearMonth:
                $targetYearMonth,
        );

if ($snapshot === null) {
    throw new
        ConfirmedAssetSnapshotNotFoundException();
}
```

以下の場合は、クライアントへ区別して公開しない。

* 指定年月の月末資産状況が存在しない
* 指定年月の月末資産状況が未確定
* 他利用者にのみ同年月の確定済み月末資産状況が存在する

すべて、

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

として扱う。

---

#### 27.7 別年月へフォールバックしない

AST-002では、指定した`targetYearMonth`について確定済み月末資産状況が存在しない場合でも、前月または翌月を代替利用しない。

以下のような処理は行わない。

```text
2026-06
確定済みデータなし
    ↓
2026-05を取得
```

または、

```text
2026-06
確定済みデータなし
    ↓
2026-07を取得
```

AST-002は、指定年月の資産状況取得に責務を限定する。

---

#### 27.8 AssetSummaryQuery

対象snapshotに紐づく資産状況算出用データを取得する。

概念例：

```php
$assetData =
    $this->assetSummaryQuery
        ->findBySnapshot(
            userId: $userId,
            snapshotId: $snapshot->id,
        );
```

Queryでは、主に以下のデータを取得する。

```text
asset_accounts
month_end_asset_balances
holding_assets
month_end_holding_values
```

資産口座の`balance_recording_unit`に応じて、資産額算出に必要となる情報を取得する。

---

#### 27.9 AST-001との共通化

AST-001とAST-002の主な違いは、対象となる確定済み月末資産状況の特定方法である。

```text
AST-001
    ↓
最新の確定済みsnapshot

AST-002
    ↓
指定年月の確定済みsnapshot
```

snapshotが特定された後の以下の処理については、AST-001と共通化する。

* 資産データ取得
* 残高記録単位による算出元切り替え
* 資産口座別集計
* 保有商品別集計
* 利用可能資産判定
* 総資産集計
* 利用可能資産集計
* DTO生成

同じ資産集計ロジックをAST-001とAST-002へ重複実装しない。

---

#### 27.10 利用者境界

資産情報取得時は、必ず操作対象利用者との利用者境界を保証する。

月末資産残高については、

```text
month_end_asset_balances
    ↓
asset_accounts
    ↓
asset_accounts.user_id
        = 操作対象利用者ID
```

によって保証する。

商品別月末評価額については、

```text
month_end_holding_values
    ↓
holding_assets
    ↓
asset_accounts
    ↓
asset_accounts.user_id
        = 操作対象利用者ID
```

によって保証する。

`snapshotId`だけを条件として資産情報を取得しない。

---

#### 27.11 口座単位の資産額

口座単位で残高を記録する資産口座では、

```text
month_end_asset_balances.balance
```

を資産額として使用する。

概念的な取得条件は、以下とする。

```text
month_end_asset_snapshot_id
    = snapshotId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.balance_recording_unit
    = 口座単位
```

商品別月末評価額が何らかの理由で存在していても、資産額へ加算しない。

---

#### 27.12 商品単位の資産額

商品単位で評価額を記録する資産口座では、

```text
month_end_holding_values.value
```

を資産額として使用する。

保有商品ごとの`value`を取得し、同一資産口座に属する保有商品の評価額を合計して資産口座の資産額とする。

```text
assetAccount.assetAmount
=
SUM(holdingAssets.assetAmount)
```

概念的な取得条件は、以下とする。

```text
month_end_asset_snapshot_id
    = snapshotId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.balance_recording_unit
    = 商品単位
```

---

#### 27.13 二重計上の防止

口座単位と商品単位の資産額を明確に分離して扱う。

以下のような単純合計は行わない。

```text
SUM(month_end_asset_balances.balance)
+
SUM(month_end_holding_values.value)
```

残高記録単位に応じて、

```text
口座単位
    → balanceのみ

商品単位
    → valueのみ
```

を使用する。

これにより、同一資産の二重計上を防止する。

---

#### 27.14 AssetAvailabilityQuery

指定年月時点の利用可能資産設定を取得する。

概念例：

```php
$availabilityMap =
    $this->assetAvailabilityQuery
        ->findForTargetYearMonth(
            userId: $userId,
            targetYearMonth:
                $targetYearMonth,
        );
```

現在日時点の設定ではなく、リクエストで指定された`targetYearMonth`を基準とする。

---

#### 27.15 利用可能資産設定の利用者境界

利用可能資産設定についても、操作対象利用者に属する資産口座の設定のみを取得する。

概念的には、

```text
asset_account_available_settings
    ↓
asset_accounts
    ↓
asset_accounts.user_id
        = 操作対象利用者ID
```

によって利用者境界を保証する。

他の利用者に属する利用可能資産設定を使用してはならない。

---

#### 27.16 指定年月時点の設定判定

利用可能資産設定は、`targetYearMonth`時点で有効な設定を使用する。

現在の最新設定だけを使用しない。

概念的には、

```text
targetYearMonth
    ↓
指定年月時点で有効な
asset_account_available_settings
    ↓
available判定
```

とする。

これにより、過去月を指定した場合でも、その年月時点の利用可能資産状態を再現する。

具体的な有効期間判定は、`asset_account_available_settings`のテーブル定義および業務ルールに従う。

---

#### 27.17 AssetSummary DTO

指定年月資産状況は、Eloquent ModelをそのままResourceへ渡すのではなく、表示用DTOとして構成する。

概念例：

```php
final readonly class AssetSummary
{
    public function __construct(
        public string $targetYearMonth,
        public int $totalAssets,
        public int $availableAssets,
        /** @var AssetAccountSummary[] */
        public array $assetAccounts,
    ) {
    }
}
```

資産口座単位は、以下のようなDTOとする。

```php
final readonly class AssetAccountSummary
{
    public function __construct(
        public int $assetAccountId,
        public string $name,
        public int $assetAmount,
        public bool $available,
        /** @var HoldingAssetSummary[] */
        public array $holdingAssets,
    ) {
    }
}
```

保有商品単位は、以下のようなDTOとする。

```php
final readonly class HoldingAssetSummary
{
    public function __construct(
        public int $holdingAssetId,
        public string $name,
        public int $assetAmount,
    ) {
    }
}
```

AST-001と同じレスポンス構造であるため、同じDTOを共通利用してよい。

---

#### 27.18 資産口座別資産状況の生成

UseCaseでは、Queryで取得したデータをもとに`AssetAccountSummary`を生成する。

口座単位の場合は、

```text
assetAmount
    = month_end_asset_balances.balance

holdingAssets
    = []
```

とする。

商品単位の場合は、

```text
holdingAssets
    = 商品別月末評価額一覧

assetAmount
    = SUM(holdingAssets.assetAmount)
```

とする。

---

#### 27.19 総資産の集計

総資産は、生成した各資産口座の`assetAmount`を合計する。

概念例：

```php
$totalAssets =
    array_sum(
        array_map(
            static fn (
                AssetAccountSummary $account,
            ): int =>
                $account->assetAmount,
            $assetAccounts,
        ),
    );
```

正常なデータ状態では、

```text
totalAssets
=
SUM(assetAccounts.assetAmount)
```

となることを保証する。

---

#### 27.20 利用可能資産の集計

利用可能資産は、指定年月時点で

```text
available = true
```

となる資産口座だけを合計する。

概念例：

```php
$availableAssets =
    array_sum(
        array_map(
            static fn (
                AssetAccountSummary $account,
            ): int =>
                $account->available
                    ? $account->assetAmount
                    : 0,
            $assetAccounts,
        ),
    );
```

利用可能資産の計算のために資産額を再取得しない。

資産口座別集計結果をそのまま利用する。

---

#### 27.21 0円の扱い

以下は、正常な資産額として扱う。

```text
balance = 0
value = 0
```

以下のようなtruthy / falsy判定を使用しない。

```php
if (! $balance) {
    // 0円まで不存在扱いになるため使用しない
}
```

0円の資産口座や0円の保有商品も、対象データが存在する限りレスポンスへ含める。

---

#### 27.22 QueryとUseCaseの責務分離

Queryは、データベースから必要なデータを取得することに責務を限定する。

UseCaseは、複数Queryの結果を組み合わせて指定年月資産状況を構成する。

概念的には、

```text
Query
    → データ取得

UseCase
    → 業務フロー制御
    → 資産状況構成
    → 集計

Resource
    → API形式へ変換
```

とする。

QueryでHTTPレスポンスを生成しない。

UseCaseでSQLやEloquent Query Builderを直接組み立てない。

---

#### 27.23 API Resource

`AssetSummary` DTOを、API ResourceによってAPIレスポンス形式へ変換する。

概念例：

```php
final class AssetSummaryByMonthResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'targetYearMonth'
                => $this->targetYearMonth,

            'totalAssets'
                => $this->totalAssets,

            'availableAssets'
                => $this->availableAssets,

            'assetAccounts'
                => AssetAccountSummaryResource::collection(
                    $this->assetAccounts,
                ),
        ];
    }
}
```

JSONフィールド名は、API共通方針に従ってcamelCaseとする。

AST-001とレスポンス項目が同一である場合は、共通の`AssetSummaryResource`を使用してよい。

---

#### 27.24 AssetAccountSummaryResource

資産口座別資産状況は、専用Resourceへ変換する。

概念例：

```php
final class AssetAccountSummaryResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'assetAccountId'
                => (string)
                    $this->assetAccountId,

            'name'
                => $this->name,

            'assetAmount'
                => $this->assetAmount,

            'available'
                => $this->available,

            'holdingAssets'
                => HoldingAssetSummaryResource::collection(
                    $this->holdingAssets,
                ),
        ];
    }
}
```

`assetAccountId`は、API共通方針に従ってstringとして返却する。

---

#### 27.25 HoldingAssetSummaryResource

保有商品別資産状況についても、専用Resourceへ変換する。

概念例：

```php
final class HoldingAssetSummaryResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'holdingAssetId'
                => (string)
                    $this->holdingAssetId,

            'name'
                => $this->name,

            'assetAmount'
                => $this->assetAmount,
        ];
    }
}
```

`holdingAssetId`は、API共通方針に従ってstringとして返却する。

---

#### 27.26 返却しない情報

API Resourceでは、指定年月資産状況の表示に必要な情報だけを返却する。

以下の情報は返却しない。

* `user_id`
* `month_end_asset_snapshot_id`
* `confirmed`
* `month_end_asset_balances.id`
* `month_end_holding_values.id`
* `balance_recording_unit`
* `asset_account_available_settings`
* `created_at`
* `updated_at`

Eloquent ModelをそのままJSON化してはならない。

---

#### 27.27 Responder

Responderは、生成済みの`AssetSummary` DTOを受け取り、API共通方針に従ったHTTPレスポンスへ変換する。

概念例：

```php
final class AssetSummaryByMonthResponder
{
    public function ok(
        AssetSummary $summary,
    ): JsonResponse {
        return response()->json(
            [
                'data' =>
                    new AssetSummaryByMonthResource(
                        $summary,
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

`requestId`などの共通Envelope項目は、API共通レスポンス処理に従う。

---

#### 27.28 Responderの責務

Responderでは、以下を行わない。

* データベース検索
* `targetYearMonth`の形式検証
* 指定年月のsnapshot検索
* 残高記録単位の判定
* 資産口座別集計
* 保有商品別集計
* 総資産集計
* 利用可能資産集計
* 利用可能資産設定の判定
* 利用者境界の判定

Responderは、取得済みのAssetSummaryをHTTPレスポンスへ変換することに責務を限定する。

---

#### 27.29 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* JSONレスポンス共通処理
* 共通例外処理
* ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

#### 27.30 Repository

AST-002では、Repositoryを使用しない。

本APIは読み取り専用APIであり、以下を行わないためである。

* 登録
* 更新
* 削除

参照処理は、Queryクラスへ集約する。

---

#### 27.31 トランザクション

AST-002は、読み取り専用APIであるため、Phase1では明示的な`DB::transaction()`を使用しない。

以下のような実装は行わない。

```php
DB::transaction(
    function () {
        // 指定年月資産状況取得のみ
    },
);
```

複数SELECT間で厳密な読み取り一貫性が必要となる要件が将来的に追加された場合は、トランザクション分離レベルを含めて別途検討する。

---

#### 27.32 ロック

AST-002では、行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

指定年月資産状況の取得によって、他の月末資産関連処理を不要にブロックしない。

---

#### 27.33 N+1問題

資産口座ごと、保有商品ごとに個別SQLを発行する実装は避ける。

以下のような取得方法は採用しない。

```text
資産口座一覧取得
    ↓
各資産口座ごとに
月末資産残高取得
    ↓
各資産口座ごとに
保有商品取得
    ↓
各保有商品ごとに
商品別月末評価額取得
```

専用Queryによって、以下を必要な単位でまとめて取得する。

* 口座単位資産データ
* 商品単位資産データ
* 利用可能資産設定

その後、UseCaseでMap化・グルーピングしてDTOを生成する。

---

#### 27.34 取得カラム

Queryでは、指定年月資産状況の構築に必要なカラムを中心に取得する。

例えば、資産口座では、

```text
id
name
balance_recording_unit
```

月末資産残高では、

```text
asset_account_id
balance
```

保有商品では、

```text
id
asset_account_id
name
```

商品別月末評価額では、

```text
holding_asset_id
value
```

を使用する。

レスポンス生成や業務判定に不要なカラムを過剰に取得しない。

---

#### 27.35 キャッシュ

Phase1では、AST-002専用のサーバー側アプリケーションキャッシュを使用しない。

指定年月の資産状況は、確定済み月末資産データおよび対象年月時点の利用可能資産設定から算出する。

Phase1では、キャッシュ管理を追加せず、実装を単純に保つ。

性能上の必要性が生じた場合は、将来的に対象年月単位のキャッシュを検討する。

---

#### 27.36 例外変換

Laravel内部例外をそのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態                  | 独自エラーコード                             |
| --------------------- | ------------------------------------ |
| 利用者未指定                | `USER_CONTEXT_REQUIRED`         |
| 利用者ID形式不正             | `INVALID_USER_ID`               |
| 利用者不存在                | `USER_NOT_FOUND`                |
| `targetYearMonth`形式不正 | `VALIDATION_ERROR`                   |
| 指定年月の確定済み月末資産状況不存在    | `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` |
| 想定外例外                 | `INTERNAL_SERVER_ERROR`              |

指定年月の月末資産状況が未確定の場合も、

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

として扱う。

他利用者にのみ指定年月の確定済みデータが存在する場合も、同じエラーとして扱う。

---

#### 27.37 想定外例外

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

#### 27.38 ログ

AST-002では、必要に応じて以下の情報をログコンテキストへ設定する。

```text
requestId
userId
targetYearMonth
snapshotId
```

以下の具体的な金額情報は、不要にアクセスログへ出力しない。

* 総資産
* 利用可能資産
* 資産口座別残高
* 商品別月末評価額

---

#### 27.39 テスト実装方針

Laravel側では、Feature Testを中心としてAST-002のAPI契約および指定年月資産状況取得処理を確認する。

Feature Testでは、主に以下を確認する。

* `200 OK`
* `400 Bad Request`
* `404 Not Found`
* `422 Unprocessable Entity`
* `500 Internal Server Error`
* `targetYearMonth`の形式検証
* 指定した対象年月がそのまま使用されること
* 最新年月へ置き換えないこと
* 指定年月が未確定の場合に取得できないこと
* 指定年月が存在しない場合に取得できないこと
* 別年月へフォールバックしないこと
* 口座単位資産の集計
* 商品単位資産の集計
* 月末資産残高と商品別月末評価額の二重計上防止
* 複数資産口座の総資産集計
* `totalAssets`と資産口座別合計の一致
* 指定年月時点の利用可能資産設定を使用すること
* 過去月の利用可能資産状態を正しく再現すること
* 利用可能資産が0円の場合
* `balance = 0`の扱い
* `value = 0`の扱い
* 口座単位では`holdingAssets = []`となること
* 商品単位では保有商品別資産状況が返却されること
* 他利用者の同一年月snapshotを使用しないこと
* 他利用者の月末資産残高が混入しないこと
* 他利用者の商品別月末評価額が混入しないこと
* 他利用者の利用可能資産設定を使用しないこと
* AST-001と同一対象年月なら同じ集計結果となること
* IDがstringとして返却されること
* JSONフィールド名がcamelCaseであること
* 返却対象外項目が含まれないこと
* データベースが更新されないこと
* 冪等性

Requestについては、以下を確認する。

```text
2026-01
2026-12
    → 正常
```

```text
2026-1
2026/01
202601
2026-00
2026-13
abc
    → VALIDATION_ERROR
```

MonthEndAssetSnapshotQueryについては、Database Testで以下を確認する。

```text
user_id一致
+
target_year_month一致
+
confirmed = true
    ↓
取得できる
```

```text
target_year_month一致
+
confirmed = false
    ↓
null
```

```text
他利用者
+
target_year_month一致
+
confirmed = true
    ↓
null
```

AssetSummaryQueryについては、以下を確認する。

```text
口座単位
    → balanceのみ使用

商品単位
    → valueのみ使用

他利用者データ
    → 取得対象外
```

AssetAvailabilityQueryについては、以下を確認する。

```text
操作対象利用者
+
targetYearMonth
    ↓
指定年月時点の設定を取得
```

UseCaseについては、Queryの結果から期待するAssetSummaryが生成されることを確認する。

概念的には、

```text
snapshot
+
口座単位資産
+
商品単位資産
+
指定年月時点の利用可能資産設定
    ↓
GetAssetSummaryByMonthUseCase
    ↓
AssetSummary
```

をUnit Testする。

特に、

```text
totalAssets
=
SUM(assetAccounts.assetAmount)
```

および、

```text
availableAssets
=
SUM(
    available = true
    のassetAccounts.assetAmount
)
```

となることを確認する。

また、商品単位の資産口座について、

```text
assetAccount.assetAmount
=
SUM(holdingAssets.assetAmount)
```

となることを確認する。

---

### 28 React・TypeScriptでの利用

本APIでは、
パスパラメータとして
`targetYearMonth`を使用する。

パスパラメータの型は、
以下とする。

```ts
export type GetAssetSummaryByMonthParams = {
  targetYearMonth: string;
};
```

資産状況のレスポンス型は、
AST-001 現在資産状況取得APIと
共通化してよい。

```ts
export type HoldingAssetSummary = {
  holdingAssetId: string;
  name: string;
  assetAmount: number;
};

export type AssetAccountSummary = {
  assetAccountId: string;
  name: string;
  assetAmount: number;
  available: boolean;
  holdingAssets: HoldingAssetSummary[];
};

export type AssetSummary = {
  targetYearMonth: string;
  totalAssets: number;
  availableAssets: number;
  assetAccounts: AssetAccountSummary[];
};

export type GetAssetSummaryByMonthResponse = {
  data: AssetSummary;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetAssetSummaryByMonthResponse>(
    `/api/v1/asset-summaries/${targetYearMonth}`,
  );
```

資産推移画面や
過去資産状況画面などから、
特定年月の資産状況を表示する際に利用する。

---

#### 28.1 targetYearMonthの扱い

`targetYearMonth`は、
`YYYY-MM`形式のstringとして扱う。

例：

```ts
const targetYearMonth =
  '2026-05';
```

フロントエンド側でも、
基本的には
stringのまま扱う。

```ts
type TargetYearMonth = string;
```

日付計算が必要な場合を除き、
`Date`へ変換して
APIへ再変換する必要はない。

---

#### 28.2 URL生成

`targetYearMonth`は、
パスパラメータとして使用する。

```ts
const url =
  `/api/v1/asset-summaries/${targetYearMonth}`;
```

例えば、

```ts
targetYearMonth = '2026-05';
```

の場合、

```http
GET /api/v1/asset-summaries/2026-05
```

となる。

---

#### 28.3 targetYearMonthの入力UI

利用者が
対象年月を選択できるUIを
提供する場合は、
年月単位で入力できる
コントロールを使用する。

概念例：

```tsx
<input
  type="month"
  value={targetYearMonth}
  onChange={(event) =>
    setTargetYearMonth(
      event.target.value,
    )
  }
/>
```

ブラウザから取得した値が、

```text
YYYY-MM
```

形式であることを前提とする。

ただし、
API側でも必ず
形式検証を行う。

---

#### 28.4 Queryとして扱う

AST-002は、
読み取り専用のGET APIであるため、
React Query等では
Queryとして扱う。

概念例：

```ts
export const fetchAssetSummaryByMonth =
  async (
    targetYearMonth: string,
  ) => {
    const response =
      await apiClient.get<GetAssetSummaryByMonthResponse>(
        `/api/v1/asset-summaries/${targetYearMonth}`,
      );

    return response.data;
  };
```

---

#### 28.5 Query Key

Query Keyには、
`targetYearMonth`を含める。

概念例：

```ts
export const assetSummaryKeys = {
  all: [
    'assetSummaries',
  ] as const,

  current: [
    'assetSummaries',
    'current',
  ] as const,

  byMonth: (
    targetYearMonth: string,
  ) =>
    [
      'assetSummaries',
      'byMonth',
      targetYearMonth,
    ] as const,
};
```

これにより、
対象年月ごとに
キャッシュを分離できる。

---

#### 28.6 Hook実装例

React Queryを使用する場合の
概念例は、
以下とする。

```ts
export const useAssetSummaryByMonth =
  (
    targetYearMonth: string,
  ) => {
    return useQuery({
      queryKey:
        assetSummaryKeys.byMonth(
          targetYearMonth,
        ),

      queryFn:
        () =>
          fetchAssetSummaryByMonth(
            targetYearMonth,
          ),

      enabled:
        targetYearMonth.length > 0,
    });
  };
```

実際のAPI Client、
React Queryおよび
キャッシュ方針は、
フロントエンド共通設計に従う。

---

#### 28.7 レスポンスのtargetYearMonth

レスポンスの
`targetYearMonth`には、
実際に取得対象となった
確定済み対象年月が返却される。

AST-002では、
正常時は
リクエストで指定した値と
一致する。

```ts
const requestedMonth =
  targetYearMonth;

const responseMonth =
  data.targetYearMonth;
```

正常時は、

```text
requestedMonth
=
responseMonth
```

となる。

---

#### 28.8 totalAssetsの扱い

`totalAssets`は、
指定年月における
総資産額として扱う。

```ts
const totalAssets =
  data.totalAssets;
```

フロントエンド側で
月末資産残高や
商品別月末評価額を
再取得して
再計算しない。

---

#### 28.9 availableAssetsの扱い

`availableAssets`は、
指定年月時点の
利用可能資産額として扱う。

```ts
const availableAssets =
  data.availableAssets;
```

現在の利用可能資産設定から
再計算しない。

AST-002から返却された値を
指定年月時点の正式値として
表示する。

---

#### 28.10 assetAccountsの扱い

`assetAccounts`は、
指定年月における
資産口座別資産状況として扱う。

概念例：

```tsx
{data.assetAccounts.map(
  (account) => (
    <AssetAccountRow
      key={account.assetAccountId}
      account={account}
    />
  ),
)}
```

各資産口座について、
以下を表示できる。

- 資産口座名
- 資産額
- 指定年月時点の利用可能状態
- 保有商品別資産状況

---

#### 28.11 holdingAssetsの扱い

商品単位で管理する
資産口座について、
`holdingAssets`から
保有商品別資産状況を表示する。

概念例：

```tsx
{account.holdingAssets.map(
  (holding) => (
    <HoldingAssetRow
      key={holding.holdingAssetId}
      holdingAsset={holding}
    />
  ),
)}
```

口座単位の資産口座では、

```text
holdingAssets = []
```

となる。

空配列を
APIエラーとして扱わない。

---

#### 28.12 IDの扱い

以下のIDは、
API共通方針に従って
stringとして扱う。

```text
assetAccountId
holdingAssetId
```

例えば、

```ts
const assetAccountId: string =
  account.assetAccountId;

const holdingAssetId: string =
  holding.holdingAssetId;
```

numberへ変換して
業務計算には使用しない。

---

#### 28.13 金額表示

金額は、
日本円のintegerとして
APIから返却される。

画面表示時は、
表示用フォーマットを適用する。

概念例：

```ts
export const formatYen =
  (amount: number): string =>
    new Intl.NumberFormat(
      'ja-JP',
      {
        style: 'currency',
        currency: 'JPY',
        maximumFractionDigits: 0,
      },
    ).format(amount);
```

例えば、

```tsx
<span>
  {formatYen(data.totalAssets)}
</span>
```

のように表示する。

---

#### 28.14 0円の扱い

以下は、
正常な業務値として扱う。

```text
totalAssets = 0
availableAssets = 0
assetAmount = 0
```

truthy / falsyによって
未取得扱いしない。

以下のような判定は避ける。

```ts
if (!data.totalAssets) {
  // 0円まで未取得扱いになるため使用しない
}
```

---

#### 28.15 ローディング表示

AST-002取得中は、
ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return <Loading />;
}
```

未取得状態と
0円の資産状況を
明確に区別する。

---

#### 28.16 targetYearMonthの変更

利用者が
対象年月を変更した場合は、
新しい`targetYearMonth`を
Query Keyへ反映し、
AST-002を再取得する。

概念例：

```ts
setTargetYearMonth(
  '2026-06',
);
```

これにより、

```text
2026-05
    ↓
2026-06
```

のように
表示対象を切り替える。

---

#### 28.17 CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND

`CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND`
が返却された場合は、
指定年月について
表示可能な確定済み資産状況が
存在しないことを表示する。

表示例：

```text
指定した年月には、
確定済みの資産状況がありません。
```

この場合、
別年月の資産状況を
フロントエンド側で
自動表示しない。

---

#### 28.18 未確定月の扱い

指定年月に
月末資産状況が存在していても、
未確定の場合は
AST-002では取得できない。

フロントエンドでは、
`CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

以下のように、
未確定データ取得用APIへ
自動的に切り替えない。

```text
AST-002失敗
    ↓
未確定データを別APIで取得
    ↓
資産状況として代替表示
```

このような処理は行わない。

---

#### 28.19 VALIDATION_ERROR

`targetYearMonth`が
不正な形式の場合は、

`VALIDATION_ERROR`

として扱う。

例えば、

```text
2026-13
```

などが該当する。

通常の年月選択UIでは
発生しないことを前提とするが、
URL直接入力などに備えて
エラー処理を行う。

---

#### 28.20 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 不正な対象年月またはURLとして扱う |
| `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` | 指定年月に確定済み資産状況がないことを表示する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

#### 28.21 自動リトライ

AST-002は、
読み取り専用GET APIであり、
冪等である。

そのため、
ネットワークエラーや
一時的な5xxエラーに対して、
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
再送しても解消しないエラーは
不要にリトライしない。

---

#### 28.22 クライアントキャッシュ

AST-002は、
React Query等による
クライアントキャッシュの
対象としてよい。

対象年月ごとに
Query Keyを分ける。

```text
2026-05
    → 独立したキャッシュ

2026-06
    → 独立したキャッシュ
```

利用者切り替え時は、
操作対象利用者に応じて
キャッシュを再取得または
無効化する。

---

#### 28.23 AST-001との使い分け

AST-001は、
最新の確定済み対象年月を
サーバー側で決定する。

AST-002は、
利用者が指定した年月を取得する。

```text
最新の資産状況
    → AST-001

2026-05の資産状況
    → AST-002
```

フロントエンド側で
AST-002へ最新年月を毎回指定して
AST-001の代替としない。

---

#### 28.24 AST-003との連携

AST-003 資産推移取得APIで
時系列グラフを表示し、
特定年月を選択した場合に、
AST-002を使用して
その年月の詳細を取得できる。

概念例：

```text
AST-003
2026-01〜2026-06の推移表示
    ↓
2026-05を選択
    ↓
AST-002
2026-05の資産状況詳細
```

---

#### 28.25 グラフからの遷移例

概念的には、
以下のように
対象年月をURLへ含めてもよい。

```tsx
navigate(
  `/assets/2026-05`,
);
```

詳細画面では、
ルートパラメータから
`targetYearMonth`を取得し、
AST-002を呼び出す。

---

#### 28.26 フロントエンドで再集計しない

APIレスポンスとして返却された

- `totalAssets`
- `availableAssets`
- `assetAccounts[].assetAmount`

を正式値として扱う。

フロントエンド側で
月末資産データを再取得し、
独自集計して
API結果を置き換えない。

---

#### 28.27 現在の設定で過去月を書き換えない

AST-002の
`available`および
`availableAssets`は、
指定年月時点の
利用可能資産設定を反映している。

フロントエンド側で
現在の設定を取得して、

```ts
account.available =
  currentSetting.available;
```

のように
書き換えてはならない。

---

### 29 設計上の補足

#### 29.1 targetYearMonthをURLに含める理由

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

#### 29.2 クエリパラメータではなくパスにする理由

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

#### 29.3 AST-001とAST-002を分ける理由

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

#### 29.4 未確定月を取得しない理由

未確定の月末資産状況は、
入力途中または
修正途中である可能性がある。

そのため、
正式な資産状況表示には使用しない。

AST-002でも、
AST-001と同様に
確定済みデータのみを対象とする。

---

#### 29.5 別年月へフォールバックしない理由

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

#### 29.6 AST-001と集計ロジックを共通化する理由

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

#### 29.7 指定年月時点の利用可能資産設定を使用する理由

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

#### 29.8 集計結果を保存しない理由

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

#### 29.9 balance_recording_unitを返却しない理由

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

#### 29.10 holdingAssetsを空配列で返す理由

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

#### 29.11 GETを採用する理由

AST-002は、
指定年月の資産状況を
参照するだけのAPIである。

データの登録、
更新、
削除を行わないため、
HTTPメソッドには
`GET`を採用する。

---

#### 29.12 AST-002が冪等である理由

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

#### 29.13 Idempotency-Keyを使用しない理由

AST-002は、
読み取り専用のGET APIである。

再実行によって
データの重複登録などが
発生しないため、
`Idempotency-Key`は使用しない。

---

#### 29.14 サーバーキャッシュを採用しない理由

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

### 30 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)
