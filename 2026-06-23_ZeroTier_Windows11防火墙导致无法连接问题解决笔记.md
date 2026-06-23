# ZeroTier Windows 11 防火墙导致无法连接问题解决笔记

## 一、问题背景

在 Windows 11 笔记本上使用 ZeroTier 远程连接同一个 ZeroTier 虚拟网络内的其他设备时，出现了突然无法连接的问题。

原本前一天晚上 ZeroTier 网络和 SSH 都可以正常使用，但第二天电脑睡眠/唤醒后，在没有主动修改配置的情况下，突然出现：

- `zerotier-cli status` 显示 `ONLINE`；
- `zerotier-cli listnetworks` 显示网络状态为 `OK`；
- `ipconfig` 中可以看到 ZeroTier 虚拟网卡和 ZeroTier IP；
- `route print` 中可以看到 ZeroTier 网段路由；
- ZeroTier Central 后台设备授权正常；
- 同一个 ZeroTier 网络中的其他电脑之间可以正常互联；
- 但是这台 Windows 11 笔记本无法 ping 通 ZeroTier 网络内的其他设备；
- 通过 SSH 连接 ZeroTier 网络内其他设备也失败。

该问题的特点是：ZeroTier 本身看起来已经正常在线，并且设备也已经加入了对应虚拟网络，但实际网络通信异常。

---

## 二、初步判断

根据排查结果，可以排除以下问题：

1. **不是 ZeroTier 服务离线问题**  
   因为执行：

   ```powershell
   zerotier-cli status
   ```

   输出已经是：

   ```text
   ONLINE
   ```

2. **不是设备未加入网络或未授权问题**  
   因为执行：

   ```powershell
   zerotier-cli listnetworks
   ```

   输出类似：

   ```text
   200 listnetworks xxxxxxxxxxxxxxxx 网络名 xx:xx:xx:xx:xx:xx OK PRIVATE ztxxxxxx 10.10.10.5/24
   ```

   其中 `OK` 表示设备已经正确加入并授权到该 ZeroTier 网络。

3. **不是没有获取 ZeroTier 虚拟 IP 的问题**  
   因为 `ipconfig` 中能够看到 ZeroTier 虚拟网卡，并且有类似：

   ```text
   10.10.10.5
   ```

   这样的 ZeroTier IP。

4. **不是路由表缺失问题**  
   因为 `route print` 中能够看到通往 ZeroTier 网段的路由。

5. **不是整个 ZeroTier 网络故障**  
   因为同一个 ZeroTier 网络内的其他电脑之间可以正常互联。

因此，问题可以初步定位到：

> Windows 11 本机的防火墙、网络配置文件、ZeroTier 虚拟网卡安全策略、VPN/TUN 模式或安全软件，对 ZeroTier 虚拟网卡上的流量进行了拦截。

---

## 三、最终解决命令

以管理员身份打开 PowerShell，执行：

```powershell
New-NetFirewallRule -DisplayName "Allow ICMPv4-In-ZeroTier" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```

执行后，ZeroTier 网络恢复正常：

- 可以 ping 通 ZeroTier 网络内其他设备；
- SSH 连接也恢复正常。

---

## 四、原因分析

该命令的作用是创建一条 Windows Defender Firewall 入站规则，允许 ICMPv4 Echo Request，也就是允许其他设备通过 IPv4 ping 本机。

需要注意的是：

- `ping` 使用的是 **ICMPv4**；
- `SSH` 使用的是 **TCP 22**。

从协议层面看，放行 ICMPv4 并不等同于直接放行 SSH 的 TCP 22 端口。

但是在本次问题中，执行 ICMPv4 入站放行规则后，ping 和 SSH 同时恢复正常。这说明本次问题并不是单纯的“SSH 服务异常”，而更像是 Windows 11 本机防火墙或网络安全策略对 ZeroTier 虚拟网卡上的通信进行了异常拦截。

更准确的经验结论是：

> ZeroTier 本身已经正常在线，设备也已经正确加入网络，但 Windows 11 本机防火墙对 ZeroTier 虚拟网卡流量进行了拦截，导致同一 ZeroTier 虚拟网络内的设备无法正常 ping / SSH。通过添加 ICMPv4 入站允许规则后，ZeroTier 虚拟网络通信恢复，说明故障点集中在 Windows 本地防火墙或网络配置文件，而不是 ZeroTier 账号、Network ID、设备授权、Managed IP 或路由配置。

---

## 五、为什么会突然出现

该问题可能与以下因素有关：

1. **Windows 睡眠/唤醒后网络配置文件变化**  
   Windows 防火墙会根据网络类型应用不同规则，例如：

   - Domain；
   - Private；
   - Public。

   睡眠/唤醒、Wi-Fi 重新连接、VPN 连接变化、ZeroTier 虚拟网卡重建后，Windows 可能把 ZeroTier 虚拟网卡识别为更严格的 `Public` 网络，从而导致原来能通过的流量被拦截。

2. **ZeroTier 虚拟网卡自动防火墙规则没有正确恢复**  
   某些情况下，ZeroTier 虚拟网卡虽然存在，IP 和路由也正常，但 Windows 防火墙没有正确放行对应流量。

3. **VPN / 代理 / TUN 模式影响虚拟网卡路由与安全策略**  
   如果电脑同时运行 VPN、代理软件或 TUN 模式，它们可能改变 Windows 的路由表、虚拟网卡优先级或防火墙行为。

4. **Windows Defender Firewall 或安全软件策略变更**  
   系统更新、安全软件更新、网络重连或防火墙策略刷新，都可能造成原本允许的虚拟网卡通信被重新限制。

---

## 六、该规则是否永久生效

执行：

```powershell
New-NetFirewallRule -DisplayName "Allow ICMPv4-In-ZeroTier" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```

创建的是 Windows 防火墙规则，默认不是临时规则。

一般情况下，该规则在重启电脑后仍然有效。

除非发生以下情况：

- 手动删除该防火墙规则；
- 重置 Windows 防火墙；
- 被第三方安全软件清理；
- 被公司、学校、单位域控组策略覆盖；
- 系统更新后防火墙策略被重置；
- ZeroTier 虚拟网卡配置变化后，规则仍存在但匹配范围不理想。

可以用下面命令检查规则是否存在：

```powershell
Get-NetFirewallRule -DisplayName "Allow ICMPv4-In-ZeroTier"
```

如果要删除该规则，可以执行：

```powershell
Remove-NetFirewallRule -DisplayName "Allow ICMPv4-In-ZeroTier"
```

---

## 七、更安全的防火墙规则写法

上面的命令会允许所有来源 ping 本机。对于个人电脑来说风险通常不高，但更规范的做法是只允许来自 ZeroTier 网段的 ICMPv4 请求。

假设 ZeroTier 网段是：

```text
10.10.10.0/24
```

可以先删除原来的宽泛规则：

```powershell
Remove-NetFirewallRule -DisplayName "Allow ICMPv4-In-ZeroTier"
```

然后创建只允许 ZeroTier 网段访问的规则：

```powershell
New-NetFirewallRule `
  -DisplayName "Allow ICMPv4-In-ZeroTier-Only" `
  -Protocol ICMPv4 `
  -IcmpType 8 `
  -Direction Inbound `
  -RemoteAddress 10.10.10.0/24 `
  -Action Allow
```

这样可以避免对所有网络来源开放 ping，只允许 ZeroTier 虚拟内网中的设备 ping 本机。

如果后续希望进一步控制规则只在特定网络配置文件下生效，可以增加 `-Profile` 参数，例如：

```powershell
-Profile Private
```

但如果 Windows 经常把 ZeroTier 虚拟网卡识别为 `Public`，添加 `-Profile Private` 反而可能导致规则不生效。因此对于 ZeroTier 场景，优先建议通过 `-RemoteAddress` 限制来源网段。

---

## 八、后续遇到类似问题的排查流程

如果 ZeroTier 后续再次出现 `ONLINE` 但无法 ping / SSH 的情况，可以按下面顺序排查。

### 1. 检查 ZeroTier 服务状态

```powershell
zerotier-cli status
```

正常应显示：

```text
ONLINE
```

如果显示 `OFFLINE`，说明 ZeroTier 客户端没有连上 ZeroTier 网络基础设施，需要优先排查网络、VPN、防火墙和 ZeroTier 服务。

### 2. 检查是否已经加入并授权网络

```powershell
zerotier-cli listnetworks
```

正常应包含：

```text
OK
```

如果显示：

```text
ACCESS_DENIED
```

说明该设备还没有在 ZeroTier Central 后台授权。

### 3. 检查是否获取 ZeroTier IP

```powershell
ipconfig
```

确认 ZeroTier 虚拟网卡下存在类似：

```text
10.x.x.x
```

这样的 ZeroTier IP。

### 4. 检查路由表

```powershell
route print
```

或者按 ZeroTier 网段筛选，例如：

```powershell
route print | findstr 10.10
```

确认路由表中存在通向 ZeroTier 网段的路由。

### 5. 检查 Windows 网络配置文件

```powershell
Get-NetConnectionProfile | Format-Table Name, InterfaceAlias, NetworkCategory
```

观察 ZeroTier 虚拟网卡是：

```text
Public
Private
DomainAuthenticated
```

如果 ZeroTier 虚拟网卡被识别成 `Public`，可能导致防火墙规则更严格。

### 6. 检查防火墙规则是否存在

```powershell
Get-NetFirewallRule -DisplayName "Allow ICMPv4-In-ZeroTier*"
```

如果规则不存在，可以重新创建。

### 7. 测试 SSH 端口是否可达

```powershell
Test-NetConnection 对方ZeroTier_IP -Port 22
```

例如：

```powershell
Test-NetConnection 10.10.10.100 -Port 22
```

如果输出中：

```text
TcpTestSucceeded : True
```

说明 SSH 端口可达。

如果是 `False`，需要继续检查目标设备的 SSH 服务、防火墙和 ZeroTier 状态。

---

## 九、经验总结

本次问题可以总结为：

> 当 Windows 11 上 ZeroTier 显示 `ONLINE`，`listnetworks` 显示 `OK`，并且已经获取 ZeroTier IP、路由表也正常，但仍然无法 ping / SSH 同一 ZeroTier 网络内的其他设备时，不要只怀疑 ZeroTier 服务或账号配置。应重点检查 Windows 11 本机防火墙、网络配置文件、ZeroTier 虚拟网卡安全策略、VPN/TUN 模式和第三方安全软件。

最有效的解决命令是：

```powershell
New-NetFirewallRule -DisplayName "Allow ICMPv4-In-ZeroTier" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```

更安全的 ZeroTier 网段限定写法是：

```powershell
New-NetFirewallRule `
  -DisplayName "Allow ICMPv4-In-ZeroTier-Only" `
  -Protocol ICMPv4 `
  -IcmpType 8 `
  -Direction Inbound `
  -RemoteAddress 10.10.10.0/24 `
  -Action Allow
```

其中 `10.10.10.0/24` 需要替换成自己的 ZeroTier 虚拟网络网段。

---

## 十、建议保留的常用命令

```powershell
zerotier-cli status
zerotier-cli listnetworks
ipconfig
route print
Get-NetConnectionProfile | Format-Table Name, InterfaceAlias, NetworkCategory
Get-NetFirewallRule -DisplayName "Allow ICMPv4-In-ZeroTier*"
Test-NetConnection 对方ZeroTier_IP -Port 22
```

如果需要重启 ZeroTier 服务：

```powershell
Restart-Service ZeroTierOneService
```

如果服务名不确定，可以执行：

```powershell
Get-Service *ZeroTier*
```

---

## 十一、注意事项

1. `zerotier-cli status = ONLINE` 只说明 ZeroTier 客户端连接到了 ZeroTier 基础网络，不代表本机防火墙一定允许通信。
2. `zerotier-cli listnetworks = OK` 说明设备已经加入并授权到 ZeroTier 网络，但仍可能被 Windows 防火墙拦截。
3. `ping` 不通不一定代表 SSH 不通，因为 ping 和 SSH 使用不同协议。
4. 如果 ping 和 SSH 同时异常，应优先检查 Windows 防火墙、网络配置文件、VPN/TUN 和安全软件。
5. 如果只允许 ICMP 后 SSH 恢复，说明问题大概率集中在 Windows 对 ZeroTier 虚拟网卡流量的整体安全策略，而不是单一的 SSH 服务故障。
