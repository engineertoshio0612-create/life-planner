# OBJ-004 目的更新

## 1. 概要

操作対象となる利用者に登録された
指定した目的の情報を更新する。

Reactの目的編集画面で入力された内容をもとに、
OBJ-004 目的更新APIを呼び出す。

更新対象となる項目は、主に以下とする。

* 目的名
* 実施予定年月
* 必要支出額
* メモ

本APIではPATCHを使用し、
変更された項目のみをリクエストとして送信する。

未指定の項目は現在の値を保持し、
`plannedYearMonth`または`memo`を未設定へ変更する場合は、
明示的に`null`を送信する。

目的の利用状態を表す`enabled`は
本APIの更新対象に含めず、
利用状態の変更は目的無効化APIで行う。

無効化された目的は更新できないため、
React側では取得した`enabled`の状態に応じて
編集操作を制御する。

更新成功後は、
目的詳細および目的一覧など
対象目的を参照しているキャッシュを
再取得または無効化し、
サーバー側の最新状態と画面表示の整合性を保つ。

入力値に問題がある場合は、
バックエンドから返却されたバリデーションエラーを
対応するフォーム項目へ表示する。

同一利用者に同名の有効な目的が存在する場合や、
対象目的が無効化済みまたは存在し

---

## 2. React・TypeScriptでの利用

### 2.1 PATCHの扱い

本APIは、
PATCHを使用する。

更新する項目のみ送信する。

未指定の項目は、
現在の値を保持する。

---

### 2.2 plannedYearMonthの扱い

`plannedYearMonth`は、
`YYYY-MM`形式で送信する。

未設定にする場合は、
`null`を送信する。

未変更の場合は、
送信しない。

```ts
{
  plannedYearMonth: null
}
```

---

### 2.3 requiredExpenseの扱い

`requiredExpense`は、
日本円の整数値として扱う。

送信前に、
数値へ変換する。

```ts
requiredExpense:
  Number(form.requiredExpense);
```

小数点は送信しない。

---

### 2.4 memoの扱い

`memo`は、
任意入力とする。

削除する場合は、
`null`を送信する。

未変更の場合は、
送信しない。

```ts
{
  memo: null
}
```

---

### 2.5 enabledの扱い

`enabled`は、
レスポンスでのみ取得する。

更新APIでは、
送信しない。

利用状態の変更は、
目的無効化APIで行う。

---

### 2.6 ローディング表示

更新処理中は、
更新ボタンを非活性化する。

更新完了または
エラーになるまで、
再送信できないようにする。

---

### 2.7 エラー表示

入力項目ごとのエラーは、
`error.details.field`
を利用して表示する。

対象となる項目は、
以下とする。

- `name`
- `plannedYearMonth`
- `requiredExpense`
- `memo`

エラーコードごとの基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `VALIDATION_ERROR` | 入力欄へエラー表示 |
| `OBJECTIVE_ALREADY_EXISTS` | 「同じ目的名が登録されています」を表示 |
| `OBJECTIVE_DISABLED` | 「無効化された目的は更新できません」を表示 |
| `OBJECTIVE_NOT_FOUND` | 一覧画面へ戻し、対象が存在しないことを表示する |
| `INVALID_OBJECTIVE_ID` | 不正なURLとしてエラー表示する |
| `INVALID_USER_ID` | 共通エラー表示 |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示 |

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [OBJ-001 目的一覧取得](./obj-001-list.md)
- [OBJ-002 目的登録](./obj-002-create.md)
- [OBJ-003 目的詳細取得](./obj-003-detail.md)
- [OBJ-005 目的無効化](./obj-005-assessments.md)
- [OBJ-006 目的達成判定](./obj-006-disabled.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/objectives/README.md)
- [Reactアーキテクチャ設計](../../../architecture/react/objectives/README.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)
