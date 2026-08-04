# Windows 免密 SSH 登录 Ubuntu 服务器配置指南



## 1. 示例参数

本文统一使用以下示例参数：

| 参数 | 示例值 | 说明 |
|---|---|---|
| Ubuntu 用户名 | `<SERVER_USER>` | 替换为服务器上的实际用户名 |
| Ubuntu 服务器地址 | `192.0.2.10` | 文档示例地址，不是真实服务器 IP |
| SSH 端口 | `22` | 如服务器使用其他端口，请自行替换 |
| Windows 密钥名称 | `id_ed25519_example_server` | 可自行修改，但公钥和私钥名称需对应 |

> `192.0.2.10` 仅用于文档示例。实际使用时，请替换为服务器的真实 IP、域名或 VPN 地址。

---

## 2. 在 Windows 本地生成 SSH 密钥

在 Windows PowerShell 中执行：

```powershell
ssh-keygen -t ed25519 `
  -f "$env:USERPROFILE\.ssh\id_ed25519_example_server" `
  -C "windows-client-to-example-server"
```

命令会生成两个文件：

```text
C:\Users\<WINDOWS_USER>\.ssh\id_ed25519_example_server
C:\Users\<WINDOWS_USER>\.ssh\id_ed25519_example_server.pub
```

两者用途如下：

| 文件 | 用途 | 是否可以上传到服务器 |
|---|---|---|
| `id_ed25519_example_server` | 私钥 | **不可以** |
| `id_ed25519_example_server.pub` | 公钥 | 可以 |

需要实现完全免交互登录时，在提示输入密钥口令时连续按两次 Enter，不设置口令。

查看公钥内容：

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519_example_server.pub"
```

复制输出的完整一行。公钥通常类似：

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... windows-client-to-example-server
```

---

## 3. 使用密码登录 Ubuntu 服务器

在 Windows PowerShell 中执行：

```powershell
ssh -p 22 <SERVER_USER>@192.0.2.10
```

首次配置时仍需输入 Ubuntu 账户密码。

登录后确认当前用户和主目录：

```bash
whoami
echo "$HOME"
```

预期输出形式如下：

```text
<SERVER_USER>
/home/<SERVER_USER>
```

---

## 4. 在 Ubuntu 上配置公钥

### 4.1 创建 `.ssh` 目录和授权文件

在 Ubuntu 服务器中执行：

```bash
mkdir -p "$HOME/.ssh"
touch "$HOME/.ssh/authorized_keys"
```

### 4.2 写入 Windows 公钥

编辑授权文件：

```bash
nano "$HOME/.ssh/authorized_keys"
```

将 Windows 中复制的公钥完整粘贴进去。

要求：

- 每个公钥单独占一行；
- 不要添加引号；
- 不要拆分成多行；
- 如果文件中已有其他公钥，应在末尾另起一行追加，不要覆盖；
- 不要粘贴不带 `.pub` 后缀的私钥内容。

保存并退出 Nano：

```text
Ctrl+O
Enter
Ctrl+X
```

### 4.3 设置目录和文件权限

执行：

```bash
chmod 700 "$HOME/.ssh"
chmod 600 "$HOME/.ssh/authorized_keys"
chmod go-w "$HOME"
```

确保 `.ssh` 目录及文件属于当前用户：

```bash
chown -R "$(id -un)":"$(id -gn)" "$HOME/.ssh"
```

检查权限：

```bash
ls -ld "$HOME/.ssh"
ls -l "$HOME/.ssh/authorized_keys"
```

正常结果应类似：

```text
drwx------ <SERVER_USER> <SERVER_GROUP> .ssh
-rw------- <SERVER_USER> <SERVER_GROUP> authorized_keys
```

完成后退出服务器：

```bash
exit
```

---

## 5. 在 Windows 中验证免密登录

在 Windows PowerShell 中执行：

```powershell
ssh `
  -o PreferredAuthentications=publickey `
  -o PasswordAuthentication=no `
  -o IdentitiesOnly=yes `
  -i "$env:USERPROFILE\.ssh\id_ed25519_example_server" `
  -p 22 `
  <SERVER_USER>@192.0.2.10
```

也可以使用单行形式：

```powershell
ssh -o PreferredAuthentications=publickey -o PasswordAuthentication=no -o IdentitiesOnly=yes -i "$env:USERPROFILE\.ssh\id_ed25519_example_server" -p 22 <SERVER_USER>@192.0.2.10
```

如果不再提示输入 Ubuntu 账户密码，并能直接进入服务器，则配置成功。

需要注意：

```text
<SERVER_USER>@192.0.2.10's password:
```

表示服务器仍在要求账户密码，公钥认证尚未成功。

```text
Enter passphrase for key ...
```

表示私钥设置了密钥口令。这不是 Ubuntu 账户密码。

---

## 6. 可选：配置 Windows SSH 别名

编辑 Windows SSH 配置文件：

```powershell
notepad "$env:USERPROFILE\.ssh\config"
```

添加：

```sshconfig
Host example-gpu-server
    HostName 192.0.2.10
    User <SERVER_USER>
    Port 22
    IdentityFile ~/.ssh/id_ed25519_example_server
    IdentitiesOnly yes
```

以后可直接执行：

```powershell
ssh example-gpu-server
```

---

## 7. 失败时的最小排查命令

### Windows 客户端调试

```powershell
ssh -vvv `
  -o IdentitiesOnly=yes `
  -i "$env:USERPROFILE\.ssh\id_ed25519_example_server" `
  -p 22 `
  <SERVER_USER>@192.0.2.10
```

重点查看是否出现：

```text
Offering public key
Server accepts key
Authenticated to ...
```

### Ubuntu 服务端检查

确认当前用户及主目录：

```bash
whoami
echo "$HOME"
```

检查公钥文件：

```bash
cat "$HOME/.ssh/authorized_keys"
```

检查权限：

```bash
ls -ld "$HOME" "$HOME/.ssh"
ls -l "$HOME/.ssh/authorized_keys"
```

查看 SSH 日志：

```bash
sudo journalctl -u ssh -n 100 --no-pager
```

---

## 8. 最终操作顺序

```text
Windows 生成密钥
        ↓
复制 .pub 公钥内容
        ↓
使用密码登录 Ubuntu
        ↓
创建 ~/.ssh/authorized_keys
        ↓
写入公钥并设置 700/600 权限
        ↓
Windows 强制使用公钥认证测试
        ↓
确认无需输入 Ubuntu 账户密码
```
