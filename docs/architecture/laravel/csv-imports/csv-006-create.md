##  CSV-006 商品別月末評価額CSV登録

### 1 概要

CSV-006では、商品別月末評価額CSVをアップロードし、CSV内容および最新の業務状態を再検証したうえで、商品別月末評価額を一括登録するためのLaravel実装方針を定義する。

本APIでは、CSV-005 商品別月末評価額CSVプレビューで登録可能と判定されたCSVであっても、その結果をそのまま信頼して登録せず、CSV-006実行時点の最新状態をもとに再度検証する。

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

CSV内のすべてのデータが登録可能であることを確認した場合のみ登録処理へ進む。

CSV内に登録できないデータが1件でも存在する場合は、正常な行だけを部分登録せず、CSV全体を登録不可とする。

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
    ├─ Import Validator
    ├─ AssetAccountQuery
    ├─ HoldingAssetQuery
    ├─ AssetAccountAvailableSettingQuery
    ├─ MonthEndAssetSnapshotQuery
    ├─ MonthEndHoldingValueQuery
    ├─ MonthEndAssetSnapshotRepository
    └─ MonthEndHoldingValueRepository
    ↓
Import Result DTO
    ↓
API Resource
    ↓
Responder
```

Requestでは、CSVファイルの必須チェック、ファイル形式、ファイルサイズなど、HTTPリクエストレベルで判定可能な内容のみを検証する。

CSV Parserでは、CSVファイルの読み込み、BOM処理、ヘッダー取得、行解析、行番号管理などを担当する。

CSV Validatorでは、`target_year_month`、`asset_account_name`、`holding_asset_name`、`value`など、データベースを参照せず判定可能なCSV入力値を検証する。

Import Validatorでは、Queryから取得した業務データを使用し、資産口座の存在、残高記録単位、保有商品の存在、資産口座と保有商品の関連、対象年月時点での有効性、月末資産状況の確定状態、既存商品別月末評価額など、登録可否に関する業務ルールを検証する。

CSV行ごとの個別SQL発行は避け、CSV全体から必要となる資産口座名、保有商品名などのキーを収集し、資産口座、保有商品、利用可能設定および既存商品別月末評価額を可能な限り一括取得する。取得結果はMap化し、各CSV行の検証をメモリ上で行う。

CSV-005とCSV-006では、CSV Definition、CSV Parser、CSV Validator、Import Validatorおよび業務データ取得用Queryを可能な限り共通利用する。

これにより、プレビュー時と登録時で同じ登録可否ルールを別々に実装することを避ける。

ただし、CSV-005とCSV-006の間で月末資産状況の確定、商品別月末評価額の登録、資産口座や保有商品の状態変更などが発生する可能性がある。

そのため、CSV-006ではCSV-005のプレビュー結果ではなく、登録実行時点のデータベースの最新状態を使用して登録可否を判断する。

すべての事前検証に成功した場合はトランザクションを開始し、対象年月の月末資産状況を最新状態で再取得する。

既存の月末資産状況が存在する場合は、必要に応じて`lockForUpdate()`を使用して行ロックを取得し、CSV登録中に同一月末資産状況の確定処理などが競合することを抑制する。

対象年月の月末資産状況が存在しない場合は、トランザクション内で`confirmed = false`の月末資産状況を新規作成する。

その後、月末資産状況の確定状態および既存商品別月末評価額を再確認し、登録可能であることを最終確認したうえで、CSV内の商品別月末評価額を一括登録する。

概念的な登録処理は、以下とする。

```text
CSV解析
    ↓
CSV構造検証
    ↓
行入力値検証
    ↓
業務データ一括取得
    ↓
業務ルール検証
    ↓
CSV全体が登録可能
    ↓
トランザクション開始
    ↓
月末資産状況を最新状態で再取得
    ↓
必要に応じてlockForUpdate
    ↓
月末資産状況が存在しなければ作成
    ↓
confirmed最終確認
    ↓
既存商品別月末評価額最終確認
    ↓
商品別月末評価額一括登録
    ↓
commit
    ↓
登録結果返却
```

月末資産状況および商品別月末評価額の永続化はRepositoryへ委譲する。

`MonthEndAssetSnapshotRepository`では、月末資産状況の作成および更新ロック付き取得を担当し、`MonthEndHoldingValueRepository`では、既存商品別月末評価額の最終確認および商品別月末評価額の一括登録を担当する。

Queryは参照・事前判定用、Repositoryは更新処理およびトランザクション内での最終確認用として責務を分離する。

同一利用者・同一対象年月の月末資産状況については、

```text
user_id
+
target_year_month
```

のUNIQUE制約を最終防衛線として使用する。

また、同一月末資産状況・同一保有商品については、

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

のUNIQUE制約を設定し、アプリケーション側の重複確認とデータベース制約の二段構えで重複登録を防止する。

同時実行によって商品別月末評価額のUNIQUE制約違反が発生した場合は、データベース内部例外をそのまま返却せず、可能な限り`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`へ変換し、`409 Conflict`として扱う。

登録途中でエラーが発生した場合はトランザクション全体をロールバックする。

CSV-006内で月末資産状況を新規作成した後に商品別月末評価額の登録で失敗した場合も、月末資産状況を含めてロールバックし、月末資産状況だけが残る状態や、一部の商品別月末評価額だけが登録される状態を許可しない。

商品別月末評価額は可能な限り一括INSERTし、CSVに含まれる資産口座名や保有商品名などの文字列をそのまま保存するのではなく、検証済みの`month_end_asset_snapshot_id`、`holding_asset_id`および`value`を使用して登録する。

CSV-006では月末資産状況を自動確定しない。新規作成した月末資産状況は`confirmed = false`とし、月末資産状況の確定はSNP-004の責務とする。

正常時は、登録対象年月および実際に新規登録した商品別月末評価額の件数をImport Result DTOとして生成する。

成功レスポンスでは、主に以下を返却する。

```text
targetYearMonth
importedCount
```

`importedCount`には、実際に新規登録した`month_end_holding_values`の件数のみを設定し、月末資産状況の作成件数は含めない。

Import Result DTOはAPI ResourceによってAPI共通方針に従ったcamelCase形式へ変換し、Responderを通して`201 Created`で返却する。

CSV-006では最新のデータベース状態を使用して登録可否を判断するため、Phase1では専用のサーバー側アプリケーションキャッシュを使用しない。

また、Phase1では`Idempotency-Key`を使用せず、事前重複確認、トランザクション、`lockForUpdate()`、UNIQUE制約およびフロントエンド側の二重送信防止によって重複登録を制御する。

---

### 2 Laravel実装方針

CSV-006では、
Action、
Request、
UseCase、
CSV Definition、
CSV Parser、
CSV Validator、
業務Validator、
Query、
Repository、
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
    ├─ Import Validator
    ├─ AssetAccountQuery
    ├─ HoldingAssetQuery
    ├─ AssetAccountAvailableSettingQuery
    ├─ MonthEndAssetSnapshotQuery
    ├─ MonthEndHoldingValueQuery
    ├─ MonthEndAssetSnapshotRepository
    └─ MonthEndHoldingValueRepository
    ↓
Import Result DTO
    ↓
API Resource
    ↓
Responder
```

CSV-006では、
CSV-005と同じ
CSV解析・登録可否判定ロジックを
可能な限り共通利用する。

一方、
CSV-006固有の

- トランザクション
- snapshotの必要時作成
- 排他制御
- 既存評価額の最終確認
- 商品別月末評価額の一括登録

は、
登録UseCase内で扱う。

---

#### 2.1 Action

Actionは、
HTTPリクエストを受け付け、
検証済みCSVファイルと
操作対象利用者を取得し、
登録UseCaseを呼び出す。

概念例：

```php
final class ImportMonthEndHoldingValueCsvAction
{
    public function __invoke(
        ImportMonthEndHoldingValueCsvRequest $request,
        ImportMonthEndHoldingValueCsvUseCase $useCase,
        MonthEndHoldingValueCsvImportResponder $responder,
        userContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                file:
                    $request->file('file'),
            );

        return $responder->created(
            $result,
        );
    }
}
```

Actionでは、
以下を行わない。

- CSV解析
- CSVヘッダー検証
- CSV入力値検証
- 対象年月特定
- CSV内重複判定
- 資産口座検索
- 保有商品検索
- 残高記録単位判定
- 対象年月時点の有効性判定
- snapshot検索
- 確定状態判定
- 既存商品別月末評価額検索
- トランザクション制御
- lock取得
- snapshot作成
- 商品別月末評価額登録
- レスポンス変換

Actionは、
UseCase呼び出しと
Responderへの受け渡しに
責務を限定する。

---

#### 2.2 Request

Requestでは、
HTTPリクエストとして
CSVファイルを
受け付けられる状態かを検証する。

概念例：

```php
final class ImportMonthEndHoldingValueCsvRequest
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

- MIME Type
- 拡張子
- 最大ファイルサイズ

は、
CSV共通仕様に従う。

---

#### 2.3 Requestで行うこと

Requestでは、
主に以下を検証する。

- `file`必須
- アップロードファイルであること
- 許可されたファイル形式であること
- ファイルサイズ上限以内であること

これらは、
CSV内容を解析する前に
判定できる
HTTP入力レベルの検証とする。

---

#### 2.4 Requestで行わないこと

Requestでは、
以下を検証しない。

- CSVヘッダー
- CSVデータ行
- `target_year_month`
- `asset_account_name`
- `holding_asset_name`
- `value`
- 1ファイル1対象年月
- CSV内重複
- 資産口座存在確認
- 残高記録単位
- 保有商品存在確認
- 保有商品と資産口座の関連
- 対象年月時点の有効性
- snapshot存在確認
- `confirmed`
- 既存商品別月末評価額

これらは、
CSV Validator、
業務Validator、
Query、
UseCaseで扱う。

---

#### 2.5 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、
`X-User-Id`を検証する。

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
userContext設定
    ↓
Request
    ↓
Action
```

Action以降では、
検証済み利用者コンテキストを
使用する。

---

#### 2.6 UseCase

CSV登録の
ユースケース全体を担当する。

主な処理は、
以下とする。

1. CSVファイルを解析する
2. CSVヘッダーを検証する
3. CSV各行の入力値を検証する
4. 対象年月を特定する
5. CSV内重複を検証する
6. 資産口座を一括取得する
7. 保有商品を一括取得する
8. 対象年月時点の有効性を検証する
9. snapshotを事前確認する
10. 既存商品別月末評価額を確認する
11. 全件登録可能であることを確認する
12. トランザクションを開始する
13. snapshotを最新状態で再取得する
14. 必要ならsnapshotを作成する
15. `confirmed`を再確認する
16. 既存商品別月末評価額を再確認する
17. 商品別月末評価額を一括登録する
18. Import Result DTOを返す

概念的には、
以下とする。

```text
CSV解析
    ↓
事前検証
    ↓
業務データ一括取得
    ↓
登録可否判定
    ↓
DB::transaction
    ↓
snapshot最終確認
    ↓
snapshot必要時作成
    ↓
既存評価額最終確認
    ↓
一括登録
    ↓
Import Result DTO
```

---

#### 2.7 CSV-005との共通化

CSV-005とCSV-006では、
以下を共通利用する。

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

同じ登録可否ルールを
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

CSV-006だけで
独自のヘッダー定義を持たない。

---

#### 2.9 CSV Parser

CSVファイル解析は、
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
- 行番号保持
- 完全な空行の除外
- CSV構造異常検出

業務データ検索や
登録処理は行わない。

---

#### 2.10 CSVヘッダー検証

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
データベース検索へ進まない。

---

#### 2.11 CSV Validator

CSV各行の
入力値検証は、
CSV Validatorへ分離する。

概念例：

```php
final class MonthEndHoldingValueCsvValidator
{
    public function validate(
        ParsedCsv $csv,
    ): CsvValidationResult {
        // CSV入力値検証
    }
}
```

主に以下を検証する。

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

データベース検索は行わない。

---

#### 2.12 valueの型変換

`value`は、
正常な整数形式の場合のみ
integerへ変換する。

以下を
暗黙変換によって
正常値にしてはならない。

```text
1000abc
1000.5
1,000
¥1000
```

また、

```text
0
```

は
正常値として扱う。

---

#### 2.13 対象年月特定

正常に解析できた
`target_year_month`を収集し、
単一対象年月であることを確認する。

概念例：

```php
$targetYearMonths =
    collect(
        $rows,
    )
        ->pluck(
            'targetYearMonth',
        )
        ->filter()
        ->unique()
        ->values();
```

2件以上存在する場合は、

```text
MULTIPLE_TARGET_YEAR_MONTHS
```

として扱う。

---

#### 2.14 CSV内重複判定

同一CSV内で、
同じ

```text
targetYearMonth
+
assetAccountName
+
holdingAssetName
```

が複数存在しないことを確認する。

重複時は、

```text
DUPLICATE_HOLDING_ASSET_IN_CSV
```

として扱う。

先勝ち・後勝ちにはしない。

---

#### 2.15 Import Validator

業務データを使用した
登録可否判定は、
専用Validatorへ分離する。

概念的には、

```text
MonthEndHoldingValueCsvImportValidator
```

が、
以下を判定する。

- 資産口座存在
- 残高記録単位
- 保有商品存在
- 保有商品と資産口座の関連
- 対象年月時点の有効性
- snapshot確定状態
- 既存商品別月末評価額

CSV-005でも
同じValidatorを使用する。

---

#### 2.16 AssetAccountQuery

CSVで使用される
資産口座名を収集し、
操作対象利用者に属する
資産口座を一括取得する。

概念例：

```php
$assetAccounts =
    $this->assetAccountQuery
        ->findActiveByNames(
            userId:
                $userId,

            names:
                $assetAccountNames,
        );
```

検索条件は、
概念的に以下とする。

```text
user_id = 操作対象利用者ID
AND
name IN (...)
AND
deleted_at IS NULL
```

---

#### 2.17 HoldingAssetQuery

対象資産口座と
保有商品名を使用し、
対象保有商品を
一括取得する。

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

保有商品名だけで
全利用者・全資産口座から
検索しない。

---

#### 2.18 Map化

取得結果は、
CSV行ごとの検証を
メモリ上で行えるよう、
Map化してよい。

資産口座：

```text
assetAccountName
    → AssetAccount
```

保有商品：

```text
assetAccountId
+
holdingAssetName
    → HoldingAsset
```

これにより、
CSV行ごとの
追加SQLを避ける。

---

#### 2.19 残高記録単位判定

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
    throw new BalanceRecordingUnitMismatchException();
}
```

実際のEnum・定数名は、
共通設計に従う。

---

#### 2.20 対象年月時点の有効性判定

対象保有商品が、
CSVの対象年月時点で
記録対象として有効かを判定する。

必要な
`asset_account_available_settings`を
一括取得し、
業務Validatorへ渡す。

現在日時ではなく、
必ず

```text
targetYearMonth
```

を基準にする。

---

#### 2.21 MonthEndAssetSnapshotQuery

事前検証では、
操作対象利用者・対象年月から
snapshotを取得する。

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

事前検証時には、
原則として
`lockForUpdate()`を使用しない。

---

#### 2.22 snapshot不存在

事前検証時に
snapshotが存在しなくても、
それだけでは
登録不可としない。

CSV-006では、
トランザクション内で
必要に応じて作成する。

---

#### 2.23 確定済みsnapshot

事前検証時点で、

```text
confirmed = true
```

の場合は、

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

として登録不可とする。

ただし、
最終的な状態確認は
トランザクション内でも行う。

---

#### 2.24 MonthEndHoldingValueQuery

snapshotが存在する場合は、
対象保有商品について
既存の商品別月末評価額を
一括取得する。

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
重複確認SQLを発行しない。

---

#### 2.25 事前検証後にトランザクションを開始する

CSV解析や
基本的な業務検証が
完了した後に、

```php
DB::transaction(
    function () {
        // DB更新処理
    },
);
```

を開始する。

CSVファイル解析開始時点から
トランザクションを
保持しない。

---

#### 2.26 トランザクション内の再取得

事前検証で使用した
snapshot状態を
そのまま最終判断には使用しない。

トランザクション内で
対象年月のsnapshotを
最新状態で再取得する。

概念例：

```php
$snapshot =
    $this->snapshotRepository
        ->findForUpdateByUserAndTargetYearMonth(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,
        );
```

---

#### 2.27 lockForUpdate

既存snapshotが存在する場合は、
必要に応じて
`lockForUpdate()`を使用する。

概念例：

```php
return MonthEndAssetSnapshot::query()
    ->where(
        'user_id',
        $userId,
    )
    ->where(
        'target_year_month',
        $targetYearMonth,
    )
    ->lockForUpdate()
    ->first();
```

これにより、
CSV登録中に
同じsnapshotの
確定処理などが
競合することを抑制する。

---

#### 2.28 snapshotの必要時作成

トランザクション内で
snapshotが存在しない場合は、
Repositoryを使用して
新規作成する。

概念例：

```php
$snapshot =
    $this->snapshotRepository
        ->create(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,

            confirmed:
                false,
        );
```

新規作成したsnapshotも
同一トランザクション内で扱う。

---

#### 2.29 snapshot重複作成への対応

同一利用者・同一対象年月について、

```text
user_id
+
target_year_month
```

にUNIQUE制約を設定する。

snapshot不存在時は、
対象行そのものがないため
`lockForUpdate()`だけでは
重複作成を完全に防げない。

そのため、
UNIQUE制約を
最終防衛線とする。

競合が発生した場合は、
API共通方針に沿って
再取得または
適切な競合処理を行う。

---

#### 2.30 confirmedの最終確認

既存snapshotを
トランザクション内で取得した後、
再度

```text
confirmed
```

を確認する。

概念例：

```php
if (
    $snapshot->confirmed
) {
    throw new
        MonthEndAssetSnapshotAlreadyConfirmedException();
}
```

CSV-005や
事前検証時の状態を
最終判断には使用しない。

---

#### 2.31 既存評価額の最終確認

トランザクション内でも、
対象保有商品について
既存の商品別月末評価額を
再確認する。

概念例：

```php
$existingValues =
    $this->monthEndHoldingValueRepository
        ->findExistingForUpdate(
            snapshotId:
                $snapshot->id,

            holdingAssetIds:
                $holdingAssetIds,
        );
```

ただし、
存在しないレコードそのものを
ロックすることはできないため、
最終的には
UNIQUE制約も利用する。

---

#### 2.32 UNIQUE制約

`month_end_holding_values`には、
概念的に以下の
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

アプリケーション側の
重複確認と
データベース制約の
二段構えとする。

---

#### 2.33 Repository

CSV-006では、
DB更新が存在するため
Repositoryを使用する。

主に以下を担当する。

```text
MonthEndAssetSnapshotRepository
    → snapshot作成
    → 更新ロック付き取得

MonthEndHoldingValueRepository
    → 既存評価額最終確認
    → 商品別月末評価額一括登録
```

Queryは
参照・判定用、
Repositoryは
更新処理用として
責務を分ける。

---

#### 2.34 MonthEndHoldingValueRepository

商品別月末評価額の
一括登録を担当する。

概念例：

```php
$this->monthEndHoldingValueRepository
    ->insertMany(
        $rows,
    );
```

登録データは、
概念的に以下とする。

```php
[
    [
        'month_end_asset_snapshot_id'
            => $snapshot->id,

        'holding_asset_id'
            => $holdingAssetId,

        'value'
            => $value,

        'created_at'
            => $now,

        'updated_at'
            => $now,
    ],
]
```

CSVの名称文字列を
そのまま保存しない。

---

#### 2.35 一括INSERT

可能な限り、
複数行を
一括INSERTする。

概念例：

```php
MonthEndHoldingValue::query()
    ->insert(
        $insertRows,
    );
```

ただし、
timestamp自動設定など
Eloquentイベントに依存する場合は、
その影響を理解したうえで
実装方式を選択する。

Phase1では、
大量データ向けの
複雑なBatch基盤は導入しない。

---

#### 2.36 現在時刻

`created_at`、
`updated_at`を
一括INSERTで設定する場合は、
同一処理内で取得した
同じ現在時刻を使用してよい。

概念例：

```php
$now =
    now();
```

CSV行ごとに
個別に`now()`を呼び出す必要はない。

---

#### 2.37 全件成功・全件失敗

CSV-006では、
部分登録を許可しない。

Repositoryで
一部だけ登録された後に
例外が発生した場合でも、
トランザクションによって
全件ロールバックする。

---

#### 2.38 UNIQUE制約違反の変換

同時実行などにより、

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

のUNIQUE制約違反が
発生する可能性がある。

この場合、
データベース例外を
そのまま500として返却しない。

可能な限り、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

へ変換し、

```text
409 Conflict
```

として扱う。

PostgreSQLの
制約名やSQLを
レスポンスへ公開しない。

---

#### 2.39 snapshot UNIQUE制約違反

snapshot新規作成時に、

```text
user_id
+
target_year_month
```

のUNIQUE制約違反が
発生した場合は、
同時実行で
別処理が先にsnapshotを
作成した可能性がある。

この場合は、
必要に応じて
既存snapshotを再取得し、
その状態を確認する。

ただし、
無制限なリトライ処理は
導入しない。

---

#### 2.40 Import Result DTO

正常登録結果は、
専用DTOで表現する。

概念例：

```php
final readonly class MonthEndHoldingValueCsvImportResult
{
    public function __construct(
        public string $targetYearMonth,
        public int $importedCount,
    ) {
    }
}
```

内部IDや
Eloquent Modelを
そのまま返却しない。

---

#### 2.41 importedCount

`importedCount`は、
実際に新規登録した

```text
month_end_holding_values
```

の件数とする。

例えば、

```php
$importedCount =
    count(
        $insertRows,
    );
```

とする。

snapshot作成件数は
含めない。

---

#### 2.42 API Resource

Import Result DTOを、
専用API Resourceで
レスポンス形式へ変換する。

概念例：

```php
final class MonthEndHoldingValueCsvImportResultResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'targetYearMonth'
                => $this->targetYearMonth,

            'importedCount'
                => $this->importedCount,
        ];
    }
}
```

JSONフィールド名は、
API共通方針に従って
camelCaseとする。

---

#### 2.43 Responder

Responderは、
Import Result DTOを
`201 Created`レスポンスへ変換する。

概念例：

```php
final class MonthEndHoldingValueCsvImportResponder
{
    public function created(
        MonthEndHoldingValueCsvImportResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new
                    MonthEndHoldingValueCsvImportResultResource(
                        $result,
                    ),
            ],
            Response::HTTP_CREATED,
        );
    }
}
```

`requestId`などの
共通項目は、
API共通レスポンス処理に従う。

---

#### 2.44 Responderで行わないこと

Responderでは、
以下を行わない。

- CSV解析
- CSV検証
- 登録可否判定
- DB検索
- transaction制御
- snapshot作成
- 商品別月末評価額登録
- importedCount再計算

Responderは、
生成済み結果を
HTTPレスポンスへ変換することに
責務を限定する。

---

#### 2.45 返却しない情報

API Resourceでは、
以下を返却しない。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`
- `asset_accounts.balance_recording_unit`
- `month_end_asset_snapshots.confirmed`
- `created_at`
- `updated_at`
- CSVファイル内容
- CSV行番号
- 登録した各`value`

成功レスポンスは、

```text
targetYearMonth
importedCount
```

に限定する。

---

#### 2.46 エラー変換

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `file`不正 | `VALIDATION_ERROR` |
| CSV解析不能 | `INVALID_CSV_FORMAT` |
| CSVヘッダー不正 | `INVALID_CSV_FORMAT` |
| データ行0件 | `CSV_DATA_REQUIRED` |
| 複数対象年月 | `MULTIPLE_TARGET_YEAR_MONTHS` |
| 資産口座不存在 | `ASSET_ACCOUNT_NOT_FOUND` |
| 残高記録単位不一致 | `BALANCE_RECORDING_UNIT_MISMATCH` |
| 保有商品不存在 | `HOLDING_ASSET_NOT_FOUND` |
| 対象年月時点で無効 | `HOLDING_ASSET_NOT_AVAILABLE` |
| `value`不正 | `INVALID_VALUE` |
| CSV内重複 | `DUPLICATE_HOLDING_ASSET_IN_CSV` |
| snapshot確定済み | `MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED` |
| 既存評価額あり | `MONTH_END_HOLDING_VALUE_ALREADY_EXISTS` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

実際のコード名は、
共通エラーコード定義に合わせる。

---

#### 2.47 複数入力エラー

トランザクション開始前の
CSV検証段階では、
可能な範囲で
複数エラーを収集してよい。

この場合は、
API共通エラー形式の

```text
error.details
```

へ格納する。

ただし、
CSV-006では
1件でもエラーが存在すれば
登録処理へ進まない。

---

#### 2.48 INTERNAL_SERVER_ERROR

想定外の例外は、
API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ
以下を含めない。

- SQL
- PostgreSQL内部エラー
- UNIQUE制約名
- テーブル名
- カラム名
- PHP内部エラー
- Laravel内部例外メッセージ
- スタックトレース
- サーバーファイルパス

詳細は、
サーバーログへ記録する。

---

#### 2.49 ログ

必要に応じて、
以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
targetYearMonth
rowCount
importedCount
errorCode
```

`apiId`は、

```text
CSV-006
```

とする。

CSVファイル全文や
全評価額を
不要にログへ出力しない。

---

#### 2.50 キャッシュ

Phase1では、
CSV-006専用の
サーバー側アプリケーションキャッシュを
使用しない。

登録可否は、
最新のDB状態を使用して判断する。

古い

- snapshot
- confirmed
- existing values
- asset account
- holding asset

を使用しない。

---

#### 2.51 Idempotency-Key

Phase1では、
`Idempotency-Key`を使用しない。

重複登録は、

- 事前重複確認
- トランザクション
- `lockForUpdate()`
- UNIQUE制約
- フロントエンドの二重送信防止

によって制御する。

---

#### 2.52 テスト実装方針

Laravel側では、
Feature Testを中心として
CSV-006のAPI契約と
一括登録フローを確認する。

主に以下を確認する。

- `201 Created`
- `400 Bad Request`
- `404 Not Found`
- `409 Conflict`
- `422 Unprocessable Entity`
- `500 Internal Server Error`
- `file`必須
- ファイル形式
- ファイルサイズ
- CSVヘッダー
- データ行0件
- 対象年月
- 1ファイル1対象年月
- 資産口座存在
- 利用者境界
- 残高記録単位
- 保有商品存在
- 資産口座と保有商品の関連
- 対象年月時点の有効性
- `value`
- 0円
- CSV内重複
- snapshot不存在時の作成
- snapshot未確定
- snapshot確定済み
- 既存商品別月末評価額
- 一部登録されないこと
- rollback
- snapshot重複作成防止
- 商品別月末評価額重複防止
- `targetYearMonth`
- `importedCount`

---

#### 2.53 CSV Parser・ValidatorのUnit Test

CSV Parserおよび
CSV Validatorは、
CSV-005と共通であるため、
同じUnit Testを使用する。

以下を重点的に確認する。

```text
ヘッダー
BOM
空行
target_year_month
asset_account_name
holding_asset_name
value
CSV内重複
```

CSV-005用と
CSV-006用で
同じテストケースを
二重管理しない。

---

#### 2.54 Import ValidatorのUnit Test

業務Validatorについて、
以下を確認する。

```text
資産口座なし
    → ASSET_ACCOUNT_NOT_FOUND

口座単位
    → BALANCE_RECORDING_UNIT_MISMATCH

保有商品なし
    → HOLDING_ASSET_NOT_FOUND

対象年月時点で無効
    → HOLDING_ASSET_NOT_AVAILABLE

snapshotなし
    → それだけではエラーにしない

snapshot未確定
    → 正常

snapshot確定済み
    → MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED

既存評価額あり
    → MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

CSV-005とCSV-006で
同じValidator結果になることを確認する。

---

#### 2.55 RepositoryのDatabase Test

`MonthEndAssetSnapshotRepository`では、
以下を確認する。

- 未確定snapshotを取得できる
- `user_id + target_year_month`で取得できる
- 他利用者のsnapshotを取得しない
- snapshotを`confirmed = false`で作成できる
- UNIQUE制約が維持される

`MonthEndHoldingValueRepository`では、
以下を確認する。

- 複数件を一括登録できる
- 正しいsnapshotへ紐づく
- 正しいholdingAssetへ紐づく
- `value = 0`を保存できる
- UNIQUE制約が維持される

---

#### 2.56 UseCaseのUnit Test

UseCaseでは、
Parser、
Validator、
Query、
Repositoryを組み合わせて
正しい登録結果になることを確認する。

概念的には、

```text
Parsed CSV
+
CSV Validation
+
Business Validation
+
Queries
    ↓
ImportMonthEndHoldingValueCsvUseCase
    ↓
Transaction
    ↓
Repositories
    ↓
Import Result DTO
```

を確認する。

特に、

```text
snapshotなし
    ↓
snapshot作成
    ↓
holding values登録
```

```text
snapshotあり未確定
    ↓
既存snapshot使用
```

```text
snapshot確定済み
    ↓
登録しない
```

```text
既存評価額あり
    ↓
登録しない
```

を確認する。

---

#### 2.57 トランザクションテスト

意図的に
商品別月末評価額登録途中で
例外を発生させ、

```text
ROLLBACK
```

されることを確認する。

特に、
CSV-006内で
snapshotを新規作成した場合は、
snapshotも
ロールバックされることを確認する。

---

#### 2.58 並行実行テスト

可能であれば、
Database Testまたは
Integration Testで
並行実行を確認する。

主な観点は、
以下とする。

```text
同一利用者
+
同一対象年月
+
snapshot不存在
```

で
複数登録が走っても、
snapshotが
重複作成されないこと。

また、

```text
同一snapshot
+
同一holding_asset
```

について
複数登録が走っても、
商品別月末評価額が
重複登録されないことを確認する。

---

#### 2.59 CSV-005との整合性テスト

同一利用者、
同一DB状態、
同一CSVで、

```text
CSV-005
canImport = true
```

となった場合に、
DB状態を変更せず
CSV-006を実行すると
登録成功することを確認する。

逆に、
CSV-005後に

```text
confirmed = true
```

へ変更した場合は、
CSV-006が

```text
409 Conflict
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

となることを確認する。

これにより、

```text
CSV-005
    → 事前確認

CSV-006
    → 最新状態で再検証
```

という責務分離を保証する。

---

### 3 関連ドキュメント

- [CSV-006 API詳細設計](../../../api/details/csv-imports/csv-006-create.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [CSVインポート Laravelアーキテクチャ設計](./README.md)
- [CSV-006 テスト設計](../../../tests/csv-imports/csv-006-create.md)
- [CSVインポート テスト設計](../../../tests/csv-imports/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)