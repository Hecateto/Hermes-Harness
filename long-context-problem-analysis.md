# 长对话记忆丢失问题 — 独立分析与 Hermes Harness 解构

**问题来源**: workspace-0/hermes/hermes-forge/research/long-context-problem.md  
**参考报告**: hermes-memory-harness-research.md (v0.9.0 源码分析)  
**分析性质**: Brainstorm / 溯因分析 / 工程方案设计  
**撰写时间**: 2026-05-17

---

## Part 1 — 独立分析：溯因与问题解构

### 1.1 现象重述

用户在单一会话中持续交互，当对话轮次超出模型上下文窗口时，系统通过 context compression 丢弃中间历史。用户后续提及前文中的细粒度信息（如特定 URL、SQL 查询结果中的某个 ID、配置参数值）时，Agent 已无法回忆。典型案例：

> 第 3 轮：Agent 调用 web_search → 获得论文 URL `https://arxiv.org/abs/2405.04532`  
> 第 5 轮：Agent 调用 read_file → 获得某配置项 `learning_rate: 3e-4`  
> ... 多轮之后 ...  
> 第 25 轮：用户问"把之前 arxiv 那篇论文的 URL 贴出来" → Agent 失忆

### 1.2 五级根因分析

#### Root Cause 1：上下文的物理稀缺性 vs 用户意图的单调增长

LLM 的 context window 是有上界（即使是 2M token 也是有限集合）的，但用户在一个 session 中累积的意图和需求是**单调不减**的。早期 session 里的每一个信息点，理论上都有在未来任意时刻被引用的概率。这是一个**不可调和的矛盾**，不是靠"窗口更大"就能彻底解决的。

> 工程视角：不要试图"装下所有"，要设计"如何快速找到"的系统。

#### Root Cause 2：均匀衰减假设与信息价值非均匀分布的错配

当前几乎所有 context compressor（包括 Hermes v0.9.0）隐含一个假设：**所有历史消息的价值随时间均匀衰减**。因此策略是保护 head（初始身份/任务）和 tail（最近几轮），压缩/丢弃 middle。

但真实的信息价值分布是高度**非均匀**的：
- **高价值-硬事实**：URL、数据库 ID、文件路径、密钥、版本号 — 低 token 数，高信息熵，丢失后 LLM 无法推理生成
- **高价值-用户偏好**：用户说"我喜欢简洁的"、"用中文" — 低 token 数，但对体验至关重要
- **中价值-过程记录**："让我试试 A... 不行... 试试 B" — 可压缩为结论
- **低价值-临时输出**：大段网页原始文本、冗长的 terminal `ls` 输出 — 本就不该长期保留

**均匀压缩把高价值和低价值信息同等对待，导致"沙里淘金时把金子也倒掉"**。

#### Root Cause 3：记忆回写是"手工的"，Agent 缺乏元认知

Hermes 的 `memory` tool 要求 Agent**主动判断**什么值得记忆。但问题在于：
1. **时序盲区**：Agent 在 20 轮后不可能还记得第 3 轮有个 URL 
2. **价值预判盲区**：Agent 不知道自己未来的任务是什么，无法预判哪些信息"以后可能用到"
3. **用户意图盲区**：用户说"这个 URL 很重要"时 Agent 可能没调用 memory，而用户也没义务每句话都加上"请记住"

> 类比：让一个人边工作边决定哪些笔记要归档 — 他通常会在**事后**发现"早该记下来的"。

#### Root Cause 4：Session 压缩是"全有或全无"事件

当 Hermes 的 context 超标时，会触发 `restart_with_compressed_messages`（`run_agent.py` line 7918-7923），创建一个子 session，中间历史变成一段 LLM 生成的 summary。这个 summary 是叙述性的、粗粒度的。

**URL 这类信息天然不适合被 summary 保留** — summary 会说"Agent 之前搜索了一篇关于 xxx 的论文"，但永远不会保留具体的 arxiv ID。

而且这是一个**离散事件**：不是渐进式地淘汰低价值信息，而是一次性把 middle 历史全部蒸馏成一段话。信息损失是阶跃式的，用户感知就是"突然什么都不记得了"。

#### Root Cause 5：Tool Result 的非结构化与高冗余

Agent 看到的 tool result 通常是原始文本：
- `web_extract` 返回一整段网页 HTML/markdown
- `terminal` 返回几十行命令输出
- `read_file` 返回几百行代码

这些信息在**进入 context 时没有结构化**，导致：
- Token 占用巨大，加速了 context overflow
- 压缩时无法识别其中的"关键字段"（URL 埋在第 47 行文本里）
- 即使被保留，LLM 的 attention 在几百行文本中"发现"那个 URL 也很困难

### 1.3 用户行为分析：为什么用户喜欢复用 Session？

这不是用户的"错误习惯"，而是理性选择：

| 用户动机 | 说明 |
|----------|------|
| **上下文连贯感** | 新 session 从零开始，agent 变得"生疏"，需要重新交代背景 |
| **状态连续** | 之前定义过的变量、文件路径、工作目录，新 session 不知道 |
| **仪式感成本** | 开启新 session 需要重新描述任务，用户觉得麻烦 |
| **信任惯性** | 用户潜意识里认为"同一个对话应该记得之前的事" |

> 结论：试图教育用户"请多开新 session"是**逆人性的**。产品应该适配用户行为，而不是反过来。

---

## Part 2 — 解决思路总览（6 个独立方向）

### 方向 1：Entity-Centric Context Management（ECCM）
**核心思想**：不把消息当黑盒，而是在 tool result 进入 context 时提取关键实体，维护一个跨 session 的**结构化实体注册表（Entity Registry）**。

- **Entity Registry** 以 `(key, value, type, confidence, source_turn)` 存储
- 在每次构造 system prompt 时，把高频/高置信度的 entity 作为"Pinned Facts"注入
- 对应用户文档中的 **Pin Slot** 概念

### 方向 2：Tool Result 结构化蒸馏（Distillation）
**核心思想**：Tool result 进入 messages 前，先经过一个轻量级蒸馏步骤，提取结构化摘要。

- **原始全文** → 存入可检索的 session archive，不出现在 LLM context 中
- **结构化摘要** → 作为 JSON/YAML 进入 messages，token 占用极小
- LLM 需要全文时，通过 `session_search` 或新的 `fetch_detail` 按需获取

### 方向 3：预测性自动记忆（Anticipatory Memory）
**核心思想**：不依赖 agent 主动调用 memory，由 harness 在关键事件发生时**自动**评估并回写。

**触发条件**：
- Tool result 包含 URL / 路径 / ID / 邮箱等 pattern
- 用户 utterance 包含"记住"、"这是"、"用 xxx"、"以后"等信号词
- Agent 进行多步探索后达成目标（过程可丢，结果要记）

**评估机制**：轻量级 heuristic scorer（无需 LLM），基于信息熵和实体类型打分。

### 方向 4：分层 Context 架构（Hierarchical Memory）
**核心思想**：借鉴计算机存储层次，设计多层互补的记忆系统。

| 层级 | 内容 | 容量 | 访问方式 | Hermes 现状 |
|------|------|------|----------|------------|
| L0: Working Memory | 当前 turn + 最近 2-3 轮 | 最小 | 每次 API call 自带 | ✅ 已有 |
| L1: Active Context | 完整 messages | context window | API messages | ✅ 已有 |
| L2: Compressed Summary | 粗粒度 narrative | 理论上无限 | 压缩后注入 | ✅ 已有 |
| L3: Entity Registry | 结构化 key-value | 中等 | 每次 system prompt 注入 | ❌ 缺失 |
| L4: Session Archive | 全文历史 | 无限 | 显式 search / retrieval | ⚠️ 有 SQLite，但检索未前置 |

当前 Hermes 主要覆盖 L0-L2，**L3 和 L4 是空白**。

### 方向 5：用户意图驱动的自适应 Context Budget
**核心思想**：Tail protection 的 token budget 不是固定 20%，而是根据用户 query 动态调整。

- **检测引用意图**：用户说"之前"、"上次"、"那个"时，harness 识别出这是一个"历史引用型查询"
- **触发检索增强**：自动调用 `session_search` 或 `search_messages_fts`，把结果注入当前 context
- **动态保护策略**：引用意图强烈时，保护更长的 tail；全新任务时，允许更激进的压缩

### 方向 6：检索指针而非内容存储（Retrieval Pointer）
**核心思想**：不要求 context 包含所有信息，只要求包含"足够找到信息的索引"。

- Context compressor 不生成"发生了什么"的 narrative summary，而生成"在哪里可以找到什么"的索引表
- 例如：不是"Agent 搜索了一篇论文"，而是"在第 5 轮通过 web_search('xxx') 获得结果，论文详情可通过 session_search('arxiv 2405') 检索"
- 这需要改造 summary 的 prompt 和结构

---

## Part 3 — Hermes Harness 解构与具体适配方案

基于源码分析（`hermes-memory-harness-research.md`），我们逐项评估上述方向在 Hermes v0.9.0 中的落地可能。

### 3.1 现状诊断：Hermes 当前的防御机制与缺口

| 机制 | 源码位置 | 对长记忆丢失问题的贡献 | 缺口 |
|------|----------|----------------------|------|
| **SQLite + FTS5** | `hermes_state.py` line 34-78 | 全文可检索，理论上能找历史 | 需要 agent 主动调用 `session_search`，无自动前置检索 |
| **MemoryStore** | `tools/memory_tool.py` | 跨 session 持久化 | 依赖 agent 主动调用 memory tool，无自动触发；无结构化实体提取 |
| **ContextCompressor** | `agent/context_compressor.py` | 保护 head+tail，压缩 middle | 均匀压缩，不对高价值 entity 做特殊保护 |
| **MemoryProvider** | `agent/memory_manager.py` | 支持外部记忆插件 | 当前只注入`<memory-context>`，无 entity registry 概念 |
| **Context Fencing** | `agent/memory_manager.py` line 54-69 | 防止召回污染 | 召回内容格式是 narrative，不利于精确提取实体 |
| **Prefix Cache 保护** | `tools/memory_tool.py` line 12-14 | 系统提示稳定 | 但 L2 的 frozen snapshot 也导致 mid-session memory 修改无法即时生效 |
| **Budget 保护** | `run_agent.py` line 7770 | 限制每轮 tool 次数 | 但没有"信息价值 Budget"的概念 |

### 3.2 方案 A：在 Hermes 中实现 Entity Registry

**设计概述**：在现有 `MemoryStore` 旁增加一个 `EntityStore`，以 SQLite 表存储结构化实体。

**Schema 设计**：
```sql
CREATE TABLE entity_registry (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT,              -- 来源 session
    entity_key TEXT,              -- 描述性键，如 "arxiv-embedding-paper-url"
    entity_value TEXT,            -- 精确值，如 "https://arxiv.org/abs/2405.04532"
    entity_type TEXT,             -- url, path, id, email, version, config_value
    source_tool TEXT,             -- web_search, read_file, terminal
    source_turn INTEGER,          -- 第几轮产生
    confidence REAL,              -- 0.0-1.0，实体提取的置信度
    access_count INTEGER DEFAULT 0,  -- 被引用/访问次数
    last_accessed REAL,           -- 时间戳
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

CREATE INDEX idx_entity_type ON entity_registry(entity_type);
CREATE INDEX idx_entity_session ON entity_registry(session_id);
```

**集成点**：
1. **自动提取**：在 `model_tools.py` 的 `handle_function_call` post-hook 中，对特定 tool 的 result 做实体提取（`tools/entity_extractor.py`）
   - `web_search` / `web_extract` → 提取 URL、DOI、arxiv ID
   - `read_file` → 提取文件路径、配置 key-value
   - `terminal` → 提取路径、版本号、进程 ID
2. **注入机制**：在 `prompt_builder.py` 的 system prompt 组装阶段，查询当前 session 的高频 entity，注入为：
   ```
   Pinned Facts from this session:
   - arxiv-embedding-paper-url: https://arxiv.org/abs/2405.04532
   - project-config-lr: 3e-4
   ```
3. **人工干预**：提供 `entity_pin` tool（扩充 memory tool 或新增），允许 agent/用户显式固定 entity

**Hermes 适配难度**：中。需要改动 `hermes_state.py`（加表）、`model_tools.py`（加 hook）、`prompt_builder.py`（加注入逻辑）。

**收益**：高。直接解决"细粒度硬事实丢失"问题。

### 3.3 方案 B：改造 ContextCompressor — 非均匀压缩策略

**设计概述**：在现有 5 步压缩算法中，增加一个"高价值消息识别与特殊保护"阶段。

**具体改动**：

**Step 0：Value Tagging（压缩前预处理）**
在 `ContextCompressor.__init__` 或 `compress()` 入口，为每条历史消息打标签：
```python
_MESSAGE_VALUE_PATTERNS = {
    "url": re.compile(r'https?://\S+'),
    "path": re.compile(r'[~/]\S+\.\w+'),
    "email": re.compile(r'\S+@\S+\.\w+'),
    "id": re.compile(r'\b(?:id|uuid)=([a-f0-9-]{8,})\b'),
    "user_affirmed": re.compile(r'(请记住|记住这个|这是|重要的|logging)'),
}

def _tag_message_value(msg: dict) -> float:
    """返回消息的价值分数 0.0-1.0"""
    score = 0.0
    content = msg.get("content", "")
    if _MESSAGE_VALUE_PATTERNS["url"].search(content):
        score += 0.5  # URL 是硬事实
    if msg.get("role") == "tool" and msg.get("tool_name") in HIGH_VALUE_TOOLS:
        score += 0.3
    if _MESSAGE_VALUE_PATTERNS["user_affirmed"].search(content):
        score += 0.4
    return min(score, 1.0)
```

**Step 1：Smart Tail Protection**
现有 `_find_tail_cut_by_tokens` 是按 token budget 保护 tail。可以改为：
- 先按 token budget 计算 tail 边界
- 再检查 middle 中是否有 value_score > 0.7 的消息
- 如果有，将边界向其扩展（或单独将其加入 tail region）

**Step 2：Enhanced Summary Prompt**
`ContextCompressor` 在 summarize middle region 时，当前 prompt 只会生成 narrative。可以要求 LLM 额外输出一个 **Entity Extraction Block**：
```
Summary: [narrative summary of the conversation turns]

Key Facts Preserved:
- URL: https://...
- ID: xxx
- Path: /home/...
```
这样 summary 既保留 narrative，又保留结构化事实。

**Hermes 适配难度**：低-中。主要是 `agent/context_compressor.py` 的改动，不需要 DB schema 变更。

**收益**：中。对现有压缩不增加新存储层，但改善压缩的信息保留率。

### 3.4 方案 C：预测性自动记忆（Anticipatory Memory）

**设计概述**：利用 Hermes 已有的 `invoke_hook` 和 `_memory_nudge_interval`，升级为主动触发器。

**具体实现**：

1. **Tool Result 扫描器**：在 `model_tools.py` line 520+ 的 post-invoke hook 中，对所有 tool result 做扫描：
   ```python
   # model_tools.py, 在 registry.dispatch 之后
   if tool_name in ("web_search", "web_extract", "read_file"):
       detected_entities = extract_entities(result)
       for ent in detected_entities:
           if ent.confidence > AUTO_MEMORY_THRESHOLD:
               auto_memory(entity=ent, reason="high_value_tool_result")
   ```
2. **用户话语扫描器**：在 `run_agent.py` 的 `run_conversation` 中，user message 进入 loop 之前，做关键词扫描：
   ```python
   AUTO_MEMORY_TRIGGERS = [
       (r"请记住", "user_explicit_request"),
       (r"这是\s*(.+?)[，。]", "user_designation"),
       (r"用\s*(https?://\S+)", "user_url_reference"),
   ]
   ```
   命中时自动调用 `memory add`。
3. **过程-结果区分**：当 agent 连续调用多个 tool 最终达成用户目标时（例如：搜索 → 读取 → 总结），harness 可以检测到"成功收敛"（用户说"谢谢"、"可以了"），自动触发 memory add 记录最终结果。

**与 Hermes 现有机制的结合**：
- `invoke_hook` 已经存在（`model_tools.py` line 520+），只需在 hook 中加入 entity detection
- `memory` tool 已有实现，只需新增一个 `_auto_memory` 内部调用路径
- 可以在 `tools/memory_tool.py` 的 `MemoryStore` 中增加 `auto_add` 方法，绕过 scanning（因为是 harness 内部调用，可信）

**Hermes 适配难度**：低。主要是在现有 hook 中增加逻辑，不改动核心架构。

**风险**：可能出现过度记忆（false positive），需要设计回收机制：`access_count` 低的 entity 在 session 结束时自动清理。

### 3.5 方案 D：Session Search 前置化（Implicit Retrieval）

**设计概述**：将 `session_search` 从"agent 主动调用"改为"harness 自动执行"。

**具体实现**：

在 `run_agent.py` 的每轮 loop 中，user message 进入 API call 之前：

```python
# run_agent.py, 在 _build_api_kwargs 之前
if self._should_trigger_implicit_search(user_message):
    # 1. 用 FTS5 搜索历史消息
    matches = state_store.search_messages(
        query=user_message, 
        source_filter=[self.source], 
        limit=3
    )
    # 2. 如果匹配度高，把结果注入 context
    if matches:
        context_injection = format_search_results(matches)
        # 作为 user message 的前缀，或作为单独的 system message
        api_messages.insert(1, {
            "role": "system",
            "content": f"Relevant historical context found:\n{context_injection}"
        })
```

**触发条件判断**（轻量级，不需要 LLM）：
- 用户消息包含"之前"、"上次"、"那个"、"还记得"等指示词
- 用户消息中的实体在当前 L0/L1 context 中不存在（可能是在引用历史）
- 用户消息长度很短（通常引用型查询较短）

**Hermes 适配难度**：低。`hermes_state.py` 的 `search_messages` 已经实现（line 1001-1091），只需在 loop 中调用。但需要注意：
- 每次调用 FTS5 增加约 50-100ms，对实时性影响小
- 注入的 context 需要计入 token budget，避免 injection 导致 overflow

**收益**：高。对用户完全透明，直接解决"用户提历史时 agent 失忆"的问题。

### 3.6 方案 E：Tool Result 双轨存储（Distillation + Archive）

**设计概述**：Tool result 进入 messages 时走"双轨"：
- **Track A (Light)**：结构化摘要进入 messages，token 占用小
- **Track B (Archive)**：原始 text 存入新表 `tool_summaries`，按需检索

**Schema**：
```sql
CREATE TABLE tool_summaries (
    id INTEGER PRIMARY KEY,
    session_id TEXT,
    turn INTEGER,
    tool_name TEXT,
    arguments TEXT,       -- JSON
    raw_result TEXT,      -- 原始完整输出
    distilled TEXT,       -- 结构化 JSON/YAML 摘要
    tokens_saved INTEGER  -- raw - distilled 的 token 差
);
```

**Distillation 策略**（按 tool）：
| Tool | Distillation 方法 |
|------|-------------------|
| `web_search` / `web_extract` | 提取 title + URL + 1 句摘要 |
| `read_file` | 文件路径 + 行数 + "包含 X 类函数/配置" |
| `terminal` | 命令 + exit_code + "成功/失败" + 关键输出行 |
| `search_files` | 匹配数 + 前 3 个文件路径 |
| `browser_*` | URL + 页面标题 + 关键文本片段 |

**注入机制**：`handle_function_call` 返回时，如果当前 messages 已接近 budget 阈值，用 distilled version 替换 raw result；原始文本存入 `tool_summaries`。Agent 需要详细内容时，通过新的 `fetch_tool_detail(tool_summary_id)` 获取。

**Hermes 适配难度**：中-高。需要：
- 每个 tool 模块增加 `distill_result()` 方法
- DB schema 变更
- 改造 `handle_function_call` 的返回逻辑
- 新增 `fetch_tool_detail` tool

**收益**：极高。根本性降低 context 的 token 占用，延缓 overflow。

---

## Part 4 — 综合评估与实施建议

### 4.1 方案对比矩阵

| 方案 | 改动范围 | 实现复杂度 | 对问题缓解度 | 用户感知 | 推荐优先级 |
|------|---------|-----------|------------|---------|-----------|
| A: Entity Registry | DB + prompt_builder + tool hook | 中 | ⭐⭐⭐⭐⭐ | 透明 | **P0** |
| B: 非均匀压缩 | context_compressor only | 低-中 | ⭐⭐⭐ | 透明 | P1 |
| C: 自动记忆 | model_tools hook | 低 | ⭐⭐⭐⭐ | 几乎透明 | **P0** |
| D: Search 前置 | run_agent loop | 低 | ⭐⭐⭐⭐⭐ | 透明 | **P0** |
| E: 双轨存储 | 所有 tool 模块 + DB | 高 | ⭐⭐⭐⭐⭐ | 需新增 tool | P1 |

### 4.2 推荐实施路径（分阶段）

#### Phase 1：立竿见影（1-2 周）

**D + C 组合**：Session Search 前置化 + 预测性自动记忆

- D 解决"用户提历史时"的即时失忆问题
- C 解决"agent 应该记住但没记"的问题
- 两者都利用现有基础设施（SQLite FTS5、memory tool、invoke_hook），改动最小，收益最大

具体代码改动点预估：
1. `run_agent.py` line ~8080：在 user message 处理后增加 `_implicit_retrieval()` 调用
2. `model_tools.py` line ~520：在 `invoke_hook` 后增加 `_auto_memory_scan()`
3. `tools/memory_tool.py`：为 MemoryStore 增加 `auto_add()` 方法（跳过 threat scan，因为是内部调用）

#### Phase 2：结构性改进（3-4 周）

**A + B 组合**：Entity Registry + 非均匀压缩

- A 建立结构性基础设施，让硬事实有"家"可归
- B 改善现有压缩的信息保留效率
- 两者互补：Entity Registry 保护压缩后必然丢失的硬事实，非均匀压缩减少需要保护的"面积"

关键交付物：
- `hermes_state.py`：`entity_registry` 表 + CRUD 方法
- `agent/prompt_builder.py`：`format_pinned_entities()` 注入逻辑
- `agent/context_compressor.py`：`value_score` + `smart_tail` 改造
- 新增 `tools/entity_registry_tool.py`：提供 `entity_add`, `entity_search`, `entity_remove`

#### Phase 3：根本性改造（6-8 周）

**E：Tool Result 双轨存储**

- 所有 tool 模块增加 `distill_result()` 方法
- 这是架构级改动，影响面广，但收益是根本性的：凡 tool 占 context 空间大的场景都被缓解
- 适合在业务稳定后作为性能优化项推进

### 4.3 一个务实的最小可行方案（MVP）

如果不确定投入多少资源，可以先做一个 **"Entity Pinner MVP"**：

1. **不改动 DB schema**，复用现有的 `memory` tool
2. 在 `invoke_hook` 中，检测到 tool result 包含 URL/路径/ID 时，**自动**调用 `memory add`（通过编程方式调用 `MemoryStore.add`，不走 LLM tool call）
3. 在 `prompt_builder.py` 中，除了注入常规的 `MEMORY.md` 内容，还注入一个特别节段 `Session Pinned Facts`，把当前 session 中通过 auto-memory 记录的 entity 列出来
4. 在 `run_agent.py` 中，用户消息包含"之前"、"那个"时，用现有的 `search_messages` 检索历史，将匹配片段自动注入 context

这个 MVP 不改架构，不增表，纯逻辑增强，可以在 1 周内落地，验证效果后再决定是否推进 Phase 2/3。

---

## Part 5 — 理论视角的延伸思考

### 5.1 从"无损压缩"到"有效索引"

用户文档中提到："无损压缩是理论不可能的"。正确。更应追求的是：

> Context 中不需要包含信息本体，只需要包含**足够找到本体的索引**。

这是 Claude Code `/compact` 指令的核心理念，也是向量数据库 RAG 的基础假设。Hermes 的 `session_search` 已经有了索引能力，关键缺口是：
1. **索引没有被自动使用**（需要 agent 主动调用 tool）
2. **索引密度不够**（原始 tool result 太冗长，难以被 FTS5 有效匹配）

### 5.2 信息熵与注意力分配

保留高熵信息、丢弃低熵信息是合理的，但要注意：
- **信息熵 = -Σ p(x) log p(x)**，是统计概念
- LLM 的 attention 不是按信息熵均匀分配的，它偏向**位置近**、**格式突出**、**与 query 相关**的信息
- 因此：即使 Entity Registry 中存储了 URL，如果它被淹没在一堆 entity 里，LLM 仍然可能"看不见"

**工程启示**：
- Entity Registry 应该有大小上限（如 20 个 entity），通过 LRU + 重要性评分淘汰
- 重要的 entity 应该在 system prompt 中以结构化格式（bullet list）呈现，提高 attention 权重
- 当前 query 相关的 entity 应该被提升到 user message 中，而非放在 system prompt 底部

### 5.3 Session 切换 vs Session 永生的权衡

用户文档中担忧"Session 切换存在记忆压缩和丢失问题"。但 Session 切换本身不是敌人：

| Session 策略 | 优点 | 缺点 |
|-------------|------|------|
| 永生 Session | 上下文完整，用户无感知断裂 | 必然溢出，压缩导致失忆 |
| 主动切换 | context 始终紧凑，agent 表现稳定 | 用户需交代背景，有断裂感 |
| **智能切换** | 在"上下文还很紧凑"时主动分 session，携带 Entity Registry | 实现复杂，但折中最优 |

"智能切换"的触发条件：
- Context 达到 70% 容量时，不是被动压缩，而是主动把当前 session "封存"
- 新 session 携带：Entity Registry + 1 句任务摘要 + 最近 3 轮对话
- 旧 session 进入 archive，可通过 search 随时检索

这其实就是把 Phase 1+2 的机制，用更主动的方式驱动 Session 生命周期管理。

---

## 附录：直接可复用的 Hermes 源码扩展点

| 目标 | 文件 | 插入位置 | 修改类型 |
|------|------|----------|---------|
| 自动提取 entity | `model_tools.py` | `invoke_hook` 后 (line 527+) | 增量 hook |
| 自动记忆触发 | `tools/memory_tool.py` | 新增 `MemoryStore.auto_add()` | 新方法 |
| Search 前置 | `run_agent.py` | user message 处理之后 (line 8080+) | 条件分支 |
| Entity 注入 prompt | `agent/prompt_builder.py` | `_build_system_prompt()` 内 | 追加节段 |
| Entity DB Schema | `hermes_state.py` | `SCHEMA_VERSION` 升级至 7 | 加表 + 迁移 |
| 非均匀压缩 | `agent/context_compressor.py` | `_find_tail_cut_by_tokens()` 之前 | 前置评分 |
| Distillation | `tools/<tool>_tool.py` | 各自增加 `distill_result()` | 各 tool 模组 |
| 双轨存储 Schema | `hermes_state.py` | `SCHEMA_VERSION` 升级 | 加表 |

---

*Brainstorm document for long-context memory loss problem.*
*可与 hermes-memory-harness-research.md 交叉阅读以理解源码上下文。*
