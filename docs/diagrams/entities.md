## エンティティ

### AssetAccount（資産口座）

役割
- 利用者が資産を保有する場所

関連
- HoldingAsset（保有商品）
- MonthEndAssetBalance（月末資産残高）
- MonthEndAssetSnapshot（月末資産状況）※設計保留
- BalanceRecordingUnit（残高記録単位）
- AssetAccountAvailableSetting（資産口座利用可能設定）

### HoldingAsset（保有商品）

役割
- 資産口座に属し、商品単位で残高を管理する金融商品

関連
- AssetAccount（資産口座）
- MonthEndHoldingValue（商品別月末評価額）
- MonthEndAssetSnapshot（月末資産状況）※設計保留

### MonthEndHoldingValue（商品別月末評価額）

役割
- 保有商品ごとの対象年月の月末時点における評価額

関連
- HoldingAsset（保有商品）
- MonthEndAssetSnapshot（月末資産状況）

### MonthEndAssetBalance（月末資産残高）

役割
- 口座単位で管理する資産口座の対象年月の月末残高

関連
- AssetAccount（資産口座）
- MonthEndAssetSnapshot（月末資産状況）

### MonthEndAssetSnapshot（月末資産状況）

役割
- 対象年月における資産口座および保有商品の残高情報全体を管理する
- 必要な資産情報が揃っているか、および確定状態を管理する

関連
- TargetYearMonth（対象年月）
- MonthEndAssetBalance（月末資産残高）
- MonthEndHoldingValue（商品別月末評価額）
- Confirmed（確定済み）
- Unconfirmed（未確定）

### NetIncome（手取り収入）

役割
- 対象年月の手取り収入

関連
- TargetYearMonth（対象年月）

### Objective（目的）

役割
- 達成可否を判断したいライフイベントまたは高額支出

関連
- RequiredExpense（必要支出額）
- TargetYearMonth（実施予定年月・任意）
- AssessmentHistory（判定履歴）

### AssessmentHistory（判定履歴）

役割
- 目的達成判定に使用した入力値、算出結果および判定結果を、判定時点のスナップショットとして保存する
- 後から目的、資産状況または手取り収入が変更されても、保存内容を変更しない

関連
- Objective（目的）
- TargetYearMonth（判定対象年月）

保持する主な値
- 判定日時
- 目的名
- 実施予定年月
- 必要支出額
- 利用可能資産
- 翌月クレジットカード支払予定額
- 平均手取り収入
- 必要残余資金
- 残余資金
- 判定結果

### AssetAccountAvailableSetting（資産口座利用可能設定）

役割
- 資産口座を利用可能資産として扱うかどうかを、適用期間ごとに管理する
- 対象年月時点で有効な利用可能資産区分を再現できるようにする

関連
- AssetAccount（資産口座）
- AvailableAssetFlag（利用可能資産区分）
- TargetYearMonth（適用開始年月）
- TargetYearMonth（適用終了年月・任意）

## 判定入力

### AvailableAssets（利用可能資産）

役割
- 対象年月時点で、利用可能資産区分が「利用する」の資産を集計した結果

算出元
- MonthEndAssetBalance（月末資産残高）
- MonthEndHoldingValue（商品別月末評価額）
- MonthEndAssetSnapshot（月末資産状況）
- AssetAccountAvailableSetting（資産口座利用可能設定）
- TargetYearMonth（対象年月）

利用先
- RemainingFunds（残余資金）
- ObjectiveAchievementAssessment（目的達成判定）

### NextMonthCreditCardPayment（翌月クレジットカード支払予定額）

役割
- 目的達成判定において、翌月に支払予定のクレジットカード金額

所属
- ObjectiveAchievementAssessment（目的達成判定）

利用先
- RemainingFunds（残余資金）
- AssessmentHistory（判定履歴）

### AverageNetIncome（平均手取り収入）

役割
- 判定対象年月以前の連続する直近3か月の手取り収入から算出した平均額

算出元
- NetIncome（手取り収入）
- TargetYearMonth（対象年月）

利用先
- ObjectiveAchievementAssessment（目的達成判定）

### RemainingFunds（残余資金）

役割
- 目的達成後に残る利用可能資産

算出元
- AvailableAssets（利用可能資産）
- RequiredExpense（必要支出額）
- NextMonthCreditCardPayment（翌月クレジットカード支払予定額）

利用先
- ObjectiveAchievementAssessment（目的達成判定）

### RequiredRemainingFunds（必要残余資金）

役割
-目的達成後も維持すべき生活資金

算出元
- AverageNetIncome（平均手取り収入）

利用先
- ObjectiveAchievementAssessment（目的達成判定）

### ObjectiveAchievementAssessment（目的達成判定）

役割
- 確定済みの月末資産状況と手取り収入をもとに、目的を実施できるか判定する

入力
- RemainingFunds（残余資金）
- RequiredRemainingFunds（必要残余資金）

出力
- AssessmentResult（判定結果）

保存先
- AssessmentHistory（判定履歴）

### AssessmentResult（判定結果）

役割
- 目的達成判定の結果

算出元
- ObjectiveAchievementAssessment（目的達成判定）

利用先
- AssessmentHistory（判定履歴）


## 状態

### Confirmed（確定済み）

役割
- 対象年月に必要な資産情報がすべて登録され、利用者によって確定された状態

対象
- MonthEndAssetSnapshot（月末資産状況）

### Unconfirmed（未確定）

役割
- 対象年月の資産情報が不足している、または修正後に再確定されていない状態

対象
- MonthEndAssetSnapshot（月末資産状況）


## 値オブジェクト（将来的）

### TargetYearMonth（対象年月）

役割
- 年月単位で資産状況や手取り収入の対象期間を表す値

利用先
- MonthEndHoldingValue（商品別月末評価額）
- MonthEndAssetBalance（月末資産残高）
- MonthEndAssetSnapshot（月末資産状況）
- NetIncome（手取り収入）
- Objective（目的）
- AssessmentHistory（判定履歴）


## 属性（エンティティに属する）

### RequiredExpense（必要支出額）

役割
- 目的を達成するために必要となる支出額

所属
- Objective（目的）

利用先
- RemainingFunds（残余資金）

### AvailableAssetFlag（利用可能資産区分）

役割
- 資産口座を利用可能資産の集計対象とするかを表す区分

所属
- AssetAccountAvailableSetting（資産口座利用可能設定）

利用先
- AvailableAssets（利用可能資産）

### BalanceRecordingUnit（残高記録単位）

役割
- 資産口座の残高を口座単位または商品単位のどちらで管理するかを表す区分

所属
- AssetAccount（資産口座）

利用先
- MonthEndAssetBalance（月末資産残高）
- MonthEndHoldingValue（商品別月末評価額）
