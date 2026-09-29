# 商品別月末評価額 Reactアーキテクチャ設計

## 1. 概要

本ディレクトリでは、
商品別月末評価額に関する
React・TypeScript実装方針を定義する。

商品別月末評価額は、
残高記録単位が商品単位である資産口座について、
保有商品ごとの月末時点の評価額を扱う。

React側では、
指定した `snapshotId` を基準として一覧を取得し、
登録状態に応じて登録・更新APIを使い分ける。

口座単位で残高を記録する資産口座については、
VAL系APIではなくBAL系APIを使用する。

---

## 2. API一覧

| ID | 機能 | Reactでの扱い |
|---|---|---|
| VAL-001 | 商品別月末評価額一覧取得 | Query |
| VAL-002 | 商品別月末評価額登録 | Mutation |
| VAL-003 | 商品別月末評価額更新 | Mutation |

商品別月末評価額が未登録の場合はVAL-002、
登録済みの場合はVAL-003を使用する。

`value = 0`は登録済みとして扱い、
未登録状態と区別する。

---

## 3. 共通実装方針

- `snapshotId`、`holdingAssetId`などのIDは`string`として扱う
- `value`は日本円の整数値として扱い、`0`を有効値とする
- 0円と未入力を明確に区別する
- `X-User-Id`は共通API Clientから付与する
- 利用者境界や対象年月時点の利用可否などの最終判定はバックエンドへ委ねる
- 確定状態は編集UIの制御に利用できるが、登録・更新可否の最終保証はバックエンドで行う
- VAL-002・VAL-003成功後はVAL-001をinvalidateし、最新状態を再取得することを基本とする
- API Client、Query / Mutation Hook、Page、Componentの責務を分離する
- エラー処理はメッセージ文字列ではなくエラーコードを基準とする

---

## 4. 関連ドキュメント

- [VAL-001 商品別月末評価額一覧取得](./val-001-list.md)
- [VAL-002 商品別月末評価額登録](./val-002-create.md)
- [VAL-003 商品別月末評価額更新](./val-003-update.md)
- [Reactアーキテクチャ共通設計](../react-architecture.md)
- [Laravelアーキテクチャ設計](../../laravel/holding-values/README.md)
- [API共通方針](../../api-common-policy.md)