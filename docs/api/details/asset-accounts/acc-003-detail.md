# ACC-003 資産口座詳細取得

## 1. 概要

操作対象となる利用者に帰属する指定された資産口座の詳細情報を取得する。

資産口座の基本情報に加えて、現在の利用可能資産区分を返却する。

主に以下の情報を取得する。

* 資産口座ID
* 資産口座名
* 資産種別
* 残高記録単位
* 利用開始年月
* 現在の利用可能資産区分
* 利用状態

概念的には、以下の情報を組み合わせてレスポンスを生成する。

```text
asset_accounts
    +
現在有効な
asset_account_available_settings
    ↓
資産口座詳細
```

本APIは参照専用であり、資産口座、利用可能資産設定、月末資産データなどの業務データを更新しない。

---

## 2. ユースケース

利用者は、資産口座一覧などから特定の資産口座を選択し、詳細情報を確認する。

主に以下の用途で使用する。

* 資産口座の登録内容を確認する
* 資産口座編集画面の初期値を取得する
* 資産種別を確認する
* 残高記録単位を確認する
* 利用開始年月を確認する
* 現在の利用可能資産区分を確認する
* 現在の利用状態を確認する

概念的な画面利用は、以下とする。

```text
ACC-001
資産口座一覧取得
    ↓
利用者が資産口座を選択
    ↓
ACC-003
資産口座詳細取得
    ↓
詳細表示
または
編集画面初期表示
```

---

## 3. エンドポイント

```http
GET /api/v1/asset-accounts/{assetAccountId}
```

`assetAccountId`には、取得対象となる資産口座IDを指定する。

---

## 4. HTTPメソッド

```text
GET
```

本APIは、指定された資産口座の詳細情報を取得する参照APIであるため、`GET`を使用する。

本APIの実行によって、以下の業務データを更新しない。

* `asset_accounts`
* `asset_account_available_settings`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`

---

## 5. 利用者コンテキスト

本APIは利用者依存APIのため、`X-User-Id`を必須とする。

操作対象となる利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者コンテキストの特定、`X-User-Id`の検証および利用者境界については、[API共通方針](../api-common-policy.md)に従う。

指定された`assetAccountId`に対応する資産口座が操作対象利用者に帰属する場合のみ、詳細情報を返却する。

概念的には、以下の条件を満たす必要がある。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID
```

---

### 5.1 利用者境界

ACC-003では、`assetAccountId`だけを条件として資産口座を取得してはならない。

以下のような取得は基本としない。

```text
asset_accounts.id
    = assetAccountId
```

操作対象利用者を含め、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID
```

として取得する。

これにより、他利用者の資産口座を参照できないようにする。

---

### 5.2 他利用者の資産口座

指定された`assetAccountId`が他利用者に帰属する場合は、対象が存在しないものとして扱う。

概念的には、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

とする。

以下のような専用エラーは返却しない。

```text
FORBIDDEN
OTHER_USER_ASSET_ACCOUNT
```

これにより、他利用者の資産口座が存在するかどうかを外部から推測しにくくする。

---

### 5.3 論理削除済み資産口座

ACC-003は、通常利用中の資産口座詳細を取得するAPIとする。

そのため、

```text
asset_accounts.deleted_at IS NOT NULL
```

の資産口座は通常の取得対象に含めない。

論理削除済み資産口座が指定された場合は、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 5.4 現在の利用可能資産区分

ACC-003では、資産口座の詳細情報とあわせて現在の利用可能資産区分を返却する。

利用可能資産区分は、

```text
asset_accounts
```

の固定カラムではなく、

```text
asset_account_available_settings
```

の履歴から取得する。

概念的には、現在年月を基準として

```text
start_year_month
    <= 現在年月

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= 現在年月
)
```

を満たす設定を取得する。

---

### 5.5 利用可能資産設定の利用者境界

`asset_account_available_settings`には直接`user_id`を持たない。

そのため、利用可能資産設定は取得対象の

```text
asset_accounts.id
```

を通して利用者境界を保証する。

概念的には、

```text
操作対象利用者
    ↓
asset_accounts.user_id
    ↓
asset_accounts.id
    ↓
asset_account_available_settings.asset_account_id
```

となる。

他利用者の資産口座に属する利用可能資産設定を取得してはならない。

---

### 5.6 利用可能資産設定が存在しない場合

ACC-002 資産口座登録では、資産口座登録時に初期利用可能資産設定を必ず1件登録する設計とする。

そのため、通常状態では現在の利用可能資産設定が取得できることを前提とする。

もし、データ不整合などにより対象年月に有効な利用可能資産設定が存在しない場合は、任意のデフォルト値へ補完しない。

例えば、

```text
isAvailable = false
```

を暗黙的に返却してはならない。

具体的な異常時の扱いは、後続の「エラーレスポンス」および「Laravel実装方針」で定義する。

---

### 5.7 X-User-Idが不正な場合

以下の場合は、資産口座検索へ進まない。

* `X-User-Id`が指定されていない
* `X-User-Id`の形式が不正
* 指定された利用者が存在しない
* 指定された利用者が論理削除済み

利用者コンテキストに関するエラーコードおよびHTTPステータスは、API共通方針に従う。

---

### 5.8 1リクエスト1利用者

1回のACC-003リクエストでは、`X-User-Id`で指定された1利用者の資産口座だけを扱う。

以下のように、URLやクエリパラメータから別の利用者IDを指定する方式は採用しない。

```text
/api/v1/users/{userId}/asset-accounts/{assetAccountId}
```

または、

```text
?userId=1
```

利用者の指定経路は、`X-User-Id`へ統一する。

---

## 6. パスパラメータ

本APIでは、取得対象となる資産口座を指定するために、以下のパスパラメータを使用する。

| パラメータ            | 型      |  必須 | 説明            |
| ---------------- | ------ | :-: | ------------- |
| `assetAccountId` | string |  ○  | 取得対象となる資産口座ID |

エンドポイントは、以下とする。

```http
GET /api/v1/asset-accounts/{assetAccountId}
```

---

### 6.1 assetAccountId

`assetAccountId`には、取得対象となる資産口座IDを指定する。

例：

```http
GET /api/v1/asset-accounts/10
```

APIでは主キーを文字列として扱うが、値としては正の整数形式を前提とする。

正常例：

```text
1
10
123
```

不正例：

```text
0
-1
abc
1.5
```

---

### 6.2 利用者境界

`assetAccountId`だけでは取得対象を確定しない。

実際の資産口座取得では、操作対象利用者IDと組み合わせて検索する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

他利用者に属する資産口座IDが指定された場合は、対象が存在しないものとして扱う。

---

## 7. クエリパラメータ

なし。

ACC-003では、指定された資産口座の現在の詳細情報を取得するため、検索条件や表示切替用のクエリパラメータを使用しない。

以下のようなクエリパラメータは受け付けない。

* `userId`
* `targetYearMonth`
* `includeDeleted`
* `includeSettings`
* `isEnabled`

利用者は`X-User-Id`から特定し、取得対象資産口座は`assetAccountId`から特定する。

---

## 8. リクエストヘッダー

以下のリクエストヘッダーを使用する。

| ヘッダー名       |  必須 | 説明                      |
| ----------- | :-: | ----------------------- |
| `X-User-Id` |  ○  | 操作対象となる利用者ID            |
| `Accept`    |  ○  | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/asset-accounts/10
Accept: application/json
X-User-Id: 1
```

本APIではリクエストボディを使用しないため、

```http
Content-Type: application/json
```

は必須としない。

---

### 8.1 X-User-Id

`X-User-Id`は、API共通方針に従って検証する。

本APIの処理開始前に、共通Middlewareで以下を確認する。

* ヘッダーが指定されていること
* IDが共通方針で定めた形式であること
* 利用者が存在すること
* 利用者が論理削除されていないこと

検証済みの利用者IDを利用者コンテキストへ保持し、Action以降で使用する。

---

## 9. リクエストボディ

なし。

ACC-003は参照専用のGET APIであるため、リクエストボディを使用しない。

以下のような情報をリクエストボディから受け付けない。

* `userId`
* `assetAccountId`
* `targetYearMonth`
* `isAvailable`
* `isEnabled`

取得対象は、

```text
X-User-Id
+
assetAccountId
```

によって決定する。

---

## 10. リクエスト項目

本APIでは、リクエストボディ項目は存在しない。

クライアントが指定する業務上の入力値は、以下のみとする。

```text
パスパラメータ
    assetAccountId

リクエストヘッダー
    X-User-Id
```

---

## 11. バリデーション

ACC-003では、主に以下を検証する。

```text
X-User-Id
assetAccountId
```

資産口座の存在確認や利用者境界確認については、単純な入力形式検証ではなく、業務データ取得時に確認する。

---

### 11.1 X-User-Id

`X-User-Id`について、以下を検証する。

* 必須であること
* 共通ID形式に一致すること
* 正の整数として扱えること
* 指定された利用者が存在すること
* 指定された利用者が論理削除されていないこと

不正な場合は、資産口座検索へ進まない。

---

### 11.2 assetAccountId 必須

`assetAccountId`は、パスパラメータとして必須とする。

エンドポイント上、`assetAccountId`が存在しない場合は、

```http
GET /api/v1/asset-accounts
```

となり、ACC-001 資産口座一覧取得として扱われる。

ACC-003では、必ず

```http
GET /api/v1/asset-accounts/{assetAccountId}
```

を使用する。

---

### 11.3 assetAccountId 形式

`assetAccountId`は、正の整数形式であることを必須とする。

正常例：

```text
1
10
999
```

不正例：

```text
0
-1
abc
1.5
1e3
```

形式不正の場合は、資産口座検索へ進まない。

---

### 11.4 assetAccountIdの型変換

Laravel内部では、パスパラメータを文字列として受け取る場合がある。

形式検証後に、必要に応じて整数へ変換して使用する。

不正な文字列をPHPの暗黙変換によって有効なIDとして扱ってはならない。

例えば、

```text
10abc
```

を

```text
10
```

として扱わない。

---

### 11.5 資産口座存在確認

指定された`assetAccountId`について、操作対象利用者に属する有効な資産口座が存在することを確認する。

概念的な条件は、以下とする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

該当する資産口座が存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 11.6 他利用者の資産口座

指定された`assetAccountId`が他利用者に属する場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下のような他利用者専用エラーは返却しない。

```text
FORBIDDEN
OTHER_USER_ASSET_ACCOUNT
```

これにより、他利用者のリソース存在有無を外部へ公開しない。

---

### 11.7 論理削除済み資産口座

対象資産口座が

```text
deleted_at IS NOT NULL
```

の場合は、通常の詳細取得対象に含めない。

この場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

ACC-003では、

```php
withTrashed()
```

を使用して論理削除済み資産口座を通常取得対象に含めない。

---

### 11.8 現在の利用可能資産設定

対象資産口座について、現在有効な

```text
asset_account_available_settings
```

を取得する。

現在年月を基準として、概念的には以下を満たす設定を対象とする。

```text
start_year_month
    <= 現在年月

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= 現在年月
)
```

現在有効な設定が複数件存在しないことを前提とする。

---

### 11.9 利用可能資産設定不存在

ACC-002では、資産口座登録時に初期利用可能資産設定を必ず登録する。

そのため、通常状態では現在有効な利用可能資産設定が1件取得できることを前提とする。

もし、データ不整合によって現在有効な設定が存在しない場合は、

```text
isAvailable = false
```

などのデフォルト値へ補完しない。

業務データ不整合として扱う。

具体的なエラーコードは、「エラーレスポンス」で定義する。

---

### 11.10 利用可能資産設定が複数存在する場合

同一資産口座について、現在年月に有効な利用可能資産設定が複数存在する状態は、正常な業務状態として扱わない。

例えば、

```text
設定A
2026-01 ～ NULL

設定B
2026-06 ～ NULL
```

のように、同時に複数設定が有効となる状態はデータ不整合である。

この場合も、任意の1件を選択してレスポンスを返してはならない。

具体的な異常時の扱いは、「エラーレスポンス」および「Laravel実装方針」で定義する。

---

### 11.11 targetYearMonthを受け付けない

ACC-003は、「現在の資産口座詳細」を取得するAPIとする。

そのため、

```text
targetYearMonth
```

を指定して過去時点の利用可能資産区分を取得する機能は持たせない。

過去年月時点の利用可能資産設定が必要な処理では、各業務API側で対象年月を基準として判定する。

---

### 11.12 isEnabledを入力させない

利用状態は、

```text
asset_accounts.deleted_at
```

からサーバー側で判定する。

クライアントから

```text
isEnabled
```

を指定して取得対象を切り替えることはできない。

ACC-003では、有効な資産口座だけを取得対象とする。

---

### 11.13 バリデーション失敗時

以下のいずれかに該当する場合は、正常レスポンスを返却しない。

* `X-User-Id`不正
* `assetAccountId`形式不正
* 利用者不存在
* 資産口座不存在
* 他利用者の資産口座
* 論理削除済み資産口座
* 利用可能資産設定の不整合

ACC-003は参照APIであるため、バリデーション失敗時にも業務データは更新しない。

---

## 12. 業務ルール

ACC-003では、操作対象利用者に帰属する指定された資産口座の詳細情報を取得する。

資産口座の基本情報に加えて、現在年月において有効な利用可能資産設定を取得し、現在の利用可能資産区分として返却する。

本APIは参照専用であり、業務データを更新しない。

---

### 12.1 操作対象利用者に帰属する資産口座だけを取得する

取得対象となる資産口座は、以下の条件をすべて満たすものとする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

`assetAccountId`だけを条件として資産口座を取得してはならない。

---

### 12.2 他利用者の資産口座を取得しない

指定された`assetAccountId`が他利用者に帰属する場合は、対象資産口座が存在しないものとして扱う。

概念的には、

```text
X-User-Id = 1

assetAccountId = 10

asset_accounts.id = 10
asset_accounts.user_id = 2
    ↓
取得対象外
    ↓
ASSET_ACCOUNT_NOT_FOUND
```

とする。

他利用者の資産口座が存在することを示す情報をレスポンスへ含めない。

---

### 12.3 論理削除済み資産口座を取得しない

ACC-003では、利用中の資産口座だけを詳細取得対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は取得対象外とする。

論理削除済み資産口座が指定された場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 12.4 資産口座の基本情報を取得する

取得対象資産口座から、主に以下の情報を取得する。

```text
id
name
asset_type
balance_recording_unit
start_year_month
deleted_at
```

`deleted_at`は、APIレスポンスへそのまま返却するためではなく、利用状態の判定に使用する。

---

### 12.5 現在の利用可能資産区分を取得する

利用可能資産区分は、

```text
asset_accounts
```

の固定属性として保持せず、

```text
asset_account_available_settings
```

から取得する。

ACC-003では、現在年月に有効な設定を取得し、

```text
isAvailable
```

として返却する。

---

### 12.6 現在年月

利用可能資産設定の判定に使用する現在年月は、サーバー側の現在日時から決定する。

概念的には、

```text
現在日
2026-08-11
    ↓
現在年月
2026-08
```

とする。

クライアントから現在年月を指定させない。

---

### 12.7 現在有効な利用可能資産設定

現在有効な利用可能資産設定は、概念的に以下の条件を満たすものとする。

```text
asset_account_id
    = 対象資産口座ID

AND

start_year_month
    <= 現在年月

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= 現在年月
)
```

この条件を満たす設定が1件存在することを正常状態とする。

---

### 12.8 利用可能資産設定の開始年月

以下の場合は、現在有効な設定として扱う。

```text
start_year_month
    = 現在年月
```

例えば、

```text
現在年月
    = 2026-08

start_year_month
    = 2026-08
```

の場合、当該設定は2026年8月から有効であるため、取得対象とする。

---

### 12.9 利用可能資産設定の終了年月

`end_year_month`が設定されている場合、その年月までは当該設定を有効とする。

例えば、

```text
start_year_month = 2026-01
end_year_month   = 2026-08
```

の場合、

```text
2026-08
```

では有効とする。

```text
2026-09
```

では有効としない。

---

### 12.10 end_year_monthがNULLの場合

```text
end_year_month IS NULL
```

の場合は、終了年月が設定されていない現在有効な設定として扱う。

概念的には、

```text
start_year_month = 2026-08
end_year_month   = NULL
    ↓
2026-08以降有効
```

となる。

---

### 12.11 利用可能資産設定が存在しない場合

ACC-002では、資産口座登録時に初期利用可能資産設定を必ず登録する。

そのため、利用開始済みの資産口座については、通常、現在有効な利用可能資産設定が1件存在することを前提とする。

現在有効な設定が存在しない場合は、データ不整合として扱う。

以下のようにデフォルト値へ補完しない。

```text
設定なし
    ↓
isAvailable = false
```

具体的なエラーコードは、「エラーレスポンス」で定義する。

---

### 12.12 利用可能資産設定が複数存在する場合

同一資産口座について、現在年月に有効な設定が複数存在する場合は、データ不整合として扱う。

例えば、

```text
設定A
start_year_month = 2026-01
end_year_month   = NULL

設定B
start_year_month = 2026-06
end_year_month   = NULL
```

の状態で現在年月が`2026-08`の場合、両方が有効となる。

この場合、以下のように任意の1件を選択してはならない。

```text
ORDER BY start_year_month DESC
LIMIT 1
```

設定期間の重複を隠蔽することになるためである。

データ不整合として扱う。

---

### 12.13 利用開始年月より前の扱い

資産口座には、

```text
start_year_month
```

としてLife Planner上での利用開始年月を保持する。

ACC-003は現在の資産口座詳細を確認するためのAPIであるため、登録済みで利用中の資産口座について基本情報自体は取得対象とする。

ただし、現在年月が`start_year_month`より前であり、現在有効な利用可能資産設定が存在しない場合に、`isAvailable`を任意の値へ補完しない。

利用開始前の資産口座をどの画面で表示するかについては、画面設計および各APIの対象年月ルールに従う。

---

### 12.14 isAvailable

レスポンスの

```text
isAvailable
```

には、現在有効な

```text
asset_account_available_settings.is_available
```

を使用する。

概念的には、

```text
asset_account_available_settings.is_available
    = true
        ↓
isAvailable = true
```

```text
asset_account_available_settings.is_available
    = false
        ↓
isAvailable = false
```

とする。

---

### 12.15 isEnabled

レスポンスの

```text
isEnabled
```

は、資産口座の利用状態を表す。

ACC-003では論理削除済み資産口座を取得対象としないため、正常レスポンスでは

```text
isEnabled = true
```

となる。

データベースの

```text
deleted_at
```

自体はレスポンスへ返却しない。

---

### 12.16 DB内部コードをそのまま返却しない

`asset_type`および`balance_recording_unit`をDB内部で数値コードとして保持している場合でも、その値をそのままレスポンスへ返却しない。

例えば、

```text
asset_type = 3
```

を、

```json
{
  "assetType": 3
}
```

として返却しない。

APIで定義した文字列表現へ変換する。

概念例：

```json
{
  "assetType": "SECURITIES"
}
```

同様に、`balance_recording_unit`も、

```text
ACCOUNT
HOLDING
```

のAPI表現へ変換する。

---

### 12.17 過去時点の利用可能資産区分を返却しない

ACC-003は、現在の資産口座詳細を取得するAPIとする。

そのため、過去年月を指定して

```text
2026-01時点では利用可能だったか
```

といった情報を取得する用途には使用しない。

過去時点の利用可能資産区分が必要な月末資産管理や目的達成判定では、各APIが持つ対象年月を基準として利用可能資産設定を判定する。

---

### 12.18 保有商品を取得しない

ACC-003では、`balanceRecordingUnit`が

```text
HOLDING
```

であっても、保有商品一覧を同時に返却しない。

保有商品は保有商品APIの責務とする。

概念的には、

```text
ACC-003
    → 資産口座詳細

HLD-001
    → 保有商品一覧
```

と責務を分離する。

---

### 12.19 月末資産データを取得しない

ACC-003では、以下の情報を取得しない。

* 月末資産スナップショット
* 月末資産残高
* 商品別月末評価額
* 最新残高
* 資産推移

これらは、月末資産APIおよび資産表示APIの責務とする。

---

### 12.20 業務データを更新しない

ACC-003は参照専用APIである。

以下のような更新処理を行わない。

```text
asset_accounts
    INSERT / UPDATE / DELETE

asset_account_available_settings
    INSERT / UPDATE / DELETE
```

詳細取得によって、`updated_at`などの業務データを変更しない。

---

## 13. 処理フロー

概念的な処理フローは、以下とする。

```text
リクエスト受付
    ↓
リクエストID生成
    ↓
X-User-Id検証
    ↓
操作対象利用者の特定
    ↓
assetAccountId形式検証
    ↓
操作対象利用者に帰属する
有効な資産口座を取得
    ↓
存在しない
    → ASSET_ACCOUNT_NOT_FOUND
    ↓
現在年月を取得
    ↓
現在有効な
利用可能資産設定を取得
    ↓
設定件数確認
    ↓
0件または複数件
    → データ不整合
    ↓
資産口座詳細DTO生成
    ↓
APIレスポンス用の形式へ変換
    ↓
正常レスポンス返却
```

---

### 13.1 利用者コンテキスト確認

共通Middlewareで`X-User-Id`を検証し、操作対象利用者を特定する。

利用者コンテキストが不正な場合は、資産口座検索へ進まない。

---

### 13.2 assetAccountId検証

パスパラメータの

```text
assetAccountId
```

が正の整数形式であることを確認する。

形式不正の場合は、データベース検索へ進まない。

---

### 13.3 資産口座取得

操作対象利用者IDと`assetAccountId`を使用して、資産口座を取得する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

該当する資産口座が存在しない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

とする。

---

### 13.4 現在年月取得

サーバー側の現在日時から、利用可能資産設定判定用の現在年月を取得する。

概念的には、

```text
now()
    ↓
YYYY-MM
```

とする。

---

### 13.5 利用可能資産設定取得

取得した資産口座IDと現在年月を使用して、現在有効な利用可能資産設定を取得する。

概念的には、

```text
asset_account_id
    = assetAccountId

AND

start_year_month
    <= 現在年月

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= 現在年月
)
```

とする。

---

### 13.6 利用可能資産設定件数確認

取得結果が1件であることを確認する。

```text
1件
    ↓
正常処理継続
```

```text
0件
    ↓
データ不整合
```

```text
2件以上
    ↓
データ不整合
```

0件の場合に`false`へ補完せず、複数件の場合に任意の1件を選択しない。

---

### 13.7 詳細情報生成

資産口座と現在有効な利用可能資産設定から、APIレスポンスに必要な詳細情報を生成する。

概念的には、

```text
asset_accounts.id
    → id

asset_accounts.name
    → name

asset_accounts.asset_type
    → assetType

asset_accounts.balance_recording_unit
    → balanceRecordingUnit

asset_accounts.start_year_month
    → startYearMonth

asset_account_available_settings.is_available
    → isAvailable

asset_accounts.deleted_at
    → isEnabled
```

とする。

---

### 13.8 レスポンス変換

データベースカラムをそのまま返却せず、DTOおよびAPI Resourceを使用してAPIレスポンス形式へ変換する。

特に、

```text
asset_type
balance_recording_unit
```

は、APIで定義した文字列表現へ変換する。

---

## 14. トランザクション境界

ACC-003は、参照専用APIであり、業務データを更新しない。

そのため、Phase1では明示的なデータベーストランザクションを使用しない。

---

### 14.1 更新処理なし

ACC-003では、以下の処理を行わない。

```text
INSERT
UPDATE
DELETE
```

主な処理は、

```text
asset_accounts
    SELECT

asset_account_available_settings
    SELECT
```

のみである。

---

### 14.2 明示的なトランザクションを使用しない理由

ACC-003では、資産口座と利用可能資産設定を参照するが、両方を更新するわけではない。

Phase1では、詳細表示のための通常参照処理として扱い、

```php
DB::transaction(function () {
    // SELECT
});
```

のような明示的なトランザクションは使用しない。

---

### 14.3 参照間の整合性

資産口座取得後に、利用可能資産設定が変更される可能性はある。

ただし、ACC-003は現在状態を表示するための参照APIであり、目的達成判定や月末確定のように、複数データを厳密に同一時点で固定して業務判断する処理ではない。

そのため、Phase1ではRepeatable Readなどによるスナップショット固定を要求しない。

---

### 14.4 厳密な一貫性が必要な処理との区別

ACC-003の詳細表示と、以下のような業務処理は区別する。

```text
月末資産確定
目的達成判定
利用可能資産設定変更
```

これらでトランザクションや排他制御が必要な場合は、各API側で個別に定義する。

ACC-003の参照要件を理由として、不要なトランザクションを導入しない。

---

### 14.5 データ不整合時

現在有効な利用可能資産設定が

```text
0件
または
複数件
```

であることを検出した場合も、ACC-003自身はデータ修復を行わない。

例えば、

```text
設定なし
    ↓
初期設定を自動INSERT
```

や、

```text
設定重複
    ↓
古い設定を自動UPDATE
```

といった処理は行わない。

参照処理の中で業務データを暗黙的に変更せず、データ不整合としてエラーを返却する。

---

## 15. 排他制御

ACC-003は、参照専用APIであり、業務データを更新しない。

そのため、Phase1では明示的な排他制御を行わない。

以下のような行ロックは使用しない。

```php
lockForUpdate()
```

また、楽観ロック用の以下も使用しない。

```text
version
updated_at比較
ETag
If-Match
```

---

### 15.1 資産口座へのロック

ACC-003では、`asset_accounts`を参照するだけであるため、対象資産口座行をロックしない。

概念的には、以下とする。

```text
asset_accounts
    SELECT
```

---

### 15.2 利用可能資産設定へのロック

`asset_account_available_settings`についても、現在有効な設定を参照するだけであるため、明示的な行ロックを行わない。

概念的には、以下とする。

```text
asset_account_available_settings
    SELECT
```

---

### 15.3 同時更新との関係

ACC-003実行中に、別リクエストによって以下の処理が同時実行される可能性がある。

* 資産口座更新
* 利用可能資産設定変更
* 資産口座無効化

Phase1では、ACC-003を現在状態の参照APIとして扱うため、参照中の状態を長時間固定しない。

必要に応じて、次回のAPI取得時に最新状態へ更新されるものとする。

---

### 15.4 厳密な同一時点保証を行わない

ACC-003では、

```text
asset_accounts取得時点
```

と

```text
asset_account_available_settings取得時点
```

を厳密に同一瞬間へ固定することを要件としない。

本APIは業務判断を確定する処理ではなく、詳細表示用の参照APIであるためである。

---

### 15.5 データ不整合はロックで補正しない

現在有効な利用可能資産設定が、

```text
0件
または
複数件
```

存在する場合に、ACC-003で排他制御を取得して状態を修復しない。

不整合状態は、エラーとして検出する。

---

## 16. 成功レスポンス

指定された資産口座の詳細取得に成功した場合は、

```http
200 OK
```

を返却する。

正常時は、API共通方針に従った成功レスポンス形式を使用する。

概念例：

```json
{
  "data": {
    "id": "10",
    "name": "証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

---

### 16.1 レスポンス項目

正常時の`data`配下には、以下を返却する。

| 項目                     | 型       | NULL | 説明                     |
| ---------------------- | ------- | :--: | ---------------------- |
| `id`                   | string  |   ×  | 資産口座ID                 |
| `name`                 | string  |   ×  | 資産口座名                  |
| `assetType`            | string  |   ×  | 資産種別                   |
| `balanceRecordingUnit` | string  |   ×  | 残高記録単位                 |
| `isAvailable`          | boolean |   ×  | 現在の利用可能資産区分            |
| `startYearMonth`       | string  |   ×  | 利用開始年月。`YYYY-MM`形式     |
| `isEnabled`            | boolean |   ×  | 利用状態。正常レスポンスでは常に`true` |

---

### 16.2 id

対象資産口座のIDを文字列として返却する。

例：

```json
{
  "id": "10"
}
```

データベースでは`bigint`で保持していても、API共通方針に従って文字列として返却する。

---

### 16.3 name

対象資産口座の資産口座名を返却する。

例：

```json
{
  "name": "証券口座"
}
```

---

### 16.4 assetType

資産種別は、API用の文字列表現で返却する。

例：

```json
{
  "assetType": "SECURITIES"
}
```

DB内部の数値コードをそのまま返却しない。

---

### 16.5 balanceRecordingUnit

残高記録単位は、API用の文字列表現で返却する。

想定する値は、以下とする。

```text
ACCOUNT
HOLDING
```

例：

```json
{
  "balanceRecordingUnit": "HOLDING"
}
```

---

### 16.6 isAvailable

現在年月において有効な、

```text
asset_account_available_settings.is_available
```

を返却する。

例：

```json
{
  "isAvailable": true
}
```

`asset_accounts`の固定カラムとして保持されている値ではない。

---

### 16.7 startYearMonth

資産口座の利用開始年月を返却する。

形式は、

```text
YYYY-MM
```

とする。

例：

```json
{
  "startYearMonth": "2026-08"
}
```

---

### 16.8 isEnabled

ACC-003では、論理削除済み資産口座を取得対象としない。

そのため、正常レスポンスでは、

```json
{
  "isEnabled": true
}
```

となる。

`deleted_at`自体は返却しない。

---

### 16.9 返却しない情報

ACC-003では、以下の内部情報をレスポンスへ返却しない。

* `user_id`
* `deleted_at`
* `created_at`
* `updated_at`
* `asset_account_available_settings.id`
* `asset_account_available_settings.asset_account_id`
* `asset_account_available_settings.start_year_month`
* `asset_account_available_settings.end_year_month`
* DB内部の数値コード

また、以下の関連データも返却しない。

* 保有商品一覧
* 月末資産残高
* 商品別月末評価額
* 月末資産状況
* 資産推移

---

## 17. エラーレスポンス

異常時は、API共通方針に従った共通エラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_NOT_FOUND",
    "message": "指定された資産口座が存在しません。"
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

---

### 17.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

この場合、資産口座検索へ進まない。

---

### 17.2 INVALID_USER_ID

`X-User-Id`の形式が不正な場合は、

```text
INVALID_USER_ID
```

を返却する。

例えば、以下を不正とする。

```text
0
-1
abc
1.5
```

---

### 17.3 USER_NOT_FOUND

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

---

### 17.4 INVALID_ASSET_ACCOUNT_ID

`assetAccountId`の形式が不正な場合は、

```text
INVALID_ASSET_ACCOUNT_ID
```

を返却する。

例えば、

```text
0
-1
abc
1.5
1e3
```

などを不正とする。

形式不正の場合は、資産口座検索へ進まない。

---

### 17.5 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

* 指定された資産口座が存在しない
* 指定された資産口座が他利用者に属する
* 指定された資産口座が論理削除済み

他利用者の資産口座であることを示す専用エラーコードは返却しない。

---

### 17.6 他利用者の資産口座

例えば、

```text
User A
    assetAccountId = 10

User B
    X-User-Id = 2
```

の状態で、User Bが

```http
GET /api/v1/asset-accounts/10
```

を実行した場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

これにより、他利用者の資産口座存在有無を外部へ公開しない。

---

### 17.7 論理削除済み資産口座

対象資産口座が、

```text
deleted_at IS NOT NULL
```

の場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

論理削除済みであることを専用エラーとして公開しない。

---

### 17.8 利用可能資産設定不存在

対象資産口座について、現在有効な

```text
asset_account_available_settings
```

が存在しない場合は、正常レスポンスを返却しない。

ACC-002で初期設定が必ず作成される前提に対するデータ不整合として扱う。

概念的な独自エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

とする。

ただし、データ不整合系エラーコードを共通化する場合は、API共通方針の定義を優先する。

---

### 17.9 利用可能資産設定重複

現在年月に有効な利用可能資産設定が複数件存在する場合も、正常レスポンスを返却しない。

概念的な独自エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

とする。

任意の1件を選択してレスポンスを返してはならない。

---

### 17.10 データ不整合時に補完しない

利用可能資産設定の不整合時に、以下のような自動補完を行わない。

```text
設定なし
    ↓
isAvailable = false
```

または、

```text
複数設定
    ↓
最新1件を採用
```

データ不整合を隠蔽せず、明示的なエラーとして扱う。

---

### 17.11 INTERNAL_SERVER_ERROR

資産口座詳細取得処理で想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスへ、以下の内部情報を含めない。

* SQL
* PostgreSQL内部エラー
* テーブル名
* カラム名
* 制約名
* Laravel内部例外メッセージ
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 17.12 主なエラー一覧

ACC-003で想定する主なエラーは、以下とする。

| HTTPステータス                   | 独自エラーコード                                    | 発生条件                     |
| --------------------------- | ------------------------------------------- | ------------------------ |
| `400 Bad Request`           | `USER_CONTEXT_REQUIRED`                     | `X-User-Id`未指定           |
| `400 Bad Request`           | `INVALID_USER_ID`                           | `X-User-Id`形式不正          |
| `400 Bad Request`           | `INVALID_ASSET_ACCOUNT_ID`                  | `assetAccountId`形式不正     |
| `404 Not Found`             | `USER_NOT_FOUND`                            | 利用者不存在、または論理削除済み         |
| `404 Not Found`             | `ASSET_ACCOUNT_NOT_FOUND`                   | 資産口座不存在、他利用者所属、または論理削除済み |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND` | 現在有効な利用可能資産設定が存在しない      |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT`  | 現在有効な利用可能資産設定が複数存在する     |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR`                     | 想定外のサーバー内部エラー            |

利用可能資産設定の不存在・重複は、クライアント入力によるものではなく、サーバー側の業務データ不整合として扱う。

そのため、Phase1では、

```http
500 Internal Server Error
```

相当とする。

具体的なデータ不整合エラーの扱いをAPI共通方針で統一する場合は、その定義を優先する。

---

### 17.13 エラー時の副作用

ACC-003は参照専用APIであるため、いずれのエラーが発生した場合も業務データを更新しない。

特に、利用可能資産設定の不整合を検出しても、

```text
asset_account_available_settings
```

を自動修復しない。


---

## 18. HTTPステータス

ACC-003では、処理結果に応じて以下のHTTPステータスを返却する。

| HTTPステータス                   | 用途                                |
| --------------------------- | --------------------------------- |
| `200 OK`                    | 資産口座詳細取得成功                        |
| `400 Bad Request`           | 利用者コンテキストまたは`assetAccountId`の形式不正 |
| `404 Not Found`             | 利用者または資産口座が存在しない                  |
| `500 Internal Server Error` | 利用可能資産設定の不整合、または想定外のサーバー内部エラー     |

具体的な独自エラーコードは、「エラーレスポンス」およびAPI共通方針に従う。

---

### 18.1 200 OK

指定された資産口座が操作対象利用者に帰属し、現在有効な利用可能資産設定も正常に取得できた場合は、

```http
200 OK
```

を返却する。

概念的には、

```text
資産口座取得成功
+
現在有効な利用可能資産設定 1件
    ↓
200 OK
```

とする。

---

### 18.2 400 Bad Request

以下の場合は、

```http
400 Bad Request
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
INVALID_ASSET_ACCOUNT_ID
```

例えば、

* `X-User-Id`未指定
* `X-User-Id`形式不正
* `assetAccountId`が正の整数形式ではない

などを対象とする。

---

### 18.3 404 Not Found

以下の場合は、

```http
404 Not Found
```

を返却する。

主なエラーコードは、以下とする。

```text
USER_NOT_FOUND
ASSET_ACCOUNT_NOT_FOUND
```

`ASSET_ACCOUNT_NOT_FOUND`には、以下を含む。

* 資産口座不存在
* 他利用者に属する資産口座
* 論理削除済み資産口座

他利用者に属することや論理削除済みであることを、個別のHTTPステータスで公開しない。

---

### 18.4 500 Internal Server Error

以下の場合は、

```http
500 Internal Server Error
```

として扱う。

主なエラーコードは、以下とする。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
INTERNAL_SERVER_ERROR
```

利用可能資産設定の不存在や重複は、クライアント入力によるものではなく、本来成立しているべき業務データの整合性が崩れている状態とする。

---

### 18.5 ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND

対象資産口座について、現在年月に有効な

```text
asset_account_available_settings
```

が存在しない場合は、

```http
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

とする。

ACC-002で初期利用可能資産設定を必ず作成する設計に対するデータ不整合として扱う。

---

### 18.6 ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT

現在年月に有効な利用可能資産設定が複数件存在する場合は、

```http
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

とする。

任意の設定を選択して`200 OK`を返却しない。

---

### 18.7 INTERNAL_SERVER_ERROR

想定外のサーバー内部エラーが発生した場合は、

```http
500 Internal Server Error
```

を返却する。

エラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

レスポンスへSQLやスタックトレースなどの内部情報を含めない。

---

## 19. 冪等性

ACC-003は、参照専用のGET APIであるため、冪等である。

同一の

```text
X-User-Id
+
assetAccountId
```

に対して、同じサーバー状態で複数回リクエストした場合、同じ結果を返却する。

---

### 19.1 業務データを変更しない

ACC-003を何度実行しても、以下の業務データを変更しない。

* `users`
* `asset_accounts`
* `asset_account_available_settings`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

詳細取得によって`updated_at`なども更新しない。

---

### 19.2 同一リクエストの再実行

例えば、

```http
GET /api/v1/asset-accounts/10
Accept: application/json
X-User-Id: 1
```

を、サーバー側の状態が変化していない間に複数回実行した場合は、同じ詳細情報を返却する。

---

### 19.3 サーバー状態が変更された場合

ACC-003自体は冪等であるが、別APIによって資産口座や利用可能資産設定が変更された場合は、後続のACC-003で返却内容が変化する。

例えば、

```text
ACC-003
isAvailable = true
    ↓
ACC-006
利用可能資産区分変更
    ↓
ACC-003
isAvailable = false
```

となり得る。

これは、ACC-003の冪等性を損なうものではない。

---

### 19.4 資産口座無効化後

ACC-003取得後に別APIによって資産口座が無効化された場合、同じ`assetAccountId`で再度ACC-003を実行すると、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となる。

これも、サーバー状態が変化した結果である。

---

### 19.5 Idempotency-Key

ACC-003はGETによる参照APIであるため、

```text
Idempotency-Key
```

を使用しない。

冪等性確保のための追加ヘッダーやサーバー側キー管理は不要とする。

---

## 20. キャッシュ

Phase1では、ACC-003専用のサーバー側アプリケーションキャッシュを使用しない。

資産口座詳細および現在の利用可能資産区分は、PostgreSQLの最新状態から取得する。

---

### 20.1 サーバー側キャッシュ

Phase1では、以下のようなACC-003専用キャッシュを導入しない。

```text
Redis
Laravel Cache
Application Memory Cache
```

資産口座詳細は単一資産口座と現在有効な利用可能資産設定を取得するだけであり、Phase1ではキャッシュ導入による複雑性を追加しない。

---

### 20.2 利用可能資産設定とキャッシュ

ACC-003では、現在年月における

```text
asset_account_available_settings
```

を取得する。

利用可能資産区分は、ACC-006などによって変更される可能性がある。

そのため、古いキャッシュによって

```text
isAvailable
```

を返却し続けないようにする必要がある。

Phase1ではサーバー側キャッシュを使用しないことで、常に最新DB状態を参照する。

---

### 20.3 React・TypeScript側のQuery Cache

フロントエンドでは、TanStack QueryなどのQuery Cacheを使用してよい。

概念的には、以下のようなQuery Keyを使用する。

```text
assetAccounts
+
assetAccountId
```

例えば、

```ts
[
  'assetAccounts',
  assetAccountId,
]
```

のようなキーで詳細取得結果を保持してよい。

具体的な実装は、「React・TypeScriptでの利用」で定義する。

---

### 20.4 ACC-004成功後

ACC-004 資産口座更新によって、対象資産口座の

* `name`
* `assetType`

などが変更された場合は、ACC-003のキャッシュが古くなる。

そのため、フロントエンドではACC-004成功後に対象資産口座の詳細Query Cacheを無効化する。

---

### 20.5 ACC-005成功後

ACC-005 資産口座無効化によって、対象資産口座が通常参照対象から除外された場合は、ACC-003の詳細キャッシュも無効化する。

無効化済み資産口座の古い詳細情報を画面上へ残し続けない。

---

### 20.6 ACC-006成功後

ACC-006 利用可能資産設定登録によって、現在の

```text
isAvailable
```

が変化する可能性がある。

そのため、ACC-006成功後は対象資産口座のACC-003詳細Query Cacheを無効化する。

概念的には、

```text
ACC-006成功
    ↓
ACC-003 Query invalidate
    ↓
ACC-003再取得
    ↓
最新isAvailableを表示
```

とする。

---

### 20.7 ACC-002成功後

ACC-002で新しい資産口座を登録した場合、その資産口座IDについてACC-003を初めて取得できるようになる。

ACC-002成功レスポンスをACC-003の正式なサーバーキャッシュとして扱わない。

必要な場合は、返却された

```text
id
```

を使用してACC-003を取得する。

---

### 20.8 HTTPキャッシュ

ACC-003はGET APIであるため、技術的にはHTTPキャッシュの対象にできる。

ただし、Phase1ではACC-003独自の

```text
ETag
Last-Modified
If-None-Match
If-Modified-Since
```

などは導入しない。

HTTPキャッシュヘッダーに関する共通方針がある場合は、API共通方針を優先する。

---

### 20.9 利用者境界とキャッシュキー

フロントエンドまたは将来のサーバーキャッシュでACC-003をキャッシュする場合は、`assetAccountId`だけでキャッシュを共有しない。

本APIは利用者依存APIであるため、概念的には

```text
userId
+
assetAccountId
```

によって利用者境界を維持する必要がある。

例えば、異なる利用者間で同じキャッシュエントリを誤って共有してはならない。

---

### 20.10 現在年月の変化

ACC-003の

```text
isAvailable
```

は、現在年月を基準として利用可能資産設定履歴から算出する。

そのため、月が変わった場合は、DB更新がなくても返却される`isAvailable`が変化する可能性がある。

例えば、

```text
設定A
2026-01 ～ 2026-08
is_available = true

設定B
2026-09 ～ NULL
is_available = false
```

の場合、

```text
2026-08
    → isAvailable = true

2026-09
    → isAvailable = false
```

となる。

将来的に長時間キャッシュを導入する場合は、この年月境界も考慮する。

Phase1ではサーバー側キャッシュを使用しないため、特別な失効処理は実装しない。

---

## 21. 関連テーブル

ACC-003では、指定された資産口座の詳細情報と、現在有効な利用可能資産設定を取得するため、以下のテーブルを使用する。

| テーブル                               | 用途                |  更新 |
| ---------------------------------- | ----------------- | :-: |
| `users`                            | 操作対象利用者の確認        |  ×  |
| `asset_accounts`                   | 対象資産口座の取得、利用者境界確認 |  ×  |
| `asset_account_available_settings` | 現在の利用可能資産区分の取得    |  ×  |

ACC-003は参照専用APIであるため、いずれのテーブルも更新しない。

---

### 21.1 users

`X-User-Id`で指定された操作対象利用者の存在確認に使用する。

概念的な条件は、以下とする。

```text
users.id
    = X-User-Id

AND

users.deleted_at
    IS NULL
```

利用者コンテキストの確認は、API共通Middlewareで行う。

ACC-003では、`users`を更新しない。

---

### 21.2 asset_accounts

指定された`assetAccountId`に対応する資産口座を取得するために使用する。

主に以下のカラムを使用する。

| カラム                      | 用途                   |
| ------------------------ | -------------------- |
| `id`                     | `assetAccountId`との照合 |
| `user_id`                | 操作対象利用者との利用者境界確認     |
| `name`                   | 資産口座名                |
| `asset_type`             | 資産種別                 |
| `balance_recording_unit` | 残高記録単位               |
| `start_year_month`       | 利用開始年月               |
| `deleted_at`             | 利用状態の判定              |

概念的な取得条件は、以下とする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = 操作対象利用者ID

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 21.3 他利用者の資産口座

`asset_accounts.id`が一致していても、

```text
asset_accounts.user_id
    != 操作対象利用者ID
```

の場合は、取得対象に含めない。

他利用者の資産口座を一度取得してから、アプリケーション側で利用者境界を判定する方式を基本としない。

---

### 21.4 論理削除済み資産口座

ACC-003では、通常利用中の資産口座だけを取得対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は取得しない。

LaravelのSoftDeletesを使用している場合は、通常スコープを利用し、

```php
withTrashed()
```

を使用しない。

---

### 21.5 asset_type

`asset_accounts.asset_type`は、DB内部の保存形式からAPI用の資産種別へ変換する。

概念的には、

```text
DB
asset_type
    ↓
API
assetType
```

とする。

APIでは、例えば以下の値として返却する。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

DB内部の数値コードを直接返却しない。

---

### 21.6 balance_recording_unit

`asset_accounts.balance_recording_unit`も、API用の文字列表現へ変換する。

概念的には、

```text
DB
balance_recording_unit
    ↓
API
balanceRecordingUnit
```

とする。

APIでは、

```text
ACCOUNT
HOLDING
```

として返却する。

---

### 21.7 start_year_month

`asset_accounts.start_year_month`は、そのまま

```text
YYYY-MM
```

形式の

```text
startYearMonth
```

として返却する。

日付形式への不要な変換は行わない。

---

### 21.8 deleted_at

`asset_accounts.deleted_at`は、APIへ直接返却しない。

ACC-003では、`deleted_at IS NULL`の資産口座だけを取得するため、正常レスポンスでは

```text
isEnabled = true
```

として扱う。

---

### 21.9 asset_account_available_settings

現在の利用可能資産区分を取得するために使用する。

主に以下のカラムを使用する。

| カラム                | 用途         |
| ------------------ | ---------- |
| `asset_account_id` | 対象資産口座との関連 |
| `start_year_month` | 設定の適用開始年月  |
| `end_year_month`   | 設定の適用終了年月  |
| `is_available`     | 利用可能資産区分   |

---

### 21.10 現在有効な利用可能資産設定

現在年月を基準として、概念的に以下の条件を満たす利用可能資産設定を取得する。

```text
asset_account_available_settings.asset_account_id
    = assetAccountId

AND

asset_account_available_settings.start_year_month
    <= 現在年月

AND

(
    asset_account_available_settings.end_year_month
        IS NULL

    OR

    asset_account_available_settings.end_year_month
        >= 現在年月
)
```

正常状態では、該当する設定が1件だけ存在することを前提とする。

---

### 21.11 is_available

現在有効な設定の

```text
asset_account_available_settings.is_available
```

を、

```text
isAvailable
```

として返却する。

概念的には、

```text
is_available = true
    ↓
isAvailable = true
```

```text
is_available = false
    ↓
isAvailable = false
```

とする。

---

### 21.12 利用可能資産設定が存在しない場合

現在年月に有効な利用可能資産設定が存在しない場合は、データ不整合として扱う。

以下のような補完は行わない。

```text
設定なし
    ↓
isAvailable = false
```

ACC-002では初期利用可能資産設定を必ず登録する設計のため、正常状態として扱わない。

---

### 21.13 利用可能資産設定が複数存在する場合

現在年月に有効な利用可能資産設定が複数存在する場合も、データ不整合として扱う。

以下のように、

```text
start_year_monthが
最も新しい設定
```

を任意に採用しない。

期間重複を参照API側で隠蔽しない。

---

### 21.14 更新しないテーブル

ACC-003では、以下のテーブルも更新しない。

* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

また、これらのデータを資産口座詳細レスポンスへ含めない。

---

## 22. 関連する機能要件

ACC-003は、資産口座詳細取得および現在の利用可能資産区分表示に関する機能要件と対応する。

主な関連要件は、以下とする。

* 資産口座管理

  * 指定した資産口座の詳細情報を取得できる
  * 資産口座名を確認できる
  * 資産種別を確認できる
  * 残高記録単位を確認できる
  * 利用開始年月を確認できる
  * 利用状態を確認できる
* 利用可能資産区分

  * 利用可能資産区分を履歴として管理する
  * 現在年月に有効な設定から現在の利用可能資産区分を判定する
  * 利用可能資産設定の履歴そのものはACC-003で一覧返却しない
* 利用者境界

  * 操作対象利用者に帰属する資産口座だけを取得できる
  * 他利用者に属する資産口座は取得できない
  * 他利用者の資産口座は対象不存在として扱う
* 論理削除

  * 論理削除済み資産口座は通常の詳細取得対象としない

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

## 23. インデックス

ACC-003では、主に以下の検索条件を使用する。

```text
asset_accounts.id
```

```text
asset_accounts.user_id
```

```text
asset_account_available_settings.asset_account_id
+
asset_account_available_settings.start_year_month
+
asset_account_available_settings.end_year_month
```

必要なインデックスは、テーブル全体の利用状況を踏まえて設定する。

---

### 23.1 asset_accounts.id

`asset_accounts.id`は主キーであるため、主キーインデックスを使用する。

ACC-003専用として追加インデックスを作成しない。

---

### 23.2 asset_accounts.user_id

ACC-003では、主キーと利用者境界を組み合わせて取得する。

概念的には、

```text
id
+
user_id
```

を条件とする。

`id`は主キーであるため、Phase1ではACC-003のためだけに

```text
(id, user_id)
```

の複合インデックスを追加する必要性は低い。

実際のクエリ実行計画と他APIの検索条件を踏まえて判断する。

---

### 23.3 asset_account_available_settings

現在有効な設定取得では、

```text
asset_account_id
```

を必須条件として使用する。

利用可能資産設定は、同一資産口座内の期間履歴から検索するため、必要に応じて

```text
asset_account_id
+
start_year_month
```

などの複合インデックスを検討する。

HLDや目的達成判定など、対象年月判定を行う他APIでも利用される検索条件を踏まえ、共通して有効なインデックスを設計する。

---

### 23.4 不要な重複インデックスを作成しない

ACC-003のためだけに、既存の主キー・FK・UNIQUE制約と役割が重複するインデックスを追加しない。

実際のSQL、データ件数、実行計画を確認したうえで追加を判断する。

---

## 24. 性能

ACC-003は、単一の資産口座詳細を取得するAPIであるため、処理負荷は小さい。

概念的には、

```text
asset_accounts
1件取得
    ↓
asset_account_available_settings
現在有効な設定取得
    ↓
DTO生成
```

で完結する。

---

### 24.1 資産口座一覧を取得しない

以下のように、利用者の全資産口座を取得してからアプリケーション側で対象IDを探す実装は行わない。

```text
資産口座一覧取得
    ↓
assetAccountIdでfilter
```

対象IDと利用者IDを条件として直接1件取得する。

---

### 24.2 利用可能資産設定を全履歴取得しない

ACC-003で必要なのは、現在有効な利用可能資産設定だけである。

そのため、

```text
対象資産口座の
利用可能資産設定を全件取得
    ↓
PHP側で現在設定を検索
```

を基本としない。

データベース側で現在年月条件を指定して取得する。

---

### 24.3 取得件数

正常状態では、概念的に以下となる。

```text
asset_accounts
    1件

asset_account_available_settings
    1件
```

大量データをレスポンスへ返却しない。

---

### 24.4 保有商品をEager Loadしない

`balanceRecordingUnit = HOLDING`の場合でも、

```text
holding_assets
```

をEager Loadしない。

ACC-003では保有商品を返却しないためである。

例えば、

```php
with('holdingAssets')
```

をACC-003のためだけに使用しない。

---

### 24.5 月末資産データを取得しない

以下の関連データをACC-003で取得しない。

```text
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
```

不要なJOINやEager Loadを避ける。

---

### 24.6 N+1問題

ACC-003は単一資産口座を対象とするため、通常の意味でのN+1問題は発生しない。

ただし、将来的にACC-001と共通Queryを使用する場合でも、詳細取得のためだけに一覧向けの不要な関連取得処理を行わない。

---

## 25. セキュリティ

ACC-003では、利用者境界を最重要のセキュリティ要件とする。

---

### 25.1 assetAccountIdだけで取得しない

以下のような取得は避ける。

```php
AssetAccount::find(
    $assetAccountId,
);
```

これだけでは、他利用者の資産口座も取得できる可能性がある。

必ず、

```text
assetAccountId
+
userId
+
deleted_at IS NULL
```

を条件に含める。

---

### 25.2 他利用者の存在を公開しない

他利用者に属する`assetAccountId`を指定された場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下のような情報を返さない。

```text
この資産口座は
別の利用者に属しています
```

---

### 25.3 論理削除状態を公開しない

論理削除済み資産口座についても、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下のような状態判別可能なエラーは返却しない。

```text
ASSET_ACCOUNT_DISABLED
ASSET_ACCOUNT_DELETED
```

---

### 25.4 内部IDを不要に返さない

ACC-003では、対象資産口座自身の

```text
id
```

は返却するが、以下の内部IDは返却しない。

* `user_id`
* `asset_account_available_settings.id`
* `asset_account_available_settings.asset_account_id`

クライアントに不要な内部関連情報を公開しない。

---

### 25.5 DB内部コードを公開しない

`asset_type`や`balance_recording_unit`がDB内部で数値コードの場合でも、その値を公開しない。

API用の意味のある文字列へ変換する。

---

### 25.6 SQLインジェクション対策

`assetAccountId`や利用者IDを検索条件に使用する場合は、EloquentまたはQuery Builderのバインド機構を使用する。

SQL文字列へパラメータを直接連結しない。

---

### 25.7 エラー情報

異常時に、以下をレスポンスへ含めない。

* SQL
* PostgreSQL内部エラー
* テーブル名
* カラム名
* 制約名
* Laravel内部例外
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

## 26. ログ・監視

ACC-003では、API共通ログ方針に従って処理結果を記録する。

参照APIであるため、レスポンス内容そのものを不要にログへ出力しない。

---

### 26.1 ログコンテキスト

必要に応じて、以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assetAccountId
httpStatus
errorCode
```

`apiId`は、

```text
ACC-003
```

とする。

---

### 26.2 正常時

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = ACC-003
assetAccountId
httpStatus = 200
```

以下は、通常ログへ不要に出力しない。

* 資産口座名
* `isAvailable`
* 資産種別
* 残高記録単位

---

### 26.3 異常時

異常時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId
assetAccountId
errorCode
httpStatus
```

---

### 26.4 データ不整合時

以下のエラーは、業務データ不整合を示すため、調査可能なログを記録する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text
assetAccountId
currentYearMonth
matchedSettingCount
```

などを内部ログへ記録してよい。

ただし、レスポンスへは公開しない。

---

### 26.5 requestId

クライアントへ返却する`requestId`とサーバーログを関連付けられるようにする。

障害調査時に、

```text
requestId
    ↓
対象ログ特定
```

が可能な状態とする。

---

## 27. テスト観点

ACC-003では、資産口座詳細の正常取得だけでなく、以下を重点的に確認する。

* 利用者境界
* 論理削除
* 現在年月判定
* 利用可能資産設定の期間境界
* 設定不存在・重複
* レスポンス契約
* 副作用なし

---

### 27.1 正常系

操作対象利用者に属する有効な資産口座と、現在有効な利用可能資産設定を1件用意する。

ACC-003を実行し、以下を確認する。

* `200 OK`となること
* `id`が対象資産口座IDとなること
* `name`が正しいこと
* `assetType`が正しいこと
* `balanceRecordingUnit`が正しいこと
* `startYearMonth`が正しいこと
* `isAvailable`が現在設定と一致すること
* `isEnabled = true`となること

---

### 27.2 X-User-Id未指定

`X-User-Id`を指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

以下を確認する。

* 資産口座検索へ進まないこと
* 業務データが更新されないこと

---

### 27.3 X-User-Id形式不正

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

となること。

---

### 27.4 利用者不存在

存在しない`X-User-Id`を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 27.5 論理削除済み利用者

論理削除済み利用者を指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

となること。

---

### 27.6 assetAccountId形式不正

以下の値をそれぞれ確認する。

```text
0
-1
abc
1.5
1e3
10abc
```

期待結果：

```text
400 Bad Request
INVALID_ASSET_ACCOUNT_ID
```

となること。

資産口座検索へ進まないことも確認する。

---

### 27.7 資産口座不存在

存在しない`assetAccountId`を指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 27.8 他利用者の資産口座

以下の状態を用意する。

```text
User A
    Asset Account A

User B
    Asset Account B
```

User Aとして、Asset Account BのIDを指定する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

User Bの資産口座情報がレスポンスへ含まれないこと。

---

### 27.9 論理削除済み資産口座

以下の資産口座を用意する。

```text
deleted_at IS NOT NULL
```

そのIDでACC-003を実行する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 27.10 assetType = CASH

DB上の資産種別が現金に対応する値の場合、レスポンスが

```text
CASH
```

となることを確認する。

---

### 27.11 assetType = BANK

銀行に対応するDB値から、

```text
BANK
```

へ正しく変換されること。

---

### 27.12 assetType = SECURITIES

証券に対応するDB値から、

```text
SECURITIES
```

へ正しく変換されること。

---

### 27.13 assetType = IDECO

iDeCoに対応するDB値から、

```text
IDECO
```

へ正しく変換されること。

---

### 27.14 assetType = CORPORATE_DC

企業型DCに対応するDB値から、

```text
CORPORATE_DC
```

へ正しく変換されること。

---

### 27.15 assetType = OTHER

その他に対応するDB値から、

```text
OTHER
```

へ正しく変換されること。

---

### 27.16 balanceRecordingUnit = ACCOUNT

DB値が口座単位に対応する場合、

```text
ACCOUNT
```

として返却されることを確認する。

---

### 27.17 balanceRecordingUnit = HOLDING

DB値が商品単位に対応する場合、

```text
HOLDING
```

として返却されることを確認する。

---

### 27.18 isAvailable = true

現在有効な利用可能資産設定が

```text
is_available = true
```

の場合、

```json
{
  "isAvailable": true
}
```

となること。

---

### 27.19 isAvailable = false

現在有効な利用可能資産設定が

```text
is_available = false
```

の場合、

```json
{
  "isAvailable": false
}
```

となること。

`false`を未設定扱いしないこと。

---

### 27.20 start_year_monthが現在年月と一致

例えば、現在年月が

```text
2026-08
```

で、

```text
start_year_month = 2026-08
end_year_month   = NULL
```

の場合、現在有効な設定として取得されること。

---

### 27.21 end_year_monthが現在年月と一致

現在年月が

```text
2026-08
```

で、

```text
start_year_month = 2026-01
end_year_month   = 2026-08
```

の場合、現在有効な設定として取得されること。

---

### 27.22 end_year_monthが前月

現在年月が

```text
2026-08
```

で、

```text
end_year_month = 2026-07
```

の場合、当該設定が現在有効として取得されないこと。

---

### 27.23 start_year_monthが翌月

現在年月が

```text
2026-08
```

で、

```text
start_year_month = 2026-09
```

の場合、当該設定が現在有効として取得されないこと。

---

### 27.24 end_year_monthがNULL

現在年月が`start_year_month`以降で、

```text
end_year_month = NULL
```

の場合、現在有効な設定として取得されること。

---

### 27.25 現在有効な設定が0件

対象資産口座について、現在年月に有効な利用可能資産設定を0件とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

以下を確認する。

* `isAvailable = false`へ補完されないこと
* 正常レスポンスを返さないこと
* 業務データを自動修復しないこと

---

### 27.26 現在有効な設定が複数件

例えば、以下の2件を用意する。

```text
設定A
start_year_month = 2026-01
end_year_month   = NULL

設定B
start_year_month = 2026-06
end_year_month   = NULL
```

現在年月を`2026-08`とする。

期待結果：

```text
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

となること。

最新1件だけを任意に返却しないこと。

---

### 27.27 過去設定と現在設定

以下の履歴を用意する。

```text
設定A
2026-01 ～ 2026-06
is_available = true

設定B
2026-07 ～ NULL
is_available = false
```

現在年月が`2026-08`の場合、

```text
設定B
```

だけが選択され、

```text
isAvailable = false
```

となること。

---

### 27.28 境界が重複しない正常履歴

例えば、

```text
設定A
2026-01 ～ 2026-06

設定B
2026-07 ～ NULL
```

の場合、現在年月に応じて必ず1件だけが選択されることを確認する。

---

### 27.29 保有商品を返却しない

対象資産口座に複数の`holding_assets`が存在する状態でACC-003を実行する。

レスポンスへ、以下が含まれないことを確認する。

* `holdingAssets`
* 保有商品名
* 保有商品ID

---

### 27.30 月末資産データを返却しない

対象資産口座に月末資産データが存在する状態でも、レスポンスへ以下が含まれないことを確認する。

* `monthEndAssetSnapshots`
* `monthEndAssetBalances`
* `monthEndHoldingValues`
* 最新残高
* 資産推移

---

### 27.31 正常レスポンス契約

正常時に、概念的に以下の形式となることを確認する。

```json
{
  "data": {
    "id": "10",
    "name": "証券口座",
    "assetType": "SECURITIES",
    "balanceRecordingUnit": "HOLDING",
    "isAvailable": true,
    "startYearMonth": "2026-08",
    "isEnabled": true
  }
}
```

以下を確認する。

* HTTPステータスが`200 OK`
* `id`がstring
* `name`がstring
* `assetType`がstring
* `balanceRecordingUnit`がstring
* `isAvailable`がboolean
* `startYearMonth`が`YYYY-MM`
* `isEnabled`がboolean
* `isEnabled = true`

---

### 27.32 返却しない情報

正常レスポンスに、以下が含まれないことを確認する。

* `user_id`
* `userId`
* `deleted_at`
* `deletedAt`
* `created_at`
* `createdAt`
* `updated_at`
* `updatedAt`
* `asset_account_available_settings.id`
* `assetAccountAvailableSettingId`
* `asset_account_id`
* `endYearMonth`
* DB内部の数値コード

---

### 27.33 副作用なし

ACC-003実行前後で、以下のテーブルに変更がないことを確認する。

* `users`
* `asset_accounts`
* `asset_account_available_settings`
* `holding_assets`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `net_incomes`
* `objectives`
* `assessment_histories`

特に、詳細取得によって

```text
updated_at
```

が更新されないことを確認する。

---

### 27.34 同一リクエストの再実行

サーバー状態を変更せずに、同じ

```text
X-User-Id
+
assetAccountId
```

でACC-003を複数回実行する。

同じレスポンス内容となることを確認する。

---

### 27.35 ACC-006後の再取得

最初に、

```text
isAvailable = true
```

となるACC-003を実行する。

その後、ACC-006によって現在設定を変更し、再度ACC-003を実行する。

最新状態の

```text
isAvailable
```

が返却されることを確認する。

---

### 27.36 資産口座無効化後の再取得

ACC-003で正常取得できる資産口座を、ACC-005で無効化する。

その後、同じIDでACC-003を実行する。

期待結果：

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となること。

---

### 27.37 currentYearMonthのテスト固定

利用可能資産設定の対象年月判定をテストする際は、Laravelの時刻固定機能などを使用して現在日時を固定する。

例えば、

```text
現在日時
2026-08-15
```

に固定し、

```text
現在年月
2026-08
```

として境界条件を確認する。

テスト実行日によって結果が変わらないようにする。

---

### 27.38 エラーレスポンス契約

代表的な異常系について、API共通のエラーレスポンス形式となることを確認する。

主に以下を対象とする。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
INVALID_ASSET_ACCOUNT_ID
ASSET_ACCOUNT_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
INTERNAL_SERVER_ERROR
```

以下も確認する。

* `error.code`が設定されること
* `error.message`が設定されること
* `requestId`が設定されること
* SQLが含まれないこと
* PostgreSQL内部エラーが含まれないこと
* スタックトレースが含まれないこと
* サーバーファイルパスが含まれないこと

---

### 27.39 データ不整合ログ

以下を発生させた場合に、調査可能なログが記録されることを確認する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text
requestId
userId
assetAccountId
currentYearMonth
```

などから対象ログを追跡できることを確認する。

---

### 27.40 INTERNAL_SERVER_ERROR

資産口座詳細取得処理で想定外の例外を発生させる。

期待結果：

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

以下を確認する。

* 業務データが更新されないこと
* 内部情報がレスポンスへ公開されないこと
* サーバーログに調査情報が記録されること
* レスポンスの`requestId`からログを追跡できること

---

## 28. Laravel実装方針

ACC-003では、Action、Query、DTO、API Resource、Responderを分離して実装する。

概念的な構成は、以下とする。

```text id="f3z6rq"
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ AssetAccountQuery
    └─ AssetAccountAvailableSettingQuery
    ↓
Detail Result DTO
    ↓
API Resource
    ↓
Responder
```

本APIはリクエストボディを使用しないため、ACC-003専用のFormRequestは作成しない。

また、参照専用APIであるため、Repositoryは使用せず、取得処理はQueryへ集約する。

---

### 28.1 Route

ACC-003は、以下のルートとして定義する。

概念例：

```php id="6jn9va"
Route::get(
    '/api/v1/asset-accounts/{assetAccountId}',
    ShowAssetAccountAction::class,
);
```

ACC-001とは、HTTPメソッドおよびパスによって区別する。

```text id="xquw2f"
GET
/api/v1/asset-accounts
    → ACC-001 資産口座一覧取得

GET
/api/v1/asset-accounts/{assetAccountId}
    → ACC-003 資産口座詳細取得
```

---

### 28.2 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* 共通例外処理
* ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、以下とする。

```text id="ah9d3g"
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

### 28.3 FormRequest

ACC-003専用のFormRequestは作成しない。

本APIでは、

```text id="2c8xiy"
リクエストボディ
    なし

クエリパラメータ
    なし
```

であり、ACC-003固有のRequest Bodyバリデーションが存在しないためである。

以下のような空のFormRequestは作成しない。

```php id="yx5blc"
final class ShowAssetAccountRequest
    extends FormRequest
{
}
```

不要なクラスを追加しない。

---

### 28.4 assetAccountIdの形式検証

`assetAccountId`は、ルート制約またはAPI共通のパスパラメータ検証方式で検証する。

概念例：

```php id="3j6h5m"
Route::get(
    '/api/v1/asset-accounts/{assetAccountId}',
    ShowAssetAccountAction::class,
)
    ->where(
        'assetAccountId',
        '[1-9][0-9]*',
    );
```

正の整数形式のみを許可する。

以下を有効なIDとして扱わない。

```text id="26c1kl"
0
-1
abc
1.5
1e3
10abc
```

形式不正は、

```text id="l8n0vg"
INVALID_ASSET_ACCOUNT_ID
```

へ変換する。

---

### 28.5 Action

Actionは、パスパラメータと利用者コンテキストを受け取り、UseCaseを呼び出す。

概念例：

```php id="vg1yuc"
final class ShowAssetAccountAction
{
    public function __invoke(
        string $assetAccountId,
        ShowAssetAccountUseCase $useCase,
        ShowAssetAccountResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                assetAccountId:
                    (int) $assetAccountId,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

`assetAccountId`は、形式検証済みであることを前提に整数へ変換する。

---

### 28.6 Actionで行わないこと

Actionでは、以下を行わない。

* 資産口座検索
* 利用者境界判定
* 論理削除判定
* 現在年月算出
* 利用可能資産設定検索
* 設定件数判定
* データ不整合判定
* Enum変換
* レスポンス配列生成
* SQL生成

Actionは、HTTP層とUseCaseの橋渡しに責務を限定する。

---

### 28.7 UseCase

ACC-003の参照ユースケース全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `assetAccountId`を受け取る
3. 操作対象利用者に属する有効な資産口座を取得する
4. 存在しない場合は業務例外を送出する
5. 現在年月を取得する
6. 現在有効な利用可能資産設定を取得する
7. 設定件数が1件であることを確認する
8. Detail Result DTOを生成する
9. DTOを返却する

概念的には、以下とする。

```text id="ue38c9"
userId
+
assetAccountId
    ↓
AssetAccountQuery
    ↓
資産口座取得
    ↓
存在確認
    ↓
CurrentYearMonth取得
    ↓
AssetAccountAvailableSettingQuery
    ↓
現在有効設定取得
    ↓
件数確認
    ↓
Detail Result DTO
```

---

### 28.8 AssetAccountQuery

資産口座取得は、専用Queryへ委譲する。

概念例：

```php id="jm6wga"
$assetAccount =
    $this->assetAccountQuery
        ->findActiveByIdAndUser(
            assetAccountId:
                $assetAccountId,

            userId:
                $userId,
        );
```

概念的な検索条件は、以下とする。

```text id="lpf1z9"
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

---

### 28.9 利用者境界をQueryへ含める

以下のように、まずIDだけで取得してからUseCase側で利用者IDを比較する方式を基本としない。

```php id="3nnwlq"
$assetAccount =
    AssetAccount::find(
        $assetAccountId,
    );
```

対象取得時点から、

```text id="nszbf2"
assetAccountId
+
userId
+
有効状態
```

を条件へ含める。

これにより、他利用者の資産口座を誤って取得することを防止する。

---

### 28.10 SoftDeletes

`AssetAccount` Modelでは、Laravelの

```php id="p3tf0c"
use SoftDeletes;
```

を使用する。

ACC-003の通常取得では、

```php id="xv7olh"
withTrashed()
```

を使用しない。

論理削除済み資産口座は、詳細取得対象外とする。

---

### 28.11 ASSET_ACCOUNT_NOT_FOUND

資産口座を取得できない場合は、

```php id="dfwx7a"
if ($assetAccount === null) {
    throw new
        AssetAccountNotFoundException();
}
```

のように業務例外を送出する。

以下を同じ例外へ集約する。

* 資産口座不存在
* 他利用者に属する
* 論理削除済み

最終的に、

```text id="eizmy1"
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

へ変換する。

---

### 28.12 CurrentYearMonth

利用可能資産設定の判定に使用する現在年月は、UseCase内でサーバー時刻から取得する。

概念例：

```php id="a8fwmr"
$currentYearMonth =
    now()->format(
        'Y-m',
    );
```

ただし、テスト容易性を高めるため、現在時刻取得を直接`now()`へ依存させず、ClockやDateProviderを利用してもよい。

概念例：

```php id="2o8hvr"
$currentYearMonth =
    $this->clock
        ->now()
        ->format(
            'Y-m',
        );
```

---

### 28.13 Clockを分離してよい理由

ACC-003の

```text id="sge4w7"
isAvailable
```

は、現在年月によって結果が変化する。

そのため、時刻依存処理をテストで固定しやすくするために、現在時刻取得を専用インターフェースへ分離してよい。

例えば、

```php id="t67d9c"
interface Clock
{
    public function now(): CarbonImmutable;
}
```

とする。

Phase1で過剰になる場合は、Laravelの時刻固定機能を使い、`now()`を直接使用してもよい。

---

### 28.14 AssetAccountAvailableSettingQuery

現在有効な利用可能資産設定の取得は、専用Queryへ委譲する。

概念例：

```php id="k0zmf2"
$settings =
    $this->availableSettingQuery
        ->findCurrentByAssetAccount(
            assetAccountId:
                $assetAccount->id,

            yearMonth:
                $currentYearMonth,
        );
```

概念的な検索条件は、以下とする。

```text id="fyso03"
asset_account_id
    = assetAccountId

AND

start_year_month
    <= currentYearMonth

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= currentYearMonth
)
```

---

### 28.15 設定を全件取得しない

以下のように、利用可能資産設定の全履歴を取得してからPHP側で現在設定を探す方式は基本としない。

```php id="mw4e40"
$settings =
    AssetAccountAvailableSetting::query()
        ->where(
            'asset_account_id',
            $assetAccountId,
        )
        ->get();
```

データベース側で現在年月条件を指定する。

---

### 28.16 件数判定

現在有効な設定は、正常状態では1件だけ存在することを前提とする。

概念例：

```php id="l2i3vk"
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingNotFoundException();
}

if ($settings->count() > 1) {
    throw new
        AssetAccountAvailableSettingConflictException();
}
```

0件または複数件を正常状態として扱わない。

---

### 28.17 `first()`だけで済ませない

以下のような実装は基本としない。

```php id="y3pfwy"
$setting =
    $query
        ->orderByDesc(
            'start_year_month',
        )
        ->first();
```

これでは、設定期間が重複していても最新1件を返してデータ不整合を隠蔽する可能性がある。

ACC-003では、該当件数を確認し、1件であることを保証する。

---

### 28.18 取得上限

複数件存在するかを検出するだけであれば、全件を取得する必要はない。

概念的には、最大2件まで取得してよい。

```php id="87fmxa"
$settings =
    AssetAccountAvailableSetting::query()
        ->where(...)
        ->limit(2)
        ->get();
```

これにより、

```text id="v9j3fp"
0件
1件
2件以上
```

を判定できる。

---

### 28.19 Repositoryを使用しない

ACC-003は参照専用APIであり、以下のDB更新を行わない。

```text id="mjkqun"
INSERT
UPDATE
DELETE
```

そのため、ACC-003専用のRepositoryは使用しない。

概念的には、

```text id="py7f2t"
Query
    → 読み取り

Repository
    → 書き込み
```

という責務分離方針に従う。

---

### 28.20 トランザクション

ACC-003では、明示的な

```php id="0g6kuw"
DB::transaction()
```

を使用しない。

参照専用であり、表示用途として厳密な同一時点スナップショットを要求しないためである。

---

### 28.21 lockForUpdateを使用しない

ACC-003では、以下を使用しない。

```php id="al5fse"
lockForUpdate()
```

資産口座や利用可能資産設定を参照するだけであり、詳細表示のために更新処理をブロックしない。

---

### 28.22 Enum Cast

`asset_type`と`balance_recording_unit`は、可能であればEloquent CastまたはPHP Enumとして扱う。

概念例：

```php id="e3d4u0"
protected function casts(): array
{
    return [
        'asset_type'
            => AssetType::class,

        'balance_recording_unit'
            => BalanceRecordingUnit::class,
    ];
}
```

DB保存形式がAPI公開値と異なる場合は、Mapper等で変換する。

---

### 28.23 DB内部コードをUseCaseへ漏らさない

例えば、

```text id="gtq6xq"
1 = CASH
2 = BANK
3 = SECURITIES
```

のようなDB内部コードを、UseCaseやResourceへ直接持ち込まない。

UseCase以降では、

```text id="8prwli"
AssetType
BalanceRecordingUnit
```

という意味のある型として扱う。

---

### 28.24 Detail Result DTO

取得結果は、専用DTOとして表現する。

概念例：

```php id="8qey2m"
final readonly class AssetAccountDetailResult
{
    public function __construct(
        public int $id,
        public string $name,
        public AssetType $assetType,
        public BalanceRecordingUnit $balanceRecordingUnit,
        public bool $isAvailable,
        public string $startYearMonth,
    ) {
    }
}
```

`isEnabled`は、ACC-003正常時に必ず`true`となるため、Resource側で固定値として設定してよい。

---

### 28.25 DTOへEloquent Modelを保持しない

以下のようなDTOは基本としない。

```php id="0rbp13"
final readonly class AssetAccountDetailResult
{
    public function __construct(
        public AssetAccount $assetAccount,
        public AssetAccountAvailableSetting $setting,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

これにより、HTTP層へEloquent Modelを直接渡さない。

---

### 28.26 UseCaseの戻り値

概念例：

```php id="m1zp84"
return new AssetAccountDetailResult(
    id:
        $assetAccount->id,

    name:
        $assetAccount->name,

    assetType:
        $assetAccount->asset_type,

    balanceRecordingUnit:
        $assetAccount->balance_recording_unit,

    isAvailable:
        $setting->is_available,

    startYearMonth:
        $assetAccount->start_year_month,
);
```

---

### 28.27 API Resource

Detail Result DTOを、専用API Resourceでレスポンス形式へ変換する。

概念例：

```php id="d61q3c"
final class AssetAccountDetailResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'name'
                => $this->name,

            'assetType'
                => $this->assetType->value,

            'balanceRecordingUnit'
                => $this->balanceRecordingUnit->value,

            'isAvailable'
                => $this->isAvailable,

            'startYearMonth'
                => $this->startYearMonth,

            'isEnabled'
                => true,
        ];
    }
}
```

APIフィールド名は、共通方針に従ってcamelCaseとする。

---

### 28.28 API Resourceで返却しない情報

以下をレスポンスへ返却しない。

* `user_id`
* `deleted_at`
* `created_at`
* `updated_at`
* `asset_account_available_settings.id`
* `asset_account_available_settings.asset_account_id`
* `asset_account_available_settings.start_year_month`
* `asset_account_available_settings.end_year_month`
* DB内部コード

また、以下の関連データも返却しない。

* 保有商品一覧
* 月末資産残高
* 商品別月末評価額
* 月末資産状況

---

### 28.29 Responder

Responderは、Detail Result DTOを`200 OK`レスポンスへ変換する。

概念例：

```php id="os6g3n"
final class ShowAssetAccountResponder
{
    public function ok(
        AssetAccountDetailResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    => new AssetAccountDetailResource(
                        $result,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

`requestId`などの共通Envelope項目は、API共通レスポンス処理に従う。

---

### 28.30 Responderで行わないこと

Responderでは、以下を行わない。

* 資産口座検索
* 利用者境界判定
* 現在年月取得
* 利用可能資産設定検索
* 設定件数判定
* データ不整合判定
* DBアクセス
* Enum変換ロジックの組み立て

Responderは、生成済みDTOをHTTPレスポンスへ変換することに責務を限定する。

---

### 28.31 例外変換

主な例外変換は、以下とする。

| 内部状態                 | 独自エラーコード                                    |
| -------------------- | ------------------------------------------- |
| `X-User-Id`未指定       | `USER_CONTEXT_REQUIRED`                     |
| `X-User-Id`形式不正      | `INVALID_USER_ID`                           |
| 利用者不存在               | `USER_NOT_FOUND`                            |
| `assetAccountId`形式不正 | `INVALID_ASSET_ACCOUNT_ID`                  |
| 資産口座不存在              | `ASSET_ACCOUNT_NOT_FOUND`                   |
| 他利用者の資産口座            | `ASSET_ACCOUNT_NOT_FOUND`                   |
| 論理削除済み資産口座           | `ASSET_ACCOUNT_NOT_FOUND`                   |
| 現在設定0件               | `ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND` |
| 現在設定複数件              | `ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT`  |
| 想定外例外                | `INTERNAL_SERVER_ERROR`                     |

---

### 28.32 AssetAccountNotFoundException

対象資産口座を取得できない場合は、業務例外を送出する。

概念例：

```php id="m9svb7"
if ($assetAccount === null) {
    throw new
        AssetAccountNotFoundException();
}
```

最終的に、

```text id="k3j57n"
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

へ変換する。

---

### 28.33 AssetAccountAvailableSettingNotFoundException

現在有効な利用可能資産設定が0件の場合は、データ不整合として専用例外を送出する。

概念例：

```php id="pijztt"
if ($settings->isEmpty()) {
    throw new
        AssetAccountAvailableSettingNotFoundException();
}
```

最終的に、

```text id="d7a8t1"
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

へ変換する。

---

### 28.34 AssetAccountAvailableSettingConflictException

現在有効な設定が複数件存在する場合は、専用例外を送出する。

概念例：

```php id="hmzqvi"
if ($settings->count() > 1) {
    throw new
        AssetAccountAvailableSettingConflictException();
}
```

最終的に、

```text id="y0ne3a"
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

へ変換する。

---

### 28.35 データ不整合時に自動修復しない

以下のような処理は行わない。

```text id="p5r4n6"
現在設定0件
    ↓
isAvailable = false
```

または、

```text id="5pf1x4"
現在設定複数件
    ↓
最新1件採用
```

ACC-003は参照APIであるため、不整合状態を暗黙的に修復しない。

---

### 28.36 想定外例外

想定外の例外は、API共通Exception Handlerで

```text id="zps3jg"
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

* SQL
* PostgreSQL内部エラー
* 制約名
* テーブル名
* カラム名
* Laravel内部例外メッセージ
* PHP内部エラー
* スタックトレース
* サーバーファイルパス

---

### 28.37 ログ

ACC-003では、必要に応じて以下をログコンテキストへ設定する。

```text id="03ox3h"
requestId
userId
apiId
assetAccountId
currentYearMonth
httpStatus
errorCode
```

`apiId`は、

```text id="cd29r4"
ACC-003
```

とする。

正常時に、資産口座名や`isAvailable`などの不要な業務情報をログへ出力しない。

---

### 28.38 データ不整合ログ

以下のエラー時は、調査に必要な情報を内部ログへ記録する。

```text id="316ihd"
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

必要に応じて、

```text id="ghaq9s"
requestId
userId
assetAccountId
currentYearMonth
matchedSettingCount
```

を記録してよい。

レスポンスへは内部状態を公開しない。

---

### 28.39 キャッシュ

Phase1では、ACC-003専用のサーバー側キャッシュを使用しない。

毎回、最新の

```text id="aq1e79"
asset_accounts
asset_account_available_settings
```

を参照する。

フロントエンド側では、TanStack Query等によるQuery Cacheを使用してよい。

---

### 28.40 テスト実装方針

Laravel側では、Feature Testを中心としてACC-003のAPI契約を確認する。

主に以下を確認する。

* `200 OK`
* `400 Bad Request`
* `404 Not Found`
* `500 Internal Server Error`
* `X-User-Id`必須
* 利用者存在確認
* `assetAccountId`形式
* 資産口座存在確認
* 利用者境界
* SoftDeletes
* 現在年月判定
* 利用可能資産設定の期間境界
* 設定0件
* 設定複数件
* API用Enum変換
* 副作用なし
* レスポンス契約

---

### 28.41 AssetAccountQueryのDatabase Test

`AssetAccountQuery`について、以下を確認する。

```text id="i6yq2u"
id一致
+
user_id一致
+
deleted_at IS NULL
    ↓
取得できる
```

以下の場合は取得できないことを確認する。

* 存在しないID
* 他利用者の資産口座
* 論理削除済み資産口座

---

### 28.42 AssetAccountAvailableSettingQueryのDatabase Test

現在年月を固定し、以下を確認する。

```text id="cdg71e"
start_year_month
    <= currentYearMonth

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= currentYearMonth
)
```

境界条件として、以下を確認する。

* `start_year_month = currentYearMonth`
* `end_year_month = currentYearMonth`
* `end_year_month = 前月`
* `start_year_month = 翌月`
* `end_year_month = NULL`

---

### 28.43 設定0件のTest

現在有効な利用可能資産設定を0件にする。

期待結果：

```text id="2qw7wm"
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

となること。

また、

```text id="ksgt0x"
isAvailable = false
```

へ補完されないことを確認する。

---

### 28.44 設定複数件のTest

現在年月に有効な設定を2件以上用意する。

期待結果：

```text id="8way71"
500 Internal Server Error
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

となること。

最新設定を任意に返却しないことを確認する。

---

### 28.45 UseCaseのUnit Test

UseCaseでは、QueryをMockして業務フローを確認する。

正常系：

```text id="qpsj7q"
AssetAccountQuery
    ↓
資産口座取得
    ↓
AvailableSettingQuery
    ↓
現在設定1件
    ↓
Detail Result DTO
```

資産口座不存在：

```text id="yhnojt"
AssetAccountQuery
    ↓
null
    ↓
AssetAccountNotFoundException
```

設定0件：

```text id="s6a43f"
AvailableSettingQuery
    ↓
0件
    ↓
AssetAccountAvailableSettingNotFoundException
```

設定複数件：

```text id="7kzzp1"
AvailableSettingQuery
    ↓
2件
    ↓
AssetAccountAvailableSettingConflictException
```

---

### 28.46 時刻依存テスト

現在年月判定では、テスト実行時刻を固定する。

例えば、

```text id="udf7ie"
2026-08-15
```

に固定し、

```text id="uq5jrm"
currentYearMonth = 2026-08
```

として利用可能資産設定を判定する。

Laravelの時刻固定機能またはClockのTest Doubleを使用し、実行日によってテスト結果が変わらないようにする。

---

### 28.47 API ResourceのTest

正常時に、以下の項目だけが`data`へ含まれることを確認する。

```json id="oj1k4l"
{
  "id": "10",
  "name": "証券口座",
  "assetType": "SECURITIES",
  "balanceRecordingUnit": "HOLDING",
  "isAvailable": true,
  "startYearMonth": "2026-08",
  "isEnabled": true
}
```

以下が含まれないことを確認する。

* `userId`
* `user_id`
* `deletedAt`
* `createdAt`
* `updatedAt`
* `assetAccountAvailableSettingId`
* `asset_account_id`
* `start_year_month`
* `end_year_month`
* DB内部の数値コード

---

### 28.48 副作用なしのTest

ACC-003実行前後で、以下のテーブルが変更されないことを確認する。

```text id="1pmhv3"
asset_accounts
asset_account_available_settings
```

特に、

```text id="slz8dh"
updated_at
```

が詳細取得によって更新されないことを確認する。

---

## 29. React・TypeScriptでの利用

ACC-003は、資産口座一覧や資産口座編集画面から、指定した資産口座の詳細情報を取得するために使用する。

主な利用フローは、以下とする。

```text
ACC-001
資産口座一覧取得
    ↓
利用者が資産口座を選択
    ↓
ACC-003
資産口座詳細取得
    ↓
詳細表示
または
編集フォーム初期値へ反映
```

ACC-003は参照専用APIであるため、TanStack Queryを使用する場合はQueryとして扱う。

---

### 29.1 TypeScript型

ACC-003の正常レスポンスデータは、以下のような型として扱う。

概念例：

```ts
export type AssetAccountDetail = {
  id: string;
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  isAvailable: boolean;
  startYearMonth: string;
  isEnabled: boolean;
};
```

API共通Envelopeを使用する場合は、以下のように定義する。

```ts
export type AssetAccountDetailResponse =
  ApiResponse<AssetAccountDetail>;
```

---

### 29.2 AssetType

`assetType`は、ACC-002などと共通の型を使用する。

概念例：

```ts
export type AssetType =
  | 'CASH'
  | 'BANK'
  | 'SECURITIES'
  | 'IDECO'
  | 'CORPORATE_DC'
  | 'OTHER';
```

ACC-003専用として同じUnion型を重複定義しない。

---

### 29.3 BalanceRecordingUnit

`balanceRecordingUnit`も、ACC-002などと共通の型を使用する。

概念例：

```ts
export type BalanceRecordingUnit =
  | 'ACCOUNT'
  | 'HOLDING';
```

---

### 29.4 assetAccountId

ACC-003では、取得対象となる`assetAccountId`をAPI Client関数の引数として渡す。

概念例：

```ts
getAssetAccountDetail(
  assetAccountId,
);
```

`assetAccountId`は、API契約に合わせて`string`として扱う。

---

### 29.5 userIdを関数引数へ含めない

ACC-003専用のAPI Client関数へ、

```ts
getAssetAccountDetail(
  userId,
  assetAccountId,
);
```

のように`userId`を渡さない。

利用者IDは、`X-User-Id`として共通API Clientから付与する。

---

### 29.6 X-User-Id

`X-User-Id`は、ACC-003専用処理ではなく、共通API Clientから付与する。

概念例：

```ts
apiClient.interceptors.request.use(
  (config) => {
    config.headers['X-User-Id'] =
      currentUserId;

    return config;
  },
);
```

ACC-003を呼び出すコンポーネントから利用者ヘッダーを直接組み立てない。

---

### 29.7 API Client

ACC-003を呼び出す専用関数を定義する。

概念例：

```ts
export const getAssetAccountDetail =
  async (
    assetAccountId: string,
  ): Promise<AssetAccountDetail> => {
    const response =
      await apiClient.get<
        AssetAccountDetailResponse
      >(
        `/api/v1/asset-accounts/${assetAccountId}`,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

### 29.8 Queryとして扱う

ACC-003は参照専用APIであるため、TanStack Queryを使用する場合はQueryとして扱う。

概念例：

```ts
export const useAssetAccountDetail =
  (
    assetAccountId: string,
  ) =>
    useQuery({
      queryKey: [
        'assetAccounts',
        assetAccountId,
      ],
      queryFn: () =>
        getAssetAccountDetail(
          assetAccountId,
        ),
    });
```

---

### 29.9 Query Key

資産口座詳細のQuery Keyは、資産口座IDを含める。

概念的には、

```text
assetAccounts
+
assetAccountId
```

とする。

例えば、

```ts
[
  'assetAccounts',
  assetAccountId,
]
```

とする。

---

### 29.10 Query Keyの共通化

Query Keyは、各コンポーネントへ直接記述せず、共通定義として管理してよい。

概念例：

```ts
export const assetAccountKeys = {
  all: [
    'assetAccounts',
  ] as const,

  detail: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
    ] as const,
};
```

ACC-003では、

```ts
queryKey:
  assetAccountKeys.detail(
    assetAccountId,
  ),
```

として使用できる。

---

### 29.11 assetAccountIdが存在しない場合

画面初期化時などで`assetAccountId`がまだ取得できていない場合は、Queryを実行しない。

概念例：

```ts
useQuery({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),

  queryFn: () =>
    getAssetAccountDetail(
      assetAccountId,
    ),

  enabled:
    assetAccountId !== '',
});
```

不完全なIDでACC-003を実行しない。

---

### 29.12 ルートパラメータから取得する場合

詳細画面では、React RouterなどのURLパラメータから`assetAccountId`を取得してよい。

概念例：

```ts
const {
  assetAccountId,
} = useParams<{
  assetAccountId: string;
}>();
```

ただし、`assetAccountId`が存在することを確認してからACC-003を実行する。

---

### 29.13 ローディング表示

ACC-003取得中は、ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return (
    <Loading />
  );
}
```

取得前の空データを正常な詳細情報として表示しない。

---

### 29.14 正常取得

正常時は、ACC-003から返却された`AssetAccountDetail`を画面表示へ使用する。

概念例：

```ts
const {
  data,
} = useAssetAccountDetail(
  assetAccountId,
);
```

---

### 29.15 詳細表示

詳細画面では、必要に応じて以下を表示する。

* 資産口座名
* 資産種別
* 残高記録単位
* 利用開始年月
* 現在の利用可能資産区分
* 利用状態

---

### 29.16 assetTypeの表示

APIから返却された`SECURITIES`などを、画面表示では日本語ラベルへ変換する。

概念例：

```ts
export const assetTypeLabels:
  Record<AssetType, string> = {
    CASH: '現金',
    BANK: '銀行',
    SECURITIES: '証券',
    IDECO: 'iDeCo',
    CORPORATE_DC: '企業型DC',
    OTHER: 'その他',
  };
```

API値自体を日本語へ変換してStateへ保存し直す必要はない。

---

### 29.17 balanceRecordingUnitの表示

概念例：

```ts
export const balanceRecordingUnitLabels:
  Record<
    BalanceRecordingUnit,
    string
  > = {
    ACCOUNT: '口座単位',
    HOLDING: '商品単位',
  };
```

---

### 29.18 isAvailableの表示

`isAvailable`は、現在の利用可能資産区分として表示する。

概念例：

```tsx
<span>
  {assetAccount.isAvailable
    ? '利用可能資産'
    : '利用対象外'}
</span>
```

表示文言は、画面設計で定義した用語に統一する。

---

### 29.19 isEnabledの表示

ACC-003の正常レスポンスでは、`isEnabled = true`となる。

そのため、通常の詳細画面で利用状態を表示する場合は、

```text
利用中
```

として表示できる。

ただし、論理削除済み資産口座はACC-003で取得できないため、ACC-003だけを利用する画面で

```text
無効
```

状態を表示する必要はない。

---

### 29.20 startYearMonthの表示

`startYearMonth`は、

```text
YYYY-MM
```

形式で返却される。

画面表示では、必要に応じて

```text
2026年8月
```

のように表示用フォーマットへ変換してよい。

API値そのものは変更しない。

---

### 29.21 編集画面の初期値

ACC-003は、ACC-004 資産口座更新画面の初期値取得にも使用できる。

概念的には、

```text
ACC-003
    ↓
name
assetType
balanceRecordingUnit
startYearMonth
isAvailable
    ↓
編集フォーム初期値
```

とする。

ただし、ACC-004が実際に更新可能とする項目だけを編集フォームへ反映する。

---

### 29.22 Form Stateへの変換

詳細DTOと編集フォームStateは別型として定義してよい。

概念例：

```ts
export type AssetAccountFormValues = {
  name: string;
  assetType: AssetType;
  balanceRecordingUnit:
    BalanceRecordingUnit;
  startYearMonth: string;
  isAvailable: boolean;
};
```

変換例：

```ts
const initialValues:
  AssetAccountFormValues = {
    name:
      assetAccount.name,

    assetType:
      assetAccount.assetType,

    balanceRecordingUnit:
      assetAccount.balanceRecordingUnit,

    startYearMonth:
      assetAccount.startYearMonth,

    isAvailable:
      assetAccount.isAvailable,
  };
```

---

### 29.23 詳細DTOを直接編集しない

ACC-003で取得した`AssetAccountDetail`を、そのまま編集用Stateとして直接書き換えない。

Query Cache上のデータをフォーム入力によって直接変更しないためである。

編集用Stateへ必要な値をコピーする。

---

### 29.24 ASSET_ACCOUNT_NOT_FOUND

ACC-003で

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

が返却された場合は、対象資産口座を表示できない状態として扱う。

例えば、

```text
指定された資産口座が
見つかりません。
```

と表示する。

---

### 29.25 他利用者かどうかを推測しない

`ASSET_ACCOUNT_NOT_FOUND`には、以下が含まれる。

* 資産口座不存在
* 他利用者の資産口座
* 論理削除済み資産口座

フロントエンドでは、どの理由かを推測して表示を分岐しない。

---

### 29.26 論理削除済みかどうかを推測しない

ACC-003から`ASSET_ACCOUNT_NOT_FOUND`が返却されても、

```text
この資産口座は
無効化されています
```

と断定しない。

API契約上、無効化済みか不存在かは区別されないためである。

---

### 29.27 INVALID_ASSET_ACCOUNT_ID

通常の画面遷移では、サーバーから取得した有効なIDを使用するため、発生頻度は低い。

URL手入力などにより

```text
INVALID_ASSET_ACCOUNT_ID
```

が返却された場合は、不正なURLまたは不正な画面状態として扱う。

例えば、資産口座一覧へ戻す導線を表示してよい。

---

### 29.28 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ACC-003画面だけで独自処理を実装しない。

---

### 29.29 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、API共通の利用者コンテキストエラーとして扱う。

必要に応じて、現在選択している利用者状態を再確認する。

---

### 29.30 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合も、API共通の利用者コンテキストエラーとして扱う。

現在選択されている利用者が有効ではない状態として処理する。

---

### 29.31 利用可能資産設定不存在

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
```

は、フロントエンド入力によるエラーではなく、サーバー側の業務データ不整合である。

画面では、一般的な取得失敗として扱う。

例えば、

```text
資産口座の詳細を取得できませんでした。
```

と表示する。

内部の

```text
利用可能資産設定が存在しない
```

という詳細を一般利用者向け画面へそのまま表示する必要はない。

---

### 29.32 利用可能資産設定重複

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

についても、サーバー側データ不整合として扱う。

フロントエンドで複数設定から独自に1件を選択して画面表示しない。

---

### 29.33 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
資産口座の詳細を取得できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

### 29.34 Retry

ACC-003は参照専用GET APIであるため、一時的な通信エラーについてTanStack QueryのRetry機能を利用してよい。

ただし、

```text
ASSET_ACCOUNT_NOT_FOUND
```

やデータ不整合系エラーに対して、無意味な再試行を繰り返さない。

正式なRetry方針は、フロントエンド共通設計に従う。

---

### 29.35 Query Cache

ACC-003の結果は、TanStack QueryのQuery Cacheへ保持してよい。

ただし、以下のAPI成功後はキャッシュが古くなる可能性がある。

```text
ACC-004
資産口座更新

ACC-005
資産口座無効化

ACC-006
利用可能資産設定登録
```

そのため、各Mutation成功時に対象詳細Queryをinvalidateする。

---

### 29.36 ACC-004成功後

ACC-004によって資産口座情報が更新された場合は、対象資産口座の詳細Queryを無効化する。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

必要に応じてACC-001一覧Queryも無効化する。

---

### 29.37 ACC-005成功後

ACC-005によって資産口座が無効化された場合は、ACC-003の通常取得対象から外れる。

そのため、対象詳細Queryを無効化または削除する。

概念例：

```ts
queryClient.removeQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

その後、資産口座一覧画面へ遷移してよい。

---

### 29.38 ACC-006成功後

ACC-006によって現在の利用可能資産区分が変更された場合は、ACC-003の`isAvailable`が変化する可能性がある。

そのため、対象詳細Queryをinvalidateする。

---

### 29.39 ACC-002成功直後

ACC-002成功後に登録した資産口座の詳細画面へ遷移する場合は、返却された`data.id`を使用してACC-003を実行する。

ACC-002のレスポンスだけをACC-003の永続的なキャッシュ代替としない。

---

### 29.40 現在年月による変化

ACC-003の`isAvailable`は、現在年月によって変化する可能性がある。

例えば、

```text
2026-08
    isAvailable = true

2026-09
    isAvailable = false
```

となる設定履歴が存在する場合、月が変わればDB更新がなくてもACC-003の結果が変化し得る。

そのため、非常に長い`staleTime`を無条件に設定しない。

具体的なキャッシュ時間は、フロントエンド共通設計で決定する。

---

### 29.41 userId変更時

利用者切替が行われた場合は、前利用者のACC-003結果をそのまま表示し続けない。

Query Cacheを利用者単位で適切に分離または無効化する。

API自体は`X-User-Id`によって利用者境界を保証するが、画面上でも前利用者のキャッシュを誤表示しないようにする。

---

### 29.42 Query Keyと利用者境界

資産口座IDがシステム全体で一意であっても、利用者切替機能があるため、必要に応じてQuery Keyへ利用者IDを含めてもよい。

概念例：

```ts
export const assetAccountKeys = {
  detail: (
    userId: string,
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      userId,
      assetAccountId,
    ] as const,
};
```

ただし、Query Keyの正式な設計はReact共通設計に従う。

---

### 29.43 フロントエンドで現在設定を再計算しない

ACC-003では、バックエンドが現在年月を基準として`isAvailable`を判定する。

フロントエンドでは、利用可能資産設定履歴を取得して独自に

```text
現在有効なのはどの設定か
```

を再計算しない。

ACC-003から返却された`isAvailable`を現在状態として使用する。

---

### 29.44 DB内部コードを扱わない

React側では、以下のようなDB内部コードを扱わない。

```text
asset_type = 3
balance_recording_unit = 2
```

APIから返却された

```text
SECURITIES
HOLDING
```

などの業務上意味のある値を使用する。

---

### 29.45 DBカラム名を型へ持ち込まない

ACC-003の型では、以下のようなsnake_caseを使用しない。

```ts
type AssetAccountDetail = {
  asset_type: string;
  balance_recording_unit: string;
  start_year_month: string;
  is_available: boolean;
};
```

API契約に従って、camelCaseで定義する。

---

### 29.46 関連データをACC-003から期待しない

ACC-003には、以下を含めない。

* 保有商品一覧
* 月末資産残高
* 商品別月末評価額
* 月末資産状況
* 資産推移

画面でこれらが必要な場合は、対応するAPIを別途利用する。

---

### 29.47 HOLDINGの場合

```text
balanceRecordingUnit = HOLDING
```

であっても、ACC-003レスポンスに保有商品一覧は含まれない。

画面で保有商品を表示する場合は、HLD-001を実行する。

概念的には、

```text
ACC-003
    ↓
資産口座詳細

HLD-001
    ↓
保有商品一覧
```

と分離する。

---

### 29.48 ACCOUNTの場合

```text
balanceRecordingUnit = ACCOUNT
```

の場合でも、ACC-003から月末残高を取得しない。

月末資産残高が必要な場合は、対応する月末資産APIを利用する。

---

### 29.49 Page・Query・表示コンポーネントの分離

画面実装では、例えば以下の責務に分離する。

```text
Page
    ↓
assetAccountId取得
エラー時画面遷移

Query Hook
    ↓
ACC-003実行
Query Cache管理

API Client
    ↓
HTTP通信

Detail Component
    ↓
詳細表示

Type
    ↓
API契約
```

1つのコンポーネントへすべての処理を直接記述しない。

---

### 29.50 概念的なディレクトリ構成

例えば、以下のように機能単位で整理できる。

```text
features/
└── asset-accounts/
    ├── api/
    │   └── getAssetAccountDetail.ts
    ├── components/
    │   └── AssetAccountDetail.tsx
    ├── hooks/
    │   └── useAssetAccountDetail.ts
    ├── types/
    │   └── assetAccount.ts
    └── pages/
        └── AssetAccountDetailPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

### 29.51 フロントエンドで行わないこと

ACC-003のReact・TypeScript実装では、以下をフロントエンドの責務としない。

* 利用者境界の最終保証
* 論理削除状態の判定
* 現在年月の業務判定
* 利用可能資産設定の期間判定
* 利用可能資産設定の重複判定
* データ不整合の補正
* DB内部コードの解釈
* 保有商品の自動取得
* 月末資産データの自動取得

フロントエンドは、

```text
assetAccountId
    ↓
ACC-003
    ↓
レスポンス表示
```

という責務を基本とする。

---

## 30. 設計上の補足

### 30.1 現在の資産口座詳細を取得するAPIとする理由

ACC-003は、指定した資産口座の現在の詳細情報を取得するAPIとする。

そのため、過去時点の状態を指定する`targetYearMonth`は受け付けない。

ACC-003で返却する`isAvailable`は、サーバー側の現在年月を基準として現在有効な利用可能資産設定から判定する。

概念的には、以下とする。

```text
assetAccountId
    ↓
現在の資産口座
    +
現在の利用可能資産区分
```

---

### 30.2 `isAvailable`を`asset_accounts`から取得しない理由

利用可能資産区分は、対象年月によって変化する履歴情報である。

そのため、

```text
asset_accounts.is_available
```

のような固定カラムから現在状態を取得しない。

ACC-003では、

```text
asset_account_available_settings
```

の期間履歴から、現在年月に有効な設定を取得する。

---

### 30.3 現在年月をクライアントから指定させない理由

ACC-003は、現在状態を確認するAPIである。

クライアントから以下のような年月を受け付けると、

```text
targetYearMonth
currentYearMonth
```

「現在詳細取得」と「過去時点詳細取得」の責務が混在する。

そのため、現在年月はサーバー側で決定する。

---

### 30.4 過去時点の利用可能資産区分をACC-003で扱わない理由

月末資産管理や目的達成判定では、対象年月時点の利用可能資産区分が必要になる。

しかし、それらは以下の各業務処理の対象年月に基づいて判定すべき情報である。

```text
月末資産
目的達成判定
```

ACC-003へ過去年月判定まで持たせず、各API側で対象年月を基準として利用可能資産設定を参照する。

---

### 30.5 現在有効な設定を1件とする理由

利用可能資産区分は、ある対象年月について1つの状態に決定できる必要がある。

同じ年月について、

```text
is_available = true
```

と

```text
is_available = false
```

の両方が有効になる状態は許容しない。

そのため、ACC-003では現在有効な設定が

```text
1件
```

であることを正常状態とする。

---

### 30.6 設定0件を`false`へ補完しない理由

ACC-002では、資産口座登録時に初期利用可能資産設定を必ず登録する。

そのため、利用開始済みの資産口座について現在有効な設定が存在しない状態は、正常な業務状態ではない。

以下のように補完すると、

```text
設定なし
    ↓
isAvailable = false
```

データ不整合が画面上では正常値として見えてしまう。

そのため、設定不存在を明示的なデータ不整合として扱う。

---

### 30.7 設定複数件で最新1件を採用しない理由

現在年月に有効な設定が複数存在する場合に、

```sql
ORDER BY start_year_month DESC
LIMIT 1
```

として最新1件だけを返却すると、期間重複というデータ不整合を隠蔽する。

そのため、ACC-003では複数件を検出した時点で正常レスポンスを返却しない。

---

### 30.8 利用可能資産設定の期間判定をDB側で行う理由

ACC-003で必要なのは、現在有効な利用可能資産設定だけである。

そのため、

```text
全履歴取得
    ↓
PHP側で期間判定
```

ではなく、

```text
現在年月を条件に
DBから対象設定取得
```

とする。

不要なデータ取得を避け、期間条件をQueryへ集約する。

---

### 30.9 設定取得を最大2件としてよい理由

ACC-003で確認したいのは、

```text
0件
1件
2件以上
```

のいずれかである。

複数件存在する場合に全履歴を取得する必要はない。

そのため、実装上は

```sql
LIMIT 2
```

として取得し、

```text
0
1
2
```

を判定してよい。

これにより、不整合検出と不要なデータ取得抑制を両立できる。

---

### 30.10 他利用者の資産口座を404とする理由

他利用者に属する`assetAccountId`が指定された場合に、

```text
FORBIDDEN
```

などを返却すると、そのIDの資産口座が実際に存在することを外部から推測できる。

そのため、

```text
存在しない
他利用者に属する
論理削除済み
```

を同じ

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 30.11 論理削除済み資産口座を通常詳細取得対象としない理由

ACC-003は、現在利用中の資産口座を確認するためのAPIである。

論理削除済み資産口座まで通常詳細取得対象へ含めると、

```text
現在利用中
過去に利用していた
```

という異なる状態を同一APIで扱うことになる。

Phase1では、ACC-003を利用中資産口座の詳細取得に限定する。

---

### 30.12 `isEnabled`を返却する理由

ACC-003の正常レスポンスでは、取得対象が利用中であるため、

```text
isEnabled = true
```

となる。

これは、DBの

```text
deleted_at = NULL
```

という内部実装をそのまま公開せず、業務上の利用状態として表現するためである。

---

### 30.13 `deletedAt`を返却しない理由

`deleted_at`は、Laravel SoftDeletesによる内部実装上のカラムである。

クライアントが必要とするのは、

```text
現在利用できるか
```

という業務上の状態であり、論理削除日時そのものではない。

そのため、

```text
isEnabled
```

として公開し、`deletedAt`は返却しない。

---

### 30.14 正常レスポンスで`isEnabled = true`のみとなる理由

ACC-003では、論理削除済み資産口座を取得対象から除外する。

そのため、

```text
isEnabled = false
```

となる正常レスポンスは存在しない。

それでもACC-001やACC-002とのレスポンス表現を揃えるため、ACC-003でも`isEnabled`を返却する。

---

### 30.15 `assetType`をAPI用文字列で返す理由

DB内部で`smallint`などのコード値を使用していても、APIでは以下のような意味のある文字列を返却する。

```text
CASH
BANK
SECURITIES
IDECO
CORPORATE_DC
OTHER
```

これにより、クライアントをDB内部コードへ依存させない。

---

### 30.16 `balanceRecordingUnit`をAPI用文字列で返す理由

残高記録単位についても、DB内部コードではなく、

```text
ACCOUNT
HOLDING
```

として公開する。

これにより、React側でも型として明確に扱える。

---

### 30.17 保有商品を同時取得しない理由

`balanceRecordingUnit = HOLDING`の場合でも、ACC-003では保有商品一覧を返却しない。

資産口座詳細と保有商品一覧は、異なるリソース・業務責務である。

そのため、

```text
ACC-003
    → 資産口座詳細

HLD-001
    → 保有商品一覧
```

と分離する。

---

### 30.18 月末資産情報を同時取得しない理由

ACC-003へ以下を含めると、資産口座マスタ情報の詳細取得APIとして責務が広がりすぎる。

* 最新残高
* 商品別月末評価額
* 月末資産状況
* 資産推移

そのため、月末資産情報は専用APIへ委譲する。

---

### 30.19 編集画面の初期値取得に利用できる理由

ACC-003は、資産口座の現在の詳細状態を返す。

そのため、ACC-004 資産口座更新画面の初期表示にも利用できる。

概念的には、以下とする。

```text
ACC-003
    ↓
現在値取得
    ↓
編集フォーム初期値
```

ただし、ACC-003で返却する全項目がACC-004で更新可能とは限らない。

更新可能項目は、ACC-004側で定義する。

---

### 30.20 Repositoryを使用しない理由

ACC-003は、参照専用APIである。

そのため、

```text
INSERT
UPDATE
DELETE
```

を行わず、参照処理はQueryへ集約する。

概念的には、

```text
Query
    → 読み取り

Repository
    → 書き込み
```

という責務分離方針に従う。

---

### 30.21 UseCaseを設ける理由

ACC-003は単純な1テーブル取得ではなく、

```text
資産口座取得
    ↓
現在年月決定
    ↓
利用可能資産設定取得
    ↓
設定件数判定
    ↓
DTO生成
```

という業務フローを持つ。

そのため、ActionからQueryを直接組み合わせるのではなく、UseCaseへ処理の流れを集約する。

---

### 30.22 Queryを分ける理由

ACC-003では、

```text
AssetAccountQuery
```

と

```text
AssetAccountAvailableSettingQuery
```

を分離する。

資産口座本体の取得条件と、利用可能資産設定の期間条件は異なる責務を持つためである。

これにより、他APIからも必要なQueryを再利用しやすくする。

---

### 30.23 DTOを使用する理由

Queryで取得したEloquent ModelをそのままHTTP層へ渡すと、以下へAPI層が依存しやすくなる。

* DBカラム名
* Modelの関連
* Cast
* SoftDeletes状態

そのため、UseCaseからはレスポンスに必要な値だけを持つDTOを返却する。

---

### 30.24 API Resourceを使用する理由

API Resourceでは、

```text
snake_case
    ↓
camelCase
```

への変換や、

```text
bigint
    ↓
string
```

への変換、EnumのAPI表現への変換を行う。

DB Modelをそのままレスポンスへ返却しない。

---

### 30.25 FormRequestを作成しない理由

ACC-003には、

```text
リクエストボディ
クエリパラメータ
```

が存在しない。

専用FormRequestを作成しても空クラスとなるため、Phase1では作成しない。

`assetAccountId`は、ルート制約または共通パスパラメータ検証で扱う。

---

### 30.26 明示的なトランザクションを使用しない理由

ACC-003は参照専用であり、詳細表示のための通常参照処理である。

厳密な同一時点スナップショットを必要とする業務判断ではないため、

```php
DB::transaction()
```

を使用しない。

不要なトランザクションを追加しない。

---

### 30.27 `lockForUpdate()`を使用しない理由

ACC-003は、資産口座や利用可能資産設定を更新しない。

詳細表示のために更新処理をブロックする必要はないため、

```php
lockForUpdate()
```

を使用しない。

---

### 30.28 読み取り中の状態変化を許容する理由

ACC-003実行中に別APIによって以下が行われる可能性はある。

* 資産口座更新
* 資産口座無効化
* 利用可能資産設定変更

ただし、ACC-003は業務確定処理ではなく、現在状態の参照APIである。

そのため、厳密なSnapshot Isolationを要求せず、次回取得時に最新状態へ反映する方式とする。

---

### 30.29 Clockを分離してよい理由

ACC-003の`isAvailable`は、現在年月によって結果が変化する。

時刻取得を直接`now()`へ固定すると、テストで年月境界を扱う際に依存が強くなる。

そのため、必要に応じて以下へ分離してよい。

```text
Clock
DateProvider
```

ただし、Phase1で過剰になる場合は、Laravelの時刻固定機能を利用し、`now()`を直接使用してもよい。

---

### 30.30 データ不整合を500系として扱う理由

以下は、クライアント入力によって発生する状態ではない。

```text
現在有効な設定が0件
現在有効な設定が複数件
```

ACC-002やACC-006によって本来維持されるべき業務データ整合性が崩れている状態である。

そのため、Phase1ではサーバー側異常として500系で扱う。

---

### 30.31 データ不整合専用コードを返す理由

HTTPステータスだけでは、

```text
想定外例外
```

と

```text
利用可能資産設定の不整合
```

を区別できない。

そのため、内部調査やテストを容易にするため、以下のような独自エラーコードを定義する。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_CONFLICT
```

ただし、API全体でデータ不整合用コードを共通化する場合は、共通方針を優先する。

---

### 30.32 参照APIでデータを自動修復しない理由

ACC-003で不整合を検出した際に、

```text
設定なし
    → INSERT

設定重複
    → UPDATE
```

などを行うと、GET APIが副作用を持つことになる。

また、不整合の原因を隠してしまう。

そのため、ACC-003では検出だけを行い、データ修復は行わない。

---

### 30.33 Query Cacheを利用してよい理由

ACC-003は参照APIであり、React画面で同じ資産口座詳細を複数回参照する可能性がある。

そのため、TanStack QueryによるQuery Cacheを利用してよい。

ただし、更新系API成功後には適切にinvalidateする。

---

### 30.34 ACC-004成功後にキャッシュを無効化する理由

ACC-004によって資産口座の基本情報が変わると、ACC-003のキャッシュは古い状態になる。

そのため、対象資産口座の詳細Queryをinvalidateする。

---

### 30.35 ACC-005成功後に詳細キャッシュを破棄する理由

資産口座が無効化されると、ACC-003の通常取得対象から外れる。

古い詳細キャッシュを保持し続けると、無効化済み資産口座を利用中として表示する可能性がある。

そのため、ACC-005成功後は対象詳細Queryをinvalidateまたはremoveする。

---

### 30.36 ACC-006成功後にキャッシュを無効化する理由

ACC-006によって利用可能資産設定が変更されると、

```text
isAvailable
```

が変化する可能性がある。

そのため、ACC-003の詳細キャッシュを再取得対象とする。

---

### 30.37 現在年月の変化もキャッシュ失効要因となる理由

ACC-003の`isAvailable`は、DB更新だけでなく年月が変わることでも変化する可能性がある。

例えば、

```text
2026-08まで
isAvailable = true

2026-09から
isAvailable = false
```

という設定では、月をまたぐだけでレスポンスが変わる。

そのため、長期間固定するキャッシュには向かない。

Phase1ではサーバー側キャッシュを使用しない。

---

### 30.38 Optimistic Updateが不要な理由

ACC-003自体は参照APIである。

更新系APIの成功後は、

```text
invalidate
    ↓
ACC-003再取得
```

で最新状態へ同期できる。

詳細DTOを複数Mutationから手動で書き換えるより、Phase1ではサーバー状態を再取得する方式を基本とする。

---

### 30.39 利用者切替時にキャッシュを分離する理由

Phase1では、`X-User-Id`によって操作対象利用者を切り替える。

前の利用者で取得した資産口座詳細を、別利用者へ切り替えた後も画面表示してはならない。

そのため、Query Keyへ利用者情報を含める、または利用者切替時に関連Queryを無効化するなど、キャッシュ上でも利用者境界を維持する。

---

### 30.40 ACC-001との表現を揃える理由

ACC-001とACC-003で同じ意味を持つ項目については、同じ表現を使用する。

例えば、

```text
id
name
assetType
balanceRecordingUnit
isAvailable
startYearMonth
isEnabled
```

について、一覧と詳細で意味や型を変えない。

これにより、React側で共通型や共通表示部品を利用しやすくする。

---

### 30.41 ACC-002との表現を揃える理由

ACC-002成功時に返却した資産口座情報と、ACC-003で取得する資産口座詳細についても、同一項目は同じ形式を使用する。

登録直後と再取得後で表現が変わらないようにする。

---

### 30.42 Phase1では詳細取得を広げすぎない

ACC-003では、以下を追加しない。

* 利用可能資産設定履歴一覧
* 保有商品一覧
* 最新月末残高
* 商品別評価額
* 資産推移
* 過去年月指定
* 無効化済み資産口座取得
* ETag
* Last-Modified
* サーバー側キャッシュ

Phase1では、

```text
指定資産口座の現在詳細
+
現在の利用可能資産区分
```

を取得することへ責務を限定する。


---

## 31. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)