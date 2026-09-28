# SNP-003 月末資産状況詳細取得

## 概要

操作対象となる利用者について、
指定した月末資産状況の
詳細情報を取得する。

取得対象には、
確定済み・未確定の
両方の月末資産状況を含む。

本APIでは、
以下の情報を取得する。

- 月末資産状況ID
- 対象年月
- 確定状態

月末資産残高および
商品別月末評価額などの明細は、
本APIでは取得しない。

React・TypeScriptでは、
月末資産状況IDを
文字列として扱う。

```ts
export type GetMonthEndAssetSnapshotParams = {
  snapshotId: string;
};
```

レスポンスは、
以下の型として扱う。

```ts
export type MonthEndAssetSnapshotDetail = {
  id: string;
  targetYearMonth: string;
  confirmed: boolean;
};

export type GetMonthEndAssetSnapshotResponse = {
  data: MonthEndAssetSnapshotDetail;
};
```

`snapshotId` は、
API共通方針に従って
文字列のまま保持し、
数値計算には使用しない。

URL生成時も、
文字列としてそのまま利用する。

`targetYearMonth` は、
`YYYY-MM` 形式の年月を表す
業務値として扱う。

JavaScriptの `Date` へ
不必要に変換せず、
原則として文字列で保持する。

表示時のみ必要に応じて、
`YYYY年M月` などの
画面向け表記へ変換する。

`confirmed` は、
月末資産状況の確定状態を表す。

```text
true  → 確定済み
false → 未確定
```

フロントエンドでは、
確定状態に応じて、
編集可能・編集不可などの
補助的な表示制御を行ってよい。

ただし、
確定可否および
確定解除可否の最終判断は、
フロントエンドでは行わない。

実行時には、
SNP-004 月末資産状況確定API、
SNP-005 月末資産状況確定解除APIで
バックエンド側の業務ルールを再検証する。

月末資産残高が必要な場合は、
BAL-001 月末資産残高一覧取得APIを使用する。

商品別月末評価額が必要な場合は、
VAL-001 商品別月末評価額一覧取得APIを使用する。

月末資産状況詳細、
月末資産残高、
商品別月末評価額は、
それぞれ別のAPIレスポンスとして扱い、
本APIに関連明細が含まれることを前提としない。

詳細取得中は、
ローディング状態を表示する。

`snapshotId` が変更された場合は、
以前の月末資産状況を
新しい対象として表示しない。

以前の取得結果を破棄するか、
新しい取得が完了するまで
ローディング状態を表示する。

`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
が返却された場合は、
月末資産状況一覧画面へ戻すなどして、
対象が存在しないことを表示する。

この場合、
対象が存在しないケースと
他利用者に属しているケースを
フロントエンドでは区別しない。

月末資産状況の確定または
確定解除後に
同じ詳細画面へ戻る場合は、
詳細情報を再取得する。

TanStack Query等を使用する場合は、
該当する詳細クエリをinvalidateし、
最新の `confirmed` 状態を
画面へ反映する。

---

## 2. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type GetMonthEndAssetSnapshotParams = {
  snapshotId: string;
};
```
レスポンス型は、以下とする。

```ts
export type MonthEndAssetSnapshotDetail = {
  id: string;
  targetYearMonth: string;
  confirmed: boolean;
};

export type GetMonthEndAssetSnapshotResponse = {
  data: MonthEndAssetSnapshotDetail;
};
```

API呼び出し例は、以下とする。

```ts
const response =
  await apiClient.get<GetMonthEndAssetSnapshotResponse>(
    `/api/v1/month-end-asset-snapshots/${snapshotId}`,
  );
```

取得した月末資産状況は、以下の画面や処理で利用する。

- 月末資産状況詳細画面
- 月末資産残高一覧画面への遷移
- 商品別月末評価額一覧画面への遷移
- 月末資産状況確定前の確認
- 月末資産状況確定解除前の確認

---

### 2.1 snapshotIdの扱い

`snapshotId`は、API共通方針に従って文字列として扱う。

```ts
const snapshotId: string = '12';
```

数値へ変換してフロントエンド側で計算には使用しない。

URL生成時も、文字列のまま使用する。

---

### 2.2 targetYearMonthの扱い

`targetYearMonth`は、`YYYY-MM`形式で返却される。

```ts
const targetYearMonth =
  response.data.targetYearMonth;
```

対象年月は、日付ではなく年月を表す業務値として扱う。

JavaScriptの`Date`オブジェクトへ不必要に変換せず、原則として文字列で保持する。

画面表示時は、必要に応じて以下のような形式へ変換してよい。

```ts
const formatYearMonth = (
  value: string,
): string => {
  const [year, month] = value.split('-');

  return `${year}年${Number(month)}月`;
};
```

---

### 2.3 confirmedの扱い

`confirmed`は、月末資産状況の確定状態を表す。

| confirmed | 状態 |
| --- | --- |
| `true` | 確定済み |
| `false` | 未確定 |

表示例：

```ts
const confirmedLabel =
  response.data.confirmed
    ? '確定済み'
    : '未確定';
```

フロントエンドでは、確定状態に応じて補助的な表示制御を行ってよい。

例えば、未確定の場合は残高入力画面への導線を表示し、確定済みの場合は編集できないことを表示する。

ただし、確定可否および確定解除可否の最終判断はフロントエンドで行わない。

---

### 2.4 月末資産残高の取得

本APIでは、月末資産残高の明細を返却しない。

必要な場合は、BAL-001 月末資産残高一覧取得APIを使用する。

```ts
await apiClient.get(
  `/api/v1/month-end-asset-snapshots/${snapshotId}/asset-balances`,
);
```

本APIのレスポンスへ月末資産残高が含まれることを前提としない。

---

### 2.5 商品別月末評価額の取得

本APIでは、商品別月末評価額の明細を返却しない。

必要な場合は、VAL-001 商品別月末評価額一覧取得APIを使用する。

```ts
await apiClient.get(
  `/api/v1/month-end-asset-snapshots/${snapshotId}/holding-values`,
);
```

月末資産状況詳細と商品別月末評価額は、別APIとして取得する。

---

### 2.6 ローディング表示

詳細取得中は、ローディング状態を表示する。

取得完了前に、以前表示していた別の月末資産状況を現在の対象として表示しない。

`snapshotId`が変更された場合は、以前の取得結果を破棄するか、新しい取得が完了するまでローディング状態を表示する。

---

### 2.7 エラー表示

エラーコードごとの基本的な扱いは、以下とする。

| エラーコード | フロントエンドの扱い |
| --- | --- |
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 不正なURLまたはパラメータとして扱う |
| `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` | 月末資産状況一覧画面へ戻し、対象が存在しないことを表示する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`の場合は、以下の原因をフロントエンドで区別しない。

- 月末資産状況が存在しない
- 他の利用者に属している

---

### 2.8 再取得

以下の操作後に同じ月末資産状況詳細画面へ戻る場合は、本APIを再取得する。

- 月末資産状況確定
- 月末資産状況確定解除

これにより、最新の`confirmed`状態を画面へ反映する。

React Query等を使用する場合は、該当する詳細クエリをinvalidateしてよい。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [SNP-001 月末資産状況一覧取得](./snp-001-list.md)
- [SNP-002 月末資産状況作成](./snp-002-create.md)
- [SNP-004 月末資産状況確定](./snp-004-confirm.md)
- [SNP-005 月末資産状況確定解除](./snp-005-unconfirm.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/month-end-asset-snapshots/README.md)
- [Reactアーキテクチャ設計](../../../architecture/react/month-end-asset-snapshots/README.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)