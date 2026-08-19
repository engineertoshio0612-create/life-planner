# BAL-002 月末資産残高登録

## 1. 概要

本ドキュメントでは、
BAL-002 月末資産残高登録APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

BAL-002では、
操作対象となる利用者について、
指定した月末資産状況に紐づく
資産口座単位の月末資産残高を新規登録する。

React・TypeScriptでは、
主に月末資産状況の入力画面から、
登録対象となる資産口座と月末残高を受け取り、
BAL-002を呼び出して登録処理を実行する。

本APIの登録対象は、
残高記録単位が資産口座単位である
資産口座のみとする。

商品単位で残高を記録する資産口座については、
BAL-002を使用せず、
VAL-002 商品別月末評価額登録APIを使用する。

フロントエンドでは、
入力値に対する基本的なバリデーションや
登録処理中の二重送信防止などを行うが、

* 月末資産状況の存在確認
* 月末資産状況の確定状態確認
* 資産口座の存在確認
* 対象年月時点の資産口座の利用可否判定
* 残高記録単位の判定
* 月末資産残高の重複登録判定

などの業務ルールについては、
フロントエンドだけで保証しない。

最終的な登録可否は、
バックエンドの判定結果に従う。

また、`balance = 0`は
0円で登録済みであることを表す
有効な入力値として扱い、
未入力状態とは明確に区別する。

概念的な利用構成は、
以下とする。

```text
月末資産状況画面
    ↓
登録対象の資産口座を選択
    ↓
月末資産残高を入力
    ↓
フロントエンドバリデーション
    ↓
BAL-002 月末資産残高登録
    ↓
登録成功
    ↓
BAL-001 または SNP-003 を再取得
    ↓
画面を最新状態へ更新
```

登録成功後は、
BAL-002のレスポンスだけで
画面全体の状態を更新しようとせず、
必要に応じてBAL-001 月末資産残高一覧取得APIや
SNP-003 月末資産状況詳細取得APIを再取得する。

また、同一の月末資産状況・資産口座に対して
すでに月末資産残高が登録されている場合は、
BAL-002による再登録で上書きせず、
BAL-003 月末資産残高更新APIを使用する。

---

## 2. React・TypeScriptでの利用

React・TypeScriptでは、本APIを月末資産状況に対して資産口座単位の月末資産残高を新規登録する処理で使用する。

主な利用箇所は、月末資産状況の入力画面における資産口座残高の登録フォームとする。

フロントエンドでは、対象となる月末資産状況の `snapshotId` をパスパラメータとして指定し、登録対象となる資産口座IDおよび月末残高をリクエストボディとして送信する。

```text
月末資産状況画面
    ↓
資産口座を選択
    ↓
月末資産残高を入力
    ↓
登録
    ↓
POST /api/v1/month-end-asset-snapshots/{snapshotId}/asset-balances
    ↓
登録成功
    ↓
月末資産状況画面へ反映
```

---

### 2.1 リクエスト型

```ts
export type CreateMonthEndAssetBalanceRequest = {
  assetAccountId: string;
  balance: number;
};
```

`assetAccountId`は、API共通方針に従って文字列として扱う。

`balance`は、日本円の整数値として扱う。

```ts
const request: CreateMonthEndAssetBalanceRequest = {
  assetAccountId: '3',
  balance: 1200000,
};
```

`balance = 0`は有効な値であるため、フロントエンドでも`0`を未入力として扱わない。

例えば、以下のような判定は行わない。

```ts
if (!balance) {
  // 0もfalseとして扱われるため不適切
}
```

未入力判定が必要な場合は、`null`、`undefined`または入力文字列の状態を明示的に判定する。

---

### 2.2 API呼び出し例

```ts
export const createMonthEndAssetBalance = async (
  snapshotId: string,
  userId: string,
  request: CreateMonthEndAssetBalanceRequest,
): Promise<CreateMonthEndAssetBalanceResponse> => {
  const response = await fetch(
    `/api/v1/month-end-asset-snapshots/${snapshotId}/asset-balances`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Accept: 'application/json',
        'X-User-Id': userId,
      },
      body: JSON.stringify(request),
    },
  );

  if (!response.ok) {
    throw await response.json();
  }

  return response.json();
};
```

Phase1では、操作対象利用者を`X-User-Id`リクエストヘッダーで指定する。

利用者IDをリクエストボディへ含めない。

---

### 2.3 レスポンス型

```ts
export type MonthEndAssetBalance = {
  assetAccountId: string;
  balance: number;
};

export type CreateMonthEndAssetBalanceResponse = {
  data: MonthEndAssetBalance;
};
```

レスポンス例：

```ts
const response: CreateMonthEndAssetBalanceResponse = {
  data: {
    assetAccountId: '3',
    balance: 1200000,
  },
};
```

本APIのレスポンスには、月末資産状況そのものの情報や資産口座名などは含まれない。

そのため、登録成功後に画面全体の最新状態が必要な場合は、必要に応じてSNP-003 月末資産状況詳細取得APIまたはBAL-001 月末資産残高一覧取得APIを再実行する。

```text
BAL-002 登録成功
    ↓
必要に応じて
    ↓
SNP-003 または BAL-001 を再取得
    ↓
画面表示を最新状態へ更新
```

---

### 2.4 フォームでの扱い

月末資産残高登録フォームでは、主に以下の状態を管理する。

```ts
type MonthEndAssetBalanceFormState = {
  assetAccountId: string;
  balance: string;
};
```

入力フォーム上では、未入力状態を表現しやすくするため、`balance`を文字列として保持してもよい。

API送信時に整数へ変換する。

```ts
const request: CreateMonthEndAssetBalanceRequest = {
  assetAccountId: form.assetAccountId,
  balance: Number(form.balance),
};
```

送信前には、少なくとも以下を確認する。

- 資産口座が選択されていること
- 残高が入力されていること
- 残高が整数であること
- 残高が0以上であること

ただし、月末資産状況の確定状態、対象年月時点の資産口座の利用可能状態、残高記録単位、重複登録などの業務ルールについては、フロントエンドだけで保証しない。

最終的な登録可否はバックエンドの判定結果に従う。

---

### 2.5 登録対象資産口座の扱い

本APIで登録できるのは、残高記録単位が資産口座単位の資産口座のみである。

```text
口座単位
    ↓
BAL-002 月末資産残高登録

商品単位
    ↓
VAL-002 商品別月末評価額登録
```

画面側で資産口座の残高記録単位を取得できる場合は、登録UIを分けることで誤操作を防止してよい。

ただし、フロントエンドでBAL-002の呼び出しを制御している場合でも、残高記録単位の最終的な検証はバックエンドで行う。

---

### 2.6 登録処理中の制御

本APIはPOSTであり、同一の月末資産状況・資産口座に対する重複登録は許可されない。

そのため、登録処理中は登録ボタンを非活性化し、意図しない二重送信を防止する。

```tsx
<button
  type="submit"
  disabled={isSubmitting}
>
  {isSubmitting ? '登録中...' : '登録'}
</button>
```

```ts
const [isSubmitting, setIsSubmitting] = useState(false);

const handleSubmit = async () => {
  if (isSubmitting) {
    return;
  }

  setIsSubmitting(true);

  try {
    await createMonthEndAssetBalance(
      snapshotId,
      userId,
      request,
    );
  } finally {
    setIsSubmitting(false);
  }
};
```

ただし、フロントエンドによるボタン制御は補助的な対策とする。

重複登録の最終的な防止は、バックエンドの業務ルールおよびデータベースのUNIQUE制約によって保証する。

---

### 2.7 エラー処理

APIから返却されたエラーコードに応じて、画面表示を切り替える。

特に以下のエラーを考慮する。

```ts
type CreateMonthEndAssetBalanceErrorCode =
  | 'USER_CONTEXT_REQUIRED'
  | 'INVALID_USER_ID'
  | 'USER_NOT_FOUND'
  | 'VALIDATION_ERROR'
  | 'MONTH_END_ASSET_SNAPSHOT_NOT_FOUND'
  | 'MONTH_END_ASSET_SNAPSHOT_CONFIRMED'
  | 'ASSET_ACCOUNT_NOT_FOUND'
  | 'ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH'
  | 'ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH'
  | 'MONTH_END_ASSET_BALANCE_ALREADY_EXISTS'
  | 'INTERNAL_SERVER_ERROR';
```

`VALIDATION_ERROR`の場合は、`error.details`を使用して入力項目単位のエラーを表示できる。

```text
VALIDATION_ERROR
    ↓
error.detailsを確認
    ↓
assetAccountId / balance
    ↓
対象フォームへエラー表示
```

`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`の場合は、対象の月末資産状況がすでに確定済みであるため、登録フォームを継続して表示するのではなく、最新状態を再取得して画面を更新することを検討する。

`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`の場合は、同一の月末資産状況・資産口座についてすでに残高が登録されている。

この場合、BAL-002を再送して上書きせず、既存残高を変更する操作ではBAL-003 月末資産残高更新APIを使用する。

---

### 2.8 React Query等を利用する場合

データ取得ライブラリを利用する場合、本APIはMutationとして扱う。

概念的には以下のように構成する。

```ts
const mutation = useMutation({
  mutationFn: (request: CreateMonthEndAssetBalanceRequest) =>
    createMonthEndAssetBalance(
      snapshotId,
      userId,
      request,
    ),
  onSuccess: async () => {
    // 月末資産状況または月末資産残高一覧を再取得する
  },
});
```

登録成功後は、月末資産状況詳細または月末資産残高一覧に対応するキャッシュを無効化し、バックエンドの最新状態を再取得する。

これにより、BAL-002のレスポンスに含まれない月末資産状況や他の資産口座残高を含めて、画面全体を最新状態へ同期できる。

なお、元仕様では **`balance = 0`** は「未登録」とは別の有効な登録値として扱うため、React側でも`if (!balance)`のような判定を使用しない。

また、BAL-002は同一の月末資産状況・資産口座に対する再POSTで既存残高を上書きせず、`409 Conflict`とする。

既存の月末資産残高を変更する場合は、BAL-003 月末資産残高更新APIを使用する。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [BAL-001 月末資産残高一覧取得](./bal-001-list.md)
- [BAL-003 月末資産残高更新](./bal-003-update.md)
- [SNP-003 月末資産状況詳細取得](../month-end-asset-snapshots/snp-003-detail.md)
- [VAL-002 商品別月末評価額登録](../holding-values/val-002-create.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/asset-balances.md)
- [Reactアーキテクチャ設計](../../../architecture/react/asset-balances.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)
