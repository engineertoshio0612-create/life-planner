# VAL-001 商品別月末評価額一覧取得

## 1. 概要

操作対象となる利用者に帰属する指定された月末資産状況について、商品単位で登録された月末評価額の一覧を取得する。

本APIでは、`month_end_asset_snapshots`に紐づく`month_end_holding_values`を取得し、対応する保有商品情報と組み合わせて返却する。

主な取得対象は、以下とする。

* 保有商品ID
* 保有商品名
* 商品種別
* 月末評価額
* 対象年月

概念的には、以下の関係から一覧を取得する。

```text
month_end_asset_snapshots
    ↓
snapshotId
    ↓
month_end_holding_values
    ↓
holding_asset_id
    ↓
holding_assets
```

VAL-001は、商品別月末評価額の参照専用APIとする。

以下の処理は行わない。

* 商品別月末評価額登録
* 商品別月末評価額更新
* 商品別月末評価額削除
* 月末資産状況作成
* 月末資産状況確定
* 月末資産状況確定解除
* 保有商品登録
* 保有商品更新

---

### 1.1 取得対象

VAL-001では、指定した月末資産状況に属する商品別月末評価額を取得する。

概念的には、

```text
snapshotId
    ↓
month_end_asset_snapshots
    ↓
target_year_month
    ↓
month_end_holding_values
    ↓
holding_assets
```

とする。

---

### 1.2 商品単位で管理する資産口座を対象とする

商品別月末評価額は、商品単位で残高を管理する資産口座に属する保有商品について記録される。

そのため、VAL-001で返却されるデータは、概念的に

```text
資産口座
balanceRecordingUnit = HOLDING
    ↓
holding_assets
    ↓
month_end_holding_values
```

という関係を持つ。

口座単位で残高を記録する資産口座については、商品別月末評価額ではなく月末残高APIの対象とする。

---

### 1.3 月末資産状況との関係

`month_end_holding_values`は、特定の月末資産状況に対する商品別評価額を表す。

そのため、VAL-001ではURLの`snapshotId`によって対象年月を特定する。

例えば、

```text
snapshotId = 20

month_end_asset_snapshots
target_year_month = 2026-08
```

の場合、VAL-001は

```text
2026年8月末の商品別評価額
```

を取得する。

---

### 1.4 現在の保有商品一覧APIとは責務を分離する

保有商品APIは、現在管理している保有商品そのものを扱う。

一方、VAL-001は、

```text
特定年月時点の
商品別月末評価額
```

を扱う。

概念的には、

```text
HLD-001
    ↓
保有商品のマスタ情報

VAL-001
    ↓
特定月の評価額情報
```

と責務を分離する。

---

### 1.5 月末残高APIとの責務分離

口座単位で記録する資産については、月末残高APIを使用する。

商品単位で記録する資産については、VAL系APIを使用する。

概念的には、

```text
balanceRecordingUnit = ACCOUNT
    ↓
月末残高API
    ↓
month_end_asset_balances

balanceRecordingUnit = HOLDING
    ↓
商品別月末評価額API
    ↓
month_end_holding_values
```

とする。

---

## 2. ユースケース

利用者は、指定した対象年月について、保有商品ごとの月末評価額を確認するためにVAL-001を使用する。

主な利用例は、以下とする。

* 月末資産状況詳細画面で商品別評価額を表示する
* 商品単位で管理している資産口座の内訳を確認する
* 商品別月末評価額の登録状況を確認する
* 月末資産状況確定前に入力状況を確認する
* 月末資産状況確定後に保存済み評価額を確認する

概念的な画面利用は、以下とする。

```text
SNP-003
月末資産状況詳細取得
    ↓
対象年月・確定状態表示
    ↓
VAL-001
商品別月末評価額一覧取得
    ↓
商品別評価額一覧表示
```

---

### 2.1 月末資産状況詳細画面

月末資産状況詳細画面では、指定した`snapshotId`に対してVAL-001を実行する。

例えば、

```text
2026年8月
月末資産状況
```

を表示する場合、

```http
GET /api/v1/month-end-asset-snapshots/20/holding-values
```

を実行し、2026年8月に紐づく商品別月末評価額を表示する。

---

### 2.2 評価額登録状況の確認

月末資産状況の確定前に、商品単位で管理する保有商品について必要な評価額が登録されているかを画面上で確認できる。

ただし、VAL-001自体では

```text
確定可能
確定不可
```

を最終判定しない。

確定可否判定は、SNP-004 月末資産状況確定APIの責務とする。

---

### 2.3 確定済み月末資産状況の参照

対象月末資産状況が

```text
confirmed = true
```

であっても、VAL-001による参照は可能とする。

確定済みであることを理由に商品別評価額を取得不可とはしない。

---

### 2.4 無効化済み保有商品との関係

対象年月時点で商品別月末評価額が保存されている場合、現在の保有商品が無効化済みであっても、過去の月末評価額として参照対象となる可能性がある。

そのため、VAL-001では現在の利用状態だけを理由として保存済みの過去評価額を除外しない。

具体的な取得条件は、業務ルールで定義する。

---

## 3. エンドポイント

```http
GET /api/v1/month-end-asset-snapshots/{snapshotId}/holding-values
```

`snapshotId`には、商品別月末評価額を取得する月末資産状況IDを指定する。

例：

```http
GET /api/v1/month-end-asset-snapshots/20/holding-values
```

---

### 3.1 月末資産状況配下のリソースとする理由

商品別月末評価額は、単独で存在するデータではなく、特定の月末資産状況に属する。

そのため、

```text
month-end-asset-snapshots
    ↓
holding-values
```

という親子関係をURLへ表現する。

以下のようなトップレベルURLとはしない。

```http
GET /api/v1/month-end-holding-values?snapshotId=20
```

---

### 3.2 snapshotIdをURLで指定する理由

商品別月末評価額一覧は、どの月末資産状況に属するかが取得条件の中心となる。

そのため、`snapshotId`をクエリパラメータではなくパスパラメータとして指定する。

概念的には、

```text
月末資産状況
    ↓
その配下の商品別評価額
```

というリソース階層を表現する。

---

## 4. HTTPメソッド

```text
GET
```

VAL-001は、商品別月末評価額の参照専用APIであるため、`GET`を使用する。

---

### 4.1 副作用を持たせない

VAL-001実行によって、以下の処理を行わない。

```text
INSERT
UPDATE
DELETE
```

主に以下のテーブルを参照するだけとする。

```text
month_end_asset_snapshots
month_end_holding_values
holding_assets
```

---

### 4.2 GETで自動登録しない

商品別月末評価額が未登録の保有商品が存在しても、VAL-001実行時に自動的に

```text
month_end_holding_values
```

へレコードを作成しない。

GET APIへ副作用を持たせない。

---

### 4.3 GETでデータを補正しない

保存済みデータに不整合が存在した場合も、VAL-001で以下の補正処理を行わない。

* 評価額を修正する
* 保有商品を変更する
* 月末資産状況を修正する
* 不足レコードを自動追加する

---

## 5. 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、`X-User-Id`リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

VAL-001では、指定された月末資産状況が操作対象利用者に帰属する場合のみ、商品別月末評価額一覧を取得できる。

---

### 5.1 利用者コンテキスト

`X-User-Id`は、API共通方針に従って検証する。

概念的には、

```text
X-User-Id
    ↓
必須確認
    ↓
形式確認
    ↓
利用者存在確認
    ↓
UserContext設定
    ↓
VAL-001
```

とする。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 5.2 利用者境界

対象となる月末資産状況は、少なくとも以下の条件で取得する。

```text
month_end_asset_snapshots.id
    = snapshotId

AND

month_end_asset_snapshots.user_id
    = 操作対象利用者ID
```

論理削除を採用する場合は、さらに

```text
deleted_at IS NULL
```

相当の条件を含める。

`snapshotId`だけで月末資産状況を取得しない。

---

### 5.3 他利用者の月末資産状況

例えば、

```text
User A
    Snapshot ID = 20

User B
    X-User-Id = 2
```

という状態で、User Bが

```http
GET /api/v1/month-end-asset-snapshots/20/holding-values
```

を実行した場合は、User Aの商品別月末評価額を返却しない。

対象月末資産状況が存在しないものとして扱う。

概念的には、

```text
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND
```

とする。

---

### 5.4 商品別月末評価額にuser_idを持たせない場合

`month_end_holding_values`に直接`user_id`を持たせない設計では、利用者境界を親となる月末資産状況から保証する。

概念的には、

```text
users
    ↓
month_end_asset_snapshots.user_id
    ↓
month_end_asset_snapshots.id
    ↓
month_end_holding_values.snapshot_id
```

とする。

---

### 5.5 holding_assetsから利用者を直接判定しない

保有商品も最終的には利用者に帰属するが、VAL-001の第一の利用者境界は

```text
snapshotId
+
UserContext
```

によって月末資産状況を確定することで保証する。

その後、その月末資産状況に紐づく商品別評価額を取得する。

---

### 5.6 userIdをパスへ含めない

以下のようなURLにはしない。

```http
GET /api/v1/users/{userId}/month-end-asset-snapshots/{snapshotId}/holding-values
```

利用者指定は、

```text
X-User-Id
```

へ統一する。

---

### 5.7 userIdをクエリパラメータで受け付けない

以下のような指定も使用しない。

```text
?userId=1
```

利用者IDの指定経路を複数持たせない。

---

## 6. パスパラメータ

VAL-001では、対象となる月末資産状況を指定するため、以下のパスパラメータを使用する。

| パラメータ        | 型      |  必須 | 説明                      |
| ------------ | ------ | :-: | ----------------------- |
| `snapshotId` | string |  ○  | 商品別月末評価額一覧を取得する月末資産状況ID |

エンドポイントは、以下とする。

```http
GET /api/v1/month-end-asset-snapshots/{snapshotId}/holding-values
```

---

### 6.1 snapshotId

`snapshotId`には、対象となる月末資産状況IDを指定する。

例：

```http
GET /api/v1/month-end-asset-snapshots/20/holding-values
```

APIではIDを文字列として扱うが、値としては正の整数形式を前提とする。

正常例：

```text
1
20
999
```

不正例：

```text
0
-1
abc
1.5
1e3
20abc
```

---

### 6.2 snapshotIdだけで取得しない

`snapshotId`だけでは対象月末資産状況を確定しない。

操作対象利用者IDと組み合わせて取得する。

概念的には、

```text
month_end_asset_snapshots.id
    = snapshotId

AND

month_end_asset_snapshots.user_id
    = 操作対象利用者ID
```

とする。

これにより、他利用者の月末資産状況に属する商品別月末評価額を取得できないようにする。

---

### 6.3 対象年月をパスへ含めない

以下のようなURLにはしない。

```http
GET /api/v1/month-end-asset-snapshots/{snapshotId}/{targetYearMonth}/holding-values
```

対象年月は、`snapshotId`から特定できるためである。

概念的には、

```text
snapshotId
    ↓
month_end_asset_snapshots
    ↓
target_year_month
```

とする。

---

### 6.4 holdingAssetIdをパスへ含めない

VAL-001は商品別月末評価額の一覧取得APIである。

そのため、

```text
holdingAssetId
```

はパスパラメータとして指定しない。

以下のようなURLは、VAL-001では使用しない。

```http
GET /api/v1/month-end-asset-snapshots/{snapshotId}/holding-values/{holdingAssetId}
```

個別取得が必要になった場合は、別APIとして設計する。


---

## 7. クエリパラメータ

なし。

VAL-001では、指定された月末資産状況に属する商品別月末評価額を全件取得するため、クエリパラメータは使用しない。

以下のような絞り込み条件は設けない。

```text
?holdingAssetId=10
?assetAccountId=5
?targetYearMonth=2026-08
?enabled=true
```

対象となる月末資産状況は、パスパラメータの

```text
snapshotId
```

によって特定する。

---

### 7.1 targetYearMonthを受け付けない

対象年月は、`snapshotId`に対応する

```text
month_end_asset_snapshots.target_year_month
```

から取得する。

そのため、

```text
?targetYearMonth=2026-08
```

のようなクエリパラメータは使用しない。

以下のように、同じ意味を持つ指定経路を複数設けない。

```text
snapshotId
+
targetYearMonth
```

---

### 7.2 holdingAssetIdで絞り込まない

VAL-001は一覧取得APIであるため、

```text
?holdingAssetId=10
```

のような個別保有商品指定は行わない。

特定商品の商品別月末評価額だけを取得する要件が必要になった場合は、別APIとして設計する。

---

### 7.3 assetAccountIdで絞り込まない

商品別月末評価額は、保有商品を介して資産口座に紐づくが、VAL-001では

```text
?assetAccountId=5
```

による絞り込みを提供しない。

月末資産状況詳細画面では、指定された月の商品別評価額全体を取得し、必要な表示単位への分類はレスポンス項目および画面設計に従う。

---

### 7.4 ページングを使用しない

Phase1では、商品別月末評価額一覧にページングを導入しない。

以下のようなクエリパラメータは使用しない。

```text
?page=1
?perPage=20
```

1利用者が1か月について管理する保有商品数は大量になることを想定していないため、指定月の一覧を全件返却する。

---

### 7.5 並び順指定を受け付けない

以下のようなソート指定もクエリパラメータでは受け付けない。

```text
?sort=holdingAssetName
?order=asc
```

VAL-001の正式な並び順は、API側で統一する。

具体的な並び順は、後続の「業務ルール」で定義する。

---

## 8. リクエストヘッダー

VAL-001では、以下のリクエストヘッダーを使用する。

| ヘッダー名       |  必須 | 説明                 |
| ----------- | :-: | ------------------ |
| `X-User-Id` |  ○  | 操作対象となる利用者ID       |
| `Accept`    |  ○  | `application/json` |

リクエスト例：

```http
GET /api/v1/month-end-asset-snapshots/20/holding-values
Accept: application/json
X-User-Id: 1
```

---

### 8.1 X-User-Id

`X-User-Id`には、操作対象となる利用者IDを指定する。

```http
X-User-Id: 1
```

API共通方針に従って、以下を確認する。

* ヘッダーが指定されていること
* ID形式が正しいこと
* 指定された利用者が存在すること
* 指定された利用者が論理削除されていないこと

検証済みの利用者IDを利用者コンテキストへ保持し、VAL-001の処理で使用する。

---

### 8.2 Accept

`Accept`には、以下を指定する。

```http
Accept: application/json
```

VAL-001の正常レスポンスおよびエラーレスポンスはJSON形式で返却する。

---

### 8.3 Content-Type

VAL-001はRequest Bodyを持たないGET APIであるため、`Content-Type`は必須としない。

概念的には、

```http
GET /api/v1/month-end-asset-snapshots/20/holding-values
Accept: application/json
X-User-Id: 1
```

で実行できる。

---

## 9. リクエストボディ

なし。

VAL-001は参照専用GET APIであるため、Request Bodyを使用しない。

以下のようなリクエストボディは送信しない。

```json
{
  "snapshotId": "20"
}
```

また、

```json
{
  "targetYearMonth": "2026-08"
}
```

のような取得条件もRequest Bodyでは指定しない。

---

### 9.1 snapshotIdをRequest Bodyへ含めない

対象月末資産状況は、パスパラメータ

```text
snapshotId
```

によって指定する。

そのため、Request Bodyへ

```json
{
  "snapshotId": "20"
}
```

を含めない。

---

### 9.2 userIdをRequest Bodyへ含めない

操作対象利用者は、

```text
X-User-Id
```

から特定する。

以下のようなRequest Bodyは使用しない。

```json
{
  "userId": "1"
}
```

利用者指定経路を複数設けない。

---

### 9.3 取得条件をRequest Bodyで指定しない

GET APIであるため、以下のような検索条件をRequest Bodyへ持たせない。

```json
{
  "holdingAssetId": "10",
  "assetAccountId": "5"
}
```

VAL-001では、指定月末資産状況に属する商品別月末評価額一覧を一括取得する。

---

## 10. リクエスト項目

VAL-001では、Request Bodyの項目は存在しない。

本APIで使用する入力情報は、以下とする。

| 入力元       | 項目           |  必須 | 用途               |
| --------- | ------------ | :-: | ---------------- |
| パスパラメータ   | `snapshotId` |  ○  | 対象となる月末資産状況を特定する |
| リクエストヘッダー | `X-User-Id`  |  ○  | 操作対象利用者を特定する     |
| リクエストヘッダー | `Accept`     |  ○  | JSONレスポンスを要求する   |
| クエリパラメータ  | なし           |  -  | 使用しない            |
| リクエストボディ  | なし           |  -  | 使用しない            |

---

### 10.1 snapshotId

商品別月末評価額を取得する月末資産状況IDを指定する。

```text
snapshotId
```

から、

```text
month_end_asset_snapshots
```

を取得し、対象年月および利用者境界を確定する。

---

### 10.2 X-User-Id

操作対象利用者を指定する。

```text
X-User-Id
```

と

```text
snapshotId
```

を組み合わせて、対象月末資産状況が操作対象利用者に帰属することを確認する。

---

### 10.3 holdingAssetIdを入力項目としない

VAL-001は一覧取得APIであるため、

```text
holdingAssetId
```

をRequest項目として受け付けない。

対象月末資産状況に属する商品別月末評価額をまとめて取得する。

---

### 10.4 targetYearMonthを入力項目としない

対象年月は、

```text
snapshotId
    ↓
month_end_asset_snapshots.target_year_month
```

から決定する。

そのため、

```text
targetYearMonth
```

をRequest項目として受け付けない。

---

### 10.5 assetAccountIdを入力項目としない

VAL-001では、

```text
assetAccountId
```

もRequest項目として受け付けない。

商品別月末評価額と資産口座との関係は、

```text
month_end_holding_values
    ↓
holding_assets
    ↓
asset_accounts
```

から取得する。

---

### 10.6 入力情報を最小化する

VAL-001では、

```text
どの利用者の
どの月末資産状況か
```

だけを指定すれば取得対象を一意に決定できる。

そのため、

```text
X-User-Id
+
snapshotId
```

以外の検索条件を追加しない。

APIの責務を、

```text
指定月末資産状況に属する
商品別月末評価額一覧を取得する
```

ことへ限定する。


---

## 11. バリデーション

VAL-001では、Request Bodyおよびクエリパラメータを使用しないため、主なバリデーション対象は以下とする。

```text
X-User-Id
snapshotId
```

概念的な検証順序は、以下とする。

```text
Request
    ↓
X-User-Id検証
    ↓
利用者コンテキスト確定
    ↓
snapshotId形式検証
    ↓
月末資産状況存在確認
    ↓
利用者境界確認
    ↓
商品別月末評価額一覧取得
```

VAL-001では、一覧取得対象が0件であること自体はバリデーションエラーとしない。

---

### 11.1 X-User-Id必須チェック

`X-User-Id`は必須とする。

以下のようにヘッダーが存在しない場合は、エラーとする。

```http
GET /api/v1/month-end-asset-snapshots/20/holding-values
Accept: application/json
```

概念的なエラーコードは、

```text
USER_CONTEXT_REQUIRED
```

とする。

`X-User-Id`の詳細な検証方式は、API共通方針に従う。

---

### 11.2 X-User-Id形式チェック

`X-User-Id`は、利用者IDとして有効な正の整数形式であることを確認する。

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

形式不正の場合は、

```text
INVALID_USER_ID
```

とする。

---

### 11.3 利用者存在チェック

`X-User-Id`で指定された利用者が存在することを確認する。

概念的には、

```text
users.id
    = X-User-Id
```

を確認する。

SoftDeletesを採用している場合は、論理削除済み利用者を有効な利用者として扱わない。

利用者が存在しない場合は、

```text
USER_NOT_FOUND
```

とする。

---

### 11.4 snapshotId必須チェック

`snapshotId`はURL構造上の必須項目とする。

```http
GET /api/v1/month-end-asset-snapshots/{snapshotId}/holding-values
```

`snapshotId`を省略した場合は、VAL-001のルート自体に一致しない。

例えば、

```http
GET /api/v1/month-end-asset-snapshots/holding-values
```

をVAL-001として処理しない。

---

### 11.5 snapshotId形式チェック

`snapshotId`は、正の整数形式であることを確認する。

正常例：

```text
1
20
999
```

不正例：

```text
0
-1
abc
1.5
1e3
20abc
```

形式不正の場合は、

```text
INVALID_SNAPSHOT_ID
```

とする。

Laravelでは、API共通方針に従ってルート制約または共通パスパラメータ検証で形式を検証する。

---

### 11.6 月末資産状況存在チェック

形式検証済みの`snapshotId`に対応する月末資産状況が存在することを確認する。

ただし、`snapshotId`だけで検索せず、操作対象利用者IDを取得条件へ含める。

概念的には、

```text
month_end_asset_snapshots.id
    = snapshotId

AND

month_end_asset_snapshots.user_id
    = UserContext.userId
```

とする。

SoftDeletesを採用している場合は、

```text
AND

month_end_asset_snapshots.deleted_at
    IS NULL
```

相当の条件も含める。

---

### 11.7 他利用者の月末資産状況

`snapshotId`自体が存在していても、別利用者に帰属する場合は取得対象としない。

例えば、

```text
snapshotId = 20
user_id = 1
```

の月末資産状況に対して、

```http
X-User-Id: 2
```

でVAL-001を実行した場合は、対象月末資産状況を取得できないものとして扱う。

この場合も、

```text
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND
```

とする。

他利用者に対象データが存在することを外部へ公開しない。

---

### 11.8 論理削除済み月末資産状況

`month_end_asset_snapshots`にSoftDeletesを採用している場合、論理削除済みの月末資産状況は取得対象外とする。

```text
deleted_at IS NOT NULL
```

の場合は、

```text
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND
```

として扱う。

以下を区別してクライアントへ返却しない。

```text
不存在
他利用者所属
論理削除済み
```

---

### 11.9 confirmedによる取得制限を行わない

対象月末資産状況の

```text
confirmed
```

は、VAL-001の取得可否条件としない。

以下のどちらでも商品別月末評価額一覧を取得できる。

```text
confirmed = false
```

```text
confirmed = true
```

確定状態は、参照可否ではなく更新可否や業務状態を制御するために使用する。

---

### 11.10 targetYearMonthの入力検証を行わない

VAL-001では、

```text
targetYearMonth
```

をRequestから受け取らない。

対象年月は、

```text
snapshotId
    ↓
month_end_asset_snapshots
    ↓
target_year_month
```

によって特定する。

そのため、VAL-001では`targetYearMonth`の形式検証を行わない。

---

### 11.11 holdingAssetIdの入力検証を行わない

VAL-001では、

```text
holdingAssetId
```

をRequestから受け取らない。

指定月末資産状況に属する商品別月末評価額を一覧として取得するため、個別の保有商品IDに対する入力バリデーションは行わない。

---

### 11.12 assetAccountIdの入力検証を行わない

VAL-001では、

```text
assetAccountId
```

もRequestから受け取らない。

そのため、資産口座IDについての入力バリデーションは行わない。

---

### 11.13 商品別月末評価額0件は正常

対象月末資産状況が存在していても、

```text
month_end_holding_values
```

に該当レコードが存在しない場合がある。

例えば、

```text
月末資産状況作成
    ↓
商品別月末評価額
まだ未登録
```

という状態である。

この場合は、エラーとしない。

概念的には、

```json
{
  "data": []
}
```

として空配列を返却する。

---

### 11.14 商品別月末評価額の未登録を入力エラーとしない

商品単位で管理する保有商品が存在しているにもかかわらず、その月の商品別評価額が未登録であっても、VAL-001ではバリデーションエラーとしない。

VAL-001の責務は、

```text
現在登録されている
商品別月末評価額一覧を取得する
```

ことである。

```text
必要な商品別月末評価額が
すべて登録されているか
```

という確定可否判定は、SNP-004 月末資産状況確定APIの責務とする。

---

### 11.15 保有商品の現在の有効状態を取得可否条件としない

保存済みの

```text
month_end_holding_values
```

が存在する場合、関連する保有商品が現在無効化済みであっても、そのことだけを理由として一覧から除外しない。

概念的には、

```text
過去月の商品別評価額
    ↓
holding_assets
現在は無効
    ↓
保存済み評価額は参照可能
```

とする。

現在の保有商品状態によって過去の月末資産情報が欠落して見えないようにする。

---

### 11.16 保有商品の論理削除との関係

`holding_assets`にSoftDeletesを採用している場合も、保存済みの商品別月末評価額との履歴参照要件を考慮する。

VAL-001では、`month_end_holding_values`が正常に保存されているにもかかわらず、

```text
holding_assets.deleted_at IS NOT NULL
```

だけを理由として商品別月末評価額そのものを一覧から消さない設計を基本とする。

具体的なJOIN方式および表示情報の扱いは、業務ルール・Laravel実装方針で定義する。

---

### 11.17 関連データ不整合を通常の入力エラーとしない

例えば、

```text
month_end_holding_values
    ↓
holding_asset_id
    ↓
対応するholding_assetsが存在しない
```

という状態は、利用者のRequest内容に起因するバリデーションエラーではない。

外部キー制約によって原則として発生を防止する。

万一発生した場合は、通常の

```text
400 Bad Request
```

として扱わず、データ不整合または想定外の内部エラーとして扱う。

---

### 11.18 一覧件数の上限チェックを行わない

Phase1では、商品別月末評価額一覧にページングを導入しない。

そのため、

```text
limit
perPage
page
```

などに対するバリデーションも存在しない。

指定された月末資産状況に属する商品別月末評価額を全件取得する。

---

### 11.19 並び順指定のバリデーションを行わない

VAL-001では、

```text
sort
order
```

をRequestから受け取らない。

そのため、並び順指定に対するバリデーションも行わない。

一覧の並び順は、API側で統一する。

---

### 11.20 FormRequestを必須としない

VAL-001では、

* Request Bodyなし
* クエリパラメータなし

であるため、VAL-001専用のFormRequestは原則として作成しない。

以下のような空FormRequestを形式的に作成しない。

```php
final class ListMonthEndHoldingValuesRequest
    extends FormRequest
{
}
```

`snapshotId`の形式検証は、ルート制約またはAPI共通のパスパラメータ検証方式で行う。

---

### 11.21 バリデーション責務

VAL-001における主な検証責務を整理すると、以下とする。

| 検証対象         | 検証内容     | 主な実施箇所          |
| ------------ | -------- | --------------- |
| `X-User-Id`  | 必須       | 共通Middleware    |
| `X-User-Id`  | 正の整数形式   | 共通Middleware    |
| 利用者          | 存在確認     | 共通Middleware    |
| `snapshotId` | 正の整数形式   | Route / 共通検証    |
| 月末資産状況       | 存在確認     | Query / UseCase |
| 月末資産状況       | 利用者境界確認  | Query / UseCase |
| 月末資産状況       | 論理削除状態確認 | Query / UseCase |
| 商品別月末評価額0件   | 正常として扱う  | UseCase         |

---

### 11.22 バリデーションで行わないこと

VAL-001のバリデーションでは、以下を行わない。

* 商品別月末評価額の登録
* 商品別月末評価額の補完
* 商品別月末評価額の更新
* 商品別月末評価額の削除
* 月末資産状況の確定可否判定
* 月末資産状況の確定
* 月末資産状況の確定解除
* 保有商品の有効化・無効化
* 未登録評価額の自動生成
* `targetYearMonth`のRequest検証
* `holdingAssetId`のRequest検証
* `assetAccountId`のRequest検証

VAL-001では、

```text
操作対象利用者が
指定された月末資産状況を
参照できること
```

を確認したうえで、

```text
その月末資産状況に
現在保存されている
商品別月末評価額一覧を取得する
```

ことに責務を限定する。

---

## 12. 業務ルール

VAL-001では、操作対象利用者に帰属する指定された月末資産状況について、現在保存されている商品別月末評価額一覧を取得する。

概念的には、

```text
X-User-Id
+
snapshotId
    ↓
month_end_asset_snapshots
    ↓
month_end_holding_values
    ↓
holding_assets
    ↓
商品別月末評価額一覧
```

とする。

VAL-001は、参照専用APIであり、取得時に評価額や保有商品情報を変更しない。

---

### 12.1 操作対象利用者の月末資産状況のみ取得できる

対象月末資産状況は、以下の条件を満たす必要がある。

```text
month_end_asset_snapshots.id
    = snapshotId

AND

month_end_asset_snapshots.user_id
    = 操作対象利用者ID
```

論理削除を採用する場合は、さらに

```text
deleted_at IS NULL
```

相当の条件を含める。

`snapshotId`だけを条件として対象月末資産状況を取得しない。

---

### 12.2 他利用者の月末資産状況は取得できない

指定された`snapshotId`が他利用者に帰属する場合は、対象月末資産状況が存在しないものとして扱う。

概念的には、

```text
X-User-Id = 2

snapshotId = 20

month_end_asset_snapshots.id = 20
month_end_asset_snapshots.user_id = 1
    ↓
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND
```

とする。

他利用者の商品別月末評価額を取得しない。

---

### 12.3 確定状態に関係なく取得できる

VAL-001は参照専用APIであるため、対象月末資産状況が

```text
confirmed = false
```

または

```text
confirmed = true
```

のどちらでも取得可能とする。

確定済みであることを理由に参照を制限しない。

---

### 12.4 対象年月はsnapshotIdから特定する

VAL-001では、`targetYearMonth`をRequestから受け付けない。

対象年月は、

```text
snapshotId
    ↓
month_end_asset_snapshots.target_year_month
```

から決定する。

同じRequest内で

```text
snapshotId
+
targetYearMonth
```

を二重指定させない。

---

### 12.5 snapshotIdに属する評価額だけを取得する

取得対象となる`month_end_holding_values`は、指定された月末資産状況に紐づくものだけとする。

概念的には、

```text
month_end_holding_values.snapshot_id
    = snapshotId
```

を条件とする。

他の月末資産状況に属する商品別月末評価額を混在させない。

---

### 12.6 商品別月末評価額0件は正常とする

対象月末資産状況が存在していても、商品別月末評価額がまだ1件も登録されていない場合がある。

例えば、

```text
SNP-002
月末資産状況作成
    ↓
VAL登録前
    ↓
VAL-001
```

という状態である。

この場合は、エラーとせず

```json
{
  "data": []
}
```

を返却する。

---

### 12.7 未登録商品をVAL-001で補完しない

商品単位で管理する保有商品が存在していても、その商品について`month_end_holding_values`が未登録である場合がある。

VAL-001では、未登録商品について

```text
評価額 = 0
```

などの仮レコードを自動生成しない。

また、GET実行時にDBへレコードを作成しない。

---

### 12.8 未登録状態と0円を区別する

商品別月末評価額が未登録であることと、

```text
value = 0
```

として登録されていることは別の状態として扱う。

概念的には、

```text
レコードなし
    → 未登録

レコードあり
value = 0
    → 0円として登録済み
```

とする。

VAL-001では、保存済みレコードのみ返却する。

---

### 12.9 確定可否は判定しない

VAL-001では、

```text
必要な商品別月末評価額が
すべて揃っているか
```

を最終判定しない。

月末資産状況を確定できるかどうかは、SNP-004 月末資産状況確定APIの責務とする。

概念的には、

```text
VAL-001
    → 現在登録済みの一覧を返す

SNP-004
    → 必須評価額が揃っているか判定する
```

とする。

---

### 12.10 保有商品情報を組み合わせて返却する

商品別月末評価額だけでは画面上で商品を識別しにくいため、関連する

```text
holding_assets
```

から表示に必要な保有商品情報を取得する。

主に、

```text
holdingAssetId
holdingAssetName
assetType
```

などをレスポンスへ含める。

具体的なレスポンス項目は、後続で定義する。

---

### 12.11 保存済み過去評価額を現在の有効状態だけで除外しない

保有商品が現在無効化済みであっても、対象年月の商品別月末評価額が保存済みであれば、過去の月末資産情報として参照できるようにする。

例えば、

```text
2026-01
holdingAsset A
評価額登録済み

2026-08
holdingAsset A
無効化
```

という状態で、2026-01のVAL-001を実行した場合、保存済みの評価額を現在の無効状態だけを理由に除外しない。

---

### 12.12 現在の保有商品一覧を基準にしない

VAL-001では、

```text
現在有効なholding_assets一覧
```

を取得し、そこから商品別月末評価額を組み立てる方式を基本としない。

取得の起点は、

```text
month_end_holding_values
```

とする。

概念的には、

```text
month_end_holding_values
    ↓
holding_assets
```

とすることで、保存済みの過去月データを現在状態の変化から独立して参照する。

---

### 12.13 評価額は保存済み値をそのまま使用する

VAL-001では、商品別評価額をその場で再計算しない。

例えば、

```text
保有数量
×
現在価格
```

のような計算は行わない。

`month_end_holding_values`へ保存されている月末評価額を返却する。

---

### 12.14 現在価格を参照しない

VAL-001は月末時点の保存済み評価額を参照するAPIである。

そのため、現在時点の商品価格や外部価格情報によって過去の月末評価額を動的に変更しない。

---

### 12.15 商品単位管理以外の残高を混在させない

VAL-001では、

```text
month_end_asset_balances
```

に保存された口座単位残高を返却しない。

概念的には、

```text
VAL-001
    → month_end_holding_values

月末残高API
    → month_end_asset_balances
```

と責務を分離する。

---

### 12.16 他の月の評価額を返さない

同じ保有商品について複数月の評価額が存在していても、指定された`snapshotId`に紐づく評価額だけを返却する。

例えば、

```text
holdingAssetId = 10

2026-07
100,000円

2026-08
110,000円
```

で、

```text
snapshotId
    → 2026-08
```

の場合は、

```text
110,000円
```

のレコードだけをVAL-001の対象とする。

---

### 12.17 並び順はAPI側で統一する

VAL-001では、クライアントから並び順を指定させない。

Phase1では、画面上で安定した順序になるよう、API側で固定順を設定する。

例えば、以下のような順序とする。

```text
assetAccountId ASC
holdingAssetId ASC
```

または、保有商品側に明示的な表示順が存在する場合は、

```text
assetAccount表示順
holdingAsset表示順
```

を優先する。

正式な並び順は、保有商品・資産口座の一覧表示ルールと整合させる。

---

### 12.18 DBの自然順に依存しない

以下のように、`ORDER BY`なしのDB返却順をAPI仕様として扱わない。

```sql
SELECT
    ...
FROM
    month_end_holding_values
WHERE
    snapshot_id = ?
```

一覧表示の順序は明示的に指定する。

---

### 12.19 一覧取得時に集計しない

VAL-001は、商品別評価額1件ごとの一覧取得を責務とする。

以下のような集計値は、必要性がなければVAL-001で算出しない。

```text
商品別評価額合計
資産口座別合計
全商品合計
```

集計が必要な場合は、月末資産状況詳細や資産表示API側の責務として検討する。

---

### 12.20 取得結果を自動修正しない

例えば、商品別月末評価額に不整合が見つかっても、VAL-001実行時に

```text
金額修正
関連商品付け替え
削除
再登録
```

などを行わない。

GET APIとして副作用を持たせない。

---

## 13. 処理フロー

VAL-001の基本処理フローは、以下とする。

```text
Request
    ↓
X-User-Id検証
    ↓
UserContext取得
    ↓
snapshotId形式検証
    ↓
月末資産状況取得
    ↓
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND？
    ↓ No
商品別月末評価額一覧取得
    ↓
holding_assets情報取得
    ↓
並び順適用
    ↓
Result DTO生成
    ↓
Resource Collection
    ↓
200 OK
```

---

### 13.1 利用者コンテキスト確認

共通Middlewareで`X-User-Id`を検証し、操作対象利用者を特定する。

利用者コンテキストが確立できない場合は、月末資産状況検索へ進まない。

---

### 13.2 snapshotId形式検証

`snapshotId`が正の整数形式であることを確認する。

形式不正の場合は、DB検索へ進まない。

---

### 13.3 月末資産状況取得

操作対象利用者IDと`snapshotId`を条件として、月末資産状況を取得する。

概念的には、

```text
month_end_asset_snapshots.id
    = snapshotId

AND

month_end_asset_snapshots.user_id
    = userId
```

とする。

---

### 13.4 月末資産状況不存在

対象月末資産状況を取得できない場合は、

```text
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND
```

として処理を終了する。

以下を同じ扱いとする。

* 月末資産状況不存在
* 他利用者所属
* 論理削除済み

---

### 13.5 商品別月末評価額取得

対象月末資産状況が確認できた後に、

```text
month_end_holding_values.snapshot_id
    = snapshotId
```

を条件として一覧を取得する。

---

### 13.6 保有商品情報取得

各評価額に対応する

```text
holding_assets
```

から、レスポンスに必要な保有商品情報を取得する。

N+1を発生させないよう、JOINまたはEager Load等でまとめて取得する。

---

### 13.7 0件の場合

商品別月末評価額が0件の場合は、エラーとせず

```json
{
  "data": []
}
```

を返却する。

---

### 13.8 並び順適用

取得した一覧へAPI仕様で定めた固定の並び順を適用する。

クライアントごとに異なる順序にならないようにする。

---

### 13.9 Result DTO生成

取得したデータから、APIレスポンスに必要なResult DTOを生成する。

Eloquent ModelをそのままActionやResponderへ返却しない構成としてよい。

---

### 13.10 Resource Collectionへ変換

Result DTOの一覧をAPI Resource Collectionへ変換し、camelCaseのレスポンス形式を生成する。

---

### 13.11 正常レスポンス

一覧取得に成功した場合は、

```http
200 OK
```

を返却する。

0件の場合も`200 OK`とする。

---

## 14. 成功レスポンス

VAL-001で商品別月末評価額一覧の取得に成功した場合は、

```http
200 OK
```

を返却する。

レスポンス例：

```json
{
  "data": [
    {
      "id": "101",
      "holdingAssetId": "10",
      "holdingAssetName": "eMAXIS Slim 全世界株式",
      "assetType": "INVESTMENT_TRUST",
      "value": 350000
    },
    {
      "id": "102",
      "holdingAssetId": "11",
      "holdingAssetName": "eMAXIS Slim 米国株式",
      "assetType": "INVESTMENT_TRUST",
      "value": 220000
    }
  ]
}
```

具体的な項目名称は、テーブル定義および保有商品APIのレスポンス表現と整合させる。

---

### 14.1 0件の場合

商品別月末評価額が登録されていない場合は、

```json
{
  "data": []
}
```

を返却する。

HTTPステータスは、

```http
200 OK
```

とする。

---

### 14.2 404とはしない

対象月末資産状況が存在していて、商品別月末評価額だけが0件の場合は、

```http
404 Not Found
```

とはしない。

一覧リソースが存在し、その要素数が0件であると解釈する。

---

### 14.3 対象年月を各行へ重複して返さない

対象年月は、親となる月末資産状況から一意に決定できる。

そのため、商品別月末評価額の各行へ

```text
targetYearMonth
```

を重複して返す必要はない。

画面上で対象年月が必要な場合は、SNP-003などから取得する。

ただし、VAL系API全体で対象年月を明示的に返す方針がある場合は、そちらを優先する。

---

### 14.4 snapshotIdを各行へ返さない

URLですでに

```text
snapshotId
```

を指定しているため、各評価額へ`snapshotId`を繰り返し返却しない。

---

### 14.5 保存済み評価額を返す

レスポンスの

```text
value
```

には、`month_end_holding_values`へ保存されている値を返却する。

現在価格等から再計算した値ではない。

---

## 15. レスポンス項目

`data`配下には、商品別月末評価額の一覧を返却する。

1件あたりの主なレスポンス項目は、以下とする。

| 項目                 | 型       | NULL | 説明          |
| ------------------ | ------- | :--: | ----------- |
| `id`               | string  |   ×  | 商品別月末評価額ID  |
| `holdingAssetId`   | string  |   ×  | 保有商品ID      |
| `holdingAssetName` | string  |   ×  | 保有商品名       |
| `assetType`        | string  |   ×  | 商品種別        |
| `value`            | integer |   ×  | 対象月末の商品別評価額 |

---

### 15.1 id

`month_end_holding_values.id`をstringとして返却する。

例えば、

```json
{
  "id": "101"
}
```

とする。

DB上では`bigint`でも、APIではIDをstringとして扱う。

---

### 15.2 holdingAssetId

対象評価額に対応する

```text
holding_assets.id
```

をstringとして返却する。

---

### 15.3 holdingAssetName

関連する保有商品の名称を返却する。

DBカラム名が

```text
name
```

であっても、VAL-001では意味を明確にするため、

```text
holdingAssetName
```

として返却してよい。

---

### 15.4 assetType

関連する保有商品の商品種別を返却する。

表現方法は、HLD系APIの`assetType`と統一する。

同じ商品種別について、APIごとに異なる値表現を使用しない。

---

### 15.5 value

対象月末に保存された商品別評価額を整数で返却する。

例えば、

```json
{
  "value": 350000
}
```

とする。

日本円整数として扱い、文字列にはしない。

---

### 15.6 NULLを返さない

`month_end_holding_values`としてレコードが存在する場合は、`value`を必須値とする。

以下のような

```json
{
  "value": null
}
```

によって未登録状態を表現しない。

未登録の場合は、その商品別月末評価額レコード自体が存在しないものとして扱う。

---

### 15.7 0円は有効値とする

以下は有効な登録済み値として扱う。

```json
{
  "value": 0
}
```

0件・未登録とは区別する。

---

### 15.8 userIdを返却しない

利用者は、`X-User-Id`と月末資産状況の利用者境界によって確定している。

そのため、各レスポンス項目へ

```text
userId
```

を含めない。

---

### 15.9 snapshotIdを返却しない

`snapshotId`はURLで指定済みのため、各商品別評価額へ重複して返却しない。

---

### 15.10 createdAt・updatedAtを返却しない

Phase1の一覧表示において、商品別月末評価額の

```text
created_at
updated_at
```

は業務上必須ではない。

そのため、VAL-001ではレスポンス項目に含めない。

必要性が発生した場合は、VAL系API全体で返却方針を再検討する。

---

### 15.11 保有商品の内部管理情報を返却しない

以下のような保有商品の内部管理情報は、VAL-001で必要がなければ返却しない。

* `userId`
* `assetAccountId`
* `deletedAt`
* `createdAt`
* `updatedAt`

VAL-001では、商品別月末評価額の表示に必要な情報へ限定する。

---

## 16. トランザクション境界

VAL-001は参照専用GET APIであり、業務データを更新しない。

そのため、Phase1では明示的なDBトランザクションを使用しない。

概念的には、

```text
月末資産状況SELECT
    ↓
商品別月末評価額SELECT
    ↓
保有商品情報SELECT
    ↓
レスポンス生成
```

とする。

---

### 16.1 DB::transactionを使用しない

VAL-001では、

```php
DB::transaction(...)
```

を原則として使用しない。

以下の処理が存在しないためである。

```text
INSERT
UPDATE
DELETE
```

---

### 16.2 参照処理を短く保つ

VAL-001では、必要なデータを効率的に取得し、速やかにレスポンスを返却する。

トランザクションを不要に開始して、DB接続やロックを長時間保持しない。

---

### 16.3 lockForUpdateを使用しない

VAL-001では、

```php
lockForUpdate()
```

を使用しない。

参照中の商品別月末評価額を更新APIから排他的にロックする必要はない。

---

### 16.4 更新APIとの同時実行

VAL-001実行中に、別Requestによって商品別月末評価額が更新される可能性はある。

Phase1では、一覧表示用途として厳密なスナップショット分離レベルを要求しない。

レスポンス取得後に再取得すれば最新状態へ同期できる設計とする。

---

### 16.5 確定処理との同時実行

VAL-001実行中に、SNP-004による月末資産状況確定が実行される可能性もある。

VAL-001では、確定状態によって取得内容自体を変更しないため、確定処理との間で行ロックを取得しない。

---

### 16.6 複数SELECTの厳密な同一時点性を要求しない

月末資産状況確認と商品別月末評価額取得を複数SELECTで行う場合でも、Phase1ではすべてを厳密な同一DBスナップショットとして保証するための明示的トランザクションを必須としない。

ただし、JOIN等で1クエリとして安全かつ明確に取得できる場合は、その方式を優先してよい。

---

### 16.7 将来的に厳密な参照整合性が必要な場合

将来的に、

```text
確定済み月末資産状況について
複数テーブルの内容を
完全に同一時点で読み取る
```

ことが業務要件として必要になった場合は、読み取りトランザクションや分離レベルを改めて検討する。

Phase1では、一覧参照APIとして過剰なトランザクション制御を導入しない。

---

20. HTTPステータス

VAL-001では、処理結果に応じて以下のHTTPステータスを返却する。

HTTPステータス用途主なエラーコード

200 OK

商品別月末評価額一覧取得成功。0件を含む

-

400 Bad Request

利用者IDまたはsnapshotIdの形式不正

USER_CONTEXT_REQUIRED、INVALID_USER_ID、INVALID_SNAPSHOT_ID

404 Not Found

利用者または月末資産状況が存在しない

USER_NOT_FOUND、MONTH_END_ASSET_SNAPSHOT_NOT_FOUND

500 Internal Server Error

想定外のサーバー内部エラー

INTERNAL_SERVER_ERROR

具体的なHTTPステータスの最終定義は、API共通方針を優先する。

20.1 200 OK

以下の場合は、



200 OK

を返却する。

 商品別月末評価額が1件以上存在する 

 商品別月末評価額が0件 

一覧要素数によってHTTPステータスを変更しない。

20.2 400 Bad Request

Requestの識別情報が形式上不正な場合に使用する。

主な対象は、



USER_CONTEXT_REQUIRED
INVALID_USER_ID
INVALID_SNAPSHOT_ID

とする。

20.3 404 Not Found

以下の場合に使用する。



USER_NOT_FOUND
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND

他利用者に属する月末資産状況についても、404 Not Foundとして扱う。

20.4 500 Internal Server Error

想定外の例外や内部データ不整合によって正常に一覧を取得できない場合は、



500 Internal Server Error

とする。

21. エラーコード

VAL-001で使用する主なエラーコードは、以下とする。

エラーコードHTTPステータス発生条件

USER_CONTEXT_REQUIRED

API共通方針に従う

X-User-Id未指定

INVALID_USER_ID

400

X-User-Id形式不正

USER_NOT_FOUND

404

利用者不存在、または論理削除済み

INVALID_SNAPSHOT_ID

400

snapshotId形式不正

MONTH_END_ASSET_SNAPSHOT_NOT_FOUND

404

月末資産状況不存在、他利用者所属、論理削除済み

INTERNAL_SERVER_ERROR

500

想定外のサーバー内部エラー

21.1 USER_CONTEXT_REQUIRED

X-User-Idが指定されていない場合に使用する。

月末資産状況検索へ進まない。

21.2 INVALID_USER_ID

X-User-IdがAPI共通方針で定めたID形式として不正な場合に使用する。

21.3 USER_NOT_FOUND

指定利用者が存在しない、または利用対象として扱えない場合に使用する。

21.4 INVALID_SNAPSHOT_ID

snapshotIdがAPI共通方針で定めたID形式として不正な場合に使用する。

21.5 MONTH_END_ASSET_SNAPSHOT_NOT_FOUND

以下の場合に使用する。



月末資産状況不存在
他利用者所属
論理削除済み

これらを同一エラーコードとして扱うことで、他利用者の月末資産状況の存在をクライアントへ公開しない。

21.6 INTERNAL_SERVER_ERROR

想定外の例外や内部データ不整合などによって正常に一覧取得できない場合に使用する。

内部実装詳細はクライアントへ公開しない。

21.7 0件専用エラーコードを設けない

商品別月末評価額が0件であることは正常状態であるため、



MONTH_END_HOLDING_VALUES_NOT_FOUND

のような専用エラーコードは設けない。

22. 冪等性

VAL-001は、参照専用GET APIであるため、冪等である。

同一のDB状態、同一の利用者コンテキスト、同一のsnapshotIdに対して複数回実行しても、業務データを変更しない。

22.1 同一Requestの再実行

例えば、



GET /api/v1/month-end-asset-snapshots/20/holding-values
X-User-Id: 1
Accept: application/json

を複数回実行しても、



INSERT
UPDATE
DELETE

は発生しない。

22.2 GETによる副作用を持たせない

VAL-001では、以下の処理を行わない。

 未登録評価額の自動作成 

 評価額0円レコードの自動生成 

 保有商品情報の補正 

 月末資産状況の更新 

 確定状態の変更 

 アクセスを理由とした業務データ更新 

GETの参照性を維持する。

22.3 DB状態が変われば結果は変化する

VAL-001が冪等であることは、常に同じレスポンス内容を保証することではない。

例えば、別Requestによって



month_end_holding_values

が登録・更新された後にVAL-001を再実行すれば、最新DB状態に応じてレスポンス内容は変化する。

22.4 確定状態変更後

SNP-004またはSNP-005によって



confirmed

が変化しても、商品別月末評価額そのものが変更されていなければ、VAL-001の一覧内容は基本的に変化しない。

VAL-001では確定状態を取得条件にしないためである。

22.5 Idempotency-Keyは使用しない

VAL-001はGETによる参照APIであるため、



Idempotency-Key

を使用しない。

冪等性制御用の専用キーを導入する必要はない。

---

## 20. HTTPステータス

VAL-001では、処理結果に応じて以下のHTTPステータスを返却する。

| HTTPステータス | 用途 | 主なエラーコード |
|---|---|---|
| `200 OK` | 商品別月末評価額一覧取得成功。0件を含む | - |
| `400 Bad Request` | 利用者IDまたは`snapshotId`の形式不正 | `USER_CONTEXT_REQUIRED`、`INVALID_USER_ID`、`INVALID_SNAPSHOT_ID` |
| `404 Not Found` | 利用者または月末資産状況が存在しない | `USER_NOT_FOUND`、`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| `500 Internal Server Error` | 想定外のサーバー内部エラー | `INTERNAL_SERVER_ERROR` |

具体的なHTTPステータスの最終定義は、API共通方針を優先する。

---

### 20.1 200 OK

以下の場合は、

```http
200 OK
```

を返却する。

- 商品別月末評価額が1件以上存在する
- 商品別月末評価額が0件

一覧要素数によってHTTPステータスを変更しない。

---

### 20.2 400 Bad Request

Requestの識別情報が形式上不正な場合に使用する。

主な対象は、

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
INVALID_SNAPSHOT_ID
```

とする。

---

### 20.3 404 Not Found

以下の場合に使用する。

```text
USER_NOT_FOUND
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND
```

他利用者に属する月末資産状況についても、`404 Not Found`として扱う。

---

### 20.4 500 Internal Server Error

想定外の例外や内部データ不整合によって正常に一覧を取得できない場合は、

```http
500 Internal Server Error
```

とする。

---

## 21. エラーコード

VAL-001で使用する主なエラーコードは、以下とする。

| エラーコード | HTTPステータス | 発生条件 |
|---|---|---|
| `USER_CONTEXT_REQUIRED` | API共通方針に従う | `X-User-Id`未指定 |
| `INVALID_USER_ID` | `400` | `X-User-Id`形式不正 |
| `USER_NOT_FOUND` | `404` | 利用者不存在、または論理削除済み |
| `INVALID_SNAPSHOT_ID` | `400` | `snapshotId`形式不正 |
| `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` | `404` | 月末資産状況不存在、他利用者所属、論理削除済み |
| `INTERNAL_SERVER_ERROR` | `500` | 想定外のサーバー内部エラー |

---

### 21.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合に使用する。

月末資産状況検索へ進まない。

---

### 21.2 INVALID_USER_ID

`X-User-Id`がAPI共通方針で定めたID形式として不正な場合に使用する。

---

### 21.3 USER_NOT_FOUND

指定利用者が存在しない、または利用対象として扱えない場合に使用する。

---

### 21.4 INVALID_SNAPSHOT_ID

`snapshotId`がAPI共通方針で定めたID形式として不正な場合に使用する。

---

### 21.5 MONTH_END_ASSET_SNAPSHOT_NOT_FOUND

以下の場合に使用する。

```text
月末資産状況不存在
他利用者所属
論理削除済み
```

これらを同一エラーコードとして扱うことで、他利用者の月末資産状況の存在をクライアントへ公開しない。

---

### 21.6 INTERNAL_SERVER_ERROR

想定外の例外や内部データ不整合などによって正常に一覧取得できない場合に使用する。

内部実装詳細はクライアントへ公開しない。

---

### 21.7 0件専用エラーコードを設けない

商品別月末評価額が0件であることは正常状態であるため、

```text
MONTH_END_HOLDING_VALUES_NOT_FOUND
```

のような専用エラーコードは設けない。

---

## 22. 冪等性

VAL-001は、参照専用GET APIであるため、冪等である。

同一のDB状態、同一の利用者コンテキスト、同一の`snapshotId`に対して複数回実行しても、業務データを変更しない。

---

### 22.1 同一Requestの再実行

例えば、

```http
GET /api/v1/month-end-asset-snapshots/20/holding-values
X-User-Id: 1
Accept: application/json
```

を複数回実行しても、

```text
INSERT
UPDATE
DELETE
```

は発生しない。

---

### 22.2 GETによる副作用を持たせない

VAL-001では、以下の処理を行わない。

- 未登録評価額の自動作成
- 評価額0円レコードの自動生成
- 保有商品情報の補正
- 月末資産状況の更新
- 確定状態の変更
- アクセスを理由とした業務データ更新

GETの参照性を維持する。

---

### 22.3 DB状態が変われば結果は変化する

VAL-001が冪等であることは、常に同じレスポンス内容を保証することではない。

例えば、別Requestによって

```text
month_end_holding_values
```

が登録・更新された後にVAL-001を再実行すれば、最新DB状態に応じてレスポンス内容は変化する。

---

### 22.4 確定状態変更後

SNP-004またはSNP-005によって

```text
confirmed
```

が変化しても、商品別月末評価額そのものが変更されていなければ、VAL-001の一覧内容は基本的に変化しない。

VAL-001では確定状態を取得条件にしないためである。

---

### 22.5 Idempotency-Keyは使用しない

VAL-001はGETによる参照APIであるため、

```text
Idempotency-Key
```

を使用しない。

冪等性制御用の専用キーを導入する必要はない。

---

## 23. 設計上の補足

### 23.1 月末資産状況配下のリソースとする理由

商品別月末評価額は、単独で存在する情報ではなく、特定の月末資産状況に紐づく。

そのため、

```text
month-end-asset-snapshots
    ↓
holding-values
```

という親子関係をURLへ表現する。

```http
GET /api/v1/month-end-asset-snapshots/{snapshotId}/holding-values
```

とすることで、

```text
どの月末資産状況の商品別評価額か
```

を明確にする。

### 23.2 targetYearMonthをRequestへ持たせない理由

対象年月は、`snapshotId`から一意に決定できる。

概念的には、

```text
snapshotId
    ↓
month_end_asset_snapshots
    ↓
target_year_month
```

となる。

そのため、

```text
snapshotId
+
targetYearMonth
```

のように同じ意味の識別情報を二重指定させない。

### 23.3 商品別月末評価額を現在の保有商品一覧から生成しない理由

VAL-001は、

```text
現在存在する保有商品一覧
```

を返すAPIではない。

対象となるのは、

```text
指定月末資産状況に
実際に保存されている
商品別月末評価額
```

である。

そのため、取得の起点は

```text
month_end_holding_values
```

とする。

概念的には、

```text
month_end_holding_values
    ↓
holding_assets
```

として取得する。

### 23.4 保存済み過去データを現在状態から独立して扱う

保有商品は、商品別月末評価額を登録した後に無効化される可能性がある。

例えば、

```text
2026-01
商品A
評価額 = 300,000円
    ↓
2026-08
商品Aを無効化
```

となっても、2026-01の商品別月末評価額は過去の月末資産情報として引き続き参照できる必要がある。

そのため、現在の保有商品状態だけを理由に過去の評価額を一覧から除外しない。

### 23.5 保有商品のSoftDeletesに注意する

`holding_assets`でSoftDeletesを使用している場合、通常のEloquent Relationshipでは、

```text
deleted_at IS NULL
```

が自動的に適用される可能性がある。

VAL-001では、保存済みの商品別月末評価額に紐づく過去の保有商品情報を参照する必要があるため、このGlobal Scopeによって過去データが欠落しないよう注意する。

必要に応じて、

```php
withTrashed()
```

またはQuery BuilderによるJOINを使用する。

### 23.6 現在の有効状態で絞り込まない

保有商品に利用状態を表す属性が存在する場合でも、

```text
enabled = true
```

だけをVAL-001の取得条件にはしない。

VAL-001は現在利用中の商品一覧ではなく、保存済みの商品別月末評価額を参照するAPIだからである。

### 23.7 未登録と0円を区別する

VAL-001では、

```text
商品別月末評価額レコードなし
```

と、

```text
value = 0
```

を明確に区別する。

概念的には、

```text
レコードなし
    → 未登録

レコードあり
value = 0
    → 0円として登録済み
```

とする。

そのため、未登録商品を

```json
{
  "value": 0
}
```

として補完しない。

### 23.8 未登録商品を一覧へ含めない理由

VAL-001の責務は、

```text
保存済みの商品別月末評価額一覧を取得する
```

ことである。

そのため、評価額が未登録の商品を一覧へ疑似的に追加しない。

もし画面要件として、

```text
商品A
未登録

商品B
350,000円
```

のような表示が必要になった場合は、

```text
保有商品一覧
+
商品別月末評価額一覧
```

を組み合わせる、または未登録状態まで返す別APIを検討する。

### 23.9 VAL-001で確定可否を判定しない理由

商品別月末評価額一覧を取得した結果、必要な商品の評価額が不足している場合がある。

ただし、

```text
月末資産状況を確定できるか
```

という判定は、VAL-001の責務ではない。

概念的には、

```text
VAL-001
    → 現在登録済みのデータを返す

SNP-004
    → 必要な評価額が揃っているか判定する
```

と分離する。

### 23.10 GETで不足データを生成しない理由

商品別月末評価額が未登録であっても、VAL-001実行時に

```text
month_end_holding_values
```

へレコードを追加しない。

GET APIに副作用を持たせないためである。

### 23.11 GETでデータ補正しない理由

VAL-001は参照APIであるため、取得時に以下を行わない。

- 評価額修正
- 保有商品情報修正
- 関連付け変更
- 月末資産状況修正
- 不足レコード追加
- 不整合データ自動修復

データ不整合がある場合は、別の更新処理または保守対応で解決する。

### 23.12 月末残高APIと分離する理由

Life Plannerでは、残高記録単位によって保存先を分離する。

概念的には、

```text
口座単位
    ↓
month_end_asset_balances
    ↓
月末残高API

商品単位
    ↓
month_end_holding_values
    ↓
VAL系API
```

とする。

VAL-001で口座単位残高を混在させない。

### 23.13 HLD系APIと分離する理由

HLD系APIは、

```text
保有商品そのもの
```

を管理する。

一方、VAL系APIは、

```text
特定月末における
保有商品の評価額
```

を扱う。

概念的には、

```text
HLD
    → 商品マスタ・現在状態

VAL
    → 月次評価額
```

とする。

### 23.14 評価額を再計算しない理由

VAL-001では、保存済みの

```text
month_end_holding_values.value
```

を返却する。

例えば、

```text
保有数量
×
現在価格
```

などからその場で評価額を再計算しない。

VAL-001は、月末時点に確定または入力された保存済み値を参照するAPIとする。

### 23.15 現在価格を使わない理由

現在価格によって過去の評価額を再計算すると、過去の月末資産状況が時間経過によって変化してしまう。

そのため、

```text
2026-01の月末評価額
```

は、2026-01として保存された値を返却する。

### 23.16 snapshotIdをレスポンス各行へ返さない理由

`snapshotId`はRequest URLですでに指定されている。

そのため、

```json
{
  "snapshotId": "20"
}
```

を各商品別評価額へ重複して返却しない。

レスポンスを必要最小限に保つ。

### 23.17 targetYearMonthを各行へ返さない理由

対象年月も、親となる月末資産状況から一意に決定できる。

そのため、各商品別評価額に

```text
targetYearMonth
```

を繰り返し持たせない。

対象年月が必要な画面では、SNP-003等の月末資産状況情報を使用する。

### 23.18 holdingAssetNameを返す理由

商品別月末評価額だけでは、画面表示時にどの商品なのかを判断しづらい。

そのため、関連する

```text
holding_assets.name
```

を

```text
holdingAssetName
```

として返却する。

これにより、React側で評価額1件ごとにHLD-003を追加実行する必要をなくす。

### 23.19 assetTypeを返す理由

商品別評価額一覧では、商品名だけでなく商品種別も表示に使用する可能性がある。

そのため、

```text
assetType
```

を返却する。

ただし、表現方法はHLD系APIと統一する。

### 23.20 HLD系とassetType定義を共通化する

例えば、

```text
INVESTMENT_TRUST
STOCK
BOND
```

などの値を使用する場合、VAL-001独自の変換ルールを作らない。

Laravel側では共通Enum、React側では共通`AssetType`型を使用する。

### 23.21 assetAccountIdを返却しない理由

現時点のVAL-001では、商品別月末評価額一覧として必要な情報を

```text
holdingAssetId
holdingAssetName
assetType
value
```

に限定している。

資産口座単位のグルーピングが明確な画面要件として必要になった場合は、`assetAccountId`や資産口座名の返却を改めて検討する。

必要になるか不明な項目を先回りして増やさない。

### 23.22 一覧0件を正常とする理由

月末資産状況を作成した直後など、商品別月末評価額がまだ登録されていないことは正常に発生し得る。

そのため、

```json
{
  "data": []
}
```

を正常レスポンスとする。

### 23.23 一覧0件専用エラーを作らない理由

一覧取得APIで0件となることは、対象月末資産状況が存在しないこととは異なる。

概念的には、

```text
snapshot不存在
    → 404

snapshot存在
holdingValues 0件
    → 200 + []
```

とする。

### 23.24 固定の並び順をAPI側で持つ理由

一覧の順序をDBの自然順に依存すると、実行環境やクエリ計画によって順序が変化する可能性がある。

そのため、VAL-001では明示的な`ORDER BY`を使用する。

### 23.25 クライアントからsortを受け付けない理由

Phase1では、商品別月末評価額一覧について複数の並び替え要件を持たない。

そのため、

```text
?sort=
?order=
```

を導入せず、API側の固定順とする。

必要性が明確になった段階で拡張する。

### 23.26 ページングを導入しない理由

1利用者が1つの月末資産状況について管理する保有商品数は、Phase1では大量件数を想定していない。

そのため、

```text
page
perPage
cursor
```

などを導入せず、全件取得とする。

### 23.27 QueryとRepositoryを分離する理由

VAL-001は参照処理のみである。

そのため、

```text
Query
    → SELECT

Repository
    → INSERT / UPDATE / DELETE
```

という共通方針に従い、VAL-001ではQueryだけを使用する。

Repositoryを形式的に追加しない。

### 23.28 FormRequestを作らない理由

VAL-001では、

```text
Request Bodyなし
Query Parameterなし
```

である。

そのため、空のFormRequestを単なる統一感のためだけに追加しない。

`snapshotId`はRouteまたは共通パスパラメータ検証で扱う。

### 23.29 UseCaseを設ける理由

VAL-001では、

```text
月末資産状況確認
    ↓
商品別月末評価額取得
    ↓
DTO変換
```

というアプリケーション処理が存在する。

Actionへ直接Queryを記述せず、UseCaseで処理の流れを管理する。

### 23.30 Actionを薄く保つ理由

Actionは、

```text
snapshotId
+
UserContext
    ↓
UseCase
    ↓
Responder
```

の橋渡しだけを担当する。

DB検索や利用者境界確認をActionへ持たせない。

### 23.31 月末資産状況の利用者境界を先に確認する理由

`month_end_holding_values`に直接`user_id`を持たない場合でも、親となる月末資産状況で利用者境界を保証できる。

概念的には、

```text
UserContext
    ↓
month_end_asset_snapshots
    ↓
利用者境界確認済みsnapshotId
    ↓
month_end_holding_values
```

とする。

### 23.32 各評価額でuserIdを再確認しない理由

正常なデータモデルでは、

```text
month_end_asset_snapshots
month_end_holding_values
holding_assets
asset_accounts
```

の参照関係が登録時点で正しく保証される。

そのため、VAL-001で各評価額ごとに利用者境界を再計算する設計にはしない。

所有関係の整合性は、登録APIとDB制約で保証する。

### 23.33 DB制約を前提とする

少なくとも、

```text
month_end_holding_values.snapshot_id
    → month_end_asset_snapshots.id

month_end_holding_values.holding_asset_id
    → holding_assets.id
```

の参照整合性は、外部キー制約で保証する。

存在しない保有商品へ評価額を紐づけられないようにする。

### 23.34 不整合をVAL-001で自動修復しない

DB制約違反相当の不整合データが存在しても、VAL-001で

```text
自動削除
自動付け替え
自動生成
```

などを行わない。

想定外の内部状態として扱い、ログ等から調査する。

### 23.35 明示的トランザクションを使用しない理由

VAL-001は参照専用APIである。

Phase1では、複数SELECTを厳密な同一時点として読み取る要件もない。

そのため、

```php
DB::transaction()
```

を必須としない。

### 23.36 lockForUpdateを使用しない理由

VAL-001は商品別月末評価額を変更しない。

そのため、

```php
lockForUpdate()
```

によって更新APIを待機させる必要はない。

GET APIとして軽量な参照処理とする。

### 23.37 確定済みでも参照可能とする理由

月末資産状況の

```text
confirmed = true
```

は、月末資産状況が確定済みであることを表す。

確定済みであることは、過去データを参照できない理由にはならない。

そのため、VAL-001では確定・未確定のどちらでも一覧取得可能とする。

### 23.38 確定状態は更新API側で利用する

商品別月末評価額の登録・更新可否は、月末資産状況の確定状態に影響される。

ただし、その判定はVAL-001ではなくVAL登録・更新API側で行う。

参照APIへ更新制約を持ち込まない。

### 23.39 ReactではQueryとして扱う理由

VAL-001はGETであり、副作用を持たない。

そのため、TanStack Queryでは

```text
Query
```

として扱う。

Mutationとして実装しない。

### 23.40 snapshotIdをQuery Keyへ含める理由

商品別月末評価額一覧は、月末資産状況ごとに異なる。

そのため、

```text
snapshotId
```

をQuery Keyへ含める。

これにより、異なる月の一覧Cacheを誤って共有しない。

### 23.41 利用者切替時のCacheに注意する

Phase1では、`X-User-Id`で操作対象利用者を切り替える。

同じ`snapshotId`文字列が別利用者環境で存在する可能性も考慮し、利用者切替時に関連Cacheを破棄する、またはQuery Keyへ`userId`を含める。

### 23.42 Query KeyへuserIdを含めてもAPI引数にはしない

Cache管理上、

```text
userId
+
snapshotId
```

をQuery Keyへ含めてもよい。

ただし、VAL-001自体のAPI Client引数へ`userId`を渡す必要はない。

HTTPでは共通API Clientが

```text
X-User-Id
```

として付与する。

### 23.43 SNP-003と並列取得できる

月末資産状況詳細画面で、

```text
SNP-003
VAL-001
```

の両方が必要でも、`snapshotId`がすでに確定していれば直列に実行する必要はない。

概念的には、

```text
snapshotId
    ├─ SNP-003
    └─ VAL-001
```

として並列取得できる。

不要な通信待ちを増やさない。

### 23.44 SNP-003の情報をVAL-001へ重複させない

SNP-003から、

```text
targetYearMonth
confirmed
```

を取得できる場合、VAL-001へ同じ情報を各行単位で重複して持たせない。

APIごとの責務を維持する。

### 23.45 VAL登録・更新後はinvalidateする

商品別月末評価額を登録または更新した後は、VAL-001のCacheが古くなる。

そのため、

```text
VAL登録・更新成功
    ↓
VAL-001 invalidate
    ↓
再取得
```

を基本とする。

### 23.46 Optimistic Updateを必須としない

商品別月末評価額更新後にVAL-001のCacheを手動更新することも可能である。

ただし、Phase1ではサーバー確定状態を再取得する単純な方式を優先してよい。

### 23.47 0件とErrorをUI上で区別する

VAL-001の

```text
data = []
```

は正常状態である。

そのため、

```text
Loading
Error
Empty
Success
```

をReact側で明確に区別する。

### 23.48 未登録商品をReact側で0円補完しない

VAL-001に存在しない保有商品について、

```text
value = 0
```

として表示すると、

```text
未登録
```

と

```text
0円登録済み
```

を区別できなくなる。

そのため、VAL-001単独の結果から疑似的な0円データを生成しない。

### 23.49 一覧表示用にHLD-003を追加呼び出ししない

VAL-001では、表示に必要な

```text
holdingAssetName
assetType
```

を返却する。

そのため、一覧1件ごとにHLD-003を呼び出す構成にはしない。

フロントエンド側のN+1的なHTTP通信を防止する。

### 23.50 Phase1では設計を広げすぎない

VAL-001では、以下を対象外とする。

- 個別商品別月末評価額取得
- 資産口座IDによる絞り込み
- 保有商品IDによる絞り込み
- 任意ソート
- ページング
- 集計値返却
- 未登録商品補完
- 確定可否判定
- 評価額自動計算
- 現在価格取得
- GETによる自動登録
- データ自動修復
- 読み取りトランザクション
- 悲観ロック

Phase1では、

```text
操作対象利用者の
指定月末資産状況に
保存されている
商品別月末評価額一覧を
安全に取得する
```

ことへ責務を限定する。

## 24. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)