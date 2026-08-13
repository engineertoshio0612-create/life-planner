# BAL-003 月末資産残高更新

## 1. 概要

操作対象となる利用者について、
指定した月末資産状況に紐づく
既存の月末資産残高を更新する。

本APIでは、
残高記録単位が
資産口座単位である資産口座について、
登録済みの月末資産残高を変更する。

月末資産残高が
まだ登録されていない場合は、
本APIでは新規登録しない。

新しい月末資産残高を登録する場合は、
BAL-002 月末資産残高登録APIを使用する。

商品単位で残高を記録する資産口座については、
本APIでは更新しない。

商品別月末評価額の更新は、
VAL-003 商品別月末評価額更新APIを使用する。

---

## 2. ユースケース

利用者は、
指定した月末資産状況に登録済みの
資産口座単位の月末資産残高を修正する。

例えば、
以下のような場合に使用する。

- 普通預金口座の入力済み月末残高を修正する
- 現金口座の入力済み月末残高を修正する
- 定期預金口座の入力済み月末残高を修正する
- 誤って入力した月末資産残高を訂正する
- 登録済みの残高を0円へ変更する

月末資産状況が
確定済みの場合は、
本APIで月末資産残高を更新できない。

確定済みの月末資産残高を修正する場合は、
SNP-005 月末資産状況確定解除APIによって
確定解除した後に更新する。

---

## 3. エンドポイント

```http
PATCH /api/v1/month-end-asset-snapshots/{snapshotId}/asset-balances/{assetAccountId}
```

---

## 4. HTTPメソッド

```text
PATCH
```

本APIは、指定した月末資産状況および資産口座に紐づく既存の月末資産残高を更新する。

月末資産残高の新規登録、月末資産状況の確定、確定解除および商品別月末評価額の更新は行わない。

---

## 5. 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、`X-User-Id`リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

指定された利用者に属する月末資産状況についてのみ、月末資産残高を更新できる。

他の利用者に属する月末資産状況の月末資産残高を更新することはできない。

月末資産状況を取得する際は、必ず以下を検索条件に含める。

```text
id = snapshotId
AND
user_id = 操作対象利用者ID
```

指定された`snapshotId`が他の利用者に属する場合は、対象となる月末資産状況が存在しないものとして扱う。

更新対象となる資産口座についても、操作対象利用者に属していることを確認する。

資産口座を取得する際は、必ず以下を検索条件に含める。

```text
id = assetAccountId
AND
user_id = 操作対象利用者ID
```

他の利用者に属する資産口座を指定して月末資産残高を更新することはできない。

更新対象となる月末資産残高は、以下の組み合わせで特定する。

```text
month_end_asset_snapshot_id = snapshotId
AND
asset_account_id = assetAccountId
```

ただし、月末資産状況および資産口座について事前に利用者境界を確認し、操作対象利用者に属するリソースであることを保証したうえで取得する。

利用者IDは、リクエストボディ、クエリパラメータまたはパスパラメータでは受け付けない。

利用者IDは、ミドルウェアで設定された利用者コンテキストから取得する。

月末資産残高自体には`user_id`を保持せず、月末資産状況および資産口座との関連から利用者境界を保証する。

`X-User-Id`が指定されていない場合、形式が不正な場合、または指定された利用者が存在しない場合は、API共通方針に従ってエラーを返却する。

本APIでは、他の利用者に属する月末資産状況、資産口座および月末資産残高の存在をレスポンスから推測できないようにする。

---

## 6. パスパラメータ

| パラメータ名 | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `snapshotId` | string | ○ | 月末資産状況ID |
| `assetAccountId` | string | ○ | 月末資産残高を更新する資産口座ID |

リクエスト例：

```http
PATCH /api/v1/month-end-asset-snapshots/12/asset-balances/3
```

`snapshotId`は、
更新対象となる月末資産残高が属する
月末資産状況を一意に識別するIDである。

`assetAccountId`は、
更新対象となる月末資産残高が属する
資産口座を一意に識別するIDである。

更新対象の月末資産残高は、
以下の組み合わせによって特定する。

```text
month_end_asset_snapshot_id = snapshotId
AND
asset_account_id = assetAccountId
```

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
PATCH /api/v1/month-end-asset-snapshots/12/asset-balances/3
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
  "balance": 1300000
}
```

資産口座IDは、
パスパラメータの
`assetAccountId`から取得するため、
リクエストボディには含めない。

---

## 10. リクエスト項目

| 項目 | 型 | 必須 | NULL | 説明 |
|---|---|:---:|:---:|---|
| `balance` | integer | ○ | × | 更新後の月末資産残高 |

`balance`は、
日本円の整数値として受け付ける。

`0`も有効な値として扱う。

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
必須のパスパラメータとする。

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

正常例：

```text
1
3
123
```

不正例：

```text
0
-1
abc
1.5
```

`assetAccountId`の形式が不正な場合は、
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
  "balance": 1300000
}
```

0円への更新も許可する。

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
  "balance": "1300000"
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

月末資産残高を更新できるのは、
月末資産状況が
未確定の場合のみとする。

```text
confirmed = false
```

月末資産状況が
確定済みの場合は、
月末資産残高を更新できない。

```text
confirmed = true
    → 更新不可
```

確定済みの場合は、
SNP-005 月末資産状況確定解除APIによって
確定解除した後に更新する。

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
更新可否を判定しない。

対象年月時点で
月末資産管理対象ではない資産口座については、
月末資産残高を更新できない。

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
本APIで月末資産残高を更新できない。

```text
balance_recording_unit = 商品単位
    → 更新不可
```

商品単位の資産口座については、
VAL-003 商品別月末評価額更新APIを使用する。

残高記録単位の判定は、
単項目バリデーションではなく
業務ルールとして扱う。

---

### 11.9 月末資産残高の存在確認

以下の組み合わせで、
更新対象となる月末資産残高が
存在することを確認する。

```text
month_end_asset_snapshot_id = snapshotId
AND
asset_account_id = assetAccountId
```

月末資産残高が
存在しない場合は、
`MONTH_END_ASSET_BALANCE_NOT_FOUND`
として扱う。

本APIでは、
月末資産残高が存在しない場合に
新しいレコードを自動作成しない。

新規登録する場合は、
BAL-002 月末資産残高登録APIを使用する。

---

### 11.10 更新前と同一値の場合

更新後の`balance`が
現在登録されている`balance`と
同一であっても、
バリデーションエラーとはしない。

例えば、

```text
現在値
balance = 1200000

リクエスト
balance = 1200000
```

の場合も、
正常な更新要求として受け付ける。

結果として
データの実質的な内容が変化しなくても、
エラーにはしない。

---

### 11.11 X-User-Id

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

### 11.12 未定義項目

リクエストボディに、
定義されていない項目が
指定された場合の扱いは、
API共通方針に従う。

本APIで定義する
リクエスト項目は、
以下のみとする。

```text
balance
```

以下の項目は、
リクエストボディから受け付けない。

- `assetAccountId`
- `snapshotId`
- `userId`
- `targetYearMonth`
- `confirmed`

---

### 11.13 業務状態に依存する検証

以下は、
Form Requestによる
単項目バリデーションではなく、
業務ルールとして扱う。

- 月末資産状況が操作対象利用者に属していること
- 月末資産状況が未確定であること
- 資産口座が操作対象利用者に属していること
- 資産口座が対象年月時点で月末資産管理対象であること
- 資産口座の残高記録単位が口座単位であること
- 更新対象となる月末資産残高が存在すること

これらの判定は、
UseCaseおよびQueryで実施する。

Form Requestは、
入力形式および
単項目として完結する検証を担当し、
データベースの状態に依存する
業務ルールを持たせない。

---

## 12. 業務ルール

- 月末資産残高は、操作対象利用者に属する月末資産状況についてのみ更新できる。
- 他の利用者に属する月末資産状況の月末資産残高は更新できない。
- 更新対象の資産口座は、操作対象利用者に属している必要がある。
- 他の利用者に属する資産口座の月末資産残高は更新できない。
- 月末資産状況が未確定の場合のみ更新できる。
- 確定済みの月末資産状況に紐づく月末資産残高は更新できない。
- 確定済みの月末資産残高を修正する場合は、SNP-005 月末資産状況確定解除APIによって確定解除した後に更新する。
- 更新対象の資産口座は、対象年月時点で月末資産管理対象である必要がある。
- 更新対象の資産口座は、残高記録単位が資産口座単位である必要がある。
- 残高記録単位が商品単位の資産口座については、本APIで更新できない。
- 商品単位の資産口座については、VAL-003 商品別月末評価額更新APIを使用する。
- 更新対象となる月末資産残高が登録済みである必要がある。
- 月末資産残高が未登録の場合、本APIでは新規登録しない。
- 未登録の月末資産残高を登録する場合は、BAL-002 月末資産残高登録APIを使用する。
- `balance = 0`への更新を許可する。
- 月末資産残高は日本円の整数値として更新する。
- 更新前と同じ`balance`が指定された場合も、正常な更新要求として扱う。
- 月末資産残高の更新によって、月末資産状況を自動的に確定しない。
- 月末資産残高の更新によって、商品別月末評価額を更新しない。
- 月末資産残高の更新によって、目的達成判定を自動実行しない。
- 月末資産残高を更新しても、過去の目的達成判定履歴は再計算しない。

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
assetAccountId検証
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
月末資産残高取得
    ↓
月末資産残高存在確認
    ↓
月末資産残高更新
    ↓
APIレスポンス生成
    ↓
200 OK返却
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

更新対象となる
月末資産残高の取得条件は、
以下とする。

```text
month_end_asset_snapshot_id = snapshotId
AND
asset_account_id = assetAccountId
```

すべての更新条件を満たした場合のみ、
既存の月末資産残高を更新する。

---

## 14. 月末資産状況の扱い

月末資産残高を更新できるのは、
月末資産状況が
未確定の場合のみとする。

```text
confirmed = false
    → 更新可能

confirmed = true
    → 更新不可
```

確定済みの場合は、
SNP-005 月末資産状況確定解除APIによって
確定解除した後に
月末資産残高を更新する。

本APIによる更新成功後も、
月末資産状況は未確定のままとする。

```text
更新前
confirmed = false

    ↓ 月末資産残高更新

更新後
confirmed = false
```

本APIでは、
`month_end_asset_snapshots.confirmed`を
更新しない。

---

## 15. 資産口座の扱い

更新対象となる資産口座は、
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
資産口座のみ更新対象とする。

現在の利用状態だけを使用して、
過去の対象年月における
更新可否を判定しない。

---

## 16. 残高記録単位

資産口座の
`balance_recording_unit`によって、
使用する更新APIを分ける。

```text
口座単位
    ↓
BAL-003 月末資産残高更新

商品単位
    ↓
VAL-003 商品別月末評価額更新
```

商品単位の資産口座について、
BAL-003で月末資産残高を
更新してはならない。

これにより、
口座単位の月末資産残高と
商品別月末評価額の責務を分離する。

---

## 17. 更新対象の特定

更新対象となる
月末資産残高は、
以下の組み合わせで特定する。

```text
month_end_asset_snapshot_id = snapshotId
AND
asset_account_id = assetAccountId
```

例えば、

```text
snapshotId = 12
assetAccountId = 3
```

の場合は、

```text
month_end_asset_snapshot_id = 12
AND
asset_account_id = 3
```

を満たす
既存の月末資産残高を更新する。

該当する月末資産残高が
存在しない場合は、
新しいレコードを作成しない。

```text
月末資産残高あり
    → BAL-003で更新

月末資産残高なし
    → BAL-003では更新不可
    → BAL-002で登録
```

---

## 18. 0円の扱い

`balance = 0`は、
有効な月末資産残高として扱う。

例えば、
現在の残高が

```text
balance = 50000
```

の場合に、

```json
{
  "balance": 0
}
```

を指定すると、
正常に0円へ更新する。

```text
更新前
balance = 50000

    ↓

更新後
balance = 0
```

`0`を
未入力または未登録として
扱ってはならない。

---

## 19. 同一値への更新

更新前と同じ`balance`が
指定された場合も、
正常な更新要求として扱う。

例えば、

```text
更新前
balance = 1200000
```

に対して、

```json
{
  "balance": 1200000
}
```

が指定された場合も、
エラーとはしない。

本APIでは、
更新前後の値が同一であることを理由に
`409 Conflict`などを返却しない。

更新結果として、
同じ値が保持される。

---

## 20. 成功レスポンス

### 20.1 HTTPステータス

```http
200 OK
```

月末資産残高の
更新に成功した場合は、
`200 OK`を返却する。

更新前と同一の値が
指定された場合も、
`200 OK`を返却する。

---

### 20.2 レスポンスボディ

```json
{
  "data": {
    "assetAccountId": "3",
    "balance": 1300000
  }
}
```

0円へ更新した場合：

```json
{
  "data": {
    "assetAccountId": "3",
    "balance": 0
  }
}
```

更新後の月末資産残高を
`data`オブジェクトとして返却する。

---

## 21. レスポンス項目

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `data` | object | × | 更新後の月末資産残高 |
| `data.assetAccountId` | string | × | 資産口座ID |
| `data.balance` | integer | × | 更新後の月末資産残高 |

`assetAccountId`は、
API共通方針に従って
文字列として返却する。

`balance`は、
日本円の整数値として返却する。

```json
{
  "balance": 1300000
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
- `previous_balance`
- `created_at`
- `updated_at`
- `month_end_holding_values`

月末資産状況そのものの情報は、
SNP-003 月末資産状況詳細取得APIで取得する。

月末資産残高一覧が必要な場合は、
BAL-001 月末資産残高一覧取得APIを使用する。

---

## 22. エラーレスポンス

異常時は、
API共通方針で定めた
共通エラーレスポンス形式を使用する。

本APIでは、
利用者コンテキストの不正、
`snapshotId`または
`assetAccountId`の不正、
月末資産状況または資産口座の不存在、
確定済み月末資産状況への更新、
対象年月時点で利用できない資産口座、
残高記録単位の不一致、
月末資産残高の不存在および
想定外のサーバーエラーを扱う。

エラーが発生した場合は、
月末資産残高を更新しない。

---

### 22.1 利用者コンテキストが指定されていない場合

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

### 22.2 利用者ID形式が不正な場合

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

### 22.3 利用者が存在しない場合

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

### 22.4 バリデーションエラー

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
- `assetAccountId`の形式が不正
- `balance`が未指定
- `balance`が`null`
- `balance`がintegerではない
- `balance`が負数
- `balance`が保持可能な範囲を超えている

---

### 22.5 月末資産状況が存在しない場合

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

### 22.6 月末資産状況が確定済みの場合

指定された月末資産状況が
確定済みの場合は、
`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`
を返却する。

```json
{
  "error": {
    "code": "MONTH_END_ASSET_SNAPSHOT_CONFIRMED",
    "message": "確定済みの月末資産状況に紐づく月末資産残高は更新できません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

確定済みデータを修正する場合は、
先にSNP-005 月末資産状況確定解除APIを実行する。

---

### 22.7 資産口座が存在しない場合

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

### 22.8 対象年月時点で利用できない資産口座の場合

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

### 22.9 残高記録単位が口座単位ではない場合

指定された資産口座の
残高記録単位が
商品単位の場合は、
`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`
を返却する。

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH",
    "message": "指定された資産口座は口座単位の残高更新対象ではありません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

商品単位の資産口座については、
VAL-003 商品別月末評価額更新APIを使用する。

---

### 22.10 月末資産残高が存在しない場合

以下の組み合わせで
月末資産残高が
存在しない場合は、
`MONTH_END_ASSET_BALANCE_NOT_FOUND`
を返却する。

```text
month_end_asset_snapshot_id = snapshotId
AND
asset_account_id = assetAccountId
```

```json
{
  "error": {
    "code": "MONTH_END_ASSET_BALANCE_NOT_FOUND",
    "message": "指定された月末資産残高が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

本APIでは、
月末資産残高が存在しない場合に
新しいレコードを作成しない。

新規登録する場合は、
BAL-002 月末資産残高登録APIを使用する。

---

### 22.11 想定外のエラーが発生した場合

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

## 23. HTTPステータス

| HTTPステータス | 条件 |
|---:|---|
| `200 OK` | 月末資産残高の更新に成功した |
| `400 Bad Request` | 利用者コンテキストが未指定、または利用者IDの形式が不正である |
| `404 Not Found` | 指定された利用者、月末資産状況、資産口座または月末資産残高が存在しない |
| `409 Conflict` | 現在の業務状態では月末資産残高を更新できない |
| `422 Unprocessable Entity` | パスパラメータまたはリクエスト項目のバリデーションエラー |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

---

### 23.1 200の扱い

月末資産残高の
更新に成功した場合は、
`200 OK`
を返却する。

更新後の`balance`が
更新前と同一の場合も、
正常な更新要求として
`200 OK`
を返却する。

`balance = 0`への更新も、
正常な更新として扱う。

---

### 23.2 400の扱い

以下の場合は、
`400 Bad Request`
を返却する。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正である

---

### 23.3 404の扱い

以下の場合は、
`404 Not Found`
を返却する。

- 指定された利用者が存在しない
- 指定された利用者が論理削除されている
- 指定された月末資産状況が存在しない
- 指定された月末資産状況が他の利用者に属している
- 指定された資産口座が存在しない
- 指定された資産口座が他の利用者に属している
- 指定された月末資産残高が存在しない

他利用者に属するデータを指定した場合も、
対象リソースの存在を公開しない。

---

### 23.4 409の扱い

リクエスト形式および
対象リソースは正しいが、
現在の業務状態では
月末資産残高を更新できない場合は、
`409 Conflict`
を返却する。

以下の場合を含む。

- 月末資産状況が確定済み
- 資産口座が対象年月時点で月末資産管理対象ではない
- 資産口座の残高記録単位が口座単位ではない

---

### 23.5 422の扱い

以下の場合は、
`422 Unprocessable Entity`
を返却する。

- `snapshotId`の形式が不正
- `assetAccountId`の形式が不正
- `balance`が未指定
- `balance`が`null`
- `balance`の型が不正
- `balance`が負数
- `balance`が許容範囲外

---

## 24. エラーコード

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
| `MONTH_END_ASSET_BALANCE_NOT_FOUND` | 404 | 更新対象の月末資産残高が存在しない | × |
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

## 25. 冪等性

本APIは、
同一の月末資産残高に対して
同一の`balance`を指定して
複数回実行した場合、
最終的なリソース状態は同一となる。

そのため、
本APIは冪等として扱う。

例えば、

```text
更新前
balance = 1200000
```

に対して、

```http
PATCH /api/v1/month-end-asset-snapshots/12/asset-balances/3
```

```json
{
  "balance": 1300000
}
```

を複数回送信した場合、

```text
1回目
balance = 1300000

2回目
balance = 1300000

3回目
balance = 1300000
```

となり、
最終的な月末資産残高は
変わらない。

更新前と同じ値を指定した場合も、
エラーとはしない。

```text
現在値
balance = 1300000

    ↓

PATCH
balance = 1300000

    ↓

200 OK
balance = 1300000
```

本APIでは、
`Idempotency-Key`は使用しない。

ただし、
冪等性は
同じ対象リソースに対して
同じ更新内容を送信した場合の
最終状態について保証するものであり、
並行更新による競合を
解決するものではない。

Phase1では、
楽観ロック用の
バージョン番号や
`If-Match`による
更新競合制御は採用しない。

複数の更新要求が
同時に実行された場合は、
最終的にデータベースへ
反映された更新結果を
現在値とする。

---

## 26. 関連テーブル

### 26.1 month_end_asset_snapshots

対象年月ごとの
月末資産状況および確定状態を保持する。

本APIでは、
更新対象となる月末資産残高が属する
月末資産状況を特定し、
利用者境界および
確定状態を確認するために参照する。

使用する主なカラムは、
以下とする。

| カラム | 用途 |
|---|---|
| `id` | 月末資産状況ID |
| `user_id` | 利用者境界 |
| `target_year_month` | 対象年月 |
| `confirmed` | 更新可否判定 |

取得条件は、
以下とする。

```text
id = snapshotId
AND
user_id = 操作対象利用者ID
```

月末資産状況が
確定済みの場合は、
月末資産残高を更新できない。

本APIでは、
`month_end_asset_snapshots`を更新しない。

---

### 26.2 asset_accounts

利用者が所有する
資産口座を保持する。

本APIでは、
更新対象となる資産口座が
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
本APIで月末資産残高を更新しない。

本APIでは、
`asset_accounts`を更新しない。

---

### 26.3 asset_account_available_settings

資産口座について、
対象年月時点で
月末資産管理対象となるかを
判定するための設定を保持する。

本APIでは、
月末資産状況の
`target_year_month`時点で
更新対象の資産口座が
月末資産管理対象であることを
確認するために参照する。

現在の資産口座の状態だけではなく、
対象年月時点の状態を判定するために使用する。

本APIでは、
`asset_account_available_settings`を更新しない。

---

### 26.4 month_end_asset_balances

資産口座単位の
月末資産残高を保持する。

本APIで
更新対象となるテーブルである。

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

更新対象は、
以下の条件で特定する。

```text
month_end_asset_snapshot_id = snapshotId
AND
asset_account_id = assetAccountId
```

更新するカラムは、
以下とする。

```text
balance = リクエストのbalance
```

`month_end_asset_snapshot_id`および
`asset_account_id`は、
本APIでは変更しない。

同一の月末資産状況・資産口座について
複数の月末資産残高が存在しないよう、
以下の組み合わせに
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
asset_account_id
```

月末資産残高が存在しない場合は、
本APIで新規レコードを作成しない。

---

### 26.5 users

操作対象となる
利用者を保持する。

`X-User-Id`で指定された利用者が
存在することを確認するために参照する。

本APIでは、
`users`を更新しない。

---

### 26.6 holding_assets

本APIでは、
`holding_assets`を参照・更新しない。

保有商品は、
残高記録単位が商品単位の
資産口座について使用する。

商品別月末評価額の更新は、
VAL-003 商品別月末評価額更新APIで行う。

---

### 26.7 month_end_holding_values

本APIでは、
`month_end_holding_values`を
参照・更新しない。

商品別月末評価額の更新は、
VAL-003 商品別月末評価額更新APIで行う。

---

### 26.8 assessment_histories

本APIでは、
`assessment_histories`を
参照・登録・更新・削除しない。

月末資産残高を更新しても、
過去の目的達成判定履歴は
再計算しない。

また、
月末資産残高の更新を契機として
目的達成判定を自動実行しない。

---

## 27. 関連する機能要件

- 月末資産管理
  - 資産口座単位で残高を記録する資産口座について、登録済みの月末資産残高を更新できる
  - 月末資産残高は日本円の整数値として管理する
  - `0円`を有効な月末資産残高として更新できる
  - 月末資産残高が未登録の場合は更新APIで新規作成しない
  - 同一月末資産状況・資産口座の月末資産残高は1件のみ保持する

- 資産口座管理
  - 資産口座ごとに残高記録単位を保持する
  - 残高記録単位が口座単位の場合のみ、本APIで月末資産残高を更新する
  - 対象年月時点で月末資産管理対象となる資産口座のみ更新対象とする

- 月末資産状況の確定
  - 未確定の月末資産状況に紐づく月末資産残高のみ更新できる
  - 確定済み月末資産状況の残高を変更する場合は、先に確定解除する
  - 月末資産残高の更新だけでは月末資産状況を自動確定しない

- 目的達成判定
  - 月末資産残高の更新を契機として目的達成判定を自動実行しない
  - 保存済みの目的達成判定履歴は再計算しない

- 利用者境界
  - 操作対象利用者に属する月末資産状況の残高のみ更新できる
  - 操作対象利用者に属する資産口座の残高のみ更新できる
  - 他の利用者に属するリソースの存在を外部へ公開しない

具体的な章番号は、
`functional-requirements.md`の
最新定義に従う。

---

## 28. テスト観点

### 28.1 正常系

- 有効な入力値で既存の月末資産残高を更新できること
- `200 OK`で返却されること
- `month_end_asset_balances`に新しいレコードが作成されないこと
- 対象レコードの`balance`のみが更新されること
- `month_end_asset_snapshot_id`が変更されないこと
- `asset_account_id`が変更されないこと
- 更新対象以外の月末資産残高が変更されないこと
- 月末資産状況の`confirmed`が変更されないこと
- 商品別月末評価額が変更されないこと
- 目的達成判定が自動実行されないこと
- 過去の目的達成判定履歴が再計算されないこと

---

### 28.2 snapshotId

- 正しい`snapshotId`を指定して更新できること
- `snapshotId = 1`を指定できること
- `snapshotId = 0`でバリデーションエラーとなること
- 負数でバリデーションエラーとなること
- 小数でバリデーションエラーとなること
- 文字列`abc`でバリデーションエラーとなること
- ID形式不正時に`422 Unprocessable Entity`となること
- ID形式不正時に`VALIDATION_ERROR`となること

---

### 28.3 assetAccountId

- 正しい`assetAccountId`を指定して更新できること
- `assetAccountId = 1`を指定できること
- `assetAccountId = 0`でバリデーションエラーとなること
- 負数でバリデーションエラーとなること
- 小数でバリデーションエラーとなること
- 文字列`abc`でバリデーションエラーとなること
- ID形式不正時に`422 Unprocessable Entity`となること
- ID形式不正時に`VALIDATION_ERROR`となること

---

### 28.4 balance

以下を確認する。

- 正の整数へ更新できること
- `balance = 0`へ更新できること
- `balance = 1`へ更新できること
- 保持可能な最大値へ更新できること
- `balance`未指定でバリデーションエラーとなること
- `balance = null`でバリデーションエラーとなること
- `balance = -1`でバリデーションエラーとなること
- 小数値でバリデーションエラーとなること
- 数値文字列でバリデーションエラーとなること
- 保持可能な範囲を超える値でバリデーションエラーとなること

---

### 28.5 月末資産状況の存在確認

- 存在する月末資産状況について更新できること
- 存在しない`snapshotId`で`404 Not Found`となること
- 存在しない場合に`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`となること
- 存在しない場合に月末資産残高が更新されないこと

---

### 28.6 月末資産状況の利用者境界

- 操作対象利用者に属する月末資産状況について更新できること
- 他利用者に属する月末資産状況について更新できないこと
- 他利用者の`snapshotId`で`404 Not Found`となること
- 他利用者の場合に`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`となること
- 他利用者の月末資産状況が存在することをレスポンスから判別できないこと
- 月末資産状況の検索条件に`user_id`が含まれていること

---

### 28.7 資産口座の存在確認

- 存在する資産口座について更新できること
- 存在しない`assetAccountId`で`404 Not Found`となること
- 存在しない場合に`ASSET_ACCOUNT_NOT_FOUND`となること
- 存在しない資産口座を指定した場合に月末資産残高が更新されないこと

---

### 28.8 資産口座の利用者境界

- 操作対象利用者に属する資産口座について更新できること
- 他利用者に属する資産口座について更新できないこと
- 他利用者の資産口座IDで`404 Not Found`となること
- 他利用者の場合に`ASSET_ACCOUNT_NOT_FOUND`となること
- 他利用者の資産口座が存在することをレスポンスから判別できないこと
- 資産口座の検索条件に`user_id`が含まれていること

---

### 28.9 月末資産状況の確定状態

未確定の場合：

```text
confirmed = false
```

について、

- 月末資産残高を更新できること

確定済みの場合：

```text
confirmed = true
```

について、

- 月末資産残高を更新できないこと
- `409 Conflict`となること
- `MONTH_END_ASSET_SNAPSHOT_CONFIRMED`となること
- 既存の月末資産残高が変更されないこと

---

### 28.10 対象年月時点の利用可能状態

資産口座の利用可能期間が
対象年月によって異なるデータを用意する。

以下を確認する。

- 対象年月時点で月末資産管理対象の資産口座について更新できること
- 対象年月時点で月末資産管理対象ではない資産口座について更新できないこと
- 対象外の場合に`409 Conflict`となること
- 対象外の場合に`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`となること
- 現在の利用状態だけを使用して過去月の更新可否を判定しないこと
- 過去月では対象だが現在は対象外の資産口座について、該当過去月の残高を更新できること
- 現在は対象だが対象年月時点では対象外の資産口座について更新できないこと

---

### 28.11 残高記録単位

残高記録単位が
口座単位の場合：

```text
balance_recording_unit = 口座単位
```

について、

- BAL-003で更新できること

商品単位の場合：

```text
balance_recording_unit = 商品単位
```

について、

- BAL-003で更新できないこと
- `409 Conflict`となること
- `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`となること
- 既存の月末資産残高が変更されないこと
- 商品別月末評価額も変更されないこと

---

### 28.12 月末資産残高の存在確認

以下の組み合わせについて、
月末資産残高が存在する状態を用意する。

```text
month_end_asset_snapshot_id = 12
asset_account_id = 3
```

以下を確認する。

- 対象レコードが正常に更新されること
- 他の月末資産残高が更新されないこと

同じ組み合わせの
月末資産残高が存在しない場合について、
以下を確認する。

- `404 Not Found`となること
- `MONTH_END_ASSET_BALANCE_NOT_FOUND`となること
- 新しい月末資産残高が作成されないこと
- BAL-003がupsertとして動作しないこと

---

### 28.13 0円

`balance = 0`を送信した場合について、
以下を確認する。

- 正常に更新できること
- `200 OK`となること
- データベースへ`0`として保存されること
- レスポンスで`balance = 0`となること
- `0`が未入力として扱われないこと
- `null`へ変換されないこと

---

### 28.14 同一値への更新

例えば、

```text
現在値
balance = 1200000
```

の状態で、

```json
{
  "balance": 1200000
}
```

を送信する。

以下を確認する。

- エラーにならないこと
- `200 OK`となること
- `balance = 1200000`のままとなること
- `409 Conflict`とならないこと
- 新しいレコードが作成されないこと

---

### 28.15 冪等性

同一リソースに対して、
同じ更新リクエストを
複数回実行する。

```json
{
  "balance": 1300000
}
```

以下を確認する。

- 1回目が`200 OK`となること
- 2回目以降も`200 OK`となること
- 最終的な`balance`が`1300000`となること
- リクエスト回数によって月末資産残高レコード数が増加しないこと
- 同じリクエストの繰り返しによって他のデータが変更されないこと

---

### 28.16 レスポンス契約

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
- 更新前の`balance`がレスポンスへ含まれないこと
- `createdAt`がレスポンスへ含まれないこと
- `updatedAt`がレスポンスへ含まれないこと
- DB内部のsnake_caseのカラム名がそのまま公開されないこと

---

### 28.17 トランザクション

- 業務条件の最終確認から月末資産残高更新までが同一トランザクション内で実行されること
- 更新途中で例外が発生した場合にロールバックされること
- 業務条件を満たさない場合に月末資産残高が更新されないこと
- 更新失敗時に中途半端な状態が残らないこと
- 月末資産残高が存在しない場合に新規レコードが作成されないこと

---

### 28.18 副作用

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
- 月末資産状況を自動確定解除しない
- 商品別月末評価額を更新しない
- 目的達成判定を自動実行しない
- 過去の目的達成判定履歴を再計算しない

---

### 28.19 エラー時

以下のエラー時に、
既存の月末資産残高が
変更されないことを確認する。

- `USER_CONTEXT_REQUIRED`
- `INVALID_USER_ID`
- `USER_NOT_FOUND`
- `VALIDATION_ERROR`
- `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
- `MONTH_END_ASSET_SNAPSHOT_CONFIRMED`
- `ASSET_ACCOUNT_NOT_FOUND`
- `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`
- `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`
- `MONTH_END_ASSET_BALANCE_NOT_FOUND`

---

### 28.20 異常系

- 想定外の例外で`500 Internal Server Error`となること
- 想定外の例外発生時にトランザクションがロールバックされること
- 想定外の例外発生時に更新前の`balance`が保持されること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログに同じリクエストIDが記録されること
- SQLがレスポンスへ含まれないこと
- PostgreSQLの制約名がレスポンスへ含まれないこと
- スタックトレースがレスポンスへ含まれないこと
- 内部例外メッセージがレスポンスへ含まれないこと

---

## 29. Laravel実装方針

### 29.1 Action

HTTPリクエストを受け付け、
月末資産状況ID、
資産口座ID、
更新後の月末資産残高および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの入力値を受け取り、
月末資産残高更新UseCaseを呼び出す。

UseCaseから受け取った更新結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- `snapshotId`の形式検証
- `assetAccountId`の形式検証
- リクエストボディの単項目バリデーション
- 利用者境界の判定
- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 資産口座の存在確認
- 対象年月時点の利用可否判定
- 残高記録単位の判定
- 月末資産残高の存在確認
- 月末資産残高の更新
- トランザクション制御
- レスポンス生成処理

---

### 29.2 UseCase

月末資産残高更新の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 月末資産状況IDを受け取る
- 資産口座IDを受け取る
- 更新後の月末資産残高を受け取る
- 月末資産状況を取得する
- 月末資産状況が未確定であることを確認する
- 資産口座を取得する
- 対象年月時点で資産口座が月末資産管理対象であることを確認する
- 資産口座の残高記録単位が口座単位であることを確認する
- 更新対象となる月末資産残高を取得する
- 月末資産残高が存在することを確認する
- 月末資産残高を更新する
- 更新結果を返却する

指定された月末資産状況が存在しない場合、
または操作対象利用者に属していない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

指定された資産口座が存在しない場合、
または操作対象利用者に属していない場合は、
`ASSET_ACCOUNT_NOT_FOUND`
として扱う。

更新対象となる月末資産残高が
存在しない場合は、
`MONTH_END_ASSET_BALANCE_NOT_FOUND`
として扱う。

---

### 29.3 Form Request / DTO

リクエストボディの
形式および単項目バリデーションを担当する。

検証対象は、
`balance`とする。

Form Requestでは、
以下を検証する。

- 必須であること
- `null`ではないこと
- integerであること
- 0以上であること
- データベースで保持可能な範囲であること

`0`は、
有効な更新値として扱う。

業務状態に依存する以下の検証は、
Form Requestでは行わない。

- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 資産口座の存在確認
- 対象年月時点の利用可否確認
- 残高記録単位確認
- 月末資産残高の存在確認

検証済みの入力値は、
入力用DTOへ変換して
UseCaseへ渡す。

例：

```php
final readonly class UpdateMonthEndAssetBalanceInput
{
    public function __construct(
        public int $balance,
    ) {
    }
}
```

---

### 29.4 パスパラメータ検証

`snapshotId`および`assetAccountId`の形式は、API共通方針に従って検証する。

それぞれ、以下を確認する。

- 指定されていること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

形式が不正な場合は、`VALIDATION_ERROR`として扱う。

存在確認および利用者境界確認は、UseCaseおよびQueryで行う。

---

### 29.5 Query

月末資産残高更新に必要なデータ取得を担当する。

主な取得対象は、以下とする。

- `month_end_asset_snapshots`
- `asset_accounts`
- `asset_account_available_settings`
- `month_end_asset_balances`

本APIでは、原則として以下を参照しない。

- `holding_assets`
- `month_end_holding_values`
- `assessment_histories`

---

### 29.6 月末資産状況取得

更新対象となる月末資産残高が属する月末資産状況は、必ず利用者境界を含めて取得する。

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

以下のように、月末資産状況IDだけで取得してはならない。

```php
MonthEndAssetSnapshot::find($snapshotId);
```

取得できなかった場合は、以下を区別せず`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`として扱う。

- 月末資産状況が存在しない
- 他の利用者に属している

---

### 29.7 確定状態確認

取得した月末資産状況が未確定であることを確認する。

```php
if ($snapshot->confirmed) {
    throw new
        MonthEndAssetSnapshotConfirmedException();
}
```

確定済みの場合は、`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`として扱う。

本API内で自動的に確定解除してはならない。

---

### 29.8 資産口座取得

更新対象となる資産口座は、必ず利用者境界を含めて取得する。

```php
$assetAccount = AssetAccount::query()
    ->where('id', $assetAccountId)
    ->where('user_id', $userId)
    ->first();
```

以下のように、資産口座IDだけで取得してはならない。

```php
AssetAccount::find($assetAccountId);
```

取得できなかった場合は、以下を区別せず`ASSET_ACCOUNT_NOT_FOUND`として扱う。

- 資産口座が存在しない
- 他の利用者に属している

---

### 29.9 対象年月時点の利用可否判定

月末資産状況の`target_year_month`を使用して、資産口座が対象年月時点で月末資産管理対象であることを確認する。

判定には、`asset_account_available_settings`を使用する。

概念的には、以下を判定する。

```text
assetAccountId
+
snapshot.target_year_month
    ↓
対象年月時点で利用可能か
```

対象年月時点で月末資産管理対象ではない場合は、`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`として扱う。

現在の資産口座の状態だけで過去月の更新可否を判断してはならない。

---

### 29.10 残高記録単位確認

資産口座の`balance_recording_unit`が口座単位であることを確認する。

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::ACCOUNT
) {
    throw new
        AssetAccountBalanceRecordingUnitMismatchException();
}
```

条件を満たさない場合は、`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`として扱う。

商品単位の資産口座については、VAL-003 商品別月末評価額更新APIを使用する。

---

### 29.11 月末資産残高取得

更新対象となる月末資産残高は、以下の組み合わせで取得する。

```php
$balance = MonthEndAssetBalance::query()
    ->where(
        'month_end_asset_snapshot_id',
        $snapshot->id,
    )
    ->where(
        'asset_account_id',
        $assetAccount->id,
    )
    ->first();
```

取得できなかった場合は、`MONTH_END_ASSET_BALANCE_NOT_FOUND`として扱う。

本APIでは、存在しない場合に新しい月末資産残高を自動作成しない。

---

### 29.12 Repository

月末資産残高の更新を担当する。

更新対象は、`balance`のみとする。

```php
$balance->balance = $input->balance;
$balance->save();

return $balance;
```

以下の項目は、本APIでは変更しない。

- `month_end_asset_snapshot_id`
- `asset_account_id`
- `created_at`

`updated_at`は、Laravelによって更新する。

Repositoryでは、以下の処理は行わない。

- 月末資産残高の新規登録
- 月末資産状況の確定
- 月末資産状況の確定解除
- 商品別月末評価額の更新
- 目的達成判定の実行

---

### 29.13 トランザクション

業務条件の最終確認から月末資産残高更新までを、1つのデータベーストランザクション内で実行する。

実装例：

```php
$balance = DB::transaction(
    function () use (
        $userId,
        $snapshotId,
        $assetAccountId,
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
                    $assetAccountId,
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

        $balance =
            $this->balanceQuery
                ->findBySnapshotAndAssetAccount(
                    $snapshot->id,
                    $assetAccount->id,
                );

        if ($balance === null) {
            throw new
                MonthEndAssetBalanceNotFoundException();
        }

        return $this->repository->updateBalance(
            $balance,
            $input->balance,
        );
    },
);
```

処理途中で例外が発生した場合は、更新前の状態へロールバックする。

---

### 29.14 同一値更新の扱い

更新前と更新後の`balance`が同一であっても、エラーとしない。

例えば、

```text
現在値
balance = 1200000

更新値
balance = 1200000
```

の場合も、正常処理として扱う。

UseCaseで差分が存在することを必須条件にはしない。

Phase1では、同一値の場合に更新SQLを省略する最適化は必須としない。

---

### 29.15 0円の扱い

`balance = 0`は、有効な更新値として扱う。

以下のようなtruthy / falsyによる判定は行わない。

```php
if (! $input->balance) {
    // 0円まで未入力扱いになるため使用しない
}
```

更新値は、明示的にintegerとして扱う。

---

### 29.16 Repositoryでupsertしない理由

本APIは、既存の月末資産残高を更新する責務のみを持つ。

そのため、以下のような処理は使用しない。

```php
MonthEndAssetBalance::updateOrCreate(
    [
        'month_end_asset_snapshot_id'
            => $snapshot->id,
        'asset_account_id'
            => $assetAccount->id,
    ],
    [
        'balance'
            => $input->balance,
    ],
);
```

この実装では、更新対象が存在しない場合に新規登録されてしまう。

登録と更新の責務を分離するため、存在しない場合は`MONTH_END_ASSET_BALANCE_NOT_FOUND`を返却する。

新規登録は、BAL-002で行う。

---

### 29.17 Responder

UseCaseから受け取った更新後の月末資産残高を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`200 OK`とともに`data`オブジェクトとして返却する。

Responderは、以下の処理を行わない。

- データベース検索
- 利用者境界の判定
- 確定状態の判定
- 対象年月時点の利用可否判定
- 残高記録単位の判定
- 月末資産残高の存在確認
- 月末資産残高の更新

---

### 29.18 API Resource

データベースカラムを直接返却せず、API Resourceを利用してAPIレスポンス形式へ変換する。

変換例：

```php
return [
    'assetAccountId'
        => (string) $this->asset_account_id,
    'balance'
        => (int) $this->balance,
];
```

`balance = 0`の場合も、そのまま整数の`0`を返却する。

以下の項目は、レスポンスへ含めない。

- 月末資産残高ID
- `snapshotId`
- `userId`
- `targetYearMonth`
- `confirmed`
- 資産口座名
- 更新前の`balance`
- `createdAt`
- `updatedAt`

---

### 29.19 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 29.20 Eloquentモデル

`MonthEndAssetBalance`モデルは、`month_end_asset_balances`テーブルへ対応する。

主に以下の属性を使用する。

```text
id
month_end_asset_snapshot_id
asset_account_id
balance
```

`balance`は、日本円の整数値として扱う。

必要に応じて、integer castを設定する。

```php
protected function casts(): array
{
    return [
        'balance' => 'integer',
    ];
}
```

---

### 29.21 排他制御

Phase1では、月末資産残高更新専用の楽観ロック用バージョン番号は持たない。

また、通常の更新処理では明示的な行ロックを必須としない。

複数の更新要求が同時に実行された場合は、最終的にデータベースへ反映された更新を現在値とする。

将来的に、複数利用者による同時編集や更新競合の検知が必要になった場合は、以下の導入を検討する。

- 楽観ロック
- `version`カラム
- `updated_at`を利用した競合検知
- `If-Match`

---

### 29.22 業務ルール判定クラス

対象年月時点の利用可否判定や残高記録単位の判定は、BAL-002と共通化してよい。

例えば、以下の責務へ分離する。

```text
AssetAccountAvailabilityValidator
    → 対象年月時点の利用可否判定

AssetAccountBalanceRecordingUnitValidator
    → 口座単位であることの判定
```

登録APIと更新APIで同じ業務ルールを別々に実装しない。

各Validatorは、HTTPレスポンス生成やデータベース更新を行わない。

---

### 29.23 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
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
| 月末資産残高不存在 | `MONTH_END_ASSET_BALANCE_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

SQL、スタックトレース、PostgreSQLの内部情報および内部例外メッセージは、APIレスポンスへ含めない。

ログには、調査に必要な範囲で以下を記録する。

- 操作対象利用者ID
- 月末資産状況ID
- 資産口座ID
- 対象年月
- 独自エラーコード
- リクエストID

更新前後の月末資産残高の具体的な金額は、不要にエラーログへ出力しない。

---

## 30. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type UpdateMonthEndAssetBalanceParams = {
  snapshotId: string;
  assetAccountId: string;
};
```

リクエスト型は、
以下とする。

```ts
export type UpdateMonthEndAssetBalanceRequest = {
  balance: number;
};
```

レスポンス型は、
以下とする。

```ts
export type UpdatedMonthEndAssetBalance = {
  assetAccountId: string;
  balance: number;
};

export type UpdateMonthEndAssetBalanceResponse = {
  data: UpdatedMonthEndAssetBalance;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.patch<UpdateMonthEndAssetBalanceResponse>(
    `/api/v1/month-end-asset-snapshots/${snapshotId}/asset-balances/${assetAccountId}`,
    {
      balance: 1300000,
    },
  );
```

更新成功後は、
返却された月末資産残高をもとに
画面上の表示を更新する。

---

### 30.1 snapshotIdの扱い

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

### 30.2 assetAccountIdの扱い

`assetAccountId`は、
API共通方針に従って
文字列として扱う。

```ts
const assetAccountId: string = '3';
```

更新対象の資産口座は、
パスパラメータで指定する。

リクエストボディへ
`assetAccountId`を
重複して含めない。

---

### 30.3 balanceの扱い

`balance`は、
日本円の整数値として扱う。

```ts
const request: UpdateMonthEndAssetBalanceRequest = {
  balance: 1300000,
};
```

`0`も有効な更新値とする。

```ts
const request: UpdateMonthEndAssetBalanceRequest = {
  balance: 0,
};
```

以下のような
truthy / falsyによる
入力有無の判定は行わない。

```ts
if (!balance) {
  // balance = 0 も未入力扱いになるため使用しない
}
```

入力有無は、
`null`、
`undefined`、
空文字列などを
明示的に判定する。

---

### 30.4 入力フォーム

月末資産残高更新フォームでは、
登録済みの`balance`を
初期値として表示する。

例えば、
BAL-001 月末資産残高一覧取得APIで
取得した値を使用する。

```ts
const [balance, setBalance] =
  useState<number | null>(
    item.balance,
  );
```

本APIは
登録済み残高の更新専用であるため、
`balance = null`の項目については
BAL-003を呼び出さない。

未登録の場合は、
BAL-002 月末資産残高登録APIを使用する。

---

### 30.5 登録・更新APIの切り替え

BAL-001のレスポンスを利用して、
登録済みか未登録かを判定できる。

```ts
if (item.balance === null) {
  // BAL-002 月末資産残高登録
} else {
  // BAL-003 月末資産残高更新
}
```

`balance = 0`は
登録済みであるため、
BAL-003を使用する。

以下の判定は行わない。

```ts
if (!item.balance) {
  // 0円を未登録と誤判定するため使用しない
}
```

---

### 30.6 月末資産状況の確定状態

BAL-003のレスポンスには、
月末資産状況の`confirmed`は
含まれない。

編集可否を画面上で
補助的に制御する場合は、
SNP-003 月末資産状況詳細取得APIで
確定状態を取得する。

```text
SNP-003
    → confirmed取得

BAL-001
    → 月末資産残高取得

BAL-003
    → 月末資産残高更新
```

`confirmed = true`の場合は、
更新フォームを非活性化するなどの
画面制御を行ってよい。

ただし、
更新可否の最終判断は
BAL-003で行う。

---

### 30.7 確定済みエラーの扱い

月末資産状況が
確定済みの場合は、

`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`

が返却される。

フロントエンドでは、
更新できないことを表示する。

表示例：

```text
確定済みの月末資産状況は
編集できません。

編集する場合は、
先に確定を解除してください。
```

必要に応じて、
SNP-005 月末資産状況確定解除APIを
実行する画面への導線を表示する。

---

### 30.8 月末資産残高不存在エラーの扱い

更新対象となる
月末資産残高が存在しない場合は、

`MONTH_END_ASSET_BALANCE_NOT_FOUND`

が返却される。

フロントエンドでは、
最新のBAL-001を再取得して
登録状態を確認する。

未登録であることが確認できた場合は、
BAL-002 月末資産残高登録APIを
使用する。

BAL-003の失敗を受けて、
同じ画面から自動的に
BAL-002へ切り替えて再送信することは
Phase1では行わない。

---

### 30.9 対象年月時点で利用できない場合

指定された資産口座が
対象年月時点で
月末資産管理対象ではない場合は、

`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`

が返却される。

表示例：

```text
この資産口座は、
対象年月の月末資産管理対象ではありません。
```

フロントエンド側で
現在の資産口座状態のみを使用して
過去月の更新可否を
独自に判定しない。

---

### 30.10 残高記録単位不一致の扱い

指定された資産口座が
商品単位で残高を記録する場合は、

`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`

が返却される。

表示例：

```text
この資産口座は、
商品別評価額で管理します。
```

商品単位の資産口座については、
VAL-003 商品別月末評価額更新APIを使用する。

---

### 30.11 更新処理中の画面制御

更新処理中は、
保存ボタンを非活性化する。

```ts
const [isUpdating, setIsUpdating] =
  useState(false);
```

更新開始時に
`isUpdating = true`とし、
成功または失敗後に
`false`へ戻す。

これにより、
意図しない連続送信を防止する。

BAL-003自体は冪等であるが、
不要な通信を抑止するため、
フロントエンドでも
二重送信を防止する。

---

### 30.12 更新成功時の扱い

更新成功後は、
画面上の残高を
レスポンスの値で更新する。

```ts
const updatedBalance =
  response.data.balance;
```

React Query等を使用する場合は、
BAL-001の一覧クエリを
invalidateしてよい。

これにより、
一覧画面へ戻った場合も
最新の残高を表示できる。

---

### 30.13 同一値更新の扱い

更新前と同じ`balance`を
送信しても、
正常終了する。

そのため、
フロントエンド側で
必ず差分を検出しなければならない
仕様とはしない。

ただし、
不要なAPI呼び出しを減らすため、
画面側で変更がない場合に
保存ボタンを非活性化してもよい。

これは、
UX上の補助的な制御とする。

---

### 30.14 バリデーションエラーの扱い

`VALIDATION_ERROR`が返却された場合は、
`error.details.field`を利用して
対象項目へエラーを表示する。

本APIの
リクエストボディで
利用者が入力する項目は、
`balance`である。

例：

```ts
if (
  detail.field === 'balance'
) {
  setFieldError(
    'balance',
    detail.message,
  );
}
```

`snapshotId`または
`assetAccountId`の形式不正は、
不正なURLまたは
画面状態として扱う。

---

### 30.15 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 入力項目または不正なURLとしてエラー表示する |
| `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` | 月末資産状況一覧画面へ戻す |
| `MONTH_END_ASSET_SNAPSHOT_CONFIRMED` | 確定済みのため編集できないことを表示する |
| `ASSET_ACCOUNT_NOT_FOUND` | 最新の一覧を再取得し、対象資産口座が存在しないことを表示する |
| `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH` | 対象年月では利用できないことを表示する |
| `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH` | 商品別評価額の管理対象であることを表示する |
| `MONTH_END_ASSET_BALANCE_NOT_FOUND` | BAL-001を再取得して登録状態を確認する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

## 31. 設計上の補足

### 31.1 更新APIを登録APIと分離する理由

月末資産残高には、

```text
未登録
```

と

```text
登録済み
```

という異なる状態が存在する。

BAL-002は
未登録状態から
新しい月末資産残高を作成する責務を持つ。

BAL-003は
すでに存在する月末資産残高を
変更する責務を持つ。

登録と更新を分離することで、
APIの責務を明確にする。

---

### 31.2 upsertを採用しない理由

BAL-003で
upsertを採用すると、
更新対象が存在しない場合にも
新しい月末資産残高が
作成されてしまう。

これでは、

```text
登録
```

と

```text
更新
```

の区別が曖昧になる。

そのため、
BAL-003では
更新対象が存在することを必須とし、
存在しない場合は

`MONTH_END_ASSET_BALANCE_NOT_FOUND`

を返却する。

---

### 31.3 assetAccountIdをパスパラメータとする理由

月末資産残高は、
月末資産状況と
資産口座の組み合わせによって
特定できる。

そのため、
更新対象を以下のURLで表現する。

```http
PATCH /api/v1/month-end-asset-snapshots/{snapshotId}/asset-balances/{assetAccountId}
```

月末資産残高IDを
クライアントへ公開しなくても、
更新対象を特定できる。

---

### 31.4 月末資産残高IDを使用しない理由

フロントエンドにとって重要なのは、

```text
どの対象年月の
どの資産口座の残高か
```

である。

内部的な
月末資産残高IDを
利用者が意識する必要はない。

そのため、
Phase1では
月末資産残高IDを
API契約へ公開しない。

---

### 31.5 PATCHを採用する理由

本APIでは、
既存の月末資産残高リソースの
`balance`のみを更新する。

リソース全体の置換ではないため、
`PUT`ではなく
`PATCH`を使用する。

```http
PATCH /api/v1/month-end-asset-snapshots/{snapshotId}/asset-balances/{assetAccountId}
```

---

### 31.6 確定済みデータを更新できない理由

確定済み月末資産状況は、
正式な資産状況として
資産推移表示や
目的達成判定に利用される。

確定済みの状態で
月末資産残高を直接変更すると、
確定時点のデータと
現在のデータに不整合が生じる。

そのため、
修正が必要な場合は、

```text
確定解除
    ↓
月末資産残高更新
    ↓
再確定
```

という手順を使用する。

---

### 31.7 対象年月時点の利用可否を再確認する理由

BAL-001で一覧取得した後に、
資産口座の設定が
変更される可能性がある。

また、
APIはフロントエンドを経由せず
直接呼び出される可能性もある。

そのため、
BAL-003実行時にも
対象年月時点の
利用可否をバックエンドで確認する。

---

### 31.8 残高記録単位を再確認する理由

フロントエンドで
口座単位の資産口座だけを
表示していたとしても、
API側で業務ルールを
保証する必要がある。

そのため、
更新実行時にも
`balance_recording_unit`を確認する。

商品単位の資産口座については、
VAL-003を使用する。

---

### 31.9 同一値への更新を許可する理由

更新前後の値が
同じであることは、
不正なリクエストではない。

例えば、
通信の再送や
画面状態のずれによって
同じ値が送信される可能性がある。

同一値を正常に受け付けることで、
PATCH操作を冪等に扱いやすくする。

---

### 31.10 0円を許可する理由

資産口座の月末残高が
実際に0円になることはあり得る。

そのため、
`balance = 0`は
正常な業務値として扱う。

未登録状態は、
月末資産残高レコードが
存在しないことで表現する。

---

### 31.11 更新によって自動確定しない理由

月末資産状況の確定には、
対象となるすべての

- 月末資産残高
- 商品別月末評価額

が揃っている必要がある。

1件の月末資産残高を
更新しただけでは、
確定条件を満たしているとは限らない。

そのため、
BAL-003から
SNP-004の処理を
自動実行しない。

---

### 31.12 目的達成判定を自動実行しない理由

月末資産残高の更新時点では、
月末資産状況は
未確定である。

未確定データを使用して
目的達成判定を
自動的に実行しない。

必要な修正を完了し、
月末資産状況を再確定した後に、
利用者が明示的に
目的達成判定を実行する。

---

### 31.13 保存済み判定履歴を再計算しない理由

目的達成判定履歴は、
判定実行時点の入力値および
結果を保持する履歴である。

後から月末資産残高を
更新したとしても、
過去に実行した判定結果を
変更しない。

そのため、
BAL-003では
`assessment_histories`を
更新しない。

---

### 31.14 冪等として扱う理由

同一の対象に対して
同じ`balance`を
複数回指定しても、
最終的なリソース状態は
同一となる。

```text
balance = 1300000
```

を何度送信しても、
最終状態は

```text
balance = 1300000
```

となる。

そのため、
BAL-003は
冪等として扱う。

---

### 31.15 Idempotency-Keyを採用しない理由

BAL-003自体が
同じ更新内容に対して
冪等である。

そのため、
Phase1では
`Idempotency-Key`を採用しない。

フロントエンド側では、
更新処理中の
保存ボタン非活性化によって
不要な連続送信を抑止する。

---

### 31.16 楽観ロックを採用しない理由

Phase1では、
認証なしの利用者を前提とし、
同一資産残高を
複数利用者が同時編集するケースを
主要要件としない。

そのため、

- `version`
- `If-Match`
- ETag

などによる
更新競合検知は採用しない。

将来的に
複数端末・複数利用者による
同時編集への対応が必要になった場合は、
楽観ロックの導入を再検討する。

---

### 31.17 キャッシュを採用しない理由

月末資産残高は、
入力作業中に
登録・更新されるデータである。

更新直後に
最新状態を確認する必要があるため、
Phase1では
アプリケーションキャッシュを採用しない。

更新成功後は、
必要に応じてBAL-001を再取得する。

---

## 32. 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)

---

## 6. パスパラメータ

| パラメータ名 | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `snapshotId` | string | ○ | 商品別月末評価額を取得する月末資産状況ID |

リクエスト例：

```http
GET /api/v1/month-end-asset-snapshots/12/holding-values
```

`snapshotId`は、
商品別月末評価額を取得する対象となる
月末資産状況を一意に識別するIDである。

対象年月は、
指定された月末資産状況の
`target_year_month`から取得する。

---

## 7. クエリパラメータ

なし。

本APIでは、
クエリパラメータを使用しない。

ページネーションは行わず、
指定された月末資産状況について
対象となる商品別月末評価額を
すべて返却する。

---

## 8. リクエストヘッダー

### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/month-end-asset-snapshots/12/holding-values
Accept: application/json
X-User-Id: 1
```

GETリクエストであるため、
`Content-Type`は必須としない。

---

## 9. リクエストボディ

なし。

本APIはGETリクエストであり、
リクエストボディを使用しない。

---

## 10. リクエスト項目

なし。

取得対象は、
以下から特定する。

```text
X-User-Id
    → 操作対象利用者

snapshotId
    → 月末資産状況

month_end_asset_snapshots.target_year_month
    → 対象年月
```

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

### 11.2 月末資産状況の存在確認

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

### 11.3 月末資産状況の確定状態

本APIは参照APIであるため、
月末資産状況が
未確定・確定済みの
どちらであっても取得可能とする。

```text
confirmed = false
    → 取得可能

confirmed = true
    → 取得可能
```

確定状態は、
本APIの取得可否条件には使用しない。

これにより、
利用者は確定前の入力状況確認と
確定後の月末資産状況確認の
両方で本APIを利用できる。

---

### 11.4 資産口座の利用者境界

一覧対象となる保有商品は、
操作対象利用者に属する
資産口座の保有商品のみに限定する。

資産口座について、
以下の条件を満たす必要がある。

```text
asset_accounts.user_id
    = 操作対象利用者ID
```

他の利用者に属する
資産口座および
その保有商品を
一覧へ含めてはならない。

---

### 11.5 対象年月時点の資産口座

一覧対象となる資産口座は、
月末資産状況の
`target_year_month`時点で
月末資産管理対象であるものに限定する。

判定には、
`asset_account_available_settings`
を使用する。

概念的には、
以下の条件となる。

```text
資産口座
+
snapshot.target_year_month
    ↓
対象年月時点で月末資産管理対象か
```

現在の資産口座の状態だけを使用して、
過去月の商品別月末評価額の
一覧対象を決定しない。

対象年月時点で
月末資産管理対象ではない
資産口座に属する保有商品は、
一覧へ含めない。

---

### 11.6 残高記録単位

一覧対象となる資産口座は、
残高記録単位が
商品単位であるものに限定する。

```text
balance_recording_unit = 商品単位
```

口座単位で残高を記録する
資産口座に属する保有商品は、
本APIの一覧対象に含めない。

```text
口座単位
    → BAL-001 月末資産残高一覧取得

商品単位
    → VAL-001 商品別月末評価額一覧取得
```

---

### 11.7 対象年月時点の保有商品

一覧対象となる保有商品は、
対象年月時点で
評価額記録対象となる
保有商品に限定する。

判定には、
保有商品の状態および
対象年月との関係を使用する。

現在の保有商品の状態だけを使用して、
過去月の商品別月末評価額の
一覧対象を決定しない。

対象年月時点で
評価額記録対象ではない保有商品は、
一覧へ含めない。

---

### 11.8 商品別月末評価額が未登録の場合

対象年月時点で
評価額記録対象となる保有商品について、
`month_end_holding_values`に
対応するレコードが存在しない場合も、
一覧対象から除外しない。

未登録の場合は、
商品別月末評価額を
`null`として扱う。

概念例：

```json
{
  "holdingAssetId": "5",
  "holdingAssetName": "全世界株式",
  "value": null
}
```

これにより、
フロントエンドは

```text
value = null
    → 未登録

value = 0
    → 0円として登録済み
```

を区別できる。

---

### 11.9 0円の扱い

商品別月末評価額として
`0`が登録されている場合は、
未登録とは扱わない。

```text
value = null
    → 未登録

value = 0
    → 登録済み

value > 0
    → 登録済み
```

一覧取得時に、
`0`を`null`へ変換してはならない。

---

### 11.10 商品別月末評価額との紐付け

商品別月末評価額は、
指定された月末資産状況と
一覧対象の保有商品との
組み合わせによって取得する。

概念的には、
以下の条件とする。

```text
month_end_asset_snapshot_id = snapshotId
AND
holding_asset_id = 対象保有商品ID
```

商品別月末評価額が
存在する場合は、
登録済みの評価額を返却する。

存在しない場合は、
対象保有商品自体を除外せず、
評価額を`null`として返却する。

---

### 11.11 X-User-Id

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

### 11.12 業務状態に依存する検証

以下は、
単項目バリデーションではなく、
一覧取得条件または
業務ルールとして扱う。

- 月末資産状況が操作対象利用者に属していること
- 資産口座が操作対象利用者に属していること
- 資産口座が対象年月時点で月末資産管理対象であること
- 資産口座の残高記録単位が商品単位であること
- 保有商品が対象年月時点で評価額記録対象であること
- 商品別月末評価額が登録済みか未登録か

本APIは一覧取得APIであるため、
個々の資産口座や保有商品が
一覧取得条件を満たさないことを理由に
リクエスト全体をエラーとはしない。

一覧取得条件を満たす
保有商品のみを抽出し、
商品別月末評価額とともに返却する。

---

## 6. パスパラメータ

| パラメータ名 | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `snapshotId` | string | ○ | 商品別月末評価額を取得する月末資産状況ID |

リクエスト例：

```http
GET /api/v1/month-end-asset-snapshots/12/holding-values
```

`snapshotId`は、
商品別月末評価額を取得する対象となる
月末資産状況を一意に識別するIDである。

対象年月は、
指定された月末資産状況の
`target_year_month`から取得する。

---

## 7. クエリパラメータ

なし。

本APIでは、
クエリパラメータを使用しない。

ページネーションは行わず、
指定された月末資産状況について
対象となる商品別月末評価額を
すべて返却する。

---

## 8. リクエストヘッダー

### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/month-end-asset-snapshots/12/holding-values
Accept: application/json
X-User-Id: 1
```

GETリクエストであるため、
`Content-Type`は必須としない。

---

## 9. リクエストボディ

なし。

本APIはGETリクエストであり、
リクエストボディを使用しない。

---

## 10. リクエスト項目

なし。

取得対象は、
以下から特定する。

```text
X-User-Id
    → 操作対象利用者

snapshotId
    → 月末資産状況

month_end_asset_snapshots.target_year_month
    → 対象年月
```

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

### 11.2 月末資産状況の存在確認

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

### 11.3 月末資産状況の確定状態

本APIは参照APIであるため、
月末資産状況が
未確定・確定済みの
どちらであっても取得可能とする。

```text
confirmed = false
    → 取得可能

confirmed = true
    → 取得可能
```

確定状態は、
本APIの取得可否条件には使用しない。

---

### 11.4 資産口座の利用者境界

一覧対象となる保有商品は、
操作対象利用者に属する
資産口座の保有商品のみに限定する。

資産口座について、
以下の条件を満たす必要がある。

```text
asset_accounts.user_id
    = 操作対象利用者ID
```

他の利用者に属する
資産口座および
その保有商品を
一覧へ含めてはならない。

---

### 11.5 対象年月時点の資産口座

一覧対象となる資産口座は、
月末資産状況の
`target_year_month`時点で
月末資産管理対象であるものに限定する。

判定には、
`asset_account_available_settings`
を使用する。

概念的には、
以下の条件となる。

```text
資産口座
+
snapshot.target_year_month
    ↓
対象年月時点で月末資産管理対象か
```

現在の資産口座の状態だけを使用して、
過去月の商品別月末評価額の
一覧対象を決定しない。

対象年月時点で
月末資産管理対象ではない
資産口座に属する保有商品は、
一覧へ含めない。

---

### 11.6 残高記録単位

一覧対象となる資産口座は、
残高記録単位が
商品単位であるものに限定する。

```text
balance_recording_unit = 商品単位
```

口座単位で残高を記録する
資産口座に属する保有商品は、
本APIの一覧対象に含めない。

```text
口座単位
    → BAL-001 月末資産残高一覧取得

商品単位
    → VAL-001 商品別月末評価額一覧取得
```

---

### 11.7 対象年月時点の保有商品

一覧対象となる保有商品は、
対象年月時点で
評価額記録対象となる
保有商品に限定する。

対象年月は、
月末資産状況の
`target_year_month`を使用する。

現在の保有商品の状態だけを使用して、
過去月の商品別月末評価額の
一覧対象を決定しない。

対象年月時点で
評価額記録対象ではない保有商品は、
一覧へ含めない。

---

### 11.8 商品別月末評価額が未登録の場合

対象年月時点で
評価額記録対象となる保有商品について、
`month_end_holding_values`に
対応するレコードが存在しない場合も、
一覧対象から除外しない。

未登録の場合は、
商品別月末評価額を
`null`として扱う。

例：

```json
{
  "holdingAssetId": "5",
  "holdingAssetName": "全世界株式",
  "value": null
}
```

これにより、
以下を区別できる。

```text
value = null
    → 未登録

value = 0
    → 0円として登録済み
```

---

### 11.9 0円の扱い

商品別月末評価額として
`0`が登録されている場合は、
未登録とは扱わない。

```text
value = null
    → 未登録

value = 0
    → 登録済み

value > 0
    → 登録済み
```

一覧取得時に、
`0`を`null`へ変換してはならない。

---

### 11.10 商品別月末評価額との紐付け

商品別月末評価額は、
指定された月末資産状況と
一覧対象の保有商品との
組み合わせによって取得する。

```text
month_end_asset_snapshot_id = snapshotId
AND
holding_asset_id = 対象保有商品ID
```

商品別月末評価額が
存在する場合は、
登録済みの評価額を返却する。

存在しない場合は、
対象保有商品自体を除外せず、
評価額を`null`として返却する。

このため、
商品別月末評価額との結合は、
未登録の保有商品が
一覧から除外されない方法で行う。

---

### 11.11 X-User-Id

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

### 11.12 一覧対象が0件の場合

一覧取得条件を満たす
保有商品が存在しない場合は、
エラーとはしない。

正常レスポンスとして、
空配列を返却する。

```json
{
  "data": []
}
```

以下の場合を含む。

- 対象年月時点で月末資産管理対象となる資産口座が存在しない
- 対象年月時点で商品単位の資産口座が存在しない
- 対象となる資産口座に保有商品が存在しない
- 対象年月時点で評価額記録対象となる保有商品が存在しない

商品別月末評価額が
未登録であることだけを理由に、
一覧を0件とはしない。

---

### 11.13 業務状態に依存する検証

以下は、
単項目バリデーションではなく、
一覧取得条件または
業務ルールとして扱う。

- 月末資産状況が操作対象利用者に属していること
- 資産口座が操作対象利用者に属していること
- 資産口座が対象年月時点で月末資産管理対象であること
- 資産口座の残高記録単位が商品単位であること
- 保有商品が対象年月時点で評価額記録対象であること
- 商品別月末評価額が登録済みか未登録か

本APIは一覧取得APIであるため、
個々の資産口座や保有商品が
一覧取得条件を満たさないことを理由に、
リクエスト全体をエラーとはしない。

一覧取得条件を満たす
保有商品のみを抽出し、
商品別月末評価額とともに返却する。

---

## 12. 業務ルール

- 指定された月末資産状況に対して、対象となる商品別月末評価額を一覧で返却する。
- 月末資産状況は、操作対象利用者に属している必要がある。
- 他の利用者に属する月末資産状況の商品別月末評価額は取得できない。
- 一覧対象となる資産口座は、操作対象利用者に属している必要がある。
- 一覧対象となる資産口座は、月末資産状況の対象年月時点で月末資産管理対象である必要がある。
- 一覧対象となる資産口座は、残高記録単位が商品単位である必要がある。
- 口座単位で残高を記録する資産口座は、本APIの一覧対象に含めない。
- 一覧対象となる保有商品は、対象年月時点で評価額記録対象となるものに限定する。
- 商品別月末評価額が未登録の保有商品も一覧へ含める。
- 商品別月末評価額が未登録の場合は、`value = null`として返却する。
- 商品別月末評価額として`0`が登録されている場合は、登録済みとして`value = 0`を返却する。
- 月末資産状況が未確定・確定済みのどちらであっても取得できる。
- 一覧対象となる保有商品が存在しない場合は、エラーとせず空配列を返却する。
- 本APIでは、商品別月末評価額の登録・更新を行わない。
- 本APIでは、月末資産状況の確定・確定解除を行わない。
- 本APIでは、目的達成判定を実行しない。
- 本APIでは、ページネーションを行わない。

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
月末資産状況取得
    ↓
存在・利用者境界確認
    ↓
target_year_month取得
    ↓
対象年月時点で月末資産管理対象となる資産口座を抽出
    ↓
残高記録単位が商品単位の資産口座に限定
    ↓
対象年月時点で評価額記録対象となる保有商品を抽出
    ↓
商品別月末評価額を結合
    ↓
商品別月末評価額未登録の場合はvalue = null
    ↓
一覧の並び順を適用
    ↓
APIレスポンス生成
    ↓
200 OK返却
```

月末資産状況の取得条件は、
以下とする。

```text
id = snapshotId
AND
user_id = 操作対象利用者ID
```

一覧対象となる資産口座は、
概念的に以下の条件を満たすものとする。

```text
user_id = 操作対象利用者ID
AND
対象年月時点で月末資産管理対象
AND
balance_recording_unit = 商品単位
```

さらに、
対象となる資産口座に属する
保有商品のうち、
対象年月時点で
評価額記録対象となるものを取得する。

商品別月末評価額は、
以下の組み合わせで紐付ける。

```text
month_end_asset_snapshot_id = snapshotId
AND
holding_asset_id = 対象保有商品ID
```

商品別月末評価額が存在しない場合も、
保有商品自体を一覧から除外しない。

---

## 14. 月末資産状況の扱い

本APIでは、
指定された月末資産状況の
`target_year_month`を
一覧取得の基準年月として使用する。

例えば、

```text
snapshotId = 12

month_end_asset_snapshots
    id = 12
    target_year_month = 2026-08
```

の場合は、
`2026-08`時点の状態をもとに
一覧対象となる資産口座および
保有商品を決定する。

月末資産状況の確定状態は、
取得可否には影響しない。

```text
confirmed = false
    → 取得可能

confirmed = true
    → 取得可能
```

これにより、
確定前の入力状況確認と
確定後の参照の
両方で本APIを利用できる。

---

## 15. 資産口座の扱い

一覧対象となる資産口座は、
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

現在の利用状態だけを使用して、
過去月の一覧対象を
決定してはならない。

例えば、

```text
2026-06
    → 月末資産管理対象

2026-07以降
    → 月末資産管理対象外
```

となっている資産口座について、
`2026-06`の月末資産状況を取得する場合は、
一覧対象として扱う。

---

## 16. 残高記録単位

資産口座の
`balance_recording_unit`によって、
使用する一覧取得APIを分ける。

```text
口座単位
    ↓
BAL-001 月末資産残高一覧取得

商品単位
    ↓
VAL-001 商品別月末評価額一覧取得
```

本APIでは、
残高記録単位が
商品単位である資産口座のみを対象とする。

口座単位の資産口座に
保有商品が存在した場合でも、
その保有商品を
本APIの一覧へ含めない。

---

## 17. 保有商品の扱い

一覧対象となる保有商品は、
対象年月時点で
評価額記録対象となるものに限定する。

例えば、
対象年月が

```text
2026-08
```

の場合は、
`2026-08`時点で
評価額記録対象となる
保有商品を一覧へ含める。

現在の保有状態だけを使用して、
過去月の一覧対象を
決定してはならない。

現在は無効化されている保有商品でも、
対象年月時点で
評価額記録対象であった場合は、
対象年月の一覧へ含める。

反対に、
現在は有効であっても、
対象年月時点で
評価額記録対象ではなかった場合は、
対象年月の一覧へ含めない。

---

## 18. 商品別月末評価額の扱い

商品別月末評価額は、
月末資産状況と
保有商品の組み合わせによって取得する。

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

対応する
`month_end_holding_values`が
存在する場合は、
登録されている評価額を返却する。

```text
商品別月末評価額あり
    ↓
value = 登録済み評価額
```

存在しない場合は、
保有商品を一覧から除外せず、
`value = null`として返却する。

```text
商品別月末評価額なし
    ↓
value = null
```

これにより、
一覧取得結果から
商品別月末評価額の
入力状況を確認できる。

---

## 19. 未登録と0円の扱い

未登録と0円は、
明確に区別する。

```text
value = null
    → 商品別月末評価額未登録

value = 0
    → 0円として登録済み

value > 0
    → 評価額登録済み
```

例えば、
以下のレスポンスは
評価額未登録を表す。

```json
{
  "holdingAssetId": "5",
  "holdingAssetName": "全世界株式",
  "value": null
}
```

以下のレスポンスは、
評価額0円として
登録済みであることを表す。

```json
{
  "holdingAssetId": "5",
  "holdingAssetName": "全世界株式",
  "value": 0
}
```

`0`を未登録として
扱ってはならない。

---

## 20. 一覧の並び順

商品別月末評価額一覧は、
画面上で安定した順序で
表示できるようにする。

基本的な並び順は、
以下とする。

```text
asset_accounts.id ASC
holding_assets.id ASC
```

同一の月末資産状況に対して
複数回取得した場合に、
データ状態が変わっていなければ
同じ順序で返却されるようにする。

Phase1では、
クライアントから
任意の並び順を指定する機能は提供しない。

---

## 21. 成功レスポンス

### 21.1 HTTPステータス

```http
200 OK
```

商品別月末評価額一覧の
取得に成功した場合は、
`200 OK`を返却する。

一覧対象が0件の場合も、
`200 OK`を返却する。

---

### 21.2 レスポンスボディ

商品別月末評価額が
登録済みの場合：

```json
{
  "data": [
    {
      "holdingAssetId": "5",
      "assetAccountId": "3",
      "holdingAssetName": "全世界株式",
      "value": 850000
    },
    {
      "holdingAssetId": "6",
      "assetAccountId": "3",
      "holdingAssetName": "S&P500",
      "value": 420000
    }
  ]
}
```

商品別月末評価額が
未登録の保有商品を含む場合：

```json
{
  "data": [
    {
      "holdingAssetId": "5",
      "assetAccountId": "3",
      "holdingAssetName": "全世界株式",
      "value": 850000
    },
    {
      "holdingAssetId": "6",
      "assetAccountId": "3",
      "holdingAssetName": "S&P500",
      "value": null
    }
  ]
}
```

0円として登録済みの場合：

```json
{
  "data": [
    {
      "holdingAssetId": "5",
      "assetAccountId": "3",
      "holdingAssetName": "全世界株式",
      "value": 0
    }
  ]
}
```

一覧対象が存在しない場合：

```json
{
  "data": []
}
```

---

## 22. レスポンス項目

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `data` | array | × | 商品別月末評価額一覧 |
| `data[].holdingAssetId` | string | × | 保有商品ID |
| `data[].assetAccountId` | string | × | 保有商品が属する資産口座ID |
| `data[].holdingAssetName` | string | × | 保有商品名 |
| `data[].value` | integer / null | ○ | 商品別月末評価額。未登録の場合は`null` |

`holdingAssetId`および
`assetAccountId`は、
API共通方針に従って
文字列として返却する。

```json
{
  "holdingAssetId": "5",
  "assetAccountId": "3"
}
```

`value`は、
日本円の整数値として返却する。

```json
{
  "value": 850000
}
```

商品別月末評価額が
未登録の場合は、
`null`として返却する。

```json
{
  "value": null
}
```

0円として登録済みの場合は、
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
- `target_year_month`
- `confirmed`
- `balance_recording_unit`
- `created_at`
- `updated_at`

月末資産状況そのものの情報は、
SNP-003 月末資産状況詳細取得APIで取得する。

資産口座単位の月末資産残高は、
BAL-001 月末資産残高一覧取得APIで取得する。

---

## 23. エラーレスポンス

異常時は、
API共通方針で定めた
共通エラーレスポンス形式を使用する。

本APIでは、
主に以下のエラーを扱う。

- 利用者コンテキスト未指定
- 利用者ID形式不正
- 利用者不存在
- `snapshotId`形式不正
- 月末資産状況不存在
- 想定外のサーバーエラー

一覧対象となる

- 資産口座が存在しない
- 商品単位の資産口座が存在しない
- 対象年月時点で評価額記録対象となる保有商品が存在しない
- 商品別月末評価額が未登録である

といった状態は、
エラーとはしない。

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

`snapshotId`が
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
        "field": "snapshotId",
        "reason": "invalidFormat",
        "message": "月末資産状況IDの形式を確認してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

以下のような場合を含む。

- `snapshotId = 0`
- `snapshotId`が負数
- `snapshotId`が小数
- `snapshotId`が数値として扱えない文字列
- API共通方針で定めたID形式に一致しない

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

他の利用者に属する
月末資産状況が存在することを
レスポンスから判別できないようにする。

---

### 23.6 一覧対象が存在しない場合

一覧取得条件を満たす
保有商品が存在しない場合は、
エラーとはしない。

```http
200 OK
```

```json
{
  "data": []
}
```

以下の場合を含む。

- 対象年月時点で月末資産管理対象となる資産口座が存在しない
- 対象年月時点で商品単位の資産口座が存在しない
- 対象資産口座に保有商品が存在しない
- 対象年月時点で評価額記録対象となる保有商品が存在しない

---

### 23.7 商品別月末評価額が未登録の場合

商品別月末評価額が
未登録であることは、
エラーとはしない。

対象となる保有商品を
一覧へ含め、
`value = null`
として返却する。

```json
{
  "data": [
    {
      "holdingAssetId": "5",
      "assetAccountId": "3",
      "holdingAssetName": "全世界株式",
      "value": null
    }
  ]
}
```

未登録状態を理由に、
`404 Not Found`などを
返却してはならない。

---

### 23.8 月末資産状況が確定済みの場合

本APIは参照APIであるため、
月末資産状況が確定済みであっても
エラーとはしない。

```text
confirmed = false
    → 200 OK

confirmed = true
    → 200 OK
```

`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`
は返却しない。

---

### 23.9 想定外のエラーが発生した場合

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
| `200 OK` | 商品別月末評価額一覧の取得に成功した |
| `400 Bad Request` | 利用者コンテキストが未指定、または利用者IDの形式が不正である |
| `404 Not Found` | 指定された利用者または月末資産状況が存在しない |
| `422 Unprocessable Entity` | `snapshotId`のバリデーションエラー |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

---

### 24.1 200の扱い

以下の場合は、
`200 OK`
を返却する。

- 商品別月末評価額一覧を正常に取得できた
- 一覧対象となる保有商品が0件
- 商品別月末評価額が未登録の保有商品が存在する
- 商品別月末評価額として0円が登録されている
- 月末資産状況が未確定
- 月末資産状況が確定済み

一覧対象が0件の場合は、
空配列を返却する。

```json
{
  "data": []
}
```

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

他の利用者に属する
月末資産状況についても、
対象リソースの存在を公開しない。

個々の資産口座、
保有商品または
商品別月末評価額が
存在しないことを理由に、
`404 Not Found`とはしない。

---

### 24.4 422の扱い

以下の場合は、
`422 Unprocessable Entity`
を返却する。

- `snapshotId`の形式が不正
- `snapshotId`が正の整数として扱えない

---

### 24.5 409を使用しない理由

本APIは、
商品別月末評価額を
参照するAPIである。

確定状態などの
業務状態によって
取得を禁止しないため、
通常の業務ルール違反として
`409 Conflict`を返却するケースは設けない。

以下は、
エラーではなく
一覧取得条件として扱う。

- 対象年月時点で月末資産管理対象ではない資産口座
- 残高記録単位が口座単位の資産口座
- 対象年月時点で評価額記録対象ではない保有商品

これらは一覧から除外する。

---

## 25. エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `USER_CONTEXT_REQUIRED` | 400 | `X-User-Id`が指定されていない | × |
| `INVALID_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `USER_NOT_FOUND` | 404 | 指定された利用者が存在しない、または論理削除されている | × |
| `VALIDATION_ERROR` | 422 | `snapshotId`が不正である | × |
| `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` | 404 | 月末資産状況が存在しない、または他利用者に属している | × |
| `INTERNAL_SERVER_ERROR` | 500 | 想定外のサーバーエラーが発生した | ○ |

エラーコードの正式な定義は、
`error-codes.md`
に従う。

本APIでは、
以下のような状態について
独自エラーコードを返却しない。

```text
対象資産口座なし
    → data = []

対象保有商品なし
    → data = []

商品別月末評価額未登録
    → value = null

商品別月末評価額0円
    → value = 0

月末資産状況確定済み
    → 通常どおり取得
```

---

## 26. 副作用

本APIは、
参照専用APIである。

データベースに対する
登録、
更新、
削除を行わない。

本APIの実行によって、
以下のデータを変更してはならない。

- `users`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `assessment_histories`

商品別月末評価額が
未登録の場合も、
一覧取得時に
`month_end_holding_values`へ
レコードを自動作成してはならない。

例えば、

```text
商品別月末評価額未登録
    ↓
VAL-001実行
    ↓
value = nullとして返却
    ↓
DBへのINSERTなし
```

とする。

また、
本APIの実行を契機として、
以下の処理を行わない。

- 月末資産状況の確定
- 月末資産状況の確定解除
- 月末資産残高の登録・更新
- 商品別月末評価額の登録・更新
- 目的達成判定の実行
- 目的達成判定履歴の登録・更新

---

## 27. トランザクション

本APIは、
参照専用APIであり、
データベースの更新処理を行わない。

そのため、
Phase1では
明示的なデータベーストランザクションを
必須としない。

基本的には、
必要なデータをQueryで取得し、
レスポンスを生成する。

ただし、
一覧取得中に
別リクエストによって
データが変更された場合、
複数SQL間で
厳密な同一時点のスナップショットを
保証するものではない。

Phase1では、
このレベルの読み取り一貫性を
追加のトランザクションによって
保証する要件は設けない。

可能な範囲で、
一覧取得に必要なデータは
JOIN等を利用して
効率的に取得する。

---

## 28. 冪等性

本APIはGETリクエストであり、
データベースの状態を変更しない。

同一のリクエストを
複数回実行しても、
本API自体による
リソース状態の変化は発生しない。

そのため、
本APIは冪等である。

例えば、

```http
GET /api/v1/month-end-asset-snapshots/12/holding-values
X-User-Id: 1
```

を複数回実行しても、
本APIの実行によって

- 商品別月末評価額が作成される
- 商品別月末評価額が更新される
- 月末資産状況が変更される
- 保有商品が変更される

ことはない。

データベースの状態が
他の処理によって変更されていなければ、
同一条件のリクエストに対して
同一内容のレスポンスを返却する。

ただし、
別のAPIによって
商品別月末評価額などが変更された場合は、
その最新状態を反映した
レスポンスとなる。

例えば、

```text
1回目のVAL-001
value = null

    ↓

VAL-002で評価額登録
value = 850000

    ↓

2回目のVAL-001
value = 850000
```

となる。

これは、
VAL-001の冪等性を
損なうものではない。

本APIでは、
`Idempotency-Key`は使用しない。

---

## 29. 関連テーブル

### 29.1 month_end_asset_snapshots

対象年月ごとの
月末資産状況および確定状態を保持する。

本APIでは、
商品別月末評価額一覧を取得する
対象の月末資産状況を特定し、
利用者境界および
対象年月を確認するために参照する。

使用する主なカラムは、
以下とする。

| カラム | 用途 |
|---|---|
| `id` | 月末資産状況ID |
| `user_id` | 利用者境界 |
| `target_year_month` | 一覧取得の対象年月 |
| `confirmed` | 月末資産状況の確定状態 |

取得条件は、
以下とする。

```text
id = snapshotId
AND
user_id = 操作対象利用者ID
```

`confirmed`の値によって、
本APIの取得可否は変更しない。

```text
confirmed = false
    → 取得可能

confirmed = true
    → 取得可能
```

本APIでは、
`month_end_asset_snapshots`を更新しない。

---

### 29.2 asset_accounts

利用者が所有する
資産口座を保持する。

本APIでは、
一覧対象となる保有商品が属する
資産口座を特定し、
以下を確認するために参照する。

- 操作対象利用者に属していること
- 残高記録単位が商品単位であること

使用する主なカラムは、
以下とする。

| カラム | 用途 |
|---|---|
| `id` | 資産口座ID |
| `user_id` | 利用者境界 |
| `balance_recording_unit` | 口座単位・商品単位の判定 |

一覧対象は、
以下の条件を満たす
資産口座に限定する。

```text
user_id = 操作対象利用者ID
AND
balance_recording_unit = 商品単位
```

本APIでは、
`asset_accounts`を更新しない。

---

### 29.3 asset_account_available_settings

資産口座について、
対象年月時点で
月末資産管理対象となるかを
判定するための設定を保持する。

本APIでは、
月末資産状況の
`target_year_month`時点で
資産口座が月末資産管理対象であることを
確認するために参照する。

概念的には、
以下を判定する。

```text
asset_account_id
+
target_year_month
    ↓
対象年月時点で月末資産管理対象か
```

現在の資産口座の状態だけを使用して、
過去月の一覧対象を
決定してはならない。

本APIでは、
`asset_account_available_settings`を
更新しない。

---

### 29.4 holding_assets

資産口座に属する
保有商品を保持する。

本APIでは、
商品別月末評価額一覧の
基準となる保有商品を取得するために参照する。

使用する主なカラムは、
以下とする。

| カラム | 用途 |
|---|---|
| `id` | 保有商品ID |
| `asset_account_id` | 所属する資産口座 |
| `name` | 保有商品名 |
| 保有期間・無効化に関するカラム | 対象年月時点の評価額記録対象判定 |

一覧対象となる保有商品は、
以下を満たすものとする。

```text
商品単位の資産口座に属している
AND
対象年月時点で評価額記録対象である
```

現在は無効化されている保有商品でも、
対象年月時点で
評価額記録対象であった場合は、
過去月の一覧へ含める。

本APIでは、
`holding_assets`を更新しない。

---

### 29.5 month_end_holding_values

保有商品ごとの
商品別月末評価額を保持する。

本APIでは、
一覧対象となる保有商品に対して
登録済みの商品別月末評価額を
取得するために参照する。

使用する主なカラムは、
以下とする。

| カラム | 用途 |
|---|---|
| `id` | 商品別月末評価額ID |
| `month_end_asset_snapshot_id` | 月末資産状況との紐付け |
| `holding_asset_id` | 保有商品との紐付け |
| `value` | 商品別月末評価額 |

商品別月末評価額は、
以下の組み合わせで取得する。

```text
month_end_asset_snapshot_id = snapshotId
AND
holding_asset_id = 対象保有商品ID
```

対応するレコードが
存在しない場合も、
保有商品を一覧から除外しない。

```text
レコードあり
    → value = 登録済み評価額

レコードなし
    → value = null
```

そのため、
一覧取得時は
未登録の保有商品を除外しない
結合方法を使用する。

同一の月末資産状況・保有商品について
複数の商品別月末評価額が
存在しないよう、
以下の組み合わせに
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

本APIでは、
`month_end_holding_values`を更新しない。

未登録の場合も、
一覧取得を契機として
レコードを自動作成しない。

---

### 29.6 users

操作対象となる
利用者を保持する。

`X-User-Id`で指定された利用者が
存在することを確認するために参照する。

本APIでは、
`users`を更新しない。

---

### 29.7 month_end_asset_balances

本APIでは、
`month_end_asset_balances`を
参照・更新しない。

資産口座単位の
月末資産残高一覧は、
BAL-001 月末資産残高一覧取得APIで取得する。

---

### 29.8 assessment_histories

本APIでは、
`assessment_histories`を
参照・登録・更新・削除しない。

商品別月末評価額一覧の取得を契機として、
目的達成判定を自動実行しない。

---

## 30. 関連する機能要件

- 月末資産管理
  - 商品単位で残高を記録する資産口座について、保有商品ごとの月末評価額を確認できる
  - 商品別月末評価額は日本円の整数値として扱う
  - 商品別月末評価額の未登録と0円を区別する
  - 商品別月末評価額が未登録の保有商品も一覧で確認できる
  - 同一月末資産状況・保有商品について商品別月末評価額は1件のみ保持する

- 資産口座管理
  - 資産口座ごとに残高記録単位を保持する
  - 残高記録単位が商品単位の資産口座のみ、本APIの対象とする
  - 対象年月時点で月末資産管理対象となる資産口座のみ対象とする

- 保有商品管理
  - 商品単位の資産口座に属する保有商品を商品別月末評価額の記録対象とする
  - 対象年月時点で評価額記録対象となる保有商品のみ一覧へ含める
  - 現在の状態だけではなく、対象年月時点の状態をもとに一覧対象を決定する

- 月末資産状況の確定
  - 未確定・確定済みのどちらの月末資産状況についても商品別月末評価額を参照できる
  - 本APIによって月末資産状況の確定状態を変更しない

- 利用者境界
  - 操作対象利用者に属する月末資産状況のみ参照できる
  - 操作対象利用者に属する資産口座および保有商品のみ一覧対象とする
  - 他の利用者に属するリソースの存在を外部へ公開しない

具体的な章番号は、
`functional-requirements.md`の
最新定義に従う。

---

## 31. テスト観点

### 31.1 正常系

- 有効な`snapshotId`で商品別月末評価額一覧を取得できること
- `200 OK`で返却されること
- 対象となる保有商品がすべて返却されること
- 登録済みの商品別月末評価額が正しく返却されること
- 商品別月末評価額が未登録の保有商品も返却されること
- 未登録の商品別月末評価額が`null`で返却されること
- 0円として登録済みの商品別月末評価額が`0`で返却されること
- 口座単位の資産口座に属する保有商品が含まれないこと
- 対象年月時点で月末資産管理対象ではない資産口座に属する保有商品が含まれないこと
- 対象年月時点で評価額記録対象ではない保有商品が含まれないこと
- 一覧取得によってデータベースが変更されないこと

---

### 31.2 snapshotId

- 正しい`snapshotId`を指定して取得できること
- `snapshotId = 1`を指定できること
- `snapshotId = 0`でバリデーションエラーとなること
- 負数でバリデーションエラーとなること
- 小数でバリデーションエラーとなること
- 文字列`abc`でバリデーションエラーとなること
- ID形式不正時に`422 Unprocessable Entity`となること
- ID形式不正時に`VALIDATION_ERROR`となること

---

### 31.3 月末資産状況の存在確認

- 存在する月末資産状況について取得できること
- 存在しない`snapshotId`で`404 Not Found`となること
- 存在しない場合に`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`となること

---

### 31.4 月末資産状況の利用者境界

- 操作対象利用者に属する月末資産状況について取得できること
- 他利用者に属する月末資産状況について取得できないこと
- 他利用者の`snapshotId`で`404 Not Found`となること
- 他利用者の場合に`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`となること
- 他利用者の月末資産状況が存在することをレスポンスから判別できないこと
- 月末資産状況の検索条件に`user_id`が含まれていること

---

### 31.5 月末資産状況の確定状態

未確定の場合：

```text
confirmed = false
```

について、

- 正常に一覧取得できること
- `200 OK`となること

確定済みの場合：

```text
confirmed = true
```

について、

- 正常に一覧取得できること
- `200 OK`となること
- `MONTH_END_ASSET_SNAPSHOT_CONFIRMED`とならないこと

---

### 31.6 資産口座の利用者境界

- 操作対象利用者に属する資産口座の保有商品のみ返却されること
- 他利用者に属する資産口座の保有商品が返却されないこと
- 他利用者の商品別月末評価額が返却されないこと
- 資産口座の抽出条件に`user_id`が含まれていること

---

### 31.7 対象年月時点の資産口座

資産口座の利用可能期間が
対象年月によって異なるデータを用意する。

以下を確認する。

- 対象年月時点で月末資産管理対象の資産口座が一覧対象となること
- 対象年月時点で月末資産管理対象ではない資産口座が一覧対象外となること
- 過去月では対象だが現在は対象外の資産口座について、該当過去月では一覧対象となること
- 現在は対象だが対象年月時点では対象外の資産口座について、一覧対象とならないこと
- 現在の利用状態だけを使用して過去月の一覧対象を判定しないこと

---

### 31.8 残高記録単位

残高記録単位が
商品単位の場合：

```text
balance_recording_unit = 商品単位
```

について、

- 保有商品が一覧対象となること

残高記録単位が
口座単位の場合：

```text
balance_recording_unit = 口座単位
```

について、

- 保有商品が存在していても一覧対象とならないこと
- リクエスト全体がエラーとならないこと

---

### 31.9 対象年月時点の保有商品

対象年月ごとに
評価額記録対象となる状態が異なる
保有商品を用意する。

以下を確認する。

- 対象年月時点で評価額記録対象の保有商品が返却されること
- 対象年月時点で評価額記録対象ではない保有商品が返却されないこと
- 現在は無効化されているが対象年月時点では対象だった保有商品が返却されること
- 現在は有効だが対象年月時点では対象ではなかった保有商品が返却されないこと
- 現在の状態だけを使用して過去月の一覧対象を判定しないこと

---

### 31.10 商品別月末評価額が登録済みの場合

以下の組み合わせで
商品別月末評価額を用意する。

```text
month_end_asset_snapshot_id = 12
holding_asset_id = 5
value = 850000
```

以下を確認する。

- 対象となる保有商品が返却されること
- `holdingAssetId = "5"`となること
- `value = 850000`となること
- 別の月末資産状況の商品別月末評価額が紐付かないこと
- 別の保有商品の商品別月末評価額が紐付かないこと

---

### 31.11 商品別月末評価額が未登録の場合

対象となる保有商品について、
対応する`month_end_holding_values`を
登録しない状態を用意する。

以下を確認する。

- 保有商品自体は一覧へ含まれること
- `value = null`となること
- `404 Not Found`とならないこと
- 一覧取得を契機として商品別月末評価額が自動登録されないこと

---

### 31.12 0円

商品別月末評価額として、

```text
value = 0
```

を登録した状態を用意する。

以下を確認する。

- 対象となる保有商品が返却されること
- `value = 0`として返却されること
- `value = null`へ変換されないこと
- 未登録として扱われないこと
- 一覧対象から除外されないこと

---

### 31.13 一覧対象が0件の場合

以下のパターンを確認する。

- 対象年月時点で月末資産管理対象となる資産口座が0件
- 商品単位の資産口座が0件
- 商品単位の資産口座に保有商品が0件
- 対象年月時点で評価額記録対象となる保有商品が0件

それぞれについて、
以下を確認する。

- `200 OK`となること
- `data`が空配列となること
- `404 Not Found`とならないこと

```json
{
  "data": []
}
```

---

### 31.14 一覧の並び順

複数の資産口座および
複数の保有商品を用意する。

以下を確認する。

- `asset_accounts.id ASC`で並ぶこと
- 同一資産口座内では`holding_assets.id ASC`で並ぶこと
- 同一データ状態で複数回取得した場合に同じ順序となること
- 商品別月末評価額の登録有無によって並び順が変化しないこと

---

### 31.15 レスポンス契約

- JSONフィールド名がcamelCaseであること
- `data`がarrayで返却されること
- `holdingAssetId`が文字列で返却されること
- `assetAccountId`が文字列で返却されること
- `holdingAssetName`が文字列で返却されること
- 登録済みの`value`がintegerで返却されること
- 未登録の`value`が`null`で返却されること
- 0円の場合に`value = 0`として返却されること
- 商品別月末評価額IDがレスポンスへ含まれないこと
- `snapshotId`がレスポンスへ含まれないこと
- `userId`がレスポンスへ含まれないこと
- `targetYearMonth`がレスポンスへ含まれないこと
- `confirmed`がレスポンスへ含まれないこと
- `createdAt`がレスポンスへ含まれないこと
- `updatedAt`がレスポンスへ含まれないこと
- DB内部のsnake_caseのカラム名がそのまま公開されないこと

---

### 31.16 副作用

本API実行によって、
以下が変更されないことを確認する。

- `users`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `assessment_histories`

以下の副作用が
発生しないことを確認する。

- 未登録の商品別月末評価額を自動作成しない
- 月末資産状況を自動確定しない
- 月末資産状況を自動確定解除しない
- 月末資産残高を登録・更新しない
- 商品別月末評価額を登録・更新しない
- 目的達成判定を自動実行しない
- 目的達成判定履歴を登録・更新しない

---

### 31.17 冪等性

同一条件で、
本APIを複数回実行する。

以下を確認する。

- すべて`200 OK`となること
- 本APIの実行によってデータベースの状態が変更されないこと
- データ状態が変更されていない場合は同一内容が返却されること
- 商品別月末評価額未登録の状態で複数回取得してもレコードが作成されないこと

---

### 31.18 N+1問題

複数の資産口座および
複数の保有商品が存在する状態で、
一覧取得を実行する。

以下を確認する。

- 保有商品ごとに個別SQLを発行しないこと
- 商品別月末評価額ごとに個別SQLを発行しないこと
- 資産口座ごとに不要な追加SQLを発行しないこと
- データ件数に比例してSQL発行回数が増加するN+1問題が発生しないこと

JOIN、
Eager Loadingまたは
必要な一括取得を利用して、
一覧取得に必要なデータを
効率的に取得する。

---

### 31.19 異常系

- `X-User-Id`未指定で`400 Bad Request`となること
- 利用者ID形式不正で`400 Bad Request`となること
- 存在しない利用者で`404 Not Found`となること
- `snapshotId`形式不正で`422 Unprocessable Entity`となること
- 存在しない月末資産状況で`404 Not Found`となること
- 他利用者の月末資産状況で`404 Not Found`となること
- 想定外の例外で`500 Internal Server Error`となること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログに同じリクエストIDが記録されること
- SQLがレスポンスへ含まれないこと
- PostgreSQLの制約名がレスポンスへ含まれないこと
- スタックトレースがレスポンスへ含まれないこと
- 内部例外メッセージがレスポンスへ含まれないこと

---

## 32. Laravel実装方針

### 32.1 Action

HTTPリクエストを受け付け、
月末資産状況IDおよび
利用者コンテキストを取得する。

パスパラメータの検証後、
商品別月末評価額一覧取得UseCaseを呼び出す。

UseCaseから受け取った一覧を、
Responderへ渡す。

Actionでは、
以下の処理を行わない。

- データベース検索
- 利用者境界の判定
- 対象年月時点の資産口座利用可否判定
- 残高記録単位の判定
- 対象年月時点の保有商品判定
- 商品別月末評価額の紐付け
- 一覧の並び順制御
- レスポンス形式への変換

---

### 32.2 UseCase

商品別月末評価額一覧取得の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 月末資産状況IDを受け取る
- 月末資産状況を取得する
- 月末資産状況が操作対象利用者に属していることを確認する
- 月末資産状況の対象年月を取得する
- 商品別月末評価額一覧を取得する
- 取得結果を返却する

指定された月末資産状況が存在しない場合、
または操作対象利用者に属していない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

月末資産状況の
`confirmed`は、
取得可否の判定には使用しない。

一覧対象となる保有商品が
存在しない場合は、
エラーとせず空のCollectionを返却する。

商品別月末評価額が
未登録の保有商品についても、
一覧へ含める。

---

### 32.3 パスパラメータ検証

`snapshotId`の形式は、
API共通方針に従って検証する。

以下を確認する。

- 指定されていること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

形式が不正な場合は、
`VALIDATION_ERROR`
として扱う。

月末資産状況の存在確認および
利用者境界の確認は、
UseCaseおよびQueryで行う。

GETリクエストであるため、
リクエストボディ用の
Form Requestは使用しない。

必要に応じて、
パスパラメータ検証用の
共通処理を利用する。

---

### 32.4 Query

商品別月末評価額一覧取得に必要な
データ検索を担当する。

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

一覧取得に必要な

- 利用者境界
- 対象年月時点の資産口座利用可否
- 残高記録単位
- 対象年月時点の保有商品
- 商品別月末評価額
- 並び順

は、
Query側で適切に検索条件へ反映する。

---

### 32.5 月末資産状況取得

月末資産状況は、
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

### 32.6 確定状態の扱い

本APIは参照APIであるため、
月末資産状況の確定状態によって
取得を制限しない。

```php
// confirmedによる取得拒否は行わない
```

以下のどちらも
正常に取得できる。

```text
confirmed = false
    → 取得可能

confirmed = true
    → 取得可能
```

そのため、
`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`
は発生させない。

---

### 32.7 対象年月の取得

一覧対象を判定する年月は、
リクエストから直接受け取らない。

月末資産状況の
`target_year_month`を使用する。

```php
$targetYearMonth =
    $snapshot->target_year_month;
```

これにより、

```text
snapshotId
    ↓
month_end_asset_snapshots
    ↓
target_year_month
    ↓
資産口座・保有商品の対象年月判定
```

という関係を保証する。

---

### 32.8 資産口座の抽出

一覧対象となる保有商品は、
操作対象利用者に属する
資産口座を起点として取得する。

最低限、
以下の条件を含める。

```php
$query->where(
    'asset_accounts.user_id',
    $userId,
);
```

さらに、
残高記録単位が
商品単位である資産口座に限定する。

概念例：

```php
$query->where(
    'asset_accounts.balance_recording_unit',
    BalanceRecordingUnit::HOLDING,
);
```

実際のEnum名・値は、
テーブル定義および
共通定義に従う。

口座単位の資産口座に属する
保有商品は、
一覧へ含めない。

---

### 32.9 対象年月時点の利用可否判定

資産口座について、
月末資産状況の
`target_year_month`時点で
月末資産管理対象であるものだけを
一覧対象とする。

判定には、
`asset_account_available_settings`
を使用する。

概念的には、
以下の条件をQueryへ組み込む。

```text
asset_account_id
+
target_year_month
    ↓
対象年月時点で月末資産管理対象
```

現在の資産口座状態のみで
過去月の一覧対象を
決定してはならない。

BAL-001などでも
同じ判定が必要になるため、
対象年月時点の利用可否判定は
共通化してよい。

---

### 32.10 保有商品の抽出

商品単位の資産口座に属する
保有商品のうち、
対象年月時点で
評価額記録対象となるものを取得する。

概念的には、
以下の条件となる。

```text
holding_assets.asset_account_id
    = 対象資産口座ID

AND

対象年月時点で評価額記録対象
```

現在の保有状態だけを使用して、
過去月の一覧対象を
決定してはならない。

現在は無効化されている保有商品でも、
対象年月時点で
評価額記録対象であった場合は、
一覧へ含める。

---

### 32.11 商品別月末評価額との結合

商品別月末評価額は、
以下の条件で紐付ける。

```text
month_end_holding_values.month_end_asset_snapshot_id
    = snapshotId

AND

month_end_holding_values.holding_asset_id
    = holding_assets.id
```

商品別月末評価額が
未登録の保有商品も
一覧へ含める必要がある。

そのため、
`month_end_holding_values`との結合には
LEFT JOIN相当の処理を使用する。

概念例：

```php
$query->leftJoin(
    'month_end_holding_values',
    function ($join) use ($snapshotId) {
        $join->on(
            'month_end_holding_values.holding_asset_id',
            '=',
            'holding_assets.id',
        )->where(
            'month_end_holding_values.month_end_asset_snapshot_id',
            '=',
            $snapshotId,
        );
    },
);
```

INNER JOINを使用して、
商品別月末評価額が
未登録の保有商品を
一覧から除外してはならない。

---

### 32.12 未登録と0円の扱い

商品別月末評価額が
存在しない場合は、
`value = null`
として取得する。

```text
month_end_holding_valuesなし
    → value = null
```

商品別月末評価額として
0円が登録されている場合は、
`value = 0`
として取得する。

```text
month_end_holding_valuesあり
value = 0
    → value = 0
```

以下のような
truthy / falsyによる判定は行わない。

```php
if (! $value) {
    // 0円まで未登録扱いになるため使用しない
}
```

未登録判定は、
`null`であるかを
明示的に確認する。

---

### 32.13 一覧の並び順

一覧の並び順は、
Queryで保証する。

基本的な並び順は、
以下とする。

```text
asset_accounts.id ASC
holding_assets.id ASC
```

概念例：

```php
$query
    ->orderBy('asset_accounts.id')
    ->orderBy('holding_assets.id');
```

Responderや
API Resource側で
並び替えを行わない。

---

### 32.14 一覧取得Queryの実装イメージ

一覧取得Queryは、
概念的に以下の責務を持つ。

```php
public function findHoldingValues(
    int $userId,
    int $snapshotId,
    string $targetYearMonth,
): Collection {
    // 利用者に属する資産口座
    // 対象年月時点で月末資産管理対象
    // 残高記録単位が商品単位
    // 対象年月時点で評価額記録対象の保有商品
    // snapshotIdに対応する商品別月末評価額をLEFT JOIN
    // asset_account_id ASC
    // holding_asset_id ASC
}
```

具体的なSQL構築は、
テーブル定義および
対象年月時点の利用可否判定方式に従う。

---

### 32.15 DTO

Queryの結果を
Eloquentモデルのまま
Responderへ渡すのではなく、
一覧表示用DTOへ変換してよい。

例：

```php
final readonly class MonthEndHoldingValueItem
{
    public function __construct(
        public int $holdingAssetId,
        public int $assetAccountId,
        public string $holdingAssetName,
        public ?int $value,
    ) {
    }
}
```

`value`は、
未登録状態を表現するため
nullableとする。

```text
null
    → 未登録

0
    → 0円として登録済み
```

---

### 32.16 Repository

本APIは参照専用であるため、
Repositoryによる
登録・更新・削除処理は行わない。

商品別月末評価額が
未登録の場合も、
Repositoryを使用して
レコードを自動作成してはならない。

```text
VAL-001
    → Queryのみ

VAL-002 / VAL-003
    → 必要に応じてRepositoryを使用
```

参照処理と更新処理の
責務を分離する。

---

### 32.17 トランザクション

本APIは参照専用であり、
データベースを更新しない。

そのため、
Phase1では
明示的なトランザクションを
必須としない。

```php
$items = $this->query->findHoldingValues(
    $userId,
    $snapshot->id,
    $snapshot->target_year_month,
);
```

一覧取得中の
厳密な読み取り一貫性を保証するための
トランザクションは使用しない。

---

### 32.18 Responder

UseCaseから受け取った
商品別月末評価額一覧を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`200 OK`とともに
`data`配列として返却する。

一覧が0件の場合も、
`200 OK`とする。

```json
{
  "data": []
}
```

Responderは、
以下の処理を行わない。

- データベース検索
- 利用者境界の判定
- 対象年月時点の利用可否判定
- 残高記録単位の判定
- 保有商品の対象判定
- 商品別月末評価額の結合
- 並び順制御

---

### 32.19 API Resource

データベースカラムを直接返却せず、
API Resourceを利用して
APIレスポンス形式へ変換する。

1件分の変換例：

```php
return [
    'holdingAssetId'
        => (string) $this->holdingAssetId,
    'assetAccountId'
        => (string) $this->assetAccountId,
    'holdingAssetName'
        => $this->holdingAssetName,
    'value'
        => $this->value === null
            ? null
            : (int) $this->value,
];
```

一覧では、
Resource Collectionを利用して返却する。

例：

```php
return MonthEndHoldingValueResource::collection(
    $items,
);
```

`value = 0`の場合も、
そのまま整数の`0`を返却する。

`null`の場合のみ、
未登録として`null`を返却する。

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

本APIで主に使用する
Eloquentモデルは、
以下とする。

```text
MonthEndAssetSnapshot
AssetAccount
AssetAccountAvailableSetting
HoldingAsset
MonthEndHoldingValue
```

必要な関連を
Eloquent Relationshipとして
定義してよい。

ただし、
Relationshipを使用することで
N+1問題が発生しないようにする。

一覧取得の条件が複雑になる場合は、
Eloquent Relationshipのみへ依存せず、
Query BuilderによるJOINを使用してよい。

---

### 32.22 N+1問題への対応

以下のように、
保有商品ごとに
商品別月末評価額を取得してはならない。

```php
foreach ($holdingAssets as $holdingAsset) {
    $value = MonthEndHoldingValue::query()
        ->where(
            'month_end_asset_snapshot_id',
            $snapshotId,
        )
        ->where(
            'holding_asset_id',
            $holdingAsset->id,
        )
        ->first();
}
```

この実装では、
保有商品の件数に比例して
SQL発行回数が増加する。

商品別月末評価額は、
JOINまたは
一括取得によって取得する。

Phase1では、
一覧取得件数が少ない場合でも、
N+1を前提とした実装は採用しない。

---

### 32.23 QueryとAPI Resourceの責務分離

Queryは、
レスポンス生成に必要な
データを取得する責務を持つ。

API Resourceは、
取得したデータを
API契約へ変換する責務を持つ。

```text
Query
    ↓
MonthEndHoldingValueItem
    ↓
API Resource
    ↓
JSON
```

Query内で
camelCaseのJSON構造を生成したり、
API Resource内で
追加のデータベース検索を
行ったりしない。

---

### 32.24 キャッシュ

Phase1では、
本API専用の
アプリケーションキャッシュを
使用しない。

商品別月末評価額は、
月末資産入力中に
登録・更新される可能性がある。

そのため、
本APIでは
実行時点のデータベース状態を
取得して返却する。

将来的に
参照量やデータ量が増加し、
性能上の必要性が生じた場合は、
キャッシュ戦略を別途検討する。

---

### 32.25 例外変換

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
| `snapshotId`形式不正 | `VALIDATION_ERROR` |
| 月末資産状況不存在 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 利用者境界外の月末資産状況 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

以下の状態は、
例外へ変換しない。

```text
対象資産口座なし
    → 空配列

対象保有商品なし
    → 空配列

商品別月末評価額未登録
    → value = null

商品別月末評価額0円
    → value = 0

月末資産状況確定済み
    → 通常どおり取得
```

SQL、
スタックトレース、
PostgreSQLの内部情報および
内部例外メッセージは、
APIレスポンスへ含めない。

ログには、
調査に必要な範囲で
以下を記録する。

- 操作対象利用者ID
- 月末資産状況ID
- 対象年月
- 独自エラーコード
- リクエストID

---

## 33. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type GetMonthEndHoldingValuesParams = {
  snapshotId: string;
};
```

一覧項目の型は、
以下とする。

```ts
export type MonthEndHoldingValueListItem = {
  holdingAssetId: string;
  assetAccountId: string;
  holdingAssetName: string;
  value: number | null;
};
```

レスポンス型は、
以下とする。

```ts
export type GetMonthEndHoldingValuesResponse = {
  data: MonthEndHoldingValueListItem[];
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetMonthEndHoldingValuesResponse>(
    `/api/v1/month-end-asset-snapshots/${snapshotId}/holding-values`,
  );
```

取得した一覧は、
商品別月末評価額一覧画面や
商品別月末評価額入力画面で利用する。

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

商品別月末評価額の
登録・更新対象を特定する際に使用する。

フロントエンド側で
数値へ変換する必要はない。

---

### 33.3 assetAccountIdの扱い

`assetAccountId`は、
保有商品が属する
資産口座を識別するために使用する。

```ts
const assetAccountId: string = '3';
```

商品別月末評価額一覧を
資産口座単位でグループ表示する場合などに
利用してよい。

ただし、
フロントエンド側で
利用者境界や
残高記録単位の最終判定には使用しない。

---

### 33.4 valueの扱い

`value`は、
以下の状態を表す。

```text
number
    → 商品別月末評価額登録済み

0
    → 0円として登録済み

null
    → 未登録
```

TypeScriptでは、
以下のように明示的に判定する。

```ts
if (item.value === null) {
  // 未登録
} else {
  // 登録済み
}
```

以下のような
truthy / falsyによる判定は行わない。

```ts
if (!item.value) {
  // value = 0 も未登録扱いになるため使用しない
}
```

---

### 33.5 金額表示

登録済みの`value`は、
日本円の整数値として表示する。

```ts
const formatCurrency = (
  value: number,
): string =>
  new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY',
    maximumFractionDigits: 0,
  }).format(value);
```

使用例：

```ts
const displayValue =
  item.value === null
    ? '未登録'
    : formatCurrency(item.value);
```

`value = null`を
0円として表示してはならない。

---

### 33.6 未登録状態の表示

`value = null`の場合は、
商品別月末評価額が
未登録であることを表示する。

表示例：

```text
未登録
```

必要に応じて、
VAL-002 商品別月末評価額登録APIを
利用する入力画面への導線を表示する。

---

### 33.7 0円の表示

`value = 0`の場合は、
登録済みの0円として表示する。

表示例：

```text
￥0
```

未登録とは扱わない。

そのため、

```text
value === null
```

と

```text
value === 0
```

を必ず区別する。

---

### 33.8 登録・更新APIの切り替え

VAL-001のレスポンスを使用して、
商品別月末評価額が
登録済みか未登録かを判定できる。

```ts
if (item.value === null) {
  // VAL-002 商品別月末評価額登録
} else {
  // VAL-003 商品別月末評価額更新
}
```

`value = 0`の場合は、
登録済みであるため
VAL-003を使用する。

以下の判定は行わない。

```ts
if (!item.value) {
  // 0円を未登録と誤判定するため使用しない
}
```

---

### 33.9 月末資産状況の確定状態

VAL-001のレスポンスには、
月末資産状況の`confirmed`は
含まれない。

編集可否を
画面上で補助的に制御する場合は、
SNP-003 月末資産状況詳細取得APIから
確定状態を取得する。

```text
SNP-003
    → confirmed取得

VAL-001
    → 商品別月末評価額一覧取得
```

例えば、

```ts
const canEdit =
  !snapshot.confirmed;
```

として、
確定済みの場合は
入力欄を非活性化してよい。

ただし、
登録・更新可否の最終判断は
VAL-002およびVAL-003で行う。

---

### 33.10 一覧のグループ表示

レスポンスには
`assetAccountId`が含まれるため、
資産口座単位で
商品別月末評価額を
グループ表示してよい。

例えば、

```text
証券口座A
    全世界株式       ￥850,000
    S&P500           未登録

証券口座B
    国内株式         ￥300,000
```

のように表示できる。

ただし、
VAL-001では
資産口座名を返却しない。

資産口座名が必要な場合は、
資産口座一覧取得API等で取得した
フロントエンド側のデータと
`assetAccountId`で対応付ける。

---

### 33.11 一覧順の扱い

APIレスポンスは、
以下の順序で返却される。

```text
assetAccountId相当の順序
    ↓
holdingAssetId相当の順序
```

具体的には、
バックエンドで

```text
asset_accounts.id ASC
holding_assets.id ASC
```

を適用する。

フロントエンドでは、
原則として
APIが返却した順序を
そのまま利用する。

同一の並び順ロジックを
フロントエンドへ
重複実装しない。

---

### 33.12 データなしの扱い

一覧対象となる
保有商品が存在しない場合は、
以下が返却される。

```json
{
  "data": []
}
```

フロントエンドでは、
エラーとして扱わない。

表示例：

```text
この対象年月には、
商品別評価額を記録する
保有商品がありません。
```

---

### 33.13 ローディング表示

一覧取得中は、
ローディング状態を表示する。

例えば、
React Query等を使用する場合は、
取得状態に応じて表示を切り替える。

```ts
if (isLoading) {
  return <Loading />;
}
```

`snapshotId`が変更された場合は、
以前の対象年月の一覧を
現在のデータとして
誤って表示しないようにする。

---

### 33.14 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 不正なURLまたはパラメータとして扱う |
| `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` | 月末資産状況一覧画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

商品別月末評価額が
未登録であること自体は、
エラーとして扱わない。

---

### 33.15 再取得

以下の操作後は、
必要に応じて
VAL-001を再取得する。

- VAL-002 商品別月末評価額登録
- VAL-003 商品別月末評価額更新

React Query等を使用する場合は、
対象の`snapshotId`に対応する
商品別月末評価額一覧クエリを
invalidateしてよい。

例：

```ts
queryClient.invalidateQueries({
  queryKey: [
    'monthEndHoldingValues',
    snapshotId,
  ],
});
```

これにより、
登録・更新後の最新状態を
一覧へ反映する。

---

## 34. 設計上の補足

### 34.1 保有商品を基準に一覧生成する理由

本APIの目的は、
登録済みの商品別月末評価額だけを
取得することではない。

対象年月時点で
評価額入力が必要な保有商品と、
その入力状況を確認することが目的である。

そのため、

```text
month_end_holding_values
```

を基準にするのではなく、

```text
対象年月時点のholding_assets
```

を基準として一覧を生成する。

これにより、
商品別月末評価額が
未登録の保有商品も
一覧へ含めることができる。

---

### 34.2 未登録をnullで表現する理由

未登録状態と
0円登録済みを
明確に区別する必要がある。

そのため、

```text
null
    → 未登録

0
    → 0円として登録済み
```

として扱う。

`0`を未登録として使用すると、
評価額が実際に0円であるケースを
正しく表現できない。

---

### 34.3 商品別月末評価額IDを返却しない理由

商品別月末評価額が
未登録の場合は、
商品別月末評価額ID自体が存在しない。

一方、
フロントエンドが操作対象として
必要とするのは、

```text
月末資産状況
+
保有商品
```

である。

そのため、
Phase1では
内部の商品別月末評価額IDを
API契約へ公開しない。

登録・更新対象は、
`snapshotId`および
`holdingAssetId`を使用して特定する。

---

### 34.4 assetAccountIdを返却する理由

保有商品は、
必ず資産口座に属する。

商品別月末評価額一覧では、
同じ月末資産状況内に
複数の商品単位資産口座が
存在する可能性がある。

そのため、
保有商品が
どの資産口座に属するかを
フロントエンドで判別できるよう、
`assetAccountId`を返却する。

---

### 34.5 assetAccountNameを返却しない理由

資産口座名は、
資産口座リソースの属性である。

VAL-001の主目的は、
保有商品と商品別月末評価額の
一覧取得であるため、
Phase1では
資産口座名を重複して返却しない。

必要な場合は、
資産口座一覧取得APIの結果と
`assetAccountId`で対応付ける。

---

### 34.6 targetYearMonthを返却しない理由

対象年月は、
親リソースとなる
月末資産状況に保持されている。

URLの`snapshotId`によって
月末資産状況が特定されているため、
各一覧項目へ
同じ対象年月を重複して返却しない。

必要な場合は、
SNP-003 月末資産状況詳細取得APIから
取得する。

---

### 34.7 confirmedを返却しない理由

`confirmed`は、
月末資産状況の属性であり、
商品別月末評価額一覧項目の
属性ではない。

そのため、
VAL-001では返却しない。

編集可否が必要な場合は、
SNP-003のレスポンスと
組み合わせて利用する。

---

### 34.8 口座単位の資産口座を除外する理由

残高記録単位が口座単位の場合は、
資産口座そのものの
月末資産残高を記録する。

そのため、
その資産口座に保有商品が存在していても、
VAL-001では
商品別評価額の記録対象としない。

```text
口座単位
    → BAL系API

商品単位
    → VAL系API
```

として責務を分離する。

---

### 34.9 対象年月時点の状態を使用する理由

資産口座や保有商品は、
現在の状態と
過去の対象年月時点の状態が
一致するとは限らない。

例えば、
現在は無効化されている商品でも、
過去月には
評価額記録対象だった可能性がある。

そのため、
月末資産状況の
`target_year_month`を基準として
一覧対象を決定する。

---

### 34.10 LEFT JOINを利用する理由

商品別月末評価額が
未登録の保有商品も
一覧へ含める必要がある。

そのため、
`holding_assets`と
`month_end_holding_values`を
結合する場合は、
LEFT JOINを使用する。

```text
holding_assets
    LEFT JOIN
month_end_holding_values
```

INNER JOINを使用すると、
商品別月末評価額が
未登録の保有商品が
一覧から消えてしまうため、
使用しない。

---

### 34.11 GETを採用する理由

本APIは、
商品別月末評価額一覧を
参照するだけであり、
サーバー側の状態を変更しない。

そのため、
HTTPメソッドには
`GET`を採用する。

本APIは冪等であり、
`Idempotency-Key`は使用しない。

---

### 34.12 ページネーションを採用しない理由

Phase1では、
1利用者が1つの月末資産状況で扱う
保有商品数は限定的であることを
前提とする。

そのため、
商品別月末評価額一覧には
ページネーションを採用しない。

将来的に
大量の商品を扱う要件が発生した場合は、
導入を再検討する。

---

### 34.13 API側で並び順を保証する理由

一覧の並び順を
フロントエンド側だけで管理すると、
複数画面で同じ並び順を
重複実装する可能性がある。

そのため、
Phase1では
バックエンド側で
安定した一覧順を保証する。

フロントエンドは、
原則として
返却された順序をそのまま使用する。

---

### 34.14 キャッシュを採用しない理由

商品別月末評価額は、
月末資産入力作業中に
登録・更新されるデータである。

また、
Phase1ではデータ量および
アクセス頻度も限定的である。

そのため、
アプリケーションキャッシュは
採用しない。

VAL-002またはVAL-003成功後は、
必要に応じて
VAL-001を再取得する。

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
