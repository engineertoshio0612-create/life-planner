# SNP-001 月末資産状況一覧取得

## 概要

操作対象となる利用者について、
月末資産状況の一覧を取得し、
月末資産状況一覧画面へ表示する。

一覧では、
対象年月ごとの以下の情報を扱う。

- 月末資産状況ID
- 対象年月
- 確定状態

確定済み・未確定の
両方を取得対象とする。

月末資産残高および
商品別月末評価額などの詳細情報は、
本APIでは取得しない。

React・TypeScriptでは、
APIレスポンスを以下の型として扱う。

```ts
export type MonthEndAssetSnapshotSummary = {
  id: string;
  targetYearMonth: string;
  confirmed: boolean;
};

export type GetMonthEndAssetSnapshotsResponse = {
  data: MonthEndAssetSnapshotSummary[];
};
```

`targetYearMonth` は、
`YYYY-MM` 形式の年月を表す
業務値として文字列で保持する。

JavaScriptの `Date` へ
不必要に変換せず、
表示時のみ必要に応じて
`YYYY年M月` などの形式へ変換する。

`confirmed` は、
月末資産状況の確定状態を表す。

```text
true  → 確定済み
false → 未確定
```

フロントエンドでは、
確定状態に応じて
表示や操作導線を制御してよい。

ただし、
確定可否および確定解除可否を
`confirmed` のみから最終判断せず、
実行時にはバックエンドAPIで
業務ルールを再検証する。

一覧は、
APIから対象年月の降順で返却されるため、
フロントエンドでは
原則として取得順をそのまま表示する。

同じ並び替えルールを
フロントエンドへ重複実装しない。

月末資産状況が存在しない場合は、
`data: []` を正常レスポンスとして扱い、
エラー表示は行わない。

必要に応じて、

```text
月末資産状況がまだ登録されていません。
```

などの空状態表示や、
月末資産状況作成への導線を表示する。

一覧取得中は
ローディング状態を表示する。

利用者を切り替えた場合は、
以前の利用者の一覧を
新しい利用者の情報として表示しない。

以前のデータを破棄するか、
新しい取得が完了するまで
ローディング状態を表示する。

一覧取得エラーは、
APIの独自エラーコードに応じて
共通エラー表示や
利用者選択画面への遷移を行う。

Phase1では、
フロントエンド独自の
長期キャッシュは前提としない。

以下の操作後は、
月末資産状況一覧を再取得する。

- 月末資産状況作成
- 月末資産状況確定
- 月末資産状況確定解除

TanStack Query等を使用する場合は、
対象クエリをinvalidateし、
最新の一覧を再取得する。

各一覧行からは、
月末資産状況詳細、
月末資産残高、
商品別月末評価額、
確定・確定解除などの
関連操作へ遷移できるようにする。

---

## 2. React・TypeScriptでの利用

レスポンス型は、
以下とする。

```ts
export type MonthEndAssetSnapshotSummary = {
  id: string;
  targetYearMonth: string;
  confirmed: boolean;
};

export type GetMonthEndAssetSnapshotsResponse = {
  data: MonthEndAssetSnapshotSummary[];
};
```

API呼び出し例は、以下とする。

```typescript
const response =
  await apiClient.get<GetMonthEndAssetSnapshotsResponse>(
    '/api/v1/month-end-asset-snapshots',
  );
```

取得した一覧は、月末資産状況一覧画面へ表示する。

各月の一覧行から、以下の操作へ遷移できる。

- 月末資産状況詳細表示
- 月末資産残高の登録・更新
- 商品別月末評価額の登録・更新
- 月末資産状況の確定
- 月末資産状況の確定解除

---

### 2.1 targetYearMonthの扱い

`targetYearMonth`は、API共通方針に従い`YYYY-MM`形式で返却される。

```typescript
const targetYearMonth = snapshot.targetYearMonth;
```

対象年月は、日付ではなく年月を表す業務値として扱う。

そのため、JavaScriptの`Date`オブジェクトへ不必要に変換せず、文字列として保持する。

表示時は、必要に応じて画面向けの表記へ変換する。

例：

```typescript
const formatYearMonth = (
  value: string,
): string => {
  const [year, month] = value.split('-');

  return `${year}年${Number(month)}月`;
};
```

---

### 2.2 confirmedの扱い

`confirmed`は、月末資産状況の確定状態を表す。

| confirmed | 表示例 |
| --- | --- |
| `true` | 確定済み |
| `false` | 未確定 |

```typescript
const statusLabel =
  snapshot.confirmed
    ? '確定済み'
    : '未確定';
```

フロントエンドでは、確定状態に応じて表示上の制御を行ってよい。

ただし、確定可否および確定解除可否を`confirmed`だけから最終判断しない。

確定および確定解除の実行時は、それぞれのAPIでバックエンド側の業務ルールを再検証する。

---

### 2.3 一覧順の扱い

APIレスポンスは、対象年月の降順で返却される。

フロントエンドでは、原則としてAPIが返却した順序をそのまま表示する。

一覧表示のために同じ並び替え処理をフロントエンドへ重複実装しない。

---

### 2.4 データなしの扱い

月末資産状況が存在しない場合は、以下が返却される。

```json
{
  "data": []
}
```

フロントエンドでは、エラーとして扱わない。

例えば、以下のような案内を表示する。

```text
月末資産状況がまだ登録されていません。
```

必要に応じて、月末資産状況作成画面への導線を表示する。

---

### 2.5 ローディング表示

一覧取得中は、ローディング状態を表示する。

取得完了前に、以前選択していた別利用者の月末資産状況を現在の利用者の情報として表示しない。

利用者を切り替えた場合は、以前の一覧データを破棄するか、新しい取得が完了するまでローディング状態を表示する。

---

### 2.6 エラー表示

エラーコードごとの基本的な扱いは、以下とする。

| エラーコード | フロントエンドの扱い |
| --- | --- |
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

月末資産状況が0件の場合は、エラー表示を行わない。

---

### 2.7 キャッシュ・再取得

Phase1では、フロントエンド独自の長期キャッシュは前提としない。

以下の操作後は、一覧を再取得する。

- 月末資産状況作成
- 月末資産状況確定
- 月末資産状況確定解除

これにより、確定状態や新しく作成された対象年月を一覧へ反映する。

React Query等を使用する場合は、該当するクエリをinvalidateして再取得してよい。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [SNP-002 月末資産状況作成](./snp-002-create.md)
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