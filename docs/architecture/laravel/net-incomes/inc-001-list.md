# INC-001 手取り収入一覧取得

## 1. 概要

本ドキュメントでは、

INC-001 手取り収入一覧取得APIを

Laravelで実装する際の

アーキテクチャおよび責務分離方針を定義する。

本APIでは、

操作対象となる利用者に登録された

手取り収入を対象年月単位で取得し、

一覧として返却する。

一覧取得では、

対象年月、手取り収入額および備考を取得し、

対象年月の昇順または降順で並び替えを行う。

また、

取得件数が増加することを考慮し、

ページネーションに対応する。

Laravel実装では、

以下の責務に分離する。

- Action
  - HTTPリクエストの受付
  - 利用者コンテキストの取得
  - UseCaseの呼び出し
  - Responderへの処理結果の受け渡し

- UseCase
  - 手取り収入一覧取得処理の制御
  - Queryの呼び出し
  - 一覧取得結果およびページネーション情報の返却

- Form Request / DTO
  - クエリパラメータの受け取り
  - 入力値のバリデーション
  - 既定値の適用

- Query
  - 操作対象利用者に帰属する手取り収入の取得
  - 対象年月による並び替え
  - ページネーション

- Responder
  - API共通方針に従ったHTTPレスポンスへの変換
  - ページネーション情報の返却
  - エラーレスポンスへの変換

- API Resource
  - データベースモデルからAPIレスポンス形式への変換
  - snake_caseからcamelCaseへの変換
  - IDの文字列化

本APIは参照処理のみを行い、

手取り収入の登録、更新および削除は行わない。

また、

すべての取得処理では

利用者コンテキストから取得した利用者IDを条件に含め、

他利用者の手取り収入を取得できないようにする。

---

## 2. Laravel実装方針

### 2.1 Action

HTTPリクエストを受け付け、
ページネーション条件、
並び順および
利用者コンテキストを取得する。

Form Requestまたは入力用DTOを利用して
クエリパラメータを受け取り、
手取り収入一覧取得UseCaseを呼び出す。

UseCaseから受け取った処理結果を、
Responderへ渡す。

検索条件、
並び替え、
ページネーション処理および
レスポンス生成処理は、
Actionへ直接記述しない。

---

### 2.2 UseCase

手取り収入一覧取得の
ユースケース処理を担当する。

主な処理は、
以下とする。

- 操作対象利用者を受け取る
- ページ番号を受け取る
- 1ページ当たりの取得件数を受け取る
- 対象年月の並び順を受け取る
- Queryを呼び出して手取り収入一覧を取得する
- 取得結果とページネーション情報を返却する

本APIは参照処理のみであるため、
データの登録および更新は行わない。

---

### 2.3 Form Request / DTO

クエリパラメータの形式および
単項目バリデーションを担当する。

主な検証対象は、
以下とする。

- `page`
- `perPage`
- `order`

未指定の場合は、
以下の値を使用する。

- `page`
  - `1`
- `perPage`
  - API共通方針で定めた既定値
- `order`
  - `desc`

操作対象利用者の存在確認は、
利用者コンテキスト設定ミドルウェアで行う。

---

### 2.4 Query

操作対象利用者に帰属する
手取り収入一覧を取得する。

取得条件には、
必ず操作対象利用者IDを含める。

```php
NetIncome::query()
    ->where('user_id', $userId)
    ->orderBy(
        'target_year_month',
        $order,
    )
    ->paginate(
        perPage: $perPage,
        page: $page,
    );

---取得対象となる主なカラムは、以下とする。

- `id`
- `target_year_month`
- `amount`
- `memo`

利用者ID、登録日時および更新日時など、レスポンスに使用しない項目は必要に応じて取得対象から除外する。

手取り収入は論理削除を採用しないため、SoftDeletesに関する検索条件は使用しない。

### 2.5 Responder

UseCaseから受け取った一覧取得結果を、API共通方針に従ったHTTPレスポンスへ変換する。

正常終了時は、`200 OK` とともに手取り収入一覧を `data` 配列で返却する。

ページネーション情報は、`meta` オブジェクトとして返却する。

手取り収入が存在しない場合も、`200 OK` と空配列を返却する。

```json
{
  "data": [],
  "meta": {
    "currentPage": 1,
    "perPage": 20,
    "total": 0,
    "lastPage": 0
  }
}
```

エラー発生時は、共通エラーレスポンス形式へ変換する。

### 2.6 API Resource

データベースカラムを直接返却せず、API Resourceを利用してレスポンス形式へ変換する。

変換例：

```php
return [
    'id' => (string) $this->id,
    'targetYearMonth' => $this->target_year_month,
    'amount' => $this->amount,
    'memo' => $this->memo,
];
```

手取り収入IDは、API共通方針に従って文字列として返却する。

金額は、日本円の整数値として返却する。

API Resourceは、Responderから利用する。

### 2.7 Middleware

以下の共通ミドルウェアを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- JSONリクエスト・レスポンス共通処理
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定ミドルウェアでは、`X-User-Id` を検証し、操作対象利用者を特定する。

---

## 3. 関連ドキュメント

- [INC-001 API詳細設計](../../../api/details/net-incomes/inc-001-list.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
- [手取り収入 Laravelアーキテクチャ設計](./README.md)
- [INC-001 テスト設計](../../../tests/net-incomes/inc-001-list.md)
- [手取り収入 テスト設計](../../../tests/net-incomes/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)