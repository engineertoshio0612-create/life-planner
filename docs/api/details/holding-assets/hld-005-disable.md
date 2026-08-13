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

## 29. テスト観点

HLD-005では、正常な無効化だけでなく、以下を重点的に確認する。

- 利用者境界
- 論理削除
- 関連データ非更新
- 再実行
- HLD-004との責務分離

---

### 29.1 正常系

操作対象利用者に属する有効な保有商品を指定する。

以下を確認する。

- `200 OK`となること
- `data.id`が対象保有商品IDとなること
- `data.disabled = true`となること
- `holding_assets.deleted_at`が設定されること
- `holding_assets.updated_at`が更新されること
- `holding_assets.name`が変更されないこと
- `holding_assets.product_type`が変更されないこと
- `holding_assets.start_year_month`が変更されないこと
- `holding_assets.memo`が変更されないこと

---

### 29.2 X-User-Id未指定

`X-User-Id`を指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

以下を確認する。

- 保有商品検索へ進まないこと
- `deleted_at`が更新されないこと

---

### 29.3 X-User-Id形式不正

例えば、

```http
X-User-Id: abc
```

を指定する。

期待結果：

```text
400 Bad Request
INVALID_USER_ID
```

業務データが更新されないこと。

---

### 29.4 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 29.5 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 29.6 holdingAssetId形式不正

不正な`holdingAssetId`を指定する。

例えば、

```text
abc
-1
0
1.5
```

などを確認する。

期待結果：

```text
400 Bad Request
INVALID_HOLDING_ASSET_ID
```

となること。

---

### 29.7 保有商品不存在

存在しない`holdingAssetId`を指定する。

期待結果：

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

となること。

---

### 29.8 他利用者の保有商品

以下の状態を用意する。

```text
User A
    Holding Asset A

User B
    Holding Asset B
```

User AとしてHolding Asset BのIDを指定する。

期待結果：

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

となること。

User Bの保有商品が更新されないこと。

---

### 29.9 論理削除済み保有商品

すでに

```text
deleted_at IS NOT NULL
```

の保有商品を指定する。

期待結果：

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

となること。

`deleted_at`が新しい日時へ更新されないことも確認する。

---

### 29.10 所属資産口座が論理削除済み

保有商品自体は論理削除されていないが、所属資産口座が論理削除済みの状態を用意する。

対象が通常の操作対象として取得されないことを確認する。

期待結果：

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

とする。

---

### 29.11 SoftDeletes

正常系で、物理DELETEではなく論理削除となることを確認する。

実行後も、

```text
holding_assets
```

レコード自体はデータベースに存在すること。

また、

```text
deleted_at IS NOT NULL
```

となること。

---

### 29.12 過去の商品別月末評価額

対象保有商品に過去の

```text
month_end_holding_values
```

が存在する状態でHLD-005を実行する。

以下を確認する。

- 過去レコードが削除されないこと
- `value`が変更されないこと
- `month_end_asset_snapshot_id`が変更されないこと
- `holding_asset_id`が変更されないこと

---

### 29.13 月末資産状況

対象保有商品に関連する月末資産状況が存在する状態でHLD-005を実行する。

以下を確認する。

- snapshotが削除されないこと
- `confirmed`が変更されないこと
- `confirmed_at`が変更されないこと

---

### 29.14 確定済み月末資産状況

過去に

```text
confirmed = true
```

の月末資産状況が存在する状態で保有商品を無効化する。

無効化自体は正常に実行できることを確認する。

また、確定済み月末資産状況が変更されないことを確認する。

---

### 29.15 同じ資産口座の他保有商品

同一資産口座に複数の保有商品を用意する。

例えば、

```text
証券口座
├─ 全世界株式
└─ S&P500
```

`全世界株式`だけを無効化する。

以下を確認する。

- `全世界株式.deleted_at`が設定されること
- `S&P500.deleted_at`は変更されないこと
- 資産口座自体は変更されないこと

---

### 29.16 再実行

有効な保有商品について、HLD-005を2回実行する。

1回目：

```text
200 OK
```

2回目：

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

となることを確認する。

1回目に設定された`deleted_at`が2回目によって変更されないことも確認する。

---

### 29.17 リクエストボディなし

以下のリクエストで正常に無効化できることを確認する。

```http
PATCH /api/v1/holding-assets/15/disable
X-User-Id: 1
Accept: application/json
```

リクエストボディを必須としないことを確認する。

---

### 29.18 isEnabledを必要としない

旧仕様のような、

```json
{
  "isEnabled": false
}
```

を必要としないことを確認する。

無効化操作は、

```text
/disable
```

エンドポイント自体で表現されていることを確認する。

---

### 29.19 HLD-004とのエンドポイント分離

以下の2APIが正しく別ルートとして登録されていることを確認する。

```text
HLD-004

PATCH
/api/v1/holding-assets/{holdingAssetId}
```

```text
HLD-005

PATCH
/api/v1/holding-assets/{holdingAssetId}/disable
```

HLD-004とHLD-005が同じHTTPメソッド・同じURLとして競合しないことを確認する。

---

### 29.20 HLD-004との責務分離

HLD-005実行時に、以下のHLD-004で扱う項目を変更しないことを確認する。

- `name`
- `product_type`
- `memo`

逆に、HLD-004の通常更新によって`deleted_at`を任意に変更できないことも別途確認する。

---

### 29.21 正常レスポンス契約

正常時に、概念的に以下の形式となることを確認する。

```json
{
  "data": {
    "id": "15",
    "disabled": true
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

以下を確認する。

- `data.id`がstringであること
- `data.disabled`がbooleanであること
- `data.disabled = true`であること
- `requestId`が設定されること

---

### 29.22 返却しない情報

正常レスポンスへ、以下が含まれないことを確認する。

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

---

### 29.23 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
INVALID_HOLDING_ASSET_ID
USER_NOT_FOUND
HOLDING_ASSET_NOT_FOUND
INTERNAL_SERVER_ERROR
```

以下も確認する。

- `error.code`が設定されること
- `error.message`が設定されること
- `requestId`が設定されること
- SQLが含まれないこと
- スタックトレースが含まれないこと
- サーバーファイルパスが含まれないこと

---

### 29.24 副作用範囲

HLD-005実行前後で、変更される業務データが

```text
対象holding_assets 1件
```

だけであることを確認する。

以下が変更されないことを確認する。

- `asset_accounts`
- `asset_account_available_settings`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

---

### 29.25 INTERNAL_SERVER_ERROR

無効化処理中に想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

- 内部情報がレスポンスへ公開されないこと
- 想定外例外の詳細がサーバーログへ記録されること

---

## 30. Laravel実装方針

HLD-005では、Action、UseCase、Query、Repository、DTO、API Resource、Responderを分離して実装する。

概念的な処理構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ HoldingAssetQuery
    └─ HoldingAssetRepository
    ↓
Disable Result DTO
    ↓
API Resource
    ↓
Responder
```

本APIは、リクエストボディを使用しないため、HLD-005専用のFormRequestは作成しない。

また、更新対象は

```text
holding_assets.deleted_at
```

のみとし、HLD-004で扱う通常更新処理とは分離する。

---

### 30.1 Route

HLD-005は、以下のルートとして定義する。

概念例：

```php
Route::patch(
    '/api/v1/holding-assets/{holdingAssetId}/disable',
    DisableHoldingAssetAction::class,
);
```

HLD-004は、

```php
Route::patch(
    '/api/v1/holding-assets/{holdingAssetId}',
    UpdateHoldingAssetAction::class,
);
```

とし、同一HTTPメソッドでもURLを分離する。

これにより、

```text
通常更新
    → HLD-004

無効化
    → HLD-005
```

という責務をルーティングレベルでも明確にする。

---

### 30.2 Action

Actionは、パスパラメータの

```text
holdingAssetId
```

と、検証済みの利用者コンテキストを取得し、UseCaseを呼び出す。

概念例：

```php
final class DisableHoldingAssetAction
{
    public function __invoke(
        string $holdingAssetId,
        DisableHoldingAssetUseCase $useCase,
        DisableHoldingAssetResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                holdingAssetId:
                    $holdingAssetId,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

Actionでは、以下を行わない。

* 利用者存在確認
* `holdingAssetId`の業務的な存在確認
* 保有商品検索SQLの組み立て
* 利用者境界判定
* `deleted_at`更新
* SoftDeletes実行
* レスポンス配列生成
* JSON変換

Actionは、UseCase呼び出しとResponderへの受け渡しに責務を限定する。

---

### 30.3 FormRequest

HLD-005専用のFormRequestは作成しない。

本APIでは、

```text
パスパラメータ
    holdingAssetId

リクエストボディ
    なし
```

であり、HLD-005固有のボディ入力値が存在しないためである。

以下のような空のFormRequestは作成しない。

```php
final class DisableHoldingAssetRequest
    extends FormRequest
{
}
```

不要なクラスを追加しない。

---

### 30.4 パスパラメータ検証

`holdingAssetId`の形式検証は、ルート制約または共通のパスパラメータ検証方式で行う。

概念例：

```php
->whereNumber(
    'holdingAssetId',
);
```

ただし、`0`や負数を有効なIDとして扱わない。

共通方針として正の整数IDのみを許可する場合は、そのルールへ統一する。

形式不正の場合は、

```text
INVALID_HOLDING_ASSET_ID
```

へ変換する。

---

### 30.5 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* 共通例外処理
* ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、以下とする。

```text
X-User-Id取得
    ↓
必須チェック
    ↓
形式チェック
    ↓
users存在確認
    ↓
UserContext設定
    ↓
Action
```

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 30.6 UseCase

HLD-005の業務処理全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `holdingAssetId`を受け取る
3. 操作対象利用者に属する有効な保有商品を取得する
4. 存在しない場合は業務例外とする
5. Repositoryへ無効化処理を委譲する
6. Disable Result DTOを返す

概念的には、以下とする。

```text
userId
+
holdingAssetId
    ↓
HoldingAssetQuery
    ↓
対象保有商品取得
    ↓
存在確認
    ↓
HoldingAssetRepository
    ↓
論理削除
    ↓
Disable Result DTO
```

UseCaseでは、HTTPレスポンスを生成しない。

---

### 30.7 HoldingAssetQuery

無効化対象となる有効な保有商品を取得するために使用する。

概念例：

```php
$holdingAsset =
    $this->holdingAssetQuery
        ->findActiveByIdAndUser(
            holdingAssetId:
                $holdingAssetId,

            userId:
                $userId,
        );
```

概念的な検索条件は、以下とする。

```text
holding_assets.id
    = holdingAssetId

AND

holding_assets.deleted_at
    IS NULL

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 30.8 利用者境界をQueryへ含める

以下のように、保有商品だけをIDで取得した後に別処理で利用者判定する方式は基本としない。

```php
$holdingAsset =
    HoldingAsset::find(
        $holdingAssetId,
    );
```

対象取得時点から、

```text
holdingAssetId
+
userId
+
有効状態
```

を条件へ含める。

これにより、他利用者の保有商品を誤って更新することを防止する。

---

### 30.9 EloquentのSoftDeletes

`HoldingAsset` Modelでは、Laravelの

```php
use SoftDeletes;
```

を使用する。

概念例：

```php
final class HoldingAsset
    extends Model
{
    use SoftDeletes;
}
```

HLD-005では、物理DELETEではなくSoftDeletesによって無効化する。

---

### 30.10 QueryではwithTrashedを使用しない

HLD-005の通常取得では、

```php
withTrashed()
```

を使用しない。

論理削除済み保有商品は、無効化対象として取得しない。

そのため、

```text
deleted_at IS NULL
```

の通常スコープを利用する。

---

### 30.11 無効化済み保有商品の扱い

論理削除済みの保有商品を指定した場合は、Query結果が`null`となる。

UseCaseでは、

```php
if ($holdingAsset === null) {
    throw new HoldingAssetNotFoundException();
}
```

のように扱う。

再度`delete()`を実行しない。

これにより、最初に設定された

```text
deleted_at
```

を保持する。

---

### 30.12 HoldingAssetRepository

保有商品の更新処理はRepositoryへ委譲する。

HLD-005では、無効化専用メソッドを用意する。

概念例：

```php
interface HoldingAssetRepository
{
    public function disable(
        HoldingAsset $holdingAsset,
    ): void;
}
```

実装例：

```php
final class EloquentHoldingAssetRepository
    implements HoldingAssetRepository
{
    public function disable(
        HoldingAsset $holdingAsset,
    ): void {
        $holdingAsset->delete();
    }
}
```

Repositoryでは、業務上の対象判定は行わない。

対象取得・利用者境界確認は、UseCaseおよびQuery側で完了していることを前提とする。

---

### 30.13 delete()を使用する

SoftDeletesを利用するため、HLD-005では概念的に以下を使用する。

```php
$holdingAsset->delete();
```

以下のように`deleted_at`を直接更新する実装は、基本としない。

```php
$holdingAsset->update([
    'deleted_at' => now(),
]);
```

Laravel標準のSoftDeletes動作へ統一する。

---

### 30.14 forceDelete()を使用しない

HLD-005では、以下を使用しない。

```php
$holdingAsset->forceDelete();
```

物理削除すると、過去の

```text
month_end_holding_values
```

との関連が失われる可能性があるためである。

---

### 30.15 HLD-004の更新処理を流用しない

HLD-005では、HLD-004の更新UseCaseへ

```text
isEnabled = false
```

のような値を渡して無効化する方式は採用しない。

概念的には、

```text
HLD-004
UpdateHoldingAssetUseCase

HLD-005
DisableHoldingAssetUseCase
```

として分離する。

これにより、

```text
通常属性更新
無効化
```

の業務操作を明確に区別する。

---

### 30.16 Mass Assignmentを使用しない

HLD-005ではリクエストボディが存在しないため、以下のような処理を行わない。

```php
$holdingAsset->update(
    $request->all(),
);
```

無効化処理は、

```php
$holdingAsset->delete();
```

と明示する。

---

### 30.17 過去の商品別月末評価額を操作しない

RepositoryまたはUseCaseから、

```text
month_end_holding_values
```

を更新・削除しない。

以下のようなcascade的な業務処理は行わない。

```text
HoldingAsset無効化
    ↓
過去MonthEndHoldingValue削除
```

過去データは保持する。

---

### 30.18 snapshotを操作しない

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

保有商品の無効化を理由として、確定済みsnapshotを未確定へ戻さない。

---

### 30.19 トランザクション

Phase1では、HLD-005専用の明示的な

```php
DB::transaction()
```

は使用しない。

更新対象が

```text
holding_assets 1件
```

だけであり、SoftDeletesの単一UPDATEで完結するためである。

概念的には、

```text
SELECT
    ↓
UPDATE deleted_at
```

となる。

---

### 30.20 トランザクションを導入しない理由

HLD-005でトランザクションを導入しても、

```text
複数テーブル更新
複数レコード更新
```

が存在しないため、Phase1では得られる利点が小さい。

不要な以下を増やさない。

* トランザクション境界
* ロック保持
* 実装複雑性

将来的に複数テーブルを更新する場合は、改めて導入を検討する。

---

### 30.21 ロック

Phase1では、HLD-005専用の`lockForUpdate()`は使用しない。

以下のような明示的な行ロックは基本としない。

```php
HoldingAsset::query()
    ->whereKey(
        $holdingAssetId,
    )
    ->lockForUpdate()
    ->first();
```

単一レコードの通常の論理削除として扱う。

---

### 30.22 HLD-004との同時実行

HLD-004とHLD-005が同じ保有商品へ同時実行される可能性はある。

Phase1では、以下は導入しない。

* `version`カラム
* 楽観ロック
* ETag
* If-Match

必要になった時点で排他制御方針を別途検討する。

---

### 30.23 Disable Result DTO

無効化結果は、専用DTOとして表現する。

概念例：

```php
final readonly class DisableHoldingAssetResult
{
    public function __construct(
        public int $id,
    ) {
    }
}
```

`disabled`はHLD-005成功時に必ず`true`となるため、DTOへ保持せずResource側で固定値として設定してもよい。

または、レスポンス契約をDTOへ明示的に持たせる場合は、

```php
final readonly class DisableHoldingAssetResult
{
    public function __construct(
        public int $id,
        public bool $disabled,
    ) {
    }
}
```

としてもよい。

Phase1では、どちらかに統一する。

---

### 30.24 UseCaseの戻り値

概念例：

```php
return new DisableHoldingAssetResult(
    id:
        $holdingAsset->id,
);
```

Eloquent ModelそのものをActionやResponderへ返却しない。

---

### 30.25 API Resource

Disable Result DTOを、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php
final class DisableHoldingAssetResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'disabled'
                => true,
        ];
    }
}
```

主キーは、API共通方針に従ってstringへ変換する。

---

### 30.26 Resourceで返却しない情報

HLD-005のResourceでは、以下を返却しない。

* `asset_account_id`
* `name`
* `product_type`
* `start_year_month`
* `memo`
* `created_at`
* `updated_at`
* `deleted_at`

Eloquent Modelを

```php
return $holdingAsset->toArray();
```

のようにそのまま返却しない。

---

### 30.27 Responder

Responderは、Disable Result DTOをAPI共通の成功Envelopeへ変換する。

概念例：

```php
final class DisableHoldingAssetResponder
{
    public function ok(
        DisableHoldingAssetResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new DisableHoldingAssetResource(
                        $result,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの共通項目は、API共通レスポンス処理に従う。

---

### 30.28 Responderで行わないこと

Responderでは、以下を行わない。

* 保有商品検索
* 利用者境界判定
* 無効化可否判定
* SoftDeletes
* `deleted_at`更新
* DBアクセス

Responderは、生成済み結果をHTTPレスポンスへ変換することに責務を限定する。

---

### 30.29 例外変換

主な例外変換は、以下とする。

| 内部状態                 | 独自エラーコード                   |
| -------------------- | -------------------------- |
| `X-User-Id`未指定       | `USER_CONTEXT_REQUIRED`    |
| `X-User-Id`形式不正      | `INVALID_USER_ID`          |
| 利用者不存在               | `USER_NOT_FOUND`           |
| `holdingAssetId`形式不正 | `INVALID_HOLDING_ASSET_ID` |
| 対象保有商品不存在            | `HOLDING_ASSET_NOT_FOUND`  |
| 他利用者の保有商品            | `HOLDING_ASSET_NOT_FOUND`  |
| 無効化済み保有商品            | `HOLDING_ASSET_NOT_FOUND`  |
| 想定外例外                | `INTERNAL_SERVER_ERROR`    |

他利用者のリソースであることを示す専用エラーコードは返却しない。

---

### 30.30 HoldingAssetNotFoundException

対象保有商品を取得できない場合は、共通またはHLD系の業務例外を送出する。

概念例：

```php
if ($holdingAsset === null) {
    throw new HoldingAssetNotFoundException();
}
```

最終的に、

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

へ変換する。

---

### 30.31 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

* SQL
* テーブル名
* カラム名
* PostgreSQL内部エラー
* PHP内部エラー
* Laravel内部例外メッセージ
* スタックトレース
* サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 30.32 ログ

HLD-005では、必要に応じて以下をログコンテキストへ設定する。

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

保有商品名や過去の商品別月末評価額などを不要にログへ出力しない。

---

### 30.33 キャッシュ

Phase1では、HLD-005専用のサーバー側アプリケーションキャッシュを使用しない。

無効化後は、SoftDeletesの通常スコープによって対象保有商品が有効一覧・詳細取得対象から除外される。

---

### 30.34 テスト実装方針

Laravel側では、Feature Testを中心としてHLD-005のAPI契約と無効化処理を確認する。

主に以下を確認する。

* `200 OK`
* `400 Bad Request`
* `404 Not Found`
* `500 Internal Server Error`
* `X-User-Id`必須
* `X-User-Id`形式検証
* 利用者存在確認
* `holdingAssetId`形式検証
* 保有商品存在確認
* 利用者境界
* SoftDeletes
* 過去データ非更新
* snapshot非更新
* 再実行
* HLD-004とのルート分離
* リクエストボディ不要
* レスポンス契約

---

### 30.35 QueryのDatabase Test

`HoldingAssetQuery`について、以下を確認する。

```text
holdingAssetId一致
+
操作対象利用者一致
+
holding_assets.deleted_at IS NULL
+
asset_accounts.deleted_at IS NULL
    ↓
取得できる
```

また、以下を取得できないことを確認する。

* 他利用者の保有商品
* 論理削除済み保有商品
* 論理削除済み資産口座配下の保有商品
* 存在しない保有商品

---

### 30.36 RepositoryのDatabase Test

`HoldingAssetRepository::disable()`について、以下を確認する。

* `deleted_at`が設定されること
* レコードが物理削除されないこと
* `updated_at`が更新されること
* 他カラムが変更されないこと

特に、以下が保持されることを確認する。

```text
asset_account_id
name
product_type
start_year_month
memo
```

---

### 30.37 SoftDeletesのTest

無効化実行後に、通常Queryでは対象保有商品が取得できないことを確認する。

一方、

```php
HoldingAsset::withTrashed()
```

を使用すれば、レコード自体は存在することを確認する。

これにより、物理削除されていないことを保証する。

---

### 30.38 UseCaseのUnit Test

UseCaseについて、QueryとRepositoryをMockし、以下を確認する。

正常系：

```text
Query
    ↓
HoldingAsset返却
    ↓
Repository::disable()
    ↓
Disable Result DTO
```

異常系：

```text
Query
    ↓
null
    ↓
HoldingAssetNotFoundException
```

また、対象不存在時に

```text
Repository::disable()
```

が呼び出されないことを確認する。

---

### 30.39 関連データ非更新テスト

Feature TestまたはDatabase Testで、HLD-005実行前後の

```text
month_end_holding_values
month_end_asset_snapshots
```

を比較する。

以下が変更されないことを確認する。

* レコード数
* 商品別月末評価額
* snapshotの`confirmed`
* snapshotの`confirmed_at`

---

### 30.40 再実行テスト

同じ保有商品に対して2回HLD-005を実行する。

1回目：

```text
200 OK
```

2回目：

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

となることを確認する。

また、1回目に設定された

```text
deleted_at
```

が2回目によって更新されないことを確認する。

---

### 30.41 HLD-004とのルートテスト

以下の2ルートが独立して登録されていることを確認する。

```http
PATCH /api/v1/holding-assets/{holdingAssetId}
```

```http
PATCH /api/v1/holding-assets/{holdingAssetId}/disable
```

HLD-005リクエストがHLD-004のActionへルーティングされないことを確認する。

---

### 30.42 レスポンスResourceのTest

正常時に、以下だけが`data`へ含まれることを確認する。

```json
{
  "id": "15",
  "disabled": true
}
```

以下が含まれないことを確認する。

* `assetAccountId`
* `name`
* `productType`
* `startYearMonth`
* `memo`
* `createdAt`
* `updatedAt`
* `deletedAt`

これにより、HLD-005のレスポンスを無効化結果の確認に必要な最小限の情報へ限定する。

---

## 31. React・TypeScriptでの利用

HLD-005は、利用者が保有商品を今後の管理対象から外す際に使用する。

フロントエンドでは、保有商品一覧または保有商品詳細画面から無効化操作を実行する。

概念的な利用フローは、以下とする。

```text
保有商品一覧
または
保有商品詳細
    ↓
無効化ボタン押下
    ↓
確認ダイアログ
    ↓
HLD-005
PATCH
/api/v1/holding-assets/{holdingAssetId}/disable
    ↓
成功
    ↓
関連Query Cache無効化
    ↓
一覧・詳細を再取得
```

HLD-005では、無効化状態をリクエストボディから指定しない。

```text
/disable
```

というエンドポイント自体が無効化操作を表す。

---

### 31.1 TypeScript型

正常レスポンスは、以下のような型として扱う。

概念例：

```ts
export type DisableHoldingAssetResult = {
  id: string;
  disabled: true;
};

export type DisableHoldingAssetResponse = {
  data: DisableHoldingAssetResult;
  requestId: string;
};
```

API共通Envelopeの正式な型が存在する場合は、共通型を利用する。

概念例：

```ts
export type DisableHoldingAssetResponse =
  ApiResponse<DisableHoldingAssetResult>;
```

---

### 31.2 disabledの型

HLD-005の正常レスポンスでは、

```text
disabled = true
```

が固定である。

そのため、TypeScriptでは

```ts
disabled: true;
```

としてリテラル型で表現してよい。

単純に

```ts
disabled: boolean;
```

としてもよいが、HLD-005では`false`が返却されないことを型でも表現できる。

---

### 31.3 リクエスト型

HLD-005では、リクエストボディを使用しない。

そのため、以下のようなリクエストDTO型は作成しない。

```ts
export type DisableHoldingAssetRequest = {
  isEnabled: false;
};
```

または、

```ts
export type DisableHoldingAssetRequest = {
  disabled: true;
};
```

無効化対象は、関数引数の

```text
holdingAssetId
```

だけで指定する。

---

### 31.4 API Client

API Clientでは、`holdingAssetId`を受け取り、HLD-005を呼び出す専用関数を定義する。

概念例：

```ts
export const disableHoldingAsset =
  async (
    holdingAssetId: string,
  ): Promise<DisableHoldingAssetResult> => {
    const response =
      await apiClient.patch<
        DisableHoldingAssetResponse
      >(
        `/api/v1/holding-assets/${holdingAssetId}/disable`,
      );

    return response.data.data;
  };
```

第2引数としてリクエストボディを渡す必要はない。

HTTP Clientの仕様上、明示的に空ボディが必要な場合のみ、

```ts
null
```

などを指定する。

---

### 31.5 X-User-Id

`X-User-Id`は、他のAPIと同様に共通API Clientから付与する。

概念例：

```ts
apiClient.interceptors.request.use(
  (config) => {
    config.headers[
      'X-User-Id'
    ] = currentUserId;

    return config;
  },
);
```

HLD-005専用処理で利用者IDを以下へ追加しない。

* URL
* クエリパラメータ
* リクエストボディ

---

### 31.6 Mutationとして扱う

HLD-005は、保有商品の状態を変更するため、TanStack Queryを使用する場合はMutationとして扱う。

概念例：

```ts
export const useDisableHoldingAsset =
  () =>
    useMutation({
      mutationFn:
        disableHoldingAsset,
    });
```

Queryとして自動実行しない。

---

### 31.7 無効化ボタン

保有商品一覧や詳細画面では、無効化対象となる保有商品IDを渡してMutationを実行する。

概念例：

```tsx
<button
  type="button"
  onClick={() => {
    handleDisable(
      holdingAsset.id,
    );
  }}
>
  無効化
</button>
```

---

### 31.8 確認ダイアログ

保有商品無効化は、Phase1では再有効化機能を提供しない。

そのため、誤操作防止として実行前に確認ダイアログを表示する。

概念例：

```text
この保有商品を無効化しますか？

無効化後は、
今後の商品別月末評価額の
登録対象から除外されます。
```

過度に長い説明は避け、利用者が操作結果を理解できる内容とする。

---

### 31.9 確認ダイアログで表示してよい情報

必要に応じて、以下を表示してよい。

* 保有商品名
* 所属資産口座名

例えば、

```text
「全世界株式」を無効化しますか？
```

のように表示する。

内部IDは利用者へ表示する必要はない。

---

### 31.10 無効化処理

概念例：

```ts
const disableMutation =
  useDisableHoldingAsset();

const handleDisable =
  (
    holdingAssetId: string,
  ): void => {
    disableMutation.mutate(
      holdingAssetId,
    );
  };
```

無効化実行時に、

```text
isEnabled = false
```

などの値を送信しない。

---

### 31.11 二重送信防止

Mutation実行中は、無効化ボタンを非活性化する。

概念例：

```tsx
<button
  type="button"
  disabled={
    disableMutation.isPending
  }
  onClick={() => {
    handleDisable(
      holdingAsset.id,
    );
  }}
>
  {disableMutation.isPending
    ? '無効化中...'
    : '無効化'}
</button>
```

ただし、フロントエンド側の二重送信防止だけを整合性保証とはしない。

---

### 31.12 成功時

HLD-005成功時は、

```text
200 OK
```

とともに、

```json
{
  "id": "15",
  "disabled": true
}
```

が返却される。

フロントエンドでは、`disabled = true`を確認して無効化完了として扱う。

---

### 31.13 成功メッセージ

正常終了後は、必要に応じて

```text
保有商品を無効化しました。
```

のような完了メッセージを表示する。

成功レスポンスから保有商品名を取得する設計ではないため、商品名を含めたい場合は画面側ですでに保持している情報を表示用途として使用してよい。

---

### 31.14 Query Cacheの無効化

HLD-005成功後は、保有商品に関するQuery Cacheを無効化する。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey: [
    'holdingAssets',
  ],
});
```

保有商品詳細Queryをキャッシュしている場合は、対象IDのQueryも無効化する。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey: [
    'holdingAsset',
    result.id,
  ],
});
```

正式なQuery Keyは、フロントエンド共通設計に従う。

---

### 31.15 一覧画面

保有商品一覧が有効な保有商品のみを表示する設計である場合、HLD-005成功後に一覧Queryを再取得すると、無効化した保有商品は一覧から除外される。

フロントエンドで独自に

```ts
items.filter(
  (item) =>
    item.id !== result.id,
);
```

としてもよいが、サーバー状態との整合性を優先する場合はQuery invalidationによる再取得を基本とする。

---

### 31.16 詳細画面

保有商品詳細画面からHLD-005を実行した場合、無効化後は通常のHLD-003詳細取得対象から除外される。

そのため、成功後は以下のような画面遷移を行う。

* 保有商品一覧へ戻る
* 無効化完了画面へ遷移する

成功後に同じ詳細APIをそのまま再取得すると、

```text
HOLDING_ASSET_NOT_FOUND
```

となることを前提とする。

---

### 31.17 HLD-004との使い分け

フロントエンドでは、以下を明確に分ける。

```text
保有商品情報編集
    ↓
HLD-004

保有商品無効化
    ↓
HLD-005
```

HLD-004の更新フォームへ以下の状態変更項目を追加しない。

```text
isEnabled
disabled
deletedAt
```

---

### 31.18 HLD-004 API Client

通常更新は、引き続き

```http
PATCH /api/v1/holding-assets/{holdingAssetId}
```

を使用する。

概念的には、

```ts
updateHoldingAsset(
  holdingAssetId,
  request,
);
```

とする。

---

### 31.19 HLD-005 API Client

無効化は、

```http
PATCH /api/v1/holding-assets/{holdingAssetId}/disable
```

を使用する。

概念的には、

```ts
disableHoldingAsset(
  holdingAssetId,
);
```

とする。

API Clientの関数名でも通常更新と無効化を明確に区別する。

---

### 31.20 状態フラグをフロントエンドで送らない

以下のような汎用状態変更関数はPhase1では作成しない。

```ts
updateHoldingAssetStatus(
  holdingAssetId,
  false,
);
```

または、

```ts
updateHoldingAsset(
  holdingAssetId,
  {
    disabled: true,
  },
);
```

無効化という業務操作を明示する

```ts
disableHoldingAsset(
  holdingAssetId,
);
```

を使用する。

---

### 31.21 無効化済み状態をローカルだけで保持しない

HLD-005成功後に、React Stateだけで

```text
disabled = true
```

へ変更して永続的な状態とみなさない。

サーバー側ではSoftDeletesによって通常取得対象から除外されるため、関連Queryを再取得して最新状態へ同期する。

---

### 31.22 404 HOLDING_ASSET_NOT_FOUND

以下の場合は、

```text
HOLDING_ASSET_NOT_FOUND
```

が返却される。

* 保有商品不存在
* 他利用者の保有商品
* すでに無効化済み
* 無効な資産口座配下の保有商品

フロントエンドでは、原因ごとにリソース存在状態を推測しない。

「対象の保有商品が見つかりません」などの共通表示とする。

---

### 31.23 二重実行時

1回目のHLD-005が正常終了した後、何らかの理由で同じIDに対して再度HLD-005を実行すると、

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

となる。

フロントエンドでは、このレスポンスを受けた場合も一覧を再取得することで、現在状態を確認できる。

---

### 31.24 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

HLD-005専用画面で独自処理を実装しない。

---

### 31.25 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、API共通の利用者コンテキストエラーとして扱う。

必要に応じて、利用者選択状態を再確認する。

---

### 31.26 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合も、API共通エラー処理に従う。

現在選択されている利用者が有効でない状態として扱う。

---

### 31.27 INVALID_HOLDING_ASSET_ID

通常のUI操作では、サーバーから取得した有効な保有商品IDを使用するため、発生頻度は低い。

発生した場合は、不正な画面状態またはURLとして共通エラー処理を行う。

---

### 31.28 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
保有商品を無効化できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

成功扱いとして一覧から対象商品を削除してはならない。

---

### 31.29 エラー時のQuery Cache

HLD-005が失敗した場合は、原則として保有商品Query Cacheを成功時と同じ扱いで更新しない。

サーバー側で無効化が完了していることを確認できないためである。

ただし、通信断などで結果が不明な場合は、必要に応じて一覧Queryを再取得して最新状態を確認してよい。

---

### 31.30 Optimistic Update

Phase1では、HLD-005に対するOptimistic Updateは必須としない。

保有商品無効化は頻繁に実行する操作ではなく、失敗時に一覧へ戻す処理も必要になるため、サーバー成功後にQuery Cacheを更新する単純な方式を基本とする。

---

### 31.31 再有効化UIを作成しない

Phase1では、無効化済み保有商品を再有効化するAPIを提供しない。

そのため、フロントエンドにも

```text
再有効化
有効に戻す
```

などの操作UIを設けない。

同じ商品を再度管理する場合は、新規保有商品登録の導線を使用する。

---

### 31.32 deletedAtを画面制御に使用しない

HLD-005成功レスポンスでは、

```text
deletedAt
```

を返却しない。

フロントエンドでは、`deletedAt`を参照して無効化成功を判定しない。

正常レスポンスの

```text
disabled = true
```

とHTTP成功を使用する。

---

### 31.33 disabledを更新APIの状態管理へ流用しない

HLD-005レスポンスの

```text
disabled
```

は、無効化操作の結果確認用である。

保有商品エンティティ全体に

```ts
disabled: boolean;
```

を必ず持たせることを意味しない。

HLD-001やHLD-003で無効化済み商品を通常返却しない設計であれば、一覧・詳細DTOへ`disabled`を追加する必要はない。

---

### 31.34 保有商品の型を不要に変更しない

HLD-005を追加したことだけを理由として、既存の

```ts
export type HoldingAsset = {
  // ...
};
```

へ

```ts
disabled: boolean;
deletedAt: string | null;
```

などを追加しない。

各APIが実際に返却するレスポンス契約に合わせて型を定義する。

---

### 31.35 APIエンドポイントを共通更新関数へ隠しすぎない

以下のような何でも処理できる汎用関数へまとめすぎない。

```ts
changeHoldingAsset(
  holdingAssetId,
  action,
  payload,
);
```

Phase1では、

```ts
updateHoldingAsset();
disableHoldingAsset();
```

のように、業務操作が分かる関数名を使用する。

これにより、フロントエンドコードからもHLD-004とHLD-005の責務が分かるようにする。


---

## 32. 設計上の補足

### 32.1 `/disable`を採用する理由

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

### 32.2 HLD-004と同じURLを使用しない理由

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

### 32.3 DELETEを採用しない理由

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

### 32.4 PATCHを採用する理由

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

### 32.5 `/disable`で操作を明示する理由

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

### 32.6 リクエストボディを使用しない理由

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

### 32.7 `isEnabled`カラムを追加しない理由

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

### 32.8 SoftDeletesを無効化表現として使用する理由

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

### 32.9 過去の商品別月末評価額を削除しない理由

保有商品を無効化しても、過去にその商品を保有していた事実は変わらない。

そのため、

```text
month_end_holding_values
```

を削除・更新しない。

過去の資産状況や資産推移を正しく再現できる状態を維持する。

---

### 32.10 過去の月末資産状況を変更しない理由

保有商品を無効化した時点で、過去に確定済みの月末資産状況まで変更すると、過去時点の資産状態が現在の操作によって変化してしまう。

そのため、HLD-005では

```text
month_end_asset_snapshots
month_end_holding_values
```

を変更しない。

---

### 32.11 確定済み月末資産状況があっても無効化できる理由

HLD-005は、過去データを変更する操作ではなく、今後の管理対象から保有商品を外す操作である。

そのため、過去に確定済みの月末資産状況が存在すること自体を無効化禁止条件とはしない。

過去の確定状態を保持したまま、保有商品だけを無効化する。

---

### 32.12 再有効化APIを用意しない理由

Phase1では、無効化済み保有商品を再有効化する機能を提供しない。

再有効化を許可すると、以下の追加ルールが必要になる。

* 過去の無効化期間をどう扱うか
* 対象年月時点の有効性をどう判定するか
* 過去の同名商品との関係をどう扱うか

Phase1では、同じ商品を再度管理する場合はHLD-002で新規登録する方式とする。

---

### 32.13 無効化済み保有商品への再実行を404とする理由

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

### 32.14 他利用者の保有商品も404とする理由

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

### 32.15 HLD-004で無効化させない理由

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

### 32.16 HLD-005で通常属性を変更させない理由

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

### 32.17 専用UseCaseを分ける理由

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

### 32.18 FormRequestを作成しない理由

HLD-005には、リクエストボディが存在しない。

専用FormRequestを作成しても、検証対象となるボディ項目がないため、空クラスになる。

Phase1では、実装クラスを増やすこと自体を目的とせず、必要な責務が存在する場合だけクラスを作成する。

---

### 32.19 QueryとRepositoryを分ける理由

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

### 32.20 `delete()`を使用する理由

LaravelのSoftDeletesを使用するため、無効化処理では

```php
$holdingAsset->delete();
```

を使用する。

`deleted_at`を直接更新する方式ではなく、Laravel標準の論理削除処理へ統一する。

これにより、SoftDeletesを利用していることが実装からも明確になる。

---

### 32.21 明示的なトランザクションを使用しない理由

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

### 32.22 明示的なロックを使用しない理由

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

### 32.23 `disabled`をレスポンスへ返す理由

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

### 32.24 `deletedAt`を返却しない理由

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

### 32.25 無効化済み商品を通常一覧へ返さない理由

HLD-001が現在利用中の保有商品一覧を取得するAPIである場合、論理削除済み商品は通常一覧へ含めない。

HLD-005成功後に一覧を再取得すると、対象商品が表示されなくなるシンプルな挙動とする。

無効化済み商品一覧が将来的に必要になった場合は、別途参照方法を検討する。

---

### 32.26 Optimistic Updateを必須としない理由

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

### 32.27 Idempotency-Keyを採用しない理由

HLD-005は、同じ保有商品を2回無効化できない。

1回目で

```text
deleted_at IS NOT NULL
```

となり、2回目は有効な対象として取得されない。

そのため、Phase1ではIdempotency-Key管理を追加しない。

---

### 32.28 無効化日時を再更新しない理由

無効化済み商品への再実行時に、

```text
deleted_at = now()
```

と再更新すると、実際に最初に無効化した日時が失われる。

そのため、無効化済み商品は更新対象へ含めず、最初の`deleted_at`を保持する。

---

### 32.29 関連APIへの影響

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

### 32.30 CSVインポートへの影響

商品別月末評価額CSVでは、対象年月時点で記録対象となる保有商品かどうかを判定する。

HLD-005によって現在無効化されていることだけを理由に、過去対象年月の評価額まで一律に無効としない。

具体的な対象年月判定は、CSV-005・CSV-006側の業務ルールに従う。

---

### 32.31 無効化APIと対象年月判定を分離する理由

HLD-005の責務は、保有商品を現在の管理対象から外すことである。

一方、

```text
ある過去年月に
その商品が記録対象だったか
```

という判定は、月末資産・CSVインポート側の責務である。

HLD-005へ過去年月判定のロジックまで持たせない。

---

### 32.32 Phase1では設計を広げすぎない

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

## 33. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)