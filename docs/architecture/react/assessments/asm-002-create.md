##  ASM-002 目的達成判定履歴一覧取得

### 1 概要

本ドキュメントでは、
ASM-002 目的達成判定履歴一覧取得APIを
React・TypeScriptから利用する際の
フロントエンドアーキテクチャおよび
責務分離方針を定義する。

ASM-002では、
操作対象利用者の指定された目的に紐づく
保存済みの目的達成判定履歴を取得し、
目的詳細画面などで
過去の判定結果を一覧表示する。

本APIは読み取り専用のGET APIであるため、
React QueryではQueryとして扱い、
取得結果をクライアントキャッシュの対象とする。

概念的な処理構成は、
以下とする。

```text
Page / Component
    ↓
Custom Hook
    ↓
React Query
    ↓
API Client
    ↓
ASM-002
    ↓
React Query Cache
    ↓
Component
```

フロントエンドでは、
API通信、
Query Key生成、
キャッシュ管理、
画面状態、
表示処理を
単一のコンポーネントへ集約せず、
それぞれの責務を分離する。

API Clientは、
`objectiveId`、
`page`、
`perPage`を使用して
ASM-002へリクエストを送信する。

`X-User-Id`は、
ASM-002固有の処理として設定せず、
操作対象利用者コンテキストを管理する
共通API ClientまたはInterceptorから付与する。

Custom Hookは、
ASM-002の取得処理と
React Queryによるキャッシュ制御を担当し、
画面コンポーネントから
通信処理の詳細を分離する。

Query Keyには、
少なくとも

```text
操作対象利用者
objectiveId
page
perPage
```

を含める。

これにより、
利用者、
目的、
ページネーション条件ごとに
取得結果を別キャッシュとして管理し、
利用者切り替え時に
別利用者の目的達成判定履歴が
誤って表示されることを防止する。

ASM-001によって
新しい目的達成判定履歴が保存された場合は、
対象利用者・対象目的に対応する
ASM-002の一覧Queryをinvalidateし、
最新の履歴一覧を再取得する。

画面では、
APIから返却された
`result`を保存済みの判定結果として扱い、

```text
achievable
difficult
unassessable
```

をそれぞれ
画面表示用のラベルへ変換する。

`unassessable`も
APIエラーではなく、
正常な判定履歴の一つとして表示する。

また、
履歴が0件の場合もエラーとはせず、
空状態として画面表示する。

ASM-002では、
過去の判定結果を表示することに責務を限定し、
React側で現在の目的、
資産状況、
手取り収入などから
目的達成判定を再計算しない。

目的達成判定の業務ロジックは
Laravel側へ集約し、
React側では、

```text
APIへ取得要求
    ↓
取得結果をキャッシュ・状態として保持
    ↓
表示用データへ変換
    ↓
画面表示
```

を主な責務とする。

これにより、
Reactコンポーネントへ
API通信、
キャッシュ制御、
目的達成判定ロジックなどが
混在することを防ぎ、
ASM-002を利用する画面の
責務を明確に分離する。

---

### 2. React・TypeScriptでの利用

ASM-002は、指定された目的に紐づく目的達成判定履歴一覧を取得するために使用する。

フロントエンドでは、主に以下の用途で利用する。

- 目的詳細画面で過去の目的達成判定履歴を表示する
- 目的達成判定実行後に最新の判定履歴を確認する
- 過去の判定結果の推移を確認する
- 判定履歴詳細画面への遷移元として利用する
- 達成可能・達成困難・判定不可の履歴を表示する

ASM-002は、読み取り専用のGET APIであるため、React QueryではQueryとして扱う。

```text
目的詳細画面
    ↓
ASM-002
    ↓
目的達成判定履歴一覧取得
    ↓
React Query Cache
    ↓
履歴一覧表示
```

ASM-002では、保存済みの目的達成判定履歴を表示することに責務を限定する。

フロントエンド側で、

- 現在の目的
- 現在の資産状況
- 現在の手取り収入

などを使用して、過去の判定結果を再計算してはならない。

---

#### 2.1 型定義

ASM-002のレスポンスに対応するTypeScriptの型を定義する。

概念例：

```ts
export type AssessmentResult =
  | 'achievable'
  | 'difficult'
  | 'unassessable';

export type AssessmentHistory = {
  id: string;
  objectiveId: string;
  result: AssessmentResult;
};

export type AssessmentHistoryListMeta = {
  currentPage: number;
  perPage: number;
  total: number;
  lastPage: number;
};

export type ListAssessmentHistoriesResponse = {
  data: AssessmentHistory[];
  meta: AssessmentHistoryListMeta;
};
```

`result`を単純な`string`として扱わず、バックエンドで定義された判定結果に対応するUnion Typeとして扱う。

これにより、

```ts
history.result === 'achievabel'
```

のようなタイプミスをTypeScriptで検出できる。

---

#### 2.2 objectiveIdの扱い

`objectiveId`は、API共通方針に従ってstringとして扱う。

```ts
const objectiveId: string = '5';
```

フロントエンド側でnumberへ変換して業務計算には使用しない。

URL生成時も、stringのまま使用する。

---

#### 2.3 pageの扱い

`page`は、1以上のintegerとして扱う。

```ts
const page = 1;
```

フロントエンドでは、ページネーションUIの現在ページとして保持してよい。

```ts
const [page, setPage] =
  useState(1);
```

未指定の場合は、バックエンド側で1ページ目として扱われる。

---

#### 2.4 perPageの扱い

`perPage`は、1ページあたりの取得件数として扱う。

```ts
const perPage = 20;
```

Phase1で表示件数変更UIを提供しない場合は、固定値として扱ってよい。

利用者が変更できる場合でも、API共通方針で定めた最大件数を超える値を送信しない。

---

#### 2.5 API Client

ASM-002を呼び出すAPI Client関数を定義する。

概念例：

```ts
type ListAssessmentHistoriesParams = {
  objectiveId: string;
  page: number;
  perPage: number;
};

export const fetchAssessmentHistories =
  async ({
    objectiveId,
    page,
    perPage,
  }: ListAssessmentHistoriesParams) => {
    const response =
      await apiClient.get<ListAssessmentHistoriesResponse>(
        `/api/v1/objectives/${objectiveId}/assessments`,
        {
          params: {
            page,
            perPage,
          },
        },
      );

    return response.data;
  };
```

`X-User-Id`は、ASM-002専用のAPI Client関数で個別に設定しない。

操作対象利用者コンテキストを管理する共通API ClientまたはInterceptorから付与する。

概念的には、

```text
React Component
    ↓
fetchAssessmentHistories()
    ↓
共通API Client
    ↓
X-User-Id付与
    ↓
ASM-002
```

とする。

---

#### 2.6 Queryとして扱う

ASM-002は、読み取り専用のGET APIであるため、React Queryでは`useQuery`を使用する。

`useMutation`は使用しない。

```text
GET
+
読み取り専用
    ↓
useQuery
```

ASM-002の呼び出しによって、目的達成判定を実行してはならない。

---

#### 2.7 Query Key

Query Keyには、少なくとも以下を含める。

```text
操作対象利用者
objectiveId
page
perPage
```

概念例：

```ts
export const assessmentHistoryKeys = {
  all: [
    'assessmentHistories',
  ] as const,

  lists: (
    userId: string,
    objectiveId: string,
  ) =>
    [
      ...assessmentHistoryKeys.all,
      'list',
      userId,
      objectiveId,
    ] as const,

  list: (
    userId: string,
    objectiveId: string,
    page: number,
    perPage: number,
  ) =>
    [
      ...assessmentHistoryKeys.lists(
        userId,
        objectiveId,
      ),
      page,
      perPage,
    ] as const,
};
```

利用者IDをQuery Keyへ含めることで、利用者切り替え時に別利用者の判定履歴キャッシュが表示されることを防止する。

---

#### 2.8 Hook

ASM-002専用のCustom Hookを定義する。

概念例：

```ts
export const useAssessmentHistories =
  (
    userId: string,
    objectiveId: string,
    page: number,
    perPage: number,
  ) => {
    return useQuery({
      queryKey:
        assessmentHistoryKeys.list(
          userId,
          objectiveId,
          page,
          perPage,
        ),

      queryFn:
        () =>
          fetchAssessmentHistories({
            objectiveId,
            page,
            perPage,
          }),

      enabled:
        userId.length > 0 &&
        objectiveId.length > 0,
    });
  };
```

利用者または目的が確定していない状態では、ASM-002を実行しない。

---

#### 2.9 履歴一覧表示

取得した`data`を使用して、目的達成判定履歴一覧を表示する。

概念例：

```tsx
{data.data.map((history) => (
  <AssessmentHistoryRow
    key={history.id}
    history={history}
  />
))}
```

一覧画面では、ASM-002から返却された

```text
id
objectiveId
result
```

を使用する。

一覧表示に必要のない判定内部情報を別APIから追加取得しない。

---

#### 2.10 判定結果の表示

`result`に応じて、画面表示用ラベルへ変換する。

概念例：

```ts
const resultLabelMap:
  Record<AssessmentResult, string> = {
    achievable: '達成可能',
    difficult: '達成困難',
    unassessable: '判定不可',
  };
```

表示例：

```tsx
<span>
  {resultLabelMap[history.result]}
</span>
```

APIから返却された値と画面表示用文言を分離する。

コンポーネント内へ、

```ts
history.result === 'achievable'
  ? '達成可能'
  : ...
```

のような条件分岐を複数箇所へ分散させない。

---

#### 2.11 判定不可の扱い

`unassessable`も、正常な目的達成判定履歴として扱う。

```ts
history.result === 'unassessable'
```

であっても、APIエラーとして扱わない。

また、一覧から除外しない。

```text
achievable
difficult
unassessable
    ↓
すべて通常の履歴
```

判定不可である理由を一覧APIのレスポンスだけからフロントエンド側で推測しない。

詳細な情報が必要な場合は、判定履歴詳細取得APIを利用する。

---

#### 2.12 履歴0件

`data`が空配列の場合は、エラー表示を行わない。

```tsx
if (data.data.length === 0) {
  return (
    <EmptyState>
      目的達成判定履歴はありません。
    </EmptyState>
  );
}
```

履歴0件は、

```text
200 OK
data = []
```

として返却される正常状態である。

目的達成判定を実行できる画面であれば、必要に応じて判定実行への導線を表示してよい。

---

#### 2.13 ローディング表示

ASM-002の取得中は、ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return <Loading />;
}
```

初回取得とページ切り替え時の再取得を必要に応じて区別する。

ページ切り替え時に現在の一覧を維持するかどうかは、React Queryの共通設計に従う。

---

#### 2.14 ページネーションUI

レスポンスの`meta`を使用して、ページネーションUIを構築する。

概念例：

```tsx
<Pagination
  currentPage={
    data.meta.currentPage
  }
  lastPage={
    data.meta.lastPage
  }
  onChange={setPage}
/>
```

総件数を表示する場合は、

```tsx
<span>
  全{data.meta.total}件
</span>
```

のように`meta.total`を使用する。

`lastPage`などをフロントエンド側で独自計算しない。

---

#### 2.15 ページ切り替え

ページ変更時は、`page`を更新する。

```ts
setPage(nextPage);
```

Query Keyに`page`が含まれているため、ページ変更によって新しいASM-002リクエストが実行される。

```text
page = 1
    ↓
Query Key A

page = 2
    ↓
Query Key B
```

ページ切り替えによって目的達成判定そのものが実行されることはない。

---

#### 2.16 perPage変更

表示件数変更UIを提供する場合は、`perPage`変更時に`page`を1へ戻す。

概念例：

```ts
const handlePerPageChange =
  (nextPerPage: number) => {
    setPerPage(nextPerPage);
    setPage(1);
  };
```

Phase1で表示件数変更UIを提供しない場合は、固定値としてよい。

---

#### 2.17 ASM-001との連携

目的達成判定によって新しい履歴が保存された場合は、ASM-002の一覧キャッシュをinvalidateする。

概念例：

```ts
await queryClient.invalidateQueries({
  queryKey:
    assessmentHistoryKeys.lists(
      userId,
      objectiveId,
    ),
});
```

これにより、

```text
目的達成判定実行
    ↓
新しい履歴保存
    ↓
ASM-002 Cache invalidate
    ↓
一覧再取得
    ↓
最新履歴表示
```

という流れにする。

ページ単位のQuery Keyだけをinvalidateするのではなく、対象目的の一覧Query全体をinvalidateできる構造とする。

---

#### 2.18 最新履歴

ASM-002は、新しい履歴から返却する。

そのため、1ページ目の先頭要素は、取得時点における最新履歴として扱える。

```ts
const latestAssessment =
  data.data[0] ?? null;
```

ただし、ASM-002はあくまで一覧取得APIである。

最新履歴専用APIのような強い契約として`data[0]`へ依存しない。

---

#### 2.19 詳細画面への遷移

各履歴から、判定履歴詳細画面へ遷移できる。

概念例：

```tsx
<Link
  to={
    `/assessment-histories/${history.id}`
  }
>
  詳細
</Link>
```

詳細画面では、一覧で取得済みのデータだけに依存せず、判定履歴詳細取得APIを使用して対象履歴を取得する。

これにより、直接URLから詳細画面へアクセスした場合でも表示できるようにする。

---

#### 2.20 利用者切り替え

Phase1では、操作対象利用者を切り替えて利用する。

利用者切り替え時は、以前の利用者の目的達成判定履歴を表示してはならない。

そのため、Query Keyには利用者IDを含める。

```text
User A
assessmentHistories / User A / Objective 5

User B
assessmentHistories / User B / Objective 5
```

`objectiveId`が同じ値であっても、利用者が異なる場合は別キャッシュとして扱う。

必要に応じて、利用者切り替え時に旧利用者のQueryをremoveまたはinvalidateする。

---

#### 2.21 OBJECTIVE_NOT_FOUND

`OBJECTIVE_NOT_FOUND`が返却された場合は、対象目的を現在参照できない状態として扱う。

原因として、

- 目的が存在しない
- 目的が論理削除されている
- 他利用者の目的である

などが考えられるが、フロントエンドでは原因を推測しない。

例えば、

```text
指定された目的を表示できません。
```

などの共通表示を行い、目的一覧画面へ戻す。

---

#### 2.22 利用者コンテキストエラー

以下のエラーは、利用者コンテキストに関する共通エラー処理として扱う。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
```

ASM-002専用コンポーネント内へ同じエラー処理を重複実装しない。

利用者選択画面への遷移など、フロントエンド共通方針に従って処理する。

---

#### 2.23 VALIDATION_ERROR

`objectiveId`、`page`、`perPage`が不正な場合は、

```text
VALIDATION_ERROR
```

が返却される。

通常のUI操作では、不正な`page`や`perPage`を送信しないようにする。

`objectiveId`の形式不正は、不正なURLまたは画面状態として扱う。

---

#### 2.24 INTERNAL_SERVER_ERROR

`INTERNAL_SERVER_ERROR`の場合は、共通エラー表示を行う。

例えば、

```text
データの取得に失敗しました。
時間をおいて再度お試しください。
```

などを表示する。

バックエンドから返却されない内部エラー情報をフロントエンド側で推測して表示しない。

---

#### 2.25 自動リトライ

ASM-002は、読み取り専用のGET APIであり、冪等である。

そのため、通信障害などの一時的なエラーについては、React Queryの自動リトライを利用してよい。

ただし、

```text
400
404
422
```

など、再実行しても解消しないエラーは自動リトライしない。

リトライ条件は、ASM-002専用に重複実装せず、可能な限りReact QueryまたはAPI Clientの共通設定へ集約する。

---

#### 2.26 クライアントキャッシュ

ASM-002は、React Queryによるクライアントキャッシュの対象とする。

キャッシュ単位は、

```text
利用者
+
目的
+
page
+
perPage
```

とする。

ただし、新しい目的達成判定履歴が保存された場合は、対象目的の一覧キャッシュをinvalidateする。

サーバー側キャッシュとは別の責務として扱う。

---

#### 2.27 現在データから再計算しない

フロントエンドでは、過去の判定履歴を表示する際に、

- 現在の目的
- 現在の月末資産状況
- 現在の資産額
- 現在の手取り収入

を使用して判定結果を再計算しない。

以下のような処理は行わない。

```ts
const result =
  calculateAssessment(
    currentObjective,
    currentAssets,
    currentNetIncome,
  );
```

ASM-002から取得した

```text
history.result
```

をそのまま過去の判定結果として表示する。

目的達成判定の業務ロジックはLaravel側へ集約する。

---

#### 2.28 コンポーネントの責務

画面コンポーネントへ、API通信、キャッシュ制御、表示変換をすべて直接記述しない。

概念的には、以下のように分離する。

```text
AssessmentHistoryListPage
    ↓
useAssessmentHistories
    ↓
fetchAssessmentHistories
    ↓
API Client
```

表示については、

```text
AssessmentHistoryList
    ↓
AssessmentHistoryRow
```

のように必要に応じてコンポーネントを分離する。

判定結果のラベル変換も、画面コンポーネントへ重複して記述しない。

---

#### 2.29 フロントエンドで保持しない業務ロジック

ASM-002を利用するReact側では、以下の業務ロジックを持たない。

- 目的達成判定
- 利用可能資産額の計算
- 手取り収入平均の計算
- 目的達成可否の判定
- 判定不可条件の判定
- 過去の判定結果の再計算
- 利用者境界の最終判定

Reactは、

```text
APIへ取得要求
    ↓
取得結果を状態として保持
    ↓
画面表示
```

を主な責務とする。

利用者境界についても、フロントエンド側で表示制御は行うものの、セキュリティ上の最終的な保証はLaravel側で行う。

---

### 3 関連ドキュメント

- [ASM-002 API詳細設計](../../../api/details/assessments/asm-002-create.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Reactアーキテクチャ共通設計](../react-architecture.md)
- [目的達成判定 Reactアーキテクチャ設計](./README.md)
- [ASM-002 Laravelアーキテクチャ設計](../../../architecture/laravel/assessments/asm-002-create.md)
- [ASM-002 テスト設計](../../../tests/assessments/asm-002-create.md)
- [目的達成判定 テスト設計](../../../tests/assessments/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
