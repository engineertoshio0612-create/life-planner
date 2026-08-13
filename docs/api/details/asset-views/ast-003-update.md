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

### 26 テスト観点

#### 26.1 正常系

操作対象利用者について、
指定期間内に
複数の確定済み月末資産状況および
必要な資産データが存在する状態で
AST-003を実行する。

以下を確認する。

- `200 OK`となること
- 指定期間内の確定済み年月のみ返却されること
- `targetYearMonth`の昇順で返却されること
- 各年月の総資産が正しく算出されること
- 各年月の利用可能資産が正しく算出されること
- 資産口座別資産状況が返却されること
- 商品単位の資産口座について保有商品別資産状況が返却されること
- 前月差分が正しく算出されること
- データベースが更新されないこと

---

#### 26.2 fromとtoを含むこと

以下を用意する。

```text
from = 2026-01
to   = 2026-03

2026-01 confirmed = true
2026-02 confirmed = true
2026-03 confirmed = true
```

期待結果：

```text
2026-01
2026-02
2026-03
```

すべてが取得されること。

---

#### 26.3 対象期間外を除外すること

以下を用意する。

```text
from = 2026-02
to   = 2026-04

2026-01 confirmed = true
2026-02 confirmed = true
2026-03 confirmed = true
2026-04 confirmed = true
2026-05 confirmed = true
```

期待結果：

```text
2026-02
2026-03
2026-04
```

のみ返却されること。

---

#### 26.4 未確定月を除外すること

以下を用意する。

```text
2026-01 confirmed = true
2026-02 confirmed = false
2026-03 confirmed = true
```

期待結果：

```text
2026-01
2026-03
```

となること。

`2026-02`を
返却しないこと。

---

#### 26.5 未確定月を0円で補完しないこと

以下を用意する。

```text
2026-01 confirmed = true
2026-02 confirmed = false
2026-03 confirmed = true
```

以下のような
レスポンスにならないことを確認する。

```text
2026-02
totalAssets = 0
availableAssets = 0
```

---

#### 26.6 データ不存在月を0円で補完しないこと

以下を用意する。

```text
2026-01 confirmed = true
2026-02 データなし
2026-03 confirmed = true
```

期待結果：

```text
2026-01
2026-03
```

のみとなること。

---

#### 26.7 指定期間内0件

指定期間内に
確定済み月末資産状況が
1件も存在しない状態とする。

期待結果：

```text
200 OK
trends = []
```

となること。

`404 Not Found`と
ならないこと。

---

#### 26.8 未確定月のみ存在する場合

指定期間内に
未確定の月末資産状況だけを
用意する。

期待結果：

```text
200 OK
trends = []
```

となること。

---

#### 26.9 from未指定

以下を実行する。

```http
GET /api/v1/asset-trends?to=2026-06
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 26.10 to未指定

以下を実行する。

```http
GET /api/v1/asset-trends?from=2026-01
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 26.11 from形式不正

以下のような値を指定する。

```text
2026-1
2026/01
202601
2026-00
2026-13
abc
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 26.12 to形式不正

以下のような値を指定する。

```text
2026-1
2026/06
202606
2026-00
2026-13
abc
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 26.13 fromがtoより後

以下を指定する。

```text
from = 2026-07
to   = 2026-06
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

---

#### 26.14 fromとtoが同一年月

以下を指定する。

```text
from = 2026-05
to   = 2026-05
```

`2026-05`に
確定済み月末資産状況が存在する場合、

期待結果：

```text
200 OK
trends = 1件
```

となること。

---

#### 26.15 targetYearMonth昇順

以下の月末資産状況を
DB上で任意の順序で用意する。

```text
2026-03
2026-01
2026-02
```

期待結果：

```text
2026-01
2026-02
2026-03
```

の順で返却されること。

---

#### 26.16 口座単位の資産口座

各対象年月について、
口座単位の資産口座を用意する。

例えば、

```text
2026-01
balance = 1000000

2026-02
balance = 1100000
```

期待結果：

```text
2026-01
assetAmount = 1000000

2026-02
assetAmount = 1100000
```

となること。

---

#### 26.17 商品単位の資産口座

各対象年月について、
商品単位の資産口座を用意する。

```text
2026-01
商品A = 500000
商品B = 300000

2026-02
商品A = 550000
商品B = 350000
```

期待結果：

```text
2026-01
assetAmount = 800000

2026-02
assetAmount = 900000
```

となること。

---

#### 26.18 商品単位の保有商品別推移

商品単位の資産口座について、
各対象年月の
保有商品別資産状況を確認する。

例えば、

```text
商品A

2026-01
assetAmount = 500000

2026-02
assetAmount = 550000
```

と正しく返却されること。

---

#### 26.19 二重計上しないこと

商品単位の資産口座について、
同一対象年月に

```text
month_end_asset_balances.balance
    = 1000000

month_end_holding_values
    = 500000 + 300000
```

を用意する。

期待結果：

```text
assetAmount = 800000
```

となり、

```text
1800000
```

とならないこと。

---

#### 26.20 複数資産口座の総資産

以下を用意する。

```text
2026-01

口座A
assetAmount = 1000000

口座B
assetAmount = 500000

口座C
assetAmount = 800000
```

期待結果：

```text
totalAssets = 2300000
```

となること。

---

#### 26.21 各年月のtotalAssets整合性

各対象年月について、

```text
totalAssets
=
SUM(assetAccounts.assetAmount)
```

となることを確認する。

---

#### 26.22 各年月の利用可能資産

以下を用意する。

```text
2026-01

口座A
assetAmount = 1000000
available = true

口座B
assetAmount = 2000000
available = false
```

期待結果：

```text
availableAssets = 1000000
```

となること。

---

#### 26.23 対象年月ごとに利用可能設定が変わる場合

同じ資産口座について、
以下を用意する。

```text
2026-01
assetAmount = 1000000
available = true

2026-02
assetAmount = 1100000
available = false
```

以下を確認する。

```text
2026-01
available = true

2026-02
available = false
```

となること。

現在の設定を
両方の年月へ
一律適用しないこと。

---

#### 26.24 総資産の前月差分

以下を用意する。

```text
2026-01
totalAssets = 1000000

2026-02
totalAssets = 1200000
```

期待結果：

```text
2026-01
totalAssetsDifference = null

2026-02
totalAssetsDifference = 200000
```

となること。

---

#### 26.25 総資産が減少した場合

以下を用意する。

```text
2026-01
totalAssets = 1200000

2026-02
totalAssets = 1000000
```

期待結果：

```text
totalAssetsDifference = -200000
```

となること。

---

#### 26.26 利用可能資産の前月差分

以下を用意する。

```text
2026-01
availableAssets = 700000

2026-02
availableAssets = 900000
```

期待結果：

```text
availableAssetsDifference = 200000
```

となること。

---

#### 26.27 欠損月がある場合の前月比較

以下を用意する。

```text
2026-01
confirmed = true
totalAssets = 1000000

2026-02
データなし

2026-03
confirmed = true
totalAssets = 1300000
```

期待結果：

```text
2026-03
totalAssetsDifference = null
availableAssetsDifference = null
```

となること。

`2026-01`との差額を
前月差分として
算出しないこと。

---

#### 26.28 前月が未確定の場合

以下を用意する。

```text
2026-01
confirmed = true

2026-02
confirmed = false

2026-03
confirmed = true
```

期待結果：

```text
2026-03
totalAssetsDifference = null
availableAssetsDifference = null
```

となること。

---

#### 26.29 fromより前の月を前月比較へ使用しないこと

以下を用意する。

```text
2026-01
confirmed = true

2026-02
confirmed = true
```

リクエスト：

```text
from = 2026-02
to   = 2026-02
```

期待結果：

```text
2026-02
totalAssetsDifference = null
availableAssetsDifference = null
```

となること。

`2026-01`を
表示対象期間外から取得して
差分計算へ使用しないこと。

---

#### 26.30 0円の資産を推移へ含めること

確定済み月末資産状況について、

```text
totalAssets = 0
availableAssets = 0
```

となるデータを用意する。

以下を確認する。

- 対象年月が`trends`に含まれること
- `0`として正常に返却されること
- データ不存在扱いにならないこと

---

#### 26.31 0円から増加した場合の前月差分

以下を用意する。

```text
2026-01
totalAssets = 0

2026-02
totalAssets = 100000
```

期待結果：

```text
2026-02
totalAssetsDifference = 100000
```

となること。

---

#### 26.32 口座単位のholdingAssets

口座単位で管理する
資産口座について、
各対象年月のレスポンスを確認する。

期待結果：

```json
{
  "holdingAssets": []
}
```

となること。

---

#### 26.33 他利用者の対象年月を含めないこと

以下を用意する。

```text
User A
2026-01 confirmed = true
2026-03 confirmed = true

User B
2026-02 confirmed = true
```

User Aとして、

```text
from = 2026-01
to   = 2026-03
```

を指定する。

期待結果：

```text
2026-01
2026-03
```

のみとなること。

User Bの`2026-02`を
含めないこと。

---

#### 26.34 他利用者の月末資産残高を含めないこと

User AとUser Bについて、
同一対象年月に
月末資産残高を用意する。

User Aとして
AST-003を実行する。

以下を確認する。

- User Aの残高のみ集計されること
- User Bの残高が`totalAssets`へ加算されないこと

---

#### 26.35 他利用者の商品別月末評価額を含めないこと

User AとUser Bについて、
同一対象年月に
商品別月末評価額を用意する。

User Aとして
AST-003を実行する。

以下を確認する。

- User Aの商品別月末評価額のみ集計されること
- User Bの商品別月末評価額が混入しないこと

---

#### 26.36 他利用者の利用可能資産設定を使用しないこと

User AとUser Bについて、
各対象年月の
利用可能資産設定を用意する。

User Aとして
AST-003を実行する。

以下を確認する。

- User Aの設定のみ使用されること
- User Bの設定によって`availableAssets`が変化しないこと

---

#### 26.37 利用者コンテキスト未指定

`X-User-Id`を
指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

---

#### 26.38 利用者ID形式不正

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

#### 26.39 利用者不存在

存在しない利用者IDを
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 26.40 論理削除済み利用者

論理削除済み利用者を
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 26.41 AST-002との整合性

AST-003で返却された
任意の`targetYearMonth`について、
同じデータ状態で
AST-002を実行する。

例えば、

```text
AST-003
2026-05
totalAssets = 3200000
availableAssets = 1400000
```

の場合、

```http
GET /api/v1/asset-summaries/2026-05
```

を実行する。

以下が一致することを確認する。

- `targetYearMonth`
- `totalAssets`
- `availableAssets`
- 資産口座別資産状況
- 保有商品別資産状況

---

#### 26.42 副作用

AST-003実行前後で、
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

- `confirmed`が変更されないこと
- 推移集計結果が保存されないこと
- 前月差分が保存されないこと
- 利用可能資産設定が変更されないこと

---

#### 26.43 冪等性

同一のデータ状態で、
同じリクエストを
複数回実行する。

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

以下を確認する。

- データベースが変更されないこと
- 同一状態であれば同じ対象年月が返却されること
- 同一状態であれば同じ集計結果となること
- 同一状態であれば同じ前月差分となること

---

#### 26.44 新しい月が確定された後の再取得

最初に、
以下の状態とする。

```text
2026-01 confirmed = true
2026-02 confirmed = false
2026-03 confirmed = true
```

AST-003を実行し、

```text
2026-01
2026-03
```

が返却されることを確認する。

その後、
`2026-02`を確定する。

再度AST-003を実行する。

期待結果：

```text
2026-01
2026-02
2026-03
```

となること。

さらに、
`2026-03`の前月差分が
`2026-02`との比較で
算出されること。

---

#### 26.45 確定解除後の再取得

以下の状態とする。

```text
2026-01 confirmed = true
2026-02 confirmed = true
2026-03 confirmed = true
```

その後、
`2026-02`を確定解除する。

AST-003を再実行する。

期待結果：

```text
2026-01
2026-03
```

となること。

また、

```text
2026-03
totalAssetsDifference = null
availableAssetsDifference = null
```

となること。

---

#### 26.46 レスポンス契約

正常取得時に、
API共通方針で定めた
Envelope形式で返却されること。

概念例：

```json
{
  "data": {
    "from": "2026-01",
    "to": "2026-03",
    "trends": [
      {
        "targetYearMonth": "2026-01",
        "totalAssets": 2500000,
        "availableAssets": 1200000,
        "totalAssetsDifference": null,
        "availableAssetsDifference": null,
        "assetAccounts": []
      },
      {
        "targetYearMonth": "2026-02",
        "totalAssets": 2700000,
        "availableAssets": 1300000,
        "totalAssetsDifference": 200000,
        "availableAssetsDifference": 100000,
        "assetAccounts": []
      }
    ]
  }
}
```

以下を確認する。

- `data`がobjectであること
- `from`が`YYYY-MM`形式であること
- `to`が`YYYY-MM`形式であること
- `trends`がarrayであること
- `targetYearMonth`が`YYYY-MM`形式であること
- `totalAssets`がintegerであること
- `availableAssets`がintegerであること
- `totalAssetsDifference`がintegerまたは`null`であること
- `availableAssetsDifference`がintegerまたは`null`であること
- `assetAccounts`がarrayであること
- `assetAccountId`がstringであること
- `available`がbooleanであること
- `holdingAssets`がarrayであること
- `holdingAssetId`がstringであること
- JSONフィールド名がcamelCaseであること

---

#### 26.47 0件レスポンス契約

指定期間内に
確定済み月末資産状況が
存在しない場合は、

```json
{
  "data": {
    "from": "2026-01",
    "to": "2026-03",
    "trends": []
  }
}
```

となることを確認する。

以下も確認する。

- `data`を`null`にしないこと
- `trends`を`null`にしないこと
- `404 Not Found`としないこと

---

#### 26.48 返却しない情報

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

#### 26.49 エラーレスポンス

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

AST-003では、
Controller、
FormRequest、
Service、
Query、
集計用DTO、
API Resource、
Responderを分離して実装する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
userContextMiddleware
    ↓
FormRequest
    ↓
Controller
    ↓
Service
    ├─ MonthEndAssetSnapshotQuery
    ├─ AssetTrendQuery
    └─ AssetAvailabilityQuery
    ↓
AssetTrend DTO
    ↓
API Resource
    ↓
Responder
```

Controllerへ
期間条件の検証、
対象年月の抽出、
資産集計、
前月差分計算、
利用可能資産判定、
利用者境界確認を
直接記述しない。

---

#### 27.1 Route

### 27 Laravel実装方針

AST-003では、
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
    ├─ AssetTrendQuery
    └─ AssetAvailabilityQuery
    ↓
AssetTrendResult DTO
    ↓
API Resource
    ↓
Responder
```

AST-003の業務フロー制御は、UseCaseへ集約する。

Actionへ、期間条件の判定、対象年月の抽出、資産集計、利用可能資産判定、前月差分計算、利用者境界確認などの業務ロジックを直接記述しない。

---

#### 27.1 Action

HTTPリクエストを受け付け、検証済みの以下の値および利用者コンテキストを取得する。

- `from`
- `to`

資産推移取得UseCaseを呼び出し、取得結果をResponderへ渡す。

概念例：

```php
final class GetAssetTrendAction
{
    public function __invoke(
        GetAssetTrendRequest $request,
        GetAssetTrendUseCase $useCase,
        AssetTrendResponder $responder,
        userContext $userContext,
    ): JsonResponse {
        $result = $useCase->execute(
            userId: $userContext->userId,
            from: $request->validated('from'),
            to: $request->validated('to'),
        );

        return $responder->ok(
            $result,
        );
    }
}
```

Actionでは、以下を行わない。

- `from`の形式検証
- `to`の形式検証
- `from <= to`の判定
- 確定済み月末資産状況の検索
- 月末資産残高の取得
- 商品別月末評価額の取得
- 残高記録単位の判定
- 利用可能資産設定の取得
- 月ごとの総資産集計
- 月ごとの利用可能資産集計
- 前月差分の算出
- 利用者境界の判定
- APIレスポンス形式への変換

---

#### 27.2 Request

クエリパラメータの以下を検証する。

- `from`
- `to`

本APIでは、パスパラメータおよびリクエストボディを使用しない。

概念例：

```php
final class GetAssetTrendRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'from' => [
                'required',
                new YearMonthRule(),
            ],

            'to' => [
                'required',
                new YearMonthRule(),
            ],
        ];
    }

    public function after(): array
    {
        return [
            function (
                Validator $validator,
            ): void {
                $from =
                    $this->input('from');

                $to =
                    $this->input('to');

                if (
                    is_string($from)
                    && is_string($to)
                    && $from > $to
                ) {
                    $validator->errors()
                        ->add(
                            'to',
                            '終了年月は開始年月以降を指定してください。',
                        );
                }
            },
        ];
    }
}
```

`YYYY-MM`形式では、文字列の大小比較でも年月順を比較できる。

ただし、年月比較処理を複数箇所で使用する場合は、Value Objectまたは専用Ruleへ分離してよい。

---

#### 27.3 Requestで行わないこと

Requestでは、以下の業務判定を行わない。

- 指定期間内に確定済み月末資産状況が存在するか
- 各対象年月に月末資産残高が存在するか
- 各対象年月に商品別月末評価額が存在するか
- 利用可能資産設定が存在するか
- 前月比較が可能か
- 資産推移を正常に算出できるか

これらは、入力値の形式検証ではなく、取得対象データの状態に依存するため、UseCaseおよびQueryで扱う。

---

#### 27.4 UseCase

資産推移取得のユースケース処理を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `from`を受け取る
3. `to`を受け取る
4. 指定期間内の確定済み月末資産状況を取得する
 対象snapshotが0件の場合は空の推移結果を生成する
6. snapshotId一覧を取得する
7. 対象期間分の資産データを一括取得する
8. 対象期間分の利用可能資産設定を取得する
9. snapshot単位に資産データをグルーピングする
10. 各年月の資産口座別資産状況を生成する
11. 各年月の総資産を算出する
12. 各年月の利用可能資産を算出する
13. 暦上の前月が取得結果内に存在する場合のみ前月差分を算出する
14. `AssetTrendResult` DTOを生成して返却する

概念的な処理は、以下とする。

```text
操作対象利用者
+
from
+
to
    ↓
確定済みsnapshot一覧取得
    ↓
0件
    → trends = []

1件以上
    ↓
snapshotId一覧
    ↓
対象期間の資産データ一括取得
    ↓
利用可能資産設定一括取得
    ↓
snapshot単位にグルーピング
    ↓
各年月の資産状況生成
    ↓
前月差分算出
    ↓
AssetTrendResult生成
```

UseCaseでは、SQLやEloquent Query Builderを直接組み立てない。

データベースアクセスは、Queryへ委譲する。

---

#### 27.5 MonthEndAssetSnapshotQuery

指定期間内に存在する操作対象利用者の確定済み月末資産状況を取得する。

概念例：

```php
final class MonthEndAssetSnapshotQuery
{
    public function findConfirmedBetween(
        int $userId,
        string $from,
        string $to,
    ): Collection {
        return MonthEndAssetSnapshot::query()
            ->where(
                'user_id',
                $userId,
            )
            ->where(
                'confirmed',
                true,
            )
            ->whereBetween(
                'target_year_month',
                [
                    $from,
                    $to,
                ],
            )
            ->orderBy(
                'target_year_month',
            )
            ->get([
                'id',
                'target_year_month',
            ]);
    }
}
```

検索条件には、必ず操作対象利用者IDを含める。

他の利用者に属する確定済み月末資産状況を取得対象へ含めない。

未確定の月末資産状況も資産推移へ含めない。

---

#### 27.6 対象snapshotが0件の場合

指定期間内に確定済み月末資産状況が1件も存在しない場合は、例外を送出しない。

空の資産推移を正常結果として返却する。

概念例：

```php
if ($snapshots->isEmpty()) {
    return new AssetTrendResult(
        from: $from,
        to: $to,
        trends: [],
    );
}
```

以下のような不存在例外には変換しない。

```text
CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND
```

AST-003は一覧取得APIであるため、0件も正常な取得結果として扱う。

---

#### 27.7 snapshotId一覧

対象snapshotが存在する場合は、資産データを一括取得するため、snapshotId一覧を生成する。

概念例：

```php
$snapshotIds =
    $snapshots
        ->pluck('id')
        ->all();
```

このID一覧を、月末資産残高および商品別月末評価額の一括取得条件として使用する。

---

#### 27.8 AssetTrendQuery

対象期間に必要となる資産データをまとめて取得する。

概念例：

```php
$assetData =
    $this->assetTrendQuery
        ->findBySnapshots(
            userId: $userId,
            snapshotIds: $snapshotIds,
        );
```

Queryでは、主に以下のデータを扱う。

```text
asset_accounts
month_end_asset_balances
holding_assets
month_end_holding_values
```

対象年月ごと、資産口座ごと、保有商品ごとに個別SQLを発行しない。

---

#### 27.9 月末資産残高の一括取得

口座単位で管理する資産口座について、対象snapshot分の月末資産残高をまとめて取得する。

概念例：

```php
$balances =
    MonthEndAssetBalance::query()
        ->join(
            'asset_accounts',
            'asset_accounts.id',
            '=',
            'month_end_asset_balances.asset_account_id',
        )
        ->whereIn(
            'month_end_asset_balances.month_end_asset_snapshot_id',
            $snapshotIds,
        )
        ->where(
            'asset_accounts.user_id',
            $userId,
        )
        ->where(
            'asset_accounts.balance_recording_unit',
            BalanceRecordingUnit::ACCOUNT,
        )
        ->get([
            'month_end_asset_balances.month_end_asset_snapshot_id',
            'asset_accounts.id as asset_account_id',
            'asset_accounts.name as asset_account_name',
            'month_end_asset_balances.balance',
        ]);
```

他の利用者に属する資産口座の月末資産残高を取得しない。

---

#### 27.10 商品別月末評価額の一括取得

商品単位で管理する資産口座について、対象snapshot分の商品別月末評価額をまとめて取得する。

概念例：

```php
$holdingValues =
    MonthEndHoldingValue::query()
        ->join(
            'holding_assets',
            'holding_assets.id',
            '=',
            'month_end_holding_values.holding_asset_id',
        )
        ->join(
            'asset_accounts',
            'asset_accounts.id',
            '=',
            'holding_assets.asset_account_id',
        )
        ->whereIn(
            'month_end_holding_values.month_end_asset_snapshot_id',
            $snapshotIds,
        )
        ->where(
            'asset_accounts.user_id',
            $userId,
        )
        ->where(
            'asset_accounts.balance_recording_unit',
            BalanceRecordingUnit::HOLDING,
        )
        ->get([
            'month_end_holding_values.month_end_asset_snapshot_id',
            'asset_accounts.id as asset_account_id',
            'asset_accounts.name as asset_account_name',
            'holding_assets.id as holding_asset_id',
            'holding_assets.name as holding_asset_name',
            'month_end_holding_values.value',
        ]);
```

他の利用者に属する資産口座および保有商品を取得対象へ含めない。

---

#### 27.11 利用者境界

資産情報の取得では、必ず操作対象利用者との利用者境界を保証する。

月末資産残高では、

```text
month_end_asset_balances
    ↓
asset_accounts
    ↓
asset_accounts.user_id
        = 操作対象利用者ID
```

商品別月末評価額では、

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

#### 27.12 二重計上の防止

AST-003でも、AST-001・AST-002と同じ残高記録単位のルールを使用する。

```text
口座単位
    → month_end_asset_balances.balance

商品単位
    → month_end_holding_values.value
```

以下のような単純合計は行わない。

```text
SUM(balance)
+
SUM(value)
```

同一資産について月末資産残高と商品別月末評価額を二重計上しない。

---

#### 27.13 AssetAvailabilityQuery

利用可能資産設定は、指定期間についてまとめて取得する。

概念例：

```php
$availabilityRows =
    $this->assetAvailabilityQuery
        ->findBetween(
            userId: $userId,
            from: $from,
            to: $to,
        );
```

各対象年月について、その年月時点で有効な`asset_account_available_settings`を判定できる形で取得する。

現在時点の利用可能資産設定を全対象年月へ使い回さない。

---

#### 27.14 利用可能資産設定の利用者境界

利用可能資産設定についても、操作対象利用者に属する資産口座の設定のみを対象とする。

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

他の利用者に属する設定を資産推移の算出へ使用してはならない。

---

#### 27.15 対象年月時点の利用可能資産判定

各推移データでは、その`targetYearMonth`時点で有効な利用可能資産設定を使用する。

例えば、

```text
2026-01
    ↓
2026-01時点の設定

2026-02
    ↓
2026-02時点の設定

2026-03
    ↓
2026-03時点の設定
```

とする。

現在の最新設定だけをすべての年月へ適用してはならない。

---

#### 27.16 AST-001・AST-002との集計ロジック共通化

AST-003における1ヶ月分の資産状況算出ルールは、AST-001・AST-002と同一とする。

```text
口座単位
    → balance

商品単位
    → SUM(value)

総資産
    → assetAccounts.assetAmount合計

利用可能資産
    → available = trueの資産口座合計
```

そのため、資産口座別資産状況を組み立てる処理は、共通のBuilderなどへ切り出してよい。

概念例：

```php
final class AssetSummaryBuilder
{
    public function buildFromLoadedData(
        MonthEndAssetSnapshot $snapshot,
        Collection $balances,
        Collection $holdingValues,
        Collection $availabilityRows,
    ): AssetSummary {
        // DBアクセスを行わず、
        // 渡されたデータから集計する
    }
}
```

AST-003からこのBuilderを使用する場合、Builder内部ではSQLを発行しない。

対象年月ごとにDBアクセスが発生する設計を避ける。

---

#### 27.17 一括取得を優先する

AST-003では、対象年月数に応じてSQL発行回数が増加しない構成を基本とする。

以下のような実装は避ける。

```php
foreach ($snapshots as $snapshot) {
    $summary =
        $this->assetSummaryBuilder
            ->build(
                userId: $userId,
                snapshotId: $snapshot->id,
                targetYearMonth:
                    $snapshot->target_year_month,
            );
}
```

上記Builder内部で毎回Queryを実行すると、対象年月数に比例してSQL発行回数が増加する。

基本的には、

```text
snapshot一覧取得
    ↓
balance一括取得
    ↓
value一括取得
    ↓
利用可能資産設定一括取得
    ↓
メモリ上で年月ごとに集計
```

とする。

---

#### 27.18 月単位へのグルーピング

一括取得した資産データは、`snapshotId`をキーとしてグルーピングする。

概念例：

```php
$balancesBySnapshot =
    $balances->groupBy(
        'month_end_asset_snapshot_id',
    );

$holdingValuesBySnapshot =
    $holdingValues->groupBy(
        'month_end_asset_snapshot_id',
    );
```

利用可能資産設定についても、各`targetYearMonth`で参照しやすい形に事前整理してよい。

その後、`snapshots`の昇順を基準として月ごとの資産状況を生成する。

---

#### 27.19 AssetTrendResult DTO

資産推移全体は、Eloquent ModelやCollectionをそのままResponderへ渡さず、専用DTOとして表現する。

概念例：

```php
final readonly class AssetTrendResult
{
    public function __construct(
        public string $from,
        public string $to,
        /** @var AssetTrendItem[] */
        public array $trends,
    ) {
    }
}
```

---

#### 27.20 AssetTrendItem DTO

1ヶ月分の資産推移は、以下のようなDTOとして表現する。

概念例：

```php
final readonly class AssetTrendItem
{
    public function __construct(
        public string $targetYearMonth,
        public int $totalAssets,
        public int $availableAssets,
        public ?int $totalAssetsDifference,
        public ?int $availableAssetsDifference,
        /** @var AssetAccountSummary[] */
        public array $assetAccounts,
    ) {
    }
}
```

資産口座単位および保有商品単位のDTOは、AST-001・AST-002と共通利用してよい。

---

#### 27.21 月ごとの資産状況生成

各snapshotについて、一括取得済みのデータだけを使用して月ごとの資産状況を生成する。

概念的には、以下とする。

```text
snapshot
    ↓
口座単位データ抽出
    ↓
商品単位データ抽出
    ↓
対象年月のavailable判定
    ↓
assetAccounts生成
    ↓
totalAssets算出
    ↓
availableAssets算出
```

月ごとのDTO生成処理中に追加SQLを発行しない。

---

#### 27.22 総資産の集計

各対象年月の総資産は、その年月の各資産口座の`assetAmount`を合計する。

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

#### 27.23 利用可能資産の集計

各対象年月の利用可能資産は、

```text
available = true
```

となる資産口座の`assetAmount`のみを合計する。

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

利用可能資産集計のためだけに資産額を再取得しない。

---

#### 27.24 前月差分算出

前月差分は、単純に`trends`配列の直前要素との差分として算出してはならない。

「前月」は、暦上の前月を意味する。

例えば、

```text
2026-01
2026-03
```

のみが取得されている場合、`2026-03`の前月として`2026-01`を使用しない。

`2026-02`が存在しないため、`2026-03`の前月差分は`null`とする。

---

#### 27.25 暦上の前月の特定

各`AssetTrendItem`について、`targetYearMonth`から暦上の前月を算出する。

概念例：

```php
$previousYearMonth =
    CarbonImmutable::createFromFormat(
        'Y-m',
        $current->targetYearMonth,
    )
        ->subMonth()
        ->format('Y-m');
```

取得済みの推移データを、

```text
targetYearMonth
    → AssetTrendItem
```

のMapへ変換しておき、暦上の前月が存在するか確認してよい。

---

#### 27.26 前月差分の計算

暦上の前月が取得結果内に存在する場合のみ、差分を算出する。

概念例：

```php
$totalAssetsDifference =
    $previous === null
        ? null
        : $current->totalAssets
            - $previous->totalAssets;

$availableAssetsDifference =
    $previous === null
        ? null
        : $current->availableAssets
            - $previous->availableAssets;
```

前月データが存在しない場合は、`0`ではなく`null`とする。

---

#### 27.27 fromより前のデータを使用しない

前月差分の算出では、AST-003の取得結果内に存在するデータだけを使用する。

例えば、

```text
from = 2026-02
to = 2026-04
```

で、DBに`2026-01`の確定済みデータが存在していても、差分算出のためだけに追加取得しない。

そのため、`2026-02`については、

```text
totalAssetsDifference = null
availableAssetsDifference = null
```

とする。

---

#### 27.28 0円の扱い

0円は、有効な資産額として扱う。

例えば、

```text
2026-01
totalAssets = 0

2026-02
totalAssets = 100000
```

の場合、

```text
totalAssetsDifference = 100000
```

とする。

以下のようなtruthy / falsy判定を使用しない。

```php
if (! $previous->totalAssets) {
    // 0円を比較不可扱いしてしまうため使用しない
}
```

---

#### 27.29 欠損月を補完しない

UseCaseでは、`from`から`to`までの全年月を生成して0円データで補完しない。

以下のような処理は行わない。

```text
2026-01
2026-02
2026-03

2026-02 snapshotなし
    ↓
2026-02を0円で生成
```

確定済みsnapshotとして実際に取得できた年月のみを`trends`へ含める。

---

#### 27.30 並び順

資産推移は、

```text
target_year_month ASC
```

の順序で返却する。

MonthEndAssetSnapshotQueryの取得時点で昇順を保証する。

UseCaseでは、その順序を維持して`AssetTrendItem`を生成する。

Responderやフロントエンドでの再ソートを前提としない。

---

#### 27.31 API Resource

資産推移全体は、専用API Resourceへ変換する。

概念例：

```php
final class AssetTrendResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'from'
                => $this->from,

            'to'
                => $this->to,

            'trends'
                => AssetTrendItemResource::collection(
                    $this->trends,
                ),
        ];
    }
}
```

データベースカラムを直接JSONへ返却しない。

---

#### 27.32 AssetTrendItemResource

1ヶ月分の資産推移は、専用Resourceへ変換する。

概念例：

```php
final class AssetTrendItemResource
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

            'totalAssetsDifference'
                => $this->totalAssetsDifference,

            'availableAssetsDifference'
                => $this->availableAssetsDifference,

            'assetAccounts'
                => AssetAccountSummaryResource::collection(
                    $this->assetAccounts,
                ),
        ];
    }
}
```

資産口座別および保有商品別のResourceは、AST-001・AST-002と共通利用してよい。

---

#### 27.33 返却しない情報

API Resourceでは、資産推移表示に必要な情報だけを返却する。

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

Eloquent ModelやQuery結果をそのままJSON化してはならない。

---

#### 27.34 Responder

Responderは、生成済みの`AssetTrendResult`を受け取り、API共通方針に従ったHTTPレスポンスへ変換する。

概念例：

```php
final class AssetTrendResponder
{
    public function ok(
        AssetTrendResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data' =>
                    new AssetTrendResource(
                        $result,
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

対象snapshotが0件の場合も、

```json
{
  "data": {
    "from": "2026-01",
    "to": "2026-03",
    "trends": []
  }
}
```

として`200 OK`を返却する。

---

#### 27.35 Responderの責務

Responderでは、以下を行わない。

- データベース検索
- `from`、`to`の検証
- 対象snapshot抽出
- 残高記録単位の判定
- 資産口座別集計
- 総資産集計
- 利用可能資産集計
- 前月差分計算
- 利用者境界判定
- 並び順制御

Responderは、生成済みの`AssetTrendResult`をHTTPレスポンスへ変換することに責務を限定する。

---

#### 27.36 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONレスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

#### 27.37 Repository

AST-003では、Repositoryを使用しない。

本APIは読み取り専用APIであり、以下を行わないためである。

- 登録
- 更新
- 削除

データベース参照は、Queryクラスへ集約する。

---

#### 27.38 トランザクション

AST-003は、読み取り専用APIであるため、Phase1では明示的な`DB::transaction()`を使用しない。

以下のような実装は行わない。

```php
DB::transaction(
    function () {
        // 資産推移取得のみ
    },
);
```

複数SELECT間で厳密な読み取り一貫性が必要となる要件が将来的に追加された場合は、トランザクション分離レベルを含めて別途検討する。

---

#### 27.39 ロック

AST-003では、行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

資産推移取得によって、月末資産状況の確定・確定解除などの更新処理を不要にブロックしない。

---

#### 27.40 N+1問題

AST-003では、対象期間に含まれる月数に比例してSQL発行回数が増加しないようにする。

以下のような実装は行わない。

```php
foreach ($snapshots as $snapshot) {
    $balances =
        MonthEndAssetBalance::query()
            ->where(
                'month_end_asset_snapshot_id',
                $snapshot->id,
            )
            ->get();

    $holdingValues =
        MonthEndHoldingValue::query()
            ->where(
                'month_end_asset_snapshot_id',
                $snapshot->id,
            )
            ->get();
}
```

基本的には、

```text
snapshot一覧
    ↓
whereIn(snapshotIds)
    ↓
月末資産残高一括取得
    ↓
商品別月末評価額一括取得
    ↓
利用可能資産設定一括取得
```

とする。

取得後に、アプリケーション上で年月ごとにグルーピングする。

---

#### 27.41 取得カラム

Queryでは、資産推移構築に必要なカラムを中心に取得する。

例えば、月末資産状況では、

```text
id
target_year_month
```

月末資産残高では、

```text
month_end_asset_snapshot_id
asset_account_id
asset_account_name
balance
```

商品別月末評価額では、

```text
month_end_asset_snapshot_id
asset_account_id
asset_account_name
holding_asset_id
holding_asset_name
value
```

を使用する。

レスポンス生成や業務判定に不要なカラムを過剰に取得しない。

---

#### 27.42 キャッシュ

Phase1では、AST-003専用のサーバー側アプリケーションキャッシュを使用しない。

資産推移は、以下によって対象となる年月が変化する可能性がある。

- 月末資産状況の確定
- 月末資産状況の確定解除

そのため、リクエストごとにデータベースから最新の確定済みデータを取得して資産推移を算出する。

---

#### 27.43 例外変換

Laravel内部例外をそのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `from`未指定・形式不正 | `VALIDATION_ERROR` |
| `to`未指定・形式不正 | `VALIDATION_ERROR` |
| `from > to` | `VALIDATION_ERROR` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

指定期間に確定済み月末資産状況が存在しないことは、エラーへ変換しない。

---

#### 27.44 想定外例外

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

#### 27.45 ログ

AST-003では、必要に応じて以下の情報をログコンテキストへ設定する。

```text
requestId
userId
from
to
targetMonthCount
```

以下の具体的な金額情報は、不要にアクセスログへ出力しない。

- 総資産
- 利用可能資産
- 資産口座別残高
- 商品別月末評価額
- 前月差分

---

#### 27.46 テスト実装方針

Laravel側では、Feature Testを中心としてAST-003のAPI契約および資産推移取得処理を確認する。

Feature Testでは、主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `422 Unprocessable Entity`
- `500 Internal Server Error`
- `from`必須
- `to`必須
- `from`の`YYYY-MM`形式
- `to`の`YYYY-MM`形式
- `from <= to`
- 指定期間内の確定済みsnapshotのみ取得されること
- 未確定snapshotが除外されること
- 他利用者のsnapshotが除外されること
- 対象snapshotが0件の場合に`trends = []`となること
- `targetYearMonth`昇順となること
- 欠損月を補完しないこと
- 口座単位資産の集計
- 商品単位資産の集計
- 月末資産残高と商品別月末評価額の二重計上防止
- 対象年月時点の利用可能資産設定を使用すること
- 各年月の`totalAssets`が正しいこと
- 各年月の`availableAssets`が正しいこと
- 暦上の前月が存在する場合に差分が算出されること
- 暦上の前月が存在しない場合に差分が`null`となること
- 欠損月を飛ばして前月比較しないこと
- `from`より前のデータを前月比較へ使用しないこと
- 0円からの差分が正しく算出されること
- AST-001・AST-002と同一対象年月で集計結果が一致すること
- IDがstringとして返却されること
- JSONフィールド名がcamelCaseであること
- 返却対象外項目が含まれないこと
- データベースが更新されないこと
- 冪等性

Requestについては、以下を確認する。

```text
from = 2026-01
to = 2026-12
    → 正常
```

```text
from = 2026-1
from = 2026/01
from = 2026-13

to = 2026-1
to = 2026/01
to = 2026-13
    → VALIDATION_ERROR
```

```text
from = 2026-12
to = 2026-01
    → VALIDATION_ERROR
```

MonthEndAssetSnapshotQueryについては、Database Testで以下を確認する。

```text
user_id一致
+
confirmed = true
+
from <= target_year_month <= to
    ↓
target_year_month ASC
```

AssetTrendQueryについては、以下を確認する。

```text
snapshotId一覧
+
操作対象利用者
    ↓
対象期間の資産情報を
一括取得
```

他利用者の以下のデータが取得結果へ混入しないことを確認する。

- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品

AssetAvailabilityQueryについては、以下を確認する。

```text
操作対象利用者
+
from〜to
    ↓
各対象年月で判定可能な
利用可能資産設定を取得
```

UseCaseについては、一括取得済みデータから期待する`AssetTrendResult`が生成されることをUnit Testする。

概念的には、

```text
snapshot一覧
+
月末資産残高一覧
+
商品別月末評価額一覧
+
利用可能資産設定
    ↓
GetAssetTrendUseCase
    ↓
AssetTrendResult
```

を確認する。

特に、以下をUnit Testする。

```text
totalAssets
availableAssets
totalAssetsDifference
availableAssetsDifference
```

前月差分については、以下を明示的に確認する。

```text
2026-01あり
2026-02あり
    ↓
2026-02は2026-01と比較
```

```text
2026-01あり
2026-02なし
2026-03あり
    ↓
2026-03の差分はnull
```

```text
from = 2026-02
DBには2026-01あり
取得結果は2026-02から
    ↓
2026-02の差分はnull
```

---

### 28 React・TypeScriptでの利用

本APIでは、
クエリパラメータとして
`from`および`to`を使用する。

クエリパラメータの型は、
以下とする。

```ts
export type GetAssetTrendQuery = {
  from: string;
  to: string;
};
```

資産口座別・保有商品別の型は、
AST-001およびAST-002と
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
```

1ヶ月分の
資産推移データの型は、
以下とする。

```ts
export type AssetTrendItem = {
  targetYearMonth: string;
  totalAssets: number;
  availableAssets: number;
  totalAssetsDifference: number | null;
  availableAssetsDifference: number | null;
  assetAccounts: AssetAccountSummary[];
};
```

資産推移全体の型は、
以下とする。

```ts
export type AssetTrend = {
  from: string;
  to: string;
  trends: AssetTrendItem[];
};

export type GetAssetTrendResponse = {
  data: AssetTrend;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetAssetTrendResponse>(
    '/api/v1/asset-trends',
    {
      params: {
        from,
        to,
      },
    },
  );
```

資産推移画面などから、
指定期間の資産推移を
グラフまたは一覧で
表示する際に利用する。

---

#### 28.1 fromの扱い

`from`は、
表示対象期間の開始年月として扱う。

形式は、

```text
YYYY-MM
```

とする。

例：

```ts
const from = '2026-01';
```

フロントエンド側でも、
基本的にはstringとして扱う。

---

#### 28.2 toの扱い

`to`は、
表示対象期間の終了年月として扱う。

形式は、

```text
YYYY-MM
```

とする。

例：

```ts
const to = '2026-06';
```

---

#### 28.3 表示対象期間の入力UI

利用者が
表示対象期間を変更できる場合は、
開始年月および終了年月を
年月単位で選択できるUIとする。

概念例：

```tsx
<input
  type="month"
  value={from}
  onChange={(event) =>
    setFrom(
      event.target.value,
    )
  }
/>

<input
  type="month"
  value={to}
  onChange={(event) =>
    setTo(
      event.target.value,
    )
  }
/>
```

フロントエンドでも
基本的な入力制御を行ってよいが、
正式なバリデーションは
バックエンドでも必ず実施する。

---

#### 28.4 fromとtoの前後関係

フロントエンドでは、
利用者操作時点で

```text
from <= to
```

となるように
入力制御してよい。

例えば、

```ts
const isInvalidRange =
  from > to;
```

として、
不正な期間では
検索ボタンを無効化してよい。

ただし、
バックエンド側の
バリデーションを省略しない。

---

#### 28.5 Queryとして扱う

AST-003は、
読み取り専用のGET APIであるため、
React Query等では
Queryとして扱う。

概念例：

```ts
export const fetchAssetTrend =
  async ({
    from,
    to,
  }: GetAssetTrendQuery) => {
    const response =
      await apiClient.get<GetAssetTrendResponse>(
        '/api/v1/asset-trends',
        {
          params: {
            from,
            to,
          },
        },
      );

    return response.data;
  };
```

---

#### 28.6 Query Key

Query Keyには、
`from`および`to`を含める。

概念例：

```ts
export const assetTrendKeys = {
  all: [
    'assetTrends',
  ] as const,

  range: (
    from: string,
    to: string,
  ) =>
    [
      ...assetTrendKeys.all,
      from,
      to,
    ] as const,
};
```

これにより、
表示対象期間ごとに
キャッシュを分離できる。

---

#### 28.7 Hook実装例

React Queryを使用する場合の
概念例は、
以下とする。

```ts
export const useAssetTrend =
  (
    from: string,
    to: string,
  ) => {
    return useQuery({
      queryKey:
        assetTrendKeys.range(
          from,
          to,
        ),

      queryFn:
        () =>
          fetchAssetTrend({
            from,
            to,
          }),

      enabled:
        from.length > 0
        && to.length > 0
        && from <= to,
    });
  };
```

実際のAPI Client、
React Queryおよび
キャッシュ方針は、
フロントエンド共通設計に従う。

---

#### 28.8 trendsの扱い

`trends`は、
対象期間内の
確定済み月末資産状況を
`targetYearMonth`昇順で
保持する。

概念例：

```tsx
{data.trends.map(
  (trend) => (
    <AssetTrendRow
      key={trend.targetYearMonth}
      trend={trend}
    />
  ),
)}
```

フロントエンド側で
独自に並べ替えることを
前提としない。

---

#### 28.9 欠損月の扱い

AST-003では、
未確定月または
月末資産状況が存在しない年月は、
`trends`へ含まれない。

例えば、

```text
2026-01
2026-03
```

が返却された場合、
`2026-02`を
フロントエンド側で
0円として補完しない。

以下のような処理は行わない。

```ts
const completedTrends =
  fillMissingMonthsWithZero(
    data.trends,
  );
```

欠損月は、
データ不存在または
未確定であることを意味し、
0円を意味しない。

---

#### 28.10 0円の扱い

以下は、
正常な業務値として扱う。

```text
totalAssets = 0
availableAssets = 0
assetAmount = 0
```

確定済みの対象年月として
`trends`に含まれている場合は、
0円として正常表示する。

```ts
if (
  trend.totalAssets === 0
) {
  // 正常な0円表示
}
```

0円を
データ不存在と混同しない。

---

#### 28.11 totalAssetsの扱い

`totalAssets`は、
対象年月における
総資産額として扱う。

```ts
const totalAssets =
  trend.totalAssets;
```

フロントエンド側で
`assetAccounts`から
再集計して
正式値を置き換えない。

---

#### 28.12 availableAssetsの扱い

`availableAssets`は、
対象年月時点の
利用可能資産額として扱う。

```ts
const availableAssets =
  trend.availableAssets;
```

現在の
利用可能資産設定から
再計算しない。

---

#### 28.13 totalAssetsDifferenceの扱い

`totalAssetsDifference`は、
暦上の前月からの
総資産増減額を表す。

```ts
const difference =
  trend.totalAssetsDifference;
```

正数の場合は、
総資産が増加している。

```text
+200000
```

負数の場合は、
総資産が減少している。

```text
-100000
```

`null`の場合は、
前月比較不可として扱う。

---

#### 28.14 availableAssetsDifferenceの扱い

`availableAssetsDifference`は、
暦上の前月からの
利用可能資産増減額を表す。

```ts
const difference =
  trend.availableAssetsDifference;
```

`null`の場合は、
前月比較不可として扱う。

---

#### 28.15 前月比較不可の表示

差分項目が`null`の場合は、
0円差分として表示しない。

以下のような
表示を行ってよい。

```text
前月比：-
```

概念例：

```tsx
<span>
  {trend.totalAssetsDifference === null
    ? '-'
    : formatYen(
        trend.totalAssetsDifference,
      )}
</span>
```

```text
null
≠
0
```

を明確に区別する。

---

#### 28.16 増減表示

前月差分に応じて、
表示を切り替えてよい。

概念例：

```ts
const getDifferenceLabel =
  (
    difference: number | null,
  ): string => {
    if (difference === null) {
      return '-';
    }

    if (difference > 0) {
      return `+${formatYen(
        difference,
      )}`;
    }

    return formatYen(
      difference,
    );
  };
```

色やアイコンなどの
具体的な表現は、
画面設計に従う。

---

#### 28.17 資産推移グラフ

`trends`を使用して、
総資産の推移グラフを
表示できる。

概念的なデータ変換例：

```ts
const totalAssetChartData =
  data.trends.map(
    (trend) => ({
      targetYearMonth:
        trend.targetYearMonth,

      value:
        trend.totalAssets,
    }),
  );
```

グラフライブラリの選定は、
フロントエンド設計に従う。

---

#### 28.18 利用可能資産推移グラフ

利用可能資産についても、
同様にグラフ表示できる。

```ts
const availableAssetChartData =
  data.trends.map(
    (trend) => ({
      targetYearMonth:
        trend.targetYearMonth,

      value:
        trend.availableAssets,
    }),
  );
```

---

#### 28.19 欠損月のグラフ表示

APIレスポンスに
存在しない年月を
フロントエンド側で
0円として追加しない。

グラフライブラリ上で
欠損期間をどのように
視覚表現するかは、
画面設計で定める。

ただし、
APIデータとして

```text
totalAssets = 0
```

を生成してはならない。

---

#### 28.20 資産口座別推移

各`trend.assetAccounts`を利用して、
資産口座別の
資産推移を表示できる。

例えば、
特定の`assetAccountId`について、
各年月の値を抽出する。

```ts
const accountTrend =
  data.trends.map(
    (trend) => {
      const account =
        trend.assetAccounts.find(
          (item) =>
            item.assetAccountId
            === targetAssetAccountId,
        );

      return {
        targetYearMonth:
          trend.targetYearMonth,

        assetAmount:
          account?.assetAmount
          ?? null,
      };
    },
  );
```

対象年月に
資産口座データが存在しない場合、
0円とみなすか
データなしとみなすかは、
画面要件に従う。

APIレスポンスを
勝手に0円データへ変換しない。

---

#### 28.21 保有商品別推移

商品単位の資産口座について、
`holdingAssets`から
保有商品別推移を
表示できる。

概念例：

```ts
const holdingTrend =
  data.trends.map(
    (trend) => {
      const holding =
        trend.assetAccounts
          .flatMap(
            (account) =>
              account.holdingAssets,
          )
          .find(
            (item) =>
              item.holdingAssetId
              === targetHoldingAssetId,
          );

      return {
        targetYearMonth:
          trend.targetYearMonth,

        assetAmount:
          holding?.assetAmount
          ?? null,
      };
    },
  );
```

---

#### 28.22 金額表示

金額は、
日本円のintegerとして
APIから返却される。

表示時は、
共通の金額フォーマッタを
使用する。

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

差分項目は
負数も許可する。

---

#### 28.23 IDの扱い

以下のIDは、
API共通方針に従って
stringとして扱う。

```text
assetAccountId
holdingAssetId
```

numberへ変換して
業務計算には使用しない。

---

#### 28.24 trendsが0件の場合

`trends`が空配列の場合は、
APIエラーとして扱わない。

```ts
if (
  data.trends.length === 0
) {
  // 資産推移なし表示
}
```

表示例：

```text
指定期間に表示できる
確定済み資産データがありません。
```

必要に応じて、
月末資産管理画面への
導線を表示する。

---

#### 28.25 ローディング表示

AST-003取得中は、
ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return <Loading />;
}
```

未取得状態と
`trends = []`を
区別する。

```text
未取得
≠
取得成功0件
```

とする。

---

#### 28.26 VALIDATION_ERROR

以下の場合は、
`VALIDATION_ERROR`
として扱う。

- `from`未指定
- `to`未指定
- `from`形式不正
- `to`形式不正
- `from > to`

通常の画面操作では
発生しないよう、
フロントエンドでも
入力制御を行う。

ただし、
URL直接操作などに備え、
APIエラー処理も実装する。

---

#### 28.27 USER_NOT_FOUND

`USER_NOT_FOUND`
が返却された場合は、
操作対象利用者を
現在利用できない状態として扱う。

利用者選択画面へ戻すなど、
API共通方針に従って処理する。

---

#### 28.28 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 期間指定の入力エラーとして扱う |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

指定期間内0件は、
エラー表示の対象としない。

---

#### 28.29 自動リトライ

AST-003は、
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
再送しても解消しない
エラーについては、
不要なリトライを行わない。

---

#### 28.30 クライアントキャッシュ

AST-003は、
React Query等による
クライアントキャッシュの
対象としてよい。

`from`および`to`ごとに
キャッシュを分離する。

例えば、

```text
2026-01〜2026-06
```

と、

```text
2026-01〜2026-12
```

は、
別のQuery Keyとして扱う。

---

#### 28.31 SNP-004との連携

SNP-004 月末資産状況確定APIが
成功した場合は、
AST-003の対象データが
増加する可能性がある。

例えば、

```text
確定前

2026-01 true
2026-02 false
2026-03 true
```

から、

```text
確定後

2026-01 true
2026-02 true
2026-03 true
```

となった場合、
AST-003を再取得する。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey:
    assetTrendKeys.all,
});
```

---

#### 28.32 SNP-005との連携

SNP-005 月末資産状況確定解除APIが
成功した場合も、
AST-003の対象年月が
減少する可能性がある。

そのため、
資産推移Queryを
invalidateする。

確定解除された年月は、
再取得後の`trends`から
除外される。

---

#### 28.33 前月差分の再計算

ある年月が
確定または確定解除されると、
隣接する年月の
前月差分も変化する可能性がある。

例えば、

```text
2026-01 true
2026-02 false
2026-03 true
```

では、

```text
2026-03
difference = null
```

となる。

その後、
`2026-02`が確定すると、

```text
2026-03
difference
=
2026-03
-
2026-02
```

となる。

そのため、
SNP-004やSNP-005成功後は、
対象月だけではなく
資産推移Query全体を
再取得する。

---

#### 28.34 BAL・VAL系APIとの連携

BAL系およびVAL系APIによって
未確定月の資産データが
変更されても、
その月が未確定である限り、
AST-003には反映されない。

```text
BAL / VAL更新
    ↓
snapshot未確定
    ↓
AST-003対象外

SNP-004で確定
    ↓
AST-003対象
```

フロントエンドでも、
この業務ルールを前提とする。

---

#### 28.35 AST-002との連携

資産推移グラフから
特定年月を選択した場合は、
AST-002 指定年月資産状況取得APIを
使用して、
その年月の資産状況を
詳細表示できる。

概念例：

```tsx
const handleTrendClick =
  (
    targetYearMonth: string,
  ) => {
    navigate(
      `/assets/${targetYearMonth}`,
    );
  };
```

遷移先では、
AST-002を使用する。

---

#### 28.36 AST-001との使い分け

最新の資産状況のみを
表示したい場合は、
AST-001を使用する。

複数月の資産推移を
表示する場合は、
AST-003を使用する。

```text
最新月の詳細
    → AST-001

特定月の詳細
    → AST-002

複数月の推移
    → AST-003
```

AST-001またはAST-002を
複数回呼び出して
資産推移を生成しない。

---

#### 28.37 フロントエンドで前月差分を再計算しない

APIレスポンスには、

```text
totalAssetsDifference
availableAssetsDifference
```

が含まれる。

フロントエンド側で、

```ts
current.totalAssets
-
previous.totalAssets
```

のように再計算して、
API結果を置き換えない。

特に、
欠損月がある場合に
配列上の直前要素と比較すると、
API仕様と異なる結果になる可能性がある。

バックエンドから返却された
差分値を正式値として扱う。

---

#### 28.38 フロントエンドで資産推移を再集計しない

以下について、
APIレスポンスを正式値として扱う。

- `totalAssets`
- `availableAssets`
- `totalAssetsDifference`
- `availableAssetsDifference`
- `assetAccounts[].assetAmount`
- `holdingAssets[].assetAmount`

React側で
月末資産データを別途取得して、
独自の資産推移を
再構成しない。

---

### 29 設計上の補足

#### 29.1 fromとtoをクエリパラメータにする理由

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

#### 29.2 fromとtoを必須にする理由

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

#### 29.3 未確定月を除外する理由

未確定の月末資産状況は、
入力途中または
修正途中の可能性がある。

資産推移は、
正式に確定された
月末資産状況の変化を
表示するため、
未確定月は対象外とする。

---

#### 29.4 未確定月を0円で補完しない理由

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

#### 29.5 データ不存在月を0円で補完しない理由

月末資産状況が
登録されていない年月も、
0円の資産状況を
意味するものではない。

そのため、
存在しない年月について
架空の資産推移データを生成しない。

---

#### 29.6 前月比較で暦上の前月を使用する理由

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

#### 29.7 前月比較不可をnullで表現する理由

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

#### 29.8 fromより前の月を前月比較に使わない理由

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

#### 29.9 AST-001・AST-002と集計ルールを共通化する理由

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

#### 29.10 AST-003で一括取得する理由

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

#### 29.11 対象年月時点の利用可能資産設定を使用する理由

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

#### 29.12 資産口座別・保有商品別データを含める理由

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

#### 29.13 集計結果を保存しない理由

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

#### 29.14 balance_recording_unitを返却しない理由

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

#### 29.15 holdingAssetsを空配列で返す理由

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

#### 29.16 0件を404としない理由

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

#### 29.17 GETを採用する理由

AST-003は、
資産推移を
参照するだけのAPIである。

データの登録、
更新または削除を行わないため、
HTTPメソッドには
`GET`を採用する。

---

#### 29.18 AST-003が冪等である理由

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

#### 29.19 Idempotency-Keyを使用しない理由

AST-003は、
読み取り専用のGET APIである。

再実行によって
重複登録などが発生しないため、
`Idempotency-Key`は使用しない。

---

#### 29.20 サーバーキャッシュを採用しない理由

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

### 30 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)
