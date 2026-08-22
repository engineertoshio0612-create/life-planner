# BAL-001 月末資産残高一覧取得

## 1. 概要

操作対象となる利用者について、
指定した月末資産状況に紐づく
月末資産残高の一覧を取得する。

本APIでは、
資産口座単位で残高を記録する
資産口座について、
対象年月の月末資産残高を返却する。

月末資産残高が
まだ登録されていない資産口座についても、
対象年月時点で
残高記録対象となる資産口座であれば
一覧へ含める。

これにより、
利用者は対象年月について、

- どの資産口座が残高記録対象であるか
- 月末資産残高が登録済みであるか
- 登録済みの場合はいくらであるか

を確認できる。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
月末資産状況IDおよび
利用者コンテキストを取得する。

月末資産残高一覧取得UseCaseを呼び出し、
取得結果をResponderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- `snapshotId`の形式検証
- 利用者境界の判定
- 月末資産状況の存在確認
- 対象年月時点の資産口座判定
- 残高記録単位の判定
- 月末資産残高の取得
- 未登録状態の組み立て
- 並び順制御
- レスポンス生成処理

本APIはGETリクエストであるため、
リクエストボディは扱わない。

---

### 2.2 UseCase

月末資産残高一覧取得の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 月末資産状況IDを受け取る
- 月末資産状況を取得する
- 対象年月を取得する
- 対象年月時点で月末資産管理対象となる資産口座を取得する
- 残高記録単位が口座単位の資産口座へ絞り込む
- 対象の月末資産残高を取得する
- 資産口座と月末資産残高を対応付ける
- 未登録資産口座を含めた一覧を生成する
- 取得結果を返却する

指定された月末資産状況が存在しない場合、
または操作対象利用者に属していない場合は、
`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
として扱う。

対象となる資産口座が0件の場合は、
エラーとせず、
空の一覧を返却する。

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
利用者境界確認は、
UseCaseおよびQueryで行う。

---

### 2.4 Query

月末資産残高一覧取得に必要な
データ取得を担当する。

主な取得対象は、
以下とする。

- `month_end_asset_snapshots`
- `asset_accounts`
- `asset_account_available_settings`
- `month_end_asset_balances`

本APIでは、
以下を原則として参照しない。

- `holding_assets`
- `month_end_holding_values`

---

### 2.5 月末資産状況取得

月末資産状況は、
必ず利用者境界を含めて取得する。

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

### 2.6 対象資産口座取得

月末資産状況の
`target_year_month`をもとに、
対象年月時点で
月末資産管理対象となる資産口座を取得する。

また、
残高記録単位が
口座単位の資産口座のみを対象とする。

概念的には、
以下の条件で取得する。

```text
user_id = 操作対象利用者ID
AND
対象年月時点で月末資産管理対象
AND
balance_recording_unit = 口座単位
```

対象年月時点の利用可否判定には、
`asset_account_available_settings`
を使用する。

現在の状態だけをもとに
過去の対象年月の一覧対象を
決定してはならない。

---

### 2.7 月末資産残高取得

対象となる月末資産状況に紐づく
月末資産残高を一括取得する。

```php
$balances = MonthEndAssetBalance::query()
    ->where(
        'month_end_asset_snapshot_id',
        $snapshot->id,
    )
    ->get([
        'asset_account_id',
        'balance',
    ])
    ->keyBy('asset_account_id');
```

資産口座ごとに
個別SELECTを実行しない。

以下のようなN+1実装は避ける。

```php
foreach ($assetAccounts as $assetAccount) {
    $balance = MonthEndAssetBalance::query()
        ->where(
            'month_end_asset_snapshot_id',
            $snapshot->id,
        )
        ->where(
            'asset_account_id',
            $assetAccount->id,
        )
        ->first();
}
```

---

### 2.8 一覧データの組み立て

対象資産口座を基準として、
取得済み月末資産残高を
対応付ける。

月末資産残高が存在しない場合は、
`balance = null`
として扱う。

例：

```php
$items = $assetAccounts->map(
    function (
        AssetAccount $assetAccount,
    ) use ($balances): array {
        $balance = $balances->get(
            $assetAccount->id,
        );

        return [
            'assetAccountId'
                => (string) $assetAccount->id,
            'assetAccountName'
                => $assetAccount->name,
            'balance'
                => $balance?->balance,
        ];
    },
);
```

`balance = 0`の場合は、
`null`へ変換してはならない。

以下のような判定は避ける。

```php
'balance' => $balance?->balance ?: null,
```

`0`まで`null`へ変換されるため、
使用しない。

---

### 2.9 並び順

一覧は、
以下の順序で返却する。

1. 資産口座名昇順
2. 資産口座ID昇順

Query側で並び替えることを基本とする。

```php
$assetAccounts = AssetAccount::query()
    ->where('user_id', $userId)
    ->orderBy('name')
    ->orderBy('id')
    ->get();
```

対象年月時点の条件および
残高記録単位の条件も、
実際のQueryへ含める。

ResponderやResourceでは、
並び順を変更しない。

---

### 2.10 Repository

本APIは参照系APIであるため、
Repositoryによる永続化処理は行わない。

以下の処理は実施しない。

- 月末資産残高の登録
- 月末資産残高の更新
- 月末資産残高の削除
- 月末資産状況の更新
- 資産口座の更新
- 利用可能資産設定の更新

データ取得は、
Queryが担当する。

---

### 2.11 トランザクション

本APIは参照処理のみであるため、
明示的なデータベーストランザクションは使用しない。

以下のデータは変更しない。

- `month_end_asset_snapshots`
- `asset_accounts`
- `asset_account_available_settings`
- `month_end_asset_balances`

Phase1では、
参照中に別処理によって
関連データが更新された場合の
厳密なスナップショット整合性は保証しない。

---

### 2.12 DTO / View Model

月末資産残高が
未登録の資産口座も
レスポンスへ含めるため、
Eloquentモデルそのものを
直接Responderへ渡すのではなく、
一覧表示用DTOまたはView Modelへ
変換してよい。

例：

```php
final readonly class MonthEndAssetBalanceListItem
{
    public function __construct(
        public string $assetAccountId,
        public string $assetAccountName,
        public ?int $balance,
    ) {
    }
}
```

これにより、

```text
資産口座
+
任意の月末資産残高
```

という本API固有の
レスポンス構造を明示できる。

---

### 2.13 Responder

UseCaseから受け取った
月末資産残高一覧を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`200 OK`とともに
`data`配列として返却する。

対象資産口座が0件の場合も、
`200 OK`と空配列を返却する。

```json
{
  "data": []
}
```

Responderは、
以下の処理を行わない。

- データベース検索
- 利用者境界の判定
- 対象年月時点の資産口座判定
- 残高記録単位の判定
- 未登録状態の判定
- 並び順制御
- 更新可否判定
- 確定可否判定

---

### 2.14 API Resource

APIレスポンスへの変換には、
API Resourceまたは
Resource Collectionを使用する。

変換例：

```php
return [
    'assetAccountId'
        => (string) $this->assetAccountId,
    'assetAccountName'
        => $this->assetAccountName,
    'balance'
        => $this->balance,
];
```

`balance`は、
以下のいずれかとなる。

```text
整数
    → 登録済み

0
    → 0円で登録済み

null
    → 未登録
```

一覧では、
Resource Collectionを利用する。

API Resourceでは、
データベース検索や
未登録状態の判定を行わない。

---

### 2.15 Middleware

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

### 2.16 Eloquentモデル

`MonthEndAssetBalance`モデルでは、
`balance`を整数値として扱う。

`balance`は、
日本円の整数値であり、
小数値として扱わない。

`AssetAccount`モデルでは、
`balance_recording_unit`を
Enumまたは値オブジェクトとして
扱ってよい。

例：

```php
protected function casts(): array
{
    return [
        'balance_recording_unit'
            => BalanceRecordingUnit::class,
    ];
}
```

これにより、
マジックナンバーによる
残高記録単位の比較を避ける。

---

### 2.17 パフォーマンス

本APIでは、
資産口座件数に応じて
SQL発行回数が増加しないようにする。

想定する取得方法は、
以下のいずれかとする。

- 対象資産口座と月末資産残高をLEFT JOINで一括取得する
- 対象資産口座と月末資産残高をそれぞれ1回ずつ取得し、アプリケーション側で対応付ける

例えば、
LEFT JOINを使用する場合は、
月末資産残高が未登録の
資産口座を一覧へ残すため、
INNER JOINを使用しない。

概念例：

```sql
SELECT
    aa.id,
    aa.name,
    meab.balance
FROM asset_accounts aa
LEFT JOIN month_end_asset_balances meab
    ON meab.asset_account_id = aa.id
   AND meab.month_end_asset_snapshot_id = :snapshotId
WHERE
    aa.user_id = :userId
ORDER BY
    aa.name ASC,
    aa.id ASC
```

実際には、
対象年月時点の利用可能条件および
残高記録単位の条件を追加する。

---

### 2.18 例外変換

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
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

以下の状態は、
例外として扱わない。

- 対象資産口座が0件
- 月末資産残高が0件
- 月末資産残高が一部未登録
- 月末資産残高がすべて未登録
- 月末資産残高が0円

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

月末資産残高の具体的な金額は、
不要にエラーログへ出力しない。

---

## 3. 関連ドキュメント

* - [BAL-001 API詳細設計](../../../api/details/asset-balances/bal-001-list.md)
* - [API一覧](../../../api/api-list.md)
* - [API共通方針](../../../api/api-common-policy.md)
* - [エラーコード一覧](../../../api/error-codes.md)
* - [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
* - [月末資産残高 Laravelアーキテクチャ設計](./README.md)
* - [BAL-001 テスト設計](../../../tests/asset-balances/bal-001-list.md)
* - [月末資産残高 テスト設計](../../../tests/asset-balances/README.md)
* - [機能要件](../../../requirements/functional-requirements.md)
* - [ユビキタス言語集](../../../glossary.md)
* - [エンティティ定義](../../../entities.md)
* - [テーブル定義書](../../../table-definition.md)
* - [ER図](../../../er-diagram-phase1.md)
