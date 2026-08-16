# ACC-004 資産口座更新

## 1. 概要

本ドキュメントでは、
ACC-004 資産口座更新APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

ACC-004では、
操作対象となる利用者に帰属する指定された資産口座について、
通常属性である以下の項目を更新する。

* 資産口座名
* 資産種別

フロントエンドでは、
ACC-003 資産口座詳細取得で現在の資産口座情報を取得し、
編集フォームへ反映したうえで、変更された項目からPATCHリクエストを生成してACC-004を実行する。

残高記録単位、利用開始年月、利用可能資産区分、利用状態は
ACC-004の更新対象とせず、必要に応じて読み取り専用情報として表示する。

利用可能資産区分の変更はACC-006、
資産口座の無効化はACC-005として、
通常属性更新とは別の業務操作として扱う。

ACC-004はサーバー状態を変更するAPIであるため、
TanStack QueryではMutationとして扱う。

更新成功後は、
ACC-001 資産口座一覧取得およびACC-003 資産口座詳細取得に対応するQuery Cacheを無効化し、
サーバー側の最新状態へ同期することを基本とする。

また、利用者境界、論理削除判定、資産口座名の一意性、
利用可能資産設定の整合性などの最終保証はLaravel側の責務とし、
フロントエンドでは独自に業務データの整合性を判定しない。

API通信、共通エラーハンドリング、Query Key、
TanStack Queryの利用方針などの共通事項については、
Reactアーキテクチャ共通設計に従うものとし、
本ドキュメントではACC-004固有の利用方法のみを定義する。


---

## 2. React・TypeScriptでの利用

ACC-004は、利用者が登録済み資産口座の通常属性を編集する際に使用する。

フロントエンドでは、ACC-003 資産口座詳細取得で現在値を取得し、編集フォームへ反映したうえでACC-004を実行する。

概念的な利用フローは、以下とする。

```text
ACC-003
資産口座詳細取得
    ↓
編集フォーム初期表示
    ↓
利用者が編集
    ↓
変更内容をRequestへ変換
    ↓
ACC-004
PATCH
/api/v1/asset-accounts/{assetAccountId}
    ↓
成功
    ↓
関連Query Cache無効化
    ↓
最新状態を再取得
```

ACC-004は、サーバー状態を変更するため、TanStack Queryを使用する場合はMutationとして扱う。

---

### 2.1 TypeScript型

ACC-004のリクエスト型は、以下のように定義する。

概念例：

```typescript
export type UpdateAssetAccountRequest = {
  name?: string;
  assetType?: AssetType;
};
```

`PATCH`であるため、各項目は任意とする。

ただし、実際にAPIを呼び出す際は、

```text
name
assetType
```

の少なくとも1項目を指定する必要がある。

---

### 2.2 正常レスポンス型

正常レスポンスは、ACC-003と同じ資産口座詳細表現を使用する。

概念例：

```typescript
export type UpdateAssetAccountResult = {
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

```typescript
export type UpdateAssetAccountResponse =
  ApiResponse<UpdateAssetAccountResult>;
```

---

### 2.3 ACC-003と共通型を使用してよい

ACC-003とACC-004の正常レスポンス項目が同一である場合は、専用の重複型を作成せず、共通型を使用してよい。

概念例：

```typescript
export type AssetAccountDetail = {
  id: string;
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  isAvailable: boolean;
  startYearMonth: string;
  isEnabled: boolean;
};

export type UpdateAssetAccountResponse =
  ApiResponse<AssetAccountDetail>;
```

同じ意味のデータにAPIごとに異なる型名を増やしすぎない。

---

### 2.4 編集可能な項目

ACC-004で編集できる項目は、以下のみとする。

```text
name
assetType
```

編集画面では、これらを入力可能な状態とする。

---

### 2.5 編集できない項目

以下は、ACC-004では更新できない。

- `balanceRecordingUnit`
- `startYearMonth`
- `isAvailable`
- `isEnabled`

ACC-003から取得したこれらの情報を画面へ表示してもよいが、ACC-004用フォームでは編集可能な入力欄にしない。

---

### 2.6 balanceRecordingUnit

残高記録単位は、Phase1では登録後変更不可とする。

そのため、編集画面で表示する場合は読み取り専用とする。

概念例：

```tsx
<ReadonlyField
  label="残高記録単位"
  value={
    balanceRecordingUnitLabels[
      assetAccount.balanceRecordingUnit
    ]
  }
/>
```

以下のような編集可能なセレクトボックスにはしない。

```tsx
<select
  value={form.balanceRecordingUnit}
>
  ...
</select>
```

---

### 2.7 startYearMonth

利用開始年月も、ACC-004では変更不可とする。

画面へ表示する場合は、読み取り専用とする。

例えば、

```text
利用開始年月
2026年8月
```

のように表示する。

---

### 2.8 isAvailable

利用可能資産区分の変更は、ACC-006の責務である。

そのため、ACC-004の編集フォームへ

```typescript
isAvailable: boolean;
```

を更新項目として含めない。

利用可能資産区分を変更するUIは、ACC-006を実行する別操作として設計する。

---

### 2.9 isEnabled

資産口座無効化は、ACC-005の責務とする。

そのため、ACC-004のRequestへ

```typescript
isEnabled?: boolean;
```

を追加しない。

以下のような汎用的な状態更新は行わない。

```typescript
updateAssetAccount(
  assetAccountId,
  {
    isEnabled: false,
  },
);
```

---

### 2.10 編集フォーム型

画面上の編集フォームStateは、以下のように定義できる。

概念例：

```typescript
export type UpdateAssetAccountFormValues = {
  name: string;
  assetType: AssetType;
};
```

ACC-003から取得した編集不可項目までフォームStateへ含める必要はない。

---

### 2.11 ACC-003から初期値を設定する

編集画面では、ACC-003で取得した

```text
name
assetType
```

を初期値として使用する。

概念例：

```typescript
const initialValues:
  UpdateAssetAccountFormValues = {
    name:
      assetAccount.name,

    assetType:
      assetAccount.assetType,
  };
```

---

### 2.12 Queryデータを直接編集しない

ACC-003で取得したQuery Cache上の

```text
AssetAccountDetail
```

を、フォーム入力によって直接書き換えない。

例えば、

```typescript
assetAccount.name =
  inputValue;
```

のような変更は行わない。

編集用Stateへ必要な値をコピーする。

---

### 2.13 変更項目だけを送信する

ACC-004はPATCHであるため、実際に変更された項目だけをRequestへ含めてよい。

例えば、資産口座名だけを変更した場合は、

```json
{
  "name": "メイン証券口座"
}
```

とする。

---

### 2.14 assetTypeだけ変更した場合

資産種別だけを変更した場合は、

```json
{
  "assetType": "BANK"
}
```

とする。

未変更の`name`を必ず送信する必要はない。

---

### 2.15 両方変更した場合

両方変更した場合は、

```json
{
  "name": "メイン証券口座",
  "assetType": "SECURITIES"
}
```

とする。

---

### 2.16 Request生成

現在値とフォーム値を比較し、変更された項目だけをRequestへ設定してよい。

概念例：

```typescript
export const buildUpdateAssetAccountRequest =
  (
    current: AssetAccountDetail,
    form: UpdateAssetAccountFormValues,
  ): UpdateAssetAccountRequest => {
    const request:
      UpdateAssetAccountRequest = {};

    if (
      form.name !== current.name
    ) {
      request.name =
        form.name;
    }

    if (
      form.assetType
        !== current.assetType
    ) {
      request.assetType =
        form.assetType;
    }

    return request;
  };
```

---

### 2.17 空Requestを送信しない

変更がない場合は、

```json
{}
```

をACC-004へ送信しない。

概念例：

```typescript
const request =
  buildUpdateAssetAccountRequest(
    assetAccount,
    form,
  );

if (
  Object.keys(request).length === 0
) {
  return;
}
```

画面上では、例えば

```text
変更内容がありません。
```

と表示してよい。

または、変更がない間は更新ボタンを非活性化してもよい。

---

### 2.18 更新ボタンの活性制御

現在値とフォーム値が同一の場合は、更新ボタンを非活性化してよい。

概念例：

```typescript
const hasChanges =
  form.name !==
    assetAccount.name
  ||
  form.assetType !==
    assetAccount.assetType;
```

```tsx
<button
  type="submit"
  disabled={
    !hasChanges
    || updateMutation.isPending
  }
>
  更新
</button>
```

ただし、サーバー側でも空Requestを拒否する。

---

### 2.19 nameの前処理

資産口座名について、API共通の文字列入力方針で前後空白を除去する場合は、Request生成時に統一して処理してよい。

概念例：

```typescript
const name =
  form.name.trim();
```

ただし、フロントエンドとバックエンドで異なる正規化ルールを持たせない。

---

### 2.20 AssetType

`assetType`は、ACC-002・ACC-003と共通の型を使用する。

概念例：

```typescript
export type AssetType =
  | 'CASH'
  | 'BANK'
  | 'SECURITIES'
  | 'IDECO'
  | 'CORPORATE_DC'
  | 'OTHER';
```

画面表示用ラベルは、共通定義を使用する。

---

### 2.21 API Client

ACC-004を呼び出す専用API Client関数を定義する。

概念例：

```typescript
export const updateAssetAccount =
  async (
    assetAccountId: string,
    request: UpdateAssetAccountRequest,
  ): Promise<AssetAccountDetail> => {
    const response =
      await apiClient.patch<
        UpdateAssetAccountResponse
      >(
        `/api/v1/asset-accounts/${assetAccountId}`,
        request,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

### 2.22 userIdをAPI Client引数へ含めない

以下のような関数にはしない。

```typescript
updateAssetAccount(
  userId,
  assetAccountId,
  request,
);
```

利用者IDは、共通API Clientから

```text
X-User-Id
```

として付与する。

---

### 2.23 X-User-Id

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

ACC-004専用コンポーネントでヘッダーを直接生成しない。

---

### 2.24 Mutationとして扱う

ACC-004は、資産口座の状態を変更するため、TanStack QueryではMutationとして扱う。

概念例：

```typescript
export type UpdateAssetAccountVariables = {
  assetAccountId: string;
  request: UpdateAssetAccountRequest;
};
```

```typescript
export const useUpdateAssetAccount =
  () => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn: (
        variables:
          UpdateAssetAccountVariables,
      ) =>
        updateAssetAccount(
          variables.assetAccountId,
          variables.request,
        ),

      onSuccess: async (
        result,
      ) => {
        await queryClient
          .invalidateQueries({
            queryKey:
              assetAccountKeys.all,
          });

        await queryClient
          .invalidateQueries({
            queryKey:
              assetAccountKeys.detail(
                result.id,
              ),
          });
      },
    });
  };
```

---

### 2.25 更新処理

概念例：

```typescript
const updateMutation =
  useUpdateAssetAccount();

const handleSubmit = (): void => {
  const request =
    buildUpdateAssetAccountRequest(
      assetAccount,
      form,
    );

  if (
    Object.keys(request).length === 0
  ) {
    return;
  }

  updateMutation.mutate({
    assetAccountId:
      assetAccount.id,

    request,
  });
};
```

---

### 2.26 二重送信防止

Mutation実行中は、更新ボタンを非活性化する。

概念例：

```tsx
<button
  type="submit"
  disabled={
    updateMutation.isPending
  }
>
  {updateMutation.isPending
    ? '更新中...'
    : '更新'}
</button>
```

フロントエンド側の二重送信防止だけをサーバー側整合性保証とはしない。

---

### 2.27 自動Retry

ACC-004のMutationでは、原則として自動Retryを行わない。

概念例：

```typescript
useMutation({
  mutationFn:
    updateAssetAccount,
  retry: false,
});
```

更新処理であり、通信結果が不明な状態で自動再送するより、利用者へ状態を通知したうえで必要に応じて再取得する方針とする。

---

### 2.28 同一Requestの再送

ACC-004は、同じ値を再度設定しても最終的な業務状態は同じとなる。

そのため、利用者が手動で再実行すること自体は可能である。

ただし、ネットワークエラー時にフロントエンドが自動的に繰り返し送信する設計はPhase1では採用しない。

---

### 2.29 成功時

ACC-004成功時は、

```http
200 OK
```

とともに、更新後の資産口座詳細が返却される。

概念例：

```json
{
  "data": {
    "id": "10",
    "name": "メイン証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

---

### 2.30 成功メッセージ

正常終了後は、必要に応じて

```text
資産口座を更新しました。
```

などの完了メッセージを表示する。

---

### 2.31 成功後の画面遷移

更新成功後は、画面設計に応じて

- 資産口座詳細画面へ戻る
- 資産口座一覧画面へ戻る
- 編集画面に留まり最新値を表示する

などの動作とする。

Phase1では、詳細画面または一覧画面へ戻るシンプルな導線としてよい。

---

### 2.32 ACC-001のQuery Cache

ACC-004によって、

```text
name
assetType
```

が変更されるため、資産口座一覧のQuery Cacheを無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.all,
});
```

---

### 2.33 ACC-003のQuery Cache

対象資産口座の詳細Queryも無効化する。

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

これにより、ACC-003から更新後の最新状態を取得する。

---

### 2.34 成功レスポンスでCacheを更新する方式

ACC-004の成功レスポンスは、ACC-003と同じ資産口座詳細表現であるため、成功結果を直接詳細Query Cacheへ設定することもできる。

概念例：

```typescript
queryClient.setQueryData(
  assetAccountKeys.detail(
    result.id,
  ),
  result,
);
```

ただし、Phase1では

```text
Mutation成功
    ↓
invalidate
    ↓
再取得
```

の単純な方式を基本としてよい。

---

### 2.35 Optimistic Update

Phase1では、ACC-004に対するOptimistic Updateを必須としない。

資産口座名については、

- 同名重複
- 他利用者境界
- 論理削除状態
- UNIQUE制約競合

など、サーバー側で最終判定する条件がある。

そのため、サーバー成功後に画面状態を更新する方式を基本とする。

---

### 2.36 VALIDATION_ERROR

Laravel側から

```text
VALIDATION_ERROR
```

が返却された場合は、`error.details`を利用して対応するフォーム項目へエラー表示する。

主な対象は、

```text
name
assetType
```

とする。

---

### 2.37 nameのフィールドエラー

例えば、

```text
資産口座名を入力してください。
```

や、

```text
資産口座名は100文字以内で入力してください。
```

などを、資産口座名入力欄の近くへ表示する。

具体的な文言は、APIレスポンスまたはフロントエンド共通エラー表示方針に従う。

---

### 2.38 assetTypeのフィールドエラー

`assetType`に未定義値が返された場合は、資産種別入力欄のエラーとして扱う。

通常のUIでは定義済みセレクト値しか送信しないため、発生頻度は低い。

---

### 2.39 ASSET_ACCOUNT_NAME_ALREADY_EXISTS

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

`name`に関連する業務エラーとして、資産口座名入力欄の近くへ表示してよい。

---

### 2.40 同時更新による重複も同じ扱いとする

DBのUNIQUE制約によって名前競合が検出された場合も、フロントエンドでは

```text
ASSET_ACCOUNT_NAME_ALREADY_EXISTS
```

として扱う。

アプリケーション事前判定か、DB制約による判定かをクライアントから区別しない。

---

### 2.41 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

となる。

- 資産口座不存在
- 他利用者所属
- 論理削除済み

フロントエンドでは、原因を推測せず、

```text
指定された資産口座が見つかりません。
```

などの共通表示とする。

---

### 2.42 更新中に無効化された場合

編集画面表示後、別処理によって対象資産口座が無効化される可能性がある。

その後ACC-004を実行すると、

```text
ASSET_ACCOUNT_NOT_FOUND
```

となり得る。

この場合は、編集画面へ残して再送を繰り返さず、資産口座一覧へ戻る導線を表示してよい。

---

### 2.43 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-004専用のフォームエラーとして表示しない。

---

### 2.44 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、API共通の利用者コンテキストエラーとして扱う。

---

### 2.45 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択されている利用者が有効ではない状態として共通処理する。

---

### 2.46 INVALID_ASSET_ACCOUNT_ID

通常の画面遷移では、ACC-001やACC-003から取得した有効なIDを使用するため、発生頻度は低い。

発生した場合は、不正なURLまたは画面状態として扱う。

---

### 2.47 ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND

現在の利用可能資産設定が存在しない場合は、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

となる。

これは、フォーム入力による問題ではなくサーバー側データ不整合である。

画面では、一般的な更新失敗として扱う。

例えば、

```text
資産口座を更新できませんでした。
```

と表示する。

---

### 2.48 ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT

現在有効な利用可能資産設定が複数存在する場合も、サーバー側データ不整合として扱う。

フロントエンドで利用可能資産設定を独自に選択して更新処理を継続しない。

---

### 2.49 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
資産口座を更新できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

### 2.50 エラー時のフォーム値

ACC-004が失敗した場合は、原則として利用者が入力したフォーム値を保持する。

入力内容を自動的に初期値へ戻さない。

これにより、入力内容を修正して再実行できるようにする。

---

### 2.51 ASSET_ACCOUNT_NOT_FOUND時のフォーム値

ただし、

```text
ASSET_ACCOUNT_NOT_FOUND
```

の場合は、対象資産口座自体が編集不能な可能性がある。

そのため、一覧へ戻すなど画面遷移を優先してよい。

---

### 2.52 サーバー状態との競合

Phase1では`version`や`If-Match`を使用しないため、編集画面表示後に別リクエストで同じ資産口座が更新されていても、ACC-004は更新競合エラーを返さない。

同一項目を更新した場合は、Last Write Winsとなる可能性がある。

フロントエンドでも独自の競合判定を実装しない。

---

### 2.53 updatedAtを送信しない

競合判定用として

```typescript
updatedAt: string;
```

をACC-004のRequestへ追加しない。

Phase1では、更新時刻比較による楽観ロックを採用しないためである。

---

### 2.54 balanceRecordingUnit変更UIを設けない

ACC-004では残高記録単位を変更できないため、編集画面に

```text
ACCOUNT
HOLDING
```

を切り替える入力UIを設けない。

表示する場合は読み取り専用とする。

---

### 2.55 利用可能資産変更UIとの分離

資産口座編集画面に利用可能資産変更機能を同時に配置する場合でも、内部的にはACC-004とは別操作として扱う。

概念的には、

```text
基本情報保存
    ↓
ACC-004

利用可能資産区分変更
    ↓
ACC-006
```

とする。

1つのACC-004 Requestへまとめない。

---

### 2.56 無効化UIとの分離

無効化ボタンを同じ画面へ配置する場合でも、

```text
通常属性更新
    → ACC-004

無効化
    → ACC-005
```

とAPIを分離する。

例えば、「保存」ボタンと「無効化」ボタンを別操作として扱う。

---

### 2.57 Pageの責務

編集Pageでは、主に以下を担当する。

- URLから`assetAccountId`取得
- ACC-003による初期データ取得
- 編集フォーム表示
- ACC-004成功後の画面遷移
- ページ単位のエラー表示

HTTP通信の詳細を直接記述しない。

---

### 2.58 Form Componentの責務

Form Componentでは、主に以下を担当する。

- `name`入力
- `assetType`入力
- クライアント側入力チェック
- フィールドエラー表示
- submitイベント通知

ACC-004のHTTP通信を直接実行しない構成としてよい。

---

### 2.59 Mutation Hookの責務

Mutation Hookでは、主に以下を担当する。

```text
ACC-004実行
成功処理
Query Cache無効化
```

画面固有の表示レイアウトをMutation Hookへ持たせない。

---

### 2.60 API Clientの責務

API Clientでは、

```http
PATCH /api/v1/asset-accounts/{assetAccountId}
```

のHTTP通信と型付きレスポンス取得を担当する。

画面遷移やToast表示などはAPI Clientへ含めない。

---

### 2.61 概念的なディレクトリ構成

例えば、以下のように機能単位で整理できる。

```text
features/
└── asset-accounts/
    ├── api/
    │   ├── getAssetAccountDetail.ts
    │   └── updateAssetAccount.ts
    ├── components/
    │   └── AssetAccountForm.tsx
    ├── hooks/
    │   ├── useAssetAccountDetail.ts
    │   └── useUpdateAssetAccount.ts
    ├── types/
    │   └── assetAccount.ts
    └── pages/
        └── EditAssetAccountPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

### 2.62 フロントエンドで行わないこと

ACC-004のReact・TypeScript実装では、以下をフロントエンドの責務としない。

- 利用者境界の最終保証
- 資産口座存在確認の最終保証
- 論理削除判定
- 資産口座名一意性の最終保証
- DBのUNIQUE競合判定
- DB内部コードへの変換
- 現在利用可能資産設定の期間判定
- データ不整合の補正
- `balanceRecordingUnit`変更
- `startYearMonth`変更
- `isAvailable`変更
- `isEnabled`変更
- 楽観ロック判定

フロントエンドは、

```text
現在値取得
    ↓
編集可能項目の入力
    ↓
変更Request生成
    ↓
ACC-004実行
    ↓
結果表示
```

に責務を限定する。

---

## 3. 関連ドキュメント

- [ACC-004 API詳細設計](../../../api/details/asset-accounts/acc-004-update.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../../laravel/laravel-architecture.md)
- [資産口座 Laravelアーキテクチャ設計](../../laravel/asset-accounts/README.md)
- [ACC-004 Laravelアーキテクチャ設計](../../laravel/asset-accounts/acc-004-update.md)
- [Reactアーキテクチャ共通設計](../react-architecture.md)
- [資産口座 Reactアーキテクチャ設計](./README.md)
- [ACC-004 テスト設計](../../../tests/asset-accounts/acc-004-update.md)
- [資産口座 テスト設計](../../../tests/asset-accounts/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)