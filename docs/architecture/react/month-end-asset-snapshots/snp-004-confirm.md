# SNP-004 月末資産状況確定

## 概要

操作対象となる利用者について、
指定した未確定の月末資産状況を
確定する。

月末資産状況を確定することで、
対象年月の資産状況を
正式な月末資産状況として扱える状態にする。

React・TypeScriptでは、
月末資産状況IDを
文字列として扱う。

```ts
export type ConfirmMonthEndAssetSnapshotParams = {
  snapshotId: string;
};
```

確定APIでは、
リクエストボディを送信せず、
対象となる `snapshotId` を
URLへ指定する。

レスポンスでは、
確定後の月末資産状況を受け取る。

```ts
export type ConfirmedMonthEndAssetSnapshot = {
  id: string;
  targetYearMonth: string;
  confirmed: boolean;
};
```

確定成功時の
`confirmed` は必ず `true` となる。

フロントエンド側で
`confirmed = true` を生成して送信せず、
APIの成功結果によってのみ
確定状態を画面へ反映する。

詳細取得APIなどで取得した
`confirmed` を利用して、
確定ボタンの表示・非表示などの
補助的な画面制御を行ってよい。

ただし、
`confirmed = false` であることだけを理由に
確定可能とは判断しない。

確定には、
以下のようなバックエンド側の
業務ルールが存在する。

- 必要な月末資産残高が登録されていること
- 必要な商品別月末評価額が登録されていること
- 確定順序を満たしていること
- 対象が未確定であること

そのため、
確定可否の最終判断は
SNP-004 APIへ委ねる。

確定実行前には、
対象年月や現在状態を表示した
確認ダイアログを表示してよい。

また、
確定処理中は
確定ボタンを非活性化し、
同じ要求の二重送信を抑止する。

ただし、
最終的な同時実行制御は
バックエンド側の
トランザクションおよび
排他制御で保証する。

確定成功後は、
以下の状態を最新化する。

- 月末資産状況詳細
- 月末資産状況一覧
- 確定状態に依存する画面表示

TanStack Query等を使用する場合は、
SNP-001の一覧クエリおよび
SNP-003の詳細クエリをinvalidateし、
最新の `confirmed` 状態を再取得する。

業務エラーが返却された場合は、
システムエラーとして一律に扱わず、
エラーコードに応じた案内を表示する。

主な業務エラーは以下とする。

- `MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED`
  - すでに確定済みであることを表示する
- `MONTH_END_ASSET_BALANCE_INCOMPLETE`
  - 月末資産残高の入力不足を表示する
- `MONTH_END_HOLDING_VALUE_INCOMPLETE`
  - 商品別月末評価額の入力不足を表示する
- `MONTH_END_ASSET_SNAPSHOT_CONFIRM_ORDER_INVALID`
  - 先に確定すべき月末資産状況があることを表示する

月末資産残高または
商品別月末評価額が不足している場合は、
必要に応じて
各入力画面への導線を表示する。

確定順序については、
フロントエンド側で
バックエンドの業務ルールを
完全に再実装しない。

バックエンドから返却された
判定結果を使用し、
必要に応じて
月末資産状況一覧画面へ誘導する。

`MONTH_END_ASSET_SNAPSHOT_NOT_FOUND`
の場合は、
月末資産状況一覧画面へ戻すなどして、
対象が存在しないことを案内する。

その他の利用者コンテキストエラーや
サーバーエラーについては、
API共通方針に従って
利用者選択への誘導または
共通エラー表示を行う。

---

## 2. React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type ConfirmMonthEndAssetSnapshotParams = {
  snapshotId: string;
};
```

レスポンス型は、
以下とする。

```ts
export type ConfirmedMonthEndAssetSnapshot = {
  id: string;
  targetYearMonth: string;
  confirmed: boolean;
};

export type ConfirmMonthEndAssetSnapshotResponse = {
  data: ConfirmedMonthEndAssetSnapshot;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.post<ConfirmMonthEndAssetSnapshotResponse>(
    `/api/v1/month-end-asset-snapshots/${snapshotId}/confirm`,
  );
```

本APIでは、
リクエストボディを送信しない。

確定成功後は、
返却された月末資産状況をもとに
画面上の確定状態を更新する。

---

### 2.1 snapshotIdの扱い

`snapshotId`は、
API共通方針に従って
文字列として扱う。

```ts
const snapshotId: string = '12';
```

フロントエンド側で
数値へ変換して計算には使用しない。

URL生成時も、
文字列のまま使用する。

---

### 2.2 confirmedの扱い

確定成功時の
`confirmed`は
`true`となる。

```ts
if (response.data.confirmed) {
  // 確定済み
}
```

フロントエンドでは、
確定成功後に
未確定状態の表示を
確定済みへ更新する。

ただし、
フロントエンド側で
`confirmed = true`を生成して
サーバーへ送信してはならない。

確定状態の変更は、
本APIの成功結果によってのみ反映する。

---

### 2.3 確定ボタンの表示制御

月末資産状況詳細取得APIなどから取得した
`confirmed`を利用して、
補助的な画面制御を行ってよい。

例えば、
`confirmed = true`の場合は、
確定ボタンを表示しない。

```ts
const canShowConfirmButton =
  !snapshot.confirmed;
```

ただし、
`confirmed = false`であることだけを理由に
確定可能と判断しない。

確定には、
以下のようなバックエンド側の
業務ルールが存在する。

- 必要な月末資産残高が登録されている
- 必要な商品別月末評価額が登録されている
- 確定順序を満たしている

そのため、
確定可否の最終判断は
SNP-004で行う。

---

### 2.4 確定確認ダイアログ

確定操作は、
対象年月の資産状況を
正式な状態へ変更する操作である。

そのため、
実行前に確認ダイアログを表示してよい。

表示例：

```text
2026年8月の月末資産状況を確定します。

確定後に内容を修正する場合は、
先に確定解除が必要です。

確定しますか？
```

確認画面では、
少なくとも以下を表示する。

- 対象年月
- 現在の確定状態
- 確定後は直接編集できないこと

---

### 2.5 確定処理中の画面制御

確定処理中は、
確定ボタンを非活性化する。

```ts
const [isConfirming, setIsConfirming] =
  useState(false);
```

処理中に
同じ確定要求を
再送信できないようにする。

これにより、
意図しない二重送信を抑止する。

ただし、
最終的な同時実行制御は
バックエンド側の
トランザクションおよび
排他制御で保証する。

---

### 2.6 確定成功時の扱い

確定成功後は、
以下の状態を更新する。

- 月末資産状況詳細
- 月末資産状況一覧
- 確定状態に依存する画面表示

React Query等を使用する場合は、
対象となる詳細クエリおよび
一覧クエリをinvalidateしてよい。

例えば、
以下を再取得する。

```text
SNP-001 月末資産状況一覧取得
SNP-003 月末資産状況詳細取得
```

これにより、
画面上の`confirmed`を
最新状態へ反映する。

---

### 2.7 確定済みエラーの扱い

すでに確定済みの
月末資産状況を確定しようとした場合は、

`MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED`

が返却される。

フロントエンドでは、
最新状態を再取得したうえで、
確定済みであることを表示する。

表示例：

```text
この月末資産状況は
すでに確定されています。
```

---

### 2.8 月末資産残高不足の扱い

必要な月末資産残高が
登録されていない場合は、

`MONTH_END_ASSET_BALANCE_INCOMPLETE`

が返却される。

フロントエンドでは、
システムエラーとして扱わず、
入力不足の業務状態として表示する。

表示例：

```text
確定に必要な月末資産残高が
すべて登録されていません。
```

必要に応じて、
BAL-001 月末資産残高一覧画面や
残高登録画面への導線を表示する。

---

### 2.9 商品別月末評価額不足の扱い

必要な商品別月末評価額が
登録されていない場合は、

`MONTH_END_HOLDING_VALUE_INCOMPLETE`

が返却される。

フロントエンドでは、
入力不足の業務状態として表示する。

表示例：

```text
確定に必要な商品別月末評価額が
すべて登録されていません。
```

必要に応じて、
VAL-001 商品別月末評価額一覧画面や
評価額登録画面への導線を表示する。

---

### 2.10 確定順序エラーの扱い

先に確定する必要がある
月末資産状況が存在する場合は、

`MONTH_END_ASSET_SNAPSHOT_CONFIRM_ORDER_INVALID`

が返却される。

表示例：

```text
先に確定する必要がある
月末資産状況があります。
```

フロントエンド側で
確定順序を独自に完全再現せず、
バックエンドの判定結果を使用する。

必要に応じて、
月末資産状況一覧画面へ戻り、
先に処理すべき対象年月を
確認できるようにする。

---

### 2.11 エラー表示

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 不正なURLまたはパラメータとして扱う |
| `MONTH_END_ASSET_SNAPSHOT_NOT_FOUND` | 月末資産状況一覧画面へ戻す |
| `MONTH_END_ASSET_SNAPSHOT_ALREADY_CONFIRMED` | 確定済みであることを表示して再取得する |
| `MONTH_END_ASSET_BALANCE_INCOMPLETE` | 月末資産残高の入力不足を表示する |
| `MONTH_END_HOLDING_VALUE_INCOMPLETE` | 商品別月末評価額の入力不足を表示する |
| `MONTH_END_ASSET_SNAPSHOT_CONFIRM_ORDER_INVALID` | 確定順序エラーを表示する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [SNP-001 月末資産状況一覧取得](./snp-001-list.md)
- [SNP-002 月末資産状況作成](./snp-002-create.md)
- [SNP-003 月末資産状況詳細取得](./snp-003-detail.md)
- [SNP-005 月末資産状況確定解除](./snp-005-unconfirm.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/month-end-asset-snapshots/README.md)
- [Reactアーキテクチャ設計](../../../architecture/react/month-end-asset-snapshots/README.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)