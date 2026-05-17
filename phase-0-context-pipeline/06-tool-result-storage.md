# Hermes Context Pipeline — L6 Tool Result Storage & Budget Economics (Phase 0.2 补完)

> **目标**: 超越 Memory 视角，从 Harness 架构全局出发，系统性地理解 Hermes 的设计哲学、工程取舍和可迁移 insight。  
> **基础**: 已完成 Memory Architecture + Long-Context Problem 两层调研。  
> **产出**: 每完成一个 Phase，产出一份独立的 `hermes-<topic>-research.md`。  
> **版本**: Hermes Agent v0.9.0  
> **分析日期**: 2026-05-17

---

## 大白话介绍

如果 AI 跑了个命令返回了 10MB 日志，直接塞进对话里会让上下文窗口爆炸（既慢又贵）。Hermes 的 Tool Result Storage 就是这套" UPS 物流系统"——决定小包裹直接送达（inline），大包裹先存仓库（sandbox 文件系统），只送一张取货单和照片预览（preview+路径）。如果一次送了太多大包裹，整辆车超重了，就从最大的开始一个个往仓库搬，直到不超重为止。

这个系统分为三层:
- **L0 — 直接送达**: 单个包裹不大（<= 100K 字符），直接塞对话里
- **L1 — 单件存仓**: 单个包裹太大，写入 sandbox，对话里只留取货单+照片
- **L2 — 整车限重**: 一轮所有包裹加起来超过 200K 字符，从最大的开始强制搬仓

---

## 1. 架构概览：三层防御体系

模块通过 **3-tier persistence** 防止 tool result 撑爆 context window：

| 层级 | 函数/机制 | 触发时机 | 持久化目标 |
|------|-----------|----------|------------|
| L0 — Inline | `maybe_persist_tool_result` early-return | 单结果长度 ≤ 阈值 | **对话上下文**（无动作） |
| L1 — Per-result | `maybe_persist_tool_result` persist path | 单结果长度 > 阈值 | **Sandbox 文件系统** (`env.execute`) |
| L2 — Per-turn | `enforce_turn_budget` | 整轮总长度 > 聚合预算 | **Sandbox 文件系统**（强制 spill） |

此外，L0 还包含一层前置防御：各 tool 内部在执行前自行 truncate（如 `search_files` 的 `limit` 参数），这是 tool 作者控制的唯一防线。

---

## 2. 关键常量与配置

### 2.1 `tools/budget_config.py`（配置层）

```python
# budget_config.py:12-14 — Pinned 阈值，不可被覆盖
PINNED_THRESHOLDS: Dict[str, float] = {
    "read_file": float("inf"),   # 防止 read_file→persist→read_file 无限循环
}

# budget_config.py:18-20 — 全局默认值
DEFAULT_RESULT_SIZE_CHARS: int = 100_000   # L1 单工具阈值
DEFAULT_TURN_BUDGET_CHARS: int = 200_000   # L2 单轮总预算
DEFAULT_PREVIEW_SIZE_CHARS: int = 1_500    # 持久化后的预览长度
```

### 2.2 `tools/tool_result_storage.py`（实现层常量）

```python
# tool_result_storage.py:37-41
PERSISTED_OUTPUT_TAG = "<persisted-output>"
PERSISTED_OUTPUT_CLOSING_TAG = "</persisted-output>"
STORAGE_DIR = "/tmp/hermes-results"          # 无 env 时的默认落盘路径
HEREDOC_MARKER = "HERMES_PERSIST_EOF"
_BUDGET_TOOL_NAME = "__budget_enforcement__"  # L2 强制 spill 时使用的伪工具名
```

### 2.3 `BudgetConfig.resolve_threshold` 优先级链

```python
# budget_config.py:38-48
def resolve_threshold(self, tool_name: str) -> int | float:
    if tool_name in PINNED_THRESHOLDS:             # 1. Pinned（如 read_file=inf）
        return PINNED_THRESHOLDS[tool_name]
    if tool_name in self.tool_overrides:           # 2. 用户/RL 覆盖
        return self.tool_overrides[tool_name]
    from tools.registry import registry
    return registry.get_max_result_size(tool_name, default=self.default_result_size)
                                                   # 3. Registry 单工具注册值 -> 全局 default
```

---

## 3. 逐函数详细分析

### 3.1 `_resolve_storage_dir(env) -> str`

```python
# tool_result_storage.py:44-57
def _resolve_storage_dir(env) -> str:
    if env is not None:
        get_temp_dir = getattr(env, "get_temp_dir", None)
        if callable(get_temp_dir):
            try:
                temp_dir = get_temp_dir()
            except Exception as exc:
                logger.debug("Could not resolve env temp dir: %s", exc)
            else:
                if temp_dir:
                    temp_dir = temp_dir.rstrip("/") or "/"
                    return f"{temp_dir}/hermes-results"
    return STORAGE_DIR   # fallback: /tmp/hermes-results
```

- **职责**: 决定 sandbox 内的结果存储目录。
- **策略**: 优先通过 `env.get_temp_dir()` 获取环境特定的 temp 目录（兼容 Termux、Docker、Modal、Daytona 等），否则回退到硬编码的 `/tmp/hermes-results`。
- **注意**: 如果 `env` 为 `None`，直接回退到本地绝对路径 `/tmp/hermes-results`；但后续 `_write_to_sandbox` 因 `env is None` 不会执行，该路径仅作为 placeholder 出现在 truncation fallback 消息中。

---

### 3.2 `generate_preview(content, max_chars) -> tuple[str, bool]`

```python
# tool_result_storage.py:60-68
def generate_preview(content: str, max_chars: int = DEFAULT_PREVIEW_SIZE_CHARS) -> tuple[str, bool]:
    if len(content) <= max_chars:
        return content, False
    truncated = content[:max_chars]
    last_nl = truncated.rfind("\n")
    if last_nl > max_chars // 2:          # 仅当换行符位于后半段时才截断到此处
        truncated = truncated[:last_nl + 1]
    return truncated, True
```

- **职责**: 在 `max_chars`（默认 1,500）内生成可读预览，优先保留完整行。
- **策略**:
  1. 若内容本身更短，完整返回，`has_more=False`。
  2. 若更长，取前 `max_chars`；如果最后一条换行符位于区间后半段（> 750），则截断到该换行符，保证 preview 不以半截行结尾。
  3. 否则硬截断在 `max_chars`。
- **信息损失**: 超过 preview 的部分被截断；原始内容仅通过落盘文件恢复。

---

### 3.3 `maybe_persist_tool_result(...)` — L0 / L1 的核心决策

```python
# tool_result_storage.py:116-172
def maybe_persist_tool_result(
    content: str,
    tool_name: str,
    tool_use_id: str,
    env=None,
    config: BudgetConfig = DEFAULT_BUDGET,
    threshold: int | float | None = None,
) -> str:
```

#### 完整决策流程

```
输入: content, tool_name, tool_use_id, env, config, threshold(override)

Step 1: 解析 effective_threshold
    effective_threshold = threshold (若不为 None)
                       否则 config.resolve_threshold(tool_name)

Step 2: INF 短路
    if effective_threshold == float("inf"):
        return content          -- L0 inline, 该工具永不持久化

Step 3: 尺寸判断
    if len(content) <= effective_threshold:
        return content          -- L0 inline, 本次结果未超限

Step 4: 超限 -> 尝试 L1 sandbox 持久化
    storage_dir = _resolve_storage_dir(env)
    remote_path = f"{storage_dir}/{tool_use_id}.txt"
    preview, has_more = generate_preview(content, max_chars=config.preview_size)

    if env is not None:
        try:
            if _write_to_sandbox(content, remote_path, env):
                -- 成功: 返回 <persisted-output> 包装块
                return _build_persisted_message(preview, has_more, len(content), remote_path)
        except Exception as exc:
            logger.warning("Sandbox write failed for %s: %s", tool_use_id, exc)

Step 5: Fallback — inline truncation（所有持久化尝试均失败）
    return preview + "\n\n[Truncated: tool response was {len(content):,} chars. "
                     "Full output could not be saved to sandbox.]"
```

#### 分支总结

| 条件 | 结果 | 层级 |
|------|------|------|
| `threshold == inf` | 原样返回 | L0 Inline |
| `len(content) <= threshold` | 原样返回 | L0 Inline |
| `env` 可用且写入成功 | `<persisted-output>` + 文件路径 | L1 Sandbox |
| `env` 不可用或写入失败 | Inline truncation（无路径） | L0 Inline（有损） |

#### 参数与返回值（Public API）

| 参数 | 类型 | 说明 |
|------|------|------|
| `content` | `str` | 原始 tool result 字符串 |
| `tool_name` | `str` | 工具名，用于 threshold 查询 |
| `tool_use_id` | `str` | 本次调用的唯一 ID，也作为文件名 |
| `env` | `BaseEnvironment \| None` | 当前 sandbox 环境实例 |
| `config` | `BudgetConfig` | 预算配置，默认 `DEFAULT_BUDGET` |
| `threshold` | `int \| float \| None` | 显式阈值覆盖，最高优先级 |

**返回值**: `str` —— 替换后的内容，可直接放入 `messages[idx]["content"]`。

---

### 3.4 `enforce_turn_budget(tool_messages, env, config)` — L2 聚合预算

```python
# tool_result_storage.py:175-226
def enforce_turn_budget(
    tool_messages: list[dict],
    env=None,
    config: BudgetConfig = DEFAULT_BUDGET,
) -> list[dict]:
```

#### 实现逻辑（逐段）

```python
# tool_result_storage.py:188-195
candidates = []
total_size = 0
for i, msg in enumerate(tool_messages):
    content = msg.get("content", "")
    size = len(content)
    total_size += size
    if PERSISTED_OUTPUT_TAG not in content:          # 已持久化的结果不参与
        candidates.append((i, size))
```

```python
# tool_result_storage.py:197-198
if total_size <= config.turn_budget:
    return tool_messages          # 未超限，直接返回
```

```python
# tool_result_storage.py:200
candidates.sort(key=lambda x: x[1], reverse=True)   # 从大到小排序
```

```python
# tool_result_storage.py:202-224
for idx, size in candidates:
    if total_size <= config.turn_budget:
        break
    msg = tool_messages[idx]
    content = msg["content"]
    tool_use_id = msg.get("tool_call_id", f"budget_{idx}")
```

后面的代码从最大的开始强制 `maybe_persist_tool_result(..., threshold=0)`，直到总大小降到预算以下。

#### 截断策略

- **目标**: 将整轮 tool result 总字符数压到 `config.turn_budget`（默认 200,000）以下。
- **选择策略**: 贪心最大优先（largest non-persisted first）。
- **操作**: 对每个候选调用 `maybe_persist_tool_result(..., threshold=0)`，强制其进入持久化/截断流程。
- **已持久化结果**: 通过 `PERSISTED_OUTPUT_TAG` 检测并跳过，避免重复处理。
- **信息损失与恢复**:
  - 被 spill 的结果由完整内联文本变为 `<persisted-output>` 包装块（路径 + preview）。
  - 只要有 sandbox 环境，模型后续可通过 `read_file` + `offset/limit` 完整恢复原始内容。
  - 若 `env=None`，fallback 为纯 inline truncation，信息永久丢失（仅保留 1,500 chars preview）。

**注意**: `enforce_turn_budget` **原地 mutate** `tool_messages` list，返回值即同一份引用。

---

## 4. 数据流图（ASCII）

```
+-----------------+      tool result string
|   Tool Execution|----------------------------+
+-----------------+                            |
                                               v
                          +----------------------------------------+
                          |   maybe_persist_tool_result()          |
                          |   (L0/L1: Per-result threshold)        |
                          +----------------------------------------+
                                               |
                    +--------------------------+--------------------------+
                    |                                                     |
         len <= threshold                                          len > threshold
         or threshold == inf                                              |
                    |                                                     v
                    |                    +--------------------------------+
                    |                    |  _write_to_sandbox()           |
                    |                    |  via env.execute(cmd, timeout=30)
                    |                    +--------------------------------+
                    |                          |              |
                    |                       success         fail / env=None
                    |                          |              |
                    v                          v              v
            +-----------+            +----------------+  +-------------------+
            | L0 Inline |            | L1 Persisted   |  | L0 Inline Trunc   |
            | (content) |            | (<persisted>   |  | (preview +        |
            +-----------+            |  preview+path) |  |  [Truncated...])  |
                                     +----------------+  +-------------------+
                    |                          |              |
                    +--------------------------+--------------+
                                               |
                                               v
                                    messages.append(tool_msg)
                                               |
                                               v
                          +----------------------------------------+
                          |   enforce_turn_budget()                |
                          |   (L2: Per-turn aggregate budget)      |
                          +----------------------------------------+
                                               |
                        +----------------------+----------------------+
                        |                                             |
              total <= turn_budget                               total > turn_budget
                        |                                             |
                        v                                             v
                  +-----------+                    +----------------------------------+
                  | No action |                    | Sort candidates by size desc     |
                  +-----------+                    | loop: call maybe_persist with    |
                                                   |       threshold=0                |
                                                   | until under budget               |
                                                   +----------------------------------+
                                                                      |
                                                                      v
                                                          +---------------------+
                                                          | Update tool_msg in  |
                                                          | place               |
                                                          +---------------------+
```

---

## 5. 与 `run_agent.py` 的 Public API 接口

`tools/tool_result_storage.py` 仅暴露两个函数给上层，调用者共有 **两处**：

### 5.1 `run_agent.py`

```python
# run_agent.py:70
from tools.tool_result_storage import maybe_persist_tool_result, enforce_turn_budget
```

**调用点 A:** `_execute_tool_calls_parallel`（并行工具执行分支，约 line 7117）

```python
# run_agent.py:7117-7122
function_result = maybe_persist_tool_result(
    content=function_result,
    tool_name=name,
    tool_use_id=tc.id,
    env=get_active_env(effective_task_id),
)
```

- 每个工具执行完后立即调用，执行 L0/L1 判断。
- `config` 使用默认 `DEFAULT_BUDGET`。

```python
# run_agent.py:7137-7139
num_tools = len(parsed_calls)
if num_tools > 0:
    turn_tool_msgs = messages[-num_tools:]
    enforce_turn_budget(turn_tool_msgs, env=get_active_env(effective_task_id))
```

- 一轮内所有工具执行完毕后，取最后 `num_tools` 条 message 执行 L2 聚合预算。

**调用点 B:** `_execute_tool_calls_sequential`（串行工具执行分支，约 line 7439 & 7485）

与并行分支逻辑完全一致：先逐个 `maybe_persist_tool_result`，再整轮 `enforce_turn_budget`。

### 5.2 `environments/agent_loop.py`

```python
# environments/agent_loop.py:25
from tools.tool_result_storage import maybe_persist_tool_result, enforce_turn_budget
```

**调用点:** 约 line 458 & 476

```python
# environments/agent_loop.py:458-464
tool_result = maybe_persist_tool_result(
    content=tool_result,
    tool_name=tool_name,
    tool_use_id=tc_id,
    env=get_active_env(self.task_id),
    config=self.budget_config,          # 显式传入 BudgetConfig
)

# environments/agent_loop.py:476-480
enforce_turn_budget(
    messages[-num_tcs:],
    env=get_active_env(self.task_id),
    config=self.budget_config,
)
```

- `agent_loop` 支持 RL 场景，使用实例变量 `self.budget_config`，允许环境级别覆盖阈值。

---

## 6. Token / Cost Tracking & Budget Economics（全局分析）

### 6.1 LLM API 成本估算层 (`agent/usage_pricing.py`)

```python
# agent/usage_pricing.py
class CanonicalUsage:
    input_tokens: int
    output_tokens: int
    cache_read_tokens: int
    cache_write_tokens: int
    reasoning_tokens: int
    request_count: int = 1
    raw_usage: Any = None

class CostResult:
    amount_usd: Decimal | None
    status: CostStatus          # "actual" | "estimated" | "included" | "unknown"
    source: CostSource          # 定价数据来源
    label: str
    fetched_at: str
    pricing_version: str
    notes: str
```

- `prompt_tokens` 属性 = `input_tokens + cache_read_tokens + cache_write_tokens`
- `total_tokens` 属性 = `prompt_tokens + output_tokens`

**官方定价快照** (`_OFFICIAL_DOCS_PRICING`)：

键为 `(provider, model)`，当前硬编码了 Anthropic 系列模型的 cache pricing，例如：

- `claude-opus-4-20250514`: input $15/M, output $75/M, cache read $1.50/M, cache write $18.75/M
- `claude-sonnet-4-20250514`: input $3/M, output $15/M, cache read $0.30/M, cache write $3.75/M

**估算入口**：

```python
estimate_usage_cost(
    model: str,
    usage: CanonicalUsage,
    provider: str | None = None,
    base_url: str | None = None,
    api_key: str = "",
) -> CostResult
```

逻辑优先级：`provider cost API` → `official docs snapshot` → `user override` → `unknown`。

### 6.2 全局 Session 级 Token / Cost 累加器 (`run_agent.py`)

**初始化位置**: `run_agent.py:1395-1407`

```python
self.session_prompt_tokens = 0
self.session_completion_tokens = 0
self.session_total_tokens = 0
self.session_api_calls = 0
self.session_input_tokens = 0
self.session_output_tokens = 0
self.session_cache_read_tokens = 0
self.session_cache_write_tokens = 0
self.session_reasoning_tokens = 0
self.session_estimated_cost_usd = 0.0
self.session_cost_status = "unknown"
self.session_cost_source = "none"
```

`reset_session_state()`（line 1478）可一键清零以上全部计数器，并调用 `context_compressor.on_session_reset()`。

**运行时累加逻辑**: `run_agent.py` 的主 LLM 调用后（约 line 8708–8795）

```python
# 1. 将 provider-specific usage 归一化为 CanonicalUsage
canonical_usage = normalize_usage(response.usage, provider=self.provider, api_mode=self.api_mode)

# 2. 更新 context_compressor（用于上下文压缩决策）
self.context_compressor.update_from_response(usage_dict)

# 3. Session 累加
self.session_prompt_tokens       += prompt_tokens
self.session_completion_tokens   += completion_tokens
self.session_total_tokens        += total_tokens
self.session_api_calls           += 1
self.session_input_tokens        += canonical_usage.input_tokens
self.session_output_tokens       += canonical_usage.output_tokens
self.session_cache_read_tokens   += canonical_usage.cache_read_tokens
self.session_cache_write_tokens  += canonical_usage.cache_write_tokens
self.session_reasoning_tokens    += canonical_usage.reasoning_tokens

# 4. 实时成本估算
cost_result = estimate_usage_cost(
    self.model, canonical_usage,
    provider=self.provider, base_url=self.base_url, api_key=getattr(self, "api_key", ""),
)
if cost_result.amount_usd is not None:
    self.session_estimated_cost_usd += float(cost_result.amount_usd)
self.session_cost_status = cost_result.status
self.session_cost_source = cost_result.source

# 5. 写入 SQLite session DB（供 /insights 查询）
if self._session_db and self.session_id:
    self._session_db.update_token_counts(
        self.session_id,
        input_tokens=canonical_usage.input_tokens,
        output_tokens=canonical_usage.output_tokens,
        cache_read_tokens=canonical_usage.cache_read_tokens,
        cache_write_tokens=canonical_usage.cache_write_tokens,
        reasoning_tokens=canonical_usage.reasoning_tokens,
        estimated_cost_usd=float(cost_result.amount_usd) if cost_result.amount_usd is not None else None,
        cost_status=cost_result.status,
        cost_source=cost_result.source,
        billing_provider=self.provider,
        billing_base_url=self.base_url,
        billing_mode="subscription_included" if cost_result.status == "included" else None,
        model=self.model,
    )
```

**要点**：
- 累加是 **per-session**、**per-API-call** 的实时行为。
- 成本按 **单次调用** 估算后累加，不存在跨调用的批量折扣逻辑。
- 数据同时落盘到 SQLite `sessions` / `messages` 表，保证 gateway、cron、delegated run 的会计数据不丢失。

### 6.3 Preflight Token 估算与 L4 压缩的衔接

`agent/model_metadata.py:1071` 的 `estimate_request_tokens_rough`：

```python
def estimate_request_tokens_rough(
    messages: List[Dict[str, Any]],
    *,
    system_prompt: str = "",
    tools: Optional[List[Dict[str, Any]]] = None,
) -> int:
    total_chars = 0
    if system_prompt:
        total_chars += len(system_prompt)
    if messages:
        total_chars += sum(len(str(msg)) for msg in messages)
    if tools:
        total_chars += len(str(tools))      # 显式计入 tool schema tokens（20-30K+）
    return (total_chars + 3) // 4           # 1 token ≈ 4 chars
```

**调用位置**: `run_agent.py:7889` + `7925`

每次 API 调用前，若消息数超过 `protect_first_n + protect_last_n + 1`，先估算当前 request 的 token 数，决定是否触发 L4 context compression。压缩完成后重新估算，最多循环 3 次。

### 6.4 CLI 中的使用展示

- **`/usage` 命令**: 实时读取 session 累加器，输出 token breakdown + cost estimate
- **`/insights` 命令**: 通过 `agent.insights.InsightsEngine` 查询 SQLite DB，生成跨天数的聚合报告
- **TUI Status Bar**: 实时展示 `context_tokens / context_length` 及百分比（ASCII 进度条），但**不直接展示累计费用**。费用需通过 `/usage` 查看。

### 6.5 Gateway 中的 `_agent_cache` 与 Prompt Caching 经济学

Gateway 通过 per-session 的 `_agent_cache` 缓存 `AIAgent` 实例，核心动机之一是 **保留 Anthropic 的 prefix prompt cache**：

> "without caching, prefix cache is rebuilt every turn, costing ~10× more"

这说明 **token/cost tracking 与 caching 策略深度耦合**——cache miss 会直接体现在 `session_cache_write_tokens` 和估算成本中。

### 6.6 三层独立预算体系总结

| 层级 | 预算单位 | 硬上限 | 告警/拦截 | 与哪层交互 |
|------|----------|--------|-----------|------------|
| API Cost（USD） | 美元 | ❌ 无 | ❌ 无（仅观测） | `usage_pricing.py` |
| Tool Result（chars） | 字符 | ✅ L1=100K, L2=200K | 静默 spill / truncation | `tool_result_storage.py` |
| Iteration（turns） | 调用次数 | ✅ 默认90轮 | CLI 打印警告 | `run_agent.py` |

---

## 7. Tool Loop 中 Budget 决策完整调用链

```
API Response (assistant_message with tool_calls)
    │
    ▼
run_agent.py: run_conversation() 主循环
    │
    ├──► _execute_tool_calls(assistant_message, messages, ...)
    │       │
    │       ├──► 判断并行/顺序：_should_parallelize_tool_batch()
    │       │
    │       ├──► 顺序路径：_execute_tool_calls_sequential()
    │       │       ├──► 逐条调用 handle_function_call() / _invoke_tool()
    │       │       ├──► 得到原始 function_result (字符串)
    │       │       ├──► maybe_persist_tool_result()    ← L1（单结果持久化）
    │       │       │       ├── 超限 ──► 写入 sandbox /tmp/hermes-results/{tc.id}.txt
    │       │       │       └── 返回 preview / persisted tag / inline truncation
    │       │       ├──► 组装 tool_msg {"role":"tool", "content": result, "tool_call_id": tc.id}
    │       │       └──► messages.append(tool_msg)
    │       │       (loop 结束)
    │       │       └──► enforce_turn_budget(messages[-num_tools_seq:])  ← L2（整轮预算）
    │       │
    │       └──► 并行路径：_execute_tool_calls_concurrent()
    │               ├──► ThreadPool 执行各 tool
    │               ├──► 每条线程内同样调用 maybe_persist_tool_result()
    │               ├──► 按原顺序收集 results
    │               ├──► 逐条 messages.append(tool_msg)
    │               (loop 结束)
    │               └──► enforce_turn_budget(messages[-num_tools:])       ← L2
    │
    ├──► estimate_request_tokens_rough(messages, system_prompt, tools)    ← Preflight 估算
    │       │
    │       └── 若 >= threshold_tokens ──► _compress_context()            ← L4 Context Compression
    │               ├──► ContextCompressor.compress()
    │               ├──► _prune_old_tool_results()   （清理旧 tool result）
    │               ├──► _generate_summary()         （LLM 摘要中间轮次）
    │               └──► _sanitize_tool_pairs()      （修复 orphaned tool_call/tool_result）
    │
    └──► 进入下一轮 while 循环
```

---

## 8. 设计疑点与潜在 Bug

### 8.1 🔴 疑点：`enforce_turn_budget` 的 `total_size` 递减可能产生余量误判

当 `env=None` 时，`maybe_persist_tool_result(threshold=0)` 会 fallback 到 inline truncation。`replacement` 是 `preview + "[Truncated: ...]"`，其长度约 1,500~2,000 字符，远小于 `size`，因此 budget 下降有效。

**但若 `env` 存在且 `_write_to_sandbox` 成功**，`replacement` 变为 `<persisted-output>` 块，其中包含文件路径、说明文字和 preview。`len(replacement)` 通常 > `config.preview_size`（可能在 1,800~2,500 左右）。

`total_size` 的更新只发生在 `replacement != content` 时。如果某个 candidate 在 spill 后因为 `<persisted-output>` 块本身仍很长，budget 缩减有限，可能需要 spill 更多 candidate 才能达标。这属于预期行为，但 **削减量的上界受限**（由 preview + 元信息长度决定）。

### 8.2 🟡 疑点：`_BUDGET_TOOL_NAME` 的 Registry 查询语义

`enforce_turn_budget` spill 时调用 `maybe_persist_tool_result(..., tool_name="__budget_enforcement__", threshold=0)`。

- 由于 `threshold=0` 显式传入，`resolve_threshold` 被短路，不会查询 registry。
- 因此该伪工具名不会触发 `PINNED_THRESHOLDS` 或 registry 逻辑，**无实际影响**。但代码语义略显奇怪。

### 8.3 🟢 安全设计亮点：`read_file` Pinned 为 `inf`

```python
PINNED_THRESHOLDS = {"read_file": float("inf")}
```

完美防止了以下无限循环：
1. Tool A 返回大结果 -> persisted to file
2. LLM 调用 `read_file` 读取该文件
3. `read_file` 返回文件内容（同样很大）-> 若被再次 persist -> 无限循环

`read_file` 的结果永远 inline，即使超过 100K 也不 spill，这是对 persisted output 恢复路径的必要保护。

### 8.4 🟡 疑点：`STORAGE_DIR` 回退路径的误导性

当 `env=None` 时，`_resolve_storage_dir` 返回 `/tmp/hermes-results`。在 `_write_to_sandbox` 中 `env is None` 会被跳过，但 `remote_path` 仍以 `/tmp/hermes-results/{tool_use_id}.txt` 出现。

- 在 fallback truncation message 中不会显示路径（纯 truncation）。
- 若在 LLM message 中硬编码了 `/tmp/hermes-results/xxx.txt`，而实际进程运行于容器/远端，`read_file` 将找不到文件。**实际上 fallback 消息不含路径，因此该风险不暴露给 LLM，安全。**

### 8.5 🟡 不存在 `--max-cost` 参数

全库搜索确认无任何 USD 成本的硬预算 CLI 参数或自动告警拦截机制。成本仅用于观测（observability），不做 enforcement。

---

## 9. 文件 I/O 路径总结

| 场景 | 目录 | 文件名 | 写入方式 | 读取方式 |
|------|------|--------|----------|----------|
| L1 Per-result persist | `{env_temp_dir}/hermes-results` 或 `/tmp/hermes-results` | `{tool_use_id}.txt` | `env.execute(heredoc, timeout=30)` | `read_file` tool |
| L2 Budget enforcement spill | 同上 | `{tool_call_id}` 或 `budget_{idx}.txt` | 同上（经由 `maybe_persist_tool_result`） | `read_file` tool |
| Fallback（env=None） | 无实际文件 | N/A | N/A | 仅 inline preview |

---

## 10. 测试覆盖速览

`tests/tools/test_tool_result_storage.py` 覆盖：

- `generate_preview` — 正常截断、末尾换行、无截断。
- `_build_persisted_message` — tag/size/路径格式。
- `_write_to_sandbox` — heredoc 写入、碰撞 marker、目录创建、失败回退。
- `maybe_persist_tool_result`:
  - 小结果 inline
  - 大结果 persist（mock env 成功/失败）
  - `threshold=inf` 短路
  - `env=None` fallback truncation
  - 自定义 `BudgetConfig`
- `enforce_turn_budget`:
  - budget 内无变化
  - 超 budget 时最大优先 spill
  - 已 persisted 结果跳过
  - 6x42K 聚合回归测试（中结果组合超限场景）
  - `env=None` fallback
- `BudgetConfig.resolve_threshold` 与 `registry.get_max_result_size` 集成：
  - `read_file` = `inf`
  - `terminal` / `search_files` = 100K

---

## 11. 结论

`tools/tool_result_storage.py` 实现了一个设计清晰的三层 tool result 容量控制系统：

1. **L0 Inline** 通过大小阈值快速放行小结果。
2. **L1 Sandbox Persist** 在单结果超限时通过 `env.execute()` 写入跨后端统一的 temp 目录，并以内联 preview + 路径引用替换原文，保证 LLM 仍可访问完整输出。
3. **L2 Turn Budget** 在整轮聚合超限后贪心 spill 最大的未持久化结果，作为最终兜底。

关键常量（100K / 200K / 1,500）与 `read_file` 的 `inf` pinned 阈值共同构成了可配置、可防御循环、可跨环境（local/Docker/SSH/Modal/Daytona）运行的持久化方案。

同时，Tool Result Budget 与 LLM API Cost Tracking（`usage_pricing.py`）是两个**完全正交**的系统：
- **Tool Result Budget** 管理空间维度单轮结果字符膨胀（L0/L1/L2 spill），预算单位是**字符**。这是 context 质量的控制阀。
- **API Cost Tracking** 管理时间维度累计 API 调用成本（USD），预算单位是**美元**，但**没有硬上限**，仅用于观测。这是财务报表。

两者协同通过 `ContextCompressor` 间接耦合：L2 spill 减少了 input messages 的字符数，从而降低了 preflight token 估算值，延迟或避免了 L4 context 压缩的触发，最终节省了 API 调用成本。
