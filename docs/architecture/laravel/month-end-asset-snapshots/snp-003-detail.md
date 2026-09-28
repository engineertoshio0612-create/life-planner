# SNP-003 月末資産状況詳細取得

## 概要

操作対象となる利用者に属する、
指定された月末資産状況の詳細情報を
1件取得するAPI。

確定済み・未確定の両方を取得対象とし、
以下の情報を返却する。

- 月末資産状況ID
- 対象年月
- 確定状態

本APIでは、
月末資産状況そのものの情報のみを取得し、
以下の関連情報は取得しない。

- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品
- 利用可能資産設定

Laravelでは、
以下の責務を分離して実装する。

- Action
  - HTTPリクエストの受付
  - 月末資産状況IDの取得
  - 利用者コンテキストの取得
  - UseCaseの呼び出し
- UseCase
  - 月末資産状況詳細取得処理の制御
- Query
  - 月末資産状況IDと利用者IDを条件とした取得
  - 利用者境界の保証
- Responder
  - HTTPレスポンスへの変換
- API Resource
  - APIレスポンス形式への変換

月末資産状況の取得では、
月末資産状況IDだけで検索せず、

```text
snapshotId
+
userId
```

を一体の検索条件として扱う。

これにより、
他利用者に属する月末資産状況を
取得できないようにする。

指定された月末資産状況が存在しない場合と、
他利用者に属している場合は、
外部レスポンスでは区別せず、

`404 Not Found`

および

`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`

として扱う。

これにより、
他利用者の月末資産状況の存在有無を
外部から判別できないようにする。

`snapshotId`については、
API共通方針に従って
ID形式を検証する。

形式が不正な場合は、
`VALIDATION_ERROR`
として扱う。

本APIは参照専用であり、
以下の処理は行わない。

- 月末資産状況の作成
- 月末資産状況の更新
- 月末資産状況の確定
- 月末資産状況の確定解除
- 確定可否判定
- 確定解除可否判定

永続化処理を行わないため、
Repositoryは使用せず、
月末資産状況の参照はQueryが担当する。

また、
参照処理のみであるため、
明示的なデータベーストランザクションや
行ロックは使用しない。

APIレスポンスでは、
IDを文字列、
対象年月を `YYYY-MM` 形式、
確定状態をbooleanとして返却する。

正常終了時は、
`200 OK` とともに
月末資産状況を
`data` オブジェクトとして返却する。

レスポンス生成に必要な
以下のカラムのみを取得し、

```text
id
target_year_month
confirmed
```

不要なJOINやEager Loadingは行わない。

Phase1では、
利用者境界を確実に保証するため、
`snapshotId` のみを条件とする
Laravel標準のRoute Model Bindingへ
取得処理を委譲しない。

独自Bindingを利用する場合も、

```text
id
+
user_id
```

による検索と同等の
利用者境界を保証する。

LaravelやPostgreSQLの内部例外、
SQL、
スタックトレースなどの内部情報は
APIレスポンスへ公開せず、
API共通方針に従って
独自エラーコードへ変換する。

Phase1では、
参照処理と他の更新処理との間で
厳密なスナップショット分離は保証しない。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
月末資産状況IDおよび
利用者コンテキストを取得する。

月末資産状況詳細取得UseCaseを呼び出し、
取得結果をResponderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 月末資産状況IDの形式検証
- 利用者境界の判定
- 月末資産状況の取得
- 存在確認
- レスポンス生成処理

本APIはGETリクエストであるため、
リクエストボディは扱わない。

---

### 2.2 UseCase

月末資産状況詳細取得の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 月末資産状況IDを受け取る
- Queryを呼び出して月末資産状況を取得する
- 取得結果を返却する

指定された月末資産状況が存在しない場合、
または操作対象利用者に属していない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

確定済み・未確定の
どちらも取得対象とする。

本UseCaseでは、
以下の処理は行わない。

- 月末資産状況の作成
- 月末資産状況の確定
- 月末資産状況の確定解除
- 月末資産残高の取得
- 商品別月末評価額の取得
- 確定可否判定
- 確定解除可否判定

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

月末資産状況の存在確認および
利用者境界の確認は、
Queryで行う。

---

### 2.4 Query

操作対象利用者に属する
月末資産状況を1件取得する。

取得条件には、
必ず月末資産状況IDおよび
操作対象利用者IDを含める。

```php
$snapshot = MonthEndAssetSnapshot::query()
    ->where('id', $snapshotId)
    ->where('user_id', $userId)
    ->first([
        'id',
        'target_year_month',
        'confirmed',
    ]);
```
以下のように、月末資産状況IDだけで取得してはならない。

```php
MonthEndAssetSnapshot::find($snapshotId);
```

取得できなかった場合は、以下を区別せず`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`として扱う。

- 指定された月末資産状況が存在しない
- 指定された月末資産状況が他の利用者に属している

確定状態は、取得条件に含めない。

以下のどちらも取得対象とする。

```text
confirmed = true
confirmed = false
```

---

### 2.5 Repository

本APIは参照系APIであるため、Repositoryによる永続化処理は行わない。

以下の処理は実施しない。

- 登録
- 更新
- 確定
- 確定解除
- 削除

月末資産状況の参照は、Queryが担当する。

---

### 2.6 トランザクション

本APIは参照処理のみであるため、明示的なデータベーストランザクションは使用しない。

月末資産状況詳細の取得によって、以下のデータを変更しない。

- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`

Phase1では、参照処理と他の更新処理との間で厳密なスナップショット分離は保証しない。

---

### 2.7 Responder

UseCaseから受け取った月末資産状況を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`200 OK`とともに`data`オブジェクトとして返却する。

指定された月末資産状況が取得できなかった場合は、`404 Not Found`および`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`へ変換する。

Responderは、以下の処理を行わない。

- データベース検索
- 利用者境界の判定
- 確定可否判定
- 確定解除可否判定
- 月末資産残高の取得
- 商品別月末評価額の取得

---

### 2.8 API Resource

データベースカラムを直接返却せず、API Resourceを利用してAPIレスポンス形式へ変換する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'targetYearMonth' => $this->target_year_month,
    'confirmed' => (bool) $this->confirmed,
];
```

以下の項目は、レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`
- 月末資産残高の明細
- 商品別月末評価額の明細
- 資産口座情報
- 保有商品情報
- 利用可能資産設定

本APIでは、API Resourceから関連モデルを参照しない。

---

### 2.9 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 2.10 Eloquentモデル

`MonthEndAssetSnapshot`モデルは、`month_end_asset_snapshots`テーブルへ対応する。

本APIでは、主に以下の属性を使用する。

```text
id
user_id
target_year_month
confirmed
```

`confirmed`は、booleanとして扱えるようにcastを設定する。

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

### 2.11 パフォーマンス

本APIでは、レスポンス生成に必要な以下のカラムのみ取得する。

```text
id
target_year_month
confirmed
```

一覧・詳細表示に不要な以下の関連情報は取得しない。

- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品
- 利用可能資産設定

不要なEager LoadingおよびJOINは行わない。

月末資産状況IDおよび利用者IDを条件として、1件のみ取得する。

---

### 2.12 Route Model Bindingを直接使用しない理由

Laravelの通常のRoute Model Bindingで`snapshotId`のみを条件に月末資産状況を取得すると、利用者境界の判定が取得処理と分離する可能性がある。

本APIでは、

```text
id
+
user_id
```

を一体の検索条件として扱う。

そのため、Phase1では`snapshotId`だけを使用した通常のRoute Model Bindingへ取得処理を委譲しない。

利用者コンテキストを含めた独自Bindingを実装する場合は、Queryと同等の利用者境界を必ず保証する。

---

### 2.13 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
| --- | --- |
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 月末資産状況ID形式不正 | `VALIDATION_ERROR` |
| 月末資産状況不存在 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 利用者境界外の月末資産状況 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

月末資産状況が存在しない場合と他利用者に属している場合は、外部レスポンスでは区別しない。

SQL、スタックトレースおよび内部例外メッセージは、APIレスポンスへ含めない。

ログには、調査に必要な範囲で以下を記録する。

- 操作対象利用者ID
- 月末資産状況ID
- 独自エラーコード
- リクエストID

月末資産残高や商品別月末評価額など、本APIで取得しない業務データを不要にログへ出力しない。

---

## 3. 関連ドキュメント

- [SNP-003 API詳細設計](../../../api/details/month-end-asset-snapshots/snp-003-detail.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [月末資産状況 Laravelアーキテクチャ設計](./README.md)
- [SNP-003 テスト設計](../../../tests/month-end-asset-snapshots/snp-003-detail.md)
- [月末資産状況 テスト設計](../../../tests/month-end-asset-snapshots/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)