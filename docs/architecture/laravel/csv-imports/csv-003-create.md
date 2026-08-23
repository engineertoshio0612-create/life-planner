##  CSV-003 月末資産残高CSV登録

### 1 概要

CSV-003では、月末資産残高CSVをアップロードし、CSV内容および最新の業務状態を再検証したうえで、月末資産残高を一括登録するためのLaravel実装方針を定義する。

本APIでは、CSV-002 月末資産残高CSVプレビューで登録可能と判定されたCSVであっても、その結果をそのまま信頼して登録せず、CSV-003実行時点の最新状態をもとに再度検証する。

主に以下を検証対象とする。

- CSVファイル形式
- CSVヘッダー
- 対象年月
- 資産口座名
- 月末残高
- 1ファイル内の対象年月統一
- 資産口座の存在
- 資産口座の利用者境界
- 残高記録単位
- 対象年月時点での資産口座の有効性
- 月末資産状況の状態
- 既存月末資産残高との重複
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
    ├─ AssetAccountQuery
    ├─ MonthEndAssetSnapshotQuery
    ├─ MonthEndAssetBalanceQuery
    ├─ MonthEndAssetSnapshotRepository
    └─ MonthEndAssetBalanceRepository
    ↓
Import Result DTO
    ↓
API Resource
    ↓
Responder
```

Requestでは、CSVファイルの必須チェック、ファイル形式、ファイルサイズなど、HTTPリクエストレベルで判定可能な内容のみを検証する。

CSV Parserでは、CSVファイルの読み込み、BOM処理、ヘッダー取得、行解析、行番号管理などを担当し、CSV Validatorでは、`target_year_month`、`asset_account_name`、`balance`などの行単位の入力値を検証する。

UseCaseでは、CSV解析結果とQueryから取得した業務データを組み合わせ、対象年月、資産口座、残高記録単位、対象年月時点での有効性、月末資産状況、既存月末資産残高、CSV内重複などを検証する。

CSV行ごとの個別SQL発行は避け、対象となる資産口座や既存月末資産残高を一括取得したうえで、可能な範囲でメモリ上で検証する。

CSV-002とCSV-003では、CSV Definition、CSV Parser、CSV Validatorおよび登録可否に関する業務検証ロジックを可能な限り共通化し、プレビュー時と登録時で判定ルールが乖離しない構成とする。

ただし、CSV-002とCSV-003の間で月末資産状況の確定や月末資産残高の登録など、業務データの状態が変更される可能性がある。そのため、CSV-003ではCSV-002のプレビュー結果ではなく、登録実行時点のデータベースの最新状態を使用して登録可否を判断する。

すべての事前検証に成功した場合はトランザクションを開始し、対象年月の月末資産状況を再取得する。既存の月末資産状況が存在する場合は必要に応じて行ロックを取得し、確定状態および既存月末資産残高を再確認する。

対象年月の月末資産状況が存在しない場合は、トランザクション内で`confirmed = false`の月末資産状況を新規作成する。

その後、CSV内の月末資産残高を一括登録する。登録途中でエラーが発生した場合はトランザクション全体をロールバックし、月末資産状況のみが作成された状態や、一部の月末資産残高のみが登録された状態を残さない。

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
月末資産状況再取得・必要に応じてロック
    ↓
確定状態再確認
    ↓
月末資産状況が存在しなければ作成
    ↓
既存月末資産残高再確認
    ↓
月末資産残高一括登録
    ↓
commit
    ↓
登録結果返却
```

月末資産状況および月末資産残高の永続化はRepositoryへ委譲し、RepositoryではCSV解析、入力値検証、業務ルール判定、トランザクション制御などを行わない。

また、同一利用者・同一対象年月の月末資産状況、および同一月末資産状況・同一資産口座の月末資産残高については、アプリケーション側の事前確認だけでなくデータベースのUNIQUE制約によっても重複登録を防止する。

CSV-003では月末資産状況を自動確定しない。新規作成した月末資産状況は`confirmed = false`とし、月末資産状況の確定はSNP-004の責務とする。

正常時は、登録対象年月および登録件数をImport Result DTOとして生成し、API ResourceおよびResponderを通してAPI共通の成功Envelopeへ変換し、`201 Created`で返却する。

---

### 2 Laravel実装方針

CSV-003では、
Action、
Request、
UseCase、
CSV Definition、
CSV Parser、
CSV Validator、
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
    ├─ AssetAccountQuery
    ├─ MonthEndAssetSnapshotQuery
    ├─ MonthEndAssetBalanceQuery
    ├─ MonthEndAssetSnapshotRepository
    └─ MonthEndAssetBalanceRepository
    ↓
Import Result DTO
    ↓
API Resource
    ↓
Responder
```

CSV-003では、
CSV-002と同じ
CSV解析・検証ロジックを
可能な限り共通利用する。

CSV-003固有の責務は、

```text
検証済みデータ
    ↓
登録直前の業務状態再確認
    ↓
トランザクション
    ↓
必要に応じてsnapshot作成
    ↓
月末資産残高一括登録
```

とする。

Actionへ、

- CSV解析
- CSVヘッダー検証
- 行入力値検証
- 資産口座検索
- 月末資産状況検索
- 重複判定
- トランザクション制御
- 一括登録

を直接記述しない。

---

#### 2.1 Action

HTTPリクエストを受け付け、
検証済みCSVファイルおよび
利用者コンテキストを取得する。

CSV登録UseCaseを呼び出し、
登録結果をResponderへ渡す。

概念例：

```php
final class ImportMonthEndAssetBalanceCsvAction
{
    public function __invoke(
        ImportMonthEndAssetBalanceCsvRequest $request,
        ImportMonthEndAssetBalanceCsvUseCase $useCase,
        MonthEndAssetBalanceCsvImportResponder $responder,
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

- CSVヘッダー検証
- CSV行解析
- `target_year_month`検証
- `asset_account_name`検証
- `balance`検証
- CSV内重複判定
- 資産口座検索
- 残高記録単位判定
- 対象年月時点の有効性判定
- 月末資産状況検索
- 確定状態判定
- 既存月末資産残高検索
- snapshot作成
- 月末資産残高登録
- トランザクション制御
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
final class ImportMonthEndAssetBalanceCsvRequest
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
CSV解析前に判定可能な
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
- `balance`
- 1ファイル1対象年月
- CSV内重複
- 資産口座存在確認
- 残高記録単位
- 対象年月時点の有効性
- 月末資産状況
- 確定状態
- 既存月末資産残高

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
検証済みの利用者コンテキストを使用する。

---

#### 2.6 UseCase

CSV登録の
ユースケース処理を担当する。

主な処理は、
以下とする。

1. 操作対象利用者IDを受け取る
2. CSVファイルを受け取る
3. CSVファイルを解析する
4. CSVヘッダーを検証する
5. 各行の入力値を検証する
6. 対象年月を特定する
7. CSV内重複を検証する
8. 対象資産口座を一括取得する
9. 資産口座に関する業務ルールを検証する
10. 月末資産状況を確認する
11. 既存月末資産残高を確認する
12. CSV全体が登録可能であることを確認する
13. トランザクションを開始する
14. 月末資産状況の状態を再確認する
15. 必要に応じて月末資産状況を作成する
16. 既存月末資産残高を再確認する
17. 月末資産残高を一括登録する
18. 登録結果DTOを生成する
19. 登録結果を返却する

概念的な処理は、
以下とする。

```text
CSV解析
    ↓
CSV構造検証
    ↓
行入力値検証
    ↓
業務データ取得
    ↓
業務ルール検証
    ↓
登録可能
    ↓
DB::transaction
    ↓
snapshot再取得
    ↓
確定状態再確認
    ↓
snapshot必要時作成
    ↓
既存balance再確認
    ↓
balance一括登録
    ↓
Import Result DTO
```

---

#### 2.7 CSV-002との共通化

CSV-002とCSV-003では、
可能な限り
同一のCSV解析・検証クラスを使用する。

例えば、
以下を共通化する。

```text
MonthEndAssetBalanceCsvDefinition
MonthEndAssetBalanceCsvParser
MonthEndAssetBalanceCsvValidator
MonthEndAssetBalanceCsvImportValidator
AssetAccountQuery
MonthEndAssetSnapshotQuery
MonthEndAssetBalanceQuery
```

概念的には、

```text
CSV-002
    ↓
共通解析・検証
    ↓
Preview DTO

CSV-003
    ↓
共通解析・検証
    ↓
登録処理
```

とする。

CSV-002とCSV-003で
同じ業務ルールを
別々に実装しない。

---

#### 2.8 CSV Definition

CSV-001、
CSV-002、
CSV-003で使用する
月末資産残高CSV仕様は、
共通Definitionへ集約する。

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

CSV-003専用の
ヘッダー定義を持たない。

---

#### 2.9 CSV Parser

CSVファイルの読み込みは、
CSV-002と共通の
Parserを使用する。

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

Parserでは、
主に以下を行う。

- CSVファイルオープン
- BOM処理
- ヘッダー取得
- CSV行読み込み
- 行番号管理
- 完全な空行の除外
- CSV構造異常の検出

Parserでは、
データベース検索や
登録処理を行わない。

---

#### 2.10 CSVヘッダー検証

CSVヘッダーは、
共通Definitionと
完全一致することを確認する。

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

ヘッダー不正時は、
業務検証や登録処理へ進まない。

---

#### 2.11 CSV Validator

CSV各行の入力値検証は、
CSV-002と共通の
Validatorを使用する。

主に以下を検証する。

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

CSV Validatorでは、
データベース検索を行わない。

---

#### 2.12 0円の扱い

`balance = 0`は、
正常値として扱う。

以下のような
truthy / falsy判定を使用しない。

```php
if (! $balance) {
    // 0円を未入力扱いしてしまう
}
```

必須チェックと
数値チェックを分離する。

---

#### 2.13 データ行0件

CSVヘッダーが正常でも、
データ行が1件も存在しない場合は、
登録不可とする。

CSV-002では
`canImport = false`
として扱うが、
CSV-003では

```text
CSV_DATA_REQUIRED
```

相当の業務エラーとして
登録処理を終了する。

snapshotおよび
月末資産残高は作成しない。

---

#### 2.14 対象年月の特定

正常な
`target_year_month`を収集し、
CSV全体の対象年月を特定する。

複数年月が存在する場合は、

```text
MULTIPLE_TARGET_YEAR_MONTHS
```

相当のエラーとする。

概念例：

```php
$targetYearMonths =
    collect($validRows)
        ->pluck('targetYearMonth')
        ->unique()
        ->values();

if (
    $targetYearMonths->count() !== 1
) {
    throw new
        MultipleTargetYearMonthsException();
}
```

---

#### 2.15 CSV内重複

CSV内で
同一資産口座が
複数回指定されていないことを確認する。

1ファイル1対象年月であるため、
実質的には

```text
assetAccountName
```

単位で一意であることを
確認してよい。

重複が存在する場合は、

```text
DUPLICATE_ASSET_ACCOUNT_IN_CSV
```

相当の登録エラーとする。

---

#### 2.16 AssetAccountQuery

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

検索条件は、
概念的に以下とする。

```text
user_id
    = 操作対象利用者ID

AND

name IN (...)

AND

deleted_at IS NULL
```

CSV行ごとに
個別SQLを発行しない。

---

#### 2.17 資産口座Map

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

CSV各行について、

```php
$assetAccount =
    $assetAccountMap->get(
        $row->assetAccountName,
    );
```

として参照する。

---

#### 2.18 資産口座不存在

操作対象利用者に
対応する資産口座が
存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

相当のエラーとする。

他利用者に
同名資産口座が存在していても、
取得対象としてはならない。

---

#### 2.19 残高記録単位判定

対象資産口座の
`balance_recording_unit`が
口座単位であることを確認する。

概念例：

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::ACCOUNT
) {
    throw new
        AssetAccountBalanceRecordingUnitMismatchException();
}
```

商品単位の場合は、
CSV-003の登録対象としない。

---

#### 2.20 対象年月時点の有効性判定

対象資産口座が、
CSVの`targetYearMonth`時点で
月末資産管理対象として
有効であることを確認する。

現在日時を基準に
判定しない。

概念的には、

```php
$this->availabilityValidator
    ->validate(
        assetAccount:
            $assetAccount,

        targetYearMonth:
            $targetYearMonth,
    );
```

とする。

対象年月時点で無効の場合は、
登録処理へ進まない。

---

#### 2.21 MonthEndAssetSnapshotQuery

対象年月の
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
必ず操作対象利用者IDを含める。

他利用者の
同一対象年月snapshotを
取得しない。

---

#### 2.22 確定状態の事前確認

既存snapshotが存在する場合は、
CSV全体の検証段階で
`confirmed`を確認する。

```php
if (
    $snapshot !== null
    && $snapshot->confirmed
) {
    throw new
        MonthEndAssetSnapshotConfirmedException();
}
```

確定済みの場合は、
登録処理へ進まない。

ただし、
この事前確認だけで
登録時の整合性を保証しない。

トランザクション内でも
最新状態を再確認する。

---

#### 2.23 MonthEndAssetBalanceQuery

既存snapshotが存在する場合は、
CSV対象資産口座について
既存月末資産残高を
一括取得する。

概念例：

```php
$existingBalances =
    $this->balanceQuery
        ->findBySnapshotAndAssetAccounts(
            userId:
                $userId,

            snapshotId:
                $snapshot->id,

            assetAccountIds:
                $assetAccountIds,
        );
```

既存残高が存在する場合は、

```text
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

相当のエラーとする。

---

#### 2.24 全件検証後に登録する

CSV行を読み込みながら
逐次INSERTしない。

以下のような実装は
行わない。

```php
foreach ($rows as $row) {
    $this->repository
        ->create(
            $row,
        );
}
```

ただし、
このループより前に
全件検証が完了していない場合を指す。

基本的には、

```text
CSV解析
    ↓
全件入力検証
    ↓
全行业務検証
    ↓
登録可能確認
    ↓
トランザクション
    ↓
一括登録
```

とする。

---

#### 2.25 Import Input DTO

CSV解析および
業務検証後の登録予定データは、
専用DTOとして保持してよい。

概念例：

```php
final readonly class MonthEndAssetBalanceImportItem
{
    public function __construct(
        public int $assetAccountId,
        public int $balance,
    ) {
    }
}
```

CSV行番号や
資産口座名など、
登録時に不要な情報を
Repositoryへ渡す必要はない。

---

#### 2.26 Import Result DTO

CSV登録結果は、
専用DTOとして表現する。

概念例：

```php
final readonly class MonthEndAssetBalanceCsvImportResult
{
    public function __construct(
        public string $targetYearMonth,
        public int $importedCount,
    ) {
    }
}
```

Eloquent Modelや
登録済みCollectionを
そのままResponderへ渡さない。

---

#### 2.27 トランザクション

CSV-003の登録処理は、
`DB::transaction()`内で実行する。

概念例：

```php
$result =
    DB::transaction(
        function () use (
            $userId,
            $targetYearMonth,
            $items,
        ): MonthEndAssetBalanceCsvImportResult {
            // snapshot再取得
            // 確定状態再確認
            // snapshot必要時作成
            // 既存balance再確認
            // balance一括登録

            return new
                MonthEndAssetBalanceCsvImportResult(
                    targetYearMonth:
                        $targetYearMonth,

                    importedCount:
                        count($items),
                );
        },
    );
```

CSV解析や
ファイルバリデーションまで
トランザクションへ含めない。

---

#### 2.28 トランザクション内での再取得

登録トランザクション開始後、
対象年月のsnapshotを
再度取得する。

概念例：

```php
$snapshot =
    $this->snapshotQuery
        ->findByUserAndTargetYearMonthForUpdate(
            userId:
                $userId,

            targetYearMonth:
                $targetYearMonth,
        );
```

CSV検証段階で取得した
snapshotの状態だけを
信頼して登録しない。

---

#### 2.29 snapshotのロック

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
同じsnapshotに対する

- CSV登録
- 月末資産状況確定
- その他の更新処理

との競合を抑制する。

---

#### 2.30 snapshot不存在時

トランザクション内で
snapshotが存在しない場合は、
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
        );
```

新規作成時の
`confirmed`は、

```text
false
```

とする。

---

#### 2.31 MonthEndAssetSnapshotRepository

月末資産状況の
新規作成を担当する。

概念例：

```php
final class MonthEndAssetSnapshotRepository
{
    public function create(
        int $userId,
        string $targetYearMonth,
    ): MonthEndAssetSnapshot {
        return MonthEndAssetSnapshot::create([
            'user_id'
                => $userId,

            'target_year_month'
                => $targetYearMonth,

            'confirmed'
                => false,
        ]);
    }
}
```

Repositoryでは、
以下を行わない。

- CSV解析
- CSV入力値検証
- 資産口座検索
- CSV内重複判定
- 月末資産状況確定
- 月末資産残高登録

---

#### 2.32 snapshotの重複作成防止

同一利用者、
同一対象年月について
snapshotを複数作成してはならない。

データベースでは、
概念的に以下の
UNIQUE制約を使用する。

```text
UNIQUE (
    user_id,
    target_year_month
)
```

複数リクエストが
同時にsnapshot不存在を確認した場合でも、
DB制約によって
重複作成を防止する。

---

#### 2.33 確定状態の最終確認

トランザクション内で
取得したsnapshotについて、
再度`confirmed`を確認する。

```php
if ($snapshot->confirmed) {
    throw new
        MonthEndAssetSnapshotConfirmedException();
}
```

CSV解析時点や
CSV-002実行時点の状態を
登録可否の最終判断として
使用しない。

---

#### 2.34 既存月末資産残高の最終確認

登録トランザクション内でも、
既存月末資産残高を
再確認する。

概念例：

```php
$existing =
    $this->balanceQuery
        ->existsAnyBySnapshotAndAssetAccounts(
            snapshotId:
                $snapshot->id,

            assetAccountIds:
                $assetAccountIds,
        );

if ($existing) {
    throw new
        MonthEndAssetBalanceAlreadyExistsException();
}
```

事前検証後に
別リクエストが
登録している可能性を考慮する。

---

#### 2.35 MonthEndAssetBalanceRepository

月末資産残高の
登録を担当する。

CSV-003では、
複数件をまとめて登録するため、
一括INSERTを使用してよい。

概念例：

```php
final class MonthEndAssetBalanceRepository
{
    /**
     * @param MonthEndAssetBalanceImportItem[] $items
     */
    public function insertAll(
        int $snapshotId,
        array $items,
    ): void {
        $now =
            now();

        $rows =
            array_map(
                static fn (
                    MonthEndAssetBalanceImportItem $item,
                ): array => [
                    'month_end_asset_snapshot_id'
                        => $snapshotId,

                    'asset_account_id'
                        => $item->assetAccountId,

                    'balance'
                        => $item->balance,

                    'created_at'
                        => $now,

                    'updated_at'
                        => $now,
                ],
                $items,
            );

        MonthEndAssetBalance::query()
            ->insert(
                $rows,
            );
    }
}
```

---

#### 2.36 一括INSERT

CSV-003では、
CSV行数分のINSERTを
1件ずつ実行するより、
可能であれば
一括INSERTを使用する。

```text
100行CSV
    ↓
100回INSERT
```

ではなく、

```text
100行CSV
    ↓
1回または少数回のINSERT
```

とする。

ただし、
CSV最大行数や
PostgreSQLのパラメータ上限などを考慮し、
必要であれば
一定件数でchunkしてよい。

---

#### 2.37 Repositoryの責務

MonthEndAssetBalanceRepositoryでは、
以下を行わない。

- CSV解析
- CSVヘッダー検証
- 資産口座存在確認
- 利用者境界判定
- 残高記録単位判定
- 確定状態判定
- 重複判定
- トランザクション開始
- HTTPレスポンス生成

Repositoryは、
UseCaseから渡された
登録可能なデータを
永続化することに
責務を限定する。

---

#### 2.38 UNIQUE制約

`month_end_asset_balances`では、
以下の組み合わせに
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
asset_account_id
```

アプリケーション側の
重複確認を通過した
複数リクエストが同時実行されても、
DB側で重複登録を防止する。

---

#### 2.39 UNIQUE制約違反

月末資産残高登録時に
UNIQUE制約違反が発生した場合は、
PostgreSQLの例外を
そのままレスポンスへ返却しない。

概念的には、

```text
UniqueViolation
    ↓
MonthEndAssetBalanceAlreadyExistsException
    ↓
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

へ変換する。

ただし、
制約名やSQLSTATEだけを
雑に判定するのではなく、
対象制約を識別したうえで
業務エラーへ変換する。

---

#### 2.40 snapshot UNIQUE制約違反

snapshot新規作成時に、

```text
user_id
+
target_year_month
```

のUNIQUE制約違反が
発生する可能性がある。

これは、
並行リクエストによって
別処理が先にsnapshotを
作成した可能性がある。

その場合は、
トランザクション設計に従って
最新snapshotを再取得し、
確定状態・既存残高を
再確認する。

単純に
`500 Internal Server Error`
へ変換しない。

---

#### 2.41 部分登録を行わない

一括登録途中で
例外が発生した場合は、
トランザクションを
ロールバックする。

```text
snapshot作成
    ↓
balance登録
    ↓
例外
    ↓
ROLLBACK
```

となる。

以下の状態を
残してはならない。

- snapshotだけ存在する
- 一部のbalanceだけ存在する
- CSV前半だけ登録済み

---

#### 2.42 confirmedを更新しない

CSV-003では、
snapshotを自動確定しない。

新規作成時は、

```text
confirmed = false
```

とする。

既存snapshotについても、

```php
$snapshot->update([
    'confirmed' => true,
]);
```

などの処理は行わない。

月末資産状況確定は、
SNP-004の責務とする。

---

#### 2.43 全資産口座の登録完了を要求しない

CSV-003では、
対象年月時点で有効な
すべての資産口座が
CSVに含まれていることを
登録条件とはしない。

CSVに含まれる
登録可能な資産口座のみを
登録する。

未登録資産が存在するかどうかは、
月末資産状況詳細表示や
確定処理で確認する。

---

#### 2.44 API Resource

登録結果DTOを、
専用API Resourceへ変換する。

概念例：

```php
final class MonthEndAssetBalanceCsvImportResource
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

データベース内部IDを
レスポンスへ返却しない。

---

#### 2.45 Responder

Responderは、
登録結果DTOを受け取り、
API共通の
成功Envelopeへ変換する。

概念例：

```php
final class MonthEndAssetBalanceCsvImportResponder
{
    public function created(
        MonthEndAssetBalanceCsvImportResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new MonthEndAssetBalanceCsvImportResource(
                        $result,
                    ),
            ],
            Response::HTTP_CREATED,
        );
    }
}
```

正常時は、

```text
201 Created
```

を返却する。

`requestId`などの
共通項目は、
API共通レスポンス処理に従う。

---

#### 2.46 Responderの責務

Responderでは、
以下を行わない。

- CSV解析
- CSV入力値検証
- 資産口座検索
- snapshot検索
- 確定状態判定
- 重複判定
- 月末資産残高登録
- `importedCount`算出
- トランザクション制御

Responderは、
生成済み登録結果DTOを
HTTPレスポンスへ
変換することに
責務を限定する。

---

#### 2.47 返却しない情報

API Resourceでは、
以下の内部情報を返却しない。

- `user_id`
- `asset_account_id`
- `month_end_asset_snapshot_id`
- `month_end_asset_balance_id`
- `confirmed`
- `balance_recording_unit`
- `created_at`
- `updated_at`
- CSV行番号
- 登録した各`balance`
- 登録した各資産口座名

Eloquent Modelを
そのままJSON化しない。

---

#### 2.48 例外

業務上の登録不可状態は、
専用例外として表現する。

例えば、
以下を想定する。

```text
InvalidCsvFormatException
CsvDataRequiredException
MultipleTargetYearMonthsException
AssetAccountNotFoundException
AssetAccountBalanceRecordingUnitMismatchException
AssetAccountNotAvailableException
InvalidBalanceException
DuplicateAssetAccountInCsvException
MonthEndAssetSnapshotConfirmedException
MonthEndAssetBalanceAlreadyExistsException
```

これらを
API共通Exception Handlerで
独自エラーコードへ変換する。

---

#### 2.49 CSV複数エラー

CSV入力内容に
複数エラーが存在する場合は、
必要に応じて
集約用例外を使用してよい。

概念例：

```php
throw new CsvImportValidationException(
    $errors,
);
```

この例外を、

```text
422 Unprocessable Entity
```

のCSV登録エラーへ変換する。

CSV-002と同様に
複数のエラーを収集できる構成としつつ、
CSV-003では
1件でもエラーがあれば
登録処理へ進まない。

---

#### 2.50 想定外例外

想定外の例外は、
API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、
以下を含めない。

- SQL
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- スタックトレース
- PHP内部エラー
- Laravel内部例外メッセージ
- サーバーファイルパス

詳細情報は、
サーバーログへ記録する。

---

#### 2.51 N+1問題

CSV行ごとに
資産口座や既存残高を
個別検索しない。

以下のような実装を避ける。

```text
1行目
    ↓
assetAccount検索
balance検索

2行目
    ↓
assetAccount検索
balance検索

3行目
    ↓
...
```

基本的には、

```text
CSV全体解析
    ↓
資産口座名一覧抽出
    ↓
asset_accounts一括取得
    ↓
snapshot 1件取得
    ↓
既存balance一括取得
    ↓
メモリ上で全件検証
```

とする。

---

#### 2.52 ロック範囲

ロックは、
CSV解析中には取得しない。

以下の処理が完了してから、
登録トランザクションを開始する。

```text
ファイル解析
CSV構造検証
入力値検証
基本業務検証
```

その後、

```text
DB::transaction
    ↓
snapshotロック
    ↓
最終状態確認
    ↓
登録
```

とする。

ロック保持時間を
必要最小限にする。

---

#### 2.53 キャッシュ

CSV-003では、
登録可否判定に
サーバー側キャッシュを使用しない。

以下は、
データベースの最新状態を参照する。

- snapshot
- `confirmed`
- 既存月末資産残高
- 資産口座状態

古いキャッシュを利用して
登録可否を判断しない。

---

#### 2.54 ログ

CSV-003では、
必要に応じて
以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
targetYearMonth
rowCount
importedCount
```

`apiId`は、

```text
CSV-003
```

とする。

以下は、
不要にログへ出力しない。

- CSVファイル全文
- 月末残高全件
- 資産口座名全件

登録失敗時には、
業務エラーコードを
ログへ記録してよい。

---

#### 2.55 テスト実装方針

Laravel側では、
Feature Testを中心として
CSV-003のAPI契約および
一括登録フロー全体を確認する。

主に以下を確認する。

- `201 Created`
- `400 Bad Request`
- `404 Not Found`
- `409 Conflict`
- `422 Unprocessable Entity`
- `500 Internal Server Error`
- `file`必須
- CSVファイル形式
- CSVヘッダー
- CSVヘッダー順序
- データ行0件
- 1ファイル1対象年月
- `asset_account_name`必須
- 資産口座不存在
- 他利用者の同名資産口座を使用しないこと
- 残高記録単位
- 対象年月時点の資産口座有効性
- `balance`必須
- `balance`整数
- `balance`0以上
- `balance = 0`
- CSV内重複
- snapshot不存在時の新規作成
- snapshot既存時の再利用
- snapshot確定済み
- 既存月末資産残高
- 一部登録されないこと
- トランザクションロールバック
- snapshotだけ残らないこと
- 同時実行時の重複防止
- `confirmed`を変更しないこと
- `importedCount`
- API ResourceによるcamelCase変換
- 返却対象外項目
- 同一CSV再実行時の競合

---

#### 2.56 CSV ParserのUnit Test

CSV Parserについて、
以下を確認する。

- 正しいヘッダーを取得できること
- 各データ行を解析できること
- 行番号を保持できること
- BOMを仕様どおり扱えること
- 完全な空行を仕様どおり扱えること
- CSV解析不能時に適切な例外となること

CSV-002と
同一Parserのテストを
共通利用してよい。

---

#### 2.57 CSV ValidatorのUnit Test

CSV Validatorについて、
以下を確認する。

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

を正常値として
確認する。

CSV-002と
同一Validatorを使用する場合は、
同一Unit Testで保証してよい。

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

MonthEndAssetSnapshotQueryでは、

```text
user_id
+
target_year_month
```

で
対象snapshotを取得できることを確認する。

MonthEndAssetBalanceQueryでは、

```text
snapshotId
+
assetAccountIds
```

によって
既存残高を取得できることを確認する。

---

#### 2.59 RepositoryのDatabase Test

MonthEndAssetSnapshotRepositoryでは、

```text
user_id
target_year_month
confirmed = false
```

で
snapshotを作成できることを確認する。

MonthEndAssetBalanceRepositoryでは、
複数のImport Itemから
正しい月末資産残高を
一括登録できることを確認する。

特に、

- `month_end_asset_snapshot_id`
- `asset_account_id`
- `balance`

が
正しく保存されることを確認する。

---

#### 2.60 UseCaseのUnit Test

UseCaseについては、
Parser、
Validator、
Query、
Repositoryを組み合わせた
ユースケース制御を確認する。

概念的には、

```text
CSV
    ↓
Parser
    ↓
Validator
    ↓
Query
    ↓
ImportMonthEndAssetBalanceCsvUseCase
    ↓
Repository
    ↓
Import Result
```

を確認する。

主に以下をテストする。

```text
正常CSV
    ↓
登録成功
```

```text
CSV入力エラー
    ↓
Repositoryを呼ばない
```

```text
snapshot確定済み
    ↓
登録しない
```

```text
既存balanceあり
    ↓
登録しない
```

```text
snapshotなし
    ↓
snapshot作成
    ↓
balance登録
```

```text
snapshotあり
    ↓
既存snapshot使用
    ↓
balance登録
```

---

#### 2.61 トランザクションのFeature Test

登録途中に
意図的な例外を発生させ、
トランザクションが
ロールバックされることを確認する。

例えば、

```text
snapshot作成
    ↓
balance登録
    ↓
例外
```

の場合に、

```text
snapshot
    → 残らない

balance
    → 残らない
```

ことを確認する。

既存snapshotを使用する場合は、

```text
既存snapshot
    → 残る

新規balance
    → 全件ロールバック
```

となることを確認する。

---

#### 2.62 UNIQUE制約のDatabase Test

以下のUNIQUE制約が
機能することを確認する。

```text
month_end_asset_snapshots

user_id
+
target_year_month
```

および、

```text
month_end_asset_balances

month_end_asset_snapshot_id
+
asset_account_id
```

同一キーで
重複登録できないことを確認する。

---

#### 2.63 CSV-002との整合性テスト

同一DB状態、
同一CSVについて、

```text
CSV-002
canImport = true
```

となる場合に、
CSV-003の事前検証も
登録可能となることを確認する。

CSV-002とCSV-003の
検証ロジックの差異によって、

```text
CSV-002
正常

CSV-003
同一状態なのに入力エラー
```

となることを防止する。

ただし、
CSV-002後に
DB状態が変更された場合は、
CSV-003で
登録不可となってよい。

---

#### 2.64 同時実行テスト

必要に応じて、
同一利用者、
同一対象年月、
同一資産口座に対する
並行登録をテストする。

最終的に、

```text
同一snapshot
+
同一assetAccount
    ↓
月末資産残高は1件のみ
```

となることを確認する。

競合したリクエストでは、
PostgreSQLの生例外ではなく、

```text
409 Conflict
MONTH_END_ASSET_BALANCE_ALREADY_EXISTS
```

へ変換されることを確認する。

---

### 3 関連ドキュメント

- [CSV-003 API詳細設計](../../../api/details/csv-imports/csv-003-create.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [CSVインポート Laravelアーキテクチャ設計](./README.md)
- [CSV-003 テスト設計](../../../tests/csv-imports/csv-003-create.md)
- [CSVインポート テスト設計](../../../tests/csv-imports/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)