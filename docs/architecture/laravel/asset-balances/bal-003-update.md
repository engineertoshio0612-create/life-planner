# BAL-003 月末資産残高更新

## 1. 概要

本ドキュメントでは、
BAL-003 月末資産残高更新APIを
Laravelで実装する際の
アーキテクチャおよび責務分離方針を定義する。

BAL-003では、
操作対象利用者の指定された月末資産状況に対して、
残高記録単位が口座単位である資産口座に紐づく
既存の月末資産残高を更新する。

本APIは、
登録済みの月末資産残高を変更するための更新APIであり、
月末資産残高の新規登録は行わない。

更新対象となる月末資産残高は、

```text
month_end_asset_snapshot_id
+
asset_account_id
```

の組み合わせによって特定する。

該当する月末資産残高が存在しない場合は、
新しいレコードを自動作成せず、
`MONTH_END_ASSET_BALANCE_NOT_FOUND`
として扱う。

月末資産残高を新規登録する場合は、
BAL-002 月末資産残高登録APIを使用する。

また、資産口座の
`balance_recording_unit` が
口座単位である場合のみ更新を許可する。

商品単位で残高を記録する資産口座については、
本APIでは更新せず、
商品別月末評価額更新APIである
VAL-003を使用する。

Laravel実装では、
HTTPリクエストの受付から、
パスパラメータおよび入力値の検証、
利用者境界の確認、
月末資産状況の取得、
確定状態の確認、
資産口座の取得、
対象年月時点の利用可否判定、
残高記録単位の判定、
更新対象となる月末資産残高の取得、
月末資産残高の更新、
APIレスポンスの生成までを
単一のクラスへ集約せず、
各責務を分離する。

概念的な処理構成は、
以下とする。

```text
Route
    ↓
Middleware
    ↓
Form Request
    ↓
Action
    ↓
Input DTO
    ↓
UseCase
    ├─ Query
    │   ├─ 月末資産状況取得
    │   ├─ 資産口座取得
    │   ├─ 対象年月時点の利用可否確認
    │   └─ 月末資産残高取得
    │
    ├─ 業務ルール判定
    │   ├─ 月末資産状況の確定状態確認
    │   ├─ 対象年月時点の利用可否判定
    │   └─ 残高記録単位判定
    │
    └─ Repository
        └─ 月末資産残高更新
    ↓
Responder
    ↓
API Resource
    ↓
JSON Response
```

指定された月末資産状況は、
必ず操作対象利用者の境界を含めて取得する。

月末資産状況が存在しない場合と、
他の利用者に属する場合は区別せず、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

同様に、指定された資産口座についても、
必ず操作対象利用者の境界を含めて取得する。

資産口座が存在しない場合と、
他の利用者に属する場合は区別せず、
`ASSET_ACCOUNT_NOT_FOUND`
として扱う。

月末資産状況が確定済みの場合は、
月末資産残高を更新せず、
`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`
として扱う。

本API内で、
月末資産状況を自動的に確定解除してはならない。

また、月末資産状況の
`target_year_month` を基準として、
資産口座が対象年月時点で
月末資産管理対象であることを確認する。

対象年月時点で利用できない場合は、
`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`
として扱う。

現在の資産口座状態のみを使用して、
過去月の更新可否を判断してはならない。

さらに、資産口座の
`balance_recording_unit` が
口座単位ではない場合は、
`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`
として扱う。

月末資産残高の更新では、
`balance`のみを変更対象とする。

以下の項目は変更しない。

* `month_end_asset_snapshot_id`
* `asset_account_id`
* `created_at`

`updated_at`は、
Laravelによって更新する。

更新前と更新後の`balance`が
同一である場合も、
エラーとはせず正常処理として扱う。

また、`balance = 0`は
有効な更新値として扱う。

未入力と0円を混同しないよう、
truthy / falsyによる判定は使用せず、
Form RequestおよびDTOによって
入力値を明示的に検証・保持する。

業務条件の最終確認から
月末資産残高の更新までは、
1つのデータベーストランザクション内で実行する。

処理途中で例外が発生した場合は、
更新前の状態へロールバックする。

なお、本APIでは
`updateOrCreate`やupsertを使用しない。

更新対象が存在しない場合に
新規登録へ切り替わる実装を避けることで、

```text
BAL-002
    → 月末資産残高の新規登録

BAL-003
    → 既存の月末資産残高の更新
```

というAPIごとの責務を明確に分離する。

Phase1では、
月末資産残高更新専用の楽観ロックや
明示的な行ロックは必須としない。

複数の更新要求が同時に実行された場合は、
最終的にデータベースへ反映された値を
現在値として扱う。

将来的に更新競合の検知が必要になった場合は、
楽観ロック、`version`カラム、
`updated_at`を利用した競合検知、
`If-Match`などの導入を検討する。

本APIは既存の月末資産残高の更新のみを責務とし、
以下の処理は行わない。

* 月末資産残高の新規登録
* 月末資産状況の確定
* 月末資産状況の確定解除
* 商品別月末評価額の更新
* 資産口座の更新
* 利用可能資産設定の更新
* 目的達成判定の実行

これらの処理は、
それぞれ対応するAPIおよびユースケースへ委譲し、
BAL-003の責務へ含めない。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
月末資産状況ID、
資産口座ID、
更新後の月末資産残高および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの入力値を受け取り、
月末資産残高更新UseCaseを呼び出す。

UseCaseから受け取った更新結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- `snapshotId`の形式検証
- `assetAccountId`の形式検証
- リクエストボディの単項目バリデーション
- 利用者境界の判定
- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 資産口座の存在確認
- 対象年月時点の利用可否判定
- 残高記録単位の判定
- 月末資産残高の存在確認
- 月末資産残高の更新
- トランザクション制御
- レスポンス生成処理

---

### 2.2 UseCase

月末資産残高更新の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 月末資産状況IDを受け取る
- 資産口座IDを受け取る
- 更新後の月末資産残高を受け取る
- 月末資産状況を取得する
- 月末資産状況が未確定であることを確認する
- 資産口座を取得する
- 対象年月時点で資産口座が月末資産管理対象であることを確認する
- 資産口座の残高記録単位が口座単位であることを確認する
- 更新対象となる月末資産残高を取得する
- 月末資産残高が存在することを確認する
- 月末資産残高を更新する
- 更新結果を返却する

指定された月末資産状況が存在しない場合、
または操作対象利用者に属していない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

指定された資産口座が存在しない場合、
または操作対象利用者に属していない場合は、
`ASSET_ACCOUNT_NOT_FOUND`
として扱う。

更新対象となる月末資産残高が
存在しない場合は、
`MONTH_END_ASSET_BALANCE_NOT_FOUND`
として扱う。

---

### 2.3 Form Request / DTO

リクエストボディの
形式および単項目バリデーションを担当する。

検証対象は、
`balance`とする。

Form Requestでは、
以下を検証する。

- 必須であること
- `null`ではないこと
- integerであること
- 0以上であること
- データベースで保持可能な範囲であること

`0`は、
有効な更新値として扱う。

業務状態に依存する以下の検証は、
Form Requestでは行わない。

- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 資産口座の存在確認
- 対象年月時点の利用可否確認
- 残高記録単位確認
- 月末資産残高の存在確認

検証済みの入力値は、
入力用DTOへ変換して
UseCaseへ渡す。

例：

```php
final readonly class UpdateMonthEndAssetBalanceInput
{
    public function __construct(
        public int $balance,
    ) {
    }
}
```

---

### 2.4 パスパラメータ検証

`snapshotId`および`assetAccountId`の形式は、API共通方針に従って検証する。

それぞれ、以下を確認する。

- 指定されていること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

形式が不正な場合は、`VALIDATION_ERROR`として扱う。

存在確認および利用者境界確認は、UseCaseおよびQueryで行う。

---

### 2.5 Query

月末資産残高更新に必要なデータ取得を担当する。

主な取得対象は、以下とする。

- `month_end_asset_snapshots`
- `asset_accounts`
- `asset_account_available_settings`
- `month_end_asset_balances`

本APIでは、原則として以下を参照しない。

- `holding_assets`
- `month_end_holding_values`
- `assessment_histories`

---

### 2.6 月末資産状況取得

更新対象となる月末資産残高が属する月末資産状況は、必ず利用者境界を含めて取得する。

```php
$snapshot = MonthEndAssetSnapshot::query()
    ->where('id', $snapshotId)
    ->where('user_id', $userId)
    ->first([
        'id',
        'user_id',
        'target_year_month',
        'confirmed',
    ]);
```

以下のように、月末資産状況IDだけで取得してはならない。

```php
MonthEndAssetSnapshot::find($snapshotId);
```

取得できなかった場合は、以下を区別せず`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`として扱う。

- 月末資産状況が存在しない
- 他の利用者に属している

---

### 2.7 確定状態確認

取得した月末資産状況が未確定であることを確認する。

```php
if ($snapshot->confirmed) {
    throw new
        MonthEndAssetSnapshotConfirmedException();
}
```

確定済みの場合は、`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`として扱う。

本API内で自動的に確定解除してはならない。

---

### 2.8 資産口座取得

更新対象となる資産口座は、必ず利用者境界を含めて取得する。

```php
$assetAccount = AssetAccount::query()
    ->where('id', $assetAccountId)
    ->where('user_id', $userId)
    ->first();
```

以下のように、資産口座IDだけで取得してはならない。

```php
AssetAccount::find($assetAccountId);
```

取得できなかった場合は、以下を区別せず`ASSET_ACCOUNT_NOT_FOUND`として扱う。

- 資産口座が存在しない
- 他の利用者に属している

---

### 2.9 対象年月時点の利用可否判定

月末資産状況の`target_year_month`を使用して、資産口座が対象年月時点で月末資産管理対象であることを確認する。

判定には、`asset_account_available_settings`を使用する。

概念的には、以下を判定する。

```text
assetAccountId
+
snapshot.target_year_month
    ↓
対象年月時点で利用可能か
```

対象年月時点で月末資産管理対象ではない場合は、`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`として扱う。

現在の資産口座の状態だけで過去月の更新可否を判断してはならない。

---

### 2.10 残高記録単位確認

資産口座の`balance_recording_unit`が口座単位であることを確認する。

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::ACCOUNT
) {
    throw new
        AssetAccountBalanceRecordingUnitMismatchException();
}
```

条件を満たさない場合は、`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`として扱う。

商品単位の資産口座については、VAL-003 商品別月末評価額更新APIを使用する。

---

### 2.11 月末資産残高取得

更新対象となる月末資産残高は、以下の組み合わせで取得する。

```php
$balance = MonthEndAssetBalance::query()
    ->where(
        'month_end_asset_snapshot_id',
        $snapshot->id,
    )
    ->where(
        'asset_account_id',
        $assetAccount->id,
    )
    ->first();
```

取得できなかった場合は、`MONTH_END_ASSET_BALANCE_NOT_FOUND`として扱う。

本APIでは、存在しない場合に新しい月末資産残高を自動作成しない。

---

### 2.12 Repository

月末資産残高の更新を担当する。

更新対象は、`balance`のみとする。

```php
$balance->balance = $input->balance;
$balance->save();

return $balance;
```

以下の項目は、本APIでは変更しない。

- `month_end_asset_snapshot_id`
- `asset_account_id`
- `created_at`

`updated_at`は、Laravelによって更新する。

Repositoryでは、以下の処理は行わない。

- 月末資産残高の新規登録
- 月末資産状況の確定
- 月末資産状況の確定解除
- 商品別月末評価額の更新
- 目的達成判定の実行

---

### 2.13 トランザクション

業務条件の最終確認から月末資産残高更新までを、1つのデータベーストランザクション内で実行する。

実装例：

```php
$balance = DB::transaction(
    function () use (
        $userId,
        $snapshotId,
        $assetAccountId,
        $input,
    ): MonthEndAssetBalance {
        $snapshot =
            $this->snapshotQuery
                ->findByUserAndId(
                    $userId,
                    $snapshotId,
                );

        if ($snapshot === null) {
            throw new
                MonthEndAssetSnapshotNotFoundException();
        }

        if ($snapshot->confirmed) {
            throw new
                MonthEndAssetSnapshotConfirmedException();
        }

        $assetAccount =
            $this->assetAccountQuery
                ->findByUserAndId(
                    $userId,
                    $assetAccountId,
                );

        if ($assetAccount === null) {
            throw new
                AssetAccountNotFoundException();
        }

        $this->availabilityValidator->validate(
            $assetAccount,
            $snapshot->target_year_month,
        );

        $this->recordingUnitValidator->validate(
            $assetAccount,
        );

        $balance =
            $this->balanceQuery
                ->findBySnapshotAndAssetAccount(
                    $snapshot->id,
                    $assetAccount->id,
                );

        if ($balance === null) {
            throw new
                MonthEndAssetBalanceNotFoundException();
        }

        return $this->repository->updateBalance(
            $balance,
            $input->balance,
        );
    },
);
```

処理途中で例外が発生した場合は、更新前の状態へロールバックする。

---

### 2.14 同一値更新の扱い

更新前と更新後の`balance`が同一であっても、エラーとしない。

例えば、

```text
現在値
balance = 1200000

更新値
balance = 1200000
```

の場合も、正常処理として扱う。

UseCaseで差分が存在することを必須条件にはしない。

Phase1では、同一値の場合に更新SQLを省略する最適化は必須としない。

---

### 2.15 0円の扱い

`balance = 0`は、有効な更新値として扱う。

以下のようなtruthy / falsyによる判定は行わない。

```php
if (! $input->balance) {
    // 0円まで未入力扱いになるため使用しない
}
```

更新値は、明示的にintegerとして扱う。

---

### 2.16 Repositoryでupsertしない理由

本APIは、既存の月末資産残高を更新する責務のみを持つ。

そのため、以下のような処理は使用しない。

```php
MonthEndAssetBalance::updateOrCreate(
    [
        'month_end_asset_snapshot_id'
            => $snapshot->id,
        'asset_account_id'
            => $assetAccount->id,
    ],
    [
        'balance'
            => $input->balance,
    ],
);
```

この実装では、更新対象が存在しない場合に新規登録されてしまう。

登録と更新の責務を分離するため、存在しない場合は`MONTH_END_ASSET_BALANCE_NOT_FOUND`を返却する。

新規登録は、BAL-002で行う。

---

### 2.17 Responder

UseCaseから受け取った更新後の月末資産残高を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`200 OK`とともに`data`オブジェクトとして返却する。

Responderは、以下の処理を行わない。

- データベース検索
- 利用者境界の判定
- 確定状態の判定
- 対象年月時点の利用可否判定
- 残高記録単位の判定
- 月末資産残高の存在確認
- 月末資産残高の更新

---

### 2.18 API Resource

データベースカラムを直接返却せず、API Resourceを利用してAPIレスポンス形式へ変換する。

変換例：

```php
return [
    'assetAccountId'
        => (string) $this->asset_account_id,
    'balance'
        => (int) $this->balance,
];
```

`balance = 0`の場合も、そのまま整数の`0`を返却する。

以下の項目は、レスポンスへ含めない。

- 月末資産残高ID
- `snapshotId`
- `userId`
- `targetYearMonth`
- `confirmed`
- 資産口座名
- 更新前の`balance`
- `createdAt`
- `updatedAt`

---

### 2.19 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 2.20 Eloquentモデル

`MonthEndAssetBalance`モデルは、`month_end_asset_balances`テーブルへ対応する。

主に以下の属性を使用する。

```text
id
month_end_asset_snapshot_id
asset_account_id
balance
```

`balance`は、日本円の整数値として扱う。

必要に応じて、integer castを設定する。

```php
protected function casts(): array
{
    return [
        'balance' => 'integer',
    ];
}
```

---

### 2.21 排他制御

Phase1では、月末資産残高更新専用の楽観ロック用バージョン番号は持たない。

また、通常の更新処理では明示的な行ロックを必須としない。

複数の更新要求が同時に実行された場合は、最終的にデータベースへ反映された更新を現在値とする。

将来的に、複数利用者による同時編集や更新競合の検知が必要になった場合は、以下の導入を検討する。

- 楽観ロック
- `version`カラム
- `updated_at`を利用した競合検知
- `If-Match`

---

### 2.22 業務ルール判定クラス

対象年月時点の利用可否判定や残高記録単位の判定は、BAL-002と共通化してよい。

例えば、以下の責務へ分離する。

```text
AssetAccountAvailabilityValidator
    → 対象年月時点の利用可否判定

AssetAccountBalanceRecordingUnitValidator
    → 口座単位であることの判定
```

登録APIと更新APIで同じ業務ルールを別々に実装しない。

各Validatorは、HTTPレスポンス生成やデータベース更新を行わない。

---

### 2.23 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 入力値不正 | `VALIDATION_ERROR` |
| 月末資産状況不存在 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 利用者境界外の月末資産状況 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 月末資産状況確定済み | `MONTH_END_ASSET_SNAPSHOT_CONFIRMED` |
| 資産口座不存在 | `ASSET_ACCOUNT_NOT_FOUND` |
| 利用者境界外の資産口座 | `ASSET_ACCOUNT_NOT_FOUND` |
| 対象年月時点で利用不可 | `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH` |
| 残高記録単位不一致 | `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH` |
| 月末資産残高不存在 | `MONTH_END_ASSET_BALANCE_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

SQL、スタックトレース、PostgreSQLの内部情報および内部例外メッセージは、APIレスポンスへ含めない。

ログには、調査に必要な範囲で以下を記録する。

- 操作対象利用者ID
- 月末資産状況ID
- 資産口座ID
- 対象年月
- 独自エラーコード
- リクエストID

更新前後の月末資産残高の具体的な金額は、不要にエラーログへ出力しない。

---


LaravelおよびPostgreSQLの
内部例外は、
そのままAPIレスポンスへ公開しない。

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `snapshotId`形式不正 | `VALIDATION_ERROR` |
| 月末資産状況不存在 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 利用者境界外の月末資産状況 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

以下の状態は、
例外へ変換しない。

```text
対象資産口座なし
    → 空配列

対象保有商品なし
    → 空配列

商品別月末評価額未登録
    → value = null

商品別月末評価額0円
    → value = 0

月末資産状況確定済み
    → 通常どおり取得
```

SQL、
スタックトレース、
PostgreSQLの内部情報および
内部例外メッセージは、
APIレスポンスへ含めない。

ログには、
調査に必要な範囲で
以下を記録する。

- 操作対象利用者ID
- 月末資産状況ID
- 対象年月
- 独自エラーコード
- リクエストID

---

## 3. 関連ドキュメント

- [BAL-003 API詳細設計](../../../api/details/asset-balances/bal-003-update.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [月末資産残高 Laravelアーキテクチャ設計](./README.md)
- [BAL-003 テスト設計](../../../tests/asset-balances/bal-003-update.md)
- [月末資産残高 テスト設計](../../../tests/asset-balances/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
