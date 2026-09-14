# OBJ-002 目的登録

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

OBJ-002では、目的登録に必要な責務を各レイヤーへ分離する。

共通的なLaravelアーキテクチャは`docs/architecture/laravel-architecture.md`に従い、本節ではOBJ-002固有の実装上の注意点のみ記載する。

---

### 2.1 Action

Actionは、検証済みのリクエストと利用者コンテキストを受け取り、目的登録UseCaseを呼び出す。

主な責務は以下とする。

* 利用者コンテキストを受け取る
* FormRequestから登録値を取得する
* UseCaseへ入力値を渡す
* 処理結果をResponderへ渡す

以下の処理はActionへ直接記述しない。

* 目的名の重複判定
* データベース検索
* 目的登録
* トランザクション制御
* レスポンス形式への変換

---

### 2.2 FormRequest

リクエストボディの入力値検証には、OBJ-002専用のFormRequestを使用する。

主な検証対象は以下とする。

```text
name
plannedYearMonth
requiredExpense
memo
```

FormRequestでは、必須、型、文字数、年月形式、金額範囲などの単項目バリデーションを担当する。

同一利用者内の目的名重複など、データベース参照を伴う業務ルールはFormRequestでは判定しない。

---

### 2.3 Input DTO

UseCaseへ渡す入力値を明確にするため、必要に応じてInput DTOを使用する。

概念的には以下の値を保持する。

```text
userId
name
plannedYearMonth
requiredExpense
memo
```

`userId`はリクエストボディから取得せず、共通の利用者コンテキストから設定する。

---

### 2.4 UseCase

UseCaseは、目的登録の業務処理全体を担当する。

主な処理は以下とする。

```text
入力値を受け取る
    ↓
同一利用者の目的名重複を確認
    ↓
重複している場合は業務エラー
    ↓
トランザクション開始
    ↓
目的登録
    ↓
登録結果生成
    ↓
トランザクション終了
```

目的登録時には、操作対象利用者IDを`objectives.user_id`へ設定する。

登録直後の目的は利用中として扱い、利用状態をリクエストから受け取らない。

---

### 2.5 Query

Queryは、目的登録前の参照処理を担当する。

OBJ-002では主に、同一利用者に同名の有効な目的が存在するか確認するために使用する。

概念的な検索条件は以下とする。

```text
user_id = 操作対象利用者ID
AND
name = 登録する目的名
AND
有効な目的
```

他利用者の同名目的は重複判定へ含めない。

---

### 2.6 Repository

Repositoryは、`objectives`への新規登録を担当する。

登録時には、主に以下を設定する。

```text
user_id
name
planned_year_month
required_expense
memo
enabled
```

`user_id`は利用者コンテキストから設定する。

登録直後は利用中とするため、`enabled`には有効状態を設定する。

HTTPリクエスト全体をそのままModelへ渡さず、登録対象項目を明示して保存する。

---

### 2.7 トランザクション

目的登録処理はトランザクション内で実行する。

OBJ-002で更新する主要テーブルは`objectives`のみだが、登録処理をユースケース単位で完結させ、例外発生時に中途半端な登録結果を残さない。

トランザクション境界はUseCase側で管理する。

---

### 2.8 同名目的の同時登録

アプリケーション側では登録前に目的名の重複確認を行う。

ただし、並行リクエストでは事前確認だけでは重複を完全に防止できないため、データベース側の一意性制約を最終的な整合性保証として使用する。

一意性制約違反が発生した場合は、PostgreSQLの例外をそのまま返却せず、

```text
OBJECTIVE_NAME_DUPLICATED
```

へ変換する。

---

### 2.9 Result DTO

UseCaseの処理結果は、必要に応じてResult DTOとして返却する。

主に以下の値を保持する。

```text
id
name
plannedYearMonth
requiredExpense
memo
enabled
```

ActionへEloquent Modelをそのまま返却しない。

---

### 2.10 API Resource

Result DTOまたは登録結果をAPIレスポンス形式へ変換する。

API Resourceでは、以下を行う。

* IDをstringとして返却する
* JSONフィールドをcamelCaseへ変換する
* DB内部のカラム名を直接公開しない
* API契約に必要な項目だけ返却する

OBJ-002では、以下のような内部項目をレスポンスへ含めない。

```text
user_id
created_at
updated_at
deleted_at
```

---

### 2.11 Responder

Responderは、登録結果を`201 Created`のHTTPレスポンスへ変換する。

業務ルール判定、データベース登録、目的名重複確認などは行わない。

エラー時の共通レスポンス形式および例外変換はLaravel共通設計に従う。

---

### 2.12 例外変換

OBJ-002固有の主な業務例外は、目的名重複とする。

```text
同一利用者に同名の有効な目的が存在
    ↓
ObjectiveNameDuplicatedException
    ↓
409 Conflict
OBJECTIVE_NAME_DUPLICATED
```

データベースの一意性制約違反によって重複を検出した場合も、同じ業務エラーへ変換する。

その他の利用者コンテキスト、バリデーション、想定外例外についてはLaravel共通設計に従う。

---

### 2.13 OBJ-002固有の実装上の注意

* `userId`をリクエストボディから取得しない。
* 登録時の利用状態をクライアントから指定させない。
* 目的登録時に目的達成判定を実行しない。
* 同一利用者内の目的名重複はQueryとDB制約の両方で防止する。
* 他利用者の同名目的は登録可能とする。
* HTTP層、業務処理、DB更新、レスポンス変換の責務を分離する。

---

## 3. 関連ドキュメント

- [OBJ-002 API詳細設計](../../../api/details/objectives/obj-002-create.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [目的 Laravelアーキテクチャ設計](./README.md)
- [OBJ-002 テスト設計](../../../tests/objectives/obj-002-create.md)
- [目的 テスト設計](../../../tests/objectives/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
