# INC-002 手取り収入登録

## 1. 概要

本ドキュメントでは、
INC-002 手取り収入登録APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

本APIでは、
操作対象となる利用者に対して、
指定された対象年月の手取り収入を
新規登録する。

手取り収入は、
利用者ごとに対象年月単位で管理し、
同一対象年月について
複数の手取り収入を登録することはできない。

対象年月に手取り収入が発生しなかった場合も、
未登録とはせず、
金額を0円として登録する。

React・TypeScript実装では、
登録リクエストおよび
登録結果のレスポンス型を定義し、
APIとの通信データを
型安全に扱う。

登録画面では、
対象年月、
手取り収入金額および
備考を入力値として扱う。

備考は任意入力とし、
未入力の場合は
`null`として扱えるようにする。

登録成功時は、
APIから返却された
登録後の手取り収入を受け取り、
手取り収入一覧画面へ戻る、
または登録した手取り収入の
詳細画面へ遷移する。

登録失敗時は、
APIから返却されたエラー内容に応じて、
対象項目ごとの
バリデーションメッセージを表示する。

また、
同一対象年月の手取り収入が
すでに登録されている場合は、
重複登録として扱い、
利用者が登録できなかった理由を
確認できるようにする。

登録された手取り収入は、
手取り収入一覧の表示、
平均手取り収入の算出および
目的達成判定で利用する。

---

## 2. React・TypeScriptでの利用

登録リクエスト型

```ts
export type CreateNetIncomeRequest = {
  targetYearMonth: string;
  amount: number;
  memo?: string | null;
};
```

レスポンス型

```ts
export type NetIncomeResponse = {
  data: {
    id: string;
    targetYearMonth: string;
    amount: number;
    memo: string | null;
  };
};
```

登録例

```ts
await apiClient.post<NetIncomeResponse>(
  '/api/v1/net-incomes',
  {
    targetYearMonth: '2026-08',
    amount: 325000,
    memo: '残業代を含む',
  },
);
```

登録成功後は、

- 一覧画面へ戻る
- または詳細画面へ遷移する

いずれかの画面遷移を行う。

登録失敗時は、
対象項目ごとのバリデーションメッセージを表示する。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [INC-001 手取り収入一覧取得](./inc-001-list.md)
- [INC-003 手取り収入詳細取得](./inc-003-detail.md)
- [INC-004 手取り収入更新](./inc-004-update.md)
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