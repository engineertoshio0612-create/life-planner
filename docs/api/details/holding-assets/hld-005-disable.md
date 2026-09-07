# HLD-005 保有商品無効化

## 1. 概要

操作対象となる利用者に帰属する利用中の保有商品を無効化する。

無効化した保有商品は、無効化後の対象年月における商品別月末評価額の登録対象としない。

過去の商品別月末評価額、月末資産状況および目的達成判定履歴は保持する。

保有商品は物理削除せず、`deleted_at`による論理削除で無効化を表現する。

---

## 2. ユースケース

利用者は、売却、解約または保有終了などにより、今後利用しなくなった保有商品を無効化する。

例えば、以下の場合に使用する。

- 投資信託を売却した
- 株式をすべて売却した
- 預金商品を解約した
- 今後の商品別月末評価額を登録しない

同じ保有商品を再度管理する場合は、無効化した保有商品を再有効化せず、新しい保有商品として登録する。

---

## 3. エンドポイント

```http
PATCH /api/v1/holding-assets/{holdingAssetId}/disable
```

保有商品の通常更新を行うHLD-004 保有商品更新APIとは、エンドポイントを分離する。

```http
PATCH /api/v1/holding-assets/{holdingAssetId}
```

HLD-004は保有商品情報の更新、HLD-005は保有商品の無効化という異なる業務操作として扱う。

---

## 4. HTTPメソッド

```text
PATCH
```

本APIは、保有商品リソース全体を削除するのではなく、利用状態を「利用中」から「無効化済み」へ変更する。

無効化は、`deleted_at`へ無効化日時を設定することで表現する。

保有商品名、商品種別、所属資産口座、利用開始年月および備考は変更しない。

また、無効化という操作自体を`/disable`で明示しているため、リクエストボディによる利用状態の指定は行わない。

---

## 5. 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、`X-User-Id`リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

指定された利用者に帰属する保有商品のみ無効化できる。

他の利用者に帰属する保有商品は、無効化できない。

他の利用者に帰属する保有商品IDが指定された場合は、対象が存在しないものとして扱い、`404 Not Found`を返却する。

---

## 6. パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
| --- | --- | :---: | --- |
| `holdingAssetId` | string | ○ | 無効化対象となる保有商品ID |

`holdingAssetId`には、無効化対象となる保有商品のIDを指定する。

指定された保有商品が操作対象利用者に帰属しない場合は、対象が存在しないものとして扱う。

---

## 7. クエリパラメータ

なし。

本APIでは、無効化対象となる保有商品は`holdingAssetId`によって特定する。

以下のような利用状態を変更するためのクエリパラメータは使用しない。

- `isEnabled`
- `disabled`
- `status`

無効化という操作は、

```http
PATCH /api/v1/holding-assets/{holdingAssetId}/disable
```

というエンドポイント自体で表現する。

---

## 8. リクエストヘッダー

### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
| --- | :---: | --- |
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
PATCH /api/v1/holding-assets/15/disable
Accept: application/json
X-User-Id: 1
```

本APIでは、リクエストボディを使用しないため、`Content-Type: application/json`は必須としない。

---

## 9. リクエストボディ

なし。

本APIは、

```http
PATCH /api/v1/holding-assets/{holdingAssetId}/disable
```

というエンドポイントによって「指定した保有商品を無効化する」という操作を明示する。

そのため、以下のような利用状態指定用のリクエストボディは送信しない。

```json
{
  "isEnabled": false
}
```

また、再有効化はPhase1の対象外であるため、

```json
{
  "isEnabled": true
}
```

のような指定も受け付けない。

---

## 10. リクエスト項目

なし。

本APIでクライアントから指定する業務上の入力値は、パスパラメータの`holdingAssetId`のみとする。

以下の項目はリクエストボディで受け付けない。

- `isEnabled`
- `id`
- `assetAccountId`
- `assetAccountName`
- `name`
- `productType`
- `startYearMonth`
- `memo`
- `userId`
- `createdAt`
- `updatedAt`
- `deletedAt`

保有商品名、商品種別および備考の変更は、HLD-004 保有商品更新APIで行う。

利用状態は、HLD-005の実行そのものによって「無効化済み」へ変更する。

---

## 11. 更新対象外項目

HLD-005では、保有商品の無効化に必要な

```text
holding_assets.deleted_at
```

のみを更新する。

以下の項目は更新しない。

- `holding_assets.id`
- `holding_assets.asset_account_id`
- `holding_assets.name`
- `holding_assets.product_type`
- `holding_assets.start_year_month`
- `holding_assets.memo`
- `holding_assets.created_at`

`holding_assets.updated_at`については、Eloquentによる通常の更新処理に従い、無効化操作に伴う更新日時として更新する。

---

### 11.1 所属資産口座

無効化によって、

```text
holding_assets.asset_account_id
```

を変更しない。

保有商品の所属資産口座を別の資産口座へ変更する操作は、HLD-005の責務に含めない。

---

### 11.2 保有商品名

無効化によって、

```text
holding_assets.name
```

を変更しない。

保有商品名の変更は、HLD-004 保有商品更新APIで扱う。

---

### 11.3 商品種別

無効化によって、

```text
holding_assets.product_type
```

を変更しない。

商品種別の変更は、HLD-004 保有商品更新APIで扱う。

---

### 11.4 利用開始年月

無効化によって、

```text
holding_assets.start_year_month
```

を変更しない。

過去にどの対象年月から保有商品として管理されていたかを保持する。

---

### 11.5 備考

無効化によって、

```text
holding_assets.memo
```

を変更しない。

備考の変更は、HLD-004 保有商品更新APIで扱う。

---

### 11.6 過去の商品別月末評価額

HLD-005では、保有商品に関連する既存の

```text
month_end_holding_values
```

を更新・削除しない。

保有商品の無効化は、過去に記録された商品別月末評価額を無効化する操作ではない。

---

### 11.7 月末資産状況

HLD-005では、

```text
month_end_asset_snapshots
```

を更新しない。

特に、

```text
confirmed
confirmed_at
```

を変更しない。

保有商品の無効化によって、過去の月末資産状況を自動的に未確定へ戻さない。

---

## 12. 業務ルール

### 12.1 操作対象利用者に帰属する保有商品のみ無効化できる

指定された`holdingAssetId`に対応する保有商品が、操作対象利用者に帰属することを必須とする。

概念的には、

```text
holding_assets
    ↓
asset_accounts
    ↓
asset_accounts.user_id
        = 操作対象利用者ID
```

を満たす必要がある。

他利用者に帰属する保有商品は、対象が存在しないものとして扱う。

---

### 12.2 論理削除で無効化する

保有商品は、物理削除しない。

無効化時に、

```text
holding_assets.deleted_at
```

へ現在日時を設定する。

概念的には、

```text
deleted_at
    NULL
        ↓
    現在日時
```

とする。

これにより、過去の商品別月末評価額との関連を維持する。

---

### 12.3 過去データを保持する

保有商品を無効化しても、過去に登録された

```text
month_end_holding_values
```

は保持する。

過去の

- 月末資産状況
- 資産状況
- 資産推移
- 目的達成判定履歴

を再現できる状態を維持する。

---

### 12.4 無効化後の新規月末評価額登録対象外

無効化された保有商品は、無効化後の対象年月について、新しい商品別月末評価額の登録対象としない。

CSVインポートなどでも、対象年月時点で無効な保有商品として扱う。

---

### 12.5 過去対象年月への影響

保有商品の有効性判定は、単純な現在状態だけではなく、各APIで定める対象年月基準のルールに従う。

HLD-005自体は、既存の商品別月末評価額を変更しない。

---

### 12.6 無効化済み保有商品

すでに

```text
holding_assets.deleted_at IS NOT NULL
```

となっている保有商品を指定した場合は、無効化対象として取得しない。

通常の有効な保有商品検索と同じ条件で対象を取得し、対象が存在しないものとして扱う。

概念的には、

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

とする。

無効化済み保有商品に対して再度成功レスポンスを返す方式は採用しない。

---

### 12.7 再有効化しない

Phase1では、無効化した保有商品を再有効化する機能を提供しない。

以下のような処理は行わない。

```text
deleted_at
    日時
      ↓
    NULL
```

同じ商品を再度管理する必要がある場合は、HLD-002 保有商品登録APIから新しい保有商品として登録する。

---

### 12.8 資産口座は無効化しない

保有商品の無効化によって、所属する

```text
asset_accounts
```

を無効化しない。

1つの資産口座に複数の保有商品が存在する場合、指定した保有商品のみを無効化する。

---

### 12.9 他の保有商品へ影響させない

同じ資産口座に属する他の保有商品は更新しない。

例えば、

```text
証券口座
├─ 全世界株式
└─ S&P500
```

という状態で`全世界株式`を無効化しても、

```text
S&P500
```

は有効なままとする。

---

### 12.10 確定済み月末資産状況が存在していても過去データを変更しない

対象保有商品を含む確定済みの月末資産状況が過去に存在していても、その月末資産状況や商品別月末評価額は変更しない。

保有商品の無効化は、現在以降の管理対象から外すための操作とする。

---

## 13. 処理フロー

概念的な処理フローは、以下とする。

```text
PATCH
/api/v1/holding-assets/{holdingAssetId}/disable
    ↓
X-User-Id検証
    ↓
holdingAssetId検証
    ↓
操作対象利用者に帰属する
有効な保有商品を取得
    ↓
存在確認
    ↓
無効化可否確認
    ↓
deleted_at更新
    ↓
成功レスポンス
```

---

### 13.1 利用者コンテキスト確認

`X-User-Id`から操作対象利用者を特定する。

利用者コンテキストが不正な場合は、保有商品検索へ進まない。

---

### 13.2 holdingAssetId検証

パスパラメータの

```text
holdingAssetId
```

が有効なID形式であることを確認する。

形式不正の場合は、保有商品検索へ進まない。

---

### 13.3 保有商品取得

操作対象利用者に帰属する有効な保有商品を取得する。

概念的には、

```text
holding_assets.id
    = holdingAssetId

AND

holding_assets.deleted_at
    IS NULL

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

---

### 13.4 対象不存在

該当する保有商品が存在しない場合は、

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

とする。

以下の場合も同じ扱いとする。

- 存在しない保有商品ID
- 他利用者の保有商品ID
- 論理削除済み保有商品
- 無効な資産口座配下の保有商品

これにより、他利用者のリソース存在有無を外部へ公開しない。

---

### 13.5 無効化

対象保有商品の

```text
deleted_at
```

へ現在日時を設定する。

LaravelのSoftDeletesを使用する場合は、概念的に

```php
$holdingAsset->delete();
```

によって無効化する。

物理DELETEは行わない。

---

### 13.6 関連データ非更新

保有商品無効化時に、以下の関連データを更新しない。

```text
asset_accounts
month_end_asset_snapshots
month_end_holding_values
```

HLD-005の更新対象は、指定した

```text
holding_assets
```

の1件のみとする。

---

## 14. 成功レスポンス

保有商品の無効化に成功した場合は、

```http
200 OK
```

を返却する。

正常時は、API共通の成功レスポンスEnvelopeを使用する。

概念例：

```json
{
  "data": {
    "id": "15",
    "disabled": true
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

---

### 14.1 レスポンス項目

正常時の`data`配下には、以下を返却する。

| 項目 | 型 | NULL | 内容 |
| --- | --- | :---: | --- |
| `id` | string | × | 無効化した保有商品ID |
| `disabled` | boolean | × | 無効化済みであることを示す。常に`true` |

---

### 14.2 id

無効化した保有商品のIDを文字列として返却する。

例：

```json
{
  "id": "15"
}
```

---

### 14.3 disabled

無効化処理が正常に完了したことを示す。

HLD-005の正常レスポンスでは、常に

```json
{
  "disabled": true
}
```

とする。

クライアントから`disabled`を指定して状態を切り替える用途には使用しない。

---

### 14.4 返却しない情報

HLD-005では、無効化結果の確認に不要な以下の情報は返却しない。

- `asset_accounts.id`
- `holding_assets.asset_account_id`
- `holding_assets.name`
- `holding_assets.product_type`
- `holding_assets.start_year_month`
- `holding_assets.memo`
- `holding_assets.created_at`
- `holding_assets.updated_at`
- `holding_assets.deleted_at`
- 過去の商品別月末評価額
- 月末資産状況

成功レスポンスは、無効化対象と処理結果を確認するための最小限の情報に限定する。

---

## 15. レスポンス項目

正常時の`data`配下には、以下の項目を返却する。

| 項目 | 型 | NULL | 内容 |
| --- | --- | :---: | --- |
| `id` | string | × | 無効化した保有商品ID |
| `disabled` | boolean | × | 無効化済みであることを示す。常に`true` |

---

### 15.1 id

無効化した保有商品のIDを文字列として返却する。

例：

```json
{
  "id": "15"
}
```

APIレスポンスでは、主キーを文字列として扱う共通方針に従う。

---

### 15.2 disabled

保有商品が無効化されたことを示す。

HLD-005の正常レスポンスでは、常に

```json
{
  "disabled": true
}
```

とする。

`disabled`は、クライアントから指定された状態をそのまま返却する項目ではない。

本APIの実行結果として無効化が完了したことを示す。

---

### 15.3 返却しない情報

HLD-005では、無効化結果の確認に不要な以下の情報は返却しない。

- `asset_account_id`
- `name`
- `product_type`
- `start_year_month`
- `memo`
- `created_at`
- `updated_at`
- `deleted_at`
- `balance_recording_unit`
- 過去の商品別月末評価額
- 月末資産状況

保有商品の詳細情報が必要な場合は、対応する参照APIを使用する。

---

## 16. エラーレスポンス

HLD-005では、API共通方針に従ったJSONエラーレスポンスを返却する。

主なエラーは、以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
| --- | --- | --- |
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`が指定されていない |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`の形式が不正 |
| `400 Bad Request` | `INVALID_HOLDING_ASSET_ID` | `holdingAssetId`の形式が不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 指定された利用者が存在しない、または論理削除済み |
| `404 Not Found` | `HOLDING_ASSET_NOT_FOUND` | 指定された保有商品が存在しない、他利用者に属する、または無効化済み |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバー内部エラー |

実際の独自エラーコード名は、API共通のエラーコード定義に合わせる。

---

### 16.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

HTTPステータスは、

```text
400 Bad Request
```

とする。

保有商品検索へ進まない。

---

### 16.2 INVALID_USER_ID

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

例えば、以下のような値を不正とする。

```text
0
-1
abc
1.5
```

---

### 16.3 INVALID_HOLDING_ASSET_ID

パスパラメータの

```text
holdingAssetId
```

の形式が不正な場合は、

```text
INVALID_HOLDING_ASSET_ID
```

を返却する。

HTTPステータスは、

```text
400 Bad Request
```

とする。

---

### 16.4 USER_NOT_FOUND

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

HTTPステータスは、

```text
404 Not Found
```

とする。

---

### 16.5 HOLDING_ASSET_NOT_FOUND

以下の場合は、

```text
HOLDING_ASSET_NOT_FOUND
```

を返却する。

- 指定された保有商品が存在しない
- 指定された保有商品が他利用者に属する
- 指定された保有商品が論理削除済み
- 所属資産口座が論理削除済み

HTTPステータスは、

```text
404 Not Found
```

とする。

他利用者のリソース存在有無を外部から判別できないよう、他利用者に属する場合も同じエラーとする。

---

### 16.6 INTERNAL_SERVER_ERROR

保有商品の無効化処理で想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

HTTPステータスは、

```text
500 Internal Server Error
```

とする。

レスポンスへ、以下の内部情報を含めない。

- SQL
- PostgreSQL内部エラー
- テーブル名
- カラム名
- 制約名
- PHP内部エラー
- Laravel内部例外メッセージ
- スタックトレース
- サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 16.7 エラーレスポンス形式

エラー時は、API共通のJSONエラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "HOLDING_ASSET_NOT_FOUND",
    "message": "指定された保有商品が存在しません。"
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

---

## 17. HTTPステータス

本APIで使用するHTTPステータスは、以下とする。

| HTTPステータス | 用途 |
| --- | --- |
| `200 OK` | 保有商品無効化成功 |
| `400 Bad Request` | 利用者コンテキストまたはパスパラメータ不正 |
| `404 Not Found` | 利用者または保有商品が存在しない |
| `500 Internal Server Error` | 想定外のサーバー内部エラー |

---

### 17.1 200 OK

指定された保有商品の無効化に成功した場合は、

```text
200 OK
```

を返却する。

---

### 17.2 400 Bad Request

以下の場合に使用する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
INVALID_HOLDING_ASSET_ID
```

---

### 17.3 404 Not Found

以下の場合に使用する。

```text
USER_NOT_FOUND
HOLDING_ASSET_NOT_FOUND
```

他利用者に属する保有商品を指定した場合も、

```text
HOLDING_ASSET_NOT_FOUND
```

として扱う。

---

### 17.4 500 Internal Server Error

想定外のサーバー内部エラーが発生した場合に使用する。

```text
INTERNAL_SERVER_ERROR
```

とする。

---

## 18. 副作用

本APIには、業務データに対する副作用がある。

正常終了した場合は、指定された保有商品の

```text
holding_assets.deleted_at
```

へ現在日時を設定する。

LaravelのSoftDeletesを使用する場合は、論理削除として扱う。

---

### 18.1 更新するテーブル

正常終了時に更新するテーブルは、

```text
holding_assets
```

のみとする。

更新対象は、指定された保有商品1件のみとする。

---

### 18.2 更新内容

概念的には、

```text
deleted_at
    NULL
      ↓
    現在日時
```

となる。

通常のEloquent更新に従い、

```text
updated_at
```

も更新される。

---

### 18.3 更新しないテーブル

HLD-005では、以下のテーブルを更新しない。

- `users`
- `asset_accounts`
- `asset_account_available_settings`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

過去の商品別月末評価額や月末資産状況を削除・変更しない。

---

### 18.4 物理削除しない

以下のような物理DELETEは行わない。

```sql
DELETE FROM holding_assets
WHERE id = ?;
```

過去データとの関連を保持するため、`deleted_at`による論理削除を使用する。

---

## 19. トランザクション

HLD-005では、更新対象が単一の`holding_assets`レコードであり、関連テーブルの更新を行わない。

そのため、Phase1ではHLD-005専用の明示的な`DB::transaction()`は必須としない。

概念的には、

```text
対象取得
    ↓
delete()
```

の単一更新で完結する。

---

### 19.1 将来的な拡張

将来的に保有商品無効化と同時に、

- 履歴テーブルへの登録
- 関連設定の更新
- 複数テーブルへの状態変更

などを行う場合は、トランザクション導入を検討する。

Phase1では、不要なトランザクションを追加しない。

---

## 20. ロック

Phase1では、HLD-005専用の明示的な行ロックは使用しない。

以下を基本的には使用しない。

```php
lockForUpdate()
```

無効化対象は1件であり、通常の論理削除処理として扱う。

---

### 20.1 同時更新

HLD-004 保有商品更新APIとHLD-005 保有商品無効化APIが、同一保有商品へ同時実行される可能性はある。

Phase1では、楽観ロック用のバージョンカラムなどは導入しない。

必要になった場合は、将来拡張として排他制御方式を検討する。

---

## 21. キャッシュ

Phase1では、HLD-005専用のサーバー側アプリケーションキャッシュを使用しない。

---

### 21.1 無効化後の参照

無効化後は、通常の有効な保有商品取得Queryから対象商品が除外される。

`deleted_at IS NULL`を前提とした取得処理によって反映する。

将来的に保有商品一覧等をキャッシュする場合は、HLD-005成功時に関連キャッシュの無効化が必要となる。

---

## 22. 冪等性

HLD-005は、同じ保有商品に対して複数回実行した場合、1回目と2回目でレスポンスが異なるため、APIとしては冪等な成功レスポンスを保証しない。

1回目の実行では、

```text
有効な保有商品
    ↓
無効化
    ↓
200 OK
```

となる。

同じ`holdingAssetId`で2回目を実行した場合は、すでに

```text
deleted_at IS NOT NULL
```

となっているため、通常の有効な保有商品として取得できない。

そのため、

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

とする。

---

### 22.1 データ状態としての冪等性

複数回リクエストされても、保有商品が複数回論理削除されたり、関連データが追加更新されたりすることはない。

最終的な業務状態は、

```text
無効化済み
```

のままとなる。

その意味では、データ状態としては同一結果に収束する。

---

### 22.2 無効化日時を再更新しない

すでに無効化済みの保有商品に対して、再度

```text
deleted_at = 現在日時
```

と更新し直さない。

最初に無効化した日時を保持する。

---

### 22.3 Idempotency-Key

Phase1では、

```text
Idempotency-Key
```

を使用しない。

HLD-005では、有効な保有商品だけを無効化対象として取得し、2回目以降は対象不存在として扱うことで重複更新を防止する。

---

### 22.4 フロントエンドでの二重送信

フロントエンドでは、HLD-005実行中に無効化ボタンを非活性化するなど、不要な二重送信を抑制してよい。

ただし、フロントエンドの制御だけを重複実行防止の保証とはしない。

バックエンド側でも、無効化済み保有商品を更新対象へ含めない。

---

## 23. 関連テーブル

HLD-005では、保有商品を無効化するため、以下のテーブルを使用する。

| テーブル | 用途 | 更新 |
| --- | --- | :---: |
| `users` | 操作対象利用者の確認 | × |
| `asset_accounts` | 保有商品の利用者境界確認 | × |
| `holding_assets` | 無効化対象の取得、論理削除 | ○ |

HLD-005では、`holding_assets`のみを更新する。

以下の関連テーブルは、無効化処理では更新しない。

- `asset_account_available_settings`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

---

### 23.1 users

`X-User-Id`で指定された操作対象利用者の存在確認に使用する。

概念的な条件は、以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者コンテキストの検証は、API共通Middlewareで行う。

HLD-005では、`users`を更新しない。

---

### 23.2 asset_accounts

指定された保有商品が操作対象利用者に帰属することを確認するために使用する。

概念的には、

```text
holding_assets.asset_account_id
    = asset_accounts.id

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

を満たす必要がある。

他利用者の資産口座に属する保有商品は、無効化対象として取得しない。

---

### 23.3 holding_assets

無効化対象となる保有商品を取得し、論理削除するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
| --- | --- |
| `id` | `holdingAssetId`との照合 |
| `asset_account_id` | 所属資産口座の確認 |
| `deleted_at` | 有効状態確認、論理削除 |
| `updated_at` | 無効化に伴う更新日時 |

概念的な取得条件は、以下とする。

```text
holding_assets.id
    = holdingAssetId

AND

holding_assets.deleted_at
    IS NULL
```

加えて、関連する`asset_accounts`を通して操作対象利用者との利用者境界を確認する。

---

### 23.4 無効化時の更新

正常終了時は、対象となる

```text
holding_assets
```

1件について、

```text
deleted_at
```

へ現在日時を設定する。

LaravelのSoftDeletesを使用する場合は、概念的に

```php
$holdingAsset->delete();
```

とする。

通常のEloquent更新に従い、

```text
updated_at
```

も更新される。

---

### 23.5 month_end_holding_values

HLD-005では、`month_end_holding_values`を更新・削除しない。

無効化対象の保有商品に過去の商品別月末評価額が存在する場合でも、その履歴を保持する。

概念的には、

```text
holding_assets
    ↓
論理削除

month_end_holding_values
    ↓
変更なし
```

とする。

---

### 23.6 month_end_asset_snapshots

HLD-005では、`month_end_asset_snapshots`を更新しない。

保有商品の無効化によって、

```text
confirmed
confirmed_at
```

を変更しない。

過去に確定済みの月末資産状況についても、状態を変更しない。

---

## 24. 関連する機能要件

HLD-005は、保有商品の無効化に関する機能要件と対応する。

主な関連要件は、以下とする。

- 保有商品
  - 利用中の保有商品を無効化できる
  - 保有商品を物理削除しない
  - 無効化後も過去データを保持する
  - 無効化後の保有商品を今後の記録対象から除外する
  - 無効化済み保有商品を再有効化しない
- 利用者境界
  - 操作対象利用者に属する保有商品のみ操作できる
  - 他利用者に属する保有商品は操作できない
- 月末資産
  - 過去の商品別月末評価額を保持する
  - 保有商品無効化によって過去の月末資産状況を変更しない
  - 保有商品無効化によって確定済み月末資産状況を未確定へ戻さない

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

## 25. インデックス

HLD-005では、主に以下の条件で保有商品を検索する。

```text
holding_assets.id
```

および、

```text
asset_accounts.user_id
```

を使用して利用者境界を確認する。

---

### 25.1 holding_assets.id

`holding_assets.id`は主キーであるため、主キーインデックスを使用する。

HLD-005専用として追加インデックスを作成しない。

---

### 25.2 asset_accounts.user_id

保有商品の利用者境界確認では、

```text
asset_accounts.user_id
```

を使用する。

既存の資産口座一覧取得などでも利用される検索条件であるため、必要なインデックスはテーブル全体の利用状況を踏まえて判断する。

HLD-005だけを理由として重複したインデックスを追加しない。

---

## 26. 性能

HLD-005は、単一の保有商品を無効化するAPIであるため、処理負荷は小さい。

概念的には、

```text
利用者コンテキスト確認
    ↓
保有商品1件取得
    ↓
論理削除1件
```

で完結する。

---

### 26.1 一覧検索を行わない

HLD-005では、保有商品一覧を取得してから対象を検索するような実装は行わない。

以下のような処理は避ける。

```text
保有商品一覧取得
    ↓
アプリケーション側で
holdingAssetId検索
```

対象IDと利用者境界を条件として直接1件取得する。

---

### 26.2 N+1問題

HLD-005は、単一リソースを対象とするため、N+1問題は発生しない。

利用者境界確認のために`asset_accounts`との関連を確認する場合も、1件取得で完結する構成とする。

---

### 26.3 不要な関連データを取得しない

無効化処理に不要な以下のデータは取得しない。

- 商品別月末評価額一覧
- 月末資産状況一覧
- 利用可能資産設定履歴

HLD-005で必要な最小限のデータだけを取得する。

---

## 27. セキュリティ

HLD-005では、操作対象利用者と保有商品の利用者境界を必ず保証する。

---

### 27.1 他利用者の保有商品

他利用者に帰属する`holdingAssetId`が指定された場合でも、そのリソースの存在を外部へ公開しない。

以下として扱う。

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

---

### 27.2 パスパラメータを信用しない

`holdingAssetId`だけを条件として保有商品を取得してはならない。

以下のような取得は避ける。

```php
HoldingAsset::find(
    $holdingAssetId,
);
```

利用者境界を含めて対象を取得する。

概念的には、

```text
holdingAssetId
+
操作対象利用者ID
+
deleted_at IS NULL
```

を条件とする。

---

### 27.3 Mass Assignmentを使用しない

本APIでは、リクエストボディを使用しない。

そのため、以下のようなリクエスト値による一括更新を行わない。

```php
$holdingAsset->update(
    $request->all(),
);
```

無効化操作は、サーバー側で明示的にSoftDeletesを実行する。

---

### 27.4 物理削除を防止する

過去データとの関連を維持する必要があるため、物理DELETEを行わない。

Laravelでは、`SoftDeletes`を利用する。

---

### 27.5 内部情報を公開しない

エラー時に、以下の情報をレスポンスへ含めない。

- SQL
- テーブル名
- カラム名
- PostgreSQL内部エラー
- Laravel内部例外
- サーバーファイルパス
- スタックトレース

---

## 28. ログ・監視

HLD-005では、API共通ログ方針に従って無効化処理の結果を記録する。

---

### 28.1 ログコンテキスト

必要に応じて、以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
holdingAssetId
httpStatus
errorCode
```

`apiId`は、

```text
HLD-005
```

とする。

---

### 28.2 正常時

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = HLD-005
holdingAssetId
httpStatus = 200
```

保有商品名や過去の商品別月末評価額などを不要にログへ出力しない。

---

### 28.3 異常時

異常時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId
holdingAssetId
errorCode
httpStatus
```

想定外例外については、調査に必要な情報をサーバーログへ記録する。

---

## 29. 設計上の補足

### 29.1 `/disable`を採用する理由

HLD-004 保有商品更新とHLD-005 保有商品無効化は、どちらも`PATCH`を使用する。

HLD-004は、

```http
PATCH /api/v1/holding-assets/{holdingAssetId}
```

で、保有商品名、商品種別、備考などの通常属性を更新する。

一方、HLD-005は、

```http
PATCH /api/v1/holding-assets/{holdingAssetId}/disable
```

として、保有商品を無効化する業務操作を明示する。

これにより、

```text
通常更新
    → /holding-assets/{holdingAssetId}

無効化
    → /holding-assets/{holdingAssetId}/disable
```

と責務を分離する。

---

### 29.2 HLD-004と同じURLを使用しない理由

HLD-004とHLD-005を以下のようにすると、

```text
HLD-004
PATCH /api/v1/holding-assets/{holdingAssetId}

HLD-005
PATCH /api/v1/holding-assets/{holdingAssetId}
```

HTTPメソッドとURLが完全に一致する。

この場合、Laravelのルーティングではリクエストだけからどちらの業務操作を実行するかを判別できない。

リクエストボディの内容によって

```text
通常更新
無効化
```

を切り替える方式も採用せず、URLレベルで操作を明確に分離する。

---

### 29.3 DELETEを採用しない理由

HLD-005は、保有商品を今後の管理対象から外す操作である。

データベース上ではLaravelのSoftDeletesを使用するため、技術的には

```http
DELETE /api/v1/holding-assets/{holdingAssetId}
```

とする選択肢もある。

ただし、本システムでは保有商品の過去データを保持しながら「無効化」という業務概念として扱う。

そのため、Phase1では

```http
PATCH /api/v1/holding-assets/{holdingAssetId}/disable
```

を採用し、物理削除を想起させる`DELETE`とは分離する。

---

### 29.4 PATCHを採用する理由

HLD-005では、保有商品リソースそのものを消去するのではなく、

```text
有効
    ↓
無効化済み
```

へ状態を変更する。

実装上は、

```text
deleted_at
    NULL
      ↓
    現在日時
```

という部分的な状態変更となる。

そのため、HTTPメソッドには`PATCH`を使用する。

---

### 29.5 `/disable`で操作を明示する理由

HLD-005は、単なる属性変更ではなく、

```text
今後の管理対象から外す
```

という業務上の意味を持つ。

そのため、

```text
PATCH /holding-assets/{id}
+
{
  "isEnabled": false
}
```

よりも、

```text
PATCH /holding-assets/{id}/disable
```

とすることで、API利用側からも操作意図を理解しやすくする。

---

### 29.6 リクエストボディを使用しない理由

`/disable`というエンドポイント自体が無効化操作を表している。

そのため、

```json
{
  "isEnabled": false
}
```

のような状態指定を追加すると、同じ意味を

```text
URL
+
リクエストボディ
```

の双方で重複表現することになる。

Phase1では、無効化対象となる`holdingAssetId`だけを指定し、リクエストボディは使用しない。

---

### 29.7 `isEnabled`カラムを追加しない理由

HLD-005のためだけに、

```text
is_enabled
enabled
disabled
status
```

などの状態カラムを追加しない。

保有商品の無効化は、既存の

```text
deleted_at
```

によるSoftDeletesで表現する。

これにより、Laravel標準の論理削除機構を利用できる。

---

### 29.8 SoftDeletesを無効化表現として使用する理由

保有商品は、無効化後も過去の商品別月末評価額から参照される。

そのため、物理削除すると過去データとの関連を維持できなくなる可能性がある。

`deleted_at`による論理削除を使用することで、

```text
通常の業務対象
    → 除外

過去データ
    → 保持
```

を両立する。

---

### 29.9 過去の商品別月末評価額を削除しない理由

保有商品を無効化しても、過去にその商品を保有していた事実は変わらない。

そのため、

```text
month_end_holding_values
```

を削除・更新しない。

過去の資産状況や資産推移を正しく再現できる状態を維持する。

---

### 29.10 過去の月末資産状況を変更しない理由

保有商品を無効化した時点で、過去に確定済みの月末資産状況まで変更すると、過去時点の資産状態が現在の操作によって変化してしまう。

そのため、HLD-005では

```text
month_end_asset_snapshots
month_end_holding_values
```

を変更しない。

---

### 29.11 確定済み月末資産状況があっても無効化できる理由

HLD-005は、過去データを変更する操作ではなく、今後の管理対象から保有商品を外す操作である。

そのため、過去に確定済みの月末資産状況が存在すること自体を無効化禁止条件とはしない。

過去の確定状態を保持したまま、保有商品だけを無効化する。

---

### 29.12 再有効化APIを用意しない理由

Phase1では、無効化済み保有商品を再有効化する機能を提供しない。

再有効化を許可すると、以下の追加ルールが必要になる。

* 過去の無効化期間をどう扱うか
* 対象年月時点の有効性をどう判定するか
* 過去の同名商品との関係をどう扱うか

Phase1では、同じ商品を再度管理する場合はHLD-002で新規登録する方式とする。

---

### 29.13 無効化済み保有商品への再実行を404とする理由

HLD-005では、通常の有効な保有商品のみを操作対象として取得する。

そのため、すでに

```text
deleted_at IS NOT NULL
```

の保有商品は、通常の操作対象として存在しないものとして扱う。

再実行時に再度成功レスポンスを返すと、最初の無効化がいつ実行されたのかという状態とAPI実行結果の意味が曖昧になる。

Phase1では、

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

とする。

---

### 29.14 他利用者の保有商品も404とする理由

他利用者に属する`holdingAssetId`が指定された場合に、

```text
FORBIDDEN
OTHER_USER_RESOURCE
```

などを返すと、そのIDの保有商品が実際に存在することを外部から推測できる。

そのため、

```text
存在しない
他利用者に属する
無効化済み
```

を同じ

```text
HOLDING_ASSET_NOT_FOUND
```

として扱う。

---

### 29.15 HLD-004で無効化させない理由

HLD-004の責務は、以下の通常属性更新である。

* 保有商品名
* 商品種別
* 備考

HLD-004に

```text
isEnabled
disabled
deletedAt
```

などを追加すると、通常更新とライフサイクル変更が同じAPIへ混在する。

そのため、無効化はHLD-005へ分離する。

---

### 29.16 HLD-005で通常属性を変更させない理由

HLD-005は、保有商品の無効化だけを行う。

無効化時に、

```text
name
product_type
memo
start_year_month
```

なども同時変更できるようにはしない。

複数の業務操作を1リクエストへ混在させず、

```text
通常属性変更
    → HLD-004

無効化
    → HLD-005
```

とする。

---

### 29.17 専用UseCaseを分ける理由

Laravel実装では、

```text
UpdateHoldingAssetUseCase
DisableHoldingAssetUseCase
```

を分離する。

無効化を通常更新UseCaseの特殊ケースとして実装すると、

```text
if disabled
if update
```

のように処理分岐が増える。

業務操作単位でUseCaseを分けることで、責務を明確にする。

---

### 29.18 FormRequestを作成しない理由

HLD-005には、リクエストボディが存在しない。

専用FormRequestを作成しても、検証対象となるボディ項目がないため、空クラスになる。

Phase1では、実装クラスを増やすこと自体を目的とせず、必要な責務が存在する場合だけクラスを作成する。

---

### 29.19 QueryとRepositoryを分ける理由

HLD-005では、

```text
Query
    → 操作対象保有商品の取得

Repository
    → SoftDeleteによる更新
```

と責務を分ける。

これにより、

```text
取得条件
利用者境界
```

と、

```text
DB更新
```

を分離する。

HLD-001〜HLD-004でも同じ責務分離方針を採用している場合は、その構成へ統一する。

---

### 29.20 `delete()`を使用する理由

LaravelのSoftDeletesを使用するため、無効化処理では

```php
$holdingAsset->delete();
```

を使用する。

`deleted_at`を直接更新する方式ではなく、Laravel標準の論理削除処理へ統一する。

これにより、SoftDeletesを利用していることが実装からも明確になる。

---

### 29.21 明示的なトランザクションを使用しない理由

Phase1のHLD-005では、更新対象は

```text
holding_assets 1件
```

のみである。

関連テーブルの更新もなく、単一のSoftDeleteで完結する。

そのため、明示的な

```php
DB::transaction()
```

は使用しない。

複数テーブル更新が必要になった場合に、改めて導入を検討する。

---

### 29.22 明示的なロックを使用しない理由

HLD-005は、単一保有商品の通常の論理削除として扱う。

Phase1では、

```text
lockForUpdate()
version
ETag
If-Match
```

などの排他制御は導入しない。

HLD-004との同時更新が実際に問題となる場合に、競合制御方式を改めて検討する。

---

### 29.23 `disabled`をレスポンスへ返す理由

HLD-005では、成功した操作結果をクライアントが明示的に確認できるよう、

```text
disabled = true
```

を返却する。

一方で、`deletedAt`というデータベース実装詳細は返却しない。

APIとしては、

```text
無効化された
```

という業務上の結果だけを公開する。

---

### 29.24 `deletedAt`を返却しない理由

`deleted_at`は、SoftDeletesを実現するためのデータベース内部表現である。

フロントエンドが必要とするのは、

```text
無効化に成功したか
```

という結果であり、DB上の論理削除日時そのものではない。

そのため、

```text
disabled = true
```

を返し、`deletedAt`は返却しない。

---

### 29.25 無効化済み商品を通常一覧へ返さない理由

HLD-001が現在利用中の保有商品一覧を取得するAPIである場合、論理削除済み商品は通常一覧へ含めない。

HLD-005成功後に一覧を再取得すると、対象商品が表示されなくなるシンプルな挙動とする。

無効化済み商品一覧が将来的に必要になった場合は、別途参照方法を検討する。

---

### 29.26 Optimistic Updateを必須としない理由

保有商品の無効化は、高頻度で連続操作する機能ではない。

また、成功後は通常一覧から商品自体が消えるため、Optimistic Updateのロールバック処理を追加するより、

```text
API成功
    ↓
Query invalidate
    ↓
再取得
```

という単純な方式をPhase1では基本とする。

---

### 29.27 Idempotency-Keyを採用しない理由

HLD-005は、同じ保有商品を2回無効化できない。

1回目で

```text
deleted_at IS NOT NULL
```

となり、2回目は有効な対象として取得されない。

そのため、Phase1ではIdempotency-Key管理を追加しない。

---

### 29.28 無効化日時を再更新しない理由

無効化済み商品への再実行時に、

```text
deleted_at = now()
```

と再更新すると、実際に最初に無効化した日時が失われる。

そのため、無効化済み商品は更新対象へ含めず、最初の`deleted_at`を保持する。

---

### 29.29 関連APIへの影響

HLD-005成功後は、通常の保有商品参照APIでは対象商品を取得できなくなる。

主に以下へ影響する。

```text
HLD-001
保有商品一覧取得

HLD-003
保有商品詳細取得
```

一方、過去の商品別月末評価額や確定済み月末資産状況を参照するAPIでは、履歴再現のため論理削除済み保有商品との関連を必要に応じて保持する。

---

### 29.30 CSVインポートへの影響

商品別月末評価額CSVでは、対象年月時点で記録対象となる保有商品かどうかを判定する。

HLD-005によって現在無効化されていることだけを理由に、過去対象年月の評価額まで一律に無効としない。

具体的な対象年月判定は、CSV-005・CSV-006側の業務ルールに従う。

---

### 29.31 無効化APIと対象年月判定を分離する理由

HLD-005の責務は、保有商品を現在の管理対象から外すことである。

一方、

```text
ある過去年月に
その商品が記録対象だったか
```

という判定は、月末資産・CSVインポート側の責務である。

HLD-005へ過去年月判定のロジックまで持たせない。

---

### 29.32 Phase1では設計を広げすぎない

HLD-005では、以下のような機能は追加しない。

* 無効化理由
* 無効化予定日
* 再有効化
* 無効化履歴専用テーブル
* ステータス遷移履歴
* Idempotency-Key
* 楽観ロック

Phase1では、

```text
保有商品を無効化する
+
過去データを保持する
```

という要件を満たす最小構成とする。

---

## 30. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)