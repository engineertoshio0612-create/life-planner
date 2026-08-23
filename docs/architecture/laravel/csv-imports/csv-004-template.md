##  CSV-004 商品別月末評価額CSVテンプレート取得

### 1 概要

CSV-004では、商品単位で管理する保有商品の月末評価額をCSVインポートする際に使用するCSVテンプレートを取得するためのLaravel実装方針を定義する。

本APIでは、商品別月末評価額CSVで必要となる以下のヘッダーを持つCSVテンプレートを生成し、CSVファイルとして返却する。

```text
target_year_month
asset_account_name
holding_asset_name
value
```

生成するCSVテンプレートは、CSV-005 商品別月末評価額CSVプレビューおよびCSV-006 商品別月末評価額CSV登録で使用する入力形式とする。

CSVテンプレートはヘッダー行のみで構成し、資産口座、保有商品、商品別月末評価額などの利用者固有データは出力しない。

また、本APIはCSVテンプレートの取得のみを責務とし、以下の処理は行わない。

- 商品別月末評価額の登録
- 商品別月末評価額の更新
- 商品別月末評価額の削除
- CSVプレビュー
- CSV入力値検証
- 対象年月の判定
- 重複登録判定
- 月末資産状況の作成
- 月末資産状況の確定

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
Template DTO
    ↓
Responder
    ↓
CSVダウンロードレスポンス
```

Actionは、商品別月末評価額CSVテンプレート取得UseCaseの呼び出しと、生成結果のResponderへの受け渡しに責務を限定する。

UseCaseでは、CSV Generatorを使用してCSVテンプレートを生成し、CSV内容およびファイル名をTemplate DTOとしてResponderへ返却する。

CSV-004では利用者固有の業務データをテンプレートへ出力しないため、資産口座、保有商品、月末資産状況、商品別月末評価額などを検索するRepositoryまたはQueryクラスは使用しない。

また、パスパラメータ、クエリパラメータ、リクエストボディを使用しないため、CSV-004専用のFormRequestは作成しない。`X-User-Id`の検証および利用者コンテキストの設定は、API共通Middlewareで行う。

CSV-004、CSV-005、CSV-006で使用するCSVヘッダー、ファイル名などの商品別月末評価額CSV仕様は、共通のCSV Definitionへ集約する。

CSV Generatorでは、PHP標準の`fputcsv()`を使用してCSVを生成し、CSV形式の生成に責務を限定する。HTTPレスポンス生成や業務データの検索は行わない。

CSVテンプレートはUTF-8で生成し、BOMおよび改行コードについてもCSV関連APIの共通仕様に従う。CSV-004だけ異なる文字コード、BOM、改行コードの仕様を持たせない。

正常時はJSON成功Envelopeを使用せず、生成したCSVテンプレートを`text/csv; charset=UTF-8`のダウンロードレスポンスとして直接返却する。

一方、利用者コンテキストの不備やCSV生成処理の失敗などのエラー時は、API共通方針に従ってJSON形式のエラーレスポンスを返却する。

CSV-004では業務データを更新しないため、トランザクションおよび行ロックは使用しない。また、業務データを取得しないためN+1問題も発生しない。

Phase1ではCSVテンプレートがヘッダー1行のみで生成コストが非常に小さいことから、CSV-004専用のサーバー側アプリケーションキャッシュも使用しない。

CSV-004、CSV-005、CSV-006では同一のCSV Definitionを使用し、CSV-004で生成した公式テンプレートがCSV-005またはCSV-006でヘッダー不正と判定されないよう、CSV仕様の整合性を維持する。

---

### 2 Laravel実装方針

CSV-004では、
Action、
UseCase、
CSV Definition、
CSV Generator、
DTO、
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
Template DTO
    ↓
Responder
    ↓
CSVダウンロードレスポンス
```

本APIでは、
利用者固有の業務データを
CSVテンプレートへ出力しない。

そのため、
テンプレート生成のための

- Repository
- Query
- API Resource

は使用しない。

また、

- パスパラメータ
- クエリパラメータ
- リクエストボディ

を使用しないため、
CSV-004専用の
FormRequestも作成しない。

---

#### 2.1 Action

HTTPリクエストを受け付け、
商品別月末評価額CSVテンプレート取得UseCaseを呼び出し、
生成結果をResponderへ渡す。

概念例：

```php
final class DownloadMonthEndHoldingValueCsvTemplateAction
{
    public function __invoke(
        DownloadMonthEndHoldingValueCsvTemplateUseCase $useCase,
        MonthEndHoldingValueCsvTemplateResponder $responder,
    ): StreamedResponse {
        $template =
            $useCase->execute();

        return $responder->download(
            $template,
        );
    }
}
```

Actionでは、
以下を行わない。

- CSVヘッダー定義
- CSV文字列生成
- CSVエスケープ処理
- 文字コード変換
- BOM付与
- ファイル名決定
- `Content-Type`設定
- `Content-Disposition`生成
- 資産口座検索
- 保有商品検索
- 商品別月末評価額検索
- 利用者存在確認

Actionは、
UseCaseの呼び出しと
Responderへの受け渡しに
責務を限定する。

---

#### 2.2 FormRequest

CSV-004専用の
FormRequestは作成しない。

本APIでは、

```text
パスパラメータなし
クエリパラメータなし
リクエストボディなし
```

であり、
CSV-004固有の
入力バリデーションが存在しないためである。

`X-User-Id`の検証は、
API共通Middlewareで行う。

以下のような
空のFormRequestは作成しない。

```php
final class DownloadMonthEndHoldingValueCsvTemplateRequest
    extends FormRequest
{
}
```

不要なクラスを
追加しない。

---

#### 2.3 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
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
Action
```

利用者存在確認条件は、
概念的に以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

Action以降では、
利用者存在確認を
再実装しない。

---

#### 2.4 UseCase

商品別月末評価額CSVテンプレート取得の
ユースケース処理を担当する。

概念例：

```php
final class DownloadMonthEndHoldingValueCsvTemplateUseCase
{
    public function __construct(
        private readonly
        MonthEndHoldingValueCsvGenerator $csvGenerator,
    ) {
    }

    public function execute():
        MonthEndHoldingValueCsvTemplate
    {
        return new MonthEndHoldingValueCsvTemplate(
            content:
                $this->csvGenerator
                    ->generateTemplate(),

            fileName:
                MonthEndHoldingValueCsvDefinition::FILE_NAME,
        );
    }
}
```

CSVテンプレートの内容は、
操作対象利用者によって
変化しない。

そのため、
UseCaseへ`userId`を
渡す必要はない。

利用者コンテキストの確認自体は、
Action到達前に
Middlewareで完了していることを
前提とする。

---

#### 2.5 UseCaseで行わないこと

UseCaseでは、
以下を行わない。

- `users`の再検索
- `asset_accounts`の検索
- `holding_assets`の検索
- `month_end_asset_snapshots`の検索
- `month_end_holding_values`の検索
- `asset_account_available_settings`の検索
- 利用者固有データのCSV出力
- 商品別月末評価額の登録
- 商品別月末評価額の更新
- 月末資産状況の作成
- 月末資産状況の確定

CSVテンプレート生成に
必要な処理だけを行う。

---

#### 2.6 Template DTO

CSV生成結果は、
文字列だけを
Responderへ渡すのではなく、
必要な情報をまとめた
専用DTOとして表現する。

概念例：

```php
final readonly class MonthEndHoldingValueCsvTemplate
{
    public function __construct(
        public string $content,
        public string $fileName,
    ) {
    }
}
```

これにより、
UseCaseからResponderへ渡す
テンプレート情報を
明示的に表現できる。

将来的に、

- 文字コード
- BOM有無
- MIME Type

などを
テンプレート単位で管理する必要が
生じた場合にも拡張しやすくなる。

---

#### 2.7 CSV Definition

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

    public const FILE_NAME =
        'month-end-holding-values-template.csv';
}
```

CSV-004だけで
独自のヘッダーを定義しない。

CSV-005、
CSV-006でも
同じDefinitionを使用する。

---

#### 2.8 CSVヘッダー

CSVテンプレートでは、
以下のヘッダーを
この順序で出力する。

```text
target_year_month
asset_account_name
holding_asset_name
value
```

生成結果は、
以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

ヘッダー名だけでなく、
ヘッダー順序も
CSV仕様の一部として扱う。

---

#### 2.9 CSV Generator

CSV文字列の生成は、
専用Generatorで行う。

概念例：

```php
final class MonthEndHoldingValueCsvGenerator
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
            $result =
                fputcsv(
                    $stream,
                    MonthEndHoldingValueCsvDefinition::HEADERS,
                );

            if ($result === false) {
                throw new CsvGenerationException();
            }

            rewind(
                $stream,
            );

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

Generatorは、
CSV形式の生成に
責務を限定する。

以下を行わない。

- HTTPレスポンス生成
- 利用者検索
- 資産口座検索
- 保有商品検索
- 商品別月末評価額検索
- 商品別月末評価額登録

---

#### 2.10 fputcsvの利用

CSV生成には、
PHP標準の`fputcsv()`を使用する。

以下のような
手動連結は
基本的に行わない。

```php
implode(
    ',',
    MonthEndHoldingValueCsvDefinition::HEADERS,
);
```

`fputcsv()`へ統一することで、
CSV生成方法を
他のCSV関連処理と
揃えやすくする。

---

#### 2.11 fopen失敗時

`php://temp`の
オープンに失敗した場合は、
そのまま処理を続行しない。

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

最終的には、
API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

---

#### 2.12 fputcsv失敗時

`fputcsv()`の戻り値が
`false`の場合は、
CSV生成失敗として扱う。

概念例：

```php
$result =
    fputcsv(
        $stream,
        MonthEndHoldingValueCsvDefinition::HEADERS,
    );

if ($result === false) {
    throw new CsvGenerationException();
}
```

失敗したCSVを
正常なテンプレートとして
返却してはならない。

---

#### 2.13 stream_get_contents失敗時

ストリームから
CSV内容を取得できなかった場合も、
CSV生成失敗とする。

概念例：

```php
$csv =
    stream_get_contents(
        $stream,
    );

if ($csv === false) {
    throw new CsvGenerationException();
}
```

空文字への
フォールバックは行わない。

---

#### 2.14 リソース解放

ストリームを開いた場合は、
正常・異常にかかわらず
確実に`fclose()`する。

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

#### 2.15 文字コード

商品別月末評価額CSVテンプレートは、
UTF-8で生成する。

CSV-005、
CSV-006で受け付ける
文字コードと統一する。

CSV-004だけ
異なる文字コードを
使用しない。

---

#### 2.16 BOM

UTF-8 BOMを
付与するかどうかは、
CSV共通仕様として統一する。

BOMを付与する場合は、
Generator内で
明示的に行う。

概念例：

```php
fwrite(
    $stream,
    "\xEF\xBB\xBF",
);
```

CSV-004、
CSV-005、
CSV-006で
BOMの扱いに
不整合を発生させない。

---

#### 2.17 改行コード

CSVテンプレートの
改行コードは、
CSV関連APIで統一する。

CSV-004だけ
独自の改行コード仕様を
持たせない。

実行環境による差異を
許容しない場合は、
CSV Generator側で
明示的に制御する。

---

#### 2.18 データ行

CSV-004では、
利用者固有の
データ行を出力しない。

生成対象は、
ヘッダー行のみとする。

以下のような
資産口座・保有商品の
検索処理を行わない。

```php
$holdingAssets =
    HoldingAsset::query()
        ->with('assetAccount')
        ->get();
```

また、
以下のような
テンプレート事前展開も行わない。

```php
foreach (
    $holdingAssets
    as $holdingAsset
) {
    fputcsv(
        $stream,
        [
            '',
            $holdingAsset
                ->assetAccount
                ->name,
            $holdingAsset->name,
            '',
        ],
    );
}
```

テンプレートは、
業務データから独立した
固定形式とする。

---

#### 2.19 Repository・Query

CSV-004では、
Repositoryおよび
Queryクラスを使用しない。

本APIで必要となる
データベース参照は、
共通Middlewareによる
`users`の存在確認のみである。

そのため、
以下のような
CSV-004専用クラスは作成しない。

```text
MonthEndHoldingValueCsvTemplateQuery
AssetAccountQuery
HoldingAssetQuery
```

不要な抽象化を
追加しない。

---

#### 2.20 API Resource

CSV-004の
正常レスポンスは、
JSONではなくCSVファイルである。

そのため、
正常レスポンス用の
Laravel API Resourceは
使用しない。

以下のような
Resourceは作成しない。

```php
final class MonthEndHoldingValueCsvTemplateResource
    extends JsonResource
{
}
```

CSVファイルを
直接HTTPレスポンスとして返却する。

---

#### 2.21 Responder

CSVダウンロードレスポンスは、
専用Responderで生成する。

概念例：

```php
final class MonthEndHoldingValueCsvTemplateResponder
{
    public function download(
        MonthEndHoldingValueCsvTemplate $template,
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

Responderでは、
CSV内容そのものを
生成しない。

---

#### 2.22 Responderの責務

Responderは、
生成済みCSVテンプレートを
HTTPレスポンスへ変換することに
責務を限定する。

主に以下を扱う。

```text
HTTPステータス
Content-Type
Content-Disposition
レスポンスボディ
```

Responderでは、
以下を行わない。

- データベース検索
- 利用者存在確認
- CSVヘッダー定義
- CSV生成
- CSV仕様判定
- 資産口座検索
- 保有商品検索

---

#### 2.23 Content-Type

正常時の
`Content-Type`は、
以下とする。

```http
Content-Type: text/csv; charset=UTF-8
```

CSVファイルを
JSONとして返却しない。

---

#### 2.24 Content-Disposition

CSVファイルとして
ダウンロードできるよう、
`Content-Disposition`を設定する。

概念的には、
以下とする。

```http
Content-Disposition: attachment; filename="month-end-holding-values-template.csv"
```

ファイル名は、
CSV Definitionで
一元管理する。

ActionやResponderへ
同じ文字列を
重複定義しない。

---

#### 2.25 正常時のJSON Envelope

正常時は、
API共通の
JSON成功Envelopeを使用しない。

以下のような
レスポンスにはしない。

```json
{
  "data": {
    "content": "target_year_month,asset_account_name,holding_asset_name,value"
  }
}
```

CSVファイルを
直接レスポンスとして返却する。

---

#### 2.26 エラー時のレスポンス

エラー時は、
CSVではなく
API共通のJSONエラーレスポンスを使用する。

```text
正常時
    → text/csv

エラー時
    → application/json
```

Middlewareまたは
Exception Handlerで発生した例外は、
API共通方針に従って
JSONへ変換する。

---

#### 2.27 例外変換

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| CSV生成処理失敗 | `INTERNAL_SERVER_ERROR` |
| その他想定外例外 | `INTERNAL_SERVER_ERROR` |

CSV-004では、
CSV入力内容に関する
業務例外は発生しない。

---

#### 2.28 想定外例外

想定外の例外は、
API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、
以下を含めない。

- スタックトレース
- Laravel内部例外メッセージ
- PHP内部エラー
- サーバーファイルパス
- クラス内部情報
- SQL
- データベース接続情報

詳細情報は、
サーバーログへ記録する。

---

#### 2.29 トランザクション

CSV-004では、
明示的な
`DB::transaction()`を使用しない。

以下のような実装は
行わない。

```php
DB::transaction(
    function () {
        // CSVテンプレート生成のみ
    },
);
```

業務データを更新しないため、
トランザクションは不要である。

---

#### 2.30 ロック

CSV-004では、
行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

CSVテンプレート取得によって、
CSV-005、
CSV-006、
月末資産関連処理を
ブロックしてはならない。

---

#### 2.31 N+1問題

CSV-004では、
業務データを取得しないため、
N+1問題は発生しない。

以下のような処理を
実装しない。

```text
資産口座一覧取得
    ↓
資産口座ごとに
保有商品一覧取得
```

CSV生成処理を
固定Definitionだけで
完結させる。

---

#### 2.32 キャッシュ

Phase1では、
CSV-004専用の
サーバー側アプリケーションキャッシュを
使用しない。

CSVテンプレートは
ヘッダー1行のみであり、
生成コストが非常に小さいためである。

また、
CSV仕様変更時に
古いテンプレートが
キャッシュされ続ける
複雑性を避ける。

---

#### 2.33 CSV-005・CSV-006との共通化

CSV-004、
CSV-005、
CSV-006では、
同一の

```text
MonthEndHoldingValueCsvDefinition
```

を使用する。

概念的には、
以下とする。

```text
CSV-004
テンプレート生成
    ↓
MonthEndHoldingValueCsvDefinition

CSV-005
ヘッダー検証
    ↓
MonthEndHoldingValueCsvDefinition

CSV-006
ヘッダー検証
    ↓
MonthEndHoldingValueCsvDefinition
```

CSVヘッダーを
各APIへ重複定義しない。

---

#### 2.34 GeneratorとParserを分離する

CSV-004は、
CSVを生成する側である。

一方、
CSV-005およびCSV-006は、
アップロードされたCSVを
解析する側である。

そのため、

```text
MonthEndHoldingValueCsvGenerator
MonthEndHoldingValueCsvParser
```

は分離する。

GeneratorとParserを
1つのクラスへ
無理に統合しない。

共通化する対象は、
主に以下とする。

```text
ヘッダー
文字コード
BOM方針
改行コード
CSV仕様
```

---

#### 2.35 ログ

CSV-004では、
API共通ログ方針に従う。

必要に応じて、
以下の情報を
ログコンテキストへ設定する。

```text
requestId
userId
apiId
```

`apiId`は、

```text
CSV-004
```

とする。

CSVテンプレート本文を
ログへ出力する必要はない。

---

#### 2.36 テスト実装方針

Laravel側では、
Feature Testを中心として
CSV-004のAPI契約を確認する。

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
- CSVファイル名
- CSVヘッダー
- CSVヘッダー順序
- データ行が存在しないこと
- 利用者固有データが含まれないこと
- 内部IDが含まれないこと
- JSON成功Envelopeを使用しないこと
- 業務データを検索しない構成であること
- 業務データが更新されないこと
- 冪等性

---

#### 2.37 CSV DefinitionのUnit Test

CSV Definitionについて、
以下のヘッダーが
正しい順序で定義されていることを
確認する。

```php
[
    'target_year_month',
    'asset_account_name',
    'holding_asset_name',
    'value',
]
```

ファイル名についても、
仕様どおりであることを確認する。

```text
month-end-holding-values-template.csv
```

---

#### 2.38 CSV GeneratorのUnit Test

CSV Generatorについて、
生成結果が
CSV共通仕様を満たすことを確認する。

主に以下を確認する。

- ヘッダーが存在すること
- ヘッダー順序が正しいこと
- データ行が存在しないこと
- UTF-8で生成されること
- BOMが仕様どおりであること
- 改行コードが仕様どおりであること
- 正常なCSVとして生成されること

期待する内容は、
以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

---

#### 2.39 CSV Generator異常系のUnit Test

CSV Generatorで
ストリーム生成や
CSV出力に失敗した場合に、
正常な空CSVを返却せず
例外となることを確認する。

概念的には、

```text
CSV生成失敗
    ↓
CsvGenerationException
    ↓
API共通Exception Handler
    ↓
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

となることを確認する。

---

#### 2.40 UseCaseのUnit Test

UseCaseについて、
CSV Generatorの生成結果を
Template DTOとして
返却できることを確認する。

概念的には、

```text
MonthEndHoldingValueCsvGenerator
    ↓
CSV文字列

MonthEndHoldingValueCsvDefinition
    ↓
ファイル名

DownloadMonthEndHoldingValueCsvTemplateUseCase
    ↓
MonthEndHoldingValueCsvTemplate
```

となることを確認する。

また、
UseCase実行時に

- `asset_accounts`
- `holding_assets`
- `month_end_holding_values`

などの
業務データ検索を
必要としない構造であることを確認する。

---

#### 2.41 ResponderのTest

Responderについて、
生成済みTemplate DTOから
正しいHTTPレスポンスを
生成できることを確認する。

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

とならないことを確認する。

---

#### 2.42 CSV-005・CSV-006との整合性テスト

CSV-004、
CSV-005、
CSV-006で
同一CSV Definitionを
使用することを基本とする。

少なくとも、
以下の整合性を
自動テストで確認する。

```text
CSV-004
生成ヘッダー
    =
MonthEndHoldingValueCsvDefinition::HEADERS
```

```text
CSV-005
期待ヘッダー
    =
MonthEndHoldingValueCsvDefinition::HEADERS
```

```text
CSV-006
期待ヘッダー
    =
MonthEndHoldingValueCsvDefinition::HEADERS
```

これにより、
CSV-004で取得した
公式テンプレートを使用したCSVが、
CSV-005またはCSV-006で
ヘッダー不正となることを防止する。

---

### 3. 関連ドキュメント

- [CSV-004 API詳細設計](../../../api/details/csv-imports/csv-004-template.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [CSVインポート Laravelアーキテクチャ設計](./README.md)
- [CSV-004 テスト設計](../../../tests/csv-imports/csv-004-template.md)
- [CSVインポート テスト設計](../../../tests/csv-imports/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)