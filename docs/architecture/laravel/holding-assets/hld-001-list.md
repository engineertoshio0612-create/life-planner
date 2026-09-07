# HLD-001 保有商品一覧取得

## 1. 概要

本ドキュメントでは、HLD-001 保有商品一覧取得APIをLaravelで実装する際のアーキテクチャおよび責務分離方針を定義する。

本APIでは、操作対象利用者に帰属する保有商品を取得し、所属する資産口座情報および利用状態を含めた一覧として返却する。

取得対象には、利用中の保有商品だけでなく無効化済みの保有商品も含める。

Laravel実装では、主に以下の処理を行う。

* `X-User-Id`から操作対象利用者を特定する
* 操作対象利用者に帰属する保有商品を取得する
* 商品単位で管理する資産口座に属する保有商品のみに限定する
* 無効化済みの保有商品を含めて取得する
* 所属する資産口座情報を取得する
* `deleted_at`から保有商品の利用状態を判定する
* 利用中、無効化済み、商品名の順で並び順を制御する
* API Resourceを使用してAPIレスポンス形式へ変換する

参照処理のみであるため、明示的なデータベーストランザクションおよび排他制御は行わない。

保有商品の取得条件、利用者境界、無効化済みデータの取得および表示順に関する責務を適切に分離し、Controllerへ業務ロジックを持たせない構成とする。

## 2. Laravel実装方針

HLD-001 保有商品一覧取得APIは、以下の責務分離で実装する。

```text
Route
    ↓
Controller
    ↓
Action
    ↓
UseCase
    ↓
Query
    ↓
Model / Database
    ↓
UseCase
    ↓
Resource
    ↓
Responder
    ↓
Response
```

### 2.1 Route

Routeでは、HLD-001のエンドポイントとControllerを対応付ける。

```text
GET /api/v1/holding-assets
```

本APIでは、パスパラメータおよびクエリパラメータを使用しない。

### 2.2 Controller

ControllerはHTTPリクエストの受付を担当する。

主な責務は以下とする。

* リクエストを受け付ける
* `UserContext`から操作対象利用者IDを取得する
* Actionを呼び出す
* Actionの実行結果をResponderへ渡す

Controllerでは、保有商品の取得条件、利用状態判定、表示順制御などの業務ロジックを実装しない。

`X-User-Id`の必須チェック、形式チェック、利用者存在確認および論理削除確認は、API共通の利用者コンテキスト生成処理で完了していることを前提とする。

### 2.3 Action

ActionはControllerとUseCaseの境界として機能する。

主な責務は以下とする。

* Controllerから操作対象利用者IDを受け取る
* HLD-001のUseCaseを呼び出す
* UseCaseの実行結果をControllerへ返却する

Actionにはデータ取得条件やEloquentによる検索処理を記述しない。

HLD-001では入力項目が存在しないため、API固有のRequest DTOは原則として使用せず、操作対象利用者IDのみをUseCaseへ渡す。

### 2.4 UseCase

UseCaseはHLD-001のアプリケーション処理を統括する。

主な責務は以下とする。

* 操作対象利用者IDを受け取る
* Queryへ保有商品一覧の取得を依頼する
* 取得結果を返却する

保有商品の検索条件やSQL構築はQueryへ委譲し、UseCaseからEloquent Query Builderを直接操作しない。

本APIは参照処理のみであり、データ更新を行わないため、UseCaseではデータベーストランザクションを開始しない。

### 2.5 Query

保有商品一覧の取得処理はQueryへ集約する。

Queryでは、主に以下の条件を適用する。

```text
holding_assets
    ↓
操作対象利用者に帰属
    ↓
商品単位で管理する資産口座に所属
    ↓
利用中・無効化済みを両方取得
    ↓
所属資産口座を取得
    ↓
利用状態順
    ↓
商品名昇順
```

取得対象は、操作対象利用者に帰属する保有商品だけとし、他利用者の保有商品を取得してはならない。

また、所属する資産口座の残高記録単位が「商品単位」である保有商品のみに限定する。

保有商品は論理削除を採用しているが、本APIでは無効化済みの商品も一覧表示する必要があるため、通常のSoftDeletesによる取得対象除外をそのまま使用しない。

利用中および無効化済みの両方を明示的に取得対象とする。

所属する資産口座情報については、一覧レスポンスで資産口座IDおよび資産口座名を返却するため、N+1問題が発生しない方法で取得する。

### 2.6 利用者境界

保有商品の取得では、必ず操作対象利用者による境界条件を適用する。

概念的には以下の条件とする。

```text
holding_assets.user_id
    = 操作対象利用者ID
```

または、エンティティ間の関連定義に応じて、所属資産口座を経由して利用者境界を保証する。

いずれの方式を採用する場合も、Query単体で他利用者の保有商品を取得できない検索条件とする。

### 2.7 無効化済み保有商品の取得

HLD-001では、利用中の商品だけでなく無効化済みの商品も取得対象とする。

利用状態は、

```text
holding_assets.deleted_at
```

から判定する。

```text
deleted_at IS NULL
    → 利用中

deleted_at IS NOT NULL
    → 無効化済み
```

Laravelの`SoftDeletes`を使用する場合は、無効化済みデータを含めるための取得方法をQuery内で明示する。

`deleted_at`自体はAPIレスポンスへ公開せず、Resourceで`isEnabled`へ変換する。

### 2.8 表示順

表示順はデータ取得時にQueryで確定させる。

以下の順序とする。

```text
1. 利用中
2. 無効化済み
3. 同一利用状態では商品名昇順
```

PHP側で取得後に並び替えるのではなく、可能な限りSQLの`ORDER BY`で順序を保証する。

これにより、呼び出し側やResourceが表示順を意識する必要がない構成とする。

### 2.9 Model

主に以下のEloquent Modelを使用する。

```text
HoldingAsset
AssetAccount
```

`HoldingAsset`から所属する`AssetAccount`を参照できるRelationを定義する。

概念的には以下とする。

```text
HoldingAsset
    belongsTo
AssetAccount
```

Modelはテーブル構造、キャスト、Relationなどの永続化モデルとしての責務を持ち、HLD-001固有のユースケース制御は持たせない。

### 2.10 Resource

API Resourceでは、取得した保有商品をHLD-001のレスポンス形式へ変換する。

主に以下を返却する。

```text
id
assetAccountId
assetAccountName
name
startYearMonth
isEnabled
```

`id`および`assetAccountId`は、API共通方針に従い文字列として返却する。

利用状態は`deleted_at`から以下のように変換する。

```text
deleted_at === null
    → isEnabled = true

deleted_at !== null
    → isEnabled = false
```

以下のようなデータベース内部情報はレスポンスへ公開しない。

```text
user_id
deleted_at
created_at
updated_at
```

Resourceはレスポンス形式への変換のみを担当し、追加のデータベースアクセスや検索処理を行わない。

### 2.11 Responder

ResponderはHLD-001のHTTP正常レスポンス生成を担当する。

正常時は以下を返却する。

```http
200 OK
```

レスポンス形式は以下とする。

```json
{
  "data": [
    {
      "id": "1",
      "assetAccountId": "3",
      "assetAccountName": "SBI証券",
      "name": "eMAXIS Slim 全世界株式",
      "startYearMonth": "2026-01",
      "isEnabled": true
    }
  ]
}
```

保有商品が存在しない場合もエラーとはせず、

```json
{
  "data": []
}
```

として`200 OK`を返却する。

### 2.12 トランザクション・排他制御

HLD-001は参照処理のみであり、データを更新しない。

そのため、明示的なデータベーストランザクションは使用しない。

また、行ロックなどの排他制御も行わない。

API実行中に保有商品または資産口座の状態が変更された場合は、Query実行時点で取得できた情報をレスポンスとして返却する。

### 2.13 キャッシュ

Phase1ではキャッシュを使用しない。

HLD-001ではQueryから取得した最新のデータをそのままレスポンスとして返却する。

将来的に保有商品数やアクセス数が増加し、パフォーマンス上の必要性が生じた場合にキャッシュ導入を検討する。

## 3. 関連ドキュメント

* [HLD-001 保有商品一覧取得 API詳細](../../../api/details/holding-assets/hld-001-list.md)
* [HLD-002 保有商品登録](./hld-002-create.md)
* [HLD-003 保有商品詳細取得](./hld-003-detail.md)
* [HLD-004 保有商品更新](./hld-004-update.md)
* [HLD-005 保有商品無効化](./hld-005-disable.md)
* [API共通方針](../../../api/api-common-policy.md)
* [API一覧](../../../api/api-list.md)
* [エラーコード一覧](../../../api/error-codes.md)
* [機能要件](../../../requirements/functional-requirements.md)
* [ユビキタス言語集](../../../requirements/glossary.md)
* [エンティティ定義](../../../requirements/entities.md)
* [テーブル定義書](../../../database/table-definition.md)
* [ER図](../../../database/er-diagram-phase1.md)
