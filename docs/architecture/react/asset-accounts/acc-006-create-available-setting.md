# ACC-006 利用可能資産設定登録

## 1. 概要

ACC-006 利用可能資産設定登録では、指定された資産口座について、新しい利用可能資産設定を登録する画面・フロントエンド処理を扱う。

ACC-006は、資産口座の通常属性を更新するAPIではなく、利用可能資産区分の変更を年月単位の履歴として追加するためのMutationとして利用する。

React側では、ACC-003 資産口座詳細取得から現在の利用可能資産区分を取得し、必要に応じてACC-005 利用可能資産設定履歴取得から現在設定や過去履歴を確認したうえで、利用者が指定した適用開始年月と利用可能資産区分をACC-006へ送信する。

概念的な処理フローは、以下とする。

```text
ACC-003
資産口座詳細取得
    ↓
現在のisAvailable確認
    ↓
必要に応じてACC-005
利用可能資産設定履歴取得
    ↓
利用者入力
    ├─ startYearMonth
    └─ isAvailable
    ↓
ACC-006 Mutation
    ↓
POST
/api/v1/asset-accounts/{assetAccountId}/available-settings
    ↓
成功
    ↓
関連Query Cache無効化
    ├─ ACC-003 資産口座詳細
    ├─ ACC-005 利用可能資産設定履歴
    └─ ACC-001 資産口座一覧
        ※ isAvailableを返却する場合
    ↓
最新状態を再取得
```

---

## 2. テスト観点

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

### 2.1 正常系：trueからfalse

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

### 2.2 正常系：falseからtrue

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

### 2.3 成功レスポンス

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

### 2.4 idの型

新規設定IDがDB上で`bigint`でも、

```json
{
  "id": "15"
}
```

のようにstringとして返却されることを確認する。

---

### 2.5 startYearMonth未指定

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

### 2.6 startYearMonth = null

以下を送信する。

```json
{
  "startYearMonth": null,
  "isAvailable": false
}
```

バリデーションエラーとなり、DB更新されないことを確認する。

---

### 2.7 startYearMonth形式不正

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

### 2.8 isAvailable未指定

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

### 2.9 isAvailable = null

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": null
}
```

バリデーションエラーとなること。

---

### 2.10 isAvailableが文字列

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

### 2.11 isAvailableが数値

以下を送信する。

```json
{
  "startYearMonth": "2026-08",
  "isAvailable": 0
}
```

boolean以外として拒否されることを確認する。

---

### 2.12 X-User-Id未指定

`X-User-Id`を指定しない。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

となること。

---

### 2.13 X-User-Id形式不正

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

### 2.14 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 2.15 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 2.16 assetAccountId形式不正

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

### 2.17 資産口座不存在

存在しない`assetAccountId`を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 2.18 他利用者の資産口座

User AとしてUser Bの資産口座を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの`asset_account_available_settings`が一切変更されないことを確認する。

---

### 2.19 論理削除済み資産口座

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

### 2.20 履歴0件

利用中資産口座に対して利用可能資産設定履歴を0件とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

となること。

設定を自動作成しないこと。

---

### 2.21 初期開始年月不整合

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

### 2.22 既存履歴の期間重複

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

### 2.23 既存履歴の期間欠落

以下を用意する。

```text
2026-01 ～ 2026-05
true

2026-07 ～ NULL
false
```

履歴不整合となること。

---

### 2.24 継続中設定0件

すべての設定に`end_year_month`が存在する状態を用意する。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

となること。

---

### 2.25 継続中設定複数件

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

### 2.26 startYearMonthが資産口座利用開始年月より前

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

### 2.27 startYearMonthが現在設定開始年月と同じ

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

### 2.28 startYearMonthが現在設定より前

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

### 2.29 1か月後から変更

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

### 2.30 年跨ぎ

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

### 2.31 同じisAvailable

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

### 2.32 同一startYearMonth重複

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

### 2.33 UNIQUE制約

DBレベルで、

```text
asset_account_id
+
start_year_month
```

が重複できないことを確認する。

並行実行などで制約違反が発生した場合は、PostgreSQL例外がAPIへそのまま露出しないこと。

---

### 2.34 継続中設定部分UNIQUE制約

部分UNIQUEインデックスを採用する場合は、同一資産口座に

```text
end_year_month IS NULL
```

の設定を2件作成できないことを確認する。

---

### 2.35 旧設定UPDATE

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

### 2.36 新設定INSERT

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

### 2.37 旧設定更新後に新設定INSERT失敗

新設定INSERTで意図的に例外を発生させる。

以下を確認する。

* APIはエラーとなる
* トランザクションがロールバックされる
* 旧設定の`end_year_month`が元の`NULL`へ戻る
* 新設定が存在しない

---

### 2.38 旧設定UPDATE失敗

旧設定UPDATEで例外を発生させる。

以下を確認する。

* 新設定をINSERTしない
* トランザクションがロールバックされる
* 履歴状態が実行前と同じ

---

### 2.39 Transaction全体

正常時は、

```text
旧設定UPDATE
+
新設定INSERT
```

の両方がコミットされること。

異常時は、両方とも確定しないことを確認する。

---

### 2.40 同時実行

同一資産口座に対して複数のACC-006を並行実行する。

以下を確認する。

* 現在設定への`lockForUpdate()`が機能する
* 同時に同じ旧設定を更新しない
* 継続中設定が複数件にならない
* 履歴期間が重複しない
* 2件目は最新状態で業務条件を再評価する

---

### 2.41 同時に異なる開始年月

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

### 2.42 同時に同じ開始年月

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

### 2.43 他の関連テーブル非更新

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

### 2.44 過去の月末資産非更新

利用可能資産設定変更によって、既存の

```text
month_end_asset_balances
month_end_holding_values
month_end_asset_snapshots
```

が更新されないことを確認する。

---

### 2.45 過去判定履歴非更新

ACC-006実行によって、

```text
assessment_histories
```

の既存レコードが変更されないことを確認する。

---

### 2.46 更新対象外項目

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

### 2.47 currentSettingIdを送信

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

### 2.48 userIdを送信

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

### 2.49 assetAccountIdをBodyへ送信

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

### 2.50 正常レスポンス契約

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

### 2.51 返却しない情報

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

### 2.52 エラーレスポンス契約

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

### 2.53 エラー時の副作用

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

### 2.54 INTERNAL_SERVER_ERROR

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

## 2. React・TypeScriptでの利用

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

### 2.1 TypeScript型

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

### 2.2 正常レスポンス型

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

### 2.3 ACC-005と共通型を使用してよい

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

### 2.4 assetAccountId

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

### 2.5 userIdをAPI Client引数へ含めない

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

### 2.6 X-User-Id

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

### 2.7 フォーム型

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

### 2.8 startYearMonth

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

### 2.9 startYearMonthをDateへ変換して保持しなくてよい

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

### 2.10 isAvailable

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

### 2.11 booleanを文字列のまま送信しない

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

### 2.12 現在状態を初期表示する

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

### 2.13 現在状態と異なる値だけ選択させてもよい

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

### 2.14 ただしサーバー側判定を残す

フロントエンドで同一状態を選択できないようにしても、サーバー側の

```text
NO_CHANGE
```

判定は必要とする。

別タブや別リクエストによって画面表示後に現在状態が変更される可能性があるためである。

---

### 2.15 現在設定開始年月

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

### 2.16 ACC-005から現在設定を取得する

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

### 2.17 現在設定をフロントエンドで最終保証しない

ACC-005の正常レスポンスから現在設定を表示できるが、ACC-006実行時の最終判定はバックエンド側で行う。

フロントエンドで取得した`currentSetting`を更新対象IDとしてRequestへ送信しない。

---

### 2.18 settingIdを送信しない

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

### 2.19 endYearMonthを送信しない

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

### 2.20 旧設定を編集させない

ACC-005で表示した過去の利用可能資産設定に対して、

```text
編集
削除
```

ボタンをPhase1では設けない。

ACC-006は、現在履歴の後ろへ新しい設定を追加する操作に限定する。

---

### 2.21 API Client

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

### 2.22 Mutationとして扱う

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

### 2.23 Mutation Hook

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

### 2.24 ACC-005のQuery Cacheを無効化する

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

### 2.25 ACC-003のQuery Cacheを無効化する

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

### 2.26 ACC-001のQuery Cache

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

### 2.27 将来開始設定でも履歴Cacheは無効化する

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

### 2.28 将来開始設定とACC-003

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

### 2.29 成功レスポンスだけで履歴を再構築しない

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

### 2.30 Optimistic Update

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

### 2.31 Mutationの自動Retry

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

### 2.32 通信結果不明時

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

### 2.33 二重送信防止

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

### 2.34 フォーム送信

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

### 2.35 クライアント側バリデーション

送信前に、最低限以下を確認してよい。

- `startYearMonth`が入力されている
- `YYYY-MM`形式である
- `isAvailable`がbooleanとして確定している

ただし、サーバー側バリデーションを省略しない。

---

### 2.36 現在設定開始年月との簡易チェック

ACC-005から現在設定を取得済みであれば、

```text
request.startYearMonth
    >
currentSetting.startYearMonth
```

をフロントエンドでも事前確認してよい。

これにより、明らかに不正な入力を送信前に防げる。

---

### 2.37 業務判定の最終保証はサーバー側とする

フロントエンドで開始年月をチェックしても、ACC-006実行直前に別リクエストで現在設定が変わる可能性がある。

そのため、

```text
START_YEAR_MONTH_INVALID
```

の最終判定はLaravel側で行う。

---

### 2.38 NO_CHANGEの事前抑止

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

### 2.39 成功時

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

### 2.40 成功メッセージ

正常終了後は、必要に応じて

```text
利用可能資産設定を変更しました。
```

などの完了メッセージを表示する。

---

### 2.41 成功後の画面

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

### 2.42 VALIDATION_ERROR

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

### 2.43 startYearMonthのフィールドエラー

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

### 2.44 isAvailableのフィールドエラー

通常のUIでは、boolean以外を送信しない構成にする。

それでも

```text
VALIDATION_ERROR
```

となった場合は、設定フォーム全体のエラーとして扱ってもよい。

---

### 2.45 ASSET_ACCOUNT_NOT_FOUND

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

### 2.46 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-006専用フォームエラーにはしない。

---

### 2.47 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、利用者コンテキストに関する共通エラーとして扱う。

---

### 2.48 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択中の利用者が有効ではない状態として共通処理する。

---

### 2.49 INVALID_ASSET_ACCOUNT_ID

通常画面ではACC-001やACC-003から取得した有効なIDを使用するため、発生頻度は低い。

発生した場合は、不正なURLまたは画面状態として扱う。

---

### 2.50 HISTORY_NOT_FOUND

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

### 2.51 HISTORY_INVALID

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

### 2.52 START_YEAR_MONTH_INVALID

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

### 2.53 NO_CHANGE

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

### 2.54 NO_CHANGE受信後は再取得してよい

画面上では異なる状態を選択したつもりでも、別リクエストによって現在状態が変化していた可能性がある。

そのため、`NO_CHANGE`受信後に

```text
ACC-003
ACC-005
```

を再取得して最新状態を表示してよい。

---

### 2.55 ALREADY_EXISTS

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

### 2.56 409 Conflict受信後

以下の409系エラーでは、

```text
START_YEAR_MONTH_INVALID
NO_CHANGE
ALREADY_EXISTS
```

必要に応じてACC-005を再取得する。

現在の履歴状態が画面表示時から変化している可能性があるためである。

---

### 2.57 INTERNAL_SERVER_ERROR

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

### 2.58 エラー時のフォーム値

ACC-006が失敗した場合は、原則として利用者が入力した

```text
startYearMonth
isAvailable
```

を保持する。

入力内容を自動的に初期化しない。

---

### 2.59 再取得後にフォーム値を見直す

409系エラーなどでACC-005を再取得した場合、現在設定が変化している可能性がある。

その場合は、現在のフォーム値が最新状態に対して有効かを再評価する。

---

### 2.60 過去履歴編集UIを設けない

ACC-005で表示した過去履歴に対して、

```text
編集
削除
```

の操作をACC-006へ結び付けない。

ACC-006は新しい設定追加専用とする。

---

### 2.61 資産口座編集UIとの分離

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

### 2.62 資産口座無効化との分離

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

### 2.63 Pageの責務

資産口座詳細または利用可能資産設定Pageでは、主に以下を担当する。

- URLから`assetAccountId`取得
- ACC-003による現在状態取得
- ACC-005による履歴取得
- ACC-006成功後の画面更新
- ページ単位のエラー表示

HTTP通信の詳細はAPI ClientやHookへ委譲する。

---

### 2.64 Form Componentの責務

利用可能資産設定フォームでは、主に以下を担当する。

- `startYearMonth`入力
- `isAvailable`選択
- クライアント側入力チェック
- フィールドエラー表示
- submitイベント通知

API Clientを直接呼び出さない構成としてよい。

---

### 2.65 Mutation Hookの責務

Mutation Hookでは、主に以下を担当する。

```text
ACC-006実行
Mutation状態管理
成功時Query invalidate
```

画面固有の表示レイアウトや業務文言をMutation Hookへ持たせない。

---

### 2.66 API Clientの責務

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

### 2.67 QueryとMutationを分離する

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

### 2.68 Query Key

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

### 2.69 利用者切替時

Phase1では、`X-User-Id`によって操作対象利用者を切り替える。

利用者切替時に、前利用者の

```text
ACC-003
ACC-005
```

のQuery Cacheを新しい利用者へ誤表示しないようにする。

---

### 2.70 Query KeyへuserIdを含めてもよい

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

### 2.71 概念的なディレクトリ構成

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

### 2.72 フロントエンドで履歴を直接更新しない

ACC-006成功後に、React側で

```text
旧設定.endYearMonth更新
+
新設定追加
```

を正式な履歴状態として手動確定しない。

サーバー側でトランザクション・ロック・制約を経て確定した状態をACC-005から再取得する。

---

### 2.73 フロントエンドで前月を業務確定しない

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

### 2.74 フロントエンドで行わないこと

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

## 3. 関連ドキュメント

- [ACC-006 API詳細設計](../../../api/details/asset-accounts/acc-006-create-available-setting.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Reactアーキテクチャ共通設計](../react-architecture.md)
- [資産口座 Reactアーキテクチャ設計](./README.md)
- [ACC-006 テスト設計](../../../tests/asset-accounts/acc-006-create-available-setting.md)
- [資産口座 テスト設計](../../../tests/asset-accounts/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)