# BAL-002 月末資産残高登録

## 1. 概要

操作対象となる利用者について、
指定した月末資産状況に紐づく
資産口座単位の月末資産残高を登録する。

本APIでは、
残高記録単位が
資産口座単位である資産口座について、
対象年月の月末資産残高を新規登録する。

商品単位で残高を記録する資産口座については、
本APIでは登録しない。

商品別月末評価額は、
VAL-002 商品別月末評価額登録APIを使用する。

---

## 2. ユースケース

利用者は、
指定した月末資産状況に対して、
対象年月時点の資産口座残高を登録する。

例えば、
以下のような場合に使用する。

- 普通預金口座の月末残高を登録する
- 現金口座の月末残高を登録する
- 定期預金口座の月末残高を登録する
- 未登録の月末資産残高を入力する
- 月末資産状況を確定するために必要な残高を登録する

すでに同じ月末資産状況・資産口座の
月末資産残高が登録されている場合は、
本APIでは上書きしない。

既存の月末資産残高を修正する場合は、
BAL-003 月末資産残高更新APIを使用する。

---

## 3. エンドポイント

``http
POST /api/v1/month-end-asset-snapshots/{snapshotId}/asset-balances
```

---

## 4. HTTPメソッド

```text
POST
```

本APIは、指定した月末資産状況に対して新しい月末資産残高リソースを作成する。

月末資産残高の更新、月末資産状況の確定、確定解除および商品別月末評価額の登録は行わない。

---

## 5. 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、`X-User-Id`リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

指定された利用者に属する月末資産状況についてのみ、月末資産残高を登録できる。

他の利用者に属する月末資産状況へ月末資産残高を登録することはできない。

月末資産状況を取得する際は、必ず以下を検索条件に含める。

```text
id = snapshotId
AND
user_id = 操作対象利用者ID
```

指定された`snapshotId`が他の利用者に属する場合は、対象となる月末資産状況が存在しないものとして扱う。

登録対象となる資産口座についても、操作対象利用者に属していることを確認する。

資産口座を取得する際は、以下を満たすことを確認する。

```text
id = assetAccountId
AND
user_id = 操作対象利用者ID
```

他の利用者に属する資産口座を指定して月末資産残高を登録することはできない。

利用者IDは、リクエストボディ、クエリパラメータまたはパスパラメータでは受け付けない。

利用者IDは、ミドルウェアで設定された利用者コンテキストから取得する。

月末資産残高の`user_id`は保持せず、月末資産状況および資産口座との関連から利用者境界を保証する。

`X-User-Id`が指定されていない場合、形式が不正な場合、または指定された利用者が存在しない場合は、API共通方針に従ってエラーを返却する。

本APIでは、他の利用者に属する月末資産状況、資産口座および月末資産残高の存在をレスポンスから推測できないようにする。

---

## 6. パスパラメータ

| パラメータ名 | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `snapshotId` | string | ○ | 月末資産残高を登録する月末資産状況ID |

リクエスト例：

```http
POST /api/v1/month-end-asset-snapshots/12/asset-balances
```

`snapshotId`は、
月末資産残高を登録する
対象の月末資産状況を
一意に識別するIDである。

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
POST /api/v1/month-end-asset-snapshots/12/asset-balances
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
  "assetAccountId": "3",
  "balance": 1200000
}
```

---

## 10. リクエスト項目

| 項目 | 型 | 必須 | NULL | 説明 |
|---|---|:---:|:---:|---|
| `assetAccountId` | string | ○ | × | 月末資産残高を登録する資産口座ID |
| `balance` | integer | ○ | × | 対象年月末時点の資産口座残高 |

`assetAccountId`は、
API共通方針に従って
文字列として受け付ける。

`balance`は、
日本円の整数値として受け付ける。

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

### 11.2 assetAccountId

`assetAccountId`は、
必須とする。

以下を検証する。

- 指定されていること
- `null`ではないこと
- 文字列であること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

正常例：

```json
{
  "assetAccountId": "3"
}
```

不正例：

```json
{
  "assetAccountId": null
}
```

```json
{
  "assetAccountId": ""
}
```

```json
{
  "assetAccountId": "0"
}
```

```json
{
  "assetAccountId": "-1"
}
```

```json
{
  "assetAccountId": "abc"
}
```

形式が不正な場合は、
`VALIDATION_ERROR`
として扱う。

---

### 11.3 balance

`balance`は、
必須とする。

以下を検証する。

- 指定されていること
- `null`ではないこと
- integerであること
- 0以上であること
- データベースで保持可能な範囲であること

正常例：

```json
{
  "balance": 1200000
}
```

0円も登録可能とする。

```json
{
  "balance": 0
}
```

以下は不正とする。

```json
{
  "balance": null
}
```

```json
{
  "balance": -1
}
```

```json
{
  "balance": 1000.5
}
```

```json
{
  "balance": "1200000"
}
```

月末資産残高は、
日本円の整数値として管理するため、
小数および数値文字列は受け付けない。

形式または値が不正な場合は、
`VALIDATION_ERROR`
として扱う。

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

---

### 11.5 月末資産状況の確定状態

月末資産残高を登録できるのは、
月末資産状況が
未確定の場合のみとする。

```text
confirmed = false
```

月末資産状況が
確定済みの場合は、
月末資産残高を登録できない。

```text
confirmed = true
    → 登録不可
```

確定済みの場合は、
月末資産状況を確定解除した後に
月末資産残高を登録する。

確定状態の判定は、
単項目バリデーションではなく
業務ルールとして扱う。

---

### 11.6 資産口座の存在確認

指定された`assetAccountId`について、
以下の条件を満たす
資産口座が存在することを確認する。

```text
id = assetAccountId
AND
user_id = 操作対象利用者ID
```

対象となる資産口座が
存在しない場合は、
`ASSET_ACCOUNT_NOT_FOUND`
として扱う。

他の利用者に属する
資産口座IDが指定された場合も、
同じエラーとして扱う。

これにより、
他の利用者に属する
資産口座の存在を
レスポンスから判別できないようにする。

---

### 11.7 対象年月時点の資産口座

指定された資産口座が、
月末資産状況の
`target_year_month`時点で
月末資産管理対象であることを確認する。

判定には、
`asset_account_available_settings`
を使用する。

現在の資産口座の状態だけを使用して、
過去の対象年月について
登録可否を判定しない。

対象年月時点で
月末資産管理対象ではない資産口座には、
月末資産残高を登録できない。

この判定は、
単項目バリデーションではなく
業務ルールとして扱う。

---

### 11.8 残高記録単位

指定された資産口座の
残高記録単位が、
資産口座単位であることを確認する。

```text
balance_recording_unit = 口座単位
```

商品単位で残高を記録する
資産口座については、
本APIで月末資産残高を登録できない。

```text
balance_recording_unit = 商品単位
    → 登録不可
```

商品単位の資産口座については、
VAL-002 商品別月末評価額登録APIを使用する。

残高記録単位の判定は、
単項目バリデーションではなく
業務ルールとして扱う。

---

### 11.9 重複登録

以下の組み合わせで、
月末資産残高が
すでに登録されていないことを確認する。

```text
month_end_asset_snapshot_id = snapshotId
AND
asset_account_id = assetAccountId
```

すでに月末資産残高が
登録されている場合は、
新しいレコードを登録しない。

既存の月末資産残高を
変更する場合は、
BAL-003 月末資産残高更新APIを使用する。

重複登録の判定は、
単項目バリデーションではなく
業務ルールとして扱う。

データベース側でも、
以下の組み合わせに対する
一意性を保証する。

```text
month_end_asset_snapshot_id
+
asset_account_id
```

---

### 11.10 X-User-Id

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

### 11.11 未定義項目

リクエストボディに、
定義されていない項目が
指定された場合の扱いは、
API共通方針に従う。

本APIで定義する
リクエスト項目は、
以下のみとする。

```text
assetAccountId
balance
```

利用者ID、
月末資産状況ID、
対象年月および
確定状態は、
リクエストボディから受け付けない。

---

### 11.12 業務状態に依存する検証

以下は、
Form Requestによる
単項目バリデーションではなく、
業務ルールとして扱う。

- 月末資産状況が操作対象利用者に属していること
- 月末資産状況が未確定であること
- 資産口座が操作対象利用者に属していること
- 資産口座が対象年月時点で月末資産管理対象であること
- 資産口座の残高記録単位が口座単位であること
- 同一の月末資産状況・資産口座について月末資産残高が未登録であること

これらの判定は、
UseCaseおよびQueryで実施する。

データベース制約で保証できる条件については、
アプリケーション側の検証に加えて
データベース側でも整合性を保証する。

---

## 12. 業務ルール

- 月末資産残高は、操作対象利用者に属する月末資産状況に対してのみ登録できる。
- 他の利用者に属する月末資産状況へ月末資産残高を登録できない。
- 登録対象の資産口座は、操作対象利用者に属している必要がある。
- 他の利用者に属する資産口座へ月末資産残高を登録できない。
- 月末資産状況が未確定の場合のみ登録できる。
- 確定済みの月末資産状況には登録できない。
- 登録対象の資産口座は、対象年月時点で月末資産管理対象である必要がある。
- 登録対象の資産口座は、残高記録単位が資産口座単位である必要がある。
- 残高記録単位が商品単位の資産口座には、本APIで月末資産残高を登録できない。
- 商品単位の資産口座については、VAL-002 商品別月末評価額登録APIを使用する。
- 同一の月末資産状況・資産口座の組み合わせについて、月末資産残高は1件のみ登録できる。
- すでに月末資産残高が登録されている場合は、新規登録できない。
- 既存の月末資産残高を変更する場合は、BAL-003 月末資産残高更新APIを使用する。
- `balance = 0`は有効な月末資産残高として登録できる。
- 月末資産残高は日本円の整数値として登録する。
- 月末資産残高の登録によって、月末資産状況を自動的に確定しない。
- 月末資産残高の登録によって、商品別月末評価額を登録・更新しない。
- 月末資産残高の登録によって、目的達成判定を自動実行しない。
- 月末資産残高の登録によって、過去の目的達成判定履歴を変更しない。

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
資産口座取得
    ↓
存在・利用者境界確認
    ↓
対象年月時点の利用可能状態確認
    ↓
残高記録単位確認
    ↓
重複登録確認
    ↓
月末資産残高登録
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

資産口座の取得条件は、
以下とする。

```text
id = assetAccountId
AND
user_id = 操作対象利用者ID
```

重複登録の確認条件は、
以下とする。

```text
month_end_asset_snapshot_id = snapshotId
AND
asset_account_id = assetAccountId
```

登録条件をすべて満たした場合のみ、
新しい月末資産残高を登録する。

---

## 14. 月末資産状況の扱い

月末資産残高を登録できるのは、
月末資産状況が
未確定の場合のみとする。

```text
confirmed = false
    → 登録可能

confirmed = true
    → 登録不可
```

確定済みの場合は、
月末資産状況を確定解除した後に
登録する必要がある。

本APIによる登録成功後も、
月末資産状況は未確定のままとする。

```text
登録前
confirmed = false

    ↓ 月末資産残高登録

登録後
confirmed = false
```

本APIでは、
月末資産状況の
`confirmed`を更新しない。

---

## 15. 資産口座の扱い

登録対象となる資産口座は、
以下の条件をすべて満たす必要がある。

```text
操作対象利用者に属している
AND
対象年月時点で月末資産管理対象である
AND
残高記録単位が口座単位である
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
資産口座のみ登録対象とする。

現在の利用状態だけを使用して、
登録可否を判定しない。

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

商品単位の資産口座について、
BAL-002で口座残高を
登録してはならない。

これにより、
同一資産口座について

```text
口座残高
+
商品別評価額
```

を重複して管理することを防止する。

---

## 17. 重複登録

同一の月末資産状況と
資産口座の組み合わせについて、
月末資産残高は1件のみ保持する。

```text
month_end_asset_snapshot_id
+
asset_account_id
```

上記の組み合わせが
すでに存在する場合は、
新しい月末資産残高を登録しない。

例えば、

```text
snapshotId = 12
assetAccountId = 3
```

の月末資産残高が
すでに存在する場合、

同じ組み合わせで
BAL-002を再実行しても
新しいレコードは作成しない。

既存残高を変更する場合は、
BAL-003 月末資産残高更新APIを使用する。

データベース側でも、
以下の組み合わせに
UNIQUE制約を設定し、
重複登録を防止する。

```text
month_end_asset_snapshot_id
+
asset_account_id
```

---

## 18. 0円の扱い

`balance = 0`は、
有効な月末資産残高として扱う。

```json
{
  "assetAccountId": "3",
  "balance": 0
}
```

これは、
未登録とは異なる。

```text
月末資産残高レコードなし
    → 未登録

月末資産残高レコードあり
balance = 0
    → 0円で登録済み
```

登録処理では、
`0`を未入力として扱ってはならない。

---

## 19. 成功レスポンス

### 19.1 HTTPステータス

```http
201 Created
```

月末資産残高の
新規登録に成功した場合は、
`201 Created`を返却する。

---

### 19.2 レスポンスボディ

```json
{
  "data": {
    "assetAccountId": "3",
    "balance": 1200000
  }
}
```

0円を登録した場合：

```json
{
  "data": {
    "assetAccountId": "3",
    "balance": 0
  }
}
```

登録後の月末資産残高を
`data`オブジェクトとして返却する。

---

## 20. レスポンス項目

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `data` | object | × | 登録した月末資産残高 |
| `data.assetAccountId` | string | × | 資産口座ID |
| `data.balance` | integer | × | 登録した月末資産残高 |

`assetAccountId`は、
API共通方針に従って
文字列として返却する。

`balance`は、
日本円の整数値として返却する。

```json
{
  "balance": 1200000
}
```

0円の場合も、
整数の`0`として返却する。

```json
{
  "balance": 0
}
```

本APIでは、
以下の情報は返却しない。

- `id`
- `month_end_asset_snapshot_id`
- `user_id`
- `target_year_month`
- `confirmed`
- `asset_account_name`
- `balance_recording_unit`
- `created_at`
- `updated_at`
- `month_end_holding_values`

月末資産状況そのものの情報は、
SNP-003 月末資産状況詳細取得APIで取得する。

月末資産残高一覧が必要な場合は、
BAL-001 月末資産残高一覧取得APIを使用する。

---

## 21. エラーレスポンス

異常時は、
API共通方針で定めた
共通エラーレスポンス形式を使用する。

本APIでは、
利用者コンテキストの不正、
`snapshotId`または
`assetAccountId`の不正、
月末資産状況または資産口座の不存在、
確定済み月末資産状況への登録、
対象年月時点で利用できない資産口座、
残高記録単位の不一致、
重複登録および
想定外のサーバーエラーを扱う。

エラーが発生した場合は、
月末資産残高を登録しない。

---

### 21.1 利用者コンテキストが指定されていない場合

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

### 21.2 利用者ID形式が不正な場合

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

### 21.3 利用者が存在しない場合

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

### 21.4 バリデーションエラー

`snapshotId`、
`assetAccountId`または
`balance`が
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
        "field": "balance",
        "reason": "min",
        "message": "月末資産残高は0以上で指定してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

以下のような場合を含む。

- `snapshotId`の形式が不正
- `assetAccountId`が未指定
- `assetAccountId`が`null`
- `assetAccountId`の形式が不正
- `balance`が未指定
- `balance`が`null`
- `balance`がintegerではない
- `balance`が負数
- `balance`が保持可能な範囲を超えている

---

### 21.5 月末資産状況が存在しない場合

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

### 21.6 月末資産状況が確定済みの場合

指定された月末資産状況が
確定済みの場合は、
`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`
を返却する。

```json
{
  "error": {
    "code": "MONTH_END_ASSET_SNAPSHOT_CONFIRMED",
    "message": "確定済みの月末資産状況には月末資産残高を登録できません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

確定済みデータを修正する必要がある場合は、
先にSNP-005 月末資産状況確定解除APIを実行する。

---

### 21.7 資産口座が存在しない場合

以下の条件を満たす
資産口座が存在しない場合は、
`ASSET_ACCOUNT_NOT_FOUND`
を返却する。

```text
id = assetAccountId
AND
user_id = 操作対象利用者ID
```

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_NOT_FOUND",
    "message": "指定された資産口座が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

以下の場合を含む。

- 指定された資産口座が存在しない
- 指定された資産口座が他の利用者に属している

他利用者の資産口座が
存在すること自体を
レスポンスから判別できないようにする。

---

### 21.8 対象年月時点で利用できない資産口座の場合

指定された資産口座が、
月末資産状況の対象年月時点で
月末資産管理対象ではない場合は、
`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`
を返却する。

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH",
    "message": "指定された資産口座は対象年月の月末資産管理対象ではありません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

現在の利用状態ではなく、
月末資産状況の
`target_year_month`時点の状態をもとに判定する。

---

### 21.9 残高記録単位が口座単位ではない場合

指定された資産口座の
残高記録単位が
商品単位の場合は、
`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`
を返却する。

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH",
    "message": "指定された資産口座は口座単位の残高登録対象ではありません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

商品単位の資産口座については、
VAL-002 商品別月末評価額登録APIを使用する。

---

### 21.10 月末資産残高がすでに登録されている場合

以下の組み合わせで
月末資産残高が
すでに存在する場合は、
`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`
を返却する。

```text
month_end_asset_snapshot_id = snapshotId
AND
asset_account_id = assetAccountId
```

```json
{
  "error": {
    "code": "MONTH_END_ASSET_BALANCE_ALREADY_EXISTS",
    "message": "指定された資産口座の月末資産残高はすでに登録されています。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

既存レコードを
本APIで上書きしない。

残高を変更する場合は、
BAL-003 月末資産残高更新APIを使用する。

アプリケーション側の
重複確認で検知した場合と、
データベースのUNIQUE制約違反で
検知した場合は、
同じエラーコードへ変換する。

---

### 21.11 想定外のエラーが発生した場合

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

## 22. HTTPステータス

| HTTPステータス | 条件 |
|---:|---|
| `201 Created` | 月末資産残高の登録に成功した |
| `400 Bad Request` | 利用者コンテキストが未指定、または利用者IDの形式が不正である |
| `404 Not Found` | 指定された利用者、月末資産状況または資産口座が存在しない |
| `409 Conflict` | 現在の業務状態では月末資産残高を登録できない、またはすでに登録済みである |
| `422 Unprocessable Entity` | リクエスト項目のバリデーションエラー |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

---

### 22.1 201の扱い

新しい月末資産残高の
登録に成功した場合は、
`201 Created`
を返却する。

`balance = 0`の場合も、
正常な新規登録として
`201 Created`
を返却する。

---

### 22.2 400の扱い

以下の場合は、
`400 Bad Request`
を返却する。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正である

---

### 22.3 404の扱い

以下の場合は、
`404 Not Found`
を返却する。

- 指定された利用者が存在しない
- 指定された利用者が論理削除されている
- 指定された月末資産状況が存在しない
- 指定された月末資産状況が他の利用者に属している
- 指定された資産口座が存在しない
- 指定された資産口座が他の利用者に属している

他利用者に属するデータを指定した場合も、
対象リソースの存在を公開しない。

---

### 22.4 409の扱い

リクエスト形式および
対象リソースは正しいが、
現在の業務状態では
月末資産残高を登録できない場合は、
`409 Conflict`
を返却する。

以下の場合を含む。

- 月末資産状況が確定済み
- 資産口座が対象年月時点で月末資産管理対象ではない
- 資産口座の残高記録単位が口座単位ではない
- 同一月末資産状況・資産口座の月末資産残高がすでに存在する

---

### 22.5 422の扱い

以下の場合は、
`422 Unprocessable Entity`
を返却する。

- `snapshotId`の形式が不正
- `assetAccountId`が未指定
- `assetAccountId`の形式が不正
- `balance`が未指定
- `balance`の型が不正
- `balance`が負数
- `balance`が許容範囲外

---

## 23. エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `USER_CONTEXT_REQUIRED` | 400 | `X-User-Id`が指定されていない | × |
| `INVALID_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `USER_NOT_FOUND` | 404 | 指定された利用者が存在しない、または論理削除されている | × |
| `VALIDATION_ERROR` | 422 | `snapshotId`、`assetAccountId`または`balance`が不正である | × |
| `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` | 404 | 月末資産状況が存在しない、または他利用者に属している | × |
| `MONTH_END_ASSET_SNAPSHOT_CONFIRMED` | 409 | 月末資産状況が確定済みである | × |
| `ASSET_ACCOUNT_NOT_FOUND` | 404 | 資産口座が存在しない、または他利用者に属している | × |
| `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH` | 409 | 資産口座が対象年月時点で月末資産管理対象ではない | × |
| `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH` | 409 | 資産口座の残高記録単位が口座単位ではない | × |
| `MONTH_END_ASSET_BALANCE_ALREADY_EXISTS` | 409 | 同一月末資産状況・資産口座の残高がすでに登録されている | × |
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

## 24. 冪等性

本APIは、
HTTP POSTを使用して
新しい月末資産残高リソースを作成する。

そのため、
HTTPメソッドとしては
冪等ではない。

ただし、
同一の月末資産状況・資産口座について
複数の月末資産残高を登録することは禁止する。

同じ内容のリクエストを
再度実行した場合は、
既存レコードを返却したり、
上書きしたりせず、

`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`

を返却する。

```text
1回目
POST /month-end-asset-snapshots/12/asset-balances

assetAccountId = 3
balance = 1200000

    ↓

201 Created


2回目
POST /month-end-asset-snapshots/12/asset-balances

assetAccountId = 3
balance = 1200000

    ↓

409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

2回目の`balance`が
1回目と異なる場合も、
本APIでは上書きしない。

```text
登録済み
balance = 1200000

    ↓

BAL-002で
balance = 1300000を送信

    ↓

409 Conflict
```

既存残高を変更する場合は、
BAL-003 月末資産残高更新APIを使用する。

同時に複数の登録要求が
実行された場合は、
以下の組み合わせに対する
データベースのUNIQUE制約によって
1件のみ登録されることを保証する。

```text
month_end_asset_snapshot_id
+
asset_account_id
```

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

## 25. 関連テーブル

### 25.1 month_end_asset_snapshots

対象年月ごとの
月末資産状況および確定状態を保持する。

本APIでは、
月末資産残高の登録先となる
月末資産状況を特定するために参照する。

使用する主なカラムは、
以下とする。

| カラム | 用途 |
|---|---|
| `id` | 月末資産状況ID |
| `user_id` | 利用者境界 |
| `target_year_month` | 対象年月 |
| `confirmed` | 登録可否判定 |

取得条件は、
以下とする。

```text
id = snapshotId
AND
user_id = 操作対象利用者ID
```

月末資産状況が
確定済みの場合は、
月末資産残高を登録できない。

本APIでは、
`month_end_asset_snapshots`を更新しない。

---

### 25.2 asset_accounts

利用者が所有する
資産口座を保持する。

本APIでは、
登録対象となる資産口座が
操作対象利用者に属していること、
および残高記録単位が
口座単位であることを確認するために参照する。

使用する主なカラムは、
以下とする。

| カラム | 用途 |
|---|---|
| `id` | 資産口座ID |
| `user_id` | 利用者境界 |
| `balance_recording_unit` | 口座単位・商品単位の判定 |

取得条件は、
以下とする。

```text
id = assetAccountId
AND
user_id = 操作対象利用者ID
```

商品単位で残高を記録する
資産口座については、
本APIで月末資産残高を登録しない。

---

### 25.3 asset_account_available_settings

資産口座について、
対象年月時点で
月末資産管理対象となるかを
判定するための設定を保持する。

本APIでは、
月末資産状況の
`target_year_month`時点で
登録対象の資産口座が
利用可能であることを確認するために参照する。

現在の資産口座の状態だけではなく、
対象年月時点の状態を判定するために使用する。

本APIでは、
`asset_account_available_settings`を更新しない。

---

### 25.4 month_end_asset_balances

資産口座単位の
月末資産残高を保持する。

本APIで
新規レコードを登録するテーブルである。

使用する主なカラムは、
以下とする。

| カラム | 用途 |
|---|---|
| `id` | 月末資産残高ID |
| `month_end_asset_snapshot_id` | 月末資産状況との紐付け |
| `asset_account_id` | 資産口座との紐付け |
| `balance` | 月末資産残高 |
| `created_at` | 登録日時 |
| `updated_at` | 更新日時 |

登録時は、
以下の値を設定する。

```text
month_end_asset_snapshot_id = snapshotId
asset_account_id            = assetAccountId
balance                     = リクエストのbalance
```

同一の月末資産状況・資産口座について
複数の月末資産残高が登録されないよう、
以下の組み合わせに
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
asset_account_id
```

本APIでは、
既存の月末資産残高を更新しない。

---

### 25.5 users

操作対象となる
利用者を保持する。

`X-User-Id`で指定された利用者が
存在することを確認するために参照する。

本APIでは、
`users`を更新しない。

---

### 25.6 holding_assets

本APIでは、
`holding_assets`を参照しない。

保有商品は、
残高記録単位が商品単位の
資産口座について使用する。

商品別月末評価額の登録は、
VAL-002 商品別月末評価額登録APIで行う。

---

### 25.7 month_end_holding_values

本APIでは、
`month_end_holding_values`を参照しない。

商品別月末評価額の登録は、
VAL-002 商品別月末評価額登録APIで行う。

---

### 25.8 assessment_histories

本APIでは、
`assessment_histories`を
参照・登録・更新・削除しない。

月末資産残高を登録しても、
過去の目的達成判定履歴は変更しない。

また、
月末資産残高の登録を契機として
目的達成判定を自動実行しない。

---

## 26. 関連する機能要件

- 月末資産管理
  - 資産口座単位で残高を記録する資産口座について、対象年月の月末資産残高を登録できる
  - 月末資産残高は日本円の整数値として管理する
  - `0円`を有効な月末資産残高として登録できる
  - 同一月末資産状況・資産口座の月末資産残高は1件のみ保持する

- 資産口座管理
  - 資産口座ごとに残高記録単位を保持する
  - 残高記録単位が口座単位の場合のみ、本APIで月末資産残高を登録する
  - 対象年月時点で月末資産管理対象となる資産口座のみ登録対象とする

- 月末資産状況の確定
  - 未確定の月末資産状況に対して月末資産残高を登録できる
  - 確定済み月末資産状況の残高を変更する場合は、先に確定解除する
  - 月末資産残高の登録だけでは月末資産状況を自動確定しない

- 目的達成判定
  - 月末資産残高の登録を契機として目的達成判定を自動実行しない
  - 保存済みの目的達成判定履歴は変更しない

- 利用者境界
  - 操作対象利用者に属する月末資産状況にのみ登録できる
  - 操作対象利用者に属する資産口座にのみ登録できる
  - 他の利用者に属するリソースの存在を外部へ公開しない

具体的な章番号は、
`functional-requirements.md`の
最新定義に従う。

---

## 27. テスト観点

### 27.1 正常系

- 有効な入力値で月末資産残高を登録できること
- `201 Created`で返却されること
- `month_end_asset_balances`へ1件登録されること
- `month_end_asset_snapshot_id`に指定した月末資産状況IDが設定されること
- `asset_account_id`に指定した資産口座IDが設定されること
- `balance`に指定した金額が設定されること
- 操作対象利用者に属するデータとして登録されること
- 月末資産状況の`confirmed`が変更されないこと
- 商品別月末評価額が登録されないこと
- 目的達成判定が自動実行されないこと

---

### 27.2 snapshotId

- 正しい`snapshotId`を指定して登録できること
- `snapshotId = 1`を指定できること
- `snapshotId = 0`でバリデーションエラーとなること
- 負数でバリデーションエラーとなること
- 小数でバリデーションエラーとなること
- 文字列`abc`でバリデーションエラーとなること
- ID形式不正時に`422 Unprocessable Entity`となること
- ID形式不正時に`VALIDATION_ERROR`となること

---

### 27.3 assetAccountId

- 正しい`assetAccountId`を指定して登録できること
- `assetAccountId`未指定で`422 Unprocessable Entity`となること
- `null`でバリデーションエラーとなること
- 空文字でバリデーションエラーとなること
- `"0"`でバリデーションエラーとなること
- 負数形式でバリデーションエラーとなること
- `"abc"`でバリデーションエラーとなること
- 数値型で送信した場合の扱いがAPI仕様と一致すること

---

### 27.4 balance

以下を確認する。

- 正の整数を登録できること
- `balance = 0`を登録できること
- `balance = 1`を登録できること
- 保持可能な最大値を登録できること
- `balance`未指定でバリデーションエラーとなること
- `balance = null`でバリデーションエラーとなること
- `balance = -1`でバリデーションエラーとなること
- 小数値でバリデーションエラーとなること
- 数値文字列でバリデーションエラーとなること
- 保持可能な範囲を超える値でバリデーションエラーとなること

---

### 27.5 月末資産状況の存在確認

- 存在する月末資産状況へ登録できること
- 存在しない`snapshotId`で`404 Not Found`となること
- 存在しない場合に`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`となること
- 存在しない場合に月末資産残高が登録されないこと

---

### 27.6 月末資産状況の利用者境界

- 操作対象利用者に属する月末資産状況へ登録できること
- 他利用者に属する月末資産状況へ登録できないこと
- 他利用者の`snapshotId`で`404 Not Found`となること
- 他利用者の場合に`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`となること
- 他利用者の月末資産状況が存在することをレスポンスから判別できないこと
- 月末資産状況の検索条件に`user_id`が含まれていること

---

### 27.7 資産口座の存在確認

- 存在する資産口座を指定して登録できること
- 存在しない`assetAccountId`で`404 Not Found`となること
- 存在しない場合に`ASSET_ACCOUNT_NOT_FOUND`となること
- 存在しない資産口座を指定した場合に月末資産残高が登録されないこと

---

### 27.8 資産口座の利用者境界

- 操作対象利用者に属する資産口座へ登録できること
- 他利用者に属する資産口座へ登録できないこと
- 他利用者の資産口座IDで`404 Not Found`となること
- 他利用者の場合に`ASSET_ACCOUNT_NOT_FOUND`となること
- 他利用者の資産口座が存在することをレスポンスから判別できないこと
- 資産口座の検索条件に`user_id`が含まれていること

---

### 27.9 月末資産状況の確定状態

未確定の場合：

```text
confirmed = false
```

について、

- 月末資産残高を登録できること

確定済みの場合：

```text
confirmed = true
```

について、

- 月末資産残高を登録できないこと
- `409 Conflict`となること
- `MONTH_END_ASSET_SNAPSHOT_CONFIRMED`となること
- 月末資産残高が新規作成されないこと

---

### 27.10 対象年月時点の利用可能状態

資産口座の利用可能期間が
対象年月によって異なるデータを用意する。

以下を確認する。

- 対象年月時点で月末資産管理対象の資産口座へ登録できること
- 対象年月時点で月末資産管理対象ではない資産口座へ登録できないこと
- 対象外の場合に`409 Conflict`となること
- 対象外の場合に`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`となること
- 現在の利用状態だけを使用して過去月の登録可否を判定しないこと
- 過去月では対象だが現在は対象外の資産口座へ、該当過去月で登録できること
- 現在は対象だが対象年月時点では対象外の資産口座へ登録できないこと

---

### 27.11 残高記録単位

残高記録単位が
口座単位の場合：

```text
balance_recording_unit = 口座単位
```

について、

- BAL-002で登録できること

商品単位の場合：

```text
balance_recording_unit = 商品単位
```

について、

- BAL-002で登録できないこと
- `409 Conflict`となること
- `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`となること
- 月末資産残高レコードが作成されないこと

---

### 27.12 重複登録

以下の組み合わせについて、
すでに月末資産残高が存在する状態を用意する。

```text
month_end_asset_snapshot_id = 12
asset_account_id = 3
```

以下を確認する。

- 同じ組み合わせで新規登録できないこと
- `409 Conflict`となること
- `MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`となること
- 既存レコードが更新されないこと
- 新しいレコードが作成されないこと
- `balance`が既存値と同じでもエラーとなること
- `balance`が既存値と異なっていてもBAL-002では上書きされないこと
- 変更が必要な場合はBAL-003を利用する設計になっていること

---

### 27.13 UNIQUE制約

同一の月末資産状況・資産口座について
複数の登録要求を同時実行する。

以下を確認する。

- 最終的に1件のみ登録されること
- UNIQUE制約によって重複登録が防止されること
- 1件のリクエストが`201 Created`となること
- 競合したリクエストが`409 Conflict`となること
- UNIQUE制約違反が`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`へ変換されること
- PostgreSQLの制約名や内部エラーがレスポンスへ公開されないこと

---

### 27.14 0円

`balance = 0`を送信した場合について、
以下を確認する。

- 正常に登録できること
- `201 Created`となること
- データベースへ`0`として保存されること
- レスポンスで`balance = 0`となること
- `0`が未入力として扱われないこと
- `null`へ変換されないこと

---

### 27.15 レスポンス契約

- JSONフィールド名がcamelCaseであること
- `data`がobjectで返却されること
- `assetAccountId`が文字列で返却されること
- `balance`がintegerで返却されること
- `balance = 0`が整数の`0`として返却されること
- 月末資産残高IDがレスポンスへ含まれないこと
- `snapshotId`がレスポンスへ含まれないこと
- `userId`がレスポンスへ含まれないこと
- `targetYearMonth`がレスポンスへ含まれないこと
- `confirmed`がレスポンスへ含まれないこと
- 資産口座名がレスポンスへ含まれないこと
- `createdAt`がレスポンスへ含まれないこと
- `updatedAt`がレスポンスへ含まれないこと
- DB内部のsnake_caseのカラム名がそのまま公開されないこと

---

### 27.16 トランザクション

- 業務条件の最終確認から月末資産残高登録までが同一トランザクション内で実行されること
- 登録途中で例外が発生した場合にロールバックされること
- 業務条件を満たさない場合に月末資産残高が登録されないこと
- 登録失敗時に不完全なレコードが残らないこと

---

### 27.17 副作用

本API実行によって、
以下が変更されないことを確認する。

- `month_end_asset_snapshots.confirmed`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_holding_values`
- `assessment_histories`

以下の副作用が
発生しないことを確認する。

- 月末資産状況を自動確定しない
- 商品別月末評価額を自動登録しない
- 目的達成判定を自動実行しない
- 過去の目的達成判定履歴を変更しない

---

### 27.18 エラー時

以下のエラー時に、
月末資産残高が登録されないことを確認する。

- `USER_CONTEXT_REQUIRED`
- `INVALID_USER_ID`
- `USER_NOT_FOUND`
- `VALIDATION_ERROR`
- `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
- `MONTH_END_ASSET_SNAPSHOT_CONFIRMED`
- `ASSET_ACCOUNT_NOT_FOUND`
- `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`
- `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`
- `MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`

---

### 27.19 異常系

- 想定外の例外で`500 Internal Server Error`となること
- 想定外の例外発生時にトランザクションがロールバックされること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログに同じリクエストIDが記録されること
- SQLがレスポンスへ含まれないこと
- PostgreSQLの制約名がレスポンスへ含まれないこと
- スタックトレースがレスポンスへ含まれないこと
- 内部例外メッセージがレスポンスへ含まれないこと

---

## 28. Laravel実装方針

### 28.1 Action

HTTPリクエストを受け付け、
月末資産状況ID、
登録対象の資産口座ID、
月末資産残高および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの入力値を受け取り、
月末資産残高登録UseCaseを呼び出す。

UseCaseから受け取った登録結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- `snapshotId`の形式検証
- リクエストボディの単項目バリデーション
- 利用者境界の判定
- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 資産口座の存在確認
- 対象年月時点の利用可否判定
- 残高記録単位の判定
- 重複登録確認
- 月末資産残高の登録
- トランザクション制御
- レスポンス生成処理

---

### 28.2 UseCase

月末資産残高登録の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 月末資産状況IDを受け取る
- 登録対象の資産口座IDを受け取る
- 月末資産残高を受け取る
- 月末資産状況を取得する
- 月末資産状況が未確定であることを確認する
- 資産口座を取得する
- 対象年月時点で資産口座が月末資産管理対象であることを確認する
- 資産口座の残高記録単位が口座単位であることを確認する
- 同一月末資産状況・資産口座の月末資産残高が未登録であることを確認する
- 月末資産残高を登録する
- 登録結果を返却する

指定された月末資産状況が存在しない場合、
または操作対象利用者に属していない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

指定された資産口座が存在しない場合、
または操作対象利用者に属していない場合は、
`ASSET_ACCOUNT_NOT_FOUND`
として扱う。

---

### 28.3 Form Request / DTO

リクエストボディの
形式および単項目バリデーションを担当する。

検証対象は、
以下とする。

- `assetAccountId`
- `balance`

Form Requestでは、
以下を検証する。

#### assetAccountId

- 必須であること
- `null`ではないこと
- 文字列であること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

#### balance

- 必須であること
- `null`ではないこと
- integerであること
- 0以上であること
- データベースで保持可能な範囲であること

業務状態に依存する以下の検証は、
Form Requestでは行わない。

- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 資産口座の存在確認
- 対象年月時点の利用可否確認
- 残高記録単位確認
- 重複登録確認

検証済みの入力値は、
入力用DTOへ変換して
UseCaseへ渡す。

例：

```php
final readonly class CreateMonthEndAssetBalanceInput
{
    public function __construct(
        public string $assetAccountId,
        public int $balance,
    ) {
    }
}
```

---

### 28.4 Query

月末資産残高登録に必要な
データ取得を担当する。

主な取得対象は、
以下とする。

- `month_end_asset_snapshots`
- `asset_accounts`
- `asset_account_available_settings`
- `month_end_asset_balances`

本APIでは、
原則として以下を参照しない。

- `holding_assets`
- `month_end_holding_values`
- `assessment_histories`

---

### 28.5 月末資産状況取得

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

### 28.6 確定状態確認

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

月末資産残高を登録するために、
本API内で自動的に
確定解除してはならない。

---

### 28.7 資産口座取得

登録対象となる資産口座は、
必ず利用者境界を含めて取得する。

```php
$assetAccount = AssetAccount::query()
    ->where(
        'id',
        $input->assetAccountId,
    )
    ->where(
        'user_id',
        $userId,
    )
    ->first();
```

以下のように、
資産口座IDだけで
取得してはならない。

```php
AssetAccount::find(
    $input->assetAccountId,
);
```

取得できなかった場合は、
以下を区別せず
`ASSET_ACCOUNT_NOT_FOUND`
として扱う。

- 資産口座が存在しない
- 他の利用者に属している

---

### 28.8 対象年月時点の利用可否判定

月末資産状況の
`target_year_month`を使用して、
資産口座が対象年月時点で
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
対象年月時点で利用可能か
```

対象年月時点で
月末資産管理対象ではない場合は、

`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`

として扱う。

現在の資産口座の状態だけで
過去月の登録可否を
判断してはならない。

---

### 28.9 残高記録単位確認

資産口座の
`balance_recording_unit`が
口座単位であることを確認する。

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::ACCOUNT
) {
    throw new
        AssetAccountBalanceRecordingUnitMismatchException();
}
```

条件を満たさない場合は、

`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`

として扱う。

商品単位の資産口座について、
本APIで月末資産残高を
登録してはならない。

---

### 28.10 重複確認

同一の月末資産状況・資産口座について、
月末資産残高が
すでに存在しないことを確認する。

```php
$exists = MonthEndAssetBalance::query()
    ->where(
        'month_end_asset_snapshot_id',
        $snapshot->id,
    )
    ->where(
        'asset_account_id',
        $assetAccount->id,
    )
    ->exists();
```

存在する場合は、

`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`

として扱う。

アプリケーション側の
重複確認だけでは
同時実行時の重複を完全には防止できないため、
データベースのUNIQUE制約も使用する。

---

### 28.11 Repository

月末資産残高の
新規登録を担当する。

登録対象は、
以下とする。

- `month_end_asset_snapshot_id`
- `asset_account_id`
- `balance`

登録例：

```php
return MonthEndAssetBalance::create([
    'month_end_asset_snapshot_id'
        => $snapshot->id,
    'asset_account_id'
        => $assetAccount->id,
    'balance'
        => $input->balance,
]);
```

`id`、
`created_at`および
`updated_at`は、
Laravelおよび
データベース側で設定する。

Repositoryでは、
以下の処理は行わない。

- 月末資産状況の確定
- 月末資産状況の確定解除
- 商品別月末評価額の登録
- 目的達成判定の実行

---

### 28.12 トランザクション

業務条件の最終確認から
月末資産残高登録までを、
1つのデータベーストランザクション内で実行する。

実装例：

```php
$balance = DB::transaction(
    function () use (
        $userId,
        $snapshotId,
        $input,
    ): MonthEndAssetBalance {
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

        $assetAccount =
            $this->assetAccountQuery
                ->findByUserAndId(
                    $userId,
                    $input->assetAccountId,
                );

        if ($assetAccount === null) {
            throw new
                AssetAccountNotFoundException();
        }

        $this->availabilityValidator->validate(
            $assetAccount,
            $snapshot->target_year_month,
        );

        $this->recordingUnitValidator->validate(
            $assetAccount,
        );

        if (
            $this->balanceQuery
                ->existsBySnapshotAndAssetAccount(
                    $snapshot->id,
                    $assetAccount->id,
                )
        ) {
            throw new
                MonthEndAssetBalanceAlreadyExistsException();
        }

        return $this->repository->create(
            $snapshot,
            $assetAccount,
            $input->balance,
        );
    },
);
```

処理途中で例外が発生した場合は、
月末資産残高を登録しない。

---

### 28.13 UNIQUE制約

以下の組み合わせに、
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
asset_account_id
```

複数リクエストが
同時に重複確認を通過しても、
データベース側で
複数レコードの作成を防止する。

UNIQUE制約違反が発生した場合は、
PostgreSQLの例外をそのまま返却せず、

`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`

へ変換する。

アプリケーション側の事前確認と
データベース制約の
両方を使用する。

---

### 28.14 Mass Assignment

クライアントから受け取った値を
そのままModelへ渡してはならない。

以下のような実装は避ける。

```php
MonthEndAssetBalance::create(
    $request->all(),
);
```

登録値は、
以下から明示的に組み立てる。

```text
snapshotId
    → 検証済み月末資産状況

assetAccountId
    → 検証済み資産口座

balance
    → Form Request / DTOの検証済み値
```

これにより、
クライアントから
意図しない外部キーや
サーバー管理項目を指定されることを防止する。

---

### 28.15 0円の扱い

`balance = 0`は、
有効な値として扱う。

以下のような
truthy / falsy判定は使用しない。

```php
if (! $input->balance) {
    // 0円も未入力扱いになるため使用しない
}
```

Form Requestでは、
`required`と
整数・最小値の検証を組み合わせ、
0円を正常値として許容する。

---

### 28.16 Responder

UseCaseから受け取った
登録後の月末資産残高を、
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
- 対象年月時点の利用可否判定
- 残高記録単位の判定
- 重複登録判定
- 月末資産残高の登録

---

### 28.17 API Resource

データベースカラムを直接返却せず、
API Resourceを利用して
APIレスポンス形式へ変換する。

変換例：

```php
return [
    'assetAccountId'
        => (string) $this->asset_account_id,
    'balance'
        => (int) $this->balance,
];
```

正常終了時は、
`balance`が必ずintegerとなる。

`balance = 0`の場合も
そのまま`0`を返却する。

以下の項目は、
レスポンスへ含めない。

- 月末資産残高ID
- `snapshotId`
- `userId`
- `targetYearMonth`
- `confirmed`
- 資産口座名
- `createdAt`
- `updatedAt`

---

### 28.18 Middleware

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

### 28.19 Eloquentモデル

`MonthEndAssetBalance`モデルは、
`month_end_asset_balances`
テーブルへ対応する。

主に以下の属性を使用する。

```text
id
month_end_asset_snapshot_id
asset_account_id
balance
```

`balance`は、
日本円の整数値として扱う。

必要に応じて、
integer castを設定する。

```php
protected function casts(): array
{
    return [
        'balance' => 'integer',
    ];
}
```

---

### 28.20 業務ルール判定クラス

対象年月時点の利用可否判定や
残高記録単位の判定を、
UseCaseへ直接書き込み続けない。

例えば、
以下の責務へ分離してよい。

```text
AssetAccountAvailabilityValidator
    → 対象年月時点の利用可否判定

AssetAccountBalanceRecordingUnitValidator
    → 口座単位であることの判定
```

各Validatorは、
HTTPレスポンス生成や
データベース登録を行わない。

---

### 28.21 例外変換

LaravelおよびPostgreSQLの内部例外は、
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
| 資産口座不存在 | `ASSET_ACCOUNT_NOT_FOUND` |
| 利用者境界外の資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 対象年月時点で利用不可 | `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH` |
| 残高記録単位不一致 | `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH` |
| 月末資産残高重複 | `MONTH_END_ASSET_BALANCE_ALREADY_EXISTS` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

UNIQUE制約違反については、
対象となる制約を判別し、
`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`
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
- 資産口座ID
- 対象年月
- 独自エラーコード
- リクエストID

月末資産残高の具体的な金額は、
不要にエラーログへ出力しない。
