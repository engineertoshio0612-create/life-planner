# INC-003 手取り収入詳細取得

## 1. 概要

本ドキュメントでは、
INC-003 手取り収入詳細取得APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

本APIでは、
操作対象となる利用者に登録された
指定の手取り収入を取得し、
詳細情報として画面へ表示する。

取得する情報は、
対象年月、
手取り収入金額および
備考とする。

React・TypeScript実装では、
手取り収入IDを
パスパラメータとしてAPIへ渡し、
レスポンス型を定義することで、
取得したデータを型安全に扱う。

取得した手取り収入は、
主に手取り収入編集画面の
初期表示に利用し、
対象年月、
手取り収入金額および
備考の初期値として設定する。

手取り収入金額が0円の場合も、
未登録とは区別し、
正常な業務データとして表示する。

備考が`null`の場合は、
画面上では空欄として扱う。

指定した手取り収入が存在しない場合や、
操作対象利用者に帰属しない場合など、
APIからエラーが返却された場合は、
共通のエラー処理方針に従って
エラー画面または
エラーメッセージを表示する。

本APIで取得したデータは、
INC-004 手取り収入更新APIを利用する
編集画面の入力値として引き継ぐ。

---

## 2. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type GetNetIncomeDetailParams = {
  netIncomeId: string;
};
```

レスポンス型は、
以下とする。

```ts
export type NetIncomeDetailResponse = {
  data: {
    id: string;
    targetYearMonth: string;
    amount: number;
    memo: string | null;
  };
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<NetIncomeDetailResponse>(
    `/api/v1/net-incomes/${netIncomeId}`,
  );
```

取得したデータは、
手取り収入編集画面の初期表示へ利用する。

取得できなかった場合は、
共通エラー画面または
エラーメッセージを表示する。

`amount`が`0`の場合も、
正常な業務データとして表示する。

`memo`が`null`の場合は、
空欄として表示する。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [INC-001 手取り収入一覧取得](./inc-001-list.md)
- [INC-002 手取り収入登録](./inc-002-create.md)
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