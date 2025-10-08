---
type: note
---
> [!info] 必要なセクションを選択して使用してください！

# 🎯 目的・対象範囲

- **目的**
    - LLMを利用したサービスを実装していく上での、プロンプト管理をLangfuseにて行う
- **対象範囲**  
    - AI Agentの7月リリースに向けたLLMOpsの一環のテスト
    - 現状SafeGenのみの暫定的な対応

# 🏗 システム構成概要
## Langfuse Prompt Management

Langfuseは、プロンプト（指示文）を管理・バージョン管理するためのCMSとして機能する。
#### 🎯 目的：

- プロンプトを **コードから分離** して管理しやすくする
- **バージョン管理** によって変更履歴やABテストが可能
- **UIやAPIから簡単に更新可能**
- 開発環境／本番環境など、**ラベルによる切り替え**ができる
#### ✍️ プロンプト管理って何？

**プロンプト管理とは、LLMに渡す指示文を体系的に保存・バージョン管理・再利用する方法**
- プロンプトの変更履歴を管理（Gitのように）
- コードを再デプロイしなくても、プロンプトだけ差し替え可能
- UIからノンエンジニアでも編集できる
- 過去のバージョンにすぐ戻せる
- 各バージョンのパフォーマンスもLangfuseで可視化
#### 💡 なぜLangfuseで管理するの？

| 特徴           | 内容                                   |
| ------------ | ------------------------------------ |
| **分離性**      | コードとプロンプトを分離して独立して管理・更新できる           |
| **バージョン管理**  | 各バージョンを保存。ラベルをつけて呼び出せる               |
| **ノーコード編集**  | LangfuseのUIで編集可能                     |
| **パフォーマンス**  | SDKによるキャッシュ機構で高速に動作し、初回以降はネットワーク遅延なし |
| **比較とABテスト** | 複数のプロンプトの性能をLangfuse Tracingで比較できる   |


# 🛠 基本設計

#### 🔧 プロンプトの構成要素（例）


``` json
{
  "name": "movie-critic",
  "type": "text",
  "prompt": "As a {{criticLevel}} movie critic, do you like {{movie}}?",
  "config": {
    "model": "gpt-3.5-turbo",
    "temperature": 0.5,
    "supported_languages": ["en", "fr"]
  },
  "version": 1,
  "labels": ["production", "staging", "latest"],
  "tags": ["movies"]
}

```

|フィールド|説明|
|---|---|
|`name`|Langfuseプロジェクト内の一意な名前|
|`type`|`text`または`chat`|
|`prompt`|実際のプロンプト。`{{}}`で変数を使える|
|`config`|モデル設定などのメタ情報（任意）|
|`version`|バージョン番号（自動で増加）|
|`labels`|呼び出し時に使えるラベル（例：production, staging）|
|`tags`|任意のタグ（例：movies）|

### 🚀 プロンプトの使い方（Python/TS両対応）

#### 1. プロンプトを取得して変数を埋め込む（Python）

``` python
langfuse = Langfuse()
prompt = langfuse.get_prompt("movie-critic")
compiled = prompt.compile(criticlevel="expert", movie="Dune 2")

```

#### 2. ラベルやバージョン指定

``` python
langfuse.get_prompt("movie-critic", label="staging")
langfuse.get_prompt("movie-critic", version=1)
```

#### 3. チャットプロンプトの使い方


``` python
chat_prompt = langfuse.get_prompt("movie-critic-chat", type="chat")
compiled = chat_prompt.compile(criticlevel="expert", movie="Dune 2")
```

### 🛟 フォールバック対応

万が一Langfuseが落ちた場合に備え、フォールバック（代替プロンプト）を指定可能です。
``` python
langfuse.get_prompt("movie-critic", fallback="Do you like {{movie}}?")
```

# **📚 参考／付録**

- ADR リンク一覧
	-  [[🗒️ 000004-LLMOpsツールの選定]]
- Langfuse
```cardlink
url: https://langfuse.com/docs/prompts/get-started
title: "Open Source Prompt Management - Langfuse"
description: "Manage and version your prompts in Langfuse (open source). When retrieved, they are cached by the Langfuse SDKs for low latency."
host: langfuse.com
favicon: https://langfuse.com/favicon-32x32.png
image: https://langfuse.com/api/og?title=Open%20Source%20Prompt%20Management&description=Manage%20and%20version%20your%20prompts%20in%20Langfuse%20(open%20source).%20When%20retrieved%2C%20they%20are%20cached%20by%20the%20Langfuse%20SDKs%20for%20low%20latency.&section=Docs
```


使い方(OpenAISDKの例)
```cardlink
url: https://langfuse.com/docs/prompts/example-openai-functions
title: "Example: Langfuse Prompt Management for OpenAI functions (Python) - Langfuse"
description: "Cookbook on how to use Langfuse Prompt Management to version control prompts collaboratively when using OpenAI functions."
host: langfuse.com
favicon: https://langfuse.com/favicon-32x32.png
image: https://langfuse.com/api/og?title=Example%3A%20Langfuse%20Prompt%20Management%20for%20OpenAI%20functions%20(Python)&description=Cookbook%20on%20how%20to%20use%20Langfuse%20Prompt%20Management%20to%20version%20control%20prompts%20collaboratively%20when%20using%20OpenAI%20functions.&section=Docs
```

