## エンティティ

### AssetAccount（資産口座）

役割
- 利用者が資産を保有する場所

関連
- @TODO
- HoldingAsset
- MonthEndAssetBalance
- AvailableAssets

### HoldingAsset（保有商品）

役割
- 商品単位で管理する金融商品

関連
- @TODO

### MonthEndHoldingValue（商品別月末評価額）

役割
- 保有商品ごとの対象年月の月末時点における評価額

関連
- @TODO

### MonthEndAssetBalance（月末資産残高）

役割
- 口座単位で管理する資産口座の対象年月の月末残高

関連
- @TODO

### MonthEndAssetSnapshot（月末資産状況）

役割
- 対象年月における資産口座および保有商品の残高情報全体

関連
- @TODO

### NetIncome（手取り収入）

役割
- 手取り収入

関連
- @TODO

### Objective（目的）

役割
- 達成可否を判断したいライフイベントまたは高額支出

関連
- @TODO

### AssessmentHistory（判定履歴）

役割
- 保存した目的達成判定結果の履歴

関連
- @TODO


## 派生概念

### AvailableAssets（利用可能資産）

役割
- 利用可能資産区分が「利用する」の資産を集計した結果

算出元
- @TODO

利用先
- @TODO

### AverageNetIncome（平均手取り収入）

役割
- 判定対象年月以前の連続する直近3か月の手取り収入から算出した平均額

算出元
- @TODO

利用先
- @TODO

### RemainingFunds（残余資金）

役割
- 目的達成後に残る利用可能資産

算出元
- @TODO

利用先
- @TODO

### RequiredRemainingFunds（必要残余資金）

役割
-目的達成後も維持すべき生活資金

算出元
- @TODO

利用先
- @TODO

### AssessmentTargetYearMonth（判定対象年月）

役割
-目的達成判定で利用する確定済み資産状況の対象年月

算出元
- @TODO

利用先
- @TODO

### ObjectiveAchievementAssessment（目的達成判定）

役割
- 確定済みの月末資産状況と手取り収入をもとに、目的を実施できるか判定する

算出元
- @TODO
- AssetAccount
- MonthEndAssetBalance
- MonthEndHoldingValue
- AvailableAssetFlag

利用先
- @TODO
- ObjectiveAchievementAssessment

### AssessmentResult（判定結果）

役割
-「達成可能」「達成困難」「判定不可」のいずれかで表される判定結果

算出元
- @TODO

利用先
- @TODO


## 状態

### Confirmed（確定済み）

役割
-対象年月に必要な資産情報がすべて登録され、利用者によって確定された状態

利用先
- @TODO

### Unconfirmed（未確定）

役割
-対象年月の資産情報が不足している、または修正後に再確定されていない状態

利用先
- @TODO


## 値オブジェクト（将来的）

### TargetYearMonth（対象年月）

役割
- 月末資産状況や手取り収入などを管理する年月

利用先
- @TODO
- MonthEndAssetBalance
- MonthEndHoldingValue
- NetIncome
- Objective
- ObjectiveAchievementAssessment


## 属性（エンティティに属する）

### RequiredExpense（必要支出額）

役割
- 目的を達成するために必要となる支出額

利用先
- @TODO

### AvailableAssetFlag（利用可能資産区分）

役割
- 資産口座が目的達成判定の計算対象となるかを表す区分

利用先
- @TODO

### BalanceRecordingUnit（残高記録単位）

役割
- 資産口座の月末残高を、資産口座単位または商品単位のどちらで記録するかを表す区分

利用先
- @TODO
