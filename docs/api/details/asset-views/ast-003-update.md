##  AST-003 資産推移取得

### 1 概要

操作対象となる利用者について、
指定期間内の
確定済み月末資産状況をもとに、
資産推移を取得する。

本APIでは、
複数の`month_end_asset_snapshots`を対象として、
対象年月ごとの

- 総資産
- 利用可能資産
- 資産口座別資産額
- 保有商品別資産額

を時系列で取得する。

取得対象となる
月末資産状況は、
操作対象利用者に属し、
`confirmed = true`であるものに限定する。

概念的には、
以下の条件で対象年月を取得する。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.confirmed
    = true

AND

target_year_month
    = 指定期間内
```

資産額の算出では、
各対象年月について
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
重複して集計しない。

利用可能資産は、
各`target_year_month`時点の
`asset_account_available_settings`をもとに
算出する。

現在時点の設定だけを使用して
過去月の利用可能資産を
再計算してはならない。

本APIは、
資産推移を参照するための
読み取り専用APIである。

資産推移の算出結果を
専用テーブルへ保存せず、
既存の月末資産データから
取得時に算出する。

---

### 2 ユースケース

利用者は、
一定期間における
資産状況の変化を確認する。

例えば、
以下のような場合に使用する。

- 総資産の月次推移をグラフで確認する
- 利用可能資産の月次推移を確認する
- 前月と比較して資産が増減したか確認する
- 資産口座別の推移を確認する
- 保有商品別の推移を確認する
- 過去から現在までの資産形成状況を確認する
- 資産推移グラフから特定年月を選択し、AST-002でその月の詳細を確認する

例えば、
以下の確定状況の場合、

```text
2026-01
confirmed = true

2026-02
confirmed = true

2026-03
confirmed = false

2026-04
confirmed = true
```

`2026-01`から
`2026-04`を対象として
AST-003を実行した場合、
推移対象となるのは、

```text
2026-01
2026-02
2026-04
```

とする。

未確定の`2026-03`は、
資産推移へ含めない。

未確定月を
0円として補完してはならない。

---

### 3 エンドポイント

```http
GET /api/v1/asset-trends
```

---

### 4 HTTPメソッド

```http
GET
```

本APIは、
指定期間内の資産推移を取得する
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

資産推移の対象となる
月末資産状況を取得する際は、
必ず操作対象利用者によって
絞り込む。

概念的な条件は、
以下とする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.confirmed
    = true
```

さらに、
指定された表示対象期間に
該当する`target_year_month`のみを
取得する。

概念的には、

```text
操作対象利用者
    ↓
確定済み月末資産状況
    ↓
指定期間内のtarget_year_month
    ↓
資産推移
```

とする。

他の利用者に属する
`month_end_asset_snapshots`を
推移対象へ含めてはならない。

例えば、

```text
User A
2026-01 confirmed = true
2026-02 confirmed = true

User B
2026-03 confirmed = true
```

の状態で、
User Aを操作対象として
AST-003を実行した場合、
User Bの`2026-03`を
User Aの資産推移へ
含めてはならない。

月末資産残高についても、
操作対象利用者に属する
月末資産状況および
資産口座に紐づくデータのみを
集計対象とする。

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
集計対象とする。

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

各対象年月について、
その年月時点で有効な
利用可能資産設定を使用する。

本APIでは、
他の利用者に属する

- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品
- 利用可能資産設定

を資産推移の算出へ
含めてはならない。

`X-User-Id`が
指定されていない場合、
形式が不正な場合、
または指定された利用者が
存在しない場合は、
API共通方針に従って
エラーを返却する。

また、
指定期間内に
他の利用者には
確定済み月末資産状況が存在していても、
その存在を
レスポンスから推測できないようにする。

---

### 6 パスパラメータ

なし。

本APIでは、
パスパラメータを使用しない。

資産推移の表示対象期間は、
クエリパラメータで指定する。

---

### 7 クエリパラメータ

本APIでは、
資産推移の表示対象期間を
クエリパラメータで指定する。

| パラメータ名 | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `from` | string | ○ | 表示対象期間の開始年月 |
| `to` | string | ○ | 表示対象期間の終了年月 |

`from`および`to`の形式は、

```text
YYYY-MM
```

とする。

例：

```http
GET /api/v1/asset-trends?from=2026-01&to=2026-06
```

この場合、
`2026-01`から
`2026-06`までを
表示対象期間とする。

開始年月および終了年月は、
両方を含む。

```text
from = 2026-01
to   = 2026-06

対象期間
2026-01
2026-02
2026-03
2026-04
2026-05
2026-06
```

ただし、
実際に資産推移へ含めるのは、
対象期間内に存在する
確定済み月末資産状況のみとする。

---

#### 7.1 from

`from`は、
資産推移の
表示対象期間の開始年月を表す。

```text
YYYY-MM
```

形式で指定する。

例：

```text
from=2026-01
```

---

#### 7.2 to

`to`は、
資産推移の
表示対象期間の終了年月を表す。

```text
YYYY-MM
```

形式で指定する。

例：

```text
to=2026-06
```

---

#### 7.3 指定しないクエリパラメータ

Phase1では、
以下の条件を
クエリパラメータとして指定しない。

- 利用者ID
- 資産口座ID
- 保有商品ID
- 残高記録単位
- 確定状態
- 利用可能資産のみを取得するか
- 資産種別

AST-003では、
操作対象利用者について、
指定期間内の
資産推移全体を取得する。

---

### 8 リクエストヘッダー

#### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/asset-trends?from=2026-01&to=2026-06
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

資産推移の取得に必要な情報は、

- `from`
- `to`
- `X-User-Id`

から取得する。

クライアントから、
以下のような
集計結果や業務データを
受け取らない。

- 総資産
- 利用可能資産
- 資産口座別資産額
- 保有商品別資産額
- 確定状態
- 利用可能資産設定

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
| `from` | クエリパラメータ | string | ○ | 表示対象期間の開始年月 |
| `to` | クエリパラメータ | string | ○ | 表示対象期間の終了年月 |
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
操作対象利用者と
指定期間をもとに、
サーバー側で取得する。

---

### 11 バリデーション

#### 11.1 from 必須チェック

`from`は、
必須のクエリパラメータとする。

未指定の場合は、
バリデーションエラーとする。

不正例：

```http
GET /api/v1/asset-trends?to=2026-06
```

---

#### 11.2 to 必須チェック

`to`は、
必須のクエリパラメータとする。

未指定の場合は、
バリデーションエラーとする。

不正例：

```http
GET /api/v1/asset-trends?from=2026-01
```

---

#### 11.3 from 形式チェック

`from`は、
以下の形式であることを検証する。

```text
YYYY-MM
```

正常例：

```text
2026-01
2026-06
2026-12
```

不正例：

```text
2026-1
26-01
2026/01
202601
2026-00
2026-13
abc
```

年4桁、
ハイフン、
月2桁の形式とし、
月は`01`から`12`の
有効な値であることを確認する。

---

#### 11.4 to 形式チェック

`to`についても、
以下の形式であることを検証する。

```text
YYYY-MM
```

正常例：

```text
2026-01
2026-06
2026-12
```

不正例：

```text
2026-1
26-06
2026/06
202606
2026-00
2026-13
abc
```

文字列形式だけではなく、
年月として有効であることを確認する。

---

#### 11.5 fromとtoの前後関係

`from`は、
`to`以前であることを
必須とする。

正常例：

```text
from = 2026-01
to   = 2026-06
```

同一年月も許可する。

```text
from = 2026-06
to   = 2026-06
```

不正例：

```text
from = 2026-07
to   = 2026-06
```

この場合は、
バリデーションエラーとする。

概念的には、

```text
from <= to
```

を満たす必要がある。

---

#### 11.6 同一年月の指定

`from`と`to`が
同一年月であっても、
正常な指定として扱う。

例えば、

```http
GET /api/v1/asset-trends?from=2026-05&to=2026-05
```

を許可する。

この場合、
`2026-05`に
確定済み月末資産状況が存在すれば、
1か月分の資産推移を返却する。

---

#### 11.7 表示対象期間

表示対象期間は、
`from`と`to`を
両端を含む期間として扱う。

```text
from = 2026-01
to   = 2026-03
```

の場合、

```text
2026-01
2026-02
2026-03
```

を対象とする。

---

#### 11.8 対象期間内の確定済み月末資産状況

指定期間内であっても、
資産推移へ含めるのは、

```text
confirmed = true
```

の月末資産状況のみとする。

例えば、

```text
2026-01
confirmed = true

2026-02
confirmed = false

2026-03
confirmed = true
```

の場合、
取得対象は、

```text
2026-01
2026-03
```

とする。

`2026-02`は
取得対象から除外する。

---

#### 11.9 未確定月を0円として補完しない

対象期間内に
未確定の月末資産状況が存在していても、
その年月を

```text
totalAssets = 0
availableAssets = 0
```

として
資産推移へ追加してはならない。

例えば、

```text
2026-01 confirmed = true
2026-02 confirmed = false
2026-03 confirmed = true
```

の場合、

```text
2026-01
2026-03
```

のみを返却する。

以下のようにはしない。

```text
2026-01
2026-02 ← 0円として補完
2026-03
```

---

#### 11.10 データが存在しない年月を0円として補完しない

対象期間内に
月末資産状況自体が
存在しない年月についても、
0円のデータを
生成してはならない。

例えば、

```text
2026-01 confirmed = true
2026-02 データなし
2026-03 confirmed = true
```

の場合、

```text
2026-01
2026-03
```

のみを
資産推移の対象とする。

---

#### 11.11 X-User-Id

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

#### 11.12 利用者境界

対象となる
月末資産状況は、
以下の条件で取得する。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.confirmed
    = true

AND

month_end_asset_snapshots.target_year_month
    >= from

AND

month_end_asset_snapshots.target_year_month
    <= to
```

他の利用者に属する
同一対象年月の
月末資産状況を
取得してはならない。

---

#### 11.13 月末資産残高の扱い

各対象年月について、
口座単位で
残高を記録する資産口座では、

```text
month_end_asset_balances.balance
```

を資産額として使用する。

商品単位で
評価額を記録する資産口座では、
月末資産残高を
総資産へ重複加算しない。

---

#### 11.14 商品別月末評価額の扱い

各対象年月について、
商品単位で
評価額を記録する資産口座では、

```text
month_end_holding_values.value
```

を資産額として使用する。

同一資産について、

```text
month_end_asset_balances.balance
+
month_end_holding_values.value
```

のように
重複して資産推移へ
加算してはならない。

---

#### 11.15 各年月の利用可能資産設定

利用可能資産は、
各対象年月時点で有効な
`asset_account_available_settings`を
使用して算出する。

例えば、

```text
2026-01
口座A available = true

2026-02
口座A available = true

2026-03
口座A available = false
```

の場合、
各年月について
それぞれの設定を反映する。

```text
2026-01
→ available = true

2026-02
→ available = true

2026-03
→ available = false
```

現在時点の設定を
全対象年月へ一律適用してはならない。

---

#### 11.16 他利用者データの除外

資産推移の算出では、
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

同一対象年月を持つ
他利用者のデータが存在していても、
資産推移へ混入させない。

---

#### 11.17 0円の扱い

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

確定済み月末資産状況が存在し、
その資産額が0円の場合は、
その年月を
資産推移から除外しない。

```text
0円
≠
データ不存在
```

とする。

---

#### 11.18 対象年月の並び順

資産推移は、
`target_year_month`の
昇順で扱う。

概念的には、

```text
2026-01
2026-02
2026-03
...
```

の順とする。

データベースからの
取得順序に依存せず、
明示的に
対象年月の昇順を保証する。

---

#### 11.19 前月比較における欠損月

前月比較を行う場合、
「前月」は
直前の確定済みデータではなく、
原則として
暦上の前月を意味する。

例えば、

```text
2026-01 confirmed = true
2026-02 データなし
2026-03 confirmed = true
```

の場合、
`2026-03`について
`2026-01`を
前月として扱わない。

`2026-02`の確定済みデータが
存在しないため、
`2026-03`の前月比較は
算出不可として扱う。

具体的なレスポンス上の表現は、
「業務ルール」および
「レスポンス項目」で定義する。

---

#### 11.20 指定期間内に確定済みデータが存在しない場合

指定された期間内に、
操作対象利用者の
確定済み月末資産状況が
1件も存在しない場合は、
入力値バリデーションエラーとはしない。

```text
from
to
```

が正常であれば、
入力値としては有効である。

確定済み資産推移データが
存在しない場合の扱いは、
業務状態として
「業務ルール」および
「エラーレスポンス」で定義する。

---

#### 11.21 業務状態に依存する検証

以下は、
単項目バリデーションではなく、
資産推移取得の
業務ルールとして扱う。

- 確定済み月末資産状況のみを対象とすること
- 未確定月を0円として補完しないこと
- データ不存在月を0円として補完しないこと
- 残高記録単位に応じて集計対象を切り替えること
- 月末資産残高と商品別月末評価額を重複加算しないこと
- 各対象年月時点の利用可能資産設定を使用すること
- 他の利用者に属する資産情報を集計しないこと
- 対象年月を昇順で返却すること
- 暦上の前月データが存在しない場合は前月比較を算出しないこと

これらの業務ルールは、
FormRequestなどの
入力値バリデーションではなく、
Serviceおよび
資産推移集計処理で保証する。

---

### 12 業務ルール

#### 12.1 指定期間内の資産推移を取得する

本APIでは、
クエリパラメータの
`from`から`to`までの期間について、
資産推移を取得する。

取得対象となる
月末資産状況は、
以下の条件をすべて満たすものとする。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.confirmed
    = true

AND

month_end_asset_snapshots.target_year_month
    >= from

AND

month_end_asset_snapshots.target_year_month
    <= to
```

`from`および`to`は、
両端を含む。

---

#### 12.2 確定済み月末資産状況のみを対象とする

資産推移へ含めるのは、
`confirmed = true`である
月末資産状況のみとする。

例えば、

```text
2026-01
confirmed = true

2026-02
confirmed = false

2026-03
confirmed = true
```

の場合、
資産推移へ含めるのは、

```text
2026-01
2026-03
```

とする。

未確定の`2026-02`は
取得対象としない。

---

#### 12.3 未確定月を0円として補完しない

対象期間内に
未確定の月末資産状況が存在していても、
その年月を
0円の資産状況として
生成してはならない。

以下のような
補完は行わない。

```text
2026-01
totalAssets = 1000000

2026-02
confirmed = false
    ↓
totalAssets = 0として補完

2026-03
totalAssets = 1200000
```

実際のレスポンスには、

```text
2026-01
2026-03
```

のみを含める。

---

#### 12.4 データ不存在月を0円として補完しない

対象期間内に
`month_end_asset_snapshots`自体が
存在しない年月についても、
0円のデータを
自動生成してはならない。

例えば、

```text
2026-01
confirmed = true

2026-02
データなし

2026-03
confirmed = true
```

の場合、
資産推移へ含めるのは、

```text
2026-01
2026-03
```

のみとする。

---

#### 12.5 対象年月の並び順

資産推移は、
`target_year_month`の
昇順で返却する。

```text
2026-01
2026-02
2026-03
...
```

データベースの
暗黙的な取得順序には依存しない。

概念的には、

```text
ORDER BY target_year_month ASC
```

とする。

---

#### 12.6 資産額の算出単位

各対象年月について、
資産口座の
残高記録単位に応じて、
資産額の算出元を切り替える。

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

```text
対象年月
    ↓
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

とする。

---

#### 12.7 二重計上を行わない

同一資産について、
月末資産残高と
商品別月末評価額を
重複して加算してはならない。

商品単位で管理する資産口座について、

```text
month_end_asset_balances.balance
+
SUM(month_end_holding_values.value)
```

とはしない。

残高記録単位に応じて、
使用する資産額を
一意に決定する。

---

#### 12.8 各年月の総資産

各対象年月の総資産は、
その年月における
各資産口座の資産額を
合計して算出する。

概念的には、

```text
totalAssets
=
口座単位資産額の合計
+
商品単位資産額の合計
```

とする。

各年月について、
独立して算出する。

---

#### 12.9 各年月の利用可能資産

利用可能資産は、
各対象年月時点の
`asset_account_available_settings`を
もとに算出する。

例えば、

```text
2026-01
口座A available = true

2026-02
口座A available = true

2026-03
口座A available = false
```

の場合、
各対象年月について
その年月時点の設定を使用する。

現在時点の設定を
過去のすべての年月へ
一律適用してはならない。

---

#### 12.10 資産口座別資産推移

資産口座別資産推移では、
各対象年月について、
資産口座ごとの
資産額を返却する。

口座単位の場合は、

```text
month_end_asset_balances.balance
```

を使用する。

商品単位の場合は、

```text
SUM(month_end_holding_values.value)
```

を使用する。

---

#### 12.11 保有商品別資産推移

商品単位で
評価額を記録する資産口座については、
各対象年月の
保有商品別資産額を返却する。

各保有商品の資産額は、

```text
month_end_holding_values.value
```

を使用する。

口座単位の資産口座について、
架空の商品別推移を
生成してはならない。

---

#### 12.12 0円の資産

確定済み月末資産状況が存在し、
資産額が0円の場合は、
その年月を
資産推移へ含める。

例えば、

```text
2026-02
confirmed = true
totalAssets = 0
```

の場合、
`2026-02`は
正常な推移データとして返却する。

```text
0円
≠
データ不存在
```

とする。

---

#### 12.13 前月比較

各対象年月について、
暦上の前月に
確定済み月末資産状況が存在する場合は、
前月比較を算出する。

概念的には、

```text
currentMonth
    ↓
calendarPreviousMonth
    ↓
確定済みデータあり
    ↓
前月比較算出
```

とする。

例えば、

```text
2026-01 totalAssets = 1000000
2026-02 totalAssets = 1200000
```

の場合、

```text
2026-02
previousMonthDifference
=
1200000 - 1000000
=
200000
```

とする。

---

#### 12.14 前月比較で直前の取得データを使用しない

暦上の前月データが
存在しない場合、
さらに前の確定済みデータを
前月として代用してはならない。

例えば、

```text
2026-01
confirmed = true

2026-02
データなし

2026-03
confirmed = true
```

の場合、
`2026-03`の前月は
`2026-02`である。

`2026-01`を
前月として扱わない。

この場合、
`2026-03`の前月比較は
算出不可とする。

---

#### 12.15 表示対象期間外の前月

`from`で指定された開始年月について、
暦上の前月が
表示対象期間外であっても、
前月比較のために
参照するかどうかは、
Phase1では
表示対象期間内のデータに限定する。

そのため、
例えば、

```text
from = 2026-02
to   = 2026-06
```

の場合、
`2026-01`が
確定済みで存在していても、
`2026-02`の前月比較には使用しない。

`2026-02`は、
取得結果内で最初の年月として
前月比較なしとする。

---

#### 12.16 前月比較の算出対象

前月比較では、
各対象年月について
少なくとも以下を算出対象とする。

- 総資産
- 利用可能資産

資産口座別および
保有商品別の前月差分については、
Phase1では
レスポンスへ含めない。

必要となった場合は、
将来拡張として検討する。

---

#### 12.17 利用者境界

資産推移の算出では、
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

---

#### 12.18 過去月を現在状態で再構成しない

資産推移では、
各対象年月について、
その年月の月末資産データと
その年月時点の
利用可能資産設定を使用する。

現在の

- 月末資産データ
- 利用可能資産設定

のみを使用して
過去月の推移を
再構成してはならない。

---

#### 12.19 集計結果を保存しない

本APIで算出した

- 各年月の総資産
- 各年月の利用可能資産
- 前月差分
- 資産口座別資産額
- 保有商品別資産額

を、
専用の資産推移テーブルへ
保存しない。

既存の月末資産データから
取得時に算出する。

---

### 13 取得条件

資産推移の対象となる
月末資産状況は、
以下の条件で取得する。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID

AND

month_end_asset_snapshots.confirmed
    = true

AND

month_end_asset_snapshots.target_year_month
    >= from

AND

month_end_asset_snapshots.target_year_month
    <= to
```

並び順は、

```text
target_year_month ASC
```

とする。

対象となる
各`month_end_asset_snapshots.id`について、
月末資産残高および
商品別月末評価額を取得する。

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

利用可能資産については、
各`target_year_month`時点の
`asset_account_available_settings`を
参照する。

---

### 14 集計方法

#### 14.1 各年月の総資産

各対象年月について、
以下を算出する。

```text
totalAssets
=
口座単位資産額の合計
+
商品単位資産額の合計
```

---

#### 14.2 各年月の利用可能資産

各対象年月について、
その年月時点で
利用可能資産として扱う
資産口座の資産額を合計する。

```text
availableAssets
=
利用可能な資産口座の
assetAmount合計
```

---

#### 14.3 資産口座別

資産口座ごとに
残高記録単位を判定する。

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

```text
month_end_holding_values.value
```

を資産額として使用する。

---

#### 14.5 総資産の前月差分

暦上の前月データが
取得結果内に存在する場合は、

```text
totalAssetsDifference
=
当月totalAssets
-
前月totalAssets
```

として算出する。

例えば、

```text
2026-01
totalAssets = 1000000

2026-02
totalAssets = 1200000
```

の場合、

```text
totalAssetsDifference = 200000
```

となる。

---

#### 14.6 利用可能資産の前月差分

暦上の前月データが
取得結果内に存在する場合は、

```text
availableAssetsDifference
=
当月availableAssets
-
前月availableAssets
```

として算出する。

---

#### 14.7 前月比較不可

以下の場合は、
前月比較を
算出不可とする。

- 取得結果の先頭年月
- 暦上の前月が取得結果に存在しない
- 暦上の前月が未確定
- 暦上の前月に月末資産状況が存在しない

この場合、
前月差分は`null`とする。

---

#### 14.8 集計値の整合性

各対象年月について、
正常なデータ状態では、

```text
totalAssets
=
SUM(assetAccounts.assetAmount)
```

が成立する。

また、
商品単位で管理する
各資産口座について、

```text
assetAccount.assetAmount
=
SUM(holdingAssets.assetAmount)
```

が成立する。

---

### 15 レスポンス

取得成功時は、
指定期間内の
確定済み月末資産状況をもとにした
資産推移を返却する。

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
    "from": "2026-01",
    "to": "2026-04",
    "trends": [
      {
        "targetYearMonth": "2026-01",
        "totalAssets": 2500000,
        "availableAssets": 1200000,
        "totalAssetsDifference": null,
        "availableAssetsDifference": null,
        "assetAccounts": [
          {
            "assetAccountId": "1",
            "name": "普通預金",
            "assetAmount": 800000,
            "available": true,
            "holdingAssets": []
          },
          {
            "assetAccountId": "2",
            "name": "証券口座",
            "assetAmount": 1700000,
            "available": false,
            "holdingAssets": [
              {
                "holdingAssetId": "10",
                "name": "投資信託A",
                "assetAmount": 1000000
              },
              {
                "holdingAssetId": "11",
                "name": "投資信託B",
                "assetAmount": 700000
              }
            ]
          }
        ]
      },
      {
        "targetYearMonth": "2026-02",
        "totalAssets": 2700000,
        "availableAssets": 1300000,
        "totalAssetsDifference": 200000,
        "availableAssetsDifference": 100000,
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
            "assetAmount": 1800000,
            "available": false,
            "holdingAssets": [
              {
                "holdingAssetId": "10",
                "name": "投資信託A",
                "assetAmount": 1100000
              },
              {
                "holdingAssetId": "11",
                "name": "投資信託B",
                "assetAmount": 700000
              }
            ]
          }
        ]
      },
      {
        "targetYearMonth": "2026-04",
        "totalAssets": 2900000,
        "availableAssets": 1400000,
        "totalAssetsDifference": null,
        "availableAssetsDifference": null,
        "assetAccounts": []
      }
    ]
  }
}
```

上記例では、
`2026-03`の確定済みデータが
存在しないため、
`2026-04`の前月差分は
`null`となる。

---

#### 11 対象期間内にデータが存在しない場合

指定期間内に
確定済み月末資産状況が
1件も存在しない場合は、
エラーとせず、
空配列を返却する。

例：

```json
{
  "data": {
    "from": "2026-01",
    "to": "2026-06",
    "trends": []
  }
}
```

資産推移0件は、
正常な一覧取得結果として扱う。

---

### 16 レスポンス項目

#### 16.1 data

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `from` | string | × | 表示対象期間の開始年月 |
| `to` | string | × | 表示対象期間の終了年月 |
| `trends` | array | × | 対象期間内の資産推移 |

`from`および`to`は、

```text
YYYY-MM
```

形式で返却する。

---

#### 16.2 trends

`trends`の各要素は、
1対象年月分の
資産状況を表す。

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `targetYearMonth` | string | × | 確定済み月末資産状況の対象年月 |
| `totalAssets` | integer | × | 対象年月における総資産 |
| `availableAssets` | integer | × | 対象年月における利用可能資産 |
| `totalAssetsDifference` | integer | ○ | 暦上の前月からの総資産増減額 |
| `availableAssetsDifference` | integer | ○ | 暦上の前月からの利用可能資産増減額 |
| `assetAccounts` | array | × | 資産口座別資産状況 |

`trends`は、
`targetYearMonth`の
昇順で返却する。

---

#### 16.3 totalAssetsDifference

`totalAssetsDifference`は、
暦上の前月からの
総資産増減額を表す。

```text
totalAssetsDifference
=
当月totalAssets
-
前月totalAssets
```

増加した場合は
正数となる。

```json
{
  "totalAssetsDifference": 200000
}
```

減少した場合は
負数となる。

```json
{
  "totalAssetsDifference": -100000
}
```

前月比較を
算出できない場合は、

```json
{
  "totalAssetsDifference": null
}
```

とする。

---

#### 16.4 availableAssetsDifference

`availableAssetsDifference`は、
暦上の前月からの
利用可能資産増減額を表す。

```text
availableAssetsDifference
=
当月availableAssets
-
前月availableAssets
```

前月比較を
算出できない場合は、
`null`とする。

---

#### 16.5 assetAccounts

各対象年月の
資産口座別資産状況は、
以下とする。

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `assetAccountId` | string | × | 資産口座ID |
| `name` | string | × | 資産口座名 |
| `assetAmount` | integer | × | 対象年月における資産口座の資産額 |
| `available` | boolean | × | 対象年月時点で利用可能資産として扱うか |
| `holdingAssets` | array | × | 保有商品別資産状況 |

---

#### 16.6 holdingAssets

商品単位で管理する
資産口座について、
保有商品別資産状況を返却する。

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `holdingAssetId` | string | × | 保有商品ID |
| `name` | string | × | 保有商品名 |
| `assetAmount` | integer | × | 対象年月における商品別月末評価額 |

口座単位で管理する
資産口座の場合は、

```json
"holdingAssets": []
```

とする。

---

#### 16.7 IDの型

以下のIDは、
API共通方針に従って
stringとして返却する。

```text
assetAccountId
holdingAssetId
```

データベース上で
`bigint`として保持していても、
APIレスポンスでは
stringへ変換する。

---

#### 16.8 金額の型

以下の金額項目は、
日本円の整数値として返却する。

- `totalAssets`
- `availableAssets`
- `totalAssetsDifference`
- `availableAssetsDifference`
- `assetAmount`

差分項目については、
負数を許可する。

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

資産推移の表示に
必要な情報のみを返却する。

月末資産データそのものではなく、
それらをもとに算出した
表示用の資産推移を返却する。

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
| `422 Unprocessable Entity` | `VALIDATION_ERROR` | `from`、`to`の形式または前後関係が不正 |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバーエラーが発生した |

指定期間内に
確定済み月末資産状況が
1件も存在しない場合は、
エラーとしない。

正常な0件として、

```text
200 OK
trends = []
```

を返却する。

エラーレスポンス形式は、
API共通方針に従う。

概念例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "from",
        "reason": "invalidFormat",
        "message": "開始年月の形式を確認してください。"
      }
    ]
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

資産推移の
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

他の利用者に属する
資産推移を
代替して返却してはならない。

---

#### 17.4 from未指定

`from`が
指定されていない場合は、

```text
VALIDATION_ERROR
```

を返却する。

例えば、

```http
GET /api/v1/asset-trends?to=2026-06
```

は不正とする。

---

#### 17.5 to未指定

`to`が
指定されていない場合は、

```text
VALIDATION_ERROR
```

を返却する。

例えば、

```http
GET /api/v1/asset-trends?from=2026-01
```

は不正とする。

---

#### 17.6 fromの形式が不正な場合

`from`が
`YYYY-MM`形式ではない場合、
または有効な年月でない場合は、

```text
VALIDATION_ERROR
```

を返却する。

例えば、
以下の場合を含む。

```text
2026-1
26-01
2026/01
202601
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
        "field": "from",
        "reason": "invalidFormat",
        "message": "開始年月の形式を確認してください。"
      }
    ]
  },
  "requestId": "01HXXXXXXXXXXXXXXX"
}
```

---

#### 17.7 toの形式が不正な場合

`to`が
`YYYY-MM`形式ではない場合、
または有効な年月でない場合は、

```text
VALIDATION_ERROR
```

を返却する。

例えば、
以下の場合を含む。

```text
2026-1
26-06
2026/06
202606
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
        "field": "to",
        "reason": "invalidFormat",
        "message": "終了年月の形式を確認してください。"
      }
    ]
  },
  "requestId": "01HXXXXXXXXXXXXXXX"
}
```

---

#### 17.8 fromがtoより後の場合

以下の関係を
満たさない場合は、

```text
from <= to
```

`VALIDATION_ERROR`
を返却する。

例えば、

```text
from = 2026-07
to   = 2026-06
```

は不正とする。

レスポンス例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "from",
        "reason": "invalidRange",
        "message": "開始年月は終了年月以前を指定してください。"
      }
    ]
  },
  "requestId": "01HXXXXXXXXXXXXXXX"
}
```

---

#### 17.9 fromとtoが同一年月の場合

以下は、
正常な指定として扱う。

```text
from = 2026-05
to   = 2026-05
```

この場合、
`2026-05`に
確定済み月末資産状況が存在すれば、
1件分の資産推移を返却する。

存在しない場合は、

```text
200 OK
trends = []
```

とする。

---

#### 17.10 指定期間内に確定済み月末資産状況が存在しない場合

指定期間内に、
操作対象利用者に属する
確定済み月末資産状況が
1件も存在しない場合は、
APIエラーとはしない。

例えば、

```text
from = 2026-01
to   = 2026-03

2026-01
confirmed = false

2026-02
データなし

2026-03
confirmed = false
```

の場合は、

```text
200 OK
```

とする。

レスポンス例：

```json
{
  "data": {
    "from": "2026-01",
    "to": "2026-03",
    "trends": []
  }
}
```

資産推移0件を理由として、
`404 Not Found`を返却しない。

---

#### 17.11 未確定月のみ存在する場合

指定期間内に
月末資産状況は存在するが、
すべて

```text
confirmed = false
```

の場合も、
エラーとしない。

未確定月を除外した結果として、

```text
trends = []
```

を返却する。

---

#### 17.12 他利用者にのみデータが存在する場合

以下の状態を用意する。

```text
User A
指定期間内に確定済みデータなし

User B
指定期間内に確定済みデータあり
```

User Aを操作対象として
AST-003を実行した場合は、

```text
200 OK
trends = []
```

とする。

User Bのデータを
返却してはならない。

また、
User Bに資産推移データが
存在することを
レスポンスから推測できないようにする。

---

#### 17.13 欠損月はエラーとしない

対象期間内に、
確定済み月末資産状況が
存在しない年月が含まれていても、
APIエラーとはしない。

例えば、

```text
2026-01 confirmed = true
2026-02 データなし
2026-03 confirmed = true
```

の場合、

```text
2026-01
2026-03
```

を正常に返却する。

`2026-02`が存在しないことを理由として、
資産推移全体を
エラーにしない。

---

#### 17.14 前月比較不可はエラーとしない

暦上の前月データが
存在しない場合は、
前月比較を
算出できない。

この状態は、
APIエラーとはしない。

例えば、

```text
2026-01 confirmed = true
2026-02 データなし
2026-03 confirmed = true
```

の場合、
`2026-03`について、

```json
{
  "totalAssetsDifference": null,
  "availableAssetsDifference": null
}
```

として返却する。

---

#### 17.15 集計対象データの不整合

確定済み月末資産状況が
存在しているにもかかわらず、
データ不整合によって
正常に資産推移を算出できない場合は、
クライアント起因エラーとして扱わない。

例えば、
アプリケーションが前提とする
月末資産データの整合性が
失われている場合は、
想定外エラーとして扱う。

クライアントへ
内部データ構造や
不整合内容を
直接公開しない。

---

#### 17.16 INTERNAL_SERVER_ERROR

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
| `200 OK` | 資産推移の取得成功 |
| `400 Bad Request` | 利用者コンテキストの指定不備 |
| `404 Not Found` | 指定された利用者が存在しない |
| `422 Unprocessable Entity` | `from`、`to`のバリデーションエラー |
| `500 Internal Server Error` | 想定外のサーバーエラー |

---

#### 18.1 200の扱い

以下の場合は、
`200 OK`
を返却する。

- 指定期間内に複数件の資産推移が存在する
- 指定期間内に1件だけ資産推移が存在する
- 指定期間内に確定済み月末資産状況が0件
- 指定期間内に未確定データしか存在しない
- 対象期間内に欠損月が存在する
- 前月比較を算出できない年月が存在する
- 総資産が0円の年月が存在する
- 利用可能資産が0円の年月が存在する

データ件数や
前月比較可否と、
API処理の成否は
分けて扱う。

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

指定期間内に
資産推移が存在しないことを理由として、
`404 Not Found`を返却しない。

---

#### 18.4 422の扱い

以下の場合は、

```text
422 Unprocessable Entity
```

を返却する。

- `from`が未指定
- `to`が未指定
- `from`の形式が不正
- `to`の形式が不正
- `from`が有効な年月ではない
- `to`が有効な年月ではない
- `from > to`

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
資産推移として算出した

- 各年月の`totalAssets`
- 各年月の`availableAssets`
- `totalAssetsDifference`
- `availableAssetsDifference`
- 資産口座別資産額
- 保有商品別資産額

を、
データベースへ保存しない。

本APIの実行によって、
月末資産状況の
`confirmed`を変更してはならない。

利用可能資産設定についても、
参照のみとする。

未確定月を
自動確定する処理も行わない。

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
指定期間内の
確定済み月末資産状況を取得
    ↓
各snapshotの資産データを取得
    ↓
各対象年月時点の
利用可能資産設定を取得
    ↓
各年月の資産状況を集計
    ↓
前月差分を算出
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
将来的に
長期間の推移取得や
大規模データ集計によって、
複数SELECT間の
厳密な読み取り一貫性が
要求される場合は、
トランザクション分離レベルを含めて
別途検討する。

---

### 21 ロック

本APIでは、
行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

資産推移取得によって、
以下の処理を
不要にブロックしてはならない。

- 月末資産状況の作成
- 月末資産状況の確定
- 月末資産状況の確定解除
- 月末資産残高の登録・更新
- 商品別月末評価額の登録・更新
- 利用可能資産設定の変更

---

### 22 キャッシュ

Phase1では、
AST-003専用の
サーバー側アプリケーションキャッシュは
使用しない。

資産推移は、
指定期間内の
確定済み月末資産データおよび
各対象年月時点の
利用可能資産設定から算出する。

月末資産状況の
確定・確定解除によって、
同じ期間指定でも
取得対象となる年月が
変化する可能性がある。

例えば、

```text
2026-01 confirmed = true
2026-02 confirmed = false
2026-03 confirmed = true
```

の状態では、

```text
2026-01
2026-03
```

が返却される。

その後、

```text
2026-02 confirmed = true
```

となれば、

```text
2026-01
2026-02
2026-03
```

へ変化する。

Phase1では、
キャッシュ無効化処理を複雑化させず、
リクエストごとに
データベースから取得して
算出する。

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
同じ`from`および`to`を指定して
複数回実行しても、
業務データの状態は変化しない。

例えば、

```text
1回目
GET /api/v1/asset-trends?from=2026-01&to=2026-06
    ↓
200 OK

2回目
GET /api/v1/asset-trends?from=2026-01&to=2026-06
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

また、
前月差分などの
集計結果も
データベースへ保存しない。

ただし、
リクエスト間に
別の処理によって

- 新しい月末資産状況が確定された
- 月末資産状況の確定が解除された
- 参照対象となる資産データが変更された

場合は、
同じ`from`および`to`でも
レスポンス内容が
変化する可能性がある。

これは、
AST-003自体の副作用によるものではなく、
参照対象データが
変更されたためである。

本APIはGETであり、
重複登録などの副作用がないため、
`Idempotency-Key`は使用しない。

---

### 24 関連テーブル

#### 24.1 month_end_asset_snapshots

資産推移の対象となる
月末資産状況を取得するために使用する。

本APIでは、
操作対象利用者について、
指定期間内に存在する
確定済み月末資産状況のみを取得する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 月末資産状況ID |
| `user_id` | 利用者境界確認 |
| `target_year_month` | 資産推移の対象年月 |
| `confirmed` | 確定済み月末資産状況のみを対象とする判定 |

取得条件は、
以下とする。

```text
user_id = 操作対象利用者ID
AND
confirmed = true
AND
target_year_month >= from
AND
target_year_month <= to
```

並び順は、
以下とする。

```text
ORDER BY target_year_month ASC
```

未確定の月末資産状況は、
資産推移へ含めない。

本APIでは、
`month_end_asset_snapshots`を更新しない。

---

#### 24.2 month_end_asset_balances

口座単位で
残高を記録する資産口座について、
各対象年月の月末資産残高を
取得するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `month_end_asset_snapshot_id` | 対象となる月末資産状況との関連 |
| `asset_account_id` | 資産口座との関連 |
| `balance` | 口座単位の資産額 |

取得対象は、
AST-003で取得した
各確定済み月末資産状況に
紐づくレコードとする。

```text
month_end_asset_snapshot_id
    IN
指定期間内の確定済みsnapshotId
```

口座単位の資産口座についてのみ、
`balance`を資産額として使用する。

商品単位の資産口座については、
`balance`を
資産推移へ重複加算しない。

本APIでは、
`month_end_asset_balances`を更新しない。

---

#### 24.3 month_end_holding_values

商品単位で
評価額を記録する資産口座について、
各対象年月の商品別月末評価額を
取得するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `month_end_asset_snapshot_id` | 対象となる月末資産状況との関連 |
| `holding_asset_id` | 保有商品との関連 |
| `value` | 商品単位の資産額 |

取得対象は、
AST-003で取得した
各確定済み月末資産状況に
紐づくレコードとする。

```text
month_end_asset_snapshot_id
    IN
指定期間内の確定済みsnapshotId
```

商品単位の資産口座では、
各対象年月について、
同一資産口座に属する
`value`を合計して
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

各対象年月について、
`balance_recording_unit`に応じて
資産額の算出元を切り替える。

```text
口座単位
    → month_end_asset_balances.balance

商品単位
    → SUM(month_end_holding_values.value)
```

他の利用者に属する
資産口座を
資産推移へ含めない。

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

各対象年月時点で、
資産口座を
利用可能資産として扱うかを
判定するために使用する。

資産推移では、
現在時点の設定を
すべての年月へ適用せず、
各`target_year_month`時点で
有効な設定を使用する。

概念的には、

```text
asset_account_id
+
target_year_month
    ↓
対象年月時点の
available判定
```

とする。

例えば、

```text
2026-01
available = true

2026-02
available = true

2026-03
available = false
```

の場合、
各年月について
それぞれの設定を反映する。

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
資産推移取得のために
参照しない。

- `net_incomes`
- `objectives`
- `assessment_histories`

AST-003は、
資産推移表示に
責務を限定する。

---

### 25 関連する機能要件

- 資産推移
  - 指定期間内の確定済み月末資産状況を時系列で取得する
  - 未確定の月末資産状況は推移へ含めない
  - データ不存在月を0円として補完しない
  - 未確定月を0円として補完しない
  - 対象年月を昇順で返却する

- 総資産
  - 各対象年月について総資産を算出する
  - 口座単位の資産口座は月末資産残高を使用する
  - 商品単位の資産口座は商品別月末評価額を使用する
  - 月末資産残高と商品別月末評価額を重複加算しない

- 利用可能資産
  - 各対象年月時点の利用可能資産設定を使用する
  - 利用可能資産として扱う資産口座のみを合計する
  - 現在の利用可能資産設定を過去月へ一律適用しない

- 前月比較
  - 総資産の前月差分を取得できる
  - 利用可能資産の前月差分を取得できる
  - 暦上の前月データが存在しない場合は比較不可とする
  - 欠損月を飛ばして直前の確定済み年月と比較しない
  - 比較不可の場合は差分を`null`とする

- 資産口座別資産推移
  - 各対象年月について資産口座別資産額を取得できる
  - 口座単位と商品単位で算出方法を切り替える

- 保有商品別資産推移
  - 商品単位で管理する資産口座について保有商品別資産額を取得できる
  - 口座単位の資産口座について架空の商品別推移を生成しない

- 利用者境界
  - 操作対象利用者に属する資産情報のみを取得する
  - 他の利用者に属する資産情報を推移へ含めない
  - 他の利用者の同一対象年月データを参照しない

- API共通
  - IDはAPIレスポンス上stringとして扱う
  - JSONフィールド名はcamelCaseとする
  - エラー時は共通エラーレスポンス形式を使用する

具体的な章番号は、
`functional-requirements.md`の
最新定義に従う。

---

### 26 設計上の補足

#### 26.1 fromとtoをクエリパラメータにする理由

AST-003では、
資産推移という集合に対して、
表示対象期間を
検索条件として指定する。

`from`および`to`は、
単一リソースを識別するIDではなく、
一覧の取得範囲を指定する条件である。

そのため、
パスではなく
クエリパラメータを使用する。

```http
GET /api/v1/asset-trends?from=2026-01&to=2026-06
```

---

#### 26.2 fromとtoを必須にする理由

期間を指定しない場合に、
全期間の資産推移を返却すると、
将来的にデータ件数が増加した際、
レスポンスサイズが
不必要に大きくなる可能性がある。

また、
画面側が必要とする期間を
明示できるようにするため、
Phase1では
`from`と`to`を必須とする。

---

#### 26.3 未確定月を除外する理由

未確定の月末資産状況は、
入力途中または
修正途中の可能性がある。

資産推移は、
正式に確定された
月末資産状況の変化を
表示するため、
未確定月は対象外とする。

---

#### 26.4 未確定月を0円で補完しない理由

未確定であることと、
資産額が0円であることは
意味が異なる。

```text
未確定
    → 正式な資産額が決まっていない

0円
    → 正式な資産額が0円
```

そのため、
未確定月を
0円の推移データへ変換しない。

---

#### 26.5 データ不存在月を0円で補完しない理由

月末資産状況が
登録されていない年月も、
0円の資産状況を
意味するものではない。

そのため、
存在しない年月について
架空の資産推移データを生成しない。

---

#### 26.6 前月比較で暦上の前月を使用する理由

資産推移における
「前月比較」は、
直前にデータが存在する月との比較ではなく、
暦上の前月との比較を意味する。

例えば、

```text
2026-01
2026-03
```

のみ存在する場合、
`2026-03`と`2026-01`を比較すると、
実際には2ヶ月分の変化を
「前月比」と表示することになる。

そのため、
暦上の前月データが存在しない場合は、
前月比較不可とする。

---

#### 26.7 前月比較不可をnullで表現する理由

前月データが存在しない場合、

```text
差分 = 0
```

とは限らない。

0を返却すると、

```text
前月と同額だった
```

と誤認される可能性がある。

そのため、

```json
{
  "totalAssetsDifference": null
}
```

として、
比較できない状態を
明示する。

---

#### 26.8 fromより前の月を前月比較に使わない理由

AST-003は、
クライアントが指定した
表示対象期間の推移を返却する。

`from`より前のデータまで
内部的に追加取得すると、
指定期間外のデータが
集計結果へ影響する。

Phase1では、
レスポンスの対象範囲と
前月比較の対象範囲を一致させ、
取得結果内でのみ比較する。

---

#### 26.9 AST-001・AST-002と集計ルールを共通化する理由

AST-001、
AST-002、
AST-003では、
1ヶ月分の資産状況の
算出ルールは同一である。

```text
口座単位
    → balance

商品単位
    → value

総資産
    → 各口座のassetAmount合計

利用可能資産
    → 対象年月時点のavailable判定
```

そのため、
共通の集計ロジックを使用する。

これにより、
同一対象年月について
APIごとに異なる資産額が
返却されることを防止する。

---

#### 26.10 AST-003で一括取得する理由

AST-003では、
複数年月を対象とする。

AST-001やAST-002の
単月取得処理を
対象月数分そのまま呼び出すと、
SQL発行回数が
対象年月数に比例して増加する。

そのため、
対象期間のデータを
まとめて取得し、
アプリケーション上で
年月ごとに集計する。

---

#### 26.11 対象年月時点の利用可能資産設定を使用する理由

利用可能資産の設定は、
年月によって変化する可能性がある。

現在の設定を
過去すべての年月へ適用すると、
過去の利用可能資産推移を
正しく再現できない。

そのため、
各対象年月時点の
設定を使用する。

---

#### 26.12 資産口座別・保有商品別データを含める理由

AST-003では、
総資産推移だけではなく、
資産口座別・保有商品別の
推移表示にも利用する。

そのため、
各対象年月について
資産口座別および
保有商品別の資産状況を返却する。

Phase1の画面で
不要となることが明確になった場合は、
将来的にレスポンス軽量化を
再検討できる。

---

#### 26.13 集計結果を保存しない理由

資産推移は、

- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 利用可能資産設定

から導出できる。

Phase1では、
推移専用テーブルへ
重複して保存しない。

これにより、
元データと
集計済みデータの
同期問題を避ける。

---

#### 26.14 balance_recording_unitを返却しない理由

資産推移の
`assetAmount`は、
バックエンド側で
残高記録単位に応じて
算出済みである。

React側で
資産額の算出方法を
再判定する必要はない。

そのため、
`balance_recording_unit`は
レスポンスへ返却しない。

---

#### 26.15 holdingAssetsを空配列で返す理由

口座単位の資産口座では、
保有商品別資産状況を持たない。

この場合も、

```json
{
  "holdingAssets": []
}
```

として返却する。

`null`との混在を避け、
TypeScriptで
一貫した配列型として扱う。

---

#### 26.16 0件を404としない理由

AST-003は、
指定期間の資産推移を
一覧として取得するAPIである。

利用者および期間指定が
正常であれば、
対象データが0件でも
リソース自体が存在しないとは扱わない。

そのため、

```text
200 OK
trends = []
```

とする。

---

#### 26.17 GETを採用する理由

AST-003は、
資産推移を
参照するだけのAPIである。

データの登録、
更新または削除を行わないため、
HTTPメソッドには
`GET`を採用する。

---

#### 26.18 AST-003が冪等である理由

同一のデータ状態で
同じ期間を指定して
複数回実行しても、
業務データを変更しない。

そのため、
本APIは冪等である。

別APIによって
月末資産状況の確定状態などが
変更された場合に
レスポンスが変化しても、
AST-003自体の
副作用ではない。

---

#### 26.19 Idempotency-Keyを使用しない理由

AST-003は、
読み取り専用のGET APIである。

再実行によって
重複登録などが発生しないため、
`Idempotency-Key`は使用しない。

---

#### 26.20 サーバーキャッシュを採用しない理由

月末資産状況の
確定・確定解除によって、
同じ期間でも
取得対象年月や前月差分が
変化する可能性がある。

Phase1では、
キャッシュ無効化処理を
複雑化させず、
リクエストごとに
最新の確定状態を参照する。

性能要件が生じた場合は、
将来的に期間単位の
キャッシュ戦略を検討する。

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
