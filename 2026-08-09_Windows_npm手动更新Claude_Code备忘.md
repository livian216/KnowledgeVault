# Windows 下手动更新 Claude Code（npm 安装版）

> 适用场景：Claude Code 通过 npm 全局安装在 Windows 系统中，并使用 PowerShell 进行更新。

## 1. 查看当前 Claude Code 版本

打开 PowerShell，执行：

```powershell
claude --version
```

示例输出：

```text
2.1.226 (Claude Code)
```

---

## 2. 查看 npm 当前发布的最新版

执行：

```powershell
npm view @anthropic-ai/claude-code version
```

如果这里显示的版本号高于 `claude --version` 的结果，说明可以进行更新。

---

## 3. 手动更新 Claude Code

建议先退出所有正在运行的 Claude Code 会话，然后执行：

```powershell
npm install -g @anthropic-ai/claude-code@latest
```

也可以使用简写：

```powershell
npm i -g @anthropic-ai/claude-code@latest
```

注意：包名与 `@latest` 之间**不要添加反斜杠 `\`**。

---

## 4. 验证更新结果

更新完成后执行：

```powershell
claude --version
```

然后再次查看 npm 最新版本：

```powershell
npm view @anthropic-ai/claude-code version
```

如果两个版本号一致，说明已经更新到最新版。

---

## 5. 检查 Claude Code 安装状态

可以执行：

```powershell
claude doctor
```

如果安装正常，通常可以看到：

```text
No installation issues found.
```

还可以检查 `claude` 命令实际所在位置：

```powershell
where.exe claude
```

npm 全局安装时，通常会得到类似：

```text
C:\Users\<用户名>\AppData\Roaming\npm\claude
C:\Users\<用户名>\AppData\Roaming\npm\claude.cmd
```

---

## 6. 遇到文件占用错误时

如果更新过程中出现类似：

```text
npm error code EBUSY
npm error EBUSY: resource busy or locked
```

一般表示 Claude Code 的程序文件仍被 Windows 中某个进程占用。

先关闭所有 Claude Code 会话、PowerShell、Windows Terminal 或 IDE 中正在运行的 Claude Code，然后重新打开 PowerShell，再执行：

```powershell
npm install -g @anthropic-ai/claude-code@latest
```

如果仍然提示文件占用，可以重启 Windows，重启后不要先启动 Claude Code，直接打开 PowerShell 执行更新命令。

---

## 日常更新速查

以后正常更新时，通常只需要执行：

```powershell
# 更新到最新版
npm install -g @anthropic-ai/claude-code@latest

# 查看安装版本
claude --version

# 查看 npm 最新版本
npm view @anthropic-ai/claude-code version

# 检查安装状态
claude doctor
```

其中最核心的更新命令是：

```powershell
npm install -g @anthropic-ai/claude-code@latest
```
