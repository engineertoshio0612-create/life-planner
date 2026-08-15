##  ASM-003 目的達成判定履歴詳細取得

### 1 概要

本ドキュメントでは、
ASM-003 目的達成判定履歴詳細取得APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

ASM-003では、
操作対象利用者の指定された目的に紐づく
特定の目的達成判定履歴1件を取得し、
過去の目的達成判定結果として表示する。

フロントエンドでは、
ASM-003から返却された保存済みの判定結果を使用し、
現在の目的、資産状況、手取り収入などを用いて
目的達成可否を再計算しない。

また、
目的達成判定履歴一覧から詳細画面へ遷移した場合でも、
一覧画面で保持しているデータだけに依存せず、
ASM-003によって対象履歴を取得する。

これにより、
一覧画面からの遷移と
詳細URLへの直接アクセスで
同一のデータ取得処理を利用できるようにする。

React・TypeScript実装では、
主に以下の責務を扱う。

* `objectiveId`および`assessmentId`をstringとして管理する
* ASM-003を読み取り専用のQueryとして実行する
* `objectiveId`と`assessmentId`を含むQuery Keyでキャッシュを分離する
* 保存済みの判定結果を過去履歴として表示する
* `unassessable`を正常な判定結果として扱う
* APIエラーコードに応じて画面遷移またはエラー表示を行う
* 現在の目的情報と過去の判定結果を混同しない

API Client、React Query、
共通エラー処理、利用者切替時のキャッシュ制御などの
横断的な実装方針については、
React共通アーキテクチャ設計に従う。

ASM-003固有の責務は、
「指定された目的達成判定履歴を取得し、
保存された過去の判定結果として表示すること」
に限定する。

---

### 2 React・TypeScriptでの利用

パスパラメータの型は、
以下とする。

```ts
export type GetAssessmentHistoryParams = {
  objectiveId: string;
  assessmentId: string;
};
```

目的達成判定履歴詳細の型は、
以下とする。

```ts
export type AssessmentResult =
  | 'achievable'
  | 'difficult'
  | 'unassessable';

export type AssessmentHistoryDetail = {
  id: string;
  objectiveId: string;
  result: AssessmentResult;
};
```

レスポンス型は、
以下とする。

```ts
export type GetAssessmentHistoryResponse = {
  data: AssessmentHistoryDetail;
};
```

API呼び出し例は、
以下とする。

```ts
const response =
  await apiClient.get<GetAssessmentHistoryResponse>(
    `/api/v1/objectives/${objectiveId}/assessments/${assessmentId}`,
  );
```

目的達成判定履歴一覧画面などから、
特定の目的達成判定履歴を選択し、
詳細を表示する際に利用する。

---

#### 2.1 objectiveIdの扱い

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

#### 2.2 assessmentIdの扱い

`assessmentId`は、
API共通方針に従って
stringとして扱う。

```ts
const assessmentId: string = '20';
```

URL生成時は、
`objectiveId`と
`assessmentId`の両方を使用する。

```ts
const url =
  `/api/v1/objectives/${objectiveId}/assessments/${assessmentId}`;
```

フロントエンド側で、
`assessmentId`だけを使用して
履歴詳細を取得しない。

---

#### 2.3 Queryとして扱う

ASM-003は、
読み取り専用のGET APIであるため、
React Query等では
Queryとして扱う。

概念例：

```ts
export const fetchAssessmentHistory =
  async ({
    objectiveId,
    assessmentId,
  }: GetAssessmentHistoryParams) => {
    const response =
      await apiClient.get<GetAssessmentHistoryResponse>(
        `/api/v1/objectives/${objectiveId}/assessments/${assessmentId}`,
      );

    return response.data;
  };
```

---

#### 2.4 Query Key

Query Keyには、
`objectiveId`および
`assessmentId`を含める。

概念例：

```ts
export const assessmentHistoryKeys = {
  all: [
    'assessmentHistories',
  ] as const,

  detail: (
    objectiveId: string,
    assessmentId: string,
  ) =>
    [
      ...assessmentHistoryKeys.all,
      objectiveId,
      assessmentId,
    ] as const,
};
```

これにより、
目的および履歴ごとに
キャッシュを分離する。

---

#### 2.5 Hook実装例

React Queryを使用する場合の
概念例は、
以下とする。

```ts
export const useAssessmentHistory =
  (
    objectiveId: string,
    assessmentId: string,
  ) => {
    return useQuery({
      queryKey:
        assessmentHistoryKeys.detail(
          objectiveId,
          assessmentId,
        ),

      queryFn:
        () =>
          fetchAssessmentHistory({
            objectiveId,
            assessmentId,
          }),

      enabled:
        objectiveId.length > 0
        && assessmentId.length > 0,
    });
  };
```

実際のAPI Clientや
React Queryの利用方針は、
フロントエンド共通設計に従う。

---

#### 2.6 ASM-002からの遷移

ASM-002 目的達成判定履歴一覧取得APIで
取得した履歴から、
詳細画面へ遷移する。

概念例：

```tsx
<Link
  to={
    `/objectives/${objectiveId}/assessments/${history.id}`
  }
>
  詳細
</Link>
```

詳細画面では、
一覧画面で保持している
`result`だけに依存せず、
ASM-003によって
対象履歴を再取得する。

---

#### 2.7 詳細画面で再取得する理由

一覧取得後に
画面遷移する場合でも、
詳細画面では
ASM-003を実行する。

これにより、
詳細画面のデータ取得責務を
ASM-003へ統一できる。

また、
ブラウザから
詳細URLへ直接アクセスした場合でも
同じ処理で表示できる。

---

#### 2.8 判定結果の表示

`result`に応じて、
表示内容を切り替える。

概念例：

```ts
const resultLabelMap: Record<
  AssessmentResult,
  string
> = {
  achievable: '達成可能',
  difficult: '達成困難',
  unassessable: '判定不可',
};
```

```tsx
<span>
  {resultLabelMap[data.result]}
</span>
```

バックエンドから返却された
保存済みの`result`を使用する。

---

#### 2.9 達成可能の表示

`result = 'achievable'`
の場合は、
過去の判定結果として
達成可能であったことを表示する。

表示例：

```text
判定結果：達成可能
```

現在の資産状況でも
達成可能であることを
意味するものとして
再解釈しない。

---

#### 2.10 達成困難の表示

`result = 'difficult'`
の場合は、
過去の判定結果として
達成困難であったことを表示する。

表示例：

```text
判定結果：達成困難
```

現在の状態を使用して
結果を書き換えない。

---

#### 2.11 判定不可の表示

`result = 'unassessable'`
の場合も、
正常な履歴詳細として表示する。

表示例：

```text
判定結果：判定不可
```

APIエラーとして扱わない。

```ts
if (
  data.result
  === 'unassessable'
) {
  // 判定不可として通常表示
}
```

---

#### 2.12 過去履歴として表示する

ASM-003で取得したデータは、
現在の判定結果ではなく、
過去の判定実行結果として扱う。

画面上でも、
必要に応じて

```text
過去の目的達成判定結果
```

であることが
分かる表示とする。

現在の目的達成可否を
確認したい場合は、
ASM-001を再実行する。

---

#### 2.13 現在データから再計算しない

フロントエンドでは、
ASM-003取得後に、

- 現在の目的
- 現在の資産額
- 現在の手取り収入

を使用して
`result`を再計算しない。

以下のような処理は行わない。

```ts
const currentResult =
  calculateAssessment(
    currentObjective,
    currentAssets,
    currentNetIncome,
  );
```

ASM-003から返却された
`result`をそのまま表示する。

---

#### 2.14 現在の目的情報と混同しない

詳細画面で
現在の目的情報を
別APIから取得して表示する場合でも、
ASM-003の`result`は
過去の履歴として扱う。

例えば、

```text
現在の目的金額
5,000,000円

過去の判定結果
達成困難
```

のように、
現在情報と
過去履歴を
同じ時点の情報として
誤認させないようにする。

---

#### 2.15 OBJECTIVE_NOT_FOUND

`OBJECTIVE_NOT_FOUND`
が返却された場合は、
対象目的を
現在参照できない状態として扱う。

例えば、

- 目的が削除された
- URLが古い
- 他利用者の目的IDが指定された

などが考えられる。

理由を推測せず、
目的一覧画面へ戻す。

---

#### 2.16 ASSESSMENT_HISTORY_NOT_FOUND

`ASSESSMENT_HISTORY_NOT_FOUND`
が返却された場合は、
指定された目的に紐づく
対象履歴を
取得できない状態として扱う。

例えば、

- 履歴が存在しない
- URLが古い
- 別の目的に紐づく履歴IDが指定された

などが考えられる。

フロントエンドでは、
履歴が別の目的に存在するかどうかを
推測しない。

必要に応じて、
ASM-002の
目的達成判定履歴一覧画面へ戻す。

---

#### 2.17 VALIDATION_ERROR

`objectiveId`または
`assessmentId`の形式が不正な場合は、
`VALIDATION_ERROR`
が返却される。

通常の画面操作では
発生しないことを前提とし、
不正なURLまたは
画面状態として扱う。

必要に応じて、
目的一覧または
判定履歴一覧へ戻す。

---

#### 2.18 エラー表示

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
| `ASSESSMENT_HISTORY_NOT_FOUND` | 目的達成判定履歴一覧画面へ戻す |
| `INTERNAL_SERVER_ERROR` | 共通エラー表示を行う |

`unassessable`は、
エラーコードとして扱わない。

---

#### 2.19 ローディング表示

詳細取得中は、
ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return <Loading />;
}
```

取得完了前に、
一覧画面の古い情報を
詳細情報として確定表示しない。

---

#### 2.20 自動リトライ

ASM-003は
読み取り専用GET APIであり、
冪等である。

そのため、
一時的な通信エラーについては、
React Query等の
標準的な自動リトライを
利用してよい。

ただし、

```text
400
404
422
```

など、
再送しても解消しない
クライアント起因エラーは、
不要にリトライしない。

---

#### 2.21 キャッシュ

ASM-003は、
React Query等による
クライアントキャッシュの対象としてよい。

目的達成判定履歴は
原則として過去の履歴であり、
ASM-003自体によって
内容が変更されることはない。

ただし、
キャッシュ保持時間などは
フロントエンド共通設計に従う。

---

#### 2.22 ASM-001との関係

ASM-001を再実行して
新しい目的達成判定履歴が作成されても、
過去の`assessmentId`に対する
ASM-003の結果は
変更されないことを前提とする。

例えば、

```text
assessmentId = 10
result = difficult

    ↓ ASM-001再実行

assessmentId = 20
result = achievable
```

となった場合でも、

```http
GET /objectives/5/assessments/10
```

では、

```text
result = difficult
```

を返却する。

最新結果を表示したい場合は、
新しい履歴を取得する。

---

#### 2.23 Query実装例

概念例：

```ts
type GetAssessmentHistoryArgs = {
  objectiveId: string;
  assessmentId: string;
};

export const getAssessmentHistory =
  async ({
    objectiveId,
    assessmentId,
  }: GetAssessmentHistoryArgs) => {
    const response =
      await apiClient.get<GetAssessmentHistoryResponse>(
        `/api/v1/objectives/${objectiveId}/assessments/${assessmentId}`,
      );

    return response.data;
  };
```

Hook例：

```ts
export const useAssessmentHistoryDetail =
  (
    objectiveId: string,
    assessmentId: string,
  ) => {
    return useQuery({
      queryKey:
        assessmentHistoryKeys.detail(
          objectiveId,
          assessmentId,
        ),

      queryFn:
        () =>
          getAssessmentHistory({
            objectiveId,
            assessmentId,
          }),

      enabled:
        objectiveId.length > 0
        && assessmentId.length > 0,
    });
  };
```

実際のAPI Client、
React Queryおよび
エラー処理の共通化方式は、
フロントエンド共通設計に従う。

---

### 関連ドキュメント

- [ASM-003 API詳細設計](../../../api/details/assessments/asm-003-list.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Reactアーキテクチャ共通設計](../react-architecture.md)
- [目的達成判定 Reactアーキテクチャ設計](./README.md)
- [ASM-003 Laravelアーキテクチャ設計](../../../architecture/laravel/assessments/asm-003-list.md)
- [ASM-003 テスト設計](../../../tests/assessments/asm-003-list.md)
- [目的達成判定 テスト設計](../../../tests/assessments/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
