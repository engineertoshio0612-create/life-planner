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

### 26 テスト観点

#### 26.1 正常系

操作対象利用者について、
確定済み月末資産状況および
必要な資産データが
存在する状態で
AST-001を実行する。

以下を確認する。

- `200 OK`となること
- 最新の確定済み対象年月が返却されること
- 総資産が正しく算出されること
- 利用可能資産が正しく算出されること
- 資産口座別資産状況が返却されること
- 商品単位の資産口座について保有商品別資産状況が返却されること
- データベースが更新されないこと

---

#### 26.2 最新の確定済み対象年月

以下の月末資産状況を用意する。

```text
2026-05
confirmed = true

2026-06
confirmed = true

2026-07
confirmed = true
```

期待結果：

```text
targetYearMonth = 2026-07
```

となること。

---

#### 26.3 最新年月が未確定

以下の状態を用意する。

```text
2026-05
confirmed = true

2026-06
confirmed = true

2026-07
confirmed = false
```

期待結果：

```text
targetYearMonth = 2026-06
```

となること。

未確定の`2026-07`を
現在資産状況へ使用しないこと。

---

#### 26.4 確定済み月末資産状況が存在しない

以下の状態を用意する。

```text
2026-05
confirmed = false

2026-06
confirmed = false
```

期待結果：

```text
404 Not Found
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

以下を確認する。

- `totalAssets = 0`の正常レスポンスを返却しないこと
- 未確定データを代替利用しないこと

---

#### 26.5 口座単位の資産口座

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

#### 26.6 商品単位の資産口座

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

#### 26.7 商品単位の複数保有商品

商品単位の資産口座に
複数の商品別月末評価額を用意する。

以下を確認する。

- すべての`value`が資産口座単位で合計されること
- 各保有商品の`assetAmount`に個別の`value`が返却されること
- 資産口座の`assetAmount`と保有商品別合計が一致すること

---

#### 26.8 二重計上しないこと

商品単位の資産口座について、
同じ対象年月に

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

残高記録単位に従って
商品別月末評価額のみを
集計すること。

---

#### 26.9 口座単位で商品別評価額を加算しないこと

口座単位の資産口座に、
何らかの理由で
商品別月末評価額が存在する状態を用意する。

期待結果：

- `month_end_asset_balances.balance`のみを使用すること
- 商品別月末評価額を総資産へ加算しないこと

---

#### 26.10 複数資産口座の総資産

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

#### 26.11 totalAssetsと資産口座別合計

正常データについて、

```text
totalAssets
=
SUM(assetAccounts.assetAmount)
```

となることを確認する。

---

#### 26.12 0円の月末資産残高

口座単位の資産口座について、

```text
balance = 0
```

を用意する。

以下を確認する。

- 未登録扱いにならないこと
- `assetAmount = 0`として返却されること
- 総資産へ0円として正常に集計されること

---

#### 26.13 0円の商品別月末評価額

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

#### 26.14 利用可能資産

複数の資産口座について、
対象年月時点の
利用可能資産設定を
以下のように用意する。

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

#### 26.15 すべて利用可能

すべての資産口座が
対象年月時点で
利用可能資産として扱われる状態とする。

期待結果：

```text
availableAssets
=
totalAssets
```

となること。

---

#### 26.16 利用可能資産が0円

すべての資産口座が
対象年月時点で
利用可能資産として
扱われない状態とする。

期待結果：

```text
availableAssets = 0
```

となること。

`0`をエラーとして扱わないこと。

---

#### 26.17 対象年月時点の利用可能資産設定

以下のように、
同じ資産口座について
対象年月によって
利用可能状態が異なる設定を用意する。

```text
2026-05
available = true

2026-06
available = false
```

AST-001の対象年月が
`2026-06`となる状態で実行する。

期待結果：

```text
available = false
```

となること。

現在の設定や
別年月の設定を
誤って使用しないこと。

---

#### 26.18 口座単位のholdingAssets

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

#### 26.19 商品単位のholdingAssets

商品単位の資産口座について、
対象年月に紐づく
保有商品別資産状況が
返却されることを確認する。

各項目について、

- `holdingAssetId`
- `name`
- `assetAmount`

が正しく返却されること。

---

#### 26.20 他利用者の最新月を使用しないこと

以下の状態を用意する。

```text
User A
2026-06
confirmed = true

User B
2026-07
confirmed = true
```

User Aを
`X-User-Id`として
AST-001を実行する。

期待結果：

```text
targetYearMonth = 2026-06
```

となること。

User Bの
`2026-07`を使用しないこと。

---

#### 26.21 他利用者の月末資産残高を含めないこと

User AとUser Bについて、
同一対象年月に
月末資産残高を用意する。

User Aとして
AST-001を実行する。

以下を確認する。

- User Aの月末資産残高のみ集計されること
- User Bの残高が`totalAssets`へ加算されないこと

---

#### 26.22 他利用者の商品別月末評価額を含めないこと

User AとUser Bについて、
商品別月末評価額を用意する。

User Aとして
AST-001を実行する。

以下を確認する。

- User Aの商品別月末評価額のみ集計されること
- User Bの商品別月末評価額が混入しないこと

---

#### 26.23 他利用者の利用可能資産設定を使用しないこと

User AとUser Bに
利用可能資産設定を用意する。

User Aとして
AST-001を実行する。

以下を確認する。

- User Aに属する資産口座の設定のみ使用されること
- User Bの設定によって`availableAssets`が変化しないこと

---

#### 26.24 利用者コンテキスト未指定

`X-User-Id`を
指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

---

#### 26.25 利用者ID形式不正

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

#### 26.26 利用者不存在

存在しない利用者IDを
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 26.27 論理削除済み利用者

論理削除済み利用者を
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 26.28 副作用

AST-001実行前後で、
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

- 月末資産状況の確定状態が変更されないこと
- 集計結果をデータベースへ保存しないこと
- 利用可能資産設定が変更されないこと

---

#### 26.29 冪等性

同一のデータ状態で
同じリクエストを
複数回実行する。

```text
1回目
GET /api/v1/asset-summaries/current
    ↓
200 OK

2回目
GET /api/v1/asset-summaries/current
    ↓
200 OK
```

以下を確認する。

- データベースが変更されないこと
- 同一状態であれば同じ対象年月となること
- 同一状態であれば同じ集計結果となること

---

#### 26.30 新しい月末資産状況確定後の再取得

最初に以下の状態とする。

```text
2026-06
confirmed = true

2026-07
confirmed = false
```

AST-001を実行し、

```text
targetYearMonth = 2026-06
```

となることを確認する。

その後、
SNP-004によって
`2026-07`を確定する。

再度AST-001を実行する。

期待結果：

```text
targetYearMonth = 2026-07
```

となること。

これはAST-001の
非冪等性によるものではなく、
参照データの変更によるものであること。

---

#### 26.31 レスポンス契約

正常取得時に、
API共通方針で定めた
Envelope形式で返却されること。

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
      }
    ]
  }
}
```

以下を確認する。

- `data`がobjectであること
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

#### 26.32 返却しない情報

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

#### 26.33 エラーレスポンス

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

AST-001では、
Action、
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

現在資産状況取得に必要な業務フローの制御は、UseCaseへ集約する。

Actionへ、最新対象年月の判定、資産集計、利用可能資産判定、利用者境界確認などの業務ロジックを直接記述しない。

---

#### 27.1 Action

HTTPリクエストを受け付け、利用者コンテキストから操作対象利用者IDを取得する。

現在資産状況取得UseCaseを呼び出し、取得結果をResponderへ渡す。

概念例：

```php
final class GetCurrentAssetSummaryAction
{
    public function __invoke(
        GetCurrentAssetSummaryUseCase $useCase,
        CurrentAssetSummaryResponder $responder,
        userContext $userContext,
    ): JsonResponse {
        $summary = $useCase->execute(
            userId: $userContext->userId,
        );

        return $responder->ok(
            $summary,
        );
    }
}
```

Actionでは、以下を行わない。

- 最新の確定済み月末資産状況の検索
- 月末資産残高の取得
- 商品別月末評価額の取得
- 残高記録単位の判定
- 総資産の集計
- 利用可能資産の集計
- 利用可能資産設定の判定
- 資産口座別資産状況の組み立て
- 保有商品別資産状況の組み立て
- 利用者境界の判定
- APIレスポンス形式への変換

---

#### 27.2 Request

本APIでは、以下を使用しない。

- パスパラメータ
- クエリパラメータ
- リクエストボディ

そのため、AST-001専用のFormRequestは作成しない。

`X-User-Id`の検証および操作対象利用者コンテキストの生成は、API共通Middlewareで行う。

以下のような空のRequestクラスは作成しない。

```php
final class GetCurrentAssetSummaryRequest
    extends FormRequest
{
}
```

不要なクラスを追加しない。

---

#### 27.3 UseCase

現在資産状況取得のユースケース処理を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. 最新の確定済み月末資産状況を取得する
3. 確定済み月末資産状況が存在しない場合は業務例外を送出する
4. 対象となる`target_year_month`と`snapshotId`を確定する
5. 対象snapshotに紐づく資産データを取得する
6. 対象年月時点の利用可能資産設定を取得する
7. 資産口座ごとの資産額を算出する
8. 商品単位の資産口座について保有商品別資産状況を構成する
9. 総資産を算出する
10. 利用可能資産を算出する
11. `AssetSummary` DTOを生成して返却する

概念的な処理は、以下とする。

```text
操作対象利用者
    ↓
最新の確定済み
月末資産状況取得
    ↓
存在しない
    → CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND

存在する
    ↓
targetYearMonth
snapshotId
確定
    ↓
資産データ取得
    ↓
対象年月時点の
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

UseCaseでは、SQLを直接記述しない。

データベースアクセスは、Queryクラスへ委譲する。

---

#### 27.4 MonthEndAssetSnapshotQuery

操作対象利用者に属する最新の確定済み月末資産状況を取得する。

検索条件には、必ず利用者IDを含める。

概念例：

```php
final class MonthEndAssetSnapshotQuery
{
    public function findLatestConfirmedForUser(
        int $userId,
    ): ?MonthEndAssetSnapshot {
        return MonthEndAssetSnapshot::query()
            ->where(
                'user_id',
                $userId,
            )
            ->where(
                'confirmed',
                true,
            )
            ->orderByDesc(
                'target_year_month',
            )
            ->first([
                'id',
                'target_year_month',
            ]);
    }
}
```

他の利用者に属する月末資産状況を検索対象へ含めない。

未確定の月末資産状況は、対象年月が最新であっても取得対象としない。

---

#### 27.5 確定済み月末資産状況不存在

MonthEndAssetSnapshotQueryの取得結果が`null`の場合は、

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

へ変換するための業務例外を送出する。

概念例：

```php
$snapshot =
    $this->monthEndAssetSnapshotQuery
        ->findLatestConfirmedForUser(
            $userId,
        );

if ($snapshot === null) {
    throw new
        ConfirmedAssetSnapshotNotFoundException();
}
```

未確定の月末資産状況を代替利用しない。

また、

```text
totalAssets = 0
availableAssets = 0
assetAccounts = []
```

のような空のAssetSummaryを代替レスポンスとして生成しない。

---

#### 27.6 AssetSummaryQuery

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

資産口座の`balance_recording_unit`に応じて、資産額算出に使用するデータを切り替えられる形で取得する。

---

#### 27.7 利用者境界

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

`snapshotId`だけを条件として、資産データを取得しない。

---

#### 27.8 口座単位の資産額

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

商品別月末評価額が何らかの理由で存在していても、総資産へ加算しない。

---

#### 27.9 商品単位の資産額

商品単位で評価額を記録する資産口座では、

```text
month_end_holding_values.value
```

を使用する。

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

#### 27.10 二重計上の防止

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

#### 27.11 AssetAvailabilityQuery

対象年月時点の利用可能資産設定を取得する。

概念例：

```php
$availabilityMap =
    $this->assetAvailabilityQuery
        ->findForTargetYearMonth(
            userId: $userId,
            targetYearMonth:
                $snapshot->target_year_month,
        );
```

現在日時点の設定ではなく、AST-001で採用された`target_year_month`を基準に取得する。

---

#### 27.12 利用可能資産設定の利用者境界

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

#### 27.13 対象年月時点の設定判定

利用可能資産設定は、対象年月時点で有効な設定を使用する。

現在の最新設定をそのまま利用しない。

概念的には、

```text
AST-001のtargetYearMonth
    ↓
対象年月時点で有効な
asset_account_available_settings
    ↓
available判定
```

とする。

具体的な有効期間判定は、`asset_account_available_settings`のテーブル定義および業務ルールに従う。

---

#### 27.14 AssetSummary DTO

現在資産状況は、Eloquent ModelをそのままResourceへ渡すのではなく、表示用DTOとして構成する。

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

---

#### 27.15 資産口座別資産状況の生成

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

#### 27.16 口座単位のholdingAssets

口座単位で残高を記録する資産口座では、`holdingAssets`を空配列とする。

概念例：

```php
new AssetAccountSummary(
    assetAccountId: $assetAccountId,
    name: $name,
    assetAmount: $balance,
    available: $available,
    holdingAssets: [],
);
```

架空の商品別内訳を生成しない。

---

#### 27.17 商品単位のholdingAssets

商品単位で評価額を記録する資産口座では、保有商品ごとに`HoldingAssetSummary`を生成する。

概念例：

```php
$holdingAssets[] =
    new HoldingAssetSummary(
        holdingAssetId:
            $holdingAsset->id,
        name:
            $holdingAsset->name,
        assetAmount:
            $holdingValue->value,
    );
```

資産口座の`assetAmount`は、生成された`HoldingAssetSummary`の`assetAmount`合計とする。

---

#### 27.18 総資産の集計

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

#### 27.19 利用可能資産の集計

利用可能資産は、対象年月時点で

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

資産額を利用可能資産集計のために再度データベースから取得しない。

資産口座別集計結果を共通利用する。

---

#### 27.20 0円の扱い

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

0円の資産口座および0円の保有商品も、対象データが存在する限りDTOへ含める。

---

#### 27.21 QueryとUseCaseの責務分離

Queryは、データベースから必要なデータを取得することに責務を限定する。

UseCaseは、複数Queryの結果を組み合わせて現在資産状況を構成する。

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

#### 27.22 AST-002との共通化

AST-001とAST-002の主な違いは、対象となる確定済み月末資産状況の特定方法である。

```text
AST-001
    ↓
最新の確定済みsnapshot

AST-002
    ↓
指定年月の確定済みsnapshot
```

snapshotが特定された後の、以下は可能な限りAST-001とAST-002で共通化する。

- 資産データ取得
- 残高記録単位による算出元切り替え
- 資産口座別集計
- 保有商品別集計
- 利用可能資産判定
- 総資産集計
- 利用可能資産集計
- DTO生成

同一の集計ロジックを二重実装しない。

---

#### 27.23 API Resource

AssetSummary DTOを、API ResourceによってAPIレスポンス形式へ変換する。

概念例：

```php
final class CurrentAssetSummaryResource
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

API Resourceでは、現在資産状況の表示に必要な情報だけを返却する。

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

Eloquent ModelをそのままJSON化してはならない。

---

#### 27.27 Responder

Responderは、生成済みの`AssetSummary` DTOを受け取り、API共通方針に従ったHTTPレスポンスへ変換する。

概念例：

```php
final class CurrentAssetSummaryResponder
{
    public function ok(
        AssetSummary $summary,
    ): JsonResponse {
        return response()->json(
            [
                'data' =>
                    new CurrentAssetSummaryResource(
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

- データベース検索
- 最新対象年月の判定
- 残高記録単位の判定
- 資産口座別集計
- 保有商品別集計
- 総資産集計
- 利用可能資産集計
- 利用可能資産設定の判定
- 利用者境界の判定

Responderは、取得済みのAssetSummaryをHTTPレスポンスへ変換することに責務を限定する。

---

#### 27.29 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONレスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

#### 27.30 Repository

AST-001では、Repositoryを使用しない。

本APIは読み取り専用APIであり、以下を行わないためである。

- 登録
- 更新
- 削除

参照処理は、Queryクラスへ集約する。

---

#### 27.31 トランザクション

AST-001は、読み取り専用APIであるため、Phase1では明示的な`DB::transaction()`を使用しない。

以下のような実装は行わない。

```php
DB::transaction(
    function () {
        // 現在資産状況取得のみ
    },
);
```

複数SELECT間で厳密な読み取り一貫性が必要となる要件が将来的に追加された場合は、トランザクション分離レベルを含めて別途検討する。

---

#### 27.32 ロック

AST-001では、行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

現在資産状況の取得によって、月末資産状況の確定・確定解除処理などを不要にブロックしない。

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

基本的には、専用Queryによって以下を必要な単位でまとめて取得する。

- 口座単位資産データ
- 商品単位資産データ
- 利用可能資産設定

その後、UseCase上でMap化・グルーピングして資産口座別DTOを生成する。

---

#### 27.34 取得カラム

Queryでは、現在資産状況の構築に必要なカラムを中心に取得する。

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

Phase1では、AST-001専用のサーバー側アプリケーションキャッシュを使用しない。

新しい月末資産状況が確定または確定解除されると、AST-001で採用される`targetYearMonth`が変化する可能性があるためである。

リクエストごとに、最新の確定済み月末資産状況を取得する。

---

#### 27.36 例外変換

Laravel内部例外をそのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 確定済み月末資産状況不存在 | `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

確定済み月末資産状況が存在しない場合は、空のAssetSummaryを生成しない。

---

#### 27.37 想定外例外

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

#### 27.38 ログ

AST-001では、必要に応じて以下の情報をログコンテキストへ設定する。

```text
requestId
userId
targetYearMonth
snapshotId
```

以下の具体的な金額情報は、不要にアクセスログへ出力しない。

- 総資産
- 利用可能資産
- 資産口座別残高
- 商品別月末評価額

---

#### 27.39 テスト実装方針

Laravel側では、Feature Testを中心としてAST-001のAPI契約および現在資産状況取得処理を確認する。

Feature Testでは、主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `404 Not Found`
- `500 Internal Server Error`
- 最新の確定済み対象年月を取得できること
- 最新年月が未確定の場合は1つ前の確定済み年月を使用すること
- 確定済み月末資産状況不存在
- 口座単位資産の集計
- 商品単位資産の集計
- 月末資産残高と商品別月末評価額の二重計上防止
- 複数資産口座の総資産集計
- `totalAssets`と資産口座別合計の一致
- 利用可能資産の集計
- 対象年月時点の利用可能資産設定を使用すること
- すべて利用可能な場合
- 利用可能資産が0円の場合
- `balance = 0`の扱い
- `value = 0`の扱い
- 口座単位では`holdingAssets = []`となること
- 商品単位では保有商品別資産状況が返却されること
- 他利用者の月末資産状況を使用しないこと
- 他利用者の月末資産残高が混入しないこと
- 他利用者の商品別月末評価額が混入しないこと
- 他利用者の利用可能資産設定を使用しないこと
- IDがstringとして返却されること
- JSONフィールド名がcamelCaseであること
- 返却対象外項目が含まれないこと
- データベースが更新されないこと
- 冪等性

MonthEndAssetSnapshotQueryについては、Database Testで以下を確認する。

```text
user_id一致
+
confirmed = true
+
target_year_month DESC
    ↓
最新1件を取得
```

AssetSummaryQueryについては、以下を確認する。

```text
口座単位
    → balanceのみ取得対象

商品単位
    → valueのみ取得対象

他利用者データ
    → 取得対象外
```

AssetAvailabilityQueryについては、以下を確認する。

```text
操作対象利用者
+
targetYearMonth
    ↓
対象年月時点の設定を取得
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
利用可能資産設定
    ↓
GetCurrentAssetSummaryUseCase
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
パスパラメータ、
クエリパラメータ、
リクエストボディを使用しない。

レスポンス型は、
以下とする。

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

export type CurrentAssetSummary = {
  targetYearMonth: string;
  totalAssets: number;
  availableAssets: number;
  assetAccounts: AssetAccountSummary[];
};

export type GetCurrentAssetSummaryResponse = {
  data: CurrentAssetSummary;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetCurrentAssetSummaryResponse>(
    '/api/v1/asset-summaries/current',
  );
```

ダッシュボードや
現在資産状況画面などから、
最新の確定済み対象年月における
資産状況を表示する際に利用する。

---

#### 28.1 targetYearMonthの扱い

`targetYearMonth`は、
現在資産状況として採用された
最新の確定済み対象年月を表す。

形式は、

```text
YYYY-MM
```

とする。

例：

```ts
const targetYearMonth =
  response.data.targetYearMonth;
```

フロントエンド側で
最新対象年月を独自に決定しない。

AST-001のレスポンスとして
返却された`targetYearMonth`を
現在資産状況の対象年月として扱う。

---

#### 28.2 totalAssetsの扱い

`totalAssets`は、
対象年月における
総資産額として扱う。

```ts
const totalAssets =
  response.data.totalAssets;
```

フロントエンド側で、

```text
月末資産残高
+
商品別月末評価額
```

を再集計して
総資産を算出しない。

総資産の算出責務は、
バックエンドへ集約する。

---

#### 28.3 availableAssetsの扱い

`availableAssets`は、
対象年月時点の
利用可能資産額として扱う。

```ts
const availableAssets =
  response.data.availableAssets;
```

フロントエンド側で
現在の利用可能資産設定から
再計算しない。

対象年月時点の
利用可能資産設定を反映した値として、
APIレスポンスをそのまま使用する。

---

#### 28.4 assetAccountsの扱い

`assetAccounts`は、
対象年月における
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
- 利用可能資産かどうか
- 保有商品別資産状況

---

#### 28.5 assetAccountIdの扱い

`assetAccountId`は、
API共通方針に従って
stringとして扱う。

```ts
const assetAccountId: string =
  account.assetAccountId;
```

numberへ変換して
業務計算には使用しない。

資産口座詳細画面などへの
ルーティングに使用できる。

---

#### 28.6 assetAmountの扱い

`assetAmount`は、
対象年月における
資産口座または
保有商品の資産額として扱う。

```ts
const assetAmount =
  account.assetAmount;
```

資産口座の
残高記録単位によって
フロントエンド側で
算出方法を切り替えない。

APIから返却された
`assetAmount`をそのまま表示する。

---

#### 28.7 availableの扱い

`available`は、
対象年月時点で
その資産口座が
利用可能資産として
扱われるかを表す。

```ts
if (account.available) {
  // 利用可能資産として表示
}
```

現在の資産口座設定を
別APIから取得して
AST-001の`available`を
上書きしない。

---

#### 28.8 holdingAssetsの扱い

`holdingAssets`は、
商品単位で管理する
資産口座について、
保有商品別の資産状況を表す。

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

口座単位で管理する
資産口座では、

```ts
account.holdingAssets.length === 0
```

となる。

空配列を
データ取得エラーとして
扱わない。

---

#### 28.9 holdingAssetIdの扱い

`holdingAssetId`は、
API共通方針に従って
stringとして扱う。

```ts
const holdingAssetId: string =
  holding.holdingAssetId;
```

保有商品詳細画面などへの
ルーティングに使用できる。

---

#### 28.10 金額表示

金額は、
日本円のintegerとして
APIから返却される。

画面表示時は、
フロントエンドで
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

例：

```tsx
<span>
  {formatYen(data.totalAssets)}
</span>
```

表示形式と
業務上の金額計算は分離する。

---

#### 28.11 Queryとして扱う

AST-001は、
読み取り専用のGET APIであるため、
React Query等では
Queryとして扱う。

概念例：

```ts
export const fetchCurrentAssetSummary =
  async () => {
    const response =
      await apiClient.get<GetCurrentAssetSummaryResponse>(
        '/api/v1/asset-summaries/current',
      );

    return response.data;
  };
```

---

#### 28.12 Query Key

Query Keyは、
現在資産状況を表す
固定キーとして管理できる。

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
};
```

操作対象利用者は、
`X-User-Id`によって
API Clientまたは
共通Contextから付与する。

利用者切り替え時は、
現在資産状況のキャッシュを
再取得できるようにする。

---

#### 28.13 Hook実装例

React Queryを使用する場合の
概念例は、
以下とする。

```ts
export const useCurrentAssetSummary =
  () => {
    return useQuery({
      queryKey:
        assetSummaryKeys.current,

      queryFn:
        fetchCurrentAssetSummary,
    });
  };
```

実際のAPI Client、
React Queryおよび
キャッシュ方針は、
フロントエンド共通設計に従う。

---

#### 28.14 ローディング表示

AST-001取得中は、
ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return <Loading />;
}
```

現在資産状況が
まだ取得できていない状態で、
0円の資産状況として
表示しない。

```text
未取得
≠
総資産0円
```

を区別する。

---

#### 28.15 総資産0円の扱い

正常レスポンスとして

```text
totalAssets = 0
```

が返却された場合は、
有効な資産状況として扱う。

```ts
if (
  data.totalAssets === 0
) {
  // 正常な0円表示
}
```

以下のような
truthy / falsy判定によって
未取得扱いにしない。

```ts
if (!data.totalAssets) {
  // 0円もfalseになるため使用しない
}
```

---

#### 28.16 利用可能資産0円の扱い

`availableAssets = 0`も、
正常な業務値として扱う。

例えば、
すべての資産口座が
利用可能資産対象外の場合に
0円となり得る。

```ts
const availableAssets =
  data.availableAssets;
```

0円を
エラーとして表示しない。

---

#### 28.17 資産口座0件の扱い

確定済み月末資産状況が存在し、
APIが正常レスポンスを返したうえで
`assetAccounts`が空配列となるケースが
仕様上発生する場合は、
正常な空状態として表示する。

```tsx
if (
  data.assetAccounts.length === 0
) {
  return (
    <EmptyState>
      表示できる資産口座がありません。
    </EmptyState>
  );
}
```

ただし、
確定済み月末資産状況自体が
存在しない場合とは区別する。

---

#### 28.18 CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND

`CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND`
が返却された場合は、
現在資産状況として表示できる
確定済み月末資産状況が
存在しないことを表示する。

表示例：

```text
確定済みの月末資産状況がありません。
月末資産状況を登録・確定してください。
```

必要に応じて、
月末資産管理画面への
導線を表示する。

この状態を、

```text
totalAssets = 0
```

として表示してはならない。

---

#### 28.19 USER_NOT_FOUND

`USER_NOT_FOUND`
が返却された場合は、
操作対象利用者を
現在利用できない状態として扱う。

利用者選択画面へ戻すなど、
API共通方針に従って処理する。

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
| `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` | 確定済み月末資産状況がないことを表示する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

#### 28.21 自動リトライ

AST-001は
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
```

など、
再送しても解消しない
クライアントまたは
業務状態起因のエラーは、
不要にリトライしない。

---

#### 28.22 クライアントキャッシュ

AST-001は、
React Query等による
クライアントキャッシュの
対象としてよい。

ただし、
最新の確定済み月末資産状況が
変更されると、
AST-001のレスポンスも変化する。

そのため、
月末資産状況の確定・確定解除後は、
必要に応じて
現在資産状況のQueryを
invalidateする。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey:
    assetSummaryKeys.current,
});
```

---

#### 28.23 SNP-004との連携

SNP-004 月末資産状況確定APIが
成功した場合は、
AST-001の取得対象年月が
変化する可能性がある。

例えば、

```text
確定前
2026-06 confirmed = true
2026-07 confirmed = false

    ↓ SNP-004

確定後
2026-07 confirmed = true
```

となった場合は、
AST-001を再取得する。

```text
targetYearMonth
2026-06
    ↓
2026-07
```

へ更新される。

---

#### 28.24 SNP-005との連携

SNP-005 月末資産状況確定解除APIによって、
現在資産状況として使用していた
最新年月が未確定になった場合は、
AST-001の対象年月が
1つ前の確定済み年月へ
変化する可能性がある。

そのため、
SNP-005成功後も
AST-001のQueryを
invalidateする。

---

#### 28.25 BAL・VAL系APIとの連携

BAL系およびVAL系APIは、
未確定の月末資産状況に対する
残高・評価額の登録更新を行う。

AST-001は
確定済み月末資産状況のみを
対象とするため、
未確定月のBAL・VAL更新だけでは
AST-001の結果は変化しない。

```text
BAL / VAL更新
    ↓
対象snapshotが未確定
    ↓
AST-001には反映されない

SNP-004で確定
    ↓
AST-001再取得
    ↓
反映される
```

この関係を
フロントエンドでも前提とする。

---

#### 28.26 AST-002との使い分け

AST-001は、
最新の確定済み対象年月を
自動的に取得する。

過去の特定年月を
表示したい場合は、
AST-002 指定年月資産状況取得APIを使用する。

```text
現在の資産状況
    → AST-001

2026-05の資産状況
    → AST-002
```

フロントエンド側で
AST-001を利用して
過去月を再現しない。

---

#### 28.27 AST-003との使い分け

AST-001は、
1つの対象年月について
資産状況の詳細を取得する。

複数月の推移を表示する場合は、
AST-003 資産推移取得APIを使用する。

```text
最新月の内訳
    → AST-001

複数月のグラフ
    → AST-003
```

AST-001を
月数分繰り返し呼び出して
資産推移を生成しない。

---

#### 28.28 フロントエンドで総資産を再計算しない

APIレスポンスには、

```text
totalAssets
assetAccounts[].assetAmount
```

の両方が含まれる。

フロントエンドでは、
表示時に

```ts
const totalAssets =
  data.assetAccounts.reduce(
    (sum, account) =>
      sum + account.assetAmount,
    0,
  );
```

のように
総資産を再計算して
APIの`totalAssets`を
置き換えない。

バックエンドで算出された
`totalAssets`を正式値として扱う。

資産口座別合計との一致確認は、
バックエンドテストで保証する。

---

#### 28.29 フロントエンドでavailableAssetsを再計算しない

同様に、

```ts
const availableAssets =
  data.assetAccounts
    .filter(
      (account) =>
        account.available,
    )
    .reduce(
      (sum, account) =>
        sum + account.assetAmount,
      0,
    );
```

によって
APIレスポンスの
`availableAssets`を
置き換えない。

APIから返却された
`availableAssets`を
正式値として表示する。

---

### 29 設計上の補足

#### 29.1 currentをURLに含める理由

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

#### 29.2 targetYearMonthを受け取らない理由

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

#### 29.3 最新の月末資産状況ではなく最新の確定済みを使用する理由

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

#### 29.4 確定済み月末資産状況がない場合に0円を返さない理由

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

#### 29.5 残高記録単位によって算出元を切り替える理由

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

#### 29.6 availableAssetsをtotalAssetsと分ける理由

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

#### 29.7 対象年月時点の利用可能資産設定を使う理由

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

#### 29.8 集計値を保存しない理由

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

#### 29.9 APIレスポンスで残高記録単位を返さない理由

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

#### 29.10 holdingAssetsを常に配列で返す理由

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

#### 29.11 GETを採用する理由

AST-001は、
現在資産状況を
参照するだけのAPIである。

データの登録、
更新または削除を行わないため、
HTTPメソッドには
`GET`を採用する。

---

#### 29.12 AST-001が冪等である理由

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

#### 29.13 Idempotency-Keyを使用しない理由

AST-001は、
読み取り専用のGET APIである。

再実行によって
データの重複登録などが
発生しないため、
`Idempotency-Key`は使用しない。

---

#### 29.14 サーバーキャッシュを採用しない理由

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

### 30 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)