---
type: note
tags:
- '#ai/ref'
---
> [!info] 必要なセクションを選択して使用してください！

# 🎯 目的・対象範囲

- **目的**
    - AI エージェントのAWSでの実装方針を明確にする
- **対象範囲** 
    - 2025年7月の製品版公開に向けた設計

> [!info] 
> 現時点（2025年4月時点）では画像などテキスト以外の扱いをどうするかは設計の対象外としている。

# 🏗 システム構成概要

## アーキテクチャ図
- [[🗒️ ai-agent & chat AWS全体アーキテクチャ図（Mermaid）]]
## ADR
- [[🗒️ 000001-AIエージェントとチャット機能をマイクロサービス化]]
- [[🗒️ 000002-ai-agentのデータ永続化にRedisを使用]]
- [[🗒️ 000003-ai-agentのロック機構にRedisを使用]]
# 🛠 基本設計
## [azuchi-chat] Web Socket ルート設計
### $connect
**概要**
- web socket接続に使用するルート。

**パラメーター**
- sessionId: String | undefined

**処理手順**
- eventからconnectionIdを取得する
- sessionIdがundefinedの場合
	- sessionIdを採番する（UUID）
	- sessionsテーブルにsessionIdを書き込む
- connectionsテーブルにconnectionIdを書き込む
### $disconnect
**概要**
- web socket接続終了時に使用するルート
**処理手順**
- eventからconnectionIdを取得する
- connectionsテーブルからconnectionIdを削除する

### $default
**概要**
- フォールバック処理に使用されるルート
**処理手順**
- 使用可能なルートの一覧を返す

### putMessage
**概要**
メッセージ作成時に使用するルート
**処理手順**
- 必要な情報をeventから取得する
- ai-agentを起動する
- messagesテーブルに書き込む

## [azuchi-chat] Dynamoテーブル設計

### sessions
セッションを管理するテーブル。`$connect`で書き込む。

|               | key       | value  | description     | example                              |
| ------------- | --------- | ------ | --------------- | ------------------------------------ |
| partition key | sessionId | String | アプリケーションで採番される値 | 11756791-ed89-4197-bb09-b096be95412e |
|               | title     | String | セッションのタイトル      | xxxをyyyする方法                          |

### connections
web socketのコネクションを管理するテーブル。`$connect`で書き込み、$disconectで削除。

|                        | key          | type   | description                      | example                              |
| ---------------------- | ------------ | ------ | -------------------------------- | ------------------------------------ |
| partition  key         | connectionId | String | Web Socket接続時にAPI Gatewayで採番される値 | H0zlrcJgtjMCEHA=                     |
|                        | sessionId    | String | アプリケーションで採番される値                  | 11756791-ed89-4197-bb09-b096be95412e |
| Global Secondary Index | userId       | String |                                  |                                      |

### messages
メッセージを管理するテーブル。`putMessage`で書き込む。

|               | key          | value  | description                                | example                              |
| ------------- | ------------ | ------ | ------------------------------------------ | ------------------------------------ |
| partition key | sessionId    | String | アプリケーションで採番される値                            | 11756791-ed89-4197-bb09-b096be95412e |
| sort key      | createdAt    | Number | レコードが作成された時間                               | 1740705881                           |
|               | connectionId |        |                                            |                                      |
|               | role         | String | メッセージを送信したクライアントの種別（USER / AGENT / SYSTEM） | USER                                 |
|               | message      | String | メッセージの本文                                   | こんにちわ！！                              |
|               | messageType  | String | メッセージの種別（テーブル下の情報を参照）                      | CONVERSATION                         |
**messageType一覧**
> [!information] 暫定なので適宜追加する
> EXCELとかWORK_ORDER_PREVIEWとかOUTPUTに応じたmessageTypeが増えていきそう

- CONVERSATION: 会話（ex. 今日は何のお手伝いをしましょうか？）
- PLAN: Agentの実行計画
- SYSTEM_LOG: システムの実行ログ（ex. 接続中です）
- AGENT_LOG: Agentの実行ログ（ex. xxxを実施しました）

## [azuchi-ai-agent] API Gateway API設計

### generateChatReply
**概要**
- チャットの返信を生成する

**パス**
- POST: api/v1/chat/sessions/{sessionId}/reply/generate

**パラメーター**
- sessionId（Path）: String
- message（Body）: String

**レスポンス**
なし
## [azuchi-ai-agent] ElasticCache キー設計
### キーの命名規則
- `<app_prefix>`：アプリケーション名（例：azuchi）
- `<service_domain_prefix>`：サービスドメイン名（例：ai-agent）
- `<category>`：大分類（例：lock／session）
- `<session_id>`：UUIDなどのセッション識別子
- `<sub_key>`：session 配下の更に細かい用途（例：history／memory）
```
<app_prefix>:<category>:<session_id>[:<sub_key>]
```
### ロック用キー
| **用途**   | **キー例**                                     | **型**  | **操作**                                        | **TTL**  |
| -------- | ------------------------------------------- | ------ | --------------------------------------------- | -------- |
| セッションロック | azuchi:ai-agent:lock:session:{{session_id}} | String | SET key value NX PX 30000Luaスクリプトで所有者チェック＋DEL | 30秒 (PX) |

### 会話履歴用キー
| **用途**       | **キー例**                                        | **型** | **操作例**                                  | **TTL** |
| ------------ | ---------------------------------------------- | ----- | ---------------------------------------- | ------- |
| 最新 N 件チャット履歴 | azuchi:ai-agent:session:{{session_id}}:history | List  | - LPUSH key <JSONメッセージ>- LTRIM key 0 N-1 | なし      |

### 会話履歴ID用キー
openai agent SDKで使用できるprevious_response_idを保持しておくキー。

| **用途**                  | **キー例**                                                     | **型**  | **操作例**                         | **TTL** |
| ----------------------- | ----------------------------------------------------------- | ------ | ------------------------------- | ------- |
| previous_response_idの保持 | azuchi:ai-agent:session:{{session_id}}:previous_response_id | String | SET key resp_789 EX 1728000<br> | 20日     |
