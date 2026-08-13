# 資産口座API詳細設計

## 1. 概要

本書では、
資産口座APIに関する詳細仕様を定義する。

資産口座APIでは、
操作対象となる利用者に帰属する資産口座の
取得、登録、更新および利用可能資産設定の管理を行う。

---

## 2. 対象API

| API ID | API名 | HTTPメソッド | URL |
|---|---|---|---|
| ACC-001 | 資産口座一覧取得 | GET | `/api/v1/asset-accounts` |
| ACC-002 | 資産口座登録 | POST | `/api/v1/asset-accounts` |
| ACC-003 | 資産口座詳細取得 | GET | `/api/v1/asset-accounts/{assetAccountId}` |
| ACC-004 | 資産口座更新 | PATCH | `/api/v1/asset-accounts/{assetAccountId}` |
| ACC-005 | 利用可能資産設定履歴取得 | GET | `/api/v1/asset-accounts/{assetAccountId}/available-settings` |
| ACC-006 | 利用可能資産設定登録 | POST | `/api/v1/asset-accounts/{assetAccountId}/available-settings` |

---



---



---



---


