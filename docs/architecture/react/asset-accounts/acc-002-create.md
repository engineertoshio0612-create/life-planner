# ACC-002 資産口座登録

## 1. 概要

本ドキュメントでは、
ACC-002 資産口座登録APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

ACC-002では、
操作対象となる利用者に帰属する資産口座を新規登録し、
資産口座の基本情報と初期の利用可能資産区分を設定する。

フロントエンドでは、
資産口座登録画面から以下の情報を入力し、
API契約に沿ったリクエストを生成してACC-002を実行する。

* 資産口座名
* 資産種別
* 残高記録単位
* 利用開始年月
* 利用可能資産区分

ACC-002はサーバー状態を変更するAPIであるため、
TanStack QueryではMutationとして扱う。

登録成功後は、
ACC-001 資産口座一覧取得に対応するQuery Cacheを無効化し、
サーバー側の最新状態を再取得することを基本とする。

また、資産口座名の重複判定、
利用者境界の保証、初期利用可能資産設定の生成、
トランザクション制御などはLaravel側の責務とし、
フロントエンドでは最終的なデータ整合性を判定しない。

API通信、共通エラーハンドリング、Query Key、
TanStack Queryの利用方針などの共通事項については、
Reactアーキテクチャ共通設計に従うものとし、
本ドキュメントではACC-002固有の利用方法のみを定義する。


---

## 2. React・TypeScriptでの利用

ACC-002は、利用者が新しい資産口座を登録する際に使用する。

フロントエンドでは、資産口座登録画面から以下の情報を入力してACC-002を実行する。

* 資産口座名
* 資産種別
* 残高記録単位
* 利用開始年月
* 利用可能資産区分

概念的な利用フローは、以下とする。

```text
資産口座一覧
    ↓
資産口座登録画面
    ↓
登録内容入力
    ↓
クライアント側入力チェック
    ↓
ACC-002
POST
/api/v1/asset-accounts
    ↓
成功
    ↓
関連Query Cache無効化
    ↓
資産口座一覧へ戻る
```

---

### 2.1 TypeScript型

ACC-002のリクエスト型は、以下のように定義する。

概念例：

```ts
export type CreateAssetAccountRequest = {
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  startYearMonth: string;
  isAvailable: boolean;
};
```

正常レスポンスのデータ型は、以下のように定義する。

```ts
export type CreateAssetAccountResult = {
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
export type CreateAssetAccountResponse =
  ApiResponse<CreateAssetAccountResult>;
```

---

### 2.2 AssetType

`assetType`は、自由な`string`ではなく、APIで定義した値だけを扱える型とする。

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

画面表示用の名称とAPI送信用の値は分離する。

概念例：

```ts
export const assetTypeOptions = [
  {
    value: 'CASH',
    label: '現金',
  },
  {
    value: 'BANK',
    label: '銀行',
  },
  {
    value: 'SECURITIES',
    label: '証券',
  },
  {
    value: 'IDECO',
    label: 'iDeCo',
  },
  {
    value: 'CORPORATE_DC',
    label: '企業型DC',
  },
  {
    value: 'OTHER',
    label: 'その他',
  },
] satisfies ReadonlyArray<{
  value: AssetType;
  label: string;
}>;
```

---

### 2.3 BalanceRecordingUnit

`balanceRecordingUnit`も、APIで定義した値だけを扱える型とする。

概念例：

```ts
export type BalanceRecordingUnit =
  | 'ACCOUNT'
  | 'HOLDING';
```

画面表示では、例えば以下のように扱う。

```ts
export const balanceRecordingUnitOptions = [
  {
    value: 'ACCOUNT',
    label: '口座単位',
  },
  {
    value: 'HOLDING',
    label: '商品単位',
  },
] satisfies ReadonlyArray<{
  value: BalanceRecordingUnit;
  label: string;
}>;
```

---

### 2.4 登録フォーム

資産口座登録画面では、以下の入力項目を表示する。

| 画面項目   | API項目                  | UI例               |
| ------ | ---------------------- | ----------------- |
| 資産口座名  | `name`                 | テキスト入力            |
| 資産種別   | `assetType`            | セレクトボックス          |
| 残高記録単位 | `balanceRecordingUnit` | ラジオボタンまたはセレクトボックス |
| 利用開始年月 | `startYearMonth`       | 年月入力              |
| 利用可能資産 | `isAvailable`          | チェックボックス          |

---

### 2.5 フォームState

概念例：

```ts
export type AssetAccountFormValues = {
  name: string;
  assetType: AssetType | '';
  balanceRecordingUnit:
    | BalanceRecordingUnit
    | '';
  startYearMonth: string;
  isAvailable: boolean;
};
```

初期値の例：

```ts
const initialValues: AssetAccountFormValues = {
  name: '',
  assetType: '',
  balanceRecordingUnit: '',
  startYearMonth: '',
  isAvailable: false,
};
```

初期値については、画面設計で決定したUX方針を優先する。

---

### 2.6 利用開始年月

`startYearMonth`は、

```text
YYYY-MM
```

形式でAPIへ送信する。

HTMLの`month`入力を使用する場合は、概念的に以下とする。

```tsx
<input
  type="month"
  value={form.startYearMonth}
  onChange={(event) => {
    setForm((current) => ({
      ...current,
      startYearMonth:
        event.target.value,
    }));
  }}
/>
```

APIへ送信するために、

```text
YYYY-MM-01
```

などの日付へ変換しない。

---

### 2.7 isAvailable

`isAvailable`は、booleanとして管理する。

概念例：

```tsx
<input
  type="checkbox"
  checked={form.isAvailable}
  onChange={(event) => {
    setForm((current) => ({
      ...current,
      isAvailable:
        event.target.checked,
    }));
  }}
/>
```

APIへは、

```json
{
  "isAvailable": true
}
```

または、

```json
{
  "isAvailable": false
}
```

として送信する。

以下のような値へ変換しない。

```text
"true"
"false"
1
0
```

---

### 2.8 利用可能資産設定の詳細を入力させない

ACC-002では、利用可能資産設定の

```text
startYearMonth
endYearMonth
```

を個別入力させない。

初期利用可能資産設定の開始年月は、資産口座の

```text
startYearMonth
```

からサーバー側で設定する。

そのため、画面では

```text
資産口座の利用開始年月
+
利用可能資産かどうか
```

だけを入力する。

---

### 2.9 userIdをフォームへ持たせない

ACC-002では、登録対象利用者をリクエストボディから指定しない。

そのため、フォーム型へ

```ts
userId: string;
```

を追加しない。

以下のようなリクエストも作成しない。

```ts
const request = {
  userId: currentUserId,
  name: form.name,
  // ...
};
```

利用者IDは、`X-User-Id`として共通API Clientから送信する。

---

### 2.10 X-User-Id

`X-User-Id`は、ACC-002専用処理ではなく、共通API Clientから付与する。

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

ACC-002のコンポーネントから直接HTTPヘッダーを組み立てない。

---

### 2.11 API Client

ACC-002を呼び出す専用関数を定義する。

概念例：

```ts
export const createAssetAccount =
  async (
    request: CreateAssetAccountRequest,
  ): Promise<CreateAssetAccountResult> => {
    const response =
      await apiClient.post<
        CreateAssetAccountResponse
      >(
        '/api/v1/asset-accounts',
        request,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さず、API Clientへ処理を集約する。

---

### 2.12 FormからRequestへの変換

フォームStateをそのままAPIへ送信するのではなく、必要に応じてRequest型へ変換する。

概念例：

```ts
const request: CreateAssetAccountRequest = {
  name: form.name.trim(),
  assetType: form.assetType as AssetType,
  balanceRecordingUnit:
    form.balanceRecordingUnit
      as BalanceRecordingUnit,
  startYearMonth:
    form.startYearMonth,
  isAvailable:
    form.isAvailable,
};
```

ただし、型アサーションに依存するより、フォームバリデーション後に型が確定する設計を優先する。

---

### 2.13 クライアント側バリデーション

API実行前に、利用者の入力ミスを早期に通知するため、フロントエンドでも基本的な入力チェックを行う。

主に以下を確認する。

```text
name
    必須
    最大文字数

assetType
    必須

balanceRecordingUnit
    必須

startYearMonth
    必須
    YYYY-MM

isAvailable
    boolean
```

ただし、フロントエンド側のバリデーションだけをデータ整合性保証とはしない。

最終的な検証はLaravel側で行う。

---

### 2.14 資産口座名の重複確認

フロントエンドでは、資産口座名の重複を登録前に確定判定しない。

例えば、ACC-001で取得済みの一覧から

```ts
assetAccounts.some(
  (assetAccount) =>
    assetAccount.name === form.name,
);
```

だけで登録可否を決定しない。

キャッシュが古い可能性や、同時登録の可能性があるためである。

同名重複の最終判定は、ACC-002のサーバー処理に任せる。

---

### 2.15 Mutationとして扱う

ACC-002は、サーバー状態を変更するため、TanStack Queryを使用する場合はMutationとして扱う。

概念例：

```ts
export const useCreateAssetAccount =
  () => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn:
        createAssetAccount,

      onSuccess: async () => {
        await queryClient
          .invalidateQueries({
            queryKey: [
              'assetAccounts',
            ],
          });
      },
    });
  };
```

Queryとして自動実行しない。

---

### 2.16 登録処理

概念例：

```ts
const createMutation =
  useCreateAssetAccount();

const handleSubmit = (
  request: CreateAssetAccountRequest,
): void => {
  createMutation.mutate(
    request,
  );
};
```

登録ボタン押下時にACC-002を実行する。

---

### 2.17 二重送信防止

Mutation実行中は、登録ボタンを非活性化する。

概念例：

```tsx
<button
  type="submit"
  disabled={
    createMutation.isPending
  }
>
  {createMutation.isPending
    ? '登録中...'
    : '登録'}
</button>
```

これにより、通常操作による短時間の連続送信を防止する。

ただし、フロントエンド側の二重送信防止をサーバー側の重複防止の代替とはしない。

---

### 2.18 登録成功時

ACC-002成功時は、

```text
201 Created
```

とともに、登録した資産口座が返却される。

概念例：

```json
{
  "data": {
    "id": "10",
    "name": "証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

フロントエンドでは、正常レスポンスを確認後に登録成功として扱う。

---

### 2.19 成功メッセージ

正常終了後は、必要に応じて

```text
資産口座を登録しました。
```

などの完了メッセージを表示する。

---

### 2.20 成功後のQuery Cache

ACC-002成功後は、資産口座一覧の内容が変化するため、ACC-001に対応するQuery Cacheを無効化する。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey: [
    'assetAccounts',
  ],
});
```

Query Keyの正式な定義は、フロントエンド共通設計に従う。

---

### 2.21 Query Keyを共通化する場合

Query Keyを各コンポーネントへ直接記述せず、共通定義として管理してよい。

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

ACC-002成功後は、

```ts
queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.all,
});
```

のように使用する。

---

### 2.22 登録成功後の画面遷移

登録成功後は、基本的に資産口座一覧画面へ戻る。

概念的には、

```text
ACC-002成功
    ↓
Query Cache無効化
    ↓
成功メッセージ
    ↓
資産口座一覧へ遷移
```

とする。

画面設計によっては、登録した資産口座の詳細画面へ遷移してもよい。

その場合は、正常レスポンスの

```text
data.id
```

を使用する。

---

### 2.23 HOLDING登録後

```text
balanceRecordingUnit = HOLDING
```

で登録した場合でも、ACC-002成功直後に保有商品を自動登録しない。

画面設計上、続けて保有商品を登録させたい場合は、

```text
資産口座登録成功
    ↓
「保有商品を登録する」
導線を表示
    ↓
HLD-002
```

のように、別の業務操作として扱う。

---

### 2.24 ACCOUNT登録後

```text
balanceRecordingUnit = ACCOUNT
```

で登録した場合も、月末残高をACC-002から自動登録しない。

月末資産データの登録は、月末資産管理APIの責務とする。

---

### 2.25 VALIDATION_ERROR

Laravel側から

```text
VALIDATION_ERROR
```

が返却された場合は、`details`の`field`を利用して対応するフォーム項目へエラーを表示する。

概念例：

```text
name
    → 資産口座名

assetType
    → 資産種別

balanceRecordingUnit
    → 残高記録単位

startYearMonth
    → 利用開始年月

isAvailable
    → 利用可能資産区分
```

---

### 2.26 フィールドエラー

概念的には、以下のような型として扱える。

```ts
export type ApiValidationError = {
  field: string;
  reason: string;
  message: string;
};
```

例えば、

```ts
const nameError =
  error.details?.find(
    (detail) =>
      detail.field === 'name',
  );
```

として、資産口座名入力欄の近くへエラーを表示する。

---

### 2.27 ASSET_ACCOUNT_NAME_ALREADY_EXISTS

同一利用者内に同名資産口座が存在する場合は、

```text
409 Conflict
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となる。

フロントエンドでは、例えば

```text
同じ名前の資産口座が
すでに登録されています。
```

と表示する。

このエラーは、`name`に関連する業務エラーとして資産口座名入力欄の近くへ表示してもよい。

---

### 2.28 論理削除済み資産口座との重複

論理削除済みの同名資産口座が存在する場合も、

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となる。

フロントエンドでは、論理削除済みかどうかを推測してメッセージを分岐しない。

サーバーから返却された業務エラーとして共通の重複メッセージを表示する。

---

### 2.29 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-002のフォーム内で独自の復旧処理を実装しない。

---

### 2.30 INVALID_USER_ID

```text
INVALID_USER_ID
```

についても、API共通の利用者コンテキストエラーとして扱う。

必要に応じて、現在の利用者選択状態を再確認する。

---

### 2.31 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択されている利用者が有効ではない状態として扱う。

ACC-002固有のフォーム入力エラーとして表示しない。

---

### 2.32 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
資産口座を登録できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

登録成功扱いとして一覧へ遷移してはならない。

---

### 2.33 通信結果が不明な場合

通信断などにより、クライアント側でACC-002の成功・失敗を判断できない場合がある。

ACC-002はIdempotency-Keyを使用しないため、無条件に同じリクエストを自動再送しない。

最初のリクエストがサーバー側で成功していた場合、再送すると

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

となる可能性がある。

Phase1では、必要に応じてACC-001を再取得して登録状態を確認する。

---

### 2.34 自動Retry

ACC-002のMutationでは、原則として自動Retryを行わない。

概念例：

```ts
useMutation({
  mutationFn:
    createAssetAccount,

  retry: false,
});
```

登録APIであり、通信結果が不明な状態での自動再送によって利用者が意図しない再登録処理が発生することを避けるためである。

---

### 2.35 Optimistic Update

Phase1では、ACC-002に対するOptimistic Updateは必須としない。

資産口座登録は高頻度操作ではなく、以下のサーバー側処理を伴う。

```text
資産口座名重複確認
    ↓
asset_accounts登録
    ↓
初期利用可能資産設定登録
    ↓
transaction commit
```

そのため、

```text
API成功
    ↓
Query Cache無効化
    ↓
ACC-001再取得
```

という単純な方式を基本とする。

---

### 2.36 サーバー状態を正とする

登録成功後に、フォーム入力値だけを使用して資産口座一覧へ新規項目を追加することを必須としない。

サーバー側では、以下が行われる。

* ID採番
* Enum変換
* 初期利用可能資産設定登録
* 利用状態決定

そのため、Phase1ではACC-001を再取得してサーバー状態へ同期する方式を基本とする。

---

### 2.37 isEnabled

ACC-002成功時の

```text
isEnabled
```

は、常に`true`となる。

ただし、フロントエンドから

```ts
isEnabled: true
```

をリクエストへ追加しない。

利用状態はサーバー側で決定する。

---

### 2.38 DB内部項目をフロントエンド型へ持ち込まない

ACC-002のRequestおよびResponse型へ、以下のようなDB内部項目を追加しない。

```text
user_id
asset_type
balance_recording_unit
deleted_at
created_at
updated_at
asset_account_id
end_year_month
```

API契約として定義されたcamelCase項目だけを使用する。

---

### 2.39 API型と画面Stateを分離する

画面の都合によってフォームStateがAPI Request型と完全には一致しない場合がある。

例えば、

```ts
type AssetAccountFormValues = {
  name: string;
  assetType: AssetType | '';
  balanceRecordingUnit:
    | BalanceRecordingUnit
    | '';
  startYearMonth: string;
  isAvailable: boolean;
};
```

では、未選択状態として空文字を持つことができる。

一方、API Requestでは、

```ts
type CreateAssetAccountRequest = {
  name: string;
  assetType: AssetType;
  balanceRecordingUnit:
    BalanceRecordingUnit;
  startYearMonth: string;
  isAvailable: boolean;
};
```

として、有効な値だけを許可する。

画面入力途中の状態とAPI契約を同じ型へ無理に統合しない。

---

### 2.40 コンポーネントの責務

資産口座登録画面では、以下の責務を分離する。

```text
Page
    ↓
登録画面全体の制御

Form
    ↓
入力UI・入力エラー表示

Mutation Hook
    ↓
ACC-002実行・成功失敗処理

API Client
    ↓
HTTP通信

Type
    ↓
API契約
```

1つのReactコンポーネントへ以下をすべて直接記述しない。

* フォーム管理
* HTTP通信
* エラーコード判定
* Query Cache操作
* 画面遷移

---

### 2.41 概念的なディレクトリ構成

Phase1では、例えば以下のように機能単位で整理できる。

```text
features/
└── asset-accounts/
    ├── api/
    │   └── createAssetAccount.ts
    ├── components/
    │   └── AssetAccountForm.tsx
    ├── hooks/
    │   └── useCreateAssetAccount.ts
    ├── types/
    │   └── assetAccount.ts
    └── pages/
        └── CreateAssetAccountPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

### 2.42 フロントエンドで行わないこと

ACC-002のReact・TypeScript実装では、以下をフロントエンドの責務としない。

* 利用者境界の最終保証
* 資産口座名一意性の最終保証
* 論理削除済み資産口座を含む重複判定
* DB保存用コードへの変換
* 初期利用可能資産設定の開始年月決定
* 初期利用可能資産設定の終了年月決定
* `asset_account_id`の決定
* transaction制御
* `isEnabled`の決定

フロントエンドは、

```text
利用者入力
    ↓
API契約に沿ったRequest生成
    ↓
ACC-002実行
    ↓
結果表示
```

に責務を限定する。
:::

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [ACC-001 資産口座一覧取得](./acc-001-list.md)
- [ACC-003 資産口座詳細取得](./acc-003-detail.md)
- [ACC-004 資産口座更新](./acc-004-update.md)
- [ACC-005 利用可能資産設定履歴取得](./acc-005-available-setting-history.md)
- [ACC-006 利用可能資産設定登録](./acc-006-create-available-setting.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/asset-accounts.md)
- [Reactアーキテクチャ設計](../../../architecture/react/asset-accounts.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)
