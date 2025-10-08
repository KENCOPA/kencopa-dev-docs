---
type: note
tags:
- '#ai/ref'
---
# AIツールのrules管理方法

## 概要

本プロジェクトでは、複数のAIツール（Claude Code、Cursor、Gemini CLI）における開発ルールの統一管理を実現するため、rulesyncライブラリを活用した新しいアプローチを実装した。Design DocsとRulesの役割を明確に分離し、正確な実装情報の提供と設計思想の参照を両立するシステムを構築した。

## 背景・なぜこの変更が必要なのか

### 解決すべき課題

1. **設定の重複管理**
   - 複数のAIツールでそれぞれ独自のrules設定フォーマット
   - 同じ内容を何度も記述する必要があった
   - 各ツールでの設定ファイル更新の手間

2. **情報の性質による混在**
   - 正確な実装情報（Rules）と設計思想（Design Docs）が混在
   - 時間経過による実装と設計ドキュメントの乖離
   - AIツールが求める「正確な実装情報」の不足

3. **品質のばらつき**
   - AIツールの回答品質が一定しない
   - 「こういう時にどこをいじれば良いか」が明確でない
   - プロジェクト固有の実装パターンが伝わらない

### ビジネス価値

- **開発効率向上**: AIツールの回答品質向上によるコードレビュー時間削減
- **品質向上**: 一貫したコーディング規約の適用とドメイン知識の正確な伝達
- **保守性向上**: ルールの一元管理と複数AIツール間での一貫性確保

## 設計方針

### 1. Design DocsとRulesの役割分離

**Design Docs（Obsidian）**:
- 実装の**方針**と**設計思想**を記述
- 新規モジュール作成時の指針
- 時間経過で詳細はずれるが、大まかな方針は維持
- 「なぜそうするか」「どのような背景か」

**Rules（rulesync）**:
- **正確な実装情報**を記述
- 現在のコードベースと完全に一致
- 変更があれば都度更新が必要
- 「具体的にどう実装するか」「現在どうなっているか」

### 2. 情報の使い分け戦略

```markdown
# rulesファイルでの記述例

## 現在の実装パターン（正確な情報）
- **新しいエンティティ**: `src/domains/{entity}/` 配下に集約を作成
- **APIエンドポイント**: `src/routes/v1/protected/{userContext|tenantContext}/` 配下
- **認可ロジック**: `src/authorizers/` 配下のauthorizer実装

## 設計思想・背景の参照
詳細な設計思想については以下を参照：
- [[🗒️ BEディレクトリ構造方針]]: アーキテクチャの設計思想
- [[🗒️ マルチテナントシステム設計]]: エンティティ設計の背景
```

### 3. フィードバックループによる正確性維持

**変更検知 → 即座更新**:
- 実装パターンの変更を検知
- rulesファイルの更新を即座に提案
- Design Docsは大きな方針変更時のみ更新

**検知対象**:
- 実装パターンの変更
- Shared機能の追加（`src/**/shared/**/*.ts`）
- 認可システムの変更
- API設計の変更
- データベース設計の変更

## 実装方針

### 1. 情報の配置戦略

**rulesファイルに直接記述**:
- 現在のディレクトリ構造
- 具体的なファイル配置場所
- 実装時の具体的な手順
- コーディング規約の詳細
- 現在使用中のValueObject一覧

**Design Docsを参照**:
- 設計判断の背景・理由
- アーキテクチャの全体像
- 代替案との比較検討
- 将来の拡張方針

### 2. rulesyncライブラリの活用
**選択理由**:
- 統一的なMarkdownフォーマットで複数AIツールに対応
- Obsidianリンクがそのまま保持される（参照用）
- 対象ツール（claudecode, cursor, geminicli）のみに絞り込み可能

**役割ベースのファイル分割**:
```
.rulesync/
├── root.md (root: true)                    # 常に参照される基本情報
├── domains.md                              # Domain層の実装ルール
├── routes.md                               # API設計の実装ルール
├── database.md                             # データベース・Prismaの実装ルール
├── unit-tests.md                           # テストコードの実装規約
├── usecases.md                            # Usecase層の実装ルール
├── infrastructures.md                     # Infrastructure層の実装ルール
├── scripts.md                             # スクリプトの実装ルール
└── architecture-change-detector.md        # 変更検知とフィードバックループ
```

refs:
```cardlink
url: https://github.com/dyoshikawa/rulesync
title: "GitHub - dyoshikawa/rulesync"
description: "Contribute to dyoshikawa/rulesync development by creating an account on GitHub."
host: github.com
favicon: https://github.githubassets.com/favicons/favicon.svg
image: https://opengraph.githubassets.com/5e8d505ba823f488dd430fec5952d28ef87b4a60b435c506395ec8dbd36ea837/dyoshikawa/rulesync
```

```cardlink
url: https://dev.classmethod.jp/articles/rulesync/
title: "rulesync: Claude CodeやCursor、Clineのrulesを統一管理するツールを公開しました | DevelopersIO"
host: dev.classmethod.jp
favicon: https://dev.classmethod.jp/favicon.ico
image: https://images.ctfassets.net/ct0aopd36mqt/wp-thumbnail-75c24212db08a605cf51b1578f9a040c/f14b12b1b1cf9b5ed2427490ff86734d/eyecatch_anthropic
```

### 3. 継続的な正確性維持

**自動化フロー**:
1. **変更検知**: アーキテクチャ関連ファイルの変更を検知
2. **即座提案**: rulesファイルの更新を提案
3. **フォーマット選択**: 必要に応じてDesign Doc/ADRも作成
4. **反映**: `npx rulesync generate` で全AIツールに配布

**更新の粒度**:
- **Rules**: 実装変更のたびに更新
- **Design Docs**: 設計方針変更時のみ更新

## 影響範囲

### 開発フロー
- **実装時**: rulesファイルで正確な手順を確認
- **設計判断時**: Design Docsで背景・方針を確認
- **変更時**: フィードバックループで即座にrules更新

### 情報の鮮度管理
- **Rules**: 常に最新の実装と一致
- **Design Docs**: 大まかな方針レベルで一致
- **変更検知**: 実装変更を即座にキャッチ

## 代替案と選択理由

### 代替案1: すべてをDesign Docsで管理
**却下理由**:
- 時間経過による実装との乖離が inevitable
- AIツールが求める「正確な実装情報」を提供できない
- 詳細な実装手順の記述に不向き

### 代替案2: すべてをRulesで管理
**却下理由**:
- 設計思想・背景情報の記述に不向き
- ファイルサイズが膨大になり管理困難
- 「なぜそうするか」の情報が欠落

### 代替案3: 完全に分離して連携なし
**却下理由**:
- 情報の断絶により全体像の把握が困難
- 設計思想と実装の整合性確認ができない
- 新規参画メンバーの理解が困難

### 採用案: 役割分離と適切な参照
**選択理由**:
- それぞれの情報の性質に適した管理方法
- 正確性と設計思想の両立
- フィードバックループによる継続的な整合性維持

## 技術的なエッセンス

### 1. 情報アーキテクチャの設計
```
正確な実装情報: rulesync files (即座更新)
       ↓
設計思想・背景: Obsidian Documents (参照)
       ↓
統合配布: AI Tool configs
```

### 2. フィードバックループ
```
実装変更 → 変更検知 → rules更新提案
       ↓
即座更新 → rulesync再生成 → AIツール反映
```

### 3. 情報の使い分け
```
「どう実装するか」→ rulesファイル
「なぜそうするか」→ Design Docs参照
「いつ更新するか」→ 実装変更の都度 vs 方針変更時
```

## 成功要因と学び

### 成功要因
1. **情報の性質理解**: RulesとDesign Docsの役割を明確に分離
2. **フィードバックループ**: 実装変更を即座にrulesに反映
3. **段階的検証**: MCPやrulesyncの動作を詳細に検証

### 重要な学び
1. **正確性の維持**: AIツールには「正確な実装情報」が最重要
2. **役割の分離**: 「方針」と「実装」は管理方法を分ける必要
3. **継続的更新**: 実装変更のたびにrulesを更新するプロセスが不可欠

### 注意点
1. **更新頻度の違い**: Rulesは頻繁、Design Docsは稀
2. **情報の整合性**: 参照関係の維持に注意が必要
3. **チーム運用**: 新しいプロセスの浸透に時間が必要

この取り組みにより、「正確な実装情報の提供」と「設計思想の参照」を両立し、AIツールを活用した効率的で高品質な開発体制を構築した。