# VAL-003 商品別月末評価額更新

## 1. 概要

操作対象となる利用者について、
指定した月末資産状況に紐づく
既存の商品別月末評価額を更新する。

更新対象は、
残高記録単位が商品単位である資産口座に属する
保有商品の登録済みの商品別月末評価額とする。

React・TypeScript側では、
VAL-001 商品別月末評価額一覧取得APIで取得した状態をもとに、
登録済みの商品別月末評価額に対して
VAL-003を実行する。

更新時には、
`snapshotId`、
`holdingAssetId`、
`value`
を使用する。

`snapshotId`および`holdingAssetId`は
パスパラメータとして指定し、
`value`のみをリクエストボディへ送信する。

`value`は日本円の整数値として扱い、
`0`も有効な更新値とする。
未入力と0円を区別し、
truthy / falsyによる判定は行わない。

商品別月末評価額が未登録の場合は、
VAL-003では新規登録せず、
VAL-002 商品別月末評価額登録APIを使用する。

また、
月末資産状況が確定済みの場合は
更新UIを非活性化してよいが、
更新可否の最終判断はバックエンドに委ねる。

更新成功後は、
対象 `snapshotId` のVAL-001を再取得し、
サーバー上の最新の商品別月末評価額を
画面へ反映することを基本とする。

利用者境界、
対象年月時点の資産口座・保有商品の利用可否、
残高記録単位、
商品別月末評価額の存在確認などの業務ルールについては、
フロントエンドで最終判定せず、
バックエンドの判定結果に従う。

口座単位で残高を記録する資産口座は
VAL-003の対象外とし、
月末資産残高の更新には
BAL-003 月末資産残高更新APIを使用する。

---

## 2. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type UpdateMonthEndHoldingValueParams = {
  snapshotId: string;
  holdingAssetId: string;
};
```

リクエスト型は、
以下とする。

```ts
export type UpdateMonthEndHoldingValueRequest = {
  value: number;
};
```

レスポンス型は、
以下とする。

```ts
export type UpdatedMonthEndHoldingValue = {
  holdingAssetId: string;
  value: number;
};

export type UpdateMonthEndHoldingValueResponse = {
  data: UpdatedMonthEndHoldingValue;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.patch<UpdateMonthEndHoldingValueResponse>(
    `/api/v1/month-end-asset-snapshots/${snapshotId}/holding-values/${holdingAssetId}`,
    {
      value: 900000,
    },
  );
```

商品別月末評価額入力画面などから、
登録済みの商品別月末評価額を
修正する際に利用する。

---

### 2.1 snapshotIdの扱い

`snapshotId`は、
API共通方針に従って
文字列として扱う。

```ts
const snapshotId: string = '12';
```

フロントエンド側で
数値へ変換して
業務計算には使用しない。

URL生成時も、
文字列のまま使用する。

---

### 2.2 holdingAssetIdの扱い

`holdingAssetId`は、
文字列として扱う。

```ts
const holdingAssetId: string = '5';
```

更新対象となる保有商品は、
パスパラメータで指定する。

```ts
const url =
  `/api/v1/month-end-asset-snapshots/${snapshotId}/holding-values/${holdingAssetId}`;
```

リクエストボディへ
`holdingAssetId`を
重複して含めない。

フロントエンド側では、
保有商品が操作対象利用者に
属しているかを
最終判定しない。

利用者境界は、
バックエンドで保証する。

---

### 2.3 valueの扱い

`value`は、
日本円の整数値として扱う。

```ts
const request: UpdateMonthEndHoldingValueRequest = {
  value: 900000,
};
```

0円への更新も許可する。

```ts
const request: UpdateMonthEndHoldingValueRequest = {
  value: 0,
};
```

以下のような
truthy / falsyによる
入力有無判定は行わない。

```ts
if (!value) {
  // value = 0も未入力扱いになるため使用しない
}
```

0円と未入力を
明確に区別する。

---

### 2.4 入力フォーム

HTMLのinput要素から取得する値は
文字列であるため、
API送信前に
数値へ変換する。

例：

```ts
const [valueInput, setValueInput] =
  useState('');

const value =
  valueInput === ''
    ? null
    : Number(valueInput);
```

未入力の場合は、
APIを実行しない。

```ts
if (value === null) {
  return;
}
```

整数であることも
必要に応じて確認する。

```ts
if (!Number.isInteger(value)) {
  return;
}
```

ただし、
最終的なバリデーションは
バックエンドで行う。

---

### 2.5 VAL-001との連携

VAL-001 商品別月末評価額一覧取得APIで
商品別月末評価額の
登録状態を確認する。

```text
value = null
    → 未登録

value !== null
    → 登録済み
```

VAL-003は、
`value !== null`の
登録済みデータに対して使用する。

```ts
if (item.value !== null) {
  // VAL-003で更新可能
}
```

`value = 0`も
登録済みであるため、
VAL-003の対象とする。

---

### 2.6 VAL-002との使い分け

商品別月末評価額の
登録状態によって、
使用するAPIを分ける。

```text
未登録
value = null
    ↓
VAL-002 商品別月末評価額登録

登録済み
value !== null
    ↓
VAL-003 商品別月末評価額更新
```

VAL-003を
upsert目的で使用しない。

商品別月末評価額が
未登録の場合に、
VAL-003の失敗を契機として
自動的にVAL-002へ切り替える処理は、
Phase1では行わない。

---

### 2.7 月末資産状況の確定状態

VAL-003を利用できるのは、
月末資産状況が
未確定の場合のみである。

編集可否を
画面上で補助的に制御する場合は、
SNP-003 月末資産状況詳細取得APIから
`confirmed`を取得する。

```ts
const canEdit =
  !snapshot.confirmed;
```

確定済みの場合は、
入力欄や保存ボタンを
非活性化してよい。

ただし、
更新可否の最終判断は
バックエンドのVAL-003で行う。

---

### 2.8 確定済みエラーの扱い

`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`
が返却された場合は、
商品別月末評価額を
更新できないことを表示する。

表示例：

```text
確定済みの月末資産状況は
編集できません。

編集する場合は、
先に確定を解除してください。
```

必要に応じて、
SNP-005 月末資産状況確定解除APIへの
導線を表示する。

---

### 2.9 商品別月末評価額不存在エラー

`MONTH_END_HOLDING_VALUE_NOT_FOUND`
が返却された場合は、
画面上では登録済みとして
表示されているデータが、
実際には存在しなくなっている可能性がある。

例えば、

```text
VAL-001取得
    ↓
value = 850000

別操作で状態変更
    ↓
現在は商品別月末評価額なし

古い画面状態からVAL-003実行
    ↓
404 Not Found
```

のようなケースが考えられる。

この場合は、
VAL-001を再取得して
最新状態を画面へ反映する。

自動的にVAL-002を実行して
新しいレコードを作成しない。

---

### 2.10 保有商品不存在エラー

`HOLDING_ASSET_NOT_FOUND`
が返却された場合は、
表示している保有商品の状態が
最新ではない可能性がある。

必要に応じて、
VAL-001を再取得する。

他の利用者に属する
保有商品であった場合も
同じエラーとなるため、
フロントエンドでは
不存在理由を推測しない。

---

### 2.11 資産口座の対象年月エラー

`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`
が返却された場合は、
対象年月時点では
その資産口座が
月末資産管理対象ではないことを表示する。

表示例：

```text
この保有商品の資産口座は、
対象年月の月末資産管理対象ではありません。
```

現在の資産口座状態だけを使用して、
フロントエンド側で
過去月の更新可否を
独自判定しない。

---

### 2.12 残高記録単位不一致の扱い

`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`
が返却された場合は、
対象資産口座が
商品単位の管理対象ではないことを表示する。

表示例：

```text
この資産口座は、
商品別評価額の更新対象ではありません。
```

口座単位の月末資産残高を
変更する場合は、
BAL-003 月末資産残高更新APIを使用する。

---

### 2.13 保有商品の対象年月エラー

`HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH`
が返却された場合は、
対象年月時点で
商品別月末評価額の
記録対象ではないことを表示する。

表示例：

```text
この保有商品は、
対象年月の商品別月末評価額の
更新対象ではありません。
```

必要に応じて、
VAL-001を再取得する。

---

### 2.14 同一値への更新

現在の値と
同じ値を送信しても、
正常な更新として扱う。

```ts
const currentValue = 900000;
const inputValue = 900000;
```

この場合も、
VAL-003を実行してよい。

```text
PATCH
value = 900000
    ↓
200 OK
```

フロントエンド側で
同一値更新を禁止する必要はない。

ただし、
不要なAPI通信を減らす目的で、

```ts
if (value === currentValue) {
  return;
}
```

のようなUI上の最適化を
行ってもよい。

この判定は、
業務ルールではなく
フロントエンド上の最適化として扱う。

---

### 2.15 0円への更新

0円への更新を
正常な操作として扱う。

```ts
await updateHoldingValue({
  value: 0,
});
```

以下のような判定によって
保存処理を止めてはならない。

```ts
if (!value) {
  return;
}
```

`0`と
未入力を必ず区別する。

---

### 2.16 更新処理中の画面制御

更新処理中は、
保存ボタンを非活性化する。

React Query等を使用する場合は、
Mutationの状態を利用する。

例：

```ts
const mutation =
  useMutation({
    mutationFn: updateMonthEndHoldingValue,
  });

const isSubmitting =
  mutation.isPending;
```

```tsx
<button
  type="submit"
  disabled={isSubmitting}
>
  保存
</button>
```

これにより、
利用者による
不要な連続クリックを抑止する。

VAL-003自体は冪等であるため、
二重送信によって
同じ値が設定されても
最終状態は同じとなる。

---

### 2.17 更新成功時の扱い

更新成功後は、
レスポンスの`value`を
画面へ反映できる。

```ts
const updatedValue =
  response.data.value;
```

例えば、

```text
更新前
value = 850000

    ↓ VAL-003

更新後
value = 900000
```

となる。

---

### 2.18 再取得

更新成功後は、
必要に応じて
VAL-001を再取得する。

React Query等を使用する場合は、
対象の`snapshotId`に対応する
一覧クエリをinvalidateしてよい。

```ts
queryClient.invalidateQueries({
  queryKey: [
    'monthEndHoldingValues',
    snapshotId,
  ],
});
```

これにより、
サーバー上の最新状態を
一覧へ反映できる。

---

### 2.19 Mutation実装例

React Queryを使用する場合の
概念例は、
以下とする。

```ts
type UpdateMonthEndHoldingValueArgs = {
  snapshotId: string;
  holdingAssetId: string;
  value: number;
};

export const updateMonthEndHoldingValue =
  async ({
    snapshotId,
    holdingAssetId,
    value,
  }: UpdateMonthEndHoldingValueArgs) => {
    const response =
      await apiClient.patch<UpdateMonthEndHoldingValueResponse>(
        `/api/v1/month-end-asset-snapshots/${snapshotId}/holding-values/${holdingAssetId}`,
        {
          value,
        },
      );

    return response.data;
  };
```

Hookの概念例：

```ts
export const useUpdateMonthEndHoldingValue =
  (
    snapshotId: string,
  ) => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn:
        updateMonthEndHoldingValue,

      onSuccess: async () => {
        await queryClient.invalidateQueries({
          queryKey: [
            'monthEndHoldingValues',
            snapshotId,
          ],
        });
      },
    });
  };
```

実際のAPI Clientおよび
React Queryの採用方針は、
フロントエンド共通設計に従う。

---

### 2.20 バリデーションエラーの扱い

`VALIDATION_ERROR`が返却された場合は、
`error.details`を利用して
対象項目へエラーを表示する。

本APIでは、
主に以下が対象となる。

```text
snapshotId
holdingAssetId
value
```

`value`については、
入力欄付近へ
メッセージを表示する。

例：

```ts
if (detail.field === 'value') {
  setFieldError(
    'value',
    detail.message,
  );
}
```

`snapshotId`または
`holdingAssetId`の形式不正は、
通常の画面操作では
発生しないことを前提とする。

発生した場合は、
不正な画面状態または
URLとして扱う。

---

### 2.21 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 入力項目または不正な画面状態としてエラー表示する |
| `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` | 月末資産状況一覧画面へ戻す |
| `MONTH_END_ASSET_SNAPSHOT_CONFIRMED` | 確定済みのため更新できないことを表示する |
| `HOLDING_ASSET_NOT_FOUND` | VAL-001を再取得し、対象商品が存在しないことを表示する |
| `MONTH_END_HOLDING_VALUE_NOT_FOUND` | VAL-001を再取得して未登録状態を反映する |
| `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH` | 対象年月では資産口座が利用できないことを表示する |
| `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH` | 口座単位の残高管理対象であることを表示する |
| `HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH` | 対象年月では評価額記録対象外であることを表示する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [VAL-001 商品別月末評価額一覧取得](./val-001-list.md)
- [VAL-002 商品別月末評価額登録](./val-002-create.md)
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