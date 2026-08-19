# BAL-002 月末資産残高登録

## 1. 概要

本ドキュメントでは、
BAL-002 月末資産残高登録APIを
Laravelで実装する際の
アーキテクチャおよび責務分離方針を定義する。

BAL-002では、
操作対象利用者の指定された月末資産状況に対して、
残高記録単位が口座単位である資産口座の
月末資産残高を新規登録する。

本APIでは、
指定された月末資産状況の対象年月を基準として、
登録対象となる資産口座が
その対象年月時点で月末資産管理対象であることを確認したうえで、
月末資産残高を登録する。

また、資産口座の
`balance_recording_unit` が
口座単位である場合のみ登録を許可する。

商品単位で残高を記録する資産口座については、
本APIでは月末資産残高を登録せず、
商品別月末評価額登録APIである
VAL-002を使用する。

Laravel実装では、
HTTPリクエストの受付から、
入力値の検証、
利用者境界の確認、
月末資産状況の取得、
確定状態の確認、
資産口座の取得、
対象年月時点の利用可否判定、
残高記録単位の判定、
重複登録の確認、
月末資産残高の登録、
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
    │   └─ 重複登録確認
    │
    ├─ 業務ルール判定
    │   ├─ 月末資産状況の確定状態確認
    │   ├─ 対象年月時点の利用可否判定
    │   └─ 残高記録単位判定
    │
    └─ Repository
        └─ 月末資産残高登録
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
登録処理を行わず、
`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`
として扱う。

本API内で月末資産状況を
自動的に確定解除してはならない。

また、対象年月時点で
資産口座が月末資産管理対象ではない場合や、
残高記録単位が口座単位ではない場合についても、
それぞれの業務エラーとして扱い、
月末資産残高を登録しない。

同一の月末資産状況と資産口座の組み合わせについては、
月末資産残高を複数登録してはならない。

そのため、

```text
month_end_asset_snapshot_id
+
asset_account_id
```

の組み合わせに対して、
アプリケーション側で重複確認を行うとともに、
データベースのUNIQUE制約によって
同時実行時の重複登録も防止する。

月末資産残高の業務条件確認から登録までは、
1つのデータベーストランザクション内で実行し、
処理途中で例外が発生した場合は
登録を行わない。

なお、`balance = 0` は
0円として登録された有効な月末資産残高として扱う。

未入力と0円を混同しないよう、
truthy / falsyによる判定は使用せず、
Form RequestおよびDTOによって
入力値を明示的に検証・保持する。

本APIは登録処理のみを責務とし、
以下の処理は行わない。

* 月末資産状況の確定
* 月末資産状況の確定解除
* 既存の月末資産残高の更新
* 商品別月末評価額の登録
* 目的達成判定の実行

これらの処理は、
それぞれ対応するAPIおよびユースケースへ委譲し、
BAL-002の責務へ含めない。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
月末資産状況ID、
登録対象の資産口座ID、
月末資産残高および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの入力値を受け取り、
月末資産残高登録UseCaseを呼び出す。

UseCaseから受け取った登録結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- `snapshotId`の形式検証
- リクエストボディの単項目バリデーション
- 利用者境界の判定
- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 資産口座の存在確認
- 対象年月時点の利用可否判定
- 残高記録単位の判定
- 重複登録確認
- 月末資産残高の登録
- トランザクション制御
- レスポンス生成処理

---

### 2.2 UseCase

月末資産残高登録の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 月末資産状況IDを受け取る
- 登録対象の資産口座IDを受け取る
- 月末資産残高を受け取る
- 月末資産状況を取得する
- 月末資産状況が未確定であることを確認する
- 資産口座を取得する
- 対象年月時点で資産口座が月末資産管理対象であることを確認する
- 資産口座の残高記録単位が口座単位であることを確認する
- 同一月末資産状況・資産口座の月末資産残高が未登録であることを確認する
- 月末資産残高を登録する
- 登録結果を返却する

指定された月末資産状況が存在しない場合、
または操作対象利用者に属していない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

指定された資産口座が存在しない場合、
または操作対象利用者に属していない場合は、
`ASSET_ACCOUNT_NOT_FOUND`
として扱う。

---

### 2.3 Form Request / DTO

リクエストボディの
形式および単項目バリデーションを担当する。

検証対象は、
以下とする。

- `assetAccountId`
- `balance`

Form Requestでは、
以下を検証する。

#### assetAccountId

- 必須であること
- `null`ではないこと
- 文字列であること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

#### balance

- 必須であること
- `null`ではないこと
- integerであること
- 0以上であること
- データベースで保持可能な範囲であること

業務状態に依存する以下の検証は、
Form Requestでは行わない。

- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 資産口座の存在確認
- 対象年月時点の利用可否確認
- 残高記録単位確認
- 重複登録確認

検証済みの入力値は、
入力用DTOへ変換して
UseCaseへ渡す。

例：

```php
final readonly class CreateMonthEndAssetBalanceInput
{
    public function __construct(
        public string $assetAccountId,
        public int $balance,
    ) {
    }
}
```

---

### 2.4 Query

月末資産残高登録に必要な
データ取得を担当する。

主な取得対象は、
以下とする。

- `month_end_asset_snapshots`
- `asset_accounts`
- `asset_account_available_settings`
- `month_end_asset_balances`

本APIでは、
原則として以下を参照しない。

- `holding_assets`
- `month_end_holding_values`
- `assessment_histories`

---

### 2.5 月末資産状況取得

登録対象となる月末資産状況は、
必ず利用者境界を含めて取得する。

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

以下のように、
月末資産状況IDだけで
取得してはならない。

```php
MonthEndAssetSnapshot::find($snapshotId);
```

取得できなかった場合は、
以下を区別せず
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

- 月末資産状況が存在しない
- 他の利用者に属している

---

### 2.6 確定状態確認

取得した月末資産状況が
未確定であることを確認する。

```php
if ($snapshot->confirmed) {
    throw new
        MonthEndAssetSnapshotConfirmedException();
}
```

確定済みの場合は、

`MONTH_END_ASSET_SNAPSHOT_CONFIRMED`

として扱う。

月末資産残高を登録するために、
本API内で自動的に
確定解除してはならない。

---

### 2.7 資産口座取得

登録対象となる資産口座は、
必ず利用者境界を含めて取得する。

```php
$assetAccount = AssetAccount::query()
    ->where(
        'id',
        $input->assetAccountId,
    )
    ->where(
        'user_id',
        $userId,
    )
    ->first();
```

以下のように、
資産口座IDだけで
取得してはならない。

```php
AssetAccount::find(
    $input->assetAccountId,
);
```

取得できなかった場合は、
以下を区別せず
`ASSET_ACCOUNT_NOT_FOUND`
として扱う。

- 資産口座が存在しない
- 他の利用者に属している

---

### 2.8 対象年月時点の利用可否判定

月末資産状況の
`target_year_month`を使用して、
資産口座が対象年月時点で
月末資産管理対象であることを確認する。

判定には、
`asset_account_available_settings`
を使用する。

概念的には、
以下を判定する。

```text
assetAccountId
+
snapshot.target_year_month
    ↓
対象年月時点で利用可能か
```

対象年月時点で
月末資産管理対象ではない場合は、

`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`

として扱う。

現在の資産口座の状態だけで
過去月の登録可否を
判断してはならない。

---

### 2.9 残高記録単位確認

資産口座の
`balance_recording_unit`が
口座単位であることを確認する。

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::ACCOUNT
) {
    throw new
        AssetAccountBalanceRecordingUnitMismatchException();
}
```

条件を満たさない場合は、

`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`

として扱う。

商品単位の資産口座について、
本APIで月末資産残高を
登録してはならない。

---

### 2.10 重複確認

同一の月末資産状況・資産口座について、
月末資産残高が
すでに存在しないことを確認する。

```php
$exists = MonthEndAssetBalance::query()
    ->where(
        'month_end_asset_snapshot_id',
        $snapshot->id,
    )
    ->where(
        'asset_account_id',
        $assetAccount->id,
    )
    ->exists();
```

存在する場合は、

`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`

として扱う。

アプリケーション側の
重複確認だけでは
同時実行時の重複を完全には防止できないため、
データベースのUNIQUE制約も使用する。

---

### 2.11 Repository

月末資産残高の
新規登録を担当する。

登録対象は、
以下とする。

- `month_end_asset_snapshot_id`
- `asset_account_id`
- `balance`

登録例：

```php
return MonthEndAssetBalance::create([
    'month_end_asset_snapshot_id'
        => $snapshot->id,
    'asset_account_id'
        => $assetAccount->id,
    'balance'
        => $input->balance,
]);
```

`id`、
`created_at`および
`updated_at`は、
Laravelおよび
データベース側で設定する。

Repositoryでは、
以下の処理は行わない。

- 月末資産状況の確定
- 月末資産状況の確定解除
- 商品別月末評価額の登録
- 目的達成判定の実行

---

### 2.12 トランザクション

業務条件の最終確認から
月末資産残高登録までを、
1つのデータベーストランザクション内で実行する。

実装例：

```php
$balance = DB::transaction(
    function () use (
        $userId,
        $snapshotId,
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
                    $input->assetAccountId,
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

        if (
            $this->balanceQuery
                ->existsBySnapshotAndAssetAccount(
                    $snapshot->id,
                    $assetAccount->id,
                )
        ) {
            throw new
                MonthEndAssetBalanceAlreadyExistsException();
        }

        return $this->repository->create(
            $snapshot,
            $assetAccount,
            $input->balance,
        );
    },
);
```

処理途中で例外が発生した場合は、
月末資産残高を登録しない。

---

### 2.13 UNIQUE制約

以下の組み合わせに、
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
asset_account_id
```

複数リクエストが
同時に重複確認を通過しても、
データベース側で
複数レコードの作成を防止する。

UNIQUE制約違反が発生した場合は、
PostgreSQLの例外をそのまま返却せず、

`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`

へ変換する。

アプリケーション側の事前確認と
データベース制約の
両方を使用する。

---

### 2.14 Mass Assignment

クライアントから受け取った値を
そのままModelへ渡してはならない。

以下のような実装は避ける。

```php
MonthEndAssetBalance::create(
    $request->all(),
);
```

登録値は、
以下から明示的に組み立てる。

```text
snapshotId
    → 検証済み月末資産状況

assetAccountId
    → 検証済み資産口座

balance
    → Form Request / DTOの検証済み値
```

これにより、
クライアントから
意図しない外部キーや
サーバー管理項目を指定されることを防止する。

---

### 2.15 0円の扱い

`balance = 0`は、
有効な値として扱う。

以下のような
truthy / falsy判定は使用しない。

```php
if (! $input->balance) {
    // 0円も未入力扱いになるため使用しない
}
```

Form Requestでは、
`required`と
整数・最小値の検証を組み合わせ、
0円を正常値として許容する。

---

### 2.16 Responder

UseCaseから受け取った
登録後の月末資産残高を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`201 Created`とともに
`data`オブジェクトとして返却する。

Responderは、
以下の処理を行わない。

- データベース検索
- 利用者境界の判定
- 確定状態の判定
- 対象年月時点の利用可否判定
- 残高記録単位の判定
- 重複登録判定
- 月末資産残高の登録

---

### 2.17 API Resource

データベースカラムを直接返却せず、
API Resourceを利用して
APIレスポンス形式へ変換する。

変換例：

```php
return [
    'assetAccountId'
        => (string) $this->asset_account_id,
    'balance'
        => (int) $this->balance,
];
```

正常終了時は、
`balance`が必ずintegerとなる。

`balance = 0`の場合も
そのまま`0`を返却する。

以下の項目は、
レスポンスへ含めない。

- 月末資産残高ID
- `snapshotId`
- `userId`
- `targetYearMonth`
- `confirmed`
- 資産口座名
- `createdAt`
- `updatedAt`

---

### 2.18 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、
`X-User-Id`を検証し、
操作対象利用者を特定する。

Action以降では、
検証済みの利用者コンテキストを使用する。

---

### 2.19 Eloquentモデル

`MonthEndAssetBalance`モデルは、
`month_end_asset_balances`
テーブルへ対応する。

主に以下の属性を使用する。

```text
id
month_end_asset_snapshot_id
asset_account_id
balance
```

`balance`は、
日本円の整数値として扱う。

必要に応じて、
integer castを設定する。

```php
protected function casts(): array
{
    return [
        'balance' => 'integer',
    ];
}
```

---

### 2.20 業務ルール判定クラス

対象年月時点の利用可否判定や
残高記録単位の判定を、
UseCaseへ直接書き込み続けない。

例えば、
以下の責務へ分離してよい。

```text
AssetAccountAvailabilityValidator
    → 対象年月時点の利用可否判定

AssetAccountBalanceRecordingUnitValidator
    → 口座単位であることの判定
```

各Validatorは、
HTTPレスポンス生成や
データベース登録を行わない。

---

### 2.21 例外変換

LaravelおよびPostgreSQLの内部例外は、
そのままAPIレスポンスへ公開しない。

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
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
| 月末資産残高重複 | `MONTH_END_ASSET_BALANCE_ALREADY_EXISTS` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

UNIQUE制約違反については、
対象となる制約を判別し、
`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`
へ変換する。

SQL、
スタックトレース、
PostgreSQLの制約名および
内部例外メッセージは、
APIレスポンスへ含めない。

ログには、
調査に必要な範囲で
以下を記録する。

- 操作対象利用者ID
- 月末資産状況ID
- 資産口座ID
- 対象年月
- 独自エラーコード
- リクエストID

月末資産残高の具体的な金額は、
不要にエラーログへ出力しない。

---

## 3. 関連ドキュメント

## 3. 関連ドキュメント

* -BAL-002 API詳細設計](../../../api/details/asset-balances/bal-002-create.md)
* -API一覧](../../../api/api-list.md)
* -API共通方針](../../../api/api-common-policy.md)
* -エラーコード一覧](../../../api/error-codes.md)
* -Laravelアーキテクチャ共通設計](../laravel-architecture.md)
* -月末資産残高 Laravelアーキテクチャ設計](./README.md)
* -BAL-002 テスト設計](../../../tests/asset-balances/bal-002-create.md)
* -月末資産残高 テスト設計](../../../tests/asset-balances/README.md)
* -機能要件](../../../requirements/functional-requirements.md)
* -ユビキタス言語集](../../../glossary.md)
* -エンティティ定義](../../../entities.md)
* -テーブル定義書](../../../table-definition.md)
* -ER図](../../../er-diagram-phase1.md)
