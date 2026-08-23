##  CSV-005 商品別月末評価額CSVプレビュー

### 1 概要

CSV-005のテストでは、アップロードされた商品別月末評価額CSVを正しく解析・検証し、CSV-006 商品別月末評価額CSV登録を実行する前のプレビューとして、登録予定内容およびエラー内容を正しく返却できることを確認する。

本APIはCSV内容の検証のみを行い、商品別月末評価額や月末資産状況などの業務データを更新しない。そのため、CSV構造、入力値、利用者境界、業務ルール、エラー集約、レスポンス契約、副作用および性能を主要なテスト対象とする。

```text
CSVファイル受信
    ↓
ファイル・ヘッダー検証
    ↓
CSV行解析
    ↓
入力値検証
    ↓
対象年月特定
    ↓
資産口座・保有商品検証
    ↓
対象年月時点の有効性検証
    ↓
snapshot・既存評価額確認
    ↓
CSV内重複確認
    ↓
エラー集約
    ↓
canImport判定
    ↓
プレビュー結果返却
```

正常なCSVでは、`200 OK`となり、`targetYearMonth`、CSV全行の`rows`、CSV上の行番号、資産口座名、保有商品名、商品別月末評価額が正しく返却され、`canImport = true`となることを確認する。

`value = 0`は正常な業務値として扱い、未入力や不正値とは区別されることを確認する。また、CSV上の行順がレスポンスでも維持されることを確認する。

CSVとして解析可能であっても、資産口座不存在、残高記録単位不一致、保有商品不存在、対象年月時点での保有商品無効、確定済み月末資産状況、既存商品別月末評価額、CSV内重複などの業務エラーが存在する場合は、`canImport = false`となり、CSV全体または該当行へ適切なエラーが設定されることを確認する。

```text
CSV全体エラーなし
+
全行エラーなし
    ↓
canImport = true

CSV全体エラーあり
または
1行以上の行エラーあり
    ↓
canImport = false
```

利用者境界については、`X-User-Id`で特定された利用者に属する資産口座、保有商品、月末資産状況、商品別月末評価額のみが判定へ使用されることを確認する。他利用者に同名の資産口座や保有商品が存在していても、操作対象利用者の判定へ影響しないことを保証する。

対象年月時点の有効性については、現在日時ではなくCSVの`target_year_month`を基準として判定されることを確認する。

複数の行に異なるエラーが存在する場合は、可能な範囲で1回のレスポンスへ集約され、CSV全体のエラーは`data.errors`、各行のエラーは`rows[].errors`へ適切に格納されることを確認する。

CSV-005はプレビュー専用APIであるため、正常・異常を問わず`asset_accounts`、`holding_assets`、`month_end_asset_snapshots`、`month_end_holding_values`などの業務データが変更されないことを確認する。対象年月のsnapshotが存在しない場合でも、CSV-005では新規作成しない。

また、同一利用者・同一DB状態・同一CSVで複数回実行した場合に同じプレビュー結果が返却されることを確認する。一方、プレビュー後にsnapshotなどの業務状態が変更された場合は、再実行時に最新状態を反映した結果へ変化し、古いプレビュー結果が使用されないことを確認する。

CSV-004およびCSV-006との整合性も重要なテスト対象とする。

```text
CSV-004
テンプレート取得
    ↓
CSV-005
プレビュー・検証
    ↓
canImport = true
    ↓
CSV-006
最新状態で再検証・登録
```

同一利用者・同一DB状態・同一CSVでは、CSV-005とCSV-006に共通の登録可否ルールが適用されることを確認する。ただし、CSV-005の`canImport = true`はCSV-006の成功を保証するものではなく、状態変更後はCSV-006が最新状態によって登録を拒否できることも確認する。

さらに、複数行CSVの検証時に資産口座、保有商品、利用可能設定、既存商品別月末評価額などを行単位で繰り返し取得するN+1問題が発生していないことを確認する。

これらのテストにより、CSV-005が業務データへ副作用を与えず、利用者境界と業務ルールを維持しながら、CSV-006による登録前の検証結果を正確かつ一貫した形式で提供できることを保証する。

---

### 2 テスト観点

CSV-005では、
正常系だけでなく、

- CSV構造
- 入力値
- 利用者境界
- 業務ルール
- エラー集約
- 副作用
- 性能

を確認する。

CSV-006と
共通化した検証ロジックについては、
同じ入力に対して
判定結果が一致することも確認する。

---

#### 2.1 正常系

以下を確認する。

- 正常なCSVで`200 OK`となる
- `canImport = true`となる
- `targetYearMonth`が正しく返却される
- CSV全行が`rows`へ返却される
- `rowNumber`がCSV上の行番号と一致する
- `assetAccountName`が正しく返却される
- `holdingAssetName`が正しく返却される
- `value`がintegerとして返却される
- `value = 0`を正常値として扱う
- `errors`が空配列となる
- 各行の`errors`が空配列となる
- CSV上の行順が維持される

---

#### 2.2 利用者

以下を確認する。

- `X-User-Id`未指定
- `X-User-Id`形式不正
- 利用者不存在
- 論理削除済み利用者
- 正常な利用者

利用者コンテキスト不正時に、
CSV解析や
業務データ検索へ
進まないことも確認する。

---

#### 2.3 ファイル入力

以下を確認する。

- `file`未指定
- 正常なCSVファイル
- `.xlsx`
- `.xls`
- `.pdf`
- 空ファイル
- 最大ファイルサイズ以内
- 最大ファイルサイズ超過
- CSVとして解析不能なファイル

---

#### 2.4 CSVヘッダー

以下を確認する。

正常：

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

異常：

- ヘッダー不足
- ヘッダー過多
- ヘッダー名不一致
- ヘッダー順序不一致
- camelCase指定
- 内部ID列追加
- ヘッダーなし

ヘッダー不正時に、
後続の業務データ検索を
行わないことも確認する。

---

#### 2.5 target_year_month

以下を確認する。

- 正常な`YYYY-MM`
- 未入力
- `2026-1`
- `2026/07`
- `202607`
- `2026-00`
- `2026-13`
- 文字列
- 複数対象年月

複数対象年月の場合は、

```text
canImport = false
targetYearMonth = null
```

となることを確認する。

---

#### 2.6 asset_account_name

以下を確認する。

- 正常な資産口座名
- 未入力
- 存在しない資産口座名
- 他利用者にのみ存在する同名資産口座
- 論理削除済み資産口座

他利用者の資産口座が
使用されないことを確認する。

---

#### 2.7 残高記録単位

以下を確認する。

- 商品単位の資産口座
- 口座単位の資産口座

口座単位の場合は、

```text
BALANCE_RECORDING_UNIT_MISMATCH
```

相当の
行エラーとなることを確認する。

---

#### 2.8 holding_asset_name

以下を確認する。

- 正常な保有商品名
- 未入力
- 存在しない保有商品名
- 他資産口座にのみ存在する同名保有商品
- 他利用者にのみ存在する同名保有商品
- 論理削除済み保有商品

対象資産口座と
保有商品の組み合わせによって
正しく特定されることを確認する。

---

#### 2.9 対象年月時点の有効性

以下を確認する。

- 対象年月時点で有効
- 対象年月時点で無効
- 現在は有効だが対象年月時点では無効
- 現在は無効でも対象年月時点では有効

現在日時ではなく、
`target_year_month`を基準に
判定されることを確認する。

---

#### 2.10 value

以下を確認する。

正常：

```text
0
1
1000
1500000
```

異常：

```text
未入力
-1
1000.5
abc
¥1000
1,000
```

0円が
未入力として扱われないことを
確認する。

---

#### 2.11 CSV内重複

以下のようなCSVを確認する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,全世界株式,1600000
```

重複行が

```text
DUPLICATE_HOLDING_ASSET_IN_CSV
```

相当のエラーとなり、

```text
canImport = false
```

となることを確認する。

---

#### 2.12 月末資産状況

以下を確認する。

```text
snapshotなし
```

```text
snapshotあり
confirmed = false
```

```text
snapshotあり
confirmed = true
```

snapshot不存在の場合は、
それだけを理由として
登録不可にならないことを確認する。

確定済みの場合は、

```text
MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED
```

相当の
CSV全体エラーとなることを確認する。

---

#### 2.13 既存商品別月末評価額

以下を確認する。

- 既存データなし
- 既存データあり・CSVと同じ値
- 既存データあり・CSVと異なる値
- 他利用者にのみ既存データあり

操作対象利用者について
既存データが存在する場合は、

```text
MONTH_END_HOLDING_VALUE_ALREADY_EXISTS
```

相当の
行エラーとなることを確認する。

既存値と同じ場合でも
正常扱いしない。

---

#### 2.14 エラー集約

複数行に
異なるエラーが存在するCSVを使用し、
可能な範囲で
複数エラーが
1回のレスポンスに
集約されることを確認する。

例えば、

```text
2行目
ASSET_ACCOUNT_NOT_FOUND

3行目
HOLDING_ASSET_NOT_FOUND

4行目
INVALID_VALUE
```

を同時に返却できることを確認する。

---

#### 2.15 CSV全体エラーと行エラー

以下を確認する。

```text
data.errors
    → CSV全体エラー

rows[].errors
    → 行エラー
```

エラーの種類に応じて
正しい位置へ
格納されることを確認する。

---

#### 2.16 canImport

以下を確認する。

```text
CSV全体エラーなし
+
全行エラーなし
    ↓
canImport = true
```

```text
CSV全体エラーあり
    ↓
canImport = false
```

```text
1行以上の行エラーあり
    ↓
canImport = false
```

フロントエンド側で
再計算する必要がない
正しい値が返却されることを確認する。

---

#### 2.17 利用者境界

最低でも、
2利用者のデータを用意して確認する。

例えば、

```text
User A
    証券口座
        全世界株式

User B
    証券口座
        全世界株式
```

の状態で、
User Aとして実行した場合に、
User Bの

- 資産口座
- 保有商品
- snapshot
- 商品別月末評価額

が
判定へ使用されないことを確認する。

---

#### 2.18 副作用

CSV-005実行前後で、
以下のテーブルの
レコード数・内容が
変更されないことを確認する。

- `asset_accounts`
- `holding_assets`
- `asset_account_available_settings`
- `month_end_asset_snapshots`
- `month_end_holding_values`

特に、
snapshot不存在の場合でも
CSV-005によって
新規snapshotが
作成されないことを確認する。

---

#### 2.19 冪等性

同一利用者、
同一DB状態、
同一CSVで
複数回実行し、

- 同じ`targetYearMonth`
- 同じ`canImport`
- 同じ`errors`
- 同じ`rows`

が返却されることを確認する。

複数回実行しても、
業務データが
変更されないことも確認する。

---

#### 2.20 状態変更後の再プレビュー

1回目のCSV-005で

```text
canImport = true
```

となった後に、
対象snapshotを

```text
confirmed = true
```

へ変更する。

同じCSVで
再度CSV-005を実行し、

```text
canImport = false
```

へ変化することを確認する。

古いプレビュー結果が
キャッシュされていないことを確認する。

---

#### 2.21 SQL発行回数

複数行CSVを使用し、
CSV行数に比例して
SQL発行回数が
増加していないことを確認する。

特に、

```text
asset_accounts
holding_assets
asset_account_available_settings
month_end_holding_values
```

について
N+1が発生していないことを確認する。

---

#### 2.22 CSV-006との整合性

同一利用者、
同一DB状態、
同一CSVについて、
CSV-005とCSV-006で
共通の登録可否ルールが
適用されることを確認する。

CSV-005で

```text
canImport = true
```

となる状態では、
状態変更がなければ
CSV-006でも
登録可能であることを確認する。

CSV-005で

```text
canImport = false
```

となるCSVについては、
CSV-006でも
同じ業務ルールによって
登録が拒否されることを確認する。

---

### 3 関連ドキュメント

- [CSV-005 API詳細設計](../../api/details/csv-imports/csv-005-preview.md)
- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [CSV-005 Laravelアーキテクチャ設計](../../architecture/laravel/csv-imports/csv-005-preview.md)
- [Laravelアーキテクチャ共通設計](../../architecture/laravel/laravel-architecture.md)
- [CSV-005 Reactアーキテクチャ設計](../../architecture/react/csv-imports/csv-005-preview.md)
- [Reactアーキテクチャ共通設計](../../architecture/react/react-architecture.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)