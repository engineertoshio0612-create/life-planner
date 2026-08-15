# ASM-004 判定履歴詳細取得

### 1 概要

本ドキュメントでは、
ASM-004 判定履歴詳細取得APIに対する
バックエンドのテスト観点を定義する。

ASM-004では、
操作対象利用者に紐づく
指定された`assessment_histories`を参照し、
保存済みの目的達成判定結果および
判定実行時点の計算根拠を取得する。

テストでは、
単にHTTPステータスやレスポンス形式を確認するだけではなく、
主に以下を確認する。

* 指定された判定履歴を正しい利用者境界で取得できること
* 達成可能・達成不可・判定不可の保存済み判定結果が正しく返却されること
* 判定不可の場合に保存済みの判定不可理由が正しく返却されること
* 判定実行時点の利用可能資産額、手取り収入、必要支出額、必要生活防衛資金などの計算根拠が正しく返却されること
* 現在の目的、資産状況、手取り収入、利用可能資産設定などから判定結果や計算根拠を再計算しないこと
* 目的が現在無効化されている場合でも過去の判定履歴を取得できること
* 判定対象となった月末資産状況が現在未確定となっている場合でも過去の判定履歴を取得できること
* 同一目的に複数の判定履歴が存在しても、指定された履歴のみ取得されること
* 他利用者の判定履歴を取得できず、その存在も外部から判別できないこと
* DB上の値がAPI仕様に従った型およびcamelCaseへ正しく変換されること
* 詳細取得によって判定履歴やその他の業務データが変更されないこと
* API共通方針に従った正常レスポンス・エラーレスポンスとなること

特に、
ASM-004は目的達成判定を再実行するAPIではなく、
過去に保存された判定履歴について、

```text
判定実行時点で
どのような条件から
どのような判定結果になったか
```

を参照するためのAPIである。

そのため、
判定履歴保存後に
目的、
月末資産状況、
月末資産残高、
商品別月末評価額、
利用可能資産設定、
手取り収入などが変更されていた場合でも、
現在の業務データを使用して再計算せず、
保存済みの判定結果および計算根拠を
そのまま返却することを確認する。

また、
本APIは参照専用の冪等なAPIであるため、
実行によって目的達成判定が再実行されたり、
判定履歴や関連する業務データが
登録・更新・削除されたりしないことを確認する。

ASM-004のテストでは、
目的達成判定ロジックそのものではなく、

```text
保存済み判定履歴を
正しい利用者境界で取得し、
保存時点の判定結果と計算根拠を
変更せずAPI形式へ変換して返却できること
```

を中心的なテスト対象とする。

---

### 2.1 正常取得

操作対象利用者に属する判定履歴を指定した場合、正常に詳細を取得できることを確認する。

例：

```text
X-User-Id = 1
assessmentHistoryId = 100

assessmentHistory 100
    ↓
User 1に所属
```

期待結果：

```http
200 OK
```

レスポンスの

```text
data.id
```

には、

```text
"100"
```

が設定されること。

---

### 2.2 レスポンス基本項目

正常取得時に、ASM-004で定義したレスポンス項目が正しく返却されることを確認する。

主に、

```text
id
objectiveId
objectiveName
targetYearMonth
assessmentResult
availableAssetAmount
requiredAmount
averageNetIncome
requiredEmergencyFund
remainingAmount
notAssessableReason
assessedAt
```

を確認する。

具体的な項目は、ASM-002の保存仕様および`assessment_histories`のテーブル定義を正とする。

---

### 2.3 ID型

DB上で`bigint`として保持されているIDが、APIレスポンスではstringとして返却されることを確認する。

例：

```json
{
  "id": "100",
  "objectiveId": "10"
}
```

以下のようなnumber型にならないことを確認する。

```json
{
  "id": 100
}
```

---

### 2.4 金額型

金額項目がintegerとして返却されることを確認する。

例：

```json
{
  "availableAssetAmount": 2500000,
  "requiredAmount": 1000000,
  "averageNetIncome": 20000
}
```

以下のような表示用文字列にならないことを確認する。

```json
{
  "availableAssetAmount": "2,500,000円"
}
```

---

### 2.5 targetYearMonth

判定対象年月が、

```text
YYYY-MM
```

形式で返却されることを確認する。

例：

```json
{
  "targetYearMonth": "2026-08"
}
```

---

### 2.6 assessedAt

判定実行日時が、API共通方針で定めた日時形式で返却されることを確認する。

例：

```json
{
  "assessedAt": "2026-08-31T12:00:00+09:00"
}
```

---

### 2.7 達成可能

保存済み判定結果が

```text
ACHIEVABLE
```

の場合、その値が変更されずに返却されることを確認する。

期待例：

```json
{
  "assessmentResult": "ACHIEVABLE"
}
```

---

### 2.8 達成不可

保存済み判定結果が

```text
NOT_ACHIEVABLE
```

の場合、正常レスポンスとして返却されることを確認する。

期待結果：

```http
200 OK
```

判定結果そのものをAPIエラーとして扱わないことを確認する。

---

### 2.9 判定不可

保存済み判定結果が

```text
NOT_ASSESSABLE
```

の場合も、

```http
200 OK
```

となることを確認する。

以下のようなHTTPエラーにしない。

```text
400
409
422
```

---

### 2.10 判定不可理由

判定不可理由が保存されている場合、その理由コードが正しく返却されることを確認する。

例：

```json
{
  "assessmentResult": "NOT_ASSESSABLE",
  "notAssessableReason": "NET_INCOME_INSUFFICIENT"
}
```

判定不可理由のコード値は、ASM-001・ASM-002との共通定義を正とする。

---

### 2.11 判定可能時のnotAssessableReason

判定結果が

```text
ACHIEVABLE
```

または

```text
NOT_ACHIEVABLE
```

の場合、仕様に従って

```json
{
  "notAssessableReason": null
}
```

となることを確認する。

---

### 2.12 保存済み計算根拠

`assessment_histories`へ保存された計算根拠が、そのままレスポンスへ反映されることを確認する。

例えば、

```text
availableAssetAmount
    = 2,500,000

averageNetIncome
    = 20,000

requiredEmergencyFund
    = 900,000

remainingAmount
    = 600,000
```

が保存されている場合、同じ値が返却されることを確認する。

---

### 2.13 現在の資産状況から再計算しない

判定履歴保存後に現在の資産状況を変更しても、ASM-004の判定結果および保存済み計算根拠が変化しないことを確認する。

例：

```text
判定時

availableAssetAmount
    = 2,000,000

    ↓

判定履歴保存

    ↓

現在

資産残高を変更

現在の利用可能資産額
    = 3,000,000

    ↓

ASM-004
```

期待結果：

```text
availableAssetAmount
    = 2,000,000
```

現在値の

```text
3,000,000
```

へ変更されないこと。

---

### 2.14 現在の手取り収入から再計算しない

判定履歴保存後に`net_incomes`が変更・追加されても、保存済みの

```text
averageNetIncome
```

が変化しないことを確認する。

ASM-004実行時に直近3か月平均を再計算しないことを確認する。

---

### 2.15 現在の利用可能資産設定を使用しない

判定履歴保存後に

```text
asset_account_available_settings
```

へ新しい設定が登録されても、過去の

```text
availableAssetAmount
```

が変化しないことを確認する。

---

### 2.16 現在の月末残高を使用しない

判定履歴保存後に

```text
month_end_asset_balances
```

の状態が変化しても、ASM-004で判定結果を再計算しないことを確認する。

---

### 2.17 現在の商品別月末評価額を使用しない

判定履歴保存後に

```text
month_end_holding_values
```

の状態が変化しても、ASM-004の保存済み判定結果へ影響しないことを確認する。

---

### 2.18 目的が現在無効化済み

判定実行後に関連する目的が無効化された場合でも、判定履歴を取得できることを確認する。

```text
判定履歴保存
    ↓
目的無効化
    ↓
ASM-004
```

期待結果：

```http
200 OK
```

以下のようなエラーにならないこと。

```text
OBJECTIVE_DISABLED
ASSESSMENT_HISTORY_NOT_FOUND
```

---

### 2.19 月末資産状況が現在未確定

判定履歴保存後に関連する月末資産状況が確定解除された場合でも、判定履歴を取得できることを確認する。

```text
判定履歴保存
    ↓
月末資産状況確定解除
    ↓
ASM-004
```

期待結果：

```http
200 OK
```

現在の`confirmed`を取得可否条件にしていないことを確認する。

---

### 2.20 複数の判定履歴

同一目的について複数回判定している場合でも、指定した`assessmentHistoryId`に対応する履歴だけが返却されることを確認する。

例：

```text
Objective 10
    ├─ History 100
    ├─ History 101
    └─ History 102
```

```http
GET /api/v1/assessment-histories/101
```

の場合は、History 101の保存済み情報を返却すること。

---

### 2.21 過去履歴を最新履歴で上書きしない

同一目的について再判定が行われた後でも、古い`assessmentHistoryId`を指定すれば古い判定結果を取得できることを確認する。

```text
History 100
ACHIEVABLE

    ↓

再判定

History 101
NOT_ACHIEVABLE
```

この状態で、

```http
GET /api/v1/assessment-histories/100
```

を実行した場合、

```text
ACHIEVABLE
```

が返却されること。

---

### 2.22 X-User-Id未指定

`X-User-Id`を指定しない場合、API共通仕様に従って

```text
USER_CONTEXT_REQUIRED
```

となることを確認する。

判定履歴検索へ進まないことも確認する。

---

### 2.23 X-User-Id形式不正

以下のような`X-User-Id`を指定した場合、

```text
0
-1
abc
1.5
```

```text
INVALID_USER_ID
```

となることを確認する。

---

### 2.24 利用者不存在

形式上有効な`X-User-Id`であっても、該当利用者が存在しない場合は、

```text
USER_NOT_FOUND
```

となることを確認する。

HTTPステータスは、

```http
404 Not Found
```

とする。

---

### 2.25 assessmentHistoryId形式不正

以下のような`assessmentHistoryId`について形式不正として扱われることを確認する。

```text
0
-1
abc
1.5
1e3
100abc
```

期待するエラーコード：

```text
INVALID_ASSESSMENT_HISTORY_ID
```

---

### 2.26 判定履歴不存在

形式上有効な`assessmentHistoryId`であっても、対象履歴が存在しない場合は、

```http
404 Not Found
```

および、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

となることを確認する。

---

### 2.27 他利用者の判定履歴

他利用者に属する判定履歴IDを指定した場合、取得できないことを確認する。

例：

```text
X-User-Id
    = 1

assessmentHistoryId
    = 100

History 100の所有者
    = User 2
```

期待結果：

```http
404 Not Found
```

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

---

### 2.28 他利用者所属を公開しない

以下をクライアントから区別できないことを確認する。

```text
存在しない判定履歴
        ↓
ASSESSMENT_HISTORY_NOT_FOUND

他利用者の判定履歴
        ↓
ASSESSMENT_HISTORY_NOT_FOUND
```

以下のようなエラーコードを返却しない。

```text
ASSESSMENT_HISTORY_FORBIDDEN
OTHER_USER_ASSESSMENT_HISTORY
```

---

### 2.29 利用者境界を取得条件へ含める

RepositoryまたはQueryのテストでは、単純に

```text
assessment_histories.id
    = assessmentHistoryId
```

だけで取得していないことを確認する。

テーブル設計に応じて、

```text
assessment_histories.user_id
    = UserContext.userId
```

または、

```text
objectives.user_id
    = UserContext.userId
```

などの利用者境界が取得条件へ含まれていることを確認する。

---

### 2.30 論理削除済み判定履歴

`assessment_histories`にSoftDeletesを採用している場合は、論理削除済み履歴が通常取得できないことを確認する。

期待結果：

```http
404 Not Found
```

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

SoftDeletesを採用しない場合、このテストは不要とする。

---

### 2.31 関連目的の想定外不存在

DB制約上は通常発生しないが、テスト上意図的に

```text
assessment_histories.objective_id
    ↓
対応するobjectivesなし
```

という状態を作成できる場合は、内部データ不整合として処理されることを確認する。

期待結果：

```http
500 Internal Server Error
```

```text
INTERNAL_SERVER_ERROR
```

---

### 2.32 関連月末資産状況の想定外不存在

同様に、

```text
assessment_histories
    ↓
month_end_asset_snapshot_id
    ↓
対応するmonth_end_asset_snapshotsなし
```

の場合も、利用者入力エラーではなく内部エラーとして扱うことを確認する。

---

### 2.33 不足データを自動補完しない

関連データに不整合が存在した場合でも、ASM-004によって

```text
目的を自動作成する
月末資産状況を自動作成する
判定根拠を再計算する
判定履歴を更新する
```

などの処理が行われないことを確認する。

---

### 2.34 DBカラム名を直接返却しない

レスポンスに、

```text
objective_id
assessment_result
available_asset_amount
assessed_at
```

などのsnake_caseのDBカラム名が直接公開されないことを確認する。

API仕様どおり、

```text
objectiveId
assessmentResult
availableAssetAmount
assessedAt
```

として返却されることを確認する。

---

### 2.35 内部情報を返却しない

レスポンスに、不要な内部情報が含まれないことを確認する。

例えば、

```text
user_id
deleted_at
SQL
SQLSTATE
DB制約名
Laravel例外情報
スタックトレース
サーバーファイルパス
```

などを外部へ公開しない。

---

### 2.36 userIdをレスポンスへ含めない

操作対象利用者は`X-User-Id`によって確定しているため、

```text
userId
```

をASM-004のレスポンス項目として返却しないことを確認する。

---

### 2.37 Request Bodyを必要としない

ASM-004はGET APIであるため、Request Bodyなしで正常に実行できることを確認する。

```http
GET /api/v1/assessment-histories/100
X-User-Id: 1
Accept: application/json
```

だけで取得できること。

---

### 2.38 クエリパラメータを必要としない

以下のような追加パラメータなしで判定履歴詳細を取得できることを確認する。

```text
objectiveId
snapshotId
userId
includeDetails
```

取得対象は、

```text
assessmentHistoryId
+
利用者コンテキスト
```

によって決定する。

---

### 2.39 トランザクションを必要としない

ASM-004実行時に、更新処理を前提とした明示的なトランザクションが使用されていないことを確認する。

特に、

```php
DB::transaction(...)
```

が不要に導入されていないことを実装レビューでも確認する。

---

### 2.40 ロックを取得しない

ASM-004実行時に、

```php
lockForUpdate()
```

などの悲観ロックを取得しないことを確認する。

ASM-004による参照によって、他の更新処理を不要にブロックしないこと。

---

### 2.41 副作用がない

ASM-004実行前後で、以下の業務テーブルのデータが変更されないことを確認する。

```text
assessment_histories
objectives
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
net_incomes
```

ASM-004は参照専用APIとして動作すること。

---

### 2.42 冪等性

同一Requestを複数回実行しても、業務データへ追加の変更が発生しないことを確認する。

例：

```text
GET 1回目
    ↓
200 OK

GET 2回目
    ↓
200 OK

GET 3回目
    ↓
200 OK
```

この間にASM-004自身によって判定履歴が追加・更新されないこと。

---

### 2.43 ASM-003からの遷移

ASM-003で取得した

```text
assessmentHistoryId
```

をASM-004へ渡した場合に、対応する判定履歴詳細を正常に取得できることを確認する。

概念的には、

```text
ASM-003
    ↓
History 100を選択
    ↓
ASM-004
/api/v1/assessment-histories/100
    ↓
History 100の詳細
```

となること。

---

### 2.44 ASM-002との整合性

ASM-002で判定結果を保存した直後にASM-004でその履歴を取得した場合、保存された内容と取得内容が一致することを確認する。

特に、

```text
assessmentResult
availableAssetAmount
requiredAmount
averageNetIncome
requiredEmergencyFund
remainingAmount
notAssessableReason
```

などの判定結果・計算根拠について整合していることを確認する。

---

### 2.45 ASM-001のプレビュー結果を取得対象にしない

ASM-001は判定結果を保存しないため、プレビューを実行しただけではASM-004で取得可能な判定履歴が生成されないことを確認する。

```text
ASM-001
    ↓
プレビュー
    ↓
assessment_histories
登録なし
    ↓
ASM-004で新規履歴として取得不可
```

となること。

---

### 2.46 Feature Test

LaravelのFeature Testでは、主に以下を確認する。

- 正常取得
- レスポンスJSON構造
- IDのstring変換
- 金額のinteger変換
- 判定結果ごとのレスポンス
- 判定不可理由
- `X-User-Id`未指定
- `X-User-Id`形式不正
- 利用者不存在
- `assessmentHistoryId`形式不正
- 判定履歴不存在
- 他利用者境界
- 目的無効化後の履歴取得
- 月末資産状況確定解除後の履歴取得
- 保存済み計算根拠の維持
- 現在値から再計算しないこと
- 副作用がないこと

---

### 2.47 Query / Repository Test

判定履歴取得をQueryまたはRepositoryへ分離する場合は、以下を確認する。

```text
assessmentHistoryId
+
UserContext.userId
```

によって正しい判定履歴を取得できること。

また、

```text
他利用者の履歴
```

を取得できないことを確認する。

---

### 2.48 Resource Test

API Resourceについては、DBまたはDTOの値がAPI仕様どおりに変換されることを確認する。

主に、

```text
bigint ID
    ↓
string

snake_case
    ↓
camelCase

金額
    ↓
integer

日時
    ↓
API共通日時形式
```

を確認する。

---

### 2.49 テストで確認しない責務

ASM-004のテストでは、以下の判定ロジックそのものを中心的なテスト対象としない。

- 利用可能資産額の算出ロジック
- 平均手取り収入の算出ロジック
- 必要生活防衛資金の算出ロジック
- 目的達成可否の判定ロジック
- 判定不可条件の判定ロジック
- 月末残高の集計ロジック
- 商品別評価額の集計ロジック

これらは、ASM-001・ASM-002および判定ドメインロジック側のテスト責務とする。

ASM-004では、

```text
保存済み判定履歴を
正しい利用者境界で取得し、
保存時点の結果と計算根拠を
変更せずAPI形式へ変換して返却できること
```

を中心にテストする。

---

## 3. 関連ドキュメント

### 3 関連ドキュメント

- [ASM-004 API詳細設計](../../api/details/assessments/asm-004-detail.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [ASM-004 Laravelアーキテクチャ設計](../../architecture/laravel/assessments/asm-004-detail.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [ASM-004 Reactアーキテクチャ設計](../../architecture/react/assessments/asm-004-detail.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
