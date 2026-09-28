# SNP-005 月末資産状況確定解除

# SNP-005 月末資産状況確定解除

## 概要

操作対象となる利用者について、
指定された確定済みの月末資産状況を
未確定状態へ戻すAPI。

月末資産状況を確定解除することで、
対象年月の月末資産残高および
商品別月末評価額を
再度修正できる状態にする。

確定解除対象は、
操作対象利用者に属する
確定済みの月末資産状況とする。

指定された月末資産状況が存在しない場合、
または他利用者に属している場合は、
外部レスポンスでは区別せず、

`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`

として扱う。

すでに未確定の場合は、

`MONTH_END_ASSET_SNAPSHOT_ALREADY_UNCONFIRMED`

として扱う。

確定解除による
時系列上の不整合を防ぐため、
確定解除できるのは
操作対象利用者における
最新の確定済み月末資産状況のみとする。

最新の確定済み月末資産状況ではない場合は、

`MONTH_END_ASSET_SNAPSHOT_UNCONFIRM_ORDER_INVALID`

として扱う。

確定解除順序は、
操作対象利用者の
確定済み月末資産状況を
対象年月の降順で取得し、
確定解除対象と
最新の確定済み月末資産状況が
同一であることによって判定する。

月末資産状況自体が存在しない年月は、
確定解除順序の判定対象としない。

Laravelでは、
以下の責務を分離して実装する。

- Action
  - HTTPリクエストの受付
  - 月末資産状況IDの取得
  - 利用者コンテキストの取得
  - UseCaseの呼び出し
- UseCase
  - 確定解除処理全体の制御
- Query
  - 確定解除対象の取得
  - 最新の確定済み月末資産状況の取得
  - 利用者境界の保証
- Validator
  - 確定解除順序の判定
- Repository
  - 月末資産状況の確定状態更新
- Responder
  - HTTPレスポンスへの変換
- API Resource
  - APIレスポンス形式への変換

確定解除条件の最終確認から
確定状態の更新までは、
1つのデータベーストランザクション内で実行する。

また、
確定解除対象となる月末資産状況には
`lockForUpdate()` を使用する。

最新の確定済み月末資産状況についても
同一トランザクション内で取得し、
ロック取得後に
以下を再確認する。

- 確定解除対象が存在すること
- 確定済みであること
- 最新の確定済み月末資産状況であること

これにより、
同一の月末資産状況に対する
複数の確定解除要求が
同時に実行された場合でも、
最新状態を確認した上で
確定解除可否を判定できるようにする。

確定解除処理では、
`confirmed` のみを `false` へ更新する。

以下のデータは変更または削除しない。

- 月末資産状況ID
- 利用者ID
- 対象年月
- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品
- 利用可能資産設定
- 目的達成判定履歴

月末資産残高および
商品別月末評価額は、
確定解除後も登録済みの内容を保持する。

確定解除は、
既存の入力内容を削除する操作ではなく、
保持したまま再編集可能な状態へ戻す操作とする。

月末資産残高および
商品別月末評価額の修正は、
それぞれの更新APIで行う。

また、
保存済みの目的達成判定履歴は
判定当時の記録として保持する。

確定解除を契機として、
目的達成判定履歴の削除・更新や
目的達成判定の自動再実行は行わない。

正常終了時は、
`200 OK` とともに、
確定解除後の月末資産状況を
`data` オブジェクトとして返却する。

レスポンスでは、
以下の情報を返却する。

- 月末資産状況ID
- 対象年月
- 確定状態

正常終了時の
`confirmed` は必ず `false` となる。

LaravelやPostgreSQLの内部例外、
SQL、
スタックトレースなどの内部情報は
APIレスポンスへ公開せず、
API共通方針に従って
独自エラーコードへ変換する。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
月末資産状況IDおよび
利用者コンテキストを取得する。

月末資産状況確定解除UseCaseを呼び出し、
処理結果をResponderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 月末資産状況IDの形式検証
- 利用者境界の判定
- 月末資産状況の取得
- 確定状態の判定
- 確定解除順序の判定
- 確定解除処理
- トランザクション制御
- 排他制御
- レスポンス生成処理

本APIは、
リクエストボディを使用しない。

---

### 2.2 UseCase

月末資産状況確定解除の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 月末資産状況IDを受け取る
- 確定解除対象の月末資産状況を取得する
- 確定済みであることを確認する
- 最新の確定済み月末資産状況であることを確認する
- 月末資産状況を確定解除する
- 確定解除後の月末資産状況を返却する

指定された月末資産状況が存在しない場合、
または操作対象利用者に属していない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

すでに未確定の場合は、
`MONTH_END_ASSET_SNAPSHOT_ALREADY_UNCONFIRMED`
として扱う。

最新の確定済み月末資産状況ではない場合は、
`MONTH_END_ASSET_SNAPSHOT_UNCONFIRM_ORDER_INVALID`
として扱う。

確定解除しても、
月末資産残高、
商品別月末評価額および
保存済みの目的達成判定履歴は変更しない。

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

月末資産状況の存在確認、
利用者境界確認、
確定状態確認および
確定解除順序確認は、
UseCaseおよびQueryで実施する。

---

### 2.4 Query

確定解除処理に必要な
データ取得を担当する。

主な取得対象は、
`month_end_asset_snapshots`
とする。

確定解除対象は、
必ず利用者境界を含めて取得する。

```php
$snapshot = MonthEndAssetSnapshot::query()
    ->where('id', $snapshotId)
    ->where('user_id', $userId)
    ->first();
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

### 2.5 最新確定済み月末資産状況の取得

操作対象利用者における
最新の確定済み月末資産状況を取得する。

検索条件には、
必ず操作対象利用者IDを含める。

```php
$latestConfirmedSnapshot =
    MonthEndAssetSnapshot::query()
        ->where('user_id', $userId)
        ->where('confirmed', true)
        ->orderByDesc('target_year_month')
        ->first();
```

確定解除対象と
最新の確定済み月末資産状況のIDが
一致することを確認する。

```php
if (
    $latestConfirmedSnapshot === null
    || $latestConfirmedSnapshot->id !== $snapshot->id
) {
    throw new
        MonthEndAssetSnapshotUnconfirmOrderInvalidException();
}
```

これにより、
後続月を確定済みのまま
過去月だけを確定解除することを防止する。

月末資産状況自体が存在しない年月は、
確定解除順序の判定対象としない。

---

### 2.6 Repository

月末資産状況の
確定解除を担当する。

更新対象は、
以下とする。

- `confirmed`
- `updated_at`

確定解除時は、
`confirmed`を`false`へ更新する。

```php
$snapshot->confirmed = false;
$snapshot->save();

return $snapshot;
```

以下のデータは変更しない。

- `id`
- `user_id`
- `target_year_month`
- 月末資産残高
- 商品別月末評価額
- 資産口座
- 保有商品
- 利用可能資産設定
- 目的達成判定履歴

月末資産残高および
商品別月末評価額は、
確定解除後もそのまま保持する。

---

### 2.7 トランザクション

確定解除条件の最終確認から
確定状態更新までを、
1つのデータベーストランザクション内で実行する。

実装例：

```php
$snapshot = DB::transaction(
    function () use (
        $userId,
        $snapshotId,
    ): MonthEndAssetSnapshot {
        $snapshot =
            $this->snapshotQuery
                ->findByUserAndIdForUpdate(
                    $userId,
                    $snapshotId,
                );

        if ($snapshot === null) {
            throw new
                MonthEndAssetSnapshotNotFoundException();
        }

        if (! $snapshot->confirmed) {
            throw new
                MonthEndAssetSnapshotAlreadyUnconfirmedException();
        }

        $latestConfirmedSnapshot =
            $this->snapshotQuery
                ->findLatestConfirmedByUserForUpdate(
                    $userId,
                );

        if (
            $latestConfirmedSnapshot === null
            || $latestConfirmedSnapshot->id
                !== $snapshot->id
        ) {
            throw new
                MonthEndAssetSnapshotUnconfirmOrderInvalidException();
        }

        return $this->repository->unconfirm(
            $snapshot,
        );
    },
);
```

確定解除条件を満たさない場合、
または処理途中で例外が発生した場合は、
確定状態を変更しない。

---

### 2.8 排他制御

確定解除対象となる
月末資産状況を取得する際は、
`lockForUpdate()`を使用する。

```php
return MonthEndAssetSnapshot::query()
    ->where('id', $snapshotId)
    ->where('user_id', $userId)
    ->lockForUpdate()
    ->first();
```

最新の確定済み月末資産状況についても、
同じトランザクション内で取得し、
確定解除順序を再確認する。

```php
return MonthEndAssetSnapshot::query()
    ->where('user_id', $userId)
    ->where('confirmed', true)
    ->orderByDesc('target_year_month')
    ->lockForUpdate()
    ->first();
```

ロック取得後に、
以下を再確認する。

- 確定解除対象が存在すること
- 確定済みであること
- 最新の確定済み月末資産状況であること

これにより、
同一月末資産状況に対する
複数の確定解除要求が
同時に実行された場合でも、
後続処理が
最新状態を確認できる。

---

### 2.9 確定解除順序判定クラス

確定解除順序は、
UseCaseへ直接クエリロジックを
記述し続けるのではなく、
専用の判定クラスへ分離してよい。

例：

```text
MonthEndAssetSnapshotUnconfirmOrderValidator
```

主な責務は、
以下とする。

- 操作対象利用者の最新確定済み月末資産状況を確認する
- 確定解除対象が最新確定済みであることを確認する
- 条件を満たさない場合に業務例外を発生させる

利用例：

```php
$this->unconfirmOrderValidator->validate(
    $userId,
    $snapshot,
);
```

Validatorは、
以下の処理を行わない。

- HTTPレスポンス生成
- 月末資産状況の更新
- 月末資産残高の更新
- 商品別月末評価額の更新

---

### 2.10 月末資産残高・商品別月末評価額の扱い

本APIでは、
`month_end_asset_balances`および
`month_end_holding_values`を
更新または削除しない。

確定解除は、
既存の入力内容を保持したまま
修正可能な状態へ戻す操作である。

そのため、
確定解除後も
登録済みの金額をそのまま保持する。

月末資産残高および
商品別月末評価額の修正は、
それぞれの更新APIで行う。

---

### 2.11 目的達成判定履歴の扱い

本APIでは、
`assessment_histories`を
更新または削除しない。

月末資産状況が
判定実行後に確定解除された場合でも、
保存済みの目的達成判定履歴は
判定当時の記録として保持する。

確定解除を契機として、
目的達成判定を
自動的に再実行しない。

---

### 2.12 Responder

UseCaseから受け取った
確定解除後の月末資産状況を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`200 OK`とともに
`data`オブジェクトとして返却する。

以下の場合は、
共通エラーレスポンス形式へ変換する。

- 対象が存在しない
- すでに未確定である
- 最新の確定済み月末資産状況ではない
- 想定外の例外が発生した

Responderは、
以下の処理を行わない。

- 確定状態の判定
- 確定解除順序の判定
- 利用者境界の判定
- 排他制御
- データベース更新

---

### 2.13 API Resource

データベースカラムを直接返却せず、
API Resourceを利用して
APIレスポンス形式へ変換する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'targetYearMonth' => $this->target_year_month,
    'confirmed' => (bool) $this->confirmed,
];
```

正常終了時の
`confirmed`は
必ず`false`となる。

以下の項目は、
レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`
- 月末資産残高の明細
- 商品別月末評価額の明細
- 目的達成判定履歴

---

### 2.14 Middleware

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

### 2.15 Eloquentモデル

`MonthEndAssetSnapshot`モデルでは、
`confirmed`をbooleanとして扱う。

```php
protected function casts(): array
{
    return [
        'confirmed' => 'boolean',
    ];
}
```

`target_year_month`は、
`YYYY-MM`形式の年月を表す
業務値として扱う。

通常の日付を表す
Carbonへのdate castは設定しない。

---

### 2.16 例外変換

LaravelおよびPostgreSQLの内部例外は、
そのままAPIレスポンスへ公開しない。

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 月末資産状況ID形式不正 | `VALIDATION_ERROR` |
| 月末資産状況不存在 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 利用者境界外 | `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` |
| 未確定済み | `MONTH_END_ASSET_SNAPSHOT_ALREADY_UNCONFIRMED` |
| 確定解除順序違反 | `MONTH_END_ASSET_SNAPSHOT_UNCONFIRM_ORDER_INVALID` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

月末資産状況が存在しない場合と
他利用者に属している場合は、
外部レスポンスでは区別しない。

SQL、
スタックトレース、
PostgreSQLの内部情報および
内部例外メッセージは、
APIレスポンスへ含めない。

ログには、
調査に必要な範囲で
以下を記録する。

- 操作対象利用者ID
- 月末資産状況ID
- 対象年月
- 独自エラーコード
- リクエストID

月末資産残高、
商品別月末評価額および
目的達成判定履歴の具体的な金額は、
不要にエラーログへ出力しない。

---

## 3. 関連ドキュメント

- [SNP-005 API詳細設計](../../../api/details/month-end-asset-snapshots/snp-005-unconfirm.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [月末資産状況 Laravelアーキテクチャ設計](./README.md)
- [SNP-005 テスト設計](../../../tests/month-end-asset-snapshots/snp-005-unconfirm.md)
- [月末資産状況 テスト設計](../../../tests/month-end-asset-snapshots/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)