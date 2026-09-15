# INC-001 手取り収入一覧取得

## 1. 概要

本ドキュメントでは、
INC-001 手取り収入一覧取得APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

本APIでは、
操作対象となる利用者に登録された
手取り収入を対象年月単位で取得し、
一覧画面へ表示する。

一覧では、
対象年月、
手取り収入金額および
備考を表示する。

React・TypeScript実装では、
APIとの通信に使用する
クエリパラメータ、
一覧項目、
ページネーション情報および
レスポンスの型を定義し、
APIレスポンスを型安全に扱う。

一覧取得では、
ページ番号、
1ページ当たりの取得件数および
対象年月の並び順を
クエリパラメータとして指定できる。

画面上では、
対象年月の降順を既定とし、
新しい手取り収入から表示する。

取得結果が0件の場合は、
エラーとして扱わず、
手取り収入が未登録であることを示す
空状態を表示する。

また、
手取り収入が登録されていない状態と、
金額が0円として登録されている状態は
明確に区別する。

一覧から手取り収入を編集する場合は、
選択された手取り収入IDを利用して、
INC-003 手取り収入詳細取得APIによる
編集対象データの取得へつなげる。

本APIで取得した一覧データは、
主に手取り収入一覧画面および
平均手取り収入に関連する画面表示で利用する。

---

## 2. React・TypeScriptでの利用

クエリパラメータの型は、以下とする。

```typescript
export type NetIncomeOrder = 'asc' | 'desc';

export type GetNetIncomesQuery = {
  page?: number;
  perPage?: number;
  order?: NetIncomeOrder;
};
```

手取り収入一覧項目の型は、以下とする。

```typescript
export type NetIncomeListItem = {
  id: string;
  targetYearMonth: string;
  amount: number;
  memo: string | null;
};
```

ページネーション情報の型は、以下とする。

```typescript
export type PaginationMeta = {
  currentPage: number;
  perPage: number;
  total: number;
  lastPage: number;
};
```

レスポンス型は、以下とする。

```typescript
export type NetIncomeListResponse = {
  data: NetIncomeListItem[];
  meta: PaginationMeta;
};
```

API呼び出し例は、以下とする。

```typescript
const response =
  await apiClient.get<NetIncomeListResponse>(
    '/api/v1/net-incomes',
    {
      params: {
        page: 1,
        perPage: 20,
        order: 'desc',
      },
    },
  );
```

一覧画面では、対象年月の降順を既定とし、新しい手取り収入から表示する。

取得結果が0件の場合は、エラー画面を表示せず、手取り収入が未登録であることを示す空状態を表示する。

金額が0の場合も、未登録とは区別して表示する。

```text
レコードなし
    → 未入力

amount = 0
    → 手取り収入0円
```

編集画面へ遷移する場合は、対象行の `id` を利用して INC-003 手取り収入詳細取得APIを呼び出す。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [INC-002 手取り収入登録](./inc-002-create.md)
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