## AST-001 現在資産状況取得

### 1 概要

操作対象となる利用者について、
最新の確定済み対象年月における
現在の資産状況を取得する。

本APIでは、
操作対象利用者に属する
`month_end_asset_snapshots`のうち、
`confirmed = true`である
最新の対象年月を特定する。

特定した対象年月について、
月末資産残高、
商品別月末評価額および
利用可能資産設定をもとに、
現在の資産状況を算出する。

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
対象年月時点の
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
現在の資産状況を
確認する。

例えば、
以下のような場合に使用する。

- ダッシュボードで現在の総資産を確認する
- 現在の利用可能資産を確認する
- 資産口座ごとの資産額を確認する
- 商品単位で管理している資産口座について保有商品ごとの評価額を確認する
- 最新の確定済み対象年月を確認する
- 目的達成判定を行う前に現在の資産状況を確認する

「現在」とは、
リクエスト実行日時点の
リアルタイム残高を意味しない。

操作対象利用者について、
最新の確定済み
月末資産状況を
現在の資産状況として扱う。

例えば、

```text
2026-05
confirmed = true

2026-06
confirmed = true

2026-07
confirmed = false
```

の場合、
AST-001で取得する対象年月は、

```text
2026-06
```

となる。

未確定の`2026-07`は、
現在資産状況の
取得対象としない。

---

### 3 エンドポイント

```http
GET /api/v1/asset-summaries/current
```

---

### 4 HTTPメソッド

```http
GET
```

本APIは、
現在の資産状況を取得する
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
クエリパラメータまたは
パスパラメータでは受け付けない。

利用者IDは、
ミドルウェアで設定された
利用者コンテキストから取得する。

最新の確定済み
月末資産状況を取得する際は、
必ず操作対象利用者によって
絞り込む。

概念的な取得条件は、
以下とする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.confirmed
    = true
```

そのうえで、
`target_year_month`が
最新のレコードを取得する。

概念的には、
以下の順序で対象を特定する。

```text
X-User-Id
    ↓
操作対象利用者
    ↓
操作対象利用者に属する
month_end_asset_snapshots
    ↓
confirmed = true
    ↓
target_year_monthが最新
    ↓
現在資産状況の対象年月
```

他の利用者に属する
`month_end_asset_snapshots`を
最新対象年月の判定に
含めてはならない。

例えば、

```text
User A
2026-06 confirmed = true

User B
2026-07 confirmed = true
```

の状態で、
User Aを操作対象として
AST-001を実行した場合、
User Bの`2026-07`ではなく、
User Aの`2026-06`を
最新の確定済み対象年月として扱う。

月末資産残高についても、
操作対象利用者に属する
月末資産状況および
資産口座に紐づくデータのみを
取得対象とする。

商品別月末評価額についても、
操作対象利用者に属する
月末資産状況、
資産口座および
保有商品に紐づくデータのみを
取得対象とする。

利用可能資産設定についても、
操作対象利用者に属する
資産口座の設定のみを参照する。

利用者境界は、
概念的に以下の関連によって保証する。

```text
users
    ↓
month_end_asset_snapshots
    ↓
month_end_asset_balances
```

および、

```text
users
    ↓
asset_accounts
    ↓
holding_assets
    ↓
month_end_holding_values
```

利用可能資産設定については、

```text
users
    ↓
asset_accounts
    ↓
asset_account_available_settings
```

の関連によって
利用者境界を保証する。

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

本APIでは、
他の利用者に属する
資産情報の存在を
レスポンスから推測できないようにする。

---

### 6 パスパラメータ

なし。

本APIでは、
対象年月をパスパラメータとして
指定しない。

取得対象となる年月は、
操作対象利用者に属する
確定済み月末資産状況のうち、
最新の`target_year_month`を
サーバー側で特定する。

```text
操作対象利用者
    ↓
confirmed = true
    ↓
target_year_monthが最新
```

特定の対象年月を指定して
資産状況を取得する場合は、
AST-002 指定年月資産状況取得APIを使用する。

---

### 7 クエリパラメータ

なし。

本APIでは、
クエリパラメータを使用しない。

以下のような条件は
クライアントから指定しない。

- 対象年月
- 利用者ID
- 資産口座ID
- 保有商品ID
- 残高記録単位
- 確定状態
- 利用可能資産のみを取得するか
- 表示対象となる資産種別

現在資産状況として使用する
対象年月は、
サーバー側で決定する。

---

### 8 リクエストヘッダー

#### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/asset-summaries/current
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

現在資産状況の算出に必要な情報は、
すべてサーバー側で取得する。

クライアントから、
資産額や対象年月などの
業務データを受け取らない。

---

### 10 リクエスト項目

本APIでは、
リクエストボディに
業務項目を持たない。

利用者IDは、
`X-User-Id`から取得する。

現在資産状況の取得に使用する
以下の情報は、
クライアントから受け付けない。

- `userId`
- `targetYearMonth`
- `snapshotId`
- `confirmed`
- `assetAccountId`
- `holdingAssetId`
- `balance`
- `value`
- 利用可能資産設定

これらは、
操作対象利用者および
最新の確定済み月末資産状況をもとに、
サーバー側で取得する。

---

### 11 バリデーション

#### 11.1 パスパラメータ

本APIには、
パスパラメータが存在しないため、
パスパラメータに対する
バリデーションは行わない。

---

#### 11.2 クエリパラメータ

本APIには、
クエリパラメータが存在しないため、
業務上の検索条件に対する
クエリパラメータの
バリデーションは行わない。

現在資産状況の対象年月は、
クライアントから受け取らず、
サーバー側で特定する。

---

#### 11.3 リクエストボディ

本APIには、
リクエストボディが存在しないため、
リクエストボディに対する
業務項目のバリデーションは行わない。

---

#### 11.4 X-User-Id

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

#### 11.5 最新の確定済み月末資産状況

操作対象利用者について、
以下の条件を満たす
月末資産状況を検索する。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.confirmed
    = true
```

複数存在する場合は、
`target_year_month`が
最新のレコードを
現在資産状況の対象とする。

概念的には、
以下の条件で取得する。

```text
user_id = 操作対象利用者ID
AND
confirmed = true
ORDER BY target_year_month DESC
LIMIT 1
```

未確定の月末資産状況は、
対象年月が新しくても
取得対象としない。

---

#### 11.6 確定済み月末資産状況が存在しない場合

操作対象利用者について、
確定済みの
`month_end_asset_snapshots`が
1件も存在しない場合は、
現在資産状況を
算出できない。

```text
操作対象利用者
    ↓
confirmed = true
    ↓
0件
```

この状態は、
正常な空データではなく、
現在資産状況の取得対象が
存在しない業務状態として扱う。

具体的なエラーコードおよび
HTTPステータスは、
本APIの
「エラーレスポンス」で定義する。

---

#### 11.7 未確定の最新年月

操作対象利用者について、
最新の月末資産状況が
未確定であっても、
それ以前に
確定済み月末資産状況が存在する場合は、
エラーとしない。

例えば、

```text
2026-05
confirmed = true

2026-06
confirmed = true

2026-07
confirmed = false
```

の場合、
現在資産状況の対象年月は、

```text
2026-06
```

とする。

`2026-07`が
未確定であることを理由として、
現在資産状況全体を
取得不可とはしない。

---

#### 11.8 月末資産残高の扱い

最新の確定済み
月末資産状況に紐づく
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

#### 11.9 商品別月末評価額の扱い

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

#### 11.10 利用可能資産設定の扱い

利用可能資産の算出では、
対象年月時点で有効な
`asset_account_available_settings`を
参照する。

現在の設定だけではなく、
現在資産状況として選択された
`target_year_month`に対応する
利用可能資産設定を使用する。

概念的には、

```text
最新の確定済みtarget_year_month
    ↓
対象年月時点の
asset_account_available_settings
    ↓
利用可能資産を算出
```

とする。

これにより、
過去の確定済み対象年月が
現在資産状況として選択された場合でも、
その対象年月に対応する設定で
利用可能資産を算出する。

---

#### 11.11 他利用者データの除外

現在資産状況の算出では、
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

他の利用者に属する
最新の確定済み月末資産状況が
存在していても、
対象年月の決定へ影響させない。

---

#### 11.12 業務状態に依存する検証

以下は、
単項目バリデーションではなく、
現在資産状況を算出するための
業務ルールとして扱う。

- 操作対象利用者に確定済み月末資産状況が存在すること
- 最新の確定済み対象年月を正しく特定すること
- 未確定の月末資産状況を対象としないこと
- 残高記録単位に応じて集計対象を切り替えること
- 月末資産残高と商品別月末評価額を重複加算しないこと
- 対象年月時点の利用可能資産設定を使用すること
- 他の利用者に属する資産情報を集計しないこと

これらの業務ルールは、
FormRequestなどの
入力値バリデーションではなく、
Serviceおよび
資産集計処理で保証する。

---

### 12 業務ルール

#### 12.1 現在資産状況の定義

本APIにおける
「現在資産状況」とは、
リクエスト実行日時点の
リアルタイムな資産残高ではない。

操作対象利用者に属する
確定済み月末資産状況のうち、
`target_year_month`が
最新のものを
現在資産状況として扱う。

```text
操作対象利用者
    ↓
confirmed = true
    ↓
target_year_month DESC
    ↓
先頭1件
```

---

#### 12.2 最新の確定済み対象年月

現在資産状況の対象年月は、
以下の条件を満たす
`month_end_asset_snapshots`から特定する。

```text
user_id = 操作対象利用者ID
AND
confirmed = true
```

複数存在する場合は、
`target_year_month`が
最新のレコードを使用する。

例えば、

```text
2026-05
confirmed = true

2026-06
confirmed = true

2026-07
confirmed = false
```

の場合、
現在資産状況の対象年月は、

```text
2026-06
```

とする。

未確定の月末資産状況は、
対象年月の決定に使用しない。

---

#### 12.3 確定済み月末資産状況が存在しない場合

操作対象利用者について、
確定済みの
`month_end_asset_snapshots`が
存在しない場合は、
現在資産状況を算出できない。

この場合、
空の資産状況を
正常レスポンスとして返却しない。

```text
confirmed = true
    ↓
0件
    ↓
現在資産状況取得不可
```

具体的なエラーコードおよび
HTTPステータスは、
「エラーレスポンス」で定義する。

---

#### 12.4 資産額の算出単位

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
残高記録単位を判定
    ├─ 口座単位
    │      ↓
    │  month_end_asset_balances.balance
    │
    └─ 商品単位
           ↓
       month_end_holding_values.value
```

---

#### 12.5 二重計上の防止

同一の資産について、
月末資産残高と
商品別月末評価額を
重複して総資産へ加算してはならない。

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

#### 12.6 口座単位の資産額

口座単位で
残高を記録する資産口座では、
対象となる月末資産状況に紐づく
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

#### 12.7 商品単位の資産額

商品単位で
評価額を記録する資産口座では、
対象となる月末資産状況に紐づく
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

#### 12.8 総資産の算出

総資産は、
対象年月における
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

数式として表すと、
概念的には以下となる。

```text
totalAssets
=
SUM(accountUnitBalances)
+
SUM(holdingUnitValues)
```

---

#### 12.9 利用可能資産の算出

利用可能資産は、
対象年月時点の
`asset_account_available_settings`を
もとに算出する。

現在設定されている値ではなく、
AST-001で現在資産状況として選択された
`target_year_month`時点の
利用可能資産設定を使用する。

概念的には、

```text
対象年月
    ↓
対象年月時点で利用可能な資産口座を特定
    ↓
該当する資産口座の資産額を合計
    ↓
利用可能資産
```

とする。

---

#### 12.10 利用可能資産の二重計上防止

利用可能資産についても、
総資産と同様に
残高記録単位に応じて
算出元を切り替える。

```text
利用可能な口座
    ├─ 口座単位
    │      ↓
    │  month_end_asset_balances.balance
    │
    └─ 商品単位
           ↓
       SUM(month_end_holding_values.value)
```

月末資産残高と
商品別月末評価額を
同一資産について
重複加算してはならない。

---

#### 12.11 資産口座別資産状況

資産口座別資産状況では、
対象年月における
各資産口座の資産額を返却する。

各資産口座について、
残高記録単位に応じて
資産額を算出する。

概念的には、

```text
資産口座A
口座単位
    ↓
balance = 1,000,000

資産口座B
商品単位
    ↓
商品1 = 500,000
商品2 = 300,000
    ↓
assetAmount = 800,000
```

となる。

---

#### 12.12 保有商品別資産状況

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
架空の商品別内訳を生成してはならない。

---

#### 12.13 利用者境界

現在資産状況の算出では、
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

特に、
最新対象年月の決定は
利用者単位で行う。

---

#### 12.14 現在の資産口座状態との関係

AST-001は、
最新の確定済み対象年月における
資産状況を表示する。

そのため、
対象年月の資産状況を
現在の資産口座状態だけで
再構成してはならない。

対象年月に記録された
月末資産データおよび
対象年月に適用される設定をもとに
資産状況を算出する。

---

#### 12.15 集計結果を保存しない

本APIで算出した

- 総資産
- 利用可能資産
- 資産口座別資産額
- 保有商品別資産額

を、
別の集計テーブルへ保存しない。

既存の月末資産データから
取得時に算出する。

---

### 13 取得条件

現在資産状況の取得では、
最初に操作対象利用者の
最新の確定済み月末資産状況を特定する。

概念的な条件は、
以下とする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.confirmed
    = true

ORDER BY
    target_year_month DESC

LIMIT 1
```

対象となる
`month_end_asset_snapshots`を
特定した後、
そのスナップショットに紐づく
月末資産データを取得する。

概念的には、

```text
snapshotId
    ↓
month_end_asset_balances

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
対象年月時点の
`asset_account_available_settings`を
参照する。

---

### 14 集計方法

#### 14.1 総資産

総資産は、
対象となる全資産口座の
資産額を合計する。

```text
totalAssets
=
口座単位資産額の合計
+
商品単位資産額の合計
```

---

#### 14.2 利用可能資産

利用可能資産は、
対象年月時点で
利用可能資産として扱う
資産口座の資産額のみを
合計する。

```text
availableAssets
=
利用可能な資産口座の
資産額合計
```

---

#### 14.3 資産口座別

資産口座ごとに、
以下を判定する。

```text
balance_recording_unit
```

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

#### 14.4 保有商品別

商品単位で管理する
資産口座について、
保有商品ごとの
`month_end_holding_values.value`を
資産額として使用する。

```text
holdingAssetAmount
=
month_end_holding_values.value
```

---

#### 14.5 集計値の整合性

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
最新の確定済み対象年月における
現在資産状況を返却する。

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
    "targetYearMonth": "2026-06",
    "totalAssets": 3500000,
    "availableAssets": 1500000,
    "assetAccounts": [
      {
        "assetAccountId": "1",
        "name": "普通預金",
        "assetAmount": 1000000,
        "available": true,
        "holdingAssets": []
      },
      {
        "assetAccountId": "2",
        "name": "証券口座",
        "assetAmount": 2500000,
        "available": false,
        "holdingAssets": [
          {
            "holdingAssetId": "10",
            "name": "投資信託A",
            "assetAmount": 1500000
          },
          {
            "holdingAssetId": "11",
            "name": "投資信託B",
            "assetAmount": 1000000
          }
        ]
      }
    ]
  }
}
```

---

#### 15.1 口座単位のレスポンス

口座単位で
残高を記録する資産口座では、
`holdingAssets`を
空配列として返却する。

例：

```json
{
  "assetAccountId": "1",
  "name": "普通預金",
  "assetAmount": 1000000,
  "available": true,
  "holdingAssets": []
}
```

架空の商品別内訳は
生成しない。

---

#### 15.2 商品単位のレスポンス

商品単位で
評価額を記録する資産口座では、
資産口座の資産額と
保有商品別資産額を返却する。

例：

```json
{
  "assetAccountId": "2",
  "name": "証券口座",
  "assetAmount": 2500000,
  "available": false,
  "holdingAssets": [
    {
      "holdingAssetId": "10",
      "name": "投資信託A",
      "assetAmount": 1500000
    },
    {
      "holdingAssetId": "11",
      "name": "投資信託B",
      "assetAmount": 1000000
    }
  ]
}
```

この場合、

```text
2500000
=
1500000
+
1000000
```

となる。

---

### 16 レスポンス項目

#### 16.1 data

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `targetYearMonth` | string | × | 現在資産状況として使用した最新の確定済み対象年月 |
| `totalAssets` | integer | × | 対象年月における総資産 |
| `availableAssets` | integer | × | 対象年月における利用可能資産 |
| `assetAccounts` | array | × | 資産口座別資産状況 |

金額は、
日本円の整数値として返却する。

---

#### 16.2 assetAccounts

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `assetAccountId` | string | × | 資産口座ID |
| `name` | string | × | 資産口座名 |
| `assetAmount` | integer | × | 対象年月における資産口座の資産額 |
| `available` | boolean | × | 対象年月時点で利用可能資産として扱うか |
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
| `assetAmount` | integer | × | 対象年月における商品別月末評価額 |

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

#### 16.4 targetYearMonth

`targetYearMonth`は、
現在資産状況として採用された
最新の確定済み対象年月を返却する。

形式は、

```text
YYYY-MM
```

とする。

例：

```json
{
  "targetYearMonth": "2026-06"
}
```

---

#### 16.5 totalAssets

`totalAssets`は、
対象年月における
総資産額を返却する。

```text
totalAssets
=
口座単位資産額の合計
+
商品単位資産額の合計
```

例：

```json
{
  "totalAssets": 3500000
}
```

---

#### 16.6 availableAssets

`availableAssets`は、
対象年月時点で
利用可能資産として扱われる
資産口座の資産額合計を返却する。

例：

```json
{
  "availableAssets": 1500000
}
```

`totalAssets`とは
別の概念として扱う。

---

#### 16.7 assetAmount

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

#### 16.8 available

`available`は、
対象年月時点で
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

現在の設定ではなく、
`targetYearMonth`時点の
利用可能資産設定をもとに決定する。

---

#### 16.9 返却しない情報

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

現在資産状況の表示に
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
| `404 Not Found` | `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` | 操作対象利用者に確定済み月末資産状況が存在しない |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバーエラーが発生した |

エラーレスポンス形式は、
API共通方針に従う。

概念例：

```json
{
  "error": {
    "code": "CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND",
    "message": "確定済みの月末資産状況が存在しません。",
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

現在資産状況の取得処理は
実行しない。

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

他の利用者の資産情報を
代替して返却してはならない。

---

#### 17.4 CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND

操作対象利用者について、
以下の条件を満たす
月末資産状況が
1件も存在しない場合は、

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

を返却する。

```text
user_id = 操作対象利用者ID
AND
confirmed = true
```

例えば、

```text
2026-05
confirmed = false

2026-06
confirmed = false
```

の場合、
現在資産状況として使用できる
月末資産状況が存在しないため、
本エラーとする。

空の資産状況として、

```json
{
  "data": {
    "totalAssets": 0,
    "availableAssets": 0,
    "assetAccounts": []
  }
}
```

のような
正常レスポンスは返却しない。

---

#### 17.5 最新年月が未確定の場合

最新の月末資産状況が
未確定であっても、
それ以前に確定済みの
月末資産状況が存在する場合は、
エラーとしない。

例えば、

```text
2026-05
confirmed = true

2026-06
confirmed = true

2026-07
confirmed = false
```

の場合は、
`2026-06`を対象として
`200 OK`を返却する。

```text
2026-07が未確定
    ↓
エラーにはしない
    ↓
最新の確定済み年月を検索
    ↓
2026-06を取得
```

---

#### 17.6 集計対象データの不整合

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
不整合内容を直接公開しない。

---

#### 17.7 INTERNAL_SERVER_ERROR

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
| `200 OK` | 現在資産状況の取得成功 |
| `400 Bad Request` | 利用者コンテキストの指定不備 |
| `404 Not Found` | 利用者または取得対象となる確定済み月末資産状況が存在しない |
| `500 Internal Server Error` | 想定外のサーバーエラー |

本APIでは、
リクエストボディ、
パスパラメータおよび
クエリパラメータを使用しないため、
業務入力項目に対する
`422 Unprocessable Entity`は
原則として使用しない。

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
現在資産状況として算出した

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
設定の登録・更新は行わない。

---

### 20 トランザクション

本APIは、
読み取り専用APIであるため、
明示的な更新トランザクションは
使用しない。

概念的には、

```text
最新の確定済み月末資産状況を取得
    ↓
月末資産残高を取得
    ↓
商品別月末評価額を取得
    ↓
対象年月時点の利用可能資産設定を取得
    ↓
資産状況を集計
    ↓
レスポンス
```

とする。

データの更新を伴わないため、
`DB::transaction()`によって
更新処理をまとめる必要はない。

ただし、
複数のSELECT間で
厳密な読み取り一貫性が
必要となる要件が
将来的に追加された場合は、
トランザクション分離レベルを含めて
別途検討する。

Phase1では、
確定済み月末資産状況は
表示用の確定データとして扱うため、
通常の読み取り処理とする。

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

本APIは
資産情報を参照するだけであり、
月末資産状況、
月末資産残高、
商品別月末評価額などの
更新処理をブロックする必要はない。

また、
AST-001の実行によって
月末資産状況の確定処理や
確定解除処理を
不要に待機させてはならない。

---

### 22 キャッシュ

Phase1では、
AST-001専用の
サーバー側アプリケーションキャッシュは
使用しない。

現在資産状況は、
最新の確定済み
月末資産状況をもとに算出するため、
確定状態の変更によって
取得対象となる年月が
変化する可能性がある。

例えば、

```text
2026-06
confirmed = true

2026-07
confirmed = false
```

の状態では、
`2026-06`が現在資産状況となる。

その後、

```text
2026-07
confirmed = true
```

となった場合は、
`2026-07`が現在資産状況となる。

Phase1では、
キャッシュ無効化処理を
追加するよりも、
リクエストごとに
最新の確定済みデータから
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

同一のデータ状態に対して
同一利用者が
同じリクエストを複数回実行しても、
業務データの状態は変化しない。

```text
GET /api/v1/asset-summaries/current
    ↓
現在資産状況取得

GET /api/v1/asset-summaries/current
    ↓
現在資産状況取得
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
新しい月末資産状況が確定された場合は、
取得対象となる
`targetYearMonth`および
レスポンス内容が
変化する可能性がある。

これは、
AST-001の副作用によるものではなく、
参照対象となる
業務データが変更されたためである。

本APIはGETであり、
重複登録などの副作用がないため、
`Idempotency-Key`は使用しない。

---

### 24 関連テーブル

#### 24.1 month_end_asset_snapshots

現在資産状況の
対象年月を特定するために使用する。

本APIでは、
操作対象利用者に属する
確定済み月末資産状況のうち、
`target_year_month`が
最新のレコードを取得する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 月末資産状況ID |
| `user_id` | 利用者境界確認 |
| `target_year_month` | 現在資産状況の対象年月決定 |
| `confirmed` | 確定済み月末資産状況のみを対象とする判定 |

取得条件は、
概念的に以下とする。

```text
user_id = 操作対象利用者ID
AND
confirmed = true

ORDER BY
target_year_month DESC

LIMIT 1
```

本APIでは、
`month_end_asset_snapshots`を更新しない。

---

#### 24.2 month_end_asset_balances

口座単位で
残高を記録する資産口座について、
対象年月の月末資産残高を
取得するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `month_end_asset_snapshot_id` | 対象となる月末資産状況との関連 |
| `asset_account_id` | 資産口座との関連 |
| `balance` | 口座単位の資産額 |

取得対象は、
AST-001で特定した
最新の確定済み月末資産状況に
紐づくレコードとする。

```text
month_end_asset_snapshot_id
    = 最新の確定済みsnapshotId
```

口座単位の資産口座についてのみ、
`balance`を資産額として使用する。

商品単位の資産口座については、
`balance`を総資産へ
重複加算しない。

本APIでは、
`month_end_asset_balances`を更新しない。

---

#### 24.3 month_end_holding_values

商品単位で
評価額を記録する資産口座について、
対象年月の商品別月末評価額を
取得するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `month_end_asset_snapshot_id` | 対象となる月末資産状況との関連 |
| `holding_asset_id` | 保有商品との関連 |
| `value` | 商品単位の資産額 |

取得対象は、
AST-001で特定した
最新の確定済み月末資産状況に
紐づくレコードとする。

```text
month_end_asset_snapshot_id
    = 最新の確定済みsnapshotId
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

#### 24.4 asset_accounts

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

#### 24.5 holding_assets

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

また、
資産口座との関連は、
以下とする。

```text
holding_assets.asset_account_id
    = asset_accounts.id
```

本APIでは、
`holding_assets`を更新しない。

---

#### 24.6 asset_account_available_settings

対象年月時点で
各資産口座を
利用可能資産として扱うかを
判定するために使用する。

判定対象となる年月は、
AST-001で特定した
最新の確定済み
`target_year_month`とする。

概念的には、
以下を判定する。

```text
asset_account_id
+
target_year_month
    ↓
対象年月時点で
利用可能資産として扱うか
```

現在時点の設定だけではなく、
AST-001の対象年月に対応する
設定を使用する。

本APIでは、
`asset_account_available_settings`を更新しない。

---

#### 24.7 users

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

#### 24.8 関連しないテーブル

本APIでは、
以下のテーブルを
現在資産状況取得のために
参照しない。

- `net_incomes`
- `objectives`
- `assessment_histories`

AST-001は、
現在資産状況の表示に
責務を限定する。

---

### 25 関連する機能要件

- 現在の資産状況
  - 最新の確定済み対象年月を現在資産状況として扱う
  - 未確定の月末資産状況は表示対象としない
  - 確定済み月末資産状況が存在しない場合は現在資産状況を取得できない

- 総資産
  - 口座単位の資産口座は月末資産残高を使用する
  - 商品単位の資産口座は商品別月末評価額を使用する
  - 月末資産残高と商品別月末評価額を重複加算しない
  - 各資産口座の資産額合計を総資産とする

- 利用可能資産
  - 対象年月時点の利用可能資産設定を使用する
  - 利用可能資産として扱う資産口座のみを合計する
  - 総資産と利用可能資産は別の値として扱う

- 資産口座別資産状況
  - 資産口座ごとの資産額を取得できる
  - 口座単位と商品単位で算出方法を切り替える

- 保有商品別資産状況
  - 商品単位で管理する資産口座について保有商品別評価額を取得できる
  - 口座単位の資産口座について架空の商品別内訳を生成しない

- 利用者境界
  - 操作対象利用者に属する資産情報のみを取得する
  - 他の利用者に属する資産情報を集計しない
  - 最新対象年月の決定も利用者単位で行う

- API共通
  - IDはAPIレスポンス上stringとして扱う
  - JSONフィールド名はcamelCaseとする
  - エラー時は共通エラーレスポンス形式を使用する

具体的な章番号は、
`functional-requirements.md`の
最新定義に従う。

---

### 26 設計上の補足

#### 26.1 currentをURLに含める理由

AST-001は、
特定年月ではなく、
最新の確定済み対象年月を
取得する特殊な資産状況取得である。

そのため、

```http
/api/v1/asset-summaries/current
```

として、
現在資産状況であることを
URL上で明示する。

---

#### 26.2 targetYearMonthを受け取らない理由

AST-001では、
「現在」を
最新の確定済み対象年月として
システム側で決定する。

クライアントから
`targetYearMonth`を受け取ると、

```text
現在資産状況取得
```

と

```text
指定年月資産状況取得
```

の責務が重複する。

そのため、

```text
AST-001
    → targetYearMonthを受け取らない

AST-002
    → targetYearMonthを受け取る
```

と分離する。

---

#### 26.3 最新の月末資産状況ではなく最新の確定済みを使用する理由

最新の月末資産状況が
入力途中で未確定の場合、
そのデータを現在資産状況として表示すると、
不完全な資産額を表示する可能性がある。

そのため、

```text
最新
+
confirmed = true
```

の月末資産状況を使用する。

---

#### 26.4 確定済み月末資産状況がない場合に0円を返さない理由

確定済み月末資産状況が
存在しないことと、

```text
総資産 = 0円
```

は異なる状態である。

0円を返却すると、

```text
まだ資産状況を確定していない
```

のか、

```text
本当に資産が0円
```

なのかを区別できない。

そのため、
確定済み月末資産状況が
存在しない場合は、
専用エラーとして扱う。

---

#### 26.5 残高記録単位によって算出元を切り替える理由

資産口座には、

```text
口座全体の残高を記録する方式
```

と

```text
保有商品ごとの評価額を記録する方式
```

が存在する。

両方を加算すると
同一資産が二重計上される可能性がある。

そのため、
`balance_recording_unit`に応じて
算出元を一意に決定する。

---

#### 26.6 availableAssetsをtotalAssetsと分ける理由

総資産には、
現在保有している
すべての対象資産を含める。

一方、
利用可能資産は、
目的達成などで
実際に利用可能とみなす資産のみを
集計する。

例えば、

```text
総資産
3,500,000円

うち利用可能資産
1,500,000円
```

のように、
異なる意味を持つため
別項目として返却する。

---

#### 26.7 対象年月時点の利用可能資産設定を使う理由

最新の確定済み対象年月が
必ずしも現在月とは限らない。

例えば、

```text
現在
2026-08

最新確定月
2026-06
```

の場合、
2026-08時点の設定ではなく、
2026-06時点の設定で
利用可能資産を算出する必要がある。

そのため、
AST-001の`targetYearMonth`を基準に
利用可能資産設定を判定する。

---

#### 26.8 集計値を保存しない理由

`totalAssets`や
`availableAssets`は、
既存の月末資産データから
導出できる値である。

Phase1では、
これらを別テーブルへ
重複保存しない。

これにより、

```text
元データ
+
集計テーブル
```

の同期問題を避ける。

---

#### 26.9 APIレスポンスで残高記録単位を返さない理由

AST-001では、
バックエンドが
残高記録単位に応じて
必要な資産額を算出済みである。

フロントエンドが
再度、

```text
口座単位か
商品単位か
```

を判定して
資産額を計算する必要はない。

そのため、
Phase1の現在資産状況レスポンスでは
`balance_recording_unit`を返却しない。

---

#### 26.10 holdingAssetsを常に配列で返す理由

口座単位の資産口座でも、
`holdingAssets`を
`null`ではなく
空配列として返却する。

```json
{
  "holdingAssets": []
}
```

これにより、
React側では、

```ts
account.holdingAssets.map(...)
```

のような
一貫した型として扱える。

---

#### 26.11 GETを採用する理由

AST-001は、
現在資産状況を
参照するだけのAPIである。

データの登録、
更新または削除を行わないため、
HTTPメソッドには
`GET`を採用する。

---

#### 26.12 AST-001が冪等である理由

AST-001は、
同一リクエストを
複数回実行しても、
業務データを変更しない。

したがって、
本APIは冪等である。

別APIによって
新しい月末資産状況が確定された場合に
レスポンスが変化しても、
AST-001自体による
副作用ではない。

---

#### 26.13 Idempotency-Keyを使用しない理由

AST-001は、
読み取り専用のGET APIである。

再実行によって
データの重複登録などが
発生しないため、
`Idempotency-Key`は使用しない。

---

#### 26.14 サーバーキャッシュを採用しない理由

SNP-004やSNP-005によって、
最新の確定済み対象年月が
変化する可能性がある。

Phase1では、
キャッシュ無効化処理を
追加するよりも、
リクエストごとに
最新の確定済みデータから
算出する。

性能要件が生じた場合は、
将来的にキャッシュ戦略を
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