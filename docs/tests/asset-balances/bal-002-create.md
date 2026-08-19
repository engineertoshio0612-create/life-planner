# BAL-002 月末資産残高登録

## 1. 概要

本ドキュメントでは、
BAL-002 月末資産残高登録APIに対する
テスト方針および主要なテスト観点を定義する。

BAL-002では、
操作対象となる利用者について、
指定した月末資産状況に紐づく
資産口座単位の月末資産残高を新規登録する。

テストでは、
有効な月末資産状況、資産口座および残高を指定した場合に、
`month_end_asset_balances`へ
正しい月末資産残高が登録され、
`201 Created`が返却されることを確認する。

また、登録対象となる資産口座は、
対象年月時点で月末資産管理対象であり、
残高記録単位が口座単位である場合に限定されることを検証する。

特に、`balance = 0`は
有効な月末資産残高として扱い、
未入力や`null`とは明確に区別する。

主なテスト対象は、
以下とする。

* 月末資産残高の正常登録
* `snapshotId`の形式検証
* `assetAccountId`の入力検証
* `balance`の入力値・境界値検証
* 月末資産状況の存在確認と利用者境界
* 資産口座の存在確認と利用者境界
* 月末資産状況の確定状態
* 対象年月時点の資産口座の利用可能状態
* 残高記録単位
* 同一月末資産状況・資産口座への重複登録
* UNIQUE制約による同時実行時の重複防止
* `0円`の登録
* レスポンス契約
* トランザクションとロールバック
* 登録処理による副作用
* 共通エラーおよび想定外例外の扱い

同一の月末資産状況・資産口座に
月末資産残高がすでに存在する場合は、
BAL-002によって既存値が上書きされず、
`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`として
処理されることを確認する。

また、アプリケーション側の重複確認だけでなく、
データベースのUNIQUE制約によって
同時実行時にも複数レコードが作成されないことを検証する。

登録処理が失敗した場合は、
トランザクションによって変更がロールバックされ、
不完全な月末資産残高が残らないことを確認する。

さらに、本APIの実行によって
月末資産状況の確定、
商品別月末評価額の登録、
目的達成判定などの
意図しない副作用が発生しないことも検証する。

---

# 2. テスト観点

## 2.1 正常系

- 有効な入力値で月末資産残高を登録できること
- `201 Created`で返却されること
- `month_end_asset_balances`へ1件登録されること
- `month_end_asset_snapshot_id`に指定した月末資産状況IDが設定されること
- `asset_account_id`に指定した資産口座IDが設定されること
- `balance`に指定した金額が設定されること
- 操作対象利用者に属するデータとして登録されること
- 月末資産状況の`confirmed`が変更されないこと
- 商品別月末評価額が登録されないこと
- 目的達成判定が自動実行されないこと

---

## 2.2 snapshotId

- 正しい`snapshotId`を指定して登録できること
- `snapshotId = 1`を指定できること
- `snapshotId = 0`でバリデーションエラーとなること
- 負数でバリデーションエラーとなること
- 小数でバリデーションエラーとなること
- 文字列`abc`でバリデーションエラーとなること
- ID形式不正時に`422 Unprocessable Entity`となること
- ID形式不正時に`VALIDATION_ERROR`となること

---

## 2.3 assetAccountId

- 正しい`assetAccountId`を指定して登録できること
- `assetAccountId`未指定で`422 Unprocessable Entity`となること
- `null`でバリデーションエラーとなること
- 空文字でバリデーションエラーとなること
- `"0"`でバリデーションエラーとなること
- 負数形式でバリデーションエラーとなること
- `"abc"`でバリデーションエラーとなること
- 数値型で送信した場合の扱いがAPI仕様と一致すること

---

## 2.4 balance

以下を確認する。

- 正の整数を登録できること
- `balance = 0`を登録できること
- `balance = 1`を登録できること
- 保持可能な最大値を登録できること
- `balance`未指定でバリデーションエラーとなること
- `balance = null`でバリデーションエラーとなること
- `balance = -1`でバリデーションエラーとなること
- 小数値でバリデーションエラーとなること
- 数値文字列でバリデーションエラーとなること
- 保持可能な範囲を超える値でバリデーションエラーとなること

---

## 2.5 月末資産状況の存在確認

- 存在する月末資産状況へ登録できること
- 存在しない`snapshotId`で`404 Not Found`となること
- 存在しない場合に`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`となること
- 存在しない場合に月末資産残高が登録されないこと

---

## 2.6 月末資産状況の利用者境界

- 操作対象利用者に属する月末資産状況へ登録できること
- 他利用者に属する月末資産状況へ登録できないこと
- 他利用者の`snapshotId`で`404 Not Found`となること
- 他利用者の場合に`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`となること
- 他利用者の月末資産状況が存在することをレスポンスから判別できないこと
- 月末資産状況の検索条件に`user_id`が含まれていること

---

## 2.7 資産口座の存在確認

- 存在する資産口座を指定して登録できること
- 存在しない`assetAccountId`で`404 Not Found`となること
- 存在しない場合に`ASSET_ACCOUNT_NOT_FOUND`となること
- 存在しない資産口座を指定した場合に月末資産残高が登録されないこと

---

## 2.8 資産口座の利用者境界

- 操作対象利用者に属する資産口座へ登録できること
- 他利用者に属する資産口座へ登録できないこと
- 他利用者の資産口座IDで`404 Not Found`となること
- 他利用者の場合に`ASSET_ACCOUNT_NOT_FOUND`となること
- 他利用者の資産口座が存在することをレスポンスから判別できないこと
- 資産口座の検索条件に`user_id`が含まれていること

---

## 2.9 月末資産状況の確定状態

未確定の場合：

```text
confirmed = false
```

について、

- 月末資産残高を登録できること

確定済みの場合：

```text
confirmed = true
```

について、

- 月末資産残高を登録できないこと
- `409 Conflict`となること
- `MONTH_END_ASSET_SNAPSHOT_CONFIRMED`となること
- 月末資産残高が新規作成されないこと

---

## 2.10 対象年月時点の利用可能状態

資産口座の利用可能期間が
対象年月によって異なるデータを用意する。

以下を確認する。

- 対象年月時点で月末資産管理対象の資産口座へ登録できること
- 対象年月時点で月末資産管理対象ではない資産口座へ登録できないこと
- 対象外の場合に`409 Conflict`となること
- 対象外の場合に`ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`となること
- 現在の利用状態だけを使用して過去月の登録可否を判定しないこと
- 過去月では対象だが現在は対象外の資産口座へ、該当過去月で登録できること
- 現在は対象だが対象年月時点では対象外の資産口座へ登録できないこと

---

## 2.11 残高記録単位

残高記録単位が
口座単位の場合：

```text
balance_recording_unit = 口座単位
```

について、

- BAL-002で登録できること

商品単位の場合：

```text
balance_recording_unit = 商品単位
```

について、

- BAL-002で登録できないこと
- `409 Conflict`となること
- `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`となること
- 月末資産残高レコードが作成されないこと

---

## 2.12 重複登録

以下の組み合わせについて、
すでに月末資産残高が存在する状態を用意する。

```text
month_end_asset_snapshot_id = 12
asset_account_id = 3
```

以下を確認する。

- 同じ組み合わせで新規登録できないこと
- `409 Conflict`となること
- `MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`となること
- 既存レコードが更新されないこと
- 新しいレコードが作成されないこと
- `balance`が既存値と同じでもエラーとなること
- `balance`が既存値と異なっていてもBAL-002では上書きされないこと
- 変更が必要な場合はBAL-003を利用する設計になっていること

---

## 2.13 UNIQUE制約

同一の月末資産状況・資産口座について
複数の登録要求を同時実行する。

以下を確認する。

- 最終的に1件のみ登録されること
- UNIQUE制約によって重複登録が防止されること
- 1件のリクエストが`201 Created`となること
- 競合したリクエストが`409 Conflict`となること
- UNIQUE制約違反が`MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`へ変換されること
- PostgreSQLの制約名や内部エラーがレスポンスへ公開されないこと

---

## 2.14 0円

`balance = 0`を送信した場合について、
以下を確認する。

- 正常に登録できること
- `201 Created`となること
- データベースへ`0`として保存されること
- レスポンスで`balance = 0`となること
- `0`が未入力として扱われないこと
- `null`へ変換されないこと

---

## 2.15 レスポンス契約

- JSONフィールド名がcamelCaseであること
- `data`がobjectで返却されること
- `assetAccountId`が文字列で返却されること
- `balance`がintegerで返却されること
- `balance = 0`が整数の`0`として返却されること
- 月末資産残高IDがレスポンスへ含まれないこと
- `snapshotId`がレスポンスへ含まれないこと
- `userId`がレスポンスへ含まれないこと
- `targetYearMonth`がレスポンスへ含まれないこと
- `confirmed`がレスポンスへ含まれないこと
- 資産口座名がレスポンスへ含まれないこと
- `createdAt`がレスポンスへ含まれないこと
- `updatedAt`がレスポンスへ含まれないこと
- DB内部のsnake_caseのカラム名がそのまま公開されないこと

---

## 2.16 トランザクション

- 業務条件の最終確認から月末資産残高登録までが同一トランザクション内で実行されること
- 登録途中で例外が発生した場合にロールバックされること
- 業務条件を満たさない場合に月末資産残高が登録されないこと
- 登録失敗時に不完全なレコードが残らないこと

---

## 2.17 副作用

本API実行によって、
以下が変更されないことを確認する。

- `month_end_asset_snapshots.confirmed`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_holding_values`
- `assessment_histories`

以下の副作用が
発生しないことを確認する。

- 月末資産状況を自動確定しない
- 商品別月末評価額を自動登録しない
- 目的達成判定を自動実行しない
- 過去の目的達成判定履歴を変更しない

---

## 2.18 エラー時

以下のエラー時に、
月末資産残高が登録されないことを確認する。

- `USER_CONTEXT_REQUIRED`
- `INVALID_USER_ID`
- `USER_NOT_FOUND`
- `VALIDATION_ERROR`
- `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
- `MONTH_END_ASSET_SNAPSHOT_CONFIRMED`
- `ASSET_ACCOUNT_NOT_FOUND`
- `ASSET_ACCOUNT_NOT_AVAILABLE_FOR_TARGET_MONTH`
- `ASSET_ACCOUNT_BALANCE_RECORDING_UNIT_MISMATCH`
- `MONTH_END_ASSET_BALANCE_ALREADY_EXISTS`

---

## 2.19 異常系

- 想定外の例外で`500 Internal Server Error`となること
- 想定外の例外発生時にトランザクションがロールバックされること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログに同じリクエストIDが記録されること
- SQLがレスポンスへ含まれないこと
- PostgreSQLの制約名がレスポンスへ含まれないこと
- スタックトレースがレスポンスへ含まれないこと
- 内部例外メッセージがレスポンスへ含まれないこと

---

## 3. 関連ドキュメント

* [API一覧](../../api/api-list.md)
* [API共通方針](../../api/api-common-policy.md)
* [BAL-002 月末資産残高登録 API詳細設計](../../api/details/asset-balances/bal-002-create.md)
* [BAL-001 月末資産残高一覧取得 テスト設計](./bal-001-list.md)
* [BAL-003 月末資産残高更新 テスト設計](./bal-003-update.md)
* [機能要件](../../requirements/functional-requirements.md)
* [ユビキタス言語集](../../glossary.md)
* [エンティティ定義](../../entities.md)
* [テーブル定義書](../../table-definition.md)
* [ER図](../../er-diagram-phase1.md)
* [Laravelアーキテクチャ設計](../../architecture/laravel/asset-balances/README.md)
* [Reactアーキテクチャ設計](../../architecture/react/asset-balances/README.md)
* [バックエンドテスト方針](../backend/README.md)
* [フロントエンドテスト方針](../frontend/README.md)
* [E2Eテスト方針](../e2e/README.md)

