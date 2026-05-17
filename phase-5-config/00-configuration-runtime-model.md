# Phase 5 — Configuration & Runtime Model 配置系统与运行时参数仲裁

> **范围**: Hermes Harness v0.9.0 的配置加载、优先级仲裁、持久化、版本迁移，以及 CLI/Gateway/Agent 三层配置如何交互。
>
> **源文件**: `cli.py` (`load_cli_config`, `CLI_CONFIG`, `save_config_value`), `hermes_cli/config.py` (`load_config`, `DEFAULT_CONFIG`, `_config_version`, `_expand_env_vars`), `gateway/config.py` (`GatewayConfig`), `gateway/display_config.py`, `gateway/run.py` (`_load_gateway_config`, `_resolve_runtime_agent_kwargs`, `_resolve_gateway_model`, `_resolve_session_agent_runtime`, `_resolve_turn_agent_config`), `run_agent.py` (`AIAgent.__init__`), `tools/budget_config.py`, `hermes_cli/tools_config.py`, `hermes_cli/skills_config.py`, `hermes_cli/mcp_config.py`。
>
> **核心概念**: Hermes 没有"统一的配置中心"。CLI 与 Gateway 使用**两套完全独立的配置文件**（`config.yaml` vs `gateway-config.yaml`），但在 Agent Core 初始化时通过一致的 5 层优先级仲裁收敛。配置系统服务于"终端用户易用性"，而非"服务编排一致性"。

---

## 1. 核心发现总览

| 发现 | 说明 |
|---|---|
| **两套配置文件并存** | CLI 读 `~/.hermes/config.yaml`（或 `./cli-config.yaml`）；Gateway 读 `gateway-config.yaml`（或 `~/.hermes/gateway-config.yaml`）。两者路径、加载函数、默认值全独立。 |
| **5 层运行时仲裁** | Agent 初始化参数优先级（高→低）：**显式构造参数** > **会话级/轮级覆盖** > **Gateway/CLI 配置文件** > **环境变量** > **代码硬编码默认值**。 |
| **无配置热重载** | 修改 YAML 后必须重启 Gateway/CLI。仅环境变量和 `_session_model_overrides` 可即时生效。 |
| **Budget 是 Gateway 全局值** | `BudgetConfig` 是 `GatewayConfig` 的子字段，创建每个 `AIAgent` 时统一注入 `max_iterations`，无 per-session / per-platform 独立预算。 |
| **`${VAR}` 递归展开** | 配置值支持 `${VAR}` 环境变量插值（仅字符串），未定义时保留原样不报错。 |
| **配置版本化** | `_config_version = 17`，加载时自动缺失字段检测 + 根级 key 迁移归一化。 |
| **Managed Mode 拦截** | NixOS/Homebrew 包管理安装下，`save_config()` 拒绝写入用户配置，提示通过包管理器更新。 |

---

## 2. CLI 配置系统

### 2.1 配置加载：双函数并存

Hermes CLI 有两个看似重复实则职责分层的配置加载函数：

#### `cli.py::load_cli_config()`（L192-520）— CLI 运行时主加载器

```python
# cli.py L203-211
user_config_path = _hermes_home / 'config.yaml'
project_config_path = Path(__file__).parent / 'cli-config.yaml'
if user_config_path.exists():
    config_path = user_config_path
else:
    config_path = project_config_path
```

- **查找顺序**: `~/.hermes/config.yaml`（优先）→ `./cli-config.yaml`（fallback）
- **合并策略**: 文件配置 `deep_merge` 到硬编码 `defaults` 字典（L214-310）
- **桥接环境变量**: 加载完成后，将 `terminal`、`browser`、`auxiliary`、`security` 等子树主动写入对应环境变量（L443-518），使下游模块（如 `terminal_tool.py`）通过 env var 读取配置。

#### `hermes_cli/config.py::load_config()`（L2408-2432）— Setup/配置命令加载器

```python
def load_config():
    config = copy.deepcopy(DEFAULT_CONFIG)   # 1. 硬编码默认值（~700 行大字典）
    if config_path.exists():
        user_config = yaml.safe_load(f)      # 2. 用户配置 deep-merge
        config = _deep_merge(config, user_config)
    return _expand_env_vars(
        _normalize_root_model_keys(
            _normalize_max_turns_config(config)
        )
    )
```

- **只读 `~/.hermes/config.yaml`**，不读 `./cli-config.yaml`
- 用于 `hermes setup` 向导、`hermes config` 命令、版本检查
- 配置版本、缺失字段检测、归一化逻辑均在此文件内实现

**两套函数的共存原因**: `cli.py` 在启动早期加载，需要支持项目级配置（`./cli-config.yaml`）以便开发者在 repo 中嵌入默认配置；`hermes_cli/config.py` 在 setup 场景使用，只关心用户持久化设置，不需要项目级覆盖。

### 2.2 配置迁移与版本控制

**`_config_version`**（`hermes_cli/config.py` L703）：当前硬编码为 **17**。每次新增 required field 时递增版本号。

**版本检查**（`check_config_version()` L1685-1694）：
```python
def check_config_version() -> Tuple[int, int]:
    config = load_config()
    current = config.get("_config_version", 0)
    latest = DEFAULT_CONFIG.get("_config_version", 1)
    return current, latest
```

**缺失字段递归检测**（`get_missing_config_fields()` L1490-1515）：
- 遍历 `DEFAULT_CONFIG` 全树
- 跳过 `_` 前缀的元数据 key
- 报告缺失键及其默认值，用于 setup wizard 提示

**自动归一化**（`load_config()` 内部调用）：
1. `_normalize_root_model_keys()`（L2343-2370）：把根级 `provider:` / `base_url:` 迁移到 `model.provider` / `model.base_url`
2. `_normalize_max_turns_config()`（L2373-2386）：把根级 `max_turns` 迁移到 `agent.max_turns`

版本相关环境变量映射（`ENV_VARS_BY_VERSION` L712-719）：
```python
ENV_VARS_BY_VERSION = {
    3: ["FIRECRAWL_API_KEY", "BROWSERBASE_API_KEY", ...],
    4: ["VOICE_TOOLS_OPENAI_KEY", "ELEVENLABS_API_KEY"],
    5: ["WHATSAPP_ENABLED", "WHATSAPP_MODE", ...],
    10: ["TAVILY_API_KEY"],
    11: ["TERMINAL_MODAL_MODE"],
}
```
用于迁移时只提示新版本新增的环境变量，避免重复骚扰。

### 2.3 顶层配置段

`DEFAULT_CONFIG`（`hermes_cli/config.py` L311-704）包含 30+ 个顶层段：

| 段名 | 关键字段 |
|---|---|
| `model` | `default`, `provider`, `base_url` |
| `providers` | 自定义 provider 字典（name → adapter + endpoint） |
| `fallback_providers` | 故障转移 provider 链 |
| `credential_pool_strategies` | 凭据池策略（round_robin / random / least_used / sequential） |
| `toolsets` | 启用的工具集列表（ composite presets + 独立 toolsets ） |
| `platform_toolsets` | 各平台（cli/telegram/discord/...）的独立 toolset 列表 |
| `agent` | `max_turns`, `timeout`, `tool_use_enforcement`, `reasoning_effort` |
| `terminal` | `backend`, `docker_image`, `container_cpu/memory`, `persistent_shell` |
| `browser` | `inactivity_timeout`, `record_sessions`, `camofox` |
| `checkpoints` | 文件系统快照开关与数量限制 |
| `compression` | 上下文压缩阈值与策略 |
| `smart_model_routing` | 简单请求路由到廉价模型 |
| `auxiliary` | vision / web_extract / compression / approval / mcp 等辅助模型 |
| `display` | `compact`, `skin`, `streaming`, `inline_diffs`, `tool_progress` |
| `privacy` | `redact_pii` |
| `tts` / `stt` / `voice` | 语音合成、识别、录音配置 |
| `memory` | 持久化记忆开关与字符限制 |
| `delegation` | 子 agent 委托的 model / provider / max_iterations |
| `skills` | 外部技能目录列表 |
| `honcho` | Honcho AI 记忆集成 |
| `approvals` | 危险命令审批模式（manual / smart / off） |
| `security` | tirith 扫描、secret redaction、website blocklist |
| `cron` | 定时任务响应包装 |
| `logging` | 日志级别与轮转 |
| `_config_version` | 配置模式版本号 |

### 2.4 CLI 初始化时的 Model/Provider 解析

`HermesCLI.__init__`（`cli.py` L1581-1780）中各参数的解析顺序：

#### Model
```python
# L1648-1651
_model_config = CLI_CONFIG.get("model", {})
_config_model = (_model_config.get("default") or _model_config.get("model") or "")
self.model = model or _config_model or ""
```
**优先级**: CLI arg `model` > `config.yaml model.default` > `config.yaml model.model` > `""`

**注意**: docstring（L1643-1647）明确说明**不检查** `LLM_MODEL` / `OPENAI_MODEL` 等环境变量：
> "LLM_MODEL/OPENAI_MODEL env vars are NOT checked — config.yaml is authoritative."

#### Provider
```python
# L1674-1679
self.requested_provider = (
    provider
    or CLI_CONFIG["model"].get("provider")
    or os.getenv("HERMES_INFERENCE_PROVIDER")
    or "auto"
)
```
**优先级**: CLI arg `provider` > `config.yaml model.provider` > `HERMES_INFERENCE_PROVIDER` env var > `"auto"`

#### Base URL
```python
# L1685-1689
self.base_url = (
    base_url
    or CLI_CONFIG["model"].get("base_url", "")
    or os.getenv("OPENROUTER_BASE_URL", "")
) or None
```
**优先级**: CLI arg `base_url` > `config.yaml model.base_url` > `OPENROUTER_BASE_URL` env var

#### API Key
```python
# L1693-1696
if self.base_url and "openrouter.ai" in self.base_url:
    self.api_key = api_key or os.getenv("OPENROUTER_API_KEY") or os.getenv("OPENAI_API_KEY")
else:
    self.api_key = api_key or os.getenv("OPENAI_API_KEY") or os.getenv("OPENROUTER_API_KEY")
```
**优先级**: CLI arg `api_key` > 环境变量（根据 base_url 智能决定优先检查哪个 key）

#### Max Turns
```python
# L1698-1707
if max_turns is not None:
    self.max_turns = max_turns
elif CLI_CONFIG["agent"].get("max_turns"):
    self.max_turns = CLI_CONFIG["agent"]["max_turns"]
elif CLI_CONFIG.get("max_turns"):         # 向后兼容根级 max_turns
    self.max_turns = CLI_CONFIG["max_turns"]
elif os.getenv("HERMES_MAX_ITERATIONS"):
    self.max_turns = int(os.getenv("HERMES_MAX_ITERATIONS"))
else:
    self.max_turns = 90
```
**优先级**: CLI arg > `agent.max_turns` > root `max_turns` > `HERMES_MAX_ITERATIONS` env var > **默认值 90**

### 2.5 配置持久化：`save_config_value()`

**函数位置**: `cli.py` L1510-1564

**目标文件选择**: 与 `load_cli_config()` 保持相同查找顺序：
1. 若 `~/.hermes/config.yaml` 存在，则写入该文件
2. 否则写入 `./cli-config.yaml`

**写入流程**:
1. `mkdir(parents=True, exist_ok=True)`
2. 按点分隔的 key path（如 `"agent.system_prompt"`）导航并设置值
3. **原子写入**: `utils.atomic_yaml_write()` 实现 temp file + fsync + `os.replace`
4. 设置文件权限 `0o600`（owner-only）

```python
# L1542-1558
keys = key_path.split('.')
current = config
for key in keys[:-1]:
    if key not in current or not isinstance(current[key], dict):
        current[key] = {}
    current = current[key]
current[keys[-1]] = value

from utils import atomic_yaml_write
atomic_yaml_write(config_path, config)
try:
    os.chmod(config_path, 0o600)
except (OSError, NotImplementedError):
    pass
```

**Managed Mode 拦截**（`hermes_cli/config.py` L2532-2557）：NixOS/Homebrew 托管安装下，`save_config()` 拒绝写入，提示用户通过包管理器更新。

### 2.6 环境变量插值：`${VAR}` 展开

**实现**: `hermes_cli/config.py` L2323-2340

```python
def _expand_env_vars(obj):
    if isinstance(obj, str):
        return re.sub(
            r"\${([^}]+)}",
            lambda m: os.environ.get(m.group(1), m.group(0)),
            obj,
        )
    if isinstance(obj, dict):
        return {k: _expand_env_vars(v) for k, v in obj.items()}
    if isinstance(obj, list):
        return [_expand_env_vars(item) for item in obj]
    return obj
```

- 仅处理**字符串值**；数字、布尔值、`None` 保持不变
- 未解析引用**保留原样**（`m.group(0)`），不报错
- 递归处理嵌套 dict/list

**调用时机**:
1. `load_config()` 返回前（L2432）
2. `load_cli_config()` 合并 defaults 后、桥接环境变量前（L386-388）

---

## 3. Gateway 配置系统

### 3.1 Gateway 配置完全独立于 CLI

| 维度 | CLI | Gateway |
|---|---|---|
| **加载函数** | `cli.py::load_cli_config()` | `gateway/run.py::_load_gateway_config()` |
| **文件路径** | `~/.hermes/config.yaml` 或 `./cli-config.yaml` | `gateway-config.yaml` 或 `~/.hermes/gateway-config.yaml` |
| **源文件** | `cli.py`, `hermes_cli/config.py` | `gateway/config.py`, `gateway/run.py` |
| **默认值** | `cli.py` 中硬编码 `defaults` 字典 | `GatewayConfig` dataclass 字段默认值 |
| **加载时机** | 进程启动时一次加载为模块级 `CLI_CONFIG` | `GatewayRunner.__init__` 时加载 |

### 3.2 `GatewayConfig` 结构

`gateway/config.py` 定义了 `GatewayConfig` dataclass（行 100-300 附近）：

```python
@dataclass
class GatewayConfig:
    model: str = "claude-sonnet-4-20250514"
    model_aliases: Dict[str, str] = field(default_factory=lambda: {
        "fast": "gpt-4.1-nano",
        "smart": "claude-sonnet-4-20250514",
        "vision": "gemini-2.5-pro",
    })
    platforms: Dict[str, PlatformConfig] = field(default_factory=dict)
    routing_rules: List[Dict[str, Any]] = field(default_factory=list)
    delivery: DeliveryConfig = field(default_factory=DeliveryConfig)
    session: SessionConfig = field(default_factory=SessionConfig)
    display: DisplayConfig = field(default_factory=DisplayConfig)
    budget: BudgetConfig = field(default_factory=BudgetConfig)
```

包含子段：`model`, `platforms`, `routing_rules`, `delivery`, `session`, `display`, `budget`, `logging`, `tracing`。

### 3.3 `_load_gateway_config()` 加载入口

`gateway/run.py` ~L280：
```python
def _load_gateway_config() -> dict:
    """
    从以下位置加载 gateway 专属 config.yaml：
    1. 环境变量 HERMES_GATEWAY_CONFIG 指定的路径
    2. 当前工作目录 ./gateway-config.yaml
    3. ~/.hermes/gateway-config.yaml（用户级）
    若均不存在，返回空 dict。
    """
```

**与 CLI 配置完全隔离**，GatewayRunner 不消费 `CLI_CONFIG`。

---

## 4. AIAgent 运行时参数仲裁（跨 CLI/Gateway 统一）

### 4.1 5 层优先级金字塔

当创建 `AIAgent` 实例时，参数按以下优先级（高→低）合并：

```
┌──────────────────────────────────────────────────┐
│ 1. 显式构造函数参数 (AIAgent.__init__)            │  ← 最高优先级
│    model=..., max_tokens=..., reasoning_config=... │
└────────────┬─────────────────────────────────────┘
             ▼
┌──────────────────────────────────────────────────┐
│ 2. 轮级/会话级覆盖                                │
│    _session_model_overrides[session_key]          │
│    turn_route["runtime"]（smart_model_routing）   │
│    resolve_turn_route() 返回的动态路由            │
└────────────┬─────────────────────────────────────┘
             ▼
┌──────────────────────────────────────────────────┐
│ 3. Gateway / CLI 配置文件                         │
│    gateway-config.yaml model.base_url 等          │
│    CLI_CONFIG["model"]["provider"] 等             │
└────────────┬─────────────────────────────────────┘
             ▼
┌──────────────────────────────────────────────────┐
│ 4. 环境变量                                       │
│    HERMES_INFERENCE_PROVIDER, OPENAI_API_KEY,     │
│    OPENROUTER_BASE_URL, HERMES_MAX_ITERATIONS...  │
└────────────┬─────────────────────────────────────┘
             ▼
┌──────────────────────────────────────────────────┐
│ 5. 代码硬编码默认值                                │  ← 最低优先级
│    model="", provider="auto", max_turns=90        │
└──────────────────────────────────────────────────┘
```

### 4.2 关键控制点函数

| 函数 | 文件 | 行号 | 职责 |
|------|------|------|------|
| `_resolve_runtime_agent_kwargs()` | `gateway/run.py` | ~319 | 每次创建 Agent 前动态解析 provider 凭据。尊重 `HERMES_INFERENCE_PROVIDER` 等环境变量。**不缓存**，因此运行时修改 env 即时生效。 |
| `_resolve_gateway_model()` | `gateway/run.py` | ~442 | Gateway 模型名的唯一可信源。支持 model 为 str（直接使用）或 dict（取 `default`/`model` key）。 |
| `_resolve_session_agent_runtime()` | `gateway/run.py` | ~850 | 会话级覆盖。若 `_session_model_overrides[session_key]` 包含完整 provider bundle（provider + api_key + base_url + api_mode），**直接短路返回**；若仅部分覆盖，保留覆盖值并回退到环境变量解析。 |
| `_resolve_turn_agent_config()` | `gateway/run.py` | ~924 | 轮级路由。调用 `agent/smart_model_routing.py::resolve_turn_route()` 根据消息内容、模型名、智能路由规则生成 turn_route。再经 `resolve_fast_mode_overrides()` 处理快速模式覆盖。 |

### 4.3 Gateway 中的 Agent 实例化模式

```python
# gateway/run.py 各处 AIAgent 创建点
turn_route = self._resolve_turn_agent_config(user_message, model, runtime_kwargs)
agent = AIAgent(
    model=turn_route["model"],
    **turn_route["runtime"],
    max_iterations=self.config.budget.max_iterations_budget,
)
```

### 4.4 CLI 中的 Agent 实例化模式

```python
# cli.py 中 HermesCLI 创建 Agent 时
agent = AIAgent(
    model=self.model,
    max_tokens=self.max_tokens,
    ...
    max_iterations=self.max_turns,
)
```

CLI 没有 `_session_model_overrides` 和 `resolve_turn_route` 逻辑（这些是 Gateway 特有的），因此 CLI 的仲裁链更短：显式参数 > CLI_CONFIG > 环境变量 > 默认值。

---

## 5. 子系统配置（Tools / Skills / MCP）

### 5.1 Tools Config (`hermes_cli/tools_config.py`)

#### 工具注册模型
`TOOLSETS`（L85-250）是一个 `OrderedDict`，将 toolset 名称映射到工具模块路径集合：
```python
TOOLSETS = OrderedDict([
    ("web", [{"module": "tools.web_search_tool", "tools": ["web_search", ...]}, ...]),
    ("terminal", [...]),
    ("file", [...]),
    ...
])
```

同时定义 `CONFIGURABLE_TOOLSETS`（三元组列表 `(key, label, description)`），用于交互式向导列出可开关的工具集。

#### 运行时仲裁：`_get_platform_tools()`（L465-563）

这是**决定某平台启用哪些工具集的统一收口函数**：

1. 读取 `platform_toolsets[platform]`；
2. 若未配置/非法，回退到 `PLATFORMS[platform]["default_toolset"]`（如 `hermes-cli`）；
3. **显式配置检测**: 检查列表中是否包含任何 `CONFIGURABLE_TOOLSETS` 的 key：
   - 若有 → 直接用成员隶属判断启/禁（避免 composite toolset 子集推导 bug）；
   - 若无 → 把 composite 名称（如 `hermes-cli`）解析为底层工具列表，再反向映射为独立 toolset key；
4. **插件 toolsets**: 新插件默认启用；在 `hermes tools` 中保存过后若不在列表中则视为用户主动禁用；
5. **MCP Server 处理**: 读取 `config["mcp_servers"]`，支持 `no_mcp` sentinel 显式关闭所有 MCP，显式列出的 MCP server name 视为 allowlist。

```python
# L494-507
if has_explicit_config:
    enabled_toolsets = {ts for ts in toolset_names if ts in configurable_keys}
else:
    all_tool_names = set()
    for ts_name in toolset_names:
        all_tool_names.update(resolve_toolset(ts_name))
    enabled_toolsets = set()
    for ts_key, _, _ in CONFIGURABLE_TOOLSETS:
        ts_tools = set(resolve_toolset(ts_key))
        if ts_tools and ts_tools.issubset(all_tool_names):
            enabled_toolsets.add(ts_key)
```

#### 持久化：`_save_platform_tools()`（L566+）
将用户在 `hermes tools` 向导中的选择写回 `config["platform_toolsets"][platform]`，同时保留非 configurable 条目（如 MCP server 名）。

### 5.2 Skills Config (`hermes_cli/skills_config.py`)

#### 三层加载架构（按优先级递减）

| 层级 | 函数 | 搜索路径 | 覆盖规则 |
|------|------|---------|---------|
| **Built-in** | `load_builtin_skills()` L99-137 | `~/.hermes/skills/built_in/` | 最高优先级，**不会被 custom/project 覆盖** |
| **Custom** | `load_custom_skills()` L140-182 | `~/.hermes/skills/` | 中等优先级，可被 project 同名覆盖 |
| **Project** | `load_project_skills()` L185-210 | `./skills/` | 最低优先级，可覆盖 custom 同名 skill |

#### 发现与去重：`discover_all_skills()`（L212-283）
- 聚合三层目录
- 同名 built-in skill **不会被覆盖**（保护核心技能）
- 同名 custom skill **会被 project 同名覆盖**
- 返回 `List[SkillSpec]`，含 `name, path, builtin, priority, description`

#### 单文件 vs 目录 skill
- **单文件**: `.py` 或 `.md`，提取 metadata YAML front-matter（`_load_skill_file` L329-350）
- **目录**: 含 `skill.py` 或 `skill.md`，同样读取 front-matter（`_load_skill_dir` L353-420）

### 5.3 MCP Config (`hermes_cli/mcp_config.py`)

#### 配置文件
- 默认加载 `~/.hermes/mcp.json`（可由 env 覆盖）
- `load_mcp_config()` L37-62
- `validate_mcp_config()` L65-95：确保每个 server 条目有必需字段

#### 双连接模式：`get_mcp_client_config()`（L126-190）

| 模式 | 识别方式 | 说明 |
|------|---------|------|
| **Native stdio** | 配置中直接含 `command` 字段 | 直接调用子进程，用标准 MCP stdio 传输 |
| **mcporter bridge** | 配置中有 `url` 字段（或 `api_base`） | 通过 mcporter HTTP bridge 代理连接 |

```python
if server_cfg.get("command"):
    # Native stdio client
    client_configs[name] = {
        "command": server_cfg["command"],
        "args": server_cfg.get("args", []),
        "env": {**os.environ, **server_cfg.get("env", {})},
    }
elif server_cfg.get("url") or server_cfg.get("api_base"):
    # mcporter bridge
    ...
```

### 5.4 Budget Config (`tools/budget_config.py`)

```python
@dataclass
class BudgetConfig:
    max_iterations_budget: int = 90
    max_token_budget: int = 0       # 0 = 不限制
    max_cost_budget: float = 0.0    # 0.0 = 不限制
    token_warning_threshold: float = 0.8
```

- **全局默认**: `BudgetConfig` 是 `GatewayConfig` 的子字段
- **创建 Agent 时统一注入**: `AIAgent(max_iterations=self.config.budget.max_iterations_budget)`
- **无 per-platform / per-session 独立预算**: 修改 `gateway-config.yaml` 中的 `budget:` 区块是全局生效的唯一方式

---

## 6. Display 配置分离

### 6.1 为何独立？

CLI 与 Gateway 的运行模式根本不同：

| 维度 | CLI Display (`cli.py`) | Gateway Display (`gateway/display_config.py`) |
|------|------------------------|-----------------------------------------------|
| **运行环境** | 终端用户交互 | 服务端/后台进程，多平台并发 |
| **配置主体** | `cli-config.yaml` 的 `display:` 节 | `gateway-config.yaml` 的 `display:` 节 |
| **皮肤引擎** | `hermes_cli.skin_engine`（终端 ANSI 渲染） | Gateway 内部 simplified skin handler |
| **流式输出** | `streaming: True` | 可按平台覆盖 |
| **忙等输入** | `interrupt` / `queue` | 通常固定为 `queue`（服务端无信号中断） |
| **消息头信息** | 服务器 CPU、上下文占用百分比 | 平台标识、session ID、线程 ID |

因此 Hermes 将 Gateway 的显示配置提取到 `gateway/display_config.py`（~200 行），定义 `DisplayConfig` dataclass 并提供 YAML 反序列化，被 `GatewayConfig` 引用。这避免了 CLI skin engine 与 Gateway 多平台渲染需求之间的冲突。

---

## 7. 配置热重载

### 7.1 不支持热重载

在 `gateway/run.py`（9005 行）与 `gateway/config.py`（1125 行）中：
- **无文件 watchers**（`watchdog`、`pyinotify`）
- **无 `SIGHUP` 重载信号处理**
- **无周期性轮询 `config.yaml` mtime**
- **无 `GatewayRunner.reload_config()` 方法**

`_load_gateway_config()` 仅在 `GatewayRunner.__init__` 或其内部懒加载函数中被调用，调用结果不持久化到可刷新的缓存。

### 7.2 运行时唯一"可变"的参数

1. **环境变量**: `_resolve_runtime_agent_kwargs()` 每次新建 `AIAgent` 都重新读取 `HERMES_INFERENCE_PROVIDER` 等 env
2. **会话级覆盖**: `_session_model_overrides[session_key] = {...}` 可即时生效

若管理员修改了 `gateway-config.yaml` 或 `~/.hermes/config.yaml`，必须重启 Gateway/CLI 进程才能生效。

---

## 8. 两套配置体系并存的设计动机

许多框架（如 Django、Flask）使用单一配置文件，通过环境变量或不同 `settings.py` 区分运行模式。Hermes 为何选择**两套完全独立的配置文件**？

| 动机 | 说明 |
|---|---|
| **用户角色分离** | CLI 用户是"开发者/终端用户"，Gateway 用户是"运维/平台管理员"。两者的配置变更频率和关注点不同，合并会导致权限混乱。 |
| **配置规模差异** | CLI 配置~700 行 defaults，包含大量终端交互参数（skin、streaming、bell_on_complete）；Gateway 配置更关注平台 token、路由规则、预算上限。合并后文件臃肿。 |
| **部署模式差异** | CLI 通常与代码仓库一起移动，支持 `./cli-config.yaml` 项目级覆盖；Gateway 通常作为 systemd/docker 服务常驻，有自己的配置挂载路径（`HERMES_GATEWAY_CONFIG`）。 |
| ** Gateway 多实例** | 同一套 CLI 配置可以连接多个 Gateway 实例，但每个 Gateway 可能服务不同平台群、使用不同模型预算。独立配置允许多实例隔离。 |
| **版本迁移复杂度** | CLI 的 `_config_version` 只关心用户级配置；Gateway 若也加入同一版本体系，会导致迁移逻辑纠缠。独立后各自演化。 |

---

## 9. 参数仲裁全链路示例

以 Gateway 模式下处理一条 Telegram 消息为例，展示配置如何逐层汇聚到 `AIAgent`：

```
1. 收到 Telegram 消息
   ├── 2. _resolve_gateway_model() 读取 gateway-config.yaml → "claude-sonnet-4"
   ├── 3. _resolve_session_agent_runtime()
   │       ├── 检查 _session_model_overrides[session_key] → 无
   │       └── 保留 gateway model
   ├── 4. _resolve_turn_agent_config()
   │       ├── resolve_turn_route(user_message, "claude-sonnet-4", gateway_config)
   │       └── smart_model_routing 判断为简单请求
   │           └── turn_route["model"] = "gpt-4.1-nano" (fast mode override)
   │       └── turn_route["runtime"] = {api_key, base_url, provider...}
   │           └── 来自 _resolve_runtime_agent_kwargs() → 读取 HERMES_INFERENCE_PROVIDER 等 env
   └── 5. AIAgent(
           model="gpt-4.1-nano",          ← turn_route["model"]
           api_key="sk-...",              ← turn_route["runtime"]
           base_url="https://...",        ← turn_route["runtime"]
           max_iterations=90,             ← GatewayConfig.budget.max_iterations_budget
           ...
       )
```

CLI 模式下的链路更短：
```
1. 用户输入 prompt
   └── HermesCLI.__init__ 已解析 model/provider/api_key 到 self.model 等字段
       └── AIAgent(model=self.model, api_key=self.api_key, max_iterations=self.max_turns)
```

---

## 10. 总结

Hermes 的配置系统呈现出鲜明的**分层隔离 + 运行时仲裁**设计：

- **加载层**: CLI 与 Gateway 各自独立文件、独立加载函数、独立默认值体系。
- **归一化层**: `load_config()` 内部做向后兼容的 key 迁移（root `provider` → `model.provider`），保证旧用户配置不崩溃。
- **仲裁层**: Agent Core 初始化时遵循统一的 5 层优先级，使 CLI 与 Gateway 在底层收敛到同一运行时语义。
- **持久化层**: 原子写入 + 权限控制 + Managed Mode 拦截，兼顾安全与跨包管理器兼容。
- **扩展层**: Tools/Skills/MCP 各自通过独立配置模块管理，但统一通过 `platform_toolsets` 参与 Gateway 的平台级门控。

这一设计的 trade-off 是**灵活性优先于一致性**：用户可以非常灵活地在不同层级覆盖配置，但也意味着需要理解 5 层优先级才能准确预测 Agent 实际使用的参数。对于运维场景，推荐的做法是：Gateway 配置通过 `gateway-config.yaml` 统一管理，环境变量仅用于敏感凭据（API key），避免在配置文件中明文存储。
