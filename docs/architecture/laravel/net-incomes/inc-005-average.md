# INC-005 平均手取り収入取得

## 1. 概要

本ドキュメントでは、
INC-005 平均手取り収入取得APIを
Laravelで実装する際の
アーキテクチャおよび責務分離方針を定義する。

本APIでは、
操作対象となる利用者について、
指定された判定対象年月の直前にあたる
連続する3か月の手取り収入を取得し、
平均手取り収入を算出して返却する。

平均手取り収入は、
目的達成判定で利用する
業務上の計算結果であり、
データベースには保存せず、
取得時点の手取り収入をもとに
都度算出する。
Laravel実装では、
以下の責務に分離する。

- Action
  - HTTPリクエストの受付
  - 利用者コンテキストの取得
  - 判定対象年月の受け取り
  - UseCaseの呼び出し
  - Responderへの処理結果の受け渡し

- UseCase
  - 平均手取り収入取得処理の制御
  - 算出対象となる3か月の特定
  - Queryによる手取り収入の取得
  - 算出対象データが揃っていることの確認
  - Domain Serviceによる平均手取り収入の算出

- Form Request / DTO
  - 判定対象年月の受け取り
  - 入力値のバリデーション
  - 入力用DTOへの変換

- Query
  - 操作対象利用者に帰属する算出対象期間の手取り収入の取得

- Domain Service
  - 3か月分の手取り収入を利用した平均値の算出

- Responder
  - API共通方針に従ったHTTPレスポンスへの変換
  - データ不足およびその他のエラーレスポンスへの変換

- API Resource / DTO
  - 算出結果からAPIレスポンス形式への変換
  - 判定対象年月、平均手取り収入および算出対象年月一覧の返却

算出対象年月の決定では、
文字列操作によって年月を直接計算せず、
年月を扱う値オブジェクトまたは
日付ライブラリを利用する。

また、
Queryから取得した件数だけで判定せず、
必要となる連続した3か月の対象年月と
実際に取得した対象年月が
完全に一致することを確認する。
算出対象となる3か月のうち、
1か月以上の手取り収入が未登録の場合は、
平均値を算出せず、
`NET_INCOME_DATA_INSUFFICIENT`として扱う。

なお、
金額が0円として登録されている手取り収入は、
登録済みの有効なデータとして
平均値の算出対象に含める。

本APIは参照および計算処理のみを行い、
手取り収入および平均手取り収入の算出結果について、
データベースへの登録や更新は行わない。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
判定対象年月および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの判定対象年月を受け取り、
平均手取り収入取得UseCaseを呼び出す。

UseCaseから受け取った処理結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 入力値のバリデーション
- 算出対象年月の決定
- 手取り収入の取得
- データ不足の判定
- 平均手取り収入の算出
- レスポンス生成処理

---

### 2.2 UseCase

平均手取り収入取得の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 判定対象年月を受け取る
- 判定対象年月直前の連続する3か月を特定する
- Queryを呼び出して対象期間の手取り収入を取得する
- 算出対象となる3か月のデータが揃っていることを確認する
- Domain Serviceを呼び出して平均手取り収入を算出する
- 算出結果および算出対象年月を返却する

算出対象となる手取り収入が不足している場合は、
`NET_INCOME_DATA_INSUFFICIENT`として扱う。

本APIは参照処理であるため、
手取り収入および算出結果を
データベースへ保存しない。

---

### 2.3 Form Request / DTO

クエリパラメータの形式および
単項目バリデーションを担当する。

検証対象は、
`targetYearMonth`とする。

以下を検証する。

- 必須であること
- 文字列であること
- `null`でないこと
- `YYYY-MM`形式であること
- 実在する年月であること
- 未定義のクエリパラメータが含まれていないこと

手取り収入データが3か月分存在するかどうかは、
データベースの状態に依存する業務ルールであるため、
Form Requestでは検証しない。

検証済みの入力値は、
入力用DTOへ変換してUseCaseへ渡す。

DTOの例：

```php
final readonly class GetAverageNetIncomeInput
{
    public function __construct(
        public string $targetYearMonth,
    ) {
    }
}
```

## 2.4 対象年月の計算

判定対象年月から、直前の連続する3か月を算出する。

例えば、判定対象年月が `2026-08` の場合は、以下を算出対象とする。

- `2026-05`
- `2026-06`
- `2026-07`

年月の加減算は、文字列の切り出しや数値の直接加算ではなく、年月を扱える値オブジェクトまたは日付ライブラリを利用する。

Laravel実装では、Carbonを利用する場合でも、年月計算をActionやQueryへ分散させず、専用の値オブジェクトまたはDomain Serviceへ集約する。

値オブジェクトの例：

```php
final readonly class YearMonth
{
    public function __construct(
        public int $year,
        public int $month,
    ) {
    }

    public static function fromString(string $value): self
    {
        [$year, $month] = array_map(
            'intval',
            explode('-', $value),
        );

        return new self($year, $month);
    }

    public function subtractMonths(int $months): self
    {
        $date = CarbonImmutable::create(
            $this->year,
            $this->month,
            1,
        )->subMonthsNoOverflow($months);

        return new self(
            $date->year,
            $date->month,
        );
    }

    public function toString(): string
    {
        return sprintf(
            '%04d-%02d',
            $this->year,
            $this->month,
        );
    }
}
```

---

## 2.5 Query

操作対象利用者について、算出対象となる3か月の手取り収入を取得する。

検索条件には、必ず操作対象利用者IDを含める。

取得例：

```php
$netIncomes = NetIncome::query()
    ->where('user_id', $userId)
    ->whereIn(
        'target_year_month',
        $calculatedMonths,
    )
    ->orderBy('target_year_month')
    ->get([
        'target_year_month',
        'amount',
    ]);
```

取得対象となるカラムは、以下とする。

- `target_year_month`
- `amount`

以下の項目は、平均値の算出に使用しないため取得不要とする。

- `id`
- `memo`
- `created_at`
- `updated_at`

取得件数が3件であることだけでなく、必要な3か月がすべて揃っていることを確認する。

例えば、意図しないデータを含む3件が取得された場合でも、算出対象年月との一致を検証する。

---

## 2.6 Domain Service

平均手取り収入の算出を担当する。

Domain Serviceは、算出対象となる3か月の手取り収入金額を受け取り、平均値を返却する。

例：

```php
final class AverageNetIncomeCalculator
{
    /**
     * @param list<int> $amounts
     */
    public function calculate(array $amounts): int
    {
        if (count($amounts) !== 3) {
            throw new InvalidArgumentException(
                '平均手取り収入の算出には3件の金額が必要です。',
            );
        }

        return intdiv(
            array_sum($amounts),
            count($amounts),
        );
    }
}
```

平均値の端数処理は、機能要件またはAPI設計で定めたルールに従う。

現時点で端数処理が未確定の場合は、`intdiv()` による切り捨てを暗黙に採用せず、以下のいずれを採用するか確定してから実装する。

- 切り捨て
- 四捨五入
- 切り上げ

計算ロジックをUseCaseやResponderへ直接記述しない。

---

## 2.7 データ不足の判定

Queryから取得した結果について、算出対象となる3か月がすべて登録されていることを確認する。

確認対象は、以下とする。

- 取得件数が3件であること
- 取得した対象年月が算出対象年月と完全に一致すること
- 同一対象年月のデータが重複していないこと

金額が0円のデータは、登録済みの有効なデータとして扱う。

以下の場合は、`NET_INCOME_DATA_INSUFFICIENT` を発生させる。

- 算出対象のうち1か月以上が未登録である
- 必要な3か月と取得結果の対象年月が一致しない
- 連続する3か月のデータを取得できない

例：

```php
$actualMonths = $netIncomes
    ->pluck('target_year_month')
    ->all();

if ($actualMonths !== $calculatedMonths) {
    throw new NetIncomeDataInsufficientException(
        requiredMonths: $calculatedMonths,
        actualMonths: $actualMonths,
    );
}
```

---

## 2.8 トランザクション

本APIは参照処理のみであるため、明示的なデータベーストランザクションは使用しない。

手取り収入の取得および平均手取り収入の算出によって、データベースの状態を変更しない。

Phase1では、算出処理中の厳密なスナップショット分離は保証しない。

---

## 2.9 Responder

UseCaseから受け取った算出結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`200 OK` とともに以下を `data` オブジェクトで返却する。

- 判定対象年月
- 平均手取り収入
- 算出対象年月一覧

データ不足の場合は、`422 Unprocessable Entity` および `NET_INCOME_DATA_INSUFFICIENT` へ変換する。

Responderは、算出対象年月の決定や平均値の計算を行わない。

---

## 2.10 API Resource

計算結果をAPIレスポンス形式へ変換する。

Eloquent Modelを直接返却するAPIではないため、算出結果用のDTOまたはResourceを使用する。

算出結果DTOの例：

```php
final readonly class AverageNetIncomeResult
{
    /**
     * @param list<string> $calculatedMonths
     */
    public function __construct(
        public string $targetYearMonth,
        public int $averageAmount,
        public array $calculatedMonths,
    ) {
    }
}
```

Resourceの変換例：

```php
return [
    'targetYearMonth' => $this->targetYearMonth,
    'averageAmount' => $this->averageAmount,
    'calculatedMonths' => $this->calculatedMonths,
];
```

`calculatedMonths` は、古い対象年月から新しい対象年月の順で返却する。

```json
{
  "calculatedMonths": [
    "2026-05",
    "2026-06",
    "2026-07"
  ]
}
```

---

## 2.11 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id` を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

## 2.12 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 判定対象年月の形式不正 | `INVALID_TARGET_YEAR_MONTH` |
| 算出対象データ不足 | `NET_INCOME_DATA_INSUFFICIENT` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

SQL、スタックトレースおよび内部例外メッセージは、APIレスポンスへ含めない。

データ不足例外には、ログ調査に必要な範囲で以下の情報を記録する。

- 操作対象利用者ID
- 判定対象年月
- 必要な算出対象年月
- 実際に取得できた対象年月
- リクエストID

手取り収入金額などの業務データを不要にエラーログへ出力しない。

---

## 3. 関連ドキュメント

- [INC-005 API詳細設計](../../../api/details/net-incomes/inc-005-average.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [手取り収入 Laravelアーキテクチャ設計](./README.md)
- [INC-005 テスト設計](../../../tests/net-incomes/inc-005-average.md)
- [手取り収入 テスト設計](../../../tests/net-incomes/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)