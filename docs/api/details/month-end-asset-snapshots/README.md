# 月末資産状況API

## 1. 概要

月末資産状況に関するAPIの詳細設計を管理する。

月末資産状況APIでは、主に以下を扱う。

- 月末資産状況一覧取得
- 月末資産状況作成
- 月末資産状況詳細取得
- 月末資産状況確定
- 月末資産状況確定解除

各API固有の仕様は、
それぞれのAPI詳細設計を参照する。

API全体に共通する仕様については、
[API共通方針](../../api-common-policy.md)
を参照する。

---

## 2. API一覧

| API ID | API名 | HTTPメソッド | エンドポイント | 詳細設計 |
|---|---|---|---|---|
| SNP-001 | 月末資産残高一覧取得 | GET | `/api/v1/month-end-asset-snapshots` | [SNP-001](./snp-001-list.md) |
| SNP-002 | 月末資産残高登録 | POST | `/api/v1/month-end-asset-snapshots` | [SNP-002](./snp-002-create.md) |
| SNP-003 | 月末資産状況詳細取得 | GET | `/api/v1/month-end-asset-snapshots/{snapshotId}` | [SNP-003](./snp-003-detail.md) |
| SNP-004 | 月末資産状況確定 | POST | `/api/v1/month-end-asset-snapshots/{snapshotId}/confirm` | [SNP-004](./snp-004-confirm.md) |
| SNP-005 | 月末資産状況確定解除 | POST | `/api/v1/month-end-asset-snapshots/{snapshotId}/unconfirm` | [SNP-005](./snp-005-unconfirm.md) |


---

## 3. APIの責務

### SNP-001 月末資産状況一覧取得

操作対象利用者の対象年月ごとの確定状態、登録済み件数、未登録件数などを取得する。

### SNP-002 月末資産状況作成

指定した対象年月の未確定な月末資産状況を作成する。

### SNP-003 月末資産状況詳細取得

月末資産状況、登録済み残高、未登録資産などを取得する。

### SNP-004 月末資産状況確定

確定条件を検証し、対象年月の月末資産状況を確定する。

### SNP-005 月末資産状況確定解除

最新の確定済み月末資産状況を未確定へ戻す。

---

## 4. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [Laravel共通設計](../../../architecture/laravel-architecture.md)
- [React共通設計](../../../architecture/react-architecture.md)