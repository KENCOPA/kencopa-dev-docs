---
type: note
moc: "[[🏷️ADR]]"
tags:
- '#ai/ref'
---
## 📜 文脈・背景

### 業務要件

- 日報機能において、工程管理データから生成された日報をメール送信する必要がある
- メールには以下の要素が含まれる：
    - 件名、本文（HTML/テキスト）
    - 複数の添付ファイル（画像、PDF、Excel等）
    - 複数の宛先（メーリングリスト展開後）
    - 返信先メールアドレスの指定

### 技術的制約

- インフラストラクチャとしてAWSを使用しており、Amazon SESが利用可能
- メール送信サイズ制限：Amazon SESは40MB（添付ファイル含む）まで対応
- 10MBを超える場合は帯域幅スロットリング（40MB/秒）の対象

### 既存システムの状況

- TypeScript + Node.js環境
- AWS Lambda + API Gatewayでのサーバーレス構成
- 既にAWS SDKは導入済み

### 検討された技術的課題

1. **Amazon SES直接利用の制限**
    - `sendEmail` APIは添付ファイルに対応していない
    - `sendRawEmail` APIはMIMEメッセージの手動構築が必要
    - Base64エンコーディング、マルチパート形式の実装が複雑
2. **開発・保守効率**
    - 手動でのMIME構築は実装コストが高い
    - 添付ファイルのエラーハンドリングの複雑さ

## 🎨 対応案

### 案1: Amazon SES直接利用

**概要**: AWS SDKのSES機能を直接使用してメール送信を実装

**メリット**:

- 外部ライブラリへの依存なし
- AWS環境との親和性が高い
- 細かい制御が可能

**デメリット**:

- `sendRawEmail`でのMIMEメッセージ手動構築が必要
- 添付ファイル対応のためのBase64エンコーディング実装
- マルチパート形式の実装が複雑
- 開発効率が悪い

### 案2: Nodemailer + Amazon SES

**概要**: NodemailerライブラリをSESトランスポートで使用

**メリット**:
- 添付ファイル対応が簡潔
- MIMEメッセージの自動構築
- 豊富なエラーハンドリング
- 環境別設定の容易さ
- 開発効率が高い

**デメリット**:
- 外部ライブラリへの依存
- 若干のオーバーヘッド

Refs:
```cardlink
url: https://qiita.com/sushitaro0310/items/327d190d4faced1191bb
title: "AWS SESを使って添付ファイル付きのメールを送信する - Qiita"
description: "結論sendRawEmail関数を用いてMIME形式のメールにすれば添付ファイルを送信できる。sendEmailでは添付ファイル付きのメールを送信できないSESでメール送信する際に最初に思いつ…"
host: qiita.com
favicon: https://cdn.qiita.com/assets/favicons/public/production-c620d3e403342b1022967ba5e3db1aaa.ico
image: https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-user-contents.imgix.net%2Fhttps%253A%252F%252Fcdn.qiita.com%252Fassets%252Fpublic%252Farticle-ogp-background-afbab5eb44e0b055cce1258705637a91.png%3Fixlib%3Drb-4.0.0%26w%3D1200%26blend64%3DaHR0cHM6Ly9xaWl0YS11c2VyLXByb2ZpbGUtaW1hZ2VzLmltZ2l4Lm5ldC9odHRwcyUzQSUyRiUyRnMzLWFwLW5vcnRoZWFzdC0xLmFtYXpvbmF3cy5jb20lMkZxaWl0YS1pbWFnZS1zdG9yZSUyRjAlMkYyODQyODEyJTJGZDllYmE0YjY1ZDE4ZjAwNjNhZTQzYTI1MTc3ODU1OTIxMTFhNDg0MSUyRnhfbGFyZ2UucG5nJTNGMTY2Mzc3MzUyMz9peGxpYj1yYi00LjAuMCZhcj0xJTNBMSZmaXQ9Y3JvcCZtYXNrPWVsbGlwc2UmZm09cG5nMzImcz1iZDNmM2UxZDA2NzJlZDg1YzBiNjMxNTVhMjlkZmYxYw%26blend-x%3D120%26blend-y%3D467%26blend-w%3D82%26blend-h%3D82%26blend-mode%3Dnormal%26s%3D7325d33a7290e82d9732f652f18d20d3?ixlib=rb-4.0.0&w=1200&fm=jpg&mark64=aHR0cHM6Ly9xaWl0YS11c2VyLWNvbnRlbnRzLmltZ2l4Lm5ldC9-dGV4dD9peGxpYj1yYi00LjAuMCZ3PTk2MCZoPTMyNCZ0eHQ9QVdTJTIwU0VTJUUzJTgyJTkyJUU0JUJEJUJGJUUzJTgxJUEzJUUzJTgxJUE2JUU2JUI3JUJCJUU0JUJCJTk4JUUzJTgzJTk1JUUzJTgyJUExJUUzJTgyJUE0JUUzJTgzJUFCJUU0JUJCJTk4JUUzJTgxJThEJUUzJTgxJUFFJUUzJTgzJUExJUUzJTgzJUJDJUUzJTgzJUFCJUUzJTgyJTkyJUU5JTgwJTgxJUU0JUJGJUExJUUzJTgxJTk5JUUzJTgyJThCJnR4dC1hbGlnbj1sZWZ0JTJDdG9wJnR4dC1jb2xvcj0lMjMxRTIxMjEmdHh0LWZvbnQ9SGlyYWdpbm8lMjBTYW5zJTIwVzYmdHh0LXNpemU9NTYmdHh0LXBhZD0wJnM9MDNmZDVkN2Q5NzdmMDRlOGU3OGJhYzUyNzE3MGQ5MzE&mark-x=120&mark-y=112&blend64=aHR0cHM6Ly9xaWl0YS11c2VyLWNvbnRlbnRzLmltZ2l4Lm5ldC9-dGV4dD9peGxpYj1yYi00LjAuMCZ3PTgzOCZoPTU4JnR4dD0lNDBzdXNoaXRhcm8wMzEwJnR4dC1jb2xvcj0lMjMxRTIxMjEmdHh0LWZvbnQ9SGlyYWdpbm8lMjBTYW5zJTIwVzYmdHh0LXNpemU9MzYmdHh0LXBhZD0wJnM9NzdlYjE1OTJlM2RmMmQyMTljNzUwZWViOTczYWIzNjc&blend-x=242&blend-y=480&blend-w=838&blend-h=46&blend-fit=crop&blend-crop=left%2Cbottom&blend-mode=normal&s=19e32edbb3efde32e5017b43b60ee17d
```

### 案3: 他のメール送信サービス（SendGrid等）

**概要**: SES以外のメール送信サービスを利用

**メリット**:
- 高機能なAPI
- 詳細な送信統計

**デメリット**:
- コスト増加
- AWS環境から離れる
- 新たなサービス管理が必要
## 🚀 決定

**日報メール送信機能には Nodemailer + Amazon SES の組み合わせを採用する。**

具体的な実装方針：

- NodemailerのSESトランスポートを使用してメール送信を実装する
-  添付ファイルはNodemailerの標準機能で処理し、MIMEメッセージの手動構築は行わない
- SESの送信制限（40MB）とスロットリング（10MB）の監視機能を実装する
- (環境別設定により、開発環境ではSMTP、本番環境ではSESを使用可能とする)
	- -> すべての環境で同様の構成とする
- エラーハンドリングとリトライ機能を含む統合的なメール送信サービスを構築する
## 🪞 結果・影響

### プラスの結果

**開発効率の向上**

- 添付ファイル対応の実装時間を大幅短縮（手動MIME構築が不要）
- エラーハンドリングのロジックが簡潔になる
- テスト実装が容易（モックやテスト用SMTPが使用可能）

**保守性の向上**

- Nodemailerの豊富なドキュメントとコミュニティサポート
- 標準的なメール送信処理のパターンに従える
- 環境別設定により開発・本番の切り替えが簡単

**機能性の向上**

- 複雑な添付ファイル形式にも対応可能
- 送信エラーの詳細な情報取得
- SES統計情報との連携

### マイナスの結果

**依存関係の増加**

- Nodemailerライブラリへの依存が追加される
- ライブラリのバージョン管理が必要

**わずかなオーバーヘッド**

- AWS SDK直接利用と比較してわずかな処理オーバーヘッド
- メモリ使用量の若干の増加

### 中立的な影響

**アーキテクチャへの影響**

- メール送信機能がNodemailerのインターフェースに準拠
- AWS以外のメール送信サービスへの将来的な移行も容易
- ドメイン層とインフラ層の分離が明確になる

## 🍜 今後の検討事項

### 短期的な検討事項（1-2ヶ月以内）

**送信監視とアラート**

- SES送信制限の監視とアラート機能の実装
- 送信失敗時の通知システム
- 送信統計の可視化

**エラーハンドリングの詳細化**

- 一時的エラーと永続的エラーの分類
- リトライ戦略の詳細設計
- 部分的送信失敗時の処理方針

### 中期的な検討事項（3-6ヶ月以内）

**パフォーマンス最適化**

- 大量送信時のバッチ処理戦略
- 添付ファイルサイズによる送信方法の最適化
- メール送信キューイング機能の検討

**セキュリティ強化**

- 送信ドメイン認証（SPF、DKIM、DMARC）の設定
- 送信者権限の詳細な制御
- 添付ファイルのウイルススキャン連携

### 長期的な検討事項（6ヶ月以上）

**スケーラビリティ対応**

- 複数AWSリージョンでのSES利用
- 送信量増加に対するコスト最適化
- 他のメール送信サービスとのハイブリッド構成

**機能拡張**

- テンプレートエンジンとの連携
- メール送信の予約機能
- 受信確認・開封確認機能

**代替手段の検討**

- 大容量ファイル送信時のファイル共有リンク生成


