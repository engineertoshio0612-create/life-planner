# SNP-002 月末資産状況作成

## 概要

操作対象となる利用者について、
指定した対象年月の
月末資産状況を作成する。

月末資産状況は、
対象年月ごとの月末資産残高および
商品別月末評価額を管理するための
親リソースとして扱う。

本APIでは、
月末資産状況のみを作成し、
月末資産残高および
商品別月末評価額は登録しない。

作成直後の月末資産状況は、
必ず未確定状態とする。

React・TypeScriptでは、
登録リクエストとして
`targetYearMonth` のみを送信する。

```ts
export type CreateMonthEndAssetSnapshotRequest = {
  targetYearMonth: string;
};
```

レスポンスでは、
以下の月末資産状況を受け取る。

```ts
export type MonthEndAssetSnapshot = {
  id: string;
  targetYearMonth: string;
  confirmed: boolean;
};
```

`targetYearMonth` は、
`YYYY-MM` 形式の年月を表す
業務値として扱う。

JavaScriptの `Date` へ
不必要に変換せず、
原則として文字列で保持する。

`confirmed` は
フロントエンドから送信しない。

作成直後は必ず `false` となり、
フロントエンドから
確定済み状態を指定して
直接作成することはできない。

確定処理は、
SNP-004 月末資産状況確定APIを使用する。

作成成功後は、
返却された月末資産状況IDを使用して、
以下の画面や処理へ遷移できる。

- 月末資産状況詳細画面
- 月末資産残高登録画面
- 商品別月末評価額登録画面
- 月末資産状況一覧画面

作成直後は、
月末資産残高や
商品別月末評価額が
未登録である可能性があるため、
作成成功のみを理由として
確定済みの資産状況として表示しない。

送信中は、
登録ボタンを非活性化するなどして、
フロントエンド側でも
意図しない二重送信を防止する。

ただし、
最終的な重複登録防止は
バックエンドおよび
データベース制約に委ねる。

同一対象年月の
月末資産状況がすでに存在する場合は、

`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`

を受け取り、
対象年月入力欄または
画面上部へ重複エラーを表示する。

`VALIDATION_ERROR` の場合は、
`error.details.field` を参照し、
`targetYearMonth` に対応する
入力エラーを表示する。

利用者コンテキストや
サーバーエラーについては、
API共通方針に従って
利用者選択への誘導または
共通エラー表示を行う。

作成成功後に
月末資産状況一覧へ戻る場合は、
一覧を再取得する。

TanStack Query等を使用する場合は、
月末資産状況一覧のクエリを
invalidateして、
作成した対象年月を
最新の一覧へ反映する。

---

## 2. React・TypeScriptでの利用

登録リクエスト型は、以下とする。

```ts
export type CreateMonthEndAssetSnapshotRequest = {
  targetYearMonth: string;
};
```

レスポンス型は、以下とする。

```ts
export type MonthEndAssetSnapshot = {
  id: string;
  targetYearMonth: string;
  confirmed: boolean;
};

export type CreateMonthEndAssetSnapshotResponse = {
  data: MonthEndAssetSnapshot;
};
```

API呼び出し例は、以下とする。

```ts
const response =
  await apiClient.post<CreateMonthEndAssetSnapshotResponse>(
    '/api/v1/month-end-asset-snapshots',
    {
      targetYearMonth: '2026-08',
    },
  );
```

登録成功後は、作成された月末資産状況を利用して、以下の画面または処理へ遷移できる。

- 月末資産状況詳細画面
- 月末資産残高登録画面
- 商品別月末評価額登録画面
- 月末資産状況一覧画面

---

### 2.1 targetYearMonthの扱い

`targetYearMonth`は、API共通方針に従い`YYYY-MM`形式で送信する。

```ts
const request: CreateMonthEndAssetSnapshotRequest = {
  targetYearMonth: '2026-08',
};
```

対象年月は、日付ではなく年月を表す業務値として扱う。

JavaScriptの`Date`オブジェクトへ不必要に変換せず、原則として文字列で保持する。

---

### 2.2 confirmedの扱い

`confirmed`は、レスポンスでのみ取得する。

登録リクエストでは、フロントエンドから送信しない。

月末資産状況作成直後は、必ず`false`で返却される。

```ts
if (!response.data.confirmed) {
  // 未確定の月末資産状況
}
```

フロントエンドから`confirmed = true`を指定して確定済みデータを直接作成してはならない。

確定処理は、SNP-004 月末資産状況確定APIを使用する。

---

### 2.3 登録後の画面制御

月末資産状況作成後は、返却された月末資産状況IDを利用して対象年月の資産入力画面へ遷移できる。

```ts
const snapshotId = response.data.id;
```

例えば、月末資産状況詳細画面へ遷移する。

```ts
navigate(
  `/month-end-assets/${snapshotId}`,
);
```

作成直後は、月末資産残高および商品別月末評価額がまだ登録されていない可能性がある。

そのため、作成成功だけを理由として確定済みの資産状況として表示しない。

---

### 2.4 二重送信防止

作成処理中は、登録ボタンを非活性化する。

```ts
const [isSubmitting, setIsSubmitting] =
  useState(false);
```

リクエスト送信開始時に`isSubmitting = true`とし、処理完了後またはエラー発生後に`false`へ戻す。

これにより、意図しない二重送信を防止する。

ただし、最終的な重複登録防止はバックエンドおよびデータベースのUNIQUE制約で保証する。

---

### 2.5 重複エラーの扱い

指定した対象年月の月末資産状況がすでに存在する場合は、`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`が返却される。

フロントエンドでは、対象年月入力欄または画面上部へ重複エラーを表示する。

表示例：

```text
指定した対象年月の月末資産状況は
すでに作成されています。
```

必要に応じて、既存の月末資産状況一覧または詳細画面への導線を表示してよい。

---

### 2.6 バリデーションエラーの扱い

`VALIDATION_ERROR`が返却された場合は、`error.details.field`を利用して対象項目へエラーメッセージを表示する。

対象となるフィールドは、以下とする。

- `targetYearMonth`

```ts
if (
  detail.field === 'targetYearMonth'
) {
  setFieldError(
    'targetYearMonth',
    detail.message,
  );
}
```

---

### 2.7 エラー表示

エラーコードごとの基本的な扱いは、以下とする。

| エラーコード | フロントエンドの扱い |
| --- | --- |
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS` | 対象年月の重複エラーを表示する |
| `VALIDATION_ERROR` | 入力項目ごとにエラーを表示する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

### 2.8 一覧再取得

月末資産状況の作成成功後に一覧画面へ戻る場合は、月末資産状況一覧を再取得する。

React Query等を使用する場合は、月末資産状況一覧のクエリをinvalidateしてよい。

これにより、作成した対象年月を一覧へ反映する。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [SNP-001 月末資産状況一覧取得](./snp-001-list.md)
- [SNP-003 月末資産状況詳細取得](./snp-003-detail.md)
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