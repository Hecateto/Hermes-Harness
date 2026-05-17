# Phase 4 — Sub-agent & Delegation System 深度调研

> **调研范围**: Hermes Agent v0.9.0 Sub-agent 委托架构
> **核心文件**: `tools/delegate_tool.py` (1103 行), `run_agent.py` (迭代预算与中断传播)
> **对应 PR**: #7295 (v0.9.0 Child Activity Propagation)

---

## 1. 总览与职责边界

`delegate_task` 是 Hermes 的"自我繁殖"接口：父 agent 在运行途中可以 spawn 出一个或多个子 agent，把独立任务分派出去。设计哲学非常明确：

- **上下文隔离**: 子 agent 拥有独立的 conversation、terminal session、toolset、iteration budget，其间的 tool calls 与 reasoning 不会污染父 agent 的上下文窗口。
- **权限最小化**: 子 agent 默认被剥夺 `delegate_task`/`clarify`/`memory`/`send_message`/`execute_code` 等危险或交互式工具，且无法递归 spawn 孙 agent。
- **结果聚合**: 父 agent 只看到一次 `delegate_task` 调用和一份结构化 JSON 摘要，里面包含每个子任务的 status、summary、token 消耗、tool trace。
- **活性保障**: v0.9.0 新增 heartbeat + progress callback relay，既防止 gateway 因父 agent 长期阻塞而误杀，也让用户实时看到子 agent 的工作进度。

---

## 2. Spawn 机制与 Context 隔离 (4.1)

### 2.1 两种委托模式

`delegate_task()` 支持 **Single** 和 **Batch** 两种模式：

| 模式 | 触发条件 | 执行方式 |
|------|---------|---------|
| Single | 提供 `goal`（字符串），无 `tasks` | 直接在当前线程同步运行 `_run_single_child`，无 ThreadPoolExecutor 开销 |
| Batch | 提供 `tasks: List[Dict]`（每个元素含 `goal`/`context`/`toolsets`） | `ThreadPoolExecutor(max_workers=max_children)` 并行提交，最终按 `task_index` 排序返回 |

Batch 上限由 `_get_max_concurrent_children()` 决定，优先级链：**config.yaml `delegation.max_concurrent_children` > env `DELEGATION_MAX_CONCURRENT_CHILDREN` > default 3**。若 `len(tasks)` 超过上限，`delegate_task` 直接返回 `tool_error` JSON，**不会自动降速或分片**。

### 2.2 Child Agent 构建流程 `_build_child_agent`

所有子 agent **必须在主线程构建**（thread-safe construction），只有 `run_conversation()` 才被丢进线程池执行。关键参数映射如下：

| 参数 | 来源规则 |
|------|---------|
| `model` / `provider` / `base_url` / `api_key` / `api_mode` | delegation config override > parent inherit（None 穿透） |
| `acp_command` / `acp_args` | override > parent inherit |
| `reasoning_config` | `delegation.reasoning_effort`（若配置合法）> parent inherit |
| `max_iterations` | 参数 > config > **default 50**（注意：parent default 是 90） |
| `enabled_toolsets` | 显式 `toolsets` ∩ parent effective toolsets → `_strip_blocked_tools()` |
| `ephemeral_system_prompt` | `_build_child_system_prompt(goal, context, workspace_hint)` |
| `quiet_mode` / `skip_memory` / `skip_context_files` | **强制 True** |
| `session_db` | 共享 parent 的 `_session_db`（便于 session 持久化） |
| `parent_session_id` | 共享 parent 的 `session_id`（建立父子血缘） |
| `iteration_budget` | **强制 None**（子 agent 拥有独立预算） |

**Toolset 继承采用"交集"策略**：子 agent 无权获得父 agent 也没有的工具。若父 agent `enabled_toolsets=None`（表示启用全部），则从 `parent_agent.valid_tool_names` 反查每个工具所属的 toolset 作为 effective toolset。

随后 `_strip_blocked_tools()` 移除包含危险工具的工具集名称（硬编码：`delegation`, `clarify`, `memory`, `code_execution`）。注意：这里与 `DELEGATE_BLOCKED_TOOLS`（单个工具黑名单）是两个层面的过滤，若未来新增包含 `execute_code` 的工具集（如 `"sandbox"`），`_strip_blocked_tools` 不会自动感知。

### 2.3 深度限制

`MAX_DEPTH = 2`，在 `delegate_task()` 入口硬拦截：

```python
depth = getattr(parent_agent, '_delegate_depth', 0)
if depth >= MAX_DEPTH:
    return json.dumps({"error": "Delegation depth limit reached (2). Subagents cannot spawn further subagents."})
```

形成刚性边界：parent(0) → child(1) → grandchild **rejected**(2)。

### 2.4 全局状态隔离 — 双重 Save/Restore

**问题根源**: `_build_child_agent()` 调用 `AIAgent()` → `get_tool_definitions()` → 覆盖进程全局变量 `model_tools._last_resolved_tool_names`。若不恢复，父 agent 后续调用 `execute_code` 或其他依赖该全局的逻辑时会使用子 agent 的截断工具列表。

**保护措施（双重保险）**:

1. **Build 阶段** (`delegate_task` 主线程): 构造所有 children 前保存 `list(model_tools._last_resolved_tool_names)`，在 `try/finally` 中构建完成后统一恢复。
2. **Run 阶段** (`_run_single_child` 子线程): 子线程结束前，再次从 `child._delegate_saved_tool_names` 恢复全局。

AGENTS.md line 435-436 明确警告开发者：该全局在子 agent 运行期间可能暂时 stale，读取它的代码必须意识到这一点。

---

## 3. 并发模型与调度策略 (4.2)

### 3.1 Batch 并行执行

Batch 模式下，ThreadPoolExecutor 一次性提交所有 futures，由 executor 自动限流（活跃线程数达到 `max_workers` 时其余排队）。`as_completed` 提供乱序完成的实时反馈：每完成一个子任务，在 spinner 上方打印完成行（`✓ [1/3] goal_preview  (45.6s)`），并更新 spinner 文本为剩余任务数。

### 3.2 并发截断 — 两层防护

模型可能在**同一 turn** 内发出**多个独立的 `delegate_task` tool_calls**（而非一个 tool_call 里带 `tasks` 数组）。为此 Hermes 设置了第二层防护：

- **第一层** (`delegate_tool.py:671-680`): 对单个 `delegate_task` 调用内的 `tasks` 数组长度做硬检查，超限直接返回 error JSON。
- **第二层** (`run_agent.py:_cap_delegate_task_calls`): 在 Post-call guardrails 阶段（line ~9918）截断同一 turn 内多余的独立 `delegate_task` 调用，保留所有非-delegate 调用，**静默丢弃**多余 delegate 调用并记 warning log。

**不一致风险**: 第一层是 rigid error，第二层是 silent truncation。模型收到 error 后可能在下一 turn 重试；但静默截断会让模型误以为某些 task 已被接受执行，实际上从未被调度。

### 3.3 同一 Batch 内 Tool 调用的并行安全

`run_agent.py` 中还独立实现了普通 tool batch 的并行执行安全策略（与 subagent 并发是不同层面的机制）：

- `_NEVER_PARALLEL_TOOLS = {"clarify"}`: 若 batch 中出现 clarify，全部 fallback 到顺序执行。
- `_PARALLEL_SAFE_TOOLS`: read-only 工具（read_file, search_files, skill_view, web_search 等）默认可并行。
- `_PATH_SCOPED_TOOLS = {read_file, write_file, patch}`: 若目标路径有重叠，则禁止并行。

Subagent 的并发是"任务级"（多个独立 AIAgent 实例），而上述策略是"工具级"（同一 agent 的多个 tool_call 在线程池中并发执行）。二者正交，互不干扰。

---

## 4. Budget 传播与 Grace Call (4.3)

### 4.1 IterationBudget 设计

`run_agent.py:170-212`，基于 `threading.Lock` 的线程安全计数器：

```python
class IterationBudget:
    def consume(self) -> bool:  # 占用一次额度
    def refund(self) -> None:   # 退还一次额度（专供 execute_code）
```

**关键设计决策：子 agent 不共享父 agent 的 budget**。`delegate_tool.py:304-306` 注释明确说明：

> Each subagent gets its own iteration budget capped at max_iterations... This means total iterations across parent + subagents can exceed the parent's max_iterations.

- 优点：子 agent 不会因父 agent 快要耗尽预算而被中途掐断。
- 缺点：总 token/API 调用成本可能远超用户预期。父 agent 在 `delegate_task` 期间被阻塞，但该阻塞时间不消耗 parent iteration 额度。

### 4.2 _budget_grace_call — "遗言机制"

当 `iteration_budget` 耗尽时，Hermes 不会立即掐断 agent，而是给予一次 **grace call**：

```python
while (api_call_count < self.max_iterations and self.iteration_budget.remaining > 0) or self._budget_grace_call:
    if self._budget_grace_call:
        self._budget_grace_call = False   # 消费令牌
    elif not self.iteration_budget.consume():
        break                               # 真正耗尽
```

设计意图（`__init__` 注释 line 786-790）：在 budget 耗尽时向模型注入一条通知消息，允许一次最终的 API call。如果模型仍返回 tool_calls 而非文本，循环在 grace flag 被消费后必然退出，随后由 `_handle_max_iterations()` 强制请求 summary（**在 while 循环外部，不计入 budget**）。

**关键细节**: `_budget_exhausted_injected` (line 791) 是一个**死代码**——全代码库（除测试外）无业务引用，当前 grace call 机制完全不依赖它。

### 4.3 Budget 耗尽时的用户体验

| 阶段 | 行为 |
|------|------|
| Loop 内部首次耗尽 | `_safe_print` 输出 warning（受 quiet_mode 控制），**不注入消息**，直接 break |
| Loop 外部无 final_response | `_handle_max_iterations()` 注入 user message 请求 summary，发起一次不带 tools 的额外 API call |

---

## 5. Child Activity Propagation (4.4)

v0.9.0 新增的核心特性（#7295），解决两个痛点：

1. **可观测性**: 子 agent 运行期间用户面对静默 spinner，不知道它在做什么。
2. **保活性**: 父 agent 被阻塞在 `delegate_task` 时，其 `_last_activity_ts` 冻结，可能被 gateway inactivity timeout 误杀。

### 5.1 Progress Callback Relay

由 `_build_child_progress_callback()` 构建适配层，将子 agent 的 tool 事件转换为父 agent 的可视化输出：

**CLI 路径** (`spinner.print_above()`):
- `tool.started`: 打印树形行 ` [N] ├─ emoji tool_name "preview..."`
- `thinking` / `reasoning.available`: 打印 `├─ 💭 "thinking snippet..."`（超过 55 字符截断）
- `tool.completed`: 不打印（避免重复）
- batch 模式加前缀 `[1]` / `[2]`，single 模式无前缀

`KawaiiSpinner.print_above()` 的实现细节：`self._out` 使用 spinner 创建时捕获的 stdout 引用（而非 `sys.stdout`），因此在 `redirect_stdout(devnull)` 下仍能正常工作；使用空格填充清行（而非 `\033[K`），避免 `prompt_toolkit` 下出现乱码 `?[K`。

**Gateway 路径** (`parent_cb` batch):
- 内部维护 `_batch: List[str]`，`_BATCH_SIZE = 5`
- 每满 5 个 tool 名称触发 flush：`parent_cb("subagent_progress", f"🔀 {summary}")`
- 子 agent 完成后 `_flush()` 发送未满 batch
- thinking 事件**不 relay 到 gateway**（避免聊天界面噪声）

**双路径并存**: 若父 agent 同时有 spinner 和 `tool_progress_callback`，两个路径同时触发。

### 5.2 Heartbeat 机制

在 `_run_single_child()` 内启动 daemon thread，`_HEARTBEAT_INTERVAL = 30` 秒：

```python
def _heartbeat_loop():
    while not _heartbeat_stop.wait(_HEARTBEAT_INTERVAL):
        touch = getattr(parent_agent, '_touch_activity', None)
        desc = f"delegate_task: subagent running {child_tool} (iteration {iter}/{max})"
        touch(desc)
```

- 动态描述：优先读取 `child.get_activity_summary()` 中的 `current_tool`、`api_call_count` / `max_iterations`。
- 清理：子 agent 结束后 `_heartbeat_stop.set()` + `join(timeout=5)`，保证不会继续 touch 父 agent。

### 5.3 Tool Trace 提取

从 `child.run_conversation()` 返回的 `messages` 中，利用 `tool_call_id` 配对：

- assistant 消息中的 `tool_calls` → 基础条目 `{"tool": name, "args_bytes": len}`
- tool 消息中的 `content` → 按 `tool_call_id` 匹配，回填 `{"result_bytes": len, "status": "ok"|"error"}`
- 若匹配失败（某些 provider 不返回 `tool_call_id`），fallback 到 `tool_trace[-1]` 追加（仅适用于串行调用，并行场景下可能错位）

该 trace 返回给父 agent 但不包含工具的完整原始输出，避免污染上下文。

### 5.4 Credential Pool 共享

`_resolve_child_credential_pool()` 逻辑：

| 场景 | 行为 |
|------|------|
| `effective_provider` 为 None | 返回父 agent 的 `_credential_pool` |
| 子 provider == 父 provider | **共享**父 pool，保证 cooldown 和 rotation 同步 |
| 子 provider != 父 provider | 尝试 `load_pool(effective_provider)`，加载独立池 |
| load_pool 失败或空 pool | 返回 None，子 agent 回退到固定 credential |

`_run_single_child()` 中通过 `acquire_lease()` / `release_lease()` 管理，允许子 agent 在 rate limit 时自动轮换 key。

### 5.5 Interrupt 传播到子 Agent

`AIAgent.interrupt()` (run_agent.py:2897-2912) 中：

```python
# 设置本 agent 的中断标志
self._interrupt_requested = True
_set_interrupt(True, self._execution_thread_id)

# 传播到子 agent
with self._active_children_lock:
    children_copy = list(self._active_children)
for child in children_copy:
    child.interrupt(message)
```

- `_active_children` 在 `_build_child_agent` 中注册，在 `close()` / `_run_single_child` finally 中注销。
- `threading.Lock` 保护列表操作，避免并发注册/注销的 race condition。

---

## 6. 关键代码位置速查

| 概念 | 文件 | 行号 |
|------|------|------|
| `delegate_task()` 主入口 | `tools/delegate_tool.py` | 623-813 |
| `_build_child_agent()` | `tools/delegate_tool.py` | 238-397 |
| `_run_single_child()` | `tools/delegate_tool.py` | 399-622 |
| `_build_child_progress_callback()` | `tools/delegate_tool.py` | 158-235 |
| `_heartbeat_loop` | `tools/delegate_tool.py` | 439-466 |
| `_resolve_child_credential_pool()` | `tools/delegate_tool.py` | 816-845 |
| `MAX_DEPTH` 限制 | `tools/delegate_tool.py` | 53, 645-653 |
| `_get_max_concurrent_children()` | `tools/delegate_tool.py` | 56-79 |
| `_cap_delegate_task_calls()` | `run_agent.py` | 3348-3376 |
| `IterationBudget` 类 | `run_agent.py` | 170-212 |
| `_delegate_depth` 初始化 | `run_agent.py` | 752 |
| `_budget_grace_call` | `run_agent.py` | 792, 8001, 8020-8026 |
| `interrupt()` 传播 | `run_agent.py` | 2897-2912 |
| `_touch_activity()` | `run_agent.py` | 2920-2923 |
| `get_activity_summary()` | `run_agent.py` | 2948-2964 |
| `_handle_max_iterations()` | `run_agent.py` | 7528-7677 |
| 并发路径 delegate 调用 | `run_agent.py` | 6917-6926 |
| 顺序路径 delegate 调用 | `run_agent.py` | 7285-7317 |
| `KawaiiSpinner.print_above()` | `agent/display.py` | 720-736 |
| `_last_resolved_tool_names` 全局 | `model_tools.py` | 197, 350-351 |

---

## 7. 风险点清单

| # | 风险 | 影响 | 缓解/备注 |
|---|------|------|----------|
| 1 | `_last_resolved_tool_names` 全局状态在并发构建或 C 扩展 segfault 场景下可能永久损坏 | 父 agent execute_code 工具列表错误 | 双重 try/finally 恢复；未来若并发构建需加锁 |
| 2 | `_strip_blocked_tools()` 与 `DELEGATE_BLOCKED_TOOLS` 不同步 | 新增危险工具集时子 agent 可能获取不应有的工具 | 需手动维护两份黑名单 |
| 3 | Batch 超限直接 error（非自动降速/分片） | 模型生成大量并行子任务时直接失败 | 需调用方自行拆分 |
| 4 | 并发截断两层防护行为不一致（error vs silent truncation） | 模型误判某些 task 已被调度 | 设计如此，需模型理解 |
| 5 | `spinner.print_above()` 非原子操作，batch 多线程竞争可能导致输出交错 | UX 层面暂时性乱序 | 并发数上限 3，影响可忽略 |
| 6 | Gateway batch flush 牺牲实时性，用户数分钟内看不到子 agent 进度 | 用户体验不一致（CLI 实时 vs Gateway 汇总） | 批量避免 gateway rate limit，设计权衡 |
| 7 | `child.close()` 异常被静默吞掉 | terminal session / browser daemon 可能泄漏 | logger.debug 记录，不抛异常 |
| 8 | 共享 credential pool 在高并发 delegation 下可能导致 slot 竞争 | 父 agent rate limit 时无可用 credential | 设计上接受，子 agent 通常运行时间短 |
| 9 | `_budget_exhausted_injected` 是死代码 | 维护者混淆 | 建议清理 |
| 10 | `_handle_max_iterations()` 的 summary call **不计入 iteration_budget** | 提供一次额外免费调用 | 硬编码 1 次且不带 tools，风险可控 |

---

## 8. 测试覆盖矩阵

| 测试文件 | 覆盖内容 | 关键测试类 |
|---------|---------|-----------|
| `tests/tools/test_delegate.py` (1282 行) | 单任务/批处理、深度限制、toolset strip、全局保存/恢复、tool trace、heartbeat、credential pool、reasoning effort | `TestDelegateTask`, `TestToolNamePreservation`, `TestDelegateObservability`, `TestDelegateHeartbeat`, `TestChildCredentialPoolResolution` |
| `tests/agent/test_subagent_progress.py` (374 行) | `print_above`、progress callback 构建、batch flush、thinking callback | `TestBuildChildProgressCallback`, `TestBatchFlush`, `TestThinkingCallback` |
| `tests/run_agent/test_real_interrupt_subagent.py` | 真实 AIAgent + mock HTTP，interrupt 传播到子 agent | — |
| `tests/run_agent/test_interactive_interrupt.py` | 交互式 interrupt，`_active_children` 列表验证 | — |
| `tests/cli/test_cli_interrupt_subagent.py` | CLI 端到端：parent → delegate → interrupt propagation | — |
| `tests/run_agent/test_agent_guardrails.py` | `_cap_delegate_task_calls()` 并发截断 | — |
| `tests/agent/test_credential_pool.py` | `acquire_lease` / `release_lease` | — |

---

## 9. 设计决策一句话总结

- **预算独立**: 子 agent 拥有独立 iteration budget，父 agent 阻塞等待但不耗额度。
- **主线程构建**: AIAgent 构造涉及全局状态修改，被约束在主线程完成，只有 run_conversation 进线程池。
- **双重重保险**: 对 `_last_resolved_tool_names` 采用"build finally + run finally"两次恢复，兼容现有全局变量设计债。
- **静默子 agent**: `quiet_mode=True` + `skip_memory=True` + `skip_context_files=True`，子 agent 不执行 context 文件注入、不向 MEMORY.md 写入、不打印 banner。
- **Grace Call**: 避免中间压力警告导致模型提前放弃（#7915），仅在 budget 真正耗尽时给予一次最终输出机会。
