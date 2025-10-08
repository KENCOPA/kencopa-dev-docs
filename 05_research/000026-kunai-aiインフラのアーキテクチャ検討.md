---
type: note
tags:
- '#ai/ref'
---
# 📜 文脈・背景

11月のリリースに向けてkunai-aiのデプロイが必要となっている。現状、FastAPIでbackground taskを使い非同期処理を実現しているが、工程表生成の処理は重たく15分以上かかる可能性がある。

運用面を加味した最小構成のアーキテクチャを決定する必要がある。

## 技術的制約
- FastAPIのbackground taskを使用した非同期処理
- 工程表生成処理が15分以上かかる可能性
- 11月リリースという短期間でのデプロイ要求

## 懸念事項の検証結果
- **Lambda + background task**: レスポンスを返した時点でLambda関数は終了するため、FastAPIのBackground Taskはサポートされない
  - 参考: https://github.com/fastapi/fastapi/issues/1233
  - このissueの作者はSQSにメッセージを送信し、新しいLambdaを起動することで解決している
- **Lambda実行時間制限**: Lambdaの最大実行時間は15分であり、工程表生成処理には不十分
  - 参考: https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/configuration-timeout.html
- **ECSスケーリング**: スケールイン時にグレースフルシャットダウンが必要。ECSはSIGTERM受信後、デフォルト30秒（最大120秒）でSIGKILLを送信するため、15分かかる処理では強制終了のリスクがある
  - stopTimeout: https://arc.net/l/quote/tfigvzih
  - task lifecycle: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-lifecycle-explanation.html

# 🎨 対応案

## 案1. Lambda
- kunai-core-backendの実装踏襲
- **問題点**: background taskがサポートされない

## 案2. Lambda + SQS  
- background taskを自前実装に変更
- **問題点**: Lambdaの15分実行時間制限により工程表生成処理に対応できない

## 案3. Lambda + SQS + ECS/Batch
- 非同期タスクの実行のみ長時間実行可能なコンピューティングリソースで実行
- **Pros**: 長期的に安定した構成
- **Cons**: 管理するインフラリソースが増加、アプリケーション実装の修正が必要

## 案4. ECS
- FastAPI with background taskをECSで実行
- **Pros**: 既存実装パターンを維持可能
- **Cons**: スケールイン時の処理中断リスク

# 🚀 決定

**kunai-aiのデプロイメントアーキテクチャにはECSを採用する**

以下の理由により案4（ECS）を選択する

1. Background Taskを使用することができるため、アプリケーションの実装を変更する必要がない（※コンテナ化に伴う変更は必要）
2. ECSに載せることで、将来案3を採用する際の知見が蓄積される（backendもECSへの移行を検討している）
3. 11月リリースが最優先であり、案の中では一番短期間での対応が可能
4. 初期段階では利用頻度・アクセス量が多くないため、スケールインやタスク入れ替えによるエラーが致命的ではない。また非同期処理のためクライアント側でリトライが可能、元から時間がかかる処理のためユーザー体験への影響は少ない

# 🪞 結果・影響

## プラスの結果
- 11月リリースに向けた迅速な対応が可能
- 既存のFastAPI + background task実装を維持
- ECSの運用知見が蓄積される
- 将来的な案3への移行時の基盤となる

## マイナスの結果・リスク
- スケールイン時やFargate platform version更新時の処理中断リスク
- 長時間処理の途中切断可能性
- 初期フェーズでのスケーリング制限

## 軽減策
- 初期フェーズではスケールしない構成を採用
- クライアント側でのリトライ機構実装
- stopTimeoutの最大値（120秒）設定を検討

# 🍜 今後の検討事項

1. **長期的アーキテクチャ移行**: 案3（Lambda + SQS + ECS/Batch）への移行時期とロードマップの策定
2. **モニタリング強化**: 処理中断やエラー率の監視体制構築
3. **グレースフルシャットダウン**: 長時間処理に対する適切な終了処理の実装
4. **スケーリング戦略**: 利用頻度増加に伴うスケーリング戦略の再検討
5. **コード分割戦略**: domain modelなど共有ソースコードの分割方針（将来の案3移行時に必要）