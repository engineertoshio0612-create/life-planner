# OBJ-003 目的詳細取得

## 1. 概要

操作対象となる利用者の
指定した目的の詳細情報を取得する。

取得した目的は、
目的編集画面および
目的達成判定画面で利用する。

---

## 2. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type GetObjectiveDetailParams = {
  objectiveId: string;
};
```

目的詳細の型は、
以下とする。

```typescript
export type ObjectiveDetail = {
  id: string;
  name: string;
  plannedYearMonth: string | null;
  requiredExpense: number;
  memo: string | null;
  enabled: boolean;
};
```

レスポンス型は、
以下とする。

```typescript
export type GetObjectiveDetailResponse = {
  data: ObjectiveDetail;
};
```

API呼び出し例は、
以下とする。

```typescript
const response =
  await apiClient.get<GetObjectiveDetailResponse>(
    `/api/v1/objectives/${objectiveId}`,
  );
```

取得したデータは、
以下の画面で利用する。

- 目的詳細画面
- 目的編集画面の初期表示
- 目的達成判定前の目的情報確認

### 2.1 plannedYearMonthの扱い

`plannedYearMonth` は、
`YYYY-MM`形式または
`null`で返却される。

```typescript
if (objective.plannedYearMonth === null) {
  // 実施予定年月は未設定
}
```

実施予定年月は、
日付ではなく年月を表す業務値である。

そのため、
JavaScriptの`Date`オブジェクトへ
不必要に変換せず、
文字列として扱う。

未設定の場合は、
画面上で以下のように表示する。

```text
未設定
```

### 2.2 requiredExpenseの扱い

`requiredExpense`は、
日本円の整数値として扱う。

画面表示時は、
必要に応じて桁区切りを行う。

```typescript
const formattedRequiredExpense =
  new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY',
    maximumFractionDigits: 0,
  }).format(objective.requiredExpense);
```

フロントエンド側で、
小数への変換や
独自の端数処理は行わない。

### 2.3 memoの扱い

`memo`は、
文字列または`null`で返却される。

`null`の場合は、
空欄または未入力として表示する。

```typescript
const memo = objective.memo ?? '';
```

編集画面では、
`null`を空文字へ変換して
フォーム初期値へ設定してよい。

### 2.4 enabledの扱い

`enabled`を利用して、
目的の利用状態を判定する。

| enabled | 状態 |
|---|---|
| `true` | 有効 |
| `false` | 無効 |

無効化された目的は、
詳細表示できる。

ただし、
以下の操作は許可しない。

- 目的情報の更新
- 再無効化
- 新しい目的達成判定の実行

フロントエンドでは、
`enabled = false`の場合に
編集ボタン、
無効化ボタンおよび
判定実行ボタンを非表示または非活性にする。

実際の更新・無効化・判定APIでも、
バックエンド側で利用状態を再検証する。

### 2.5 ローディング表示

目的詳細の取得中は、
ローディング表示を行う。

取得完了前に、
以前表示していた別の目的情報を
現在の目的として表示しない。

目的IDが変更された場合は、
以前の取得結果を破棄するか、
新しい取得が完了するまで
ローディング状態を表示する。

### 2.6 エラー表示

エラーコードごとの基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `INVALID_OBJECTIVE_ID` | 不正なURLとしてエラー表示する |
| `OBJECTIVE_NOT_FOUND` | 目的一覧画面へ戻し、対象が存在しないことを表示する |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

`OBJECTIVE_NOT_FOUND`の場合は、
以下の原因をフロントエンドで区別しない。

- 目的が存在しない
- 論理削除されている
- 他の利用者に帰属している

---

## 3. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)