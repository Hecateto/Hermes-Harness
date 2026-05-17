# Hermes Context Pipeline 深度拆解 — L1: System Prompt 组装层

**文件**: `agent/prompt_builder.py` (1,038 行) + `run_agent.py::_build_system_prompt` (lines 3097–3262)

**分析结论**: Hermes 的 system prompt 组装不是简单的字符串拼接，而是一套**分层的、带缓存的、带安全过滤的、模型感知的**精密管道。每 session 仅构建一次的核心决策，直接服务于 prefix cache 最大化。

---

## 大白话介绍

想象你要写一份给 AI 的"入职手册"。这份手册不是直接打开一个空白文档就写，而是像拼乐高一样，从 13 个不同的盒子里取出零件按顺序拼起来：

1. **先拼身份卡**（你是谁、叫什么）
2. **再拼公司文化**（.hermes.md、AGENTS.md 等本地文件里写的规矩）
3. **加上顾客档案**（用户之前让你记住的偏好）
4. **贴上安全告示**（扫描有没有坏人偷偷塞了奇怪指令）
5. **附上工具说明书**（你能用哪些工具、怎么用）

拼完之后，这份手册被贴到冰箱上（`_cached_system_prompt`），只要没发生大事（context compression、新 session），下一桌客人来就不用重新拼，直接看冰箱上的——这就是**prefix cache**的秘密。

关键洞察：**句子顺序不能乱**，因为 cache 只认前缀；**安全扫描必须拼完之后做**，因为任何一层都可能是攻击入口。

---

## 1. 整体架构: 13 层组装顺序

`_build_system_prompt()` 在 `run_agent.py:3097` 中实现，13 个 layer 按以下顺序拼接（`"\n\n".join`）:

| # | Layer | 来源文件 | 条件/触发 | 缓存策略 |
|---|-------|---------|----------|---------|
| 1 | **Agent Identity** | SOUL.md 或 DEFAULT_AGENT_IDENTITY | 无条件 | 每 session 一次 |
| 2 | **Tool-aware Guidance** | MEMORY_GUIDANCE / SESSION_SEARCH_GUIDANCE / SKILLS_GUIDANCE | 对应 tool 在 `valid_tool_names` 中 | 内联常量 |
| 3 | **Nous Subscription** | build_nous_subscription_prompt() | managed_nous_tools_enabled() | 动态计算 |
| 4 | **Tool-use Enforcement** | TOOL_USE_ENFORCEMENT_GUIDANCE + 模型特定指导 | config `agent.tool_use_enforcement` + 模型名匹配 | 内联常量 |
| 5 | **User/Gateway System Msg** | 外部传入 `system_message` | 非 None 时 | 外部决定 |
| 6 | **Built-in Memory Block** | memory_store.format_for_system_prompt("memory") | `_memory_enabled` | 每 turn 更新? 需查 |
| 7 | **User Profile Block** | memory_store.format_for_system_prompt("user") | `_user_profile_enabled` | 每 turn 更新? 需查 |
| 8 | **External Memory Provider** | memory_manager.build_system_prompt() | `_memory_manager` 存在 | 依赖 provider |
| 9 | **Skills Index** | build_skills_system_prompt() | skills tools 已加载 | **双层缓存** |
| 10| **Project Context Files** | build_context_files_prompt() | `!skip_context_files` | 每 session |
| 11| **Timestamp + Metadata** | hermes_time.now() | 无条件 | 冻结在 build 时刻 |
| 12| **Alibaba Workaround** | 硬编码模型身份提示 | provider == "alibaba" | 动态 |
| 13| **Environment + Platform Hints** | build_environment_hints() + PLATFORM_HINTS | WSL/平台匹配 | 动态 |

**关键设计**: "Called once per session (cached on `self._cached_system_prompt`)" (line 3101)。这个决策的工程学动机是**最大化 Anthropic/Claude prefix cache 命中**——系统提示在每轮 API call 中保持不变，前缀 token 的 KV cache 可以复用。

> **盲区暴露**: Layer 6/7 的 memory block 是否真的"per session 不变"？如果 `format_for_system_prompt("memory")` 在每轮都返回最新内容，那么 `_cached_system_prompt` 的缓存声明和实际行为可能矛盾。这个假设需要 memory_store 源码验证。

---

## 2. Prompt Injection 防御系统 (prompt_builder.py:36–73)

这是 system prompt **入口处的安全闸**。

### 2.1 威胁模式库 (`_CONTEXT_THREAT_PATTERNS`, line 36)

10 条正则，覆盖 5 类攻击向量：

| 类别 | 模式示例 | 内部标签 |
|-----|---------|---------|
| **指令覆盖** | `ignore previous instructions` | `prompt_injection` |
| **欺骗隐藏** | `do not tell the user` | `deception_hide` |
| **系统提示覆盖** | `system prompt override` | `sys_prompt_override` |
| **规则否定** | `disregard your instructions` | `disregard_rules` |
| **限制绕过** | `act as if you have no restrictions` | `bypass_restrictions` |
| **HTML 注释注入** | `<!-- ... ignore ... -->` | `html_comment_injection` |
| **隐藏 div** | `<div style="display:none` | `hidden_div` |
| **翻译执行** | `translate ... and execute` | `translate_execute` |
| **凭证外泄** | `curl ... ${KEY}` | `exfil_curl` |
| **密钥读取** | `cat .env credentials` | `read_secrets` |

### 2.2 不可见字符检测 (`_CONTEXT_INVISIBLE_CHARS`, line 49)

10 个 Unicode 控制字符：
- **零宽字符**: `\u200b` (ZWSP), `\u200c` (ZWNJ), `\u200d` (ZWJ), `\u2060` (WJ), `\ufeff` (BOM)
- **双向文本覆盖**: `\u202a`–`\u202e` (LRE, RLE, PDF, LRO, RLO)

### 2.3 防御行为 (`_scan_context_content`, line 55)

```python
if findings:
    logger.warning("Context file %s blocked: %s", filename, ", ".join(findings))
    return f"[BLOCKED: {filename} contained potential prompt injection ...]"
```

**行为特征**: 不是拒绝加载（抛异常），而是**替换为阻断声明**。这样 agent 仍然知道这个文件存在，但不会执行其内容。这是一种**fail-soft** 设计。

**作用范围**: 仅对项目上下文文件（SOUL.md, .hermes.md, AGENTS.md, CLAUDE.md, .cursorrules）生效，**不对 skills prompt 或用户输入生效**。

> **安全盲区**: prompt injection scan 发生在 system prompt 组装阶段。如果用户消息中包含 injection payload，这里不会拦截。实际拦截在哪里？需查 gateway 或 run_agent.py 的用户消息处理路径。

---

## 3. 项目上下文文件发现机制

### 3.1 .hermes.md / HERMES.md 搜索策略 (`_find_hermes_md`, line 92)

```
搜索路径: cwd → parent → ... → git root (inclusive)
文件名匹配: .hermes.md 或 HERMES.md
终止条件: 找到 .git 目录时停止向上遍历
```

**设计意图**: 项目级配置应放在项目根目录（git root），子目录可以放覆盖配置，但搜索到 git root 就停止，防止跨项目污染。

### 3.2 优先级仲裁 (`build_context_files_prompt`, line 998)

```python
project_context = (
    _load_hermes_md(cwd_path)
    or _load_agents_md(cwd_path)
    or _load_claude_md(cwd_path)
    or _load_cursorrules(cwd_path)
)
```

**first-match-wins**: 只加载**一种**项目上下文。.hermes.md (walk to git root) > AGENTS.md (cwd only) > CLAUDE.md (cwd only) > .cursorrules (cwd + .cursor/rules/*.mdc)。

> **深层含义**: Hermes 强烈推荐使用 `.hermes.md` 作为项目配置标准，其他格式（AGENTS.md, .cursorrules）只是向后兼容。

### 3.3 YAML Frontmatter 剥离 (`_strip_yaml_frontmatter`, line 113)

```python
if content.startswith("---"):
    end = content.find("\n---", 3)
    if end != -1:
        body = content[end + 4:].lstrip("\n")
```

当前仅剥离不解析，注释说明"frontmatter may contain structured config (model overrides, tool settings) that will be handled separately in a future PR"。这是一个**预留扩展点**。

### 3.4 截断策略 (`_truncate_content`, line 873)

- **上限**: `CONTEXT_FILE_MAX_CHARS = 20_000`
- **分配**: head 70%, tail 20%, 中间丢弃并插入标记

```
head = content[:14000]
tail = content[-4000]
marker = "[...truncated filename: kept 14000+4000 of total chars...]"
```

> **设计问题**: 70%/20% 的分配对代码文件（尾部更重要）和文档（头部更重要）一视同仁。这是**uniform truncation**，和 ContextCompressor 的 uniform compression 是同根病灶。

---

## 4. Skills Prompt 双层缓存系统 (prompt_builder.py:420–800)

这是 prompt builder 中最复杂的子系统，也是**工程上最值得借鉴**的部分。

### 4.1 缓存层级

```
Layer 1: 内存 LRU (OrderedDict, max=8, _SKILLS_PROMPT_CACHE_LOCK)
  └─ key: (skills_dir, external_dirs, available_tools, available_toolsets, platform_hint)
Layer 2: 磁盘快照 (.skills_prompt_snapshot.json)
  └─ 验证: manifest (mtime_ns + st_size of all SKILL.md/DESCRIPTION.md)
冷路径: 完整文件系统扫描 + 写 snapshot
```

### 4.2 缓存失效策略

不监听文件系统事件，而是**惰性验证**：
- 内存缓存未命中 → 检查磁盘快照
- 磁盘快照的 manifest 与当前文件系统的 mtime/size 比较 → 不匹配则走冷路径
- 冷路径扫完全部 SKILL.md → 写 snapshot 供下次复用

这是一种**乐观缓存**——假设 skill 文件不常变更，用 stat 信息代替哈希以节省 I/O。

### 4.3 Skill 条件过滤 (`_skill_should_show`, line 544)

```python
# fallback_for: 主工具可用时隐藏该 skill
if ts in available_toolsets: return False
# requires: 必需工具不可用时隐藏该 skill
if ts not in available_toolsets: return False
```

这实现了 skill 的**条件激活**——例如 "huggingface-hub" skill 只在 hf 工具可用时注入。

### 4.4 多目录优先级

```
~/.hermes/skills/     (本地, 可写, 优先)
external_dirs[]       (外部, 只读, 本地同名 skill 会覆盖)
```

```python
for ext_dir in external_dirs:
    if skill_name in seen_skill_names:
        continue  # 本地已有，跳过外部
```

---

## 5. 模型特定行为指导

### 5.1 Tool-use Enforcement 触发规则 (run_agent.py:3148–3171)

config.yaml 中 `agent.tool_use_enforcement` 支持 4 种模式：

| 配置值 | 行为 |
|-------|-----|
| `"auto"` (default) | 匹配 `TOOL_USE_ENFORCEMENT_MODELS` = ("gpt", "codex", "gemini", "gemma", "grok") |
| `true` / `"always"` | 所有模型都注入 |
| `false` / `"never"` | 不注入 |
| list[str] | 自定义模型名子串匹配 |

注入内容分两层：
1. **通用层** (`TOOL_USE_ENFORCEMENT_GUIDANCE`): 禁止"描述意图而不执行"
2. **模型层**: OpenAI 的 execution discipline (tool persistence, prerequisite checks, verification) 或 Google 的 operational directives (absolute paths, parallel calls)

### 5.2 Developer Role 切换

`DEVELOPER_ROLE_MODELS = ("gpt-5", "codex")` (line 283)

注意: `"developer"` 角色**不在** `_build_system_prompt 内部处理。注释说明"The swap happens at the API boundary in `_build_api_kwargs()` so internal message representation stays consistent ("system" everywhere)" (line 281–282)。这是**adapter 层的职责**，不是 prompt builder 层的。

---

## 6. 平台与环境感知

### 6.1 PLATFORM_HINTS (line 285)

11 个平台：whatsapp, telegram, discord, slack, signal, email, cron, cli, sms, bluebubbles, weixin, wecom。

每个平台定义：
- 是否支持 markdown
- 媒体文件发送语法 (`MEDIA:/abs/path`)
- 文件格式限制
- 响应长度/风格约束

**关键语法**: `MEDIA:/absolute/path/to/file` 是 Hermes 内部约定，不是 markdown 标准。

### 6.2 Environment Hints (line 399)

当前仅检测 WSL (`is_wsl()`)。这是可扩展点，注释预留了 Termux, Docker 等场景。

---

## 7. 被忽略的关键细节

### 7.1 cache 的真实语义

`_build_system_prompt` 的 docstring 说"cached on `self._cached_system_prompt`"。通过源码确认，缓存失效逻辑在 `run_agent.py:3424` 的 `_invalidate_system_prompt()` 中：

```python
def _invalidate_system_prompt(self):
    self._cached_system_prompt = None
    if self._memory_store:
        self._memory_store.load_from_disk()
```

**调用时机**:
- Context compression 后 (`run_agent.py:6758`)
- CLI 新 session / fork session / reset session 时 (`cli.py:4132`, `4218`, `4332`)

**关键结论**: system prompt **不会在普通 turn 结束后自动重建**。只有 context compression、session 重启、或显式重置时才会 invalidate。这意味着：
- 用户通过 `memory add` 写入的新记忆会在本 session 中持久化到 SQLite
- 但如果不触发 compression/reset，这些新记忆**不会**进入当前 session 的 system prompt
- 这是**有意为之**——避免每轮重建破坏 prefix cache

### 7.2 `_sanitize_api_messages` (run_agent.py:3278)

这是另一个与 prompt 相关的关键函数（就在 system prompt 之下），负责：
1. **角色白名单过滤**: 只保留 system/user/assistant/tool/function/developer
2. **orphan tool_call 清理**: 修复被 context compression 打断的 tool_call/tool_result 对

这说明 context compression 后可能出现**消息结构断裂**，需要在每次 API call 前修复。

### 7.3 `TERMINAL_CWD` 环境变量

_GATEWAY 模式的关键设计_ (line 3219):
```python
_context_cwd = os.getenv("TERMINAL_CWD") or None
```

Gateway 进程运行在 hermes-agent 安装目录，如果不覆盖 cwd，会加载到安装目录的 AGENTS.md，导致**每轮增加 ~10k token**。

---

## 8. 当前发现的 4 个潜在问题

| # | 问题 | 位置 | 影响 | 优先级 |
|---|------|------|------|-------|
| 1 | **System prompt cache invalidation 未明确** | run_agent.py:3101 | 如果 memory 每轮更新但 system prompt 未 rebuild，新记忆不会进入系统提示 | 高 |
| 2 | **Uniform truncation** | prompt_builder.py:411–413 | 对所有文件类型使用 70/20 固定比例，可能丢失关键尾部信息 | 中 |
| 3 | **Prompt injection scan 不覆盖 user message** | prompt_builder.py:36–73 | 用户消息中的 injection payload 无此层防御 | 中 |
| 4 | **Skills cache key 不包含文件内容哈希** | prompt_builder.py:608–614 | manifest 用 mtime/size，可能误判（文件 touch 不改变内容） | 低 |

---

## 9. 与整体 Context Pipeline 的接口

prompt_builder.py 是 context 管道的**静态端**——它在 session 开始时构建一个相对稳定的基底。与之交互的其他模块：

- **下游 (消费者)**: `run_agent.py` 将 `_build_system_prompt()` 的结果作为 `messages[0]` (role="system") 送入 API
- **上游 (依赖)**: `memory_store.format_for_system_prompt()` 为其提供 memory/user block (layer 6/7)
- **并行输入**: `memory_manager.build_system_prompt()` 提供外部 memory provider 的 block (layer 8)
- **交叉关注点**: `_sanitize_api_messages()` 在其后运行，修复可能的结构断裂

**下一步应深入**: `memory_store` 的 `format_for_system_prompt` 实现，验证问题 #1（cache invalidation）是否成立。以及 `_sanitize_api_messages` 的完整逻辑，理解 context compression 造成的 orphan 问题。

---

*文档生成时间: 基于 hermes-agent v0.9.0 源码*  
*分析范围: L1 System Prompt 组装层*  
*待验证假设: memory block 是否每 session 冻结？由 L2 (Memory Manager) 解答。*
