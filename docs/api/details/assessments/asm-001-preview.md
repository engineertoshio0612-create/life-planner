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

### 2 ユースケース

利用者は、
登録済みの目的について、
現在の資産状況および
手取り収入などをもとに、
目的を達成できる見込みがあるかを確認する。

例えば、
以下のような場合に使用する。

- 目標金額まで資産を増やせる見込みがあるか確認する
- 目標期限までに必要金額へ到達できるか確認する
- 最新の確定済み月末資産状況をもとに再判定する
- 手取り収入や資産状況が変化した後に再判定する
- 過去の判定結果とは別に、新しい判定履歴を残す
- 判定材料が不足している場合に、判定不可として履歴を残す

同じ目的について
複数回判定を実行した場合は、
判定のたびに
新しい目的達成判定履歴を作成する。

過去の目的達成判定履歴を
上書きしない。

---

### 3 エンドポイント

```http
POST /api/v1/objectives/{objectiveId}/assessments
```

---

### 4 HTTPメソッド

```http
POST
```

本APIは、
指定された目的に対して
新しい目的達成判定を実行し、
その結果を
目的達成判定履歴として新規登録する。

同じ目的に対して
同一条件で再実行した場合でも、
新しい判定履歴を作成する。

そのため、
既存の目的達成判定履歴を
更新するAPIとしては扱わない。

---

### 5 認証・利用者の扱い

Phase1では、
認証機能を実装しない。

操作対象となる利用者は、
`X-User-Id`
リクエストヘッダーで指定する。

```http
X-User-Id: 1
```

指定された利用者に属する
目的についてのみ、
目的達成判定を実行できる。

他の利用者に属する目的について、
目的達成判定を実行することはできない。

目的を取得する際は、
必ず以下を検索条件に含める。

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
```

指定された`objectiveId`が
他の利用者に属する場合は、
対象となる目的が
存在しないものとして扱う。

目的達成判定に使用する
月末資産状況についても、
操作対象利用者に属するデータのみを
参照する。

```text
month_end_asset_snapshots.user_id
    = 操作対象利用者ID
```

月末資産残高は、
対象となる月末資産状況および
資産口座を経由して
利用者境界を保証する。

```text
month_end_asset_balances.month_end_asset_snapshot_id
    = month_end_asset_snapshots.id

AND

month_end_asset_snapshots.user_id
    = 操作対象利用者ID
```

商品別月末評価額についても、
月末資産状況および
保有商品の所属資産口座を経由して
利用者境界を保証する。

```text
month_end_holding_values.month_end_asset_snapshot_id
    = month_end_asset_snapshots.id

AND

month_end_asset_snapshots.user_id
    = 操作対象利用者ID
```

または、

```text
month_end_holding_values.holding_asset_id
    = holding_assets.id

AND

holding_assets.asset_account_id
    = asset_accounts.id

AND

asset_accounts.user_id
    = 操作対象利用者ID
```

手取り収入についても、
操作対象利用者に属するデータのみを
判定材料として使用する。

```text
net_incomes.user_id
    = 操作対象利用者ID
```

新しく作成する
目的達成判定履歴は、
指定された目的に紐づけて保存する。

```text
assessment_histories.objective_id
    = objectiveId
```

目的達成判定履歴自体に
`user_id`を保持しない場合は、

```text
assessment_histories
    ↓
objectives
    ↓
users
```

の関連によって
利用者境界を保証する。

利用者IDは、
リクエストボディ、
クエリパラメータまたは
パスパラメータでは受け付けない。

利用者IDは、
ミドルウェアで設定された
利用者コンテキストから取得する。

`X-User-Id`が指定されていない場合、
形式が不正な場合、
または指定された利用者が存在しない場合は、
API共通方針に従って
エラーを返却する。

本APIでは、
他の利用者に属する

- 目的
- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 手取り収入
- 目的達成判定履歴

の存在を
レスポンスから推測できないようにする。

---

### 6 パスパラメータ

| パラメータ名 | 型 | 必須 | 説明 |
|---|---|:---:|---|
| `objectiveId` | string | ○ | 目的達成判定を実行する目的ID |

リクエスト例：

```http
POST /api/v1/objectives/5/assessments
```

`objectiveId`は、
目的達成判定の対象となる
目的を一意に識別するIDである。

目的達成判定履歴IDは、
本APIでは指定しない。

判定実行によって、
新しい目的達成判定履歴を作成する。

---

### 7 クエリパラメータ

なし。

本APIでは、
クエリパラメータを使用しない。

判定に使用する
月末資産状況や
手取り収入などを、
クエリパラメータで指定しない。

判定対象となるデータは、
操作対象利用者および
目的の情報をもとに
サーバー側で決定する。

---

### 8 リクエストヘッダー

#### 8.1 必須ヘッダー

| ヘッダー名 | 必須 | 説明 |
|---|:---:|---|
| `X-User-Id` | ○ | 操作対象となる利用者ID |
| `Accept` | ○ | `application/json`を指定する |

リクエスト例：

```http
POST /api/v1/objectives/5/assessments
Accept: application/json
X-User-Id: 1
```

本APIでは、
リクエストボディを使用しないため、
`Content-Type`は必須としない。

---

### 9 リクエストボディ

なし。

本APIでは、
リクエストボディを使用しない。

目的達成判定に必要な情報は、
以下からサーバー側で取得する。

- `objectiveId`で指定された目的
- 操作対象利用者の確定済み月末資産状況
- 判定対象となる月末資産残高
- 判定対象となる商品別月末評価額
- 操作対象利用者の手取り収入

判定に使用する値を
クライアントから直接指定させない。

例えば、
以下のような項目は
リクエストボディから受け付けない。

- `userId`
- `targetAmount`
- `targetDate`
- `currentAssets`
- `netIncome`
- `assessmentResult`

これにより、
登録済みの業務データをもとに
一貫した条件で
目的達成判定を実行する。

---

### 10 リクエスト項目

なし。

本APIでは、
リクエストボディに
業務項目を持たない。

判定対象となる目的は、
パスパラメータの
`objectiveId`から特定する。

操作対象利用者は、
`X-User-Id`から取得する。

その他の判定材料は、
サーバー側で
登録済みデータから取得する。

---

### 11 バリデーション

#### 11.1 objectiveId

`objectiveId`は、
必須のパスパラメータとする。

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 正の整数として扱えること

正常例：

```text
1
5
123
```

不正例：

```text
0
-1
abc
1.5
```

`objectiveId`の形式が不正な場合は、
`VALIDATION_ERROR`
として扱う。

---

#### 11.2 目的の存在確認

指定された`objectiveId`について、
以下の条件を満たす
目的が存在することを確認する。

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
```

対象となる目的が
存在しない場合は、
`OBJECTIVE_NOT_FOUND`
として扱う。

他の利用者に属する
目的IDが指定された場合も、
同じエラーとして扱う。

これにより、
他の利用者に属する目的の存在を
レスポンスから判別できないようにする。

---

#### 11.3 目的の判定対象状態

指定された目的が、
目的達成判定を実行可能な状態であることを
確認する。

論理削除されている目的は、
判定対象としない。

目的が無効化されている場合の
判定可否については、
目的管理APIおよび
機能要件で定義した
目的の状態管理ルールに従う。

判定対象外となる目的について、
目的達成判定を実行しない。

---

#### 11.4 最新の確定済み月末資産状況

目的達成判定では、
操作対象利用者に属する
確定済み月末資産状況を
判定材料として使用する。

対象となる月末資産状況は、
少なくとも以下の条件を満たすものとする。

```text
user_id = 操作対象利用者ID
AND
confirmed = true
```

複数存在する場合は、
`target_year_month`が
最新の月末資産状況を使用する。

```text
confirmed = true
    ↓
target_year_month DESC
    ↓
先頭1件
```

未確定の月末資産状況は、
目的達成判定に使用しない。

---

#### 11.5 確定済み月末資産状況が存在しない場合

操作対象利用者について、
確定済みの月末資産状況が
存在しない場合は、
目的達成可否を
計算できない状態として扱う。

この場合、
APIエラーとはせず、
判定結果を
「判定不可」とする。

```text
確定済み月末資産状況あり
    → 判定処理を継続

確定済み月末資産状況なし
    → 判定不可
```

判定不可となった場合も、
目的達成判定履歴を作成する。

---

#### 11.6 月末資産データ

最新の確定済み月末資産状況に紐づく
月末資産データを取得する。

資産口座の
残高記録単位に応じて、
以下を判定材料として使用する。

```text
口座単位
    ↓
month_end_asset_balances.balance

商品単位
    ↓
month_end_holding_values.value
```

月末資産状況が
確定済みであることを前提とするため、
SNP-004 月末資産状況確定APIで
保証されたデータを使用する。

目的達成判定実行時に、
月末資産残高や
商品別月末評価額の
入力完了条件を
改めてクライアントへ要求しない。

---

#### 11.7 手取り収入

目的達成判定に必要な
手取り収入は、
操作対象利用者に属する
`net_incomes`から取得する。

```text
net_incomes.user_id
    = 操作対象利用者ID
```

Phase1では、
目的達成判定に使用する
手取り収入として、
直近3ヶ月分の手取り収入をもとに
平均手取り収入を算出する。

判定に必要な件数の
手取り収入が存在する場合は、
その平均値を
判定材料として使用する。

---

#### 11.8 手取り収入が不足している場合

目的達成判定に必要な
手取り収入データが不足している場合は、
入力値の不正とは扱わない。

この場合は、
目的達成可否を
計算できない状態として扱い、
判定結果を
「判定不可」とする。

```text
判定に必要な手取り収入あり
    → 判定処理を継続

判定に必要な手取り収入なし
または不足
    → 判定不可
```

判定不可となった場合も、
目的達成判定履歴を作成する。

---

#### 11.9 目的の判定材料

目的達成判定に必要となる
目的側の情報が
登録されていることを確認する。

具体的な判定項目は、
目的エンティティおよび
機能要件で定義された
目的達成判定ルールに従う。

判定に必要な目的情報が不足している場合は、
リクエスト形式の不正とは扱わず、
判定結果を
「判定不可」とする。

判定不可となった場合も、
目的達成判定履歴を作成する。

---

#### 11.10 判定不可とAPIエラーの区別

目的達成判定では、
「リクエスト自体を処理できない状態」と
「判定材料が不足しているため判定できない状態」を
区別する。

```text
リクエスト・対象リソースに問題あり
    ↓
APIエラー

判定対象は正しいが
判定材料が不足
    ↓
判定不可
    ↓
目的達成判定履歴を保存
```

APIエラーとなる代表例は、
以下とする。

- `X-User-Id`が未指定
- `X-User-Id`の形式が不正
- 操作対象利用者が存在しない
- `objectiveId`の形式が不正
- 指定された目的が存在しない
- 指定された目的が他の利用者に属している

判定不可となる代表例は、
以下とする。

- 確定済み月末資産状況が存在しない
- 判定に必要な手取り収入が不足している
- 目的達成判定に必要な業務データが不足している

判定不可は、
`4xx`系エラーとして扱わない。

---

#### 11.11 X-User-Id

以下を検証する。

- 指定されていること
- API共通方針で定めたID形式であること
- 指定された利用者が存在すること
- 指定された利用者が論理削除されていないこと

`X-User-Id`が指定されていない場合は、
`USER_CONTEXT_REQUIRED`
として扱う。

形式が不正な場合は、
`INVALID_USER_ID`
として扱う。

指定された利用者が存在しない場合、
または論理削除されている場合は、
`USER_NOT_FOUND`
として扱う。

---

#### 11.12 業務状態に依存する検証

以下は、
単項目バリデーションではなく、
目的達成判定の
業務ルールとして検証する。

- 目的が操作対象利用者に属していること
- 目的が判定対象となる状態であること
- 最新の確定済み月末資産状況を使用すること
- 未確定の月末資産状況を判定に使用しないこと
- 月末資産状況に紐づく資産データのみを使用すること
- 操作対象利用者の手取り収入のみを使用すること
- 判定に必要な情報が不足している場合は判定不可とすること

入力形式の不正による
`VALIDATION_ERROR`と、
目的達成判定の結果としての
「判定不可」は
明確に区別して扱う。

---

### 12 業務ルール

#### 12.1 判定対象となる目的

目的達成判定は、
`objectiveId`で指定された
目的に対して実行する。

対象となる目的は、
必ず操作対象利用者に
属していなければならない。

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
```

他の利用者に属する目的を
判定対象としてはならない。

---

#### 12.2 判定時点のデータを使用する

目的達成判定では、
API実行時点で登録されている
最新の業務データを使用する。

クライアントから、
判定用の資産額や
手取り収入などを
直接指定させない。

判定に必要な情報は、
サーバー側で取得する。

---

#### 12.3 月末資産状況の扱い

目的達成判定では、
操作対象利用者に属する
確定済み月末資産状況のみを
使用する。

```text
user_id = 操作対象利用者ID
AND
confirmed = true
```

未確定の月末資産状況は、
判定材料として使用しない。

確定済み月末資産状況が
複数存在する場合は、
`target_year_month`が
最も新しいものを使用する。

```text
confirmed = true
    ↓
target_year_month DESC
    ↓
先頭1件
```

---

#### 12.4 総資産額の算出

目的達成判定に使用する
総資産額は、
最新の確定済み月末資産状況に紐づく
資産データから算出する。

資産口座の
残高記録単位に応じて、
使用する金額を切り替える。

```text
口座単位
    ↓
month_end_asset_balances.balance

商品単位
    ↓
month_end_holding_values.value
```

口座単位の資産口座について、
商品別月末評価額を
総資産額へ加算してはならない。

商品単位の資産口座について、
月末資産残高を
重複して加算してはならない。

総資産額は、
判定対象となる各資産の金額を
合計して算出する。

---

#### 12.5 確定済みデータを信頼する

月末資産状況が
確定済みである場合、
SNP-004 月末資産状況確定APIによって
必要な月末資産残高および
商品別月末評価額が
登録済みであることを前提とする。

ASM-001では、
月末資産状況確定時と
同じ入力完了チェックを
重複して実行しない。

ただし、
データ不整合などによって
判定に必要な資産データを
取得できない場合は、
目的達成判定を
正常な「達成可能」または
「達成困難」として扱わない。

---

#### 12.6 手取り収入の扱い

目的達成判定では、
操作対象利用者に属する
手取り収入のみを使用する。

```text
net_incomes.user_id
    = 操作対象利用者ID
```

Phase1では、
直近3ヶ月分の
手取り収入を使用し、
平均手取り収入を算出する。

概念的には、
以下のように算出する。

```text
直近3ヶ月の手取り収入合計
÷
3
=
平均手取り収入
```

他の利用者の手取り収入を
判定材料へ含めてはならない。

---

#### 12.7 直近3ヶ月の扱い

手取り収入は、
対象年月が新しい順に取得し、
直近3ヶ月分を使用する。

概念的には、
以下とする。

```text
user_id = 操作対象利用者ID
    ↓
target_year_month DESC
    ↓
3件取得
```

3ヶ月分の手取り収入が
揃っていない場合は、
判定に必要な情報が不足しているものとして
「判定不可」とする。

1ヶ月または2ヶ月分だけを使用して
平均手取り収入を算出しない。

---

#### 12.8 目的情報の扱い

目的達成判定では、
指定された目的に登録されている
判定対象情報を使用する。

目的側の具体的な判定項目および
計算式については、
機能要件で定義された
目的達成判定ルールに従う。

判定実行時に、
目的の内容を変更しない。

また、
クライアントから送信された値によって
目的の条件を
一時的に上書きして判定しない。

---

#### 12.9 判定結果

目的達成判定の結果は、
業務上定義された
判定結果として扱う。

Phase1では、
少なくとも以下の状態を区別する。

```text
達成可能
達成困難
判定不可
```

具体的な物理値、
Enum値および
APIレスポンス値については、
テーブル定義書および
API共通定義に従う。

---

#### 12.10 判定不可

判定対象となる目的は存在するが、
目的達成可否を判定するための
情報が不足している場合は、
APIエラーではなく
「判定不可」とする。

代表的な条件は、
以下とする。

- 確定済み月末資産状況が存在しない
- 直近3ヶ月分の手取り収入が揃っていない
- 目的達成判定に必要な目的情報が不足している
- 判定に必要な資産情報を取得できない

判定不可の場合も、
ASM-001自体は
正常に実行されたものとして扱う。

---

#### 12.11 判定不可の場合も履歴を保存する

判定結果が
「判定不可」であっても、
目的達成判定を実行した事実として
`assessment_histories`
へ履歴を保存する。

これにより、

```text
いつ
どの目的について
どのような結果になったか
```

を後から確認できるようにする。

判定不可であることを理由に、
履歴登録を省略しない。

---

#### 12.12 判定履歴は新規登録する

ASM-001を実行するたびに、
新しい目的達成判定履歴を
作成する。

```text
1回目の判定
    ↓
assessment_histories レコードA

2回目の判定
    ↓
assessment_histories レコードB
```

過去の履歴を
更新または上書きしない。

同一の目的について
同じ判定結果となった場合も、
新しい履歴として保存する。

---

#### 12.13 判定時点の結果を保存する

目的達成判定履歴には、
判定実行時点の結果を保存する。

その後、

- 目的が更新された
- 月末資産状況が追加された
- 月末資産残高が変更された
- 商品別月末評価額が変更された
- 手取り収入が追加・更新された

場合でも、
過去の目的達成判定履歴を
自動的に再計算または更新しない。

過去の履歴は、
判定実行時点の結果として保持する。

---

#### 12.14 再判定

最新の資産状況や
手取り収入をもとに
再判定したい場合は、
ASM-001を再度実行する。

```text
過去の判定履歴
    ↓
変更しない

最新データでASM-001実行
    ↓
新しい判定履歴を作成
```

これにより、
過去の判定結果と
最新の判定結果を
比較できる。

---

#### 12.15 判定処理による他データの変更禁止

ASM-001では、
目的達成判定履歴の新規登録以外の
業務データを変更しない。

以下を変更してはならない。

- `objectives`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `holding_assets`
- `net_incomes`

目的達成判定は、
既存データを参照して
判定結果を生成する処理とする。

---

### 13 判定処理

目的達成判定は、
概念的に以下の順序で実行する。

```text
操作対象利用者の確認
    ↓
目的の取得
    ↓
目的の利用者境界確認
    ↓
最新の確定済み月末資産状況を取得
    ↓
判定対象となる資産額を取得・集計
    ↓
直近3ヶ月の手取り収入を取得
    ↓
平均手取り収入を算出
    ↓
判定に必要な情報が揃っているか確認
    ↓
不足あり
    → 判定不可

不足なし
    → 目的達成判定を実行
    ↓
判定結果を生成
    ↓
目的達成判定履歴を新規登録
    ↓
登録した判定結果を返却
```

判定途中で
APIエラーとなる例外が発生した場合は、
目的達成判定履歴を
作成しない。

---

### 14 判定不可の処理

判定材料が不足している場合は、
処理を異常終了させず、
判定不可として
目的達成判定履歴を作成する。

概念的には、
以下とする。

```text
判定材料確認
    ↓
不足あり
    ↓
result = 判定不可
    ↓
assessment_historiesへ登録
    ↓
正常レスポンス
```

判定不可となった理由について、
`assessment_histories`に
理由を保持する設計となっている場合は、
判定時点の理由も保存する。

具体的な保存項目は、
テーブル定義書に従う。

---

### 15 判定結果保存

目的達成判定の結果は、
`assessment_histories`
へ保存する。

目的達成判定履歴は、
指定された目的へ紐づける。

```text
assessment_histories.objective_id
    = objectiveId
```

保存する項目は、
`assessment_histories`の
テーブル定義に従う。

少なくとも、
判定結果を
判定実行時点の履歴として保持する。

履歴登録後に
元となった目的や資産情報が変更されても、
保存済み履歴を変更しない。

---

### 16 レスポンス

目的達成判定および
目的達成判定履歴の登録に成功した場合は、
作成した目的達成判定履歴を返却する。

判定結果が
「判定不可」の場合も、
APIとしては正常終了とする。

成功時のHTTPステータスは、
新しい目的達成判定履歴を
作成するため、

```http
201 Created
```

とする。

レスポンスは、
API共通方針で定めた
Envelope形式を使用する。

例：

```json
{
  "data": {
    "id": "15",
    "objectiveId": "5",
    "result": "achievable"
  }
}
```

実際の`result`の値および
その他の返却項目は、
目的達成判定結果の定義および
`assessment_histories`の
テーブル定義に従う。

---

### 17 レスポンス項目

目的達成判定実行成功時は、
作成した目的達成判定履歴の情報を返却する。

| 項目 | 型 | NULL | 説明 |
|---|---|:---:|---|
| `id` | string | × | 作成された目的達成判定履歴ID |
| `objectiveId` | string | × | 判定対象となった目的ID |
| `result` | string | × | 目的達成判定結果 |

IDは、
API共通方針に従って
stringとして返却する。

```json
{
  "data": {
    "id": "15",
    "objectiveId": "5",
    "result": "achievable"
  }
}
```

判定不可の場合も、
同じレスポンス構造を使用する。

概念例：

```json
{
  "data": {
    "id": "16",
    "objectiveId": "5",
    "result": "unassessable"
  }
}
```

判定不可理由を
レスポンス項目として返却する場合は、
`assessment_histories`の
保存項目および
目的達成判定結果の共通定義に合わせて
別途定義する。

本APIでは、
判定処理に使用した
内部データをそのまま返却しない。

例えば、
以下の情報を
レスポンスへ無条件に含めない。

- `user_id`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `holding_assets`
- `net_incomes`
- `created_at`
- `updated_at`

判定結果の説明に必要な
スナップショット値を
`assessment_histories`へ保存している場合は、
テーブル定義および
APIレスポンス仕様に従って
必要な項目のみを返却する。

---

### 18 エラーレスポンス

異常時は、
API共通方針で定めた
共通エラーレスポンス形式を使用する。

本APIでは、
利用者コンテキストの不正、
`objectiveId`の不正、
目的の不存在、
目的が判定対象外である場合、
および
想定外のサーバーエラーを扱う。

一方、
判定材料が不足している場合は、
APIエラーとはせず、
判定結果を
「判定不可」として正常終了する。

APIエラーが発生した場合は、
目的達成判定履歴を登録しない。

---

#### 18.1 利用者コンテキストが指定されていない場合

`X-User-Id`が
指定されていない場合は、
`USER_CONTEXT_REQUIRED`
を返却する。

```json
{
  "error": {
    "code": "USER_CONTEXT_REQUIRED",
    "message": "操作対象の利用者を指定してください。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

目的達成判定は実行せず、
目的達成判定履歴も登録しない。

---

#### 18.2 利用者ID形式が不正な場合

`X-User-Id`が
API共通方針で定めたID形式に
一致しない場合は、
`INVALID_USER_ID`
を返却する。

```json
{
  "error": {
    "code": "INVALID_USER_ID",
    "message": "利用者IDの形式が不正です。",
    "details": [
      {
        "field": "X-User-Id",
        "reason": "invalidFormat",
        "message": "利用者IDの形式を確認してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

#### 18.3 利用者が存在しない場合

指定された利用者が存在しない場合、
または論理削除されている場合は、
`USER_NOT_FOUND`
を返却する。

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "指定された利用者が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

#### 18.4 objectiveIdのバリデーションエラー

`objectiveId`が
バリデーション条件を満たさない場合は、
`VALIDATION_ERROR`
を返却する。

例えば、
以下の場合を含む。

```text
objectiveId = 0
objectiveId = -1
objectiveId = abc
objectiveId = 1.5
```

レスポンス例：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります。",
    "details": [
      {
        "field": "objectiveId",
        "reason": "invalidFormat",
        "message": "目的IDの形式を確認してください。"
      }
    ],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

---

#### 18.5 目的が存在しない場合

以下の条件を満たす
目的が存在しない場合は、
`OBJECTIVE_NOT_FOUND`
を返却する。

```text
objectives.id = objectiveId
AND
objectives.user_id = 操作対象利用者ID
```

レスポンス例：

```json
{
  "error": {
    "code": "OBJECTIVE_NOT_FOUND",
    "message": "指定された目的が見つかりません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

以下の場合を含む。

- 指定された`objectiveId`が存在しない
- 指定された目的が他の利用者に属している
- 指定された目的が論理削除されている

他利用者の目的が
存在すること自体を
レスポンスから判別できないようにする。

---

#### 18.6 目的が判定対象外の場合

指定された目的が存在するものの、
業務ルール上
目的達成判定の対象外となる状態の場合は、
`OBJECTIVE_NOT_ASSESSABLE`
を返却する。

```json
{
  "error": {
    "code": "OBJECTIVE_NOT_ASSESSABLE",
    "message": "指定された目的は目的達成判定の対象ではありません。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

具体的な判定対象外条件は、
目的管理および
機能要件で定義された
目的の状態管理ルールに従う。

判定対象外の目的については、
目的達成判定履歴を作成しない。

---

#### 18.7 確定済み月末資産状況が存在しない場合

操作対象利用者について、
確定済み月末資産状況が
存在しない場合は、
APIエラーとはしない。

```text
確定済み月末資産状況なし
    ↓
判定不可
    ↓
目的達成判定履歴登録
    ↓
201 Created
```

判定結果として
「判定不可」を返却する。

概念例：

```json
{
  "data": {
    "id": "16",
    "objectiveId": "5",
    "result": "unassessable"
  }
}
```

---

#### 18.8 手取り収入が不足している場合

直近3ヶ月分の
手取り収入が揃っていない場合も、
APIエラーとはしない。

```text
手取り収入3ヶ月未満
    ↓
判定不可
    ↓
目的達成判定履歴登録
    ↓
201 Created
```

1ヶ月または2ヶ月分の
手取り収入だけで
平均値を算出して判定しない。

---

#### 18.9 その他の判定材料が不足している場合

目的達成判定に必要な
業務データが不足している場合は、
原則としてAPIエラーとはせず、
判定不可として扱う。

例えば、
以下を含む。

- 判定に必要な目的情報が不足している
- 判定に必要な資産情報を取得できない
- 判定式に必要な業務値を確定できない

この場合も、
判定不可として
目的達成判定履歴を登録する。

ただし、
単なる判定材料不足ではなく
データベース不整合や
想定外のシステム障害である場合は、
`INTERNAL_SERVER_ERROR`
として扱う。

---

#### 18.10 想定外のエラーが発生した場合

データベース接続エラーなど、
想定外のサーバーエラーが発生した場合は、
`INTERNAL_SERVER_ERROR`
を返却する。

```json
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "サーバー内部でエラーが発生しました。",
    "details": [],
    "requestId": "01JABCDEFGHJKMNPQRSTVWXYZ"
  }
}
```

この場合、
目的達成判定履歴を
登録しない。

SQL、
スタックトレース、
PostgreSQLの制約名および
内部例外メッセージは、
レスポンスへ含めない。

---

### 19 HTTPステータス

| HTTPステータス | 条件 |
|---:|---|
| `201 Created` | 目的達成判定を実行し、目的達成判定履歴を登録した |
| `400 Bad Request` | 利用者コンテキストが未指定、または利用者IDの形式が不正である |
| `404 Not Found` | 指定された利用者または目的が存在しない |
| `409 Conflict` | 目的が現在の業務状態では判定対象外である |
| `422 Unprocessable Entity` | `objectiveId`のバリデーションエラー |
| `500 Internal Server Error` | 想定外のサーバーエラーが発生した |

---

#### 19.1 201の扱い

目的達成判定が実行され、
新しい目的達成判定履歴が
作成された場合は、
`201 Created`
を返却する。

以下の判定結果すべてを
正常終了として扱う。

```text
達成可能
達成困難
判定不可
```

判定不可であっても、
目的達成判定を実行した結果として
履歴が作成されるため、
`201 Created`を返却する。

---

#### 19.2 400の扱い

以下の場合は、
`400 Bad Request`
を返却する。

- `X-User-Id`が指定されていない
- `X-User-Id`の形式が不正である

---

#### 19.3 404の扱い

以下の場合は、
`404 Not Found`
を返却する。

- 指定された利用者が存在しない
- 指定された利用者が論理削除されている
- 指定された目的が存在しない
- 指定された目的が他の利用者に属している
- 指定された目的が論理削除されている

他利用者に属する目的を指定した場合も、
対象リソースの存在を公開しない。

---

#### 19.4 409の扱い

指定された目的自体は存在するが、
現在の業務状態では
目的達成判定の対象とならない場合は、
`409 Conflict`
を返却する。

例えば、
目的管理の業務ルールで
無効な目的を
判定対象外としている場合などが該当する。

---

#### 19.5 422の扱い

`objectiveId`の形式が
不正な場合は、
`422 Unprocessable Entity`
を返却する。

判定材料不足による
「判定不可」は、
`422`として扱わない。

---

### 20 エラーコード

| エラーコード | HTTPステータス | 条件 | 再試行 |
|---|---:|---|:---:|
| `USER_CONTEXT_REQUIRED` | 400 | `X-User-Id`が指定されていない | × |
| `INVALID_USER_ID` | 400 | 利用者IDの形式が不正である | × |
| `USER_NOT_FOUND` | 404 | 指定された利用者が存在しない、または論理削除されている | × |
| `VALIDATION_ERROR` | 422 | `objectiveId`の形式が不正である | × |
| `OBJECTIVE_NOT_FOUND` | 404 | 目的が存在しない、論理削除済み、または他利用者に属している | × |
| `OBJECTIVE_NOT_ASSESSABLE` | 409 | 目的が現在の業務状態では判定対象外である | × |
| `INTERNAL_SERVER_ERROR` | 500 | 想定外のサーバーエラーが発生した | ○ |

判定材料不足は、
エラーコードとして扱わない。

以下は、
目的達成判定結果の
「判定不可」として扱う。

- 確定済み月末資産状況が存在しない
- 直近3ヶ月分の手取り収入が不足している
- その他の判定材料が不足している

エラーコードの正式な定義は、
`error-codes.md`
に従う。

---

### 21 副作用

本APIでは、
目的達成判定を実行し、
`assessment_histories`へ
新しい目的達成判定履歴を
1件登録する。

判定結果が
「判定不可」の場合も、
履歴を登録する。

本APIの実行によって、
以下の業務データは変更しない。

- `objectives`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `net_incomes`
- 既存の`assessment_histories`

また、
以下の処理を自動実行しない。

- 月末資産状況の作成
- 月末資産状況の確定
- 月末資産状況の確定解除
- 月末資産残高の登録・更新
- 商品別月末評価額の登録・更新
- 手取り収入の登録・更新
- 目的の登録・更新
- 過去の目的達成判定履歴の再計算
- 過去の目的達成判定履歴の上書き

---

### 22 トランザクション

目的達成判定に必要なデータ取得後、
判定結果の確定から
目的達成判定履歴の登録までを、
1つのデータベーストランザクション内で実行する。

概念的には、
以下の処理を行う。

```text
目的取得
    ↓
最新の確定済み月末資産状況取得
    ↓
資産情報取得・集計
    ↓
手取り収入取得・平均算出
    ↓
判定材料確認
    ↓
判定結果生成
    ↓
assessment_histories INSERT
```

判定結果が
「判定不可」の場合も、
正常な判定結果として
`assessment_histories`へ登録する。

履歴登録時に
想定外の例外が発生した場合は、
トランザクションをロールバックする。

目的達成判定履歴が
不完全な状態で残ってはならない。

---

#### 22.1 参照データの扱い

ASM-001では、
判定元となる

- `objectives`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `net_incomes`

を更新しない。

そのため、
これらの参照データを
判定実行のためだけに
不要にロックしない。

ただし、
判定結果として保存する内容と
参照データの一貫性について
将来的に厳密な要件が生じた場合は、
トランザクション分離レベルや
スナップショット値保存方式を
別途検討する。

---

### 23 冪等性

本APIは、
目的達成判定を実行するたびに
新しい目的達成判定履歴を作成する。

そのため、
本APIは冪等ではない。

同一の目的について、
同一の業務データの状態で
同じリクエストを複数回実行した場合でも、
実行回数分の
目的達成判定履歴を作成する。

例えば、

```text
1回目
POST /api/v1/objectives/5/assessments
    ↓
assessment_histories
id = 15
result = achievable
    ↓
201 Created


2回目
POST /api/v1/objectives/5/assessments
    ↓
assessment_histories
id = 16
result = achievable
    ↓
201 Created
```

判定結果が同じであっても、
1回目の履歴を返却したり、
既存履歴を上書きしたりしない。

判定不可の場合も同様とする。

```text
1回目
result = unassessable
    ↓
履歴Aを作成

2回目
result = unassessable
    ↓
履歴Bを作成
```

これは、
目的達成判定履歴が

```text
判定を実行した事実
```

を保持するためである。

Phase1では、
`Idempotency-Key`は採用しない。

フロントエンドでは、
判定処理中に
判定実行ボタンを非活性化し、
意図しない連続実行を抑止する。

ただし、
複数回実行された場合に
複数の判定履歴が作成されること自体は、
本APIの仕様とする。

---

### 24 関連テーブル

#### 24.1 objectives

目的達成判定の
対象となる目的を保持する。

本APIでは、
`objectiveId`で指定された目的を取得し、
操作対象利用者との
利用者境界を確認するために参照する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 目的ID |
| `user_id` | 利用者境界確認 |
| 目的達成判定に必要な各カラム | 判定条件の取得 |
| `deleted_at` | 論理削除状態の確認 |

取得条件は、
以下とする。

```text
id = objectiveId
AND
user_id = 操作対象利用者ID
AND
deleted_at IS NULL
```

他の利用者に属する目的は、
判定対象としない。

本APIでは、
`objectives`を更新しない。

---

#### 24.2 assessment_histories

目的達成判定の
実行結果を履歴として保持する。

本APIで
新しいレコードを登録する
主要テーブルである。

主に以下の情報を保存する。

- 判定対象となった目的
- 判定結果
- 判定実行時点で保持する必要がある判定情報
- 登録日時

具体的な保存カラムは、
`assessment_histories`の
テーブル定義に従う。

目的との関連は、
以下とする。

```text
assessment_histories.objective_id
    = objectives.id
```

ASM-001を実行するたびに、
新しい目的達成判定履歴を
1件登録する。

既存の
`assessment_histories`は
更新しない。

---

#### 24.3 month_end_asset_snapshots

対象年月ごとの
月末資産状況を保持する。

本APIでは、
操作対象利用者について
最新の確定済み月末資産状況を
取得するために参照する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 月末資産状況ID |
| `user_id` | 利用者境界確認 |
| `target_year_month` | 最新の確定済み月末資産状況を決定 |
| `confirmed` | 確定済みデータのみを対象とする判定 |

取得条件は、
概念的に以下とする。

```text
user_id = 操作対象利用者ID
AND
confirmed = true
ORDER BY target_year_month DESC
LIMIT 1
```

未確定の月末資産状況は、
目的達成判定に使用しない。

本APIでは、
`month_end_asset_snapshots`を更新しない。

---

#### 24.4 month_end_asset_balances

口座単位で残高を記録する
資産口座について、
月末資産残高を保持する。

本APIでは、
最新の確定済み月末資産状況に紐づく
口座単位の資産額を取得するために参照する。

概念的には、
以下の関連から取得する。

```text
month_end_asset_balances.month_end_asset_snapshot_id
    = month_end_asset_snapshots.id
```

取得した`balance`は、
目的達成判定に使用する
総資産額の算出対象とする。

商品単位で残高を記録する資産口座については、
`month_end_asset_balances`を
総資産額へ重複加算しない。

本APIでは、
`month_end_asset_balances`を更新しない。

---

#### 24.5 month_end_holding_values

商品単位で残高を記録する
資産口座について、
保有商品ごとの
商品別月末評価額を保持する。

本APIでは、
最新の確定済み月末資産状況に紐づく
商品別月末評価額を取得するために参照する。

概念的には、
以下の関連から取得する。

```text
month_end_holding_values.month_end_asset_snapshot_id
    = month_end_asset_snapshots.id
```

取得した`value`は、
目的達成判定に使用する
総資産額の算出対象とする。

口座単位で残高を記録する資産口座については、
商品別月末評価額を
総資産額へ加算しない。

本APIでは、
`month_end_holding_values`を更新しない。

---

#### 24.6 asset_accounts

資産口座を保持する。

本APIでは、
各資産口座の
残高記録単位を確認し、

```text
month_end_asset_balances
```

または

```text
month_end_holding_values
```

のどちらを
総資産額の算出に使用するかを
判断するために参照する。

主に以下のカラムを使用する。

| カラム | 用途 |
|---|---|
| `id` | 資産口座ID |
| `user_id` | 利用者境界確認 |
| `balance_recording_unit` | 口座単位・商品単位の判定 |

概念的には、
以下のように扱う。

```text
balance_recording_unit = 口座単位
    → month_end_asset_balances.balance

balance_recording_unit = 商品単位
    → month_end_holding_values.value
```

本APIでは、
`asset_accounts`を更新しない。

---

#### 24.7 holding_assets

商品単位で残高を記録する
資産口座について、
保有商品を保持する。

本APIでは、
`month_end_holding_values`と
資産口座との関連を確認するために参照する。

概念的には、
以下の関連を使用する。

```text
month_end_holding_values.holding_asset_id
    = holding_assets.id

holding_assets.asset_account_id
    = asset_accounts.id
```

本APIでは、
`holding_assets`を更新しない。

---

#### 24.8 net_incomes

利用者ごとの
対象年月の手取り収入を保持する。

本APIでは、
操作対象利用者について
直近3ヶ月分の
手取り収入を取得するために参照する。

取得条件は、
概念的に以下とする。

```text
user_id = 操作対象利用者ID
ORDER BY target_year_month DESC
LIMIT 3
```

3件取得できた場合は、
その合計を3で除算し、
平均手取り収入を算出する。

```text
直近3ヶ月の手取り収入合計
÷
3
=
平均手取り収入
```

3ヶ月分が揃っていない場合は、
判定結果を
「判定不可」とする。

本APIでは、
`net_incomes`を更新しない。

---

#### 24.9 users

操作対象となる
利用者を保持する。

`X-User-Id`で指定された利用者が
存在することを確認するために参照する。

概念的には、
以下を確認する。

```text
id = X-User-Id
AND
deleted_at IS NULL
```

本APIでは、
`users`を更新しない。

---

#### 24.10 asset_account_available_settings

本APIでは、
原則として
`asset_account_available_settings`を
目的達成判定実行時に
再判定するためには使用しない。

目的達成判定では、
確定済み月末資産状況に
保存されている月末資産データを使用する。

対象年月時点で
どの資産口座が
月末資産管理対象であるかについては、
SNP-004 月末資産状況確定APIまでの
業務ルールによって保証されていることを
前提とする。

本APIでは、
`asset_account_available_settings`を
更新しない。

---

### 25 関連する機能要件

- 目的管理
  - 利用者は目的を登録できる
  - 目的達成判定は登録済みの目的に対して実行する
  - 他の利用者に属する目的は判定できない

- 月末資産管理
  - 目的達成判定には確定済みの月末資産状況のみを使用する
  - 複数の確定済み月末資産状況が存在する場合は、最新の対象年月を使用する
  - 未確定の月末資産状況は判定に使用しない

- 資産状況
  - 口座単位で記録する資産口座は月末資産残高を使用する
  - 商品単位で記録する資産口座は商品別月末評価額を使用する
  - 同一資産を重複して総資産額へ加算しない
  - 判定対象となる資産額を合計して総資産額を算出する

- 手取り収入管理
  - 操作対象利用者の手取り収入のみを使用する
  - 直近3ヶ月分の手取り収入から平均手取り収入を算出する
  - 3ヶ月分が揃っていない場合は判定不可とする

- 目的達成判定
  - 登録されている目的、資産状況および手取り収入などをもとに判定する
  - 判定結果として達成可能、達成困難、判定不可を区別する
  - 判定に必要な情報が不足している場合はAPIエラーではなく判定不可とする
  - 判定不可の場合も目的達成判定履歴を保存する
  - 判定実行ごとに新しい目的達成判定履歴を作成する
  - 過去の目的達成判定履歴を上書きしない
  - 元データが後から変更されても過去の判定履歴を自動再計算しない

- 利用者境界
  - 操作対象利用者に属するデータのみを判定材料として使用する
  - 他の利用者に属する目的や判定材料を参照しない
  - 他の利用者に属するリソースの存在をレスポンスから推測できないようにする

具体的な章番号および
詳細な判定計算式については、
`functional-requirements.md`の
最新定義に従う。

---

### 26 テスト観点

#### 26.1 正常系

判定に必要な情報が
すべて揃っている状態で
ASM-001を実行する。

以下を確認する。

- 目的達成判定を実行できること
- `201 Created`となること
- `assessment_histories`へ1件登録されること
- 登録された履歴が指定した目的に紐づくこと
- 判定結果が正しく保存されること
- レスポンスの判定結果と保存された判定結果が一致すること
- 既存の目的達成判定履歴が変更されないこと
- 判定元となる業務データが変更されないこと

---

#### 26.2 達成可能

業務ルール上、
目的を達成可能と判定される
データを用意する。

期待結果：

```text
201 Created
```

判定結果が
「達成可能」となること。

`assessment_histories`へ
達成可能の判定結果が
保存されること。

---

#### 26.3 達成困難

業務ルール上、
目的を達成困難と判定される
データを用意する。

期待結果：

```text
201 Created
```

判定結果が
「達成困難」となること。

`assessment_histories`へ
達成困難の判定結果が
保存されること。

---

#### 26.4 判定不可

判定対象となる目的は存在するが、
判定材料が不足している状態で実行する。

期待結果：

```text
201 Created
```

以下を確認する。

- APIエラーとならないこと
- 判定結果が「判定不可」となること
- `assessment_histories`へ1件登録されること
- 過去の判定履歴が変更されないこと

---

#### 26.5 確定済み月末資産状況が存在しない

操作対象利用者について、
確定済み月末資産状況が
存在しない状態で実行する。

期待結果：

```text
201 Created
判定結果 = 判定不可
```

以下を確認する。

- `404`や`409`などのAPIエラーとならないこと
- 判定不可として履歴が保存されること
- 未確定の月末資産状況を代わりに使用しないこと

---

#### 26.6 未確定の月末資産状況のみ存在する

以下の状態を用意する。

```text
month_end_asset_snapshots
confirmed = false
```

確定済み月末資産状況は
存在しないものとする。

期待結果：

```text
201 Created
判定結果 = 判定不可
```

未確定データを
目的達成判定に使用しないこと。

---

#### 26.7 最新の確定済み月末資産状況

複数の確定済み月末資産状況を用意する。

例：

```text
2026-05 confirmed = true
2026-06 confirmed = true
2026-07 confirmed = true
```

期待結果：

```text
2026-07
```

の月末資産状況を
判定に使用すること。

古い確定済み月末資産状況を
誤って使用しないこと。

---

#### 26.8 最新月が未確定の場合

以下の状態を用意する。

```text
2026-06 confirmed = true
2026-07 confirmed = true
2026-08 confirmed = false
```

期待結果：

```text
2026-07
```

を最新の確定済み月末資産状況として
使用すること。

`2026-08`の未確定データを
使用しないこと。

---

#### 26.9 口座単位の資産口座

残高記録単位が
口座単位の資産口座を用意する。

```text
balance_recording_unit = 口座単位
```

期待結果：

- `month_end_asset_balances.balance`が総資産額へ加算されること
- 同じ資産口座の商品別月末評価額を重複加算しないこと

---

#### 26.10 商品単位の資産口座

残高記録単位が
商品単位の資産口座を用意する。

```text
balance_recording_unit = 商品単位
```

期待結果：

- `month_end_holding_values.value`が総資産額へ加算されること
- 同じ資産口座の月末資産残高を重複加算しないこと

---

#### 26.11 複数資産口座の総資産額

以下を組み合わせる。

```text
口座単位資産口座A
口座単位資産口座B
商品単位資産口座C
```

商品単位資産口座Cには
複数の保有商品を登録する。

期待結果：

```text
口座Aのbalance
+
口座Bのbalance
+
口座Cの商品1 value
+
口座Cの商品2 value
...
=
総資産額
```

となること。

---

#### 26.12 0円の資産

月末資産残高または
商品別月末評価額に
0円のデータを含める。

以下を確認する。

- `0`を未登録として扱わないこと
- 0円として正常に集計されること
- 判定処理が異常終了しないこと

---

#### 26.13 直近3ヶ月の手取り収入

例えば、
以下の手取り収入を用意する。

```text
2026-05
2026-06
2026-07
2026-08
```

期待結果：

```text
2026-06
2026-07
2026-08
```

の3件を使用すること。

最も古い
`2026-05`を
平均計算に含めないこと。

---

#### 26.14 平均手取り収入

直近3ヶ月の
手取り収入として、
計算結果を確認しやすい値を用意する。

例：

```text
300000
330000
360000
```

期待結果：

```text
平均手取り収入 = 330000
```

となること。

算出された平均手取り収入が
目的達成判定に使用されること。

---

#### 26.15 手取り収入が2ヶ月分

手取り収入を
2ヶ月分だけ登録する。

期待結果：

```text
201 Created
判定結果 = 判定不可
```

2件だけを使用して
平均値を算出しないこと。

---

#### 26.16 手取り収入が1ヶ月分

手取り収入を
1ヶ月分だけ登録する。

期待結果：

```text
201 Created
判定結果 = 判定不可
```

1件だけを使用して
判定を実行しないこと。

---

#### 26.17 手取り収入が存在しない

`net_incomes`が
1件も存在しない状態で実行する。

期待結果：

```text
201 Created
判定結果 = 判定不可
```

目的達成判定履歴が
保存されること。

---

#### 26.18 objectiveId形式不正

以下のような
不正な`objectiveId`を指定する。

```text
0
-1
abc
1.5
```

期待結果：

```text
422 Unprocessable Entity
VALIDATION_ERROR
```

以下を確認する。

- 目的達成判定が実行されないこと
- `assessment_histories`へレコードが登録されないこと

---

#### 26.19 目的不存在

存在しない
`objectiveId`を指定する。

期待結果：

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

目的達成判定履歴が
登録されないこと。

---

#### 26.20 他利用者の目的

操作対象利用者とは
異なる利用者に属する
`objectiveId`を指定する。

期待結果：

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

以下を確認する。

- 他利用者の目的が存在することをレスポンスから判別できないこと
- 他利用者の目的を判定できないこと
- `assessment_histories`へレコードが登録されないこと

---

#### 26.21 論理削除済みの目的

論理削除されている目的を指定する。

期待結果：

```text
404 Not Found
OBJECTIVE_NOT_FOUND
```

目的達成判定履歴が
登録されないこと。

---

#### 26.22 判定対象外の目的

目的は存在するが、
目的管理の業務ルール上
判定対象外となる状態を用意する。

期待結果：

```text
409 Conflict
OBJECTIVE_NOT_ASSESSABLE
```

目的達成判定履歴が
登録されないこと。

---

#### 26.23 利用者コンテキスト未指定

`X-User-Id`を
指定せずに実行する。

期待結果：

```text
400 Bad Request
USER_CONTEXT_REQUIRED
```

目的達成判定履歴が
登録されないこと。

---

#### 26.24 利用者ID形式不正

不正な`X-User-Id`を指定する。

例：

```http
X-User-Id: abc
```

期待結果：

```text
400 Bad Request
INVALID_USER_ID
```

目的達成判定履歴が
登録されないこと。

---

#### 26.25 利用者不存在

存在しない利用者IDを
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

目的達成判定履歴が
登録されないこと。

---

#### 26.26 論理削除済み利用者

論理削除済み利用者を
`X-User-Id`へ指定する。

期待結果：

```text
404 Not Found
USER_NOT_FOUND
```

目的達成判定履歴が
登録されないこと。

---

#### 26.27 他利用者の資産情報を使用しないこと

User AとUser Bについて、
それぞれ以下を登録する。

- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 手取り収入

User Aを
`X-User-Id`として
ASM-001を実行する。

以下を確認する。

- User Aの資産情報のみが使用されること
- User Bの月末資産状況が使用されないこと
- User Bの月末資産残高が総資産額へ加算されないこと
- User Bの商品別月末評価額が総資産額へ加算されないこと
- User Bの手取り収入が平均値へ含まれないこと

---

#### 26.28 判定履歴の新規登録

同一目的について
ASM-001を2回実行する。

期待結果：

```text
1回目
assessment_histories A

2回目
assessment_histories B
```

以下を確認する。

- 2件の履歴が存在すること
- 1回目の履歴が上書きされないこと
- それぞれ異なる履歴IDを持つこと
- 同じ判定結果でも新しい履歴が作成されること

---

#### 26.29 判定不可履歴の複数登録

判定不可となる状態で
ASM-001を複数回実行する。

以下を確認する。

- 実行回数分の履歴が作成されること
- 過去の判定不可履歴が上書きされないこと
- すべて正常レスポンスとなること

---

#### 26.30 過去履歴を再計算しないこと

1回目のASM-001実行後に、
以下のいずれかを変更する。

- 目的
- 月末資産状況
- 月末資産残高
- 商品別月末評価額
- 手取り収入

その後、
過去の`assessment_histories`を確認する。

以下を確認する。

- 過去の判定結果が変更されていないこと
- 過去履歴が自動更新されていないこと

---

#### 26.31 再判定

判定元データを変更した後に、
ASM-001を再度実行する。

以下を確認する。

- 最新データを使用して新しい判定が行われること
- 新しい`assessment_histories`が作成されること
- 以前の判定履歴が保持されること

---

#### 26.32 副作用

正常終了後に、
以下のテーブルが
変更されていないことを確認する。

- `objectives`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `asset_account_available_settings`
- `holding_assets`
- `net_incomes`
- 既存の`assessment_histories`

新規登録されるのは、
今回の目的達成判定に対応する
`assessment_histories`のみであること。

---

#### 26.33 トランザクション

判定結果生成後、
`assessment_histories`登録時に
例外を発生させる。

以下を確認する。

- `201 Created`とならないこと
- 不完全な目的達成判定履歴が残らないこと
- トランザクションがロールバックされること
- 判定元データが変更されないこと

---

#### 26.34 冪等性

同一の目的、
同一の判定材料で
ASM-001を複数回実行する。

期待結果：

```text
1回目
201 Created
履歴A

2回目
201 Created
履歴B
```

以下を確認する。

- 本APIが冪等ではないこと
- 実行回数分の判定履歴が作成されること
- 同じ判定結果でも履歴が増えること
- 既存履歴を再利用しないこと

---

#### 26.35 レスポンス契約

正常終了時に、
API共通方針で定めた
Envelope形式で返却されること。

概念例：

```json
{
  "data": {
    "id": "15",
    "objectiveId": "5",
    "result": "achievable"
  }
}
```

以下を確認する。

- `data`がobjectであること
- `id`がstringであること
- `objectiveId`がstringであること
- `result`が定義された値であること
- 判定不可の場合も同じレスポンス構造であること
- JSONフィールド名がcamelCaseであること

本APIでは、
以下の情報が
レスポンスへ無条件に含まれていないことを確認する。

- `user_id`
- `month_end_asset_snapshots`
- `month_end_asset_balances`
- `month_end_holding_values`
- `asset_accounts`
- `holding_assets`
- `net_incomes`
- `created_at`
- `updated_at`

---

#### 26.36 エラーレスポンス

各APIエラーについて、
API共通方針で定めた
共通エラーレスポンス形式で
返却されることを確認する。

以下を確認する。

- `error.code`が期待するエラーコードであること
- `error.message`が設定されていること
- `error.details`が配列であること
- `error.requestId`が設定されていること
- SQLが含まれていないこと
- PostgreSQLの制約名が含まれていないこと
- スタックトレースが含まれていないこと
- 内部例外メッセージが含まれていないこと

---

#### 26.37 APIエラー時の履歴

以下のAPIエラーを発生させる。

- `USER_CONTEXT_REQUIRED`
- `INVALID_USER_ID`
- `USER_NOT_FOUND`
- `VALIDATION_ERROR`
- `OBJECTIVE_NOT_FOUND`
- `OBJECTIVE_NOT_ASSESSABLE`
- `INTERNAL_SERVER_ERROR`

各ケースについて、
`assessment_histories`へ
新しいレコードが
登録されていないことを確認する。

「判定不可」と
APIエラーが
明確に区別されていることを確認する。

---

### 27 Laravel実装方針

ASM-001では、Action、Request、UseCase、Query、Repository、目的達成判定ロジック、Model、API Resource、Responderを分離して実装する。

概念的な処理構成は、以下とする。

```text
Route
    ↓
Middleware
    ↓
Request
    ↓
Action
    ↓
UseCase
    ├─ ObjectiveQuery
    ├─ MonthEndAssetSnapshotQuery
    ├─ AssetAssessmentQuery
    ├─ NetIncomeQuery
    ├─ AssessmentCalculator
    └─ AssessmentHistoryRepository
    ↓
API Resource
    ↓
Responder
```

目的達成判定に必要な業務フローの制御は、UseCaseへ集約する。

目的達成判定の具体的な計算ロジックは、`AssessmentCalculator`へ分離する。

Actionへ目的達成判定の業務ロジックを直接記述しない。

---

#### 27.1 Action

HTTPリクエストを受け付け、目的IDおよび利用者コンテキストを取得する。

目的達成判定UseCaseを呼び出し、処理結果をResponderへ渡す。

概念例：

```php
final class ExecuteAssessmentAction
{
    public function __invoke(
        ExecuteAssessmentRequest $request,
        ExecuteAssessmentUseCase $useCase,
        AssessmentHistoryResponder $responder,
        string $objectiveId,
    ): JsonResponse {
        $assessmentHistory = $useCase->execute(
            userId: $request->userId(),
            objectiveId: $objectiveId,
        );

        return $responder->created(
            $assessmentHistory,
        );
    }
}
```

Actionでは、以下の処理を行わない。

* 目的IDの形式検証
* 目的の検索
* 利用者境界の判定
* 目的の判定対象状態の判定
* 最新の確定済み月末資産状況の検索
* 総資産額の集計
* 直近3ヶ月の手取り収入取得
* 平均手取り収入の算出
* 判定不可条件の判定
* 目的達成判定の計算
* 目的達成判定履歴の登録
* トランザクション制御
* レスポンス形式への変換

---

#### 27.2 Request

パスパラメータの`objectiveId`を検証する。

本APIでは、リクエストボディを使用しない。

`objectiveId`は、`prepareForValidation()`でバリデーション対象へ追加する。

概念例：

```php
final class ExecuteAssessmentRequest extends FormRequest
{
    protected function prepareForValidation(): void
    {
        $this->merge([
            'objectiveId'
                => $this->route('objectiveId'),
        ]);
    }

    public function rules(): array
    {
        return [
            'objectiveId' => [
                'required',
                'integer',
                'min:1',
            ],
        ];
    }
}
```

Requestでは、以下の処理を行わない。

* 目的の存在確認
* 利用者境界の判定
* 目的の判定対象状態の判定
* 月末資産状況の取得
* 資産額の集計
* 手取り収入の取得
* 判定不可条件の判定
* 目的達成判定

これらは、入力形式の検証ではなく業務ルールとして扱う。

---

#### 27.3 UseCase

目的達成判定実行のユースケース処理を担当する。

主な処理は、以下とする。

1. 操作対象利用者IDを受け取る
2. 目的IDを受け取る
3. 操作対象利用者に属する目的を取得する
4. 目的が判定対象となる状態であることを確認する
5. 最新の確定済み月末資産状況を取得する
6. 判定対象となる資産額を取得・集計する
7. 直近3ヶ月の手取り収入を取得する
8. 平均手取り収入を算出する
9. 判定に必要な情報が揃っているか確認する
10. 判定可能な場合は目的達成判定を実行する
11. 判定できない場合は判定不可結果を生成する
12. 目的達成判定履歴を新規登録する
13. 登録した目的達成判定履歴を返却する

概念的には、以下の流れとする。

```text
目的取得
    ↓
判定対象状態確認
    ↓
最新確定済み月末資産状況取得
    ↓
資産額取得・集計
    ↓
直近3ヶ月の手取り収入取得
    ↓
平均手取り収入算出
    ↓
判定材料確認
    ├─ 不足あり
    │      ↓
    │   判定不可結果生成
    │
    └─ 不足なし
           ↓
       AssessmentCalculator
           ↓
       判定結果生成
    ↓
AssessmentHistoryRepository
    ↓
目的達成判定履歴登録
```

判定材料が不足している場合は、例外によって処理を終了しない。

```text
判定材料不足
    ↓
AssessmentResult::unassessable()
    ↓
目的達成判定履歴登録
    ↓
201 Created
```

一方、目的が存在しないなどAPIとして処理できない状態については、業務例外を送出する。

APIエラーが発生した場合は、目的達成判定履歴を登録しない。

UseCaseでは、目的達成判定履歴以外の業務データを更新しない。

以下のデータは参照のみとする。

* `objectives`
* `month_end_asset_snapshots`
* `month_end_asset_balances`
* `month_end_holding_values`
* `asset_accounts`
* `holding_assets`
* `net_incomes`

---

#### 27.4 ObjectiveQuery

操作対象利用者に属する目的を取得する。

検索条件には、必ず利用者IDを含める。

概念例：

```php
$objective =
    Objective::query()
        ->whereKey($objectiveId)
        ->where('user_id', $userId)
        ->first();
```

他の利用者に属する目的が存在する場合も、取得結果は`null`とする。

これにより、他の利用者に属する目的の存在をAPIから推測できないようにする。

Queryでは、目的達成判定を実行しない。

---

#### 27.5 MonthEndAssetSnapshotQuery

操作対象利用者に属する最新の確定済み月末資産状況を取得する。

概念例：

```php
return MonthEndAssetSnapshot::query()
    ->where('user_id', $userId)
    ->where('confirmed', true)
    ->orderByDesc('target_year_month')
    ->first();
```

未確定の月末資産状況は、判定材料として取得しない。

確定済み月末資産状況が存在しない場合は、Queryから例外を送出せず、`null`を返却する。

`null`の場合に判定不可とする判断は、UseCase側で行う。

---

#### 27.6 AssetAssessmentQuery

目的達成判定に使用する資産額を取得・集計する。

最新の確定済み月末資産状況に紐づく資産データのみを対象とする。

資産口座の残高記録単位に応じて、

```text
口座単位
    ↓
month_end_asset_balances.balance

商品単位
    ↓
month_end_holding_values.value
```

を使用する。

口座単位と商品単位の金額を重複して加算しない。

Queryは、総資産額など目的達成判定に必要な参照結果を返却する。

目的達成判定そのものは実行しない。

---

#### 27.7 NetIncomeQuery

操作対象利用者に属する直近3ヶ月分の手取り収入を取得する。

概念例：

```php
return NetIncome::query()
    ->where('user_id', $userId)
    ->orderByDesc('target_year_month')
    ->limit(3)
    ->get();
```

他の利用者の手取り収入を取得しない。

3件未満の場合も、Queryでは例外を送出しない。

取得件数が不足している場合に判定不可とする判断は、UseCase側で行う。

---

#### 27.8 平均手取り収入

直近3ヶ月分の手取り収入が取得できた場合は、平均手取り収入を算出する。

```text
直近3ヶ月の手取り収入合計
÷
3
=
平均手取り収入
```

1ヶ月または2ヶ月分のみで平均値を算出して目的達成判定へ使用しない。

3ヶ月分が揃っていない場合は、判定不可とする。

平均値の算出結果は、目的達成判定用の入力値として`AssessmentCalculator`へ渡す。

---

#### 27.9 AssessmentInput

目的達成判定ロジックへ渡す入力値は、専用のDTOまたはValue Objectとして表現する。

概念例：

```php
final readonly class AssessmentInput
{
    public function __construct(
        public Objective $objective,
        public int $totalAssets,
        public int $averageNetIncome,
    ) {
    }
}
```

`AssessmentCalculator`へEloquent ModelやQuery Builderを直接依存させることは避ける。

判定ロジックに必要な値を`AssessmentInput`へまとめて渡す。

---

#### 27.10 AssessmentCalculator

目的達成判定の具体的な計算処理を担当する。

概念例：

```php
final class AssessmentCalculator
{
    public function calculate(
        AssessmentInput $input,
    ): AssessmentResult {
        // 目的達成判定計算

        return AssessmentResult::achievable();
    }
}
```

`AssessmentCalculator`では、以下を行わない。

* データベース検索
* Eloquentによるデータ取得
* 利用者境界の判定
* 目的達成判定履歴の登録
* HTTPレスポンス生成
* ログ出力

これにより、目的達成判定ロジックをデータベースやHTTPから独立させ、Unit Test可能な構成とする。

---

#### 27.11 AssessmentResult

目的達成判定結果は、単純な文字列ではなく、EnumまたはValue Objectで表現する。

概念例：

```php
enum AssessmentResultType: string
{
    case Achievable = 'achievable';
    case Difficult = 'difficult';
    case Unassessable = 'unassessable';
}
```

判定結果に追加情報が必要な場合は、Value Objectとして表現する。

概念例：

```php
final readonly class AssessmentResult
{
    public function __construct(
        public AssessmentResultType $type,
    ) {
    }
}
```

これにより、

```text
'achievable'
'difficult'
'unassessable'
```

などのマジック文字列をUseCaseやRepositoryへ散在させない。

---

#### 27.12 判定不可

判定不可となる条件は、UseCaseで判定材料を確認し、`AssessmentResult`として表現する。

例えば、

```text
確定済み月末資産状況なし
直近3ヶ月の手取り収入不足
目的の判定材料不足
必要な資産情報を取得できない
```

などの場合は、

```php
$result =
    AssessmentResult::unassessable();
```

のように判定不可結果を生成する。

判定不可はAPIエラーではないため、業務例外を送出しない。

判定不可の場合も、目的達成判定履歴を新規登録する。

---

#### 27.13 AssessmentHistoryRepository

目的達成判定結果を、`assessment_histories`へ新規登録する。

概念例：

```php
return AssessmentHistory::create([
    'objective_id' => $objective->id,
    'result' => $result->type->value,
]);
```

ASM-001を実行するたびに、新しいレコードを登録する。

```text
ASM-001実行
    ↓
常にINSERT
```

過去の目的達成判定履歴を検索して更新しない。

以下のような実装は行わない。

```php
updateOrCreate()
```

```php
firstOrCreate()
```

同一目的について同一条件で再判定した場合も、新しい履歴を登録する。

---

#### 27.14 トランザクション

目的達成判定結果の確定から目的達成判定履歴の登録までを、トランザクション内で実行する。

概念例：

```php
$assessmentHistory =
    DB::transaction(
        function () use (
            $userId,
            $objectiveId,
        ): AssessmentHistory {
            // 判定に必要なデータ取得
            // 判定材料確認
            // 目的達成判定
            // assessment_histories登録
        },
    );
```

処理途中でAPIエラーまたは想定外例外が発生した場合は、目的達成判定履歴を登録しない。

履歴登録後に例外が発生した場合も、トランザクションをロールバックする。

ASM-001では、`assessment_histories`以外の業務データを更新しない。

---

#### 27.15 排他制御

ASM-001では、同一目的に対して複数の判定要求が同時に実行された場合でも、それぞれを独立した目的達成判定履歴として保存する。

同一目的について複数の履歴が存在することは、正常な状態である。

そのため、履歴の重複登録を防止するためのUNIQUE制約や`lockForUpdate()`は使用しない。

```text
同時実行A
    ↓
assessment_histories A

同時実行B
    ↓
assessment_histories B
```

ただし、各リクエスト内で不完全な履歴が残らないことは、トランザクションによって保証する。

---

#### 27.16 Model

目的達成判定履歴には、`AssessmentHistory` Modelを使用する。

概念例：

```php
final class AssessmentHistory extends Model
{
    protected $fillable = [
        'objective_id',
        'result',
    ];
}
```

実際の`fillable`、`casts`、リレーションについては、`assessment_histories`のテーブル定義に従う。

目的との関連は、Eloquent Relationとして定義してよい。

---

#### 27.17 API Resource

データベースカラムを直接レスポンスへ返却せず、API Resourceを使用してAPIレスポンス形式へ変換する。

概念例：

```php
final class AssessmentHistoryResource
    extends JsonResource
{
    public function toArray(
        Request $request,
    ): array {
        return [
            'id' => (string) $this->id,
            'objectiveId'
                => (string) $this->objective_id,
            'result' => $this->result,
        ];
    }
}
```

IDは、API共通方針に従ってstringへ変換する。

データベースのsnake_caseカラムをそのまま返却しない。

---

#### 27.18 Responder

Responderは、登録済みの目的達成判定履歴を受け取り、API共通方針に従ったHTTPレスポンスへ変換する。

概念例：

```php
final class AssessmentHistoryResponder
{
    public function created(
        AssessmentHistory $history,
    ): JsonResponse {
        return response()->json(
            [
                'data' =>
                    new AssessmentHistoryResource(
                        $history,
                    ),
            ],
            Response::HTTP_CREATED,
        );
    }
}
```

正常終了時は、

```text
201 Created
```

を返却する。

判定結果が`unassessable`の場合も、正常終了として`201 Created`を返却する。

Responderでは、以下を行わない。

* データベース検索
* 目的達成判定
* 判定不可判定
* 利用者境界の判定
* 目的達成判定履歴の登録

---

#### 27.19 Middleware

以下の共通Middlewareを適用する。

* 利用者コンテキスト設定
* リクエストID生成
* JSONレスポンス共通処理
* 共通例外処理
* ログコンテキスト設定

利用者コンテキスト設定Middlewareでは、`X-User-Id`を検証し、操作対象利用者を特定する。

Action以降では、検証済みの利用者コンテキストを使用する。

---

#### 27.20 例外

APIとして処理できない業務状態は、専用例外として表現する。

例えば、以下を想定する。

```text
ObjectiveNotFoundException
ObjectiveNotAssessableException
```

これらを共通Exception HandlerでAPIエラーコードへ変換する。

概念例：

```text
ObjectiveNotFoundException
    ↓
OBJECTIVE_NOT_FOUND
    ↓
404 Not Found
```

判定不可については、例外を使用しない。

```text
判定材料不足
    ↓
AssessmentResult::unassessable()
    ↓
assessment_historiesへ保存
    ↓
201 Created
```

---

#### 27.21 想定外例外

想定外の例外は、API共通Exception Handlerで`INTERNAL_SERVER_ERROR`へ変換する。

レスポンスへ、以下を含めない。

* SQL
* PostgreSQLの制約名
* スタックトレース
* PHP内部エラー
* Laravel内部例外メッセージ

詳細情報は、サーバーログへ記録する。

---

#### 27.22 ログ

ASM-001では、必要に応じて以下の情報をログへ記録する。

```text
requestId
userId
objectiveId
assessmentHistoryId
result
```

判定不可理由を保存する設計の場合は、判定不可理由もログへ記録してよい。

ただし、以下の情報を不要にログへ出力しない。

* 目的の詳細情報
* 総資産額
* 個別資産額
* 手取り収入額

---

#### 27.23 テスト実装方針

Laravel側では、Feature Testを中心としてASM-001のAPI契約およびユースケース全体を確認する。

Feature Testでは、主に以下を確認する。

* `201 Created`
* `400 Bad Request`
* `404 Not Found`
* `409 Conflict`
* `422 Unprocessable Entity`
* `500 Internal Server Error`
* 利用者境界
* 最新確定済み月末資産状況の選択
* 未確定月末資産状況を使用しないこと
* 総資産額の集計
* 口座単位・商品単位の重複加算防止
* 直近3ヶ月の手取り収入取得
* 平均手取り収入の算出
* 判定不可
* 判定不可の場合も履歴が登録されること
* 目的達成判定履歴の新規登録
* 同一条件で複数回実行した場合の履歴追加
* APIエラー時に履歴が登録されないこと
* 想定外例外時に不完全な履歴が残らないこと
* 他の業務データへ副作用がないこと

目的達成判定の具体的な計算ロジックについては、`AssessmentCalculator`をUnit Testの対象とする。

Unit Testでは、データベースやHTTPへ依存せず、

```text
AssessmentInput
    ↓
AssessmentCalculator
    ↓
AssessmentResult
```

を確認する。

概念例：

```php
$result =
    $calculator->calculate(
        new AssessmentInput(
            objective: $objective,
            totalAssets: 3_000_000,
            averageNetIncome: 350_000,
        ),
    );

$this->assertSame(
    AssessmentResultType::Achievable,
    $result->type,
);
```

これにより、目的達成判定の計算ロジックをLaravelのHTTP層およびデータベースアクセスから分離して独立してテストできる構成とする。


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

### 29 設計上の補足

#### 29.1 POSTを採用する理由

ASM-001は、
目的達成判定を実行するだけでなく、
実行結果として
新しい目的達成判定履歴を作成する。

そのため、
HTTPメソッドには
`POST`を採用する。

---

#### 29.2 assessmentsを目的配下に配置する理由

目的達成判定は、
特定の目的に対して実行する。

また、
目的達成判定履歴も
目的に紐づくリソースである。

そのため、

```http
/objectives/{objectiveId}/assessments
```

として、
目的配下のサブリソースとして表現する。

---

#### 29.3 判定条件をリクエストボディで受け取らない理由

目的達成判定は、
システムに登録されている
正式な業務データをもとに
実行する。

クライアントから、

```text
現在資産
手取り収入
目的金額
```

などを直接受け取ると、
保存済みデータとは異なる条件で
判定できてしまう。

そのため、
クライアントからは
判定対象となる`objectiveId`のみを受け取り、
判定材料は
バックエンド側で取得する。

---

#### 29.4 最新の確定済み月末資産状況をサーバー側で決定する理由

フロントエンド側で
`snapshotId`を選択すると、
未確定データや
古い月末資産状況を
誤って指定できる可能性がある。

目的達成判定では、

```text
確定済み
+
最新
```

という業務ルールがあるため、
バックエンド側で
判定対象の月末資産状況を決定する。

---

#### 29.5 判定不可をAPIエラーとしない理由

判定不可は、
リクエストそのものが
不正な状態ではない。

例えば、
手取り収入が不足している利用者が
目的達成判定を実行する操作自体は
正しい。

そのため、

```text
判定材料不足
```

を

```text
4xx APIエラー
```

として扱わず、

```text
目的達成判定の結果
```

として表現する。

---

#### 29.6 判定不可でも履歴を保存する理由

目的達成判定履歴は、
判定結果だけではなく、

```text
利用者がその時点で判定を実行した
```

という事実を保持する。

そのため、
判定不可であっても
履歴を保存する。

これにより、
後から判定履歴を見たときに、

```text
当時は必要情報が不足していた
```

という状態も確認できる。

---

#### 29.7 毎回新しい履歴を作成する理由

目的達成判定は、
時間の経過によって

- 資産状況
- 手取り収入
- 目的条件

が変化する。

そのため、
最新の判定結果だけを
上書きして保持するのではなく、
実行ごとの結果を
履歴として残す。

これにより、
判定結果の推移を確認できる。

---

#### 29.8 ASM-001が冪等ではない理由

同一のリクエストを
複数回実行すると、
実行回数分の
目的達成判定履歴が作成される。

```text
1回目
    → 履歴A

2回目
    → 履歴B
```

最終的なデータ状態が
同一にならないため、
本APIは冪等ではない。

---

#### 29.9 Idempotency-Keyを採用しない理由

Phase1では、
利用者による
目的達成判定の実行頻度は低く、
決済処理のような
厳密な二重実行防止は
必要ないものとする。

そのため、
`Idempotency-Key`は採用しない。

代わりに、
フロントエンドで

- 実行中ボタンを非活性化する
- 自動リトライを無効にする

ことで、
意図しない重複実行を抑止する。

---

#### 29.10 判定ロジックをフロントエンドへ持たせない理由

目的達成判定を
React側でも実装すると、
バックエンドとの
判定結果の差異が生じる可能性がある。

そのため、
目的達成可否を決定する
業務ロジックは、
バックエンドへ集約する。

Reactは、

```text
判定を依頼する
+
結果を表示する
```

ことに責務を限定する。

---

#### 29.11 過去履歴を再計算しない理由

目的達成判定履歴は、
判定実行時点の結果を
保持する履歴である。

判定後に、

- 目的
- 資産
- 手取り収入

が変更されても、
過去の履歴を
現在の条件で書き換えない。

最新条件での結果が必要な場合は、
ASM-001を再実行して
新しい履歴を作成する。

---

#### 29.12 キャッシュを採用しない理由

ASM-001は、
新しい目的達成判定履歴を
作成する更新系APIである。

Phase1では、
ASM-001の結果を
アプリケーションキャッシュへ
保存しない。

判定成功後は、
ASM-002の一覧を
必要に応じて再取得する。

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