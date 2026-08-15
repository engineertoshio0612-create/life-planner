##  ASM-001 目的達成判定実行

### 1 概要

操作対象となる利用者について、
指定された目的に対する
目的達成判定を実行する。

目的達成判定では、
指定された目的、
確定済みの月末資産状況、
月末資産残高、
商品別月末評価額、
手取り収入など、
判定に必要な情報をもとに
目的を達成可能か判定する。

判定結果は、
`assessment_histories`
へ目的達成判定履歴として保存する。

判定に必要な情報が不足している場合は、
エラーとはせず、
判定結果を
「判定不可」として保存する。

本APIは、
目的そのものを更新しない。

また、
月末資産状況、
月末資産残高、
商品別月末評価額、
手取り収入などの
判定元データも変更しない。

---

### 28 React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type ExecuteAssessmentParams = {
  objectiveId: string;
};
```

本APIでは、
リクエストボディを使用しない。

レスポンス型は、
以下とする。

```ts
export type AssessmentResult =
  | 'achievable'
  | 'difficult'
  | 'unassessable';

export type ExecutedAssessment = {
  id: string;
  objectiveId: string;
  result: AssessmentResult;
};

export type ExecuteAssessmentResponse = {
  data: ExecutedAssessment;
};
```

実際の`result`の値は、
バックエンド側で定義する
EnumおよびAPI仕様に合わせる。

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.post<ExecuteAssessmentResponse>(
    `/api/v1/objectives/${objectiveId}/assessments`,
  );
```

目的詳細画面などから、
利用者が明示的に
目的達成判定を実行する際に使用する。

---

#### 28.1 objectiveIdの扱い

`objectiveId`は、
API共通方針に従って
stringとして扱う。

```ts
const objectiveId: string = '5';
```

フロントエンド側で
numberへ変換して
業務計算には使用しない。

URL生成時も、
stringのまま使用する。

---

#### 28.2 リクエストボディ

本APIでは、
リクエストボディを送信しない。

以下のような
判定材料を
フロントエンドから送信してはならない。

```ts
{
  currentAssets: 3000000,
  averageNetIncome: 350000,
  targetAmount: 5000000,
}
```

目的達成判定に使用する値は、
バックエンドで
登録済みの業務データから取得する。

これにより、
画面上の一時的な値と
サーバー上の正式なデータが
混在することを防止する。

---

#### 28.3 判定実行ボタン

目的詳細画面などに、
目的達成判定を実行する
ボタンを配置する。

例：

```tsx
<button
  type="button"
  onClick={handleExecuteAssessment}
>
  目的達成判定を実行
</button>
```

判定実行は、
画面表示時に自動実行せず、
利用者による
明示的な操作を基本とする。

ASM-001は実行のたびに
新しい判定履歴を作成するため、
画面表示や再レンダリングを契機として
自動的に呼び出してはならない。

---

#### 28.4 Mutationとして扱う

ASM-001は、
目的達成判定履歴を
新規登録するため、
React Query等では
QueryではなくMutationとして扱う。

概念例：

```ts
export const executeAssessment =
  async (
    objectiveId: string,
  ): Promise<ExecuteAssessmentResponse> => {
    const response =
      await apiClient.post<ExecuteAssessmentResponse>(
        `/api/v1/objectives/${objectiveId}/assessments`,
      );

    return response;
  };
```

Hookの概念例：

```ts
export const useExecuteAssessment =
  () => {
    return useMutation({
      mutationFn:
        ({
          objectiveId,
        }: ExecuteAssessmentParams) =>
          executeAssessment(objectiveId),
    });
  };
```

---

#### 28.5 実行中の画面制御

ASM-001は冪等ではないため、
利用者による
意図しない連続実行を
抑止する。

判定処理中は、
判定実行ボタンを非活性化する。

```tsx
const mutation =
  useExecuteAssessment();

<button
  type="button"
  disabled={mutation.isPending}
  onClick={() =>
    mutation.mutate({
      objectiveId,
    })
  }
>
  {mutation.isPending
    ? '判定中...'
    : '目的達成判定を実行'}
</button>
```

ただし、
バックエンド仕様としては、
複数回実行された場合に
実行回数分の
目的達成判定履歴が作成される。

フロントエンドの非活性化は、
UX上の二重操作防止とする。

---

#### 28.6 達成可能の表示

`result = 'achievable'`
の場合は、
目的を達成可能と判定されたことを
表示する。

概念例：

```ts
if (
  response.data.result
  === 'achievable'
) {
  // 達成可能表示
}
```

表示例：

```text
現在の資産状況では、
目的を達成できる見込みがあります。
```

具体的な文言は、
画面設計に従う。

---

#### 28.7 達成困難の表示

`result = 'difficult'`
の場合は、
目的達成が困難と判定されたことを
表示する。

概念例：

```ts
if (
  response.data.result
  === 'difficult'
) {
  // 達成困難表示
}
```

表示例：

```text
現在の資産状況では、
目的の達成は難しい見込みです。
```

判定結果を
利用者に不必要に断定的な表現で
表示しない。

最終的な表示文言は、
画面設計に従う。

---

#### 28.8 判定不可の表示

`result = 'unassessable'`
の場合は、
APIエラーとして扱わない。

概念例：

```ts
if (
  response.data.result
  === 'unassessable'
) {
  // 判定不可表示
}
```

表示例：

```text
判定に必要な情報が不足しているため、
現在は目的達成可否を判定できません。
```

判定不可理由を
APIレスポンスで返却する設計となった場合は、
その理由に応じて
必要な登録操作への導線を表示してよい。

例えば、

```text
手取り収入が不足
    → 手取り収入登録画面

確定済み月末資産状況なし
    → 月末資産管理画面
```

などを検討できる。

---

#### 28.9 判定不可とAPIエラーを区別する

フロントエンドでは、
以下を明確に区別する。

```text
201 Created
result = unassessable
    → 正常な判定結果

4xx / 5xx
    → APIエラー
```

判定不可を
エラートーストや
通信失敗として表示しない。

また、
`mutation.isError`だけを使用して
判定不可を表現しない。

---

#### 28.10 判定成功後の表示

ASM-001成功後は、
レスポンスとして返却された
新しい判定履歴を
画面へ反映する。

```ts
const assessment =
  response.data;
```

例えば、
以下を表示できる。

```text
判定結果
達成可能

判定履歴ID
15
```

ただし、
判定履歴IDを
画面上で表示する必要がなければ、
内部的なルーティング用途だけに
使用してよい。

---

#### 28.11 判定履歴一覧との連携

ASM-001成功後は、
ASM-002 目的達成判定履歴一覧取得APIの
キャッシュを更新または
再取得する。

React Query等を使用する場合は、
対象目的の
判定履歴一覧クエリをinvalidateする。

```ts
queryClient.invalidateQueries({
  queryKey: [
    'assessmentHistories',
    objectiveId,
  ],
});
```

これにより、
新しく作成された判定履歴を
一覧へ反映できる。

---

#### 28.12 目的詳細との連携

目的詳細画面で
最新の目的情報を表示している場合でも、
ASM-001へ
画面上の目的情報を
リクエストボディとして送信しない。

判定対象は、
`objectiveId`によって
バックエンドで再取得する。

これにより、
フロントエンドが保持している
古い目的情報で
判定されることを防止する。

---

#### 28.13 月末資産状況との連携

フロントエンド側で、
最新の月末資産状況IDを
ASM-001へ指定しない。

```text
フロント
最新snapshotIdを決定
    ↓
ASM-001へ送信
```

という構成は採用しない。

ASM-001内部で、

```text
操作対象利用者
+
confirmed = true
+
target_year_month DESC
```

により、
最新の確定済み月末資産状況を
決定する。

これにより、
判定ロジックを
バックエンドへ集約する。

---

#### 28.14 手取り収入との連携

フロントエンド側で
直近3ヶ月の手取り収入を取得し、
平均値を計算して
ASM-001へ送信しない。

以下の処理は、
バックエンドで行う。

```text
直近3ヶ月の手取り収入取得
    ↓
平均値算出
    ↓
目的達成判定
```

フロントエンドでは、
判定結果の表示に責務を限定する。

---

#### 28.15 エラー処理

エラーコードごとの
基本的な扱いは、
以下とする。

| エラーコード | フロントエンドの扱い |
|---|---|
| `USER_CONTEXT_REQUIRED` | 利用者の選択を促す |
| `INVALID_USER_ID` | 共通エラー表示を行う |
| `USER_NOT_FOUND` | 利用者選択画面へ戻す |
| `VALIDATION_ERROR` | 不正なURLまたは画面状態として扱う |
| `OBJECTIVE_NOT_FOUND` | 目的一覧画面へ戻す |
| `OBJECTIVE_NOT_ASSESSABLE` | 現在は判定対象外であることを表示する |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

`OBJECTIVE_NOT_FOUND`の場合は、
他利用者の目的であるか、
本当に存在しないかを
フロントエンドで判別しない。

---

#### 28.16 objectiveIdのバリデーションエラー

通常の画面操作では、
不正な`objectiveId`が
発生しないことを前提とする。

例えば、

```text
objectiveId = abc
```

などがURLへ直接入力された場合は、
`VALIDATION_ERROR`
となる。

フロントエンドでは、
入力フォームのエラーとしてではなく、
不正な画面状態として扱う。

必要に応じて、
目的一覧画面へ戻す。

---

#### 28.17 OBJECTIVE_NOT_FOUND

`OBJECTIVE_NOT_FOUND`
が返却された場合は、
対象の目的を
現在参照できない状態として扱う。

例えば、
以下が考えられる。

- 目的が削除された
- 他の画面で状態が変更された
- URLが古い
- 他利用者の目的IDが指定された

理由を推測せず、
目的一覧へ戻す。

---

#### 28.18 OBJECTIVE_NOT_ASSESSABLE

`OBJECTIVE_NOT_ASSESSABLE`
が返却された場合は、
目的自体は存在するが、
現在の業務状態では
判定対象外であることを表示する。

表示例：

```text
この目的は、
現在は目的達成判定の対象ではありません。
```

具体的な理由を
APIが返却する設計となった場合は、
その理由に応じて
表示を変更してよい。

---

#### 28.19 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`
が返却された場合は、
共通エラーUIを使用する。

表示例：

```text
目的達成判定を実行できませんでした。
時間をおいて再度お試しください。
```

ASM-001は冪等ではないため、
通信タイムアウトなどで
サーバー側の処理結果が不明な場合に、
クライアント側で
無条件に自動再送しない。

必要に応じて、
判定履歴一覧を再取得し、
新しい履歴が作成されているか
確認できる設計を検討する。

---

#### 28.20 自動リトライ

ASM-001は
実行ごとに
新しい判定履歴を作成する。

そのため、
React Query等のMutationで
自動リトライを設定する場合は注意する。

Phase1では、
ASM-001のMutationについて
自動リトライを無効とする。

概念例：

```ts
useMutation({
  mutationFn: executeAssessment,
  retry: false,
});
```

これにより、
一時的な通信エラーによる
意図しない履歴の重複作成を防止する。

---

#### 28.21 Mutation実装例

概念例：

```ts
type ExecuteAssessmentArgs = {
  objectiveId: string;
};

export const executeAssessment =
  async ({
    objectiveId,
  }: ExecuteAssessmentArgs) => {
    const response =
      await apiClient.post<ExecuteAssessmentResponse>(
        `/api/v1/objectives/${objectiveId}/assessments`,
      );

    return response.data;
  };
```

Hook例：

```ts
export const useExecuteAssessment =
  (
    objectiveId: string,
  ) => {
    const queryClient =
      useQueryClient();

    return useMutation({
      mutationFn:
        () =>
          executeAssessment({
            objectiveId,
          }),

      retry: false,

      onSuccess: async () => {
        await queryClient.invalidateQueries({
          queryKey: [
            'assessmentHistories',
            objectiveId,
          ],
        });
      },
    });
  };
```

実際のAPI Client、
React Queryおよび
エラー処理の共通化方式は、
フロントエンド共通設計に従う。

---

### 30 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [ASM-002 目的達成判定結果保存](./asm-002-create.md)
- [ASM-003 判定履歴一覧取得](./asm-003-list.md)
- [ASM-003 判定履歴詳細取得](./asm-004-detail.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
- [Laravelアーキテクチャ設計](../../../architecture/laravel/assessments.md)
- [Reactアーキテクチャ設計](../../../architecture/react/assessments.md)
- [バックエンドテスト方針](../../../tests/backend/README.md)
- [フロントエンドテスト方針](../../../tests/frontend/README.md)
- [E2Eテスト方針](../../../tests/e2e/README.md)