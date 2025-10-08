---
type: note
tags:
- '#ai/ref'
- '#design-document'
---
# 歩掛データ機能 Domain層 詳細設計書

## 対応する基本設計書のセクション

参照元: [[🗒️ 工種体系マスタ歩掛データ機能]]

## 実装仕様

### Domain Aggregateの実装詳細

#### WorkUnit Aggregate

**定義**
工種体系における「細別」に対応する集約ルート。細別単位で歩掛データを管理し、現場条件に応じた歩掛計算の窓口となる。

**責務**
- 細別に対する歩掛データの総合管理
- 歩掛計算の統制とエントリーポイント
- 条件項目マスタの管理
- 歩掛データ全体の整合性保証

**構成要素**
- WorkUnitId（集約ID）
- WorkNorm（歩掛アグリゲーション、0..1）
- WorkNormConditionItem配列（条件項目、0..*）

**主要メソッド**
- `setWorkNorm()`: WorkNormエンティティの設定
- `addWorkNormConditionItem()`: 条件項目の追加（重複チェック付き）
- `removeWorkNormConditionItem()`: 条件項目の削除（関連する条件付歩掛も自動削除）
- `calculateWorkNorm()`: 入力条件に基づく歩掛計算
- `validateConditionItems()`: 条件項目の妥当性チェック

**バリデーション責務**
- 条件項目の重複チェック
- 必須条件項目の設定確認
- 条件項目削除時の影響範囲チェック
- 条件項目と条件値の型整合性確認

**不変条件**
- WorkUnitは最大1つのWorkNormを参照できる（0..1の関係）
- WorkUnitに属する条件項目は同一条件名で複数存在してはならない
- WorkUnitが削除される場合、関連するWorkNormも削除される

##### WorkNormConditionItem Aggregation（WorkUnit、WorkNormConditionに属する）

**定義**
歩掛を決定するための条件項目（例：地質、気温、作業環境など）の定義。条件の種類、入力制約、必須設定などを管理する。

**責務**
- 条件項目の定義と管理（条件の"型と値"の統制）
- 条件値の型制約設定とバリデーション
- 条件項目の業務制約とメタデータの管理

**基本属性（基本設計書のデータモデルより）**
- id: WorkNormConditionItemId
- conditionName: ConditionName（条件名）
- conditionKind: ConditionKind（条件の型定義）
- workUnitId: 所属するWorkUnitのID（関係から推定）

**拡張属性（基本設計書のテーブル定義より）**
- conditionKey: 条件の内部キー（condition_item_key）
- isRequired: 必須条件かどうか（condition_item_is_required）
- sortOrder: 条件項目の評価・処理順序（condition_item_sort_order）
- unit: 数値/範囲型条件の単位制約（condition_item_unit）
- textPattern: テキスト型条件の形式制約（condition_item_text_pattern）
- numberMin/Max: 数値型条件の値域制約（condition_item_number_min/max）
- numberMinInclusive/MaxInclusive: 境界値の包含設定
- notes: 条件項目の業務的説明（condition_item_notes）

**主要メソッド**
- `validateConditionValue()`: 条件値の妥当性チェック
- `updateConditionKind()`: 条件型の変更
- `setRequired()`: 必須設定の変更

**制約事項**
- 条件名は同一WorkUnit内で一意
- 条件型変更時は既存の条件値との整合性チェック必須

###### ConditionName ValueObject（WorkNormConditionItemに属する）

**定義**
条件項目の名称（例：「地質」「気温」「作業時間」など）を表すValue Object。ユーザーが理解しやすい日本語の名前。

**制約事項**
- 最大長：100文字
- 空文字・null・未定義は不可
- 前後空白は自動トリム
- 特殊文字の使用可能

###### ConditionKind Interface実装（WorkNormConditionItemに属する）

**定義**
条件項目の型（選択式、数値、範囲、真偽値、文字列）を定義するインターフェース。条件値の入力方法と制約を決定する。

**共通インターフェース**
- `validateValue()`: 条件値の型適合性チェック
- `getDefaultValue()`: デフォルト値の取得
- `getInputConstraints()`: 入力制約の取得

**ChoiceConditionKind**
- 選択肢リストの管理
- 単一選択・複数選択の設定
- 選択肢の追加・削除・変更

**NumberConditionKind**
- 数値範囲の制約設定
- 単位の設定
- 小数点桁数の制限

**RangeConditionKind**
- 範囲値の制約設定
- 境界値の包含設定
- 単位の統一

**TextConditionKind**
- 正規表現パターンの設定
- 最大長の制限
- 改行文字の許可設定

**BooleanConditionKind**
- デフォルト値の設定
- 表示ラベルのカスタマイズ

#### WorkNorm Aggregation

**定義**
細別の歩掛データを管理する集約。歩掛単位数量と基本歩掛を持ち、現場条件に基づいて最終的な歩掛を計算する責務を持つ。

**責務**
- 歩掛計算エンジンとしての中核機能
- 基本歩掛と条件付歩掛の統合管理
- 歩掛データの一貫性保証
- 複雑な条件組み合わせの処理

**属性**
- workUnitId: 所属するWorkUnitのID
- workNormUnitQuantity: 歩掛単位数量
- baseWorkNorms: 基本歩掛のコレクション

**主要メソッド**
- `calculate()`: 条件配列を受け取り最終歩掛を計算
- `updateUnitQuantity()`: 歩掛単位数量の更新
- `addBaseWorkNorm()`: 基本歩掛の追加
- `removeBaseWorkNorm()`: 基本歩掛の削除（関連する条件付歩掛も削除）

**集約としての責務**
- 基本歩掛と条件付歩掛の整合性管理
- 歩掛計算の一意性保証
- 変更時の不変条件維持
- 条件付歩掛追加時の競合検出

**計算ロジック**
1. カテゴリごとに基本歩掛を取得
2. 入力条件に合致する条件付歩掛を優先度順に検索
3. マッチした場合は上書き値または補正率を適用
4. マッチしない場合は基本歩掛を使用
5. 全カテゴリの結果を統合して返却

**不変条件**
- WorkNormUnitQuantityは必須（nullや未設定は不可）
- BaseWorkNormCollectionは0個以上の基本歩掛を含む（空の場合も許可）
- 同一カテゴリの基本歩掛は複数存在してはならない

##### WorkNormUnitQuantity ValueObject（WorkNormに属する）

**定義**
歩掛の基準となる単位数量（例：100m³/日）。この数量を達成するために必要なリソース量が歩掛として定義される。

**属性**
- value: 歩掛単位数量の数値（例：100）
- unit: 歩掛単位数量の単位（例：m³/日、m²/日、m/日）

**制約事項**
- 数値は0.01以上の実数
- 単位は必須（空文字・null不可）
- 単位の前後空白は自動トリム

**バリデーション**
- 負数・0は不可
- 無限大・NaNは不可
- 単位の妥当性チェック（作業量単位/時間単位の形式）

**主要メソッド**
- `equals()`: 同一の歩掛単位数量かの判定
- `toString()`: "100 m³/日" 形式での文字列表現

##### BaseWorkNormCollection EntityCollection（WorkNormに属する）

**定義**
基本歩掛の集合。労務・機械・材料などのカテゴリ別に基本歩掛を管理し、重複チェックや検索機能を提供する。

**責務**
- 基本歩掛の集合管理
- カテゴリ別の基本歩掛検索
- 重複チェック

**主要メソッド**
- `findByCategory()`: カテゴリ別検索
- `add()`: 基本歩掛追加（重複チェック付き）
- `remove()`: 基本歩掛削除
- `validateUniqueness()`: 一意性チェック

#### BaseWorkNorm Entity

**定義**
カテゴリ（労務・機械・材料）別の基準となる歩掛値。条件が設定されていない場合や条件にマッチしない場合に使用される基本の歩掛。

**責務**
- カテゴリ別の基本歩掛値の管理
- 条件付歩掛の管理と競合検出
- 歩掛計算の基準値提供

**属性**
- id: BaseWorkNormId
- workNormCategory: 歩掛カテゴリ（労務/機械/材料）
- workNormMeasure: 歩掛値
- conditionalWorkNorms: 条件付歩掛のコレクション

**主要メソッド**
- `addConditionalWorkNorm()`: 条件付歩掛の追加（重複チェック付き）
- `findMatchingConditionalNorm()`: 条件に合致する条件付歩掛の検索
- `updateMeasure()`: 基本歩掛値の更新

**ビジネスルール**
- 同一カテゴリ内で条件が重複する条件付歩掛は追加不可
- 優先度が同じ場合の条件重複は競合エラー

**競合検出責務**
- 条件の完全一致検出
- 条件の包含関係検出
- 優先度による競合回避の判定

##### WorkNormCategory ValueObject（BaseWorkNormに属する）

**定義**
歩掛のカテゴリ（労務・機械・材料）を表すValue Object。歩掛の種類を分類するための固定値。

**制約事項**
- 許可値：「労務」「機械」「材料」のみ
- 空文字・null・未定義は不可
- 大文字小文字の区別あり
- 将来的な拡張性を考慮した設計

**バリデーション**
- 入力値が許可リストに含まれるかチェック
- 無効値の場合は例外をスロー

##### WorkNormMeasure ValueObject（BaseWorkNormに属する）

**定義**
歩掛の数量と単位（例：2.5人日、1.0台時）を表すValue Object。リソースの必要量を数値と単位で表現する。

**制約事項**
- 数値範囲：0.01 ≤ value ≤ 999
- 小数点以下は2桁まで
- 単位の最大長：10文字
- 単位は必須（空文字・null不可）

**主要機能**
- 補正率の適用（元の値×補正率）
- 複数歩掛の統合計算（業務ロジックに依存）

**バリデーション**
- 0や負数は例外をスロー
- 無効な単位は例外をスロー

##### ConditionalWorkNormCollection EntityCollection（BaseWorkNormに属する）

**定義**
条件付歩掛の集合。特定の条件が満たされた場合に基本歩掛の代わりに使用される歩掛を管理し、競合検出や優先度による選択を行う。

**責務**
- 条件付歩掛の重複検出
- 優先度による条件マッチング
- 競合検出の実装

**重複検出ロジック**
1. 完全一致：全ての条件が完全に一致する場合
2. 包含関係：同一優先度で条件が重複する場合

**競合検出の詳細実装**
- `detectConflict()`: 新規条件付歩掛と既存条件の競合チェック
- `isExactMatch()`: 条件の完全一致判定
- `hasOverlap()`: 条件の重複判定
- 優先度が異なる場合は競合なしと判定

**検索ロジック**
- 優先度の昇順（数値が小さい順）でソート
- 上から順に条件マッチングを実行
- 最初にマッチした条件付歩掛を返却

#### ConditionalWorkNorm Entity

**定義**
特定の条件（例：地質=砂質土、気温≤5℃）が満たされた場合に適用される歩掛。基本歩掛を上書きするか、補正率で調整する。

**責務**
- 条件付きの歩掛値設定と管理
- 条件マッチングの精密な判定
- 条件と歩掛値の整合性保証

**属性**
- id: ConditionalWorkNormId
- workNormCategory: 歩掛カテゴリ
- overriddenWorkNormMeasure: 上書き歩掛値（排他的）
- adjustmentWorkNormRate: 補正率（排他的）
- workNormConditions: 条件のコレクション
- priority: 優先度（1-999、小さいほど高優先度）

**主要メソッド**
- `matches()`: 入力条件配列との適合判定
- `calculateFinalWorkNorm()`: 基本歩掛に対する最終歩掛の計算
- `validateConditions()`: 条件の妥当性チェック

**バリデーション責務**
- 条件値と条件項目の型整合性チェック
- 必須条件の設定確認
- 条件値の制約範囲チェック

**制約事項**
- overriddenWorkNormMeasureとadjustmentWorkNormRateは排他的（どちらか一方のみ設定可能）
- 全ての条件がAND結合で評価される
- 優先度は必須で、1-999の範囲

##### Priority ValueObject（ConditionalWorkNormに属する）

**定義**
条件付歩掛の優先度。複数の条件がマッチした場合、数値が小さい（高優先度）の条件が適用される。

**制約事項**
- 範囲：1 ≤ value ≤ 999（1が最高優先度）
- 整数のみ許可
- 0や負数は明示的に禁止

**比較メソッド**
- `isHigherThan()`: 数値の小さい方が高優先度
- `equals()`: 同一優先度の判定

##### OverriddenWorkNormMeasure ValueObject（ConditionalWorkNormに属する）

**定義**
条件マッチ時に基本歩掛を完全に置き換えるための歩掛値。補正率ではなく、新しい絶対値で上書きする。

**制約事項**
- WorkNormMeasureと同様の制約
- 上書き専用の意味論を持つ

##### AdjustmentWorkNormRate ValueObject（ConditionalWorkNormに属する）

**定義**
条件マッチ時に基本歩掛に乗算する補正率（例：1.2倍、0.8倍）。基本歩掛に対して相対的な調整を行う。

**制約事項**
- 範囲：0.01 ≤ value ≤ 10.0（1%〜1000%）
- 小数点以下は2桁まで
- 負数・0は不可

##### WorkNormConditionCollection EntityCollection（ConditionalWorkNormに属する）

**責務**
- 条件の組み合わせ管理
- 全条件のAND評価
- 条件マッチングの実行

**主要メソッド**
- `allMatch()`: 入力条件配列との全件一致判定
- `equals()`: 他のコレクションとの完全一致判定
- `overlaps()`: 他のコレクションとの重複判定

#### WorkNormCondition Entity

**責務**
- 個別条件の管理
- 条件項目との参照関係
- 条件値の妥当性保証

**属性**
- id: WorkNormConditionId
- workNormConditionItem: 条件項目への参照
- conditionValue: 条件値

**主要メソッド**
- `matches()`: 入力値とのマッチング判定
- `validateValue()`: 条件値の妥当性チェック

##### ConditionValue Interface実装（WorkNormConditionに属する）

**共通インターフェース**
- `type`: 条件タイプの識別子
- `matches()`: 他の条件値との適合判定
- `toJson()`: JSON形式での出力

**ChoiceConditionValue**
定義：プルダウンやラジオボタンで選択される項目の値（例：地質の「砂質土」「粘土」）。
- choiceOptionId: 選択された選択肢のID（CHOICE_OPTIONSテーブルへの参照）
- textValue: 実際の入力値・表示値（CONDITION_VALUESテーブルのcondition_value_textに格納）
- 空文字・null・未定義は不可
- 前後空白は自動トリム
- textValueの最大長：制限なし（基本設計書による）

**基本設計書との対応**
- `condition_value_choice_option_id`: choiceOptionIdに対応
- `condition_value_text`: textValueに対応
- 選択肢定義はCHOICE_OPTIONSテーブルで管理、実際の値はCONDITION_VALUESテーブルのtextカラムに格納

**NumberConditionValue**
定義：数値で入力される条件の値（例：気温「25℃」、深度「3.5m」）。
- 数値と単位（オプション）を保持
- 数値範囲：-999 ≤ value ≤ 999
- 単位の最大長：10文字
- NaN・Infinity・-Infinityは不可

**RangeConditionValue**
定義：範囲で指定される条件の値（例：気温「0℃～5℃」、水深「2.0m～5.0m」）。
- 最小値・最大値・境界値の包含設定・単位を保持
- 最小値と最大値の差は0.01以上必要
- 境界値の包含設定は必須
- 開区間のみの設定は不可
- 数値の妥当性チェック
- 他の数値・範囲との重複判定機能

**BooleanConditionValue**
定義：真偽値で表される条件の値（例：雨天作業「true/false」、夜間作業「true/false」）。
- true/falseのみ保持
- 文字列での「true」「false」は受け付けない
- バリデーション不要

**TextConditionValue**
定義：自由テキストで入力される条件の値（例：施工業者名、特殊条件の詳細）。
- テキスト値と正規表現パターン（オプション）を保持
- 最大長：500文字
- 正規表現パターンの最大長：200文字
- 改行文字の使用可能
- HTMLタグは自動エスケープ
- 空文字は不可、前後空白は自動トリム
- パターンが設定されている場合は値との適合性をチェック



## エラー処理

### カスタム例外の設計指針

**基底クラス**
- 既存の共通例外基底クラスを継承
- エラーコードとメッセージを保持
- Domain層固有の例外として識別可能

**具体的な例外クラス**
1. `InvalidWorkNormCategoryError`: 無効な歩掛カテゴリ
2. `InvalidWorkNormMeasureError`: 無効な歩掛値
3. `InvalidPriorityError`: 無効な優先度
4. `ConditionalWorkNormConflictError`: 条件付歩掛の競合
5. `InvalidConditionValueError`: 無効な条件値

**エラーメッセージの設計**
- ユーザーが理解しやすい日本語メッセージ
- 修正方法を示唆する内容
- 技術的詳細は含めない

## テスト観点

### ユニットテスト対象

**ValueObject**
- 境界値テスト（最小値・最大値・範囲外）
- 無効値の拒否
- 等価性の判定
- JSON変換の正確性

**Entity**
- ビジネスルールの検証
- 状態変更の正確性
- 不変条件の維持
- メソッドの副作用

**Collection**
- 要素の追加・削除・検索
- 重複検出の正確性
- ソートの正確性
- パフォーマンス

**Domain Service**
- 複雑なバリデーションロジック
- 競合検出の網羅性
- エラーメッセージの適切性

### 統合テストで確認すべき項目

1. **複雑な条件での歩掛計算の正確性**
2. **優先度による条件選択の正確性**
3. **大量データでのパフォーマンス**
4. **エラーハンドリングの一貫性**
5. **不変条件の維持**

### テストデータの設計

**境界値ケース**
- 優先度の最小値・最大値
- 歩掛値の0・最大実用値
- 条件数の最小・最大

**異常ケース**
- 必須条件の未設定
- 条件の競合
- 無効な条件値
- 存在しない参照

## 実装上の注意点

### パフォーマンス考慮事項

**条件マッチングの最適化**
- 条件数が多い場合の計算量を考慮
- 早期リターンによる無駄な処理の回避
- インデックス化可能な条件の識別

**メモリ使用量の最適化**
- Immutableオブジェクトの適切な使用
- 大量の条件付歩掛を扱う際の効率化
- 不要なオブジェクト生成の回避

**バリデーション処理の最適化**
- 重い検証処理の分離
- キャッシュ可能な検証結果の活用
- 段階的なバリデーション

### 既知の制限事項

**スケーラビリティ**
- 条件項目数の実用的上限：50項目程度
- 条件付歩掛数の実用的上限：1000件程度
- 同時計算処理の限界

**精度の問題**
- 浮動小数点計算の丸め誤差
- 補正率の累積誤差
- 必要に応じてDecimal型の検討

**複雑性の管理**
- 条件の組み合わせ爆発への対策
- UI側での適切な制限設定
- 運用ガイドラインの整備

## 他Projectとの依存関係

### Infrastructure層への接続仕様

**WorkUnitRepository**
- WorkUnit集約の完全な永続化・復元
- 楽観的ロック制御
- カスケード削除の実装

**WorkNormConditionItemRepository**
- 条件項目マスタの管理
- WorkUnit単位での一括取得
- キャッシュ戦略の適用

**ConditionItemPresetRepository**
- プリセットの CRUD操作
- テナント分離
- 検索・フィルタリング機能

### Application層への依存関係

**WorkNormApplicationService**
- トランザクション境界の管理
- ドメインイベントの発行
- 外部システムとの連携

**WorkNormQueryService**
- 複雑な検索クエリの実行
- レポート用データの集計
- パフォーマンス最適化

### 実装順序の推奨

**Phase1: 基盤の実装**
1. ValueObjectの実装とテスト
2. 基本的なEntityの実装
3. Collection の基本機能

**Phase2: ビジネスロジック**
1. 条件マッチングロジック
2. 歩掛計算ロジック
3. バリデーションロジック

**Phase3: 高度な機能**
1. 競合検出機能
2. Domain Service
3. パフォーマンス最適化

**Phase4: 統合・検証**
1. 統合テスト
2. パフォーマンステスト
3. 本番環境での検証

### インターフェース定義方針

**Repository Interface**
- 既存のRepository基底クラスを継承
- 標準的なCRUD操作を実装
- Domain固有の検索メソッドを追加

**Factory Interface**
- ConditionValueの型別生成
- JSONからの復元機能
- バリデーション付きの生成

**Event Interface**
- ドメインイベントの定義
- 歩掛変更時の通知
- 他の境界コンテキストとの連携

## ValueObjectの制約事項詳細

### 共通制約事項

**null安全性**
- 全てのValueObjectはnull・undefinedを受け付けない
- 空文字列は明示的に禁止されているもの以外は許可
- オプショナルな値はnullではなく専用の「未設定」ValueObjectを使用

**イミュータビリティ**
- 全ての値は作成後変更不可
- 変更が必要な場合は新しいインスタンスを生成
- 内部状態の直接アクセスは禁止

**等価性**
- 同一型・同一値の場合は等価
- 異なる型との比較はfalse
- ハッシュ値の一貫性を保証

### 個別制約事項

**WorkNormCategory**
- 許可値：「労務」「機械」「材料」
- 将来的な拡張性を考慮した設計
- 国際化対応は当面考慮しない

**WorkNormMeasure**
- 数値範囲：0 ≤ value ≤ 999
- 小数点以下は2桁まで
- 単位の最大長：10文字

**Priority**
- 範囲：1 ≤ value ≤ 999
- 整数のみ
- 0や負数は明示的に禁止

**ChoiceConditionValue**
- 選択肢IDの最大長：50文字
- 表示値の最大長：100文字
- 特殊文字の使用可能

**NumberConditionValue**
- 数値範囲：-999 ≤ value ≤ 999
- 単位の最大長：10文字
- 無限大・NaNは明示的に禁止

**RangeConditionValue**
- 最小値と最大値の差は0.01以上必要
- 境界値の包含設定は必須
- 開区間のみの設定は不可

**BooleanConditionValue**
- true/falseのみ
- 文字列での「true」「false」は受け付けない

**TextConditionValue**
- 最大長：500文字
- 正規表現パターンの最大長：200文字
- 改行文字の使用可能
- HTMLタグは自動エスケープ