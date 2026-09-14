# OBJ-006 目的達成判定

## 1. 概要

操作対象となる利用者に登録された目的について、
指定した判定対象年月時点での
目的達成可否を判定する。

判定では、主に以下の情報を利用する。

* 目的情報
* 判定対象年月の確定済み月末資産状況
* 判定対象年月時点の利用可能資産設定
* 平均手取り収入
* 翌月クレジットカード支払予定額

これらの情報から利用可能資産および
目的達成後の残額などを算出し、
目的を達成可能か判定する。

判定結果は、以下のいずれかとして扱う。

* 達成
* 未達成
* 判定不可

判定対象となる目的は、
操作対象利用者に帰属する
利用中の目的に限定する。

目的が存在しない場合、
無効化されている場合、
判定に必要な確定済み月末資産状況が存在しない場合など、
判定を実行するための条件を満たさない場合は
業務エラーとして扱う。

判定が正常に完了した場合は、
判定結果と判定時点の


---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、目的ID、判定対象年月、翌月クレジットカード支払予定額および利用者コンテキストを取得する。

Form Requestまたは入力用DTOから検証済みの入力値を受け取り、目的達成判定UseCaseを呼び出す。

UseCaseから受け取った判定結果を、Responderへ渡す。

以下の処理は、Actionへ直接記述しない。

- 入力値の単項目バリデーション
- 利用者境界の判定
- 判定対象データの取得
- 判定計算
- 判定履歴登録
- トランザクション制御
- レスポンス生成処理

---

### 2.2 UseCase

目的達成判定のユースケース処理を担当する。

主な処理は、以下とする。

- 操作対象利用者を受け取る
- 判定対象となる目的を取得する
- 確定済み月末資産状況を取得する
- 判定対象年月時点の利用可能資産設定を取得する
- 利用可能資産を算出する
- 平均手取り収入を算出する
- 判定計算を実行する
- 判定履歴を登録する
- 判定結果を返却する

判定条件を満たさない場合は、業務エラーとして扱う。

判定履歴は、判定成功時のみ登録する。

---

### 2.3 Form Request / DTO

入力値の形式および単項目バリデーションを担当する。

主な検証対象は、以下とする。

- `targetYearMonth`
- `nextMonthCreditCardPayment`

Form Requestでは、以下を検証する。

- 必須項目
- データ型
- NULL可否
- `YYYY-MM` 形式
- 数値範囲
- 未定義項目

データベース状態に依存する以下の業務ルールは、UseCaseで実施する。

- 目的の存在確認
- 利用者境界確認
- 利用状態確認
- 確定済み月末資産状況の存在確認
- 平均手取り収入算出可否
- 利用可能資産算出可否

DTO例：

```php
final readonly class AssessObjectiveInput
{
    public function __construct(
        public string $targetYearMonth,
        public int $nextMonthCreditCardPayment,
    ) {
    }
}
```

---

### 2.4 Query

判定対象データ取得を担当する。

取得対象は、以下とする。

- `objectives`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_account_available_settings`
- `asset_accounts`
- `holding_assets`
- `net_incomes`

目的取得時は、必ず利用者境界を条件へ含める。

```php
$objective = Objective::query()
    ->where('id', $objectiveId)
    ->where('user_id', $userId)
    ->first();
```

論理削除済み目的は、取得対象としない。

無効化された目的は取得後、`OBJECTIVE_DISABLED` として扱う。

---

### 2.5 Repository

判定履歴登録のみ担当する。

Repositoryでは、`assessment_histories` への登録を実施する。

保存対象は、以下とする。

- `objective_id`
- 判定対象年月
- 判定時点の目的情報
- 利用可能資産
- 平均手取り収入
- 翌月クレジットカード支払予定額
- 判定結果
- 判定根拠

目的、月末資産情報、手取り収入および利用可能資産設定は更新しない。

---

### 2.6 判定サービス

判定計算は、UseCaseへ直接記述せず、専用ドメインサービスへ委譲する。

例：

```php
$result = $assessmentService->assess(
    objective: $objective,
    availableAssets: $availableAssets,
    averageNetIncome: $averageNetIncome,
    nextMonthCreditCardPayment: $input->nextMonthCreditCardPayment,
);
```

判定ロジックは、Controller、Repository、Queryへ記述しない。

---

### 2.7 トランザクション

以下の処理を、1つのデータベーストランザクション内で実行する。

- 判定対象取得
- 判定計算
- 判定履歴登録

実装例：

```php
$result = DB::transaction(
    function () use (
        $userId,
        $objectiveId,
        $input,
    ) {
        $result =
            $this->useCase->execute(
                $userId,
                $objectiveId,
                $input,
            );

        return $result;
    },
);
```

判定途中で例外が発生した場合は、判定履歴を登録しない。

---

### 2.8 Responder

UseCaseから受け取った判定結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、以下のHTTPステータスで返却する。

```text
201 Created
```

判定履歴が登録されたことを、レスポンスから判別できる。

Responderは、以下を行わない。

- 判定計算
- 判定履歴登録
- データ取得
- 業務ルール判定

---

### 2.9 API Resource

判定結果を、APIレスポンス形式へ変換する。

Resourceでは、フィールド名をcamelCaseへ変換する。

内部DBカラムは公開しない。

例：

```php
return [
    'assessmentResult' => $this->assessment_result,
    'availableAssets' => $this->available_assets,
    'requiredExpense' => $this->required_expense,
    'averageNetIncome' => $this->average_net_income,
    'remainingAmount' => $this->remaining_amount,
];
```

---

### 2.10 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSON共通処理
- 共通例外処理
- ログコンテキスト設定

---

### 2.11 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 目的ID形式不正 | `INVALID_OBJECTIVE_ID` |
| 目的不存在 | `OBJECTIVE_NOT_FOUND` |
| 無効化済み目的 | `OBJECTIVE_DISABLED` |
| 確定済み月末資産状況不存在 | `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` |
| 平均手取り収入不足 | `NET_INCOME_DATA_INSUFFICIENT` |
| 利用可能資産算出不可 | `AVAILABLE_ASSETS_CALCULATION_NOT_AVAILABLE` |
| 判定不可 | `ASSESSMENT_NOT_AVAILABLE` |
| 入力値不正 | `VALIDATION_ERROR` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

SQL、スタックトレースおよび内部例外メッセージは、APIレスポンスへ含めない。

ログには、調査に必要な範囲で以下を記録する。

- 利用者ID
- 目的ID
- 判定対象年月
- リクエストID
- 独自エラーコード

判定に利用した金額内訳や計算途中の情報は、通常ログへ出力しない。


## 2. React・TypeScriptでの利用

リクエスト型は、
以下とする。

```ts
export type AssessObjectiveRequest = {
  targetYearMonth: string;
  nextMonthCreditCardPayment: number;
};
```

判定結果の型は、
以下とする。

```ts
export type ObjectiveAssessmentResult = {
  assessmentResult: string;
  availableAssets: number;
  requiredExpense: number;
  averageNetIncome: number;
  remainingAmount: number;
};
```

レスポンス型は、
以下とする。

```ts
export type AssessObjectiveResponse = {
  data: ObjectiveAssessmentResult;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.post<AssessObjectiveResponse>(
    `/api/v1/objectives/${objectiveId}/assessments`,
    {
      targetYearMonth: '2026-08',
      nextMonthCreditCardPayment: 120000,
    },
  );
```

取得した判定結果は、
以下の表示に利用する。

- 目的達成判定結果
- 判定に使用した利用可能資産
- 判定時点の必要支出額
- 判定に使用した平均手取り収入
- 判定後の残額

---

## 3. 関連ドキュメント

- [OBJ-006 API詳細設計](../../../api/details/objectives/obj-006-disabled.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [目的 Laravelアーキテクチャ設計](./README.md)
- [OBJ-006 テスト設計](../../../tests/objectives/obj-006-disabled.md)
- [目的 テスト設計](../../../tests/objectives/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
