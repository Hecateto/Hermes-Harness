# Hermes Context Pipeline 深度拆解 — L4: Context Compressor

**文件**: `agent/context_compressor.py` (820 行)

**分析结论**: Context Compressor 是一个**四层渐进式压缩引擎**，按成本从低到高依次为：(1) tool result 裁剪、(2) token-based tail 保护、(3) structured LLM summary、(4) tool-call/result 配对修复。迭代式 summary 更新、focus topic 支持、以及 robust 的 role alternation 处理使它远超 simple truncation baseline。820 行中超过 40% 用于处理 edge case，这是 long-context 工程的真实面貌。

---

## 大白话介绍

想象你的书桌上堆满了便签纸（对话历史），已经堆到快倒了（接近 context limit）。你不能直接把所有纸扔掉，因为上面可能记着重要电话号码。Hermes 的处理方式是**四层渐进清理**：

1. **第一层：先扔复印件**（Prune Old Tool Results）。那些"终端输出"、"文件搜索结果"最占地方，且不重要——把旧的换成一张纸条"这部分内容在 summary 里有"，空间立刻省出大半，**零成本**。
2. **第二层：保护最新便签**（Token-Based Tail Protection）。从最新一张开始数，数到 token 预算就停，这些保留不动。不数"张数"而数"字量"，因为有的便签只有几个字，有的却是一本小册子。
3. **第三层：请人写简短摘要**（LLM Summary）。中间那些既没保护又太长的，花钱请 LLM 写个交接文档。格式有讲究：已完成什么、卡在哪、关键决定、待办事项……像项目周报。**而且第二次压缩是在第一次摘要基础上增量更新**，不是从头重写，避免信息逐次衰减。
4. **第四层：检查工具便签配对**（Tool Pair Sanitization）。AI 有时候叫了一个工具（assistant tool_call），但对应的返回结果（tool result）可能被 summary 吞掉了。API 会报错。所以必须补上 stub："这个结果在 summary 里"。

**精华**: 820 行代码里 40% 在处理 edge case——这不是过度工程，而是 long-context 系统的真实复杂度。每次压缩后还会重新发令（`_invalidate_system_prompt`），让 Prompt Builder 重新组装 system prompt。

---

## 1. 初始化参数与 Token 预算 (line 103–170)

### 1.1 默认参数

| 参数 | 默认值 | 含义 |
|-----|-------|------|
| `threshold_percent` | 0.50 | context 达到多少比例触发压缩 |
| `protect_first_n` | 3 | 保护最前面 N 条消息 |
| `protect_last_n` | 20 | 向后兼容：保护最后 N 条消息 |
| `summary_target_ratio` | 0.20 | summary 占据 threshold 的多少比例 |
| `summary_model_override` | None | 可指定不同模型做 summary (如用 cheap model) |

### 1.2 Token 预算计算

```python
self.threshold_tokens = max(
    int(self.context_length * threshold_percent),  # e.g. 50% of 128K = 64K
    MINIMUM_CONTEXT_LENGTH,                         # floor: 8K tokens
)
```

**设计意图**: `MINIMUM_CONTEXT_LENGTH` (8K tokens) 防止大模型context_window很大时过早压缩——如果模型支持 1M tokens，50% 就是 500K，但没必要等待那么久才开始压缩。

```python
self.tail_token_budget = self.threshold_tokens * summary_target_ratio  # e.g. 64K * 0.2 = 12.8K
self.max_summary_tokens = min(self.context_length * 0.05, _SUMMARY_TOKENS_CEILING)  # 5% or 8K
```

| 预算 | 计算方式 | 说明 |
|-----|---------|------|
| `threshold_tokens` | max(50% ctx, 8K) | 触发压缩的门槛 |
| `tail_token_budget` | threshold * 20% | 压缩后保留的尾部大小 |
| `max_summary_tokens` | min(5% ctx, 8K ceiling) | summary 模型的输出上限 |
| `summary_budget` (动态) | content_tokens * 0.3 | 按被压缩内容量动态调整 |

> **关键洞察**: summary 的 token 预算**与被压缩内容的 token 数成正比**（`_SUMMARY_RATIO = 0.3`），而非固定值。这意味着压缩 100K tokens 的内容会得到比压缩 10K 更长的 summary，保持信息密度相对稳定。

---

## 2. 四层压缩引擎

### 2.1 Layer 1: Prune Old Tool Results (line 186–241)

```python
def _prune_old_tool_results(self, messages, protect_tail_count, protect_tail_tokens=None):
    # 1. 确定 prune boundary：token budget 优先，msg count 作为硬下限
    # 2. 对 boundary 之前的 tool 消息，如果 content > 200 chars，替换为占位符
```

```
原始: {"role": "tool", "content": "[2000 lines of terminal output...]"}
裁剪后: {"role": "tool", "content": "[Older tool result — context available in compaction summary]"}
```

**为什么先裁 tool results？**
- Terminal read_file、search_files、execute_code 的输出经常很长
- tool result 是 content 的"payload"，其语义价值通常低于对话消息
- 零成本（无 LLM 调用），先做可立即释放大量 tokens

### 2.2 Layer 2: Token-Based Tail Protection (line 604–660)

```python
def _find_tail_cut_by_tokens(self, messages, head_end, token_budget=None):
    # 从尾部向前 walk，累积 tokens，直到达到 budget
    soft_ceiling = token_budget * 1.5  # 允许超支 50% 避免在 oversized message 中间切断
    min_tail = 3  # 硬下限：至少保留 3 条消息
```

**算法**:
1. 从末尾倒序遍历，估算每条消息的 tokens（`len(content) // 4 + 10` + tool args)
2. 累积 tokens 超过 `soft_ceiling` 时停止，但保证至少保留 `min_tail` 条
3. 如果累积完所有消息仍没超支，强制在 head_end+1 处切断（至少压缩一些东西）
4. 边界对齐：调用 `_align_boundary_backward()` 避免切在 tool group 中间

**为什么 token-based 优于 message-count-based？**
- 20 条 short messages = ~2000 tokens ✓
- 20 条 tool results with file contents = ~20000 tokens ✗
- token budget 自动适应内容密度差异

### 2.3 Layer 3: Structured LLM Summary (line 318–483)

#### 3.3.1 Summary Model 选择

```python
call_kwargs = {
    "task": "compression",
    "main_runtime": {...},  # 继承主模型的 provider/url/key
    "max_tokens": summary_budget * 2,  # 给 summarizer 更多空间
}
if self.summary_model:
    call_kwargs["model"] = self.summary_model  # 可用 cheaper/faster model
```

可使用 summary model override 做**模型级联**：主模型跑推理，cheap model（如 4o-mini）做 summarization，节省成本。

#### 3.3.2 迭代式更新 (line 406–420)

```python
if self._previous_summary:
    # 用 previous summary + 新增 turns 做增量更新
    prompt = f"""更新之前的 summary，保留相关信息，添加新进展..."""
else:
    # 首次压缩：从头总结
    prompt = f"""创建结构化交接 summary..."""
```

**迭代更新的价值**: 首次压缩可能压缩 50 turns → 1K token summary。第二次压缩如果从头再来，会丢失第一次 summary 中的信息。迭代更新保 iterative accumulation，防止信息衰减。

#### 3.3.3 Structured Handoff Template (line 367–404)

```
## Goal
## Constraints & Preferences
## Progress
  ### Done
  ### In Progress
  ### Blocked
## Key Decisions
## Resolved Questions
## Pending User Asks
## Relevant Files
## Remaining Work
## Critical Context
## Tools & Patterns
```

**设计来源**: 融合 Claude Code `/compact` 和 OpenCode 的交接格式。

- **"Pending User Asks"** vs **"Resolved Questions"**: 显式区分已回答 vs 未回答问题，防止 agent 重复回答用户已经问过并得到答案的问题
- **"Remaining Work"** (not "Next Steps"): 用被动语态避免 agent 把 summary 误读为指令列表
- **Target ~{budget} tokens**: 在 prompt 中注入动态预算，引导 summarizer 控制长度

#### 3.3.4 Focus Topic 支持 (line 436–440)

```python
if focus_topic:
    prompt += f"""FOCUS TOPIC: "{focus_topic}"
    PRIORITISE preserving all information related to the focus topic..."""
```

用户可以通过 `/compress <topic>` 指定压缩时的保留重点。这在多任务 session 中非常有用——比如用户说"/compress 只保留 deployment 相关内容"，其余可以更激进地压缩。

#### 3.3.5 失败降级 (line 468–483)

```python
except RuntimeError:
    self._summary_failure_cooldown_until = now + _SUMMARY_FAILURE_COOLDOWN_SECONDS
    return None
```

- Summary 失败时进入 cooldown（默认 60s），防止在 provider 不可用时反复重试
- 返回 None 后，调用方会插入静态 fallback marker（line 756–766），而非静默丢弃

### 2.4 Layer 4: Tool Pair Sanitization (line 506–564)

#### 3.4.1 两种 Orphan 场景

**场景 A**: tool result 存在，但对应的 assistant tool_call 被 summary 掉了
→ API 报错: "No tool call found for function call output with call_id ..."

**场景 B**: assistant 有 tool_calls，但 tool results 被 summary 掉了
→ API 报错: "Every tool_call must be followed by a tool result"

#### 3.4.2 修复策略

```python
def _sanitize_tool_pairs(self, messages):
    # 1. 收集所有 surviving assistant tool_call IDs
    surviving_call_ids = {tc.id for tc in assistant.tool_calls}
    
    # 2. 收集所有 surviving tool result IDs  
    result_call_ids = {msg.tool_call_id for msg in tool_messages}
    
    # 3. Remove orphaned results: result_call_ids - surviving_call_ids
    messages = [m for m in messages if not (orphan)]
    
    # 4. Add stub results for orphaned calls: surviving_call_ids - result_call_ids
    for each missing result:
        insert {"role": "tool", "content": "[Result from earlier conversation...]", "tool_call_id": cid}
```

**为什么必须保留 stub 而非删除 orphaned calls？**
- API 要求 assistant 的 tool_calls 必须有对应的 tool results
- 删除 tool_calls 会改变 assistant message 的结构，可能导致后续解析问题
- Stub content 提示模型："这个结果在 summary 里"

---

## 3. Role Alternation 的精细处理 (line 768–803)

LLM API 通常要求 messages 严格遵循 user/assistant/user/assistant 交替。Summary message 插入到 head 和 tail 之间时，可能打破这个规则。

```python
last_head_role = messages[compress_start - 1].get("role", "user")
first_tail_role = messages[compress_end].get("role", "user")

# Priority: avoid colliding with head (already committed), then tail
if last_head_role in ("assistant", "tool"):
    summary_role = "user"
else:
    summary_role = "assistant"

# 如果与 tail 冲突且翻转不与 head 冲突，翻转
if summary_role == first_tail_role:
    flipped = "assistant" if summary_role == "user" else "user"
    if flipped != last_head_role:
        summary_role = flipped
    else:
        # 两败俱伤：head=assistant, tail=user → user 和 assistant 都不行
        # 合并 summary 到第一条 tail message
        _merge_summary_into_tail = True
```

**Merge into tail 的场景**:
```
Head ends with: assistant (with tool_calls)
Tail starts with: assistant

→ 如果 insert user: user | assistant ✓... but wait, tail is assistant
→ 如果 insert assistant: assistant | assistant ✗
→ 所以 merge summary into the first tail assistant message
```

> **设计细节**: `_merge_summary_into_tail` 这个分支似乎很少被触发（需要 head 和 tail 都是 assistant），但代码明确处理了它，说明曾经遇到过这个问题。

---

## 4. Summary 内容的前缀处理 (line 485–493)

```python
SUMMARY_PREFIX = "[CONTEXT COMPACTION — REFERENCE ONLY]"
LEGACY_SUMMARY_PREFIX = "[Context Compaction - Referenced Only]"

def _with_summary_prefix(self, summary: str) -> str:
    text = summary.strip()
    for prefix in (LEGACY_SUMMARY_PREFIX, SUMMARY_PREFIX):
        if text.startswith(prefix):
            text = text[len(prefix):].lstrip()
            break
    return f"{SUMMARY_PREFIX}\\n{text}"
```

**为什么要 strip 再重新加 prefix？**
- 迭代更新时，summary 可能自带旧 prefix → 去重
- 标准化格式，无论来源如何最终输出一致

**为什么 prefix 要这么长？**
- 显式标注"REFERENCE ONLY"，告诫 agent 不要把这个当成新的用户指令去执行
- 与 system note (line 748–751) 形成双重提示

---

## 5. 压缩触发条件总览

```
should_compress(prompt_tokens):
    return prompt_tokens >= self.threshold_tokens

compress() 入口检查:
    n_messages <= protect_first_n + 3 + 1 → 不压缩 (消息太少)
    compress_start >= compress_end → 不压缩 (head 覆盖到了 tail)
```

### 典型场景计算 (假设 4o, 128K context)

| 场景 | threshold | tail_budget | max_summary |
|-----|-----------|-------------|-------------|
| Default | max(64K, 8K) = **64K** | 64K * 20% = **12.8K** | min(6.4K, 8K) = **6.4K** |
| Small model (32K) | max(16K, 8K) = **16K** | 16K * 20% = **3.2K** | min(1.6K, 8K) = **1.6K** |
| Large model (1M) | max(500K, 8K) = **500K** | 500K * 20% = **100K** | min(50K, 8K) = **8K** |

**异常**: 1M context model 的 summary 被 hard cap 在 8K，因为 `_SUMMARY_TOKENS_CEILING = 8192`。这是为了防止 summarizer 的输出本身过长。

---

## 6. 与 MemoryManager 的钩子交互

在 L2 中我们发现 MemoryManager 有 `on_pre_compress(messages)` 钩子。那么 compressor 是否调用了它？

搜索 `on_pre_compress` 在 compressor 中的使用:

需要在 run_agent.py 中确认 compressor.compress() 的调用方是否先调用了 `memory_manager.on_pre_compress()`。

**假设**: run_agent.py 的压缩流程大致如下:
1. `memory_manager.on_pre_compress(messages)` → 获取需要保留的 memory context
2. 将此 text 注入到 summary prompt 中，或加入 protected head
3. 调用 `compressor.compress(messages)`

如果此假设不成立，则 critical memory（如"绝不要删除 .env"）可能在压缩中丢失。

---

## 7. 设计选择的得与失

### 做得好的

1. **四层渐进压缩**: 从最cheap到最expensive，避免不必要的LLM调用
2. **迭代式summary**: 防止信息在多次压缩中衰减
3. **token-based tail保护**: 比固定消息数更适应不同内容密度
4. **tool pair sanitization**: 正确处理API约束，避免格式错误
5. **focus topic**: 用户可控的压缩策略
6. **失败降级**: 不静默丢失，而是插入明确的fallback marker

### 潜在问题

1. **Char-based token估算**: `_CHARS_PER_TOKEN = 4` 是 rough heuristic。CJK 文本 token 密度可能是 1:1.5，英文代码是 1:4。估算偏差可能导致过早/过晚压缩。
2. **Summary model 默认继承主模型**: 如果没有 override，用同一个 expensive model 做 summarization 浪费成本
3. **迭代更新可能累积错误**: 如果某次 summary 有幻觉，后续迭代会在此基础上继续累积
4. **`_merge_summary_into_tail` 场景未完全测试**: 这个 edge case 的交互复杂，stub tool results + merged summary 可能产生奇怪的模型行为

---

## 8. 待验证假设

| # | 假设 | 验证状态 |
|---|------|---------|
| 1 | MemoryManager.on_pre_compress 在压缩前被调用 | ✅ **已确认** | `run_agent.py:6746-6750` 在 `compressor.compress()` 之前调用，但返回值被丢弃 |
| 2 | Compression count 影响 retry/backoff 策略 | ❓ 仍为未验证 | 未读 gateway/run.py，保持开放 |
| 3 | Summary 的 `_previous_summary` 在 session 重启后持久化 | ❌ 不持久化 → in-memory only, session 重启丢失 |
| 4 | `call_llm(task="compression")` 使用与主模型相同的 API key/URL | ✅ **已确认** | 传入 `main_runtime` 字典（含 provider/url/key），若未配置 `summary_model_override` 则使用主模型 |

---

*文档生成时间: 基于 hermes-agent v0.9.0 源码*  
*分析范围: L4 Context Compressor*  
*上下游依赖: 上游被 L1 Prompt Builder 的 _estimate_messages_tokens 驱动触发; 下游影响 L5 Session 的 message 持久化*
