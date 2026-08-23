##  CSV-005 商品別月末評価額CSVプレビュー

### 1 概要

CSV-005では、商品別月末評価額CSVをアップロードし、登録前にCSV内容および業務ルールを検証するためのLaravel実装方針を定義する。

本APIでは、操作対象利用者から受け取ったCSVファイルを解析し、各行について商品別月末評価額として登録可能な内容であるかを検証したうえで、登録予定内容およびエラー内容をプレビューとして返却する。

主に以下を検証対象とする。

- CSVファイル形式
- CSVヘッダー
- 対象年月
- 資産口座名
- 保有商品名
- 商品別月末評価額
- 1ファイル内の対象年月統一
- 資産口座の存在
- 資産口座の利用者境界
- 残高記録単位
- 保有商品の存在
- 保有商品と資産口座の関連
- 対象年月時点での保有商品の有効性
- 月末資産状況の状態
- 既存商品別月末評価額との重複
- CSV内の重複

CSVプレビューでは登録可否の検証のみを行い、商品別月末評価額をデータベースへ登録しない。また、月末資産状況、資産口座、保有商品、利用可能資産設定などの業務データも更新しない。

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
    ├─ HoldingAssetQuery
    ├─ AssetAccountAvailableSettingQuery
    ├─ MonthEndAssetSnapshotQuery
    └─ MonthEndHoldingValueQuery
    ↓
Preview DTO
    ↓
API Resource
    ↓
Responder
```

Requestでは、CSVファイルの必須チェック、ファイル形式、ファイルサイズなど、HTTPリクエストレベルで判定可能な内容のみを検証する。

CSV Parserでは、CSVファイルの読み込み、BOM処理、ヘッダー取得、行解析、行番号管理などを担当し、CSV Validatorでは、`target_year_month`、`asset_account_name`、`holding_asset_name`、`value`などの行単位の入力値を検証する。

UseCaseでは、CSV解析結果とQueryから取得した業務データを組み合わせ、対象年月、資産口座、残高記録単位、保有商品、資産口座と保有商品の関連、対象年月時点での有効性、月末資産状況、既存商品別月末評価額、CSV内重複などを検証する。

データベースアクセスはQueryへ委譲し、CSV行ごとの個別SQL発行を避ける。CSV全体から必要となる資産口座名、保有商品名などのキーを収集し、資産口座、保有商品、利用可能設定および既存商品別月末評価額を可能な限り一括取得したうえで、メモリ上で各行の登録可否を判定する。

保有商品は商品名だけでシステム全体から検索せず、CSVで指定された資産口座との関連を含めて特定する。これにより、異なる資産口座や他利用者に同名の保有商品が存在する場合でも、操作対象利用者の正しい保有商品だけを検証対象とする。

また、対象年月時点で商品別月末評価額の記録対象として有効な保有商品であるかを確認し、現在日時ではなくCSVで指定された`target_year_month`を基準として有効性を判定する。

CSV構造が正常に解析できる場合は、可能な範囲で複数の入力エラーおよび業務エラーを収集し、CSV全体エラーと行単位エラーをPreview DTOへ保持する。

`canImport`は、CSV全体エラーおよび各行のエラーが1件も存在しない場合のみ`true`とする。正常な行が存在していても、エラーが1件でも存在する場合は`false`とする。

CSV解析不能やCSVヘッダー不正など、プレビュー処理そのものを継続できない状態はHTTPエラーとして扱う。

一方、データ行0件、対象年月不正、資産口座不存在、残高記録単位不一致、保有商品不存在、対象年月時点での保有商品無効、商品別月末評価額不正、CSV内重複、確定済み月末資産状況、既存商品別月末評価額など、CSV内容に対する入力・業務上の問題はHTTPエラーとせず、`200 OK`のプレビュー結果内にエラー情報として返却する。

対象年月の月末資産状況が存在しない場合は、それだけを理由としてプレビューエラーとはしない。CSV-006 商品別月末評価額CSV登録で必要に応じて作成可能であるため、月末資産状況が存在しない状態のまま登録可否判定を継続する。CSV-005自身では月末資産状況を作成しない。

CSV-005では業務データを更新しないため、Repository、トランザクションおよび行ロックは使用しない。また、CSV-005からCSV-006までの間にデータベース状態が変更される可能性があるため、プレビュー時点の状態をロックして登録可否を保証しない。

Phase1ではプレビュー結果およびプレビュー済みCSVをデータベースへ保存しない。CSV-006ではCSVファイルを再送し、その実行時点の最新の業務データを使用して登録可否を再検証する。

CSV-004、CSV-005、CSV-006では、CSVヘッダーなどの商品別月末評価額CSV仕様を共通Definitionへ集約する。

また、CSV-005とCSV-006では、CSV Parser、CSV Validator、業務データ取得用Queryおよび登録可否に関する業務検証ロジックを可能な限り共通化し、プレビュー時と登録時で判定ルールが乖離しない構成とする。

ただし、CSV-005とCSV-006の間で月末資産状況の確定、商品別月末評価額の登録、資産口座や保有商品の状態変更などが発生した場合は、CSV-005の`canImport = true`とCSV-006実行時の登録可否が異なることを正常な挙動として扱う。

---

### 2 Laravel実装方針

CSV-005では、
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
    ├─ HoldingAssetQuery
    ├─ AssetAccountAvailableSettingQuery
    ├─ MonthEndAssetSnapshotQuery
    └─ MonthEndHoldingValueQuery
    ↓
Preview DTO
    ↓
API Resource
    ↓
Responder
```

CSV-005では、
CSV-006と同じ
CSV解析・登録可否判定ロジックを
可能な限り共通利用する。

Actionへ、

- CSV解析
- CSVヘッダー検証
- 行入力値検証
- 資産口座検索
- 保有商品検索
- 対象年月時点の有効性判定
- 月末資産状況検索
- 既存商品別月末評価額検索
- CSV内重複判定
- `canImport`判定

を直接記述しない。

---

#### 2.1 Action

HTTPリクエストを受け付け、
検証済みCSVファイルおよび
利用者コンテキストを取得する。

CSVプレビューUseCaseを呼び出し、
取得結果をResponderへ渡す。

概念例：

```php
final class PreviewMonthEndHoldingValueCsvAction
{
    public function __invoke(
        PreviewMonthEndHoldingValueCsvRequest $request,
        PreviewMonthEndHoldingValueCsvUseCase $useCase,
        MonthEndHoldingValueCsvPreviewResponder $responder,
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

Actionでは、
以下を行わない。

- CSVヘッダー検証
- CSV行解析
- `target_year_month`検証
- `asset_account_name`検証
- `holding_asset_name`検証
- `value`検証
- 1ファイル1対象年月判定
- CSV内重複判定
- 資産口座検索
- 保有商品検索
- 残高記録単位判定
- 対象年月時点の有効性判定
- 月末資産状況検索
- 確定状態判定
- 既存商品別月末評価額検索
- `canImport`判定
- レスポンス形式への変換

Actionは、
UseCaseの呼び出しと
Responderへの受け渡しに
責務を限定する。

---

#### 2.2 Request

Requestでは、
HTTPリクエストとして
CSVファイルを受け付けられる状態かを
検証する。

概念例：

```php
final class PreviewMonthEndHoldingValueCsvRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [
            'file' => [
                'required',
                'file',
                'mimes:csv',
                'max:' . config(
                    'csv.max_file_size_kb',
                ),
            ],
        ];
    }
}
```

実際の

- ファイルサイズ上限
- MIME Type
- 拡張子

の扱いは、
CSV共通仕様に従う。

Requestでは、
CSV内部の業務データまでは
検証しない。

---

#### 2.3 Requestで行うこと

Requestでは、
主に以下を検証する。

- `file`が指定されていること
- HTTPアップロードファイルであること
- 許可されたファイル形式であること
- ファイルサイズ上限以内であること

これらは、
CSV内容を解析する前に
判定可能な
HTTPリクエストレベルの
バリデーションとする。

---

#### 2.4 Requestで行わないこと

Requestでは、
以下のCSV内容・業務ルールを
検証しない。

- CSVヘッダー
- CSVデータ行の存在
- `target_year_month`
- `asset_account_name`
- `holding_asset_name`
- `value`
- 1ファイル1対象年月
- CSV内重複
- 資産口座存在確認
- 残高記録単位
- 保有商品存在確認
- 保有商品と資産口座の関連確認
- 対象年月時点の有効性
- 月末資産状況
- 確定状態
- 既存商品別月末評価額

これらは、
CSV Validator、
UseCase、
Queryで扱う。

---

#### 2.5 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONレスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、
`X-User-Id`を検証し、
操作対象利用者を特定する。

概念的には、
以下とする。

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

Action以降では、
検証済みの
利用者コンテキストを使用する。

---

#### 2.6 UseCase

CSVプレビューの
ユースケース処理を担当する。

主な処理は、
以下とする。

1. 操作対象利用者IDを受け取る
2. CSVファイルを受け取る
3. CSVを解析する
4. CSVヘッダーを検証する
5. 各行の入力値を検証する
6. 対象年月を特定する
7. CSV内重複を検証する
8. 資産口座を一括取得する
9. 保有商品を一括取得する
10. 対象年月時点の有効性を確認する
11. 月末資産状況を取得する
12. 既存商品別月末評価額を取得する
13. 業務エラーを集約する
14. `canImport`を判定する
15. Preview DTOを生成する

概念的には、
以下とする。

```text
CSV解析
    ↓
CSV構造検証
    ↓
行入力値検証
    ↓
対象年月特定
    ↓
CSV内重複判定
    ↓
資産口座一括取得
    ↓
保有商品一括取得
    ↓
対象年月時点の有効性確認
    ↓
snapshot取得
    ↓
既存評価額一括取得
    ↓
エラー集約
    ↓
canImport判定
    ↓
Preview DTO
```

UseCaseでは、
SQLやEloquent Query Builderを
直接組み立てない。

データベースアクセスは、
Queryへ委譲する。

---

#### 2.7 CSV-006との共通化

CSV-005とCSV-006では、
可能な限り
同じCSV解析・検証ロジックを使用する。

例えば、
以下を共通化する。

```text
MonthEndHoldingValueCsvDefinition
MonthEndHoldingValueCsvParser
MonthEndHoldingValueCsvValidator
MonthEndHoldingValueCsvImportValidator
AssetAccountQuery
HoldingAssetQuery
AssetAccountAvailableSettingQuery
MonthEndAssetSnapshotQuery
MonthEndHoldingValueQuery
```

概念的には、

```text
CSV-005
    ↓
共通解析・検証
    ↓
Preview DTO
```

```text
CSV-006
    ↓
共通解析・検証
    ↓
登録処理
```

とする。

CSV-005とCSV-006で
同じ業務ルールを
別々に実装しない。

---

#### 2.8 CSV Definition

CSV-004、
CSV-005、
CSV-006で使用する
商品別月末評価額CSV仕様は、
共通Definitionへ集約する。

概念例：

```php
final class MonthEndHoldingValueCsvDefinition
{
    public const HEADERS = [
        'target_year_month',
        'asset_account_name',
        'holding_asset_name',
        'value',
    ];
}
```

CSV-005だけで
独自のヘッダー定義を持たない。

---

#### 2.9 CSV Parser

CSVファイルの読み込みは、
専用Parserへ分離する。

概念例：

```php
final class MonthEndHoldingValueCsvParser
{
    public function parse(
        UploadedFile $file,
    ): ParsedCsv {
        // CSV解析
    }
}
```

Parserでは、
主に以下を行う。

- ファイルオープン
- BOM処理
- ヘッダー取得
- データ行読み込み
- CSV上の行番号管理
- 完全な空行の除外
- CSV構造異常の検出

Parserでは、
業務データ検索や
登録可否判定を行わない。

---

#### 2.10 Parsed Row DTO

CSV Parserから返す
各行は、
構造化されたDTOとして
表現してよい。

概念例：

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

CSV上の行番号を保持し、
プレビュー結果の

```text
rowNumber
```

として使用できるようにする。

---

#### 2.11 CSVヘッダー検証

CSVヘッダーは、
共通Definitionと
完全一致することを確認する。

概念例：

```php
if (
    $parsedCsv->headers
    !== MonthEndHoldingValueCsvDefinition::HEADERS
) {
    throw new InvalidCsvFormatException();
}
```

以下を不正とする。

- ヘッダー不足
- ヘッダー名不一致
- ヘッダー順序不一致
- 余分なヘッダー

ヘッダー不正時は、
後続の業務検証へ進まない。

---

#### 2.12 CSV Validator

CSV各行の入力値検証は、
専用Validatorへ分離する。

概念例：

```php
final class MonthEndHoldingValueCsvValidator
{
    public function validateRow(
        ParsedCsvRow $row,
    ): CsvRowValidationResult {
        // 入力値検証
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

holding_asset_name
    required

value
    required
    integer
    min:0
```

CSV Validatorでは、
データベース検索を行わない。

---

#### 2.13 valueの正規化

`value`は、
正常な整数形式の場合のみ
integerへ変換する。

PHPの暗黙的な型変換によって、
以下を正常値として扱ってはならない。

```text
1000abc
1000.5
1,000
¥1000
```

概念例：

```php
$value =
    filter_var(
        $rawValue,
        FILTER_VALIDATE_INT,
    );
```

実際には、
未入力と`0`を
正しく区別できる実装とする。

---

#### 2.14 0円の扱い

`value = 0`は、
正常値として扱う。

以下のような
truthy / falsy判定を使用しない。

```php
if (! $value) {
    // 0円まで未入力扱いになる
}
```

必須判定と
数値判定を明確に分離する。

---

#### 2.15 データ行0件

CSVヘッダーが正常でも、
データ行が存在しない場合は、
HTTP例外にはしない。

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

正常に取得できた
`target_year_month`を収集し、
CSV全体の対象年月を特定する。

概念例：

```php
$targetYearMonths =
    collect($validRows)
        ->pluck('targetYearMonth')
        ->unique()
        ->values();
```

2種類以上存在する場合は、
CSV全体エラーとして

```text
MULTIPLE_TARGET_YEAR_MONTHS
```

を生成する。

単一の対象年月として
特定できない場合は、
Preview DTOの

```text
targetYearMonth
```

を`null`としてよい。

---

#### 2.17 CSV内重複判定

同一CSV内で、
同じ資産口座・保有商品が
複数回指定されていないか確認する。

概念的なキーは、

```text
targetYearMonth
+
assetAccountName
+
holdingAssetName
```

とする。

1ファイル1対象年月が
成立している場合は、
実質的に

```text
assetAccountName
+
holdingAssetName
```

で判定してよい。

重複時は、

```text
DUPLICATE_HOLDING_ASSET_IN_CSV
```

相当の
行エラーを生成する。

---

#### 2.18 AssetAccountQuery

CSVで指定された
資産口座名について、
操作対象利用者の資産口座を
一括取得する。

概念例：

```php
$assetAccounts =
    $this->assetAccountQuery
        ->findActiveByNames(
            userId: $userId,
            names: $assetAccountNames,
        );
```

概念的な検索条件は、
以下とする。

```text
user_id = 操作対象利用者ID
AND
name IN (...)
AND
deleted_at IS NULL
```

CSV行ごとに
個別SQLを発行しない。

---

#### 2.19 資産口座Map

取得した資産口座は、
名前をキーとして
Map化してよい。

概念例：

```php
$assetAccountMap =
    $assetAccounts->keyBy(
        'name',
    );
```

各行では、

```php
$assetAccount =
    $assetAccountMap->get(
        $row->assetAccountName,
    );
```

として参照する。

---

#### 2.20 資産口座不存在

操作対象利用者に
該当資産口座が
存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

相当の
行エラーを生成する。

他利用者に
同名資産口座が存在していても、
取得対象へ含めない。

---

#### 2.21 残高記録単位判定

対象資産口座の

```text
balance_recording_unit
```

が
商品単位であることを確認する。

概念例：

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::HOLDING
) {
    // BALANCE_RECORDING_UNIT_MISMATCH
}
```

実際のEnum名・値は、
共通定義に従う。

口座単位の場合は、
行エラーを生成する。

---

#### 2.22 HoldingAssetQuery

CSV内で指定された
資産口座と保有商品名について、
対象保有商品を
まとめて取得する。

概念例：

```php
$holdingAssets =
    $this->holdingAssetQuery
        ->findActiveByAssetAccountsAndNames(
            assetAccountIds:
                $assetAccountIds,

            names:
                $holdingAssetNames,
        );
```

検索条件には、
資産口座との関連を
必ず含める。

保有商品名だけで
システム全体を検索しない。

---

#### 2.23 保有商品Map

取得した保有商品は、
概念的に

```text
assetAccountId
+
holdingAssetName
```

をキーとして
Map化してよい。

例えば、

```php
$key =
    $assetAccountId
    . ':'
    . $holdingAssetName;
```

として扱う。

これにより、
異なる資産口座に
同名商品が存在する場合でも
正しく特定できる。

---

#### 2.24 保有商品不存在

同じ行で指定された
資産口座内に
対象保有商品が
存在しない場合は、

```text
HOLDING_ASSET_NOT_FOUND
```

相当の
行エラーを生成する。

他資産口座や
他利用者に
同名商品が存在していても、
正常とは判定しない。

---

#### 2.25 対象年月時点の有効性判定

対象保有商品が、
CSVの`targetYearMonth`時点で
商品別月末評価額の
記録対象として有効かを判定する。

判定に
`asset_account_available_settings`
を使用する設計である場合は、
必要な設定をまとめて取得する。

概念的には、

```php
$this->availabilityValidator
    ->validate(
        holdingAsset:
            $holdingAsset,

        targetYearMonth:
            $targetYearMonth,

        availableSettings:
            $availableSettings,
    );
```

とする。

現在日時を基準に
判定しない。

---

#### 2.26 AssetAccountAvailableSettingQuery

対象年月時点の
利用可能状態判定に必要な設定を
一括取得する。

CSV行ごとに
個別Queryを発行しない。

必要な資産口座IDを収集したうえで、
対象年月に必要な設定だけを
まとめて取得する。

---

#### 2.27 MonthEndAssetSnapshotQuery

CSVの対象年月について、
操作対象利用者の
月末資産状況を取得する。

概念例：

```php
$snapshot =
    $this->snapshotQuery
        ->findByUserAndTargetYearMonth(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,
        );
```

検索条件には、
必ず利用者IDを含める。

---

#### 2.28 snapshot不存在

snapshotが存在しない場合は、
それだけを理由として
プレビューエラーを生成しない。

CSV-006で
必要に応じて作成可能であるため、

```text
snapshot = null
```

のまま
登録可否判定を継続する。

CSV-005では
snapshotを作成しない。

---

#### 2.29 確定状態判定

snapshotが存在する場合は、
`confirmed`を確認する。

概念例：

```php
if (
    $snapshot !== null
    && $snapshot->confirmed
) {
    // CSV全体エラー
}
```

確定済みの場合は、

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

相当の
CSV全体エラーを生成する。

---

#### 2.30 MonthEndHoldingValueQuery

snapshotが存在する場合は、
CSV対象保有商品について
既存の商品別月末評価額を
まとめて取得する。

概念例：

```php
$existingValues =
    $this->monthEndHoldingValueQuery
        ->findBySnapshotAndHoldingAssets(
            snapshotId:
                $snapshot->id,

            holdingAssetIds:
                $holdingAssetIds,
        );
```

CSV行ごとに
個別SQLを発行しない。

---

#### 2.31 既存評価額Map

既存の商品別月末評価額は、

```text
holding_asset_id
```

をキーとして
Map化してよい。

概念例：

```php
$existingValueMap =
    $existingValues->keyBy(
        'holding_asset_id',
    );
```

各CSV行で
既存データ有無を
メモリ上で判定する。

---

#### 2.32 既存商品別月末評価額

同一snapshot、
同一保有商品について
既存評価額が存在する場合は、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

相当の
行エラーを生成する。

既存値が
CSV値と同じ場合でも、
正常とは判定しない。

---

#### 2.33 N+1問題

CSV行ごとに
個別SQLを発行してはならない。

以下のような構成を避ける。

```text
1行目
    ↓
assetAccount検索
holdingAsset検索
availableSetting検索
existingValue検索

2行目
    ↓
同様の検索

3行目
    ↓
...
```

基本的には、

```text
CSV全体解析
    ↓
必要キー収集
    ↓
asset_accounts一括取得
    ↓
holding_assets一括取得
    ↓
available_settings一括取得
    ↓
snapshot 1件取得
    ↓
existingValues一括取得
    ↓
メモリ上で検証
```

とする。

---

#### 2.34 CsvPreviewError DTO

プレビューで使用するエラーは、
専用DTOとして表現する。

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

CSV全体エラーと
行エラーで
同じ構造を使用してよい。

---

#### 2.35 Preview Row DTO

各CSV行の
プレビュー結果は、
専用DTOとして表現する。

概念例：

```php
final readonly class MonthEndHoldingValueCsvPreviewRow
{
    public function __construct(
        public int $rowNumber,
        public ?string $assetAccountName,
        public ?string $holdingAssetName,
        public ?int $value,

        /** @var CsvPreviewError[] */
        public array $errors,
    ) {
    }
}
```

内部の

- `asset_account_id`
- `holding_asset_id`
- `month_end_asset_snapshot_id`

は
Preview Row DTOへ
公開しない。

---

#### 2.36 Preview DTO

プレビュー結果全体は、
専用DTOとして表現する。

概念例：

```php
final readonly class MonthEndHoldingValueCsvPreview
{
    public function __construct(
        public ?string $targetYearMonth,
        public bool $canImport,

        /** @var CsvPreviewError[] */
        public array $errors,

        /** @var MonthEndHoldingValueCsvPreviewRow[] */
        public array $rows,
    ) {
    }
}
```

Eloquent Modelや
Query結果を
そのままResponderへ渡さない。

---

#### 2.37 canImportの判定

`canImport`は、
CSV全体エラーおよび
各行エラーから
UseCaseで判定する。

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
            MonthEndHoldingValueCsvPreviewRow $row,
        ): bool =>
            count(
                $row->errors,
            ) > 0,
    );

$canImport =
    ! $hasGlobalErrors
    && ! $hasRowErrors;
```

正常行が存在しても、
エラーが1件でも存在すれば

```text
canImport = false
```

とする。

---

#### 2.38 複数エラー収集

CSV構造が正常な場合は、
可能な範囲で
複数エラーを収集する。

例えば、

```text
2行目
ASSET_ACCOUNT_NOT_FOUND

4行目
HOLDING_ASSET_NOT_FOUND

6行目
INVALID_VALUE
```

を
1回のレスポンスで
返却できるようにする。

1件目の行エラーで
例外を送出して
全体を終了する方式は
基本としない。

---

#### 2.39 HTTPエラーとプレビューエラーの分離

以下は、
HTTPエラーとして扱う。

- 利用者コンテキスト不正
- `file`未指定
- アップロードファイル不正
- ファイルサイズ超過
- CSV解析不能
- CSVヘッダー不正
- 想定外例外

一方、
以下は
プレビュー結果内の
業務エラーとして扱う。

- データ行0件
- `target_year_month`不正
- 対象年月混在
- `asset_account_name`不正
- 資産口座不存在
- 残高記録単位不一致
- `holding_asset_name`不正
- 保有商品不存在
- 対象年月時点の保有商品無効
- `value`不正
- CSV内重複
- 確定済み月末資産状況
- 既存商品別月末評価額

この2種類を
明確に分離する。

---

#### 2.40 API Resource

Preview DTOを、
専用API Resourceによって
APIレスポンス形式へ変換する。

概念例：

```php
final class MonthEndHoldingValueCsvPreviewResource
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
                => MonthEndHoldingValueCsvPreviewRowResource::collection(
                    $this->rows,
                ),
        ];
    }
}
```

JSONフィールド名は、
API共通方針に従って
camelCaseとする。

---

#### 2.41 Row Resource

行単位の結果も、
専用Resourceへ変換する。

概念例：

```php
final class MonthEndHoldingValueCsvPreviewRowResource
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

            'holdingAssetName'
                => $this->holdingAssetName,

            'value'
                => $this->value,

            'errors'
                => CsvPreviewErrorResource::collection(
                    $this->errors,
                ),
        ];
    }
}
```

---

#### 2.42 Error Resource

プレビュー用エラーも、
専用Resourceへ変換する。

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

CSV内部で使用した
技術的な情報を
公開しない。

---

#### 2.43 返却しない情報

API Resourceでは、
以下の内部情報を返却しない。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`
- `asset_accounts.balance_recording_unit`
- `month_end_asset_snapshots.confirmed`
- `created_at`
- `updated_at`
- `asset_account_available_settings`

Queryで取得した
Eloquent Modelを
そのままJSON化しない。

---

#### 2.44 Responder

Responderは、
生成済みPreview DTOを受け取り、
API共通の成功Envelope形式へ変換する。

概念例：

```php
final class MonthEndHoldingValueCsvPreviewResponder
{
    public function ok(
        MonthEndHoldingValueCsvPreview $preview,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new MonthEndHoldingValueCsvPreviewResource(
                        $preview,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの
共通項目は、
API共通レスポンス処理に従う。

---

#### 2.45 Responderの責務

Responderでは、
以下を行わない。

- CSV解析
- CSVヘッダー検証
- 行入力値検証
- 対象年月特定
- CSV内重複判定
- 資産口座検索
- 保有商品検索
- 有効性判定
- snapshot検索
- 既存評価額検索
- `canImport`判定

Responderは、
Preview DTOを
HTTPレスポンスへ変換することに
責務を限定する。

---

#### 2.46 Repository

CSV-005では、
Repositoryを使用しない。

本APIは、
業務データを更新しない
参照・検証専用APIであるため、

- 資産口座取得
- 保有商品取得
- 利用可能設定取得
- snapshot取得
- 既存商品別月末評価額取得

はQueryクラスが担当する。

登録・更新・削除処理は
存在しないため、
CSV-005専用Repositoryは
作成しない。

---

#### 2.47 トランザクション

CSV-005では、
業務データを更新しないため、
明示的な
`DB::transaction()`を使用しない。

以下のような実装は
行わない。

```php
DB::transaction(
    function () {
        // CSVプレビューのみ
    },
);
```

CSV-006で
登録時に最新状態を再検証するため、
CSV-005で
長時間の読み取りトランザクションを
保持しない。

---

#### 2.48 ロック

CSV-005では、
行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

CSV-005からCSV-006までの間、
DB状態をロックして
登録可否を保証しない。

---

#### 2.49 プレビュー結果の保存

Phase1では、
Preview DTOや
プレビュー済みCSVを
データベースへ保存しない。

以下の方式は採用しない。

```text
CSV-005
    ↓
previewId生成
    ↓
DB保存
    ↓
CSV-006でpreviewId指定
```

CSV-006では、
CSVファイルを再送し、
同じ業務ルールを
再検証する。

---

#### 2.50 キャッシュ

Phase1では、
CSV-005専用の
サーバー側アプリケーションキャッシュを
使用しない。

同じCSVでも、

- 資産口座の状態
- 保有商品の状態
- 利用可能設定
- snapshotの`confirmed`
- 既存商品別月末評価額

が変更されれば、
プレビュー結果も変化する。

そのため、
毎回最新状態を参照する。

---

#### 2.51 ログ

CSV-005では、
必要に応じて
以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
fileName
rowCount
canImport
errorCount
```

`apiId`は、

```text
CSV-005
```

とする。

以下は、
不要にログへ出力しない。

- CSVファイル全文
- 全`value`
- 全資産口座名
- 全保有商品名

---

#### 2.52 例外変換

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `file`未指定・ファイル検証不正 | `VALIDATION_ERROR` |
| CSV解析不能 | `INVALID_CSV_FORMAT` |
| CSVヘッダー不正 | `INVALID_CSV_FORMAT` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

CSV行単位や
CSV全体の業務エラーは、
原則として例外へ変換しない。

Preview DTOの

```text
errors
rows[].errors
```

へ格納する。

---

#### 2.53 想定外例外

想定外の例外は、
API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、
以下を含めない。

- SQL
- テーブル名
- カラム名
- PostgreSQL内部エラー
- 制約名
- PHP内部エラー
- Laravel内部例外メッセージ
- スタックトレース
- サーバーファイルパス

詳細情報は、
サーバーログへ記録する。

---

#### 2.54 テスト実装方針

Laravel側では、
Feature Testを中心として
CSV-005のAPI契約および
プレビューフロー全体を確認する。

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
- データ行0件
- `target_year_month`必須
- `target_year_month`形式
- 1ファイル1対象年月
- `asset_account_name`必須
- 資産口座不存在
- 他利用者の同名資産口座を参照しないこと
- 残高記録単位
- `holding_asset_name`必須
- 保有商品不存在
- 他資産口座の同名保有商品を参照しないこと
- 他利用者の同名保有商品を参照しないこと
- 対象年月時点の有効性
- `value`必須
- `value`整数
- `value`0以上
- `value = 0`
- CSV内重複
- snapshot不存在
- snapshot未確定
- snapshot確定済み
- 既存商品別月末評価額
- 複数エラー収集
- `rowNumber`
- `targetYearMonth`
- `canImport`
- 業務データ非更新
- 冪等性

---

#### 2.55 CSV ParserのUnit Test

CSV Parserについて、
以下を確認する。

- 正しいヘッダーを取得できること
- データ行を正しく解析できること
- CSV上の行番号を保持できること
- BOMを仕様どおり処理できること
- 完全な空行を仕様どおり扱うこと
- `,,,`をデータ行として扱えること
- CSV解析不能時に適切な例外となること
- ヘッダー不正時に後続の業務検証へ進まないこと

---

#### 2.56 CSV ValidatorのUnit Test

CSV Validatorについて、
以下を個別に確認する。

```text
target_year_month
    required
    YYYY-MM

asset_account_name
    required

holding_asset_name
    required

value
    required
    integer
    min:0
```

特に、

```text
value = 0
```

を正常値として
必ず確認する。

---

#### 2.57 業務検証のUnit Test

CSV内容と
業務データを照合する
検証ロジックについて、
以下を確認する。

```text
資産口座あり
    → 正常

資産口座なし
    → ASSET_ACCOUNT_NOT_FOUND

口座単位資産口座
    → BALANCE_RECORDING_UNIT_MISMATCH

保有商品あり
    → 正常

保有商品なし
    → HOLDING_ASSET_NOT_FOUND

別資産口座にのみ同名商品あり
    → HOLDING_ASSET_NOT_FOUND

対象年月時点で無効
    → HOLDING_ASSET_NOT_AVAILABLE

snapshotなし
    → それだけではエラーにしない

snapshot未確定
    → 登録可能

snapshot確定済み
    → MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED

既存評価額あり
    → MONTH_END_HOLDING_VALUE_ALREADY_EXISTS

CSV内重複
    → DUPLICATE_HOLDING_ASSET_IN_CSV
```

---

#### 2.58 QueryのDatabase Test

AssetAccountQueryでは、
以下を確認する。

```text
user_id一致
+
name一致
+
deleted_at IS NULL
    ↓
取得できる
```

他利用者の
同名資産口座は
取得できないことを確認する。

HoldingAssetQueryでは、

```text
asset_account_id一致
+
name一致
+
deleted_at IS NULL
    ↓
取得できる
```

ことを確認する。

他資産口座・他利用者の
同名保有商品が
混入しないことも確認する。

MonthEndAssetSnapshotQueryでは、

```text
user_id
+
target_year_month
```

で
正しいsnapshotだけを
取得できることを確認する。

MonthEndHoldingValueQueryでは、

```text
snapshotId
+
holdingAssetIds
```

で
既存評価額を
正しく取得できることを確認する。

---

#### 2.59 UseCaseのUnit Test

UseCaseについては、
Parser、
Validator、
Queryの結果を組み合わせて
正しいPreview DTOを
生成できることを確認する。

概念的には、

```text
ParsedCsv
+
CsvRowValidationResult
+
AssetAccountQuery結果
+
HoldingAssetQuery結果
+
AvailableSettingQuery結果
+
SnapshotQuery結果
+
ExistingValueQuery結果
    ↓
PreviewMonthEndHoldingValueCsvUseCase
    ↓
MonthEndHoldingValueCsvPreview
```

を確認する。

特に、

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
複数行エラーあり
    ↓
複数エラーを保持
```

となることを確認する。

また、
UseCase実行によって
業務データが
更新されないことも確認する。

---

#### 2.60 CSV-006との整合性テスト

CSV-005とCSV-006で、
同一利用者、
同一DB状態、
同一CSVに対する
登録可否判定が一致することを確認する。

概念的には、

```text
CSV-005
canImport = true

同一DB状態
+
同一CSV
    ↓
CSV-006
再検証
    ↓
登録可能
```

となることを確認する。

ただし、
CSV-005とCSV-006の間で
DB状態が変更された場合は、
結果が変化してよい。

例えば、

```text
CSV-005
confirmed = false
    ↓
canImport = true

その後
confirmed = true

CSV-006
    ↓
登録不可
```

となることを
正常な挙動として扱う。

---

### 3. 関連ドキュメント

- [CSV-005 API詳細設計](../../../api/details/csv-imports/csv-005-preview.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [CSVインポート Laravelアーキテクチャ設計](./README.md)
- [CSV-005 テスト設計](../../../tests/csv-imports/csv-005-preview.md)
- [CSVインポート テスト設計](../../../tests/csv-imports/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)