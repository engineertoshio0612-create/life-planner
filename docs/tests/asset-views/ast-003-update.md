##  AST-003 資産推移取得

### 1 概要

本ドキュメントでは、AST-003 資産推移取得APIに対するテスト方針および主要なテスト観点を定義する。

AST-003は、操作対象利用者について、`from`および`to`で指定された期間内の確定済み月末資産状況を取得し、対象年月ごとの資産状況および前月差分を時系列で返却する読み取り専用APIである。

テストでは、主に以下が仕様どおり動作することを確認する。

* 指定期間内の確定済み月末資産状況のみ取得されること
* `from`および`to`を含む範囲で取得されること
* 未確定月およびデータ不存在月を0円として補完しないこと
* `targetYearMonth`昇順で返却されること
* 各年月の総資産および利用可能資産が正しく算出されること
* 資産口座別および保有商品別資産状況が正しく返却されること
* 暦上の前月との資産差分が正しく算出されること
* 他利用者のデータが混入しないこと
* API実行による副作用が発生しないこと

```text id="krq5r3"
操作対象利用者
    +
from / to
    ↓
指定期間内の
confirmed = true
month_end_asset_snapshots を取得
    ↓
各対象年月の資産状況を算出
    ↓
暦上の前月との比較
    ↓
targetYearMonth昇順で返却
```

資産額については、各対象年月ごとに、口座単位の資産口座では`month_end_asset_balances.balance`、商品単位の資産口座では`month_end_holding_values.value`を使用し、同一資産を二重計上しないことを確認する。

利用可能資産についても、現在の設定を全期間へ適用するのではなく、各`targetYearMonth`時点の`asset_account_available_settings`が使用されることを検証する。

前月差分は、レスポンス上の直前データではなく、暦上の前月との比較結果であることを重要なテスト観点とする。前月が不存在または未確定の場合や、表示期間外である場合は、`totalAssetsDifference`および`availableAssetsDifference`が`null`となることを確認する。

```text id="d9ozmv"
前月が確定済み
    → 前月との差分を返却

前月が不存在・未確定
    → difference = null

前月がfromより前
    → difference = null
```

また、指定期間内に確定済みデータが存在しない場合はエラーとせず、`200 OK`かつ`trends = []`となることを確認する。一方、0円の確定済み資産状況は正常なデータとして`trends`へ含め、データ不存在と区別する。

入力値については、`from`・`to`の必須性、`YYYY-MM`形式、実在する月であること、および`from <= to`の関係を検証する。

利用者境界については、月末資産状況、月末資産残高、商品別月末評価額および利用可能資産設定のすべてで、操作対象利用者以外のデータが取得・集計されないことを確認する。

さらに、AST-003で返却された各年月の資産状況が、同一データ状態におけるAST-002 指定年月資産状況取得の結果と一致することを確認する。

最後に、GET APIとしてデータベースを更新しないこと、同一データ状態で繰り返し実行しても同じ結果となること、確定・確定解除後の再取得で対象年月および前月差分が正しく変化すること、正常・0件・異常レスポンスがAPI共通方針で定めた契約に従うことを検証する。


---

### 2 テスト観点

#### 2.1 正常系

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

#### 2.2 fromとtoを含むこと

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

#### 2.3 対象期間外を除外すること

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

#### 2.4 未確定月を除外すること

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

#### 2.5 未確定月を0円で補完しないこと

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

#### 2.6 データ不存在月を0円で補完しないこと

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

#### 2.7 指定期間内0件

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

#### 2.8 未確定月のみ存在する場合

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

#### 2.9 from未指定

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

#### 2.10 to未指定

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

#### 2.11 from形式不正

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

#### 2.12 to形式不正

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

#### 2.13 fromがtoより後

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

#### 2.14 fromとtoが同一年月

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

#### 2.15 targetYearMonth昇順

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

#### 2.16 口座単位の資産口座

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

#### 2.17 商品単位の資産口座

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

#### 2.18 商品単位の保有商品別推移

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

#### 2.19 二重計上しないこと

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

#### 2.20 複数資産口座の総資産

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

#### 2.21 各年月のtotalAssets整合性

各対象年月について、

```text
totalAssets
=
SUM(assetAccounts.assetAmount)
```

となることを確認する。

---

#### 2.22 各年月の利用可能資産

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

#### 2.23 対象年月ごとに利用可能設定が変わる場合

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

#### 2.24 総資産の前月差分

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

#### 2.25 総資産が減少した場合

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

#### 2.26 利用可能資産の前月差分

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

#### 2.27 欠損月がある場合の前月比較

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

#### 2.28 前月が未確定の場合

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

#### 2.29 fromより前の月を前月比較へ使用しないこと

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

#### 2.30 0円の資産を推移へ含めること

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

#### 2.31 0円から増加した場合の前月差分

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

#### 2.32 口座単位のholdingAssets

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

#### 2.33 他利用者の対象年月を含めないこと

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

#### 2.34 他利用者の月末資産残高を含めないこと

User AとUser Bについて、
同一対象年月に
月末資産残高を用意する。

User Aとして
AST-003を実行する。

以下を確認する。

- User Aの残高のみ集計されること
- User Bの残高が`totalAssets`へ加算されないこと

---

#### 2.35 他利用者の商品別月末評価額を含めないこと

User AとUser Bについて、
同一対象年月に
商品別月末評価額を用意する。

User Aとして
AST-003を実行する。

以下を確認する。

- User Aの商品別月末評価額のみ集計されること
- User Bの商品別月末評価額が混入しないこと

---

#### 2.36 他利用者の利用可能資産設定を使用しないこと

User AとUser Bについて、
各対象年月の
利用可能資産設定を用意する。

User Aとして
AST-003を実行する。

以下を確認する。

- User Aの設定のみ使用されること
- User Bの設定によって`availableAssets`が変化しないこと

---

#### 2.37 利用者コンテキスト未指定

`X-User-Id`を
指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

---

#### 2.38 利用者ID形式不正

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

#### 2.39 利用者不存在

存在しない利用者IDを
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 2.40 論理削除済み利用者

論理削除済み利用者を
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

---

#### 2.41 AST-002との整合性

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

#### 2.42 副作用

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

#### 2.43 冪等性

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

#### 2.44 新しい月が確定された後の再取得

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

#### 2.45 確定解除後の再取得

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

#### 2.46 レスポンス契約

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

#### 2.47 0件レスポンス契約

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

#### 2.48 返却しない情報

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

#### 2.49 エラーレスポンス

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

### 3 関連ドキュメント

- [AST-003 API詳細設計](../../api/details/asset-views/ast-003-update.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [AST-001 現在資産状況取得 テスト設計](./ast-001-list.md)
- [AST-002 指定年月資産状況取得 テスト設計](./ast-002-create.md)
- [AST-003 Laravelアーキテクチャ設計](../../architecture/laravel/asset-views/ast-003-update.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [AST-003 Reactアーキテクチャ設計](../../architecture/react/asset-views/ast-003-update.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)

