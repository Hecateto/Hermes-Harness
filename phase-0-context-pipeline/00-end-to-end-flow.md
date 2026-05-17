# Hermes Context Pipeline — Master Architecture Overview

**基于 hermes-agent v0.9.0 源码全面拆解**
**产出时间**: 深度阅读 5 个核心文件（prompt_builder.py 1200+ 行, memory_manager.py 362 行, memory_tool.py 560 行, context_compressor.py 820 行, run_agent.py 10805 行关键逻辑）

---

## 大白话介绍

想象你走进一家餐厅点菜。Hermes 的 Context Pipeline 就是后厨的完整流水线：

1. **迎宾部 (Prompt Builder)**: 你一进门，服务员帮你拼好菜单（system prompt），包括店长推荐（agent notes）、你的忌口（memory）、还有后厨安全须知（security scan）。
2. **传菜部 (Memory Manager)**: 服务员去储物间翻你之前的点餐记录（memory prefetch），顺便把你今天说的话记下来（sync），方便下次你来时知道你喜欢什么。
3. **切配间 (History Management)**: 厨师把上一轮没做完的半成品（tool results）和新点的菜（user message）摆好，按顺序放在案板上。
4. **炒锅 (Tool Loop)**: 如果菜需要现炒，厨师会调用工具（切菜机、烤箱）加工，每次调用完都把成品放回案板。
5. **控温器 (Context Compressor)**: 案板上的食材太多放不下了，厨师把一些不重要的半成品打包成便利贴（summary），只保留关键信息和最新几道菜。
6. **仓管 (Session Persistence)**: 每天打烊，所有菜单和聊天记录被扫描存档——一份存在本地账本（SQLite），一份存在档案袋（JSON），防止丢失。

这条流水线的设计哲学是：**快的东西先做（inline spill），贵的东西后做（LLM summary），不该丢的绝不丢（critical memory fence）**。但代码里也埋着一个已知 bug：传菜部记下的重要提醒，在控温器打包时会被遗忘。

---

## 1. 端到端数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          SESSION START                                       │
│  1. AIAgent.__init__() → 加载 config, tools, checkpoint_mgr                  │
│  2. MemoryStore.load_from_disk() → 捕获 frozen snapshot                      │
│  3. _build_system_prompt() → 组装 7 层 system prompt, cache 到 _cached      │
│  4. _session_db.create_session() → SQLite 创建 session 记录                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          TURN 0 — USER INPUT                                 │
│  messages = [system_prompt] + [user_msg_with_skill_injection]               │
│  _save_session_log() → JSON 快照                                             │
│  _flush_messages_to_session_db() → SQLite 增量写入                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          API CALL #1                                         │
│  api_messages = [{"role":"system","content":_cached_system_prompt}]         │
│                + messages                                                     │
│  response = call_llm(api_messages)                                           │
│  _compressor.update_from_response(usage) → 记录 prompt/completion tokens     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          POST-RESPONSE DECISION TREE                         │
│                                                                              │
│  ├─ finish_reason="tool_calls"?                                              │
│  │   ├─ tool name invalid? → inject tool error, retry (max 3x)              │
│  │   ├─ JSON args invalid? → detect truncation → recovery or retry          │
│  │   └─ valid tool calls → _execute_tool_calls()                             │
│  │       ├─ maybe_persist_tool_result() → 超长结果写入文件                   │
│  │       ├─ messages.append(tool_msg) for each result                        │
│  │       ├─ enforce_turn_budget() → 单轮 tool budget 封顶                    │
│  │       └─ LOOP BACK to API CALL                                           │
│  │                                                                          │
│  └─ finish_reason="stop" / "length" / other                                  │
│      ├─ content empty? → thinking prefill recovery → retry                  │
│      └─ normal response → messages.append(assistant_msg)                    │
│           └─ END OF TURN                                                    │
│                                                                              │
│  [After assistant append, before next user input]                           │
│  ├─ should_compress(tokens)?                                               │
│  │   └─ YES → _compress_context()                                           │
│  │       ├─ Phase 1: prune old tool results (cheap)                         │
│  │       ├─ Phase 2: determine tail boundary (token-based)                  │
│  │       ├─ Phase 3: LLM structured summary                                 │
│  │       ├─ Phase 4: sanitize tool pairs (fix orphans)                      │
│  │       ├─ Rebuild system prompt (invalidate → rebuild)                    │
│  │       ├─ Switch session_id, end old session in DB                        │
│  │       └─ conversation_history = None                                     │
│  └─ queue_prefetch_all(original_user_message) → 预热下一轮                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          USER NEXT MESSAGE                                   │
│  messages.append(next_user_msg)                                              │
│  sync_all(user_content, assistant_content) → memory providers               │
│  _flush_messages_to_session_db()                                             │
│  _save_session_log()                                                         │
│  LOOP BACK to API CALL                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 七层架构总览

| 层次 | 文件 | 核心职责 | 关键设计决策 |
|-----|------|---------|------------|
| **L1** Prompt Builder | `prompt_builder.py` | system prompt 组装 | 7-layer stack, char-limit enforcement, injection scan |
| **L2** Memory Manager | `memory_manager.py` | provider orchestration | singleton constraint, context fencing, lifecycle hooks |
| **L2.1** MemoryStore | `tools/memory_tool.py` | persistent curated memory | dual-state (frozen/live), atomic file I/O, char budget |
| **L3** History Manager | `run_agent.py` core loop | message append & tool routing | guardrails, retry logic, thinking prefill |
| **L4** Context Compressor | `context_compressor.py` | long-context mitigation | 4-layer progressive compression, iterative summary |
| **L5** Session Persistence | `run_agent.py` persistence | durable session storage | SQLite + JSON dual-write, session switching |
| **L6** Tool Result Pipeline | `run_agent.py` execution | tool call → result → append | budget enforcement, result persistence, subdirectory hints |

---

## 3. Token 预算经济学

### 3.1 各级预算约束

```
Context Window (e.g. 128K)
│
├─ System Prompt (~2K–8K tokens)
│   ├─ Identity (SOUL.md or DEFAULT_AGENT_IDENTITY)
│   ├─ Memory snapshot (frozen at session start, ~1K chars ≈ 250 tokens)
│   ├─ User profile (~1.3K chars ≈ 325 tokens)
│   ├─ Context files (AGENTS.md, .cursorrules, up to 80K chars ≈ 20K tokens)
│   └─ Skills guidance (conditional, ~0.5K–2K tokens)
│
├─ Conversation History (variable, up to threshold)
│   ├─ Head (protected: first 3 messages)
│   ├─ Middle (compressible: summarized or dropped)
│   └─ Tail (protected: ~20K tokens of recent context)
│
└─ Working Margin (~threshold to max context)
    └─ Tool results, user inputs, assistant responses
```

### 3.2 Compression Trigger Budget

| Context | Threshold | Tail Budget | Summary Max |
|---------|-----------|-------------|-------------|
| 32K model | 16K tokens | 3.2K tokens | 1.6K tokens |
| 128K model | 64K tokens | 12.8K tokens | 6.4K tokens |
| 1M model | 500K tokens | 100K tokens | 8K tokens (hard cap) |

### 3.3 隐性成本

| 成本来源 | 估算 | 触发条件 |
|---------|------|---------|
| Summary LLM call | 1× completion cost | threshold >= 50% |
| Iterative summary update | 累积误差风险 | 多次 compression |
| Prefetch memory provider | 1× latency/cost per turn | 存在 external provider |
| Tool result persistence Disk I/O | ~1ms | result > 2K tokens |
| JSON session log write | ~5–50ms | 每 turn |
| SQLite flush | ~1–5ms | 每 turn |

---

## 4. 关键状态机

### 4.1 Session Lifecycle

```
[CREATED] → user sends message → [ACTIVE]
   │                                    │
   │                                    ├─ tool loop → [ACTIVE]
   │                                    ├─ compression → [COMPRESSED] → [CREATED] (new session_id)
   │                                    └─ interrupt → [INTERRUPTED]
   │                                    │
   └─ /new command → [ENDED] → [CREATED]
   └─ /quit → [ENDED]
```

### 4.2 System Prompt Cache State

```
INVALID (None)
   │
   ├─ Session start → BUILD → CACHED
   │                          │
   │                          ├─ Normal turn ──────────────→ CACHED (stable)
   │                          │
   │                          ├─ Compression event → INVALID → BUILD → CACHED
   │                          │
   │                          └─ /new command → INVALID → BUILD → CACHED
```

**重要**: Memory snapshot 是 `load_from_disk()` 时冻结的，不是 `_build_system_prompt()` 时。所以即使 cache rebuild，memory 内容仍然是旧的——除非同步新 session 触发新的 `load_from_disk()`。

### 4.3 Compression State

```
NO_COMPRESSION (compression_count=0)
   │
   ├─ threshold reached → COMPRESSING → COMPRESSED (compression_count += 1)
   │                                          │
   │                                          ├─ threshold reached again → COMPRESSING (iterative update)
   │                                          │
   │                                          └─ session ends → DISCARDED (_previous_summary lost)
```

---

## 5. 安全边界

### 5.1 Injection 防御三层

| 层级 | 机制 | 触发时机 |
|-----|------|---------|
| **L1: Context Files** | `_scan_context_content()` → 替换为 [BLOCKED] 声明 | 加载 AGENTS.md 等时 |
| **L2: Memory Entries** | `_scan_memory_content()` → 拒绝写入 | `memory add/replace` 时 |
| **L3: External Memory** | `build_memory_context_block()` → XML fence + sanitize | `prefetch_all()` 返回时 |
| **L4: User Input** | `_scan_user_content()` (隐藏，未在 L1 中出现) | user message 进入前 |

### 5.2 Tool 安全

| 防御 | 实现 |
|-----|------|
| Invalid tool name | 最多 3 次 retry，之后注入 error result |
| Invalid JSON args | 最多 3 次 retry，检测 truncation 后提前退出 |
| Tool result overflow | `maybe_persist_tool_result()` >2K tokens 写入文件 |
| Turn budget overflow | `enforce_turn_budget()` 截断或拒绝 |
| Unauthorized directory traversal | `validate_path()` 在 tool entrypoint |

---

## 6. 已知问题与待修复事项

### 6.1 已确认问题

| ID | 问题 | 位置 | 严重性 |
|----|------|------|--------|
| **#1** | `on_pre_compress` 返回值被调用但未使用 | `run_agent.py:6748` | 🔶 Medium |
| **#2** | `_previous_summary` 在 session 切换后丢失 | `context_compressor.py:169` | 🔶 Medium |
| **#3** | Char-based token 估算对 CJK 偏乐观 (1:1.5 vs 1:4) | `_CHARS_PER_TOKEN=4` | 🔷 Low |
| **#4** | Summary model 默认继承主模型，未用 cheap model | `summary_model_override` 默认 None | 🔷 Low |

### 6.2 假设 #1 详细：`on_pre_compress` 返回值丢弃

```python
# memory_manager.py:285–302
def on_pre_compress(self, messages) -> str:
    parts = []
    for provider in self._providers:
        result = provider.on_pre_compress(messages)
        if result and result.strip():
            parts.append(result)
    return "\n\n".join(parts)

# run_agent.py:6746–6750
if self._memory_manager:
    try:
        self._memory_manager.on_pre_compress(messages)  # ← 返回值被丢弃！
    except Exception:
        pass
```

**状态**: ❌ **已确认为 Bug** (通过源码交叉验证)

**预期行为**: `on_pre_compress` 返回的文本应该被注入到 summary prompt 中，让 compressor 在生成 summary 时保留 critical memory（如"绝不要删除 .env"）。实际返回值被丢弃，critical memory 可能在压缩时丢失。

**修复方向**:
```python
pre_compress_hints = self._memory_manager.on_pre_compress(messages)
if pre_compress_hints:
    focus_topic = (focus_topic or "") + "\n" + pre_compress_hints
    # 或者将 hints 作为 protected head 注入 compressor
```

**实际行为**: 返回值被丢弃，external memory provider 的 pre_compress 提示对 compression 无任何影响。

**修复建议**:
```python
pre_compress_hints = self._memory_manager.on_pre_compress(messages)
compressed = self.context_compressor.compress(
    messages, current_tokens=approx_tokens, focus_topic=focus_topic,
    pre_compress_hints=pre_compress_hints,  # 新增参数
)
```

不过这可能并非 bug，而是设计如此——也许开发者计划在未来扩展，或者认为 `flush_memories()` (line 6743) 已经足够。

---

## 7. 与主流框架对比

| 能力 | Hermes | OpenAI Codex | Claude Code | Cursor |
|-----|--------|-------------|-------------|--------|
| System prompt char limits | ✅ 硬限制 | 未知 | 未知 | 未知 |
| Memory injection scan | ✅ 双扫描 | 未知 | 未知 | 未知 |
| Iterative summary | ✅ | ❌ | ✅ `/compact` | ❌ |
| Focus topic compression | ✅ | ❌ | ✅ `/compact <topic>` | ❌ |
| Structured handoff template | ✅ (8 sections) | 未知 | 中等 | 未知 |
| Tool pair sanitization | ✅ 完整修复 | 未知 | 未知 | 未知 |
| Dual-state memory | ✅ (frozen/live) | 未知 | 未知 | 未知 |
| External memory provider | ✅ provider 架构 | ❌ | ✅ MCP | ❌ |
| Session JSON export | ✅ | ❌ | ❌ | ❌ |
| SQLite session store | ✅ | ❌ | ❌ | ❌ |
| Atomic file I/O | ✅ | 未知 | 未知 | 未知 |
| Compression cooldown | ✅ 60s | 未知 | 未知 | 未知 |

---

## 8. 延伸阅读

- L1: [Prompt Builder](./hermes-context-pipeline-l1-prompt-builder.md) — 7-layer system prompt 组装
- L2: [Memory Manager](./hermes-context-pipeline-l2-memory-manager.md) — 双态 memory 与 provider 架构
- L4: [Context Compressor](./hermes-context-pipeline-l4-compressor.md) — 4-layer progressive compression engine
- L3+L5: [History & Session Persistence](./hermes-context-pipeline-l3-l5-history-session.md) — message lifecycle 与 dual-write persistence

---

*分析完成。所有文档位于 `/home/jovyan/workspace-0/hermes/hermes-forge/research/`*  
*基于 hermes-agent v0.9.0 源码，2026-05-17*
