---
type: note
tags:
- '#ai/ref'
---
# Playwright×LLM 適応型 UI 自動化 — Phase 3: 運用拡張と生成

Phase 2 までで得た堅牢な実行基盤を、運用品質（可観測性・セキュリティ・拡張性）と開発体験（コード自動生成/再利用）まで引き上げる。外部連携と長期運用を前提にした最終段階の設計をまとめる。

- 参照: [[🗒️ 000024-playwright-llm-adaptive-automation-adr.md]]（「拡張機能」「ターゲットサイトリスト」「sitemap 取得」）
- 参照: [[🗒️ playwright-llm-detailed-design.md]]（履歴・コード生成・非機能章）
- 前段: [[🗒️ playwright-llm-phase1-基盤MVP.md]], [[🗒️ playwright-llm-phase2-堅牢化と複雑UI対応.md]]

## 目的・スコープ

- 目的: 実運用に耐える可観測性と安全性を整備し、学習・再利用を通じてコスト/時間を最適化する。
- スコープ:
  - 履歴（ヒストリー）システム: サイトごとの成功フロー・失敗要因を蓄積、90 日で自動削除
  - サイトプロファイル外部連携: 実行時に`getTargetSites()`で取得、ゼロでも動作継続
  - 動的 sitemap 取得/キャッシュ: `robots.txt`→`sitemap.xml`、3s/2MB 制限、ETag/Last-Modified 対応
  - 自然言語インテント解析の拡張: 単一文からサイト特定 → アクション分解 → 条件抽出 → ワークフロー生成
  - コード自動生成: 実行ログから Playwright TS コード生成・保存（`generated/`）
  - セキュリティ/運用: 認証情報管理、実行モード切替、監視/アラート、マスキング強化

## 履歴システム（History）

- 目的: 同一サイトでの成功/失敗から継続学習し、2 回目以降を高速化・高成功率化。
- 内容:
  - 成功シーケンスのテンプレ化（次回の初期候補）
  - 失敗パターンの原因集計とヒント提示
  - 90 日以上は自動削除（サイズ肥大防止）

## サイトプロファイル/動的 sitemap

- `getTargetSites(ctx): Promise<SiteProfile[]>` を外部依存として定義（失敗時は空配列）。
- 実行時に未訪問ドメインは`sitemap`を動的取得して LLM プロンプトへ URL 列挙を注入（最大 300URL または 10,000 文字）。
- キャッシュはインメモリと`/tmp`（永続ストア不使用）。

## 自然言語インテント解析（拡張）

- 単文から「サイト特定 → アクション分解 → 条件抽出 → ワークフロー化」まで自動化。
- 既存のチェックリスト化器を拡張し、サイト固有の用語をプロファイルで補強。

## コード自動生成

- 目的: ログ（JSONL）から実行済み手順を TS コードとして再現可能にし、回帰テスト資産を作る。
- 出力: `generated/{sessionId}/spec.ts`
- 方針: 高レベル API（`getByRole`中心）を優先し、コメントに元ログの根拠（confidence/rationale）を残す。

## セキュリティ/運用

- 認証情報管理: 環境変数/OS キーチェーン/暗号化ローカルストレージ（選択）
- 実行モード: `development`（DevTools、slowMo）/`production`（headless, ERROR 以上のログ）
- 監視/アラート: 失敗率・回復率・平均実行時間を可視化、閾値超過で通知
- マスキング強化: PII/セッション/トークンは保存前に強制置換、スクショは設定で無効化可能

## 非機能 KPI（最終）

- 回復率（エマージェンシー適用時）: 70%以上
- 自然言語入力の正解解釈率: 90%以上
- 2 回目以降の実行時間: 50%以上短縮（履歴活用）
- 開発者の手動介入: 80%削減

## 実装タスク（ToDo）

- `src/services/history.ts`: 成功/失敗の永続化、統計、TTL 削除
- `src/services/site-config.ts`: `getTargetSites()`の実装ポイント（TBD 外部）
- `src/services/sitemap-loader.ts`: robots/sitemap 取得・キャッシュ
- `src/generators/code-generator.ts`: ログ →TS コード生成
- `src/services/auth-manager.ts`: 秘匿情報の取得/保管
- `src/services/metrics.ts`: 指標収集としきい値アラート

## 参照

- [[🗒️ 000024-playwright-llm-adaptive-automation-adr.md]]
- [[🗒️ playwright-llm-detailed-design.md]]
- [[🗒️ playwright-llm-phase1-基盤MVP.md]]
- [[🗒️ playwright-llm-phase2-堅牢化と複雑UI対応.md]]
