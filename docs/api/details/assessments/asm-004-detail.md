# ASM-004 判定履歴詳細取得

## 1. 概要

操作対象となる利用者に紐づく指定された目的達成判定履歴について、判定結果および判定実行時点の計算根拠を取得する。

ASM-004では、`assessment_histories`に保存された判定結果を中心として、関連する目的情報および判定対象となった月末資産状況の情報を組み合わせて返却する。

主な取得対象は、以下とする。

- 判定履歴ID
- 目的ID
- 目的名
- 判定結果
- 判定実行日時
- 判定対象となった月末資産状況
- 判定時点の利用可能資産額
- 判定時点の手取り収入
- 必要支出額
- 必要生活防衛資金
- 判定根拠

概念的には、以下の関係から判定履歴詳細を取得する。

```text
assessmentHistoryId
    ↓
assessment_histories
    ├─ objective_id
    │    ↓
    │  objectives
    │
    └─ month_end_asset_snapshot_id
         ↓
       month_end_asset_snapshots
```

ASM-004は、保存済み判定履歴を参照するためのGET APIとする。

---

### 1.1 保存済み判定履歴を取得する

ASM-004では、ASM-002で保存された

```text
assessment_histories
```

の内容を取得する。

判定詳細表示時に現在の資産状況や現在の手取り収入から判定結果を再計算しない。

---

### 1.2 判定時点の結果を保持する

判定履歴は、

```text
判定を実行した時点で
どのような条件から
どの結果になったか
```

を後から確認するための履歴である。

そのため、目的、資産状況、手取り収入などが後から変更されても、保存済み判定履歴の計算結果を動的に変更しない。

---

### 1.3 ASM-001との違い

ASM-001 目的達成判定プレビューは、判定結果を保存せず、現在の入力・資産状態から判定結果を算出する。

一方、ASM-004は、すでに保存された判定履歴を参照する。

概念的には、

```text
ASM-001
    ↓
現在状態から算出
    ↓
保存しない

ASM-004
    ↓
保存済みassessment_histories
    ↓
過去の判定結果を参照
```

とする。

---

### 1.4 ASM-002との違い

ASM-002は、目的達成判定を実行し、結果と計算根拠を`assessment_histories`へ保存する。

ASM-004は、その保存済み結果を取得する。

```text
ASM-002
    ↓
判定実行
    ↓
assessment_histories INSERT

ASM-004
    ↓
assessment_histories SELECT
```

と責務を分離する。

---

### 1.5 ASM-003との違い

ASM-003は、操作対象利用者の判定履歴一覧を取得する。

ASM-004は、一覧から選択された1件の判定履歴について、より詳細な計算根拠まで取得する。

概念的には、

```text
ASM-003
    ↓
判定履歴一覧
    ↓
assessmentHistoryId選択
    ↓
ASM-004
    ↓
判定履歴詳細
```

とする。

---

### 1.6 現在の目的状態に依存しない

判定履歴に紐づく目的が現在無効化されていても、保存済み判定履歴は過去情報として参照できる。

例えば、

```text
2026-08
目的A
enabled = true
    ↓
ASM-002
判定履歴保存

2027-01
目的A
enabled = false
    ↓
ASM-004
過去の判定履歴を参照可能
```

とする。

目的の現在の`enabled`状態だけを理由に履歴参照を拒否しない。

---

### 1.7 現在の資産状況から再計算しない

ASM-004では、現在の

- 月末資産残高
- 商品別月末評価額
- 利用可能資産設定
- 手取り収入

を再取得して判定結果を再計算しない。

ASM-004の正は、ASM-002実行時に保存された判定履歴とする。

---

### 1.8 参照専用APIとする

ASM-004では、以下を行わない。

- 目的達成判定の再実行
- 判定履歴の保存
- 判定履歴の更新
- 判定履歴の削除
- 目的情報の更新
- 月末資産状況の更新
- 手取り収入の更新

保存済み判定履歴の詳細取得だけに責務を限定する。

---

## 2. ユースケース

利用者は、過去に実行・保存した目的達成判定について、

```text
なぜその判定結果になったのか
```

を確認するためにASM-004を使用する。

主な利用例は、以下とする。

- 判定履歴一覧から特定履歴を選択する
- 過去の判定結果を確認する
- 判定対象だった目的を確認する
- 判定対象となった月末資産状況を確認する
- 利用可能資産額を確認する
- 手取り収入を確認する
- 必要支出額を確認する
- 生活防衛資金を含む判定根拠を確認する

---

### 2.1 判定履歴一覧から詳細表示する

主な利用フローは、以下とする。

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
判定結果・計算根拠表示
```

---

### 2.2 過去判定の根拠確認

例えば、過去の判定が

```text
判定結果
達成可能
```

だった場合に、

```text
利用可能資産
必要支出額
生活防衛資金
手取り収入
```

など、その結果に至った計算根拠を確認できる。

---

### 2.3 現在値との比較には使用しない

ASM-004は過去判定の詳細表示を目的とする。

そのため、

```text
現在の資産状況
VS
過去判定時の資産状況
```

という比較機能をASM-004自身には持たせない。

比較表示が必要になった場合は、画面側で別APIの現在値と組み合わせる、または専用APIを検討する。

---

### 2.4 再判定はASM-002を使用する

過去判定履歴を確認した後に現在の条件で再判定したい場合は、ASM-004ではなくASM-002を使用する。

概念的には、

```text
ASM-004
過去判定確認
    ↓
現在条件で再判定したい
    ↓
ASM-002
新しい判定履歴保存
```

とする。

過去履歴を上書きしない。

---

## 3. エンドポイント

```http
GET /api/v1/assessment-histories/{assessmentHistoryId}
```

`assessmentHistoryId`には、取得対象となる判定履歴IDを指定する。

例：

```http
GET /api/v1/assessment-histories/100
```

---

### 3.1 assessment-historiesをトップレベルリソースとする理由

判定履歴は、目的から生成される情報ではあるが、保存後は

```text
過去に実行された判定
```

という独立した参照対象となる。

そのため、

```http
GET /api/v1/objectives/{objectiveId}/assessment-histories/{assessmentHistoryId}
```

ではなく、

```http
GET /api/v1/assessment-histories/{assessmentHistoryId}
```

とする。

---

### 3.2 ASM-003とのURL整合性

ASM-003は、

```http
GET /api/v1/assessment-histories
```

で判定履歴一覧を取得する。

ASM-004は、その配下の個別リソースとして、

```http
GET /api/v1/assessment-histories/{assessmentHistoryId}
```

を使用する。

概念的には、

```text
assessment-histories
    ↓
一覧
    ↓
assessmentHistoryId
    ↓
詳細
```

とする。

---

### 3.3 objectiveIdをURLへ含めない理由

判定履歴は、`assessmentHistoryId`から紐づく目的を一意に特定できる。

概念的には、

```text
assessmentHistoryId
    ↓
assessment_histories
    ↓
objective_id
    ↓
objectives
```

となる。

そのため、

```text
assessmentHistoryId
+
objectiveId
```

をURLで二重指定させない。

---

## 4. HTTPメソッド

```http
GET
```

ASM-004は、保存済み判定履歴の参照専用APIであるため、`GET`を使用する。

---

### 4.1 副作用を持たせない

ASM-004実行によって、以下を行わない。

```text
INSERT
UPDATE
DELETE
```

主に、

```text
assessment_histories
objectives
month_end_asset_snapshots
```

を参照する。

---

### 4.2 GETで再判定しない

ASM-004実行時に、現在のデータを使用して

```text
目的達成判定を再計算
```

しない。

GET実行によって判定結果が変化する設計にはしない。

---

### 4.3 GETで履歴を補正しない

保存済み判定履歴に何らかの不整合が存在しても、ASM-004で以下を行わない。

- 判定結果の修正
- 計算根拠の再作成
- 関連目的の変更
- 月末資産状況の変更

---

## 5. 認証・利用者の扱い

Phase1では、認証機能を実装しない。

操作対象となる利用者は、`X-User-Id`リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

指定された利用者に属する判定履歴のみ取得できる。

---

### 5.1 利用者コンテキスト

`X-User-Id`は、API共通方針に従って検証する。

概念的には、

```text
X-User-Id
    ↓
必須確認
    ↓
形式確認
    ↓
利用者存在確認
    ↓
UserContext設定
    ↓
ASM-004
```

とする。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 5.2 利用者境界

判定履歴に直接`user_id`を保持している場合は、概念的に以下の条件で取得する。

```text
assessment_histories.id
    = assessmentHistoryId

AND

assessment_histories.user_id
    = 操作対象利用者ID
```

一方、`assessment_histories`に直接`user_id`を持たない場合は、関連する目的または月末資産状況を介して利用者境界を保証する。

例えば、

```text
assessment_histories.objective_id
    ↓
objectives.user_id
    =
操作対象利用者ID
```

とする。

実際の取得方式は、テーブル定義を正とする。

---

### 5.3 他利用者の判定履歴

例えば、

```text
User A
    Assessment History ID = 100

User B
    X-User-Id = 2
```

という状態で、User Bが

```http
GET /api/v1/assessment-histories/100
```

を実行した場合は、User Aの判定履歴を返却しない。

対象判定履歴が存在しないものとして扱う。

概念的には、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

とする。

---

### 5.4 他利用者所属を公開しない

他利用者に属する判定履歴を指定した場合に、

```text
ASSESSMENT_HISTORY_FORBIDDEN
OTHER_USER_ASSESSMENT_HISTORY
```

のような専用エラーを返却しない。

不存在との区別をクライアントへ公開しない。

---

### 5.5 目的が現在無効化済みでも参照可能とする

判定履歴に紐づく目的が現在

```text
enabled = false
```

であっても、過去に保存された判定履歴は参照可能とする。

現在の目的利用状態をASM-004の取得可否条件にしない。

---

### 5.6 月末資産状況が確定解除されても履歴を保持する

ASM-002実行後に、関連する月末資産状況が確定解除されたとしても、保存済みの判定履歴そのものを自動削除しない設計であれば、ASM-004ではその履歴を参照可能とする。

保存済み判定履歴は、判定実行時点の記録として扱う。

---

### 5.7 userIdをRequestから受け付けない

利用者IDは、

```text
X-User-Id
```

からのみ取得する。

以下のようなクエリパラメータは使用しない。

```text
?userId=1
```

また、Request Bodyも使用しない。

---

## 6. パスパラメータ

ASM-004では、取得対象となる判定履歴を指定するため、以下のパスパラメータを使用する。

| パラメータ | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `assessmentHistoryId` | string | ○ | 取得対象となる判定履歴ID |

エンドポイントは、以下とする。

```http
GET /api/v1/assessment-histories/{assessmentHistoryId}
```

---

### 6.1 assessmentHistoryId

`assessmentHistoryId`には、詳細取得する判定履歴IDを指定する。

例：

```http
GET /api/v1/assessment-histories/100
```

API上ではIDをstringとして扱うが、値としては正の整数形式を前提とする。

正常例：

```text
1
100
999
```

不正例：

```text
0
-1
abc
1.5
1e3
100abc
```

---

### 6.2 assessmentHistoryIdだけで取得しない

`assessmentHistoryId`だけを条件として他利用者の判定履歴まで取得可能な実装にはしない。

操作対象利用者との利用者境界を含めて取得する。

概念的には、

```text
assessmentHistoryId
+
UserContext.userId
```

によって対象判定履歴を確定する。

---

### 6.3 objectiveIdをパスへ含めない

以下のようなURLにはしない。

```http
GET /api/v1/objectives/{objectiveId}/assessment-histories/{assessmentHistoryId}
```

判定履歴から目的IDを取得できるためである。

```text
assessmentHistoryId
    ↓
assessment_histories.objective_id
```

とし、識別情報を二重指定させない。

---

### 6.4 snapshotIdをパスへ含めない

判定対象となった月末資産状況も、判定履歴から特定できる。

そのため、

```http
GET /api/v1/month-end-asset-snapshots/{snapshotId}/assessment-histories/{assessmentHistoryId}
```

のようなURLにはしない。

概念的には、

```text
assessmentHistoryId
    ↓
assessment_histories
    ↓
month_end_asset_snapshot_id
```

とする。

---

## 7. クエリパラメータ

ASM-004では、クエリパラメータを使用しない。

取得対象となる判定履歴は、

```text
assessmentHistoryId
```

によって一意に指定する。

そのため、以下のようなクエリパラメータは定義しない。

```text
?objectiveId=10
?snapshotId=20
?userId=1
?includeDetails=true
```

---

### 7.1 objectiveIdを指定しない

目的IDは、判定履歴から取得できる。

概念的には、

```text
assessmentHistoryId
    ↓
assessment_histories
    ↓
objective_id
```

となる。

そのため、

```http
GET /api/v1/assessment-histories/100?objectiveId=10
```

のような二重指定は行わない。

---

### 7.2 snapshotIdを指定しない

判定対象となった月末資産状況も、判定履歴から特定できる。

概念的には、

```text
assessmentHistoryId
    ↓
assessment_histories
    ↓
month_end_asset_snapshot_id
```

となる。

そのため、

```text
?snapshotId=
```

は使用しない。

---

### 7.3 userIdを指定しない

操作対象利用者は、

```text
X-User-Id
```

によって決定する。

そのため、

```text
?userId=
```

を使用しない。

利用者コンテキストの指定方法をAPIごとに変更しない。

---

### 7.4 include系パラメータを設けない

ASM-004は、判定履歴詳細を取得するAPIであるため、

```text
?includeObjective=true
?includeSnapshot=true
?includeCalculationBasis=true
```

のような取得項目切替用パラメータはPhase1では設けない。

判定履歴詳細画面に必要な判定結果および計算根拠を固定レスポンスとして返却する。

---

## 8. リクエストヘッダー

ASM-004では、API共通方針に従って以下のリクエストヘッダーを使用する。

| ヘッダー | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | レスポンス形式。`application/json`を指定する |

Request Bodyは使用しない。

---

### 8.1 X-User-Id

操作対象となる利用者を指定する。

例：

```http
X-User-Id: 1
```

ASM-004では、指定された利用者に属する判定履歴のみ取得可能とする。

---

### 8.2 X-User-Idの責務

`X-User-Id`は、単なる検索条件ではなく、Phase1における操作対象利用者を決定する利用者コンテキストとして扱う。

概念的には、

```text
X-User-Id
    ↓
利用者コンテキスト設定
    ↓
assessmentHistoryId
    ↓
利用者境界を含めて判定履歴取得
```

とする。

---

### 8.3 X-User-Idをレスポンス取得条件へ反映する

例えば、

```text
X-User-Id = 1
assessmentHistoryId = 100
```

の場合、判定履歴IDが100であるだけではなく、その判定履歴が利用者1に属していることを確認する。

他利用者に属する場合は、判定履歴を返却しない。

---

### 8.4 Accept

レスポンス形式として、

```http
Accept: application/json
```

を指定する。

ASM-004専用のメディアタイプは定義しない。

---

### 8.5 Content-Type

ASM-004ではRequest Bodyを送信しないため、

```http
Content-Type: application/json
```

を必須としない。

---

### 8.6 X-Request-Id

API共通方針でクライアントからの`X-Request-Id`指定を許可している場合は、その共通仕様に従う。

ASM-004専用のリクエストID仕様は定義しない。

---

## 9. リクエストBody

ASM-004では、Request Bodyを使用しない。

```http
GET /api/v1/assessment-histories/{assessmentHistoryId}
```

に対して、Bodyなしでリクエストする。

例：

```http
GET /api/v1/assessment-histories/100
X-User-Id: 1
Accept: application/json
```

---

### 9.1 GETでBodyを使用しない

以下のようなRequest Bodyは送信しない。

```json
{
  "assessmentHistoryId": "100"
}
```

`assessmentHistoryId`は、パスパラメータとして指定する。

---

### 9.2 userIdをBodyへ含めない

以下のようなRequest Bodyも使用しない。

```json
{
  "userId": "1"
}
```

操作対象利用者は、

```text
X-User-Id
```

によって決定する。

---

### 9.3 objectiveIdをBodyへ含めない

以下のようなRequest Bodyも使用しない。

```json
{
  "objectiveId": "10"
}
```

目的は、指定された判定履歴から特定する。

---

### 9.4 snapshotIdをBodyへ含めない

以下のようなRequest Bodyも使用しない。

```json
{
  "snapshotId": "20"
}
```

判定対象となった月末資産状況は、判定履歴から特定する。

---

### 9.5 再判定用データを受け付けない

ASM-004では、以下のような判定条件をRequest Bodyから受け付けない。

```text
利用可能資産額
手取り収入
必要支出額
生活防衛資金
```

これらを指定して判定を再計算するAPIではない。

再判定が必要な場合は、ASM-002を使用する。

---

## 10. リクエスト項目

ASM-004ではRequest Bodyを使用しないため、Body上のリクエスト項目は存在しない。

ASM-004へ入力される主要な値を整理すると、以下となる。

| 項目 | 入力元 | 型 | 必須 | 説明 |
|---|---|---|:---:|---|
| `assessmentHistoryId` | パスパラメータ | string | ○ | 取得対象となる判定履歴ID |
| `X-User-Id` | リクエストヘッダー | string | ○ | 操作対象となる利用者ID |
| `Accept` | リクエストヘッダー | string | ○ | レスポンス形式。`application/json` |

---

### 10.1 assessmentHistoryId

取得対象となる判定履歴IDを指定する。

```http
GET /api/v1/assessment-histories/100
```

API上では`string`として扱う。

値としては、正の整数形式を前提とする。

---

### 10.2 X-User-Id

操作対象となる利用者IDを指定する。

```http
X-User-Id: 1
```

`assessmentHistoryId`と組み合わせて、利用者境界を保証する。

---

### 10.3 Accept

レスポンス形式として、

```http
Accept: application/json
```

を指定する。

---

### 10.4 判定条件を入力項目としない

ASM-004では、以下をリクエスト項目として受け付けない。

- 目的金額
- 利用可能資産額
- 手取り収入
- 平均手取り収入
- 必要支出額
- 生活防衛資金
- 判定結果
- 判定対象年月
- 判定実行日時

これらは、クライアントが指定する値ではなく、保存済み判定履歴から取得する情報とする。

---

### 10.5 現在値を入力として受け付けない

ASM-004は、過去に保存された判定履歴を再現・表示するためのAPIである。

そのため、

```text
現在の資産状況
現在の手取り収入
現在の利用可能資産設定
```

をRequestとして受け取り、保存済み判定結果へ反映することはしない。

ASM-004の入力は、

```text
assessmentHistoryId
+
利用者コンテキスト
```

を基本とする。

---

## 11. バリデーション

ASM-004では、Request Bodyおよびクエリパラメータを使用しない。

そのため、主なバリデーション対象は、

```text
X-User-Id
assessmentHistoryId
Accept
```

とする。

概念的な検証順序は、以下とする。

```text
Request
    ↓
X-User-Id必須確認
    ↓
X-User-Id形式確認
    ↓
利用者存在確認
    ↓
assessmentHistoryId形式確認
    ↓
判定履歴取得
    ↓
利用者境界確認
    ↓
判定履歴詳細取得
```

---

### 11.1 バリデーション対象

ASM-004で検証する主な項目は、以下とする。

| 項目 | 入力元 | 検証内容 |
|---|---|---|
| `X-User-Id` | リクエストヘッダー | 必須、正の整数形式、利用者存在確認 |
| `assessmentHistoryId` | パスパラメータ | 必須、正の整数形式 |
| `Accept` | リクエストヘッダー | API共通方針に従う |

---

### 11.2 X-User-Id必須チェック

`X-User-Id`は、Phase1における操作対象利用者を決定するため、必須とする。

正常例：

```http
X-User-Id: 1
```

未指定の場合は、

```text
USER_CONTEXT_REQUIRED
```

として扱う。

判定履歴検索へは進まない。

---

### 11.3 X-User-Id形式チェック

`X-User-Id`は、正の整数形式を必須とする。

正常例：

```text
1
10
999
```

不正例：

```text
0
-1
abc
1.5
1e3
10abc
```

形式不正の場合は、

```text
INVALID_USER_ID
```

として扱う。

---

### 11.4 利用者存在チェック

形式上正しい`X-User-Id`であっても、対象利用者が存在することを確認する。

概念的には、

```text
X-User-Id
    ↓
users
    ↓
存在確認
```

とする。

利用者が存在しない場合は、

```text
USER_NOT_FOUND
```

として扱う。

---

### 11.5 論理削除済み利用者

`users`でSoftDeletesを採用している場合、論理削除済み利用者は通常の利用者として扱わない。

例えば、

```text
users.id = 1
deleted_at = 2026-08-01 10:00:00
```

の場合は、

```text
USER_NOT_FOUND
```

として扱う。

---

### 11.6 assessmentHistoryId必須チェック

`assessmentHistoryId`は、パスパラメータとして必須とする。

エンドポイント：

```http
GET /api/v1/assessment-histories/{assessmentHistoryId}
```

`assessmentHistoryId`が存在しないURLは、ASM-004のRoute自体にマッチしない。

そのため、通常はLaravelのルーティング処理によって404となる。

---

### 11.7 assessmentHistoryId形式チェック

`assessmentHistoryId`は、正の整数形式を必須とする。

正常例：

```text
1
100
999
```

不正例：

```text
0
-1
abc
1.5
1e3
100abc
```

形式不正の場合は、

```text
INVALID_ASSESSMENT_HISTORY_ID
```

として扱う。

---

### 11.8 assessmentHistoryIdの型

API上では、

```text
assessmentHistoryId
```

をstringとして受け取る。

ただし、DB上の主キーが`bigint`であるため、値としては正の整数形式だけを許可する。

概念的には、

```text
HTTP
"100"
    ↓
形式検証
    ↓
100
    ↓
Query
```

とする。

---

### 11.9 assessmentHistoryId = 0

以下は不正とする。

```http
GET /api/v1/assessment-histories/0
```

`0`は有効な判定履歴IDとして扱わない。

```text
INVALID_ASSESSMENT_HISTORY_ID
```

とする。

---

### 11.10 負数

以下も不正とする。

```http
GET /api/v1/assessment-histories/-1
```

負数を判定履歴IDとして許可しない。

```text
INVALID_ASSESSMENT_HISTORY_ID
```

として扱う。

---

### 11.11 小数

以下も不正とする。

```http
GET /api/v1/assessment-histories/1.5
```

判定履歴IDは整数であるため、

```text
INVALID_ASSESSMENT_HISTORY_ID
```

として扱う。

---

### 11.12 指数表記

以下も不正とする。

```http
GET /api/v1/assessment-histories/1e3
```

数値として解釈可能であっても、ID表現として指数表記を許可しない。

```text
INVALID_ASSESSMENT_HISTORY_ID
```

として扱う。

---

### 11.13 数字と文字列の混在

以下も不正とする。

```http
GET /api/v1/assessment-histories/100abc
```

部分的に数値として解釈しない。

```text
INVALID_ASSESSMENT_HISTORY_ID
```

として扱う。

---

### 11.14 判定履歴存在チェック

`assessmentHistoryId`の形式が正常であっても、対象判定履歴が存在することを確認する。

概念的には、

```text
assessmentHistoryId
+
UserContext.userId
    ↓
assessment_histories
    ↓
対象判定履歴取得
```

とする。

対象を取得できない場合は、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

として扱う。

---

### 11.15 利用者境界を含めて存在確認する

判定履歴の存在確認では、

```text
assessmentHistoryId
```

だけで検索した後に利用者を比較する方式を基本としない。

取得時点から利用者境界を含める。

`assessment_histories`が`user_id`を保持する場合は、概念的に、

```text
assessment_histories.id
    = assessmentHistoryId

AND

assessment_histories.user_id
    = UserContext.userId
```

とする。

---

### 11.16 assessment_historiesにuser_idを持たない場合

`assessment_histories`に直接`user_id`を持たない設計の場合は、関連する`objectives`などを介して利用者境界を確認する。

概念的には、

```text
assessment_histories
    ↓ objective_id
objectives
    ↓ user_id
users
```

として、

```text
objectives.user_id
    = UserContext.userId
```

を取得条件へ含める。

実際の取得条件は、テーブル定義を正とする。

---

### 11.17 他利用者の判定履歴

指定された`assessmentHistoryId`が存在しても、他利用者に属する場合は取得不可とする。

例えば、

```text
assessmentHistoryId = 100

履歴の所有者
    = User 1

X-User-Id
    = 2
```

の場合は、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

として扱う。

---

### 11.18 他利用者所属専用エラーを設けない

他利用者に属する場合でも、

```text
ASSESSMENT_HISTORY_FORBIDDEN
OTHER_USER_ASSESSMENT_HISTORY
```

のような専用エラーコードは使用しない。

以下を同一の扱いとする。

```text
判定履歴不存在
        ↓
ASSESSMENT_HISTORY_NOT_FOUND

他利用者所属
        ↓
ASSESSMENT_HISTORY_NOT_FOUND
```

これにより、他利用者の判定履歴の存在をクライアントへ公開しない。

---

### 11.19 判定履歴の論理削除

`assessment_histories`でSoftDeletesを採用する場合は、論理削除済み判定履歴を通常取得対象に含めない。

その場合、

```text
assessment_histories.deleted_at
    IS NOT NULL
```

の履歴は、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

として扱う。

SoftDeletesを採用しない場合は、この検証は不要とする。

---

### 11.20 目的の現在状態をバリデーションしない

判定履歴に紐づく目的が現在無効化されていても、ASM-004の取得エラーとはしない。

例えば、

```text
assessment_histories
    ↓
objective_id = 10

objectives.id = 10
enabled = false
```

であっても、保存済み判定履歴は取得可能とする。

以下のような検証は行わない。

```text
objective.enabled = true ?
```

---

### 11.21 月末資産状況の確定状態をバリデーションしない

判定履歴に紐づく月末資産状況の現在の

```text
confirmed
```

を、ASM-004の取得可否条件にしない。

例えば、判定保存後に確定解除され、

```text
confirmed = false
```

となっていても、保存済み判定履歴を参照可能とする。

---

### 11.22 現在の判定条件を再検証しない

ASM-004では、以下の現在値について判定可能条件を再検証しない。

- 利用可能資産設定
- 月末残高
- 商品別月末評価額
- 手取り収入
- 目的金額
- 必要支出額
- 生活防衛資金

これらはASM-002実行時に判定済みである。

ASM-004は保存済み結果の参照だけを行う。

---

### 11.23 保存済み計算根拠を現在値で検証し直さない

例えば、判定履歴に

```text
availableAssets = 2,000,000
```

が保存されており、現在の利用可能資産額が

```text
2,500,000
```

になっていても、ASM-004では

```text
2,000,000
```

を過去の判定根拠として扱う。

現在値との差異をバリデーションエラーにしない。

---

### 11.24 関連データの整合性

正常なデータでは、判定履歴に保存された関連IDについて参照整合性が保証されていることを前提とする。

概念的には、

```text
assessment_histories.objective_id
    ↓
objectives.id

assessment_histories.month_end_asset_snapshot_id
    ↓
month_end_asset_snapshots.id
```

の関係をDBの外部キー制約で保証する。

---

### 11.25 関連データ不存在を利用者入力エラーにしない

DB不整合等によって、

```text
assessment_histories
    ↓
objective_id
    ↓
objectives不存在
```

のような通常発生しない状態が存在しても、

```text
INVALID_OBJECTIVE_ID
```

などの利用者入力エラーとして扱わない。

ASM-004へのRequest自体が原因ではないためである。

内部データ不整合として扱う。

---

### 11.26 Accept

`Accept`は、API共通方針に従って検証する。

ASM-004では、

```http
Accept: application/json
```

を使用する。

ASM-004専用のメディアタイプ検証は定義しない。

---

### 11.27 Request Bodyのバリデーション

ASM-004ではRequest Bodyを使用しないため、Body項目に対するバリデーションルールは存在しない。

例えば、

```text
objectiveId
snapshotId
availableAssets
netIncome
requiredAmount
```

などをRequest Bodyから検証する処理は行わない。

---

### 11.28 クエリパラメータのバリデーション

ASM-004ではクエリパラメータを使用しない。

そのため、

```text
page
perPage
sort
order
objectiveId
snapshotId
```

などに対するASM-004固有のバリデーションは定義しない。

---

### 11.29 FormRequest

Request Bodyおよびクエリパラメータが存在しないため、ASM-004専用のFormRequestは原則として作成しない。

以下のような空FormRequestを形式的に作成しない。

```php
final class
    ShowAssessmentHistoryRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [];
    }
}
```

`assessmentHistoryId`は、RouteまたはAPI共通のパスパラメータ検証方式で検証する。

---

### 11.30 Route制約

Laravelでは、`assessmentHistoryId`へRoute制約を設定してよい。

概念例：

```php
Route::get(
    '/api/v1/assessment-histories/{assessmentHistoryId}',
    ShowAssessmentHistoryAction::class,
)
    ->where(
        'assessmentHistoryId',
        '[1-9][0-9]*',
    );
```

ただし、Route不一致による404と

```text
INVALID_ASSESSMENT_HISTORY_ID
```

を明確に区別する必要がある場合は、Route制約だけで完結させず、共通パスパラメータ検証方式に従う。

---

### 11.31 バリデーションと業務上の存在確認を分離する

ASM-004では、以下を区別する。

```text
形式上不正
    ↓
バリデーションエラー

形式上正常
+
対象不存在
    ↓
Not Found
```

例えば、

```text
assessmentHistoryId = abc
    ↓
INVALID_ASSESSMENT_HISTORY_ID

assessmentHistoryId = 999
形式正常
対象履歴なし
    ↓
ASSESSMENT_HISTORY_NOT_FOUND
```

とする。

---

### 11.32 検証順序

ASM-004の基本的な検証順序は、以下とする。

```text
1. X-User-Id必須確認
        ↓
2. X-User-Id形式確認
        ↓
3. 利用者存在確認
        ↓
4. assessmentHistoryId形式確認
        ↓
5. assessmentHistoryId
   + userIdで履歴取得
        ↓
6. 判定履歴不存在確認
        ↓
7. 判定履歴詳細取得
```

---

### 11.33 バリデーション結果とエラーコード

主な検証結果とエラーコードの対応は、以下とする。

| 検証内容 | エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `assessmentHistoryId`形式不正 | `INVALID_ASSESSMENT_HISTORY_ID` |
| 判定履歴不存在 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| 他利用者所属 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| 論理削除済み判定履歴 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| 関連データの想定外不整合 | `INTERNAL_SERVER_ERROR` |

---

### 11.34 バリデーションで行わないこと

ASM-004のバリデーションでは、以下を行わない。

- 目的達成判定の再実行
- 現在の利用可能資産額の再計算
- 現在の手取り収入確認
- 月末残高の入力完了確認
- 商品別月末評価額の入力完了確認
- 月末資産状況の確定可否判定
- 目的の現在の有効状態による取得制限
- 判定履歴の計算根拠再生成
- 保存済み判定結果と現在値の一致確認
- 判定履歴の自動補正

ASM-004では、

```text
正しい利用者コンテキスト
+
正しいassessmentHistoryId
+
操作対象利用者が参照可能な判定履歴
```

であることの確認を中心とする。

---

## 12. 業務ルール

ASM-004では、操作対象利用者に属する指定された判定履歴について、判定実行時に保存された判定結果および計算根拠を取得する。

ASM-004は、過去の判定結果を再現・確認するための参照APIであり、現在の資産状況から目的達成判定を再実行しない。

基本的な処理は、以下とする。

```text
assessmentHistoryId
+
UserContext.userId
    ↓
判定履歴取得
    ↓
利用者境界確認
    ↓
関連する目的情報取得
    ↓
関連する月末資産状況取得
    ↓
保存済み判定結果・計算根拠取得
    ↓
判定履歴詳細として返却
```

---

### 12.1 保存済み判定履歴を正とする

ASM-004では、

```text
assessment_histories
```

に保存されている判定結果および計算根拠を正とする。

ASM-004実行時点の現在データから判定結果を再計算しない。

---

### 12.2 ASM-002実行時点の情報を表示する

ASM-002で判定結果を保存した時点の

- 判定結果
- 判定対象となった目的
- 判定対象となった月末資産状況
- 利用可能資産額
- 手取り収入
- 必要支出額
- 生活防衛資金
- その他の判定計算根拠

を、ASM-004で確認できるようにする。

具体的な保存項目は、`assessment_histories`のテーブル定義を正とする。

---

### 12.3 現在値で上書きしない

判定履歴保存後に現在値が変更されても、過去履歴へ反映しない。

例えば、

```text
2026-08
ASM-002実行

利用可能資産額
2,000,000円

平均手取り収入
300,000円

判定結果
達成可能
```

として保存された後に、

```text
2026-09

利用可能資産額
2,300,000円

平均手取り収入
320,000円
```

となっても、2026-08の履歴は保存時点の値を表示する。

---

### 12.4 現在の資産情報を再取得して計算しない

ASM-004では、判定結果を生成する目的で以下を再取得しない。

```text
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
net_incomes
```

これらはASM-001・ASM-002における判定計算の入力情報であり、ASM-004では保存済みの計算根拠を参照する。

---

### 12.5 判定結果を再計算しない

例えば、

```text
利用可能資産額
-
必要支出額
-
生活防衛資金
```

などの計算をASM-004で再実行して、保存済み判定結果と照合しない。

ASM-004は、

```text
計算するAPI
```

ではなく、

```text
保存された計算結果を取得するAPI
```

とする。

---

### 12.6 判定履歴は上書きしない

ASM-004では、判定履歴を更新しない。

また、現在値と保存済み値に差異があっても、

```text
assessment_histories
```

を補正しない。

再判定する場合は、ASM-002によって新しい判定履歴を作成する。

---

### 12.7 再判定は別履歴として扱う

同じ目的について再判定した場合は、過去履歴を更新せず、新しい判定履歴として保存する。

概念的には、

```text
Objective A

2026-08
Assessment History 100

2026-09
Assessment History 120
```

のように、それぞれ独立した履歴として扱う。

ASM-004では、指定された1件だけを取得する。

---

### 12.8 目的の現在状態に依存しない

判定履歴に紐づく目的が現在無効化されていても、過去履歴は取得可能とする。

```text
判定実行時
objective = 有効
    ↓
判定履歴保存
    ↓
後日
objective = 無効
    ↓
ASM-004
参照可能
```

現在の目的状態を理由に過去履歴を非表示にしない。

---

### 12.9 目的の現在値を判定根拠として使用しない

目的に設定された金額などが判定後に変更可能な仕様である場合、現在の`objectives`の値を過去判定の計算根拠として使用しない。

判定時点の値が`assessment_histories`へ保存されている場合は、その保存値を使用する。

---

### 12.10 目的名の扱い

目的名について、ASM-002実行時点の名称を`assessment_histories`へスナップショットとして保存している場合は、その保存値を優先する。

一方、目的名を履歴へ保存しないテーブル設計の場合は、`objectives`から現在の目的名を取得する。

どちらを採用するかは、`assessment_histories`のテーブル定義を正とする。

---

### 12.11 判定根拠は履歴として保持する

判定履歴には、単なる

```text
達成可能
達成不可
判定不可
```

などの判定結果だけでなく、

```text
なぜその結果になったか
```

を確認できる情報を保持する。

これにより、ASM-004で過去判定の説明可能性を確保する。

---

### 12.12 判定不可も履歴として扱う

ASM-002によって「判定不可」が保存可能な仕様である場合は、ASM-004でも通常の判定履歴として取得する。

例えば、

```text
assessmentResult
    = NOT_ASSESSABLE
```

であっても、404やエラーとはしない。

判定不可となった理由を保存している場合は、その理由も返却する。

---

### 12.13 月末資産状況の現在の確定状態に依存しない

判定実行後に関連する月末資産状況が確定解除された場合でも、保存済み判定履歴を参照可能とする。

現在の

```text
month_end_asset_snapshots.confirmed
```

をASM-004の取得可否条件にはしない。

---

### 12.14 月末資産状況から判定結果を再構築しない

`month_end_asset_snapshots`は、判定対象となった年月などを特定するために参照してよい。

ただし、

```text
month_end_asset_snapshots
    ↓
month_end_asset_balances
    ↓
month_end_holding_values
    ↓
現在の設定を適用
    ↓
判定結果を再構築
```

という処理は行わない。

---

### 12.15 他利用者の履歴を取得しない

指定された`assessmentHistoryId`が存在しても、操作対象利用者に属さない場合は取得しない。

以下を同一に扱う。

```text
判定履歴不存在
        ↓
ASSESSMENT_HISTORY_NOT_FOUND

他利用者所属
        ↓
ASSESSMENT_HISTORY_NOT_FOUND
```

---

### 12.16 他利用者所属を公開しない

ASM-004では、

```text
FORBIDDEN
OTHER_USER_ASSESSMENT_HISTORY
```

などによって、他利用者の履歴が存在することを公開しない。

利用者境界を越えた情報漏えいを防止する。

---

### 12.17 判定履歴0件という概念は持たない

ASM-004は単一リソース取得APIである。

指定された`assessmentHistoryId`に対応する取得可能な判定履歴が存在しなければ、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

とする。

ASM-003のように、

```json
{
  "data": []
}
```

を返すAPIではない。

---

### 12.18 関連データの参照整合性

正常な状態では、

```text
assessment_histories
    ↓
objectives

assessment_histories
    ↓
month_end_asset_snapshots
```

の参照整合性がDB制約によって保証されていることを前提とする。

---

### 12.19 関連データ不整合を自動修復しない

例えば、

```text
assessment_histories.objective_id
    ↓
対応するobjectivesが存在しない
```

などの想定外データ不整合が発生しても、ASM-004で以下を行わない。

- 履歴削除
- 関連ID変更
- 目的自動作成
- 判定結果再計算

内部データ不整合として扱う。

---

### 12.20 参照専用とする

ASM-004では、以下の業務データを変更しない。

```text
assessment_histories
objectives
month_end_asset_snapshots
net_incomes
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
```

GETによる副作用を持たせない。

---

## 13. 処理フロー

ASM-004の基本的な処理フローは、以下とする。

```text
Request
    ↓
X-User-Id取得
    ↓
利用者コンテキスト確認
    ↓
assessmentHistoryId形式確認
    ↓
assessmentHistoryId
+
userId
    ↓
判定履歴取得
    ↓
存在する？
    ├─ No
    │    ↓
    │ ASSESSMENT_HISTORY_NOT_FOUND
    │
    └─ Yes
         ↓
      関連目的情報取得
         ↓
      関連月末資産状況取得
         ↓
      保存済み判定結果取得
         ↓
      保存済み計算根拠取得
         ↓
      Response DTO生成
         ↓
      API Resource
         ↓
      200 OK
```

---

### 13.1 利用者コンテキスト確認

最初に、

```text
X-User-Id
```

から操作対象利用者を確定する。

利用者コンテキストが確定できない場合は、判定履歴検索へ進まない。

---

### 13.2 assessmentHistoryId形式確認

パスパラメータの

```text
assessmentHistoryId
```

が正の整数形式であることを確認する。

形式不正の場合は、

```text
INVALID_ASSESSMENT_HISTORY_ID
```

として処理を終了する。

---

### 13.3 判定履歴取得

形式確認後、操作対象利用者の境界を含めて判定履歴を取得する。

概念的には、

```text
assessmentHistoryId
+
UserContext.userId
```

を条件とする。

---

### 13.4 判定履歴不存在

対象判定履歴を取得できない場合は、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

として処理を終了する。

この時点で、

```text
本当に存在しない

または

他利用者に属する
```

をクライアントへ区別して通知しない。

---

### 13.5 目的情報取得

判定履歴に紐づく目的情報を取得する。

概念的には、

```text
assessment_histories.objective_id
    ↓
objectives.id
```

とする。

ただし、判定時点の目的情報を`assessment_histories`へスナップショット保存している項目は、保存値を優先する。

---

### 13.6 月末資産状況取得

判定対象となった月末資産状況を取得する。

概念的には、

```text
assessment_histories
    ↓
month_end_asset_snapshot_id
    ↓
month_end_asset_snapshots
```

とする。

主に、判定対象年月など履歴詳細表示に必要な情報を取得する。

---

### 13.7 計算根拠取得

判定結果の計算根拠は、ASM-002実行時に`assessment_histories`へ保存された値を取得する。

現在の各種テーブルから再計算しない。

---

### 13.8 Response DTO生成

取得した情報を、ASM-004専用のResponse DTOへ変換する。

概念的には、

```text
assessment_histories
+
objectives
+
month_end_asset_snapshots
    ↓
AssessmentHistoryDetail DTO
```

とする。

---

### 13.9 API Resourceによる変換

Response DTOをAPI Resourceへ渡し、

```text
snake_case
    ↓
camelCase
```

など、APIレスポンス形式へ変換する。

DBカラムをそのままJSONへ返却しない。

---

## 14. トランザクション境界

ASM-004は、参照専用GET APIであるため、明示的なトランザクションは原則として使用しない。

```php
DB::transaction(...)
```

で処理全体を囲まない。

---

### 14.1 更新処理を行わない

ASM-004では、

```text
INSERT
UPDATE
DELETE
```

を行わない。

そのため、更新処理の原子性を保証するためのトランザクションは不要とする。

---

### 14.2 複数SELECTを許容する

判定履歴、目的、月末資産状況などを複数SELECTで取得する場合でも、Phase1では厳密な同一時点読み取りを業務要件としない。

そのため、参照トランザクションを必須としない。

---

### 14.3 保存済み判定値を正とするため影響を受けにくい

ASM-004の主要な判定情報は、

```text
assessment_histories
```

に保存済みである。

現在の資産状況を複数テーブルから集めて再計算するAPIではないため、読み取り途中のデータ変更による判定結果の不整合を考慮する必要性は低い。

---

### 14.4 将来的な変更

将来的に、ASM-004で複数テーブルの現在状態を厳密な同一時点で取得する必要が生じた場合は、読み取りトランザクションの導入を再検討する。

Phase1では導入しない。

---

## 15. 排他制御

ASM-004は、参照専用GET APIであるため、排他制御を行わない。

以下を使用しない。

```php
lockForUpdate()
```

また、

```text
悲観ロック
楽観ロック
```

もASM-004では不要とする。

---

### 15.1 lockForUpdateを使用しない

ASM-004は判定履歴を更新しないため、

```php
->lockForUpdate()
```

によって対象履歴をロックしない。

参照処理が他の処理を不要に待機させないようにする。

---

### 15.2 判定履歴は履歴データとして扱う

`assessment_histories`は、判定実行時点の結果を保持する履歴データである。

ASM-004による参照時に履歴内容を変更しないため、参照のための排他制御は不要とする。

---

### 15.3 関連目的をロックしない

ASM-004で`objectives`を参照しても、対象目的をロックしない。

目的の現在状態が変更されたとしても、保存済み判定結果そのものを再計算しないためである。

---

### 15.4 月末資産状況をロックしない

`month_end_asset_snapshots`についても、参照のためのロックを行わない。

確定・確定解除などの更新処理をASM-004によって不要にブロックしない。

---

## 16. 成功レスポンス

ASM-004で判定履歴詳細の取得に成功した場合は、

```http
200 OK
```

を返却する。

レスポンスには、

```text
判定履歴
目的
判定対象月
判定結果
判定時点の計算根拠
```

を含める。

概念例：

```json
{
  "data": {
    "id": "100",
    "objectiveId": "10",
    "objectiveName": "一人暮らし開始資金",
    "targetYearMonth": "2026-08",
    "assessmentResult": "ACHIEVABLE",
    "availableAssetAmount": 2500000,
    "requiredAmount": 1000000,
    "averageNetIncome": 300000,
    "requiredEmergencyFund": 900000,
    "remainingAmount": 600000,
    "assessedAt": "2026-08-31T12:00:00+09:00"
  }
}
```

具体的な項目名および計算根拠項目は、`assessment_histories`のテーブル定義およびASM-002の保存仕様を正とする。

---

### 16.1 200 OK

正常に判定履歴詳細を取得できた場合は、

```http
200 OK
```

を返却する。

ASM-004は単一リソース取得APIであるため、対象が存在しない場合に

```json
{
  "data": null
}
```

を`200 OK`で返却しない。

---

### 16.2 Envelope形式

API共通方針に従い、正常レスポンスは

```text
data
```

でラップする。

概念例：

```json
{
  "data": {
    "id": "100"
  }
}
```

ASM-004専用のEnvelope形式は定義しない。

---

### 16.3 IDはstringとして返却する

以下のIDは、APIではstringとして返却する。

```text
id
objectiveId
```

月末資産状況IDをレスポンスへ含める設計の場合は、

```text
monthEndAssetSnapshotId
```

もstringとして返却する。

DB上の主キーが`bigint`であっても、APIレスポンスではstringとして扱う。

---

### 16.4 金額はintegerとして返却する

判定根拠となる金額は、日本円整数として返却する。

例えば、

```json
{
  "availableAssetAmount": 2500000,
  "requiredAmount": 1000000
}
```

のように、数値として返却する。

表示用の

```text
"2,500,000円"
```

のような文字列にはしない。

金額フォーマットはReact側で行う。

---

### 16.5 判定結果はコード値で返却する

判定結果は、表示文言ではなくAPI共通のコード値として返却する。

概念例：

```json
{
  "assessmentResult": "ACHIEVABLE"
}
```

例えば、

```text
ACHIEVABLE
NOT_ACHIEVABLE
NOT_ASSESSABLE
```

などとする。

実際の値は、目的達成判定の共通定義を正とする。

---

### 16.6 判定不可も正常レスポンスとする

保存された判定結果が

```text
NOT_ASSESSABLE
```

であっても、ASM-004自体のエラーではない。

正常に履歴を取得できているため、

```http
200 OK
```

を返却する。

判定不可理由を保存している場合は、その理由をレスポンスへ含める。

---

### 16.7 判定実行日時

判定履歴を保存した日時または判定を実行した日時を、

```text
assessedAt
```

として返却する。

概念例：

```json
{
  "assessedAt": "2026-08-31T12:00:00+09:00"
}
```

日時形式は、API共通方針に従う。

---

### 16.8 現在値を混在させない

レスポンスへ判定時点の計算根拠と現在の計算値を同じ意味の項目として混在させない。

例えば、

```text
availableAssetAmount
```

が判定時点の値であるなら、現在の利用可能資産額で上書きしない。

---

### 16.9 現在値が必要な場合

現在の資産状況と過去の判定結果を比較したい場合は、

```text
ASM-004
+
現在状態を取得する別API
```

を画面側で組み合わせる。

ASM-004のレスポンスへ現在値を無制限に追加しない。

---

### 16.10 DBカラムを直接返却しない

以下のようなDBカラム名をそのままレスポンスへ公開しない。

```text
objective_id
month_end_asset_snapshot_id
assessment_result
available_asset_amount
assessed_at
```

APIでは、

```text
objectiveId
monthEndAssetSnapshotId
assessmentResult
availableAssetAmount
assessedAt
```

のようにcamelCaseへ変換する。

---

### 16.11 内部管理情報を返却しない

ASM-004では、画面表示やAPI利用に不要な内部情報を返却しない。

例えば、

```text
user_id
deleted_at
DB内部制約情報
内部計算用フラグ
```

などは、必要性がない限りレスポンスへ含めない。

---

### 16.12 レスポンス項目の正は保存仕様とする

ASM-004で返却できる判定時点の計算根拠は、ASM-002で`assessment_histories`へ保存されている情報に依存する。

そのため、

```text
ASM-002で保存していない情報を
ASM-004で過去値として再現する
```

ことはしない。

過去判定の詳細表示に必要な項目がある場合は、ASM-004だけではなくASM-002の保存項目および`assessment_histories`のテーブル設計も合わせて変更する。

---

## 17. レスポンス項目

`data`配下には、指定された判定履歴の詳細情報を返却する。

主なレスポンス項目は、以下とする。

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `id` | string | × | 判定履歴ID |
| `objectiveId` | string | × | 判定対象となった目的ID |
| `objectiveName` | string | × | 判定対象となった目的名 |
| `targetYearMonth` | string | × | 判定対象となった年月。`YYYY-MM`形式 |
| `assessmentResult` | string | × | 判定結果 |
| `availableAssetAmount` | integer | ○ | 判定時点の利用可能資産額 |
| `requiredAmount` | integer | ○ | 判定時点の必要額 |
| `averageNetIncome` | integer | ○ | 判定時点で使用した平均手取り収入 |
| `requiredEmergencyFund` | integer | ○ | 判定時点で必要とした生活防衛資金 |
| `remainingAmount` | integer | ○ | 必要額等を控除した後の残額 |
| `notAssessableReason` | string | ○ | 判定不可の場合の理由コード |
| `assessedAt` | string | × | 判定実行日時 |

具体的な項目は、ASM-002で保存する`assessment_histories`の項目を正とする。

---

### 17.1 id

判定履歴IDをstringとして返却する。

例：

```json
{
  "id": "100"
}
```

DB上の主キーが`bigint`であっても、APIではstringとして返却する。

---

### 17.2 objectiveId

判定対象となった目的IDを返却する。

例：

```json
{
  "objectiveId": "10"
}
```

`objectives.id`に対応する値をstringとして返却する。

---

### 17.3 objectiveName

判定対象となった目的名を返却する。

例：

```json
{
  "objectiveName": "一人暮らし開始資金"
}
```

判定時点の目的名を`assessment_histories`へ保存している場合は、保存済み値を使用する。

保存していない場合は、現在の`objectives.name`を参照して返却する。

どちらを正とするかは、テーブル定義およびASM-002の保存仕様に従う。

---

### 17.4 targetYearMonth

判定対象となった月末資産状況の対象年月を返却する。

形式は、

```text
YYYY-MM
```

とする。

例：

```json
{
  "targetYearMonth": "2026-08"
}
```

---

### 17.5 assessmentResult

保存された目的達成判定結果を返却する。

概念的な値は、以下とする。

```text
ACHIEVABLE
NOT_ACHIEVABLE
NOT_ASSESSABLE
```

例：

```json
{
  "assessmentResult": "ACHIEVABLE"
}
```

実際のコード値は、目的達成判定API全体の共通定義を正とする。

---

### 17.6 availableAssetAmount

判定時点で使用した利用可能資産額を返却する。

例：

```json
{
  "availableAssetAmount": 2500000
}
```

現在の利用可能資産額を再計算して返却しない。

---

### 17.7 requiredAmount

目的達成判定において必要とした金額を返却する。

例：

```json
{
  "requiredAmount": 1000000
}
```

具体的に何を含む金額かは、ASM-002で保存する判定計算ロジックを正とする。

---

### 17.8 averageNetIncome

判定時点で使用した平均手取り収入を返却する。

例：

```json
{
  "averageNetIncome": 300000
}
```

ASM-004実行時点の`net_incomes`から再計算しない。

---

### 17.9 requiredEmergencyFund

目的達成後に確保すべき生活防衛資金として判定時に使用した金額を返却する。

例：

```json
{
  "requiredEmergencyFund": 900000
}
```

判定時点で保存された値を返却する。

---

### 17.10 remainingAmount

判定計算後の残額を返却する。

概念的には、

```text
利用可能資産額
-
必要支出額
-
必要生活防衛資金
```

などによって算出された保存済み結果を返却する。

例：

```json
{
  "remainingAmount": 600000
}
```

ASM-004で再計算しない。

---

### 17.11 notAssessableReason

判定結果が

```text
NOT_ASSESSABLE
```

の場合に、判定不可理由を返却する。

例：

```json
{
  "notAssessableReason": "NET_INCOME_INSUFFICIENT"
}
```

判定可能な場合は、

```json
{
  "notAssessableReason": null
}
```

としてよい。

実際の理由コードは、ASM-001・ASM-002と共通定義を使用する。

---

### 17.12 assessedAt

判定を実行した日時を返却する。

例：

```json
{
  "assessedAt": "2026-08-31T12:00:00+09:00"
}
```

日時形式は、API共通方針に従う。

---

### 17.13 金額項目の型

以下の金額項目は、日本円整数として返却する。

```text
availableAssetAmount
requiredAmount
averageNetIncome
requiredEmergencyFund
remainingAmount
```

以下のような表示用文字列にはしない。

```json
{
  "availableAssetAmount": "2,500,000円"
}
```

表示フォーマットはReact側の責務とする。

---

### 17.14 NULLの扱い

判定結果によって一部の計算根拠を算出できない場合は、該当項目を`null`として返却してよい。

例えば、手取り収入不足により判定不可となり、残額まで算出しない場合は、

```json
{
  "remainingAmount": null
}
```

となり得る。

NULL可否は、ASM-002の保存仕様およびテーブル定義を正とする。

---

### 17.15 userIdを返却しない

操作対象利用者は、

```text
X-User-Id
```

によって確定している。

そのため、レスポンスへ

```text
userId
```

を含めない。

---

### 17.16 内部DBカラムを返却しない

以下のようなDB内部表現をそのまま返却しない。

```text
objective_id
month_end_asset_snapshot_id
assessment_result
available_asset_amount
created_at
updated_at
```

APIではcamelCaseへ変換する。

---

### 17.17 現在の目的状態を返却しない

ASM-004は判定履歴詳細を取得するAPIであるため、現在の

```text
objectives.enabled
```

を判定履歴の主要項目として返却しない。

現在の目的状態が必要な場合は、OBJ-003等を利用する。

---

### 17.18 現在の月末資産状況確定状態を返却しない

同様に、

```text
month_end_asset_snapshots.confirmed
```

の現在値も、過去判定履歴の主要な計算根拠ではない。

画面要件として必要になった場合は、別APIまたは追加仕様として検討する。

---

## 18. エラーレスポンス

エラー時は、API共通方針で定めたエラーレスポンス形式を使用する。

ASM-004独自のエラーEnvelopeは定義しない。

概念例：

```json
{
  "error": {
    "code": "ASSESSMENT_HISTORY_NOT_FOUND",
    "message": "指定された判定履歴が見つかりません。"
  },
  "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
}
```

---

### 18.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合は、

```text
USER_CONTEXT_REQUIRED
```

を返却する。

この場合、判定履歴検索へ進まない。

---

### 18.2 INVALID_USER_ID

`X-User-Id`がAPI共通方針で定めたID形式として不正な場合は、

```text
INVALID_USER_ID
```

として扱う。

例えば、

```text
0
-1
abc
1.5
```

などを不正とする。

---

### 18.3 USER_NOT_FOUND

指定された利用者が存在しない、または利用対象として扱えない場合は、

```text
USER_NOT_FOUND
```

を返却する。

利用者コンテキストを確立できないため、判定履歴取得へ進まない。

---

### 18.4 INVALID_ASSESSMENT_HISTORY_ID

`assessmentHistoryId`が不正な形式の場合は、

```text
INVALID_ASSESSMENT_HISTORY_ID
```

を返却する。

例えば、

```text
0
-1
abc
1.5
1e3
100abc
```

などを不正とする。

---

### 18.5 ASSESSMENT_HISTORY_NOT_FOUND

以下の場合は、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

を返却する。

- 指定された判定履歴が存在しない
- 指定された判定履歴が他利用者に属する
- SoftDeletesを採用している場合に判定履歴が論理削除済み

これらをクライアントへ個別には公開しない。

---

### 18.6 他利用者所属

例えば、

```text
X-User-Id = 2

assessmentHistoryId = 100

履歴所有者 = User 1
```

の場合でも、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

として扱う。

以下のような専用エラーは返却しない。

```text
ASSESSMENT_HISTORY_FORBIDDEN
OTHER_USER_ASSESSMENT_HISTORY
```

他利用者の判定履歴の存在を外部へ公開しないためである。

---

### 18.7 目的が無効化済みでもエラーにしない

判定履歴に紐づく目的が現在

```text
enabled = false
```

であっても、

```text
OBJECTIVE_DISABLED
```

などのエラーにはしない。

過去判定履歴は正常に参照可能とする。

---

### 18.8 現在の月末資産状況が未確定でもエラーにしない

判定保存後に月末資産状況が確定解除され、

```text
confirmed = false
```

となっていても、

```text
SNAPSHOT_NOT_CONFIRMED
```

などのエラーにはしない。

ASM-004では保存済み判定履歴を参照する。

---

### 18.9 判定不可はエラーにしない

保存されている判定結果が

```text
NOT_ASSESSABLE
```

であっても、ASM-004のAPIエラーではない。

正常な判定履歴として、

```http
200 OK
```

を返却する。

判定不可理由をレスポンス項目として返却する。

---

### 18.10 関連目的の想定外不存在

正常なDB制約下では発生しないが、

```text
assessment_histories.objective_id
    ↓
対応するobjectivesが存在しない
```

などの内部データ不整合が発生した場合は、利用者入力エラーとはしない。

```text
INTERNAL_SERVER_ERROR
```

として扱う。

---

### 18.11 関連月末資産状況の想定外不存在

同様に、

```text
assessment_histories.month_end_asset_snapshot_id
    ↓
対応するmonth_end_asset_snapshotsが存在しない
```

場合も、Request不正とはしない。

内部データ不整合として扱う。

---

### 18.12 保存済み判定根拠の不整合

判定履歴に必要な計算根拠が欠落しているなど、ASM-002の保存仕様上通常発生しない状態が存在する場合も、

```text
INTERNAL_SERVER_ERROR
```

として扱う。

ASM-004で現在値から不足項目を補完しない。

---

### 18.13 INTERNAL_SERVER_ERROR

想定外の例外、DBエラー、内部データ不整合などによって正常に判定履歴詳細を取得できない場合は、

```text
INTERNAL_SERVER_ERROR
```

を返却する。

レスポンスへ以下を含めない。

- SQL
- SQLSTATE
- PostgreSQL内部エラー
- 制約名
- テーブル名
- カラム名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

詳細はサーバーログへ記録する。

---

### 18.14 エラーメッセージへ内部状態を出さない

例えば、関連目的が存在しない場合でも、

```text
assessment_histories.objective_id=10 に対応する
objectivesが存在しません
```

のような内部構造を含むメッセージをクライアントへ返却しない。

利用者向けには、共通のサーバーエラーとして扱う。

---

### 18.15 error.codeをクライアント判定の基準とする

React側では、`message`文字列ではなく

```text
error.code
```

を基準としてエラー処理を分岐する。

概念例：

```ts
switch (error.code) {
  case 'INVALID_ASSESSMENT_HISTORY_ID':
    // ID形式不正
    break;

  case 'ASSESSMENT_HISTORY_NOT_FOUND':
    // 対象履歴不存在
    break;

  default:
    // 共通エラー
    break;
}
```

エラーメッセージの文言変更によってクライアント処理が壊れないようにする。

---

## 19. HTTPステータス

ASM-004では、処理結果に応じて以下のHTTPステータスを返却する。

| HTTPステータス | 用途 |
|---|---|
| `200 OK` | 判定履歴詳細取得成功 |
| `400 Bad Request` | リクエスト値の形式不正 |
| `404 Not Found` | 利用者または判定履歴が存在しない |
| `406 Not Acceptable` | `Accept`ヘッダーがAPI共通仕様を満たさない場合 |
| `500 Internal Server Error` | 想定外の内部エラー |

具体的なエラーコードとの対応は、API共通方針およびASM-004のエラーコード定義に従う。

---

### 19.1 200 OK

操作対象利用者に属する指定された判定履歴を正常に取得できた場合は、

```http
200 OK
```

を返却する。

判定結果が

```text
ACHIEVABLE
NOT_ACHIEVABLE
NOT_ASSESSABLE
```

のいずれであっても、判定履歴自体を正常に取得できた場合は`200 OK`とする。

---

### 19.2 400 Bad Request

リクエスト値の形式が不正な場合は、

```http
400 Bad Request
```

を返却する。

主な対象は、以下とする。

```text
INVALID_USER_ID
INVALID_ASSESSMENT_HISTORY_ID
```

例えば、

```http
GET /api/v1/assessment-histories/abc
```

のように、`assessmentHistoryId`が正の整数形式ではない場合が該当する。

---

### 19.3 X-User-Id未指定

`X-User-Id`が指定されていない場合は、API共通方針に従って

```text
USER_CONTEXT_REQUIRED
```

を返却する。

HTTPステータスについても、利用者コンテキストのAPI共通仕様を正とする。

ASM-004独自のHTTPステータス定義は設けない。

---

### 19.4 404 Not Found

以下の場合は、

```http
404 Not Found
```

を返却する。

```text
USER_NOT_FOUND
ASSESSMENT_HISTORY_NOT_FOUND
```

---

### 19.5 判定履歴不存在

形式上有効な`assessmentHistoryId`であっても、対象判定履歴が存在しない場合は、

```http
404 Not Found
```

とする。

例：

```text
assessmentHistoryId = 999

該当履歴なし
    ↓
404 Not Found
ASSESSMENT_HISTORY_NOT_FOUND
```

---

### 19.6 他利用者所属

指定された判定履歴が存在していても、他利用者に属する場合は、

```http
404 Not Found
```

とする。

```text
判定履歴不存在
        ↓
404

他利用者所属
        ↓
404
```

とし、両者を区別しない。

`403 Forbidden`は使用しない。

---

### 19.7 406 Not Acceptable

`Accept`ヘッダーについて、API共通方針で`application/json`以外を許可しない場合は、

```http
406 Not Acceptable
```

を使用する。

ASM-004固有のメディアタイプ仕様は定義しない。

---

### 19.8 500 Internal Server Error

想定外の例外、DBアクセスエラー、関連データの不整合などにより判定履歴詳細を正常に取得できない場合は、

```http
500 Internal Server Error
```

を返却する。

エラーコードは、

```text
INTERNAL_SERVER_ERROR
```

とする。

---

### 19.9 判定不可にエラーステータスを使用しない

保存された判定結果が

```text
NOT_ASSESSABLE
```

であっても、

```http
422 Unprocessable Entity
```

などのエラーステータスを返却しない。

判定不可は保存済み判定結果の一種であるため、

```http
200 OK
```

とする。

---

### 19.10 409 Conflictは使用しない

ASM-004は参照専用APIであるため、更新競合を表す

```http
409 Conflict
```

は使用しない。

---

### 19.11 HTTPステータスとエラーコード

主な対応関係は、以下とする。

| HTTPステータス | エラーコード |
|---|---|
| `200 OK` | なし |
| `400 Bad Request` | `INVALID_USER_ID` |
| `400 Bad Request` | `INVALID_ASSESSMENT_HISTORY_ID` |
| API共通仕様に従う | `USER_CONTEXT_REQUIRED` |
| `404 Not Found` | `USER_NOT_FOUND` |
| `404 Not Found` | `ASSESSMENT_HISTORY_NOT_FOUND` |
| `406 Not Acceptable` | API共通仕様に従う |
| `500 Internal Server Error` | `INTERNAL_SERVER_ERROR` |

---

## 20. エラーコード

ASM-004で使用する主なエラーコードは、以下とする。

| エラーコード | HTTPステータス | 説明 |
|---|---:|---|
| `USER_CONTEXT_REQUIRED` | API共通仕様に従う | `X-User-Id`が指定されていない |
| `INVALID_USER_ID` | `400` | `X-User-Id`の形式が不正 |
| `USER_NOT_FOUND` | `404` | 操作対象利用者が存在しない |
| `INVALID_ASSESSMENT_HISTORY_ID` | `400` | 判定履歴IDの形式が不正 |
| `ASSESSMENT_HISTORY_NOT_FOUND` | `404` | 判定履歴が存在しない、または操作対象利用者に属さない |
| `INTERNAL_SERVER_ERROR` | `500` | 想定外の内部エラー |

---

### 20.1 USER_CONTEXT_REQUIRED

`X-User-Id`が指定されていない場合に使用する。

```text
USER_CONTEXT_REQUIRED
```

ASM-004専用ではなく、API共通の利用者コンテキストエラーとする。

---

### 20.2 INVALID_USER_ID

`X-User-Id`が正の整数形式ではない場合に使用する。

```text
INVALID_USER_ID
```

例えば、

```text
0
-1
abc
1.5
```

などを対象とする。

---

### 20.3 USER_NOT_FOUND

指定された`X-User-Id`に対応する利用者が存在しない場合に使用する。

```text
USER_NOT_FOUND
```

論理削除済み利用者を利用対象外とする場合も、同じエラーコードを使用する。

---

### 20.4 INVALID_ASSESSMENT_HISTORY_ID

`assessmentHistoryId`が正の整数形式ではない場合に使用する。

```text
INVALID_ASSESSMENT_HISTORY_ID
```

例えば、

```text
0
-1
abc
1.5
1e3
100abc
```

などを対象とする。

---

### 20.5 ASSESSMENT_HISTORY_NOT_FOUND

以下の場合に使用する。

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

- 指定された判定履歴が存在しない
- 指定された判定履歴が他利用者に属する
- SoftDeletesを採用している場合に論理削除済み

他利用者所属専用のエラーコードは設けない。

---

### 20.6 INTERNAL_SERVER_ERROR

以下のような想定外の内部エラーに使用する。

```text
INTERNAL_SERVER_ERROR
```

例えば、

- DBアクセスエラー
- 想定外の例外
- 判定履歴と目的の参照不整合
- 判定履歴と月末資産状況の参照不整合
- 必須となる保存済み判定根拠の欠落

などが該当する。

---

### 20.7 判定結果をエラーコードにしない

以下はAPIエラーコードではなく、判定履歴の業務データとして扱う。

```text
ACHIEVABLE
NOT_ACHIEVABLE
NOT_ASSESSABLE
```

例えば、

```text
assessmentResult
    = NOT_ASSESSABLE
```

であっても、

```text
error.code
    = NOT_ASSESSABLE
```

とはしない。

---

### 20.8 判定不可理由もAPIエラーコードにしない

判定不可理由が、

```text
NET_INCOME_INSUFFICIENT
```

などで保存されている場合も、APIエラーコードとは分離する。

概念的には、

```json
{
  "data": {
    "assessmentResult": "NOT_ASSESSABLE",
    "notAssessableReason": "NET_INCOME_INSUFFICIENT"
  }
}
```

とする。

---

## 21. 冪等性

ASM-004は参照専用GET APIであるため、冪等である。

同一条件で複数回実行しても、ASM-004自身によってサーバー状態は変更されない。

```text
GET
/api/v1/assessment-histories/100

    ↓

何度実行しても
判定履歴を変更しない
```

---

### 21.1 副作用を持たない

ASM-004では、以下を行わない。

```text
INSERT
UPDATE
DELETE
```

そのため、同一リクエストを複数回送信しても判定履歴が増加したり、変更されたりしない。

---

### 21.2 Idempotency-Keyを使用しない

ASM-004では、

```text
Idempotency-Key
```

を使用しない。

GET自体が冪等な参照操作であるため、重複実行防止用のキー管理は不要とする。

---

### 21.3 再試行可能

一時的な通信障害などの場合、クライアントはASM-004を再試行してよい。

再試行によって判定履歴が重複作成されることはない。

---

### 21.4 冪等性とレスポンス同一性は別とする

ASM-004が冪等であることは、必ずしも

```text
毎回完全に同一のJSONが返る
```

ことを意味しない。

例えば、目的名を現在の`objectives.name`から取得する設計の場合、目的名変更後には表示値が変化する可能性がある。

判定時点の表示内容まで完全に固定する必要がある項目は、ASM-002実行時に`assessment_histories`へ保存する。

---

## 22. キャッシュ

ASM-004はGET APIであるため、技術的にはHTTPキャッシュの対象にできる。

ただし、Phase1ではサーバー側の明示的なHTTPキャッシュは原則として導入しない。

---

### 22.1 Phase1ではHTTPキャッシュを必須としない

判定履歴詳細は、通常の画面操作において大量アクセスされることを想定していない。

そのため、Phase1では、

```text
ETag
Last-Modified
Cache-Controlによる長期キャッシュ
```

などを必須要件としない。

---

### 22.2 判定履歴はキャッシュと相性がよい

`assessment_histories`を作成後に変更しない履歴データとして扱う場合、ASM-004のレスポンスは比較的キャッシュしやすい。

将来的にアクセス量や性能要件が明確になった場合は、

```text
ETag
If-None-Match
```

などの導入を検討できる。

---

### 22.3 利用者境界を越えてキャッシュしない

キャッシュを導入する場合は、

```text
assessmentHistoryId
```

だけをキャッシュキーとして扱い、他利用者へレスポンスが共有されないよう注意する。

少なくとも、

```text
userId
+
assessmentHistoryId
```

によって利用者境界を分離する必要がある。

---

### 22.4 React側のキャッシュ

React側でTanStack Queryを使用する場合は、クライアントキャッシュを利用してよい。

概念的には、

```text
userId
+
assessmentHistoryId
```

をQuery Keyへ含める。

具体的な実装は、React・TypeScriptでの利用で定義する。

---

### 22.5 キャッシュによって認可を省略しない

キャッシュ済みレスポンスが存在する場合でも、バックエンドで利用者境界確認を省略する設計にはしない。

キャッシュは利用者境界を保証する仕組みの代替ではない。

---

## 23. 関連テーブル

ASM-004では、主に以下のテーブルを参照する。

| テーブル | 用途 |
|---|---|
| `assessment_histories` | 保存済み判定結果および計算根拠の取得 |
| `objectives` | 判定対象となった目的情報の取得 |
| `month_end_asset_snapshots` | 判定対象となった月末資産状況情報の取得 |

---

### 23.1 assessment_histories

ASM-004の中心となるテーブルである。

主に、

- 判定履歴ID
- 目的ID
- 月末資産状況ID
- 判定結果
- 判定時点の計算根拠
- 判定実行日時

などを取得する。

具体的なカラムは、テーブル定義書を正とする。

---

### 23.2 objectives

判定履歴に紐づく目的情報を取得するために参照する。

概念的には、

```text
assessment_histories.objective_id
    ↓
objectives.id
```

で関連付ける。

---

### 23.3 month_end_asset_snapshots

判定対象となった月末資産状況を特定するために参照する。

概念的には、

```text
assessment_histories
    ↓
month_end_asset_snapshot_id
    ↓
month_end_asset_snapshots.id
```

とする。

主に、

```text
target_year_month
```

などを取得する。

---

### 23.4 判定計算用テーブルを直接参照しない

ASM-004では、判定結果を再計算しないため、原則として以下を判定計算目的で直接参照しない。

```text
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
net_incomes
```

これらは主にASM-001・ASM-002で使用する。

---

### 23.5 users

`X-User-Id`から利用者コンテキストを確立する共通処理では、

```text
users
```

を参照する。

ただし、ASM-004固有の主要関連テーブルには含めず、API共通の利用者コンテキスト処理として扱う。

---

## 24. トランザクション

ASM-004は参照専用GET APIであるため、明示的なDBトランザクションは原則として使用しない。

```php
DB::transaction(function () {
    // ...
});
```

で処理を囲まない。

---

### 24.1 トランザクションを使用しない理由

ASM-004では、

```text
INSERT
UPDATE
DELETE
```

を行わない。

そのため、複数の更新処理を原子的に成功または失敗させる必要がない。

---

### 24.2 参照処理のみとする

ASM-004で行うDB操作は、基本的に

```text
SELECT
```

のみとする。

概念的には、

```text
assessment_histories
+
objectives
+
month_end_asset_snapshots
    ↓
SELECT
    ↓
Response
```

となる。

---

### 24.3 複数テーブル参照でもトランザクションを必須としない

`assessment_histories`、`objectives`、`month_end_asset_snapshots`を複数回のSELECTで取得する場合でも、Phase1では読み取りトランザクションを必須としない。

ASM-004の中心となる判定結果および計算根拠は、保存済みの履歴情報を正とするためである。

---

### 24.4 Queryでまとめて取得してよい

実装上は、必要に応じてJOINやEager Loadingを使用し、

```text
assessment_histories
    ├─ objectives
    └─ month_end_asset_snapshots
```

を効率的に取得してよい。

トランザクションを使わずとも、1つのQueryで必要情報を取得することは可能である。

---

### 24.5 読み取り一貫性が必要になった場合

将来的に、ASM-004で

```text
複数テーブルの現在状態を
厳密な同一時点として取得する
```

という要件が追加された場合は、読み取りトランザクションを検討する。

Phase1では、その要件を持たない。

---

### 24.6 ロックを使用しない

トランザクションを使用しないことに加えて、

```php
lockForUpdate()
```

などの悲観ロックも使用しない。

ASM-004による参照が、ASM-002や他の更新APIを不要にブロックしないようにする。

---

### 24.7 GETによるデータ変更を行わない

ASM-004の処理途中で、

```text
不足している判定根拠を補完する
関連データを更新する
判定履歴を修正する
参照日時を業務テーブルへ保存する
```

などの処理を追加しない。

そのような更新が必要になった場合は、GETとは別の責務としてAPI設計を見直す。

ASM-004では、

```text
保存済み判定履歴を
副作用なく取得する
```

ことをトランザクション設計上の基本方針とする。

---

## 25. ロック

ASM-004は、保存済み判定履歴を取得する参照専用GET APIであるため、DBレコードに対する明示的なロックは使用しない。

以下のような悲観ロックは行わない。

```php
->lockForUpdate()
```

また、楽観ロックによるバージョンチェックも行わない。

---

### 25.1 判定履歴をロックしない

`assessment_histories`は、ASM-004では参照のみ行う。

そのため、

```text
assessment_histories
    ↓
SELECT
    ↓
Response
```

とし、取得対象レコードをロックしない。

ASM-004による参照が、他の処理を不要に待機させないようにする。

---

### 25.2 lockForUpdateを使用しない

以下のような実装にはしない。

```php
$assessmentHistory = AssessmentHistory::query()
    ->whereKey($assessmentHistoryId)
    ->lockForUpdate()
    ->first();
```

ASM-004では取得した判定履歴をその後更新しないため、`lockForUpdate()`を使用する必要がない。

---

### 25.3 objectivesをロックしない

判定履歴詳細表示のために`objectives`を参照する場合でも、対象目的をロックしない。

例えば、ASM-004の処理中に目的の現在状態が変更されたとしても、保存済み判定結果を再計算することはない。

そのため、

```text
objectives
    ↓
SELECTのみ
```

とする。

---

### 25.4 month_end_asset_snapshotsをロックしない

判定対象となった月末資産状況を参照する場合も、

```php
->lockForUpdate()
```

を使用しない。

ASM-004による参照によって、以下をブロックしない。

- 月末資産状況確定
- 月末資産状況確定解除
- その他の更新処理

---

### 25.5 assessment_historiesを履歴データとして扱う

`assessment_histories`は、

```text
判定実行時点の結果と
計算根拠を保存する履歴
```

として扱う。

ASM-004では履歴内容を変更しないため、参照時の排他制御を必要としない。

---

### 25.6 楽観ロックを使用しない

ASM-004では、

```text
version
lock_version
updatedAt
```

などを利用した楽観ロックも行わない。

更新競合を検出する必要がないためである。

---

### 25.7 将来的な変更

将来的に判定履歴そのものを編集可能にする仕様が追加された場合は、更新API側で排他制御の必要性を検討する。

ASM-004については、参照専用である限りロックを使用しない。

---

## 26. キャッシュ

ASM-004はGET APIであるため、技術的にはキャッシュ可能である。

ただし、Phase1ではバックエンド側の明示的なレスポンスキャッシュを必須としない。

---

### 26.1 Phase1ではサーバーキャッシュを導入しない

Phase1では、

```text
Redis
Application Cache
Response Cache
```

などを利用したASM-004専用のサーバーキャッシュは原則として導入しない。

まずは通常のDB参照による単純な実装を優先する。

---

### 26.2 HTTPキャッシュを必須としない

Phase1では、

```text
ETag
Last-Modified
If-None-Match
If-Modified-Since
```

などを利用した条件付きGETも必須要件としない。

性能上の必要性が明確になった段階で導入を検討する。

---

### 26.3 判定履歴はキャッシュと相性がよい

判定履歴を作成後に変更しない履歴データとして扱う場合、ASM-004のレスポンスは比較的キャッシュしやすい。

概念的には、

```text
ASM-002
    ↓
判定履歴作成
    ↓
以降は変更しない
    ↓
ASM-004
参照のみ
```

となる。

そのため、将来的にアクセス量が増えた場合はキャッシュ導入を検討できる。

---

### 26.4 利用者境界をキャッシュキーへ反映する

サーバーキャッシュを将来的に導入する場合、

```text
assessmentHistoryId
```

だけをキャッシュキーとして使用しない。

少なくとも、

```text
userId
+
assessmentHistoryId
```

によって利用者境界を分離する。

概念例：

```text
assessment-history-detail:{userId}:{assessmentHistoryId}
```

とする。

---

### 26.5 他利用者へキャッシュを共有しない

例えば、

```text
User 1
assessmentHistoryId = 100
```

のレスポンスを、

```text
User 2
assessmentHistoryId = 100
```

のRequestへ返却してはならない。

キャッシュを導入しても、利用者境界を維持する。

---

### 26.6 キャッシュによって利用者境界確認を省略しない

キャッシュが存在することを理由に、利用者コンテキストの確認を省略しない。

概念的には、

```text
X-User-Id
    ↓
UserContext確立
    ↓
利用者境界を考慮したCache
    ↓
Response
```

とする。

キャッシュは認可・利用者境界保証の代替にはしない。

---

### 26.7 React側のキャッシュ

React側でTanStack Queryを使用する場合は、ASM-004の取得結果をQuery Cacheへ保持してよい。

概念的なQuery Keyは、

```text
assessmentHistories
detail
userId
assessmentHistoryId
```

などとする。

具体的な実装は、「React・TypeScriptでの利用」で定義する。

---

### 26.8 キャッシュの無効化

判定履歴を作成後に変更しない設計であれば、ASM-004のCacheを頻繁に無効化する必要はない。

ただし、

```text
利用者切替
```

が行われた場合は、前利用者の判定履歴を誤表示しないようにする。

利用者切替時に関連Query Cacheを破棄する、またはQuery Keyへ`userId`を含める。

---

### 26.9 現在情報を含める場合の注意

`objectiveName`などを現在の`objectives`から取得する設計の場合、目的情報の変更によってASM-004のレスポンスが変化する可能性がある。

その場合は、

```text
判定履歴は不変
```

であっても、レスポンス全体が完全な不変データとは限らない。

長期キャッシュを導入する場合は、この点を考慮する。

---

### 26.10 完全な履歴スナップショットとする場合

将来的に、

```text
objectiveName
targetYearMonth
判定条件
判定結果
計算根拠
```

など、ASM-004で必要な情報をすべて判定時点のスナップショットとして保存する場合は、レスポンスの不変性が高くなる。

その場合は、より長期間のキャッシュを検討できる。

---

## 27. 冪等性

ASM-004は、参照専用GET APIであるため冪等である。

同一の

```text
X-User-Id
assessmentHistoryId
```

を指定して複数回実行しても、ASM-004自身によってサーバー上の業務データは変更されない。

---

### 27.1 同一Requestを複数回実行できる

例えば、

```http
GET /api/v1/assessment-histories/100
X-User-Id: 1
```

を複数回実行しても、

```text
assessment_histories
objectives
month_end_asset_snapshots
```

などの業務データは変更されない。

---

### 27.2 副作用を持たない

ASM-004では、

```text
INSERT
UPDATE
DELETE
```

を行わない。

そのため、同じRequestを再送しても、以下のような副作用は発生しない。

- 判定履歴が増える
- 判定履歴が更新される
- 判定結果が再計算される
- 月末資産状況が変更される
- 目的が変更される

---

### 27.3 Idempotency-Keyを使用しない

ASM-004では、

```text
Idempotency-Key
```

を使用しない。

GET自体が冪等な参照操作であり、重複実行による業務データ変更が発生しないためである。

---

### 27.4 通信エラー時に再試行できる

ASM-004は冪等であるため、一時的な通信エラーが発生した場合にクライアントから再試行してよい。

概念的には、

```text
ASM-004
    ↓
通信エラー
    ↓
Retry
    ↓
ASM-004
```

としても、判定履歴が重複作成されることはない。

---

### 27.5 冪等性とレスポンスの完全一致は別である

冪等性は、

```text
同じRequestを繰り返しても
サーバー状態に追加の変化を与えない
```

ことを意味する。

必ずしも、

```text
毎回完全に同じJSONを返す
```

ことを意味しない。

---

### 27.6 現在の目的情報を参照する場合

例えば、`objectiveName`を現在の`objectives.name`から取得する設計の場合、

```text
1回目のASM-004
    ↓
目的名 = 引っ越し資金

目的名変更
    ↓

2回目のASM-004
    ↓
目的名 = 一人暮らし開始資金
```

となる可能性がある。

この場合でも、ASM-004自身はサーバー状態を変更していないため、冪等性は維持される。

---

### 27.7 判定結果・計算根拠は再計算しない

ASM-004を複数回実行しても、

```text
現在の資産状況
現在の手取り収入
現在の利用可能資産設定
```

などから判定結果を再計算しない。

保存済みの

```text
assessment_histories
```

を正として返却する。

これにより、過去判定の結果および計算根拠を安定して参照できる。

---

### 27.8 冪等性の対象

ASM-004における冪等性は、

```text
GETによる判定履歴詳細取得が
業務データへ副作用を与えない
```

ことを保証する。

アクセスログ、APM、監査用の技術ログなどがRequestごとに記録されることは、ASM-004の業務上の冪等性を損なうものとはしない。

---

## 28. 関連テーブル

ASM-004では、主に以下のテーブルを参照する。

| テーブル | 用途 | 更新 |
|---|---|:---:|
| `assessment_histories` | 保存済みの判定結果および計算根拠を取得する | × |
| `objectives` | 判定対象となった目的情報を取得する | × |
| `month_end_asset_snapshots` | 判定対象となった月末資産状況および対象年月を取得する | × |

ASM-004は参照専用GET APIであるため、いずれのテーブルも更新しない。

---

### 28.1 assessment_histories

ASM-004の中心となるテーブルである。

指定された

```text
assessmentHistoryId
```

に対応する判定履歴を取得する。

主に以下の情報を参照する。

- 判定履歴ID
- 目的ID
- 月末資産状況ID
- 判定結果
- 判定時点の利用可能資産額
- 判定時点の必要額
- 判定時点の平均手取り収入
- 判定時点の必要生活防衛資金
- 判定後の残額
- 判定不可理由
- 判定実行日時

具体的なカラム名およびNULL可否は、`assessment_histories`のテーブル定義書を正とする。

---

### 28.2 assessment_historiesを判定結果の正とする

ASM-004では、判定結果および判定時点の計算根拠について、

```text
assessment_histories
```

に保存された値を正とする。

概念的には、

```text
assessment_histories
    ↓
assessment_result
available_asset_amount
required_amount
average_net_income
required_emergency_fund
remaining_amount
not_assessable_reason
assessed_at
```

などを取得する。

現在の資産状況から再計算して置き換えない。

---

### 28.3 objectives

判定履歴に紐づく目的情報を取得するために参照する。

概念的な関連は、

```text
assessment_histories.objective_id
    ↓
objectives.id
```

とする。

主に、

```text
目的ID
目的名
```

など、判定履歴詳細画面に必要な情報を取得する。

---

### 28.4 目的の現在状態を取得条件にしない

関連する目的が現在、

```text
enabled = false
```

であっても、過去の判定履歴は参照可能とする。

そのため、

```text
objectives.enabled = true
```

をASM-004のJOIN条件または取得条件へ含めない。

---

### 28.5 目的の論理削除との扱い

目的にSoftDeletesを採用している場合でも、過去の判定履歴を継続して参照する必要がある。

そのため、判定履歴詳細で現在の目的情報を参照する設計では、

```text
objectives.deleted_at IS NULL
```

だけを理由として保存済み判定履歴自体を取得不可にしないよう注意する。

判定時点の目的情報を`assessment_histories`へスナップショット保存している場合は、その保存値を優先する。

---

### 28.6 objectiveNameの取得元

`objectiveName`の取得元は、ASM-002の保存仕様に従う。

判定時点の目的名を

```text
assessment_histories
```

へ保存している場合は、保存済み目的名を返却する。

保存していない場合は、

```text
objectives.name
```

を参照する。

過去時点の名称を厳密に再現する必要がある場合は、ASM-002側で目的名を履歴へ保存する設計を優先する。

---

### 28.7 month_end_asset_snapshots

判定対象となった月末資産状況を取得するために参照する。

概念的な関連は、

```text
assessment_histories.month_end_asset_snapshot_id
    ↓
month_end_asset_snapshots.id
```

とする。

主に、

```text
target_year_month
```

など、判定対象となった年月を取得するために使用する。

---

### 28.8 月末資産状況の現在の確定状態を取得条件にしない

判定履歴保存後に月末資産状況が確定解除され、

```text
confirmed = false
```

となっていても、過去の判定履歴は参照可能とする。

そのため、

```text
month_end_asset_snapshots.confirmed = true
```

をASM-004の取得条件へ含めない。

---

### 28.9 targetYearMonth

レスポンスの

```text
targetYearMonth
```

は、関連する

```text
month_end_asset_snapshots.target_year_month
```

から取得する。

判定履歴に対象年月を直接保存している設計の場合は、保存済み値を使用してもよい。

どちらを正とするかは、`assessment_histories`のテーブル定義を正とする。

---

### 28.10 users

`X-User-Id`による利用者コンテキスト確認では、

```text
users
```

を参照する。

ただし、`users`はASM-004固有の判定履歴詳細取得処理ではなく、API共通の利用者コンテキスト処理で使用する。

そのため、ASM-004の主な関連テーブルは、

```text
assessment_histories
objectives
month_end_asset_snapshots
```

とする。

---

### 28.11 assessment_historiesにuser_idを持つ場合

`assessment_histories`に直接

```text
user_id
```

を持つ設計の場合は、利用者境界を以下の条件で保証する。

```text
assessment_histories.id
    = assessmentHistoryId

AND

assessment_histories.user_id
    = UserContext.userId
```

この場合、判定履歴取得時点で利用者境界を確定できる。

---

### 28.12 assessment_historiesにuser_idを持たない場合

`assessment_histories`に直接`user_id`を持たない場合は、関連する目的などを介して利用者境界を保証する。

概念的には、

```text
assessment_histories
    ↓ objective_id
objectives
    ↓ user_id
users
```

として、

```text
objectives.user_id
    = UserContext.userId
```

を取得条件へ含める。

実際の方式は、テーブル定義を正とする。

---

### 28.13 他利用者の判定履歴を取得しない

利用者境界は、判定履歴取得時点から検索条件へ含める。

以下のように、

```text
assessmentHistoryIdだけで取得
    ↓
後からuserId比較
```

する方式を基本としない。

対象取得時点で利用者境界を保証する。

---

### 28.14 month_end_asset_balancesを直接参照しない

ASM-004では、判定結果を再計算しないため、

```text
month_end_asset_balances
```

を判定計算目的で参照しない。

月末残高は、ASM-001・ASM-002で判定実行時に使用する。

ASM-004では保存済み判定根拠を返却する。

---

### 28.15 month_end_holding_valuesを直接参照しない

同様に、

```text
month_end_holding_values
```

もASM-004では判定再計算目的で参照しない。

商品別月末評価額の現在値から過去の判定結果を再構築しない。

---

### 28.16 asset_account_available_settingsを直接参照しない

判定時に使用した利用可能資産設定についても、ASM-004で現在の設定履歴を再参照して利用可能資産額を再計算しない。

```text
asset_account_available_settings
```

は、主にASM-001・ASM-002の判定計算処理で使用する。

---

### 28.17 net_incomesを直接参照しない

ASM-004では、現在の

```text
net_incomes
```

から平均手取り収入を再計算しない。

判定時に使用した平均手取り収入が`assessment_histories`へ保存されている場合は、その保存値を返却する。

---

### 28.18 判定計算用テーブルとの責務分離

概念的には、以下のように分離する。

```text
ASM-001 / ASM-002
    ↓
objectives
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
net_incomes
    ↓
判定計算

ASM-002
    ↓
assessment_historiesへ保存

ASM-004
    ↓
assessment_historiesを中心に参照
```

ASM-004で判定計算処理を再利用しない。

---

### 28.19 外部キー制約

正常なデータ状態では、少なくとも概念的に以下の参照整合性をDB制約で保証する。

```text
assessment_histories.objective_id
    → objectives.id

assessment_histories.month_end_asset_snapshot_id
    → month_end_asset_snapshots.id
```

実際の外部キー定義は、テーブル定義書を正とする。

---

### 28.20 関連データ不整合

外部キー制約等によって通常は発生しないが、

```text
assessment_histories
    ↓
objective_id
    ↓
objectives不存在
```

または、

```text
assessment_histories
    ↓
month_end_asset_snapshot_id
    ↓
month_end_asset_snapshots不存在
```

のような状態が発生した場合は、利用者入力エラーとはしない。

内部データ不整合として、

```text
INTERNAL_SERVER_ERROR
```

へ変換する。

---

### 28.21 関連データをASM-004で修復しない

関連データに不整合が存在しても、ASM-004で以下を行わない。

- 判定履歴の削除
- `objective_id`の変更
- `month_end_asset_snapshot_id`の変更
- 不足する目的の自動作成
- 不足する月末資産状況の自動作成
- 計算根拠の再生成
- 判定結果の再計算

参照専用APIとして副作用を持たせない。

---

### 28.22 更新対象テーブルは存在しない

ASM-004では、以下のすべてを参照のみとする。

```text
assessment_histories
objectives
month_end_asset_snapshots
```

概念的には、

```text
SELECT
    ↓
Response
```

のみであり、

```text
INSERT
UPDATE
DELETE
```

を行わない。

---

## 29. 関連する機能要件

ASM-004 判定履歴詳細取得は、主に以下の機能要件に対応する。

| 機能要件 | 内容 | ASM-004との関係 |
|---|---|---|
| 11.8 判定結果の表示 | 目的達成判定の結果および判定根拠を表示する | 保存済み判定履歴の結果・計算根拠を詳細表示する |
| 11.10 判定履歴 | 過去に実行・保存した目的達成判定を履歴として参照できる | 指定した判定履歴1件の詳細を取得する |

---

### 29.1 11.8 判定結果の表示

ASM-004では、過去に保存された目的達成判定について、

```text
判定結果
+
判定時点の計算根拠
```

を取得する。

これにより、利用者は単に

```text
達成可能
達成不可
判定不可
```

という結果だけではなく、

```text
なぜその判定結果になったのか
```

を確認できる。

主に以下の情報を詳細表示へ利用する。

- 判定対象となった目的
- 判定対象年月
- 判定結果
- 判定時点の利用可能資産額
- 判定時点の必要額
- 判定時点の平均手取り収入
- 判定時点の必要生活防衛資金
- 判定後の残額
- 判定不可理由
- 判定実行日時

具体的な計算根拠項目は、ASM-002で保存する`assessment_histories`の内容を正とする。

---

### 29.2 判定時点の情報を表示する

11.8の判定結果表示では、ASM-004実行時点の現在値ではなく、

```text
ASM-002による判定実行時点
```

の情報を表示する。

そのため、ASM-004では現在の

```text
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
net_incomes
```

を利用して判定結果を再計算しない。

保存済み判定履歴を参照する。

---

### 29.3 判定不可も表示対象とする

判定結果が

```text
NOT_ASSESSABLE
```

であっても、正常な判定履歴として詳細表示できるようにする。

判定不可理由が保存されている場合は、

```text
notAssessableReason
```

などによって理由も確認できるようにする。

---

### 29.4 11.10 判定履歴

ASM-004は、11.10 判定履歴のうち、

```text
保存済み判定履歴1件を
詳細に参照する
```

部分を担当する。

ASM-003との責務分担は、以下とする。

```text
ASM-003
判定履歴一覧取得
    ↓
過去の判定履歴を一覧表示
    ↓
assessmentHistoryId選択
    ↓
ASM-004
判定履歴詳細取得
```

---

### 29.5 ASM-003との連携

ASM-003では、利用者が判定履歴を一覧から選択できる情報を返却する。

ASM-004では、その

```text
assessmentHistoryId
```

を使用して、詳細な判定結果と計算根拠を取得する。

概念的には、

```text
判定履歴一覧
    ↓
2026-08 / 一人暮らし / 達成可能
    ↓
詳細を表示
    ↓
ASM-004
    ↓
判定時点の計算根拠
```

とする。

---

### 29.6 過去履歴を現在状態から独立して参照する

11.10の判定履歴では、保存後に目的の状態が変化しても、過去履歴を保持する。

例えば、

```text
判定実行時
objective.enabled = true
    ↓
ASM-002
判定履歴保存
    ↓
後日
objective.enabled = false
```

となっても、ASM-004による過去履歴参照は可能とする。

---

### 29.7 月末資産状況の現在状態にも依存しない

判定履歴保存後に、関連する月末資産状況が確定解除された場合でも、過去に保存された判定履歴は参照可能とする。

```text
判定実行時
snapshot.confirmed = true
    ↓
判定履歴保存
    ↓
後日
snapshot.confirmed = false
    ↓
ASM-004
履歴参照可能
```

とする。

---

### 29.8 過去履歴を上書きしない

11.10 判定履歴では、再判定によって既存履歴を上書きしない。

同じ目的について再判定した場合は、

```text
既存履歴
    → 保持

新しい判定
    → 新しいassessment_histories
```

とする。

ASM-004では、指定された履歴1件をそのまま参照する。

---

### 29.9 ASM-002との関係

ASM-004で詳細表示できる情報は、ASM-002で判定履歴として保存された情報に依存する。

概念的には、

```text
ASM-002
判定実行
    ↓
判定結果
計算根拠
    ↓
assessment_histories保存
    ↓
ASM-004
詳細取得
```

とする。

そのため、11.8で表示が必要な判定時点情報が増えた場合は、ASM-004だけではなく、ASM-002の保存項目も見直す必要がある。

---

### 29.10 ASM-001との関係

ASM-001は判定結果を保存せず、プレビューとして現在の計算結果を返却する。

ASM-004は、ASM-001のプレビュー結果を参照するAPIではない。

```text
ASM-001
    → 保存なし

ASM-002
    → 履歴保存

ASM-004
    → ASM-002で保存した履歴参照
```

という関係とする。

---

### 29.11 ASM-004で担当しない機能要件

ASM-004では、以下の機能は担当しない。

- 目的達成判定の計算
- 判定プレビュー
- 判定結果の新規保存
- 再判定の実行
- 判定履歴一覧取得
- 目的の登録・更新・無効化
- 月末資産状況の確定・確定解除
- 判定履歴の更新・削除

これらは、それぞれの専用APIへ責務を分離する。

---

### 29.12 対応関係

ASM系APIと機能要件の対応を整理すると、概念的には以下とする。

```text
11.8 判定結果の表示
    ├─ ASM-001
    │    → プレビュー結果表示
    │
    ├─ ASM-002
    │    → 保存時の判定結果
    │
    └─ ASM-004
         → 保存済み判定結果の詳細表示

11.10 判定履歴
    ├─ ASM-003
    │    → 履歴一覧
    │
    └─ ASM-004
         → 履歴詳細
```

ASM-004では、

```text
保存済み判定履歴を
後から詳細に確認できること
```

を中心的な責務とする。

---

## 30. テスト観点

ASM-004では、指定された判定履歴について、

```text
正しい利用者境界で取得できること
+
保存済み判定結果を正しく取得できること
+
判定時点の計算根拠を正しく返却できること
+
現在状態から再計算しないこと
```

を中心にテストする。

主なテスト観点は、以下とする。

| 分類 | テスト観点 | 期待結果 |
|---|---|---|
| 正常系 | 操作対象利用者に属する判定履歴を指定 | `200 OK`で詳細を取得できる |
| 正常系 | 判定結果が達成可能 | 保存済みの達成可能結果を返却する |
| 正常系 | 判定結果が達成不可 | 保存済みの達成不可結果を返却する |
| 正常系 | 判定結果が判定不可 | `200 OK`で判定不可結果を返却する |
| 正常系 | 判定不可理由が保存されている | 保存済み理由を返却する |
| 正常系 | 判定時点の計算根拠が保存されている | 保存済み計算根拠を返却する |
| 正常系 | 目的が現在無効化済み | 過去の判定履歴を取得できる |
| 正常系 | 月末資産状況が現在未確定 | 過去の判定履歴を取得できる |
| 異常系 | `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| 異常系 | `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 異常系 | 利用者不存在 | `USER_NOT_FOUND` |
| 異常系 | `assessmentHistoryId`形式不正 | `INVALID_ASSESSMENT_HISTORY_ID` |
| 異常系 | 判定履歴不存在 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| 利用者境界 | 他利用者の判定履歴を指定 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| データ整合性 | 関連データに想定外の不整合 | `INTERNAL_SERVER_ERROR` |
| 副作用 | GET実行前後の業務データ | 変更されない |

---

### 30.1 正常取得

操作対象利用者に属する判定履歴を指定した場合、正常に詳細を取得できることを確認する。

例：

```text
X-User-Id = 1
assessmentHistoryId = 100

assessmentHistory 100
    ↓
User 1に所属
```

期待結果：

```http
200 OK
```

レスポンスの

```text
data.id
```

には、

```text
"100"
```

が設定されること。

---

### 30.2 レスポンス基本項目

正常取得時に、ASM-004で定義したレスポンス項目が正しく返却されることを確認する。

主に、

```text
id
objectiveId
objectiveName
targetYearMonth
assessmentResult
availableAssetAmount
requiredAmount
averageNetIncome
requiredEmergencyFund
remainingAmount
notAssessableReason
assessedAt
```

を確認する。

具体的な項目は、ASM-002の保存仕様および`assessment_histories`のテーブル定義を正とする。

---

### 30.3 ID型

DB上で`bigint`として保持されているIDが、APIレスポンスではstringとして返却されることを確認する。

例：

```json
{
  "id": "100",
  "objectiveId": "10"
}
```

以下のようなnumber型にならないことを確認する。

```json
{
  "id": 100
}
```

---

### 30.4 金額型

金額項目がintegerとして返却されることを確認する。

例：

```json
{
  "availableAssetAmount": 2500000,
  "requiredAmount": 1000000,
  "averageNetIncome": 300000
}
```

以下のような表示用文字列にならないことを確認する。

```json
{
  "availableAssetAmount": "2,500,000円"
}
```

---

### 30.5 targetYearMonth

判定対象年月が、

```text
YYYY-MM
```

形式で返却されることを確認する。

例：

```json
{
  "targetYearMonth": "2026-08"
}
```

---

### 30.6 assessedAt

判定実行日時が、API共通方針で定めた日時形式で返却されることを確認する。

例：

```json
{
  "assessedAt": "2026-08-31T12:00:00+09:00"
}
```

---

### 30.7 達成可能

保存済み判定結果が

```text
ACHIEVABLE
```

の場合、その値が変更されずに返却されることを確認する。

期待例：

```json
{
  "assessmentResult": "ACHIEVABLE"
}
```

---

### 30.8 達成不可

保存済み判定結果が

```text
NOT_ACHIEVABLE
```

の場合、正常レスポンスとして返却されることを確認する。

期待結果：

```http
200 OK
```

判定結果そのものをAPIエラーとして扱わないことを確認する。

---

### 30.9 判定不可

保存済み判定結果が

```text
NOT_ASSESSABLE
```

の場合も、

```http
200 OK
```

となることを確認する。

以下のようなHTTPエラーにしない。

```text
400
409
422
```

---

### 30.10 判定不可理由

判定不可理由が保存されている場合、その理由コードが正しく返却されることを確認する。

例：

```json
{
  "assessmentResult": "NOT_ASSESSABLE",
  "notAssessableReason": "NET_INCOME_INSUFFICIENT"
}
```

判定不可理由のコード値は、ASM-001・ASM-002との共通定義を正とする。

---

### 30.11 判定可能時のnotAssessableReason

判定結果が

```text
ACHIEVABLE
```

または

```text
NOT_ACHIEVABLE
```

の場合、仕様に従って

```json
{
  "notAssessableReason": null
}
```

となることを確認する。

---

### 30.12 保存済み計算根拠

`assessment_histories`へ保存された計算根拠が、そのままレスポンスへ反映されることを確認する。

例えば、

```text
availableAssetAmount
    = 2,500,000

averageNetIncome
    = 300,000

requiredEmergencyFund
    = 900,000

remainingAmount
    = 600,000
```

が保存されている場合、同じ値が返却されることを確認する。

---

### 30.13 現在の資産状況から再計算しない

判定履歴保存後に現在の資産状況を変更しても、ASM-004の判定結果および保存済み計算根拠が変化しないことを確認する。

例：

```text
判定時

availableAssetAmount
    = 2,000,000

    ↓

判定履歴保存

    ↓

現在

資産残高を変更

現在の利用可能資産額
    = 3,000,000

    ↓

ASM-004
```

期待結果：

```text
availableAssetAmount
    = 2,000,000
```

現在値の

```text
3,000,000
```

へ変更されないこと。

---

### 30.14 現在の手取り収入から再計算しない

判定履歴保存後に`net_incomes`が変更・追加されても、保存済みの

```text
averageNetIncome
```

が変化しないことを確認する。

ASM-004実行時に直近3か月平均を再計算しないことを確認する。

---

### 30.15 現在の利用可能資産設定を使用しない

判定履歴保存後に

```text
asset_account_available_settings
```

へ新しい設定が登録されても、過去の

```text
availableAssetAmount
```

が変化しないことを確認する。

---

### 30.16 現在の月末残高を使用しない

判定履歴保存後に

```text
month_end_asset_balances
```

の状態が変化しても、ASM-004で判定結果を再計算しないことを確認する。

---

### 30.17 現在の商品別月末評価額を使用しない

判定履歴保存後に

```text
month_end_holding_values
```

の状態が変化しても、ASM-004の保存済み判定結果へ影響しないことを確認する。

---

### 30.18 目的が現在無効化済み

判定実行後に関連する目的が無効化された場合でも、判定履歴を取得できることを確認する。

```text
判定履歴保存
    ↓
目的無効化
    ↓
ASM-004
```

期待結果：

```http
200 OK
```

以下のようなエラーにならないこと。

```text
OBJECTIVE_DISABLED
ASSESSMENT_HISTORY_NOT_FOUND
```

---

### 30.19 月末資産状況が現在未確定

判定履歴保存後に関連する月末資産状況が確定解除された場合でも、判定履歴を取得できることを確認する。

```text
判定履歴保存
    ↓
月末資産状況確定解除
    ↓
ASM-004
```

期待結果：

```http
200 OK
```

現在の`confirmed`を取得可否条件にしていないことを確認する。

---

### 30.20 複数の判定履歴

同一目的について複数回判定している場合でも、指定した`assessmentHistoryId`に対応する履歴だけが返却されることを確認する。

例：

```text
Objective 10
    ├─ History 100
    ├─ History 101
    └─ History 102
```

```http
GET /api/v1/assessment-histories/101
```

の場合は、History 101の保存済み情報を返却すること。

---

### 30.21 過去履歴を最新履歴で上書きしない

同一目的について再判定が行われた後でも、古い`assessmentHistoryId`を指定すれば古い判定結果を取得できることを確認する。

```text
History 100
ACHIEVABLE

    ↓

再判定

History 101
NOT_ACHIEVABLE
```

この状態で、

```http
GET /api/v1/assessment-histories/100
```

を実行した場合、

```text
ACHIEVABLE
```

が返却されること。

---

### 30.22 X-User-Id未指定

`X-User-Id`を指定しない場合、API共通仕様に従って

```text
USER_CONTEXT_REQUIRED
```

となることを確認する。

判定履歴検索へ進まないことも確認する。

---

### 30.23 X-User-Id形式不正

以下のような`X-User-Id`を指定した場合、

```text
0
-1
abc
1.5
```

```text
INVALID_USER_ID
```

となることを確認する。

---

### 30.24 利用者不存在

形式上有効な`X-User-Id`であっても、該当利用者が存在しない場合は、

```text
USER_NOT_FOUND
```

となることを確認する。

HTTPステータスは、

```http
404 Not Found
```

とする。

---

### 30.25 assessmentHistoryId形式不正

以下のような`assessmentHistoryId`について形式不正として扱われることを確認する。

```text
0
-1
abc
1.5
1e3
100abc
```

期待するエラーコード：

```text
INVALID_ASSESSMENT_HISTORY_ID
```

---

### 30.26 判定履歴不存在

形式上有効な`assessmentHistoryId`であっても、対象履歴が存在しない場合は、

```http
404 Not Found
```

および、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

となることを確認する。

---

### 30.27 他利用者の判定履歴

他利用者に属する判定履歴IDを指定した場合、取得できないことを確認する。

例：

```text
X-User-Id
    = 1

assessmentHistoryId
    = 100

History 100の所有者
    = User 2
```

期待結果：

```http
404 Not Found
```

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

---

### 30.28 他利用者所属を公開しない

以下をクライアントから区別できないことを確認する。

```text
存在しない判定履歴
        ↓
ASSESSMENT_HISTORY_NOT_FOUND

他利用者の判定履歴
        ↓
ASSESSMENT_HISTORY_NOT_FOUND
```

以下のようなエラーコードを返却しない。

```text
ASSESSMENT_HISTORY_FORBIDDEN
OTHER_USER_ASSESSMENT_HISTORY
```

---

### 30.29 利用者境界を取得条件へ含める

RepositoryまたはQueryのテストでは、単純に

```text
assessment_histories.id
    = assessmentHistoryId
```

だけで取得していないことを確認する。

テーブル設計に応じて、

```text
assessment_histories.user_id
    = UserContext.userId
```

または、

```text
objectives.user_id
    = UserContext.userId
```

などの利用者境界が取得条件へ含まれていることを確認する。

---

### 30.30 論理削除済み判定履歴

`assessment_histories`にSoftDeletesを採用している場合は、論理削除済み履歴が通常取得できないことを確認する。

期待結果：

```http
404 Not Found
```

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

SoftDeletesを採用しない場合、このテストは不要とする。

---

### 30.31 関連目的の想定外不存在

DB制約上は通常発生しないが、テスト上意図的に

```text
assessment_histories.objective_id
    ↓
対応するobjectivesなし
```

という状態を作成できる場合は、内部データ不整合として処理されることを確認する。

期待結果：

```http
500 Internal Server Error
```

```text
INTERNAL_SERVER_ERROR
```

---

### 30.32 関連月末資産状況の想定外不存在

同様に、

```text
assessment_histories
    ↓
month_end_asset_snapshot_id
    ↓
対応するmonth_end_asset_snapshotsなし
```

の場合も、利用者入力エラーではなく内部エラーとして扱うことを確認する。

---

### 30.33 不足データを自動補完しない

関連データに不整合が存在した場合でも、ASM-004によって

```text
目的を自動作成する
月末資産状況を自動作成する
判定根拠を再計算する
判定履歴を更新する
```

などの処理が行われないことを確認する。

---

### 30.34 DBカラム名を直接返却しない

レスポンスに、

```text
objective_id
assessment_result
available_asset_amount
assessed_at
```

などのsnake_caseのDBカラム名が直接公開されないことを確認する。

API仕様どおり、

```text
objectiveId
assessmentResult
availableAssetAmount
assessedAt
```

として返却されることを確認する。

---

### 30.35 内部情報を返却しない

レスポンスに、不要な内部情報が含まれないことを確認する。

例えば、

```text
user_id
deleted_at
SQL
SQLSTATE
DB制約名
Laravel例外情報
スタックトレース
サーバーファイルパス
```

などを外部へ公開しない。

---

### 30.36 userIdをレスポンスへ含めない

操作対象利用者は`X-User-Id`によって確定しているため、

```text
userId
```

をASM-004のレスポンス項目として返却しないことを確認する。

---

### 30.37 Request Bodyを必要としない

ASM-004はGET APIであるため、Request Bodyなしで正常に実行できることを確認する。

```http
GET /api/v1/assessment-histories/100
X-User-Id: 1
Accept: application/json
```

だけで取得できること。

---

### 30.38 クエリパラメータを必要としない

以下のような追加パラメータなしで判定履歴詳細を取得できることを確認する。

```text
objectiveId
snapshotId
userId
includeDetails
```

取得対象は、

```text
assessmentHistoryId
+
利用者コンテキスト
```

によって決定する。

---

### 30.39 トランザクションを必要としない

ASM-004実行時に、更新処理を前提とした明示的なトランザクションが使用されていないことを確認する。

特に、

```php
DB::transaction(...)
```

が不要に導入されていないことを実装レビューでも確認する。

---

### 30.40 ロックを取得しない

ASM-004実行時に、

```php
lockForUpdate()
```

などの悲観ロックを取得しないことを確認する。

ASM-004による参照によって、他の更新処理を不要にブロックしないこと。

---

### 30.41 副作用がない

ASM-004実行前後で、以下の業務テーブルのデータが変更されないことを確認する。

```text
assessment_histories
objectives
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
net_incomes
```

ASM-004は参照専用APIとして動作すること。

---

### 30.42 冪等性

同一Requestを複数回実行しても、業務データへ追加の変更が発生しないことを確認する。

例：

```text
GET 1回目
    ↓
200 OK

GET 2回目
    ↓
200 OK

GET 3回目
    ↓
200 OK
```

この間にASM-004自身によって判定履歴が追加・更新されないこと。

---

### 30.43 ASM-003からの遷移

ASM-003で取得した

```text
assessmentHistoryId
```

をASM-004へ渡した場合に、対応する判定履歴詳細を正常に取得できることを確認する。

概念的には、

```text
ASM-003
    ↓
History 100を選択
    ↓
ASM-004
/api/v1/assessment-histories/100
    ↓
History 100の詳細
```

となること。

---

### 30.44 ASM-002との整合性

ASM-002で判定結果を保存した直後にASM-004でその履歴を取得した場合、保存された内容と取得内容が一致することを確認する。

特に、

```text
assessmentResult
availableAssetAmount
requiredAmount
averageNetIncome
requiredEmergencyFund
remainingAmount
notAssessableReason
```

などの判定結果・計算根拠について整合していることを確認する。

---

### 30.45 ASM-001のプレビュー結果を取得対象にしない

ASM-001は判定結果を保存しないため、プレビューを実行しただけではASM-004で取得可能な判定履歴が生成されないことを確認する。

```text
ASM-001
    ↓
プレビュー
    ↓
assessment_histories
登録なし
    ↓
ASM-004で新規履歴として取得不可
```

となること。

---

### 30.46 Feature Test

LaravelのFeature Testでは、主に以下を確認する。

- 正常取得
- レスポンスJSON構造
- IDのstring変換
- 金額のinteger変換
- 判定結果ごとのレスポンス
- 判定不可理由
- `X-User-Id`未指定
- `X-User-Id`形式不正
- 利用者不存在
- `assessmentHistoryId`形式不正
- 判定履歴不存在
- 他利用者境界
- 目的無効化後の履歴取得
- 月末資産状況確定解除後の履歴取得
- 保存済み計算根拠の維持
- 現在値から再計算しないこと
- 副作用がないこと

---

### 30.47 Query / Repository Test

判定履歴取得をQueryまたはRepositoryへ分離する場合は、以下を確認する。

```text
assessmentHistoryId
+
UserContext.userId
```

によって正しい判定履歴を取得できること。

また、

```text
他利用者の履歴
```

を取得できないことを確認する。

---

### 30.48 Resource Test

API Resourceについては、DBまたはDTOの値がAPI仕様どおりに変換されることを確認する。

主に、

```text
bigint ID
    ↓
string

snake_case
    ↓
camelCase

金額
    ↓
integer

日時
    ↓
API共通日時形式
```

を確認する。

---

### 30.49 テストで確認しない責務

ASM-004のテストでは、以下の判定ロジックそのものを中心的なテスト対象としない。

- 利用可能資産額の算出ロジック
- 平均手取り収入の算出ロジック
- 必要生活防衛資金の算出ロジック
- 目的達成可否の判定ロジック
- 判定不可条件の判定ロジック
- 月末残高の集計ロジック
- 商品別評価額の集計ロジック

これらは、ASM-001・ASM-002および判定ドメインロジック側のテスト責務とする。

ASM-004では、

```text
保存済み判定履歴を
正しい利用者境界で取得し、
保存時点の結果と計算根拠を
変更せずAPI形式へ変換して返却できること
```

を中心にテストする。

---

## 31. Laravel実装方針

ASM-004では、Action、UseCase、Query、DTO、API Resource、Responderを分離して実装する。

参照専用GET APIであるため、Repositoryは原則として使用しない。

概念的な構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
Action
    ↓
UseCase
    ├─ AssessmentHistoryQuery
    ├─ ObjectiveQuery
    └─ MonthEndAssetSnapshotQuery
    ↓
AssessmentHistoryDetailResult DTO
    ↓
API Resource
    ↓
Responder
```

ASM-004では、保存済みの判定履歴を正として取得し、現在の資産状況から目的達成判定を再計算しない。

---

### 31.1 Route

ASM-004は、以下のルートとして定義する。

概念例：

```php
Route::get(
    '/api/v1/assessment-histories/{assessmentHistoryId}',
    ShowAssessmentHistoryAction::class,
);
```

`assessmentHistoryId`は、正の整数形式のみ許可する。

概念例：

```php
Route::get(
    '/api/v1/assessment-histories/{assessmentHistoryId}',
    ShowAssessmentHistoryAction::class,
)
    ->where(
        'assessmentHistoryId',
        '[1-9][0-9]*',
    );
```

ただし、形式不正時に

```text
INVALID_ASSESSMENT_HISTORY_ID
```

を返却する共通方針がある場合は、Route制約だけで完結させず、共通のパスパラメータ検証方式に従う。

---

### 31.2 Middleware

以下の共通Middlewareを適用する。

- 利用者コンテキスト設定
- リクエストID生成
- 共通例外処理
- ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証する。

概念的には、

```text
X-User-Id取得
    ↓
必須確認
    ↓
形式確認
    ↓
users存在確認
    ↓
UserContext設定
    ↓
Action
```

とする。

Action以降では、検証済みの利用者コンテキストを使用する。

---

### 31.3 FormRequest

ASM-004では、

```text
Request Bodyなし
Query Parameterなし
```

であるため、専用FormRequestは原則として作成しない。

以下のような空FormRequestを形式的に追加しない。

```php
final class ShowAssessmentHistoryRequest
    extends FormRequest
{
    public function rules(): array
    {
        return [];
    }
}
```

`assessmentHistoryId`は、RouteまたはAPI共通のパスパラメータ検証方式で扱う。

---

### 31.4 assessmentHistoryIdの形式検証

`assessmentHistoryId`は、正の整数形式を必須とする。

正常例：

```text
1
100
999
```

不正例：

```text
0
-1
abc
1.5
1e3
100abc
```

形式不正の場合は、

```text
INVALID_ASSESSMENT_HISTORY_ID
```

へ変換する。

---

### 31.5 Action

Actionは、

```text
assessmentHistoryId
+
UserContext
```

を受け取り、UseCaseを呼び出す。

概念例：

```php
final class ShowAssessmentHistoryAction
{
    public function __invoke(
        string $assessmentHistoryId,
        ShowAssessmentHistoryUseCase $useCase,
        ShowAssessmentHistoryResponder $responder,
        UserContext $userContext,
    ): JsonResponse {
        $result =
            $useCase->execute(
                userId:
                    $userContext->userId,

                assessmentHistoryId:
                    (int) $assessmentHistoryId,
            );

        return $responder->ok(
            $result,
        );
    }
}
```

---

### 31.6 Actionで行わないこと

Actionでは、以下を行わない。

- 判定履歴検索
- 利用者境界判定
- 目的検索
- 月末資産状況検索
- 判定結果の再計算
- 計算根拠の再生成
- DBアクセス
- レスポンス配列生成
- エラーコード判定

Actionは、

```text
HTTP入力
    ↓
UseCase
    ↓
Responder
```

の橋渡しに責務を限定する。

---

### 31.7 UseCase

ASM-004のアプリケーション処理全体を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. `assessmentHistoryId`を受け取る
3. 利用者境界を含めて判定履歴を取得する
4. 対象不存在の場合は業務例外を送出する
5. 必要に応じて目的情報を取得する
6. 必要に応じて月末資産状況を取得する
7. 保存済みの判定結果と計算根拠を取得する
8. Result DTOへ変換する
9. Result DTOを返却する

---

### 31.8 UseCaseの概念フロー

```text
userId
+
assessmentHistoryId
    ↓
AssessmentHistoryQuery
    ↓
判定履歴取得
    ↓
存在する？
    ├─ No
    │    ↓
    │ ASSESSMENT_HISTORY_NOT_FOUND
    │
    └─ Yes
         ↓
      必要な関連情報取得
         ↓
      保存済み判定結果取得
         ↓
      保存済み計算根拠取得
         ↓
      Result DTO
```

---

### 31.9 AssessmentHistoryQuery

判定履歴の取得は、専用Queryへ委譲する。

概念例：

```php
$assessmentHistory =
    $this->assessmentHistoryQuery
        ->findDetailByIdAndUser(
            assessmentHistoryId:
                $assessmentHistoryId,

            userId:
                $userId,
        );
```

ASM-004ではQueryが読み取り責務を担う。

---

### 31.10 利用者境界をQueryへ含める

判定履歴を、

```php
AssessmentHistory::find(
    $assessmentHistoryId,
);
```

のようにIDだけで取得した後に利用者を確認する方式を基本としない。

対象取得時点から利用者境界を含める。

`assessment_histories`に`user_id`を持つ場合は、概念的に、

```text
assessment_histories.id
    = assessmentHistoryId

AND

assessment_histories.user_id
    = userId
```

とする。

---

### 31.11 assessment_historiesにuser_idを持たない場合

`assessment_histories`に直接`user_id`を持たない場合は、関連する目的を介して利用者境界を保証する。

概念的には、

```text
assessment_histories
    ↓ objective_id
objectives
    ↓ user_id
```

として、

```text
objectives.user_id
    = userId
```

をQuery条件へ含める。

実際の方式は、テーブル定義を正とする。

---

### 31.12 ASSESSMENT_HISTORY_NOT_FOUND

対象判定履歴を取得できない場合は、

```php
throw new
    AssessmentHistoryNotFoundException();
```

とする。

以下を同じ例外へ集約する。

- 判定履歴不存在
- 他利用者所属
- SoftDeletes採用時の論理削除済み

最終的に、

```text
404 Not Found
ASSESSMENT_HISTORY_NOT_FOUND
```

へ変換する。

---

### 31.13 他利用者所属専用例外を作らない

以下のような専用例外は作成しない。

```text
AssessmentHistoryForbiddenException
```

他利用者所属も`AssessmentHistoryNotFoundException`として扱う。

他利用者の履歴存在をクライアントへ公開しない。

---

### 31.14 判定履歴を取得の起点とする

ASM-004では、取得の起点を

```text
assessment_histories
```

とする。

以下のように現在の目的や現在の月末資産状況から判定結果を再構築しない。

```text
objectives
+
month_end_asset_snapshots
+
net_incomes
+
asset_account_available_settings
    ↓
再判定
```

ASM-004ではこの処理を行わない。

---

### 31.15 判定結果は保存済み値を使用する

例えば、

```text
assessment_result
available_asset_amount
required_amount
average_net_income
required_emergency_fund
remaining_amount
not_assessable_reason
```

などが`assessment_histories`へ保存されている場合は、その値をそのまま使用する。

---

### 31.16 現在の資産情報を参照しない

判定結果・計算根拠の取得目的で、

```text
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
net_incomes
```

を再参照しない。

これらはASM-001・ASM-002の判定計算処理で使用する。

---

### 31.17 現在値から再計算しない

UseCase内で、

```php
$availableAssets =
    $calculator->calculate(...);

$result =
    $assessmentService->assess(...);
```

のような判定ロジックを呼び出さない。

ASM-004は、保存済み判定履歴取得に責務を限定する。

---

### 31.18 ObjectiveQuery

目的情報を別Queryで取得する必要がある場合は、`ObjectiveQuery`を使用する。

概念例：

```php
$objective =
    $this->objectiveQuery
        ->findForAssessmentHistory(
            objectiveId:
                $assessmentHistory->objectiveId,
        );
```

ただし、判定時点の目的情報を`assessment_histories`へスナップショット保存している場合は、不要な目的Queryを行わない。

---

### 31.19 目的の現在のenabledを取得条件にしない

目的を参照する場合でも、

```text
enabled = true
```

を取得条件へ含めない。

現在無効化済みの目的でも、過去の判定履歴は参照可能とする。

---

### 31.20 目的のSoftDeletes

目的が論理削除済みでも過去判定履歴を参照可能とする仕様の場合は、EloquentのSoftDeletes Global Scopeに注意する。

必要に応じて、

```php
withTrashed()
```

を使用する、または判定時点の目的名等を履歴へ保存して現在の`objectives`への依存を減らす。

---

### 31.21 objectiveNameの取得方針

判定時点の目的名を`assessment_histories`へ保存している場合は、保存済み値を使用する。

概念例：

```php
$objectiveName =
    $assessmentHistory
        ->objective_name;
```

保存していない場合だけ、`objectives.name`を参照する。

---

### 31.22 判定時点情報はスナップショット保存を優先する

ASM-004で過去時点の値を正確に再現する必要がある情報は、

```text
assessment_histories
```

へ保存する設計を優先する。

例えば、

```text
objectiveName
requiredAmount
averageNetIncome
requiredEmergencyFund
availableAssetAmount
remainingAmount
```

などである。

現在テーブルを参照して過去値を推測しない。

---

### 31.23 MonthEndAssetSnapshotQuery

判定対象年月などを`month_end_asset_snapshots`から取得する場合は、専用Queryを使用する。

概念例：

```php
$snapshot =
    $this->monthEndAssetSnapshotQuery
        ->findForAssessmentHistory(
            snapshotId:
                $assessmentHistory
                    ->monthEndAssetSnapshotId,
        );
```

---

### 31.24 confirmedを取得条件にしない

月末資産状況を取得する場合でも、

```text
confirmed = true
```

を条件へ含めない。

判定履歴保存後に確定解除されていても、過去履歴は参照可能とする。

---

### 31.25 targetYearMonthの取得方針

判定時点の対象年月を`assessment_histories`へ保存している場合は、保存値を使用する。

保存していない場合は、

```text
month_end_asset_snapshots.target_year_month
```

を参照する。

過去時点の表示内容を完全に固定したい場合は、履歴側への保存を優先する。

---

### 31.26 JOINでまとめて取得してよい

ASM-004では、必要な情報が明確であるため、`AssessmentHistoryQuery`内で

```text
assessment_histories
+
objectives
+
month_end_asset_snapshots
```

をJOINして1回のQueryで取得してもよい。

概念例：

```php
AssessmentHistory::query()
    ->select([
        'assessment_histories.id',
        'assessment_histories.objective_id',
        'assessment_histories.assessment_result',
        'assessment_histories.available_asset_amount',
        'assessment_histories.required_amount',
        'assessment_histories.average_net_income',
        'assessment_histories.required_emergency_fund',
        'assessment_histories.remaining_amount',
        'assessment_histories.not_assessable_reason',
        'assessment_histories.assessed_at',
        'objectives.name as objective_name',
        'month_end_asset_snapshots.target_year_month',
    ])
    ->join(
        'objectives',
        'objectives.id',
        '=',
        'assessment_histories.objective_id',
    )
    ->join(
        'month_end_asset_snapshots',
        'month_end_asset_snapshots.id',
        '=',
        'assessment_histories.month_end_asset_snapshot_id',
    );
```

実際のカラム名は、テーブル定義を正とする。

---

### 31.27 JOIN時の利用者境界

JOINで取得する場合も、利用者境界を必ずQuery条件へ含める。

例えば、`objectives.user_id`を利用者境界とする場合は、

```php
->where(
    'objectives.user_id',
    $userId,
)
```

を含める。

---

### 31.28 JOIN時のSoftDeletesに注意する

目的や月末資産状況にSoftDeletesを採用している場合、通常のJOINではEloquent Global Scopeが期待どおり適用されない場合がある。

ASM-004の

```text
過去履歴を現在状態に依存せず参照する
```

という仕様に合わせて、JOIN条件を明示する。

---

### 31.29 N+1を発生させない

ASM-004は単一履歴取得APIなので大量N+1は発生しにくいが、関連情報を1項目ずつ不要にQueryしない。

例えば、

```text
履歴取得
    ↓
目的取得
    ↓
月末資産状況取得
```

の3Queryが明確で十分軽量なら許容できる。

一方、必要情報をJOINで自然に取得できる場合は、Queryをまとめてもよい。

過剰な最適化も避ける。

---

### 31.30 Repositoryを使用しない

ASM-004ではデータ更新を行わないため、Repositoryは原則として使用しない。

概念的には、

```text
Query
    → SELECT

Repository
    → INSERT / UPDATE / DELETE
```

というプロジェクト共通方針に従う。

---

### 31.31 Queryで更新しない

`AssessmentHistoryQuery`では、以下を行わない。

- 判定履歴更新
- 判定履歴削除
- 計算根拠補完
- 目的更新
- 月末資産状況更新
- 再判定

Queryは読み取りだけに責務を限定する。

---

### 31.32 Result DTO

取得した判定履歴詳細を、専用Result DTOとして表現する。

概念例：

```php
final readonly class AssessmentHistoryDetailResult
{
    public function __construct(
        public int $id,
        public int $objectiveId,
        public string $objectiveName,
        public string $targetYearMonth,
        public string $assessmentResult,
        public ?int $availableAssetAmount,
        public ?int $requiredAmount,
        public ?int $averageNetIncome,
        public ?int $requiredEmergencyFund,
        public ?int $remainingAmount,
        public ?string $notAssessableReason,
        public string $assessedAt,
    ) {
    }
}
```

具体的な項目・NULL可否は、ASM-002の保存仕様を正とする。

---

### 31.33 DTOへEloquent Modelを保持しない

以下のようなResult DTOは基本としない。

```php
final readonly class AssessmentHistoryDetailResult
{
    public function __construct(
        public AssessmentHistory $model,
    ) {
    }
}
```

APIレスポンスに必要な値だけを保持する。

---

### 31.34 Enumを使用してよい

`assessmentResult`や`notAssessableReason`をEnumとして管理している場合は、UseCaseまたはQuery結果変換時に共通Enumを利用する。

例えば、

```php
AssessmentResult::from(
    $row->assessment_result,
);
```

とする。

ASM-004専用の異なるコード定義を作らない。

---

### 31.35 ASM-001・ASM-002とEnumを共通化する

以下の値は、ASM系APIで共通化する。

```text
ACHIEVABLE
NOT_ACHIEVABLE
NOT_ASSESSABLE
```

判定不可理由についても、ASM-001・ASM-002・ASM-004で同じEnumまたはValue Objectを使用する。

---

### 31.36 API Resource

Result DTOを、専用API ResourceでJSON形式へ変換する。

概念例：

```php
final class AssessmentHistoryDetailResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id'
                => (string) $this->id,

            'objectiveId'
                => (string) $this->objectiveId,

            'objectiveName'
                => $this->objectiveName,

            'targetYearMonth'
                => $this->targetYearMonth,

            'assessmentResult'
                => $this->assessmentResult,

            'availableAssetAmount'
                => $this->availableAssetAmount,

            'requiredAmount'
                => $this->requiredAmount,

            'averageNetIncome'
                => $this->averageNetIncome,

            'requiredEmergencyFund'
                => $this->requiredEmergencyFund,

            'remainingAmount'
                => $this->remainingAmount,

            'notAssessableReason'
                => $this->notAssessableReason,

            'assessedAt'
                => $this->assessedAt,
        ];
    }
}
```

---

### 31.37 IDをstringへ変換する

API Resourceでは、

```php
(string) $this->id
```

のように、`bigint`のIDをstringへ変換する。

主に、

```text
id
objectiveId
```

が対象となる。

---

### 31.38 金額を文字列化しない

以下の金額項目は、

```text
availableAssetAmount
requiredAmount
averageNetIncome
requiredEmergencyFund
remainingAmount
```

integerまたはnullとして返却する。

以下のような表示用文字列へ変換しない。

```php
number_format(
    $this->availableAssetAmount,
) . '円';
```

表示フォーマットはReact側で行う。

---

### 31.39 日時変換

`assessedAt`は、API共通方針の日時形式へ変換する。

例えば、Carbonを使用する場合は、

```php
$this->assessedAt
    ->toIso8601String();
```

などを利用してよい。

ただし、タイムゾーン・形式はAPI共通方針を正とする。

---

### 31.40 Resourceで再計算しない

API Resourceでは、

```text
availableAssetAmount
-
requiredAmount
-
requiredEmergencyFund
```

などを行って`remainingAmount`を再計算しない。

Resourceは表現変換だけを担当する。

---

### 31.41 Responder

Responderは、Result DTOを`200 OK`レスポンスへ変換する。

概念例：

```php
final class ShowAssessmentHistoryResponder
{
    public function ok(
        AssessmentHistoryDetailResult $result,
    ): JsonResponse {
        return response()->json(
            [
                'data'
                    =>
                    new AssessmentHistoryDetailResource(
                        $result,
                    ),
            ],
            Response::HTTP_OK,
        );
    }
}
```

実際のEnvelope生成は、API共通方針に従う。

---

### 31.42 Responderで行わないこと

Responderでは、以下を行わない。

- 判定履歴検索
- 利用者境界判定
- 関連目的検索
- 月末資産状況検索
- 再判定
- 計算根拠生成
- DBアクセス
- 業務例外判定

HTTPレスポンス生成だけに責務を限定する。

---

### 31.43 トランザクションを使用しない

ASM-004は参照専用GET APIであるため、

```php
DB::transaction(...)
```

を原則として使用しない。

複数更新を原子的に保証する必要がないためである。

---

### 31.44 lockForUpdateを使用しない

ASM-004では、

```php
lockForUpdate()
```

を使用しない。

判定履歴、目的、月末資産状況を参照するだけであり、更新APIを不要にブロックしない。

---

### 31.45 キャッシュ

Phase1では、ASM-004専用のサーバー側キャッシュを原則として導入しない。

React側では、TanStack QueryによるQuery Cacheを使用してよい。

---

### 31.46 例外変換

主な例外変換は、以下とする。

| 内部状態 | エラーコード |
|---|---|
| `X-User-Id`未指定 | `USER_CONTEXT_REQUIRED` |
| `X-User-Id`形式不正 | `INVALID_USER_ID` |
| 利用者不存在 | `USER_NOT_FOUND` |
| `assessmentHistoryId`形式不正 | `INVALID_ASSESSMENT_HISTORY_ID` |
| 判定履歴不存在 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| 他利用者所属 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| 論理削除済み判定履歴 | `ASSESSMENT_HISTORY_NOT_FOUND` |
| 関連データの想定外不整合 | `INTERNAL_SERVER_ERROR` |
| その他想定外例外 | `INTERNAL_SERVER_ERROR` |

---

### 31.47 AssessmentHistoryNotFoundException

対象履歴を取得できない場合は、

```php
throw new
    AssessmentHistoryNotFoundException();
```

とする。

最終的に、

```text
404 Not Found
ASSESSMENT_HISTORY_NOT_FOUND
```

へ変換する。

---

### 31.48 関連データ不整合

例えば、

```text
assessment_histories.objective_id
    ↓
objectives不存在
```

または、

```text
assessment_histories.month_end_asset_snapshot_id
    ↓
month_end_asset_snapshots不存在
```

などの状態は、利用者入力エラーとして扱わない。

外部キー制約で原則として防止する。

万一発生した場合は、

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

として扱う。

---

### 31.49 保存済み判定根拠の欠落

テーブル定義上必須となっている判定根拠が欠落している場合も、現在値から補完しない。

例えば、

```text
assessmentResult = ACHIEVABLE

しかし
availableAssetAmount = NULL
```

が仕様上あり得ない場合は、内部データ不整合として扱う。

---

### 31.50 想定外例外

想定外の例外は、API共通Exception Handlerで

```text
500 Internal Server Error
INTERNAL_SERVER_ERROR
```

へ変換する。

レスポンスへ、以下を含めない。

- SQL
- SQLSTATE
- PostgreSQL内部エラー
- テーブル名
- カラム名
- 制約名
- Laravel内部例外メッセージ
- PHP内部エラー
- スタックトレース
- サーバーファイルパス

---

### 31.51 ログ

ASM-004では、必要に応じて以下をログコンテキストへ設定する。

```text
requestId
userId
apiId
assessmentHistoryId
httpStatus
errorCode
```

`apiId`は、

```text
ASM-004
```

とする。

---

### 31.52 正常時ログ

正常終了時は、必要に応じて

```text
requestId
userId
apiId = ASM-004
assessmentHistoryId
httpStatus = 200
```

を記録する。

判定根拠となる詳細な金額を通常のアクセスログへ不要に出力しない。

---

### 31.53 Not Found時ログ

`ASSESSMENT_HISTORY_NOT_FOUND`の場合は、必要に応じて

```text
requestId
userId
assessmentHistoryId
errorCode
httpStatus
```

を記録する。

ただし、

```text
本当に不存在だった
他利用者所属だった
```

という内部判定理由をAPIレスポンスへは公開しない。

---

### 31.54 Feature Test

LaravelのFeature Testでは、主に以下を確認する。

- 正常取得
- `200 OK`
- IDのstring変換
- 金額のinteger変換
- 判定結果コード
- 判定不可理由
- `assessedAt`の日時形式
- `X-User-Id`未指定
- `X-User-Id`形式不正
- 利用者不存在
- `assessmentHistoryId`形式不正
- 判定履歴不存在
- 他利用者境界
- 目的無効化後も参照可能
- 月末資産状況確定解除後も参照可能
- 現在資産から再計算しない
- 現在手取り収入から再計算しない
- 副作用なし

---

### 31.55 AssessmentHistoryQueryのDatabase Test

以下の条件で正しく取得できることを確認する。

```text
assessmentHistoryId一致
+
利用者境界一致
```

また、以下を取得できないことを確認する。

- 存在しない判定履歴
- 他利用者の判定履歴
- 論理削除済み判定履歴
  - SoftDeletes採用時のみ

---

### 31.56 利用者境界Test

例えば、

```text
User 1
    History 100

User 2
    History 200
```

という状態で、

```text
userId = 1
assessmentHistoryId = 200
```

を指定した場合、Query結果が`null`となることを確認する。

---

### 31.57 保存済み値取得Test

判定履歴に、

```text
assessmentResult
availableAssetAmount
requiredAmount
averageNetIncome
requiredEmergencyFund
remainingAmount
notAssessableReason
```

を設定しておき、Result DTOへ同じ値が設定されることを確認する。

---

### 31.58 現在値変更後のTest

判定履歴保存後に、

```text
net_incomes
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
```

を変更する。

その後ASM-004を実行しても、保存済みの判定根拠が変更されないことを確認する。

---

### 31.59 目的無効化後のTest

判定履歴保存後に対象目的を

```text
enabled = false
```

へ変更する。

その後もASM-004で履歴詳細を取得できることを確認する。

---

### 31.60 月末資産状況確定解除後のTest

判定履歴保存後に対象月末資産状況を

```text
confirmed = false
```

へ変更する。

その後もASM-004で履歴詳細を取得できることを確認する。

---

### 31.61 API Resource Test

API Resourceでは、主に以下を確認する。

```text
id
    → string

objectiveId
    → string

snake_case
    → camelCase

金額
    → integer / null

日時
    → API共通形式
```

---

### 31.62 判定不可Resource Test

判定結果が

```text
NOT_ASSESSABLE
```

の場合に、

```json
{
  "assessmentResult": "NOT_ASSESSABLE",
  "notAssessableReason": "..."
}
```

として正しく変換されることを確認する。

---

### 31.63 Resourceで再計算しないこと

Resource Testでは、`remainingAmount`などがResource内部で再計算されず、Result DTOの保存済み値をそのまま返却することを確認する。

---

### 31.64 副作用なしTest

ASM-004実行前後で、

```text
assessment_histories
objectives
month_end_asset_snapshots
month_end_asset_balances
month_end_holding_values
asset_account_available_settings
net_incomes
```

の業務データが変更されないことを確認する。

---

### 31.65 ASM-002とのIntegration Test

ASM-002で判定結果を保存した後、返却された

```text
assessmentHistoryId
```

を使用してASM-004を実行する。

ASM-002で保存した

```text
判定結果
+
計算根拠
```

と、ASM-004で取得した内容が一致することを確認する。

---

### 31.66 ASM-003とのIntegration Test

ASM-003で判定履歴一覧を取得し、一覧に含まれる

```text
assessmentHistoryId
```

をASM-004へ渡した場合に、対応する詳細を取得できることを確認する。

---

### 31.67 ASM-001とは履歴連携しない

ASM-001はプレビュー専用であり、判定履歴を保存しない。

そのため、ASM-001実行だけではASM-004で取得可能な新しい`assessmentHistoryId`が生成されないことを確認する。

---

### 31.68 実装上の責務分離

ASM-004では、最終的に以下の責務分離を維持する。

```text
Middleware
    → 利用者コンテキスト

Action
    → HTTP入力とUseCaseの橋渡し

UseCase
    → アプリケーション処理

Query
    → 判定履歴・関連情報取得

Result DTO
    → 取得結果表現

API Resource
    → API形式への変換

Responder
    → HTTPレスポンス生成
```

ASM-004では、

```text
保存済み判定履歴を
利用者境界を守って取得し、
判定時点の結果と計算根拠を
変更せず返却する
```

ことへLaravel実装の責務を限定する。

---

## 32. React・TypeScriptでの利用

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

### 32.1 TypeScript型

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

### 32.2 レスポンス型

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

### 32.3 AssessmentResultの共通型

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

### 32.4 NotAssessableReasonの共通型

判定不可理由についても、ASM-001・ASM-002と共通型を使用する。

概念例：

```ts
export type NotAssessableReason =
  | 'NET_INCOME_INSUFFICIENT'
  | 'ASSET_DATA_INSUFFICIENT';
```

実際のコード値は、目的達成判定の共通定義を正とする。

---

### 32.5 正常レスポンス型

API共通Envelopeを使用する場合は、以下のように定義する。

概念例：

```ts
export type GetAssessmentHistoryDetailResponse =
  ApiResponse<AssessmentHistoryDetail>;
```

---

### 32.6 IDはstringとして扱う

以下のIDは、React・TypeScript側で`string`として扱う。

```text
assessmentHistoryId
id
objectiveId
```

DB上で`bigint`であっても、フロントエンドで`number`へ変換しない。

---

### 32.7 金額はnumberとして扱う

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

### 32.8 assessedAtはstringとして受け取る

`assessedAt`は、APIから日時文字列として受け取る。

概念例：

```ts
assessedAt: string;
```

必要に応じて、表示時にDateへ変換する。

API Client層で独自の日時フォーマットへ変換しない。

---

### 32.9 API Client

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

### 32.10 userIdをAPI Client引数へ含めない

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

### 32.11 X-User-Id

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

### 32.12 Request Bodyを送信しない

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

### 32.13 Queryとして扱う

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

### 32.14 Query Key

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

### 32.15 assessmentHistoryIdをQuery Keyへ含める

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

### 32.16 利用者切替を考慮する

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

### 32.17 Query KeyへuserIdを含めてもよい

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

### 32.18 enabledによるQuery制御

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

### 32.19 ASM-003からの遷移

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

### 32.20 Route Parameter

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

### 32.21 詳細Page

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

### 32.22 Loading状態

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

### 32.23 Error状態

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

### 32.24 Empty Stateは使用しない

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

### 32.25 判定結果表示

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

### 32.26 判定結果による表示分岐

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

### 32.27 判定不可はError表示にしない

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

### 32.28 notAssessableReason

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

### 32.29 判定可能時のnotAssessableReason

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

### 32.30 金額表示

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

### 32.31 nullの金額を考慮する

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

### 32.32 保存済み計算根拠を表示する

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

### 32.33 remainingAmountを再計算しない

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

### 32.34 averageNetIncomeを再計算しない

React側で`net_incomes`を取得し、過去判定の

```text
averageNetIncome
```

を再計算しない。

ASM-004レスポンスの保存済み値を使用する。

---

### 32.35 現在の資産額を組み込まない

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

### 32.36 現在値との比較を行う場合

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

### 32.37 objectiveName

目的名は、

```text
assessmentHistory.objectiveName
```

をそのまま表示する。

目的名表示のためだけにOBJ-003を追加呼び出ししない。

---

### 32.38 objectiveId

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

### 32.39 目的の現在状態を履歴画面で推測しない

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

### 32.40 targetYearMonth

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

### 32.41 assessedAt

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

### 32.42 日時フォーマットを各Componentへ重複実装しない

`assessedAt`の表示変換は、共通Formatterへ集約する。

各Componentで個別に

```ts
new Date(...).toLocaleString(...)
```

を記述しない。

---

### 32.43 ASM-002保存直後との連携

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

### 32.44 ASM-002のレスポンスをそのまま永続表示に使わない

ASM-002成功時に判定結果が返却されても、判定履歴詳細画面ではURLの`assessmentHistoryId`を基準としてASM-004を取得する構成としてよい。

これにより、ページ再読み込み後も同じ履歴を取得できる。

---

### 32.45 ASM-003のCacheとの関係

ASM-003の一覧CacheとASM-004の詳細Cacheは分離する。

概念的には、

```text
assessmentHistories/list
assessmentHistories/detail/{id}
```

とする。

一覧データだけを使って詳細画面を完全に再構築しない。

---

### 32.46 一覧データをinitialDataとして利用してもよい

ASM-003の一覧レスポンスにASM-004と共通する項目が十分含まれている場合は、表示高速化のため一覧Cacheを`initialData`等に利用してもよい。

ただし、ASM-004固有の

```text
詳細な計算根拠
```

はASM-004で取得する。

---

### 32.47 ASM-004は詳細情報の正とする

判定履歴詳細画面では、ASM-003の一覧データよりASM-004の詳細レスポンスを正とする。

一覧表示用に省略・加工された値から詳細を推測しない。

---

### 32.48 GETのRetry

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

### 32.49 404を無意味にRetryしない

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

のような確定的な業務エラーについては、自動Retryを行わない。

Retry条件は、React共通のAPIエラー処理方針に従う。

---

### 32.50 INVALID_ASSESSMENT_HISTORY_ID

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

### 32.51 ASSESSMENT_HISTORY_NOT_FOUND

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

### 32.52 他利用者所属を表示しない

`ASSESSMENT_HISTORY_NOT_FOUND`を受信しても、

```text
他の利用者の判定履歴です。
```

などと表示しない。

API契約上、不存在との区別はできない。

---

### 32.53 USER_CONTEXT_REQUIRED

```text
USER_CONTEXT_REQUIRED
```

は、API共通の利用者コンテキストエラーとして扱う。

ASM-004専用の処理をPageへ重複実装しない。

---

### 32.54 INVALID_USER_ID

```text
INVALID_USER_ID
```

も、共通利用者コンテキストエラーとして扱う。

---

### 32.55 USER_NOT_FOUND

```text
USER_NOT_FOUND
```

の場合は、現在選択している利用者が有効ではない状態として共通処理する。

---

### 32.56 INTERNAL_SERVER_ERROR

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

### 32.57 error.codeで分岐する

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

### 32.58 判定結果とAPIエラーを混同しない

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

### 32.59 Componentの責務

判定履歴詳細Componentでは、主に以下を担当する。

- 目的名表示
- 判定対象年月表示
- 判定結果表示
- 判定実行日時表示
- 判定時点の計算根拠表示
- 判定不可理由表示

API通信や利用者境界判定は行わない。

---

### 32.60 CalculationBasis Component

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

### 32.61 Result Component

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

### 32.62 Query Hookの責務

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

### 32.63 API Clientの責務

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

### 32.64 概念的なディレクトリ構成

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

### 32.65 ASM系の共通型を再利用する

以下の概念は、ASM系APIで共通化する。

```text
AssessmentResult
NotAssessableReason
```

APIごとに同じUnion Typeを重複定義しない。

---

### 32.66 API Response型と画面用型

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

### 32.67 現在情報との混在に注意する

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

### 32.68 フロントエンドで行わないこと

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

## 33. 設計上の補足

### 33.1 ASM-004は「再判定API」ではない

ASM-004は、保存済みの判定履歴を参照するためのAPIである。

そのため、

```text
現在の資産状況
現在の手取り収入
現在の利用可能資産設定
```

を使用して判定結果を再計算しない。

概念的には、

```text
ASM-002
    ↓
判定実行
    ↓
assessment_historiesへ保存

ASM-004
    ↓
保存済み履歴を参照
```

とする。

---

### 33.2 保存済み判定結果を正とする

ASM-004では、

```text
assessment_histories
```

へ保存された判定結果および計算根拠を正とする。

現在の関連データと保存済み履歴の値に差異があっても、現在値で上書きしない。

---

### 33.3 過去履歴の再現性を重視する

判定履歴は、

```text
その時点で
どの条件を使って
どの判定結果になったか
```

を後から確認できることが重要である。

そのため、過去時点の再現に必要な情報は、可能な限りASM-002実行時に履歴として保存する。

---

### 33.4 現在テーブルへの依存を増やしすぎない

ASM-004のレスポンスを生成するために、現在の

```text
objectives
month_end_asset_snapshots
net_incomes
asset_account_available_settings
```

などへ過剰に依存しない。

過去時点の値を現在テーブルから再構築すると、後からデータが変更された際に履歴表示まで変化する可能性がある。

---

### 33.5 履歴へ保存すべき項目を意識する

過去判定の詳細表示に必要な項目については、ASM-002で`assessment_histories`へ保存することを優先する。

例えば、

```text
判定結果
利用可能資産額
必要額
平均手取り収入
必要生活防衛資金
残額
判定不可理由
```

などである。

---

### 33.6 objectiveNameの扱いに注意する

`objectiveName`を現在の

```text
objectives.name
```

から取得する場合、目的名変更後に過去履歴の表示名も変化する。

例えば、

```text
判定時
「一人暮らし」

    ↓

後日
目的名を
「大阪で一人暮らし」
へ変更
```

した場合、ASM-004で現在名を参照すると過去履歴にも新しい名称が表示される。

---

### 33.7 判定時点の目的名を固定したい場合

過去履歴の表示内容を判定時点の状態として厳密に固定する場合は、

```text
assessment_histories.objective_name
```

のように判定時点の目的名を保存する設計を検討する。

ASM-004で現在の`objectives.name`へ依存しなくて済む。

---

### 33.8 targetYearMonthの扱い

`targetYearMonth`も、関連する

```text
month_end_asset_snapshots.target_year_month
```

から取得できる。

ただし、履歴表示の完全な独立性を重視する場合は、判定時点の対象年月を履歴側へ保存してもよい。

実際の方針は、テーブル定義を正とする。

---

### 33.9 目的の現在のenabled状態に依存しない

判定履歴保存後に、対象目的が

```text
enabled = false
```

となっても、ASM-004の履歴参照には影響させない。

概念的には、

```text
目的無効化
    ↓
今後の新規判定対象外

過去判定履歴
    ↓
引き続き参照可能
```

とする。

---

### 33.10 目的無効化と履歴削除を連動させない

OBJ-005によって目的が無効化されても、

```text
assessment_histories
```

を削除しない。

過去判定履歴は、目的の現在状態とは別の履歴情報として保持する。

---

### 33.11 目的の論理削除にも注意する

将来的に目的の論理削除を行う場合も、過去判定履歴をどのように参照するかを別途考慮する。

過去履歴の表示に現在の`objectives`が必須だと、目的削除後に詳細表示できなくなる可能性がある。

そのため、履歴の自己完結性を意識した設計とする。

---

### 33.12 月末資産状況の現在のconfirmed状態に依存しない

判定履歴保存後に、

```text
confirmed = true
    ↓
confirmed = false
```

へ確定解除されても、ASM-004の履歴参照は可能とする。

現在の確定状態を取得条件へ含めない。

---

### 33.13 確定解除によって過去判定を消さない

SNP-005によって月末資産状況が確定解除されても、

```text
assessment_histories
```

を削除しない。

判定実行時点では確定済みだったという過去事実を保持する。

---

### 33.14 現在値と過去値を混在させない

ASM-004レスポンスでは、

```text
判定時点の値
```

と

```text
現在の値
```

を同じ意味の項目として混在させない。

例えば、

```text
availableAssetAmount
```

が判定時点の値なら、現在資産額で上書きしない。

---

### 33.15 現在値との比較は別責務とする

将来的に、

```text
過去判定時点
VS
現在
```

を比較する機能が必要になった場合は、

```text
ASM-004
+
現在状態取得API
```

を組み合わせる、または専用比較APIを検討する。

ASM-004へ現在値を安易に追加しない。

---

### 33.16 判定結果コードを共通化する

以下の判定結果コードは、ASM系API全体で共通化する。

```text
ACHIEVABLE
NOT_ACHIEVABLE
NOT_ASSESSABLE
```

ASM-004だけ異なるコード値を定義しない。

---

### 33.17 判定不可理由も共通化する

`notAssessableReason`についても、ASM-001、ASM-002、ASM-004で共通のコード体系を使用する。

Laravelでは共通EnumやValue Object、Reactでは共通Union Typeとして管理してよい。

---

### 33.18 判定不可をAPIエラーとしない

```text
NOT_ASSESSABLE
```

は、判定処理の結果であり、ASM-004の通信エラーではない。

そのため、

```http
200 OK
```

で返却し、`assessmentResult`として表現する。

---

### 33.19 判定不可理由とAPIエラーコードを混同しない

例えば、

```text
NET_INCOME_INSUFFICIENT
```

が判定不可理由である場合、

```text
error.code
```

には使用しない。

概念的には、

```json
{
  "data": {
    "assessmentResult": "NOT_ASSESSABLE",
    "notAssessableReason": "NET_INCOME_INSUFFICIENT"
  }
}
```

とする。

---

### 33.20 remainingAmountをASM-004で再計算しない

`remainingAmount`がASM-002で保存される設計の場合、ASM-004では保存済み値を返却する。

以下のような計算をASM-004で再実行しない。

```text
availableAssetAmount
-
requiredAmount
-
requiredEmergencyFund
```

---

### 33.21 API Resourceでも再計算しない

API Resourceは表現変換だけを担当する。

以下を行わない。

```text
計算
判定
補完
DB検索
```

`remainingAmount`等も、Result DTOに含まれる値をそのまま返却する。

---

### 33.22 React側でも再計算しない

React側でも、

```text
availableAssetAmount
- requiredAmount
- requiredEmergencyFund
```

によって過去判定の残額を再計算しない。

ASM-004レスポンスの

```text
remainingAmount
```

を表示する。

---

### 33.23 過去履歴の説明可能性を確保する

ASM-004は、

```text
判定結果だけ見ても
なぜそうなったのかわからない
```

状態を避けるためのAPIでもある。

そのため、判定履歴保存時には最低限、

```text
判定結果
+
主要な計算根拠
```

を保存する。

---

### 33.24 assessment_historiesを単なる結果テーブルにしない

`assessment_histories`へ

```text
assessment_result
```

だけを保存すると、過去の判定理由を後から説明できない。

ASM-004の要件を満たすため、計算根拠も履歴として保持する設計とする。

---

### 33.25 ASM-002との整合性を最優先する

ASM-004で返却する情報は、ASM-002で保存した情報から再現できる必要がある。

そのため、

```text
ASM-004で項目追加
```

だけを行わず、

```text
ASM-002
assessment_histories
ASM-004
```

をセットで見直す。

---

### 33.26 ASM-001との責務を混在させない

ASM-001は、現在状態による判定プレビューである。

ASM-004は、保存済み履歴詳細である。

概念的には、

```text
ASM-001
    → 今どうなるか

ASM-004
    → あの時どう判定されたか
```

という違いを維持する。

---

### 33.27 ASM-003と詳細責務を分離する

ASM-003では、一覧表示に必要な概要情報だけを返却する。

ASM-004では、詳細な計算根拠まで返却する。

一覧APIへ詳細情報をすべて詰め込まない。

---

### 33.28 一覧レスポンスだけで詳細を再構築しない

React側で、ASM-003の一覧データだけを使って詳細画面を完全に構築しない。

ASM-004を判定履歴詳細の正とする。

---

### 33.29 assessmentHistoryIdをトップレベルリソースとして扱う

ASM-004は、

```http
GET /api/v1/assessment-histories/{assessmentHistoryId}
```

とする。

判定履歴は、保存後は独立した参照対象リソースとして扱う。

---

### 33.30 objectiveIdをURLへ重複指定しない

以下のようなURLにはしない。

```http
GET /api/v1/objectives/{objectiveId}/assessment-histories/{assessmentHistoryId}
```

`assessmentHistoryId`から目的を一意に特定できるためである。

---

### 33.31 snapshotIdもURLへ重複指定しない

同様に、

```http
GET /api/v1/month-end-asset-snapshots/{snapshotId}/assessment-histories/{assessmentHistoryId}
```

とはしない。

判定履歴から対象Snapshotを取得できるためである。

---

### 33.32 利用者境界をQuery段階で保証する

以下の方式を基本としない。

```text
assessmentHistoryIdで取得
    ↓
取得後にuserId確認
```

取得Queryの条件に利用者境界を含める。

---

### 33.33 他利用者所属を404へ集約する

以下は同じ外部レスポンスとする。

```text
履歴不存在
        ↓
ASSESSMENT_HISTORY_NOT_FOUND

他利用者所属
        ↓
ASSESSMENT_HISTORY_NOT_FOUND
```

他利用者の履歴存在を推測できる情報を返さない。

---

### 33.34 403 Forbiddenを基本的に使用しない

他利用者に属する判定履歴を指定された場合でも、ASM-004では

```http
403 Forbidden
```

ではなく、

```http
404 Not Found
```

を使用する。

利用者境界外のリソース存在を外部へ公開しないためである。

---

### 33.35 GETへ副作用を持たせない

ASM-004実行時に以下を行わない。

- 判定履歴更新
- 判定履歴補完
- 判定再実行
- 関連目的更新
- 月末資産状況更新
- 参照したことを業務テーブルへ保存

GETとして副作用を持たせない。

---

### 33.36 不足する判定根拠を現在値から補完しない

例えば、本来保存されるべき

```text
averageNetIncome
```

が欠落していても、現在の`net_incomes`から再計算して返却しない。

仕様上あり得ない欠落であれば、内部データ不整合として扱う。

---

### 33.37 データ不整合を自動修復しない

ASM-004では、以下を行わない。

```text
不足データのINSERT
不正FKの修正
判定結果のUPDATE
履歴のDELETE
```

参照APIとして問題を隠蔽しない。

---

### 33.38 DB制約で参照整合性を保証する

少なくとも、

```text
assessment_histories.objective_id
    → objectives.id

assessment_histories.month_end_asset_snapshot_id
    → month_end_asset_snapshots.id
```

などの関係は、可能な限り外部キー制約で保証する。

ASM-004のアプリケーションコードだけに整合性保証を依存させない。

---

### 33.39 QueryとRepositoryを分離する

ASM-004では読み取りだけを行うため、Queryを使用する。

```text
Query
    → SELECT

Repository
    → INSERT / UPDATE / DELETE
```

という共通設計に従う。

---

### 33.40 Repositoryを形式的に作らない

ASM-004のためだけに、

```text
AssessmentHistoryRepository
```

を作成し、中身がSELECTだけになる構成には原則としてしない。

読み取り責務はQueryへ集約する。

---

### 33.41 Actionを薄く保つ

Actionでは、

```text
assessmentHistoryId
+
UserContext
    ↓
UseCase
    ↓
Responder
```

だけを担当する。

判定履歴検索や計算根拠処理をActionへ持たせない。

---

### 33.42 UseCaseで再判定しない

UseCaseだからといって、ASM-001・ASM-002で使う判定Serviceを再利用して判定し直さない。

ASM-004のUseCaseは、

```text
保存済み履歴を
取得・組み立てる
```

ことを担当する。

---

### 33.43 FormRequestを無理に作らない

ASM-004では、

```text
Request Bodyなし
Query Parameterなし
```

である。

そのため、空FormRequestを統一感だけのために作成しない。

---

### 33.44 Result DTOを使用する

Eloquent ModelをそのままResponderへ渡さない。

概念的には、

```text
Query結果
    ↓
Result DTO
    ↓
API Resource
    ↓
Response
```

とする。

---

### 33.45 API ResourceでDB構造を隠蔽する

例えば、

```text
assessment_result
available_asset_amount
assessed_at
```

を、

```text
assessmentResult
availableAssetAmount
assessedAt
```

へ変換する。

DB設計とAPI契約を分離する。

---

### 33.46 bigint IDをstringで返却する

以下のIDは、APIレスポンスではstringとして扱う。

```text
id
objectiveId
```

必要に応じて`monthEndAssetSnapshotId`を返す場合もstringとする。

---

### 33.47 金額を表示用文字列にしない

APIでは、

```json
{
  "availableAssetAmount": 2500000
}
```

として返却する。

以下のようにしない。

```json
{
  "availableAssetAmount": "2,500,000円"
}
```

表示はReact側で行う。

---

### 33.48 nullと0を区別する

判定根拠の金額項目について、

```text
null
```

と

```text
0
```

を別の状態として扱う。

```text
null
    → 算出していない・値が存在しない

0
    → 0円という算出済み値
```

とする。

---

### 33.49 現在の目的状態を無理に返却しない

ASM-004は過去履歴詳細APIであるため、

```text
enabled
```

などの現在状態をレスポンスへ安易に追加しない。

現在状態が必要ならOBJ系APIを利用する。

---

### 33.50 現在のconfirmedを無理に返却しない

同様に、

```text
confirmed
```

は判定時点の計算根拠ではない。

現在のSnapshot状態が必要ならSNP系APIを利用する。

---

### 33.51 キャッシュ導入時は履歴の不変性を確認する

判定履歴自体が不変でも、

```text
objectiveName
```

を現在の`objectives`から取得している場合は、ASM-004レスポンス全体は不変とは限らない。

長期キャッシュを導入する場合は注意する。

---

### 33.52 利用者ごとにキャッシュを分離する

キャッシュを導入する場合は、

```text
assessmentHistoryId
```

だけをキャッシュキーにしない。

少なくとも、

```text
userId
+
assessmentHistoryId
```

で分離する。

---

### 33.53 ReactのQuery Keyでも利用者境界を考慮する

Phase1では利用者切替が存在するため、TanStack QueryのCacheで前利用者の履歴を誤表示しないようにする。

利用者切替時のinvalidate、またはQuery Keyへの`userId`追加を行う。

---

### 33.54 ASM-004はQueryとして扱う

React側では、ASM-004をMutationとして扱わない。

```text
GET
+
副作用なし
```

であるため、TanStack QueryのQueryを使用する。

---

### 33.55 GETのRetryは許容できる

ASM-004は冪等であるため、一時的な通信エラーについては限定的なRetryを許可してよい。

ただし、

```text
ASSESSMENT_HISTORY_NOT_FOUND
```

などの確定的エラーを無意味にRetryしない。

---

### 33.56 判定不可と通信エラーをUIで区別する

React側では、

```text
NOT_ASSESSABLE
```

と

```text
API Error
```

を明確に区別する。

```text
NOT_ASSESSABLE
    → 正常な判定結果表示

API Error
    → エラー表示
```

とする。

---

### 33.57 ASM-004だけで現在状態を推測しない

判定履歴詳細から、

```text
現在も目的が有効
現在も資産額が同じ
現在も確定済み
```

などを推測しない。

ASM-004が表すのは、原則として判定履歴および判定時点の情報である。

---

### 33.58 Phase1では履歴編集APIを設けない

判定履歴は、判定結果の監査・参照用途として扱うため、Phase1では

```http
PATCH /assessment-histories/{id}
```

のような履歴更新APIを設けない。

---

### 33.59 Phase1では履歴削除APIも設けない

同様に、判定履歴を利用者が任意に削除するAPIもPhase1では設けない。

判定履歴は過去の判定記録として保持する。

---

### 33.60 判定履歴の物理削除を通常運用にしない

判定履歴は時系列で残すことに価値がある。

そのため、通常操作で古い履歴を物理削除する設計にはしない。

保存期間要件が将来発生した場合は別途検討する。

---

### 33.61 監査ログと判定履歴を混同しない

`assessment_histories`は、

```text
目的達成判定という
業務上の結果履歴
```

である。

一方、アクセスログや操作監査ログとは目的が異なる。

判定履歴を技術監査ログの代替として扱わない。

---

### 33.62 履歴テーブルを現在状態として使わない

`assessment_histories`は過去判定の記録であり、現在の目的達成可否を常に示すテーブルではない。

現在の条件で判定したい場合は、

```text
ASM-001
または
ASM-002
```

を使用する。

---

### 33.63 最新履歴を現在判定と決めつけない

最新の`assessment_histories`レコードであっても、その後に資産・収入等が変更されている可能性がある。

そのため、

```text
最新履歴
=
現在の判定結果
```

とは限らない。

---

### 33.64 再判定は新規履歴として保存する

同じ目的について新しい条件で再判定する場合は、既存履歴を更新せず、

```text
新しいassessment_histories
```

として保存する。

これにより、時系列変化を追跡できる。

---

### 33.65 Phase1では詳細取得に責務を限定する

ASM-004では、以下を対象外とする。

- 目的達成判定の新規実行
- 判定プレビュー
- 判定履歴保存
- 判定履歴更新
- 判定履歴削除
- 判定結果再計算
- 現在値との比較
- 現在資産情報取得
- 現在手取り収入取得
- 判定根拠の自動補完
- 履歴データの自動修復
- 悲観ロック
- 楽観ロック
- 書き込みトランザクション

Phase1では、

```text
操作対象利用者が
指定した判定履歴について、
保存された判定結果と
判定時点の計算根拠を
安全かつ副作用なく取得する
```

ことへ責務を限定する。

---

## 34. 関連ドキュメント

- [API一覧](../../api-list.md)
- [API共通方針](../../api-common-policy.md)
- [ASM-001 目的達成判定プレビュー](./asm-001-preview.md)
- [ASM-002 目的達成判定結果保存](./asm-002-create.md)
- [ASM-003 判定履歴一覧取得](./asm-003-list.md)
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