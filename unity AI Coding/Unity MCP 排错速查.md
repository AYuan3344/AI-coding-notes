---
tags:
  - Unity
  - MCP
  - 排错
  - 速查
date: 2026-09-21
状态: 可用
---

# Unity MCP 排错速查

配套：[[Unity MCP 安装记录]]

## 报错 → 原因 → 处理

| 现象 | 根因 | 处理 |
| --- | --- | --- |
| `Python not found in PATH` | ① PATH 里只有商店占位 stub；② `uv` 所在目录是后加的，**Unity 启动早于 PATH 变更** | 装独立 Python（安装器自动前置 PATH）→ **重启 Unity** |
| uv 显示 `Not Found` | `~/.local/bin` 不在 Unity 进程的 PATH 里 | 重启 Unity；或在窗口里用 **Choose UV Install Location** 手动指 `uvx.exe` |
| Unity 里 Add package from git URL 失败 | git 智能协议 / codeload 被网络阻断，且本机无 git | 改用 [[Unity MCP 安装记录#Step 3 · 安装 Unity 包（重打包 tarball 法）\|重打包 tarball 法]] |
| package manifest 报 `Cannot resolve file:` | tarball 路径变了 / 被删 | 重建 tarball 并修正 `manifest.json` 路径 |
| 客户端连不上 / 无工具响应 | Bridge 未启动；或客户端未信任该 MCP | Unity 窗口点 **Start Bridge**；客户端连接器里点**信任**后重开会话 |
| Auto-Setup 后仍连不上 | 端口被占 / 服务器版本不匹配 | 看 **HTTP Server Command** 折叠项里的实际 `uvx` 命令，手动跑一遍看报错 |
| 改了 PATH 但 Unity 没反应 | 环境变量进程启动时读取，改完必须重启 | 关掉**所有** Unity 实例再开 |

> [!important] 一条铁律
> **任何 PATH / 环境变量相关改动，都要「完全重启 Unity」才生效**。窗口里的 `Verify again` 只能重跑检测，不能刷新进程环境。

## 分层诊断（从下往上查）

```powershell
# 第 1 层：uv 是否可用
uv --version                       # 期望 uv x.y.z
where uv                           # 期望 ~\.local\bin\uv.exe

# 第 2 层：MCP 服务器能否启动
mcp-for-unity -h                   # 期望打印 usage（--transport stdio|http 等）
uvx --from mcpforunityserver mcp-for-unity -h

# 第 3 层：Python 是否被识别
python --version                   # 期望 Python 3.13.x（不能是商店 stub）
where python                       # 官方版必须排在 WindowsApps 之前
py -0p                             # 列出已注册解释器
uv python list                     # 看 uv 能看到哪些解释器

# 第 4 层：Unity 侧包是否就绪
# F:\f8th\trunk\config\3dclient\Packages\packages-lock.json  → source: local-tarball
# F:\f8th\trunk\config\3dclient\Library\ScriptAssemblies\     → MCPForUnity.Editor.dll

# 第 5 层：桥接是否在跑
netstat -ano | findstr "8080 6500"
tasklist | findstr /I "uvx mcp-for-unity Unity"
```

## 关键认知（少走弯路）

- **Unity 检测 Python 的顺序**（源码 `WindowsPlatformDetector.cs`）：
  `python3.exe/python.exe`（PATH） → `where` → `uv python list` 兜底
- **Unity 检测时还会额外扫描**这些目录，PATH 没配也可能命中：
  `%LOCALAPPDATA%\Programs\Python`、`%ProgramFiles%\Python3*`、`%APPDATA%\npm`、`~\.local\bin`
- **`ExecPath.TryRun` 用父进程 PATH 解析可执行文件**，所以给子进程改 PATH 不等于父进程能找到该程序
- **Windows 上 stdio 配置必须写 `uvx.exe` 绝对路径**，不能只写 `uvx`
- **VS Code 的配置键是 `servers`**，WorkBuddy / Cursor 等用 `mcpServers`，别抄错

## 客户端配置文件位置

| 客户端 | 路径 | 顶层键 |
| --- | --- | --- |
| WorkBuddy | `~\.workbuddy\mcp.json` | `mcpServers` |
| VS Code | `%APPDATA%\Code\User\mcp.json` | `servers` |
| Cursor | `%USERPROFILE%\.cursor\mcp.json` | `mcpServers` |

## 手动起一个服务器（排查用）

```powershell
# stdio：给客户端用，手工跑会「卡住」属正常（在等 stdin）
uvx --from mcpforunityserver mcp-for-unity --transport stdio

# http：手工排查更方便
uvx --from mcpforunityserver mcp-for-unity --transport http --http-port 8080
# 然后浏览 http://localhost:8080/mcp
```

Unity 窗口里的 **HTTP Server Command** 折叠项会显示它实际会执行的完整命令，可直接复制出来对照。
