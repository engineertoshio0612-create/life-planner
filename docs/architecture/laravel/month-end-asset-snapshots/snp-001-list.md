# SNP-001 月末資産状況一覧取得

## 概要

操作対象となる利用者に属する
月末資産状況の一覧を取得するAPI。

月末資産状況一覧画面で使用するため、
対象年月ごとの以下の情報を返却する。

- 月末資産状況ID
- 対象年月
- 確定状態

確定済み・未確定の両方を取得対象とし、
対象年月の降順で返却する。

月末資産状況が存在しない場合は、
エラーとはせず、
`200 OK` と空の `data` 配列を返却する。

本APIでは一覧表示に必要な情報のみを取得し、
月末資産残高および
商品別月末評価額などの明細は取得しない。

原則として、
`month_end_asset_snapshots`
のみを参照し、
不要なJOINやEager Loadingは行わない。

Laravelでは、
以下の責務を分離して実装する。

- Action
  - HTTPリクエストの受付
  - 利用者コンテキストの取得
  - UseCaseの呼び出し
- UseCase
  - 月末資産状況一覧取得処理の制御
- Query
  - 利用者境界を含む一覧取得
  - 対象年月降順の制御
- Responder
  - HTTPレスポンスへの変換
- API Resource
  - APIレスポンス形式への変換

利用者境界はQueryで必ず制御し、
操作対象利用者IDを条件に含めて
月末資産状況を取得する。

本APIは参照専用であり、
月末資産状況の作成・更新・削除、
確定・確定解除は行わない。

また、
永続化処理を行わないためRepositoryは使用せず、
明示的なトランザクションや
行ロックも使用しない。

APIレスポンスでは、
IDを文字列、
対象年月を `YYYY-MM` 形式、
確定状態をbooleanとして返却する。

LaravelやPostgreSQLの内部例外、
SQL、
スタックトレースなどの内部情報は
APIレスポンスへ公開せず、
API共通方針に従って
独自エラーコードへ変換する。

Phase1ではページネーションを採用せず、
操作対象利用者の月末資産状況を
すべて取得する。

また、
一覧取得中の厳密な
スナップショット整合性は保証しない。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
利用者コンテキストを取得する。

月末資産状況一覧取得UseCaseを呼び出し、
取得結果をResponderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 利用者境界の判定
- 月末資産状況一覧の取得
- 並び順制御
- レスポンス生成処理

本APIはGETリクエストであるため、
リクエストボディは扱わない。

---

### 2.2 UseCase

月末資産状況一覧取得の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- Queryを呼び出して月末資産状況一覧を取得する
- 取得結果を返却する

月末資産状況が存在しない場合も、
エラーとはせず、
空の一覧を返却する。

本UseCaseでは、
以下の処理は行わない。

- 月末資産状況の作成
- 月末資産状況の確定
- 月末資産状況の確定解除
- 月末資産残高の取得
- 商品別月末評価額の取得

---

### 2.3 Query

操作対象利用者に属する
月末資産状況一覧を取得する。

取得条件には、
必ず操作対象利用者IDを含める。

```php
$snapshots = MonthEndAssetSnapshot::query()
    ->where('user_id', $userId)
    ->orderByDesc('target_year_month')
    ->get([
        'id',
        'target_year_month',
        'confirmed',
    ]);
```

取得対象は、確定済み・未確定の両方の月末資産状況とする。

取得順は、`target_year_month` の降順とする。

以下のように、利用者条件を含めない一覧取得は行わない。

```php
MonthEndAssetSnapshot::query()
    ->orderByDesc('target_year_month')
    ->get();
```
また、本APIでは一覧表示に不要な関連テーブルをJOINしない。

原則として、`month_end_asset_snapshots` のみを参照する。

### 2.4 Repository

本APIは参照系APIであるため、Repositoryによる永続化処理は行わない。

以下の処理は実施しない。

- 登録
- 更新
- 確定
- 確定解除
- 削除

月末資産状況の参照は、Queryが担当する。

### 2.5 トランザクション

本APIは参照処理のみであるため、明示的なデータベーストランザクションは使用しない。

一覧取得によって、以下のデータを変更しない。

- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`

Phase1では、一覧取得中の厳密なスナップショット分離は保証しない。

### 2.6 Responder

UseCaseから受け取った月末資産状況一覧を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`200 OK` とともに `data` 配列として返却する。

月末資産状況が存在しない場合も、`200 OK` と空配列を返却する。

```json
{
  "data": []
}
```

Responderは、以下の処理を行わない。

- データベース検索
- 利用者境界の判定
- 並び順制御
- 確定可否判定
- 確定解除可否判定

### 2.7 API Resource

データベースカラムを直接返却せず、API Resourceを利用してAPIレスポンス形式へ変換する。

1件分の変換例：

```php
return [
    'id' => (string) $this->id,
    'targetYearMonth' => $this->target_year_month,
    'confirmed' => (bool) $this->confirmed,
];
```

一覧では、Resource Collectionを利用して返却する。

例：

```php
return MonthEndAssetSnapshotResource::collection(
    $snapshots,
);
```

`target_year_month` は、Phase1では `YYYY-MM` 形式の年月文字列として扱う。

通常の日付を表すCarbonへの自動変換は行わない。

### 2.10 パフォーマンス

本APIでは、一覧レスポンスに必要なカラムのみ取得する。

取得対象は、以下とする。

- `id`
- `target_year_month`
- `confirmed`

一覧表示に使用しない以下の情報は取得しない。

- 月末資産残高明細
- 商品別月末評価額明細
- 資産口座
- 保有商品

そのため、不要なEager LoadingおよびJOINは行わない。

Phase1ではページネーションを採用しないため、操作対象利用者の月末資産状況をすべて取得する。

将来的に件数が増加した場合は、API共通方針に従ってページネーションの導入を検討する。

### 2.11 例外変換

LaravelおよびPostgreSQLの内部例外は、そのままAPIレスポンスへ公開しない。

主な例外変換は、以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

月末資産状況が0件であることは例外としない。

SQL、スタックトレースおよび内部例外メッセージは、APIレスポンスへ含めない。

ログには、調査に必要な範囲で以下を記録する。

- 操作対象利用者ID
- 独自エラーコード
- リクエストID

月末資産残高や商品別月末評価額などの業務データを不要にログへ出力しない。

---

## 3. 関連ドキュメント

- [SNP-001 API詳細設計](../../../api/details/month-end-asset-snapshots/snp-001-list.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [月末資産状況 Laravelアーキテクチャ設計](./README.md)
- [SNP-001 テスト設計](../../../tests/month-end-asset-snapshots/snp-001-list.md)
- [月末資産状況 テスト設計](../../../tests/month-end-asset-snapshots/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)