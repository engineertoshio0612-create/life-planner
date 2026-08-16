# ACC-005 利用可能資産設定履歴取得

## 1. 概要

操作対象となる利用者に帰属する指定された資産口座について、利用可能資産設定の履歴一覧を取得する。

利用可能資産区分は、資産口座の固定属性としてではなく、

```text
asset_account_available_settings
```

によって年月単位の履歴として管理する。

ACC-005では、指定された資産口座に紐づく利用可能資産設定履歴を取得し、過去から現在までの設定変更内容を確認できるようにする。

主に以下の情報を返却する。

- 利用可能資産設定ID
- 適用開始年月
- 適用終了年月
- 利用可能資産区分

概念的には、以下の情報を取得する。

```text
asset_accounts
    ↓
操作対象利用者との
利用者境界確認
    ↓
asset_account_available_settings
    ↓
設定履歴一覧
```

本APIは参照専用であり、資産口座および利用可能資産設定を更新しない。

---

### 1.1 利用可能資産設定とは

利用可能資産設定は、対象資産口座を目的達成判定に利用する資産として扱うかどうかを表す。

例えば、

```text
2026-01 ～ 2026-06
利用可能

2026-07 ～ 2026-12
利用対象外

2027-01 ～
利用可能
```

のように、年月単位で利用可能資産区分が変化する可能性がある。

そのため、現在値だけではなく設定履歴を保持する。

---

### 1.2 ACC-003との違い

ACC-003 資産口座詳細取得では、現在年月に有効な利用可能資産設定だけを取得し、

```text
isAvailable
```

として返却する。

一方、ACC-005では、指定された資産口座に対する利用可能資産設定履歴を一覧として返却する。

概念的には、

```text
ACC-003
    ↓
現在状態
    ↓
isAvailable

ACC-005
    ↓
設定履歴
    ↓
過去から現在までの
利用可能資産区分
```

と責務を分離する。

---

### 1.3 ACC-006との違い

ACC-005は、利用可能資産設定履歴を参照するAPIである。

利用可能資産区分を変更する場合は、

```text
ACC-006
利用可能資産設定登録
```

を使用する。

概念的には、

```text
ACC-005
    → 履歴参照

ACC-006
    → 新しい設定登録
```

とする。

ACC-005から設定履歴を変更しない。

---

## 2. ユースケース

利用者は、資産口座の利用可能資産区分が過去どのように設定されていたかを確認する際にACC-005を使用する。

主に以下の用途で使用する。

- 現在の利用可能資産設定を確認する
- 過去の利用可能資産区分を確認する
- 設定変更履歴を確認する
- ACC-006実行前に現在の設定状態を確認する
- 資産口座の設定内容を画面上で確認する

概念的な画面利用は、以下とする。

```text
ACC-001
資産口座一覧取得
    ↓
資産口座選択
    ↓
ACC-003
資産口座詳細取得
    ↓
利用可能資産設定履歴を表示
    ↓
ACC-005
利用可能資産設定履歴取得
```

必要に応じて、履歴画面からACC-006による新しい利用可能資産設定登録へ遷移できる。

---

### 2.1 履歴表示

ACC-005では、例えば以下のような履歴を表示できる。

```text
2027-01 ～ 現在
利用可能

2026-07 ～ 2026-12
利用対象外

2026-01 ～ 2026-06
利用可能
```

APIレスポンスでは、表示文言ではなく年月およびbooleanとして返却する。

画面表示用の文言は、React側で変換する。

---

### 2.2 現在設定の確認

最新の履歴について、

```text
endYearMonth = null
```

である場合は、現在継続中の設定として画面表示できる。

ただし、ACC-005自体は現在設定だけを返すAPIではなく、履歴全体を返却する。

---

## 3. エンドポイント

```http
GET /api/v1/asset-accounts/{assetAccountId}/available-settings
```

`assetAccountId`には、利用可能資産設定履歴を取得する資産口座IDを指定する。

例：

```http
GET /api/v1/asset-accounts/10/available-settings
```

---

### 3.1 資産口座配下のリソースとする理由

利用可能資産設定は、単独で存在するリソースではなく、特定の資産口座に紐づく設定履歴である。

そのため、

```text
asset-accounts
    ↓
available-settings
```

という親子関係をURLで表現する。

以下のようなトップレベルURLとはしない。

```http
GET /api/v1/asset-account-available-settings
```

Phase1では、資産口座を起点として利用可能資産設定履歴を取得する。

---

## 4. HTTPメソッド

```text
GET
```

ACC-005は、利用可能資産設定履歴を取得する参照APIであるため、`GET`を使用する。

本APIの実行によって、以下の業務データを更新しない。

- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `assessment_histories`

---

### 4.1 副作用を持たない

ACC-005実行によって、

```text
asset_account_available_settings.updated_at
```

などを変更しない。

また、履歴に不整合が存在した場合でも、GET処理内で自動修復しない。

---

## 5. 利用者コンテキスト

本APIは利用者依存APIのため、`X-User-Id`を必須とする。

操作対象となる利用者は、以下のリクエストヘッダーから特定する。

```http
X-User-Id: 1
```

利用者コンテキストの特定、`X-User-Id`の検証および利用者境界については、[API共通方針](../api-common-policy.md)に従う。

指定された`assetAccountId`に対応する資産口座が操作対象利用者に帰属する場合のみ、利用可能資産設定履歴を返却する。

概念的には、以下の条件を満たす必要がある。

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

### 5.1 利用者境界

ACC-005では、`assetAccountId`だけを条件として利用可能資産設定履歴を取得してはならない。

まず、

```text
assetAccountId
+
操作対象利用者ID
+
利用中状態
```

によって対象資産口座を確定する。

その資産口座IDを使用して、

```text
asset_account_available_settings.asset_account_id
```

を検索する。

概念的には、

```text
X-User-Id
    ↓
UserContext
    ↓
asset_accounts
    ↓
利用者境界確認
    ↓
asset_account_available_settings
```

とする。

---

### 5.2 他利用者の資産口座

指定された`assetAccountId`が他利用者に帰属する場合は、対象資産口座が存在しないものとして扱う。

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

ACC-005では、利用中の資産口座を履歴取得対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は通常取得対象に含めない。

論理削除済み資産口座が指定された場合も、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 5.4 利用可能資産設定にはuser_idを持たせない

```text
asset_account_available_settings
```

には、直接

```text
user_id
```

を保持しない。

利用者境界は、関連する

```text
asset_accounts.user_id
```

を通して保証する。

概念的には、

```text
users
    ↓
asset_accounts.user_id
    ↓
asset_accounts.id
    ↓
asset_account_available_settings.asset_account_id
```

という関係になる。

---

### 5.5 利用可能資産設定IDだけで取得しない

ACC-005は資産口座単位の履歴一覧取得APIである。

そのため、

```text
asset_account_available_settings.id
```

だけを使用して設定を取得する処理は行わない。

必ず、利用者境界確認済みの

```text
assetAccountId
```

を起点として履歴一覧を取得する。

---

### 5.6 他利用者の設定履歴を取得しない

例えば、

```text
User A
    Asset Account A
        Setting A1
        Setting A2

User B
    Asset Account B
        Setting B1
```

という状態で、User AとしてAsset Account Bを指定した場合は、

```text
Setting B1
```

を取得してはならない。

資産口座段階で

```text
ASSET_ACCOUNT_NOT_FOUND
```

として処理を終了する。

---

### 5.7 利用可能資産設定が0件の場合

ACC-002では、資産口座登録時に初期利用可能資産設定を必ず1件登録する。

そのため、通常状態では利用中資産口座に対して設定履歴が1件以上存在することを前提とする。

もし、

```text
asset_account_available_settings
    0件
```

である場合は、単なる空配列として正常扱いせず、業務データ不整合として扱う。

具体的なエラーコードは、後続の「エラーレスポンス」で定義する。

---

### 5.8 履歴重複などの不整合

ACC-005は設定履歴そのものを取得するAPIであるため、期間の重複などが存在する場合も履歴を確認できる。

ただし、明らかな業務データ不整合を正常な履歴として黙って扱うかどうかは、後続の「業務ルール」で定義する。

ACC-005自身が設定履歴を自動修復することはない。

---

### 5.9 X-User-Idが不正な場合

以下の場合は、資産口座検索や利用可能資産設定履歴取得へ進まない。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正
- 指定された利用者が存在しない
- 指定された利用者が論理削除済み

利用者コンテキストに関するエラーコードおよびHTTPステータスは、API共通方針に従う。

---

### 5.10 1リクエスト1利用者

1回のACC-005リクエストでは、`X-User-Id`で指定された1利用者の資産口座だけを扱う。

以下のように、URLやクエリパラメータから別利用者IDを指定する方式は採用しない。

```text
/api/v1/users/{userId}/asset-accounts/{assetAccountId}/available-settings
```

または、

```text
?userId=1
```

利用者の指定経路は、`X-User-Id`へ統一する。

---

## 6. パスパラメータ

本APIでは、利用可能資産設定履歴を取得する資産口座を指定するために、以下のパスパラメータを使用する。

| パラメータ | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `assetAccountId` | string | ○ | 利用可能資産設定履歴を取得する資産口座ID |

エンドポイントは、以下とする。

```http
GET /api/v1/asset-accounts/{assetAccountId}/available-settings
```

---

### 6.1 assetAccountId

`assetAccountId`には、利用可能資産設定履歴を取得する資産口座IDを指定する。

例：

```http
GET /api/v1/asset-accounts/10/available-settings
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

この条件で取得できた資産口座に対してのみ、利用可能資産設定履歴を取得する。

---

## 7. クエリパラメータ

なし。

ACC-005では、指定された資産口座に紐づく利用可能資産設定履歴をすべて取得する。

そのため、Phase1では絞り込みやページング用のクエリパラメータを使用しない。

以下のようなクエリパラメータは受け付けない。

- `userId`
- `targetYearMonth`
- `isAvailable`
- `includeDeleted`
- `currentOnly`
- `fromYearMonth`
- `toYearMonth`

---

### 7.1 currentOnlyを設けない

現在有効な設定だけを取得する用途は、ACC-003 資産口座詳細取得で扱う。

ACC-005は、利用可能資産設定の履歴一覧取得に責務を限定する。

そのため、

```text
?currentOnly=true
```

のような取得モード切替は設けない。

---

### 7.2 targetYearMonthを設けない

特定年月時点の利用可能資産区分を1件だけ取得するAPIとはしない。

ACC-005では、履歴全体を返却し、必要な表示はフロントエンド側で行う。

---

## 8. リクエストヘッダー

以下のリクエストヘッダーを使用する。

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
GET /api/v1/asset-accounts/10/available-settings
Accept: application/json
X-User-Id: 1
```

本APIではリクエストボディを使用しないため、

```text
Content-Type: application/json
```

は必須としない。

---

### 8.1 X-User-Id

`X-User-Id`は、API共通方針に従って検証する。

本APIの処理開始前に、共通Middlewareで以下を確認する。

- ヘッダーが指定されていること
- IDが共通方針で定めた形式であること
- 利用者が存在すること
- 利用者が論理削除されていないこと

検証済み利用者IDを利用者コンテキストへ保持し、Action以降で使用する。

---

## 9. リクエストボディ

なし。

ACC-005は参照専用のGET APIであるため、リクエストボディを使用しない。

以下のような情報をリクエストボディから受け付けない。

- `userId`
- `assetAccountId`
- `targetYearMonth`
- `isAvailable`
- `startYearMonth`
- `endYearMonth`

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

ACC-005では、主に以下を検証する。

```text
X-User-Id
assetAccountId
```

資産口座の存在確認や利用者境界確認、利用可能資産設定履歴の存在確認については、DB取得処理の中で判定する。

---

### 11.1 X-User-Id

`X-User-Id`について、以下を検証する。

- 必須であること
- 共通ID形式に一致すること
- 正の整数として扱えること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

不正な場合は、資産口座検索へ進まない。

---

### 11.2 assetAccountId 必須

`assetAccountId`は、パスパラメータとして必須とする。

ACC-005では、以下の形式を使用する。

```http
GET /api/v1/asset-accounts/{assetAccountId}/available-settings
```

`assetAccountId`を省略したURLはACC-005として成立しない。

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
10abc
```

形式不正の場合は、資産口座検索へ進まない。

---

### 11.4 assetAccountIdの型変換

Laravel内部では、ルートパラメータを文字列として受け取る場合がある。

形式検証後に、必要に応じて整数へ変換して使用する。

PHPの暗黙的な型変換によって、不正な値を有効なIDとして扱ってはならない。

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

指定された`assetAccountId`について、操作対象利用者に属する利用中の資産口座が存在することを確認する。

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

以下のような専用エラーは返却しない。

```text
FORBIDDEN
OTHER_USER_ASSET_ACCOUNT
```

他利用者の資産口座存在有無を外部へ公開しないためである。

---

### 11.7 論理削除済み資産口座

対象資産口座が

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の場合も、ACC-005の通常取得対象には含めない。

この場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 11.8 利用可能資産設定履歴の存在確認

対象資産口座に紐づく

```text
asset_account_available_settings
```

が1件以上存在することを確認する。

概念的には、

```text
asset_account_available_settings.asset_account_id
    = assetAccountId
```

に該当する履歴を取得する。

ACC-002では初期利用可能資産設定を必ず登録するため、通常は1件以上存在することを前提とする。

---

### 11.9 履歴0件

対象資産口座に対する利用可能資産設定履歴が0件の場合は、正常な空一覧として扱わない。

以下のように、

```json
{
  "data": []
}
```

を返却しない。

ACC-002で初期設定を必ず作成する設計に対するデータ不整合として扱う。

具体的なエラーコードは、後続の「エラーレスポンス」で定義する。

---

### 11.10 start_year_month

取得した履歴の

```text
start_year_month
```

は、各設定の適用開始年月を表す。

ACC-005は参照APIであるため、取得時に値そのものを変更しない。

ただし、DB上で想定形式から外れた値が存在する場合は、業務データ不整合として扱う設計を検討してよい。

通常は、DB制約およびACC-002・ACC-006の登録時バリデーションによって整合性を保証する。

---

### 11.11 end_year_month

```text
end_year_month
```

は、設定の適用終了年月を表す。

現在継続中の設定では、

```text
NULL
```

を許可する。

過去設定では、年月値が設定される。

---

### 11.12 start_year_monthとend_year_monthの大小関係

`end_year_month`が`NULL`ではない場合は、

```text
start_year_month
    <= end_year_month
```

となっていることを正常な履歴状態とする。

例えば、

```text
start_year_month = 2026-08
end_year_month   = 2026-07
```

のような状態は、正常な設定期間ではない。

ACC-005がこの不整合を検出するかどうかは、後続の「業務ルール」で定義する。

---

### 11.13 履歴期間の重複

同一資産口座について、複数の設定期間が重複する状態は、本来の業務ルール上発生しないことを前提とする。

例えば、

```text
設定A
2026-01 ～ 2026-08

設定B
2026-08 ～ NULL
```

のように、同じ年月が複数設定へ含まれる場合は、期間定義によっては重複となる。

ACC-005では、履歴一覧取得時に期間整合性をどこまで検証するかを後続の「業務ルール」で定義する。

---

### 11.14 現在設定が複数件存在する場合

例えば、

```text
設定A
2026-01 ～ NULL

設定B
2026-07 ～ NULL
```

のように、終了年月未設定の履歴が複数存在する状態は正常状態ではない。

ACC-005は履歴一覧取得APIであるため、単純に最新1件だけへ補正しない。

具体的な扱いは、後続の「業務ルール」および「エラーレスポンス」で定義する。

---

### 11.15 is_available

各履歴の

```text
is_available
```

は、booleanとして扱う。

APIでは、

```text
true
false
```

として返却する。

DB内部表現をそのままクライアントへ公開しない。

---

### 11.16 クエリパラメータによる絞り込みを行わない

ACC-005では、以下のような入力値によって履歴を絞り込まない。

```text
targetYearMonth
fromYearMonth
toYearMonth
isAvailable
```

Phase1では、対象資産口座の利用可能資産設定履歴全体を取得する。

---

### 11.17 バリデーション失敗時

以下のいずれかに該当する場合は、正常レスポンスを返却しない。

- `X-User-Id`不正
- `assetAccountId`形式不正
- 利用者不存在
- 資産口座不存在
- 他利用者の資産口座
- 論理削除済み資産口座
- 利用可能資産設定履歴0件
- 後続で定義する業務データ不整合

ACC-005は参照専用APIであるため、異常時にも業務データは更新しない。

---

## 12. 業務ルール

### 12.1 取得対象

ACC-005では、操作対象利用者に帰属する指定された資産口座の利用可能資産設定履歴を取得する。

取得対象は、

```text
asset_account_available_settings
```

に登録されている指定資産口座の全履歴とする。

概念的には、

```text
X-User-Id
    ↓
操作対象利用者
    ↓
assetAccountId
    ↓
asset_accounts
    ↓
利用者境界確認
    ↓
asset_account_available_settings
    ↓
履歴一覧
```

とする。

---

### 12.2 利用中の資産口座のみ対象とする

ACC-005では、

```text
asset_accounts.deleted_at
    IS NULL
```

の資産口座のみを取得対象とする。

論理削除済み資産口座については、利用可能資産設定履歴がDB上に残っていてもACC-005から取得しない。

---

### 12.3 利用者境界

指定された資産口座が操作対象利用者に帰属することを必須とする。

概念的な条件は、以下とする。

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

条件を満たさない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

---

### 12.4 他利用者の履歴を返却しない

利用可能資産設定自体には`user_id`を保持しないため、利用者境界は親となる資産口座を通して保証する。

以下のように、

```text
asset_account_available_settings.asset_account_id
    = assetAccountId
```

だけで履歴を取得してはならない。

必ず、

```text
userId
    ↓
asset_accounts
    ↓
assetAccountId
    ↓
asset_account_available_settings
```

の順序で取得対象を確定する。

---

### 12.5 履歴は全件取得する

ACC-005では、指定された資産口座に紐づく利用可能資産設定履歴を全件取得する。

現在有効な設定だけに限定しない。

例えば、

```text
2026-01 ～ 2026-06
isAvailable = true

2026-07 ～ 2026-12
isAvailable = false

2027-01 ～ NULL
isAvailable = true
```

という履歴が存在する場合は、3件すべてを返却する。

---

### 12.6 現在設定だけを抽出しない

ACC-005は履歴取得APIであるため、

```text
currentYearMonth
```

を基準として現在設定1件だけを抽出する処理は行わない。

現在の`isAvailable`を取得する責務は、ACC-003 資産口座詳細取得が持つ。

---

### 12.7 特定年月だけを抽出しない

ACC-005では、

```text
targetYearMonth
```

を指定して特定年月時点の設定だけを取得する機能を持たない。

Phase1では、資産口座単位の履歴全体を返却する。

---

### 12.8 履歴の並び順

利用可能資産設定履歴は、適用開始年月の降順で返却する。

概念的には、

```text
start_year_month DESC
```

とする。

例えば、

```text
2027-01 ～ NULL
2026-07 ～ 2026-12
2026-01 ～ 2026-06
```

の順で返却する。

これにより、現在または最新の設定を先頭で確認できる。

---

### 12.9 同一開始年月は存在しない前提とする

同一資産口座に対して、

```text
asset_account_id
+
start_year_month
```

は一意であることを前提とする。

そのため、

```text
2026-07 ～ 2026-12
2026-07 ～ NULL
```

のように同一開始年月を持つ設定が複数存在しないようにする。

DB制約については、テーブル定義書の一意性制約に従う。

---

### 12.10 初期設定が必ず存在する

ACC-002 資産口座登録では、資産口座登録と同時に初期利用可能資産設定を1件登録する。

そのため、利用中の資産口座については、

```text
asset_account_available_settings
    >= 1件
```

を正常状態とする。

---

### 12.11 履歴0件はデータ不整合とする

利用中の資産口座に対して利用可能資産設定履歴が0件の場合は、正常な空一覧とはしない。

概念的には、

```text
asset_accounts
    存在
    ↓
asset_account_available_settings
    0件
    ↓
データ不整合
```

とする。

以下のような正常レスポンスにはしない。

```json
{
  "data": []
}
```

ACC-002の登録ルールが満たされていない状態としてエラーを返却する。

---

### 12.12 設定期間は年月単位で管理する

利用可能資産設定の適用期間は、日付単位ではなく年月単位で管理する。

使用する値は、

```text
YYYY-MM
```

形式とする。

例えば、

```text
2026-08
2027-01
```

のように表現する。

---

### 12.13 startYearMonth

`startYearMonth`は、その利用可能資産設定が適用開始される年月を表す。

例えば、

```text
startYearMonth = 2026-08
```

の場合は、2026年8月からその設定が有効であることを表す。

---

### 12.14 endYearMonth

`endYearMonth`は、その利用可能資産設定が適用される最後の年月を表す。

例えば、

```text
startYearMonth = 2026-01
endYearMonth   = 2026-06
```

の場合は、

```text
2026-01
～
2026-06
```

までその設定が有効である。

---

### 12.15 endYearMonthがNULLの場合

`endYearMonth = null`は、終了年月が設定されていない継続中の設定を表す。

例えば、

```text
startYearMonth = 2026-08
endYearMonth   = null
```

は、

```text
2026-08 ～
```

という設定期間を表す。

APIでは、`null`をそのまま返却する。

---

### 12.16 isAvailable

`isAvailable`は、その期間において対象資産口座を利用可能資産として扱うかを表す。

```text
true
    → 利用可能資産として扱う

false
    → 利用可能資産として扱わない
```

APIではbooleanとして返却する。

---

### 12.17 履歴期間は重複しない

同一資産口座について、複数の利用可能資産設定が同じ年月に適用される状態は許可しない。

正常な例：

```text
設定A
2026-01 ～ 2026-06

設定B
2026-07 ～ 2026-12

設定C
2027-01 ～ NULL
```

不正な例：

```text
設定A
2026-01 ～ 2026-08

設定B
2026-08 ～ NULL
```

この場合、2026年8月に2件の設定が適用されるため期間重複となる。

---

### 12.18 期間の連続性を前提とする

利用可能資産設定は、ACC-002による初期登録とACC-006による設定変更によって管理する。

ACC-006で新しい設定を登録する際は、直前の設定を終了させたうえで新しい設定を開始する。

概念的には、

```text
旧設定
2026-01 ～ NULL
    ↓
ACC-006
    ↓
旧設定
2026-01 ～ 2026-07

新設定
2026-08 ～ NULL
```

とする。

そのため、正常な履歴では設定期間が連続していることを前提とする。

---

### 12.19 期間の欠落を正常状態としない

例えば、

```text
設定A
2026-01 ～ 2026-05

設定B
2026-07 ～ NULL
```

のように、

```text
2026-06
```

の設定が存在しない状態は、正常な履歴状態とはしない。

利用可能資産設定は、資産口座の利用開始以降、各年月について一意に判定できることを前提とする。

---

### 12.20 資産口座利用開始年月との整合性

最初の利用可能資産設定は、資産口座の

```text
asset_accounts.start_year_month
```

から開始することを前提とする。

概念的には、

```text
asset_accounts.start_year_month
    =
最初の
asset_account_available_settings.start_year_month
```

とする。

ACC-002で資産口座登録時にこの状態を作成する。

---

### 12.21 利用開始年月より前の設定を許可しない

例えば、

```text
asset_accounts.start_year_month
    = 2026-08

asset_account_available_settings.start_year_month
    = 2026-07
```

のように、資産口座利用開始年月より前から利用可能資産設定が開始されている状態は正常状態としない。

---

### 12.22 最終履歴は終了年月NULLを前提とする

現在利用中の資産口座では、最新の利用可能資産設定は

```text
end_year_month = NULL
```

であることを前提とする。

概念的には、

```text
過去設定
    end_year_month
        = YYYY-MM

最新設定
    end_year_month
        = NULL
```

とする。

---

### 12.23 終了年月NULLの履歴は1件のみ

同一資産口座について、

```text
end_year_month = NULL
```

の履歴は1件だけ存在することを前提とする。

例えば、

```text
設定A
2026-01 ～ NULL

設定B
2026-08 ～ NULL
```

のような状態は正常状態ではない。

---

### 12.24 履歴整合性の検証

ACC-005は、利用可能資産設定履歴を利用者へ表示するAPIである。

そのため、取得した履歴について最低限以下の整合性を確認する。

- 履歴が1件以上存在する
- 最初の開始年月が資産口座利用開始年月と一致する
- `startYearMonth <= endYearMonth`
- 設定期間が重複していない
- 設定期間に欠落がない
- 最新設定の`endYearMonth`が`null`
- `endYearMonth = null`の設定が1件だけ存在する

---

### 12.25 不整合履歴を正常レスポンスとして返さない

履歴整合性に違反している場合は、取得できたレコードをそのまま正常レスポンスとして返却しない。

例えば、

```text
設定A
2026-01 ～ NULL

設定B
2026-08 ～ NULL
```

という状態で、

```json
{
  "data": [
    {
      "startYearMonth": "2026-08",
      "endYearMonth": null,
      "isAvailable": false
    },
    {
      "startYearMonth": "2026-01",
      "endYearMonth": null,
      "isAvailable": true
    }
  ]
}
```

のように正常レスポンスを返却しない。

データ不整合として扱う。

---

### 12.26 不整合を自動修復しない

ACC-005は参照専用APIであるため、履歴不整合を検出してもDBを自動更新しない。

例えば、

```text
最新1件以外の
end_year_monthを自動設定する

欠落期間を
前後の設定で自動補完する

重複設定を削除する
```

などの処理は行わない。

概念的には、

```text
不整合検出
    ↓
エラー
    ↓
ログ記録
```

までとする。

---

### 12.27 ACC-006で履歴整合性を維持する

新しい利用可能資産設定を登録する際の履歴変更責務は、

```text
ACC-006
利用可能資産設定登録
```

が持つ。

ACC-005は、ACC-006によって構築された履歴を参照する。

概念的には、

```text
ACC-002
    ↓
初期設定作成

ACC-006
    ↓
設定履歴追加・期間更新

ACC-005
    ↓
設定履歴参照
```

とする。

---

### 12.28 ACC-003との現在設定判定の整合性

ACC-003では、現在年月に該当する利用可能資産設定を取得して

```text
isAvailable
```

を返却する。

ACC-005の履歴ルールも同じ期間定義を使用する。

つまり、

```text
start_year_month
    <= targetYearMonth

AND

(
    end_year_month IS NULL
    OR
    end_year_month >= targetYearMonth
)
```

を対象年月における設定有効条件とする。

---

### 12.29 利用可能資産区分の意味を変更しない

ACC-005では、DBに保存されている

```text
is_available
```

を別の業務条件によって再計算しない。

例えば、

```text
資産種別
残高
保有商品
目的
```

などから利用可能資産区分を推測しない。

利用可能資産区分は、登録された設定値をそのまま業務状態として扱う。

---

### 12.30 資産口座属性を履歴へ複製しない

ACC-005では、利用可能資産設定履歴として必要な情報だけを返却する。

資産口座の

```text
name
assetType
balanceRecordingUnit
startYearMonth
isEnabled
```

などを各履歴レコードへ重複して含めない。

資産口座自体の情報は、ACC-003で取得する。

---

### 12.31 履歴IDを返却する

各利用可能資産設定を識別できるように、

```text
asset_account_available_settings.id
```

をAPIでは

```text
id
```

として返却する。

ただし、このIDを使用してACC-005から個別設定を更新・削除することはない。

---

### 12.32 履歴を削除しない

ACC-005はGET APIであり、利用可能資産設定履歴を削除しない。

また、Phase1では過去の利用可能資産設定を物理削除するAPIも提供しない。

履歴は、過去時点の利用可能資産状態を再現するために保持する。

---

### 12.33 ページングを行わない

利用可能資産設定は、資産口座単位かつ年月単位で変更される履歴であり、Phase1では大量件数になることを想定しない。

そのため、ACC-005ではページングを行わず全件取得する。

---

## 13. 処理フロー

ACC-005の基本処理フローは、以下とする。

```text
Request
    ↓
X-User-Id検証
    ↓
UserContext取得
    ↓
assetAccountId形式検証
    ↓
資産口座取得
    ↓
利用者境界確認
    ↓
利用可能資産設定履歴取得
    ↓
履歴0件確認
    ↓
履歴整合性確認
    ↓
startYearMonth降順
    ↓
Response DTO生成
    ↓
API Resource
    ↓
200 OK
```

---

### 13.1 資産口座取得

操作対象利用者IDと`assetAccountId`を使用して、対象資産口座を取得する。

概念的には、

```text
asset_accounts.id
    = assetAccountId

AND

asset_accounts.user_id
    = userId

AND

asset_accounts.deleted_at
    IS NULL
```

とする。

取得できない場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として処理を終了する。

---

### 13.2 履歴取得

資産口座確認後、以下の条件で利用可能資産設定履歴を取得する。

```text
asset_account_available_settings.asset_account_id
    = assetAccountId
```

並び順は、

```text
start_year_month DESC
```

とする。

---

### 13.3 履歴0件判定

取得結果が0件の場合は、正常な空一覧として返却しない。

概念的には、

```text
履歴件数
    ↓
0件
    ↓
データ不整合エラー
```

とする。

---

### 13.4 履歴整合性確認

取得した履歴について、以下を確認する。

```text
最初の設定開始年月
    =
asset_accounts.start_year_month

各履歴
    startYearMonth <= endYearMonth

期間重複なし

期間欠落なし

endYearMonth = null
    1件のみ

最新設定
    endYearMonth = null
```

不整合がある場合は、正常レスポンスを生成しない。

---

### 13.5 レスポンス生成

正常な履歴を、APIレスポンス用DTOへ変換する。

概念的には、

```text
AssetAccountAvailableSetting
    ↓
AvailableSettingHistory DTO
    ↓
API Resource
    ↓
camelCase
```

とする。

DBカラム名を直接レスポンスへ使用しない。

---

## 14. レスポンス

### 14.1 成功レスポンス

HTTPステータス：

```http
200 OK
```

レスポンス例：

```json
{
  "data": [
    {
      "id": "3",
      "startYearMonth": "2027-01",
      "endYearMonth": null,
      "isAvailable": true
    },
    {
      "id": "2",
      "startYearMonth": "2026-07",
      "endYearMonth": "2026-12",
      "isAvailable": false
    },
    {
      "id": "1",
      "startYearMonth": "2026-01",
      "endYearMonth": "2026-06",
      "isAvailable": true
    }
  ]
}
```

API共通Envelopeに`meta`等を含める場合は、API共通方針に従う。

---

### 14.2 data

`data`には、指定資産口座の利用可能資産設定履歴を配列として返却する。

並び順は、

```text
startYearMonth DESC
```

とする。

正常時には1件以上の要素が存在する。

---

## 15. レスポンス項目

各履歴のレスポンス項目は、以下とする。

| 項目名 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `id` | string | × | 利用可能資産設定ID |
| `startYearMonth` | string | × | 設定の適用開始年月。`YYYY-MM`形式 |
| `endYearMonth` | string | ○ | 設定の適用終了年月。`YYYY-MM`形式。継続中の場合は`null` |
| `isAvailable` | boolean | × | 利用可能資産として扱うか |

---

### 15.1 id

`id`には、

```text
asset_account_available_settings.id
```

を文字列へ変換して返却する。

例：

```json
{
  "id": "3"
}
```

DBの主キー型が`bigint`であっても、APIではIDを文字列として扱う。

---

### 15.2 startYearMonth

`startYearMonth`には、

```text
asset_account_available_settings.start_year_month
```

を返却する。

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

### 15.3 endYearMonth

`endYearMonth`には、

```text
asset_account_available_settings.end_year_month
```

を返却する。

終了年月が存在する場合は、

```json
{
  "endYearMonth": "2026-12"
}
```

とする。

継続中の場合は、

```json
{
  "endYearMonth": null
}
```

とする。

空文字には変換しない。

---

### 15.4 isAvailable

`isAvailable`には、

```text
asset_account_available_settings.is_available
```

をbooleanへ変換して返却する。

例：

```json
{
  "isAvailable": true
}
```

または、

```json
{
  "isAvailable": false
}
```

DB内部値を文字列や数値としてそのまま返却しない。

---

### 15.5 レスポンスへ含めない項目

ACC-005では、以下の情報を返却しない。

```text
asset_account_id
created_at
updated_at
```

また、資産口座側の以下の情報も履歴ごとには返却しない。

```text
user_id
name
asset_type
balance_recording_unit
start_year_month
deleted_at
```

資産口座自体の詳細は、ACC-003の責務とする。

---

### 15.6 現在設定フラグを追加しない

Phase1では、

```text
isCurrent
```

のような派生項目を返却しない。

最新設定は、

```text
endYearMonth = null
```

によって判別できる。

APIレスポンス項目を必要以上に増やさない。

---

### 15.7 表示用文言を返却しない

以下のような画面表示専用文言はAPIから返却しない。

```text
利用可能
利用対象外
現在
2026年8月
```

React側で、

```text
isAvailable
startYearMonth
endYearMonth
```

を使用して表示形式へ変換する。

---

## 16. トランザクション境界

ACC-005は参照専用APIであり、業務データを更新しない。

そのため、Phase1では明示的なトランザクションを使用しない。

概念的には、

```text
AssetAccount SELECT
    ↓
AvailableSettings SELECT
    ↓
Response
```

のみで完結する。

---

### 16.1 DB::transactionを使用しない

以下のような

```php
DB::transaction(function () {
    // 資産口座取得
    // 履歴取得
});
```

処理は、Phase1では不要とする。

ACC-005では、

```text
INSERT
UPDATE
DELETE
```

を行わないためである。

---

### 16.2 読み取り整合性

ACC-005実行中にACC-006による利用可能資産設定変更が同時実行された場合、DBのトランザクション分離レベルや各SELECTの実行タイミングによっては、取得状態に差が生じる可能性がある。

Phase1では、履歴参照に対して厳密なスナップショット読み取りを必須としない。

---

### 16.3 不整合検出時も更新しない

履歴取得中にデータ不整合を検出しても、ACC-005内で修復用トランザクションを開始しない。

概念的には、

```text
不整合検出
    ↓
エラー返却
    ↓
内部ログ記録
```

とする。

データ修復は、ACC-005とは別の運用・保守責務として扱う。

---

## 17. 排他制御

ACC-005は、参照専用APIであり、業務データを更新しない。

そのため、Phase1では明示的な排他制御を行わない。

以下のような行ロックは使用しない。

```php
lockForUpdate()
```

また、楽観ロック用の

```text
version
updated_at比較
ETag
If-Match
```

なども使用しない。

---

### 17.1 資産口座へのロック

ACC-005では、`asset_accounts`を利用者境界確認のために参照するだけである。

そのため、対象資産口座行をロックしない。

概念的には、

```text
asset_accounts
    SELECT
```

のみとする。

---

### 17.2 利用可能資産設定へのロック

`asset_account_available_settings`についても、履歴一覧を参照するだけであるため、明示的な行ロックを行わない。

概念的には、

```text
asset_account_available_settings
    SELECT
```

のみとする。

---

### 17.3 ACC-006との同時実行

ACC-005実行中に、別リクエストでACC-006 利用可能資産設定登録が実行される可能性がある。

例えば、

```text
ACC-005
履歴取得開始
    ↓
ACC-006
旧設定終了
+
新設定登録
    ↓
ACC-005
履歴取得完了
```

のような並行実行が発生し得る。

Phase1では、履歴表示用の参照APIとして扱うため、ACC-006の更新処理をブロックしない。

---

### 17.4 厳密な同一時点保証を行わない

ACC-005では、

```text
asset_accounts取得時点
```

と

```text
asset_account_available_settings取得時点
```

を厳密に同一瞬間へ固定することを要件としない。

本APIは履歴の参照・表示を目的とし、月末確定や目的達成判定のような業務確定処理ではないためである。

---

### 17.5 不整合検出時にロックしない

履歴整合性チェックで、

- 履歴0件
- 期間重複
- 期間欠落
- 複数の継続中設定
- 開始年月不整合

などを検出した場合も、ACC-005でロックを取得して修復処理を行わない。

概念的には、

```text
不整合検出
    ↓
エラー返却
    ↓
ログ記録
```

までとする。

---

### 17.6 将来的な整合性要件

将来的に、履歴一覧取得について厳密な同一時点スナップショットが必要になった場合は、

```text
Repeatable Read
明示的トランザクション
```

などを検討する。

Phase1では対象外とする。

---

## 18. エラーレスポンス

異常時は、API共通方針に従った共通エラーレスポンス形式を使用する。

概念例：

```json
{
  "error": {
    "code": "ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID",
    "message": "利用可能資産設定履歴の整合性が不正です。"
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

---

### 18.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

この場合、資産口座検索や履歴取得へ進まない。

---

### 18.2 INVALID_USER_ID

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

### 18.3 USER_NOT_FOUND

指定された利用者が存在しない場合、または論理削除済みの場合は、

```text
USER_NOT_FOUND
```

を返却する。

---

### 18.4 INVALID_ASSET_ACCOUNT_ID

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
10abc
```

などを不正とする。

形式不正の場合は、資産口座検索へ進まない。

---

### 18.5 ASSET_ACCOUNT_NOT_FOUND

以下の場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

- 指定された資産口座が存在しない
- 指定された資産口座が他利用者に属する
- 指定された資産口座が論理削除済み

他利用者所属や論理削除済みであることを個別のエラーコードで公開しない。

---

### 18.6 他利用者の資産口座

例えば、

```text
User A
    assetAccountId = 10

User B
    X-User-Id = 2
```

の状態で、User Bが

```http
GET /api/v1/asset-accounts/10/available-settings
```

を実行した場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

User Aの利用可能資産設定履歴を返却しない。

---

### 18.7 論理削除済み資産口座

対象資産口座が

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の場合も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

を返却する。

論理削除済みであることを専用エラーとして公開しない。

---

### 18.8 履歴0件

対象資産口座に対する

```text
asset_account_available_settings
```

が0件の場合は、正常な空配列として扱わない。

概念的なエラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

とする。

ACC-002で初期設定を必ず登録する設計に対するデータ不整合として扱う。

---

### 18.9 初期設定開始年月不整合

最初の利用可能資産設定の

```text
start_year_month
```

が、資産口座の

```text
asset_accounts.start_year_month
```

と一致しない場合は、履歴不整合として扱う。

概念的には、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

を返却する。

---

### 18.10 startYearMonthとendYearMonthの不整合

以下のように、

```text
start_year_month
    >
end_year_month
```

となっている履歴が存在する場合は、正常レスポンスを返却しない。

例えば、

```text
start_year_month = 2026-08
end_year_month   = 2026-07
```

は不正とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

とする。

---

### 18.11 履歴期間重複

同一資産口座について、設定期間が重複している場合は、履歴不整合として扱う。

例えば、

```text
設定A
2026-01 ～ 2026-08

設定B
2026-08 ～ NULL
```

のように、2026-08が複数設定へ含まれる場合は不正とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

とする。

---

### 18.12 履歴期間欠落

設定期間に年月の欠落が存在する場合も、履歴不整合として扱う。

例えば、

```text
設定A
2026-01 ～ 2026-05

設定B
2026-07 ～ NULL
```

では、

```text
2026-06
```

がどの設定にも属さない。

この場合も、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

とする。

---

### 18.13 継続中設定が複数存在する場合

以下のように、

```text
end_year_month = NULL
```

の設定が複数存在する場合は、正常状態としない。

例：

```text
設定A
2026-01 ～ NULL

設定B
2026-08 ～ NULL
```

この場合も、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

を返却する。

---

### 18.14 最新設定に終了年月が存在する場合

利用中の資産口座では、最新の設定は

```text
end_year_month = NULL
```

であることを前提とする。

例えば、

```text
設定A
2026-01 ～ 2026-06

設定B
2026-07 ～ 2026-12
```

だけが存在し、最新設定にも終了年月が設定されている場合は、現在以降の設定が存在しない。

この場合も、履歴不整合として扱う。

---

### 18.15 同一開始年月の重複

同一資産口座で、

```text
asset_account_id
+
start_year_month
```

が重複している場合も、正常な履歴状態としない。

例えば、

```text
設定A
start_year_month = 2026-08

設定B
start_year_month = 2026-08
```

は不正とする。

DBのUNIQUE制約によって通常は防止するが、取得時に検出された場合は履歴不整合として扱う。

---

### 18.16 不整合を補完しない

履歴不整合時に、以下のような自動補完を行わない。

```text
期間欠落
    ↓
前の設定を延長
```

```text
複数継続設定
    ↓
最新1件を採用
```

```text
最新設定に終了年月あり
    ↓
endYearMonthをNULL扱い
```

データ不整合を正常レスポンスで隠蔽しない。

---

### 18.17 不整合時に一部履歴を返さない

履歴の一部だけが正常であっても、不整合を含む状態で

```text
正常な履歴だけ返却
```

とはしない。

ACC-005は、資産口座の利用可能資産設定履歴全体を1つの整合した履歴として扱う。

不整合がある場合は、正常レスポンスを返却しない。

---

### 18.18 INTERNAL_SERVER_ERROR

履歴取得処理で想定外の例外が発生した場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスへ、以下の内部情報を含めない。

- SQL
- PostgreSQL内部エラー
- テーブル名
- カラム名
- 制約名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

### 18.19 主なエラー一覧

ACC-005で想定する主なエラーは、以下とする。

| HTTPステータス | 独自エラーコード | 発生条件 |
|---|---|---|
| `400 Bad Request` | `USER_CONTEXT_REQUIRED` | `X-User-Id`未指定 |
| `400 Bad Request` | `INVALID_USER_ID` | `X-User-Id`形式不正 |
| `400 Bad Request` | `INVALID_ASSET_ACCOUNT_ID` | `assetAccountId`形式不正 |
| `404 Not Found` | `USER_NOT_FOUND` | 利用者不存在、または論理削除済み |
| `404 Not Found` | `ASSET_ACCOUNT_NOT_FOUND` | 資産口座不存在、他利用者所属、または論理削除済み |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND` | 利用可能資産設定履歴が0件 |
| `500 Internal Server Error` | `ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID` | 期間重複、期間欠落、複数継続設定などの履歴不整合 |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` | 想定外のサーバー内部エラー |

履歴0件や履歴整合性不正は、クライアント入力によるものではなく、サーバー側の業務データ不整合として扱う。

具体的なデータ不整合エラーコードをAPI全体で共通化する場合は、API共通方針を優先する。

---

### 18.20 エラー時の副作用

ACC-005は参照専用APIであるため、いずれのエラーが発生した場合も業務データを更新しない。

特に、履歴不整合を検出しても、

```text
asset_account_available_settings
```

への

```text
INSERT
UPDATE
DELETE
```

を行わない。

---

## 19. HTTPステータス

ACC-005では、処理結果に応じて以下のHTTPステータスを返却する。

| HTTPステータス | 用途 |
|---|---|
| `200 OK` | 利用可能資産設定履歴取得成功 |
| `400 Bad Request` | 利用者コンテキストまたは`assetAccountId`の形式不正 |
| `404 Not Found` | 利用者または資産口座が存在しない |
| `500 Internal Server Error` | 利用可能資産設定履歴の不整合、または想定外のサーバー内部エラー |

具体的な独自エラーコードは、「エラーレスポンス」およびAPI共通方針に従う。

---

### 19.1 200 OK

指定された資産口座が操作対象利用者に帰属し、利用可能資産設定履歴が正常に取得できた場合は、

```http
200 OK
```

を返却する。

概念的には、

```text
資産口座取得成功
    ↓
利用可能資産設定履歴取得
    ↓
履歴1件以上
    ↓
履歴整合性確認
    ↓
200 OK
```

とする。

---

### 19.2 400 Bad Request

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

- `X-User-Id`未指定
- `X-User-Id`形式不正
- `assetAccountId`が正の整数形式ではない

などを対象とする。

---

### 19.3 404 Not Found

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

- 指定された資産口座が存在しない
- 指定された資産口座が他利用者に属する
- 指定された資産口座が論理削除済み

他利用者所属や論理削除済みであることを個別のHTTPステータスで公開しない。

---

### 19.4 500 Internal Server Error

以下の場合は、

```http
500 Internal Server Error
```

として扱う。

主なエラーコードは、以下とする。

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
INTERNAL_SERVER_ERROR
```

利用可能資産設定履歴の不存在や不整合は、クライアント入力によるものではなく、サーバー側の業務データ不整合として扱う。

---

### 19.5 ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND

対象資産口座について、

```text
asset_account_available_settings
```

が1件も存在しない場合は、

```http
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

とする。

ACC-002で初期利用可能資産設定を必ず作成する設計に対するデータ不整合として扱う。

---

### 19.6 ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID

以下のような履歴整合性違反を検出した場合は、

```http
500 Internal Server Error
```

とする。

エラーコードは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

とする。

主な対象は、以下とする。

- 初期設定開始年月が資産口座利用開始年月と一致しない
- `startYearMonth > endYearMonth`
- 設定期間が重複している
- 設定期間に欠落がある
- `endYearMonth = null`の設定が複数存在する
- 最新設定の`endYearMonth`が`null`ではない
- 同一開始年月の設定が複数存在する

---

### 19.7 INTERNAL_SERVER_ERROR

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

## 20. 冪等性

ACC-005は、参照専用のGET APIであるため、冪等である。

同一の

```text
X-User-Id
+
assetAccountId
```

に対して同じサーバー状態で複数回リクエストした場合、同じ履歴一覧を返却する。

---

### 20.1 業務データを変更しない

ACC-005を何度実行しても、以下の業務データを変更しない。

- `users`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

履歴取得によって、

```text
created_at
updated_at
deleted_at
```

なども変更しない。

---

### 20.2 同一リクエストの再実行

例えば、

```http
GET /api/v1/asset-accounts/10/available-settings
Accept: application/json
X-User-Id: 1
```

を、サーバー側の状態が変化していない間に複数回実行した場合は、同じ履歴内容を返却する。

---

### 20.3 並び順も同一とする

ACC-005では、履歴を

```text
startYearMonth DESC
```

で返却する。

そのため、サーバー状態が変化していない場合は、再実行時も同じ並び順となる。

---

### 20.4 サーバー状態が変更された場合

ACC-005自体は冪等であるが、ACC-006によって利用可能資産設定履歴が変更された場合は、後続のACC-005で返却内容が変化する。

例えば、

```text
ACC-005
    ↓
履歴2件

ACC-006
    ↓
旧設定終了
+
新設定登録

ACC-005
    ↓
履歴3件
```

となり得る。

これは、ACC-005の冪等性を損なうものではない。

---

### 20.5 資産口座無効化後

ACC-005取得後に資産口座がACC系の無効化APIによって論理削除された場合は、同じ`assetAccountId`で再度ACC-005を実行すると、

```text
404 Not Found
ASSET_ACCOUNT_NOT_FOUND
```

となる。

これも、サーバー状態が変更された結果である。

---

### 20.6 Idempotency-Key

ACC-005はGETによる参照APIであるため、

```text
Idempotency-Key
```

を使用しない。

冪等性確保のための追加ヘッダーやサーバー側キー管理は不要とする。

---

## 21. キャッシュ

Phase1では、ACC-005専用のサーバー側アプリケーションキャッシュを使用しない。

利用可能資産設定履歴は、PostgreSQLの最新状態から取得する。

---

### 21.1 サーバー側キャッシュ

Phase1では、以下のようなACC-005専用キャッシュを導入しない。

```text
Redis
Laravel Cache
Application Memory Cache
```

利用可能資産設定履歴は資産口座単位で比較的少数件となることを想定し、Phase1ではキャッシュ導入による複雑性を追加しない。

---

### 21.2 ACC-006による履歴変更

ACC-006が成功すると、

```text
asset_account_available_settings
```

の履歴内容が変化する。

そのため、フロントエンドでACC-005の結果をQuery Cacheへ保持している場合は、ACC-006成功後に対象資産口座の履歴Queryを無効化する。

概念的には、

```text
ACC-006成功
    ↓
ACC-005 Query invalidate
    ↓
ACC-005再取得
    ↓
最新履歴表示
```

とする。

---

### 21.3 React・TypeScript側のQuery Cache

フロントエンドでは、TanStack QueryなどのQuery Cacheを使用してよい。

概念的には、以下のようなQuery Keyを使用する。

```text
assetAccounts
+
assetAccountId
+
availableSettings
```

例えば、

```typescript
[
  'assetAccounts',
  assetAccountId,
  'availableSettings',
]
```

のようなキーで履歴取得結果を保持してよい。

具体的な実装は、「React・TypeScriptでの利用」で定義する。

---

### 21.4 Query Keyの共通化

Query Keyは、各コンポーネントへ直接記述せず、共通定義として管理してよい。

概念例：

```typescript
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

  availableSettings: (
    assetAccountId: string,
  ) =>
    [
      'assetAccounts',
      assetAccountId,
      'availableSettings',
    ] as const,
};
```

---

### 21.5 ACC-006成功後のキャッシュ無効化

ACC-006成功後は、ACC-005だけでなくACC-003の

```text
isAvailable
```

も変化する可能性がある。

そのため、少なくとも以下を無効化する。

```text
ACC-005
利用可能資産設定履歴

ACC-003
資産口座詳細
```

概念例：

```typescript
await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.availableSettings(
      assetAccountId,
    ),
});

await queryClient.invalidateQueries({
  queryKey:
    assetAccountKeys.detail(
      assetAccountId,
    ),
});
```

---

### 21.6 資産口座無効化後

対象資産口座が無効化された場合は、ACC-005の通常取得対象から外れる。

そのため、フロントエンドでは対象資産口座の利用可能資産設定履歴Queryを無効化または削除する。

概念例：

```typescript
queryClient.removeQueries({
  queryKey:
    assetAccountKeys.availableSettings(
      assetAccountId,
    ),
});
```

---

### 21.7 ACC-004成功後

ACC-004では、

```text
asset_account_available_settings
```

を更新しない。

そのため、利用可能資産設定履歴そのものはACC-004によって変化しない。

したがって、ACC-004成功だけを理由としてACC-005の履歴Queryを必ず無効化する必要はない。

ただし、画面上で資産口座名などと履歴を一体表示している場合は、画面構成に応じて関連Queryを再取得してよい。

---

### 21.8 ACC-003とのキャッシュ分離

ACC-003とACC-005は、同じ資産口座を対象とするが、取得内容は異なる。

```text
ACC-003
    → 現在の資産口座詳細

ACC-005
    → 利用可能資産設定履歴
```

そのため、Query Cacheも同一キーへまとめず、別キーとして管理する。

---

### 21.9 履歴全体をキャッシュする

ACC-005はページングを行わず、履歴全体を返却する。

そのため、フロントエンドでは取得した配列全体を1つのQuery Cacheとして保持してよい。

個々の履歴レコードごとに別Query Cacheを作成する必要はない。

---

### 21.10 履歴順序をフロントエンドで再構築しない

ACC-005では、サーバー側で

```text
startYearMonth DESC
```

として返却する。

フロントエンドでは、原則としてこの並び順をそのまま使用する。

画面都合で並び替える場合を除き、APIレスポンスを毎回独自ソートして正式な順序を再定義しない。

---

### 21.11 Optimistic Update

ACC-005自体は参照APIであるため、Optimistic Updateの対象ではない。

ACC-006成功時にACC-005のQuery Cacheを直接書き換えることも技術的には可能だが、Phase1では

```text
ACC-006成功
    ↓
ACC-005 invalidate
    ↓
履歴再取得
```

という単純な方式を基本とする。

---

### 21.12 サーバー状態を正とする

利用可能資産設定履歴の最終状態は、PostgreSQLに保存されたデータを正とする。

React側でACC-006のRequest内容だけから履歴全体を再構築しない。

例えば、

```text
旧設定のendYearMonth
新設定のstartYearMonth
新規設定ID
```

などは、サーバー側の確定状態を再取得して表示する。

---

### 21.13 HTTPキャッシュ

ACC-005はGET APIであるため、技術的にはHTTPキャッシュの対象にできる。

ただし、Phase1ではACC-005独自の

```text
ETag
Last-Modified
If-None-Match
If-Modified-Since
```

などは導入しない。

HTTPキャッシュに関する共通方針がある場合は、API共通方針を優先する。

---

### 21.14 利用者境界とキャッシュキー

フロントエンドまたは将来のサーバーキャッシュでACC-005をキャッシュする場合は、`assetAccountId`だけで異なる利用者間のキャッシュを共有しない。

本APIは利用者依存APIであるため、概念的には

```text
userId
+
assetAccountId
+
availableSettings
```

によって利用者境界を維持する必要がある。

---

### 21.15 利用者切替時

利用者切替が行われた場合は、前利用者の利用可能資産設定履歴を新しい利用者へ表示し続けない。

以下のいずれかで対応する。

```text
Query KeyへuserIdを含める

または

利用者切替時に
関連Query Cacheを無効化する
```

API側の`X-User-Id`による利用者境界だけでなく、画面キャッシュ上でも利用者境界を維持する。

---

### 21.16 長時間キャッシュ

利用可能資産設定履歴は、ACC-006が実行されない限り頻繁には変化しない。

ただし、Phase1では長時間キャッシュを前提とした複雑な失効設計は導入しない。

TanStack Queryの`staleTime`などは、フロントエンド共通方針に従う。

---

## 22. 関連テーブル

ACC-005では、指定された資産口座に紐づく利用可能資産設定履歴を取得するため、以下のテーブルを使用する。

| テーブル | 用途 | 更新 |
|---|---|:---:|
| `users` | 操作対象利用者の確認 | × |
| `asset_accounts` | 対象資産口座の取得、利用者境界確認 | × |
| `asset_account_available_settings` | 利用可能資産設定履歴の取得 | × |

ACC-005は参照専用APIであるため、いずれのテーブルも更新しない。

---

### 22.1 users

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

ACC-005では、`users`を更新しない。

---

### 22.2 asset_accounts

指定された`assetAccountId`に対応する資産口座を取得し、利用者境界を確認するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | `assetAccountId`との照合 |
| `user_id` | 操作対象利用者との利用者境界確認 |
| `start_year_month` | 初期利用可能資産設定との整合性確認 |
| `deleted_at` | 利用状態の判定 |

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

### 22.3 他利用者の資産口座

`asset_accounts.id`が一致していても、

```text
asset_accounts.user_id
    != 操作対象利用者ID
```

の場合は、取得対象に含めない。

他利用者の資産口座を一度取得してからアプリケーション側で利用者境界を判定する方式を基本としない。

---

### 22.4 論理削除済み資産口座

ACC-005では、利用中の資産口座だけを履歴取得対象とする。

そのため、

```text
asset_accounts.deleted_at
    IS NOT NULL
```

の資産口座は取得しない。

LaravelのSoftDeletesを使用している場合は、通常スコープを使用し、

```php
withTrashed()
```

を使用しない。

---

### 22.5 asset_account_available_settings

指定された資産口座の利用可能資産設定履歴を取得するために使用する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 利用可能資産設定ID |
| `asset_account_id` | 対象資産口座との関連 |
| `start_year_month` | 設定の適用開始年月 |
| `end_year_month` | 設定の適用終了年月 |
| `is_available` | 利用可能資産区分 |

---

### 22.6 履歴取得条件

対象資産口座に紐づく利用可能資産設定履歴を、以下の条件で取得する。

```text
asset_account_available_settings.asset_account_id
    = assetAccountId
```

現在年月による絞り込みは行わない。

ACC-005では、対象資産口座の履歴全体を取得する。

---

### 22.7 履歴の並び順

取得した履歴は、

```text
start_year_month DESC
```

で並べる。

概念的には、

```text
2027-01 ～ NULL
2026-07 ～ 2026-12
2026-01 ～ 2026-06
```

の順とする。

---

### 22.8 id

```text
asset_account_available_settings.id
```

は、APIではstringへ変換して返却する。

概念的には、

```text
bigint
    ↓
string
```

とする。

---

### 22.9 start_year_month

```text
asset_account_available_settings.start_year_month
```

は、

```text
YYYY-MM
```

形式の

```text
startYearMonth
```

として返却する。

---

### 22.10 end_year_month

```text
asset_account_available_settings.end_year_month
```

は、

```text
YYYY-MM
```

または

```text
NULL
```

として保持する。

APIでは、

```text
endYearMonth
```

として返却する。

継続中設定の場合は、

```json
{
  "endYearMonth": null
}
```

とする。

---

### 22.11 is_available

```text
asset_account_available_settings.is_available
```

は、

```text
isAvailable
```

として返却する。

APIではbooleanとして扱う。

```text
true
false
```

以外の表現へ変換しない。

---

### 22.12 初期設定との整合性

最古の利用可能資産設定の

```text
start_year_month
```

は、

```text
asset_accounts.start_year_month
```

と一致することを正常状態とする。

概念的には、

```text
asset_accounts.start_year_month
    =
MIN(
    asset_account_available_settings.start_year_month
)
```

となる。

---

### 22.13 履歴期間の整合性

同一資産口座に紐づく利用可能資産設定履歴について、以下を正常状態とする。

- 各履歴で`start_year_month <= end_year_month`
- 設定期間が重複しない
- 設定期間に欠落がない
- 最新設定の`end_year_month`が`NULL`
- `end_year_month = NULL`の設定が1件のみ
- 同一`start_year_month`が複数存在しない

---

### 22.14 更新しない関連テーブル

ACC-005では、以下のテーブルも更新しない。

- `holding_assets`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`
- `objectives`
- `assessment_histories`

また、これらのデータをACC-005のレスポンスへ含めない。

---

## 23. 関連する機能要件

ACC-005は、資産口座の利用可能資産設定履歴確認に関する機能要件と対応する。

主な関連要件は、以下とする。

- 資産口座管理
  - 資産口座ごとに利用可能資産区分を管理できる
  - 利用可能資産区分を年月単位の履歴として管理できる
  - 過去の利用可能資産区分を確認できる
- 利用可能資産設定
  - 資産口座登録時に初期設定を作成する
  - 利用可能資産区分の変更履歴を保持する
  - 設定期間を重複させない
  - 設定期間に欠落を作らない
  - 現在継続中の設定を1件だけ保持する
- 利用者境界
  - 操作対象利用者に帰属する資産口座の履歴だけを取得できる
  - 他利用者に属する資産口座の履歴は取得できない
  - 他利用者の資産口座は対象不存在として扱う
- 論理削除
  - 論理削除済み資産口座は通常の履歴取得対象としない

具体的な章番号は、`functional-requirements.md`の最新定義に従う。

---

## 24. インデックス

ACC-005では、主に以下の検索条件を使用する。

```text
asset_accounts.id
```

```text
asset_accounts.user_id
```

```text
asset_account_available_settings.asset_account_id
```

また、履歴の並び順として

```text
asset_account_available_settings.start_year_month
```

を使用する。

---

### 24.1 asset_accounts.id

`asset_accounts.id`は主キーであるため、主キーインデックスを使用する。

ACC-005専用として追加インデックスを作成しない。

---

### 24.2 asset_accounts.user_id

ACC-005では、

```text
id
+
user_id
```

を条件として対象資産口座を取得する。

`id`は主キーであるため、Phase1ではACC-005専用に

```text
(id, user_id)
```

の複合インデックスを追加する必要性は低い。

実際の実行計画を踏まえて判断する。

---

### 24.3 asset_account_available_settings.asset_account_id

ACC-005では、

```text
asset_account_id
```

を条件として履歴全体を取得する。

FK列としてインデックスを設定する、または既存インデックスを利用する。

---

### 24.4 asset_account_id + start_year_month

履歴取得では、

```text
asset_account_id
```

で絞り込み、

```text
start_year_month DESC
```

で並び替える。

そのため、必要に応じて

```text
asset_account_id
+
start_year_month
```

の複合インデックスを検討する。

また、

```text
asset_account_id
+
start_year_month
```

をUNIQUE制約としている場合は、そのインデックスを履歴取得にも利用できる。

---

### 24.5 不要な重複インデックスを作成しない

ACC-005のためだけに、既存の

- 主キー
- FK用インデックス
- UNIQUE制約

と役割が重複するインデックスを追加しない。

実際のSQL、データ件数、実行計画を確認したうえで追加を判断する。

---

## 25. 性能

ACC-005は、単一資産口座に紐づく利用可能資産設定履歴を取得するAPIである。

利用可能資産設定は年月単位で変更されるため、Phase1では1資産口座あたりの件数は比較的少ないことを想定する。

---

### 25.1 資産口座一覧を取得しない

以下のように、操作対象利用者の資産口座一覧を取得してから対象資産口座を探す実装は行わない。

```text
資産口座一覧取得
    ↓
PHP側でassetAccountId検索
```

対象IDと利用者IDを条件として直接1件取得する。

---

### 25.2 履歴を資産口座IDで直接絞り込む

利用可能資産設定について、全資産口座分を取得してからPHP側で絞り込まない。

以下の条件をDBへ指定する。

```text
asset_account_id
    = assetAccountId
```

---

### 25.3 ページングを行わない

利用可能資産設定履歴は、年月単位で変更される比較的少数件のデータである。

Phase1ではページングを導入せず、対象資産口座の履歴を全件取得する。

---

### 25.4 不要な関連データを取得しない

ACC-005では、以下を取得しない。

- 保有商品一覧
- 月末資産残高
- 商品別月末評価額
- 月末資産状況
- 手取り収入
- 目的
- 目的達成判定履歴

必要な

```text
asset_accounts
+
asset_account_available_settings
```

だけを参照する。

---

### 25.5 N+1問題

ACC-005は単一資産口座を対象とし、履歴一覧も単一Queryで取得できるため、通常の意味でのN+1問題は発生しない。

各履歴について追加Queryを実行しない。

---

### 25.6 履歴整合性確認

履歴整合性確認は、取得した履歴配列に対してアプリケーション側で1回走査する程度とする。

概念的には、

```text
履歴取得
    ↓
開始年月順に整列
    ↓
前後レコードを比較
```

とし、各履歴ごとに追加SQLを実行しない。

---

## 26. セキュリティ

ACC-005では、利用者境界を最重要のセキュリティ要件とする。

---

### 26.1 assetAccountIdだけで履歴を取得しない

以下のような処理は基本としない。

```php
AssetAccountAvailableSetting::query()
    ->where(
        'asset_account_id',
        $assetAccountId,
    )
    ->get();
```

その前に、対象資産口座が

```text
userId
+
assetAccountId
+
deleted_at IS NULL
```

を満たすことを確認する。

---

### 26.2 他利用者の履歴を公開しない

他利用者に属する`assetAccountId`が指定された場合は、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下のような情報を返却しない。

```text
この資産口座は
別利用者に属しています
```

---

### 26.3 論理削除状態を公開しない

論理削除済み資産口座も、

```text
ASSET_ACCOUNT_NOT_FOUND
```

として扱う。

以下のような状態判別可能な専用エラーを返却しない。

```text
ASSET_ACCOUNT_DISABLED
ASSET_ACCOUNT_DELETED
```

---

### 26.4 user_idを返却しない

ACC-005のレスポンスには、

```text
user_id
```

を含めない。

利用者は`X-User-Id`によって既に確定しており、履歴レコードごとに所有者情報を返す必要はない。

---

### 26.5 asset_account_idを返却しない

ACC-005のURLで対象資産口座は既に指定されている。

そのため、各履歴に

```text
assetAccountId
```

や

```text
asset_account_id
```

を重複して返却しない。

---

### 26.6 内部日時を返却しない

以下のDB管理項目は返却しない。

```text
created_at
updated_at
```

利用可能資産設定の業務上必要な期間は、

```text
startYearMonth
endYearMonth
```

で表現する。

---

### 26.7 SQLインジェクション対策

`assetAccountId`や利用者IDを検索条件に使用する場合は、EloquentまたはQuery Builderのバインド機構を使用する。

SQL文字列へ入力値を直接連結しない。

---

### 26.8 エラー情報

異常時に、以下をレスポンスへ含めない。

- SQL
- PostgreSQL内部エラー
- テーブル名
- カラム名
- 制約名
- Laravel内部例外
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

詳細情報は、サーバーログへ記録する。

---

## 27. ログ・監視

ACC-005では、API共通ログ方針に従って処理結果を記録する。

参照APIであるため、取得した履歴内容そのものを通常ログへ不要に出力しない。

---

### 27.1 ログコンテキスト

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
ACC-005
```

とする。

---

### 27.2 正常時

正常終了時は、必要に応じて以下を記録する。

```text
requestId
userId
apiId = ACC-005
assetAccountId
historyCount
httpStatus = 200
```

履歴件数は障害調査上有用な場合に記録してよい。

---

### 27.3 ログへ履歴内容を出力しない

通常ログへ、以下の履歴内容を不要に出力しない。

```text
startYearMonth
endYearMonth
isAvailable
```

必要な識別情報と処理結果だけを記録する。

---

### 27.4 異常時

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

### 27.5 履歴0件

以下のエラーでは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

必要に応じて、

```text
assetAccountId
historyCount = 0
```

を内部ログへ記録する。

---

### 27.6 履歴不整合

以下のエラーでは、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

原因調査に必要な範囲で、内部ログへ不整合種別を記録してよい。

例えば、

```text
INITIAL_START_MISMATCH
INVALID_PERIOD
PERIOD_OVERLAP
PERIOD_GAP
MULTIPLE_OPEN_PERIODS
LATEST_PERIOD_CLOSED
DUPLICATE_START_YEAR_MONTH
```

などを内部的な診断情報として使用してよい。

これらをAPIレスポンスの独自エラーコードとして細分化する必要はない。

---

### 27.7 requestId

クライアントへ返却する`requestId`とサーバーログを関連付けられるようにする。

概念的には、

```text
requestId
    ↓
ACC-005ログ
    ↓
履歴不整合原因調査
```

が可能な状態とする。

---

## 28. 設計上の補足

### 28.1 履歴取得APIとして分離する理由

ACC-005は、利用可能資産設定の履歴一覧取得に責務を限定する。

現在状態だけを返却するACC-003とは分離する。

概念的には、

```text
ACC-003
    ↓
現在のisAvailable

ACC-005
    ↓
過去から現在までの履歴
```

とする。

現在状態と履歴参照を同じAPIへ混在させないことで、それぞれの責務を明確にする。

---

### 28.2 利用可能資産区分を履歴管理する理由

利用可能資産区分は、対象年月によって変化する可能性がある。

例えば、

```text
2026-01 ～ 2026-06
isAvailable = true

2026-07 ～ 2026-12
isAvailable = false

2027-01 ～
isAvailable = true
```

のような状態を表現する必要がある。

単純に

```text
asset_accounts.is_available
```

として現在値だけを保持すると、過去時点の状態を再現できなくなる。

そのため、

```text
asset_account_available_settings
```

として履歴管理する。

---

### 28.3 年月単位で管理する理由

Life Plannerの資産管理や目的達成判定は、月単位を基本としている。

そのため、利用可能資産設定も

```text
YYYY-MM
```

単位で管理する。

日単位や時刻単位まで細かく管理しない。

概念的には、

```text
2026-08-15
```

ではなく、

```text
2026-08
```

を業務上の設定単位とする。

---

### 28.4 startYearMonthとendYearMonthで期間を表現する理由

利用可能資産設定には、

```text
startYearMonth
endYearMonth
```

を持たせる。

これにより、特定年月にどの設定が有効だったかを明示的に判定できる。

例えば、

```text
startYearMonth = 2026-01
endYearMonth   = 2026-06
```

は、

```text
2026年1月から
2026年6月まで
```

を表す。

---

### 28.5 endYearMonthをNULL許可する理由

現在継続中の設定については、終了年月がまだ確定していない。

そのため、

```text
endYearMonth = null
```

として表現する。

例えば、

```text
startYearMonth = 2026-08
endYearMonth   = null
```

は、

```text
2026-08 ～
```

という継続中の設定を表す。

将来年月を

```text
9999-12
```

などのダミー値で表現しない。

---

### 28.6 最新設定をendYearMonth = nullとする理由

現在利用中の資産口座では、最新設定が現在以降も継続している必要がある。

そのため、正常な履歴では、

```text
最新設定
    endYearMonth = null
```

とする。

これにより、現在設定を明確に識別できる。

---

### 28.7 継続中設定を1件だけとする理由

同一資産口座で、

```text
endYearMonth = null
```

の設定が複数存在すると、

```text
現在どの設定が有効なのか
```

を一意に判断できない。

例えば、

```text
設定A
2026-01 ～ NULL
isAvailable = true

設定B
2026-08 ～ NULL
isAvailable = false
```

では、2026-08以降に2つの異なる状態が同時に成立する。

そのため、継続中設定は1件だけとする。

---

### 28.8 履歴期間を重複させない理由

同じ年月に複数の設定が有効になると、利用可能資産区分を一意に判断できない。

例えば、

```text
設定A
2026-01 ～ 2026-08

設定B
2026-08 ～ NULL
```

では、2026-08が両設定に含まれる。

そのため、正常な履歴では期間を重複させない。

---

### 28.9 履歴期間に欠落を作らない理由

利用可能資産設定は、資産口座の利用開始以降、各年月について必ず状態を判定できる必要がある。

例えば、

```text
設定A
2026-01 ～ 2026-05

設定B
2026-07 ～ NULL
```

では、

```text
2026-06
```

の状態を判断できない。

そのため、期間に欠落を作らない。

---

### 28.10 期間を連続させる理由

利用可能資産設定履歴は、概念的に

```text
前設定終了年月の翌月
    =
次設定開始年月
```

となるように管理する。

例えば、

```text
2026-01 ～ 2026-06
2026-07 ～ 2026-12
2027-01 ～ NULL
```

のようにする。

これにより、資産口座利用開始以降の任意の年月について設定を一意に特定できる。

---

### 28.11 初期設定を資産口座利用開始年月から開始する理由

資産口座は、

```text
startYearMonth
```

から利用開始される。

そのため、最初の利用可能資産設定も同じ年月から開始する。

概念的には、

```text
asset_accounts.start_year_month
    =
最古の設定.start_year_month
```

とする。

これにより、資産口座の利用開始時点から利用可能資産区分を判定できる。

---

### 28.12 ACC-002で初期設定を作成する理由

資産口座だけを登録して利用可能資産設定を後から任意登録とすると、

```text
資産口座は存在する
+
利用可能資産設定は存在しない
```

という不完全な状態が発生する。

そのため、ACC-002では資産口座登録時に初期利用可能資産設定も作成する。

概念的には、

```text
ACC-002
    ↓
asset_accounts
    INSERT
    +
asset_account_available_settings
    INSERT
```

とする。

---

### 28.13 履歴0件を正常な空配列としない理由

ACC-002で初期設定を必ず作成するため、利用中の資産口座で

```text
履歴0件
```

は通常発生しない。

そのため、

```json
{
  "data": []
}
```

として正常状態に見せない。

業務データ不整合として扱う。

---

### 28.14 ACC-005で履歴整合性を検証する理由

ACC-005は、利用可能資産設定履歴を利用者へ表示するAPIである。

不整合状態の履歴をそのまま返却すると、

```text
どの期間が正しいか
どの設定が現在有効か
```

をクライアント側で判断しなければならなくなる。

そのため、バックエンドで履歴が正常であることを確認したうえでレスポンスを返却する。

---

### 28.15 フロントエンドへ履歴整合性判定を任せない理由

期間重複や期間欠落の判定は、業務ルールである。

React側で

```text
期間を並べる
    ↓
重複を探す
    ↓
欠落を探す
```

という処理を実装すると、バックエンドと業務ルールが重複する。

そのため、整合性の最終保証はLaravel側で行う。

---

### 28.16 GETで不整合を自動修復しない理由

ACC-005は参照専用GET APIである。

履歴不整合を検出した際に、

```text
期間終了年月を修正
履歴を追加
履歴を削除
```

すると、GETが副作用を持つ。

HTTP APIとしての予測可能性も低下するため、ACC-005では修復しない。

---

### 28.17 不整合履歴の一部だけを返さない理由

例えば、3件中2件だけが正常でも、その2件だけを返却すると履歴全体の意味が変わる。

ACC-005では、

```text
利用可能資産設定履歴全体
```

を1つの整合した時系列として扱う。

そのため、1件でも業務上重大な不整合があれば正常レスポンスを返却しない。

---

### 28.18 不整合理由をAPIエラーコードとして細分化しない理由

履歴不整合には、

```text
INITIAL_START_MISMATCH
INVALID_PERIOD
PERIOD_OVERLAP
PERIOD_GAP
INVALID_OPEN_PERIOD_COUNT
LATEST_PERIOD_CLOSED
```

など複数の原因が存在する。

しかし、これらはクライアントが個別対応すべき業務エラーではない。

そのため、外部向けには、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_INVALID
```

へ集約する。

内部ログやUnit Testでは、詳細理由を区別してよい。

---

### 28.19 履歴0件だけ別エラーとする理由

履歴0件は、

```text
履歴自体が存在しない
```

という状態であり、存在する履歴の期間不整合とは性質が異なる。

そのため、

```text
ASSET_ACCOUNT_AVAILABLE_SETTING_HISTORY_NOT_FOUND
```

として分離する。

これにより、障害調査時にも原因を把握しやすくする。

---

### 28.20 履歴の返却順を降順とする理由

画面上では、現在または最新の設定を最初に確認する利用が多いと想定する。

そのため、APIでは

```text
startYearMonth DESC
```

で返却する。

例えば、

```text
2027-01 ～ 現在
2026-07 ～ 2026-12
2026-01 ～ 2026-06
```

の順とする。

---

### 28.21 整合性検証では昇順を使用してよい理由

履歴整合性確認では、古い設定から新しい設定へ順に比較すると、

```text
前設定.endYearMonth
次設定.startYearMonth
```

を比較しやすい。

そのため、

```text
検証
    → ASC

レスポンス
    → DESC
```

と内部処理順とAPI返却順を分離してよい。

---

### 28.22 履歴一覧をページングしない理由

利用可能資産設定は、年月単位で変更される。

1資産口座について、短期間に大量の履歴が作成されることは想定しにくい。

Phase1では、ページングを導入することでAPI・React双方の複雑性を増やす必要性が低い。

そのため、履歴全件を返却する。

---

### 28.23 クエリパラメータを設けない理由

ACC-005では、

```text
currentOnly
targetYearMonth
fromYearMonth
toYearMonth
isAvailable
```

などの絞り込み条件を持たせない。

履歴取得APIとして全履歴を返すことに責務を限定する。

現在状態はACC-003、設定変更はACC-006へ分離する。

---

### 28.24 currentOnlyを設けない理由

以下のようなAPIにすると、

```http
GET /api/v1/asset-accounts/{assetAccountId}/available-settings?currentOnly=true
```

ACC-005が

```text
履歴取得
+
現在状態取得
```

の2責務を持つことになる。

現在設定はACC-003で既に取得できるため、重複機能を追加しない。

---

### 28.25 targetYearMonthを設けない理由

特定年月時点の利用可能資産区分が必要な処理は、月末資産管理や目的達成判定などの各ユースケース側で対象年月を基準に判定する。

ACC-005を汎用的な年月検索APIへ広げない。

---

### 28.26 現在年月に依存させない理由

ACC-005は履歴全体を返却する。

そのため、サーバーの

```text
currentYearMonth
```

によって取得結果を変化させない。

同じDB状態であれば、年月が変わっても同じ履歴一覧を返却する。

---

### 28.27 `isCurrent`を返さない理由

現在継続中の設定は、

```text
endYearMonth = null
```

から判定できる。

そのため、

```text
isCurrent
```

という重複した派生情報をPhase1では返却しない。

API項目を必要以上に増やさない。

---

### 28.28 `assetAccountId`を各履歴へ返さない理由

ACC-005では、URLで既に

```text
assetAccountId
```

を指定している。

そのため、各履歴へ

```text
assetAccountId
```

を繰り返し含める必要はない。

レスポンスを履歴固有の情報に限定する。

---

### 28.29 資産口座情報を各履歴へ含めない理由

以下の情報は、利用可能資産設定履歴そのものではない。

```text
name
assetType
balanceRecordingUnit
startYearMonth
isEnabled
```

資産口座詳細はACC-003から取得できるため、ACC-005へ重複して含めない。

---

### 28.30 `createdAt`と`updatedAt`を返さない理由

利用可能資産設定において業務上重要なのは、

```text
startYearMonth
endYearMonth
isAvailable
```

である。

DBへいつINSERTされたか、いつUPDATEされたかという

```text
created_at
updated_at
```

はPhase1の画面要件では不要とする。

---

### 28.31 履歴IDを返す理由

各設定履歴をReact上で一意に識別できるように、

```text
id
```

を返却する。

例えば、

```tsx
key={setting.id}
```

として利用できる。

ただし、履歴IDを返すことと過去履歴の直接編集を許可することは別である。

---

### 28.32 過去履歴を直接編集させない理由

利用可能資産設定は、時系列として整合した履歴を形成する。

過去の1レコードだけを自由に編集すると、

```text
期間重複
期間欠落
現在設定不整合
```

が発生しやすい。

そのため、Phase1では過去履歴の直接編集APIを提供しない。

---

### 28.33 ACC-006で設定変更する理由

利用可能資産区分を変更する際は、

```text
旧設定を終了
+
新設定を追加
```

という履歴操作が必要になる。

そのため、単純なUPDATEではなくACC-006へ責務を集約する。

概念的には、

```text
旧設定
2026-01 ～ NULL
    ↓
ACC-006
    ↓
旧設定
2026-01 ～ 2026-07

新設定
2026-08 ～ NULL
```

とする。

---

### 28.34 ACC-006成功後にACC-005を再取得する理由

ACC-006では、

```text
旧設定.endYearMonth
```

と

```text
新設定
```

の両方が変化する。

React側でRequest内容だけから履歴全体を組み立てるより、

```text
ACC-006成功
    ↓
ACC-005再取得
```

とする方が、サーバー上の確定状態を正として扱える。

---

### 28.35 ACC-003もACC-006後に再取得する理由

ACC-006によって現在の利用可能資産区分が変化すると、

```text
ACC-003.isAvailable
```

も変化する。

そのため、ACC-006成功後は、

```text
ACC-005
履歴

+
ACC-003
現在状態
```

の双方を再取得対象とする。

---

### 28.36 ACC-004後に履歴再取得を必須としない理由

ACC-004では、

```text
name
assetType
```

のみを更新する。

利用可能資産設定履歴は変更されない。

そのため、ACC-004成功だけを理由にACC-005のQuery Cacheを必ず失効させる必要はない。

---

### 28.37 資産口座無効化後に履歴を通常取得させない理由

ACC-005は、現在利用中の資産口座の設定履歴確認を目的とする。

資産口座が論理削除された場合は、通常画面の管理対象から外れる。

そのため、無効化済み資産口座についてACC-005を通常利用させない。

---

### 28.38 論理削除済み資産口座の履歴を物理削除しない理由

資産口座を無効化しても、

```text
asset_account_available_settings
```

の過去履歴を削除しない。

過去の月末資産や目的達成判定を再現するために必要となる可能性があるためである。

ACC-005で通常取得できないことと、DBから履歴を削除することは別とする。

---

### 28.39 Repositoryを使用しない理由

ACC-005は参照専用APIであり、

```text
INSERT
UPDATE
DELETE
```

を行わない。

そのため、取得処理はQueryへ集約し、Repositoryは使用しない。

概念的には、

```text
Query
    → 読み取り

Repository
    → 書き込み
```

とする。

---

### 28.40 AssetAccountQueryを使用する理由

利用可能資産設定には、直接

```text
user_id
```

を持たせない。

そのため、まずAssetAccountQueryで

```text
userId
+
assetAccountId
+
利用中状態
```

を確認する。

利用者境界を確定した後に履歴を取得する。

---

### 28.41 AssetAccountAvailableSettingQueryを分離する理由

資産口座の存在確認と利用可能資産設定履歴の取得は、検索対象・条件が異なる。

そのため、

```text
AssetAccountQuery
    → 親リソース確認

AssetAccountAvailableSettingQuery
    → 履歴取得
```

と分離する。

---

### 28.42 HistoryValidatorを分離する理由

利用可能資産設定履歴には、

- 初期年月
- 期間大小
- 期間重複
- 期間欠落
- 継続中設定件数
- 最新設定

など、複数の整合性ルールが存在する。

これらをUseCaseへ直接書き並べると、UseCaseの責務が大きくなる。

そのため、

```text
AvailableSettingHistoryValidator
```

へ期間整合性判定を分離する。

---

### 28.43 ValidatorをDBアクセス責務にしない理由

HistoryValidatorは、取得済みの履歴を受け取り、整合性を判定する。

Validator自身がDBから履歴を取得しない。

概念的には、

```text
Query
    ↓
履歴取得
    ↓
Validator
    ↓
業務ルール判定
```

とする。

これにより、Queryと業務判定の責務を分離する。

---

### 28.44 YearMonth Value Objectを導入してよい理由

利用可能資産設定では、年月比較や翌月計算を繰り返し使用する。

例えば、

```text
2026-12
    ↓
2027-01
```

の計算や、

```text
2026-07 < 2026-08
```

の比較が必要になる。

将来的に複数のユースケースで同じ年月ロジックを使う場合は、

```text
YearMonth
```

Value Objectへ集約してよい。

---

### 28.45 Phase1でYearMonthを必須としない理由

Value Objectは有効だが、Phase1で利用箇所が限定されている場合は、クラス数を増やしすぎる可能性がある。

そのため、Laravelの日付機能等で十分に安全に扱えるなら、Phase1では専用Value Objectを必須としない。

設計の複雑さと再利用性を見て判断する。

---

### 28.46 FormRequestを作成しない理由

ACC-005には、

```text
Request Body
Query Parameter
```

が存在しない。

専用FormRequestを作成しても空クラスとなるため、作成しない。

`assetAccountId`はルート制約または共通パスパラメータ検証で扱う。

---

### 28.47 明示的なトランザクションを使用しない理由

ACC-005は、参照専用であり業務データを変更しない。

また、履歴表示用途として厳密な同一時点読み取りを必須としない。

そのため、

```php
DB::transaction()
```

を使用しない。

---

### 28.48 `lockForUpdate()`を使用しない理由

ACC-005が履歴参照中であることを理由に、ACC-006の設定変更を待機させる必要はない。

そのため、

```php
lockForUpdate()
```

を使用しない。

更新時の整合性保証は、ACC-006側で行う。

---

### 28.49 DTOを使用する理由

Eloquent ModelをそのままResourceへ渡すと、

```text
asset_account_id
created_at
updated_at
```

などのDB構造へAPI層が依存しやすくなる。

そのため、UseCaseからは必要な値だけを持つResult DTOを返却する。

---

### 28.50 Resource Collectionを使用する理由

ACC-005は一覧取得APIである。

そのため、履歴1件の変換をAPI Resourceへ集約し、一覧はResource Collectionとして返却する。

概念的には、

```text
Result DTO[]
    ↓
Resource Collection
    ↓
data: []
```

とする。

---

### 28.51 Responderを使用する理由

Responderは、

```text
UseCase Result
    ↓
HTTP 200
    ↓
共通Envelope
```

への変換に責務を限定する。

履歴取得や整合性判定をResponderへ持ち込まない。

---

### 28.52 TanStack QueryのQueryとして扱う理由

ACC-005はGETによる参照APIである。

そのため、React側ではMutationではなくQueryとして扱う。

Query Cacheによって、同じ履歴を不要に再取得することを抑制できる。

---

### 28.53 ACC-003とQuery Keyを分ける理由

ACC-003とACC-005は同じ資産口座を対象とするが、データの意味が異なる。

```text
ACC-003
    → 資産口座現在詳細

ACC-005
    → 利用可能資産設定履歴
```

そのため、同じQuery Keyへ両レスポンスを混在させない。

---

### 28.54 ACC-006成功後にinvalidateする理由

ACC-006では、利用可能資産設定履歴が確実に変化する。

そのため、ACC-005のキャッシュを最新状態ではないものとして扱い、

```text
invalidate
    ↓
再取得
```

する。

---

### 28.55 Optimistic Updateを必須としない理由

ACC-006は、

```text
旧設定終了
+
新設定登録
```

という複数レコードの状態変更を伴う。

React側でACC-005の履歴配列を先に推測更新すると、サーバー確定状態との差異が生じる可能性がある。

Phase1では再取得方式を基本とする。

---

### 28.56 利用者切替時にキャッシュ境界を考慮する理由

Phase1では、`X-User-Id`によって操作対象利用者を切り替える。

API側で利用者境界を保証していても、ReactのQuery Cacheに前利用者の履歴が残っていると、画面上で一時的に誤表示する可能性がある。

そのため、Query Keyまたは利用者切替時のinvalidateによってキャッシュ境界も維持する。

---

### 28.57 Phase1ではサーバーキャッシュを導入しない理由

利用可能資産設定履歴は、資産口座単位で比較的少数件である。

また、ACC-006実行後のキャッシュ失効設計も必要になる。

Phase1では、性能上の必要性より複雑性の方が大きいと判断し、Redis等の専用キャッシュを導入しない。

---

### 28.58 Phase1ではHTTP条件付きリクエストを導入しない理由

ACC-005はGET APIであるため、

```text
ETag
Last-Modified
If-None-Match
If-Modified-Since
```

を利用することも可能である。

ただし、Phase1では履歴件数・アクセス頻度とも大規模ではないため、これらを導入しない。

必要になった段階でAPI共通方針として検討する。

---

### 28.59 Phase1では履歴検索機能を広げすぎない

ACC-005では、以下を追加しない。

- ページング
- 年月範囲検索
- 現在設定のみ取得
- 利用可能状態での絞り込み
- 過去履歴編集
- 過去履歴削除
- 履歴の手動補正
- `isCurrent`
- サーバー側キャッシュ
- ETag
- Last-Modified

Phase1では、

```text
指定資産口座の
利用可能資産設定履歴を
整合性確認したうえで
全件取得する
```

ことへ責務を限定する。

---

## 29. 関連ドキュメント

- [API共通方針](../api-common-policy.md)
- [API一覧](../api-list.md)
- [エラーコード一覧](../error-codes.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../requirements/glossary.md)
- [エンティティ定義](../../requirements/entities.md)
- [テーブル定義書](../../database/table-definition.md)
- [ER図](../../database/er-diagram-phase1.md)