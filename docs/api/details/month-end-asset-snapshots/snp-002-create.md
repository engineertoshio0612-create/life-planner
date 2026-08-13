# SNP-002 月末資産状況作成

## 1. 概要

操作対象となる利用者について、
指定した対象年月の
月末資産状況を作成する。

月末資産状況は、
対象年月ごとの月末資産残高および
商品別月末評価額を管理するための
親リソースとして扱う。

作成直後の月末資産状況は、
未確定状態とする。

本APIでは、
月末資産状況のみ作成し、
月末資産残高および
商品別月末評価額は登録しない。

作成後は、
資産口座の残高記録単位に応じて、
月末資産残高APIまたは
商品別月末評価額APIを使用して
対象年月の資産額を登録する。

---

## 2. ユースケース

利用者は、
新しい対象年月の
月末資産状況を作成する。

例えば、
以下のような場合に使用する。

- 新しい月の月末資産入力を開始する
- 月末資産残高を登録するための対象年月を作成する
- 商品別月末評価額を登録するための対象年月を作成する

月末資産状況を作成しただけでは、
正式な資産状況とは扱わない。

必要な残高情報を登録し、
確定条件を満たした後に
SNP-004 月末資産状況確定APIを実行する。

---

## 3. エンドポイント

```http
POST /api/v1/month-end-asset-snapshots
```

---

## 4. HTTPメソッド

```text
POST
```

本APIは、新しい月末資産状況リソースを作成する。

一覧取得、月末資産残高登録、商品別月末評価額登録、確定および確定解除は行わない。

---

## 5. 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、`X-User-Id`リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

作成する月末資産状況は、指定された利用者へ紐付ける。

他の利用者の月末資産状況として作成することはできない。

利用者IDは、リクエストボディ、クエリパラメータまたはパスパラメータでは受け付けない。

利用者IDは、ミドルウェアで設定された利用者コンテキストから取得する。

作成直後の確定状態は、未確定とする。

```text
confirmed = false
```

確定状態は、リクエストから指定できない。

登録日時および更新日時は、サーバー側で設定する。

以下の項目は、クライアントから指定できない。

- `id`
- `userId`
- `confirmed`
- `confirmedAt`
- `createdAt`
- `updatedAt`

---

## 6. パスパラメータ

なし。

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

リクエスト例

```http
POST /api/v1/month-end-asset-snapshots
Content-Type: application/json
Accept: application/json
X-User-Id: 1

```

---

## 9. リクエストボディ

{
  "targetYearMonth": "2026-08"
}

---

## 10. リクエスト項目

| 項目 | 型 | 必須 | NULL | 説明 |
| --- | --- | :---: | :---: | --- |
| `targetYearMonth` | string | ○ | × | 月末資産状況を作成する対象年月。`YYYY-MM`形式 |

以下の項目は、リクエストボディでは受け付けない。

- `id`
- `userId`
- `confirmed`
- `confirmedAt`
- `createdAt`
- `updatedAt`

利用者IDは、`X-User-Id`から取得する。

確定状態は、サーバー側で`false`を設定する。

---

## 11. バリデーション

### 11.1 targetYearMonth

`targetYearMonth`は、必須項目とする。

以下を検証する。

- 指定されていること
- `null`でないこと
- 文字列であること
- `YYYY-MM`形式であること
- 実在する年月であること
- APIで許容する年月範囲内であること

正常例：

```text
2026-08
```

不正例：

```text
2026-8
2026/08
202608
26-08
2026-13
```

形式または値が不正な場合は、`VALIDATION_ERROR`として扱う。

---

### 11.2 対象年月の重複

同一利用者について、同じ`targetYearMonth`の月末資産状況を複数作成することはできない。

以下の組み合わせは、一意でなければならない。

```text
user_id
target_year_month
```

すでに同一対象年月の月末資産状況が存在する場合は、重複エラーとして扱う。

重複確認は、アプリケーション側だけでなく、データベースのUNIQUE制約でも保証する。

---

### 11.3 confirmed

`confirmed`は、リクエストから受け付けない。

作成時は、サーバー側で必ず`false`を設定する。

```text
confirmed = false
```

クライアントから確定済みの月末資産状況を直接作成することはできない。

月末資産状況の確定は、SNP-004 月末資産状況確定APIを使用する。

---

### 11.4 X-User-Id

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

`X-User-Id`が指定されていない場合は、`USER_CONTEXT_REQUIRED`として扱う。

形式が不正な場合は、`INVALID_USER_ID`として扱う。

指定された利用者が存在しない場合、または論理削除されている場合は、`USER_NOT_FOUND`として扱う。

---

### 11.5 未定義項目

定義されていない項目がリクエストボディに含まれている場合の扱いは、API共通方針に従う。

特に、以下のサーバー管理項目をクライアントから変更できないようにする。

- `id`
- `userId`
- `confirmed`
- `confirmedAt`
- `createdAt`
- `updatedAt`

---

### 11.6 業務状態に依存する検証

以下は、単項目バリデーションではなく、業務ルールとして検証する。

- 同一利用者・同一対象年月の月末資産状況がすでに存在しないこと
- 指定された対象年月について新しい月末資産状況を作成可能であること

これらの検証は、Form Requestなどの入力値検証ではなく、UseCaseで実施する。

---

## 12. 業務ルール

- 月末資産状況は、操作対象利用者単位で管理する。
- 月末資産状況は、対象年月ごとに1件のみ作成できる。
- 同一利用者・同一対象年月の月末資産状況を重複して作成することはできない。
- 作成時の月末資産状況は、必ず未確定状態とする。
- クライアントから確定状態を指定することはできない。
- 本APIでは、月末資産状況のみ作成する。
- 月末資産残高は、本APIでは登録しない。
- 商品別月末評価額は、本APIでは登録しない。
- 月末資産状況を作成しただけでは、正式な資産状況として扱わない。
- 月末資産状況の確定は、SNP-004 月末資産状況確定APIで行う。
- 月末資産状況の作成時点では、確定条件を満たしているかどうかを判定しない。
- 月末資産状況は、対象年月の順序にかかわらず作成できる。
- 月末資産状況の作成順と確定順は分離して扱う。
- 対象年月を飛ばして月末資産状況を作成することは許可する。
- 対象年月を飛ばして確定できるかどうかは、SNP-004で判定する。
- 作成済みの月末資産状況は、本APIによって上書きしない。

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
リクエストボディ検証
    ↓
同一利用者・同一対象年月の重複確認
    ↓
月末資産状況作成
    ├─ user_id = 操作対象利用者ID
    ├─ target_year_month = 指定された対象年月
    └─ confirmed = false
    ↓
作成した月末資産状況取得
    ↓
APIレスポンス生成
    ↓
201 Created返却
```

---

同一利用者・同一対象年月の月末資産状況がすでに存在する場合は、新しいレコードを作成しない。

月末資産残高および商品別月末評価額の自動生成は行わない。

---

## 14. トランザクション境界

月末資産状況の作成処理は、データベーストランザクション内で実行する。

トランザクション内では、以下を実行する。

1. 同一利用者・同一対象年月の重複確認
2. 月末資産状況の作成

処理途中で例外が発生した場合は、月末資産状況の作成をロールバックする。

本APIでは、以下の登録・更新を同一トランザクション内で行わない。

- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品
- 利用可能資産設定

---

## 15. 排他制御

Phase1では、明示的な悲観ロックおよび楽観ロックは採用しない。

同一利用者・同一対象年月に対する同時作成については、データベースのUNIQUE制約によって重複登録を防止する。

一意制約は、以下の組み合わせに設定する。

```text
user_id
target_year_month
```

アプリケーション側で事前に重複確認を行った場合でも、同時実行によって複数のリクエストが重複確認を通過する可能性がある。

そのため、最終的な一意性はデータベース制約で保証する。

UNIQUE制約違反が発生した場合は、PostgreSQLの例外をそのまま返却せず、APIで定義した月末資産状況重複エラーへ変換する。

---

## 16. 成功レスポンス

### 16.1 HTTPステータス

```text
201 Created
```

新しい月末資産状況リソースを作成するため、`201 Created`を返却する。

---

### 16.2 レスポンスボディ

```json
{
  "data": {
    "id": "12",
    "targetYearMonth": "2026-08",
    "confirmed": false
  }
}
```

作成直後のため、`confirmed`は必ず`false`となる。

---

## 17. レスポンス項目

| 項目 | 型 | NULL | 説明 |
| --- | --- | :---: | --- |
| `data` | object | × | 作成した月末資産状況 |
| `data.id` | string | × | 月末資産状況ID |
| `data.targetYearMonth` | string | × | 対象年月（YYYY-MM） |
| `data.confirmed` | boolean | × | 確定状態。作成直後は`false` |

月末資産状況IDは、API共通方針に従って文字列として返却する。

`targetYearMonth`は、`YYYY-MM`形式で返却する。

`confirmed`は、作成直後の月末資産状況が未確定であるため、`false`を返却する。

本APIでは、
以下の情報は返却しない。

- `user_id`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `holding_assets`
- `asset_account_available_settings`
- `created_at`
- `updated_at`

月末資産残高および商品別月末評価額は、月末資産状況作成後にそれぞれの登録APIを使用して登録する。

---

## 18. エラーレスポンス

異常時は、
API共通方針で定めた
共通エラーレスポンス形式を使用する。

本APIでは、
入力値の不正、
利用者不存在、
同一対象年月の重複および
想定外のサーバーエラーを扱う。

エラーが発生した場合は、
月末資産状況を作成しない。

---

### 18.1 利用者コンテキストが指定されていない場合

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

### 18.2 利用者ID形式が不正な場合

`X-User-Id`が
API共通方針で定めたID形式に一致しない場合は、
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

### 18.3 利用者が存在しない場合

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

### 18.4 バリデーションエラー

`targetYearMonth`が
バリデーション条件を満たさない場合は、
`VALIDATION_ERROR`
を返却する。

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "targetYearMonth",
        "reason": "format",
        "message": "対象年月はYYYY-MM形式で指定してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

以下のような場合を含む。

- `targetYearMonth`が未指定
- `targetYearMonth`が`null`
- `targetYearMonth`が文字列ではない
- `targetYearMonth`が`YYYY-MM`形式ではない
- 実在しない年月が指定されている
- 許容範囲外の年月が指定されている

---

### 18.5 同一対象年月の月末資産状況が存在する場合

操作対象利用者について、
指定された対象年月の
月末資産状況がすでに存在する場合は、
`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`
を返却する。

```json
{
  "error": {
    "code": "MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS",
    "message": "指定された対象年月の月末資産状況はすでに存在します。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

アプリケーション側の
事前重複確認で検知した場合と、
データベースのUNIQUE制約違反で
検知した場合は、
同じエラーコードへ変換する。

既存の月末資産状況を
本APIで上書きしない。

---

### 18.6 想定外のエラーが発生した場合

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

## 19. HTTPステータス

| HTTPステータス | 条件 |
|---:|---|
| `201 Created` | 月末資産状況の作成に成功した |
| `400 Bad Request` | 利用者コンテキストが未指定、または利用者IDの形式が不正である |
| `404 Not Found` | 指定された利用者が存在しない |
| `409 Conflict` | 同一利用者・同一対象年月の月末資産状況がすでに存在する |
| `422 Unprocessable Entity` | リクエスト項目のバリデーションエラー |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

---

### 19.1 201の扱い

新しい月末資産状況リソースの
作成に成功した場合は、
`201 Created`
を返却する。

作成された月末資産状況は、
必ず未確定状態とする。

```text
confirmed = false
```

---

### 19.2 400の扱い

以下の場合は、
`400 Bad Request`
を返却する。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正である

---

### 19.3 404の扱い

以下の場合は、
`404 Not Found`
を返却する。

- 指定された利用者が存在しない
- 指定された利用者が論理削除されている

---

### 19.4 409の扱い

以下の場合は、
`409 Conflict`
を返却する。

- 同一利用者・同一対象年月の月末資産状況がすでに存在する
- 同時実行によってデータベースのUNIQUE制約違反が発生した

リクエスト自体は正しいが、
現在のリソース状態と競合しているため、
`409 Conflict`として扱う。

---

### 19.5 422の扱い

以下の場合は、
`422 Unprocessable Entity`
を返却する。

- `targetYearMonth`が未指定
- `targetYearMonth`が`null`
- `targetYearMonth`の型が不正
- `targetYearMonth`の形式が不正
- 実在しない年月が指定されている
- 許容範囲外の年月が指定されている

---

## 20. エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `USER_CONTEXT_REQUIRED` | 400 | `X-User-Id`が指定されていない | × |
| `INVALID_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `USER_NOT_FOUND` | 404 | 指定された利用者が存在しない、または論理削除されている | × |
| `MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS` | 409 | 同一利用者・同一対象年月の月末資産状況がすでに存在する | × |
| `VALIDATION_ERROR` | 422 | リクエスト項目がバリデーション条件を満たさない | × |
| `INTERNAL_SERVER_ERROR` | 500 | 想定外のサーバーエラーが発生した | ○ |

エラーコードの正式な定義は、
`error-codes.md`
に従う。

`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`は、
アプリケーション側の重複確認と
データベースのUNIQUE制約違反の
どちらで検知した場合も使用する。

---

## 21. 冪等性

本APIは、
HTTP POSTを使用して
新しい月末資産状況を作成する。

そのため、
HTTPメソッドとしては
冪等ではない。

ただし、
同一利用者・同一対象年月について
複数の月末資産状況を作成することは禁止する。

同じ内容のリクエストを
再度実行した場合は、
既存の月末資産状況を返却したり、
上書きしたりせず、
`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`
を返却する。

```text
1回目
POST targetYearMonth = 2026-08
    ↓
201 Created

2回目
POST targetYearMonth = 2026-08
    ↓
409 Conflict
MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS
```

同時に複数の作成リクエストが
実行された場合も、
データベースのUNIQUE制約によって
1件のみ作成されることを保証する。

Phase1では、
`Idempotency-Key`は採用しない。

フロントエンドでは、
作成処理中に
登録ボタンを非活性化し、
意図しない二重送信を防止する。

ただし、
二重送信が発生した場合でも、
データベース制約によって
重複レコードが作成されないことを
最終的に保証する。

---

## 22. 関連テーブル

### 22.1 month_end_asset_snapshots

対象年月ごとの
月末資産状況および確定状態を保持する。

本APIで
新規レコードを登録するテーブルである。

使用する主なカラムは、
以下とする。

| カラム | 用途 |
|---|---|
| `id` | 月末資産状況ID |
| `user_id` | 操作対象利用者ID |
| `target_year_month` | 対象年月 |
| `confirmed` | 確定状態 |
| `created_at` | 登録日時 |
| `updated_at` | 更新日時 |

登録時は、
以下の値を設定する。

```text
user_id           = 操作対象利用者ID
target_year_month = リクエストのtargetYearMonth
confirmed         = false
```

`id`、`created_at`および`updated_at`は、サーバー側で設定する。

同一利用者・同一対象年月の重複登録を防止するため、以下の組み合わせにUNIQUE制約を設定する。

```text
user_id
target_year_month
```

本APIでは、既存の月末資産状況を更新しない。

---

### 22.2 month_end_asset_balances

資産口座単位の月末資産残高を保持する。

本APIでは、月末資産状況の作成と同時に月末資産残高を登録しない。

月末資産状況作成後に、月末資産残高登録APIを使用して必要な残高を登録する。

---

### 22.3 month_end_holding_values

保有商品単位の商品別月末評価額を保持する。

本APIでは、月末資産状況の作成と同時に商品別月末評価額を登録しない。

月末資産状況作成後に、商品別月末評価額登録APIを使用して必要な評価額を登録する。

---

### 22.4 users

操作対象となる利用者を保持する。

`X-User-Id`で指定された利用者が存在することを確認するために参照する。

作成する月末資産状況の`user_id`には、操作対象利用者IDを設定する。

本APIでは、`users`を更新しない。

---

## 23. 関連する機能要件

- 月末資産管理
  - 利用者は対象年月ごとの月末資産状況を作成できる
  - 月末資産状況は利用者単位で管理する
  - 同一利用者・同一対象年月の月末資産状況は1件のみ保持する
  - 作成した月末資産状況へ月末資産残高または商品別月末評価額を登録する
- 月末資産状況の確定
  - 作成直後の月末資産状況は未確定とする
  - 月末資産状況を作成しただけでは正式な資産状況として扱わない
  - 必要な残高情報を登録した後に月末資産状況を確定する
  - 月末資産状況の作成と確定は別操作として扱う
- 利用者境界
  - 操作対象利用者に紐づく月末資産状況のみ作成する
  - 他利用者の月末資産状況として作成できない

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

## 24. テスト観点

### 24.1 正常系

- 有効な対象年月を指定して月末資産状況を作成できること
- `201 Created`で返却されること
- 月末資産状況が1件登録されること
- 指定した`targetYearMonth`が保存されること
- 操作対象利用者IDが`user_id`へ設定されること
- 作成直後の`confirmed`が`false`となること
- `created_at`が設定されること
- `updated_at`が設定されること
- 月末資産残高が同時に登録されないこと
- 商品別月末評価額が同時に登録されないこと

---

### 24.2 利用者境界

- `X-User-Id`で指定した利用者に紐づいて作成されること
- リクエストボディから利用者IDを指定できないこと
- 他利用者の月末資産状況として作成できないこと
- 同じ対象年月でも利用者が異なる場合は、それぞれ作成できること

例えば、以下は許可する。

```text
user_id = 1, target_year_month = 2026-08
user_id = 2, target_year_month = 2026-08
```

---

### 24.3 対象年月

- `YYYY-MM`形式の対象年月を指定できること
- `2026-01`を指定できること
- `2026-12`を指定できること
- `2026-1`がバリデーションエラーとなること
- `2026/01`がバリデーションエラーとなること
- `202601`がバリデーションエラーとなること
- `2026-00`がバリデーションエラーとなること
- `2026-13`がバリデーションエラーとなること
- 未指定でバリデーションエラーとなること
- `null`でバリデーションエラーとなること
- 文字列以外でバリデーションエラーとなること
- APIで定義した許容範囲外の年月がバリデーションエラーとなること

---

### 24.4 重複

- 同一利用者・同一対象年月を重複して作成できないこと
- 重複時に`409 Conflict`となること
- 重複時に`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`となること
- 重複時に既存レコードが上書きされないこと
- 重複時に新しいレコードが作成されないこと
- 異なる利用者で同一対象年月を作成できること
- 同一利用者で異なる対象年月を作成できること

---

### 24.5 作成順

- 既存の最新対象年月より後の年月を作成できること
- 既存の最新対象年月より前の年月を作成できること
- 対象年月を飛ばして作成できること
- 作成順にかかわらず作成直後は未確定となること
- 作成順序を理由として本APIが確定可否を判定しないこと

例えば、`2026-06`が存在する状態で`2026-08`を作成できる。

```text
2026-06
2026-08
```

`2026-07`を飛ばして作成したこと自体は、エラーとしない。

確定順序については、SNP-004 月末資産状況確定APIで判定する。

---

### 24.6 確定状態

- 作成時に`confirmed = false`となること
- クライアントから`confirmed = true`を指定して作成できないこと
- クライアントから`confirmed = false`を指定する必要がないこと
- 月末資産状況作成時に確定処理が実行されないこと
- 作成時に確定条件の判定が実行されないこと

---

### 24.7 サーバー管理項目

- `id`をクライアントから指定できないこと
- `userId`をクライアントから指定できないこと
- `confirmed`をクライアントから指定できないこと
- `confirmedAt`をクライアントから指定できないこと
- `createdAt`をクライアントから指定できないこと
- `updatedAt`をクライアントから指定できないこと
- サーバー管理項目を指定した場合の扱いがAPI共通方針と一致すること

---

### 24.8 同時作成

- 同一利用者・同一対象年月について複数の作成要求を同時実行しても1件のみ登録されること
- UNIQUE制約によって最終的な重複登録が防止されること
- 1件が`201 Created`となること
- 競合したリクエストが`409 Conflict`となること
- 競合時に`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`へ変換されること
- PostgreSQLの制約違反情報がそのままレスポンスへ公開されないこと

---

### 24.9 トランザクション

- 月末資産状況の作成がトランザクション内で実行されること
- 登録処理中に例外が発生した場合はロールバックされること
- 登録失敗時に不完全な月末資産状況が残らないこと
- 月末資産残高がトランザクション内で自動生成されないこと
- 商品別月末評価額がトランザクション内で自動生成されないこと

---

### 24.10 レスポンス契約

- JSONフィールド名がcamelCaseであること
- `data`がobjectで返却されること
- 月末資産状況IDが文字列で返却されること
- `targetYearMonth`が`YYYY-MM`形式で返却されること
- `confirmed`がbooleanで返却されること
- 作成直後の`confirmed`が`false`で返却されること
- `userId`がレスポンスへ含まれないこと
- `createdAt`がレスポンスへ含まれないこと
- `updatedAt`がレスポンスへ含まれないこと
- 月末資産残高がレスポンスへ含まれないこと
- 商品別月末評価額がレスポンスへ含まれないこと
- DB内部のsnake_caseのカラム名がそのまま公開されないこと

---

### 24.11 副作用

- 月末資産状況以外のテーブルへレコードが登録されないこと
- 既存の月末資産状況が更新されないこと
- 資産口座が更新されないこと
- 保有商品が更新されないこと
- 月末資産残高が登録・更新されないこと
- 商品別月末評価額が登録・更新されないこと
- 利用可能資産設定が更新されないこと

---

### 24.12 エラー時

- `USER_CONTEXT_REQUIRED`時に月末資産状況が作成されないこと
- `INVALID_USER_ID`時に月末資産状況が作成されないこと
- `USER_NOT_FOUND`時に月末資産状況が作成されないこと
- `VALIDATION_ERROR`時に月末資産状況が作成されないこと
- `MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`時に月末資産状況が追加されないこと
- `INTERNAL_SERVER_ERROR`時に不完全なレコードが残らないこと

---

### 24.13 異常系

- 想定外の例外で`500 Internal Server Error`となること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログに同じリクエストIDが記録されること
- SQLがレスポンスへ含まれないこと
- PostgreSQLの制約名がレスポンスへ含まれないこと
- スタックトレースがレスポンスへ含まれないこと
- 内部例外メッセージがレスポンスへ含まれないこと

---

## 25. Laravel実装方針

### 25.1 Action

HTTPリクエストを受け付け、
入力値および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの対象年月を受け取り、
月末資産状況作成UseCaseを呼び出す。

UseCaseから受け取った作成結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 入力値の単項目バリデーション
- 利用者境界の判定
- 同一対象年月の重複確認
- 月末資産状況の作成
- トランザクション制御
- レスポンス生成処理

---

### 25.2 UseCase

月末資産状況作成の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 対象年月を受け取る
- 同一利用者・同一対象年月の月末資産状況が存在しないことを確認する
- 新しい月末資産状況を作成する
- 作成結果を返却する

すでに同一対象年月の
月末資産状況が存在する場合は、
`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`
として扱う。

作成時の確定状態は、
必ず未確定とする。

```text
confirmed = false
```
本UseCaseでは、以下の処理は行わない。

- 月末資産残高の登録
- 商品別月末評価額の登録
- 確定条件の判定
- 月末資産状況の確定
- 月末資産状況の確定解除

---

### 25.3 Form Request / DTO

入力値の形式および単項目バリデーションを担当する。

検証対象は、`targetYearMonth`とする。

Form Requestでは、以下を検証する。

- 必須であること
- `null`でないこと
- 文字列であること
- `YYYY-MM`形式であること
- 実在する年月であること
- APIで許容する年月範囲内であること
- 未定義項目が含まれていないこと
- サーバー管理項目が含まれていないこと

以下の項目は、クライアントから受け付けない。

- `id`
- `userId`
- `confirmed`
- `confirmedAt`
- `createdAt`
- `updatedAt`

同一対象年月の重複確認は、データベース状態に依存する業務ルールであるため、Form Requestでは行わない。

検証済みの入力値は、入力用DTOへ変換してUseCaseへ渡す。

DTOの例：

```php
final readonly class CreateMonthEndAssetSnapshotInput
{
    public function __construct(
        public string $targetYearMonth,
    ) {
    }
}
```

---

### 25.4 Query

同一利用者・同一対象年月の月末資産状況が存在するか確認する。

検索条件には、必ず操作対象利用者IDを含める。

```php
$exists = MonthEndAssetSnapshot::query()
    ->where('user_id', $userId)
    ->where(
        'target_year_month',
        $input->targetYearMonth,
    )
    ->exists();
```

存在する場合は、`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`として扱う。

本Queryでは、月末資産残高および商品別月末評価額を参照しない。

---

### 25.5 Repository

月末資産状況の作成を担当する。

登録対象は、以下とする。

- `user_id`
- `target_year_month`
- `confirmed`

登録時には、以下の値を設定する。

```text
user_id           = 操作対象利用者ID
target_year_month = 指定された対象年月
confirmed         = false
```

登録例：

```php
return MonthEndAssetSnapshot::create([
    'user_id' => $userId,
    'target_year_month' => $input->targetYearMonth,
    'confirmed' => false,
]);
```

`id`、`created_at`および`updated_at`は、Laravelおよびデータベース側で設定する。

本Repositoryでは、以下の処理は行わない。

- 月末資産残高の作成
- 商品別月末評価額の作成
- 確定処理
- 確定解除処理

---

### 25.6 トランザクション

以下の処理を、1つのデータベーストランザクション内で実行する。

- 同一利用者・同一対象年月の重複確認
- 月末資産状況の作成

実装例：

```php
$snapshot = DB::transaction(
    function () use (
        $userId,
        $input,
    ): MonthEndAssetSnapshot {
        if (
            $this->query
                ->existsByUserAndTargetYearMonth(
                    $userId,
                    $input->targetYearMonth,
                )
        ) {
            throw new
                MonthEndAssetSnapshotAlreadyExistsException();
        }

        return $this->repository->create(
            $userId,
            $input,
        );
    },
);
```

処理途中で例外が発生した場合は、作成処理をロールバックする。

アプリケーション側の事前重複確認だけでは、同時実行時の重複を完全には防止できない。

そのため、データベースのUNIQUE制約を最終的な整合性保証として使用する。

---

### 25.7 UNIQUE制約違反の扱い

同一利用者・同一対象年月について、以下の組み合わせにUNIQUE制約を設定する。

```text
user_id
target_year_month
```

複数のリクエストが同時に実行され、アプリケーション側の重複確認を同時に通過した場合でも、データベース制約によって1件のみ登録されることを保証する。

PostgreSQLのUNIQUE制約違反が発生した場合は、内部例外をそのまま公開せず、`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`へ変換する。

PostgreSQLのエラーコード、制約名およびSQLはレスポンスへ含めない。

---

### 25.8 Responder

UseCaseから受け取った作成結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`201 Created`とともに作成した月末資産状況を`data`オブジェクトで返却する。

以下の場合は、共通エラーレスポンス形式へ変換する。

- 入力値が不正である
- 同一対象年月がすでに存在する
- 利用者が存在しない
- 想定外の例外が発生した

Responderは、以下の処理を行わない。

- 重複確認
- データベース登録
- 確定状態の決定
- 確定可否判定

---

### 25.9 API Resource

データベースカラムを直接返却せず、API Resourceを利用してAPIレスポンス形式へ変換する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'targetYearMonth' => $this->target_year_month,
    'confirmed' => (bool) $this->confirmed,
];
```

作成直後のため、`confirmed`は必ず`false`となる。

以下の項目は、レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`
- 月末資産残高
- 商品別月末評価額

---

### 25.10 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 25.11 Eloquentモデル

`MonthEndAssetSnapshot`モデルは、`month_end_asset_snapshots`テーブルへ対応する。

登録対象として、以下の属性を使用する。

```text
user_id
target_year_month
confirmed
```

`confirmed`は、booleanとして扱えるようにcastを設定する。

```php
protected function casts(): array
{
    return [
        'confirmed' => 'boolean',
    ];
}
```

`target_year_month`は、通常の日付ではなく、`YYYY-MM`形式の年月を表す業務値として扱う。

Phase1では、Eloquentのdate castは設定しない。

---

### 25.12 Mass Assignment

Mass Assignmentを利用する場合は、クライアント入力をそのままModelへ渡さない。

以下のような実装は禁止する。

```php
MonthEndAssetSnapshot::create(
    $request->all(),
);
```

これにより、クライアントから以下のようなサーバー管理項目を意図せず変更されることを防止する。

- `user_id`
- `confirmed`

登録値は、検証済みDTOおよび利用者コンテキストから明示的に組み立てる。

---

### 25.13 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 同一対象年月重複 | `MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS` |
| 入力値不正 | `VALIDATION_ERROR` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

UNIQUE制約違反については、対象となる制約を判別し、`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`へ変換する。

SQL、スタックトレース、PostgreSQLの制約名および内部例外メッセージは、APIレスポンスへ含めない。

ログには、調査に必要な範囲で以下を記録する。

- 操作対象利用者ID
- 対象年月
- 独自エラーコード
- リクエストID

月末資産残高や商品別月末評価額など、本APIで扱わない業務データを不要にログへ出力しない。

---

## 26. React・TypeScriptでの利用

登録リクエスト型は、以下とする。

```ts
export type CreateMonthEndAssetSnapshotRequest = {
  targetYearMonth: string;
};
```

レスポンス型は、以下とする。

```ts
export type MonthEndAssetSnapshot = {
  id: string;
  targetYearMonth: string;
  confirmed: boolean;
};

export type CreateMonthEndAssetSnapshotResponse = {
  data: MonthEndAssetSnapshot;
};
```

API呼び出し例は、以下とする。

```ts
const response =
  await apiClient.post<CreateMonthEndAssetSnapshotResponse>(
    '/api/v1/month-end-asset-snapshots',
    {
      targetYearMonth: '2026-08',
    },
  );
```

登録成功後は、作成された月末資産状況を利用して、以下の画面または処理へ遷移できる。

- 月末資産状況詳細画面
- 月末資産残高登録画面
- 商品別月末評価額登録画面
- 月末資産状況一覧画面

---

### 26.1 targetYearMonthの扱い

`targetYearMonth`は、API共通方針に従い`YYYY-MM`形式で送信する。

```ts
const request: CreateMonthEndAssetSnapshotRequest = {
  targetYearMonth: '2026-08',
};
```

対象年月は、日付ではなく年月を表す業務値として扱う。

JavaScriptの`Date`オブジェクトへ不必要に変換せず、原則として文字列で保持する。

---

### 26.2 confirmedの扱い

`confirmed`は、レスポンスでのみ取得する。

登録リクエストでは、フロントエンドから送信しない。

月末資産状況作成直後は、必ず`false`で返却される。

```ts
if (!response.data.confirmed) {
  // 未確定の月末資産状況
}
```

フロントエンドから`confirmed = true`を指定して確定済みデータを直接作成してはならない。

確定処理は、SNP-004 月末資産状況確定APIを使用する。

---

### 26.3 登録後の画面制御

月末資産状況作成後は、返却された月末資産状況IDを利用して対象年月の資産入力画面へ遷移できる。

```ts
const snapshotId = response.data.id;
```

例えば、月末資産状況詳細画面へ遷移する。

```ts
navigate(
  `/month-end-assets/${snapshotId}`,
);
```

作成直後は、月末資産残高および商品別月末評価額がまだ登録されていない可能性がある。

そのため、作成成功だけを理由として確定済みの資産状況として表示しない。

---

### 26.4 二重送信防止

作成処理中は、登録ボタンを非活性化する。

```ts
const [isSubmitting, setIsSubmitting] =
  useState(false);
```

リクエスト送信開始時に`isSubmitting = true`とし、処理完了後またはエラー発生後に`false`へ戻す。

これにより、意図しない二重送信を防止する。

ただし、最終的な重複登録防止はバックエンドおよびデータベースのUNIQUE制約で保証する。

---

### 26.5 重複エラーの扱い

指定した対象年月の月末資産状況がすでに存在する場合は、`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`が返却される。

フロントエンドでは、対象年月入力欄または画面上部へ重複エラーを表示する。

表示例：

```text
指定した対象年月の月末資産状況は
すでに作成されています。
```

必要に応じて、既存の月末資産状況一覧または詳細画面への導線を表示してよい。

---

### 26.6 バリデーションエラーの扱い

`VALIDATION_ERROR`が返却された場合は、`error.details.field`を利用して対象項目へエラーメッセージを表示する。

対象となるフィールドは、以下とする。

- `targetYearMonth`

```ts
if (
  detail.field === 'targetYearMonth'
) {
  setFieldError(
    'targetYearMonth',
    detail.message,
  );
}
```

---

### 26.7 エラー表示

エラーコードごとの基本的な扱いは、以下とする。

| エラーコード | フロントエンドの扱い |
| --- | --- |
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS` | 対象年月の重複エラーを表示する |
| `VALIDATION_ERROR` | 入力項目ごとにエラーを表示する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

### 26.8 一覧再取得

月末資産状況の作成成功後に一覧画面へ戻る場合は、月末資産状況一覧を再取得する。

React Query等を使用する場合は、月末資産状況一覧のクエリをinvalidateしてよい。

これにより、作成した対象年月を一覧へ反映する。

---

## 27. 設計上の補足

### 27.1 月末資産状況を親リソースとして先に作成する理由

月末資産状況は、対象年月に属する月末資産残高および商品別月末評価額をまとめる単位である。

そのため、各残高を登録する前に対象年月の月末資産状況を作成する。

これにより、以下のデータを同じ対象年月単位で管理できる。

- 月末資産残高
- 商品別月末評価額
- 確定状態

---

### 27.2 作成時に残高を自動登録しない理由

月末資産状況作成時点では、各資産口座および保有商品の実際の月末金額は確定していない。

そのため、月末資産状況作成と残高登録を分離する。

これにより、利用者は作成後に必要な残高情報を順次入力できる。

---

### 27.3 作成時に確定条件を判定しない理由

本APIの責務は、対象年月の月末資産状況を作成することである。

確定条件の判定まで行うと、作成と確定の責務が混在する。

そのため、確定条件の検証はSNP-004 月末資産状況確定APIへ委譲する。

---

### 27.4 作成直後を未確定とする理由

月末資産状況作成直後は、月末資産残高または商品別月末評価額が未登録である可能性がある。

正式な資産状況として扱える状態とは限らないため、必ず未確定状態で作成する。

```text
confirmed = false
```

---

### 27.5 対象年月を飛ばして作成可能とする理由

月末資産状況の作成順序と確定順序は別の業務ルールとして扱う。

利用者が将来月の入力準備を行う場合や、過去月を後から入力する場合にも月末資産状況を作成できるようにする。

そのため、対象年月が連続しているかどうかは作成時には制限しない。

未確定月を飛ばして確定できるかどうかは、SNP-004で判定する。

---

### 27.6 同一対象年月を1件に限定する理由

同一利用者について、同じ対象年月に複数の月末資産状況が存在すると、どのデータが正式な対象なのか判断できなくなる。

そのため、以下の組み合わせを一意とする。

```text
user_id
target_year_month
```

アプリケーション側の確認だけでなく、データベースのUNIQUE制約でも保証する。

---

### 27.7 POSTを採用する理由

本APIは、新しい月末資産状況リソースを作成する。

そのため、HTTPメソッドには`POST`を採用する。

```http
POST /api/v1/month-end-asset-snapshots
```

---

### 27.8 201 Createdを返却する理由

本APIの正常終了時には、新しい月末資産状況リソースが作成される。

そのため、`200 OK`ではなく`201 Created`を返却する。

---

### 27.9 既存データを返却しない理由

同一対象年月の月末資産状況がすでに存在する場合でも、既存データを正常レスポンスとして返却しない。

本APIは新規作成APIであり、既存リソースの取得を兼ねないためである。

重複時は、`409 Conflict`および`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`を返却する。

既存データを確認する場合は、一覧取得APIまたは詳細取得APIを使用する。

---

### 27.10 Idempotency-Keyを採用しない理由

本APIはPOSTであり、HTTP仕様上は冪等ではない。

Phase1では、個人利用を前提としており、フロントエンドの二重送信防止で意図しない連続送信を抑止する。

さらに、同一利用者・同一対象年月についてはデータベースのUNIQUE制約によって重複レコードを防止できる。

そのため、Phase1では`Idempotency-Key`を採用しない。

---

### 27.11 利用者IDをリクエストで受け付けない理由

作成先となる利用者は、利用者コンテキストによって決定する。

リクエストボディから`userId`を指定できるようにすると、他利用者のデータを作成できる可能性がある。

そのため、利用者IDは`X-User-Id`から取得する。

---

### 27.12 confirmedをリクエストで受け付けない理由

月末資産状況の確定は、残高登録状況や対象年月の確定順など、複数の業務ルールを満たす必要がある。

クライアントから`confirmed = true`を指定できるようにすると、これらの業務ルールを迂回できる。

そのため、作成APIでは`confirmed`を受け付けない。

---

### 27.13 confirmedAtをリクエストで受け付けない理由

確定日時は、実際に確定処理が成功した時点でサーバー側が設定すべき情報である。

月末資産状況作成時点では未確定であるため、`confirmedAt`を指定できない。

確定日時の設定は、SNP-004 月末資産状況確定APIで行う。

---

### 27.14 キャッシュを採用しない理由

月末資産状況の作成は、頻繁に繰り返される処理ではない。

また、作成後は一覧や詳細の状態が変化するため、キャッシュ無効化の制御が必要になる。

Phase1では、実装複雑性に対する効果が小さいため、アプリケーションキャッシュは採用しない。

---

## 28. 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)