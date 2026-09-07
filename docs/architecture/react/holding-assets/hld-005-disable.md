# HLD-005 保有商品無効化

## 1. 概要

本ドキュメントでは、HLD-005 保有商品無効化APIをReact・TypeScriptから利用する際のフロントエンドアーキテクチャおよび責務分離方針を定義する。

本機能では、操作対象となる利用者に帰属する利用中の保有商品を、今後の管理対象から無効化する。

フロントエンドでは、保有商品一覧または保有商品詳細画面から無効化操作を実行する。

保有商品の無効化は通常属性の更新とは異なる業務操作として扱い、HLD-004 保有商品更新APIへ状態フラグを送信する方式ではなく、HLD-005専用のエンドポイントを使用する。

React実装では、画面表示、確認ダイアログ、Mutation、API通信、成功・失敗時の処理およびサーバー状態との同期について責務を分離し、以下の構成を基本とする。

```text
Page / Component
    ↓
確認ダイアログ
    ↓
Mutation Hook
    ↓
API Client
    ↓
HLD-005
    ↓
Query Cache無効化
    ↓
一覧・詳細の最新状態を反映
```

HLD-005ではリクエストボディを使用せず、無効化対象となる`holdingAssetId`のみをAPI Clientへ渡す。`isEnabled`、`disabled`、`deletedAt`などの状態値をフロントエンドから送信しない。

無効化はPhase1では再有効化できない操作であるため、実行前に確認ダイアログを表示し、Mutation実行中は操作ボタンを非活性化して二重送信を防止する。

正常終了時は、APIから返却された無効化結果を確認したうえで、保有商品一覧および対象となる保有商品詳細などの関連Query Cacheを無効化し、サーバー側の最新状態へ同期する。

詳細画面から無効化した場合は、無効化後の対象が通常の詳細取得対象から除外されることを考慮し、保有商品一覧など画面設計で定めた遷移先へ移動する。

APIから対象不存在、利用者コンテキストエラーまたはサーバーエラーが返却された場合は、共通エラー処理に従って利用者へ通知する。

Phase1ではOptimistic Updateおよび再有効化UIは必須とせず、HLD-005の成功を確認してからQuery Cacheを更新することで、サーバー状態を正として画面へ反映する。


---

## 2. React・TypeScriptでの利用

HLD-005は、利用者が保有商品を今後の管理対象から外す際に使用する。

フロントエンドでは、保有商品一覧または保有商品詳細画面から無効化操作を実行する。

概念的な利用フローは、以下とする。

```text
保有商品一覧
または
保有商品詳細
    ↓
無効化ボタン押下
    ↓
確認ダイアログ
    ↓
HLD-005
PATCH
/api/v1/holding-assets/{holdingAssetId}/disable
    ↓
成功
    ↓
関連Query Cache無効化
    ↓
一覧・詳細を再取得
```

HLD-005では、無効化状態をリクエストボディから指定しない。

```text
/disable
```

というエンドポイント自体が無効化操作を表す。

---

### 2.1 TypeScript型

正常レスポンスは、以下のような型として扱う。

概念例：

```ts
export type DisableHoldingAssetResult = {
  id: string;
  disabled: true;
};

export type DisableHoldingAssetResponse = {
  data: DisableHoldingAssetResult;
  requestId: string;
};
```

API共通Envelopeの正式な型が存在する場合は、共通型を利用する。

概念例：

```ts
export type DisableHoldingAssetResponse =
  ApiResponse<DisableHoldingAssetResult>;
```

---

### 2.2 disabledの型

HLD-005の正常レスポンスでは、

```text
disabled = true
```

が固定である。

そのため、TypeScriptでは

```ts
disabled: true;
```

としてリテラル型で表現してよい。

単純に

```ts
disabled: boolean;
```

としてもよいが、HLD-005では`false`が返却されないことを型でも表現できる。

---

### 2.3 リクエスト型

HLD-005では、リクエストボディを使用しない。

そのため、以下のようなリクエストDTO型は作成しない。

```ts
export type DisableHoldingAssetRequest = {
  isEnabled: false;
};
```

または、

```ts
export type DisableHoldingAssetRequest = {
  disabled: true;
};
```

無効化対象は、関数引数の

```text
holdingAssetId
```

だけで指定する。

---

### 2.4 API Client

API Clientでは、`holdingAssetId`を受け取り、HLD-005を呼び出す専用関数を定義する。

概念例：

```ts
export const disableHoldingAsset =
  async (
    holdingAssetId: string,
  ): Promise<DisableHoldingAssetResult> => {
    const response =
      await apiClient.patch<
        DisableHoldingAssetResponse
      >(
        `/api/v1/holding-assets/${holdingAssetId}/disable`,
      );

    return response.data.data;
  };
```

第2引数としてリクエストボディを渡す必要はない。

HTTP Clientの仕様上、明示的に空ボディが必要な場合のみ、

```ts
null
```

などを指定する。

---

### 2.5 X-User-Id

`X-User-Id`は、他のAPIと同様に共通API Clientから付与する。

概念例：

```ts
apiClient.interceptors.request.use(
  (config) => {
    config.headers[
      'X-User-Id'
    ] = currentUserId;

    return config;
  },
);
```

HLD-005専用処理で利用者IDを以下へ追加しない。

* URL
* クエリパラメータ
* リクエストボディ

---

### 2.6 Mutationとして扱う

HLD-005は、保有商品の状態を変更するため、TanStack Queryを使用する場合はMutationとして扱う。

概念例：

```ts
export const useDisableHoldingAsset =
  () =>
    useMutation({
      mutationFn:
        disableHoldingAsset,
    });
```

Queryとして自動実行しない。

---

### 2.7 無効化ボタン

保有商品一覧や詳細画面では、無効化対象となる保有商品IDを渡してMutationを実行する。

概念例：

```tsx
<button
  type="button"
  onClick={() => {
    handleDisable(
      holdingAsset.id,
    );
  }}
>
  無効化
</button>
```

---

### 2.8 確認ダイアログ

保有商品無効化は、Phase1では再有効化機能を提供しない。

そのため、誤操作防止として実行前に確認ダイアログを表示する。

概念例：

```text
この保有商品を無効化しますか？

無効化後は、
今後の商品別月末評価額の
登録対象から除外されます。
```

過度に長い説明は避け、利用者が操作結果を理解できる内容とする。

---

### 2.9 確認ダイアログで表示してよい情報

必要に応じて、以下を表示してよい。

* 保有商品名
* 所属資産口座名

例えば、

```text
「全世界株式」を無効化しますか？
```

のように表示する。

内部IDは利用者へ表示する必要はない。

---

### 2.10 無効化処理

概念例：

```ts
const disableMutation =
  useDisableHoldingAsset();

const handleDisable =
  (
    holdingAssetId: string,
  ): void => {
    disableMutation.mutate(
      holdingAssetId,
    );
  };
```

無効化実行時に、

```text
isEnabled = false
```

などの値を送信しない。

---

### 2.11 二重送信防止

Mutation実行中は、無効化ボタンを非活性化する。

概念例：

```tsx
<button
  type="button"
  disabled={
    disableMutation.isPending
  }
  onClick={() => {
    handleDisable(
      holdingAsset.id,
    );
  }}
>
  {disableMutation.isPending
    ? '無効化中...'
    : '無効化'}
</button>
```

ただし、フロントエンド側の二重送信防止だけを整合性保証とはしない。

---

### 2.12 成功時

HLD-005成功時は、

```text
200 OK
```

とともに、

```json
{
  "id": "15",
  "disabled": true
}
```

が返却される。

フロントエンドでは、`disabled = true`を確認して無効化完了として扱う。

---

### 2.13 成功メッセージ

正常終了後は、必要に応じて

```text
保有商品を無効化しました。
```

のような完了メッセージを表示する。

成功レスポンスから保有商品名を取得する設計ではないため、商品名を含めたい場合は画面側ですでに保持している情報を表示用途として使用してよい。

---

### 2.14 Query Cacheの無効化

HLD-005成功後は、保有商品に関するQuery Cacheを無効化する。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey: [
    'holdingAssets',
  ],
});
```

保有商品詳細Queryをキャッシュしている場合は、対象IDのQueryも無効化する。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey: [
    'holdingAsset',
    result.id,
  ],
});
```

正式なQuery Keyは、フロントエンド共通設計に従う。

---

### 2.15 一覧画面

保有商品一覧が有効な保有商品のみを表示する設計である場合、HLD-005成功後に一覧Queryを再取得すると、無効化した保有商品は一覧から除外される。

フロントエンドで独自に

```ts
items.filter(
  (item) =>
    item.id !== result.id,
);
```

としてもよいが、サーバー状態との整合性を優先する場合はQuery invalidationによる再取得を基本とする。

---

### 2.16 詳細画面

保有商品詳細画面からHLD-005を実行した場合、無効化後は通常のHLD-003詳細取得対象から除外される。

そのため、成功後は以下のような画面遷移を行う。

* 保有商品一覧へ戻る
* 無効化完了画面へ遷移する

成功後に同じ詳細APIをそのまま再取得すると、

```text
HOLDING_ASSET_NOT_FOUND
```

となることを前提とする。

---

### 2.17 HLD-004との使い分け

フロントエンドでは、以下を明確に分ける。

```text
保有商品情報編集
    ↓
HLD-004

保有商品無効化
    ↓
HLD-005
```

HLD-004の更新フォームへ以下の状態変更項目を追加しない。

```text
isEnabled
disabled
deletedAt
```

---

### 2.18 HLD-004 API Client

通常更新は、引き続き

```http
PATCH /api/v1/holding-assets/{holdingAssetId}
```

を使用する。

概念的には、

```ts
updateHoldingAsset(
  holdingAssetId,
  request,
);
```

とする。

---

### 2.19 HLD-005 API Client

無効化は、

```http
PATCH /api/v1/holding-assets/{holdingAssetId}/disable
```

を使用する。

概念的には、

```ts
disableHoldingAsset(
  holdingAssetId,
);
```

とする。

API Clientの関数名でも通常更新と無効化を明確に区別する。

---

### 2.20 状態フラグをフロントエンドで送らない

以下のような汎用状態変更関数はPhase1では作成しない。

```ts
updateHoldingAssetStatus(
  holdingAssetId,
  false,
);
```

または、

```ts
updateHoldingAsset(
  holdingAssetId,
  {
    disabled: true,
  },
);
```

無効化という業務操作を明示する

```ts
disableHoldingAsset(
  holdingAssetId,
);
```

を使用する。

---

### 2.21 無効化済み状態をローカルだけで保持しない

HLD-005成功後に、React Stateだけで

```text
disabled = true
```

へ変更して永続的な状態とみなさない。

サーバー側ではSoftDeletesによって通常取得対象から除外されるため、関連Queryを再取得して最新状態へ同期する。

---

### 2.22 404 HOLDING_ASSET_NOT_FOUND

以下の場合は、

```text
HOLDING_ASSET_NOT_FOUND
```

が返却される。

* 保有商品不存在
* 他利用者の保有商品
* すでに無効化済み
* 無効な資産口座配下の保有商品

フロントエンドでは、原因ごとにリソース存在状態を推測しない。

「対象の保有商品が見つかりません」などの共通表示とする。

---

### 2.23 二重実行時

1回目のHLD-005が正常終了した後、何らかの理由で同じIDに対して再度HLD-005を実行すると、

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

となる。

フロントエンドでは、このレスポンスを受けた場合も一覧を再取得することで、現在状態を確認できる。

---

### 2.24 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

HLD-005専用画面で独自処理を実装しない。

---

### 2.25 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、API共通の利用者コンテキストエラーとして扱う。

必要に応じて、利用者選択状態を再確認する。

---

### 2.26 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合も、API共通エラー処理に従う。

現在選択されている利用者が有効でない状態として扱う。

---

### 2.27 INVALID_HOLDING_ASSET_ID

通常のUI操作では、サーバーから取得した有効な保有商品IDを使用するため、発生頻度は低い。

発生した場合は、不正な画面状態またはURLとして共通エラー処理を行う。

---

### 2.28 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
保有商品を無効化できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

成功扱いとして一覧から対象商品を削除してはならない。

---

### 2.29 エラー時のQuery Cache

HLD-005が失敗した場合は、原則として保有商品Query Cacheを成功時と同じ扱いで更新しない。

サーバー側で無効化が完了していることを確認できないためである。

ただし、通信断などで結果が不明な場合は、必要に応じて一覧Queryを再取得して最新状態を確認してよい。

---

### 2.30 Optimistic Update

Phase1では、HLD-005に対するOptimistic Updateは必須としない。

保有商品無効化は頻繁に実行する操作ではなく、失敗時に一覧へ戻す処理も必要になるため、サーバー成功後にQuery Cacheを更新する単純な方式を基本とする。

---

### 2.31 再有効化UIを作成しない

Phase1では、無効化済み保有商品を再有効化するAPIを提供しない。

そのため、フロントエンドにも

```text
再有効化
有効に戻す
```

などの操作UIを設けない。

同じ商品を再度管理する場合は、新規保有商品登録の導線を使用する。

---

### 2.32 deletedAtを画面制御に使用しない

HLD-005成功レスポンスでは、

```text
deletedAt
```

を返却しない。

フロントエンドでは、`deletedAt`を参照して無効化成功を判定しない。

正常レスポンスの

```text
disabled = true
```

とHTTP成功を使用する。

---

### 2.33 disabledを更新APIの状態管理へ流用しない

HLD-005レスポンスの

```text
disabled
```

は、無効化操作の結果確認用である。

保有商品エンティティ全体に

```ts
disabled: boolean;
```

を必ず持たせることを意味しない。

HLD-001やHLD-003で無効化済み商品を通常返却しない設計であれば、一覧・詳細DTOへ`disabled`を追加する必要はない。

---

### 2.34 保有商品の型を不要に変更しない

HLD-005を追加したことだけを理由として、既存の

```ts
export type HoldingAsset = {
  // ...
};
```

へ

```ts
disabled: boolean;
deletedAt: string | null;
```

などを追加しない。

各APIが実際に返却するレスポンス契約に合わせて型を定義する。

---

### 2.35 APIエンドポイントを共通更新関数へ隠しすぎない

以下のような何でも処理できる汎用関数へまとめすぎない。

```ts
changeHoldingAsset(
  holdingAssetId,
  action,
  payload,
);
```

Phase1では、

```ts
updateHoldingAsset();
disableHoldingAsset();
```

のように、業務操作が分かる関数名を使用する。

これにより、フロントエンドコードからもHLD-004とHLD-005の責務が分かるようにする。

---

## 3. 関連ドキュメント

- [HLD-005 API詳細設計](../../../api/details/holding-assets/hld-005-disable.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Reactアーキテクチャ共通設計](../react-architecture.md)
- [保有商品 Reactアーキテクチャ設計](./README.md)
- [HLD-005 テスト設計](../../../tests/holding-assets/hld-005-disable.md)
- [保有商品 テスト設計](../../../tests/holding-assets/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
