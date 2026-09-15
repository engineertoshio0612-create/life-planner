# INC-004 手取り収入更新

## 1. 概要

本ドキュメントでは、
INC-004 手取り収入更新APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

本APIでは、
操作対象となる利用者に登録された
指定の手取り収入について、
対象年月、
手取り収入金額および
備考を更新する。

React・TypeScript実装では、
INC-003 手取り収入詳細取得APIから
取得したデータを
編集画面の初期値として使用し、
利用者が変更した内容を
INC-004 手取り収入更新APIへ送信する。
更新対象となる項目は、
以下とする。

- 対象年月
- 手取り収入金額
- 備考

対象年月は、
業務上の年月を表す
`YYYY-MM`形式の文字列として扱う。

手取り収入金額が0円の場合も、
未入力とは区別し、
有効な業務データとして送信する。

備考を未設定へ変更する場合は、
`null`として扱い、
必要に応じて送信前に
空文字から`null`へ正規化する。

更新成功後は、
APIから返却された
更新後の手取り収入を使用して
画面状態を更新する。

その後、
画面仕様に応じて、
手取り収入一覧画面への遷移、
詳細画面の再表示、
または更新完了メッセージの表示を行う。

入力値に問題がある場合は、
APIから返却されたフィールド情報を利用して、
対象となる入力項目へ
バリデーションエラーを表示する。

更新後の対象年月が
他の手取り収入と重複する場合は、
`NET_INCOME_ALREADY_EXISTS`を判定し、
対象年月に関するエラーとして表示する。

更新対象が存在しない場合は、
`NET_INCOME_NOT_FOUND`を判定し、
一覧画面へ戻すなど、
存在しないデータを
継続して編集しないよう制御する。

また、
更新リクエスト送信中は
更新ボタンを非活性にし、
フロントエンド側でも
不要な二重送信を防止する。

更新された手取り収入は、
手取り収入一覧の表示、
平均手取り収入の算出および
目的達成判定で利用する。

---

## 2. React・TypeScriptでの利用

レスポンス型は、以下とする。

```typescript
export type NetIncome = {
  id: string;
  targetYearMonth: string;
  amount: number;
  memo: string | null;
};

export type UpdateNetIncomeResponse = {
  data: NetIncome;
};
```

API呼び出し例は、以下とする。

```typescript
const response =
  await apiClient.patch<UpdateNetIncomeResponse>(
    `/api/v1/net-incomes/${netIncomeId}`,
    {
      targetYearMonth: '2026-08',
      amount: 330000,
      memo: '残業代を修正',
    },
  );
```

更新画面では、INC-003 手取り収入詳細取得APIのレスポンスを初期値として使用する。

更新成功後は、返却された更新後データを使用して画面状態を更新する。

必要に応じて、以下のいずれかの画面遷移を行う。

- 手取り収入一覧画面へ戻る
- 手取り収入詳細画面を再表示する
- 更新完了メッセージを表示したまま編集画面に留まる

### 27.1 金額0円の扱い

`amount` が `0` の場合も、有効な手取り収入として送信する。

```typescript
const request: UpdateNetIncomeRequest = {
  targetYearMonth: '2026-08',
  amount: 0,
  memo: '休職により収入なし',
};
```

`0` を未入力として扱ってはならない。

例えば、以下のような判定は行わない。

```typescript
if (!request.amount) {
  // 0円まで未入力扱いになるため使用しない
}
```

未入力判定が必要な場合は、`undefined` または `null` との比較を明示する。

```typescript
if (request.amount === undefined) {
  // 未入力
}
```

### 27.2 備考の扱い

備考を未設定へ変更する場合は、`null` を送信する。

```typescript
const request: UpdateNetIncomeRequest = {
  targetYearMonth: '2026-08',
  amount: 330000,
  memo: null,
};
```

空文字を送信した場合も、バックエンドでは `null` へ正規化する。

フロントエンドでも、送信前に空文字を `null` へ変換してよい。

```typescript
const normalizedMemo =
  form.memo.trim() === ''
    ? null
    : form.memo.trim();
```

### 27.3 対象年月の扱い

`targetYearMonth` は、API共通方針に従い `YYYY-MM` 形式で送信する。

```typescript
const targetYearMonth = '2026-08';
```

JavaScriptの`Date`オブジェクトへ不必要に変換せず、業務上の年月を表す文字列として扱う。

対象年月を変更した場合は、他の手取り収入との重複によって `NET_INCOME_ALREADY_EXISTS` が返却される可能性がある。

### 27.4 エラー表示

バリデーションエラーでは、`error.details` の `field` を利用して入力項目ごとにエラーメッセージを表示する。

想定するフィールドは、以下とする。

- `targetYearMonth`
- `amount`
- `memo`

対象年月重複時は、`NET_INCOME_ALREADY_EXISTS` を判定し、対象年月入力欄へエラーを表示する。

```typescript
if (
  error.response?.data.error.code
  === 'NET_INCOME_ALREADY_EXISTS'
) {
  setFieldError(
    'targetYearMonth',
    '指定した対象年月の手取り収入はすでに登録されています。',
  );
}
```

`NET_INCOME_NOT_FOUND` が返却された場合は、一覧画面へ戻し、対象データが存在しないことを表示する。

### 27.5 更新中の画面制御

更新リクエスト送信中は、二重送信を防止するため、更新ボタンを非活性にする。

更新完了後またはエラー発生後に、操作可能な状態へ戻す。

Phase1では、冪等性キーは使用しない。

同一リクエストが二重送信された場合でも、最終状態は同じとなるが、不要なAPI呼び出しを防ぐためフロントエンドでも二重送信を抑止する。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [INC-001 手取り収入一覧取得](./inc-001-list.md)
- [INC-002 手取り収入登録](./inc-002-create.md)
- [INC-003 手取り収入詳細取得](./inc-003-detail.md)
- [INC-005 平均手取り収入取得](./inc-005-average.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/net-incomes/README.md)
- [Reactアーキテクチャ設計](../../../architecture/react/net-incomes/README.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)