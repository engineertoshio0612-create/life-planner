# VAL-001 商品別月末評価額一覧取得

## 1. 概要

操作対象となる利用者について、
指定した月末資産状況に紐づく
商品別月末評価額の一覧を取得・表示する。

VAL-001は参照専用のGET APIとして扱い、
月末資産状況詳細画面などから
URL等で取得した `snapshotId` を使用して実行する。

React・TypeScript側では、
TanStack QueryのQueryとして商品別月末評価額を取得し、
月末資産状況ごとにQuery Cacheを管理する。

取得した商品別月末評価額について、
主に以下の情報を一覧表示する。

- 保有商品名
- 商品種別
- 月末評価額

商品別月末評価額が0件の場合は、
エラーではなく正常なEmpty Stateとして扱う。

また、
未登録の商品別月末評価額を
React側で0円として補完せず、
APIから取得した保存済みの評価額のみを表示する。

月末資産状況詳細画面では、
SNP-003 月末資産状況詳細取得と併用し、
月末資産状況の情報と
商品別月末評価額をそれぞれ取得する。

`confirmed` はVAL-001の取得可否には使用せず、
商品別月末評価額の登録・更新UIなど、
編集可否の制御に使用する。

HTTP通信はAPI Client、
サーバー状態およびCache管理はQuery Hook、
Loading・Error・Empty Stateや一覧配置はPage・Componentが担当し、
各責務を分離する。

---

## 2. React・TypeScriptでの利用

VAL-001は、月末資産状況詳細画面などで、指定された月末資産状況に属する商品別月末評価額一覧を取得・表示する際に使用する。

VAL-001は参照専用GET APIであるため、TanStack Queryを使用する場合はQueryとして扱う。

概念的な利用フローは、以下とする。

```text
月末資産状況詳細画面
    ↓
URL等からsnapshotId取得
    ↓
SNP-003
月末資産状況詳細取得
    ↓
VAL-001
商品別月末評価額一覧取得
    ↓
商品別月末評価額一覧表示
```

### 2.1 TypeScript型

VAL-001では、Request Bodyを使用しない。

API呼び出しに必要な値は、

```text
snapshotId
```

のみとする。

概念例：

```ts
export type GetMonthEndHoldingValuesVariables = {
  snapshotId: string;
};
```

### 2.2 レスポンス型

商品別月末評価額1件を、以下のような型として定義する。

概念例：

```ts
export type MonthEndHoldingValue = {
  id: string;
  holdingAssetId: string;
  holdingAssetName: string;
  assetType: AssetType;
  value: number;
};
```

`assetType`は、HLD系APIと同じ共通型を使用する。

### 2.3 assetTypeの共通型

商品種別は、VAL-001専用の型を新たに定義しない。

例えば、HLD系APIで以下の型を使用している場合は、

```ts
export type AssetType =
  | 'INVESTMENT_TRUST'
  | 'STOCK'
  | 'BOND';
```

VAL-001でも同じ`AssetType`を使用する。

実際の値は、API共通定義および保有商品設計を正とする。

### 2.4 正常レスポンス型

API共通Envelopeを使用する場合は、以下のように定義する。

概念例：

```ts
export type GetMonthEndHoldingValuesResponse =
  ApiResponse<MonthEndHoldingValue[]>;
```

レスポンス例：

```json
{
  "data": [
    {
      "id": "101",
      "holdingAssetId": "10",
      "holdingAssetName": "eMAXIS Slim 全世界株式",
      "assetType": "INVESTMENT_TRUST",
      "value": 350000
    }
  ]
}
```

### 2.5 IDはstringとして扱う

以下のIDは、React・TypeScript側ではstringとして扱う。

```text
snapshotId
id
holdingAssetId
```

DB上で`bigint`であっても、フロントエンドで`number`へ変換しない。

### 2.6 valueはnumberとして扱う

商品別月末評価額は、

```ts
value: number;
```

として扱う。

日本円整数であるため、通常の金額表示では小数処理を行わない。

### 2.7 API Client

VAL-001を呼び出す専用API Client関数を定義する。

概念例：

```ts
export const getMonthEndHoldingValues =
  async (
    snapshotId: string,
  ): Promise<MonthEndHoldingValue[]> => {
    const response =
      await apiClient.get<
        GetMonthEndHoldingValuesResponse
      >(
        `/api/v1/month-end-asset-snapshots/${snapshotId}/holding-values`,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

### 2.8 userIdをAPI Client引数へ含めない

以下のようなAPI Clientにはしない。

```ts
getMonthEndHoldingValues(
  userId,
  snapshotId,
);
```

利用者IDは、共通API Clientから

```text
X-User-Id
```

として付与する。

VAL-001固有の引数は、

```text
snapshotId
```

だけとする。

### 2.9 X-User-Id

`X-User-Id`は、VAL-001専用処理ではなく、共通API Clientから付与する。

概念例：

```ts
apiClient.interceptors.request.use(
  (config) => {
    config.headers['X-User-Id'] =
      currentUserId;

    return config;
  },
);
```

各Page、Component、Query Hookから直接`X-User-Id`を設定しない。

### 2.10 Request Bodyを送信しない

VAL-001はGET APIであるため、Request Bodyを送信しない。

以下のような呼び出しにはしない。

```ts
apiClient.get(
  '/api/v1/month-end-asset-snapshots/20/holding-values',
  {
    data: {
      snapshotId: '20',
    },
  },
);
```

`snapshotId`はURLへ含める。

### 2.11 Queryとして扱う

VAL-001はサーバー状態を変更しないため、TanStack QueryではQueryとして扱う。

概念例：

```ts
export const useMonthEndHoldingValues =
  (
    snapshotId: string,
  ) => {
    return useQuery({
      queryKey:
        monthEndHoldingValueKeys.list(
          snapshotId,
        ),

      queryFn: () =>
        getMonthEndHoldingValues(
          snapshotId,
        ),
    });
  };
```

Mutationとして実装しない。

### 2.12 Query Key

商品別月末評価額のQuery Keyは、共通定義として管理する。

概念例：

```ts
export const monthEndHoldingValueKeys = {
  all: [
    'monthEndHoldingValues',
  ] as const,

  lists: () =>
    [
      ...monthEndHoldingValueKeys.all,
      'list',
    ] as const,

  list: (
    snapshotId: string,
  ) =>
    [
      ...monthEndHoldingValueKeys.lists(),
      snapshotId,
    ] as const,
};
```

これにより、月末資産状況ごとの商品別月末評価額を別Cacheとして管理する。

### 2.13 snapshotIdをQuery Keyへ含める

以下のようなQuery Keyにはしない。

```ts
[
  'monthEndHoldingValues',
]
```

VAL-001の結果は、`snapshotId`によって異なる。

そのため、

```ts
[
  'monthEndHoldingValues',
  'list',
  snapshotId,
]
```

のように、`snapshotId`をQuery Keyへ含める。

### 2.14 利用者切替を考慮する

Phase1では、`X-User-Id`によって操作対象利用者を切り替える。

そのため、利用者切替時に前利用者の商品別月末評価額を誤表示しないようにする。

正式な方式は、React共通設計に従う。

### 2.15 Query KeyへuserIdを含めてもよい

利用者ごとのCache境界を明示する場合は、Query Keyへ`userId`を含めてもよい。

概念例：

```ts
export const monthEndHoldingValueKeys = {
  list: (
    userId: string,
    snapshotId: string,
  ) =>
    [
      'monthEndHoldingValues',
      userId,
      'list',
      snapshotId,
    ] as const,
};
```

ただし、`userId`をVAL-001のAPI Client引数へ渡すという意味ではない。

`userId`はCache管理上だけ使用し、HTTP Requestでは共通API Clientが`X-User-Id`として付与する。

### 2.16 enabledによるQuery制御

`snapshotId`が取得できていない状態では、VAL-001を実行しない。

概念例：

```ts
return useQuery({
  queryKey:
    monthEndHoldingValueKeys.list(
      snapshotId,
    ),

  queryFn: () =>
    getMonthEndHoldingValues(
      snapshotId,
    ),

  enabled:
    snapshotId.length > 0,
});
```

不完全なURL情報でAPI Requestを送信しない。

### 2.17 SNP-003との併用

月末資産状況詳細画面では、SNP-003とVAL-001を併用してよい。

概念的には、

```text
snapshotId
    ├─ SNP-003
    │    ↓
    │  targetYearMonth
    │  confirmed
    │
    └─ VAL-001
         ↓
       商品別月末評価額一覧
```

とする。

VAL-001では`targetYearMonth`や`confirmed`を重複取得する必要はない。

### 2.18 SNP-003の成功を必須条件としなくてもよい

SNP-003とVAL-001が互いに独立して取得可能であれば、必ずしも

```text
SNP-003成功
    ↓
VAL-001実行
```

という直列処理にしなくてよい。

`snapshotId`が確定している場合は、並列で取得してよい。

概念的には、

```text
snapshotId
    ├─────────────┐
    ↓             ↓
SNP-003        VAL-001
    ↓             ↓
月情報          商品別評価額
    └──────┬──────┘
           ↓
        画面表示
```

不要なウォーターフォールを発生させない。

### 2.19 0件を正常状態として扱う

VAL-001が

```json
{
  "data": []
}
```

を返した場合は、エラー表示にしない。

例えば、

```text
商品別月末評価額は
まだ登録されていません。
```

などのEmpty Stateを表示する。

正式な文言は、画面設計に従う。

### 2.20 0件と通信エラーを区別する

以下は異なる状態として扱う。

```text
data = []
    → 正常
    → 登録済み評価額0件

API Error
    → 異常
    → 一覧取得失敗
```

例えば、

```tsx
if (query.isError) {
  return (
    <ErrorMessage />
  );
}

if (
  query.data?.length === 0
) {
  return (
    <EmptyState />
  );
}
```

のように表示を分ける。

### 2.21 ローディング状態

VAL-001取得中は、ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return (
    <LoadingIndicator />
  );
}
```

SNP-003とVAL-001を並列取得する場合は、画面全体を一律にブロックするか、各セクション単位でローディング表示するかを画面設計で決定する。

### 2.22 一覧表示

取得したデータは、商品別月末評価額一覧として表示する。

概念例：

```tsx
{holdingValues.map(
  (holdingValue) => (
    <MonthEndHoldingValueRow
      key={holdingValue.id}
      holdingValue={
        holdingValue
      }
    />
  ),
)}
```

### 2.23 keyにはidを使用する

Reactの一覧描画では、

```text
holdingValue.id
```

を`key`として使用する。

配列indexを`key`として使用しない。

概念例：

```tsx
<MonthEndHoldingValueRow
  key={holdingValue.id}
  holdingValue={holdingValue}
/>
```

### 2.24 金額表示

`value`は日本円整数として受け取る。

表示時は、共通の金額Formatterを使用する。

概念例：

```ts
export const formatYen = (
  value: number,
): string =>
  new Intl.NumberFormat(
    'ja-JP',
    {
      style: 'currency',
      currency: 'JPY',
    },
  ).format(value);
```

例えば、

```text
350000
```

を、

```text
￥350,000
```

などとして表示する。

正式な表記は、画面共通方針に従う。

### 2.25 コンポーネント内で金額計算をしない

VAL-001で取得した

```text
value
```

は、保存済みの商品別月末評価額である。

コンポーネント内で、

```text
数量
×
現在価格
```

などによって再計算しない。

APIから取得した保存済み評価額を表示する。

### 2.26 holdingAssetName

商品名表示には、

```ts
holdingValue.holdingAssetName
```

を使用する。

商品名を表示するためだけにHLD-003などを商品件数分追加実行しない。

VAL-001レスポンスに含まれる保有商品名を使用する。

### 2.27 assetType

商品種別表示には、

```ts
holdingValue.assetType
```

を使用する。

表示ラベルへの変換は、HLD系画面と同じ共通変換処理を使用する。

概念例：

```ts
export const assetTypeLabels:
  Record<AssetType, string> = {
    INVESTMENT_TRUST:
      '投資信託',

    STOCK:
      '株式',

    BOND:
      '債券',
  };
```

実際のEnum値・表示名は、共通定義を正とする。

### 2.28 APIの並び順を基本とする

VAL-001では、API側で固定の並び順を適用する。

そのため、画面固有の要件がなければReact側で再ソートしない。

```ts
query.data?.sort(...)
```

を各コンポーネントで個別実装しない。

### 2.29 クライアント側で未登録商品を補完しない

VAL-001に含まれていない保有商品について、React側で勝手に

```ts
{
  holdingAssetId: '10',
  value: 0,
}
```

のような疑似データを生成しない。

以下は別状態である。

```text
商品別月末評価額未登録

≠

商品別月末評価額0円
```

### 2.30 未登録状況の表示が必要な場合

画面要件として、

```text
どの商品が未登録なのか
```

まで表示する必要がある場合は、VAL-001だけでは判定できない可能性がある。

その場合は、

```text
保有商品情報
+
VAL-001
```

を組み合わせるか、未登録状況を返却する別API設計を検討する。

VAL-001のレスポンスだけから存在しない評価額を0円として推測しない。

### 2.31 確定状態によってVAL-001を停止しない

SNP-003で

```text
confirmed = true
```

を取得した場合でも、VAL-001は実行可能とする。

確定済み月末資産状況でも、商品別月末評価額を参照できるためである。

### 2.32 確定状態は編集UIの制御に使用する

`confirmed`は、VAL-001の取得可否ではなく、商品別月末評価額の登録・更新UIを表示するかどうかの判断に使用してよい。

概念的には、

```text
confirmed = false
    ↓
評価額編集UIを表示可能

confirmed = true
    ↓
参照のみ
```

とする。

ただし、更新可否の最終保証はバックエンド側で行う。

### 2.33 VAL登録・更新後のCache無効化

商品別月末評価額の登録または更新に成功した場合は、対象`snapshotId`のVAL-001 Query Cacheを無効化する。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey:
    monthEndHoldingValueKeys.list(
      snapshotId,
    ),
});
```

### 2.34 登録後は再取得を基本とする

商品別月末評価額登録後に、React側で一覧Cacheへ手動追加することも可能である。

ただし、Phase1では実装を単純化するため、

```text
登録成功
    ↓
VAL-001 invalidate
    ↓
VAL-001再取得
```

を基本とする。

### 2.35 更新後も再取得を基本とする

商品別月末評価額更新後も、

```text
更新成功
    ↓
VAL-001 invalidate
    ↓
VAL-001再取得
```

を基本とする。

一覧Cacheを複雑に手動編集しない。

### 2.36 SNP-004確定成功後

SNP-004成功によって商品別月末評価額自体が変更されない場合は、VAL-001のCacheを必ずしも無効化する必要はない。

ただし、月末資産状況詳細画面全体を最新状態へ同期する方針であれば、関連Queryをまとめてinvalidateしてもよい。

### 2.37 SNP-005確定解除成功後

SNP-005成功時も、商品別月末評価額自体が変更されない場合は、VAL-001の再取得を必須とはしない。

確定状態はSNP系Queryを更新する。

### 2.38 INVALID_SNAPSHOT_ID

以下のエラーを受信した場合は、

```text
INVALID_SNAPSHOT_ID
```

不正なURLまたは不正な画面状態として扱う。

通常画面では、一覧画面などへ戻る導線を表示してよい。

### 2.39 MONTH_END_ASSET_SNAPSHOT_NOT_FOUND

以下の場合は、

```text
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND
```

を受信する。

- 月末資産状況不存在
- 他利用者所属
- 論理削除済み

React側では、その理由を推測しない。

例えば、

```text
指定された月末資産状況が
見つかりません。
```

などの共通表示とする。

### 2.40 他利用者所属を推測しない

`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`を受信しても、

```text
他の利用者の
月末資産状況です。
```

などと表示しない。

API契約上、不存在との区別はできない。

### 2.41 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

VAL-001専用のエラー処理を各Componentへ実装しない。

### 2.42 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、利用者コンテキストに関する共通エラーとして扱う。

### 2.43 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択されている利用者が有効ではない状態として共通処理する。

### 2.44 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
商品別月末評価額を
取得できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

### 2.45 エラーコードで分岐する

フロントエンドでは、`message`文字列ではなく、

```text
error.code
```

を基準としてエラー処理を分岐する。

概念例：

```ts
switch (error.code) {
  case 'INVALID_SNAPSHOT_ID':
    // 不正なID
    break;

  case 'MONTH_END_ASSET_SNAPSHOT_NOT_FOUND':
    // 対象不存在
    break;

  default:
    // 共通エラー
    break;
}
```

### 2.46 GETのRetry

VAL-001は副作用を持たないGET APIであるため、一時的な通信エラーに対する限定的なRetryを許可してよい。

概念例：

```ts
useQuery({
  queryKey:
    monthEndHoldingValueKeys.list(
      snapshotId,
    ),

  queryFn: () =>
    getMonthEndHoldingValues(
      snapshotId,
    ),

  retry: 1,
});
```

ただし、404等の業務上確定したエラーについて無意味なRetryを行わないよう、React共通方針に従う。

### 2.47 Pageの責務

月末資産状況詳細Pageでは、主に以下を担当する。

- URLから`snapshotId`を取得する
- SNP-003を利用して月末資産状況を取得する
- VAL-001を利用して商品別月末評価額を取得する
- Loading / Error / Empty Stateを制御する
- 商品別月末評価額一覧を配置する
- 確定状態に応じて編集UIを制御する

HTTP通信処理そのものは、API ClientやQuery Hookへ委譲する。

### 2.48 一覧Componentの責務

商品別月末評価額一覧Componentでは、主に以下を担当する。

- 商品別月末評価額一覧表示
- 商品名表示
- 商品種別表示
- 評価額表示
- 0件時のEmpty State表示

以下は行わない。

- API通信
- 利用者境界判定
- 未登録評価額生成
- 評価額再計算
- 確定可否判定

### 2.49 Row Componentの責務

1件分の表示Componentでは、例えば、

```ts
type Props = {
  holdingValue:
    MonthEndHoldingValue;
};
```

を受け取り、

- 保有商品名
- 商品種別
- 月末評価額

などを表示する。

DB構造やAPI通信方式をRow Componentへ持ち込まない。

### 2.50 Query Hookの責務

Query Hookでは、主に以下を担当する。

- VAL-001実行
- Query Key管理
- Loading状態管理
- Error状態管理
- Cache管理

画面固有の表示文言やレイアウトはQuery Hookへ持たせない。

### 2.51 API Clientの責務

API Clientでは、

```text
GET
/api/v1/month-end-asset-snapshots/{snapshotId}/holding-values
```

のHTTP通信と、型付きレスポンス取得を担当する。

以下をAPI Clientへ含めない。

- Toast表示
- 画面遷移
- Loading表示
- Empty State表示
- 金額フォーマット
- Query Cache操作
- 確定状態によるUI判定

### 2.52 概念的なディレクトリ構成

例えば、以下のように整理できる。

```text
features/
└── month-end-assets/
    ├── api/
    │   ├── getMonthEndAssetSnapshot.ts
    │   └── getMonthEndHoldingValues.ts
    ├── components/
    │   ├── MonthEndHoldingValueList.tsx
    │   └── MonthEndHoldingValueRow.tsx
    ├── hooks/
    │   ├── useMonthEndAssetSnapshot.ts
    │   └── useMonthEndHoldingValues.ts
    ├── types/
    │   ├── monthEndAssetSnapshot.ts
    │   └── monthEndHoldingValue.ts
    └── pages/
        └── MonthEndAssetDetailPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

### 2.53 HLD系の共通型を再利用する

VAL-001では、保有商品に関する共通概念について、HLD系で定義済みの型を可能な範囲で再利用する。

例えば、

```text
AssetType
```

を共通化する。

一方、HLD-001のレスポンス型そのものをVAL-001へ流用する必要はない。

APIごとの責務に応じてレスポンス型を定義する。

### 2.54 APIレスポンス型と画面表示型を必要に応じて分離する

Phase1では、VAL-001のレスポンス型をそのまま表示へ使用してよい。

将来的に画面固有の情報が増えた場合は、

```text
API Response
    ↓
View Model
    ↓
Component
```

と分離してよい。

現時点では、不要な変換層を追加しない。

### 2.55 フロントエンドで行わないこと

VAL-001のReact・TypeScript実装では、以下をフロントエンドの責務としない。

- 利用者境界の最終保証
- 月末資産状況存在確認の最終保証
- 論理削除判定
- 商品別月末評価額のDB検索条件決定
- 未登録評価額の自動生成
- 未登録を0円へ変換
- 保存済み評価額の再計算
- 商品別月末評価額の確定可否判定
- 保有商品の現在状態による過去データ除外
- DB上の並び順保証
- `assetType`の業務ルール判定

フロントエンドは、

```text
snapshotId取得
    ↓
VAL-001実行
    ↓
Loading / Error / Empty判定
    ↓
商品別月末評価額一覧表示
```

という責務を基本とする。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [VAL-002 商品別月末評価額登録](./val-002-create.md)
- [VAL-003 商品別月末評価額更新](./val-003-update.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/holding-values/README.md)
- [Reactアーキテクチャ設計](../../../architecture/react/holding-values/README.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)