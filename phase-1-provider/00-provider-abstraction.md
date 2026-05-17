# Phase 1 — Provider 抽象层与 API 边界归一化

> **范围**: Hermes Harness v0.9.0 如何同时对接 OpenAI、Anthropic、Codex、Copilot ACP、OpenRouter、Kimi、MiniMax 等不同 API 语义，仍保持内部统一的 OpenAI-style message format。
>
> **源文件**: `agent/anthropic_adapter.py` (1410 行), `run_agent.py` (10805 行), `agent/auxiliary_client.py`, `agent/copilot_acp_client.py`, `agent/error_classifier.py`, `agent/usage_pricing.py`, `agent/model_metadata.py`, `hermes_cli/codex_models.py`, `hermes_cli/models.py`, `hermes_cli/providers.py`, `cli.py`。
>
> **核心概念**: Harness 内部只讲一种格式（OpenAI 风格 messages / function calling），每个 provider 的 adapter 负责"向上兼容"式翻译。Provider 切换（fallback）和 reasoning 内容统一是第一类设计关切。

---

## 1. Anthropic Adapter (`agent/anthropic_adapter.py`)

Anthropic adapter 是一个**纯格式转换 + 客户端配置层**（1410 行），**不**包含 SSE 流式解析逻辑——那部分在上层的 `run_agent.py` 中处理。

### 1.1 Role 映射：OpenAI → Anthropic Messages API

**核心函数**: `convert_messages_to_anthropic()` (第 917–1184 行)

- **System 消息被提取到顶层参数**。Anthropic 的 Messages API 把 `system` 放在请求顶层而非 message list 中。函数返回 `(system_prompt, anthropic_messages)` (第 921–923 行)。
  - 如果 system content 是带 `cache_control` 标记的 list，则保留为 block 列表 (第 940–946 行)；否则压平为普通字符串 (第 947–952 行)。
- **`assistant` role 保留**，但其 content 被重建为 Anthropic block 列表：
  - 先前置保留的 `thinking` / `redacted_thinking` blocks (第 956 行，通过 `_extract_preserved_thinking_blocks`)。
  - 再追加 text (第 957–963 行)。
  - OpenAI 的 `tool_calls` 转为 `tool_use` blocks (第 964–978 行)。
- **`tool` role → `user` + `tool_result` blocks**。Anthropic 没有 `tool` role；tool result 以 content block 形式放在 `user` 消息中 (第 986–1008 行)。
  - 连续的 `tool_result` blocks 被合并为单条 `user` 消息，避免触发 Anthropic 严格的 role alternation 违规 (第 998–1008 行)。
- **`user` role 保留**。Content 从 OpenAI 多模态 parts 转为 Anthropic blocks (第 1011–1026 行)。空 user message 会被填充 `"(empty message)"`，因为 Anthropic 拒绝空 content。
- **强制 role 交替**。连续的同 role message 被合并：
  - 连续 `user`: 字符串拼接或列表扩展 (第 1066–1083 行)。
  - 连续 `assistant`: 合并，但**第二条 message 的 thinking blocks 被丢弃**，因为它们的 signature 绑定在不同 turn 边界上 (第 1084–1106 行)。

### 1.2 Tool 格式转换

**涉及函数**:
- `convert_tools_to_anthropic()` (第 779–791 行)
- `build_anthropic_kwargs()` 中的 tool_choice 映射 (第 1301–1313 行)

OpenAI tool 定义:
```python
{"type": "function", "function": {"name": "...", "description": "...", "parameters": {...}}}
```
Anthropic 期望:
```python
{"name": "...", "description": "...", "input_schema": {...}}
```

Tool-call ID 通过 `_sanitize_tool_id()` 净化为 `[a-zA-Z0-9_-]` (第 766–776 行)。

**tool_choice 映射表**:
| OpenAI `tool_choice` | Anthropic 参数 |
|---|---|
| `auto` / `None` | `{"type": "auto"}` |
| `required` | `{"type": "any"}` |
| `none` | **完全省略 `tools` 字段** (Anthropic 没有 "none") |
| 指定名称字符串 | `{"type": "tool", "name": <name>}` |

### 1.3 Thinking Budget 与 Adaptive Effort 映射

**常量定义** (第 30–37 行):
```python
THINKING_BUDGET = {"xhigh": 32000, "high": 16000, "medium": 8000, "low": 4000}
ADAPTIVE_EFFORT_MAP = {
    "xhigh": "max", "high": "high", "medium": "medium",
    "low": "low", "minimal": "low",
}
```

Hermes 暴露单一的 `reasoning_config.effort` 枚举 (`xhigh` 到 `minimal`)。Adapter 将其映射到 Anthropic 两套不同的 extended-thinking API：

1. **Legacy/manual thinking** (Claude 3.7 Sonnet, Claude 4 等): `thinking.type = "enabled"` + `budget_tokens`。
2. **Adaptive thinking** (Claude 4.6): `thinking.type = "adaptive"` + `output_config.effort`。

**`build_anthropic_kwargs()` 中的应用** (第 1320–1333 行):
```python
if reasoning_config.get("enabled") is not False and "haiku" not in model.lower():
    effort = str(reasoning_config.get("effort", "medium")).lower()
    budget = THINKING_BUDGET.get(effort, 8000)
    if _supports_adaptive_thinking(model):
        kwargs["thinking"] = {"type": "adaptive"}
        kwargs["output_config"] = {"effort": ADAPTIVE_EFFORT_MAP.get(effort, "medium")}
    else:
        kwargs["thinking"] = {"type": "enabled", "budget_tokens": budget}
        kwargs["temperature"] = 1
        kwargs["max_tokens"] = max(effective_max_tokens, budget + 4096)
```

关键细节:
- Haiku 模型被显式排除在 extended thinking 之外 (第 1321 行)。
- Legacy 模式下 `temperature` 强制为 `1`，且 `max_tokens` 至少为 `budget + 4096`，防止 thinking budget 挤占可见输出。

**Adaptive 检测函数** `_supports_adaptive_thinking()` (第 93–95 行):
```python
def _supports_adaptive_thinking(model: str) -> bool:
    return any(v in model for v in ("4-6", "4.6"))
```
同时匹配连字符和点号形式（覆盖 Anthropic 官方命名和 OpenRouter 点号命名）。故意收窄——未知未来模型回落到 legacy `thinking.enabled` 路径，更安全。

### 1.4 Max Output Token 上限查表

**表**: `_ANTHROPIC_OUTPUT_LIMITS` (第 43–65 行)  
**查表函数**: `_get_anthropic_max_output()` (第 72–90 行)

对模型名做 normalize（点号转连字符）后做**最长前缀子串匹配**。Fallback 为 128,000 tokens。此表至关重要，因为 Anthropic 强制要求 `max_tokens` 字段；之前 hardcode 16,384 会严重 starvation thinking-enabled 模型（thinking tokens 占用了输出额度）。

### 1.5 认证：三条路径

| 路径 | 检测方式 | Header / 机制 |
|---|---|---|
| 普通 API Key | 前缀 `sk-ant-api*` | `x-api-key` header（SDK 默认） |
| OAuth / Setup Token | 前缀 `sk-ant-oat*` 或 `eyJ` (JWT) | `Authorization: Bearer` + Claude Code 身份 headers |
| Claude Code 凭证文件 | `~/.claude/.credentials.json` | Bearer auth + **自动刷新** |

**Token 解析优先级** (`resolve_anthropic_token()`, 第 525–566 行):
1. `ANTHROPIC_TOKEN` 环境变量
2. `CLAUDE_CODE_OAUTH_TOKEN` 环境变量
3. Claude Code 凭证文件（过期时自动刷新）
4. `ANTHROPIC_API_KEY` 环境变量

**OAuth 自动刷新** (`_refresh_oauth_token()`, 第 424–442 行):
调用 Anthropic OAuth token endpoint，将刷新后的凭证写回 `~/.claude/.credentials.json`。`~/.claude.json` 被**显式排除** (第 331–342 行)，因为 Opencode 的订阅流基于 OAuth，应走可刷新路径而非静态 key。

**Claude Code 身份伪装** (第 287–291 行):
OAuth 请求注入 `user-agent: claude-cli/{version}` 和 `x-app: cli`。原因：Anthropic 基础设施按 user-agent 和 header 路由请求；缺少这些指纹会导致间歇性 500。版本号动态检测 (`claude --version`)，fallback 为 `2.1.74`，因为 Anthropic 会拒绝过旧版本 (第 124–134 行)。

### 1.6 Beta Headers

**通用 betas** (所有 auth 类型都会发送，除非被过滤):
- `interleaved-thinking-2025-05-14` — 允许 `thinking` / `redacted_thinking` blocks 与 `text` / `tool_use` 交错出现在响应流中。
- `fine-grained-tool-streaming-2025-05-14` — 支持 per-delta tool 流式（逐参数 delta 而非整块 tool block）。

**OAuth-only betas**: `claude-code-20250219`, `oauth-2025-04-20` — 仅用于 OAuth/setup-token auth，确保 Claude Code 路由兼容性。

**Fast-mode beta**: `fast-mode-2026-02-01` — 当 `fast_mode=True` 且指向 Anthropic 官方 endpoint 时附加。在 Opus 4.6 上可实现约 2.5× 输出吞吐。

**Provider 级过滤**: MiniMax 的 Anthropic-compatible endpoints 遇到 `fine-grained-tool-streaming` beta 会报错。Adapter 对 `api.minimax.io/anthropic` 或 `api.minimaxi.com/anthropic` 主动剥离该 beta (第 103–106 行、第 230–240 行)。

---

## 2. Codex Adapter 设计

**注意**: 不存在单独的 `codex_responses.py` 文件。Codex 支持分散在 `run_agent.py`（核心逻辑）、`hermes_cli/codex_models.py`（模型注册表）、`agent/models_dev.py`（选择辅助）。

### 2.1 模型定义

| 文件 | 职责 |
|---|---|
| `hermes_cli/codex_models.py` (176 行) | `CODEX_MODELS` dict，family→config。Families: `codex-mini`, `codex`, `codex-reasoning` |
| `hermes_cli/models.py` (1909 行) | 通用模型注册表 `MODELS`，`get_model_config()`, `get_model_family()` |
| `agent/models_dev.py` (681 行) | 模型选择辅助：`smol_model()`, `large_model()`, `vision_model()` |

### 2.2 api_mode 解析

**`run_agent.py` 第 670–710 行**: `api_mode` 在 agent 初始化时根据 `provider`、`base_url` 和显式 `api_mode` 参数解析。

伪代码逻辑:
```
if api_mode in {chat_completions, codex_responses, anthropic_messages}:
    直接使用
elif provider == "openai-codex":
    api_mode = codex_responses
elif provider is None 且 base_url 含 "chatgpt.com/backend-api/codex":
    api_mode = codex_responses; provider = "openai-codex"
elif provider == "anthropic" 或 base_url 含 "api.anthropic.com":
    api_mode = anthropic_messages
elif base_url 以 "/anthropic" 结尾:
    api_mode = anthropic_messages  # 第三方 Anthropic-compatible
else:
    api_mode = chat_completions
```

初始解析后，**GPT-5.x 模型**会被 `_model_requires_responses_api()` (第 1876 行) 强制改回 `codex_responses`——这些模型被 OpenAI 和 OpenRouter 拒绝在 `/v1/chat/completions` 上使用。

### 2.3 请求预检与校验

**`_preflight_codex_api_kwargs(api_kwargs, allow_stream=False)`** (~第 3710 行):
- 校验 `model`、`instructions`、`input` 类型。
- 强制 `store=False`。
- **`allow_stream=False` 时硬阻断 `stream=True`**，抛出 `ValueError`。这是主循环中禁用 Codex 流式的闸门。
- Tool 标准化为 `{"type": "function", "name": ..., "description": ..., "strict": ..., "parameters": ...}`。

### 2.4 流式架构

主循环无条件设置 `_use_streaming = True`（用于 health-check 信号），但预检会**剥离** `stream=True`。因此：

- **标准 Codex 路径是非流式的**。`_interruptible_api_call()` 直接调用 `client.responses.create(**api_kwargs)`，然后通过 `_normalize_codex_response()` 规范化。
- **流式路径存在但被绕过**。`_run_codex_stream()` (~第 4250 行) 负责处理 `ResponsesStream` 事件，但仅在 `stream=True` 逃过预检时才可达（内部 fallback 路径或未来代码变更）。

### 2.5 Response 规范化

**`_normalize_codex_response(response)`** (~第 3870 行):
遍历 `response.output` items:
- `type == "message"` → 提取文本；追踪 `phase` (`commentary` vs `final_answer`)。
- `type == "reasoning"` → 提取可见 reasoning 文本 + 捕获完整 `encrypted_content` blob、`id`、`summary`，用于多 turn replay。
- `type == "function_call"` / `"custom_tool_call"` → 构建 `SimpleNamespace` tool-call 对象，含 `response_item_id`（必须以 `fc_` 开头）。

**Finish reason 判定**:
- 存在 tool_calls → `"tool_calls"`
- Incomplete/queued items，或 `saw_commentary_phase and not saw_final_answer_phase` → `"incomplete"`
- 仅有 reasoning 无 text → `"incomplete"`（防止空 content retry burn）
- 否则 → `"stop"`

### 2.6 Chat History → Codex Input 转换

**`_chat_messages_to_responses_input(messages)`** (~第 3515 行):
- 丢弃 `role="system"`。
- `role="assistant"`: 先 replay `msg["codex_reasoning_items"]`（加密 blobs），再输出 assistant text，再将 `tool_calls` 映射为 `"type": "function_call"` items。
- `role="tool"` → `"type": "function_call_output"`（含 `call_id` + `output`）。
- `role="user"` → 标准 `{role, content}`。

**`_preflight_codex_input_items(raw_items)`** (~第 3619 行): 校验每个 item；按 `id` 去重 reasoning items。

### 2.7 Reasoning 连续性

Codex 的 reasoning 对 Harness 是**加密且不可解释的**。Harness 只负责原样保存并在后续 turn 中回放。

- `_build_assistant_message()` (~第 6404 行) 将 `codex_reasoning_items` 保留在返回的 message dict 上。
- 下一 turn，`_chat_messages_to_responses_input()` 回放这些 blobs，维持模型的 thinking state。

### 2.8 Ack-Continuation 启发式

**`_looks_like_codex_intermediate_ack()`** (~第 1938 行):
- 检测短消息（≤1200 字符）包含未来承诺短语（`i'll`, `let me`, `i will`）+ 动作标记（`look into`, `inspect`, `fix`, `run`, `search`）+ 工作区标记。
- 若命中，主循环 (~第 10274 行) 追加一条 interim assistant message，并注入 system-style user nudge: `"[System: Continue now. Execute the required tool calls...]"`。
- 每 turn 最多允许 **2** 次此类 continuation (`codex_ack_continuations < 2`)。

---

## 3. Provider Fallback Chain（降级链）

### 3.1 配置加载

**`cli.py` 第 1755–1763 行**: 从 `CLI_CONFIG` 读取 `fallback_providers`（新列表格式）或 legacy `fallback_model`（单 dict）。Legacy dict 会被标准化为单元素列表。

```python
fb = CLI_CONFIG.get("fallback_providers") or CLI_CONFIG.get("fallback_model") or []
if isinstance(fb, dict):
    fb = [fb] if fb.get("provider") and fb.get("model") else []
self._fallback_model = fb
```

### 3.2 初始化 (`run_agent.py` ~985–1010)

```python
if isinstance(fallback_model, list):
    self._fallback_chain = [f for f in fallback_model
                            if isinstance(f, dict) and f.get("provider") and f.get("model")]
elif isinstance(fallback_model, dict) and fallback_model.get("provider") and fallback_model.get("model"):
    self._fallback_chain = [fallback_model]
else:
    self._fallback_chain = []
self._fallback_index = 0
self._fallback_activated = False
```

启动时 UI 打印降级链（单条或多条 provider 用 `→` 分隔）。

### 3.3 降级激活 (`_try_activate_fallback`, ~第 5560 行)

核心行为:
1. 检查 `_fallback_index` 是否已走到 `_fallback_chain` 末尾。
2. 取下一条 entry；index 自增。
3. 通过 `agent.auxiliary_client.resolve_provider_client()` 解析 provider client，传 `raw_codex=True`。支持自定义 `base_url` 和 `api_key` hint。Ollama Cloud 特殊处理：若无显式 key 则从环境变量 `OLLAMA_API_KEY` 拉取。
4. 通过 `normalize_model_for_provider()` 标准化模型名。
5. **重新推导 `api_mode`**。判断规则:
   - `provider == "openai-codex"` → `codex_responses`
   - `provider == "anthropic"` 或 base URL 以 `/anthropic` 结尾 → `anthropic_messages`
   - `_is_direct_openai_url()` → `codex_responses`
   - `_model_requires_responses_api()` (GPT-5.x) → `codex_responses`
   - 其余 → `chat_completions`
6. 若 `api_mode == "anthropic_messages"`，通过 `agent.anthropic_adapter.build_anthropic_client()` 构建原生 Anthropic client；否则原地替换 OpenAI client。
7. 设置 `_fallback_activated = True`。

### 3.4 主 Provider 恢复 (`_restore_primary_runtime`, ~第 5700 行)

**Turn-scoped 设计**——每 turn 结束后自动恢复主 provider，让下一 turn 重新从主 provider 开始尝试。

从 `_primary_runtime` 快照（初始化时捕获）恢复:
- 核心 runtime: `model`, `provider`, `base_url`, `api_mode`, `api_key`, `client_kwargs`, `_use_prompt_caching`
- 重建 client（Anthropic 或 OpenAI）
- 恢复 context compressor 的模型设置
- 重置 `_fallback_activated = False`, `_fallback_index = 0`

这防止**瞬时故障把会话永久钉死在降级 provider 上**。Gateway 会跨消息缓存 agent，因此该恢复机制对 Gateway 同样必要。

### 3.5 触发条件

**Rate-limit / billing 耗尽** (`run_agent.py` ~9155–9175):
```python
is_rate_limited = classified.reason in (FailoverReason.rate_limit, FailoverReason.billing)
if is_rate_limited and self._fallback_index < len(self._fallback_chain):
    pool = self._credential_pool
    pool_may_recover = pool is not None and pool.has_available()
    if not pool_may_recover:
        self._emit_status("⚠️ Rate limited — switching to fallback provider...")
        if self._try_activate_fallback():
            retry_count = 0; compression_attempts = 0; primary_recovery_attempted = False
            continue
```

**关键顺序**: 凭证池轮换 **优先于** provider 降级。若池子还有未用凭证 (`pool.has_available()`)，先换 key；池子耗尽后才切 provider。

**空 response 耗尽** (`run_agent.py` ~10200–10270):
- Retry 烧尽后，若 response 真正为空（无 content、无 tool calls），尝试降级。
- 降级成功则 loop 以 `retry_count = 0` 重启。
- 降级链耗尽或未配置，则 turn 以 `"(empty)"` content 终止。

### 3.6 Client 创建边界情况 — Copilot ACP

**`_create_openai_client()`** (~第 4084 行) 检测 `provider == "copilot-acp"` 或 `base_url.startswith("acp://copilot")`，此时实例化 `CopilotACPClient` 而非标准 `OpenAI`。这意味着与 Copilot ACP 的降级切换会重建一个完全不同的 client class。

---

## 4. Reasoning / Thinking 内容统一化

### 4.1 从 Provider Response 中提取

**主循环 — `AIAgent._extract_reasoning()`** (`run_agent.py` ~2010–2075):

结构化 reasoning 字段的解析顺序:
1. `assistant_message.reasoning` — DeepSeek、Qwen 等。
2. `assistant_message.reasoning_content` — Moonshot AI、Novita 等。
3. `assistant_message.reasoning_details` — OpenRouter 统一数组格式。提取 `summary`、`thinking`、`content` 或 `text` 字段。

**Inline tag fallback**: 若无结构化字段，则用正则从 `assistant_message.content` 中提取以下标签包裹的内容:
- ``
- `<thinking>`
- `<reasoning>`
- `<thought>`
- `<REASONING_SCRATCHPAD>`

结果合并为 `"\n\n".join(reasoning_parts)`，存入 turn message dict 的 `msg["reasoning"]` (~第 6444 行)。

**Auxiliary client 镜像 — `extract_content_or_reasoning()`** (`auxiliary_client.py` ~2390–2441):
为主循环的行为做镜像，用于 vision/summarization 任务。**关键差异**: 先从 `content` 中 strip inline reasoning blocks，返回 cleaned text；仅在 `content` 为空或全是 reasoning tags 时才 fallback 到结构化字段。

### 4.2 载体数据结构

- **Turn message dict**: `{"role": "assistant", "content": ..., "reasoning": reasoning_text, "finish_reason": ...}`
- **`reasoning_details` 原样保留**，使 OpenRouter、Anthropic、OpenAI 等 provider 能在跨 turns 时维持 reasoning 连续性（包括 `signature`、`encrypted_content` 等不透明字段）。
- **Codex 专用**: `assistant_message.codex_reasoning_items` 单独保留，用于多 turn replay。
- **Copilot ACP 规范化**: `copilot_acp_client.py` 合成 `SimpleNamespace(content=cleaned_text, reasoning=reasoning_text, reasoning_content=reasoning_text, reasoning_details=None)`，让 harness 其余部分把 Copilot ACP 当成 OpenAI/DeepSeek 一样处理。

### 4.3 渲染 / 展示

**发现**: `agent/display.py` **不包含** reasoning content 的渲染逻辑。唯一 mention 是 KawaiiSpinner 的形容词列表里有个 "reasoning"（spinner 动词，不是展示内容）。

**真正的渲染在 `cli.py`**（`agent/` 之外）:
- `_stream_reasoning_delta()` — 将 reasoning tokens 流入 dim box。
- `_emit_reasoning_preview()` / `_flush_reasoning_preview()` — 合并微小 chunk，打印 `[thinking] ...`。
- `_on_thinking()` — 更新 TUI spinner 文本。

`HermesCLI` 注册的 `reasoning_callback` 驱动实际视觉输出。

### 4.4 Usage 与计价

**`CanonicalUsage` dataclass** (`usage_pricing.py` 第 27–35 行):
```python
@dataclass(frozen=True)
class CanonicalUsage:
    input_tokens: int = 0
    output_tokens: int = 0
    cache_read_tokens: int = 0
    cache_write_tokens: int = 0
    reasoning_tokens: int = 0
    request_count: int = 1
    raw_usage: Optional[dict[str, Any]] = None
```

**解析逻辑**: `parse_usage()` 从 OpenAI 的 `output_tokens_details.reasoning_tokens` 提取 reasoning_tokens (第 467–478 行)。目前仅用于 telemetry 和会话统计 (`session_reasoning_tokens`, `run_agent.py` ~8743)，**未单独计价**。

### 4.5 模型元数据

`agent/model_metadata.py` 没有专门的 reasoning capability flag。唯一相关 mention 是 Grok 模型 context length 的注释（如 `grok-4.20-0309-reasoning`）。

真正的 reasoning capability 元数据在 `agent/models_dev.py`:
```python
@dataclass(frozen=True)
class Capabilities:
    reasoning: bool = False
```
数据来源是上游 `models.dev` registry，用于 provider-aware 的 context 和 feature gating。

### 4.6 Provider 特殊处理

**Copilot ACP** (`copilot_acp_client.py` ~392–510): 通过 server-sent `session/update` 消息中的 `sessionUpdate: "agent_thought_chunk"` 流式传输 reasoning。`_run_prompt()` 返回 `(text_parts, reasoning_parts)`，将 Copilot 原生 thought chunks 归一化为 harness 通用的 `reasoning` 字段。

**Error Classifier** (`error_classifier.py` ~313–326): 将 Anthropic thinking-signature 400 错误分类为 `FailoverReason.thinking_signature`。检测**不**绑定 `provider == "anthropic"`，因为 OpenRouter 会代理 Anthropic 错误（此时 `provider` 显示为 `"openrouter"`）。判定条件: `status_code == 400` 且 `"signature" in error_msg` 且 `"thinking" in error_msg`。

**Auxiliary Client 安全锁** (`auxiliary_client.py` ~507–515): 通过 auxiliary client 调用 Anthropic 模型时，`reasoning_config` 被显式 hardcode 为 `None`。防止 vision/summarization 等辅助任务意外触发 Claude 4.6 上昂贵且耗时的 adaptive thinking。

---

## 5. 设计哲学总结

| 原则 | 体现 |
|---|---|
| **Normalization over leakage** | 每个 provider 的怪癖（role alternation、thinking signature、tool ID 格式、空 content 拒绝）都在 adapter 内处理。上层代码只讲 OpenAI format。 |
| **Turn-scoped fallback** | 降级按 turn 激活、自动恢复。防止瞬时 provider 故障永久拉低会话质量。 |
| **Credential pool 优先于 fallback** | Rate-limit 先触发凭证轮换，池子耗尽后才切 provider。 |
| **Adaptive vs legacy thinking 清晰分叉** | Anthropic adapter 对 Claude 4.6 adaptive 模式和旧版 `thinking.enabled` 模式做了干净分叉，各自维护常量表。 |
| **Codex reasoning 作为 opaque blobs** | Harness 从不解析 Codex 加密 reasoning，只保存并原样回放。 |
| **Reasoning content 作为一等字段** | 提取、保留在 message dict 中、计入 usage，但渲染通过 CLI callback——保持 `agent/` 核心无 UI 逻辑。 |
