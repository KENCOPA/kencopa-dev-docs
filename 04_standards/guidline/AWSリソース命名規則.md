---
type: note
tags:
- ai/ref
---
# 命名規則の目的

このガイドラインは以下の目標を達成するために策定されています：

- **一目での識別**
    - リソース名だけで環境、プロダクト、サービス機能、リソースタイプが即座に判別できる
- **一貫性の確保**
    - すべてのリソースが統一された命名体系に従うことで管理が容易になる
- **チーム自律性の維持**
    - マイクロサービスアーキテクチャにおいて、各チームが自律的に働ける体制をサポート
- **組織変更への耐性**
    - リポジトリの分割・統合やチーム構成の変更に対して柔軟に対応できる設計
- **自動化の促進**
    - 規則的な命名によりリソースの自動検索・操作が可能になる

# 基本原則

- すべてのリソース名は英数字小文字とハイフン(-)のみを使用する
    - _理由_:
        - AWS サービス間で一貫して使用可能な文字セットを採用することで互換性を確保
        - ほぼ全ての AWS サービスではリソース名にハイフンを許容する。一方で、アンダースコアを許容しない WebACL のようなサービスがある
        - 環境名、サービス名などの単位で区切りを明確にできる
- リソース名は一貫したパターンに従う
    - _理由_:
        - 予測可能な命名によりリソースの検索と理解が容易になる
- 名前は短く、かつ説明的であること
    - _理由_:
        - 多くのAWSサービスには名前の長さ制限があり、短縮しつつも意味を失わない名前が必要
- AWS サービス名をリソース名に含めない
    - _理由_:
        - サービス名は AWS コンソールやツールですでに表示されており、重複を避け名前を短くする

# 命名パターン(リソース名)

## **基本パターン**

---

- リソース名
    - `{env}-{system-name}-{service-domain}[-{service-sub-domain}]-{usage}[-{resource-specific}][-{resource-type}]`

## 各要素の説明

---

### `{env}`（環境）

_**Required: Yes**_

| 値      | 説明           | AWSアカウント     | 説明             |
| ------ | ------------ | ------------ | -------------- |
| `prod` | 本番環境         | 本番アカウント      | 実際のユーザーが利用する環境 |
| `stg`  | ステージング環境     | ステージングアカウント  | 本番リリース前の検証環境   |
| `dev`  | 開発/サンドボックス環境 | sandboxアカウント | 開発者が利用するテスト環境  |

_理由_: 環境を先頭に配置することで、環境ごとのリソースをすぐに区別でき、フィルタリングが容易になる

### `{system-name}`（システム名）

_**Required: Yes**_

実際のプロダクト名を識別できる値を指定します。

|値|プロダクト名|備考|
|---|---|---|
|`azuchi`|Kencopa||

_理由_: プロジェクト/プロダクトの所有権を明確にし、複数のシステムが存在する環境でも識別が容易になる

### `{service-domain}`（サービスドメイン）

_**Required: Yes**_

| 値          | 説明          | 備考                                       |
| ---------- | ----------- | ---------------------------------------- |
| `core`     | コア機能        | 基本的なユーザー管理や業務ロジック、認証などの中核機能              |
| `safegen`  | 安全生成機能      | Retrieval-Augmented Generation による安全生成機能 |
| `weather`  | 天気情報関連      | 天気予報や気象情報に関連する機能                         |
| `ai-agent` | AIエージェント機能  |                                          |
| `chat`     | チャット機能      |                                          |
| `common`   | 全サービスドメイン共通 | すべてのサービスドメインに関連するリソースなど                  |

- DNS, IAM Role(Terraform, GH Actionsなど）… | | `ops` | 運用機能関連 | サービス起動停止やヘルスチェックなど |

_理由_: マイクロサービスの境界とチーム責任範囲を明確にし、リポジトリ構成の変更に影響されない識別子を提供

### `{service-sub-domain}`（サービスサブドメイン）

_**Required: No**_

同じドメイン内で、サービスドメインをさらに分割したい場合のみ指定します。

|値|説明|説明|
|---|---|---|
|service-sub-domain|サービスサブドメイン名||

### `{usage}`（用途）

_**Required: Yes**_

|要素|説明|例|
|---|---|---|
|**usage**|具体的な用途や機能を表す識別子|`cognito-custom-message-job` , `file-export-job` , `api`, `app`など|

_理由_: 同じタイプのリソースでも用途によって区別し、具体的な役割を明示

例：

subnet

- `...-api-`
    - APIリソース用のサブネット
- `...-db-`
    - DBリソース用のサブネット

### `{resource-specific}`（リソース固有識別子）

_**Required: No**_

|要素|説明|例|
|---|---|---|
|**resource-specific**|（オプション）リソース固有の識別子|`primary`, `replica`, `public-a`, `private-b` など|

_理由_: 同一環境・同一プロダクト・同一ドメイン・同一タイプ・同一用途のリソースが複数存在する場合の識別子として機能

**アベイラビリティゾーン:**

AZ 名にはリージョンコードを含めず、末尾のアルファベットだけとする。

|**AZ ID**|**Identifier**|
|---|---|
|ap-northeast-1a|a|
|ap-northeast-1c|c|
|ap-northeast-1d|d|

- 利用可能な文字: `[a-d]{1}`

**アクセス修飾子:**

VPC のサブネットは、パブリックサブネットの場合インターネットに直接アクセスできる。パブリックサブネットを区別したい場合はリソース名にアクセス修飾子を付与する。

|**Name**|**Identifier**|
|---|---|
|パブリックサブネット|public|
|プライベートサブネット|private|

### `{resource-type}` (リソース種別)

_**Required: YES**_

そのリソースの役割・または技術的機能を指定します。

|値|説明|説明|
|---|---|---|
|resource-type|リソース種別|`auth`, `main`, `search`, `log`, `backup` など|

例:

- Lambda関数
    - `...-main`
        - その用途のメイン機能の処理
    - `...-dlq`
        - その用途のDLQを処理する機能

## 命名例

---

### azuchi-backendリポジトリの例

命名例は以下のようになる。

基本：

- サービスドメイン : `core`
- リソース種別 : `api`
    - apiとしての機能を提供するため

1. **Lambda関数**:
    - `{env}-{system-name}-core-{usage}-{resource-type}`
    - 例:
        - `prod-azuchi-core-api-main` : オンライン処理
        - `prod-azuchi-core-api-auth` : 認証系
    - ここで「core」はサービスドメイン、「api」は用途、「user」や「auth」はリソース種別を表します
2. **IAMロール**:
    - `{env}-{system-name}-core-{usage}`
    - 例:
        - `prod-azuchi-core-api-main`
    - Lambda関数`prod-azuchi-core-api-main` のIAM Roleであることがわかります
3. **CloudWatch ロググループ**:
    - `{env}-{system-name}-core-api-{usage}`
    - 例:
        - `prod-azuchi-core-api-user`
    - Lambda関数`prod-azuchi-core-api-main` のロググループであることがわかります
4. **API Gateway**:
    - `{env}-{system-name}-core-api-{用途}`
    - 例:
        - api: 　`prod-azuchi-core-api-main`
        - api stage blue : `prod-azuchi-core-api-main-blue`
        - api stage green: `prod-azuchi-core-api-main-green`
    - オンライン処理に使われるAPIGatewayということがわかります
    - b/gデプロイメントなどでステージを複数作成する場合はそれぞれのステージ名が明確になります

# 命名パターン（SSM Parameter Store)

## **基本パターン**

---

- リソース名
    - `{env}**/**{system-name}/{service-domain}[/{service-sub-domain}][/{resource-type}]/{aws-service-name}/{resource-usage-name}[/{additional-info}]/{value-type}`

### **`{aws-service-name}` (AWSサービス名)**

- S3, Cognito, Lambda Functionなど、参照するAWSサービスの名前
    - なるべくサービスの正式名書で書く
        - ×lambda, ⭕️lambda-function
- これにより、パラメータがどのAWSサービスに関連するかが明確になります

### `{resource-usage-name}` (リソース名）

- S3バケット名など、参照するAWSサービスのリソース用途名
- 何のリソースか識別できれば良い

### `{additional-info}` (追加情報)

- リソース名に追加情報を加えたい場合など
    - 例
        - …rds/main/writer/url
        - …rds/main/reader/url

### **`{value-type}`**

- id, name, arn, urlなど、格納されている値の種類
- 同じリソースでも異なる識別子を保存することが多いため、この要素が重要です

# マイクロサービスとリポジトリの対応関係

命名規則の中で導入した `service-domain` は、マイクロサービスの境界とチーム責任範囲を表します。これにより、リポジトリ構成が変更されても、リソース名から機能領域の識別が可能になります。

## 現在のリポジトリとサービスドメインの対応

---

| リポジトリ名                                | サービスドメイン   | リソース名パターン                          | 管理するリソース例                                                         | 備考                         |
| ------------------------------------- | ---------- | ---------------------------------- | ----------------------------------------------------------------- | -------------------------- |
| **azuchi-infra-v2**                   | `core`     | `{env}-azuchi-core-*`              | VPC, Route53, Aurora, Subnet, Security Group, Amplify, Cognito, … |                            |
| **azuchi-frontend**                   | `core`     | `{env}-azuchi-core-*`              | …                                                                 |                            |
| **azuchi-backend**                    | `core`     | `{env}-azuchi-core-*`              | Lambda Function, APIGW, IAM, CW Log Group, …                      |                            |
| rag-backend                           |            |                                    |                                                                   |                            |
| (→ azuchi-safegen-api)                | `safegen`  | `{env}-azuchi-safegen-*`           | Lambda Function, APIGW, IAM, CW Log Group, …                      |                            |
| azuchi-safegen-infra                  | `safegen`  | `{env}-azuchi-safegen-*`           | Bedrock, Pinecorn, …                                              |                            |
| **azuchi-weather-api**                | `weather`  | `{env}-azuchi-weather-*`           | Lambda Function, APIGW, IAM, DynamoDB, …                          |                            |
| **azuchi-file-export-job**            | `core`     | `{env}-azuchi-core-*`              | SQS, Lambda Function, IAM, …                                      |                            |
| **azuchi-cognito-custom-message-job** | `core`     | `{env}-azuchi-core-*`              | Lambda Function, IAM, …                                           |                            |
| azuchi-ops                            | `core`     | `{env}-azuchi-ops-*`               | Lambda Function, IAM, EventBridge Rule, …                         |                            |
| azuchi-common-aws-gh-actions-linkage  | `common`   | `{env}-azuchi-common-gh-actions-*` | IAM, …                                                            | 各リポジトリのgh actionsのaws権限を管理 |
| azuchi-common-infra                   | `common`   | `{env}-azuchi-common-infra-*`      | Route53, …                                                        | 各環境ごとのサブドメインを管理            |
| azuchi-ai-agent                       | `ai-agent` | `{env}-azuchi-ai-agent-`           |                                                                   |                            |
| azuchi-chat                           | `chat`     | `{env}-azuchi-chat-`               |                                                                   |                            |

## リポジトリ構成変更への対応

---

将来的なリポジトリ構成の変更に柔軟に対応できるよう、以下の原則を定めます：

1. **サービスドメイン中心の組織**: リソースはまず第一にサービスドメインで分類
    - リポジトリがどう変わっても、サービスドメインを維持することで一貫性を確保
2. **リポジトリ分割の場合**:
    - 同一サービスドメイン内でリソースタイプ別に分割可能
    - 例:
        - `rag-backend` を `azuchi-rag-api` と `azuchi-rag-infra` というリポジトリに分割しても、両リポジトリのリソースは `{env}-azuchi-rag-*` というパターンを保持
3. **リポジトリ統合の場合**:
    - 複数のサービスドメインを1つのリポジトリで管理することが可能
    - 例:
        - 現在別々の`azuchi-file-export-job`と`azuchi-email-message-job`リポジトリを`azuchi-background-jobs`という1つのリポジトリに統合した場合でも、リソース名はそれぞれ`{env}-azuchi-core-*`と`{env}-azuchi-core-*`のまま変更しない
4. **新サービスの追加**:
    - 新しいサービスドメインを導入する場合は、新しいドメイン識別子を定義
    - 例:
        - 支払い処理サービスを追加する場合、`payment` ドメインを新設

## マイクロサービス化の進展に応じた発展

---

マイクロサービスアーキテクチャへの移行が進むにつれ、以下のように命名規則を発展させることが可能です：

1. **チーム識別子とサービスドメインの連携**:
    - 各チームが担当するサービスドメインを明確に定義
    - チーム再編があっても、サービスドメインは維持
2. **マルチサービスドメイン対応**:
    - 一部のリソースが複数のドメインにまたがる場合、`shared` や `common` などのドメインを使用
    - 例: 複数サービスで共有するデータベース: `prod-azuchi-shared-db-multiservice`
3. **サービスドメイン細分化**:
    - 必要に応じてサブドメインを導入可能
    - パターン: `{env}-{system-name}-{domain}-{subdomain}-{resource-type}-{usage}`
    - 例: `prod-azuchi-core-authentication-api-oauth`

# タグ付けポリシー

リソース名に加えて、すべてのリソースに以下のタグを付与することを必須とします：

- `ManagedBy`: 管理ツール（Terraform, CloudFormation, Manual）
- `Owner`: 担当チームまたは担当者
- `Repository`: 管理リポジトリ名

タグ付けの理由：

- リソース名だけでは伝えきれない詳細情報を提供
- コスト配分やリソース管理を効率化
- Infrastructure as Code (IaC) ツールとの相互運用性確保
- 監査やコンプライアンス要件の充足

## Owner一覧

---

|name|保持するサービスドメイン|description|
|---|---|---|
|team1|core, weather||
|team2|safegen||
|central|common|中央管理者|
|組織横断リソースを管理|||

# 実装上の注意事項

- **AWS サービス固有の制限**:
    
    - 各サービスには命名規則の制限があるため、必要に応じてパターンを調整
    - S3バケット
        - グローバルに一意である必要があり、特別な命名が必要な場合がある
    - CloudFormation スタック
        - 名前の長さに制限があるため、略語の使用を検討
- **既存リソースへの適用**
    
    - 命名規則を既存リソースに適用する場合は段階的アプローチを推奨
    - 全リソースのインベントリを作成
    - 影響分析を実施（依存関係の確認）
    - 優先順位を設定（重要度・影響度に基づく）
    - 移行計画を策定（タイムライン・ロールバック策）
- **自動化ツールの活用**
    
    - 命名規則の検証や適用を自動化
    - CI/CDパイプラインに命名規則チェックを組み込む
    
    [Infrastructure as Code テンプレートに命名規則を組み込む](https://www.notion.so/Infrastructure-as-Code-4cf8383e84494e2d993bb08c2089880e?pvs=21)
    

## グローバルに一意なリソースへの対応

S3バケットなど、グローバルに一意な名前が必要なリソースには以下の拡張パターンを適用：

```
{組織識別子}-{env}-{system-name}-{system-type}-{usage}
```

例: `mycompany-prod-azuchi-stor-logs`

この方式により、AWS全体での名前衝突を回避します。

# References

- [https://future-architect.github.io/coding-standards/documents/forAWSResource/AWSインフラリソース命名規約.html](https://future-architect.github.io/coding-standards/documents/forAWSResource/AWS%E3%82%A4%E3%83%B3%E3%83%95%E3%83%A9%E3%83%AA%E3%82%BD%E3%83%BC%E3%82%B9%E5%91%BD%E5%90%8D%E8%A6%8F%E7%B4%84.html)