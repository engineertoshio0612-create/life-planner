# 資産口座 テスト設計

## 1. 概要

本ディレクトリでは、資産口座機能に関するAPIのテスト設計を管理する。

各ドキュメントでは、API詳細設計およびLaravelアーキテクチャ設計を前提として、正常系・異常系、利用者境界、入力値、レスポンス契約、データ整合性、副作用などの主要なテスト観点を定義する。

API固有の詳細なテストケースは各ドキュメントで定義し、本READMEでは資産口座機能全体の構成のみを示す。

---

## 2. 対象API

| API ID  | API名         | テスト設計                                                                          |
| ------- | ------------ | ------------------------------------------------------------------------------ |
| ACC-001 | 資産口座一覧取得     | [acc-001-list.md](./acc-001-list.md)                                           |
| ACC-002 | 資産口座登録       | [acc-002-create.md](./acc-002-create.md)                                       |
| ACC-003 | 資産口座詳細取得     | [acc-003-detail.md](./acc-003-detail.md)                                       |
| ACC-004 | 資産口座更新       | [acc-004-update.md](./acc-004-update.md)                                       |
| ACC-005 | 利用可能資産設定履歴取得 | [acc-005-available-setting-history.md](./acc-005-available-setting-history.md) |
| ACC-006 | 利用可能資産設定登録   | [acc-006-create-available-setting.md](./acc-006-create-available-setting.md)   |

---

## 3. 共通テスト方針

資産口座APIでは、主に以下を確認する。

* `X-User-Id`による利用者境界が維持されること
* 他利用者の資産口座を参照・更新できないこと
* 論理削除を各APIの仕様どおりに扱うこと
* 利用可能資産設定の履歴整合性が維持されること
* DB内部情報をAPIレスポンスへ公開しないこと
* 更新APIでトランザクションおよびDB制約が機能すること
* APIで定義された範囲以外の業務データへ副作用が発生しないこと

共通的なテスト実装方針は、バックエンド・フロントエンド・E2Eの各テスト方針に従う。

---

## 4. 関連ドキュメント

- [API一覧](../../api/api-list.md)
- [API共通方針](../../api/api-common-policy.md)
- [エラーコード一覧](../../api/error-codes.md)
- [資産口座 Laravelアーキテクチャ設計](../../architecture/laravel/asset-accounts/README.md)
- [資産口座 Reactアーキテクチャ設計](../../architecture/react/asset-accounts/README.md)
- [機能要件](../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../glossary.md)
- [エンティティ定義](../../entities.md)
- [テーブル定義書](../../table-definition.md)
- [ER図](../../er-diagram-phase1.md)
