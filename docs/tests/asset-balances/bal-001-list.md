# BAL-001 月末資産残高一覧取得

## 1. 概要

本ドキュメントでは、
BAL-001 月末資産残高一覧取得APIに対する
テスト方針および主要なテスト観点を定義する。

BAL-001では、
操作対象となる利用者について、
指定した月末資産状況に紐づく
資産口座単位の月末資産残高一覧を取得する。

テストでは、
対象年月時点で月末資産管理対象となる
資産口座が正しく抽出され、
残高記録単位が口座単位である資産口座のみが
一覧へ含まれることを確認する。

また、月末資産残高が未登録の場合でも
資産口座自体は一覧へ含め、
`balance = null`として返却されることを確認する。

特に、以下の状態を明確に区別して検証する。

```text
balance = 0
    → 0円で登録済み

balance = null
    → 月末資産残高が未登録
```

主なテスト対象は、
以下とする。

* 月末資産残高一覧の正常取得
* `snapshotId`の形式検証
* 月末資産状況の存在確認
* 利用者境界
* 残高記録単位
* 対象年月時点の資産口座の利用可能状態
* 登録済み・未登録の月末資産残高
* `0円`と未登録の区別
* 0件時のレスポンス
* 一覧の並び順
* 月末資産状況の確定状態
* レスポンス契約
* N+1の防止
* GETリクエストによる副作用がないこと
* 共通エラーおよび想定外例外の扱い

また、他利用者に属する月末資産状況や
資産口座、月末資産残高が
レスポンスへ混入しないことを確認し、
利用者境界がAPI全体で維持されていることを検証する。

本APIは参照系APIであるため、
テスト実行前後で関連データが変更されず、
未登録残高の自動作成などの
副作用が発生しないことも確認する。


---

## 26. テスト観点

### 26.1 正常系

- 月末資産残高一覧を取得できること
- `200 OK`で返却されること
- 対象年月時点で月末資産管理対象となる資産口座が一覧へ含まれること
- 残高記録単位が口座単位の資産口座のみ一覧へ含まれること
- 登録済みの月末資産残高が正しく返却されること
- 月末資産残高が未登録の資産口座も一覧へ含まれること
- 未登録の場合に`balance = null`となること
- `0円`で登録済みの場合に`balance = 0`となること
- 確定済みの月末資産状況について一覧取得できること
- 未確定の月末資産状況について一覧取得できること
- 一覧取得によってデータが変更されないこと

---

### 26.2 snapshotId

- 正しい`snapshotId`を指定して一覧取得できること
- `snapshotId = 1`を指定できること
- `snapshotId = 0`でバリデーションエラーとなること
- 負数でバリデーションエラーとなること
- 小数でバリデーションエラーとなること
- 文字列`abc`でバリデーションエラーとなること
- ID形式不正時に`422 Unprocessable Entity`となること
- ID形式不正時に`VALIDATION_ERROR`となること

---

### 26.3 月末資産状況の存在確認

- 存在する月末資産状況について一覧取得できること
- 存在しない`snapshotId`で`404 Not Found`となること
- 存在しない場合に`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`となること
- 月末資産状況が存在しない場合に空配列として扱わないこと

---

### 26.4 利用者境界

- 操作対象利用者に属する月末資産状況のみ参照できること
- 他利用者に属する月末資産状況を参照できないこと
- 他利用者に属する`snapshotId`で`404 Not Found`となること
- 他利用者に属する場合に`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`となること
- 他利用者の月末資産状況が存在することをレスポンスから判別できないこと
- 月末資産状況の検索条件に`user_id`が含まれていること
- 他利用者の資産口座が一覧へ含まれないこと
- 他利用者の月末資産残高が一覧へ含まれないこと

---

### 26.5 残高記録単位

以下の資産口座が存在する状態を想定する。

```text
普通預金
    balance_recording_unit = 口座単位

現金
    balance_recording_unit = 口座単位

証券口座
    balance_recording_unit = 商品単位
```

この場合、
以下を確認する。

- 普通預金が一覧へ含まれること
- 現金が一覧へ含まれること
- 証券口座が一覧へ含まれないこと
- 商品単位の資産口座に紐づく商品情報を取得しないこと
- 商品別月末評価額を取得しないこと

---

### 26.6 対象年月時点の利用可能状態

資産口座の利用可能期間が
対象年月によって異なる状態を用意する。

以下を確認する。

- 対象年月時点で利用可能な資産口座が一覧へ含まれること
- 対象年月時点で利用可能ではない資産口座が一覧へ含まれないこと
- 現在の利用状態だけを使用して過去月の一覧対象を判定しないこと
- `asset_account_available_settings`をもとに対象年月時点の状態を判定できること
- 過去月では対象だが現在は対象外の資産口座を、過去月の一覧で正しく取得できること
- 現在は対象だが対象年月時点では対象外だった資産口座を、過去月の一覧へ含めないこと

---

### 26.7 登録済み月末資産残高

月末資産残高が
登録済みの場合について、
以下を確認する。

- 正しい資産口座に対応する残高が返却されること
- 別の資産口座の残高と取り違えないこと
- 別の月末資産状況に紐づく残高を取得しないこと
- 別の対象年月の残高を取得しないこと
- 日本円の整数値として返却されること

---

### 26.8 未登録月末資産残高

対象となる資産口座に
月末資産残高が存在しない場合について、
以下を確認する。

- 資産口座自体は一覧へ含まれること
- `balance = null`となること
- 未登録を理由にエラーとならないこと
- 一部だけ未登録の場合でも一覧全体を取得できること
- すべて未登録の場合でも一覧全体を取得できること

---

### 26.9 0円の扱い

以下を区別できることを確認する。

```text
balance = 0
    → 登録済み

balance = null
    → 未登録
```

以下を確認する。

- `0円`の残高が`0`として返却されること
- `0円`を`null`へ変換しないこと
- `0円`の資産口座を未登録として扱わないこと
- `0`と`null`をレスポンス上で区別できること

---

### 26.10 0件

対象年月時点で
残高記録単位が口座単位の
資産口座が存在しない場合について、
以下を確認する。

- `200 OK`となること
- `data`が空配列となること

```json
{
  "data": []
}
```

- `404 Not Found`とならないこと
- 独自エラーコードを返却しないこと

---

### 26.11 並び順

以下を確認する。

- 資産口座名昇順で返却されること
- 資産口座名が同一の場合は資産口座ID昇順となること
- 月末資産残高の登録有無によって並び順が変わらないこと
- `balance`の金額によって並び順が変わらないこと
- 複数回取得してもデータ状態が同じであれば安定した順序で返却されること

---

### 26.12 確定状態

以下の両方について、
一覧取得できることを確認する。

```text
confirmed = false
confirmed = true
```

以下を確認する。

- 未確定でも取得できること
- 確定済みでも取得できること
- 確定状態によって一覧対象が変化しないこと
- 本APIで確定可否を判定しないこと
- 本APIで更新可否を判定しないこと

---

### 26.13 レスポンス契約

- JSONフィールド名がcamelCaseであること
- `data`がarrayで返却されること
- `assetAccountId`が文字列で返却されること
- `assetAccountName`が文字列で返却されること
- 登録済みの`balance`がintegerで返却されること
- 未登録の`balance`が`null`で返却されること
- `balance = 0`がintegerの`0`として返却されること
- `userId`がレスポンスへ含まれないこと
- `snapshotId`がレスポンスへ含まれないこと
- `targetYearMonth`がレスポンスへ含まれないこと
- `confirmed`がレスポンスへ含まれないこと
- 月末資産残高IDがレスポンスへ含まれないこと
- `createdAt`がレスポンスへ含まれないこと
- `updatedAt`がレスポンスへ含まれないこと
- 商品別月末評価額がレスポンスへ含まれないこと
- DB内部のsnake_caseのカラム名がそのまま公開されないこと

---

### 26.14 N+1

複数の資産口座を取得する場合でも、
資産口座ごとに
月末資産残高取得SQLを
個別発行しないことを確認する。

例えば、
以下のような実装を避ける。

```text
資産口座1件取得
    ↓
残高SELECT

資産口座1件取得
    ↓
残高SELECT

資産口座1件取得
    ↓
残高SELECT
```

資産口座と
対象月末資産残高を
JOINまたは一括取得し、
資産口座件数に比例して
SQL発行回数が増加しないことを確認する。

---

### 26.15 副作用

本API実行前後で、
以下のデータが変更されないことを確認する。

- `users`
- `month_end_asset_snapshots`
- `asset_accounts`
- `asset_account_available_settings`
- `month_end_asset_balances`
- `holding_assets`
- `month_end_holding_values`

以下の副作用が
発生しないことを確認する。

- 未登録の月末資産残高を自動作成しない
- `balance = null`をデータベースへ登録しない
- 月末資産状況の確定状態を変更しない
- 資産口座の利用状態を変更しない

---

### 26.16 エラー時

- `USER_CONTEXT_REQUIRED`時に共通エラーレスポンスとなること
- `INVALID_USER_ID`時に共通エラーレスポンスとなること
- `USER_NOT_FOUND`時に共通エラーレスポンスとなること
- `VALIDATION_ERROR`時に共通エラーレスポンスとなること
- `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`時に共通エラーレスポンスとなること
- エラー発生時にもデータが変更されないこと

---

### 26.17 異常系

- 想定外の例外で`500 Internal Server Error`となること
- 共通エラーレスポンス形式で返却されること
- エラーレスポンスとログに同じリクエストIDが記録されること
- SQLがレスポンスへ含まれないこと
- PostgreSQLの内部情報がレスポンスへ含まれないこと
- スタックトレースがレスポンスへ含まれないこと
- 内部例外メッセージがレスポンスへ含まれないこと

---

## 27. Laravel実装方針

### 27.1 Action

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

### 27.2 UseCase

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

### 27.3 パスパラメータ検証

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

### 27.4 Query

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

### 27.5 月末資産状況取得

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

### 27.6 対象資産口座取得

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

### 27.7 月末資産残高取得

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

### 27.8 一覧データの組み立て

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

### 27.9 並び順

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

### 27.10 Repository

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

### 27.11 トランザクション

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

### 27.12 DTO / View Model

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

### 27.13 Responder

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

### 27.14 API Resource

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

### 27.15 Middleware

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

### 27.16 Eloquentモデル

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

### 27.17 パフォーマンス

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

### 27.18 例外変換

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

## 28. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type GetMonthEndAssetBalancesParams = {
  snapshotId: string;
};
```

一覧項目の型は、
以下とする。

```ts
export type MonthEndAssetBalanceListItem = {
  assetAccountId: string;
  assetAccountName: string;
  balance: number | null;
};
```

レスポンス型は、
以下とする。

```ts
export type GetMonthEndAssetBalancesResponse = {
  data: MonthEndAssetBalanceListItem[];
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetMonthEndAssetBalancesResponse>(
    `/api/v1/month-end-asset-snapshots/${snapshotId}/asset-balances`,
  );
```

取得した一覧は、
月末資産残高一覧画面や
月末資産入力画面で利用する。

---

### 28.1 snapshotIdの扱い

`snapshotId`は、
API共通方針に従って
文字列として扱う。

```ts
const snapshotId: string = '12';
```

フロントエンド側で
数値へ変換して
業務計算には使用しない。

URL生成時も、
文字列のまま使用する。

---

### 28.2 balanceの扱い

`balance`は、
登録状態によって
以下のいずれかとなる。

```text
整数
    → 登録済み

0
    → 0円で登録済み

null
    → 未登録
```

TypeScriptでは、
以下のように扱う。

```ts
if (item.balance === null) {
  // 未登録
} else {
  // 登録済み
}
```

以下のような
truthy / falsyによる判定は行わない。

```ts
if (!item.balance) {
  // balance = 0 も未登録扱いになるため使用しない
}
```

`0`と`null`は
明確に区別する。

---

### 28.3 金額表示

登録済みの`balance`は、
日本円の整数値として表示する。

```ts
const formatCurrency = (
  value: number,
): string =>
  new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY',
    maximumFractionDigits: 0,
  }).format(value);
```

使用例：

```ts
const displayBalance =
  item.balance === null
    ? '未登録'
    : formatCurrency(item.balance);
```

フロントエンドで
小数への変換や
独自の端数処理は行わない。

---

### 28.4 未登録状態の表示

`balance = null`の場合は、
入力漏れまたは
未入力であることを
利用者が判断できるように表示する。

表示例：

```text
未登録
```

必要に応じて、
月末資産残高登録画面への
導線を表示する。

`null`を
`0円`として表示してはならない。

---

### 28.5 0円の表示

`balance = 0`の場合は、
登録済みの0円として表示する。

表示例：

```text
￥0
```

`0円`を
未登録として表示してはならない。

---

### 28.6 一覧順の扱い

APIレスポンスは、
以下の順序で返却される。

1. 資産口座名昇順
2. 資産口座ID昇順

フロントエンドでは、
原則として
APIが返却した順序を
そのまま表示する。

同じ並び順を
フロントエンドへ
重複実装しない。

---

### 28.7 データなしの扱い

対象年月時点で
残高記録単位が口座単位の
資産口座が存在しない場合は、
以下が返却される。

```json
{
  "data": []
}
```

フロントエンドでは、
エラーとして扱わない。

表示例：

```text
この対象年月には、
口座単位で残高を記録する
資産口座がありません。
```

---

### 28.8 月末資産状況の確定状態

本APIのレスポンスには、
月末資産状況の`confirmed`は
含まれない。

編集可否を判断する必要がある場合は、
SNP-003 月末資産状況詳細取得APIで
確定状態を取得する。

例えば、
以下のように
複数APIの結果を組み合わせる。

```text
SNP-003
    → confirmed取得

BAL-001
    → 月末資産残高一覧取得
```

フロントエンドでは、
BAL-001のレスポンスだけを使用して
更新可否を判断しない。

---

### 28.9 登録・更新画面への遷移

`balance = null`の場合は、
月末資産残高登録APIを使用する画面へ
遷移できる。

`balance !== null`の場合は、
月末資産残高更新APIを使用する画面へ
遷移できる。

ただし、
月末資産状況が確定済みの場合は、
バックエンド側で更新が拒否される。

そのため、
フロントエンドの表示制御は
補助的なものとし、
更新API側でも
必ず業務ルールを検証する。

---

### 28.10 ローディング表示

一覧取得中は、
ローディング状態を表示する。

`snapshotId`が変更された場合は、
以前の月末資産残高一覧を
現在の対象年月の情報として
表示しない。

新しい取得が完了するまで、
以前の一覧を破棄するか
ローディング表示へ切り替える。

---

### 28.11 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 不正なURLまたはパラメータとして扱う |
| `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` | 月末資産状況一覧画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

月末資産残高が
未登録であること自体は、
エラーとして扱わない。

---

### 28.12 再取得

以下の操作後は、
必要に応じて
BAL-001を再取得する。

- 月末資産残高登録
- 月末資産残高更新

React Query等を使用する場合は、
対象となる
月末資産残高一覧クエリを
invalidateしてよい。

これにより、
登録・更新後の最新残高を
画面へ反映する。

---

## 3. 関連ドキュメント

* [API一覧](../../api/api-list.md)
* [API共通方針](../../api/api-common-policy.md)
* [BAL-001 月末資産残高一覧取得 API詳細設計](../../api/details/asset-balances/bal-001-list.md)
* [BAL-002 月末資産残高登録 テスト設計](./bal-002-create.md)
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
