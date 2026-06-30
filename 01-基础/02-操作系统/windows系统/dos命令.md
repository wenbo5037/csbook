# 🛡️ 网络安全原生命令（CMD & PowerShell）
## 目录总览

| 章节 | 主题 | 核心能力 |
|------|------|----------|
| 一 | 系统信息与资产枚举 | 信息收集、权限审计、AD 枚举 |
| 二 | 网络侦察与连接分析 | 拓扑发现、端口扫描、防火墙分析 |
| 三 | 文件系统与数据检索 | 敏感信息搜索、取证、时间线构建 |
| 四 | 安全防御、持久化检测与对抗 | IOC 排查、防御验证、横向移动检测 |
| 五 | PowerShell 专属安全能力 | .NET 交互、远程管理、日志分析 |
| 六 | 高阶冷门与底层协议 | PKI、虚拟化、内核、RPC/DCOM/LDAP |
| 七 | 学习方法论与知识管理 | 实战方法论 |

---

## 一、系统信息与资产枚举

> **核心目标**：搞清楚"我在哪台机器上、什么身份、有什么权限、这个环境长什么样"

---

### 1.1 主机基础信息识别

#### `systeminfo` — 系统全景快照

```cmd
:: 基本用法：显示完整系统信息
systeminfo

:: 输出为 CSV 格式（方便后续脚本处理）
systeminfo /fo csv

:: 输出为列表格式（便于 findstr 过滤）
systeminfo /fo list | findstr /i "OS Name"
systeminfo /fo list | findstr /i "Hotfix"

:: 远程查询（需要管理员凭据）
systeminfo /s <远程主机IP> /u <域名\用户名> /p <密码>
```

**安全场景应用**：
- 🔍 **应急响应**：快速获取系统启动时间（`System Boot Time`）判断是否近期重启过
- 🔍 **漏洞评估**：通过 `Hotfix(s)` 列表对比 CVE 数据库，判断是否缺少关键补丁
- 🔍 **架构识别**：`System Type` 字段区分 x86/x64/ARM64，决定后续利用链选择

**关键字段解读**：
| 字段 | 安全意义 |
|------|----------|
| `OS Name / Version` | 确定攻击面（如 Win7 有大量已知漏洞） |
| `System Boot Time` | 判断运行时长，长时间未重启可能留存更多痕迹 |
| `Hotfix(s)` | 补丁状态，缺失关键补丁 = 可利用漏洞 |
| `System Type` | x64 系统可能有 WoW64 兼容性利用点 |
| `Hyper-V Requirements` | 是否运行在虚拟机中（沙箱/蜜罐检测） |

---

#### `hostname` / `set` — 环境快速指纹

```cmd
:: 获取计算机名
hostname

:: 查看所有环境变量（信息量极大！）
set

:: 精确过滤特定变量
echo %COMPUTERNAME%
echo %USERDOMAIN%
echo %LOGONSERVER%
echo %PROCESSOR_ARCHITECTURE%
echo %HOMEDRIVE%%HOMEPATH%
echo %APPDATA%
echo %TEMP%
echo %PATH%
```

**安全场景应用**：
- 🔍 `%USERDOMAIN%` vs `%COMPUTERNAME%`：若两者不同，说明当前机器加入了域
- 🔍 `%LOGONSERVER%`：指向认证用的域控（DC），是内网渗透的第一跳目标
- 🔍 `%PATH%`：路径劫持攻击的入口，检查是否存在可写目录在系统 PATH 中
- 🔍 `%TEMP%` / `%APPDATA%`：恶意软件常见的落盘位置

---

#### `ver` / PowerShell 版本查询

```cmd
:: CMD 中查看版本号
ver
:: 输出示例：Microsoft Windows [Version 10.0.22631.3737]
```

```powershell
# PowerShell 精确版本查询
[System.Environment]::OSVersion
# 输出包含 Platform, Version, VersionString, Major, Minor, Build, Revision

# 获取完整构建号（Build Number 是判断功能可用性的关键）
(Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion").CurrentBuild

# 更详细的版本信息
(Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion") | 
    Select-Object ProductName, CurrentBuild, UBR, DisplayVersion, InstallationType

# 判断 Windows 10 vs 11（Build >= 22000 即为 Win11）
$build = [System.Environment]::OSVersion.Version.Build
if ($build -ge 22000) { "Windows 11" } else { "Windows 10" }
```

**版本对照表（安全相关功能）**：

| Build | 版本 | 关键安全特性 |
|-------|------|-------------|
| 14393 | Win10 1607 / Server 2016 | Credential Guard, Device Guard |
| 17763 | Win10 1809 / Server 2019 | WDAC 增强, HVCI |
| 19041 | Win10 2004 | WSL2 GA, Virtualization-Based Security |
| 22000 | Win11 21H2 | Pluton TPM, Smart App Control |
| 22631 | Win11 23H2 | Local Security Authority protection |
| 26100 | Win11 24H2 | Recall（争议性功能）, Rust in kernel |

---

### 1.2 用户、组与权限审计

#### `whoami` — 身份全景扫描（⭐ 极重要）

```cmd
:: 基本：显示 域\用户名
whoami

:: 完整信息：用户、组、特权、标签（最常用）
whoami /all

:: 仅查看特权列表（判断是否有提权机会）
whoami /priv

:: 仅查看组成员身份
whoami /groups

:: 以 XML 格式输出（方便脚本解析）
whoami /all /fo xml

:: 查看当前用户的 SID（安全标识符）
whoami /user

:: 查看完整性级别（Integrity Level）
whoami /all | findstr /i "Mandatory Label"
```

**特权清单与安全意义**：

| 特权名称 | 危险等级 | 攻击用途 |
|----------|---------|----------|
| `SeDebugPrivilege` | 🔴 极高 | 注入任意进程（如 LSASS），提取凭据 |
| `SeImpersonatePrivilege` | 🔴 极高 | Potato 系列提权（Juicy/Sweet/Rogue） |
| `SeTakeOwnershipPrivilege` | 🟠 高 | 夺取任意文件/注册表键所有权 |
| `SeBackupPrivilege` | 🟠 高 | 绕过 ACL 读取任意文件（含 SAM/NTDS） |
| `SeRestorePrivilege` | 🟠 高 | 绕过 ACL 写入任意文件 |
| `SeLoadDriverPrivilege` | 🟠 高 | 加载恶意内核驱动 |
| `SeTcbPrivilege` | 🔴 极高 | 充当操作系统的一部分 |
| `SeAssignPrimaryTokenPrivilege` | 🟠 高 | 替换进程令牌，提权 |
| `SeShutdownPrivilege` | 🟢 低 | 关机/重启（拒绝服务） |

**完整性级别（Integrity Level）**：
```
Untrusted (0)    → 沙箱内进程（IE Protected Mode）
Low (1000)       → 受限进程
Medium (2000)    → 标准用户（默认）
High (3000)      → 管理员（UAC 提升后）
System (4000)    → SYSTEM 账户
```

---

#### `net user` / `net localgroup` / `net group` — 用户与组枚举

```cmd
:: ===== 本地用户枚举 =====
:: 列出所有本地用户
net user

:: 查看特定用户详细信息
net user administrator
net user <username>

:: 创建用户（渗透后持久化 / 管理操作）
net user hacker P@ssw0rd123! /add
:: 将用户加入管理员组
net localgroup administrators hacker /add

:: ===== 本地组枚举 =====
:: 列出所有本地组
net localgroup

:: 查看管理员组成员
net localgroup administrators
:: 查看远程桌面用户组
net localgroup "Remote Desktop Users"
:: 查看所有组的成员
net localgroup administrators && net localgroup "backup operators" && net localgroup "remote desktop users"

:: ===== 域环境枚举（需要域成员身份）=====
:: 列出所有域用户
net user /domain
:: 查看特定域用户信息
net user <username> /domain
:: 列出所有域组
net group /domain
:: 查看域管理员组
net group "Domain Admins" /domain
:: 查看域控制器组
net group "Domain Controllers" /domain
:: 查看企业管理员（林中最高权限组）
net group "Enterprise Admins" /domain
```

**PowerShell 增强版**：
```powershell
# 本地用户详细信息（含启用状态、密码过期、最后登录时间）
Get-LocalUser | Format-Table Name, Enabled, LastLogon, PasswordLastSet, Description -AutoSize

# 本地管理员组详细成员
Get-LocalGroupMember -Group "Administrators" | Format-Table Name, PrincipalSource, ObjectClass

# 远程查询（需要凭据和 WinRM）
$cred = Get-Credential
Invoke-Command -ComputerName <target> -Credential $cred -ScriptBlock {
    Get-LocalUser | Select-Object Name, Enabled, LastLogon
}
```

---

#### `nltest` — 域信任关系与 DC 定位（⭐ AD 渗透核心）

```cmd
:: 列出域中所有域控制器
nltest /dclist:<domain_name>
:: 示例：nltest /dclist:corp.example.com

:: 获取特定域的首选 DC（含 GC 信息）
nltest /dsgetdc:<domain_name>
:: 加 /gc 参数强制返回全局编录服务器
nltest /dsgetdc:<domain_name> /gc

:: 查看域信任关系（极其重要！用于跨域/跨林攻击路径分析）
nltest /domain_trusts
:: 输出中关注：
::   - Trust Direction（双向/单向/入站/出站）
::   - Trust Type（林信任/外部信任/林转移）
::   - Trust Attributes（如 SID Filtering 是否启用）

:: 验证与特定 DC 的安全通道
nltest /sc_verify:<domain_name>

:: 强制刷新 DC 缓存
nltest /dcgetdc:<domain_name> /forcerediscover

:: 查看当前站点信息
nltest /dsgetsite
```

**安全场景深度解读**：

```
# 信任关系分析示例输出
Trust information for domain corp.local:
  0: DEV (NT 5) (Direct Outbound) (Direct Inbound)
     Trust Attributes: 0x00000008 (FOREST_TRANSITIVE)
  1: PARTNER (NT 5) (Direct Outbound)
     Trust Attributes: 0x00000004 (TREAT_AS_EXTERNAL)
```

- **FOREST_TRANSITIVE**（林信任）：可跨林传递 Kerberos 票据 → 可利用 SID History 注入
- **TREAT_AS_EXTERNAL**：外部信任，SID Filtering 通常启用 → 利用受限
- **Direct Outbound**：当前域信任目标域 → 可请求目标域资源的服务票据

---

#### `dsquery` + `dsget` — AD 对象 LDAP 精细查询

```cmd
:: 前提：需要安装 RSAT（Remote Server Administration Tools）
:: 但在域控上原生可用

:: 查询所有用户（限制返回数量）
dsquery user -limit 500

:: 按条件查询用户
dsquery user -name "admin*"
dsquery user -desc "*administrator*" -limit 100
dsquery user -inactive 4    # 4周未活动的用户（僵尸账户检测）

:: 查询所有计算机
dsquery computer -limit 500
dsquery computer -inactive 8   # 8周未活动的计算机

:: 查询所有组
dsquery group -limit 500
dsquery group -name "*admin*"

:: 查询 OU 结构
dsquery ou

:: 组合使用：查询用户并获取详细信息
dsquery user -name "john" | dsget user -samid -email -dept -manager -memberof

:: 查询组嵌套关系（组成员的组成员）
dsquery group -name "Domain Admins" | dsget group -members -expand

:: 查询特定 OU 下的所有对象
dsquery * "OU=IT,DC=corp,DC=local" -filter "(&(objectCategory=person)(objectClass=user))"
```

**PowerShell AD 替代方案（无需 RSAT）**：
```powershell
# 使用 .NET System.DirectoryServices 直接查询 LDAP
$domain = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
$root = [ADSI]"LDAP://$($domain.Name)"
$searcher = New-Object System.DirectoryServices.DirectorySearcher($root)
$searcher.Filter = "(&(objectCategory=person)(objectClass=user)(adminCount=1))"
$searcher.PropertiesToLoad.Add("samAccountName")
$searcher.PropertiesToLoad.Add("lastLogonTimestamp")
$results = $searcher.FindAll()
foreach ($r in $results) {
    $r.Properties["samAccountName"]
}
```

---

#### `klist` — Kerberos 票据缓存分析

```cmd
:: 查看当前用户的所有 Kerberos 票据
klist

:: 查看特定票据的详细信息
klist -li     # 使用登录 ID
klist -s      # 显示票据的会话密钥类型

:: 清除所有 Kerberos 票据（防御：清除攻击者获取的票据）
klist purge

:: 清除特定服务的票据
klist purge -li <LoginId>

:: 查看票据缓存文件位置
klist -c
```

**安全意义**：
- 🔴 **攻击视角**：票据缓存中可能包含高价值服务（如 MSSQL、CIFS）的 TGS → Pass-the-Ticket 攻击
- 🟢 **防御视角**：异常的服务票据请求模式（短时间内大量不同 SPN）= Kerberoasting 指标
- 🔍 **取证**：`klist` 输出中的 `EndTime` 和 `RenewTime` 可帮助构建活动时间线

---

### 1.3 进程、服务与启动项

#### `tasklist` / `taskkill` — 进程全景

```cmd
:: 基础进程列表
tasklist

:: 显示进程关联的服务（关键！用于发现伪装服务）
tasklist /svc

:: 显示详细模式（含 CPU 时间、窗口标题等）
tasklist /v

:: 显示命令行参数（需管理员权限，极重要！）
wmic process get name,processid,commandline /format:csv > processes.csv

:: 按条件过滤
tasklist /fi "imagename eq svchost.exe" /svc
tasklist /fi "username eq SYSTEM"
tasklist /fi "memusage gt 100000"    # 内存 > 100MB 的进程
tasklist /fi "status eq running"

:: 显示进程树（PowerShell 增强版）
tasklist /v /fo csv | sort

:: 远程查看进程
tasklist /s <远程IP> /u <用户> /p <密码> /svc

:: 终止进程
taskkill /pid <PID> /f          # 按 PID 强制终止
taskkill /im notepad.exe /f     # 按名称强制终止
taskkill /im notepad.exe /t     # 终止进程树（含子进程）
```

**进程分析实战技巧**：
```powershell
# PowerShell 高级进程分析：发现可疑进程
# 1. 没有可执行路径的进程（可能是注入）
Get-Process | Where-Object { $_.Path -eq $null -and $_.SessionId -ne 0 } |
    Select-Object Name, Id, StartTime

# 2. 从异常位置启动的进程
Get-Process | Where-Object { $_.Path -match "temp|appdata|users\\public" } |
    Select-Object Name, Id, Path

# 3. 进程父子关系（发现进程注入或恶意子进程）
Get-CimInstance Win32_Process | 
    Select-Object Name, ProcessId, ParentProcessId, CommandLine, ExecutablePath |
    Sort-Object ParentProcessId | Format-Table -AutoSize

# 4. 检测进程名伪装（如 svchost.exe 不在 system32 目录下）
Get-Process | Where-Object { 
    $_.Name -eq "svchost" -and $_.Path -notlike "*\Windows\System32\*" 
} | Select-Object Name, Id, Path
```

---

#### `sc` — 服务管理（⭐ 横向移动核心工具）

```cmd
:: ===== 查询服务 =====
:: 列出所有服务及状态
sc query type= service state= all

:: 查看特定服务状态
sc query <service_name>
sc query wuauserv        # Windows Update 服务

:: 查看服务配置（二进制路径、启动类型、依赖关系）
sc qc <service_name>
:: 输出关键字段：
::   BINARY_PATH_NAME → 可执行文件路径（可能被篡改）
::   START_TYPE       → 自动/手动/禁用
::   SERVICE_START_NAME → 运行账户（LocalSystem/NetworkService/自定义）

:: 查看服务安全描述符（谁可以启动/停止/修改服务）
sc sdshow <service_name>
:: SDDL 格式输出，解析需要理解安全描述符语法

:: ===== 创建服务（横向移动经典手法）=====
:: 在远程机器上创建服务
sc \\<target_ip> create <service_name> binpath= "cmd /c <payload>" start= auto
sc \\<target_ip> start <service_name>
sc \\<target_ip> delete <service_name>

:: 修改现有服务的二进制路径（服务劫持）
sc config <service_name> binpath= "C:\malicious.exe"

:: 修改服务运行账户（权限提升）
sc config <service_name> obj= ".\LocalSystem"

:: 查看服务失败操作（可能触发重启实现持久化）
sc qfailure <service_name>
```

**PowerShell 服务深度分析**：
```powershell
# 找出所有自动启动但当前未运行的服务（可能被关闭的恶意服务）
Get-Service | Where-Object { 
    $_.StartType -eq "Automatic" -and $_.Status -ne "Running" 
} | Select-Object Name, DisplayName, Status, StartType

# 查找服务二进制路径异常（未引用路径漏洞 → 提权）
Get-CimInstance Win32_Service | Where-Object {
    $_.PathName -notlike '*"*' -and $_.PathName -match '\s'
} | Select-Object Name, PathName, StartName | Format-List

# 导出所有服务信息用于审计
Get-CimInstance Win32_Service | Select-Object Name, DisplayName, State, 
    StartMode, StartName, PathName | Export-Csv -Path "services_audit.csv" -NoTypeInformation
```

---

#### `schtasks` — 计划任务（⭐ 持久化检测核心）

```cmd
:: 列出所有计划任务（详细列表格式）
schtasks /query /fo LIST /v

:: 导出为 CSV（便于后续分析）
schtasks /query /fo CSV /v > scheduled_tasks.csv

:: 查看特定任务详情
schtasks /query /tn "\Microsoft\Windows\UpdateOrchestrator\*" /v /fo LIST

:: 按文件夹查询
schtasks /query /fo LIST /v | findstr /i "TaskName\|Task To Run\|Run As User\|Last Run"

:: 创建计划任务（攻击者持久化手段）
schtasks /create /tn "WindowsDefender" /tr "C:\temp\backdoor.exe" /sc onlogon /ru SYSTEM

:: 删除计划任务
schtasks /delete /tn "WindowsDefender" /f

:: 远程管理计划任务
schtasks /query /s <target_ip> /u <user> /p <pass> /v /fo LIST
schtasks /create /s <target_ip> /tn "task_name" /tr "command" /sc daily /st 03:00
```

**PowerShell 高级任务分析**：
```powershell
# 获取所有计划任务的完整信息
Get-ScheduledTask | Where-Object { $_.State -ne "Disabled" } | ForEach-Object {
    $task = $_
    $actions = ($task.Actions | ForEach-Object { $_.Execute + " " + $_.Arguments }) -join "; "
    $triggers = ($task.Triggers | ForEach-Object { $_.CimClass.CimClassName }) -join "; "
    [PSCustomObject]@{
        Name       = $task.TaskName
        Path       = $task.TaskPath
        State      = $task.State
        Actions    = $actions
        Triggers   = $triggers
        Principal  = $task.Principal.UserId
    }
} | Format-Table -AutoSize -Wrap

# 发现可疑任务：非微软路径、非 SYSTEM/NETWORK 运行的
Get-ScheduledTask | Where-Object {
    $_.Actions.Execute -notmatch "Microsoft|Windows|svchost" -and 
    $_.Actions.Execute -ne $null
} | Select-Object TaskName, @{N='Exe';E={$_.Actions.Execute}}, 
    @{N='Args';E={$_.Actions.Arguments}}, State
```

---

#### `wmic` — WMI 命令行（信息收集的瑞士军刀）

```cmd
:: ===== 进程查询 =====
:: 获取所有进程的命令行（最常用！）
wmic process get name,processid,commandline /format:csv

:: 查找特定进程
wmic process where "name='svchost.exe'" get processid,commandline

:: ===== 补丁查询（比 systeminfo 快得多）=====
wmic qfe list brief /format:csv
wmic qfe get hotfixid,installedon,description

:: ===== 服务查询 =====
wmic service get name,pathname,startmode,state,startname /format:csv

:: ===== 启动项查询 =====
wmic startup get name,command,location /format:csv

:: ===== 用户账户 =====
wmic useraccount get name,sid,disabled,lockout /format:csv

:: ===== 共享资源 =====
wmic share get name,path,description /format:csv

:: ===== 网络适配器 =====
wmic nicconfig get description,ipaddress,macaddress /format:csv

:: ===== 远程执行（横向移动）=====
wmic /node:<target> process call create "cmd /c <payload>"

:: ===== 产品/软件清单 =====
wmic product get name,version,vendor /format:csv
```

> ⚠️ **注意**：`wmic` 在 Windows 11 24H2+ 和 Windows Server 2025 中已被标记为弃用，建议使用 PowerShell CIM cmdlet 替代。

---

#### `Get-CimInstance` — PowerShell CIM 替代方案

```powershell
# 等价于 wmic 的 CIM 查询，支持远程和更强大的过滤

# 进程信息
Get-CimInstance Win32_Process | Select-Object Name, ProcessId, CommandLine, ExecutablePath

# 补丁信息
Get-CimInstance Win32_QuickFixEngineering | Select-Object HotFixID, InstalledOn, Description

# 服务信息
Get-CimInstance Win32_Service | Select-Object Name, State, StartMode, PathName, StartName

# 系统信息
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version, BuildNumber, 
    LastBootUpTime, FreePhysicalMemory, TotalVisibleMemorySize

# 磁盘信息
Get-CimInstance Win32_LogicalDisk | Select-Object DeviceID, VolumeName, Size, FreeSpace

# 远程查询（需要 WinRM）
Get-CimInstance -ComputerName <target> -ClassName Win32_Process -Credential (Get-Credential)

# 使用 CIM Session（批量远程操作更高效）
$session = New-CimSession -ComputerName server1, server2, server3 -Credential (Get-Credential)
Get-CimInstance -CimSession $session -ClassName Win32_OperatingSystem
Remove-CimSession $session
```

---

## 二、网络侦察与连接分析

> **核心目标**：搞清楚"网络拓扑、开放端口、活跃连接、防火墙规则、共享资源"

---

### 2.1 网络配置与拓扑发现

#### `ipconfig` — 网络接口全景

```cmd
:: 完整网络配置（含 DNS、DHCP、MAC 地址）
ipconfig /all

:: 查看 DNS 缓存（泄露浏览历史和内网资产）
ipconfig /displaydns

:: 刷新 DNS 缓存（防御：清除痕迹）
ipconfig /flushdns

:: 释放/续租 DHCP 地址
ipconfig /release
ipconfig /renew

:: 注册 DNS（修复 DNS 记录）
ipconfig /registerdns

:: 将 DNS 缓存导出到文件
ipconfig /displaydns > dns_cache.txt
```

**DNS 缓存分析的安全价值**：
```cmd
:: 从 DNS 缓存中提取所有域名（发现内网资产）
ipconfig /displaydns | findstr "Record Name" | sort /unique
:: 关注以下模式：
::   *.internal.corp     → 内网域名
::   *.dev.corp          → 开发环境
::   *.admin.corp        → 管理面板
::   jira/confluence/git → 协作工具
```

---

#### `arp` / `route` — 网络拓扑发现

```cmd
:: 查看 ARP 缓存表（发现同网段活跃主机）
arp -a

:: 查看特定 IP 的 MAC 地址
arp -a | findstr <IP>

:: 添加静态 ARP 条目（中间人攻击/防御）
arp -s <IP> <MAC>

:: 完整路由表
route print

:: 仅查看 IPv4 路由
route print -4

:: 添加持久路由（网络后门）
route add <目标网段> mask <子网掩码> <网关> -p
```

**PowerShell 网络发现增强**：
```powershell
# ARP 表对象化输出
Get-NetNeighbor | Where-Object { 
    $_.State -eq "Reachable" -or $_.State -eq "Permanent" 
} | Select-Object IPAddress, LinkLayerAddress, InterfaceAlias, State | 
    Sort-Object IPAddress | Format-Table -AutoSize

# 路由表分析
Get-NetRoute | Where-Object { $_.DestinationPrefix -ne "255.255.255.255/32" } |
    Select-Object DestinationPrefix, NextHop, RouteMetric, InterfaceAlias |
    Sort-Object RouteMetric | Format-Table -AutoSize

# 网络适配器详细信息
Get-NetAdapter | Select-Object Name, Status, LinkSpeed, MacAddress, InterfaceDescription
```

---

#### `nslookup` / `Resolve-DnsName` — DNS 侦察

```cmd
:: 基本正向解析
nslookup www.example.com

:: 反向解析（IP → 主机名）
nslookup 192.168.1.1

:: 指定 DNS 服务器
nslookup www.example.com 8.8.8.8

:: 查询特定记录类型
nslookup -type=mx example.com        # 邮件服务器
nslookup -type=ns example.com        # 名称服务器
nslookup -type=txt example.com       # TXT 记录（SPF/DKIM/DMARC）
nslookup -type=soa example.com       # 起始授权机构
nslookup -type=any example.com       # 所有记录

:: 交互模式（批量查询）
nslookup
> server 8.8.8.8
> set type=any
> example.com
> exit

:: 区域传输尝试（如果 DNS 服务器配置不当）
nslookup
> server <target_dns_server>
> ls -d <domain>
```

```powershell
# PowerShell DNS 解析（功能更强大）
Resolve-DnsName -Name "example.com" -Type ANY
Resolve-DnsName -Name "example.com" -Server "8.8.8.8"

# 批量域名解析
$domains = @("mail.corp.local", "vpn.corp.local", "git.corp.local")
$domains | ForEach-Object { Resolve-DnsName $_ -ErrorAction SilentlyContinue } |
    Select-Object Name, Type, IPAddress | Format-Table -AutoSize

# 反向 DNS 扫描（发现内网主机名）
1..254 | ForEach-Object {
    Resolve-DnsName "192.168.1.$_" -ErrorAction SilentlyContinue |
        Select-Object Name, NameHost
}
```

---

### 2.2 端口、连接与防火墙

#### `netstat` — 网络连接全景（⭐ 应急响应必用）

```cmd
:: 所有连接和监听端口，显示 PID
netstat -ano

:: 显示每个连接对应的进程名（需要管理员权限）
netstat -abno

:: 仅 TCP 连接
netstat -ano -p tcp

:: 仅 UDP 监听
netstat -ano -p udp

:: 显示路由统计
netstat -r

:: 显示每个协议的统计
netstat -s

:: 每 5 秒刷新一次（持续监控）
netstat -ano 5

:: 组合过滤：查找特定端口的连接
netstat -ano | findstr ":445"     # SMB
netstat -ano | findstr ":3389"    # RDP
netstat -ano | findstr ":5985"    # WinRM
netstat -ano | findstr "ESTABLISHED"  # 已建立连接
```

**安全分析技巧**：
```cmd
:: 发现异常外联（ESTABLISHED 连接到非标准端口）
netstat -ano | findstr "ESTABLISHED" | findstr /v ":80 :443 :53"

:: 查找监听在 0.0.0.0 的服务（对所有接口暴露）
netstat -ano | findstr "LISTENING" | findstr "0.0.0.0"

:: 查找大量 TIME_WAIT（可能的端口扫描指标）
netstat -ano | findstr "TIME_WAIT" | find /c "TIME_WAIT"
```

```powershell
# PowerShell 增强版 netstat 分析
Get-NetTCPConnection | 
    Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State, 
    @{N='Process';E={(Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).ProcessName}},
    OwningProcess |
    Sort-Object State, RemotePort | Format-Table -AutoSize

# 发现可疑外联：非标准端口 + ESTABLISHED 状态
Get-NetTCPConnection -State Established | Where-Object {
    $_.RemotePort -notin @(80, 443, 53, 389, 636, 88) -and 
    $_.RemoteAddress -ne "127.0.0.1" -and
    $_.RemoteAddress -ne "::1"
} | Select-Object LocalPort, RemoteAddress, RemotePort, 
    @{N='Process';E={(Get-Process -Id $_.OwningProcess -EA SilentlyContinue).ProcessName}} |
    Format-Table -AutoSize
```

---

#### `Test-NetConnection` (`tnc`) — PowerShell 一体化网络诊断

```powershell
# 基本连通性测试
Test-NetConnection -ComputerName "example.com"

# 端口连通性测试（替代 telnet）
Test-NetConnection -ComputerName "192.168.1.1" -Port 445
Test-NetConnection -ComputerName "dc01.corp.local" -Port 3389

# 详细模式（含路由信息）
Test-NetConnection -ComputerName "example.com" -InformationLevel Detailed

# 批量端口扫描（轻量级端口扫描器）
$ports = @(22, 80, 135, 139, 443, 445, 1433, 3306, 3389, 5985, 5986, 8080, 8443)
foreach ($port in $ports) {
    $result = Test-NetConnection -ComputerName "192.168.1.100" -Port $port -WarningAction SilentlyContinue
    if ($result.TcpTestSucceeded) {
        "[OPEN] Port $port"
    }
}

# 路由追踪
Test-NetConnection -ComputerName "example.com" -TraceRoute

# ICMP 测试
Test-NetConnection -ComputerName "192.168.1.1" -CommonTCPPort SMB
Test-NetConnection -ComputerName "192.168.1.1" -CommonTCPPort RDP
Test-NetConnection -ComputerName "192.168.1.1" -CommonTCPPort WINRM
Test-NetConnection -ComputerName "192.168.1.1" -CommonTCPPort HTTP
```

---

#### `netsh advfirewall` — 防火墙规则管理

```cmd
:: ===== 查看防火墙状态 =====
netsh advfirewall show allprofiles state

:: ===== 导出所有防火墙规则 =====
netsh advfirewall firewall show rule name=all dir=in
netsh advfirewall firewall show rule name=all dir=out
:: 导出到文件
netsh advfirewall firewall show rule name=all dir=in > fw_inbound.txt
netsh advfirewall firewall show rule name=all dir=out > fw_outbound.txt

:: 仅显示已启用的规则
netsh advfirewall firewall show rule name=all dir=in status=enabled

:: 查看特定规则详情
netsh advfirewall firewall show rule name="Remote Desktop" verbose

:: ===== 管理规则 =====
:: 添加规则：允许特定端口
netsh advfirewall firewall add rule name="Allow SMB" dir=in action=allow protocol=tcp localport=445

:: 添加规则：阻止特定 IP
netsh advfirewall firewall add rule name="Block Attacker" dir=in action=block remoteip=10.0.0.100

:: 添加规则：允许特定程序
netsh advfirewall firewall add rule name="Allow App" dir=out action=allow program="C:\app\app.exe"

:: 删除规则
netsh advfirewall firewall delete rule name="Allow SMB"

:: 重置所有防火墙规则（危险！）
netsh advfirewall reset

:: ===== 导出/导入完整配置 =====
netsh advfirewall export "C:\fw_backup.wfw"
netsh advfirewall import "C:\fw_backup.wfw"
```

**PowerShell 防火墙分析**：
```powershell
# 获取所有已启用的入站规则
Get-NetFirewallRule -Direction Inbound -Enabled True | 
    Select-Object DisplayName, Action, Profile, 
    @{N='LocalPort';E={($_ | Get-NetFirewallPortFilter).LocalPort}},
    @{N='Protocol';E={($_ | Get-NetFirewallPortFilter).Protocol}} |
    Where-Object { $_.Action -eq "Allow" } |
    Sort-Object LocalPort | Format-Table -AutoSize

# 发现危险的防火墙规则（允许所有端口入站）
Get-NetFirewallRule -Direction Inbound -Enabled True -Action Allow |
    Where-Object { 
        ($rule = $_ | Get-NetFirewallPortFilter) -and 
        $rule.LocalPort -eq $null -and 
        $rule.Protocol -eq $null 
    } | Select-Object DisplayName, DisplayGroup

# 发现允许外联到任意目标的规则
Get-NetFirewallRule -Direction Outbound -Enabled True -Action Allow |
    Where-Object { ($_.Profile -match "Any") } |
    Select-Object DisplayName, Action, Profile | Format-Table
```

---

### 2.3 原生网络探测与共享

#### `ping` / `tracert` / `pathping`

```cmd
:: 基本 ping
ping <target>

:: 持续 ping（监控）
ping <target> -t

:: 指定包大小（检测 MTU 问题）
ping <target> -l 1500

:: 不分片（PMTU 发现）
ping <target> -f -l 1472

:: 指定 TTL（存活时间探测）
ping <target> -i 1

:: IPv6 ping
ping <target> -6

:: 路由追踪
tracert <target>
tracert -d <target>         # 不解析主机名（更快）
tracert -w 1000 <target>    # 超时 1 秒

:: 路径分析（结合 ping + tracert）
pathping <target>
pathping -n <target>        # 不解析主机名
pathping -q 10 <target>     # 每跳 10 个查询
```

---

#### `nbtstat` — NetBIOS 信息

```cmd
:: 通过 NetBIOS 名称查询远程主机信息
nbtstat -a <IP地址>
:: 输出包含：计算机名、工作组/域名、MAC 地址

:: 查看本地 NetBIOS 名称表
nbtstat -n

:: 查看 NetBIOS 名称缓存
nbtstat -c

:: 查看 NetBIOS 会话统计
nbtstat -s

:: 刷新 NetBIOS 名称缓存
nbtstat -R

:: 重新注册 NetBIOS 名称
nbtstat -RR
```

---

#### `net view` / `net use` — SMB 共享枚举

```cmd
:: 查看本机共享
net share

:: 查看远程主机共享（SMB 枚举）
net view \\<target_ip>
net view \\<target_ip> /all     # 包含隐藏共享（如 C$, ADMIN$）

:: 查看域中的所有计算机
net view /domain
net view /domain:<domain_name>

:: 映射网络驱动器
net use Z: \\<target>\share /user:<domain\user> <password>

:: 查看当前映射
net use

:: 空会话测试（无凭据访问，如果成功说明配置不安全）
net use \\<target>\IPC$ "" /user:""

:: 删除映射
net use Z: /delete

:: 删除所有映射
net use * /delete
```

```powershell
# PowerShell SMB 精细管理

# 查看本地所有共享
Get-SmbShare | Select-Object Name, Path, Description, CurrentUses | Format-Table -AutoSize

# 查看当前活跃的 SMB 会话（发现谁连到了本机）
Get-SmbSession | Select-Object ClientComputerName, ClientUserName, Dialect, NumOpens

# 查看当前 SMB 连接（本机连出去的）
Get-SmbConnection | Select-Object ServerName, ShareName, Credential, Dialect

# 查看 SMB 共享权限
Get-SmbShareAccess -Name "共享名" | Format-Table AccountName, AccessControlType, AccessRight

# 查看 SMB 服务器配置
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol, EnableSMB2Protocol, 
    EncryptData, RequireSecuritySignature, AnnounceServer
```

---

## 三、文件系统与数据检索

> **核心目标**：搜索敏感文件、提取取证数据、构建活动时间线

---

### 3.1 文件遍历、属性与哈希

#### `dir` / `Get-ChildItem` — 文件系统深度遍历

```cmd
:: 递归列出所有文件（含隐藏、系统文件）
dir /s /b /a:h,s,d C:\

:: 按扩展名搜索
dir /s /b *.config
dir /s /b *.xml
dir /s /b *.bak
dir /s /b *.log

:: 搜索特定文件
dir /s /b unattend.xml
dir /s /b sysprep.xml
dir /s /b web.config
dir /s /b *.kdbx          # KeePass 数据库

:: 查看备用数据流（ADS）— 恶意软件常用隐藏手法
dir /r C:\Users\
:: 输出中 :$DATA 后的额外流名即为 ADS
```

```powershell
# PowerShell 高级文件搜索

# 递归搜索含敏感关键字的文件名
Get-ChildItem -Path C:\ -Recurse -Force -ErrorAction SilentlyContinue |
    Where-Object { $_.Name -match "password|credential|secret|key|token|config" } |
    Select-Object FullName, Length, LastWriteTime | Format-Table -AutoSize

# 搜索最近 7 天内修改的文件（时间线构建）
$cutoff = (Get-Date).AddDays(-7)
Get-ChildItem -Path C:\Users -Recurse -Force -ErrorAction SilentlyContinue |
    Where-Object { $_.LastWriteTime -gt $cutoff -and -not $_.PSIsContainer } |
    Sort-Object LastWriteTime -Descending |
    Select-Object FullName, LastWriteTime, Length | Format-Table -AutoSize

# 搜索大文件（可能包含数据库或日志）
Get-ChildItem -Path C:\ -Recurse -Force -ErrorAction SilentlyContinue |
    Where-Object { $_.Length -gt 100MB } |
    Select-Object FullName, @{N='SizeMB';E={[math]::Round($_.Length/1MB,2)}}, LastWriteTime |
    Sort-Object SizeMB -Descending | Select-Object -First 50

# 检测备用数据流（ADS）
Get-ChildItem -Path C:\ -Recurse -Force -ErrorAction SilentlyContinue | 
    Get-Item -Stream * -ErrorAction SilentlyContinue | 
    Where-Object { $_.Stream -ne ':$DATA' } |
    Select-Object FileName, Stream, Length

# 搜索特定目录下的敏感文件
$sensitiveExtensions = @("*.kdbx", "*.pem", "*.key", "*.pfx", "*.p12", "*.rdp", "*.pst")
$paths = @("$env:USERPROFILE", "$env:APPDATA", "$env:LOCALAPPDATA", "$env:TEMP")
foreach ($path in $paths) {
    foreach ($ext in $sensitiveExtensions) {
        Get-ChildItem -Path $path -Filter $ext -Recurse -Force -ErrorAction SilentlyContinue
    }
}
```

---

#### `icacls` / `Get-Acl` / `Set-Acl` — NTFS 权限审计

```cmd
:: 查看文件/目录权限
icacls C:\Windows\System32\config\SAM
icacls C:\Users\Administrator\Desktop

:: 递归查看目录权限
icacls C:\inetpub /t

:: 查看特定用户的有效权限
icacls C:\temp /findsid <SID>

:: 保存权限（备份）
icacls C:\important /save acl_backup.txt /t

:: 恢复权限
icacls C:\ /restore acl_backup.txt

:: 授予权限
icacls C:\file.txt /grant Everyone:F
icacls C:\file.txt /grant "Domain Users":(R,X)

:: 移除权限
icacls C:\file.txt /remove Everyone

:: 设置所有者
icacls C:\file.txt /setowner "Administrator"
```

```powershell
# PowerShell ACL 深度分析

# 查看文件权限（对象化输出）
Get-Acl "C:\Windows\System32\config\SAM" | Format-List Path, Owner, Access

# 详细解析 ACE（访问控制条目）
(Get-Acl "C:\Windows\System32\config\SAM").Access | 
    Select-Object IdentityReference, FileSystemRights, AccessControlType, 
    InheritanceFlags, IsInherited | Format-Table -AutoSize

# 发现弱权限文件（Everyone 或 Users 有完全控制权限）
Get-ChildItem C:\ -Recurse -Force -ErrorAction SilentlyContinue |
    ForEach-Object {
        $acl = Get-Acl $_.FullName -ErrorAction SilentlyContinue
        if ($acl) {
            $weak = $acl.Access | Where-Object {
                ($_.IdentityReference -match "Everyone|BUILTIN\\Users") -and
                ($_.FileSystemRights -match "FullControl|Write") -and
                ($_.AccessControlType -eq "Allow")
            }
            if ($weak) {
                [PSCustomObject]@{
                    Path = $_.FullName
                    WeakACE = ($weak | ForEach-Object { "$($_.IdentityReference):$($_.FileSystemRights)" }) -join "; "
                }
            }
        }
    } | Export-Csv "weak_permissions.csv" -NoTypeInformation

# 修改 ACL
$acl = Get-Acl "C:\target"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "Everyone", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.AddAccessRule($rule)
Set-Acl -Path "C:\target" -AclObject $acl
```

---

#### `certutil` / `Get-FileHash` — 文件完整性校验

```cmd
:: 计算文件哈希
certutil -hashfile <filename> MD5
certutil -hashfile <filename> SHA1
certutil -hashfile <filename> SHA256

:: 批量计算目录下所有文件哈希（配合 for 循环）
for /r C:\temp %f in (*) do certutil -hashfile "%f" SHA256 >> hashes.txt
```

```powershell
# PowerShell 哈希计算（更灵活）
Get-FileHash "C:\Windows\System32\cmd.exe" -Algorithm SHA256

# 支持的算法：MD5, SHA1, SHA256, SHA384, SHA512, MACTripleDES, RIPEMD160
Get-FileHash "C:\Windows\System32\cmd.exe" -Algorithm SHA256 | Format-List

# 批量计算并对比（检测被篡改的系统文件）
$knownGood = @{
    "cmd.exe"     = "EXPECTED_HASH_HERE"
    "powershell.exe" = "EXPECTED_HASH_HERE"
}
foreach ($file in $knownGood.Keys) {
    $path = "C:\Windows\System32\$file"
    $current = (Get-FileHash $path -Algorithm SHA256).Hash
    if ($current -ne $knownGood[$file]) {
        "[ALERT] $file has been modified!"
    }
}

# 对整个目录计算哈希（构建基线）
Get-ChildItem C:\Windows\System32\*.exe -Force |
    ForEach-Object {
        [PSCustomObject]@{
            FileName = $_.Name
            Hash     = (Get-FileHash $_.FullName -Algorithm SHA256).Hash
            Size     = $_.Length
        }
    } | Export-Csv "system32_baseline.csv" -NoTypeInformation
```

---

#### `findstr` / `Select-String` — 关键字搜索

```cmd
:: 递归搜索文件内容
findstr /s /i "password" C:\Users\*.txt
findstr /s /i "connectionstring" C:\inetpub\*.config

:: 正则搜索
findstr /s /r /i "password.*=.*[a-zA-Z0-9]" *.ini

:: 仅显示文件名
findstr /s /m "secret" C:\Users\*.*

:: 搜索包含 IP 地址的文件
findstr /s /r "[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*" *.txt

:: 搜索包含邮箱的文件
findstr /s /i "@.*\.com" C:\Users\*.txt
```

```powershell
# PowerShell Select-String（更强大）

# 递归搜索敏感内容
Get-ChildItem C:\Users -Recurse -Include *.txt,*.xml,*.config,*.ini,*.log -Force -EA SilentlyContinue |
    Select-String -Pattern "password|passwd|credential|api.key|secret|token" -CaseSensitive:$false |
    Select-Object Path, LineNumber, Line | Format-Table -AutoSize -Wrap

# 搜索连接字符串
Get-ChildItem C:\inetpub -Recurse -Include *.config,*.xml -Force -EA SilentlyContinue |
    Select-String -Pattern "connectionstring|data source|server=|password=" |
    Select-Object Path, LineNumber, Line

# 搜索硬编码凭据（常见模式）
Get-ChildItem C:\ -Recurse -Include *.ps1,*.bat,*.cmd,*.vbs -Force -EA SilentlyContinue |
    Select-String -Pattern "(?i)(password|pass|pwd)\s*[=:]\s*['\"][^'\"]+['\"]" |
    Select-Object Path, LineNumber, Line

# 搜索内网 IP 地址
Get-ChildItem C:\Users -Recurse -Include *.txt,*.log,*.csv -Force -EA SilentlyContinue |
    Select-String -Pattern "\b(?:10|172\.(?:1[6-9]|2[0-9]|3[01])|192\.168)\.\d{1,3}\.\d{1,3}\b" |
    Select-Object Path, LineNumber, Line | Select-Object -First 50
```

---

### 3.2 敏感信息与取证数据提取

#### `wevtutil` / `Get-WinEvent` — 事件日志分析（⭐ 应急响应核心）

```cmd
:: 列出所有日志通道
wevtutil el > event_logs.txt

:: 查询安全日志（最近 20 条）
wevtutil qe Security /c:20 /rd:true /f:text

:: 查询特定事件（登录成功 Event ID 4624）
wevtutil qe Security /q:"*[System[EventID=4624]]" /c:50 /rd:true /f:text

:: 查询登录失败（暴力破解检测 Event ID 4625）
wevtutil qe Security /q:"*[System[EventID=4625]]" /c:100 /rd:true /f:text

:: 查询账户管理事件（用户创建/修改 Event ID 4720/4722/4724）
wevtutil qe Security /q:"*[System[EventID=4720 or EventID=4722 or EventID=4724]]" /rd:true

:: 查询进程创建（Event ID 4688，需启用审计策略）
wevtutil qe Security /q:"*[System[EventID=4688]]" /c:50 /rd:true /f:text

:: 查询服务安装（Event ID 7045）
wevtutil qe System /q:"*[System[EventID=7045]]" /rd:true /f:text

:: 查询 PowerShell 脚本执行（Event ID 4104）
wevtutil qe "Microsoft-Windows-PowerShell/Operational" /q:"*[System[EventID=4104]]" /c:20 /rd:true

:: 导出日志到文件
wevtutil epl Security C:\evidence\security.evtx
wevtutil qe Security /f:xml /rd:true /c:1000 > C:\evidence\security_1000.xml
```

```powershell
# PowerShell 事件日志分析（功能极其强大）

# 获取最近登录事件
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    ID = 4624
    StartTime = (Get-Date).AddDays(-1)
} -MaxEvents 50 | ForEach-Object {
    $xml = [xml]$_.ToXml()
    [PSCustomObject]@{
        TimeCreated     = $_.TimeCreated
        LogonType       = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'LogonType' } | Select -ExpandProperty '#text'
        TargetUser      = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'TargetUserName' } | Select -ExpandProperty '#text'
        SourceIP        = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'IpAddress' } | Select -ExpandProperty '#text'
        ProcessName     = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'ProcessName' } | Select -ExpandProperty '#text'
    }
} | Format-Table -AutoSize

# 暴力破解检测（统计失败登录次数）
Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4625} -MaxEvents 5000 -EA SilentlyContinue |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        [PSCustomObject]@{
            Time       = $_.TimeCreated
            SourceIP   = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'IpAddress' }).'#text'
            TargetUser = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'TargetUserName' }).'#text'
            FailReason = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'Status' }).'#text'
        }
    } | Group-Object SourceIP | Sort-Object Count -Descending | Select-Object Count, Name

# 检测新服务安装
Get-WinEvent -FilterHashtable @{LogName='System'; ID=7045} -MaxEvents 100 -EA SilentlyContinue |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        [PSCustomObject]@{
            TimeCreated = $_.TimeCreated
            ServiceName = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'ServiceName' }).'#text'
            ImagePath   = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'ImagePath' }).'#text'
            AccountName = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'AccountName' }).'#text'
        }
    } | Format-Table -AutoSize

# XPath 高级过滤
Get-WinEvent -LogName Security -FilterXPath "*[System[EventID=4688] and EventData[Data[@Name='CommandLine'] and (Data[@Name='CommandLine']='*powershell*' or Data[@Name='CommandLine']='*cmd*')]]" -MaxEvents 100
```

**关键事件 ID 速查表**：

| Event ID | 日志 | 描述 | 安全意义 |
|----------|------|------|----------|
| 4624 | Security | 登录成功 | 追踪用户/服务登录 |
| 4625 | Security | 登录失败 | 暴力破解检测 |
| 4634 | Security | 注销 | 会话生命周期 |
| 4648 | Security | 显式凭据登录 | 凭据使用（runas） |
| 4672 | Security | 特殊特权分配 | 管理员操作 |
| 4688 | Security | 进程创建 | 命令执行追踪 |
| 4697 | Security | 服务安装 | 持久化检测 |
| 4698 | Security | 计划任务创建 | 持久化检测 |
| 4720 | Security | 用户账户创建 | 后门账户 |
| 4724 | Security | 密码重置 | 账户接管 |
| 4732 | Security | 成员添加到安全组 | 权限提升 |
| 5140 | Security | 共享访问 | SMB 横向移动 |
| 5156 | Security | 网络连接 | 防火墙允许的连接 |
| 7045 | System | 新服务安装 | 持久化/恶意软件 |
| 1102 | Security | 日志清除 | 反取证 |
| 4104 | PS Operational | 脚本块日志 | PowerShell 执行 |
| 800 | PS Classic | 管道执行 | 传统 PS 日志 |

---

#### `reg query` — 注册表查询

```cmd
:: 查询特定键值
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
reg query "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"

:: 递归查询（含子键）
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /s

:: 搜索特定值
reg query HKLM /s /f "password" /t REG_SZ
reg query HKCU /s /f "mimikatz" /t REG_SZ

:: 查看 SAM 相关信息（需要 SYSTEM 权限）
reg query HKLM\SAM\SAM /s

:: 查看远程注册表
reg query \\<target>\HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run

:: 查看自动登录凭据
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon

:: 查看 RDP 保存的凭据位置
reg query "HKCU\SOFTWARE\Microsoft\Terminal Server Client\Servers" /s

:: 查看已安装的软件
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall" /s | findstr DisplayName

:: 查看 PuTTY 保存的会话（含主机名）
reg query "HKCU\SOFTWARE\SimonTatham\PuTTY\Sessions" /s

:: 查看 SNMP 社区字符串
reg query "HKLM\SYSTEM\CurrentControlSet\Services\SNMP\Parameters\ValidCommunities" /s
```

```powershell
# PowerShell 注册表操作

# 查看所有启动项（Run 键）
$paths = @(
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run",
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce",
    "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run",
    "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce",
    "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Run"
)
foreach ($path in $paths) {
    if (Test-Path $path) {
        Write-Host "`n=== $path ===" -ForegroundColor Yellow
        Get-ItemProperty $path | Select-Object * -ExcludeProperty PS* | Format-List
    }
}

# 搜索注册表中的密码
$regPaths = @("HKLM:\SOFTWARE", "HKCU:\SOFTWARE")
foreach ($rp in $regPaths) {
    Get-ChildItem $rp -Recurse -ErrorAction SilentlyContinue | ForEach-Object {
        $key = $_
        $key.GetValueNames() | ForEach-Object {
            $value = $key.GetValue($_)
            if ($value -match "password|pass|pwd|secret|key" -and $value -is [string]) {
                [PSCustomObject]@{
                    Path  = $key.Name
                    Name  = $_
                    Value = $value
                }
            }
        }
    }
}
```

---

### 3.3 时间线与活动痕迹

#### `forfiles` / 时间线构建

```cmd
:: 查找最近 7 天内修改的文件
forfiles /p C:\Users /s /d +01/01/2026 /c "cmd /c echo @path @fdate @ftime @fsize"

:: 查找 30 天前的文件
forfiles /p C:\ /s /d -30 /c "cmd /c echo @path @fdate"

:: 按文件类型搜索
forfiles /p C:\ /s /m *.exe /d +01/01/2026 /c "cmd /c echo @path @fdate @ftime"
```

```powershell
# 构建完整的时间线（取证核心技能）
# 合并文件修改时间、事件日志、注册表修改

# 1. 文件系统时间线
$fileTimeline = Get-ChildItem C:\Users -Recurse -Force -EA SilentlyContinue |
    Where-Object { -not $_.PSIsContainer } |
    Select-Object @{N='Timestamp';E={$_.LastWriteTime}}, 
                  @{N='Type';E={'FileModified'}}, 
                  @{N='Detail';E={$_.FullName}}

# 2. 最近访问文件（LNK 文件解析）
$recentPath = "$env:APPDATA\Microsoft\Windows\Recent"
Get-ChildItem $recentPath -Filter "*.lnk" | Select-Object Name, LastWriteTime |
    Sort-Object LastWriteTime -Descending | Select-Object -First 50

# 3. Prefetch 文件（程序执行历史，需管理员权限）
Get-ChildItem "C:\Windows\Prefetch" -Filter "*.pf" -EA SilentlyContinue |
    Select-Object Name, LastWriteTime, Length |
    Sort-Object LastWriteTime -Descending | Select-Object -First 50

# 4. 合并为统一时间线
$timeline = @()
$timeline += $fileTimeline | Select-Object -First 200
# 添加事件日志条目...
$timeline | Sort-Object Timestamp | Export-Csv "timeline.csv" -NoTypeInformation
```

---

## 四、安全防御、持久化检测与对抗

> **核心目标**：检测恶意行为、验证防御机制、识别横向移动痕迹

---

### 4.1 恶意行为指标（IOC）排查

#### 异常进程链与命令行分析

```powershell
# 检测可疑父子进程关系
# 正常：explorer.exe → notepad.exe
# 可疑：winword.exe → cmd.exe / powershell.exe（钓鱼文档指标）
$suspiciousChains = Get-CimInstance Win32_Process | ForEach-Object {
    $child = $_
    $parent = Get-CimInstance Win32_Process -Filter "ProcessId=$($child.ParentProcessId)" -EA SilentlyContinue
    [PSCustomObject]@{
        ParentName = if ($parent) { $parent.Name } else { "N/A (exited)" }
        ParentPID  = $child.ParentProcessId
        ChildName  = $child.Name
        ChildPID   = $child.ProcessId
        ChildCmd   = $child.CommandLine
    }
}

# 定义可疑模式
$suspicious = $suspiciousChains | Where-Object {
    # Office 应用启动了脚本解释器
    ($_.ParentName -match "winword|excel|powerpoint|outlook" -and 
     $_.ChildName -match "cmd|powershell|wscript|cscript|mshta") -or
    # 浏览器启动了命令行
    ($_.ParentName -match "chrome|firefox|msedge|iexplore" -and 
     $_.ChildName -match "cmd|powershell") -or
    # svchost 启动了非标准子进程
    ($_.ParentName -eq "svchost.exe" -and 
     $_.ChildName -match "cmd|powershell|whoami|net")
}
$suspicious | Format-Table -AutoSize -Wrap
```

#### Base64 编码命令解码

```powershell
# 解码 Base64 编码的 PowerShell 命令（-EncodedCommand 参数）
# 攻击者常用此方法混淆恶意命令

$encodedCmd = "JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAA="

# 解码（EncodedCommand 使用 Unicode/UTF-16LE 编码）
$decoded = [System.Text.Encoding]::Unicode.GetString([Convert]::FromBase64String($encodedCmd))
Write-Host "Decoded command: $decoded" -ForegroundColor Red

# 批量解码脚本块日志中的 Base64
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-PowerShell/Operational'; ID=4104} -MaxEvents 100 -EA SilentlyContinue |
    ForEach-Object {
        $scriptBlock = $_.Properties[2].Value  # ScriptBlockText
        if ($scriptBlock -match '$$Convert$$::FromBase64String$[''"]([^''"]+)[''"]$' -or
            $scriptBlock -match '-EncodedCommand\s+([A-Za-z0-9+/=]+)') {
            $base64 = $Matches[1]
            try {
                $decoded = [System.Text.Encoding]::Unicode.GetString([Convert]::FromBase64String($base64))
                [PSCustomObject]@{
                    Time    = $_.TimeCreated
                    Encoded = $base64.Substring(0, [Math]::Min(80, $base64.Length)) + "..."
                    Decoded = $decoded
                }
            } catch {
                # 非 Unicode 编码，尝试 UTF-8
                try {
                    $decoded = [System.Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($base64))
                    [PSCustomObject]@{
                        Time    = $_.TimeCreated
                        Encoded = $base64.Substring(0, [Math]::Min(80, $base64.Length)) + "..."
                        Decoded = $decoded
                    }
                } catch {}
            }
        }
    } | Format-Table -AutoSize -Wrap
```

#### WMI 永久事件订阅检测（⭐ 高级持久化）

```powershell
# WMI 永久事件订阅是一种高级持久化技术
# 攻击者创建 WMI 事件过滤器 + 消费者绑定，实现重启后自动执行

# 1. 检查事件过滤器（__EventFilter）
Get-WmiObject -Namespace "root\subscription" -Class __EventFilter | 
    Select-Object Name, Query, QueryLanguage, EventNamespace | Format-List

# 2. 检查事件消费者
# ActiveScriptEventConsumer（VBScript/JScript）
Get-WmiObject -Namespace "root\subscription" -Class ActiveScriptEventConsumer |
    Select-Object Name, ScriptingEngine, ScriptText | Format-List

# CommandLineEventConsumer（命令行执行）
Get-WmiObject -Namespace "root\subscription" -Class CommandLineEventConsumer |
    Select-Object Name, CommandLineTemplate, ExecutablePath | Format-List

# LogFileEventConsumer（日志写入）
Get-WmiObject -Namespace "root\subscription" -Class LogFileEventConsumer |
    Select-Object Name, Filename, MaximumFileSize | Format-List

# 3. 检查绑定关系（__FilterToConsumerBinding）
Get-WmiObject -Namespace "root\subscription" -Class __FilterToConsumerBinding |
    Select-Object Filter, Consumer | Format-List

# 4. 清除可疑的 WMI 持久化
# 删除绑定
Get-WmiObject -Namespace "root\subscription" -Class __FilterToConsumerBinding |
    Where-Object { $_.Filter -match "suspicious_name" } | Remove-WmiObject

# 删除消费者
Get-WmiObject -Namespace "root\subscription" -Class CommandLineEventConsumer |
    Where-Object { $_.Name -match "suspicious_name" } | Remove-WmiObject

# 删除过滤器
Get-WmiObject -Namespace "root\subscription" -Class __EventFilter |
    Where-Object { $_.Name -match "suspicious_name" } | Remove-WmiObject
```

#### IFEO 劫持检测

```cmd
:: 检查 Image File Execution Options（调试器劫持）
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options" /s

:: 重点关注 Debugger 值（正常系统极少配置）
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options" /s /f "Debugger"

:: 检查 SilentProcessExit（监控进程退出）
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit" /s
```

```powershell
# PowerShell IFEO 全面检测
$ifeoPath = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options"
Get-ChildItem $ifeoPath -ErrorAction SilentlyContinue | ForEach-Object {
    $props = Get-ItemProperty $_.PSPath -ErrorAction SilentlyContinue
    if ($props.Debugger -or $props.MonitorProcess -or $props.GlobalFlag) {
        [PSCustomObject]@{
            ImageName      = $_.PSChildName
            Debugger       = $props.Debugger
            MonitorProcess = $props.MonitorProcess
            GlobalFlag     = $props.GlobalFlag
        }
    }
} | Format-Table -AutoSize
```

---

### 4.2 防御机制验证与策略检查

#### Windows Defender 状态检查

```powershell
# Defender 完整状态报告
Get-MpComputerStatus | Select-Object 
    AMRunningMode,           # 运行模式（Normal/Passive/EDR Blocked）
    AMServiceEnabled,        # 反恶意软件服务是否启用
    AntispywareEnabled,      # 反间谍是否启用
    AntivirusEnabled,        # 反病毒是否启用
    RealTimeProtectionEnabled, # 实时保护
    BehaviorMonitorEnabled,  # 行为监控
    IoavProtectionEnabled,   # 下载扫描
    IsTamperProtected,       # 篡改保护
    AntivirusSignatureLastUpdated,  # 签名更新时间
    QuickScanEndTime,        # 上次快速扫描
    FullScanEndTime          # 上次完整扫描

# 查看威胁检测历史
Get-MpThreatDetection | Select-Object ThreatID, ProcessName, InitialDetectionTime, 
    Resources, DomainUser, ActionSuccess | Sort-Object InitialDetectionTime -Descending

# 查看当前威胁
Get-MpThreat | Select-Object ThreatName, SeverityID, IsActive, DidThreatExecute, 
    InitialDetectionTime, CleaningAction

# 查看排除项（攻击者可能添加排除项绕过检测）
Get-MpPreference | Select-Object -ExpandProperty ExclusionPath
Get-MpPreference | Select-Object -ExpandProperty ExclusionProcess
Get-MpPreference | Select-Object -ExpandProperty ExclusionExtension

# Defender 扫描操作
# 快速扫描
"%ProgramFiles%\Windows Defender\MpCmdRun.exe" -Scan -ScanType 1
# 完整扫描
"%ProgramFiles%\Windows Defender\MpCmdRun.exe" -Scan -ScanType 2
# 自定义路径扫描
"%ProgramFiles%\Windows Defender\MpCmdRun.exe" -Scan -ScanType 3 -File "C:\temp"
# 更新签名
"%ProgramFiles%\Windows Defender\MpCmdRun.exe" -SignatureUpdate
# 移除所有检测到的威胁
"%ProgramFiles%\Windows Defender\MpCmdRun.exe" -RemoveDefinitions -All
```

#### AppLocker / WDAC 策略分析

```powershell
# 查看生效的 AppLocker 策略
Get-AppLockerPolicy -Effective | Select-Object -ExpandProperty RuleCollections | 
    Format-Table Name, RuleCollectionType, EnforcementMode -AutoSize

# 导出 AppLocker 策略为 XML
Get-AppLockerPolicy -Effective -Xml > C:\applocker_policy.xml

# 详细分析每条规则
Get-AppLockerPolicy -Effective | ForEach-Object {
    $_.RuleCollections | ForEach-Object {
        $collection = $_
        $collection | ForEach-Object {
            [PSCustomObject]@{
                CollectionType = $collection.RuleCollectionType
                EnforcementMode = $collection.EnforcementMode
                RuleName = $_.Name
                Action = $_.Action
                Description = $_.Description
                UserOrGroup = $_.UserOrGroupSid
            }
        }
    }
} | Format-Table -AutoSize

# 测试文件是否会被 AppLocker 阻止
Get-AppLockerFileInformation -Path "C:\temp\suspicious.exe"
Test-AppLockerPolicy -XmlPolicy "C:\applocker_policy.xml" -Path "C:\temp\suspicious.exe"

# WDAC（Windows Defender Application Control）策略
Get-CIPolicy -Path "C:\Windows\System32\CodeIntegrity\CIPolicies\Active\*" -ErrorAction SilentlyContinue

# 检查 Code Integrity 状态
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace "root\Microsoft\Windows\DeviceGuard" -EA SilentlyContinue |
    Select-Object CodeIntegrityPolicyEnforcementStatus, UsermodeCodeIntegrityPolicyEnforcementStatus
```

#### 审计策略验证

```cmd
:: 查看完整审计策略
auditpol /get /category:*

:: 仅查看关键子类别
auditpol /get /subcategory:"Logon"
auditpol /get /subcategory:"Logoff"
auditpol /get /subcategory:"Process Creation"
auditpol /get /subcategory:"Account Lockout"
auditpol /get /subcategory:"File System"
auditpol /get /subcategory:"Registry"

:: 导出审计策略
auditpol /backup /file:C:\audit_policy.bak

:: 比较审计策略
auditpol /restore /file:C:\audit_policy.bak
```

```powershell
# PowerShell 审计策略分析
Get-AuditPolicy -Category * | ForEach-Object {
    $_.Subcategories | ForEach-Object {
        [PSCustomObject]@{
            Category    = $_.Category
            Subcategory = $_.Name
            Success     = $_.InclusionSetting -match "Success"
            Failure     = $_.InclusionSetting -match "Failure"
            Setting     = $_.InclusionSetting
        }
    }
} | Where-Object { -not ($_.Success -or $_.Failure) } |
    Select-Object Category, Subcategory | Format-Table -AutoSize
# 输出未启用的审计项（安全合规检查）
```

#### PowerShell 执行策略与语言模式

```powershell
# 查看所有范围的执行策略
Get-ExecutionPolicy -List
# 范围优先级：MachinePolicy > UserPolicy > Process > CurrentUser > LocalMachine

# 查看当前语言模式（安全关键！）
$ExecutionContext.SessionState.LanguageMode
# FullLanguage    = 完整功能（默认）
# ConstrainedLanguage = 受限语言模式（限制了 .NET 调用、COM 等）
# NoLanguage      = 无语言模式

# 如果处于 ConstrainedLanguage 模式，许多高级功能受限
# 攻击者可能尝试绕过：
# 1. 通过 PowerShell 2.0（如果已安装）
powershell.exe -version 2 -Command "$ExecutionContext.SessionState.LanguageMode"

# 2. 通过环境变量绕过
$env:__PSLockdownPolicy  # 如果设置了此变量，会触发 CLM

# 检查 PowerShell 日志配置
# 脚本块日志
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -EA SilentlyContinue
# 模块日志
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging" -EA SilentlyContinue
# 转录日志
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription" -EA SilentlyContinue
```

---

### 4.3 横向移动与远程执行痕迹

#### RDP/SMB/WinRM 登录分析

```powershell
# 综合分析横向移动痕迹

# 1. RDP 连接日志（Event ID 4624 LogonType 10 = RDP）
Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4624} -MaxEvents 5000 -EA SilentlyContinue |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        $logonType = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'LogonType' }).'#text'
        if ($logonType -eq '10' -or $logonType -eq '3') {  # RDP(10) or Network(3)
            [PSCustomObject]@{
                Time       = $_.TimeCreated
                LogonType  = switch($logonType) { '3' {'Network/SMB'} '10' {'RDP'} default {$logonType} }
                User       = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'TargetUserName' }).'#text'
                SourceIP   = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'IpAddress' }).'#text'
                AuthPkg    = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'AuthenticationPackageName' }).'#text'
            }
        }
    } | Where-Object { $_.SourceIP -ne '-' -and $_.SourceIP -ne '' } |
    Sort-Object Time -Descending | Format-Table -AutoSize

# 2. SMB 共享访问日志（Event ID 5140/5145）
Get-WinEvent -FilterHashtable @{LogName='Security'; ID=5140} -MaxEvents 1000 -EA SilentlyContinue |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        [PSCustomObject]@{
            Time      = $_.TimeCreated
            User      = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'SubjectUserName' }).'#text'
            SourceIP  = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'IpAddress' }).'#text'
            ShareName = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'ShareName' }).'#text'
        }
    } | Group-Object SourceIP | Sort-Object Count -Descending

# 3. WinRM 连接（Event ID 91 in Microsoft-Windows-WinRM/Operational）
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-WinRM/Operational'} -MaxEvents 100 -EA SilentlyContinue |
    Select-Object TimeCreated, Id, Message | Format-Table -AutoSize
```

#### 远程服务创建检测

```cmd
:: 在远程机器创建服务（经典横向移动手法）
:: 攻击者操作示例：
sc \\target create "WindowsHealth" binpath= "cmd /c powershell -enc <base64>" start= auto
sc \\target start "WindowsHealth"

:: 防御：检测 Event ID 7045（新服务安装）
:: 关注非常规路径的二进制文件
```

```powershell
# 检测远程创建的服务
Get-WinEvent -FilterHashtable @{LogName='System'; ID=7045} -MaxEvents 500 -EA SilentlyContinue |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        $svcName = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'ServiceName' }).'#text'
        $imgPath = ($xml.Event.EventData.Data | Where-Object { $_.Name -eq 'ImagePath' }).'#text'
        [PSCustomObject]@{
            Time        = $_.TimeCreated
            ServiceName = $svcName
            ImagePath   = $imgPath
            Suspicious  = ($imgPath -match "cmd|powershell|temp|appdata|users\\public" -or 
                          $imgPath -match "-enc|-encoded|bypass|download")
        }
    } | Where-Object { $_.Suspicious } | Format-Table -AutoSize -Wrap
```

#### `winrs` / `Invoke-Command` — 原生远程执行

```cmd
:: WinRM 远程命令执行（Windows Remote Shell）
winrs -r:<target> "whoami"
winrs -r:<target> -u:<domain\user> -p:<password> "ipconfig /all"

:: 交互式远程 shell
winrs -r:<target> cmd
```

```powershell
# PowerShell 远程管理

# 一对一交互式会话
Enter-PSSession -ComputerName <target> -Credential (Get-Credential)

# 一对多远程执行（fan-out）
Invoke-Command -ComputerName server1, server2, server3 -ScriptBlock {
    Get-Process | Where-Object { $_.CPU -gt 100 } | Select-Object Name, CPU
} -Credential (Get-Credential)

# 持久化远程会话
$session = New-PSSession -ComputerName <target> -Credential (Get-Credential)
Invoke-Command -Session $session -ScriptBlock { Get-EventLog -LogName Security -Newest 100 }
Enter-PSSession $session
Remove-PSSession $session

# 检测 WinRM 远程执行痕迹
# Event ID 91 (WinRM/Operational) 和 Event ID 4104 (PowerShell)
Get-WinEvent -FilterHashtable @{
    LogName = 'Microsoft-Windows-WinRM/Operational'
    StartTime = (Get-Date).AddDays(-7)
} -MaxEvents 100 -EA SilentlyContinue | Format-Table TimeCreated, Id, Message -AutoSize
```

#### RDP 会话管理

```cmd
:: 查看当前 RDP 会话
quser
:: 或
query user

:: 查看所有会话（含断开连接的）
qwinsta
:: 或
query session

:: 断开特定会话
logoff <session_id>

:: 远程查看
quser /server:<target>
qwinsta /server:<target>
```

```powershell
# PowerShell RDP 会话管理
Get-RDUserSession -ConnectionBroker <broker> -EA SilentlyContinue |
    Select-Object UserName, HostServer, SessionState, CreateTime, DisconnectTime

# 查看 RDP 连接历史（注册表）
Get-ChildItem "HKCU:\SOFTWARE\Microsoft\Terminal Server Client\Servers" -EA SilentlyContinue |
    ForEach-Object {
        [PSCustomObject]@{
            Server   = $_.PSChildName
            Username = (Get-ItemProperty $_.PSPath -EA SilentlyContinue).UsernameHint
        }
    } | Format-Table -AutoSize

# 查看 RDP 证书（自签名 = 可能是攻击者部署的）
Get-ChildItem "Cert:\LocalMachine\Remote Desktop\" | 
    Select-Object Subject, Thumbprint, NotBefore, NotAfter, Issuer
```

---

## 五、PowerShell 专属安全能力

---

### 5.1 对象化系统与 .NET 交互

#### WMI/CIM 深度利用

```powershell
# CIM 远程进程创建（替代 wmic）
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{
    CommandLine = "cmd /c whoami > C:\temp\output.txt"
} -ComputerName <target> -Credential (Get-Credential)

# 枚举所有 WMI 命名空间和类（发现安全相关的 WMI 类）
Get-CimClass -Namespace "root\SecurityCenter2" | Select-Object CimClassName
Get-CimClass -Namespace "root\Microsoft\Windows\Defender" | Select-Object CimClassName

# 安全中心状态
Get-CimInstance -Namespace "root\SecurityCenter2" -ClassName AntiVirusProduct |
    Select-Object DisplayName, ProductState, PathToSignedProductExe

# BitLocker 状态
Get-CimInstance -Namespace "root\cimv2\Security\MicrosoftVolumeEncryption" -ClassName Win32_EncryptableVolume |
    Select-Object DriveLetter, ProtectionStatus, EncryptionMethod
```

#### .NET 程序集加载与动态编译

```powershell
# 加载 .NET 程序集
[Reflection.Assembly]::Load([Convert]::FromBase64String("<base64_encoded_dll>"))
[Reflection.Assembly]::LoadFile("C:\path\to\assembly.dll")
[Reflection.Assembly]::LoadWithPartialName("System.Web")

# 动态编译 C# 代码（攻击者常用技术）
$source = @"
using System;
using System.Runtime.InteropServices;
public class NativeMethods {
    [DllImport("kernel32.dll")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
    
    [DllImport("kernel32.dll")]
    public static extern IntPtr GetModuleHandle(string lpModuleName);
}
"@
Add-Type -TypeDefinition $source -Language CSharp
# 现在可以调用 [NativeMethods]::GetProcAddress(...)

# 查看已加载的程序集（检测恶意加载）
[AppDomain]::CurrentDomain.GetAssemblies() | Select-Object FullName | Format-Table -AutoSize

# 检测可疑的动态编译
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-PowerShell/Operational'; ID=4104} -EA SilentlyContinue |
    Where-Object { $_.Message -match "Add-Type|Reflection\.Assembly" } |
    Select-Object TimeCreated, Message
```

---

### 5.2 脚本自动化与远程管理

#### 后台作业与并行执行

```powershell
# 创建后台作业
Start-Job -ScriptBlock {
    Get-ChildItem C:\ -Recurse -Force -EA SilentlyContinue |
        Where-Object { $_.Extension -eq ".kdbx" } |
        Select-Object FullName, LastWriteTime
} -Name "SearchKeePass"

# 查看所有作业状态
Get-Job | Select-Object Name, State, HasMoreData, PSBeginTime

# 获取作业结果
Receive-Job -Name "SearchKeePass"

# 等待作业完成（带超时）
Wait-Job -Name "SearchKeePass" -Timeout 60

# 并行执行（PowerShell 7+ ForEach-Object -Parallel）
$servers = @("server1", "server2", "server3", "server4", "server5")
$servers | ForEach-Object -Parallel {
    $result = Test-NetConnection -ComputerName $_ -Port 445 -WarningAction SilentlyContinue
    if ($result.TcpTestSucceeded) { "[OPEN] $_:445 (SMB)" }
} -ThrottleLimit 10

# Windows PowerShell 5.1 的并行替代方案（Runspace）
$runspacePool = [runspacefactory]::CreateRunspacePool(1, 10)
$runspacePool.Open()
$jobs = @()
foreach ($server in $servers) {
    $ps = [powershell]::Create().AddScript({
        param($server)
        Test-NetConnection -ComputerName $server -Port 445 -WarningAction SilentlyContinue
    }).AddArgument($server)
    $ps.RunspacePool = $runspacePool
    $jobs += @{ Pipe = $ps; Status = $ps.BeginInvoke() }
}
foreach ($job in $jobs) {
    $job.Pipe.EndInvoke($job.Status)
    $job.Pipe.Dispose()
}
$runspacePool.Close()
```

#### 数据序列化与传递

```powershell
# 导出对象为 XML（保留类型信息）
Get-Process | Export-Clixml -Path "processes.xml"
$imported = Import-Clixml -Path "processes.xml"

# JSON 序列化（API 交互、跨平台传递）
$data = @{
    hostname = $env:COMPUTERNAME
    username = $env:USERNAME
    processes = (Get-Process | Select-Object Name, Id, CPU | ConvertTo-Json)
}
$json = $data | ConvertTo-Json -Depth 5
$json | Out-File "report.json"

# CSV 导出（表格数据分析）
Get-EventLog -LogName Security -Newest 1000 |
    Select-Object TimeGenerated, EntryType, Source, EventID, Message |
    Export-Csv "security_events.csv" -NoTypeInformation -Encoding UTF8

# 压缩导出（减少传输大小）
Get-ChildItem C:\evidence | Compress-Archive -DestinationPath "evidence.zip" -CompressionLevel Optimal
```

---

### 5.3 安全对抗与日志分析

#### 脚本块日志分析

```powershell
# 提取和分析 PowerShell 脚本块日志
# Event ID 4104 = 脚本块日志（记录所有执行的 PS 代码）

$scriptLogs = Get-WinEvent -FilterHashtable @{
    LogName = 'Microsoft-Windows-PowerShell/Operational'
    ID = 4104
} -MaxEvents 1000 -EA SilentlyContinue

# 分析高危脚本模式
$suspiciousPatterns = @(
    "Invoke-Expression", "IEX", "iex",
    "DownloadString", "DownloadFile", "WebClient",
    "FromBase64String", "EncodedCommand",
    "Add-Type", "DllImport",
    "Start-Process", "CreateProcess",
    "Mimikatz", "Invoke-Mimikatz",
    "Invoke-Shellcode", "Invoke-ReflectivePEInjection",
    "Net.WebClient", "System.Net.Http",
    "certutil", "bitsadmin",
    "rundll32", "regsvr32",
    "WScript.Shell", "Shell.Application",
    "bypass", "-ep bypass", "-ExecutionPolicy Bypass"
)

$scriptLogs | ForEach-Object {
    $scriptText = $_.Properties[2].Value
    $matchedPatterns = $suspiciousPatterns | Where-Object { $scriptText -match $_ }
    if ($matchedPatterns) {
        [PSCustomObject]@{
            Time           = $_.TimeCreated
            ScriptLength   = $scriptText.Length
            MatchedPatterns = ($matchedPatterns | Select-Object -First 5) -join ", "
            ScriptPreview  = $scriptText.Substring(0, [Math]::Min(200, $scriptText.Length))
        }
    }
} | Sort-Object Time -Descending | Format-Table -AutoSize -Wrap
```

---

## 六、高阶冷门与底层协议

---

### 6.1 证书与 PKI 安全

```cmd
:: 查看本机所有证书存储
certutil -store

:: 查看特定存储
certutil -store My              # 个人证书
certutil -store Root            # 受信任的根证书
certutil -store CA              # 中间证书颁发机构
certutil -store TrustedPublisher # 受信任的发布者

:: 查看证书详细信息
certutil -dump <cert_file.cer>

:: 验证证书链
certutil -verify <cert_file.cer>

:: 查看 CRL（证书吊销列表）缓存
certutil -urlcache *

:: 导出证书
certutil -exportpfx My <thumbprint> output.pfx

:: 查看证书用途
certutil -v -store My <serial_number>
```

```powershell
# PowerShell 证书存储对象化遍历

# 列出所有证书
Get-ChildItem Cert:\ -Recurse | Where-Object { $_ -is [System.Security.Cryptography.X509Certificates.X509Certificate2] } |
    Select-Object @{N='Store';E={$_.PSParentPath -replace '.*\\Cert:\\', ''}},
                  Subject, Issuer, Thumbprint, NotBefore, NotAfter,
                  @{N='HasPrivateKey';E={$_.HasPrivateKey}} |
    Format-Table -AutoSize

# 查看即将过期的证书
Get-ChildItem Cert:\ -Recurse | Where-Object { 
    $_ -is [System.Security.Cryptography.X509Certificates.X509Certificate2] -and 
    $_.NotAfter -lt (Get-Date).AddDays(90) 
} | Select-Object Subject, NotAfter, @{N='Store';E={$_.PSParentPath}}

# 查看自签名证书（可能是攻击者部署的）
Get-ChildItem Cert:\ -Recurse | Where-Object { 
    $_ -is [System.Security.Cryptography.X509Certificates.X509Certificate2] -and 
    $_.Subject -eq $_.Issuer 
} | Select-Object Subject, Thumbprint, NotBefore, NotAfter,
    @{N='Store';E={$_.PSParentPath -replace '.*\\Cert:\\', ''}}

# 检查可疑的根证书（对比已知信任列表）
Get-ChildItem Cert:\LocalMachine\Root | 
    Where-Object { $_.Issuer -notmatch "Microsoft|DigiCert|GlobalSign|Comodo|Let's Encrypt|Amazon" } |
    Select-Object Subject, Issuer, Thumbprint, NotBefore, NotAfter
```

---

### 6.2 虚拟化与容器边界

```cmd
:: Hyper-V 虚拟机管理
hvc list                    # 列出所有虚拟机
hvc start <vm_name>
hvc stop <vm_name>

:: 虚拟机计算管理
vmcompute

:: WSL2 管理
wsl --list --verbose        # 列出所有 WSL 发行版
wsl --list --running        # 仅显示运行中的
wsl --status                # WSL 配置状态
wsl --shutdown              # 关闭所有 WSL 实例

:: WSL 配置
wslconfig /list
```

```powershell
# Hyper-V 详细管理（需 Hyper-V 模块）
Get-VM | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime, 
    @{N='NetworkAdapters';E={($_ | Get-VMNetworkAdapter).IPAddresses -join ", "}} |
    Format-Table -AutoSize

# 检测是否在虚拟机中运行
$systemInfo = Get-CimInstance Win32_ComputerSystem
$isVM = $systemInfo.Model -match "Virtual|VMware|VirtualBox|HVM|KVM" -or
        $systemInfo.Manufacturer -match "Microsoft|VMware|Xen|QEMU"
if ($isVM) { "Running in VM: $($systemInfo.Model)" } else { "Physical machine" }

# 检测沙箱指标
$sandboxIndicators = @(
    (Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory -lt 2GB,   # 内存 < 2GB
    (Get-CimInstance Win32_Processor).NumberOfCores -lt 2,                 # CPU < 2 核
    (Get-ChildItem C:\Users -Directory).Count -le 1,                        # 用户目录 <= 1
    $env:COMPUTERNAME -match "sandbox|test|malware|analysis",              # 主机名包含关键词
    (Get-Process).Count -lt 30                                              # 进程数 < 30
)
$sandboxScore = ($sandboxIndicators | Where-Object { $_ }).Count
"Sandbox probability: $sandboxScore/5 indicators matched"
```

---

### 6.3 内核、驱动与固件

```cmd
:: 驱动签名状态
driverquery /v /si
:: 关注 "Signed" 列，未签名驱动 = 潜在后门

:: UEFI 启动配置
bcdedit /enum {fwbootmgr}
bcdedit /enum {current}
bcdedit /enum all

:: 检查调试模式（如果启用，可能被利用）
bcdedit /enum {current} | findstr debug
bcdedit /enum {current} | findstr testsigning

:: ETW（Event Tracing for Windows）会话管理
logman query -ets               # 查看活跃的 ETW 会话
logman query "EventLog-Security" -ets  # 查看安全日志的 ETW 会话
logman start MyTrace -p "Microsoft-Windows-Security-Auditing" -ets
logman stop MyTrace -ets
```

```powershell
# 驱动安全分析
Get-CimInstance Win32_SystemDriver | 
    Where-Object { $_.State -eq "Running" } |
    Select-Object Name, DisplayName, Path, State, StartMode,
    @{N='Signed';E={
        $sig = Get-AuthenticodeSignature $_.Path -EA SilentlyContinue
        if ($sig) { $sig.Status } else { "N/A" }
    }} | Where-Object { $_.Signed -ne "Valid" -and $_.Signed -ne "N/A" }

# 检测测试签名模式的驱动
Get-CimInstance Win32_SystemDriver | 
    Where-Object { $_.State -eq "Running" } |
    ForEach-Object {
        $sig = Get-AuthenticodeSignature $_.Path -EA SilentlyContinue
        if ($sig -and $sig.SignerCertificate -and 
            $sig.SignerCertificate.Subject -match "Test|Self-Signed") {
            [PSCustomObject]@{
                Driver  = $_.Name
                Path    = $_.Path
                Subject = $sig.SignerCertificate.Subject
                Status  = $sig.Status
            }
        }
    }

# Secure Boot 状态
Confirm-SecureBootUEFI  # True = 启用, False = 禁用

# TPM 状态
Get-CimInstance -Namespace "root\cimv2\Security\MicrosoftTpm" -ClassName Win32_Tpm -EA SilentlyContinue |
    Select-Object IsActivated_InitialValue, IsEnabled_InitialValue, IsOwned_InitialValue, SpecVersion
```

---

### 6.4 RPC/DCOM/LDAP 底层

```cmd
:: RPC 配置查看（替代 Sysinternals rpcdump）
reg query "HKLM\SOFTWARE\Microsoft\Rpc" /s

:: RPC 过滤器（Windows 10+）
netsh rpc filter show filter
netsh rpc filter show rules

:: DCOM 配置
dcomcnfg   # GUI（系统自带）

:: LDAP 浏览器（系统自带 GUI 工具，需在域控或安装 RSAT 的机器上）
ldp.exe       # 原生 LDAP 浏览器
adsiedit.msc  # AD 编辑器
```

```powershell
# DCOM 应用枚举
Get-CimInstance Win32_DCOMApplicationSetting -EA SilentlyContinue |
    Select-Object AppID, Caption, AuthenticationLevel, RunAsUser |
    Where-Object { $_.AuthenticationLevel -ne $null }

# 使用 .NET 进行 LDAP 查询（无需 RSAT）
$ldapPath = "LDAP://DC=corp,DC=local"
$entry = New-Object System.DirectoryServices.DirectoryEntry($ldapPath)
$searcher = New-Object System.DirectoryServices.DirectorySearcher($entry)
$searcher.Filter = "(&(objectClass=user)(servicePrincipalName=*))"  # 查找有 SPN 的用户（Kerberoasting 目标）
$searcher.PropertiesToLoad.AddRange(@("samAccountName", "servicePrincipalName", "lastLogonTimestamp"))
$results = $searcher.FindAll()
foreach ($r in $results) {
    [PSCustomObject]@{
        User = $r.Properties["samAccountName"][0]
        SPN  = ($r.Properties["servicePrincipalName"] | Out-String).Trim()
    }
}

# 枚举域中的 GPO
$searcher.Filter = "(objectClass=groupPolicyContainer)"
$searcher.SearchScope = "Subtree"
$searcher.PropertiesToLoad.AddRange(@("displayName", "gPCFileSysPath", "whenCreated"))
$searcher.FindAll() | ForEach-Object {
    [PSCustomObject]@{
        Name = $_.Properties["displayName"][0]
        Path = $_.Properties["gPCFileSysPath"][0]
        Created = $_.Properties["whenCreated"][0]
    }
} | Format-Table -AutoSize
```

---

## 七、学习方法论与知识管理

---

### 7.1 场景驱动记忆法

不要按字母顺序背命令！而是按**实战场景**串联命令组合：

#### 场景 1：应急响应 — 入侵主机排查

```
┌─────────────────────────────────────────────────────────┐
│  1. systeminfo                    → 系统基线              │
│  2. whoami /all                   → 当前身份与特权         │
│  3. netstat -ano                  → 异常外联              │
│  4. tasklist /svc /v              → 可疑进程              │
│  5. schtasks /query /fo LIST /v   → 持久化计划任务        │
│  6. Get-WinEvent Security 4624/4625 → 登录分析           │
│  7. Get-WinEvent System 7045      → 恶意服务安装          │
│  8. icacls <suspicious_file>      → 文件权限分析          │
│  9. Get-FileHash <file>           → 文件完整性            │
│ 10. Get-MpThreatDetection         → Defender 检测历史     │
└─────────────────────────────────────────────────────────┘
```

#### 场景 2：内网渗透 — 信息收集阶段

```
┌─────────────────────────────────────────────────────────┐
│  1. ipconfig /all                 → 网络拓扑              │
│  2. arp -a                        → 同网段主机            │
│  3. route print                   → 路由信息              │
│  4. net user /domain              → 域用户枚举            │
│  5. net group "Domain Admins" /domain → 高价值目标        │
│  6. nltest /domain_trusts         → 信任关系              │
│  7. nltest /dclist:<domain>       → 域控制器定位          │
│  8. klist                         → Kerberos 票据         │
│  9. net view /domain              → 共享枚举              │
│ 10. netsh advfirewall show rule    → 防火墙规则            │
└─────────────────────────────────────────────────────────┘
```

#### 场景 3：持久化检测 — 后门排查

```
┌─────────────────────────────────────────────────────────┐
│  1. 注册表 Run 键          → reg query ... Run /s        │
│  2. 计划任务               → schtasks /query /v          │
│  3. 服务                   → sc query type= service      │
│  4. WMI 事件订阅           → Get-WmiObject __EventFilter │
│  5. IFEO 劫持              → reg query ... IFEO /s       │
│  6. 启动文件夹             → dir %APPDATA%\...\Startup   │
│  7. COM 劫持               → reg query ... InprocServer32│
│  8. DLL 劫持               → KnownDLLs 注册表检查        │
│  9. 浏览器扩展             → 各浏览器扩展目录检查         │
│ 10. 凭据持久化             → cmdkey /list                │
└─────────────────────────────────────────────────────────┘
```

---

### 7.2 底层原理优先

理解以下核心机制，命令只是"调用接口"：

| 机制 | 核心知识点 | 相关命令 |
|------|-----------|----------|
| **LSASS** | 存储所有登录会话的凭据（NTLM Hash、Kerberos TGT/TGS） | `tasklist`, `klist`, `wevtutil` |
| **SAM** | 本地账户密码哈希数据库，被 SYSTEM 独占 | `reg query`, `icacls`, `vssadmin` |
| **ETW** | Windows 内核级事件追踪框架，几乎所有安全日志的源头 | `logman`, `wevtutil`, `Get-WinEvent` |
| **注册表** | Windows 配置的核心存储，持久化/权限/策略都在此 | `reg query`, `regedit` |
| **WinRM** | Windows 原生远程管理协议（基于 HTTP/HTTPS + SOAP） | `winrs`, `Invoke-Command` |
| **Kerberos** | AD 域认证协议，票据机制是横向移动的核心 | `klist`, `nltest`, `setspn` |
| **NTFS ACL** | 安全描述符（DACL/SACL），权限控制的基础 | `icacls`, `Get-Acl` |
| **Token** | 访问令牌包含 SID、特权、完整性级别 | `whoami`, `runas` |

---

### 7.3 构建个人速查库（推荐结构）

```
📁 Cybersecurity-Commands/
├── 📁 01-Enumeration/
│   ├── system_info.md          # systeminfo, hostname, ver
│   ├── user_group.md           # whoami, net user, net group
│   ├── ad_enum.md              # nltest, dsquery, klist
│   └── process_service.md     # tasklist, sc, schtasks, wmic
├── 📁 02-Network/
│   ├── config.md               # ipconfig, arp, route, nslookup
│   ├── connection.md           # netstat, Test-NetConnection
│   ├── firewall.md             # netsh advfirewall
│   └── smb_share.md           # net view, net use, Get-SmbShare
├── 📁 03-FileSystem/
│   ├── search.md               # dir, Get-ChildItem, findstr
│   ├── permission.md           # icacls, Get-Acl
│   ├── hash.md                 # certutil, Get-FileHash
│   └── forensics.md           # wevtutil, reg query, timeline
├── 📁 04-Defense/
│   ├── ioc_detection.md       # 进程分析、Base64 解码
│   ├── persistence.md         # WMI、IFEO、注册表
│   ├── defender.md            # Get-Mp*, AppLocker
│   └── lateral_movement.md   # 登录分析、远程执行
├── 📁 05-PowerShell/
│   ├── dotnet.md              # .NET 交互
│   ├── remote.md              # PSSession, Invoke-Command
│   └── logging.md            # 脚本块日志分析
├── 📁 06-Advanced/
│   ├── pki.md                 # certutil, Cert:\
│   ├── virtualization.md      # Hyper-V, WSL
│   ├── kernel.md              # driverquery, bcdedit, ETW
│   └── protocol.md           # RPC, DCOM, LDAP
└── 📁 Scenarios/              # 场景化命令流
    ├── incident_response.md
    ├── internal_pentest.md
    ├── persistence_check.md
    └── compliance_audit.md
```

每个文件包含：
- **命令语法**（完整参数说明）
- **输出示例**（真实截图/文本）
- **安全场景**（攻击/防御/取证角度）
- **常见坑点**（版本差异、权限要求、常见错误）
- **关联命令**（同一场景下的组合用法）

---

### 7.4 隔离靶场搭建指南

```
┌──────────────────────────────────────────────────────────────┐
│  推荐靶场拓扑（使用 VMware/VirtualBox）                       │
│                                                              │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐    │
│  │  DC (域控)   │     │  Server     │     │  Attacker   │    │
│  │  Win Server │◄───►│  Win Server │◄───►│  Win 10/11  │    │
│  │  2022       │     │  2019       │     │  + Kali     │    │
│  │  - AD DS    │     │  - IIS      │     │             │    │
│  │  - DNS      │     │  - MSSQL    │     │             │    │
│  │  - DHCP     │     │  - 共享     │     │             │    │
│  └─────────────┘     └─────────────┘     └─────────────┘    │
│         │                    │                    │          │
│         └────────────────────┴────────────────────┘          │
│                    NAT/Host-Only 网络                         │
│                                                              │
│  ┌─────────────┐                                             │
│  │  Log Server │     日志集中收集                              │
│  │  Win 10 +   │     - WEF (Windows Event Forwarding)        │
│  │  Sysmon     │     - Sysmon 配置                           │
│  └─────────────┘                                             │
└──────────────────────────────────────────────────────────────┘
```

**练习方法**：
1. **正向练习**：在 Attacker 机上执行命令，观察效果
2. **逆向检测**：在 Log Server 上查看对应的日志/痕迹
3. **防御验证**：配置防御策略后，验证攻击命令是否被阻止
4. **版本对比**：在不同 Windows 版本上测试同一命令的差异

---

### 7.5 版本差异意识

| 功能 | Win10 | Win11 21H2 | Win11 24H2 | Server 2022 | Server 2025 |
|------|-------|-----------|-----------|------------|------------|
| PowerShell 默认版本 | 5.1 | 5.1 (pwsh 7 可装) | 5.1 + pwsh 7 | 5.1 | 5.1 + pwsh 7 |
| WMIC | ✅ | ✅ (弃用警告) | ❌ 已移除 | ✅ (弃用警告) | ❌ 已移除 |
| Windows Terminal | 可装 | 默认 | 默认 | 可装 | 默认 |
| WinGet | 可装 | ✅ | ✅ | ❌ | 可装 |
| Local Security Authority | ❌ | ✅ | ✅ 增强 | ❌ | ✅ |
| Pluton TPM | ❌ | 可选 | 可选 | ❌ | 可选 |
| Recall | ❌ | ❌ | ✅ (争议) | ❌ | ❌ |

---

### 7.6 官方文档锚定

| 资源 | URL | 用途 |
|------|-----|------|
| **Microsoft Learn - PowerShell** | learn.microsoft.com/powershell | Cmdlet 官方文档 |
| **Microsoft Learn - Windows Commands** | learn.microsoft.com/windows-server/administration/windows-commands | CMD 命令参考 |
| **SS64** | ss64.com/nt + ss64.com/ps | 第三方速查（含示例） |
| **MITRE ATT&CK** | attack.mitre.org | 攻击技术与对应命令映射 |
| **LOLBAS** | lolbas-project.github.io | Living Off The Land 二进制与脚本 |
| **Atomic Red Team** | github.com/redcanaryco/atomic-red-team | 攻击复现测试用例 |
| **Windows Internals** | 书籍 | 内核机制深度理解 |

---

## 📋 命令总计数统计

| 类别 | CMD 命令 | PowerShell Cmdlet | 合计 |
|------|---------|-------------------|------|
| 系统枚举 | ~35 | ~25 | ~60 |
| 网络侦察 | ~25 | ~20 | ~45 |
| 文件系统 | ~20 | ~30 | ~50 |
| 安全防御 | ~15 | ~40 | ~55 |
| PowerShell 专属 | — | ~35 | ~35 |
| 高阶冷门 | ~20 | ~25 | ~45 |
| **总计** | **~115** | **~175** | **~290** |

> 💡 **核心建议**：不需要记住所有 290 个命令，但要熟练掌握每个场景下的 **Top 5 命令**，其余的通过速查库和 `/?` / `Get-Help` 随时查阅。

---

*本指南涵盖 Windows 原生命令体系的全部核心内容。建议配合实际靶场环境逐步实操，每掌握一个场景就更新你的个人速查库。安全之路，始于足下！* 🚀