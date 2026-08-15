# 目的達成判定 Reactアーキテクチャ設計

## 1 概要

本ドキュメントでは、
目的達成判定機能をReact・TypeScriptで実装する際の
共通アーキテクチャおよび責務分離方針を定義する。

目的達成判定機能では、主に以下を扱う。

* 目的達成判定のプレビュー
* 目的達成判定結果の保存
* 目的達成判定履歴の一覧表示
* 目的達成判定履歴の詳細表示

各API固有の実装方針は個別ドキュメントへ分離し、
本ドキュメントではASM系機能に共通する
React・TypeScript側の設計方針を扱う。

---

## 2 対象API

| API ID  | API名        | Reactアーキテクチャ設計                             |
| ------- | ----------- | ------------------------------------------ |
| ASM-001 | 目的達成判定プレビュー | [asm-001-preview.md](./asm-001-preview.md) |
| ASM-002 | 目的達成判定結果保存  | [asm-002-create.md](./asm-002-create.md)   |
| ASM-003 | 判定履歴一覧取得    | [asm-003-list.md](./asm-003-list.md)       |
| ASM-004 | 判定履歴詳細取得    | [asm-004-detail.md](./asm-004-detail.md)   |

---

## 3 基本方針

目的達成判定機能では、
フロントエンドで目的達成可否を独自に判定しない。

判定結果および判定根拠は、
バックエンドから返却された値を正とする。

概念的には、以下とする。

```text
React
    ↓
ASM API
    ↓
Laravel
    ↓
目的達成判定
    ↓
判定結果・計算根拠
    ↓
React
    ↓
表示
```

React側では、
API呼び出し、状態管理、表示形式への変換、
画面表示を担当する。

---

## 4 判定ロジックをReactへ実装しない

React側では、以下を再計算しない。

* 利用可能資産額
* 平均手取り収入
* 必要支出額
* 必要生活防衛資金
* 残額
* 目的達成可否
* 判定不可条件

例えば、以下のような判定処理を
フロントエンドへ実装しない。

```ts
const result =
  calculateAssessment(
    availableAssets,
    averageNetIncome,
    objective,
  );
```

目的達成判定の業務ロジックは
バックエンド側の責務とする。

---

## 5 共通型

ASM系APIで共通する型は、
APIごとに重複定義せず共通化する。

概念例：

```ts
export type AssessmentResult =
  | 'ACHIEVABLE'
  | 'NOT_ACHIEVABLE'
  | 'NOT_ASSESSABLE';
```

判定不可理由についても、
共通型として管理する。

```ts
export type NotAssessableReason =
  | 'NET_INCOME_INSUFFICIENT'
  | 'ASSET_DATA_INSUFFICIENT';
```

実際のコード値は、
API詳細設計および目的達成判定の共通仕様を正とする。

---

## 6 IDの扱い

ASM系APIで使用するIDは、
API共通方針に従って`string`として扱う。

主な対象は以下とする。

```text
objectiveId
assessmentHistoryId
```

DB上の型が`bigint`であっても、
React・TypeScript側で`number`へ変換しない。

---

## 7 金額の扱い

金額は日本円整数として
`number`または`number | null`で扱う。

APIから取得した金額を
Component内で直接文字列加工せず、
共通Formatterを利用する。

概念例：

```ts
export const formatYen = (
  value: number,
): string => {
  return new Intl.NumberFormat(
    'ja-JP',
    {
      style: 'currency',
      currency: 'JPY',
    },
  ).format(value);
};
```

`null`は`0円`へ変換せず、
画面設計に従って未設定・算出不可として表示する。

---

## 8 API Client

ASM系APIのHTTP通信は、
専用API Client関数へ分離する。

概念的な構成は以下とする。

```text
Page / Component
    ↓
Query / Mutation Hook
    ↓
API Client
    ↓
共通apiClient
    ↓
Laravel API
```

Componentから直接`fetch`や`axios`を呼び出さない。

また、API Clientでは以下を行わない。

* Toast表示
* 画面遷移
* 判定結果の表示文言生成
* 金額フォーマット
* 判定ロジック
* Query Cache操作

---

## 9 利用者コンテキスト

操作対象利用者は、
API共通方針に従って`X-User-Id`で指定する。

`X-User-Id`は、
共通API Clientから付与する。

各ASM APIの関数へ
`userId`をHTTPリクエスト用の引数として
個別に渡す構成を基本としない。

```text
利用者選択
    ↓
共通利用者コンテキスト
    ↓
共通API Client
    ↓
X-User-Id
    ↓
ASM API
```

---

## 10 TanStack Query

サーバー状態の管理には、
TanStack Queryを利用する。

参照系APIはQueryとして扱う。

```text
ASM-003
ASM-004
    ↓
Query
```

保存などサーバー状態を変更するAPIは
Mutationとして扱う。

```text
ASM-002
    ↓
Mutation
```

ASM-001の扱いについては、
プレビューというAPI固有の性質を踏まえ、
個別設計を正とする。

---

## 11 Query Key

目的達成判定関連のQuery Keyは、
機能単位で共通管理する。

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

一覧と詳細のCacheは分離する。

---

## 12 利用者切替とCache

Phase1では、
操作対象利用者を切り替えてAPIを利用する。

そのため、利用者切替後に
前利用者の判定結果や判定履歴を
誤表示しないことを必須とする。

Cache境界については、

```text
利用者切替時に
ASM系Cacheを破棄する
```

または、

```text
Query KeyへuserIdを含める
```

などの方式を使用する。

正式な方式は、
Reactアーキテクチャ共通設計に従う。

---

## 13 プレビューと保存済み履歴を区別する

ASM-001のプレビュー結果と、
ASM-002によって保存された判定履歴は
異なる状態として扱う。

```text
ASM-001
    ↓
現在状態からプレビュー
    ↓
保存しない

ASM-002
    ↓
判定実行
    ↓
履歴保存

ASM-003 / ASM-004
    ↓
保存済み履歴参照
```

保存済み履歴を表示する際に、
現在の資産情報等を使用して
結果を再計算しない。

---

## 14 判定不可の扱い

`NOT_ASSESSABLE`は、
APIエラーではなく正常な判定結果として扱う。

```text
ACHIEVABLE
NOT_ACHIEVABLE
NOT_ASSESSABLE
    ↓
判定結果
```

一方、

```text
ASSESSMENT_HISTORY_NOT_FOUND
INTERNAL_SERVER_ERROR
```

などはAPIエラーとして扱う。

判定結果とAPIエラーを混同しない。

---

## 15 エラー処理

APIエラーは、
API共通エラー型を使用して処理する。

React側では、
エラーメッセージ文字列ではなく
`error.code`を基準として処理を分岐する。

利用者コンテキストなどの
ASM系APIに限定されないエラーは、
共通エラー処理へ委譲する。

各Pageへ同じエラー処理を重複実装しない。

---

## 16 表示Component

目的達成判定の表示責務は、
必要に応じてComponentへ分離する。

例えば、以下が考えられる。

```text
AssessmentResultBadge
AssessmentCalculationBasis
AssessmentHistoryList
AssessmentHistoryDetail
```

Componentでは表示を担当し、
目的達成判定そのものを行わない。

---

## 17 概念的なディレクトリ構成

概念的には、以下のように整理する。

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
    │   ├── AssessmentHistoryList.tsx
    │   └── AssessmentHistoryDetail.tsx
    ├── hooks/
    │   ├── useAssessmentHistories.ts
    │   └── useAssessmentHistoryDetail.ts
    ├── types/
    │   └── assessment.ts
    └── pages/
        ├── AssessmentHistoryListPage.tsx
        └── AssessmentHistoryDetailPage.tsx
```

正式なディレクトリ構成は、
Reactアーキテクチャ共通設計に従う。

---

## 18 責務分離

目的達成判定機能では、
概念的に以下の責務分離を維持する。

```text
Page
    → 画面単位の状態・表示制御

Component
    → UI表示

Query / Mutation Hook
    → Server State管理

API Client
    → HTTP通信

共通apiClient
    → X-User-Id等の共通HTTP処理

Laravel API
    → 業務処理・目的達成判定
```

React側へバックエンドの
目的達成判定ロジックを複製しない。

---

## 19 本ドキュメントで扱わないこと

本ドキュメントでは、以下の詳細を扱わない。

* ASM各API固有のリクエスト・レスポンス
* 目的達成判定の計算ロジック
* Laravel側の実装詳細
* DBアクセス方式
* テーブル設計
* 具体的な画面レイアウト
* テストケースの詳細

これらは、それぞれのAPI詳細設計、
Laravelアーキテクチャ設計、
画面設計、テスト設計を正とする。

---

## 20 関連ドキュメント

* [API一覧](../../../api/api-list.md)
* [API共通方針](../../../api/api-common-policy.md)
* [エラーコード一覧](../../../api/error-codes.md)
* [Reactアーキテクチャ共通設計](../react-architecture.md)
* [ASM-001 Reactアーキテクチャ設計](./asm-001-preview.md)
* [ASM-002 Reactアーキテクチャ設計](./asm-002-create.md)
* [ASM-003 Reactアーキテクチャ設計](./asm-003-list.md)
* [ASM-004 Reactアーキテクチャ設計](./asm-004-detail.md)
* [目的達成判定 Laravelアーキテクチャ設計](../../../architecture/laravel/assessments/README.md)
* [目的達成判定 テスト設計](../../../tests/assessments/README.md)
* [機能要件](../../../requirements/functional-requirements.md)
* [ユビキタス言語集](../../../glossary.md)
* [エンティティ定義](../../../entities.md)
* [テーブル定義書](../../../table-definition.md)
* [ER図](../../../er-diagram-phase1.md)
