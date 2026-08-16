# 資産口座 Laravelアーキテクチャ設計

## 1. 概要

本ディレクトリでは、資産口座APIをLaravelで実装する際のアーキテクチャ設計を管理する。

Laravel全体で共通する責務分離、利用者コンテキスト、エラーハンドリング、トランザクションなどの方針は、[Laravelアーキテクチャ共通設計](../laravel-architecture.md)で定義する。

本ディレクトリでは、それらの共通方針を前提として、ACC-001〜ACC-006固有の実装方針を各ドキュメントで定義する。

資産口座では、資産口座の基本情報と、目的達成判定で利用する資産として扱うかを示す利用可能資産設定を分離して管理する。

```text
asset_accounts
    ↓
資産口座の基本情報

asset_account_available_settings
    ↓
利用可能資産区分の履歴
```

利用可能資産区分は資産口座の固定属性とせず、`asset_account_available_settings`によって年月単位の履歴として管理する。

---

## 2. API別アーキテクチャ設計

| API ID | API名 | ドキュメント |
|---|---|---|
| ACC-001 | 資産口座一覧取得 | acc-001-list.md |
| ACC-002 | 資産口座登録 | acc-002-create.md |
| ACC-003 | 資産口座詳細取得 | acc-003-detail.md |
| ACC-004 | 資産口座更新 | acc-004-update.md |
| ACC-005 | 利用可能資産設定履歴取得 | acc-005-available-setting-history.md |
| ACC-006 | 利用可能資産設定登録 | acc-006-create-available-setting.md |

各API固有の処理フロー、UseCase、Query / Repository、トランザクション、ロックなどの詳細は、それぞれのドキュメントで定義する。

---

## 3. 関連ドキュメント

- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Reactアーキテクチャ共通設計](../react-architecture.md)
- [資産口座 Reactアーキテクチャ設計](./README.md)
- [資産口座 テスト設計](../../../tests/asset-accounts/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../requirements/glossary.md)
- [エンティティ定義](../../../requirements/entities.md)
- [テーブル定義書](../../../database/table-definition.md)
- [ER図](../../../database/er-diagram-phase1.md)