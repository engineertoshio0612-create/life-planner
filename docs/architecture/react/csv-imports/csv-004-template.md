##  CSV-004 商品別月末評価額CSVテンプレート取得

### 1 概要

商品単位で管理する
保有商品の月末評価額を
CSVインポートするための
CSVテンプレートを取得する。

本APIでは、
以下の項目を持つ
商品単位用CSVテンプレートを返却する。

```text
対象年月
資産口座名
保有商品名
月末評価額
```

CSVテンプレートは、CSV-005 商品別月末評価額CSVプレビュー、CSV-006 商品別月末評価額CSV登録で使用する入力形式とする。

Phase1では、商品単位用CSVの基本的なフォーマットを以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

本APIでは、CSVテンプレートの取得のみを行う。

以下の処理は行わない。

- 商品別月末評価額の登録
- 商品別月末評価額の更新
- 商品別月末評価額の削除
- CSVプレビュー
- CSV入力値検証
- 対象年月の判定
- 重複登録判定
- 月末資産状況の作成
- 月末資産状況の確定

商品単位用のCSVテンプレートを提供することは、機能要件の「CSVテンプレート」および「CSVフォーマット」に対応する。

---

### 2 React・TypeScriptでの利用

CSV-004は、
商品別月末評価額CSVを作成するための
公式テンプレートを
フロントエンドから取得する際に使用する。

利用者は、
取得したCSVテンプレートへ

- 対象年月
- 資産口座名
- 保有商品名
- 商品別月末評価額

を入力し、
CSV-005 商品別月末評価額CSVプレビュー、
CSV-006 商品別月末評価額CSV登録で使用する。

概念的な利用フローは、
以下とする。

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

CSV-004は、
JSON APIではなく
CSVファイルを返却するため、
通常のJSONレスポンス処理とは
分離して扱う。

---

#### 2.1 TypeScript型

CSV-004の正常レスポンスは、
JSONではなくCSVファイルである。

そのため、
正常レスポンス用の
業務DTO型は定義しない。

API Clientでは、
レスポンスを

```ts
Blob
```

として扱う。

概念例：

```ts
export type MonthEndHoldingValueCsvTemplate =
  Blob;
```

ただし、
単純に`Blob`を返却する場合は、
専用type aliasを
作成しなくてもよい。

---

#### 2.2 API Client

CSVテンプレート取得は、
通常のJSON取得APIとは分けて
専用関数として定義する。

概念例：

```ts
export const downloadMonthEndHoldingValueCsvTemplate =
  async (): Promise<Blob> => {
    const response =
      await apiClient.get<Blob>(
        '/api/v1/month-end-holding-values/csv-template',
        {
          responseType: 'blob',
          headers: {
            Accept: 'text/csv',
          },
        },
      );

    return response.data;
  };
```

`responseType`は、

```text
blob
```

とする。

CSVレスポンスを
JSONとして解析しない。

---

#### 2.3 X-User-Id

`X-User-Id`は、
他のAPIと同様に
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

CSV-004専用処理で
利用者IDを
URLやクエリパラメータへ
追加しない。

---

#### 2.4 responseType

CSV-004では、
正常レスポンスを
`Blob`として取得する。

以下のように、
JSONを前提とした
API共通型を使用しない。

```ts
apiClient.get<
  ApiResponse<SomeType>
>(...);
```

正常時は、

```text
text/csv
```

であるため、
JSON成功Envelopeは存在しない。

---

#### 2.5 テンプレートダウンロード

取得した`Blob`から
Object URLを生成し、
ブラウザのダウンロード処理を行う。

概念例：

```ts
export const saveBlobAsFile =
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

    anchor.href =
      url;

    anchor.download =
      fileName;

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

CSVテンプレート本文を
React Stateへ
文字列として保持する必要はない。

---

#### 2.6 ファイル名

Phase1では、
フロントエンド側でも
以下のファイル名を
使用してよい。

```text
month-end-holding-values-template.csv
```

概念例：

```ts
const FILE_NAME =
  'month-end-holding-values-template.csv';
```

ただし、
将来的に
`Content-Disposition`から
ファイル名を取得する
共通処理を実装する場合は、
そちらへ統一してよい。

---

#### 2.7 Content-Dispositionからのファイル名取得

サーバーが
`Content-Disposition`へ
ファイル名を設定しているため、
必要に応じて
レスポンスヘッダーから
ファイル名を取得してよい。

ただし、
Phase1では
固定ファイル名で十分であれば、
無理に解析処理を追加しない。

複雑な

```text
filename*
RFC 5987
```

対応などは、
必要になった時点で
共通化する。

---

#### 2.8 Mutationとして扱う

CSV-004は
業務データを更新しないが、
利用者が明示的に
「テンプレートを取得する」
操作によって実行する。

TanStack Queryを使用する場合は、
Mutationとして扱ってよい。

概念例：

```ts
export const useDownloadMonthEndHoldingValueCsvTemplate =
  () =>
    useMutation({
      mutationFn:
        downloadMonthEndHoldingValueCsvTemplate,
    });
```

CSVファイル取得を
画面表示時に
自動実行する必要はない。

---

#### 2.9 テンプレート取得ボタン

概念例：

```tsx
<button
  type="button"
  disabled={
    downloadMutation.isPending
  }
  onClick={
    handleDownloadTemplate
  }
>
  {downloadMutation.isPending
    ? '取得中...'
    : 'CSVテンプレートを取得'}
</button>
```

取得処理中は、
意図しない連続操作を避けるため
ボタンを無効化してよい。

ただし、
CSV-004自体は冪等であるため、
複数回取得されても
業務上の問題はない。

---

#### 2.10 テンプレート取得処理

概念例：

```ts
const downloadMutation =
  useDownloadMonthEndHoldingValueCsvTemplate();

const handleDownloadTemplate =
  (): void => {
    downloadMutation.mutate(
      undefined,
      {
        onSuccess:
          (blob) => {
            saveBlobAsFile(
              blob,
              'month-end-holding-values-template.csv',
            );
          },
      },
    );
  };
```

正常時は、
CSVファイルとして
保存処理を行う。

---

#### 2.11 CSV内容をReactで生成しない

商品別月末評価額CSVの
正式なテンプレートは、
CSV-004から取得する。

React側で、
以下のような
CSVヘッダーを
独自生成しない。

```ts
const headers = [
  'target_year_month',
  'asset_account_name',
  'holding_asset_name',
  'value',
];
```

正式なCSV仕様を
フロントエンドと
バックエンドで
二重管理しない。

---

#### 2.12 利用者固有データを追加しない

CSV-004で取得した
テンプレートへ、
React側で

- 資産口座一覧
- 保有商品一覧
- 商品別月末評価額

を自動追記して
別テンプレートを生成しない。

CSV-004で返却された
固定テンプレートを
そのまま利用者へ提供する。

---

#### 2.13 CSV-005への利用

利用者が
CSVテンプレートを編集した後は、
CSV-005へ
アップロードして
内容をプレビューする。

概念的には、

```text
CSV-004
テンプレート取得
    ↓
編集済みCSV
    ↓
File
    ↓
CSV-005
```

となる。

CSV-004の
レスポンスBlob自体を
CSV-005へ直接送信する
必要はない。

利用者が編集して
選択した`File`を
CSV-005へ送信する。

---

#### 2.14 CSV-006への利用

CSV-005で

```text
canImport = true
```

となった場合は、
同一のCSVファイルを
CSV-006へ送信する。

```text
CSV-004
テンプレート取得
    ↓
利用者編集
    ↓
CSV-005
プレビュー
    ↓
CSV-006
登録
```

CSV-004は、
この一連のCSVインポートフローの
入口として使用する。

---

#### 2.15 正常時のContent-Type

正常時は、

```text
text/csv
```

を受信する。

フロントエンドでは、
正常時のレスポンスを
JSONとして処理しない。

---

#### 2.16 エラー時のContent-Type

CSV-004では、
正常時はCSV、
エラー時はJSONとなる。

```text
成功
    → text/csv

失敗
    → application/json
```

Axiosなどで
`responseType: 'blob'`を使用すると、
エラー時のJSONも
`Blob`として受け取る可能性がある。

そのため、
ファイルダウンロードAPI用の
共通エラー変換処理を
用意してよい。

---

#### 2.17 Blobエラーの扱い

例えば、
Axiosで
`responseType: 'blob'`を指定している場合、
エラーレスポンスも
Blobになる可能性がある。

必要に応じて、
`Content-Type`を確認し、

```text
application/json
```

の場合は
BlobをJSONへ変換する。

概念例：

```ts
const contentType =
  error.response?.headers[
    'content-type'
  ];

if (
  contentType?.includes(
    'application/json',
  )
) {
  const text =
    await error.response.data.text();

  const apiError =
    JSON.parse(
      text,
    );

  // 共通エラー処理
}
```

この処理は、
CSV-001など
他のダウンロードAPIでも
必要となるため、
可能であれば共通化する。

---

#### 2.18 利用者関連エラー

以下のエラーは、
API共通方針に従って処理する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
```

CSV-004専用画面だけで
独自の利用者エラー処理を
作成しない。

---

#### 2.19 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`の場合は、
CSVテンプレートを
取得できなかったことを
利用者へ表示する。

例えば、

```text
CSVテンプレートを取得できませんでした。
時間をおいて再度お試しください。
```

などの
共通サーバーエラー表示を使用する。

エラー時に
空のCSVファイルを
保存させない。

---

#### 2.20 エラー時はダウンロードしない

API呼び出しが失敗した場合は、
取得したレスポンスを
CSVファイルとして
保存しない。

特に、
JSONエラーレスポンスを

```text
month-end-holding-values-template.csv
```

として
誤って保存しないようにする。

---

#### 2.21 Object URLの解放

`URL.createObjectURL()`を
使用した場合は、
ダウンロード操作後に

```ts
URL.revokeObjectURL(
  url,
);
```

を実行する。

不要なObject URLを
ブラウザ上に残さない。

---

#### 2.22 キャッシュ

CSV-004の
テンプレート内容は固定であるが、
Phase1では
フロントエンド側で
特別なキャッシュ管理を
行う必要はない。

利用者が
テンプレート取得操作を行うたびに
CSV-004を実行してよい。

---

#### 2.23 冪等性

CSV-004は
冪等なGET APIであるため、
利用者が複数回
テンプレート取得ボタンを押しても
業務データへ影響しない。

フロントエンドでは、
重複登録防止のような
厳密な排他制御は不要である。

---

#### 2.24 React側でCSV仕様を検証しない

CSV-004は
テンプレート取得APIであり、
CSV入力値を検証するAPIではない。

React側で
テンプレート取得時に

- `target_year_month`
- `asset_account_name`
- `holding_asset_name`
- `value`

の入力ルールを
検証する必要はない。

利用者が編集したCSVの
正式な検証は、
CSV-005およびCSV-006で行う。

---

### 2 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [エラーコード一覧](../../error-codes.md)
- [CSV-005 商品別月末評価額CSVプレビュー](./csv-005-preview.md)
- [CSV-006 商品別月末評価額CSV登録](./csv-006-create.md)
- [CSV-001 月末資産残高CSVテンプレート取得](./csv-001-create.md)
- [CSV-002 月末資産残高CSVプレビュー](./csv-002-preview.md)
- [CSV-003 月末資産残高CSV登録](./csv-003-create.md)
- [SNP-003 月末資産状況詳細取得](../month-end-assets/snp-003-detail.md)
- [SNP-004 月末資産状況確定](../month-end-assets/snp-004-confirm.md)
- [VAL-001 商品別月末評価額一覧取得](../month-end-assets/val-001-list.md)
- [商品別月末評価額API詳細](../month-end-assets/README.md)
- [資産口座API詳細](../asset-accounts/README.md)
- [保有商品API詳細](../holding-assets/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)
- [Laravel CSVインポート設計](../../../architecture/laravel/csv-imports.md)
- [React CSVインポート設計](../../../architecture/react/csv-imports.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)