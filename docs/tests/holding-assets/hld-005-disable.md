# HLD-005 保有商品無効化

## 1. 概要

本ドキュメントでは、HLD-005 保有商品無効化APIに対するテスト方針および主要なテスト観点を定義する。

本APIでは、操作対象となる利用者に帰属する利用中の保有商品を無効化する処理を検証する。

保有商品の無効化は物理削除ではなく、`holding_assets.deleted_at`を設定する論理削除として行う。

無効化後の保有商品は今後の商品別月末評価額の登録対象から除外する一方、過去の商品別月末評価額、月末資産状況および目的達成判定履歴は変更せず保持する。

主に以下を確認する。

* 保有商品の正常な無効化
* 利用者境界
* 保有商品IDおよび利用者コンテキストの検証
* SoftDeletesによる論理削除
* 無効化済み保有商品に対する再実行
* 所属資産口座の状態による操作可否
* 過去の商品別月末評価額および月末資産状況の保持
* 同一資産口座に所属する他の保有商品への非影響
* HLD-004 保有商品更新とのエンドポイントおよび責務分離
* 正常・異常レスポンス契約
* 副作用範囲
* 想定外エラー発生時の処理

特に無効化処理については、対象となる`holding_assets`の1レコードのみが変更され、保有商品の通常属性や資産口座、過去の商品別月末評価額、月末資産状況および目的達成判定履歴などの関連データが変更されないことを確認する。

また、他利用者に帰属する保有商品、無効化済み保有商品および通常の操作対象外となる保有商品を無効化できないことを検証する。

正常終了時は、`200 OK`が返却され、対象レコードの`deleted_at`が設定されること、およびAPI共通方針に従って保有商品IDと`disabled = true`を含む無効化結果が返却されることを確認する。


---

## 2. テスト観点

HLD-005では、正常な無効化だけでなく、以下を重点的に確認する。

- 利用者境界
- 論理削除
- 関連データ非更新
- 再実行
- HLD-004との責務分離

---

### 2.1 正常系

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

### 2.2 X-User-Id未指定

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

### 2.3 X-User-Id形式不正

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

### 2.4 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 2.5 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 2.6 holdingAssetId形式不正

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

### 2.7 保有商品不存在

存在しない`holdingAssetId`を指定する。

期待結果：

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

となること。

---

### 2.8 他利用者の保有商品

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

### 2.9 論理削除済み保有商品

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

### 2.10 所属資産口座が論理削除済み

保有商品自体は論理削除されていないが、所属資産口座が論理削除済みの状態を用意する。

対象が通常の操作対象として取得されないことを確認する。

期待結果：

```text
404 Not Found
HOLDING_ASSET_NOT_FOUND
```

とする。

---

### 2.11 SoftDeletes

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

### 2.12 過去の商品別月末評価額

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

### 2.13 月末資産状況

対象保有商品に関連する月末資産状況が存在する状態でHLD-005を実行する。

以下を確認する。

- snapshotが削除されないこと
- `confirmed`が変更されないこと
- `confirmed_at`が変更されないこと

---

### 2.14 確定済み月末資産状況

過去に

```text
confirmed = true
```

の月末資産状況が存在する状態で保有商品を無効化する。

無効化自体は正常に実行できることを確認する。

また、確定済み月末資産状況が変更されないことを確認する。

---

### 2.15 同じ資産口座の他保有商品

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

### 2.16 再実行

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

### 2.17 リクエストボディなし

以下のリクエストで正常に無効化できることを確認する。

```http
PATCH /api/v1/holding-assets/15/disable
X-User-Id: 1
Accept: application/json
```

リクエストボディを必須としないことを確認する。

---

### 2.18 isEnabledを必要としない

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

### 2.19 HLD-004とのエンドポイント分離

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

### 2.20 HLD-004との責務分離

HLD-005実行時に、以下のHLD-004で扱う項目を変更しないことを確認する。

- `name`
- `product_type`
- `memo`

逆に、HLD-004の通常更新によって`deleted_at`を任意に変更できないことも別途確認する。

---

### 2.21 正常レスポンス契約

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

### 2.22 返却しない情報

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

### 2.23 エラーレスポンス契約

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

### 2.24 副作用範囲

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

### 2.25 INTERNAL_SERVER_ERROR

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

## 3. 関連ドキュメント

- [HLD-005 API詳細設計](../../api/details/holding-assets/hld-005-disable.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [HLD-005 Laravelアーキテクチャ設計](../../architecture/laravel/holding-assets/hld-005-disable.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [HLD-005 Reactアーキテクチャ設計](../../architecture/react/holding-assets/hld-005-disable.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
