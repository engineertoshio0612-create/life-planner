## 1. 概要

操作対象となる利用者について、
指定した月末資産状況に紐づく
既存の商品別月末評価額を更新する。

更新対象は、
残高記録単位が商品単位である資産口座に属し、
対象年月時点で評価額記録対象となる保有商品とする。

更新時には、
対象の月末資産状況が未確定であること、
資産口座が対象年月時点で月末資産管理対象であること、
保有商品が対象年月時点で評価額記録対象であることを確認する。

また、
月末資産状況および保有商品について
操作対象利用者との利用者境界を確認する。

指定した月末資産状況・保有商品に対応する
商品別月末評価額が登録されていない場合は、
本APIでは新規登録しない。

新しい商品別月末評価額を登録する場合は、
VAL-002 商品別月末評価額登録APIを使用する。

更新対象は商品別月末評価額の `value` のみとし、
月末資産状況や保有商品との紐付けは変更しない。

口座単位で残高を記録する資産口座は
本APIの更新対象外とし、
月末資産残高の更新には
BAL-003 月末資産残高更新APIを使用する。

---

## 2. テスト観点

### 2.1 正常系

#### 商品別月末評価額を更新できること

以下の条件を満たす場合、
商品別月末評価額を
正常に更新できること。

- `X-User-Id`が正しく指定されている
- 操作対象利用者が存在する
- `snapshotId`が正しい
- `holdingAssetId`が正しい
- 月末資産状況が操作対象利用者に属している
- 月末資産状況が未確定である
- 保有商品が操作対象利用者に属する資産口座のものである
- 商品別月末評価額が登録済みである
- 資産口座の残高記録単位が商品単位である
- 資産口座が対象年月時点で月末資産管理対象である
- 保有商品が対象年月時点で評価額記録対象である
- `value`が有効な整数値である

期待結果：

```text
200 OK
```

`month_end_holding_values.value`が
指定した値へ更新されること。

---

### 2.2 0円への更新

以下を指定する。

```json
{
  "value": 0
}
```

期待結果：

```text
200 OK
```

`value = 0`として
正常に更新されること。

0円を
未入力または未登録として
扱わないこと。

---

### 2.3 同一値への更新

現在値と
同じ`value`を指定する。

```text
現在値
value = 900000
```

```json
{
  "value": 900000
}
```

期待結果：

```text
200 OK
```

同一値であることを理由に
エラーとならないこと。

複数回同じリクエストを実行しても、
最終的な`value`が
同じ状態になること。

---

### 2.4 利用者コンテキスト未指定

`X-User-Id`を
指定せずにリクエストする。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

商品別月末評価額が
更新されないこと。

---

### 2.5 利用者ID形式不正

不正な`X-User-Id`を
指定する。

例：

```http
X-User-Id: abc
```

期待結果：

```text
400 Bad Request
INVALID_USER_ID
```

商品別月末評価額が
更新されないこと。

---

### 2.6 利用者不存在

存在しない利用者IDを
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

商品別月末評価額が
更新されないこと。

---

### 2.7 論理削除済み利用者

論理削除されている利用者IDを
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

商品別月末評価額が
更新されないこと。

---

### 2.8 snapshotId形式不正

以下のような
不正な`snapshotId`を指定する。

```text
0
-1
abc
1.5
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

商品別月末評価額が
更新されないこと。

---

### 2.9 holdingAssetId形式不正

以下のような
不正な`holdingAssetId`を指定する。

```text
0
-1
abc
1.5
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

商品別月末評価額が
更新されないこと。

---

### 2.10 value未指定

`value`を指定せずに
リクエストする。

```json
{}
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

商品別月末評価額が
更新されないこと。

---

### 2.11 valueがnull

以下を指定する。

```json
{
  "value": null
}
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

既存の商品別月末評価額が
削除されないこと。

未登録状態へ
変更されないこと。

---

### 2.12 valueが負数

以下を指定する。

```json
{
  "value": -1
}
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

商品別月末評価額が
更新されないこと。

---

### 2.13 valueが小数

以下を指定する。

```json
{
  "value": 100.5
}
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

商品別月末評価額が
更新されないこと。

---

### 2.14 valueが文字列

以下を指定する。

```json
{
  "value": "900000"
}
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

文字列から整数へ
暗黙的に変換して
更新しないこと。

---

### 2.15 valueが保持可能範囲を超える

データベースで
保持可能な金額範囲を超える
`value`を指定する。

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

データベースエラーになる前に
入力値として拒否されること。

---

### 2.16 月末資産状況不存在

存在しない`snapshotId`を指定する。

期待結果：

```text
404 Not Found
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND
```

商品別月末評価額が
更新されないこと。

---

### 2.17 他利用者の月末資産状況

`X-User-Id`とは
異なる利用者に属する
`snapshotId`を指定する。

期待結果：

```text
404 Not Found
MONTH_END_ASSET_SNAPSHOT_NOT_FOUND
```

他利用者の月末資産状況が
存在することを
レスポンスから判別できないこと。

商品別月末評価額が
更新されないこと。

---

### 2.18 月末資産状況が確定済み

`confirmed = true`の
月末資産状況を指定する。

期待結果：

```text
409 Conflict
MONTH_END_ASSET_SNAPSHOT_CONFIRMED
```

商品別月末評価額が
更新されないこと。

---

### 2.19 保有商品不存在

存在しない
`holdingAssetId`を指定する。

期待結果：

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

商品別月末評価額が
更新されないこと。

---

### 2.20 他利用者の保有商品

他の利用者に属する
資産口座の`holdingAssetId`を指定する。

期待結果：

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

他利用者の保有商品が
存在することを
レスポンスから判別できないこと。

商品別月末評価額が
更新されないこと。

---

### 2.21 商品別月末評価額不存在

月末資産状況および
保有商品は存在するが、

```text
month_end_asset_snapshot_id = snapshotId
AND
holding_asset_id = holdingAssetId
```

を満たす
`month_end_holding_values`が
存在しない状態で実行する。

期待結果：

```text
404 Not Found
MONTH_END_HOLDING_VALUE_NOT_FOUND
```

新しい
`month_end_holding_values`レコードが
作成されないこと。

---

### 2.22 資産口座が対象年月時点で利用不可

保有商品が属する資産口座が、
`target_year_month`時点で
月末資産管理対象ではない状態で実行する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH
```

商品別月末評価額が
更新されないこと。

---

### 2.23 残高記録単位が口座単位

保有商品が属する資産口座の
`balance_recording_unit`が
口座単位の状態で実行する。

期待結果：

```text
409 Conflict
ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH
```

商品別月末評価額が
更新されないこと。

`month_end_asset_balances`も
変更されないこと。

---

### 2.24 保有商品が対象年月時点で評価額記録対象外

指定した保有商品が
`target_year_month`時点で
評価額記録対象ではない状態で実行する。

期待結果：

```text
409 Conflict
HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH
```

商品別月末評価額が
更新されないこと。

---

### 2.25 現在は無効だが対象年月時点では有効な保有商品

現在は無効化されているが、
`target_year_month`時点では
評価額記録対象であった
保有商品を指定する。

その他の更新条件を
すべて満たしている状態とする。

期待結果：

```text
200 OK
```

現在状態ではなく、
対象年月時点の状態を基準に
正しく更新できること。

---

### 2.26 現在は有効だが対象年月時点では対象外の保有商品

現在は有効であるが、
`target_year_month`時点では
評価額記録対象ではなかった
保有商品を指定する。

期待結果：

```text
409 Conflict
HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH
```

現在状態だけを基準として
誤って更新されないこと。

---

### 2.27 不要な項目を指定した場合

リクエストボディへ、
本APIで受け付けない項目を指定する。

例：

```json
{
  "value": 900000,
  "userId": "2",
  "holdingAssetId": "10"
}
```

API共通方針で定めた
未知項目の扱いに従うこと。

少なくとも、
リクエストボディの値によって

- 操作対象利用者
- 月末資産状況
- 保有商品
- 資産口座

を変更できないこと。

---

### 2.28 副作用

正常更新後に、
更新対象となる
`month_end_holding_values.value`
以外の業務データが
変更されていないことを確認する。

少なくとも、
以下が変更されていないこと。

- `month_end_asset_snapshots.confirmed`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_balances`
- 更新対象以外の`month_end_holding_values`
- `assessment_histories`

また、
以下が自動実行されていないこと。

- 月末資産状況の確定
- 月末資産状況の確定解除
- 商品別月末評価額の新規登録
- 商品別月末評価額の削除
- 月末資産残高の登録・更新
- 目的達成判定
- 目的達成判定履歴の再計算

---

### 2.29 トランザクション

商品別月末評価額更新処理の途中で
例外を発生させる。

期待結果：

- トランザクションがロールバックされること
- `month_end_holding_values.value`が更新前の値を保持していること
- 中途半端な更新状態が残らないこと

---

### 2.30 冪等性

同一の`snapshotId`、
`holdingAssetId`および
`value`で
複数回リクエストする。

例：

```text
1回目
PATCH value = 900000
    ↓
200 OK

2回目
PATCH value = 900000
    ↓
200 OK
```

期待結果：

- いずれも正常終了すること
- 最終的な`value`が`900000`であること
- 商品別月末評価額レコードが増加しないこと
- 重複データが作成されないこと

---

### 2.31 業務状態変更後の再実行

1回目の更新成功後に、
SNP-004 月末資産状況確定APIで
月末資産状況を確定する。

その後、
同一の更新リクエストを再実行する。

期待結果：

```text
409 Conflict
MONTH_END_ASSET_SNAPSHOT_CONFIRMED
```

1回目に更新した
商品別月末評価額が
変更されないこと。

---

### 2.32 レスポンス

正常更新時に、
以下の形式で返却されること。

```json
{
  "data": {
    "holdingAssetId": "5",
    "value": 900000
  }
}
```

以下を確認する。

- HTTPステータスが`200 OK`であること
- `holdingAssetId`がstringであること
- `value`がintegerであること
- `value = 0`の場合も`0`として返却されること
- API共通方針の成功レスポンス形式に従っていること

本APIでは、
以下の情報がレスポンスへ
含まれていないことを確認する。

- `id`
- `month_end_asset_snapshot_id`
- `user_id`
- `asset_account_id`
- `target_year_month`
- `confirmed`
- `holding_asset_name`
- `asset_account_name`
- `balance_recording_unit`
- `previous_value`
- `created_at`
- `updated_at`
- `month_end_asset_balances`

---

### 2.33 エラーレスポンス

各異常系について、
API共通方針で定めた
共通エラーレスポンス形式で
返却されることを確認する。

特に以下を確認する。

- `error.code`が期待するエラーコードであること
- `error.message`が設定されていること
- `error.details`が配列であること
- `error.requestId`が設定されていること
- 内部例外メッセージが含まれていないこと
- SQLが含まれていないこと
- PostgreSQLの制約名が含まれていないこと
- スタックトレースが含まれていないこと

---

### 2.34 利用者境界

複数の利用者について、

```text
User A
User B
```

それぞれに

- 月末資産状況
- 資産口座
- 保有商品
- 商品別月末評価額

を作成する。

`X-User-Id`に
User Aを指定した状態で、
User BのリソースIDを使用して
更新を試みる。

期待結果：

- User Bの月末資産状況を更新できないこと
- User Bの保有商品に紐づく商品別月末評価額を更新できないこと
- User Bのデータが存在することをレスポンスから推測できないこと
- User Bの`month_end_holding_values`が変更されていないこと

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
月末資産状況ID、
保有商品ID、
更新後の商品別月末評価額および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOから
検証済みの入力値を受け取り、
商品別月末評価額更新UseCaseを呼び出す。

UseCaseから受け取った更新結果を、
Responderへ渡す。

Actionでは、
以下の処理を行わない。

- `snapshotId`の形式検証
- `holdingAssetId`の形式検証
- リクエストボディの単項目バリデーション
- データベース検索
- 利用者境界の判定
- 月末資産状況の存在確認
- 月末資産状況の確定状態確認
- 保有商品の存在確認
- 商品別月末評価額の存在確認
- 対象年月時点の資産口座利用可否判定
- 残高記録単位の判定
- 対象年月時点の保有商品判定
- 商品別月末評価額の更新
- トランザクション制御
- レスポンス形式への変換

---

### 2.2 UseCase

商品別月末評価額更新の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 月末資産状況IDを受け取る
- 保有商品IDを受け取る
- 更新後の商品別月末評価額を受け取る
- 月末資産状況を取得する
- 月末資産状況が未確定であることを確認する
- 保有商品および所属資産口座を取得する
- 保有商品が操作対象利用者の資産口座に属していることを確認する
- 商品別月末評価額を取得する
- 商品別月末評価額が登録済みであることを確認する
- 資産口座の残高記録単位が商品単位であることを確認する
- 資産口座が対象年月時点で月末資産管理対象であることを確認する
- 保有商品が対象年月時点で評価額記録対象であることを確認する
- 商品別月末評価額を更新する
- 更新結果を返却する

指定された月末資産状況が存在しない場合、
または操作対象利用者に属していない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

指定された保有商品が存在しない場合、
または他の利用者に属する資産口座の
保有商品である場合は、
`HOLDING_ASSET_NOT_FOUND`
として扱う。

指定された月末資産状況・保有商品の
商品別月末評価額が存在しない場合は、
`MONTH_END_HOLDING_VALUE_NOT_FOUND`
として扱う。

---

### 2.3 Form Request / DTO

リクエストボディの
形式および単項目バリデーションを担当する。

検証対象は、
以下とする。

- `value`

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
- 商品別月末評価額の存在確認
- 残高記録単位確認
- 対象年月時点の資産口座利用可否確認
- 対象年月時点の保有商品判定

検証済みの入力値は、
入力用DTOへ変換して
UseCaseへ渡す。

例：

```php
final readonly class UpdateMonthEndHoldingValueInput
{
    public function __construct(
        public int $value,
    ) {
    }
}
```

---

### 2.4 パスパラメータ検証

`snapshotId`および
`holdingAssetId`は、
API共通方針に従って検証する。

以下を確認する。

- 指定されていること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

形式が不正な場合は、
`VALIDATION_ERROR`
として扱う。

パスパラメータの形式検証と、
対象リソースの存在確認は
分離する。

```text
形式不正
    → VALIDATION_ERROR

形式正常・リソース不存在
    → NOT_FOUND系エラー
```

---

### 2.5 Query

商品別月末評価額更新に必要な
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

Queryでは、
登録・更新処理を行わない。

---

### 2.6 月末資産状況取得

更新対象となる
商品別月末評価額が属する
月末資産状況は、
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

### 2.7 確定状態確認

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

本API内で、
自動的に確定解除してはならない。

---

### 2.8 対象年月の取得

資産口座および
保有商品の業務状態を判定する年月には、
月末資産状況の
`target_year_month`を使用する。

```php
$targetYearMonth =
    $snapshot->target_year_month;
```

リクエストから
対象年月を受け取らない。

```text
snapshotId
    ↓
month_end_asset_snapshots
    ↓
target_year_month
    ↓
資産口座・保有商品の対象年月判定
```

---

### 2.9 保有商品取得

指定された保有商品は、
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
        $holdingAssetId,
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
HoldingAsset::find($holdingAssetId);
```

取得できなかった場合は、
以下を区別せず
`HOLDING_ASSET_NOT_FOUND`
として扱う。

- 保有商品が存在しない
- 他の利用者に属する資産口座の保有商品である

---

### 2.10 商品別月末評価額取得

更新対象の商品別月末評価額は、
以下の組み合わせで取得する。

```text
month_end_asset_snapshot_id = snapshotId
AND
holding_asset_id = holdingAssetId
```

概念例：

```php
$holdingValue = MonthEndHoldingValue::query()
    ->where(
        'month_end_asset_snapshot_id',
        $snapshot->id,
    )
    ->where(
        'holding_asset_id',
        $holdingAsset->id,
    )
    ->first();
```

取得できなかった場合は、

`MONTH_END_HOLDING_VALUE_NOT_FOUND`

として扱う。

本APIでは、
該当レコードが存在しない場合に
新しい商品別月末評価額を
作成してはならない。

---

### 2.11 残高記録単位確認

保有商品が属する資産口座の
`balance_recording_unit`が
商品単位であることを確認する。

概念例：

```php
if (
    $holdingAsset->balance_recording_unit
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

口座単位の場合は、
BAL-003 月末資産残高更新APIを使用する。

---

### 2.12 対象年月時点の資産口座利用可否判定

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

現在の状態だけを使用して、
過去月の更新可否を
判断してはならない。

---

### 2.13 対象年月時点の保有商品判定

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
過去月の更新可否を
判断してはならない。

---

### 2.14 Repository

既存の商品別月末評価額の
更新を担当する。

更新する値は、
`value`のみとする。

概念例：

```php
public function updateValue(
    MonthEndHoldingValue $holdingValue,
    int $value,
): MonthEndHoldingValue {
    $holdingValue->value = $value;
    $holdingValue->save();

    return $holdingValue;
}
```

以下の項目は、
本APIで変更しない。

- `id`
- `month_end_asset_snapshot_id`
- `holding_asset_id`
- `created_at`

`updated_at`は、
Laravelによって更新する。

Repositoryでは、
以下の処理を行わない。

- 月末資産状況の確定
- 月末資産状況の確定解除
- 商品別月末評価額の新規登録
- 商品別月末評価額の削除
- 月末資産残高の登録・更新
- 目的達成判定の実行

---

### 2.15 create・upsertを使用しない

本APIは、
既存の商品別月末評価額を
更新する責務のみを持つ。

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
商品別月末評価額が
存在しない場合に
新しいレコードが作成されてしまう。

```text
登録
    → VAL-002

更新
    → VAL-003
```

の責務を維持するため、
更新対象が存在しない場合は

`MONTH_END_HOLDING_VALUE_NOT_FOUND`

を返却する。

---

### 2.16 Mass Assignment

クライアントから受け取った値を
そのままEloquentモデルへ
渡してはならない。

以下のような実装は避ける。

```php
$holdingValue->update(
    $request->all(),
);
```

更新値は、
検証済みの`value`のみを
明示的に指定する。

```php
$holdingValue->update([
    'value' => $input->value,
]);
```

これにより、
クライアントから

- `month_end_asset_snapshot_id`
- `holding_asset_id`
- `created_at`
- その他のサーバー管理項目

を意図せず変更されることを防止する。

---

### 2.17 0円の扱い

`value = 0`は、
有効な更新値として扱う。

以下のような
truthy / falsyによる判定は行わない。

```php
if (! $input->value) {
    // 0円まで未入力扱いになるため使用しない
}
```

Form Requestでは、
0を正常値として受け付ける。

```text
value = null
    → 不正

value = 0
    → 正常

value > 0
    → 正常
```

---

### 2.18 同一値への更新

更新前と同一の`value`が
指定された場合も、
正常な更新要求として扱う。

例えば、

```text
現在値
value = 900000

更新値
value = 900000
```

の場合も、
専用エラーを発生させない。

概念的には、
通常どおり更新処理を実行してよい。

```php
$holdingValue->value = $input->value;
$holdingValue->save();
```

Phase1では、
値が変化していないことを理由に
更新処理を特別に分岐させる必要はない。

---

### 2.19 トランザクション

業務条件の最終確認から
商品別月末評価額の更新までを、
1つのデータベーストランザクション内で実行する。

実装例：

```php
$holdingValue = DB::transaction(
    function () use (
        $userId,
        $snapshotId,
        $holdingAssetId,
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
                    $holdingAssetId,
                );

        if ($holdingAsset === null) {
            throw new
                HoldingAssetNotFoundException();
        }

        $holdingValue =
            $this->holdingValueQuery
                ->findBySnapshotAndHoldingAsset(
                    $snapshot->id,
                    $holdingAsset->id,
                );

        if ($holdingValue === null) {
            throw new
                MonthEndHoldingValueNotFoundException();
        }

        $this->recordingUnitValidator
            ->validateForHoldingValue(
                $holdingAsset->assetAccount,
            );

        $this->assetAccountAvailabilityValidator
            ->validate(
                $holdingAsset->assetAccount,
                $snapshot->target_year_month,
            );

        $this->holdingAssetAvailabilityValidator
            ->validate(
                $holdingAsset,
                $snapshot->target_year_month,
            );

        return $this->repository->updateValue(
            $holdingValue,
            $input->value,
        );
    },
);
```

処理途中で例外が発生した場合は、
商品別月末評価額を
更新前の状態へロールバックする。

---

### 2.20 排他制御

Phase1では、
商品別月末評価額更新専用の
楽観ロック用バージョン番号は
導入しない。

ただし、
更新処理中に
対象レコードの状態が変化する可能性を
考慮する必要がある場合は、
必要に応じて
`lockForUpdate()`を利用してよい。

概念例：

```php
$holdingValue = MonthEndHoldingValue::query()
    ->where(
        'month_end_asset_snapshot_id',
        $snapshotId,
    )
    ->where(
        'holding_asset_id',
        $holdingAssetId,
    )
    ->lockForUpdate()
    ->first();
```

Phase1では、
複雑な競合制御を追加するよりも、
トランザクション境界を明確にし、
必要な場合のみ
行ロックを採用する。

---

### 2.21 確定状態との競合

商品別月末評価額の更新と
月末資産状況の確定が
同時に実行される場合、

```text
VAL-003
    → confirmed = false確認

SNP-004
    → 確定処理

VAL-003
    → value更新
```

のような競合を
避ける必要がある。

そのため、
必要に応じて
月末資産状況取得時に
行ロックを使用する。

概念例：

```php
$snapshot = MonthEndAssetSnapshot::query()
    ->where('id', $snapshotId)
    ->where('user_id', $userId)
    ->lockForUpdate()
    ->first();
```

VAL-003とSNP-004で
同じ月末資産状況に対する
更新系処理の整合性を保つ。

具体的なロック方針は、
SNP-004と統一する。

---

### 2.22 業務ルール判定クラス

BAL系APIおよび
VAL系APIで共通して使用する
業務ルールについては、
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

VAL-002とVAL-003で
同一の業務ルールを
別々に実装しない。

各Validatorでは、
HTTPレスポンス生成や
商品別月末評価額の更新を行わない。

---

### 2.23 Responder

UseCaseから受け取った
更新後の商品別月末評価額を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`200 OK`とともに
`data`オブジェクトとして返却する。

Responderは、
以下の処理を行わない。

- データベース検索
- 利用者境界の判定
- 確定状態の判定
- 商品別月末評価額の存在確認
- 残高記録単位の判定
- 対象年月時点の資産口座利用可否判定
- 対象年月時点の保有商品判定
- 商品別月末評価額の更新

---

### 2.24 API Resource

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
整数の`0`として返却する。

本APIでは、
以下の情報は返却しない。

- `id`
- `month_end_asset_snapshot_id`
- `user_id`
- `asset_account_id`
- `target_year_month`
- `confirmed`
- `holding_asset_name`
- `asset_account_name`
- `balance_recording_unit`
- `previous_value`
- `created_at`
- `updated_at`
- `month_end_asset_balances`

---

### 2.25 Middleware

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

### 2.26 Eloquentモデル

`MonthEndHoldingValue`モデルは、
`month_end_holding_values`
テーブルへ対応する。

主に以下の属性を使用する。

```text
id
month_end_asset_snapshot_id
holding_asset_id
value
created_at
updated_at
```

`value`は、
日本円の整数値として扱う。

必要に応じて
integer castを設定する。

```php
protected function casts(): array
{
    return [
        'value' => 'integer',
    ];
}
```

必要に応じて、
以下のRelationを定義する。

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

### 2.27 更新後モデルの扱い

更新成功後は、
更新済みのモデルまたはDTOを
Responderへ返却する。

必要に応じて、
更新後の値を確実に取得するため
`refresh()`を使用してよい。

```php
$holdingValue->update([
    'value' => $value,
]);

$holdingValue->refresh();

return $holdingValue;
```

ただし、
API Resourceで必要となる項目が
すでにモデル上で確定している場合は、
不要な再検索を行わない。

---

### 2.28 N+1問題

本APIは、
単一の商品別月末評価額を
更新するAPIであるため、
一覧APIのような
典型的なN+1問題は発生しにくい。

ただし、
以下のようにRelationshipを
段階的に遅延ロードし、
不要なSQLを増加させない。

```text
holdingValue
    ↓ 個別取得
holdingAsset
    ↓ 個別取得
assetAccount
    ↓ 個別取得
availableSettings
```

更新処理で必要となる関連情報は、
JOIN、
Eager Loadingまたは
専用Queryによって
必要な範囲でまとめて取得する。

---

### 2.29 キャッシュ

Phase1では、
本API専用の
アプリケーションキャッシュを
使用しない。

商品別月末評価額は
更新対象となる業務データであり、
更新直後に最新状態を
参照できる必要がある。

VAL-003成功後に
VAL-001を取得した場合は、
更新後の値が返却されることを前提とする。

---

### 2.30 例外変換

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
| 商品別月末評価額不存在 | `MONTH_END_HOLDING_VALUE_NOT_FOUND` |
| 資産口座が対象年月時点で利用不可 | `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH` |
| 残高記録単位不一致 | `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH` |
| 保有商品が対象年月時点で対象外 | `HOLDING_ASSET_NOT_AVAILABLE_FOR_TARGET_MONTH` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

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

更新前後の商品別月末評価額については、
不要にエラーログへ出力しない。

---

## 3. 関連ドキュメント

- [VAL-003 API詳細設計](../../../api/details/holding-values/val-003-update.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [商品別月末評価額 Laravelアーキテクチャ設計](./README.md)
- [VAL-003 テスト設計](../../../tests/holding-values/val-003-update.md)
- [商品別月末評価額 テスト設計](../../../tests/holding-values/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)