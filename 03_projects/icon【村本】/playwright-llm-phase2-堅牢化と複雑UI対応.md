---
type: note
moc: "[[🏷️基本設計書]]"
tags:
- '#ai/ref'
---
# Playwright×LLM 適応型 UI 自動化 — Phase 2: 堅牢化と複雑UI対応

Phase 1のMVPを、現実のWebアプリで壊れにくく実用可能な水準へ引き上げる。セレクタ戦略の拡張、仮想リスト等の複雑UIへの対応、緊急時の拡張入力（軽量DOM）の導入、失敗時の詳細ログ設計を行う。

- 参照: [[🗒️ 000024-playwright-llm-adaptive-automation-adr.md]]（特に「複雑UI対応」「拡張機能」「信頼度」）
- 参照: [[🗒️ playwright-llm-detailed-design.md]]（該当各章の実装案）
- 前段: [[🗒️ playwright-llm-phase1-基盤MVP.md]]

## 追加の目的・スコープ

- 目的: UI変化や要素不在に強いフォールバックと、複雑UI（仮想リスト/Shadow DOM）の基本対応を実装する。
- スコープ:
  - セレクタ戦略の拡張（`getByLabel`/`getByPlaceholder`/`locator`/近接関係の活用は段階導入）
  - エマージェンシーモード（失敗が続く/停滞時にLLM・入力情報を拡張）
  - 軽量DOM（body配下、CSS/JS除去）の一時投入
  - 仮想リスト探索の実装（段階スクロール+ループ防止）
  - Shadow DOM（open）探索の導入
  - 信頼度（confidence）と根拠（rationale）の運用
  - 失敗時ログ（ExecutionError）の拡充とマスキング規約
  - シークレット管理: SecretStorePortをAWS Secrets Managerへ接続する公式アダプタの提供と設定フロー整備

## Step モデルの登り方（Phase 1 → Phase 2）

- Phase 1.1 時点で Step メタは `id`/`originalInstruction`/`attempt` のみに削ぎ落とされ、intent/target/value などのヒューリスティック推定は廃止済み。Phase 2 では新たな推定ロジックを Step 生成側に足さず、LLM の応答・検証結果からメタ情報を学習的に蓄積する方向に舵を切る。
  - 追加シグナル（軽量 DOM・履歴・ARIA 差分）は `stepHistory` の `metadataKeys` などとして記録し、LLM に与えるかは別途プロンプト設計で検討する。
  - intent `"observe"`/`"scroll"` 等の拡張は LLM 応答の表現に一本化し、Step メタはあくまで履歴タグとして扱う。
- `ActionDecisionInput.currentStepMeta` は optional。空の場合はプロンプトで { "error": "TARGET_NOT_FOUND" } を明示的に返すよう促し、フォールバックヒューリスティックには戻らない。
- `stepHistory` は `metadataPresent`/`metadataMissing`/`metadataKeys` を保持し、Phase 2 ではここに LLM が返した補足情報（例: 採用したセレクタの種類や根拠）を追記して診断に活用する。

除外（Phase 3以降）: 履歴システム、自然言語インテント解析の高度化、コード自動生成、サイトプロファイル外部連携、監視/アラート運用整備等。

## セレクタ戦略（拡張）

- 通常時（ariaモード）:
  1. `getByRole(name)`
  2. `getByText`
  3. `getByTestId`（整合時のみ）

- 緊急時（domモード; 軽量DOM投入時）:
  1. `getByRole(name)`
  2. `getByLabel`
  3. `getByPlaceholder`
  4. `getByText`（正規化/近似を許容）
  5. `locator(css)`
  6. 近接/祖先（`:has`, `label:has-text()` 等）

`resolveCandidateCount`で0/複数件はLLMに戻さず即フォールバック（失敗カウント）。

## エマージェンシーモード

- 発動条件（例）:
  - 連続3回失敗、または30秒以上の停滞
  - もしくは直近5件の平均`confidence < 0.3`
- 動作:
  - 小規模→大規模LLMへ一時切替（コストは最小限）
  - 入力を`Accessibility Tree`→`軽量DOM`へ拡張（body配下; CSS/JS除去）
- 解除:
  - ステップ成功、または連続2回の高`confidence`で通常モードへ復帰

## 複雑UI対応

### 仮想リスト

- 検出条件例:
  - `overflow-y: auto|scroll` かつ `scrollHeight/clientHeight >= 2.0`
  - あるいは `role in [list, listbox, grid, table, tree]`
- 探索規約:
  - スクロール刻み: `0.8 * clientHeight`
  - 試行上限: 10回
  - ループ防止: 直近スクロール位置の差分<5pxが2回続いたら中断
- ログ:
  - `containerSelector`, `clientHeight`, `scrollHeight`, `ratio`, `attemptCount`, `positions[]`

### Shadow DOM

- open Shadowのみ対応。通常セレクタ→Shadow内探索の順で試行。

### Canvas

- 非対応（`CANVAS_UNSUPPORTED`即時返却）。

## 信頼度の扱い

- LLMに数値`0.00–1.00`と1行の根拠を返させる。
- 直近5件の移動平均<0.3が継続で、エマージェンシー判定材料とする。
- `resolveCandidateCount`で弾かれた試行は`confidence=0`として窓に記録。

## 失敗時ログ仕様（ExecutionError）

- 分類例: `ELEMENT_NOT_FOUND`/`TIMEOUT`/`NAVIGATION_FAILED`/`ASSERTION_FAILED`/`CANVAS_UNSUPPORTED`
- 保存場所: `logs/{sessionId}/errors/step-{step}-{ts}.json`
- スクリーンショット: `logs/{sessionId}/screenshots/error-{ts}.png`（パス参照推奨）
- マスキング: email/電話/カード/トークン等は保存前に`****`へ置換

## 非機能要件（強化）

- リトライとフォールバックでステップ成功率をPhase 1比+20%以上
- エマージェンシー適用時の回復率70%以上（目標）
- ログと指標でデバッグ時間を有意に削減（目安50%）

## 実装タスク（ToDo）

- `src/core/emergency.ts`: エマージェンシー判定/切替
- `src/core/executor.ts`: domモードの追加（軽量DOM時のみ拡張セレクタを許可）
- `src/core/virtual-list.ts`: 仮想リスト探索実装
- `src/core/shadow.ts`: Shadow DOM探索ユーティリティ
- `src/core/confidence.ts`: 信頼度算出/移動平均
- `src/services/errors.ts`: ExecutionErrorデータモデル/保存/マスク
- `src/services/screenshot.ts`: 失敗時スクショ保存
- `src/adapters/secrets/secretsManager.ts`: AWS Secrets Managerクライアント実装（取得/リリース）と設定ドキュメント

## 参照

- [[🗒️ 000024-playwright-llm-adaptive-automation-adr.md]]
- [[🗒️ playwright-llm-detailed-design.md]]
- [[🗒️ playwright-llm-phase1-基盤MVP.md]]
