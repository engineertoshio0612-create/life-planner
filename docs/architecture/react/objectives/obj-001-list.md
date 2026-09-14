# OBJ-001 目的一覧取得

## 1. 概要

操作対象となる利用者に登録された
目的一覧を取得し、
目的一覧画面へ表示する。

一覧では、目的ごとに主に以下の情報を扱う。

* 目的ID
* 目的名
* 実施予定年月
* 必要支出額
* メモ
* 利用状態

利用中の目的だけでなく、
無効化された目的も取得対象とし、
`enabled`の値によって利用状態を判別する。

取得した目的情報は、
目的一覧の表示に加えて、
目的詳細、目的更新、目的無効化および
目的達成判定などの後続操作へ遷移するために利用する。

一覧取得結果が0件の場合は、
エラーとして扱わず、
目的が登録されていない状態として表示する。

React側では、
TanStack Queryを利用して取得状態を管理し、
取得中、取得成功、データなしおよび
エラーの各状態に応じて表示を切り替える。

本APIの利用では目的情報の参照のみを行い、
目的の登録、更新、無効化および
目的達成判定は行わない。

---

## 2. React・TypeScriptでの利用

登録リクエスト型は、
以下とする。

```ts
export type CreateObjectiveRequest = {
  name: string;
  plannedYearMonth: string | null;
  requiredExpense: number;
  memo: string | null;
};
```

レスポンス型は、
以下とする。

```ts
export type Objective = {
  id: string;
  name: string;
  plannedYearMonth: string | null;
  requiredExpense: number;
  memo: string | null;
  enabled: boolean;
};

export type CreateObjectiveResponse = {
  data: Objective;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.post<
    CreateObjectiveResponse
  >(
    '/api/v1/objectives',
    {
      name: 'マイホーム購入',
      plannedYearMonth: '2030-03',
      requiredExpense: 5000000,
      memo: '頭金として利用する',
    },
  );
```

登録成功後は、
目的一覧画面または
目的詳細画面へ遷移する。

---

### 2.1 nameの扱い

`name`は、
入力必須項目とする。

入力前後の空白は除去して送信する。

```ts
const request = {
  ...form,
  name: form.name.trim(),
};
```

---

### 2.2 plannedYearMonthの扱い

`plannedYearMonth`は、
`YYYY-MM`形式で送信する。

未設定の場合は、
`null`を送信する。

```ts
plannedYearMonth: null
```

JavaScriptの`Date`オブジェクトへ
変換せず、
年月を表す文字列として扱う。

---

### 2.3 requiredExpenseの扱い

`requiredExpense`は、
日本円の整数値として扱う。

送信前に、
数値へ変換する。

```ts
requiredExpense:
  Number(form.requiredExpense);
```

小数点は送信しない。

---

### 2.4 memoの扱い

`memo`は、
任意入力とする。

未入力の場合は、
`null`を送信する。

```ts
memo:
  form.memo === ''
    ? null
    : form.memo;
```

---

### 2.5 enabledの扱い

`enabled`は、
レスポンスでのみ取得する。

登録時は、
フロントエンドから送信しない。

登録直後は、
`true`で返却される。

---

### 2.6 ローディング表示

登録処理中は、
送信ボタンを非活性化する。

二重送信を防止するため、
登録完了またはエラーになるまで
再送信できないようにする。

---

### 2.7 エラー表示

入力項目ごとのエラーは、
`error.details.field`
を利用して表示する。

対象となる項目は、
以下とする。

- `name`
- `plannedYearMonth`
- `requiredExpense`
- `memo`

エラーコードごとの基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `VALIDATION_ERROR` | 入力欄へエラー表示 |
| `OBJECTIVE_ALREADY_EXISTS` | 「同じ目的名が登録されています」を表示 |
| `INVALID_USER_ID` | 共通エラー表示 |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示 |

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [OBJ-002 目的登録](./obj-002-create.md)
- [OBJ-003 目的詳細取得](./obj-003-detail.md)
- [OBJ-004 目的更新](./obj-004-update.md)
- [OBJ-005 目的無効化](./obj-005-assessments.md)
- [OBJ-006 目的達成判定](./obj-006-disabled.md)
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
