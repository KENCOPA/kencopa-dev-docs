---
type: note
moc: "[[🏷️ADR]]"
tags:
- '#ai/ref'
- ElasticCache
---
> [!info] 必要なセクションを選択して使用してください！

# 📜 文脈・背景

AIエージェントの構築にあたりエージェントのメモリ管理にElasticCacheを使用することになった。しかし、チームにElasticCacheに詳しい人間がいないため、どのようにリソースを構築するのがよいか検討する必要があった。
- [[🗒️ ai-agent & chat]]
- [[🗒️ ai-agent & chat AWS全体アーキテクチャ図（Mermaid）]]

# 🎨 対応案
## 案1. **ElasticCache Serverless**

- 先の負荷見積もりが難しい開発初期や PoC。従量課金でスケール自動化
- 後述のノードベースに比べると3倍ほど高い
	- Serverless：1時間あたり $0.151（1ヶ月で約$108.7）
	- ノードベース：1時間あたり $0.05 （1ヶ月で約$36.0）
```cardlink
url: https://dev.classmethod.jp/articles/amazon-elasticache-serverless-ops-made-easy/
title: "Amazon ElastiCacheがサーバーレスになって運用がバチクソ楽になった #AWSreInvent | DevelopersIO"
host: dev.classmethod.jp
favicon: https://dev.classmethod.jp/favicon.ico
image: https://devio2023-media.developers.io/wp-content/uploads/2023/08/amazon-elasticache.png
```

## 案2. ノードベース（プロビジョンド）

- 予測可能なワークロード、きめ細かなチューニングが欲しい本番環境
-  容量計算とシャード再分割を自分で管理する必要がある
# 🚀 決定

Dev / Stg / Prodの全環境でServerlessを使用する。運用してみないと必要とされるワークロードやコストが推定できないためである。
下記の5項目だけを設定したミニマムな運用から開始する
 1. `Cache 名 (--serverless-cache-name)`
 2. `エンジン (--engine)`
 3. `VPC／サブネット (--subnet-ids)`
 4. `Security Group (--security-group-ids)`
 5. `スケール上限・下限 (--cache-usage-limits)`

>[!note] DataStorageの上限に加えてECPUの上限を設定しないでよいのか
>とりあえずは設定しない方針。Elastic Serverlessの課金体系はDataStorageに対する価格が支配的であり、ECPUに対する価格は少額になる傾向がある。下手に上限を設定するとスロットリングのリスクがあるため、ECPUの上限は設けない。
# 🪞 結果・影響

## プラスの結果
- 運用の手間が大幅に削減される。

## マイナスの結果
- コストがかかる。
- 特にDev / Stg環境でServerlessにするべきかは疑問なため、CloudWatchでの監視を行いパフォーマンスに問題がないか、コストは適当かを見直し続ける必要がある。


