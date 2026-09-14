# OBJ-002 目的登録

## 1. 概要

操作対象となる利用者の目的を
Reactの目的登録画面から新規登録する。

目的には、主に以下の情報を入力する。

* 目的名
* 実施予定年月
* 必要支出額
* メモ

操作対象となる利用者は、
共通の利用者コンテキストによって管理するため、
利用者IDを登録フォームから入力または送信しない。

また、登録直後の目的は利用中として扱うため、
利用状態をフロントエンドから指定しない。

入力された目的情報を
OBJ-002 目的登録APIへ送信し、
登録成功時には登録された目的情報を取得する。

登録した目的は、
目的一覧および目的詳細などの画面で利用し、
目的達成判定の対象として扱う。

目的一覧をキャッシュしている場合は、
登録成功後に再取得またはキャッシュを無効化し、
サーバー側の最新状態と画面表示の整合性を保つ。

入力値に問題がある場合は、
バックエンドから返却されたバリデーションエラーを
対応するフォーム項目へ表示する。

同一利用者に同名の有効な目的が存在する場合は、
目的名の重複を利用者へ通知し、
登録処理を完了したものとして扱わない。

---

## 2. React・TypeScriptでの利用

OBJ-002は、React側から新しい目的を登録する際に使用する。

React・TypeScriptに共通するAPIクライアント、利用者コンテキスト、エラーハンドリング等の方針は`docs/architecture/react-architecture.md`に従い、本節ではOBJ-002固有の利用方法を記載する。

---

### 2.1 Request型

目的登録時のリクエスト型は以下とする。

```ts
export type CreateObjectiveRequest = {
  name: string;
  plannedYearMonth: string;
  requiredExpense: number;
  memo?: string | null;
};
```

`userId`はRequest型に含めない。

操作対象利用者は共通のAPIクライアントで管理し、`X-User-Id`としてリクエストヘッダーへ付与する。

また、登録時の利用状態はバックエンド側で決定するため、`enabled`もRequest型には含めない。

---

### 2.2 Response型

目的登録成功時のレスポンス型は以下とする。

```ts
export type CreateObjectiveResponse = {
  data: {
    id: string;
    name: string;
    plannedYearMonth: string;
    requiredExpense: number;
    memo: string | null;
    enabled: boolean;
  };
};
```

`id`はAPI共通方針に従い、TypeScript側でも`string`として扱う。

---

### 2.3 API Client

API Clientでは、目的登録用の関数を定義する。

概念例：

```ts
export async function createObjective(
  request: CreateObjectiveRequest,
): Promise<CreateObjectiveResponse> {
  return apiClient.post('/api/v1/objectives', request);
}
```

`X-User-Id`の付与など、API全体に共通するヘッダー制御を本関数へ個別実装しない。

---

### 2.4 登録フォーム

目的登録画面では、主に以下の入力項目を扱う。

```text
目的名
実施予定年月
必要支出額
メモ
```

利用者IDおよび利用状態はフォーム項目として扱わない。

フロントエンド側でも入力チェックを行ってよいが、バックエンド側のバリデーションを正とする。

---

### 2.5 登録成功時

目的登録に成功した場合は、返却された目的を利用して画面を更新する。

目的一覧をキャッシュしている場合は、登録成功後に目的一覧を再取得またはキャッシュ無効化する。

```text
目的登録
    ↓
201 Created
    ↓
目的一覧を更新
    ↓
登録結果を画面へ反映
```

具体的なキャッシュ管理方法は、React共通設計に従う。

---

### 2.6 エラー時

OBJ-002固有でフロントエンドが主に考慮する業務エラーは以下とする。

```text
OBJECTIVE_NAME_DUPLICATED
```

このエラーを受け取った場合は、同一利用者に同名の有効な目的が存在することを利用者へ通知する。

入力値のバリデーションエラーについては、該当するフォーム項目へエラー内容を表示する。

利用者コンテキストや想定外エラーなどの共通エラー処理は、React共通設計に従う。

---

### 2.7 OBJ-002固有の利用上の注意

* `userId`をリクエストボディへ含めない。
* `enabled`を登録フォームから指定しない。
* `id`はTypeScriptでも`string`として扱う。
* 登録成功後は目的一覧との整合性を保つ。
* `OBJECTIVE_NAME_DUPLICATED`は目的登録フォームで扱う業務エラーとする。
* フロントエンドの入力チェックだけに依存せず、バックエンドから返却されたバリデーション結果を適切に表示する。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [OBJ-001 目的一覧取得](./obj-001-list.md)
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
