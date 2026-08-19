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

## 27. 設計上の補足

### 27.1 月末資産状況を起点として登録する

BAL-002では、月末資産残高を資産口座単独のリソースとして登録するのではなく、指定された月末資産状況に対する子リソースとして登録する。

```text
month_end_asset_snapshots
    ↓
month_end_asset_balances
```

そのため、エンドポイントも以下の構造とする。

```http
POST /api/v1/month-end-asset-snapshots/{snapshotId}/asset-balances
```

月末資産残高の対象年月はリクエストから直接受け取らず、`snapshotId`で特定した月末資産状況の`target_year_month`を使用する。

これにより、

```text
snapshotId
    ↓
月末資産状況
    ↓
target_year_month
```

という関係を一意にし、リクエストで指定した年月と月末資産状況の年月が矛盾する状態を防止する。

---

### 27.2 利用者IDを月末資産残高へ重複保持しない

`month_end_asset_balances`には、`user_id`を保持しない。

利用者境界は、

```text
month_end_asset_balances
    ↓
month_end_asset_snapshots
    ↓
user_id
```

および、

```text
month_end_asset_balances
    ↓
asset_accounts
    ↓
user_id
```

の関連によって保証する。

登録時には、月末資産状況と資産口座の両方について、操作対象利用者に属していることを確認する。

これにより、`month_end_asset_balances.user_id`を追加した場合に発生し得る、

```text
snapshot.user_id
asset_account.user_id
balance.user_id
```

の不整合を避ける。

---

### 27.3 対象年月時点の状態を基準とする

資産口座がBAL-002の登録対象であるかどうかは、現在の資産口座状態ではなく、月末資産状況の`target_year_month`時点の状態を基準として判定する。

判定には、`asset_account_available_settings`を使用する。

```text
snapshot.target_year_month
        +
assetAccountId
        ↓
対象年月時点で月末資産管理対象か判定
```

例えば、現在は利用対象外となっている資産口座であっても、対象年月時点では利用対象であった場合、その過去月については登録可能となり得る。

反対に、現在は利用対象であっても、対象年月時点では利用対象ではなかった場合、その対象月については登録できない。

過去データの整合性を維持するため、現在状態だけを参照して登録可否を判断しない。

---

### 27.4 口座単位と商品単位の残高管理を混在させない

資産口座の`balance_recording_unit`によって、月末資産の記録方法を分離する。

```text
口座単位
    ↓
BAL-002 月末資産残高登録

商品単位
    ↓
VAL-002 商品別月末評価額登録
```

商品単位の資産口座についてBAL-002による口座残高登録を許可すると、

```text
口座残高
    +
商品別評価額合計
```

という二重管理が発生する可能性がある。

そのため、BAL-002では`balance_recording_unit`が口座単位であることを業務ルールとして検証し、商品単位の場合は登録を拒否する。

---

### 27.5 重複登録と更新を明確に分離する

同一の月末資産状況・資産口座について、月末資産残高は1件のみ保持する。

一意性の単位は以下とする。

```text
month_end_asset_snapshot_id
+
asset_account_id
```

すでに月末資産残高が存在する場合、BAL-002では既存レコードを上書きしない。

```text
未登録
    ↓
BAL-002
    ↓
新規登録

登録済み
    ↓
BAL-002
    ↓
409 Conflict

登録済み
    ↓
BAL-003
    ↓
更新
```

これにより、

* BAL-002は新規登録
* BAL-003は既存残高の更新

という責務を明確に分離する。

---

### 27.6 アプリケーション側確認とUNIQUE制約を併用する

重複登録については、UseCaseで事前確認を行う。

ただし、事前確認だけでは、複数リクエストが同時に実行された場合に以下の競合が発生し得る。

```text
Request A
    ↓
重複なし

Request B
    ↓
重複なし

Request A → INSERT
Request B → INSERT
```

そのため、データベース側でも、

```text
month_end_asset_snapshot_id
+
asset_account_id
```

にUNIQUE制約を設定する。

```text
アプリケーション側
    ↓
利用者に分かりやすい事前判定

データベース側
    ↓
同時実行時を含む最終的な一意性保証
```

という二段構えとする。

#### UNIQUE制約違反が発生した場合は、PostgreSQLの例外をそのまま公開せず、`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`へ変換する。

### 27.7 `balance = 0`と未登録を区別する

`balance = 0`は、有効な月末資産残高として扱う。

```text
レコードなし
    ↓
未登録

レコードあり
balance = 0
    ↓
0円で登録済み
```

そのため、LaravelおよびReact・TypeScriptの双方で、`balance`に対して単純なtruthy / falsy判定を使用しない。

```php
if (! $input->balance) {
    // balance = 0も未入力として扱われるため使用しない
}
```

登録有無は金額ではなく、月末資産残高レコードの存在によって判定する。

この区別は、残高0円の口座を正しく月末資産状況へ反映するために必要となる。

---

### 27.8 確定済み月末資産状況を暗黙的に変更しない

BAL-002で月末資産残高を登録できるのは、月末資産状況が未確定の場合のみとする。

```text
confirmed = false
    ↓
登録可能

confirmed = true
    ↓
登録不可
```

確定済みの月末資産状況に対して登録要求が行われても、本API内で自動的に確定解除しない。

確定済みデータを変更する場合は、

```text
SNP-005 月末資産状況確定解除
    ↓
BAL-002 月末資産残高登録
```

のように、利用者が明示的に状態を変更してから登録する。

これにより、「確定済み」という業務状態が別APIの副作用によって暗黙的に変更されることを防止する。

---

### 27.9 月末資産残高登録だけでは関連処理を実行しない

BAL-002の責務は、資産口座単位の月末資産残高を新規登録することに限定する。

登録成功を契機として、以下の処理を自動実行しない。

* 月末資産状況の確定
* 月末資産状況の確定解除
* 商品別月末評価額の登録・更新
* 目的達成判定の実行
* 過去の目的達成判定履歴の変更

```text
BAL-002
    ↓
月末資産残高を登録
    ↓
終了
```

各処理を独立したAPI・ユースケースとして扱うことで、APIごとの責務を限定し、副作用を予測しやすくする。

Repositoryについても、月末資産残高の新規登録だけを担当し、月末資産状況や目的達成判定には関与しない。

---

### 27.10 クライアント指定値をそのまま永続化しない

月末資産残高登録では、クライアントから受け取ったリクエスト全体をそのままEloquent Modelへ渡さない。

以下のような実装は避ける。

```php
MonthEndAssetBalance::create(
    $request->all(),
);
```

登録値は、検証済みの情報から明示的に組み立てる。

```text
snapshotId
    ↓
検証済み月末資産状況
    ↓
month_end_asset_snapshot_id

assetAccountId
    ↓
検証済み資産口座
    ↓
asset_account_id

balance
    ↓
Form Request / DTO
    ↓
balance
```

これにより、クライアントから`user_id`、外部キー、サーバー管理項目などを意図せず上書きされることを防止する。

---

### 27.11 業務ルールと入力バリデーションを分離する

`assetAccountId`や`balance`の形式確認はForm Request / DTOで行う。

一方、以下のようなデータベース状態や対象年月に依存する判定は、UseCaseおよび業務ルール判定クラスで行う。

```text
Form Request / DTO
    ↓
入力形式の検証

UseCase / Validator
    ↓
業務状態の検証
```

業務ルールとして扱う主な内容は以下とする。

* 月末資産状況が操作対象利用者に属している
* 月末資産状況が未確定である
* 資産口座が操作対象利用者に属している
* 対象年月時点で月末資産管理対象である
* 残高記録単位が口座単位である
* 同一の月末資産状況・資産口座について未登録である

特に、対象年月時点の利用可否や残高記録単位の判定については、UseCaseへ条件分岐を書き込み続けず、必要に応じて専用Validatorへ分離する。

---

### 27.12 APIレスポンスには必要最小限の情報だけを返す

BAL-002の成功レスポンスでは、登録結果として必要な以下の情報だけを返却する。

```json
{
  "data": {
    "assetAccountId": "3",
    "balance": 1200000
  }
}
```

月末資産残高ID、月末資産状況ID、利用者ID、対象年月、確定状態、資産口座名などは返却しない。

これらの情報が必要な場合は、それぞれの参照APIを利用する。

API Resourceでは、Eloquent ModelやデータベースカラムをそのままJSONへ公開せず、API仕様に定義した項目へ明示的に変換する。

---

### 27.13 内部例外とAPIエラーを分離する

Laravel、Eloquent、PostgreSQLなどから発生した内部例外は、そのままAPIレスポンスへ公開しない。

```text
Laravel / PostgreSQL例外
    ↓
共通例外変換
    ↓
独自エラーコード
    ↓
APIレスポンス
```

例えば、月末資産残高のUNIQUE制約違反は、

```text
PostgreSQL UNIQUE制約違反
    ↓
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
    ↓
409 Conflict
```

へ変換する。

APIレスポンスには、以下を含めない。

* SQL
* スタックトレース
* PostgreSQLの制約名
* 内部例外メッセージ

一方、ログには調査に必要な範囲で、利用者ID、月末資産状況ID、資産口座ID、対象年月、独自エラーコード、リクエストIDなどを記録する。

月末資産残高そのものの金額については、不要にエラーログへ出力しない。

---

### 27.14 設計上の責務境界

BAL-002では、各レイヤーの責務を以下のように分離する。

```text
Middleware
    ↓
利用者コンテキスト・共通処理

Action
    ↓
HTTPリクエスト受付

Form Request / DTO
    ↓
入力形式検証

UseCase
    ↓
月末資産残高登録ユースケース

Query
    ↓
月末資産状況・資産口座・重複状態取得

業務ルールValidator
    ↓
対象年月時点利用可否
残高記録単位判定

Repository
    ↓
月末資産残高登録

API Resource / Responder
    ↓
APIレスポンス生成
```

月末資産残高登録に関する処理をActionやEloquent Modelへ集中させず、入力検証、データ取得、業務ルール判定、永続化、レスポンス生成を分離する。

これにより、月末資産管理に関するルールが増えた場合でも、各責務を独立して変更・テストしやすい構成とする。

## 28. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)
