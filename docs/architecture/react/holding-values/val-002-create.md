# VAL-002 商品別月末評価額登録

## 1. 概要

操作対象となる利用者について、
指定した月末資産状況に紐づく
商品別月末評価額を新規登録する。

登録対象は、
残高記録単位が商品単位である資産口座に属する
保有商品の未登録の商品別月末評価額とする。

React・TypeScript側では、
`VAL-001 商品別月末評価額一覧取得`で取得した状態をもとに、
未登録の商品別月末評価額に対して
VAL-002を実行する。

登録時には、
`snapshotId`、
`holdingAssetId`、
`value`
を使用する。

`value`は日本円の整数値として扱い、
`0`も有効な登録値とする。
未入力と0円を区別し、
truthy / falsyによる判定は行わない。

商品別月末評価額が
すでに登録されている場合は、
VAL-002による上書きは行わず、
既存値の変更には
VAL-003 商品別月末評価額更新APIを使用する。

また、
月末資産状況が確定済みの場合は
登録UIを非活性化してよいが、
登録可否の最終判断はバックエンドに委ねる。

登録成功後は、
対象 `snapshotId` のVAL-001を再取得し、
最新の商品別月末評価額一覧を
画面へ反映することを基本とする。

利用者境界、
対象年月時点の資産口座・保有商品の利用可否、
残高記録単位、
重複登録などの業務ルールについては、
フロントエンドで最終判定せず、
バックエンドの判定結果に従う。

口座単位で残高を記録する資産口座は
VAL-002の対象外とし、
月末資産残高の登録には
BAL-002 月末資産残高登録APIを使用する。

---

## 2. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type CreateMonthEndHoldingValueParams = {
  snapshotId: string;
};
```

リクエスト型は、
以下とする。

```ts
export type CreateMonthEndHoldingValueRequest = {
  holdingAssetId: string;
  value: number;
};
```

レスポンス型は、
以下とする。

```ts
export type CreatedMonthEndHoldingValue = {
  holdingAssetId: string;
  value: number;
};

export type CreateMonthEndHoldingValueResponse = {
  data: CreatedMonthEndHoldingValue;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.post<CreateMonthEndHoldingValueResponse>(
    `/api/v1/month-end-asset-snapshots/${snapshotId}/holding-values`,
    {
      holdingAssetId,
      value: 850000,
    },
  );
```

商品別月末評価額入力画面などから
未登録の商品別月末評価額を
新規登録する際に利用する。

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

登録対象となる保有商品は、
リクエストボディで指定する。

```ts
const request: CreateMonthEndHoldingValueRequest = {
  holdingAssetId,
  value: 850000,
};
```

フロントエンド側で
保有商品の所有者境界を
最終判定しない。

登録可能かどうかは、
バックエンドで
操作対象利用者、
資産口座、
対象年月との関係を確認する。

---

### 2.3 valueの扱い

`value`は、
日本円の整数値として扱う。

```ts
const request: CreateMonthEndHoldingValueRequest = {
  holdingAssetId: '5',
  value: 850000,
};
```

0円も有効な登録値とする。

```ts
const request: CreateMonthEndHoldingValueRequest = {
  holdingAssetId: '5',
  value: 0,
};
```

以下のような
truthy / falsyによる
入力有無の判定は行わない。

```ts
if (!value) {
  // value = 0 も未入力扱いになるため使用しない
}
```

入力有無は、
`null`、
`undefined`、
空文字列などを
明示的に判定する。

---

### 2.4 入力フォーム

商品別月末評価額入力フォームでは、
利用者が入力した値を
integerへ変換してから
APIへ送信する。

HTMLの入力値は
文字列として取得されるため、
空文字列と0円を
明確に区別する。

例：

```ts
const [valueInput, setValueInput] =
  useState('');

const value =
  valueInput === ''
    ? null
    : Number(valueInput);
```

API呼び出し前に、
`value === null`でないことを確認する。

```ts
if (value === null) {
  return;
}
```

`Number()`による変換結果についても、
必要に応じて
整数であることを確認する。

```ts
if (!Number.isInteger(value)) {
  return;
}
```

ただし、
最終的な入力値検証は
バックエンドで行う。

---

### 2.5 VAL-001との連携

VAL-001 商品別月末評価額一覧取得APIで
`value = null`となっている保有商品を、
VAL-002の登録対象とする。

```ts
if (item.value === null) {
  // VAL-002で新規登録
}
```

`value = 0`は
登録済みであるため、
VAL-002を使用しない。

```ts
if (item.value !== null) {
  // VAL-003で更新
}
```

以下の判定は行わない。

```ts
if (!item.value) {
  // value = 0を未登録と誤判定するため使用しない
}
```

---

### 2.6 登録・更新APIの切り替え

商品別月末評価額の状態によって、
使用するAPIを切り替える。

```text
value = null
    ↓
VAL-002 商品別月末評価額登録

value !== null
    ↓
VAL-003 商品別月末評価額更新
```

フロントエンドでは、
VAL-002をupsert目的で使用しない。

登録後の評価額を変更する場合は、
VAL-003を使用する。

---

### 2.7 月末資産状況の確定状態

VAL-002を利用できるのは、
月末資産状況が
未確定の場合のみである。

編集可否を
画面上で補助的に制御する場合は、
SNP-003 月末資産状況詳細取得APIから
確定状態を取得する。

```ts
const canEdit =
  !snapshot.confirmed;
```

`confirmed = true`の場合は、
登録フォームや
保存ボタンを非活性化してよい。

ただし、
登録可否の最終判断は
VAL-002で行う。

---

### 2.8 確定済みエラーの扱い

月末資産状況が
確定済みの場合は、

`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`

が返却される。

フロントエンドでは、
登録できないことを表示する。

表示例：

```text
確定済みの月末資産状況には
商品別月末評価額を登録できません。

編集する場合は、
先に確定を解除してください。
```

必要に応じて、
SNP-005 月末資産状況確定解除APIを
利用する導線を表示する。

---

### 2.9 保有商品不存在エラーの扱い

`HOLDING_ASSET_NOT_FOUND`
が返却された場合は、
表示している保有商品情報が
最新ではない可能性がある。

そのため、
VAL-001を再取得し、
現在の商品別月末評価額一覧を
更新してよい。

他の利用者に属する
保有商品であった場合も
同じエラーとなるため、
フロントエンドでは
存在理由を推測しない。

---

### 2.10 資産口座の対象年月エラー

保有商品が属する資産口座が
対象年月時点で
月末資産管理対象ではない場合は、

`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`

が返却される。

表示例：

```text
この保有商品の資産口座は、
対象年月の月末資産管理対象ではありません。
```

フロントエンド側で
現在の資産口座状態だけを使用して
過去月の登録可否を
独自に判定しない。

---

### 2.11 残高記録単位不一致の扱い

指定した保有商品が属する
資産口座の残高記録単位が
商品単位ではない場合は、

`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`

が返却される。

表示例：

```text
この資産口座は、
商品別評価額の登録対象ではありません。
```

口座単位の資産口座については、
BAL-002 月末資産残高登録APIを使用する。

---

### 2.12 保有商品の対象年月エラー

指定した保有商品が
対象年月時点で
評価額記録対象ではない場合は、

`HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH`

が返却される。

表示例：

```text
この保有商品は、
対象年月の商品別月末評価額の
登録対象ではありません。
```

エラー発生後は、
必要に応じてVAL-001を再取得し、
最新の一覧状態を表示する。

---

### 2.13 重複登録エラーの扱い

商品別月末評価額が
すでに登録されている場合は、

`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`

が返却される。

これは、
例えば以下のような場合に発生し得る。

```text
VAL-001取得
    ↓
value = null

別タブ・別リクエストで登録
    ↓
現在は登録済み

古い画面状態のままVAL-002実行
    ↓
409 Conflict
```

この場合は、
VAL-001を再取得し、
最新状態を画面へ反映する。

VAL-002失敗後に
自動的にVAL-003へ切り替えて
同じ値を更新する処理は、
Phase1では行わない。

---

### 2.14 登録処理中の画面制御

登録処理中は、
保存ボタンを非活性化する。

```ts
const [isSubmitting, setIsSubmitting] =
  useState(false);
```

登録開始時に
`isSubmitting = true`とし、
成功または失敗後に
`false`へ戻す。

これにより、
利用者による
意図しない連続クリックを抑止する。

ただし、
二重登録防止の最終的な保証は、
バックエンドおよび
データベースのUNIQUE制約で行う。

---

### 2.15 登録成功時の扱い

登録成功後は、
レスポンスの`value`を
画面へ反映する。

```ts
const createdValue =
  response.data.value;
```

登録前：

```text
value = null
```

登録後：

```text
value = 850000
```

となる。

登録成功後は、
VAL-003による更新対象として
扱える。

---

### 2.16 再取得

登録成功後は、
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
登録後の最新状態を
一覧へ反映できる。

---

### 2.17 バリデーションエラーの扱い

`VALIDATION_ERROR`が返却された場合は、
`error.details.field`を利用して
対象項目へエラーを表示する。

本APIでは、
主に以下が対象となる。

```text
holdingAssetId
value
snapshotId
```

`value`については、
入力項目付近へ
エラーメッセージを表示する。

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
画面状態または
保持しているIDが不正な状態として扱う。

---

### 2.18 エラー表示

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
| `MONTH_END_ASSET_SNAPSHOT_CONFIRMED` | 確定済みのため登録できないことを表示する |
| `HOLDING_ASSET_NOT_FOUND` | VAL-001を再取得し、対象商品が存在しないことを表示する |
| `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH` | 対象年月では資産口座が利用できないことを表示する |
| `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH` | 口座単位の残高管理対象であることを表示する |
| `HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH` | 対象年月では評価額記録対象外であることを表示する |
| `MONTH_END_HOLDING_VALUE_ALREADY_EXISTS` | VAL-001を再取得して登録済み状態を反映する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [VAL-001 商品別月末評価額一覧取得](./val-001-list.md)
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
