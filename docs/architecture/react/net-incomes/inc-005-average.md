# INC-005 平均手取り収入取得

## 1. 概要

本ドキュメントでは、
INC-005 平均手取り収入取得APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

本APIでは、
操作対象となる利用者について、
指定された判定対象年月の直前にあたる
連続する3か月の手取り収入から算出された
平均手取り収入を取得する。

React・TypeScript実装では、
判定対象年月を

`YYYY-MM`形式の文字列としてAPIへ渡し、

平均手取り収入、
判定対象年月および
算出対象となった3か月を
レスポンスとして受け取る。

取得した平均手取り収入は、
主に以下で利用する。

- 目的達成判定画面
- 判定前の入力値確認
- 平均手取り収入の計算根拠表示

平均手取り収入の算出は、
バックエンドの業務ロジックとして扱う。

そのため、
フロントエンドでは平均値を再計算せず、
APIから返却された`averageAmount`を
表示および確認用の値として使用する。

同様に、
平均値の算出対象年月についても、
フロントエンドで再計算せず、
APIから返却された`calculatedMonths`を使用する。

これにより、
バックエンドとフロントエンドで
平均値や算出対象年月の
計算結果が異なることを防止する。

算出対象となる
連続した3か月の手取り収入が不足している場合は、
`NET_INCOME_DATA_INSUFFICIENT`を
システムエラーとして扱わず、
平均手取り収入を算出できない
業務上の状態として画面へ表示する。

Phase1では、
必要な手取り収入の入力を促すメッセージを表示し、
手取り収入一覧画面または
登録画面へ遷移できるようにする。

また、
判定対象年月を変更して
平均手取り収入を再取得する場合は、
以前の対象年月に対する取得結果を
現在の結果として表示しない。

新しい取得処理が完了するまでは、
ローディング状態を表示するなど、
表示中の平均値が確定していないことを
利用者が判断できるようにする。

---

## 2. React・TypeScriptでの利用

クエリパラメータの型は、
以下とする。

```ts
export type GetAverageNetIncomeQuery = {
  targetYearMonth: string;
};
```

---

レスポンス型は、以下とする。

```typescript
export type AverageNetIncome = {
  targetYearMonth: string;
  averageAmount: number;
  calculatedMonths: [
    string,
    string,
    string,
  ];
};

export type AverageNetIncomeResponse = {
  data: AverageNetIncome;
};
```

`calculatedMonths` は、平均手取り収入の算出対象となった連続する3か月を、古い対象年月から順に保持する。

API呼び出し例は、以下とする。

```typescript
const response =
  await apiClient.get<AverageNetIncomeResponse>(
    '/api/v1/net-incomes/average',
    {
      params: {
        targetYearMonth: '2026-08',
      },
    },
  );
```

取得結果の例は、以下とする。

```json
{
  "data": {
    "targetYearMonth": "2026-08",
    "averageAmount": 310000,
    "calculatedMonths": [
      "2026-05",
      "2026-06",
      "2026-07"
    ]
  }
}
```

取得した平均手取り収入は、以下の画面表示に利用する。

* 目的達成判定画面
* 判定前の入力値確認
* 平均手取り収入の計算根拠表示

### 2.1 targetYearMonthの扱い

`targetYearMonth` は、API共通方針に従い `YYYY-MM` 形式で送信する。

```typescript
const targetYearMonth = '2026-08';
```

対象年月は、日付ではなく年月を表す業務値として扱う。

JavaScriptの `Date` オブジェクトへ不必要に変換せず、原則として文字列で保持する。

フロントエンドで年月の加減算が必要な場合も、文字列の切り出しや月部分への直接加算は行わない。

### 2.2 averageAmountの扱い

`averageAmount` は、日本円の整数値として扱う。

フロントエンドでは再計算せず、APIが返却した値を表示および目的達成判定画面の確認用に使用する。

```typescript
const formattedAverageAmount =
  new Intl.NumberFormat('ja-JP', {
    style: 'currency',
    currency: 'JPY',
    maximumFractionDigits: 0,
  }).format(response.data.averageAmount);
```

平均値の端数処理は、バックエンドの業務ルールに従う。

フロントエンド独自の切り捨て、切り上げまたは四捨五入は行わない。

### 2.3 calculatedMonthsの扱い

`calculatedMonths` は、平均手取り収入の算出根拠として画面へ表示できる。

表示例：

```text
算出対象：
2026-05、2026-06、2026-07
```

フロントエンドでは、`targetYearMonth` から対象年月を再計算せず、APIが返却した `calculatedMonths` を使用する。

これにより、バックエンドとフロントエンドで対象年月の計算結果がずれることを防止する。

### 2.4 データ不足時の扱い

算出対象となる3か月のデータが不足している場合は、`NET_INCOME_DATA_INSUFFICIENT` が返却される。

フロントエンドでは、システムエラーとして扱わず、平均手取り収入を算出できない業務状態として表示する。

表示例：

```text
平均手取り収入を算出できません。

判定対象年月より前の連続する3か月について、
手取り収入を入力してください。
```

不足している対象年月を画面へ具体的に表示する場合は、将来、エラーレスポンスへ不足年月を含めることを検討する。

Phase1では、共通メッセージを表示し、手取り収入一覧または登録画面へ遷移できるようにする。

### 2.5 ローディング表示

平均手取り収入の取得中は、計算結果が確定していないことを示すローディング表示を行う。

取得中に、直前に取得した別の判定対象年月の平均値を現在の結果として表示しない。

判定対象年月を変更した場合は、以前の取得結果をいったん破棄するか、新しい取得が完了するまで計算中であることを明示する。

### 2.6 エラー表示

入力値に問題がある場合は、`error.details` の `field` を利用して対象年月の入力欄へエラーを表示する。

想定するフィールドは、以下とする。

* `targetYearMonth`

エラーコードごとの基本的な扱いは、以下とする。

| エラーコード                         | フロントエンドの扱い         |
| ------------------------------ | ------------------ |
| `INVALID_TARGET_YEAR_MONTH`    | 対象年月入力欄へ形式エラーを表示する |
| `NET_INCOME_DATA_INSUFFICIENT` | データ不足の案内を表示する      |
| `USER_NOT_FOUND`          | 利用者選択画面へ戻す       |
| `INTERNAL_SERVER_ERROR`        | 共通エラー表示を行う         |

---

## 3. 関連ドキュメント
- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [INC-001 手取り収入一覧取得](./inc-001-list.md)
- [INC-002 手取り収入登録](./inc-002-create.md)
- [INC-003 手取り収入詳細取得](./inc-003-detail.md)
- [INC-004 手取り収入更新](./inc-004-update.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/net-incomes/README.md)
- [Reactアーキテクチャ設計](../../../architecture/react/net-incomes/README.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)