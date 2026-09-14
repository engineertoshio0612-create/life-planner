# OBJ-005 目的無効化

## 1. 概要

操作対象となる利用者に登録された
有効な目的を無効化する。

Reactでは、OBJ-003 目的詳細取得などで
対象目的の現在の利用状態を確認し、
利用者による無効化操作を受けて
OBJ-005 目的無効化APIを実行する。

無効化では、目的の利用状態のみを変更する。

```text
enabled = true
    ↓
OBJ-005
    ↓
enabled = false
```

目的名、実施予定年月、必要支出額および
メモなどの目的情報は変更しない。

目的無効化は論理削除とは異なる業務操作であり、
無効化後も目的一覧および目的詳細から
過去の目的情報として参照できる。

無効化された目的は、
新しい目的達成判定の対象外とする。

一方で、無効化以前に保存

---

## 2. React・TypeScriptでの利用

OBJ-005は、指定した目的を無効化する際に使用する。

フロントエンドでは、OBJ-003で目的詳細を取得し、現在の`enabled`状態を確認したうえで、OBJ-005をMutationとして実行する。

概念的な利用フローは、以下とする。

```text
OBJ-003
目的詳細取得
    ↓
enabled = true
    ↓
利用者が無効化を選択
    ↓
確認ダイアログ
    ↓
OBJ-005
PATCH
/api/v1/objectives/{objectiveId}/disabled
    ↓
成功
    ↓
OBJ-001 / OBJ-003
Query Cache無効化
    ↓
最新状態再取得
```

OBJ-005はサーバー状態を変更するAPIであるため、TanStack Queryを使用する場合はMutationとして扱う。

---

### 2.1 TypeScript型

OBJ-005では、Request Bodyを使用しない。

そのため、OBJ-005専用のRequest Body型は定義しない。

Mutation実行に必要な値は、対象となる

```text
objectiveId
```

のみとする。

概念例：

```typescript
export type DisableObjectiveVariables = {
  objectiveId: string;
};
```

---

### 2.2 正常レスポンス型

OBJ-005成功時は、無効化後の目的情報を返却する。

概念例：

```typescript
export type Objective = {
  id: string;
  name: string;
  plannedYearMonth: string;
  requiredExpense: number;
  memo: string | null;
  enabled: boolean;
};
```

API共通Envelopeを使用する場合は、以下のように定義する。

```typescript
export type DisableObjectiveResponse =
  ApiResponse<Objective>;
```

---

### 2.3 OBJ-001・OBJ-003と共通型を使用してよい

OBJ-001、OBJ-003、OBJ-005で目的1件のレスポンス項目が同じである場合は、共通の`Objective`型を使用してよい。

概念例：

```typescript
export type Objective = {
  id: string;
  name: string;
  plannedYearMonth: string;
  requiredExpense: number;
  memo: string | null;
  enabled: boolean;
};
```

APIごとに同じ意味の型を重複定義しない。

---

### 2.4 objectiveId

`objectiveId`は、API Client関数の引数として渡す。

概念例：

```typescript
disableObjective(
  objectiveId,
);
```

API契約に合わせて、`objectiveId`は`string`として扱う。

---

### 2.5 userIdをAPI Client引数へ含めない

以下のような関数にはしない。

```typescript
disableObjective(
  userId,
  objectiveId,
);
```

利用者IDは、共通API Clientから

```text
X-User-Id
```

として付与する。

---

### 2.6 X-User-Id

`X-User-Id`は、OBJ-005専用処理ではなく、共通API Clientから付与する。

概念例：

```typescript
apiClient.interceptors.request.use(
  (config) => {
    config.headers['X-User-Id'] =
      currentUserId;

    return config;
  },
);
```

各コンポーネントやHookから直接ヘッダーを生成しない。

---

### 2.7 Request Bodyを送信しない

OBJ-005では、以下のようなRequest Bodyを送信しない。

```json
{
  "enabled": false
}
```

目的を無効化することは、

```text
PATCH /api/v1/objectives/{objectiveId}/disabled
```

というエンドポイント自体で表現する。

---

### 2.8 enabledをクライアントから指定しない

以下のようなAPI Clientにはしない。

```typescript
disableObjective(
  objectiveId,
  false,
);
```

または、

```typescript
disableObjective(
  objectiveId,
  {
    enabled: false,
  },
);
```

OBJ-005では、呼び出された時点で

```text
enabled = false
```

へ変更することが確定している。

---

### 2.9 API Client

OBJ-005を呼び出す専用API Client関数を定義する。

概念例：

```typescript
export const disableObjective =
  async (
    objectiveId: string,
  ): Promise<Objective> => {
    const response =
      await apiClient.patch<
        DisableObjectiveResponse
      >(
        `/api/v1/objectives/${objectiveId}/disabled`,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

### 2.10 `/disabled`をURLへ含める

OBJ-005では、必ず

```text
/disabled
```

を含むURLを使用する。

```typescript
`/api/v1/objectives/${objectiveId}/disabled`
```

以下のOBJ-004用URLへ誤ってPATCHしない。

```typescript
`/api/v1/objectives/${objectiveId}`
```

---

### 2.11 OBJ-004とのAPI Client分離

目的更新と目的無効化は、別API Client関数とする。

概念的には、

```text
updateObjective()
    ↓
OBJ-004

disableObjective()
    ↓
OBJ-005
```

とする。

以下のような汎用関数へ無理にまとめない。

```typescript
updateObjective(
  objectiveId,
  {
    enabled: false,
  },
);
```

---

### 2.12 Mutationとして扱う

OBJ-005はサーバー状態を変更するため、TanStack QueryではMutationとして扱う。

概念例：

```typescript
export const useDisableObjective =
  () => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn: (
        objectiveId: string,
      ) =>
        disableObjective(
          objectiveId,
        ),
    });
  };
```

---

### 2.13 Mutation Variables

画面によっては、Mutation Variablesとしてオブジェクト形式を使用してよい。

概念例：

```typescript
export type DisableObjectiveVariables = {
  objectiveId: string;
};
```

```typescript
mutationFn: (
  variables:
    DisableObjectiveVariables,
) =>
  disableObjective(
    variables.objectiveId,
  ),
```

今後引数が増える可能性を考慮する場合に使用してよい。

---

### 2.14 Query Key

目的関連のQuery Keyは、共通定義として管理してよい。

概念例：

```typescript
export const objectiveKeys = {
  all: [
    'objectives',
  ] as const,

  list: () =>
    [
      'objectives',
      'list',
    ] as const,

  detail: (
    objectiveId: string,
  ) =>
    [
      'objectives',
      'detail',
      objectiveId,
    ] as const,
};
```

正式なQuery Key設計は、React共通設計に従う。

---

### 2.15 OBJ-001のQuery Cacheを無効化する

OBJ-005成功後は、目的一覧における

```text
enabled
```

の状態が変化する。

そのため、OBJ-001の一覧Queryを無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    objectiveKeys.list(),
});
```

---

### 2.16 OBJ-003のQuery Cacheを無効化する

対象目的の詳細も、

```text
enabled = false
```

へ変化する。

そのため、OBJ-003の詳細Queryも無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    objectiveKeys.detail(
      objectiveId,
    ),
});
```

---

### 2.17 Mutation Hook

概念例：

```typescript
export const useDisableObjective =
  () => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn: (
        objectiveId: string,
      ) =>
        disableObjective(
          objectiveId,
        ),

      onSuccess: async (
        _result,
        objectiveId,
      ) => {
        await Promise.all([
          queryClient.invalidateQueries({
            queryKey:
              objectiveKeys.list(),
          }),

          queryClient.invalidateQueries({
            queryKey:
              objectiveKeys.detail(
                objectiveId,
              ),
          }),
        ]);
      },

      retry: false,
    });
  };
```

---

### 2.18 成功レスポンスで詳細Cacheを更新してもよい

OBJ-005成功レスポンスでは、更新後の目的情報が返却される。

そのため、技術的には

```typescript
queryClient.setQueryData(
  objectiveKeys.detail(
    result.id,
  ),
  result,
);
```

のように、OBJ-003のCacheを直接更新してもよい。

ただし、Phase1では実装を単純化するため、

```text
OBJ-005成功
    ↓
invalidate
    ↓
再取得
```

を基本としてよい。

---

### 2.19 目的一覧は再取得を基本とする

OBJ-001については、一覧条件によって

```text
無効化済み目的を残す
```

場合と、

```text
有効目的だけ表示して
対象が一覧から消える
```

場合があり得る。

そのため、一覧CacheをReact側で手動編集するより、OBJ-001を再取得する方式を基本とする。

---

### 2.20 無効化ボタン

目的詳細画面などに、無効化操作を配置する。

概念例：

```tsx
<button
  type="button"
  onClick={
    handleDisable
  }
>
  目的を無効化
</button>
```

---

### 2.21 enabled = falseの場合は無効化操作を表示しない

OBJ-003で取得した目的が、

```text
enabled = false
```

の場合は、通常、無効化ボタンを表示しない。

概念例：

```tsx
{objective.enabled && (
  <button
    type="button"
    onClick={
      handleDisable
    }
  >
    目的を無効化
  </button>
)}
```

---

### 2.22 フロントエンドだけで無効化済みを保証しない

画面上で無効化ボタンを非表示にしても、バックエンド側の

```text
OBJECTIVE_DISABLED
```

判定は必要とする。

画面表示後に別タブや別Requestで目的が無効化される可能性があるためである。

---

### 2.23 確認ダイアログ

目的無効化は、今後の目的達成判定対象から外れる操作であるため、実行前に確認ダイアログを表示してよい。

概念例：

```text
「この目的を無効化しますか？」

無効化後は、
新しい目的達成判定の
対象外になります。
```

実際の文言は、画面設計に従う。

---

### 2.24 確認ダイアログで目的名を表示してよい

誤操作防止のため、例えば、

```text
「一人暮らし」を
無効化しますか？
```

のように、対象目的名を表示してよい。

目的名は、OBJ-003などですでに取得済みの値を使用する。

---

### 2.25 再有効化できないことを伝えてよい

Phase1では再有効化APIを提供しないため、必要に応じて確認ダイアログで、

```text
Phase1では、
無効化後に再有効化できません。
```

相当の注意を画面設計に応じて表示してよい。

---

### 2.26 二重送信防止

Mutation実行中は、無効化ボタンを非活性化する。

概念例：

```tsx
<button
  type="button"
  disabled={
    mutation.isPending
  }
  onClick={
    handleDisable
  }
>
  {mutation.isPending
    ? '無効化中...'
    : '目的を無効化'}
</button>
```

---

### 2.27 二重送信防止は補助とする

フロントエンドでボタンを非活性化しても、同時実行や直接API実行は可能である。

そのため、整合性の最終保証はLaravel側の

```text
enabled = true
```

を条件とした更新で行う。

---

### 2.28 Mutationの自動Retry

OBJ-005では、Mutationの無条件な自動Retryを行わない。

概念例：

```typescript
useMutation({
  mutationFn:
    disableObjective,
  retry: false,
});
```

更新APIであるため、通信結果不明時に同じRequestを自動的に再送しない。

---

### 2.29 通信結果不明時

OBJ-005送信後に通信エラーとなり、

```text
サーバー側では成功したか
失敗したか
```

を判断できない場合は、OBJ-003を再取得して最新の

```text
enabled
```

を確認する。

概念的には、

```text
OBJ-005
    ↓
通信結果不明
    ↓
OBJ-003再取得
    ↓
enabled確認
```

とする。

---

### 2.30 成功時

OBJ-005成功時は、

```http
200 OK
```

となる。

レスポンスの

```text
enabled
```

は、必ず

```text
false
```

となる。

---

### 2.31 成功メッセージ

必要に応じて、

```text
目的を無効化しました。
```

などの完了メッセージを表示する。

---

### 2.32 成功後の画面

画面設計に応じて、以下のいずれかとしてよい。

* 目的詳細画面に留まる
* 目的一覧へ戻る
* 詳細を再取得して無効状態を表示する

例えば、詳細画面に留まる場合は、

```text
OBJ-005成功
    ↓
OBJ-003再取得
    ↓
enabled = false
    ↓
無効化済み表示
```

とする。

---

### 2.33 無効化済み表示

無効化後の目的には、必要に応じて

```text
無効
無効化済み
利用停止中
```

などの状態表示を行う。

正式な表示文言は、画面設計・用語集に従う。

---

### 2.34 OBJECTIVE_DISABLED

以下のエラーを受信した場合は、

```text
OBJECTIVE_DISABLED
```

対象目的がすでに無効化されている。

例えば、

```text
この目的はすでに無効化されています。
```

などを表示してよい。

---

### 2.35 OBJECTIVE_DISABLED受信後は再取得する

画面表示時には

```text
enabled = true
```

だったが、別Requestによって先に無効化された可能性がある。

そのため、

```text
OBJECTIVE_DISABLED
    ↓
OBJ-003再取得
```

として最新状態へ同期してよい。

---

### 2.36 OBJECTIVE_NOT_FOUND

以下の場合は、

```text
OBJECTIVE_NOT_FOUND
```

となる。

* 目的不存在
* 他利用者所属
* 論理削除済み

フロントエンドでは理由を推測せず、

```text
指定された目的が見つかりません。
```

などの共通表示とする。

---

### 2.37 他利用者かどうかを推測しない

`OBJECTIVE_NOT_FOUND`を受けても、

```text
他の利用者の目的です。
```

などと表示しない。

API契約上、不存在との区別はできない。

---

### 2.38 INVALID_OBJECTIVE_ID

通常画面では、OBJ-001やOBJ-003から取得した有効なIDを使用するため、発生頻度は低い。

URL手入力等で発生した場合は、不正な画面状態として扱い、目的一覧へ戻す導線を表示してよい。

---

### 2.39 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

OBJ-005専用のエラー処理を各コンポーネントへ実装しない。

---

### 2.40 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、利用者コンテキストに関する共通エラーとして扱う。

---

### 2.41 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択されている利用者が有効ではない状態として共通処理する。

---

### 2.42 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
目的を無効化できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

### 2.43 エラーコードで分岐する

フロントエンドでは、`message`文字列ではなく、

```text
error.code
```

を基準として分岐する。

概念例：

```typescript
switch (error.code) {
  case 'OBJECTIVE_NOT_FOUND':
    // 対象不存在
    break;

  case 'OBJECTIVE_DISABLED':
    // 無効化済み
    break;

  default:
    // 共通エラー
    break;
}
```

---

### 2.44 OBJ-004との画面責務分離

同じ目的詳細画面で、目的情報の編集と無効化を提供する場合でも、別操作として扱う。

```text
目的編集
    ↓
OBJ-004

目的無効化
    ↓
OBJ-005
```

編集フォームに

```text
enabled
```

を含めない。

---

### 2.45 編集フォームからenabledを除外する

OBJ-004用フォーム型は、概念的に

```typescript
export type UpdateObjectiveFormValues = {
  name: string;
  plannedYearMonth: string;
  requiredExpense: number;
  memo: string | null;
};
```

とし、

```typescript
enabled: boolean;
```

を含めない。

利用状態変更をOBJ-004へ混在させない。

---

### 2.46 過去の目的達成判定履歴をReact側で削除しない

OBJ-005成功後も、保存済みの目的達成判定履歴は残る。

そのため、React側で

```text
目的無効化
    ↓
過去判定履歴Cache削除
```

のような処理を無条件に行わない。

---

### 2.47 新規判定対象からは除外する

新しい目的達成判定を行う画面で、有効目的だけを使用する場合は、OBJ-005成功後に対象となるQueryを再取得する。

無効化済み目的が新規判定対象として表示され続けないようにする。

---

### 2.48 利用者切替時

Phase1では、`X-User-Id`によって操作対象利用者を切り替える。

利用者切替時に、前利用者の目的情報やMutation後のCacheを新しい利用者へ誤表示しない。

---

### 2.49 Query KeyへuserIdを含めてもよい

利用者境界をCache上でも明確にする場合は、以下のようにしてよい。

```typescript
export const objectiveKeys = {
  detail: (
    userId: string,
    objectiveId: string,
  ) =>
    [
      'objectives',
      userId,
      'detail',
      objectiveId,
    ] as const,
};
```

正式な方式は、React共通設計に従う。

---

### 2.50 Pageの責務

目的詳細Pageでは、主に以下を担当する。

* URLから`objectiveId`取得
* OBJ-003による目的詳細取得
* 無効化確認UIの表示
* OBJ-005実行
* 成功後の表示更新
* ページ単位のエラー表示

HTTP通信処理そのものは、API ClientやMutation Hookへ委譲する。

---

### 2.51 Disable Componentの責務

無効化操作用コンポーネントでは、主に以下を担当する。

* 無効化ボタン表示
* `enabled`による表示制御
* 確認ダイアログ表示
* Mutation実行イベント通知
* 処理中状態表示

DB状態の最終判定は行わない。

---

### 2.52 Mutation Hookの責務

Mutation Hookでは、主に以下を担当する。

```text
OBJ-005実行
Mutation状態管理
成功時Query invalidate
```

画面固有の確認ダイアログ文言やレイアウトをMutation Hookへ持たせない。

---

### 2.53 API Clientの責務

API Clientでは、

```text
PATCH
/api/v1/objectives/{objectiveId}/disabled
```

のHTTP通信と、型付きレスポンス取得を担当する。

以下をAPI Clientへ含めない。

* Toast表示
* 確認ダイアログ
* 画面遷移
* 無効化済み表示
* Query Cache操作

---

### 2.54 概念的なディレクトリ構成

例えば、以下のように整理できる。

```text
features/
└── objectives/
    ├── api/
    │   ├── getObjectives.ts
    │   ├── getObjectiveDetail.ts
    │   ├── updateObjective.ts
    │   └── disableObjective.ts
    ├── components/
    │   ├── ObjectiveDetail.tsx
    │   ├── ObjectiveForm.tsx
    │   └── DisableObjectiveButton.tsx
    ├── hooks/
    │   ├── useObjectives.ts
    │   ├── useObjectiveDetail.ts
    │   ├── useUpdateObjective.ts
    │   └── useDisableObjective.ts
    ├── types/
    │   └── objective.ts
    └── pages/
        └── ObjectiveDetailPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

### 2.55 フロントエンドで行わないこと

OBJ-005のReact・TypeScript実装では、以下をフロントエンドの責務としない。

* 利用者境界の最終保証
* 目的存在確認の最終保証
* 論理削除判定
* 無効化済み状態の最終保証
* 同時更新制御
* `enabled`のDB更新
* `updated_at`の更新
* 過去判定履歴の変更
* 再有効化
* Request Bodyによる`enabled`指定

フロントエンドは、

```text
目的状態取得
    ↓
利用者へ無効化操作を提示
    ↓
OBJ-005実行
    ↓
結果表示
    ↓
関連Query再取得
```

という責務を基本とする。

---

## 3. 関連ドキュメント

* [API一覧](../../api-list.md)
* [API共通方針](../../api-common-policy.md)
* [OBJ-001 目的一覧取得](./obj-001-list.md)
* [OBJ-002 目的登録](./obj-002-create.md)
* [OBJ-003 目的詳細取得](./obj-003-detail.md)
* [OBJ-004 目的更新](./obj-004-update.md)
* [OBJ-006 目的達成判定](./obj-006-disabled.md)
* [機能要件](../../../requirements/functional-requirements.md)
* [ユビキタス言語集](../../../glossary.md)
* [エンティティ定義](../../../entities.md)
* [テーブル定義書](../../../table-definition.md)
* [ER図](../../../er-diagram-phase1.md)
* [Laravelアーキテクチャ設計](../../../architecture/laravel/objectives/README.md)
* [Reactアーキテクチャ設計](../../../architecture/react/objectives/README.md)
* [バックエンドテスト方針](../../../tests/backend/README.md)
* [フロントエンドテスト方針](../../../tests/frontend/README.md)
* [E2Eテスト方針](../../../tests/e2e/README.md)
