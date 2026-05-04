---
title: MCP 集成与插件系统
created: 2026-04-07
updated: 2026-04-07
type: concept
tags: [architecture, mcp, plugins, extensibility]
sources: [hermes-agent 源码分析 2026-04-07]
---

# MCP 集成与插件系统

## 设计原理

Hermes 通过 **MCP（Model Context Protocol）** 和**插件系统**实现可扩展性，允许连接外部工具和自定义行为。

## MCP 集成

```python
# tools/mcp_tool.py (~2176 行)

class MCPServerTask:
    """MCP 服务器任务"""
    
    def __init__(self, config: dict):
        self.servers = {}
        self.tools = {}
    
    async def connect_server(self, name: str, config: dict):
        """连接 MCP 服务器"""
        transport = config.get("transport", "stdio")
        
        if transport == "stdio":
            process = await asyncio.create_subprocess_exec(
                *config["command"],
                stdin=asyncio.subprocess.PIPE,
                stdout=asyncio.subprocess.PIPE,
            )
            self.servers[name] = {
                "process": process,
                "transport": transport,
            }
        elif transport == "http":
            self.servers[name] = {
                "url": config["url"],
                "transport": transport,
            }
        
        # 获取服务器工具
        tools = await self._list_tools(name)
        for tool in tools:
            self.tools[f"{name}:{tool['name']}"] = tool
    
    async def call_tool(self, tool_name: str, args: dict) -> dict:
        """调用 MCP 工具"""
        server_name, tool_name = tool_name.split(":", 1)
        server = self.servers[server_name]
        
        if server["transport"] == "stdio":
            return await self._call_stdio_tool(server, tool_name, args)
        elif server["transport"] == "http":
            return await self._call_http_tool(server, tool_name, args)
```

### MCP OAuth 支持

```python
# tools/mcp_oauth.py

async def authenticate_mcp_server(server_config: dict) -> dict:
    """MCP 服务器 OAuth 认证"""
    auth_type = server_config.get("auth", {}).get("type")
    
    if auth_type == "oauth":
        # 实现 OAuth 流程
        auth_url = server_config["auth"]["url"]
        client_id = server_config["auth"]["client_id"]
        # ...
        return {"access_token": token, "expires_at": expires}
    
    elif auth_type == "api_key":
        return {"api_key": server_config["auth"]["api_key"]}
    
    return {}
```

## 插件系统

```python
# hermes_cli/plugins.py

class Plugin:
    """插件基类"""
    
    name: str = ""
    version: str = "1.0.0"
    description: str = ""
    
    def on_load(self):
        """插件加载时调用"""
        pass
    
    def on_unload(self):
        """插件卸载时调用"""
        pass

# 钩子系统
_HOOKS = {
    "on_session_start": [],
    "pre_llm_call": [],
    "post_llm_call": [],
    "on_tool_call": [],
    "on_session_end": [],
}

def register_hook(hook_name: str, callback: callable):
    """注册钩子回调"""
    if hook_name in _HOOKS:
        _HOOKS[hook_name].append(callback)

def invoke_hook(hook_name: str, **kwargs) -> list:
    """调用钩子"""
    results = []
    for callback in _HOOKS.get(hook_name, []):
        try:
            result = callback(**kwargs)
            results.append(result)
        except Exception as e:
            logger.warning(f"Hook {hook_name} failed: {e}")
    return results
```

### 内存插件

```python
# plugins/memory/__init__.py

class MemoryPlugin(Plugin):
    """内存插件（Honcho 集成）"""
    
    name = "honcho-memory"
    
    def on_session_start(self, session_id: str, **kwargs):
        """会话开始时预热缓存"""
        self._warm_cache(session_id)
    
    def pre_llm_call(self, user_message: str, **kwargs):
        """LLM 调用前注入上下文"""
        context = self._fetch_context(user_message)
        return {"context": context}
    
    def on_session_end(self, messages: list, **kwargs):
        """会话结束时持久化"""
        self._persist_session(messages)
```

## 插件 CLI

```bash
# 插件管理
hermes plugins list           # 列出已安装插件
hermes plugins install <name> # 安装插件
hermes plugins remove <name>  # 移除插件
hermes plugins update <name>  # 更新插件
```

## v0.12.0 内置插件清单

`/tmp/hermes-agent/plugins/` 目录新增/扩展：

| 插件 | 路径 | 说明 |
|------|------|------|
| `spotify` | `plugins/spotify/` | 7 个 native tool（play / search / queue / playlists / devices）+ PKCE OAuth + 交互式 setup 向导，可在 `hermes tools` 中切换 |
| `google_meet` | `plugins/google_meet/` | 加入会议、转录、说话、跟进；OpenAI realtime transport + Node bot server，全管道 bundled |
| `observability/langfuse` | `plugins/observability/langfuse/` | Langfuse observability bundled |
| `hermes-achievements` | `plugins/hermes-achievements/` | 扫描全部 session 历史输出"成就" |
| `kanban` | `plugins/kanban/` | 看板插件 + per-platform home-channel 通知 toggle（v0.12.0 #19864） |
| `image_gen` | `plugins/image_gen/` | 图像生成 |
| `context_engine` | `plugins/context_engine/` | Context Engine 插件化 |
| `memory` | `plugins/memory/` | Memory provider |
| `disk-cleanup` / `example-dashboard` / `strike-freedom-cockpit` | … | 例示/工具型插件 |

平台插件目录 `plugins/platforms/`：

- `irc/`：v2026.4.23 加入，参考实现
- `teams/`：v0.12.0 新增（**第 19 个消息平台**），通过 `register(ctx)` 调 `ctx.register_platform(...)` 注入，最大消息长度 28KB

### 新增 Hook 类型（v0.12.0）

- `pre_gateway_dispatch`（#15050）：插件可在 gateway 派发前拦截
- `pre_approval_request` / `post_approval_response`（#16776）：审批请求前后注入
- `post_tool_call` 增加 `duration_ms` 字段（#15429，灵感来自 Claude Code 2.1.119）

### 直接 URL 安装

- 技能：`hermes skills install <https://...>`（#16323）
- NixOS module：`extraPackages` + 声明式插件安装（@alt-glitch，#15953/#17047）

## 配置

```yaml
# ~/.hermes/config.yaml
mcp_servers:
  filesystem:
      command: ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/root/work"]
    github:
      command: ["npx", "-y", "@modelcontextprotocol/server-github"]
      env:
        GITHUB_PERSONAL_ACCESS_TOKEN: "${GITHUB_TOKEN}"

plugins:
  enabled:
    - honcho-memory
    - custom-plugin
```

## 优越性分析

### 与其他 Agent 框架对比

| 特性 | Hermes | Cursor | Claude Code |
|------|--------|--------|-------------|
| MCP 支持 | ✅ 完整 | ✅ | ✅ |
| MCP OAuth | ✅ | ❌ | ✅ |
| 插件系统 | ✅ 钩子系统 | ❌ | ❌ |
| 自定义工具 | ✅ 注册表 | ❌ | ❌ |
| 插件 CLI | ✅ | N/A | N/A |

## 相关页面

- [[tool-registry-architecture]] — 插件通过 registry.register() 注册工具
- [[hook-system-architecture]] — 插件钩子系统与网关事件钩子互补
- [[model-tools-dispatch]] — MCP 工具通过 discover 机制集成到编排层

## 相关文件

- `tools/mcp_tool.py` — MCP 服务器任务
- `tools/mcp_oauth.py` — MCP OAuth
- `hermes_cli/plugins.py` — 插件系统
- `plugins/` — 插件目录
