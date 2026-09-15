# INC-003 手取り収入詳細取得

## 1. 概要

本ドキュメントでは、

INC-003 手取り収入詳細取得APIを

Laravelで実装する際の

アーキテクチャおよび責務分離方針を定義する。

本APIでは、

操作対象となる利用者に登録された

指定の手取り収入を取得し、

詳細情報として返却する。

手取り収入は、

利用者ごとに対象年月単位で管理し、

詳細取得では、

対象年月、

手取り収入金額および

備考を取得する。

Laravel実装では、

以下の責務に分離する。

- Action
  - HTTPリクエストの受付
  - パスパラメータの取得
  - 利用者コンテキストの取得
  - UseCaseの呼び出し
  - Responderへの処理結果の受け渡し

- UseCase
  - 手取り収入詳細取得処理の制御
  - Queryによる手取り収入の取得
  - 対象データが存在しない場合の業務例外への変換

- Query
  - 操作対象利用者に帰属する手取り収入の取得
  - 利用者IDおよび手取り収入IDによる検索

- Responder
  - API共通方針に従ったHTTPレスポンスへの変換
  - エラーレスポンスへの変換

- API Resource
  - データベースモデルからAPIレスポンス形式への変換
  - snake_caseからcamelCaseへの変換
  - IDの文字列化

取得処理では、

手取り収入IDだけで検索せず、

利用者コンテキストから取得した利用者IDを

必ず検索条件に含める。

これにより、

指定された手取り収入が存在しない場合と、

他利用者に帰属する場合を区別せず、

`NET_INCOME_NOT_FOUND`として扱うことで、

利用者境界を保証する。

本APIは参照処理のみを行い、

手取り収入の登録、

更新および削除は行わない。

取得した手取り収入は、

主に手取り収入編集画面の初期表示および

登録内容の確認に利用する。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
パスパラメータおよび
利用者コンテキストを取得する。

手取り収入詳細取得UseCaseを呼び出し、
取得結果をResponderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 利用者境界の判定
- 手取り収入取得処理
- レスポンス生成処理

---

### 2.2 UseCase

手取り収入詳細取得の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 手取り収入IDを受け取る
- 手取り収入を取得する
- 取得結果を返却する

指定された手取り収入が存在しない場合、
または操作対象利用者に帰属しない場合は、
`NET_INCOME_NOT_FOUND`として扱う。

---

### 2.3 Query

操作対象利用者に帰属する
手取り収入を取得する。

取得条件には、
必ず操作対象利用者IDを含める。

取得例：

```php
$netIncome = NetIncome::query()
    ->where('id', $netIncomeId)
    ->where('user_id', $userId)
    ->first();
```

以下の取得は行わない。

```php
NetIncome::find($netIncomeId);
```

取得できなかった場合は、
`NET_INCOME_NOT_FOUND`として扱う。

論理削除を採用しないため、
SoftDeletesに関する検索条件は使用しない。

---

### 2.4 Responder

UseCaseから受け取った
手取り収入を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`200 OK`とともに
`data`オブジェクトで返却する。

対象が存在しない場合は、
共通エラーレスポンス形式へ変換する。

---

### 2.5 API Resource

データベースカラムを直接返却せず、
API Resourceを利用して
レスポンス形式へ変換する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'targetYearMonth' => $this->target_year_month,
    'amount' => $this->amount,
    'memo' => $this->memo,
];
```

以下の項目は、
レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`

API Resourceは、
Responderから利用する。

---

### 2.6 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、
`X-User-Id`を検証し、
操作対象利用者を特定する。

Action以降の処理では、
検証済みの利用者コンテキストを使用する。

---

### 2.7 例外変換

LaravelおよびPostgreSQLの内部例外は、
そのままAPIレスポンスへ公開しない。

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 手取り収入ID形式不正 | `INVALID_NET_INCOME_ID` |
| 手取り収入不存在 | `NET_INCOME_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

SQL、
スタックトレースおよび
内部例外メッセージは、
APIレスポンスへ含めない。

---

## 3. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)