# KCDevDocuments → KenKopaDevDocuments 移行レポート

## 実施日時
2025-10-08

## 移行サマリー

### 総対象ファイル数
- **総ファイル数**: 151個
- **移行完了**: 110ファイル
- **除外（MOC）**: 33ファイル
- **スキップ（既存）**: 8ファイル

### カテゴリ別内訳

#### 03_projects/ (50ファイル)
プロダクト固有の設計文書
- Kunai関連設計文書: 約30ファイル
- Azuchi関連設計文書: 約15ファイル
- インフラ・アーキテクチャ設計: 5ファイル

#### 04_standards/ (51ファイル)
組織標準・ガイドライン・ADR
- ADR (000XXX-形式): 25ファイル
- 運用ガイドライン: 24ファイル
  - Devin, Linear, Obsidian, MOC, Tag, branch, v0等の利用ガイドライン
  - オンボーディング、リリース手順等
- その他標準文書: 2ファイル

#### 05_research/ (9ファイル)
技術調査・検討事項
- 検討事項(💭): 3ファイル
  - BEユニットテスト方針
  - FE-BEエラーハンドリング方法
  - ソフトウェアテスト戦略
- AI・RAG関連: 3ファイル
  - ai-agent & chat関連文書
  - Prompt Management with Langfuse
- 歩掛データ機能: 2ファイル
- その他: 1ファイル

## 除外したファイル（MOC: 33ファイル）

以下のMOCファイルは移行対象外として除外しました（🏷️プレフィックス、type: moc）:

1. 🏷️ ADR.md
2. 🏷️ azuchi-backend.md
3. 🏷️ azuchi-chat.md
4. 🏷️ azuchi-common-infra.md
5. 🏷️ azuchi-frontend.md
6. 🏷️ azuchi-infra-v2.md
7. 🏷️ azuchi-safegen-api.md
8. 🏷️ design-document.md
9. 🏷️ guideline.md
10. 🏷️ kencopa-ai.md
11. 🏷️ kunai-core-backend.md
12. 🏷️ kunai-core-frontend.md
13. 🏷️ kunai-infra.md
14. 🏷️ procedure-manual.md
15. 🏷️ product kunai.md
16. 🏷️ 作業中.md
17. 🏷️ 検討事項.md
18. その他MOCタグファイル

## スキップしたファイル（既存: 8ファイル）

以下のファイルは既にKenKopaDevDocsに存在するためスキップしました:

1. **AOA工程表機能.md**
   - 既存: 03_projects/kunai/概念設計/AI工程表エディター/AOA工程.md

2. **README-template.md**
   - 既存: 同名ファイルが存在

3. **土木工事積算基準歩掛り分析.md**
   - 既存: 05_research/建設業/土木工事積算基準歩掛り分析.md

4. **土木工程表生成に必要な変数の整理.md**
   - 既存: 05_research/建設業/土木工程表生成に必要な変数の整理.md

5. **建築工程表生成に必要な変数の整理.md**
   - 既存: 05_research/建設業/建築工程表生成に必要な変数の整理.md

6. **💭 AOA工程表 SVG vs Canvas パフォーマンス実測分析.md**
   - 既存: 類似ドキュメントが存在

7. **🗒️ 000001-AIエージェントとチャット機能をマイクロサービス化.md**
   - 既存: 類似ADRが存在

8. **🗒️ ユーザー招待機能.md**
   - 既存: 類似設計文書が存在

## 変更内容

### メタデータの削減
移行時に以下の変更を実施しました:

**変更前の例**:
```yaml
---
type: note
tags:
  - Devin
moc:
  - "[[🏷️ guideline]]"
下書き: true
提案済み: true
承認済み: true
代替済み: 
非推奨: 
棄却:
---
```

**変更後の例**:
```yaml
---
type: note
tags:
  - Devin
  - guideline
---
```

### ファイル名の変更
- 絵文字プレフィックス(🗒️, 💭)を削除
- 例: `🗒️ Devin利用ガイドライン.md` → `Devin利用ガイドライン.md`

### 保持した内容
- 本文内容は完全に無変更
- 重要なタグ情報は保持
- ドキュメントタイプ(type)は保持

## 分類ロジック

### 04_standards/ に分類
- ADRファイル（🗒️ 000XXX-で始まる）
- ガイドライン・運用ルール（moc: guideline, procedure-manual）
- コーディング規約・命名規則
- 開発プロセス・ツール利用手順

### 05_research/ に分類
- 💭（検討事項）ファイル
- 技術調査・リサーチ文書
- 建設業ドメイン知識（歩掛データ等）
- AI・RAG関連の調査文書

### 03_projects/ に分類
- プロダクト固有の設計文書（kunai, azuchi関連）
- 機能設計・アーキテクチャ図
- 実装仕様書
- その他のdesign-document

## 検証結果

### ファイル数確認
- ✅ 03_projects/: 50ファイル
- ✅ 04_standards/: 51ファイル
- ✅ 05_research/: 9ファイル
- ✅ 合計: 110ファイル（予定通り）

### 品質確認
- ✅ 全ファイルの本文内容は無変更
- ✅ メタデータは最小限に削減
- ✅ KCDevDocuments側は一切変更なし
- ✅ 絵文字プレフィックスは全て削除

## 備考

### 今後の対応
- 移行したドキュメント間のリンク切れチェック（必要に応じて）
- KenKopaDevDocs独自のMOC構造への統合（別タスク）
- 重複コンテンツの統合検討（スキップした8ファイル）

### 移行元リポジトリ
- リポジトリ: KENCOPA/kc-dev-documents
- パス: 02. resources/
- 状態: 無変更（読み取り専用として使用）
