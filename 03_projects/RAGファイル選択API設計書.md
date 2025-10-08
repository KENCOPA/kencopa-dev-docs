---
type: note
moc: "[[🏷️基本設計書]]"
tags:
- ai/ref
---
## 🎯 目的・対象範囲

### 目的

共有ライブラリの「ファイルの再利用性」を活かしつつ、ユーザーが意図した特定ファイルを丸ごと文脈として扱える経路を追加し、既存の「全ナレッジ検索（chunking & embedding）」を補完する。

位置づけと使い分け:

- 既存（全ナレッジ検索）: 知識検索・部分抽出に強い（建設業特化の基盤知識をカバー）
- 本実装（ファイル丸ごと注入）: 要約・全体俯瞰・指定ファイルに限定した回答に強い
- 両者は置き換えではなく補完関係（課題と強みが異なる）

### 用語と役割の整理

- DriveFiles: 業務ファイルの正本（メタデータと階層）を表すアプリ内の「一覧・カタログ」モデル。バックエンドの DriveFiles の「ファイル一覧取得 API」（正式名称は実装に合わせて記載）が返す結果が、UI に表示されるファイルツリー/一覧の正体。保持するのは表示名、パス/階層、所属スコープ（tenant/site）などの属性と、S3 上の実体を指す参照（bucket・key）。UI からの追加・改名・移動・削除はまず DriveFiles に反映され、実体は S3（例: tenant-drive-files / site-drive-files）に保存される。DriveFiles が業務上の状態を一意に示す「正本」。
- Knowledge Base（KB）: DriveFiles 等の元データを同期して検索に最適化（チャンク化・ベクトル化）した派生インデックス。正本ではなく、非同期同期のため DriveFiles の更新と一時的に乖離しうる。返すのは関連チャンクと出典メタデータで、全文やバイナリは返さない。本ドキュメントの対象フロー（選択ファイル／S3 直読）では KB は使用しない。

### 対象範囲

- **対象システム**: AI エージェント RAG システム
- **対象機能**: 共有ライブラリからの選択的ファイル参照機能
- **開発期間**: 約 1.5 ヶ月

## 🏗 システム構成概要

### 現状のフロー

1. ユーザーが質問を投げる
2. RAG システムが共有ライブラリ全体に対して類似度検索を実行
3. ヒットした多数のチャンク（情報の断片）をコンテキストとして LLM に渡す
4. LLM が多数のチャンクから回答を生成する → 精度そこそこ

### 新しいフロー（S3 直接アクセス方式）

1. ユーザーがチャット UI で「ライブラリからファイルを選択」を開始
2. フロントがバックエンドの DriveFiles の「ファイル一覧取得 API」を呼び、対象スコープ（tenant/site）のファイル一覧を取得
3. UI が取得した一覧を表示し、ユーザーが使用するファイルを選択（複数可。サイズ上限超過は選択不可／選択数は上限内）
4. 選択されたアイテムから S3 参照（bucket + object key）を特定
5. フロントは azuchi-core の REST エンドポイントに「ユーザーメッセージ＋選択ファイルの S3 参照」を送信（versionId は送らず常に最新を参照）
6. azuchi-core は azuchi-chat の REST エンドポイントを呼び出し、メッセージ・ファイル名・S3 参照を Dynamo に永続化
7. azuchi-chat は azuchi-ai-agent の REST エンドポイントを呼び出して SQS を発火
8. ai-agent は SQS を受信し、S3 から各ファイルの内容を直接取得してサイズ制御のうえコンテキストに編成。全体検索の KB は実行しない。
9. LLM に投入して回答を生成する。レスポンスは azuchi-ai-agent → azuchi-chat → azuchi-core の順で返却される（回答本文＋使用したファイルの表示名と S3 のパス）

## 🛠 基本設計

### アーキテクチャ設計と役割分担

3 つの観点による総合評価

#### 観点 1: 実装の効率性・既存資産の活用

**ハイブリッドアプローチ（azuchi-backend + ai-agent）**

azuchi-backend 側のメリット:

- ✅ DriveFiles API が既に実装済み（ファイル一覧取得はそのまま活用）
- ✅ 認証・認可の仕組みが整備済み
- ✅ フロントエンドとの統一的なインターフェース

ai-agent 側のメリット:

- ✅ **SQS メッセージサイズ制限（256KB）を回避** - S3 参照（bucket + object key）のみ送信
- ✅ 大容量ファイル（数 MB〜数十 MB）の処理が可能
- ✅ boto3 での S3 アクセス実装は簡単
- ✅ ファイル内容を直接 Context.knowledge に追加可能

デメリット:

- ❌ ai-agent に S3 アクセス処理の新規実装が必要（ただし簡単）

#### 観点 2: ドメイン駆動設計・マイクロサービス原則

**責務の明確な分離**

azuchi-backend の責務:

- ✅ **UI サポート機能** - ファイル選択のためのメタデータ提供
- ✅ **認証・認可** - ユーザーが選択可能なファイルの制御
- ✅ **DriveFiles 管理** - ビジネスエンティティとしてのファイル情報管理

ai-agent の責務:

- ✅ **RAG データソース管理** - S3 からのファイル取得と処理
- ✅ **AI 推論の前処理** - Context.knowledge への追加
- ✅ **Knowledge Base 制御** - 全体検索のスキップ処理

azuchi-chat の責務:

- ✅ **メッセージ・ファイル参照受理と永続化** - ユーザーメッセージ・ファイル名・S3 参照 を Dynamo に保存（既存チャット経路）
- ✅ **エージェント連携** - azuchi-ai-agent の REST を呼び出して SQS を発火

azuchi-core の責務:

- ✅ **API フロントドア** - フロントエンドからの REST を受理（認証・認可）
- ✅ **チャット連携** - azuchi-chat の REST を呼び出してメッセージを永続化
- ✅ **結果返却** - azuchi-ai-agent/azuchi-chat 経由の結果を集約してフロントへ返却

境界づけられたコンテキスト:

- ✅ DriveFiles ドメイン（メタデータ） → azuchi-backend
- ✅ RAG 処理ドメイン（実データ） → ai-agent
- ✅ 各サービスが自律的に自分のドメインを管理

### 観点 3: API レイヤーとしての責務とデータサイズ制約

**重要な技術的制約:**

- **SQS メッセージサイズ制限**: 最大 256KB
- **ファイルサイズ**: 数 MB〜数十 MB の PDF やドキュメント
- **レスポンスタイム**: フロントエンドへの迅速な応答

**データフローの最適化**

azuchi-backend の役割:

- ✅ **メタデータの提供** - ファイル一覧、storageKey など（UI 側で S3 参照へ変換）
- ✅ **認証・認可** - JWT 認証とアクセス制御
- ✅ **API ゲートウェイ** - フロントエンドへの統一インターフェース

ai-agent の役割:

- ✅ **実データの取得** - S3 から直接ファイル内容を取得
- ✅ **サイズ制約の回避** - S3 参照のみ受信（数百バイト）
- ✅ **効率的な処理** - ファイル内容を直接メモリで処理

データサイズによる判断:

- ❌ ファイル内容を SQS 経由で送信 → 不可能（サイズ超過）
- ✅ S3 参照（bucket + object key）のみ SQS 経由で送信 → 最適（軽量で確実）

### 最終結論: **ハイブリッドアプローチ（azuchi-backend + ai-agent）** 🎯

**決定理由:**

1. **データサイズ制約の解決** - SQS の 256KB 制限を回避し、大容量ファイルに対応
2. **責務の明確な分離** - メタデータ管理と実データ処理を適切に分離
3. **既存資産の活用** - DriveFiles API をそのまま使用、追加実装を最小化
4. **ドメインの一貫性** - 各サービスが自分のドメインに集中
5. **実装の効率性** - シンプルで確実な実装パス

**役割分担:**

azuchi-backend:

- DriveFiles API でファイル一覧を提供（既存実装を活用）
- 認証・認可の処理
- （参考）storageKey などのメタデータ提供（UI 側で S3 参照へ変換）

ai-agent:

- S3 参照（bucket + object key）を受け取り直接ファイル取得（新規実装）
- 取得したファイル内容を Context.knowledge に追加
- Knowledge Base 全体検索のスキップ処理

### モジュール構成

```text
# azuchi-backend（既存活用）
azuchi-backend/
└─ src/
   └─ routes/
      └─ v1/
         └─ drive-files/         # 既存のDriveFiles API
            └─ [既存の実装]      # ファイル一覧取得はこれを使用

# ai-agent（新規実装）
ai-agent/
└─ src/
   ├─ handlers/
   │  └─ message_handler.py     # SQSメッセージ処理（エージェント起動前にファイル取得・注入を完了）
   ├─ gateways/
   │  └─ s3/
   │     └─ file_store.py       # S3 バケット + オブジェクトキーでのファイル取得
   └─ services/
      └─ context_service.py     # Context.knowledge への追加処理（ルールベースで前処理として実施）
```

### データモデル

#### ファイル選択セッション

```typescript
interface FileSelectionSession {
  sessionId: string;
  userId: string;
  selectedFiles: S3FileReference[];
  createdAt: Date;
  expiresAt: Date;
}

interface S3FileReference {
  bucket: string;
  key: string;
  fileName: string;
  fileSize: number;
  contentType: string;
}
```

## ✅ 決定事項（2025-09-15 時点）

- v1 は「固定の上限」で制御する（合計トークン/バイトの動的制御は v2 で検討）。
- 選択ファイルがあっても KB 検索は従来どおり実行し、S3直読で前処理注入した内容と併用する（KBスキップしない）。
- 一覧は初期はフルフェッチ＋フロント（SPA）でローカルフィルタ。ページング/ソート/無限スクロールは実装しない。
- 既存 WebSocket 経路で `s3Objects[]` を受け渡す。REST は暫定互換の `POST /chat/sessions/reply/generate` を受理するが、WS を主経路とする。
- 既存の `storageKey` は当面維持しつつ、推奨は `s3Objects[]`（`{ bucket, key, fileName, fileSize, contentType }`）。

#### エージェント実行前に注入する理由（Tool 化しない）

- **安定性**: LLM のツール呼び出し分岐に依存せず、決定的にファイルを文脈へ追加できる。
- **責務分離**: 取得（前処理）と推論（エージェント）を分け、エージェントの探索空間を増やさない。
- **単純化**: 監査・再現が容易（受信ペイロードと注入結果が 1:1 で対応）。

### API 仕様

#### DriveFiles ファイル一覧取得 API（既存）

| エンドポイント      | メソッド | 説明                                       | リクエスト例                      | レスポンス例                                                                                                                                                            |
| ------------------- | -------- | ------------------------------------------ | --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| /api/v1/drive-files | GET      | DriveFiles の一覧取得（テナント/サイト別） | Query: `?tenantId=...&siteId=...` | `{ files: [{ id: "file-id", fileName: "工事計画書.pdf", storageKey: "/{tenantId}/工事計画書.pdf", fileSize: 1024000, createdAt: "2024-01-01T00:00:00Z" }], total: 10 }` |

補足: バージョンの扱い

- クライアントはバージョン指定不要。サーバーがテナントの「最新 version」（DynamoDB 上で最大の`version`）を自動選択して一覧を返す。
- 過去版の一覧はサポートしない。

補足: レスポンスの S3 参照フィールド

- 互換性: 既存の`storageKey`は当面維持。
- 推奨: `s3BucketId`, `s3ObjectKey` を追加提供（UI で`s3Objects[]`生成に使用）。
- フォーマット: `objectKey`は先頭スラッシュ無し（例: `{tenantId}/工事計画書.pdf`）。
- サイズ情報: `fileSize`（単位: bytes）は UI 側での選択可否判定に使用する。

##### 初期方針（決定）: フロントでのローカルフィルタ

- 初期は DriveFiles をフロントへ全件返し、フロントで名称フィルタを行う。
- ページング・ソート・無限スクロールは実装しない（必要になった段階で再検討）。

#### 実装詳細（azuchi-backend）

既存の DriveFiles API を使用するため、新規実装は不要。フロントエンドは以下のフローで処理：

1. 既存の `/api/v1/drive-files` API を使用してファイル一覧を取得
2. DriveFiles のメタデータから S3 の参照情報（bucket + object key）を特定
3. 選択されたファイルの S3 参照配列（`s3Objects[]`）を ai-agent に送信

ai-agent 側での処理：

```python
# S3 の bucket + object key を指定して直接取得（前処理として実行）
def get_file_from_s3(bucket: str, key: str) -> bytes:
    import boto3
    s3 = boto3.client('s3')
    res = s3.get_object(Bucket=bucket, Key=key)
    return res['Body'].read()
```

### 共通仕様（入力制約とバリデーション／v1）

- ファイルサイズ上限: `FILE_SELECTION_MAX_BYTES`（bytes）。
  - クライアント: DriveFiles 一覧 API の `fileSize` を用いて上限超過を選択不可にする（無効化＋理由表示）。
  - サーバー: 受信した `s3Objects[]` を同一上限で再検証し、超過は 400 `FILE_TOO_LARGE`。
- 選択数上限: `MAX_SELECTABLE_FILES`（例: 5）。
  - クライアント: カウンタ表示と上限到達時の一時無効化。
  - サーバー: 上限超過は 400 `TOO_MANY_FILES`。
- エラーハンドリング（UI）: 400 受信時はトースト通知し、該当行を無効化へ更新。

推奨デフォルト（環境変数で調整可能）:

- `FILE_SELECTION_MAX_BYTES=20000000`（約 20MB）
- `MAX_SELECTABLE_FILES=5`

#### WebSocket メッセージ仕様（主経路）

- 経路: フロント → `API Gateway (WebSocket)` → `putMessage Lambda` → `DynamoDB` 保存 → `azuchi-ai-agent` 呼出 → `SQS` エンキュー。
- アクション: `PUT_MESSAGE`（既存）。
- 互換: 旧 `s3BucketId + s3ObjectKeys[]` は受理し、`s3Objects[]` に変換。

Request Payload 例（省略フィールドは既存定義に準拠）:

```json
{
  "action": "PUT_MESSAGE",
  "sessionId": "sess_123",
  "messageId": "msg_123",
  "message": "この仕様の要件を満たしますか？",
  "s3Objects": [
    {
      "bucket": "prod-azuchi-core-tenant-drive-files",
      "key": "t_abc/施工計画書.pdf",
      "fileName": "施工計画書.pdf",
      "fileSize": 1048576,
      "contentType": "application/pdf"
    }
  ]
}
```

`putMessage` Lambda は `s3Objects[]` をメッセージメタとして DynamoDB に保存し、`azuchi-ai-agent` 呼び出し時に SQS メッセージへ含める。

#### REST（暫定互換）

- エンドポイント（互換）: `POST /chat/sessions/reply/generate`
- Body に `s3Objects[]` と `message` を含める。レスポンスは既存スキーマに準拠。

#### エラーマトリクス（v1）

| HTTP | code                  | 条件                                   | UI 表示（案）                        |
| ---- | --------------------- | -------------------------------------- | ------------------------------------ |
| 400  | FILE_TOO_LARGE        | 任意 1 ファイルが上限超                | 「サイズ上限を超えています」         |
| 400  | TOO_MANY_FILES        | 選択件数が `MAX_SELECTABLE_FILES` 超過 | 「選択可能数の上限に達しています」   |
| 400  | INVALID_S3_OBJECT     | bucket/key 不正                        | 「ファイル参照が不正です」           |
| 403  | FORBIDDEN_FILE        | 認可外のファイル                       | 「アクセス権がありません」           |
| 404  | FILE_NOT_FOUND        | S3/DriveFiles に実体なし               | 「ファイルが見つかりません」         |
| 422  | TOKEN_BUDGET_EXCEEDED | 送信直前の合計予算チェック（任意）     | 「コンテキスト上限を超過しました」   |
| 500  | S3_READ_FAILED        | S3 読取失敗                            | 「サーバー側でエラーが発生しました」 |

### フロントエンド実装

#### ファイル選択 UI

- チャット画面でファイルを選択する UI の実装（デザイン待ち）
- 選択したファイルが UI 上で分かる表示
- 既存の DriveFiles API を叩いて選択できる機能
- 選択したファイルの S3 参照配列（`s3Objects[]`）をメッセージと一緒に送信する実装
- サイズ上限・選択上限の UI 制御（第 1 弾で実装）
  - ファイル一覧取得 API の `fileSize` に基づき、選択不可（チェック不可）を表示
  - 共通仕様に準拠（無効化理由表示、選択数カウンタ、送信前再検証など）。

#### 一覧の表示（v1 決定）

- 名称フィルタ: 入力デバウンス 300ms、部分一致、クライアントサイドフィルタ。
- 表示最適化: 件数が多い場合に限り仮想化を検討（初期は必須ではない）。

#### データ送信方法

- フロントは azuchi-core の REST エンドポイントへ POST（本文にユーザーメッセージと `s3Objects[]` を含める）
- azuchi-core は azuchi-chat を呼び出し、通常経路でメッセージを Dynamo に保存。続けて azuchi-chat から azuchi-ai-agent の REST を呼び出して SQS を発火
- メッセージペイロードに `s3Objects`（配列）を追加
- 互換受理: 旧フォーマット（`s3BucketId` + `s3ObjectKeys[]`）は受理し、API ゲートウェイ層で`s3Objects[]`に変換

例（フロントエンド → azuchi-core → azuchi-chat → ai-agent）:

```json
{
  "type": "user_message",
  "message": "この計画書を要約して",
  "s3Objects": [
    {
      "bucketId": "prod-azuchi-core-tenant-drive-files",
      "objectKey": "{tenantId}/工事計画書.pdf"
    },
    {
      "bucketId": "prod-azuchi-core-site-drive-files",
      "objectKey": "{siteId}/安全計画.xlsx"
    }
  ]
}
```

型例（フロント/チャット共通想定）:

```ts
type S3ObjectRef = { bucketId: string; objectKey: string };

type ChatUserMessage = {
  type: "user_message";
  message: string;
  s3Objects?: S3ObjectRef[]; // 推奨（複数バケット対応）
  // 互換: 旧フィールド（受信側で s3Objects に変換）
  s3BucketId?: string;
  s3ObjectKeys?: string[];
};
```

### バックエンド実装

#### azuchi-backend での処理

1. JWT 認証によるユーザー検証（既存の DriveFiles API で実装済み）
2. テナント ID/サイト ID に基づく認可チェック（既存実装を活用）
3. Prisma で DriveFiles テーブルから一覧取得
4. ファイル情報（storageKey 等のメタデータ）をフロントエンドに返却

#### ai-agent での処理

##### 現在の RAG 処理フロー（調査結果）

1. **Knowledge Base 検索**: `retrieve_with_bedrock`ツールが Bedrock の Knowledge Base から関連ドキュメントを検索
2. **コンテキスト追加**: `add_knowledge_to_context`ツールが検索結果を`Context.knowledge`配列に追加
3. **プロンプト構築**: `Context.knowledge`の内容が LLM のプロンプトに含まれる

##### 新機能での処理フロー

1. azuchi-core → azuchi-chat 経由の WS/REST で `s3Objects[]` を受信（azuchi-ai-agent が受付後に SQS へエンキュー）
2. **S3 から直接ファイル内容を取得**（該当バケットから object key ごとに取得）
3. 取得したファイル内容を前処理（抽出/要約など）し、プロンプトに注入（Context）。
4. Knowledge Base 検索（`retrieve_with_bedrock`）も従来どおり実行し、知識を併用。
5. **回答生成時の処理**:
   - LLM が選択ファイルのコンテキストから回答を生成
   - 回答に必要な情報が見つからない場合は、その旨を明示的に返す
   - 例：「選択されたファイルには、お問い合わせの内容に関する情報が見つかりませんでした。」
   - **注意**: 全社共通ナレッジの扱いは将来拡張で再検討する（現時点では仕様外）

#### エラー仕様

- フロント事前判定とサーバー最終検証は「共通仕様」に準拠。
- 400 エラーは `FILE_TOO_LARGE`（サイズ）/`TOO_MANY_FILES`（件数）を返す。UI はトースト通知＋該当行を無効化更新。

## 🔧 非機能要件対応

- **性能要件**: ファイル一覧取得は 1 秒以内
- **セキュリティ**: テナント単位でのアクセス制御
- **スケーラビリティ**: Lambda 自動スケール
- **コンテキスト長制限**: LLM のトークン上限を考慮した警告表示

#### 制限とフォールバック方針（大容量ファイル）

- 第 1 弾: **UI 制限**
  - 設定値 `FILE_SELECTION_MAX_BYTES` を超えるファイルは選択不可（例: 10–25MB、環境で調整）。
  - UI で無効化と理由提示（トーストではなくインライン/ツールチップ）。
  - バックエンドは最終防衛線として同一上限を検証し、違反時は 400 を返す（詳細は「エラー仕様」）。
- 第 2 弾（将来）: **前処理サマリ**（良さげな案だったのドキュメントに残しました。）
  - 合計サイズがコンテキスト予算を超える場合、先に要約（chunk + summarize）してから要約結果のみを注入。
  - UI 上で「自動要約で利用する」オプションを提供する想定。
- **明示フィードバック**: いずれの段階でも、超過時は UI に理由と代替（対象削減/分割アップロード/将来の自動要約）を提示。
- **ログ**: 超過発生率・無効化件数・（将来）要約成功率をメトリクス化（閾値の最適化に利用）。

### インフラ・権限（追補）

#### 参照先 S3 バケット（例）

- テナント共通: `prod-azuchi-core-tenant-drive-files`（`/{tenantId}/{file_name}`）
- サイト共通: `prod-azuchi-core-site-drive-files`（`/{siteId}/{file_name}`）
- セッション添付: `prod-azuchi-core-session-attachments`（`/{userId}/{sessionId}/{file_name}`）
- 全社共通（選択対象外）: DriveFiles 未対象の Google Drive 由来の同期領域（KB のみ参照）

#### ai-agent の IAM ポリシー例（最小権限）

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::prod-azuchi-core-tenant-drive-files",
        "arn:aws:s3:::prod-azuchi-core-tenant-drive-files/*",
        "arn:aws:s3:::prod-azuchi-core-site-drive-files",
        "arn:aws:s3:::prod-azuchi-core-site-drive-files/*",
        "arn:aws:s3:::prod-azuchi-core-session-attachments",
        "arn:aws:s3:::prod-azuchi-core-session-attachments/*"
      ]
    }
  ]
}
```

#### セキュリティ注意（S3参照の正規化・署名方針）

- 参照正規化（必須）
  - 受信した `s3Objects[]` の各要素について、`bucket` は許可リスト（tenant/site/session 向けバケット）に一致するかを検証。
  - `objectKey` はスコープに応じた許可プレフィックスに一致するかを検証（例: tenantは`{tenantId}/`, siteは`{siteId}/`, sessionは`{userId}/{sessionId}/`）。
  - 上記に一致しない場合は 403 `FORBIDDEN_FILE` もしくは 400 `INVALID_S3_OBJECT` を返す。
- 署名方針（必要時のみ）
  - 原則は ai-agent の IAM で対象バケットに `GetObject/ListBucket` を付与し、サーバー間で直接取得（署名不要）。
  - クロスアカウントや一時的な制約がある場合のみ、chat/backend 側で短寿命の PreSigned GET URL（例: 60秒）を発行して ai-agent に渡す。
  - 監査のため、正規化後の `bucket/objectKey` と発行有無/TTL をログに記録。

## 🧪 検証観点（v1）

- DriveFiles 連携: フルフェッチ→ローカルフィルタで操作感が良好。
- S3 参照（bucket+key）での直接アクセス: 正常/404/403/失敗再試行。
- 送信直前再検証: 上限到達時の UX とエラー一致。
- 手動確認フロー（参考）: 選択 → 送信 → S3直読 → KB 検索併用 → 回答生成。
- パフォーマンス: 初期ロード/フィルタの体感遅延が大きくない（p95 再描画 < 200ms 目安）。
- マルチバケット同時送信。

## 📊 利用分析（余裕がある時に実装）

- メッセージメタに `usedFileKeys[]` を保存し、CloudWatch Logs/Events へ `rag_file_selected` を送出。
- 週次で「選択モード vs 全体検索」の比率と上位利用ファイルを集計可能にする（将来 Redshift/Athena 連携）。

## 🚨 リスクと対策

### 技術的リスク

1. **コンテキスト長の制限**: 選択ファイルの合計サイズ制限を実装
2. **UI の明確性**: モード切替の視覚的フィードバック
3. **期待値のズレ**: ユーザーガイドラインの作成
4. **フォールバック戦略**:
   - 選択されたファイルに回答に必要な情報が含まれていない場合、その旨を明確にユーザーに伝える
   - 例：「選択されたファイルには、ご質問に関する情報が見つかりませんでした」
   - 全体検索への自動切り替えは行わない（ユーザーの意図を尊重）
5. **全社共通ナレッジの扱い**: 現在はファイル選択時に含まれないが、将来的に「選択ファイル＋全社共通ナレッジ」のハイブリッドモードを検討予定

### 運用リスク

- ファイル選択モードと全体検索モードの使い分けについての教育

## 参考

本機能は KB 検索を従来どおり使用する。以下は用語整理として記載。

### Knowledge Base 検索の実体

- 本機能（v1）は KB 検索を使用する。選択ファイルは S3 直読で前処理注入し、KB 由来の知識と併存させる。
- KB は関連チャンク（page_content）と出典メタデータ（file/url/tenantId/siteId/version 等）を返し、全文やバイナリは返さない。
- KB は正本ではなく、DriveFiles との非同期同期により未反映・除外・失敗がありうる。

### DriveFiles と Knowledge Base の関係

- DriveFiles: 正本。S3 上の実体（tenant-drive-files / site-drive-files）を UI/API で管理（一覧・検索・更新・階層）。
- Knowledge Base: DriveFiles 等から同期した検索用インデックス。DriveFiles にある＝必ず KB にある、とは限らない（非同期/除外/失敗）。
- セッション共通ナレッジ（チャット添付）は必要に応じて S3 直読で扱い、KB とは独立に併用されることがある。

### 同期と一貫性に関する注記（KB）

- KB は「最終的整合」。DriveFiles 側の更新から KB 反映までにラグが発生しうる。
- 取り込み対象は設定・フィルタに依存。すべての DriveFiles が KB に載るわけではない。
- 監視/運用: 同期失敗や未反映を検出できるメトリクス/ログの整備を推奨。
