---
type: note
---
> [!info] 必要なセクションを選択して使用してください！

> 目的
> Kencopa で利用する Large Language Model向け API キーの取得・保護・運用方法を一元的にまとめるドキュメント。

---

## 1. 基本原則

1. **中央ゲートウェイ優先 – OpenRouter**
    
    すべての実行トラフィックは OpenRouter 経由でルーティングする。
    
2. **プロダクト単位のキー発行**
    
    プロダクト／サービスごとに専用の OpenRouter API キーを発行。
    
    本番（prod）とローカル（local）は必ず分離。
    
3. **高ティア用にプロバイダー別組織を持つ**
    
    GPT‑o3 や Claude 4 Opus など上位モデルの利用には、各プロバイダーで組織アカウントを用意。

---

## 2. プロバイダー別設定

### 2.1 OpenRouter

- **コンソール:** [https://openrouter.ai/keys](https://openrouter.ai/keys)
	- 招待されていない人は申請してください
- **キー命名:** `prod-<product>`, `stg-<product>`

### 2.2 OpenAI

- **URL:** [https://platform.openai.com/settings/organization/general](https://platform.openai.com/settings/organization/general)
- **組織:** `Kencopa`
- **管理者:** 遠藤（主）
- **キー:** 原則発行せず、調査やフェイルオーバー用のみ

### 2.3 Anthropic

- **URL:** [https://console.anthropic.com/dashboard](https://console.anthropic.com/dashboard)
- **組織:** `Kencopa`
- **管理者:** 遠藤
- **キー:** 原則発行せず、調査やフェイルオーバー用のみ

### 2.4 Google Gemini

- **組織機能:** 現在未対応
- **暫定対応:** 遠藤の Google Workspace プロジェクトキーを共有
- **将来:** 2025 Q4 予定の組織課金開始後に移行

---

## 9. 参考リンク

|プロバイダー|コンソール|Docs|
|---|---|---|
|OpenRouter|[https://openrouter.ai/keys](https://openrouter.ai/keys)|[](https://docs.openrouter.ai/)[https://docs.openrouter.ai](https://docs.openrouter.ai)|
|OpenAI|[](https://platform.openai.com/)[https://platform.openai.com](https://platform.openai.com)|[https://platform.openai.com/docs](https://platform.openai.com/docs)|
|Anthropic|[https://console.anthropic.com/dashboard](https://console.anthropic.com/dashboard)|[https://docs.anthropic.com/](https://docs.anthropic.com/)|
|Google Gemini|[](https://aistudio.google.com/apikey)[https://ai.google.dev/](https://ai.google.dev/)||

---


