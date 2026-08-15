# ASM-004 判定履歴詳細取得

### 1 概要

本ドキュメントでは、
ASM-004 判定履歴詳細取得APIを
React・TypeScriptから利用する際の
フロントエンド実装方針を定義する。

ASM-004では、
操作対象利用者に紐づく
指定された目的達成判定履歴1件について、
判定結果および判定実行時点の計算根拠を取得し、
判定履歴詳細画面へ表示する。

本APIは、
ASM-002によって保存された
過去の目的達成判定履歴を参照するための
読み取り専用APIである。

フロントエンドでは、
ASM-004から返却された保存済みの

* 判定結果
* 判定対象年月
* 判定実行日時
* 利用可能資産額
* 必要支出額
* 平均手取り収入
* 必要生活防衛資金
* 残額
* 判定不可理由

などを、
判定実行時点の情報として表示する。

現在の資産状況、
手取り収入、
利用可能資産設定などを別途取得して、
過去の判定結果や計算根拠を
フロントエンドで再計算しない。

概念的な利用フローは、
以下とする。

```text
ASM-003
判定履歴一覧取得
    ↓
判定履歴を選択
    ↓
assessmentHistoryId
    ↓
ASM-004
判定履歴詳細取得
    ↓
TanStack Query
    ↓
判定履歴詳細画面
    ├─ 判定結果
    ├─ 判定時点の計算根拠
    └─ 判定不可理由
```

ASM-004は参照専用GET APIであるため、
TanStack QueryではQueryとして扱い、
`assessmentHistoryId`を含むQuery Keyによって
判定履歴ごとのキャッシュを管理する。

また、
ASM-003の一覧データと
ASM-004の詳細データはキャッシュを分離し、
判定履歴詳細画面では
ASM-004のレスポンスを正とする。

React・TypeScript実装では、
主に以下の責務を扱う。

* URLから`assessmentHistoryId`を取得する
* ASM-004をQueryとして実行する
* LoadingおよびError状態を管理する
* 保存済みの判定結果を表示用ラベルへ変換する
* 判定時点の計算根拠を表示する
* `NOT_ASSESSABLE`を正常な判定結果として扱う
* 金額および日時を共通Formatterで表示形式へ変換する
* 利用者切替時に前利用者のキャッシュを誤表示しない
* APIエラーと判定結果を明確に区別する

API Client、TanStack Query、
利用者コンテキスト、Query Cache、
共通エラー処理、Formatterなどの
横断的な実装方針については、
Reactアーキテクチャ共通設計に従う。

ASM-004のReact・TypeScript実装責務は、

```text
assessmentHistoryIdを使用して
保存済み判定履歴を取得し、
判定結果と判定時点の計算根拠を
再計算せず詳細画面へ表示する
```

ことに限定する。

---

## 2. React・TypeScriptでの利用

ASM-004は、判定履歴一覧から特定の判定履歴を選択し、その判定結果および判定時点の計算根拠を詳細表示する際に使用する。

ASM-004は参照専用GET APIであるため、TanStack Queryを使用する場合はQueryとして扱う。

概念的な利用フローは、以下とする。

```text
ASM-003
判定履歴一覧取得
    ↓
利用者が履歴を選択
    ↓
assessmentHistoryId
    ↓
ASM-004
判定履歴詳細取得
    ↓
判定結果
+
判定時点の計算根拠
    ↓
詳細画面表示
```

---

### 2.1 TypeScript型

ASM-004では、Request Bodyを使用しない。

API呼び出しに必要な値は、

```text
assessmentHistoryId
```

のみとする。

概念例：

```ts
export type GetAssessmentHistoryDetailVariables = {
  assessmentHistoryId: string;
};
```

---

### 2.2 レスポンス型

ASM-004の判定履歴詳細を表す型を定義する。

概念例：

```ts
export type AssessmentHistoryDetail = {
  id: string;
  objectiveId: string;
  objectiveName: string;
  targetYearMonth: string;
  assessmentResult: AssessmentResult;
  availableAssetAmount: number | null;
  requiredAmount: number | null;
  averageNetIncome: number | null;
  requiredEmergencyFund: number | null;
  remainingAmount: number | null;
  notAssessableReason: NotAssessableReason | null;
  assessedAt: string;
};
```

具体的な項目およびNULL可否は、ASM-002の保存仕様とASM-004のレスポンス定義を正とする。

---

### 2.3 AssessmentResultの共通型

判定結果は、ASM-004専用の文字列型を個別に定義せず、ASM系APIで共通化する。

概念例：

```ts
export type AssessmentResult =
  | 'ACHIEVABLE'
  | 'NOT_ACHIEVABLE'
  | 'NOT_ASSESSABLE';
```

ASM-001、ASM-002、ASM-003、ASM-004で同じコード値を使用する。

---

### 2.4 NotAssessableReasonの共通型

判定不可理由についても、ASM-001・ASM-002と共通型を使用する。

概念例：

```ts
export type NotAssessableReason =
  | 'NET_INCOME_INSUFFICIENT'
  | 'ASSET_DATA_INSUFFICIENT';
```

実際のコード値は、目的達成判定の共通定義を正とする。

---

### 2.5 正常レスポンス型

API共通Envelopeを使用する場合は、以下のように定義する。

概念例：

```ts
export type GetAssessmentHistoryDetailResponse =
  ApiResponse<AssessmentHistoryDetail>;
```

---

### 2.6 IDはstringとして扱う

以下のIDは、React・TypeScript側で`string`として扱う。

```text
assessmentHistoryId
id
objectiveId
```

DB上で`bigint`であっても、フロントエンドで`number`へ変換しない。

---

### 2.7 金額はnumberとして扱う

以下の金額項目は、`number | null`として扱う。

```text
availableAssetAmount
requiredAmount
averageNetIncome
requiredEmergencyFund
remainingAmount
```

日本円整数として扱い、画面表示時にフォーマットする。

---

### 2.8 assessedAtはstringとして受け取る

`assessedAt`は、APIから日時文字列として受け取る。

概念例：

```ts
assessedAt: string;
```

必要に応じて、表示時にDateへ変換する。

API Client層で独自の日時フォーマットへ変換しない。

---

### 2.9 API Client

ASM-004を呼び出す専用API Client関数を定義する。

概念例：

```ts
export const getAssessmentHistoryDetail =
  async (
    assessmentHistoryId: string,
  ): Promise<AssessmentHistoryDetail> => {
    const response =
      await apiClient.get<
        GetAssessmentHistoryDetailResponse
      >(
        `/api/v1/assessment-histories/${assessmentHistoryId}`,
      );

    return response.data.data;
  };
```

コンポーネントから直接`fetch`や`axios`を呼び出さない。

---

### 2.10 userIdをAPI Client引数へ含めない

以下のようなAPI Clientにはしない。

```ts
getAssessmentHistoryDetail(
  userId,
  assessmentHistoryId,
);
```

利用者IDは、共通API Clientから

```text
X-User-Id
```

として付与する。

ASM-004固有の引数は、

```text
assessmentHistoryId
```

だけとする。

---

### 2.11 X-User-Id

`X-User-Id`は、ASM-004専用処理ではなく、共通API Clientから付与する。

概念例：

```ts
apiClient.interceptors.request.use(
  (config) => {
    config.headers['X-User-Id'] =
      currentUserId;

    return config;
  },
);
```

各Page、Component、Query Hookから直接設定しない。

---

### 2.12 Request Bodyを送信しない

ASM-004はGET APIであるため、Request Bodyを送信しない。

以下のような呼び出しにはしない。

```ts
apiClient.get(
  '/api/v1/assessment-histories/100',
  {
    data: {
      assessmentHistoryId: '100',
    },
  },
);
```

`assessmentHistoryId`はURLへ含める。

---

### 2.13 Queryとして扱う

ASM-004はサーバー状態を変更しないため、TanStack QueryではQueryとして扱う。

概念例：

```ts
export const useAssessmentHistoryDetail =
  (
    assessmentHistoryId: string,
  ) => {
    return useQuery({
      queryKey:
        assessmentHistoryKeys.detail(
          assessmentHistoryId,
        ),

      queryFn: () =>
        getAssessmentHistoryDetail(
          assessmentHistoryId,
        ),
    });
  };
```

Mutationとして実装しない。

---

### 2.14 Query Key

判定履歴関連のQuery Keyは、共通定義として管理する。

概念例：

```ts
export const assessmentHistoryKeys = {
  all: [
    'assessmentHistories',
  ] as const,

  lists: () =>
    [
      ...assessmentHistoryKeys.all,
      'list',
    ] as const,

  detail: (
    assessmentHistoryId: string,
  ) =>
    [
      ...assessmentHistoryKeys.all,
      'detail',
      assessmentHistoryId,
    ] as const,
};
```

---

### 2.15 assessmentHistoryIdをQuery Keyへ含める

以下のようなQuery Keyにはしない。

```ts
[
  'assessmentHistories',
  'detail',
]
```

判定履歴詳細は、`assessmentHistoryId`ごとに異なる。

そのため、

```ts
[
  'assessmentHistories',
  'detail',
  assessmentHistoryId,
]
```

のようにIDをQuery Keyへ含める。

---

### 2.16 利用者切替を考慮する

Phase1では、`X-User-Id`によって操作対象利用者を切り替える。

そのため、利用者切替時に前利用者の判定履歴詳細を誤表示しないようにする。

以下のいずれかとする。

```text
利用者切替時に
assessmentHistories系Cacheを破棄する

または

Query KeyへuserIdを含める
```

正式な方式は、React共通設計に従う。

---

### 2.17 Query KeyへuserIdを含めてもよい

Cache境界を明示する場合は、以下のようにしてよい。

概念例：

```ts
export const assessmentHistoryKeys = {
  detail: (
    userId: string,
    assessmentHistoryId: string,
  ) =>
    [
      'assessmentHistories',
      userId,
      'detail',
      assessmentHistoryId,
    ] as const,
};
```

ただし、`userId`はCache管理上の情報であり、ASM-004 API Clientの引数にはしない。

---

### 2.18 enabledによるQuery制御

`assessmentHistoryId`が取得できていない場合は、ASM-004を実行しない。

概念例：

```ts
return useQuery({
  queryKey:
    assessmentHistoryKeys.detail(
      assessmentHistoryId,
    ),

  queryFn: () =>
    getAssessmentHistoryDetail(
      assessmentHistoryId,
    ),

  enabled:
    assessmentHistoryId.length > 0,
});
```

---

### 2.19 ASM-003からの遷移

判定履歴一覧画面では、ASM-003で取得した

```text
assessmentHistoryId
```

を使用して詳細画面へ遷移する。

概念例：

```tsx
<Link
  to={
    `/assessment-histories/${history.id}`
  }
>
  詳細を見る
</Link>
```

詳細画面側では、URLから`assessmentHistoryId`を取得し、ASM-004を実行する。

---

### 2.20 Route Parameter

React Routerを使用する場合は、概念的に以下のように`assessmentHistoryId`を取得する。

```ts
const {
  assessmentHistoryId,
} = useParams<{
  assessmentHistoryId: string;
}>();
```

取得できない場合は、不正な画面状態として扱う。

---

### 2.21 詳細Page

判定履歴詳細Pageでは、主に以下を担当する。

- URLから`assessmentHistoryId`取得
- ASM-004実行
- Loading表示
- Error表示
- 判定結果表示
- 判定時点の計算根拠表示
- 判定不可理由表示

HTTP通信そのものは、API ClientおよびQuery Hookへ委譲する。

---

### 2.22 Loading状態

ASM-004取得中は、ローディング状態を表示する。

概念例：

```tsx
if (query.isPending) {
  return (
    <LoadingIndicator />
  );
}
```

---

### 2.23 Error状態

取得エラーの場合は、API共通エラー型を使用して表示を分岐する。

概念例：

```tsx
if (query.isError) {
  return (
    <AssessmentHistoryError
      error={query.error}
    />
  );
}
```

---

### 2.24 Empty Stateは使用しない

ASM-004は単一リソース取得APIである。

そのため、

```text
data = null
```

や

```text
空配列
```

を正常なEmpty Stateとして扱わない。

対象履歴が存在しない場合は、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

としてError状態を表示する。

---

### 2.25 判定結果表示

`assessmentResult`は、画面表示用ラベルへ変換する。

概念例：

```ts
export const assessmentResultLabels:
  Record<AssessmentResult, string> = {
    ACHIEVABLE:
      '達成可能',

    NOT_ACHIEVABLE:
      '達成不可',

    NOT_ASSESSABLE:
      '判定不可',
  };
```

APIのコード値をそのまま利用者へ表示しない。

---

### 2.26 判定結果による表示分岐

概念例：

```tsx
switch (
  assessmentHistory.assessmentResult
) {
  case 'ACHIEVABLE':
    return (
      <AchievableResult />
    );

  case 'NOT_ACHIEVABLE':
    return (
      <NotAchievableResult />
    );

  case 'NOT_ASSESSABLE':
    return (
      <NotAssessableResult />
    );
}
```

表示コンポーネントの詳細は、画面設計に従う。

---

### 2.27 判定不可はError表示にしない

`assessmentResult`が

```text
NOT_ASSESSABLE
```

であっても、API取得は成功している。

そのため、

```text
Error Page
```

として扱わない。

通常の判定結果表示領域で、

```text
判定不可
+
判定不可理由
```

を表示する。

---

### 2.28 notAssessableReason

判定不可の場合は、

```text
assessmentHistory.notAssessableReason
```

を使用して理由を表示する。

概念例：

```ts
export const notAssessableReasonLabels:
  Record<NotAssessableReason, string> = {
    NET_INCOME_INSUFFICIENT:
      '判定に必要な手取り収入が不足しています。',

    ASSET_DATA_INSUFFICIENT:
      '判定に必要な資産情報が不足しています。',
  };
```

実際の文言は、画面設計・用語定義を正とする。

---

### 2.29 判定可能時のnotAssessableReason

判定結果が

```text
ACHIEVABLE
NOT_ACHIEVABLE
```

の場合は、`notAssessableReason`を表示しない。

```tsx
{assessmentHistory.assessmentResult
  === 'NOT_ASSESSABLE' &&
  assessmentHistory.notAssessableReason && (
    <NotAssessableReason />
  )}
```

のように条件表示する。

---

### 2.30 金額表示

判定根拠となる金額は、共通Formatterを使用する。

概念例：

```ts
export const formatYen = (
  value: number,
): string =>
  new Intl.NumberFormat(
    'ja-JP',
    {
      style: 'currency',
      currency: 'JPY',
    },
  ).format(value);
```

---

### 2.31 nullの金額を考慮する

判定不可等により一部金額が`null`となる場合は、Formatterへそのまま渡さない。

概念例：

```ts
const displayAmount =
  value === null
    ? '-'
    : formatYen(value);
```

`null`を

```text
0円
```

として表示しない。

---

### 2.32 保存済み計算根拠を表示する

画面では、ASM-004から返却された

```text
availableAssetAmount
requiredAmount
averageNetIncome
requiredEmergencyFund
remainingAmount
```

を表示する。

フロントエンドで現在値から再計算しない。

---

### 2.33 remainingAmountを再計算しない

以下のような計算をReact側で行わない。

```ts
const remainingAmount =
  availableAssetAmount
  - requiredAmount
  - requiredEmergencyFund;
```

ASM-004の

```text
assessmentHistory.remainingAmount
```

を表示する。

保存済み判定根拠と画面計算結果がずれることを防止する。

---

### 2.34 averageNetIncomeを再計算しない

React側で`net_incomes`を取得し、過去判定の

```text
averageNetIncome
```

を再計算しない。

ASM-004レスポンスの保存済み値を使用する。

---

### 2.35 現在の資産額を組み込まない

過去判定の詳細表示では、現在の資産額を判定根拠として混在させない。

概念的には、

```text
ASM-004
    → 判定時点

現在資産API
    → 現在時点
```

と明確に分離する。

---

### 2.36 現在値との比較を行う場合

将来的に過去判定と現在値を比較表示する場合は、

```text
過去
    → ASM-004

現在
    → 別API
```

として取得する。

画面上でも、

```text
判定時点
現在
```

を明確に区別する。

---

### 2.37 objectiveName

目的名は、

```text
assessmentHistory.objectiveName
```

をそのまま表示する。

目的名表示のためだけにOBJ-003を追加呼び出ししない。

---

### 2.38 objectiveId

`objectiveId`は、必要に応じて目的詳細画面へのリンクに使用してよい。

概念例：

```tsx
<Link
  to={
    `/objectives/${assessmentHistory.objectiveId}`
  }
>
  目的を確認
</Link>
```

ただし、現在の目的が無効化済み・論理削除済み等の場合の遷移可否は、OBJ-003の仕様を正とする。

---

### 2.39 目的の現在状態を履歴画面で推測しない

ASM-004には現在の

```text
enabled
```

を含めない。

そのため、判定履歴詳細画面で

```text
この目的は現在有効です
```

などとASM-004だけから判断しない。

必要ならOBJ系APIを別途利用する。

---

### 2.40 targetYearMonth

判定対象年月は、

```text
assessmentHistory.targetYearMonth
```

を使用して表示する。

例えば、

```text
2026-08
```

を、

```text
2026年8月
```

などへ表示変換してよい。

---

### 2.41 assessedAt

判定実行日時は、共通日時Formatterを使用して表示する。

概念例：

```ts
export const formatDateTime = (
  value: string,
): string => {
  return new Intl.DateTimeFormat(
    'ja-JP',
    {
      dateStyle: 'medium',
      timeStyle: 'short',
    },
  ).format(
    new Date(value),
  );
};
```

---

### 2.42 日時フォーマットを各Componentへ重複実装しない

`assessedAt`の表示変換は、共通Formatterへ集約する。

各Componentで個別に

```ts
new Date(...).toLocaleString(...)
```

を記述しない。

---

### 2.43 ASM-002保存直後との連携

ASM-002成功後に判定結果詳細へ遷移する場合は、保存された

```text
assessmentHistoryId
```

を使用してASM-004へ遷移してよい。

概念的には、

```text
ASM-002
    ↓
assessmentHistoryId = 100
    ↓
/assessment-histories/100
    ↓
ASM-004
```

とする。

---

### 2.44 ASM-002のレスポンスをそのまま永続表示に使わない

ASM-002成功時に判定結果が返却されても、判定履歴詳細画面ではURLの`assessmentHistoryId`を基準としてASM-004を取得する構成としてよい。

これにより、ページ再読み込み後も同じ履歴を取得できる。

---

### 2.45 ASM-003のCacheとの関係

ASM-003の一覧CacheとASM-004の詳細Cacheは分離する。

概念的には、

```text
assessmentHistories/list
assessmentHistories/detail/{id}
```

とする。

一覧データだけを使って詳細画面を完全に再構築しない。

---

### 2.46 一覧データをinitialDataとして利用してもよい

ASM-003の一覧レスポンスにASM-004と共通する項目が十分含まれている場合は、表示高速化のため一覧Cacheを`initialData`等に利用してもよい。

ただし、ASM-004固有の

```text
詳細な計算根拠
```

はASM-004で取得する。

---

### 2.47 ASM-004は詳細情報の正とする

判定履歴詳細画面では、ASM-003の一覧データよりASM-004の詳細レスポンスを正とする。

一覧表示用に省略・加工された値から詳細を推測しない。

---

### 2.48 GETのRetry

ASM-004は副作用を持たないGET APIであるため、一時的な通信エラーに対して限定的なRetryを許可してよい。

概念例：

```ts
useQuery({
  queryKey:
    assessmentHistoryKeys.detail(
      assessmentHistoryId,
    ),

  queryFn: () =>
    getAssessmentHistoryDetail(
      assessmentHistoryId,
    ),

  retry: 1,
});
```

---

### 2.49 404を無意味にRetryしない

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

のような確定的な業務エラーについては、自動Retryを行わない。

Retry条件は、React共通のAPIエラー処理方針に従う。

---

### 2.50 INVALID_ASSESSMENT_HISTORY_ID

以下のエラーを受信した場合は、

```text
INVALID_ASSESSMENT_HISTORY_ID
```

URLまたは画面状態が不正であるものとして扱う。

例えば、

```text
判定履歴一覧へ戻る
```

導線を表示してよい。

---

### 2.51 ASSESSMENT_HISTORY_NOT_FOUND

以下の場合は、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

を受信する。

- 判定履歴不存在
- 他利用者所属
- 論理削除済み履歴

React側では、原因を推測せず、

```text
指定された判定履歴が見つかりません。
```

などの共通表示とする。

---

### 2.52 他利用者所属を表示しない

`ASSESSMENT_HISTORY_NOT_FOUND`を受信しても、

```text
他の利用者の判定履歴です。
```

などと表示しない。

API契約上、不存在との区別はできない。

---

### 2.53 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ASM-004専用の処理をPageへ重複実装しない。

---

### 2.54 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、共通利用者コンテキストエラーとして扱う。

---

### 2.55 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択している利用者が有効ではない状態として共通処理する。

---

### 2.56 INTERNAL_SERVER_ERROR

```text
INTERNAL_SERVER_ERROR
```

の場合は、共通サーバーエラーとして扱う。

例えば、

```text
判定履歴を取得できませんでした。
時間をおいて再度お試しください。
```

などを表示する。

---

### 2.57 error.codeで分岐する

React側では、`message`文字列ではなく、

```text
error.code
```

を基準としてエラー処理を行う。

概念例：

```ts
switch (error.code) {
  case 'INVALID_ASSESSMENT_HISTORY_ID':
    // 不正な履歴ID
    break;

  case 'ASSESSMENT_HISTORY_NOT_FOUND':
    // 履歴不存在
    break;

  default:
    // 共通エラー
    break;
}
```

---

### 2.58 判定結果とAPIエラーを混同しない

以下はAPIエラーではない。

```text
ACHIEVABLE
NOT_ACHIEVABLE
NOT_ASSESSABLE
```

これらは、

```text
assessmentHistory.assessmentResult
```

として扱う。

一方、

```text
ASSESSMENT_HISTORY_NOT_FOUND
INTERNAL_SERVER_ERROR
```

などはAPIエラーとして扱う。

---

### 2.59 Componentの責務

判定履歴詳細Componentでは、主に以下を担当する。

- 目的名表示
- 判定対象年月表示
- 判定結果表示
- 判定実行日時表示
- 判定時点の計算根拠表示
- 判定不可理由表示

API通信や利用者境界判定は行わない。

---

### 2.60 CalculationBasis Component

判定根拠の表示量が多い場合は、専用Componentへ分離してよい。

概念例：

```tsx
<AssessmentCalculationBasis
  availableAssetAmount={
    detail.availableAssetAmount
  }
  requiredAmount={
    detail.requiredAmount
  }
  averageNetIncome={
    detail.averageNetIncome
  }
  requiredEmergencyFund={
    detail.requiredEmergencyFund
  }
  remainingAmount={
    detail.remainingAmount
  }
/>
```

---

### 2.61 Result Component

判定結果についても、表示責務を分離してよい。

概念例：

```tsx
<AssessmentResultBadge
  result={
    detail.assessmentResult
  }
/>
```

判定ロジック自体はComponentへ持たせない。

---

### 2.62 Query Hookの責務

Query Hookでは、主に以下を担当する。

```text
ASM-004実行
Query Key管理
Loading状態管理
Error状態管理
Cache管理
```

表示文言や画面レイアウトはQuery Hookへ持たせない。

---

### 2.63 API Clientの責務

API Clientでは、

```text
GET
/api/v1/assessment-histories/{assessmentHistoryId}
```

のHTTP通信と、型付きレスポンス取得を担当する。

以下をAPI Clientへ含めない。

- Toast表示
- 画面遷移
- 金額フォーマット
- 日時フォーマット
- 判定結果表示文言
- Query Cache操作
- 判定結果再計算

---

### 2.64 概念的なディレクトリ構成

例えば、以下のように整理できる。

```text
features/
└── assessments/
    ├── api/
    │   ├── previewAssessment.ts
    │   ├── createAssessment.ts
    │   ├── getAssessmentHistories.ts
    │   └── getAssessmentHistoryDetail.ts
    ├── components/
    │   ├── AssessmentResultBadge.tsx
    │   ├── AssessmentCalculationBasis.tsx
    │   └── AssessmentHistoryDetail.tsx
    ├── hooks/
    │   ├── useAssessmentHistories.ts
    │   └── useAssessmentHistoryDetail.ts
    ├── types/
    │   └── assessment.ts
    └── pages/
        └── AssessmentHistoryDetailPage.tsx
```

正式なディレクトリ構成は、Reactアーキテクチャ設計に従う。

---

### 2.65 ASM系の共通型を再利用する

以下の概念は、ASM系APIで共通化する。

```text
AssessmentResult
NotAssessableReason
```

APIごとに同じUnion Typeを重複定義しない。

---

### 2.66 API Response型と画面用型

Phase1では、ASM-004のレスポンス型をそのまま画面表示へ使用してよい。

将来的に表示用の計算・変換が増えた場合は、

```text
API Response
    ↓
View Model
    ↓
Component
```

へ分離してよい。

現時点では不要な変換層を増やさない。

---

### 2.67 現在情報との混在に注意する

ASM-004は、過去の判定時点を表示するAPIである。

そのため、画面上でも、

```text
判定時点の値
```

であることが分かる表示にする。

現在の資産額等と並べる場合は、

```text
判定時点
現在
```

の区別を明示する。

---

### 2.68 フロントエンドで行わないこと

ASM-004のReact・TypeScript実装では、以下をフロントエンドの責務としない。

- 利用者境界の最終保証
- 判定履歴存在確認の最終保証
- 他利用者所属判定
- 保存済み判定結果の再計算
- 利用可能資産額の再計算
- 平均手取り収入の再計算
- 必要生活防衛資金の再計算
- `remainingAmount`の再計算
- 判定結果の再判定
- 判定不可条件の再判定
- 現在値による過去履歴上書き
- DBカラム値の補完

フロントエンドは、

```text
assessmentHistoryId取得
    ↓
ASM-004実行
    ↓
Loading / Error判定
    ↓
保存済み判定結果取得
    ↓
判定時点の計算根拠表示
```

という責務を基本とする。

---

## 3. 関連ドキュメント

- [ASM-004 API詳細設計](../../../api/details/assessments/asm-004-detail.md)
- [API一覧](../../../api/api-list.md)
- [API共通方針](../../../api/api-common-policy.md)
- [エラーコード一覧](../../../api/error-codes.md)
- [Reactアーキテクチャ共通設計](../react-architecture.md)
- [目的達成判定 Reactアーキテクチャ設計](./README.md)
- [ASM-004 Laravelアーキテクチャ設計](../../../architecture/laravel/assessments/asm-004-detail.md)
- [ASM-004 テスト設計](../../../tests/assessments/asm-004-detail.md)
- [目的達成判定 テスト設計](../../../tests/assessments/README.md)
- [機能要件](../../../requirements/functional-requirements.md)
- [ユビキタス言語集](../../../glossary.md)
- [エンティティ定義](../../../entities.md)
- [テーブル定義書](../../../table-definition.md)
- [ER図](../../../er-diagram-phase1.md)
