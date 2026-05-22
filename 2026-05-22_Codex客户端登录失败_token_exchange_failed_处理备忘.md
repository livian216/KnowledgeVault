# Codex 客户端登录失败问题处理备忘

## 1. 问题现象

在使用 Codex 客户端登录时，浏览器侧已经进入登录流程，但最终提示登录失败，错误信息类似：

```text
Sign-in could not be completed

Token exchange failed: error sending request for url
(https://auth.openai.com/oauth/token)

Error code: token_exchange_failed
```

该问题的关键点是：  
**不是账号密码错误，也不是 ChatGPT 账号本身失效，而是 Codex 客户端在登录流程的“Token 交换”阶段无法正常访问 OpenAI 的认证服务。**

---

## 2. 问题发生的背景

本次问题出现前，并没有主动更换 VPN 软件，也没有明显修改网络环境，但近期更新了 Codex 客户端或 Codex 扩展。

更新后，原本可以正常登录和使用的 Codex 客户端，突然无法完成登录。

---

## 3. 原因分析

Codex 的登录流程大致可以理解为：

```text
Codex 客户端发起登录
        ↓
打开浏览器进行 OpenAI / ChatGPT 账号认证
        ↓
浏览器认证通过后返回授权信息
        ↓
Codex 客户端向 https://auth.openai.com/oauth/token 请求 token
        ↓
Codex 客户端获得 token 后完成登录
```

本次失败发生在最后的 token 请求阶段。

也就是说，浏览器可能可以正常访问 OpenAI 登录页面，但 Codex 客户端本身作为一个独立进程，可能没有正确走本机 VPN / 代理。

因此就会出现：

```text
浏览器能登录
Codex 客户端不能完成 token exchange
```

这种情况常见原因包括：

1. Codex 客户端更新后，对系统代理或环境变量代理的读取方式发生变化；
2. 浏览器使用了代理，但 Codex 客户端进程没有使用代理；
3. Windows 系统代理、WinHTTP 代理、环境变量代理之间不一致；
4. Codex 客户端访问 `https://auth.openai.com/oauth/token` 时没有走 VPN 代理，导致请求失败；
5. 旧版本可用的认证缓存或代理继承方式，在新版本中不再稳定。

本次问题最终通过设置 Windows 用户级环境变量代理解决，说明根本原因更偏向于：

> Codex 客户端进程没有正确继承或识别当前 VPN 代理配置，需要通过 `HTTP_PROXY` 和 `HTTPS_PROXY` 环境变量显式指定代理地址。

---

## 4. 正确解决方案

在 Windows PowerShell 中执行以下命令：

```powershell
setx HTTP_PROXY "http://127.0.0.1:7890"
setx HTTPS_PROXY "http://127.0.0.1:7890"
```

执行完成后，需要：

```text
1. 彻底退出 Codex 客户端；
2. 关闭正在使用的 PowerShell、VS Code、Cursor 等相关程序；
3. 重新打开 Codex 客户端；
4. 再次执行登录。
```

`setx` 设置的是用户级环境变量，通常需要重新打开相关程序后才会生效。

---

## 5. 关于 7890 端口的说明

上面的 `7890` 只是本次环境中 VPN / 代理软件使用的本地代理端口。

不同代理软件、不同配置下，本地代理端口可能不同。例如常见端口包括：

```text
7890
7897
1080
10808
10809
20171
```

因此不能机械照抄 `7890`，需要根据自己电脑上的 VPN / 代理软件实际配置进行适配。

例如：

如果你的代理软件 HTTP 端口是 `10809`，则应设置为：

```powershell
setx HTTP_PROXY "http://127.0.0.1:10809"
setx HTTPS_PROXY "http://127.0.0.1:10809"
```

如果你的代理软件 Mixed Port 是 `7890`，则可以使用：

```powershell
setx HTTP_PROXY "http://127.0.0.1:7890"
setx HTTPS_PROXY "http://127.0.0.1:7890"
```

---

## 6. 如何查看自己的代理端口

### 6.1 从代理软件界面查看

一般在代理软件的设置页面中查找以下字段：

```text
HTTP Port
HTTPS Port
Mixed Port
SOCKS Port
本地监听端口
局域网监听端口
```

优先使用：

```text
HTTP Port
Mixed Port
```

不建议一开始就使用 SOCKS 端口，除非确认目标程序支持 SOCKS 代理。

---

### 6.2 从 Windows 系统代理查看

在 PowerShell 中执行：

```powershell
netsh winhttp show proxy
```

或者查看当前用户代理设置：

```powershell
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" | Select-Object ProxyEnable, ProxyServer
```

如果输出中看到类似：

```text
127.0.0.1:7890
```

则说明当前系统代理端口可能是 `7890`。

---

### 6.3 查看本机监听端口

在 PowerShell 中执行：

```powershell
netstat -ano | findstr LISTENING
```

重点查看是否存在类似：

```text
127.0.0.1:7890
127.0.0.1:10809
127.0.0.1:10808
```

如果看到对应端口处于监听状态，说明本机确实有程序在该端口提供服务。

---

## 7. 如何验证代理是否有效

假设代理端口是 `7890`，可以在 PowerShell 中执行：

```powershell
curl.exe -x http://127.0.0.1:7890 -I https://auth.openai.com
```

如果能够返回 HTTP 响应头，说明该代理端口可以访问 OpenAI 认证服务。

也可以直接在设置环境变量后执行：

```powershell
curl.exe -I https://auth.openai.com
```

如果能正常返回，说明当前 PowerShell 环境下访问认证服务基本正常。

---

## 8. 如果需要撤销代理环境变量

如果后续不再希望让程序默认走该代理，可以删除用户级环境变量。

在 PowerShell 中执行：

```powershell
[Environment]::SetEnvironmentVariable("HTTP_PROXY", $null, "User")
[Environment]::SetEnvironmentVariable("HTTPS_PROXY", $null, "User")
```

执行后同样需要重新打开相关程序才能完全生效。

---

## 9. 本次问题总结

本次问题的本质不是 Codex 账号异常，也不是浏览器登录失败，而是：

> Codex 客户端更新后，客户端进程没有正确使用本机 VPN / 代理，导致其向 `https://auth.openai.com/oauth/token` 请求 token 时失败。

最终有效解决方式是：

```powershell
setx HTTP_PROXY "http://127.0.0.1:7890"
setx HTTPS_PROXY "http://127.0.0.1:7890"
```

其中 `7890` 需要替换为自己本机 VPN / 代理软件实际使用的 HTTP 或 Mixed 代理端口。

处理此类问题时，可以按照以下顺序排查：

```text
1. 确认浏览器可以正常访问 OpenAI；
2. 确认 Codex 客户端报错是否为 token_exchange_failed；
3. 查看本机 VPN / 代理软件的本地代理端口；
4. 使用 HTTP_PROXY 和 HTTPS_PROXY 显式指定代理；
5. 重启 Codex 客户端后重新登录；
6. 如仍失败，再考虑清理 Codex 缓存或重装客户端。
```
