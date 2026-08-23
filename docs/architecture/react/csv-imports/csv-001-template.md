##  CSV-001 月末資産残高CSVテンプレート取得

### 1 概要

CSV-001は、月末資産残高CSVインポートで使用するCSVテンプレートを取得するためのAPIである。

React・TypeScriptでは、CSVインポート画面の「CSVテンプレートを取得」操作から本APIを呼び出し、返却されたCSVファイルをブラウザへ保存する。

本APIの正常レスポンスは通常のJSONではなく、`text/csv`形式のファイルレスポンスであるため、API Clientでは`responseType: 'blob'`を指定し、取得結果を`Blob`として扱う。

```text
CSVインポート画面
    ↓
CSVテンプレート取得
    ↓
CSV-001
    ↓
Blobとしてレスポンス取得
    ↓
CSVファイルとして保存
```

取得したテンプレートには、月末資産残高CSVで必要となる以下の項目が含まれる。

```text
対象年月
資産口座名
月末残高
```

CSVテンプレートの内容はバックエンドを正とし、React側でCSVヘッダーやテンプレート内容を独自生成しない。これにより、CSV-002 月末資産残高CSVプレビューおよびCSV-003 月末資産残高CSV登録とのCSV仕様の不整合を防止する。

本APIは利用者のボタン操作を契機として実行するファイル取得処理であり、業務データを変更しない冪等なGET APIである。そのため、Phase1ではReact Queryによるキャッシュ管理を必須とせず、API Clientを直接呼び出す構成としてよい。

`X-User-Id`の付与、エラー処理、ローディング状態などはフロントエンド共通方針に従う。特に`responseType: 'blob'`を使用するため、4xx・5xx時にJSON形式のエラーレスポンスがBlobとして返却される場合を考慮し、正常なCSVレスポンスとAPIエラーを適切に区別する。

CSV-001はCSVインポート処理の入口として位置付け、取得したテンプレートへの入力後、CSV-002によるプレビュー、CSV-003による登録へ進む画面フローとする。

```text
CSV-001
テンプレート取得
    ↓
利用者がCSVを編集
    ↓
CSV-002
プレビュー
    ↓
内容確認
    ↓
CSV-003
登録
```

CSV-001からCSV-002またはCSV-003を自動実行せず、それぞれの操作を明確に分離する。


---

### 2 React・TypeScriptでの利用

CSV-001は、
月末資産残高CSVテンプレートを
CSVファイルとして取得するAPIである。

正常時は、
JSONレスポンスではなく、
`text/csv`の
ファイルレスポンスを返却する。

そのため、
通常のJSON APIとは
異なる扱いとする。

API呼び出し例は、
以下とする。

```ts
export const downloadMonthEndAssetBalanceCsvTemplate =
  async (): Promise<Blob> => {
    const response =
      await apiClient.get(
        '/api/v1/month-end-asset-balances/csv-template',
        {
          responseType: 'blob',
        },
      );

    return response.data;
  };
```

取得したBlobを使用して、
ブラウザからCSVファイルを
保存する。

---

#### 2.1 レスポンス型

CSV-001の正常レスポンスは、
JSONではない。

そのため、
以下のような
JSONレスポンス型は定義しない。

```ts
type CsvTemplateResponse = {
  data: {
    csv: string;
  };
};
```

正常時は、
`Blob`として扱う。

```ts
type MonthEndAssetBalanceCsvTemplate =
  Blob;
```

---

#### 2.2 responseType

Axios等のAPI Clientを
使用する場合は、
`responseType`に

```text
blob
```

を指定する。

概念例：

```ts
const response =
  await apiClient.get(
    '/api/v1/month-end-asset-balances/csv-template',
    {
      responseType: 'blob',
    },
  );
```

CSVレスポンスを
JSONとして解析しない。

---

#### 2.3 ファイル保存

取得したBlobから、
ブラウザ上で
ダウンロード処理を行う。

概念例：

```ts
export const saveCsvFile =
  (
    blob: Blob,
    fileName: string,
  ): void => {
    const url =
      URL.createObjectURL(
        blob,
      );

    const anchor =
      document.createElement(
        'a',
      );

    anchor.href = url;
    anchor.download = fileName;

    document.body.appendChild(
      anchor,
    );

    anchor.click();

    anchor.remove();

    URL.revokeObjectURL(
      url,
    );
  };
```

CSV-001では、
例えば以下の
ファイル名を使用する。

```text
month-end-asset-balances-template.csv
```

---

#### 2.4 Content-Dispositionの利用

可能であれば、
レスポンスの
`Content-Disposition`から
ファイル名を取得する。

概念例：

```ts
const contentDisposition =
  response.headers[
    'content-disposition'
  ];
```

ただし、
Phase1では
CSVテンプレートのファイル名は
固定であるため、
フロントエンド共通定数として
保持してもよい。

概念例：

```ts
export const
  MONTH_END_ASSET_BALANCE_CSV_TEMPLATE_FILE_NAME =
    'month-end-asset-balances-template.csv';
```

バックエンドと
フロントエンドで
ファイル名を二重管理する場合は、
変更時の整合性に注意する。

---

#### 2.5 Content-Typeの確認

正常時の
`Content-Type`は、

```text
text/csv; charset=UTF-8
```

を想定する。

ただし、
通常の画面処理では
`200 OK`かどうかを確認したうえで、
取得したBlobを
CSVファイルとして扱えばよい。

---

#### 2.6 ダウンロードボタン

CSVインポート画面では、
テンプレート取得用の
ボタンを配置する。

概念例：

```tsx
<button
  type="button"
  onClick={
    handleDownloadTemplate
  }
>
  CSVテンプレートを取得
</button>
```

クリック時に
CSV-001を実行する。

---

#### 2.7 ダウンロード処理例

概念例：

```ts
const handleDownloadTemplate =
  async (): Promise<void> => {
    const blob =
      await downloadMonthEndAssetBalanceCsvTemplate();

    saveCsvFile(
      blob,
      'month-end-asset-balances-template.csv',
    );
  };
```

実際のエラー処理や
ローディング状態管理は、
フロントエンド共通設計に従う。

---

#### 2.8 Mutationとして扱うか

CSV-001は、
HTTPメソッドとしては
GETであり、
業務データを変更しない。

React Queryを使用する場合、
通常のデータ取得Queryとして
扱うこともできる。

ただし、
CSVテンプレート取得は
利用者のボタン操作によって
都度ファイルを取得する
命令的な処理である。

そのため、
Phase1では
必ずしもReact Queryへ
載せる必要はない。

例えば、
API Clientを
直接呼び出してよい。

```ts
const blob =
  await downloadMonthEndAssetBalanceCsvTemplate();
```

---

#### 2.9 キャッシュ

CSVテンプレートは、
固定内容であるが、
Phase1では
ブラウザ側で
積極的にキャッシュする必要はない。

利用者が
テンプレート取得ボタンを押した時点で、
APIから取得する。

CSVテンプレートの仕様変更後に
古いファイルを
使い続ける可能性を減らすためにも、
明示的な長期キャッシュは行わない。

---

#### 2.10 ローディング状態

テンプレート取得中は、
ボタンを一時的に
無効化してよい。

概念例：

```tsx
<button
  type="button"
  disabled={isDownloading}
  onClick={
    handleDownloadTemplate
  }
>
  {isDownloading
    ? '取得中...'
    : 'CSVテンプレートを取得'}
</button>
```

これにより、
短時間に
同じダウンロード処理を
何度も実行することを防ぐ。

---

#### 2.11 複数回取得

CSV-001は
冪等なGET APIであるため、
利用者が複数回実行しても
問題ない。

例えば、

```text
1回目
テンプレート取得
    ↓
CSV保存

2回目
テンプレート取得
    ↓
CSV保存
```

としても、
業務データは変更されない。

---

#### 2.12 X-User-Id

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

CSV-001専用処理で、
利用者IDを
URLやリクエストボディへ
設定しない。

---

#### 2.13 USER_CONTEXT_REQUIRED

`USER_CONTEXT_REQUIRED`
が返却された場合は、
利用者が
選択されていない状態として扱う。

CSVテンプレートの
保存処理は行わない。

利用者選択を
促す画面または
共通エラー処理へ遷移する。

---

#### 2.14 INVALID_USER_ID

`INVALID_USER_ID`
が返却された場合は、
共通エラーとして扱う。

通常の画面操作では、
フロントエンドが保持する
利用者IDを使用するため、
発生しないことを前提とする。

---

#### 2.15 USER_NOT_FOUND

`USER_NOT_FOUND`
が返却された場合は、
操作対象利用者を
現在利用できない状態として扱う。

利用者選択画面へ戻すなど、
API共通方針に従って処理する。

---

#### 2.16 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`
が返却された場合は、
CSVテンプレートを
保存しない。

共通の
サーバーエラー表示を行う。

必要に応じて、
再取得できる導線を表示する。

---

#### 2.17 Blobレスポンス時のエラー処理

API Clientで
`responseType: 'blob'`を指定した場合、
エラー時のJSONレスポンスも
Blobとして受け取る場合がある。

そのため、
共通API Clientで
Blobレスポンス時の
エラーJSONを解析できるようにする。

概念例：

```ts
const parseBlobError =
  async (
    blob: Blob,
  ): Promise<ApiErrorResponse> => {
    const text =
      await blob.text();

    return JSON.parse(
      text,
    ) as ApiErrorResponse;
  };
```

CSV-001専用画面で
独自のエラー形式を
定義しない。

---

#### 2.18 正常BlobとエラーBlobを区別する

HTTPステータスが
成功の場合のみ、
レスポンスBlobを
CSVファイルとして保存する。

エラー時に、

```text
JSONエラーレスポンス
```

を

```text
.csv
```

として
保存してはならない。

概念的には、

```text
200 OK
    ↓
CSVとして保存

4xx / 5xx
    ↓
APIエラーとして処理
```

とする。

---

#### 2.19 CSV内容をReact側で生成しない

CSV-001のテンプレートは、
バックエンドから取得する。

React側で、

```ts
const csv =
  'target_year_month,asset_account_name,balance';
```

のように
同じテンプレートを
独自生成しない。

テンプレート定義を
バックエンド側へ集約することで、
CSV-002・CSV-003との
仕様不整合を防止する。

---

#### 2.20 CSVヘッダーをフロントエンドで検証しない

CSV-001で取得した
テンプレートについて、
フロントエンド側で
ヘッダーが正しいかを
再検証する必要はない。

CSV仕様の保証は、
バックエンドおよび
テストの責務とする。

---

#### 2.21 CSV-002との連携

利用者は、
CSV-001で取得した
テンプレートへ
月末資産残高データを入力する。

その後、
CSV-002 月末資産残高CSVプレビューへ
ファイルを送信する。

概念的な画面フローは、
以下とする。

```text
CSVテンプレート取得
    ↓
利用者がCSV入力
    ↓
ファイル選択
    ↓
CSV-002
プレビュー
```

CSV-001実行直後に
CSV-002を
自動実行しない。

---

#### 2.22 CSV-003との連携

CSV-003は、
CSV-002で内容を確認した後に
月末資産残高を
一括登録するAPIである。

フロントエンドでは、
基本的に以下の流れとする。

```text
CSV-001
テンプレート取得
    ↓
CSV編集
    ↓
CSV-002
プレビュー
    ↓
内容確認
    ↓
CSV-003
登録
```

CSV-001から
直接CSV-003へ
連携する処理は行わない。

---

### 3 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [CSV-002 月末資産残高CSVプレビュー](./csv-002-preview.md)
- [CSV-003 月末資産残高CSV登録](./csv-003-create.md)
- [CSV-004 商品別月末評価額CSVテンプレート取得](./csv-004-template.md)
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
