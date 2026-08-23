##  CSV-006 商品別月末評価額CSV登録

### 1 概要

CSV-006は、CSV-005 商品別月末評価額CSVプレビューで内容を確認したCSVファイルをバックエンドへ再送し、商品別月末評価額を一括登録するためのAPIである。

React・TypeScriptでは、CSV-005でプレビューに使用した同一の`File`を登録完了まで保持し、利用者が登録操作を行ったタイミングでCSV-006へ送信する。

CSV-005のプレビュー結果から登録データを再構築したり、`targetYearMonth`、`canImport`、`previewId`などを追加して送信したりせず、元のCSVファイルのみを`FormData`へ設定する。

```text
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
       利用者が内容確認
          ↓
       CSV-006
       同じFileを再送
          ↓
       最新状態で再検証
          ↓
       一括登録
          ↓
       関連Query再取得
```

CSV-006は業務データを更新するPOST APIであるため、TanStack QueryではMutationとして扱う。

登録ボタンは、CSVファイルが選択され、CSV-005によるプレビューが完了し、`canImport = true`であり、CSV-006を実行中でない場合に有効化する。

ただし、CSV-005で取得した`canImport = true`はCSV-006の成功を保証するものではない。

CSV-005とCSV-006の間に、月末資産状況の確定、既存商品別月末評価額の登録、資産口座や保有商品の状態変更などが発生する可能性があるため、CSV-006では実行時点の最新状態をバックエンドで再検証する。

```text
CSV-005
canImport = true
    ↓
業務状態が変更される可能性
    ↓
CSV-006
最新状態で再検証
    ├─ 登録可能
    │     ↓
    │   一括登録
    │
    └─ 登録不可
          ↓
       409 / 422等
```

CSV-006が成功した場合は、レスポンスとして返却される`targetYearMonth`および`importedCount`を登録完了表示に使用する。

`importedCount`はバックエンドが実際に登録した件数を正とし、React側でCSV行数から再計算しない。

登録成功後は、保持していた`File`およびCSV-005のプレビュー結果を破棄し、古い`canImport = true`の状態から同じCSVを再登録できないようにする。

また、商品別月末評価額や月末資産状況など、CSV登録によって影響を受けるTanStack QueryのQuery Cacheをinvalidateし、最新状態をバックエンドから再取得する。

```text
CSV-006
登録成功
    ↓
File・Preview状態を破棄
    ↓
関連Query Cacheをinvalidate
    ↓
バックエンドから最新状態を再取得
```

CSV-006は非冪等な登録APIであるため、Mutationの自動Retryは原則として行わない。

通信エラーなどによってレスポンスを受信できなかった場合でも、バックエンドでは登録が完了している可能性がある。そのため、同じCSVファイルを自動的に再送せず、必要に応じて商品別月末評価額や月末資産状況を再取得して現在状態を確認する。

CSV-006で`409 Conflict`が返却された場合は、CSV-005実行後に業務状態が変化した競合として扱う。必要に応じて古いプレビュー状態を無効化し、最新状態の再取得やCSV-005の再実行を利用者へ促す。

`422 Unprocessable Entity`が返却された場合は、CSVファイルまたはCSV内容が登録条件を満たしていないものとして、`error.details`に含まれる行番号やフィールド、エラー内容を利用者が確認できる形で表示する。

Phase1では、CSVの正式な解析、資産口座・保有商品の存在確認、対象年月時点の有効性判定、既存データとの重複判定、月末資産状況の作成・確定状態判定、トランザクション制御などをReact側へ重複実装しない。

フロントエンドは、CSVファイルの選択、CSV-005によるプレビュー、利用者による内容確認、CSV-006の実行、Mutation状態管理、成功・エラー表示および関連Queryの再取得を担当し、最終的な業務整合性の保証はバックエンドへ集約する。

---

### 2. React・TypeScriptでの利用

CSV-006は、CSV-005 商品別月末評価額CSVプレビューで内容を確認した後に、同じCSVファイルを再送して商品別月末評価額を一括登録する際に使用する。 :contentReference[oaicite:0]{index=0}

CSV-006は業務データを変更するPOST APIであるため、TanStack Queryを使用する場合はMutationとして扱う。

概念的な利用フローは、以下とする。

```text
CSVファイル選択
    ↓
CSV-005
商品別月末評価額CSVプレビュー
    ↓
canImport = true
    ↓
プレビュー内容表示
    ↓
利用者が登録実行
    ↓
CSV-006
同じCSVファイルを再送
    ↓
サーバー側で再検証
    ↓
201 Created
    ↓
関連Query Cache無効化
    ↓
最新データ再取得
```

---

#### 2.1 TypeScript型

CSV-006では、`multipart/form-data`でCSVファイルを送信する。

API呼び出しに必要な値は、

```text
file
```

のみとする。

概念例：

```ts
export type ImportMonthEndHoldingValueCsvVariables = {
  file: File;
};
```

`userId`、`targetYearMonth`、`assetAccountId`、`holdingAssetId`などをMutation引数へ含めない。

---

#### 2.2 正常レスポンス型

CSV-006の正常レスポンスは、以下の型として定義する。

概念例：

```ts
export type MonthEndHoldingValueCsvImportResult = {
  targetYearMonth: string;
  importedCount: number;
};
```

API共通Envelopeを使用する場合は、以下のように定義する。

```ts
export type ImportMonthEndHoldingValueCsvResponse =
  ApiResponse<MonthEndHoldingValueCsvImportResult>;
```

レスポンス例：

```json
{
  "data": {
    "targetYearMonth": "2026-07",
    "importedCount": 2
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

---

#### 2.3 importedCount

`importedCount`は、実際に新規登録された

```text
month_end_holding_values
```

の件数として扱う。

型は、

```text
number
```

とする。

正常レスポンスでは1以上となる。

---

#### 2.4 targetYearMonth

`targetYearMonth`は、

```text
YYYY-MM
```

形式のstringとして扱う。

概念例：

```ts
targetYearMonth: string;
```

必要に応じて、画面表示時に

```text
2026-07
    ↓
2026年7月
```

などへ変換する。

---

#### 2.5 API Client

CSV-006を呼び出す専用API Client関数を定義する。

概念例：

```ts
export const importMonthEndHoldingValueCsv =
  async (
    file: File,
  ): Promise<MonthEndHoldingValueCsvImportResult> => {
    const formData =
      new FormData();

    formData.append(
      'file',
      file,
    );

    const response =
      await apiClient.post<
        ImportMonthEndHoldingValueCsvResponse
      >(
        '/api/v1/month-end-holding-values/imports',
        formData,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

#### 2.6 Content-Typeを手動設定しない

CSV-006では、`FormData`を使用する。

そのため、以下のように

```ts
headers: {
  'Content-Type': 'multipart/form-data',
}
```

を手動設定しないことを基本とする。

ブラウザまたはHTTPクライアントに

```text
boundary
```

を含む`Content-Type`生成を任せる。

概念的には、

```ts
await apiClient.post(
  '/api/v1/month-end-holding-values/imports',
  formData,
);
```

とする。

---

#### 2.7 X-User-Id

`X-User-Id`は、CSV-006専用処理ではなく、共通API Clientから付与する。

概念例：

```ts
apiClient.interceptors.request.use(
  (config) => {
    config.headers['X-User-Id'] =
      currentUserId;

    return config;
  },
);
```

CSVアップロードComponentやMutation Hookから直接設定しない。

---

#### 2.8 userIdをMutation引数へ含めない

以下のようなMutation関数にはしない。

```ts
importMonthEndHoldingValueCsv(
  userId,
  file,
);
```

利用者IDは、共通API Clientが

```text
X-User-Id
```

として付与する。

CSV-006固有の入力は、

```text
file
```

だけとする。

---

#### 2.9 Mutationとして扱う

CSV-006は商品別月末評価額を新規登録するため、TanStack QueryではMutationとして扱う。

概念例：

```ts
export const useImportMonthEndHoldingValueCsv =
  () => {
    return useMutation({
      mutationFn:
        ({
          file,
        }: ImportMonthEndHoldingValueCsvVariables) =>
          importMonthEndHoldingValueCsv(
            file,
          ),
    });
  };
```

Queryとして実装しない。

---

#### 2.10 CSV-005との責務分離

CSV-005は、CSV内容のプレビューおよび登録可否確認を行う。

CSV-006は、実際の登録処理を行う。

概念的には、

```text
CSV-005
    → Queryではなく
      プレビュー用POST
    → 業務データ更新なし

CSV-006
    → Mutation
    → 業務データ更新あり
```

とする。

HTTPメソッドがどちらもPOSTであっても、フロントエンド上の責務は明確に分離する。

---

#### 2.11 CSV-005と同じFileを保持する

CSV-005成功後にCSV-006を実行するため、選択された

```text
File
```

を登録完了またはファイル再選択まで保持する。

概念的には、

```ts
const [
  selectedFile,
  setSelectedFile,
] = useState<File | null>(
  null,
);
```

とする。

CSV-005のプレビュー結果だけを保持し、元のFileを破棄しない。

---

#### 2.12 previewIdを保持しない

CSV-006では、

```text
previewId
previewToken
```

を使用しない。

そのため、React側でもCSV-005実行後に登録用IDを保持する設計にはしない。

概念的には、

```text
CSV-005
    ↓
Preview Result
+
元のFile
    ↓
CSV-006
元のFileを再送
```

とする。

---

#### 2.13 canImportを登録保証として扱わない

CSV-005で

```text
canImport === true
```

の場合に登録ボタンを表示・活性化してよい。

ただし、

```text
canImport = true
    =
CSV-006も必ず成功する
```

とは扱わない。

CSV-005とCSV-006の間に業務状態が変更される可能性があるためである。

---

#### 2.14 登録ボタン

CSV-005の結果が

```text
canImport = true
```

の場合にのみ、登録ボタンを活性化してよい。

概念例：

```tsx
<button
  type="button"
  disabled={
    !preview.data?.canImport ||
    importMutation.isPending
  }
  onClick={
    handleImport
  }
>
  登録する
</button>
```

ただし、最終的な登録可否はCSV-006のバックエンド処理で保証する。

---

#### 2.15 CSV-005未実行でのCSV-006送信をフロントでは防いでよい

通常の画面フローでは、

```text
ファイル選択
    ↓
CSV-005
    ↓
内容確認
    ↓
CSV-006
```

とする。

そのため、CSV-005未実行状態では登録ボタンを表示しない、または非活性にしてよい。

ただし、CSV-006自体はCSV-005の実行履歴に依存せず単体で安全に検証できるAPIとする。

---

#### 2.16 handleImport

概念例：

```ts
const handleImport =
  async (): Promise<void> => {
    if (
      selectedFile === null
    ) {
      return;
    }

    await importMutation.mutateAsync({
      file:
        selectedFile,
    });
  };
```

CSV内容をReact側で再構築して送信しない。

選択済みの元CSVファイルを送信する。

---

#### 2.17 FormDataへ余分な値を追加しない

以下のような実装にはしない。

```ts
formData.append(
  'targetYearMonth',
  preview.targetYearMonth,
);

formData.append(
  'canImport',
  'true',
);

formData.append(
  'previewId',
  preview.id,
);
```

CSV-006へ送信する業務項目は、

```text
file
```

だけとする。

---

#### 2.18 CSV内容をクライアントで書き換えない

CSV-005のプレビュー結果をもとに、React側で新しいCSVを生成し直してCSV-006へ送信しない。

概念的には、

```text
利用者が選択したCSV
    ↓
CSV-005

同じFile
    ↓
CSV-006
```

とする。

---

#### 2.19 二重送信防止

CSV-006実行中は、登録ボタンを非活性化する。

概念例：

```tsx
<button
  disabled={
    importMutation.isPending
  }
>
  {importMutation.isPending
    ? '登録中...'
    : '登録する'}
</button>
```

これにより、意図しない連続クリックを抑制する。

---

#### 2.20 フロントエンドの二重送信防止だけに依存しない

ボタン非活性化は、UX上の重複送信防止である。

バックエンドでは、

```text
重複確認
トランザクション
UNIQUE制約
```

によって最終的な整合性を保証する。

React側の制御をデータ整合性保証とはしない。

---

#### 2.21 Mutation中の画面操作

CSV-006実行中は、少なくとも以下を制御してよい。

- 登録ボタンを非活性化
- CSVファイル再選択を非活性化
- 再プレビューボタンを非活性化

登録中に対象Fileが切り替わらないようにする。

---

#### 2.22 成功時

CSV-006が成功した場合は、

```http
201 Created
```

として扱う。

Mutation成功時に、例えば

```text
2026年7月の商品別月末評価額を
2件登録しました。
```

などの完了表示を行ってよい。

表示値には、

```text
targetYearMonth
importedCount
```

を使用する。

---

#### 2.23 成功メッセージ

概念例：

```ts
const message =
  `${formatYearMonth(
    result.targetYearMonth,
  )}の商品別月末評価額を`
  + `${result.importedCount}件登録しました。`;
```

文言は、画面設計を正とする。

---

#### 2.24 登録後にプレビュー状態を破棄する

CSV-006成功後は、同じCSVを誤って再登録しないよう、

```text
selectedFile
previewResult
```

をクリアしてよい。

概念例：

```ts
setSelectedFile(
  null,
);

setPreviewResult(
  null,
);
```

画面遷移する場合は、遷移によって状態が破棄されてもよい。

---

#### 2.25 成功後の画面遷移

CSV登録成功後は、例えば

```text
月末資産状況詳細画面
```

へ遷移してよい。

レスポンスには`snapshotId`が含まれないため、遷移方法は画面設計に応じて決定する。

例えば、`targetYearMonth`を使って対象年月の一覧・詳細へ戻る設計としてよい。

---

#### 2.26 成功後に登録内容をレスポンスから再構築しない

CSV-006成功レスポンスには、

```text
targetYearMonth
importedCount
```

しか含まれない。

そのため、成功レスポンスだけから商品別評価額一覧をローカルCacheへ手動追加しない。

登録後は参照APIを再取得することを基本とする。

---

#### 2.27 VAL-001のCache無効化

CSV-006成功後は、商品別月末評価額一覧が変更されている。

対象Snapshotをフロント側で特定できる場合は、VAL-001のQuery Cacheを無効化する。

概念的には、

```ts
await queryClient.invalidateQueries({
  queryKey:
    monthEndHoldingValueKeys.all,
});
```

または、対象Snapshotが分かる場合はより限定したQuery Keyを無効化する。

---

#### 2.28 snapshotIdがレスポンスにない場合

CSV-006レスポンスには、

```text
snapshotId
```

を含めない。

そのため、対象SnapshotのIDを画面コンテキストとして既に保持していない場合は、`targetYearMonth`に関連する月末資産状況Queryを無効化して再取得する。

概念的には、

```text
CSV-006成功
    ↓
SNP系Query invalidate
    ↓
対象年月のSnapshot再取得
    ↓
VAL-001再取得
```

としてよい。

---

#### 2.29 月末資産状況Cacheも無効化する

CSV-006によって対象年月のsnapshotが新規作成される可能性がある。

そのため、月末資産状況一覧などの関連Query Cacheも無効化する。

例えば、

```ts
await queryClient.invalidateQueries({
  queryKey:
    monthEndAssetSnapshotKeys.all,
});
```

とする。

---

#### 2.30 資産推移系Cache

CSV-006による登録結果が資産状況・資産推移表示に影響する場合は、対象となる参照Queryも無効化してよい。

ただし、実際にどのQueryへ影響するかは各参照APIの仕様を正とする。

不要な全Query無効化は避ける。

---

#### 2.31 invalidateの基本方針

Phase1では、成功レスポンスから複雑にCacheを手動更新するより、

```text
CSV-006成功
    ↓
関連Query invalidate
    ↓
サーバーから最新状態再取得
```

を基本とする。

---

#### 2.32 CSV-005のPreview Cache

CSV-005のプレビュー結果をTanStack QueryのCacheとして保持している場合は、CSV-006成功後にそのPreview状態を破棄または無効化する。

登録後も

```text
canImport = true
```

の古いPreviewを表示し続けないようにする。

---

#### 2.33 CSV-005後にCSV-006が409になる場合

CSV-005で

```text
canImport = true
```

だったとしても、CSV-006で

```http
409 Conflict
```

となる場合がある。

主に以下である。

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED

MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

これは、CSV-005後にサーバー状態が変化した正常な競合ケースとして扱う。

---

#### 2.34 MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED

以下のエラーを受信した場合は、

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

例えば、

```text
対象年月の月末資産状況が
既に確定されているため、
登録できません。
```

などを表示する。

CSV-005のPreview結果を登録可能状態として表示し続けない。

---

#### 2.35 MONTH_END_HOLDING_VALUE_ALREADY_EXISTS

以下のエラーを受信した場合は、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

既に商品別月末評価額が登録されていることを利用者へ通知する。

既存値をCSV値で上書きする確認画面などは表示しない。

CSV-006には上書き機能がないためである。

---

#### 2.36 409発生後の再取得

競合エラーが発生した場合は、現在状態がプレビュー時点から変化している可能性が高い。

そのため、必要に応じて

```text
関連Query invalidate
+
CSV-005再実行を促す
```

としてよい。

---

#### 2.37 422エラー

CSVファイルやCSV内容に問題がある場合は、

```http
422 Unprocessable Entity
```

として扱う。

主なエラーコードは、以下とする。

```text
VALIDATION_ERROR
INVALID_CSV_FORMAT
CSV_DATA_REQUIRED
MULTIPLE_TARGET_YEAR_MONTHS
ASSET_ACCOUNT_NOT_FOUND
BALANCE_RECORDING_UNIT_MISMATCH
HOLDING_ASSET_NOT_FOUND
HOLDING_ASSET_NOT_AVAILABLE
INVALID_VALUE
DUPLICATE_HOLDING_ASSET_IN_CSV
```

---

#### 2.38 行単位エラー表示

`error.details`にCSV行番号が含まれる場合は、利用者が修正箇所を確認できる形で表示する。

概念的な型：

```ts
export type CsvImportErrorDetail = {
  rowNumber?: number;
  field?: string;
  code?: string;
  message: string;
};
```

---

#### 2.39 複数エラー表示

複数のCSVエラーが返却された場合は、最初の1件だけでなく可能な範囲で一覧表示する。

概念例：

```tsx
<ul>
  {error.details?.map(
    (
      detail,
      index,
    ) => (
      <li
        key={
          `${detail.rowNumber ?? 'general'}-${index}`
        }
      >
        {detail.message}
      </li>
    ),
  )}
</ul>
```

---

#### 2.40 rowNumber

`rowNumber`は、CSVファイル上の修正箇所を示すために使用する。

React側でDB上の行番号などとして扱わない。

---

#### 2.41 field

`field`が存在する場合は、例えば、

```text
target_year_month
asset_account_name
holding_asset_name
value
```

に対応する表示名へ変換してよい。

概念例：

```ts
const csvFieldLabels = {
  target_year_month:
    '対象年月',

  asset_account_name:
    '資産口座名',

  holding_asset_name:
    '保有商品名',

  value:
    '商品別月末評価額',
} as const;
```

---

#### 2.42 error.codeで処理を分岐する

フロントエンドでは、`error.message`ではなく、

```text
error.code
```

を基準としてエラー処理を分岐する。

概念例：

```ts
switch (error.code) {
  case 'MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED':
    // 確定済み
    break;

  case 'MONTH_END_HOLDING_VALUE_ALREADY_EXISTS':
    // 既存評価額あり
    break;

  case 'INVALID_CSV_FORMAT':
    // CSV形式不正
    break;

  default:
    // 共通エラー
    break;
}
```

---

#### 2.43 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

CSV-006専用Componentに同じ処理を重複実装しない。

---

#### 2.44 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、利用者コンテキストに関する共通エラーとして扱う。

---

#### 2.45 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択されている利用者が有効ではない状態として共通処理する。

---

#### 2.46 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
商品別月末評価額を
登録できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

#### 2.47 CSV-006は自動Retryしない

CSV-006は非冪等なPOST APIであり、サーバー側で登録済みかどうかをクライアントが判断できないケースがある。

そのため、TanStack QueryのMutationで自動Retryを原則として行わない。

概念例：

```ts
useMutation({
  mutationFn:
    importMonthEndHoldingValueCsv,

  retry: false,
});
```

---

#### 2.48 通信失敗時に安易に再送しない

CSV-006実行後に通信が切断された場合、

```text
サーバーでは登録成功
+
クライアントでは結果不明
```

となる可能性がある。

この状態で自動的に同じCSVを再送しない。

---

#### 2.49 通信結果不明時

レスポンスを受け取れなかった場合は、必要に応じて

```text
商品別月末評価額一覧
月末資産状況
```

を再取得し、現在状態を確認する導線を提供してよい。

Phase1では`Idempotency-Key`を使用しないため、同一Mutationの自動再送で初回結果を再現する設計にはしない。

---

#### 2.50 同じCSVを利用者が再送した場合

利用者が手動で同じCSVを再登録した場合は、バックエンドから

```text
409 Conflict
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

となる可能性がある。

React側で同じFileかどうかを比較して登録成功扱いにしない。

---

#### 2.51 フロントエンドで重複判定しない

React側で、

```text
このCSVは以前登録した
```

という履歴管理を行ってCSV-006の重複判定を代替しない。

正式な重複判定は、サーバー側の

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

に基づいて行う。

---

#### 2.52 CSVファイル選択

CSVファイル選択Componentでは、`accept`属性を設定してよい。

概念例：

```tsx
<input
  type="file"
  accept=".csv,text/csv"
  onChange={
    handleFileChange
  }
/>
```

ただし、`accept`はブラウザUI上の補助であり、ファイル検証の保証ではない。

最終検証はバックエンドで行う。

---

#### 2.53 Fileのnull確認

ファイル未選択状態では、CSV-005・CSV-006を実行しない。

概念例：

```ts
if (
  selectedFile === null
) {
  return;
}
```

ただし、バックエンド側の`file`必須検証も維持する。

---

#### 2.54 ファイルサイズのクライアント事前確認

CSV共通仕様の最大ファイルサイズがフロントエンドでも共有されている場合は、送信前に簡易チェックを行ってよい。

ただし、フロントエンドの検証だけを正式な制約にはしない。

バックエンド側でも必ず検証する。

---

#### 2.55 CSV内容をブラウザ側で正式検証しない

React側でCSVを読み込んで、

```text
ヘッダー
対象年月
資産口座
保有商品
value
```

などをバックエンドと同等に正式検証する必要はない。

登録可否の正は、CSV-005・CSV-006のバックエンド検証とする。

---

#### 2.56 フロント側で簡易表示してもよい

UX向上のため、選択したFileについて

```text
ファイル名
ファイルサイズ
```

などを表示してよい。

概念例：

```tsx
<p>
  {selectedFile.name}
</p>
```

ただし、CSV解析結果としてはCSV-005のレスポンスを正とする。

---

#### 2.57 プレビュー結果と選択Fileを紐づける

CSV-005実行後に利用者が別のCSVを選択した場合は、以前のPreview結果を破棄する。

概念的には、

```text
File A選択
    ↓
CSV-005 Preview A
    ↓
File B選択
    ↓
Preview A破棄
    ↓
CSV-005 Preview B
```

とする。

File Bを選択しているのにFile Aの

```text
canImport = true
```

を使って登録ボタンを活性化しない。

---

#### 2.58 CSV-006実行対象とPreview対象を一致させる

登録時には、CSV-005で確認したFileと現在選択されているFileが同じ状態であることを画面状態管理上保証する。

利用者がFileを変更した場合は、再プレビューを必須とするUIにしてよい。

---

#### 2.59 プレビュー後のFile内容変更

ブラウザの`File`オブジェクトは、選択時点のファイルを表す。

利用者がローカルファイルを編集した場合にブラウザ内のFileが自動更新されるとは限らない。

変更内容を登録したい場合は、再選択・再プレビューを行うUIとする。

---

#### 2.60 確認画面

CSV-005で取得したプレビュー内容を表示し、登録前に利用者が確認できるようにする。

例えば、

```text
対象年月
資産口座名
保有商品名
商品別月末評価額
```

を一覧表示してよい。

CSV-006成功レスポンスには明細が含まれないため、登録前確認はCSV-005の責務とする。

---

#### 2.61 登録確認ダイアログ

必要に応じて、CSV-006実行前に

```text
この内容で登録しますか？
```

という確認ダイアログを表示してよい。

ただし、バックエンドの再検証・競合検出は引き続き必要とする。

---

#### 2.62 登録中表示

CSV-006は同期APIであるため、Mutation中は

```text
登録中...
```

などの状態を表示する。

QueueやPolling前提の進捗UIはPhase1では不要とする。

---

#### 2.63 進捗率を表示しない

Phase1ではCSV-006を同期APIとして実装し、バックエンドから進捗情報を返さない。

そのため、

```text
45%
80%
```

などの登録進捗率を疑似的に表示しない。

単純なLoading状態とする。

---

#### 2.64 importedCountを利用した完了表示

成功時には、

```text
result.importedCount
```

を使用して、実際に登録された件数を表示する。

CSV行数をReact側で数えて登録件数として表示しない。

ヘッダーや空行等の扱いとずれる可能性があるためである。

---

#### 2.65 targetYearMonthも成功レスポンスを正とする

完了表示に使用する対象年月は、可能であれば

```text
result.targetYearMonth
```

を正とする。

CSVプレビュー結果の値だけに依存しない。

---

#### 2.66 Mutation Hookの責務

Mutation Hookでは、主に以下を担当する。

```text
CSV-006実行
Mutation状態管理
成功時Cache無効化
```

画面固有のダイアログやレイアウトをHookへ持たせない。

---

#### 2.67 onSuccess

概念例：

```ts
export const useImportMonthEndHoldingValueCsv =
  () => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn:
        ({
          file,
        }: ImportMonthEndHoldingValueCsvVariables) =>
          importMonthEndHoldingValueCsv(
            file,
          ),

      retry:
        false,

      onSuccess:
        async () => {
          await Promise.all([
            queryClient.invalidateQueries({
              queryKey:
                monthEndAssetSnapshotKeys.all,
            }),

            queryClient.invalidateQueries({
              queryKey:
                monthEndHoldingValueKeys.all,
            }),
          ]);
        },
    });
  };
```

実際のQuery Keyは、React共通設計を正とする。

---

#### 2.68 onErrorでToastを固定しない

共通Mutation Hookですべてのエラーを同一Toastへ変換すると、

```text
CSV行エラー
409競合
500エラー
```

の表示を分けにくくなる。

エラーオブジェクトをPageまたはエラー表示Componentへ渡し、必要な表示分岐を行ってよい。

---

#### 2.69 API Clientの責務

API Clientでは、以下を担当する。

```text
FormData生成
POST送信
型付きレスポンス取得
```

以下は担当しない。

- ファイル選択
- Preview表示
- 登録確認
- Toast
- 画面遷移
- Query Cache無効化
- CSV行エラー表示
- 登録ボタン制御

---

#### 2.70 Pageの責務

商品別月末評価額CSV登録Pageでは、主に以下を担当する。

- CSVファイル選択状態管理
- CSV-005プレビュー実行
- プレビュー結果表示
- `canImport`による登録ボタン制御
- CSV-006実行
- 登録中表示
- 登録成功表示
- CSVエラー表示
- 登録後の画面遷移

---

#### 2.71 FileInput Componentの責務

CSV FileInput Componentでは、主に以下を担当する。

- CSVファイル選択
- 選択ファイル名表示
- ファイル変更通知

以下は行わない。

- CSV-005実行
- CSV-006実行
- 業務ルール判定
- 資産口座確認
- 保有商品確認

---

#### 2.72 Preview Componentの責務

CSV-005で取得したプレビュー結果を表示する。

主に、

- 対象年月
- 資産口座名
- 保有商品名
- 商品別月末評価額
- CSVエラー
- `canImport`

などを表示する。

CSV-006の登録処理は行わない。

---

#### 2.73 Import Button Component

登録ボタンを独立Componentとする場合は、

```ts
type Props = {
  disabled: boolean;
  isPending: boolean;
  onClick: () => void;
};
```

など、表示・操作に必要な値だけを受け取る。

API ClientをButton Componentから直接呼び出さない。

---

#### 2.74 Error List Component

CSV行単位のエラー表示が複雑になる場合は、専用Componentへ分離してよい。

概念例：

```tsx
<CsvImportErrorList
  details={
    error.details
  }
/>
```

---

#### 2.75 エラー表示ではコードをそのまま見せない

例えば、

```text
HOLDING_ASSET_NOT_FOUND
```

をそのまま利用者向け画面へ表示するのではなく、必要に応じて分かりやすい文言へ変換する。

ただし、デバッグ・開発環境でコードを補助表示する方針がある場合は、共通設計に従う。

---

#### 2.76 CSV-005のエラーとCSV-006のエラーを区別する

CSV-005では、業務エラーがあっても

```text
200 OK
canImport = false
```

となる。

CSV-006では、登録不可の場合は

```text
4xx Error
```

となる。

React側で同じHTTP状態管理として扱わないよう注意する。

---

#### 2.77 CSV-005のcanImport = false

CSV-005で

```text
canImport = false
```

の場合は、CSV-006を実行しないUIとする。

利用者には、プレビューで返却されたエラー内容を修正してCSVを再選択・再プレビューするよう促す。

---

#### 2.78 CSV-005のcanImport = true後の409

CSV-006で409になった場合は、

```text
CSVファイル自体の入力不正
```

とは限らない。

プレビュー後に業務状態が変わった可能性があるため、

```text
最新状態が変更されたため、
再度プレビューしてください。
```

などの導線を設けてよい。

---

#### 2.79 422の場合はCSV修正を促す

422の場合は、CSV内容またはCSVが参照している業務データが登録条件を満たしていない。

可能であれば`error.details`を表示し、CSVの修正箇所を確認できるようにする。

---

#### 2.80 ファイルをサーバー保存済みとみなさない

CSV-005実行後も、サーバー側にCSVファイルが保持されているとは考えない。

CSV-006実行時には必ずブラウザ側のFileを再送する。

---

#### 2.81 ページ再読み込み

CSVファイルは、通常のブラウザ状態ではページ再読み込み後に保持できない。

そのため、ページ再読み込み後は、

```text
CSV再選択
    ↓
CSV-005再実行
    ↓
CSV-006
```

を基本とする。

FileをLocalStorage等へ保存しない。

---

#### 2.82 FileをLocalStorageへ保存しない

アップロードCSVをBase64等へ変換してLocalStorageへ永続保存しない。

Phase1では、選択中のFileを画面メモリ上でのみ保持する。

---

#### 2.83 CSV内容をログ出力しない

フロントエンド側でも、

```ts
console.log(
  file,
);

console.log(
  preview.rows,
);
```

などによって本番環境のConsoleへ資産情報を不要に出力しない。

---

#### 2.84 エラーオブジェクトにも注意する

HTTPクライアントのエラーオブジェクト全体を

```ts
console.error(
  error,
);
```

として本番環境へ常時出力すると、Request情報等が含まれる可能性がある。

ログ方針は、React共通設計に従う。

---

#### 2.85 概念的なディレクトリ構成

例えば、以下のように整理できる。

```text
features/
└── csv-imports/
    ├── api/
    │   ├── previewMonthEndHoldingValueCsv.ts
    │   └── importMonthEndHoldingValueCsv.ts
    ├── components/
    │   ├── CsvFileInput.tsx
    │   ├── MonthEndHoldingValueCsvPreview.tsx
    │   ├── CsvImportErrorList.tsx
    │   └── CsvImportButton.tsx
    ├── hooks/
    │   ├── usePreviewMonthEndHoldingValueCsv.ts
    │   └── useImportMonthEndHoldingValueCsv.ts
    ├── types/
    │   └── monthEndHoldingValueCsv.ts
    └── pages/
        └── MonthEndHoldingValueCsvImportPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

#### 2.86 CSV-005と型を共通化する

CSV-005とCSV-006で共通する概念については、型を共通化してよい。

例えば、

```text
CSV行エラー
対象年月
CSVフィールド名
```

などである。

一方、

```text
Preview Response
Import Response
```

は責務が異なるため、別の型として定義する。

---

#### 2.87 Import ResponseをPreview Responseから流用しない

以下のように、CSV-005レスポンス型をCSV-006へそのまま流用しない。

```ts
type ImportResponse =
  PreviewResponse;
```

CSV-006成功レスポンスは、

```text
targetYearMonth
importedCount
```

に限定されているため、専用型を定義する。

---

#### 2.88 CSV登録結果をView Modelへ変換してもよい

Phase1では、APIレスポンスをそのまま完了表示へ使用してよい。

将来的に表示要件が複雑になった場合は、

```text
Import Result
    ↓
View Model
    ↓
Completion Component
```

へ分離してよい。

現時点では不要な変換層を追加しない。

---

#### 2.89 フロントエンドで行わないこと

CSV-006のReact・TypeScript実装では、以下をフロントエンドの責務としない。

- 利用者境界の最終保証
- CSV仕様の最終検証
- CSVヘッダーの最終検証
- 資産口座存在確認
- 残高記録単位判定
- 保有商品存在確認
- 資産口座と保有商品の関連保証
- 対象年月時点の有効性判定
- snapshot存在確認
- snapshot作成
- snapshot確定状態の最終判定
- 既存商品別月末評価額の重複判定
- トランザクション制御
- ロック制御
- UNIQUE制約による重複防止
- importedCountの算出
- CSV-005の`canImport`を登録保証として扱うこと

フロントエンドは、

```text
CSV選択
    ↓
CSV-005でプレビュー
    ↓
利用者が内容確認
    ↓
同じFileをCSV-006へ送信
    ↓
Mutation状態管理
    ↓
成功・エラー表示
    ↓
関連Query再取得
```

という責務を基本とする。

---

### 3 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [CSV-001 月末資産残高CSVテンプレート取得](./csv-001-template.md)
- [CSV-002 月末資産残高CSVプレビュー](./csv-002-preview.md)
- [CSV-003 月末資産残高CSV登録](./csv-003-create.md)
- [CSV-004 商品別月末評価額CSVテンプレート取得](./csv-004-template.md)
- [CSV-005 商品別月末評価額CSVプレビュー](./csv-005-preview.md)
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