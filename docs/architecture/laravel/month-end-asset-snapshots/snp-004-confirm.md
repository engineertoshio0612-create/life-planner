# SNP-004 月末資産状況確定

## 概要

操作対象となる利用者について、
指定された未確定の月末資産状況を
確定するAPI。

月末資産状況を確定することで、
対象年月の資産状況を
正式な月末資産状況として扱える状態にする。

確定対象は、
操作対象利用者に属する
未確定の月末資産状況とする。

指定された月末資産状況が存在しない場合、
または他利用者に属している場合は、
外部レスポンスでは区別せず、

`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`

として扱う。

すでに確定済みの場合は、

`MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED`

として扱い、
再度確定することはできない。

確定時には、
主に以下の条件を検証する。

- 対象の月末資産状況が未確定であること
- 確定順序を満たしていること
- 対象年月時点で必要な月末資産残高が登録されていること
- 対象年月時点で必要な商品別月末評価額が登録されていること

確定順序では、
確定対象より前の対象年月に
未確定の月末資産状況が存在しないことを確認する。

月末資産状況そのものが存在しない年月は、
未確定月として扱わない。

確定条件の判定では、
現在の利用状態だけではなく、
対象年月時点で
月末資産管理対象となる
資産口座および保有商品を基準とする。

資産口座の残高記録単位に応じて、
確認対象を切り替える。

```text
対象年月時点で月末資産管理対象
    ↓
残高記録単位を確認
    ├─ 口座単位
    │    → 月末資産残高を確認
    │
    └─ 商品単位
         → 保有商品を取得
              ↓
           商品別月末評価額を確認
```

必要な月末資産残高が不足している場合は、

`MONTH_END_ASSET_BALANCE_INCOMPLETE`

として扱う。

必要な商品別月末評価額が不足している場合は、

`MONTH_END_HOLDING_VALUE_INCOMPLETE`

として扱う。

金額または評価額が `0円` のデータは、
未登録とはせず、
有効な登録済みデータとして扱う。

確定対象より前に
未確定の月末資産状況が存在する場合は、

`MONTH_END_ASSET_SNAPSHOT_CONFIRM_ORDER_INVALID`

として扱う。

Laravelでは、
以下の責務を分離して実装する。

- Action
  - HTTPリクエストの受付
  - 月末資産状況IDの取得
  - 利用者コンテキストの取得
  - UseCaseの呼び出し
- UseCase
  - 確定処理全体の制御
- Query
  - 確定対象および確定条件判定に必要なデータ取得
  - 利用者境界の保証
- Validator
  - 確定順序の判定
  - 月末資産残高の登録完了判定
  - 商品別月末評価額の登録完了判定
- Repository
  - 月末資産状況の確定状態更新
- Responder
  - HTTPレスポンスへの変換
- API Resource
  - APIレスポンス形式への変換

確定条件が複数存在するため、
すべての業務ルールをUseCaseへ直接記述せず、
用途ごとのValidatorへ分離する。

確定条件の最終確認から
確定状態の更新までは、
1つのデータベーストランザクション内で実行する。

また、
確定対象となる月末資産状況には
`lockForUpdate()` を使用し、
行ロック取得後に
確定状態および各確定条件を再確認する。

これにより、
同一の月末資産状況に対する
複数の確定要求が同時に実行された場合でも、
後続処理が最新状態を確認した上で
確定可否を判定できるようにする。

確定処理では、
`confirmed` のみを `true` へ更新し、
以下のデータは変更しない。

- 対象年月
- 利用者ID
- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品
- 利用可能資産設定

正常終了時は、
`200 OK` とともに、
確定後の月末資産状況を
`data` オブジェクトとして返却する。

レスポンスでは、
以下の情報を返却する。

- 月末資産状況ID
- 対象年月
- 確定状態

正常終了時の
`confirmed` は必ず `true` となる。

LaravelやPostgreSQLの内部例外、
SQL、
スタックトレースなどの内部情報は
APIレスポンスへ公開せず、
API共通方針に従って
独自エラーコードへ変換する。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
月末資産状況IDおよび
利用者コンテキストを取得する。

月末資産状況確定UseCaseを呼び出し、
処理結果をResponderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 月末資産状況IDの形式検証
- 利用者境界の判定
- 月末資産状況の取得
- 確定済み状態の判定
- 確定順序の判定
- 必要な月末資産残高の確認
- 必要な商品別月末評価額の確認
- 確定処理
- トランザクション制御
- 排他制御
- レスポンス生成処理

本APIは、
リクエストボディを使用しない。

---

### 2.2 UseCase

月末資産状況確定の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 月末資産状況IDを受け取る
- 確定対象の月末資産状況を取得する
- 未確定であることを確認する
- 確定順序を確認する
- 対象年月時点で必要となる資産口座を取得する
- 資産口座の残高記録単位を判定する
- 必要な月末資産残高が登録されていることを確認する
- 必要な商品別月末評価額が登録されていることを確認する
- 月末資産状況を確定する
- 確定後の月末資産状況を返却する

指定された月末資産状況が存在しない場合、
または操作対象利用者に属していない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

すでに確定済みの場合は、
`MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED`
として扱う。

確定条件を満たさない場合は、
対応する業務エラーを発生させる。

---

### 2.3 パスパラメータ検証

`snapshotId`の形式は、
API共通方針に従って検証する。

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

形式が不正な場合は、
`VALIDATION_ERROR`
として扱う。

月末資産状況の存在確認、
利用者境界確認および
確定状態確認は、
UseCaseおよびQueryで実施する。

---

### 2.4 Query

確定処理に必要な
データ取得を担当する。

主な取得対象は、
以下とする。

- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `holding_assets`
- `asset_account_available_settings`

確定対象となる月末資産状況は、
必ず利用者境界を含めて取得する。

```php
$snapshot = MonthEndAssetSnapshot::query()
    ->where('id', $snapshotId)
    ->where('user_id', $userId)
    ->first();
```

以下のように、月末資産状況IDだけで取得してはならない。

```php
MonthEndAssetSnapshot::find($snapshotId);
```

確定対象より前の対象年月に未確定の月末資産状況が存在するかどうかも、操作対象利用者を条件として取得する。

---

### 2.5 対象年月時点の資産取得

確定条件を判定するため、対象年月時点で月末資産管理の対象となる資産口座および保有商品を取得する。

現在の利用状態だけで対象年月時点の状態を判断しない。

資産口座の利用可能期間など、対象年月時点の状態を判定するための情報がある場合は、その情報を使用する。

例えば、資産口座ごとに以下を判定する。

```text
対象年月時点で月末資産管理対象
    ↓
残高記録単位を確認
    ├─ 口座単位
    │    → month_end_asset_balancesを確認
    │
    └─ 商品単位
         → holding_assetsを取得
              ↓
           month_end_holding_valuesを確認
```

---

### 2.6 月末資産残高確認

残高記録単位が口座単位の資産口座について、対象となる月末資産状況に紐づく月末資産残高が存在することを確認する。

例えば、必要な資産口座IDと登録済み資産口座IDを比較する。

```php
$requiredAssetAccountIds =
    $assetAccounts
        ->where('balance_recording_unit', BalanceRecordingUnit::ACCOUNT)
        ->pluck('id')
        ->sort()
        ->values();

$registeredAssetAccountIds =
    $assetBalances
        ->pluck('asset_account_id')
        ->sort()
        ->values();
```

不足が存在する場合は、`MONTH_END_ASSET_BALANCE_INCOMPLETE`として扱う。

金額が`0円`で登録されている場合は、未登録ではなく有効な月末資産残高として扱う。

---

### 2.7 商品別月末評価額確認

残高記録単位が商品単位の資産口座について、対象年月時点で必要な保有商品の商品別月末評価額がすべて登録されていることを確認する。

必要な保有商品IDと登録済み保有商品IDを比較する。

不足が存在する場合は、`MONTH_END_HOLDING_VALUE_INCOMPLETE`として扱う。

評価額が`0円`で登録されている場合は、未登録ではなく有効な商品別月末評価額として扱う。

---

### 2.8 確定順序確認

確定対象より前の対象年月に未確定の月末資産状況が存在しないことを確認する。

例えば、以下の条件で検索する。

```php
$existsPreviousUnconfirmed =
    MonthEndAssetSnapshot::query()
        ->where('user_id', $userId)
        ->where(
            'target_year_month',
            '<',
            $snapshot->target_year_month,
        )
        ->where('confirmed', false)
        ->exists();
```

存在する場合は、`MONTH_END_ASSET_SNAPSHOT_CONFIRM_ORDER_INVALID`として扱う。

月末資産状況そのものが存在しない年月は、未確定月として扱わない。

---

### 2.9 Repository

月末資産状況の確定状態更新を担当する。

更新対象は、以下とする。

- `confirmed`
- `updated_at`

確定時は、`confirmed`を`true`へ更新する。

```php
$snapshot->confirmed = true;
$snapshot->save();

return $snapshot;
```

以下のデータは更新しない。

- `target_year_month`
- `user_id`
- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品
- 利用可能資産設定

---

### 2.10 トランザクション

確定条件の最終確認から確定状態更新までを、1つのデータベーストランザクション内で実行する。

実装例：

```php
$snapshot = DB::transaction(
    function () use (
        $userId,
        $snapshotId,
    ): MonthEndAssetSnapshot {
        $snapshot =
            $this->snapshotQuery
                ->findByUserAndIdForUpdate(
                    $userId,
                    $snapshotId,
                );

        if ($snapshot === null) {
            throw new
                MonthEndAssetSnapshotNotFoundException();
        }

        if ($snapshot->confirmed) {
            throw new
                MonthEndAssetSnapshotAlreadyConfirmedException();
        }

        $this->confirmOrderValidator->validate(
            $userId,
            $snapshot,
        );

        $this->assetBalanceValidator->validate(
            $userId,
            $snapshot,
        );

        $this->holdingValueValidator->validate(
            $userId,
            $snapshot,
        );

        return $this->repository->confirm(
            $snapshot,
        );
    },
);
```

確定条件を満たさない場合、または処理途中で例外が発生した場合は、確定状態を更新しない。

---

### 2.11 排他制御

確定対象となる月末資産状況を取得する際は、`lockForUpdate()`を使用する。

```php
return MonthEndAssetSnapshot::query()
    ->where('id', $snapshotId)
    ->where('user_id', $userId)
    ->lockForUpdate()
    ->first();
```

ロック取得後に、以下を再確認する。

- 対象が存在すること
- 未確定であること
- 確定順序を満たしていること
- 必要な月末資産残高が登録されていること
- 必要な商品別月末評価額が登録されていること

これにより、同一月末資産状況に対する複数の確定要求が同時に実行された場合でも、後続処理が最新の確定状態を確認できる。

---

### 2.12 業務ルール判定クラス

確定条件が複数存在するため、UseCaseへすべての判定ロジックを直接記述しない。

例えば、以下の責務へ分離する。

```text
MonthEndAssetSnapshotConfirmOrderValidator
    → 確定順序判定

MonthEndAssetBalanceCompletionValidator
    → 月末資産残高の登録完了判定

MonthEndHoldingValueCompletionValidator
    → 商品別月末評価額の登録完了判定
```

UseCaseは、これらを組み合わせて確定ユースケース全体を制御する。

各Validatorは、HTTPレスポンス生成やデータベース更新を行わない。

---

### 2.13 Responder

UseCaseから受け取った確定後の月末資産状況を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`200 OK`とともに`data`オブジェクトとして返却する。

以下の場合は、共通エラーレスポンス形式へ変換する。

- 対象が存在しない
- すでに確定済み
- 月末資産残高が不足している
- 商品別月末評価額が不足している
- 確定順序を満たしていない
- 想定外の例外が発生した

Responderは、以下の処理を行わない。

- 確定条件の判定
- 利用者境界の判定
- 排他制御
- データベース更新

---

### 2.14 API Resource

データベースカラムを直接返却せず、API Resourceを利用してAPIレスポンス形式へ変換する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'targetYearMonth' => $this->target_year_month,
    'confirmed' => (bool) $this->confirmed,
];
```

正常終了時の`confirmed`は必ず`true`となる。

以下の項目は、レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`
- 月末資産残高の明細
- 商品別月末評価額の明細

---

### 2.15 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 2.16 Eloquentモデル

`MonthEndAssetSnapshot`モデルでは、`confirmed`をbooleanとして扱う。

```php
protected function casts(): array
{
    return [
        'confirmed' => 'boolean',
    ];
}
```

`target_year_month`は、`YYYY-MM`形式の年月を表す業務値として扱う。

通常の日付を表すCarbonへのdate castは設定しない。

---

### 2.17 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 月末資産状況ID形式不正 | `VALIDATION_ERROR` |
| 月末資産状況不存在 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 利用者境界外 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 確定済み | `MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED` |
| 月末資産残高不足 | `MONTH_END_ASSET_BALANCE_INCOMPLETE` |
| 商品別月末評価額不足 | `MONTH_END_HOLDING_VALUE_INCOMPLETE` |
| 確定順序違反 | `MONTH_END_ASSET_SNAPSHOT_CONFIRM_ORDER_INVALID` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

SQL、スタックトレース、PostgreSQLの内部情報および内部例外メッセージは、APIレスポンスへ含めない。

ログには、調査に必要な範囲で以下を記録する。

- 操作対象利用者ID
- 月末資産状況ID
- 対象年月
- 独自エラーコード
- リクエストID

月末資産残高や商品別月末評価額の具体的な金額は、不要にエラーログへ出力しない。

---

## 3. 関連ドキュメント

- [SNP-004 API詳細設計](../../../api/details/month-end-asset-snapshots/snp-004-confirm.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [月末資産状況 Laravelアーキテクチャ設計](./README.md)
- [SNP-004 テスト設計](../../../tests/month-end-asset-snapshots/snp-004-confirm.md)
- [月末資産状況 テスト設計](../../../tests/month-end-asset-snapshots/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)