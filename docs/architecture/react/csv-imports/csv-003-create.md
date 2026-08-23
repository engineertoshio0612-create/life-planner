##  CSV-003 月末資産残高CSV登録

### 1 概要

CSV-003は、CSV-002 月末資産残高CSVプレビューで登録可能と確認した月末資産残高CSVをバックエンドへ送信し、月末資産残高を一括登録するためのAPIである。

React・TypeScriptでは、CSV-002でプレビューに使用した同一の`File`を登録完了まで保持し、利用者が登録操作を行ったタイミングでCSV-003へ再送する。

```text id="3vmn25"
CSVファイル選択
    ↓
CSV-002
プレビュー
    ↓
canImport確認
    ├─ false
    │     ↓
    │   エラー表示
    │
    └─ true
          ↓
       登録ボタン有効化
          ↓
       利用者が登録実行
          ↓
       CSV-003
          ↓
       最新状態で再検証
          ↓
       一括登録
```

CSV-003へはCSV-002のプレビュー結果から再構築したデータを送信せず、利用者が選択したCSVファイルそのものを`FormData`へ設定して送信する。`targetYearMonth`、`balance`、`canImport`、`previewId`などの業務情報をフロントエンドから追加送信しない。

CSV-003は業務データを更新するAPIであるため、TanStack Queryを使用する場合はMutationとして扱う。登録処理中は登録ボタンを無効化し、通常操作による二重送信を防止する。

登録ボタンは、少なくとも以下の条件を満たす場合に有効化する。

```text id="dbwddq"
file != null
AND
preview != null
AND
preview.canImport = true
AND
CSV-003実行中ではない
```

ただし、CSV-002で取得した`canImport = true`はCSV-003の成功を保証するものではない。CSV-002とCSV-003の間に月末資産状況の確定、既存月末資産残高の登録、資産口座の状態変更などが発生する可能性があるため、CSV-003では実行時点の最新状態をバックエンドで再検証する。

```text id="rm6i3w"
CSV-002
canImport = true
    ↓
時間経過・業務状態変更
    ↓
CSV-003
    ↓
最新状態で再解析・再検証
    ├─ 登録可能
    │     ↓
    │   一括登録
    │
    └─ 登録不可
          ↓
       409 / 422等
```

CSV-003が成功した場合は、レスポンスとして返却される`targetYearMonth`および`importedCount`を完了メッセージや画面遷移に利用できる。登録件数をCSV行数からフロントエンドで再計算する必要はない。

登録成功後は、保持していた`File`およびCSV-002のプレビュー結果をクリアし、登録済みCSVに対する古い状態を画面へ残さない。また、月末資産残高や月末資産状況など、登録によって影響を受けるTanStack QueryのQuery Cacheをinvalidateし、必要なデータをバックエンドから再取得する。

CSV-003のレスポンスには登録された各資産口座・月末残高の詳細は含まれないため、登録後の一覧をレスポンスやCSVファイルからReact側で再構築しない。最新状態が必要な場合は、BAL-001などの参照APIを再取得する。

CSV-003が`409 Conflict`や`422 Unprocessable Entity`で失敗した場合は、CSV-002の古いプレビュー結果だけを根拠に登録可能と判断し続けない。業務状態の変化による失敗では、必要に応じてプレビューを無効化し、CSV-002を再実行できるようにする。

また、通信エラー発生時はサーバー側で登録が完了している可能性があるため、同じCSVを無条件に再送することだけを復旧手段としない。必要に応じて参照APIを再取得し、現在の登録状態を確認する。

Phase1では、CSVの正式な解析、入力値検証、既存データ確認、重複判定などをReact側へ重複実装しない。フロントエンドはCSV-002の結果を登録操作の制御に利用し、CSV-003実行時の最終的な登録可否および業務整合性の保証はバックエンドへ集約する。


---

### 2 React・TypeScriptでの利用

CSV-003は、
CSV-002 月末資産残高CSVプレビューで
登録可能と確認したCSVファイルを
実際に登録するために使用する。

フロントエンドでは、
CSV-002で使用した
同一の`File`を保持し、
利用者が登録操作を行ったタイミングで
CSV-003へ再送する。

概念的な利用フローは、
以下とする。

```text
CSVファイル選択
    ↓
CSV-002
プレビュー
    ↓
canImport確認
    ├─ false
    │     ↓
    │   エラー表示
    │
    └─ true
          ↓
       登録ボタン有効化
          ↓
       利用者が登録実行
          ↓
       CSV-003
          ↓
       成功
          ↓
       関連Query再取得
```

CSV-002の
`canImport = true`は、
CSV-003の登録成功を
保証するものではない。

CSV-003では、
実行時点の最新状態をもとに
再検証される。

---

#### 2.1 TypeScript型

CSV登録結果は、
以下のような型として扱う。

概念例：

```ts
export type MonthEndAssetBalanceCsvImportResult = {
  targetYearMonth: string;
  importedCount: number;
};

export type ImportMonthEndAssetBalanceCsvResponse = {
  data: MonthEndAssetBalanceCsvImportResult;
  requestId: string;
};
```

API共通Envelopeの
共通型が存在する場合は、
以下のように
共通型を使用する。

```ts
export type ImportMonthEndAssetBalanceCsvResponse =
  ApiResponse<MonthEndAssetBalanceCsvImportResult>;
```

---

#### 2.2 Fileの保持

CSV-002で
プレビューしたCSVファイルは、
CSV-003実行まで
`File`として保持する。

概念例：

```ts
const [file, setFile] =
  useState<File | null>(
    null,
  );
```

CSV-002の
プレビュー結果から
新しいCSVファイルを
生成し直さない。

---

#### 2.3 同じFileを使用する

CSV-003では、
CSV-002で使用した
同じ`File`を送信する。

```text
CSV-002
File A
    ↓
canImport = true
    ↓
CSV-003
File A
```

以下のように、
プレビュー結果の

- `rows`
- `targetYearMonth`
- `balance`

などから
CSVを再構築しない。

正式な登録対象は、
利用者が選択した
CSVファイルそのものとする。

---

#### 2.4 FormData

CSVファイルは、
`FormData`へ設定する。

概念例：

```ts
const formData =
  new FormData();

formData.append(
  'file',
  file,
);
```

項目名は、

```text
file
```

とする。

以下の業務項目を
追加送信しない。

- `userId`
- `targetYearMonth`
- `assetAccountId`
- `balance`
- `confirmed`
- `previewId`
- `canImport`

---

#### 2.5 Content-Type

CSV-002と同様に、
`multipart/form-data`の
`Content-Type`は、
ブラウザまたはHTTP Clientへ
設定させる。

以下のような
手動設定は行わない。

```ts
headers: {
  'Content-Type':
    'multipart/form-data',
}
```

`FormData`を送信し、
boundaryを
HTTP Client側で生成させる。

---

#### 2.6 X-User-Id

`X-User-Id`は、
通常のAPIと同様に
共通API Clientから付与する。

CSV-003専用処理で
`userId`を
`FormData`へ追加しない。

概念例：

```ts
apiClient.interceptors.request.use(
  (config) => {
    config.headers[
      'X-User-Id'
    ] = currentuserId;

    return config;
  },
);
```

CSV-002実行時の
利用者コンテキストを
サーバー側で引き継ぐ前提にしない。

---

#### 2.7 API Client

CSV登録APIは、
CSVファイルを受け取る
専用関数として定義する。

概念例：

```ts
export const importMonthEndAssetBalanceCsv =
  async (
    file: File,
  ): Promise<MonthEndAssetBalanceCsvImportResult> => {
    const formData =
      new FormData();

    formData.append(
      'file',
      file,
    );

    const response =
      await apiClient.post<
        ImportMonthEndAssetBalanceCsvResponse
      >(
        '/api/v1/month-end-asset-balances/imports',
        formData,
      );

    return response.data.data;
  };
```

API Clientでは、
CSV内容の解析や
登録可否判定を行わない。

---

#### 2.8 Mutationとして扱う

CSV-003は、
業務データを更新するため、
TanStack Queryを使用する場合は
Mutationとして扱う。

概念例：

```ts
export const useImportMonthEndAssetBalanceCsv =
  () =>
    useMutation({
      mutationFn:
        importMonthEndAssetBalanceCsv,
    });
```

Queryとして
自動実行しない。

---

#### 2.9 登録可能状態

登録ボタンは、
少なくとも以下を
すべて満たす場合のみ
有効化する。

```text
file != null
AND
preview != null
AND
preview.canImport = true
AND
CSV-003実行中ではない
```

概念例：

```ts
const canSubmit =
  file !== null
  && preview !== null
  && preview.canImport
  && !importMutation.isPending;
```

---

#### 2.10 登録ボタン

概念例：

```tsx
<button
  type="button"
  disabled={!canSubmit}
  onClick={handleImport}
>
  {importMutation.isPending
    ? '登録中...'
    : '登録する'}
</button>
```

登録実行中は、
二重送信防止のため
ボタンを無効化する。

ただし、
フロントエンドの
ボタン制御だけを
重複登録防止の保証としない。

---

#### 2.11 登録処理

概念例：

```ts
const handleImport =
  (): void => {
    if (
      file === null
      || preview === null
      || !preview.canImport
    ) {
      return;
    }

    importMutation.mutate(
      file,
    );
  };
```

CSV-002の
`preview.rows`を
CSV-003へ送信しない。

---

#### 2.12 登録確認

CSV-003は、
複数件の
月末資産残高を
一括登録する。

誤操作防止のため、
画面設計に応じて
登録前に確認ダイアログを
表示してよい。

例えば、

```text
2026年7月の月末資産残高を
2件登録します。

登録しますか？
```

のように、

- 対象年月
- 登録予定件数

を表示してよい。

登録予定件数は、
`preview.rows.length`などを
表示用途として使用できる。

ただし、
登録可否自体は
`preview.canImport`を使用する。

---

#### 2.13 CSV-002結果を登録条件としてのみ利用する

フロントエンドでは、
CSV-002の

```text
canImport = true
```

を
登録ボタンの有効化に使用する。

ただし、
CSV-003のバックエンドでは
CSV-002の結果を
信頼しない。

```text
Frontend
    ↓
canImport = true
    ↓
登録ボタン有効

Backend
    ↓
CSV-003
    ↓
再解析・再検証
```

とする。

---

#### 2.14 登録成功

CSV-003が成功した場合は、

```text
201 Created
```

とともに、

- `targetYearMonth`
- `importedCount`

が返却される。

概念例：

```json
{
  "targetYearMonth": "2026-07",
  "importedCount": 2
}
```

画面では、
必要に応じて

```text
2026年7月の月末資産残高を
2件登録しました。
```

のように表示してよい。

---

#### 2.15 importedCount

`importedCount`は、
実際に新規登録された
月末資産残高件数である。

TypeScriptでは、

```ts
importedCount: number;
```

として扱う。

正常レスポンスでは、
1以上となる。

フロントエンドで
CSV行数から
登録件数を再計算する必要はない。

---

#### 2.16 targetYearMonth

`targetYearMonth`は、
正常登録された
対象年月を表す。

```ts
targetYearMonth: string;
```

として扱う。

CSV-003成功後の
完了メッセージや
遷移先の対象年月指定に
利用してよい。

---

#### 2.17 登録成功後の状態クリア

CSV-003が成功した場合は、
保持している

- `file`
- CSV-002プレビュー結果

をクリアする。

概念例：

```ts
setFile(
  null,
);

setPreview(
  null,
);
```

登録済みCSVに対する
古いプレビュー状態を
画面へ残さない。

---

#### 2.18 file inputのリセット

React Stateで
`file = null`にしても、
ブラウザの
`<input type="file">`の表示が
自動的にクリアされない場合がある。

必要に応じて
`ref`または
`key`を使用して
ファイル入力をリセットする。

具体的な実装は、
画面実装時に決定する。

---

#### 2.19 Query Cacheの無効化

CSV-003成功後は、
月末資産残高や
月末資産状況に関連する
Query Cacheを無効化する。

例えば、
TanStack Queryでは
概念的に以下を行う。

```ts
await queryClient.invalidateQueries({
  queryKey: [
    'monthEndAssetBalances',
    result.targetYearMonth,
  ],
});
```

必要に応じて、
月末資産状況も
再取得対象とする。

```ts
await queryClient.invalidateQueries({
  queryKey: [
    'monthEndAssetSnapshots',
    result.targetYearMonth,
  ],
});
```

正式なQuery Keyは、
フロントエンド共通設計に従う。

---

#### 2.20 資産状況系Queryの扱い

CSV-003では、
登録した月末資産状況は
未確定のままである。

AST-001、
AST-002、
AST-003は
確定済み資産状況を
参照するAPIであるため、
CSV-003成功直後に
必ずしも表示結果が
変化するとは限らない。

そのため、
資産状況系Query Cacheを
無条件にすべて再取得する必要はない。

確定処理成功後に
資産状況系Queryを
invalidateする設計としてよい。

---

#### 2.21 登録成功後の画面遷移

CSV-003成功後は、
画面設計に応じて

- CSVインポート画面に留まる
- 月末資産状況詳細画面へ移動する
- 月末資産残高一覧画面へ移動する

などを選択できる。

対象年月は、

```ts
result.targetYearMonth
```

を使用する。

API内部の

```text
snapshotId
```

を
遷移のために必要としない。

---

#### 2.22 409 Conflict

CSV-003では、
CSV-002成功後でも
`409 Conflict`が返却される可能性がある。

代表例は、
以下とする。

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

これは、
CSV-002後に
業務状態が変更された場合などに
発生し得る。

---

#### 2.23 確定済みエラー

`MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED`
が返却された場合は、
対象年月が
既に確定済みとなったことを
利用者へ表示する。

例えば、

```text
対象年月はすでに確定されているため、
CSVを登録できません。
```

のように表示する。

古いプレビュー結果の

```text
canImport = true
```

を保持し続けない。

必要に応じて
プレビュー結果を無効化する。

---

#### 2.24 既存月末資産残高エラー

`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`
が返却された場合は、
対象年月・資産口座に
既存データが存在することを
利用者へ表示する。

既存データを
CSV値で上書きするための
再送操作を提供しない。

必要な場合は、
月末資産残高更新画面へ
誘導する。

---

#### 2.25 422 Unprocessable Entity

CSV-003では、
CSV内容に
登録できない入力・業務エラーがある場合、

```text
422 Unprocessable Entity
```

となる。

例えば、

- CSV形式不正
- 対象年月不正
- 資産口座不存在
- 残高記録単位不一致
- 月末残高不正
- CSV内重複

などを対象とする。

CSV-002のように

```text
200 OK
canImport = false
```

とはならない。

---

#### 2.26 CSV複数エラー

CSV-003で
複数のCSV行エラーが
`error.details`として返却される場合は、
各行のエラーを
一覧表示してよい。

概念的な型は、
API共通エラー型に
従う。

例えば、

```ts
export type CsvImportErrorDetail = {
  rowNumber?: number;
  field?: string;
  code?: string;
  message: string;
};
```

正式な型は、
共通エラー仕様に合わせる。

---

#### 2.27 CSV-003失敗後のプレビュー

CSV-003が
業務状態変化によって失敗した場合は、
必要に応じて
CSV-002を再実行できるようにする。

概念的には、

```text
CSV-002
canImport = true
    ↓
CSV-003
409 Conflict
    ↓
プレビュー無効化
    ↓
CSV-002
再実行
```

とする。

ただし、
確定済みなど
再プレビューしても
登録できないことが明確な場合は、
画面設計に応じて
適切な案内を表示する。

---

#### 2.28 VALIDATION_ERROR

`VALIDATION_ERROR`の場合は、
アップロードファイル自体の
入力不正として扱う。

例えば、

- `file`未指定
- ファイル形式不正
- ファイルサイズ超過

などである。

CSV登録処理が
開始されていないことを前提として
エラー表示する。

---

#### 2.29 INVALID_CSV_FORMAT

`INVALID_CSV_FORMAT`の場合は、
CSV自体を
正常に解析できないことを表示する。

利用者へ
CSV-001のテンプレートを使用して
再作成するよう案内してよい。

---

#### 2.30 利用者関連エラー

以下のエラーは、
API共通方針に従って処理する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
```

CSV-003画面だけで
独自の利用者エラー処理を
作成しない。

---

#### 2.31 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`の場合は、
共通サーバーエラーとして扱う。

登録成功とみなして
画面状態をクリアしてはならない。

ただし、
通信状態によっては
サーバー側で登録済みかどうか
クライアントから判断できない場合がある。

必要に応じて、
参照APIを再取得して
現在状態を確認する。

---

#### 2.32 通信エラー

CSV-003実行後に
ネットワークエラーとなった場合、
サーバー側では
登録が完了している可能性がある。

そのため、

```text
通信エラー
    ↓
同じCSVを無条件再送
```

だけを
唯一の復旧手段としない。

必要に応じて、
BAL-001などから
現在の登録状態を
再取得できるようにする。

---

#### 2.33 同一CSVの再送

CSV-003成功後に
同じCSVを再度送信すると、

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

となる。

フロントエンドでは、
登録成功後に

- `file`
- `preview`

をクリアすることで、
通常操作における
意図しない再送を防止する。

---

#### 2.34 二重送信防止

Mutation実行中は、
登録ボタンを非活性化する。

概念例：

```tsx
disabled={
  !canSubmit
  || importMutation.isPending
}
```

ただし、
バックエンドでも

- 重複確認
- トランザクション
- UNIQUE制約

によって
重複登録を防止する。

---

#### 2.35 Idempotency-Key

Phase1では、
CSV-003に

```text
Idempotency-Key
```

を付与しない。

フロントエンドでも、
独自の
Idempotency Key生成処理を
実装しない。

再送時は、
バックエンドの
重複チェックによって
競合として扱われる。

---

#### 2.36 登録済みかをフロントエンドだけで判断しない

CSV-003実行前に、
React側の画面Stateだけを見て

```text
この対象年月は未登録
```

と判断しない。

CSV-003では、
バックエンドが
最新のデータベース状態を
再確認する。

フロントエンドのStateは、
業務整合性の保証には使用しない。

---

#### 2.37 CSVをReact側で再解析しない

CSV-003のために
React側でCSVを正式解析しない。

以下のロジックを
バックエンドと二重実装しない。

- ヘッダー検証
- `target_year_month`検証
- 資産口座存在確認
- 残高記録単位判定
- `balance`検証
- CSV内重複判定
- 既存残高判定

正式な登録可否は、
バックエンドへ集約する。

---

#### 2.38 成功後に登録済みデータをレスポンスから構築しない

CSV-003のレスポンスには、
登録した各資産口座・残高は
含まれない。

そのため、
登録後の月末資産残高一覧を
CSV-003レスポンスから
React側で再構築しない。

必要な場合は、
BAL-001を再取得する。

---

### 3 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [CSV-001 月末資産残高CSVテンプレート取得](./csv-001-template.md)
- [CSV-002 月末資産残高CSVプレビュー](./csv-002-preview.md)
- [CSV-004 商品別月末評価額CSVテンプレート取得](./csv-004-template.md)
- [CSV-005 商品別月末評価額CSVプレビュー](./csv-005-preview.md)
- [CSV-006 商品別月末評価額CSV登録](./csv-006-create.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/csv-imports.md)
- [Reactアーキテクチャ設計](../../../architecture/react/csv-imports.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)
