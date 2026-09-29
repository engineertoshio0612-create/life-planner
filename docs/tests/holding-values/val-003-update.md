# VAL-003 商品別月末評価額更新

## 1. 概要

操作対象となる利用者について、
指定した月末資産状況に紐づく
既存の商品別月末評価額を更新する機能をテストする。

本テストでは、
残高記録単位が商品単位である資産口座に属し、
対象年月時点で評価額記録対象となる保有商品について、
登録済みの月末評価額を正しく更新できることを確認する。

主なテスト対象は以下とする。

- 商品別月末評価額の正常更新
- 0円および同一値への更新
- リクエストおよびバリデーション
- 利用者境界
- 月末資産状況および確定状態
- 商品別月末評価額の存在確認
- 対象年月時点の資産口座・保有商品の利用可否
- 残高記録単位
- レスポンス契約
- 副作用
- トランザクションおよびロールバック
- 排他・同時実行
- 異常系およびエラーレスポンス

更新できるのは、
未確定の月末資産状況に紐づく
登録済みの商品別月末評価額のみとする。

商品別月末評価額が存在しない場合は、
VAL-003では新規登録せず、
`MONTH_END_HOLDING_VALUE_NOT_FOUND` として扱うことを確認する。

新しい商品別月末評価額の登録は、
VAL-002 商品別月末評価額登録APIの責務とする。

また、
`value = 0` および現在値と同一の値への更新も
正常な更新として扱うことを確認する。

資産口座および保有商品の更新可否は、
現在の状態だけではなく、
月末資産状況の対象年月時点の状態を基準として
判定されることを確認する。

更新処理では、
業務条件の最終確認からUPDATEまでを
トランザクション内で実行し、
業務ルール違反や例外発生時に
中途半端な更新状態が残らないことを確認する。

更新対象は商品別月末評価額の `value` のみとし、
月末資産状況、保有商品、資産口座、
月末資産残高などへ
意図しない副作用が発生しないことを確認する。

口座単位で管理する月末資産残高は
本APIの更新・テスト対象外とし、
BAL-003 月末資産残高更新APIの責務とする。

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

## 3. 関連ドキュメント

- [VAL-003 API詳細設計](../../api/details/holding-values/val-003-update.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [VAL-003 Laravelアーキテクチャ設計](../../architecture/laravel/holding-values/val-003-update.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [VAL-003 Reactアーキテクチャ設計](../../architecture/react/holding-values/val-003-update.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)