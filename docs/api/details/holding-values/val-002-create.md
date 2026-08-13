# VAL-002 商品別月末評価額登録

## 1. 概要

操作対象となる利用者について、
指定した月末資産状況に紐づく
商品別月末評価額を新規登録する。

本APIでは、
残高記録単位が
商品単位である資産口座に属する
保有商品について、
対象年月末時点の評価額を登録する。

商品別月末評価額が
すでに登録されている場合は、
本APIでは上書きしない。

既存の商品別月末評価額を変更する場合は、
VAL-003 商品別月末評価額更新APIを使用する。

口座単位で残高を記録する資産口座については、
本APIでは登録しない。

資産口座単位の月末資産残高は、
BAL-002 月末資産残高登録APIを使用する。

---

## 2. ユースケース

利用者は、
指定した月末資産状況に対して、
対象年月時点の保有商品の
月末評価額を登録する。

例えば、
以下のような場合に使用する。

- 投資信託の月末評価額を登録する
- 株式の月末評価額を登録する
- ETFなどの保有商品の月末評価額を登録する
- 未登録の商品別月末評価額を入力する
- 月末資産状況を確定するために必要な評価額を登録する
- 評価額が0円となった保有商品を0円として登録する

月末資産状況が
確定済みの場合は、
本APIで商品別月末評価額を登録できない。

確定済みの月末資産状況を修正する場合は、
SNP-005 月末資産状況確定解除APIによって
確定解除した後に登録する。

---

## 3. エンドポイント

```http
POST /api/v1/month-end-asset-snapshots/{snapshotId}/holding-values
```

---

## 4. HTTPメソッド

```text
POST
```

本APIは、指定した月末資産状況に対して新しい商品別月末評価額リソースを作成する。

商品別月末評価額の更新、月末資産残高の登録・更新、月末資産状況の確定および確定解除は行わない。

---

## 5. 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、`X-User-Id`リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

指定された利用者に属する月末資産状況についてのみ、商品別月末評価額を登録できる。

他の利用者に属する月末資産状況へ商品別月末評価額を登録することはできない。

月末資産状況を取得する際は、必ず以下を検索条件に含める。

```text
id = snapshotId
AND
user_id = 操作対象利用者ID
```

指定された`snapshotId`が他の利用者に属する場合は、対象となる月末資産状況が存在しないものとして扱う。

登録対象となる保有商品についても、操作対象利用者に属する資産口座を経由して利用者境界を確認する。

保有商品を取得する場合は、少なくとも以下の関連を満たすことを確認する。

```text
holding_assets.id = holdingAssetId
AND
holding_assets.asset_account_id = asset_accounts.id
AND
asset_accounts.user_id = 操作対象利用者ID
```

他の利用者に属する資産口座の保有商品へ商品別月末評価額を登録することはできない。

指定された`holdingAssetId`が他の利用者に属する場合は、対象となる保有商品が存在しないものとして扱う。

商品別月末評価額自体には`user_id`を保持せず、

```text
month_end_holding_values
    ↓
month_end_asset_snapshots
    ↓
users
```

および

```text
month_end_holding_values
    ↓
holding_assets
    ↓
asset_accounts
    ↓
users
```

の関連から利用者境界を保証する。

利用者IDは、リクエストボディ、クエリパラメータまたはパスパラメータでは受け付けない。

利用者IDは、ミドルウェアで設定された利用者コンテキストから取得する。

`X-User-Id`が指定されていない場合、形式が不正な場合、または指定された利用者が存在しない場合は、API共通方針に従ってエラーを返却する。

本APIでは、他の利用者に属する以下のリソースの存在をレスポンスから推測できないようにする。

- 月末資産状況
- 資産口座
- 保有商品
- 商品別月末評価額

---

## 6. パスパラメータ

| パラメータ名 | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `snapshotId` | string | ○ | 商品別月末評価額を登録する月末資産状況ID |

リクエスト例：

```http
POST /api/v1/month-end-asset-snapshots/12/holding-values
```

`snapshotId`は、
商品別月末評価額を登録する対象となる
月末資産状況を一意に識別するIDである。

対象年月は、
指定された月末資産状況の
`target_year_month`から取得する。

リクエストボディから
対象年月を指定することはできない。

---

## 7. クエリパラメータ

なし。

本APIでは、
クエリパラメータを使用しない。

---

## 8. リクエストヘッダー

### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Content-Type` | ○ | `application/json`を指定する |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
POST /api/v1/month-end-asset-snapshots/12/holding-values
Content-Type: application/json
Accept: application/json
X-User-Id: 1
```

---

## 9. リクエストボディ

リクエストボディは、
JSON形式とする。

```json
{
  "holdingAssetId": "5",
  "value": 850000
}
```

0円を登録する場合：

```json
{
  "holdingAssetId": "5",
  "value": 0
}
```

---

## 10. リクエスト項目

| 項目 | 型 | 必須 | NULL | 説明 |
|---|---|:---:|:---:|---|
| `holdingAssetId` | string | ○ | × | 商品別月末評価額を登録する保有商品ID |
| `value` | integer | ○ | × | 月末時点の商品評価額。日本円の整数値 |

`holdingAssetId`は、
API共通方針に従って
文字列として受け付ける。

`value`は、
日本円の整数値として受け付ける。

商品別月末評価額が
0円の場合は、
`0`を指定できる。

```json
{
  "holdingAssetId": "5",
  "value": 0
}
```

`null`は、
未登録状態を表すため、
登録APIでは受け付けない。

---

## 11. バリデーション

### 11.1 snapshotId

`snapshotId`は、
必須のパスパラメータとする。

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

正常例：

```text
1
12
123
```

不正例：

```text
0
-1
abc
1.5
```

`snapshotId`の形式が不正な場合は、
`VALIDATION_ERROR`
として扱う。

---

### 11.2 holdingAssetId

`holdingAssetId`は、
必須項目とする。

以下を検証する。

- 指定されていること
- `null`ではないこと
- 文字列であること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

正常例：

```json
{
  "holdingAssetId": "5"
}
```

不正例：

```json
{
  "holdingAssetId": null
}
```

```json
{
  "holdingAssetId": ""
}
```

```json
{
  "holdingAssetId": "0"
}
```

```json
{
  "holdingAssetId": "-1"
}
```

```json
{
  "holdingAssetId": "abc"
}
```

形式が不正な場合は、
`VALIDATION_ERROR`
として扱う。

---

### 11.3 value

`value`は、
必須項目とする。

以下を検証する。

- 指定されていること
- `null`ではないこと
- 整数であること
- 0以上であること
- 金額カラムで保持可能な範囲であること

正常例：

```json
{
  "value": 850000
}
```

```json
{
  "value": 0
}
```

不正例：

```json
{
  "value": null
}
```

```json
{
  "value": -1
}
```

```json
{
  "value": 100.5
}
```

```json
{
  "value": "850000"
}
```

商品別月末評価額は、
日本円の整数値として扱うため、
小数値は受け付けない。

また、
JSON文字列として送信された金額を
暗黙的に整数へ変換しない。

`value = 0`は
有効な登録値として扱う。

---

### 11.4 月末資産状況の存在確認

指定された`snapshotId`について、
以下の条件を満たす
月末資産状況が存在することを確認する。

```text
id = snapshotId
AND
user_id = 操作対象利用者ID
```

対象となる月末資産状況が
存在しない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

他の利用者に属する
月末資産状況IDが指定された場合も、
同じエラーとして扱う。

これにより、
他の利用者に属する
月末資産状況の存在を
レスポンスから判別できないようにする。

---

### 11.5 月末資産状況の確定状態

指定された月末資産状況が
未確定であることを確認する。

```text
confirmed = false
    → 登録可能

confirmed = true
    → 登録不可
```

確定済みの場合は、
商品別月末評価額を登録しない。

確定済みの月末資産状況を
修正する場合は、
SNP-005 月末資産状況確定解除APIによって
確定解除した後に登録する。

確定済みの場合は、
`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`
として扱う。

---

### 11.6 保有商品の存在確認

指定された`holdingAssetId`について、
操作対象利用者に属する
保有商品が存在することを確認する。

保有商品単体ではなく、
所属する資産口座を経由して
利用者境界を確認する。

概念的には、
以下の条件とする。

```text
holding_assets.id = holdingAssetId
AND
holding_assets.asset_account_id = asset_accounts.id
AND
asset_accounts.user_id = 操作対象利用者ID
```

対象となる保有商品が
存在しない場合は、
`HOLDING_ASSET_NOT_FOUND`
として扱う。

他の利用者に属する
保有商品が指定された場合も、
同じエラーとして扱う。

これにより、
他の利用者に属する
保有商品の存在を
レスポンスから判別できないようにする。

---

### 11.7 資産口座の残高記録単位

登録対象となる保有商品が属する
資産口座について、
残高記録単位が
商品単位であることを確認する。

```text
balance_recording_unit = 商品単位
    → 登録可能

balance_recording_unit = 口座単位
    → 登録不可
```

口座単位で残高を記録する
資産口座については、
商品別月末評価額を登録できない。

口座単位の月末資産残高は、
BAL-002 月末資産残高登録APIを使用する。

残高記録単位が
商品単位ではない場合は、
業務ルール違反として扱う。

---

### 11.8 対象年月時点の資産口座

登録対象となる保有商品が属する
資産口座について、
月末資産状況の
`target_year_month`時点で
月末資産管理対象であることを確認する。

判定には、
`asset_account_available_settings`
を使用する。

概念的には、
以下を判定する。

```text
asset_account_id
+
snapshot.target_year_month
    ↓
対象年月時点で月末資産管理対象か
```

対象年月時点で
月末資産管理対象ではない資産口座には、
商品別月末評価額を登録できない。

現在の資産口座の状態だけを使用して、
過去月への登録可否を
判定してはならない。

---

### 11.9 対象年月時点の保有商品

指定された保有商品が、
月末資産状況の
`target_year_month`時点で
評価額記録対象であることを確認する。

```text
対象年月時点で評価額記録対象
    → 登録可能

対象年月時点で評価額記録対象外
    → 登録不可
```

現在の保有商品の状態だけを使用して、
過去月への登録可否を
判定してはならない。

例えば、
現在は無効化されている保有商品でも、
対象年月時点で
評価額記録対象であった場合は、
登録可能とする。

反対に、
現在は有効であっても、
対象年月時点で
評価額記録対象ではなかった場合は、
登録できない。

---

### 11.10 商品別月末評価額の重複確認

同一の月末資産状況・保有商品について、
商品別月末評価額が
すでに登録されていないことを確認する。

確認条件は、
以下とする。

```text
month_end_asset_snapshot_id = snapshotId
AND
holding_asset_id = holdingAssetId
```

すでに商品別月末評価額が
存在する場合は、
新しいレコードを登録しない。

```text
未登録
    → VAL-002で登録

登録済み
    → VAL-002では登録不可
```

既存の商品別月末評価額を変更する場合は、
VAL-003 商品別月末評価額更新APIを使用する。

重複登録は、
業務ルール違反として扱う。

また、
アプリケーション側の事前確認だけではなく、
データベースでも以下の組み合わせに
UNIQUE制約を設定し、
同時リクエストによる
重複登録を防止する。

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

---

### 11.11 0円の扱い

`value = 0`は、
有効な商品別月末評価額として
登録できる。

```text
value = 0
    → 登録可能
```

0円を
未登録として扱ってはならない。

登録後は、

```text
value = null
    → 未登録

value = 0
    → 0円として登録済み
```

として区別する。

---

### 11.12 X-User-Id

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

`X-User-Id`が指定されていない場合は、
`USER_CONTEXT_REQUIRED`
として扱う。

形式が不正な場合は、
`INVALID_USER_ID`
として扱う。

指定された利用者が存在しない場合、
または論理削除されている場合は、
`USER_NOT_FOUND`
として扱う。

---

### 11.13 不要なリクエスト項目

本APIでは、
以下の項目をリクエストボディから
受け付けない。

- `id`
- `userId`
- `snapshotId`
- `assetAccountId`
- `targetYearMonth`
- `confirmed`
- `createdAt`
- `updatedAt`

これらは、
パスパラメータ、
利用者コンテキスト、
既存リソースとの関連または
サーバー側の処理によって決定する。

クライアントから
任意に指定させない。

---

### 11.14 業務状態に依存する検証

以下は、
単項目バリデーションではなく、
業務ルールとして検証する。

- 月末資産状況が操作対象利用者に属していること
- 月末資産状況が未確定であること
- 保有商品が操作対象利用者に属する資産口座のものであること
- 資産口座の残高記録単位が商品単位であること
- 資産口座が対象年月時点で月末資産管理対象であること
- 保有商品が対象年月時点で評価額記録対象であること
- 同一月末資産状況・保有商品について商品別月末評価額が未登録であること

これらの業務ルールに違反した場合は、
入力形式の不正を表す
`VALIDATION_ERROR`とは区別して扱う。

---

## 12. 業務ルール

- 商品別月末評価額は、操作対象利用者に属する月末資産状況に対してのみ登録できる。
- 他の利用者に属する月末資産状況へ商品別月末評価額を登録できない。
- 登録対象となる保有商品は、操作対象利用者に属する資産口座の保有商品である必要がある。
- 他の利用者に属する資産口座の保有商品へ商品別月末評価額を登録できない。
- 月末資産状況が未確定の場合のみ登録できる。
- 確定済みの月末資産状況には商品別月末評価額を登録できない。
- 確定済みの月末資産状況を修正する場合は、SNP-005 月末資産状況確定解除APIによって確定解除した後に登録する。
- 保有商品が属する資産口座は、対象年月時点で月末資産管理対象である必要がある。
- 保有商品が属する資産口座は、残高記録単位が商品単位である必要がある。
- 残高記録単位が口座単位の資産口座に属する保有商品には、本APIで商品別月末評価額を登録できない。
- 口座単位の月末資産残高は、BAL-002 月末資産残高登録APIを使用する。
- 登録対象となる保有商品は、対象年月時点で評価額記録対象である必要がある。
- 同一の月末資産状況・保有商品の組み合わせについて、商品別月末評価額は1件のみ登録できる。
- すでに商品別月末評価額が登録されている場合は、新規登録できない。
- 既存の商品別月末評価額を変更する場合は、VAL-003 商品別月末評価額更新APIを使用する。
- `value = 0`は有効な商品別月末評価額として登録できる。
- 商品別月末評価額は日本円の整数値として登録する。
- 商品別月末評価額の登録によって、月末資産状況を自動的に確定しない。
- 商品別月末評価額の登録によって、月末資産残高を登録・更新しない。
- 商品別月末評価額の登録によって、目的達成判定を自動実行しない。
- 商品別月末評価額を登録しても、過去の目的達成判定履歴は再計算しない。

---

## 13. 処理フロー

```text
リクエスト受付
    ↓
リクエストID生成
    ↓
X-User-Id検証
    ↓
操作対象利用者確認
    ↓
snapshotId検証
    ↓
リクエストボディ検証
    ↓
月末資産状況取得
    ↓
存在・利用者境界確認
    ↓
月末資産状況の確定状態確認
    ↓
保有商品取得
    ↓
所属資産口座・利用者境界確認
    ↓
資産口座の残高記録単位確認
    ↓
対象年月時点の資産口座利用可否確認
    ↓
対象年月時点の保有商品状態確認
    ↓
商品別月末評価額の重複確認
    ↓
商品別月末評価額登録
    ↓
APIレスポンス生成
    ↓
201 Created返却
```

月末資産状況の取得条件は、
以下とする。

```text
id = snapshotId
AND
user_id = 操作対象利用者ID
```

保有商品の利用者境界は、
所属する資産口座を経由して確認する。

```text
holding_assets.id = holdingAssetId
AND
holding_assets.asset_account_id = asset_accounts.id
AND
asset_accounts.user_id = 操作対象利用者ID
```

重複登録の確認条件は、
以下とする。

```text
month_end_asset_snapshot_id = snapshotId
AND
holding_asset_id = holdingAssetId
```

すべての登録条件を満たした場合のみ、
新しい商品別月末評価額を登録する。

---

## 14. 月末資産状況の扱い

商品別月末評価額を登録できるのは、
月末資産状況が
未確定の場合のみとする。

```text
confirmed = false
    → 登録可能

confirmed = true
    → 登録不可
```

確定済みの場合は、
SNP-005 月末資産状況確定解除APIによって
確定解除した後に登録する。

本APIによる登録成功後も、
月末資産状況は未確定のままとする。

```text
登録前
confirmed = false

    ↓ 商品別月末評価額登録

登録後
confirmed = false
```

本APIでは、
`month_end_asset_snapshots.confirmed`を
更新しない。

---

## 15. 資産口座の扱い

登録対象となる保有商品が属する
資産口座は、
以下の条件をすべて満たす必要がある。

```text
操作対象利用者に属している
AND
対象年月時点で月末資産管理対象である
AND
残高記録単位が商品単位である
```

対象年月は、
月末資産状況の
`target_year_month`を使用する。

例えば、

```text
月末資産状況
target_year_month = 2026-08
```

の場合は、
`2026-08`時点で
月末資産管理対象となっている
資産口座の保有商品のみ
登録対象とする。

現在の利用状態だけを使用して、
過去月への登録可否を
判定しない。

---

## 16. 残高記録単位

資産口座の
`balance_recording_unit`によって、
使用する月末資産登録APIを分ける。

```text
口座単位
    ↓
BAL-002 月末資産残高登録

商品単位
    ↓
VAL-002 商品別月末評価額登録
```

口座単位の資産口座について、
VAL-002で商品別月末評価額を
登録してはならない。

これにより、
同一資産口座について

```text
口座単位の月末資産残高
+
商品別月末評価額
```

を重複して管理することを防止する。

---

## 17. 保有商品の扱い

登録対象となる保有商品は、
対象年月時点で
評価額記録対象である必要がある。

対象年月は、
月末資産状況の
`target_year_month`を使用する。

例えば、

```text
target_year_month = 2026-08
```

の場合は、
`2026-08`時点で
評価額記録対象となっている
保有商品のみ登録できる。

現在は無効化されている保有商品でも、
対象年月時点で
評価額記録対象であった場合は、
対象年月の商品別月末評価額を
登録できる。

反対に、
現在は有効であっても、
対象年月時点で
評価額記録対象ではなかった場合は、
登録できない。

---

## 18. 重複登録

同一の月末資産状況と
保有商品の組み合わせについて、
商品別月末評価額は1件のみ保持する。

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

例えば、

```text
snapshotId = 12
holdingAssetId = 5
```

の商品別月末評価額が
すでに存在する場合、

同じ組み合わせで
VAL-002を再実行しても
新しいレコードは作成しない。

既存の商品別月末評価額を変更する場合は、
VAL-003 商品別月末評価額更新APIを使用する。

データベース側でも、
以下の組み合わせに
UNIQUE制約を設定し、
重複登録を防止する。

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

---

## 19. 0円の扱い

`value = 0`は、
有効な商品別月末評価額として扱う。

```json
{
  "holdingAssetId": "5",
  "value": 0
}
```

これは、
未登録とは異なる。

```text
商品別月末評価額レコードなし
    → 未登録

商品別月末評価額レコードあり
value = 0
    → 0円で登録済み
```

登録処理では、
`0`を未入力として扱ってはならない。

---

## 20. トランザクション境界

業務条件の最終確認から
商品別月末評価額の登録までを、
1つのデータベーストランザクション内で実行する。

トランザクション内では、
主に以下を行う。

1. 月末資産状況の最終確認
2. 月末資産状況の確定状態確認
3. 保有商品の存在・利用者境界確認
4. 資産口座の残高記録単位確認
5. 対象年月時点の資産口座利用可否確認
6. 対象年月時点の保有商品状態確認
7. 商品別月末評価額の重複確認
8. 商品別月末評価額の登録

登録条件を満たさない場合、
または処理途中で例外が発生した場合は、
商品別月末評価額を登録しない。

---

## 21. 成功レスポンス

### 21.1 HTTPステータス

```http
201 Created
```

商品別月末評価額の
新規登録に成功した場合は、
`201 Created`を返却する。

---

### 21.2 レスポンスボディ

```json
{
  "data": {
    "holdingAssetId": "5",
    "value": 850000
  }
}
```

0円を登録した場合：

```json
{
  "data": {
    "holdingAssetId": "5",
    "value": 0
  }
}
```

登録後の商品別月末評価額を
`data`オブジェクトとして返却する。

---

## 22. レスポンス項目

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `data` | object | × | 登録した商品別月末評価額 |
| `data.holdingAssetId` | string | × | 保有商品ID |
| `data.value` | integer | × | 登録した商品別月末評価額 |

`holdingAssetId`は、
API共通方針に従って
文字列として返却する。

`value`は、
日本円の整数値として返却する。

```json
{
  "value": 850000
}
```

0円の場合も、
整数の`0`として返却する。

```json
{
  "value": 0
}
```

本APIでは、
以下の情報は返却しない。

- `id`
- `month_end_asset_snapshot_id`
- `user_id`
- `asset_account_id`
- `target_year_month`
- `confirmed`
- `holding_asset_name`
- `asset_account_name`
- `balance_recording_unit`
- `created_at`
- `updated_at`
- `month_end_asset_balances`

月末資産状況そのものの情報は、
SNP-003 月末資産状況詳細取得APIで取得する。

商品別月末評価額一覧が必要な場合は、
VAL-001 商品別月末評価額一覧取得APIを使用する。

---

## 23. エラーレスポンス

異常時は、
API共通方針で定めた
共通エラーレスポンス形式を使用する。

本APIでは、
利用者コンテキストの不正、
`snapshotId`または
`holdingAssetId`の不正、
月末資産状況または保有商品の不存在、
確定済み月末資産状況への登録、
対象年月時点で利用できない資産口座、
残高記録単位の不一致、
対象年月時点で評価額記録対象ではない保有商品、
重複登録および
想定外のサーバーエラーを扱う。

エラーが発生した場合は、
商品別月末評価額を登録しない。

---

### 23.1 利用者コンテキストが指定されていない場合

`X-User-Id`が
指定されていない場合は、
`USER_CONTEXT_REQUIRED`
を返却する。

```json
{
  "error": {
    "code": "USER_CONTEXT_REQUIRED",
    "message": "操作対象の利用者を指定してください。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 23.2 利用者ID形式が不正な場合

`X-User-Id`が
API共通方針で定めたID形式に
一致しない場合は、
`INVALID_USER_ID`
を返却する。

```json
{
  "error": {
    "code": "INVALID_USER_ID",
    "message": "利用者IDの形式が不正です。",
    "details": [
      {
        "field": "X-User-Id",
        "reason": "invalidFormat",
        "message": "利用者IDの形式を確認してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 23.3 利用者が存在しない場合

指定された利用者が存在しない場合、
または論理削除されている場合は、
`USER_NOT_FOUND`
を返却する。

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "指定された利用者が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 23.4 バリデーションエラー

`snapshotId`、
`holdingAssetId`または
`value`が
バリデーション条件を満たさない場合は、
`VALIDATION_ERROR`
を返却する。

例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "value",
        "reason": "min",
        "message": "商品別月末評価額は0以上で指定してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

以下のような場合を含む。

- `snapshotId`の形式が不正
- `holdingAssetId`が未指定
- `holdingAssetId`が`null`
- `holdingAssetId`の形式が不正
- `value`が未指定
- `value`が`null`
- `value`がintegerではない
- `value`が負数
- `value`が保持可能な範囲を超えている

---

### 23.5 月末資産状況が存在しない場合

以下の条件を満たす
月末資産状況が存在しない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
を返却する。

```text
id = snapshotId
AND
user_id = 操作対象利用者ID
```

```json
{
  "error": {
    "code": "MONTH_END_ASSET_SNAPSHOT_NOT_FOUND",
    "message": "指定された月末資産状況が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

以下の場合を含む。

- 指定された`snapshotId`が存在しない
- 指定された`snapshotId`が他の利用者に属している

---

### 23.6 月末資産状況が確定済みの場合

指定された月末資産状況が
確定済みの場合は、
`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`
を返却する。

```json
{
  "error": {
    "code": "MONTH_END_ASSET_SNAPSHOT_CONFIRMED",
    "message": "確定済みの月末資産状況には商品別月末評価額を登録できません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

確定済みデータを修正する必要がある場合は、
先にSNP-005 月末資産状況確定解除APIを実行する。

---

### 23.7 保有商品が存在しない場合

指定された`holdingAssetId`について、
操作対象利用者に属する
保有商品が存在しない場合は、
`HOLDING_ASSET_NOT_FOUND`
を返却する。

```json
{
  "error": {
    "code": "HOLDING_ASSET_NOT_FOUND",
    "message": "指定された保有商品が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

以下の場合を含む。

- 指定された保有商品が存在しない
- 指定された保有商品が他の利用者に属する資産口座に紐づいている

他利用者の保有商品が
存在すること自体を
レスポンスから判別できないようにする。

---

### 23.8 対象年月時点で利用できない資産口座の場合

指定された保有商品が属する資産口座が、
月末資産状況の対象年月時点で
月末資産管理対象ではない場合は、
`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`
を返却する。

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH",
    "message": "指定された保有商品の資産口座は対象年月の月末資産管理対象ではありません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

現在の利用状態ではなく、
月末資産状況の
`target_year_month`時点の状態をもとに判定する。

---

### 23.9 残高記録単位が商品単位ではない場合

指定された保有商品が属する
資産口座の残高記録単位が
口座単位の場合は、
`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`
を返却する。

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH",
    "message": "指定された保有商品の資産口座は商品単位の評価額登録対象ではありません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

口座単位の資産口座については、
BAL-002 月末資産残高登録APIを使用する。

---

### 23.10 対象年月時点で評価額記録対象ではない場合

指定された保有商品が、
月末資産状況の
`target_year_month`時点で
評価額記録対象ではない場合は、
`HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH`
を返却する。

```json
{
  "error": {
    "code": "HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH",
    "message": "指定された保有商品は対象年月の商品別月末評価額登録対象ではありません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

現在の保有状態だけではなく、
対象年月時点の状態をもとに判定する。

---

### 23.11 商品別月末評価額がすでに登録されている場合

以下の組み合わせで
商品別月末評価額が
すでに存在する場合は、
`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`
を返却する。

```text
month_end_asset_snapshot_id = snapshotId
AND
holding_asset_id = holdingAssetId
```

```json
{
  "error": {
    "code": "MONTH_END_HOLDING_VALUE_ALREADY_EXISTS",
    "message": "指定された保有商品の商品別月末評価額はすでに登録されています。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

既存レコードを
本APIで上書きしない。

評価額を変更する場合は、
VAL-003 商品別月末評価額更新APIを使用する。

アプリケーション側の
重複確認で検知した場合と、
データベースのUNIQUE制約違反で
検知した場合は、
同じエラーコードへ変換する。

---

### 23.12 想定外のエラーが発生した場合

データベース接続エラーなど、
想定外のサーバーエラーが発生した場合は、
`INTERNAL_SERVER_ERROR`
を返却する。

```json
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "サーバー内部でエラーが発生しました。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

SQL、
スタックトレース、
PostgreSQLの制約名および
内部例外メッセージは、
レスポンスへ含めない。

---

## 24. HTTPステータス

| HTTPステータス | 条件 |
|---:|---|
| `201 Created` | 商品別月末評価額の登録に成功した |
| `400 Bad Request` | 利用者コンテキストが未指定、または利用者IDの形式が不正である |
| `404 Not Found` | 指定された利用者、月末資産状況または保有商品が存在しない |
| `409 Conflict` | 現在の業務状態では商品別月末評価額を登録できない、またはすでに登録済みである |
| `422 Unprocessable Entity` | パスパラメータまたはリクエスト項目のバリデーションエラー |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

---

### 24.1 201の扱い

新しい商品別月末評価額の
登録に成功した場合は、
`201 Created`
を返却する。

`value = 0`の場合も、
正常な新規登録として
`201 Created`
を返却する。

---

### 24.2 400の扱い

以下の場合は、
`400 Bad Request`
を返却する。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正である

---

### 24.3 404の扱い

以下の場合は、
`404 Not Found`
を返却する。

- 指定された利用者が存在しない
- 指定された利用者が論理削除されている
- 指定された月末資産状況が存在しない
- 指定された月末資産状況が他の利用者に属している
- 指定された保有商品が存在しない
- 指定された保有商品が他の利用者に属している

他利用者に属するデータを指定した場合も、
対象リソースの存在を公開しない。

---

### 24.4 409の扱い

リクエスト形式および
対象リソースは正しいが、
現在の業務状態では
商品別月末評価額を登録できない場合は、
`409 Conflict`
を返却する。

以下の場合を含む。

- 月末資産状況が確定済み
- 保有商品が属する資産口座が対象年月時点で月末資産管理対象ではない
- 資産口座の残高記録単位が商品単位ではない
- 保有商品が対象年月時点で評価額記録対象ではない
- 同一月末資産状況・保有商品の商品別月末評価額がすでに存在する

---

### 24.5 422の扱い

以下の場合は、
`422 Unprocessable Entity`
を返却する。

- `snapshotId`の形式が不正
- `holdingAssetId`が未指定
- `holdingAssetId`の形式が不正
- `value`が未指定
- `value`が`null`
- `value`の型が不正
- `value`が負数
- `value`が許容範囲外

---

## 25. エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `USER_CONTEXT_REQUIRED` | 400 | `X-User-Id`が指定されていない | × |
| `INVALID_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `USER_NOT_FOUND` | 404 | 指定された利用者が存在しない、または論理削除されている | × |
| `VALIDATION_ERROR` | 422 | `snapshotId`、`holdingAssetId`または`value`が不正である | × |
| `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` | 404 | 月末資産状況が存在しない、または他利用者に属している | × |
| `MONTH_END_ASSET_SNAPSHOT_CONFIRMED` | 409 | 月末資産状況が確定済みである | × |
| `HOLDING_ASSET_NOT_FOUND` | 404 | 保有商品が存在しない、または他利用者に属している | × |
| `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH` | 409 | 保有商品が属する資産口座が対象年月時点で月末資産管理対象ではない | × |
| `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH` | 409 | 資産口座の残高記録単位が商品単位ではない | × |
| `HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH` | 409 | 保有商品が対象年月時点で評価額記録対象ではない | × |
| `MONTH_END_HOLDING_VALUE_ALREADY_EXISTS` | 409 | 同一月末資産状況・保有商品の評価額がすでに登録されている | × |
| `INTERNAL_SERVER_ERROR` | 500 | 想定外のサーバーエラーが発生した | ○ |

エラーコードの正式な定義は、
`error-codes.md`
に従う。

業務状態を修正すれば
再実行可能になる場合でも、
同一リクエストを
そのまま自動再送して
解消するとは限らない。

---

## 26. 副作用

本APIでは、
`month_end_holding_values`へ
新しい商品別月末評価額を1件登録する。

登録対象以外の
業務データは変更しない。

本APIの実行によって、
以下を変更してはならない。

- `month_end_asset_snapshots.confirmed`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_balances`
- 既存の`month_end_holding_values`
- `assessment_histories`

また、
以下の処理を自動実行しない。

- 月末資産状況の確定
- 月末資産状況の確定解除
- 月末資産残高の登録・更新
- 他の保有商品の評価額登録・更新
- 目的達成判定
- 目的達成判定履歴の再計算

---

## 27. トランザクション

業務条件の最終確認から
商品別月末評価額登録までを、
1つのデータベーストランザクション内で実行する。

主に以下の処理を
同一トランザクション内で行う。

```text
月末資産状況取得
    ↓
確定状態確認
    ↓
保有商品・所属資産口座確認
    ↓
対象年月時点の利用可否確認
    ↓
残高記録単位確認
    ↓
対象年月時点の保有商品状態確認
    ↓
重複登録確認
    ↓
商品別月末評価額INSERT
```

処理途中で
業務例外または
想定外の例外が発生した場合は、
トランザクションをロールバックする。

商品別月末評価額だけが
中途半端に登録された状態を
残してはならない。

ただし、
アプリケーション側の
重複確認だけでは
並行リクエストによる競合を
完全には防げないため、
データベースのUNIQUE制約でも
重複登録を防止する。

---

## 28. 冪等性

本APIは、
HTTP POSTを使用して
新しい商品別月末評価額リソースを作成する。

そのため、
HTTPメソッドとしては
冪等ではない。

同一の月末資産状況・保有商品について
複数の商品別月末評価額を登録することは禁止する。

同じ内容のリクエストを
再度実行した場合は、
既存レコードを返却したり、
上書きしたりせず、

`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`

を返却する。

```text
1回目
POST /month-end-asset-snapshots/12/holding-values

holdingAssetId = 5
value = 850000

    ↓

201 Created


2回目
POST /month-end-asset-snapshots/12/holding-values

holdingAssetId = 5
value = 850000

    ↓

409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

2回目の`value`が
1回目と異なる場合も、
本APIでは上書きしない。

```text
登録済み
value = 850000

    ↓

VAL-002で
value = 900000を送信

    ↓

409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

既存の商品別月末評価額を変更する場合は、
VAL-003 商品別月末評価額更新APIを使用する。

同時に複数の登録要求が
実行された場合は、
以下の組み合わせに対する
データベースのUNIQUE制約によって
1件のみ登録されることを保証する。

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

アプリケーション側の
重複確認を両方のリクエストが
通過した場合でも、
UNIQUE制約違反を

`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`

へ変換する。

Phase1では、
`Idempotency-Key`は採用しない。

フロントエンドでは、
登録処理中に
登録ボタンを非活性化し、
意図しない二重送信を防止する。

ただし、
重複登録防止の最終的な保証は、
バックエンドおよび
データベース制約で行う。

---

## 32. Laravel実装方針

### 32.1 Action

HTTPリクエストを受け付け、
月末資産状況ID、
登録対象の保有商品ID、
商品別月末評価額および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの入力値を受け取り、
商品別月末評価額登録UseCaseを呼び出す。

UseCaseから受け取った登録結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- `snapshotId`の形式検証
- リクエストボディの単項目バリデーション
- 利用者境界の判定
- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 保有商品の存在確認
- 所属資産口座の利用者境界確認
- 対象年月時点の資産口座利用可否判定
- 残高記録単位の判定
- 対象年月時点の保有商品状態判定
- 重複登録確認
- 商品別月末評価額の登録
- トランザクション制御
- レスポンス生成処理

---

### 32.2 UseCase

商品別月末評価額登録の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 月末資産状況IDを受け取る
- 保有商品IDを受け取る
- 商品別月末評価額を受け取る
- 月末資産状況を取得する
- 月末資産状況が未確定であることを確認する
- 保有商品および所属資産口座を取得する
- 保有商品が操作対象利用者の資産口座に属していることを確認する
- 資産口座が対象年月時点で月末資産管理対象であることを確認する
- 資産口座の残高記録単位が商品単位であることを確認する
- 保有商品が対象年月時点で評価額記録対象であることを確認する
- 同一月末資産状況・保有商品の商品別月末評価額が未登録であることを確認する
- 商品別月末評価額を登録する
- 登録結果を返却する

指定された月末資産状況が存在しない場合、
または操作対象利用者に属していない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

指定された保有商品が存在しない場合、
または他の利用者に属する資産口座の
保有商品である場合は、
`HOLDING_ASSET_NOT_FOUND`
として扱う。

---

### 32.3 Form Request / DTO

リクエストボディの
形式および単項目バリデーションを担当する。

検証対象は、
以下とする。

- `holdingAssetId`
- `value`

#### holdingAssetId

以下を検証する。

- 必須であること
- `null`ではないこと
- 文字列であること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

#### value

以下を検証する。

- 必須であること
- `null`ではないこと
- integerであること
- 0以上であること
- 金額カラムで保持可能な範囲であること

業務状態に依存する以下の検証は、
Form Requestでは行わない。

- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 保有商品の存在確認
- 保有商品の利用者境界確認
- 対象年月時点の資産口座利用可否確認
- 残高記録単位確認
- 対象年月時点の保有商品状態確認
- 重複登録確認

検証済みの入力値は、
入力用DTOへ変換して
UseCaseへ渡す。

例：

```php
final readonly class CreateMonthEndHoldingValueInput
{
    public function __construct(
        public string $holdingAssetId,
        public int $value,
    ) {
    }
}
```

---

### 32.4 Query

商品別月末評価額登録に必要な
データ取得を担当する。

主な取得対象は、
以下とする。

- `month_end_asset_snapshots`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_holding_values`

本APIでは、
原則として以下を参照しない。

- `month_end_asset_balances`
- `assessment_histories`

---

### 32.5 月末資産状況取得

登録対象となる月末資産状況は、
必ず利用者境界を含めて取得する。

```php
$snapshot = MonthEndAssetSnapshot::query()
    ->where('id', $snapshotId)
    ->where('user_id', $userId)
    ->first([
        'id',
        'user_id',
        'target_year_month',
        'confirmed',
    ]);
```

以下のように、
月末資産状況IDだけで
取得してはならない。

```php
MonthEndAssetSnapshot::find($snapshotId);
```

取得できなかった場合は、
以下を区別せず
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

- 月末資産状況が存在しない
- 他の利用者に属している

---

### 32.6 確定状態確認

取得した月末資産状況が
未確定であることを確認する。

```php
if ($snapshot->confirmed) {
    throw new
        MonthEndAssetSnapshotConfirmedException();
}
```

確定済みの場合は、

`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`

として扱う。

本API内で
自動的に確定解除してはならない。

---

### 32.7 保有商品取得

登録対象となる保有商品は、
所属する資産口座とともに取得し、
利用者境界を確認する。

概念例：

```php
$holdingAsset = HoldingAsset::query()
    ->join(
        'asset_accounts',
        'asset_accounts.id',
        '=',
        'holding_assets.asset_account_id',
    )
    ->where(
        'holding_assets.id',
        $input->holdingAssetId,
    )
    ->where(
        'asset_accounts.user_id',
        $userId,
    )
    ->first([
        'holding_assets.id',
        'holding_assets.asset_account_id',
        'asset_accounts.balance_recording_unit',
    ]);
```

以下のように、
保有商品IDだけで
取得してはならない。

```php
HoldingAsset::find(
    $input->holdingAssetId,
);
```

取得できなかった場合は、
以下を区別せず
`HOLDING_ASSET_NOT_FOUND`
として扱う。

- 保有商品が存在しない
- 他の利用者に属する資産口座の保有商品である

---

### 32.8 対象年月時点の資産口座利用可否判定

月末資産状況の
`target_year_month`を使用して、
保有商品が属する資産口座が
対象年月時点で
月末資産管理対象であることを確認する。

判定には、
`asset_account_available_settings`
を使用する。

概念的には、
以下を判定する。

```text
assetAccountId
+
snapshot.target_year_month
    ↓
対象年月時点で月末資産管理対象か
```

対象年月時点で
月末資産管理対象ではない場合は、

`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`

として扱う。

現在の資産口座の状態だけを使用して、
過去月への登録可否を
判断してはならない。

---

### 32.9 残高記録単位確認

保有商品が属する資産口座の
`balance_recording_unit`が
商品単位であることを確認する。

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::HOLDING
) {
    throw new
        AssetAccountBalanceRecordingUnitMismatchException();
}
```

実際のEnum名・定数名は、
共通定義に従う。

条件を満たさない場合は、

`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`

として扱う。

口座単位の資産口座については、
BAL-002 月末資産残高登録APIを使用する。

---

### 32.10 対象年月時点の保有商品判定

指定された保有商品が、
月末資産状況の
`target_year_month`時点で
評価額記録対象であることを確認する。

概念的には、

```text
holdingAssetId
+
snapshot.target_year_month
    ↓
対象年月時点で評価額記録対象か
```

を判定する。

対象年月時点で
評価額記録対象ではない場合は、

`HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH`

として扱う。

現在の有効・無効状態だけを使用して、
過去月への登録可否を
判断してはならない。

---

### 32.11 重複確認

同一の月末資産状況・保有商品について、
商品別月末評価額が
すでに存在しないことを確認する。

```php
$exists = MonthEndHoldingValue::query()
    ->where(
        'month_end_asset_snapshot_id',
        $snapshot->id,
    )
    ->where(
        'holding_asset_id',
        $holdingAsset->id,
    )
    ->exists();
```

存在する場合は、

`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`

として扱う。

アプリケーション側の
重複確認だけでは、
同時実行時の重複を
完全には防止できない。

そのため、
データベースのUNIQUE制約も使用する。

---

### 32.12 Repository

商品別月末評価額の
新規登録を担当する。

登録する値は、
以下とする。

```text
month_end_asset_snapshot_id
holding_asset_id
value
```

登録例：

```php
return MonthEndHoldingValue::create([
    'month_end_asset_snapshot_id'
        => $snapshot->id,
    'holding_asset_id'
        => $holdingAsset->id,
    'value'
        => $input->value,
]);
```

`id`、
`created_at`および
`updated_at`は、
Laravelおよび
データベース側で設定する。

Repositoryでは、
以下の処理を行わない。

- 月末資産状況の確定
- 月末資産状況の確定解除
- 月末資産残高の登録・更新
- 他の保有商品の評価額登録・更新
- 目的達成判定の実行

---

### 32.13 トランザクション

業務条件の最終確認から
商品別月末評価額登録までを、
1つのデータベーストランザクション内で実行する。

実装例：

```php
$holdingValue = DB::transaction(
    function () use (
        $userId,
        $snapshotId,
        $input,
    ): MonthEndHoldingValue {
        $snapshot =
            $this->snapshotQuery
                ->findByUserAndId(
                    $userId,
                    $snapshotId,
                );

        if ($snapshot === null) {
            throw new
                MonthEndAssetSnapshotNotFoundException();
        }

        if ($snapshot->confirmed) {
            throw new
                MonthEndAssetSnapshotConfirmedException();
        }

        $holdingAsset =
            $this->holdingAssetQuery
                ->findByUserAndId(
                    $userId,
                    $input->holdingAssetId,
                );

        if ($holdingAsset === null) {
            throw new
                HoldingAssetNotFoundException();
        }

        $this->assetAccountAvailabilityValidator
            ->validate(
                $holdingAsset->assetAccount,
                $snapshot->target_year_month,
            );

        $this->recordingUnitValidator
            ->validateForHoldingValue(
                $holdingAsset->assetAccount,
            );

        $this->holdingAssetAvailabilityValidator
            ->validate(
                $holdingAsset,
                $snapshot->target_year_month,
            );

        if (
            $this->holdingValueQuery
                ->existsBySnapshotAndHoldingAsset(
                    $snapshot->id,
                    $holdingAsset->id,
                )
        ) {
            throw new
                MonthEndHoldingValueAlreadyExistsException();
        }

        return $this->repository->create(
            $snapshot,
            $holdingAsset,
            $input->value,
        );
    },
);
```

処理途中で例外が発生した場合は、
商品別月末評価額を登録しない。

---

### 32.14 UNIQUE制約

以下の組み合わせに、
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

複数リクエストが
同時に重複確認を通過しても、
データベース側で
複数レコードの作成を防止する。

UNIQUE制約違反が発生した場合は、
PostgreSQLの例外をそのまま返却せず、

`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`

へ変換する。

アプリケーション側の事前確認と
データベース制約の
両方を使用する。

---

### 32.15 Mass Assignment

クライアントから受け取った値を
そのままEloquentモデルへ渡してはならない。

以下のような実装は避ける。

```php
MonthEndHoldingValue::create(
    $request->all(),
);
```

登録値は、
検証済みの情報から
明示的に組み立てる。

```text
month_end_asset_snapshot_id
    → 利用者境界確認済みのsnapshot.id

holding_asset_id
    → 利用者境界確認済みのholdingAsset.id

value
    → Form Request / DTOの検証済み値
```

これにより、
クライアントから
意図しない外部キーや
サーバー管理項目を
指定されることを防止する。

---

### 32.16 0円の扱い

`value = 0`は、
有効な登録値として扱う。

以下のような
truthy / falsyによる判定は行わない。

```php
if (! $input->value) {
    // 0円も未入力扱いになるため使用しない
}
```

Form Requestでは、
`required`、
integer、
最小値0などを使用し、
0円を正常値として許容する。

---

### 32.17 業務ルール判定クラス

BAL系APIやVAL系APIで
共通して使用する業務ルールは、
必要に応じて
判定クラスへ分離する。

例えば、
以下の責務へ分離できる。

```text
AssetAccountAvailabilityValidator
    → 対象年月時点の資産口座利用可否判定

AssetAccountBalanceRecordingUnitValidator
    → 残高記録単位の判定

HoldingAssetAvailabilityValidator
    → 対象年月時点の保有商品判定
```

登録APIと更新APIで
同じ業務ルールを
別々に実装しない。

各Validatorは、
HTTPレスポンス生成や
データベース登録を行わない。

---

### 32.18 Responder

UseCaseから受け取った
登録後の商品別月末評価額を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`201 Created`とともに
`data`オブジェクトとして返却する。

Responderは、
以下の処理を行わない。

- データベース検索
- 利用者境界の判定
- 確定状態の判定
- 対象年月時点の資産口座利用可否判定
- 残高記録単位の判定
- 対象年月時点の保有商品判定
- 重複登録判定
- 商品別月末評価額の登録

---

### 32.19 API Resource

データベースカラムを直接返却せず、
API Resourceを利用して
APIレスポンス形式へ変換する。

変換例：

```php
return [
    'holdingAssetId'
        => (string) $this->holding_asset_id,
    'value'
        => (int) $this->value,
];
```

`value = 0`の場合も、
そのまま整数の`0`を返却する。

以下の項目は、
レスポンスへ含めない。

- 商品別月末評価額ID
- `snapshotId`
- `userId`
- `assetAccountId`
- `targetYearMonth`
- `confirmed`
- 保有商品名
- 資産口座名
- `createdAt`
- `updatedAt`

---

### 32.20 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、
`X-User-Id`を検証し、
操作対象利用者を特定する。

Action以降では、
検証済みの利用者コンテキストを使用する。

---

### 32.21 Eloquentモデル

`MonthEndHoldingValue`モデルは、
`month_end_holding_values`
テーブルへ対応する。

主に以下の属性を使用する。

```text
id
month_end_asset_snapshot_id
holding_asset_id
value
```

`value`は、
日本円の整数値として扱う。

必要に応じて、
integer castを設定する。

```php
protected function casts(): array
{
    return [
        'value' => 'integer',
    ];
}
```

また、
必要に応じて以下のRelationを定義する。

```php
public function snapshot(): BelongsTo
{
    return $this->belongsTo(
        MonthEndAssetSnapshot::class,
        'month_end_asset_snapshot_id',
    );
}

public function holdingAsset(): BelongsTo
{
    return $this->belongsTo(
        HoldingAsset::class,
        'holding_asset_id',
    );
}
```

---

### 32.22 upsertを使用しない

本APIは、
未登録の商品別月末評価額を
新規登録する責務のみを持つ。

そのため、
以下のような
`updateOrCreate`は使用しない。

```php
MonthEndHoldingValue::updateOrCreate(
    [
        'month_end_asset_snapshot_id'
            => $snapshot->id,
        'holding_asset_id'
            => $holdingAsset->id,
    ],
    [
        'value'
            => $input->value,
    ],
);
```

この実装では、
登録済みの商品別月末評価額を
VAL-002で更新できてしまう。

登録と更新の責務を分離するため、
登録済みの場合は

`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`

として扱う。

既存値の変更は、
VAL-003 商品別月末評価額更新APIで行う。

---

### 32.23 排他・競合への対応

アプリケーション側で
重複確認を行っても、
以下のような並行実行が発生し得る。

```text
リクエストA
重複なし確認

リクエストB
重複なし確認

リクエストA
INSERT

リクエストB
INSERT
```

そのため、
重複登録防止は
UNIQUE制約を最終防衛線とする。

Phase1では、
登録処理専用の
楽観ロックや
`Idempotency-Key`は使用しない。

---

### 32.24 例外変換

LaravelおよびPostgreSQLの
内部例外は、
そのままAPIレスポンスへ公開しない。

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 入力値不正 | `VALIDATION_ERROR` |
| 月末資産状況不存在 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 利用者境界外の月末資産状況 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 月末資産状況確定済み | `MONTH_END_ASSET_SNAPSHOT_CONFIRMED` |
| 保有商品不存在 | `HOLDING_ASSET_NOT_FOUND` |
| 利用者境界外の保有商品 | `HOLDING_ASSET_NOT_FOUND` |
| 資産口座が対象年月時点で利用不可 | `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH` |
| 残高記録単位不一致 | `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH` |
| 保有商品が対象年月時点で対象外 | `HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH` |
| 商品別月末評価額重複 | `MONTH_END_HOLDING_VALUE_ALREADY_EXISTS` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

UNIQUE制約違反については、
対象となる制約を判別し、
`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`
へ変換する。

SQL、
スタックトレース、
PostgreSQLの制約名および
内部例外メッセージは、
APIレスポンスへ含めない。

ログには、
調査に必要な範囲で
以下を記録する。

- 操作対象利用者ID
- 月末資産状況ID
- 保有商品ID
- 対象年月
- 独自エラーコード
- リクエストID

商品別月末評価額の具体的な金額は、
不要にエラーログへ出力しない。

---

## 33. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type CreateMonthEndHoldingValueParams = {
  snapshotId: string;
};
```

リクエスト型は、
以下とする。

```ts
export type CreateMonthEndHoldingValueRequest = {
  holdingAssetId: string;
  value: number;
};
```

レスポンス型は、
以下とする。

```ts
export type CreatedMonthEndHoldingValue = {
  holdingAssetId: string;
  value: number;
};

export type CreateMonthEndHoldingValueResponse = {
  data: CreatedMonthEndHoldingValue;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.post<CreateMonthEndHoldingValueResponse>(
    `/api/v1/month-end-asset-snapshots/${snapshotId}/holding-values`,
    {
      holdingAssetId,
      value: 850000,
    },
  );
```

商品別月末評価額入力画面などから
未登録の商品別月末評価額を
新規登録する際に利用する。

---

### 33.1 snapshotIdの扱い

`snapshotId`は、
API共通方針に従って
文字列として扱う。

```ts
const snapshotId: string = '12';
```

フロントエンド側で
数値へ変換して
業務計算には使用しない。

URL生成時も、
文字列のまま使用する。

---

### 33.2 holdingAssetIdの扱い

`holdingAssetId`は、
文字列として扱う。

```ts
const holdingAssetId: string = '5';
```

登録対象となる保有商品は、
リクエストボディで指定する。

```ts
const request: CreateMonthEndHoldingValueRequest = {
  holdingAssetId,
  value: 850000,
};
```

フロントエンド側で
保有商品の所有者境界を
最終判定しない。

登録可能かどうかは、
バックエンドで
操作対象利用者、
資産口座、
対象年月との関係を確認する。

---

### 33.3 valueの扱い

`value`は、
日本円の整数値として扱う。

```ts
const request: CreateMonthEndHoldingValueRequest = {
  holdingAssetId: '5',
  value: 850000,
};
```

0円も有効な登録値とする。

```ts
const request: CreateMonthEndHoldingValueRequest = {
  holdingAssetId: '5',
  value: 0,
};
```

以下のような
truthy / falsyによる
入力有無の判定は行わない。

```ts
if (!value) {
  // value = 0 も未入力扱いになるため使用しない
}
```

入力有無は、
`null`、
`undefined`、
空文字列などを
明示的に判定する。

---

### 33.4 入力フォーム

商品別月末評価額入力フォームでは、
利用者が入力した値を
integerへ変換してから
APIへ送信する。

HTMLの入力値は
文字列として取得されるため、
空文字列と0円を
明確に区別する。

例：

```ts
const [valueInput, setValueInput] =
  useState('');

const value =
  valueInput === ''
    ? null
    : Number(valueInput);
```

API呼び出し前に、
`value === null`でないことを確認する。

```ts
if (value === null) {
  return;
}
```

`Number()`による変換結果についても、
必要に応じて
整数であることを確認する。

```ts
if (!Number.isInteger(value)) {
  return;
}
```

ただし、
最終的な入力値検証は
バックエンドで行う。

---

### 33.5 VAL-001との連携

VAL-001 商品別月末評価額一覧取得APIで
`value = null`となっている保有商品を、
VAL-002の登録対象とする。

```ts
if (item.value === null) {
  // VAL-002で新規登録
}
```

`value = 0`は
登録済みであるため、
VAL-002を使用しない。

```ts
if (item.value !== null) {
  // VAL-003で更新
}
```

以下の判定は行わない。

```ts
if (!item.value) {
  // value = 0を未登録と誤判定するため使用しない
}
```

---

### 33.6 登録・更新APIの切り替え

商品別月末評価額の状態によって、
使用するAPIを切り替える。

```text
value = null
    ↓
VAL-002 商品別月末評価額登録

value !== null
    ↓
VAL-003 商品別月末評価額更新
```

フロントエンドでは、
VAL-002をupsert目的で使用しない。

登録後の評価額を変更する場合は、
VAL-003を使用する。

---

### 33.7 月末資産状況の確定状態

VAL-002を利用できるのは、
月末資産状況が
未確定の場合のみである。

編集可否を
画面上で補助的に制御する場合は、
SNP-003 月末資産状況詳細取得APIから
確定状態を取得する。

```ts
const canEdit =
  !snapshot.confirmed;
```

`confirmed = true`の場合は、
登録フォームや
保存ボタンを非活性化してよい。

ただし、
登録可否の最終判断は
VAL-002で行う。

---

### 33.8 確定済みエラーの扱い

月末資産状況が
確定済みの場合は、

`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`

が返却される。

フロントエンドでは、
登録できないことを表示する。

表示例：

```text
確定済みの月末資産状況には
商品別月末評価額を登録できません。

編集する場合は、
先に確定を解除してください。
```

必要に応じて、
SNP-005 月末資産状況確定解除APIを
利用する導線を表示する。

---

### 33.9 保有商品不存在エラーの扱い

`HOLDING_ASSET_NOT_FOUND`
が返却された場合は、
表示している保有商品情報が
最新ではない可能性がある。

そのため、
VAL-001を再取得し、
現在の商品別月末評価額一覧を
更新してよい。

他の利用者に属する
保有商品であった場合も
同じエラーとなるため、
フロントエンドでは
存在理由を推測しない。

---

### 33.10 資産口座の対象年月エラー

保有商品が属する資産口座が
対象年月時点で
月末資産管理対象ではない場合は、

`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`

が返却される。

表示例：

```text
この保有商品の資産口座は、
対象年月の月末資産管理対象ではありません。
```

フロントエンド側で
現在の資産口座状態だけを使用して
過去月の登録可否を
独自に判定しない。

---

### 33.11 残高記録単位不一致の扱い

指定した保有商品が属する
資産口座の残高記録単位が
商品単位ではない場合は、

`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`

が返却される。

表示例：

```text
この資産口座は、
商品別評価額の登録対象ではありません。
```

口座単位の資産口座については、
BAL-002 月末資産残高登録APIを使用する。

---

### 33.12 保有商品の対象年月エラー

指定した保有商品が
対象年月時点で
評価額記録対象ではない場合は、

`HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH`

が返却される。

表示例：

```text
この保有商品は、
対象年月の商品別月末評価額の
登録対象ではありません。
```

エラー発生後は、
必要に応じてVAL-001を再取得し、
最新の一覧状態を表示する。

---

### 33.13 重複登録エラーの扱い

商品別月末評価額が
すでに登録されている場合は、

`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`

が返却される。

これは、
例えば以下のような場合に発生し得る。

```text
VAL-001取得
    ↓
value = null

別タブ・別リクエストで登録
    ↓
現在は登録済み

古い画面状態のままVAL-002実行
    ↓
409 Conflict
```

この場合は、
VAL-001を再取得し、
最新状態を画面へ反映する。

VAL-002失敗後に
自動的にVAL-003へ切り替えて
同じ値を更新する処理は、
Phase1では行わない。

---

### 33.14 登録処理中の画面制御

登録処理中は、
保存ボタンを非活性化する。

```ts
const [isSubmitting, setIsSubmitting] =
  useState(false);
```

登録開始時に
`isSubmitting = true`とし、
成功または失敗後に
`false`へ戻す。

これにより、
利用者による
意図しない連続クリックを抑止する。

ただし、
二重登録防止の最終的な保証は、
バックエンドおよび
データベースのUNIQUE制約で行う。

---

### 33.15 登録成功時の扱い

登録成功後は、
レスポンスの`value`を
画面へ反映する。

```ts
const createdValue =
  response.data.value;
```

登録前：

```text
value = null
```

登録後：

```text
value = 850000
```

となる。

登録成功後は、
VAL-003による更新対象として
扱える。

---

### 33.16 再取得

登録成功後は、
必要に応じて
VAL-001を再取得する。

React Query等を使用する場合は、
対象の`snapshotId`に対応する
一覧クエリをinvalidateしてよい。

```ts
queryClient.invalidateQueries({
  queryKey: [
    'monthEndHoldingValues',
    snapshotId,
  ],
});
```

これにより、
登録後の最新状態を
一覧へ反映できる。

---

### 33.17 バリデーションエラーの扱い

`VALIDATION_ERROR`が返却された場合は、
`error.details.field`を利用して
対象項目へエラーを表示する。

本APIでは、
主に以下が対象となる。

```text
holdingAssetId
value
snapshotId
```

`value`については、
入力項目付近へ
エラーメッセージを表示する。

例：

```ts
if (detail.field === 'value') {
  setFieldError(
    'value',
    detail.message,
  );
}
```

`snapshotId`または
`holdingAssetId`の形式不正は、
画面状態または
保持しているIDが不正な状態として扱う。

---

### 33.18 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 入力項目または不正な画面状態としてエラー表示する |
| `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` | 月末資産状況一覧画面へ戻す |
| `MONTH_END_ASSET_SNAPSHOT_CONFIRMED` | 確定済みのため登録できないことを表示する |
| `HOLDING_ASSET_NOT_FOUND` | VAL-001を再取得し、対象商品が存在しないことを表示する |
| `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH` | 対象年月では資産口座が利用できないことを表示する |
| `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH` | 口座単位の残高管理対象であることを表示する |
| `HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH` | 対象年月では評価額記録対象外であることを表示する |
| `MONTH_END_HOLDING_VALUE_ALREADY_EXISTS` | VAL-001を再取得して登録済み状態を反映する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

## 34. 設計上の補足

### 34.1 登録APIを更新APIと分離する理由

商品別月末評価額には、

```text
未登録
```

と

```text
登録済み
```

という異なる状態が存在する。

VAL-002は、
未登録状態から
新しい商品別月末評価額を
作成する責務を持つ。

VAL-003は、
既存の商品別月末評価額を
変更する責務を持つ。

登録と更新を分離することで、
APIの責務を明確にする。

---

### 34.2 upsertを採用しない理由

VAL-002で
upsertを採用すると、
登録済みの商品別月末評価額を
本APIで変更できてしまう。

これでは、

```text
登録
```

と

```text
更新
```

の責務が曖昧になる。

そのため、
登録済みの場合は

`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`

を返却し、
変更はVAL-003で行う。

---

### 34.3 holdingAssetIdをリクエストボディで指定する理由

本APIは、

```text
月末資産状況に対して
どの保有商品の評価額を追加するか
```

を指定する新規登録APIである。

そのため、
親リソースとなる
月末資産状況は
`snapshotId`としてパスに含め、

登録対象となる
`holdingAssetId`は
リクエストボディで指定する。

```http
POST /api/v1/month-end-asset-snapshots/{snapshotId}/holding-values
```

```json
{
  "holdingAssetId": "5",
  "value": 850000
}
```

---

### 34.4 商品別月末評価額IDを公開しない理由

フロントエンドが
商品別月末評価額を操作する際に
必要となる業務上の識別情報は、

```text
月末資産状況
+
保有商品
```

である。

そのため、
内部的な
商品別月末評価額IDを
API契約へ公開しない。

登録後の更新も、

```text
snapshotId
+
holdingAssetId
```

によって対象を特定する。

---

### 34.5 valueをnullで登録できない理由

`null`は、
商品別月末評価額が
まだ登録されていない状態を表す。

一方、
VAL-002は
商品別月末評価額レコードを
新しく作成するAPIである。

そのため、

```text
value = null
```

を登録値として許可しない。

未登録状態は、
`month_end_holding_values`の
レコードが存在しないことで表現する。

---

### 34.6 0円を許可する理由

保有商品の月末評価額が
実際に0円になることはあり得る。

そのため、
`value = 0`は
正常な業務値として扱う。

```text
レコードなし
    → 未登録

レコードあり + value = 0
    → 0円として登録済み
```

を明確に区別する。

---

### 34.7 確定済みデータへ登録できない理由

確定済み月末資産状況は、
その対象年月の
正式な資産状況として扱う。

確定後に
商品別月末評価額を追加すると、
確定時点のデータと
現在の内容に不整合が生じる。

そのため、
修正が必要な場合は、

```text
確定解除
    ↓
商品別月末評価額登録
    ↓
再確定
```

の手順を使用する。

---

### 34.8 対象年月時点の資産口座を確認する理由

現在は月末資産管理対象の
資産口座であっても、
対象年月時点では
まだ管理対象ではなかった可能性がある。

反対に、
現在は対象外でも、
過去の対象年月では
管理対象だった可能性がある。

そのため、
現在状態ではなく
`snapshot.target_year_month`を基準に
登録可否を判定する。

---

### 34.9 対象年月時点の保有商品を確認する理由

保有商品についても、
現在状態と
対象年月時点の状態が
一致するとは限らない。

過去月の商品別月末評価額を
正しく記録できるよう、
対象年月時点の状態を
基準として登録可否を判定する。

---

### 34.10 残高記録単位を確認する理由

商品別月末評価額を使用するのは、
残高記録単位が
商品単位の資産口座のみである。

口座単位の資産口座へ
商品別月末評価額を登録すると、

```text
口座残高
+
商品別評価額
```

が混在し、
資産額を二重計上する原因となる。

そのため、
VAL-002でも
残高記録単位を必ず確認する。

---

### 34.11 登録によって自動確定しない理由

1件の商品別月末評価額を
登録しただけでは、
月末資産状況全体の
必要な入力が揃っているとは限らない。

そのため、
VAL-002から
SNP-004 月末資産状況確定APIを
自動実行しない。

確定は、
利用者による
明示的な操作とする。

---

### 34.12 目的達成判定を自動実行しない理由

商品別月末評価額の登録時点では、
月末資産状況は未確定である。

未確定データを使用して
目的達成判定を
自動的に実行しない。

必要な入力を完了し、
月末資産状況を確定した後に、
利用者が明示的に
目的達成判定を実行する。

---

### 34.13 保存済み判定履歴を再計算しない理由

目的達成判定履歴は、
判定を実行した時点の
入力値および判定結果を
保持する履歴である。

後から商品別月末評価額を
登録しても、
過去に保存された
目的達成判定履歴を
変更しない。

そのため、
VAL-002では
`assessment_histories`を
更新しない。

---

### 34.14 POSTを採用する理由

本APIは、
月末資産状況配下に
新しい商品別月末評価額リソースを
作成する。

そのため、
HTTPメソッドには
`POST`を採用する。

正常登録時は、
`201 Created`を返却する。

---

### 34.15 Idempotency-Keyを採用しない理由

本APIはPOSTであり、
HTTPメソッドとしては
冪等ではない。

ただし、
同一の

```text
snapshotId
+
holdingAssetId
```

について
複数レコードを作成できないよう、
UNIQUE制約によって
重複登録を防止する。

Phase1では、
決済などのように
通信再送による二重実行を
特別に吸収する必要性は低いため、
`Idempotency-Key`は採用しない。

---

### 34.16 フロントエンドだけに二重送信防止を依存しない理由

登録処理中に
保存ボタンを非活性化しても、

- 複数タブ
- API直接実行
- ネットワーク再送
- 並行リクエスト

などによって、
重複登録要求が発生する可能性がある。

そのため、
フロントエンドの制御は
UX上の補助とする。

重複登録防止の最終的な保証は、
バックエンドおよび
データベースのUNIQUE制約で行う。

---

### 34.17 キャッシュを採用しない理由

商品別月末評価額は、
月末資産入力中に
登録・更新されるデータである。

登録直後に
最新状態を確認する必要があるため、
Phase1では
アプリケーションキャッシュを採用しない。

登録成功後は、
必要に応じてVAL-001を再取得する。

---

## 35. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)
