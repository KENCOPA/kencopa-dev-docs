---
type: note
moc: "[[🏷️基本設計書]]"
tags:
- AZC-1048
- '#ai/ref'
---
# Safegen向けロガー実装リファクタリング詳細計画

## 1. 現状分析

### 1.1 rag-backend（safegen）の現在のロガー実装

現在のrag-backend（safegen）では、標準のPython loggingモジュールを直接使用しています：

```python
# src/main.py
import logging
logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)
```

各モジュールでは以下のようにロガーを取得しています：

```python
# 各モジュール（例：src/app/api.py）
import logging
logger = logging.getLogger(__name__)
```

### 1.2 現状の課題

この実装には以下の課題があります：

- 構造化ロギング（JSON形式）がない
- AWS Lambda環境の最適化（Request IDの追跡など）がされていない
- 環境に応じたフォーマット切り替えができない
- ロギング設定の一貫性が保証されていない
- 例外情報の構造化処理がない

### 1.3 azuchi-file-export-jobのロガー実装の特徴

一方、azuchi-file-export-jobには以下の特徴を持つ優れたロガー実装があります：

- シングルトンパターンを使用したLoggerクラス
- JSON形式でログを出力するJsonFormatterクラス
- AWS Lambda環境向けの最適化（Request IDの追跡）
- 環境変数に基づくログレベル設定
- 環境に応じたフォーマット切り替え（開発環境ではテキスト形式、本番環境ではJSON形式）
- 例外情報の構造化処理
- サードパーティライブラリのログレベル調整

## 2. リファクタリング実装

### 2.1 新しいロガーモジュールの実装 (`src/utils/logger.py`)

```python
from typing import Any, Dict, Optional
import logging
import json
import os
from datetime import datetime

class JsonFormatter(logging.Formatter):
    """JSON形式でログを出力するフォーマッター"""
    def __init__(self, aws_request_id: Optional[str] = None):
        super().__init__()
        self.aws_request_id = aws_request_id

    def format(self, record: logging.LogRecord) -> str:
        log_data: Dict[str, Any] = {
            "timestamp": datetime.fromtimestamp(record.created).isoformat(),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno
        }
        
        # リクエストIDを設定
        if hasattr(record, "aws_request_id"):
            log_data["awsRequestId"] = record.aws_request_id
        elif self.aws_request_id:
            log_data["awsRequestId"] = self.aws_request_id
        
        # トレースIDを設定（safegen特有の要件）
        if hasattr(record, "trace_id"):
            log_data["traceId"] = record.trace_id
        
        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)
        
        if hasattr(record, "extra"):
            log_data.update(record.extra)
            
        return json.dumps(log_data, ensure_ascii=False)

class Logger:
    """シングルトンロガークラス"""
    _instance = None
    _logger = None
    _aws_request_id = None
    _trace_id = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        if self._logger is None:
            self._setup_logger()

    def _setup_logger(self, log_level: Optional[str] = None) -> None:
        """ロガーを設定します。"""
        # ログレベルの設定
        level = log_level or os.environ.get("LOG_LEVEL", "INFO")
        numeric_level = getattr(logging, level.upper(), logging.INFO)

        # ルートロガーのハンドラーをクリア
        root_logger = logging.getLogger()
        for handler in root_logger.handlers[:]:
            root_logger.removeHandler(handler)

        # アプリケーションロガーの設定
        self._logger = logging.getLogger("safegen")
        self._logger.setLevel(numeric_level)
        # 親ロガーへの伝播を防止
        self._logger.propagate = False

        # 既存のハンドラーをクリア
        for handler in self._logger.handlers[:]:
            self._logger.removeHandler(handler)

        # ハンドラーの設定
        handler = logging.StreamHandler()
        handler.setLevel(numeric_level)

        # 環境に応じてフォーマッターを設定
        if os.environ.get("ENV") == "local":
            formatter = logging.Formatter('[%(levelname)s] %(message)s')
        else:
            formatter = JsonFormatter(aws_request_id=self._aws_request_id)

        handler.setFormatter(formatter)
        self._logger.addHandler(handler)

        # 他のライブラリのログレベルをWARNINGに設定
        logging.getLogger("boto3").setLevel(logging.WARNING)
        logging.getLogger("botocore").setLevel(logging.WARNING)
        logging.getLogger("urllib3").setLevel(logging.WARNING)
        logging.getLogger("requests").setLevel(logging.WARNING)
        logging.getLogger("langchain").setLevel(logging.WARNING)
        logging.getLogger("openai").setLevel(logging.WARNING)

    def set_aws_request_id(self, aws_request_id: Optional[str]) -> None:
        """AWSリクエストIDを設定します。"""
        self._aws_request_id = aws_request_id
        self._setup_logger()
        
    def set_trace_id(self, trace_id: Optional[str]) -> None:
        """トレースIDを設定します。"""
        self._trace_id = trace_id
        
    def with_context(self, **kwargs) -> Dict[str, Any]:
        """ログに追加するコンテキスト情報を作成します。"""
        context = {}
        if self._aws_request_id:
            context["aws_request_id"] = self._aws_request_id
        if self._trace_id:
            context["trace_id"] = self._trace_id
        context.update(kwargs)
        return {"extra": context}

    def __getattr__(self, name):
        """ロガーメソッドを委譲します。"""
        if name in ["info", "debug", "warning", "error", "critical"]:
            def wrapper(*args, **kwargs):
                # コンテキスト情報をログレコードに追加
                if "extra" not in kwargs:
                    kwargs["extra"] = {}
                if self._aws_request_id:
                    kwargs["extra"]["aws_request_id"] = self._aws_request_id
                if self._trace_id:
                    kwargs["extra"]["trace_id"] = self._trace_id
                return getattr(self._logger, name)(*args, **kwargs)
            return wrapper
        return getattr(self._logger, name)

# グローバルロガーインスタンス
logger = Logger()
```

### 2.2 Lambda ハンドラーでのロガー初期化 (`src/main.py`)

```python
import asyncio
import inspect
import json
import logging

from app.api import handler as api_handler
from utils.logger import logger

def is_pytest_environment():
    """現在の環境がpytestかどうかを判定"""
    for frame in inspect.stack():
        if frame.filename.find('pytest') > -1:
            return True
    return False

def handler(event, context):
    """
    AWS Lambda ハンドラー関数
    
    環境に応じて適切な方法でasync関数を実行します：
    - pytest環境: async関数をそのまま返す（pytestがawaitする）
    - Lambda環境: 同期的に実行して結果を返す
    """
    # AWS Lambda環境の場合はリクエストIDを設定
    if context and hasattr(context, 'aws_request_id'):
        logger.set_aws_request_id(context.aws_request_id)
    
    async def _handler(event, context):
        return await api_handler(event, context)

    # pytestの場合はコルーチンをそのまま返す（@pytest.mark.asyncioが処理）
    if is_pytest_environment():
        return _handler(event, context)

    # Lambda環境ではコルーチンを同期的に実行
    loop = asyncio.get_event_loop()
    return loop.run_until_complete(_handler(event, context))
```

### 2.3 トレースIDの設定 (`src/app/api.py`)

```python
import uuid
import json
from utils.logger import logger

async def handler(event, context):
    """APIリクエストを処理するハンドラー関数"""
    try:
        # リクエストボディの解析
        try:
            event = json.loads(event["body"])
        except Exception as e:
            logger.error(f"イベントデータの取得に失敗しました: {str(e)}")
            return {
                "statusCode": 400,
                "body": json.dumps({"error": "イベントデータの取得に失敗しました"}, ensure_ascii=False)
            }

        # トレースIDの生成と設定
        trace_id = str(uuid.uuid4())
        logger.set_trace_id(trace_id)
        
        # 以下、既存の処理...
```

### 2.4 既存のロギング呼び出しの修正例

#### 2.4.1 src/app/chain.py

```python
# 修正前
logger.error(f"チェーン実行に失敗しました: {str(e)}")

# 修正後（構造化ログの活用）
logger.error("チェーン実行に失敗しました", **logger.with_context(
    error_type=type(e).__name__,
    error_details=str(e),
    chain_type=chain_type,
    input_variables=input_variables.keys()
))
```

#### 2.4.2 src/app/weather.py

```python
# 修正前
logger.error(f"天気情報の取得に失敗しました: {str(e)}")

# 修正後（構造化ログの活用）
logger.error("天気情報の取得に失敗しました", **logger.with_context(
    error_type=type(e).__name__,
    error_details=str(e),
    address=event["site_address"],
    date=event["work_date"]
))
```

#### 2.4.3 src/prompts/langfuse_prompts.py

```python
# 修正前
logger.info(f"プロンプトを生成しました: {prompt_id}")

# 修正後（構造化ログの活用）
logger.info("プロンプトを生成しました", **logger.with_context(
    prompt_id=prompt_id,
    template_name=template_name,
    variables=list(variables.keys())
))
```

## 3. 移行戦略

### 3.1 段階的な移行手順

1. **ロガーモジュールの実装:**
    - `src/utils/logger.py`を作成
2. **メインハンドラーの修正:**
    - `src/main.py`のLambdaハンドラーを修正
    - AWS Request IDの設定を追加
    - pytestとの互換性確保
3. **APIハンドラーの修正:**
    - `src/app/api.py`のハンドラー関数を修正
    - トレースIDの生成と設定を追加
    - リクエストボディ解析のエラーハンドリングを改善
4. **既存のロギング呼び出しの修正:**
    - 優先度の高いモジュールから順次修正
    - まずはエラーログの修正を優先
    - 構造化ログを活用するよう更新
    - `chain.py`, `weather.py`, `langfuse_prompts.py`などの重要モジュールから対応

## 4. ロガー使用方法ガイドライン

### 4.1 基本的な使用方法

```python
from utils.logger import logger

# 基本的なログ出力
logger.info("処理を開始します")
logger.debug("デバッグ情報")
logger.warning("警告メッセージ")
logger.error("エラーが発生しました")
logger.critical("致命的なエラー")

# 構造化ログの活用
logger.info("ユーザー情報を取得しました", **logger.with_context(
    user_id=user_id,
    request_type="get_user_info"
))

# 例外のログ記録
try:
    # 処理
except Exception as e:
    logger.error("処理に失敗しました", **logger.with_context(
        error_type=type(e).__name__,
        error_details=str(e),
        function_name="process_data"
    ))
```

### 4.2 ベストプラクティス

1. **適切なログレベルの使用**
    
    - DEBUG: 詳細なデバッグ情報
    - INFO: 正常な処理の流れ
    - WARNING: 注意が必要な状況
    - ERROR: エラー（回復可能）
    - CRITICAL: 致命的なエラー（回復不能）
2. **構造化ログの活用**
    
    - 文字列結合よりも`with_context`の使用を推奨
    - フォーマット文字列内にエラーメッセージを埋め込まない
3. **コンテキスト情報の適切な付加**
    
    - リクエストのパラメータ
    - 処理対象のデータID
    - エラー詳細
4. **個人情報・機密情報の取り扱い**
    
    - 個人を特定できる情報はログに記録しない
    - パスワードなどの機密情報をログに出力しない
新たな環境変数は追加しないでください
log-levelは開発環境(local,dev,stg,prod)などで分けるようにしてください
