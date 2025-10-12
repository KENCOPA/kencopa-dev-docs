---
type: note
moc: "[[🏷️基本設計書]]"
tags:
- '#ai/ref'
---
# Playwright×LLM 適応型 UI 自動化 — Phase 1: 基盤 MVP

本ドキュメントは、以下の原典をもとに最小構成でエンドツーエンドに動く MVP を定義する。

- 参照: [[🗒️ 000024-playwright-llm-adaptive-automation-adr.md]]（ADR）
- 参照: [[🗒️ playwright-llm-detailed-design.md]]（詳細設計）

以降の Phase 2/3 の記述は別ドキュメントに分割した。既存ドキュメントに書かれているフェーズ表現には依存せず、新しい 3 段階で段階的に完成度を高める。

## 目的・スコープ

- 目的: Accessibility Tree ベースで、小規模 LLM×Playwright の軽量ループを実装し、一般的な Web UI を自然言語手順で自動操作できる状態を作る。
- スコープ:
  - チェックリスト化（日本語 →[]形式、小規模 LLM で番号や補足意図を除去）
  - アクセシビリティスナップショット抽出（操作可能要素の軽量表現）
  - アクション決定（ゼロショット、ステートレス）
  - 実行（getByRole/getByText/getByTestId の限定サブセット）
  - ステップ完了判定（シンプル判定）
  - 実行ログ（JSONL）

除外（Phase 2 以降）: エマージェンシーモード、仮想リスト・Shadow DOM 拡張、動的 DOM 投入、履歴学習、コード自動生成、サイトプロファイル・sitemap 取得、詳細な失敗ログ仕様など。

### チェックリスト生成ポリシー

- 自然言語テキストはゼロショット LLM に入力し、`["[] アクション", ...]`形式の JSON 配列を返すスキーマつきプロンプトを採用する。
- 番号・箇条書き・序数などは LLM 側で除去し、元の記述に存在しないアクションを追加しないことを明文化する。
- LLM 呼び出し時の`maxTokens`は 3000 を上限とし、設計上それより小さい値で失敗させない。

## 完了の定義（DoD）

- 自然言語の手順（例: 「Amazon で XX を検索して…」）を MVP で最後まで実行できるシナリオが少なくとも 1 つ動作する。
- LLM 呼び出しは小規模モデル前提、1 ステップあたり 3000 トークン以下を原則とする。
- 主要なセレクタは`getByRole(name)`優先、次に`getByText`、必要時のみ`getByTestId`。
- 事前プリチェック（候補数カウント）で 0 件/複数件は即フォールバック（失敗カウント）。
- 成功・失敗を含む実行ログが`logs/`配下へ保存される。

## アーキテクチャの核（ADR 要約: Phase 1 に適用）

- 入力は自然言語の手順。まずチェックリストへ正規化。
- DOM 全体ではなく Accessibility Tree を取得し、操作可能要素のみに絞った軽量表現を LLM へ提示。
- LLM は次アクション（メソッド・セレクタ・値）を返す。自信度の導入は任意（数値を返さない形でもよい）だが、可能なら数値+根拠 1 行を付与。
- 実行は Playwright API で行い、各ステップ完了の可否を再び LLM に簡易判定させる。
- 逐次実行・逐次フォールバック。並列探索や分岐は行わない。

## セレクタ優先順位（Phase 1）

1. `getByRole(name指定)`
2. `getByText`
3. `getByTestId`（Accessibility 情報と整合する場合に限る）

軽量 DOM の投入や`getByLabel`/`getByPlaceholder`/`locator(css)`は Phase 2 以降。

## データモデル（MVP）

```ts
type Step = {
  id: string; // "step_1" など
  originalInstruction: string;
  attempt?: number;
  intent?:
    | "navigate"
    | "click"
    | "fill"
    | "select"
    | "wait"
    | "assert"
    | "end"
    | "observe"
    | "scroll";
  target?: { role?: string; text?: string; testId?: string };
  value?: string | string[]; // 資格情報プレースホルダ（例: "{{LOGIN_ID}}"）のみ格納
  timeoutMs?: number;
};

interface ActionDecisionInput {
  ariaSnapshot: string;
  currentStep: string;
  checklist: string[];
  stepIndex: number;
  currentStepMeta?: Step;
}

interface ActionDecisionOutput {
  method: "getByRole" | "getByText" | "getByTestId";
  selector: { role?: string; name?: string; text?: string; testId?: string };
  action: "click" | "fill" | "select" | "wait";
  value?:
    | { kind: "placeholder"; value: CredentialPlaceholder }
    | { kind: "literal"; value: string };
  confidence?: number; // 任意
  rationale?: string; // 任意
}
```

- `value` フィールドは入力アクション用データ。秘匿値は `{ kind: 'placeholder', value: CredentialPlaceholder }` で流通させ、実値の解決は SecretManager/実行ポートに委ねる。公開してよいリテラルは `{ kind: 'literal', value: string }` として扱い、ログ出力時は共通サニタイズを通す。

### Step モデルの実装方針（Phase 1 更新 / 2025-09-17）

- `Step` は各ループで `deriveNextStep` により動的生成し、識別子 (`id`) と正規化済み指示 (`originalInstruction`)＋`attempt` だけを必須とする。intent / target / value などのヒューリスティック補完は廃止し、LLM 側での推定に全面委譲する。
- `WorkflowContext.stepHistory` には `metadataPresent`・`metadataKeys` を含む履歴エントリを積み上げ、ヒントが欠落した試行 (`metadataMissing: true`) をログ解析から即座に特定できるようにする。
- `ActionDecisionInput.currentStepMeta` は optional。値が空またはヒントゼロの場合でもプロンプトに「テキストのみで判断せよ」「特定できなければ { "error": "TARGET_NOT_FOUND" } を返せ」と明示して、意図しないヒューリスティック復活を防ぐ。
- ヒューリスティックフォールバック（クリック/入力等の推定ロジック）は削除済み。LLM が失敗した場合は即座に `DecisionError('LLM fallback unavailable')` を返し、`runActionLoop` → `processStep` で `LLM_FALLBACK_DISABLED` ログとともにステップ失敗として扱う。

## 主要フロー（MVP）

```ts
async function executeWorkflow(workflowText: string) {
  const checklist = await convertToChecklist(workflowText);
  for (let i = 0; i < checklist.length; i++) {
    const currentStep = checklist[i].replace("[]", "").trim();
    let completed = false;
    let attempts = 0;

    while (!completed && attempts < 3) {
      const aria = await extractAriaSnapshot(page);
      const action = await decideAction({
        ariaSnapshot: aria,
        currentStep,
        checklist,
        stepIndex: i,
      });

      const count = await resolveCandidateCount(page, action);
      if (count !== 1) {
        attempts++;
        continue;
      }

      await executeAction(page, action, "aria");
      completed = await checkStepCompletion(currentStep, aria);
    }
  }
}
```

## ログ（MVP）

- 形式: JSON Lines（`logs/{sessionId}/steps.jsonl`）
- 含有: `stepNumber`/`originalInstruction`/`decision(method,selector)`/`result(success)`/`timestamp` に加えて `metadataPresent`/`metadataMissing`/`metadataKeys`
- 機密値: すべてのログ（JSONL・コンソール出力・LLM 入出力ログ）は実際の資格情報を含めない。`value` など秘匿値を持ち得るフィールドは書き込み前の共通サニタイザで削除または `***` に置換し、プレースホルダ名のみ残す。

## 資格情報・秘匿データの扱い（2025-09-16 更新）

- 認証情報（ログイン ID/メール/パスワード等）は外部の安全なストア（例: Secrets Manager。最終決定は別途）に保存し、ワークフロー実行時に専用ポート経由でオンデマンド取得する。
- LLM に渡すプロンプトやチェックリストには実値を含めず、`{{LOGIN_ID}}` や `{{PASSWORD}}` などのプレースホルダのみ展開する。プレースホルダの意味（種別）だけをコンテキストとして渡し、値そのものは共有しない。
- Playwright 実行直前にだけ `CredentialKey -> 実値` を解決し、DOM 操作後は速やかに破棄する。リトライ時もセッション内限定で再利用し、永続化は行わない。
- ログ、LLM コール結果、アプリ内部のイベントストリームではプレースホルダとステップメタデータのみ保持し、秘匿値がメモリ外に流出しないよう共通サニタイザを適用する。
- 変数解決やサニタイズ処理は Effect/Zod ベースの関数型 DDD パターンで実装し、ステップ実行フローに副作用境界を明示する（例: `resolveSecretEffect`, `sanitizeLogEntry`）。

## 非機能要件（MVP 目標）

- トークン: 1 ステップあたり 3000 トークン以下
- レイテンシ: 1 ステップ平均 2s 以内（LLM 200ms 目安、抽出 50ms 目安）
- リトライ: 各ステップ最大 3 回

## 実装タスク（ToDo）

- `src/core/normalizer.ts`: 手順正規化/チェックリスト化
- `src/core/extractor.ts`: Accessibility Tree 抽出 → 軽量化
- `src/core/decision.ts`: アクション決定（最小プロンプト）
- `src/core/executor.ts`: 実行（aria モード限定）
- `src/core/completion.ts`: 完了判定（true/false）
- `src/core/precheck.ts`: `resolveCandidateCount`
- `src/adapters/playwright.ts`, `src/adapters/llm.ts`: 統合
- `src/services/logs.ts`: JSONL ロガー（サニタイザを通してプレースホルダのみ保存）
- 秘匿情報解決ポート: 資格情報ストアからプレースホルダを実値にマッピングする Effect を定義
- プレースホルダ展開/整合ユーティリティ: チェックリスト生成や LLM プロンプト生成でプレースホルダの整合性を維持

## 既知の制約（Phase 2 で解消予定）

- 複雑 UI（仮想リスト/Shadow DOM）未対応
- エマージェンシー/軽量 DOM 投入なし
- 詳細な失敗ログ・エラー分類なし

## 参照

- [[🗒️ 000024-playwright-llm-adaptive-automation-adr.md]]
- [[🗒️ playwright-llm-detailed-design.md]]
