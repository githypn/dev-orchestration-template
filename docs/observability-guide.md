# オブザーバビリティ導入ガイド

> OpenTelemetry GenAI Semantic Conventions に準拠した計装テンプレートの導入・使用方法

## 目的

このテンプレートは、マルチエージェントワークフローにおける以下の操作を
OpenTelemetry でトレースするための標準化された計装モジュールを提供する。

- **エージェント操作**: タスク実行、計画立案、委譲等
- **ツール実行**: 外部ツール呼び出し、コマンド実行等
- **LLM 呼び出し**: モデルへのプロンプト送信・レスポンス受信

すべてのデコレータは OTel SDK がインストールされていない場合にパススルー（no-op）
として動作し、既存コードに副作用を与えない。

---

## 前提条件

### 必要な依存ライブラリ

| パッケージ | 最小バージョン | 用途 |
|---|---|---|
| `opentelemetry-sdk` | 1.20.0 | コア SDK（TracerProvider 等） |
| `opentelemetry-exporter-otlp` | 1.20.0 | OTLP エクスポータ（本番環境向け） |

### Python バージョン

- Python 3.11 以上

---

## セットアップ手順

### 1. 依存ライブラリの追加

`pyproject.toml`（`pyproject.toml.template` から生成）に以下が含まれていることを確認する:

```toml
[project.optional-dependencies]
observability = [
    "opentelemetry-sdk>=1.20.0",
    "opentelemetry-exporter-otlp>=1.20.0",
]
```

インストール:

```bash
# uv の場合
uv sync --extra observability

# pip の場合
pip install -e ".[observability]"
```

### 2. プレースホルダーの置換

`src/observability/tracing.py` 内の以下のプレースホルダーをプロジェクトに合わせて置換する
（`bootstrap.sh` 実行時に自動置換される）:

| プレースホルダー | 説明 | 例 |
|---|---|---|
| `{{PROJECT_NAME}}` | プロジェクト名（`project-config.yml` の `project.name`） | `my-project` |
| `{{TRACER_NAME}}` | トレーサー識別子。通常は `パッケージ名.observability` | `my_package.observability` |

### 3. TracerProvider の初期化

アプリケーションのエントリーポイントで `init_tracer()` を呼び出す:

```python
from src.observability.tracing import init_tracer

# 基本的な初期化
init_tracer()

# コンソール出力を有効にする場合（開発・デバッグ用）
init_tracer(enable_console_export=True)

# カスタムサービス名を指定する場合
init_tracer(service_name="my-service", enable_console_export=True)
```

### 4. デコレータの適用

対象の関数にデコレータを付与する（詳細は次章を参照）。

---

## デコレータの使用方法と記録属性

### `@trace_agent_operation`

エージェントの操作（タスク計画、実行委譲等）をトレースする。

```python
from src.observability.tracing import trace_agent_operation

@trace_agent_operation("orchestrator.plan_task")
def plan_task(task: str) -> str:
    """タスクを分解し計画を立案する。"""
    ...
```

**記録属性**:

| 属性名 | 型 | 説明 |
|---|---|---|
| `gen_ai.agent.operation` | string | 操作名（引数 or `__qualname__`） |
| `gen_ai.system` | string | システム名（`SERVICE_NAME`） |
| `agent.status` | string | `"success"` / `"error"` |

### `@trace_tool_execution`

ツールの実行（シェルコマンド、ファイル操作等）をトレースする。

```python
from src.observability.tracing import trace_tool_execution

@trace_tool_execution("shell.run_command")
def run_shell_command(cmd: str) -> str:
    """シェルコマンドを実行する。"""
    ...
```

**記録属性**:

| 属性名 | 型 | 説明 |
|---|---|---|
| `tool.name` | string | ツール名（引数 or `__qualname__`） |
| `gen_ai.system` | string | システム名（`SERVICE_NAME`） |
| `tool.status` | string | `"success"` / `"error"` |

### `@trace_llm_call`

LLM 呼び出し（モデルへのプロンプト送信）をトレースする。

```python
from src.observability.tracing import trace_llm_call

@trace_llm_call("claude-3-opus")
def call_claude(prompt: str) -> str:
    """Claude にプロンプトを送信する。"""
    ...
```

**記録属性**:

| 属性名 | 型 | 説明 |
|---|---|---|
| `gen_ai.operation.name` | string | 操作種別（`"chat"`） |
| `gen_ai.request.model` | string | リクエスト時のモデル名 |
| `gen_ai.system` | string | システム名（`SERVICE_NAME`） |
| `gen_ai.response.finish_reason` | string | `"stop"` / `"error"` |

> **トークン数の記録**: `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` は
> LLM API のレスポンスから取得する必要があるため、デコレータ内では自動設定されない。
> 必要に応じてスパンに手動で属性を追加すること。

---

## OpenTelemetry GenAI Semantic Conventions のステータスに関する注記

本ドキュメント記載時点（2026年2月）で、OpenTelemetry GenAI Semantic Conventions の
ステータスは **"Development"** である。"Stable" には未達のため、属性名が将来変更される
可能性がある。

最新仕様は以下を定期的に確認し、属性名の変更に追従すること:
- https://opentelemetry.io/docs/specs/semconv/gen-ai/

---

## ローカル開発環境での確認方法

### stdout exporter を用いたトレース確認

```bash
# 1. OTel SDK をインストール
uv sync --extra observability

# 2. コンソール出力モードで初期化し、トレースを確認
python -c "
from src.observability.tracing import init_tracer, get_tracer

# コンソール出力を有効化
init_tracer(enable_console_export=True)

# テストスパンを生成
tracer = get_tracer()
if tracer is not None:
    with tracer.start_as_current_span('test-span') as span:
        span.set_attribute('test.key', 'test-value')
    print('トレース出力を確認してください（上記の JSON 出力）')
else:
    print('OTel SDK が見つかりません')
"
```

正常に動作すると、コンソールに JSON 形式のスパン情報が出力される。

### CI での確認

`.github/workflows/ci.yml` の「OpenTelemetry 計装確認」ステップのコメントを
解除することで、CI パイプラインでも計装の初期化を検証できる。

---

## ファイル構成

```
src/
└── observability/
    ├── __init__.py       # パッケージ初期化（空ファイル）
    └── tracing.py        # 計装デコレータ（3種）+ TracerProvider 初期化
```
