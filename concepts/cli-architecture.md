---
title: CLI 架构与终端交互设计
created: 2026-04-07
updated: 2026-05-07
type: concept
tags: [architecture, cli, terminal, ux, i18n, goal]
sources: [cli.py, hermes_cli/goals.py, hermes_cli/curator.py, locales/, hermes-agent 源码分析 2026-04-07]
---

> **v2026.4.30 ~ v2026.5.7 增量**：
>
> - **`hermes -z <prompt>` 一次性模式**（v2026.4.30+）—— 非交互式 one-shot，支持 `--model` / `--provider` / `HERMES_INFERENCE_MODEL`。
> - **`hermes update --check`**（v2026.4.30+）preflight，opt-in pre-update HERMES_HOME 备份。
> - **`hermes update --yes/-y`** 跳过交互（#18261）。
> - **`/goal`**（v2026.5.7+）—— Ralph loop，跨轮目标锁定。源码 `hermes_cli/goals.py`（535 行）。`/goal <text>` 设定，`/goal resume` 继续，`/goal clear` 清除。
> - **`/new <session_name>`**（v2026.5.7+）接受可选 session 名（#19637）。
> - **`/curator archive | prune | list-archived`**（v2026.5.7+）curator 子命令。`hermes_cli/curator.py:258` 起。
> - **i18n —— 7 个 locale**：源码 `locales/` 下 `en/zh/ja/de/es/fr/uk/tr.yaml`。`display.language` 配置启用。
> - **100 条新 CLI 启动 tip**（#20168）覆盖 cron / kanban / curator / plugins 等。

# CLI 架构与终端交互设计

## 设计原理

Hermes CLI 提供完整的终端用户体验：自动补全、多行编辑、流式输出、工具调用可视化。基于 `prompt_toolkit` 和 `rich` 构建。

## 核心组件

```python
# cli.py
class HermesCLI:
    """Hermes CLI 主类"""
    
    def __init__(self):
        self.agent = None
        self.config = load_cli_config()
        self.session_db = SessionDB(...)
        self.todo_store = TodoStore()
    
    def run(self):
        """主循环"""
        while True:
            user_input = self._get_input()  # prompt_toolkit 输入
            if user_input.startswith("/"):
                self._handle_command(user_input)
            else:
                self._handle_message(user_input)
```

## 输入系统

```python
from prompt_toolkit import PromptSession
from prompt_toolkit.history import FileHistory
from prompt_toolkit.auto_suggest import AutoSuggestFromHistory

session = PromptSession(
    history=FileHistory("~/.hermes/input_history"),
    auto_suggest=AutoSuggestFromHistory(),
    completer=SlashCommandCompleter(),  # 定义在 hermes_cli/commands.py
)

user_input = session.prompt(get_active_prompt_symbol())  # 提示符号通过 skin engine 配置
```

### 斜杠命令补全

```python
class SlashCommandCompleter(Completer):
    def get_completions(self, document, complete_event):
        text = document.text_before_cursor
        if text.startswith("/"):
            for cmd_name, cmd_def in COMMANDS.items():
                if cmd_name.startswith(text[1:]):
                    yield Completion(cmd_name, start_position=-len(text[1:]))
```

## 显示系统

### KawaiiSpinner

```python
# agent/display.py
class KawaiiSpinner:
    """动画加载指示器"""
    
    SPINNERS: dict          # 9 种命名动画集 ('dots', 'bounce', 'grow', ...)
    KAWAII_WAITING: list     # 10 个多字符颜文字
    KAWAII_THINKING: list    # 15 个多字符颜文字
    THINKING_VERBS: list    # 15 个动词 ("pondering", "contemplating", "musing", "cogitating", "ruminating", ...)
    
    def show(self, message: str):
        """显示加载动画"""
        # 使用 Rich 面板和动画
```

### 工具调用预览

```python
def build_tool_preview(tool_name: str, args: dict) -> str:
    """构建工具调用预览"""
    preview = f"🔧 {tool_name}("
    for key, value in list(args.items())[:3]:
        preview += f"\n  {key}={preview_value(value)},"
    preview += "\n)"
    return preview

def get_cute_tool_message(tool_name: str) -> str:
    """获取可爱的工具执行消息"""
    emoji = _get_tool_emoji(tool_name)
    return f"{emoji} Calling {tool_name}..."
```

## Skin 引擎

```python
# hermes_cli/skin_engine.py
@dataclass
class SkinConfig:
    """皮肤配置数据类"""
    ...

# 模块级函数（非类）
def init_skin_from_config(): ...
def get_active_skin() -> SkinConfig: ...
def list_skins() -> list: ...
def set_active_skin(name: str): ...

# 配置示例
# ~/.hermes/config.yaml
display:
  skin: "default"  # 或自定义皮肤名称
```

## v0.12.0 新增 CLI 能力（2026-04-30）

### 一次性运行模式（`hermes -z`）

```bash
hermes -z "summarize the diff in the current branch" --model openai:gpt-5.5
HERMES_INFERENCE_MODEL=anthropic:claude-opus-4-7 hermes -z "..."
```

源：`hermes_cli/_parser.py:97`。`-z/--oneshot` 加载工具/记忆/`AGENTS.md`/规则、自动跳过审批、只输出最终回复（无 banner、spinner、tool preview、session_id 行），适合脚本和管道。

### 新增/调整斜杠命令

| 命令 | 作用 | 备注 |
|------|------|------|
| `/busy` | 切换 busy 输入模式（busy / queue / steer） | v0.12.0 |
| `/btw` | `/background` 别名 | v0.12.0 |
| `/reload` | TUI 中热加载 `.env` | v0.12.0 |
| `/reload-skills` | 重新扫描技能目录 | v2026.4.23 引入；v0.12.0 在 TUI 也走 live exec |
| `/mouse` | 在 ConPTY 上手动启停鼠标支持，修复 WSL2 ghost-mouse | v0.12.0 |
| `/topic on\|off` | Telegram DM 主题模式（gateway 命令） | v0.12.0 之后 |
| `/provider` / `/plan` | **已删除** | v0.12.0 ([#15047](https://github.com/NousResearch/hermes-agent/pull/15047)) |

### 更新与备份

- `hermes update --check` 预飞检查（不实际更新）
- `HERMES_HOME` 备份默认 opt-in；`backup_keep` 至少保留 1 份
- `checkpoints/` 与 SQLite WAL/SHM/journal sidecar 自动排除

### `/fast` 收敛

`/fast` 仅在 Anthropic Opus 4.6 上有效（fast mode 由 Anthropic API 限定）。源：`agent/anthropic_adapter.py:79 _FAST_MODE_SUPPORTED_SUBSTRINGS = ("opus-4-6", "opus-4.6")`，并通过 `_supports_fast_mode()` 守卫。其他模型设置 `extra_body["speed"]="fast"` 会被剥除，避免 400。

## 优越性分析

### 与其他 Agent 框架对比

| 特性 | Hermes | Claude Code | Codex CLI |
|------|--------|-------------|-----------|
| 斜杠命令补全 | ✅ 自动 | ✅ | ❌ |
| 多行编辑 | ✅ | ✅ | ✅ |
| 输入历史 | ✅ 文件持久化 | ✅ | ✅ |
| 动画加载 | ✅ KawaiiSpinner | ✅ 简单 | ✅ 简单 |
| 主题系统 | ✅ Skin Engine | ❌ | ❌ |
| 工具调用预览 | ✅ 格式化 | ✅ | ❌ |

## 相关页面

- [[configuration-and-profiles]] — 配置管理与 Profile 系统
- [[hook-system-architecture]] — Hook 与插件扩展系统
- [[session-search-and-sessiondb]] — 会话搜索与 SessionDB
- [[voice-mode-architecture]] — 语音模式（Push-to-talk → STT → TTS）
- [[skin-engine]] — 皮肤/主题自定义
- [[context-references]] — @file/@diff/@url 引用系统
- [[worktree-isolation]] — Git Worktree 并行隔离
- [[code-execution-sandbox]] — 代码执行沙箱
- [[persistent-goals-ralph-loop]] — `/goal` 持久目标 Ralph 循环
- [[kanban-multi-profile-board]] — `/kanban` 跨 profile 协作板

## 相关文件

- `cli.py` — CLI 主类
- `hermes_cli/main.py` — 入口点和子命令
- `hermes_cli/commands.py` — 斜杠命令定义
- `hermes_cli/dump.py` — `hermes dump` 环境摘要（纯文本，用于调试/提 issue）
- `agent/display.py` — 显示系统
- `hermes_cli/skin_engine.py` — 皮肤引擎
