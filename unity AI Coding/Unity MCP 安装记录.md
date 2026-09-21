---
tags:
  - Unity
  - MCP
  - AI-Coding
  - 环境搭建
date: 2026-09-21
项目: F:\f8th\trunk\config\3dclient
Unity版本: 2022.3.13f1
状态: 已安装完成
---

# Unity MCP 安装记录

> [!info] 这是什么
> 把 **MCP for Unity**（CoplayDev/unity-mcp）装进 Unity 编辑器，让 AI 客户端（WorkBuddy / VS Code Copilot 等）通过 [MCP 协议](https://modelcontextprotocol.io/introduction) 用自然语言操作编辑器：建场景、改 C# 脚本、管资源、跑测试、出包。
>
> 相关：[[Unity MCP 排错速查]]

## 0. 结论速览

| 组件 | 版本 | 位置 |
| --- | --- | --- |
| Unity 包 | `com.coplaydev.unity-mcp` **10.2.1-beta.6** | `F:\f8th\trunk\config\3dclient\Library\PackageCache\` |
| 包安装方式 | **本地 tarball**（`file:` 引用） | `C:\Users\18807\.workbuddy\unity-packages\com.coplaydev.unity-mcp-10.2.1-beta.6.tgz` |
| uv 运行时 | **0.12.17** | `C:\Users\18807\.local\bin\uv.exe` / `uvx.exe` |
| MCP 服务器 | PyPI `mcpforunityserver` | `uv tool install` → `C:\Users\18807\.local\bin\mcp-for-unity.exe` |
| Python | **3.13.15**（官方独立安装） | `C:\Users\18807\AppData\Local\Programs\Python\Python313` |
| WorkBuddy 配置 | — | `C:\Users\18807\.workbuddy\mcp.json` |
| VS Code 配置 | — | `%APPDATA%\Code\User\mcp.json` |

> [!success] 更正说明（2026-09-21 14:20 复核）
> 初版笔记曾判断「GitHub 被阻断、本机无 git」，**该结论已推翻**。复核实测：本机装的是 **git 2.55.0.windows.5**（在系统 PATH），`git ls-remote https://github.com/CoplayDev/unity-mcp.git beta` **可正常返回**，`github.com` 与 `codeload.github.com` 均 200。
> 当初的报错来自 WorkBuddy 沙箱给子进程注入的本地代理（`HTTP_PROXY=127.0.0.1:54562`），是**工具环境假象**，不是机器网络问题。
> 因此：**官方 UPM git URL 方式是可行且更推荐的**；本次采用的 tarball 法依然有效，作为离线 / 免 git 的备选保留。详见 [[#坑 1（已更正）— 一次被工具环境误导的误判]]。

---

## 1. 背景与目标

- 目标工程：`F:\f8th\trunk\config\3dclient`（Unity 2022.3.13f1）
- 需求版本：`beta` 分支最新（`10.2.1-beta.6`）
- 客户端：WorkBuddy + VS Code
- 本包官方要求：**Unity ≥ 2021.3 LTS**，Python **≥ 3.10**（经 `uv` 管理）

> [!note] 同机其他工程为何不装
> `F:\f2btw`、`G:\f2btw` 是 **Unity 2018.4.21f1**，低于版本门槛，装不了。`F:\f8b`、`F:\f8c` 同为 2022.3.13f1，需要时可照样操作。

---

## 2. 环境前置检查（安装前实测）

| 检查项 | 初始状态 | 说明 |
| --- | --- | --- |
| `uv` / `uvx` | ❌ 未安装 | MCP 服务器的运行基础 |
| `python` | ⚠️ 只有微软商店占位 stub | 跑不出版本号 |
| `git` | ✅ 已装 **2.55.0.windows.5** | `C:\Program Files\Git`，`cmd` 已在系统 PATH |
| 版本管理 | SVN + Git 并存 | SlikSvn 在 `C:\Program Files\SlikSvn\bin` |
| `~/.workbuddy/mcp.json` | ❌ 不存在 | 从未配置过 MCP |
| `%APPDATA%\Code\User\mcp.json` | ❌ 不存在 | — |

---

## 3. 三个关键坑（先看这里，能省几小时）

### 坑 1（已更正）— 一次被工具环境误导的误判

> [!danger] 别重复踩这个思维陷阱
> **现象**：安装时用 shell 测试 GitHub，`codeload` 返回 502、git 协议连接被重置、`uv` 的 GitHub Release 下载失败 → 顺势判断「GitHub 被墙、本机没 git」，绕道用了重打包 tarball。
>
> **真相**：这些报错来自 **WorkBuddy 沙箱给子进程注入的本地代理**（`HTTP_PROXY=http://127.0.0.1:54562`）。复核时直接调用真实环境验证，全部正常：

| 通道 | 初判（沙箱内） | 复核（真实环境） |
| --- | --- | --- |
| `git --version` | ❌ 找不到 | ✅ **2.55.0.windows.5** |
| `git ls-remote .../unity-mcp.git beta` | ❌ 连接重置 | ✅ 返回 `refs/heads/beta` |
| `github.com` 页面 | — | ✅ 200 |
| `codeload.github.com/...tar.gz` | ❌ 502 | ✅ 200（仅慢） |
| `api.github.com` | ✅ | ✅ |
| `pypi.org` / `package.openupm.com` | ✅ | ✅ |

> [!tip] 教训
> **工具沙箱的网络代理会污染网络类结论。** 凡是「网络不通」的判断，都要意识到它可能只是执行环境的代理行为；下结论前最好用一条最直接的命令（如 `git ls-remote`）在真实环境复核，而不是从 HTTP 状态码倒推。

**结论**：官方 `Add package from git URL` 方式**可行**，且比 tarball 法更省心（无绝对路径、可随分支更新）。
**本次仍采用 tarball 法**（当时基于误判），两种方式都记录在 [[#Step 3 · 安装 Unity 包]]，可随时切换。

> [!note] 顺带确认过的另一条路
> OpenUPM（`openupm add com.coplaydev.unity-mcp`）最高只有 **10.2.0**，**没有 beta 版**，如需 beta 只能走 git URL 或 tarball。

### 坑 2 — Unity 报 `Python not found in PATH`

读了包内源码 `Editor/Dependencies/PlatformDetectors/WindowsPlatformDetector.cs`，检测顺序是：

```
python3.exe / python.exe（PATH） → where 命令 → uv python list 兜底
```

根因是**两个问题叠加**：

1. 系统 PATH 里只有微软商店的**占位 stub**，执行不出 `Python x.y.z`，前两步判失败；
2. 兜底依赖 `uv`，而 `uv.exe` 所在的 `~/.local/bin` 是**安装 uv 时才加进用户 PATH** 的 —— 而 Unity 进程在**那之前就已启动**，进程环境变量是旧快照，什么都看不到。

> [!important] 关键认知
> `ExecPath.TryRun` 用**父进程 PATH** 解析可执行文件，所以 **环境变量只在进程启动时读取** —— 改完 PATH 必须**重启 Unity** 才生效，`Verify again` 按钮救不了。

**对策**：装一个真正的独立 Python，并让安装器自动前置 PATH（见 Step 5）。

### 坑 3 — 本次沙箱的 bash 缺基础命令

`dirname` / `head` / `tail` / `cp` / `grep` 全部 `command not found`，只有 `echo`、`where` 可用。
**对策**：所有文件与网络操作改用 **Python 脚本**执行（`urllib` 下载、`shutil` 复制、`winreg` 改 PATH），可靠得多。

---

## 4. 安装步骤（可复现）

### Step 1 · 安装 uv

官方脚本走 `iex` 被安全策略拦截、GitHub Release 又被网络阻断，因此改从 **PyPI** 装进隔离环境，再把二进制复制到标准位置：

```powershell
# 1) 建隔离环境并安装 uv
& "C:\Users\18807\.workbuddy\binaries\python\versions\3.13.12\python.exe" -m venv "C:\Users\18807\.workbuddy\binaries\python\envs\default"
& "C:\Users\18807\.workbuddy\binaries\python\envs\default\Scripts\python.exe" -m pip install --upgrade uv

# 2) 复制到标准目录（同时加入用户 PATH）
#    ~/.local/bin 已被 uv 的官方布局约定识别
```

验证：`uv --version` → `uv 0.12.17`

### Step 2 · 安装 MCP 服务器

```powershell
uv tool install mcpforunityserver
```

产物：`C:\Users\18807\.local\bin\mcp-for-unity.exe`（同时会注册 `unity-mcp.exe`）
支持的 transport：`stdio` | `http`，端口参数见 `mcp-for-unity -h`。

### Step 3 · 安装 Unity 包

#### 方式 A · 官方推荐：Package Manager 从 git URL 安装 ✅

本机 git 与网络均正常（见 [[#坑 1（已更正）— 一次被工具环境误导的误判]]），这是**首选**方式。

**图形界面**：`Window → Package Manager → + → Add package from git URL`，填入：

```
https://github.com/CoplayDev/unity-mcp.git?path=/MCPForUnity#beta
```

> [!tip] 锚点怎么选
> `#beta` = beta 分支最新（本次为 `10.2.1-beta.6`）；`#v10.0.0` = 正式 tag（2026-06-30 发布，稳定）；`#main` = 主干。
> 注意 `?path=/MCPForUnity` **不能省** —— 仓库里只有这个子目录是 UPM 包。

**改 manifest 等价写法**（`Packages/manifest.json`，**先备份**）：

```json
{
  "dependencies": {
    "com.coplaydev.unity-mcp": "https://github.com/CoplayDev/unity-mcp.git?path=/MCPForUnity#beta"
  }
}
```

优点：无绝对路径、可随分支更新、不污染版本库。

#### 方式 B · 备选：重打包 tarball（本次采用）

适用场景：需要**离线/内网**安装、或确实无法使用 git 时。

思路：**下载仓库源码包，自己拼一个符合 UPM 规范的 tarball。**

```python
# 伪代码，核心逻辑
data = urlopen("https://api.github.com/repos/CoplayDev/unity-mcp/tarball/beta").read()
src  = tarfile.open(fileobj=io.BytesIO(data))
# 只保留 <root>/MCPForUnity/ 子树，并改写到 package/ 前缀下
# 输出：com.coplaydev.unity-mcp-10.2.1-beta.6.tgz（718 个文件，约 640 KB）
```

要点：
- tarball 必须是 **npm 包规范**：所有文件位于 `package/` 前缀下，`package/package.json` 为根；
- 不能直接把 GitHub 的**仓库 tarball** 丢给 UPM（它的根目录名是 `owner-repo-sha`，UPM 不认）；
- 打包后校验 `package.json` 的 `name` / `version` 是否正确。

保存位置：`C:\Users\18807\.workbuddy\unity-packages\com.coplaydev.unity-mcp-10.2.1-beta.6.tgz`

然后把 tarball 写进 `Packages/manifest.json`（**先备份**，这就是**当前已安装状态**）：

```json
{
  "dependencies": {
    "com.coplaydev.unity-mcp": "file:C:/Users/18807/.workbuddy/unity-packages/com.coplaydev.unity-mcp-10.2.1-beta.6.tgz"
  }
}
```

打开 Unity 后自动解析并编译，可获得：

- `Library/PackageCache/com.coplaydev.unity-mcp@<hash>`
- `Library/ScriptAssemblies/MCPForUnity.Editor.dll`、`MCPForUnity.Runtime.dll`

### Step 4 · 配置 MCP 客户端

> [!tip] Windows 上用 stdio 必须写 uvx 的**绝对路径**
> 官方文档明确说明：Windows 下 stdio 模式的 `command` 要填 `uvx.exe` 完整路径，不能只写 `uvx`。

**WorkBuddy** → `C:\Users\18807\.workbuddy\mcp.json`

```json
{
  "mcpServers": {
    "unityMCP": {
      "command": "C:/Users/18807/.local/bin/uvx.exe",
      "args": ["--from", "mcpforunityserver", "mcp-for-unity", "--transport", "stdio"]
    }
  }
}
```

**VS Code** → `%APPDATA%\Code\User\mcp.json`（注意是 `servers` 键，不是 `mcpServers`）

```json
{
  "servers": {
    "unityMCP": {
      "type": "stdio",
      "command": "C:/Users/18807/.local/bin/uvx.exe",
      "args": ["--from", "mcpforunityserver", "mcp-for-unity", "--transport", "stdio"]
    }
  }
}
```

### Step 5 · 安装独立 Python（解决坑 2）

用官方安装器做**当前用户**静默安装（无需管理员）：

```powershell
python-3.13.15-amd64.exe /quiet InstallAllUsers=0 PrependPath=1 `
  Include_test=0 Include_doc=0 Include_launcher=1 InstallLauncherAllUsers=0 `
  AssociateFiles=0 Shortcuts=0
```

- 安装包来源：`https://www.python.org/ftp/python/3.13.15/python-3.13.15-amd64.exe`（官网可直连，偶发 SSL 中断，需重试；备选华为云 / npmmirror 镜像）
- 结果：`Python 3.13.15` + `pip 26.2.1` + `py` 启动器，安装器已自动把
  `Python313\Scripts`、`Python313`、`Launcher` 三个目录**前置**到用户 PATH

### Step 6 · Unity 内激活

1. **重启 Unity**（必须！见坑 2）
2. `Window → MCP for Unity`（快捷键 `Ctrl+Shift+M`）
3. Python / uv 两行变绿后点 **Auto-Setup**
4. Unity Bridge 若显示 Stopped，点 **Start Bridge**
5. 到客户端的连接器管理里**信任** `unityMCP`，重开会话

---

## 5. 验证清单

- [x] `uv --version` → `uv 0.12.17`
- [x] `mcp-for-unity -h` 能打印 usage（说明服务器可启动）
- [x] `python --version` → `Python 3.13.15`（`where python` 官方版排在商店 stub 之前）
- [x] `py -0p` 列出 3.13
- [x] `packages-lock.json` 中 `source: local-tarball`，包版本 `10.2.1-beta.6`
- [x] `MCPForUnity.Editor.dll` 已生成（编译无错）
- [ ] Unity 窗口内 Python / uv 双绿、Bridge 显示 Running ← *需重启 Unity 后确认*
- [ ] 客户端连接成功，能响应「创建一个 Cube 并加 Rigidbody」

---

## 6. 路径与命令速查

| 用途 | 路径 / 命令 |
| --- | --- |
| uv / uvx | `C:\Users\18807\.local\bin\uv.exe`、`uvx.exe` |
| MCP 服务器入口 | `C:\Users\18807\.local\bin\mcp-for-unity.exe` |
| 启动服务器（手动） | `uvx --from mcpforunityserver mcp-for-unity --transport stdio` |
| UPM 包 tarball | `C:\Users\18807\.workbuddy\unity-packages\*.tgz` |
| 官方 Python | `C:\Users\18807\AppData\Local\Programs\Python\Python313` |
| Unity 包缓存 | `F:\f8th\trunk\config\3dclient\Library\PackageCache\` |
| 客户端配置 | `~\.workbuddy\mcp.json`、`%APPDATA%\Code\User\mcp.json` |
| 编辑器窗口 | `Window → MCP for Unity`（`Ctrl+Shift+M`） |

---

## 7. 回滚 / 卸载

1. **撤掉工程依赖**：`Packages/manifest.json` 删除 `com.coplaydev.unity-mcp` 一行
   （原始备份：`Packages/manifest.json.bak-unitymcp`），再删 `Library/PackageCache/com.coplaydev.unity-mcp@*`
2. **卸 MCP 服务器**：`uv tool uninstall mcpforunityserver`
3. **撤客户端配置**：删各自 mcp.json 里的 `unityMCP` 节点
4. **卸 Python**：`设置 → 应用 → Python 3.13.15 → 卸载`，并清理用户 PATH 中三条 Python 目录

---

## 8. 遗留事项与注意

> [!warning] 提交 SVN 前请注意
> 方式 B 写进 `manifest.json` 的是**绝对本地路径**的 `file:` 引用，提交会让同事的工程解析失败。
> 建议二选一：**提交前撤掉该行**；或直接切到 [[#方式 A · 官方推荐：Package Manager 从 git URL 安装 ✅|方式 A]] 的 git URL 写法（本机 git 与网络都正常，一步解决）。

- **本机网络正常**：`github.com`、`git` 协议、`codeload`、`pypi.org` 均实测可用；仅 **WorkBuddy 工具沙箱**会给命令注入本地代理，可能导致工具侧偶发网络报错（见坑 1）
- **方式可随时切换**：把 manifest 那一行从 `file:...tgz` 换成 git URL 即可，无需重装
- **版本错位属正常**：Unity 包为 `10.2.1-beta.6`，PyPI 服务器包目前最高 `10.2.0`
- **beta 分支为开发版**：追求稳定可切到正式 tag `v10.0.0`
- **PATH 依赖**：客户端配置里写死的是 `~/.local/bin/uvx.exe`，若该文件被移动需同步更新两处配置
- **pip 镜像未配置**：如需加速可另行设置清华 / 腾讯源（不影响 MCP 使用）
