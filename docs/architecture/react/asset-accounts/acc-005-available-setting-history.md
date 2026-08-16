# ACC-005 利用可能資産設定履歴取得

## 1. 概要

本ドキュメントでは、
ACC-005 利用可能資産設定履歴取得APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

ACC-005では、
操作対象となる利用者に帰属する指定された資産口座について、
過去から現在までの利用可能資産設定履歴を取得する。

フロントエンドでは、主に資産口座詳細画面からACC-005を利用し、
以下の情報を履歴として表示する。

* 利用可能資産設定ID
* 適用開始年月
* 適用終了年月
* 利用可能資産区分

ACC-003 資産口座詳細取得が現在の`isAvailable`を取得するのに対し、
ACC-005では利用可能資産区分の変更履歴全体を取得する。

ACC-005は参照専用APIであるため、
TanStack QueryではQueryとして扱い、
ACC-003とは独立したQuery Keyで管理する。

取得した履歴はAPIで保証された並び順を基本としてそのまま利用し、
年月や`isAvailable`などはAPI上の値を保持したまま、
画面表示時に必要な表示形式へ変換する。

また、履歴の期間重複、期間欠落、継続中設定の件数などの
業務データ整合性はLaravel側で検証する。

フロントエンドでは不整合履歴の補完や修正を行わず、
正常レスポンスとして取得した履歴を表示することに責務を限定する。

ACC-006 利用可能資産設定登録の成功後は、
ACC-005の履歴QueryおよびACC-003の詳細Queryを無効化し、
サーバー側の最新状態を再取得することを基本とする。

API通信、共通エラーハンドリング、Query Key、Retry、
TanStack Queryの利用方針などの共通事項については、
Reactアーキテクチャ共通設計に従うものとし、
本ドキュメントではACC-005固有の利用方法のみを定義する。

---

## 2. React・TypeScriptでの利用

ACC-005は、指定した資産口座の利用可能資産設定履歴を画面へ表示するために使用する。

主な利用フローは、以下とする。

```text
ACC-003
資産口座詳細取得
    ↓
資産口座詳細画面
    ↓
利用可能資産設定履歴表示
    ↓
ACC-005
利用可能資産設定履歴取得
```

また、利用可能資産区分を変更する画面では、ACC-006実行前後の履歴確認にも使用できる。

ACC-005は参照専用APIであるため、TanStack Queryを使用する場合はQueryとして扱う。

---

### 2.1 TypeScript型

利用可能資産設定履歴1件は、以下のような型として扱う。

概念例：

```typescript
export type AssetAccountAvailableSettingHistory = {
  id: string;
  startYearMonth: string;
  endYearMonth: string | null;
  isAvailable: boolean;
};
```

ACC-005の正常レスポンスは、履歴配列となる。

```typescript
export type AssetAccountAvailableSettingHistoryResponse =
  ApiResponse<
    AssetAccountAvailableSettingHistory[]
  >;
```

---

### 2.2 id

`id`は、利用可能資産設定IDとして`string`で扱う。

概念例：

```typescript
type AssetAccountAvailableSettingHistory = {
  id: string;
};
```

DB上の`bigint`をReact側で数値型へ変換しない。

---

### 2.3 startYearMonth

`startYearMonth`は、

```text
YYYY-MM
```

形式の文字列として扱う。

概念例：

```typescript
type AssetAccountAvailableSettingHistory = {
  startYearMonth: string;
};
```

表示時には、必要に応じて

```text
2026-08
    ↓
2026年8月
```

へ変換してよい。

---

### 2.4 endYearMonth

`endYearMonth`は、

```text
string | null
```

として扱う。

概念例：

```typescript
type AssetAccountAvailableSettingHistory = {
  endYearMonth: string | null;
};
```

終了年月が存在しない場合は、

```text
null
```

となる。

空文字や特別なダミー年月へ変換しない。

---

### 2.5 isAvailable

`isAvailable`は、booleanとして扱う。

概念例：

```typescript
type AssetAccountAvailableSettingHistory = {
  isAvailable: boolean;
};
```

画面表示時は、必要に応じて

```text
true
    → 利用可能

false
    → 利用対象外
```

などの表示文言へ変換する。

---

### 2.6 isCurrentを型へ追加しない

Phase1では、APIから

```text
isCurrent
```

は返却されない。

そのため、APIレスポンス型へ

```typescript
isCurrent: boolean;
```

を追加しない。

現在継続中の設定は、

```typescript
endYearMonth === null
```

から判定できる。

---

### 2.7 assetAccountId

ACC-005では、対象となる資産口座IDをAPI Client関数の引数として渡す。

概念例：

```typescript
getAssetAccountAvailableSettings(
  assetAccountId,
);
```

`assetAccountId`は、API契約に合わせて`string`として扱う。

---

### 2.8 userIdを関数引数へ含めない

ACC-005専用API Clientへ、

```typescript
getAssetAccountAvailableSettings(
  userId,
  assetAccountId,
);
```

のように`userId`を渡さない。

利用者IDは、共通API Clientから

```text
X-User-Id
```

として付与する。

---

### 2.9 X-User-Id

`X-User-Id`は、ACC-005専用処理ではなく、共通API Clientから付与する。

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

各画面やHookから直接ヘッダーを組み立てない。

---

### 2.10 API Client

ACC-005を呼び出す専用関数を定義する。

概念例：

```typescript
export const getAssetAccountAvailableSettings =
  async (
    assetAccountId: string,
  ): Promise<
    AssetAccountAvailableSettingHistory[]
  > => {
    const response =
      await apiClient.get<
        AssetAccountAvailableSettingHistoryResponse
      >(
        `/api/v1/asset-accounts/${assetAccountId}/available-settings`,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

### 2.11 Queryとして扱う

ACC-005は参照専用GET APIであるため、TanStack QueryではQueryとして扱う。

概念例：

```typescript
export const useAssetAccountAvailableSettings =
  (
    assetAccountId: string,
  ) =>
    useQuery({
      queryKey:
        assetAccountKeys
          .availableSettings(
            assetAccountId,
          ),

      queryFn: () =>
        getAssetAccountAvailableSettings(
          assetAccountId,
        ),
    });
```

---

### 2.12 Query Key

ACC-005のQuery Keyは、資産口座詳細とは別キーにする。

概念的には、

```text
assetAccounts
+
assetAccountId
+
availableSettings
```

とする。

例えば、

```typescript
[
  'assetAccounts',
  assetAccountId,
  'availableSettings',
]
```

とする。

---

### 2.13 Query Keyの共通化

Query Keyは、機能内で共通管理してよい。

概念例：

```typescript
export const assetAccountKeys = {
  all: [
    'assetAccounts',
  ] as const,

  detail: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
    ] as const,

  availableSettings: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
      'availableSettings',
    ] as const,
};
```

ACC-003とACC-005のQuery Cacheを同じキーへ混在させない。

---

### 2.14 assetAccountIdが存在しない場合

画面初期化時などで`assetAccountId`が取得できていない場合は、ACC-005を実行しない。

概念例：

```typescript
useQuery({
  queryKey:
    assetAccountKeys
      .availableSettings(
        assetAccountId,
      ),

  queryFn: () =>
    getAssetAccountAvailableSettings(
      assetAccountId,
    ),

  enabled:
    assetAccountId !== '',
});
```

---

### 2.15 ルートパラメータから取得する場合

資産口座詳細画面では、React Routerなどから`assetAccountId`を取得してよい。

概念例：

```typescript
const {
  assetAccountId,
} = useParams<{
  assetAccountId: string;
}>();
```

値が存在することを確認してからACC-005を実行する。

---

### 2.16 ACC-003と並行取得してよい

資産口座詳細画面では、

```text
ACC-003
資産口座詳細

ACC-005
利用可能資産設定履歴
```

をそれぞれ独立したQueryとして並行取得してよい。

概念的には、

```text
AssetAccountDetailPage
    ├─ useAssetAccountDetail()
    └─ useAssetAccountAvailableSettings()
```

とする。

---

### 2.17 ACC-003の取得成功を必須条件にしなくてよい

ACC-005自身でも資産口座の利用者境界と存在確認を行う。

そのため、技術的には

```text
ACC-003成功
    ↓
ACC-005実行
```

と逐次実行する必要はない。

同じ`assetAccountId`が確定していれば、並行して取得してよい。

ただし、画面構成上必要であればACC-003成功後に表示する方式でもよい。

---

### 2.18 ローディング表示

ACC-005取得中は、履歴領域へローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return (
    <AvailableSettingHistoryLoading />
  );
}
```

履歴取得中に空配列を正常状態として表示しない。

---

### 2.19 履歴一覧表示

正常取得後は、返却された配列を使用して履歴一覧を表示する。

概念例：

```typescript
const {
  data: settings,
} =
  useAssetAccountAvailableSettings(
    assetAccountId,
  );
```

```tsx
<ul>
  {settings?.map(
    (setting) => (
      <li key={setting.id}>
        ...
      </li>
    ),
  )}
</ul>
```

---

### 2.20 APIの並び順を利用する

ACC-005では、API側で

```text
startYearMonth DESC
```

として返却する。

そのため、通常画面では受け取った順序をそのまま表示してよい。

例えば、

```text
2027-01 ～ 現在
2026-07 ～ 2026-12
2026-01 ～ 2026-06
```

のように、最新履歴から表示する。

---

### 2.21 不要な再ソートを行わない

以下のように、毎回React側で正式な履歴順を再構築する必要はない。

```typescript
settings.sort(...);
```

API契約上の並び順を使用する。

画面要件として古い順表示が必要な場合のみ、表示用にコピーして並び替えてよい。

---

### 2.22 現在設定の表示

継続中設定は、

```typescript
setting.endYearMonth === null
```

で判定できる。

概念例：

```typescript
const isCurrent =
  setting.endYearMonth === null;
```

これは画面表示用の派生値であり、API型へ保存し直す必要はない。

---

### 2.23 期間表示

期間は、例えば以下のように表示できる。

```text
2026年1月 ～ 2026年6月
```

継続中の場合は、

```text
2027年1月 ～ 現在
```

のように表示してよい。

概念例：

```typescript
const formatPeriod = (
  startYearMonth: string,
  endYearMonth: string | null,
): string => {
  const start =
    formatYearMonth(
      startYearMonth,
    );

  if (
    endYearMonth === null
  ) {
    return `${start} ～ 現在`;
  }

  return `${start} ～ ${formatYearMonth(
    endYearMonth,
  )}`;
};
```

---

### 2.24 年月表示関数を共通化する

`YYYY-MM`を

```text
2026年8月
```

へ変換する処理は、各コンポーネントへ重複記述せず、共通Utilityへ分離してよい。

概念例：

```typescript
export const formatYearMonth =
  (
    yearMonth: string,
  ): string => {
    const [
      year,
      month,
    ] = yearMonth.split('-');

    return `${year}年${Number(
      month,
    )}月`;
  };
```

正式な日付Utilityは、React共通設計に従う。

---

### 2.25 isAvailableの表示

`isAvailable`は、表示用ラベルへ変換する。

概念例：

```typescript
export const getAvailableLabel =
  (
    isAvailable: boolean,
  ): string =>
    isAvailable
      ? '利用可能'
      : '利用対象外';
```

または、

```tsx
<span>
  {setting.isAvailable
    ? '利用可能'
    : '利用対象外'}
</span>
```

とする。

---

### 2.26 API値を表示文言へ置き換えてState保存しない

以下のように、取得直後に

```text
true
    ↓
利用可能
```

へ変換してAPIデータ自体を書き換えない。

型としてはbooleanを維持し、表示時にだけ文言へ変換する。

---

### 2.27 履歴0件UIを正常系として用意しない

ACC-005の契約では、利用中資産口座に利用可能資産設定履歴が0件となることはデータ不整合である。

そのため、通常状態として

```text
履歴はありません
```

を表示する空一覧UIを必須とはしない。

履歴0件は、サーバーエラーとして扱う。

---

### 2.28 ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND

以下が返却された場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

サーバー側の業務データ不整合として扱う。

フロントエンドで空配列へ変換して履歴なし表示にはしない。

例えば、

```text
利用可能資産設定履歴を
取得できませんでした。
```

などの一般的な取得失敗として表示する。

---

### 2.29 ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID

以下が返却された場合も、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

サーバー側データ不整合として扱う。

フロントエンドで、

- 期間重複を修正する
- 欠落期間を補完する
- 最新設定を推測する
- 複数の継続設定から1件を選ぶ

などの処理は行わない。

---

### 2.30 不整合理由を推測しない

APIからは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

までしか返却しない。

そのため、フロントエンドで

```text
期間重複です
期間が欠落しています
最新設定がありません
```

などと不整合理由を推測して表示しない。

一般利用者向けには共通エラー表示とする。

---

### 2.31 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

となる。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

フロントエンドでは、理由を推測せず、

```text
指定された資産口座が
見つかりません。
```

などの共通表示とする。

---

### 2.32 他利用者かどうかを推測しない

`ASSET_ACCOUNT_NOT_FOUND`を受けても、

```text
他の利用者の資産口座です
```

などと表示しない。

API契約上、存在しない場合との区別はできない。

---

### 2.33 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-005専用画面で独自処理を実装しない。

---

### 2.34 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、利用者コンテキストに関する共通エラーとして扱う。

---

### 2.35 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択されている利用者が有効ではない状態として共通処理する。

---

### 2.36 INVALID_ASSET_ACCOUNT_ID

通常の画面遷移では、ACC-001やACC-003から取得した有効な資産口座IDを使用するため、発生頻度は低い。

URL手入力などで発生した場合は、不正な画面状態として扱い、資産口座一覧へ戻す導線を表示してよい。

---

### 2.37 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
利用可能資産設定履歴を
取得できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

### 2.38 Retry

ACC-005は参照専用GET APIであるため、一時的な通信エラーについてはTanStack QueryのRetry機能を利用してよい。

ただし、

```text
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

のような再実行で改善しにくいエラーに対して無意味なRetryを繰り返さない。

具体的なRetry方針は、React共通設計に従う。

---

### 2.39 ACC-006との連携

利用可能資産区分を変更する場合は、ACC-006をMutationとして実行する。

概念的には、

```text
ACC-005
履歴表示
    ↓
利用者が設定変更
    ↓
ACC-006
新規設定登録
    ↓
成功
    ↓
ACC-005再取得
```

とする。

---

### 2.40 ACC-006成功後のQuery Cache

ACC-006成功後は、ACC-005の履歴内容が変化する。

そのため、対象資産口座の履歴Queryをinvalidateする。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys
      .availableSettings(
        assetAccountId,
      ),
});
```

---

### 2.41 ACC-003のQuery Cacheも無効化する

ACC-006によって現在の利用可能資産区分が変化すると、ACC-003の

```text
isAvailable
```

も変化する。

そのため、ACC-006成功後はACC-003の詳細Queryもinvalidateする。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

---

### 2.42 ACC-006成功レスポンスだけで履歴を再構築しない

技術的には、ACC-006のRequestやResponseからQuery Cacheを手動更新することもできる。

ただし、Phase1では

```text
ACC-006成功
    ↓
ACC-005 invalidate
    ↓
サーバーから最新履歴取得
```

を基本とする。

特に、

```text
旧設定のendYearMonth
新設定ID
新設定startYearMonth
```

などの最終確定状態はサーバーを正とする。

---

### 2.43 Optimistic Update

ACC-005自体は参照Queryである。

ACC-006実行時に、ACC-005の履歴をOptimistic UpdateすることはPhase1では必須としない。

履歴は期間整合性を伴うため、サーバー成功後の再取得を基本とする。

---

### 2.44 ACC-004成功後

ACC-004は、

```text
name
assetType
```

を更新するが、

```text
asset_account_available_settings
```

は変更しない。

そのため、ACC-004成功だけを理由としてACC-005の履歴Queryを必ずinvalidateする必要はない。

ただし、同一画面上でACC-003の資産口座情報とACC-005の履歴を一体表示している場合は、ACC-003側を再取得する。

---

### 2.45 資産口座無効化後

資産口座が無効化されると、ACC-005の通常取得対象から外れる。

そのため、無効化成功後は対象資産口座の履歴Query Cacheを削除または無効化する。

概念例：

```typescript
queryClient.removeQueries({
  queryKey:
    assetAccountKeys
      .availableSettings(
        assetAccountId,
      ),
});
```

---

### 2.46 利用者切替時

利用者切替が行われた場合は、前利用者の利用可能資産設定履歴を新しい利用者へ誤表示しない。

Query Keyへ利用者IDを含める、または利用者切替時に関連Cacheを無効化する。

---

### 2.47 Query KeyへuserIdを含めてもよい

利用者切替を明示的に考慮する場合は、以下のようなQuery Keyとしてよい。

概念例：

```typescript
export const assetAccountKeys = {
  availableSettings: (
    userId: string,
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      userId,
      assetAccountId,
      'availableSettings',
    ] as const,
};
```

正式なQuery Key設計は、React共通設計に従う。

---

### 2.48 履歴レコード単位のQueryを作らない

ACC-005は履歴一覧を1レスポンスで返却する。

そのため、

```text
availableSettingHistory/{settingId}
```

のような個別QueryをACC-005のためだけに作成する必要はない。

配列全体を1つのQuery Cacheとして扱う。

---

### 2.49 履歴IDを編集用途に使用しない

ACC-005で返却される

```text
id
```

は、履歴レコードの識別用に使用できる。

例えばReactの

```tsx
key={setting.id}
```

に使用してよい。

ただし、Phase1では個別履歴を直接編集・削除するAPIは提供しないため、

```text
settingIdを使って
過去履歴を編集する
```

ようなUIは作らない。

---

### 2.50 過去履歴を編集しない

ACC-005で表示した過去履歴について、直接

```text
startYearMonth
endYearMonth
isAvailable
```

を変更する編集UIは提供しない。

利用可能資産区分の変更は、ACC-006で新しい設定を登録することで行う。

---

### 2.51 現在設定も直接書き換えない

現在継続中の設定についても、履歴レコードそのものを直接PATCHする方式は採用しない。

概念的には、

```text
現在設定
    ↓
直接UPDATE
```

ではなく、

```text
ACC-006
    ↓
旧設定を終了
+
新設定を登録
```

とする。

---

### 2.52 履歴コンポーネントの責務

履歴表示コンポーネントでは、主に以下を担当する。

- 履歴配列の表示
- 年月の表示変換
- `isAvailable`の表示変換
- 継続中設定の表示
- ローディング表示
- 取得エラー表示

利用者境界判定や履歴整合性判定をコンポーネントへ持たせない。

---

### 2.53 Query Hookの責務

Query Hookでは、主に以下を担当する。

```text
ACC-005実行
Query Key管理
Query Cache管理
Retry
```

履歴表示レイアウトや業務文言生成をQuery Hookへ持たせない。

---

### 2.54 API Clientの責務

API Clientでは、

```http
GET /api/v1/asset-accounts/{assetAccountId}/available-settings
```

のHTTP通信と型付きレスポンス取得を担当する。

以下はAPI Clientの責務に含めない。

- Toast表示
- 画面遷移
- 履歴期間表示
- 利用可能/利用対象外の文言変換
- データ不整合補正

---

### 2.55 Pageの責務

資産口座詳細Pageでは、必要に応じて

```text
ACC-003
+
ACC-005
```

を組み合わせる。

概念的には、

```text
AssetAccountDetailPage
    ├─ AssetAccountDetailSection
    │      ↓
    │    ACC-003
    │
    └─ AvailableSettingHistorySection
           ↓
         ACC-005
```

とする。

---

### 2.56 概念的なディレクトリ構成

例えば、以下のように整理できる。

```text
features/
└── asset-accounts/
    ├── api/
    │   ├── getAssetAccountDetail.ts
    │   ├── getAssetAccountAvailableSettings.ts
    │   └── createAssetAccountAvailableSetting.ts
    ├── components/
    │   ├── AssetAccountDetail.tsx
    │   └── AvailableSettingHistory.tsx
    ├── hooks/
    │   ├── useAssetAccountDetail.ts
    │   ├── useAssetAccountAvailableSettings.ts
    │   └── useCreateAssetAccountAvailableSetting.ts
    ├── types/
    │   └── assetAccount.ts
    └── pages/
        └── AssetAccountDetailPage.tsx
```

正式な構成は、Reactアーキテクチャ設計に従う。

---

### 2.57 フロントエンドで履歴整合性を再検証しない

ACC-005では、バックエンド側で履歴整合性を確認する。

そのため、React側で再度、

- 初期開始年月一致
- 期間重複
- 期間欠落
- 継続中設定件数
- 最新設定の終了年月

を業務ルールとして検証しない。

正常レスポンスで返却された履歴は、整合しているものとして扱う。

---

### 2.58 表示上の派生値はReact側で計算してよい

業務整合性判定は行わないが、画面表示に必要な派生値はReact側で計算してよい。

例えば、

```typescript
const isCurrent =
  setting.endYearMonth === null;
```

や、

```typescript
const periodLabel =
  formatPeriod(
    setting.startYearMonth,
    setting.endYearMonth,
  );
```

などである。

---

### 2.59 DBカラム名を使用しない

React側の型では、以下のようなsnake_caseを使用しない。

```typescript
type AvailableSetting = {
  start_year_month: string;
  end_year_month: string | null;
  is_available: boolean;
};
```

API契約に従い、

```typescript
type AvailableSetting = {
  startYearMonth: string;
  endYearMonth: string | null;
  isAvailable: boolean;
};
```

とする。

---

### 2.60 assetAccountIdを各履歴へ追加しない

APIからは各履歴へ

```text
assetAccountId
```

を返却しない。

React側でも、APIレスポンス変換時に不要に各履歴へ資産口座IDを複製する必要はない。

必要な場合は、画面コンテキスト側で親の`assetAccountId`を保持する。

---

### 2.61 現在年月を使って履歴をフィルタしない

ACC-005は全履歴取得APIである。

React側で、

```text
現在年月より前だけ
現在設定だけ
```

などへ自動的に絞り込まない。

履歴一覧として全件表示することを基本とする。

画面要件として折りたたみ等を行う場合でも、取得データ自体を欠落させない。

---

### 2.62 フロントエンドで行わないこと

ACC-005のReact・TypeScript実装では、以下をフロントエンドの責務としない。

- 利用者境界の最終保証
- 資産口座存在確認
- 論理削除判定
- 履歴0件の補完
- 初期開始年月整合性判定
- 期間重複判定
- 期間欠落判定
- 継続中設定件数判定
- 不整合履歴の補正
- 現在設定のDB上の確定
- 過去履歴の直接編集
- DB内部項目の解釈

フロントエンドは、

```text
assetAccountId
    ↓
ACC-005
    ↓
正常な履歴一覧取得
    ↓
表示用形式へ変換
    ↓
画面表示
```

という責務を基本とする。

---

## 3. 関連ドキュメント

- [ACC-005 API詳細設計](../../../api/details/asset-accounts/acc-005-available-setting-history.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../../laravel/laravel-architecture.md)
- [資産口座 Laravelアーキテクチャ設計](../../laravel/asset-accounts/README.md)
- [ACC-005 Laravelアーキテクチャ設計](../../laravel/asset-accounts/acc-005-available-setting-history.md)
- [Reactアーキテクチャ共通設計](../react-architecture.md)
- [資産口座 Reactアーキテクチャ設計](./README.md)
- [ACC-005 テスト設計](../../../tests/asset-accounts/acc-005-available-setting-history.md)
- [資産口座 テスト設計](../../../tests/asset-accounts/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)
