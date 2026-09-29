# VAL-002 商品別月末評価額登録

## 1. 概要

操作対象となる利用者について、
指定した月末資産状況に紐づく
商品別月末評価額を新規登録する。

本APIでは、
残高記録単位が
商品単位である資産口座に属する
保有商品について、
対象年月末時点の評価額を登録する。

商品別月末評価額が
すでに登録されている場合は、
本APIでは上書きしない。

既存の商品別月末評価額を変更する場合は、
VAL-003 商品別月末評価額更新APIを使用する。

口座単位で残高を記録する資産口座については、
本APIでは登録しない。

資産口座単位の月末資産残高は、
BAL-002 月末資産残高登録APIを使用する。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
月末資産状況ID、
登録対象の保有商品ID、
商品別月末評価額および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの入力値を受け取り、
商品別月末評価額登録UseCaseを呼び出す。

UseCaseから受け取った登録結果を、
Responderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- `snapshotId`の形式検証
- リクエストボディの単項目バリデーション
- 利用者境界の判定
- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 保有商品の存在確認
- 所属資産口座の利用者境界確認
- 対象年月時点の資産口座利用可否判定
- 残高記録単位の判定
- 対象年月時点の保有商品状態判定
- 重複登録確認
- 商品別月末評価額の登録
- トランザクション制御
- レスポンス生成処理

---

### 2.2 UseCase

商品別月末評価額登録の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 月末資産状況IDを受け取る
- 保有商品IDを受け取る
- 商品別月末評価額を受け取る
- 月末資産状況を取得する
- 月末資産状況が未確定であることを確認する
- 保有商品および所属資産口座を取得する
- 保有商品が操作対象利用者の資産口座に属していることを確認する
- 資産口座が対象年月時点で月末資産管理対象であることを確認する
- 資産口座の残高記録単位が商品単位であることを確認する
- 保有商品が対象年月時点で評価額記録対象であることを確認する
- 同一月末資産状況・保有商品の商品別月末評価額が未登録であることを確認する
- 商品別月末評価額を登録する
- 登録結果を返却する

指定された月末資産状況が存在しない場合、
または操作対象利用者に属していない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

指定された保有商品が存在しない場合、
または他の利用者に属する資産口座の
保有商品である場合は、
`HOLDING_ASSET_NOT_FOUND`
として扱う。

---

### 2.3 Form Request / DTO

リクエストボディの
形式および単項目バリデーションを担当する。

検証対象は、
以下とする。

- `holdingAssetId`
- `value`

#### holdingAssetId

以下を検証する。

- 必須であること
- `null`ではないこと
- 文字列であること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

#### value

以下を検証する。

- 必須であること
- `null`ではないこと
- integerであること
- 0以上であること
- 金額カラムで保持可能な範囲であること

業務状態に依存する以下の検証は、
Form Requestでは行わない。

- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 保有商品の存在確認
- 保有商品の利用者境界確認
- 対象年月時点の資産口座利用可否確認
- 残高記録単位確認
- 対象年月時点の保有商品状態確認
- 重複登録確認

検証済みの入力値は、
入力用DTOへ変換して
UseCaseへ渡す。

例：

```php
final readonly class CreateMonthEndHoldingValueInput
{
    public function __construct(
        public string $holdingAssetId,
        public int $value,
    ) {
    }
}
```

---

### 2.4 Query

商品別月末評価額登録に必要な
データ取得を担当する。

主な取得対象は、
以下とする。

- `month_end_asset_snapshots`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_holding_values`

本APIでは、
原則として以下を参照しない。

- `month_end_asset_balances`
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

本API内で
自動的に確定解除してはならない。

---

### 2.7 保有商品取得

登録対象となる保有商品は、
所属する資産口座とともに取得し、
利用者境界を確認する。

概念例：

```php
$holdingAsset = HoldingAsset::query()
    ->join(
        'asset_accounts',
        'asset_accounts.id',
        '=',
        'holding_assets.asset_account_id',
    )
    ->where(
        'holding_assets.id',
        $input->holdingAssetId,
    )
    ->where(
        'asset_accounts.user_id',
        $userId,
    )
    ->first([
        'holding_assets.id',
        'holding_assets.asset_account_id',
        'asset_accounts.balance_recording_unit',
    ]);
```

以下のように、
保有商品IDだけで
取得してはならない。

```php
HoldingAsset::find(
    $input->holdingAssetId,
);
```

取得できなかった場合は、
以下を区別せず
`HOLDING_ASSET_NOT_FOUND`
として扱う。

- 保有商品が存在しない
- 他の利用者に属する資産口座の保有商品である

---

### 2.8 対象年月時点の資産口座利用可否判定

月末資産状況の
`target_year_month`を使用して、
保有商品が属する資産口座が
対象年月時点で
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
対象年月時点で月末資産管理対象か
```

対象年月時点で
月末資産管理対象ではない場合は、

`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`

として扱う。

現在の資産口座の状態だけを使用して、
過去月への登録可否を
判断してはならない。

---

### 2.9 残高記録単位確認

保有商品が属する資産口座の
`balance_recording_unit`が
商品単位であることを確認する。

```php
if (
    $assetAccount->balance_recording_unit
    !== BalanceRecordingUnit::HOLDING
) {
    throw new
        AssetAccountBalanceRecordingUnitMismatchException();
}
```

実際のEnum名・定数名は、
共通定義に従う。

条件を満たさない場合は、

`ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`

として扱う。

口座単位の資産口座については、
BAL-002 月末資産残高登録APIを使用する。

---

### 2.10 対象年月時点の保有商品判定

指定された保有商品が、
月末資産状況の
`target_year_month`時点で
評価額記録対象であることを確認する。

概念的には、

```text
holdingAssetId
+
snapshot.target_year_month
    ↓
対象年月時点で評価額記録対象か
```

を判定する。

対象年月時点で
評価額記録対象ではない場合は、

`HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH`

として扱う。

現在の有効・無効状態だけを使用して、
過去月への登録可否を
判断してはならない。

---

### 2.11 重複確認

同一の月末資産状況・保有商品について、
商品別月末評価額が
すでに存在しないことを確認する。

```php
$exists = MonthEndHoldingValue::query()
    ->where(
        'month_end_asset_snapshot_id',
        $snapshot->id,
    )
    ->where(
        'holding_asset_id',
        $holdingAsset->id,
    )
    ->exists();
```

存在する場合は、

`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`

として扱う。

アプリケーション側の
重複確認だけでは、
同時実行時の重複を
完全には防止できない。

そのため、
データベースのUNIQUE制約も使用する。

---

### 2.12 Repository

商品別月末評価額の
新規登録を担当する。

登録する値は、
以下とする。

```text
month_end_asset_snapshot_id
holding_asset_id
value
```

登録例：

```php
return MonthEndHoldingValue::create([
    'month_end_asset_snapshot_id'
        => $snapshot->id,
    'holding_asset_id'
        => $holdingAsset->id,
    'value'
        => $input->value,
]);
```

`id`、
`created_at`および
`updated_at`は、
Laravelおよび
データベース側で設定する。

Repositoryでは、
以下の処理を行わない。

- 月末資産状況の確定
- 月末資産状況の確定解除
- 月末資産残高の登録・更新
- 他の保有商品の評価額登録・更新
- 目的達成判定の実行

---

### 2.13 トランザクション

業務条件の最終確認から
商品別月末評価額登録までを、
1つのデータベーストランザクション内で実行する。

実装例：

```php
$holdingValue = DB::transaction(
    function () use (
        $userId,
        $snapshotId,
        $input,
    ): MonthEndHoldingValue {
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

        $holdingAsset =
            $this->holdingAssetQuery
                ->findByUserAndId(
                    $userId,
                    $input->holdingAssetId,
                );

        if ($holdingAsset === null) {
            throw new
                HoldingAssetNotFoundException();
        }

        $this->assetAccountAvailabilityValidator
            ->validate(
                $holdingAsset->assetAccount,
                $snapshot->target_year_month,
            );

        $this->recordingUnitValidator
            ->validateForHoldingValue(
                $holdingAsset->assetAccount,
            );

        $this->holdingAssetAvailabilityValidator
            ->validate(
                $holdingAsset,
                $snapshot->target_year_month,
            );

        if (
            $this->holdingValueQuery
                ->existsBySnapshotAndHoldingAsset(
                    $snapshot->id,
                    $holdingAsset->id,
                )
        ) {
            throw new
                MonthEndHoldingValueAlreadyExistsException();
        }

        return $this->repository->create(
            $snapshot,
            $holdingAsset,
            $input->value,
        );
    },
);
```

処理途中で例外が発生した場合は、
商品別月末評価額を登録しない。

---

### 2.14 UNIQUE制約

以下の組み合わせに、
UNIQUE制約を設定する。

```text
month_end_asset_snapshot_id
+
holding_asset_id
```

複数リクエストが
同時に重複確認を通過しても、
データベース側で
複数レコードの作成を防止する。

UNIQUE制約違反が発生した場合は、
PostgreSQLの例外をそのまま返却せず、

`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`

へ変換する。

アプリケーション側の事前確認と
データベース制約の
両方を使用する。

---

### 2.15 Mass Assignment

クライアントから受け取った値を
そのままEloquentモデルへ渡してはならない。

以下のような実装は避ける。

```php
MonthEndHoldingValue::create(
    $request->all(),
);
```

登録値は、
検証済みの情報から
明示的に組み立てる。

```text
month_end_asset_snapshot_id
    → 利用者境界確認済みのsnapshot.id

holding_asset_id
    → 利用者境界確認済みのholdingAsset.id

value
    → Form Request / DTOの検証済み値
```

これにより、
クライアントから
意図しない外部キーや
サーバー管理項目を
指定されることを防止する。

---

### 2.16 0円の扱い

`value = 0`は、
有効な登録値として扱う。

以下のような
truthy / falsyによる判定は行わない。

```php
if (! $input->value) {
    // 0円も未入力扱いになるため使用しない
}
```

Form Requestでは、
`required`、
integer、
最小値0などを使用し、
0円を正常値として許容する。

---

### 2.17 業務ルール判定クラス

BAL系APIやVAL系APIで
共通して使用する業務ルールは、
必要に応じて
判定クラスへ分離する。

例えば、
以下の責務へ分離できる。

```text
AssetAccountAvailabilityValidator
    → 対象年月時点の資産口座利用可否判定

AssetAccountBalanceRecordingUnitValidator
    → 残高記録単位の判定

HoldingAssetAvailabilityValidator
    → 対象年月時点の保有商品判定
```

登録APIと更新APIで
同じ業務ルールを
別々に実装しない。

各Validatorは、
HTTPレスポンス生成や
データベース登録を行わない。

---

### 2.18 Responder

UseCaseから受け取った
登録後の商品別月末評価額を、
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
- 対象年月時点の資産口座利用可否判定
- 残高記録単位の判定
- 対象年月時点の保有商品判定
- 重複登録判定
- 商品別月末評価額の登録

---

### 2.19 API Resource

データベースカラムを直接返却せず、
API Resourceを利用して
APIレスポンス形式へ変換する。

変換例：

```php
return [
    'holdingAssetId'
        => (string) $this->holding_asset_id,
    'value'
        => (int) $this->value,
];
```

`value = 0`の場合も、
そのまま整数の`0`を返却する。

以下の項目は、
レスポンスへ含めない。

- 商品別月末評価額ID
- `snapshotId`
- `userId`
- `assetAccountId`
- `targetYearMonth`
- `confirmed`
- 保有商品名
- 資産口座名
- `createdAt`
- `updatedAt`

---

### 2.20 Middleware

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

### 2.21 Eloquentモデル

`MonthEndHoldingValue`モデルは、
`month_end_holding_values`
テーブルへ対応する。

主に以下の属性を使用する。

```text
id
month_end_asset_snapshot_id
holding_asset_id
value
```

`value`は、
日本円の整数値として扱う。

必要に応じて、
integer castを設定する。

```php
protected function casts(): array
{
    return [
        'value' => 'integer',
    ];
}
```

また、
必要に応じて以下のRelationを定義する。

```php
public function snapshot(): BelongsTo
{
    return $this->belongsTo(
        MonthEndAssetSnapshot::class,
        'month_end_asset_snapshot_id',
    );
}

public function holdingAsset(): BelongsTo
{
    return $this->belongsTo(
        HoldingAsset::class,
        'holding_asset_id',
    );
}
```

---

### 2.22 upsertを使用しない

本APIは、
未登録の商品別月末評価額を
新規登録する責務のみを持つ。

そのため、
以下のような
`updateOrCreate`は使用しない。

```php
MonthEndHoldingValue::updateOrCreate(
    [
        'month_end_asset_snapshot_id'
            => $snapshot->id,
        'holding_asset_id'
            => $holdingAsset->id,
    ],
    [
        'value'
            => $input->value,
    ],
);
```

この実装では、
登録済みの商品別月末評価額を
VAL-002で更新できてしまう。

登録と更新の責務を分離するため、
登録済みの場合は

`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`

として扱う。

既存値の変更は、
VAL-003 商品別月末評価額更新APIで行う。

---

### 2.23 排他・競合への対応

アプリケーション側で
重複確認を行っても、
以下のような並行実行が発生し得る。

```text
リクエストA
重複なし確認

リクエストB
重複なし確認

リクエストA
INSERT

リクエストB
INSERT
```

そのため、
重複登録防止は
UNIQUE制約を最終防衛線とする。

Phase1では、
登録処理専用の
楽観ロックや
`Idempotency-Key`は使用しない。

---

### 2.24 例外変換

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
| 入力値不正 | `VALIDATION_ERROR` |
| 月末資産状況不存在 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 利用者境界外の月末資産状況 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 月末資産状況確定済み | `MONTH_END_ASSET_SNAPSHOT_CONFIRMED` |
| 保有商品不存在 | `HOLDING_ASSET_NOT_FOUND` |
| 利用者境界外の保有商品 | `HOLDING_ASSET_NOT_FOUND` |
| 資産口座が対象年月時点で利用不可 | `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH` |
| 残高記録単位不一致 | `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH` |
| 保有商品が対象年月時点で対象外 | `HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH` |
| 商品別月末評価額重複 | `MONTH_END_HOLDING_VALUE_ALREADY_EXISTS` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

UNIQUE制約違反については、
対象となる制約を判別し、
`MONTH_END_HOLDING_VALUE_ALREADY_EXISTS`
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
- 保有商品ID
- 対象年月
- 独自エラーコード
- リクエストID

商品別月末評価額の具体的な金額は、
不要にエラーログへ出力しない。

---

## 3. 関連ドキュメント

- [VAL-002 API詳細設計](../../../api/details/holding-values/val-002-create.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [商品別月末評価額 Laravelアーキテクチャ設計](./README.md)
- [VAL-002 テスト設計](../../../tests/holding-values/val-002-create.md)
- [商品別月末評価額 テスト設計](../../../tests/holding-values/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
