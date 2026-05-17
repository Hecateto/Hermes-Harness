# Hermes Context Pipeline — L3 Prefix Cache / Prompt Caching 设计 (Phase 0.3 补完)

> **目标**: 深入理解 Hermes 如何利用 Anthropic/Claude 的 prefix caching 机制降低多轮对话成本，以及 prompt builder 的 layer ordering 和 frozen snapshot 如何为此服务。  
> **基础**: 已完成 Memory Architecture (L2) + Tool Result Storage (L6) 调研。  
> **版本**: Hermes Agent v0.9.0  
> **分析日期**: 2026-05-17

---

## 大白话介绍

Anthropic 的 Claude 有一个"记忆背诵"的捷径：如果你这一轮发送的 prompt 前缀和上一轮完全一样，它不需要重新计算前缀部分的注意力，直接复用上轮的缓存。这个特性能把多轮对话的前缀成本降到 1/10。

Hermes 为了最大化利用这个捷径，做了三件核心设计：
1. **冻结 system prompt**：每轮对话开头的系统指令（identity + memory + skills + context files）在整段会话中**逐字不变**，确保前缀始终 match。
2. **4 个 cache 标记点**：在每轮 API 请求前，在 system prompt + 最近 3 条消息上打 `cache_control` 标记，告诉 Anthropic "把这些段落缓存起来”。
3. **分离易变内容**：临时指令（ephemeral system prompt）、prefill、plugin context 等可变内容绝不写入缓存的 system prompt，只在 API 调用时动态拼接，避免污染前缀。

---

## 1. 核心模块：`agent/prompt_caching.py`

### 1.1 整体策略：`system_and_3`

| 属性 | 说明 |
|------|------|
| 策略名 | `system_and_3` |
| Breakpoint 上限 | 4 个（Anthropic API 硬性上限） |
| Breakpoint 分布 | **第 1 个**：首条 `role: system` 消息<br>**第 2–4 个**：最近 3 条非 system 消息（滑动窗口） |
| 函数特性 | 纯函数，无类状态，deep-copy 输入列表，不污染原始消息 |
| 调用入口 | `run_agent.py:8141` 每次 API 调用前注入 |

```python
# agent/prompt_caching.py:41-72
def apply_anthropic_cache_control(
    api_messages: List[Dict[str, Any]],
    cache_ttl: str = "5m",
    native_anthropic: bool = False,
) -> List[Dict[str, Any]]:
    messages = copy.deepcopy(api_messages)          # 无副作用
    marker = {"type": "ephemeral"}
    if cache_ttl == "1h":
        marker["ttl"] = "1h"                        # 注：非标准字段，可能被 API 忽略

    breakpoints_used = 0
    if messages[0].get("role") == "system":
        _apply_cache_marker(messages[0], marker, native_anthropic)
        breakpoints_used += 1

    remaining = 4 - breakpoints_used
    non_sys = [i for i in range(len(messages)) if messages[i].get("role") != "system"]
    for idx in non_sys[-remaining:]:
        _apply_cache_marker(messages[idx], marker, native_anthropic)
    return messages
```

### 1.2 `_apply_cache_marker` — 按 role 与 content 类型分 4 分支

```python
# agent/prompt_caching.py:15-38
def _apply_cache_marker(msg: dict, cache_marker: dict, native_anthropic: bool = False) -> None:
    role = msg.get("role", "")
    content = msg.get("content")

    # 分支 A: role == "tool"
    if role == "tool":
        if native_anthropic:
            msg["cache_control"] = cache_marker    # Native: 允许 top-level
        return                                       # OpenRouter: 直接跳过

    # 分支 B: content 为 None 或空串（常见于 tool_calls 消息的 content=""）
    if content is None or content == "":
        msg["cache_control"] = cache_marker
        return

    # 分支 C: content 为 string -> 升级为 list-of-blocks
    if isinstance(content, str):
        msg["content"] = [
            {"type": "text", "text": content, "cache_control": cache_marker}
        ]
        return

    # 分支 D: content 为 list -> 只在最后一个 part 注入
    if isinstance(content, list) and content:
        last = content[-1]
        if isinstance(last, dict):
            last["cache_control"] = cache_marker
```

| 场景 | 注入位置 | 备注 |
|------|----------|------|
| `role == "system"` | 取决于 content 类型（list 时注入 `content[-1]`） | 始终占 1 个 breakpoint |
| `role == "tool"` + `native_anthropic=True` | message 顶层 | 后续由 `anthropic_adapter.py` 迁移到 `tool_result` block |
| `role == "tool"` + `native_anthropic=False` | **Skip，不注入** | OpenRouter 兼容处理 |
| `content == ""` | message 顶层 | assistant 的 tool_calls 消息常见 |
| `content` 为 string | 包装为唯一 text block 并注入 | 降级为 list-of-parts |
| `content` 为 list | `content[-1]`（最后一个 dict part） | 避免中间 block 被标记 |

### 1.3 OpenRouter 兼容逻辑

当目标为 OpenRouter（`native_anthropic=False`）时：
- **Tool message 被完全 skip**。OpenRouter 的 Claude 代理对 `role: tool` 上的 top-level `cache_control` 不兼容，会导致 **silent hang**（请求静默挂起，无错误返回）。
- **System / User / Assistant 消息正常注入**。`cache_control` 挂在 content block 或 message 顶层，OpenRouter 会透传至下游 Anthropic 模型。

对应 RELEASE_v0.4.0 的修复说明：
> Fix: skip top-level `cache_control` on role:tool for OpenRouter ([#2391](https://github.com/NousResearch/hermes-agent/pull/2391))

---

## 2. 不同 Provider 的 cache_control 策略矩阵

| Provider / 通路 | 启用条件 | system 支持 | tool 支持 | `native_anthropic` | 特殊行为 |
|-----------------|----------|-------------|-----------|--------------------|----------|
| **Anthropic Native** | `provider=="anthropic"` + `api_mode=="anthropic_messages"` | ✅ 最后一个 block | ✅ top-level → adapter 迁到 `tool_result` block | `True` | 原生 `cache_read_input_tokens` / `cache_creation_input_tokens` 读数 |
| **OpenRouter + Claude** | base_url 含 openrouter + model 名含 "claude" | ✅ 最后一个 block | ❌ **Skip**（避免 silent hang） | `False` | `prompt_tokens_details.cached_tokens` 读数 |
| **OpenAI / Google / Fireworks / Mistral** | 不启用 | ❌ | ❌ | N/A | 无 prompt caching 逻辑 |
| **Qwen Portal** | `_is_qwen_portal` 判定 | ✅ 仅 system 最后一个 part | ❌ | N/A | 独立实现，只注入 system，不采用 `system_and_3` |

---

## 3. Frozen Snapshot + Prefix Cache 的三重保障

### 3.1 为什么 system prompt 稳定性是 cache 生效的前提

Anthropic 的 prompt caching 是 **prefix-based（前缀匹配）**：只有当本轮请求的 token 前缀与上轮某次请求的 prefix **逐字节一致**时，KV cache 才能命中。

如果 system prompt 在中途发生变化（例如 memory tool 写入了新条目），下一轮的 prefix 就会改变 → **cache miss** → system prompt 所有 token 需要重新计费（通常 1x 而非 0.1x），费用瞬间上升约 **10 倍**，延迟也增加。

Hermes 通过三层机制防止 prefix 漂移：

#### 保障 1：MemoryStore 的 `_system_prompt_snapshot`

```python
# tools/memory_tool.py:100-117
class MemoryStore:
    """
    Maintains two parallel states:
      - _system_prompt_snapshot: frozen at load time, used for system prompt injection.
        Never mutated mid-session. Keeps prefix cache stable.
      - memory_entries / user_entries: live state, mutated by tool calls, persisted to disk.
        Tool responses always reflect this live state.
    """
```

- `load_from_disk()` 在会话开始时捕获 MEMORY.md / USER.md 内容，生成 snapshot。
- 会话中 `memory add/replace/remove` 只修改 live entries 并写回磁盘，**不修改 snapshot**。
- 因此注入 system prompt 的 memory block 在整段会话中 **bit-perfect stable**。

#### 保障 2：`AIAgent._cached_system_prompt` 的跨 turn 冻结

```python
# run_agent.py:3097-3103
def _build_system_prompt(self, system_message: str = None) -> str:
    """
    Called once per session (cached on self._cached_system_prompt) and only
    rebuilt after context compression events. This ensures the system prompt
    is stable across all turns in a session, maximizing prefix cache hits.
    """
```

- `_cached_system_prompt` 在首次调用时构建，之后整段会话复用。
- 继续已有会话时，从 SQLite `session_db` 读取历史 system prompt，**避免重新读取磁盘 memory 文件**导致 prefix 漂移：

```python
# run_agent.py:7823-7873
if self._cached_system_prompt is None:
    stored_prompt = None
    if conversation_history and self._session_db:
        session_row = self._session_db.get_session(self.session_id)
        if session_row:
            stored_prompt = session_row.get("system_prompt") or None

    if stored_prompt:
        # Continuing session — reuse the exact system prompt from
        # the previous turn so the Anthropic cache prefix matches.
        self._cached_system_prompt = stored_prompt
    else:
        # First turn of a new session — build from scratch.
        self._cached_system_prompt = self._build_system_prompt(system_message)
        self._session_db.update_system_prompt(self.session_id, self._cached_system_prompt)
```

#### 保障 3：Gateway 的 `_agent_cache` 按 session 复用 AIAgent 实例

```python
# gateway/run.py:577-584
# Cache AIAgent instances per session to preserve prompt caching.
# Without this, a new AIAgent is created per message, rebuilding the
# system prompt (including memory) every turn — breaking prefix cache
# and costing ~10x more on providers with prompt caching (Anthropic).
self._agent_cache: Dict[str, tuple] = {}
self._agent_cache_lock = _threading.Lock()
```

- Gateway 默认 per-message 创建 `AIAgent`。如果不缓存实例，每轮都会 rebuild system prompt → prefix 变化 → cache miss。
- `_agent_cache` 按 `session_key` 复用 `AIAgent`，`_cached_system_prompt` 和 memory snapshot 在整个 session 期间稳定。

---

## 4. 动态内容与 Prefix Cache 的隔离

### 4.1 哪些内容**不**参与 prefix cache？

| 内容类型 | 是否参与 Prefix Cache | 注入时机 | 原因 |
|----------|----------------------|----------|------|
| `_cached_system_prompt` | ✅ 是 | Session start / Compression rebuild |  frozen，逐字节稳定 |
| `frozen snapshot` (memory/user) | ✅ 是 | 静态包含在 cached system prompt 中 | 写后不变 |
| `ephemeral_system_prompt` | ❌ 否 | API-call time 动态拼接 | 临时/可变指令，若参与 cache 会破坏 prefix 稳定性 |
| `prefill_messages` | ❌ 否 | API-call time 插入 user message | 每轮可能不同 |
| Plugin context (pre_llm_call hooks) | ❌ 否 | API-call time 注入到 user message | 插件产出不可预测 |
| L4 Compressor 重建后的 system prompt | ✅ 是（重建后） | Compression event 后 rebuild | 重建后生成新的稳定 prompt，再次固化到 DB |

### 4.2 `ephemeral_system_prompt` 的隔离设计

```python
# run_agent.py:3175-3176
# Note: ephemeral_system_prompt is NOT included here. It's injected at
# API-call time only so it stays out of the cached/stored system prompt.
```

实际注入位置：

```python
# run_agent.py:8120-8127
effective_system = self._cached_system_prompt or ""
if self.ephemeral_system_prompt:
    effective_system = (effective_system + "\n\n" + self.ephemeral_system_prompt).strip()
if effective_system:
    api_messages = [{"role": "system", "content": effective_system}] + api_messages
```

**关键**: `ephemeral_system_prompt` 只拼接到 **API 调用时的副本**上，绝不写入 `_cached_system_prompt` 或 SQLite session DB。这保证了一方面 ephemeral 内容不影响跨 turn 的 prefix 稳定性，另一方面 ephemeral 指令（如 RL system prompt、skills 即时指导）不会被固化到 session 恢复数据中。

---

## 5. Anthropic Adapter 对 cache_control 的透传与清理

`agent/anthropic_adapter.py` 负责将 OpenAI-format messages 转换为 Anthropic native format，同时处理 `cache_control` 的保留与剥离。

### 5.1 System prompt 的 cache_control 保留

```python
# agent/anthropic_adapter.py:939-946
if role == "system":
    if isinstance(content, list):
        has_cache = any(p.get("cache_control") for p in content if isinstance(p, dict))
        if has_cache:
            system = [p for p in content if isinstance(p, dict)]   # 保留 list-of-blocks
        else:
            system = "\n".join(p["text"] for p in content if p.get("type") == "text")
    else:
        system = content
```

- 当 system message 的 content list 中存在 `cache_control` 时，system prompt 以 **list of blocks** 形式返回（而非扁平字符串），从而把 marker 完整传给 Anthropic SDK。

### 5.2 Tool result 的 cache_control 透传

```python
# agent/anthropic_adapter.py:991-997
tool_result = {
    "type": "tool_result",
    "tool_use_id": _sanitize_tool_id(m.get("tool_call_id", "")),
    "content": result_content,
}
if isinstance(m.get("cache_control"), dict):
    tool_result["cache_control"] = dict(m["cache_control"])
```

- 把 `role: tool` message 顶层（native Anthropic 模式下由 `prompt_caching.py` 注入）的 `cache_control` 迁移到 Anthropic 格式的 `tool_result` block 内部。
- OpenRouter 模式下 `prompt_caching.py` 根本不会向 `role: tool` 注入顶层 marker，因此这里自然无内容可透传。

### 5.3 Thinking block 的 cache_control 强制剥离

```python
# agent/anthropic_adapter.py:1178-1182
# Strip cache_control from any remaining thinking/redacted_thinking
# blocks — cache markers interfere with signature validation.
for b in m["content"]:
    if isinstance(b, dict) and b.get("type") in _THINKING_TYPES:
        b.pop("cache_control", None)
```

- `cache_control` 标记会干扰 thinking content 的 signature 校验，因此 adapter 在转换前强制剥离 thinking block 上的 marker。

---

## 6. L4 Context Compressor 对 Prefix Cache 的影响

### 6.1 压缩触发时会重建 system prompt

```python
# run_agent.py:6724+
compressed = self.context_compressor.compress(messages, ...)
self._invalidate_system_prompt()
new_system_prompt = self._build_system_prompt(system_message)
self._cached_system_prompt = new_system_prompt
if self._session_db:
    self._session_db.update_system_prompt(self.session_id, new_system_prompt)
```

- `_invalidate_system_prompt()` 清空 `_cached_system_prompt`，并重新从 disk 加载 memory。
- 然后重新调用 `_build_system_prompt()`，生成**新的** system prompt。

### 6.2 cache_control 在压缩后是重新注入的

关键时序：
1. 内部历史 `messages` 进入 `context_compressor.compress()`（修改 conversation history 部分）。
2. `api_messages` 从内部历史 + 临时拼接项组装而成。
3. **最后**调用 `apply_anthropic_cache_control(api_messages, ...)` 对 `api_messages` 动态注入。

因此：
- 压缩不保留旧的 cache_control 标记；旧的标记随 `api_messages` 每轮重新构建而丢失。
- 但由于 `apply_anthropic_cache_control` 在 API 调用前强制重新注入，cache_control **始终存在**。
- 副作用：压缩导致 system prompt 内容变化，会触发一次 **cache creation（write cost，1.25x）**，但新的 system prompt 在后续多轮对话中重新稳定，再次享受 **cache read（0.1x）**收益。

---

## 7. Cache Metrics 日志与成本追踪

每次 API 调用后，`run_agent.py` 输出 cache hit 率统计：

```python
# run_agent.py:8801-8814
if self._use_prompt_caching:
    if self.api_mode == "anthropic_messages":
        cached = getattr(response.usage, 'cache_read_input_tokens', 0) or 0
        written = getattr(response.usage, 'cache_creation_input_tokens', 0) or 0
    else:
        details = getattr(response.usage, 'prompt_tokens_details', None)
        cached = getattr(details, 'cached_tokens', 0) or 0 if details else 0
        written = getattr(details, 'cache_write_tokens', 0) or 0 if details else 0
    prompt = usage_dict["prompt_tokens"]
    hit_pct = (cached / prompt * 100) if prompt > 0 else 0
    self._vprint(f"... Cache: {cached:,}/{prompt:,} tokens ({hit_pct:.0f}% hit, {written:,} written)")
```

- **Anthropic Native**: 直接读取 `cache_read_input_tokens` / `cache_creation_input_tokens`。
- **OpenRouter / 其他兼容层**: 读取 `prompt_tokens_details.cached_tokens` / `cache_write_tokens`。
- `written` 代表本次创建的 cache 页数；`cached` 代表命中 cache 的 token 数。

---

## 8. Qwen Portal 的特殊处理

针对 Qwen Portal（`portal.qwen.ai`）有两处独立的 cache_control 注入，不经过 `apply_anthropic_cache_control`：

```python
# run_agent.py:6028-6034 (deepcopy 路径)
for msg in prepared:
    if isinstance(msg, dict) and msg.get("role") == "system":
        content = msg.get("content")
        if isinstance(content, list) and content and isinstance(content[-1], dict):
            content[-1]["cache_control"] = {"type": "ephemeral"}
        break
```

特点：
- 只给 **system message 的最后一个 content part** 打上 `cache_control`。
- 不处理 tool/user/assistant 消息，不采用 `system_and_3` 策略。
- 这是因为 Qwen Portal 的 cache 语义与 Anthropic 不同，Hermes 采用保守注入。

---

## 9. 完整调用链路图（从 Gateway 到 API）

```
Gateway 收到新消息
    │
    ├──► _agent_cache.get(session_key) ?
    │       ├── 命中 ──► 复用已有 AIAgent 实例
    │       └── 未命中 ──► 新建 AIAgent（load_from_disk → 生成 frozen snapshot）
    │
    ▼
AIAgent.run()
    │
    ├──► _cached_system_prompt 为空？
    │       ├── 继续会话 ──► 从 SQLite session_db 读取旧 system prompt
    │       └── 新会话 ──► _build_system_prompt() → 拼接 7 层（含 frozen snapshot）
    │                      存入 SQLite & _cached_system_prompt
    │
    ├──► 组装 api_messages
    │       ├── _cached_system_prompt（稳定前缀）
    │       ├── ephemeral_system_prompt（API-call 时拼接，不入 cache）
    │       ├── prefill_messages（API-call 时插入）
    │       ├── plugin context（pre_llm_call hooks 产出，不入 cache）
    │       └── conversation history（messages，每轮增长）
    │
    ├──► Preflight token 估算
    │       └── 超过 budget ──► _compress_context()
    │               ├── ContextCompressor.compress()
    │               ├── _invalidate_system_prompt() → rebuild → 新 snapshot
    │               └── 下轮开始享受新 prefix cache（承受一次 write cost）
    │
    ├──► _use_prompt_caching ?
    │       ├── 否 ──► api_messages 原样发送
    │       └── 是 ──► apply_anthropic_cache_control(api_messages)
    │               ├── system message → cache_control on last part
    │               ├── last 3 non-system messages → cache_control
    │               └── OpenRouter: skip role:tool / Native: allow role:tool
    │
    ├──► anthropic_adapter.convert_messages_to_anthropic()
    │       ├── system cache_control → 保留为 list-of-blocks
    │       ├── tool cache_control → 迁移到 tool_result block
    │       └── thinking block → 剥离 cache_control（避免 signature 校验失败）
    │
    └──► Anthropic API / OpenRouter / Qwen Portal
            │
            └── Cache miss → write KV（1.25x cost）
                Cache hit  → read KV（0.1x cost，延迟大幅降低）
```

---

## 10. 设计疑点与注意事项

| # | 疑点 | 说明 |
|---|------|------|
| 1 | `ttl` 字段非标准 | `cache_ttl == "1h"` 时注入的 `"ttl": "1h"` 不是 Anthropic `cache_control` 规范的一部分（规范只要求 `{"type": "ephemeral"}`），可能被 API 忽略。 |
| 2 | System list content 仅标记 last part | 若 system prompt 由多个 block 组成（如 text + image），只有最后一个 block 获得 marker。前面的 block 不会被缓存。 |
| 3 | OpenRouter tool skip 的副作用 | 长工具结果链无法被单独缓存，可能增加输入成本；但这是 provider 兼容性约束下的必要权衡。 |
| 4 | Compression 后的 write cost | L4 压缩导致 system prompt 重建，触发一次 cache creation（1.25x），但压缩是低频事件，长期节省仍远大于此代价。 |

---

## 11. 结论

Hermes 的 Prefix Cache / Prompt Caching 设计是一个**以空间换时间**的经典工程优化，其有效性建立在一个前提上：**system prompt 前缀在整段会话中逐字节不变**。

为此，Hermes 构建了三重保障：
1. **MemoryStore 的 `_system_prompt_snapshot`** —— 把易变的 memory/user profile 在 session start 时冻结，后续写入只影响 disk 和 live entries。
2. **`AIAgent._cached_system_prompt`** —— 每 session 构建一次，续载时从 DB 读取旧值，绝不重新读取 disk memory。
3. **Gateway 的 `_agent_cache`** —— 避免 per-message 重建 AIAgent 导致的 prefix 漂移。

在此基础上，`agent/prompt_caching.py` 以纯函数方式实现 `system_and_3` 策略：
- System prompt 占 1 个 breakpoint，享受最稳定的前缀缓存收益。
- 最近 3 条非 system 消息占剩余 breakpoint，覆盖 conversation history 的滑动窗口。
- 通过 `native_anthropic` 参数区分 Anthropic Native 与 OpenRouter 的 tool role 处理，规避 silent hang。

所有可变内容（ephemeral system prompt、prefill、plugin context）均被严格隔离到 API-call-time 的副本上，绝不污染 `_cached_system_prompt`，从而确保多轮对话的平均输入成本维持在 **~0.1x** 水平（相比无 cache 的 1x）。
