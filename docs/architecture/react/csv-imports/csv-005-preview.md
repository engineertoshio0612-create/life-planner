##  CSV-005 商品別月末評価額CSVプレビュー

### 1 概要

CSV-004は、商品単位で管理する保有商品の月末評価額をCSVインポートするための公式CSVテンプレートを取得するAPIである。

React・TypeScriptでは、CSVインポート画面のテンプレート取得操作から本APIを呼び出し、返却されたCSVファイルをブラウザへ保存する。

本APIの正常レスポンスは通常のJSONではなく`text/csv`形式のファイルレスポンスであるため、API Clientでは`responseType: 'blob'`を指定し、取得結果を`Blob`として扱う。

```text
CSVインポート画面
    ↓
CSVテンプレート取得
    ↓
CSV-004
    ↓
Blobとしてレスポンス取得
    ↓
CSVファイルとして保存
```

取得するCSVテンプレートには、商品別月末評価額CSVで必要となる以下の項目が含まれる。

```text
対象年月
資産口座名
保有商品名
月末評価額
```

Phase1におけるCSVヘッダーは、以下を基本とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

CSVテンプレートの内容はバックエンドを正とし、React側でCSVヘッダーやテンプレート内容を独自生成しない。また、資産口座一覧や保有商品一覧などの利用者固有データをReact側で追加して、別のテンプレートを生成しない。

これにより、CSV-005 商品別月末評価額CSVプレビューおよびCSV-006 商品別月末評価額CSV登録とのCSV仕様の不整合を防止する。

```text
CSV-004
テンプレート取得
    ↓
CSVファイル保存
    ↓
利用者がCSV編集
    ↓
CSV-005
プレビュー
    ↓
CSV-006
登録
```

CSV-004は業務データを変更しない冪等なGET APIであるが、利用者による明示的なテンプレート取得操作を契機として実行する。TanStack Queryを使用する場合はMutationとして扱ってよく、画面表示時に自動実行する必要はない。

テンプレート取得中はボタンを無効化するなど、連続操作を抑制してよい。ただし、複数回実行されても業務データには影響しないため、登録APIのような厳密な二重送信防止は必要としない。

正常時は取得した`Blob`からObject URLを生成してCSVファイルとして保存し、保存処理後は`URL.revokeObjectURL()`によって不要なObject URLを解放する。Phase1では`month-end-holding-values-template.csv`を固定ファイル名として使用してよい。

また、`responseType: 'blob'`を使用するため、正常時のCSVレスポンスとエラー時のJSONレスポンスを適切に区別する。エラー時のJSONが`Blob`として取得された場合は共通エラー処理によって解析し、JSONエラーレスポンスをCSVファイルとして保存してはならない。この処理はCSV-001など他のファイルダウンロードAPIと共通化してよい。

`X-User-Id`の付与や利用者関連エラー、サーバーエラーなどはAPI共通方針に従い、CSV-004固有の利用者コンテキスト管理やエラー処理を実装しない。

CSV-004はCSVテンプレートの提供のみを責務とし、CSV内容の正式な解析や入力値検証は行わない。利用者が編集したCSVの検証はCSV-005、登録時の最終的な再検証および登録処理はCSV-006へ委ね、React側でもCSV仕様や業務ルールを重複実装しない。

---

### 2 React・TypeScriptでの利用

CSV-005は、
利用者が選択した
商品別月末評価額CSVを送信し、
CSV-006で登録する前に
登録予定内容とエラー内容を確認するために使用する。

フロントエンドでは、
CSVファイルを`File`として保持し、
`FormData`へ設定して
APIへ送信する。

概念的な利用フローは、
以下とする。

```text
CSV-004
テンプレート取得
    ↓
利用者がCSV編集
    ↓
CSVファイル選択
    ↓
CSV-005
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
       CSV-006
```

---

#### 2.1 TypeScript型

CSVプレビュー結果は、
以下のような型として扱う。

概念例：

```ts
export type CsvPreviewError = {
  code: string;
  message: string;
};

export type MonthEndHoldingValueCsvPreviewRow = {
  rowNumber: number;
  assetAccountName: string | null;
  holdingAssetName: string | null;
  value: number | null;
  errors: CsvPreviewError[];
};

export type MonthEndHoldingValueCsvPreview = {
  targetYearMonth: string | null;
  canImport: boolean;
  errors: CsvPreviewError[];
  rows: MonthEndHoldingValueCsvPreviewRow[];
};

export type PreviewMonthEndHoldingValueCsvResponse = {
  data: MonthEndHoldingValueCsvPreview;
  requestId: string;
};
```

API共通Envelopeの
正式な型定義が存在する場合は、
共通型を利用する。

概念例：

```ts
export type PreviewMonthEndHoldingValueCsvResponse =
  ApiResponse<MonthEndHoldingValueCsvPreview>;
```

---

#### 2.2 CSVファイルの保持

CSVファイルは、
ブラウザの`File`として保持する。

概念例：

```ts
const [file, setFile] =
  useState<File | null>(
    null,
  );
```

CSV-005実行後も、
CSV-006で
同じCSVファイルを再送するため、
登録処理が完了するまで
`File`を保持する。

---

#### 2.3 ファイル選択

ファイル選択UIでは、
CSVファイルを
選択できるようにする。

概念例：

```tsx
<input
  type="file"
  accept=".csv,text/csv"
  onChange={(event) => {
    const selectedFile =
      event.target.files?.[0]
      ?? null;

    setFile(
      selectedFile,
    );
  }}
/>
```

`accept`属性は、
利用者のファイル選択を
補助するために使用する。

バックエンド側の
ファイル形式検証を
代替するものではない。

---

#### 2.4 FormData

CSVファイルは、
`FormData`へ
`file`という項目名で設定する。

概念例：

```ts
const formData =
  new FormData();

formData.append(
  'file',
  file,
);
```

以下のような
業務データは
`FormData`へ追加しない。

```text
userId
targetYearMonth
assetAccountId
holdingAssetId
value
confirmed
```

`userId`は
`X-User-Id`で扱い、
その他の情報は
CSV内容から取得する。

---

#### 2.5 Content-Type

`multipart/form-data`の
`Content-Type`は、
ブラウザまたは
HTTP Clientに設定させる。

以下のように
手動設定しない。

```ts
headers: {
  'Content-Type':
    'multipart/form-data',
}
```

`FormData`を送信することで、
必要なboundaryを
HTTP Client側に生成させる。

---

#### 2.6 API Client

API呼び出しは、
CSVファイルを受け取る
専用関数として定義する。

概念例：

```ts
export const previewMonthEndHoldingValueCsv =
  async (
    file: File,
  ): Promise<MonthEndHoldingValueCsvPreview> => {
    const formData =
      new FormData();

    formData.append(
      'file',
      file,
    );

    const response =
      await apiClient.post<
        PreviewMonthEndHoldingValueCsvResponse
      >(
        '/api/v1/month-end-holding-values/imports/preview',
        formData,
      );

    return response.data.data;
  };
```

CSV解析や
登録可否判定を
API Client内で行わない。

---

#### 2.7 X-User-Id

`X-User-Id`は、
通常のAPIと同様に
共通API Clientから付与する。

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

CSV-005専用処理で
`userId`を
`FormData`へ追加しない。

---

#### 2.8 Mutationとして扱う

CSV-005は、
業務データを変更しないが、
利用者操作によって
CSVファイルを送信し、
明示的にプレビューを実行するAPIである。

TanStack Queryを使用する場合は、
Mutationとして扱ってよい。

概念例：

```ts
export const usePreviewMonthEndHoldingValueCsv =
  () =>
    useMutation({
      mutationFn:
        previewMonthEndHoldingValueCsv,
    });
```

画面表示時に
自動実行するQueryとしては
扱わない。

---

#### 2.9 プレビュー実行

利用者が
CSVファイルを選択した後、
プレビューボタンから
CSV-005を実行する。

概念例：

```ts
const previewMutation =
  usePreviewMonthEndHoldingValueCsv();

const handlePreview =
  (): void => {
    if (file === null) {
      return;
    }

    previewMutation.mutate(
      file,
    );
  };
```

---

#### 2.10 プレビューボタン

CSVファイルが
選択されていない場合、
プレビューボタンを
無効化してよい。

また、
プレビュー実行中も
二重送信防止のため
無効化してよい。

概念例：

```tsx
<button
  type="button"
  disabled={
    file === null
    || previewMutation.isPending
  }
  onClick={
    handlePreview
  }
>
  {previewMutation.isPending
    ? '確認中...'
    : 'プレビュー'}
</button>
```

ただし、
`file`必須チェックは
バックエンドでも行う。

---

#### 2.11 プレビュー結果の保持

CSV-005成功後は、
プレビュー結果を
画面Stateへ保持してよい。

概念例：

```ts
const [preview, setPreview] =
  useState<
    MonthEndHoldingValueCsvPreview
    | null
  >(null);
```

Mutationの
`data`をそのまま利用する場合は、
別Stateを作成しなくてもよい。

同じ情報を
複数のStateへ
不要に二重管理しない。

---

#### 2.12 targetYearMonth

`targetYearMonth`は、
CSV全体から特定された
対象年月を表す。

正常例：

```json
{
  "targetYearMonth": "2026-07"
}
```

対象年月が混在しているなど、
単一の対象年月として
特定できない場合は、

```json
{
  "targetYearMonth": null
}
```

となる。

そのため、
TypeScriptでは、

```ts
targetYearMonth:
  string | null;
```

として扱う。

---

#### 2.13 canImport

`canImport`は、
プレビュー時点で
CSV全体を登録可能かを表す。

```text
true
    → プレビュー時点では登録可能

false
    → 登録不可となるエラーあり
```

登録ボタンの制御には、
この値を使用する。

フロントエンドで
`errors`配列を独自に解析して
登録可否を再計算しない。

---

#### 2.14 canImportは登録成功保証ではない

`canImport = true`は、
CSV-006が
必ず成功することを意味しない。

CSV-005実行後に、

- 月末資産状況が確定される
- 商品別月末評価額が別処理で登録される
- 資産口座の状態が変更される
- 保有商品の状態が変更される

可能性がある。

そのため、
CSV-006で
業務エラーが返却される可能性を
考慮する。

---

#### 2.15 CSV全体エラー

`data.errors`には、
CSV全体に関係する
エラーを表示する。

例えば、

```text
CSV_DATA_REQUIRED
MULTIPLE_TARGET_YEAR_MONTHS
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

などを対象とする。

概念例：

```tsx
{preview.errors.length > 0 && (
  <ul>
    {preview.errors.map(
      (error) => (
        <li key={error.code}>
          {error.message}
        </li>
      ),
    )}
  </ul>
)}
```

---

#### 2.16 rows

`rows`には、
CSV各行の
プレビュー結果が返却される。

概念例：

```tsx
<tbody>
  {preview.rows.map(
    (row) => (
      <CsvPreviewRow
        key={row.rowNumber}
        row={row}
      />
    ),
  )}
</tbody>
```

バックエンドから返却された
CSV上の並び順を維持して
表示する。

---

#### 2.17 rowNumber

`rowNumber`は、
CSVファイル上の
実際の行番号として扱う。

ヘッダーを1行目とするため、
最初のデータ行は、

```text
rowNumber = 2
```

となる。

画面では、

```text
2行目
3行目
4行目
```

のように
利用者へ表示してよい。

---

#### 2.18 assetAccountName

`assetAccountName`は、
CSVへ入力された
資産口座名を表示する。

概念例：

```tsx
<td>
  {row.assetAccountName
    ?? '-'}
</td>
```

内部の

```text
asset_accounts.id
```

は
フロントエンドで扱わない。

---

#### 2.19 holdingAssetName

`holdingAssetName`は、
CSVへ入力された
保有商品名を表示する。

概念例：

```tsx
<td>
  {row.holdingAssetName
    ?? '-'}
</td>
```

内部の

```text
holding_assets.id
```

は
フロントエンドで扱わない。

---

#### 2.20 value

`value`は、
正常に整数として解析できた場合、
`number`として扱う。

```ts
value:
  number | null;
```

表示時は、
日本円として
フォーマットしてよい。

概念例：

```tsx
<td>
  {row.value === null
    ? '-'
    : formatYen(
        row.value,
      )}
</td>
```

---

#### 2.21 0円とnull

`value = 0`は、
正常な業務値である。

一方、

```text
value = null
```

は、
正常な整数として
扱えなかった状態などを表す。

そのため、
以下のような
truthy / falsy判定を行わない。

```tsx
{row.value
  ? formatYen(
      row.value,
    )
  : '-'}
```

上記では、
0円まで`-`となる。

以下のように
明示的に`null`を判定する。

```tsx
{row.value === null
  ? '-'
  : formatYen(
      row.value,
    )}
```

---

#### 2.22 行エラー

各行の`errors`には、
そのCSV行に関する
入力・業務エラーが設定される。

例えば、

```text
ASSET_ACCOUNT_NOT_FOUND
BALANCE_RECORDING_UNIT_MISMATCH
HOLDING_ASSET_NOT_FOUND
HOLDING_ASSET_NOT_AVAILABLE
INVALID_VALUE
DUPLICATE_HOLDING_ASSET_IN_CSV
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

などを対象とする。

概念例：

```tsx
{row.errors.length > 0 && (
  <ul>
    {row.errors.map(
      (error) => (
        <li key={error.code}>
          {error.message}
        </li>
      ),
    )}
  </ul>
)}
```

1行に
複数エラーが
存在する可能性があるため、
1行1エラーを前提としない。

---

#### 2.23 エラー行の表示

`row.errors.length > 0`
の場合は、
該当行を
視覚的に強調してよい。

例えば、

- 背景色
- アイコン
- 枠線
- エラーメッセージ

などを使用する。

具体的な表示方法は、
画面設計に従う。

---

#### 2.24 200 OKかつcanImport=false

CSV-005では、
CSV内容に
入力・業務エラーが存在していても、
プレビュー処理自体が
正常に完了した場合は、

```text
200 OK
```

となる。

そのため、
HTTPステータスだけで
登録可能と判断しない。

必ず、

```ts
preview.canImport
```

を確認する。

---

#### 2.25 HTTPエラー

以下は、
プレビュー結果ではなく
HTTPエラーとして扱う。

- `X-User-Id`不正
- `file`未指定
- ファイル形式不正
- ファイルサイズ超過
- CSV解析不能
- CSVヘッダー不正
- サーバー内部エラー

これらの場合は、
`MonthEndHoldingValueCsvPreview`として
処理しない。

---

#### 2.26 VALIDATION_ERROR

`VALIDATION_ERROR`の場合は、
主にアップロードファイルに関する
入力不正として扱う。

例えば、

- ファイル未指定
- 許可されないファイル形式
- ファイルサイズ超過

などを表示する。

フロントエンドで
同じ検証を完全に
再実装する必要はない。

UX向上を目的とした
事前チェックは行ってよい。

---

#### 2.27 INVALID_CSV_FORMAT

`INVALID_CSV_FORMAT`の場合は、
CSVをプレビュー可能な形式として
解析できなかったことを表示する。

例えば、

```text
CSVファイルの形式を確認してください。
商品別月末評価額CSVテンプレートを使用して
再度作成してください。
```

のように案内する。

必要に応じて、
CSV-004の
テンプレート取得導線を表示する。

---

#### 2.28 利用者関連エラー

以下のエラーは、
API共通方針に従って処理する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
```

CSV-005画面だけで
独自処理を実装しない。

必要に応じて、
利用者選択画面へ戻す。

---

#### 2.29 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`の場合は、
共通サーバーエラーとして扱う。

CSV登録処理へは進まない。

必要に応じて、
同じCSVを
再プレビューできる導線を表示する。

---

#### 2.30 エラーコードと表示文言

プレビュー内の
`error.code`は、
画面処理の判別に使用できる。

ただし、
フロントエンドで
業務判定そのものを
再実装しない。

バックエンドから
`message`が返却される場合は、
基本的にその文言を表示してよい。

フロントエンド側で
文言を管理する場合は、
エラーコードとの対応を
共通定義へ集約する。

---

#### 2.31 ファイル変更時

利用者が
プレビュー後に
別のCSVファイルを選択した場合は、
以前のプレビュー結果を
無効化する。

概念例：

```ts
const handleFileChange =
  (
    nextFile: File | null,
  ): void => {
    setFile(
      nextFile,
    );

    setPreview(
      null,
    );
  };
```

新しいCSVファイルに対して、
以前の

```text
canImport = true
```

を流用してはならない。

---

#### 2.32 再プレビュー

CSV-005は、
業務データを変更しないため、
同じCSVまたは
修正したCSVを
再度プレビューできる。

概念的な操作は、
以下とする。

```text
CSV-005
    ↓
canImport = false
    ↓
利用者がCSV修正
    ↓
ファイル再選択
    ↓
CSV-005
    ↓
再プレビュー
```

---

#### 2.33 CSV-006登録ボタン

CSV-006の登録ボタンは、
少なくとも以下を
すべて満たす場合に
有効化する。

```text
file != null
AND
preview != null
AND
preview.canImport = true
```

概念例：

```ts
const canImport =
  file !== null
  && preview !== null
  && preview.canImport;
```

```tsx
<button
  type="button"
  disabled={!canImport}
>
  登録する
</button>
```

---

#### 2.34 CSV-006へ同じFileを送信する

CSV-006では、
CSV-005で使用した
同じ`File`を再送する。

```text
CSV-005
File A
    ↓
canImport = true
    ↓
CSV-006
File A
```

プレビュー結果から
CSV内容を再構築して
CSV-006へ送信しない。

---

#### 2.35 CSV-006成功後

CSV-006が成功した場合は、
保持していた

- CSVファイル
- プレビュー結果

をクリアしてよい。

概念例：

```ts
setFile(
  null,
);

setPreview(
  null,
);
```

また、
関連する

- 商品別月末評価額
- 月末資産状況

のQuery Cacheを
invalidateする。

具体的なinvalidate対象は、
フロントエンド共通設計に従う。

---

#### 2.36 CSV-006失敗時

CSV-005で

```text
canImport = true
```

となっていても、
CSV-006で
登録不可となる可能性がある。

その場合は、
CSV-006のエラーを表示し、
必要に応じて
CSV-005を再実行する。

CSV-005の結果だけを理由に
CSV-006の業務エラーを
無視しない。

---

#### 2.37 CSVをReact側で正式解析しない

Phase1では、
CSV内容の正式な解析は
バックエンド側で行う。

React側で
CSV Parserライブラリを利用して、
CSV-005と同じ

- ヘッダー検証
- 対象年月検証
- 資産口座検証
- 保有商品検証
- 評価額検証
- CSV内重複検証

を二重実装しない。

フロントエンドは、
CSV-005の結果を
表示・操作制御へ使用する。

---

#### 2.38 canImportをReact側で再計算しない

以下のような
独自判定は行わない。

```ts
const canImport =
  preview.errors.length === 0
  && preview.rows.every(
    (row) =>
      row.errors.length === 0,
  );
```

正式な登録可否は、
バックエンドが返却する

```text
canImport
```

を使用する。

これにより、
登録可否判定を
バックエンドへ集約する。

---

### 3 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [CSV-001 月末資産残高CSVテンプレート取得](./csv-001-template.md)
- [CSV-002 月末資産残高CSVプレビュー](./csv-002-preview.md)
- [CSV-003 月末資産残高CSV登録](./csv-003-create.md)
- [CSV-004 商品別月末評価額CSVテンプレート取得](./csv-004-template.md)
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

