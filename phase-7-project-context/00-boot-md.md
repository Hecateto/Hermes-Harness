# Phase 7.3 — BOOT.md 启动自检机制

**源码位置**: `gateway/builtin_hooks/boot_md.py` (87 行)

---

## 1. 定位与设计意图

BOOT.md 是 Hermes 的**启动自检清单（startup boot checklist）**。它解决了一个运维场景：Gateway 作为长期运行的后台服务，每次重启后需要自动执行一系列检查或初始化动作（例如查看 cron job 状态、汇报健康情况、扫描日志异常），而不需要等待用户发送第一条消息。

与 `.hermes.md`、`AGENTS.md`、`SOUL.md` 等不同，BOOT.md **不是静态注入 prompt 的上下文文件**，而是一次性的动态 Agent 执行流。

---

## 2. 触发条件与执行路径

### 2.1 触发条件

| 条件 | 说明 |
|------|------|
| 文件存在 | `~/.hermes/BOOT.md` 必须存在 |
| 内容非空 | 读取后 `.strip()` 不能为空字符串 |
| 事件源 | Gateway 启动时发出的 `startup` 事件（由 `gateway/hooks.py` 的事件总线分发） |

若条件不满足，hook **静默跳过**，不产生任何日志噪音。

### 2.2 执行流程

```
Gateway 启动
    └── handle("startup", context)  [async gateway hook]
            └── BOOT_FILE.exists() ?
                    ├── No → 静默返回
                    └── Yes → threading.Thread(daemon=True, target=_run_boot_agent)
                                    └── AIAgent.run_conversation(_build_boot_prompt(content))
```

### 2.3 Agent 参数

`_run_boot_agent` 内部实例化的 `AIAgent` 使用了保守的隔离参数：

| 参数 | 取值 | 意图 |
|------|------|------|
| `quiet_mode=True` | 静默 | 避免在启动时向平台发送大量状态消息 |
| `skip_context_files=True` | 跳过项目上下文 | 不需要加载 CWD 的 `.hermes.md` / `AGENTS.md` |
| `skip_memory=True` | 跳过 memory | 不检索历史记忆，避免污染用户 session |
| `max_iterations=20` | 最多 20 轮 | 防止启动检查清单中的某个步骤陷入无限循环 |

### 2.4 输出抑制

Agent 执行完成后，若输出包含字符串 `[SILENT]`，则视为"无事可报"，仅记录 `logger.info("boot-md completed (nothing to report)")`；否则截取前 200 个字符记录日志。

```python
response = result.get("final_response", "")
if response and "[SILENT]" not in response:
    logger.info("boot-md completed: %s", response[:200])
```

---

## 3. 安全边界

### 3.1 文件来源

BOOT.md 固定路径为 `~/.hermes/BOOT.md`，**仅限本地文件系统**，不接受远程输入或平台消息注入，因此天然规避了 prompt injection 攻击面。

### 3.2 线程隔离

执行在 **`daemon=True` 的后台线程**中，不阻塞 Gateway 的主 asyncio 事件循环启动。即使 Agent 内部崩溃（例如 API 调用异常、`max_iterations` 耗尽），也仅捕获异常并记录错误，不会影响 Gateway 的可用性。

```python
except Exception as e:
    logger.error("boot-md agent failed: %s", e)
```

### 3.3 Session 隔离

`skip_memory=True` 确保启动检查不会读取或写入用户的持久记忆，避免把启动时的临时状态混入正常对话历史。

---

## 4. 与 Prompt Builder 中 BOOT.md 注入的区别

在 `prompt_builder.py` 中，BOOT.md 是作为 **system prompt 附件**被扫描并注入到每一轮对话的上下文栈中的（优先级位于 SOUL.md、AGENTS.md、.hermes.md 之后）。它的作用是告诉 Agent"用户在项目根目录写了一个启动偏好"。

而 `gateway/builtin_hooks/boot_md.py` 中的 BOOT.md 则是**真正的执行语义**——它在 Gateway 启动时主动运行一次 Agent 任务，具有副作用（可能调用 send_message、执行代码等）。两者虽同名，但一个是**静态上下文附件**，一个是**动态启动钩子**。

---

**Insight**: BOOT.md 体现了 Hermes 的设计哲学——**平台生命周期事件应该可编程**。Gateway 不是被动等待消息的聊天机器人，而是一个可以自主执行启动脚本、定时任务（cron）、事件响应的 Agent 运行时。
