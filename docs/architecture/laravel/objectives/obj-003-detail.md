# OBJ-003 目的詳細取得

## 1. 概要

操作対象となる利用者の目的を新規登録する。

目的には、主に以下の情報を登録する。

* 目的名
* 実施予定年月
* 必要支出額
* メモ

登録時には、共通の利用者コンテキストから
操作対象利用者を特定し、
登録した目的をその利用者へ紐付ける。

登録直後の目的は利用中として扱い、
利用状態をクライアントから指定することはできない。

同一利用者に同名の有効な目的が存在する場合は、
重複登録を許可せず業務エラーとする。

一方で、
他利用者に登録された同名の目的は
重複として扱わない。

登録した目的は、
目的達成判定の対象として利用できる。

本APIでは目的情報の登録のみを行い、
目的達成判定そのものは実行しない。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
目的IDおよび
利用者コンテキストを取得する。

目的詳細取得UseCaseを呼び出し、
取得結果をResponderへ渡す。

以下の処理は、
Actionへ直接記述しない。

- 目的IDの形式検証
- 利用者境界の判定
- 目的取得処理
- 論理削除状態の判定
- レスポンス生成処理

---

### 2.2 UseCase

目的詳細取得の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- 目的IDを受け取る
- Queryを呼び出して目的を取得する
- 取得結果を返却する

指定された目的が存在しない場合、
論理削除されている場合、
または操作対象利用者に帰属しない場合は、
`OBJECTIVE_NOT_FOUND`として扱う。

無効化された目的は、
詳細取得の対象とする。

---

### 2.3 パスパラメータ検証

`objectiveId`の形式検証は、
Route Model Bindingへ全面的に依存せず、
API共通方針に従って実施する。

以下を検証する。

- 必須であること
- API共通方針で定めたID形式であること

形式が不正な場合は、
`INVALID_OBJECTIVE_ID`として扱う。

目的の存在確認および
利用者境界の確認は、
Queryで行う。

---

### 2.4 Query

操作対象利用者に帰属する
目的を1件取得する。

取得条件には、
必ず目的IDおよび
操作対象利用者IDを含める。

取得例：

```php
$objective = Objective::query()
    ->where('id', $objectiveId)
    ->where('user_id', $userId)
    ->first();
```

LaravelのSoftDeletesを利用する場合、
通常のEloquentクエリによって
論理削除済みの目的を取得対象から除外する。

`enabled = false` の目的は
論理削除済みではないため、
通常どおり取得対象とする。

以下のように、
目的IDだけで取得してはならない。

```php
Objective::find($objectiveId);
```

取得できなかった場合は、
原因を外部へ区別して公開せず、
`OBJECTIVE_NOT_FOUND` として扱う。

### 2.5 Repository

本APIは参照系APIであるため、
目的の登録、
更新および無効化は行わない。

永続化処理を必要としないため、
Repositoryは使用しない。

目的の取得は、
参照専用のQueryが担当する。

### 2.6 トランザクション

本APIは参照処理のみであるため、
明示的なデータベーストランザクションは使用しない。

目的情報の取得によって、
データベースの状態を変更しない。

Phase1では、
取得処理中の厳密なスナップショット分離は保証しない。

### 2.7 Responder

UseCaseから受け取った目的を、
API共通方針に従った
HTTPレスポンスへ変換する。

正常終了時は、
`200 OK` とともに
目的詳細を `data` オブジェクトで返却する。

目的が取得できない場合は、
`404 Not Found` および
`OBJECTIVE_NOT_FOUND` を返却する。

Responderは、
以下を行わない。

- 目的の取得
- 利用者境界の判定
- 利用状態の判定
- 業務ルールの判定

### 2.8 API Resource

データベースカラムを直接返却せず、
API Resourceを利用して
APIレスポンス形式へ変換する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'name' => $this->name,
    'plannedYearMonth' => $this->planned_year_month,
    'requiredExpense' => $this->required_expense,
    'memo' => $this->memo,
    'enabled' => $this->enabled,
];
```

`planned_year_month` が未設定の場合は、
`plannedYearMonth` へ `null` を設定する。

`memo` が未設定の場合は、
`memo` へ `null` を設定する。

以下の項目は、
レスポンスへ含めない。

- `userId`
- `createdAt`
- `updatedAt`
- `deletedAt`

目的達成判定結果および
判定履歴も、
本APIでは返却しない。

### 2.9 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、
`X-User-Id` を検証し、
操作対象利用者を特定する。

Action以降では、
検証済みの利用者コンテキストを使用する。

### 2.10 例外変換

LaravelおよびPostgreSQLの内部例外は、
そのままAPIレスポンスへ公開しない。

主な例外変換は、
以下とする。

| 内部状態 | 独自エラーコード |
|---|---|
| 利用者未指定 | `USER_CONTEXT_REQUIRED` |
| 利用者ID形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| 目的ID形式不正 | `INVALID_OBJECTIVE_ID` |
| 目的不存在 | `OBJECTIVE_NOT_FOUND` |
| 論理削除済み目的 | `OBJECTIVE_NOT_FOUND` |
| 利用者境界外の目的 | `OBJECTIVE_NOT_FOUND` |
| 想定外例外 | `INTERNAL_SERVER_ERROR` |

目的が存在しない場合、
論理削除されている場合、
または他利用者に帰属する場合を、
外部レスポンスでは区別しない。

SQL、
スタックトレースおよび
内部例外メッセージは、
APIレスポンスへ含めない。

ログには、
調査に必要な範囲で
以下を記録する。

- 操作対象利用者ID
- 指定された目的ID
- 独自エラーコード
- リクエストID

---

## 3. 関連ドキュメント

- [OBJ-003 API詳細設計](../../../api/details/objectives/obj-003-detail.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [目的 Laravelアーキテクチャ設計](./README.md)
- [OBJ-003 テスト設計](../../../tests/objectives/obj-003-detail.md)
- [目的 テスト設計](../../../tests/objectives/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
