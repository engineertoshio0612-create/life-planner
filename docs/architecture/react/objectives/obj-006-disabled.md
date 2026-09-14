# OBJ-006 目的達成判定

## 1. 概要

操作対象となる利用者に登録された目的について、
指定した判定対象年月時点での
目的達成可否を判定する。

Reactでは、目的達成判定画面から
主に以下の情報を入力し、
OBJ-006 目的達成判定APIを実行する。

* 判定対象年月
* 翌月クレジットカード支払予定額

目的情報、利用可能資産および
平均手取り収入などの判定に必要な情報は、
バックエンド側で取得・算出する。

フロントエンドでは、
これらの値を独自に算出して送信せず、
APIから返却された判定結果を正として扱う。

判定成功時には、主に以下の情報を表示する。

* 目的達成判定結果
* 利用可能資産
* 必要支出額
* 平均手取り収入
* 判定後の残額

OBJ-006は判定成功時に
新しい目的達成判定履歴を保存するため、
TanStack QueryではMutationとして扱う。

判定処理中は実行操作を非活性化し、
同一操作の二重送信によって
不要な判定履歴が複数登録されないようにする。

無効化された目的や、
判定に必要な月末資産状況、手取り収入、
利用可能資産設定などが不足している場合は、
エラーコードに応じて
判定を実行できない理由を利用者へ表示する。

判定成功時は、
バックエンドから返却された結果を利用して
判定結果を表示するとともに、
必要に応じて判定履歴に関するQueryを
再取得または無効化する。

本APIの利用では目的達成判定の実行と
その結果の表示を行い、
目的情報、資産情報、手取り収入および
利用可能資産設定そのものは変更しない。

---

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

### 2.1 targetYearMonthの扱い

`targetYearMonth`は、
判定の基準となる対象年月を
`YYYY-MM`形式で送信する。

```ts
const targetYearMonth = '2026-08';
```

対象年月は、
日付ではなく年月を表す業務値として扱う。

JavaScriptの`Date`オブジェクトへ
不必要に変換せず、
原則として文字列で保持する。

---

### 2.2 nextMonthCreditCardPaymentの扱い

`nextMonthCreditCardPayment`は、
翌月クレジットカード支払予定額を
日本円の整数値で指定する。

```ts
const nextMonthCreditCardPayment = 120000;
```

支払予定額がない場合は、
`0`を送信する。

```ts
const request: AssessObjectiveRequest = {
  targetYearMonth: '2026-08',
  nextMonthCreditCardPayment: 0,
};
```

`0`は有効な入力値であり、
未入力として扱ってはならない。

例えば、
以下の判定は行わない。

```ts
if (!request.nextMonthCreditCardPayment) {
  // 0円まで未入力として扱われるため使用しない
}
```

未入力判定は、
`undefined`などと明示的に比較する。

---

### 2.3 averageNetIncomeの扱い

`averageNetIncome`は、
バックエンドが判定対象年月より前の
連続する3か月の手取り収入から算出する。

フロントエンドから
平均手取り収入を送信しない。

また、
フロントエンドで独自に再計算せず、
APIから返却された値を
判定根拠として表示する。

これにより、
フロントエンドとバックエンドで
平均値や端数処理が異なることを防止する。

---

### 2.4 availableAssetsの扱い

`availableAssets`は、
判定対象年月時点の
利用可能資産設定および
確定済み月末資産状況から
バックエンドが算出する。

フロントエンドから
利用可能資産額を送信しない。

取得した値は、
判定結果の根拠として表示する。

```ts
const formattedAvailableAssets =
  new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY',
    maximumFractionDigits: 0,
  }).format(response.data.availableAssets);
```

---

### 2.5 assessmentResultの扱い

`assessmentResult`は、
API仕様で定義した判定結果の値として扱う。

フロントエンドでは、
文字列を直接画面へ表示するのではなく、
定義済みの値に応じて表示内容を切り替える。

例：

```ts
export type AssessmentResult =
  | 'achievable'
  | 'notAchievable';
```

```ts
const assessmentResultLabels: Record<
  AssessmentResult,
  string
> = {
  achievable: '達成可能',
  notAchievable: '達成困難',
};
```

実際に使用する値は、
OBJ-006のレスポンス項目および
`assessment_histories`の定義と一致させる。

未定義の値を受信した場合は、
正常な判定結果として扱わず、
共通エラー表示または
フォールバック表示を行う。

---

### 2.6 remainingAmountの扱い

`remainingAmount`は、
判定計算後に残る金額を表す。

日本円の整数値として扱い、
フロントエンドで再計算しない。

負数を許容する設計の場合は、
マイナス値もそのまま表示する。

```ts
const formattedRemainingAmount =
  new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY',
    maximumFractionDigits: 0,
  }).format(response.data.remainingAmount);
```

`remainingAmount`の正式名称および計算式は、
機能要件、
ユビキタス言語および
`assessment_histories`の定義と統一する。

---

### 2.7 判定実行前の画面制御

判定実行前に、
以下を確認できるようにする。

- 判定対象となる目的
- 判定対象年月
- 必要支出額
- 翌月クレジットカード支払予定額

無効化された目的では、
判定実行ボタンを非表示または非活性にする。

ただし、
フロントエンドの状態だけを信頼せず、
OBJ-006でも目的の利用状態を再検証する。

---

### 2.8 判定実行中の画面制御

判定処理中は、
実行ボタンを非活性にする。

判定が完了するまで、
同じリクエストを再送信できないようにする。

```ts
const [isAssessing, setIsAssessing] =
  useState(false);
```

```ts
if (isAssessing) {
  return;
}
```

本APIは判定成功ごとに
新しい判定履歴を登録するため、
二重送信によって
同じ内容の履歴が複数作成される可能性がある。

---

### 2.9 判定成功時の扱い

判定成功時は、
APIから返却された結果を使用して
判定結果画面を表示する。

判定結果をフロントエンドで再構築せず、
レスポンスを表示用データとして使用する。

必要に応じて、
以下の画面へ遷移できるようにする。

- 判定結果画面
- 判定履歴一覧画面
- 目的詳細画面

判定成功時には、
判定履歴も登録済みである。

---

### 2.10 判定不可時の扱い

判定条件を満たさない場合は、
システムエラーとして扱わず、
判定を実行できない業務状態として表示する。

エラーコードごとの基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `OBJECTIVE_DISABLED` | 無効化された目的は判定できないことを表示する |
| `CONFIRMED_ASSET_SNAPSHOT_NOT_FOUND` | 月末資産状況の登録・確定を案内する |
| `NET_INCOME_DATA_INSUFFICIENT` | 必要な3か月分の手取り収入登録を案内する |
| `AVAILABLE_ASSETS_CALCULATION_NOT_AVAILABLE` | 利用可能資産設定の確認を案内する |
| `ASSESSMENT_NOT_AVAILABLE` | 判定条件を満たしていないことを表示する |
| `VALIDATION_ERROR` | 入力項目ごとにエラーを表示する |
| `OBJECTIVE_NOT_FOUND` | 目的一覧画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

判定不可の場合は、
判定結果画面へ遷移しない。

---

### 2.11 バリデーションエラーの表示

バリデーションエラーでは、
`error.details.field`を利用して
対象項目へエラーメッセージを表示する。

想定するフィールドは、
以下とする。

- `targetYearMonth`
- `nextMonthCreditCardPayment`

```ts
if (
  detail.field ===
  'nextMonthCreditCardPayment'
) {
  setFieldError(
    'nextMonthCreditCardPayment',
    detail.message,
  );
}
```

---

### 2.12 金額表示

以下の金額項目は、
すべて日本円の整数値として扱う。

- `availableAssets`
- `requiredExpense`
- `averageNetIncome`
- `nextMonthCreditCardPayment`
- `remainingAmount`

画面表示時は、
`Intl.NumberFormat`を使用して
桁区切りを行う。

フロントエンドで
独自の端数処理や
小数計算を行わない。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [OBJ-001 目的一覧取得](./obj-001-list.md)
- [OBJ-002 目的登録](./obj-002-create.md)
- [OBJ-003 目的詳細取得](./obj-003-detail.md)
- [OBJ-004 目的更新](./obj-004-update.md)
- [OBJ-005 目的無効化](./obj-005-assessments.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/objectives/README.md)
- [Reactアーキテクチャ設計](../../../architecture/react/objectives/README.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)
