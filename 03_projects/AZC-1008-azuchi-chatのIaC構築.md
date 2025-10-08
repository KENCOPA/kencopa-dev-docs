---
type: note
tags:
- AZC-1008
- '#ai/ref'
---
> [!info] 必要なセクションを選択して使用してください！

# 🔗 前提条件・依存関係

[[🗒️ ai-agent & chat]]

# ✅ 機能要件

**SAMを使ってdeplyを実施します**。
- 実装の際は下記の`deploy.sh`と`utils.sh`を参考にしてください。
```sh
# deploy.sh
#!/usr/bin/env bash

set -e

ARG_ENV="--env"
ALLOWED_ENVS=("dev" "stg" "prod")
ARG_OPENWEATHER_API_KEY="--openweather-api-key"
ARG_GOOGLE_MAPS_API_KEY="--google-maps-api-key"

usage() {
	cat <<EOF
Usage: $0 [options]
    [Required]
    ${ARG_ENV} 
        environment name (required)
    ${ARG_OPENWEATHER_API_KEY}
        OpenWeather API key (required)
    ${ARG_GOOGLE_MAPS_API_KEY}
        Google Maps API key (required)

To see help text, you can run:
  -h, --help        
EOF
}

PJT_DIR=$(
	cd $(dirname $0)/../
	pwd
)
. ${PJT_DIR}/scripts/utils.sh

while [[ "$#" -gt 0 ]]; do
	case "$1" in
	${ARG_ENV})
		ENV="$2"
		if ! [[ "${ALLOWED_ENVS[@]}" =~ "${ENV}" ]]; then
			log_err "Invalid environment: ${ENV}"
			usage
			exit 1
		fi
		shift
		;;
	${ARG_OPENWEATHER_API_KEY})
		OPENWEATHER_API_KEY="$2"
		shift
		;;
	${ARG_GOOGLE_MAPS_API_KEY})
		GOOGLE_MAPS_API_KEY="$2"
		shift
		;;
	-h | --help)
		usage
		exit 0
		;;
	*)
		echo "Unknown option: $1"
		exit 1
		;;
	esac
	shift
done
arg_check_and_log "${ARG_ENV}" "${ENV}" || {
	usage
	exit 1
}
arg_check_and_log "${ARG_OPENWEATHER_API_KEY}" "${OPENWEATHER_API_KEY}" || {
	usage
	exit 1
}
arg_check_and_log "${ARG_GOOGLE_MAPS_API_KEY}" "${GOOGLE_MAPS_API_KEY}" || {
	usage
	exit 1
}

log_title "Deploy start"

readonly SYSTEM_NAME="azuchi"
readonly SERVICE_DOMAIN="weather"
readonly SYSTEM_TYPE="api"

readonly prefix=${ENV}-${SYSTEM_NAME}-${SERVICE_DOMAIN}-${SYSTEM_TYPE}
readonly ssm_prefix=/${ENV}/${SYSTEM_NAME}/${SERVICE_DOMAIN}/${SYSTEM_TYPE}

log_info "# -------------------------------- #"
log_info "ENV        : ${ENV}"
log_info "prefix     : ${prefix}"
log_info "ssm_prefix : ${ssm_prefix}"
log_info "# -------------------------------- #"

log_info "Function Deploy"
exec_cmd sam deploy \
	--template-file ${PJT_DIR}/templates/function.cfn.yaml \
	--stack-name ${prefix}-main \
	--capabilities CAPABILITY_NAMED_IAM \
	--no-fail-on-empty-changeset \
	--region "ap-northeast-1" \
	--s3-bucket ${ENV}-azuchi-common-infra-sam-deploy \
	--parameter-overrides \
	Env=${ENV} \
	Prefix=${prefix} \
	OpenWeatherApiKey=${OPENWEATHER_API_KEY} \
	GoogleMapsApiKey=${GOOGLE_MAPS_API_KEY}

log_info "Successfully deployed Function"
```

```sh
# utils.sh
#!/usr/bin/env bash

# ANSI color codes
readonly RED='\033[0;31m'
readonly YELLOW='\033[1;33m'
readonly GREEN='\033[0;32m'
readonly BLUE='\033[0;34m'
readonly NC='\033[0m' # No Color

# 引数のチェックとログ出力
arg_check_and_log() {
	local arg_name="$1"
	local arg_value="$2"

	if [ -z "${arg_value}" ]; then
		log_err "Argument ${arg_name} is required"
		return 1
	fi
	return 0
}

# Log title with blue background
log_title() {
	local message="$1"
	echo -e "# ----------------------------------------------- #"
	echo -e "# ${message}"
	echo -e "# ----------------------------------------------- #"
}

# Log info message with green color
log_info() {
	local message="$1"
	echo -e "${GREEN}[INFO] ${message}${NC}"
}

# Log warning message with yellow color
log_warn() {
	local message="$1"
	echo -e "${YELLOW}[WARN] ${message}${NC}"
}

# Log error message with red color
log_err() {
	local message="$1"
	echo -e "${RED}[ERROR] ${message}${NC}" >&2
}

# Log command execution with cyan color
_log_cmd() {
	local message="$1"
	echo -e "${CYAN}[CMD] ${message}${NC}"
}

# Execute command with logging
# Usage: exec_cmd command arg1 arg2 ...
exec_cmd() {
	# 全ての引数を配列として保持
	local args=("$@")

	# コマンドを表示用に結合
	local cmd_display="${args[*]}"

	# Display command being executed
	_log_cmd "${cmd_display}"

	# Execute command with all arguments
	"${args[@]}"
	return $?
}

# Get AWS account ID
get_aws_account_id() {
	set -x
	res=$(aws sts get-caller-identity --query "Account" --output text)
	set +x
	if [ -z "${res}" ]; then
		log_err "Failed to get AWS account ID"
		exit 1
	fi
	echo "${res}"
}
```

**github actionでsamをデプロイします**
- デプロイルールは下記の通りです。
	- dev環境：developブランチへのマージ
	- stg環境：mainブランチへのマージ
	- prod環境：mainブランチでのタグ作成
-実装の際は下記の`deploy-dev.yml`を参照してください。

```yaml
name: Deploy to Dev

on:
  push:
    branches:
      - develop

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    uses: ./.github/workflows/build-and-deploy.yml
    with:
      environment: dev
    secrets:
      AWS_ACCOUNT_ID: ${{ secrets.ORG_AZUCHI_AWS_ACCOUNT_ID_DEV }}
```

**huskeyのpre-commitでlintチェックを実施します**
- 実装の際は下記の`pre-commit`と`lint-staged.config.js`を参考にしてください。

```
npx lint-staged --allow-empty
```

```js
module.exports = {
	'src/**/*.{ts}': files => [
		`npx @biomejs/biome format --verbose --write ${files.join(' ')}`,
		`npx @biomejs/biome lint --verbose --write ${files.join(' ')}`,
	],
	'scripts/**/*.sh': files => {
		return files.map(f => `shfmt -w ${f}`)
	},
}
```
## ディレクトリ構成
```
.
├── .github
│   └── workflows/
├── .husky/
│   └── pre-commit
├── scripts/
│   ├── deploy.sh
│   ├── init-localstack.sh
│   └── utils.sh
├── src/
│   └── {lambda_function_name}/
├── templates/
│   └── function.cfn.yaml
├── .env.example
├── .gitignore
├── README.md
├── biome.jsonc
├── docker-compose.yml
├── lint-staged.config.js
├── package-lock.json
├── package.json
└── tsconfig.json
```

# ⚙️ 実装手順

まず関連するチケット情報を確認します。

Ran tool
ノート情報に基づいて、azuchi-chatのIaC構築の実装手順を細かくまとめます。

# azuchi-chatのIaC構築：実装手順

## 1. 初期セットアップ

- [x] GitHubリポジトリ「azuchi-chat」を作成する
- [x] ローカル環境にリポジトリをクローンする
- [ ] 基本ディレクトリ構造を作成する
  - `.github/workflows/`
  - `.husky/`
  - `scripts/`
  - `src/`
  - `templates/`
  - `tests/`
- [ ] .gitignoreファイルを作成する
- [ ] README.mdを作成する

## 2. 依存関係とツールのセットアップ

- [ ] package.jsonを作成する
- [ ] TypeScriptの依存関係をインストールする
- [ ] AWS SDKの依存関係をインストールする
- [ ] lintツール（Biome）をインストールする
- [ ] tsconfig.jsonを作成する
- [ ] Huskyをセットアップする
- [ ] lint-staged.config.jsを作成する
- [ ] biome.jsoncを作成する

## 3. SAM関連のファイル作成

- [ ] SAMテンプレート（templates/function.cfn.yaml）を作成する
- [ ] WebSocket API定義を実装する
- [ ] DynamoDBテーブル（sessions, connections, messages）を定義する
- [ ] Lambda関数（connect, disconnect, default, message）を定義する
- [ ] IAMロールとポリシーを定義する
- [ ] CloudWatch Logsリソースを定義する

## 4. スクリプト作成

- [ ] utils.shを作成する
- [ ] deploy.shを作成する
- [ ] start-local.shを作成する

## 5. WebSocketハンドラーの実装

- [ ] src/connect/index.tsを作成する
- [ ] src/disconnect/index.tsを作成する
- [ ] src/default/index.tsを作成する
- [ ] src/message/index.tsを作成する

## 6. ユーティリティとヘルパー関数

- [ ] src/utils/dynamoDB.tsを作成する
- [ ] src/utils/response.tsを作成する
- [ ] src/utils/validate.tsを作成する

## 7. 環境設定と設定ファイル

- [ ] .env.exampleを作成する
- [ ] docker-compose.ymlを作成する
- [ ] samconfig.tomlを作成する

## 8. CI/CD設定

- [ ] .github/workflows/build-and-deploy.ymlを作成する
- [ ] .github/workflows/deploy-dev.ymlを作成する
- [ ] .github/workflows/deploy-stg.ymlを作成する
- [ ] .github/workflows/deploy-prod.ymlを作成する

## 9. テスト設定と実装

- [ ] テスト用の設定ファイルを作成する
- [ ] 単体テストを作成する
- [ ] 統合テストを作成する
- [ ] イベントテスト用のJSONファイルを作成する

## 10. ローカル開発環境構築

- [ ] DynamoDB Localのセットアップ手順を実装する
- [ ] SAM localのセットアップ手順を実装する
- [ ] ローカルテスト用のスクリプトを作成する

## 11. APIドキュメント作成

- [ ] docs/api.mdを作成する
- [ ] OpenAPI定義ファイルを作成する

## 12. 運用関連ドキュメント作成

- [ ] docs/setup.mdを作成する
- [ ] docs/troubleshooting.mdを作成する
- [ ] docs/operations.mdを作成する

## 13. セキュリティとモニタリング設定

- [ ] IAMポリシーの最小権限原則を適用する
- [ ] CloudWatch Dashboardテンプレートを作成する
- [ ] CloudWatch Alarmsを設定する

## 14. テストとデプロイ

- [ ] ローカル環境での単体テストを実行する
- [ ] SAM localでのテストを実行する
- [ ] dev環境へのデプロイを実行する
- [ ] デプロイ検証を実行する

## 15. リソース管理と最適化

- [ ] AWS費用の見積もりと最適化
- [ ] スケーリング設定の確認
- [ ] セキュリティ設定の確認

## 16. 最終確認と文書化

- [ ] すべてのコードをレビューする
- [ ] READMEを完成させる
- [ ] デプロイ手順を最終確認する
- [ ] 運用ドキュメントを完成させる

## 17. azuchi-ai-agentとの連携テスト

- [ ] azuchi-ai-agentの連携APIを呼び出すテストを実装する
- [ ] エンドツーエンドテストシナリオを実装する
- [ ] 統合テスト結果を文書化する

## 18. 本番リリース準備

- [ ] stg環境でのテストを実施する
- [ ] 本番環境のデプロイパラメータを確認する
- [ ] 監視とアラートの最終確認を行う



