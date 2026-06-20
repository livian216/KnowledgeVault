# Windows 11 安装 Claude Code 并接入 DeepSeek V4 Pro 教程

> 版本日期：2026-06-20  
> 适用对象：Windows 11 用户，希望在本机 Claude Code 中使用 DeepSeek V4 Pro API，或者通过 CC Switch 管理多个模型源。  
> 推荐结论：如果只使用 DeepSeek，优先使用“第一章：Claude Code 直接连接 DeepSeek”；如果后期需要频繁切换 DeepSeek、Claude 官方、Kimi、硅基流动、第三方中转等多个模型源，再使用“第二章：Claude Code 通过 CC Switch 连接 DeepSeek”。

---

## 资料依据与可行性说明

本教程不是凭空整理，主要依据如下资料交叉整理：

1. Anthropic 官方 Claude Code 文档：Windows PowerShell 原生安装命令、WinGet 安装命令、Git for Windows 建议、环境变量配置方式。  
   - https://code.claude.com/docs/en/overview  
   - https://code.claude.com/docs/en/env-vars

2. DeepSeek 官方 API 文档：DeepSeek Anthropic API、Claude Code 接入 DeepSeek 的环境变量、`deepseek-v4-pro[1m]` 与 `deepseek-v4-flash` 的模型配置方式。  
   - https://api-docs.deepseek.com/quick_start/agent_integrations/claude_code  
   - https://api-docs.deepseek.com/guides/anthropic_api

3. CC Switch 官方 GitHub README：Windows 安装包、Provider 管理、一键切换、Claude Code 支持、配置写入与热切换机制说明。  
   - https://github.com/farion1231/cc-switch

4. CC Switch 社区问题记录：Claude Code + DeepSeek V4 在某些版本组合下可能出现 thinking mode / reasoning_content 兼容问题。因此，CC Switch 方案虽然方便，但要保留版本兼容性检查。  
   - https://github.com/farion1231/cc-switch/issues/3246

需要特别说明：**Claude Code 直接连接 DeepSeek 是 DeepSeek 官方文档明确给出的方案，可信度最高；CC Switch 是第三方社区工具，适合多模型切换，但不是必须组件。**

---

## 第一章：Claude Code 直接连接 DeepSeek

这一章讲的是最稳妥的方案：在 Windows 11 上安装 Claude Code，然后通过环境变量让 Claude Code 直接访问 DeepSeek 的 Anthropic 兼容接口。

整体链路如下：

```text
Claude Code CLI  →  DeepSeek Anthropic API  →  deepseek-v4-pro[1m]
```

这种方式不经过额外本地代理，也不依赖 CC Switch。优点是链路短、变量少、出问题时容易排查；缺点是如果以后经常切换多个模型源，就需要手动改环境变量或配置文件。

### 1.1 准备工作

需要提前准备：

1. Windows 11 系统。
2. PowerShell 或 Windows Terminal。
3. Git for Windows，建议安装。Claude Code 官方说明，在 Windows 原生环境下推荐安装 Git for Windows，这样 Claude Code 可以使用 Bash 工具；如果没有 Git for Windows，则会退回使用 PowerShell。
4. DeepSeek Platform 账号与 API Key。
5. 网络环境能够访问 DeepSeek API：

```text
https://api.deepseek.com/anthropic
```

Git for Windows 下载地址：

```text
https://git-scm.com/download/win
```

DeepSeek API Key 获取地址：

```text
https://platform.deepseek.com/
```

### 1.2 安装 Claude Code

#### 方式 A：官方原生安装方式，推荐

打开 PowerShell，执行：

```powershell
irm https://claude.ai/install.ps1 | iex
```

安装完成后检查版本：

```powershell
claude --version
```

然后进入任意工程目录测试：

```powershell
cd D:\your-project
claude
```

#### 方式 B：WinGet 安装方式

如果你习惯使用 WinGet，也可以执行：

```powershell
winget install Anthropic.ClaudeCode
```

后续升级可以执行：

```powershell
winget upgrade Anthropic.ClaudeCode
```

注意：官方文档说明，WinGet 安装方式默认不会自动更新，需要你手动执行 upgrade。

#### 方式 C：npm 安装方式，备用

如果原生安装脚本访问异常，也可以使用 npm 安装。先安装 Node.js 18+，然后执行：

```powershell
npm install -g @anthropic-ai/claude-code
```

如果 PowerShell 报错说禁止运行 `npm.ps1`，可以改用：

```powershell
npm.cmd install -g @anthropic-ai/claude-code
```

或者直接打开 CMD 执行：

```cmd
npm install -g @anthropic-ai/claude-code
```

检查安装结果：

```powershell
claude --version
```

### 1.3 临时接入 DeepSeek V4 Pro

这种方式只对当前 PowerShell 窗口有效，适合先测试是否可以跑通。

在 PowerShell 中执行：

```powershell
$env:ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
$env:ANTHROPIC_AUTH_TOKEN="你的DeepSeek_API_Key"
$env:ANTHROPIC_MODEL="deepseek-v4-pro[1m]"
$env:ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-pro[1m]"
$env:ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-pro[1m]"
$env:ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-flash"
$env:CLAUDE_CODE_SUBAGENT_MODEL="deepseek-v4-flash"
$env:CLAUDE_CODE_EFFORT_LEVEL="max"
```

然后进入项目目录启动 Claude Code：

```powershell
cd D:\your-project
claude
```

可以先输入一个简单任务测试：

```text
请用一句话说明你现在能否正常工作。
```

如果能正常返回，说明 Claude Code 已经能够通过 DeepSeek API 工作。

### 1.4 永久接入 DeepSeek V4 Pro

上面的 `$env:` 方式只在当前窗口有效。要让配置长期生效，可以把环境变量写入 Windows 用户环境变量。

在 PowerShell 中执行：

```powershell
[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "https://api.deepseek.com/anthropic", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_AUTH_TOKEN", "你的DeepSeek_API_Key", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", "deepseek-v4-pro[1m]", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_OPUS_MODEL", "deepseek-v4-pro[1m]", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_SONNET_MODEL", "deepseek-v4-pro[1m]", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_HAIKU_MODEL", "deepseek-v4-flash", "User")
[Environment]::SetEnvironmentVariable("CLAUDE_CODE_SUBAGENT_MODEL", "deepseek-v4-flash", "User")
[Environment]::SetEnvironmentVariable("CLAUDE_CODE_EFFORT_LEVEL", "max", "User")
```

执行后，关闭当前 PowerShell 窗口，重新打开一个新的 PowerShell，再检查：

```powershell
Get-ChildItem Env:ANTHROPIC_BASE_URL
Get-ChildItem Env:ANTHROPIC_MODEL
```

预期看到：

```text
ANTHROPIC_BASE_URL = https://api.deepseek.com/anthropic
ANTHROPIC_MODEL    = deepseek-v4-pro[1m]
```

然后运行：

```powershell
cd D:\your-project
claude
```

### 1.5 为什么这里使用 ANTHROPIC_AUTH_TOKEN

DeepSeek 的 Claude Code 接入文档使用的是：

```text
ANTHROPIC_AUTH_TOKEN
```

而不是：

```text
ANTHROPIC_API_KEY
```

原因是 Claude Code 在接入第三方兼容接口或 gateway 时，通常通过 `ANTHROPIC_AUTH_TOKEN` 发送 Bearer Token，更适合 DeepSeek 这种 Anthropic 兼容接口。

因此，本教程建议 Claude Code + DeepSeek 使用：

```powershell
$env:ANTHROPIC_AUTH_TOKEN="你的DeepSeek_API_Key"
```

不要优先使用：

```powershell
$env:ANTHROPIC_API_KEY="你的DeepSeek_API_Key"
```

### 1.6 验证是否正常工作

推荐按下面顺序验证：

```powershell
claude --version
```

```powershell
cd D:\your-project
claude
```

进入 Claude Code 后测试：

```text
请读取当前目录结构，并告诉我这个项目可能是什么类型。不要修改任何文件。
```

如果它能读取项目结构、分析代码，并且没有 API 鉴权错误、网络错误或模型不存在错误，说明基本配置成功。

### 1.7 恢复为未配置 DeepSeek 的状态

如果你之后想删除 DeepSeek 环境变量，可以执行：

```powershell
[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", $null, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_AUTH_TOKEN", $null, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", $null, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_OPUS_MODEL", $null, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_SONNET_MODEL", $null, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_HAIKU_MODEL", $null, "User")
[Environment]::SetEnvironmentVariable("CLAUDE_CODE_SUBAGENT_MODEL", $null, "User")
[Environment]::SetEnvironmentVariable("CLAUDE_CODE_EFFORT_LEVEL", $null, "User")
```

然后重新打开 PowerShell。

### 1.8 常见问题

#### 问题 1：`irm` 命令无法识别

一般是你在 CMD 里执行了 PowerShell 命令。请确认窗口提示符是类似：

```text
PS C:\Users\你的用户名>
```

如果提示符不是 `PS` 开头，请打开 PowerShell 再执行：

```powershell
irm https://claude.ai/install.ps1 | iex
```

#### 问题 2：`&&` 不是有效语句分隔符

一般是把 CMD 命令复制到了 PowerShell 中。PowerShell 和 CMD 命令不要混用。

#### 问题 3：环境变量设置后不生效

执行 `[Environment]::SetEnvironmentVariable(..., "User")` 后，必须关闭当前终端，重新打开一个新的 PowerShell 窗口。旧窗口不会自动刷新用户环境变量。

#### 问题 4：DeepSeek 报 401 或鉴权失败

重点检查：

1. `ANTHROPIC_AUTH_TOKEN` 是否填写了正确的 DeepSeek API Key。
2. API Key 是否包含多余空格、中文引号或换行。
3. DeepSeek 账户是否有可用余额或额度。
4. 是否把 `ANTHROPIC_AUTH_TOKEN` 错写成了 `ANTHROPIC_AUTH_TOKE`、`ANTHROPIC_API_TOKEN` 等。

#### 问题 5：模型名错误

DeepSeek 官方 Claude Code 文档中给出的模型名是：

```text
deepseek-v4-pro[1m]
deepseek-v4-flash
```

建议先完全照这个写，不要自己改成：

```text
DeepSeek-V4-Pro
DeepSeek V4 Pro
DeepSeek-v4-pro
```

模型名大小写、横杠、方括号都可能影响识别。

---

## 第二章：Claude Code 通过 CC Switch 连接 DeepSeek

这一章适合后期需要频繁切换多个模型源的情况。比如你希望在 DeepSeek、Claude 官方、Kimi、硅基流动、OpenAI 兼容代理、中转站之间切换，而不想每次都手动改环境变量。

整体链路大致如下：

```text
Claude Code CLI  →  CC Switch 写入/切换配置  →  DeepSeek Anthropic API  →  deepseek-v4-pro[1m]
```

需要注意：CC Switch 不是 Anthropic 官方工具，也不是 DeepSeek 官方工具，而是第三方社区工具。它的价值是降低多供应商配置维护成本，但也多了一层兼容性变量。

### 2.1 什么时候建议使用 CC Switch

建议使用 CC Switch 的情况：

1. 你不只使用 DeepSeek，还要经常切换多个模型源。
2. 你不想反复手动修改环境变量或 `.claude/settings.json`。
3. 你希望图形化管理 Claude Code、Codex、Gemini CLI、OpenCode 等工具的 Provider。
4. 你希望统一管理 MCP、Skills、Prompts 等配置。

不建议一开始就使用 CC Switch 的情况：

1. 你只想固定使用 DeepSeek V4 Pro。
2. 你正在排查 Claude Code 是否能正常运行。
3. 你希望链路尽可能简单、可控、稳定。
4. 你在处理公司源码、合同材料、涉密项目，并且暂时不确定第三方工具如何保存和同步配置。

建议顺序是：

```text
先用第一章的直接连接方案跑通 Claude Code + DeepSeek，确认 API Key、模型名、网络都没问题；
再安装 CC Switch，把已经跑通的配置迁移进去。
```

### 2.2 安装 CC Switch

打开 CC Switch GitHub：

```text
https://github.com/farion1231/cc-switch
```

进入 Releases 页面，下载 Windows 版本：

```text
CC-Switch-v{version}-Windows.msi
```

或者下载便携版：

```text
CC-Switch-v{version}-Windows-Portable.zip
```

官方 README 中说明 Windows 10 及以上系统可用，因此 Windows 11 满足要求。

安装建议：

1. 普通用户优先下载 `.msi` 安装包。
2. 不想写入系统安装项的用户，可以下载 Portable 版本。
3. 尽量使用 Releases 中的最新版。
4. 不建议从不明网盘、群文件、第三方搬运站下载安装包。

### 2.3 在 CC Switch 中添加 DeepSeek Provider

启动 CC Switch 后，按照下面逻辑配置。

由于 CC Switch 版本更新较快，按钮名称可能略有差异，但核心字段基本一致。

#### 步骤 1：添加 Provider

在主界面点击：

```text
Add Provider / 添加 Provider / 添加供应商
```

如果内置列表中有 DeepSeek 预设，可以直接选择 DeepSeek。

如果没有 DeepSeek 预设，选择：

```text
Custom Provider / 自定义供应商
```

#### 步骤 2：选择或指定 Claude Code

在工具类型或适用 App 中选择：

```text
Claude Code
```

如果 CC Switch 支持“通用 Provider”，也可以创建一个通用 Provider，再同步到 Claude Code。

#### 步骤 3：填写 DeepSeek Anthropic API 信息

核心字段建议如下：

```text
Provider Name: DeepSeek V4 Pro
Protocol / Format: Anthropic Compatible
Base URL: https://api.deepseek.com/anthropic
API Key: 你的 DeepSeek API Key
Default Model: deepseek-v4-pro[1m]
Small / Fast Model: deepseek-v4-flash
```

如果高级配置中允许填写环境变量，建议写入：

```text
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
ANTHROPIC_AUTH_TOKEN=你的DeepSeek_API_Key
ANTHROPIC_MODEL=deepseek-v4-pro[1m]
ANTHROPIC_DEFAULT_OPUS_MODEL=deepseek-v4-pro[1m]
ANTHROPIC_DEFAULT_SONNET_MODEL=deepseek-v4-pro[1m]
ANTHROPIC_DEFAULT_HAIKU_MODEL=deepseek-v4-flash
CLAUDE_CODE_SUBAGENT_MODEL=deepseek-v4-flash
CLAUDE_CODE_EFFORT_LEVEL=max
```

如果界面只要求填写模型列表，可以至少添加：

```text
deepseek-v4-pro[1m]
deepseek-v4-flash
```

#### 步骤 4：保存并启用 Provider

保存后，回到 Provider 列表，选择刚刚创建的：

```text
DeepSeek V4 Pro
```

点击：

```text
Enable / 启用 / 切换到此 Provider
```

CC Switch README 中说明，通常切换 Provider 后需要重启终端或对应 CLI 工具才能生效；但 Claude Code 当前支持 provider data 热切换。不过为了避免环境残留，建议你第一次配置后仍然重新打开 PowerShell。

### 2.4 使用 CC Switch 的测试模型功能

如果当前版本的 CC Switch 提供：

```text
Test Model / 测试模型
```

建议一定要先点测试。

测试通过后，再打开 PowerShell：

```powershell
cd D:\your-project
claude
```

进入 Claude Code 后输入：

```text
请只做项目结构分析，不要修改任何文件。
```

如果 Claude Code 能读取项目结构并正常返回，说明 CC Switch → DeepSeek 的链路基本可用。

### 2.5 CC Switch 方案的关键注意事项

#### 注意 1：CC Switch 不是必须组件

如果你只用 DeepSeek，不需要频繁切换模型，直接使用第一章的环境变量方案即可。这样链路更短，出问题更容易定位。

#### 注意 2：优先使用 DeepSeek 官方 API，不要随便使用不明中转

如果你选择 DeepSeek，就尽量使用：

```text
https://api.deepseek.com/anthropic
```

不要轻易把公司源码、项目资料、合同文件发送到不明中转站。

#### 注意 3：DeepSeek V4 Pro 模型名要写准确

建议使用官方文档中的：

```text
deepseek-v4-pro[1m]
```

不要随意改写成：

```text
DeepSeek-V4-Pro
DeepSeek V4 Pro
v4-pro
```

#### 注意 4：CC Switch 可能存在版本兼容问题

社区 issue 中曾记录过 Claude Code + DeepSeek V4 + CC Switch 在某些版本组合下出现 HTTP 400 thinking mode 相关错误。这类问题通常不是你的 API Key 错误，而是 Claude Code、DeepSeek V4 thinking mode、CC Switch 代理/转换逻辑之间的协议兼容问题。

如果出现类似错误，例如：

```text
HTTP 400
thinking mode
reasoning_content
content[].thinking
```

建议按下面顺序处理：

1. 升级 CC Switch 到最新版。
2. 升级 Claude Code 到最新版。
3. 重新测试 DeepSeek Provider。
4. 临时切回第一章的“直接连接 DeepSeek”方案，判断是不是 CC Switch 中间层导致的问题。
5. 如果直接连接正常、CC Switch 不正常，说明问题大概率在 CC Switch 的版本兼容或代理转换层。

### 2.6 如何在 CC Switch 和直接连接方案之间切换

#### 从直接连接切换到 CC Switch

如果你之前已经用第一章方式设置了用户环境变量，而现在希望交给 CC Switch 管理，建议先删除用户环境变量，避免两套配置冲突。

执行：

```powershell
[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", $null, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_AUTH_TOKEN", $null, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", $null, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_OPUS_MODEL", $null, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_SONNET_MODEL", $null, "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_HAIKU_MODEL", $null, "User")
[Environment]::SetEnvironmentVariable("CLAUDE_CODE_SUBAGENT_MODEL", $null, "User")
[Environment]::SetEnvironmentVariable("CLAUDE_CODE_EFFORT_LEVEL", $null, "User")
```

然后重新打开 PowerShell，再让 CC Switch 启用 DeepSeek Provider。

#### 从 CC Switch 切回直接连接

如果 CC Switch 出现兼容问题，可以临时关闭 CC Switch Provider，然后重新使用第一章的临时环境变量：

```powershell
$env:ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
$env:ANTHROPIC_AUTH_TOKEN="你的DeepSeek_API_Key"
$env:ANTHROPIC_MODEL="deepseek-v4-pro[1m]"
$env:ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-pro[1m]"
$env:ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-pro[1m]"
$env:ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-flash"
$env:CLAUDE_CODE_SUBAGENT_MODEL="deepseek-v4-flash"
$env:CLAUDE_CODE_EFFORT_LEVEL="max"
```

再运行：

```powershell
cd D:\your-project
claude
```

### 2.7 推荐使用策略

如果你当前只是想稳定使用 Claude Code + DeepSeek V4 Pro，推荐：

```text
第一优先：Claude Code 直接连接 DeepSeek
```

如果你已经确认 DeepSeek 直连方案可用，并且后期要经常切换模型源，推荐：

```text
第二阶段：引入 CC Switch 管理多个 Provider
```

实际工作中建议保留两套方案：

1. 日常多模型切换：使用 CC Switch。
2. 出现奇怪兼容问题：临时切回直接连接 DeepSeek。

这样既能享受 CC Switch 的便利，也能保留一个稳定、可排查的基础方案。

---

## 最终建议

对 Windows 11 用户来说，最稳的落地路径是：

```text
1. 安装 Git for Windows。
2. 用 PowerShell 原生安装 Claude Code。
3. 用 DeepSeek 官方环境变量方式直连 DeepSeek V4 Pro。
4. 确认 Claude Code 能在真实工程目录中正常读写、分析、运行命令。
5. 如果后期需要频繁切换模型，再安装 CC Switch。
6. 在 CC Switch 中添加 DeepSeek Provider，并保留直连方案作为故障排查备用方案。
```

一句话总结：

```text
DeepSeek 直连方案是基础方案、稳定方案；CC Switch 是多模型管理方案、便利方案，但不是必需方案。
```
