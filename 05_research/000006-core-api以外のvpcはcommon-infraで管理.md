---
type: note
---
> [!info] 必要なセクションを選択して使用してください！

# 📜 文脈・背景

複数のサービス（core, ai-agent etc）が立ち上がる中で複数のVPCを管理する必要が出てきた。1アカウントに配置できるVPCはリージョンあたり5と決まっているため、将来の拡張性を踏まえどのようにVPCを管理するか決定する必要があった。

```cardlink
url: https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/amazon-vpc-limits.html
title: "Amazon VPC クォータ - Amazon Virtual Private Cloud"
description: "必要に応じて、以下のデフォルトの Amazon VPC コンポーネントのクォータを引き上げることをリクエストします。"
host: docs.aws.amazon.com
favicon: https://docs.aws.amazon.com/assets/images/favicon.ico
```

# 🎨 対応案

1. `azuchi-core-api`のVPCに他サービスのリソースをデプロイする
	- 他サービスが`azuchi-core-api`に依存してしまうため却下
	- サービス間の依存関係はなくして疎結合にしておきたい
2. サービスのリポジトリでVPCを個別に管理する
	- サービス間の依存関係はなくすことができ、疎結合を保つことができる
	- 一方でVPCのクォータ制限を考慮しづらくなってしまうため却下
3. `azuchi-common-infra`で全てのVPCを中央集権的に管理する
	- 本来はこれで管理したかった
	- しかし、`azuchi-core-api`は既に全環境にデプロイ済み & β版リリース直前であり、このタイミングからの環境再構築は工数的に現実的ではないため却下
4. `azuchi-core-api`以外のVPCを`azuchi-common-infra`で全てのVPCを中央集権的に管理する
	- 3. の妥協案
	- `azuchi-core-api`を除く今後作成する全てのVPCは`azuchi-common-infra`で管理する

# 🚀 決定

対応案4を採択し、`azuchi-core-api`以外のVPCを`azuchi-common-infra`で全てのVPCを中央集権的に管理する。

# 🪞 結果・影響

Bastionサーバーもcommon-infraに定義する。Private SubnetごとにBastionサーバーを準備するのではなく、Bastionサーバー用のPublic Subnetを1つ準備しそこから全てのPrivate Subnetにアクセスする。
[[🗒️ common-infraにおけるbastionサーバーのイメージ]]
![[Pasted image 20250505112735.png]]
