---
type: note
---
> [!info] 必要なセクションを選択して使用してください！

# 📌 目的・対象範囲

- **目的**
  - AWS AmplifyとGithub Repositoryを連携し、hookによりAmplifyへの自動デプロイを実現する

# 🚩 前提条件

- TerraformによるAmplifyリソース管理

# 🛠 使用するツール・環境


# 📋 実施手順

## ① PAT(Personal Access Token)を発行する

Fine-grained tokenから発行する。
![[スクリーンショット 2025-07-24 14.22.38.png]]


## ② WebhooksにR/W権限を追加する
![[スクリーンショット 2025-07-24 14.27.45.png]]
## ③ 発行されたトークンをTFCの変数に追加しデプロイする

変数名: `AMPLIFY_GITHUB_ACCESS_TOKEN`
![[スクリーンショット 2025-07-24 14.29.13.png]]

## ④ AWSマネコンのAmplify画面より、リリースを実施し、GithubからCloneできていることを確認する

![[スクリーンショット 2025-07-24 14.41.11.png]]
# 🚧 注意事項

- TFCの変数はすべてIaCで管理しています。変数を追加する際はこちらから設定してください。
	- kunai-coreの例 [Fetching Data#vvuo](https://github.com/KENCOPA/infra/blob/main/tfc/workspace_aws_kunai_core.tf)
- Fine-grained tokenの有効期限はMax366日であることに注意

# ✅ 確認方法

手順が正しく完了したことを確認する方法を記載します。

# 📚 関連資料・リンク

```cardlink
url: https://dev.classmethod.jp/articles/amplify-hosting-github-with-basic-auth/
title: "Amplify Hostingを使用してGitHubリポジトリのコンテンツを公開してみた with ベーシック認証 | DevelopersIO"
host: dev.classmethod.jp
favicon: https://dev.classmethod.jp/favicon.ico
image: https://images.ctfassets.net/ct0aopd36mqt/wp-thumbnail-875a3d2bbdba43ad15684c9ee2e357b1/c77a61f7c646b6997c54cc55538ec251/aws-amplify
```



