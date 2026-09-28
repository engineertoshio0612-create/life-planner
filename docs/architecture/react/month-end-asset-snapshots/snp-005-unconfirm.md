# SNP-005 月末資産状況確定解除

## 概要

操作対象となる利用者について、
指定した確定済みの月末資産状況を
未確定状態へ戻す。

月末資産状況を確定解除することで、
対象年月の月末資産残高および
商品別月末評価額を
再度修正できる状態にする。

確定解除できるのは、
操作対象利用者における
最新の確定済み月末資産状況のみとする。

React・TypeScriptでは、
月末資産状況IDを
文字列として扱う。

```ts
export type UnconfirmMonthEndAssetSnapshotParams = {
  snapshotId: string;
};
```

確定解除APIでは、
リクエストボディを送信せず、
対象となる `snapshotId` を
URLへ指定する。

レスポンスでは、
確定解除後の月末資産状況を受け取る。

```ts
export type UnconfirmedMonthEndAssetSnapshot = {
  id: string;
  targetYearMonth: string;
  confirmed: boolean;
};
```

確定解除成功時の
`confirmed` は必ず `false` となる。

フロントエンド側で
`confirmed = false` を生成して送信せず、
APIの成功結果によってのみ
確定状態を画面へ反映する。

詳細取得APIなどで取得した
`confirmed` を利用して、
確定解除ボタンの表示・非表示などの
補助的な画面制御を行ってよい。

ただし、
`confirmed = true` であることだけを理由に
確定解除可能とは判断しない。

確定解除できるのは、
最新の確定済み月末資産状況のみであるため、
確定解除可否の最終判断は
SNP-005 APIへ委ねる。

確定解除実行前には、
対象年月、
現在の確定状態、
確定解除後に未確定となること、
登録済みの残高・評価額が保持されることを
確認できるダイアログを表示してよい。

確定解除処理中は、
確定解除ボタンを非活性化し、
同じ要求の二重送信を抑止する。

ただし、
最終的な同時実行制御は、
バックエンド側の
トランザクションおよび
排他制御で保証する。

確定解除成功後は、
以下の状態を最新化する。

- 月末資産状況詳細
- 月末資産状況一覧
- 確定状態に依存する画面表示

TanStack Query等を使用する場合は、
SNP-001の一覧クエリおよび
SNP-003の詳細クエリをinvalidateし、
最新の `confirmed` 状態を再取得する。

確定解除後も、
既存の月末資産残高および
商品別月末評価額は保持される。

そのため、
フロントエンド側でも
確定解除成功を理由として
登録済みデータを初期化しない。

必要に応じて、
月末資産残高または
商品別月末評価額の
一覧・編集画面へ遷移する。

すでに未確定の場合は、

`MONTH_END_ASSET_SNAPSHOT_ALREADY_UNCONFIRMED`

を受け取り、
最新状態を再取得したうえで、
すでに未確定であることを表示する。

確定解除対象が
最新の確定済み月末資産状況ではない場合は、

`MONTH_END_ASSET_SNAPSHOT_UNCONFIRM_ORDER_INVALID`

を受け取り、
先に新しい対象年月を
確定解除する必要があることを表示する。

確定解除順序については、
フロントエンド側で
バックエンドの業務ルールを
完全に再実装しない。

必要に応じて
月末資産状況一覧画面へ戻し、
最新の確定済み対象年月を
確認できるようにする。

また、
確定解除しても
保存済みの目的達成判定履歴は変更しない。

フロントエンドでも、
確定解除成功を契機として
判定履歴の削除・再計算は行わず、
判定実行時点の履歴として扱う。

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
export type UnconfirmMonthEndAssetSnapshotParams = {
  snapshotId: string;
};
```

レスポンス型は、
以下とする。

```ts
export type UnconfirmedMonthEndAssetSnapshot = {
  id: string;
  targetYearMonth: string;
  confirmed: boolean;
};

export type UnconfirmMonthEndAssetSnapshotResponse = {
  data: UnconfirmedMonthEndAssetSnapshot;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.post<UnconfirmMonthEndAssetSnapshotResponse>(
    `/api/v1/month-end-asset-snapshots/${snapshotId}/unconfirm`,
  );
```

本APIでは、
リクエストボディを送信しない。

確定解除成功後は、
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

確定解除成功時の
`confirmed`は
`false`となる。

```ts
if (!response.data.confirmed) {
  // 未確定
}
```

フロントエンドでは、
確定解除成功後に
確定済み状態の表示を
未確定へ更新する。

ただし、
フロントエンド側で

```ts
confirmed: false
```

を生成して
サーバーへ送信してはならない。

確定状態の変更は、
本APIの成功結果によってのみ反映する。

---

### 2.3 確定解除ボタンの表示制御

月末資産状況詳細取得APIなどから取得した
`confirmed`を利用して、
補助的な画面制御を行ってよい。

例えば、
未確定の場合は
確定解除ボタンを表示しない。

```ts
const canShowUnconfirmButton =
  snapshot.confirmed;
```

ただし、
`confirmed = true`であることだけを理由に
確定解除可能と判断しない。

確定解除できるのは、
操作対象利用者における
最新の確定済み月末資産状況のみである。

そのため、
確定解除可否の最終判断は
SNP-005で行う。

---

### 2.4 確定解除確認ダイアログ

確定解除は、
正式な月末資産状況を
再び編集可能な状態へ戻す操作である。

そのため、
実行前に確認ダイアログを表示してよい。

表示例：

```text
2026年8月の月末資産状況の
確定を解除します。

確定解除後は、
この対象年月の資産状況は
正式な現在資産状況および
新しい目的達成判定には利用されません。

確定を解除しますか？
```

確認画面では、
少なくとも以下を表示する。

- 対象年月
- 現在の確定状態
- 確定解除後は未確定となること
- 登録済みの残高・評価額は削除されないこと

---

### 2.5 確定解除処理中の画面制御

確定解除処理中は、
確定解除ボタンを非活性化する。

```ts
const [isUnconfirming, setIsUnconfirming] =
  useState(false);
```

処理中に
同じ確定解除要求を
再送信できないようにする。

これにより、
意図しない二重送信を抑止する。

ただし、
最終的な同時実行制御は
バックエンド側の
トランザクションおよび
排他制御で保証する。

---

### 2.6 確定解除成功時の扱い

確定解除成功後は、
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

### 2.7 確定解除後の編集

確定解除成功後は、
対象となる月末資産状況が
未確定となる。

そのため、
必要に応じて以下の画面へ遷移できる。

- 月末資産残高一覧・編集画面
- 商品別月末評価額一覧・編集画面

確定解除時点では、
既存の月末資産残高および
商品別月末評価額は保持される。

フロントエンドでは、
確定解除成功後に
登録済みデータを初期化しない。

---

### 2.8 未確定エラーの扱い

すでに未確定の
月末資産状況を確定解除しようとした場合は、

`MONTH_END_ASSET_SNAPSHOT_ALREADY_UNCONFIRMED`

が返却される。

フロントエンドでは、
最新状態を再取得したうえで、
すでに未確定であることを表示する。

表示例：

```text
この月末資産状況は
すでに未確定です。
```

---

### 2.9 確定解除順序エラーの扱い

確定解除対象が
最新の確定済み月末資産状況ではない場合は、

`MONTH_END_ASSET_SNAPSHOT_UNCONFIRM_ORDER_INVALID`

が返却される。

表示例：

```text
この月末資産状況は
確定解除できません。

より新しい確定済みの
月末資産状況を先に確定解除してください。
```

フロントエンド側で
確定解除順序の業務ルールを
完全に再実装しない。

必要に応じて
一覧画面へ戻り、
最新の確定済み対象年月を
確認できるようにする。

---

### 2.10 目的達成判定履歴の扱い

確定解除しても、
保存済みの目的達成判定履歴は
変更されない。

フロントエンドでも、
確定解除成功を契機として
過去の判定履歴を
削除または再計算しない。

判定履歴は、
判定実行時点の情報として
そのまま表示する。

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
| `MONTH_END_ASSET_SNAPSHOT_ALREADY_UNCONFIRMED` | 未確定であることを表示して再取得する |
| `MONTH_END_ASSET_SNAPSHOT_UNCONFIRM_ORDER_INVALID` | 確定解除順序エラーを表示する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

---

## 3. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [SNP-001 月末資産状況一覧取得](./snp-001-list.md)
- [SNP-002 月末資産状況作成](./snp-002-create.md)
- [SNP-003 月末資産状況詳細取得](./snp-003-detail.md)
- [SNP-004 月末資産状況確定](./snp-004-confirm.md)
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