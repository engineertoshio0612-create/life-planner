# 目的達成判定 Laravelアーキテクチャ設計

## 1 概要

本ドキュメントでは、
目的達成判定機能をLaravelで実装する際の
共通アーキテクチャおよび責務分離方針を定義する。

目的達成判定機能では、主に以下を扱う。

* 目的達成判定のプレビュー
* 目的達成判定結果の保存
* 目的達成判定履歴の一覧取得
* 目的達成判定履歴の詳細取得

各API固有の実装方針は個別ドキュメントへ分離し、
本ドキュメントではASM系機能に共通する
Laravel側の設計方針を扱う。

---

## 2 対象API

| API ID  | API名        | Laravelアーキテクチャ設計                           |
| ------- | ----------- | ------------------------------------------ |
| ASM-001 | 目的達成判定プレビュー | [asm-001-preview.md](./asm-001-preview.md) |
| ASM-002 | 目的達成判定結果保存  | [asm-002-create.md](./asm-002-create.md)   |
| ASM-003 | 判定履歴一覧取得    | [asm-003-list.md](./asm-003-list.md)       |
| ASM-004 | 判定履歴詳細取得    | [asm-004-detail.md](./asm-004-detail.md)   |

---

## 3 基本方針

目的達成判定機能では、
HTTP処理、アプリケーション処理、
判定ロジック、データアクセス、
レスポンス生成を単一クラスへ集約しない。

概念的には、以下のように責務を分離する。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ↓
Domain / Query / Repository
    ↓
Result DTO
    ↓
API Resource
    ↓
Responder
```

APIごとの特性に応じて、
必要な構成要素だけを使用する。

不要なService、Repository、FormRequestなどを
形式的に追加しない。

---

## 4 Action

Actionは、
HTTPリクエストとUseCaseを橋渡しする。

主に以下を担当する。

* パスパラメータの受け取り
* FormRequest等から検証済み入力の受け取り
* `UserContext`から操作対象利用者IDを取得する
* UseCaseを呼び出す
* Responderへ処理結果を渡す

Actionでは、以下を行わない。

* DB検索
* 業務ルール判定
* 目的達成判定ロジック
* 利用可能資産額の計算
* 平均手取り収入の計算
* レスポンス配列の直接生成
* エラーコードの直接判定

---

## 5 UseCase

UseCaseは、
各APIのアプリケーション処理全体を制御する。

概念的には、

```text
入力
    ↓
必要なデータ取得
    ↓
業務ルール確認
    ↓
Domain処理
    ↓
必要に応じて保存
    ↓
Result DTO
```

という流れを担当する。

UseCaseは、
HTTPレスポンス形式やLaravel固有の表示処理に依存しない。

---

## 6 Domain

目的達成判定そのものの業務ロジックは、
Domainへ配置する。

例えば、以下の処理が対象となる。

* 利用可能資産額の算出
* 平均手取り収入の算出
* 必要支出額の算出
* 必要生活防衛資金の算出
* 目的達成可否の判定
* 判定不可条件の判定
* 判定根拠の生成

ASM-001とASM-002で
同一の目的達成判定ロジックを使用する場合は、
Domain処理を共通化する。

APIごとに同じ判定ロジックを重複実装しない。

---

## 7 Query

参照専用のデータ取得は、
専用Queryへ委譲する。

主な対象は以下とする。

* 目的情報取得
* 月末資産状況取得
* 手取り収入取得
* 判定履歴一覧取得
* 判定履歴詳細取得

概念的には、

```text
Query
    → SELECT
```

の責務に限定する。

Queryでは、
INSERT、UPDATE、DELETEや
目的達成判定そのものを行わない。

---

## 8 Repository

データの登録・更新が必要な場合は、
必要に応じてRepositoryを使用する。

概念的には、

```text
Query
    → 読み取り

Repository
    → 登録・更新・削除
```

と責務を分離する。

ASM-003、ASM-004などの
参照専用GET APIでは、
Repositoryを原則として使用しない。

---

## 9 利用者境界

ASM系APIでは、
操作対象利用者を`X-User-Id`から特定する。

利用者コンテキストの設定は
共通Middlewareで行う。

概念的には、以下とする。

```text
X-User-Id
    ↓
必須確認
    ↓
形式確認
    ↓
users存在確認
    ↓
UserContext
    ↓
Action
```

Action以降では、
検証済みの`UserContext`を利用する。

---

## 10 データ取得時に利用者境界を含める

他利用者のリソースについて、
IDだけで取得した後に所属を確認する方式を基本としない。

可能な限り取得条件そのものへ
利用者境界を含める。

例えば、

```text
objective_id
+
user_id
```

または、

```text
assessment_history_id
+
objectives.user_id
```

などで絞り込む。

他利用者所属であることを
クライアントへ公開しない。

---

## 11 目的達成判定プレビュー

ASM-001では、
現在の入力および資産状態から
目的達成判定を実行する。

ただし、判定結果を
`assessment_histories`へ保存しない。

概念的には、

```text
現在状態取得
    ↓
Domain
    ↓
目的達成判定
    ↓
Result DTO
    ↓
レスポンス
```

とする。

---

## 12 目的達成判定結果保存

ASM-002では、
ASM-001と共通の判定ロジックを使用して
目的達成判定を実行する。

判定結果および
後から判定内容を確認するために必要な計算根拠を
`assessment_histories`へ保存する。

概念的には、

```text
現在状態取得
    ↓
Domain
    ↓
目的達成判定
    ↓
判定結果・計算根拠
    ↓
assessment_histories
```

とする。

---

## 13 保存済み判定履歴

ASM-003およびASM-004では、
ASM-002によって保存された
`assessment_histories`を正とする。

現在の資産状況や
現在の手取り収入を使用して
過去の判定結果を再計算しない。

```text
ASM-002
    ↓
判定時点の結果を保存

ASM-003 / ASM-004
    ↓
保存済み結果を参照
```

---

## 14 判定履歴一覧と詳細

ASM-003では、
判定履歴一覧として必要な情報だけを取得する。

ASM-004では、
特定の判定履歴1件について、
判定結果および判定時点の計算根拠を取得する。

```text
ASM-003
    ↓
判定履歴一覧
    ↓
assessmentHistoryId
    ↓
ASM-004
    ↓
判定履歴詳細
```

一覧取得と詳細取得の責務を分離する。

---

## 15 過去履歴を現在状態から変更しない

判定履歴保存後に、以下が変更されても、

* 目的
* 月末資産状況
* 手取り収入
* 利用可能資産設定

保存済み判定結果そのものを
動的に変更しない。

履歴表示に必要な過去時点の値は、
可能な限り`assessment_histories`へ
保存することを優先する。

---

## 16 Result DTO

UseCaseの処理結果は、
必要に応じて専用Result DTOへ変換する。

Result DTOには、
APIレスポンスまたは後続処理に必要な値だけを保持する。

Eloquent Modelをそのまま保持する構成を基本としない。

概念例：

```text
Eloquent / Query結果
    ↓
UseCase
    ↓
Result DTO
    ↓
API Resource
```

これにより、
データアクセス層とAPI表現を分離する。

---

## 17 API Resource

API Resourceは、
Result DTO等をAPIレスポンス形式へ変換する。

主に以下を担当する。

* `snake_case`から`camelCase`への変換
* `bigint` IDのstring変換
* 日時のAPI共通形式への変換
* nullable値の表現

API Resourceでは、
目的達成判定や金額計算を行わない。

---

## 18 Responder

Responderは、
API Resource等をHTTPレスポンスへ変換する。

主に以下を担当する。

* HTTPステータス
* 共通Envelope
* API Resourceの返却

Responderでは、以下を行わない。

* DBアクセス
* 業務ルール判定
* 利用者境界確認
* 目的達成判定
* 判定根拠計算

---

## 19 IDの扱い

DB上で`bigint`を使用するIDは、
API Resourceで`string`へ変換する。

主な対象は以下とする。

```text
id
objectiveId
assessmentHistoryId
monthEndAssetSnapshotId
```

実際のレスポンス項目は、
各API詳細設計を正とする。

---

## 20 金額の扱い

金額は、日本円整数として扱う。

Laravel側で、

```text
"1,000,000円"
```

のような表示用文字列へ変換しない。

APIレスポンスでは、
integerまたは仕様上必要な場合はnullとして返却する。

表示フォーマットは
React側の責務とする。

---

## 21 Enum・Value Object

目的達成判定結果や
判定不可理由など、
ASM系APIで共有するコード値は
共通EnumまたはValue Objectとして管理してよい。

例えば、

```text
ACHIEVABLE
NOT_ACHIEVABLE
NOT_ASSESSABLE
```

などをAPIごとに
個別定義しない。

ASM-001、ASM-002、ASM-003、ASM-004で
同じ意味・コード値を使用する。

---

## 22 FormRequest

Request BodyやQuery Parameterの
バリデーションが必要なAPIでは
FormRequestを使用してよい。

一方、

```text
Request Bodyなし
Query Parameterなし
```

のGET APIで、
ルートパラメータしか存在しない場合は、
空のFormRequestを形式的に作成しない。

パスパラメータの検証方式は、
API共通方針に従う。

---

## 23 トランザクション

複数の更新を
原子的に保証する必要があるAPIでは
トランザクションを使用する。

参照専用GET APIでは、
原則としてトランザクションを使用しない。

```text
ASM-003
ASM-004
    ↓
原則トランザクションなし
```

ASM-002など、
保存処理を含むAPIについては
個別設計を正とする。

---

## 24 ロック

参照専用APIでは、
`lockForUpdate()`を使用しない。

更新競合への対応が必要な場合のみ、
各APIの業務要件に基づいて使用する。

不要なロックによって
他APIの処理をブロックしない。

---

## 25 エラー処理

業務例外は、
共通Exception Handler等で
API共通エラー形式へ変換する。

ASM系で想定される主なエラーには、
以下がある。

```text
USER_CONTEXT_REQUIRED
INVALID_USER_ID
USER_NOT_FOUND
OBJECTIVE_NOT_FOUND
ASSESSMENT_HISTORY_NOT_FOUND
INTERNAL_SERVER_ERROR
```

API固有のエラーコードは、
各API詳細設計を正とする。

---

## 26 Not Foundの扱い

他利用者所属のリソースは、
原則として不存在と同様に扱う。

例えば、判定履歴の場合は、

```text
対象履歴不存在
他利用者所属
論理削除済み
```

を必要に応じて

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

へ集約する。

他利用者のデータが存在することを
レスポンスから推測できないようにする。

---

## 27 ログ

ログには、必要に応じて以下を設定する。

```text
requestId
userId
apiId
objectiveId
assessmentHistoryId
httpStatus
errorCode
```

APIごとに不要な値は記録しない。

また、通常のアクセスログへ
詳細な資産額や手取り収入などの
業務データを不要に出力しない。

---

## 28 キャッシュ

Phase1では、
ASM系API専用のサーバー側キャッシュを
原則として導入しない。

必要性が明確になった場合にのみ検討する。

クライアント側のQuery Cacheについては、
Reactアーキテクチャ設計を正とする。

---

## 29 テスト方針

ASM系のLaravel実装では、
主に以下を確認する。

* 正常系
* 利用者境界
* バリデーション
* Not Found
* 判定結果
* 判定不可
* 保存済み計算根拠
* ID型変換
* 金額型
* 日時形式
* 副作用
* 現在状態から過去履歴を再計算しないこと

詳細なテストケースは、
目的達成判定テスト設計へ分離する。

---

## 30 概念的なディレクトリ構成

概念的には、以下のように整理できる。

```text
app/
├── Actions/
│   └── Assessments/
├── UseCases/
│   └── Assessments/
├── Domain/
│   └── Assessments/
├── Queries/
│   └── Assessments/
├── Repositories/
│   └── Assessments/
├── DTOs/
│   └── Assessments/
├── Http/
│   └── Resources/
│       └── Assessments/
└── Responders/
    └── Assessments/
```

ただし、
すべてのディレクトリを
すべてのAPIで使用する必要はない。

正式な配置・命名規則は、
Laravelアーキテクチャ共通設計を正とする。

---

## 31 責務分離

目的達成判定機能では、
最終的に以下の責務分離を維持する。

```text
Middleware
    → 利用者コンテキスト等の横断処理

Action
    → HTTP入力とUseCaseの橋渡し

UseCase
    → アプリケーション処理

Domain
    → 目的達成判定の業務ロジック

Query
    → 読み取り

Repository
    → 必要なデータ変更

Result DTO
    → 処理結果表現

API Resource
    → API形式への変換

Responder
    → HTTPレスポンス生成
```

目的達成判定機能では、
各層へ責務を適切に分離し、
同じ判定ロジックや取得処理を
複数APIへ重複実装しない。

---

## 32 本ドキュメントで扱わないこと

本ドキュメントでは、以下の詳細を扱わない。

* ASM各API固有のリクエスト・レスポンス
* 個別のバリデーションルール
* 個別のエラーコード
* 判定計算式の詳細
* テーブル定義
* React側の実装詳細
* 具体的なテストケース

これらは、それぞれのAPI詳細設計、
Reactアーキテクチャ設計、
テーブル定義書、テスト設計を正とする。

---

## 33 関連ドキュメント

* [API一覧](../../../api/api-list.md)
* [API共通方針](../../../api/api-common-policy.md)
* [エラーコード一覧](../../../api/error-codes.md)
* [Laravelアーキテクチャ共通設計](../laravel-architecture.md)
* [ASM-001 Laravelアーキテクチャ設計](./asm-001-preview.md)
* [ASM-002 Laravelアーキテクチャ設計](./asm-002-create.md)
* [ASM-003 Laravelアーキテクチャ設計](./asm-003-list.md)
* [ASM-004 Laravelアーキテクチャ設計](./asm-004-detail.md)
* [目的達成判定 Reactアーキテクチャ設計](../../../architecture/react/assessments/README.md)
* [目的達成判定 テスト設計](../../../tests/assessments/README.md)
* [機能要件](../../../requirements/functional-requirements.md)
* [ユビキタス言語集](../../../glossary.md)
* [エンティティ定義](../../../entities.md)
* [テーブル定義書](../../../table-definition.md)
* [ER図](../../../er-diagram-phase1.md)
