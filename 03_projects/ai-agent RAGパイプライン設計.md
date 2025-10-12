---
type: note
moc: "[[🏷️research]]"
tags:
- '#ai/ref'
---
> [!info] 必要なセクションを選択して使用してください！

# 🎯 目的・対象範囲

- **目的**
    - AIエージェントが利用するRAGシステムの事前アップロードデータの前処理パイプラインの構築
- **対象範囲**  
    - azuchi-ai-agent

# 🏗 システム構成概要

## アーキテクチャ図
- [[🗒️ ai-agent & chat AWS全体アーキテクチャ図（Mermaid）]]
## ADR
- [[🗒️ 000010-RAGのpreprocessパイプラインにはLambdaを使う]]
# 🛠 基本設計

## データ更新方針
- 年に1回データ全件増分更新
- google driveにrawデータを格納しておき、Lambdaを手動実行する
- 更新時はgoogle driveに新規ディレクトリを作成し、そこに必要なファイルを全て格納する
	- 仮に更新対象が1つのファイルだけであってもRAGさせたいデータを全て新規ディレクトリに格納する
## Lambdaの処理概要
下記の処理を実現する単一のLambdaを実装する
1. google driveのファイルをs3に格納する
2. メタデータを作成しs3に格納する
3. bedrock knowledge baseの増分同期を実行し、S3のファイルを読み込む
### 1. google driveのファイルをs3に格納する
`manifest.json`に記載されているディレクトリにあるファイルを全てs3にputする。
google driveのディレクトリ構造は下記の通り。
```
. /
├── manifest.json
├── 2024/ 
│ └── 0103 
└── 2025/ 
  ├── 0104 
  └── 0105/ 
    └── 土木工事共通仕様書.pdf
	└── ...
```
`yyyy` > `mmdd`の順にディレクトリが構成される。基本的に1つの`yyyy`ディレクトリには1つの`mmdd`ディレクトリしか配置されない（年１回更新のため）想定であるが、ファイル格納ミスや急なデータ更新が必要になった場合はその限りではない。
> [!warning] **重要な注意点**
> このパイプラインではバージョン管理キーとして yyyymmdd を使用します。Bedrock Knowledge Base を参照するクライアントは指定された yyyymmdd のメタタグの付いたファイル全体を読み込むため、たとえ一つのファイルだけを更新したい場合でも、そのバージョンディレクトリ内には必ずすべてのファイルを揃えて配置してください。

また、`manifest.json`は下記の形式は下記の通り。
```json
{
	target: "20250105"
	metadata: [
		{
			"file": "hoge.png",
			"url": "https:hoge",
			...
		}
	]
}
```
Lambdaはここで指定されたバージョンディレクトリを参照するため、ファイルを配置した後に`manifest.json`の更新を忘れないように注意。
### 2. メタデータの作成
`manifest.json`からメタデータを抽出し、{file}.metadata.jsonを作成する。作成後、1と同様にS3にアップロードする。
### 3. bedrock knowledge baseの増分同期を実行する

AWS SDKを使ってBedrock Knowledge Baseの同期を実行する。


