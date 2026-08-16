# ACC-003 資産口座詳細取得

## 1. 概要

操作対象となる利用者に帰属する指定された資産口座の詳細情報を取得する。

資産口座の基本情報に加えて、現在の利用可能資産区分を返却する。

主に以下の情報を取得する。

* 資産口座ID
* 資産口座名
* 資産種別
* 残高記録単位
* 利用開始年月
* 現在の利用可能資産区分
* 利用状態

概念的には、以下の情報を組み合わせてレスポンスを生成する。

```text
asset_accounts
    +
現在有効な
asset_account_available_settings
    ↓
資産口座詳細
```

本APIは参照専用であり、資産口座、利用可能資産設定、月末資産データなどの業務データを更新しない。

---

## 2. React・TypeScriptでの利用

ACC-003は、資産口座一覧や資産口座編集画面から、指定した資産口座の詳細情報を取得するために使用する。

主な利用フローは、以下とする。

```text
ACC-001
資産口座一覧取得
    ↓
利用者が資産口座を選択
    ↓
ACC-003
資産口座詳細取得
    ↓
詳細表示
または
編集フォーム初期値へ反映
```

ACC-003は参照専用APIであるため、TanStack Queryを使用する場合はQueryとして扱う。

---

### 2.1 TypeScript型

ACC-003の正常レスポンスデータは、以下のような型として扱う。

概念例：

```ts
export type AssetAccountDetail = {
  id: string;
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  isAvailable: boolean;
  startYearMonth: string;
  isEnabled: boolean;
};
```

API共通Envelopeを使用する場合は、以下のように定義する。

```ts
export type AssetAccountDetailResponse =
  ApiResponse<AssetAccountDetail>;
```

---

### 2.2 AssetType

`assetType`は、ACC-002などと共通の型を使用する。

概念例：

```ts
export type AssetType =
  | 'CASH'
  | 'BANK'
  | 'SECURITIES'
  | 'IDECO'
  | 'CORPORATE_DC'
  | 'OTHER';
```

ACC-003専用として同じUnion型を重複定義しない。

---

### 2.3 BalanceRecordingUnit

`balanceRecordingUnit`も、ACC-002などと共通の型を使用する。

概念例：

```ts
export type BalanceRecordingUnit =
  | 'ACCOUNT'
  | 'HOLDING';
```

---

### 2.4 assetAccountId

ACC-003では、取得対象となる`assetAccountId`をAPI Client関数の引数として渡す。

概念例：

```ts
getAssetAccountDetail(
  assetAccountId,
);
```

`assetAccountId`は、API契約に合わせて`string`として扱う。

---

### 2.5 userIdを関数引数へ含めない

ACC-003専用のAPI Client関数へ、

```ts
getAssetAccountDetail(
  userId,
  assetAccountId,
);
```

のように`userId`を渡さない。

利用者IDは、`X-User-Id`として共通API Clientから付与する。

---

### 2.6 X-User-Id

`X-User-Id`は、ACC-003専用処理ではなく、共通API Clientから付与する。

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

ACC-003を呼び出すコンポーネントから利用者ヘッダーを直接組み立てない。

---

### 2.7 API Client

ACC-003を呼び出す専用関数を定義する。

概念例：

```ts
export const getAssetAccountDetail =
  async (
    assetAccountId: string,
  ): Promise<AssetAccountDetail> => {
    const response =
      await apiClient.get<
        AssetAccountDetailResponse
      >(
        `/api/v1/asset-accounts/${assetAccountId}`,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

### 2.8 Queryとして扱う

ACC-003は参照専用APIであるため、TanStack Queryを使用する場合はQueryとして扱う。

概念例：

```ts
export const useAssetAccountDetail =
  (
    assetAccountId: string,
  ) =>
    useQuery({
      queryKey: [
        'assetAccounts',
        assetAccountId,
      ],
      queryFn: () =>
        getAssetAccountDetail(
          assetAccountId,
        ),
    });
```

---

### 2.9 Query Key

資産口座詳細のQuery Keyは、資産口座IDを含める。

概念的には、

```text
assetAccounts
+
assetAccountId
```

とする。

例えば、

```ts
[
  'assetAccounts',
  assetAccountId,
]
```

とする。

---

### 2.10 Query Keyの共通化

Query Keyは、各コンポーネントへ直接記述せず、共通定義として管理してよい。

概念例：

```ts
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
};
```

ACC-003では、

```ts
queryKey:
  assetAccountKeys.detail(
    assetAccountId,
  ),
```

として使用できる。

---

### 2.11 assetAccountIdが存在しない場合

画面初期化時などで`assetAccountId`がまだ取得できていない場合は、Queryを実行しない。

概念例：

```ts
useQuery({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),

  queryFn: () =>
    getAssetAccountDetail(
      assetAccountId,
    ),

  enabled:
    assetAccountId !== '',
});
```

不完全なIDでACC-003を実行しない。

---

### 2.12 ルートパラメータから取得する場合

詳細画面では、React RouterなどのURLパラメータから`assetAccountId`を取得してよい。

概念例：

```ts
const {
  assetAccountId,
} = useParams<{
  assetAccountId: string;
}>();
```

ただし、`assetAccountId`が存在することを確認してからACC-003を実行する。

---

### 2.13 ローディング表示

ACC-003取得中は、ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return (
    <Loading />
  );
}
```

取得前の空データを正常な詳細情報として表示しない。

---

### 2.14 正常取得

正常時は、ACC-003から返却された`AssetAccountDetail`を画面表示へ使用する。

概念例：

```ts
const {
  data,
} = useAssetAccountDetail(
  assetAccountId,
);
```

---

### 2.15 詳細表示

詳細画面では、必要に応じて以下を表示する。

* 資産口座名
* 資産種別
* 残高記録単位
* 利用開始年月
* 現在の利用可能資産区分
* 利用状態

---

### 2.16 assetTypeの表示

APIから返却された`SECURITIES`などを、画面表示では日本語ラベルへ変換する。

概念例：

```ts
export const assetTypeLabels:
  Record<AssetType, string> = {
    CASH: '現金',
    BANK: '銀行',
    SECURITIES: '証券',
    IDECO: 'iDeCo',
    CORPORATE_DC: '企業型DC',
    OTHER: 'その他',
  };
```

API値自体を日本語へ変換してStateへ保存し直す必要はない。

---

### 2.17 balanceRecordingUnitの表示

概念例：

```ts
export const balanceRecordingUnitLabels:
  Record<
    BalanceRecordingUnit,
    string
  > = {
    ACCOUNT: '口座単位',
    HOLDING: '商品単位',
  };
```

---

### 2.18 isAvailableの表示

`isAvailable`は、現在の利用可能資産区分として表示する。

概念例：

```tsx
<span>
  {assetAccount.isAvailable
    ? '利用可能資産'
    : '利用対象外'}
</span>
```

表示文言は、画面設計で定義した用語に統一する。

---

### 2.19 isEnabledの表示

ACC-003の正常レスポンスでは、`isEnabled = true`となる。

そのため、通常の詳細画面で利用状態を表示する場合は、

```text
利用中
```

として表示できる。

ただし、論理削除済み資産口座はACC-003で取得できないため、ACC-003だけを利用する画面で

```text
無効
```

状態を表示する必要はない。

---

### 2.20 startYearMonthの表示

`startYearMonth`は、

```text
YYYY-MM
```

形式で返却される。

画面表示では、必要に応じて

```text
2026年8月
```

のように表示用フォーマットへ変換してよい。

API値そのものは変更しない。

---

### 2.21 編集画面の初期値

ACC-003は、ACC-004 資産口座更新画面の初期値取得にも使用できる。

概念的には、

```text
ACC-003
    ↓
name
assetType
balanceRecordingUnit
startYearMonth
isAvailable
    ↓
編集フォーム初期値
```

とする。

ただし、ACC-004が実際に更新可能とする項目だけを編集フォームへ反映する。

---

### 2.22 Form Stateへの変換

詳細DTOと編集フォームStateは別型として定義してよい。

概念例：

```ts
export type AssetAccountFormValues = {
  name: string;
  assetType: AssetType;
  balanceRecordingUnit:
    BalanceRecordingUnit;
  startYearMonth: string;
  isAvailable: boolean;
};
```

変換例：

```ts
const initialValues:
  AssetAccountFormValues = {
    name:
      assetAccount.name,

    assetType:
      assetAccount.assetType,

    balanceRecordingUnit:
      assetAccount.balanceRecordingUnit,

    startYearMonth:
      assetAccount.startYearMonth,

    isAvailable:
      assetAccount.isAvailable,
  };
```

---

### 2.23 詳細DTOを直接編集しない

ACC-003で取得した`AssetAccountDetail`を、そのまま編集用Stateとして直接書き換えない。

Query Cache上のデータをフォーム入力によって直接変更しないためである。

編集用Stateへ必要な値をコピーする。

---

### 2.24 ASSET_ACCOUNT_NOT_FOUND

ACC-003で

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

が返却された場合は、対象資産口座を表示できない状態として扱う。

例えば、

```text
指定された資産口座が
見つかりません。
```

と表示する。

---

### 2.25 他利用者かどうかを推測しない

`ASSET_ACCOUNT_NOT_FOUND`には、以下が含まれる。

* 資産口座不存在
* 他利用者の資産口座
* 論理削除済み資産口座

フロントエンドでは、どの理由かを推測して表示を分岐しない。

---

### 2.26 論理削除済みかどうかを推測しない

ACC-003から`ASSET_ACCOUNT_NOT_FOUND`が返却されても、

```text
この資産口座は
無効化されています
```

と断定しない。

API契約上、無効化済みか不存在かは区別されないためである。

---

### 2.27 INVALID_ASSET_ACCOUNT_ID

通常の画面遷移では、サーバーから取得した有効なIDを使用するため、発生頻度は低い。

URL手入力などにより

```text
INVALID_ASSET_ACCOUNT_ID
```

が返却された場合は、不正なURLまたは不正な画面状態として扱う。

例えば、資産口座一覧へ戻す導線を表示してよい。

---

### 2.28 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-003画面だけで独自処理を実装しない。

---

### 2.29 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、API共通の利用者コンテキストエラーとして扱う。

必要に応じて、現在選択している利用者状態を再確認する。

---

### 2.30 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合も、API共通の利用者コンテキストエラーとして扱う。

現在選択されている利用者が有効ではない状態として処理する。

---

### 2.31 利用可能資産設定不存在

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

は、フロントエンド入力によるエラーではなく、サーバー側の業務データ不整合である。

画面では、一般的な取得失敗として扱う。

例えば、

```text
資産口座の詳細を取得できませんでした。
```

と表示する。

内部の

```text
利用可能資産設定が存在しない
```

という詳細を一般利用者向け画面へそのまま表示する必要はない。

---

### 2.32 利用可能資産設定重複

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

についても、サーバー側データ不整合として扱う。

フロントエンドで複数設定から独自に1件を選択して画面表示しない。

---

### 2.33 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
資産口座の詳細を取得できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

### 2.34 Retry

ACC-003は参照専用GET APIであるため、一時的な通信エラーについてTanStack QueryのRetry機能を利用してよい。

ただし、

```text
ASSET_ACCOUNT_NOT_FOUND
```

やデータ不整合系エラーに対して、無意味な再試行を繰り返さない。

正式なRetry方針は、フロントエンド共通設計に従う。

---

### 2.35 Query Cache

ACC-003の結果は、TanStack QueryのQuery Cacheへ保持してよい。

ただし、以下のAPI成功後はキャッシュが古くなる可能性がある。

```text
ACC-004
資産口座更新

ACC-005
資産口座無効化

ACC-006
利用可能資産設定登録
```

そのため、各Mutation成功時に対象詳細Queryをinvalidateする。

---

### 2.36 ACC-004成功後

ACC-004によって資産口座情報が更新された場合は、対象資産口座の詳細Queryを無効化する。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

必要に応じてACC-001一覧Queryも無効化する。

---

### 2.37 ACC-005成功後

ACC-005によって資産口座が無効化された場合は、ACC-003の通常取得対象から外れる。

そのため、対象詳細Queryを無効化または削除する。

概念例：

```ts
queryClient.removeQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

その後、資産口座一覧画面へ遷移してよい。

---

### 2.38 ACC-006成功後

ACC-006によって現在の利用可能資産区分が変更された場合は、ACC-003の`isAvailable`が変化する可能性がある。

そのため、対象詳細Queryをinvalidateする。

---

### 2.39 ACC-002成功直後

ACC-002成功後に登録した資産口座の詳細画面へ遷移する場合は、返却された`data.id`を使用してACC-003を実行する。

ACC-002のレスポンスだけをACC-003の永続的なキャッシュ代替としない。

---

### 2.40 現在年月による変化

ACC-003の`isAvailable`は、現在年月によって変化する可能性がある。

例えば、

```text
2026-08
    isAvailable = true

2026-09
    isAvailable = false
```

となる設定履歴が存在する場合、月が変わればDB更新がなくてもACC-003の結果が変化し得る。

そのため、非常に長い`staleTime`を無条件に設定しない。

具体的なキャッシュ時間は、フロントエンド共通設計で決定する。

---

### 2.41 userId変更時

利用者切替が行われた場合は、前利用者のACC-003結果をそのまま表示し続けない。

Query Cacheを利用者単位で適切に分離または無効化する。

API自体は`X-User-Id`によって利用者境界を保証するが、画面上でも前利用者のキャッシュを誤表示しないようにする。

---

### 2.42 Query Keyと利用者境界

資産口座IDがシステム全体で一意であっても、利用者切替機能があるため、必要に応じてQuery Keyへ利用者IDを含めてもよい。

概念例：

```ts
export const assetAccountKeys = {
  detail: (
    userId: string,
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      userId,
      assetAccountId,
    ] as const,
};
```

ただし、Query Keyの正式な設計はReact共通設計に従う。

---

### 2.43 フロントエンドで現在設定を再計算しない

ACC-003では、バックエンドが現在年月を基準として`isAvailable`を判定する。

フロントエンドでは、利用可能資産設定履歴を取得して独自に

```text
現在有効なのはどの設定か
```

を再計算しない。

ACC-003から返却された`isAvailable`を現在状態として使用する。

---

### 2.44 DB内部コードを扱わない

React側では、以下のようなDB内部コードを扱わない。

```text
asset_type = 3
balance_recording_unit = 2
```

APIから返却された

```text
SECURITIES
HOLDING
```

などの業務上意味のある値を使用する。

---

### 2.45 DBカラム名を型へ持ち込まない

ACC-003の型では、以下のようなsnake_caseを使用しない。

```ts
type AssetAccountDetail = {
  asset_type: string;
  balance_recording_unit: string;
  start_year_month: string;
  is_available: boolean;
};
```

API契約に従って、camelCaseで定義する。

---

### 2.46 関連データをACC-003から期待しない

ACC-003には、以下を含めない。

* 保有商品一覧
* 月末資産残高
* 商品別月末評価額
* 月末資産状況
* 資産推移

画面でこれらが必要な場合は、対応するAPIを別途利用する。

---

### 2.47 HOLDINGの場合

```text
balanceRecordingUnit = HOLDING
```

であっても、ACC-003レスポンスに保有商品一覧は含まれない。

画面で保有商品を表示する場合は、HLD-001を実行する。

概念的には、

```text
ACC-003
    ↓
資産口座詳細

HLD-001
    ↓
保有商品一覧
```

と分離する。

---

### 2.48 ACCOUNTの場合

```text
balanceRecordingUnit = ACCOUNT
```

の場合でも、ACC-003から月末残高を取得しない。

月末資産残高が必要な場合は、対応する月末資産APIを利用する。

---

### 2.49 Page・Query・表示コンポーネントの分離

画面実装では、例えば以下の責務に分離する。

```text
Page
    ↓
assetAccountId取得
エラー時画面遷移

Query Hook
    ↓
ACC-003実行
Query Cache管理

API Client
    ↓
HTTP通信

Detail Component
    ↓
詳細表示

Type
    ↓
API契約
```

1つのコンポーネントへすべての処理を直接記述しない。

---

### 2.50 概念的なディレクトリ構成

例えば、以下のように機能単位で整理できる。

```text
features/
└── asset-accounts/
    ├── api/
    │   └── getAssetAccountDetail.ts
    ├── components/
    │   └── AssetAccountDetail.tsx
    ├── hooks/
    │   └── useAssetAccountDetail.ts
    ├── types/
    │   └── assetAccount.ts
    └── pages/
        └── AssetAccountDetailPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

### 2.51 フロントエンドで行わないこと

ACC-003のReact・TypeScript実装では、以下をフロントエンドの責務としない。

* 利用者境界の最終保証
* 論理削除状態の判定
* 現在年月の業務判定
* 利用可能資産設定の期間判定
* 利用可能資産設定の重複判定
* データ不整合の補正
* DB内部コードの解釈
* 保有商品の自動取得
* 月末資産データの自動取得

フロントエンドは、

```text
assetAccountId
    ↓
ACC-003
    ↓
レスポンス表示
```

という責務を基本とする。

---

## 3. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)