# SNP-002 月末資産状況作成

## 概要

操作対象となる利用者について、
指定した対象年月の
月末資産状況を新規作成するAPI。

月末資産状況は、
月末資産残高および
商品別月末評価額を管理するための
親リソースとして扱う。

作成時には、
以下の情報を登録する。

- 操作対象利用者ID
- 対象年月
- 確定状態

作成直後の確定状態は、
必ず未確定とする。

```text
confirmed = false
```

本APIでは、
月末資産状況のみを作成し、
以下の処理は行わない。

- 月末資産残高の登録
- 商品別月末評価額の登録
- 月末資産状況の確定
- 月末資産状況の確定解除

Laravelでは、
以下の責務を分離して実装する。

- Action
  - HTTPリクエストの受付
  - 利用者コンテキストの取得
  - UseCaseの呼び出し
- Form Request / DTO
  - 対象年月の入力検証
  - 検証済み入力値の受け渡し
- UseCase
  - 作成処理全体の制御
  - 重複確認
  - Repositoryの呼び出し
- Query
  - 同一利用者・同一対象年月の存在確認
- Repository
  - 月末資産状況の永続化
- Responder
  - HTTPレスポンスへの変換
- API Resource
  - APIレスポンス形式への変換

入力値として受け付けるのは、
原則として `targetYearMonth` のみとする。

`targetYearMonth` は、
`YYYY-MM` 形式の
実在する年月であることを検証する。

以下のサーバー管理項目は、
クライアントから受け付けない。

- `id`
- `userId`
- `confirmed`
- `confirmedAt`
- `createdAt`
- `updatedAt`

同一利用者・同一対象年月について、
すでに月末資産状況が存在する場合は、
新規作成を行わず、

`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`

として扱う。

重複確認および作成処理は、
1つのデータベーストランザクション内で実行する。

ただし、
アプリケーション側の事前確認だけでは
同時実行時の重複を完全には防止できないため、

`user_id` と `target_year_month`

の組み合わせに設定された
UNIQUE制約を
最終的な整合性保証として使用する。

UNIQUE制約違反が発生した場合も、
PostgreSQLの内部例外をそのまま公開せず、

`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`

へ変換する。

登録値は、
クライアント入力をそのままModelへ渡さず、
検証済みDTOと
利用者コンテキストから
明示的に組み立てる。

そのため、
以下のようなMass Assignmentは行わない。

```php
MonthEndAssetSnapshot::create(
    $request->all(),
);
```

正常終了時は、
`201 Created` とともに、
作成した月末資産状況の以下の情報を返却する。

- 月末資産状況ID
- 対象年月
- 確定状態

APIレスポンスでは、
IDを文字列、
対象年月を `YYYY-MM` 形式、
確定状態をbooleanとして返却する。

作成直後であるため、
`confirmed` は必ず `false` となる。

LaravelやPostgreSQLの内部例外、
SQL、
スタックトレース、
制約名などの内部情報は
APIレスポンスへ公開せず、
API共通方針に従って
独自エラーコードへ変換する。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
入力値および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの対象年月を受け取り、
月末資産状況作成UseCaseを呼び出す。

UseCaseから受け取った作成結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 入力値の単項目バリデーション
- 利用者境界の判定
- 同一対象年月の重複確認
- 月末資産状況の作成
- トランザクション制御
- レスポンス生成処理

---

### 2.2 UseCase

月末資産状況作成の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 対象年月を受け取る
- 同一利用者・同一対象年月の月末資産状況が存在しないことを確認する
- 新しい月末資産状況を作成する
- 作成結果を返却する

すでに同一対象年月の
月末資産状況が存在する場合は、
`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`
として扱う。

作成時の確定状態は、
必ず未確定とする。

```text
confirmed = false
```
本UseCaseでは、以下の処理は行わない。

- 月末資産残高の登録
- 商品別月末評価額の登録
- 確定条件の判定
- 月末資産状況の確定
- 月末資産状況の確定解除

---

### 2.3 Form Request / DTO

入力値の形式および単項目バリデーションを担当する。

検証対象は、`targetYearMonth`とする。

Form Requestでは、以下を検証する。

- 必須であること
- `null`でないこと
- 文字列であること
- `YYYY-MM`形式であること
- 実在する年月であること
- APIで許容する年月範囲内であること
- 未定義項目が含まれていないこと
- サーバー管理項目が含まれていないこと

以下の項目は、クライアントから受け付けない。

- `id`
- `userId`
- `confirmed`
- `confirmedAt`
- `createdAt`
- `updatedAt`

同一対象年月の重複確認は、データベース状態に依存する業務ルールであるため、Form Requestでは行わない。

検証済みの入力値は、入力用DTOへ変換してUseCaseへ渡す。

DTOの例：

```php
final readonly class CreateMonthEndAssetSnapshotInput
{
    public function __construct(
        public string $targetYearMonth,
    ) {
    }
}
```

---

### 2.4 Query

同一利用者・同一対象年月の月末資産状況が存在するか確認する。

検索条件には、必ず操作対象利用者IDを含める。

```php
$exists = MonthEndAssetSnapshot::query()
    ->where('user_id', $userId)
    ->where(
        'target_year_month',
        $input->targetYearMonth,
    )
    ->exists();
```

存在する場合は、`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`として扱う。

本Queryでは、月末資産残高および商品別月末評価額を参照しない。

---

### 2.5 Repository

月末資産状況の作成を担当する。

登録対象は、以下とする。

- `user_id`
- `target_year_month`
- `confirmed`

登録時には、以下の値を設定する。

```text
user_id           = 操作対象利用者ID
target_year_month = 指定された対象年月
confirmed         = false
```

登録例：

```php
return MonthEndAssetSnapshot::create([
    'user_id' => $userId,
    'target_year_month' => $input->targetYearMonth,
    'confirmed' => false,
]);
```

`id`、`created_at`および`updated_at`は、Laravelおよびデータベース側で設定する。

本Repositoryでは、以下の処理は行わない。

- 月末資産残高の作成
- 商品別月末評価額の作成
- 確定処理
- 確定解除処理

---

### 2.6 トランザクション

以下の処理を、1つのデータベーストランザクション内で実行する。

- 同一利用者・同一対象年月の重複確認
- 月末資産状況の作成

実装例：

```php
$snapshot = DB::transaction(
    function () use (
        $userId,
        $input,
    ): MonthEndAssetSnapshot {
        if (
            $this->query
                ->existsByUserAndTargetYearMonth(
                    $userId,
                    $input->targetYearMonth,
                )
        ) {
            throw new
                MonthEndAssetSnapshotAlreadyExistsException();
        }

        return $this->repository->create(
            $userId,
            $input,
        );
    },
);
```

処理途中で例外が発生した場合は、作成処理をロールバックする。

アプリケーション側の事前重複確認だけでは、同時実行時の重複を完全には防止できない。

そのため、データベースのUNIQUE制約を最終的な整合性保証として使用する。

---

### 2.7 UNIQUE制約違反の扱い

同一利用者・同一対象年月について、以下の組み合わせにUNIQUE制約を設定する。

```text
user_id
target_year_month
```

複数のリクエストが同時に実行され、アプリケーション側の重複確認を同時に通過した場合でも、データベース制約によって1件のみ登録されることを保証する。

PostgreSQLのUNIQUE制約違反が発生した場合は、内部例外をそのまま公開せず、`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`へ変換する。

PostgreSQLのエラーコード、制約名およびSQLはレスポンスへ含めない。

---

### 2.8 Responder

UseCaseから受け取った作成結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`201 Created`とともに作成した月末資産状況を`data`オブジェクトで返却する。

以下の場合は、共通エラーレスポンス形式へ変換する。

- 入力値が不正である
- 同一対象年月がすでに存在する
- 利用者が存在しない
- 想定外の例外が発生した

Responderは、以下の処理を行わない。

- 重複確認
- データベース登録
- 確定状態の決定
- 確定可否判定

---

### 2.9 API Resource

データベースカラムを直接返却せず、API Resourceを利用してAPIレスポンス形式へ変換する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'targetYearMonth' => $this->target_year_month,
    'confirmed' => (bool) $this->confirmed,
];
```

作成直後のため、`confirmed`は必ず`false`となる。

以下の項目は、レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`
- 月末資産残高
- 商品別月末評価額

---

### 2.10 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 2.11 Eloquentモデル

`MonthEndAssetSnapshot`モデルは、`month_end_asset_snapshots`テーブルへ対応する。

登録対象として、以下の属性を使用する。

```text
user_id
target_year_month
confirmed
```

`confirmed`は、booleanとして扱えるようにcastを設定する。

```php
protected function casts(): array
{
    return [
        'confirmed' => 'boolean',
    ];
}
```

`target_year_month`は、通常の日付ではなく、`YYYY-MM`形式の年月を表す業務値として扱う。

Phase1では、Eloquentのdate castは設定しない。

---

### 2.12 Mass Assignment

Mass Assignmentを利用する場合は、クライアント入力をそのままModelへ渡さない。

以下のような実装は禁止する。

```php
MonthEndAssetSnapshot::create(
    $request->all(),
);
```

これにより、クライアントから以下のようなサーバー管理項目を意図せず変更されることを防止する。

- `user_id`
- `confirmed`

登録値は、検証済みDTOおよび利用者コンテキストから明示的に組み立てる。

---

### 2.13 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 同一対象年月重複 | `MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS` |
| 入力値不正 | `VALIDATION_ERROR` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

UNIQUE制約違反については、対象となる制約を判別し、`MONTH_END_ASSET_SNAPSHOT_ALREADY_EXISTS`へ変換する。

SQL、スタックトレース、PostgreSQLの制約名および内部例外メッセージは、APIレスポンスへ含めない。

ログには、調査に必要な範囲で以下を記録する。

- 操作対象利用者ID
- 対象年月
- 独自エラーコード
- リクエストID

月末資産残高や商品別月末評価額など、本APIで扱わない業務データを不要にログへ出力しない。

---

## 3. 関連ドキュメント

- [SNP-002 API詳細設計](../../../api/details/month-end-asset-snapshots/snp-002-create.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [月末資産状況 Laravelアーキテクチャ設計](./README.md)
- [SNP-002 テスト設計](../../../tests/month-end-asset-snapshots/snp-002-create.md)
- [月末資産状況 テスト設計](../../../tests/month-end-asset-snapshots/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)