# ACC-001 資産口座一覧取得

## 1. 概要

本ドキュメントでは、
ACC-001 資産口座一覧取得APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

ACC-001では、
操作対象となる利用者に帰属する資産口座の一覧を取得し、
利用中・無効化済みを含む資産口座情報を画面表示に利用する。

フロントエンドでは、
APIレスポンスをTypeScriptの型として定義し、
資産口座の基本情報に加えて、

* 現在利用可能な資産口座であるか
* 目的達成判定に利用可能な資産であるか

を判定できるようにする。

`isEnabled`は、
資産口座の利用中・無効化済みの表示制御に利用する。

`isAvailable`は、
目的達成判定への利用可否を示す表示に利用する。

API通信、エラーハンドリング、
TanStack Queryの利用方針などの共通事項については、
Reactアーキテクチャ共通設計に従うものとし、
本ドキュメントではACC-001固有の利用方法のみを定義する。


---

## 2. React・TypeScriptでの利用

レスポンス型の例は、
以下とする。

```ts
export type AssetType =
  | 'CASH'
  | 'BANK'
  | 'SECURITIES'
  | 'IDECO'
  | 'CORPORATE_DC'
  | 'OTHER';

export type BalanceRecordingUnit =
  | 'ACCOUNT'
  | 'HOLDING';

export type AssetAccountListItem = {
  id: string;
  name: string;
  assetType: AssetType;
  balanceRecordingUnit: BalanceRecordingUnit;
  isAvailable: boolean;
  startYearMonth: string;
  isEnabled: boolean;
};

export type AssetAccountListResponse = {
  data: AssetAccountListItem[];
};
```

フロントエンドは、
`isEnabled`を利用して
利用中と無効化済みの表示を切り替える。

`isAvailable`は、
目的達成判定へ利用する資産かどうかを示す表示に利用する。

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [ACC-002 資産口座登録](./acc-002-create.md)
- [ACC-003 資産口座詳細取得](./acc-003-detail.md)
- [ACC-004 資産口座更新](./acc-004-update.md)
- [ACC-005 利用可能資産設定履歴取得](./acc-005-available-setting-history.md)
- [ACC-006 利用可能資産設定登録](./acc-006-create-available-setting.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/asset-accounts.md)
- [Reactアーキテクチャ設計](../../../architecture/react/asset-accounts.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)
