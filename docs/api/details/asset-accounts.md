# 資産口座API詳細設計

## 1. 概要

本書では、
資産口座APIに関する詳細仕様を定義する。

資産口座APIでは、
操作対象となる利用者に帰属する資産口座の
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

操作対象となる利用者に帰属する
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

## 5. 利用者コンテキスト

本APIは利用者依存APIのため、
`X-User-Id`を必須とする。

利用者コンテキストの特定、
`X-User-Id`の検証および
利用者境界については、
[API共通方針](../api-common-policy.md)に従う。

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

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json` |

`X-User-Id`の検証および利用者境界は、
API共通方針に従う。

---

## 9. リクエストボディ

なし。

---

## 10. バリデーション

### 10.1 リクエストヘッダー

以下を検証する。

- `X-User-Id`が指定されていること
- `X-User-Id`がIDの共通形式に一致すること
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
X-User-Id検証
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
    "code": "USER_CONTEXT_REQUIRED",
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
| `404 Not Found` | 指定された利用者が存在しない |
| `500 Internal Server Error` | 想定外のサーバーエラー、またはデータ不整合が発生した |

資産口座が0件の場合は、
`404 Not Found`ではなく`200 OK`を返却する。

---

## 22. エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `USER_CONTEXT_REQUIRED` | 400 | `X-User-Id`が指定されていない | × |
| `INVALID_USER_ID` | 400 | `X-User-Id`の形式が不正である | × |
| `USER_NOT_FOUND` | 404 | 指定された利用者が存在しない | × |
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

- `X-User-Id`が指定されていない場合にエラーとなること
- `X-User-Id`の形式が不正な場合にエラーとなること
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
    ->where('user_id', $userId)
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

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキストは、
`X-User-Id`を検証し、
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

---

# ACC-002 資産口座登録

## 1. 概要

操作対象となる利用者に帰属する資産口座を新規登録する。

資産口座登録時には、以下の基本情報を登録する。

* 資産口座名
* 資産種別
* 残高記録単位
* 利用開始年月

また、資産口座登録時点の利用可能資産区分もあわせて登録する。

そのため、ACC-002では`asset_accounts`だけでなく、初期の利用可能資産設定として

```text
asset_account_available_settings
```

にも1件登録する。

概念的には、以下の処理となる。

```text
資産口座登録
    ↓
asset_accounts
1件登録
    ↓
初期利用可能資産設定
1件登録
```

両方の登録は同一処理として扱い、途中で一方のみが登録された状態を残さない。

---

## 2. ユースケース

利用者は、新しく管理したい資産口座をLife Plannerへ登録する。

例えば、以下のような資産口座を登録する。

* 現金
* 銀行口座
* 証券口座
* iDeCo
* 企業型DC
* その他の資産口座

登録時には、その資産口座を

```text
口座単位
```

で残高管理するか、

```text
商品単位
```

で管理するかを指定する。

また、目的達成判定に利用する資産として扱うかどうかを、初期の利用可能資産区分として指定する。

登録後は、ACC-001 資産口座一覧取得やACC-003 資産口座詳細取得で登録内容を確認できる。

---

## 3. エンドポイント

```http
POST /api/v1/asset-accounts
```

---

## 4. HTTPメソッド

```text
POST
```

本APIは、新しい資産口座を作成するため、`POST`を使用する。

正常終了時には、

```text
asset_accounts
```

へ新規レコードを登録する。

また、資産口座登録時点の初期利用可能資産設定として、

```text
asset_account_available_settings
```

にも新規レコードを登録する。

そのため、本APIは業務データに対する副作用を持つ。

---

## 5. 利用者コンテキスト

本APIは利用者依存APIのため、`X-User-Id`を必須とする。

操作対象となる利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者コンテキストの特定、`X-User-Id`の検証および利用者境界については、[API共通方針](../api-common-policy.md)に従う。

資産口座登録時には、サーバー側で特定した操作対象利用者IDを

```text
asset_accounts.user_id
```

へ設定する。

クライアントから`userId`または`user_id`をリクエストボディとして受け付けない。

概念的には、以下とする。

```text
X-User-Id
    ↓
利用者コンテキスト
    ↓
asset_accounts.user_id
```

これにより、登録先利用者をクライアント側から任意に差し替えられないようにする。

---

### 5.1 利用者境界

ACC-002で登録する資産口座は、必ず操作対象利用者に帰属させる。

以下のような他利用者IDを指定して資産口座を登録することはできない。

```json
{
  "userId": "2"
}
```

`userId`はACC-002のリクエスト項目として定義しない。

---

### 5.2 資産口座名の一意性と利用者境界

資産口座名の重複判定は、システム全体ではなく、操作対象利用者の範囲で行う。

概念的には、

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = 登録する資産口座名
```

によって重複を確認する。

例えば、

```text
User A
    証券口座

User B
    証券口座
```

のように、異なる利用者が同じ資産口座名を持つことは許可する。

一方、同一利用者内で

```text
証券口座
証券口座
```

のように同名の資産口座を複数登録することは許可しない。

論理削除済み資産口座を重複判定へ含めるかどうかは、資産口座の再登録ルールに従う。

---

### 5.3 初期利用可能資産設定の利用者境界

ACC-002では、資産口座登録と同時に初期の

```text
asset_account_available_settings
```

を登録する。

この設定には直接`user_id`を持たせず、登録した

```text
asset_accounts.id
```

を通して操作対象利用者に帰属させる。

概念的には、

```text
操作対象利用者
    ↓
asset_accounts.user_id
    ↓
asset_accounts.id
    ↓
asset_account_available_settings.asset_account_id
```

となる。

他利用者の資産口座IDを利用可能資産設定の登録先として使用してはならない。

---

### 5.4 X-User-Idが不正な場合

以下の場合は、資産口座登録処理へ進まない。

* `X-User-Id`が指定されていない
* `X-User-Id`の形式が不正
* 指定された利用者が存在しない
* 指定された利用者が論理削除済み

この場合、

```text
asset_accounts
asset_account_available_settings
```

のどちらにもレコードを登録しない。

利用者コンテキストに関するエラーコードおよびHTTPステータスは、API共通方針に従う。

---

## 6. パスパラメータ

なし。

本APIでは、新しい資産口座を登録するため、既存の資産口座IDをパスパラメータとして指定しない。

---

## 7. クエリパラメータ

なし。

Phase1では、資産口座登録時の動作を変更するためのクエリパラメータは提供しない。

登録する資産口座の情報は、リクエストボディで指定する。

---

## 8. リクエストヘッダー

| ヘッダー名          |  必須 | 説明                 |
| -------------- | :-: | ------------------ |
| `X-User-Id`    |  ○  | 操作対象となる利用者ID       |
| `Content-Type` |  ○  | `application/json` |
| `Accept`       |  ○  | `application/json` |

`X-User-Id`の検証および利用者境界は、API共通方針に従う。

リクエスト例：

```http
POST /api/v1/asset-accounts
Content-Type: application/json
Accept: application/json
X-User-Id: 1
```

---

## 9. リクエストボディ

資産口座の基本情報および登録時点の利用可能資産区分を指定する。

例：

```json
{
  "name": "証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

---

### 9.1 リクエスト項目

| 項目                     | 型       |  必須 | NULL | 説明                 |
| ---------------------- | ------- | :-: | :--: | ------------------ |
| `name`                 | string  |  ○  |   ×  | 資産口座名              |
| `assetType`            | string  |  ○  |   ×  | 資産種別               |
| `balanceRecordingUnit` | string  |  ○  |   ×  | 残高記録単位             |
| `startYearMonth`       | string  |  ○  |   ×  | 利用開始年月。`YYYY-MM`形式 |
| `isAvailable`          | boolean |  ○  |   ×  | 登録時点における利用可能資産区分   |

---

### 9.2 name

資産口座名を指定する。

例：

```json
{
  "name": "証券口座"
}
```

同一利用者内では、同じ資産口座名を重複して登録できない。

資産口座名の長さは、`asset_accounts.name`のテーブル定義に従う。

---

### 9.3 assetType

資産種別を指定する。

指定可能な値は、以下とする。

| 値              | 意味    |
| -------------- | ----- |
| `CASH`         | 現金    |
| `BANK`         | 銀行    |
| `SECURITIES`   | 証券    |
| `IDECO`        | iDeCo |
| `CORPORATE_DC` | 企業型DC |
| `OTHER`        | その他   |

リクエストでは、データベースで保持する数値コードを直接指定せず、APIで定義した文字列を使用する。

---

### 9.4 balanceRecordingUnit

月末資産をどの単位で記録するかを指定する。

指定可能な値は、以下とする。

| 値         | 意味   |
| --------- | ---- |
| `ACCOUNT` | 口座単位 |
| `HOLDING` | 商品単位 |

`ACCOUNT`の場合は、資産口座単位で月末残高を記録する。

`HOLDING`の場合は、その資産口座に属する保有商品単位で月末評価額を記録する。

---

### 9.5 startYearMonth

資産口座の利用開始年月を指定する。

形式は、

```text
YYYY-MM
```

とする。

例：

```json
{
  "startYearMonth": "2026-08"
}
```

日付単位ではなく、年月単位で管理する。

---

### 9.6 isAvailable

登録時点において、目的達成判定へ利用する資産として扱うかどうかを指定する。

```text
true
    → 利用可能資産として扱う

false
    → 利用可能資産として扱わない
```

例：

```json
{
  "isAvailable": true
}
```

`isAvailable`は、`asset_accounts`へ直接保持しない。

ACC-002では、初期利用可能資産設定として

```text
asset_account_available_settings
```

へ登録する。

初期設定の

```text
start_year_month
```

には、資産口座の

```text
asset_accounts.start_year_month
```

と同じ年月を設定する。

初期設定であるため、

```text
end_year_month
```

は`NULL`とする。

概念的には、

```text
asset_accounts.start_year_month
    = 2026-08

asset_account_available_settings.start_year_month
    = 2026-08

asset_account_available_settings.end_year_month
    = NULL

asset_account_available_settings.is_available
    = true
```

となる。

---

### 9.7 リクエストで受け付けない項目

以下の項目は、クライアントから受け付けない。

* `id`
* `userId`
* `user_id`
* `isEnabled`
* `deletedAt`
* `deleted_at`
* `createdAt`
* `created_at`
* `updatedAt`
* `updated_at`
* `assetAccountId`
* `asset_account_id`
* `endYearMonth`
* `end_year_month`

これらは、サーバー側で設定するか、ACC-002では変更対象としない。

特に、

```text
user_id
```

は、`X-User-Id`から特定した利用者コンテキストを使用してサーバー側で設定する。

また、

```text
asset_account_id
```

は、新規登録した`asset_accounts.id`を使用して初期利用可能資産設定へ設定する。

---

## 10. バリデーション

リクエストヘッダーおよびリクエストボディについて、以下の検証を行う。

---

### 10.1 リクエストヘッダー

以下を検証する。

* `X-User-Id`が指定されていること
* `X-User-Id`がIDの共通形式に一致すること
* 指定された利用者が存在すること
* 指定された利用者が論理削除されていないこと
* `Content-Type`が`application/json`であること
* `Accept`が`application/json`であること

利用者コンテキストに関する検証は、API共通方針に従う。

---

### 10.2 name

以下を検証する。

* 必須であること
* 文字列であること
* 空文字ではないこと
* テーブル定義で定めた最大文字数以内であること
* 同一利用者内で同名の資産口座が存在しないこと

概念的な重複判定は、以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = name
```

異なる利用者に同名の資産口座が存在することは許可する。

---

### 10.3 assetType

以下を検証する。

* 必須であること
* 文字列であること
* 定義済みの列挙値であること

指定可能な値は、以下とする。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

以下のような未定義値は受け付けない。

```json
{
  "assetType": "CRYPTO"
}
```

---

### 10.4 balanceRecordingUnit

以下を検証する。

* 必須であること
* 文字列であること
* 定義済みの列挙値であること

指定可能な値は、以下とする。

```text
ACCOUNT
HOLDING
```

以下のような未定義値は受け付けない。

```json
{
  "balanceRecordingUnit": "UNKNOWN"
}
```

---

### 10.5 startYearMonth

以下を検証する。

* 必須であること
* 文字列であること
* `YYYY-MM`形式であること
* 実在する年月であること

正常例：

```text
2026-01
2026-08
2027-12
```

異常例：

```text
2026-1
2026/08
202608
2026-00
2026-13
abcd-ef
```

Phase1では、年月単位で管理するため、日付を含む値は受け付けない。

---

### 10.6 isAvailable

以下を検証する。

* 必須であること
* booleanであること
* `NULL`ではないこと

正常例：

```json
{
  "isAvailable": true
}
```

```json
{
  "isAvailable": false
}
```

以下のような文字列や数値による指定は受け付けない。

```json
{
  "isAvailable": "true"
}
```

```json
{
  "isAvailable": 1
}
```

---

### 10.7 項目間バリデーション

ACC-002では、`balanceRecordingUnit`によって登録時に別の入力項目を追加要求しない。

例えば、

```text
balanceRecordingUnit = HOLDING
```

の場合でも、ACC-002で保有商品の同時登録は行わない。

資産口座登録後に、HLD-002 保有商品登録APIを使用して保有商品を登録する。

---

### 10.8 利用開始年月と初期利用可能資産設定

初期利用可能資産設定の適用開始年月は、クライアントから個別に指定させない。

必ず、

```text
asset_account_available_settings.start_year_month
    =
asset_accounts.start_year_month
```

とする。

これにより、資産口座の利用開始前から利用可能資産設定だけが存在する状態を防止する。

---

### 10.9 バリデーションエラー

リクエストボディの項目バリデーションに失敗した場合は、API共通方針で定めたバリデーションエラー形式を使用する。

バリデーションエラーとなった場合は、

```text
asset_accounts
asset_account_available_settings
```

のどちらにもレコードを登録しない。

---

## 11. 業務ルール

ACC-002では、操作対象利用者に帰属する資産口座と、その資産口座に対する初期利用可能資産設定を登録する。

---

### 11.1 操作対象利用者に帰属する資産口座として登録する

新規資産口座の

```text
asset_accounts.user_id
```

には、`X-User-Id`から特定した操作対象利用者IDを設定する。

クライアントから`user_id`を指定させない。

概念的には、

```text
X-User-Id
    ↓
利用者コンテキスト
    ↓
asset_accounts.user_id
```

とする。

---

### 11.2 同一利用者内で資産口座名を重複させない

同一利用者内では、同じ資産口座名を複数登録できない。

概念的には、

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = 登録する資産口座名
```

に該当する資産口座が存在する場合、登録不可とする。

異なる利用者間では、同じ資産口座名を登録できる。

例えば、

```text
User A
    └─ 証券口座

User B
    └─ 証券口座
```

は許可する。

---

### 11.3 論理削除済み資産口座との重複

資産口座一覧では、論理削除済み資産口座も履歴として保持する。

そのため、同一利用者に論理削除済みの同名資産口座が存在する場合も、同名の新規登録を許可しない。

これにより、同一利用者内で資産口座名による識別が曖昧になることを防止する。

重複判定では、Laravel SoftDeletesの通常スコープだけに依存せず、論理削除済みレコードも含めて確認する。

概念的には、

```text
同一user_id
+
同一name
+
利用中・論理削除済みの両方
    ↓
重複あり
```

とする。

---

### 11.4 資産種別

`assetType`には、APIで定義した資産種別のみ指定できる。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

APIで受け取った文字列は、Laravel側でデータベース保存形式へ変換する。

データベース内部の数値コードをクライアントへ意識させない。

---

### 11.5 残高記録単位

`balanceRecordingUnit`には、以下のいずれかを指定する。

```text
ACCOUNT
HOLDING
```

`ACCOUNT`の場合は、資産口座単位で月末残高を管理する。

`HOLDING`の場合は、保有商品単位で月末評価額を管理する。

ACC-002では、残高記録単位を登録するだけとし、月末残高または保有商品を同時登録しない。

---

### 11.6 商品単位の場合でも保有商品を自動登録しない

```text
balanceRecordingUnit = HOLDING
```

で資産口座を登録した場合でも、ACC-002では保有商品を登録しない。

資産口座登録後に、HLD-002 保有商品登録APIを使用して必要な保有商品を登録する。

概念的には、

```text
ACC-002
資産口座登録
    ↓
balanceRecordingUnit = HOLDING
    ↓
必要に応じて
HLD-002
保有商品登録
```

とする。

---

### 11.7 利用開始年月

`startYearMonth`は、資産口座をLife Planner上で管理し始める年月を表す。

年月単位で管理し、

```text
YYYY-MM
```

形式とする。

資産口座の利用開始年月より前の月末資産管理において、当該資産口座を記録対象として扱わない。

具体的な対象年月判定は、月末資産API側の業務ルールに従う。

---

### 11.8 登録時は利用中として扱う

ACC-002で登録した資産口座は、新規登録時点では利用中として扱う。

そのため、

```text
asset_accounts.deleted_at
```

は`NULL`とする。

クライアントから`isEnabled`または`deletedAt`を指定させない。

概念的には、

```text
新規登録
    ↓
deleted_at = NULL
    ↓
isEnabled = true
```

となる。

---

### 11.9 初期利用可能資産設定を必ず登録する

ACC-002では、資産口座だけを登録して利用可能資産設定が存在しない状態を作らない。

資産口座登録と同時に、

```text
asset_account_available_settings
```

へ初期設定を1件登録する。

これにより、ACC-001などで対象年月における利用可能資産区分を必ず判定できる状態とする。

---

### 11.10 初期利用可能資産設定の開始年月

初期利用可能資産設定の

```text
asset_account_available_settings.start_year_month
```

には、

```text
asset_accounts.start_year_month
```

と同じ年月を設定する。

概念例：

```text
asset_accounts
start_year_month = 2026-08

        ↓

asset_account_available_settings
start_year_month = 2026-08
```

クライアントから利用可能資産設定の開始年月を別途指定させない。

---

### 11.11 初期利用可能資産設定の終了年月

初期利用可能資産設定の

```text
asset_account_available_settings.end_year_month
```

は、

```text
NULL
```

とする。

登録時点では、利用可能資産区分の終了年月は決まっていないためである。

将来、利用可能資産区分を変更する場合は、ACC-006 利用可能資産設定登録APIで履歴を追加する。

---

### 11.12 isAvailable

リクエストの

```text
isAvailable
```

は、初期利用可能資産設定の

```text
asset_account_available_settings.is_available
```

へ保存する。

概念的には、

```text
Request
isAvailable = true

        ↓

asset_account_available_settings
is_available = true
```

とする。

`asset_accounts`へ`is_available`を保持しない。

---

### 11.13 利用可能資産区分を履歴として管理する

利用可能資産区分は、資産口座の固定属性としてではなく、対象年月によって変化する履歴情報として扱う。

そのため、

```text
asset_accounts
```

ではなく、

```text
asset_account_available_settings
```

で管理する。

ACC-002では、その履歴の最初の1件を資産口座と同時に登録する。

---

### 11.14 初期利用可能資産設定の関連先

初期利用可能資産設定の

```text
asset_account_id
```

には、ACC-002で新規登録した

```text
asset_accounts.id
```

を設定する。

クライアントから`assetAccountId`を指定させない。

概念的には、

```text
asset_accounts
INSERT
    ↓
生成されたid
    ↓
asset_account_available_settings.asset_account_id
```

とする。

---

### 11.15 資産口座と初期設定を一体として扱う

ACC-002において、

```text
asset_accounts
```

の登録だけ成功し、

```text
asset_account_available_settings
```

の登録だけ失敗した状態は許容しない。

逆に、資産口座が存在しない状態で初期利用可能資産設定だけが登録される状態も許容しない。

両方の登録を1つの業務処理として扱う。

---

## 12. 処理フロー

概念的な処理フローは、以下とする。

```text
リクエスト受付
    ↓
リクエストID生成
    ↓
X-User-Id検証
    ↓
操作対象利用者の特定
    ↓
リクエストボディ検証
    ↓
同一利用者内の資産口座名重複確認
    ↓
トランザクション開始
    ↓
asset_accounts登録
    ↓
生成されたasset_accounts.idを取得
    ↓
asset_account_available_settings登録
    ↓
トランザクションコミット
    ↓
APIレスポンス用の形式へ変換
    ↓
正常レスポンス返却
```

---

### 12.1 利用者コンテキスト確認

共通Middlewareで`X-User-Id`を検証し、操作対象利用者を特定する。

利用者コンテキストが不正な場合は、リクエストボディの業務処理へ進まない。

---

### 12.2 リクエストボディ検証

以下の入力値を検証する。

```text
name
assetType
balanceRecordingUnit
startYearMonth
isAvailable
```

バリデーションに失敗した場合は、データベース登録処理へ進まない。

---

### 12.3 資産口座名重複確認

操作対象利用者について、同名の資産口座が存在しないことを確認する。

論理削除済み資産口座も重複確認対象とする。

概念的には、

```text
userId
+
name
    ↓
利用中・論理削除済みを含めて検索
    ↓
存在する
    → 登録不可

存在しない
    → 登録処理へ進む
```

とする。

---

### 12.4 資産口座登録

`asset_accounts`へ資産口座を登録する。

概念的な登録内容は、以下とする。

```text
user_id
    = 操作対象利用者ID

name
    = request.name

asset_type
    = request.assetTypeをDB保存形式へ変換

balance_recording_unit
    = request.balanceRecordingUnitをDB保存形式へ変換

start_year_month
    = request.startYearMonth

deleted_at
    = NULL
```

`id`、`created_at`、`updated_at`は、Laravelおよびデータベースの通常の登録処理に従う。

---

### 12.5 初期利用可能資産設定登録

資産口座登録後、生成された資産口座IDを使用して

```text
asset_account_available_settings
```

へ初期設定を登録する。

概念的な登録内容は、以下とする。

```text
asset_account_id
    = 新規登録したasset_accounts.id

start_year_month
    = request.startYearMonth

end_year_month
    = NULL

is_available
    = request.isAvailable
```

---

### 12.6 登録結果生成

資産口座と初期利用可能資産設定の登録完了後、APIレスポンスに必要な登録結果を生成する。

データベースカラムをそのままレスポンスへ返却しない。

API用のDTOおよびAPI Resourceを通してレスポンス形式へ変換する。

---

## 13. トランザクション境界

ACC-002では、明示的なデータベーストランザクションを使用する。

理由は、1回のAPI実行で

```text
asset_accounts
```

と

```text
asset_account_available_settings
```

の2テーブルへ関連するデータを登録するためである。

---

### 13.1 トランザクション対象

以下の処理を同一トランザクション内で実行する。

```text
BEGIN
    ↓
asset_accounts INSERT
    ↓
asset_account_available_settings INSERT
    ↓
COMMIT
```

どちらかの登録に失敗した場合は、

```text
ROLLBACK
```

する。

---

### 13.2 資産口座登録失敗

`asset_accounts`の登録に失敗した場合は、初期利用可能資産設定を登録しない。

概念的には、

```text
asset_accounts
INSERT失敗
    ↓
ROLLBACK
    ↓
asset_account_available_settings
INSERTしない
```

となる。

---

### 13.3 初期利用可能資産設定登録失敗

`asset_accounts`の登録成功後に、

```text
asset_account_available_settings
```

の登録に失敗した場合は、先に登録した`asset_accounts`もロールバックする。

概念的には、

```text
asset_accounts
INSERT成功
    ↓
asset_account_available_settings
INSERT失敗
    ↓
ROLLBACK
    ↓
asset_accountsのINSERTも取消
```

とする。

---

### 13.4 トランザクション外で行う処理

以下は、原則としてトランザクション開始前に行う。

* `X-User-Id`の検証
* 操作対象利用者の特定
* リクエスト形式のバリデーション

不要な時間、トランザクションを保持しないためである。

資産口座名の重複については、事前確認だけに依存せず、データベースの一意性制約と組み合わせて整合性を保証する。

---

### 13.5 トランザクションの実装単位

Laravelでは、UseCaseからトランザクション境界を制御する。

概念的には、

```php
DB::transaction(function () {
    // 資産口座登録
    // 初期利用可能資産設定登録
});
```

とする。

Repository単位で個別にトランザクションを開始せず、

```text
資産口座登録
+
初期利用可能資産設定登録
```

というユースケース全体を1つのトランザクション境界とする。


---

## 14. 排他制御

ACC-002では、同一利用者に対して同名の資産口座が同時登録される可能性を考慮する。

アプリケーション側では登録前に重複確認を行うが、重複確認だけでは同時実行時の競合を完全には防止できない。

そのため、同一利用者内の資産口座名についてデータベースの一意性制約を最終防衛線として使用する。

概念的には、以下の組み合わせに一意性を保証する。

```text
user_id
+
name
```

論理削除済み資産口座も同名再登録不可とする設計のため、一意性制約は論理削除状態にかかわらず同一利用者・同一名称を重複させない構成とする。

---

### 14.1 登録前の重複確認

ACC-002では、登録処理前に

```text
user_id
+
name
```

で既存資産口座を確認する。

概念的には、

```text
同一利用者
+
同一資産口座名
    ↓
存在する
    → 登録不可

存在しない
    → 登録処理へ進む
```

とする。

ただし、この確認だけを一意性保証としてはならない。

---

### 14.2 同時実行

例えば、同一利用者について同じ資産口座名を2つのリクエストが同時に登録する場合を考える。

```text
Request A
重複なし確認

Request B
重複なし確認

Request A
INSERT

Request B
INSERT
```

この場合、アプリケーション側の事前確認だけでは両方が登録処理へ進む可能性がある。

そのため、データベースのUNIQUE制約によって2件目の登録を防止する。

---

### 14.3 行ロック

ACC-002では、新規レコードを登録する処理であり、登録前には対象となる`asset_accounts`行が存在しない。

そのため、

```php
lockForUpdate()
```

による既存行ロックは、同名新規登録の競合防止には適していない。

Phase1では、同名資産口座の重複防止を

```text
事前重複確認
+
UNIQUE制約
```

によって行う。

---

### 14.4 利用者行をロックしない

同一利用者の資産口座登録を直列化するために、

```text
users
```

の行を`lockForUpdate()`する方式は採用しない。

資産口座登録のたびに利用者行をロックすると、同一利用者に関する他の処理まで不要にブロックする可能性があるためである。

---

### 14.5 初期利用可能資産設定

ACC-002では、新規作成した資産口座に対して初期利用可能資産設定を1件登録する。

資産口座IDはACC-002内で新規生成されたものであるため、通常は同じ`asset_account_id`へ別リクエストが同時に初期設定を登録することはない。

初期設定の整合性は、資産口座登録と同一トランザクションで登録することによって保証する。

---

### 14.6 UNIQUE制約違反

同時実行によって`asset_accounts`の一意性制約違反が発生した場合は、PostgreSQLの例外をそのままクライアントへ返却しない。

業務上の

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換し、適切なHTTPステータスで返却する。

制約名やSQLはレスポンスへ公開しない。

---

## 15. 成功レスポンス

資産口座および初期利用可能資産設定の登録に成功した場合は、

```text
201 Created
```

を返却する。

正常時は、API共通方針に従った成功レスポンス形式を使用する。

概念例：

```json
{
  "data": {
    "id": "10",
    "name": "証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

---

### 15.1 レスポンス項目

| 項目                          | 型       | NULL | 説明                  |
| --------------------------- | ------- | :--: | ------------------- |
| `data.id`                   | string  |   ×  | 登録した資産口座ID          |
| `data.name`                 | string  |   ×  | 資産口座名               |
| `data.assetType`            | string  |   ×  | 資産種別                |
| `data.balanceRecordingUnit` | string  |   ×  | 残高記録単位              |
| `data.isAvailable`          | boolean |   ×  | 登録した初期利用可能資産区分      |
| `data.startYearMonth`       | string  |   ×  | 利用開始年月。`YYYY-MM`形式  |
| `data.isEnabled`            | boolean |   ×  | 利用状態。新規登録時は常に`true` |

---

### 15.2 id

新規登録した資産口座IDを返却する。

API共通方針に従い、データベースでは`bigint`であってもAPIレスポンスでは文字列として返却する。

例：

```json
{
  "id": "10"
}
```

---

### 15.3 name

登録した資産口座名を返却する。

例：

```json
{
  "name": "証券口座"
}
```

---

### 15.4 assetType

登録した資産種別を、API用の文字列で返却する。

例：

```json
{
  "assetType": "SECURITIES"
}
```

データベース内部の数値コードをそのまま返却しない。

---

### 15.5 balanceRecordingUnit

登録した残高記録単位を、API用の文字列で返却する。

例：

```json
{
  "balanceRecordingUnit": "HOLDING"
}
```

想定する値は、以下とする。

```text
ACCOUNT
HOLDING
```

---

### 15.6 isAvailable

ACC-002で登録した初期利用可能資産設定の`is_available`を返却する。

例：

```json
{
  "isAvailable": true
}
```

`asset_accounts`の直接カラムとして保持している値ではない。

---

### 15.7 startYearMonth

資産口座の利用開始年月を返却する。

形式は、

```text
YYYY-MM
```

とする。

例：

```json
{
  "startYearMonth": "2026-08"
}
```

---

### 15.8 isEnabled

新規登録直後の資産口座は利用中であるため、

```json
{
  "isEnabled": true
}
```

を返却する。

データベース上では、

```text
deleted_at = NULL
```

に対応する。

`deleted_at`自体はレスポンスへ返却しない。

---

### 15.9 返却しない情報

ACC-002では、以下の内部情報をレスポンスへ返却しない。

* `user_id`
* `deleted_at`
* `created_at`
* `updated_at`
* `asset_account_available_settings.id`
* `asset_account_available_settings.asset_account_id`
* `asset_account_available_settings.start_year_month`
* `asset_account_available_settings.end_year_month`
* データベース内部の数値コード

利用可能資産設定については、登録結果として必要な

```text
isAvailable
```

のみ返却する。

---

## 16. エラーレスポンス

異常時は、API共通方針で定めた共通エラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "name",
        "reason": "required",
        "message": "資産口座名を指定してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

### 16.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

この場合、資産口座登録処理へ進まない。

---

### 16.2 INVALID_USER_ID

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

例えば、以下のような値を不正とする。

```text
0
-1
abc
1.5
```

---

### 16.3 USER_NOT_FOUND

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

この場合、

```text
asset_accounts
asset_account_available_settings
```

のどちらにも登録しない。

---

### 16.4 VALIDATION_ERROR

リクエストボディの項目バリデーションに失敗した場合は、

```text
VALIDATION_ERROR
```

を返却する。

対象となる主な項目は、以下とする。

* `name`
* `assetType`
* `balanceRecordingUnit`
* `startYearMonth`
* `isAvailable`

複数の入力エラーがある場合は、`details`へ複数件返却してよい。

---

### 16.5 name未指定

`name`が指定されていない場合は、`VALIDATION_ERROR`として扱う。

概念例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "name",
        "reason": "required",
        "message": "資産口座名を指定してください。"
      }
    ]
  }
}
```

---

### 16.6 assetType不正

`assetType`が定義済みの値ではない場合は、`VALIDATION_ERROR`として扱う。

例えば、

```json
{
  "assetType": "CRYPTO"
}
```

のような未定義値は受け付けない。

---

### 16.7 balanceRecordingUnit不正

`balanceRecordingUnit`が

```text
ACCOUNT
HOLDING
```

以外の場合は、`VALIDATION_ERROR`として扱う。

---

### 16.8 startYearMonth不正

`startYearMonth`が

```text
YYYY-MM
```

形式ではない、または実在しない年月の場合は、`VALIDATION_ERROR`として扱う。

例えば、

```text
2026-13
2026/08
202608
```

などを不正とする。

---

### 16.9 isAvailable不正

`isAvailable`がbooleanではない場合は、`VALIDATION_ERROR`として扱う。

例えば、

```json
{
  "isAvailable": "true"
}
```

や、

```json
{
  "isAvailable": 1
}
```

は不正とする。

---

### 16.10 ASSET_ACCOUNT_NAME_ALREADY_EXISTS

操作対象利用者について、同じ資産口座名が既に存在する場合は、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

を返却する。

論理削除済み資産口座も重複判定対象とする。

概念的には、

```text
User A
    証券口座
```

が存在する状態で、User Aとして再度

```json
{
  "name": "証券口座"
}
```

を登録した場合にエラーとする。

---

### 16.11 他利用者の同名資産口座

他利用者にのみ同名資産口座が存在する場合は、重複エラーとしない。

例えば、

```text
User A
    証券口座

User B
    資産口座なし
```

の状態で、User Bとして

```json
{
  "name": "証券口座"
}
```

を登録することは許可する。

---

### 16.12 同時登録による重複

事前重複確認を通過した後に、並行リクエストによって同一利用者・同一名称の資産口座が先に登録された場合は、データベースのUNIQUE制約によって重複を防止する。

この場合も、クライアントへは

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

として返却する。

PostgreSQLの一意制約違反をそのまま公開しない。

---

### 16.13 初期利用可能資産設定登録失敗

`asset_accounts`の登録後に、初期

```text
asset_account_available_settings
```

の登録に失敗した場合は、トランザクションをロールバックする。

クライアントへ初期設定だけの専用内部エラーを公開することは基本としない。

想定外の登録失敗である場合は、

```text
INTERNAL_SERVER_ERROR
```

として扱う。

---

### 16.14 INTERNAL_SERVER_ERROR

資産口座登録処理で想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

トランザクション中の場合は、すべてロールバックする。

レスポンスへ、以下の内部情報を含めない。

* SQL
* PostgreSQL内部エラー
* 制約名
* テーブル名
* カラム名
* Laravel内部例外メッセージ
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 16.15 エラー時のデータ更新

いずれのエラーの場合も、ACC-002の登録処理が正常完了していない限り、

```text
asset_accountsだけ登録済み
```

または、

```text
asset_account_available_settingsだけ登録済み
```

という状態を残してはならない。

トランザクションによって、

```text
両方成功
または
両方未登録
```

を保証する。

---

## 17. HTTPステータス

ACC-002では、処理結果に応じて以下のHTTPステータスを返却する。

| HTTPステータス                   | 用途                |
| --------------------------- | ----------------- |
| `201 Created`               | 資産口座登録成功          |
| `400 Bad Request`           | 利用者コンテキスト不正、入力値不正 |
| `404 Not Found`             | 操作対象利用者不存在        |
| `409 Conflict`              | 同一利用者内の資産口座名重複    |
| `500 Internal Server Error` | 想定外のサーバー内部エラー     |

具体的なエラーコードは、API共通方針およびACC-002で定義したエラーレスポンスに従う。

---

### 17.1 201 Created

資産口座および初期利用可能資産設定の登録が正常に完了した場合は、

```http
201 Created
```

を返却する。

以下の両方が正常に登録された時点で成功とする。

```text
asset_accounts
    +
asset_account_available_settings
```

どちらか一方だけが登録された状態では、`201 Created`を返却しない。

---

### 17.2 400 Bad Request

以下の場合は、

```http
400 Bad Request
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
VALIDATION_ERROR
```

例えば、以下を対象とする。

* `X-User-Id`未指定
* `X-User-Id`形式不正
* `name`未指定
* `assetType`不正
* `balanceRecordingUnit`不正
* `startYearMonth`不正
* `isAvailable`不正

---

### 17.3 404 Not Found

指定された利用者が存在しない場合、または論理削除済みの場合は、

```http
404 Not Found
```

を返却する。

エラーコードは、

```text
USER_NOT_FOUND
```

とする。

---

### 17.4 409 Conflict

操作対象利用者について、同名の資産口座が既に存在する場合は、

```http
409 Conflict
```

を返却する。

エラーコードは、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

とする。

アプリケーション側の事前重複確認によって検出した場合だけでなく、同時実行によってデータベースのUNIQUE制約違反が発生した場合も、同じHTTPステータスおよびエラーコードへ変換する。

---

### 17.5 500 Internal Server Error

資産口座登録処理中に想定外の例外が発生した場合は、

```http
500 Internal Server Error
```

を返却する。

エラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

トランザクション開始後に発生した場合は、登録処理をロールバックする。

---

## 18. 冪等性

ACC-002は、新しい資産口座を登録するAPIであるため、冪等ではない。

同一リクエストを複数回送信した場合、毎回同じ結果になることを保証しない。

---

### 18.1 同一リクエストの再送

例えば、以下のリクエストを送信する。

```json
{
  "name": "証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

1回目の登録に成功した場合は、

```http
201 Created
```

となる。

同じ利用者が同じ`name`で再度登録した場合は、同一利用者内の資産口座名重複となるため、

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となる。

---

### 18.2 重複防止と冪等性は区別する

同一利用者内の資産口座名重複を防止することは、ACC-002自体を冪等にすることを意味しない。

ACC-002は、

```http
POST /api/v1/asset-accounts
```

によって新しいリソースを作成するAPIであり、同一リクエストの再送に対して最初の成功レスポンスを再現することは保証しない。

---

### 18.3 Idempotency-Key

Phase1では、ACC-002専用の

```text
Idempotency-Key
```

は導入しない。

同一利用者・同一資産口座名の重複登録については、

```text
アプリケーション側の事前重複確認
    +
データベースのUNIQUE制約
```

によって防止する。

将来的に、通信再送などに対して同一登録結果を保証する必要が生じた場合は、Idempotency-Keyの導入を検討する。

---

## 19. キャッシュ

Phase1では、ACC-002専用のサーバー側アプリケーションキャッシュを使用しない。

資産口座および初期利用可能資産設定は、PostgreSQLへ直接登録する。

---

### 19.1 登録処理のキャッシュ

ACC-002は登録APIであるため、リクエスト結果をHTTPキャッシュの対象としない。

```http
POST /api/v1/asset-accounts
```

のレスポンスを利用して、後続の同一リクエストをキャッシュから処理する方式は採用しない。

---

### 19.2 資産口座一覧への影響

ACC-002成功後は、ACC-001 資産口座一覧取得の取得結果が変化する。

概念的には、

```text
ACC-002
資産口座登録成功
    ↓
asset_accounts追加
    ↓
ACC-001の取得結果が変化
```

となる。

フロントエンドでTanStack QueryなどのQuery Cacheを使用する場合は、ACC-002成功後に資産口座一覧のQuery Cacheを無効化する。

具体的な処理は、「React・TypeScriptでの利用」で定義する。

---

### 19.3 資産口座詳細への影響

ACC-002成功後は、新しく生成された

```text
data.id
```

を使用して、ACC-003 資産口座詳細取得を実行できる。

Phase1では、ACC-002成功レスポンスをサーバー側キャッシュへ保存してACC-003へ流用する方式は採用しない。

必要な場合は、ACC-003から最新状態を取得する。

---

### 19.4 利用可能資産設定への影響

ACC-002では、資産口座と同時に

```text
asset_account_available_settings
```

へ初期利用可能資産設定を登録する。

そのため、対象年月における利用可能資産区分を参照する処理についても、ACC-002成功後は最新のデータベース状態を参照する必要がある。

Phase1では、利用可能資産設定についてもサーバー側アプリケーションキャッシュを使用しない。

---

### 19.5 Cache-Control

ACC-002の成功レスポンスは、登録操作の結果を含むため、共有キャッシュによる再利用を前提としない。

HTTPキャッシュに関する具体的なヘッダー設定は、API共通方針に従う。

ACC-002独自のキャッシュ制御ルールは設けない。


---

## 20. 関連テーブル

ACC-002では、資産口座および初期利用可能資産設定を登録するため、以下のテーブルを使用する。

| テーブル                               | 用途               |  更新 |
| ---------------------------------- | ---------------- | :-: |
| `users`                            | 操作対象利用者の確認       |  ×  |
| `asset_accounts`                   | 資産口座の新規登録、同名重複確認 |  ○  |
| `asset_account_available_settings` | 初期利用可能資産設定の新規登録  |  ○  |

ACC-002では、

```text
asset_accounts
asset_account_available_settings
```

の2テーブルを更新する。

`users`は、利用者コンテキスト確認のために参照するのみとする。

---

### 20.1 users

`X-User-Id`で指定された操作対象利用者の存在確認に使用する。

概念的な条件は、以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者コンテキストの検証は、API共通Middlewareで行う。

ACC-002では、`users`を更新しない。

---

### 20.2 asset_accounts

資産口座の基本情報を新規登録するために使用する。

また、同一利用者内の資産口座名重複確認にも使用する。

主に以下のカラムを使用する。

| カラム                      | 用途                         |
| ------------------------ | -------------------------- |
| `id`                     | 資産口座ID。初期利用可能資産設定の関連先として使用 |
| `user_id`                | 操作対象利用者との関連                |
| `name`                   | 資産口座名、同名重複確認               |
| `asset_type`             | 資産種別                       |
| `balance_recording_unit` | 残高記録単位                     |
| `start_year_month`       | 利用開始年月                     |
| `created_at`             | 登録日時                       |
| `updated_at`             | 更新日時                       |
| `deleted_at`             | 利用状態。新規登録時は`NULL`          |

---

### 20.3 asset_accountsへの登録

リクエスト内容と利用者コンテキストから、概念的に以下を登録する。

```text
user_id
    = 操作対象利用者ID

name
    = request.name

asset_type
    = request.assetTypeを
      DB保存形式へ変換した値

balance_recording_unit
    = request.balanceRecordingUnitを
      DB保存形式へ変換した値

start_year_month
    = request.startYearMonth

deleted_at
    = NULL
```

`id`、`created_at`、`updated_at`は、Laravelおよびデータベースの通常の登録処理に従う。

---

### 20.4 資産口座名重複確認

同一利用者内では、資産口座名を重複させない。

概念的な検索条件は、以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = request.name
```

論理削除済み資産口座も重複確認対象に含める。

Laravel SoftDeletesを使用する場合は、重複確認時に`withTrashed()`を利用する。

---

### 20.5 asset_accountsの一意性

同時登録による重複を防止するため、概念的には

```text
user_id
+
name
```

の組み合わせに一意性を保証する。

これにより、アプリケーション側の重複確認を同時に通過した場合でも、同一利用者に同名資産口座が複数作成されることを防止する。

---

### 20.6 asset_account_available_settings

資産口座登録時の初期利用可能資産設定を新規登録するために使用する。

主に以下のカラムを使用する。

| カラム                | 用途                     |
| ------------------ | ---------------------- |
| `id`               | 利用可能資産設定ID             |
| `asset_account_id` | 登録した資産口座との関連           |
| `start_year_month` | 初期設定の適用開始年月            |
| `end_year_month`   | 初期設定の適用終了年月。登録時は`NULL` |
| `is_available`     | 利用可能資産区分               |
| `created_at`       | 登録日時                   |
| `updated_at`       | 更新日時                   |

---

### 20.7 初期利用可能資産設定への登録

`asset_accounts`の登録後、生成された資産口座IDを使用して概念的に以下を登録する。

```text
asset_account_id
    = 新規登録したasset_accounts.id

start_year_month
    = request.startYearMonth

end_year_month
    = NULL

is_available
    = request.isAvailable
```

これにより、資産口座の利用開始年月から有効な初期利用可能資産設定が必ず1件存在する状態とする。

---

### 20.8 start_year_monthの整合性

初期利用可能資産設定の

```text
start_year_month
```

は、資産口座の

```text
asset_accounts.start_year_month
```

と一致させる。

以下のような状態をACC-002から作らない。

```text
asset_accounts.start_year_month
    = 2026-08

asset_account_available_settings.start_year_month
    = 2026-09
```

または、

```text
asset_accounts.start_year_month
    = 2026-08

asset_account_available_settings.start_year_month
    = 2026-07
```

初期設定の開始年月は、クライアントから別項目として受け付けない。

---

### 20.9 end_year_month

初期利用可能資産設定の

```text
end_year_month
```

は、新規登録時は

```text
NULL
```

とする。

利用可能資産区分の変更時に、ACC-006の業務ルールに従って現在の設定期間を終了させ、新しい設定履歴を追加する。

---

### 20.10 is_available

リクエストの

```text
isAvailable
```

を、

```text
asset_account_available_settings.is_available
```

へ保存する。

`asset_accounts`へ同じ値を重複保持しない。

---

### 20.11 更新しないテーブル

ACC-002では、以下のテーブルを更新しない。

* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

資産口座登録によって、保有商品や月末資産データを自動作成しない。

---

## 21. 関連する機能要件

ACC-002は、資産口座登録および初期利用可能資産設定に関する機能要件と対応する。

主な関連要件は、以下とする。

* `4.1 概要`

  * 資産口座は月末資産管理および目的達成判定の基礎情報となる
* `4.2 登録`

  * 資産口座を新規登録できる
  * 資産口座名、資産種別、残高記録単位、利用開始年月を登録する
* `4.3 資産種別`

  * 定義済みの資産種別から選択する
* `4.4 残高記録単位`

  * 口座単位または商品単位を指定する
  * 商品単位の場合でも資産口座登録時に保有商品を自動登録しない
* `4.5 利用開始年月`

  * 資産口座を管理対象とする開始年月を設定する
* `4.6 利用可能資産区分`

  * 目的達成判定へ利用する資産かどうかを管理する
  * 資産口座登録時に初期設定を登録する
* `4.7 利用可能資産区分の変更`

  * 利用可能資産区分は履歴として管理する
  * 初期設定以降の変更は設定履歴を追加する
* 利用者境界

  * 操作対象利用者に帰属する資産口座として登録する
  * 他利用者のIDをリクエストボディから指定できない
* 資産口座名

  * 同一利用者内で重複する資産口座名を登録しない
  * 異なる利用者間では同名を許可する

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

## 22. テスト観点

ACC-002では、資産口座登録だけでなく、以下を重点的に確認する。

* 利用者境界
* 入力値
* 同名重複
* 初期利用可能資産設定
* トランザクション
* 同時登録

---

### 22.1 正常系

以下のような正常なリクエストを送信する。

```json
{
  "name": "証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

以下を確認する。

* `201 Created`となること
* `asset_accounts`が1件登録されること
* `asset_account_available_settings`が1件登録されること
* 両レコードが正しく関連していること
* 正常レスポンスが返却されること

---

### 22.2 user_id

新規登録された`asset_accounts.user_id`が、

```text
X-User-Id
```

から特定した操作対象利用者IDと一致することを確認する。

リクエストボディから`user_id`を設定していないことも確認する。

---

### 22.3 name

正常な資産口座名を指定し、そのまま

```text
asset_accounts.name
```

へ保存されることを確認する。

以下も確認する。

* `name`未指定
* `name = null`
* 空文字
* 最大文字数以内
* 最大文字数超過

不正な場合は、資産口座および初期利用可能資産設定が登録されないことを確認する。

---

### 22.4 assetType

以下の正常値をそれぞれ確認する。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

APIの文字列値が正しいDB保存形式へ変換されることを確認する。

以下のような未定義値も確認する。

```text
CRYPTO
UNKNOWN
```

不正な場合は、登録されないこと。

---

### 22.5 balanceRecordingUnit

以下の正常値を確認する。

```text
ACCOUNT
HOLDING
```

API用の文字列が正しいDB保存形式へ変換されることを確認する。

以下のような不正値も確認する。

```text
UNKNOWN
ASSET
```

不正時に業務データが登録されないこと。

---

### 22.6 ACCOUNT

```text
balanceRecordingUnit = ACCOUNT
```

で資産口座を正常登録できることを確認する。

また、ACC-002実行によって

```text
month_end_asset_balances
```

が自動登録されないことを確認する。

---

### 22.7 HOLDING

```text
balanceRecordingUnit = HOLDING
```

で資産口座を正常登録できることを確認する。

また、ACC-002実行によって

```text
holding_assets
```

が自動登録されないことを確認する。

保有商品登録はHLD-002の責務であることを確認する。

---

### 22.8 startYearMonth

正常値として、例えば以下を確認する。

```text
2026-01
2026-08
2027-12
```

以下の不正値も確認する。

```text
2026-1
2026/08
202608
2026-00
2026-13
abc
```

不正な場合は、`asset_accounts`および`asset_account_available_settings`が登録されないこと。

---

### 22.9 isAvailable = true

```json
{
  "isAvailable": true
}
```

を指定する。

初期利用可能資産設定の

```text
is_available = true
```

となることを確認する。

---

### 22.10 isAvailable = false

```json
{
  "isAvailable": false
}
```

を指定する。

初期利用可能資産設定の

```text
is_available = false
```

となることを確認する。

`false`を未指定扱いしないことも確認する。

---

### 22.11 isAvailableの型

以下を不正値として確認する。

```json
{
  "isAvailable": "true"
}
```

```json
{
  "isAvailable": 1
}
```

```json
{
  "isAvailable": null
}
```

boolean以外を受け付けないこと。

---

### 22.12 初期利用可能資産設定の開始年月

資産口座の

```text
start_year_month
```

と、初期利用可能資産設定の

```text
start_year_month
```

が一致することを確認する。

例えば、

```text
asset_accounts.start_year_month
    = 2026-08

asset_account_available_settings.start_year_month
    = 2026-08
```

となること。

---

### 22.13 初期利用可能資産設定の終了年月

ACC-002で登録した初期設定の

```text
end_year_month
```

が

```text
NULL
```

となることを確認する。

---

### 22.14 asset_account_id

初期利用可能資産設定の

```text
asset_account_id
```

が、同じACC-002で新規登録した

```text
asset_accounts.id
```

と一致することを確認する。

他資産口座へ紐づかないこと。

---

### 22.15 登録時の利用状態

新規資産口座の

```text
deleted_at
```

が

```text
NULL
```

となることを確認する。

正常レスポンスの

```text
isEnabled
```

が

```text
true
```

となることも確認する。

---

### 22.16 同一利用者の同名資産口座

操作対象利用者に

```text
証券口座
```

が存在する状態で、同じ利用者として再度同名を登録する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

以下を確認する。

* 新しい`asset_accounts`が登録されないこと
* 新しい`asset_account_available_settings`も登録されないこと

---

### 22.17 論理削除済み同名資産口座

同一利用者に

```text
name = 証券口座
deleted_at IS NOT NULL
```

の資産口座を用意する。

同名でACC-002を実行する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となること。

論理削除済みデータも重複判定対象となることを確認する。

---

### 22.18 他利用者の同名資産口座

以下の状態を用意する。

```text
User A
    証券口座

User B
    資産口座なし
```

User Bとして

```text
証券口座
```

を登録する。

正常に登録できることを確認する。

User Aの資産口座によって重複エラーにならないこと。

---

### 22.19 X-User-Id未指定

`X-User-Id`を指定しない。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

以下を確認する。

* `asset_accounts`が登録されないこと
* `asset_account_available_settings`が登録されないこと

---

### 22.20 X-User-Id形式不正

例えば、

```http
X-User-Id: abc
```

を指定する。

期待結果：

```text
400 Bad Request
INVALID_USER_ID
```

業務データが更新されないこと。

---

### 22.21 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 22.22 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

資産口座が登録されないこと。

---

### 22.23 userIdをリクエストへ含める

クライアントから以下のような`userId`を送信しても、その値を

```text
asset_accounts.user_id
```

として使用しないことを確認する。

```json
{
  "name": "証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "startYearMonth": "2026-08",
  "isAvailable": true,
  "userId": "999"
}
```

未定義項目を拒否する共通方針の場合は、バリデーションエラーとなることを確認する。

---

### 22.24 初期設定登録失敗時のロールバック

`asset_accounts`の登録後に、意図的に

```text
asset_account_available_settings
```

の登録を失敗させる。

概念的には、

```text
asset_accounts INSERT
    ↓
成功

available_settings INSERT
    ↓
失敗

ROLLBACK
```

となることを確認する。

最終的に、

```text
asset_accounts
    → 新規レコードなし

asset_account_available_settings
    → 新規レコードなし
```

となること。

---

### 22.25 資産口座登録失敗

`asset_accounts`のINSERTを失敗させる。

以下を確認する。

* `asset_account_available_settings`の登録へ進まないこと
* 中途半端な業務データが残らないこと

---

### 22.26 同時登録

同一利用者について、同じ`name`のACC-002を並行実行する。

以下を確認する。

* 同名資産口座が2件作成されないこと
* UNIQUE制約によって一意性が維持されること
* 一方のリクエストだけが成功すること
* 競合したリクエストが適切な`409 Conflict`となること
* PostgreSQLの制約エラーがそのまま返却されないこと

---

### 22.27 同時登録時の初期設定

同一名で並行登録した場合に、失敗側のリクエストによって孤立した

```text
asset_account_available_settings
```

が作成されないことを確認する。

---

### 22.28 正常レスポンス契約

正常時に、概念的に以下の形式となることを確認する。

```json
{
  "data": {
    "id": "10",
    "name": "証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

以下を確認する。

* HTTPステータスが`201 Created`であること
* `id`がstringであること
* `name`がstringであること
* `assetType`がAPI用文字列であること
* `balanceRecordingUnit`がAPI用文字列であること
* `isAvailable`がbooleanであること
* `startYearMonth`が`YYYY-MM`形式であること
* `isEnabled = true`であること

---

### 22.29 返却しない情報

正常レスポンスに、以下が含まれないことを確認する。

* `userId`
* `user_id`
* `deletedAt`
* `deleted_at`
* `createdAt`
* `created_at`
* `updatedAt`
* `updated_at`
* `assetAccountAvailableSettingId`
* `asset_account_available_settings.id`
* `asset_account_id`
* `endYearMonth`
* DB内部の数値コード

---

### 22.30 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
VALIDATION_ERROR
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
INTERNAL_SERVER_ERROR
```

以下も確認する。

* `error.code`が設定されること
* `error.message`が設定されること
* 必要に応じて`error.details`が設定されること
* `requestId`が設定されること
* SQLが含まれないこと
* PostgreSQLの制約名が含まれないこと
* スタックトレースが含まれないこと
* サーバーファイルパスが含まれないこと

---

### 22.31 副作用範囲

正常終了時に更新される業務テーブルが、

```text
asset_accounts
asset_account_available_settings
```

だけであることを確認する。

以下が変更されないことも確認する。

* `users`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

---

### 22.32 INTERNAL_SERVER_ERROR

資産口座または初期利用可能資産設定の登録処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

* トランザクションがロールバックされること
* 一部登録データが残らないこと
* SQLがレスポンスへ含まれないこと
* PostgreSQL内部エラーが公開されないこと
* スタックトレースが公開されないこと
* エラーログとレスポンスの`requestId`を関連付けられること

---

## 23. Laravel実装方針

ACC-002では、Action、FormRequest、UseCase、Query、Repository、DTO、API Resource、Responderを分離して実装する。

概念的な構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
FormRequest
    ↓
Action
    ↓
UseCase
    ├─ AssetAccountQuery
    ├─ AssetAccountRepository
    └─ AssetAccountAvailableSettingRepository
    ↓
Create Result DTO
    ↓
API Resource
    ↓
Responder
```

ACC-002では、

```text
asset_accounts
+
asset_account_available_settings
```

の2テーブルを同一ユースケースで登録する。

Actionへ、以下を直接記述しない。

* 入力値検証
* 利用者境界判定
* 資産口座名重複確認
* Enum変換
* トランザクション制御
* 資産口座登録
* 初期利用可能資産設定登録
* レスポンス変換

---

### 23.1 Route

ACC-002は、以下のルートとして定義する。

概念例：

```php
Route::post(
    '/api/v1/asset-accounts',
    CreateAssetAccountAction::class,
);
```

ACC-001と同じURLを使用するが、HTTPメソッドによって責務を分離する。

```text
GET
/api/v1/asset-accounts
    → ACC-001 資産口座一覧取得

POST
/api/v1/asset-accounts
    → ACC-002 資産口座登録
```

---

### 23.2 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* 共通例外処理
* ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、以下とする。

```text
X-User-Id取得
    ↓
必須チェック
    ↓
形式チェック
    ↓
users存在確認
    ↓
UserContext設定
    ↓
FormRequest
    ↓
Action
```

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 23.3 FormRequest

リクエストボディの入力値検証には、専用FormRequestを使用する。

概念例：

```php
final class CreateAssetAccountRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => [
                'required',
                'string',
                'max:100',
            ],

            'assetType' => [
                'required',
                'string',
                Rule::enum(
                    AssetType::class,
                ),
            ],

            'balanceRecordingUnit' => [
                'required',
                'string',
                Rule::enum(
                    BalanceRecordingUnit::class,
                ),
            ],

            'startYearMonth' => [
                'required',
                'string',
                new YearMonthRule(),
            ],

            'isAvailable' => [
                'required',
                'boolean',
            ],
        ];
    }
}
```

最大文字数やEnum定義については、テーブル定義およびAPI共通定義に従う。

---

### 23.4 FormRequestで行うこと

FormRequestでは、以下の入力形式を検証する。

```text
name
assetType
balanceRecordingUnit
startYearMonth
isAvailable
```

主に以下を確認する。

* 必須
* 型
* 最大文字数
* Enum値
* `YYYY-MM`形式
* boolean

HTTPリクエストとして不正な入力を、UseCaseへ渡さない。

---

### 23.5 FormRequestで行わないこと

FormRequestでは、以下の業務ルールを判定しない。

* 同一利用者内の資産口座名重複
* 論理削除済み資産口座との重複
* 資産口座登録
* 初期利用可能資産設定登録
* トランザクション制御
* 利用者IDの決定

これらは、UseCase、Query、Repositoryで扱う。

---

### 23.6 userIdをRequestから取得しない

ACC-002では、リクエストボディに

```text
userId
user_id
```

を定義しない。

操作対象利用者IDは、共通Middlewareによって設定された

```text
UserContext
```

から取得する。

概念例：

```php
$userId =
    $userContext->userId;
```

これにより、クライアントが任意の利用者IDを指定して資産口座を登録することを防止する。

---

### 23.7 Action

Actionは、検証済みリクエストと利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php
final class CreateAssetAccountAction
{
    public function __invoke(
        CreateAssetAccountRequest $request,
        CreateAssetAccountUseCase $useCase,
        CreateAssetAccountResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                name:
                    $request->string(
                        'name',
                    )->toString(),

                assetType:
                    AssetType::from(
                        $request->string(
                            'assetType',
                        )->toString(),
                    ),

                balanceRecordingUnit:
                    BalanceRecordingUnit::from(
                        $request->string(
                            'balanceRecordingUnit',
                        )->toString(),
                    ),

                startYearMonth:
                    $request->string(
                        'startYearMonth',
                    )->toString(),

                isAvailable:
                    $request->boolean(
                        'isAvailable',
                    ),
            );

        return $responder->created(
            $result,
        );
    }
}
```

実際には、引数が増える場合はInput DTOを使用してよい。

---

### 23.8 Actionで行わないこと

Actionでは、以下を行わない。

* 資産口座名重複確認
* `asset_accounts`検索
* `asset_accounts`登録
* `asset_account_available_settings`登録
* transaction制御
* SQL生成
* レスポンス配列生成
* DB保存用数値コードの直接操作

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 23.9 Input DTO

ActionからUseCaseへ複数の入力値を渡す場合は、専用Input DTOを使用してよい。

概念例：

```php
final readonly class CreateAssetAccountInput
{
    public function __construct(
        public int $userId,
        public string $name,
        public AssetType $assetType,
        public BalanceRecordingUnit $balanceRecordingUnit,
        public string $startYearMonth,
        public bool $isAvailable,
    ) {
    }
}
```

概念的には、

```php
$input =
    new CreateAssetAccountInput(
        userId:
            $userContext->userId,

        name:
            $request->string(
                'name',
            )->toString(),

        assetType:
            AssetType::from(
                $request->string(
                    'assetType',
                )->toString(),
            ),

        balanceRecordingUnit:
            BalanceRecordingUnit::from(
                $request->string(
                    'balanceRecordingUnit',
                )->toString(),
            ),

        startYearMonth:
            $request->string(
                'startYearMonth',
            )->toString(),

        isAvailable:
            $request->boolean(
                'isAvailable',
            ),
    );
```

とする。

---

### 23.10 Enum

`assetType`および`balanceRecordingUnit`は、PHP Enumとして表現する。

概念例：

```php
enum AssetType: string
{
    case CASH = 'CASH';
    case BANK = 'BANK';
    case SECURITIES = 'SECURITIES';
    case IDECO = 'IDECO';
    case CORPORATE_DC = 'CORPORATE_DC';
    case OTHER = 'OTHER';
}
```

```php
enum BalanceRecordingUnit: string
{
    case ACCOUNT = 'ACCOUNT';
    case HOLDING = 'HOLDING';
}
```

データベースへ`smallint`などで保存する場合は、保存用コードへの変換責務をModel Cast、Mapper、またはEnum側へ集約する。

ActionやUseCase内にマジックナンバーを直接記述しない。

---

### 23.11 YearMonth

`startYearMonth`は、単なる自由文字列としてUseCase内で扱い続けず、必要に応じて年月を表すValue Objectへ変換してよい。

概念例：

```php
$startYearMonth =
    YearMonth::fromString(
        $input->startYearMonth,
    );
```

Value Objectを採用する場合は、以下を集約できる。

* `YYYY-MM`形式保証
* 年月比較
* DB保存形式への変換

Phase1で過剰な抽象化になる場合は、FormRequestで形式保証したstringとして扱ってもよい。

---

### 23.12 UseCase

ACC-002の業務処理全体を担当する。

主な処理は、以下とする。

1. 入力値を受け取る
2. 同一利用者内の資産口座名重複を確認する
3. 重複している場合は業務例外を送出する
4. トランザクションを開始する
5. 資産口座を登録する
6. 新規資産口座IDを取得する
7. 初期利用可能資産設定を登録する
8. Create Result DTOを生成する
9. トランザクションをコミットする
10. DTOを返却する

概念的には、以下とする。

```text
CreateAssetAccountInput
    ↓
AssetAccountQuery
    ↓
同名重複確認
    ↓
DB::transaction
    ↓
AssetAccountRepository
    ↓
asset_accounts INSERT
    ↓
AssetAccountAvailableSettingRepository
    ↓
asset_account_available_settings INSERT
    ↓
Create Result DTO
```

---

### 23.13 資産口座名の重複確認

UseCaseでは、Queryを使用して同一利用者内に同名資産口座が存在しないことを確認する。

概念例：

```php
$exists =
    $this->assetAccountQuery
        ->existsByUserAndNameIncludingDeleted(
            userId:
                $input->userId,

            name:
                $input->name,
        );

if ($exists) {
    throw new
        AssetAccountNameAlreadyExistsException();
}
```

論理削除済み資産口座も重複確認対象とする。

---

### 23.14 AssetAccountQuery

資産口座名の重複確認を担当する。

概念的な検索条件は、以下とする。

```text
user_id
    = 操作対象利用者ID

AND

name
    = 登録する資産口座名
```

SoftDeletesを使用している場合でも、重複確認では論理削除済み資産口座を含める。

概念例：

```php
return AssetAccount::query()
    ->withTrashed()
    ->where(
        'user_id',
        $userId,
    )
    ->where(
        'name',
        $name,
    )
    ->exists();
```

---

### 23.15 QueryとRepositoryを分離する

ACC-002では、

```text
AssetAccountQuery
    → 重複確認・参照

AssetAccountRepository
    → 資産口座登録

AssetAccountAvailableSettingRepository
    → 初期利用可能資産設定登録
```

と責務を分離する。

QueryへINSERT処理を持たせず、Repositoryへ検索条件の組み立てを集中させない。

---

### 23.16 トランザクション

ACC-002では、UseCaseがトランザクション境界を管理する。

概念例：

```php
$result =
    DB::transaction(
        function () use (
            $input,
        ): CreateAssetAccountResult {
            // asset account登録
            // 初期available setting登録
            // DTO生成
        },
    );
```

Repositoryごとに個別のtransactionを開始しない。

---

### 23.17 AssetAccountRepository

資産口座の新規登録を担当する。

概念的なインターフェースは、以下とする。

```php
interface AssetAccountRepository
{
    public function create(
        int $userId,
        string $name,
        AssetType $assetType,
        BalanceRecordingUnit $balanceRecordingUnit,
        string $startYearMonth,
    ): AssetAccount;
}
```

実装では、`asset_accounts`へ新規レコードを登録する。

---

### 23.18 asset_accounts登録

概念例：

```php
$assetAccount =
    new AssetAccount();

$assetAccount->user_id =
    $userId;

$assetAccount->name =
    $name;

$assetAccount->asset_type =
    $assetType;

$assetAccount->balance_recording_unit =
    $balanceRecordingUnit;

$assetAccount->start_year_month =
    $startYearMonth;

$assetAccount->save();
```

実際のEnum Cast方法は、Model定義に従う。

---

### 23.19 Mass Assignment

ACC-002では、以下のようにHTTPリクエスト配列をそのままModelへ渡さない。

```php
AssetAccount::create(
    $request->all(),
);
```

特に、

```text
user_id
deleted_at
```

などのクライアント指定を防止するため、登録値は明示的に設定する。

---

### 23.20 user_id

`asset_accounts.user_id`には、必ず

```text
UserContext.userId
```

を設定する。

Requestに含まれる未知の`userId`や`user_id`を参照しない。

---

### 23.21 deleted_at

新規登録時は、SoftDeletesの通常動作に従い、

```text
deleted_at = NULL
```

とする。

Repositoryから明示的に`deleted_at`を設定する必要はない。

---

### 23.22 AssetAccountAvailableSettingRepository

初期利用可能資産設定の登録を担当する。

概念的なインターフェースは、以下とする。

```php
interface AssetAccountAvailableSettingRepository
{
    public function createInitial(
        int $assetAccountId,
        string $startYearMonth,
        bool $isAvailable,
    ): AssetAccountAvailableSetting;
}
```

---

### 23.23 初期利用可能資産設定登録

概念例：

```php
$setting =
    new AssetAccountAvailableSetting();

$setting->asset_account_id =
    $assetAccount->id;

$setting->start_year_month =
    $input->startYearMonth;

$setting->end_year_month =
    null;

$setting->is_available =
    $input->isAvailable;

$setting->save();
```

資産口座登録時は、必ず1件の初期設定を作成する。

---

### 23.24 start_year_month

初期利用可能資産設定の

```text
start_year_month
```

には、必ず資産口座の

```text
start_year_month
```

と同じ値を使用する。

概念的には、

```php
$setting->start_year_month =
    $assetAccount->start_year_month;
```

としてもよい。

クライアントから初期設定専用の開始年月を受け取らない。

---

### 23.25 end_year_month

初期利用可能資産設定の

```text
end_year_month
```

は、

```text
null
```

とする。

ACC-002で終了年月を指定させない。

---

### 23.26 is_available

Requestの

```text
isAvailable
```

は、

```text
asset_account_available_settings.is_available
```

へ保存する。

`asset_accounts`へ同じフラグを保存しない。

---

### 23.27 登録順序

トランザクション内では、以下の順序で登録する。

```text
asset_accounts
    ↓
生成されたid取得
    ↓
asset_account_available_settings
```

初期利用可能資産設定は、新規資産口座IDが必要であるため、順序を逆にしない。

---

### 23.28 初期設定登録失敗時

初期利用可能資産設定の登録に失敗した場合は、例外を送出してトランザクションをロールバックする。

その結果、

```text
asset_accounts
    → 登録されない

asset_account_available_settings
    → 登録されない
```

状態へ戻す。

---

### 23.29 同名資産口座のUNIQUE制約

アプリケーション側の事前重複確認だけでなく、データベース側でも

```text
user_id
+
name
```

の一意性を保証する。

これにより、同時リクエストによる重複登録を防止する。

---

### 23.30 UNIQUE制約違反の変換

同時登録によってUNIQUE制約違反が発生した場合は、PostgreSQL例外をそのまま返却しない。

概念的には、

```text
unique violation
    ↓
AssetAccountNameAlreadyExistsException
    ↓
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

制約名やSQLをクライアントへ公開しない。

---

### 23.31 ロック

ACC-002では、新規レコードを作成するため、対象となる資産口座行が事前に存在しない。

そのため、同名登録防止のための

```text
lockForUpdate()
```

は使用しない。

また、利用者行をロックして同一利用者の登録処理を直列化する方式も採用しない。

---

### 23.32 Create Result DTO

登録結果は、専用DTOとして表現する。

概念例：

```php
final readonly class CreateAssetAccountResult
{
    public function __construct(
        public int $id,
        public string $name,
        public AssetType $assetType,
        public BalanceRecordingUnit $balanceRecordingUnit,
        public bool $isAvailable,
        public string $startYearMonth,
    ) {
    }
}
```

`isEnabled`は、ACC-002成功時に必ず`true`となるため、Resource側で固定値として設定してよい。

---

### 23.33 DTOへ内部Modelを持たせない

以下のように、Eloquent ModelをそのままDTOへ保持しない。

```php
final readonly class CreateAssetAccountResult
{
    public function __construct(
        public AssetAccount $assetAccount,
        public AssetAccountAvailableSetting $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけをDTOへ保持する。

---

### 23.34 UseCaseの戻り値

概念例：

```php
return new CreateAssetAccountResult(
    id:
        $assetAccount->id,

    name:
        $assetAccount->name,

    assetType:
        $assetAccount->asset_type,

    balanceRecordingUnit:
        $assetAccount->balance_recording_unit,

    isAvailable:
        $setting->is_available,

    startYearMonth:
        $assetAccount->start_year_month,
);
```

ActionへEloquent Modelをそのまま返却しない。

---

### 23.35 API Resource

Create Result DTOを、専用API Resourceによってレスポンス形式へ変換する。

概念例：

```php
final class CreateAssetAccountResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'name'
                => $this->name,

            'assetType'
                => $this->assetType->value,

            'balanceRecordingUnit'
                => $this->balanceRecordingUnit->value,

            'isAvailable'
                => $this->isAvailable,

            'startYearMonth'
                => $this->startYearMonth,

            'isEnabled'
                => true,
        ];
    }
}
```

APIフィールド名は、API共通方針に従ってcamelCaseとする。

---

### 23.36 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

* `user_id`
* `deleted_at`
* `created_at`
* `updated_at`
* `asset_account_available_settings.id`
* `asset_account_available_settings.asset_account_id`
* `asset_account_available_settings.start_year_month`
* `asset_account_available_settings.end_year_month`
* DB内部の数値コード

Eloquent Modelを

```php
return $assetAccount->toArray();
```

のように直接返却しない。

---

### 23.37 Responder

Responderは、Create Result DTOを`201 Created`レスポンスへ変換する。

概念例：

```php
final class CreateAssetAccountResponder
{
    public function created(
        CreateAssetAccountResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new CreateAssetAccountResource(
                        $result,
                    ),
            ],
            Response::HTTP_CREATED,
        );
    }
}
```

`requestId`などの共通Envelope項目は、API共通レスポンス処理に従う。

---

### 23.38 Responderで行わないこと

Responderでは、以下を行わない。

* 入力値検証
* 資産口座名重複確認
* 利用者境界判定
* transaction制御
* 資産口座登録
* 初期利用可能資産設定登録
* Enum変換
* DB検索

Responderは、生成済みDTOをHTTPレスポンスへ変換することに責務を限定する。

---

### 23.39 例外変換

主な例外変換は、以下とする。

| 内部状態            | 独自エラーコード                            |
| --------------- | ----------------------------------- |
| `X-User-Id`未指定  | `USER_CONTEXT_REQUIRED`             |
| `X-User-Id`形式不正 | `INVALID_USER_ID`                   |
| 利用者不存在          | `USER_NOT_FOUND`                    |
| Request入力値不正    | `VALIDATION_ERROR`                  |
| 同名資産口座存在        | `ASSET_ACCOUNT_NAME_ALREADY_EXISTS` |
| UNIQUE制約による同名競合 | `ASSET_ACCOUNT_NAME_ALREADY_EXISTS` |
| 想定外例外           | `INTERNAL_SERVER_ERROR`             |

---

### 23.40 AssetAccountNameAlreadyExistsException

同一利用者内に同名資産口座が存在する場合は、業務例外を送出する。

概念例：

```php
if ($exists) {
    throw new
        AssetAccountNameAlreadyExistsException();
}
```

API共通Exception Handlerで、

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

---

### 23.41 ValidationException

FormRequestによる入力値不正は、Laravelのバリデーション例外をAPI共通形式へ変換する。

概念的には、

```text
ValidationException
    ↓
400 Bad Request
VALIDATION_ERROR
```

とする。

HTTPステータスは、API共通方針の定義を優先する。

---

### 23.42 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

トランザクション中に例外が発生した場合は、Laravelのtransaction機構によってロールバックする。

レスポンスへ、以下を含めない。

* SQL
* PostgreSQL内部エラー
* UNIQUE制約名
* テーブル名
* カラム名
* Laravel内部例外メッセージ
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

---

### 23.43 ログ

ACC-002では、必要に応じて以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
httpStatus
errorCode
```

`apiId`は、

```text
ACC-002
```

とする。

正常時には、必要に応じて新規登録した

```text
assetAccountId
```

を記録してよい。

ただし、不要な資産情報をログへ出力しない。

---

### 23.44 キャッシュ

Phase1では、ACC-002専用のサーバー側アプリケーションキャッシュを使用しない。

資産口座登録成功後は、DBが最新状態となる。

React側でTanStack Queryなどを使用する場合は、ACC-001等の関連Query Cacheを無効化する。

---

### 23.45 テスト実装方針

Laravel側では、Feature Testを中心としてACC-002のAPI契約と登録フローを確認する。

主に以下を確認する。

* `201 Created`
* `400 Bad Request`
* `404 Not Found`
* `409 Conflict`
* `500 Internal Server Error`
* `X-User-Id`必須
* 利用者存在確認
* `name`
* `assetType`
* `balanceRecordingUnit`
* `startYearMonth`
* `isAvailable`
* 資産口座名重複
* 論理削除済み同名資産口座
* 他利用者の同名資産口座
* 資産口座登録
* 初期利用可能資産設定登録
* transaction
* rollback
* 同時登録
* UNIQUE制約
* レスポンス契約

---

### 23.46 FormRequestのTest

FormRequestについて、以下を確認する。

```text
name
    required
    string
    max length

assetType
    required
    enum

balanceRecordingUnit
    required
    enum

startYearMonth
    required
    YYYY-MM

isAvailable
    required
    boolean
```

特に、

```text
isAvailable = false
```

を正常値として確認する。

---

### 23.47 AssetAccountQueryのDatabase Test

重複確認Queryについて、以下を確認する。

```text
同一user_id
+
同一name
+
有効
    ↓
存在あり
```

```text
同一user_id
+
同一name
+
論理削除済み
    ↓
存在あり
```

```text
別user_id
+
同一name
    ↓
操作対象利用者については存在なし
```

論理削除済みデータを正しく重複対象へ含めることを確認する。

---

### 23.48 AssetAccountRepositoryのDatabase Test

資産口座登録について、以下を確認する。

* `user_id`が正しく設定される
* `name`が正しく設定される
* `asset_type`が正しく保存される
* `balance_recording_unit`が正しく保存される
* `start_year_month`が正しく保存される
* `deleted_at = NULL`となる
* `created_at`が設定される
* `updated_at`が設定される

---

### 23.49 AssetAccountAvailableSettingRepositoryのDatabase Test

初期利用可能資産設定について、以下を確認する。

* `asset_account_id`が新規資産口座IDと一致する
* `start_year_month`が資産口座の利用開始年月と一致する
* `end_year_month = NULL`となる
* `is_available = true`を保存できる
* `is_available = false`を保存できる

---

### 23.50 UseCaseのUnit Test

UseCaseでは、QueryとRepositoryをMockし、業務フローを確認する。

正常系：

```text
重複なし
    ↓
AssetAccountRepository::create()
    ↓
AvailableSettingRepository::createInitial()
    ↓
Create Result DTO
```

異常系：

```text
重複あり
    ↓
AssetAccountNameAlreadyExistsException
```

重複時に、以下が呼び出されないことを確認する。

```text
AssetAccountRepository::create()
AssetAccountAvailableSettingRepository::createInitial()
```

---

### 23.51 トランザクションテスト

初期利用可能資産設定登録時に意図的に例外を発生させる。

以下を確認する。

```text
asset_accounts INSERT
    ↓
available_settings INSERT失敗
    ↓
ROLLBACK
```

最終的に、

```text
asset_accounts
    → 新規登録なし

asset_account_available_settings
    → 新規登録なし
```

となることを確認する。

---

### 23.52 同時登録テスト

可能であれば、Integration TestまたはDatabase Testで同一利用者・同一名称の並行登録を確認する。

以下を確認する。

* 同名資産口座が2件登録されないこと
* 1件のみ成功すること
* 競合側が`409 Conflict`相当になること
* `ASSET_ACCOUNT_NAME_ALREADY_EXISTS`へ変換されること
* 孤立した初期利用可能資産設定が残らないこと

---

### 23.53 API ResourceのTest

正常時に、以下の項目だけが`data`へ含まれることを確認する。

```json
{
  "id": "10",
  "name": "証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "isAvailable": true,
  "startYearMonth": "2026-08",
  "isEnabled": true
}
```

以下が含まれないことを確認する。

* `userId`
* `user_id`
* `deletedAt`
* `createdAt`
* `updatedAt`
* `assetAccountAvailableSettingId`
* `asset_account_id`
* `endYearMonth`
* DB内部の数値コード

---

## 24. React・TypeScriptでの利用

ACC-002は、利用者が新しい資産口座を登録する際に使用する。

フロントエンドでは、資産口座登録画面から以下の情報を入力してACC-002を実行する。

* 資産口座名
* 資産種別
* 残高記録単位
* 利用開始年月
* 利用可能資産区分

概念的な利用フローは、以下とする。

```text
資産口座一覧
    ↓
資産口座登録画面
    ↓
登録内容入力
    ↓
クライアント側入力チェック
    ↓
ACC-002
POST
/api/v1/asset-accounts
    ↓
成功
    ↓
関連Query Cache無効化
    ↓
資産口座一覧へ戻る
```

---

### 24.1 TypeScript型

ACC-002のリクエスト型は、以下のように定義する。

概念例：

```ts
export type CreateAssetAccountRequest = {
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  startYearMonth: string;
  isAvailable: boolean;
};
```

正常レスポンスのデータ型は、以下のように定義する。

```ts
export type CreateAssetAccountResult = {
  id: string;
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  isAvailable: boolean;
  startYearMonth: string;
  isEnabled: boolean;
};
```

API共通Envelopeを使用する場合は、以下のように定義する。

```ts
export type CreateAssetAccountResponse =
  ApiResponse<CreateAssetAccountResult>;
```

---

### 24.2 AssetType

`assetType`は、自由な`string`ではなく、APIで定義した値だけを扱える型とする。

概念例：

```ts
export type AssetType =
  | 'CASH'
  | 'BANK'
  | 'SECURITIES'
  | 'IDECO'
  | 'CORPORATE_DC'
  | 'OTHER';
```

画面表示用の名称とAPI送信用の値は分離する。

概念例：

```ts
export const assetTypeOptions = [
  {
    value: 'CASH',
    label: '現金',
  },
  {
    value: 'BANK',
    label: '銀行',
  },
  {
    value: 'SECURITIES',
    label: '証券',
  },
  {
    value: 'IDECO',
    label: 'iDeCo',
  },
  {
    value: 'CORPORATE_DC',
    label: '企業型DC',
  },
  {
    value: 'OTHER',
    label: 'その他',
  },
] satisfies ReadonlyArray<{
  value: AssetType;
  label: string;
}>;
```

---

### 24.3 BalanceRecordingUnit

`balanceRecordingUnit`も、APIで定義した値だけを扱える型とする。

概念例：

```ts
export type BalanceRecordingUnit =
  | 'ACCOUNT'
  | 'HOLDING';
```

画面表示では、例えば以下のように扱う。

```ts
export const balanceRecordingUnitOptions = [
  {
    value: 'ACCOUNT',
    label: '口座単位',
  },
  {
    value: 'HOLDING',
    label: '商品単位',
  },
] satisfies ReadonlyArray<{
  value: BalanceRecordingUnit;
  label: string;
}>;
```

---

### 24.4 登録フォーム

資産口座登録画面では、以下の入力項目を表示する。

| 画面項目   | API項目                  | UI例               |
| ------ | ---------------------- | ----------------- |
| 資産口座名  | `name`                 | テキスト入力            |
| 資産種別   | `assetType`            | セレクトボックス          |
| 残高記録単位 | `balanceRecordingUnit` | ラジオボタンまたはセレクトボックス |
| 利用開始年月 | `startYearMonth`       | 年月入力              |
| 利用可能資産 | `isAvailable`          | チェックボックス          |

---

### 24.5 フォームState

概念例：

```ts
export type AssetAccountFormValues = {
  name: string;
  assetType: AssetType | '';
  balanceRecordingUnit:
    | BalanceRecordingUnit
    | '';
  startYearMonth: string;
  isAvailable: boolean;
};
```

初期値の例：

```ts
const initialValues: AssetAccountFormValues = {
  name: '',
  assetType: '',
  balanceRecordingUnit: '',
  startYearMonth: '',
  isAvailable: false,
};
```

初期値については、画面設計で決定したUX方針を優先する。

---

### 24.6 利用開始年月

`startYearMonth`は、

```text
YYYY-MM
```

形式でAPIへ送信する。

HTMLの`month`入力を使用する場合は、概念的に以下とする。

```tsx
<input
  type="month"
  value={form.startYearMonth}
  onChange={(event) => {
    setForm((current) => ({
      ...current,
      startYearMonth:
        event.target.value,
    }));
  }}
/>
```

APIへ送信するために、

```text
YYYY-MM-01
```

などの日付へ変換しない。

---

### 24.7 isAvailable

`isAvailable`は、booleanとして管理する。

概念例：

```tsx
<input
  type="checkbox"
  checked={form.isAvailable}
  onChange={(event) => {
    setForm((current) => ({
      ...current,
      isAvailable:
        event.target.checked,
    }));
  }}
/>
```

APIへは、

```json
{
  "isAvailable": true
}
```

または、

```json
{
  "isAvailable": false
}
```

として送信する。

以下のような値へ変換しない。

```text
"true"
"false"
1
0
```

---

### 24.8 利用可能資産設定の詳細を入力させない

ACC-002では、利用可能資産設定の

```text
startYearMonth
endYearMonth
```

を個別入力させない。

初期利用可能資産設定の開始年月は、資産口座の

```text
startYearMonth
```

からサーバー側で設定する。

そのため、画面では

```text
資産口座の利用開始年月
+
利用可能資産かどうか
```

だけを入力する。

---

### 24.9 userIdをフォームへ持たせない

ACC-002では、登録対象利用者をリクエストボディから指定しない。

そのため、フォーム型へ

```ts
userId: string;
```

を追加しない。

以下のようなリクエストも作成しない。

```ts
const request = {
  userId: currentUserId,
  name: form.name,
  // ...
};
```

利用者IDは、`X-User-Id`として共通API Clientから送信する。

---

### 24.10 X-User-Id

`X-User-Id`は、ACC-002専用処理ではなく、共通API Clientから付与する。

概念例：

```ts
apiClient.interceptors.request.use(
  (config) => {
    config.headers['X-User-Id'] =
      currentUserId;

    return config;
  },
);
```

ACC-002のコンポーネントから直接HTTPヘッダーを組み立てない。

---

### 24.11 API Client

ACC-002を呼び出す専用関数を定義する。

概念例：

```ts
export const createAssetAccount =
  async (
    request: CreateAssetAccountRequest,
  ): Promise<CreateAssetAccountResult> => {
    const response =
      await apiClient.post<
        CreateAssetAccountResponse
      >(
        '/api/v1/asset-accounts',
        request,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さず、API Clientへ処理を集約する。

---

### 24.12 FormからRequestへの変換

フォームStateをそのままAPIへ送信するのではなく、必要に応じてRequest型へ変換する。

概念例：

```ts
const request: CreateAssetAccountRequest = {
  name: form.name.trim(),
  assetType: form.assetType as AssetType,
  balanceRecordingUnit:
    form.balanceRecordingUnit
      as BalanceRecordingUnit,
  startYearMonth:
    form.startYearMonth,
  isAvailable:
    form.isAvailable,
};
```

ただし、型アサーションに依存するより、フォームバリデーション後に型が確定する設計を優先する。

---

### 24.13 クライアント側バリデーション

API実行前に、利用者の入力ミスを早期に通知するため、フロントエンドでも基本的な入力チェックを行う。

主に以下を確認する。

```text
name
    必須
    最大文字数

assetType
    必須

balanceRecordingUnit
    必須

startYearMonth
    必須
    YYYY-MM

isAvailable
    boolean
```

ただし、フロントエンド側のバリデーションだけをデータ整合性保証とはしない。

最終的な検証はLaravel側で行う。

---

### 24.14 資産口座名の重複確認

フロントエンドでは、資産口座名の重複を登録前に確定判定しない。

例えば、ACC-001で取得済みの一覧から

```ts
assetAccounts.some(
  (assetAccount) =>
    assetAccount.name === form.name,
);
```

だけで登録可否を決定しない。

キャッシュが古い可能性や、同時登録の可能性があるためである。

同名重複の最終判定は、ACC-002のサーバー処理に任せる。

---

### 24.15 Mutationとして扱う

ACC-002は、サーバー状態を変更するため、TanStack Queryを使用する場合はMutationとして扱う。

概念例：

```ts
export const useCreateAssetAccount =
  () => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn:
        createAssetAccount,

      onSuccess: async () => {
        await queryClient
          .invalidateQueries({
            queryKey: [
              'assetAccounts',
            ],
          });
      },
    });
  };
```

Queryとして自動実行しない。

---

### 24.16 登録処理

概念例：

```ts
const createMutation =
  useCreateAssetAccount();

const handleSubmit = (
  request: CreateAssetAccountRequest,
): void => {
  createMutation.mutate(
    request,
  );
};
```

登録ボタン押下時にACC-002を実行する。

---

### 24.17 二重送信防止

Mutation実行中は、登録ボタンを非活性化する。

概念例：

```tsx
<button
  type="submit"
  disabled={
    createMutation.isPending
  }
>
  {createMutation.isPending
    ? '登録中...'
    : '登録'}
</button>
```

これにより、通常操作による短時間の連続送信を防止する。

ただし、フロントエンド側の二重送信防止をサーバー側の重複防止の代替とはしない。

---

### 24.18 登録成功時

ACC-002成功時は、

```text
201 Created
```

とともに、登録した資産口座が返却される。

概念例：

```json
{
  "data": {
    "id": "10",
    "name": "証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

フロントエンドでは、正常レスポンスを確認後に登録成功として扱う。

---

### 24.19 成功メッセージ

正常終了後は、必要に応じて

```text
資産口座を登録しました。
```

などの完了メッセージを表示する。

---

### 24.20 成功後のQuery Cache

ACC-002成功後は、資産口座一覧の内容が変化するため、ACC-001に対応するQuery Cacheを無効化する。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey: [
    'assetAccounts',
  ],
});
```

Query Keyの正式な定義は、フロントエンド共通設計に従う。

---

### 24.21 Query Keyを共通化する場合

Query Keyを各コンポーネントへ直接記述せず、共通定義として管理してよい。

概念例：

```ts
export const assetAccountKeys = {
  all: [
    'assetAccounts',
  ] as const,

  detail: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
    ] as const,
};
```

ACC-002成功後は、

```ts
queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.all,
});
```

のように使用する。

---

### 24.22 登録成功後の画面遷移

登録成功後は、基本的に資産口座一覧画面へ戻る。

概念的には、

```text
ACC-002成功
    ↓
Query Cache無効化
    ↓
成功メッセージ
    ↓
資産口座一覧へ遷移
```

とする。

画面設計によっては、登録した資産口座の詳細画面へ遷移してもよい。

その場合は、正常レスポンスの

```text
data.id
```

を使用する。

---

### 24.23 HOLDING登録後

```text
balanceRecordingUnit = HOLDING
```

で登録した場合でも、ACC-002成功直後に保有商品を自動登録しない。

画面設計上、続けて保有商品を登録させたい場合は、

```text
資産口座登録成功
    ↓
「保有商品を登録する」
導線を表示
    ↓
HLD-002
```

のように、別の業務操作として扱う。

---

### 24.24 ACCOUNT登録後

```text
balanceRecordingUnit = ACCOUNT
```

で登録した場合も、月末残高をACC-002から自動登録しない。

月末資産データの登録は、月末資産管理APIの責務とする。

---

### 24.25 VALIDATION_ERROR

Laravel側から

```text
VALIDATION_ERROR
```

が返却された場合は、`details`の`field`を利用して対応するフォーム項目へエラーを表示する。

概念例：

```text
name
    → 資産口座名

assetType
    → 資産種別

balanceRecordingUnit
    → 残高記録単位

startYearMonth
    → 利用開始年月

isAvailable
    → 利用可能資産区分
```

---

### 24.26 フィールドエラー

概念的には、以下のような型として扱える。

```ts
export type ApiValidationError = {
  field: string;
  reason: string;
  message: string;
};
```

例えば、

```ts
const nameError =
  error.details?.find(
    (detail) =>
      detail.field === 'name',
  );
```

として、資産口座名入力欄の近くへエラーを表示する。

---

### 24.27 ASSET_ACCOUNT_NAME_ALREADY_EXISTS

同一利用者内に同名資産口座が存在する場合は、

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となる。

フロントエンドでは、例えば

```text
同じ名前の資産口座が
すでに登録されています。
```

と表示する。

このエラーは、`name`に関連する業務エラーとして資産口座名入力欄の近くへ表示してもよい。

---

### 24.28 論理削除済み資産口座との重複

論理削除済みの同名資産口座が存在する場合も、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となる。

フロントエンドでは、論理削除済みかどうかを推測してメッセージを分岐しない。

サーバーから返却された業務エラーとして共通の重複メッセージを表示する。

---

### 24.29 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-002のフォーム内で独自の復旧処理を実装しない。

---

### 24.30 INVALID_USER_ID

```text
INVALID_USER_ID
```

についても、API共通の利用者コンテキストエラーとして扱う。

必要に応じて、現在の利用者選択状態を再確認する。

---

### 24.31 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択されている利用者が有効ではない状態として扱う。

ACC-002固有のフォーム入力エラーとして表示しない。

---

### 24.32 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
資産口座を登録できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

登録成功扱いとして一覧へ遷移してはならない。

---

### 24.33 通信結果が不明な場合

通信断などにより、クライアント側でACC-002の成功・失敗を判断できない場合がある。

ACC-002はIdempotency-Keyを使用しないため、無条件に同じリクエストを自動再送しない。

最初のリクエストがサーバー側で成功していた場合、再送すると

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となる可能性がある。

Phase1では、必要に応じてACC-001を再取得して登録状態を確認する。

---

### 24.34 自動Retry

ACC-002のMutationでは、原則として自動Retryを行わない。

概念例：

```ts
useMutation({
  mutationFn:
    createAssetAccount,

  retry: false,
});
```

登録APIであり、通信結果が不明な状態での自動再送によって利用者が意図しない再登録処理が発生することを避けるためである。

---

### 24.35 Optimistic Update

Phase1では、ACC-002に対するOptimistic Updateは必須としない。

資産口座登録は高頻度操作ではなく、以下のサーバー側処理を伴う。

```text
資産口座名重複確認
    ↓
asset_accounts登録
    ↓
初期利用可能資産設定登録
    ↓
transaction commit
```

そのため、

```text
API成功
    ↓
Query Cache無効化
    ↓
ACC-001再取得
```

という単純な方式を基本とする。

---

### 24.36 サーバー状態を正とする

登録成功後に、フォーム入力値だけを使用して資産口座一覧へ新規項目を追加することを必須としない。

サーバー側では、以下が行われる。

* ID採番
* Enum変換
* 初期利用可能資産設定登録
* 利用状態決定

そのため、Phase1ではACC-001を再取得してサーバー状態へ同期する方式を基本とする。

---

### 24.37 isEnabled

ACC-002成功時の

```text
isEnabled
```

は、常に`true`となる。

ただし、フロントエンドから

```ts
isEnabled: true
```

をリクエストへ追加しない。

利用状態はサーバー側で決定する。

---

### 24.38 DB内部項目をフロントエンド型へ持ち込まない

ACC-002のRequestおよびResponse型へ、以下のようなDB内部項目を追加しない。

```text
user_id
asset_type
balance_recording_unit
deleted_at
created_at
updated_at
asset_account_id
end_year_month
```

API契約として定義されたcamelCase項目だけを使用する。

---

### 24.39 API型と画面Stateを分離する

画面の都合によってフォームStateがAPI Request型と完全には一致しない場合がある。

例えば、

```ts
type AssetAccountFormValues = {
  name: string;
  assetType: AssetType | '';
  balanceRecordingUnit:
    | BalanceRecordingUnit
    | '';
  startYearMonth: string;
  isAvailable: boolean;
};
```

では、未選択状態として空文字を持つことができる。

一方、API Requestでは、

```ts
type CreateAssetAccountRequest = {
  name: string;
  assetType: AssetType;
  balanceRecordingUnit:
    BalanceRecordingUnit;
  startYearMonth: string;
  isAvailable: boolean;
};
```

として、有効な値だけを許可する。

画面入力途中の状態とAPI契約を同じ型へ無理に統合しない。

---

### 24.40 コンポーネントの責務

資産口座登録画面では、以下の責務を分離する。

```text
Page
    ↓
登録画面全体の制御

Form
    ↓
入力UI・入力エラー表示

Mutation Hook
    ↓
ACC-002実行・成功失敗処理

API Client
    ↓
HTTP通信

Type
    ↓
API契約
```

1つのReactコンポーネントへ以下をすべて直接記述しない。

* フォーム管理
* HTTP通信
* エラーコード判定
* Query Cache操作
* 画面遷移

---

### 24.41 概念的なディレクトリ構成

Phase1では、例えば以下のように機能単位で整理できる。

```text
features/
└── asset-accounts/
    ├── api/
    │   └── createAssetAccount.ts
    ├── components/
    │   └── AssetAccountForm.tsx
    ├── hooks/
    │   └── useCreateAssetAccount.ts
    ├── types/
    │   └── assetAccount.ts
    └── pages/
        └── CreateAssetAccountPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

### 24.42 フロントエンドで行わないこと

ACC-002のReact・TypeScript実装では、以下をフロントエンドの責務としない。

* 利用者境界の最終保証
* 資産口座名一意性の最終保証
* 論理削除済み資産口座を含む重複判定
* DB保存用コードへの変換
* 初期利用可能資産設定の開始年月決定
* 初期利用可能資産設定の終了年月決定
* `asset_account_id`の決定
* transaction制御
* `isEnabled`の決定

フロントエンドは、

```text
利用者入力
    ↓
API契約に沿ったRequest生成
    ↓
ACC-002実行
    ↓
結果表示
```

に責務を限定する。
:::

---

## 25. 設計上の補足

### 25.1 資産口座登録と初期利用可能資産設定を同時登録する理由

ACC-002では、資産口座を登録すると同時に、

```text
asset_account_available_settings
```

へ初期利用可能資産設定を1件登録する。

これは、資産口座が存在するにもかかわらず、利用可能資産区分を判定できない状態を作らないためである。

概念的には、

```text
asset_accounts
だけ存在
    ↓
対象年月の
利用可能資産区分が不明
```

という状態を避ける。

そのため、資産口座登録と初期利用可能資産設定登録を1つの業務操作として扱う。

---

### 25.2 `isAvailable`を`asset_accounts`へ持たせない理由

利用可能資産区分は、資産口座の固定属性ではなく、対象年月によって変化する可能性がある。

例えば、

```text
2026-08～
利用可能

2027-01～
利用不可
```

のように、時系列で状態が変化し得る。

そのため、

```text
asset_accounts.is_available
```

のような単一フラグでは管理せず、

```text
asset_account_available_settings
```

によって履歴として管理する。

ACC-002では、その履歴の初回レコードを登録する。

---

### 25.3 初期設定の開始年月を資産口座の利用開始年月と一致させる理由

初期利用可能資産設定の

```text
start_year_month
```

は、資産口座の

```text
asset_accounts.start_year_month
```

と同じ年月とする。

これにより、

```text
資産口座の利用開始前から
利用可能資産設定が存在する
```

または、

```text
資産口座利用開始後しばらく
利用可能資産設定が存在しない
```

という不整合を防止する。

初期設定の開始年月をクライアントから別途入力させないことで、整合性ルールをバックエンドへ集約する。

---

### 25.4 初期設定の`end_year_month`をNULLとする理由

ACC-002で登録する利用可能資産設定は、登録時点における最新の設定である。

そのため、

```text
end_year_month = NULL
```

とする。

将来、利用可能資産区分が変更された場合に、現在の設定の終了年月を確定し、新しい設定履歴を追加する。

これにより、利用可能資産区分を期間履歴として管理できる。

---

### 25.5 `userId`をリクエストボディへ含めない理由

ACC-002では、操作対象利用者を

```text
X-User-Id
```

から特定する。

リクエストボディにも

```text
userId
```

を持たせると、

```text
X-User-Id = 1

Request.userId = 2
```

のような矛盾した入力を許容することになる。

そのため、利用者IDの入力経路を`X-User-Id`へ統一する。

---

### 25.6 同一利用者内で資産口座名を一意とする理由

資産口座名は、画面表示やCSVなどで利用者が資産口座を識別するために使用する。

同一利用者内で同名資産口座を許可すると、

```text
証券口座
証券口座
```

のように、どちらを指しているのか判別しにくくなる。

そのため、同一利用者内では資産口座名を一意とする。

一方、異なる利用者間では同名を許可する。

---

### 25.7 論理削除済み資産口座も重複対象とする理由

Phase1では、論理削除済み資産口座と同じ名称の再登録を許可しない。

これは、過去データ上の

```text
資産口座名
```

と現在の新しい資産口座名が同一になることで、履歴参照やCSV上の識別が曖昧になることを避けるためである。

そのため、重複確認では

```text
withTrashed()
```

を使用し、論理削除済み資産口座も対象に含める。

---

### 25.8 DBのUNIQUE制約も使用する理由

アプリケーション側で登録前に重複確認を行っても、同時実行では競合を完全には防止できない。

例えば、

```text
Request A
重複なし

Request B
重複なし

Request A
INSERT

Request B
INSERT
```

となる可能性がある。

そのため、

```text
user_id
+
name
```

のUNIQUE制約をデータベース側にも設定する。

アプリケーション側の事前確認は、利用者へ分かりやすい業務エラーを返すために使用し、DB制約は最終防衛線として使用する。

---

### 25.9 `lockForUpdate()`を使用しない理由

ACC-002は、存在しない資産口座を新規登録する処理である。

登録前にはロック対象となる資産口座行が存在しないため、

```text
lockForUpdate()
```

だけでは同名新規登録の競合を防止できない。

そのため、Phase1では

```text
事前重複確認
+
UNIQUE制約
```

によって重複登録を防止する。

---

### 25.10 利用者行をロックしない理由

同一利用者の資産口座登録を直列化するために、

```text
users
```

をロックする方式は採用しない。

利用者行をロックすると、資産口座登録とは直接関係のない同一利用者の処理まで待機させる可能性がある。

ACC-002では、必要な一意性だけをDB制約によって保証する。

---

### 25.11 資産種別を文字列でAPI公開する理由

データベースでは、`asset_type`を`smallint`などの内部コードで保持する場合がある。

ただし、クライアントへ

```text
1
2
3
```

のような内部コードを公開すると、意味が分かりにくく、DB実装にも依存する。

そのため、APIでは

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

のような意味のある文字列を使用する。

---

### 25.12 残高記録単位を文字列でAPI公開する理由

`balance_recording_unit`も、DB内部コードではなく、

```text
ACCOUNT
HOLDING
```

としてAPI公開する。

これにより、フロントエンドからも業務上の意味を理解しやすくする。

---

### 25.13 `startYearMonth`を年月単位で扱う理由

資産口座の管理開始は、日単位ではなく月末資産管理の対象年月単位で意味を持つ。

そのため、

```text
YYYY-MM-DD
```

ではなく、

```text
YYYY-MM
```

で管理する。

不要な日付精度を持ち込まない。

---

### 25.14 `balanceRecordingUnit = HOLDING`でも保有商品を同時登録しない理由

資産口座登録と保有商品登録は、異なる業務操作である。

ACC-002で保有商品まで登録すると、

* 資産口座情報
* 初期利用可能資産設定
* 複数の保有商品

を1回のAPIで扱うことになり、責務が大きくなる。

そのため、

```text
ACC-002
資産口座登録

HLD-002
保有商品登録
```

として分離する。

---

### 25.15 `balanceRecordingUnit = ACCOUNT`でも月末残高を登録しない理由

資産口座登録と月末資産残高登録は、異なるライフサイクルを持つ。

資産口座を登録した時点では、必ずしも月末残高を登録する必要はない。

そのため、ACC-002では資産口座のマスタ情報だけを登録し、月末資産残高は月末資産管理APIで扱う。

---

### 25.16 2テーブル登録を同一トランザクションにする理由

ACC-002では、

```text
asset_accounts
```

と

```text
asset_account_available_settings
```

の両方が揃って初めて正常な資産口座として扱える。

どちらか一方だけが登録される状態を防ぐため、同一トランザクションで登録する。

---

### 25.17 Repositoryごとにトランザクションを持たせない理由

トランザクション境界は、

```text
資産口座登録
+
初期利用可能資産設定登録
```

というユースケース単位で決まる。

Repository単位でトランザクションを開始すると、

```text
AssetAccountRepository
    → commit

AvailableSettingRepository
    → 失敗
```

のように、ユースケース全体の原子性を保証しにくくなる。

そのため、UseCaseでトランザクション境界を管理する。

---

### 25.18 QueryとRepositoryを分ける理由

ACC-002では、

```text
Query
    → 既存データ確認

Repository
    → 新規登録
```

と責務を分離する。

これにより、参照条件と更新処理を明確に分けられる。

また、他のACC系APIでも同じ責務分離方針を再利用しやすくなる。

---

### 25.19 Input DTOを使用してよい理由

ACC-002では、UseCaseへ渡す入力項目が複数存在する。

```text
userId
name
assetType
balanceRecordingUnit
startYearMonth
isAvailable
```

これらを個別引数として増やし続けるより、Input DTOへまとめることで、UseCaseの入力契約を明確にできる。

ただし、Phase1でかえって複雑になる場合は、過剰にDTOを増やさない。

---

### 25.20 Enumを使用する理由

`assetType`や`balanceRecordingUnit`を自由文字列として扱うと、

```text
SECURITIES
securities
Security
```

などの不整合が発生しやすい。

PHP Enumを利用することで、許可された値だけをUseCase以降で扱えるようにする。

---

### 25.21 Mass Assignmentを避ける理由

ACC-002では、クライアントから送信された入力値をそのままModelへ設定しない。

例えば、

```php
AssetAccount::create(
    $request->all(),
);
```

とすると、将来的なRequest変更によって

```text
user_id
deleted_at
```

などの意図しない項目まで更新対象へ含まれるリスクがある。

登録項目を明示的に設定することで、更新可能範囲を明確にする。

---

### 25.22 `isEnabled`をリクエストで受け取らない理由

新規登録された資産口座は、必ず利用中状態から開始する。

そのため、クライアントに

```text
isEnabled
```

を指定させる必要はない。

サーバー側で

```text
deleted_at = NULL
```

となることで、利用中状態を表現する。

---

### 25.23 `isEnabled`をレスポンスへ返す理由

新規登録直後の利用状態をクライアントが明示的に確認できるよう、

```text
isEnabled = true
```

を返却する。

一方で、

```text
deletedAt
```

のようなDB実装詳細はAPIへ公開しない。

---

### 25.24 `isAvailable`を成功レスポンスへ返す理由

ACC-002では、資産口座だけでなく初期利用可能資産設定も登録する。

利用者が指定した

```text
isAvailable
```

が正しく反映されたことを登録結果として確認できるよう、成功レスポンスへ返却する。

---

### 25.25 初期設定IDを返さない理由

ACC-002の主目的は、資産口座登録である。

初期利用可能資産設定の

```text
id
```

は、内部履歴管理用のIDであり、フロントエンドが登録直後に使用する必要はない。

そのため、

```text
assetAccountAvailableSettingId
```

は返却しない。

---

### 25.26 `endYearMonth`を返さない理由

ACC-002で作成する初期利用可能資産設定の

```text
end_year_month
```

は必ず`NULL`である。

登録結果として固定的な内部状態を返却する必要はないため、レスポンスには含めない。

---

### 25.27 Optimistic Updateを必須としない理由

ACC-002では、

* 同名重複確認
* 資産口座登録
* 初期利用可能資産設定登録

がサーバー側で実行される。

クライアントだけで成功を先取りすると、失敗時のロールバック処理が複雑になる。

そのため、Phase1では

```text
API成功
    ↓
Query invalidate
    ↓
ACC-001再取得
```

を基本とする。

---

### 25.28 Mutationの自動Retryを行わない理由

ACC-002は、新規登録APIである。

ネットワークエラー時に自動Retryすると、最初のリクエストがサーバー側で成功していた場合に同じ登録処理が再送される可能性がある。

Phase1ではIdempotency-Keyを採用しないため、Mutationの自動Retryを基本的に無効化する。

---

### 25.29 Idempotency-Keyを採用しない理由

ACC-002では、同一利用者内の同名資産口座登録を

```text
事前重複確認
+
UNIQUE制約
```

で防止する。

Idempotency-Keyを導入すると、

* Key保存
* 有効期限管理
* 同一レスポンスの再現
* 利用者との関連付け

などの追加設計が必要になる。

Phase1では、必要性に対して複雑性が大きいため採用しない。

---

### 25.30 サーバー状態を正とする理由

ACC-002成功後の最終状態は、PostgreSQLに保存されたデータを正とする。

React側のフォーム入力値だけでは、

* 採番されたID
* 実際に登録されたEnum値
* 初期利用可能資産設定
* 利用状態

を完全には保証できない。

そのため、一覧表示ではACC-001を再取得して最新状態へ同期する。

---

### 25.31 ACC-001との整合性

ACC-002で登録した資産口座は、ACC-001の一覧取得対象となる。

ACC-002成功レスポンスとACC-001の項目定義は、可能な限り同じ意味の項目について同じAPI表現を使用する。

例えば、

```text
assetType
balanceRecordingUnit
startYearMonth
isEnabled
```

について、APIごとに異なる表現を使用しない。

---

### 25.32 ACC-003との整合性

ACC-002成功後、返却された

```text
id
```

を使用してACC-003 資産口座詳細取得を実行できる。

ACC-002で登録した値が、ACC-003でも同じ意味・形式で取得できることを前提とする。

---

### 25.33 ACC-006との責務分離

ACC-002では、利用可能資産設定の初期値だけを登録する。

その後の利用可能資産区分変更は、ACC-006で扱う。

概念的には、

```text
ACC-002
    ↓
初期設定作成

ACC-006
    ↓
設定履歴変更
```

とする。

ACC-002へ将来期間の設定変更まで持たせない。

---

### 25.34 Phase1では設計を広げすぎない

ACC-002では、以下のような機能は追加しない。

* 複数資産口座の一括登録
* 保有商品の同時登録
* 月末残高の同時登録
* CSVによる資産口座登録
* 将来開始予定の利用可能設定
* 複数期間の利用可能設定同時登録
* Idempotency-Key
* 利用者行ロック

Phase1では、

```text
資産口座1件
+
初期利用可能資産設定1件
```

を安全に登録することへ責務を限定する。

---

## 26. 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)


---

# ACC-003 資産口座詳細取得

## 1. 概要

操作対象となる利用者に帰属する指定された資産口座の詳細情報を取得する。

資産口座の基本情報に加えて、現在の利用可能資産区分を返却する。

主に以下の情報を取得する。

* 資産口座ID
* 資産口座名
* 資産種別
* 残高記録単位
* 利用開始年月
* 現在の利用可能資産区分
* 利用状態

概念的には、以下の情報を組み合わせてレスポンスを生成する。

```text
asset_accounts
    +
現在有効な
asset_account_available_settings
    ↓
資産口座詳細
```

本APIは参照専用であり、資産口座、利用可能資産設定、月末資産データなどの業務データを更新しない。

---

## 2. ユースケース

利用者は、資産口座一覧などから特定の資産口座を選択し、詳細情報を確認する。

主に以下の用途で使用する。

* 資産口座の登録内容を確認する
* 資産口座編集画面の初期値を取得する
* 資産種別を確認する
* 残高記録単位を確認する
* 利用開始年月を確認する
* 現在の利用可能資産区分を確認する
* 現在の利用状態を確認する

概念的な画面利用は、以下とする。

```text
ACC-001
資産口座一覧取得
    ↓
利用者が資産口座を選択
    ↓
ACC-003
資産口座詳細取得
    ↓
詳細表示
または
編集画面初期表示
```

---

## 3. エンドポイント

```http
GET /api/v1/asset-accounts/{assetAccountId}
```

`assetAccountId`には、取得対象となる資産口座IDを指定する。

---

## 4. HTTPメソッド

```text
GET
```

本APIは、指定された資産口座の詳細情報を取得する参照APIであるため、`GET`を使用する。

本APIの実行によって、以下の業務データを更新しない。

* `asset_accounts`
* `asset_account_available_settings`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`

---

## 5. 利用者コンテキスト

本APIは利用者依存APIのため、`X-User-Id`を必須とする。

操作対象となる利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者コンテキストの特定、`X-User-Id`の検証および利用者境界については、[API共通方針](../api-common-policy.md)に従う。

指定された`assetAccountId`に対応する資産口座が操作対象利用者に帰属する場合のみ、詳細情報を返却する。

概念的には、以下の条件を満たす必要がある。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID
```

---

### 5.1 利用者境界

ACC-003では、`assetAccountId`だけを条件として資産口座を取得してはならない。

以下のような取得は基本としない。

```text
asset_accounts.id
    = assetAccountId
```

操作対象利用者を含め、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID
```

として取得する。

これにより、他利用者の資産口座を参照できないようにする。

---

### 5.2 他利用者の資産口座

指定された`assetAccountId`が他利用者に帰属する場合は、対象が存在しないものとして扱う。

概念的には、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

とする。

以下のような専用エラーは返却しない。

```text
FORBIDDEN
OTHER_USER_ASSET_ACCOUNT
```

これにより、他利用者の資産口座が存在するかどうかを外部から推測しにくくする。

---

### 5.3 論理削除済み資産口座

ACC-003は、通常利用中の資産口座詳細を取得するAPIとする。

そのため、

```text
asset_accounts.deleted_at IS NOT NULL
```

の資産口座は通常の取得対象に含めない。

論理削除済み資産口座が指定された場合は、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 5.4 現在の利用可能資産区分

ACC-003では、資産口座の詳細情報とあわせて現在の利用可能資産区分を返却する。

利用可能資産区分は、

```text
asset_accounts
```

の固定カラムではなく、

```text
asset_account_available_settings
```

の履歴から取得する。

概念的には、現在年月を基準として

```text
start_year_month
    <= 現在年月

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= 現在年月
)
```

を満たす設定を取得する。

---

### 5.5 利用可能資産設定の利用者境界

`asset_account_available_settings`には直接`user_id`を持たない。

そのため、利用可能資産設定は取得対象の

```text
asset_accounts.id
```

を通して利用者境界を保証する。

概念的には、

```text
操作対象利用者
    ↓
asset_accounts.user_id
    ↓
asset_accounts.id
    ↓
asset_account_available_settings.asset_account_id
```

となる。

他利用者の資産口座に属する利用可能資産設定を取得してはならない。

---

### 5.6 利用可能資産設定が存在しない場合

ACC-002 資産口座登録では、資産口座登録時に初期利用可能資産設定を必ず1件登録する設計とする。

そのため、通常状態では現在の利用可能資産設定が取得できることを前提とする。

もし、データ不整合などにより対象年月に有効な利用可能資産設定が存在しない場合は、任意のデフォルト値へ補完しない。

例えば、

```text
isAvailable = false
```

を暗黙的に返却してはならない。

具体的な異常時の扱いは、後続の「エラーレスポンス」および「Laravel実装方針」で定義する。

---

### 5.7 X-User-Idが不正な場合

以下の場合は、資産口座検索へ進まない。

* `X-User-Id`が指定されていない
* `X-User-Id`の形式が不正
* 指定された利用者が存在しない
* 指定された利用者が論理削除済み

利用者コンテキストに関するエラーコードおよびHTTPステータスは、API共通方針に従う。

---

### 5.8 1リクエスト1利用者

1回のACC-003リクエストでは、`X-User-Id`で指定された1利用者の資産口座だけを扱う。

以下のように、URLやクエリパラメータから別の利用者IDを指定する方式は採用しない。

```text
/api/v1/users/{userId}/asset-accounts/{assetAccountId}
```

または、

```text
?userId=1
```

利用者の指定経路は、`X-User-Id`へ統一する。

---

## 6. パスパラメータ

本APIでは、取得対象となる資産口座を指定するために、以下のパスパラメータを使用する。

| パラメータ            | 型      |  必須 | 説明            |
| ---------------- | ------ | :-: | ------------- |
| `assetAccountId` | string |  ○  | 取得対象となる資産口座ID |

エンドポイントは、以下とする。

```http
GET /api/v1/asset-accounts/{assetAccountId}
```

---

### 6.1 assetAccountId

`assetAccountId`には、取得対象となる資産口座IDを指定する。

例：

```http
GET /api/v1/asset-accounts/10
```

APIでは主キーを文字列として扱うが、値としては正の整数形式を前提とする。

正常例：

```text
1
10
123
```

不正例：

```text
0
-1
abc
1.5
```

---

### 6.2 利用者境界

`assetAccountId`だけでは取得対象を確定しない。

実際の資産口座取得では、操作対象利用者IDと組み合わせて検索する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

他利用者に属する資産口座IDが指定された場合は、対象が存在しないものとして扱う。

---

## 7. クエリパラメータ

なし。

ACC-003では、指定された資産口座の現在の詳細情報を取得するため、検索条件や表示切替用のクエリパラメータを使用しない。

以下のようなクエリパラメータは受け付けない。

* `userId`
* `targetYearMonth`
* `includeDeleted`
* `includeSettings`
* `isEnabled`

利用者は`X-User-Id`から特定し、取得対象資産口座は`assetAccountId`から特定する。

---

## 8. リクエストヘッダー

以下のリクエストヘッダーを使用する。

| ヘッダー名       |  必須 | 説明                      |
| ----------- | :-: | ----------------------- |
| `X-User-Id` |  ○  | 操作対象となる利用者ID            |
| `Accept`    |  ○  | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/asset-accounts/10
Accept: application/json
X-User-Id: 1
```

本APIではリクエストボディを使用しないため、

```http
Content-Type: application/json
```

は必須としない。

---

### 8.1 X-User-Id

`X-User-Id`は、API共通方針に従って検証する。

本APIの処理開始前に、共通Middlewareで以下を確認する。

* ヘッダーが指定されていること
* IDが共通方針で定めた形式であること
* 利用者が存在すること
* 利用者が論理削除されていないこと

検証済みの利用者IDを利用者コンテキストへ保持し、Action以降で使用する。

---

## 9. リクエストボディ

なし。

ACC-003は参照専用のGET APIであるため、リクエストボディを使用しない。

以下のような情報をリクエストボディから受け付けない。

* `userId`
* `assetAccountId`
* `targetYearMonth`
* `isAvailable`
* `isEnabled`

取得対象は、

```text
X-User-Id
+
assetAccountId
```

によって決定する。

---

## 10. リクエスト項目

本APIでは、リクエストボディ項目は存在しない。

クライアントが指定する業務上の入力値は、以下のみとする。

```text
パスパラメータ
    assetAccountId

リクエストヘッダー
    X-User-Id
```

---

## 11. バリデーション

ACC-003では、主に以下を検証する。

```text
X-User-Id
assetAccountId
```

資産口座の存在確認や利用者境界確認については、単純な入力形式検証ではなく、業務データ取得時に確認する。

---

### 11.1 X-User-Id

`X-User-Id`について、以下を検証する。

* 必須であること
* 共通ID形式に一致すること
* 正の整数として扱えること
* 指定された利用者が存在すること
* 指定された利用者が論理削除されていないこと

不正な場合は、資産口座検索へ進まない。

---

### 11.2 assetAccountId 必須

`assetAccountId`は、パスパラメータとして必須とする。

エンドポイント上、`assetAccountId`が存在しない場合は、

```http
GET /api/v1/asset-accounts
```

となり、ACC-001 資産口座一覧取得として扱われる。

ACC-003では、必ず

```http
GET /api/v1/asset-accounts/{assetAccountId}
```

を使用する。

---

### 11.3 assetAccountId 形式

`assetAccountId`は、正の整数形式であることを必須とする。

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
```

形式不正の場合は、資産口座検索へ進まない。

---

### 11.4 assetAccountIdの型変換

Laravel内部では、パスパラメータを文字列として受け取る場合がある。

形式検証後に、必要に応じて整数へ変換して使用する。

不正な文字列をPHPの暗黙変換によって有効なIDとして扱ってはならない。

例えば、

```text
10abc
```

を

```text
10
```

として扱わない。

---

### 11.5 資産口座存在確認

指定された`assetAccountId`について、操作対象利用者に属する有効な資産口座が存在することを確認する。

概念的な条件は、以下とする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

該当する資産口座が存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 11.6 他利用者の資産口座

指定された`assetAccountId`が他利用者に属する場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下のような他利用者専用エラーは返却しない。

```text
FORBIDDEN
OTHER_USER_ASSET_ACCOUNT
```

これにより、他利用者のリソース存在有無を外部へ公開しない。

---

### 11.7 論理削除済み資産口座

対象資産口座が

```text
deleted_at IS NOT NULL
```

の場合は、通常の詳細取得対象に含めない。

この場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

ACC-003では、

```php
withTrashed()
```

を使用して論理削除済み資産口座を通常取得対象に含めない。

---

### 11.8 現在の利用可能資産設定

対象資産口座について、現在有効な

```text
asset_account_available_settings
```

を取得する。

現在年月を基準として、概念的には以下を満たす設定を対象とする。

```text
start_year_month
    <= 現在年月

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= 現在年月
)
```

現在有効な設定が複数件存在しないことを前提とする。

---

### 11.9 利用可能資産設定不存在

ACC-002では、資産口座登録時に初期利用可能資産設定を必ず登録する。

そのため、通常状態では現在有効な利用可能資産設定が1件取得できることを前提とする。

もし、データ不整合によって現在有効な設定が存在しない場合は、

```text
isAvailable = false
```

などのデフォルト値へ補完しない。

業務データ不整合として扱う。

具体的なエラーコードは、「エラーレスポンス」で定義する。

---

### 11.10 利用可能資産設定が複数存在する場合

同一資産口座について、現在年月に有効な利用可能資産設定が複数存在する状態は、正常な業務状態として扱わない。

例えば、

```text
設定A
2026-01 ～ NULL

設定B
2026-06 ～ NULL
```

のように、同時に複数設定が有効となる状態はデータ不整合である。

この場合も、任意の1件を選択してレスポンスを返してはならない。

具体的な異常時の扱いは、「エラーレスポンス」および「Laravel実装方針」で定義する。

---

### 11.11 targetYearMonthを受け付けない

ACC-003は、「現在の資産口座詳細」を取得するAPIとする。

そのため、

```text
targetYearMonth
```

を指定して過去時点の利用可能資産区分を取得する機能は持たせない。

過去年月時点の利用可能資産設定が必要な処理では、各業務API側で対象年月を基準として判定する。

---

### 11.12 isEnabledを入力させない

利用状態は、

```text
asset_accounts.deleted_at
```

からサーバー側で判定する。

クライアントから

```text
isEnabled
```

を指定して取得対象を切り替えることはできない。

ACC-003では、有効な資産口座だけを取得対象とする。

---

### 11.13 バリデーション失敗時

以下のいずれかに該当する場合は、正常レスポンスを返却しない。

* `X-User-Id`不正
* `assetAccountId`形式不正
* 利用者不存在
* 資産口座不存在
* 他利用者の資産口座
* 論理削除済み資産口座
* 利用可能資産設定の不整合

ACC-003は参照APIであるため、バリデーション失敗時にも業務データは更新しない。

---

## 12. 業務ルール

ACC-003では、操作対象利用者に帰属する指定された資産口座の詳細情報を取得する。

資産口座の基本情報に加えて、現在年月において有効な利用可能資産設定を取得し、現在の利用可能資産区分として返却する。

本APIは参照専用であり、業務データを更新しない。

---

### 12.1 操作対象利用者に帰属する資産口座だけを取得する

取得対象となる資産口座は、以下の条件をすべて満たすものとする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

`assetAccountId`だけを条件として資産口座を取得してはならない。

---

### 12.2 他利用者の資産口座を取得しない

指定された`assetAccountId`が他利用者に帰属する場合は、対象資産口座が存在しないものとして扱う。

概念的には、

```text
X-User-Id = 1

assetAccountId = 10

asset_accounts.id = 10
asset_accounts.user_id = 2
    ↓
取得対象外
    ↓
ASSET_ACCOUNT_NOT_FOUND
```

とする。

他利用者の資産口座が存在することを示す情報をレスポンスへ含めない。

---

### 12.3 論理削除済み資産口座を取得しない

ACC-003では、利用中の資産口座だけを詳細取得対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は取得対象外とする。

論理削除済み資産口座が指定された場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 12.4 資産口座の基本情報を取得する

取得対象資産口座から、主に以下の情報を取得する。

```text
id
name
asset_type
balance_recording_unit
start_year_month
deleted_at
```

`deleted_at`は、APIレスポンスへそのまま返却するためではなく、利用状態の判定に使用する。

---

### 12.5 現在の利用可能資産区分を取得する

利用可能資産区分は、

```text
asset_accounts
```

の固定属性として保持せず、

```text
asset_account_available_settings
```

から取得する。

ACC-003では、現在年月に有効な設定を取得し、

```text
isAvailable
```

として返却する。

---

### 12.6 現在年月

利用可能資産設定の判定に使用する現在年月は、サーバー側の現在日時から決定する。

概念的には、

```text
現在日
2026-08-11
    ↓
現在年月
2026-08
```

とする。

クライアントから現在年月を指定させない。

---

### 12.7 現在有効な利用可能資産設定

現在有効な利用可能資産設定は、概念的に以下の条件を満たすものとする。

```text
asset_account_id
    = 対象資産口座ID

AND

start_year_month
    <= 現在年月

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= 現在年月
)
```

この条件を満たす設定が1件存在することを正常状態とする。

---

### 12.8 利用可能資産設定の開始年月

以下の場合は、現在有効な設定として扱う。

```text
start_year_month
    = 現在年月
```

例えば、

```text
現在年月
    = 2026-08

start_year_month
    = 2026-08
```

の場合、当該設定は2026年8月から有効であるため、取得対象とする。

---

### 12.9 利用可能資産設定の終了年月

`end_year_month`が設定されている場合、その年月までは当該設定を有効とする。

例えば、

```text
start_year_month = 2026-01
end_year_month   = 2026-08
```

の場合、

```text
2026-08
```

では有効とする。

```text
2026-09
```

では有効としない。

---

### 12.10 end_year_monthがNULLの場合

```text
end_year_month IS NULL
```

の場合は、終了年月が設定されていない現在有効な設定として扱う。

概念的には、

```text
start_year_month = 2026-08
end_year_month   = NULL
    ↓
2026-08以降有効
```

となる。

---

### 12.11 利用可能資産設定が存在しない場合

ACC-002では、資産口座登録時に初期利用可能資産設定を必ず登録する。

そのため、利用開始済みの資産口座については、通常、現在有効な利用可能資産設定が1件存在することを前提とする。

現在有効な設定が存在しない場合は、データ不整合として扱う。

以下のようにデフォルト値へ補完しない。

```text
設定なし
    ↓
isAvailable = false
```

具体的なエラーコードは、「エラーレスポンス」で定義する。

---

### 12.12 利用可能資産設定が複数存在する場合

同一資産口座について、現在年月に有効な設定が複数存在する場合は、データ不整合として扱う。

例えば、

```text
設定A
start_year_month = 2026-01
end_year_month   = NULL

設定B
start_year_month = 2026-06
end_year_month   = NULL
```

の状態で現在年月が`2026-08`の場合、両方が有効となる。

この場合、以下のように任意の1件を選択してはならない。

```text
ORDER BY start_year_month DESC
LIMIT 1
```

設定期間の重複を隠蔽することになるためである。

データ不整合として扱う。

---

### 12.13 利用開始年月より前の扱い

資産口座には、

```text
start_year_month
```

としてLife Planner上での利用開始年月を保持する。

ACC-003は現在の資産口座詳細を確認するためのAPIであるため、登録済みで利用中の資産口座について基本情報自体は取得対象とする。

ただし、現在年月が`start_year_month`より前であり、現在有効な利用可能資産設定が存在しない場合に、`isAvailable`を任意の値へ補完しない。

利用開始前の資産口座をどの画面で表示するかについては、画面設計および各APIの対象年月ルールに従う。

---

### 12.14 isAvailable

レスポンスの

```text
isAvailable
```

には、現在有効な

```text
asset_account_available_settings.is_available
```

を使用する。

概念的には、

```text
asset_account_available_settings.is_available
    = true
        ↓
isAvailable = true
```

```text
asset_account_available_settings.is_available
    = false
        ↓
isAvailable = false
```

とする。

---

### 12.15 isEnabled

レスポンスの

```text
isEnabled
```

は、資産口座の利用状態を表す。

ACC-003では論理削除済み資産口座を取得対象としないため、正常レスポンスでは

```text
isEnabled = true
```

となる。

データベースの

```text
deleted_at
```

自体はレスポンスへ返却しない。

---

### 12.16 DB内部コードをそのまま返却しない

`asset_type`および`balance_recording_unit`をDB内部で数値コードとして保持している場合でも、その値をそのままレスポンスへ返却しない。

例えば、

```text
asset_type = 3
```

を、

```json
{
  "assetType": 3
}
```

として返却しない。

APIで定義した文字列表現へ変換する。

概念例：

```json
{
  "assetType": "SECURITIES"
}
```

同様に、`balance_recording_unit`も、

```text
ACCOUNT
HOLDING
```

のAPI表現へ変換する。

---

### 12.17 過去時点の利用可能資産区分を返却しない

ACC-003は、現在の資産口座詳細を取得するAPIとする。

そのため、過去年月を指定して

```text
2026-01時点では利用可能だったか
```

といった情報を取得する用途には使用しない。

過去時点の利用可能資産区分が必要な月末資産管理や目的達成判定では、各APIが持つ対象年月を基準として利用可能資産設定を判定する。

---

### 12.18 保有商品を取得しない

ACC-003では、`balanceRecordingUnit`が

```text
HOLDING
```

であっても、保有商品一覧を同時に返却しない。

保有商品は保有商品APIの責務とする。

概念的には、

```text
ACC-003
    → 資産口座詳細

HLD-001
    → 保有商品一覧
```

と責務を分離する。

---

### 12.19 月末資産データを取得しない

ACC-003では、以下の情報を取得しない。

* 月末資産スナップショット
* 月末資産残高
* 商品別月末評価額
* 最新残高
* 資産推移

これらは、月末資産APIおよび資産表示APIの責務とする。

---

### 12.20 業務データを更新しない

ACC-003は参照専用APIである。

以下のような更新処理を行わない。

```text
asset_accounts
    INSERT / UPDATE / DELETE

asset_account_available_settings
    INSERT / UPDATE / DELETE
```

詳細取得によって、`updated_at`などの業務データを変更しない。

---

## 13. 処理フロー

概念的な処理フローは、以下とする。

```text
リクエスト受付
    ↓
リクエストID生成
    ↓
X-User-Id検証
    ↓
操作対象利用者の特定
    ↓
assetAccountId形式検証
    ↓
操作対象利用者に帰属する
有効な資産口座を取得
    ↓
存在しない
    → ASSET_ACCOUNT_NOT_FOUND
    ↓
現在年月を取得
    ↓
現在有効な
利用可能資産設定を取得
    ↓
設定件数確認
    ↓
0件または複数件
    → データ不整合
    ↓
資産口座詳細DTO生成
    ↓
APIレスポンス用の形式へ変換
    ↓
正常レスポンス返却
```

---

### 13.1 利用者コンテキスト確認

共通Middlewareで`X-User-Id`を検証し、操作対象利用者を特定する。

利用者コンテキストが不正な場合は、資産口座検索へ進まない。

---

### 13.2 assetAccountId検証

パスパラメータの

```text
assetAccountId
```

が正の整数形式であることを確認する。

形式不正の場合は、データベース検索へ進まない。

---

### 13.3 資産口座取得

操作対象利用者IDと`assetAccountId`を使用して、資産口座を取得する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

該当する資産口座が存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

とする。

---

### 13.4 現在年月取得

サーバー側の現在日時から、利用可能資産設定判定用の現在年月を取得する。

概念的には、

```text
now()
    ↓
YYYY-MM
```

とする。

---

### 13.5 利用可能資産設定取得

取得した資産口座IDと現在年月を使用して、現在有効な利用可能資産設定を取得する。

概念的には、

```text
asset_account_id
    = assetAccountId

AND

start_year_month
    <= 現在年月

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= 現在年月
)
```

とする。

---

### 13.6 利用可能資産設定件数確認

取得結果が1件であることを確認する。

```text
1件
    ↓
正常処理継続
```

```text
0件
    ↓
データ不整合
```

```text
2件以上
    ↓
データ不整合
```

0件の場合に`false`へ補完せず、複数件の場合に任意の1件を選択しない。

---

### 13.7 詳細情報生成

資産口座と現在有効な利用可能資産設定から、APIレスポンスに必要な詳細情報を生成する。

概念的には、

```text
asset_accounts.id
    → id

asset_accounts.name
    → name

asset_accounts.asset_type
    → assetType

asset_accounts.balance_recording_unit
    → balanceRecordingUnit

asset_accounts.start_year_month
    → startYearMonth

asset_account_available_settings.is_available
    → isAvailable

asset_accounts.deleted_at
    → isEnabled
```

とする。

---

### 13.8 レスポンス変換

データベースカラムをそのまま返却せず、DTOおよびAPI Resourceを使用してAPIレスポンス形式へ変換する。

特に、

```text
asset_type
balance_recording_unit
```

は、APIで定義した文字列表現へ変換する。

---

## 14. トランザクション境界

ACC-003は、参照専用APIであり、業務データを更新しない。

そのため、Phase1では明示的なデータベーストランザクションを使用しない。

---

### 14.1 更新処理なし

ACC-003では、以下の処理を行わない。

```text
INSERT
UPDATE
DELETE
```

主な処理は、

```text
asset_accounts
    SELECT

asset_account_available_settings
    SELECT
```

のみである。

---

### 14.2 明示的なトランザクションを使用しない理由

ACC-003では、資産口座と利用可能資産設定を参照するが、両方を更新するわけではない。

Phase1では、詳細表示のための通常参照処理として扱い、

```php
DB::transaction(function () {
    // SELECT
});
```

のような明示的なトランザクションは使用しない。

---

### 14.3 参照間の整合性

資産口座取得後に、利用可能資産設定が変更される可能性はある。

ただし、ACC-003は現在状態を表示するための参照APIであり、目的達成判定や月末確定のように、複数データを厳密に同一時点で固定して業務判断する処理ではない。

そのため、Phase1ではRepeatable Readなどによるスナップショット固定を要求しない。

---

### 14.4 厳密な一貫性が必要な処理との区別

ACC-003の詳細表示と、以下のような業務処理は区別する。

```text
月末資産確定
目的達成判定
利用可能資産設定変更
```

これらでトランザクションや排他制御が必要な場合は、各API側で個別に定義する。

ACC-003の参照要件を理由として、不要なトランザクションを導入しない。

---

### 14.5 データ不整合時

現在有効な利用可能資産設定が

```text
0件
または
複数件
```

であることを検出した場合も、ACC-003自身はデータ修復を行わない。

例えば、

```text
設定なし
    ↓
初期設定を自動INSERT
```

や、

```text
設定重複
    ↓
古い設定を自動UPDATE
```

といった処理は行わない。

参照処理の中で業務データを暗黙的に変更せず、データ不整合としてエラーを返却する。

---

## 15. 排他制御

ACC-003は、参照専用APIであり、業務データを更新しない。

そのため、Phase1では明示的な排他制御を行わない。

以下のような行ロックは使用しない。

```php
lockForUpdate()
```

また、楽観ロック用の以下も使用しない。

```text
version
updated_at比較
ETag
If-Match
```

---

### 15.1 資産口座へのロック

ACC-003では、`asset_accounts`を参照するだけであるため、対象資産口座行をロックしない。

概念的には、以下とする。

```text
asset_accounts
    SELECT
```

---

### 15.2 利用可能資産設定へのロック

`asset_account_available_settings`についても、現在有効な設定を参照するだけであるため、明示的な行ロックを行わない。

概念的には、以下とする。

```text
asset_account_available_settings
    SELECT
```

---

### 15.3 同時更新との関係

ACC-003実行中に、別リクエストによって以下の処理が同時実行される可能性がある。

* 資産口座更新
* 利用可能資産設定変更
* 資産口座無効化

Phase1では、ACC-003を現在状態の参照APIとして扱うため、参照中の状態を長時間固定しない。

必要に応じて、次回のAPI取得時に最新状態へ更新されるものとする。

---

### 15.4 厳密な同一時点保証を行わない

ACC-003では、

```text
asset_accounts取得時点
```

と

```text
asset_account_available_settings取得時点
```

を厳密に同一瞬間へ固定することを要件としない。

本APIは業務判断を確定する処理ではなく、詳細表示用の参照APIであるためである。

---

### 15.5 データ不整合はロックで補正しない

現在有効な利用可能資産設定が、

```text
0件
または
複数件
```

存在する場合に、ACC-003で排他制御を取得して状態を修復しない。

不整合状態は、エラーとして検出する。

---

## 16. 成功レスポンス

指定された資産口座の詳細取得に成功した場合は、

```http
200 OK
```

を返却する。

正常時は、API共通方針に従った成功レスポンス形式を使用する。

概念例：

```json
{
  "data": {
    "id": "10",
    "name": "証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

---

### 16.1 レスポンス項目

正常時の`data`配下には、以下を返却する。

| 項目                     | 型       | NULL | 説明                     |
| ---------------------- | ------- | :--: | ---------------------- |
| `id`                   | string  |   ×  | 資産口座ID                 |
| `name`                 | string  |   ×  | 資産口座名                  |
| `assetType`            | string  |   ×  | 資産種別                   |
| `balanceRecordingUnit` | string  |   ×  | 残高記録単位                 |
| `isAvailable`          | boolean |   ×  | 現在の利用可能資産区分            |
| `startYearMonth`       | string  |   ×  | 利用開始年月。`YYYY-MM`形式     |
| `isEnabled`            | boolean |   ×  | 利用状態。正常レスポンスでは常に`true` |

---

### 16.2 id

対象資産口座のIDを文字列として返却する。

例：

```json
{
  "id": "10"
}
```

データベースでは`bigint`で保持していても、API共通方針に従って文字列として返却する。

---

### 16.3 name

対象資産口座の資産口座名を返却する。

例：

```json
{
  "name": "証券口座"
}
```

---

### 16.4 assetType

資産種別は、API用の文字列表現で返却する。

例：

```json
{
  "assetType": "SECURITIES"
}
```

DB内部の数値コードをそのまま返却しない。

---

### 16.5 balanceRecordingUnit

残高記録単位は、API用の文字列表現で返却する。

想定する値は、以下とする。

```text
ACCOUNT
HOLDING
```

例：

```json
{
  "balanceRecordingUnit": "HOLDING"
}
```

---

### 16.6 isAvailable

現在年月において有効な、

```text
asset_account_available_settings.is_available
```

を返却する。

例：

```json
{
  "isAvailable": true
}
```

`asset_accounts`の固定カラムとして保持されている値ではない。

---

### 16.7 startYearMonth

資産口座の利用開始年月を返却する。

形式は、

```text
YYYY-MM
```

とする。

例：

```json
{
  "startYearMonth": "2026-08"
}
```

---

### 16.8 isEnabled

ACC-003では、論理削除済み資産口座を取得対象としない。

そのため、正常レスポンスでは、

```json
{
  "isEnabled": true
}
```

となる。

`deleted_at`自体は返却しない。

---

### 16.9 返却しない情報

ACC-003では、以下の内部情報をレスポンスへ返却しない。

* `user_id`
* `deleted_at`
* `created_at`
* `updated_at`
* `asset_account_available_settings.id`
* `asset_account_available_settings.asset_account_id`
* `asset_account_available_settings.start_year_month`
* `asset_account_available_settings.end_year_month`
* DB内部の数値コード

また、以下の関連データも返却しない。

* 保有商品一覧
* 月末資産残高
* 商品別月末評価額
* 月末資産状況
* 資産推移

---

## 17. エラーレスポンス

異常時は、API共通方針に従った共通エラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_NOT_FOUND",
    "message": "指定された資産口座が存在しません。"
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

---

### 17.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

この場合、資産口座検索へ進まない。

---

### 17.2 INVALID_USER_ID

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

例えば、以下を不正とする。

```text
0
-1
abc
1.5
```

---

### 17.3 USER_NOT_FOUND

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

---

### 17.4 INVALID_ASSET_ACCOUNT_ID

`assetAccountId`の形式が不正な場合は、

```text
INVALID_ASSET_ACCOUNT_ID
```

を返却する。

例えば、

```text
0
-1
abc
1.5
1e3
```

などを不正とする。

形式不正の場合は、資産口座検索へ進まない。

---

### 17.5 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

* 指定された資産口座が存在しない
* 指定された資産口座が他利用者に属する
* 指定された資産口座が論理削除済み

他利用者の資産口座であることを示す専用エラーコードは返却しない。

---

### 17.6 他利用者の資産口座

例えば、

```text
User A
    assetAccountId = 10

User B
    X-User-Id = 2
```

の状態で、User Bが

```http
GET /api/v1/asset-accounts/10
```

を実行した場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

これにより、他利用者の資産口座存在有無を外部へ公開しない。

---

### 17.7 論理削除済み資産口座

対象資産口座が、

```text
deleted_at IS NOT NULL
```

の場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

論理削除済みであることを専用エラーとして公開しない。

---

### 17.8 利用可能資産設定不存在

対象資産口座について、現在有効な

```text
asset_account_available_settings
```

が存在しない場合は、正常レスポンスを返却しない。

ACC-002で初期設定が必ず作成される前提に対するデータ不整合として扱う。

概念的な独自エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

とする。

ただし、データ不整合系エラーコードを共通化する場合は、API共通方針の定義を優先する。

---

### 17.9 利用可能資産設定重複

現在年月に有効な利用可能資産設定が複数件存在する場合も、正常レスポンスを返却しない。

概念的な独自エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

とする。

任意の1件を選択してレスポンスを返してはならない。

---

### 17.10 データ不整合時に補完しない

利用可能資産設定の不整合時に、以下のような自動補完を行わない。

```text
設定なし
    ↓
isAvailable = false
```

または、

```text
複数設定
    ↓
最新1件を採用
```

データ不整合を隠蔽せず、明示的なエラーとして扱う。

---

### 17.11 INTERNAL_SERVER_ERROR

資産口座詳細取得処理で想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスへ、以下の内部情報を含めない。

* SQL
* PostgreSQL内部エラー
* テーブル名
* カラム名
* 制約名
* Laravel内部例外メッセージ
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 17.12 主なエラー一覧

ACC-003で想定する主なエラーは、以下とする。

| HTTPステータス                   | 独自エラーコード                                    | 発生条件                     |
| --------------------------- | ------------------------------------------- | ------------------------ |
| `400 Bad Request`           | `USER_CONTEXT_REQUIRED`                     | `X-User-Id`未指定           |
| `400 Bad Request`           | `INVALID_USER_ID`                           | `X-User-Id`形式不正          |
| `400 Bad Request`           | `INVALID_ASSET_ACCOUNT_ID`                  | `assetAccountId`形式不正     |
| `404 Not Found`             | `USER_NOT_FOUND`                            | 利用者不存在、または論理削除済み         |
| `404 Not Found`             | `ASSET_ACCOUNT_NOT_FOUND`                   | 資産口座不存在、他利用者所属、または論理削除済み |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND` | 現在有効な利用可能資産設定が存在しない      |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT`  | 現在有効な利用可能資産設定が複数存在する     |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR`                     | 想定外のサーバー内部エラー            |

利用可能資産設定の不存在・重複は、クライアント入力によるものではなく、サーバー側の業務データ不整合として扱う。

そのため、Phase1では、

```http
500 Internal Server Error
```

相当とする。

具体的なデータ不整合エラーの扱いをAPI共通方針で統一する場合は、その定義を優先する。

---

### 17.13 エラー時の副作用

ACC-003は参照専用APIであるため、いずれのエラーが発生した場合も業務データを更新しない。

特に、利用可能資産設定の不整合を検出しても、

```text
asset_account_available_settings
```

を自動修復しない。


---

## 18. HTTPステータス

ACC-003では、処理結果に応じて以下のHTTPステータスを返却する。

| HTTPステータス                   | 用途                                |
| --------------------------- | --------------------------------- |
| `200 OK`                    | 資産口座詳細取得成功                        |
| `400 Bad Request`           | 利用者コンテキストまたは`assetAccountId`の形式不正 |
| `404 Not Found`             | 利用者または資産口座が存在しない                  |
| `500 Internal Server Error` | 利用可能資産設定の不整合、または想定外のサーバー内部エラー     |

具体的な独自エラーコードは、「エラーレスポンス」およびAPI共通方針に従う。

---

### 18.1 200 OK

指定された資産口座が操作対象利用者に帰属し、現在有効な利用可能資産設定も正常に取得できた場合は、

```http
200 OK
```

を返却する。

概念的には、

```text
資産口座取得成功
+
現在有効な利用可能資産設定 1件
    ↓
200 OK
```

とする。

---

### 18.2 400 Bad Request

以下の場合は、

```http
400 Bad Request
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
INVALID_ASSET_ACCOUNT_ID
```

例えば、

* `X-User-Id`未指定
* `X-User-Id`形式不正
* `assetAccountId`が正の整数形式ではない

などを対象とする。

---

### 18.3 404 Not Found

以下の場合は、

```http
404 Not Found
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_NOT_FOUND
ASSET_ACCOUNT_NOT_FOUND
```

`ASSET_ACCOUNT_NOT_FOUND`には、以下を含む。

* 資産口座不存在
* 他利用者に属する資産口座
* 論理削除済み資産口座

他利用者に属することや論理削除済みであることを、個別のHTTPステータスで公開しない。

---

### 18.4 500 Internal Server Error

以下の場合は、

```http
500 Internal Server Error
```

として扱う。

主なエラーコードは、以下とする。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
INTERNAL_SERVER_ERROR
```

利用可能資産設定の不存在や重複は、クライアント入力によるものではなく、本来成立しているべき業務データの整合性が崩れている状態とする。

---

### 18.5 ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND

対象資産口座について、現在年月に有効な

```text
asset_account_available_settings
```

が存在しない場合は、

```http
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

とする。

ACC-002で初期利用可能資産設定を必ず作成する設計に対するデータ不整合として扱う。

---

### 18.6 ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT

現在年月に有効な利用可能資産設定が複数件存在する場合は、

```http
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

とする。

任意の設定を選択して`200 OK`を返却しない。

---

### 18.7 INTERNAL_SERVER_ERROR

想定外のサーバー内部エラーが発生した場合は、

```http
500 Internal Server Error
```

を返却する。

エラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

レスポンスへSQLやスタックトレースなどの内部情報を含めない。

---

## 19. 冪等性

ACC-003は、参照専用のGET APIであるため、冪等である。

同一の

```text
X-User-Id
+
assetAccountId
```

に対して、同じサーバー状態で複数回リクエストした場合、同じ結果を返却する。

---

### 19.1 業務データを変更しない

ACC-003を何度実行しても、以下の業務データを変更しない。

* `users`
* `asset_accounts`
* `asset_account_available_settings`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

詳細取得によって`updated_at`なども更新しない。

---

### 19.2 同一リクエストの再実行

例えば、

```http
GET /api/v1/asset-accounts/10
Accept: application/json
X-User-Id: 1
```

を、サーバー側の状態が変化していない間に複数回実行した場合は、同じ詳細情報を返却する。

---

### 19.3 サーバー状態が変更された場合

ACC-003自体は冪等であるが、別APIによって資産口座や利用可能資産設定が変更された場合は、後続のACC-003で返却内容が変化する。

例えば、

```text
ACC-003
isAvailable = true
    ↓
ACC-006
利用可能資産区分変更
    ↓
ACC-003
isAvailable = false
```

となり得る。

これは、ACC-003の冪等性を損なうものではない。

---

### 19.4 資産口座無効化後

ACC-003取得後に別APIによって資産口座が無効化された場合、同じ`assetAccountId`で再度ACC-003を実行すると、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となる。

これも、サーバー状態が変化した結果である。

---

### 19.5 Idempotency-Key

ACC-003はGETによる参照APIであるため、

```text
Idempotency-Key
```

を使用しない。

冪等性確保のための追加ヘッダーやサーバー側キー管理は不要とする。

---

## 20. キャッシュ

Phase1では、ACC-003専用のサーバー側アプリケーションキャッシュを使用しない。

資産口座詳細および現在の利用可能資産区分は、PostgreSQLの最新状態から取得する。

---

### 20.1 サーバー側キャッシュ

Phase1では、以下のようなACC-003専用キャッシュを導入しない。

```text
Redis
Laravel Cache
Application Memory Cache
```

資産口座詳細は単一資産口座と現在有効な利用可能資産設定を取得するだけであり、Phase1ではキャッシュ導入による複雑性を追加しない。

---

### 20.2 利用可能資産設定とキャッシュ

ACC-003では、現在年月における

```text
asset_account_available_settings
```

を取得する。

利用可能資産区分は、ACC-006などによって変更される可能性がある。

そのため、古いキャッシュによって

```text
isAvailable
```

を返却し続けないようにする必要がある。

Phase1ではサーバー側キャッシュを使用しないことで、常に最新DB状態を参照する。

---

### 20.3 React・TypeScript側のQuery Cache

フロントエンドでは、TanStack QueryなどのQuery Cacheを使用してよい。

概念的には、以下のようなQuery Keyを使用する。

```text
assetAccounts
+
assetAccountId
```

例えば、

```ts
[
  'assetAccounts',
  assetAccountId,
]
```

のようなキーで詳細取得結果を保持してよい。

具体的な実装は、「React・TypeScriptでの利用」で定義する。

---

### 20.4 ACC-004成功後

ACC-004 資産口座更新によって、対象資産口座の

* `name`
* `assetType`

などが変更された場合は、ACC-003のキャッシュが古くなる。

そのため、フロントエンドではACC-004成功後に対象資産口座の詳細Query Cacheを無効化する。

---

### 20.5 ACC-005成功後

ACC-005 資産口座無効化によって、対象資産口座が通常参照対象から除外された場合は、ACC-003の詳細キャッシュも無効化する。

無効化済み資産口座の古い詳細情報を画面上へ残し続けない。

---

### 20.6 ACC-006成功後

ACC-006 利用可能資産設定登録によって、現在の

```text
isAvailable
```

が変化する可能性がある。

そのため、ACC-006成功後は対象資産口座のACC-003詳細Query Cacheを無効化する。

概念的には、

```text
ACC-006成功
    ↓
ACC-003 Query invalidate
    ↓
ACC-003再取得
    ↓
最新isAvailableを表示
```

とする。

---

### 20.7 ACC-002成功後

ACC-002で新しい資産口座を登録した場合、その資産口座IDについてACC-003を初めて取得できるようになる。

ACC-002成功レスポンスをACC-003の正式なサーバーキャッシュとして扱わない。

必要な場合は、返却された

```text
id
```

を使用してACC-003を取得する。

---

### 20.8 HTTPキャッシュ

ACC-003はGET APIであるため、技術的にはHTTPキャッシュの対象にできる。

ただし、Phase1ではACC-003独自の

```text
ETag
Last-Modified
If-None-Match
If-Modified-Since
```

などは導入しない。

HTTPキャッシュヘッダーに関する共通方針がある場合は、API共通方針を優先する。

---

### 20.9 利用者境界とキャッシュキー

フロントエンドまたは将来のサーバーキャッシュでACC-003をキャッシュする場合は、`assetAccountId`だけでキャッシュを共有しない。

本APIは利用者依存APIであるため、概念的には

```text
userId
+
assetAccountId
```

によって利用者境界を維持する必要がある。

例えば、異なる利用者間で同じキャッシュエントリを誤って共有してはならない。

---

### 20.10 現在年月の変化

ACC-003の

```text
isAvailable
```

は、現在年月を基準として利用可能資産設定履歴から算出する。

そのため、月が変わった場合は、DB更新がなくても返却される`isAvailable`が変化する可能性がある。

例えば、

```text
設定A
2026-01 ～ 2026-08
is_available = true

設定B
2026-09 ～ NULL
is_available = false
```

の場合、

```text
2026-08
    → isAvailable = true

2026-09
    → isAvailable = false
```

となる。

将来的に長時間キャッシュを導入する場合は、この年月境界も考慮する。

Phase1ではサーバー側キャッシュを使用しないため、特別な失効処理は実装しない。

---

## 21. 関連テーブル

ACC-003では、指定された資産口座の詳細情報と、現在有効な利用可能資産設定を取得するため、以下のテーブルを使用する。

| テーブル                               | 用途                |  更新 |
| ---------------------------------- | ----------------- | :-: |
| `users`                            | 操作対象利用者の確認        |  ×  |
| `asset_accounts`                   | 対象資産口座の取得、利用者境界確認 |  ×  |
| `asset_account_available_settings` | 現在の利用可能資産区分の取得    |  ×  |

ACC-003は参照専用APIであるため、いずれのテーブルも更新しない。

---

### 21.1 users

`X-User-Id`で指定された操作対象利用者の存在確認に使用する。

概念的な条件は、以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者コンテキストの確認は、API共通Middlewareで行う。

ACC-003では、`users`を更新しない。

---

### 21.2 asset_accounts

指定された`assetAccountId`に対応する資産口座を取得するために使用する。

主に以下のカラムを使用する。

| カラム                      | 用途                   |
| ------------------------ | -------------------- |
| `id`                     | `assetAccountId`との照合 |
| `user_id`                | 操作対象利用者との利用者境界確認     |
| `name`                   | 資産口座名                |
| `asset_type`             | 資産種別                 |
| `balance_recording_unit` | 残高記録単位               |
| `start_year_month`       | 利用開始年月               |
| `deleted_at`             | 利用状態の判定              |

概念的な取得条件は、以下とする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 21.3 他利用者の資産口座

`asset_accounts.id`が一致していても、

```text
asset_accounts.user_id
    != 操作対象利用者ID
```

の場合は、取得対象に含めない。

他利用者の資産口座を一度取得してから、アプリケーション側で利用者境界を判定する方式を基本としない。

---

### 21.4 論理削除済み資産口座

ACC-003では、通常利用中の資産口座だけを取得対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は取得しない。

LaravelのSoftDeletesを使用している場合は、通常スコープを利用し、

```php
withTrashed()
```

を使用しない。

---

### 21.5 asset_type

`asset_accounts.asset_type`は、DB内部の保存形式からAPI用の資産種別へ変換する。

概念的には、

```text
DB
asset_type
    ↓
API
assetType
```

とする。

APIでは、例えば以下の値として返却する。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

DB内部の数値コードを直接返却しない。

---

### 21.6 balance_recording_unit

`asset_accounts.balance_recording_unit`も、API用の文字列表現へ変換する。

概念的には、

```text
DB
balance_recording_unit
    ↓
API
balanceRecordingUnit
```

とする。

APIでは、

```text
ACCOUNT
HOLDING
```

として返却する。

---

### 21.7 start_year_month

`asset_accounts.start_year_month`は、そのまま

```text
YYYY-MM
```

形式の

```text
startYearMonth
```

として返却する。

日付形式への不要な変換は行わない。

---

### 21.8 deleted_at

`asset_accounts.deleted_at`は、APIへ直接返却しない。

ACC-003では、`deleted_at IS NULL`の資産口座だけを取得するため、正常レスポンスでは

```text
isEnabled = true
```

として扱う。

---

### 21.9 asset_account_available_settings

現在の利用可能資産区分を取得するために使用する。

主に以下のカラムを使用する。

| カラム                | 用途         |
| ------------------ | ---------- |
| `asset_account_id` | 対象資産口座との関連 |
| `start_year_month` | 設定の適用開始年月  |
| `end_year_month`   | 設定の適用終了年月  |
| `is_available`     | 利用可能資産区分   |

---

### 21.10 現在有効な利用可能資産設定

現在年月を基準として、概念的に以下の条件を満たす利用可能資産設定を取得する。

```text
asset_account_available_settings.asset_account_id
    = assetAccountId

AND

asset_account_available_settings.start_year_month
    <= 現在年月

AND

(
    asset_account_available_settings.end_year_month
        IS NULL

    OR

    asset_account_available_settings.end_year_month
        >= 現在年月
)
```

正常状態では、該当する設定が1件だけ存在することを前提とする。

---

### 21.11 is_available

現在有効な設定の

```text
asset_account_available_settings.is_available
```

を、

```text
isAvailable
```

として返却する。

概念的には、

```text
is_available = true
    ↓
isAvailable = true
```

```text
is_available = false
    ↓
isAvailable = false
```

とする。

---

### 21.12 利用可能資産設定が存在しない場合

現在年月に有効な利用可能資産設定が存在しない場合は、データ不整合として扱う。

以下のような補完は行わない。

```text
設定なし
    ↓
isAvailable = false
```

ACC-002では初期利用可能資産設定を必ず登録する設計のため、正常状態として扱わない。

---

### 21.13 利用可能資産設定が複数存在する場合

現在年月に有効な利用可能資産設定が複数存在する場合も、データ不整合として扱う。

以下のように、

```text
start_year_monthが
最も新しい設定
```

を任意に採用しない。

期間重複を参照API側で隠蔽しない。

---

### 21.14 更新しないテーブル

ACC-003では、以下のテーブルも更新しない。

* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

また、これらのデータを資産口座詳細レスポンスへ含めない。

---

## 22. 関連する機能要件

ACC-003は、資産口座詳細取得および現在の利用可能資産区分表示に関する機能要件と対応する。

主な関連要件は、以下とする。

* 資産口座管理

  * 指定した資産口座の詳細情報を取得できる
  * 資産口座名を確認できる
  * 資産種別を確認できる
  * 残高記録単位を確認できる
  * 利用開始年月を確認できる
  * 利用状態を確認できる
* 利用可能資産区分

  * 利用可能資産区分を履歴として管理する
  * 現在年月に有効な設定から現在の利用可能資産区分を判定する
  * 利用可能資産設定の履歴そのものはACC-003で一覧返却しない
* 利用者境界

  * 操作対象利用者に帰属する資産口座だけを取得できる
  * 他利用者に属する資産口座は取得できない
  * 他利用者の資産口座は対象不存在として扱う
* 論理削除

  * 論理削除済み資産口座は通常の詳細取得対象としない

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

## 23. インデックス

ACC-003では、主に以下の検索条件を使用する。

```text
asset_accounts.id
```

```text
asset_accounts.user_id
```

```text
asset_account_available_settings.asset_account_id
+
asset_account_available_settings.start_year_month
+
asset_account_available_settings.end_year_month
```

必要なインデックスは、テーブル全体の利用状況を踏まえて設定する。

---

### 23.1 asset_accounts.id

`asset_accounts.id`は主キーであるため、主キーインデックスを使用する。

ACC-003専用として追加インデックスを作成しない。

---

### 23.2 asset_accounts.user_id

ACC-003では、主キーと利用者境界を組み合わせて取得する。

概念的には、

```text
id
+
user_id
```

を条件とする。

`id`は主キーであるため、Phase1ではACC-003のためだけに

```text
(id, user_id)
```

の複合インデックスを追加する必要性は低い。

実際のクエリ実行計画と他APIの検索条件を踏まえて判断する。

---

### 23.3 asset_account_available_settings

現在有効な設定取得では、

```text
asset_account_id
```

を必須条件として使用する。

利用可能資産設定は、同一資産口座内の期間履歴から検索するため、必要に応じて

```text
asset_account_id
+
start_year_month
```

などの複合インデックスを検討する。

HLDや目的達成判定など、対象年月判定を行う他APIでも利用される検索条件を踏まえ、共通して有効なインデックスを設計する。

---

### 23.4 不要な重複インデックスを作成しない

ACC-003のためだけに、既存の主キー・FK・UNIQUE制約と役割が重複するインデックスを追加しない。

実際のSQL、データ件数、実行計画を確認したうえで追加を判断する。

---

## 24. 性能

ACC-003は、単一の資産口座詳細を取得するAPIであるため、処理負荷は小さい。

概念的には、

```text
asset_accounts
1件取得
    ↓
asset_account_available_settings
現在有効な設定取得
    ↓
DTO生成
```

で完結する。

---

### 24.1 資産口座一覧を取得しない

以下のように、利用者の全資産口座を取得してからアプリケーション側で対象IDを探す実装は行わない。

```text
資産口座一覧取得
    ↓
assetAccountIdでfilter
```

対象IDと利用者IDを条件として直接1件取得する。

---

### 24.2 利用可能資産設定を全履歴取得しない

ACC-003で必要なのは、現在有効な利用可能資産設定だけである。

そのため、

```text
対象資産口座の
利用可能資産設定を全件取得
    ↓
PHP側で現在設定を検索
```

を基本としない。

データベース側で現在年月条件を指定して取得する。

---

### 24.3 取得件数

正常状態では、概念的に以下となる。

```text
asset_accounts
    1件

asset_account_available_settings
    1件
```

大量データをレスポンスへ返却しない。

---

### 24.4 保有商品をEager Loadしない

`balanceRecordingUnit = HOLDING`の場合でも、

```text
holding_assets
```

をEager Loadしない。

ACC-003では保有商品を返却しないためである。

例えば、

```php
with('holdingAssets')
```

をACC-003のためだけに使用しない。

---

### 24.5 月末資産データを取得しない

以下の関連データをACC-003で取得しない。

```text
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
```

不要なJOINやEager Loadを避ける。

---

### 24.6 N+1問題

ACC-003は単一資産口座を対象とするため、通常の意味でのN+1問題は発生しない。

ただし、将来的にACC-001と共通Queryを使用する場合でも、詳細取得のためだけに一覧向けの不要な関連取得処理を行わない。

---

## 25. セキュリティ

ACC-003では、利用者境界を最重要のセキュリティ要件とする。

---

### 25.1 assetAccountIdだけで取得しない

以下のような取得は避ける。

```php
AssetAccount::find(
    $assetAccountId,
);
```

これだけでは、他利用者の資産口座も取得できる可能性がある。

必ず、

```text
assetAccountId
+
userId
+
deleted_at IS NULL
```

を条件に含める。

---

### 25.2 他利用者の存在を公開しない

他利用者に属する`assetAccountId`を指定された場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下のような情報を返さない。

```text
この資産口座は
別の利用者に属しています
```

---

### 25.3 論理削除状態を公開しない

論理削除済み資産口座についても、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下のような状態判別可能なエラーは返却しない。

```text
ASSET_ACCOUNT_DISABLED
ASSET_ACCOUNT_DELETED
```

---

### 25.4 内部IDを不要に返さない

ACC-003では、対象資産口座自身の

```text
id
```

は返却するが、以下の内部IDは返却しない。

* `user_id`
* `asset_account_available_settings.id`
* `asset_account_available_settings.asset_account_id`

クライアントに不要な内部関連情報を公開しない。

---

### 25.5 DB内部コードを公開しない

`asset_type`や`balance_recording_unit`がDB内部で数値コードの場合でも、その値を公開しない。

API用の意味のある文字列へ変換する。

---

### 25.6 SQLインジェクション対策

`assetAccountId`や利用者IDを検索条件に使用する場合は、EloquentまたはQuery Builderのバインド機構を使用する。

SQL文字列へパラメータを直接連結しない。

---

### 25.7 エラー情報

異常時に、以下をレスポンスへ含めない。

* SQL
* PostgreSQL内部エラー
* テーブル名
* カラム名
* 制約名
* Laravel内部例外
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

## 26. ログ・監視

ACC-003では、API共通ログ方針に従って処理結果を記録する。

参照APIであるため、レスポンス内容そのものを不要にログへ出力しない。

---

### 26.1 ログコンテキスト

必要に応じて、以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assetAccountId
httpStatus
errorCode
```

`apiId`は、

```text
ACC-003
```

とする。

---

### 26.2 正常時

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = ACC-003
assetAccountId
httpStatus = 200
```

以下は、通常ログへ不要に出力しない。

* 資産口座名
* `isAvailable`
* 資産種別
* 残高記録単位

---

### 26.3 異常時

異常時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId
assetAccountId
errorCode
httpStatus
```

---

### 26.4 データ不整合時

以下のエラーは、業務データ不整合を示すため、調査可能なログを記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text
assetAccountId
currentYearMonth
matchedSettingCount
```

などを内部ログへ記録してよい。

ただし、レスポンスへは公開しない。

---

### 26.5 requestId

クライアントへ返却する`requestId`とサーバーログを関連付けられるようにする。

障害調査時に、

```text
requestId
    ↓
対象ログ特定
```

が可能な状態とする。

---

## 27. テスト観点

ACC-003では、資産口座詳細の正常取得だけでなく、以下を重点的に確認する。

* 利用者境界
* 論理削除
* 現在年月判定
* 利用可能資産設定の期間境界
* 設定不存在・重複
* レスポンス契約
* 副作用なし

---

### 27.1 正常系

操作対象利用者に属する有効な資産口座と、現在有効な利用可能資産設定を1件用意する。

ACC-003を実行し、以下を確認する。

* `200 OK`となること
* `id`が対象資産口座IDとなること
* `name`が正しいこと
* `assetType`が正しいこと
* `balanceRecordingUnit`が正しいこと
* `startYearMonth`が正しいこと
* `isAvailable`が現在設定と一致すること
* `isEnabled = true`となること

---

### 27.2 X-User-Id未指定

`X-User-Id`を指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

以下を確認する。

* 資産口座検索へ進まないこと
* 業務データが更新されないこと

---

### 27.3 X-User-Id形式不正

例えば、

```http
X-User-Id: abc
```

を指定する。

期待結果：

```text
400 Bad Request
INVALID_USER_ID
```

となること。

---

### 27.4 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 27.5 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 27.6 assetAccountId形式不正

以下の値をそれぞれ確認する。

```text
0
-1
abc
1.5
1e3
10abc
```

期待結果：

```text
400 Bad Request
INVALID_ASSET_ACCOUNT_ID
```

となること。

資産口座検索へ進まないことも確認する。

---

### 27.7 資産口座不存在

存在しない`assetAccountId`を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 27.8 他利用者の資産口座

以下の状態を用意する。

```text
User A
    Asset Account A

User B
    Asset Account B
```

User Aとして、Asset Account BのIDを指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの資産口座情報がレスポンスへ含まれないこと。

---

### 27.9 論理削除済み資産口座

以下の資産口座を用意する。

```text
deleted_at IS NOT NULL
```

そのIDでACC-003を実行する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 27.10 assetType = CASH

DB上の資産種別が現金に対応する値の場合、レスポンスが

```text
CASH
```

となることを確認する。

---

### 27.11 assetType = BANK

銀行に対応するDB値から、

```text
BANK
```

へ正しく変換されること。

---

### 27.12 assetType = SECURITIES

証券に対応するDB値から、

```text
SECURITIES
```

へ正しく変換されること。

---

### 27.13 assetType = IDECO

iDeCoに対応するDB値から、

```text
IDECO
```

へ正しく変換されること。

---

### 27.14 assetType = CORPORATE_DC

企業型DCに対応するDB値から、

```text
CORPORATE_DC
```

へ正しく変換されること。

---

### 27.15 assetType = OTHER

その他に対応するDB値から、

```text
OTHER
```

へ正しく変換されること。

---

### 27.16 balanceRecordingUnit = ACCOUNT

DB値が口座単位に対応する場合、

```text
ACCOUNT
```

として返却されることを確認する。

---

### 27.17 balanceRecordingUnit = HOLDING

DB値が商品単位に対応する場合、

```text
HOLDING
```

として返却されることを確認する。

---

### 27.18 isAvailable = true

現在有効な利用可能資産設定が

```text
is_available = true
```

の場合、

```json
{
  "isAvailable": true
}
```

となること。

---

### 27.19 isAvailable = false

現在有効な利用可能資産設定が

```text
is_available = false
```

の場合、

```json
{
  "isAvailable": false
}
```

となること。

`false`を未設定扱いしないこと。

---

### 27.20 start_year_monthが現在年月と一致

例えば、現在年月が

```text
2026-08
```

で、

```text
start_year_month = 2026-08
end_year_month   = NULL
```

の場合、現在有効な設定として取得されること。

---

### 27.21 end_year_monthが現在年月と一致

現在年月が

```text
2026-08
```

で、

```text
start_year_month = 2026-01
end_year_month   = 2026-08
```

の場合、現在有効な設定として取得されること。

---

### 27.22 end_year_monthが前月

現在年月が

```text
2026-08
```

で、

```text
end_year_month = 2026-07
```

の場合、当該設定が現在有効として取得されないこと。

---

### 27.23 start_year_monthが翌月

現在年月が

```text
2026-08
```

で、

```text
start_year_month = 2026-09
```

の場合、当該設定が現在有効として取得されないこと。

---

### 27.24 end_year_monthがNULL

現在年月が`start_year_month`以降で、

```text
end_year_month = NULL
```

の場合、現在有効な設定として取得されること。

---

### 27.25 現在有効な設定が0件

対象資産口座について、現在年月に有効な利用可能資産設定を0件とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

以下を確認する。

* `isAvailable = false`へ補完されないこと
* 正常レスポンスを返さないこと
* 業務データを自動修復しないこと

---

### 27.26 現在有効な設定が複数件

例えば、以下の2件を用意する。

```text
設定A
start_year_month = 2026-01
end_year_month   = NULL

設定B
start_year_month = 2026-06
end_year_month   = NULL
```

現在年月を`2026-08`とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

となること。

最新1件だけを任意に返却しないこと。

---

### 27.27 過去設定と現在設定

以下の履歴を用意する。

```text
設定A
2026-01 ～ 2026-06
is_available = true

設定B
2026-07 ～ NULL
is_available = false
```

現在年月が`2026-08`の場合、

```text
設定B
```

だけが選択され、

```text
isAvailable = false
```

となること。

---

### 27.28 境界が重複しない正常履歴

例えば、

```text
設定A
2026-01 ～ 2026-06

設定B
2026-07 ～ NULL
```

の場合、現在年月に応じて必ず1件だけが選択されることを確認する。

---

### 27.29 保有商品を返却しない

対象資産口座に複数の`holding_assets`が存在する状態でACC-003を実行する。

レスポンスへ、以下が含まれないことを確認する。

* `holdingAssets`
* 保有商品名
* 保有商品ID

---

### 27.30 月末資産データを返却しない

対象資産口座に月末資産データが存在する状態でも、レスポンスへ以下が含まれないことを確認する。

* `monthEndAssetSnapshots`
* `monthEndAssetBalances`
* `monthEndHoldingValues`
* 最新残高
* 資産推移

---

### 27.31 正常レスポンス契約

正常時に、概念的に以下の形式となることを確認する。

```json
{
  "data": {
    "id": "10",
    "name": "証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

以下を確認する。

* HTTPステータスが`200 OK`
* `id`がstring
* `name`がstring
* `assetType`がstring
* `balanceRecordingUnit`がstring
* `isAvailable`がboolean
* `startYearMonth`が`YYYY-MM`
* `isEnabled`がboolean
* `isEnabled = true`

---

### 27.32 返却しない情報

正常レスポンスに、以下が含まれないことを確認する。

* `user_id`
* `userId`
* `deleted_at`
* `deletedAt`
* `created_at`
* `createdAt`
* `updated_at`
* `updatedAt`
* `asset_account_available_settings.id`
* `assetAccountAvailableSettingId`
* `asset_account_id`
* `endYearMonth`
* DB内部の数値コード

---

### 27.33 副作用なし

ACC-003実行前後で、以下のテーブルに変更がないことを確認する。

* `users`
* `asset_accounts`
* `asset_account_available_settings`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

特に、詳細取得によって

```text
updated_at
```

が更新されないことを確認する。

---

### 27.34 同一リクエストの再実行

サーバー状態を変更せずに、同じ

```text
X-User-Id
+
assetAccountId
```

でACC-003を複数回実行する。

同じレスポンス内容となることを確認する。

---

### 27.35 ACC-006後の再取得

最初に、

```text
isAvailable = true
```

となるACC-003を実行する。

その後、ACC-006によって現在設定を変更し、再度ACC-003を実行する。

最新状態の

```text
isAvailable
```

が返却されることを確認する。

---

### 27.36 資産口座無効化後の再取得

ACC-003で正常取得できる資産口座を、ACC-005で無効化する。

その後、同じIDでACC-003を実行する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 27.37 currentYearMonthのテスト固定

利用可能資産設定の対象年月判定をテストする際は、Laravelの時刻固定機能などを使用して現在日時を固定する。

例えば、

```text
現在日時
2026-08-15
```

に固定し、

```text
現在年月
2026-08
```

として境界条件を確認する。

テスト実行日によって結果が変わらないようにする。

---

### 27.38 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
INTERNAL_SERVER_ERROR
```

以下も確認する。

* `error.code`が設定されること
* `error.message`が設定されること
* `requestId`が設定されること
* SQLが含まれないこと
* PostgreSQL内部エラーが含まれないこと
* スタックトレースが含まれないこと
* サーバーファイルパスが含まれないこと

---

### 27.39 データ不整合ログ

以下を発生させた場合に、調査可能なログが記録されることを確認する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text
requestId
userId
assetAccountId
currentYearMonth
```

などから対象ログを追跡できることを確認する。

---

### 27.40 INTERNAL_SERVER_ERROR

資産口座詳細取得処理で想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

* 業務データが更新されないこと
* 内部情報がレスポンスへ公開されないこと
* サーバーログに調査情報が記録されること
* レスポンスの`requestId`からログを追跡できること

---

## 28. Laravel実装方針

ACC-003では、Action、Query、DTO、API Resource、Responderを分離して実装する。

概念的な構成は、以下とする。

```text id="f3z6rq"
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ AssetAccountQuery
    └─ AssetAccountAvailableSettingQuery
    ↓
Detail Result DTO
    ↓
API Resource
    ↓
Responder
```

本APIはリクエストボディを使用しないため、ACC-003専用のFormRequestは作成しない。

また、参照専用APIであるため、Repositoryは使用せず、取得処理はQueryへ集約する。

---

### 28.1 Route

ACC-003は、以下のルートとして定義する。

概念例：

```php id="6jn9va"
Route::get(
    '/api/v1/asset-accounts/{assetAccountId}',
    ShowAssetAccountAction::class,
);
```

ACC-001とは、HTTPメソッドおよびパスによって区別する。

```text id="xquw2f"
GET
/api/v1/asset-accounts
    → ACC-001 資産口座一覧取得

GET
/api/v1/asset-accounts/{assetAccountId}
    → ACC-003 資産口座詳細取得
```

---

### 28.2 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* 共通例外処理
* ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、以下とする。

```text id="ah9d3g"
X-User-Id取得
    ↓
必須チェック
    ↓
形式チェック
    ↓
users存在確認
    ↓
UserContext設定
    ↓
Action
```

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 28.3 FormRequest

ACC-003専用のFormRequestは作成しない。

本APIでは、

```text id="2c8xiy"
リクエストボディ
    なし

クエリパラメータ
    なし
```

であり、ACC-003固有のRequest Bodyバリデーションが存在しないためである。

以下のような空のFormRequestは作成しない。

```php id="yx5blc"
final class ShowAssetAccountRequest
    extends FormRequest
{
}
```

不要なクラスを追加しない。

---

### 28.4 assetAccountIdの形式検証

`assetAccountId`は、ルート制約またはAPI共通のパスパラメータ検証方式で検証する。

概念例：

```php id="3j6h5m"
Route::get(
    '/api/v1/asset-accounts/{assetAccountId}',
    ShowAssetAccountAction::class,
)
    ->where(
        'assetAccountId',
        '[1-9][0-9]*',
    );
```

正の整数形式のみを許可する。

以下を有効なIDとして扱わない。

```text id="26c1kl"
0
-1
abc
1.5
1e3
10abc
```

形式不正は、

```text id="l8n0vg"
INVALID_ASSET_ACCOUNT_ID
```

へ変換する。

---

### 28.5 Action

Actionは、パスパラメータと利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php id="vg1yuc"
final class ShowAssetAccountAction
{
    public function __invoke(
        string $assetAccountId,
        ShowAssetAccountUseCase $useCase,
        ShowAssetAccountResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                assetAccountId:
                    (int) $assetAccountId,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

`assetAccountId`は、形式検証済みであることを前提に整数へ変換する。

---

### 28.6 Actionで行わないこと

Actionでは、以下を行わない。

* 資産口座検索
* 利用者境界判定
* 論理削除判定
* 現在年月算出
* 利用可能資産設定検索
* 設定件数判定
* データ不整合判定
* Enum変換
* レスポンス配列生成
* SQL生成

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 28.7 UseCase

ACC-003の参照ユースケース全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `assetAccountId`を受け取る
3. 操作対象利用者に属する有効な資産口座を取得する
4. 存在しない場合は業務例外を送出する
5. 現在年月を取得する
6. 現在有効な利用可能資産設定を取得する
7. 設定件数が1件であることを確認する
8. Detail Result DTOを生成する
9. DTOを返却する

概念的には、以下とする。

```text id="ue38c9"
userId
+
assetAccountId
    ↓
AssetAccountQuery
    ↓
資産口座取得
    ↓
存在確認
    ↓
CurrentYearMonth取得
    ↓
AssetAccountAvailableSettingQuery
    ↓
現在有効設定取得
    ↓
件数確認
    ↓
Detail Result DTO
```

---

### 28.8 AssetAccountQuery

資産口座取得は、専用Queryへ委譲する。

概念例：

```php id="jm6wga"
$assetAccount =
    $this->assetAccountQuery
        ->findActiveByIdAndUser(
            assetAccountId:
                $assetAccountId,

            userId:
                $userId,
        );
```

概念的な検索条件は、以下とする。

```text id="lpf1z9"
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 28.9 利用者境界をQueryへ含める

以下のように、まずIDだけで取得してからUseCase側で利用者IDを比較する方式を基本としない。

```php id="3nnwlq"
$assetAccount =
    AssetAccount::find(
        $assetAccountId,
    );
```

対象取得時点から、

```text id="nszbf2"
assetAccountId
+
userId
+
有効状態
```

を条件へ含める。

これにより、他利用者の資産口座を誤って取得することを防止する。

---

### 28.10 SoftDeletes

`AssetAccount` Modelでは、Laravelの

```php id="p3tf0c"
use SoftDeletes;
```

を使用する。

ACC-003の通常取得では、

```php id="xv7olh"
withTrashed()
```

を使用しない。

論理削除済み資産口座は、詳細取得対象外とする。

---

### 28.11 ASSET_ACCOUNT_NOT_FOUND

資産口座を取得できない場合は、

```php id="dfwx7a"
if ($assetAccount === null) {
    throw new
        AssetAccountNotFoundException();
}
```

のように業務例外を送出する。

以下を同じ例外へ集約する。

* 資産口座不存在
* 他利用者に属する
* 論理削除済み

最終的に、

```text id="eizmy1"
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

へ変換する。

---

### 28.12 CurrentYearMonth

利用可能資産設定の判定に使用する現在年月は、UseCase内でサーバー時刻から取得する。

概念例：

```php id="a8fwmr"
$currentYearMonth =
    now()->format(
        'Y-m',
    );
```

ただし、テスト容易性を高めるため、現在時刻取得を直接`now()`へ依存させず、ClockやDateProviderを利用してもよい。

概念例：

```php id="2o8hvr"
$currentYearMonth =
    $this->clock
        ->now()
        ->format(
            'Y-m',
        );
```

---

### 28.13 Clockを分離してよい理由

ACC-003の

```text id="sge4w7"
isAvailable
```

は、現在年月によって結果が変化する。

そのため、時刻依存処理をテストで固定しやすくするために、現在時刻取得を専用インターフェースへ分離してよい。

例えば、

```php id="t67d9c"
interface Clock
{
    public function now(): CarbonImmutable;
}
```

とする。

Phase1で過剰になる場合は、Laravelの時刻固定機能を使い、`now()`を直接使用してもよい。

---

### 28.14 AssetAccountAvailableSettingQuery

現在有効な利用可能資産設定の取得は、専用Queryへ委譲する。

概念例：

```php id="k0zmf2"
$settings =
    $this->availableSettingQuery
        ->findCurrentByAssetAccount(
            assetAccountId:
                $assetAccount->id,

            yearMonth:
                $currentYearMonth,
        );
```

概念的な検索条件は、以下とする。

```text id="fyso03"
asset_account_id
    = assetAccountId

AND

start_year_month
    <= currentYearMonth

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= currentYearMonth
)
```

---

### 28.15 設定を全件取得しない

以下のように、利用可能資産設定の全履歴を取得してからPHP側で現在設定を探す方式は基本としない。

```php id="mw4e40"
$settings =
    AssetAccountAvailableSetting::query()
        ->where(
            'asset_account_id',
            $assetAccountId,
        )
        ->get();
```

データベース側で現在年月条件を指定する。

---

### 28.16 件数判定

現在有効な設定は、正常状態では1件だけ存在することを前提とする。

概念例：

```php id="l2i3vk"
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingNotFoundException();
}

if ($settings->count() > 1) {
    throw new
        AssetAccountAvailableSettingConflictException();
}
```

0件または複数件を正常状態として扱わない。

---

### 28.17 `first()`だけで済ませない

以下のような実装は基本としない。

```php id="y3pfwy"
$setting =
    $query
        ->orderByDesc(
            'start_year_month',
        )
        ->first();
```

これでは、設定期間が重複していても最新1件を返してデータ不整合を隠蔽する可能性がある。

ACC-003では、該当件数を確認し、1件であることを保証する。

---

### 28.18 取得上限

複数件存在するかを検出するだけであれば、全件を取得する必要はない。

概念的には、最大2件まで取得してよい。

```php id="87fmxa"
$settings =
    AssetAccountAvailableSetting::query()
        ->where(...)
        ->limit(2)
        ->get();
```

これにより、

```text id="v9j3fp"
0件
1件
2件以上
```

を判定できる。

---

### 28.19 Repositoryを使用しない

ACC-003は参照専用APIであり、以下のDB更新を行わない。

```text id="mjkqun"
INSERT
UPDATE
DELETE
```

そのため、ACC-003専用のRepositoryは使用しない。

概念的には、

```text id="py7f2t"
Query
    → 読み取り

Repository
    → 書き込み
```

という責務分離方針に従う。

---

### 28.20 トランザクション

ACC-003では、明示的な

```php id="0g6kuw"
DB::transaction()
```

を使用しない。

参照専用であり、表示用途として厳密な同一時点スナップショットを要求しないためである。

---

### 28.21 lockForUpdateを使用しない

ACC-003では、以下を使用しない。

```php id="al5fse"
lockForUpdate()
```

資産口座や利用可能資産設定を参照するだけであり、詳細表示のために更新処理をブロックしない。

---

### 28.22 Enum Cast

`asset_type`と`balance_recording_unit`は、可能であればEloquent CastまたはPHP Enumとして扱う。

概念例：

```php id="e3d4u0"
protected function casts(): array
{
    return [
        'asset_type'
            => AssetType::class,

        'balance_recording_unit'
            => BalanceRecordingUnit::class,
    ];
}
```

DB保存形式がAPI公開値と異なる場合は、Mapper等で変換する。

---

### 28.23 DB内部コードをUseCaseへ漏らさない

例えば、

```text id="gtq6xq"
1 = CASH
2 = BANK
3 = SECURITIES
```

のようなDB内部コードを、UseCaseやResourceへ直接持ち込まない。

UseCase以降では、

```text id="8prwli"
AssetType
BalanceRecordingUnit
```

という意味のある型として扱う。

---

### 28.24 Detail Result DTO

取得結果は、専用DTOとして表現する。

概念例：

```php id="8qey2m"
final readonly class AssetAccountDetailResult
{
    public function __construct(
        public int $id,
        public string $name,
        public AssetType $assetType,
        public BalanceRecordingUnit $balanceRecordingUnit,
        public bool $isAvailable,
        public string $startYearMonth,
    ) {
    }
}
```

`isEnabled`は、ACC-003正常時に必ず`true`となるため、Resource側で固定値として設定してよい。

---

### 28.25 DTOへEloquent Modelを保持しない

以下のようなDTOは基本としない。

```php id="0rbp13"
final readonly class AssetAccountDetailResult
{
    public function __construct(
        public AssetAccount $assetAccount,
        public AssetAccountAvailableSetting $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

これにより、HTTP層へEloquent Modelを直接渡さない。

---

### 28.26 UseCaseの戻り値

概念例：

```php id="m1zp84"
return new AssetAccountDetailResult(
    id:
        $assetAccount->id,

    name:
        $assetAccount->name,

    assetType:
        $assetAccount->asset_type,

    balanceRecordingUnit:
        $assetAccount->balance_recording_unit,

    isAvailable:
        $setting->is_available,

    startYearMonth:
        $assetAccount->start_year_month,
);
```

---

### 28.27 API Resource

Detail Result DTOを、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php id="d61q3c"
final class AssetAccountDetailResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'name'
                => $this->name,

            'assetType'
                => $this->assetType->value,

            'balanceRecordingUnit'
                => $this->balanceRecordingUnit->value,

            'isAvailable'
                => $this->isAvailable,

            'startYearMonth'
                => $this->startYearMonth,

            'isEnabled'
                => true,
        ];
    }
}
```

APIフィールド名は、共通方針に従ってcamelCaseとする。

---

### 28.28 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

* `user_id`
* `deleted_at`
* `created_at`
* `updated_at`
* `asset_account_available_settings.id`
* `asset_account_available_settings.asset_account_id`
* `asset_account_available_settings.start_year_month`
* `asset_account_available_settings.end_year_month`
* DB内部コード

また、以下の関連データも返却しない。

* 保有商品一覧
* 月末資産残高
* 商品別月末評価額
* 月末資産状況

---

### 28.29 Responder

Responderは、Detail Result DTOを`200 OK`レスポンスへ変換する。

概念例：

```php id="os6g3n"
final class ShowAssetAccountResponder
{
    public function ok(
        AssetAccountDetailResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new AssetAccountDetailResource(
                        $result,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの共通Envelope項目は、API共通レスポンス処理に従う。

---

### 28.30 Responderで行わないこと

Responderでは、以下を行わない。

* 資産口座検索
* 利用者境界判定
* 現在年月取得
* 利用可能資産設定検索
* 設定件数判定
* データ不整合判定
* DBアクセス
* Enum変換ロジックの組み立て

Responderは、生成済みDTOをHTTPレスポンスへ変換することに責務を限定する。

---

### 28.31 例外変換

主な例外変換は、以下とする。

| 内部状態                 | 独自エラーコード                                    |
| -------------------- | ------------------------------------------- |
| `X-User-Id`未指定       | `USER_CONTEXT_REQUIRED`                     |
| `X-User-Id`形式不正      | `INVALID_USER_ID`                           |
| 利用者不存在               | `USER_NOT_FOUND`                            |
| `assetAccountId`形式不正 | `INVALID_ASSET_ACCOUNT_ID`                  |
| 資産口座不存在              | `ASSET_ACCOUNT_NOT_FOUND`                   |
| 他利用者の資産口座            | `ASSET_ACCOUNT_NOT_FOUND`                   |
| 論理削除済み資産口座           | `ASSET_ACCOUNT_NOT_FOUND`                   |
| 現在設定0件               | `ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND` |
| 現在設定複数件              | `ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT`  |
| 想定外例外                | `INTERNAL_SERVER_ERROR`                     |

---

### 28.32 AssetAccountNotFoundException

対象資産口座を取得できない場合は、業務例外を送出する。

概念例：

```php id="m9svb7"
if ($assetAccount === null) {
    throw new
        AssetAccountNotFoundException();
}
```

最終的に、

```text id="k3j57n"
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

へ変換する。

---

### 28.33 AssetAccountAvailableSettingNotFoundException

現在有効な利用可能資産設定が0件の場合は、データ不整合として専用例外を送出する。

概念例：

```php id="pijztt"
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingNotFoundException();
}
```

最終的に、

```text id="d7a8t1"
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

へ変換する。

---

### 28.34 AssetAccountAvailableSettingConflictException

現在有効な設定が複数件存在する場合は、専用例外を送出する。

概念例：

```php id="hmzqvi"
if ($settings->count() > 1) {
    throw new
        AssetAccountAvailableSettingConflictException();
}
```

最終的に、

```text id="y0ne3a"
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

へ変換する。

---

### 28.35 データ不整合時に自動修復しない

以下のような処理は行わない。

```text id="p5r4n6"
現在設定0件
    ↓
isAvailable = false
```

または、

```text id="5pf1x4"
現在設定複数件
    ↓
最新1件採用
```

ACC-003は参照APIであるため、不整合状態を暗黙的に修復しない。

---

### 28.36 想定外例外

想定外の例外は、API共通Exception Handlerで

```text id="zps3jg"
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

* SQL
* PostgreSQL内部エラー
* 制約名
* テーブル名
* カラム名
* Laravel内部例外メッセージ
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

---

### 28.37 ログ

ACC-003では、必要に応じて以下をログコンテキストへ設定する。

```text id="03ox3h"
requestId
userId
apiId
assetAccountId
currentYearMonth
httpStatus
errorCode
```

`apiId`は、

```text id="cd29r4"
ACC-003
```

とする。

正常時に、資産口座名や`isAvailable`などの不要な業務情報をログへ出力しない。

---

### 28.38 データ不整合ログ

以下のエラー時は、調査に必要な情報を内部ログへ記録する。

```text id="316ihd"
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text id="ghaq9s"
requestId
userId
assetAccountId
currentYearMonth
matchedSettingCount
```

を記録してよい。

レスポンスへは内部状態を公開しない。

---

### 28.39 キャッシュ

Phase1では、ACC-003専用のサーバー側キャッシュを使用しない。

毎回、最新の

```text id="aq1e79"
asset_accounts
asset_account_available_settings
```

を参照する。

フロントエンド側では、TanStack Query等によるQuery Cacheを使用してよい。

---

### 28.40 テスト実装方針

Laravel側では、Feature Testを中心としてACC-003のAPI契約を確認する。

主に以下を確認する。

* `200 OK`
* `400 Bad Request`
* `404 Not Found`
* `500 Internal Server Error`
* `X-User-Id`必須
* 利用者存在確認
* `assetAccountId`形式
* 資産口座存在確認
* 利用者境界
* SoftDeletes
* 現在年月判定
* 利用可能資産設定の期間境界
* 設定0件
* 設定複数件
* API用Enum変換
* 副作用なし
* レスポンス契約

---

### 28.41 AssetAccountQueryのDatabase Test

`AssetAccountQuery`について、以下を確認する。

```text id="i6yq2u"
id一致
+
user_id一致
+
deleted_at IS NULL
    ↓
取得できる
```

以下の場合は取得できないことを確認する。

* 存在しないID
* 他利用者の資産口座
* 論理削除済み資産口座

---

### 28.42 AssetAccountAvailableSettingQueryのDatabase Test

現在年月を固定し、以下を確認する。

```text id="cdg71e"
start_year_month
    <= currentYearMonth

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= currentYearMonth
)
```

境界条件として、以下を確認する。

* `start_year_month = currentYearMonth`
* `end_year_month = currentYearMonth`
* `end_year_month = 前月`
* `start_year_month = 翌月`
* `end_year_month = NULL`

---

### 28.43 設定0件のTest

現在有効な利用可能資産設定を0件にする。

期待結果：

```text id="2qw7wm"
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

となること。

また、

```text id="ksgt0x"
isAvailable = false
```

へ補完されないことを確認する。

---

### 28.44 設定複数件のTest

現在年月に有効な設定を2件以上用意する。

期待結果：

```text id="8way71"
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

となること。

最新設定を任意に返却しないことを確認する。

---

### 28.45 UseCaseのUnit Test

UseCaseでは、QueryをMockして業務フローを確認する。

正常系：

```text id="qpsj7q"
AssetAccountQuery
    ↓
資産口座取得
    ↓
AvailableSettingQuery
    ↓
現在設定1件
    ↓
Detail Result DTO
```

資産口座不存在：

```text id="yhnojt"
AssetAccountQuery
    ↓
null
    ↓
AssetAccountNotFoundException
```

設定0件：

```text id="s6a43f"
AvailableSettingQuery
    ↓
0件
    ↓
AssetAccountAvailableSettingNotFoundException
```

設定複数件：

```text id="7kzzp1"
AvailableSettingQuery
    ↓
2件
    ↓
AssetAccountAvailableSettingConflictException
```

---

### 28.46 時刻依存テスト

現在年月判定では、テスト実行時刻を固定する。

例えば、

```text id="udf7ie"
2026-08-15
```

に固定し、

```text id="uq5jrm"
currentYearMonth = 2026-08
```

として利用可能資産設定を判定する。

Laravelの時刻固定機能またはClockのTest Doubleを使用し、実行日によってテスト結果が変わらないようにする。

---

### 28.47 API ResourceのTest

正常時に、以下の項目だけが`data`へ含まれることを確認する。

```json id="oj1k4l"
{
  "id": "10",
  "name": "証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "isAvailable": true,
  "startYearMonth": "2026-08",
  "isEnabled": true
}
```

以下が含まれないことを確認する。

* `userId`
* `user_id`
* `deletedAt`
* `createdAt`
* `updatedAt`
* `assetAccountAvailableSettingId`
* `asset_account_id`
* `start_year_month`
* `end_year_month`
* DB内部の数値コード

---

### 28.48 副作用なしのTest

ACC-003実行前後で、以下のテーブルが変更されないことを確認する。

```text id="1pmhv3"
asset_accounts
asset_account_available_settings
```

特に、

```text id="slz8dh"
updated_at
```

が詳細取得によって更新されないことを確認する。

---

## 29. React・TypeScriptでの利用

ACC-003は、資産口座一覧や資産口座編集画面から、指定した資産口座の詳細情報を取得するために使用する。

主な利用フローは、以下とする。

```text
ACC-001
資産口座一覧取得
    ↓
利用者が資産口座を選択
    ↓
ACC-003
資産口座詳細取得
    ↓
詳細表示
または
編集フォーム初期値へ反映
```

ACC-003は参照専用APIであるため、TanStack Queryを使用する場合はQueryとして扱う。

---

### 29.1 TypeScript型

ACC-003の正常レスポンスデータは、以下のような型として扱う。

概念例：

```ts
export type AssetAccountDetail = {
  id: string;
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  isAvailable: boolean;
  startYearMonth: string;
  isEnabled: boolean;
};
```

API共通Envelopeを使用する場合は、以下のように定義する。

```ts
export type AssetAccountDetailResponse =
  ApiResponse<AssetAccountDetail>;
```

---

### 29.2 AssetType

`assetType`は、ACC-002などと共通の型を使用する。

概念例：

```ts
export type AssetType =
  | 'CASH'
  | 'BANK'
  | 'SECURITIES'
  | 'IDECO'
  | 'CORPORATE_DC'
  | 'OTHER';
```

ACC-003専用として同じUnion型を重複定義しない。

---

### 29.3 BalanceRecordingUnit

`balanceRecordingUnit`も、ACC-002などと共通の型を使用する。

概念例：

```ts
export type BalanceRecordingUnit =
  | 'ACCOUNT'
  | 'HOLDING';
```

---

### 29.4 assetAccountId

ACC-003では、取得対象となる`assetAccountId`をAPI Client関数の引数として渡す。

概念例：

```ts
getAssetAccountDetail(
  assetAccountId,
);
```

`assetAccountId`は、API契約に合わせて`string`として扱う。

---

### 29.5 userIdを関数引数へ含めない

ACC-003専用のAPI Client関数へ、

```ts
getAssetAccountDetail(
  userId,
  assetAccountId,
);
```

のように`userId`を渡さない。

利用者IDは、`X-User-Id`として共通API Clientから付与する。

---

### 29.6 X-User-Id

`X-User-Id`は、ACC-003専用処理ではなく、共通API Clientから付与する。

概念例：

```ts
apiClient.interceptors.request.use(
  (config) => {
    config.headers['X-User-Id'] =
      currentUserId;

    return config;
  },
);
```

ACC-003を呼び出すコンポーネントから利用者ヘッダーを直接組み立てない。

---

### 29.7 API Client

ACC-003を呼び出す専用関数を定義する。

概念例：

```ts
export const getAssetAccountDetail =
  async (
    assetAccountId: string,
  ): Promise<AssetAccountDetail> => {
    const response =
      await apiClient.get<
        AssetAccountDetailResponse
      >(
        `/api/v1/asset-accounts/${assetAccountId}`,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

### 29.8 Queryとして扱う

ACC-003は参照専用APIであるため、TanStack Queryを使用する場合はQueryとして扱う。

概念例：

```ts
export const useAssetAccountDetail =
  (
    assetAccountId: string,
  ) =>
    useQuery({
      queryKey: [
        'assetAccounts',
        assetAccountId,
      ],
      queryFn: () =>
        getAssetAccountDetail(
          assetAccountId,
        ),
    });
```

---

### 29.9 Query Key

資産口座詳細のQuery Keyは、資産口座IDを含める。

概念的には、

```text
assetAccounts
+
assetAccountId
```

とする。

例えば、

```ts
[
  'assetAccounts',
  assetAccountId,
]
```

とする。

---

### 29.10 Query Keyの共通化

Query Keyは、各コンポーネントへ直接記述せず、共通定義として管理してよい。

概念例：

```ts
export const assetAccountKeys = {
  all: [
    'assetAccounts',
  ] as const,

  detail: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
    ] as const,
};
```

ACC-003では、

```ts
queryKey:
  assetAccountKeys.detail(
    assetAccountId,
  ),
```

として使用できる。

---

### 29.11 assetAccountIdが存在しない場合

画面初期化時などで`assetAccountId`がまだ取得できていない場合は、Queryを実行しない。

概念例：

```ts
useQuery({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),

  queryFn: () =>
    getAssetAccountDetail(
      assetAccountId,
    ),

  enabled:
    assetAccountId !== '',
});
```

不完全なIDでACC-003を実行しない。

---

### 29.12 ルートパラメータから取得する場合

詳細画面では、React RouterなどのURLパラメータから`assetAccountId`を取得してよい。

概念例：

```ts
const {
  assetAccountId,
} = useParams<{
  assetAccountId: string;
}>();
```

ただし、`assetAccountId`が存在することを確認してからACC-003を実行する。

---

### 29.13 ローディング表示

ACC-003取得中は、ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return (
    <Loading />
  );
}
```

取得前の空データを正常な詳細情報として表示しない。

---

### 29.14 正常取得

正常時は、ACC-003から返却された`AssetAccountDetail`を画面表示へ使用する。

概念例：

```ts
const {
  data,
} = useAssetAccountDetail(
  assetAccountId,
);
```

---

### 29.15 詳細表示

詳細画面では、必要に応じて以下を表示する。

* 資産口座名
* 資産種別
* 残高記録単位
* 利用開始年月
* 現在の利用可能資産区分
* 利用状態

---

### 29.16 assetTypeの表示

APIから返却された`SECURITIES`などを、画面表示では日本語ラベルへ変換する。

概念例：

```ts
export const assetTypeLabels:
  Record<AssetType, string> = {
    CASH: '現金',
    BANK: '銀行',
    SECURITIES: '証券',
    IDECO: 'iDeCo',
    CORPORATE_DC: '企業型DC',
    OTHER: 'その他',
  };
```

API値自体を日本語へ変換してStateへ保存し直す必要はない。

---

### 29.17 balanceRecordingUnitの表示

概念例：

```ts
export const balanceRecordingUnitLabels:
  Record<
    BalanceRecordingUnit,
    string
  > = {
    ACCOUNT: '口座単位',
    HOLDING: '商品単位',
  };
```

---

### 29.18 isAvailableの表示

`isAvailable`は、現在の利用可能資産区分として表示する。

概念例：

```tsx
<span>
  {assetAccount.isAvailable
    ? '利用可能資産'
    : '利用対象外'}
</span>
```

表示文言は、画面設計で定義した用語に統一する。

---

### 29.19 isEnabledの表示

ACC-003の正常レスポンスでは、`isEnabled = true`となる。

そのため、通常の詳細画面で利用状態を表示する場合は、

```text
利用中
```

として表示できる。

ただし、論理削除済み資産口座はACC-003で取得できないため、ACC-003だけを利用する画面で

```text
無効
```

状態を表示する必要はない。

---

### 29.20 startYearMonthの表示

`startYearMonth`は、

```text
YYYY-MM
```

形式で返却される。

画面表示では、必要に応じて

```text
2026年8月
```

のように表示用フォーマットへ変換してよい。

API値そのものは変更しない。

---

### 29.21 編集画面の初期値

ACC-003は、ACC-004 資産口座更新画面の初期値取得にも使用できる。

概念的には、

```text
ACC-003
    ↓
name
assetType
balanceRecordingUnit
startYearMonth
isAvailable
    ↓
編集フォーム初期値
```

とする。

ただし、ACC-004が実際に更新可能とする項目だけを編集フォームへ反映する。

---

### 29.22 Form Stateへの変換

詳細DTOと編集フォームStateは別型として定義してよい。

概念例：

```ts
export type AssetAccountFormValues = {
  name: string;
  assetType: AssetType;
  balanceRecordingUnit:
    BalanceRecordingUnit;
  startYearMonth: string;
  isAvailable: boolean;
};
```

変換例：

```ts
const initialValues:
  AssetAccountFormValues = {
    name:
      assetAccount.name,

    assetType:
      assetAccount.assetType,

    balanceRecordingUnit:
      assetAccount.balanceRecordingUnit,

    startYearMonth:
      assetAccount.startYearMonth,

    isAvailable:
      assetAccount.isAvailable,
  };
```

---

### 29.23 詳細DTOを直接編集しない

ACC-003で取得した`AssetAccountDetail`を、そのまま編集用Stateとして直接書き換えない。

Query Cache上のデータをフォーム入力によって直接変更しないためである。

編集用Stateへ必要な値をコピーする。

---

### 29.24 ASSET_ACCOUNT_NOT_FOUND

ACC-003で

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

が返却された場合は、対象資産口座を表示できない状態として扱う。

例えば、

```text
指定された資産口座が
見つかりません。
```

と表示する。

---

### 29.25 他利用者かどうかを推測しない

`ASSET_ACCOUNT_NOT_FOUND`には、以下が含まれる。

* 資産口座不存在
* 他利用者の資産口座
* 論理削除済み資産口座

フロントエンドでは、どの理由かを推測して表示を分岐しない。

---

### 29.26 論理削除済みかどうかを推測しない

ACC-003から`ASSET_ACCOUNT_NOT_FOUND`が返却されても、

```text
この資産口座は
無効化されています
```

と断定しない。

API契約上、無効化済みか不存在かは区別されないためである。

---

### 29.27 INVALID_ASSET_ACCOUNT_ID

通常の画面遷移では、サーバーから取得した有効なIDを使用するため、発生頻度は低い。

URL手入力などにより

```text
INVALID_ASSET_ACCOUNT_ID
```

が返却された場合は、不正なURLまたは不正な画面状態として扱う。

例えば、資産口座一覧へ戻す導線を表示してよい。

---

### 29.28 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-003画面だけで独自処理を実装しない。

---

### 29.29 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、API共通の利用者コンテキストエラーとして扱う。

必要に応じて、現在選択している利用者状態を再確認する。

---

### 29.30 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合も、API共通の利用者コンテキストエラーとして扱う。

現在選択されている利用者が有効ではない状態として処理する。

---

### 29.31 利用可能資産設定不存在

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

は、フロントエンド入力によるエラーではなく、サーバー側の業務データ不整合である。

画面では、一般的な取得失敗として扱う。

例えば、

```text
資産口座の詳細を取得できませんでした。
```

と表示する。

内部の

```text
利用可能資産設定が存在しない
```

という詳細を一般利用者向け画面へそのまま表示する必要はない。

---

### 29.32 利用可能資産設定重複

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

についても、サーバー側データ不整合として扱う。

フロントエンドで複数設定から独自に1件を選択して画面表示しない。

---

### 29.33 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
資産口座の詳細を取得できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

### 29.34 Retry

ACC-003は参照専用GET APIであるため、一時的な通信エラーについてTanStack QueryのRetry機能を利用してよい。

ただし、

```text
ASSET_ACCOUNT_NOT_FOUND
```

やデータ不整合系エラーに対して、無意味な再試行を繰り返さない。

正式なRetry方針は、フロントエンド共通設計に従う。

---

### 29.35 Query Cache

ACC-003の結果は、TanStack QueryのQuery Cacheへ保持してよい。

ただし、以下のAPI成功後はキャッシュが古くなる可能性がある。

```text
ACC-004
資産口座更新

ACC-005
資産口座無効化

ACC-006
利用可能資産設定登録
```

そのため、各Mutation成功時に対象詳細Queryをinvalidateする。

---

### 29.36 ACC-004成功後

ACC-004によって資産口座情報が更新された場合は、対象資産口座の詳細Queryを無効化する。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

必要に応じてACC-001一覧Queryも無効化する。

---

### 29.37 ACC-005成功後

ACC-005によって資産口座が無効化された場合は、ACC-003の通常取得対象から外れる。

そのため、対象詳細Queryを無効化または削除する。

概念例：

```ts
queryClient.removeQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

その後、資産口座一覧画面へ遷移してよい。

---

### 29.38 ACC-006成功後

ACC-006によって現在の利用可能資産区分が変更された場合は、ACC-003の`isAvailable`が変化する可能性がある。

そのため、対象詳細Queryをinvalidateする。

---

### 29.39 ACC-002成功直後

ACC-002成功後に登録した資産口座の詳細画面へ遷移する場合は、返却された`data.id`を使用してACC-003を実行する。

ACC-002のレスポンスだけをACC-003の永続的なキャッシュ代替としない。

---

### 29.40 現在年月による変化

ACC-003の`isAvailable`は、現在年月によって変化する可能性がある。

例えば、

```text
2026-08
    isAvailable = true

2026-09
    isAvailable = false
```

となる設定履歴が存在する場合、月が変わればDB更新がなくてもACC-003の結果が変化し得る。

そのため、非常に長い`staleTime`を無条件に設定しない。

具体的なキャッシュ時間は、フロントエンド共通設計で決定する。

---

### 29.41 userId変更時

利用者切替が行われた場合は、前利用者のACC-003結果をそのまま表示し続けない。

Query Cacheを利用者単位で適切に分離または無効化する。

API自体は`X-User-Id`によって利用者境界を保証するが、画面上でも前利用者のキャッシュを誤表示しないようにする。

---

### 29.42 Query Keyと利用者境界

資産口座IDがシステム全体で一意であっても、利用者切替機能があるため、必要に応じてQuery Keyへ利用者IDを含めてもよい。

概念例：

```ts
export const assetAccountKeys = {
  detail: (
    userId: string,
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      userId,
      assetAccountId,
    ] as const,
};
```

ただし、Query Keyの正式な設計はReact共通設計に従う。

---

### 29.43 フロントエンドで現在設定を再計算しない

ACC-003では、バックエンドが現在年月を基準として`isAvailable`を判定する。

フロントエンドでは、利用可能資産設定履歴を取得して独自に

```text
現在有効なのはどの設定か
```

を再計算しない。

ACC-003から返却された`isAvailable`を現在状態として使用する。

---

### 29.44 DB内部コードを扱わない

React側では、以下のようなDB内部コードを扱わない。

```text
asset_type = 3
balance_recording_unit = 2
```

APIから返却された

```text
SECURITIES
HOLDING
```

などの業務上意味のある値を使用する。

---

### 29.45 DBカラム名を型へ持ち込まない

ACC-003の型では、以下のようなsnake_caseを使用しない。

```ts
type AssetAccountDetail = {
  asset_type: string;
  balance_recording_unit: string;
  start_year_month: string;
  is_available: boolean;
};
```

API契約に従って、camelCaseで定義する。

---

### 29.46 関連データをACC-003から期待しない

ACC-003には、以下を含めない。

* 保有商品一覧
* 月末資産残高
* 商品別月末評価額
* 月末資産状況
* 資産推移

画面でこれらが必要な場合は、対応するAPIを別途利用する。

---

### 29.47 HOLDINGの場合

```text
balanceRecordingUnit = HOLDING
```

であっても、ACC-003レスポンスに保有商品一覧は含まれない。

画面で保有商品を表示する場合は、HLD-001を実行する。

概念的には、

```text
ACC-003
    ↓
資産口座詳細

HLD-001
    ↓
保有商品一覧
```

と分離する。

---

### 29.48 ACCOUNTの場合

```text
balanceRecordingUnit = ACCOUNT
```

の場合でも、ACC-003から月末残高を取得しない。

月末資産残高が必要な場合は、対応する月末資産APIを利用する。

---

### 29.49 Page・Query・表示コンポーネントの分離

画面実装では、例えば以下の責務に分離する。

```text
Page
    ↓
assetAccountId取得
エラー時画面遷移

Query Hook
    ↓
ACC-003実行
Query Cache管理

API Client
    ↓
HTTP通信

Detail Component
    ↓
詳細表示

Type
    ↓
API契約
```

1つのコンポーネントへすべての処理を直接記述しない。

---

### 29.50 概念的なディレクトリ構成

例えば、以下のように機能単位で整理できる。

```text
features/
└── asset-accounts/
    ├── api/
    │   └── getAssetAccountDetail.ts
    ├── components/
    │   └── AssetAccountDetail.tsx
    ├── hooks/
    │   └── useAssetAccountDetail.ts
    ├── types/
    │   └── assetAccount.ts
    └── pages/
        └── AssetAccountDetailPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

### 29.51 フロントエンドで行わないこと

ACC-003のReact・TypeScript実装では、以下をフロントエンドの責務としない。

* 利用者境界の最終保証
* 論理削除状態の判定
* 現在年月の業務判定
* 利用可能資産設定の期間判定
* 利用可能資産設定の重複判定
* データ不整合の補正
* DB内部コードの解釈
* 保有商品の自動取得
* 月末資産データの自動取得

フロントエンドは、

```text
assetAccountId
    ↓
ACC-003
    ↓
レスポンス表示
```

という責務を基本とする。

---

## 30. 設計上の補足

### 30.1 現在の資産口座詳細を取得するAPIとする理由

ACC-003は、指定した資産口座の現在の詳細情報を取得するAPIとする。

そのため、過去時点の状態を指定する`targetYearMonth`は受け付けない。

ACC-003で返却する`isAvailable`は、サーバー側の現在年月を基準として現在有効な利用可能資産設定から判定する。

概念的には、以下とする。

```text
assetAccountId
    ↓
現在の資産口座
    +
現在の利用可能資産区分
```

---

### 30.2 `isAvailable`を`asset_accounts`から取得しない理由

利用可能資産区分は、対象年月によって変化する履歴情報である。

そのため、

```text
asset_accounts.is_available
```

のような固定カラムから現在状態を取得しない。

ACC-003では、

```text
asset_account_available_settings
```

の期間履歴から、現在年月に有効な設定を取得する。

---

### 30.3 現在年月をクライアントから指定させない理由

ACC-003は、現在状態を確認するAPIである。

クライアントから以下のような年月を受け付けると、

```text
targetYearMonth
currentYearMonth
```

「現在詳細取得」と「過去時点詳細取得」の責務が混在する。

そのため、現在年月はサーバー側で決定する。

---

### 30.4 過去時点の利用可能資産区分をACC-003で扱わない理由

月末資産管理や目的達成判定では、対象年月時点の利用可能資産区分が必要になる。

しかし、それらは以下の各業務処理の対象年月に基づいて判定すべき情報である。

```text
月末資産
目的達成判定
```

ACC-003へ過去年月判定まで持たせず、各API側で対象年月を基準として利用可能資産設定を参照する。

---

### 30.5 現在有効な設定を1件とする理由

利用可能資産区分は、ある対象年月について1つの状態に決定できる必要がある。

同じ年月について、

```text
is_available = true
```

と

```text
is_available = false
```

の両方が有効になる状態は許容しない。

そのため、ACC-003では現在有効な設定が

```text
1件
```

であることを正常状態とする。

---

### 30.6 設定0件を`false`へ補完しない理由

ACC-002では、資産口座登録時に初期利用可能資産設定を必ず登録する。

そのため、利用開始済みの資産口座について現在有効な設定が存在しない状態は、正常な業務状態ではない。

以下のように補完すると、

```text
設定なし
    ↓
isAvailable = false
```

データ不整合が画面上では正常値として見えてしまう。

そのため、設定不存在を明示的なデータ不整合として扱う。

---

### 30.7 設定複数件で最新1件を採用しない理由

現在年月に有効な設定が複数存在する場合に、

```sql
ORDER BY start_year_month DESC
LIMIT 1
```

として最新1件だけを返却すると、期間重複というデータ不整合を隠蔽する。

そのため、ACC-003では複数件を検出した時点で正常レスポンスを返却しない。

---

### 30.8 利用可能資産設定の期間判定をDB側で行う理由

ACC-003で必要なのは、現在有効な利用可能資産設定だけである。

そのため、

```text
全履歴取得
    ↓
PHP側で期間判定
```

ではなく、

```text
現在年月を条件に
DBから対象設定取得
```

とする。

不要なデータ取得を避け、期間条件をQueryへ集約する。

---

### 30.9 設定取得を最大2件としてよい理由

ACC-003で確認したいのは、

```text
0件
1件
2件以上
```

のいずれかである。

複数件存在する場合に全履歴を取得する必要はない。

そのため、実装上は

```sql
LIMIT 2
```

として取得し、

```text
0
1
2
```

を判定してよい。

これにより、不整合検出と不要なデータ取得抑制を両立できる。

---

### 30.10 他利用者の資産口座を404とする理由

他利用者に属する`assetAccountId`が指定された場合に、

```text
FORBIDDEN
```

などを返却すると、そのIDの資産口座が実際に存在することを外部から推測できる。

そのため、

```text
存在しない
他利用者に属する
論理削除済み
```

を同じ

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 30.11 論理削除済み資産口座を通常詳細取得対象としない理由

ACC-003は、現在利用中の資産口座を確認するためのAPIである。

論理削除済み資産口座まで通常詳細取得対象へ含めると、

```text
現在利用中
過去に利用していた
```

という異なる状態を同一APIで扱うことになる。

Phase1では、ACC-003を利用中資産口座の詳細取得に限定する。

---

### 30.12 `isEnabled`を返却する理由

ACC-003の正常レスポンスでは、取得対象が利用中であるため、

```text
isEnabled = true
```

となる。

これは、DBの

```text
deleted_at = NULL
```

という内部実装をそのまま公開せず、業務上の利用状態として表現するためである。

---

### 30.13 `deletedAt`を返却しない理由

`deleted_at`は、Laravel SoftDeletesによる内部実装上のカラムである。

クライアントが必要とするのは、

```text
現在利用できるか
```

という業務上の状態であり、論理削除日時そのものではない。

そのため、

```text
isEnabled
```

として公開し、`deletedAt`は返却しない。

---

### 30.14 正常レスポンスで`isEnabled = true`のみとなる理由

ACC-003では、論理削除済み資産口座を取得対象から除外する。

そのため、

```text
isEnabled = false
```

となる正常レスポンスは存在しない。

それでもACC-001やACC-002とのレスポンス表現を揃えるため、ACC-003でも`isEnabled`を返却する。

---

### 30.15 `assetType`をAPI用文字列で返す理由

DB内部で`smallint`などのコード値を使用していても、APIでは以下のような意味のある文字列を返却する。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

これにより、クライアントをDB内部コードへ依存させない。

---

### 30.16 `balanceRecordingUnit`をAPI用文字列で返す理由

残高記録単位についても、DB内部コードではなく、

```text
ACCOUNT
HOLDING
```

として公開する。

これにより、React側でも型として明確に扱える。

---

### 30.17 保有商品を同時取得しない理由

`balanceRecordingUnit = HOLDING`の場合でも、ACC-003では保有商品一覧を返却しない。

資産口座詳細と保有商品一覧は、異なるリソース・業務責務である。

そのため、

```text
ACC-003
    → 資産口座詳細

HLD-001
    → 保有商品一覧
```

と分離する。

---

### 30.18 月末資産情報を同時取得しない理由

ACC-003へ以下を含めると、資産口座マスタ情報の詳細取得APIとして責務が広がりすぎる。

* 最新残高
* 商品別月末評価額
* 月末資産状況
* 資産推移

そのため、月末資産情報は専用APIへ委譲する。

---

### 30.19 編集画面の初期値取得に利用できる理由

ACC-003は、資産口座の現在の詳細状態を返す。

そのため、ACC-004 資産口座更新画面の初期表示にも利用できる。

概念的には、以下とする。

```text
ACC-003
    ↓
現在値取得
    ↓
編集フォーム初期値
```

ただし、ACC-003で返却する全項目がACC-004で更新可能とは限らない。

更新可能項目は、ACC-004側で定義する。

---

### 30.20 Repositoryを使用しない理由

ACC-003は、参照専用APIである。

そのため、

```text
INSERT
UPDATE
DELETE
```

を行わず、参照処理はQueryへ集約する。

概念的には、

```text
Query
    → 読み取り

Repository
    → 書き込み
```

という責務分離方針に従う。

---

### 30.21 UseCaseを設ける理由

ACC-003は単純な1テーブル取得ではなく、

```text
資産口座取得
    ↓
現在年月決定
    ↓
利用可能資産設定取得
    ↓
設定件数判定
    ↓
DTO生成
```

という業務フローを持つ。

そのため、ActionからQueryを直接組み合わせるのではなく、UseCaseへ処理の流れを集約する。

---

### 30.22 Queryを分ける理由

ACC-003では、

```text
AssetAccountQuery
```

と

```text
AssetAccountAvailableSettingQuery
```

を分離する。

資産口座本体の取得条件と、利用可能資産設定の期間条件は異なる責務を持つためである。

これにより、他APIからも必要なQueryを再利用しやすくする。

---

### 30.23 DTOを使用する理由

Queryで取得したEloquent ModelをそのままHTTP層へ渡すと、以下へAPI層が依存しやすくなる。

* DBカラム名
* Modelの関連
* Cast
* SoftDeletes状態

そのため、UseCaseからはレスポンスに必要な値だけを持つDTOを返却する。

---

### 30.24 API Resourceを使用する理由

API Resourceでは、

```text
snake_case
    ↓
camelCase
```

への変換や、

```text
bigint
    ↓
string
```

への変換、EnumのAPI表現への変換を行う。

DB Modelをそのままレスポンスへ返却しない。

---

### 30.25 FormRequestを作成しない理由

ACC-003には、

```text
リクエストボディ
クエリパラメータ
```

が存在しない。

専用FormRequestを作成しても空クラスとなるため、Phase1では作成しない。

`assetAccountId`は、ルート制約または共通パスパラメータ検証で扱う。

---

### 30.26 明示的なトランザクションを使用しない理由

ACC-003は参照専用であり、詳細表示のための通常参照処理である。

厳密な同一時点スナップショットを必要とする業務判断ではないため、

```php
DB::transaction()
```

を使用しない。

不要なトランザクションを追加しない。

---

### 30.27 `lockForUpdate()`を使用しない理由

ACC-003は、資産口座や利用可能資産設定を更新しない。

詳細表示のために更新処理をブロックする必要はないため、

```php
lockForUpdate()
```

を使用しない。

---

### 30.28 読み取り中の状態変化を許容する理由

ACC-003実行中に別APIによって以下が行われる可能性はある。

* 資産口座更新
* 資産口座無効化
* 利用可能資産設定変更

ただし、ACC-003は業務確定処理ではなく、現在状態の参照APIである。

そのため、厳密なSnapshot Isolationを要求せず、次回取得時に最新状態へ反映する方式とする。

---

### 30.29 Clockを分離してよい理由

ACC-003の`isAvailable`は、現在年月によって結果が変化する。

時刻取得を直接`now()`へ固定すると、テストで年月境界を扱う際に依存が強くなる。

そのため、必要に応じて以下へ分離してよい。

```text
Clock
DateProvider
```

ただし、Phase1で過剰になる場合は、Laravelの時刻固定機能を利用し、`now()`を直接使用してもよい。

---

### 30.30 データ不整合を500系として扱う理由

以下は、クライアント入力によって発生する状態ではない。

```text
現在有効な設定が0件
現在有効な設定が複数件
```

ACC-002やACC-006によって本来維持されるべき業務データ整合性が崩れている状態である。

そのため、Phase1ではサーバー側異常として500系で扱う。

---

### 30.31 データ不整合専用コードを返す理由

HTTPステータスだけでは、

```text
想定外例外
```

と

```text
利用可能資産設定の不整合
```

を区別できない。

そのため、内部調査やテストを容易にするため、以下のような独自エラーコードを定義する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

ただし、API全体でデータ不整合用コードを共通化する場合は、共通方針を優先する。

---

### 30.32 参照APIでデータを自動修復しない理由

ACC-003で不整合を検出した際に、

```text
設定なし
    → INSERT

設定重複
    → UPDATE
```

などを行うと、GET APIが副作用を持つことになる。

また、不整合の原因を隠してしまう。

そのため、ACC-003では検出だけを行い、データ修復は行わない。

---

### 30.33 Query Cacheを利用してよい理由

ACC-003は参照APIであり、React画面で同じ資産口座詳細を複数回参照する可能性がある。

そのため、TanStack QueryによるQuery Cacheを利用してよい。

ただし、更新系API成功後には適切にinvalidateする。

---

### 30.34 ACC-004成功後にキャッシュを無効化する理由

ACC-004によって資産口座の基本情報が変わると、ACC-003のキャッシュは古い状態になる。

そのため、対象資産口座の詳細Queryをinvalidateする。

---

### 30.35 ACC-005成功後に詳細キャッシュを破棄する理由

資産口座が無効化されると、ACC-003の通常取得対象から外れる。

古い詳細キャッシュを保持し続けると、無効化済み資産口座を利用中として表示する可能性がある。

そのため、ACC-005成功後は対象詳細Queryをinvalidateまたはremoveする。

---

### 30.36 ACC-006成功後にキャッシュを無効化する理由

ACC-006によって利用可能資産設定が変更されると、

```text
isAvailable
```

が変化する可能性がある。

そのため、ACC-003の詳細キャッシュを再取得対象とする。

---

### 30.37 現在年月の変化もキャッシュ失効要因となる理由

ACC-003の`isAvailable`は、DB更新だけでなく年月が変わることでも変化する可能性がある。

例えば、

```text
2026-08まで
isAvailable = true

2026-09から
isAvailable = false
```

という設定では、月をまたぐだけでレスポンスが変わる。

そのため、長期間固定するキャッシュには向かない。

Phase1ではサーバー側キャッシュを使用しない。

---

### 30.38 Optimistic Updateが不要な理由

ACC-003自体は参照APIである。

更新系APIの成功後は、

```text
invalidate
    ↓
ACC-003再取得
```

で最新状態へ同期できる。

詳細DTOを複数Mutationから手動で書き換えるより、Phase1ではサーバー状態を再取得する方式を基本とする。

---

### 30.39 利用者切替時にキャッシュを分離する理由

Phase1では、`X-User-Id`によって操作対象利用者を切り替える。

前の利用者で取得した資産口座詳細を、別利用者へ切り替えた後も画面表示してはならない。

そのため、Query Keyへ利用者情報を含める、または利用者切替時に関連Queryを無効化するなど、キャッシュ上でも利用者境界を維持する。

---

### 30.40 ACC-001との表現を揃える理由

ACC-001とACC-003で同じ意味を持つ項目については、同じ表現を使用する。

例えば、

```text
id
name
assetType
balanceRecordingUnit
isAvailable
startYearMonth
isEnabled
```

について、一覧と詳細で意味や型を変えない。

これにより、React側で共通型や共通表示部品を利用しやすくする。

---

### 30.41 ACC-002との表現を揃える理由

ACC-002成功時に返却した資産口座情報と、ACC-003で取得する資産口座詳細についても、同一項目は同じ形式を使用する。

登録直後と再取得後で表現が変わらないようにする。

---

### 30.42 Phase1では詳細取得を広げすぎない

ACC-003では、以下を追加しない。

* 利用可能資産設定履歴一覧
* 保有商品一覧
* 最新月末残高
* 商品別評価額
* 資産推移
* 過去年月指定
* 無効化済み資産口座取得
* ETag
* Last-Modified
* サーバー側キャッシュ

Phase1では、

```text
指定資産口座の現在詳細
+
現在の利用可能資産区分
```

を取得することへ責務を限定する。


---

## 31. 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)

---

# ACC-004 資産口座更新

## 1. 概要

操作対象となる利用者に帰属する指定された資産口座の基本情報を更新する。

ACC-004では、資産口座の通常属性を更新する。

主な更新対象は、以下とする。

- 資産口座名
- 資産種別

一方、以下はACC-004では更新しない。

- 利用者
- 残高記録単位
- 利用開始年月
- 利用可能資産区分
- 利用状態

利用可能資産区分の変更は、履歴管理を伴うためACC-006 利用可能資産設定登録APIで扱う。

資産口座の無効化は、通常属性更新とは異なる業務操作としてACC-005 資産口座無効化APIで扱う。

概念的には、以下の責務分離とする。

```text
ACC-004
    ↓
資産口座の通常属性更新
    ├─ name
    └─ asset_type

ACC-005
    ↓
資産口座無効化

ACC-006
    ↓
利用可能資産区分変更
```

本APIでは、`asset_accounts`のみを更新し、`asset_account_available_settings`は更新しない。

---

## 2. ユースケース

利用者は、登録済み資産口座の基本情報に変更があった場合にACC-004を使用する。

例えば、以下の場合に使用する。

- 資産口座名を変更したい
- 登録時に誤った資産種別を設定したため修正したい

概念的な画面利用は、以下とする。

```text
ACC-001
資産口座一覧取得
    ↓
資産口座選択
    ↓
ACC-003
資産口座詳細取得
    ↓
編集画面
    ↓
利用者が編集
    ↓
ACC-004
資産口座更新
```

ACC-004成功後は、ACC-001またはACC-003を再取得することで、最新状態を確認できる。

---

### 2.1 更新対象としない操作

以下の操作にはACC-004を使用しない。

```text
残高記録単位変更
    → ACC-004では扱わない

利用開始年月変更
    → ACC-004では扱わない

利用可能資産区分変更
    → ACC-006

資産口座無効化
    → ACC-005
```

異なる業務上の意味を持つ操作を1つの更新APIへ集約しない。

---

## 3. エンドポイント

```http
PATCH /api/v1/asset-accounts/{assetAccountId}
```

`assetAccountId`には、更新対象となる資産口座IDを指定する。

例：

```http
PATCH /api/v1/asset-accounts/10
```

---

## 4. HTTPメソッド

```text
PATCH
```

資産口座リソース全体を置き換えるのではなく、更新可能な一部項目だけを変更するため、`PATCH`を使用する。

概念的には、

```text
asset_accounts
    ↓
name
asset_type
    ↓
部分更新
```

とする。

以下のようなリソース全体置換を前提としない。

```http
PUT /api/v1/asset-accounts/{assetAccountId}
```

---

### 4.1 ACC-005との分離

資産口座無効化は、ACC-004のリクエストボディへ

```json
{
  "isEnabled": false
}
```

のような値を指定して実行しない。

通常属性更新とライフサイクル変更を分離し、

```text
ACC-004
PATCH /api/v1/asset-accounts/{assetAccountId}

ACC-005
資産口座無効化用エンドポイント
```

として扱う。

ACC-005の具体的なURLは、ACC-005詳細設計に従う。

---

### 4.2 更新による副作用

ACC-004成功時は、対象となる`asset_accounts`1件を更新する。

通常のLaravel・Eloquent更新に従い、`updated_at`も更新される。

一方、以下の関連データはACC-004では更新しない。

- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `assessment_histories`

---

## 5. 利用者コンテキスト

本APIは利用者依存APIのため、`X-User-Id`を必須とする。

操作対象となる利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者コンテキストの特定、`X-User-Id`の検証および利用者境界については、[API共通方針](../api-common-policy.md)に従う。

指定された`assetAccountId`に対応する資産口座が操作対象利用者に帰属する場合のみ、更新できる。

概念的には、以下の条件を満たす必要がある。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 5.1 利用者境界

ACC-004では、`assetAccountId`だけを条件として更新対象を取得してはならない。

以下のような取得は基本としない。

```text
asset_accounts.id
    = assetAccountId
```

操作対象利用者IDも含めて、

```text
assetAccountId
+
操作対象利用者ID
+
利用中状態
```

を条件として更新対象を取得する。

これにより、他利用者の資産口座を誤って更新することを防止する。

---

### 5.2 他利用者の資産口座

指定された`assetAccountId`が他利用者に帰属する場合は、対象が存在しないものとして扱う。

概念的には、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

とする。

以下のような他利用者専用エラーは返却しない。

```text
FORBIDDEN
OTHER_USER_ASSET_ACCOUNT
```

これにより、他利用者の資産口座が存在するかどうかを外部から推測しにくくする。

---

### 5.3 論理削除済み資産口座

ACC-004では、利用中の資産口座のみを更新対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は更新対象に含めない。

論理削除済み資産口座を指定した場合も、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

再有効化はACC-004の責務に含めない。

---

### 5.4 userIdをリクエストボディから受け付けない

ACC-004では、

```text
userId
user_id
```

を更新項目として受け付けない。

資産口座の所有利用者を変更する機能は提供しない。

例えば、

```json
{
  "userId": "2"
}
```

のような指定によって資産口座を別利用者へ移動できない。

利用者境界は、

```text
X-User-Id
    ↓
UserContext
    ↓
更新対象取得条件
```

によって保証する。

---

### 5.5 利用可能資産設定の利用者境界

ACC-004では、`asset_account_available_settings`を更新しない。

利用可能資産区分を変更する場合は、ACC-006で操作対象利用者に帰属する資産口座であることを確認したうえで、設定履歴を更新する。

ACC-004から利用可能資産設定へ暗黙的な変更を加えない。

---

### 5.6 保有商品の利用者境界

対象資産口座に`holding_assets`が存在していても、ACC-004では保有商品を更新しない。

保有商品の更新は、各HLD APIの利用者境界に従って行う。

資産口座情報の更新を理由として、配下の保有商品を一括更新しない。

---

### 5.7 X-User-Idが不正な場合

以下の場合は、資産口座検索および更新処理へ進まない。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正
- 指定された利用者が存在しない
- 指定された利用者が論理削除済み

この場合、`asset_accounts`を更新しない。

利用者コンテキストに関するエラーコードおよびHTTPステータスは、API共通方針に従う。

---

### 5.8 1リクエスト1利用者

1回のACC-004リクエストでは、`X-User-Id`で指定された1利用者の資産口座だけを扱う。

URLやクエリパラメータから別利用者を指定する方式は採用しない。

例えば、

```text
/api/v1/users/{userId}/asset-accounts/{assetAccountId}
```

や、

```text
?userId=1
```

は使用しない。

利用者の指定経路は、`X-User-Id`へ統一する。

---

## 6. パスパラメータ

本APIでは、更新対象となる資産口座を指定するために以下のパスパラメータを使用する。

| パラメータ | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `assetAccountId` | string | ○ | 更新対象となる資産口座ID |

エンドポイントは、以下とする。

```http
PATCH /api/v1/asset-accounts/{assetAccountId}
```

---

### 6.1 assetAccountId

`assetAccountId`には、更新対象となる資産口座IDを指定する。

例：

```http
PATCH /api/v1/asset-accounts/10
```

APIでは主キーを文字列として扱うが、値としては正の整数形式を前提とする。

正常例：

```text
1
10
123
```

不正例：

```text
0
-1
abc
1.5
```

---

### 6.2 利用者境界

`assetAccountId`だけでは更新対象を確定しない。

実際の更新対象取得では、操作対象利用者IDと組み合わせて検索する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

他利用者に属する資産口座IDが指定された場合は、対象が存在しないものとして扱う。

---

## 7. クエリパラメータ

なし。

ACC-004では、更新対象は`assetAccountId`で指定し、更新内容はリクエストボディで指定する。

以下のような更新動作を切り替えるためのクエリパラメータは使用しない。

- `userId`
- `isEnabled`
- `force`
- `includeDeleted`
- `targetYearMonth`

---

## 8. リクエストヘッダー

以下のリクエストヘッダーを使用する。

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Content-Type` | ○ | `application/json` |
| `Accept` | ○ | `application/json` |

リクエスト例：

```http
PATCH /api/v1/asset-accounts/10
Content-Type: application/json
Accept: application/json
X-User-Id: 1
```

---

### 8.1 X-User-Id

`X-User-Id`は、API共通方針に従って検証する。

本APIの処理開始前に、共通Middlewareで以下を確認する。

- ヘッダーが指定されていること
- IDが共通方針で定めた形式であること
- 利用者が存在すること
- 利用者が論理削除されていないこと

検証済み利用者IDを利用者コンテキストへ保持し、Action以降で使用する。

---

## 9. リクエストボディ

ACC-004では、更新可能な資産口座の通常属性を指定する。

更新可能項目は、以下とする。

- `name`
- `assetType`

リクエスト例：

```json
{
  "name": "メイン証券口座",
  "assetType": "SECURITIES"
}
```

`PATCH`であるため、更新したい項目だけを指定できる。

例えば、資産口座名だけを変更する場合は、以下とする。

```json
{
  "name": "メイン証券口座"
}
```

資産種別だけを変更する場合は、以下とする。

```json
{
  "assetType": "SECURITIES"
}
```

---

### 9.1 リクエスト項目

| 項目 | 型 | 必須 | NULL | 説明 |
|---|---|:---:|:---:|---|
| `name` | string | △ | × | 資産口座名 |
| `assetType` | string | △ | × | 資産種別 |

`△`は、項目単体では任意だが、少なくとも1項目の指定を必須とすることを表す。

---

### 9.2 name

資産口座名を指定する。

例：

```json
{
  "name": "メイン証券口座"
}
```

同一利用者内では、他の資産口座と同じ名前へ変更できない。

ただし、更新対象自身が現在使用している名前と同一値を指定した場合は、重複とは扱わない。

概念的には、

```text
同一user_id
+
同一name
+
id != 更新対象assetAccountId
```

に該当する資産口座が存在する場合に重複とする。

論理削除済み資産口座も同名重複判定の対象とする。

---

### 9.3 assetType

資産種別を指定する。

指定可能な値は、ACC-002と同じく以下とする。

| 値 | 意味 |
|---|---|
| `CASH` | 現金 |
| `BANK` | 銀行 |
| `SECURITIES` | 証券 |
| `IDECO` | iDeCo |
| `CORPORATE_DC` | 企業型DC |
| `OTHER` | その他 |

リクエストでは、DB内部の数値コードを直接指定しない。

---

### 9.4 PATCHとしての部分更新

ACC-004では、指定された項目だけを更新する。

例えば、

```json
{
  "name": "メイン証券口座"
}
```

の場合は、

```text
name
```

だけを更新し、

```text
asset_type
```

は変更しない。

同様に、

```json
{
  "assetType": "SECURITIES"
}
```

の場合は、

```text
asset_type
```

だけを更新し、

```text
name
```

は変更しない。

未指定項目を

```text
NULL
空文字
デフォルト値
```

へ置き換えない。

---

### 9.5 空リクエスト

以下のような更新項目を1件も含まないリクエストは受け付けない。

```json
{}
```

ACC-004は更新APIであるため、少なくとも

```text
name
assetType
```

のいずれか1項目を指定することを必須とする。

---

### 9.6 同一値の指定

現在の値と同じ値が指定された場合は、入力自体は有効として扱ってよい。

例えば、現在の資産口座名が

```text
証券口座
```

の状態で、

```json
{
  "name": "証券口座"
}
```

を送信しても、バリデーションエラーとはしない。

実際にUPDATEを発行するかどうかは、Laravel実装方針で定義する。

---

### 9.7 リクエストで受け付けない項目

以下の項目は、ACC-004では受け付けない。

- `id`
- `userId`
- `user_id`
- `balanceRecordingUnit`
- `balance_recording_unit`
- `startYearMonth`
- `start_year_month`
- `isAvailable`
- `is_available`
- `isEnabled`
- `deletedAt`
- `deleted_at`
- `createdAt`
- `created_at`
- `updatedAt`
- `updated_at`

これらは、ACC-004の更新責務に含めない。

---

### 9.8 balanceRecordingUnitを受け付けない

残高記録単位は、資産口座登録後に変更すると、既存の月末資産データや保有商品管理方法に影響する。

そのため、Phase1ではACC-004の更新対象としない。

以下のようなリクエストは受け付けない。

```json
{
  "balanceRecordingUnit": "ACCOUNT"
}
```

---

### 9.9 startYearMonthを受け付けない

利用開始年月を後から変更すると、既存の

- 利用可能資産設定履歴
- 月末資産データ
- 対象年月判定

との整合性へ影響する。

そのため、ACC-004では`startYearMonth`を更新しない。

---

### 9.10 isAvailableを受け付けない

利用可能資産区分は、履歴として管理する。

そのため、

```json
{
  "isAvailable": false
}
```

のような指定によってACC-004から変更しない。

利用可能資産区分の変更は、ACC-006で扱う。

---

### 9.11 isEnabledを受け付けない

資産口座の無効化は、ACC-005の責務とする。

以下のような状態変更はACC-004で扱わない。

```json
{
  "isEnabled": false
}
```

---

## 10. バリデーション

ACC-004では、主に以下を検証する。

```text
X-User-Id
assetAccountId
name
assetType
更新項目件数
```

資産口座存在確認や利用者境界確認は、更新対象取得時に行う。

---

### 10.1 X-User-Id

`X-User-Id`について、以下を検証する。

- 必須であること
- 共通ID形式に一致すること
- 正の整数として扱えること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

不正な場合は、資産口座検索や更新処理へ進まない。

---

### 10.2 assetAccountId

`assetAccountId`は、正の整数形式であることを必須とする。

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

形式不正の場合は、更新対象検索へ進まない。

---

### 10.3 更新項目が1件以上指定されていること

ACC-004では、

```text
name
assetType
```

のうち、少なくとも1項目が指定されていることを必須とする。

以下は不正とする。

```json
{}
```

概念的には、

```text
name未指定
AND
assetType未指定
    ↓
VALIDATION_ERROR
```

とする。

---

### 10.4 name

`name`が指定された場合は、以下を検証する。

- 文字列であること
- `NULL`ではないこと
- 空文字ではないこと
- テーブル定義で定めた最大文字数以内であること

概念的には、FormRequestで

```text
sometimes
required
string
max
```

相当の検証を行う。

---

### 10.5 nameの重複判定

`name`が指定された場合は、同一利用者内に同名の別資産口座が存在しないことを確認する。

概念的な重複条件は、以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = request.name

AND

asset_accounts.id
    != assetAccountId
```

論理削除済み資産口座も重複判定対象とする。

---

### 10.6 自分自身の現在名

更新対象自身の現在の`name`と同じ値を指定した場合は、重複エラーとしない。

例えば、

```text
更新対象
id = 10
name = 証券口座
```

の状態で、

```json
{
  "name": "証券口座"
}
```

を送信した場合は正常な入力とする。

---

### 10.7 論理削除済み同名資産口座

同一利用者に、

```text
name = 証券口座
deleted_at IS NOT NULL
```

の別資産口座が存在する場合も、同名への変更を許可しない。

ACC-002と同じ名前一意性ルールを適用する。

---

### 10.8 他利用者の同名資産口座

他利用者にのみ同名資産口座が存在する場合は、重複エラーとしない。

例えば、

```text
User A
    証券口座

User B
    銀行口座
```

の状態で、User Bの資産口座名を

```text
証券口座
```

へ変更することは許可する。

---

### 10.9 assetType

`assetType`が指定された場合は、以下を検証する。

- 文字列であること
- `NULL`ではないこと
- 定義済みの列挙値であること

指定可能な値は、以下とする。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

未定義値は受け付けない。

例：

```json
{
  "assetType": "CRYPTO"
}
```

は不正とする。

---

### 10.10 未指定項目は検証対象外とする

PATCHであるため、指定されていない更新可能項目について必須チェックを行わない。

例えば、

```json
{
  "name": "メイン証券口座"
}
```

の場合に、

```text
assetType未指定
```

をエラーとしない。

---

### 10.11 NULL

ACC-004の更新可能項目は、`NULL`を許可しない。

以下は不正とする。

```json
{
  "name": null
}
```

```json
{
  "assetType": null
}
```

「項目を変更しない」場合は、`NULL`を送信するのではなく、その項目自体をリクエストから省略する。

---

### 10.12 空文字

`name`に空文字を指定することはできない。

例えば、

```json
{
  "name": ""
}
```

は不正とする。

必要に応じて、空白のみの文字列についてもAPI共通の文字列入力方針に従って不正とする。

---

### 10.13 更新対象資産口座の存在確認

形式検証済みの`assetAccountId`について、操作対象利用者に属する有効な資産口座が存在することを確認する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

該当しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 10.14 他利用者の資産口座

指定された`assetAccountId`が他利用者に属する場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

他利用者に属することを専用エラーで公開しない。

---

### 10.15 論理削除済み資産口座

対象資産口座が

```text
deleted_at IS NOT NULL
```

の場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

ACC-004から再有効化しない。

---

### 10.16 更新対象外項目

未定義項目を拒否するAPI共通方針を採用する場合は、以下のような項目が含まれているリクエストをバリデーションエラーとする。

```json
{
  "name": "証券口座",
  "isAvailable": false
}
```

あるいは、

```json
{
  "balanceRecordingUnit": "ACCOUNT"
}
```

これにより、クライアントがACC-004で変更できない項目を誤って送信しても黙って無視しない。

---

### 10.17 バリデーション失敗時

入力値検証に失敗した場合は、更新処理へ進まない。

特に、

```text
asset_accounts
```

の内容を変更しない。

複数の項目エラーが存在する場合は、API共通方針に従って`error.details`へ複数件返却してよい。

---

## 11. 業務ルール

ACC-004では、操作対象利用者に帰属する利用中の資産口座について、通常属性のみを更新する。

更新対象は、以下とする。

```text
name
asset_type
```

以下の項目はACC-004では更新しない。

```text
user_id
balance_recording_unit
start_year_month
deleted_at
```

また、利用可能資産区分を管理する

```text
asset_account_available_settings
```

も更新しない。

---

### 11.1 操作対象利用者に帰属する資産口座のみ更新できる

更新対象となる資産口座は、以下の条件をすべて満たすものとする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

`assetAccountId`だけを条件として資産口座を取得してはならない。

---

### 11.2 他利用者の資産口座は更新できない

指定された`assetAccountId`が他利用者に帰属する場合は、対象が存在しないものとして扱う。

概念的には、

```text
X-User-Id = 1

assetAccountId = 10

asset_accounts.id = 10
asset_accounts.user_id = 2
    ↓
更新対象外
    ↓
ASSET_ACCOUNT_NOT_FOUND
```

とする。

他利用者に属することを示す専用エラーは返却しない。

---

### 11.3 論理削除済み資産口座は更新できない

ACC-004では、利用中の資産口座のみを更新対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は更新対象に含めない。

論理削除済み資産口座を再有効化する処理も行わない。

---

### 11.4 更新可能項目

ACC-004で更新可能な項目は、以下とする。

```text
name
asset_type
```

APIでは、それぞれ

```text
name
assetType
```

として受け付ける。

---

### 11.5 nameのみの更新

`name`だけが指定された場合は、資産口座名だけを更新する。

例えば、

```json
{
  "name": "メイン証券口座"
}
```

の場合は、

```text
asset_accounts.name
```

のみを変更する。

```text
asset_accounts.asset_type
```

は変更しない。

---

### 11.6 assetTypeのみの更新

`assetType`だけが指定された場合は、資産種別だけを更新する。

例えば、

```json
{
  "assetType": "SECURITIES"
}
```

の場合は、

```text
asset_accounts.asset_type
```

のみを変更する。

```text
asset_accounts.name
```

は変更しない。

---

### 11.7 複数項目の更新

`name`と`assetType`の両方が指定された場合は、両方を同一リクエストで更新する。

例えば、

```json
{
  "name": "メイン証券口座",
  "assetType": "SECURITIES"
}
```

の場合は、

```text
name
asset_type
```

を更新する。

---

### 11.8 未指定項目は変更しない

PATCHであるため、リクエストに含まれない項目は現在値を保持する。

例えば、

```json
{
  "name": "メイン証券口座"
}
```

の場合に、

```text
asset_type
```

を

```text
NULL
デフォルト値
```

へ変更しない。

---

### 11.9 同一値の指定

現在値と同じ値が指定された場合も、入力自体は正常とする。

例えば、

```text
現在
name = 証券口座
```

の状態で、

```json
{
  "name": "証券口座"
}
```

を送信しても、業務エラーとはしない。

実際にDB UPDATEが発生するかは、EloquentのDirty判定に従ってよい。

---

### 11.10 同一利用者内で資産口座名を重複させない

`name`を変更する場合は、同一利用者内に同名の別資産口座が存在しないことを必須とする。

概念的には、

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = request.name

AND

asset_accounts.id
    != assetAccountId
```

に該当する資産口座が存在する場合は、更新不可とする。

---

### 11.11 論理削除済み資産口座も名前重複判定対象とする

ACC-002と同様に、論理削除済みの資産口座も資産口座名の重複判定対象とする。

例えば、

```text
id = 20
name = 証券口座
deleted_at IS NOT NULL
```

という別資産口座が存在する場合、利用中の資産口座を

```text
証券口座
```

へ変更することは許可しない。

---

### 11.12 更新対象自身の現在名は重複としない

更新対象自身が現在持っている名前を再度指定した場合は、重複エラーとしない。

概念的には、

```text
id != assetAccountId
```

を重複確認条件に含める。

---

### 11.13 異なる利用者間では同名を許可する

資産口座名の一意性は、システム全体ではなく利用者単位とする。

例えば、

```text
User A
    証券口座

User B
    銀行口座
```

という状態で、User Bの資産口座を

```text
証券口座
```

へ変更することは許可する。

---

### 11.14 assetTypeは定義済み値のみ更新できる

`assetType`には、APIで定義した以下の値のみ指定できる。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

DB内部で`smallint`などを使用していても、クライアントから数値コードを直接指定させない。

---

### 11.15 assetType変更によって他データを自動変更しない

資産種別を変更しても、以下の関連データを自動変更しない。

- 保有商品
- 月末資産残高
- 商品別月末評価額
- 利用可能資産設定
- 月末資産状況
- 目的達成判定履歴

ACC-004では、`asset_accounts.asset_type`だけを更新する。

---

### 11.16 残高記録単位は更新しない

```text
balance_recording_unit
```

は、ACC-004の更新対象外とする。

残高記録単位を変更すると、

```text
ACCOUNT
    ↕
HOLDING
```

に伴って、

- 月末資産残高
- 保有商品
- 商品別月末評価額

などの既存データとの整合性ルールが必要になるためである。

Phase1では、登録後の変更を許可しない。

---

### 11.17 利用開始年月は更新しない

```text
start_year_month
```

も、ACC-004の更新対象外とする。

利用開始年月を変更すると、

- 利用可能資産設定履歴
- 月末資産対象判定
- 過去月の資産記録

との整合性に影響するためである。

---

### 11.18 利用可能資産区分は更新しない

```text
isAvailable
```

は、ACC-004では受け付けない。

利用可能資産区分は、

```text
asset_account_available_settings
```

によって履歴管理する。

そのため、変更はACC-006で行う。

---

### 11.19 利用状態は更新しない

ACC-004では、

```text
isEnabled
deleted_at
```

を変更しない。

資産口座無効化は、ACC-005で扱う。

通常属性更新とライフサイクル変更を同じAPIへ混在させない。

---

### 11.20 user_idは変更しない

資産口座の所有利用者を変更する機能は提供しない。

以下のような変更はACC-004の責務に含めない。

```text
User Aの資産口座
    ↓
User Bへ移管
```

`asset_accounts.user_id`は変更しない。

---

### 11.21 関連データを更新しない

ACC-004では、以下の関連テーブルを更新しない。

```text
asset_account_available_settings
holding_assets
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
assessment_histories
```

資産口座の通常属性変更だけに責務を限定する。

---

## 12. 処理フロー

概念的な処理フローは、以下とする。

```text
リクエスト受付
    ↓
リクエストID生成
    ↓
X-User-Id検証
    ↓
操作対象利用者の特定
    ↓
assetAccountId形式検証
    ↓
リクエストボディ検証
    ↓
操作対象利用者に帰属する
有効な資産口座を取得
    ↓
存在しない
    → ASSET_ACCOUNT_NOT_FOUND
    ↓
name指定あり？
    ├─ Yes
    │    ↓
    │  同名重複確認
    │    ↓
    │  重複あり
    │    → ASSET_ACCOUNT_NAME_ALREADY_EXISTS
    │
    └─ No
    ↓
指定された更新項目のみ反映
    ↓
asset_accounts更新
    ↓
更新結果DTO生成
    ↓
APIレスポンス変換
    ↓
正常レスポンス返却
```

---

### 12.1 利用者コンテキスト確認

共通Middlewareで`X-User-Id`を検証し、操作対象利用者を特定する。

利用者コンテキストが不正な場合は、更新対象検索へ進まない。

---

### 12.2 assetAccountId検証

パスパラメータの

```text
assetAccountId
```

が正の整数形式であることを確認する。

形式不正の場合は、DB検索へ進まない。

---

### 12.3 リクエストボディ検証

以下について検証する。

```text
name
assetType
```

また、少なくとも1項目が指定されていることを確認する。

空リクエストの場合は、更新処理へ進まない。

---

### 12.4 更新対象取得

操作対象利用者IDと`assetAccountId`を条件として、利用中の資産口座を取得する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

---

### 12.5 対象不存在

該当する資産口座が存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下を同じ扱いとする。

- 存在しない資産口座ID
- 他利用者に属する資産口座
- 論理削除済み資産口座

---

### 12.6 資産口座名重複確認

`name`が指定されている場合のみ、資産口座名の重複確認を行う。

概念的には、

```text
user_id
    = 操作対象利用者ID

AND

name
    = request.name

AND

id
    != assetAccountId
```

とする。

論理削除済み資産口座も検索対象に含める。

---

### 12.7 nameが未指定の場合

`name`がリクエストに含まれていない場合は、資産口座名重複確認を行わない。

不要なDB検索を実行しない。

---

### 12.8 更新値生成

リクエストに指定された項目だけを更新対象として生成する。

概念的には、

```text
name指定あり
    → name更新

assetType指定あり
    → asset_type更新
```

とする。

未指定項目は現在値を保持する。

---

### 12.9 資産口座更新

対象となる

```text
asset_accounts
```

1件を更新する。

概念的には、

```sql
UPDATE asset_accounts
SET
    指定された項目
WHERE
    id = assetAccountId
```

となる。

実際には、Eloquent Modelを通して更新する。

---

### 12.10 updated_at

通常のEloquent更新によって変更が発生した場合は、

```text
updated_at
```

も更新される。

リクエストから`updatedAt`を指定させない。

---

### 12.11 関連テーブル非更新

ACC-004成功時も、以下を更新しない。

```text
asset_account_available_settings
holding_assets
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
assessment_histories
```

---

### 12.12 更新結果生成

更新後の資産口座から、APIレスポンスに必要な結果DTOを生成する。

DB Modelをそのままレスポンスへ返却しない。

必要な項目だけをDTOおよびAPI Resourceへ渡す。

---

## 13. トランザクション境界

ACC-004では、Phase1において明示的なDBトランザクションを必須としない。

更新対象が、

```text
asset_accounts
```

の1レコードだけであり、関連テーブルを同時更新しないためである。

---

### 13.1 更新対象

正常時に更新する業務データは、

```text
asset_accounts 1件
```

のみとする。

主な更新対象カラムは、

```text
name
asset_type
updated_at
```

となる。

---

### 13.2 単一UPDATEで完結する

ACC-004の更新処理は、概念的には

```text
対象取得
    ↓
重複確認
    ↓
asset_accounts UPDATE
```

で完結する。

複数テーブルを原子的に更新する必要がないため、

```php
DB::transaction()
```

を必須としない。

---

### 13.3 重複確認とUPDATE

`name`を更新する場合は、

```text
重複確認
    ↓
UPDATE
```

という複数SQLになる可能性がある。

ただし、重複確認だけで名前一意性を保証しない。

データベースの

```text
user_id
+
name
```

に対するUNIQUE制約を最終防衛線として使用する。

そのため、単純な重複確認とUPDATEを1トランザクションへ入れるだけで競合を防止しようとはしない。

---

### 13.4 同時更新による名前競合

例えば、同一利用者の別々の資産口座を同じ名前へ変更する2リクエストが同時実行された場合、

```text
Request A
重複なし

Request B
重複なし

Request A
UPDATE

Request B
UPDATE
```

となる可能性がある。

この場合は、DBのUNIQUE制約によって一方の更新を失敗させる。

具体的な例外変換は、後続の「排他制御」「エラーレスポンス」「Laravel実装方針」で定義する。

---

### 13.5 Repository単位でトランザクションを開始しない

ACC-004では、Repository内で独自に

```php
DB::transaction()
```

を開始しない。

将来的にACC-004の業務要件が拡張され、

```text
asset_accounts更新
+
別テーブル更新
```

を原子的に扱う必要が生じた場合は、UseCaseをトランザクション境界とする。

---

### 13.6 将来的にトランザクションを検討するケース

将来的に、資産口座更新と同時に

- 変更履歴の登録
- 監査用テーブルへの書き込み
- 関連設定の更新

などを行う場合は、明示的なトランザクションを導入する。

Phase1では、通常属性1レコード更新という現在の責務に合わせて不要なトランザクションを追加しない。

---

## 14. 排他制御

ACC-004では、同一資産口座に対する同時更新と、同一利用者内の資産口座名競合を考慮する。

Phase1では、明示的な行ロックや楽観ロックは導入しない。

資産口座名の一意性については、

```text
アプリケーション側の事前重複確認
+
データベースのUNIQUE制約
```

によって保証する。

---

### 14.1 同一資産口座への同時更新

同じ`assetAccountId`に対して、複数のACC-004が同時実行される可能性がある。

例えば、

```text
Request A
nameを更新

Request B
assetTypeを更新
```

のようなケースが考えられる。

Phase1では、以下のような明示的な排他制御は使用しない。

```php
lockForUpdate()
```

また、

```text
version
updated_at比較
ETag
If-Match
```

などによる楽観ロックも導入しない。

---

### 14.2 Last Write Wins

同一項目に対する同時更新が発生した場合は、Phase1では後から完了した更新内容が最終状態となる可能性を許容する。

概念的には、

```text
Request A
name = 証券口座A

Request B
name = 証券口座B

        ↓

最終的に
後から反映された値が残る
```

となる。

ACC-004は高頻度な競合更新を前提としたAPIではないため、Phase1では複雑な競合制御を追加しない。

---

### 14.3 資産口座名の同時競合

同一利用者に属する別々の資産口座を、同じ名前へ変更する複数リクエストが同時実行される可能性がある。

例えば、

```text
資産口座A
name = A

資産口座B
name = B
```

の状態から、

```text
Request A
資産口座A
name = 証券口座

Request B
資産口座B
name = 証券口座
```

を同時実行するケースである。

---

### 14.4 事前重複確認だけでは保証しない

以下のように、両リクエストが事前重複確認を通過する可能性がある。

```text
Request A
重複なし確認

Request B
重複なし確認

Request A
UPDATE

Request B
UPDATE
```

そのため、アプリケーション側の重複確認だけを一意性保証とはしない。

---

### 14.5 UNIQUE制約

同一利用者内の資産口座名については、データベース側でも

```text
user_id
+
name
```

の組み合わせに一意性を保証する。

これにより、並行更新時にも同名資産口座が複数存在する状態を防止する。

---

### 14.6 論理削除済み資産口座も一意性対象とする

ACC-002およびACC-004の業務ルールでは、論理削除済み資産口座も同名重複判定対象とする。

そのため、一意性制約も

```text
deleted_at IS NULL
```

だけを対象とする部分UNIQUE制約にはしない。

利用中・論理削除済みを問わず、

```text
user_id
+
name
```

の重複を防止する。

---

### 14.7 UNIQUE制約違反

同時更新によってUNIQUE制約違反が発生した場合は、PostgreSQLの例外をそのまま返却しない。

業務上の

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

クライアントから見た結果は、事前重複確認で検出した場合と同じものとする。

---

### 14.8 lockForUpdateを使用しない

ACC-004では、Phase1で

```php
lockForUpdate()
```

を使用しない。

行ロックを使用しても、別々の資産口座を同じ名前へ変更する競合については、ロック対象が異なるため名前一意性を保証できない。

資産口座名の競合は、DBのUNIQUE制約で最終的に保証する。

---

### 14.9 利用者行をロックしない

同一利用者内の資産口座更新を直列化するために、

```text
users
```

の行をロックする方式は採用しない。

利用者行をロックすると、同一利用者に関する他の処理まで不要に待機させる可能性があるためである。

---

### 14.10 将来的な競合制御

将来的に、

- 複数端末からの編集
- 複数ユーザーによる共同編集
- 更新競合の検出
- 編集内容の上書き防止

が必要になった場合は、

```text
versionカラム
updated_at比較
ETag
If-Match
```

などによる楽観ロックの導入を検討する。

Phase1では対象外とする。

---

## 15. 成功レスポンス

資産口座更新に成功した場合は、

```text
200 OK
```

を返却する。

正常時は、API共通方針に従った成功レスポンス形式を使用する。

概念例：

```json
{
  "data": {
    "id": "10",
    "name": "メイン証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

ACC-004で直接更新する項目は

```text
name
assetType
```

だけだが、更新後の資産口座状態をクライアントが確認できるよう、資産口座詳細として必要な情報を返却する。

---

### 15.1 レスポンス項目

正常時の`data`配下には、以下を返却する。

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `id` | string | × | 更新した資産口座ID |
| `name` | string | × | 更新後の資産口座名 |
| `assetType` | string | × | 更新後の資産種別 |
| `balanceRecordingUnit` | string | × | 残高記録単位 |
| `isAvailable` | boolean | × | 現在の利用可能資産区分 |
| `startYearMonth` | string | × | 利用開始年月。`YYYY-MM`形式 |
| `isEnabled` | boolean | × | 利用状態。正常時は`true` |

---

### 15.2 id

更新した資産口座IDを文字列として返却する。

例：

```json
{
  "id": "10"
}
```

データベース上で`bigint`として保持していても、API共通方針に従ってstringで返却する。

---

### 15.3 name

更新後の資産口座名を返却する。

`name`がリクエストで未指定だった場合は、更新前から保持している現在値を返却する。

---

### 15.4 assetType

更新後の資産種別をAPI用文字列として返却する。

例えば、

```json
{
  "assetType": "SECURITIES"
}
```

とする。

DB内部の数値コードをそのまま返却しない。

---

### 15.5 balanceRecordingUnit

ACC-004では残高記録単位を更新しない。

ただし、更新後の資産口座状態として現在値を返却する。

想定値は、

```text
ACCOUNT
HOLDING
```

とする。

---

### 15.6 isAvailable

ACC-004では利用可能資産区分を更新しない。

レスポンスでは、現在有効な

```text
asset_account_available_settings.is_available
```

を

```text
isAvailable
```

として返却する。

これにより、ACC-003と同じ資産口座詳細表現を使用できる。

---

### 15.7 startYearMonth

ACC-004では利用開始年月を更新しない。

現在保持している

```text
asset_accounts.start_year_month
```

を

```text
YYYY-MM
```

形式で返却する。

---

### 15.8 isEnabled

ACC-004では、利用中の資産口座だけを更新対象とする。

そのため、正常レスポンスでは

```json
{
  "isEnabled": true
}
```

となる。

`deleted_at`はレスポンスへ返却しない。

---

### 15.9 返却しない情報

ACC-004では、以下の内部情報を返却しない。

- `user_id`
- `deleted_at`
- `created_at`
- `updated_at`
- `asset_account_available_settings.id`
- `asset_account_available_settings.asset_account_id`
- `asset_account_available_settings.start_year_month`
- `asset_account_available_settings.end_year_month`
- DB内部の数値コード

また、以下の関連データも返却しない。

- 保有商品一覧
- 月末資産残高
- 商品別月末評価額
- 月末資産状況
- 資産推移

---

## 16. エラーレスポンス

異常時は、API共通方針に従った共通エラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_NAME_ALREADY_EXISTS",
    "message": "同じ名前の資産口座がすでに存在します。"
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

---

### 16.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

この場合、資産口座検索や更新処理へ進まない。

---

### 16.2 INVALID_USER_ID

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

例えば、以下を不正とする。

```text
0
-1
abc
1.5
```

---

### 16.3 USER_NOT_FOUND

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

この場合、資産口座を更新しない。

---

### 16.4 INVALID_ASSET_ACCOUNT_ID

`assetAccountId`の形式が不正な場合は、

```text
INVALID_ASSET_ACCOUNT_ID
```

を返却する。

例えば、

```text
0
-1
abc
1.5
1e3
10abc
```

などを不正とする。

---

### 16.5 VALIDATION_ERROR

リクエストボディの入力値検証に失敗した場合は、

```text
VALIDATION_ERROR
```

を返却する。

主な対象は、以下とする。

- `name`
- `assetType`
- 更新項目未指定
- 更新対象外項目の指定

---

### 16.6 更新項目未指定

以下のように更新可能項目が1件も指定されていない場合は、

```json
{}
```

`VALIDATION_ERROR`として扱う。

概念的なエラー詳細は、

```text
少なくとも1つの更新項目を指定してください。
```

とする。

---

### 16.7 name不正

`name`が指定されている場合に、以下を不正とする。

- `NULL`
- 文字列以外
- 空文字
- 最大文字数超過
- API共通方針上不正となる空白文字列

これらは、

```text
VALIDATION_ERROR
```

として扱う。

---

### 16.8 assetType不正

`assetType`が定義済みの値ではない場合は、

```text
VALIDATION_ERROR
```

とする。

例えば、

```json
{
  "assetType": "CRYPTO"
}
```

は不正とする。

---

### 16.9 更新対象外項目

ACC-004で更新を許可していない項目が送信された場合、未定義項目を拒否するAPI共通方針に従って`VALIDATION_ERROR`とする。

例えば、

```json
{
  "isAvailable": false
}
```

```json
{
  "balanceRecordingUnit": "ACCOUNT"
}
```

```json
{
  "startYearMonth": "2026-09"
}
```

```json
{
  "isEnabled": false
}
```

などを対象とする。

---

### 16.10 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

- 指定された資産口座が存在しない
- 指定された資産口座が他利用者に属する
- 指定された資産口座が論理削除済み

他利用者に属することや論理削除済みであることを個別のエラーコードで公開しない。

---

### 16.11 他利用者の資産口座

例えば、

```text
User A
    assetAccountId = 10

User B
    X-User-Id = 2
```

の状態で、User Bが

```http
PATCH /api/v1/asset-accounts/10
```

を実行した場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

他利用者の資産口座を更新しない。

---

### 16.12 論理削除済み資産口座

対象資産口座が

```text
deleted_at IS NOT NULL
```

の場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

ACC-004から再有効化しない。

---

### 16.13 ASSET_ACCOUNT_NAME_ALREADY_EXISTS

`name`を更新する際、同一利用者内に同名の別資産口座が存在する場合は、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

を返却する。

論理削除済みの別資産口座も重複判定対象とする。

---

### 16.14 更新対象自身の名前

更新対象自身の現在の名前と同じ値を指定した場合は、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

としない。

例えば、

```text
id = 10
name = 証券口座
```

に対して、

```json
{
  "name": "証券口座"
}
```

を送信することは許可する。

---

### 16.15 他利用者の同名資産口座

他利用者にのみ同名資産口座が存在する場合は、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

としない。

名前の一意性は利用者単位で判定する。

---

### 16.16 同時更新による名前競合

事前重複確認を通過した後、別リクエストが同じ名前を先に確定した結果、DBのUNIQUE制約違反が発生する場合がある。

この場合も、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

PostgreSQLの

- SQLSTATE
- 制約名
- SQL
- テーブル名

などをクライアントへ公開しない。

---

### 16.17 利用可能資産設定の不整合

ACC-004の成功レスポンスで現在の`isAvailable`を返却する設計であるため、更新後に現在有効な

```text
asset_account_available_settings
```

を取得する必要がある。

現在有効な設定が0件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

とする。

現在有効な設定が複数件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

とする。

これらは、サーバー側の業務データ不整合として扱う。

---

### 16.18 データ不整合時に補完しない

利用可能資産設定が取得できない場合に、

```text
isAvailable = false
```

へ補完しない。

また、複数件存在する場合に最新1件を任意採用しない。

データ不整合を正常レスポンスで隠蔽しない。

---

### 16.19 更新後レスポンス生成時のエラー

資産口座自体のUPDATEが成功した後に、レスポンス生成に必要な利用可能資産設定取得でデータ不整合が判明する可能性がある。

Phase1ではACC-004を単一レコード更新として明示的なトランザクションを必須としていないため、このケースでは資産口座の更新自体が完了している可能性がある。

この状態を避けたい場合は、Laravel実装方針において、

```text
更新前に
現在有効な利用可能資産設定を検証する
```

構成とする。

概念的には、

```text
資産口座取得
    ↓
現在設定1件確認
    ↓
name重複確認
    ↓
UPDATE
    ↓
成功レスポンス
```

とし、UPDATE後に初めて設定不整合を検出する構成を避ける。

---

### 16.20 INTERNAL_SERVER_ERROR

資産口座更新処理で想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスへ、以下の内部情報を含めない。

- SQL
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 16.21 主なエラー一覧

ACC-004で想定する主なエラーは、以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`未指定 |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`形式不正 |
| `400 Bad Request` | `INVALID_ASSET_ACCOUNT_ID` | `assetAccountId`形式不正 |
| `400 Bad Request` | `VALIDATION_ERROR` | リクエストボディ不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 利用者不存在、または論理削除済み |
| `404 Not Found` | `ASSET_ACCOUNT_NOT_FOUND` | 資産口座不存在、他利用者所属、または論理削除済み |
| `409 Conflict` | `ASSET_ACCOUNT_NAME_ALREADY_EXISTS` | 同一利用者内の資産口座名重複 |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND` | 現在有効な利用可能資産設定が存在しない |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT` | 現在有効な利用可能資産設定が複数存在する |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバー内部エラー |

具体的なHTTPステータスとエラーコードの最終定義は、API共通方針を優先する。

---

### 16.22 エラー時の更新

以下のエラーをUPDATE前に検出した場合は、`asset_accounts`を更新しない。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

特に、利用可能資産設定の不整合確認をUPDATE前に行うことで、

```text
更新だけ成功
+
レスポンス生成失敗
```

という状態を可能な限り避ける。

---

## 17. HTTPステータス

ACC-004では、処理結果に応じて以下のHTTPステータスを返却する。

| HTTPステータス | 用途 |
|---|---|
| `200 OK` | 資産口座更新成功 |
| `400 Bad Request` | 利用者コンテキスト、`assetAccountId`、またはリクエストボディの不正 |
| `404 Not Found` | 利用者または資産口座が存在しない |
| `409 Conflict` | 同一利用者内の資産口座名重複 |
| `500 Internal Server Error` | 利用可能資産設定の不整合、または想定外のサーバー内部エラー |

具体的な独自エラーコードは、「エラーレスポンス」およびAPI共通方針に従う。

---

### 17.1 200 OK

指定された資産口座の更新に成功した場合は、

```http
200 OK
```

を返却する。

概念的には、

```text
更新対象資産口座取得
    ↓
入力値・業務ルール確認
    ↓
必要に応じて資産口座名重複確認
    ↓
asset_accounts更新
    ↓
200 OK
```

とする。

---

### 17.2 400 Bad Request

以下の場合は、

```http
400 Bad Request
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
```

例えば、

- `X-User-Id`未指定
- `X-User-Id`形式不正
- `assetAccountId`形式不正
- 更新項目未指定
- `name`不正
- `assetType`不正
- 更新対象外項目の指定

などを対象とする。

---

### 17.3 404 Not Found

以下の場合は、

```http
404 Not Found
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_NOT_FOUND
ASSET_ACCOUNT_NOT_FOUND
```

`ASSET_ACCOUNT_NOT_FOUND`には、以下を含む。

- 指定された資産口座が存在しない
- 指定された資産口座が他利用者に属する
- 指定された資産口座が論理削除済み

他利用者所属や論理削除済みであることを個別のHTTPステータスで公開しない。

---

### 17.4 409 Conflict

同一利用者内に、更新後の`name`と同じ名前を持つ別の資産口座が存在する場合は、

```http
409 Conflict
```

を返却する。

エラーコードは、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

とする。

アプリケーション側の事前重複確認で検出した場合だけでなく、並行更新によってDBのUNIQUE制約違反が発生した場合も、同じHTTPステータスおよびエラーコードへ変換する。

---

### 17.5 500 Internal Server Error

以下の場合は、

```http
500 Internal Server Error
```

として扱う。

主なエラーコードは、以下とする。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
INTERNAL_SERVER_ERROR
```

利用可能資産設定の不存在または重複は、クライアント入力ではなくサーバー側の業務データ不整合として扱う。

---

### 17.6 ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND

ACC-004の成功レスポンスで現在の`isAvailable`を返却するため、現在有効な利用可能資産設定が1件存在することを前提とする。

対象設定が存在しない場合は、

```http
500 Internal Server Error
```

とし、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

を返却する。

---

### 17.7 ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT

現在年月において有効な利用可能資産設定が複数件存在する場合は、

```http
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

とする。

任意の1件を採用して正常レスポンスを返却しない。

---

### 17.8 INTERNAL_SERVER_ERROR

想定外のサーバー内部エラーが発生した場合は、

```http
500 Internal Server Error
```

を返却する。

エラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

レスポンスへ内部例外やSQLなどを公開しない。

---

## 18. 冪等性

ACC-004は、同じ更新内容を同じ資産口座へ複数回適用した場合、最終的な業務状態が同一となるため、データ状態としては冪等な更新として扱う。

例えば、

```json
{
  "name": "メイン証券口座"
}
```

を複数回送信した場合、最終的な

```text
asset_accounts.name
```

は、

```text
メイン証券口座
```

となる。

---

### 18.1 同一リクエストの再実行

以下のリクエストを同じ資産口座へ複数回送信した場合を考える。

```json
{
  "name": "メイン証券口座",
  "assetType": "SECURITIES"
}
```

1回目の実行後に、同じ内容で再度実行しても、業務上の最終状態は変化しない。

概念的には、

```text
1回目
name = メイン証券口座
asset_type = SECURITIES

2回目
name = メイン証券口座
asset_type = SECURITIES

    ↓

最終状態は同一
```

となる。

---

### 18.2 PATCHと冪等性

HTTPメソッドとして`PATCH`を使用しているが、PATCH自体が必ず非冪等であることを意味しない。

ACC-004では、

```text
指定された属性を
指定された値へ更新する
```

という処理であり、加算や履歴追加のような実行回数依存の処理を行わない。

そのため、同じ入力を繰り返した場合の最終データ状態は同一となる。

---

### 18.3 updated_at

同一内容の再送時に、Laravel/EloquentのDirty判定によって実際にUPDATEが発行されない場合は、

```text
updated_at
```

も変更されない可能性がある。

一方、実装方法によっては同一値更新でも`updated_at`が変化する可能性がある。

ACC-004の冪等性は、主に業務属性である

```text
name
asset_type
```

の最終状態について評価する。

---

### 18.4 同一値更新をエラーにしない

現在値と同じ値を指定した場合も、以下のような専用エラーにはしない。

```text
NO_CHANGES
ALREADY_UPDATED
```

正常な更新リクエストとして扱う。

---

### 18.5 Idempotency-Key

Phase1では、ACC-004専用の

```text
Idempotency-Key
```

は使用しない。

ACC-004は履歴レコードを追加するAPIではなく、指定資産口座の属性を指定値へ更新するAPIであるため、Phase1では追加の冪等性キー管理を必要としない。

---

### 18.6 同時更新と冪等性は区別する

ACC-004が同一入力に対して同じ最終状態へ収束することと、同時更新時の競合を検出できることは別である。

例えば、

```text
Request A
name = A

Request B
name = B
```

が同時実行された場合、Phase1ではLast Write Winsとなる可能性がある。

これは冪等性の問題ではなく、排他制御・競合制御の問題として扱う。

---

## 19. キャッシュ

Phase1では、ACC-004専用のサーバー側アプリケーションキャッシュを使用しない。

ACC-004成功時は、PostgreSQL上の

```text
asset_accounts
```

が最新状態となる。

---

### 19.1 更新API自体をキャッシュしない

ACC-004は更新APIであるため、

```http
PATCH /api/v1/asset-accounts/{assetAccountId}
```

のレスポンスをHTTPキャッシュによって後続リクエストへ再利用することを前提としない。

---

### 19.2 ACC-001への影響

ACC-004で資産口座名または資産種別を変更すると、ACC-001 資産口座一覧取得のレスポンス内容も変化する。

概念的には、

```text
ACC-004
    ↓
name / assetType更新
    ↓
ACC-001の取得結果が変化
```

となる。

フロントエンドでTanStack Queryなどを使用する場合は、ACC-004成功後に資産口座一覧Queryを無効化する。

---

### 19.3 ACC-003への影響

ACC-004成功後は、対象資産口座のACC-003 資産口座詳細取得結果も変化する。

そのため、対象`assetAccountId`の詳細Query Cacheを無効化する。

概念的には、

```text
ACC-004成功
    ↓
ACC-003 detail cache invalidate
    ↓
必要に応じて再取得
```

とする。

---

### 19.4 利用可能資産設定キャッシュ

ACC-004では、

```text
asset_account_available_settings
```

を更新しない。

そのため、ACC-004単独を理由として利用可能資産設定そのもののサーバー側キャッシュを無効化する必要はない。

ただし、ACC-003相当の詳細レスポンスをフロントエンド側でキャッシュしている場合は、`name`や`assetType`が変わるため詳細キャッシュ全体を無効化する。

---

### 19.5 保有商品キャッシュ

ACC-004では、

```text
holding_assets
```

を更新しない。

そのため、保有商品一覧そのもののキャッシュを必ず無効化する必要はない。

ただし、画面側で資産口座名を保有商品表示に合成している場合は、UI全体のキャッシュ設計に応じて関連Queryの再取得を検討する。

---

### 19.6 月末資産キャッシュ

ACC-004では、

```text
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
```

を更新しない。

そのため、月末資産データそのものはACC-004によって変更されない。

ただし、画面表示上資産口座名や資産種別を月末資産データと組み合わせて表示している場合は、表示用Queryの構成に応じてキャッシュ無効化を検討する。

---

### 19.7 Query Cacheの基本方針

React側では、ACC-004成功後に最低限、

```text
資産口座一覧
+
対象資産口座詳細
```

のQuery Cacheを無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey: assetAccountKeys.all,
});

await queryClient.invalidateQueries({
  queryKey: assetAccountKeys.detail(
    assetAccountId,
  ),
});
```

正式なQuery Keyは、React共通設計に従う。

---

### 19.8 Optimistic Update

Phase1では、ACC-004に対するOptimistic Updateは必須としない。

ACC-004では、

- 利用者境界確認
- 資産口座存在確認
- 資産口座名重複確認
- DBのUNIQUE制約確認

など、サーバー側でしか最終判断できない処理が存在する。

そのため、

```text
API成功
    ↓
Query invalidate
    ↓
再取得
```

という単純な方式を基本とする。

---

### 19.9 サーバー状態を正とする

ACC-004成功後の最終状態は、PostgreSQLへ保存されたデータを正とする。

React側で送信したRequestだけを基準に永続的な表示状態を確定しない。

必要に応じてACC-001またはACC-003を再取得して最新状態へ同期する。

---

### 19.10 HTTPキャッシュ

ACC-004自体はHTTPキャッシュ対象としない。

また、ACC-004成功後にACC-003などのGETレスポンスが古い状態を返し続けないよう、将来的にHTTPキャッシュを導入する場合は、関連リソースの失効方針を合わせて設計する。

Phase1では、ACC系API独自の

```text
ETag
Last-Modified
If-Match
If-None-Match
```

などは導入しない。

---

### 19.11 利用者境界とキャッシュ

フロントエンドで資産口座一覧・詳細をキャッシュする場合は、利用者切替時に前利用者のデータを誤表示しないようにする。

概念的には、

```text
userId
+
assetAccountId
```

または利用者切替時のQuery無効化によって、キャッシュ上でも利用者境界を維持する。

---

### 19.12 ACC-005・ACC-006との関係

ACC-004成功後は、主に

```text
name
assetType
```

が変化する。

一方、

```text
isEnabled
    → ACC-005

isAvailable
    → ACC-006
```

によって変更される。

そのため、各Mutation成功後に対応する資産口座一覧・詳細Queryを無効化する方針を共通化するとよい。

---

## 20. 関連テーブル

ACC-004では、指定された資産口座の通常属性を更新するため、以下のテーブルを使用する。

| テーブル | 用途 | 更新 |
|---|---|:---:|
| `users` | 操作対象利用者の確認 | × |
| `asset_accounts` | 更新対象資産口座の取得、利用者境界確認、通常属性更新、同名重複確認 | ○ |
| `asset_account_available_settings` | 現在の利用可能資産区分の取得 | × |

ACC-004で直接更新する業務テーブルは、

```text
asset_accounts
```

のみとする。

---

### 20.1 users

`X-User-Id`で指定された操作対象利用者の存在確認に使用する。

概念的な条件は、以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者コンテキストの確認は、API共通Middlewareで行う。

ACC-004では、`users`を更新しない。

---

### 20.2 asset_accounts

更新対象となる資産口座の取得、利用者境界確認、資産口座名重複確認、通常属性更新に使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | `assetAccountId`との照合 |
| `user_id` | 操作対象利用者との利用者境界確認 |
| `name` | 資産口座名、同名重複確認、更新 |
| `asset_type` | 資産種別、更新 |
| `balance_recording_unit` | 現在値の返却 |
| `start_year_month` | 現在値の返却 |
| `updated_at` | 更新日時 |
| `deleted_at` | 利用状態の判定 |

---

### 20.3 更新対象取得

更新対象資産口座は、概念的に以下の条件で取得する。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

他利用者に属する資産口座、論理削除済み資産口座は更新対象として取得しない。

---

### 20.4 資産口座名の重複確認

`name`が指定された場合は、同一利用者内に同名の別資産口座が存在しないことを確認する。

概念的な条件は、以下とする。

```text
asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.name
    = request.name

AND

asset_accounts.id
    != assetAccountId
```

論理削除済み資産口座も重複判定対象に含める。

Laravel SoftDeletesを使用する場合は、重複確認時に

```php
withTrashed()
```

を使用する。

---

### 20.5 asset_accountsの一意性

同一利用者内の資産口座名については、データベース側でも

```text
user_id
+
name
```

の組み合わせに一意性を保証する。

これにより、並行更新時にアプリケーション側の事前重複確認を複数リクエストが通過しても、同名資産口座が複数存在する状態を防止する。

---

### 20.6 name

リクエストに`name`が指定された場合のみ、

```text
asset_accounts.name
```

を更新する。

未指定の場合は、現在値を保持する。

現在値と同一の名前が指定された場合は、更新対象自身を重複判定から除外する。

---

### 20.7 asset_type

リクエストに`assetType`が指定された場合のみ、

```text
asset_accounts.asset_type
```

を更新する。

APIで受け取った

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

を、DB内部の保存形式へ変換して保存する。

---

### 20.8 balance_recording_unit

```text
asset_accounts.balance_recording_unit
```

は、ACC-004では更新しない。

成功レスポンス生成のために現在値を使用するのみとする。

---

### 20.9 start_year_month

```text
asset_accounts.start_year_month
```

も、ACC-004では更新しない。

成功レスポンスでは現在値を

```text
startYearMonth
```

として返却する。

---

### 20.10 deleted_at

```text
asset_accounts.deleted_at
```

は、更新対象の有効状態確認に使用する。

ACC-004では変更しない。

正常レスポンスでは対象資産口座が利用中であるため、

```text
isEnabled = true
```

として扱う。

---

### 20.11 updated_at

実際に資産口座の属性が更新された場合は、Eloquentの通常動作に従って

```text
updated_at
```

が更新される。

クライアントから`updatedAt`を指定させない。

---

### 20.12 asset_account_available_settings

ACC-004の成功レスポンスで現在の

```text
isAvailable
```

を返却するために参照する。

ACC-004では、このテーブルを更新しない。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `asset_account_id` | 対象資産口座との関連 |
| `start_year_month` | 設定の適用開始年月 |
| `end_year_month` | 設定の適用終了年月 |
| `is_available` | 現在の利用可能資産区分 |

---

### 20.13 現在有効な利用可能資産設定

現在年月を基準として、概念的に以下の条件を満たす利用可能資産設定を取得する。

```text
asset_account_available_settings.asset_account_id
    = assetAccountId

AND

asset_account_available_settings.start_year_month
    <= 現在年月

AND

(
    asset_account_available_settings.end_year_month
        IS NULL

    OR

    asset_account_available_settings.end_year_month
        >= 現在年月
)
```

正常状態では、該当する設定が1件だけ存在することを前提とする。

---

### 20.14 利用可能資産設定を更新しない

ACC-004では、

```text
asset_account_available_settings
```

への

```text
INSERT
UPDATE
DELETE
```

を行わない。

利用可能資産区分の変更は、ACC-006の責務とする。

---

### 20.15 更新しない関連テーブル

ACC-004では、以下のテーブルを更新しない。

- `users`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

資産口座の通常属性変更だけに責務を限定する。

---

## 21. 関連する機能要件

ACC-004は、資産口座の通常属性更新に関する機能要件と対応する。

主な関連要件は、以下とする。

- 資産口座管理
  - 登録済み資産口座を編集できる
  - 資産口座名を変更できる
  - 資産種別を変更できる
- 資産口座名
  - 同一利用者内では重複させない
  - 更新対象自身の現在名は重複としない
  - 異なる利用者間では同名を許可する
  - 論理削除済み資産口座も重複判定対象とする
- 残高記録単位
  - Phase1では登録後に変更しない
- 利用開始年月
  - Phase1では登録後に変更しない
- 利用可能資産区分
  - ACC-004では変更しない
  - 利用可能資産区分の変更はACC-006で扱う
- 利用状態
  - ACC-004では変更しない
  - 資産口座無効化はACC-005で扱う
- 利用者境界
  - 操作対象利用者に帰属する資産口座だけを更新できる
  - 他利用者の資産口座は対象不存在として扱う

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

## 22. インデックス

ACC-004では、主に以下の条件を使用する。

```text
asset_accounts.id
```

```text
asset_accounts.user_id
```

```text
asset_accounts.name
```

また、成功レスポンス生成時には

```text
asset_account_available_settings.asset_account_id
+
start_year_month
+
end_year_month
```

を使用する。

---

### 22.1 asset_accounts.id

`asset_accounts.id`は主キーであるため、主キーインデックスを使用する。

ACC-004専用として追加インデックスを作成しない。

---

### 22.2 user_id + name

同一利用者内の資産口座名一意性を保証するため、

```text
user_id
+
name
```

のUNIQUE制約を使用する。

この制約は、ACC-002の登録時とACC-004の更新時で共通して利用する。

---

### 22.3 論理削除済み資産口座

論理削除済み資産口座も名前一意性の対象とするため、Phase1では

```text
WHERE deleted_at IS NULL
```

を条件とする部分UNIQUEインデックスを使用しない。

---

### 22.4 asset_account_available_settings

現在有効な設定取得では、

```text
asset_account_id
```

を必須条件として使用する。

必要に応じて、

```text
asset_account_id
+
start_year_month
```

などの複合インデックスを検討する。

ACC-003やACC-006など他APIでも利用する検索条件を踏まえて設計する。

---

### 22.5 不要なインデックスを追加しない

ACC-004だけを理由として、主キーやUNIQUE制約と役割が重複するインデックスを追加しない。

実際のデータ件数や実行計画を確認したうえで追加を判断する。

---

## 23. 性能

ACC-004は、単一の資産口座を更新するAPIであるため、処理負荷は小さい。

概念的には、

```text
資産口座1件取得
    ↓
必要に応じて名前重複確認
    ↓
現在の利用可能資産設定確認
    ↓
資産口座1件更新
```

で完結する。

---

### 23.1 一覧取得を行わない

以下のように、利用者の資産口座一覧を全件取得してから対象IDを探す実装は行わない。

```text
資産口座一覧取得
    ↓
PHP側でassetAccountId検索
```

対象IDと利用者IDを条件として直接1件取得する。

---

### 23.2 name未指定時は重複確認しない

`name`がリクエストに含まれない場合は、名前重複確認Queryを実行しない。

例えば、

```json
{
  "assetType": "BANK"
}
```

の場合は、資産口座名の重複検索を省略する。

---

### 23.3 利用可能資産設定を全履歴取得しない

成功レスポンスに必要なのは、現在有効な

```text
isAvailable
```

だけである。

そのため、利用可能資産設定の全履歴を取得してPHP側で判定しない。

DB側で現在年月条件を指定して取得する。

---

### 23.4 取得件数

通常は、

```text
asset_accounts
    1件

asset_account_available_settings
    1件
```

だけを扱う。

大量データをアプリケーションメモリへ読み込まない。

---

### 23.5 保有商品を取得しない

対象資産口座の`balance_recording_unit`が

```text
HOLDING
```

であっても、

```text
holding_assets
```

をEager Loadしない。

ACC-004では保有商品を更新・返却しないためである。

---

### 23.6 月末資産データを取得しない

以下のデータをACC-004の更新処理のために取得しない。

- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`

通常属性更新に不要な関連データを読み込まない。

---

### 23.7 N+1問題

ACC-004は、単一資産口座を対象とするため、通常の意味でのN+1問題は発生しない。

不要な関連リレーションを読み込まない構成とする。

---

## 24. セキュリティ

ACC-004では、利用者境界と更新可能項目の限定を重要なセキュリティ要件とする。

---

### 24.1 assetAccountIdだけで取得しない

以下のような更新対象取得は避ける。

```php
AssetAccount::find(
    $assetAccountId,
);
```

必ず、

```text
assetAccountId
+
userId
+
deleted_at IS NULL
```

を条件とする。

---

### 24.2 他利用者の資産口座を更新しない

他利用者に属する`assetAccountId`が指定された場合でも、その資産口座を取得・更新しない。

レスポンスは、

```text
ASSET_ACCOUNT_NOT_FOUND
```

とし、他利用者のリソース存在有無を外部へ公開しない。

---

### 24.3 論理削除済み資産口座を更新しない

論理削除済み資産口座を通常更新対象に含めない。

ACC-004から`deleted_at`を`NULL`へ戻して再有効化することもできない。

---

### 24.4 Mass Assignmentを避ける

以下のようにRequest全体をそのままModelへ渡さない。

```php
$assetAccount->update(
    $request->all(),
);
```

更新対象は、

```text
name
asset_type
```

だけに限定する。

---

### 24.5 更新対象外項目を変更させない

以下をクライアント指定によって更新できないようにする。

- `user_id`
- `balance_recording_unit`
- `start_year_month`
- `deleted_at`
- `created_at`
- `updated_at`

また、

```text
isAvailable
```

から利用可能資産設定を変更させない。

---

### 24.6 user_idを変更しない

クライアントが

```json
{
  "userId": "999"
}
```

などを送信しても、資産口座の所有利用者を変更できない。

所有者変更はACC-004の責務に含めない。

---

### 24.7 DB内部コードを受け付けない

資産種別をDB内部の数値コードで指定させない。

APIで定義した

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

だけを受け付ける。

---

### 24.8 SQLインジェクション対策

`assetAccountId`、利用者ID、資産口座名などを検索条件へ使用する場合は、EloquentまたはQuery Builderのバインド機構を使用する。

SQL文字列へ入力値を直接連結しない。

---

### 24.9 内部情報を公開しない

エラー時に、以下の情報をレスポンスへ含めない。

- SQL
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- Laravel内部例外
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

---

## 25. ログ・監視

ACC-004では、API共通ログ方針に従って更新処理の結果を記録する。

---

### 25.1 ログコンテキスト

必要に応じて、以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assetAccountId
httpStatus
errorCode
```

`apiId`は、

```text
ACC-004
```

とする。

---

### 25.2 正常時

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = ACC-004
assetAccountId
httpStatus = 200
```

更新後の資産口座名や資産種別を通常ログへ不要に記録しない。

---

### 25.3 異常時

異常時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId
assetAccountId
errorCode
httpStatus
```

---

### 25.4 資産口座名競合

以下のエラーが発生した場合は、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

必要に応じて競合発生をログへ記録する。

ただし、PostgreSQLの制約名やSQLをレスポンスへ公開しない。

---

### 25.5 利用可能資産設定の不整合

以下の場合は、データ不整合として調査可能なログを記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text
assetAccountId
currentYearMonth
matchedSettingCount
```

を内部ログへ記録してよい。

---

### 25.6 requestId

クライアントへ返却する`requestId`とサーバーログを関連付けられるようにする。

障害調査時に、

```text
requestId
    ↓
対象ログ
```

を追跡できる状態とする。

---

## 26. テスト観点

ACC-004では、通常属性更新だけでなく、

- PATCHによる部分更新
- 利用者境界
- 論理削除
- 資産口座名重複
- 更新対象外項目
- 同時更新
- 関連データ非更新
- レスポンス契約

を重点的に確認する。

---

### 26.1 nameのみ更新

以下のリクエストを送信する。

```json
{
  "name": "メイン証券口座"
}
```

以下を確認する。

- `200 OK`となること
- `asset_accounts.name`が更新されること
- `asset_accounts.asset_type`が変更されないこと
- `balance_recording_unit`が変更されないこと
- `start_year_month`が変更されないこと
- `deleted_at`が変更されないこと

---

### 26.2 assetTypeのみ更新

以下のリクエストを送信する。

```json
{
  "assetType": "SECURITIES"
}
```

以下を確認する。

- `200 OK`となること
- `asset_accounts.asset_type`が更新されること
- `asset_accounts.name`が変更されないこと
- その他の更新対象外項目が変更されないこと

---

### 26.3 nameとassetTypeを同時更新

以下を指定する。

```json
{
  "name": "メイン証券口座",
  "assetType": "SECURITIES"
}
```

両方が正しく更新されることを確認する。

---

### 26.4 空リクエスト

以下を送信する。

```json
{}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

`asset_accounts`が更新されないことも確認する。

---

### 26.5 name未指定

`assetType`だけを指定し、`name`未指定がエラーにならないことを確認する。

PATCHとして、未指定項目が必須扱いされないこと。

---

### 26.6 assetType未指定

`name`だけを指定し、`assetType`未指定がエラーにならないことを確認する。

---

### 26.7 name = null

以下を送信する。

```json
{
  "name": null
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.8 name = 空文字

以下を送信する。

```json
{
  "name": ""
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.9 name最大文字数

テーブル定義で定めた最大文字数ちょうどの値を更新できることを確認する。

最大文字数を超えた場合は、

```text
VALIDATION_ERROR
```

となること。

---

### 26.10 assetType正常値

以下をそれぞれ確認する。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

API値が正しいDB保存形式へ変換されること。

---

### 26.11 assetType不正

例えば、

```json
{
  "assetType": "CRYPTO"
}
```

を送信する。

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.12 assetType = null

以下を送信する。

```json
{
  "assetType": null
}
```

バリデーションエラーとなり、業務データが更新されないこと。

---

### 26.13 自分自身と同じname

現在、

```text
id = 10
name = 証券口座
```

の資産口座に対して、

```json
{
  "name": "証券口座"
}
```

を送信する。

重複エラーにならず、正常に処理できることを確認する。

---

### 26.14 同一利用者の別資産口座とのname重複

以下の状態を用意する。

```text
User A
├─ id = 10 / 証券口座
└─ id = 20 / 銀行口座
```

`id = 20`を

```json
{
  "name": "証券口座"
}
```

へ変更する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となること。

---

### 26.15 論理削除済み資産口座とのname重複

同一利用者に、

```text
name = 証券口座
deleted_at IS NOT NULL
```

の別資産口座を用意する。

利用中資産口座を同じ名前へ変更する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となること。

---

### 26.16 他利用者の同名資産口座

以下の状態を用意する。

```text
User A
    証券口座

User B
    銀行口座
```

User Bの`銀行口座`を

```text
証券口座
```

へ変更する。

正常に更新できることを確認する。

---

### 26.17 X-User-Id未指定

`X-User-Id`を指定しない。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

資産口座が更新されないこと。

---

### 26.18 X-User-Id形式不正

例えば、

```http
X-User-Id: abc
```

を指定する。

期待結果：

```text
400 Bad Request
INVALID_USER_ID
```

となること。

---

### 26.19 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 26.20 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 26.21 assetAccountId形式不正

以下の値を確認する。

```text
0
-1
abc
1.5
1e3
10abc
```

期待結果：

```text
400 Bad Request
INVALID_ASSET_ACCOUNT_ID
```

となること。

---

### 26.22 資産口座不存在

存在しない`assetAccountId`を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 26.23 他利用者の資産口座

以下の状態を用意する。

```text
User A
    Asset Account A

User B
    Asset Account B
```

User AとしてAsset Account Bを更新する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの資産口座が変更されないことも確認する。

---

### 26.24 論理削除済み資産口座

対象資産口座を

```text
deleted_at IS NOT NULL
```

としてACC-004を実行する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 26.25 balanceRecordingUnitを送信

以下を送信する。

```json
{
  "balanceRecordingUnit": "ACCOUNT"
}
```

更新対象外項目を拒否する共通方針の場合は、

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

`balance_recording_unit`が変更されないことを確認する。

---

### 26.26 startYearMonthを送信

以下を送信する。

```json
{
  "startYearMonth": "2026-09"
}
```

ACC-004では変更できないことを確認する。

`asset_accounts.start_year_month`が変更されないこと。

---

### 26.27 isAvailableを送信

以下を送信する。

```json
{
  "isAvailable": false
}
```

ACC-004では利用可能資産設定を変更できないことを確認する。

```text
asset_account_available_settings
```

に変更がないこと。

---

### 26.28 isEnabledを送信

以下を送信する。

```json
{
  "isEnabled": false
}
```

ACC-004から無効化できないことを確認する。

```text
deleted_at
```

が変更されないこと。

---

### 26.29 userIdを送信

以下を送信する。

```json
{
  "name": "証券口座",
  "userId": "999"
}
```

未定義項目を拒否する共通方針の場合は、バリデーションエラーとなること。

少なくとも、

```text
asset_accounts.user_id
```

が変更されないことを確認する。

---

### 26.30 isAvailable = true

現在有効な利用可能資産設定が

```text
is_available = true
```

の場合、成功レスポンスで

```json
{
  "isAvailable": true
}
```

となることを確認する。

---

### 26.31 isAvailable = false

現在有効な設定が

```text
is_available = false
```

の場合、

```json
{
  "isAvailable": false
}
```

となることを確認する。

---

### 26.32 現在利用可能資産設定が0件

現在有効な利用可能資産設定を0件とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

となること。

資産口座更新前に設定不整合を検出する実装の場合は、`asset_accounts`が変更されないことも確認する。

---

### 26.33 現在利用可能資産設定が複数件

現在有効な設定を2件以上用意する。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

となること。

任意の1件を成功レスポンスへ使用しないこと。

---

### 26.34 同一値更新

現在値と同じ

```text
name
assetType
```

を指定する。

業務エラーにならないことを確認する。

EloquentのDirty判定によってUPDATEが発行されない場合も正常とする。

---

### 26.35 updated_at

実際に値が変更された場合は、

```text
updated_at
```

が更新されることを確認する。

同一値更新時の`updated_at`の扱いは、実装方針に従う。

---

### 26.36 同時名前更新

同一利用者に属する2つの資産口座を、並行して同じ名前へ変更する。

以下を確認する。

- 同名資産口座が2件存在する状態にならないこと
- DBのUNIQUE制約が機能すること
- 一方が成功すること
- 競合側が`409 Conflict`相当となること
- `ASSET_ACCOUNT_NAME_ALREADY_EXISTS`へ変換されること

---

### 26.37 同一資産口座への同時更新

同じ資産口座に対して異なる値を並行更新する。

Phase1では楽観ロックを行わないため、Last Write Winsとなり得ることを確認する。

想定外の500エラーやデッドロックを通常発生させないことも確認する。

---

### 26.38 関連データ非更新

ACC-004実行前後で、以下が変更されないことを確認する。

- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

---

### 26.39 正常レスポンス契約

正常時に、概念的に以下の形式となることを確認する。

```json
{
  "data": {
    "id": "10",
    "name": "メイン証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

以下を確認する。

- HTTPステータスが`200 OK`
- `id`がstring
- `name`がstring
- `assetType`がAPI用文字列
- `balanceRecordingUnit`がAPI用文字列
- `isAvailable`がboolean
- `startYearMonth`が`YYYY-MM`
- `isEnabled = true`

---

### 26.40 返却しない情報

正常レスポンスへ、以下が含まれないことを確認する。

- `user_id`
- `userId`
- `deleted_at`
- `deletedAt`
- `created_at`
- `createdAt`
- `updated_at`
- `updatedAt`
- `asset_account_available_settings.id`
- `assetAccountAvailableSettingId`
- `asset_account_id`
- `endYearMonth`
- DB内部の数値コード
- 保有商品一覧
- 月末資産データ

---

### 26.41 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
INTERNAL_SERVER_ERROR
```

以下も確認する。

- `error.code`が設定されること
- `error.message`が設定されること
- 必要に応じて`error.details`が設定されること
- `requestId`が設定されること
- SQLが含まれないこと
- PostgreSQL内部エラーが含まれないこと
- 制約名が含まれないこと
- スタックトレースが含まれないこと
- サーバーファイルパスが含まれないこと

---

### 26.42 副作用範囲

正常終了時に変更される業務データが、

```text
対象asset_accounts 1件
```

だけであることを確認する。

更新対象カラムも、リクエストで指定された

```text
name
asset_type
```

に限定されることを確認する。

---

### 26.43 INTERNAL_SERVER_ERROR

更新処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

- 内部情報がレスポンスへ公開されないこと
- サーバーログに調査情報が記録されること
- レスポンスの`requestId`からログを追跡できること

---

## 27. Laravel実装方針

ACC-004では、Action、FormRequest、UseCase、Query、Repository、DTO、API Resource、Responderを分離して実装する。

概念的な構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
FormRequest
    ↓
Action
    ↓
UseCase
    ├─ AssetAccountQuery
    ├─ AssetAccountAvailableSettingQuery
    └─ AssetAccountRepository
    ↓
Update Result DTO
    ↓
API Resource
    ↓
Responder
```

ACC-004では、`asset_accounts`の通常属性だけを更新する。

更新対象は、

```text
name
asset_type
```

に限定する。

一方、

```text
balance_recording_unit
start_year_month
deleted_at
```

および

```text
asset_account_available_settings
```

は更新しない。

---

### 27.1 Route

ACC-004は、以下のルートとして定義する。

概念例：

```php
Route::patch(
    '/api/v1/asset-accounts/{assetAccountId}',
    UpdateAssetAccountAction::class,
);
```

ACC-003とは同じパスを使用するが、HTTPメソッドによって責務を分離する。

```text
GET
/api/v1/asset-accounts/{assetAccountId}
    → ACC-003 資産口座詳細取得

PATCH
/api/v1/asset-accounts/{assetAccountId}
    → ACC-004 資産口座更新
```

---

### 27.2 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、以下とする。

```text
X-User-Id取得
    ↓
必須チェック
    ↓
形式チェック
    ↓
users存在確認
    ↓
UserContext設定
    ↓
FormRequest
    ↓
Action
```

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 27.3 FormRequest

リクエストボディの入力値検証には、ACC-004専用のFormRequestを使用する。

概念例：

```php
final class UpdateAssetAccountRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => [
                'sometimes',
                'required',
                'string',
                'max:100',
            ],

            'assetType' => [
                'sometimes',
                'required',
                'string',
                Rule::enum(
                    AssetType::class,
                ),
            ],
        ];
    }
}
```

ただし、`name`と`assetType`の両方が未指定の場合はバリデーションエラーとする。

---

### 27.4 空リクエストの検証

以下のような空リクエストは許可しない。

```json
{}
```

FormRequestまたは専用Validatorで、

```text
name未指定
AND
assetType未指定
```

の場合は、

```text
VALIDATION_ERROR
```

とする。

概念例：

```php
public function after(): array
{
    return [
        function (
            Validator $validator,
        ): void {
            if (
                ! $this->has('name')
                && ! $this->has('assetType')
            ) {
                $validator->errors()->add(
                    'request',
                    '少なくとも1つの更新項目を指定してください。',
                );
            }
        },
    ];
}
```

---

### 27.5 FormRequestで行うこと

FormRequestでは、以下の入力形式を検証する。

```text
name
assetType
```

主に以下を確認する。

- 項目指定有無
- 文字列型
- NULL禁止
- 空文字禁止
- 最大文字数
- Enum値
- 少なくとも1項目指定

HTTP入力として不正な値をUseCaseへ渡さない。

---

### 27.6 FormRequestで行わないこと

FormRequestでは、以下の業務ルールを扱わない。

- 資産口座存在確認
- 利用者境界確認
- 論理削除判定
- 資産口座名重複確認
- 現在利用可能資産設定確認
- DB更新
- 同時更新競合判定

これらは、UseCase、Query、Repositoryで扱う。

---

### 27.7 更新対象外項目

ACC-004では、以下を更新対象として定義しない。

```text
userId
balanceRecordingUnit
startYearMonth
isAvailable
isEnabled
deletedAt
```

API共通方針として未定義項目を拒否する場合は、これらが送信された時点で`VALIDATION_ERROR`とする。

Request全体をそのままModelへ渡す実装は行わない。

---

### 27.8 assetAccountIdの形式検証

`assetAccountId`は、ルート制約またはAPI共通のパスパラメータ検証方式で検証する。

概念例：

```php
Route::patch(
    '/api/v1/asset-accounts/{assetAccountId}',
    UpdateAssetAccountAction::class,
)
    ->where(
        'assetAccountId',
        '[1-9][0-9]*',
    );
```

正の整数形式のみを許可する。

形式不正の場合は、

```text
INVALID_ASSET_ACCOUNT_ID
```

へ変換する。

---

### 27.9 Action

Actionは、検証済みリクエスト、パスパラメータ、利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php
final class UpdateAssetAccountAction
{
    public function __invoke(
        string $assetAccountId,
        UpdateAssetAccountRequest $request,
        UpdateAssetAccountUseCase $useCase,
        UpdateAssetAccountResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $input =
            new UpdateAssetAccountInput(
                userId:
                    $userContext->userId,

                assetAccountId:
                    (int) $assetAccountId,

                name:
                    $request->has('name')
                        ? $request->string(
                            'name',
                        )->toString()
                        : null,

                assetType:
                    $request->has('assetType')
                        ? AssetType::from(
                            $request->string(
                                'assetType',
                            )->toString(),
                        )
                        : null,
            );

        $result =
            $useCase->execute(
                $input,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

---

### 27.10 Input DTO

PATCHでは、「未指定」と「指定済み」を区別する必要がある。

そのため、Input DTOでは更新可能項目をnullableとして表現してよい。

概念例：

```php
final readonly class UpdateAssetAccountInput
{
    public function __construct(
        public int $userId,
        public int $assetAccountId,
        public ?string $name,
        public ?AssetType $assetType,
    ) {
    }
}
```

ただし、ACC-004では`NULL`値そのものを更新値として許可しないため、

```text
null
    = Requestで未指定
```

という意味に限定する。

---

### 27.11 未指定とNULL入力を混同しない

例えば、

```json
{
  "name": null
}
```

はバリデーションエラーとする。

一方、

```json
{
  "assetType": "SECURITIES"
}
```

の場合の

```text
name = null
```

は、Input DTO上で「name未指定」を意味する。

Actionでは`has()`や`array_key_exists()`相当の判定を使用し、未指定とNULL入力を区別する。

---

### 27.12 Actionで行わないこと

Actionでは、以下を行わない。

- 資産口座検索
- 利用者境界判定
- 同名重複確認
- 現在年月取得
- 利用可能資産設定検索
- UPDATE
- UNIQUE制約例外処理
- レスポンス配列生成

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 27.13 UseCase

ACC-004の業務処理全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `assetAccountId`を受け取る
3. 更新対象資産口座を取得する
4. 対象不存在の場合は業務例外を送出する
5. 現在年月を取得する
6. 現在有効な利用可能資産設定が1件であることを確認する
7. `name`指定時のみ同名重複を確認する
8. 指定された項目のみ更新する
9. 更新結果DTOを生成する
10. DTOを返却する

概念的には、以下とする。

```text
UpdateAssetAccountInput
    ↓
AssetAccountQuery
    ↓
更新対象取得
    ↓
CurrentYearMonth取得
    ↓
AssetAccountAvailableSettingQuery
    ↓
現在設定1件確認
    ↓
name指定あり？
    ├─ Yes
    │    ↓
    │ AssetAccountQuery
    │ 同名重複確認
    └─ No
    ↓
AssetAccountRepository
    ↓
部分更新
    ↓
Update Result DTO
```

---

### 27.14 AssetAccountQuery

更新対象資産口座の取得は、Queryへ委譲する。

概念例：

```php
$assetAccount =
    $this->assetAccountQuery
        ->findActiveByIdAndUser(
            assetAccountId:
                $input->assetAccountId,

            userId:
                $input->userId,
        );
```

概念的な条件は、以下とする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 27.15 利用者境界をQueryへ含める

以下のように、IDだけで取得してから所有利用者を判定する方式は基本としない。

```php
$assetAccount =
    AssetAccount::find(
        $assetAccountId,
    );
```

対象取得時点で、

```text
assetAccountId
+
userId
+
有効状態
```

を条件へ含める。

---

### 27.16 SoftDeletes

`AssetAccount` Modelでは、Laravelの

```php
use SoftDeletes;
```

を使用する。

ACC-004の更新対象取得では、

```php
withTrashed()
```

を使用しない。

論理削除済み資産口座は更新対象外とする。

---

### 27.17 ASSET_ACCOUNT_NOT_FOUND

対象資産口座を取得できない場合は、業務例外を送出する。

概念例：

```php
if ($assetAccount === null) {
    throw new
        AssetAccountNotFoundException();
}
```

以下を同じ例外へ集約する。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

最終的に、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

へ変換する。

---

### 27.18 現在利用可能資産設定をUPDATE前に確認する

ACC-004の成功レスポンスでは、現在の

```text
isAvailable
```

を返却する。

そのため、資産口座UPDATE後に初めて利用可能資産設定不整合を検出する構成は避ける。

概念的には、

```text
更新対象取得
    ↓
現在設定確認
    ↓
設定1件
    ↓
更新
```

とする。

これにより、

```text
asset_accounts更新成功
    ↓
現在設定取得失敗
    ↓
500レスポンス
```

という分かりにくい状態を避ける。

---

### 27.19 CurrentYearMonth

現在利用可能資産設定判定には、サーバー側の現在年月を使用する。

概念例：

```php
$currentYearMonth =
    now()->format(
        'Y-m',
    );
```

ACC-003と同様に、必要に応じてClockやDateProviderへ時刻依存を分離してよい。

---

### 27.20 AssetAccountAvailableSettingQuery

現在有効な利用可能資産設定の取得は、専用Queryへ委譲する。

概念例：

```php
$settings =
    $this->availableSettingQuery
        ->findCurrentByAssetAccount(
            assetAccountId:
                $assetAccount->id,

            yearMonth:
                $currentYearMonth,
        );
```

概念的な条件は、以下とする。

```text
asset_account_id
    = assetAccountId

AND

start_year_month
    <= currentYearMonth

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= currentYearMonth
)
```

---

### 27.21 利用可能資産設定件数

現在有効な設定は、1件であることを必須とする。

概念例：

```php
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingNotFoundException();
}

if ($settings->count() > 1) {
    throw new
        AssetAccountAvailableSettingConflictException();
}
```

0件の場合に`false`へ補完せず、複数件の場合に任意の1件を採用しない。

---

### 27.22 利用可能資産設定は更新しない

ACC-004では、

```text
asset_account_available_settings
```

に対する

```text
INSERT
UPDATE
DELETE
```

を行わない。

`isAvailable`変更はACC-006へ委譲する。

---

### 27.23 name指定時のみ重複確認する

`name`がRequestで指定された場合のみ、同名重複確認を行う。

概念例：

```php
if ($input->name !== null) {
    $exists =
        $this->assetAccountQuery
            ->existsDuplicateNameIncludingDeleted(
                userId:
                    $input->userId,

                name:
                    $input->name,

                exceptAssetAccountId:
                    $input->assetAccountId,
            );

    if ($exists) {
        throw new
            AssetAccountNameAlreadyExistsException();
    }
}
```

`assetType`だけの更新では、不要な名前重複Queryを実行しない。

---

### 27.24 論理削除済みも重複確認対象とする

重複確認では、SoftDeletesの通常スコープを外し、論理削除済み資産口座も検索対象に含める。

概念例：

```php
return AssetAccount::query()
    ->withTrashed()
    ->where(
        'user_id',
        $userId,
    )
    ->where(
        'name',
        $name,
    )
    ->whereKeyNot(
        $exceptAssetAccountId,
    )
    ->exists();
```

実際のメソッドは、Laravelバージョンに合わせて適切な記述とする。

---

### 27.25 更新対象自身を除外する

名前重複確認では、更新対象自身を除外する。

概念的には、

```text
id != assetAccountId
```

を条件へ含める。

これにより、現在の名前と同じ名前を再指定しても重複エラーとならない。

---

### 27.26 AssetAccountRepository

資産口座更新は、Repositoryへ委譲する。

概念的なインターフェースは、以下とする。

```php
interface AssetAccountRepository
{
    public function update(
        AssetAccount $assetAccount,
        ?string $name,
        ?AssetType $assetType,
    ): AssetAccount;
}
```

ただし、引数が増える場合は更新専用DTOを使用してよい。

---

### 27.27 Repositoryで部分更新する

Repositoryでは、指定された項目だけを更新する。

概念例：

```php
public function update(
    AssetAccount $assetAccount,
    ?string $name,
    ?AssetType $assetType,
): AssetAccount {
    if ($name !== null) {
        $assetAccount->name =
            $name;
    }

    if ($assetType !== null) {
        $assetAccount->asset_type =
            $assetType;
    }

    $assetAccount->save();

    return $assetAccount;
}
```

未指定項目をNULLやデフォルト値へ置き換えない。

---

### 27.28 Mass Assignmentを使用しない

以下のような処理は行わない。

```php
$assetAccount->update(
    $request->all(),
);
```

更新対象を

```text
name
asset_type
```

へ限定するため、明示的に値を設定する。

---

### 27.29 更新対象外カラムを変更しない

Repositoryでは、以下を変更しない。

```text
user_id
balance_recording_unit
start_year_month
deleted_at
```

また、別Repositoryを呼び出して

```text
asset_account_available_settings
```

を更新しない。

---

### 27.30 Enum Cast

`asset_type`は、PHP EnumまたはEloquent Castとして扱う。

概念例：

```php
protected function casts(): array
{
    return [
        'asset_type'
            => AssetType::class,

        'balance_recording_unit'
            => BalanceRecordingUnit::class,
    ];
}
```

UseCaseやRepositoryへDB内部の数値コードを直接持ち込まない。

---

### 27.31 同一値更新

現在値と同一値が指定された場合は、業務エラーとしない。

Eloquentでは、Dirty判定により実際の変更がない場合にUPDATEが発行されないことがある。

概念的には、

```php
$assetAccount->save();
```

を使用し、Laravel標準のDirty判定に従ってよい。

---

### 27.32 updated_at

実際にEloquent上で変更が発生した場合は、

```text
updated_at
```

が通常どおり更新される。

クライアントから`updatedAt`を指定させない。

同一値更新でDirtyでない場合は、`updated_at`が変わらなくてもよい。

---

### 27.33 トランザクション

Phase1では、ACC-004専用の

```php
DB::transaction()
```

は必須としない。

更新対象が

```text
asset_accounts 1件
```

だけであり、関連テーブルを更新しないためである。

現在設定確認や名前重複確認はUPDATE前に行い、単一UPDATEで完結させる。

---

### 27.34 lockForUpdateを使用しない

ACC-004では、Phase1で

```php
lockForUpdate()
```

を使用しない。

同一資産口座への更新競合はLast Write Winsとなる可能性を許容する。

また、別資産口座間の同名競合はUNIQUE制約で保証する。

---

### 27.35 UNIQUE制約

同一利用者内の資産口座名は、DB側でも

```text
user_id
+
name
```

で一意性を保証する。

アプリケーション側の事前重複確認を同時実行した複数リクエストが通過しても、最終的にDB制約で重複を防止する。

---

### 27.36 UNIQUE制約違反の変換

並行更新などによってUNIQUE制約違反が発生した場合は、PostgreSQL例外をそのまま返却しない。

概念的には、

```text
unique violation
    ↓
AssetAccountNameAlreadyExistsException
    ↓
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

制約名やSQLSTATEはレスポンスへ公開しない。

---

### 27.37 Update Result DTO

更新結果は、専用DTOとして表現する。

概念例：

```php
final readonly class UpdateAssetAccountResult
{
    public function __construct(
        public int $id,
        public string $name,
        public AssetType $assetType,
        public BalanceRecordingUnit $balanceRecordingUnit,
        public bool $isAvailable,
        public string $startYearMonth,
    ) {
    }
}
```

`isEnabled`は、ACC-004成功時には必ず`true`となるため、Resource側で固定値として設定してよい。

---

### 27.38 DTOへEloquent Modelを保持しない

以下のようなDTOは基本としない。

```php
final readonly class UpdateAssetAccountResult
{
    public function __construct(
        public AssetAccount $assetAccount,
        public AssetAccountAvailableSetting $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

---

### 27.39 UseCaseの戻り値

概念例：

```php
return new UpdateAssetAccountResult(
    id:
        $assetAccount->id,

    name:
        $assetAccount->name,

    assetType:
        $assetAccount->asset_type,

    balanceRecordingUnit:
        $assetAccount->balance_recording_unit,

    isAvailable:
        $setting->is_available,

    startYearMonth:
        $assetAccount->start_year_month,
);
```

Eloquent ModelをActionへ直接返却しない。

---

### 27.40 API Resource

Update Result DTOを、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php
final class UpdateAssetAccountResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'name'
                => $this->name,

            'assetType'
                => $this->assetType->value,

            'balanceRecordingUnit'
                => $this->balanceRecordingUnit->value,

            'isAvailable'
                => $this->isAvailable,

            'startYearMonth'
                => $this->startYearMonth,

            'isEnabled'
                => true,
        ];
    }
}
```

APIフィールド名は、API共通方針に従ってcamelCaseとする。

---

### 27.41 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

- `user_id`
- `deleted_at`
- `created_at`
- `updated_at`
- `asset_account_available_settings.id`
- `asset_account_available_settings.asset_account_id`
- `asset_account_available_settings.start_year_month`
- `asset_account_available_settings.end_year_month`
- DB内部コード

また、以下も返却しない。

- 保有商品一覧
- 月末資産残高
- 商品別月末評価額
- 月末資産状況
- 資産推移

---

### 27.42 Responder

Responderは、Update Result DTOを`200 OK`レスポンスへ変換する。

概念例：

```php
final class UpdateAssetAccountResponder
{
    public function ok(
        UpdateAssetAccountResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new UpdateAssetAccountResource(
                        $result,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの共通Envelope項目は、API共通レスポンス処理に従う。

---

### 27.43 Responderで行わないこと

Responderでは、以下を行わない。

- 資産口座検索
- 利用者境界判定
- 資産口座名重複確認
- 現在年月取得
- 利用可能資産設定取得
- DB更新
- UNIQUE制約処理
- 業務例外判定

Responderは、生成済みDTOをHTTPレスポンスへ変換することに責務を限定する。

---

### 27.44 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `assetAccountId`形式不正 | `INVALID_ASSET_ACCOUNT_ID` |
| Request入力値不正 | `VALIDATION_ERROR` |
| 資産口座不存在 | `ASSET_ACCOUNT_NOT_FOUND` |
| 他利用者の資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 論理削除済み資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 同名資産口座存在 | `ASSET_ACCOUNT_NAME_ALREADY_EXISTS` |
| UNIQUE制約による同名競合 | `ASSET_ACCOUNT_NAME_ALREADY_EXISTS` |
| 現在設定0件 | `ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND` |
| 現在設定複数件 | `ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

---

### 27.45 AssetAccountNameAlreadyExistsException

同一利用者内に同名の別資産口座が存在する場合は、業務例外を送出する。

概念例：

```php
if ($exists) {
    throw new
        AssetAccountNameAlreadyExistsException();
}
```

最終的に、

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

へ変換する。

---

### 27.46 利用可能資産設定不整合例外

現在設定が0件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

へ変換する。

現在設定が複数件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

へ変換する。

どちらも500系のデータ不整合として扱う。

---

### 27.47 ValidationException

FormRequestによる入力値不正は、API共通形式の

```text
400 Bad Request
VALIDATION_ERROR
```

へ変換する。

複数のフィールドエラーがある場合は、`error.details`へ複数件返却してよい。

---

### 27.48 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- SQL
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

---

### 27.49 ログ

ACC-004では、必要に応じて以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assetAccountId
httpStatus
errorCode
```

`apiId`は、

```text
ACC-004
```

とする。

正常時には、更新対象IDを記録してよい。

ただし、資産口座名などの不要な業務情報を通常ログへ出力しない。

---

### 27.50 データ不整合ログ

以下の場合は、調査に必要な情報を内部ログへ記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text
assetAccountId
currentYearMonth
matchedSettingCount
```

を記録してよい。

---

### 27.51 キャッシュ

Phase1では、ACC-004専用のサーバー側キャッシュを使用しない。

React側でTanStack Queryを使用する場合は、ACC-004成功後に

```text
ACC-001
資産口座一覧

ACC-003
対象資産口座詳細
```

のQuery Cacheを無効化する。

---

### 27.52 テスト実装方針

Laravel側では、Feature Testを中心としてACC-004のAPI契約と部分更新処理を確認する。

主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `404 Not Found`
- `409 Conflict`
- `500 Internal Server Error`
- `X-User-Id`必須
- `assetAccountId`形式
- `name`のみ更新
- `assetType`のみ更新
- 複数項目更新
- 空リクエスト
- NULL
- 未定義項目
- 利用者境界
- SoftDeletes
- 資産口座名重複
- 論理削除済み資産口座との名前重複
- 現在利用可能資産設定確認
- UNIQUE制約
- 同時更新
- 関連データ非更新
- レスポンス契約

---

### 27.53 FormRequestのTest

FormRequestについて、以下を確認する。

```text
name
    sometimes
    required
    string
    max length

assetType
    sometimes
    required
    enum

name未指定
+
assetType未指定
    → error
```

以下も確認する。

- `name = null`はエラー
- `assetType = null`はエラー
- 空リクエストはエラー
- `name`だけは正常
- `assetType`だけは正常

---

### 27.54 AssetAccountQueryのDatabase Test

更新対象取得について、以下を確認する。

```text
id一致
+
user_id一致
+
deleted_at IS NULL
    ↓
取得できる
```

以下は取得できないこと。

- 存在しないID
- 他利用者の資産口座
- 論理削除済み資産口座

---

### 27.55 名前重複QueryのDatabase Test

以下を確認する。

```text
同一user_id
+
同一name
+
別id
    ↓
重複
```

```text
同一user_id
+
同一name
+
論理削除済み別id
    ↓
重複
```

```text
同一user_id
+
同一name
+
同じid
    ↓
重複ではない
```

```text
別user_id
+
同一name
    ↓
重複ではない
```

---

### 27.56 AssetAccountAvailableSettingQueryのDatabase Test

現在年月を固定し、現在有効な設定が正しく取得できることを確認する。

境界条件として、以下を確認する。

- `start_year_month = currentYearMonth`
- `end_year_month = currentYearMonth`
- `end_year_month = 前月`
- `start_year_month = 翌月`
- `end_year_month = NULL`

---

### 27.57 AssetAccountRepositoryのDatabase Test

`name`だけを渡した場合に、

```text
name
    → 更新

asset_type
    → 維持
```

となることを確認する。

`assetType`だけの場合は、

```text
asset_type
    → 更新

name
    → 維持
```

となることを確認する。

また、以下が変更されないことも確認する。

```text
user_id
balance_recording_unit
start_year_month
deleted_at
```

---

### 27.58 同一値更新Test

現在値と同じ値を指定する。

以下を確認する。

- 業務エラーにならない
- 最終業務状態が変わらない
- EloquentがDirtyでない場合は正常
- `updated_at`が変わらなくても正常

---

### 27.59 UseCaseのUnit Test

QueryとRepositoryをMockし、以下を確認する。

正常系：

```text
AssetAccountQuery
    ↓
資産口座取得
    ↓
AvailableSettingQuery
    ↓
現在設定1件
    ↓
必要なら名前重複確認
    ↓
Repository::update()
    ↓
Update Result DTO
```

対象不存在：

```text
AssetAccountQuery
    ↓
null
    ↓
AssetAccountNotFoundException
```

名前重複：

```text
DuplicateQuery
    ↓
true
    ↓
AssetAccountNameAlreadyExistsException
```

設定0件：

```text
AvailableSettingQuery
    ↓
0件
    ↓
AssetAccountAvailableSettingNotFoundException
```

設定複数件：

```text
AvailableSettingQuery
    ↓
2件以上
    ↓
AssetAccountAvailableSettingConflictException
```

---

### 27.60 name未指定時のTest

`assetType`だけを更新する場合に、名前重複確認Queryが呼び出されないことを確認する。

これにより、不要なDBアクセスを防止する。

---

### 27.61 UNIQUE制約競合Test

可能であれば、Integration TestまたはDatabase Testで、同一利用者の別資産口座を同じ名前へ並行更新する。

以下を確認する。

- 同名資産口座が2件存在しないこと
- 競合側がDB制約で失敗すること
- PostgreSQL例外がそのまま公開されないこと
- `ASSET_ACCOUNT_NAME_ALREADY_EXISTS`へ変換されること

---

### 27.62 API ResourceのTest

正常時に、以下の項目だけが`data`へ含まれることを確認する。

```json
{
  "id": "10",
  "name": "メイン証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "isAvailable": true,
  "startYearMonth": "2026-08",
  "isEnabled": true
}
```

以下が含まれないことを確認する。

- `userId`
- `user_id`
- `deletedAt`
- `createdAt`
- `updatedAt`
- `assetAccountAvailableSettingId`
- `asset_account_id`
- `endYearMonth`
- DB内部コード

---

### 27.63 関連データ非更新Test

ACC-004実行前後で、以下に変更がないことを確認する。

```text
asset_account_available_settings
holding_assets
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
net_incomes
objectives
assessment_histories
```

ACC-004の副作用が対象`asset_accounts`1件に限定されていることを保証する。

---

## 28. React・TypeScriptでの利用

ACC-004は、利用者が登録済み資産口座の通常属性を編集する際に使用する。

フロントエンドでは、ACC-003 資産口座詳細取得で現在値を取得し、編集フォームへ反映したうえでACC-004を実行する。

概念的な利用フローは、以下とする。

```text
ACC-003
資産口座詳細取得
    ↓
編集フォーム初期表示
    ↓
利用者が編集
    ↓
変更内容をRequestへ変換
    ↓
ACC-004
PATCH
/api/v1/asset-accounts/{assetAccountId}
    ↓
成功
    ↓
関連Query Cache無効化
    ↓
最新状態を再取得
```

ACC-004は、サーバー状態を変更するため、TanStack Queryを使用する場合はMutationとして扱う。

---

### 28.1 TypeScript型

ACC-004のリクエスト型は、以下のように定義する。

概念例：

```typescript
export type UpdateAssetAccountRequest = {
  name?: string;
  assetType?: AssetType;
};
```

`PATCH`であるため、各項目は任意とする。

ただし、実際にAPIを呼び出す際は、

```text
name
assetType
```

の少なくとも1項目を指定する必要がある。

---

### 28.2 正常レスポンス型

正常レスポンスは、ACC-003と同じ資産口座詳細表現を使用する。

概念例：

```typescript
export type UpdateAssetAccountResult = {
  id: string;
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  isAvailable: boolean;
  startYearMonth: string;
  isEnabled: boolean;
};
```

API共通Envelopeを使用する場合は、以下のように定義する。

```typescript
export type UpdateAssetAccountResponse =
  ApiResponse<UpdateAssetAccountResult>;
```

---

### 28.3 ACC-003と共通型を使用してよい

ACC-003とACC-004の正常レスポンス項目が同一である場合は、専用の重複型を作成せず、共通型を使用してよい。

概念例：

```typescript
export type AssetAccountDetail = {
  id: string;
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  isAvailable: boolean;
  startYearMonth: string;
  isEnabled: boolean;
};

export type UpdateAssetAccountResponse =
  ApiResponse<AssetAccountDetail>;
```

同じ意味のデータにAPIごとに異なる型名を増やしすぎない。

---

### 28.4 編集可能な項目

ACC-004で編集できる項目は、以下のみとする。

```text
name
assetType
```

編集画面では、これらを入力可能な状態とする。

---

### 28.5 編集できない項目

以下は、ACC-004では更新できない。

- `balanceRecordingUnit`
- `startYearMonth`
- `isAvailable`
- `isEnabled`

ACC-003から取得したこれらの情報を画面へ表示してもよいが、ACC-004用フォームでは編集可能な入力欄にしない。

---

### 28.6 balanceRecordingUnit

残高記録単位は、Phase1では登録後変更不可とする。

そのため、編集画面で表示する場合は読み取り専用とする。

概念例：

```tsx
<ReadonlyField
  label="残高記録単位"
  value={
    balanceRecordingUnitLabels[
      assetAccount.balanceRecordingUnit
    ]
  }
/>
```

以下のような編集可能なセレクトボックスにはしない。

```tsx
<select
  value={form.balanceRecordingUnit}
>
  ...
</select>
```

---

### 28.7 startYearMonth

利用開始年月も、ACC-004では変更不可とする。

画面へ表示する場合は、読み取り専用とする。

例えば、

```text
利用開始年月
2026年8月
```

のように表示する。

---

### 28.8 isAvailable

利用可能資産区分の変更は、ACC-006の責務である。

そのため、ACC-004の編集フォームへ

```typescript
isAvailable: boolean;
```

を更新項目として含めない。

利用可能資産区分を変更するUIは、ACC-006を実行する別操作として設計する。

---

### 28.9 isEnabled

資産口座無効化は、ACC-005の責務とする。

そのため、ACC-004のRequestへ

```typescript
isEnabled?: boolean;
```

を追加しない。

以下のような汎用的な状態更新は行わない。

```typescript
updateAssetAccount(
  assetAccountId,
  {
    isEnabled: false,
  },
);
```

---

### 28.10 編集フォーム型

画面上の編集フォームStateは、以下のように定義できる。

概念例：

```typescript
export type UpdateAssetAccountFormValues = {
  name: string;
  assetType: AssetType;
};
```

ACC-003から取得した編集不可項目までフォームStateへ含める必要はない。

---

### 28.11 ACC-003から初期値を設定する

編集画面では、ACC-003で取得した

```text
name
assetType
```

を初期値として使用する。

概念例：

```typescript
const initialValues:
  UpdateAssetAccountFormValues = {
    name:
      assetAccount.name,

    assetType:
      assetAccount.assetType,
  };
```

---

### 28.12 Queryデータを直接編集しない

ACC-003で取得したQuery Cache上の

```text
AssetAccountDetail
```

を、フォーム入力によって直接書き換えない。

例えば、

```typescript
assetAccount.name =
  inputValue;
```

のような変更は行わない。

編集用Stateへ必要な値をコピーする。

---

### 28.13 変更項目だけを送信する

ACC-004はPATCHであるため、実際に変更された項目だけをRequestへ含めてよい。

例えば、資産口座名だけを変更した場合は、

```json
{
  "name": "メイン証券口座"
}
```

とする。

---

### 28.14 assetTypeだけ変更した場合

資産種別だけを変更した場合は、

```json
{
  "assetType": "BANK"
}
```

とする。

未変更の`name`を必ず送信する必要はない。

---

### 28.15 両方変更した場合

両方変更した場合は、

```json
{
  "name": "メイン証券口座",
  "assetType": "SECURITIES"
}
```

とする。

---

### 28.16 Request生成

現在値とフォーム値を比較し、変更された項目だけをRequestへ設定してよい。

概念例：

```typescript
export const buildUpdateAssetAccountRequest =
  (
    current: AssetAccountDetail,
    form: UpdateAssetAccountFormValues,
  ): UpdateAssetAccountRequest => {
    const request:
      UpdateAssetAccountRequest = {};

    if (
      form.name !== current.name
    ) {
      request.name =
        form.name;
    }

    if (
      form.assetType
        !== current.assetType
    ) {
      request.assetType =
        form.assetType;
    }

    return request;
  };
```

---

### 28.17 空Requestを送信しない

変更がない場合は、

```json
{}
```

をACC-004へ送信しない。

概念例：

```typescript
const request =
  buildUpdateAssetAccountRequest(
    assetAccount,
    form,
  );

if (
  Object.keys(request).length === 0
) {
  return;
}
```

画面上では、例えば

```text
変更内容がありません。
```

と表示してよい。

または、変更がない間は更新ボタンを非活性化してもよい。

---

### 28.18 更新ボタンの活性制御

現在値とフォーム値が同一の場合は、更新ボタンを非活性化してよい。

概念例：

```typescript
const hasChanges =
  form.name !==
    assetAccount.name
  ||
  form.assetType !==
    assetAccount.assetType;
```

```tsx
<button
  type="submit"
  disabled={
    !hasChanges
    || updateMutation.isPending
  }
>
  更新
</button>
```

ただし、サーバー側でも空Requestを拒否する。

---

### 28.19 nameの前処理

資産口座名について、API共通の文字列入力方針で前後空白を除去する場合は、Request生成時に統一して処理してよい。

概念例：

```typescript
const name =
  form.name.trim();
```

ただし、フロントエンドとバックエンドで異なる正規化ルールを持たせない。

---

### 28.20 AssetType

`assetType`は、ACC-002・ACC-003と共通の型を使用する。

概念例：

```typescript
export type AssetType =
  | 'CASH'
  | 'BANK'
  | 'SECURITIES'
  | 'IDECO'
  | 'CORPORATE_DC'
  | 'OTHER';
```

画面表示用ラベルは、共通定義を使用する。

---

### 28.21 API Client

ACC-004を呼び出す専用API Client関数を定義する。

概念例：

```typescript
export const updateAssetAccount =
  async (
    assetAccountId: string,
    request: UpdateAssetAccountRequest,
  ): Promise<AssetAccountDetail> => {
    const response =
      await apiClient.patch<
        UpdateAssetAccountResponse
      >(
        `/api/v1/asset-accounts/${assetAccountId}`,
        request,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

### 28.22 userIdをAPI Client引数へ含めない

以下のような関数にはしない。

```typescript
updateAssetAccount(
  userId,
  assetAccountId,
  request,
);
```

利用者IDは、共通API Clientから

```text
X-User-Id
```

として付与する。

---

### 28.23 X-User-Id

概念例：

```typescript
apiClient.interceptors.request.use(
  (config) => {
    config.headers['X-User-Id'] =
      currentUserId;

    return config;
  },
);
```

ACC-004専用コンポーネントでヘッダーを直接生成しない。

---

### 28.24 Mutationとして扱う

ACC-004は、資産口座の状態を変更するため、TanStack QueryではMutationとして扱う。

概念例：

```typescript
export type UpdateAssetAccountVariables = {
  assetAccountId: string;
  request: UpdateAssetAccountRequest;
};
```

```typescript
export const useUpdateAssetAccount =
  () => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn: (
        variables:
          UpdateAssetAccountVariables,
      ) =>
        updateAssetAccount(
          variables.assetAccountId,
          variables.request,
        ),

      onSuccess: async (
        result,
      ) => {
        await queryClient
          .invalidateQueries({
            queryKey:
              assetAccountKeys.all,
          });

        await queryClient
          .invalidateQueries({
            queryKey:
              assetAccountKeys.detail(
                result.id,
              ),
          });
      },
    });
  };
```

---

### 28.25 更新処理

概念例：

```typescript
const updateMutation =
  useUpdateAssetAccount();

const handleSubmit = (): void => {
  const request =
    buildUpdateAssetAccountRequest(
      assetAccount,
      form,
    );

  if (
    Object.keys(request).length === 0
  ) {
    return;
  }

  updateMutation.mutate({
    assetAccountId:
      assetAccount.id,

    request,
  });
};
```

---

### 28.26 二重送信防止

Mutation実行中は、更新ボタンを非活性化する。

概念例：

```tsx
<button
  type="submit"
  disabled={
    updateMutation.isPending
  }
>
  {updateMutation.isPending
    ? '更新中...'
    : '更新'}
</button>
```

フロントエンド側の二重送信防止だけをサーバー側整合性保証とはしない。

---

### 28.27 自動Retry

ACC-004のMutationでは、原則として自動Retryを行わない。

概念例：

```typescript
useMutation({
  mutationFn:
    updateAssetAccount,
  retry: false,
});
```

更新処理であり、通信結果が不明な状態で自動再送するより、利用者へ状態を通知したうえで必要に応じて再取得する方針とする。

---

### 28.28 同一Requestの再送

ACC-004は、同じ値を再度設定しても最終的な業務状態は同じとなる。

そのため、利用者が手動で再実行すること自体は可能である。

ただし、ネットワークエラー時にフロントエンドが自動的に繰り返し送信する設計はPhase1では採用しない。

---

### 28.29 成功時

ACC-004成功時は、

```http
200 OK
```

とともに、更新後の資産口座詳細が返却される。

概念例：

```json
{
  "data": {
    "id": "10",
    "name": "メイン証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

---

### 28.30 成功メッセージ

正常終了後は、必要に応じて

```text
資産口座を更新しました。
```

などの完了メッセージを表示する。

---

### 28.31 成功後の画面遷移

更新成功後は、画面設計に応じて

- 資産口座詳細画面へ戻る
- 資産口座一覧画面へ戻る
- 編集画面に留まり最新値を表示する

などの動作とする。

Phase1では、詳細画面または一覧画面へ戻るシンプルな導線としてよい。

---

### 28.32 ACC-001のQuery Cache

ACC-004によって、

```text
name
assetType
```

が変更されるため、資産口座一覧のQuery Cacheを無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.all,
});
```

---

### 28.33 ACC-003のQuery Cache

対象資産口座の詳細Queryも無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

これにより、ACC-003から更新後の最新状態を取得する。

---

### 28.34 成功レスポンスでCacheを更新する方式

ACC-004の成功レスポンスは、ACC-003と同じ資産口座詳細表現であるため、成功結果を直接詳細Query Cacheへ設定することもできる。

概念例：

```typescript
queryClient.setQueryData(
  assetAccountKeys.detail(
    result.id,
  ),
  result,
);
```

ただし、Phase1では

```text
Mutation成功
    ↓
invalidate
    ↓
再取得
```

の単純な方式を基本としてよい。

---

### 28.35 Optimistic Update

Phase1では、ACC-004に対するOptimistic Updateを必須としない。

資産口座名については、

- 同名重複
- 他利用者境界
- 論理削除状態
- UNIQUE制約競合

など、サーバー側で最終判定する条件がある。

そのため、サーバー成功後に画面状態を更新する方式を基本とする。

---

### 28.36 VALIDATION_ERROR

Laravel側から

```text
VALIDATION_ERROR
```

が返却された場合は、`error.details`を利用して対応するフォーム項目へエラー表示する。

主な対象は、

```text
name
assetType
```

とする。

---

### 28.37 nameのフィールドエラー

例えば、

```text
資産口座名を入力してください。
```

や、

```text
資産口座名は100文字以内で入力してください。
```

などを、資産口座名入力欄の近くへ表示する。

具体的な文言は、APIレスポンスまたはフロントエンド共通エラー表示方針に従う。

---

### 28.38 assetTypeのフィールドエラー

`assetType`に未定義値が返された場合は、資産種別入力欄のエラーとして扱う。

通常のUIでは定義済みセレクト値しか送信しないため、発生頻度は低い。

---

### 28.39 ASSET_ACCOUNT_NAME_ALREADY_EXISTS

同一利用者内に同名資産口座が存在する場合は、

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となる。

フロントエンドでは、例えば

```text
同じ名前の資産口座が
すでに登録されています。
```

と表示する。

`name`に関連する業務エラーとして、資産口座名入力欄の近くへ表示してよい。

---

### 28.40 同時更新による重複も同じ扱いとする

DBのUNIQUE制約によって名前競合が検出された場合も、フロントエンドでは

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

として扱う。

アプリケーション事前判定か、DB制約による判定かをクライアントから区別しない。

---

### 28.41 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

となる。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

フロントエンドでは、原因を推測せず、

```text
指定された資産口座が見つかりません。
```

などの共通表示とする。

---

### 28.42 更新中に無効化された場合

編集画面表示後、別処理によって対象資産口座が無効化される可能性がある。

その後ACC-004を実行すると、

```text
ASSET_ACCOUNT_NOT_FOUND
```

となり得る。

この場合は、編集画面へ残して再送を繰り返さず、資産口座一覧へ戻る導線を表示してよい。

---

### 28.43 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-004専用のフォームエラーとして表示しない。

---

### 28.44 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、API共通の利用者コンテキストエラーとして扱う。

---

### 28.45 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択されている利用者が有効ではない状態として共通処理する。

---

### 28.46 INVALID_ASSET_ACCOUNT_ID

通常の画面遷移では、ACC-001やACC-003から取得した有効なIDを使用するため、発生頻度は低い。

発生した場合は、不正なURLまたは画面状態として扱う。

---

### 28.47 ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND

現在の利用可能資産設定が存在しない場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

となる。

これは、フォーム入力による問題ではなくサーバー側データ不整合である。

画面では、一般的な更新失敗として扱う。

例えば、

```text
資産口座を更新できませんでした。
```

と表示する。

---

### 28.48 ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT

現在有効な利用可能資産設定が複数存在する場合も、サーバー側データ不整合として扱う。

フロントエンドで利用可能資産設定を独自に選択して更新処理を継続しない。

---

### 28.49 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
資産口座を更新できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

### 28.50 エラー時のフォーム値

ACC-004が失敗した場合は、原則として利用者が入力したフォーム値を保持する。

入力内容を自動的に初期値へ戻さない。

これにより、入力内容を修正して再実行できるようにする。

---

### 28.51 ASSET_ACCOUNT_NOT_FOUND時のフォーム値

ただし、

```text
ASSET_ACCOUNT_NOT_FOUND
```

の場合は、対象資産口座自体が編集不能な可能性がある。

そのため、一覧へ戻すなど画面遷移を優先してよい。

---

### 28.52 サーバー状態との競合

Phase1では`version`や`If-Match`を使用しないため、編集画面表示後に別リクエストで同じ資産口座が更新されていても、ACC-004は更新競合エラーを返さない。

同一項目を更新した場合は、Last Write Winsとなる可能性がある。

フロントエンドでも独自の競合判定を実装しない。

---

### 28.53 updatedAtを送信しない

競合判定用として

```typescript
updatedAt: string;
```

をACC-004のRequestへ追加しない。

Phase1では、更新時刻比較による楽観ロックを採用しないためである。

---

### 28.54 balanceRecordingUnit変更UIを設けない

ACC-004では残高記録単位を変更できないため、編集画面に

```text
ACCOUNT
HOLDING
```

を切り替える入力UIを設けない。

表示する場合は読み取り専用とする。

---

### 28.55 利用可能資産変更UIとの分離

資産口座編集画面に利用可能資産変更機能を同時に配置する場合でも、内部的にはACC-004とは別操作として扱う。

概念的には、

```text
基本情報保存
    ↓
ACC-004

利用可能資産区分変更
    ↓
ACC-006
```

とする。

1つのACC-004 Requestへまとめない。

---

### 28.56 無効化UIとの分離

無効化ボタンを同じ画面へ配置する場合でも、

```text
通常属性更新
    → ACC-004

無効化
    → ACC-005
```

とAPIを分離する。

例えば、「保存」ボタンと「無効化」ボタンを別操作として扱う。

---

### 28.57 Pageの責務

編集Pageでは、主に以下を担当する。

- URLから`assetAccountId`取得
- ACC-003による初期データ取得
- 編集フォーム表示
- ACC-004成功後の画面遷移
- ページ単位のエラー表示

HTTP通信の詳細を直接記述しない。

---

### 28.58 Form Componentの責務

Form Componentでは、主に以下を担当する。

- `name`入力
- `assetType`入力
- クライアント側入力チェック
- フィールドエラー表示
- submitイベント通知

ACC-004のHTTP通信を直接実行しない構成としてよい。

---

### 28.59 Mutation Hookの責務

Mutation Hookでは、主に以下を担当する。

```text
ACC-004実行
成功処理
Query Cache無効化
```

画面固有の表示レイアウトをMutation Hookへ持たせない。

---

### 28.60 API Clientの責務

API Clientでは、

```http
PATCH /api/v1/asset-accounts/{assetAccountId}
```

のHTTP通信と型付きレスポンス取得を担当する。

画面遷移やToast表示などはAPI Clientへ含めない。

---

### 28.61 概念的なディレクトリ構成

例えば、以下のように機能単位で整理できる。

```text
features/
└── asset-accounts/
    ├── api/
    │   ├── getAssetAccountDetail.ts
    │   └── updateAssetAccount.ts
    ├── components/
    │   └── AssetAccountForm.tsx
    ├── hooks/
    │   ├── useAssetAccountDetail.ts
    │   └── useUpdateAssetAccount.ts
    ├── types/
    │   └── assetAccount.ts
    └── pages/
        └── EditAssetAccountPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

### 28.62 フロントエンドで行わないこと

ACC-004のReact・TypeScript実装では、以下をフロントエンドの責務としない。

- 利用者境界の最終保証
- 資産口座存在確認の最終保証
- 論理削除判定
- 資産口座名一意性の最終保証
- DBのUNIQUE競合判定
- DB内部コードへの変換
- 現在利用可能資産設定の期間判定
- データ不整合の補正
- `balanceRecordingUnit`変更
- `startYearMonth`変更
- `isAvailable`変更
- `isEnabled`変更
- 楽観ロック判定

フロントエンドは、

```text
現在値取得
    ↓
編集可能項目の入力
    ↓
変更Request生成
    ↓
ACC-004実行
    ↓
結果表示
```

に責務を限定する。

---

## 29. 設計上の補足

### 29.1 通常属性更新に責務を限定する理由

ACC-004は、資産口座の通常属性を更新するAPIとする。

更新対象は、

```text
name
assetType
```

に限定する。

一方、

```text
balanceRecordingUnit
startYearMonth
isAvailable
isEnabled
```

は扱わない。

概念的には、

```text
ACC-004
    ↓
通常属性更新
    ├─ name
    └─ assetType
```

とする。

資産口座のライフサイクルや履歴管理を伴う操作まで同じAPIへ含めない。

---

### 29.2 PATCHを採用する理由

ACC-004では、資産口座リソース全体を置き換えるのではなく、指定された一部属性だけを更新する。

そのため、

```http
PATCH /api/v1/asset-accounts/{assetAccountId}
```

を使用する。

例えば、

```json
{
  "name": "メイン証券口座"
}
```

の場合は、`name`だけを更新し、

```text
asset_type
balance_recording_unit
start_year_month
```

などの未指定項目を変更しない。

---

### 29.3 PUTを採用しない理由

`PUT`は、リソース全体の置換として解釈されることがある。

ACC-004では、

```text
更新したい項目だけを送信する
```

という部分更新を前提とするため、`PUT`ではなく`PATCH`を採用する。

---

### 29.4 残高記録単位を更新対象外とする理由

`balanceRecordingUnit`は、資産口座登録後の月末資産管理方法を決定する重要な属性である。

例えば、

```text
ACCOUNT
    ↓
HOLDING
```

へ変更すると、

- 既存の月末資産残高
- 保有商品
- 商品別月末評価額

との整合性ルールが必要になる。

そのため、Phase1では登録後の変更を許可しない。

---

### 29.5 利用開始年月を更新対象外とする理由

`startYearMonth`を変更すると、

- 利用可能資産設定履歴
- 月末資産管理対象
- 過去の資産記録
- 対象年月判定

へ影響する。

単純な属性変更ではなく、既存履歴の意味まで変化する可能性があるため、ACC-004では更新しない。

---

### 29.6 利用可能資産区分を更新対象外とする理由

利用可能資産区分は、対象年月によって変化する履歴情報として管理する。

そのため、

```text
asset_accounts.is_available
```

のような通常属性として更新しない。

変更は、

```text
ACC-006
利用可能資産設定登録
```

で扱う。

概念的には、

```text
ACC-004
    → 通常属性

ACC-006
    → 時系列設定
```

と責務を分離する。

---

### 29.7 無効化を更新対象外とする理由

資産口座無効化は、

```text
現在利用中
    ↓
今後の管理対象外
```

というライフサイクル変更である。

単なる通常属性更新とは業務上の意味が異なるため、

```text
ACC-005
資産口座無効化
```

へ分離する。

ACC-004のRequestへ

```json
{
  "isEnabled": false
}
```

のような状態指定を追加しない。

---

### 29.8 userIdを更新対象にしない理由

資産口座の所有利用者は、登録時に確定する。

ACC-004から

```text
User A
    ↓
User B
```

へ資産口座を移管する機能は提供しない。

そのため、

```text
userId
user_id
```

は更新対象に含めない。

---

### 29.9 更新対象項目を明示的に限定する理由

ACC-004では、Request全体をそのままEloquentへ渡さない。

例えば、

```php
$assetAccount->update(
    $request->all(),
);
```

のような実装は避ける。

更新対象を

```text
name
asset_type
```

へ明示的に限定することで、

- `user_id`
- `deleted_at`
- `balance_recording_unit`
- `start_year_month`

などの意図しない変更を防止する。

---

### 29.10 空PATCHを許可しない理由

以下のような更新項目が存在しないRequestは受け付けない。

```json
{}
```

ACC-004は資産口座を更新するためのAPIであり、変更対象が1件もないRequestは業務上意味を持たないためである。

そのため、

```text
name
または
assetType
```

の少なくとも1項目を必須とする。

---

### 29.11 未指定項目とNULLを区別する理由

PATCHでは、

```text
項目未指定
```

と

```text
項目にNULLを指定
```

は異なる意味を持つ。

ACC-004では、

```text
未指定
    → 変更しない

NULL指定
    → バリデーションエラー
```

とする。

例えば、

```json
{
  "assetType": "SECURITIES"
}
```

では、`name`を変更しない。

一方、

```json
{
  "name": null
}
```

は不正とする。

---

### 29.12 同一値更新を許可する理由

現在値と同じ値が指定された場合でも、業務エラーとはしない。

例えば、

```text
現在
name = 証券口座
```

に対して、

```json
{
  "name": "証券口座"
}
```

を送信しても正常なRequestとして扱う。

フロントエンド側では変更なしとして送信を省略してよいが、サーバー側でも同一値指定をエラーにしない。

---

### 29.13 NO_CHANGESエラーを設けない理由

同一値が指定された場合に、

```text
NO_CHANGES
ALREADY_UPDATED
```

などの専用業務エラーは定義しない。

同一値更新は有害な操作ではなく、最終業務状態も変化しないためである。

---

### 29.14 同一利用者内で資産口座名を一意とする理由

資産口座名は、画面やCSVなどで利用者が資産口座を識別するために使う。

同一利用者に

```text
証券口座
証券口座
```

が存在すると、利用者から見て識別しにくくなる。

そのため、

```text
user_id
+
name
```

を一意とする。

---

### 29.15 更新対象自身を重複判定から除外する理由

更新対象自身の現在の名前と同じ名前を指定した場合まで重複とすると、正常な同一値更新ができない。

そのため、重複確認では、

```text
id != assetAccountId
```

を条件へ含める。

---

### 29.16 論理削除済み資産口座も重複対象とする理由

ACC-002と同じく、論理削除済み資産口座も同名重複対象とする。

これにより、過去に存在した資産口座と現在の資産口座が同じ名前になることを防止する。

履歴参照やCSVなどで名称による識別が曖昧になることを避ける。

---

### 29.17 異なる利用者間では同名を許可する理由

資産口座名の一意性は、利用者ごとに保証すればよい。

例えば、

```text
User A
    証券口座

User B
    証券口座
```

は問題ない。

そのため、システム全体で資産口座名を一意にはしない。

---

### 29.18 DBのUNIQUE制約も使用する理由

アプリケーション側で更新前に重複確認をしても、並行更新では競合を完全に防止できない。

例えば、

```text
Request A
重複なし確認

Request B
重複なし確認

Request A
UPDATE

Request B
UPDATE
```

となる可能性がある。

そのため、

```text
user_id
+
name
```

のUNIQUE制約をDB側にも設定する。

---

### 29.19 アプリケーション側でも重複確認する理由

DB制約だけに依存すると、すべての同名エラーがDB例外として発生する。

通常ケースでは、事前に業務上の重複を判定し、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

として分かりやすく返却する。

DB制約は、同時更新時などの最終防衛線とする。

---

### 29.20 `lockForUpdate()`を使用しない理由

同一資産口座を行ロックしても、別々の資産口座を同じ名前へ変更する競合は防止できない。

例えば、

```text
Asset Account A
    lock

Asset Account B
    lock
```

は、別行なので双方取得できる。

名前一意性はUNIQUE制約で保証するため、Phase1では`lockForUpdate()`を使用しない。

---

### 29.21 利用者行をロックしない理由

同一利用者の更新をすべて直列化するために

```text
users
```

の行をロックする方式は採用しない。

資産口座更新以外の同一利用者処理まで不要に待機させるためである。

---

### 29.22 Last Write Winsを許容する理由

Phase1では、同一資産口座への同時編集を高頻度利用ケースとして想定しない。

そのため、同じ属性に対する競合更新についてはLast Write Winsとなる可能性を許容する。

例えば、

```text
Request A
name = A

Request B
name = B
```

では、最終的に後から反映された値が残る。

---

### 29.23 楽観ロックを採用しない理由

Phase1では、

```text
version
updatedAt
ETag
If-Match
```

などの競合検出機構を導入しない。

これらを導入すると、

- version管理
- Requestへの競合情報追加
- 409等の競合エラー
- React側の競合解決UI

が必要になる。

Phase1の利用規模に対して複雑性が大きいため、将来課題とする。

---

### 29.24 明示的なトランザクションを必須としない理由

ACC-004で更新する業務データは、

```text
asset_accounts 1件
```

だけである。

関連テーブルとの複数更新を行わないため、Phase1では

```php
DB::transaction()
```

を必須としない。

単一UPDATEの原子性はDBが保証する。

---

### 29.25 利用可能資産設定を更新前に確認する理由

ACC-004の成功レスポンスでは、

```text
isAvailable
```

も返却する。

そのため、現在有効な利用可能資産設定に不整合がある状態で資産口座だけ更新してから500エラーとなる構成は避ける。

概念的には、

```text
現在設定確認
    ↓
正常
    ↓
asset_accounts更新
```

とする。

---

### 29.26 利用可能資産設定の0件を補完しない理由

現在有効な設定が存在しない場合に、

```text
isAvailable = false
```

として正常レスポンスを返すと、データ不整合を隠蔽する。

ACC-002で初期設定を必ず作成する設計のため、0件は正常状態とはしない。

---

### 29.27 利用可能資産設定の複数件を補正しない理由

現在有効な設定が複数存在する場合に、

```text
最新1件を採用
```

すると、期間重複を隠蔽する。

ACC-004ではデータ修復を行わず、データ不整合として扱う。

---

### 29.28 ACC-004で利用可能資産設定を更新しない理由

成功レスポンス生成のために

```text
asset_account_available_settings
```

を参照するが、更新は行わない。

参照することと変更責務を持つことは別である。

利用可能資産設定の変更は、ACC-006へ限定する。

---

### 29.29 assetType変更時に関連データを自動変更しない理由

資産種別を変更しても、

- 保有商品
- 月末残高
- 商品別月末評価額
- 利用可能資産設定

を自動変更しない。

`assetType`は資産口座の分類情報であり、ACC-004では関連データの構造変更まで行わない。

---

### 29.30 保有商品を更新しない理由

資産口座と保有商品は、別リソースとして扱う。

そのため、資産口座の名前や資産種別を変更しても、

```text
holding_assets
```

を更新しない。

保有商品の変更はHLD APIの責務とする。

---

### 29.31 月末資産データを更新しない理由

資産口座の通常属性変更によって、過去に記録した

```text
month_end_asset_balances
month_end_holding_values
month_end_asset_snapshots
```

を変更しない。

過去時点の資産記録そのものは維持する。

---

### 29.32 Repositoryを使用する理由

ACC-004では、実際に

```text
asset_accounts
```

を更新する。

そのため、

```text
Query
    → 読み取り

Repository
    → 書き込み
```

という責務分離方針に従い、更新処理をRepositoryへ置く。

---

### 29.33 QueryとRepositoryを分ける理由

ACC-004では、

```text
AssetAccountQuery
    → 更新対象取得
    → 名前重複確認

AssetAccountRepository
    → asset_accounts更新
```

と分離する。

これにより、検索条件と状態変更処理が混在することを防止する。

---

### 29.34 UseCaseを設ける理由

ACC-004では、

```text
資産口座取得
    ↓
現在設定確認
    ↓
名前重複確認
    ↓
部分更新
    ↓
結果生成
```

という業務処理の流れを持つ。

Actionへこれらを直接記述せず、UseCaseへオーケストレーションを集約する。

---

### 29.35 FormRequestを使用する理由

ACC-004には、

```text
name
assetType
```

というRequest Bodyが存在する。

さらに、

```text
PATCHの未指定項目
NULL禁止
Enum
最大文字数
最低1項目
```

などのHTTP入力検証が必要になる。

そのため、専用FormRequestを使用する。

---

### 29.36 PATCHのInput DTOを使う理由

PATCHでは、

```text
未指定
```

と

```text
入力済み
```

の区別が必要になる。

そのため、ActionからUseCaseへRequestオブジェクトそのものを渡さず、更新対象を明確にしたInput DTOへ変換してよい。

---

### 29.37 DTOでNULLを未指定として使用する際の注意

ACC-004では、APIとしてNULL更新を許可しない。

そのため、Input DTO上の

```text
?string
?AssetType
```

の`null`は、

```text
Request項目未指定
```

という意味だけに使用する。

入力値として送信されたJSONの`null`とはFormRequestで区別する。

---

### 29.38 Mass Assignmentを避ける理由

ACC-004では更新対象が明確に限定されている。

Request配列をそのままEloquentへ渡すより、

```text
name指定あり
    → name変更

assetType指定あり
    → asset_type変更
```

と明示的に処理することで、更新範囲をコード上でも明確にできる。

---

### 29.39 EloquentのDirty判定を利用してよい理由

現在値と同一値の場合、Eloquentは変更なしと判定できる。

この場合に、無理にUPDATEを発行する必要はない。

そのため、

```text
同一値
    ↓
save()
    ↓
Dirtyなし
```

というLaravel標準動作を利用してよい。

---

### 29.40 `updated_at`をAPIへ返さない理由

Phase1では、`updated_at`を楽観ロックや画面表示要件に使用しない。

そのため、ACC-004成功レスポンスへ`updatedAt`を含めない。

DB内部の管理日時として扱う。

---

### 29.41 成功レスポンスをACC-003と揃える理由

ACC-004成功後にクライアントが必要とするのは、更新後の資産口座状態である。

そのため、ACC-003と同じ意味を持つ

```text
id
name
assetType
balanceRecordingUnit
isAvailable
startYearMonth
isEnabled
```

を返却する。

これにより、React側で共通型を利用しやすくする。

---

### 29.42 ACC-004で更新していない値も返す理由

ACC-004で直接変更するのは、

```text
name
assetType
```

だけである。

ただし、成功レスポンスを資産口座詳細として完結させるため、

```text
balanceRecordingUnit
isAvailable
startYearMonth
isEnabled
```

も返却する。

これにより、クライアントが更新後の資産口座状態を1つのレスポンスで確認できる。

---

### 29.43 `isEnabled = true`のみ返る理由

ACC-004では、論理削除済み資産口座を更新できない。

そのため、正常レスポンスでは

```text
isEnabled = true
```

となる。

`false`を返すケースはACC-004の正常系には存在しない。

---

### 29.44 `deletedAt`を返さない理由

SoftDeletesの

```text
deleted_at
```

はDB内部実装である。

APIでは業務上の状態として

```text
isEnabled
```

を返却する。

---

### 29.45 Optimistic Updateを必須としない理由

ACC-004では、

- 名前重複
- UNIQUE制約競合
- 利用者境界
- 論理削除
- 利用可能資産設定不整合

など、サーバー側で最終判定する条件がある。

そのため、Phase1では先に画面を更新するOptimistic Updateを必須としない。

---

### 29.46 Query invalidateを基本とする理由

ACC-004成功後は、

```text
ACC-001
資産口座一覧

ACC-003
資産口座詳細
```

の表示内容が変化する可能性がある。

そのため、

```text
Mutation成功
    ↓
Query invalidate
    ↓
最新状態再取得
```

を基本とする。

---

### 29.47 成功レスポンスをQuery Cacheへ直接設定できる理由

ACC-004成功レスポンスは、ACC-003と同じ詳細表現を返す。

そのため、

```text
ACC-004 response
    ↓
ACC-003 detail cache
```

へ直接設定することも可能である。

ただし、Phase1では実装の単純さを優先してinvalidate方式を基本としてよい。

---

### 29.48 Mutationの自動Retryを行わない理由

ACC-004は更新APIである。

同一入力は最終状態として冪等だが、通信結果不明時に自動再送すると、ユーザーから見た処理状況が分かりにくくなる。

Phase1ではMutationの自動Retryを原則無効とする。

---

### 29.49 Idempotency-Keyを使用しない理由

ACC-004は、

```text
指定属性を
指定値へ更新する
```

処理であり、履歴レコード追加や加算処理ではない。

同一入力の再実行で業務状態が増殖しないため、Phase1ではIdempotency-Keyを使用しない。

---

### 29.50 ACC-003を編集画面初期値に使用する理由

ACC-004専用に「編集用データ取得API」を追加しない。

ACC-003で現在の資産口座詳細を取得し、その中から

```text
name
assetType
```

を編集フォーム初期値へ使用する。

API数を不要に増やさない。

---

### 29.51 編集不可項目を画面で読み取り専用にする理由

ACC-003では、

```text
balanceRecordingUnit
startYearMonth
isAvailable
isEnabled
```

も取得できる。

ただし、これらをACC-004で変更できるようにはしない。

同じ画面に表示する場合でも、

```text
表示
    ≠
編集可能
```

として扱う。

---

### 29.52 ACC-005との責務分離

資産口座無効化は、ACC-004に含めない。

概念的には、

```text
保存ボタン
    ↓
ACC-004

無効化ボタン
    ↓
ACC-005
```

とする。

同じ編集画面に両操作を配置することはできるが、APIとしては分離する。

---

### 29.53 ACC-006との責務分離

利用可能資産区分の変更も、ACC-004へ含めない。

画面上で基本情報と利用可能資産区分を同時に見せる場合でも、

```text
基本情報変更
    → ACC-004

利用可能資産設定変更
    → ACC-006
```

として別Mutationにする。

---

### 29.54 HLD APIとの責務分離

資産口座が`HOLDING`管理であっても、ACC-004で保有商品を更新しない。

保有商品の

- 名前
- 商品種別
- 備考
- 無効化

などはHLD APIで扱う。

---

### 29.55 Phase1では更新履歴を保存しない

ACC-004による

```text
name
assetType
```

の変更履歴を専用履歴テーブルへ保存しない。

Phase1では、現在値のみを`asset_accounts`へ保持する。

将来的に監査要件が生じた場合は、変更履歴テーブルや監査ログの導入を検討する。

---

### 29.56 Phase1では設計を広げすぎない

ACC-004では、以下の機能を追加しない。

- 残高記録単位変更
- 利用開始年月変更
- 利用可能資産区分変更
- 資産口座無効化
- 資産口座再有効化
- 所有利用者変更
- 更新履歴保存
- 楽観ロック
- ETag
- Idempotency-Key
- 複数資産口座一括更新

Phase1では、

```text
name
+
assetType
```

の通常属性更新へ責務を限定する。

---

## 30. 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)

---

# ACC-005 利用可能資産設定履歴取得

## 1. 概要

操作対象となる利用者に帰属する指定された資産口座について、利用可能資産設定の履歴一覧を取得する。

利用可能資産区分は、資産口座の固定属性としてではなく、

```text
asset_account_available_settings
```

によって年月単位の履歴として管理する。

ACC-005では、指定された資産口座に紐づく利用可能資産設定履歴を取得し、過去から現在までの設定変更内容を確認できるようにする。

主に以下の情報を返却する。

- 利用可能資産設定ID
- 適用開始年月
- 適用終了年月
- 利用可能資産区分

概念的には、以下の情報を取得する。

```text
asset_accounts
    ↓
操作対象利用者との
利用者境界確認
    ↓
asset_account_available_settings
    ↓
設定履歴一覧
```

本APIは参照専用であり、資産口座および利用可能資産設定を更新しない。

---

### 1.1 利用可能資産設定とは

利用可能資産設定は、対象資産口座を目的達成判定に利用する資産として扱うかどうかを表す。

例えば、

```text
2026-01 ～ 2026-06
利用可能

2026-07 ～ 2026-12
利用対象外

2027-01 ～
利用可能
```

のように、年月単位で利用可能資産区分が変化する可能性がある。

そのため、現在値だけではなく設定履歴を保持する。

---

### 1.2 ACC-003との違い

ACC-003 資産口座詳細取得では、現在年月に有効な利用可能資産設定だけを取得し、

```text
isAvailable
```

として返却する。

一方、ACC-005では、指定された資産口座に対する利用可能資産設定履歴を一覧として返却する。

概念的には、

```text
ACC-003
    ↓
現在状態
    ↓
isAvailable

ACC-005
    ↓
設定履歴
    ↓
過去から現在までの
利用可能資産区分
```

と責務を分離する。

---

### 1.3 ACC-006との違い

ACC-005は、利用可能資産設定履歴を参照するAPIである。

利用可能資産区分を変更する場合は、

```text
ACC-006
利用可能資産設定登録
```

を使用する。

概念的には、

```text
ACC-005
    → 履歴参照

ACC-006
    → 新しい設定登録
```

とする。

ACC-005から設定履歴を変更しない。

---

## 2. ユースケース

利用者は、資産口座の利用可能資産区分が過去どのように設定されていたかを確認する際にACC-005を使用する。

主に以下の用途で使用する。

- 現在の利用可能資産設定を確認する
- 過去の利用可能資産区分を確認する
- 設定変更履歴を確認する
- ACC-006実行前に現在の設定状態を確認する
- 資産口座の設定内容を画面上で確認する

概念的な画面利用は、以下とする。

```text
ACC-001
資産口座一覧取得
    ↓
資産口座選択
    ↓
ACC-003
資産口座詳細取得
    ↓
利用可能資産設定履歴を表示
    ↓
ACC-005
利用可能資産設定履歴取得
```

必要に応じて、履歴画面からACC-006による新しい利用可能資産設定登録へ遷移できる。

---

### 2.1 履歴表示

ACC-005では、例えば以下のような履歴を表示できる。

```text
2027-01 ～ 現在
利用可能

2026-07 ～ 2026-12
利用対象外

2026-01 ～ 2026-06
利用可能
```

APIレスポンスでは、表示文言ではなく年月およびbooleanとして返却する。

画面表示用の文言は、React側で変換する。

---

### 2.2 現在設定の確認

最新の履歴について、

```text
endYearMonth = null
```

である場合は、現在継続中の設定として画面表示できる。

ただし、ACC-005自体は現在設定だけを返すAPIではなく、履歴全体を返却する。

---

## 3. エンドポイント

```http
GET /api/v1/asset-accounts/{assetAccountId}/available-settings
```

`assetAccountId`には、利用可能資産設定履歴を取得する資産口座IDを指定する。

例：

```http
GET /api/v1/asset-accounts/10/available-settings
```

---

### 3.1 資産口座配下のリソースとする理由

利用可能資産設定は、単独で存在するリソースではなく、特定の資産口座に紐づく設定履歴である。

そのため、

```text
asset-accounts
    ↓
available-settings
```

という親子関係をURLで表現する。

以下のようなトップレベルURLとはしない。

```http
GET /api/v1/asset-account-available-settings
```

Phase1では、資産口座を起点として利用可能資産設定履歴を取得する。

---

## 4. HTTPメソッド

```text
GET
```

ACC-005は、利用可能資産設定履歴を取得する参照APIであるため、`GET`を使用する。

本APIの実行によって、以下の業務データを更新しない。

- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `assessment_histories`

---

### 4.1 副作用を持たない

ACC-005実行によって、

```text
asset_account_available_settings.updated_at
```

などを変更しない。

また、履歴に不整合が存在した場合でも、GET処理内で自動修復しない。

---

## 5. 利用者コンテキスト

本APIは利用者依存APIのため、`X-User-Id`を必須とする。

操作対象となる利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者コンテキストの特定、`X-User-Id`の検証および利用者境界については、[API共通方針](../api-common-policy.md)に従う。

指定された`assetAccountId`に対応する資産口座が操作対象利用者に帰属する場合のみ、利用可能資産設定履歴を返却する。

概念的には、以下の条件を満たす必要がある。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 5.1 利用者境界

ACC-005では、`assetAccountId`だけを条件として利用可能資産設定履歴を取得してはならない。

まず、

```text
assetAccountId
+
操作対象利用者ID
+
利用中状態
```

によって対象資産口座を確定する。

その資産口座IDを使用して、

```text
asset_account_available_settings.asset_account_id
```

を検索する。

概念的には、

```text
X-User-Id
    ↓
UserContext
    ↓
asset_accounts
    ↓
利用者境界確認
    ↓
asset_account_available_settings
```

とする。

---

### 5.2 他利用者の資産口座

指定された`assetAccountId`が他利用者に帰属する場合は、対象資産口座が存在しないものとして扱う。

概念的には、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

とする。

以下のような専用エラーは返却しない。

```text
FORBIDDEN
OTHER_USER_ASSET_ACCOUNT
```

これにより、他利用者の資産口座が存在するかどうかを外部から推測しにくくする。

---

### 5.3 論理削除済み資産口座

ACC-005では、利用中の資産口座を履歴取得対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は通常取得対象に含めない。

論理削除済み資産口座が指定された場合も、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 5.4 利用可能資産設定にはuser_idを持たせない

```text
asset_account_available_settings
```

には、直接

```text
user_id
```

を保持しない。

利用者境界は、関連する

```text
asset_accounts.user_id
```

を通して保証する。

概念的には、

```text
users
    ↓
asset_accounts.user_id
    ↓
asset_accounts.id
    ↓
asset_account_available_settings.asset_account_id
```

という関係になる。

---

### 5.5 利用可能資産設定IDだけで取得しない

ACC-005は資産口座単位の履歴一覧取得APIである。

そのため、

```text
asset_account_available_settings.id
```

だけを使用して設定を取得する処理は行わない。

必ず、利用者境界確認済みの

```text
assetAccountId
```

を起点として履歴一覧を取得する。

---

### 5.6 他利用者の設定履歴を取得しない

例えば、

```text
User A
    Asset Account A
        Setting A1
        Setting A2

User B
    Asset Account B
        Setting B1
```

という状態で、User AとしてAsset Account Bを指定した場合は、

```text
Setting B1
```

を取得してはならない。

資産口座段階で

```text
ASSET_ACCOUNT_NOT_FOUND
```

として処理を終了する。

---

### 5.7 利用可能資産設定が0件の場合

ACC-002では、資産口座登録時に初期利用可能資産設定を必ず1件登録する。

そのため、通常状態では利用中資産口座に対して設定履歴が1件以上存在することを前提とする。

もし、

```text
asset_account_available_settings
    0件
```

である場合は、単なる空配列として正常扱いせず、業務データ不整合として扱う。

具体的なエラーコードは、後続の「エラーレスポンス」で定義する。

---

### 5.8 履歴重複などの不整合

ACC-005は設定履歴そのものを取得するAPIであるため、期間の重複などが存在する場合も履歴を確認できる。

ただし、明らかな業務データ不整合を正常な履歴として黙って扱うかどうかは、後続の「業務ルール」で定義する。

ACC-005自身が設定履歴を自動修復することはない。

---

### 5.9 X-User-Idが不正な場合

以下の場合は、資産口座検索や利用可能資産設定履歴取得へ進まない。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正
- 指定された利用者が存在しない
- 指定された利用者が論理削除済み

利用者コンテキストに関するエラーコードおよびHTTPステータスは、API共通方針に従う。

---

### 5.10 1リクエスト1利用者

1回のACC-005リクエストでは、`X-User-Id`で指定された1利用者の資産口座だけを扱う。

以下のように、URLやクエリパラメータから別利用者IDを指定する方式は採用しない。

```text
/api/v1/users/{userId}/asset-accounts/{assetAccountId}/available-settings
```

または、

```text
?userId=1
```

利用者の指定経路は、`X-User-Id`へ統一する。

---

# ACC-005 利用可能資産設定履歴取得

## 1. 概要

操作対象となる利用者に帰属する指定された資産口座について、利用可能資産設定の履歴一覧を取得する。

利用可能資産区分は、資産口座の固定属性としてではなく、

```text
asset_account_available_settings
```

によって年月単位の履歴として管理する。

ACC-005では、指定された資産口座に紐づく利用可能資産設定履歴を取得し、過去から現在までの設定変更内容を確認できるようにする。

主に以下の情報を返却する。

- 利用可能資産設定ID
- 適用開始年月
- 適用終了年月
- 利用可能資産区分

概念的には、以下の情報を取得する。

```text
asset_accounts
    ↓
操作対象利用者との
利用者境界確認
    ↓
asset_account_available_settings
    ↓
設定履歴一覧
```

本APIは参照専用であり、資産口座および利用可能資産設定を更新しない。

---

### 1.1 利用可能資産設定とは

利用可能資産設定は、対象資産口座を目的達成判定に利用する資産として扱うかどうかを表す。

例えば、

```text
2026-01 ～ 2026-06
利用可能

2026-07 ～ 2026-12
利用対象外

2027-01 ～
利用可能
```

のように、年月単位で利用可能資産区分が変化する可能性がある。

そのため、現在値だけではなく設定履歴を保持する。

---

### 1.2 ACC-003との違い

ACC-003 資産口座詳細取得では、現在年月に有効な利用可能資産設定だけを取得し、

```text
isAvailable
```

として返却する。

一方、ACC-005では、指定された資産口座に対する利用可能資産設定履歴を一覧として返却する。

概念的には、

```text
ACC-003
    ↓
現在状態
    ↓
isAvailable

ACC-005
    ↓
設定履歴
    ↓
過去から現在までの
利用可能資産区分
```

と責務を分離する。

---

### 1.3 ACC-006との違い

ACC-005は、利用可能資産設定履歴を参照するAPIである。

利用可能資産区分を変更する場合は、

```text
ACC-006
利用可能資産設定登録
```

を使用する。

概念的には、

```text
ACC-005
    → 履歴参照

ACC-006
    → 新しい設定登録
```

とする。

ACC-005から設定履歴を変更しない。

---

## 2. ユースケース

利用者は、資産口座の利用可能資産区分が過去どのように設定されていたかを確認する際にACC-005を使用する。

主に以下の用途で使用する。

- 現在の利用可能資産設定を確認する
- 過去の利用可能資産区分を確認する
- 設定変更履歴を確認する
- ACC-006実行前に現在の設定状態を確認する
- 資産口座の設定内容を画面上で確認する

概念的な画面利用は、以下とする。

```text
ACC-001
資産口座一覧取得
    ↓
資産口座選択
    ↓
ACC-003
資産口座詳細取得
    ↓
利用可能資産設定履歴を表示
    ↓
ACC-005
利用可能資産設定履歴取得
```

必要に応じて、履歴画面からACC-006による新しい利用可能資産設定登録へ遷移できる。

---

### 2.1 履歴表示

ACC-005では、例えば以下のような履歴を表示できる。

```text
2027-01 ～ 現在
利用可能

2026-07 ～ 2026-12
利用対象外

2026-01 ～ 2026-06
利用可能
```

APIレスポンスでは、表示文言ではなく年月およびbooleanとして返却する。

画面表示用の文言は、React側で変換する。

---

### 2.2 現在設定の確認

最新の履歴について、

```text
endYearMonth = null
```

である場合は、現在継続中の設定として画面表示できる。

ただし、ACC-005自体は現在設定だけを返すAPIではなく、履歴全体を返却する。

---

## 3. エンドポイント

```http
GET /api/v1/asset-accounts/{assetAccountId}/available-settings
```

`assetAccountId`には、利用可能資産設定履歴を取得する資産口座IDを指定する。

例：

```http
GET /api/v1/asset-accounts/10/available-settings
```

---

### 3.1 資産口座配下のリソースとする理由

利用可能資産設定は、単独で存在するリソースではなく、特定の資産口座に紐づく設定履歴である。

そのため、

```text
asset-accounts
    ↓
available-settings
```

という親子関係をURLで表現する。

以下のようなトップレベルURLとはしない。

```http
GET /api/v1/asset-account-available-settings
```

Phase1では、資産口座を起点として利用可能資産設定履歴を取得する。

---

## 4. HTTPメソッド

```text
GET
```

ACC-005は、利用可能資産設定履歴を取得する参照APIであるため、`GET`を使用する。

本APIの実行によって、以下の業務データを更新しない。

- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `assessment_histories`

---

### 4.1 副作用を持たない

ACC-005実行によって、

```text
asset_account_available_settings.updated_at
```

などを変更しない。

また、履歴に不整合が存在した場合でも、GET処理内で自動修復しない。

---

## 5. 利用者コンテキスト

本APIは利用者依存APIのため、`X-User-Id`を必須とする。

操作対象となる利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者コンテキストの特定、`X-User-Id`の検証および利用者境界については、[API共通方針](../api-common-policy.md)に従う。

指定された`assetAccountId`に対応する資産口座が操作対象利用者に帰属する場合のみ、利用可能資産設定履歴を返却する。

概念的には、以下の条件を満たす必要がある。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 5.1 利用者境界

ACC-005では、`assetAccountId`だけを条件として利用可能資産設定履歴を取得してはならない。

まず、

```text
assetAccountId
+
操作対象利用者ID
+
利用中状態
```

によって対象資産口座を確定する。

その資産口座IDを使用して、

```text
asset_account_available_settings.asset_account_id
```

を検索する。

概念的には、

```text
X-User-Id
    ↓
UserContext
    ↓
asset_accounts
    ↓
利用者境界確認
    ↓
asset_account_available_settings
```

とする。

---

### 5.2 他利用者の資産口座

指定された`assetAccountId`が他利用者に帰属する場合は、対象資産口座が存在しないものとして扱う。

概念的には、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

とする。

以下のような専用エラーは返却しない。

```text
FORBIDDEN
OTHER_USER_ASSET_ACCOUNT
```

これにより、他利用者の資産口座が存在するかどうかを外部から推測しにくくする。

---

### 5.3 論理削除済み資産口座

ACC-005では、利用中の資産口座を履歴取得対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は通常取得対象に含めない。

論理削除済み資産口座が指定された場合も、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 5.4 利用可能資産設定にはuser_idを持たせない

```text
asset_account_available_settings
```

には、直接

```text
user_id
```

を保持しない。

利用者境界は、関連する

```text
asset_accounts.user_id
```

を通して保証する。

概念的には、

```text
users
    ↓
asset_accounts.user_id
    ↓
asset_accounts.id
    ↓
asset_account_available_settings.asset_account_id
```

という関係になる。

---

### 5.5 利用可能資産設定IDだけで取得しない

ACC-005は資産口座単位の履歴一覧取得APIである。

そのため、

```text
asset_account_available_settings.id
```

だけを使用して設定を取得する処理は行わない。

必ず、利用者境界確認済みの

```text
assetAccountId
```

を起点として履歴一覧を取得する。

---

### 5.6 他利用者の設定履歴を取得しない

例えば、

```text
User A
    Asset Account A
        Setting A1
        Setting A2

User B
    Asset Account B
        Setting B1
```

という状態で、User AとしてAsset Account Bを指定した場合は、

```text
Setting B1
```

を取得してはならない。

資産口座段階で

```text
ASSET_ACCOUNT_NOT_FOUND
```

として処理を終了する。

---

### 5.7 利用可能資産設定が0件の場合

ACC-002では、資産口座登録時に初期利用可能資産設定を必ず1件登録する。

そのため、通常状態では利用中資産口座に対して設定履歴が1件以上存在することを前提とする。

もし、

```text
asset_account_available_settings
    0件
```

である場合は、単なる空配列として正常扱いせず、業務データ不整合として扱う。

具体的なエラーコードは、後続の「エラーレスポンス」で定義する。

---

### 5.8 履歴重複などの不整合

ACC-005は設定履歴そのものを取得するAPIであるため、期間の重複などが存在する場合も履歴を確認できる。

ただし、明らかな業務データ不整合を正常な履歴として黙って扱うかどうかは、後続の「業務ルール」で定義する。

ACC-005自身が設定履歴を自動修復することはない。

---

### 5.9 X-User-Idが不正な場合

以下の場合は、資産口座検索や利用可能資産設定履歴取得へ進まない。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正
- 指定された利用者が存在しない
- 指定された利用者が論理削除済み

利用者コンテキストに関するエラーコードおよびHTTPステータスは、API共通方針に従う。

---

### 5.10 1リクエスト1利用者

1回のACC-005リクエストでは、`X-User-Id`で指定された1利用者の資産口座だけを扱う。

以下のように、URLやクエリパラメータから別利用者IDを指定する方式は採用しない。

```text
/api/v1/users/{userId}/asset-accounts/{assetAccountId}/available-settings
```

または、

```text
?userId=1
```

利用者の指定経路は、`X-User-Id`へ統一する。

---

## 6. パスパラメータ

本APIでは、利用可能資産設定履歴を取得する資産口座を指定するために、以下のパスパラメータを使用する。

| パラメータ | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `assetAccountId` | string | ○ | 利用可能資産設定履歴を取得する資産口座ID |

エンドポイントは、以下とする。

```http
GET /api/v1/asset-accounts/{assetAccountId}/available-settings
```

---

### 6.1 assetAccountId

`assetAccountId`には、利用可能資産設定履歴を取得する資産口座IDを指定する。

例：

```http
GET /api/v1/asset-accounts/10/available-settings
```

APIでは主キーを文字列として扱うが、値としては正の整数形式を前提とする。

正常例：

```text
1
10
123
```

不正例：

```text
0
-1
abc
1.5
```

---

### 6.2 利用者境界

`assetAccountId`だけでは取得対象を確定しない。

実際の資産口座取得では、操作対象利用者IDと組み合わせて検索する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

この条件で取得できた資産口座に対してのみ、利用可能資産設定履歴を取得する。

---

## 7. クエリパラメータ

なし。

ACC-005では、指定された資産口座に紐づく利用可能資産設定履歴をすべて取得する。

そのため、Phase1では絞り込みやページング用のクエリパラメータを使用しない。

以下のようなクエリパラメータは受け付けない。

- `userId`
- `targetYearMonth`
- `isAvailable`
- `includeDeleted`
- `currentOnly`
- `fromYearMonth`
- `toYearMonth`

---

### 7.1 currentOnlyを設けない

現在有効な設定だけを取得する用途は、ACC-003 資産口座詳細取得で扱う。

ACC-005は、利用可能資産設定の履歴一覧取得に責務を限定する。

そのため、

```text
?currentOnly=true
```

のような取得モード切替は設けない。

---

### 7.2 targetYearMonthを設けない

特定年月時点の利用可能資産区分を1件だけ取得するAPIとはしない。

ACC-005では、履歴全体を返却し、必要な表示はフロントエンド側で行う。

---

## 8. リクエストヘッダー

以下のリクエストヘッダーを使用する。

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/asset-accounts/10/available-settings
Accept: application/json
X-User-Id: 1
```

本APIではリクエストボディを使用しないため、

```text
Content-Type: application/json
```

は必須としない。

---

### 8.1 X-User-Id

`X-User-Id`は、API共通方針に従って検証する。

本APIの処理開始前に、共通Middlewareで以下を確認する。

- ヘッダーが指定されていること
- IDが共通方針で定めた形式であること
- 利用者が存在すること
- 利用者が論理削除されていないこと

検証済み利用者IDを利用者コンテキストへ保持し、Action以降で使用する。

---

## 9. リクエストボディ

なし。

ACC-005は参照専用のGET APIであるため、リクエストボディを使用しない。

以下のような情報をリクエストボディから受け付けない。

- `userId`
- `assetAccountId`
- `targetYearMonth`
- `isAvailable`
- `startYearMonth`
- `endYearMonth`

取得対象は、

```text
X-User-Id
+
assetAccountId
```

によって決定する。

---

## 10. リクエスト項目

本APIでは、リクエストボディ項目は存在しない。

クライアントが指定する業務上の入力値は、以下のみとする。

```text
パスパラメータ
    assetAccountId

リクエストヘッダー
    X-User-Id
```

---

## 11. バリデーション

ACC-005では、主に以下を検証する。

```text
X-User-Id
assetAccountId
```

資産口座の存在確認や利用者境界確認、利用可能資産設定履歴の存在確認については、DB取得処理の中で判定する。

---

### 11.1 X-User-Id

`X-User-Id`について、以下を検証する。

- 必須であること
- 共通ID形式に一致すること
- 正の整数として扱えること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

不正な場合は、資産口座検索へ進まない。

---

### 11.2 assetAccountId 必須

`assetAccountId`は、パスパラメータとして必須とする。

ACC-005では、以下の形式を使用する。

```http
GET /api/v1/asset-accounts/{assetAccountId}/available-settings
```

`assetAccountId`を省略したURLはACC-005として成立しない。

---

### 11.3 assetAccountId 形式

`assetAccountId`は、正の整数形式であることを必須とする。

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

形式不正の場合は、資産口座検索へ進まない。

---

### 11.4 assetAccountIdの型変換

Laravel内部では、ルートパラメータを文字列として受け取る場合がある。

形式検証後に、必要に応じて整数へ変換して使用する。

PHPの暗黙的な型変換によって、不正な値を有効なIDとして扱ってはならない。

例えば、

```text
10abc
```

を

```text
10
```

として扱わない。

---

### 11.5 資産口座存在確認

指定された`assetAccountId`について、操作対象利用者に属する利用中の資産口座が存在することを確認する。

概念的な条件は、以下とする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

該当する資産口座が存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 11.6 他利用者の資産口座

指定された`assetAccountId`が他利用者に属する場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下のような専用エラーは返却しない。

```text
FORBIDDEN
OTHER_USER_ASSET_ACCOUNT
```

他利用者の資産口座存在有無を外部へ公開しないためである。

---

### 11.7 論理削除済み資産口座

対象資産口座が

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の場合も、ACC-005の通常取得対象には含めない。

この場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 11.8 利用可能資産設定履歴の存在確認

対象資産口座に紐づく

```text
asset_account_available_settings
```

が1件以上存在することを確認する。

概念的には、

```text
asset_account_available_settings.asset_account_id
    = assetAccountId
```

に該当する履歴を取得する。

ACC-002では初期利用可能資産設定を必ず登録するため、通常は1件以上存在することを前提とする。

---

### 11.9 履歴0件

対象資産口座に対する利用可能資産設定履歴が0件の場合は、正常な空一覧として扱わない。

以下のように、

```json
{
  "data": []
}
```

を返却しない。

ACC-002で初期設定を必ず作成する設計に対するデータ不整合として扱う。

具体的なエラーコードは、後続の「エラーレスポンス」で定義する。

---

### 11.10 start_year_month

取得した履歴の

```text
start_year_month
```

は、各設定の適用開始年月を表す。

ACC-005は参照APIであるため、取得時に値そのものを変更しない。

ただし、DB上で想定形式から外れた値が存在する場合は、業務データ不整合として扱う設計を検討してよい。

通常は、DB制約およびACC-002・ACC-006の登録時バリデーションによって整合性を保証する。

---

### 11.11 end_year_month

```text
end_year_month
```

は、設定の適用終了年月を表す。

現在継続中の設定では、

```text
NULL
```

を許可する。

過去設定では、年月値が設定される。

---

### 11.12 start_year_monthとend_year_monthの大小関係

`end_year_month`が`NULL`ではない場合は、

```text
start_year_month
    <= end_year_month
```

となっていることを正常な履歴状態とする。

例えば、

```text
start_year_month = 2026-08
end_year_month   = 2026-07
```

のような状態は、正常な設定期間ではない。

ACC-005がこの不整合を検出するかどうかは、後続の「業務ルール」で定義する。

---

### 11.13 履歴期間の重複

同一資産口座について、複数の設定期間が重複する状態は、本来の業務ルール上発生しないことを前提とする。

例えば、

```text
設定A
2026-01 ～ 2026-08

設定B
2026-08 ～ NULL
```

のように、同じ年月が複数設定へ含まれる場合は、期間定義によっては重複となる。

ACC-005では、履歴一覧取得時に期間整合性をどこまで検証するかを後続の「業務ルール」で定義する。

---

### 11.14 現在設定が複数件存在する場合

例えば、

```text
設定A
2026-01 ～ NULL

設定B
2026-07 ～ NULL
```

のように、終了年月未設定の履歴が複数存在する状態は正常状態ではない。

ACC-005は履歴一覧取得APIであるため、単純に最新1件だけへ補正しない。

具体的な扱いは、後続の「業務ルール」および「エラーレスポンス」で定義する。

---

### 11.15 is_available

各履歴の

```text
is_available
```

は、booleanとして扱う。

APIでは、

```text
true
false
```

として返却する。

DB内部表現をそのままクライアントへ公開しない。

---

### 11.16 クエリパラメータによる絞り込みを行わない

ACC-005では、以下のような入力値によって履歴を絞り込まない。

```text
targetYearMonth
fromYearMonth
toYearMonth
isAvailable
```

Phase1では、対象資産口座の利用可能資産設定履歴全体を取得する。

---

### 11.17 バリデーション失敗時

以下のいずれかに該当する場合は、正常レスポンスを返却しない。

- `X-User-Id`不正
- `assetAccountId`形式不正
- 利用者不存在
- 資産口座不存在
- 他利用者の資産口座
- 論理削除済み資産口座
- 利用可能資産設定履歴0件
- 後続で定義する業務データ不整合

ACC-005は参照専用APIであるため、異常時にも業務データは更新しない。

---

## 12. 業務ルール

### 12.1 取得対象

ACC-005では、操作対象利用者に帰属する指定された資産口座の利用可能資産設定履歴を取得する。

取得対象は、

```text
asset_account_available_settings
```

に登録されている指定資産口座の全履歴とする。

概念的には、

```text
X-User-Id
    ↓
操作対象利用者
    ↓
assetAccountId
    ↓
asset_accounts
    ↓
利用者境界確認
    ↓
asset_account_available_settings
    ↓
履歴一覧
```

とする。

---

### 12.2 利用中の資産口座のみ対象とする

ACC-005では、

```text
asset_accounts.deleted_at
    IS NULL
```

の資産口座のみを取得対象とする。

論理削除済み資産口座については、利用可能資産設定履歴がDB上に残っていてもACC-005から取得しない。

---

### 12.3 利用者境界

指定された資産口座が操作対象利用者に帰属することを必須とする。

概念的な条件は、以下とする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

条件を満たさない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 12.4 他利用者の履歴を返却しない

利用可能資産設定自体には`user_id`を保持しないため、利用者境界は親となる資産口座を通して保証する。

以下のように、

```text
asset_account_available_settings.asset_account_id
    = assetAccountId
```

だけで履歴を取得してはならない。

必ず、

```text
userId
    ↓
asset_accounts
    ↓
assetAccountId
    ↓
asset_account_available_settings
```

の順序で取得対象を確定する。

---

### 12.5 履歴は全件取得する

ACC-005では、指定された資産口座に紐づく利用可能資産設定履歴を全件取得する。

現在有効な設定だけに限定しない。

例えば、

```text
2026-01 ～ 2026-06
isAvailable = true

2026-07 ～ 2026-12
isAvailable = false

2027-01 ～ NULL
isAvailable = true
```

という履歴が存在する場合は、3件すべてを返却する。

---

### 12.6 現在設定だけを抽出しない

ACC-005は履歴取得APIであるため、

```text
currentYearMonth
```

を基準として現在設定1件だけを抽出する処理は行わない。

現在の`isAvailable`を取得する責務は、ACC-003 資産口座詳細取得が持つ。

---

### 12.7 特定年月だけを抽出しない

ACC-005では、

```text
targetYearMonth
```

を指定して特定年月時点の設定だけを取得する機能を持たない。

Phase1では、資産口座単位の履歴全体を返却する。

---

### 12.8 履歴の並び順

利用可能資産設定履歴は、適用開始年月の降順で返却する。

概念的には、

```text
start_year_month DESC
```

とする。

例えば、

```text
2027-01 ～ NULL
2026-07 ～ 2026-12
2026-01 ～ 2026-06
```

の順で返却する。

これにより、現在または最新の設定を先頭で確認できる。

---

### 12.9 同一開始年月は存在しない前提とする

同一資産口座に対して、

```text
asset_account_id
+
start_year_month
```

は一意であることを前提とする。

そのため、

```text
2026-07 ～ 2026-12
2026-07 ～ NULL
```

のように同一開始年月を持つ設定が複数存在しないようにする。

DB制約については、テーブル定義書の一意性制約に従う。

---

### 12.10 初期設定が必ず存在する

ACC-002 資産口座登録では、資産口座登録と同時に初期利用可能資産設定を1件登録する。

そのため、利用中の資産口座については、

```text
asset_account_available_settings
    >= 1件
```

を正常状態とする。

---

### 12.11 履歴0件はデータ不整合とする

利用中の資産口座に対して利用可能資産設定履歴が0件の場合は、正常な空一覧とはしない。

概念的には、

```text
asset_accounts
    存在
    ↓
asset_account_available_settings
    0件
    ↓
データ不整合
```

とする。

以下のような正常レスポンスにはしない。

```json
{
  "data": []
}
```

ACC-002の登録ルールが満たされていない状態としてエラーを返却する。

---

### 12.12 設定期間は年月単位で管理する

利用可能資産設定の適用期間は、日付単位ではなく年月単位で管理する。

使用する値は、

```text
YYYY-MM
```

形式とする。

例えば、

```text
2026-08
2027-01
```

のように表現する。

---

### 12.13 startYearMonth

`startYearMonth`は、その利用可能資産設定が適用開始される年月を表す。

例えば、

```text
startYearMonth = 2026-08
```

の場合は、2026年8月からその設定が有効であることを表す。

---

### 12.14 endYearMonth

`endYearMonth`は、その利用可能資産設定が適用される最後の年月を表す。

例えば、

```text
startYearMonth = 2026-01
endYearMonth   = 2026-06
```

の場合は、

```text
2026-01
～
2026-06
```

までその設定が有効である。

---

### 12.15 endYearMonthがNULLの場合

`endYearMonth = null`は、終了年月が設定されていない継続中の設定を表す。

例えば、

```text
startYearMonth = 2026-08
endYearMonth   = null
```

は、

```text
2026-08 ～
```

という設定期間を表す。

APIでは、`null`をそのまま返却する。

---

### 12.16 isAvailable

`isAvailable`は、その期間において対象資産口座を利用可能資産として扱うかを表す。

```text
true
    → 利用可能資産として扱う

false
    → 利用可能資産として扱わない
```

APIではbooleanとして返却する。

---

### 12.17 履歴期間は重複しない

同一資産口座について、複数の利用可能資産設定が同じ年月に適用される状態は許可しない。

正常な例：

```text
設定A
2026-01 ～ 2026-06

設定B
2026-07 ～ 2026-12

設定C
2027-01 ～ NULL
```

不正な例：

```text
設定A
2026-01 ～ 2026-08

設定B
2026-08 ～ NULL
```

この場合、2026年8月に2件の設定が適用されるため期間重複となる。

---

### 12.18 期間の連続性を前提とする

利用可能資産設定は、ACC-002による初期登録とACC-006による設定変更によって管理する。

ACC-006で新しい設定を登録する際は、直前の設定を終了させたうえで新しい設定を開始する。

概念的には、

```text
旧設定
2026-01 ～ NULL
    ↓
ACC-006
    ↓
旧設定
2026-01 ～ 2026-07

新設定
2026-08 ～ NULL
```

とする。

そのため、正常な履歴では設定期間が連続していることを前提とする。

---

### 12.19 期間の欠落を正常状態としない

例えば、

```text
設定A
2026-01 ～ 2026-05

設定B
2026-07 ～ NULL
```

のように、

```text
2026-06
```

の設定が存在しない状態は、正常な履歴状態とはしない。

利用可能資産設定は、資産口座の利用開始以降、各年月について一意に判定できることを前提とする。

---

### 12.20 資産口座利用開始年月との整合性

最初の利用可能資産設定は、資産口座の

```text
asset_accounts.start_year_month
```

から開始することを前提とする。

概念的には、

```text
asset_accounts.start_year_month
    =
最初の
asset_account_available_settings.start_year_month
```

とする。

ACC-002で資産口座登録時にこの状態を作成する。

---

### 12.21 利用開始年月より前の設定を許可しない

例えば、

```text
asset_accounts.start_year_month
    = 2026-08

asset_account_available_settings.start_year_month
    = 2026-07
```

のように、資産口座利用開始年月より前から利用可能資産設定が開始されている状態は正常状態としない。

---

### 12.22 最終履歴は終了年月NULLを前提とする

現在利用中の資産口座では、最新の利用可能資産設定は

```text
end_year_month = NULL
```

であることを前提とする。

概念的には、

```text
過去設定
    end_year_month
        = YYYY-MM

最新設定
    end_year_month
        = NULL
```

とする。

---

### 12.23 終了年月NULLの履歴は1件のみ

同一資産口座について、

```text
end_year_month = NULL
```

の履歴は1件だけ存在することを前提とする。

例えば、

```text
設定A
2026-01 ～ NULL

設定B
2026-08 ～ NULL
```

のような状態は正常状態ではない。

---

### 12.24 履歴整合性の検証

ACC-005は、利用可能資産設定履歴を利用者へ表示するAPIである。

そのため、取得した履歴について最低限以下の整合性を確認する。

- 履歴が1件以上存在する
- 最初の開始年月が資産口座利用開始年月と一致する
- `startYearMonth <= endYearMonth`
- 設定期間が重複していない
- 設定期間に欠落がない
- 最新設定の`endYearMonth`が`null`
- `endYearMonth = null`の設定が1件だけ存在する

---

### 12.25 不整合履歴を正常レスポンスとして返さない

履歴整合性に違反している場合は、取得できたレコードをそのまま正常レスポンスとして返却しない。

例えば、

```text
設定A
2026-01 ～ NULL

設定B
2026-08 ～ NULL
```

という状態で、

```json
{
  "data": [
    {
      "startYearMonth": "2026-08",
      "endYearMonth": null,
      "isAvailable": false
    },
    {
      "startYearMonth": "2026-01",
      "endYearMonth": null,
      "isAvailable": true
    }
  ]
}
```

のように正常レスポンスを返却しない。

データ不整合として扱う。

---

### 12.26 不整合を自動修復しない

ACC-005は参照専用APIであるため、履歴不整合を検出してもDBを自動更新しない。

例えば、

```text
最新1件以外の
end_year_monthを自動設定する

欠落期間を
前後の設定で自動補完する

重複設定を削除する
```

などの処理は行わない。

概念的には、

```text
不整合検出
    ↓
エラー
    ↓
ログ記録
```

までとする。

---

### 12.27 ACC-006で履歴整合性を維持する

新しい利用可能資産設定を登録する際の履歴変更責務は、

```text
ACC-006
利用可能資産設定登録
```

が持つ。

ACC-005は、ACC-006によって構築された履歴を参照する。

概念的には、

```text
ACC-002
    ↓
初期設定作成

ACC-006
    ↓
設定履歴追加・期間更新

ACC-005
    ↓
設定履歴参照
```

とする。

---

### 12.28 ACC-003との現在設定判定の整合性

ACC-003では、現在年月に該当する利用可能資産設定を取得して

```text
isAvailable
```

を返却する。

ACC-005の履歴ルールも同じ期間定義を使用する。

つまり、

```text
start_year_month
    <= targetYearMonth

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= targetYearMonth
)
```

を対象年月における設定有効条件とする。

---

### 12.29 利用可能資産区分の意味を変更しない

ACC-005では、DBに保存されている

```text
is_available
```

を別の業務条件によって再計算しない。

例えば、

```text
資産種別
残高
保有商品
目的
```

などから利用可能資産区分を推測しない。

利用可能資産区分は、登録された設定値をそのまま業務状態として扱う。

---

### 12.30 資産口座属性を履歴へ複製しない

ACC-005では、利用可能資産設定履歴として必要な情報だけを返却する。

資産口座の

```text
name
assetType
balanceRecordingUnit
startYearMonth
isEnabled
```

などを各履歴レコードへ重複して含めない。

資産口座自体の情報は、ACC-003で取得する。

---

### 12.31 履歴IDを返却する

各利用可能資産設定を識別できるように、

```text
asset_account_available_settings.id
```

をAPIでは

```text
id
```

として返却する。

ただし、このIDを使用してACC-005から個別設定を更新・削除することはない。

---

### 12.32 履歴を削除しない

ACC-005はGET APIであり、利用可能資産設定履歴を削除しない。

また、Phase1では過去の利用可能資産設定を物理削除するAPIも提供しない。

履歴は、過去時点の利用可能資産状態を再現するために保持する。

---

### 12.33 ページングを行わない

利用可能資産設定は、資産口座単位かつ年月単位で変更される履歴であり、Phase1では大量件数になることを想定しない。

そのため、ACC-005ではページングを行わず全件取得する。

---

## 13. 処理フロー

ACC-005の基本処理フローは、以下とする。

```text
Request
    ↓
X-User-Id検証
    ↓
UserContext取得
    ↓
assetAccountId形式検証
    ↓
資産口座取得
    ↓
利用者境界確認
    ↓
利用可能資産設定履歴取得
    ↓
履歴0件確認
    ↓
履歴整合性確認
    ↓
startYearMonth降順
    ↓
Response DTO生成
    ↓
API Resource
    ↓
200 OK
```

---

### 13.1 資産口座取得

操作対象利用者IDと`assetAccountId`を使用して、対象資産口座を取得する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

取得できない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として処理を終了する。

---

### 13.2 履歴取得

資産口座確認後、以下の条件で利用可能資産設定履歴を取得する。

```text
asset_account_available_settings.asset_account_id
    = assetAccountId
```

並び順は、

```text
start_year_month DESC
```

とする。

---

### 13.3 履歴0件判定

取得結果が0件の場合は、正常な空一覧として返却しない。

概念的には、

```text
履歴件数
    ↓
0件
    ↓
データ不整合エラー
```

とする。

---

### 13.4 履歴整合性確認

取得した履歴について、以下を確認する。

```text
最初の設定開始年月
    =
asset_accounts.start_year_month

各履歴
    startYearMonth <= endYearMonth

期間重複なし

期間欠落なし

endYearMonth = null
    1件のみ

最新設定
    endYearMonth = null
```

不整合がある場合は、正常レスポンスを生成しない。

---

### 13.5 レスポンス生成

正常な履歴を、APIレスポンス用DTOへ変換する。

概念的には、

```text
AssetAccountAvailableSetting
    ↓
AvailableSettingHistory DTO
    ↓
API Resource
    ↓
camelCase
```

とする。

DBカラム名を直接レスポンスへ使用しない。

---

## 14. レスポンス

### 14.1 成功レスポンス

HTTPステータス：

```http
200 OK
```

レスポンス例：

```json
{
  "data": [
    {
      "id": "3",
      "startYearMonth": "2027-01",
      "endYearMonth": null,
      "isAvailable": true
    },
    {
      "id": "2",
      "startYearMonth": "2026-07",
      "endYearMonth": "2026-12",
      "isAvailable": false
    },
    {
      "id": "1",
      "startYearMonth": "2026-01",
      "endYearMonth": "2026-06",
      "isAvailable": true
    }
  ]
}
```

API共通Envelopeに`meta`等を含める場合は、API共通方針に従う。

---

### 14.2 data

`data`には、指定資産口座の利用可能資産設定履歴を配列として返却する。

並び順は、

```text
startYearMonth DESC
```

とする。

正常時には1件以上の要素が存在する。

---

## 15. レスポンス項目

各履歴のレスポンス項目は、以下とする。

| 項目名 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `id` | string | × | 利用可能資産設定ID |
| `startYearMonth` | string | × | 設定の適用開始年月。`YYYY-MM`形式 |
| `endYearMonth` | string | ○ | 設定の適用終了年月。`YYYY-MM`形式。継続中の場合は`null` |
| `isAvailable` | boolean | × | 利用可能資産として扱うか |

---

### 15.1 id

`id`には、

```text
asset_account_available_settings.id
```

を文字列へ変換して返却する。

例：

```json
{
  "id": "3"
}
```

DBの主キー型が`bigint`であっても、APIではIDを文字列として扱う。

---

### 15.2 startYearMonth

`startYearMonth`には、

```text
asset_account_available_settings.start_year_month
```

を返却する。

形式は、

```text
YYYY-MM
```

とする。

例：

```json
{
  "startYearMonth": "2026-08"
}
```

---

### 15.3 endYearMonth

`endYearMonth`には、

```text
asset_account_available_settings.end_year_month
```

を返却する。

終了年月が存在する場合は、

```json
{
  "endYearMonth": "2026-12"
}
```

とする。

継続中の場合は、

```json
{
  "endYearMonth": null
}
```

とする。

空文字には変換しない。

---

### 15.4 isAvailable

`isAvailable`には、

```text
asset_account_available_settings.is_available
```

をbooleanへ変換して返却する。

例：

```json
{
  "isAvailable": true
}
```

または、

```json
{
  "isAvailable": false
}
```

DB内部値を文字列や数値としてそのまま返却しない。

---

### 15.5 レスポンスへ含めない項目

ACC-005では、以下の情報を返却しない。

```text
asset_account_id
created_at
updated_at
```

また、資産口座側の以下の情報も履歴ごとには返却しない。

```text
user_id
name
asset_type
balance_recording_unit
start_year_month
deleted_at
```

資産口座自体の詳細は、ACC-003の責務とする。

---

### 15.6 現在設定フラグを追加しない

Phase1では、

```text
isCurrent
```

のような派生項目を返却しない。

最新設定は、

```text
endYearMonth = null
```

によって判別できる。

APIレスポンス項目を必要以上に増やさない。

---

### 15.7 表示用文言を返却しない

以下のような画面表示専用文言はAPIから返却しない。

```text
利用可能
利用対象外
現在
2026年8月
```

React側で、

```text
isAvailable
startYearMonth
endYearMonth
```

を使用して表示形式へ変換する。

---

## 16. トランザクション境界

ACC-005は参照専用APIであり、業務データを更新しない。

そのため、Phase1では明示的なトランザクションを使用しない。

概念的には、

```text
AssetAccount SELECT
    ↓
AvailableSettings SELECT
    ↓
Response
```

のみで完結する。

---

### 16.1 DB::transactionを使用しない

以下のような

```php
DB::transaction(function () {
    // 資産口座取得
    // 履歴取得
});
```

処理は、Phase1では不要とする。

ACC-005では、

```text
INSERT
UPDATE
DELETE
```

を行わないためである。

---

### 16.2 読み取り整合性

ACC-005実行中にACC-006による利用可能資産設定変更が同時実行された場合、DBのトランザクション分離レベルや各SELECTの実行タイミングによっては、取得状態に差が生じる可能性がある。

Phase1では、履歴参照に対して厳密なスナップショット読み取りを必須としない。

---

### 16.3 不整合検出時も更新しない

履歴取得中にデータ不整合を検出しても、ACC-005内で修復用トランザクションを開始しない。

概念的には、

```text
不整合検出
    ↓
エラー返却
    ↓
内部ログ記録
```

とする。

データ修復は、ACC-005とは別の運用・保守責務として扱う。

---

## 17. 排他制御

ACC-005は、参照専用APIであり、業務データを更新しない。

そのため、Phase1では明示的な排他制御を行わない。

以下のような行ロックは使用しない。

```php
lockForUpdate()
```

また、楽観ロック用の

```text
version
updated_at比較
ETag
If-Match
```

なども使用しない。

---

### 17.1 資産口座へのロック

ACC-005では、`asset_accounts`を利用者境界確認のために参照するだけである。

そのため、対象資産口座行をロックしない。

概念的には、

```text
asset_accounts
    SELECT
```

のみとする。

---

### 17.2 利用可能資産設定へのロック

`asset_account_available_settings`についても、履歴一覧を参照するだけであるため、明示的な行ロックを行わない。

概念的には、

```text
asset_account_available_settings
    SELECT
```

のみとする。

---

### 17.3 ACC-006との同時実行

ACC-005実行中に、別リクエストでACC-006 利用可能資産設定登録が実行される可能性がある。

例えば、

```text
ACC-005
履歴取得開始
    ↓
ACC-006
旧設定終了
+
新設定登録
    ↓
ACC-005
履歴取得完了
```

のような並行実行が発生し得る。

Phase1では、履歴表示用の参照APIとして扱うため、ACC-006の更新処理をブロックしない。

---

### 17.4 厳密な同一時点保証を行わない

ACC-005では、

```text
asset_accounts取得時点
```

と

```text
asset_account_available_settings取得時点
```

を厳密に同一瞬間へ固定することを要件としない。

本APIは履歴の参照・表示を目的とし、月末確定や目的達成判定のような業務確定処理ではないためである。

---

### 17.5 不整合検出時にロックしない

履歴整合性チェックで、

- 履歴0件
- 期間重複
- 期間欠落
- 複数の継続中設定
- 開始年月不整合

などを検出した場合も、ACC-005でロックを取得して修復処理を行わない。

概念的には、

```text
不整合検出
    ↓
エラー返却
    ↓
ログ記録
```

までとする。

---

### 17.6 将来的な整合性要件

将来的に、履歴一覧取得について厳密な同一時点スナップショットが必要になった場合は、

```text
Repeatable Read
明示的トランザクション
```

などを検討する。

Phase1では対象外とする。

---

## 18. エラーレスポンス

異常時は、API共通方針に従った共通エラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID",
    "message": "利用可能資産設定履歴の整合性が不正です。"
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

---

### 18.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

この場合、資産口座検索や履歴取得へ進まない。

---

### 18.2 INVALID_USER_ID

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

例えば、以下を不正とする。

```text
0
-1
abc
1.5
```

---

### 18.3 USER_NOT_FOUND

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

---

### 18.4 INVALID_ASSET_ACCOUNT_ID

`assetAccountId`の形式が不正な場合は、

```text
INVALID_ASSET_ACCOUNT_ID
```

を返却する。

例えば、

```text
0
-1
abc
1.5
1e3
10abc
```

などを不正とする。

形式不正の場合は、資産口座検索へ進まない。

---

### 18.5 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

- 指定された資産口座が存在しない
- 指定された資産口座が他利用者に属する
- 指定された資産口座が論理削除済み

他利用者所属や論理削除済みであることを個別のエラーコードで公開しない。

---

### 18.6 他利用者の資産口座

例えば、

```text
User A
    assetAccountId = 10

User B
    X-User-Id = 2
```

の状態で、User Bが

```http
GET /api/v1/asset-accounts/10/available-settings
```

を実行した場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

User Aの利用可能資産設定履歴を返却しない。

---

### 18.7 論理削除済み資産口座

対象資産口座が

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

論理削除済みであることを専用エラーとして公開しない。

---

### 18.8 履歴0件

対象資産口座に対する

```text
asset_account_available_settings
```

が0件の場合は、正常な空配列として扱わない。

概念的なエラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

とする。

ACC-002で初期設定を必ず登録する設計に対するデータ不整合として扱う。

---

### 18.9 初期設定開始年月不整合

最初の利用可能資産設定の

```text
start_year_month
```

が、資産口座の

```text
asset_accounts.start_year_month
```

と一致しない場合は、履歴不整合として扱う。

概念的には、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

を返却する。

---

### 18.10 startYearMonthとendYearMonthの不整合

以下のように、

```text
start_year_month
    >
end_year_month
```

となっている履歴が存在する場合は、正常レスポンスを返却しない。

例えば、

```text
start_year_month = 2026-08
end_year_month   = 2026-07
```

は不正とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

とする。

---

### 18.11 履歴期間重複

同一資産口座について、設定期間が重複している場合は、履歴不整合として扱う。

例えば、

```text
設定A
2026-01 ～ 2026-08

設定B
2026-08 ～ NULL
```

のように、2026-08が複数設定へ含まれる場合は不正とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

とする。

---

### 18.12 履歴期間欠落

設定期間に年月の欠落が存在する場合も、履歴不整合として扱う。

例えば、

```text
設定A
2026-01 ～ 2026-05

設定B
2026-07 ～ NULL
```

では、

```text
2026-06
```

がどの設定にも属さない。

この場合も、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

とする。

---

### 18.13 継続中設定が複数存在する場合

以下のように、

```text
end_year_month = NULL
```

の設定が複数存在する場合は、正常状態としない。

例：

```text
設定A
2026-01 ～ NULL

設定B
2026-08 ～ NULL
```

この場合も、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

を返却する。

---

### 18.14 最新設定に終了年月が存在する場合

利用中の資産口座では、最新の設定は

```text
end_year_month = NULL
```

であることを前提とする。

例えば、

```text
設定A
2026-01 ～ 2026-06

設定B
2026-07 ～ 2026-12
```

だけが存在し、最新設定にも終了年月が設定されている場合は、現在以降の設定が存在しない。

この場合も、履歴不整合として扱う。

---

### 18.15 同一開始年月の重複

同一資産口座で、

```text
asset_account_id
+
start_year_month
```

が重複している場合も、正常な履歴状態としない。

例えば、

```text
設定A
start_year_month = 2026-08

設定B
start_year_month = 2026-08
```

は不正とする。

DBのUNIQUE制約によって通常は防止するが、取得時に検出された場合は履歴不整合として扱う。

---

### 18.16 不整合を補完しない

履歴不整合時に、以下のような自動補完を行わない。

```text
期間欠落
    ↓
前の設定を延長
```

```text
複数継続設定
    ↓
最新1件を採用
```

```text
最新設定に終了年月あり
    ↓
endYearMonthをNULL扱い
```

データ不整合を正常レスポンスで隠蔽しない。

---

### 18.17 不整合時に一部履歴を返さない

履歴の一部だけが正常であっても、不整合を含む状態で

```text
正常な履歴だけ返却
```

とはしない。

ACC-005は、資産口座の利用可能資産設定履歴全体を1つの整合した履歴として扱う。

不整合がある場合は、正常レスポンスを返却しない。

---

### 18.18 INTERNAL_SERVER_ERROR

履歴取得処理で想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスへ、以下の内部情報を含めない。

- SQL
- PostgreSQL内部エラー
- テーブル名
- カラム名
- 制約名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 18.19 主なエラー一覧

ACC-005で想定する主なエラーは、以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`未指定 |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`形式不正 |
| `400 Bad Request` | `INVALID_ASSET_ACCOUNT_ID` | `assetAccountId`形式不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 利用者不存在、または論理削除済み |
| `404 Not Found` | `ASSET_ACCOUNT_NOT_FOUND` | 資産口座不存在、他利用者所属、または論理削除済み |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND` | 利用可能資産設定履歴が0件 |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID` | 期間重複、期間欠落、複数継続設定などの履歴不整合 |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバー内部エラー |

履歴0件や履歴整合性不正は、クライアント入力によるものではなく、サーバー側の業務データ不整合として扱う。

具体的なデータ不整合エラーコードをAPI全体で共通化する場合は、API共通方針を優先する。

---

### 18.20 エラー時の副作用

ACC-005は参照専用APIであるため、いずれのエラーが発生した場合も業務データを更新しない。

特に、履歴不整合を検出しても、

```text
asset_account_available_settings
```

への

```text
INSERT
UPDATE
DELETE
```

を行わない。

---

## 19. HTTPステータス

ACC-005では、処理結果に応じて以下のHTTPステータスを返却する。

| HTTPステータス | 用途 |
|---|---|
| `200 OK` | 利用可能資産設定履歴取得成功 |
| `400 Bad Request` | 利用者コンテキストまたは`assetAccountId`の形式不正 |
| `404 Not Found` | 利用者または資産口座が存在しない |
| `500 Internal Server Error` | 利用可能資産設定履歴の不整合、または想定外のサーバー内部エラー |

具体的な独自エラーコードは、「エラーレスポンス」およびAPI共通方針に従う。

---

### 19.1 200 OK

指定された資産口座が操作対象利用者に帰属し、利用可能資産設定履歴が正常に取得できた場合は、

```http
200 OK
```

を返却する。

概念的には、

```text
資産口座取得成功
    ↓
利用可能資産設定履歴取得
    ↓
履歴1件以上
    ↓
履歴整合性確認
    ↓
200 OK
```

とする。

---

### 19.2 400 Bad Request

以下の場合は、

```http
400 Bad Request
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
INVALID_ASSET_ACCOUNT_ID
```

例えば、

- `X-User-Id`未指定
- `X-User-Id`形式不正
- `assetAccountId`が正の整数形式ではない

などを対象とする。

---

### 19.3 404 Not Found

以下の場合は、

```http
404 Not Found
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_NOT_FOUND
ASSET_ACCOUNT_NOT_FOUND
```

`ASSET_ACCOUNT_NOT_FOUND`には、以下を含む。

- 指定された資産口座が存在しない
- 指定された資産口座が他利用者に属する
- 指定された資産口座が論理削除済み

他利用者所属や論理削除済みであることを個別のHTTPステータスで公開しない。

---

### 19.4 500 Internal Server Error

以下の場合は、

```http
500 Internal Server Error
```

として扱う。

主なエラーコードは、以下とする。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
INTERNAL_SERVER_ERROR
```

利用可能資産設定履歴の不存在や不整合は、クライアント入力によるものではなく、サーバー側の業務データ不整合として扱う。

---

### 19.5 ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND

対象資産口座について、

```text
asset_account_available_settings
```

が1件も存在しない場合は、

```http
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

とする。

ACC-002で初期利用可能資産設定を必ず作成する設計に対するデータ不整合として扱う。

---

### 19.6 ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID

以下のような履歴整合性違反を検出した場合は、

```http
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

とする。

主な対象は、以下とする。

- 初期設定開始年月が資産口座利用開始年月と一致しない
- `startYearMonth > endYearMonth`
- 設定期間が重複している
- 設定期間に欠落がある
- `endYearMonth = null`の設定が複数存在する
- 最新設定の`endYearMonth`が`null`ではない
- 同一開始年月の設定が複数存在する

---

### 19.7 INTERNAL_SERVER_ERROR

想定外のサーバー内部エラーが発生した場合は、

```http
500 Internal Server Error
```

を返却する。

エラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

レスポンスへSQLやスタックトレースなどの内部情報を含めない。

---

## 20. 冪等性

ACC-005は、参照専用のGET APIであるため、冪等である。

同一の

```text
X-User-Id
+
assetAccountId
```

に対して同じサーバー状態で複数回リクエストした場合、同じ履歴一覧を返却する。

---

### 20.1 業務データを変更しない

ACC-005を何度実行しても、以下の業務データを変更しない。

- `users`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

履歴取得によって、

```text
created_at
updated_at
deleted_at
```

なども変更しない。

---

### 20.2 同一リクエストの再実行

例えば、

```http
GET /api/v1/asset-accounts/10/available-settings
Accept: application/json
X-User-Id: 1
```

を、サーバー側の状態が変化していない間に複数回実行した場合は、同じ履歴内容を返却する。

---

### 20.3 並び順も同一とする

ACC-005では、履歴を

```text
startYearMonth DESC
```

で返却する。

そのため、サーバー状態が変化していない場合は、再実行時も同じ並び順となる。

---

### 20.4 サーバー状態が変更された場合

ACC-005自体は冪等であるが、ACC-006によって利用可能資産設定履歴が変更された場合は、後続のACC-005で返却内容が変化する。

例えば、

```text
ACC-005
    ↓
履歴2件

ACC-006
    ↓
旧設定終了
+
新設定登録

ACC-005
    ↓
履歴3件
```

となり得る。

これは、ACC-005の冪等性を損なうものではない。

---

### 20.5 資産口座無効化後

ACC-005取得後に資産口座がACC系の無効化APIによって論理削除された場合は、同じ`assetAccountId`で再度ACC-005を実行すると、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となる。

これも、サーバー状態が変更された結果である。

---

### 20.6 Idempotency-Key

ACC-005はGETによる参照APIであるため、

```text
Idempotency-Key
```

を使用しない。

冪等性確保のための追加ヘッダーやサーバー側キー管理は不要とする。

---

## 21. キャッシュ

Phase1では、ACC-005専用のサーバー側アプリケーションキャッシュを使用しない。

利用可能資産設定履歴は、PostgreSQLの最新状態から取得する。

---

### 21.1 サーバー側キャッシュ

Phase1では、以下のようなACC-005専用キャッシュを導入しない。

```text
Redis
Laravel Cache
Application Memory Cache
```

利用可能資産設定履歴は資産口座単位で比較的少数件となることを想定し、Phase1ではキャッシュ導入による複雑性を追加しない。

---

### 21.2 ACC-006による履歴変更

ACC-006が成功すると、

```text
asset_account_available_settings
```

の履歴内容が変化する。

そのため、フロントエンドでACC-005の結果をQuery Cacheへ保持している場合は、ACC-006成功後に対象資産口座の履歴Queryを無効化する。

概念的には、

```text
ACC-006成功
    ↓
ACC-005 Query invalidate
    ↓
ACC-005再取得
    ↓
最新履歴表示
```

とする。

---

### 21.3 React・TypeScript側のQuery Cache

フロントエンドでは、TanStack QueryなどのQuery Cacheを使用してよい。

概念的には、以下のようなQuery Keyを使用する。

```text
assetAccounts
+
assetAccountId
+
availableSettings
```

例えば、

```typescript
[
  'assetAccounts',
  assetAccountId,
  'availableSettings',
]
```

のようなキーで履歴取得結果を保持してよい。

具体的な実装は、「React・TypeScriptでの利用」で定義する。

---

### 21.4 Query Keyの共通化

Query Keyは、各コンポーネントへ直接記述せず、共通定義として管理してよい。

概念例：

```typescript
export const assetAccountKeys = {
  all: [
    'assetAccounts',
  ] as const,

  detail: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
    ] as const,

  availableSettings: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
      'availableSettings',
    ] as const,
};
```

---

### 21.5 ACC-006成功後のキャッシュ無効化

ACC-006成功後は、ACC-005だけでなくACC-003の

```text
isAvailable
```

も変化する可能性がある。

そのため、少なくとも以下を無効化する。

```text
ACC-005
利用可能資産設定履歴

ACC-003
資産口座詳細
```

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.availableSettings(
      assetAccountId,
    ),
});

await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

---

### 21.6 資産口座無効化後

対象資産口座が無効化された場合は、ACC-005の通常取得対象から外れる。

そのため、フロントエンドでは対象資産口座の利用可能資産設定履歴Queryを無効化または削除する。

概念例：

```typescript
queryClient.removeQueries({
  queryKey:
    assetAccountKeys.availableSettings(
      assetAccountId,
    ),
});
```

---

### 21.7 ACC-004成功後

ACC-004では、

```text
asset_account_available_settings
```

を更新しない。

そのため、利用可能資産設定履歴そのものはACC-004によって変化しない。

したがって、ACC-004成功だけを理由としてACC-005の履歴Queryを必ず無効化する必要はない。

ただし、画面上で資産口座名などと履歴を一体表示している場合は、画面構成に応じて関連Queryを再取得してよい。

---

### 21.8 ACC-003とのキャッシュ分離

ACC-003とACC-005は、同じ資産口座を対象とするが、取得内容は異なる。

```text
ACC-003
    → 現在の資産口座詳細

ACC-005
    → 利用可能資産設定履歴
```

そのため、Query Cacheも同一キーへまとめず、別キーとして管理する。

---

### 21.9 履歴全体をキャッシュする

ACC-005はページングを行わず、履歴全体を返却する。

そのため、フロントエンドでは取得した配列全体を1つのQuery Cacheとして保持してよい。

個々の履歴レコードごとに別Query Cacheを作成する必要はない。

---

### 21.10 履歴順序をフロントエンドで再構築しない

ACC-005では、サーバー側で

```text
startYearMonth DESC
```

として返却する。

フロントエンドでは、原則としてこの並び順をそのまま使用する。

画面都合で並び替える場合を除き、APIレスポンスを毎回独自ソートして正式な順序を再定義しない。

---

### 21.11 Optimistic Update

ACC-005自体は参照APIであるため、Optimistic Updateの対象ではない。

ACC-006成功時にACC-005のQuery Cacheを直接書き換えることも技術的には可能だが、Phase1では

```text
ACC-006成功
    ↓
ACC-005 invalidate
    ↓
履歴再取得
```

という単純な方式を基本とする。

---

### 21.12 サーバー状態を正とする

利用可能資産設定履歴の最終状態は、PostgreSQLに保存されたデータを正とする。

React側でACC-006のRequest内容だけから履歴全体を再構築しない。

例えば、

```text
旧設定のendYearMonth
新設定のstartYearMonth
新規設定ID
```

などは、サーバー側の確定状態を再取得して表示する。

---

### 21.13 HTTPキャッシュ

ACC-005はGET APIであるため、技術的にはHTTPキャッシュの対象にできる。

ただし、Phase1ではACC-005独自の

```text
ETag
Last-Modified
If-None-Match
If-Modified-Since
```

などは導入しない。

HTTPキャッシュに関する共通方針がある場合は、API共通方針を優先する。

---

### 21.14 利用者境界とキャッシュキー

フロントエンドまたは将来のサーバーキャッシュでACC-005をキャッシュする場合は、`assetAccountId`だけで異なる利用者間のキャッシュを共有しない。

本APIは利用者依存APIであるため、概念的には

```text
userId
+
assetAccountId
+
availableSettings
```

によって利用者境界を維持する必要がある。

---

### 21.15 利用者切替時

利用者切替が行われた場合は、前利用者の利用可能資産設定履歴を新しい利用者へ表示し続けない。

以下のいずれかで対応する。

```text
Query KeyへuserIdを含める

または

利用者切替時に
関連Query Cacheを無効化する
```

API側の`X-User-Id`による利用者境界だけでなく、画面キャッシュ上でも利用者境界を維持する。

---

### 21.16 長時間キャッシュ

利用可能資産設定履歴は、ACC-006が実行されない限り頻繁には変化しない。

ただし、Phase1では長時間キャッシュを前提とした複雑な失効設計は導入しない。

TanStack Queryの`staleTime`などは、フロントエンド共通方針に従う。

---

## 22. 関連テーブル

ACC-005では、指定された資産口座に紐づく利用可能資産設定履歴を取得するため、以下のテーブルを使用する。

| テーブル | 用途 | 更新 |
|---|---|:---:|
| `users` | 操作対象利用者の確認 | × |
| `asset_accounts` | 対象資産口座の取得、利用者境界確認 | × |
| `asset_account_available_settings` | 利用可能資産設定履歴の取得 | × |

ACC-005は参照専用APIであるため、いずれのテーブルも更新しない。

---

### 22.1 users

`X-User-Id`で指定された操作対象利用者の存在確認に使用する。

概念的な条件は、以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者コンテキストの確認は、API共通Middlewareで行う。

ACC-005では、`users`を更新しない。

---

### 22.2 asset_accounts

指定された`assetAccountId`に対応する資産口座を取得し、利用者境界を確認するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | `assetAccountId`との照合 |
| `user_id` | 操作対象利用者との利用者境界確認 |
| `start_year_month` | 初期利用可能資産設定との整合性確認 |
| `deleted_at` | 利用状態の判定 |

概念的な取得条件は、以下とする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 22.3 他利用者の資産口座

`asset_accounts.id`が一致していても、

```text
asset_accounts.user_id
    != 操作対象利用者ID
```

の場合は、取得対象に含めない。

他利用者の資産口座を一度取得してからアプリケーション側で利用者境界を判定する方式を基本としない。

---

### 22.4 論理削除済み資産口座

ACC-005では、利用中の資産口座だけを履歴取得対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は取得しない。

LaravelのSoftDeletesを使用している場合は、通常スコープを使用し、

```php
withTrashed()
```

を使用しない。

---

### 22.5 asset_account_available_settings

指定された資産口座の利用可能資産設定履歴を取得するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 利用可能資産設定ID |
| `asset_account_id` | 対象資産口座との関連 |
| `start_year_month` | 設定の適用開始年月 |
| `end_year_month` | 設定の適用終了年月 |
| `is_available` | 利用可能資産区分 |

---

### 22.6 履歴取得条件

対象資産口座に紐づく利用可能資産設定履歴を、以下の条件で取得する。

```text
asset_account_available_settings.asset_account_id
    = assetAccountId
```

現在年月による絞り込みは行わない。

ACC-005では、対象資産口座の履歴全体を取得する。

---

### 22.7 履歴の並び順

取得した履歴は、

```text
start_year_month DESC
```

で並べる。

概念的には、

```text
2027-01 ～ NULL
2026-07 ～ 2026-12
2026-01 ～ 2026-06
```

の順とする。

---

### 22.8 id

```text
asset_account_available_settings.id
```

は、APIではstringへ変換して返却する。

概念的には、

```text
bigint
    ↓
string
```

とする。

---

### 22.9 start_year_month

```text
asset_account_available_settings.start_year_month
```

は、

```text
YYYY-MM
```

形式の

```text
startYearMonth
```

として返却する。

---

### 22.10 end_year_month

```text
asset_account_available_settings.end_year_month
```

は、

```text
YYYY-MM
```

または

```text
NULL
```

として保持する。

APIでは、

```text
endYearMonth
```

として返却する。

継続中設定の場合は、

```json
{
  "endYearMonth": null
}
```

とする。

---

### 22.11 is_available

```text
asset_account_available_settings.is_available
```

は、

```text
isAvailable
```

として返却する。

APIではbooleanとして扱う。

```text
true
false
```

以外の表現へ変換しない。

---

### 22.12 初期設定との整合性

最古の利用可能資産設定の

```text
start_year_month
```

は、

```text
asset_accounts.start_year_month
```

と一致することを正常状態とする。

概念的には、

```text
asset_accounts.start_year_month
    =
MIN(
    asset_account_available_settings.start_year_month
)
```

となる。

---

### 22.13 履歴期間の整合性

同一資産口座に紐づく利用可能資産設定履歴について、以下を正常状態とする。

- 各履歴で`start_year_month <= end_year_month`
- 設定期間が重複しない
- 設定期間に欠落がない
- 最新設定の`end_year_month`が`NULL`
- `end_year_month = NULL`の設定が1件のみ
- 同一`start_year_month`が複数存在しない

---

### 22.14 更新しない関連テーブル

ACC-005では、以下のテーブルも更新しない。

- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

また、これらのデータをACC-005のレスポンスへ含めない。

---

## 23. 関連する機能要件

ACC-005は、資産口座の利用可能資産設定履歴確認に関する機能要件と対応する。

主な関連要件は、以下とする。

- 資産口座管理
  - 資産口座ごとに利用可能資産区分を管理できる
  - 利用可能資産区分を年月単位の履歴として管理できる
  - 過去の利用可能資産区分を確認できる
- 利用可能資産設定
  - 資産口座登録時に初期設定を作成する
  - 利用可能資産区分の変更履歴を保持する
  - 設定期間を重複させない
  - 設定期間に欠落を作らない
  - 現在継続中の設定を1件だけ保持する
- 利用者境界
  - 操作対象利用者に帰属する資産口座の履歴だけを取得できる
  - 他利用者に属する資産口座の履歴は取得できない
  - 他利用者の資産口座は対象不存在として扱う
- 論理削除
  - 論理削除済み資産口座は通常の履歴取得対象としない

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

## 24. インデックス

ACC-005では、主に以下の検索条件を使用する。

```text
asset_accounts.id
```

```text
asset_accounts.user_id
```

```text
asset_account_available_settings.asset_account_id
```

また、履歴の並び順として

```text
asset_account_available_settings.start_year_month
```

を使用する。

---

### 24.1 asset_accounts.id

`asset_accounts.id`は主キーであるため、主キーインデックスを使用する。

ACC-005専用として追加インデックスを作成しない。

---

### 24.2 asset_accounts.user_id

ACC-005では、

```text
id
+
user_id
```

を条件として対象資産口座を取得する。

`id`は主キーであるため、Phase1ではACC-005専用に

```text
(id, user_id)
```

の複合インデックスを追加する必要性は低い。

実際の実行計画を踏まえて判断する。

---

### 24.3 asset_account_available_settings.asset_account_id

ACC-005では、

```text
asset_account_id
```

を条件として履歴全体を取得する。

FK列としてインデックスを設定する、または既存インデックスを利用する。

---

### 24.4 asset_account_id + start_year_month

履歴取得では、

```text
asset_account_id
```

で絞り込み、

```text
start_year_month DESC
```

で並び替える。

そのため、必要に応じて

```text
asset_account_id
+
start_year_month
```

の複合インデックスを検討する。

また、

```text
asset_account_id
+
start_year_month
```

をUNIQUE制約としている場合は、そのインデックスを履歴取得にも利用できる。

---

### 24.5 不要な重複インデックスを作成しない

ACC-005のためだけに、既存の

- 主キー
- FK用インデックス
- UNIQUE制約

と役割が重複するインデックスを追加しない。

実際のSQL、データ件数、実行計画を確認したうえで追加を判断する。

---

## 25. 性能

ACC-005は、単一資産口座に紐づく利用可能資産設定履歴を取得するAPIである。

利用可能資産設定は年月単位で変更されるため、Phase1では1資産口座あたりの件数は比較的少ないことを想定する。

---

### 25.1 資産口座一覧を取得しない

以下のように、操作対象利用者の資産口座一覧を取得してから対象資産口座を探す実装は行わない。

```text
資産口座一覧取得
    ↓
PHP側でassetAccountId検索
```

対象IDと利用者IDを条件として直接1件取得する。

---

### 25.2 履歴を資産口座IDで直接絞り込む

利用可能資産設定について、全資産口座分を取得してからPHP側で絞り込まない。

以下の条件をDBへ指定する。

```text
asset_account_id
    = assetAccountId
```

---

### 25.3 ページングを行わない

利用可能資産設定履歴は、年月単位で変更される比較的少数件のデータである。

Phase1ではページングを導入せず、対象資産口座の履歴を全件取得する。

---

### 25.4 不要な関連データを取得しない

ACC-005では、以下を取得しない。

- 保有商品一覧
- 月末資産残高
- 商品別月末評価額
- 月末資産状況
- 手取り収入
- 目的
- 目的達成判定履歴

必要な

```text
asset_accounts
+
asset_account_available_settings
```

だけを参照する。

---

### 25.5 N+1問題

ACC-005は単一資産口座を対象とし、履歴一覧も単一Queryで取得できるため、通常の意味でのN+1問題は発生しない。

各履歴について追加Queryを実行しない。

---

### 25.6 履歴整合性確認

履歴整合性確認は、取得した履歴配列に対してアプリケーション側で1回走査する程度とする。

概念的には、

```text
履歴取得
    ↓
開始年月順に整列
    ↓
前後レコードを比較
```

とし、各履歴ごとに追加SQLを実行しない。

---

## 26. セキュリティ

ACC-005では、利用者境界を最重要のセキュリティ要件とする。

---

### 26.1 assetAccountIdだけで履歴を取得しない

以下のような処理は基本としない。

```php
AssetAccountAvailableSetting::query()
    ->where(
        'asset_account_id',
        $assetAccountId,
    )
    ->get();
```

その前に、対象資産口座が

```text
userId
+
assetAccountId
+
deleted_at IS NULL
```

を満たすことを確認する。

---

### 26.2 他利用者の履歴を公開しない

他利用者に属する`assetAccountId`が指定された場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下のような情報を返却しない。

```text
この資産口座は
別利用者に属しています
```

---

### 26.3 論理削除状態を公開しない

論理削除済み資産口座も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下のような状態判別可能な専用エラーを返却しない。

```text
ASSET_ACCOUNT_DISABLED
ASSET_ACCOUNT_DELETED
```

---

### 26.4 user_idを返却しない

ACC-005のレスポンスには、

```text
user_id
```

を含めない。

利用者は`X-User-Id`によって既に確定しており、履歴レコードごとに所有者情報を返す必要はない。

---

### 26.5 asset_account_idを返却しない

ACC-005のURLで対象資産口座は既に指定されている。

そのため、各履歴に

```text
assetAccountId
```

や

```text
asset_account_id
```

を重複して返却しない。

---

### 26.6 内部日時を返却しない

以下のDB管理項目は返却しない。

```text
created_at
updated_at
```

利用可能資産設定の業務上必要な期間は、

```text
startYearMonth
endYearMonth
```

で表現する。

---

### 26.7 SQLインジェクション対策

`assetAccountId`や利用者IDを検索条件に使用する場合は、EloquentまたはQuery Builderのバインド機構を使用する。

SQL文字列へ入力値を直接連結しない。

---

### 26.8 エラー情報

異常時に、以下をレスポンスへ含めない。

- SQL
- PostgreSQL内部エラー
- テーブル名
- カラム名
- 制約名
- Laravel内部例外
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

## 27. ログ・監視

ACC-005では、API共通ログ方針に従って処理結果を記録する。

参照APIであるため、取得した履歴内容そのものを通常ログへ不要に出力しない。

---

### 27.1 ログコンテキスト

必要に応じて、以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assetAccountId
httpStatus
errorCode
```

`apiId`は、

```text
ACC-005
```

とする。

---

### 27.2 正常時

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = ACC-005
assetAccountId
historyCount
httpStatus = 200
```

履歴件数は障害調査上有用な場合に記録してよい。

---

### 27.3 ログへ履歴内容を出力しない

通常ログへ、以下の履歴内容を不要に出力しない。

```text
startYearMonth
endYearMonth
isAvailable
```

必要な識別情報と処理結果だけを記録する。

---

### 27.4 異常時

異常時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId
assetAccountId
errorCode
httpStatus
```

---

### 27.5 履歴0件

以下のエラーでは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

必要に応じて、

```text
assetAccountId
historyCount = 0
```

を内部ログへ記録する。

---

### 27.6 履歴不整合

以下のエラーでは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

原因調査に必要な範囲で、内部ログへ不整合種別を記録してよい。

例えば、

```text
INITIAL_START_MISMATCH
INVALID_PERIOD
PERIOD_OVERLAP
PERIOD_GAP
MULTIPLE_OPEN_PERIODS
LATEST_PERIOD_CLOSED
DUPLICATE_START_YEAR_MONTH
```

などを内部的な診断情報として使用してよい。

これらをAPIレスポンスの独自エラーコードとして細分化する必要はない。

---

### 27.7 requestId

クライアントへ返却する`requestId`とサーバーログを関連付けられるようにする。

概念的には、

```text
requestId
    ↓
ACC-005ログ
    ↓
履歴不整合原因調査
```

が可能な状態とする。

---

## 28. テスト観点

ACC-005では、正常な履歴一覧取得だけでなく、

- 利用者境界
- 論理削除
- 並び順
- 履歴0件
- 初期設定開始年月
- 期間整合性
- 期間重複
- 期間欠落
- 継続中設定
- レスポンス契約
- 副作用なし

を重点的に確認する。

---

### 28.1 正常系：履歴1件

以下の状態を用意する。

```text
asset_accounts.start_year_month
    = 2026-08

利用可能資産設定
2026-08 ～ NULL
is_available = true
```

ACC-005を実行し、以下を確認する。

- `200 OK`
- `data`が1件
- `startYearMonth = "2026-08"`
- `endYearMonth = null`
- `isAvailable = true`

---

### 28.2 正常系：複数履歴

以下の履歴を用意する。

```text
2026-01 ～ 2026-06
true

2026-07 ～ 2026-12
false

2027-01 ～ NULL
true
```

以下を確認する。

- `200 OK`
- 3件取得できる
- 全履歴が返却される
- 最新設定だけに限定されない

---

### 28.3 並び順

複数履歴が存在する場合、レスポンスが

```text
startYearMonth DESC
```

となることを確認する。

期待順：

```text
2027-01
2026-07
2026-01
```

---

### 28.4 idの型

DB上のIDが`bigint`であっても、レスポンスでは

```json
{
  "id": "3"
}
```

のようにstringとなることを確認する。

---

### 28.5 endYearMonth = null

最新の継続中設定について、

```json
{
  "endYearMonth": null
}
```

となることを確認する。

以下の値に変換されないこと。

```text
""
"0000-00"
"9999-12"
```

---

### 28.6 isAvailable = true

```text
is_available = true
```

の場合、

```json
{
  "isAvailable": true
}
```

となることを確認する。

---

### 28.7 isAvailable = false

```text
is_available = false
```

の場合、

```json
{
  "isAvailable": false
}
```

となることを確認する。

`false`を未設定扱いしない。

---

### 28.8 X-User-Id未指定

`X-User-Id`を指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

となること。

---

### 28.9 X-User-Id形式不正

例えば、

```http
X-User-Id: abc
```

を指定する。

期待結果：

```text
400 Bad Request
INVALID_USER_ID
```

となること。

---

### 28.10 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 28.11 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 28.12 assetAccountId形式不正

以下をそれぞれ確認する。

```text
0
-1
abc
1.5
1e3
10abc
```

期待結果：

```text
400 Bad Request
INVALID_ASSET_ACCOUNT_ID
```

となること。

---

### 28.13 資産口座不存在

存在しない`assetAccountId`を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 28.14 他利用者の資産口座

以下の状態を用意する。

```text
User A
    Asset Account A
        Setting A

User B
    Asset Account B
        Setting B
```

User AとしてAsset Account Bを指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

Setting Bがレスポンスへ含まれないこと。

---

### 28.15 論理削除済み資産口座

対象資産口座を

```text
deleted_at IS NOT NULL
```

とする。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

DBに利用可能資産設定履歴が残っていても取得できないこと。

---

### 28.16 履歴0件

利用中の資産口座を用意し、利用可能資産設定履歴を0件とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

となること。

以下も確認する。

- `data: []`を返さない
- 設定を自動登録しない
- 業務データを変更しない

---

### 28.17 初期開始年月一致

以下の状態を用意する。

```text
asset_accounts.start_year_month
    = 2026-01

最古設定
start_year_month
    = 2026-01
```

正常に取得できることを確認する。

---

### 28.18 初期開始年月不一致

以下の状態を用意する。

```text
asset_accounts.start_year_month
    = 2026-01

最古設定
start_year_month
    = 2026-02
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 28.19 利用開始前の設定

以下の状態を用意する。

```text
asset_accounts.start_year_month
    = 2026-08

最古設定
start_year_month
    = 2026-07
```

履歴不整合となることを確認する。

---

### 28.20 startYearMonth = endYearMonth

以下の設定を用意する。

```text
start_year_month = 2026-08
end_year_month   = 2026-08
```

1か月だけ有効な設定として正常に扱われることを確認する。

---

### 28.21 startYearMonth > endYearMonth

以下を用意する。

```text
start_year_month = 2026-08
end_year_month   = 2026-07
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 28.22 正常な連続期間

以下の履歴を用意する。

```text
設定A
2026-01 ～ 2026-06

設定B
2026-07 ～ 2026-12

設定C
2027-01 ～ NULL
```

正常に取得できることを確認する。

---

### 28.23 期間重複

以下を用意する。

```text
設定A
2026-01 ～ 2026-08

設定B
2026-08 ～ NULL
```

2026-08が両設定へ含まれるため、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 28.24 期間欠落

以下を用意する。

```text
設定A
2026-01 ～ 2026-05

設定B
2026-07 ～ NULL
```

2026-06が欠落しているため、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 28.25 1か月単位の連続性

以下を用意する。

```text
設定A
2026-01 ～ 2026-01

設定B
2026-02 ～ NULL
```

正常な連続履歴として扱われることを確認する。

---

### 28.26 継続中設定1件

以下のように、

```text
end_year_month = NULL
```

の設定が1件だけ存在する場合は、正常とする。

---

### 28.27 継続中設定が複数件

以下を用意する。

```text
設定A
2026-01 ～ NULL

設定B
2026-08 ～ NULL
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 28.28 継続中設定が0件

利用中資産口座について、すべての設定に`end_year_month`が存在する状態を用意する。

例えば、

```text
設定A
2026-01 ～ 2026-06

設定B
2026-07 ～ 2026-12
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 28.29 最新設定がNULL

開始年月が最も新しい設定の

```text
end_year_month
```

が`NULL`であることを正常条件として確認する。

---

### 28.30 同一startYearMonth

可能であればDB制約を迂回したテストデータ等で、

```text
同一asset_account_id
+
同一start_year_month
```

の設定を複数用意する。

履歴不整合として正常レスポンスを返さないことを確認する。

通常の登録経路では、DBのUNIQUE制約によって防止されることも確認する。

---

### 28.31 現在年月に依存しない

ACC-005は履歴全体取得APIであるため、サーバーの現在年月を変えても同じDB状態であれば同じ履歴一覧が返却されることを確認する。

ACC-003のような現在年月フィルタを行っていないことを確認する。

---

### 28.32 過去設定も返却する

終了済みの

```text
end_year_month IS NOT NULL
```

の設定もレスポンスへ含まれることを確認する。

---

### 28.33 資産口座情報を返却しない

レスポンスへ、以下が含まれないことを確認する。

- `assetAccountId`
- `assetAccountName`
- `assetType`
- `balanceRecordingUnit`
- `assetAccountStartYearMonth`
- `isEnabled`

資産口座詳細はACC-003の責務とする。

---

### 28.34 DB内部項目を返却しない

レスポンスへ、以下が含まれないことを確認する。

- `asset_account_id`
- `created_at`
- `updated_at`
- `user_id`
- `deleted_at`

---

### 28.35 isCurrentを返却しない

Phase1では、各履歴へ

```text
isCurrent
```

が追加されていないことを確認する。

現在設定は、

```text
endYearMonth = null
```

から判断できる。

---

### 28.36 正常レスポンス契約

正常時に、概念的に以下の形式となることを確認する。

```json
{
  "data": [
    {
      "id": "3",
      "startYearMonth": "2027-01",
      "endYearMonth": null,
      "isAvailable": true
    },
    {
      "id": "2",
      "startYearMonth": "2026-07",
      "endYearMonth": "2026-12",
      "isAvailable": false
    },
    {
      "id": "1",
      "startYearMonth": "2026-01",
      "endYearMonth": "2026-06",
      "isAvailable": true
    }
  ]
}
```

以下を確認する。

- HTTPステータスが`200 OK`
- `data`が配列
- 正常時は1件以上
- `id`がstring
- `startYearMonth`が`YYYY-MM`
- `endYearMonth`がstringまたは`null`
- `isAvailable`がboolean
- `startYearMonth DESC`である

---

### 28.37 副作用なし

ACC-005実行前後で、以下のテーブルに変更がないことを確認する。

- `users`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

特に、

```text
created_at
updated_at
deleted_at
```

が履歴取得によって変更されないことを確認する。

---

### 28.38 同一リクエストの再実行

サーバー状態を変更せずに、同じ

```text
X-User-Id
+
assetAccountId
```

でACC-005を複数回実行する。

同じ履歴内容、同じ並び順となることを確認する。

---

### 28.39 ACC-006後の再取得

最初にACC-005で履歴を取得する。

その後、ACC-006によって新しい利用可能資産設定を登録する。

再度ACC-005を実行し、

- 新しい履歴が追加されていること
- 旧設定の`endYearMonth`が更新されていること
- 新設定が先頭に表示されること
- 履歴全体の連続性が維持されていること

を確認する。

---

### 28.40 資産口座無効化後

ACC-005で正常取得できる資産口座を無効化する。

その後、同じ`assetAccountId`でACC-005を実行する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

履歴データ自体がDBに残っていても取得できないことを確認する。

---

### 28.41 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
INTERNAL_SERVER_ERROR
```

以下も確認する。

- `error.code`が設定されること
- `error.message`が設定されること
- `requestId`が設定されること
- SQLが含まれないこと
- PostgreSQL内部エラーが含まれないこと
- 制約名が含まれないこと
- スタックトレースが含まれないこと
- サーバーファイルパスが含まれないこと

---

### 28.42 履歴不整合ログ

以下のエラーを発生させた場合に、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

調査可能なログが記録されることを確認する。

必要に応じて、

```text
requestId
userId
assetAccountId
historyCount
invalidReason
```

などから原因を追跡できることを確認する。

---

### 28.43 INTERNAL_SERVER_ERROR

履歴取得処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

- 業務データが更新されないこと
- 内部情報がレスポンスへ公開されないこと
- サーバーログに調査情報が記録されること
- レスポンスの`requestId`からログを追跡できること

---

## 29. Laravel実装方針

ACC-005では、Action、UseCase、Query、Validator、DTO、API Resource、Responderを分離して実装する。

概念的な構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ AssetAccountQuery
    ├─ AssetAccountAvailableSettingQuery
    └─ AvailableSettingHistoryValidator
    ↓
History Result DTO
    ↓
API Resource Collection
    ↓
Responder
```

ACC-005は参照専用APIであるため、Repositoryは使用しない。

また、リクエストボディを使用しないため、ACC-005専用のFormRequestは作成しない。

---

### 29.1 Route

ACC-005は、以下のルートとして定義する。

概念例：

```php
Route::get(
    '/api/v1/asset-accounts/{assetAccountId}/available-settings',
    ListAssetAccountAvailableSettingsAction::class,
);
```

同じ資産口座配下であっても、ACC-003とはURLによって責務を分離する。

```text
GET
/api/v1/asset-accounts/{assetAccountId}
    → ACC-003 資産口座詳細取得

GET
/api/v1/asset-accounts/{assetAccountId}/available-settings
    → ACC-005 利用可能資産設定履歴取得
```

---

### 29.2 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、

```text
X-User-Id取得
    ↓
必須チェック
    ↓
形式チェック
    ↓
users存在確認
    ↓
UserContext設定
    ↓
Action
```

とする。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 29.3 FormRequest

ACC-005専用のFormRequestは作成しない。

本APIでは、

```text
リクエストボディ
    なし

クエリパラメータ
    なし
```

であり、ACC-005固有のRequest Bodyバリデーションが存在しないためである。

以下のような空のFormRequestは作成しない。

```php
final class ListAssetAccountAvailableSettingsRequest
    extends FormRequest
{
}
```

不要なクラスを追加しない。

---

### 29.4 assetAccountIdの形式検証

`assetAccountId`は、ルート制約またはAPI共通のパスパラメータ検証方式で検証する。

概念例：

```php
Route::get(
    '/api/v1/asset-accounts/{assetAccountId}/available-settings',
    ListAssetAccountAvailableSettingsAction::class,
)
    ->where(
        'assetAccountId',
        '[1-9][0-9]*',
    );
```

正の整数形式のみを許可する。

以下を有効なIDとして扱わない。

```text
0
-1
abc
1.5
1e3
10abc
```

形式不正は、

```text
INVALID_ASSET_ACCOUNT_ID
```

へ変換する。

---

### 29.5 Action

Actionは、パスパラメータと利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php
final class ListAssetAccountAvailableSettingsAction
{
    public function __invoke(
        string $assetAccountId,
        ListAssetAccountAvailableSettingsUseCase $useCase,
        ListAssetAccountAvailableSettingsResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                assetAccountId:
                    (int) $assetAccountId,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

`assetAccountId`は、形式検証済みであることを前提に整数へ変換する。

---

### 29.6 Actionで行わないこと

Actionでは、以下を行わない。

- 資産口座検索
- 利用者境界判定
- 論理削除判定
- 利用可能資産設定履歴検索
- 履歴0件判定
- 履歴整合性判定
- 並び順制御
- DBアクセス
- レスポンス配列生成

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 29.7 UseCase

ACC-005の参照ユースケース全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `assetAccountId`を受け取る
3. 操作対象利用者に属する有効な資産口座を取得する
4. 対象不存在の場合は業務例外を送出する
5. 利用可能資産設定履歴を取得する
6. 履歴0件を確認する
7. 履歴整合性を検証する
8. API返却順へ整列する
9. Result DTOへ変換する
10. DTO一覧を返却する

概念的には、

```text
userId
+
assetAccountId
    ↓
AssetAccountQuery
    ↓
資産口座取得
    ↓
AssetAccountAvailableSettingQuery
    ↓
履歴取得
    ↓
0件確認
    ↓
AvailableSettingHistoryValidator
    ↓
履歴整合性確認
    ↓
Result DTO一覧
```

とする。

---

### 29.8 AssetAccountQuery

対象資産口座の取得は、専用Queryへ委譲する。

概念例：

```php
$assetAccount =
    $this->assetAccountQuery
        ->findActiveByIdAndUser(
            assetAccountId:
                $assetAccountId,

            userId:
                $userId,
        );
```

概念的な条件は、以下とする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 29.9 利用者境界をQueryへ含める

以下のように、IDだけで資産口座を取得してからUseCase側で所有利用者を判定する方式を基本としない。

```php
$assetAccount =
    AssetAccount::find(
        $assetAccountId,
    );
```

対象取得時点から、

```text
assetAccountId
+
userId
+
利用中状態
```

を条件へ含める。

これにより、他利用者の資産口座を誤って取得することを防止する。

---

### 29.10 SoftDeletes

`AssetAccount` Modelでは、Laravelの

```php
use SoftDeletes;
```

を使用する。

ACC-005の資産口座取得では、

```php
withTrashed()
```

を使用しない。

論理削除済み資産口座は、履歴取得対象外とする。

---

### 29.11 ASSET_ACCOUNT_NOT_FOUND

対象資産口座を取得できない場合は、業務例外を送出する。

概念例：

```php
if ($assetAccount === null) {
    throw new
        AssetAccountNotFoundException();
}
```

以下を同じ例外へ集約する。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

最終的に、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

へ変換する。

---

### 29.12 AssetAccountAvailableSettingQuery

利用可能資産設定履歴の取得は、専用Queryへ委譲する。

概念例：

```php
$settings =
    $this->availableSettingQuery
        ->findHistoryByAssetAccount(
            assetAccountId:
                $assetAccount->id,
        );
```

概念的な条件は、以下とする。

```text
asset_account_id
    = assetAccountId
```

ACC-005では、現在年月条件を付与しない。

対象資産口座の履歴全体を取得する。

---

### 29.13 履歴の並び順

DB取得時は、履歴整合性検証を行いやすいように

```text
start_year_month ASC
```

で取得してよい。

概念例：

```php
return AssetAccountAvailableSetting::query()
    ->where(
        'asset_account_id',
        $assetAccountId,
    )
    ->orderBy(
        'start_year_month',
    )
    ->get();
```

履歴整合性確認では、古い設定から新しい設定へ順番に比較する方が実装しやすいためである。

---

### 29.14 API返却順と検証順を分けてよい

ACC-005のAPI返却順は、

```text
startYearMonth DESC
```

とする。

一方、履歴整合性検証では、

```text
startYearMonth ASC
```

で処理してよい。

概念的には、

```text
DB取得
    ASC
    ↓
履歴整合性検証
    ↓
Result生成
    ↓
DESCへ変換
    ↓
API返却
```

とする。

または、DBからDESCで取得し、Validator内部でASCへ並び替えてもよい。

実装上、責務が分かりやすい方式を採用する。

---

### 29.15 履歴0件

取得した利用可能資産設定履歴が0件の場合は、正常な空配列として扱わない。

概念例：

```php
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingHistoryNotFoundException();
}
```

最終的に、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

へ変換する。

---

### 29.16 AvailableSettingHistoryValidator

利用可能資産設定履歴の整合性確認は、専用Validatorへ分離する。

概念的な構成は、以下とする。

```text
AvailableSettingHistoryValidator
    ├─ 初期開始年月確認
    ├─ 各期間の大小確認
    ├─ 期間連続性確認
    ├─ 期間重複確認
    ├─ 継続中設定件数確認
    └─ 最新設定確認
```

UseCaseへ複雑な期間検証ロジックを直接書き並べない。

---

### 29.17 Validatorへ渡す値

Validatorには、少なくとも以下を渡す。

```text
assetAccount.startYearMonth
+
利用可能資産設定履歴
```

概念例：

```php
$this->historyValidator
    ->validate(
        assetAccountStartYearMonth:
            $assetAccount->start_year_month,

        settings:
            $settings,
    );
```

---

### 29.18 初期設定開始年月の検証

最古の設定の

```text
start_year_month
```

が、

```text
asset_accounts.start_year_month
```

と一致することを確認する。

概念例：

```php
$first =
    $settings->first();

if (
    $first->start_year_month
    !== $assetAccountStartYearMonth
) {
    throw new
        AssetAccountAvailableSettingHistoryInvalidException(
            reason:
                AvailableSettingHistoryInvalidReason::INITIAL_START_MISMATCH,
        );
}
```

---

### 29.19 各設定期間の大小関係

`end_year_month`が`NULL`ではない設定について、

```text
start_year_month
    <= end_year_month
```

であることを確認する。

例えば、

```text
2026-08 ～ 2026-07
```

のような設定は不正とする。

---

### 29.20 年月比較

年月比較は、文字列の大小比較へ無造作に依存せず、年月として意味のある型へ変換して扱ってよい。

例えば、

```text
YearMonth
```

のValue Objectを導入してもよい。

概念例：

```php
final readonly class YearMonth
{
    public function __construct(
        public int $year,
        public int $month,
    ) {
    }
}
```

ただし、Phase1で過剰になる場合は、`YYYY-MM`形式が保証されていることを前提にCarbon等へ変換して比較してよい。

---

### 29.21 期間連続性の検証

古い設定から順に前後レコードを比較する。

概念的には、

```text
前設定.endYearMonth
    の翌月
        =
次設定.startYearMonth
```

であることを確認する。

例えば、

```text
前設定
2026-01 ～ 2026-06

次設定
2026-07 ～ 2026-12
```

は正常とする。

---

### 29.22 翌月計算

年月単位で翌月を求める処理は、文字列連結などで独自実装しない。

例えば、

```php
$expectedNextStart =
    CarbonImmutable::createFromFormat(
        'Y-m',
        $previous->end_year_month,
    )
        ->addMonth()
        ->format('Y-m');
```

のように、日付ライブラリまたはYearMonth Value Objectを使用する。

12月から翌年1月への年跨ぎも正しく扱う。

---

### 29.23 期間重複の検出

例えば、

```text
前設定
2026-01 ～ 2026-08

次設定
2026-08 ～ NULL
```

の場合は、

```text
前設定.endYearMonth
    >=
次設定.startYearMonth
```

となるため、期間重複として扱う。

ただし、期間連続性チェックを

```text
前設定終了年月の翌月
    =
次設定開始年月
```

として実装すれば、重複と欠落の双方を判定できる。

---

### 29.24 期間欠落の検出

例えば、

```text
前設定
2026-01 ～ 2026-05

次設定
2026-07 ～ NULL
```

では、

```text
前設定終了年月の翌月
    = 2026-06

次設定開始年月
    = 2026-07
```

となるため、期間欠落として扱う。

---

### 29.25 end_year_month = NULLの位置

`end_year_month = NULL`は、最新設定だけに許可する。

古い設定に`NULL`が存在した状態で後続設定が存在する場合は、履歴不整合とする。

例えば、

```text
設定A
2026-01 ～ NULL

設定B
2026-08 ～ NULL
```

は不正とする。

---

### 29.26 継続中設定件数

利用中資産口座では、

```text
end_year_month = NULL
```

の履歴が1件だけ存在することを正常状態とする。

概念例：

```php
$openCount =
    $settings
        ->filter(
            fn ($setting) =>
                $setting->end_year_month
                    === null,
        )
        ->count();

if ($openCount !== 1) {
    throw new
        AssetAccountAvailableSettingHistoryInvalidException(
            reason:
                AvailableSettingHistoryInvalidReason::INVALID_OPEN_PERIOD_COUNT,
        );
}
```

---

### 29.27 最新設定の確認

開始年月が最も新しい設定の

```text
end_year_month
```

が`NULL`であることを確認する。

最新設定に終了年月が存在する場合は、利用中資産口座として現在状態を一意に判定できないため不整合とする。

---

### 29.28 同一開始年月

同一資産口座について、

```text
asset_account_id
+
start_year_month
```

は一意であることを前提とする。

通常はDBのUNIQUE制約によって防止する。

Validatorでも検出可能な場合は、不整合として扱ってよい。

---

### 29.29 不整合理由の内部表現

履歴不整合は、API外部向けには

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ統一する。

一方、内部的には原因を区別してよい。

概念例：

```php
enum AvailableSettingHistoryInvalidReason: string
{
    case INITIAL_START_MISMATCH =
        'INITIAL_START_MISMATCH';

    case INVALID_PERIOD =
        'INVALID_PERIOD';

    case PERIOD_OVERLAP =
        'PERIOD_OVERLAP';

    case PERIOD_GAP =
        'PERIOD_GAP';

    case INVALID_OPEN_PERIOD_COUNT =
        'INVALID_OPEN_PERIOD_COUNT';

    case LATEST_PERIOD_CLOSED =
        'LATEST_PERIOD_CLOSED';

    case DUPLICATE_START_YEAR_MONTH =
        'DUPLICATE_START_YEAR_MONTH';
}
```

これは、ログ・テスト・障害調査用として使用する。

---

### 29.30 内部理由をレスポンスへ公開しない

例えば、

```text
PERIOD_GAP
MULTIPLE_OPEN_PERIODS
```

などの内部診断情報をそのままAPIの`error.code`へ使用しない。

外部向けには、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ集約する。

これにより、API契約を過度に細分化しない。

---

### 29.31 履歴を自動修復しない

Validatorは履歴を検証するだけとする。

以下を行わない。

```text
期間欠落
    → 前設定を自動延長

複数NULL
    → 古い設定を自動終了

最新設定終了済み
    → end_year_monthをNULLへ変更
```

ACC-005はGET APIであり、参照処理に副作用を持たせない。

---

### 29.32 Repositoryを使用しない

ACC-005では、

```text
INSERT
UPDATE
DELETE
```

を行わない。

そのため、ACC-005専用Repositoryは使用しない。

概念的には、

```text
Query
    → 読み取り

Repository
    → 書き込み
```

という共通責務分離方針に従う。

---

### 29.33 トランザクション

ACC-005では、明示的な

```php
DB::transaction()
```

を使用しない。

参照専用であり、履歴表示用途として厳密な同一時点スナップショットを要件としないためである。

---

### 29.34 lockForUpdateを使用しない

ACC-005では、

```php
lockForUpdate()
```

を使用しない。

履歴取得中にACC-006の更新を不必要にブロックしない。

---

### 29.35 Result DTO

利用可能資産設定履歴1件を、専用DTOとして表現する。

概念例：

```php
final readonly class
    AssetAccountAvailableSettingHistoryResult
{
    public function __construct(
        public int $id,
        public string $startYearMonth,
        public ?string $endYearMonth,
        public bool $isAvailable,
    ) {
    }
}
```

---

### 29.36 Collection Result

UseCaseの戻り値として、DTO配列または専用Collection DTOを使用してよい。

概念例：

```php
final readonly class
    AssetAccountAvailableSettingHistoryListResult
{
    /**
     * @param array<
     *   AssetAccountAvailableSettingHistoryResult
     * > $items
     */
    public function __construct(
        public array $items,
    ) {
    }
}
```

Phase1では、単純なDTO配列でもよい。

過剰にWrapper DTOを増やさない。

---

### 29.37 DTOへEloquent Modelを保持しない

以下のようなDTOは基本としない。

```php
final readonly class
    AssetAccountAvailableSettingHistoryResult
{
    public function __construct(
        public AssetAccountAvailableSetting
            $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

---

### 29.38 DTO生成

履歴整合性確認後に、各Eloquent ModelからDTOを生成する。

概念例：

```php
$results =
    $settings
        ->sortByDesc(
            'start_year_month',
        )
        ->map(
            fn (
                AssetAccountAvailableSetting $setting,
            ) =>
                new AssetAccountAvailableSettingHistoryResult(
                    id:
                        $setting->id,

                    startYearMonth:
                        $setting->start_year_month,

                    endYearMonth:
                        $setting->end_year_month,

                    isAvailable:
                        (bool) $setting->is_available,
                ),
        )
        ->values()
        ->all();
```

---

### 29.39 API Resource

利用可能資産設定履歴1件を、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php
final class
    AssetAccountAvailableSettingHistoryResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'startYearMonth'
                => $this->startYearMonth,

            'endYearMonth'
                => $this->endYearMonth,

            'isAvailable'
                => $this->isAvailable,
        ];
    }
}
```

---

### 29.40 Resource Collection

一覧レスポンスでは、Resource Collectionを使用する。

概念例：

```php
return
    AssetAccountAvailableSettingHistoryResource
        ::collection(
            $results,
        );
```

または、Responder内でResource Collectionを組み立てる。

---

### 29.41 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

- `asset_account_id`
- `created_at`
- `updated_at`
- `user_id`
- 資産口座名
- 資産種別
- 残高記録単位
- 資産口座の利用開始年月
- `deleted_at`

また、派生値として

```text
isCurrent
```

もPhase1では返却しない。

---

### 29.42 snake_caseをそのまま返さない

DBカラム名をそのままAPIへ公開しない。

例えば、

```text
start_year_month
end_year_month
is_available
```

は、

```text
startYearMonth
endYearMonth
isAvailable
```

へ変換する。

---

### 29.43 Responder

Responderは、履歴DTO一覧を`200 OK`レスポンスへ変換する。

概念例：

```php
final class
    ListAssetAccountAvailableSettingsResponder
{
    /**
     * @param array<
     *   AssetAccountAvailableSettingHistoryResult
     * > $results
     */
    public function ok(
        array $results,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    =>
                    AssetAccountAvailableSettingHistoryResource
                        ::collection(
                            $results,
                        ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの共通Envelope項目は、API共通レスポンス処理に従う。

---

### 29.44 Responderで行わないこと

Responderでは、以下を行わない。

- 資産口座検索
- 利用者境界確認
- 履歴取得
- 並び順判定
- 履歴0件判定
- 期間重複判定
- 期間欠落判定
- 継続中設定判定
- DBアクセス

Responderは、生成済みDTOをHTTPレスポンスへ変換することに責務を限定する。

---

### 29.45 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `assetAccountId`形式不正 | `INVALID_ASSET_ACCOUNT_ID` |
| 資産口座不存在 | `ASSET_ACCOUNT_NOT_FOUND` |
| 他利用者の資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 論理削除済み資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 利用可能資産設定履歴0件 | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND` |
| 履歴整合性不正 | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

---

### 29.46 HistoryNotFoundException

履歴が0件の場合は、専用例外を送出する。

概念例：

```php
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingHistoryNotFoundException();
}
```

最終的に、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

へ変換する。

---

### 29.47 HistoryInvalidException

履歴整合性に違反した場合は、専用例外を送出する。

概念例：

```php
throw new
    AssetAccountAvailableSettingHistoryInvalidException(
        reason:
            AvailableSettingHistoryInvalidReason::PERIOD_GAP,
    );
```

外部向けには、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ変換する。

---

### 29.48 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- SQL
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

---

### 29.49 ログ

ACC-005では、必要に応じて以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assetAccountId
historyCount
httpStatus
errorCode
```

`apiId`は、

```text
ACC-005
```

とする。

---

### 29.50 正常時ログ

正常時には、必要に応じて

```text
requestId
userId
apiId = ACC-005
assetAccountId
historyCount
httpStatus = 200
```

を記録する。

履歴の具体的な

```text
startYearMonth
endYearMonth
isAvailable
```

は、通常ログへ不要に出力しない。

---

### 29.51 履歴不整合ログ

以下の場合は、調査に必要な情報を内部ログへ記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

必要に応じて、

```text
requestId
userId
assetAccountId
historyCount
invalidReason
```

を記録する。

---

### 29.52 invalidReason

`ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID`の場合は、内部的に

```text
INITIAL_START_MISMATCH
INVALID_PERIOD
PERIOD_OVERLAP
PERIOD_GAP
INVALID_OPEN_PERIOD_COUNT
LATEST_PERIOD_CLOSED
DUPLICATE_START_YEAR_MONTH
```

などを記録してよい。

クライアントには公開しない。

---

### 29.53 キャッシュ

Phase1では、ACC-005専用のサーバー側キャッシュを使用しない。

毎回、

```text
asset_accounts
+
asset_account_available_settings
```

の最新DB状態を参照する。

React側では、TanStack Query等によるQuery Cacheを使用してよい。

---

### 29.54 テスト実装方針

Laravel側では、Feature Testを中心としてACC-005のAPI契約を確認する。

また、履歴整合性ロジックについてはValidatorのUnit Testを重点的に作成する。

主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `404 Not Found`
- `500 Internal Server Error`
- `X-User-Id`必須
- `assetAccountId`形式
- 利用者境界
- SoftDeletes
- 履歴1件
- 履歴複数件
- 並び順
- 履歴0件
- 初期開始年月
- 期間大小
- 期間重複
- 期間欠落
- 継続中設定件数
- 最新設定
- レスポンス契約
- 副作用なし

---

### 29.55 AssetAccountQueryのDatabase Test

対象資産口座取得について、以下を確認する。

```text
id一致
+
user_id一致
+
deleted_at IS NULL
    ↓
取得できる
```

以下の場合は取得できないことを確認する。

- 存在しないID
- 他利用者の資産口座
- 論理削除済み資産口座

---

### 29.56 AssetAccountAvailableSettingQueryのDatabase Test

以下の条件で対象資産口座の履歴だけを取得できることを確認する。

```text
asset_account_id
    = assetAccountId
```

以下も確認する。

- 他資産口座の設定を取得しない
- 過去設定も取得する
- 現在設定も取得する
- 現在年月で絞り込まない
- 並び順が想定どおりである

---

### 29.57 HistoryValidatorのUnit Test

履歴Validatorでは、年月境界を含めて細かくUnit Testを作成する。

正常例：

```text
2026-01 ～ 2026-06
2026-07 ～ 2026-12
2027-01 ～ NULL
```

正常として通過することを確認する。

---

### 29.58 初期開始年月のUnit Test

以下を確認する。

正常：

```text
assetAccount.startYearMonth
    = 2026-01

最古設定.startYearMonth
    = 2026-01
```

異常：

```text
assetAccount.startYearMonth
    = 2026-01

最古設定.startYearMonth
    = 2026-02
```

異常時は、

```text
INITIAL_START_MISMATCH
```

として検出できることを確認する。

---

### 29.59 期間大小のUnit Test

以下を確認する。

正常：

```text
2026-08 ～ 2026-08
```

異常：

```text
2026-08 ～ 2026-07
```

異常時は、

```text
INVALID_PERIOD
```

として検出できることを確認する。

---

### 29.60 期間連続性のUnit Test

正常例：

```text
2026-01 ～ 2026-06
2026-07 ～ NULL
```

異常例：

```text
2026-01 ～ 2026-05
2026-07 ～ NULL
```

後者は、

```text
PERIOD_GAP
```

として検出できることを確認する。

---

### 29.61 期間重複のUnit Test

以下を用意する。

```text
2026-01 ～ 2026-08
2026-08 ～ NULL
```

重複として検出できることを確認する。

---

### 29.62 年跨ぎのUnit Test

以下のような年跨ぎの連続性を確認する。

```text
設定A
2026-01 ～ 2026-12

設定B
2027-01 ～ NULL
```

正常に連続期間として扱われること。

---

### 29.63 12月から1月への翌月計算

翌月計算が、

```text
2026-12
    ↓
2027-01
```

となることを確認する。

単純な月数加算によって

```text
2026-13
```

などにならないこと。

---

### 29.64 継続中設定件数のUnit Test

以下を確認する。

正常：

```text
end_year_month = NULL
    1件
```

異常：

```text
end_year_month = NULL
    0件
```

異常：

```text
end_year_month = NULL
    2件以上
```

正常以外では履歴不整合となること。

---

### 29.65 最新設定のUnit Test

最新設定が、

```text
end_year_month = NULL
```

の場合は正常とする。

最新設定に

```text
end_year_month = 2026-12
```

などが設定されている場合は、

```text
LATEST_PERIOD_CLOSED
```

として検出できることを確認する。

---

### 29.66 UseCaseのUnit Test

QueryとValidatorをMockして、業務フローを確認する。

正常系：

```text
AssetAccountQuery
    ↓
資産口座取得
    ↓
AvailableSettingQuery
    ↓
履歴1件以上
    ↓
HistoryValidator
    ↓
正常
    ↓
Result DTO一覧
```

資産口座不存在：

```text
AssetAccountQuery
    ↓
null
    ↓
AssetAccountNotFoundException
```

履歴0件：

```text
AvailableSettingQuery
    ↓
0件
    ↓
AssetAccountAvailableSettingHistoryNotFoundException
```

履歴不整合：

```text
HistoryValidator
    ↓
invalid
    ↓
AssetAccountAvailableSettingHistoryInvalidException
```

---

### 29.67 API ResourceのTest

正常時に、各履歴が以下の形式となることを確認する。

```json
{
  "id": "3",
  "startYearMonth": "2027-01",
  "endYearMonth": null,
  "isAvailable": true
}
```

以下が含まれないことを確認する。

- `assetAccountId`
- `asset_account_id`
- `userId`
- `user_id`
- `createdAt`
- `created_at`
- `updatedAt`
- `updated_at`
- `isCurrent`

---

### 29.68 Resource CollectionのTest

複数履歴の場合に、

```text
startYearMonth DESC
```

で返却されることを確認する。

例えば、

```text
2027-01
2026-07
2026-01
```

の順となること。

---

### 29.69 副作用なしのTest

ACC-005実行前後で、以下のテーブルに変更がないことを確認する。

```text
asset_accounts
asset_account_available_settings
```

特に、

```text
created_at
updated_at
deleted_at
```

が履歴取得によって変更されないことを確認する。

---

### 29.70 GET処理で修復しないことのTest

履歴不整合データを用意してACC-005を実行する。

以下を確認する。

```text
履歴不整合
    ↓
500エラー
```

となり、

```text
asset_account_available_settings
```

に対する

```text
INSERT
UPDATE
DELETE
```

が一切行われないこと。

---

## 30. React・TypeScriptでの利用

ACC-005は、指定した資産口座の利用可能資産設定履歴を画面へ表示するために使用する。

主な利用フローは、以下とする。

```text
ACC-003
資産口座詳細取得
    ↓
資産口座詳細画面
    ↓
利用可能資産設定履歴表示
    ↓
ACC-005
利用可能資産設定履歴取得
```

また、利用可能資産区分を変更する画面では、ACC-006実行前後の履歴確認にも使用できる。

ACC-005は参照専用APIであるため、TanStack Queryを使用する場合はQueryとして扱う。

---

### 30.1 TypeScript型

利用可能資産設定履歴1件は、以下のような型として扱う。

概念例：

```typescript
export type AssetAccountAvailableSettingHistory = {
  id: string;
  startYearMonth: string;
  endYearMonth: string | null;
  isAvailable: boolean;
};
```

ACC-005の正常レスポンスは、履歴配列となる。

```typescript
export type AssetAccountAvailableSettingHistoryResponse =
  ApiResponse<
    AssetAccountAvailableSettingHistory[]
  >;
```

---

### 30.2 id

`id`は、利用可能資産設定IDとして`string`で扱う。

概念例：

```typescript
type AssetAccountAvailableSettingHistory = {
  id: string;
};
```

DB上の`bigint`をReact側で数値型へ変換しない。

---

### 30.3 startYearMonth

`startYearMonth`は、

```text
YYYY-MM
```

形式の文字列として扱う。

概念例：

```typescript
type AssetAccountAvailableSettingHistory = {
  startYearMonth: string;
};
```

表示時には、必要に応じて

```text
2026-08
    ↓
2026年8月
```

へ変換してよい。

---

### 30.4 endYearMonth

`endYearMonth`は、

```text
string | null
```

として扱う。

概念例：

```typescript
type AssetAccountAvailableSettingHistory = {
  endYearMonth: string | null;
};
```

終了年月が存在しない場合は、

```text
null
```

となる。

空文字や特別なダミー年月へ変換しない。

---

### 30.5 isAvailable

`isAvailable`は、booleanとして扱う。

概念例：

```typescript
type AssetAccountAvailableSettingHistory = {
  isAvailable: boolean;
};
```

画面表示時は、必要に応じて

```text
true
    → 利用可能

false
    → 利用対象外
```

などの表示文言へ変換する。

---

### 30.6 isCurrentを型へ追加しない

Phase1では、APIから

```text
isCurrent
```

は返却されない。

そのため、APIレスポンス型へ

```typescript
isCurrent: boolean;
```

を追加しない。

現在継続中の設定は、

```typescript
endYearMonth === null
```

から判定できる。

---

### 30.7 assetAccountId

ACC-005では、対象となる資産口座IDをAPI Client関数の引数として渡す。

概念例：

```typescript
getAssetAccountAvailableSettings(
  assetAccountId,
);
```

`assetAccountId`は、API契約に合わせて`string`として扱う。

---

### 30.8 userIdを関数引数へ含めない

ACC-005専用API Clientへ、

```typescript
getAssetAccountAvailableSettings(
  userId,
  assetAccountId,
);
```

のように`userId`を渡さない。

利用者IDは、共通API Clientから

```text
X-User-Id
```

として付与する。

---

### 30.9 X-User-Id

`X-User-Id`は、ACC-005専用処理ではなく、共通API Clientから付与する。

概念例：

```typescript
apiClient.interceptors.request.use(
  (config) => {
    config.headers['X-User-Id'] =
      currentUserId;

    return config;
  },
);
```

各画面やHookから直接ヘッダーを組み立てない。

---

### 30.10 API Client

ACC-005を呼び出す専用関数を定義する。

概念例：

```typescript
export const getAssetAccountAvailableSettings =
  async (
    assetAccountId: string,
  ): Promise<
    AssetAccountAvailableSettingHistory[]
  > => {
    const response =
      await apiClient.get<
        AssetAccountAvailableSettingHistoryResponse
      >(
        `/api/v1/asset-accounts/${assetAccountId}/available-settings`,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

### 30.11 Queryとして扱う

ACC-005は参照専用GET APIであるため、TanStack QueryではQueryとして扱う。

概念例：

```typescript
export const useAssetAccountAvailableSettings =
  (
    assetAccountId: string,
  ) =>
    useQuery({
      queryKey:
        assetAccountKeys
          .availableSettings(
            assetAccountId,
          ),

      queryFn: () =>
        getAssetAccountAvailableSettings(
          assetAccountId,
        ),
    });
```

---

### 30.12 Query Key

ACC-005のQuery Keyは、資産口座詳細とは別キーにする。

概念的には、

```text
assetAccounts
+
assetAccountId
+
availableSettings
```

とする。

例えば、

```typescript
[
  'assetAccounts',
  assetAccountId,
  'availableSettings',
]
```

とする。

---

### 30.13 Query Keyの共通化

Query Keyは、機能内で共通管理してよい。

概念例：

```typescript
export const assetAccountKeys = {
  all: [
    'assetAccounts',
  ] as const,

  detail: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
    ] as const,

  availableSettings: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
      'availableSettings',
    ] as const,
};
```

ACC-003とACC-005のQuery Cacheを同じキーへ混在させない。

---

### 30.14 assetAccountIdが存在しない場合

画面初期化時などで`assetAccountId`が取得できていない場合は、ACC-005を実行しない。

概念例：

```typescript
useQuery({
  queryKey:
    assetAccountKeys
      .availableSettings(
        assetAccountId,
      ),

  queryFn: () =>
    getAssetAccountAvailableSettings(
      assetAccountId,
    ),

  enabled:
    assetAccountId !== '',
});
```

---

### 30.15 ルートパラメータから取得する場合

資産口座詳細画面では、React Routerなどから`assetAccountId`を取得してよい。

概念例：

```typescript
const {
  assetAccountId,
} = useParams<{
  assetAccountId: string;
}>();
```

値が存在することを確認してからACC-005を実行する。

---

### 30.16 ACC-003と並行取得してよい

資産口座詳細画面では、

```text
ACC-003
資産口座詳細

ACC-005
利用可能資産設定履歴
```

をそれぞれ独立したQueryとして並行取得してよい。

概念的には、

```text
AssetAccountDetailPage
    ├─ useAssetAccountDetail()
    └─ useAssetAccountAvailableSettings()
```

とする。

---

### 30.17 ACC-003の取得成功を必須条件にしなくてよい

ACC-005自身でも資産口座の利用者境界と存在確認を行う。

そのため、技術的には

```text
ACC-003成功
    ↓
ACC-005実行
```

と逐次実行する必要はない。

同じ`assetAccountId`が確定していれば、並行して取得してよい。

ただし、画面構成上必要であればACC-003成功後に表示する方式でもよい。

---

### 30.18 ローディング表示

ACC-005取得中は、履歴領域へローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return (
    <AvailableSettingHistoryLoading />
  );
}
```

履歴取得中に空配列を正常状態として表示しない。

---

### 30.19 履歴一覧表示

正常取得後は、返却された配列を使用して履歴一覧を表示する。

概念例：

```typescript
const {
  data: settings,
} =
  useAssetAccountAvailableSettings(
    assetAccountId,
  );
```

```tsx
<ul>
  {settings?.map(
    (setting) => (
      <li key={setting.id}>
        ...
      </li>
    ),
  )}
</ul>
```

---

### 30.20 APIの並び順を利用する

ACC-005では、API側で

```text
startYearMonth DESC
```

として返却する。

そのため、通常画面では受け取った順序をそのまま表示してよい。

例えば、

```text
2027-01 ～ 現在
2026-07 ～ 2026-12
2026-01 ～ 2026-06
```

のように、最新履歴から表示する。

---

### 30.21 不要な再ソートを行わない

以下のように、毎回React側で正式な履歴順を再構築する必要はない。

```typescript
settings.sort(...);
```

API契約上の並び順を使用する。

画面要件として古い順表示が必要な場合のみ、表示用にコピーして並び替えてよい。

---

### 30.22 現在設定の表示

継続中設定は、

```typescript
setting.endYearMonth === null
```

で判定できる。

概念例：

```typescript
const isCurrent =
  setting.endYearMonth === null;
```

これは画面表示用の派生値であり、API型へ保存し直す必要はない。

---

### 30.23 期間表示

期間は、例えば以下のように表示できる。

```text
2026年1月 ～ 2026年6月
```

継続中の場合は、

```text
2027年1月 ～ 現在
```

のように表示してよい。

概念例：

```typescript
const formatPeriod = (
  startYearMonth: string,
  endYearMonth: string | null,
): string => {
  const start =
    formatYearMonth(
      startYearMonth,
    );

  if (
    endYearMonth === null
  ) {
    return `${start} ～ 現在`;
  }

  return `${start} ～ ${formatYearMonth(
    endYearMonth,
  )}`;
};
```

---

### 30.24 年月表示関数を共通化する

`YYYY-MM`を

```text
2026年8月
```

へ変換する処理は、各コンポーネントへ重複記述せず、共通Utilityへ分離してよい。

概念例：

```typescript
export const formatYearMonth =
  (
    yearMonth: string,
  ): string => {
    const [
      year,
      month,
    ] = yearMonth.split('-');

    return `${year}年${Number(
      month,
    )}月`;
  };
```

正式な日付Utilityは、React共通設計に従う。

---

### 30.25 isAvailableの表示

`isAvailable`は、表示用ラベルへ変換する。

概念例：

```typescript
export const getAvailableLabel =
  (
    isAvailable: boolean,
  ): string =>
    isAvailable
      ? '利用可能'
      : '利用対象外';
```

または、

```tsx
<span>
  {setting.isAvailable
    ? '利用可能'
    : '利用対象外'}
</span>
```

とする。

---

### 30.26 API値を表示文言へ置き換えてState保存しない

以下のように、取得直後に

```text
true
    ↓
利用可能
```

へ変換してAPIデータ自体を書き換えない。

型としてはbooleanを維持し、表示時にだけ文言へ変換する。

---

### 30.27 履歴0件UIを正常系として用意しない

ACC-005の契約では、利用中資産口座に利用可能資産設定履歴が0件となることはデータ不整合である。

そのため、通常状態として

```text
履歴はありません
```

を表示する空一覧UIを必須とはしない。

履歴0件は、サーバーエラーとして扱う。

---

### 30.28 ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND

以下が返却された場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

サーバー側の業務データ不整合として扱う。

フロントエンドで空配列へ変換して履歴なし表示にはしない。

例えば、

```text
利用可能資産設定履歴を
取得できませんでした。
```

などの一般的な取得失敗として表示する。

---

### 30.29 ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID

以下が返却された場合も、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

サーバー側データ不整合として扱う。

フロントエンドで、

- 期間重複を修正する
- 欠落期間を補完する
- 最新設定を推測する
- 複数の継続設定から1件を選ぶ

などの処理は行わない。

---

### 30.30 不整合理由を推測しない

APIからは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

までしか返却しない。

そのため、フロントエンドで

```text
期間重複です
期間が欠落しています
最新設定がありません
```

などと不整合理由を推測して表示しない。

一般利用者向けには共通エラー表示とする。

---

### 30.31 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

となる。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

フロントエンドでは、理由を推測せず、

```text
指定された資産口座が
見つかりません。
```

などの共通表示とする。

---

### 30.32 他利用者かどうかを推測しない

`ASSET_ACCOUNT_NOT_FOUND`を受けても、

```text
他の利用者の資産口座です
```

などと表示しない。

API契約上、存在しない場合との区別はできない。

---

### 30.33 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-005専用画面で独自処理を実装しない。

---

### 30.34 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、利用者コンテキストに関する共通エラーとして扱う。

---

### 30.35 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択されている利用者が有効ではない状態として共通処理する。

---

### 30.36 INVALID_ASSET_ACCOUNT_ID

通常の画面遷移では、ACC-001やACC-003から取得した有効な資産口座IDを使用するため、発生頻度は低い。

URL手入力などで発生した場合は、不正な画面状態として扱い、資産口座一覧へ戻す導線を表示してよい。

---

### 30.37 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
利用可能資産設定履歴を
取得できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

### 30.38 Retry

ACC-005は参照専用GET APIであるため、一時的な通信エラーについてはTanStack QueryのRetry機能を利用してよい。

ただし、

```text
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

のような再実行で改善しにくいエラーに対して無意味なRetryを繰り返さない。

具体的なRetry方針は、React共通設計に従う。

---

### 30.39 ACC-006との連携

利用可能資産区分を変更する場合は、ACC-006をMutationとして実行する。

概念的には、

```text
ACC-005
履歴表示
    ↓
利用者が設定変更
    ↓
ACC-006
新規設定登録
    ↓
成功
    ↓
ACC-005再取得
```

とする。

---

### 30.40 ACC-006成功後のQuery Cache

ACC-006成功後は、ACC-005の履歴内容が変化する。

そのため、対象資産口座の履歴Queryをinvalidateする。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys
      .availableSettings(
        assetAccountId,
      ),
});
```

---

### 30.41 ACC-003のQuery Cacheも無効化する

ACC-006によって現在の利用可能資産区分が変化すると、ACC-003の

```text
isAvailable
```

も変化する。

そのため、ACC-006成功後はACC-003の詳細Queryもinvalidateする。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

---

### 30.42 ACC-006成功レスポンスだけで履歴を再構築しない

技術的には、ACC-006のRequestやResponseからQuery Cacheを手動更新することもできる。

ただし、Phase1では

```text
ACC-006成功
    ↓
ACC-005 invalidate
    ↓
サーバーから最新履歴取得
```

を基本とする。

特に、

```text
旧設定のendYearMonth
新設定ID
新設定startYearMonth
```

などの最終確定状態はサーバーを正とする。

---

### 30.43 Optimistic Update

ACC-005自体は参照Queryである。

ACC-006実行時に、ACC-005の履歴をOptimistic UpdateすることはPhase1では必須としない。

履歴は期間整合性を伴うため、サーバー成功後の再取得を基本とする。

---

### 30.44 ACC-004成功後

ACC-004は、

```text
name
assetType
```

を更新するが、

```text
asset_account_available_settings
```

は変更しない。

そのため、ACC-004成功だけを理由としてACC-005の履歴Queryを必ずinvalidateする必要はない。

ただし、同一画面上でACC-003の資産口座情報とACC-005の履歴を一体表示している場合は、ACC-003側を再取得する。

---

### 30.45 資産口座無効化後

資産口座が無効化されると、ACC-005の通常取得対象から外れる。

そのため、無効化成功後は対象資産口座の履歴Query Cacheを削除または無効化する。

概念例：

```typescript
queryClient.removeQueries({
  queryKey:
    assetAccountKeys
      .availableSettings(
        assetAccountId,
      ),
});
```

---

### 30.46 利用者切替時

利用者切替が行われた場合は、前利用者の利用可能資産設定履歴を新しい利用者へ誤表示しない。

Query Keyへ利用者IDを含める、または利用者切替時に関連Cacheを無効化する。

---

### 30.47 Query KeyへuserIdを含めてもよい

利用者切替を明示的に考慮する場合は、以下のようなQuery Keyとしてよい。

概念例：

```typescript
export const assetAccountKeys = {
  availableSettings: (
    userId: string,
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      userId,
      assetAccountId,
      'availableSettings',
    ] as const,
};
```

正式なQuery Key設計は、React共通設計に従う。

---

### 30.48 履歴レコード単位のQueryを作らない

ACC-005は履歴一覧を1レスポンスで返却する。

そのため、

```text
availableSettingHistory/{settingId}
```

のような個別QueryをACC-005のためだけに作成する必要はない。

配列全体を1つのQuery Cacheとして扱う。

---

### 30.49 履歴IDを編集用途に使用しない

ACC-005で返却される

```text
id
```

は、履歴レコードの識別用に使用できる。

例えばReactの

```tsx
key={setting.id}
```

に使用してよい。

ただし、Phase1では個別履歴を直接編集・削除するAPIは提供しないため、

```text
settingIdを使って
過去履歴を編集する
```

ようなUIは作らない。

---

### 30.50 過去履歴を編集しない

ACC-005で表示した過去履歴について、直接

```text
startYearMonth
endYearMonth
isAvailable
```

を変更する編集UIは提供しない。

利用可能資産区分の変更は、ACC-006で新しい設定を登録することで行う。

---

### 30.51 現在設定も直接書き換えない

現在継続中の設定についても、履歴レコードそのものを直接PATCHする方式は採用しない。

概念的には、

```text
現在設定
    ↓
直接UPDATE
```

ではなく、

```text
ACC-006
    ↓
旧設定を終了
+
新設定を登録
```

とする。

---

### 30.52 履歴コンポーネントの責務

履歴表示コンポーネントでは、主に以下を担当する。

- 履歴配列の表示
- 年月の表示変換
- `isAvailable`の表示変換
- 継続中設定の表示
- ローディング表示
- 取得エラー表示

利用者境界判定や履歴整合性判定をコンポーネントへ持たせない。

---

### 30.53 Query Hookの責務

Query Hookでは、主に以下を担当する。

```text
ACC-005実行
Query Key管理
Query Cache管理
Retry
```

履歴表示レイアウトや業務文言生成をQuery Hookへ持たせない。

---

### 30.54 API Clientの責務

API Clientでは、

```http
GET /api/v1/asset-accounts/{assetAccountId}/available-settings
```

のHTTP通信と型付きレスポンス取得を担当する。

以下はAPI Clientの責務に含めない。

- Toast表示
- 画面遷移
- 履歴期間表示
- 利用可能/利用対象外の文言変換
- データ不整合補正

---

### 30.55 Pageの責務

資産口座詳細Pageでは、必要に応じて

```text
ACC-003
+
ACC-005
```

を組み合わせる。

概念的には、

```text
AssetAccountDetailPage
    ├─ AssetAccountDetailSection
    │      ↓
    │    ACC-003
    │
    └─ AvailableSettingHistorySection
           ↓
         ACC-005
```

とする。

---

### 30.56 概念的なディレクトリ構成

例えば、以下のように整理できる。

```text
features/
└── asset-accounts/
    ├── api/
    │   ├── getAssetAccountDetail.ts
    │   ├── getAssetAccountAvailableSettings.ts
    │   └── createAssetAccountAvailableSetting.ts
    ├── components/
    │   ├── AssetAccountDetail.tsx
    │   └── AvailableSettingHistory.tsx
    ├── hooks/
    │   ├── useAssetAccountDetail.ts
    │   ├── useAssetAccountAvailableSettings.ts
    │   └── useCreateAssetAccountAvailableSetting.ts
    ├── types/
    │   └── assetAccount.ts
    └── pages/
        └── AssetAccountDetailPage.tsx
```

正式な構成は、Reactアーキテクチャ設計に従う。

---

### 30.57 フロントエンドで履歴整合性を再検証しない

ACC-005では、バックエンド側で履歴整合性を確認する。

そのため、React側で再度、

- 初期開始年月一致
- 期間重複
- 期間欠落
- 継続中設定件数
- 最新設定の終了年月

を業務ルールとして検証しない。

正常レスポンスで返却された履歴は、整合しているものとして扱う。

---

### 30.58 表示上の派生値はReact側で計算してよい

業務整合性判定は行わないが、画面表示に必要な派生値はReact側で計算してよい。

例えば、

```typescript
const isCurrent =
  setting.endYearMonth === null;
```

や、

```typescript
const periodLabel =
  formatPeriod(
    setting.startYearMonth,
    setting.endYearMonth,
  );
```

などである。

---

### 30.59 DBカラム名を使用しない

React側の型では、以下のようなsnake_caseを使用しない。

```typescript
type AvailableSetting = {
  start_year_month: string;
  end_year_month: string | null;
  is_available: boolean;
};
```

API契約に従い、

```typescript
type AvailableSetting = {
  startYearMonth: string;
  endYearMonth: string | null;
  isAvailable: boolean;
};
```

とする。

---

### 30.60 assetAccountIdを各履歴へ追加しない

APIからは各履歴へ

```text
assetAccountId
```

を返却しない。

React側でも、APIレスポンス変換時に不要に各履歴へ資産口座IDを複製する必要はない。

必要な場合は、画面コンテキスト側で親の`assetAccountId`を保持する。

---

### 30.61 現在年月を使って履歴をフィルタしない

ACC-005は全履歴取得APIである。

React側で、

```text
現在年月より前だけ
現在設定だけ
```

などへ自動的に絞り込まない。

履歴一覧として全件表示することを基本とする。

画面要件として折りたたみ等を行う場合でも、取得データ自体を欠落させない。

---

### 30.62 フロントエンドで行わないこと

ACC-005のReact・TypeScript実装では、以下をフロントエンドの責務としない。

- 利用者境界の最終保証
- 資産口座存在確認
- 論理削除判定
- 履歴0件の補完
- 初期開始年月整合性判定
- 期間重複判定
- 期間欠落判定
- 継続中設定件数判定
- 不整合履歴の補正
- 現在設定のDB上の確定
- 過去履歴の直接編集
- DB内部項目の解釈

フロントエンドは、

```text
assetAccountId
    ↓
ACC-005
    ↓
正常な履歴一覧取得
    ↓
表示用形式へ変換
    ↓
画面表示
```

という責務を基本とする。

---

## 31. 設計上の補足

### 31.1 履歴取得APIとして分離する理由

ACC-005は、利用可能資産設定の履歴一覧取得に責務を限定する。

現在状態だけを返却するACC-003とは分離する。

概念的には、

```text
ACC-003
    ↓
現在のisAvailable

ACC-005
    ↓
過去から現在までの履歴
```

とする。

現在状態と履歴参照を同じAPIへ混在させないことで、それぞれの責務を明確にする。

---

### 31.2 利用可能資産区分を履歴管理する理由

利用可能資産区分は、対象年月によって変化する可能性がある。

例えば、

```text
2026-01 ～ 2026-06
isAvailable = true

2026-07 ～ 2026-12
isAvailable = false

2027-01 ～
isAvailable = true
```

のような状態を表現する必要がある。

単純に

```text
asset_accounts.is_available
```

として現在値だけを保持すると、過去時点の状態を再現できなくなる。

そのため、

```text
asset_account_available_settings
```

として履歴管理する。

---

### 31.3 年月単位で管理する理由

Life Plannerの資産管理や目的達成判定は、月単位を基本としている。

そのため、利用可能資産設定も

```text
YYYY-MM
```

単位で管理する。

日単位や時刻単位まで細かく管理しない。

概念的には、

```text
2026-08-15
```

ではなく、

```text
2026-08
```

を業務上の設定単位とする。

---

### 31.4 startYearMonthとendYearMonthで期間を表現する理由

利用可能資産設定には、

```text
startYearMonth
endYearMonth
```

を持たせる。

これにより、特定年月にどの設定が有効だったかを明示的に判定できる。

例えば、

```text
startYearMonth = 2026-01
endYearMonth   = 2026-06
```

は、

```text
2026年1月から
2026年6月まで
```

を表す。

---

### 31.5 endYearMonthをNULL許可する理由

現在継続中の設定については、終了年月がまだ確定していない。

そのため、

```text
endYearMonth = null
```

として表現する。

例えば、

```text
startYearMonth = 2026-08
endYearMonth   = null
```

は、

```text
2026-08 ～
```

という継続中の設定を表す。

将来年月を

```text
9999-12
```

などのダミー値で表現しない。

---

### 31.6 最新設定をendYearMonth = nullとする理由

現在利用中の資産口座では、最新設定が現在以降も継続している必要がある。

そのため、正常な履歴では、

```text
最新設定
    endYearMonth = null
```

とする。

これにより、現在設定を明確に識別できる。

---

### 31.7 継続中設定を1件だけとする理由

同一資産口座で、

```text
endYearMonth = null
```

の設定が複数存在すると、

```text
現在どの設定が有効なのか
```

を一意に判断できない。

例えば、

```text
設定A
2026-01 ～ NULL
isAvailable = true

設定B
2026-08 ～ NULL
isAvailable = false
```

では、2026-08以降に2つの異なる状態が同時に成立する。

そのため、継続中設定は1件だけとする。

---

### 31.8 履歴期間を重複させない理由

同じ年月に複数の設定が有効になると、利用可能資産区分を一意に判断できない。

例えば、

```text
設定A
2026-01 ～ 2026-08

設定B
2026-08 ～ NULL
```

では、2026-08が両設定に含まれる。

そのため、正常な履歴では期間を重複させない。

---

### 31.9 履歴期間に欠落を作らない理由

利用可能資産設定は、資産口座の利用開始以降、各年月について必ず状態を判定できる必要がある。

例えば、

```text
設定A
2026-01 ～ 2026-05

設定B
2026-07 ～ NULL
```

では、

```text
2026-06
```

の状態を判断できない。

そのため、期間に欠落を作らない。

---

### 31.10 期間を連続させる理由

利用可能資産設定履歴は、概念的に

```text
前設定終了年月の翌月
    =
次設定開始年月
```

となるように管理する。

例えば、

```text
2026-01 ～ 2026-06
2026-07 ～ 2026-12
2027-01 ～ NULL
```

のようにする。

これにより、資産口座利用開始以降の任意の年月について設定を一意に特定できる。

---

### 31.11 初期設定を資産口座利用開始年月から開始する理由

資産口座は、

```text
startYearMonth
```

から利用開始される。

そのため、最初の利用可能資産設定も同じ年月から開始する。

概念的には、

```text
asset_accounts.start_year_month
    =
最古の設定.start_year_month
```

とする。

これにより、資産口座の利用開始時点から利用可能資産区分を判定できる。

---

### 31.12 ACC-002で初期設定を作成する理由

資産口座だけを登録して利用可能資産設定を後から任意登録とすると、

```text
資産口座は存在する
+
利用可能資産設定は存在しない
```

という不完全な状態が発生する。

そのため、ACC-002では資産口座登録時に初期利用可能資産設定も作成する。

概念的には、

```text
ACC-002
    ↓
asset_accounts
    INSERT
    +
asset_account_available_settings
    INSERT
```

とする。

---

### 31.13 履歴0件を正常な空配列としない理由

ACC-002で初期設定を必ず作成するため、利用中の資産口座で

```text
履歴0件
```

は通常発生しない。

そのため、

```json
{
  "data": []
}
```

として正常状態に見せない。

業務データ不整合として扱う。

---

### 31.14 ACC-005で履歴整合性を検証する理由

ACC-005は、利用可能資産設定履歴を利用者へ表示するAPIである。

不整合状態の履歴をそのまま返却すると、

```text
どの期間が正しいか
どの設定が現在有効か
```

をクライアント側で判断しなければならなくなる。

そのため、バックエンドで履歴が正常であることを確認したうえでレスポンスを返却する。

---

### 31.15 フロントエンドへ履歴整合性判定を任せない理由

期間重複や期間欠落の判定は、業務ルールである。

React側で

```text
期間を並べる
    ↓
重複を探す
    ↓
欠落を探す
```

という処理を実装すると、バックエンドと業務ルールが重複する。

そのため、整合性の最終保証はLaravel側で行う。

---

### 31.16 GETで不整合を自動修復しない理由

ACC-005は参照専用GET APIである。

履歴不整合を検出した際に、

```text
期間終了年月を修正
履歴を追加
履歴を削除
```

すると、GETが副作用を持つ。

HTTP APIとしての予測可能性も低下するため、ACC-005では修復しない。

---

### 31.17 不整合履歴の一部だけを返さない理由

例えば、3件中2件だけが正常でも、その2件だけを返却すると履歴全体の意味が変わる。

ACC-005では、

```text
利用可能資産設定履歴全体
```

を1つの整合した時系列として扱う。

そのため、1件でも業務上重大な不整合があれば正常レスポンスを返却しない。

---

### 31.18 不整合理由をAPIエラーコードとして細分化しない理由

履歴不整合には、

```text
INITIAL_START_MISMATCH
INVALID_PERIOD
PERIOD_OVERLAP
PERIOD_GAP
INVALID_OPEN_PERIOD_COUNT
LATEST_PERIOD_CLOSED
```

など複数の原因が存在する。

しかし、これらはクライアントが個別対応すべき業務エラーではない。

そのため、外部向けには、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ集約する。

内部ログやUnit Testでは、詳細理由を区別してよい。

---

### 31.19 履歴0件だけ別エラーとする理由

履歴0件は、

```text
履歴自体が存在しない
```

という状態であり、存在する履歴の期間不整合とは性質が異なる。

そのため、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

として分離する。

これにより、障害調査時にも原因を把握しやすくする。

---

### 31.20 履歴の返却順を降順とする理由

画面上では、現在または最新の設定を最初に確認する利用が多いと想定する。

そのため、APIでは

```text
startYearMonth DESC
```

で返却する。

例えば、

```text
2027-01 ～ 現在
2026-07 ～ 2026-12
2026-01 ～ 2026-06
```

の順とする。

---

### 31.21 整合性検証では昇順を使用してよい理由

履歴整合性確認では、古い設定から新しい設定へ順に比較すると、

```text
前設定.endYearMonth
次設定.startYearMonth
```

を比較しやすい。

そのため、

```text
検証
    → ASC

レスポンス
    → DESC
```

と内部処理順とAPI返却順を分離してよい。

---

### 31.22 履歴一覧をページングしない理由

利用可能資産設定は、年月単位で変更される。

1資産口座について、短期間に大量の履歴が作成されることは想定しにくい。

Phase1では、ページングを導入することでAPI・React双方の複雑性を増やす必要性が低い。

そのため、履歴全件を返却する。

---

### 31.23 クエリパラメータを設けない理由

ACC-005では、

```text
currentOnly
targetYearMonth
fromYearMonth
toYearMonth
isAvailable
```

などの絞り込み条件を持たせない。

履歴取得APIとして全履歴を返すことに責務を限定する。

現在状態はACC-003、設定変更はACC-006へ分離する。

---

### 31.24 currentOnlyを設けない理由

以下のようなAPIにすると、

```http
GET /api/v1/asset-accounts/{assetAccountId}/available-settings?currentOnly=true
```

ACC-005が

```text
履歴取得
+
現在状態取得
```

の2責務を持つことになる。

現在設定はACC-003で既に取得できるため、重複機能を追加しない。

---

### 31.25 targetYearMonthを設けない理由

特定年月時点の利用可能資産区分が必要な処理は、月末資産管理や目的達成判定などの各ユースケース側で対象年月を基準に判定する。

ACC-005を汎用的な年月検索APIへ広げない。

---

### 31.26 現在年月に依存させない理由

ACC-005は履歴全体を返却する。

そのため、サーバーの

```text
currentYearMonth
```

によって取得結果を変化させない。

同じDB状態であれば、年月が変わっても同じ履歴一覧を返却する。

---

### 31.27 `isCurrent`を返さない理由

現在継続中の設定は、

```text
endYearMonth = null
```

から判定できる。

そのため、

```text
isCurrent
```

という重複した派生情報をPhase1では返却しない。

API項目を必要以上に増やさない。

---

### 31.28 `assetAccountId`を各履歴へ返さない理由

ACC-005では、URLで既に

```text
assetAccountId
```

を指定している。

そのため、各履歴へ

```text
assetAccountId
```

を繰り返し含める必要はない。

レスポンスを履歴固有の情報に限定する。

---

### 31.29 資産口座情報を各履歴へ含めない理由

以下の情報は、利用可能資産設定履歴そのものではない。

```text
name
assetType
balanceRecordingUnit
startYearMonth
isEnabled
```

資産口座詳細はACC-003から取得できるため、ACC-005へ重複して含めない。

---

### 31.30 `createdAt`と`updatedAt`を返さない理由

利用可能資産設定において業務上重要なのは、

```text
startYearMonth
endYearMonth
isAvailable
```

である。

DBへいつINSERTされたか、いつUPDATEされたかという

```text
created_at
updated_at
```

はPhase1の画面要件では不要とする。

---

### 31.31 履歴IDを返す理由

各設定履歴をReact上で一意に識別できるように、

```text
id
```

を返却する。

例えば、

```tsx
key={setting.id}
```

として利用できる。

ただし、履歴IDを返すことと過去履歴の直接編集を許可することは別である。

---

### 31.32 過去履歴を直接編集させない理由

利用可能資産設定は、時系列として整合した履歴を形成する。

過去の1レコードだけを自由に編集すると、

```text
期間重複
期間欠落
現在設定不整合
```

が発生しやすい。

そのため、Phase1では過去履歴の直接編集APIを提供しない。

---

### 31.33 ACC-006で設定変更する理由

利用可能資産区分を変更する際は、

```text
旧設定を終了
+
新設定を追加
```

という履歴操作が必要になる。

そのため、単純なUPDATEではなくACC-006へ責務を集約する。

概念的には、

```text
旧設定
2026-01 ～ NULL
    ↓
ACC-006
    ↓
旧設定
2026-01 ～ 2026-07

新設定
2026-08 ～ NULL
```

とする。

---

### 31.34 ACC-006成功後にACC-005を再取得する理由

ACC-006では、

```text
旧設定.endYearMonth
```

と

```text
新設定
```

の両方が変化する。

React側でRequest内容だけから履歴全体を組み立てるより、

```text
ACC-006成功
    ↓
ACC-005再取得
```

とする方が、サーバー上の確定状態を正として扱える。

---

### 31.35 ACC-003もACC-006後に再取得する理由

ACC-006によって現在の利用可能資産区分が変化すると、

```text
ACC-003.isAvailable
```

も変化する。

そのため、ACC-006成功後は、

```text
ACC-005
履歴

+
ACC-003
現在状態
```

の双方を再取得対象とする。

---

### 31.36 ACC-004後に履歴再取得を必須としない理由

ACC-004では、

```text
name
assetType
```

のみを更新する。

利用可能資産設定履歴は変更されない。

そのため、ACC-004成功だけを理由にACC-005のQuery Cacheを必ず失効させる必要はない。

---

### 31.37 資産口座無効化後に履歴を通常取得させない理由

ACC-005は、現在利用中の資産口座の設定履歴確認を目的とする。

資産口座が論理削除された場合は、通常画面の管理対象から外れる。

そのため、無効化済み資産口座についてACC-005を通常利用させない。

---

### 31.38 論理削除済み資産口座の履歴を物理削除しない理由

資産口座を無効化しても、

```text
asset_account_available_settings
```

の過去履歴を削除しない。

過去の月末資産や目的達成判定を再現するために必要となる可能性があるためである。

ACC-005で通常取得できないことと、DBから履歴を削除することは別とする。

---

### 31.39 Repositoryを使用しない理由

ACC-005は参照専用APIであり、

```text
INSERT
UPDATE
DELETE
```

を行わない。

そのため、取得処理はQueryへ集約し、Repositoryは使用しない。

概念的には、

```text
Query
    → 読み取り

Repository
    → 書き込み
```

とする。

---

### 31.40 AssetAccountQueryを使用する理由

利用可能資産設定には、直接

```text
user_id
```

を持たせない。

そのため、まずAssetAccountQueryで

```text
userId
+
assetAccountId
+
利用中状態
```

を確認する。

利用者境界を確定した後に履歴を取得する。

---

### 31.41 AssetAccountAvailableSettingQueryを分離する理由

資産口座の存在確認と利用可能資産設定履歴の取得は、検索対象・条件が異なる。

そのため、

```text
AssetAccountQuery
    → 親リソース確認

AssetAccountAvailableSettingQuery
    → 履歴取得
```

と分離する。

---

### 31.42 HistoryValidatorを分離する理由

利用可能資産設定履歴には、

- 初期年月
- 期間大小
- 期間重複
- 期間欠落
- 継続中設定件数
- 最新設定

など、複数の整合性ルールが存在する。

これらをUseCaseへ直接書き並べると、UseCaseの責務が大きくなる。

そのため、

```text
AvailableSettingHistoryValidator
```

へ期間整合性判定を分離する。

---

### 31.43 ValidatorをDBアクセス責務にしない理由

HistoryValidatorは、取得済みの履歴を受け取り、整合性を判定する。

Validator自身がDBから履歴を取得しない。

概念的には、

```text
Query
    ↓
履歴取得
    ↓
Validator
    ↓
業務ルール判定
```

とする。

これにより、Queryと業務判定の責務を分離する。

---

### 31.44 YearMonth Value Objectを導入してよい理由

利用可能資産設定では、年月比較や翌月計算を繰り返し使用する。

例えば、

```text
2026-12
    ↓
2027-01
```

の計算や、

```text
2026-07 < 2026-08
```

の比較が必要になる。

将来的に複数のユースケースで同じ年月ロジックを使う場合は、

```text
YearMonth
```

Value Objectへ集約してよい。

---

### 31.45 Phase1でYearMonthを必須としない理由

Value Objectは有効だが、Phase1で利用箇所が限定されている場合は、クラス数を増やしすぎる可能性がある。

そのため、Laravelの日付機能等で十分に安全に扱えるなら、Phase1では専用Value Objectを必須としない。

設計の複雑さと再利用性を見て判断する。

---

### 31.46 FormRequestを作成しない理由

ACC-005には、

```text
Request Body
Query Parameter
```

が存在しない。

専用FormRequestを作成しても空クラスとなるため、作成しない。

`assetAccountId`はルート制約または共通パスパラメータ検証で扱う。

---

### 31.47 明示的なトランザクションを使用しない理由

ACC-005は、参照専用であり業務データを変更しない。

また、履歴表示用途として厳密な同一時点読み取りを必須としない。

そのため、

```php
DB::transaction()
```

を使用しない。

---

### 31.48 `lockForUpdate()`を使用しない理由

ACC-005が履歴参照中であることを理由に、ACC-006の設定変更を待機させる必要はない。

そのため、

```php
lockForUpdate()
```

を使用しない。

更新時の整合性保証は、ACC-006側で行う。

---

### 31.49 DTOを使用する理由

Eloquent ModelをそのままResourceへ渡すと、

```text
asset_account_id
created_at
updated_at
```

などのDB構造へAPI層が依存しやすくなる。

そのため、UseCaseからは必要な値だけを持つResult DTOを返却する。

---

### 31.50 Resource Collectionを使用する理由

ACC-005は一覧取得APIである。

そのため、履歴1件の変換をAPI Resourceへ集約し、一覧はResource Collectionとして返却する。

概念的には、

```text
Result DTO[]
    ↓
Resource Collection
    ↓
data: []
```

とする。

---

### 31.51 Responderを使用する理由

Responderは、

```text
UseCase Result
    ↓
HTTP 200
    ↓
共通Envelope
```

への変換に責務を限定する。

履歴取得や整合性判定をResponderへ持ち込まない。

---

### 31.52 TanStack QueryのQueryとして扱う理由

ACC-005はGETによる参照APIである。

そのため、React側ではMutationではなくQueryとして扱う。

Query Cacheによって、同じ履歴を不要に再取得することを抑制できる。

---

### 31.53 ACC-003とQuery Keyを分ける理由

ACC-003とACC-005は同じ資産口座を対象とするが、データの意味が異なる。

```text
ACC-003
    → 資産口座現在詳細

ACC-005
    → 利用可能資産設定履歴
```

そのため、同じQuery Keyへ両レスポンスを混在させない。

---

### 31.54 ACC-006成功後にinvalidateする理由

ACC-006では、利用可能資産設定履歴が確実に変化する。

そのため、ACC-005のキャッシュを最新状態ではないものとして扱い、

```text
invalidate
    ↓
再取得
```

する。

---

### 31.55 Optimistic Updateを必須としない理由

ACC-006は、

```text
旧設定終了
+
新設定登録
```

という複数レコードの状態変更を伴う。

React側でACC-005の履歴配列を先に推測更新すると、サーバー確定状態との差異が生じる可能性がある。

Phase1では再取得方式を基本とする。

---

### 31.56 利用者切替時にキャッシュ境界を考慮する理由

Phase1では、`X-User-Id`によって操作対象利用者を切り替える。

API側で利用者境界を保証していても、ReactのQuery Cacheに前利用者の履歴が残っていると、画面上で一時的に誤表示する可能性がある。

そのため、Query Keyまたは利用者切替時のinvalidateによってキャッシュ境界も維持する。

---

### 31.57 Phase1ではサーバーキャッシュを導入しない理由

利用可能資産設定履歴は、資産口座単位で比較的少数件である。

また、ACC-006実行後のキャッシュ失効設計も必要になる。

Phase1では、性能上の必要性より複雑性の方が大きいと判断し、Redis等の専用キャッシュを導入しない。

---

### 31.58 Phase1ではHTTP条件付きリクエストを導入しない理由

ACC-005はGET APIであるため、

```text
ETag
Last-Modified
If-None-Match
If-Modified-Since
```

を利用することも可能である。

ただし、Phase1では履歴件数・アクセス頻度とも大規模ではないため、これらを導入しない。

必要になった段階でAPI共通方針として検討する。

---

### 31.59 Phase1では履歴検索機能を広げすぎない

ACC-005では、以下を追加しない。

- ページング
- 年月範囲検索
- 現在設定のみ取得
- 利用可能状態での絞り込み
- 過去履歴編集
- 過去履歴削除
- 履歴の手動補正
- `isCurrent`
- サーバー側キャッシュ
- ETag
- Last-Modified

Phase1では、

```text
指定資産口座の
利用可能資産設定履歴を
整合性確認したうえで
全件取得する
```

ことへ責務を限定する。

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

# ACC-006 利用可能資産設定登録

## 1. 概要

操作対象となる利用者に帰属する指定された資産口座について、新しい利用可能資産設定を登録する。

ACC-006では、現在有効な利用可能資産設定を終了し、新しい設定を追加することで、利用可能資産区分の変更履歴を保持する。

概念的には、以下のように処理する。

```text
現在設定
2026-01 ～ NULL
isAvailable = true
    ↓
ACC-006
startYearMonth = 2026-08
isAvailable = false
    ↓
旧設定
2026-01 ～ 2026-07
isAvailable = true

新設定
2026-08 ～ NULL
isAvailable = false
```

ACC-006は、単純に現在設定の

```text
is_available
```

を上書きするAPIではない。

利用可能資産区分を年月単位の履歴として保持するため、

```text
旧設定の終了
+
新設定の登録
```

を1つの業務操作として扱う。

---

### 1.1 更新対象

ACC-006では、以下のテーブルを更新する。

```text
asset_account_available_settings
```

主な更新内容は、以下とする。

```text
現在設定
    end_year_month更新

新設定
    INSERT
```

一方、以下はACC-006では更新しない。

```text
asset_accounts.name
asset_accounts.asset_type
asset_accounts.balance_recording_unit
asset_accounts.start_year_month
asset_accounts.deleted_at
```

---

### 1.2 ACC-004との違い

ACC-004 資産口座更新は、資産口座の通常属性を更新する。

更新対象は、

```text
name
assetType
```

とする。

一方、ACC-006では、

```text
isAvailable
```

に相当する利用可能資産区分を履歴として変更する。

概念的には、

```text
ACC-004
    ↓
通常属性更新

ACC-006
    ↓
利用可能資産設定履歴変更
```

と責務を分離する。

---

### 1.3 ACC-005との違い

ACC-005は、利用可能資産設定履歴を取得する参照APIである。

ACC-006は、新しい利用可能資産設定を登録する更新APIである。

概念的には、

```text
ACC-005
    → 履歴取得

ACC-006
    → 履歴追加
```

とする。

---

### 1.4 現在設定を直接UPDATEしない理由

例えば、現在の設定が

```text
2026-01 ～ NULL
isAvailable = true
```

の場合に、

```text
is_available = false
```

へ直接変更すると、2026-01以降がすべて`false`だったことになり、過去状態を再現できなくなる。

そのため、

```text
旧設定
    ↓
終了年月を設定

新設定
    ↓
新しい状態を登録
```

とする。

---

### 1.5 新規設定の意味

ACC-006で登録する新しい設定は、

```text
startYearMonth
```

以降に適用される利用可能資産区分を表す。

例えば、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

を登録した場合は、

```text
2026-08以降
利用可能資産として扱わない
```

という意味になる。

---

## 2. ユースケース

利用者は、資産口座を目的達成判定などで利用可能資産として扱うかどうかを変更したい場合にACC-006を使用する。

主な利用例は、以下とする。

- 現在利用可能な資産口座を利用対象外へ変更する
- 現在利用対象外の資産口座を利用可能へ変更する
- 将来の年月から利用可能資産区分を変更する
- 資産口座詳細画面から設定変更する

概念的な画面利用は、以下とする。

```text
ACC-003
資産口座詳細取得
    ↓
現在のisAvailable確認
    ↓
ACC-005
利用可能資産設定履歴確認
    ↓
利用者が設定変更
    ↓
ACC-006
利用可能資産設定登録
    ↓
ACC-003 / ACC-005再取得
```

---

### 2.1 利用可能から利用対象外へ変更

例えば、現在の設定が

```text
2026-01 ～ NULL
isAvailable = true
```

で、2026-08から利用対象外へ変更する場合は、

```text
旧設定
2026-01 ～ 2026-07
true

新設定
2026-08 ～ NULL
false
```

となる。

---

### 2.2 利用対象外から利用可能へ変更

現在、

```text
2026-01 ～ NULL
isAvailable = false
```

の場合に、2026-08から利用可能へ変更すると、

```text
旧設定
2026-01 ～ 2026-07
false

新設定
2026-08 ～ NULL
true
```

となる。

---

### 2.3 過去履歴を直接編集しない

ACC-006では、既存の過去履歴を指定して

```text
過去設定のisAvailable変更
```

を行わない。

設定変更は、

```text
現在設定終了
+
新設定登録
```

として行う。

---

## 3. エンドポイント

```http
POST /api/v1/asset-accounts/{assetAccountId}/available-settings
```

`assetAccountId`には、利用可能資産設定を追加する資産口座IDを指定する。

例：

```http
POST /api/v1/asset-accounts/10/available-settings
```

---

### 3.1 資産口座配下のリソースとする理由

利用可能資産設定は、特定の資産口座に紐づく履歴リソースである。

そのため、

```text
asset-accounts
    ↓
available-settings
```

という親子関係をURLへ表現する。

トップレベルの

```http
POST /api/v1/asset-account-available-settings
```

とはしない。

---

### 3.2 ACC-005とのURL関係

ACC-005とACC-006は、同じリソースURLを使用し、HTTPメソッドによって責務を分離する。

```text
GET
/api/v1/asset-accounts/{assetAccountId}/available-settings
    → ACC-005
      利用可能資産設定履歴取得

POST
/api/v1/asset-accounts/{assetAccountId}/available-settings
    → ACC-006
      利用可能資産設定登録
```

---

## 4. HTTPメソッド

```text
POST
```

ACC-006では、新しい

```text
asset_account_available_settings
```

レコードを追加するため、`POST`を使用する。

---

### 4.1 PATCHを使用しない理由

ACC-006は、既存の1レコードを単純更新する操作ではない。

概念的には、

```text
旧設定
    UPDATE

+
新設定
    INSERT
```

となる。

業務上は、新しい利用可能資産設定を登録する操作であるため、`POST`とする。

---

### 4.2 既存設定IDをURLに含めない理由

ACC-006では、現在設定のIDをクライアントに指定させない。

以下のようなURLにはしない。

```http
PATCH /api/v1/asset-account-available-settings/{settingId}
```

現在有効な設定は、サーバー側で対象資産口座の履歴から特定する。

これにより、クライアントが誤った履歴レコードを直接更新することを防止する。

---

### 4.3 副作用

ACC-006成功時は、通常、

```text
asset_account_available_settings

旧設定
    1件UPDATE

新設定
    1件INSERT
```

という2つの状態変更が発生する。

そのため、これらは同一トランザクション内で処理する。

具体的なトランザクション境界は、後続で定義する。

---

## 5. 利用者コンテキスト

本APIは利用者依存APIのため、`X-User-Id`を必須とする。

操作対象となる利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者コンテキストの特定、`X-User-Id`の検証および利用者境界については、[API共通方針](../api-common-policy.md)に従う。

指定された`assetAccountId`に対応する資産口座が操作対象利用者に帰属する場合のみ、利用可能資産設定を登録できる。

概念的には、以下の条件を満たす必要がある。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 5.1 利用者境界

ACC-006では、`assetAccountId`だけを条件として資産口座を取得してはならない。

まず、

```text
assetAccountId
+
操作対象利用者ID
+
利用中状態
```

によって対象資産口座を確定する。

その後、対象資産口座に紐づく

```text
asset_account_available_settings
```

を更新する。

概念的には、

```text
X-User-Id
    ↓
UserContext
    ↓
asset_accounts
    ↓
利用者境界確認
    ↓
asset_account_available_settings
    ↓
設定変更
```

とする。

---

### 5.2 他利用者の資産口座

指定された`assetAccountId`が他利用者に帰属する場合は、対象資産口座が存在しないものとして扱う。

概念的には、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

とする。

以下のような専用エラーは返却しない。

```text
FORBIDDEN
OTHER_USER_ASSET_ACCOUNT
```

他利用者の資産口座存在有無を外部から推測しにくくするためである。

---

### 5.3 論理削除済み資産口座

ACC-006では、利用中の資産口座だけを設定変更対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座には、新しい利用可能資産設定を登録しない。

論理削除済み資産口座を指定した場合も、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 5.4 利用可能資産設定にはuser_idを持たせない

```text
asset_account_available_settings
```

には、直接

```text
user_id
```

を保持しない。

利用者境界は、親となる

```text
asset_accounts.user_id
```

を通して保証する。

概念的には、

```text
users
    ↓
asset_accounts.user_id
    ↓
asset_accounts.id
    ↓
asset_account_available_settings.asset_account_id
```

という関係とする。

---

### 5.5 settingIdをクライアントから受け付けない

ACC-006では、現在有効な利用可能資産設定IDをリクエストから受け付けない。

例えば、

```json
{
  "currentSettingId": "15",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

のような形式にはしない。

現在設定は、対象資産口座の履歴からサーバー側で特定する。

---

### 5.6 assetAccountIdをリクエストボディから受け付けない

対象資産口座は、URLの

```text
assetAccountId
```

で指定する。

そのため、Request Bodyへ

```json
{
  "assetAccountId": "10"
}
```

のような値を持たせない。

資産口座指定経路を複数にしない。

---

### 5.7 userIdをリクエストボディから受け付けない

以下のような指定によって別利用者の資産口座へ設定を追加できない。

```json
{
  "userId": "999",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

利用者は、

```text
X-User-Id
    ↓
UserContext
```

によってのみ確定する。

---

### 5.8 他利用者の設定を更新しない

例えば、

```text
User A
    Asset Account A
        Setting A

User B
    Asset Account B
        Setting B
```

という状態で、User AとしてAsset Account Bを指定した場合は、

```text
Setting B
```

を取得・更新しない。

資産口座の利用者境界確認時点で、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として処理を終了する。

---

### 5.9 X-User-Idが不正な場合

以下の場合は、資産口座検索や利用可能資産設定更新へ進まない。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正
- 指定された利用者が存在しない
- 指定された利用者が論理削除済み

この場合、

```text
asset_account_available_settings
```

を更新しない。

利用者コンテキストに関するエラーコードおよびHTTPステータスは、API共通方針に従う。

---

### 5.10 1リクエスト1利用者

1回のACC-006リクエストでは、`X-User-Id`で指定された1利用者の資産口座だけを扱う。

以下のように、URLやクエリパラメータから別利用者を指定する方式は採用しない。

```text
/api/v1/users/{userId}/asset-accounts/{assetAccountId}/available-settings
```

または、

```text
?userId=1
```

利用者の指定経路は、`X-User-Id`へ統一する。

---

## 6. パスパラメータ

本APIでは、利用可能資産設定を登録する対象資産口座を指定するために、以下のパスパラメータを使用する。

| パラメータ            | 型      |  必須 | 説明                  |
| ---------------- | ------ | :-: | ------------------- |
| `assetAccountId` | string |  ○  | 利用可能資産設定を登録する資産口座ID |

エンドポイントは、以下とする。

```http
POST /api/v1/asset-accounts/{assetAccountId}/available-settings
```

---

### 6.1 assetAccountId

`assetAccountId`には、利用可能資産設定を変更する資産口座IDを指定する。

例：

```http
POST /api/v1/asset-accounts/10/available-settings
```

APIでは主キーを文字列として扱うが、値としては正の整数形式を前提とする。

正常例：

```text
1
10
123
```

不正例：

```text
0
-1
abc
1.5
```

---

### 6.2 利用者境界

`assetAccountId`だけでは更新対象を確定しない。

実際の資産口座取得では、操作対象利用者IDと組み合わせて検索する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

この条件を満たす資産口座に対してのみ、利用可能資産設定を登録できる。

---

## 7. クエリパラメータ

なし。

ACC-006では、対象資産口座は`assetAccountId`で指定し、新しい利用可能資産設定はリクエストボディで指定する。

以下のようなクエリパラメータは使用しない。

* `userId`
* `settingId`
* `currentSettingId`
* `isAvailable`
* `startYearMonth`
* `force`

---

## 8. リクエストヘッダー

以下のリクエストヘッダーを使用する。

| ヘッダー名          |  必須 | 説明                 |
| -------------- | :-: | ------------------ |
| `X-User-Id`    |  ○  | 操作対象となる利用者ID       |
| `Content-Type` |  ○  | `application/json` |
| `Accept`       |  ○  | `application/json` |

リクエスト例：

```http
POST /api/v1/asset-accounts/10/available-settings
Content-Type: application/json
Accept: application/json
X-User-Id: 1
```

---

### 8.1 X-User-Id

`X-User-Id`は、API共通方針に従って検証する。

本APIの処理開始前に、共通Middlewareで以下を確認する。

* ヘッダーが指定されていること
* IDが共通方針で定めた形式であること
* 利用者が存在すること
* 利用者が論理削除されていないこと

検証済み利用者IDを利用者コンテキストへ保持し、Action以降で使用する。

---

## 9. リクエストボディ

ACC-006では、新しく適用する利用可能資産設定を指定する。

リクエスト項目は、以下とする。

* `startYearMonth`
* `isAvailable`

リクエスト例：

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

このリクエストは、

```text
2026-08から
利用可能資産として扱わない
```

という意味になる。

---

### 9.1 リクエスト項目

| 項目               | 型       |  必須 | NULL | 説明                       |
| ---------------- | ------- | :-: | :--: | ------------------------ |
| `startYearMonth` | string  |  ○  |   ×  | 新しい設定の適用開始年月。`YYYY-MM`形式 |
| `isAvailable`    | boolean |  ○  |   ×  | 利用可能資産として扱うか             |

---

### 9.2 startYearMonth

新しい利用可能資産設定を適用開始する年月を指定する。

形式は、

```text
YYYY-MM
```

とする。

正常例：

```text
2026-08
2027-01
2030-12
```

不正例：

```text
2026-8
2026/08
202608
2026-13
2026-00
abc
```

---

### 9.3 isAvailable

新しい設定期間で、対象資産口座を利用可能資産として扱うかをbooleanで指定する。

利用可能にする場合：

```json
{
  "isAvailable": true
}
```

利用対象外にする場合：

```json
{
  "isAvailable": false
}
```

以下のような文字列や数値は受け付けない。

```json
{
  "isAvailable": "true"
}
```

```json
{
  "isAvailable": 1
}
```

---

### 9.4 endYearMonthを受け付けない

新しい設定の

```text
endYearMonth
```

は、クライアントから指定させない。

ACC-006で登録する新設定は、現在継続中の設定として

```text
end_year_month = NULL
```

で登録する。

終了年月は、次回ACC-006で新しい設定を登録する際にサーバー側で確定する。

---

### 9.5 現在設定IDを受け付けない

以下のような`settingId`は受け付けない。

```json
{
  "currentSettingId": "15",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

現在有効な設定は、サーバー側で対象資産口座の履歴から特定する。

---

### 9.6 assetAccountIdを受け付けない

Request Bodyへ、

```json
{
  "assetAccountId": "10"
}
```

を含めない。

対象資産口座は、URLの

```text
assetAccountId
```

によって指定する。

---

### 9.7 userIdを受け付けない

Request Bodyへ、

```json
{
  "userId": "1"
}
```

を含めない。

操作対象利用者は、`X-User-Id`から特定する。

---

### 9.8 更新対象外項目

以下の項目は、ACC-006では受け付けない。

* `id`
* `userId`
* `user_id`
* `assetAccountId`
* `asset_account_id`
* `settingId`
* `currentSettingId`
* `endYearMonth`
* `end_year_month`
* `createdAt`
* `created_at`
* `updatedAt`
* `updated_at`

未定義項目を拒否するAPI共通方針を採用する場合は、これらを含むRequestをバリデーションエラーとする。

---

## 10. バリデーション

ACC-006では、主に以下を検証する。

```text
X-User-Id
assetAccountId
startYearMonth
isAvailable
```

加えて、入力形式検証後に以下の業務条件を確認する。

```text
対象資産口座存在
現在設定存在
履歴整合性
startYearMonthの妥当性
現在設定との差分
```

業務上の詳細判定は、後続の「業務ルール」で定義する。

---

### 10.1 X-User-Id

`X-User-Id`について、以下を検証する。

* 必須であること
* 共通ID形式に一致すること
* 正の整数として扱えること
* 指定された利用者が存在すること
* 指定された利用者が論理削除されていないこと

不正な場合は、資産口座検索や設定更新へ進まない。

---

### 10.2 assetAccountId

`assetAccountId`は、正の整数形式であることを必須とする。

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

形式不正の場合は、資産口座検索へ進まない。

---

### 10.3 startYearMonth 必須

`startYearMonth`は必須とする。

以下のように未指定の場合は不正とする。

```json
{
  "isAvailable": false
}
```

---

### 10.4 startYearMonth NULL

`startYearMonth`に`null`を指定することはできない。

```json
{
  "startYearMonth": null,
  "isAvailable": false
}
```

は不正とする。

---

### 10.5 startYearMonth形式

`startYearMonth`は、厳密な

```text
YYYY-MM
```

形式であることを確認する。

例えば、

```text
2026-08
```

は正常とする。

以下は不正とする。

```text
2026-8
26-08
2026/08
202608
2026-13
2026-00
```

---

### 10.6 実在する年月

正規表現で見た目だけを確認するのではなく、実在する年月であることを確認する。

例えば、

```text
2026-13
2026-00
```

は不正とする。

---

### 10.7 isAvailable 必須

`isAvailable`は必須とする。

以下のように未指定の場合は不正とする。

```json
{
  "startYearMonth": "2026-08"
}
```

---

### 10.8 isAvailable NULL

`isAvailable`に`null`を指定することはできない。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": null
}
```

は不正とする。

---

### 10.9 isAvailable型

`isAvailable`は、JSON booleanのみを受け付ける。

正常：

```json
{
  "isAvailable": true
}
```

```json
{
  "isAvailable": false
}
```

不正：

```json
{
  "isAvailable": "true"
}
```

```json
{
  "isAvailable": "false"
}
```

```json
{
  "isAvailable": 1
}
```

```json
{
  "isAvailable": 0
}
```

---

### 10.10 資産口座存在確認

形式検証済みの`assetAccountId`について、操作対象利用者に属する利用中の資産口座が存在することを確認する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

該当しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 10.11 他利用者の資産口座

指定された`assetAccountId`が他利用者に属する場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

他利用者に属することを専用エラーで公開しない。

---

### 10.12 論理削除済み資産口座

対象資産口座が

```text
deleted_at IS NOT NULL
```

の場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

ACC-006によって論理削除済み資産口座へ新しい設定を追加しない。

---

### 10.13 利用開始年月より前を許可しない

`startYearMonth`は、対象資産口座の

```text
asset_accounts.start_year_month
```

より前には指定できない。

例えば、

```text
資産口座利用開始年月
2026-08

Request
startYearMonth = 2026-07
```

は不正とする。

---

### 10.14 現在設定開始年月より後を必須とする

ACC-006は、現在有効な設定を終了して新しい設定を追加する操作である。

そのため、新しい`startYearMonth`は、現在設定の

```text
start_year_month
```

より後であることを必須とする。

例えば、

```text
現在設定
startYearMonth = 2026-01
endYearMonth   = NULL
```

の場合、

```text
2026-02
2026-08
2027-01
```

などは候補となる。

一方、

```text
2026-01
2025-12
```

は許可しない。

---

### 10.15 現在設定と同じ開始年月を許可しない

例えば、現在設定が

```text
2026-08 ～ NULL
```

の場合に、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

を送信すると、旧設定の有効期間を正しく終了できない。

そのため、現在設定と同じ`startYearMonth`は不正とする。

---

### 10.16 新設定開始年月の前月を算出できること

ACC-006では、旧設定の終了年月を

```text
新設定.startYearMonthの前月
```

として設定する。

例えば、

```text
新設定
2026-08
```

の場合、

```text
旧設定.endYearMonth
    = 2026-07
```

となる。

そのため、`startYearMonth`は安全に前月計算できる正しい年月値であることを前提とする。

---

### 10.17 年跨ぎ

以下のような年跨ぎも正しく扱う。

```text
新設定.startYearMonth
    = 2027-01

旧設定.endYearMonth
    = 2026-12
```

年月計算を文字列操作だけで実装しない。

---

### 10.18 現在設定の存在確認

ACC-006では、現在継続中の設定として

```text
end_year_month = NULL
```

の履歴が1件存在することを前提とする。

0件の場合は、正常な設定変更を実行できない。

この場合は、業務データ不整合として扱う。

---

### 10.19 現在設定が複数件の場合

以下のように、

```text
end_year_month = NULL
```

の履歴が複数存在する場合も正常状態ではない。

例えば、

```text
2026-01 ～ NULL
true

2026-08 ～ NULL
false
```

のような状態では、どちらを終了させるべきか一意に判断できない。

そのため、ACC-006を実行しない。

---

### 10.20 履歴全体の整合性

ACC-006実行前に、必要に応じて既存履歴全体が正常な状態であることを確認する。

主に以下を確認する。

* 履歴が1件以上存在する
* 初期設定開始年月が資産口座利用開始年月と一致する
* 設定期間が重複していない
* 設定期間に欠落がない
* 継続中設定が1件だけ存在する
* 最新設定が継続中である

不整合状態に新しい履歴を追加して、さらに状態を複雑化しない。

---

### 10.21 同じisAvailableを指定した場合

現在設定と同じ`isAvailable`を指定する場合は、新しい履歴を追加する必要がない。

例えば、

```text
現在設定
isAvailable = true
```

に対して、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

を送信するケースである。

これは状態変更を伴わないため、ACC-006では業務エラーとして扱う。

具体的な独自エラーコードは、後続の「エラーレスポンス」で定義する。

---

### 10.22 同一状態の履歴を追加しない理由

以下のような履歴を作成しない。

```text
2026-01 ～ 2026-07
true

2026-08 ～ NULL
true
```

業務状態が変化していないにもかかわらず履歴を分割すると、設定履歴に不要なレコードが増えるためである。

ACC-006は、利用可能資産区分が実際に変化する場合だけ実行する。

---

### 10.23 過去設定のstartYearMonthを指定しない

ACC-006は、過去履歴を修正するAPIではない。

例えば、

```text
現在設定
2026-08 ～ NULL

Request
startYearMonth = 2026-05
```

のように、現在設定より前の年月を指定して過去履歴を差し替えることはできない。

---

### 10.24 将来年月

Phase1では、現在より将来の年月を`startYearMonth`として指定することを許可してよい。

例えば、現在年月が

```text
2026-08
```

であっても、

```json
{
  "startYearMonth": "2026-10",
  "isAvailable": false
}
```

を登録できる。

この場合、

```text
旧設定
～ 2026-09

新設定
2026-10 ～ NULL
```

となる。

ただし、将来予約を許可するかどうかの最終業務方針は、機能要件に従う。

---

### 10.25 将来設定が既に存在する場合

既存履歴に現在より将来開始の継続中設定が登録されている場合、さらにその前後へ新設定を挿入すると履歴再構成が必要になる可能性がある。

Phase1では、ACC-006を

```text
現在の最新設定を終了し
その後ろへ新設定を追加する
```

操作に限定する。

既存履歴の途中へ設定を挿入する機能は提供しない。

---

### 10.26 更新対象外項目

未定義項目を拒否するAPI共通方針を採用する場合は、以下のようなRequestをバリデーションエラーとする。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false,
  "endYearMonth": "2026-12"
}
```

または、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false,
  "currentSettingId": "10"
}
```

サーバー側で管理する値をクライアントから指定させない。

---

### 10.27 バリデーション失敗時

入力値検証または業務前提条件の確認に失敗した場合は、利用可能資産設定を変更しない。

特に、

```text
asset_account_available_settings
```

に対する

```text
UPDATE
INSERT
```

を途中まで実行しない。

実際の更新は、すべての事前条件を確認した後にトランザクション内で行う。

---

## 11. 業務ルール

ACC-006では、操作対象利用者に帰属する指定された資産口座について、現在の利用可能資産設定を終了し、新しい利用可能資産設定を登録する。

概念的には、

```text
現在設定
2026-01 ～ NULL
isAvailable = true
    ↓
ACC-006
startYearMonth = 2026-08
isAvailable = false
    ↓
旧設定
2026-01 ～ 2026-07
true

新設定
2026-08 ～ NULL
false
```

とする。

ACC-006は、現在設定の`is_available`を直接上書きするAPIではない。

---

### 11.1 操作対象利用者に帰属する資産口座のみ変更できる

対象資産口座は、以下の条件をすべて満たす必要がある。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

`assetAccountId`だけを条件として対象資産口座を取得しない。

---

### 11.2 他利用者の資産口座は変更できない

指定された`assetAccountId`が他利用者に帰属する場合は、対象不存在として扱う。

概念的には、

```text
X-User-Id = 1

assetAccountId = 10

asset_accounts.id = 10
asset_accounts.user_id = 2
    ↓
更新対象外
    ↓
ASSET_ACCOUNT_NOT_FOUND
```

とする。

---

### 11.3 論理削除済み資産口座は変更できない

ACC-006では、利用中の資産口座のみを設定変更対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座には、新しい利用可能資産設定を登録しない。

---

### 11.4 利用可能資産設定は履歴として管理する

利用可能資産区分は、単一の現在値として上書きしない。

以下のような処理は行わない。

```text
現在設定
is_available = true
    ↓
UPDATE
is_available = false
```

これでは、過去時点の状態を再現できなくなるためである。

---

### 11.5 旧設定を終了する

新しい設定を登録する前に、現在継続中の設定の

```text
end_year_month
```

を更新する。

終了年月は、

```text
新設定.startYearMonthの前月
```

とする。

例えば、

```text
新設定.startYearMonth
    = 2026-08
```

の場合、

```text
旧設定.endYearMonth
    = 2026-07
```

とする。

---

### 11.6 新設定を登録する

旧設定終了後、新しい利用可能資産設定を追加する。

新設定は、

```text
start_year_month
    = request.startYearMonth

end_year_month
    = NULL

is_available
    = request.isAvailable
```

とする。

---

### 11.7 新設定は必ず継続中として登録する

ACC-006で登録する新しい設定には、

```text
end_year_month = NULL
```

を設定する。

クライアントから終了年月を指定させない。

終了年月は、次回ACC-006実行時にサーバー側で設定する。

---

### 11.8 現在設定は1件のみ存在する

ACC-006実行前には、現在継続中の設定として

```text
end_year_month = NULL
```

の履歴が1件だけ存在することを正常状態とする。

---

### 11.9 現在設定0件は不整合とする

現在継続中設定が0件の場合は、正常な設定変更を行えない。

例えば、

```text
2026-01 ～ 2026-06
true

2026-07 ～ 2026-12
false
```

しか存在せず、`end_year_month = NULL`の履歴がない状態は不整合とする。

---

### 11.10 現在設定複数件は不整合とする

以下のように、

```text
2026-01 ～ NULL
true

2026-08 ～ NULL
false
```

という状態では、現在設定を一意に判断できない。

そのため、ACC-006を実行しない。

---

### 11.11 既存履歴全体が正常であることを前提とする

ACC-006実行前に、既存の利用可能資産設定履歴が正常であることを確認する。

主に以下を確認する。

- 履歴が1件以上存在する
- 最初の設定開始年月が資産口座利用開始年月と一致する
- 各設定で`startYearMonth <= endYearMonth`
- 設定期間が重複していない
- 設定期間に欠落がない
- `endYearMonth = null`の設定が1件のみ
- 最新設定が継続中である

不整合履歴へ新しい履歴を追加しない。

---

### 11.12 新設定開始年月は現在設定開始年月より後とする

`startYearMonth`は、現在設定の

```text
start_year_month
```

より後であることを必須とする。

例えば、

```text
現在設定
2026-01 ～ NULL
```

の場合、

```text
2026-02
2026-08
2027-01
```

は指定できる。

一方、

```text
2026-01
2025-12
```

は指定できない。

---

### 11.13 現在設定と同じ開始年月は指定できない

例えば、

```text
現在設定
2026-08 ～ NULL
```

に対して、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

を指定することはできない。

旧設定の終了年月を

```text
2026-07
```

とした場合、

```text
startYearMonth > endYearMonth
```

となってしまうためである。

---

### 11.14 過去履歴の途中へ設定を挿入しない

ACC-006は、履歴の末尾へ新しい設定を追加するAPIとする。

例えば、

```text
既存履歴

2026-01 ～ 2026-06
true

2026-07 ～ NULL
false
```

に対して、

```text
startYearMonth = 2026-04
```

の設定を挿入しない。

既存履歴の途中編集や再構成は、Phase1では対象外とする。

---

### 11.15 利用開始年月より前には設定できない

新しい`startYearMonth`は、資産口座の

```text
asset_accounts.start_year_month
```

より前に指定できない。

例えば、

```text
asset_accounts.start_year_month
    = 2026-08
```

の場合、

```text
2026-07
```

から始まる設定を追加できない。

---

### 11.16 同じisAvailableでは新しい履歴を作成しない

現在設定と同じ`isAvailable`を指定した場合は、新しい履歴を追加しない。

例えば、

```text
現在設定
2026-01 ～ NULL
isAvailable = true
```

に対して、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

を指定しても、業務状態は変化しない。

そのため、ACC-006では設定変更なしとして業務エラーとする。

---

### 11.17 不要な履歴分割を防止する

以下のような履歴は作成しない。

```text
2026-01 ～ 2026-07
true

2026-08 ～ NULL
true
```

状態が同一にもかかわらず期間だけ分割すると、履歴件数が不要に増えるためである。

ACC-006は、利用可能資産区分が実際に変化する場合だけ新しい履歴を追加する。

---

### 11.18 利用可能から利用対象外への変更

現在、

```text
2026-01 ～ NULL
isAvailable = true
```

の場合に、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

を登録すると、

```text
旧設定
2026-01 ～ 2026-07
true

新設定
2026-08 ～ NULL
false
```

となる。

---

### 11.19 利用対象外から利用可能への変更

現在、

```text
2026-01 ～ NULL
isAvailable = false
```

の場合に、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

を登録すると、

```text
旧設定
2026-01 ～ 2026-07
false

新設定
2026-08 ～ NULL
true
```

となる。

---

### 11.20 年跨ぎ

年月の前月計算では、年跨ぎを正しく扱う。

例えば、

```text
新設定.startYearMonth
    = 2027-01
```

の場合、

```text
旧設定.endYearMonth
    = 2026-12
```

となる。

---

### 11.21 現在より将来の年月を指定できる

Phase1で将来予約を許可する場合は、現在年月より後の

```text
startYearMonth
```

を指定できる。

例えば、現在年月が

```text
2026-08
```

で、

```json
{
  "startYearMonth": "2026-10",
  "isAvailable": false
}
```

を登録した場合、

```text
旧設定
～ 2026-09

新設定
2026-10 ～ NULL
```

となる。

---

### 11.22 将来設定登録後の意味

将来開始設定を登録すると、旧設定には将来の終了年月が設定される。

例えば、

```text
現在年月
2026-08

旧設定
2026-01 ～ NULL
true
```

に対して、

```text
新設定開始
2026-10
false
```

を登録すると、

```text
旧設定
2026-01 ～ 2026-09
true

新設定
2026-10 ～ NULL
false
```

となる。

---

### 11.23 将来予約が存在する状態でさらに追加しない

Phase1では、既に将来開始の最新設定が存在する状態で、その前後へさらに設定を追加して履歴を再構成する処理は扱わない。

ACC-006は、

```text
現在の最新履歴
    ↓
その後ろへ
新しい設定を追加
```

する操作に限定する。

---

### 11.24 過去設定を直接変更しない

ACC-006では、過去履歴の

```text
start_year_month
end_year_month
is_available
```

を自由に編集しない。

変更する既存レコードは、現在継続中の最新設定の

```text
end_year_month
```

だけとする。

---

### 11.25 asset_accountsは更新しない

ACC-006では、

```text
asset_accounts
```

を更新しない。

以下を変更しない。

```text
name
asset_type
balance_recording_unit
start_year_month
deleted_at
```

利用可能資産区分は、`asset_account_available_settings`だけで管理する。

---

### 11.26 他の関連テーブルを更新しない

ACC-006では、以下を更新しない。

```text
holding_assets
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
net_incomes
objectives
assessment_histories
```

利用可能資産設定履歴変更だけに責務を限定する。

---

### 11.27 過去の月末資産データを変更しない

利用可能資産設定を変更しても、既に登録されている

```text
month_end_asset_balances
month_end_holding_values
month_end_asset_snapshots
```

を変更しない。

設定履歴は、各対象年月時点でどの状態だったかを判定するために使用する。

---

### 11.28 過去の目的達成判定履歴を変更しない

ACC-006によって新しい利用可能資産区分を登録しても、

```text
assessment_histories
```

に保存済みの過去判定結果を自動更新しない。

既存履歴は、判定実行時点の結果として保持する。

---

## 12. 処理フロー

ACC-006の基本処理フローは、以下とする。

```text
Request
    ↓
X-User-Id検証
    ↓
UserContext取得
    ↓
assetAccountId形式検証
    ↓
Request Body検証
    ↓
トランザクション開始
    ↓
対象資産口座取得
    ↓
ASSET_ACCOUNT_NOT_FOUND？
    ↓ No
既存利用可能資産設定履歴取得
    ↓
履歴0件？
    ↓ No
履歴整合性確認
    ↓
現在設定1件を特定
    ↓
startYearMonth妥当性確認
    ↓
isAvailable変更あり？
    ↓ Yes
現在設定をロック
    ↓
旧設定.endYearMonth
    = 新設定.startYearMonthの前月
    ↓
新設定INSERT
    ↓
Result DTO生成
    ↓
コミット
    ↓
201 Created
```

---

### 12.1 利用者コンテキスト確認

共通Middlewareで`X-User-Id`を検証し、操作対象利用者を特定する。

利用者コンテキストが不正な場合は、トランザクションやDB更新へ進まない。

---

### 12.2 assetAccountId検証

`assetAccountId`が正の整数形式であることを確認する。

形式不正の場合は、資産口座検索や更新処理へ進まない。

---

### 12.3 Request Body検証

以下を検証する。

```text
startYearMonth
isAvailable
```

主に、

- 必須
- NULL禁止
- 型
- `YYYY-MM`形式
- 実在年月

を確認する。

---

### 12.4 トランザクション開始

入力形式の検証完了後、業務更新処理を同一トランザクション内で実行する。

概念的には、

```php
DB::transaction(function () {
    // 対象取得
    // 履歴確認
    // 旧設定更新
    // 新設定登録
});
```

とする。

---

### 12.5 資産口座取得

操作対象利用者IDと`assetAccountId`を条件として、利用中の資産口座を取得する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

---

### 12.6 資産口座不存在

対象資産口座を取得できない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

としてトランザクションを終了する。

以下を同じ扱いとする。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

---

### 12.7 既存履歴取得

対象資産口座に紐づく

```text
asset_account_available_settings
```

の履歴を取得する。

概念的には、

```text
asset_account_id
    = assetAccountId
```

を条件とする。

---

### 12.8 履歴0件確認

既存履歴が0件の場合は、ACC-002の初期登録ルールに反する。

そのため、新設定登録へ進まずデータ不整合として扱う。

---

### 12.9 履歴整合性確認

既存履歴について、ACC-005と同じ基本的な整合性ルールを確認する。

主に以下とする。

```text
初期開始年月一致
期間大小正常
期間重複なし
期間欠落なし
継続中設定1件
最新設定が継続中
```

不整合状態へ新しい履歴を追加しない。

---

### 12.10 現在設定取得

整合性確認後、最新かつ継続中の

```text
end_year_month = NULL
```

の設定を現在設定として扱う。

現在設定をクライアント指定のIDから取得しない。

---

### 12.11 startYearMonth業務検証

新設定の`startYearMonth`について、少なくとも以下を確認する。

```text
assetAccount.startYearMonth
    <=
request.startYearMonth

AND

currentSetting.startYearMonth
    <
request.startYearMonth
```

Phase1では、履歴途中への挿入を許可しない。

---

### 12.12 isAvailable差分確認

現在設定の

```text
is_available
```

とRequestの

```text
isAvailable
```

が異なることを確認する。

同一の場合は、不要な履歴追加となるため登録しない。

---

### 12.13 旧設定終了年月算出

Requestの

```text
startYearMonth
```

から1か月減算し、旧設定の終了年月を算出する。

概念的には、

```text
request.startYearMonth
    = 2026-08
        ↓
previousYearMonth
    = 2026-07
```

とする。

---

### 12.14 旧設定更新

現在設定の

```text
end_year_month
```

へ、算出した前月を設定する。

例えば、

```text
Before

2026-01 ～ NULL
true
```

を、

```text
After

2026-01 ～ 2026-07
true
```

へ更新する。

---

### 12.15 新設定INSERT

新しい履歴を以下の内容で登録する。

```text
asset_account_id
    = assetAccountId

start_year_month
    = request.startYearMonth

end_year_month
    = NULL

is_available
    = request.isAvailable
```

---

### 12.16 更新順序

概念的には、

```text
旧設定UPDATE
    ↓
新設定INSERT
```

とする。

ただし、両方を同一トランザクション内で実行するため、途中状態を外部へ確定させない。

---

### 12.17 新設定登録失敗時

旧設定のUPDATE後に新設定INSERTが失敗した場合は、トランザクション全体をロールバックする。

以下の状態を残してはならない。

```text
旧設定
終了済み

+
新設定
存在しない
```

---

### 12.18 旧設定更新失敗時

旧設定の更新に失敗した場合も、新設定を登録しない。

トランザクションをロールバックする。

---

### 12.19 Result DTO生成

新設定登録成功後、APIレスポンスに必要なResult DTOを生成する。

必要に応じて、

```text
新規設定ID
startYearMonth
endYearMonth
isAvailable
```

を保持する。

---

### 12.20 コミット

旧設定更新と新設定登録の両方が正常に完了した場合のみ、トランザクションをコミットする。

---

## 13. 成功レスポンス

ACC-006で新しい利用可能資産設定の登録に成功した場合は、

```http
201 Created
```

を返却する。

レスポンス例：

```json
{
  "data": {
    "id": "15",
    "startYearMonth": "2026-08",
    "endYearMonth": null,
    "isAvailable": false
  }
}
```

新規作成された利用可能資産設定を返却する。

---

### 13.1 レスポンス項目

正常時の`data`配下には、以下を返却する。

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `id` | string | × | 新しく登録した利用可能資産設定ID |
| `startYearMonth` | string | × | 新設定の適用開始年月。`YYYY-MM`形式 |
| `endYearMonth` | string | ○ | 新設定の終了年月。登録直後は`null` |
| `isAvailable` | boolean | × | 新設定の利用可能資産区分 |

---

### 13.2 id

新しく作成された

```text
asset_account_available_settings.id
```

をstringとして返却する。

---

### 13.3 startYearMonth

Requestで指定された

```text
startYearMonth
```

を、新規設定の適用開始年月として返却する。

---

### 13.4 endYearMonth

新規設定は継続中設定として登録されるため、正常レスポンスでは

```json
{
  "endYearMonth": null
}
```

となる。

---

### 13.5 isAvailable

Requestで指定された新しい利用可能資産区分をbooleanとして返却する。

---

### 13.6 旧設定をレスポンスへ含めない

ACC-006成功時に旧設定の

```text
end_year_month
```

も更新されるが、レスポンスへ旧設定を含めない。

必要な場合は、ACC-005を再取得して履歴全体を確認する。

---

### 13.7 資産口座情報を返却しない

ACC-006では、以下の資産口座情報をレスポンスへ含めない。

- `assetAccountId`
- `name`
- `assetType`
- `balanceRecordingUnit`
- `startYearMonth`（資産口座側）
- `isEnabled`

資産口座詳細はACC-003の責務とする。

---

## 14. トランザクション境界

ACC-006では、明示的なDBトランザクションを必須とする。

ACC-006は、

```text
旧設定UPDATE
+
新設定INSERT
```

という複数の状態変更を1つの業務操作として扱うためである。

---

### 14.1 UseCaseをトランザクション境界とする

トランザクションは、Repository単位ではなく、ACC-006のUseCase全体を境界とする。

概念的には、

```text
UseCase
    ↓
DB::transaction
    ├─ 資産口座確認
    ├─ 履歴確認
    ├─ 現在設定ロック
    ├─ 旧設定更新
    └─ 新設定登録
```

とする。

---

### 14.2 原子的に処理する

以下の2処理は、必ず同時に成功または失敗させる。

```text
1.
現在設定.end_year_month更新

2.
新しい利用可能資産設定INSERT
```

片方だけを確定させない。

---

### 14.3 旧設定更新後にINSERT失敗

例えば、

```text
旧設定UPDATE
    成功

新設定INSERT
    失敗
```

となった場合は、旧設定UPDATEもロールバックする。

結果として、実行前の

```text
旧設定
end_year_month = NULL
```

の状態へ戻す。

---

### 14.4 新設定だけ登録しない

以下の状態も発生させない。

```text
旧設定
end_year_month = NULL

新設定
start_year_month = 2026-08
end_year_month = NULL
```

これは継続中設定が複数存在する状態になるためである。

旧設定終了と新設定登録を同一トランザクションで扱う。

---

### 14.5 現在設定取得と更新

ACC-006では、同時実行による競合を防止するため、トランザクション内で現在設定を取得し、必要に応じて行ロックを取得する。

具体的な

```php
lockForUpdate()
```

の使用方針は、後続の「排他制御」で定義する。

---

### 14.6 履歴検証と更新の関係

履歴整合性を確認してから更新する。

ただし、確認から更新までの間に別ACC-006が割り込む可能性があるため、更新対象となる現在設定についてはトランザクション内で排他制御する。

---

### 14.7 Repository内で独自トランザクションを開始しない

以下のように、各Repositoryが個別トランザクションを開始しない。

```text
AvailableSettingRepository
    ↓
transaction

別処理
    ↓
別transaction
```

ACC-006という1つの業務操作全体でトランザクションを管理する。

---

### 14.8 ロールバック対象

以下の場合は、トランザクション全体をロールバックする。

- 現在設定更新失敗
- 新設定INSERT失敗
- UNIQUE制約違反
- DB例外
- 更新対象消失
- 同時更新競合
- その他想定外例外

---

### 14.9 バリデーションエラーはトランザクション開始前に検出する

HTTP入力として判定できる

```text
startYearMonth形式不正
isAvailable型不正
必須項目不足
```

などは、原則としてトランザクション開始前に検出する。

不要なトランザクションを開始しない。

---

### 14.10 業務条件はトランザクション内で再確認する

以下のようなDB状態に依存する条件は、トランザクション内で確認する。

```text
対象資産口座存在
現在設定存在
現在設定件数
現在設定startYearMonth
現在設定isAvailable
既存履歴状態
```

特に、更新直前のDB状態を基準として判定する。

---

### 14.11 トランザクション中に外部I/Oを行わない

ACC-006のトランザクション内では、以下のようなDB以外の外部I/Oを行わない。

```text
外部API
メール送信
ファイル操作
長時間処理
```

Phase1ではDB更新だけにトランザクションを限定する。

---

### 14.12 トランザクションを短く保つ

ACC-006では、必要な

```text
SELECT
LOCK
UPDATE
INSERT
```

だけをトランザクション内で実行する。

レスポンス整形や不要な関連データ取得をトランザクション内へ含めない。

---

## 15. 排他制御

ACC-006では、同一資産口座に対する利用可能資産設定登録の同時実行を考慮する。

ACC-006は、

```text
現在設定を終了
+
新設定を登録
```

という複数レコード更新を伴うため、Phase1では現在設定に対して明示的な行ロックを使用する。

概念的には、

```text
トランザクション開始
    ↓
現在設定取得
    ↓
lockForUpdate()
    ↓
業務条件再確認
    ↓
旧設定UPDATE
    ↓
新設定INSERT
    ↓
コミット
```

とする。

---

### 15.1 lockForUpdateを使用する

現在継続中の

```text
end_year_month = NULL
```

の利用可能資産設定を取得する際は、トランザクション内で

```php
lockForUpdate()
```

を使用する。

これにより、同一の現在設定を複数リクエストが同時に終了させることを防止する。

---

### 15.2 同一資産口座への同時実行

例えば、以下の2リクエストが同時に実行される場合を考える。

```text
Request A
2026-08から
isAvailable = false

Request B
2026-09から
isAvailable = false
```

排他制御を行わない場合、

```text
Request A
現在設定取得

Request B
現在設定取得

Request A
旧設定終了
新設定登録

Request B
同じ旧設定を終了
新設定登録
```

となり、履歴が不整合になる可能性がある。

---

### 15.3 現在設定をロックする理由

ACC-006で変更する既存レコードは、

```text
現在継続中設定
```

である。

そのため、対象となる現在設定行をロックすることで、同じ設定を基点とした同時履歴追加を直列化する。

---

### 15.4 資産口座行は原則ロックしない

Phase1では、以下の

```text
asset_accounts
```

行自体を原則としてロックしない。

利用者境界確認と対象存在確認のために参照するだけであり、ACC-006で更新するのは

```text
asset_account_available_settings
```

だからである。

ただし、将来的に資産口座無効化との厳密な競合制御が必要になった場合は、資産口座行のロックも検討する。

---

### 15.5 ロック取得後に業務条件を再確認する

現在設定を`lockForUpdate()`で取得した後は、そのロック済み状態を基準として業務条件を再確認する。

主に以下を確認する。

- 現在設定が1件だけ存在する
- 現在設定の`startYearMonth`より新設定開始年月が後である
- `isAvailable`が現在設定と異なる
- 現在設定がまだ`endYearMonth = null`である

ロック前に取得した古い状態だけを基準に更新しない。

---

### 15.6 同時実行時の待機

Request Aが現在設定をロックしている間、Request Bが同じ現在設定を取得しようとした場合は、Request Aのトランザクション終了まで待機する。

概念的には、

```text
Request A
lock取得
    ↓
更新
    ↓
commit
    ↓
Request B
lock取得
```

となる。

---

### 15.7 待機後は最新状態を基準に判定する

Request Bは、Request Aのコミット後に最新の現在設定を取得する。

その結果、Request Bの

```text
startYearMonth
isAvailable
```

が最新状態に対して業務条件を満たさない可能性がある。

この場合は、業務エラーとして処理し、履歴を無理に追加しない。

---

### 15.8 UNIQUE制約も併用する

行ロックだけに依存せず、DB側でも

```text
asset_account_id
+
start_year_month
```

の組み合わせにUNIQUE制約を設定する。

これにより、同一資産口座に同じ開始年月の設定が複数作成されることを防止する。

---

### 15.9 UNIQUE制約違反

並行実行などによってUNIQUE制約違反が発生した場合は、PostgreSQL例外をそのまま返却しない。

業務上の競合エラーへ変換する。

具体的な独自エラーコードは、後続の「エラーレスポンス」で定義する。

---

### 15.10 デッドロックを避ける

ACC-006では、ロック取得順を一定にする。

概念的には、

```text
1. assetAccountIdで対象確認
2. 現在設定を取得・ロック
3. 旧設定更新
4. 新設定登録
```

の順序を統一する。

複数テーブル・複数行を無秩序にロックしない。

---

### 15.11 ロック保持時間を短くする

`lockForUpdate()`取得後は、必要な業務判定とDB更新だけを行う。

以下をロック保持中に行わない。

```text
外部API呼び出し
メール送信
長時間計算
画面用レスポンス整形
```

トランザクションとロック保持時間を短く保つ。

---

### 15.12 将来的な楽観ロック

Phase1では、ACC-006について`version`や`updated_at`比較による楽観ロックは使用しない。

現在設定への行ロックによって同一資産口座の設定変更を直列化する。

将来的に高い同時実行性が必要になった場合は、楽観ロックへの変更を検討してよい。

---

## 16. エラーレスポンス

異常時は、API共通方針に従った共通エラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE",
    "message": "利用可能資産区分が現在の設定と同じです。"
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

---

### 16.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

この場合、資産口座検索や設定更新へ進まない。

---

### 16.2 INVALID_USER_ID

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

例えば、以下を不正とする。

```text
0
-1
abc
1.5
```

---

### 16.3 USER_NOT_FOUND

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

---

### 16.4 INVALID_ASSET_ACCOUNT_ID

`assetAccountId`の形式が不正な場合は、

```text
INVALID_ASSET_ACCOUNT_ID
```

を返却する。

例えば、

```text
0
-1
abc
1.5
1e3
10abc
```

などを不正とする。

---

### 16.5 VALIDATION_ERROR

リクエストボディの形式検証に失敗した場合は、

```text
VALIDATION_ERROR
```

を返却する。

主な対象は、

- `startYearMonth`未指定
- `startYearMonth = null`
- `startYearMonth`形式不正
- 実在しない年月
- `isAvailable`未指定
- `isAvailable = null`
- `isAvailable`がboolean以外
- 更新対象外項目の指定

とする。

---

### 16.6 startYearMonth形式不正

例えば、

```json
{
  "startYearMonth": "2026/08",
  "isAvailable": false
}
```

や、

```json
{
  "startYearMonth": "2026-13",
  "isAvailable": false
}
```

は、

```text
VALIDATION_ERROR
```

として扱う。

---

### 16.7 isAvailable型不正

以下のような値は受け付けない。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": "false"
}
```

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": 0
}
```

JSON booleanのみを許可する。

---

### 16.8 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

- 指定された資産口座が存在しない
- 指定された資産口座が他利用者に属する
- 指定された資産口座が論理削除済み

他利用者所属や論理削除済みであることを個別のエラーコードで公開しない。

---

### 16.9 他利用者の資産口座

例えば、

```text
User A
    assetAccountId = 10

User B
    X-User-Id = 2
```

の状態で、User Bが

```http
POST /api/v1/asset-accounts/10/available-settings
```

を実行した場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

User Aの利用可能資産設定を変更しない。

---

### 16.10 論理削除済み資産口座

対象資産口座が

```text
deleted_at IS NOT NULL
```

の場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

ACC-006から論理削除済み資産口座へ新しい設定を追加しない。

---

### 16.11 ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND

対象資産口座に利用可能資産設定履歴が1件も存在しない場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

を返却する。

ACC-002で初期設定を必ず登録する設計に対するデータ不整合として扱う。

---

### 16.12 ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID

既存履歴に以下のような整合性不正が存在する場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

を返却する。

主な対象は、

- 初期設定開始年月不一致
- `startYearMonth > endYearMonth`
- 期間重複
- 期間欠落
- 継続中設定が0件
- 継続中設定が複数件
- 最新設定が終了済み
- 同一開始年月の重複

とする。

不整合履歴へ新しい設定を追加しない。

---

### 16.13 現在設定0件

既存履歴は存在していても、

```text
end_year_month = NULL
```

の設定が0件の場合は、正常な設定変更を実行できない。

この場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

として扱う。

---

### 16.14 現在設定複数件

```text
end_year_month = NULL
```

の設定が複数存在する場合も、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

とする。

どの設定を終了させるか任意に決定しない。

---

### 16.15 ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID

Requestの

```text
startYearMonth
```

が業務上登録できない年月の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

とする。

主な対象は、

- 資産口座利用開始年月より前
- 現在設定開始年月より前
- 現在設定開始年月と同じ
- 履歴途中への挿入となる年月

とする。

---

### 16.16 現在設定と同じ開始年月

例えば、

```text
現在設定
2026-08 ～ NULL
```

に対して、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

を指定した場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

として扱う。

---

### 16.17 過去開始年月

例えば、

```text
現在設定
2026-08 ～ NULL
```

に対して、

```json
{
  "startYearMonth": "2026-05",
  "isAvailable": true
}
```

を指定した場合も、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

とする。

ACC-006は過去履歴を再構成するAPIではない。

---

### 16.18 ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE

Requestの`isAvailable`が現在設定と同じ場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

を返却する。

例えば、

```text
現在設定
isAvailable = true
```

に対して、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

を指定した場合である。

---

### 16.19 同一状態では履歴を作成しない

`ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE`の場合は、

```text
旧設定UPDATE
新設定INSERT
```

のどちらも行わない。

以下のような不要な履歴分割を防止する。

```text
2026-01 ～ 2026-07
true

2026-08 ～ NULL
true
```

---

### 16.20 ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS

同一資産口座に同じ`startYearMonth`の設定が既に存在する場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

として扱う。

これは、アプリケーション側の事前確認またはDBのUNIQUE制約違反から検出できる。

---

### 16.21 UNIQUE制約違反

並行実行等によって、

```text
asset_account_id
+
start_year_month
```

のUNIQUE制約違反が発生した場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

へ変換する。

PostgreSQLの内部例外をそのまま返却しない。

---

### 16.22 同時更新競合

同一資産口座に対して複数のACC-006が同時実行された場合、`lockForUpdate()`によって直列化する。

待機後に最新状態を再確認した結果、Request内容が最新履歴に対して不正となった場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

または、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

など、実際の最新状態に応じた業務エラーを返却する。

---

### 16.23 LOCK_TIMEOUT

DB側のロック待機が許容時間を超えた場合は、API共通方針に専用競合エラーを定義するなら、

```text
RESOURCE_BUSY
```

などへ変換してよい。

ただし、Phase1で専用コードを設けない場合は、

```text
INTERNAL_SERVER_ERROR
```

として扱う。

具体的な方針はAPI共通方針を優先する。

---

### 16.24 トランザクション失敗

以下のいずれかが失敗した場合は、トランザクション全体をロールバックする。

- 現在設定UPDATE
- 新設定INSERT
- DB制約確認
- コミット

旧設定だけを終了させた状態や、新設定だけを追加した状態を残さない。

---

### 16.25 UPDATE成功後にINSERT失敗

例えば、

```text
旧設定UPDATE
    成功

新設定INSERT
    失敗
```

となった場合でも、レスポンス上だけエラーにするのではなく、旧設定UPDATEもロールバックする。

---

### 16.26 INTERNAL_SERVER_ERROR

想定外のサーバー内部エラーが発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスへ、以下の内部情報を含めない。

- SQL
- PostgreSQL内部エラー
- SQLSTATE
- 制約名
- テーブル名
- カラム名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 16.27 主なエラー一覧

ACC-006で想定する主なエラーは、以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`未指定 |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`形式不正 |
| `400 Bad Request` | `INVALID_ASSET_ACCOUNT_ID` | `assetAccountId`形式不正 |
| `400 Bad Request` | `VALIDATION_ERROR` | Request Body形式不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 利用者不存在、または論理削除済み |
| `404 Not Found` | `ASSET_ACCOUNT_NOT_FOUND` | 資産口座不存在、他利用者所属、または論理削除済み |
| `409 Conflict` | `ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID` | 新設定開始年月が業務上不正 |
| `409 Conflict` | `ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE` | 現在設定と同じ`isAvailable`を指定 |
| `409 Conflict` | `ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS` | 同一開始年月の設定が既に存在する |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND` | 利用可能資産設定履歴が0件 |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID` | 既存履歴が不整合 |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバー内部エラー |

具体的なHTTPステータスや独自エラーコードの最終定義は、API共通方針を優先する。

---

### 16.28 エラー時の更新

以下のエラーをUPDATE前に検出した場合は、

```text
asset_account_available_settings
```

を一切変更しない。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

---

### 16.29 DB更新中のエラー

旧設定UPDATEまたは新設定INSERTの途中でエラーが発生した場合は、トランザクションをロールバックする。

そのため、エラーレスポンス返却時に

```text
旧設定だけ変更済み
```

または、

```text
新設定だけ登録済み
```

という中途半端な状態を残さない。

---

## 17. HTTPステータス

ACC-006では、処理結果に応じて以下のHTTPステータスを返却する。

| HTTPステータス | 用途 |
|---|---|
| `201 Created` | 利用可能資産設定登録成功 |
| `400 Bad Request` | 利用者コンテキスト、`assetAccountId`、またはRequest Bodyの形式不正 |
| `404 Not Found` | 利用者または資産口座が存在しない |
| `409 Conflict` | 新設定開始年月不正、状態変更なし、同一開始年月競合 |
| `500 Internal Server Error` | 利用可能資産設定履歴の不整合、または想定外のサーバー内部エラー |

具体的な独自エラーコードは、「エラーレスポンス」およびAPI共通方針に従う。

---

### 17.1 201 Created

新しい利用可能資産設定の登録に成功した場合は、

```http
201 Created
```

を返却する。

概念的には、

```text
対象資産口座確認
    ↓
既存履歴確認
    ↓
現在設定ロック
    ↓
旧設定終了
    ↓
新設定登録
    ↓
コミット
    ↓
201 Created
```

とする。

---

### 17.2 200 OKではなく201 Createdとする理由

ACC-006では、既存設定を更新するだけでなく、

```text
asset_account_available_settings
```

へ新しい履歴レコードを作成する。

そのため、新規リソース作成を表す

```text
201 Created
```

を使用する。

---

### 17.3 400 Bad Request

以下の場合は、

```text
400 Bad Request
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
```

例えば、

- `X-User-Id`未指定
- `X-User-Id`形式不正
- `assetAccountId`形式不正
- `startYearMonth`未指定
- `startYearMonth`形式不正
- 実在しない年月
- `isAvailable`未指定
- `isAvailable`がboolean以外
- 更新対象外項目の指定

などを対象とする。

---

### 17.4 404 Not Found

以下の場合は、

```text
404 Not Found
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_NOT_FOUND
ASSET_ACCOUNT_NOT_FOUND
```

`ASSET_ACCOUNT_NOT_FOUND`には、以下を含む。

- 指定された資産口座が存在しない
- 指定された資産口座が他利用者に属する
- 指定された資産口座が論理削除済み

他利用者所属や論理削除済みであることを個別のHTTPステータスで公開しない。

---

### 17.5 409 Conflict

現在のDB状態に対してRequest内容を適用できない場合は、

```text
409 Conflict
```

を返却する。

主なエラーコードは、以下とする。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

---

### 17.6 START_YEAR_MONTH_INVALID

新しい設定の`startYearMonth`が現在の履歴状態に対して登録できない場合は、

```text
409 Conflict
```

とし、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

を返却する。

主な対象は、

- 資産口座利用開始年月より前
- 現在設定開始年月より前
- 現在設定開始年月と同じ
- 履歴途中への挿入となる年月

とする。

---

### 17.7 NO_CHANGE

現在設定と同じ`isAvailable`を指定した場合は、

```text
409 Conflict
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

とする。

この場合、旧設定UPDATEも新設定INSERTも行わない。

---

### 17.8 ALREADY_EXISTS

同一資産口座に同じ`startYearMonth`の設定が既に存在する場合は、

```text
409 Conflict
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

とする。

アプリケーション側の事前確認で検出した場合も、DBのUNIQUE制約違反で検出した場合も、同じHTTPステータスおよび独自エラーコードへ変換する。

---

### 17.9 500 Internal Server Error

以下の場合は、

```text
500 Internal Server Error
```

として扱う。

主なエラーコードは、以下とする。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
INTERNAL_SERVER_ERROR
```

既存履歴の不存在や不整合は、クライアント入力によるものではなく、サーバー側の業務データ不整合として扱う。

---

### 17.10 HISTORY_NOT_FOUND

対象資産口座について、

```text
asset_account_available_settings
```

が1件も存在しない場合は、

```text
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

とする。

ACC-002で初期設定を必ず作成する設計に対するデータ不整合として扱う。

---

### 17.11 HISTORY_INVALID

既存履歴に以下のような不整合が存在する場合は、

```text
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

とする。

主な対象は、

- 初期設定開始年月不一致
- `startYearMonth > endYearMonth`
- 期間重複
- 期間欠落
- 継続中設定0件
- 継続中設定複数件
- 最新設定が終了済み
- 同一開始年月の重複

とする。

---

### 17.12 INTERNAL_SERVER_ERROR

想定外のサーバー内部エラーが発生した場合は、

```text
500 Internal Server Error
```

を返却する。

エラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

レスポンスへ内部例外やSQLなどを公開しない。

---

## 18. 冪等性

ACC-006は、新しい利用可能資産設定履歴を追加するPOST APIであるため、HTTP操作としては冪等ではない。

同一Requestを無条件に複数回実行して毎回新しい履歴を追加する設計にはしないが、POST自体を冪等操作として定義しない。

---

### 18.1 同一Requestの再実行

例えば、現在設定が

```text
2026-01 ～ NULL
isAvailable = true
```

の状態で、

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

を実行すると、

```text
2026-01 ～ 2026-07
true

2026-08 ～ NULL
false
```

となる。

その後、同一Requestを再実行した場合は、最新状態に対して再判定する。

---

### 18.2 再実行時に同じ履歴を追加しない

1回目の実行後は、現在設定が

```text
2026-08 ～ NULL
isAvailable = false
```

となっている。

そのため、同一Requestを再実行すると、

```text
startYearMonth
    = 現在設定.startYearMonth

AND

isAvailable
    = 現在設定.isAvailable
```

となる。

この場合は、新しい履歴を追加しない。

---

### 18.3 再実行時に返り得るエラー

同一Requestの再実行では、最新状態に応じて、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

または、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

となり得る。

どちらを優先するかは、業務チェック順序をLaravel実装方針で統一する。

---

### 18.4 重複防止

以下のDB制約によっても、同じ開始年月の設定を複数登録できないようにする。

```text
asset_account_id
+
start_year_month
```

UNIQUE制約違反は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

へ変換する。

---

### 18.5 同一状態の履歴増殖を防止する

現在設定と同じ`isAvailable`で新規履歴を追加しない。

これにより、

```text
2026-01 ～ 2026-07
true

2026-08 ～ 2026-09
true

2026-10 ～ NULL
true
```

のような意味のない履歴増殖を防止する。

---

### 18.6 Idempotency-Key

Phase1では、ACC-006専用の

```text
Idempotency-Key
```

は使用しない。

理由は、以下の仕組みによって重複した履歴作成を防止できるためである。

```text
業務状態再確認
+
現在設定への行ロック
+
asset_account_id / start_year_month
UNIQUE制約
```

---

### 18.7 通信結果不明時

クライアントがACC-006送信後にネットワークエラーとなり、

```text
サーバーでは成功したのか
失敗したのか
```

を判断できない場合がある。

Phase1では、即座に同じPOSTを自動再送するのではなく、

```text
ACC-005
利用可能資産設定履歴取得

または

ACC-003
資産口座詳細取得
```

によって現在状態を再取得したうえで利用者へ表示する方針とする。

---

### 18.8 自動Retryを前提としない

ACC-006は更新APIであるため、クライアント側の無条件な自動Retryを前提としない。

TanStack Query等では、Mutationの自動Retryを原則無効とする。

---

### 18.9 排他制御と冪等性は別とする

現在設定への`lockForUpdate()`は、同時実行による履歴破壊を防止するための排他制御である。

これは、HTTPリクエスト自体を冪等にする仕組みではない。

概念的には、

```text
lockForUpdate()
    → 同時更新制御

UNIQUE制約
    → 重複データ防止

業務チェック
    → 不要な履歴追加防止

Idempotency
    → 別概念
```

として区別する。

---

## 19. キャッシュ

Phase1では、ACC-006専用のサーバー側アプリケーションキャッシュを使用しない。

ACC-006成功後は、PostgreSQL上の

```text
asset_account_available_settings
```

が最新状態となる。

---

### 19.1 更新API自体をキャッシュしない

ACC-006は状態変更を行うPOST APIであるため、

```http
POST /api/v1/asset-accounts/{assetAccountId}/available-settings
```

のレスポンスをHTTPキャッシュによって後続Requestへ再利用することを前提としない。

---

### 19.2 ACC-005への影響

ACC-006成功時は、利用可能資産設定履歴が必ず変化する。

概念的には、

```text
ACC-006
    ↓
旧設定.endYearMonth更新
+
新設定追加
    ↓
ACC-005の取得結果が変化
```

となる。

そのため、React側でACC-005をQuery Cacheしている場合は、ACC-006成功後に対象資産口座の履歴Queryを無効化する。

---

### 19.3 ACC-003への影響

ACC-003では、現在年月に有効な

```text
isAvailable
```

を返却する。

ACC-006によって現在年月に適用される設定が変化した場合は、ACC-003のレスポンスも変化する。

そのため、ACC-006成功後は、対象資産口座の詳細Queryも無効化する。

---

### 19.4 ACC-005 Query Cacheの無効化

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.availableSettings(
      assetAccountId,
    ),
});
```

これにより、ACC-005を再実行して最新履歴を取得する。

---

### 19.5 ACC-003 Query Cacheの無効化

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

これにより、最新の`isAvailable`を再取得する。

---

### 19.6 ACC-001への影響

ACC-001 資産口座一覧取得が`isAvailable`を返却する設計の場合は、ACC-006成功後に資産口座一覧Queryも無効化する。

概念的には、

```text
ACC-006成功
    ↓
ACC-001
ACC-003
ACC-005
```

のうち、`isAvailable`を含むQueryを再取得対象とする。

ACC-001が`isAvailable`を返却しない設計であれば、ACC-006だけを理由として必ず無効化する必要はない。

---

### 19.7 将来開始設定の場合

ACC-006で現在より将来の

```text
startYearMonth
```

を登録した場合、ACC-005の履歴は即座に変化する。

一方、ACC-003の現在の

```text
isAvailable
```

はまだ変化しない場合がある。

例えば、

```text
現在年月
2026-08

現在設定
2026-01 ～ 2026-09
true

新設定
2026-10 ～ NULL
false
```

の場合、2026-08時点の`isAvailable`は引き続き`true`となる。

---

### 19.8 将来設定でもACC-003を再取得してよい

将来開始設定の場合でも、React側で適用年月判定を独自に行うより、

```text
ACC-006成功
    ↓
ACC-003再取得
```

としてよい。

サーバー側の現在年月判定を正とし、フロントエンドに不要な業務ロジックを持たせない。

---

### 19.9 ACC-005を必ず再取得する

ACC-006成功時は、新しい履歴が追加され、旧設定の終了年月も変更される。

そのため、ACC-005のキャッシュは必ず古くなる。

Phase1では、

```text
ACC-006成功
    ↓
ACC-005 invalidate
    ↓
再取得
```

を基本とする。

---

### 19.10 成功レスポンスだけで履歴Cacheを再構築しない

ACC-006の成功レスポンスでは、新しく作成した設定だけを返却する。

旧設定については、レスポンスへ含めない。

そのため、React側で

```text
新設定を配列へ追加
+
旧設定.endYearMonthを手動計算
```

して履歴Cacheを再構築することは、Phase1では基本としない。

---

### 19.11 サーバー状態を正とする

利用可能資産設定履歴の最終状態は、PostgreSQL上のデータを正とする。

ACC-006成功後は、ACC-005を再取得し、

```text
旧設定のendYearMonth
新設定ID
新設定startYearMonth
新設定isAvailable
```

をサーバー確定状態から取得する。

---

### 19.12 Optimistic Update

Phase1では、ACC-006に対するOptimistic Updateを必須としない。

ACC-006では、

- 既存履歴整合性確認
- 現在設定確認
- 開始年月判定
- 同一状態判定
- 行ロック
- UNIQUE制約
- トランザクション

など、サーバー側で最終判断する条件が多いためである。

---

### 19.13 Mutation成功後に更新する

React側では、

```text
ACC-006成功
    ↓
Query invalidate
    ↓
最新状態再取得
```

を基本とする。

サーバー成功前に履歴表示を先に確定しない。

---

### 19.14 サーバー側キャッシュ

Phase1では、以下のようなACC-006専用キャッシュを使用しない。

```text
Redis
Laravel Cache
Application Memory Cache
```

利用可能資産設定変更は高頻度処理ではなく、DB状態との整合性を優先する。

---

### 19.15 ACC-004への影響

ACC-006では、

```text
asset_accounts.name
asset_accounts.asset_type
```

を変更しない。

そのため、ACC-004で編集する通常属性自体はACC-006によって変化しない。

ただし、ACC-003相当の詳細表示を同一Queryで管理している場合は、`isAvailable`変更のため詳細Queryを無効化する。

---

### 19.16 月末資産関連Cache

ACC-006では、以下のテーブルを直接更新しない。

```text
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
```

そのため、月末資産データ自体のQuery Cacheを必ず無効化する必要はない。

ただし、表示時に対象年月の利用可能資産区分を組み合わせているQueryがある場合は、そのQuery設計に応じて失効を検討する。

---

### 19.17 目的達成判定関連Cache

ACC-006は、

```text
assessment_histories
```

を更新しない。

既存の目的達成判定履歴も変更しない。

ただし、将来新たに判定を実行する際には、対象年月の利用可能資産設定が判定条件へ影響する可能性がある。

過去の判定履歴CacheをACC-006成功だけで書き換えない。

---

### 19.18 利用者境界とQuery Cache

ACC-006成功後にQuery Cacheを無効化する場合も、利用者境界を考慮する。

Phase1では`X-User-Id`で操作対象利用者を切り替えるため、前利用者のQuery Cacheを別利用者へ誤表示しないようにする。

---

### 19.19 Query KeyへuserIdを含める方式

必要に応じて、

```typescript
[
  'assetAccounts',
  userId,
  assetAccountId,
  'availableSettings',
]
```

のように、Query Keyへ`userId`を含めてよい。

または、利用者切替時に資産口座関連Queryを一括無効化する。

正式な方式は、React共通設計に従う。

---

### 19.20 HTTPキャッシュ

ACC-006自体はHTTPキャッシュ対象としない。

また、ACC-006成功後にACC-003やACC-005の古いGETレスポンスが長期間返却されないよう、将来的にHTTPキャッシュを導入する場合は、関連リソースの失効設計を合わせて行う。

---

### 19.21 ETag等

Phase1では、

```text
ETag
Last-Modified
If-Match
If-None-Match
```

などのHTTP条件付きRequestをACC-006へ導入しない。

同時更新制御は、現在設定への`lockForUpdate()`とDB制約によって行う。

---

### 19.22 キャッシュ方針まとめ

ACC-006成功後の基本的なキャッシュ更新方針は、以下とする。

```text
ACC-006成功
    ↓
ACC-005
利用可能資産設定履歴
    → invalidate

ACC-003
資産口座詳細
    → invalidate

ACC-001
資産口座一覧
    → isAvailableを含む場合はinvalidate
```

Phase1では、クライアント側で履歴を複雑に手動再構築せず、サーバーからの再取得を基本とする。

---

## 20. 関連テーブル

ACC-006では、指定された資産口座の利用可能資産設定を変更するため、以下のテーブルを使用する。

| テーブル                               | 用途                         |  更新 |
| ---------------------------------- | -------------------------- | :-: |
| `users`                            | 操作対象利用者の確認                 |  ×  |
| `asset_accounts`                   | 対象資産口座の取得、利用者境界確認、利用開始年月確認 |  ×  |
| `asset_account_available_settings` | 既存履歴取得、現在設定ロック、旧設定終了、新設定登録 |  ○  |

ACC-006で直接更新する業務テーブルは、

```text
asset_account_available_settings
```

のみとする。

---

### 20.1 users

`X-User-Id`で指定された操作対象利用者の存在確認に使用する。

概念的な条件は、以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者コンテキストの確認は、API共通Middlewareで行う。

ACC-006では、`users`を更新しない。

---

### 20.2 asset_accounts

指定された`assetAccountId`に対応する資産口座を取得し、利用者境界および利用開始年月を確認するために使用する。

主に以下のカラムを使用する。

| カラム                | 用途                      |
| ------------------ | ----------------------- |
| `id`               | `assetAccountId`との照合    |
| `user_id`          | 操作対象利用者との利用者境界確認        |
| `start_year_month` | 新設定開始年月および履歴初期年月との整合性確認 |
| `deleted_at`       | 利用状態の判定                 |

---

### 20.3 対象資産口座取得

対象資産口座は、概念的に以下の条件で取得する。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

他利用者に属する資産口座、論理削除済み資産口座は設定変更対象として取得しない。

---

### 20.4 asset_accountsは更新しない

ACC-006では、以下の資産口座属性を変更しない。

```text
name
asset_type
balance_recording_unit
start_year_month
deleted_at
```

利用可能資産区分は、

```text
asset_account_available_settings
```

によって履歴管理する。

---

### 20.5 asset_account_available_settings

ACC-006の主たる更新対象テーブルとする。

主に以下のカラムを使用する。

| カラム                | 用途          |
| ------------------ | ----------- |
| `id`               | 利用可能資産設定ID  |
| `asset_account_id` | 対象資産口座との関連  |
| `start_year_month` | 設定適用開始年月    |
| `end_year_month`   | 設定適用終了年月    |
| `is_available`     | 利用可能資産区分    |
| `created_at`       | 新設定登録日時     |
| `updated_at`       | 旧設定終了時の更新日時 |

---

### 20.6 既存履歴取得

対象資産口座に紐づく既存履歴を、

```text
asset_account_available_settings.asset_account_id
    = assetAccountId
```

で取得する。

履歴整合性確認では、必要に応じて

```text
start_year_month ASC
```

で取得する。

---

### 20.7 現在設定取得

現在設定は、

```text
asset_account_id
    = assetAccountId

AND

end_year_month
    IS NULL
```

の条件で取得する。

正常状態では、該当する設定が1件だけ存在する。

---

### 20.8 現在設定をロックする

ACC-006では、現在設定をトランザクション内で取得し、

```php
lockForUpdate()
```

を使用する。

これにより、同一資産口座への設定変更を直列化する。

---

### 20.9 旧設定の更新

現在設定の

```text
end_year_month
```

を、

```text
request.startYearMonth
の前月
```

へ更新する。

例えば、

```text
request.startYearMonth
    = 2026-08
```

の場合、

```text
end_year_month
    = 2026-07
```

とする。

---

### 20.10 新設定の登録

新しい設定は、以下の内容で登録する。

```text
asset_account_id
    = assetAccountId

start_year_month
    = request.startYearMonth

end_year_month
    = NULL

is_available
    = request.isAvailable
```

新設定登録時に、旧設定の値をコピーしてから一部変更する方式にはしない。

---

### 20.11 asset_account_id

新設定の

```text
asset_account_id
```

は、URLで指定され、利用者境界確認済みの`assetAccountId`を使用する。

Request Bodyから指定された値は使用しない。

---

### 20.12 start_year_month

新設定の

```text
start_year_month
```

には、Requestの

```text
startYearMonth
```

を保存する。

形式は、

```text
YYYY-MM
```

とする。

---

### 20.13 end_year_month

新設定は継続中設定として登録するため、

```text
end_year_month
    = NULL
```

とする。

クライアント指定値を保存しない。

---

### 20.14 is_available

Requestの

```text
isAvailable
```

を、

```text
asset_account_available_settings.is_available
```

へ保存する。

APIではboolean、DBではテーブル定義に従ったboolean表現を使用する。

---

### 20.15 一意性

同一資産口座内では、

```text
asset_account_id
+
start_year_month
```

を一意とする。

これにより、以下のような状態を防止する。

```text
assetAccountId = 10

2026-08
true

2026-08
false
```

---

### 20.16 現在設定の一意性

業務ルール上は、同一資産口座について

```text
end_year_month IS NULL
```

の設定は1件のみとする。

PostgreSQLを使用する場合は、必要に応じて

```text
asset_account_id
WHERE end_year_month IS NULL
```

に対する部分UNIQUEインデックスを検討してよい。

これにより、アプリケーション側の`lockForUpdate()`に加えて、DB側でも継続中設定複数件を防止できる。

---

### 20.17 履歴期間の連続性

正常な履歴では、前設定と次設定について、

```text
前設定.end_year_month
    の翌月
        =
次設定.start_year_month
```

となる。

例えば、

```text
2026-01 ～ 2026-06
2026-07 ～ NULL
```

は正常とする。

---

### 20.18 初期設定との整合性

最古の利用可能資産設定の

```text
start_year_month
```

は、

```text
asset_accounts.start_year_month
```

と一致することを正常状態とする。

概念的には、

```text
asset_accounts.start_year_month
    =
MIN(
    asset_account_available_settings.start_year_month
)
```

となる。

---

### 20.19 更新しない関連テーブル

ACC-006では、以下のテーブルを更新しない。

* `users`
* `asset_accounts`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

既存の月末資産データや目的達成判定履歴へ副作用を与えない。

---

## 21. 関連する機能要件

ACC-006は、利用可能資産区分の履歴管理および変更に関する機能要件と対応する。

主な関連要件は、以下とする。

* 資産口座管理

  * 資産口座ごとに利用可能資産区分を設定できる
  * 利用可能資産区分を変更できる
  * 利用可能資産区分の変更履歴を保持する
* 利用可能資産設定

  * 資産口座登録時に初期設定を作成する
  * 設定変更時は旧設定を終了する
  * 新設定を履歴として追加する
  * 同一状態の不要な履歴を追加しない
  * 設定期間を重複させない
  * 設定期間に欠落を作らない
  * 継続中設定は1件のみとする
* 利用者境界

  * 操作対象利用者に帰属する資産口座だけを変更できる
  * 他利用者の資産口座は対象不存在として扱う
* 論理削除

  * 論理削除済み資産口座へ新しい設定を追加しない

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

## 22. インデックス

ACC-006では、主に以下の検索条件を使用する。

```text
asset_accounts.id
```

```text
asset_account_available_settings.asset_account_id
```

```text
asset_account_available_settings.start_year_month
```

```text
asset_account_available_settings.end_year_month
```

---

### 22.1 asset_accounts.id

`asset_accounts.id`は主キーであるため、主キーインデックスを使用する。

ACC-006専用として追加インデックスを作成しない。

---

### 22.2 asset_account_id

利用可能資産設定履歴は、

```text
asset_account_id
```

を条件として取得する。

FK列としてインデックスを設定する、または既存インデックスを利用する。

---

### 22.3 asset_account_id + start_year_month

同一資産口座内で開始年月を一意とするため、

```text
asset_account_id
+
start_year_month
```

にUNIQUE制約を設定する。

このインデックスは、

* 履歴取得
* 開始年月重複確認
* 並行登録時の競合防止

にも利用できる。

---

### 22.4 継続中設定の部分UNIQUEインデックス

PostgreSQLでは、必要に応じて

```text
asset_account_id
WHERE end_year_month IS NULL
```

の部分UNIQUEインデックスを検討する。

概念例：

```sql
CREATE UNIQUE INDEX
    uq_asset_account_available_settings_current
ON
    asset_account_available_settings (
        asset_account_id
    )
WHERE
    end_year_month IS NULL;
```

これにより、同一資産口座に継続中設定が複数存在する状態をDB側でも防止できる。

---

### 22.5 lockForUpdateとDB制約を併用する

ACC-006では、

```text
lockForUpdate
    → 同時更新を直列化

UNIQUE制約
    → 重複開始年月防止

部分UNIQUE制約
    → 継続中設定複数件防止
```

という複数層の整合性保証を組み合わせてよい。

---

### 22.6 不要なインデックスを追加しない

ACC-006だけを理由として、既存の

* 主キー
* FKインデックス
* UNIQUE制約

と役割が重複するインデックスを追加しない。

実際のSQLと実行計画を確認したうえで必要性を判断する。

---

## 23. 性能

ACC-006は、単一資産口座の利用可能資産設定履歴を変更するAPIである。

通常は、

```text
資産口座1件取得
    ↓
設定履歴取得
    ↓
現在設定1件ロック
    ↓
旧設定1件UPDATE
    ↓
新設定1件INSERT
```

で完結する。

---

### 23.1 資産口座一覧を取得しない

操作対象利用者の資産口座一覧を全件取得してから対象資産口座を探さない。

```text
assetAccountId
+
userId
```

を条件として直接1件取得する。

---

### 23.2 他資産口座の履歴を取得しない

利用可能資産設定履歴は、

```text
asset_account_id
    = assetAccountId
```

で絞り込む。

全資産口座分の設定履歴を取得してPHP側で絞り込まない。

---

### 23.3 履歴件数

利用可能資産設定は年月単位の変更履歴であるため、1資産口座あたりの件数は比較的少ないことを想定する。

Phase1では、履歴全件を取得して整合性確認してよい。

---

### 23.4 履歴整合性確認

履歴整合性確認は、取得済み履歴を開始年月順に1回走査する程度とする。

各履歴レコードごとに追加SQLを発行しない。

概念的には、

```text
履歴SELECT 1回
    ↓
PHP側で前後比較
```

とする。

---

### 23.5 現在設定取得

排他制御時は、現在設定のみを

```text
end_year_month IS NULL
```

で取得し、

```php
lockForUpdate()
```

する。

必要以上の履歴行をロックしない。

---

### 23.6 ロック範囲を最小化する

ACC-006では、対象資産口座の現在設定行だけを原則ロックする。

履歴全件や同一利用者の全資産口座をロックしない。

---

### 23.7 トランザクションを短く保つ

ロック取得後は、

```text
業務条件再確認
旧設定UPDATE
新設定INSERT
```

を速やかに実行する。

以下をトランザクション中に行わない。

* 外部API呼び出し
* メール送信
* ファイル処理
* 不要なEager Load
* 大量データ取得

---

### 23.8 N+1問題

ACC-006は単一資産口座を対象とし、履歴を一括取得するため、通常の意味でのN+1問題は発生しない。

---

## 24. セキュリティ

ACC-006では、利用者境界、更新可能項目の限定、履歴操作のサーバー側制御を重要なセキュリティ要件とする。

---

### 24.1 assetAccountIdだけで取得しない

以下のような対象取得は避ける。

```php
AssetAccount::find(
    $assetAccountId,
);
```

必ず、

```text
assetAccountId
+
userId
+
deleted_at IS NULL
```

を条件とする。

---

### 24.2 他利用者の設定を変更しない

他利用者に属する資産口座IDが指定された場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

他利用者の利用可能資産設定を検索・ロック・更新しない。

---

### 24.3 settingIdをクライアントに指定させない

現在設定の

```text
id
```

をRequestから指定させない。

現在設定は、サーバー側で

```text
asset_account_id
+
end_year_month IS NULL
```

から特定する。

これにより、任意の過去履歴を直接更新することを防止する。

---

### 24.4 Mass Assignmentを避ける

以下のようにRequest全体をそのままModelへ渡さない。

```php
$setting->update(
    $request->all(),
);
```

更新・登録対象を明示的に限定する。

---

### 24.5 endYearMonthをクライアント指定させない

旧設定の終了年月は、

```text
新設定開始年月の前月
```

としてサーバー側で計算する。

Requestから

```text
endYearMonth
```

を指定させない。

---

### 24.6 assetAccountIdをRequest Bodyから受け付けない

対象資産口座はURLだけから取得する。

Request Body中の`assetAccountId`によって対象を変更できないようにする。

---

### 24.7 userIdをRequest Bodyから受け付けない

利用者IDは`X-User-Id`からのみ取得する。

Request Body中の`userId`によって所有者境界を変更できないようにする。

---

### 24.8 過去履歴を直接編集させない

ACC-006では、利用可能資産設定IDを指定した任意UPDATEを許可しない。

これにより、クライアント側から

```text
過去履歴期間改ざん
```

を行えないようにする。

---

### 24.9 SQLインジェクション対策

以下の入力値を検索条件へ使用する場合は、EloquentまたはQuery Builderのバインド機構を使用する。

* `assetAccountId`
* 利用者ID
* `startYearMonth`

SQL文字列へ入力値を直接連結しない。

---

### 24.10 内部情報を公開しない

エラー時に、以下をレスポンスへ含めない。

* SQL
* SQLSTATE
* PostgreSQL内部エラー
* 制約名
* テーブル名
* カラム名
* Laravel内部例外
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

---

## 25. ログ・監視

ACC-006では、API共通ログ方針に従って設定変更結果を記録する。

---

### 25.1 ログコンテキスト

必要に応じて、以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assetAccountId
httpStatus
errorCode
```

`apiId`は、

```text
ACC-006
```

とする。

---

### 25.2 正常時

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = ACC-006
assetAccountId
newSettingId
httpStatus = 201
```

設定変更の追跡が必要な場合は、

```text
startYearMonth
```

を内部ログへ記録してもよい。

---

### 25.3 isAvailableのログ

`isAvailable`は機密情報ではないが、通常ログへ不要に大量出力しない。

設定変更調査上必要であれば、

```text
previousIsAvailable
newIsAvailable
```

を構造化ログとして記録してよい。

---

### 25.4 異常時

異常時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId
assetAccountId
errorCode
httpStatus
```

---

### 25.5 履歴不整合

以下のエラーでは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

調査可能な情報を内部ログへ記録する。

必要に応じて、

```text
assetAccountId
historyCount
invalidReason
```

を記録してよい。

---

### 25.6 同時更新競合

以下のような競合が発生した場合は、必要に応じて内部ログへ記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

ロック待機時間や競合回数を監視対象にしてもよい。

---

### 25.7 DB制約違反

UNIQUE制約違反を業務エラーへ変換した場合、内部ログでは原因調査のためにSQLSTATEや制約識別情報を記録してよい。

ただし、APIレスポンスへは公開しない。

---

### 25.8 トランザクションロールバック

旧設定UPDATE後に新設定INSERTが失敗した場合など、トランザクションがロールバックされたことを障害調査可能な状態にする。

必要に応じて、

```text
transactionRolledBack = true
```

相当の内部ログを記録してよい。

---

### 25.9 requestId

クライアントへ返却する`requestId`とサーバーログを関連付けられるようにする。

概念的には、

```text
requestId
    ↓
ACC-006ログ
    ↓
トランザクション
    ↓
競合・不整合原因調査
```

を可能とする。

---

## 26. テスト観点

ACC-006では、入力値検証だけでなく、

* 利用者境界
* 履歴整合性
* 開始年月
* 状態変更有無
* 旧設定更新
* 新設定登録
* トランザクション
* 排他制御
* UNIQUE制約
* 副作用範囲

を重点的に確認する。

---

### 26.1 正常系：trueからfalse

現在設定を、

```text
2026-01 ～ NULL
is_available = true
```

とする。

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

実行後、

```text
旧設定
2026-01 ～ 2026-07
true

新設定
2026-08 ～ NULL
false
```

となることを確認する。

---

### 26.2 正常系：falseからtrue

現在設定を、

```text
2026-01 ～ NULL
false
```

とする。

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

実行後、

```text
2026-01 ～ 2026-07
false

2026-08 ～ NULL
true
```

となることを確認する。

---

### 26.3 成功レスポンス

正常時に、

```text
201 Created
```

となることを確認する。

概念的なレスポンス：

```json
{
  "data": {
    "id": "15",
    "startYearMonth": "2026-08",
    "endYearMonth": null,
    "isAvailable": false
  }
}
```

---

### 26.4 idの型

新規設定IDがDB上で`bigint`でも、

```json
{
  "id": "15"
}
```

のようにstringとして返却されることを確認する。

---

### 26.5 startYearMonth未指定

以下を送信する。

```json
{
  "isAvailable": false
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.6 startYearMonth = null

以下を送信する。

```json
{
  "startYearMonth": null,
  "isAvailable": false
}
```

バリデーションエラーとなり、DB更新されないことを確認する。

---

### 26.7 startYearMonth形式不正

以下を確認する。

```text
2026-8
2026/08
202608
2026-00
2026-13
abc
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.8 isAvailable未指定

以下を送信する。

```json
{
  "startYearMonth": "2026-08"
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.9 isAvailable = null

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": null
}
```

バリデーションエラーとなること。

---

### 26.10 isAvailableが文字列

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": "false"
}
```

期待結果：

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.11 isAvailableが数値

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": 0
}
```

boolean以外として拒否されることを確認する。

---

### 26.12 X-User-Id未指定

`X-User-Id`を指定しない。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

となること。

---

### 26.13 X-User-Id形式不正

例えば、

```http
X-User-Id: abc
```

を指定する。

期待結果：

```text
400 Bad Request
INVALID_USER_ID
```

となること。

---

### 26.14 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 26.15 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 26.16 assetAccountId形式不正

以下を確認する。

```text
0
-1
abc
1.5
1e3
10abc
```

期待結果：

```text
400 Bad Request
INVALID_ASSET_ACCOUNT_ID
```

となること。

---

### 26.17 資産口座不存在

存在しない`assetAccountId`を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 26.18 他利用者の資産口座

User AとしてUser Bの資産口座を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの`asset_account_available_settings`が一切変更されないことを確認する。

---

### 26.19 論理削除済み資産口座

対象資産口座を

```text
deleted_at IS NOT NULL
```

とする。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

新しい設定が追加されないこと。

---

### 26.20 履歴0件

利用中資産口座に対して利用可能資産設定履歴を0件とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

となること。

設定を自動作成しないこと。

---

### 26.21 初期開始年月不整合

以下の状態を用意する。

```text
asset_accounts.start_year_month
    = 2026-01

最古設定.start_year_month
    = 2026-02
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 26.22 既存履歴の期間重複

以下を用意する。

```text
2026-01 ～ 2026-08
true

2026-08 ～ NULL
false
```

ACC-006を実行しても新設定を追加せず、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 26.23 既存履歴の期間欠落

以下を用意する。

```text
2026-01 ～ 2026-05
true

2026-07 ～ NULL
false
```

履歴不整合となること。

---

### 26.24 継続中設定0件

すべての設定に`end_year_month`が存在する状態を用意する。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 26.25 継続中設定複数件

以下を用意する。

```text
2026-01 ～ NULL
true

2026-08 ～ NULL
false
```

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

どちらか一方を任意に更新しないこと。

---

### 26.26 startYearMonthが資産口座利用開始年月より前

例えば、

```text
assetAccount.startYearMonth
    = 2026-08
```

に対して、

```json
{
  "startYearMonth": "2026-07",
  "isAvailable": false
}
```

を送信する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

となること。

---

### 26.27 startYearMonthが現在設定開始年月と同じ

現在設定：

```text
2026-08 ～ NULL
true
```

Request：

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

となること。

---

### 26.28 startYearMonthが現在設定より前

現在設定：

```text
2026-08 ～ NULL
false
```

Request：

```json
{
  "startYearMonth": "2026-05",
  "isAvailable": true
}
```

履歴途中への挿入として拒否されること。

---

### 26.29 1か月後から変更

現在設定：

```text
2026-08 ～ NULL
true
```

Request：

```json
{
  "startYearMonth": "2026-09",
  "isAvailable": false
}
```

正常に、

```text
2026-08 ～ 2026-08
true

2026-09 ～ NULL
false
```

となること。

---

### 26.30 年跨ぎ

現在設定：

```text
2026-01 ～ NULL
true
```

Request：

```json
{
  "startYearMonth": "2027-01",
  "isAvailable": false
}
```

旧設定終了年月が

```text
2026-12
```

となること。

---

### 26.31 同じisAvailable

現在設定：

```text
2026-01 ～ NULL
true
```

Request：

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": true
}
```

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

となること。

以下を確認する。

* 旧設定を更新しない
* 新設定を追加しない
* 履歴件数が変化しない

---

### 26.32 同一startYearMonth重複

同一資産口座に

```text
start_year_month = 2026-08
```

の設定が既に存在する状態で、同じ開始年月の登録を試みる。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

または、より前の業務チェックで`START_YEAR_MONTH_INVALID`となる場合は、チェック順序に従った結果となること。

---

### 26.33 UNIQUE制約

DBレベルで、

```text
asset_account_id
+
start_year_month
```

が重複できないことを確認する。

並行実行などで制約違反が発生した場合は、PostgreSQL例外がAPIへそのまま露出しないこと。

---

### 26.34 継続中設定部分UNIQUE制約

部分UNIQUEインデックスを採用する場合は、同一資産口座に

```text
end_year_month IS NULL
```

の設定を2件作成できないことを確認する。

---

### 26.35 旧設定UPDATE

正常系で、現在設定の

```text
end_year_month
```

だけが新設定開始年月の前月へ更新されることを確認する。

以下は変更されないこと。

```text
start_year_month
is_available
asset_account_id
```

---

### 26.36 新設定INSERT

正常系で、新設定が以下の値で登録されることを確認する。

```text
asset_account_id
    = 対象資産口座ID

start_year_month
    = Request値

end_year_month
    = NULL

is_available
    = Request値
```

---

### 26.37 旧設定更新後に新設定INSERT失敗

新設定INSERTで意図的に例外を発生させる。

以下を確認する。

* APIはエラーとなる
* トランザクションがロールバックされる
* 旧設定の`end_year_month`が元の`NULL`へ戻る
* 新設定が存在しない

---

### 26.38 旧設定UPDATE失敗

旧設定UPDATEで例外を発生させる。

以下を確認する。

* 新設定をINSERTしない
* トランザクションがロールバックされる
* 履歴状態が実行前と同じ

---

### 26.39 Transaction全体

正常時は、

```text
旧設定UPDATE
+
新設定INSERT
```

の両方がコミットされること。

異常時は、両方とも確定しないことを確認する。

---

### 26.40 同時実行

同一資産口座に対して複数のACC-006を並行実行する。

以下を確認する。

* 現在設定への`lockForUpdate()`が機能する
* 同時に同じ旧設定を更新しない
* 継続中設定が複数件にならない
* 履歴期間が重複しない
* 2件目は最新状態で業務条件を再評価する

---

### 26.41 同時に異なる開始年月

例えば、

```text
Request A
2026-08 / false

Request B
2026-09 / false
```

を並行実行する。

Request A成功後、Request Bが最新状態を基準に再判定されることを確認する。

不要な同一状態履歴が作成されないこと。

---

### 26.42 同時に同じ開始年月

以下を並行実行する。

```text
Request A
2026-08 / false

Request B
2026-08 / false
```

以下を確認する。

* 同じ開始年月の設定が2件作成されない
* 一方のみ成功すること
* 競合側が業務エラーへ変換されること
* DB制約違反が外部へ露出しないこと

---

### 26.43 他の関連テーブル非更新

ACC-006実行前後で、以下が変更されないことを確認する。

* `asset_accounts`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

---

### 26.44 過去の月末資産非更新

利用可能資産設定変更によって、既存の

```text
month_end_asset_balances
month_end_holding_values
month_end_asset_snapshots
```

が更新されないことを確認する。

---

### 26.45 過去判定履歴非更新

ACC-006実行によって、

```text
assessment_histories
```

の既存レコードが変更されないことを確認する。

---

### 26.46 更新対象外項目

以下のようなRequestを送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false,
  "endYearMonth": "2026-12"
}
```

未定義項目を拒否する共通方針の場合は、

```text
400 Bad Request
VALIDATION_ERROR
```

となること。

---

### 26.47 currentSettingIdを送信

以下を送信する。

```json
{
  "currentSettingId": "10",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

Request指定のIDによって更新対象を変更できないことを確認する。

---

### 26.48 userIdを送信

以下を送信する。

```json
{
  "userId": "999",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

`user_id`が変更されず、別利用者の資産口座を操作できないことを確認する。

---

### 26.49 assetAccountIdをBodyへ送信

以下を送信する。

```json
{
  "assetAccountId": "999",
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

URLの`assetAccountId`以外を対象決定に使用しないことを確認する。

---

### 26.50 正常レスポンス契約

正常時に、以下の形式となることを確認する。

```json
{
  "data": {
    "id": "15",
    "startYearMonth": "2026-08",
    "endYearMonth": null,
    "isAvailable": false
  }
}
```

以下を確認する。

* HTTPステータスが`201 Created`
* `id`がstring
* `startYearMonth`が`YYYY-MM`
* `endYearMonth = null`
* `isAvailable`がboolean

---

### 26.51 返却しない情報

正常レスポンスへ、以下が含まれないことを確認する。

* `assetAccountId`
* `asset_account_id`
* `userId`
* `user_id`
* `createdAt`
* `created_at`
* `updatedAt`
* `updated_at`
* 旧設定情報
* 資産口座名
* 資産種別
* 残高記録単位
* `isEnabled`

---

### 26.52 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
VALIDATION_ERROR
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
INTERNAL_SERVER_ERROR
```

以下も確認する。

* `error.code`が設定されること
* `error.message`が設定されること
* 必要に応じて`error.details`が設定されること
* `requestId`が設定されること
* SQLが含まれないこと
* SQLSTATEが含まれないこと
* PostgreSQL内部エラーが含まれないこと
* 制約名が含まれないこと
* スタックトレースが含まれないこと

---

### 26.53 エラー時の副作用

業務エラー発生時に、

```text
asset_account_available_settings
```

が中途半端に変更されないことを確認する。

特に、

```text
旧設定終了済み
+
新設定なし
```

や、

```text
旧設定継続中
+
新設定追加済み
```

という状態を残さないこと。

---

### 26.54 INTERNAL_SERVER_ERROR

更新処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

* トランザクションがロールバックされること
* 内部情報がレスポンスへ公開されないこと
* サーバーログに調査情報が記録されること
* レスポンスの`requestId`からログを追跡できること

---

## 27. Laravel実装方針

ACC-006では、Action、FormRequest、UseCase、Query、Repository、Validator、DTO、API Resource、Responderを分離して実装する。

概念的な構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
FormRequest
    ↓
Action
    ↓
UseCase
    ├─ AssetAccountQuery
    ├─ AssetAccountAvailableSettingQuery
    ├─ AvailableSettingHistoryValidator
    └─ AssetAccountAvailableSettingRepository
    ↓
Create Result DTO
    ↓
API Resource
    ↓
Responder
```

ACC-006は、

```text
旧設定.end_year_month更新
+
新設定INSERT
```

という複数レコード更新を1つの業務操作として扱うため、UseCase全体をトランザクション境界とする。

---

### 27.1 Route

ACC-006は、以下のルートとして定義する。

概念例：

```php
Route::post(
    '/api/v1/asset-accounts/{assetAccountId}/available-settings',
    CreateAssetAccountAvailableSettingAction::class,
);
```

ACC-005とは同じURLを使用し、HTTPメソッドによって責務を分離する。

```text
GET
/api/v1/asset-accounts/{assetAccountId}/available-settings
    → ACC-005
      利用可能資産設定履歴取得

POST
/api/v1/asset-accounts/{assetAccountId}/available-settings
    → ACC-006
      利用可能資産設定登録
```

---

### 27.2 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、

```text
X-User-Id取得
    ↓
必須チェック
    ↓
形式チェック
    ↓
users存在確認
    ↓
UserContext設定
    ↓
FormRequest
    ↓
Action
```

とする。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 27.3 FormRequest

ACC-006では、リクエストボディの入力値検証に専用FormRequestを使用する。

概念例：

```php
final class CreateAssetAccountAvailableSettingRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'startYearMonth' => [
                'required',
                'string',
                'date_format:Y-m',
            ],

            'isAvailable' => [
                'required',
                'boolean',
            ],
        ];
    }
}
```

ただし、Laravelの`boolean`ルールで許容される値がAPI契約より広い場合は、JSON booleanのみを許容する専用Ruleまたは追加検証を使用する。

---

### 27.4 JSON booleanを厳密に扱う

ACC-006では、

```text
true
false
```

だけを`isAvailable`として許可する。

以下は受け付けない。

```text
"true"
```

```text
"false"
```

```text
1
```

```text
0
```

Laravel標準の`boolean`ルールがこれらを許容する場合は、例えば専用Ruleを使用する。

概念例：

```php
final class StrictBoolean implements ValidationRule
{
    public function validate(
        string $attribute,
        mixed $value,
        Closure $fail,
    ): void {
        if (! is_bool($value)) {
            $fail(
                'booleanで指定してください。',
            );
        }
    }
}
```

---

### 27.5 startYearMonthの形式検証

`startYearMonth`は、厳密な

```text
YYYY-MM
```

形式として検証する。

例えば、

```text
2026-08
```

は正常とする。

以下は不正とする。

```text
2026-8
2026/08
202608
2026-00
2026-13
```

見た目だけでなく、実在する年月であることも確認する。

---

### 27.6 FormRequestで行うこと

FormRequestでは、HTTP入力として判断できる以下を検証する。

```text
startYearMonth
isAvailable
```

主に以下とする。

- 必須
- NULL禁止
- 型
- `YYYY-MM`形式
- 実在年月
- JSON boolean
- 未定義項目

---

### 27.7 FormRequestで行わないこと

FormRequestでは、以下のDB状態に依存する業務ルールを扱わない。

- 資産口座存在確認
- 利用者境界確認
- 論理削除判定
- 既存履歴0件判定
- 履歴整合性判定
- 現在設定特定
- 開始年月の業務妥当性
- 現在設定との`isAvailable`比較
- 行ロック
- DB更新

これらは、UseCase、Query、Validator、Repositoryで扱う。

---

### 27.8 更新対象外項目

ACC-006では、以下をRequest項目として定義しない。

```text
id
userId
assetAccountId
settingId
currentSettingId
endYearMonth
createdAt
updatedAt
```

API共通方針として未定義項目を拒否する場合は、これらが送信された時点で

```text
VALIDATION_ERROR
```

とする。

---

### 27.9 assetAccountIdの形式検証

`assetAccountId`は、ルート制約またはAPI共通のパスパラメータ検証方式で検証する。

概念例：

```php
Route::post(
    '/api/v1/asset-accounts/{assetAccountId}/available-settings',
    CreateAssetAccountAvailableSettingAction::class,
)
    ->where(
        'assetAccountId',
        '[1-9][0-9]*',
    );
```

形式不正は、

```text
INVALID_ASSET_ACCOUNT_ID
```

へ変換する。

---

### 27.10 Action

Actionは、検証済みRequest、`assetAccountId`、利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php
final class CreateAssetAccountAvailableSettingAction
{
    public function __invoke(
        string $assetAccountId,
        CreateAssetAccountAvailableSettingRequest $request,
        CreateAssetAccountAvailableSettingUseCase $useCase,
        CreateAssetAccountAvailableSettingResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $input =
            new CreateAssetAccountAvailableSettingInput(
                userId:
                    $userContext->userId,

                assetAccountId:
                    (int) $assetAccountId,

                startYearMonth:
                    $request->string(
                        'startYearMonth',
                    )->toString(),

                isAvailable:
                    $request->boolean(
                        'isAvailable',
                    ),
            );

        $result =
            $useCase->execute(
                $input,
            );

        return $responder->created(
            $result,
        );
    }
}
```

---

### 27.11 Input DTO

ActionからUseCaseへは、専用Input DTOを渡す。

概念例：

```php
final readonly class
    CreateAssetAccountAvailableSettingInput
{
    public function __construct(
        public int $userId,
        public int $assetAccountId,
        public string $startYearMonth,
        public bool $isAvailable,
    ) {
    }
}
```

RequestオブジェクトそのものをUseCaseへ渡さない。

---

### 27.12 Actionで行わないこと

Actionでは、以下を行わない。

- 資産口座検索
- 利用者境界判定
- 履歴取得
- 履歴整合性判定
- 現在設定取得
- 前月計算
- `lockForUpdate()`
- UPDATE
- INSERT
- トランザクション制御
- レスポンス配列生成

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 27.13 UseCase

ACC-006の業務処理全体を担当する。

主な処理は、以下とする。

1. Input DTOを受け取る
2. トランザクションを開始する
3. 対象資産口座を取得する
4. 対象不存在の場合は業務例外を送出する
5. 既存履歴を取得する
6. 履歴0件を確認する
7. 既存履歴整合性を検証する
8. 現在設定を`lockForUpdate()`で取得する
9. ロック後の最新状態を基準に業務条件を再確認する
10. 新設定開始年月の妥当性を確認する
11. `isAvailable`が現在設定と異なることを確認する
12. 旧設定終了年月を算出する
13. 旧設定を更新する
14. 新設定を登録する
15. Result DTOを生成する
16. コミットする
17. Result DTOを返却する

---

### 27.14 UseCaseの概念フロー

概念的には、以下とする。

```text
Input DTO
    ↓
DB::transaction
    ↓
AssetAccountQuery
    ↓
資産口座取得
    ↓
AvailableSettingQuery
    ↓
既存履歴取得
    ↓
HistoryValidator
    ↓
履歴整合性確認
    ↓
AvailableSettingQuery
    ↓
現在設定lockForUpdate
    ↓
最新状態再確認
    ↓
開始年月チェック
    ↓
isAvailable差分チェック
    ↓
前月算出
    ↓
Repository
    ├─ 旧設定終了
    └─ 新設定登録
    ↓
Result DTO
```

---

### 27.15 DB::transaction

ACC-006では、UseCase全体を

```php
DB::transaction()
```

で囲む。

概念例：

```php
return DB::transaction(
    function () use (
        $input,
    ): CreateAssetAccountAvailableSettingResult {
        // 業務処理
    },
);
```

旧設定UPDATEと新設定INSERTを原子的に扱う。

---

### 27.16 AssetAccountQuery

対象資産口座取得は、Queryへ委譲する。

概念例：

```php
$assetAccount =
    $this->assetAccountQuery
        ->findActiveByIdAndUser(
            assetAccountId:
                $input->assetAccountId,

            userId:
                $input->userId,
        );
```

取得条件は、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

---

### 27.17 利用者境界をQueryへ含める

以下のようなID単独取得を基本としない。

```php
AssetAccount::find(
    $assetAccountId,
);
```

対象取得時点で、

```text
assetAccountId
+
userId
+
利用中状態
```

を条件へ含める。

---

### 27.18 SoftDeletes

`AssetAccount` Modelでは、

```php
use SoftDeletes;
```

を使用する。

ACC-006では論理削除済み資産口座を対象としないため、

```php
withTrashed()
```

を使用しない。

---

### 27.19 ASSET_ACCOUNT_NOT_FOUND

対象資産口座を取得できない場合は、

```php
throw new
    AssetAccountNotFoundException();
```

とする。

以下を同じ例外へ集約する。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

最終的に、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

へ変換する。

---

### 27.20 AssetAccountAvailableSettingQuery

既存履歴取得は、専用Queryへ委譲する。

概念例：

```php
$settings =
    $this->availableSettingQuery
        ->findHistoryByAssetAccount(
            assetAccountId:
                $assetAccount->id,
        );
```

履歴検証しやすいように、

```text
start_year_month ASC
```

で取得してよい。

---

### 27.21 履歴0件

履歴が0件の場合は、正常状態としない。

概念例：

```php
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingHistoryNotFoundException();
}
```

最終的に、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

へ変換する。

---

### 27.22 AvailableSettingHistoryValidator

既存履歴の整合性確認には、ACC-005と同じValidatorを再利用する。

概念的には、

```php
$this->historyValidator
    ->validate(
        assetAccountStartYearMonth:
            $assetAccount->start_year_month,

        settings:
            $settings,
    );
```

とする。

同じ業務ルールをACC-005とACC-006で重複実装しない。

---

### 27.23 Validatorの確認内容

主に以下を確認する。

```text
最古設定開始年月
    =
assetAccount.startYearMonth

各設定
startYearMonth <= endYearMonth

期間重複なし

期間欠落なし

継続中設定1件

最新設定
endYearMonth = null
```

---

### 27.24 ValidatorでDB更新しない

`AvailableSettingHistoryValidator`は、整合性判定だけを行う。

以下を行わない。

- DB検索
- UPDATE
- INSERT
- 自動修復
- ロック

QueryとRepositoryの責務を持たせない。

---

### 27.25 現在設定をロック付きで取得する

履歴整合性確認後、現在設定をトランザクション内で再取得する。

概念例：

```php
$currentSetting =
    $this->availableSettingQuery
        ->findCurrentForUpdate(
            assetAccountId:
                $assetAccount->id,
        );
```

Query内部では、

```php
->where(
    'asset_account_id',
    $assetAccountId,
)
->whereNull(
    'end_year_month',
)
->lockForUpdate()
```

を使用する。

---

### 27.26 現在設定は1件を前提とする

`findCurrentForUpdate()`では、継続中設定が1件のみであることを前提とする。

0件または複数件の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ変換する。

単純に

```php
->first()
```

だけで複数件を黙って無視しない。

---

### 27.27 ロック後に再確認する

履歴整合性確認後からロック取得までの間に、別ACC-006が更新している可能性がある。

そのため、ロック取得後の現在設定を基準に、

```text
startYearMonth
isAvailable
```

の業務条件を再確認する。

---

### 27.28 startYearMonthの業務検証

新設定開始年月は、少なくとも

```text
request.startYearMonth
    >
currentSetting.startYearMonth
```

であることを確認する。

また、

```text
request.startYearMonth
    >=
assetAccount.startYearMonth
```

も満たすことを確認する。

---

### 27.29 開始年月不正

条件を満たさない場合は、

```php
throw new
    AssetAccountAvailableSettingStartYearMonthInvalidException();
```

とする。

最終的に、

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

へ変換する。

---

### 27.30 現在設定との差分確認

現在設定の

```text
is_available
```

とRequestの

```text
isAvailable
```

が異なることを確認する。

概念例：

```php
if (
    $currentSetting->is_available
    === $input->isAvailable
) {
    throw new
        AssetAccountAvailableSettingNoChangeException();
}
```

---

### 27.31 NO_CHANGE

現在設定と同じ状態の場合は、

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

へ変換する。

この場合、Repositoryを呼び出して旧設定を更新しない。

---

### 27.32 YearMonthの扱い

`startYearMonth`から前月を算出する必要がある。

年月計算を文字列操作で行わず、`CarbonImmutable`または`YearMonth` Value Objectを使用する。

概念例：

```php
$previousYearMonth =
    CarbonImmutable::createFromFormat(
        '!Y-m',
        $input->startYearMonth,
    )
        ->subMonth()
        ->format('Y-m');
```

---

### 27.33 年跨ぎを考慮する

例えば、

```text
2027-01
```

の前月は、

```text
2026-12
```

となる。

以下のような独自文字列演算は行わない。

```text
month - 1
```

---

### 27.34 YearMonth Value Object

年月処理が複数ユースケースへ広がる場合は、Value Objectを導入してよい。

概念例：

```php
final readonly class YearMonth
{
    public function __construct(
        public int $year,
        public int $month,
    ) {
    }

    public function previous(): self
    {
        // 前月返却
    }

    public function isAfter(
        self $other,
    ): bool {
        // 比較
    }

    public function toString(): string
    {
        return sprintf(
            '%04d-%02d',
            $this->year,
            $this->month,
        );
    }
}
```

Phase1では、過剰にならない範囲で採用する。

---

### 27.35 Repository

`asset_account_available_settings`の更新・登録は、Repositoryへ委譲する。

概念的なインターフェースは、以下とする。

```php
interface
    AssetAccountAvailableSettingRepository
{
    public function close(
        AssetAccountAvailableSetting $setting,
        string $endYearMonth,
    ): void;

    public function create(
        int $assetAccountId,
        string $startYearMonth,
        bool $isAvailable,
    ): AssetAccountAvailableSetting;
}
```

---

### 27.36 旧設定終了

Repositoryでは、現在設定の

```text
end_year_month
```

だけを更新する。

概念例：

```php
public function close(
    AssetAccountAvailableSetting $setting,
    string $endYearMonth,
): void {
    $setting->end_year_month =
        $endYearMonth;

    $setting->save();
}
```

以下を変更しない。

```text
asset_account_id
start_year_month
is_available
```

---

### 27.37 新設定登録

新設定は、明示的な値で登録する。

概念例：

```php
public function create(
    int $assetAccountId,
    string $startYearMonth,
    bool $isAvailable,
): AssetAccountAvailableSetting {
    $setting =
        new AssetAccountAvailableSetting();

    $setting->asset_account_id =
        $assetAccountId;

    $setting->start_year_month =
        $startYearMonth;

    $setting->end_year_month =
        null;

    $setting->is_available =
        $isAvailable;

    $setting->save();

    return $setting;
}
```

---

### 27.38 既存Modelを複製しない

以下のような現在設定Modelの複製をそのまま保存する方式は避ける。

```php
$newSetting =
    $currentSetting->replicate();
```

新設定の意味を明確にするため、必要項目を明示的に設定する。

---

### 27.39 Mass Assignmentを避ける

以下のような処理は行わない。

```php
AssetAccountAvailableSetting::create(
    $request->all(),
);
```

クライアントから指定可能な値とサーバー管理値を明確に分離する。

---

### 27.40 end_year_monthはサーバー管理とする

新設定では、

```text
end_year_month = null
```

とする。

旧設定の終了年月は、

```text
新設定開始年月の前月
```

としてサーバー側で計算する。

Request Body中の`endYearMonth`は使用しない。

---

### 27.41 asset_account_idはサーバー管理とする

新設定の

```text
asset_account_id
```

は、利用者境界確認済みの資産口座IDを使用する。

Request Bodyから受け取らない。

---

### 27.42 トランザクション内の更新順

概念的には、

```text
1. 現在設定lockForUpdate
2. 業務条件再確認
3. 旧設定close
4. 新設定create
```

の順とする。

旧設定終了前に新設定を作成して、一時的に継続中設定を複数作る構成は避ける。

---

### 27.43 UPDATE後にINSERT失敗した場合

新設定INSERTで例外が発生した場合は、

```text
旧設定.end_year_month更新
```

もトランザクションによってロールバックする。

UseCase外で別々にコミットしない。

---

### 27.44 UNIQUE制約

同一資産口座内では、

```text
asset_account_id
+
start_year_month
```

をUNIQUE制約とする。

アプリケーション側の業務チェックだけに依存しない。

---

### 27.45 継続中設定の部分UNIQUE制約

PostgreSQLでは、必要に応じて

```text
asset_account_id
WHERE end_year_month IS NULL
```

の部分UNIQUEインデックスを使用する。

概念例：

```sql
CREATE UNIQUE INDEX
    uq_asset_account_available_settings_current
ON
    asset_account_available_settings (
        asset_account_id
    )
WHERE
    end_year_month IS NULL;
```

ACC-006の`lockForUpdate()`と合わせて、DB側でも整合性を保証する。

---

### 27.46 UNIQUE制約違反の変換

同一開始年月のUNIQUE制約違反が発生した場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

へ変換する。

概念的には、

```text
PostgreSQL unique violation
    ↓
AssetAccountAvailableSettingAlreadyExistsException
    ↓
409 Conflict
```

とする。

SQLSTATEや制約名はクライアントへ公開しない。

---

### 27.47 部分UNIQUE制約違反

継続中設定の部分UNIQUE制約違反が発生した場合は、同時更新競合または履歴不整合として扱う。

API共通方針に応じて、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

または競合系エラーへ変換する。

Phase1では、内部DB例外をそのまま返却しないことを優先する。

---

### 27.48 lockForUpdate

現在設定取得には、

```php
lockForUpdate()
```

を使用する。

ただし、必ず

```php
DB::transaction()
```

内で使用する。

トランザクション外で形式的に呼び出すだけの実装にはしない。

---

### 27.49 ロック対象

原則として、現在継続中の利用可能資産設定1件だけをロックする。

履歴全件を`lockForUpdate()`しない。

また、利用者の全資産口座をロックしない。

---

### 27.50 ロック後の処理を短く保つ

ロック取得後は、

```text
最新状態確認
前月算出
旧設定UPDATE
新設定INSERT
```

を速やかに実行する。

以下をロック保持中に行わない。

- 外部API
- メール送信
- ファイル処理
- 大量データ取得
- レスポンス整形

---

### 27.51 Result DTO

新規登録した利用可能資産設定を、専用Result DTOで表現する。

概念例：

```php
final readonly class
    CreateAssetAccountAvailableSettingResult
{
    public function __construct(
        public int $id,
        public string $startYearMonth,
        public ?string $endYearMonth,
        public bool $isAvailable,
    ) {
    }
}
```

正常登録直後の`endYearMonth`は`null`となる。

---

### 27.52 DTOへEloquent Modelを保持しない

以下のようなResult DTOは基本としない。

```php
final readonly class
    CreateAssetAccountAvailableSettingResult
{
    public function __construct(
        public AssetAccountAvailableSetting $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

---

### 27.53 UseCaseの戻り値

概念例：

```php
return new
    CreateAssetAccountAvailableSettingResult(
        id:
            $newSetting->id,

        startYearMonth:
            $newSetting->start_year_month,

        endYearMonth:
            $newSetting->end_year_month,

        isAvailable:
            (bool) $newSetting->is_available,
    );
```

Eloquent ModelをActionへ直接返却しない。

---

### 27.54 API Resource

Result DTOを、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php
final class
    CreateAssetAccountAvailableSettingResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'startYearMonth'
                => $this->startYearMonth,

            'endYearMonth'
                => $this->endYearMonth,

            'isAvailable'
                => $this->isAvailable,
        ];
    }
}
```

API共通方針に従ってcamelCaseで返却する。

---

### 27.55 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

- `asset_account_id`
- `user_id`
- `created_at`
- `updated_at`
- 旧設定ID
- 旧設定`start_year_month`
- 旧設定`end_year_month`
- 旧設定`is_available`
- 資産口座名
- 資産種別
- 残高記録単位
- `deleted_at`

旧履歴を確認する場合は、ACC-005を使用する。

---

### 27.56 Responder

Responderは、Result DTOを`201 Created`へ変換する。

概念例：

```php
final class
    CreateAssetAccountAvailableSettingResponder
{
    public function created(
        CreateAssetAccountAvailableSettingResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    =>
                    new CreateAssetAccountAvailableSettingResource(
                        $result,
                    ),
            ],
            Response::HTTP_CREATED,
        );
    }
}
```

`requestId`等の共通Envelope項目は、API共通レスポンス処理に従う。

---

### 27.57 Locationヘッダー

`201 Created`では、新規リソースを示す`Location`ヘッダーを設定することもできる。

ただし、Phase1では利用可能資産設定の個別取得APIを用意していないため、ACC-006専用の`Location`ヘッダーは必須としない。

---

### 27.58 Responderで行わないこと

Responderでは、以下を行わない。

- 資産口座検索
- 履歴取得
- 履歴整合性判定
- 現在設定取得
- 行ロック
- 前月計算
- UPDATE
- INSERT
- トランザクション制御
- 業務例外判定

HTTPレスポンス生成だけに責務を限定する。

---

### 27.59 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `assetAccountId`形式不正 | `INVALID_ASSET_ACCOUNT_ID` |
| Request Body不正 | `VALIDATION_ERROR` |
| 資産口座不存在 | `ASSET_ACCOUNT_NOT_FOUND` |
| 他利用者の資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 論理削除済み資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 履歴0件 | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND` |
| 履歴不整合 | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID` |
| 開始年月不正 | `ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID` |
| 現在設定と同一状態 | `ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE` |
| 同一開始年月重複 | `ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

---

### 27.60 START_YEAR_MONTH_INVALID

開始年月の業務妥当性に違反した場合は、専用例外を送出する。

概念例：

```php
if (
    ! $newStartYearMonth
        ->isAfter(
            $currentStartYearMonth,
        )
) {
    throw new
        AssetAccountAvailableSettingStartYearMonthInvalidException();
}
```

最終的に、

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

へ変換する。

---

### 27.61 NO_CHANGE

現在設定と同じ`isAvailable`の場合は、

```php
throw new
    AssetAccountAvailableSettingNoChangeException();
```

とする。

最終的に、

```text
409 Conflict
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

へ変換する。

---

### 27.62 HISTORY_NOT_FOUND

履歴が0件の場合は、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

へ変換する。

ACC-002の初期設定登録ルールに反する状態として扱う。

---

### 27.63 HISTORY_INVALID

既存履歴整合性に違反した場合は、

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ変換する。

内部的には、ACC-005と同じ`invalidReason`を保持してよい。

---

### 27.64 DB例外変換

PostgreSQLのDB例外は、制約名やSQLSTATEを判定材料として内部で利用してよい。

ただし、レスポンスへそのまま公開しない。

例えば、

```text
unique violation
    ↓
業務例外へ変換
```

とする。

---

### 27.65 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- SQL
- SQLSTATE
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

---

### 27.66 ログ

ACC-006では、必要に応じて以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assetAccountId
httpStatus
errorCode
```

`apiId`は、

```text
ACC-006
```

とする。

---

### 27.67 正常時ログ

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = ACC-006
assetAccountId
newSettingId
startYearMonth
httpStatus = 201
```

必要であれば、

```text
previousIsAvailable
newIsAvailable
```

も内部構造化ログへ記録してよい。

---

### 27.68 履歴不整合ログ

以下の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

調査に必要な情報を記録する。

概念例：

```text
assetAccountId
historyCount
invalidReason
```

---

### 27.69 競合ログ

以下の場合は、必要に応じて競合として記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

同時実行が多発する場合に監視できる状態としてよい。

---

### 27.70 トランザクションロールバックログ

旧設定UPDATE後に新設定INSERTが失敗した場合などは、必要に応じて

```text
transactionRolledBack = true
```

相当の内部ログを記録してよい。

---

### 27.71 キャッシュ

Phase1では、ACC-006専用のサーバー側キャッシュを使用しない。

React側では、ACC-006成功後に少なくとも以下をinvalidateする。

```text
ACC-005
利用可能資産設定履歴

ACC-003
資産口座詳細
```

ACC-001が`isAvailable`を返却する場合は、一覧Queryもinvalidate対象とする。

---

### 27.72 テスト実装方針

Laravel側では、Feature Testを中心にAPI契約とトランザクション動作を確認する。

また、以下についてはUnit TestまたはDatabase Testも行う。

- FormRequest
- AssetAccountQuery
- AvailableSettingQuery
- HistoryValidator
- Repository
- UseCase
- API Resource

---

### 27.73 FormRequestのTest

以下を確認する。

```text
startYearMonth
    required
    string
    YYYY-MM
    実在年月

isAvailable
    required
    strict boolean
```

主な異常系：

```text
startYearMonth未指定
startYearMonth = null
2026-8
2026/08
2026-13
isAvailable未指定
isAvailable = null
"true"
1
0
```

---

### 27.74 AssetAccountQueryのDatabase Test

以下を確認する。

```text
id一致
+
user_id一致
+
deleted_at IS NULL
    ↓
取得できる
```

以下は取得できないこと。

- 存在しないID
- 他利用者の資産口座
- 論理削除済み資産口座

---

### 27.75 AvailableSettingQueryの履歴取得Test

対象資産口座について、履歴全件が取得できることを確認する。

```text
asset_account_id
    = assetAccountId
```

他資産口座の履歴が混在しないこと。

---

### 27.76 findCurrentForUpdateのTest

現在設定取得について、

```text
asset_account_id
    = assetAccountId

AND

end_year_month IS NULL
```

となる1件を取得できることを確認する。

Integration Testでは、可能であれば`FOR UPDATE`が発行されることも確認する。

---

### 27.77 HistoryValidatorの再利用Test

ACC-005とACC-006で同じ履歴整合性Validatorを利用する。

以下の同じ履歴に対して、両UseCaseで判定結果が一致することを確認する。

```text
初期開始年月不一致
期間重複
期間欠落
継続中設定0件
継続中設定複数件
最新設定終了済み
```

---

### 27.78 開始年月のUnit Test

現在設定が

```text
2026-08 ～ NULL
```

の場合を考える。

正常：

```text
2026-09
2026-12
2027-01
```

異常：

```text
2026-08
2026-07
2025-12
```

異常時に`StartYearMonthInvalidException`となることを確認する。

---

### 27.79 同一状態のUnit Test

現在設定：

```text
isAvailable = true
```

Request：

```text
isAvailable = true
```

の場合、

```text
AssetAccountAvailableSettingNoChangeException
```

となること。

Repositoryが呼ばれないことも確認する。

---

### 27.80 前月計算のUnit Test

以下を確認する。

```text
2026-08
    ↓
2026-07
```

```text
2027-01
    ↓
2026-12
```

```text
2026-01
    ↓
2025-12
```

年跨ぎを正しく扱えること。

---

### 27.81 RepositoryのDatabase Test

`close()`について、以下を確認する。

```text
end_year_month
    → 指定年月へ更新
```

以下は変更されないこと。

```text
asset_account_id
start_year_month
is_available
```

---

### 27.82 create()のDatabase Test

新設定が、以下で作成されることを確認する。

```text
asset_account_id
    = 指定ID

start_year_month
    = 指定年月

end_year_month
    = NULL

is_available
    = 指定boolean
```

---

### 27.83 UseCase正常系Test

Query、Validator、Repositoryを使用して、以下の順で処理されることを確認する。

```text
資産口座取得
    ↓
履歴取得
    ↓
履歴検証
    ↓
現在設定ロック
    ↓
最新状態検証
    ↓
旧設定close
    ↓
新設定create
    ↓
Result DTO
```

---

### 27.84 UseCaseのNO_CHANGE Test

現在設定とRequestの`isAvailable`が同一の場合は、

```text
NO_CHANGE
```

となり、

```text
Repository::close()
Repository::create()
```

が呼ばれないことを確認する。

---

### 27.85 UseCaseの開始年月不正Test

`startYearMonth`が現在設定開始年月以下の場合は、

```text
START_YEAR_MONTH_INVALID
```

となり、DB更新されないことを確認する。

---

### 27.86 トランザクションロールバックTest

旧設定更新後、新設定作成時に例外を発生させる。

以下を確認する。

```text
旧設定.end_year_month
    → 元のNULLへ戻る

新設定
    → 存在しない
```

---

### 27.87 UNIQUE制約Test

同一

```text
asset_account_id
+
start_year_month
```

で複数設定を登録できないことを確認する。

制約違反は、APIでは

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

へ変換されること。

---

### 27.88 継続中設定部分UNIQUE Test

部分UNIQUE制約を採用する場合は、同一資産口座に

```text
end_year_month IS NULL
```

の設定を2件作れないことを確認する。

---

### 27.89 同時実行Integration Test

可能であれば、同一資産口座に対するACC-006を並行実行する。

以下を確認する。

- `lockForUpdate()`によって直列化される
- 同じ現在設定を二重終了しない
- 継続中設定が複数にならない
- 同一開始年月が重複しない
- 後続Requestが最新状態を基準に再評価される

---

### 27.90 同時同一Request Test

以下を同時送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": false
}
```

以下を確認する。

- 1つの履歴だけが登録される
- 2件目は業務エラーとなる
- DB例外がそのまま返らない
- 履歴期間が壊れない

---

### 27.91 API ResourceのTest

正常時に、以下の項目だけを返却することを確認する。

```json
{
  "id": "15",
  "startYearMonth": "2026-08",
  "endYearMonth": null,
  "isAvailable": false
}
```

以下を返さないこと。

- `assetAccountId`
- `asset_account_id`
- `userId`
- `createdAt`
- `updatedAt`
- 旧設定情報
- DB内部情報

---

### 27.92 Feature Test

Feature Testでは、少なくとも以下を確認する。

```text
201 Created
400 Bad Request
404 Not Found
409 Conflict
500 Internal Server Error
```

主なケースは、

- 正常登録
- 利用者コンテキスト不正
- `assetAccountId`不正
- Request Body不正
- 他利用者資産口座
- 論理削除済み資産口座
- 履歴0件
- 履歴不整合
- 開始年月不正
- `NO_CHANGE`
- 同一開始年月重複
- トランザクションロールバック
- レスポンス契約

とする。

---

### 27.93 副作用範囲Test

ACC-006正常終了時に変更される業務データが、

```text
asset_account_available_settings

旧設定1件
    UPDATE

新設定1件
    INSERT
```

だけであることを確認する。

以下が変更されないことも確認する。

```text
asset_accounts
holding_assets
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
net_incomes
objectives
assessment_histories
```

---

## 28. React・TypeScriptでの利用

ACC-006は、指定した資産口座の利用可能資産区分を変更する際に使用する。

フロントエンドでは、ACC-003で現在状態を取得し、必要に応じてACC-005で利用可能資産設定履歴を確認したうえで、ACC-006をMutationとして実行する。

概念的な利用フローは、以下とする。

```text
ACC-003
資産口座詳細取得
    ↓
現在のisAvailable確認
    ↓
必要に応じてACC-005
利用可能資産設定履歴取得
    ↓
利用者が
startYearMonth
isAvailable
を入力
    ↓
ACC-006
POST
/api/v1/asset-accounts/{assetAccountId}/available-settings
    ↓
成功
    ↓
関連Query Cache無効化
    ↓
ACC-003 / ACC-005再取得
```

ACC-006は、サーバー状態を変更するため、TanStack Queryを使用する場合はMutationとして扱う。

---

### 28.1 TypeScript型

ACC-006のリクエスト型は、以下のように定義する。

概念例：

```typescript
export type CreateAssetAccountAvailableSettingRequest = {
  startYearMonth: string;
  isAvailable: boolean;
};
```

両項目とも必須とする。

---

### 28.2 正常レスポンス型

ACC-006成功時は、新しく登録された利用可能資産設定を返却する。

概念例：

```typescript
export type AssetAccountAvailableSetting = {
  id: string;
  startYearMonth: string;
  endYearMonth: string | null;
  isAvailable: boolean;
};
```

API共通Envelopeを使用する場合は、以下のように定義する。

```typescript
export type CreateAssetAccountAvailableSettingResponse =
  ApiResponse<AssetAccountAvailableSetting>;
```

---

### 28.3 ACC-005と共通型を使用してよい

ACC-005で返却する利用可能資産設定履歴1件と、ACC-006成功時に返却する新規設定の項目が同一である場合は、共通型を使用してよい。

概念例：

```typescript
export type AssetAccountAvailableSetting = {
  id: string;
  startYearMonth: string;
  endYearMonth: string | null;
  isAvailable: boolean;
};
```

ACC-005：

```typescript
export type AssetAccountAvailableSettingHistoryResponse =
  ApiResponse<
    AssetAccountAvailableSetting[]
  >;
```

ACC-006：

```typescript
export type CreateAssetAccountAvailableSettingResponse =
  ApiResponse<
    AssetAccountAvailableSetting
  >;
```

同じ意味のデータに対してAPIごとに不要な重複型を増やさない。

---

### 28.4 assetAccountId

対象資産口座IDは、API Client関数の引数として渡す。

概念例：

```typescript
createAssetAccountAvailableSetting(
  assetAccountId,
  request,
);
```

`assetAccountId`はAPI契約に合わせて`string`として扱う。

---

### 28.5 userIdをAPI Client引数へ含めない

以下のような関数にはしない。

```typescript
createAssetAccountAvailableSetting(
  userId,
  assetAccountId,
  request,
);
```

利用者IDは、共通API Clientから

```text
X-User-Id
```

として付与する。

---

### 28.6 X-User-Id

`X-User-Id`は、ACC-006専用処理ではなく、共通API Clientから付与する。

概念例：

```typescript
apiClient.interceptors.request.use(
  (config) => {
    config.headers['X-User-Id'] =
      currentUserId;

    return config;
  },
);
```

各コンポーネントやHookから直接ヘッダーを生成しない。

---

### 28.7 フォーム型

利用可能資産設定登録フォームは、以下のように定義できる。

概念例：

```typescript
export type CreateAssetAccountAvailableSettingFormValues = {
  startYearMonth: string;
  isAvailable: boolean;
};
```

Request型とフォーム型が同一であれば、共通利用してもよい。

ただし、画面固有の入力状態が増える場合は分離する。

---

### 28.8 startYearMonth

`startYearMonth`は、

```text
YYYY-MM
```

形式の文字列として扱う。

HTMLの

```html
<input type="month">
```

を使用する場合は、通常

```text
YYYY-MM
```

形式で取得できる。

概念例：

```tsx
<input
  type="month"
  value={form.startYearMonth}
  onChange={(event) => {
    setForm({
      ...form,
      startYearMonth:
        event.target.value,
    });
  }}
/>
```

---

### 28.9 startYearMonthをDateへ変換して保持しなくてよい

API契約は

```text
YYYY-MM
```

であるため、フォームStateをJavaScriptの`Date`へ必ず変換する必要はない。

概念的には、

```typescript
startYearMonth: string;
```

として保持してよい。

タイムゾーンの影響を不要に持ち込まない。

---

### 28.10 isAvailable

`isAvailable`は、booleanとして保持する。

概念例：

```tsx
<select
  value={
    form.isAvailable
      ? 'true'
      : 'false'
  }
  onChange={(event) => {
    setForm({
      ...form,
      isAvailable:
        event.target.value === 'true',
    });
  }}
>
  <option value="true">
    利用可能
  </option>

  <option value="false">
    利用対象外
  </option>
</select>
```

Request送信時には、文字列ではなくbooleanへ変換する。

---

### 28.11 booleanを文字列のまま送信しない

HTMLフォームでは、`select`や`radio`から文字列として値を取得する場合がある。

以下のように、

```json
{
  "isAvailable": "false"
}
```

を送信しない。

送信時には、

```json
{
  "isAvailable": false
}
```

とする。

---

### 28.12 現在状態を初期表示する

ACC-003で取得した

```text
isAvailable
```

を使用して、現在の利用可能資産区分を表示する。

例えば、

```text
現在の設定
利用可能
```

または、

```text
現在の設定
利用対象外
```

とする。

ただし、ACC-006の新設定入力値を現在値でそのまま確定しない。

---

### 28.13 現在状態と異なる値だけ選択させてもよい

ACC-006では、現在設定と同じ`isAvailable`を指定すると

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

となる。

そのため、フロントエンドでは現在値と反対の選択肢だけを提示することもできる。

例えば、現在が

```text
isAvailable = true
```

なら、

```text
利用対象外へ変更
```

という操作ボタンとして表現してよい。

---

### 28.14 ただしサーバー側判定を残す

フロントエンドで同一状態を選択できないようにしても、サーバー側の

```text
NO_CHANGE
```

判定は必要とする。

別タブや別リクエストによって画面表示後に現在状態が変更される可能性があるためである。

---

### 28.15 現在設定開始年月

ACC-006の`startYearMonth`は、現在設定の開始年月より後である必要がある。

ただし、ACC-003では現在の利用可能資産設定の開始年月を返却しない設計であるため、必要な場合はACC-005の履歴を利用する。

概念的には、

```text
ACC-005
履歴先頭
    ↓
endYearMonth = null
    ↓
現在設定
    ↓
startYearMonth
```

とする。

---

### 28.16 ACC-005から現在設定を取得する

ACC-005は、

```text
startYearMonth DESC
```

で返却するため、正常状態では先頭レコードが継続中設定となる。

概念例：

```typescript
const currentSetting =
  settings?.find(
    (setting) =>
      setting.endYearMonth === null,
  );
```

API契約上、継続中設定は1件のみである。

---

### 28.17 現在設定をフロントエンドで最終保証しない

ACC-005の正常レスポンスから現在設定を表示できるが、ACC-006実行時の最終判定はバックエンド側で行う。

フロントエンドで取得した`currentSetting`を更新対象IDとしてRequestへ送信しない。

---

### 28.18 settingIdを送信しない

以下のようなRequestにはしない。

```typescript
type CreateRequest = {
  currentSettingId: string;
  startYearMonth: string;
  isAvailable: boolean;
};
```

ACC-006では、現在設定をサーバー側で特定する。

---

### 28.19 endYearMonthを送信しない

フォームに

```text
終了年月
```

を入力するUIは設けない。

旧設定の終了年月は、

```text
新設定開始年月の前月
```

としてサーバー側で算出する。

---

### 28.20 旧設定を編集させない

ACC-005で表示した過去の利用可能資産設定に対して、

```text
編集
削除
```

ボタンをPhase1では設けない。

ACC-006は、現在履歴の後ろへ新しい設定を追加する操作に限定する。

---

### 28.21 API Client

ACC-006を呼び出す専用API Client関数を定義する。

概念例：

```typescript
export const createAssetAccountAvailableSetting =
  async (
    assetAccountId: string,
    request:
      CreateAssetAccountAvailableSettingRequest,
  ): Promise<
    AssetAccountAvailableSetting
  > => {
    const response =
      await apiClient.post<
        CreateAssetAccountAvailableSettingResponse
      >(
        `/api/v1/asset-accounts/${assetAccountId}/available-settings`,
        request,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

### 28.22 Mutationとして扱う

ACC-006はサーバー状態を変更するため、TanStack QueryではMutationとして扱う。

概念例：

```typescript
export type CreateAssetAccountAvailableSettingVariables = {
  assetAccountId: string;
  request:
    CreateAssetAccountAvailableSettingRequest;
};
```

---

### 28.23 Mutation Hook

概念例：

```typescript
export const useCreateAssetAccountAvailableSetting =
  () => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn: (
        variables:
          CreateAssetAccountAvailableSettingVariables,
      ) =>
        createAssetAccountAvailableSetting(
          variables.assetAccountId,
          variables.request,
        ),

      onSuccess: async (
        _result,
        variables,
      ) => {
        await queryClient
          .invalidateQueries({
            queryKey:
              assetAccountKeys
                .availableSettings(
                  variables.assetAccountId,
                ),
          });

        await queryClient
          .invalidateQueries({
            queryKey:
              assetAccountKeys.detail(
                variables.assetAccountId,
              ),
          });
      },

      retry: false,
    });
  };
```

---

### 28.24 ACC-005のQuery Cacheを無効化する

ACC-006成功時は、

```text
旧設定.endYearMonth更新
+
新設定追加
```

が発生するため、ACC-005の履歴Cacheは必ず古くなる。

そのため、

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys
      .availableSettings(
        assetAccountId,
      ),
});
```

を実行する。

---

### 28.25 ACC-003のQuery Cacheを無効化する

ACC-006によって現在の

```text
isAvailable
```

が変化する可能性があるため、ACC-003の詳細Queryも無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

---

### 28.26 ACC-001のQuery Cache

ACC-001 資産口座一覧取得が`isAvailable`を返却する設計の場合は、一覧Queryも無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.all,
});
```

ACC-001が利用可能資産区分を返却しない場合は、必須ではない。

---

### 28.27 将来開始設定でも履歴Cacheは無効化する

例えば、現在年月が

```text
2026-08
```

で、

```json
{
  "startYearMonth": "2026-10",
  "isAvailable": false
}
```

を登録した場合でも、履歴は即座に変化する。

そのため、ACC-005は必ず再取得する。

---

### 28.28 将来開始設定とACC-003

将来開始設定の場合、現在時点の`isAvailable`は変化しないことがある。

例えば、

```text
現在年月
2026-08

旧設定
2026-01 ～ 2026-09
true

新設定
2026-10 ～ NULL
false
```

では、ACC-003の現在値は引き続き`true`となる。

React側で独自に現在値を変更せず、ACC-003再取得結果をそのまま使用する。

---

### 28.29 成功レスポンスだけで履歴を再構築しない

ACC-006成功レスポンスでは、新規設定だけが返却される。

旧設定の更新後状態は返却されない。

そのため、以下のような履歴Cacheの手動再構築はPhase1では基本としない。

```typescript
queryClient.setQueryData(
  key,
  (old) => {
    // 旧設定endYearMonthを計算
    // 新設定を追加
  },
);
```

ACC-005を再取得してサーバー確定状態を使用する。

---

### 28.30 Optimistic Update

Phase1では、ACC-006に対するOptimistic Updateを必須としない。

ACC-006は、

- 履歴整合性
- 開始年月
- 現在設定
- `NO_CHANGE`
- 同時更新
- UNIQUE制約
- 行ロック

などのサーバー側判定を伴うためである。

---

### 28.31 Mutationの自動Retry

ACC-006では、Mutationの無条件な自動Retryを行わない。

概念例：

```typescript
useMutation({
  mutationFn:
    createAssetAccountAvailableSetting,
  retry: false,
});
```

ACC-006はPOSTであり、通信結果不明時に同じRequestを自動的に再送しない。

---

### 28.32 通信結果不明時

ACC-006送信後に通信エラーとなり、サーバー側で成功したか判断できない場合は、

```text
ACC-005
履歴再取得

+
ACC-003
現在状態再取得
```

によって現在状態を確認する。

同一Requestを即座に自動再送しない。

---

### 28.33 二重送信防止

Mutation実行中は、登録ボタンを非活性化する。

概念例：

```tsx
<button
  type="submit"
  disabled={
    mutation.isPending
  }
>
  {mutation.isPending
    ? '変更中...'
    : '設定を変更'}
</button>
```

ただし、フロントエンド側の二重送信防止だけを整合性保証にはしない。

サーバー側でも

```text
lockForUpdate
UNIQUE制約
業務状態再確認
```

を行う。

---

### 28.34 フォーム送信

概念例：

```typescript
const mutation =
  useCreateAssetAccountAvailableSetting();

const handleSubmit =
  (
    values:
      CreateAssetAccountAvailableSettingFormValues,
  ): void => {
    mutation.mutate({
      assetAccountId,
      request: {
        startYearMonth:
          values.startYearMonth,

        isAvailable:
          values.isAvailable,
      },
    });
  };
```

---

### 28.35 クライアント側バリデーション

送信前に、最低限以下を確認してよい。

- `startYearMonth`が入力されている
- `YYYY-MM`形式である
- `isAvailable`がbooleanとして確定している

ただし、サーバー側バリデーションを省略しない。

---

### 28.36 現在設定開始年月との簡易チェック

ACC-005から現在設定を取得済みであれば、

```text
request.startYearMonth
    >
currentSetting.startYearMonth
```

をフロントエンドでも事前確認してよい。

これにより、明らかに不正な入力を送信前に防げる。

---

### 28.37 業務判定の最終保証はサーバー側とする

フロントエンドで開始年月をチェックしても、ACC-006実行直前に別リクエストで現在設定が変わる可能性がある。

そのため、

```text
START_YEAR_MONTH_INVALID
```

の最終判定はLaravel側で行う。

---

### 28.38 NO_CHANGEの事前抑止

現在の`isAvailable`が取得できている場合は、同じ状態を選択できないようにしてよい。

例えば、

```tsx
<option
  value="true"
  disabled={
    currentIsAvailable === true
  }
>
  利用可能
</option>
```

ただし、バックエンドの`NO_CHANGE`判定は残す。

---

### 28.39 成功時

ACC-006成功時は、

```text
201 Created
```

となる。

例えば、

```json
{
  "data": {
    "id": "15",
    "startYearMonth": "2026-08",
    "endYearMonth": null,
    "isAvailable": false
  }
}
```

が返却される。

---

### 28.40 成功メッセージ

正常終了後は、必要に応じて

```text
利用可能資産設定を変更しました。
```

などの完了メッセージを表示する。

---

### 28.41 成功後の画面

成功後は、画面設計に応じて

- 設定画面に留まる
- 資産口座詳細画面へ戻る
- 履歴一覧を再表示する

などの動作とする。

Phase1では、

```text
Mutation成功
    ↓
ACC-003 / ACC-005再取得
    ↓
最新状態表示
```

という構成としてよい。

---

### 28.42 VALIDATION_ERROR

Laravel側から

```text
VALIDATION_ERROR
```

が返却された場合は、`error.details`を利用して対応するフォーム項目へエラーを表示する。

主な対象は、

```text
startYearMonth
isAvailable
```

とする。

---

### 28.43 startYearMonthのフィールドエラー

例えば、

```text
適用開始年月を入力してください。
```

または、

```text
適用開始年月はYYYY-MM形式で入力してください。
```

などを入力欄付近へ表示する。

具体的な文言は、API共通エラー表示方針に従う。

---

### 28.44 isAvailableのフィールドエラー

通常のUIでは、boolean以外を送信しない構成にする。

それでも

```text
VALIDATION_ERROR
```

となった場合は、設定フォーム全体のエラーとして扱ってもよい。

---

### 28.45 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

となる。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

フロントエンドでは理由を推測せず、

```text
指定された資産口座が見つかりません。
```

などの共通表示とする。

---

### 28.46 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-006専用フォームエラーにはしない。

---

### 28.47 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、利用者コンテキストに関する共通エラーとして扱う。

---

### 28.48 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択中の利用者が有効ではない状態として共通処理する。

---

### 28.49 INVALID_ASSET_ACCOUNT_ID

通常画面ではACC-001やACC-003から取得した有効なIDを使用するため、発生頻度は低い。

発生した場合は、不正なURLまたは画面状態として扱う。

---

### 28.50 HISTORY_NOT_FOUND

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

は、サーバー側の業務データ不整合として扱う。

フロントエンドで初期設定を補完しない。

例えば、

```text
利用可能資産設定を
変更できませんでした。
```

などを表示する。

---

### 28.51 HISTORY_INVALID

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

の場合も、フロントエンドで履歴を補正しない。

以下を行わない。

- 期間重複の解消
- 欠落期間の補完
- 継続中設定の選択
- 終了年月の自動修正

一般的な設定変更失敗として扱う。

---

### 28.52 START_YEAR_MONTH_INVALID

以下の場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_START_YEAR_MONTH_INVALID
```

となる。

主に、

- 現在設定開始年月と同じ
- 現在設定開始年月より前
- 資産口座利用開始年月より前
- 履歴途中への挿入

などである。

フロントエンドでは、

```text
適用開始年月を確認してください。
```

などを表示し、`startYearMonth`入力欄へ関連付けてよい。

---

### 28.53 NO_CHANGE

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

の場合は、現在設定と同じ状態が指定されている。

例えば、

```text
現在の利用可能資産区分と
同じ設定です。
```

などを表示してよい。

---

### 28.54 NO_CHANGE受信後は再取得してよい

画面上では異なる状態を選択したつもりでも、別リクエストによって現在状態が変化していた可能性がある。

そのため、`NO_CHANGE`受信後に

```text
ACC-003
ACC-005
```

を再取得して最新状態を表示してよい。

---

### 28.55 ALREADY_EXISTS

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_ALREADY_EXISTS
```

の場合は、同じ開始年月の設定が既に存在する。

例えば、

```text
指定した適用開始年月の設定は
すでに登録されています。
```

などを表示する。

ただし、同時更新によって発生した可能性もあるため、履歴を再取得してよい。

---

### 28.56 409 Conflict受信後

以下の409系エラーでは、

```text
START_YEAR_MONTH_INVALID
NO_CHANGE
ALREADY_EXISTS
```

必要に応じてACC-005を再取得する。

現在の履歴状態が画面表示時から変化している可能性があるためである。

---

### 28.57 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
利用可能資産設定を
変更できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

### 28.58 エラー時のフォーム値

ACC-006が失敗した場合は、原則として利用者が入力した

```text
startYearMonth
isAvailable
```

を保持する。

入力内容を自動的に初期化しない。

---

### 28.59 再取得後にフォーム値を見直す

409系エラーなどでACC-005を再取得した場合、現在設定が変化している可能性がある。

その場合は、現在のフォーム値が最新状態に対して有効かを再評価する。

---

### 28.60 過去履歴編集UIを設けない

ACC-005で表示した過去履歴に対して、

```text
編集
削除
```

の操作をACC-006へ結び付けない。

ACC-006は新しい設定追加専用とする。

---

### 28.61 資産口座編集UIとの分離

資産口座詳細画面で通常属性編集と利用可能資産設定変更を同時に表示する場合でも、内部的には別Mutationとする。

```text
name / assetType
    ↓
ACC-004

isAvailable変更
    ↓
ACC-006
```

1つのRequestへ統合しない。

---

### 28.62 資産口座無効化との分離

同じ画面に資産口座無効化操作があっても、

```text
通常属性更新
    → ACC-004

利用可能資産設定変更
    → ACC-006

資産口座無効化
    → 無効化API
```

として別操作とする。

---

### 28.63 Pageの責務

資産口座詳細または利用可能資産設定Pageでは、主に以下を担当する。

- URLから`assetAccountId`取得
- ACC-003による現在状態取得
- ACC-005による履歴取得
- ACC-006成功後の画面更新
- ページ単位のエラー表示

HTTP通信の詳細はAPI ClientやHookへ委譲する。

---

### 28.64 Form Componentの責務

利用可能資産設定フォームでは、主に以下を担当する。

- `startYearMonth`入力
- `isAvailable`選択
- クライアント側入力チェック
- フィールドエラー表示
- submitイベント通知

API Clientを直接呼び出さない構成としてよい。

---

### 28.65 Mutation Hookの責務

Mutation Hookでは、主に以下を担当する。

```text
ACC-006実行
Mutation状態管理
成功時Query invalidate
```

画面固有の表示レイアウトや業務文言をMutation Hookへ持たせない。

---

### 28.66 API Clientの責務

API Clientでは、

```http
POST /api/v1/asset-accounts/{assetAccountId}/available-settings
```

のHTTP通信と型付きレスポンス取得を担当する。

以下をAPI Clientへ含めない。

- Toast表示
- 画面遷移
- 現在設定表示
- 履歴期間表示
- 業務エラー文言生成

---

### 28.67 QueryとMutationを分離する

利用可能資産設定では、

```text
ACC-005
    → Query

ACC-006
    → Mutation
```

とする。

同じHookで取得と更新をまとめすぎない。

---

### 28.68 Query Key

例えば、以下のように共通定義する。

```typescript
export const assetAccountKeys = {
  all: [
    'assetAccounts',
  ] as const,

  detail: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
    ] as const,

  availableSettings: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
      'availableSettings',
    ] as const,
};
```

---

### 28.69 利用者切替時

Phase1では、`X-User-Id`によって操作対象利用者を切り替える。

利用者切替時に、前利用者の

```text
ACC-003
ACC-005
```

のQuery Cacheを新しい利用者へ誤表示しないようにする。

---

### 28.70 Query KeyへuserIdを含めてもよい

利用者境界をQuery Cache上でも明確にする場合は、

```typescript
[
  'assetAccounts',
  userId,
  assetAccountId,
  'availableSettings',
]
```

のように`userId`を含めてよい。

正式なQuery Key方針は、Reactアーキテクチャ設計に従う。

---

### 28.71 概念的なディレクトリ構成

例えば、以下のように整理できる。

```text
features/
└── asset-accounts/
    ├── api/
    │   ├── getAssetAccountDetail.ts
    │   ├── getAssetAccountAvailableSettings.ts
    │   └── createAssetAccountAvailableSetting.ts
    ├── components/
    │   ├── AssetAccountDetail.tsx
    │   ├── AvailableSettingHistory.tsx
    │   └── AvailableSettingForm.tsx
    ├── hooks/
    │   ├── useAssetAccountDetail.ts
    │   ├── useAssetAccountAvailableSettings.ts
    │   └── useCreateAssetAccountAvailableSetting.ts
    ├── types/
    │   └── assetAccount.ts
    └── pages/
        └── AssetAccountDetailPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

### 28.72 フロントエンドで履歴を直接更新しない

ACC-006成功後に、React側で

```text
旧設定.endYearMonth更新
+
新設定追加
```

を正式な履歴状態として手動確定しない。

サーバー側でトランザクション・ロック・制約を経て確定した状態をACC-005から再取得する。

---

### 28.73 フロントエンドで前月を業務確定しない

画面表示のために、

```text
2026-08
    ↓
2026-07
```

を計算することはできる。

ただし、旧設定の正式な`endYearMonth`をフロントエンド側で確定しない。

正式状態はサーバー側の処理結果を正とする。

---

### 28.74 フロントエンドで行わないこと

ACC-006のReact・TypeScript実装では、以下をフロントエンドの責務としない。

- 利用者境界の最終保証
- 資産口座存在確認の最終保証
- 論理削除判定
- 履歴0件判定
- 履歴整合性の最終保証
- 期間重複判定
- 期間欠落判定
- 継続中設定の最終特定
- 行ロック
- 同時更新制御
- UNIQUE制約判定
- 旧設定のDB更新
- 新設定のDB登録
- トランザクション制御
- 履歴の自動修復
- 過去履歴の直接編集

フロントエンドは、

```text
現在状態・履歴取得
    ↓
利用者入力
    ↓
ACC-006実行
    ↓
結果表示
    ↓
ACC-003 / ACC-005再取得
```

に責務を限定する。

---

## 29. 設計上の補足

### 29.1 利用可能資産設定を履歴管理する理由

利用可能資産区分は、単なる現在値ではなく、対象年月によって変化する業務状態として扱う。

例えば、

```text
2026-01 ～ 2026-07
isAvailable = true

2026-08 ～ NULL
isAvailable = false
```

のように、どの年月でどの状態だったかを後から判定できる必要がある。

そのため、

```text
asset_accounts.is_available
```

のような固定属性として保持せず、

```text
asset_account_available_settings
```

で履歴管理する。

---

### 29.2 現在設定を直接上書きしない理由

例えば、現在設定が

```text
2026-01 ～ NULL
isAvailable = true
```

の場合に、

```text
is_available = false
```

へ直接UPDATEすると、

```text
2026-01以降
常にfalseだった
```

という意味に変わってしまう。

過去状態を保持するため、

```text
旧設定を終了
+
新設定を追加
```

する方式とする。

---

### 29.3 ACC-006を登録APIとする理由

ACC-006では、新しい利用可能資産設定を1件追加する。

概念的には、

```text
旧設定
    UPDATE

新設定
    INSERT
```

となるが、業務上の主目的は

```text
新しい設定を登録する
```

ことである。

そのため、

```http
POST /api/v1/asset-accounts/{assetAccountId}/available-settings
```

を使用する。

---

### 29.4 PATCHを採用しない理由

ACC-006は、既存の現在設定1件だけを部分更新する操作ではない。

実際には、

```text
現在設定を終了する
+
新設定を追加する
```

という履歴変更を行う。

そのため、

```http
PATCH /api/v1/asset-account-available-settings/{settingId}
```

のような既存履歴直接更新APIとはしない。

---

### 29.5 settingIdをURLへ含めない理由

ACC-006では、どの現在設定を終了するかをクライアントに指定させない。

以下のようなURLにはしない。

```http
POST /api/v1/asset-account-available-settings/{settingId}/next
```

現在設定は、サーバー側で

```text
asset_account_id
+
end_year_month IS NULL
```

から特定する。

これにより、クライアントが過去設定や誤った設定を直接変更することを防止する。

---

### 29.6 資産口座配下のURLとする理由

利用可能資産設定は、単独で存在するものではなく、必ず資産口座に帰属する。

そのため、

```text
asset-accounts
    ↓
available-settings
```

という親子関係をURLへ表現する。

```http
POST /api/v1/asset-accounts/{assetAccountId}/available-settings
```

とする。

---

### 29.7 ACC-005と同じURLを使用する理由

ACC-005とACC-006は、同じ利用可能資産設定リソースを扱う。

そのため、

```text
GET
    → 履歴取得

POST
    → 新設定登録
```

として、HTTPメソッドによって操作を表現する。

```text
ACC-005
GET
/api/v1/asset-accounts/{assetAccountId}/available-settings

ACC-006
POST
/api/v1/asset-accounts/{assetAccountId}/available-settings
```

とする。

---

### 29.8 ACC-004へisAvailableを含めない理由

ACC-004は、資産口座の通常属性更新APIである。

更新対象は、

```text
name
assetType
```

とする。

一方、

```text
isAvailable
```

は、履歴を伴う状態変更である。

そのため、

```text
通常属性
    → ACC-004

利用可能資産区分
    → ACC-006
```

と責務を分離する。

---

### 29.9 ACC-005との責務分離

ACC-005は、履歴を参照する。

ACC-006は、履歴を変更する。

概念的には、

```text
ACC-005
    → Query

ACC-006
    → Command
```

とする。

履歴取得処理の中で設定変更を行わない。

---

### 29.10 旧設定の終了年月をクライアント指定させない理由

ACC-006では、旧設定の

```text
endYearMonth
```

をRequestから受け付けない。

旧設定終了年月は、

```text
新設定.startYearMonthの前月
```

として一意に決定できるためである。

例えば、

```text
新設定開始
2026-08
```

なら、

```text
旧設定終了
2026-07
```

となる。

---

### 29.11 新設定のendYearMonthをNULLとする理由

ACC-006で登録する新しい設定は、その時点で最新の設定となる。

そのため、

```text
end_year_month = NULL
```

として登録する。

終了年月は、次回ACC-006でさらに新しい設定を追加するときに確定する。

---

### 29.12 9999-12などを使わない理由

継続中設定を表すために、

```text
9999-12
2099-12
```

などの特殊年月を使用しない。

終了年月未確定という状態を自然に表現するため、

```text
NULL
```

を使用する。

---

### 29.13 新設定開始年月を現在設定開始年月より後にする理由

現在設定が

```text
2026-08 ～ NULL
```

の場合に、新設定を

```text
2026-08 ～
```

から開始すると、旧設定の終了年月が

```text
2026-07
```

となり、

```text
startYearMonth
    >
endYearMonth
```

になる。

そのため、

```text
new.startYearMonth
    >
current.startYearMonth
```

を必須とする。

---

### 29.14 過去履歴への挿入を許可しない理由

例えば、

```text
2026-01 ～ 2026-06
true

2026-07 ～ NULL
false
```

という履歴に対して、

```text
2026-04
```

から新設定を入れる場合、既存履歴を複数件再構成する必要がある。

Phase1ではその複雑性を扱わず、

```text
最新履歴の後ろへ
新設定を追加する
```

操作に限定する。

---

### 29.15 同じisAvailableを登録できない理由

現在設定が

```text
isAvailable = true
```

の場合に、

```text
isAvailable = true
```

の新設定を登録しても、業務状態は変わらない。

例えば、

```text
2026-01 ～ 2026-07
true

2026-08 ～ NULL
true
```

のような意味のない履歴分割となる。

そのため、

```text
current.isAvailable
    !=
request.isAvailable
```

を必須とする。

---

### 29.16 NO_CHANGEを正常扱いしない理由

現在状態と同じ設定を送信した場合に、

```text
201 Created
```

として新しい履歴を作成しない。

また、何もせず

```text
200 OK
```

として成功扱いすることもしない。

ACC-006は新しい業務状態を履歴として登録するAPIであるため、変更がない場合は

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NO_CHANGE
```

として扱う。

---

### 29.17 履歴が0件の場合に自動作成しない理由

ACC-002では、資産口座登録時に初期設定を必ず作成する。

そのため、利用中資産口座で

```text
利用可能資産設定履歴
0件
```

は通常状態ではない。

ACC-006で初期設定を自動補完すると、既存データ不整合を隠蔽することになる。

そのため、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

として扱う。

---

### 29.18 既存履歴不整合時に追加しない理由

既に、

```text
期間重複
期間欠落
複数継続中設定
```

などの不整合が存在する場合に新しい履歴を追加すると、状態がさらに複雑になる。

そのため、ACC-006実行前に既存履歴を検証し、不整合があれば更新を行わない。

---

### 29.19 ACC-005と同じValidatorを使用する理由

ACC-005とACC-006で利用可能資産設定履歴の正常条件が異なると、APIごとに履歴の解釈が変わってしまう。

そのため、

```text
AvailableSettingHistoryValidator
```

を共通利用し、

```text
ACC-005
    → 参照前に検証

ACC-006
    → 更新前に検証
```

とする。

---

### 29.20 Validatorを共通化することで得られる効果

履歴整合性ルールを1か所へ集約することで、

- 初期開始年月
- 期間大小
- 期間重複
- 期間欠落
- 継続中設定数
- 最新設定

について、APIごとの判定差異を防止できる。

---

### 29.21 Validatorで履歴を修復しない理由

Validatorは、

```text
正常
または
不正
```

を判定する責務とする。

以下のような処理は行わない。

```text
期間欠落
    → 前設定を延長

複数継続設定
    → 古い設定を終了

最新設定終了済み
    → NULLへ戻す
```

検証と修復を同じ責務にしない。

---

### 29.22 トランザクションを必須とする理由

ACC-006は、

```text
旧設定UPDATE
+
新設定INSERT
```

という複数DB更新を伴う。

片方だけ成功すると、履歴が不整合になる。

そのため、両処理を

```php
DB::transaction()
```

内で実行する。

---

### 29.23 Repository単位でトランザクションを張らない理由

ACC-006では、

```text
close()
create()
```

の両方が1つの業務操作である。

各Repositoryメソッド単位で別トランザクションにすると、

```text
close()
    commit

create()
    rollback
```

という中途半端な状態が発生し得る。

そのため、UseCase全体をトランザクション境界とする。

---

### 29.24 lockForUpdateを使用する理由

同一資産口座に対してACC-006が同時実行されると、複数Requestが同じ現在設定を取得する可能性がある。

そのため、現在設定を

```php
lockForUpdate()
```

でロックし、同一履歴への更新を直列化する。

---

### 29.25 履歴全件をロックしない理由

更新対象となる既存レコードは、現在継続中設定1件である。

そのため、過去履歴全件をロックする必要はない。

必要最小限の

```text
現在設定
```

だけをロックし、ロック範囲を小さくする。

---

### 29.26 資産口座行を原則ロックしない理由

ACC-006では、

```text
asset_accounts
```

を更新しない。

利用者境界や利用開始年月確認のために参照するだけである。

そのため、Phase1では資産口座行自体を原則ロックしない。

---

### 29.27 ロック後に再確認する理由

ロック取得前に履歴を確認していても、別Requestが先に変更している可能性がある。

そのため、`lockForUpdate()`取得後の現在設定を基準として、

```text
startYearMonth
isAvailable
```

を再確認する。

古い画面状態や古いQuery結果だけを信頼しない。

---

### 29.28 DB制約も併用する理由

アプリケーション側のロックと業務チェックだけではなく、DB側でも最終整合性を保証する。

少なくとも、

```text
asset_account_id
+
start_year_month
```

を一意とする。

必要に応じて、

```text
asset_account_id
WHERE end_year_month IS NULL
```

の部分UNIQUEインデックスも使用する。

---

### 29.29 部分UNIQUEインデックスを検討する理由

業務ルールでは、同一資産口座に

```text
end_year_month = NULL
```

の設定は1件のみ許可する。

PostgreSQLの部分UNIQUEインデックスを使用すれば、このルールをDBレベルでも保証できる。

概念的には、

```sql
CREATE UNIQUE INDEX
    uq_asset_account_available_settings_current
ON
    asset_account_available_settings (
        asset_account_id
    )
WHERE
    end_year_month IS NULL;
```

とする。

---

### 29.30 ApplicationとDBの二重防御とする理由

概念的には、

```text
UseCase
    → 業務条件

lockForUpdate
    → 同時更新制御

UNIQUE制約
    → 同一開始年月防止

部分UNIQUE
    → 現在設定複数件防止
```

という複数層の整合性保証とする。

DB制約だけ、またはアプリケーションだけに責務を偏らせない。

---

### 29.31 Repositoryを使用する理由

ACC-006では、

```text
UPDATE
INSERT
```

を行う。

そのため、

```text
Query
    → 読み取り

Repository
    → 書き込み
```

という共通方針に従い、状態変更をRepositoryへ集約する。

---

### 29.32 QueryとRepositoryを分離する理由

例えば、

```text
AssetAccountAvailableSettingQuery
    → 履歴取得
    → 現在設定取得

AssetAccountAvailableSettingRepository
    → 現在設定終了
    → 新設定作成
```

とする。

検索条件と状態変更処理を同じクラスへ混在させない。

---

### 29.33 UseCaseを設ける理由

ACC-006には、

```text
資産口座確認
履歴取得
履歴検証
現在設定ロック
開始年月判定
状態差分判定
旧設定終了
新設定登録
```

という複数の処理が存在する。

これらのオーケストレーションをActionへ直接書かず、UseCaseへ集約する。

---

### 29.34 Actionを薄くする理由

Actionは、

```text
HTTP入力
    ↓
Input DTO
    ↓
UseCase
    ↓
Responder
```

の橋渡しに限定する。

業務ルールやDB更新をActionへ持たせない。

ADRパターンとしても、Actionの責務をHTTP境界へ限定する。

---

### 29.35 FormRequestを使用する理由

ACC-006には、

```text
startYearMonth
isAvailable
```

という明確なRequest Bodyがある。

そのため、

- 必須
- NULL禁止
- 型
- `YYYY-MM`
- strict boolean

などのHTTP入力検証をFormRequestへ集約する。

---

### 29.36 DB状態依存の判定をFormRequestへ持たせない理由

例えば、

```text
現在設定開始年月より後か
現在とisAvailableが異なるか
```

は、現在のDB状態に依存する。

これらをFormRequestで判定すると、HTTP入力検証と業務状態判定が混在する。

そのため、UseCase側で扱う。

---

### 29.37 Input DTOを使用する理由

UseCaseがLaravelのRequestへ直接依存しないように、

```text
FormRequest
    ↓
Input DTO
    ↓
UseCase
```

とする。

これにより、HTTP層とアプリケーション層の依存関係を整理する。

---

### 29.38 Mass Assignmentを避ける理由

ACC-006では、クライアント指定可能項目とサーバー管理項目が明確に分かれている。

クライアント指定：

```text
startYearMonth
isAvailable
```

サーバー管理：

```text
asset_account_id
end_year_month
```

そのため、

```php
Model::create(
    $request->all(),
);
```

のようなMass Assignmentは避ける。

---

### 29.39 replicateを使用しない理由

現在設定を

```php
$currentSetting->replicate();
```

して新設定を作る方式は、意図しない属性まで引き継ぐ可能性がある。

新設定は、

```text
asset_account_id
start_year_month
end_year_month
is_available
```

を明示的に指定して作成する。

---

### 29.40 YearMonth処理を文字列演算しない理由

ACC-006では、

```text
開始年月比較
前月計算
年跨ぎ
```

が必要になる。

例えば、

```text
2027-01
    ↓
2026-12
```

を扱う。

そのため、

```text
month - 1
```

のような単純文字列・数値処理に依存しない。

---

### 29.41 YearMonth Value Objectを検討する理由

利用可能資産設定だけでなく、Life Plannerでは月単位の概念を多く扱う。

そのため、将来的に

```text
YearMonth
```

をValue Objectとして導入すると、

- 比較
- 前月
- 翌月
- 表示形式
- 妥当性

を共通化できる。

---

### 29.42 Phase1でYearMonth Value Objectを必須としない理由

Value Objectの導入は設計上有効だが、利用箇所が少ない段階ではクラス数だけが増える可能性がある。

Phase1では、Laravel標準機能等で十分安全に扱える場合は必須としない。

必要性が高まった段階でリファクタリングしてよい。

---

### 29.43 成功レスポンスで新設定だけ返す理由

ACC-006では、新しく作成された設定を返却する。

旧設定も更新されるが、旧設定まで含めるとレスポンス責務が履歴全体へ広がる。

履歴全体が必要な場合は、

```text
ACC-005
```

を再取得する。

---

### 29.44 旧設定をレスポンスへ含めない理由

ACC-006成功時に、

```json
{
  "previousSetting": {},
  "newSetting": {}
}
```

のような複合レスポンスにはしない。

ACC-006は新規登録された利用可能資産設定リソースを返す。

履歴表示はACC-005へ委譲する。

---

### 29.45 201 Createdを返す理由

ACC-006では、新しい

```text
asset_account_available_settings
```

レコードを作成する。

そのため、

```text
201 Created
```

を返却する。

旧設定UPDATEを含むことだけを理由に`200 OK`とはしない。

---

### 29.46 Locationヘッダーを必須としない理由

`201 Created`では`Location`ヘッダーを利用できる。

ただし、Phase1では利用可能資産設定の個別取得APIを持たない。

そのため、新規設定を指す明確な個別URLが存在せず、`Location`を必須とはしない。

---

### 29.47 Result DTOを使用する理由

Eloquent ModelをそのままResponderへ渡すと、HTTP層がDB構造に依存しやすい。

そのため、

```text
UseCase
    ↓
Result DTO
    ↓
Resource
```

とし、API返却に必要な値だけを渡す。

---

### 29.48 API Resourceを使用する理由

DBでは、

```text
start_year_month
end_year_month
is_available
```

だが、APIでは、

```text
startYearMonth
endYearMonth
isAvailable
```

とする。

この変換責務をAPI Resourceへ集約する。

---

### 29.49 Responderを使用する理由

Responderは、

```text
Result DTO
    ↓
Resource
    ↓
201 Created
```

へ変換する。

UseCaseにHTTPステータスやJSONレスポンス生成を持たせない。

---

### 29.50 ACC-006成功後にACC-005を再取得する理由

ACC-006によって、

```text
旧設定.endYearMonth
+
新設定
```

の両方が変化する。

成功レスポンスでは新設定だけを返すため、履歴全体の最新状態は

```text
ACC-005
```

から再取得する。

---

### 29.51 ACC-006成功後にACC-003を再取得する理由

ACC-003は、現在年月における

```text
isAvailable
```

を返却する。

ACC-006によって現在設定が変わった可能性があるため、ACC-003も再取得対象とする。

---

### 29.52 将来開始設定でもACC-005を再取得する理由

新設定の適用開始が将来であっても、履歴自体は登録直後に変化している。

そのため、ACC-005は必ず最新化する。

---

### 29.53 将来開始設定では現在状態が変わらない場合がある

例えば、

```text
現在年月
2026-08

旧設定
2026-01 ～ 2026-09
true

新設定
2026-10 ～ NULL
false
```

では、現在年月の

```text
isAvailable
```

はまだ`true`である。

そのため、React側でACC-006 Request値をそのまま現在状態へ反映しない。

---

### 29.54 Optimistic Updateを必須としない理由

ACC-006には、

- 履歴整合性
- 現在設定確認
- 開始年月判定
- `NO_CHANGE`
- 行ロック
- UNIQUE制約
- トランザクション

が関係する。

サーバー側で拒否される可能性があるため、Phase1ではOptimistic Updateを必須としない。

---

### 29.55 Mutationの自動Retryを行わない理由

ACC-006はPOSTによる更新APIである。

通信切断時に、サーバー側で

```text
成功済み
```

なのか

```text
未実行
```

なのか判断できない可能性がある。

そのため、同一POSTを無条件に自動再送しない。

---

### 29.56 Idempotency-Keyを採用しない理由

Phase1では、専用の

```text
Idempotency-Key
```

を導入しない。

重複や競合については、

```text
現在状態再確認
lockForUpdate
UNIQUE制約
NO_CHANGE判定
```

で防止する。

必要性が高まった場合は、将来的に検討する。

---

### 29.57 通信結果不明時に再取得する理由

ACC-006送信後に通信結果が不明となった場合は、

```text
ACC-005
+
ACC-003
```

を再取得する。

これにより、同じPOSTを無条件再送せずに実際のサーバー状態を確認できる。

---

### 29.58 Query Cacheを手動再構築しない理由

ACC-006成功レスポンスだけでは、旧設定の更新後状態を完全には取得できない。

そのため、

```text
旧設定.endYearMonthを
Reactで計算

+
新設定を配列へ追加
```

する方式を基本としない。

サーバーからACC-005を再取得する。

---

### 29.59 利用者境界をサーバーで最終保証する理由

React側で現在利用者と資産口座の対応を保持していても、Requestは改変できる。

そのため、

```text
userId
+
assetAccountId
```

による利用者境界はLaravel側で必ず確認する。

---

### 29.60 userIdをRequest Bodyへ含めない理由

利用者指定経路を

```text
X-User-Id
```

へ統一する。

Request Bodyにも`userId`を持たせると、

```text
Header user
Body user
```

という2つの値が存在し、不整合時のルールが必要になる。

そのため、Bodyへ含めない。

---

### 29.61 assetAccountIdをRequest Bodyへ含めない理由

対象資産口座はURLで既に表現されている。

そのため、

```text
URL assetAccountId
+
Body assetAccountId
```

という重複指定を避ける。

---

### 29.62 isAvailableを資産口座テーブルへ複製しない理由

現在状態を高速取得する目的で、

```text
asset_accounts.is_available
```

にも値を複製すると、

```text
asset_accounts
+
asset_account_available_settings
```

で同じ意味のデータを二重管理することになる。

Phase1では履歴テーブルを正とし、現在状態も履歴から判定する。

---

### 29.63 過去の月末資産データを更新しない理由

利用可能資産設定を新しく登録しても、既存の

```text
month_end_asset_balances
month_end_holding_values
month_end_asset_snapshots
```

を変更しない。

過去記録そのものと、その年月における利用可能資産区分は別概念として扱う。

---

### 29.64 過去の目的達成判定履歴を更新しない理由

`assessment_histories`は、判定実行時点の結果を履歴として保存する。

ACC-006によって後から設定が変わっても、過去判定結果を自動で書き換えない。

---

### 29.65 利用可能資産設定変更をイベントとして扱わない理由

Phase1では、

```text
AvailableSettingChanged
```

のようなドメインイベントを発行して関連処理を非同期実行する設計は必須としない。

ACC-006の責務は、

```text
履歴整合性を保って
設定を変更する
```

ことに限定する。

---

### 29.66 監査履歴を別途作らない理由

利用可能資産設定自体が時系列履歴となっている。

そのため、Phase1では同じ変更内容を記録する別監査テーブルを追加しない。

運用上の追跡は、通常ログと設定履歴を利用する。

---

### 29.67 履歴削除APIを作らない理由

利用可能資産設定履歴は、過去状態を再現するための業務データである。

そのため、Phase1では

```text
DELETE available-setting
```

のような履歴削除APIを提供しない。

誤登録修正が業務要件として必要になった場合は、履歴訂正方式を別途設計する。

---

### 29.68 過去履歴編集APIを作らない理由

過去履歴を自由に編集すると、

```text
期間重複
期間欠落
初期年月不整合
```

を生みやすい。

Phase1では履歴末尾への追加だけに限定し、履歴全体の整合性を単純に保つ。

---

### 29.69 再有効化専用APIを作らない理由

利用可能資産区分はbooleanであり、

```text
true
false
```

の双方をACC-006で登録できる。

そのため、

```text
enable
disable
```

の専用APIへ分割しない。

---

### 29.70 ACC-006と資産口座無効化を分離する理由

```text
isAvailable = false
```

と

```text
資産口座無効化
```

は意味が異なる。

`isAvailable = false`は、

```text
資産口座は利用中
ただし利用可能資産として扱わない
```

という状態である。

一方、資産口座無効化は、

```text
資産口座自体を
通常の管理対象から外す
```

操作である。

そのため、別APIとして扱う。

---

### 29.71 Phase1では複数設定一括登録を行わない

ACC-006では、1回のRequestで1つの新設定だけを登録する。

以下のような一括登録は行わない。

```json
{
  "settings": [
    {
      "startYearMonth": "2026-08",
      "isAvailable": false
    },
    {
      "startYearMonth": "2027-01",
      "isAvailable": true
    }
  ]
}
```

履歴追加ルールと排他制御を単純に保つためである。

---

### 29.72 将来予約を許可する場合の注意

将来年月からの設定変更を許可する場合、登録直後には

```text
最新履歴
```

と

```text
現在年月に有効な設定
```

が一致しないことがある。

例えば、

```text
現在年月
2026-08

設定A
2026-01 ～ 2026-09
true

設定B
2026-10 ～ NULL
false
```

では、

```text
最新履歴
    → 設定B

現在有効
    → 設定A
```

となる。

---

### 29.73 最新設定と現在設定を同一視しない

将来予約を許可する場合は、

```text
endYearMonth = null
```

だからといって現在年月にその設定が有効とは限らない。

現在年月での有効設定は、

```text
start_year_month
    <= currentYearMonth

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= currentYearMonth
)
```

で判定する。

---

### 29.74 ACC-003は現在年月基準で判定する

ACC-003の

```text
isAvailable
```

は、最新履歴ではなく現在年月に有効な履歴から取得する。

これにより、将来開始設定をACC-006で登録しても、適用開始前に現在状態が切り替わらない。

---

### 29.75 ACC-005は履歴全体を返す

ACC-005では、将来開始設定も含めて履歴全体を返却する。

これにより、利用者は

```text
現在設定
+
将来予約済み設定
```

を履歴として確認できる。

---

### 29.76 将来予約が存在する場合の追加登録

Phase1では、既に将来予約設定が存在する状態でさらに履歴途中へ新しい設定を挿入することは扱わない。

例えば、

```text
現在
2026-08

設定A
2026-01 ～ 2026-09

設定B
2026-10 ～ NULL
```

が存在する状態で、

```text
2026-09から別設定
```

を追加するような操作は対象外とする。

---

### 29.77 将来予約機能を広げすぎない

Phase1では、以下のような高度な予約管理機能を追加しない。

- 複数将来設定予約
- 将来予約変更
- 将来予約取消
- 履歴途中挿入
- 予約優先順位
- 日単位適用
- 時刻単位適用

必要になった段階で別ユースケースとして設計する。

---

### 29.78 Phase1では設計を広げすぎない

ACC-006では、以下を対象外とする。

- 過去履歴編集
- 過去履歴削除
- 履歴途中挿入
- 複数設定一括登録
- 履歴自動修復
- 資産口座通常属性更新
- 資産口座無効化
- 月末資産データ更新
- 過去判定履歴再計算
- Idempotency-Key
- 楽観ロック
- ETag
- ドメインイベント
- 専用監査履歴

Phase1では、

```text
現在の履歴末尾を終了し
新しい利用可能資産設定を
安全に1件追加する
```

ことへ責務を限定する。

---

## 30. 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [資産口座API詳細](./asset-accounts.md)
- [保有商品API詳細](./holding-assets.md)
- [月末資産API詳細](./month-end-assets.md)
- [資産状況・資産推移API詳細](./asset-views.md)
- [目的達成判定API詳細](./assessments.md)
- [CSVインポートAPI詳細](./csv-imports.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
