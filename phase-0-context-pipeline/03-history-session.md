# Hermes Context Pipeline 深度拆解 — L3+L5: History Management, Tool Loop & Session Persistence

**文件**: `run_agent.py` (10805 行, 核心逻辑占 ~3000 行)

**分析结论**: run_agent.py 的 message history 不是简单的 append-only，而是一个**充满 guardrails 的状态机**。每一轮循环都要处理 invalid tool、invalid JSON、truncation、thinking prefill、interrupt、compression 等 edge case。Session persistence 采用**双写策略**——SQLite 做消息级增量记录，JSON 做 session 级全量快照，两者互相保护。

---

## 大白话介绍

想象一场多人接力赛：

**Message History 就是接力棒**。不是简单的"把棒子传给下一个人"，而是：

1. 发令枪响（用户发送消息），裁判先在棒子上贴一张外部档案馆的摘要（`<memory-context>`）——但这张贴纸**只给当前轮次的 AI 看**，不会被扫描存档。
2. AI 选手开始跑，可能边跑边喊"把棒子给我递过来"（调用 tool）。裁判记录每一次递棒子的结果（tool result）。
3. 如果棒子越来越长（context 膨胀），裁判把中间那段 speeches 写成便利贴（summary），只保留开头和最近几轮——这就是 **context compression**。
4. 每天比赛结束，裁判做**双备份**：一份详细记录存在账本（SQLite，逐条消息），一份完整档案袋封好（JSON，整个 session）。这样即使账本丢了还有档案袋，档案袋丢了还有账本。

**隐藏机制**: `_apply_persist_user_message_override` 相当于"赛后修改记录"——如果本轮运动中 AI 看到的用户消息其实被偷偷加了料（比如 skill 注入的内容），存档时会用"干净版"替换掉，避免污染历史。

---

## L3: Message History & Tool Loop

### 1. Message List 的生命周期

```
[Session Start]
  messages = [system_prompt]              ← _cached_system_prompt
  
[Turn 0: User Input]
  messages.append(user_msg)               ← original_user_message + skill_injection
  
[API Call]
  api_messages = [system] + messages      ← system 每次前置，前缀缓存
  response = call_llm(api_messages)
  
[Turn 0: Assistant Response]
  if tool_calls:
    messages.append(assistant_msg)        ← 含 tool_calls
    [Execute tools] → messages.append(tool_msg) for each result
    [Next API call with updated messages]
  else:
    messages.append(assistant_msg)        ← 最终回复
    [Break loop]
```

### 2. Tool Result 管道 (line 7100–7140)

```python
# 1. 调用工具
function_result = self.tools[name](**args)

# 2. 持久化大型结果（可能写入磁盘，返回 path）
function_result = maybe_persist_tool_result(
    content=function_result, tool_name=name, tool_use_id=tc.id, env=...)

# 3. Subdirectory hints（某些工具在特定子目录更有效）
subdir_hints = self._subdirectory_hints.check_tool_call(name, args)
if subdir_hints:
    function_result += subdir_hints

# 4. 组装 tool message
tool_msg = {
    "role": "tool",
    "content": function_result,
    "tool_call_id": tc.id,
}
messages.append(tool_msg)

# 5. Turn-level budget enforcement（防止单轮 tool results 过长）
enforce_turn_budget(messages[-num_tools:], env=...)
```

**`maybe_persist_tool_result`** 是关键阀门：当 tool result 超过 TOKEN_BUDGET（默认 2K tokens）时，将内容写入 `~/.hermes/tool_results/` 下的文件，返回 `"[Content persisted to: /path/to/file]"`。这防止了超长 terminal 输出或 file read 结果直接撑爆 context。

### 3. Housekeeping Tools & Response Muting (line 9940–9952)

```python
_HOUSEKEEPING_TOOLS = frozenset({"memory", "todo", "skill_manage", "session_search"})
_all_housekeeping = all(tc.function.name in _HOUSEKEEPING_TOOLS for tc in tool_calls)
if _all_housekeeping and self._has_stream_consumers():
    self._mute_post_response = True
```

当 assistant 在同一轮中既输出了 content（如"好的，我记住了"）又调用了 housekeeping tools（如 `memory`），且这些 tools 是**唯一的**工具调用时，系统会**mute 后续输出**。这样用户不会因为"记住了"这种废话被打扰，而这些 housekeeping 操作在后台静默完成。

### 4. Invalid Tool / Invalid JSON 恢复机制

#### 4.1 Invalid Tool Name (line 9810–9822)

```python
if tc.function.name not in self.valid_tool_names:
    messages.append(assistant_msg)
    for tc in assistant_message.tool_calls:
        messages.append({
            "role": "tool",
            "tool_call_id": tc.id,
            "content": f"Tool '{tc.function.name}' does not exist. Available tools: ...",
        })
    continue  # 直接 retry 下一轮
```

#### 4.2 Invalid JSON Arguments (line 9827–9913)

```python
# 尝试 json.loads 每个 tool call 的 arguments
for tc in assistant_message.tool_calls:
    try:
        json.loads(tc.function.arguments)
    except json.JSONDecodeError:
        invalid_json_args.append(...)

if invalid_json_args:
    # 检测是否是 truncation（args 不以 } 或 ] 结尾）
    if truncated:
        return partial response  # 直接结束，不 retry
    
    # Retry 最多 3 次
    if retries < 3:
        continue  # 不 append 任何消息，直接 retry API call
    else:
        # 注入 recovery tool results
        messages.append(assistant_msg_with_broken_tool_calls)
        for tc in tool_calls:
            messages.append({"role": "tool", "tool_call_id": tc.id, 
                             "content": "Error: Invalid JSON arguments..."})
        continue
```

**关键设计**: 用 tool error results（而非 user message）来反馈 JSON 错误，保持 role alternation 的完整性。

### 5. Thinking Prefill 处理 (line 9954–9974)

```python
while messages and messages[-1].get("_thinking_prefill"):
    messages.pop()
    _had_prefill = True

if _had_prefill:
    self._thinking_prefill_retries = 0  # reset counters
    self._empty_content_retries = 0
```

当模型输出 thinking blocks（如 `<thinking>...</thinking>`）后接 tool calls，系统会在 append assistant_msg 前**pop 掉 thinking prefill 辅助消息**。这些辅助消息带有 `_thinking_prefill=True` 标记，是 recovery mechanism 的一部分——当模型输出空 content 时，系统会注入 thinking prompt 诱导模型恢复。

### 6. Compression 触发与 Token 计算 (line 10013–10068)

```python
_compressor = self.context_compressor
if _compressor.last_prompt_tokens > 0:
    _real_tokens = _compressor.last_prompt_tokens + assistant_message.usage.completion_tokens
else:
    _real_tokens = estimate_messages_tokens_rough(messages)

if _compressor.should_compress(_real_tokens):
    self._safe_print("  ⟳ compacting context...")
    messages, active_system_prompt = self._compress_context(
        messages, system_message, approx_tokens=_compressor.last_prompt_tokens)
    conversation_history = None  # 新 session，重置 history
```

**Token 计算逻辑**: 
- `last_prompt_tokens`: 上一次 API call 返回的 prompt_tokens
- `completion_tokens`: 当前 assistant response 的 tokens
- 两者之和 ≈ 下一次 API call 的 prompt size（因为 tool results 还没被计入，但这是 lower bound）

---

## L5: Session Persistence

### 1. 双写架构

| 存储 | 格式 | 写入策略 | 用途 |
|-----|------|---------|------|
| SQLite (`~/.hermes/sessions.db`) | 结构化 | 增量 (`_last_flushed_db_idx`) | 会话检索、session_search、resume |
| JSON (`~/.hermes/sessions/<id>.json`) | 原始消息 | 全量覆盖（防 smaller overwrite） | 调试、导出、人工检查 |

### 2. SQLite 增量写入 (`_flush_messages_to_session_db`, line 2277–2316)

```python
def _flush_messages_to_session_db(self, messages, conversation_history=None):
    start_idx = len(conversation_history) if conversation_history else 0
    flush_from = max(start_idx, self._last_flushed_db_idx)
    for msg in messages[flush_from:]:
        self._session_db.append_message(session_id=..., role=..., content=...,
                                         tool_name=..., tool_calls=...,
                                         tool_call_id=..., finish_reason=...)
    self._last_flushed_db_idx = len(messages)
```

**防重复写入**: `_last_flushed_db_idx` 跟踪上次 flush 的位置，确保同一消息不会被多次写入（修复 bug #860）。

**`conversation_history` 参数**: 这是 resume 场景下的原始历史消息，flush 时会跳过这些已经存在于 DB 中的消息。

### 3. JSON 全量快照 (`_save_session_log`, line 2807–2871)

```python
def _save_session_log(self, messages=None):
    # Guard: never overwrite larger log with fewer messages
    if self.session_log_file.exists():
        existing = json.loads(self.session_log_file.read_text())
        if existing.get("message_count", 0) > len(cleaned):
            return  # 保护 resume 场景下Agent从 partial history 启动时不会覆盖完整日志
    
    atomic_json_write(self.session_log_file, entry, indent=2)
```

**防覆盖保护**: 如果现有 JSON 的 message_count 大于当前，跳过写入。这保护了 `--resume` 场景——如果 resume 加载的 session 没有完整 SQLite 数据，agent 从 partial history 启动时不会 clobber 完整的 JSON log。

### 4. Compression 导致的 Session 切换 (line 6762–6770)

```python
if self._session_db:
    old_title = self._session_db.get_session_title(self.session_id)
    self._session_db.end_session(self.session_id, "compression")
    old_session_id = self.session_id
    self.session_id = f"{datetime.now().strftime('%Y%m%d_%H%M%S')}_{uuid.uuid4().hex[:6]}"
```

**Session 切换 implications**:
- 旧 session 被标记为 `ended_reason="compression"`
- 新 session 获得新的 `session_id` 和空的 DB 记录
- `conversation_history = None` 确保后续 flush 写入新 session
- **但 JSON log 文件也跟着变了**——compression 后 JSON log 切换到新文件，旧文件的完整记录仍保留

### 5. `_apply_persist_user_message_override` (line 2286)

在 flush 到 DB 前，这个方法可能修改 messages。需要进一步确认其功能——可能与用户隐私设置有关，如将用户消息替换为匿名内容。

---

## 6. 关键假设验证汇总

| # | 假设 | 验证状态 | 结论 |
|---|------|---------|------|
| 1 | System prompt memory block 是 frozen snapshot | ✅ | L1 docstring + MemoryStore.format_for_system_prompt 证实 |
| 2 | System prompt 在 compression 后 rebuild | ✅ | `_compress_context` 中调 `_invalidate_system_prompt()` + `_build_system_prompt()` |
| 3 | `<memory-context>` 注入到 user/assistant message | ✅ **已确认** | `run_agent.py:8075-8086` 注入到 `current_turn_user_idx` 的 user message content 末尾，API-only；非独立 message append |
| 4 | `on_pre_compress` 返回值未被使用 | ❌ **已确认为 Bug** | `run_agent.py:6748` 调完丢弃，critical memory 在压缩时丢失 |
| 5 | Tool result budget 通过 `maybe_persist_tool_result` 控制 | ✅ | >2K tokens 时写入文件 |
| 6 | Session 切换导致 `_previous_summary` 丢失 | ✅ | compressor 的迭代 summary 在内存中，session 切换后丢失 |

---

*文档范围: L3 History Management + L5 Session Persistence*  
*上下游: 上游驱动 L4 Compressor 触发; 下游被 user/gateway 消费*
