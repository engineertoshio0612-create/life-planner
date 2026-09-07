# HLD-001 保有商品一覧取得

## 1. 概要

本ドキュメントでは、HLD-001 保有商品一覧取得APIをReact・TypeScriptから利用する際のフロントエンド実装方針を定義する。

本APIは、操作対象利用者に帰属する保有商品を取得し、保有商品一覧画面へ表示するために使用する。

一覧では、利用中の保有商品だけでなく無効化済みの保有商品も表示対象とし、主に以下の情報を表示する。

* 保有商品名
* 所属資産口座
* 利用開始年月
* 利用状態

React・TypeScript実装では、主に以下の処理を行う。

* HLD-001 保有商品一覧取得APIを呼び出す
* APIレスポンスをTypeScriptの型として扱う
* TanStack Queryを使用して取得状態を管理する
* Loading、Error、データなしの状態を適切に表示する
* 取得した保有商品をAPIで保証された順序のまま一覧表示する
* `isEnabled`を利用して利用中・無効化済みを画面上で判別できるようにする
* 保有商品詳細、登録、更新および無効化機能への導線として利用する

保有商品の利用者境界、商品単位で管理する資産口座への限定、利用状態の判定および表示順の決定はバックエンドの責務とする。

フロントエンドではこれらの業務ルールを再判定せず、HLD-001から返却された結果をAPI契約に従って表示する構成とする。

## 2. React・TypeScriptでの利用

HLD-001は、操作対象利用者に帰属する保有商品の一覧を取得し、保有商品一覧画面へ表示するために使用する。

本APIは読み取り専用のGET APIであるため、TanStack QueryではQueryとして扱う。

```text
保有商品一覧画面
    ↓
Query Hook
    ↓
HLD-001
    ↓
保有商品一覧取得
    ↓
Query Cache
    ↓
一覧表示
```

フロントエンドでは、HLD-001から返却された保有商品を表示することを基本責務とする。

保有商品の利用者境界、商品単位で管理する資産口座への限定、利用状態の判定および表示順の決定はバックエンドの責務とし、React側で同じ業務ルールを再実装しない。

### 2.1 型定義

HLD-001のレスポンスに対応するTypeScriptの型を定義する。

概念例：

```typescript
export type HoldingAssetListItem = {
  id: string;
  assetAccountId: string;
  assetAccountName: string;
  name: string;
  startYearMonth: string;
  isEnabled: boolean;
};

export type HoldingAssetListResponse = {
  data: HoldingAssetListItem[];
};
```

APIレスポンスのIDは文字列として扱う。

```text
id
assetAccountId
```

について、フロントエンド側で`number`へ変換しない。

`isEnabled`はAPIが返却する利用状態をそのまま使用し、React側で論理削除状態を推測しない。

### 2.2 API Client

HLD-001を呼び出すAPI Clientを定義する。

概念例：

```typescript
export async function getHoldingAssets():
  Promise<HoldingAssetListResponse> {
  return apiClient.get<HoldingAssetListResponse>(
    '/api/v1/holding-assets',
  );
}
```

HLD-001では以下を使用しない。

```text
Path Parameter
Query Parameter
Request Body
```

`X-User-Id`などのAPI共通ヘッダーは、各API Clientから個別に設定するのではなく、共通HTTP Clientで付与する。

API ClientはHTTP通信およびAPIレスポンス型への変換を担当し、画面表示やQuery Cacheの制御は行わない。

### 2.3 Query Key

保有商品関連のQuery Keyを共通定義する。

概念例：

```typescript
export const holdingAssetKeys = {
  all: ['holdingAssets'] as const,

  list: () =>
    [
      ...holdingAssetKeys.all,
      'list',
    ] as const,

  detail: (
    holdingAssetId: string,
  ) =>
    [
      ...holdingAssetKeys.all,
      'detail',
      holdingAssetId,
    ] as const,
};
```

HLD-001では、

```typescript
holdingAssetKeys.list()
```

を使用する。

Query KeyをPageやComponentへ直接記述せず、保有商品機能の共通定義として管理する。

### 2.4 Query Hook

HLD-001専用のQuery Hookを定義する。

概念例：

```typescript
export function useHoldingAssets() {
  return useQuery({
    queryKey:
      holdingAssetKeys.list(),

    queryFn:
      getHoldingAssets,
  });
}
```

Pageから直接`fetch`や`apiClient.get()`を呼び出さず、HTTP通信およびQuery Cache管理をQuery Hookへ委譲する。

### 2.5 Query Cache

HLD-001の取得結果はTanStack QueryのQuery Cacheへ保持する。

```text
HLD-001
    ↓
HoldingAssetListResponse
    ↓
TanStack Query Cache
    ↓
保有商品一覧画面
```

同一画面への再遷移などでは、TanStack Queryのキャッシュ戦略に従って取得結果を再利用できる。

キャッシュの有効期間などはHLD-001固有に独自設定せず、原則としてReactアーキテクチャの共通方針へ従う。

### 2.6 保有商品一覧Page

保有商品一覧Pageでは、主に以下を担当する。

* HLD-001のQuery Hookを呼び出す
* Loading状態を表示する
* Error状態を表示する
* Empty Stateを表示する
* 保有商品一覧を表示する
* 登録画面への導線を表示する
* 必要に応じて詳細画面への導線を表示する

概念例：

```tsx
export function HoldingAssetListPage() {
  const query =
    useHoldingAssets();

  if (query.isPending) {
    return <LoadingIndicator />;
  }

  if (query.isError) {
    return (
      <HoldingAssetListError
        error={query.error}
      />
    );
  }

  if (query.data.data.length === 0) {
    return (
      <HoldingAssetEmptyState />
    );
  }

  return (
    <HoldingAssetList
      holdingAssets={
        query.data.data
      }
    />
  );
}
```

HTTP通信そのものはPageへ記述しない。

### 2.7 Loading状態

HLD-001の取得中は、一覧の代わりにLoading状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return (
    <LoadingIndicator />
  );
}
```

取得完了前に空配列として扱い、

```text
保有商品がありません
```

などのEmpty Stateを一時的に表示しない。

### 2.8 Error状態

HLD-001の取得に失敗した場合は、API共通エラー型を利用してError状態を表示する。

概念例：

```tsx
if (query.isError) {
  return (
    <HoldingAssetListError
      error={query.error}
    />
  );
}
```

HLD-001で想定されるAPIエラーには、主に以下がある。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INTERNAL_SERVER_ERROR
```

個々のComponentへHTTPステータスやエラーコード判定を分散させず、可能な限りAPI共通のエラー処理へ集約する。

### 2.9 Empty State

HLD-001では、保有商品が存在しない場合もエラーではなく、

```json
{
  "data": []
}
```

が返却される。

そのため、

```typescript
query.data.data.length === 0
```

の場合は正常なEmpty Stateとして扱う。

概念例：

```tsx
if (query.data.data.length === 0) {
  return (
    <HoldingAssetEmptyState />
  );
}
```

Empty Stateでは、例えば以下を表示できる。

```text
保有商品が登録されていません。
```

必要に応じて、HLD-002 保有商品登録画面への導線を表示する。

### 2.10 一覧Component

一覧表示専用Componentでは、取得済みの保有商品をpropsとして受け取る。

概念例：

```typescript
type Props = {
  holdingAssets:
    HoldingAssetListItem[];
};
```

```tsx
export function HoldingAssetList({
  holdingAssets,
}: Props) {
  return (
    <div>
      {holdingAssets.map(
        (holdingAsset) => (
          <HoldingAssetListItem
            key={holdingAsset.id}
            holdingAsset={
              holdingAsset
            }
          />
        ),
      )}
    </div>
  );
}
```

一覧ComponentからHLD-001を直接呼び出さない。

### 2.11 一覧表示項目

保有商品ごとに、主に以下を表示する。

```text
name
assetAccountName
startYearMonth
isEnabled
```

概念例：

```tsx
<HoldingAssetListItem
  name={holdingAsset.name}
  assetAccountName={
    holdingAsset.assetAccountName
  }
  startYearMonth={
    holdingAsset.startYearMonth
  }
  isEnabled={
    holdingAsset.isEnabled
  }
/>
```

`id`および`assetAccountId`は、画面遷移やComponent内部の識別などで使用する。

### 2.12 利用状態表示

保有商品の利用状態は、APIから返却された

```text
isEnabled
```

を使用して表示する。

概念例：

```tsx
{holdingAsset.isEnabled
  ? '利用中'
  : '無効'}
```

表示上、利用中と無効化済みを区別できるようにする。

ただし、React側で

```text
deleted_at
```

などのデータベース内部情報を使用して利用状態を再計算しない。

### 2.13 表示順

HLD-001では、以下の表示順がバックエンドで保証される。

```text
1. 利用中
2. 無効化済み
3. 同一利用状態では商品名昇順
```

そのため、React側では原則としてレスポンス配列をそのまま表示する。

```tsx
query.data.data.map(...)
```

に対して、以下のような再ソートを行わない。

```typescript
// 原則行わない
data.sort(...);
```

APIの表示順とフロントエンド独自の表示順が二重管理されることを防ぐ。

### 2.14 検索・絞り込み・ページネーション

Phase1のHLD-001では、以下を提供しない。

```text
商品名検索
資産口座による絞り込み
利用状態による絞り込み
並び順変更
ページネーション
```

そのため、HLD-001用のReact実装でも、これらを前提としたQuery Parameter管理やPagination状態を持たない。

取得した保有商品を全件表示する。

### 2.15 詳細画面への遷移

一覧からHLD-003 保有商品詳細取得を利用する詳細画面へ遷移する場合は、

```text
holdingAsset.id
```

を使用する。

概念例：

```tsx
<Link
  to={`/holding-assets/${holdingAsset.id}`}
>
  詳細を見る
</Link>
```

詳細画面側ではURLから`holdingAssetId`を取得し、HLD-003を実行する。

一覧データだけで詳細画面の状態を完結させず、詳細画面ではHLD-003から最新の詳細情報を取得する構成を基本とする。

### 2.16 HLD-002登録成功後

HLD-002 保有商品登録が成功した場合、HLD-001の一覧内容が変化する。

そのため、HLD-002成功後はHLD-001のQuery Cacheを無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    holdingAssetKeys.list(),
});
```

```text
HLD-002成功
    ↓
HLD-001 Query invalidate
    ↓
HLD-001再取得
    ↓
最新の保有商品一覧表示
```

### 2.17 HLD-004更新成功後

HLD-004 保有商品更新が成功した場合も、商品名など一覧表示項目が変更される可能性がある。

そのため、HLD-001のQuery Cacheを無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    holdingAssetKeys.list(),
});
```

これにより、更新後の一覧表示へ最新の商品情報を反映する。

### 2.18 HLD-005無効化成功後

HLD-005 保有商品無効化が成功すると、

```text
isEnabled
```

および一覧上の表示位置が変化する。

そのため、HLD-005成功後はHLD-001のQuery Cacheを無効化し、一覧を再取得する。

```text
HLD-005成功
    ↓
HLD-001 Query invalidate
    ↓
HLD-001再取得
    ↓
利用中・無効化済みの最新状態を表示
```

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    holdingAssetKeys.list(),
});
```

フロントエンド側で対象要素の`isEnabled`を書き換えて並び替えるよりも、HLD-001を再取得してバックエンドが保証する最新状態および表示順を利用する構成を基本とする。

### 2.19 Component責務分離

保有商品一覧画面では、以下のように責務を分離する。

```text
Page
    ↓
画面全体の状態制御

Query Hook
    ↓
HLD-001実行
Query Cache管理

API Client
    ↓
HTTP通信

List Component
    ↓
保有商品一覧表示

List Item Component
    ↓
保有商品1件の表示

Type
    ↓
API契約

Query Key
    ↓
Query Cache識別
```

1つのComponentへ以下をすべて直接記述しない。

* HTTP通信
* Query Cache管理
* Loading判定
* Errorコード判定
* 一覧描画
* 画面遷移
* APIレスポンス型定義

### 2.20 概念的なディレクトリ構成

Phase1では、例えば以下のように保有商品機能単位で整理できる。

```text
features/
└── holding-assets/
    ├── api/
    │   └── getHoldingAssets.ts
    ├── components/
    │   ├── HoldingAssetList.tsx
    │   ├── HoldingAssetListItem.tsx
    │   └── HoldingAssetEmptyState.tsx
    ├── hooks/
    │   └── useHoldingAssets.ts
    ├── queries/
    │   └── holdingAssetKeys.ts
    ├── types/
    │   └── holdingAsset.ts
    └── pages/
        └── HoldingAssetListPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

### 2.21 フロントエンドで行わないこと

HLD-001のReact・TypeScript実装では、以下をフロントエンドの責務としない。

* 利用者境界の最終保証
* 他利用者の保有商品除外判定
* 商品単位で管理する資産口座への所属判定
* 無効化済みデータを取得対象へ含めるかの判定
* `deleted_at`からの利用状態判定
* `isEnabled`の再計算
* 利用中・無効化済みの表示順決定
* 商品名昇順の決定
* データベース内部カラムの補完
* トランザクション制御
* 排他制御

フロントエンドは、

```text
HLD-001実行
    ↓
Loading / Error判定
    ↓
レスポンス取得
    ↓
Empty State判定
    ↓
APIが返却した順序で一覧表示
```

という責務を基本とする。

## 3. 関連ドキュメント

- [HLD-001 保有商品一覧取得 API詳細](../../../api/details/holding-assets/hld-001-list.md)
- [HLD-001 保有商品一覧取得 Laravelアーキテクチャ設計](../../laravel/holding-assets/hld-001-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [API一覧](../../../api/api-list.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)
