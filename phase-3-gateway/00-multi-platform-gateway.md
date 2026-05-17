# Phase 3 — Multi-Platform Gateway 与 Batch Runner 架构

> **范围**: Hermes Harness v0.9.0 的 Gateway 运行时、多平台适配层、Session 隔离、Hook/Plugin 双系统、以及非交互式 Batch Runner。
>
> **源文件**: `gateway/run.py` (~9005 行), `gateway/session.py` (~1087 行), `gateway/session_context.py`, `gateway/delivery.py`, `gateway/stream_consumer.py`, `gateway/hooks.py`, `gateway/platforms/base.py`, `gateway/platforms/telegram.py`, `gateway/platforms/discord.py`, `gateway/platforms/sms.py`, `hermes_cli/plugins.py`, `batch_runner.py`。
>
> **核心概念**: Gateway 不是集中式 Web Server，而是纯 asyncio 长生命周期控制器；每个平台自带传输层（长轮询 / WebSocket / HTTP webhook）。事件路由、Session 隔离、流式桥接、投递抽象均围绕这一前提展开。

---

## 1. Gateway 目录结构与核心模块

| 文件/目录 | 行数/体积 | 职责概述 |
|---|---|---|
| `gateway/run.py` | ~9,005 行 / 415 KB | 主入口与 `GatewayRunner` 生命周期。负责环境初始化、平台适配器启停、消息处理主循环（`_handle_message`）、授权、命令拦截、Agent 调用、流式消费、工具进度回显。 |
| `gateway/session.py` | ~1,087 行 / 41 KB | Session 存储与隔离。`SessionStore`、`SessionEntry`、`SessionSource`、会话键构造、自动重置策略、SQLite + JSONL 双写。 |
| `gateway/session_context.py` | ~100+ 行 | 会话上下文构建器 `build_session_context()`，用于把平台来源信息注入 Agent 的 system prompt。 |
| `gateway/delivery.py` | ~256 行 / 8.5 KB | 投递路由器 `DeliveryRouter`。解析 `origin/local/telegram:chat_id` 等投递目标，负责 Cron 输出与 Agent 响应的本地/平台分发。 |
| `gateway/stream_consumer.py` | ~585 行 / 26 KB | 流式消费桥 `GatewayStreamConsumer`。把 Agent 工作线程中的**同步**流式 token 回调，桥接到**异步**的消息编辑/发送。 |
| `gateway/hooks.py` | ~150 行 / 6.4 KB | Gateway Event Hook 注册与发射。事件驱动、异步、fire-and-forget。 |
| `gateway/platforms/base.py` | ~2,071 行 | `BasePlatformAdapter` 抽象基类。定义 `connect/send/edit_message/send_typing` 等统一接口，以及代理、UTF-16 截断等通用工具。 |
| `gateway/platforms/api_server.py` | ~1,904 行 | OpenAI-compatible HTTP API 适配器（**aiohttp**，非 FastAPI），默认监听 `127.0.0.1:8642`。 |
| `gateway/platforms/*.py` | 多个 | 各平台具体实现：TG、Discord、Slack、WhatsApp、Signal、Email、SMS、Mattermost、Feishu、DingTalk、Matrix、HomeAssistant、WebHook、BlueBubbles、WeCom、WeiXin 等。 |

---

## 2. 事件路由与传输层架构（关键纠正）

### 2.1 无 FastAPI 集中路由

**源码核实**：
- `gateway/run.py` 全文件搜索 `FastAPI`、`APIRouter`、`app = FastAPI`、`uvicorn`：**零命中**。
- `gateway/` 全目录搜索 `FastAPI(`：**零命中**。
- FastAPI 依赖仅出现在：
  - `hermes_cli/web_server.py`（独立 Web UI 后端，监听 `:9119`，与 Gateway 运行时无关）。
  - `pyproject.toml` 的 `extras = ["web", "rl"]` 中。

> **结论**：`run.py` 不是 Web Server。Gateway 核心没有集中式 HTTP / WebSocket Server，也不使用 FastAPI 做路由。

### 2.2 实际的“路由”机制：事件驱动 + Adapter 自托管

Gateway 的“事件路由”由 `GatewayRunner` 的消息处理管道 + 各 `PlatformAdapter` 的独立传输组成：

**启动时注册**（`run.py:1566-1581`）：
```python
for platform, platform_config in self.config.platforms.items():
    if not platform_config.enabled:
        continue
    adapter = self._create_adapter(platform, platform_config)
    if not adapter:
        continue
    adapter.set_message_handler(self._handle_message)
    adapter.set_fatal_error_handler(self._handle_adapter_fatal_error)
    adapter.set_session_store(self.session_store)
    adapter.set_busy_session_handler(self._handle_active_session_busy_message)
    success = await adapter.connect()   # 各 adapter 内部启动自己的 client/server
    if success:
        self.adapters[platform] = adapter
```

每个适配器在 `connect()` 内部自行建立与平台的链接：
- **Telegram / WhatsApp / Discord / Slack / DingTalk / Matrix**：使用各自 SDK 的长轮询或 Gateway 连接。
- **Mattermost / Home Assistant**：使用 `aiohttp.ClientWebSocketResponse` 作为 **客户端** WebSocket。
- **Feishu**：可选 `websockets` 库的长连接模式。
- **api_server / webhook / sms / bluebubbles / wecom_callback**：使用 `aiohttp.web.Application()` 启动 **服务端** HTTP 端口（但这不是 FastAPI，也不是 `run.py` 的一部分）。

**入站消息流向**：
```
平台事件 → PlatformAdapter.connect() 内部循环
              → adapter 解析原生事件并封装为 MessageEvent
              → adapter.message_handler(event)  # 即 GatewayRunner._handle_message(event)
                → 授权检查 / 命令拦截 / 会话恢复
                → AIAgent.run_conversation()
                → 流式/最终响应通过 adapter.send() 回写到平台
```

### 2.3 适配器工厂：硬编码映射而非插件发现

`GatewayRunner._create_adapter()`（`run.py:2111-2258`）是一个巨型条件分支，根据 `Platform` 枚举硬编码导入对应适配器类：

```python
# run.py:2127-2132
if platform == Platform.TELEGRAM:
    from gateway.platforms.telegram import TelegramAdapter, check_telegram_requirements
    if not check_telegram_requirements():
        return None
    return TelegramAdapter(config)
elif platform == Platform.DISCORD:
    from gateway.platforms.discord import DiscordAdapter, check_discord_requirements
    ...
elif platform == Platform.API_SERVER:
    from gateway.platforms.api_server import APIServerAdapter, check_api_server_requirements
    return APIServerAdapter(config)
```

映射覆盖：TELEGRAM、DISCORD、WHATSAPP、SLACK、SIGNAL、HOMEASSISTANT、EMAIL、SMS、DINGTALK、FEISHU、WECOM_CALLBACK、WECOM、WEIXIN、MATTERMOST、MATRIX、API_SERVER、WEBHOOK、BLUEBUBBLES。

### 2.4 适配器重连与故障隔离

`run.py` 维护 `self._failed_platforms: Dict[Platform, dict]`。启动失败时以 30 秒为初始间隔排队重试，不会导致整个 Gateway 退出。

---

## 3. Session 隔离机制

### 3.1 核心数据结构

**`SessionSource`**（`session.py` 前部）：描述消息来源的平台、用户、聊天、线程信息。

**`SessionEntry`**（`session.py:328-375`）：单条会话持久化记录：
```python
@dataclass
class SessionEntry:
    session_key: str
    session_id: str
    created_at: datetime
    updated_at: datetime
    origin: Optional[SessionSource] = None
    display_name: Optional[str] = None
    platform: Optional[Platform] = None
    chat_type: str = "dm"
    input_tokens: int = 0
    output_tokens: int = 0
    ...
    memory_flushed: bool = False
    suspended: bool = False          # /stop 后挂起，防止死循环恢复
```

### 3.2 会话键构造：极端细粒度的隔离

`build_session_key()`（`session.py:436-492`）是隔离的“单一事实来源”：

- **DM 私聊**：
  - `agent:main:{platform}:dm:{chat_id}:{thread_id}`（有 chat_id + thread_id）
  - `agent:main:{platform}:dm:{chat_id}`（有 chat_id）
  - `agent:main:{platform}:dm:{thread_id}`（无 chat_id，以 thread_id 兜底）
  - `agent:main:{platform}:dm`（无任何标识时共享一个 session）

- **Group / Channel**：
  - 基础：`agent:main:{platform}:{chat_type}:{chat_id}:{thread_id}`
  - 当 `group_sessions_per_user=True`（默认）且**非 thread** 时，追加 `user_id`，实现群内每人独立会话。
  - 当 `thread_sessions_per_user=False`（默认）且**有 thread_id** 时，**不追加 user_id**，实现**同一 thread 内多人共享上下文**（如 Telegram 论坛主题、Discord/Slack 线程）。

### 3.3 SessionStore 持久化与生命周期

`SessionStore`（`session.py:495`）提供：

- **双写存储**：优先 `SQLite`（`hermes_state.SessionDB`），降级到 `sessions.json` + `.jsonl` 传统文件。
- **自动重置策略**（`get_or_create_session` 内部调用 `_should_reset`）：
  - `idle`：超过 `idle_minutes` 无活动则重置。
  - `daily`：在每天的指定整点（`at_hour`）重置。
  - `both`：二者同时生效。
  - 有后台进程活跃时会话**不会被重置**（`has_active_processes_fn` 检查）。

- **启动时 session 保护**（`run.py:1536-1558`）：
  - 若上次不是干净退出（无 `.clean_shutdown` 标记），`SessionStore.suspend_recently_active()` 会挂起最近 120 秒内活跃的 session，防止网关崩溃重启后陷入死循环。

- **挂起与恢复**：
  - `suspend_session(session_key)`（`session.py:786`）标记 `suspended=True`，下次 `get_or_create_session()` 遇到该标记会**强制生成新 session_id**，实现“干净开始”。

- **代码片段**：
```python
# session.py:683-768
def get_or_create_session(self, source: SessionSource, force_new: bool = False) -> SessionEntry:
    session_key = self._generate_session_key(source)
    ...
    if entry.suspended:
        reset_reason = "suspended"
    else:
        reset_reason = self._should_reset(entry, source)
    if not reset_reason:
        entry.updated_at = now
        self._save()
        return entry
    else:
        # 创建新的 session_id，并在 SQLite 中 end_session 旧会话
        session_id = f"{now.strftime('%Y%m%d_%H%M%S')}_{uuid.uuid4().hex[:8]}"
        entry = SessionEntry(session_key=..., session_id=session_id, ...)
```

---

## 4. Stream Consumer 与 Delivery 抽象

### 4.1 GatewayStreamConsumer：Sync → Async 桥接

**设计目标**：Agent 内部在**同步线程池**中运行 LLM 推理，流式 token 通过同步回调 `stream_delta_callback(text)` 输出。而平台 API（Telegram/Discord 等）是**异步**的。`GatewayStreamConsumer` 负责这一“sync → async”桥接，并在过程中实现**单条消息渐进编辑**。

**核心类**（`stream_consumer.py:48`）：
```python
class GatewayStreamConsumer:
    def __init__(self, adapter, chat_id, config, metadata=None):
        self._queue: queue.Queue = queue.Queue()   # 线程安全队列
        self._accumulated = ""
        self._message_id: Optional[str] = None
        self._edit_supported = True
        ...
```

**同步侧**（Agent 线程安全调用）：
- `on_delta(text: str)`：把 token 放入 `queue.Queue`。
- `on_segment_break()`：放入 `_NEW_SEGMENT` 标记，触发“截断当前消息、新起一条”。
- `on_commentary(text: str)`：放入 `_COMMENTARY` 标记及文本，用于工具间插话（"I'll inspect the repo first."）。
- `finish()`：放入 `_DONE` 标记。

**异步侧**（`asyncio` Task）：
- `run()` 从队列中消费，累积文本，按 `edit_interval` 节奏调用 `_send_or_edit()`。
- 首次发送：`adapter.send(...)`，拿到 `message_id`。
- 后续更新：`adapter.edit_message(chat_id, message_id, new_text + cursor)`。

**平台差异处理**：
- **编辑能力探测**：若平台不支持编辑（如 BlueBubbles/iMessage），`type(adapter).edit_message is BasePlatformAdapter.edit_message` 为真，则跳过工具进度编辑、流式消费直接走 fallback 发送新消息。
- **光标处理**：默认 `cursor = " ▉"`。若平台不支持编辑（`SUPPORTS_MESSAGE_EDITING = False`），则在 `run.py:7825` 把 cursor 设为空串，防止光标永久卡在消息末尾。
- **洪水控制自适应退避**：
  - `_MAX_FLOOD_STRIKES = 3`。
  - 每次编辑遭遇 flood control，`_current_edit_interval` 倍增（上限 10s）。
  - 超过 3 次后进入 `fallback` 模式：停止编辑，最终通过 `adapter.send()` 发送剩余尾部文本。

**在 run.py 中的装配点**（`run.py:7813-7841`）：
```python
from gateway.stream_consumer import GatewayStreamConsumer, StreamConsumerConfig
_adapter = self.adapters.get(source.platform)
_adapter_supports_edit = getattr(_adapter, "SUPPORTS_MESSAGE_EDITING", True)
_effective_cursor = _scfg.cursor if _adapter_supports_edit else ""
_consumer_cfg = StreamConsumerConfig(
    edit_interval=_scfg.edit_interval,
    buffer_threshold=_scfg.buffer_threshold,
    cursor=_effective_cursor,
)
_stream_consumer = GatewayStreamConsumer(
    adapter=_adapter,
    chat_id=source.chat_id,
    config=_consumer_cfg,
    metadata={"thread_id": _progress_thread_id} if _progress_thread_id else None,
)
_stream_delta_cb = _stream_consumer.on_delta
stream_consumer_holder[0] = _stream_consumer

# 随后绑定到 agent
agent.stream_delta_callback = _stream_delta_cb
agent.interim_assistant_callback = _interim_assistant_cb
```

### 4.2 DeliveryRouter：多态投递目标解析

**`DeliveryTarget.parse()`**（`delivery.py:46-92`）支持：
- `"origin"` → 回到消息来源 chat/thread。
- `"local"` → 仅保存到本地文件。
- `"telegram"` → 该平台的 `home channel`。
- `"telegram:123456"` → 指定 chat_id。
- `"telegram:123456:789"` → 指定 chat_id + thread_id。

**`DeliveryRouter`**（`delivery.py:107`）在 `GatewayRunner.__init__` 中创建，在 `start()` 时注入 adapters：
```python
# run.py:1682-1683
self.delivery_router = DeliveryRouter(self.config)
...
self.delivery_router.adapters = self.adapters
```

**`deliver()` 方法**：遍历 targets，区分 `Platform.LOCAL`（调用 `_deliver_local` 保存 Markdown 文件到 `~/.hermes/cron/output/`）和平台投递（`_deliver_to_platform`）。

**`_deliver_to_platform()`**（`delivery.py:224-252`）处理：
- **超长截断**：若内容超过 `MAX_PLATFORM_OUTPUT = 4000` 字符，截断到 `TRUNCATED_VISIBLE = 3800`，并把完整内容写入 `~/.hermes/cron/output/{job_id}_{timestamp}.txt`，在消息尾部附加文件路径提示。
- **Thread 透传**：若 `target.thread_id` 存在，自动写入 `send_metadata["thread_id"]`，保持线程回复语义。
- **最终调用**：`adapter.send(target.chat_id, content, metadata=send_metadata)`。

---

## 5. Gateway Event Hook vs CLI Plugin 双系统

### 5.1 Gateway Event Hook 系统

**核心类**: `HookRegistry`（`gateway/hooks.py` L34-L136）

**注册机制**:
- `discover_and_load()` 先调用 `_register_builtin_hooks()` 注册内建钩子，再扫描 `~/.hermes/hooks/<name>/`。
- 每个用户钩子目录必须包含 `HOOK.yaml`（元数据，至少含 `name` 与 `events`）和 `handler.py`（顶层必须定义 `handle` 函数）。
- 动态加载：`importlib.util.spec_from_file_location(...)` + `exec_module(module)`，然后 `getattr(module, "handle")`。

**支持的事件类型**（L9-L17）：

| 事件 | 说明 |
|------|------|
| `gateway:startup` | Gateway 进程启动 |
| `session:start` | 新会话创建（该会话第一条消息） |
| `session:end` | 会话结束（用户执行 `/new` 或 `/reset`） |
| `session:reset` | 会话重置完成（新 session entry 已创建） |
| `agent:start` | Agent 开始处理消息 |
| `agent:step` | 工具调用循环中的每一次迭代 |
| `agent:end` | Agent 处理结束 |
| `command:*` | 通配符匹配，任意 slash command 执行时触发 |

**执行模型**（`emit()`，L138-L170）：
- **`async def emit(...)`**：完全异步，支持 sync 与 async handler（通过 `asyncio.iscoroutine` 检测）。
- **顺序执行**：`for fn in handlers` 串行调用，无优先级/权重机制。
- **错误隔离**：每个 handler 独立 try/except，失败不阻塞主流程。
- **无返回值**：fire-and-forget。
- **通配符支持**：`base:*` 形式（如 `command:*`）。

**Gateway 运行时调用点**（`run.py` 中的关键位置）：

| 调用位置 | 行号 | 事件 | 上下文内容 |
|----------|------|------|------------|
| `_run_gateway()` | 1692 | `gateway:startup` | `{"platforms": [...]}` |
| `_dispatch_message()` | 3116 | `session:start` | `platform`, `user_id`, `session_id`, `session_key` |
| `_dispatch_message()` | 3556 | `agent:start` | `platform`, `user_id`, `session_id`, `message`（截断 500 字符） |
| `_run_agent()` → `_step_callback_sync` | 7701 | `agent:step` | `platform`, `user_id`, `session_id`, `iteration`, `tool_names` |
| `_dispatch_message()` | 3645 | `agent:end` | `platform`, `user_id`, `session_id`, `message`, `response`, `api_calls`, `duration` |
| `_handle_command()` | 2658 | `command:{command}` | `platform`, `user_id`, `chat_id`, `command`, `args` |
| `_handle_reset()` | 4002 | `session:end` | `platform`, `user_id`, `session_key` |
| `_handle_reset()` | 4009 | `session:reset` | 同上 |

注意：`agent:step` 是从同步回调 `_step_callback_sync` 通过 `asyncio.run_coroutine_threadsafe` 桥接到异步事件循环的（L7700-L7710），因为 Agent 核心运行在自己的线程/执行模型中。

**数据契约**：所有 handler 统一签名 `handle(event_type: str, context: dict) -> None`。`context` 为事件相关字典，无强类型 schema。

### 5.2 CLI Plugin 系统

**核心类**: `PluginManager`（`hermes_cli/plugins.py` L271）、`PluginContext`（L124）、`PluginManifest`（L93）、`LoadedPlugin`（L108）。

**加载入口** `discover_and_load()`（L287）扫描三类来源：
1. **User plugins**：`~/.hermes/plugins/<name>/`
2. **Project plugins**：`./.hermes/plugins/<name>/`（需 `HERMES_ENABLE_PROJECT_PLUGINS` 环境变量开启）
3. **Pip plugins**：通过 `importlib.metadata` 读取 `hermes_agent.plugins` entry-point group（L371-L394）

目录插件要求 `plugin.yaml`（或 `plugin.yml`）manifest 和 `__init__.py` 中定义 `register(ctx)` 函数。

**暴露的生命周期钩子**（`VALID_HOOKS`，L55-L66）：
```python
VALID_HOOKS: Set[str] = {
    "pre_tool_call",
    "post_tool_call",
    "pre_llm_call",
    "post_llm_call",
    "pre_api_request",
    "post_api_request",
    "on_session_start",
    "on_session_end",
    "on_session_finalize",
    "on_session_reset",
}
```

对比 Gateway Event Hooks，CLI Plugin 的钩子更偏向 **Agent 内部执行细节**：工具调用前后、LLM 调用前后、API 请求前后。

**执行模型**（`invoke_hook()`，L500-L534）：
```python
def invoke_hook(self, hook_name: str, **kwargs: Any) -> List[Any]:
    callbacks = self._hooks.get(hook_name, [])
    results: List[Any] = []
    for cb in callbacks:
        try:
            ret = cb(**kwargs)
            if ret is not None:
                results.append(ret)
        except Exception as exc:
            logger.warning("Hook '%s' callback %s raised: %s", ...)
    return results
```

关键特征：
- **同步执行**：`invoke_hook` 是普通函数，非 async。
- **收集返回值**：返回所有非 `None` 结果的 `List[Any]`。`pre_llm_call` 的返回值可用于向当前轮次的 user message 注入临时上下文（L508-L518 注释）。
- **错误隔离**：每个 callback 独立 try/except。

**Agent 核心调用点**：

| 调用位置 | 文件 | 行号 | 钩子 | 说明 |
|----------|------|------|------|------|
| `run_conversation()` | `run_agent.py` | 7857 | `on_session_start` | 全新会话首次构建 system prompt 时触发一次 |
| `run_conversation()` | `run_agent.py` | 7948 | `pre_llm_call` | 每轮工具循环前，可返回上下文注入 user message |
| `_do_api_request()` | `run_agent.py` | 8239 | `pre_api_request` | 每次 API 请求前 |
| `_do_api_request()` | `run_agent.py` | 9635 | `post_api_request` | 每次 API 请求后（含 usage、finish_reason 等） |
| `handle_function_call()` | `model_tools.py` | 502 | `pre_tool_call` | 工具调用执行前 |
| `handle_function_call()` | `model_tools.py` | 531 | `post_tool_call` | 工具调用执行后 |
| `_finalize_session()` | `cli.py` | 629 | `on_session_finalize` | CLI 模式下会话结束 |
| `_handle_reset()` | `gateway/run.py` | 3996 | `on_session_finalize` | Gateway 模式下 `/reset` 旧会话结束 |
| `_handle_reset()` | `gateway/run.py` | 4032 | `on_session_reset` | Gateway 模式下新会话创建后 |

**PluginContext 提供的扩展能力**：
- `register_tool()`：向全局 `tools.registry` 注册自定义工具。
- `inject_message()`：向活跃对话注入消息（仅 CLI 模式，依赖 `_cli_ref`）。
- `register_cli_command()`：注册 CLI 子命令（如 `hermes honcho ...`）。
- `register_context_engine()`：替换内置的 `ContextCompressor`。

### 5.3 内置钩子示例：`builtin_hooks/boot_md.py`

该内置钩子负责在 Gateway 启动时读取 `~/.hermes/BOOT.md`，若文件存在且非空，则在后台线程中启动一个一次性 Agent 会话来执行启动清单。

注册方式：在 `HookRegistry._register_builtin_hooks()`（L54-L67）中硬编码注册：
```python
def _register_builtin_hooks(self) -> None:
    try:
        from gateway.builtin_hooks.boot_md import handle as boot_md_handle
        self._handlers.setdefault("gateway:startup", []).append(boot_md_handle)
        self._loaded_hooks.append({
            "name": "boot-md",
            "description": "Run ~/.hermes/BOOT.md on gateway startup",
            "events": ["gateway:startup"],
            "path": "(builtin)",
        })
    except Exception as e:
        print(f"[hooks] Could not load built-in boot-md hook: {e}", flush=True)
```

Handler 实现（`builtin_hooks/boot_md.py` L69-L87）：
```python
async def handle(event_type: str, context: dict) -> None:
    if not BOOT_FILE.exists():
        return
    content = BOOT_FILE.read_text(encoding="utf-8").strip()
    if not content:
        return
    logger.info("Running BOOT.md (%d chars)", len(content))
    thread = threading.Thread(
        target=_run_boot_agent,
        args=(content,),
        name="boot-md",
        daemon=True,
    )
    thread.start()
```

- 符合 Gateway Hook 数据契约：`(event_type, context)`
- 使用 `async def`，因为 `emit()` 支持 await async handler
- 为了不阻塞 Gateway 启动，实际 Agent 执行放在 `daemon=True` 的后台线程中
- 若 Agent 返回 `[SILENT]` 则静默不输出（L61）

### 5.4 两套系统根本差异

| 维度 | Gateway Event Hook (`gateway/hooks.py`) | CLI Plugin (`hermes_cli/plugins.py`) |
|------|----------------------------------------|-------------------------------------|
| **设计目标** | 网关层生命周期事件通知 | Agent 核心深度扩展（工具、命令、上下文） |
| **加载来源** | `~/.hermes/hooks/<name>/` | `~/.hermes/plugins/<name>/`、`./.hermes/plugins/`、pip entry-points |
| **入口函数** | `handler.py::handle(event_type, context)` | `__init__.py::register(ctx)` |
| **元数据文件** | `HOOK.yaml`（events 列表） | `plugin.yaml`（版本、作者、requires_env 等） |
| **事件/钩子粒度** | 会话级与消息级（startup, session:start/end, agent:start/end/step, command:*） | 执行级（pre/post tool_call, pre/post llm_call, pre/post api_request, on_session_*） |
| **执行模型** | `async def emit(...)`，异步顺序执行 | `def invoke_hook(...)`，同步顺序执行 |
| **返回值** | 无（fire-and-forget） | 收集所有非 `None` 返回值并返回 `List[Any]` |
| **与主流程耦合** | 松耦合，失败不阻塞 | 中等耦合，`pre_llm_call` 返回值可改变 user message |
| **扩展能力** | 仅事件监听 | 注册工具、注册 CLI 命令、替换上下文引擎、消息注入 |
| **运行环境** | 仅在 Gateway 模式（`gateway/run.py`） | CLI 与 Gateway 共享（`run_agent.py`、`model_tools.py`） |
| **I/O 模型** | 面向异步 I/O（adapter send/typing 等） | 面向同步 Agent 循环 |
| **并发安全** | 依赖 asyncio event loop | 依赖调用方线程（通常是主线程或 Agent 线程） |
| **错误处理** | `print` 到 stdout，continue | `logger.warning`，continue |

**为什么需要两套系统？**
1. **生命周期不同**：Gateway Hooks 绑定**消息/会话生命周期**（收到消息 → Agent 处理 → 发送回复 → 会话结束），是网关层面的编排。CLI Plugins 绑定**Agent 内部执行生命周期**（LLM 调用、工具调用、API 请求），是核心层面的编排。
2. **并发模型不同**：Gateway 运行在纯 asyncio 事件循环中，Hook 需要 async 支持才能不阻塞消息收发。Agent 核心（`AIAgent.run_conversation`）是**同步循环**，`invoke_hook` 以同步方式嵌入其中，避免在同步代码中引入 async 复杂度。
3. **I/O 耦合与信任边界**：Gateway Hooks 更适合做**跨平台通知、日志、外部集成**，但无法深入修改 Agent 的 tool schemas 或注入 LLM 上下文。CLI Plugins 与 Agent 核心共享进程地址空间，可直接注册工具、修改 argument parser、替换 `ContextCompressor`，属于**高权限扩展**。

**混用现象**：在 `gateway/run.py` 中两套系统同时存在。Gateway 自己的 `HookRegistry` 负责 `session:start/end/reset`、`agent:start/end/step`、`command:*` 等事件，但在 `/reset` 处理中又显式调用了 CLI Plugin 的 `invoke_hook("on_session_finalize", ...)` 和 `invoke_hook("on_session_reset", ...)`（L3994-L4032）。这表明 `on_session_finalize` / `on_session_reset` 是**跨层共享**事件，属于设计上的职责分层，而非冗余。

---

## 6. Platform Adapter 抽象与跨平台映射

### 6.1 BasePlatformAdapter 接口

`BasePlatformAdapter`（`gateway/platforms/base.py:779`）是抽象基类（ABC），定义了所有平台适配器必须实现的统一接口：

**核心抽象方法（必须实现）**：
- `connect(self) -> bool`（行 943）：连接平台并开始接收消息。
- `disconnect(self) -> None`（行 952）：断开平台连接。
- `send(self, chat_id, content, reply_to=None, metadata=None) -> SendResult`（行 957）：发送消息到指定聊天。
- `get_chat_info(self, chat_id)`（行 1920）：获取聊天信息。

**可选方法（提供默认回退实现）**：
- `edit_message()`（行 978）：默认返回 `SendResult(success=False, error="Not supported")`。
- `send_typing()` / `stop_typing()`（行 991/1000）：默认空操作。
- `send_image()`（行 1008）：默认将图片 URL 以文本形式发送。
- `send_animation()`（行 1027）：默认回退到 `send_image`。
- `send_voice()` / `play_tts()`（行 1098/1118）：默认发送文件路径文本。
- `send_video()`（行 1132）：默认发送路径文本。

**`MessageEvent` 数据类**（行 655-716）是跨平台消息的统一归一化表示：
```python
@dataclass
class MessageEvent:
    text: str
    message_type: MessageType = MessageType.TEXT
    source: SessionSource = None
    raw_message: Any = None
    message_id: Optional[str] = None
    media_urls: List[str] = field(default_factory=list)
    media_types: List[str] = field(default_factory=list)
    reply_to_message_id: Optional[str] = None
    reply_to_text: Optional[str] = None
    auto_skill: Optional[str | list[str]] = None
    internal: bool = False
    timestamp: datetime = field(default_factory=datetime.now)
```

**`SendResult` 数据类**（行 719-726）：
```python
@dataclass
class SendResult:
    success: bool
    message_id: Optional[str] = None
    error: Optional[str] = None
    raw_response: Any = None
    retryable: bool = False  # 瞬态连接错误时基类自动重试
```

### 6.2 媒体附件处理

基类提供 `extract_images()` 静态方法（行 1050-1096），可从 markdown (`![alt](url)`) 和 HTML (`<img src="url">`) 中提取图片 URL，支持 `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp` 以及 `fal.media`, `fal-cdn`, `replicate.delivery` 等 CDN 域名。

同时提供 `extract_media()` 方法处理 `MEDIA:<path>` 标签（TTS 工具生成），以及 `extract_local_files()` 帮助小模型自动检测裸文件路径。

### 6.3 Thread/Topic 支持

基类通过 `SessionSource` 中的 `thread_id` 和 `chat_topic` 字段建模线程/话题。`build_source()` 方法（行 1890-1917）是构造 `SessionSource` 的工厂：
```python
def build_source(self, chat_id, chat_name=None, chat_type="dm",
                 user_id=None, user_name=None, thread_id=None,
                 chat_topic=None, ...):
```

在 `_process_message_background()` 中（行 1593），线程元数据通过 metadata 字典传递给平台方法：
```python
_thread_metadata = {"thread_id": event.source.thread_id} if event.source.thread_id else None
```

### 6.4 消息处理与中断模型

`handle_message()`（行 1482-1569）是核心入口，采用 **快速返回 + 后台任务** 模式：
- 检查 `_active_sessions` 判断会话是否已有活跃处理任务。
- 若活跃且新消息为照片连拍（PHOTO burst），通过 `merge_pending_message_event()` 合并到 pending 而不中断。
- 若活跃且为其他消息，设置 `_active_sessions[session_key].set()` 触发中断信号。
- 特殊命令（`/approve`, `/deny`, `/status`, `/stop`, `/new`, `/reset`, `/background`, `/restart`）绕过活跃会话守卫直接分发。
- 新建会话时同步标记活跃（防止竞态），然后 `asyncio.create_task()` 启动后台任务。

`_process_message_background()`（行 1593-1679+）负责：
- 启动 `_keep_typing()` 持续输入指示器（每 2 秒刷新）。
- 调用 `_message_handler(event)` 获取 Agent 响应。
- 提取媒体、图片、本地文件。
- 自动 TTS：对语音输入（VOICE 类型）自动生成语音回复。
- 通过 `_send_with_retry()` 发送响应（支持 5 次重试、指数退避，仅对 `_RETRYABLE_ERROR_PATTERNS` 中的连接错误重试）。

### 6.5 消息分片

`truncate_message()`（行 1941-2071）将超长消息按平台限制分片，**保留代码块边界**：
- 在三重反引号代码块中间分割时，当前 chunk 会补全关闭 fence，下一个 chunk 用相同语言标签重新打开。
- 多 chunk 响应会附加 `(1/3)` 指示器。
- 支持自定义长度函数（如 Telegram 的 UTF-16 码单位计数 `utf16_len`）。
- 避免在行内代码段（``...``）中间分割。

---

## 7. Telegram 适配器

`TelegramAdapter(BasePlatformAdapter)`（`telegram.py:115`）。

### 7.1 概念映射

| Telegram 概念 | 基类抽象映射 |
|---|---|
| `chat.id` | `chat_id` |
| `message.message_thread_id` (Forum Topics) | `thread_id` (via metadata) |
| `message.message_id` | `message_id` / `reply_to` |
| `message.reply_to_message` | `reply_to_message_id` + `reply_to_text` |
| `message.chat.type` (private/group/channel) | `chat_type` ("dm"/"group"/"channel") |
| `message.caption` + `message.photo` | `media_urls` + `media_types` + `text` |

**DM Topics 支持**（行 157-160）：维护 `_dm_topics` 映射（topic_name → message_thread_id），在启动时从 `config.extra["dm_topics"]` 加载。

### 7.2 Webhook / Long Polling 集成

`connect()` 方法（行 479+）构建 `python-telegram-bot` 的 `Application`：
- 默认使用 **Long Polling**（`start_polling()`，行 604+）。
- 若环境变量 `TELEGRAM_WEBHOOK_URL` 设置，则启动 **Webhook** 模式（`start_webhook()`，行 626-639），端口默认 8443。
- 通过 `error_callback` 注册轮询错误处理器 `_polling_error_callback_ref`。
- `_handle_polling_network_error()`（行 196）：网络中断后指数退避重连（5s, 10s, 20s... 上限 60s，最多 10 次），超过后标记为 retryable-fatal。
- `_handle_polling_conflict()`（行 263）：处理 "Conflict: terminated by other getUpdates request"（多实例竞争），最多重试 `MAX_CONFLICT_RETRIES` 次，最终标记 fatal error `telegram_polling_conflict`。

### 7.3 群组/频道/私聊特殊处理

- **消息长度限制**：`MAX_MESSAGE_LENGTH = 4096`（行 127），`_SPLIT_THRESHOLD = 4000` 用于检测客户端分片。
- **媒体聚合**：`_media_batch_delay_seconds`（默认 0.8s）缓冲连拍照片，`MERGE_PENDING_MESSAGE_EVENT` 将同一相册的多个 `PHOTO` 事件合并为单个 `MessageEvent`。
- **文本聚合**：`_text_batch_delay_seconds`（默认 0.6s）和 `_text_batch_split_delay_seconds`（默认 2.0s）缓冲客户端长消息分片，合并为单个事件。
- **mention 模式**：`_reply_to_mode` 控制回复策略（'first' 等）。
- **交互式 Model Picker**：`_model_picker_state` 按 chat 维护状态。
- **审批按钮状态**：`_approval_state` 映射 `message_id → session_key`。

### 7.4 消息发送

`send()` 方法支持 MarkdownV2/HTML 格式、`message_thread_id` 传至 Telegram API 实现 Topic 回复、`reply_to_message_id`，并由 `_send_with_retry()` 包装。

`send_typing()`（行 1765-1782）调用 `bot.send_chat_action(action="typing", message_thread_id=...)`。

---

## 8. Discord 适配器

### 8.1 线程/频道映射

Discord 的 `channel_id` 映射到 `chat_id`。Discord 的 Thread 通过 `thread_id` 传递（metadata 中），与基类模型一致。

### 8.2 斜杠命令/交互处理

Discord 适配器实现了对 Discord Interactions（Slash Commands）的支持，将交互事件转换为 `MessageEvent` 后走统一处理流程。

### 8.3 速率限制与消息格式桥接

Discord 消息长度限制为 2000 字符（普通消息）/ 4000 字符（nitro/embed）。

**`send_typing()` 特殊实现**（行 1419-1451）：Discord 的 typing gateway event 对 bot 不可靠，因此启动 **后台循环任务** 每 8 秒调用一次 `POST /channels/{channel_id}/typing` API（typing 指示器持续约 10 秒）。任务存储在 `_typing_tasks[chat_id]` 中。

`stop_typing()`（行 1453-1455）取消该后台任务。

---

## 9. SMS 适配器

`SMSAdapter(BasePlatformAdapter)` 是极简实现。

| SMS 概念 | 基类映射 |
|---|---|
| 电话号码 (From/To) | `chat_id` |
| 短信文本 | `text` |
| — | `thread_id` = None |
| — | `media_urls` = 空（或 MMS 支持极有限）|

**限制**：
- **无编辑功能**：继承基类默认 `edit_message()`，返回 `Not supported`。
- **无线程/话题**：SMS 不存在线程概念，`thread_id` 始终为 None。
- **媒体约束**：SMS/MMS 对媒体大小和类型有严格运营商限制；适配器主要处理纯文本，媒体回退到发送 URL。
- **无 typing 指示器**：基类默认空操作。
- **消息长度限制**：160/70 字符（取决于编码），适配器可能需要分片。

---

## 10. 跨平台语义对比

| 功能维度 | Telegram | Discord | SMS |
|---|---|---|---|
| **消息长度限制** | 4096 | 2000/4000 | ~160 (GSM-7) / ~70 (UCS-2) |
| **编辑消息** | ✅ `editMessageText` | ✅ `edit_message` | ❌ 不支持 |
| **Reply/Thread** | ✅ Forum Topics (`message_thread_id`) + 回复 | ✅ Threads + 回复 | ❌ 无 |
| **Typing 指示器** | ✅ `sendChatAction(action="typing")` | ✅ 后台循环 POST /typing | ❌ 无 |
| **图片发送** | ✅ `sendPhoto` 原生 | ✅ `send` with embed/attachment | ⚠️ MMS 受限 |
| **语音消息** | ✅ `sendVoice` | ✅ 文件附件 | ❌ 不支持 |
| **GIF/Animation** | ✅ `sendAnimation` | ✅ Embed | ❌ 不支持 |
| **Markdown 格式** | MarkdownV2/HTML | Discord Markdown | 纯文本 |
| **Webhook/Polling** | ✅ 两者皆支持 | ✅ Gateway WebSocket | ❌ 无（接收通过 Twilio 回调） |
| **消息聚合** | 照片连拍/文本分片缓冲 | 无特殊聚合 | 无 |
| **速率限制** | 有（每分钟 20-30 条） | 严格（全局/频道/用户级） | 运营商限制 |
| **中断支持** | ✅ 活跃会话 Event 信号 | ✅ 活跃会话 Event 信号 | ❌ 通常无并发会话 |

---

## 11. Batch Runner

### 11.1 目的与差异

`BatchRunner`（`batch_runner.py` 行 514）用于**批量、非交互式**处理数据集中的 Agent prompts。与交互式 Gateway/CLI 的区别：
- **Gateway/CLI**：实时处理用户输入，维护长期会话状态，多轮对话。
- **Batch Runner**：离线处理数据集，每个 prompt 独立运行，无状态保留（除了 checkpoint），追求吞吐量和可恢复性。

### 11.2 复用 Agent 核心

在 `_process_batch_worker()` 函数（行 370+）中，每个工作进程独立：
1. 初始化 `AgentExecutor`（非交互式执行器）。
2. 构建消息列表：`[system_message] + prefill + [user_message]`。
3. 调用 `agent_executor.run(messages, ...)` 运行 Agent。
4. Agent 执行工具调用循环（最多 `max_iterations` 次）。
5. 收集工具统计（`tool_stats`）和推理统计（`reasoning_stats`）。

```python
# batch_runner.py ~370-500
def _process_batch_worker(task):
    ...
    agent_executor = AgentExecutor(...config...)
    history = [...system..., ...prefill..., {"role": "user", "content": prompt_text}]
    result = agent_executor.run(history, reasoning_config=..., ...)
```

### 11.3 输入格式

- **JSONL 文件**，每行一个 JSON 对象，必须包含 `"prompt"` 字段。
- 可选支持 `"conversations"` 字段（list of `{role, content}` 格式的兼容格式）。
- 通过 `--dataset_file` 指定。
- 通过 `--max_samples N` 可截断仅处理前 N 条。

示例：
```jsonl
{"prompt": "What is 2+2?"}
{"prompt": "Explain quantum computing"}
```

### 11.4 并发模型

- 使用 Python `multiprocessing.Pool`（行 880：`with Pool(processes=self.num_workers) as pool:`）。
- **默认 workers**：4（`num_workers=4`）。
- **调度方式**：`pool.imap_unordered(_process_batch_worker, tasks)` —— 无序返回、惰性求值。
- 每个 batch 作为一个独立任务提交给 worker 进程。
- 每个 worker 内部**串行**处理该 batch 中的 prompts（单进程内无并发）。

### 11.5 检查点与恢复

**检查点文件**：`data/{run_name}/checkpoint.json`

**内容**：
```json
{
  "run_name": "...",
  "completed_prompts": [0, 1, 3, ...],
  "batch_stats": {"0": {"processed": 10, "skipped": 2, ...}},
  "last_updated": "2026-05-17T09:43:00"
}
```

**智能恢复机制**（`--resume`）：
1. `_scan_completed_prompts_by_content()`（行 714）：**按内容匹配**扫描所有 `batch_*.jsonl` 输出文件，提取已成功处理（非 failed）的 prompt 文本集合。
2. `_filter_dataset_by_completed()`（行 758）：根据 prompt 文本内容过滤数据集，排除已完成的条目。
3. 重建 batch 列表，仅处理未完成项。
4. 恢复时保留已有 checkpoint 中的进度，不会覆盖。

**增量检查点**：每完成一个 batch 立即更新 checkpoint（行 926-939），使用 `atomic_json_write` 和 `Lock` 保证线程安全。即使进程崩溃，已完成的 batch 也能被恢复。

### 11.6 输出格式

**目录结构**：`data/{run_name}/`

| 文件 | 说明 |
|---|---|
| `batch_*.jsonl` | 各 batch 的原始输出（保留用于调试） |
| `trajectories.jsonl` | 合并后的完整轨迹文件（去重+过滤） |
| `statistics.json` | 最终统计信息 |
| `checkpoint.json` | 检查点状态 |

**单条 trajectory 记录包含**：
```json
{
  "prompt": "...",
  "conversations": [
    {"from": "human", "value": "..."},
    {"from": "gpt", "value": "..."}
  ],
  "tool_calls": [...],
  "tool_stats": {"tool_name": {"count": 1, "success": 1, "failure": 0}},
  "reasoning_stats": {"total_assistant_turns": 3, "turns_with_reasoning": 2, ...},
  "failed": false,
  "error": null
}
```

**后处理合并**（行 992-1036）：
- 合并所有 `batch_*.jsonl` 到 `trajectories.jsonl`。
- **过滤损坏条目**：检测 `tool_stats` 中包含无效工具名（不在 `ALL_POSSIBLE_TOOLS` 中）的记录，提示并丢弃（通常由模型幻觉导致）。
- **过滤无效 JSON**。

### 11.7 其他关键配置

- **`ephemeral_system_prompt`**：运行时临时系统提示，**不会**保存到 trajectory 中。
- **`prefill_messages`**：从 JSON 文件加载 few-shot 预填充对话上下文。
- **`reasoning_config`**：OpenRouter 推理配置（`effort`: none/minimal/low/medium/high/xhigh）。
- **`max_tokens`**：限制模型输出长度。
- **`log_prefix_chars`**：日志中工具调用/响应预览字符数。

---

## 12. 设计哲学总结

| 原则 | 体现 |
|---|---|
| **Adapter-Centric，无中心路由** | Gateway 是 "nanny / controller"，每个平台自己管自己的长连接/轮询/webhook。新增平台只需新增 `BasePlatformAdapter` 子类，无需改动核心路由。 |
| **极端细粒度的 Session 隔离** | `build_session_key()` 精确区分 DM / Group / Thread / Per-user-in-group，配合 SQLite + JSONL 双写、自动过期、启动时挂起保护。 |
| **Sync → Async 桥接精巧** | `GatewayStreamConsumer` 用 `queue.Queue` 桥接 sync agent callback → async platform API，统一了“先发送再编辑”的流式 UX，并内置了 flood-control 自适应退避与无编辑能力平台的 graceful fallback。 |
| **Hook / Plugin 职责分层** | Gateway Event Hook（async、松耦合、fire-and-forget）覆盖网关层生命周期通知；CLI Plugin（sync、高权限、返回值收集）覆盖 Agent 核心执行级扩展。两者互补。 |
| **Batch Runner = 无状态 Harness 变体** | 复用同一 `AgentExecutor` 核心，但用 `multiprocessing.Pool` 做并行、按 prompt content 做恢复、以 JSONL 做输入输出，形成与交互式 loop 清晰的复用/分叉边界。 |
