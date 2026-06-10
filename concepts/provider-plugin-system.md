---
title: ProviderProfile 插件系统
created: 2026-05-20
updated: 2026-06-09
type: concept
tags: [provider, transport, plugin, model-routing, openrouter-anthropic-reasoning-mandatory, anthropic-modern-thinking-default, openrouter-live-context-length, nemo-relay-plugins-toml-lifecycle, langfuse-sanitized-response, vision-tool-messages-flag]
source_files:
  - providers/base.py
  - providers/__init__.py
  - plugins/model-providers/
  - plugins/model-providers/README.md
  - plugins/model-providers/openrouter/__init__.py
  - plugins/observability/nemo_relay/__init__.py
  - plugins/observability/langfuse/__init__.py
verified_against: hermes-agent HEAD (a5d05cf30, 2026-06-09)
---

# ProviderProfile 插件系统（v0.13.0 起）

> **2026-06-09 增量（hermes-agent `a5d05cf30`）— OpenRouter Anthropic 守护 + NeMo-Relay plugins.toml lifecycle 四连 + Langfuse sanitized response 修**：
>
> - **OpenRouter Anthropic reasoning-mandatory 保护**（PR #43012 `46fedef07` + PR #42991 `1febb0824`）—— `plugins/model-providers/openrouter/__init__.py:21-29 _ANTHROPIC_REASONING_OPTIONAL_SUBSTRINGS` 列举仍接受 explicit "disable thinking" form 的 Anthropic 旧 family（与 `_LEGACY_MANUAL_THINKING_CLAUDE_SUBSTRINGS` 同 shape）；`:32-44 _anthropic_reasoning_is_mandatory(model)` 默 mandatory（unknown Anthropic 视为 mandatory，与 Anthropic 直连 default-to-modern 一致）；`:118-140` 三分支：mandatory 时 `pass` 完全省 reasoning；`reasoning_config` 显式给则用；否则默 `{enabled: True, effort: "medium"}`。修 chat_completions 不回放 signed thinking blocks 致 tool-replay turn 上 OpenRouter 发 `thinking: {type: "disabled"}` 被 4.6+/fable 模型 HTTP 400 拒。
> - **OpenRouter live `context_length` step-5f branch**（PR #42986 `967c325da fix(models)`）—— `agent/model_metadata.py:1815-1835` 新 step-5f 在 step-6 hardcoded catch-all 之前查 OpenRouter live `entry.get("context_length")`。Guard `if effective_provider == "openrouter":` 修原 step-6 死代码（OpenRouter selection 设 `effective_provider="openrouter"`，原 `not effective_provider` 把守的 fallback 永不触发）；保留 Kimi-family 32k underreport guard `not (or_ctx == 32768 and _model_name_suggests_kimi(model))`。
> - **Vision providers 拒 list-type tool content 主动降级**（已合入；新增字段 `ProviderProfile.supports_vision_tool_messages: bool = True`，Xiaomi MiMo 设 False；`_tool_result_content_for_active_model` proactively 查此字段，False 时返 text summary 替 list content 免 400 "text is not set" round-trip）。
> - **NeMo-Relay plugins.toml lifecycle 四连**（`9d61076f8 + ecd4679d8 + 728612c29 + 021d1034d` 全在 `plugins/observability/nemo_relay/__init__.py`）—— 完整状态机：`_atof_subscriber_name = "hermes.nemo_relay.atof"` at `:69` 单一来源；构造器 `:63-73` set `_plugin_config_needs_reinit = False` 然后 `if not self._plugin_config_initialized: self._activate_direct_fallbacks()`；`_activate_direct_fallbacks() at :103-105`；`_maybe_reinitialize_plugins_toml() at :107-115` 在 `_plugin_config_needs_reinit and not _plugin_config_initialized` 时重试，成功清 direct ATOF + 取消 reinit flag，失败走 direct fallbacks；`_plugins_toml_owns_exporter(exporter_name) at :117-121` 组合 `_plugin_config_initialized + _observability_exporter_enabled(...)`；`_configure_atof at :142-155` 现 idempotent 且用常量名；`_clear_atof() at :157-166` 调 `deregister(self._atof_subscriber_name)` 后 `atof_exporter = None`；`_clear_plugins_toml at :90-101` 包 try/finally 让 clear 失败也 re-arm（before：`_plugin_config_initialized = True` + `_plugin_config_needs_reinit = False` 让下次 session 既跳 reinit 又跳 fallback）；`ensure_session at :168-` 先调 `_maybe_reinitialize_plugins_toml()`；per-session ATIF exporter gate 改 `self.settings.atif_enabled and not self._plugins_toml_owns_exporter("atif")`；finalize `:235-241` 加 `elif self.settings.plugins_config and not self.sessions: self._plugin_config_needs_reinit = True`。`_Settings.adaptive_mode` 默 `"observe" → "observe_only"` 对齐上游 NeMo Relay contract `[components.config.tool_parallelism] mode`（`_adaptive_mode(config) at :696-707`）。
> - **NeMo-Relay 保留 downstream errors**（PR #42691 `85852b71d`）—— 共享 `_run_managed_with_downstream_preservation(next_call, normalize_payload, shape_response, make_managed_execute) at :288-323` 把原 provider/tool exception（连带 retry-classification signal 如 `status_code`）从 Relay's wrapper 后救出。`_original_downstream_error(exc) at :844-850` 按 class-name `_DownstreamExecutionError` + `.original` 属性 shape 探，不 import 私类；`_is_relay_wrapped_callback_error(exc, callback_error) at :853-865` 容错 `str.startswith` match `f"internal error: {callback_error.__class__.__name__}: {callback_error}"`（Relay 未来追加 traceback 后缀也不丢 unwrap）。`execute_llm at :325-363` + `execute_tool at :366-` 共享 scaffolding 去重 ~20 行。`from collections.abc import Callable` 加 at `:12`。
> - **Langfuse sanitized response usage 恢复**（`9f1c16a7f fix(langfuse)`）—— `plugins/observability/langfuse/__init__.py:849` 翻 gate 让 sanitized response (post_api_request 路径) 的 usage dict fallback 真正运行；之前 `getattr(response, "usage", None)` 总 None 致每 turn token/cost 静默 0。`:857-` fallback `elif isinstance(usage, dict) and usage:` 现从 summary dict 拿 `input_tokens / output_tokens / cache_read_tokens / ...` 翻译成 Langfuse-convention key（`"input" / "output" / "cache_read_input_tokens" / "cache_creation_input_tokens"`）。真实 response object（post_llm_call / legacy path）继续走原 branch。
>
> 详见 [[2026-06-09-update#7-anthropic-and-models]]、[[2026-06-09-update#8-models-curated]]、[[2026-06-09-update#9-observability-cluster]]。

Hermes 之前把每个 provider 的"怪癖"散落在 `run_agent.py`、`agent/auxiliary_client.py` 和 `agent/model_metadata.py` 的 if/elif 分支里。v0.13.0 用 `ProviderProfile` ABC + `plugins/model-providers/` 插件目录把它**收敛到声明式数据类**。

这是 [[provider-transport-architecture]] 的姐妹页 ——

- **Transport**：负责数据路径（convert_messages → convert_tools → build_kwargs → normalize_response），少而稳，4 个内置。
- **ProviderProfile**：声明性描述某个 provider 的差异点，多且常增减，**纯数据**。

```
┌────────────────────────────────────────────────────────────┐
│ AIAgent                                                    │
│    │                                                       │
│    │ get_provider_profile("nvidia") → ProviderProfile      │
│    │ get_transport("chat_completions") → Transport         │
│    │                                                       │
│    │ kwargs = transport.build_kwargs(profile, ...)         │
│    │ resp   = transport.send(...)                          │
│    └ resp   = transport.normalize_response(resp, profile)  │
└────────────────────────────────────────────────────────────┘
```

---

## 1. `ProviderProfile` ABC

源码：`providers/base.py:39`。

```python
@dataclass
class ProviderProfile:
    # ── Identity ─────────────────────────────────────────
    name: str
    api_mode: str = "chat_completions"   # 对应 Transport
    aliases: tuple = ()

    # ── Human-readable metadata ───────────────────────────
    display_name: str = ""               # "GMI Cloud"
    description: str = ""                # 子标题
    signup_url: str = ""                 # 注册链接

    # ── Auth & endpoints ─────────────────────────────────
    env_vars: tuple = ()                 # ("XAI_API_KEY", ...)
    base_url: str = ""
    models_url: str = ""                 # 显式覆盖；空则 base_url + "/models"
    auth_type: str = "api_key"
        # api_key | oauth_device_code | oauth_external | copilot | aws_sdk
    supports_health_check: bool = True   # doctor 是否探活

    # ── Model catalog ─────────────────────────────────────
    fallback_models: tuple = ()          # /model picker 兜底（仅 agentic 模型）
    hostname: str = ""                   # URL → provider 反查（model_metadata.py）

    # ── Client-level quirks ───────────────────────────────
    default_headers: dict[str, str]

    # ── Request-level quirks ──────────────────────────────
    fixed_temperature: Any = None        # None=透传, OMIT_TEMPERATURE=不发
    default_max_tokens: int | None = None
    default_aux_model: str = ""          # auxiliary 任务默认模型
```

钩子方法（子类可选覆盖）：

```python
def prepare_messages(self, messages) -> list:
    """codex 字段清洗后 / developer 角色翻译前调用"""
    return messages

def build_extra_body(self, *, session_id=None, **ctx) -> dict:
    """provider 特定的 extra_body"""
    return {}

def build_api_kwargs_extras(self, *, reasoning_config=None, **ctx):
    """返回 (extra_body_additions, top_level_kwargs)
       split 让 OpenRouter (extra_body.reasoning) 和 Kimi (api_kwargs.reasoning_effort)
       这种不同位置的差异由 profile 负责"""
    return {}, {}

def fetch_models(self, *, api_key=None, timeout=8.0) -> list[str] | None:
    """从 provider 的 /models 端点拉真实可用 model 列表，失败返 None"""
    ...
```

**关键设计**：profile 是**纯数据**，**不构造 client，不轮换凭证，不管流式**。所有这些仍住在 `AIAgent`，因此 profile 是无状态可序列化的描述。

---

## 2. 插件目录布局

```
plugins/model-providers/<name>/
├── __init__.py           # 调用 register_provider(profile) 一次
└── plugin.yaml           # 清单：name / kind: model-provider / version / description
```

29 个内置（HEAD 期），按字母序：

```
ai-gateway     alibaba        alibaba-coding-plan  anthropic
arcee          azure-foundry  bedrock              copilot
copilot-acp    custom         deepseek             gemini
gmi            huggingface    kilocode             kimi-coding
minimax        nous           novita               nvidia
ollama-cloud   openai-codex   opencode-zen         openrouter
qwen-oauth     stepfun        xai                  xiaomi          zai
```

### 用户覆盖

```
$HERMES_HOME/plugins/model-providers/<same-name>/
```

**同名时覆盖内置**（`register_provider` 是 last-writer-wins）。允许任意用户 monkey-patch 或替换内置 profile，**不需要 fork 仓库**。

### 向后兼容

`providers/*.py` 单文件 profile 仍被 `pkgutil.iter_modules` 发现，让 editable 安装的旧代码不破。**新写的 profile 应该用插件目录布局**。

---

## 3. 发现机制

源码：`providers/__init__.py`。

```python
def _discover_providers() -> None:
    """两个目录扫一遍：
       1. plugins/model-providers/   （仓库内置）
       2. $HERMES_HOME/plugins/model-providers/  （用户覆盖）

       每个 __init__.py 被 importlib import 一次 → 触发 register_provider()
    """

# 公共 API：
def get_provider_profile(name: str) -> ProviderProfile | None:
    """匹配 name 或 alias，未匹配返 None；不抛"""

def list_providers() -> list[ProviderProfile]:
    """所有已注册"""
```

**懒发现**：第一次 `get_provider_profile` 或 `list_providers` 被调用时才扫，不在 import 时扫 —— 这也是 v0.14.0 冷启动 −19s 的一部分。

---

## 4. 实例：怎么加一个新 Provider

```python
# plugins/model-providers/myprov/__init__.py
from providers import register_provider
from providers.base import ProviderProfile

my_provider = ProviderProfile(
    name="my-provider",
    aliases=("myp", "my"),
    display_name="My Provider",
    description="Cheap multi-model API",
    signup_url="https://my-provider.example.com/keys",
    env_vars=("MYP_API_KEY", "MYP_BASE_URL"),
    base_url="https://api.my-provider.example.com/v1",
    default_aux_model="my-cheap-7b",
)

register_provider(my_provider)
```

```yaml
# plugins/model-providers/myprov/plugin.yaml
name: my-provider
kind: model-provider
version: 1.0.0
description: ...
```

即可：

- `hermes auth add my-provider` 起作用。
- `hermes model` picker 里出现。
- `hermes doctor` 跑 /models 探活。
- model_metadata 反向查 hostname。
- transport 由 `api_mode="chat_completions"` 自动决定。

---

## 5. 与 Transport 的分工

| 维度 | Profile | Transport |
|------|---------|-----------|
| 数量 | 29 内置 + 用户 | 4 内置（anthropic / chat_completions / bedrock / codex 系列） |
| 形态 | dataclass，可序列化 | 类，含 IO |
| 持有 | provider 元信息 | 数据路径方法 |
| 增长频率 | 高（新 provider 频繁） | 低（新 API 协议才增） |
| 关键决策 | `api_mode` 字段 | — |

`api_mode` 字符串是关键的耦合点 ——

```python
profile = get_provider_profile("nvidia")        # api_mode="chat_completions"
transport = get_transport(profile.api_mode)     # ChatCompletionsTransport()
```

未来加新协议（比如 Cohere v2、Mistral le-platforme），只需：

- 加一个 Transport 子类到 `agent/transports/`
- 把 profile 的 `api_mode` 指过去

---

## 6. 与 `agent/model_metadata.py` 的关系

`agent/model_metadata.py` 的 10 级上下文长度解析链（[[smart-model-routing]]）现在**第一步**就是 `get_provider_profile`：

- profile.hostname → URL 反查
- profile.fetch_models() → 真实可用模型列表（命中 models.dev / OpenRouter manifest / Nous Portal manifest 之前作为 short-circuit）

remote model catalog manifest（v0.12.0）通过 profile.fetch_models 的远端 manifest 实现：OpenRouter / Nous Portal 的 fetch_models 现在去拉 manifest，**新模型不发版即可见**。

---

## 7. 不变量

- profile 是**纯数据** —— 不持有 client、不持有 token、不管流式、不管重试。
- `register_provider` last-writer-wins，user > bundled。
- 第一次解析时才扫盘（懒发现）。
- alias 也参与匹配。
- 任意 `api_mode` 字符串都必须对应 `agent/transports/` 里的一个注册项，否则 transport 返回 None，调用方走 legacy 兜底。
- `providers/*.py` 单文件 profile **仍支持**，但优先级低于插件目录。

---

## 7.1 OpenCode Go reasoning controls（2026-05-23+）

PR `3589960` + 跟进 `70aaa77`：`plugins/model-providers/opencode-zen/__init__.py` +63 行，把 OpenCode Go 的 reasoning controls（如 `reasoning_effort`）暴露成 provider profile field。`70aaa77` 进一步把 OpenCode Go 路径下的 Kimi 子模型 reasoning_effort 输出对齐 `KimiProfile` 形状 —— 同 provider 下不同模型分支不再分歧。

171 行新增测试（`tests/model_providers/test_opencode_go_profile.py`）pin 行为契约。

## 8. 验证

```
providers/base.py:39                 class ProviderProfile
providers/base.py:74-80              ProviderProfile.get_hostname()
providers/base.py:90                 prepare_messages
providers/base.py:100                build_extra_body
providers/base.py:113                build_api_kwargs_extras
providers/base.py:130                fetch_models
providers/__init__.py:1-30           设计 docstring
plugins/model-providers/README.md    用户文档（39 行 + 模板）
plugins/model-providers/             29 个目录
agent/transports/__init__.py:51-68   _discover_transports()
```
