# 月末資産残高 Laravelアーキテクチャ設計

## 1. 概要

本ディレクトリでは、
月末資産残高に関するAPIをLaravelで実装する際の
アーキテクチャおよび責務分離方針を定義する。

対象APIは以下とする。

| API ID  | API名       | ドキュメント                                   |
| ------- | ---------- | ---------------------------------------- |
| BAL-001 | 月末資産残高一覧取得 | [bal-001-list.md](./bal-001-list.md)     |
| BAL-002 | 月末資産残高登録   | [bal-002-create.md](./bal-002-create.md) |
| BAL-003 | 月末資産残高更新   | [bal-003-update.md](./bal-003-update.md) |

## 2. 共通設計方針

月末資産残高は、
月末資産状況と資産口座の組み合わせに対して管理する。

```text
MonthEndAssetSnapshot
    ↓
MonthEndAssetBalance
    ↓
AssetAccount
```

対象となるのは、
残高記録単位が**口座単位**である資産口座のみとする。

商品単位で残高を記録する資産口座については、
商品別月末評価額API（VAL系）で扱い、
BAL系APIでは処理しない。

また、対象年月時点で資産口座が
月末資産管理対象であるかの判定には、
`asset_account_available_settings`を使用する。

現在の資産口座状態だけを参照して、
過去月の取得・登録・更新可否を判断してはならない。

## 3. 責務分離

Laravel実装では、概ね以下の責務へ分離する。

```text
Middleware
    ↓
Action
    ↓
UseCase
    ├─ Query
    ├─ 業務ルール判定
    └─ Repository
    ↓
Responder
    ↓
API Resource
```

ActionはHTTPリクエストとUseCaseの橋渡しに限定し、
業務ルールやデータベース操作を直接記述しない。

Queryはデータ取得、
Repositoryは登録・更新、
UseCaseは処理の流れと業務ルールの調整を担当する。

対象年月時点の利用可否判定や
残高記録単位の判定など、
BAL-002とBAL-003で共通する業務ルールは
Validator等へ分離し、重複実装を避ける。

## 4. 登録・更新の責務

BAL-002とBAL-003では、
登録と更新の責務を明確に分離する。

```text
BAL-002
    → 未登録の月末資産残高を新規登録

BAL-003
    → 登録済みの月末資産残高を更新
```

BAL-003では`upsert`や`updateOrCreate`を使用せず、
更新対象が存在しない場合は
`MONTH_END_ASSET_BALANCE_NOT_FOUND`として扱う。

また、`balance = 0`は有効な残高として扱い、
未登録状態や未入力と区別する。

## 5. 関連ドキュメント

* [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
* [API一覧](../../../api/api-list.md)
* [API共通方針](../../../api/api-common-policy.md)
* [エラーコード一覧](../../../api/error-codes.md)
* [月末資産残高 API詳細設計](../../../api/details/asset-balances/README.md)
* [月末資産残高 テスト設計](../../../tests/asset-balances/README.md)
* [機能要件](../../../requirements/functional-requirements.md)
* [テーブル定義書](../../../table-definition.md)
* [ER図](../../../er-diagram-phase1.md)
