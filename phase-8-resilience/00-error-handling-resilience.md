# Phase 8 — Error Handling & Resilience Patterns

Hermes 作为一个面向生产环境的 Agent 运行时，其错误处理体系分布在三个层面：
1. **API 传输层** (`agent/retry_utils.py`、`agent/errors.py`)：backoff、provider 切换、错误分类
2. **Plugin 执行层** (`plugin_manager.py`)：隔离 hook 回调，防止插件崩溃拖垮主循环
3. **生命周期层** (`agent.py`、`gateway/run.py`)：优雅关闭、资源泄漏防护、终止路径

---

## §1 Retry 策略全景 (8.1)

### 1.1 Jittered Backoff 工具 (`agent/retry_utils.py`)

核心函数 `jittered_backoff` 使用**单调递增 counter + 随机 jitter** 实现延迟增长：

```python
def jittered_backoff(
    base_delay: float = 5.0,    # 基础等待秒数
    max_delay: float = 120.0,   # 上限 2 分钟
    jitter_ratio: float = 0.5   # 抖动幅度 ±50%
) -> Iterator[float]:
    counter = 0
    while True:
        delay = base_delay + 5 * counter      # 每次 +5s
        delay = min(delay, max_delay)          # 120s 封顶
        jitter = delay * jitter_ratio          # ±50%
        delay = random.uniform(delay - jitter, delay + jitter)
        yield max(0, delay)
        counter += 1
```

**关键特性**：
- 线程安全：counter 是闭包局部变量，无共享状态竞争
- 首次延迟约 2.5–7.5s，第 24 次后稳定在 60–180s 区间
- 与 OpenAI SDK 的 `retry_exponential` 不同——这里是**固定线性增长 + 抖动**，而非指数增长

### 1.2 API 调用循环中的 Retry (`run_agent.py`)

主对话循环 `run_conversation()` 内包装 LLM API 调用的结构：

```
try:
    while True:
        if _interrupt_requested: raise InterruptedError
        
        self._invoke_hooks("pre_api_request", ...)   # hook 可注入参数
        
        try:
            response = call_llm_api(...)             # 实际 API 调用
        except SomeAPIError:
            classify_and_handle(error)               # 见 §2
            if retriable:
                delay = next(jittered_backoff())
                sleep_with_interrupt_check(delay)    # 每 200ms 检查中断
                _touch_activity()                    # 每 30s 活跃标记
                continue
            else:
                if exhausted_retries:
                    _try_recover_primary_transport()  # 最后尝试恢复主 provider
                raise
        
        self._invoke_hooks("post_api_request", ...)   # hook 可消费响应
        
        if response_is_valid:
            break
        else:
            # 无效响应（malformed / empty）
            _try_activate_fallback()                  # 先尝试 fallback provider
            # ... retry with backoff ...
except InterruptedError:
    return {"interrupted": True, ...}
```

**Retry 分类表**：

| 场景 | 策略 | max_retries |
|------|------|-------------|
| 无效 API 响应（空/malformed） | fallback → jittered backoff | 3 |
| API Error（网络/服务异常） | 即时分类 → jittered backoff | 3 |

---

## §2 智能降级与 Provider 切换 (8.2)

### 2.1 Fallback Chain 状态机

每个 Agent 实例维护一条 `List[str]` 形式的 `_fallback_chain`，以及当前位置 `_fallback_index`：

```
_fallback_chain = ["anthropic", "openrouter", "ollama", ...]
                     ↑
                  _fallback_index
```

当主 provider 连续失败达到阈值，触发 `_try_activate_fallback()`：

1. 遍历 `_fallback_chain`，**跳过当前活跃的 provider**
2. 对于每个候选：
   - 若配置了 `base_url`，则使用 `HTTPTransport(base_url)` 而非默认 OpenAI client
   - 若 hint 包含 `OLLAMA_API_KEY`，临时指定 `api_key`（Ollama Cloud 场景）
   - 推导 `api_mode`：`chat_completions` / `codex_responses` / `anthropic_messages`
   - 更新 `_custom_headers`（保留用户注入的 header，如 Kimi Coding 的 `User-Agent: Kimi-Sentinel`）
   - 重建 `AsyncOpenAI` client，清除 HTTPX 的 stale connection pool
3. `_fallback_index += 1`；若遍历完毕仍未成功，标记 `retry_exhausted`

### 2.2 降级路径矩阵

| 错误信号 | 降级动作 | 备注 |
|----------|----------|------|
| 429 Too Many Requests | **Eager fallback** | 若 credential pool 未耗尽，立即切 provider |
| 413 Payload Too Large | Context **Compression**（最多 3 次） | 压缩历史后重试，仍 413 则 fallback |
| Context Overflow（context length exceeded） | 下调 `context_length` tier + compression | 逐级降档（256k → 200k → 128k ...） |
| 524 / 504 / 500 / 502 / 503 / 529 | 按错误码分类提示用户 | 无自动修复，依赖 retry + fallback |
| Anthropic long-context tier 溢出 | 限制到 **200k** + compression | Anthropic API 的硬性边界 |
| `max_tokens` 过大 | 临时下调 `_ephemeral_max_output_tokens` | 不影响全局配置，仅本次调用 |

### 2.3 Prompt Caching 兼容性重评估

切换 provider 后，`_model` 前缀可能变更（例如从 `anthropic/claude-4-sonnet` 切到直接 Anthropic endpoint）。此时：
- OpenRouter + Claude 组合 → 开启 **native prompt caching**
- 原生 Anthropic endpoint → 同样需要重新 parsing `_model_name`，清除旧 cache prefix
- `_max_context_length` 从新 provider 的 `model_info` 重新查询

### 2.4 主 Provider 恢复机制

在 max_retries 全部耗尽后，Agent 会执行一次**额外的主 provider 恢复尝试**：

```python
_try_recover_primary_transport()
```

- 仅针对**直接 endpoint**（跳过 OpenRouter / Nous 代理层）
- 重建 client 实例，强制清理底层 TCP 连接池
- 逻辑上属于"最后的稻草"，若仍失败则本次对话彻底终止并返回错误摘要

---

## §3 Plugin 隔离机制 (8.3)

### 3.1 Context Engine 互斥注册

`PluginContext` 的 `register_context_engine()` 确保**全局只有一个 context engine**：

```python
def register_context_engine(self, engine):
    if self._manager._context_engine is not None:
        self._logger.warning("Context engine already registered")
        return
    if not isinstance(engine, ContextEngine):
        self._logger.warning("...must be instance of ContextEngine")
        return
    self._manager._context_engine = engine
```

- 拒绝重复注册，防止多个插件竞争改写上下文栈
- 类型检查确保 engine 是 `ContextEngine` 实例（而非任意 dict/list）

### 3.2 Hook 回调独立 try/except

`PluginManager.invoke_hook()` 的隔离设计是 Hermes 插件系统的核心安全边界：

```python
def invoke_hook(self, hook_name, *args, **kwargs):
    results = []
    for callback in self._hooks.get(hook_name, []):
        try:
            result = callback(*args, **kwargs)
            if result is not None:
                results.append(result)
        except Exception as e:
            # 关键：一个插件的异常不影响其他插件
            self._logger.error(f"Hook {hook_name} failed in {callback}: {e}")
    return results
```

**设计原则**：`misbehaving plugin cannot break the core agent loop`——单个插件崩溃仅影响自身，不会导致 Agent 对话异常终止。

### 3.3 与 Gateway Hook 的对比

| 维度 | CLI Plugins (`plugin_manager.py`) | Gateway Hooks (`gateway/hooks.py`) |
|------|-----------------------------------|-----------------------------------|
| 执行模式 | Sync，return-value-collecting | Async，fire-and-forget |
| 异常处理 | 每个 callback 独立 try/except | 依赖 asyncio gather / try |
| 返回值 | 收集为 `List[Any]` 供调用方消费 | 通常无返回值 |
| 用途 | `pre_llm_call` 上下文注入、tool 注册 | 事件通知、消息发送、心跳 |

---

## §4 优雅终止与 Nuclear Option (8.4)

### 4.1 搜索结论：当前版本**不存在显式 Nuclear Option**

在 v0.9.0 全代码库中，未发现以下模式的 hard-kill 实现：
- `os._exit()` / `sys.exit()` 在 Agent 主循环中强制退出
- `SIGKILL` 信号发送给子进程或自身
- `nuclear_kill_agent`、`force_terminate` 等命名的函数
- `threading` 的 `Thread._stop()` 或 C 层 pthread_cancel

Hermes 采用的是**全链路优雅终止（graceful shutdown）**策略。

### 4.2 Agent 层面：`AIAgent.close()`

Agent 关闭执行 5 步资源释放，**每一步都有独立的 try/except**，避免某一步失败导致后续清理被跳过：

```python
async def close(self):
    # 1. 终止所有 background process
    try: self._kill_all_bg_processes()
    except: ...
    
    # 2. 清理 terminal VM (sandbox)
    try: self._cleanup_terminal_vm()
    except: ...
    
    # 3. 清理 browser session
    try: self._cleanup_browser()
    except: ...
    
    # 4. 递归关闭所有 child agents
    try: await self._close_child_agents()
    except: ...
    
    # 5. 关闭 OpenAI client (HTTPX connection pool)
    try: await self.client.aclose()
    except: ...
```

### 4.3 Memory Provider 层面：`shutdown_memory_provider()`

```python
async def shutdown_memory_provider(self):
    if self.memory_manager:
        await self.memory_manager.on_session_end()   # session 收尾
        await self.memory_manager.shutdown_all()      # 持久化/关闭 backend
    if self.context_compressor:
        await self.context_compressor.on_session_end() # compressor 状态落地
```

### 4.4 中断处理：从信号到异常

用户按下 Ctrl-C 或平台发送中断信号时：

1. `_interrupt_requested` 原子 flag 置 `True`
2. API 调用循环中**每 200ms 检查**该 flag
3. 一旦检测到：
   - 关闭 Anthropic client（若使用原生 streaming）
   - 关闭 OpenAI/HTTPX request client
   - 抛出 `InterruptedError`
4. 外层 `run_conversation()` catch `InterruptedError` → 返回 `{"interrupted": True, ...}`
5. Gateway 会话层收到 `interrupted` 后，调用 `agent.shutdown_memory_provider()`

```
User Interrupt / SIGINT
    └── set _interrupt_requested = True
            └── LLM API 调用中检测
                    ├── close Anthropic streaming client
                    ├── close OpenAI request
                    └── raise InterruptedError
                            └── run_conversation() catch
                                    └── return {"interrupted": True}
                                            └── gateway/run.py
                                                    └── shutdown_memory_provider()
```

### 4.5 Streaming 中断的 partial 恢复

若 streaming 已被打断（已交付部分 token），Agent 不会丢弃已有内容，而是使用 `_current_streamed_assistant_text` 构建一个 stub response，确保对话历史完整：

```python
# streaming 中断后的 fallback
final_response = self._current_streamed_assistant_text
```

### 4.6 TCP Socket 强制清理

针对 HTTPX/OpenAI client 可能出现的 `CLOSE-WAIT` 堆积问题，Hermes 提供了 `_force_close_tcp_sockets()`：

```python
def _force_close_tcp_sockets(self):
    # 遍历 HTTPX 底层 connection pool 的所有连接
    for conn in self._get_active_connections():
        try:
            sock = conn.socket
            sock.shutdown(socket.SHUT_RDWR)   # 双向关闭
            sock.close()                       # 释放 FD
        except (OSError, AttributeError):
            pass
```

- 用于 provider 切换后旧 client 的连接残留
- 防止大量并发 retry 后系统 FD 耗尽

### 4.7 CLI 与 Gateway 的终止差异

| 环境 | 终止路径 | 资源回收 |
|------|----------|----------|
| CLI (`cli.py`) | `KeyboardInterrupt` → `close()` → 进程退出 | 同步单线程，较简单 |
| Gateway (`gateway/run.py`) | `session_expiring` → `shutdown_memory_provider()` → asyncio 清理 | 多会话并发，需逐个回收 |

---

## 总结：Resilience 设计哲学

Hermes 的错误处理体现了一种**"防御纵深"（defense in depth）**的工程哲学：

1. **传输层**：Retry + Jitter + Provider Fallback 三重冗余保证对话不挂
2. **插件层**：单个插件崩溃被独立捕获，主循环不受影响
3. **生命周期层**：没有一键核爆，只有逐级优雅关闭，确保 memory/session/tool state 不丢失
4. **资源层**：TCP socket 强制清理、HTTPX connection pool 重建，防止基础设施泄漏拖垮长期运行的 Gateway

> **注意**：没有显式的 nuclear option 并不意味着 Hermes 缺乏紧急制动能力。通过 `_interrupt_requested` flag + 逐级 `close()` + `shutdown_memory_provider()`，用户和平台可以在任何时刻安全地终止 Agent，且不会留下孤儿进程或 stale memory 状态。
