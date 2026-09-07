# HLD-002 保有商品登録

## 1. 概要

本ドキュメントでは、HLD-002 保有商品登録APIをReact・TypeScriptから利用する際のフロントエンドアーキテクチャおよび責務分離方針を定義する。

本機能では、操作対象となる利用者の資産口座に対して、新しい保有商品を登録する。

利用者は、保有商品名、商品種別、所属資産口座、利用開始年月および備考を入力し、登録処理を実行する。

所属資産口座として選択できるのは、操作対象利用者に帰属し、残高記録単位が「商品単位」である有効な資産口座のみとする。

React実装では、画面表示、フォーム状態管理、入力値検証、API通信およびサーバー状態管理の責務を分離し、以下の構成を基本とする。

```text
Page / Component
    ↓
Form
    ↓
Mutation Hook
    ↓
API Client
    ↓
HLD-002
```

リクエストおよびレスポンスはTypeScriptの型として定義し、API契約とフロントエンド実装の整合性を保つ。

登録処理にはTanStack QueryのMutationを利用し、成功時は保有商品一覧など関連するQueryのキャッシュを無効化または再取得し、最新状態を画面へ反映する。

入力形式などフロントエンドで判定可能な内容は登録前に検証する。一方、資産口座の利用状態、残高記録単位、利用開始年月の整合性および商品名の重複など、サーバー側の状態に依存する業務ルールについてはAPIの判定結果を正とする。

APIからバリデーションエラーや業務エラーが返却された場合は、共通エラーレスポンスをフロントエンド用のエラー情報へ変換し、対象項目または画面上の適切な位置へ表示する。

正常終了時は、HLD-002から返却された登録済み保有商品を受け取り、保有商品一覧または詳細画面など、画面設計で定めた遷移先へ反映する。


---

## 2. React・TypeScriptでの利用

リクエスト型の例は、以下とする。

```typescript
export type ProductType =
  | 'INVESTMENT_TRUST'
  | 'DOMESTIC_STOCK'
  | 'FOREIGN_STOCK'
  | 'ETF'
  | 'DEPOSIT'
  | 'BOND'
  | 'OTHER';

export type CreateHoldingAssetRequest = {
  assetAccountId: string;
  name: string;
  productType: ProductType;
  startYearMonth: string;
  memo: string | null;
};
```

レスポンス型の例は、以下とする。

```typescript
export type HoldingAsset = {
  id: string;
  assetAccountId: string;
  assetAccountName: string;
  name: string;
  productType: ProductType;
  startYearMonth: string;
  memo: string | null;
  isEnabled: boolean;
};

export type CreateHoldingAssetResponse = {
  data: HoldingAsset;
};
```

---

## 3. 関連ドキュメント

- [HLD-002 API詳細設計](../../../api/details/holding-assets/hld-002-create.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Reactアーキテクチャ共通設計](../react-architecture.md)
- [保有商品 Reactアーキテクチャ設計](./README.md)
- [HLD-002 テスト設計](../../../tests/holding-assets/hld-002-create.md)
- [保有商品 テスト設計](../../../tests/holding-assets/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
