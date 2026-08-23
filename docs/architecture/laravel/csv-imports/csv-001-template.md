##  CSV-001 月末資産残高CSVテンプレート取得

### 1. 概要

CSV-001では、月末資産残高CSVインポートで使用するCSVテンプレートを取得するためのLaravel実装方針を定義する。

本APIでは、口座単位で月末資産残高を登録するためのCSVテンプレートを生成し、CSVファイルとして返却する。テンプレートには、月末資産残高CSVで必要となる以下のヘッダーを含める。

```text
target_year_month
asset_account_name
balance
```

CSVテンプレートはヘッダー行のみで構成し、資産口座や月末資産残高などの利用者固有データは出力しない。

また、本APIはテンプレートの取得のみを責務とし、以下の業務データに対する登録・更新・削除は行わない。

* 月末資産状況
* 月末資産残高
* 商品別月末評価額
* 資産口座
* 保有商品
* 利用可能資産設定

Laravel側では、以下の責務を分離して実装する。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ↓
CSV Definition
    ↓
CSV Generator
    ↓
Responder
    ↓
CSVダウンロードレスポンス
```

CSVヘッダー、ファイル名、文字コード、BOM、改行コードなどのCSV仕様は、CSV-002 月末資産残高CSVプレビューおよびCSV-003 月末資産残高CSV登録と整合するよう共通化する。

本APIでは利用者固有の業務データを参照しないため、CSVテンプレート生成専用のRepositoryまたはQueryクラスは使用しない。また、パスパラメータ、クエリパラメータ、リクエストボディを使用しないため、CSV-001専用のFormRequestも作成しない。`X-User-Id`の検証および利用者コンテキストの設定は、API共通Middlewareで行う。

正常時はJSON成功Envelopeを使用せず、生成したCSVテンプレートを`text/csv`のダウンロードレスポンスとして直接返却する。一方、利用者コンテキストの不備やCSV生成処理の失敗などのエラー時は、API共通方針に従ってJSON形式のエラーレスポンスを返却する。

CSV-001では業務データを更新しないため、トランザクションおよび行ロックは使用しない。また、Phase1ではCSVテンプレートの生成コストが小さいことから、専用のサーバー側アプリケーションキャッシュも使用しない。


---

### 2 Laravel実装方針

CSV-001では、
Action、
UseCase、
CSV Definition、
CSV Generator、
Responderを分離して実装する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ↓
CSV Definition
    ↓
CSV Generator
    ↓
Responder
    ↓
CSVダウンロードレスポンス
```

本APIでは、利用者固有の業務データをCSVテンプレートへ出力しない。

そのため、CSVテンプレート生成のためのRepositoryまたはQueryクラスは使用しない。

また、以下を使用しないため、CSV-001専用のFormRequestも作成しない。

- パスパラメータ
- クエリパラメータ
- リクエストボディ

---

#### 2.1 Action

HTTPリクエストを受け付け、CSVテンプレート取得UseCaseを呼び出し、生成結果をResponderへ渡す。

概念例：

```php
final class DownloadMonthEndAssetBalanceCsvTemplateAction
{
    public function __invoke(
        DownloadMonthEndAssetBalanceCsvTemplateUseCase $useCase,
        MonthEndAssetBalanceCsvTemplateResponder $responder,
    ): StreamedResponse {
        $template =
            $useCase->execute();

        return $responder->download(
            $template,
        );
    }
}
```

Actionでは、以下を行わない。

- CSVヘッダーの定義
- CSV文字列の生成
- CSVエスケープ処理
- 文字コード変換
- BOM付与
- ファイル名の決定
- `Content-Type`の決定
- `Content-Disposition`の生成
- 利用者存在確認
- データベース検索

Actionは、UseCaseの呼び出しとResponderへの受け渡しに責務を限定する。

---

#### 2.2 FormRequest

本APIでは、専用のFormRequestを作成しない。

本APIは、

```text
パスパラメータなし
クエリパラメータなし
リクエストボディなし
```

であるため、CSV-001固有の入力バリデーションが存在しない。

`X-User-Id`の検証は、API共通Middlewareで行う。

そのため、以下のような空のFormRequestは作成しない。

```php
final class DownloadMonthEndAssetBalanceCsvTemplateRequest
    extends FormRequest
{
}
```

不要なクラスを追加しない。

---

#### 2.3 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
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
Action
```

利用者の存在確認条件は、以下とする。

```text
users.id = X-User-Id
AND
users.deleted_at IS NULL
```

Action以降では、利用者存在確認を再実装しない。

---

#### 2.4 UseCase

CSVテンプレート取得のユースケース処理を担当する。

概念例：

```php
final class DownloadMonthEndAssetBalanceCsvTemplateUseCase
{
    public function __construct(
        private readonly
        MonthEndAssetBalanceCsvGenerator $csvGenerator,
    ) {
    }

    public function execute():
        MonthEndAssetBalanceCsvTemplate
    {
        return new MonthEndAssetBalanceCsvTemplate(
            content:
                $this->csvGenerator
                    ->generateTemplate(),

            fileName:
                MonthEndAssetBalanceCsvDefinition::FILE_NAME,
        );
    }
}
```

CSVテンプレートの内容は、利用者によって変化しない。

そのため、UseCaseへ`userId`を渡す必要はない。

利用者コンテキストの確認自体は、Action到達前にMiddlewareで完了していることを前提とする。

---

#### 2.5 UseCaseで行わないこと

UseCaseでは、以下を行わない。

- `users`の再検索
- `asset_accounts`の検索
- `holding_assets`の検索
- `month_end_asset_snapshots`の検索
- `month_end_asset_balances`の検索
- `month_end_holding_values`の検索
- `asset_account_available_settings`の検索
- 利用者固有データのCSV出力
- 月末資産残高の登録
- 月末資産残高の更新
- 月末資産残高の削除

CSVテンプレートの生成に必要な処理だけを行う。

---

#### 2.6 CSV Template DTO

CSV生成結果を単純な文字列だけでResponderへ渡すのではなく、必要に応じて専用DTOとして表現する。

概念例：

```php
final readonly class MonthEndAssetBalanceCsvTemplate
{
    public function __construct(
        public string $content,
        public string $fileName,
    ) {
    }
}
```

これにより、UseCaseからResponderへ渡すCSVテンプレート情報を明示的に表現できる。

将来的に、以下の情報をテンプレート単位で保持する必要が生じた場合も拡張しやすくなる。

- 文字コード
- BOM有無
- MIME Type

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

    public const FILE_NAME =
        'month-end-asset-balances-template.csv';
}
```

CSV-001だけで独自のヘッダーを定義しない。

CSV-002およびCSV-003でも、同じDefinitionを使用する。

---

#### 2.8 CSVヘッダー

CSVテンプレートでは、以下のヘッダーをこの順序で出力する。

```text
target_year_month
asset_account_name
balance
```

生成結果は、以下とする。

```csv
target_year_month,asset_account_name,balance
```

ヘッダー名だけでなく、ヘッダー順序もCSV仕様の一部として扱う。

---

#### 2.9 CSV Generator

CSV文字列の生成は、専用Generatorで行う。

概念例：

```php
final class MonthEndAssetBalanceCsvGenerator
{
    public function generateTemplate(): string
    {
        $stream =
            fopen(
                'php://temp',
                'r+',
            );

        if ($stream === false) {
            throw new CsvGenerationException();
        }

        try {
            fputcsv(
                $stream,
                MonthEndAssetBalanceCsvDefinition::HEADERS,
            );

            rewind($stream);

            $csv =
                stream_get_contents(
                    $stream,
                );

            if ($csv === false) {
                throw new CsvGenerationException();
            }

            return $csv;
        } finally {
            fclose(
                $stream,
            );
        }
    }
}
```

Generatorは、CSV形式の生成に責務を限定する。

以下を行わない。

- HTTPレスポンス生成
- 利用者検索
- 資産口座検索
- 業務データ取得
- 月末資産残高登録

---

#### 2.10 fputcsvの利用

CSV生成では、PHP標準の`fputcsv()`を使用する。

以下のような手動連結は基本的に行わない。

```php
implode(
    ',',
    MonthEndAssetBalanceCsvDefinition::HEADERS,
);
```

`fputcsv()`を使用することで、将来的にCSV項目へ以下を含む場合でも、CSVとして適切にエスケープできる。

```text
,
"
改行
```

---

#### 2.11 fopen失敗時

`php://temp`のオープンに失敗した場合は、そのまま処理を続行しない。

概念例：

```php
$stream =
    fopen(
        'php://temp',
        'r+',
    );

if ($stream === false) {
    throw new CsvGenerationException();
}
```

CSV生成失敗は、最終的にAPI共通Exception Handlerによって

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

---

#### 2.12 fputcsv失敗時

`fputcsv()`の戻り値も必要に応じて確認する。

概念例：

```php
$result =
    fputcsv(
        $stream,
        MonthEndAssetBalanceCsvDefinition::HEADERS,
    );

if ($result === false) {
    throw new CsvGenerationException();
}
```

CSV生成処理の失敗を正常な空CSVとして扱ってはならない。

---

#### 2.13 リソース解放

CSV Generatorでストリームを開いた場合は、処理結果にかかわらず確実に`fclose()`する。

概念的には、

```php
try {
    // CSV生成
} finally {
    fclose(
        $stream,
    );
}
```

とする。

---

#### 2.14 文字コード

CSVテンプレートは、UTF-8で生成する。

CSV-002およびCSV-003で受け付ける文字コードと統一する。

CSV-001だけ異なる文字コードへ変換しない。

---

#### 2.15 BOM

UTF-8 BOMを付与するかどうかは、CSV共通仕様として統一する。

BOMを付与する場合は、CSV Generator内で明示的に行う。

概念例：

```php
fwrite(
    $stream,
    "\xEF\xBB\xBF",
);
```

CSV-001ではBOMあり、CSV-002・CSV-003ではBOM非対応、という不整合を発生させてはならない。

---

#### 2.16 改行コード

CSVテンプレートの改行コードは、CSV関連APIで統一する。

CSV-001、CSV-002、CSV-003で異なる改行コード仕様を個別定義しない。

実行環境による差異を許容しない仕様とする場合は、CSV Generator側で明示的に制御する。

---

#### 2.17 データ行

CSV-001では、利用者固有のデータ行を出力しない。

生成対象は、ヘッダー行のみとする。

以下のような資産口座一覧取得処理を行わない。

```php
$assetAccounts =
    AssetAccount::query()
        ->where(
            'user_id',
            $userId,
        )
        ->get();
```

また、以下のようなデータ行生成も行わない。

```php
foreach (
    $assetAccounts
    as $assetAccount
) {
    fputcsv(
        $stream,
        [
            '',
            $assetAccount->name,
            '',
        ],
    );
}
```

CSVテンプレートは、業務データから独立した固定構造とする。

---

#### 2.18 Repository・Query

CSV-001では、RepositoryおよびQueryクラスを使用しない。

本APIで必要となるデータベース参照は、共通Middlewareによる`users`の存在確認のみである。

そのため、以下のようなクラスはCSV-001のためには作成しない。

```text
MonthEndAssetBalanceCsvTemplateQuery
AssetAccountQuery
MonthEndAssetSnapshotQuery
```

不要な抽象化を追加しない。

---

#### 2.19 API Resource

本APIの正常レスポンスは、JSONではなくCSVファイルである。

そのため、正常レスポンス用のLaravel API Resourceは使用しない。

以下のようなResourceは作成しない。

```php
final class MonthEndAssetBalanceCsvTemplateResource
    extends JsonResource
{
}
```

CSVファイルを直接HTTPレスポンスとして返却する。

---

#### 2.20 Responder

CSVダウンロードレスポンスの生成は、専用Responderで行う。

概念例：

```php
final class MonthEndAssetBalanceCsvTemplateResponder
{
    public function download(
        MonthEndAssetBalanceCsvTemplate $template,
    ): StreamedResponse {
        return response()->streamDownload(
            static function () use (
                $template,
            ): void {
                echo $template->content;
            },
            $template->fileName,
            [
                'Content-Type'
                    => 'text/csv; charset=UTF-8',
            ],
        );
    }
}
```

Responderでは、CSV内容そのものを生成しない。

---

#### 2.21 Responderの責務

Responderは、生成済みCSVテンプレートをHTTPレスポンスへ変換することに責務を限定する。

主に以下を扱う。

```text
HTTPステータス
Content-Type
Content-Disposition
レスポンスボディ
```

Responderでは、以下を行わない。

- データベース検索
- 利用者境界判定
- CSVヘッダー定義
- CSV生成
- CSV項目検証
- CSV内容の業務判定
- 月末資産残高登録

---

#### 2.22 Content-Type

正常時の`Content-Type`は、以下とする。

```http
Content-Type: text/csv; charset=UTF-8
```

CSVファイルをJSONとして返却しない。

---

#### 2.23 Content-Disposition

CSVファイルとしてダウンロードできるよう、`Content-Disposition`を設定する。

概念的には、以下とする。

```http
Content-Disposition: attachment; filename="month-end-asset-balances-template.csv"
```

ファイル名は、CSV Definitionで一元管理する。

ActionやResponderへ文字列リテラルとして重複定義しない。

---

#### 2.24 正常時のJSON Envelope

正常時は、API共通のJSON成功Envelopeを使用しない。

以下のような形式にはしない。

```json
{
  "data": {
    "content": "target_year_month,asset_account_name,balance"
  }
}
```

CSVファイルを直接レスポンスとして返却する。

---

#### 2.25 エラー時のレスポンス

エラー時は、CSVではなくAPI共通のJSONエラーレスポンスを使用する。

```text
正常時
    → text/csv

エラー時
    → application/json
```

MiddlewareまたはException Handlerで発生した例外は、API共通方針に従ってJSONへ変換する。

---

#### 2.26 例外変換

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| CSV生成処理失敗 | `INTERNAL_SERVER_ERROR` |
| その他想定外例外 | `INTERNAL_SERVER_ERROR` |

CSV-001では、CSV内容に関する業務例外は発生しない。

---

#### 2.27 想定外例外

CSV生成処理などで想定外の例外が発生した場合は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- スタックトレース
- Laravel内部例外メッセージ
- PHP内部エラー
- ファイルパス
- クラス内部情報
- SQL
- データベース接続情報

詳細情報は、サーバーログへ記録する。

---

#### 2.28 トランザクション

CSV-001では、明示的な`DB::transaction()`を使用しない。

以下のような実装は行わない。

```php
DB::transaction(
    function () use (
        $useCase,
    ) {
        return $useCase->execute();
    },
);
```

本APIでは、業務データを更新しないため、トランザクションは不要である。

---

#### 2.29 ロック

CSV-001では、行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

CSVテンプレート取得によって、CSV登録処理や月末資産関連処理を不要にブロックしない。

---

#### 2.30 キャッシュ

Phase1では、CSVテンプレート専用のサーバー側アプリケーションキャッシュを使用しない。

CSVテンプレートはヘッダー行のみであり、生成コストが非常に小さい。

また、CSV仕様変更後に古いテンプレートがキャッシュされ続ける複雑性を避ける。

必要になった場合は、将来的にHTTPキャッシュなどを別途検討する。

---

#### 2.31 CSV-002・CSV-003との共通化

CSV-001、CSV-002、CSV-003では、同一の

```text
MonthEndAssetBalanceCsvDefinition
```

を使用する。

概念的には、以下とする。

```text
CSV-001
テンプレート生成
    ↓
MonthEndAssetBalanceCsvDefinition

CSV-002
ヘッダー検証
    ↓
MonthEndAssetBalanceCsvDefinition

CSV-003
ヘッダー検証
    ↓
MonthEndAssetBalanceCsvDefinition
```

CSVヘッダーを各APIへ重複定義しない。

---

#### 2.32 CSV Generatorの共通化範囲

CSV Generatorは、CSV-001のテンプレート生成を担当する。

CSV-002およびCSV-003は、CSVファイルを読み込む側であるため、Generator自体を共通利用する必要はない。

共通化する対象は、主に以下とする。

```text
ヘッダー
文字コード
BOM方針
改行コード
CSV仕様
```

ParserとGeneratorを無理に1クラスへ統合しない。

---

#### 2.33 ログ

CSV-001では、API共通のアクセスログを使用する。

必要に応じて、以下の情報をログコンテキストへ設定する。

```text
requestId
userId
apiId
```

`apiId`は、

```text
CSV-001
```

とする。

CSVテンプレート内容をログへ出力する必要はない。

---

#### 2.34 テスト実装方針

Laravel側では、Feature Testを中心としてCSV-001のAPI契約を確認する。

主に以下を確認する。

- `200 OK`
- `400 Bad Request`
- `404 Not Found`
- `500 Internal Server Error`
- `X-User-Id`必須
- `X-User-Id`形式検証
- 利用者存在確認
- 論理削除済み利用者の除外
- `Content-Type`
- `Content-Disposition`
- ファイル名
- CSVヘッダー
- CSVヘッダー順序
- データ行が存在しないこと
- 利用者固有データが含まれないこと
- JSON成功Envelopeを使用しないこと
- 業務データが更新されないこと
- 冪等性

---

#### 2.35 CSV DefinitionのUnit Test

CSV Definitionについて、以下のヘッダーが正しい順序で定義されていることを確認する。

```php
[
    'target_year_month',
    'asset_account_name',
    'balance',
]
```

ファイル名についても、

```text
month-end-asset-balances-template.csv
```

となることを確認する。

---

#### 2.36 CSV GeneratorのUnit Test

CSV Generatorについて、生成結果がCSV共通仕様を満たすことを確認する。

主に以下を確認する。

- ヘッダーが存在すること
- ヘッダー順序が正しいこと
- 不要なデータ行が存在しないこと
- UTF-8で生成されること
- BOMが仕様どおりであること
- 改行コードが仕様どおりであること
- 正常にCSV形式として生成されること

概念的な期待値は、以下とする。

```csv
target_year_month,asset_account_name,balance
```

---

#### 2.37 CSV Generator異常系のUnit Test

CSV Generatorでストリーム生成やCSV出力が失敗した場合に、正常な空CSVを返却せず例外となることを確認する。

概念的には、

```text
CSV生成失敗
    ↓
CsvGenerationException
    ↓
共通Exception Handler
    ↓
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

となることを確認する。

---

#### 2.38 UseCaseのUnit Test

UseCaseについて、CSV Generatorの生成結果をCSV Template DTOとして返却できることを確認する。

概念例：

```text
MonthEndAssetBalanceCsvGenerator
    ↓
CSV文字列

MonthEndAssetBalanceCsvDefinition
    ↓
ファイル名

DownloadMonthEndAssetBalanceCsvTemplateUseCase
    ↓
MonthEndAssetBalanceCsvTemplate
```

UseCase実行時に、データベース検索が必要ない構造となっていることも確認する。

---

#### 2.39 ResponderのTest

Responderについて、生成済みCSV Template DTOから正しいHTTPレスポンスを生成できることを確認する。

主に以下を確認する。

```text
HTTP 200
Content-Type
Content-Disposition
ファイル名
CSVレスポンスボディ
```

正常時に

```text
application/json
```

とならないことも確認する。

---

#### 2.40 CSV-002・CSV-003との整合性テスト

CSV-001、CSV-002、CSV-003で同一のCSV Definitionを使用することを基本とする。

少なくとも、以下の整合性を自動テストで確認する。

```text
CSV-001
生成ヘッダー
    =
MonthEndAssetBalanceCsvDefinition::HEADERS
```

```text
CSV-002
期待ヘッダー
    =
MonthEndAssetBalanceCsvDefinition::HEADERS
```

```text
CSV-003
期待ヘッダー
    =
MonthEndAssetBalanceCsvDefinition::HEADERS
```

これにより、公式テンプレートを使用したCSVがCSV-002またはCSV-003でヘッダー不正になることを防止する。

---

### 3. 関連ドキュメント

- [CSV-001 API詳細設計](../../../api/details/csv-imports/csv-001-template.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [CSVインポート Laravelアーキテクチャ設計](./README.md)
- [CSV-001 テスト設計](../../../tests/csv-imports/csv-001-template.md)
- [CSVインポート テスト設計](../../../tests/csv-imports/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
