##  CSV-002 月末資産残高CSVプレビュー

### 1. 概要

CSV-002では、月末資産残高CSVをアップロードし、登録前にCSV内容および業務ルールを検証するためのLaravel実装方針を定義する。

本APIでは、操作対象利用者から受け取ったCSVファイルを解析し、各行について月末資産残高として登録可能な内容であるかを検証したうえで、登録予定内容およびエラー内容をプレビューとして返却する。

主に以下を検証対象とする。

* CSVファイル形式
* CSVヘッダー
* 対象年月
* 資産口座名
* 月末残高
* 1ファイル内の対象年月統一
* 資産口座の存在
* 資産口座の利用者境界
* 残高記録単位
* 対象年月時点での資産口座の有効性
* 月末資産状況の状態
* 既存月末資産残高との重複
* CSV内の重複

CSVプレビューでは登録可否の検証のみを行い、月末資産残高をデータベースへ登録しない。また、月末資産状況、資産口座、利用可能資産設定などの業務データも更新しない。

Laravel側では、以下の責務を分離して実装する。

```text
Route
    ↓
Middleware
    ↓
Request
    ↓
Action
    ↓
UseCase
    ├─ CSV Parser
    ├─ CSV Validator
    ├─ AssetAccountQuery
    ├─ MonthEndAssetSnapshotQuery
    └─ MonthEndAssetBalanceQuery
    ↓
Preview DTO
    ↓
API Resource
    ↓
Responder
```

Requestでは、CSVファイルの必須チェック、ファイル形式、ファイルサイズなど、HTTPリクエストレベルで判定可能な内容のみを検証する。

CSV Parserでは、CSVファイルの読み込み、BOM処理、ヘッダー取得、行解析、行番号管理などを担当し、CSV Validatorでは、`target_year_month`、`asset_account_name`、`balance`などの行単位の入力値を検証する。

UseCaseでは、CSV解析結果とQueryから取得した業務データを組み合わせ、対象年月、資産口座、残高記録単位、対象年月時点での有効性、月末資産状況、既存月末資産残高、CSV内重複などを検証する。データベースアクセスはQueryへ委譲し、CSV行ごとの個別SQL発行を避け、必要な業務データを一括取得したうえでメモリ上で検証する。

CSV構造が正常に解析できる場合は、可能な範囲で複数の入力エラーおよび業務エラーを収集し、CSV全体エラーと行単位エラーをPreview DTOへ保持する。`canImport`は、CSV全体エラーおよび行エラーが1件も存在しない場合のみ`true`とする。

CSV解析不能やCSVヘッダー不正など、プレビュー処理そのものを継続できない状態はHTTPエラーとして扱う。一方、資産口座不存在、残高不正、CSV内重複、確定済み月末資産状況など、CSV内容に対する入力・業務上の問題はHTTPエラーとせず、`200 OK`のプレビュー結果内にエラー情報として返却する。

CSV-002では業務データを更新しないため、Repository、トランザクションおよび行ロックは使用しない。プレビュー結果についてもPhase1ではデータベースへ保存せず、CSV-003 月末資産残高CSV登録時にCSVファイルを再送し、最新の業務データを使用して登録可否を再検証する。

CSV-001、CSV-002、CSV-003では、CSVヘッダーなどのCSV仕様を共通Definitionへ集約する。また、CSV-002とCSV-003ではCSV Parser、CSV Validatorおよび登録可否に関する業務検証ロジックを可能な限り共通化し、プレビュー時と登録時で判定ルールが乖離しない構成とする。


---

### 2 Laravel実装方針

CSV-002では、
Action、
Request、
UseCase、
CSV Definition、
CSV Parser、
CSV Validator、
Query、
DTO、
API Resource、
Responderを分離して実装する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
Middleware
    ↓
Request
    ↓
Action
    ↓
UseCase
    ├─ CSV Parser
    ├─ CSV Validator
    ├─ AssetAccountQuery
    ├─ MonthEndAssetSnapshotQuery
    └─ MonthEndAssetBalanceQuery
    ↓
Preview DTO
    ↓
API Resource
    ↓
Responder
```

CSVプレビューに関するユースケース制御は、UseCaseへ集約する。

Actionへ、以下の処理を直接記述しない。

- CSV解析
- CSVヘッダー検証
- 行単位バリデーション
- 資産口座検索
- 月末資産状況検索
- 既存月末資産残高検索
- 業務ルール判定
- `canImport`判定

---

#### 2.1 Action

HTTPリクエストを受け付け、検証済みのCSVファイルおよび利用者コンテキストを取得する。

CSVプレビューUseCaseを呼び出し、取得結果をResponderへ渡す。

概念例：

```php
final class PreviewMonthEndAssetBalanceCsvAction
{
    public function __invoke(
        PreviewMonthEndAssetBalanceCsvRequest $request,
        PreviewMonthEndAssetBalanceCsvUseCase $useCase,
        MonthEndAssetBalanceCsvPreviewResponder $responder,
        userContext $userContext,
    ): JsonResponse {
        $preview =
            $useCase->execute(
                userId:
                    $userContext->userId,

                file:
                    $request->file('file'),
            );

        return $responder->ok(
            $preview,
        );
    }
}
```

Actionでは、以下を行わない。

- CSVヘッダー検証
- CSV行解析
- `target_year_month`の検証
- `asset_account_name`の検証
- `balance`の検証
- 1ファイル1対象年月の判定
- 資産口座検索
- 残高記録単位判定
- 対象年月時点の資産口座有効性判定
- 月末資産状況検索
- 確定状態判定
- 既存月末資産残高検索
- CSV内重複判定
- `canImport`判定
- レスポンス形式への変換

Actionは、UseCaseの呼び出しとResponderへの受け渡しに責務を限定する。

---

#### 2.2 Request

Requestでは、HTTPリクエストとしてCSVファイルを受け付けられる状態かを検証する。

概念例：

```php
final class PreviewMonthEndAssetBalanceCsvRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'file' => [
                'required',
                'file',
                'mimes:csv,txt',
                'max:' . config(
                    'csv.max_file_size_kb',
                ),
            ],
        ];
    }
}
```

実際の以下の扱いは、CSV共通仕様に従う。

- ファイルサイズ上限
- MIME Type
- 拡張子

Laravelのアップロードファイル検証だけに依存せず、CSV Parserでも実際にCSVとして解析可能であることを確認する。

---

#### 2.3 Requestで行うこと

Requestでは、主に以下を検証する。

- `file`が指定されていること
- HTTPアップロードファイルであること
- 許可されたファイル形式であること
- ファイルサイズ上限以内であること

これらは、CSV内容を解析する前に判定できるHTTPリクエストレベルのバリデーションとする。

---

#### 2.4 Requestで行わないこと

Requestでは、以下のCSV内容および業務ルール検証を行わない。

- CSVヘッダー検証
- CSVデータ行の存在確認
- `target_year_month`必須確認
- `target_year_month`形式確認
- 1ファイル1対象年月確認
- `asset_account_name`必須確認
- `balance`必須確認
- `balance`整数確認
- `balance`0以上確認
- 資産口座存在確認
- 残高記録単位確認
- 対象年月時点の資産口座有効性確認
- 月末資産状況確認
- 確定状態確認
- 既存月末資産残高確認
- CSV内重複確認

これらは、UseCase、CSV Validator、Queryの責務とする。

---

#### 2.5 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONレスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証し、操作対象利用者を特定する。

概念的な処理は、以下とする。

```text
X-User-Id取得
    ↓
必須チェック
    ↓
形式チェック
    ↓
users存在確認
    ↓
利用者コンテキスト設定
    ↓
Request
    ↓
Action
```

CSV解析処理へ進む前に、有効な操作対象利用者が確定していることを前提とする。

---

#### 2.6 UseCase

CSVプレビューのユースケース処理を担当する。

概念例：

```php
final class PreviewMonthEndAssetBalanceCsvUseCase
{
    public function execute(
        int $userId,
        UploadedFile $file,
    ): MonthEndAssetBalanceCsvPreview {
        // CSV解析
        // CSV構造検証
        // 行入力値検証
        // 対象年月特定
        // 業務データ取得
        // 業務ルール検証
        // Preview DTO生成
    }
}
```

UseCaseでは、概念的に以下の順序で処理する。

```text
CSV解析
    ↓
ヘッダー検証
    ↓
データ行抽出
    ↓
行入力値検証
    ↓
対象年月特定
    ↓
CSV内重複確認
    ↓
必要な業務データ一括取得
    ↓
業務ルール検証
    ↓
CSV全体エラー集約
    ↓
行エラー集約
    ↓
canImport判定
    ↓
Preview DTO生成
```

UseCaseでは、SQLやEloquent Query Builderを直接組み立てない。

データベースアクセスは、Queryへ委譲する。

---

#### 2.7 CSV Definition

CSV-001、CSV-002、CSV-003で使用する月末資産残高CSV仕様は、共通Definitionへ集約する。

概念例：

```php
final class MonthEndAssetBalanceCsvDefinition
{
    public const HEADERS = [
        'target_year_month',
        'asset_account_name',
        'balance',
    ];
}
```

CSV-002では、このDefinitionを使用してヘッダーを検証する。

以下のように、CSV-001、CSV-002、CSV-003で個別にヘッダーを定義しない。

```text
CSV-001
    ↓
MonthEndAssetBalanceCsvDefinition

CSV-002
    ↓
MonthEndAssetBalanceCsvDefinition

CSV-003
    ↓
MonthEndAssetBalanceCsvDefinition
```

---

#### 2.8 CSV Parser

CSVファイルの読み込みは、専用Parserへ分離する。

概念例：

```php
final class MonthEndAssetBalanceCsvParser
{
    public function parse(
        UploadedFile $file,
    ): ParsedCsv {
        // CSV解析
    }
}
```

Parserは、CSVファイルを構造化された生データへ変換する責務を持つ。

Parserでは、業務データの検索や登録可否判定を行わない。

---

#### 2.9 CSV Parserの責務

CSV Parserでは、主に以下を行う。

- ファイルオープン
- BOM除去
- CSV行読み込み
- ヘッダー取得
- 行番号管理
- 完全な空行の除外
- CSVとして解析不能な状態の検出

Parserは、各行を概念的に以下の形式へ変換する。

```php
final readonly class ParsedCsvRow
{
    public function __construct(
        public int $rowNumber,

        /** @var array<string, string|null> */
        public array $values,
    ) {
    }
}
```

CSV上の行番号を保持することで、プレビュー結果の`rowNumber`として利用できるようにする。

---

#### 2.10 CSV構造異常

以下のように、CSVそのものを正常に解析できない場合は、プレビュー結果ではなくHTTPエラーとして扱う。

- 空ファイル
- ヘッダー取得不能
- CSV解析不能
- 想定外の列構造

概念的には、

```text
CSV Parser
    ↓
InvalidCsvFormatException
    ↓
共通Exception Handler
    ↓
422 Unprocessable Entity
INVALID_CSV_FORMAT
```

とする。

---

#### 2.11 CSVヘッダー検証

CSVヘッダーは、共通Definitionと完全一致することを確認する。

概念例：

```php
if (
    $parsedCsv->headers
    !== MonthEndAssetBalanceCsvDefinition::HEADERS
) {
    throw new InvalidCsvFormatException();
}
```

以下を不正とする。

- ヘッダー不足
- ヘッダー名不一致
- ヘッダー順序不一致
- 余分なヘッダー

ヘッダーが不正な場合は、後続の行検証へ進まない。

---

#### 2.12 CSV Validator

CSV各行の入力値検証は、専用Validatorへ分離する。

概念例：

```php
final class MonthEndAssetBalanceCsvValidator
{
    public function validateRow(
        ParsedCsvRow $row,
    ): CsvRowValidationResult {
        // 行入力値検証
    }
}
```

主に以下を確認する。

```text
target_year_month
    required
    YYYY-MM

asset_account_name
    required

balance
    required
    integer
    min:0
```

CSV Validatorでは、データベース検索を行わない。

---

#### 2.13 行入力値の正規化

CSV入力値は、業務データ検索やDTO生成に使用する前に、必要な型へ変換する。

例えば、`balance`は正常な整数形式の場合のみintegerへ変換する。

概念例：

```php
$balance =
    filter_var(
        $rawBalance,
        FILTER_VALIDATE_INT,
    );
```

PHPの暗黙的な型変換によって、

```text
1000abc
```

などを正常値として扱ってはならない。

---

#### 2.14 0円の扱い

`balance = 0`は、有効な月末残高として扱う。

以下のようなtruthy / falsy判定を使用しない。

```php
if (! $balance) {
    // 0円まで未入力扱いになるため使用しない
}
```

必須判定と数値判定を明確に分離する。

---

#### 2.15 データ行0件

CSVヘッダーは正常だが、データ行が1件も存在しない場合は、CSV構造異常として例外にはしない。

CSV全体エラーとして、

```text
CSV_DATA_REQUIRED
```

を生成する。

概念的には、

```text
CSV解析成功
+
データ行0件
    ↓
200 OK
canImport = false
```

とする。

---

#### 2.16 対象年月の特定

各行の`target_year_month`が正常な形式の場合、CSV内の対象年月を収集する。

概念例：

```php
$targetYearMonths =
    collect($validRows)
        ->pluck(
            'targetYearMonth',
        )
        ->unique()
        ->values();
```

2種類以上存在する場合は、CSV全体エラーとして

```text
MULTIPLE_TARGET_YEAR_MONTHS
```

を生成する。

対象年月を1件に特定できない場合は、Preview DTOの

```text
targetYearMonth
```

を`null`としてよい。

---

#### 2.17 CSV内重複判定

同一CSV内で同じ対象年月かつ同じ資産口座が複数回指定されていないか確認する。

概念的なキーは、以下とする。

```text
targetYearMonth
+
assetAccountName
```

1ファイル1対象年月が成立している場合は、実質的に`assetAccountName`単位で重複確認してよい。

重複時は、

```text
DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

相当の行エラーを生成する。

先勝ち、後勝ち、残高合算などの暗黙的な解決は行わない。

---

#### 2.18 AssetAccountQuery

CSV内で指定された資産口座名について、操作対象利用者に属する資産口座をまとめて取得する。

概念例：

```php
$assetAccounts =
    $this->assetAccountQuery
        ->findActiveByNames(
            userId: $userId,
            names: $assetAccountNames,
        );
```

検索条件は、概念的に以下とする。

```text
user_id = 操作対象利用者ID
AND
name IN (...)
AND
deleted_at IS NULL
```

他の利用者に属する同名資産口座を取得対象へ含めない。

---

#### 2.19 資産口座のMap化

取得した資産口座は、名前をキーとしてMap化してよい。

概念例：

```php
$assetAccountMap =
    $assetAccounts->keyBy(
        'name',
    );
```

各CSV行では、

```php
$assetAccount =
    $assetAccountMap->get(
        $row->assetAccountName,
    );
```

として参照する。

CSV行ごとに同じ資産口座検索SQLを発行しない。

---

#### 2.20 資産口座不存在

操作対象利用者に該当する資産口座が存在しない場合は、HTTP 404にはしない。

行単位のプレビューエラーとして、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を生成する。

他の利用者にのみ同名資産口座が存在する場合も、同じ扱いとする。

---

#### 2.21 残高記録単位の判定

取得した資産口座について、`balance_recording_unit`が口座単位であることを確認する。

概念例：

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::ACCOUNT
) {
    // BALANCE_RECORDING_UNIT_MISMATCH
}
```

実際のEnum値は、共通定義に従う。

商品単位の資産口座は、月末資産残高CSVの登録対象としない。

---

#### 2.22 対象年月時点の資産口座有効性

資産口座がCSVの`targetYearMonth`時点で月末資産残高の記録対象として有効かを判定する。

現在日時を基準に判定してはならない。

必ず、

```text
CSVのtargetYearMonth
```

を基準とする。

具体的な判定方法は、資産口座および関連するテーブル定義・機能要件に従う。

対象年月時点で無効な場合は、

```text
ASSET_ACCOUNT_NOT_AVAILABLE
```

相当の行エラーを生成する。

---

#### 2.23 MonthEndAssetSnapshotQuery

対象年月を単一に特定できた場合は、操作対象利用者に属する月末資産状況を取得する。

概念例：

```php
$snapshot =
    $this->monthEndAssetSnapshotQuery
        ->findByUserAndTargetYearMonth(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,
        );
```

検索条件は、以下とする。

```text
user_id = 操作対象利用者ID
AND
target_year_month = targetYearMonth
```

他利用者の同一対象年月データを取得しない。

---

#### 2.24 月末資産状況不存在

対象年月の月末資産状況が存在しない場合は、CSV-003と同一の業務ルールで判定する。

CSV-003で登録時に月末資産状況を作成する仕様であれば、CSV-002では不存在のみを理由としてプレビューエラーを生成しない。

ただし、CSV-002では実際に

```text
month_end_asset_snapshots
```

を作成しない。

---

#### 2.25 確定状態判定

対象年月の月末資産状況が存在する場合は、`confirmed`を確認する。

概念例：

```php
if (
    $snapshot !== null
    && $snapshot->confirmed
) {
    // CSV全体を登録不可とする
}
```

確定済みの場合は、CSV全体エラーとして

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

を生成する。

この状態は、HTTPエラーとしてではなく、

```text
200 OK
canImport = false
```

のプレビュー結果として扱う。

---

#### 2.26 MonthEndAssetBalanceQuery

対象snapshotが既に存在する場合は、CSV対象資産口座について既存月末資産残高をまとめて取得する。

概念例：

```php
$existingBalances =
    $this->monthEndAssetBalanceQuery
        ->findBySnapshotAndAssetAccounts(
            userId:
                $userId,

            snapshotId:
                $snapshot->id,

            assetAccountIds:
                $assetAccountIds,
        );
```

`snapshotId`だけではなく、操作対象利用者との利用者境界も保証する。

---

#### 2.27 既存残高のMap化

既存の月末資産残高は、`asset_account_id`をキーとしてMap化してよい。

概念例：

```php
$existingBalanceMap =
    $existingBalances->keyBy(
        'asset_account_id',
    );
```

各CSV行について、既存データの有無をメモリ上で判定する。

---

#### 2.28 既存月末資産残高

同一対象年月、同一資産口座について既に月末資産残高が存在し、CSV-003が新規登録のみを許可する仕様の場合は、

```text
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

相当の行エラーを生成する。

CSV-002では、既存レコードを以下のように変更しない。

- 更新
- 削除
- 上書き

---

#### 2.29 N+1問題

CSV-002では、CSV行ごとに個別SQLを発行してはならない。

以下のような構成を避ける。

```text
1行目
    ↓
asset_accounts検索
snapshot検索
balance検索

2行目
    ↓
asset_accounts検索
snapshot検索
balance検索

3行目
    ↓
...
```

基本的には、

```text
CSV全体解析
    ↓
必要な資産口座名収集
    ↓
資産口座一括取得
    ↓
snapshot 1件取得
    ↓
既存残高一括取得
    ↓
メモリ上で業務検証
```

とする。

---

#### 2.30 CsvPreviewError DTO

プレビューで使用するエラーは、専用DTOとして表現する。

概念例：

```php
final readonly class CsvPreviewError
{
    public function __construct(
        public string $code,
        public string $message,
    ) {
    }
}
```

CSV全体エラーと行エラーで同じ構造を使用してよい。

---

#### 2.31 Preview Row DTO

各CSV行のプレビュー結果は、専用DTOとして表現する。

概念例：

```php
final readonly class MonthEndAssetBalanceCsvPreviewRow
{
    public function __construct(
        public int $rowNumber,
        public ?string $assetAccountName,
        public ?int $balance,

        /** @var CsvPreviewError[] */
        public array $errors,
    ) {
    }
}
```

CSV内部の以下の項目などは、Preview Row DTOへ含めない。

- `asset_account_id`
- `month_end_asset_snapshot_id`

---

#### 2.32 Preview DTO

プレビュー結果全体は、以下のようなDTOとして表現する。

概念例：

```php
final readonly class MonthEndAssetBalanceCsvPreview
{
    public function __construct(
        public ?string $targetYearMonth,
        public bool $canImport,

        /** @var CsvPreviewError[] */
        public array $errors,

        /** @var MonthEndAssetBalanceCsvPreviewRow[] */
        public array $rows,
    ) {
    }
}
```

Eloquent ModelやQuery結果をそのままResponderへ渡さない。

---

#### 2.33 canImportの判定

`canImport`は、CSV全体エラーおよび各行エラーをもとにUseCaseで判定する。

概念例：

```php
$hasGlobalErrors =
    count(
        $globalErrors,
    ) > 0;

$hasRowErrors =
    collect(
        $rows,
    )->contains(
        static fn (
            MonthEndAssetBalanceCsvPreviewRow $row,
        ): bool =>
            count(
                $row->errors,
            ) > 0,
    );

$canImport =
    ! $hasGlobalErrors
    && ! $hasRowErrors;
```

正常行が1件以上存在していても、エラーが1件でも存在する場合は、

```text
canImport = false
```

とする。

---

#### 2.34 複数エラー収集

CSV構造が正常に解析可能な場合は、可能な範囲で複数エラーを収集する。

そのため、1行目の業務エラーで即座に例外を送出して処理全体を終了する方式は基本としない。

以下のような実装は避ける。

```php
throw new AssetAccountNotFoundException();
```

代わりに、プレビュー用エラーDTOへエラーを追加する。

概念的には、

```text
2行目
    ASSET_ACCOUNT_NOT_FOUND

4行目
    INVALID_BALANCE

6行目
    DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

を1回のレスポンスで返却できるようにする。

---

#### 2.35 HTTPエラーとプレビューエラーの分離

以下は、HTTPエラーとして扱う。

- 利用者コンテキスト不正
- `file`未指定
- アップロードファイル不正
- ファイルサイズ超過
- CSV解析不能
- CSVヘッダー不正
- 想定外例外

一方、以下はプレビュー結果内の業務エラーとして扱う。

- データ行0件
- `target_year_month`不正
- 対象年月混在
- `asset_account_name`不正
- `balance`不正
- 資産口座不存在
- 残高記録単位不一致
- 対象年月時点の資産口座無効
- CSV内重複
- 確定済み月末資産状況
- 既存月末資産残高

UseCaseでは、この2種類を明確に分離する。

---

#### 2.36 API Resource

Preview DTOを、専用API ResourceによってAPIレスポンス形式へ変換する。

概念例：

```php
final class MonthEndAssetBalanceCsvPreviewResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'targetYearMonth'
                => $this->targetYearMonth,

            'canImport'
                => $this->canImport,

            'errors'
                => CsvPreviewErrorResource::collection(
                    $this->errors,
                ),

            'rows'
                => MonthEndAssetBalanceCsvPreviewRowResource::collection(
                    $this->rows,
                ),
        ];
    }
}
```

JSONフィールド名は、API共通方針に従ってcamelCaseとする。

---

#### 2.37 Row Resource

行単位の結果も、専用Resourceへ変換する。

概念例：

```php
final class MonthEndAssetBalanceCsvPreviewRowResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'rowNumber'
                => $this->rowNumber,

            'assetAccountName'
                => $this->assetAccountName,

            'balance'
                => $this->balance,

            'errors'
                => CsvPreviewErrorResource::collection(
                    $this->errors,
                ),
        ];
    }
}
```

---

#### 2.38 Error Resource

プレビュー用エラーも、専用Resourceへ変換する。

概念例：

```php
final class CsvPreviewErrorResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'code'
                => $this->code,

            'message'
                => $this->message,
        ];
    }
}
```

CSV内部で使用した技術的な情報をエラーResourceへ含めない。

---

#### 2.39 返却しない情報

API Resourceでは、以下の内部情報を返却しない。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- `month_end_asset_balance_id`
- `balance_recording_unit`
- `confirmed`
- `created_at`
- `updated_at`

Queryで取得したEloquent ModelをそのままJSON化しない。

---

#### 2.40 Responder

Responderは、生成済みPreview DTOを受け取り、API共通の成功Envelope形式へ変換する。

概念例：

```php
final class MonthEndAssetBalanceCsvPreviewResponder
{
    public function ok(
        MonthEndAssetBalanceCsvPreview $preview,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new MonthEndAssetBalanceCsvPreviewResource(
                        $preview,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの共通項目は、API共通レスポンス処理に従う。

---

#### 2.41 Responderの責務

Responderでは、以下を行わない。

- CSV解析
- CSVヘッダー検証
- 行入力値検証
- 対象年月特定
- CSV内重複判定
- 資産口座検索
- snapshot検索
- 既存残高検索
- 業務ルール判定
- `canImport`判定

Responderは、生成済みPreview DTOをHTTPレスポンスへ変換することに責務を限定する。

---

#### 2.42 Repository

CSV-002では、Repositoryを使用しない。

本APIは業務データを更新しない参照・検証専用APIであるため、以下はQueryクラスが担当する。

- 資産口座取得
- 月末資産状況取得
- 既存月末資産残高取得

登録・更新・削除処理は存在しないため、CSV-002専用Repositoryは作成しない。

---

#### 2.43 トランザクション

CSV-002では、業務データを更新しないため、明示的な`DB::transaction()`を使用しない。

以下のような実装は行わない。

```php
DB::transaction(
    function () {
        // CSVプレビューのみ
    },
);
```

CSV-003では、複数行を原子的に登録するためトランザクションを使用するが、CSV-002では不要とする。

---

#### 2.44 ロック

CSV-002では、行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

CSV-002からCSV-003までの間、DB状態をロックして登録可否を保証する設計は採用しない。

CSV-003実行時に最新状態で再検証する。

---

#### 2.45 プレビュー結果の保存

Phase1では、Preview DTOやプレビュー済みCSVをデータベースへ保存しない。

以下のような一時管理は行わない。

```text
CSV-002
    ↓
previewId生成
    ↓
DBへプレビュー結果保存
    ↓
CSV-003でpreviewId指定
```

CSV-003では、CSVファイルを再送し、同じ業務ルールを再検証する。

---

#### 2.46 CSV-003との共通化

CSV-002とCSV-003では、CSV解析および登録可否判定ロジックを可能な限り共通化する。

例えば、以下を共通利用する。

```text
MonthEndAssetBalanceCsvDefinition
MonthEndAssetBalanceCsvParser
MonthEndAssetBalanceCsvValidator
MonthEndAssetBalanceCsvImportValidator
```

CSV-002とCSV-003で同じ業務ルールを個別実装しない。

---

#### 2.47 CSV-003との責務差

CSV-002とCSV-003の主な違いは、検証後に登録を行うかである。

```text
CSV-002

CSV解析
    ↓
検証
    ↓
Preview DTO
    ↓
終了
```

```text
CSV-003

CSV解析
    ↓
再検証
    ↓
トランザクション開始
    ↓
必要に応じてsnapshot作成
    ↓
月末資産残高一括登録
    ↓
commit
```

CSV-002で

```text
canImport = true
```

となっていても、CSV-003では必ず再検証する。

---

#### 2.48 キャッシュ

Phase1では、CSV-002専用のサーバー側アプリケーションキャッシュを使用しない。

同一CSVファイルであっても、以下が変更されればプレビュー結果も変化する。

- 資産口座の状態
- 月末資産状況の存在
- `confirmed`
- 既存月末資産残高

そのため、プレビュー実行ごとに最新の業務データを参照する。

---

#### 2.49 ログ

CSV-002では、必要に応じて以下の情報をログコンテキストへ設定する。

```text
requestId
userId
apiId
fileName
rowCount
```

`apiId`は、

```text
CSV-002
```

とする。

以下の内容は、不要にログへ出力しない。

- CSVファイル全文
- 資産口座名の全件
- 月末残高の全件

プレビューエラーについても、通常の入力エラーを大量にエラーログへ出力しない。

---

#### 2.50 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `file`未指定・ファイル検証不正 | `VALIDATION_ERROR` |
| CSV解析不能 | `INVALID_CSV_FORMAT` |
| CSVヘッダー不正 | `INVALID_CSV_FORMAT` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

CSV行単位の入力・業務エラーは、原則として例外へ変換しない。

Preview DTOの`errors`へ格納する。

---

#### 2.51 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- SQL
- テーブル名
- カラム名
- PostgreSQLの制約名
- PostgreSQL内部エラー
- スタックトレース
- PHP内部エラー
- Laravel内部例外メッセージ
- サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

#### 2.52 テスト実装方針

Laravel側では、Feature Testを中心としてCSV-002のAPI契約およびプレビューフロー全体を確認する。

主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `404 Not Found`
- `422 Unprocessable Entity`
- `500 Internal Server Error`
- `file`必須
- ファイル形式
- ファイルサイズ
- CSVヘッダー
- CSVヘッダー順序
- 余分なヘッダー
- データ行0件
- `target_year_month`必須
- `target_year_month`形式
- 1ファイル1対象年月
- `asset_account_name`必須
- 資産口座不存在
- 他利用者の同名資産口座を参照しないこと
- 残高記録単位
- 対象年月時点の資産口座有効性
- `balance`必須
- `balance`整数
- `balance`0以上
- `balance = 0`
- 月末資産状況不存在
- 確定済み月末資産状況
- 既存月末資産残高
- CSV内重複
- 複数エラー収集
- `rowNumber`
- `targetYearMonth`
- `canImport`
- 業務データ非更新
- 冪等性

---

#### 2.53 CSV ParserのUnit Test

CSV Parserについて、以下を確認する。

- 正しいヘッダーを取得できること
- データ行を正しく解析できること
- CSV上の行番号を保持できること
- BOMを仕様どおり処理できること
- 完全な空行を仕様どおり扱うこと
- `,,`をデータ行として扱えること
- CSV解析不能時に適切な例外となること
- ヘッダー不正時に後続の業務検証へ進まないこと

---

#### 2.54 CSV ValidatorのUnit Test

CSV Validatorについて、以下を個別に確認する。

```text
target_year_month
    required
    YYYY-MM

asset_account_name
    required

balance
    required
    integer
    min:0
```

特に、

```text
balance = 0
```

を正常値として必ずテストする。

---

#### 2.55 業務検証のUnit Test

CSV内容と業務データを照合する検証ロジックについて、以下を確認する。

```text
資産口座あり
    → 正常

資産口座なし
    → ASSET_ACCOUNT_NOT_FOUND

他利用者にのみ同名口座あり
    → ASSET_ACCOUNT_NOT_FOUND

商品単位口座
    → BALANCE_RECORDING_UNIT_MISMATCH

対象年月時点で口座無効
    → ASSET_ACCOUNT_NOT_AVAILABLE

snapshot不存在
    → CSV-003の仕様に従った判定

snapshot未確定
    → 登録可能

snapshot確定済み
    → MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED

既存balanceあり
    → MONTH_END_ASSET_BALANCE_ALREADY_EXISTS

CSV内同一口座重複
    → DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

---

#### 2.56 QueryのDatabase Test

Queryクラスについて、必要に応じてDatabase Testを行う。

AssetAccountQueryでは、以下を確認する。

```text
user_id一致
+
name一致
+
deleted_at IS NULL
    ↓
取得できる
```

```text
他利用者
+
name一致
    ↓
取得できない
```

MonthEndAssetSnapshotQueryでは、以下を確認する。

```text
user_id一致
+
target_year_month一致
    ↓
取得できる
```

```text
他利用者
+
target_year_month一致
    ↓
取得できない
```

MonthEndAssetBalanceQueryでは、以下を確認する。

```text
操作対象利用者
+
snapshotId
+
assetAccountIds
    ↓
対象となる既存残高のみ取得
```

他利用者の以下のデータがCSV-002の判定へ混入しないことを確認する。

- 同名資産口座
- 同一対象年月snapshot
- 月末資産残高

---

#### 2.57 UseCaseのUnit Test

UseCaseについては、Parser、Validator、Queryの結果を組み合わせて正しいPreview DTOを生成できることを確認する。

概念的には、

```text
ParsedCsv
+
CsvRowValidationResult
+
AssetAccountQuery結果
+
MonthEndAssetSnapshotQuery結果
+
MonthEndAssetBalanceQuery結果
    ↓
PreviewMonthEndAssetBalanceCsvUseCase
    ↓
MonthEndAssetBalanceCsvPreview
```

をテストする。

特に、以下を確認する。

```text
エラーなし
    ↓
canImport = true
```

```text
CSV全体エラーあり
    ↓
canImport = false
```

```text
行エラーあり
    ↓
canImport = false
```

```text
複数行にエラーあり
    ↓
複数エラーを保持
```

また、UseCase実行によって業務データが更新されないことを確認する。

---

#### 2.58 CSV-003との共通検証テスト

CSV-002とCSV-003で、同一システム状態かつ同一CSVに対する登録可否判定が一致することを確認する。

概念的には、

```text
CSV-002
検証結果
    ↓
canImport = true

同一DB状態
+
同一CSV
    ↓
CSV-003
再検証
    ↓
登録可能
```

となることを確認する。

ただし、CSV-002とCSV-003の間で業務データが変更された場合は、結果が変化してよい。

例えば、

```text
CSV-002
confirmed = false
    ↓
canImport = true

その後
confirmed = true

CSV-003
    ↓
登録不可
```

となることを正常な挙動として扱う。

---

## 3. 関連ドキュメント

- [CSV-002 API詳細設計](../../../api/details/csv-imports/csv-002-preview.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [CSVインポート Laravelアーキテクチャ設計](./README.md)
- [CSV-002 テスト設計](../../../tests/csv-imports/csv-002-preview.md)
- [CSVインポート テスト設計](../../../tests/csv-imports/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
