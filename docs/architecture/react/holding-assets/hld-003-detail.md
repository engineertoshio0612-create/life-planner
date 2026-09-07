# HLD-003 保有商品詳細取得

## 1. 概要

操作対象となる利用者に帰属する
保有商品の詳細情報を取得する。

本APIは、
保有商品の編集画面表示および
詳細情報の確認に利用する。

---

## 2. React・TypeScriptでの利用

レスポンス型の例は、以下とする。

```typescript
export type ProductType =
  | 'INVESTMENT_TRUST'
  | 'DOMESTIC_STOCK'
  | 'FOREIGN_STOCK'
  | 'ETF'
  | 'DEPOSIT'
  | 'BOND'
  | 'OTHER';

export type HoldingAssetDetail = {
  id: string;
  assetAccountId: string;
  assetAccountName: string;
  name: string;
  productType: ProductType;
  startYearMonth: string;
  memo: string | null;
  isEnabled: boolean;
};

export type HoldingAssetDetailResponse = {
  data: HoldingAssetDetail;
};
```

取得結果は、保有商品詳細画面および保有商品編集画面の初期表示に利用する。

所属資産口座名は、追加の資産口座取得APIを呼び出さずに画面へ表示するために使用する。

---

## 27. 関連ドキュメント

- [API共通方針](../../api-common-policy.md)
- [API一覧](../../api-list.md)
- [エラーコード一覧](../../error-codes.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)