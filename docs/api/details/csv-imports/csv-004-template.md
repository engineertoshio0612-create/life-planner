##  CSV-004 商品別月末評価額CSVテンプレート取得

### 1 概要

商品単位で管理する
保有商品の月末評価額を
CSVインポートするための
CSVテンプレートを取得する。

本APIでは、
以下の項目を持つ
商品単位用CSVテンプレートを返却する。

```text
対象年月
資産口座名
保有商品名
月末評価額
```

CSVテンプレートは、CSV-005 商品別月末評価額CSVプレビュー、CSV-006 商品別月末評価額CSV登録で使用する入力形式とする。

Phase1では、商品単位用CSVの基本的なフォーマットを以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

本APIでは、CSVテンプレートの取得のみを行う。

以下の処理は行わない。

- 商品別月末評価額の登録
- 商品別月末評価額の更新
- 商品別月末評価額の削除
- CSVプレビュー
- CSV入力値検証
- 対象年月の判定
- 重複登録判定
- 月末資産状況の作成
- 月末資産状況の確定

商品単位用のCSVテンプレートを提供することは、機能要件の「CSVテンプレート」および「CSVフォーマット」に対応する。

---

### 2 ユースケース

利用者が、商品単位で管理している複数の保有商品について、月末評価額をCSVで一括登録したい場合に使用する。

基本的な利用フローは、以下とする。

```text
CSV-004
商品別月末評価額CSVテンプレート取得
    ↓
利用者がCSVへ入力
    ↓
CSV-005
商品別月末評価額CSVプレビュー
    ↓
内容確認
    ↓
CSV-006
商品別月末評価額CSV登録
```

利用者は、取得したテンプレートへ以下を入力する。

- 対象年月
- 資産口座名
- 保有商品名
- 月末評価額

1つのCSVファイルでは、1つの対象年月のみを登録対象とする。

---

### 3 エンドポイント

```http
GET /api/v1/month-end-holding-values/csv-template
```

API一覧に定義された商品別月末評価額CSVテンプレート取得用のエンドポイントを使用する。

---

### 4 HTTPメソッド

```text
GET
```

本APIは、商品別月末評価額CSVテンプレートを取得する参照APIであるため、`GET`を使用する。

本APIの実行によって、業務データは変更しない。

以下のテーブルへの登録・更新・削除は行わない。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

CSVファイルのアップロードも行わない。

---

### 5 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、他の利用者依存APIと同様に、

```text
X-User-Id
```

リクエストヘッダーで指定する。

例：

```http
X-User-Id: 1
```

CSVインポートでは、操作対象利用者に属する資産情報のみを検証および登録対象とする。

商品単位CSVでは、主に以下のリソースが利用者境界の対象となる。

- 資産口座
- 保有商品
- 月末資産状況
- 商品別月末評価額

他の利用者に属する資産情報をCSVインポートによって参照または登録対象としてはならない。

ただし、CSV-004ではテンプレート取得のみを行うため、商品別月末評価額の登録対象となる業務データを検索・更新しない。

---

#### 5.1 X-User-Id

操作対象利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者IDは、以下では受け付けない。

- パスパラメータ
- クエリパラメータ
- リクエストボディ

本APIのエンドポイントは、利用者IDをURLへ含めない。

```http
GET /api/v1/month-end-holding-values/csv-template
```

とする。

---

#### 5.2 利用者存在確認

`X-User-Id`で指定された利用者が有効な利用者であることを確認する。

概念的には、以下の条件で確認する。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者が存在しない、または論理削除済みの場合は、テンプレートを返却しない。

---

#### 5.3 利用者固有データをテンプレートへ出力しない

CSV-004では、操作対象利用者に属する以下のデータをCSVテンプレートへ自動出力しない。

- 資産口座
- 保有商品

例えば、以下のような利用者固有のデータ行は生成しない。

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,投資信託A,
,証券口座,投資信託B,
```

テンプレートは、CSV入力形式を示すための固定構造として扱う。

---

#### 5.4 他利用者データを参照しない

CSV-004では、テンプレート生成のために他利用者の以下のデータを参照しない。

- 資産口座
- 保有商品
- 商品別月末評価額

操作対象利用者以外の業務データがCSVテンプレートへ含まれることはない。

---

#### 5.5 資産口座・保有商品の存在確認を行わない

CSV-004では、テンプレート取得時に資産口座や保有商品が存在することを必須としない。

例えば、操作対象利用者に商品単位の資産口座や保有商品がまだ登録されていない場合でも、CSVテンプレート自体は取得可能とする。

資産口座および保有商品の存在確認は、CSV-005およびCSV-006でCSV内容を検証する際に行う。

---

#### 5.6 残高記録単位による取得制限を行わない

CSV-004では、操作対象利用者に

```text
balance_recording_unit
    = 商品単位
```

の資産口座が存在するかどうかによって、テンプレート取得可否を変更しない。

本APIは、商品単位用CSVの入力形式を提供することに責務を限定する。

残高記録単位とCSV種別の整合性は、CSVインポート時の入力チェックで確認する。

---

#### 5.7 保有商品をテンプレートへ事前展開しない

商品単位CSVでは、保有商品名を入力項目として使用する。

ただし、CSV-004で

```text
holding_assets
```

を検索し、登録済み保有商品をテンプレートへ事前展開する方式は採用しない。

テンプレートは、以下のヘッダーを持つ固定形式とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

保有商品の存在確認や対象年月時点での有効性確認は、CSV-005およびCSV-006で行う。

---

#### 5.8 X-User-Id未指定

`X-User-Id`が指定されていない場合は、API共通方針に従って

```text
USER_CONTEXT_REQUIRED
```

として扱う。

テンプレート生成処理へ進まない。

---

#### 5.9 X-User-Id形式不正

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

として扱う。

例えば、

```text
abc
-1
0
```

などを有効な利用者IDとして扱わない。

---

#### 5.10 利用者不存在

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

として扱う。

利用者が存在しない状態でCSVテンプレートを返却しない。

---

#### 5.11 将来の認証導入

将来的に認証機能を導入した場合は、`X-User-Id`による利用者指定を廃止し、

```text
認証済み利用者
```

から操作対象利用者を特定する方式へ変更する。

ただし、CSVテンプレートの以下には影響させない。

- ヘッダー
- CSV形式
- CSV-005・CSV-006との関係

CSVテンプレートは、CSV-005 商品別月末評価額CSVプレビュー、CSV-006 商品別月末評価額CSV登録で使用する入力形式とする。

Phase1では、商品単位用CSVの基本的なフォーマットを以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

本APIでは、CSVテンプレートの取得のみを行う。

以下の処理は行わない。

- 商品別月末評価額の登録
- 商品別月末評価額の更新
- 商品別月末評価額の削除
- CSVプレビュー
- CSV入力値検証
- 対象年月の判定
- 重複登録判定
- 月末資産状況の作成
- 月末資産状況の確定

商品単位用のCSVテンプレートを提供することは、機能要件の「CSVテンプレート」および「CSVフォーマット」に対応する。

---

### 2 ユースケース

利用者が、商品単位で管理している複数の保有商品について、月末評価額をCSVで一括登録したい場合に使用する。

基本的な利用フローは、以下とする。

```text
CSV-004
商品別月末評価額CSVテンプレート取得
    ↓
利用者がCSVへ入力
    ↓
CSV-005
商品別月末評価額CSVプレビュー
    ↓
内容確認
    ↓
CSV-006
商品別月末評価額CSV登録
```

利用者は、取得したテンプレートへ以下を入力する。

- 対象年月
- 資産口座名
- 保有商品名
- 月末評価額

1つのCSVファイルでは、1つの対象年月のみを登録対象とする。

---

### 3 エンドポイント

```http
GET /api/v1/month-end-holding-values/csv-template
```

API一覧に定義された商品別月末評価額CSVテンプレート取得用のエンドポイントを使用する。

---

### 4 HTTPメソッド

```text
GET
```

本APIは、商品別月末評価額CSVテンプレートを取得する参照APIであるため、`GET`を使用する。

本APIの実行によって、業務データは変更しない。

以下のテーブルへの登録・更新・削除は行わない。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

CSVファイルのアップロードも行わない。

---

### 5 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、他の利用者依存APIと同様に、

```text
X-User-Id
```

リクエストヘッダーで指定する。

例：

```http
X-User-Id: 1
```

CSVインポートでは、操作対象利用者に属する資産情報のみを検証および登録対象とする。

商品単位CSVでは、主に以下のリソースが利用者境界の対象となる。

- 資産口座
- 保有商品
- 月末資産状況
- 商品別月末評価額

他の利用者に属する資産情報をCSVインポートによって参照または登録対象としてはならない。

ただし、CSV-004ではテンプレート取得のみを行うため、商品別月末評価額の登録対象となる業務データを検索・更新しない。

---

#### 5.1 X-User-Id

操作対象利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者IDは、以下では受け付けない。

- パスパラメータ
- クエリパラメータ
- リクエストボディ

本APIのエンドポイントは、利用者IDをURLへ含めない。

```http
GET /api/v1/month-end-holding-values/csv-template
```

とする。

---

#### 5.2 利用者存在確認

`X-User-Id`で指定された利用者が有効な利用者であることを確認する。

概念的には、以下の条件で確認する。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者が存在しない、または論理削除済みの場合は、テンプレートを返却しない。

---

#### 5.3 利用者固有データをテンプレートへ出力しない

CSV-004では、操作対象利用者に属する以下のデータをCSVテンプレートへ自動出力しない。

- 資産口座
- 保有商品

例えば、以下のような利用者固有のデータ行は生成しない。

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,投資信託A,
,証券口座,投資信託B,
```

テンプレートは、CSV入力形式を示すための固定構造として扱う。

---

#### 5.4 他利用者データを参照しない

CSV-004では、テンプレート生成のために他利用者の以下のデータを参照しない。

- 資産口座
- 保有商品
- 商品別月末評価額

操作対象利用者以外の業務データがCSVテンプレートへ含まれることはない。

---

#### 5.5 資産口座・保有商品の存在確認を行わない

CSV-004では、テンプレート取得時に資産口座や保有商品が存在することを必須としない。

例えば、操作対象利用者に商品単位の資産口座や保有商品がまだ登録されていない場合でも、CSVテンプレート自体は取得可能とする。

資産口座および保有商品の存在確認は、CSV-005およびCSV-006でCSV内容を検証する際に行う。

---

#### 5.6 残高記録単位による取得制限を行わない

CSV-004では、操作対象利用者に

```text
balance_recording_unit
    = 商品単位
```

の資産口座が存在するかどうかによって、テンプレート取得可否を変更しない。

本APIは、商品単位用CSVの入力形式を提供することに責務を限定する。

残高記録単位とCSV種別の整合性は、CSVインポート時の入力チェックで確認する。

---

#### 5.7 保有商品をテンプレートへ事前展開しない

商品単位CSVでは、保有商品名を入力項目として使用する。

ただし、CSV-004で

```text
holding_assets
```

を検索し、登録済み保有商品をテンプレートへ事前展開する方式は採用しない。

テンプレートは、以下のヘッダーを持つ固定形式とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

保有商品の存在確認や対象年月時点での有効性確認は、CSV-005およびCSV-006で行う。

---

#### 5.8 X-User-Id未指定

`X-User-Id`が指定されていない場合は、API共通方針に従って

```text
USER_CONTEXT_REQUIRED
```

として扱う。

テンプレート生成処理へ進まない。

---

#### 5.9 X-User-Id形式不正

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

として扱う。

例えば、

```text
abc
-1
0
```

などを有効な利用者IDとして扱わない。

---

#### 5.10 利用者不存在

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

として扱う。

利用者が存在しない状態でCSVテンプレートを返却しない。

---

#### 5.11 将来の認証導入

将来的に認証機能を導入した場合は、`X-User-Id`による利用者指定を廃止し、

```text
認証済み利用者
```

から操作対象利用者を特定する方式へ変更する。

ただし、CSVテンプレートの以下には影響させない。

- ヘッダー
- CSV形式
- CSV-005・CSV-006との関係

---

### 6 パスパラメータ

本APIでは、
パスパラメータを使用しない。

エンドポイントは、
以下とする。

```http
GET /api/v1/month-end-holding-values/csv-template
```

利用者ID、
対象年月、
資産口座ID、
保有商品IDなどを
URLへ含めない。

操作対象利用者は、
`X-User-Id`
リクエストヘッダーから特定する。

---

### 7 クエリパラメータ

本APIでは、
クエリパラメータを使用しない。

CSVテンプレートは、
固定形式で返却する。

そのため、
以下のような
クエリパラメータによって
テンプレート内容を変更しない。

- `targetYearMonth`
- `assetAccountId`
- `holdingAssetId`
- `includeSample`
- `encoding`
- `bom`

CSVフォーマット、
文字コード、
BOMなどは、
CSV共通仕様として統一する。

---

### 8 リクエストヘッダー

本APIでは、
以下のリクエストヘッダーを使用する。

| ヘッダー | 必須 | 内容 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | CSVレスポンスを受け取れる形式を指定する |

リクエスト例：

```http
GET /api/v1/month-end-holding-values/csv-template
X-User-Id: 1
Accept: text/csv
```

本APIでは、
リクエストボディを使用しないため、

```http
Content-Type
```

の指定は不要とする。

---

#### 8.1 X-User-Id

`X-User-Id`は、
API共通方針に従って検証する。

正常例：

```http
X-User-Id: 1
```

以下の場合は、
エラーとする。

```text
未指定
形式不正
存在しない利用者
論理削除済み利用者
```

具体的な独自エラーコードは、
「エラーレスポンス」で定義する。

---

#### 8.2 Accept

正常時は、
CSVファイルを返却するため、
`Accept`には

```http
text/csv
```

を指定することを基本とする。

ただし、
API共通クライアントの都合により
`*/*`などを許容する場合は、
API共通方針に従う。

正常レスポンスは
JSONではなくCSVとする。

---

### 9 リクエストボディ

本APIでは、
リクエストボディを使用しない。

以下の情報を
リクエストボディで
受け付けない。

- `userId`
- `targetYearMonth`
- `assetAccountId`
- `holdingAssetId`
- `assetAccountName`
- `holdingAssetName`
- `value`
- CSVテンプレート設定

本APIは、
商品別月末評価額CSVの
固定テンプレートを取得することに
責務を限定する。

---

### 10 レスポンスCSV仕様

本APIでは、
商品別月末評価額CSVとして
以下のヘッダーを持つ
テンプレートを返却する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

ヘッダー順序も
CSV仕様の一部とする。

---

#### 10.1 target_year_month

商品別月末評価額の
対象年月を入力する列とする。

形式は、

```text
YYYY-MM
```

とする。

例：

```text
2026-07
```

CSV-004では、
値を入力済みの状態では返却しない。

ヘッダーだけを提供する。

---

#### 10.2 asset_account_name

保有商品が属する
資産口座名を入力する列とする。

CSV-005およびCSV-006では、
この値と
操作対象利用者を組み合わせて
資産口座を特定する。

CSV-004では、
利用者の資産口座名を
テンプレートへ自動出力しない。

---

#### 10.3 holding_asset_name

商品別月末評価額を登録する
保有商品名を入力する列とする。

CSV-005およびCSV-006では、
資産口座と
`holding_asset_name`の組み合わせをもとに
対象保有商品を特定する。

CSV-004では、
既存の保有商品名を
テンプレートへ自動展開しない。

---

#### 10.4 value

対象年月末時点の
保有商品の評価額を入力する列とする。

Phase1では、
日本円の整数として扱う。

CSV-004では、
値を設定せず
ヘッダーのみを返却する。

---

#### 10.5 ヘッダー順序

ヘッダーは、
以下の順序で固定する。

```text
1. target_year_month
2. asset_account_name
3. holding_asset_name
4. value
```

CSV-005、
CSV-006でも
同じ順序を正式な入力形式として使用する。

---

#### 10.6 利用者固有データを含めない

CSVテンプレートには、
利用者固有のデータ行を含めない。

以下のような
出力は行わない。

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,投資信託A,
,証券口座,投資信託B,
```

正常なテンプレートは、
基本的に以下のみとする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

---

#### 10.7 内部IDを含めない

CSVテンプレートには、
以下の内部ID列を含めない。

- `user_id`
- `asset_account_id`
- `holding_asset_id`
- `month_end_asset_snapshot_id`
- `month_end_holding_value_id`

CSV入力では、
利用者が理解しやすい

- 資産口座名
- 保有商品名

を使用する。

内部IDへの変換は、
CSV-005およびCSV-006で
バックエンド側が行う。

---

### 11 バリデーション

本APIでは、
CSVファイルのアップロードや
業務入力値を受け付けないため、
CSV内容に対する
バリデーションは行わない。

本APIで必要となる
バリデーションは、
主に利用者コンテキストに関する
API共通検証のみとする。

概念的には、
以下となる。

```text
リクエスト受信
    ↓
X-User-Id検証
    ↓
利用者存在確認
    ↓
CSVテンプレート生成
```

---

#### 11.1 X-User-Id 必須

`X-User-Id`は、
必須とする。

未指定の場合は、
API共通方針に従って
エラーとする。

テンプレート生成処理へ
進まない。

---

#### 11.2 X-User-Id 形式

`X-User-Id`は、
有効な利用者ID形式であることを確認する。

概念的には、
1以上の整数として扱う。

正常例：

```text
1
2
100
```

不正例：

```text
0
-1
abc
1.5
```

---

#### 11.3 利用者存在確認

指定された利用者が
存在することを確認する。

概念的な条件は、
以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

存在しない場合、
または論理削除済みの場合は、
CSVテンプレートを返却しない。

---

#### 11.4 資産口座存在確認を行わない

CSV-004では、
商品単位の資産口座が
存在することを
テンプレート取得条件としない。

例えば、
操作対象利用者に
資産口座が0件でも、
CSVテンプレートは取得可能とする。

以下のような
バリデーションは行わない。

```text
商品単位の資産口座が
1件以上存在すること
```

---

#### 11.5 保有商品存在確認を行わない

操作対象利用者に
保有商品が存在しない場合でも、
CSVテンプレートは取得可能とする。

CSV-004では、
以下の存在確認を行わない。

```text
holding_assets
    1件以上存在
```

保有商品の存在確認は、
CSV-005およびCSV-006で
CSV内容を検証する際に行う。

---

#### 11.6 残高記録単位を検証しない

CSV-004では、
商品単位CSV用の
テンプレートを返却するだけである。

そのため、
利用者の資産口座について

```text
balance_recording_unit
    = 商品単位
```

であることを
取得前に検証しない。

残高記録単位の整合性確認は、
CSV-005およびCSV-006で行う。

---

#### 11.7 target_year_monthを検証しない

CSV-004のリクエストには、
`target_year_month`を含めない。

そのため、
対象年月の

- 必須
- `YYYY-MM`形式
- 有効年月
- 1ファイル1対象年月

などは、
CSV-004では検証しない。

これらは、
CSV-005およびCSV-006で
アップロードされたCSVに対して検証する。

---

#### 11.8 asset_account_nameを検証しない

CSV-004では、
`asset_account_name`を
入力値として受け取らない。

そのため、

- 必須
- 存在確認
- 利用者境界
- 残高記録単位

などの検証は行わない。

---

#### 11.9 holding_asset_nameを検証しない

CSV-004では、
`holding_asset_name`を
入力値として受け取らない。

そのため、

- 必須
- 保有商品存在確認
- 資産口座との関連確認
- 対象年月時点の有効性確認

などは行わない。

---

#### 11.10 valueを検証しない

CSV-004では、
`value`を
入力値として受け取らない。

そのため、

- 必須
- 整数
- 0以上
- 日本円として有効

などの検証は行わない。

商品別月末評価額の
入力値検証は、
CSV-005およびCSV-006で行う。

---

#### 11.11 CSVテンプレート定義の整合性

CSV-004では、
生成するテンプレートのヘッダーが
CSV共通Definitionと
一致することを保証する。

概念的には、

```text
CSV-004
生成ヘッダー

=

CSV-005
期待ヘッダー

=

CSV-006
期待ヘッダー
```

とする。

正式なヘッダーは、

```text
target_year_month
asset_account_name
holding_asset_name
value
```

とする。

---

#### 11.12 CSVテンプレート生成失敗

CSVテンプレート生成処理そのものが
失敗した場合は、
入力バリデーションエラーとして
扱わない。

想定外のサーバー内部エラーとして扱う。

具体的なHTTPステータスおよび
独自エラーコードは、
「エラーレスポンス」で定義する。

---

### 12 業務ルール

CSV-004では、
商品別月末評価額CSVを入力するための
固定形式のテンプレートを返却する。

本APIでは、
業務データの登録・更新・削除を行わない。

CSV-005 商品別月末評価額CSVプレビュー、
CSV-006 商品別月末評価額CSV登録で使用する
CSV形式の起点となる。

---

#### 12.1 CSVテンプレートは固定形式とする

商品別月末評価額CSVテンプレートは、
以下の固定ヘッダーを持つ。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

利用者や
登録済みデータによって
ヘッダー構成を変更しない。

---

#### 12.2 ヘッダー順序を固定する

ヘッダー順序は、
以下とする。

```text
1. target_year_month
2. asset_account_name
3. holding_asset_name
4. value
```

CSV-005およびCSV-006でも、
同じヘッダー名・順序を
正式なCSV形式として使用する。

---

#### 12.3 1行目をヘッダーとする

CSVテンプレートの
1行目は、
必ずヘッダー行とする。

概念的には、
以下となる。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

CSV-005およびCSV-006では、
1行目をヘッダーとして解析する。

---

#### 12.4 データ行を含めない

CSV-004では、
テンプレートに
データ行を含めない。

正常なテンプレートは、
以下の1行のみとする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

サンプルデータや
登録済みデータを
自動出力しない。

---

#### 12.5 利用者固有データを含めない

CSVテンプレートには、
操作対象利用者の

- 資産口座
- 保有商品
- 商品別月末評価額

を出力しない。

例えば、
以下のようなテンプレートは
生成しない。

```csv
target_year_month,asset_account_name,holding_asset_name,value
,証券口座,全世界株式,
,証券口座,S&P500,
```

テンプレートは、
利用者に依存しない
固定形式とする。

---

#### 12.6 内部IDを使用しない

商品別月末評価額CSVでは、
資産口座および保有商品を
内部IDではなく名称で指定する。

そのため、
テンプレートには以下を含めない。

```text
user_id
asset_account_id
holding_asset_id
month_end_asset_snapshot_id
month_end_holding_value_id
```

CSV入力では、

```text
asset_account_name
holding_asset_name
```

を使用する。

内部IDへの変換は、
CSV-005およびCSV-006の
バックエンド処理で行う。

---

#### 12.7 target_year_month

`target_year_month`には、
商品別月末評価額の
対象年月を入力する。

形式は、

```text
YYYY-MM
```

とする。

例：

```text
2026-07
```

1つのCSVファイルでは、
1つの対象年月のみを扱う。

CSV-004では値を設定せず、
列のみを提供する。

---

#### 12.8 asset_account_name

`asset_account_name`には、
保有商品が属する
資産口座名を入力する。

CSV-005およびCSV-006では、

```text
操作対象利用者
+
asset_account_name
```

によって
対象資産口座を特定する。

他利用者の同名資産口座を
対象としてはならない。

---

#### 12.9 holding_asset_name

`holding_asset_name`には、
商品別月末評価額を登録する
保有商品名を入力する。

CSV-005およびCSV-006では、
概念的に

```text
操作対象利用者
+
資産口座
+
holding_asset_name
```

によって
対象保有商品を特定する。

同じ名称の保有商品が
別の資産口座に存在していても、
指定された資産口座に属する
保有商品を対象とする。

---

#### 12.10 value

`value`には、
対象年月末時点の
保有商品の評価額を入力する。

Phase1では、
日本円の整数として扱う。

概念例：

```text
1500000
```

CSV上では、
桁区切りや通貨記号を使用しない。

例えば、

```text
1,500,000
¥1500000
```

のような形式を
正式な入力形式とはしない。

具体的な入力値検証は、
CSV-005およびCSV-006で行う。

---

#### 12.11 0円を有効値とする

商品別月末評価額では、

```text
value = 0
```

を有効な値として扱う。

0円を

- 未入力
- NULL
- 登録対象外

として扱わない。

CSV-004では値を持たないが、
CSV-005およびCSV-006で
0円を正常値として扱う。

---

#### 12.12 商品単位の資産口座を対象とする

商品別月末評価額CSVは、
残高記録単位が
商品単位の資産口座に属する
保有商品を対象とする。

概念的には、

```text
asset_accounts.balance_recording_unit
    = 商品単位
```

であることを
CSV-005およびCSV-006で確認する。

CSV-004では、
この業務ルールの判定自体は行わない。

---

#### 12.13 保有商品単位で1行とする

CSVでは、
1つの保有商品について
1行を使用する。

概念例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

同一対象年月、
同一資産口座、
同一保有商品を
複数行に分けて登録することは想定しない。

重複判定は、
CSV-005およびCSV-006で行う。

---

#### 12.14 1ファイル1対象年月とする

1つのCSVファイルに、
複数の対象年月を混在させない。

正常例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-06,証券口座,全世界株式,1400000
2026-07,証券口座,S&P500,800000
```

CSV-004では
対象年月を受け取らないため、
この判定は行わない。

---

#### 12.15 テンプレート取得時に業務データを検索しない

CSV-004では、
テンプレート生成のために

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

を検索しない。

テンプレート生成は、
業務データの状態に
依存させない。

---

#### 12.16 CSV-005・CSV-006と同じ定義を使用する

CSV-004、
CSV-005、
CSV-006では、
商品別月末評価額CSVの
同一Definitionを使用する。

概念的には、

```text
MonthEndHoldingValueCsvDefinition
    ├─ CSV-004
    ├─ CSV-005
    └─ CSV-006
```

とする。

これにより、

```text
CSV-004で取得したテンプレート
    ↓
CSV-005では形式不正
```

となる実装差異を防止する。

---

#### 12.17 CSVテンプレート取得による副作用を発生させない

CSV-004を何度実行しても、
業務データの状態は変化しない。

以下を行わない。

- 月末資産状況作成
- 商品別月末評価額作成
- 資産口座更新
- 保有商品更新
- CSVインポート履歴作成

テンプレート取得回数によって
CSV内容が変化することもない。

---

### 13 処理フロー

正常系の処理フローは、
以下とする。

```text
GET
/api/v1/month-end-holding-values/csv-template
    ↓
X-User-Id取得
    ↓
利用者コンテキスト検証
    ↓
商品別月末評価額CSV Definition取得
    ↓
CSVヘッダー生成
    ↓
CSVレスポンス生成
    ↓
200 OK
```

本APIでは、
テンプレート生成のための
業務データ検索を行わない。

---

#### 13.1 利用者コンテキスト確認

API共通処理で、
`X-User-Id`から
操作対象利用者を特定する。

以下の場合は、
CSV生成へ進まない。

```text
X-User-Id未指定
X-User-Id形式不正
利用者不存在
論理削除済み利用者
```

---

#### 13.2 CSV Definition取得

商品別月末評価額CSVの
共通Definitionから、
ヘッダー定義を取得する。

概念的には、

```text
target_year_month
asset_account_name
holding_asset_name
value
```

を取得する。

CSV-004専用に
別のヘッダー定義を
ハードコードしない。

---

#### 13.3 CSV生成

取得したヘッダーを使用して、
CSVを生成する。

生成内容は、
概念的に以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

データ行は追加しない。

---

#### 13.4 CSVレスポンス生成

生成したCSVを、
ファイルダウンロード可能な
HTTPレスポンスとして返却する。

正常時は、

```http
200 OK
```

とする。

レスポンス本文は、
JSONではなくCSVとする。

---

#### 13.5 異常系

利用者コンテキストに
問題がある場合は、
API共通の
JSONエラーレスポンスを返却する。

概念的には、

```text
利用者コンテキストエラー
    ↓
JSONエラーレスポンス
```

となる。

CSV生成処理中に
想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

として扱う。

---

### 14 正常レスポンス

正常時は、

```http
200 OK
```

を返却する。

レスポンス本文は、
CSVファイルとする。

概念例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

本APIでは、
API共通のJSON成功Envelopeを
使用しない。

---

#### 14.1 Content-Type

正常レスポンスの
`Content-Type`は、

```http
text/csv
```

とする。

文字コード情報を
付与する場合は、
CSV共通仕様に従う。

概念例：

```http
Content-Type: text/csv; charset=UTF-8
```

---

#### 14.2 Content-Disposition

ブラウザから
CSVファイルとして
保存できるよう、

```http
Content-Disposition
```

を設定する。

概念例：

```http
Content-Disposition: attachment; filename="month-end-holding-values-template.csv"
```

正式なファイル名は、
CSV共通の命名方針に従う。

---

#### 14.3 レスポンスボディ

レスポンスボディは、
CSV文字列とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

JSONへ変換しない。

---

#### 14.4 requestId

正常レスポンスが
CSVファイルそのものであるため、
JSON本文へ

```text
requestId
```

を含めない。

リクエスト追跡が必要な場合は、
API共通方針に従って
HTTPレスポンスヘッダーなどで扱う。

エラーレスポンスについては、
API共通のJSONエラー形式を使用する。

---

### 15 レスポンス項目

本APIの正常レスポンスは
JSONではなくCSVであるため、
JSONレスポンス項目は存在しない。

CSVの列定義は、
以下とする。

| 順序 | CSV列名 | 内容 | 入力例 |
|---:|---|---|---|
| 1 | `target_year_month` | 商品別月末評価額の対象年月 | `2026-07` |
| 2 | `asset_account_name` | 保有商品が属する資産口座名 | `証券口座` |
| 3 | `holding_asset_name` | 商品別月末評価額を登録する保有商品名 | `全世界株式` |
| 4 | `value` | 対象年月末時点の商品別評価額 | `1500000` |

---

#### 15.1 target_year_month

CSV上の物理名：

```text
target_year_month
```

意味：

```text
商品別月末評価額の対象年月
```

入力形式：

```text
YYYY-MM
```

CSV-004では、
値を設定しない。

---

#### 15.2 asset_account_name

CSV上の物理名：

```text
asset_account_name
```

意味：

```text
保有商品が属する資産口座名
```

CSV-005およびCSV-006で、
操作対象利用者に属する
資産口座を特定するために使用する。

---

#### 15.3 holding_asset_name

CSV上の物理名：

```text
holding_asset_name
```

意味：

```text
商品別月末評価額を登録する保有商品名
```

CSV-005およびCSV-006で、
指定された資産口座に属する
保有商品を特定するために使用する。

---

#### 15.4 value

CSV上の物理名：

```text
value
```

意味：

```text
対象年月末時点の商品別月末評価額
```

Phase1では、
日本円の整数として扱う。

CSV-004では、
値を設定しない。

---

#### 15.5 返却しない情報

CSVテンプレートには、
以下の情報を返却しない。

- `user_id`
- `asset_account_id`
- `holding_asset_id`
- `month_end_asset_snapshot_id`
- `month_end_holding_value_id`
- `balance_recording_unit`
- `confirmed`
- `created_at`
- `updated_at`
- 利用可能資産設定
- 登録済み商品別月末評価額
- 登録済み資産口座名
- 登録済み保有商品名

CSV-004は、
商品別月末評価額CSVの
入力形式を提供することだけを
責務とする。

---

### 12 業務ルール

CSV-004では、
商品別月末評価額CSVを入力するための
固定形式のテンプレートを返却する。

本APIでは、
業務データの登録・更新・削除を行わない。

CSV-005 商品別月末評価額CSVプレビュー、
CSV-006 商品別月末評価額CSV登録で使用する
CSV形式の起点となる。

---

#### 12.1 CSVテンプレートは固定形式とする

商品別月末評価額CSVテンプレートは、
以下の固定ヘッダーを持つ。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

利用者や
登録済みデータによって
ヘッダー構成を変更しない。

---

#### 12.2 ヘッダー順序を固定する

ヘッダー順序は、
以下とする。

```text
1. target_year_month
2. asset_account_name
3. holding_asset_name
4. value
```

CSV-005およびCSV-006でも、
同じヘッダー名・順序を
正式なCSV形式として使用する。

---

#### 12.3 データ行を含めない

CSV-004では、
テンプレートに
データ行を含めない。

正常なテンプレートは、
以下の1行のみとする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

サンプルデータや
登録済みデータを
自動出力しない。

---

#### 12.4 利用者固有データを含めない

CSVテンプレートには、
操作対象利用者の

- 資産口座
- 保有商品
- 商品別月末評価額

を出力しない。

テンプレートは、
利用者に依存しない
固定形式とする。

---

#### 12.5 内部IDを使用しない

商品別月末評価額CSVでは、
資産口座および保有商品を
内部IDではなく名称で指定する。

テンプレートには、
以下の内部IDを含めない。

- `user_id`
- `asset_account_id`
- `holding_asset_id`
- `month_end_asset_snapshot_id`
- `month_end_holding_value_id`

CSV入力では、

```text
asset_account_name
holding_asset_name
```

を使用する。

内部IDへの変換は、
CSV-005およびCSV-006で
バックエンド側が行う。

---

#### 12.6 target_year_month

`target_year_month`には、
商品別月末評価額の
対象年月を入力する。

形式は、

```text
YYYY-MM
```

とする。

例：

```text
2026-07
```

1つのCSVファイルでは、
1つの対象年月のみを扱う。

CSV-004では、
値を設定せず
入力列のみを提供する。

---

#### 12.7 asset_account_name

`asset_account_name`には、
保有商品が属する
資産口座名を入力する。

CSV-005およびCSV-006では、

```text
操作対象利用者
+
asset_account_name
```

によって
対象資産口座を特定する。

他利用者の
同名資産口座を
対象としてはならない。

---

#### 12.8 holding_asset_name

`holding_asset_name`には、
商品別月末評価額を登録する
保有商品名を入力する。

CSV-005およびCSV-006では、

```text
操作対象利用者
+
資産口座
+
holding_asset_name
```

によって
対象保有商品を特定する。

同名の保有商品が
別の資産口座に存在する場合でも、
指定された資産口座に属する
保有商品を対象とする。

---

#### 12.9 value

`value`には、
対象年月末時点の
保有商品の評価額を入力する。

Phase1では、
日本円の整数として扱う。

正常例：

```text
1500000
0
```

以下のような形式は、
正式な入力形式としない。

```text
1,500,000
¥1500000
1500000.5
```

具体的な入力値検証は、
CSV-005およびCSV-006で行う。

---

#### 12.10 0円を有効値とする

商品別月末評価額では、

```text
value = 0
```

を有効な値として扱う。

0円を

- 未入力
- NULL
- 登録対象外

として扱わない。

---

#### 12.11 商品単位の資産口座を対象とする

商品別月末評価額CSVでは、
残高記録単位が
商品単位の資産口座に属する
保有商品を対象とする。

概念的には、

```text
asset_accounts.balance_recording_unit
    = 商品単位
```

であることを
CSV-005およびCSV-006で確認する。

CSV-004では、
この業務ルールの判定自体は行わない。

---

#### 12.12 保有商品単位で1行とする

CSVでは、
1つの保有商品について
1行を使用する。

概念例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

同一対象年月、
同一資産口座、
同一保有商品を
複数行に分けて登録しない。

重複判定は、
CSV-005およびCSV-006で行う。

---

#### 12.13 1ファイル1対象年月とする

1つのCSVファイルに、
複数の対象年月を混在させない。

正常例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-07,証券口座,全世界株式,1500000
2026-07,証券口座,S&P500,800000
```

不正例：

```csv
target_year_month,asset_account_name,holding_asset_name,value
2026-06,証券口座,全世界株式,1400000
2026-07,証券口座,S&P500,800000
```

CSV-004では
対象年月を受け取らないため、
この判定自体は行わない。

---

#### 12.14 CSV-005・CSV-006と同じ定義を使用する

CSV-004、
CSV-005、
CSV-006では、
商品別月末評価額CSVの
同一CSV定義を使用する。

概念的には、

```text
MonthEndHoldingValueCsvDefinition
    ├─ CSV-004
    ├─ CSV-005
    └─ CSV-006
```

とする。

これにより、

```text
CSV-004で取得したテンプレート
    ↓
CSV-005・CSV-006では形式不正
```

となる実装差異を防止する。

---

#### 12.15 テンプレート取得時に業務データを検索しない

CSV-004では、
テンプレート生成のために

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`

を検索しない。

テンプレート生成は、
業務データの状態に
依存させない。

---

#### 12.16 副作用を発生させない

CSV-004を何度実行しても、
業務データの状態は変化しない。

以下を行わない。

- 月末資産状況作成
- 商品別月末評価額作成
- 資産口座更新
- 保有商品更新
- 月末資産状況確定

テンプレート取得回数によって
テンプレート内容が変化することもない。

---

### 13 処理フロー

正常系の処理フローは、
以下とする。

```text
GET
/api/v1/month-end-holding-values/csv-template
    ↓
X-User-Id取得
    ↓
利用者コンテキスト検証
    ↓
商品別月末評価額CSV Definition取得
    ↓
CSVヘッダー生成
    ↓
CSVレスポンス生成
    ↓
200 OK
```

本APIでは、
テンプレート生成のための
業務データ検索を行わない。

---

#### 13.1 利用者コンテキスト確認

API共通処理で、
`X-User-Id`から
操作対象利用者を特定する。

以下の場合は、
CSV生成へ進まない。

```text
X-User-Id未指定
X-User-Id形式不正
利用者不存在
論理削除済み利用者
```

---

#### 13.2 CSV Definition取得

商品別月末評価額CSVの
共通Definitionから、
ヘッダー定義を取得する。

概念的には、
以下となる。

```text
target_year_month
asset_account_name
holding_asset_name
value
```

CSV-004専用に
別のヘッダー定義を
持たない。

---

#### 13.3 CSV生成

取得したヘッダー定義から、
CSVを生成する。

生成内容は、
以下とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

データ行は追加しない。

---

#### 13.4 CSVレスポンス生成

生成したCSVを、
ファイルとして取得可能な
HTTPレスポンスへ変換する。

正常時は、

```http
200 OK
```

を返却する。

レスポンス本文は、
JSONではなくCSVとする。

---

#### 13.5 異常系

利用者コンテキストに
問題がある場合は、
API共通の
JSONエラーレスポンスを返却する。

CSV生成処理中に
想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

として扱う。

---

### 14 正常レスポンス

正常時は、

```http
200 OK
```

を返却する。

レスポンス本文は、
CSVファイルとする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

本APIでは、
API共通の
JSON成功Envelopeを使用しない。

---

#### 14.1 Content-Type

正常レスポンスの
`Content-Type`は、

```http
text/csv; charset=UTF-8
```

を基本とする。

正式な文字コード指定は、
CSV共通仕様に従う。

---

#### 14.2 Content-Disposition

ブラウザから
CSVファイルとして
取得できるよう、

```http
Content-Disposition
```

を設定する。

概念例：

```http
Content-Disposition: attachment; filename="month-end-holding-values-template.csv"
```

正式なファイル名は、
CSV共通の命名方針に従う。

---

#### 14.3 レスポンスボディ

レスポンスボディは、
CSV文字列とする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

JSONへ変換しない。

---

#### 14.4 成功Envelope

本APIでは、
正常レスポンスが
CSVファイルそのものであるため、
以下のJSON成功Envelopeを使用しない。

```json
{
  "data": {},
  "requestId": "..."
}
```

CSVファイルへ
API管理用のメタ情報を
混在させない。

---

#### 14.5 requestId

正常レスポンスの
CSV本文には、

```text
requestId
```

を含めない。

リクエスト追跡情報を
レスポンスへ付与する場合は、
API共通方針に従って
HTTPレスポンスヘッダーで扱う。

異常時は、
API共通のJSONエラーレスポンスを使用する。

---

### 15 レスポンス項目

本APIの正常レスポンスは
JSONではなくCSVであるため、
JSONレスポンス項目は存在しない。

CSV列は、
以下とする。

| 順序 | CSV列名 | 内容 | 入力例 |
|---:|---|---|---|
| 1 | `target_year_month` | 商品別月末評価額の対象年月 | `2026-07` |
| 2 | `asset_account_name` | 保有商品が属する資産口座名 | `証券口座` |
| 3 | `holding_asset_name` | 商品別月末評価額を登録する保有商品名 | `全世界株式` |
| 4 | `value` | 対象年月末時点の商品別月末評価額 | `1500000` |

---

#### 15.1 target_year_month

CSV上の物理名：

```text
target_year_month
```

意味：

```text
商品別月末評価額の対象年月
```

入力形式：

```text
YYYY-MM
```

CSV-004では、
値を設定しない。

---

#### 15.2 asset_account_name

CSV上の物理名：

```text
asset_account_name
```

意味：

```text
保有商品が属する資産口座名
```

CSV-005およびCSV-006で、
操作対象利用者に属する
資産口座を特定するために使用する。

CSV-004では、
登録済み資産口座名を
出力しない。

---

#### 15.3 holding_asset_name

CSV上の物理名：

```text
holding_asset_name
```

意味：

```text
商品別月末評価額を登録する保有商品名
```

CSV-005およびCSV-006で、
指定された資産口座に属する
保有商品を特定するために使用する。

CSV-004では、
登録済み保有商品名を
出力しない。

---

#### 15.4 value

CSV上の物理名：

```text
value
```

意味：

```text
対象年月末時点の商品別月末評価額
```

Phase1では、
日本円の整数として扱う。

CSV-004では、
値を設定しない。

---

#### 15.5 返却しない情報

CSVテンプレートには、
以下の情報を返却しない。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`
- `asset_accounts.balance_recording_unit`
- `month_end_asset_snapshots.confirmed`
- `created_at`
- `updated_at`
- `asset_account_available_settings`
- `asset_accounts.name`
- `holding_assets.name`
- `month_end_holding_values.value`

CSV-004は、
商品別月末評価額CSVの
入力形式を提供することだけを
責務とする。

---

### 16 エラーレスポンス

CSV-004では、
正常時はCSVファイルを返却する。

一方、
エラー時はCSVではなく、
API共通方針に従った
JSONエラーレスポンスを返却する。

概念的には、
以下とする。

```text
正常時
    ↓
text/csv

エラー時
    ↓
application/json
```

本APIでは、
CSVファイルのアップロードや
CSV内容の検証を行わないため、
CSV入力内容に関する
業務エラーは発生しない。

---

#### 16.1 エラー一覧

本APIで想定する
主なエラーは、
以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`が指定されていない |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`の形式が不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 指定された利用者が存在しない、または論理削除済み |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | CSV生成処理などで想定外のエラーが発生した |

CSV-004では、
以下のようなエラーは発生しない。

- CSVヘッダー不正
- CSVデータ行不正
- 対象年月不正
- 資産口座不存在
- 保有商品不存在
- 月末評価額不正
- CSV内重複
- 月末資産状況確定済み
- 商品別月末評価額登録済み

これらは、
CSV-005およびCSV-006で扱う。

---

#### 16.2 USER_CONTEXT_REQUIRED

`X-User-Id`が
指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

HTTPステータスは、

```http
400 Bad Request
```

とする。

CSVテンプレート生成処理へ
進まない。

---

#### 16.3 INVALID_USER_ID

`X-User-Id`の形式が
不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

HTTPステータスは、

```http
400 Bad Request
```

とする。

例えば、
以下のような値を
不正とする。

```text
0
-1
abc
1.5
```

---

#### 16.4 USER_NOT_FOUND

指定された利用者が
存在しない場合、
または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

HTTPステータスは、

```http
404 Not Found
```

とする。

利用者が存在しない状態で
CSVテンプレートを
返却しない。

---

#### 16.5 INTERNAL_SERVER_ERROR

CSVテンプレート生成処理などで
想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

HTTPステータスは、

```http
500 Internal Server Error
```

とする。

レスポンスへ、
以下の内部情報を含めない。

- スタックトレース
- PHP内部エラー
- Laravel内部例外メッセージ
- サーバーファイルパス
- クラス名
- SQL
- データベース接続情報

詳細情報は、
サーバーログへ記録する。

---

#### 16.6 エラーレスポンス形式

エラー時は、
API共通の
JSONエラーレスポンスを使用する。

概念例：

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "指定された利用者が存在しません。"
  },
  "requestId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

正常時のCSVレスポンス形式と
エラー時のJSON形式を
混在させない。

---

### 17 HTTPステータス

本APIで使用する
HTTPステータスは、
以下とする。

| HTTPステータス | 用途 |
|---|---|
| `200 OK` | CSVテンプレート取得成功 |
| `400 Bad Request` | 利用者コンテキストの指定不備 |
| `404 Not Found` | 指定された利用者が存在しない |
| `500 Internal Server Error` | 想定外のサーバー内部エラー |

本APIでは、
通常、

```text
409 Conflict
422 Unprocessable Entity
```

は使用しない。

---

#### 17.1 200 OK

CSVテンプレートを
正常に生成できた場合は、

```http
200 OK
```

を返却する。

レスポンス本文は、
以下のCSVとする。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

---

#### 17.2 400 Bad Request

以下の場合に使用する。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
```

CSVテンプレートの
内容に関するエラーには使用しない。

---

#### 17.3 404 Not Found

指定された利用者が
存在しない場合、
または論理削除済みの場合に使用する。

```text
USER_NOT_FOUND
```

---

#### 17.4 500 Internal Server Error

CSV生成処理などで
想定外のエラーが発生した場合に使用する。

```text
INTERNAL_SERVER_ERROR
```

正常な空CSVへ
フォールバックしない。

---

### 18 副作用

本APIには、
業務データに対する
副作用はない。

CSV-004の実行によって、
データベースの
登録・更新・削除を行わない。

---

#### 18.1 更新しないテーブル

本APIでは、
以下のテーブルを更新しない。

- `users`
- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_account_available_settings`
- `net_incomes`
- `objectives`
- `assessment_histories`

CSVテンプレート取得によって、
業務データの状態を
変更してはならない。

---

#### 18.2 CSVインポート履歴を作成しない

Phase1では、
CSVテンプレート取得履歴を
データベースへ保存しない。

以下のような
履歴データを作成しない。

```text
csv_download_histories
csv_template_histories
csv_import_sessions
```

必要なアクセス記録は、
API共通の
サーバーログで扱う。

---

#### 18.3 ファイルを永続保存しない

生成したCSVテンプレートを、
サーバー上の
永続ファイルとして保存しない。

例えば、

```text
storage/app/templates/...
```

へ
リクエストごとに
ファイルを作成する方式は
採用しない。

レスポンス用に
メモリまたは一時ストリーム上で
生成する。

---

### 19 トランザクション

CSV-004では、
データベース更新を行わないため、
明示的な
トランザクションを使用しない。

以下のような実装は
行わない。

```php
DB::transaction(
    function () {
        // CSVテンプレート生成
    },
);
```

テンプレート生成処理に
トランザクションは不要である。

---

#### 19.1 読み取りトランザクションも使用しない

CSV-004では、
テンプレート生成のために
業務データを検索しない。

そのため、
読み取り整合性を目的とした
トランザクションも不要とする。

---

### 20 ロック

CSV-004では、
データベースの
行ロックを使用しない。

以下を使用しない。

```php
lockForUpdate()
```

または、

```sql
SELECT ... FOR UPDATE
```

本APIの実行によって、
CSV-005、
CSV-006、
月末資産状況関連処理などを
ブロックしてはならない。

---

### 21 キャッシュ

Phase1では、
CSV-004専用の
サーバー側アプリケーションキャッシュを
使用しない。

CSVテンプレートは
ヘッダー1行のみであり、
生成コストが小さいためである。

また、
CSV仕様変更時に
古いテンプレートが
キャッシュされ続ける
複雑性を避ける。

---

#### 21.1 テンプレート内容

CSVテンプレートは
利用者固有データに依存しないため、
同じバージョンのAPIでは
同一内容となる。

概念的には、
常に以下を返却する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

---

#### 21.2 将来のHTTPキャッシュ

将来的に
必要となった場合は、

```text
ETag
Cache-Control
```

などの
HTTPキャッシュを検討してよい。

ただし、
Phase1では実装しない。

---

### 22 冪等性

本APIは、
参照専用の`GET` APIであり、
冪等である。

同じ条件で
複数回実行しても、
業務データの状態は変化しない。

---

#### 22.1 同一利用者での複数回実行

同じ

```text
X-User-Id
```

で
複数回実行した場合は、
同じCSVテンプレートを返却する。

概念的には、

```text
1回目
    ↓
target_year_month,asset_account_name,holding_asset_name,value

2回目
    ↓
target_year_month,asset_account_name,holding_asset_name,value
```

となる。

---

#### 22.2 利用者が異なる場合

CSVテンプレートは
利用者固有データを含まないため、
有効な利用者であれば
利用者が異なっても
同じテンプレートを返却する。

例えば、

```text
User A
    ↓
同じテンプレート

User B
    ↓
同じテンプレート
```

とする。

---

#### 22.3 業務データ変更の影響を受けない

CSV-004のテンプレート内容は、

- 資産口座の追加
- 資産口座の削除
- 保有商品の追加
- 保有商品の削除
- 商品別月末評価額の登録
- 月末資産状況の確定

などによって
変化しない。

CSV仕様そのものが
変更された場合のみ、
テンプレート内容を変更する。

---

#### 22.4 Idempotency-Key

本APIは
`GET`かつ参照専用であるため、

```text
Idempotency-Key
```

を使用しない。

Idempotency Keyによる
重複実行制御は不要である。

---

#### 22.5 複数回ダウンロード

利用者が
同じテンプレートを
複数回ダウンロードしても
問題ない。

ダウンロード回数によって、

- CSV内容
- 業務データ
- 利用者状態

を変更しない。

---

#### 22.6 CSV-005・CSV-006との関係

CSV-004を
何度取得しても、
CSV-005およびCSV-006の
登録・検証状態には影響しない。

CSV-004は、
CSV入力形式を提供することだけに
責務を限定する。

---

### 23 関連テーブル

CSV-004では、
商品別月末評価額CSVテンプレートを
固定形式で生成するため、
業務データを保持するテーブルを
直接参照しない。

テンプレート取得時に、
以下のテーブルから
CSVデータ行を生成しない。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`
- `asset_account_available_settings`

ただし、
`X-User-Id`による
利用者コンテキストの確認では、
API共通処理として
`users`を参照する。

---

#### 23.1 users

`X-User-Id`で指定された
利用者の存在確認に使用する。

概念的な参照条件は、
以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

CSVテンプレートの
内容生成には使用しない。

---

#### 23.2 asset_accounts

CSV-004では、
`asset_accounts`を参照しない。

以下の情報を
テンプレートへ出力しない。

- `asset_accounts.id`
- `asset_accounts.name`
- `asset_accounts.balance_recording_unit`

商品単位の資産口座が
存在しない場合でも、
テンプレートは取得可能とする。

---

#### 23.3 holding_assets

CSV-004では、
`holding_assets`を参照しない。

以下の情報を
テンプレートへ出力しない。

- `holding_assets.id`
- `holding_assets.name`

登録済み保有商品の
一覧を取得して
CSVデータ行へ展開する処理は行わない。

---

#### 23.4 month_end_asset_snapshots

CSV-004では、
`month_end_asset_snapshots`を参照しない。

以下の情報によって
テンプレート取得可否を
変更しない。

- 対象年月のsnapshot存在有無
- `month_end_asset_snapshots.confirmed`

月末資産状況が
1件も存在しない利用者でも、
テンプレートは取得可能とする。

---

#### 23.5 month_end_holding_values

CSV-004では、
`month_end_holding_values`を参照しない。

登録済みの
商品別月末評価額を
テンプレートへ出力しない。

CSV-004は、
既存データのエクスポートAPIではなく、
CSVインポート用テンプレートの
取得APIである。

---

#### 23.6 asset_account_available_settings

CSV-004では、
`asset_account_available_settings`を
参照しない。

利用可能資産としての
設定状態は、
CSVテンプレートの形式に
影響しない。

---

#### 23.7 テーブル更新

本APIでは、
関連するすべてのテーブルに対して
INSERT、
UPDATE、
DELETEを行わない。

概念的には、

```text
users
    → SELECTのみ

その他の業務テーブル
    → 参照なし・更新なし
```

とする。

---

### 24 インデックス

CSV-004専用の
インデックスは追加しない。

本APIでは、
テンプレート生成のために
業務テーブルを検索しないためである。

---

#### 24.1 users

利用者存在確認では、

```text
users.id
```

を使用する。

`id`は主キーであるため、
主キーインデックスを利用する。

CSV-004のために
追加インデックスを作成しない。

---

#### 24.2 業務テーブル

以下のテーブルについて、
CSV-004を理由とした
インデックス追加は行わない。

- `asset_accounts`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_holding_values`
- `asset_account_available_settings`

CSV-005、
CSV-006などで必要となる
インデックスについては、
各APIの検索条件をもとに
別途判断する。

---

### 25 性能

CSV-004は、
固定されたCSVヘッダーのみを
返却するため、
処理負荷は小さい。

概念的な処理は、

```text
利用者コンテキスト確認
    ↓
CSV Definition取得
    ↓
CSVヘッダー生成
    ↓
レスポンス返却
```

のみとする。

---

#### 25.1 業務データ検索を行わない

テンプレート生成時に、

```text
asset_accounts
holding_assets
month_end_asset_snapshots
month_end_holding_values
```

などを検索しない。

そのため、
利用者が保有する

- 資産口座数
- 保有商品数
- 商品別月末評価額件数

によって、
CSV-004の処理量が
増加しない設計とする。

---

#### 25.2 N+1問題

CSV-004では、
業務データの一覧取得を行わないため、
N+1問題は発生しない。

以下のような処理は行わない。

```text
資産口座一覧取得
    ↓
各資産口座ごとに
保有商品一覧取得
```

---

#### 25.3 CSV生成

CSVテンプレートは、
ヘッダー1行のみであるため、
大規模なメモリ使用を伴わない。

概念的には、
以下の固定内容を生成する。

```csv
target_year_month,asset_account_name,holding_asset_name,value
```

大規模CSV向けの
chunk処理や
ストリーミング処理は
Phase1では不要とする。

---

#### 25.4 ファイルI/O

CSVテンプレートを
サーバー上へ
永続ファイルとして保存しない。

概念的には、

```text
CSV Definition
    ↓
メモリ上でCSV生成
    ↓
HTTPレスポンス
```

とする。

以下のような処理は
原則として行わない。

```text
CSV生成
    ↓
storageへ保存
    ↓
ファイル再読込
    ↓
レスポンス
```

---

#### 25.5 キャッシュ

CSVテンプレートは
固定かつ生成コストが小さいため、
Phase1では
専用キャッシュを導入しない。

キャッシュ管理による
複雑性を増やさないことを
優先する。

---

### 26 セキュリティ

CSV-004では、
CSVテンプレート自体に
利用者固有情報を含めない。

ただし、
Phase1のAPI共通方針として
`X-User-Id`による
利用者コンテキストの検証を行う。

---

#### 26.1 利用者境界

`X-User-Id`で
指定された利用者が
有効であることを確認する。

CSV-004では
利用者固有の業務データを
取得しないため、
他利用者の

- 資産口座
- 保有商品
- 商品別月末評価額

がレスポンスへ
混入することはない。

---

#### 26.2 内部IDを公開しない

CSVテンプレートには、
以下の内部IDを含めない。

- `users.id`
- `asset_accounts.id`
- `holding_assets.id`
- `month_end_asset_snapshots.id`
- `month_end_holding_values.id`

CSV入力では、

```text
asset_account_name
holding_asset_name
```

を使用する。

---

#### 26.3 業務データを公開しない

CSVテンプレートには、
以下の既存業務データを含めない。

- `asset_accounts.name`
- `holding_assets.name`
- `month_end_holding_values.value`
- `asset_accounts.balance_recording_unit`
- `month_end_asset_snapshots.confirmed`
- `asset_account_available_settings`

CSV-004を
業務データの取得手段として
利用できない設計とする。

---

#### 26.4 CSV Injection

CSV-004では、
利用者入力値や
データベース値を
CSVデータ行へ出力しない。

返却内容は、
システムで定義した
固定ヘッダーのみである。

そのため、
Phase1のCSV-004では
CSV Injection対策としての
利用者入力値エスケープ処理は
不要とする。

CSVへ業務データを出力するAPIを
将来追加する場合は、
別途CSV Injection対策を検討する。

---

#### 26.5 Content-Disposition

`Content-Disposition`の
ファイル名は、
システム側で定義した
固定値を使用する。

利用者入力値を
そのままファイル名へ
埋め込まない。

概念例：

```http
Content-Disposition: attachment; filename="month-end-holding-values-template.csv"
```

---

#### 26.6 エラー情報

想定外エラー時に、
以下の内部情報を
レスポンスへ含めない。

- スタックトレース
- PHP内部エラー
- Laravel内部例外メッセージ
- サーバーファイルパス
- クラス名
- SQL
- データベース接続情報

詳細情報は、
サーバーログへ記録する。

---

### 27 ログ

CSV-004では、
API共通ログ方針に従って
必要な情報を記録する。

概念的なログコンテキストは、
以下とする。

```text
requestId
userId
apiId
```

`apiId`は、

```text
CSV-004
```

とする。

---

#### 27.1 正常時

正常時は、
必要に応じて
以下を記録する。

```text
requestId
userId
apiId = CSV-004
httpStatus = 200
```

CSVテンプレート本文そのものを
ログへ記録する必要はない。

---

#### 27.2 異常時

異常時は、
必要に応じて
以下を記録する。

```text
requestId
userId
apiId
errorCode
httpStatus
```

想定外例外については、
調査に必要な情報を
サーバーログへ記録する。

---

#### 27.3 ログへ記録しない情報

以下の情報を
不要にログへ記録しない。

- CSVテンプレート全文
- 他利用者の情報
- データベース接続情報
- 不要なHTTPヘッダー全文

`X-User-Id`については、
API共通ログ方針に従って
操作対象利用者IDとして扱う。

---

### 28 設計上の補足

#### 28.1 GETを採用する理由

CSV-004は、
固定されたCSVテンプレートを
取得するだけであり、
業務データを変更しない。

そのため、
HTTPメソッドには
`GET`を採用する。

---

#### 28.2 JSONではなくCSVを直接返す理由

本APIの目的は、
利用者がそのまま編集できる
CSVファイルを提供することである。

JSONで

```json
{
  "headers": [
    "target_year_month",
    "asset_account_name",
    "holding_asset_name",
    "value"
  ]
}
```

を返却し、
React側でCSVへ変換する方式は
採用しない。

CSV生成責務を
バックエンドへ集約する。

---

#### 28.3 利用者固有データを含めない理由

CSV-004は、
テンプレート取得APIであり、
業務データのエクスポートAPIではない。

利用者固有データを
テンプレートへ含めると、

- 資産口座検索
- 保有商品検索
- 対象年月判定
- 利用者ごとのテンプレート差異

が発生し、
責務が増える。

Phase1では、
固定形式のテンプレートに限定する。

---

#### 28.4 資産口座名をテンプレートへ展開しない理由

登録済み資産口座名を
テンプレートへ事前展開すると、
CSV取得時点と
CSV登録時点で
資産口座状態が変化する可能性がある。

また、
商品単位CSVでは
資産口座と保有商品の
組み合わせも必要になる。

そのため、
CSV-004では
固定ヘッダーのみを返却し、
正式な存在確認は
CSV-005・CSV-006で行う。

---

#### 28.5 保有商品名をテンプレートへ展開しない理由

保有商品一覧を
CSVテンプレートへ出力すると、
CSV-004が
業務データ取得APIとしての
責務を持つことになる。

Phase1では、
CSVテンプレート生成と
業務データ参照を分離するため、
保有商品名を
自動出力しない。

---

#### 28.6 内部IDをCSVへ含めない理由

利用者にとって、
`asset_account_id`や
`holding_asset_id`は
意味を理解しにくい内部値である。

CSVでは、

```text
asset_account_name
holding_asset_name
```

を使用する。

バックエンド側で
操作対象利用者との関連を確認しながら
内部IDへ変換する。

---

#### 28.7 CSV-004・CSV-005・CSV-006でDefinitionを共通化する理由

テンプレート生成と
CSV受付側で
ヘッダー定義を別々に持つと、

```text
CSV-004
4列

CSV-005
別の4列

CSV-006
さらに別定義
```

という不整合が発生する可能性がある。

そのため、
共通の

```text
MonthEndHoldingValueCsvDefinition
```

へ集約する。

---

#### 28.8 GeneratorとParserを分離する理由

CSV-004は
CSVを生成する。

CSV-005・CSV-006は
CSVを解析する。

責務が異なるため、

```text
Generator
    → CSV出力

Parser
    → CSV入力
```

として分離する。

共通Definitionだけを
共有する。

---

#### 28.9 FormRequestを作成しない理由

CSV-004には、

- パスパラメータ
- クエリパラメータ
- リクエストボディ

が存在しない。

CSV-004専用の
入力バリデーションがないため、
空のFormRequestを作成しない。

利用者コンテキストは
共通Middlewareが担当する。

---

#### 28.10 API Resourceを使用しない理由

Laravel API Resourceは、
JSONレスポンス形式へ
データを変換する用途に適している。

CSV-004の正常レスポンスは
CSVファイルであるため、
API Resourceを経由させない。

CSVファイル用Responderから
直接レスポンスを生成する。

---

#### 28.11 Repository・Queryを使用しない理由

CSV-004では、
テンプレート生成のための
業務データ検索を行わない。

そのため、
CSV-004専用の

```text
Repository
Query
```

を作成しない。

不要な層を追加せず、
テンプレート生成に必要な
最小構成とする。

---

#### 28.12 Blobとして扱う理由

ブラウザで
CSVファイルとして保存するため、
フロントエンドでは
レスポンスを`Blob`として扱う。

CSV本文を
通常のJSONレスポンスとして
扱わない。

---

#### 28.13 ダウンロードAPIのエラー処理を共通化する理由

CSV-004のように、

```text
成功時
    → CSV

失敗時
    → JSON
```

となるAPIでは、
通常のJSON APIと
エラー処理方法が異なる場合がある。

CSV-001など
同様のダウンロードAPIでも
同じ問題が発生するため、
Blobレスポンスから
JSONエラーを解析する処理は
共通化してよい。

---

#### 28.14 キャッシュを使用しない理由

テンプレート生成コストは
非常に小さい。

Phase1で
キャッシュを導入すると、

- CSV仕様変更時の無効化
- APIバージョンとの対応
- キャッシュ制御

が追加で必要になる。

そのため、
単純な都度生成を採用する。

---

#### 28.15 Idempotency-Keyを使用しない理由

CSV-004は
副作用のないGET APIであり、
同じリクエストを
複数回実行しても
業務データが変化しない。

そのため、

```text
Idempotency-Key
```

は不要である。

---

### 29 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [エラーコード一覧](../../error-codes.md)
- [CSV-005 商品別月末評価額CSVプレビュー](./csv-005-preview.md)
- [CSV-006 商品別月末評価額CSV登録](./csv-006-create.md)
- [CSV-001 月末資産残高CSVテンプレート取得](./csv-001-create.md)
- [CSV-002 月末資産残高CSVプレビュー](./csv-002-preview.md)
- [CSV-003 月末資産残高CSV登録](./csv-003-create.md)
- [SNP-003 月末資産状況詳細取得](../month-end-assets/snp-003-detail.md)
- [SNP-004 月末資産状況確定](../month-end-assets/snp-004-confirm.md)
- [VAL-001 商品別月末評価額一覧取得](../month-end-assets/val-001-list.md)
- [商品別月末評価額API詳細](../month-end-assets/README.md)
- [資産口座API詳細](../asset-accounts/README.md)
- [保有商品API詳細](../holding-assets/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)
- [Laravel CSVインポート設計](../../../architecture/laravel/csv-imports.md)
- [React CSVインポート設計](../../../architecture/react/csv-imports.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)