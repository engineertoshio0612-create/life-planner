# HLD-004 保有商品更新

## 1. 概要

本ドキュメントでは、HLD-004 保有商品更新APIをReact・TypeScriptから利用する際のフロントエンドアーキテクチャおよび責務分離方針を定義する。

本機能では、操作対象となる利用者に帰属する保有商品の更新可能な情報を変更する。

更新対象は、以下とする。

* 保有商品名
* 商品種別
* 備考

所属資産口座、利用開始年月および利用状態は編集対象としない。

更新画面では、HLD-003 保有商品詳細取得APIから取得した保有商品情報をフォームの初期値として使用し、利用者が変更した項目のみをHLD-004へ送信する。

React実装では、画面表示、フォーム状態管理、入力値検証、変更差分の抽出、API通信およびサーバー状態管理の責務を分離し、以下の構成を基本とする。

```text
Page / Component
    ↓
HLD-003 Query
    ↓
Form
    ↓
変更差分の抽出
    ↓
Mutation Hook
    ↓
API Client
    ↓
HLD-004
```

リクエストおよびレスポンスはTypeScriptの型として定義し、API契約とフロントエンド実装の整合性を保つ。

更新リクエストには変更された項目のみを含め、備考を未設定へ変更する場合は`memo: null`を明示的に送信する。

所属資産口座、利用開始年月および利用状態は編集不可として扱い、利用状態の変更にはHLD-005 保有商品無効化APIを使用する。

正常終了時は、HLD-004から返却された更新後の保有商品を画面へ反映し、必要に応じて保有商品一覧および詳細情報に関するQueryキャッシュを最新状態へ更新する。

APIからバリデーションエラー、重複エラーまたはその他の業務エラーが返却された場合は、共通エラーレスポンスに従って、対象項目または画面上の適切な位置へエラー内容を表示する。

---

## 2. React・TypeScriptでの利用

更新リクエスト型の例は、以下とする。

```typescript
export type ProductType =
  | 'INVESTMENT_TRUST'
  | 'DOMESTIC_STOCK'
  | 'FOREIGN_STOCK'
  | 'ETF'
  | 'DEPOSIT'
  | 'BOND'
  | 'OTHER';

export type UpdateHoldingAssetRequest = {
  name?: string;
  productType?: ProductType;
  memo?: string | null;
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

export type UpdateHoldingAssetResponse = {
  data: HoldingAsset;
};
```

更新画面では、HLD-003 保有商品詳細取得APIのレスポンスを初期値として使用する。

利用者が変更した項目のみを更新リクエストへ含める。

例えば、備考のみを変更する場合は、以下のリクエストを送信する。

```json
{
  "memo": "課税口座からNISA口座へ変更予定"
}
```

備考を未設定へ戻す場合は、`null` を送信する。

```json
{
  "memo": null
}
```

フロントエンドでは、以下の項目を編集不可として表示する。

- 所属資産口座
- 利用開始年月
- 利用状態

所属資産口座の変更が必要な場合は、現在の保有商品を無効化した上で、新しい保有商品を登録するよう案内する。

利用状態の変更は、HLD-005 保有商品無効化APIを使用する。

---

## 3. 関連ドキュメント

- [HLD-004 API詳細設計](../../../api/details/holding-assets/hld-004-update.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Reactアーキテクチャ共通設計](../react-architecture.md)
- [保有商品 Reactアーキテクチャ設計](./README.md)
- [HLD-004 テスト設計](../../../tests/holding-assets/hld-004-update.md)
- [保有商品 テスト設計](../../../tests/holding-assets/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
