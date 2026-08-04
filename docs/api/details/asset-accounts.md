# 資産口座API詳細設計

## 1. 概要

本書では、
資産口座APIに関する詳細仕様を定義する。

資産口座APIでは、
操作対象となるデモ利用者に帰属する資産口座の
取得、登録、更新および利用可能資産設定の管理を行う。

---

## 2. 対象API

| API ID | API名 | HTTPメソッド | URL |
|---|---|---|---|
| ACC-001 | 資産口座一覧取得 | GET | `/api/v1/asset-accounts` |
| ACC-002 | 資産口座登録 | POST | `/api/v1/asset-accounts` |
| ACC-003 | 資産口座詳細取得 | GET | `/api/v1/asset-accounts/{assetAccountId}` |
| ACC-004 | 資産口座更新 | PATCH | `/api/v1/asset-accounts/{assetAccountId}` |
| ACC-005 | 利用可能資産設定履歴取得 | GET | `/api/v1/asset-accounts/{assetAccountId}/available-settings` |
| ACC-006 | 利用可能資産設定登録 | POST | `/api/v1/asset-accounts/{assetAccountId}/available-settings` |

---

# ACC-001 資産口座一覧取得

## 1. 概要

操作対象となるデモ利用者に帰属する
資産口座の一覧を取得する。

利用中の資産口座だけでなく、
無効化された資産口座も取得対象とする。

資産口座ごとに、
現在の利用状態および利用可能資産区分を返却する。

---

## 2. ユースケース

利用者は、
登録済みの資産口座を一覧で確認する。

一覧では、
以下の情報を確認できる。

- 資産口座名
- 資産種別
- 残高記録単位
- 利用可能資産区分
- 利用開始年月
- 利用状態

無効化された資産口座も表示し、
利用中の資産口座と区別できるようにする。

---

## 3. エンドポイント

```http
GET /api/v1/asset-accounts
```

---

## 4. HTTPメソッド

```http
GET
```

本APIは、
資産口座を参照するのみであり、
データの登録、更新および無効化は行わない。

---

## 5. 認証・利用者の扱い

Phase1では、
認証機能を実装しない。

操作対象となるデモ利用者は、
`X-Demo-User-Id`リクエストヘッダーで指定する。

```http
X-Demo-User-Id: 1
```

指定された利用者に帰属する
資産口座のみを取得する。

他の利用者に帰属する資産口座は、
レスポンスへ含めない。

---

## 6. パスパラメータ

なし。

---

## 7. クエリパラメータ

なし。

Phase1では、
以下の機能を提供しない。

- 資産口座名による検索
- 資産種別による絞り込み
- 利用状態による絞り込み
- 利用可能資産区分による絞り込み
- 並び順の指定
- ページネーション

資産口座数が少ないことを前提とし、
操作対象利用者に帰属する資産口座を全件取得する。

---

## 8. リクエストヘッダー

### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-Demo-User-Id` | ○ | 操作対象となるデモ利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/asset-accounts
Accept: application/json
X-Demo-User-Id: 1
```

### 8.2 X-Demo-User-Id

`X-Demo-User-Id`は、
API共通方針に従って検証する。

本APIの処理開始前に、
共通ミドルウェアで以下を確認する。

- ヘッダーが指定されていること
- IDが共通方針で定めた形式であること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

---

## 9. リクエストボディ

なし。

---

## 10. バリデーション

### 10.1 リクエストヘッダー

以下を検証する。

- `X-Demo-User-Id`が指定されていること
- `X-Demo-User-Id`がIDの共通形式に一致すること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

### 10.2 クエリパラメータ

クエリパラメータを受け付けないため、
項目単位のバリデーションは行わない。

---

## 11. 業務ルール

- 指定された利用者に帰属する資産口座のみ取得する。
- 他の利用者に帰属する資産口座は取得しない。
- 利用中および無効化済みの資産口座を取得する。
- 無効化済みの資産口座は、利用中の資産口座と識別できる形式で返却する。
- 利用中の資産口座を先に表示する。
- 同じ利用状態の資産口座は、資産口座名の昇順で表示する。
- 利用可能資産区分は、API実行時の対象年月に有効な設定から取得する。
- 対象年月は、日本標準時におけるAPI実行時点の年月とする。
- 適用可能な利用可能資産設定が存在しない場合は、データ不整合として扱う。
- データベースの内部カラムをそのままレスポンスへ公開しない。

---

## 12. 利用可能資産区分の取得

資産口座の利用可能資産区分は、
`asset_account_available_settings`から取得する。

API実行時の対象年月について、
以下の条件を満たす設定を有効な設定として扱う。

```text
startYearMonth <= API実行時の対象年月
かつ
endYearMonthがNULL
または
API実行時の対象年月 <= endYearMonth
```

同一資産口座について、
有効な設定が複数件取得された場合は、
期間重複によるデータ不整合として扱う。

利用可能資産設定の期間重複は、
登録処理およびデータベース制約によって防止する。

---

## 13. 表示順

資産口座は、
以下の優先順位で並び替える。

1. 利用中
2. 無効化済み
3. 資産口座名の昇順

利用状態は、
`asset_accounts.deleted_at`から判定する。

```text
deleted_atがNULL
    → 利用中

deleted_atがNULLではない
    → 無効化済み
```

資産口座名の比較方法は、
PostgreSQLおよびLaravelの実装方針に従う。

Phase1では、
利用者が任意の並び順を指定する機能は提供しない。

---

## 14. 処理フロー

```text
リクエスト受付
    ↓
リクエストID生成
    ↓
X-Demo-User-Id検証
    ↓
操作対象利用者の特定
    ↓
利用者に帰属する資産口座を取得
    ↓
API実行時の対象年月に有効な
利用可能資産設定を取得
    ↓
利用中・無効化済み・資産口座名の順で並び替え
    ↓
APIレスポンス用の形式へ変換
    ↓
正常レスポンス返却
```

---

## 15. トランザクション境界

参照処理のみであるため、
明示的なデータベーストランザクションは使用しない。

資産口座と利用可能資産設定は、
1回のAPI処理内で取得する。

---

## 16. 排他制御

参照処理のみであるため、
排他制御は行わない。

API実行中に資産口座または利用可能資産設定が更新された場合、
データベースから取得できた時点の情報を返却する。

Phase1では、
一覧取得時点の厳密なスナップショット整合性は保証しない。

---

## 17. 成功レスポンス

### 17.1 HTTPステータス

```http
200 OK
```

### 17.2 レスポンスボディ

```json
{
  "data": [
    {
      "id": "1",
      "name": "普通預金",
      "assetType": "BANK",
      "balanceRecordingUnit": "ACCOUNT",
      "isAvailable": true,
      "startYearMonth": "2026-01",
      "isEnabled": true
    },
    {
      "id": "2",
      "name": "証券口座",
      "assetType": "SECURITIES",
      "balanceRecordingUnit": "HOLDING",
      "isAvailable": true,
      "startYearMonth": "2026-01",
      "isEnabled": true
    },
    {
      "id": "3",
      "name": "旧普通預金",
      "assetType": "BANK",
      "balanceRecordingUnit": "ACCOUNT",
      "isAvailable": false,
      "startYearMonth": "2025-01",
      "isEnabled": false
    }
  ]
}
```

---

## 18. レスポンス項目

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `data` | array | × | 資産口座一覧 |
| `data[].id` | string | × | 資産口座ID |
| `data[].name` | string | × | 資産口座名 |
| `data[].assetType` | string | × | 資産種別 |
| `data[].balanceRecordingUnit` | string | × | 残高記録単位 |
| `data[].isAvailable` | boolean | × | API実行時の対象年月における利用可能資産区分 |
| `data[].startYearMonth` | string | × | 利用開始年月。`YYYY-MM`形式 |
| `data[].isEnabled` | boolean | × | 現在の利用状態 |

### 18.1 id

資産口座IDは、
API共通方針に従い文字列で返却する。

```json
{
  "id": "1"
}
```

データベースでは`bigint`として保持するが、
APIレスポンスでは文字列へ変換する。

### 18.2 assetType

資産種別は、
APIで定義した列挙値を文字列で返却する。

想定する値は、
以下のとおりとする。

| 値 | 意味 |
|---|---|
| `CASH` | 現金 |
| `BANK` | 銀行 |
| `SECURITIES` | 証券 |
| `IDECO` | iDeCo |
| `CORPORATE_DC` | 企業型DC |
| `OTHER` | その他 |

列挙値の最終的な定義は、
資産口座登録APIの設計時に確定する。

### 18.3 balanceRecordingUnit

残高記録単位は、
以下の文字列で返却する。

| 値 | 意味 |
|---|---|
| `ACCOUNT` | 口座単位 |
| `HOLDING` | 商品単位 |

データベースで数値として保持する場合も、
APIでは意味が読み取れる文字列へ変換する。

### 18.4 isAvailable

`isAvailable`は、
API実行時の対象年月に有効な
利用可能資産設定から取得する。

```json
{
  "isAvailable": true
}
```

過去の利用可能資産設定履歴は、
本APIでは返却しない。

設定履歴は、
ACC-005 利用可能資産設定履歴取得APIで返却する。

### 18.5 startYearMonth

利用開始年月は、
`YYYY-MM`形式で返却する。

```json
{
  "startYearMonth": "2026-01"
}
```

### 18.6 isEnabled

利用状態は、
booleanで返却する。

```text
true
    → 利用中

false
    → 無効化済み
```

データベースの`deleted_at`は、
APIレスポンスへ含めない。

---

## 19. データが存在しない場合

操作対象利用者に資産口座が登録されていない場合も、
API処理自体は正常終了とする。

```http
200 OK
```

```json
{
  "data": []
}
```

一覧の検索結果が0件であるため、
`404 Not Found`は返却しない。

---

## 20. エラーレスポンス

異常時は、
API共通方針で定めた
共通エラーレスポンス形式を使用する。

例：

```json
{
  "error": {
    "code": "DEMO_USER_CONTEXT_REQUIRED",
    "message": "操作対象の利用者が指定されていません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

## 21. HTTPステータス

| HTTPステータス | 条件 |
|---:|---|
| `200 OK` | 資産口座一覧の取得に成功した |
| `400 Bad Request` | 利用者IDの指定がない、または形式が不正である |
| `404 Not Found` | 指定されたデモ利用者が存在しない |
| `500 Internal Server Error` | 想定外のサーバーエラー、またはデータ不整合が発生した |

資産口座が0件の場合は、
`404 Not Found`ではなく`200 OK`を返却する。

---

## 22. エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `DEMO_USER_CONTEXT_REQUIRED` | 400 | `X-Demo-User-Id`が指定されていない | × |
| `INVALID_DEMO_USER_ID` | 400 | `X-Demo-User-Id`の形式が不正である | × |
| `DEMO_USER_NOT_FOUND` | 404 | 指定されたデモ利用者が存在しない | × |
| `AVAILABLE_SETTING_NOT_FOUND` | 500 | 対象年月に有効な利用可能資産設定が存在しない | × |
| `AVAILABLE_SETTING_PERIOD_CONFLICT` | 500 | 対象年月に複数の利用可能資産設定が適用されている | × |
| `INTERNAL_SERVER_ERROR` | 500 | 想定外のサーバーエラーが発生した | ○ |

エラーコードの正式な定義は、
[エラーコード一覧](../error-codes.md)に従う。

---

## 23. 冪等性

本APIは参照処理であり、
同一条件で複数回実行してもデータを変更しない。

資産口座および利用可能資産設定に変更がない限り、
同一の結果を返却する。

---

## 24. キャッシュ

Phase1では、
HTTPキャッシュ制御およびアプリケーションキャッシュを導入しない。

資産口座数が少なく、
キャッシュを導入するほどの性能上の効果が見込まれないためである。

将来、
アクセス頻度やデータ量が増えた場合に再検討する。

---

## 25. 関連テーブル

### 25.1 asset_accounts

資産口座の基本情報を取得する。

使用する主なカラムは、
以下とする。

| カラム | 用途 |
|---|---|
| `id` | 資産口座ID |
| `user_id` | 利用者境界 |
| `name` | 資産口座名 |
| `asset_type` | 資産種別 |
| `balance_recording_unit` | 残高記録単位 |
| `start_year_month` | 利用開始年月 |
| `deleted_at` | 利用状態 |

### 25.2 asset_account_available_settings

API実行時の対象年月に有効な
利用可能資産区分を取得する。

使用する主なカラムは、
以下とする。

| カラム | 用途 |
|---|---|
| `asset_account_id` | 資産口座との関連 |
| `start_year_month` | 適用開始年月 |
| `end_year_month` | 適用終了年月 |
| `is_available` | 利用可能資産区分 |

---

## 26. 関連する機能要件

- `4.1 概要`
  - 資産口座は月末資産管理および目的達成判定の基礎情報となる
- `4.6 利用可能資産区分`
  - 目的達成判定に利用する資産かどうかを表す
- `4.7 利用可能資産区分の変更`
  - 対象年月時点で有効な設定を使用する
- `4.8 無効化`
  - 無効化後も過去の履歴を保持する
- `4.9 一覧表示`
  - 一覧で表示する項目
- `4.10 表示順`
  - 利用中、利用終了、資産口座名の順で表示する

---

## 27. テスト観点

### 27.1 正常系

- 操作対象利用者の資産口座を取得できること
- 複数の資産口座を取得できること
- 利用中の資産口座が先に返却されること
- 無効化済みの資産口座が一覧に含まれること
- 同じ利用状態では資産口座名の昇順で返却されること
- API実行時の対象年月に有効な利用可能資産設定が返却されること
- 残高記録単位がAPI用の文字列へ変換されること
- IDが文字列で返却されること

### 27.2 利用者境界

- 指定された利用者に帰属する資産口座のみ返却されること
- 他の利用者に帰属する資産口座が含まれないこと
- 他の利用者に同名の資産口座が存在しても影響しないこと

### 27.3 データなし

- 資産口座が0件の場合に`200 OK`となること
- 資産口座が0件の場合に`data: []`となること

### 27.4 無効化

- `deleted_at`が設定された資産口座も取得されること
- 無効化済み資産口座の`isEnabled`が`false`となること
- 利用中資産口座の`isEnabled`が`true`となること

### 27.5 利用可能資産設定

- 対象年月が適用開始年月と同じ場合に設定が適用されること
- 対象年月が適用終了年月と同じ場合に設定が適用されること
- 適用終了年月がNULLの設定を取得できること
- 対象年月に有効な設定がない場合にエラーとなること
- 対象年月に複数の設定が適用される場合にエラーとなること

### 27.6 ヘッダー

- `X-Demo-User-Id`が指定されていない場合にエラーとなること
- `X-Demo-User-Id`の形式が不正な場合にエラーとなること
- 存在しない利用者IDの場合にエラーとなること
- 論理削除済み利用者IDの場合にエラーとなること

### 27.7 レスポンス契約

- 共通レスポンス形式に従っていること
- JSONフィールド名がcamelCaseであること
- `deletedAt`がレスポンスへ含まれないこと
- `userId`がレスポンスへ含まれないこと
- データベースの数値コードがそのまま公開されないこと
- `data`が配列として返却されること

### 27.8 異常系

- 想定外の例外発生時に`500 Internal Server Error`となること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログに同じリクエストIDが記録されること
- SQLやスタックトレースなどの内部情報がレスポンスへ含まれないこと

---

## 28. Laravel実装方針

### 28.1 Action

HTTPリクエストを受け付け、
入力値および利用者コンテキストを取得する。

必要なUseCaseを呼び出し、
処理結果をResponderへ渡す。

業務ルール、
検索条件、
並び替え、
利用可能資産区分の判定および
レスポンス生成処理は、
Actionへ直接記述しない。

### 28.2 UseCase

資産口座一覧取得の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者の資産口座を取得する
- API実行時点で有効な利用可能資産設定を取得する
- 利用中・無効化済み・資産口座名の順で並び替える
- Responderへ返却するデータを生成する

### 28.3 Query

操作対象利用者に帰属する
資産口座一覧を取得する。

利用可能資産設定は、
API実行時点で有効な設定のみ取得する。

無効化済み資産口座も一覧表示するため、
Laravel SoftDeletesを使用する場合は、
`withTrashed()`を利用する。

取得例：

```php
AssetAccount::query()
    ->withTrashed()
    ->where('user_id', $demoUserId)
    ->with('availableSettings')
    ->orderByRaw('deleted_at IS NULL DESC')
    ->orderBy('name')
    ->get();
```

実際の実装では、
対象年月時点で有効な利用可能資産設定のみ取得し、
不要な履歴は読み込まない。

### 28.4 Responder

UseCaseから受け取った処理結果を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`200 OK`とともに
資産口座一覧を`data`配列で返却する。

資産口座が存在しない場合も、
`200 OK`と空配列を返却する。

```json
{
  "data": []
}
```

エラー発生時は、
共通エラーレスポンス形式に従って返却する。

### 28.5 API Resource

データベースカラムを直接返却せず、
API Resourceを利用して
APIレスポンス形式へ変換する。

資産口座IDは、
API共通方針に従い
文字列として返却する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'name' => $this->name,
    'assetType' => $this->assetType,
    'balanceRecordingUnit' => $this->balanceRecordingUnit,
    'isAvailable' => $this->isAvailable,
    'startYearMonth' => $this->startYearMonth,
    'isEnabled' => $this->deleted_at === null,
];
```

API Resourceは、
Responderから利用する。

### 28.6 Middleware

以下の共通ミドルウェアを適用する。

- デモ利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

デモ利用者コンテキストは、
`X-Demo-User-Id`を検証し、
操作対象利用者を特定する。

存在しない利用者、
または論理削除済み利用者が指定された場合は、
処理を終了し、
共通エラーレスポンスを返却する。

---

## 29. React・TypeScriptでの利用

レスポンス型の例は、
以下とする。

```ts
export type AssetType =
  | 'CASH'
  | 'BANK'
  | 'SECURITIES'
  | 'IDECO'
  | 'CORPORATE_DC'
  | 'OTHER';

export type BalanceRecordingUnit =
  | 'ACCOUNT'
  | 'HOLDING';

export type AssetAccountListItem = {
  id: string;
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  isAvailable: boolean;
  startYearMonth: string;
  isEnabled: boolean;
};

export type AssetAccountListResponse = {
  data: AssetAccountListItem[];
};
```

フロントエンドは、
`isEnabled`を利用して
利用中と無効化済みの表示を切り替える。

`isAvailable`は、
目的達成判定へ利用する資産かどうかを示す表示に利用する。

---

## 30. 設計上の補足

### 30.1 無効化済み資産口座を返却する理由

機能要件では、
資産口座一覧に利用状態を表示し、
利用中の資産口座と利用終了した資産口座を
分けて表示することを求めている。

そのため、
通常のSoftDeletesによる検索結果だけではなく、
無効化済み資産口座も取得対象とする。

### 30.2 利用可能資産設定の返却範囲

本APIでは、
API実行時の対象年月に有効な
利用可能資産区分のみ返却する。

設定履歴全体は、
ACC-005 利用可能資産設定履歴取得APIで返却する。

### 30.3 ページネーションを採用しない理由

Phase1では、
1利用者が登録する資産口座数は少数であることを前提とする。

ページネーションを導入すると、
画面表示およびAPI利用が複雑になる一方で、
性能上の効果が小さいため採用しない。

### 30.4 API実行時の対象年月

利用可能資産区分を判定する対象年月は、
日本標準時におけるAPI実行時の年月とする。

将来、
過去または将来の対象年月時点における
資産口座一覧が必要になった場合は、
対象年月をクエリパラメータとして指定する方式を検討する。

---

## 31. 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)