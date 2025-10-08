---
type: note
moc: "[[🏷️ADR]]"
tags: 
下書き: true
提案済み: 
承認済み: 
代替済み: 
非推奨: 
棄却:
---

## アプリケーションログ・メトリクス基盤 提案書

### 1. はじめに

本提案書は、Lambdaで稼働するアプリケーションのログおよびメトリクス基盤について、AWS CloudWatch、Datadog、New Relicを比較検討する。各ツールの特性を理解し、最適な基盤選択の一助とすることを目的とする。

### 2. 現状と要件

Lambdaベースのアプリケーションは、従来のサーバー型アーキテクチャとは異なる監視アプローチを必要とする。分散されたコンポーネントからのログの集約、メトリクスの可視化、アラート設定を効率的に行う基盤の構築が求められる。

### 3. ログ・メトリクス基盤 比較検討

| 項目         | AWS CloudWatch                                                                                                                    | Datadog                                                                                                                | New Relic                                                                                                              |
| :--------- | :-------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------- |
| **提供形態**   | AWSサービスとして提供                                                                                                                      | SaaSとして提供                                                                                                              | SaaSとして提供                                                                                                              |
| **統合性**    | AWSサービスとの**ネイティブな統合**を強みとする（Lambda, EC2, S3など）                                                                                    | 豊富なAWSインテグレーションに加え、多数のOSS/SaaSとの統合。**広範なエコシステム**に対応                                                                    | 豊富なAWSインテグレーションに加え、多数のOSS/SaaSとの統合。**幅広い技術スタック**に対応                                                                    |
| **機能**     | - ログ収集（CloudWatch Logs）<br>- メトリクス収集（CloudWatch Metrics）<br>- ダッシュボード<br>- アラーム<br>- Logs Insights (ログ分析)<br>- ServiceLens (トレース) | - ログ収集・分析<br>- メトリクス収集・可視化<br>- **APM (分散トレーシング)**<br>- **インフラ監視**<br>- **リアルユーザーモニタリング (RUM)**<br>- **シンセティックモニタリング** | - ログ収集・分析<br>- メトリクス収集・可視化<br>- **APM (分散トレーシング)**<br>- **インフラ監視**<br>- **リアルユーザーモニタリング (RUM)**<br>- **シンセティックモニタリング** |
| **トレース**   | ServiceLensによる分散トレース機能を提供。主にAWSサービス間の可視化に強み                                                                                       | **広範な言語・フレームワーク**に対応した**詳細な分散トレーシング**機能を提供。**アプリケーション内部のボトルネック特定に優れる**                                                 | **広範な言語・フレームワーク**に対応した**詳細な分散トレーシング**機能を提供。**アプリケーション内部のボトルネック特定に優れる**                                                 |
| **コスト**    | 従量課金制を採用（データ取り込み量、保存期間、クエリ量などに依存）                                                                                                 | ユーザー数、ホスト数、データ量などに基づく**多様な課金モデル**                                                                                      | ユーザー数、データ量などに基づく**多様な課金モデル**                                                                                           |
| **セットアップ** | AWSコンソール、CLI、CDKなどを利用し、**比較的容易に設定可能**                                                                                             | エージェント導入など、**初期設定にある程度の工数**を要する                                                                                        | エージェント導入など、**初期設定にある程度の工数**を要する                                                                                        |
| **学習コスト**  | AWSのエコシステムに対する理解が前提となるが、**既存AWSユーザーには学習コストが低い**                                                                                   | 独自のUIや概念を習得する必要があり、**新規ユーザーには学習コスト**が発生                                                                                | 独自のUIや概念を習得する必要があり、**新規ユーザーには学習コスト**が発生                                                                                |


### 4. 各ツールの特性と考慮事項

#### 4.1. AWS CloudWatch

CloudWatchはAWSが提供する監視サービスであり、Lambdaを含むAWSのサービスと**緊密に連携**する。Lambdaファンクションのログは自動的にCloudWatch Logsに送信され、メトリクスも自動的にCloudWatch Metricsに発行されるため、**セットアップの手間が少ない**。ログ分析のためのLogs Insightsや、分散トレースのためのServiceLensといった機能も内包する。**ServiceLensは主にAWSサービス間の連携を可視化するのに有効**だ。コストは従量課金であり、利用量に応じた費用が発生する。AWSの既存エコシステムに深く組み込まれているため、既にAWSを利用しているチームにとっては学習コストが低い。

#### 4.2. Datadog

Datadogは、豊富なインテグレーションと包括的な監視機能を提供するSaaS型ツールである。ログ管理、メトリクス可視化に加え、**APM、インフラ監視、RUM、シンセティックモニタリング**など、多岐にわたる監視要件に対応できる。特に**分散トレーシングに関しては、広範な言語やフレームワークに対応し、アプリケーション内部の処理フローやボトルネックを詳細に可視化できる**点が強みだ。AWSインテグレーションも強力であり、Lambdaのメトリクスやログも収集可能だ。ただし、SaaSモデルのため、コストは利用規模に応じて変動し、特に高度な機能を利用する場合は高額になる可能性がある。エージェント導入など、初期設定には一定の工数がかかる。

#### 4.3. New Relic

New RelicもDatadogと同様に、APMを中核とする包括的な監視機能を提供するSaaS型ツールである。ログ管理、メトリクス監視に加え、**APM、インフラ監視、RUMなど、アプリケーション全体のパフォーマンスを包括的に把握**できる。**分散トレーシング機能もDatadogと同様に充実しており、アプリケーション内部の深いレベルでの問題特定に貢献する**。AWSインテグレーションも充実しており、Lambdaの監視にも対応する。Datadogと同様、SaaSモデルであり、高度な機能を利用するにつれてコストは増加する傾向にある。

---

### 5. 将来的なDatadogへの移行パス

現状、AWS CloudWatchを基盤として採用した場合でも、将来的にDatadogのようなより高度な監視ツールへの移行は**十分可能**である。

- **ログ転送の柔軟性**: CloudWatch LogsからDatadog Logsへのログ転送は、Lambdaを利用して容易に実現できる。CloudWatch Logsのサブスクリプションフィルターを設定し、Datadogが提供するLog Forwarder Lambda関数をトリガーすることで、リアルタイムにログをDatadogに連携できる。
- **メトリクス収集の連携**: CloudWatch Metricsで収集されたカスタムメトリクスや、AWSサービスが発行するメトリクスも、Datadogの**AWSインテグレーション**を通じてDatadogに収集可能。
- **トレースデータの連携**: AWS X-Ray（CloudWatch ServiceLensの基盤）で収集したトレースデータをDatadogに転送することも可能だが、Datadogが提供する専用のトレーシングライブラリをアプリケーションに組み込むことで、**より詳細で一貫性のあるトレースデータ**を直接Datadogに送信できる。これにより、将来的な移行を見据えた設計が可能となる。
- **オープンスタンダードへの対応**: DatadogはOpenTelemetryなど、業界標準のテレメトリーデータ収集に対応しているため、将来的に多様なデータソースからの統合監視も視野に入れられる。

CloudWatchで必要十分な監視を実現しつつ、アプリケーションの成長や監視要件の高度化に応じて、Datadogへの段階的な移行を検討できる柔軟性がある。

---

### 6. APM/トレースについて（OpenTelemetryの採用と将来的な移行パス）

※トレースは今回のスコープではないがツール選定に関わるため記載する。

アプリケーションのオブザーバビリティ戦略において、**OpenTelemetry**の採用は非常に有効な選択肢である。OpenTelemetryは、ベンダーニュートラルなテレメトリーデータ（トレース、メトリクス、ログ）の収集とエクスポートのためのオープンスタンダードである。

- **ベンダーロックインの回避**: OpenTelemetry SDKをアプリケーションに組み込むことで、特定の監視ツールに依存することなくテレメトリーデータを生成できる。これにより、将来的に監視ツールを変更する際にも、アプリケーションコードへの大きな変更なしに移行が可能となる。
- **データ収集の一元化**: ログ、メトリクス、トレースといった異なる種類のテレメトリーデータを単一のSDKで収集し、一貫した方法で管理できる。
- **拡張性**: OpenTelemetryコレクターを利用することで、データのフィルタリング、集計、複数のバックエンドへのルーティングなど、高度なデータ処理が可能となる。

#### 6.1. CloudWatchとOpenTelemetry

AWS X-Ray（ServiceLensの基盤）は、OpenTelemetryプロトコルをサポートしており、OpenTelemetry SDKから生成されたトレースデータをX-Rayに送信できる。これにより、アプリケーションコードにOpenTelemetryを導入しつつ、CloudWatch ServiceLensで基本的なトレース可視化を行うことが可能となる。
(補足) APMについてはApplication Signalsという新サービスもあるがX-Rayとの棲み分けがよくわかっていないので要調査。

```cardlink
url: https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/monitoring/CloudWatch-Application-Monitoring-Sections.html
title: "Application Signals - Amazon CloudWatch"
description: "Application Signals、Synthetics、Evidently、および CloudWatch RUM などの CloudWatch の機能に関するセクションが含まれています。Application Signals を有効にする方法についての情報が含まれています。"
host: docs.aws.amazon.com
favicon: https://docs.aws.amazon.com/assets/images/favicon.ico
```

```cardlink
url: https://qiita.com/AoTo0330/items/4d3cf0f6126f1a2a76c5
title: "Amazon CloudWatch Application Signals 徹底解説 - Qiita"
description: "はじめにこんにちは、Datadog Japan で Sales Engineer をしている AoTo(@AoTo0330) です。この投稿は Japan AWS Jr. Champions A…"
host: qiita.com
favicon: https://cdn.qiita.com/assets/favicons/public/production-c620d3e403342b1022967ba5e3db1aaa.ico
image: https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-user-contents.imgix.net%2Fhttps%253A%252F%252Fcdn.qiita.com%252Fassets%252Fpublic%252Fadvent-calendar-ogp-background-7940cd1c8db80a7ec40711d90f43539e.jpg%3Fixlib%3Drb-4.0.0%26w%3D1200%26blend64%3DaHR0cHM6Ly9xaWl0YS11c2VyLXByb2ZpbGUtaW1hZ2VzLmltZ2l4Lm5ldC9odHRwcyUzQSUyRiUyRmF2YXRhcnMuZ2l0aHVidXNlcmNvbnRlbnQuY29tJTJGdSUyRjY4MDYyMDM1JTNGdiUzRDQ_aXhsaWI9cmItNC4wLjAmYXI9MSUzQTEmZml0PWNyb3AmbWFzaz1lbGxpcHNlJmZtPXBuZzMyJnM9MTM0ZDlmNDE4MTdmZTFkZjlmYWNhZWRmZGMwNTA5M2I%26blend-x%3D120%26blend-y%3D462%26blend-w%3D90%26blend-h%3D90%26blend-mode%3Dnormal%26mark64%3DaHR0cHM6Ly9xaWl0YS1vcmdhbml6YXRpb24taW1hZ2VzLmltZ2l4Lm5ldC9odHRwcyUzQSUyRiUyRnMzLWFwLW5vcnRoZWFzdC0xLmFtYXpvbmF3cy5jb20lMkZxaWl0YS1vcmdhbml6YXRpb24taW1hZ2UlMkZiYzg1OTViYWYyYzUzZjZhYjUyMjRiYjY2YjlmYTY3Mjc1OTk4YzA2JTJGb3JpZ2luYWwuanBnJTNGMTczNDc4NjM1MD9peGxpYj1yYi00LjAuMCZ3PTQ0Jmg9NDQmZml0PWNyb3AmbWFzaz1jb3JuZXJzJmNvcm5lci1yYWRpdXM9OCZib3JkZXI9MiUyQ0ZGRkZGRiZmbT1wbmczMiZzPTQ5NWNlMzhhYjE4MGQ3YTAyZjExNDRlZmQ4YjY3NWYw%26mark-x%3D186%26mark-y%3D515%26mark-w%3D40%26mark-h%3D40%26s%3D954aef31675c33f8b19fd13409611c40?ixlib=rb-4.0.0&w=1200&fm=jpg&mark64=aHR0cHM6Ly9xaWl0YS11c2VyLWNvbnRlbnRzLmltZ2l4Lm5ldC9-dGV4dD9peGxpYj1yYi00LjAuMCZ3PTk2MCZoPTMyNCZ0eHQ9QW1hem9uJTIwQ2xvdWRXYXRjaCUyMEFwcGxpY2F0aW9uJTIwU2lnbmFscyUyMCVFNSVCRSVCOSVFNSVCQSU5NSVFOCVBNyVBMyVFOCVBQSVBQyZ0eHQtYWxpZ249bGVmdCUyQ3RvcCZ0eHQtY29sb3I9JTIzM0EzQzNDJnR4dC1mb250PUhpcmFnaW5vJTIwU2FucyUyMFc2JnR4dC1zaXplPTU2JnR4dC1wYWQ9MCZzPTZjNWFmZGNkNmM4NWZmOWVlNDJhNGMyNGMwY2NmMWMy&mark-x=120&mark-y=112&blend64=aHR0cHM6Ly9xaWl0YS11c2VyLWNvbnRlbnRzLmltZ2l4Lm5ldC9-dGV4dD9peGxpYj1yYi00LjAuMCZ3PTgzOCZoPTU4JnR4dD0lNDBBb1RvMDMzMCZ0eHQtY29sb3I9JTIzM0EzQzNDJnR4dC1mb250PUhpcmFnaW5vJTIwU2FucyUyMFc2JnR4dC1zaXplPTM2JnR4dC1wYWQ9MCZzPTQyODhmMmE2MzNjZGNjZDlmN2MyNmZlNmRhZmY3NzRj&blend-x=242&blend-y=454&blend-w=838&blend-h=46&blend-fit=crop&blend-crop=left%2Cbottom&blend-mode=normal&txt64=RGF0YWRvZw&txt-x=242&txt-y=539&txt-width=838&txt-clip=end%2Cellipsis&txt-color=%233A3C3C&txt-font=Hiragino%20Sans%20W6&txt-size=28&s=89b52a746cea077cec7f3701ad8e4b8a
```


#### 6.2. DatadogとOpenTelemetry

DatadogはOpenTelemetryの主要な貢献者であり、OpenTelemetryからのデータ収集に積極的に対応している。OpenTelemetryコレクターを介して、トレース、メトリクス、ログの全てをDatadogに送信し、Datadogの強力なAPM機能やダッシュボードで統合的に分析できる。これにより、**CloudWatchからDatadogへの移行時も、OpenTelemetryを採用していればアプリケーションコード側の変更を最小限に抑えつつ、Datadogの高度なAPM機能を最大限に活用できる**。

#### 6.3. 移行パスの示唆

初期段階では、AWS CloudWatchとX-Ray（ServiceLens）で基本的な監視をスタートし、**アプリケーションコードにはOpenTelemetry SDKを導入する**ことを推奨する。これにより、AWSサービス間のトレース可視化を行いつつ、将来的な要件の変化に備えることができる。

アプリケーションの複雑化や、より詳細なAPM機能（コードレベルのパフォーマンス分析、高度なサービスマップ、深掘りされたエラー分析など）が必要になった際には、OpenTelemetryコレクターの設定を変更するだけで、**Datadogにスムーズにテレメトリーデータを転送し、高度な監視機能へとアップグレードすることが容易に可能**となる。このアプローチは、初期投資を抑えつつ、将来の拡張性とベンダー選択の柔軟性を最大化する。
### 7. まとめ

Lambdaアプリケーションのログ・メトリクス基盤として、**AWS CloudWatch**はAWSサービスとのネイティブな統合、コスト効率、そして必要十分な機能を提供する点で、**初期導入の選択肢として最適**である。特に、AWSサービス間の連携監視においてはその強みを発揮する。将来的な監視要件の高度化、特に**アプリケーション内部の詳細なトレースやAPM機能**が必要になった際には、Datadogのようなツールが有効な選択肢となり、スムーズな移行パスも確保されているため、柔軟な拡張性を有する。
