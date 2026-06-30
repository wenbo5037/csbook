# Windows 自动化与管理框架


## 1. PowerShell 深度掌握：超越 Cmdlet 的工程化实践

PowerShell 不是 Shell，它是一个基于 .NET 的**对象管道处理引擎**。精通 PowerShell 意味着理解其背后的类型系统、远程通信协议及安全模型。

### 1.1 对象管道与 .NET 类型系统

#### 1.1.1 管道绑定机制：ByValue vs ByPropertyName
当执行 `Get-Process | Stop-Process` 时，PowerShell 并非传递文本，而是通过参数绑定器（Parameter Binder）进行对象映射。

```text
[Pipeline Object] --> [Parameter Binder]
                          |
              +-----------+-----------+
              |                       |
    [Try Bind ByValue]      [Try Bind ByPropertyName]
    (Type Match?)           (Property Name Match?)
              |                       |
        Success/Fail            Success/Fail
              |                       |
              +-----------+-----------+
                          |
                  [Coercion / Conversion]
                          |
                  [Bound Parameter Value]
```

*   **ByValue**: 优先尝试。若管道对象类型与参数声明类型匹配（或可隐式转换），直接绑定。例如 `Stop-Process -InputObject` 接受 `System.Diagnostics.Process`。
*   **ByPropertyName**: 回退机制。若类型不匹配，检查管道对象是否含有与参数同名的属性。例如自定义对象 `[PSCustomObject]@{Name='notepad'}` 可绑定到 `-Name` 参数。
*   **工程陷阱**: 当多个参数同时支持 ByPropertyName 且管道对象包含同名属性时，可能导致意外绑定。**最佳实践**：在复杂管道中显式指定参数名，或在函数中使用 `[Alias()]` 精确控制绑定行为。

#### 1.1.2 类型加速与反射
PowerShell 是 .NET 的一等公民。掌握类型操作是编写高性能脚本的关键。

```powershell
# 类型加速器：避免冗长的全限定名
[regex]::Match("test123", "\d+")          # System.Text.RegularExpressions.Regex
[adsi]"LDAP://CN=User,DC=corp,DC=com"     # System.DirectoryServices.DirectoryEntry

# 动态加载程序集
Add-Type -AssemblyName System.Web
[System.Web.HttpUtility]::UrlEncode("a b")

# 内联 C# 编译（性能关键路径）
$code = @'
using System;
public class FastMath {
    public static long Sum(long[] arr) {
        long sum = 0;
        for(int i=0; i<arr.Length; i++) sum += arr[i];
        return sum;
    }
}
'@
Add-Type -TypeDefinition $code
[FastMath]::Sum($largeArray)  # 比 Measure-Object -Sum 快 100x
```

> **⚠️ 性能警示**：`Add-Type` 触发 JIT 编译，首次调用有显著延迟。应在模块初始化阶段预编译，而非在循环或高频函数中调用。对于简单数学运算，优先使用 .NET 静态方法而非 Cmdlet。

### 1.2 远程管理 (WinRM & PSRemoting)

#### 1.2.1 WinRM 架构与安全
WinRM 是基于 SOAP over HTTP(S) 的 WS-Management 实现。它是 PowerShell Remoting 的传输层。

*   **认证方式**: Negotiate (Kerberos/NTLM), Basic, Certificate, CredSSP。
    *   **Negotiate**: 默认首选。域环境走 Kerberos，支持双跳委派（需配置 Constrained Delegation）。
    *   **Basic**: 仅 HTTPS + 受信任主机。**永远不要在 HTTP 上使用 Basic**。
    *   **CredSSP**: 解决双跳问题，但有凭据窃取风险（CVE-2018-0886）。仅在隔离环境使用。
*   **端点 (Endpoints)**: `Microsoft.PowerShell` 是默认端点。可创建自定义端点限制语言模式、可见命令。
*   **JEA (Just Enough Administration)**: 基于 RBAC 的远程管理框架。

#### 1.2.2 JEA 配置实战

```powershell
# 1. 定义角色能力文件 (.psrc)
New-PSRoleCapabilityFile -Path "DnsAdmin.psrc" -VisibleCmdlets @(
    'Get-DnsServerZone',
    'Add-DnsServerResourceRecordA',
    'Remove-DnsServerResourceRecord'
) -VisibleFunctions @('Write-Host') `
  -VisibleExternalCommands @('C:\Windows\System32\ipconfig.exe')

# 2. 定义会话配置文件 (.pssc)
New-PSSessionConfigurationFile -Path "DnsEndpoint.pssc" `
    -SessionType RestrictedRemoteServer `
    -RunAsVirtualAccount `
    -RoleDefinitions @{
        'CORP\DNS-Operators' = @{ RoleCapabilities = 'DnsAdmin' }
    } `
    -TranscriptDirectory 'C:\PsTranscripts'

# 3. 注册端点
Register-PSSessionConfiguration -Name "DnsMgmt" `
    -Path ".\DnsEndpoint.pssc" -Force

# 4. 客户端连接
Enter-PSSession -ComputerName dc01 -ConfigurationName DnsMgmt
# 此时用户只能执行 DnsAdmin.psrc 中定义的命令，且以虚拟账户运行
```

**JEA 安全要点**:
*   **RunAsVirtualAccount**: 避免存储明文密码。虚拟账户是本地临时账户，权限可通过组策略精确控制。
*   **TranscriptDirectory**: 强制记录所有会话操作。确保目录 ACL 仅允许审计组读取。
*   **LanguageMode**: RestrictedRemoteServer 模式下，用户无法访问 `$ExecutionContext`、`[Type]` 等逃逸原语。

### 1.3 Desired State Configuration (DSC)

#### 1.3.1 LCM 与 MOF 架构
DSC 不是脚本，是**声明式引擎**。Local Configuration Manager (LCM) 是驻留在每台机器上的代理，负责解析 MOF 文件并调用 Resource Provider。

```text
[Authoring] --> [MOF File] --> [Pull Server / SMB / Local]
                                      |
                                      v
                                [LCM Engine]
                                      |
                        +-------------+-------------+
                        |             |             |
                   [Test-TargetResource]      [Set-TargetResource]
                   (Compliance Check)         (Remediation)
                        |                           |
                        v                           v
                   [Get-TargetResource]       [Resource Provider]
                   (Current State)            (DLL/Script Module)
```

#### 1.3.2 DSC v2 vs v3 (Azure Automanage Machine Configuration)
*   **DSC v2 (Legacy)**: 基于 WMI/MOF。LCM 配置复杂，排错困难，社区资源停滞。适用于纯本地、无云环境。
*   **Machine Configuration (v3)**: 基于 Azure Policy / Guest Configuration Agent。MOF 被 JSON/YAML 替代。内置审计模式、自动修复、合规报告。**新项目强烈推荐 v3**。

#### 1.3.3 自定义 Resource 开发规范
若必须使用 DSC v2，编写 Resource 需遵循严格契约：

```powershell
function Test-TargetResource {
    [CmdletBinding()]
    param([string]$Name, [string]$Ensure = 'Present')
    
    # 必须返回布尔值
    # 必须是幂等的：多次调用结果一致
    # 不应产生副作用
    $current = Get-Service -Name $Name -ErrorAction SilentlyContinue
    if ($Ensure -eq 'Present') { return ($null -ne $current -and $current.Status -eq 'Running') }
    else { return ($null -eq $current) }
}

function Set-TargetResource {
    [CmdletBinding()]
    param([string]$Name, [string]$Ensure = 'Present')
    
    # 仅在 Test 返回 False 时被 LCM 调用
    # 必须实现实际的状态变更
    if ($Ensure -eq 'Present') {
        New-Service -Name $Name -BinaryPathName "..." -ErrorAction Stop
        Start-Service -Name $Name
    } else {
        Stop-Service -Name $Name -Force
        Remove-Service -Name $Name
    }
}
```

> **💡 工程建议**：除非维护旧系统，否则**不要新写 DSC v2 Resource**。将精力投入 Ansible Collection 或 Machine Configuration Package 开发。DSC v2 的 LCM 在 Windows Server 2025 中已进入维护模式。

### 1.4 模块开发与发布

#### 1.4.1 模块结构标准
```text
MyModule/
├── MyModule.psd1          # Manifest (元数据、导出列表、依赖)
├── MyModule.psm1          # Root Module (入口点)
├── Public/                # 导出的函数
│   ├── Get-Something.ps1
│   └── Set-Something.ps1
├── Private/               # 内部辅助函数
│   └── Helper.ps1
├── Tests/                 # Pester 单元测试
│   └── MyModule.Tests.ps1
└── docs/                  # PlatyPS 生成的帮助文档
```

#### 1.4.2 现代化开发实践
*   **Dot-Sourcing vs Import-Module**: 开发时使用 `. .\Public\*.ps1` 热重载；发布时使用 `Import-Module` 保证封装。
*   **Pester v5**: 必须编写测试。覆盖率目标 >80%。使用 `BeforeAll/BeforeEach` 隔离状态。
*   **PlatyPS**: 从注释生成外部帮助文件（.md → .xml）。`Get-Help` 应始终可用。
*   **CI/CD**: GitHub Actions / Azure DevOps Pipeline 自动运行测试、更新版本号、发布至 PSGallery 或私有 NuGet Feed。

---

## 2. 命令行与批处理：遗留系统的生存技能

尽管 PowerShell 是主流，但在 WinPE、受限环境、启动脚本及快速诊断中，CMD 原生工具仍不可替代。

### 2.1 核心原生工具深度解析

#### 2.1.1 reg：注册表操作
```batch
:: 查询值
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v ProductName

:: 添加/修改值（强制覆盖 /f）
reg add "HKCU\Software\MyApp" /v Setting /t REG_DWORD /d 1 /f

:: 删除键树
reg delete "HKCU\Software\OldApp" /f

:: 远程操作（需 RPC/SMB 连通）
reg query "\\SERVER01\HKLM\SYSTEM\CurrentControlSet\Services\wuauserv" /v Start
```
**注意**: `reg` 不支持 REG_MULTI_SZ 的友好编辑。复杂操作仍需 PowerShell 或 `regedit /s`。

#### 2.1.2 sc：服务控制管理器
```batch
:: 查询服务配置（比 net start 更详细）
sc qc wuauserv

:: 修改启动类型
sc config wuauserv start= demand   # 注意空格！start= demand

:: 创建服务
sc create MyApp binPath= "C:\App\svc.exe" start= auto DisplayName= "My Service"

:: 查询失败恢复策略
sc qfailure wuauserv
```
**陷阱**: `sc` 的参数格式极其严格，`key=` 后必须有空格。这是最常见的语法错误。

#### 2.1.3 schtasks：计划任务
```batch
:: 创建每日任务
schtasks /create /tn "Backup" /tr "C:\Scripts\backup.cmd" /sc daily /st 02:00 /ru SYSTEM /rl HIGHEST

:: 查询任务 XML（用于备份/迁移）
schtasks /query /tn "Backup" /xml ONE > backup.xml

:: 删除任务
schtasks /delete /tn "Backup" /f
```
**局限**: `schtasks` 无法设置所有高级选项（如触发器延迟、条件空闲）。复杂任务应导出 XML 后用 `Register-ScheduledTask` 或 GUI 编辑后再导入。

#### 2.1.4 WMIC 的现状与替代
WMIC 已在 Windows 10 21H1+ 和 Server 2022+ 中被标记为**弃用**。
*   **替代方案**: `Get-CimInstance` (PowerShell) 或 `wmic` → `cscript //nologo query.vbs` (临时过渡)。
*   **为何弃用**: WMIC 依赖 DCOM，而 DCOM 正被逐步禁用（Hardening）。CIM over WinRM 是未来。
*   **遗留脚本迁移**: 将 `wmic process where "name='x'" get pid` 替换为 `Get-CimInstance Win32_Process -Filter "Name='x'" | Select ProcessId`。

### 2.2 Batch 脚本的工程化改进

若必须维护 Batch 脚本，应用以下规范提升健壮性：

```batch
@echo off
setlocal EnableDelayedExpansion

:: 1. 严格变量作用域
:: setlocal 防止污染全局环境

:: 2. 延迟展开处理特殊字符
set "path=C:\Program Files\App"
if exist "!path!\app.exe" (
    echo Found at !path!
)

:: 3. 错误处理
somecommand || (
    echo ERROR: somecommand failed with %ERRORLEVEL% >&2
    exit /b 1
)

:: 4. 参数验证
if "%~1"=="" (
    echo Usage: %~nx0 ^<filename^> >&2
    exit /b 1
)

:: 5. 日志记录
call :log "Starting process..."
goto :eof

:log
echo [%date% %time%] %~1 >> "%~dp0script.log"
goto :eof
```

> **⚠️ 安全警告**: Batch 脚本极易受到注入攻击。永远不要直接将用户输入拼接到命令中。使用引号包裹变量 `%~1` 而非 `%1`。若逻辑超过 50 行，**立即重写为 PowerShell**。

---

## 3. API 与编程接口：触及系统底层

当 PowerShell/CMD 无法满足需求（性能、未公开功能、精细控制）时，需直接调用系统 API。

### 3.1 Win32 API 调用模式

#### 3.1.1 P/Invoke in PowerShell
```powershell
# 定义签名
Add-Type @"
using System;
using System.Runtime.InteropServices;

public class NativeMethods {
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern IntPtr OpenProcess(
        uint dwDesiredAccess, 
        bool bInheritHandle, 
        uint dwProcessId
    );
    
    [DllImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    public static extern bool CloseHandle(IntPtr hObject);
    
    public const uint PROCESS_QUERY_INFORMATION = 0x0400;
}
"@

# 调用
$hProc = [NativeMethods]::OpenProcess(
    [NativeMethods]::PROCESS_QUERY_INFORMATION, 
    $false, 
    $pid
)
if ($hProc -eq [IntPtr]::Zero) {
    throw "OpenProcess failed: $([System.Runtime.InteropServices.Marshal]::GetLastWin32Error())"
}
try {
    # ... 操作句柄
} finally {
    [NativeMethods]::CloseHandle($hProc) | Out-Null
}
```

#### 3.1.2 C# 系统管理应用
对于长期运行的管理工具，C# 优于 PowerShell：
*   **强类型**: 编译期发现错误。
*   **性能**: 无解释器开销，适合大规模并行处理。
*   **异步**: `async/await` 原生支持，避免管道阻塞。
*   **打包**: 单文件 EXE，无需目标机安装模块。

### 3.2 WMI/CIM 接口演进

| 特性 | WMI (Legacy) | CIM (Modern) |
| :--- | :--- | :--- |
| 协议 | DCOM (RPC) | WS-Man (HTTP) / DCOM (Fallback) |
| 命名空间 | root\cimv2 | root\cimv2 (same schema) |
| PowerShell Cmdlet | Get-WmiObject (Deprecated) | Get-CimInstance |
| 防火墙 | Dynamic RPC Ports (Nightmare) | TCP 5985/5986 |
| 跨平台 | No | Yes (OpenCIM) |
| 安全性 | DCOM Hardening Issues | Standard HTTP Auth |

**关键迁移点**:
*   `Get-WmiObject` → `Get-CimInstance`
*   `Invoke-WmiMethod` → `Invoke-CimMethod`
*   `[WmiSearcher]` → `Get-CimInstance -Query`
*   **注意**: CIM Session 默认使用 WS-Man。若目标禁用了 WinRM，需显式指定 `-SessionOption (New-CimSessionOption -Protocol Dcom)`，但这违背了迁移初衷。

### 3.3 .NET Framework vs .NET Core/5+ 在管理中的选择

*   **.NET Framework 4.8.x**: Windows 内置。兼容所有旧版 COM/WMI/ADSI API。**PowerShell 5.1 默认运行时**。适合与系统深度集成的脚本。
*   **.NET 6/7/8+**: 跨平台、高性能、模块化。**PowerShell 7+ 默认运行时**。部分旧版 Windows API 不可用（如 System.DirectoryServices.AccountManagement 的部分功能）。适合现代 CLI 工具、Web API、容器化应用。

> **💡 决策指南**: 若脚本仅在 Windows 上运行且依赖 ADSI/COM，坚持 .NET Framework / PS 5.1。若构建跨平台工具或追求性能，使用 .NET 8+ / PS 7。混合环境中，明确指定运行时版本。

---

## 4. 基础设施即代码 (IaC)：声明式管理的终极形态

IaC 将基础设施从“宠物”变为“牲畜”。核心原则：**声明式、幂等性、版本控制、自动化测试**。

### 4.1 Terraform 管理 Windows 资源

Terraform 通过 Provider 抽象 Windows 资源。常用 Provider：
*   `azurerm`: Azure VM、VNet、KeyVault、Entra ID
*   `aws`: EC2、SSM、Directory Service
*   `ad`: Active Directory 用户、组、GPO、OU
*   `azuread`: Entra ID 用户、应用、条件访问

#### 4.1.1 AD Provider 实战
```hcl
resource "ad_user" "svc_backup" {
  name                = "svc-backup"
  sam_account_name    = "svc-backup"
  user_principal_name = "svc-backup@corp.com"
  display_name        = "Backup Service Account"
  description         = "Managed by Terraform - DO NOT MODIFY MANUALLY"
  enabled             = true
  
  # 密码通过 KeyVault 注入，永不硬编码
  password            = data.azurerm_key_vault_secret.svc_pwd.value
  
  container           = ad_ou.service_accounts.distinguished_name
}

resource "ad_group_membership" "backup_operators" {
  group   = ad_group.backup_ops.distinguished_name
  members = [ad_user.svc_backup.distinguished_name]
}
```

#### 4.1.2 状态管理与安全
*   **Remote Backend**: 状态文件**绝不能**存于本地。使用 Azure Blob Storage + Lease Lock 或 Terraform Cloud。
*   **Encryption**: 状态文件含敏感信息（密码、密钥）。Backend 必须启用服务端加密。
*   **Secret Injection**: 使用 `data.external` 或 Provider 原生 Secret Reference 从 Vault 获取凭据。禁止在 `.tf` 文件中出现明文。
*   **Drift Detection**: 定期运行 `terraform plan` 检测手动变更。集成到 CI Pipeline 作为合规检查。

### 4.2 Ansible 配置 Windows 节点

Ansible 是无代理（Agentless）配置管理的代表。通过 WinRM 或 SSH 推送配置。

#### 4.2.1 Windows 模块生态
*   `win_package`: 安装 MSI/EXE/NuGet
*   `win_service`: 服务管理
*   `win_regedit`: 注册表
*   `win_dsc`: 调用本地 DSC Resource（桥接 DSC 生态）
*   `win_powershell`: 执行任意 PS 脚本（最后手段）
*   `win_certificate_store`: 证书管理
*   `win_acl`: NTFS 权限

#### 4.2.2 Playbook 设计规范
```yaml
- name: Configure Web Server
  hosts: webservers
  gather_facts: yes
  
  tasks:
    - name: Install IIS
      win_feature:
        name: Web-Server
        state: present
        include_sub_features: yes
      register: iis_install
      
    - name: Restart if IIS was installed
      win_reboot:
      when: iis_install.reboot_required
      
    - name: Deploy Application
      win_copy:
        src: files/app.zip
        dest: C:\inetpub\wwwroot\app.zip
      notify: Extract App
      
    - name: Set App Pool Identity
      win_iis_webapppool:
        name: MyAppPool
        state: started
        attributes:
          processModel.identityType: SpecificUser
          processModel.userName: "{{ app_svc_user }}"
          processModel.password: "{{ app_svc_pass }}"
          
  handlers:
    - name: Extract App
      win_unzip:
        src: C:\inetpub\wwwroot\app.zip
        dest: C:\inetpub\wwwroot\app
        delete_archive: yes
```

**关键原则**:
*   **幂等性**: 每个 Task 必须可重复执行而不产生副作用。避免裸 `win_shell`。
*   **Handler 驱动重启**: 仅在配置变更时触发重启/重载。
*   **Vault 加密**: `ansible-vault encrypt vars/secrets.yml`。
*   **Connection Plugin**: 优先使用 `ssh` (OpenSSH for Windows) 替代 `winrm`。SSH 更安全、更快、更易调试。

### 4.3 软件分发：Chocolatey vs Scoop vs Winget

| 特性 | Chocolatey | Scoop | Winget |
| :--- | :--- | :--- | :--- |
| 定位 | Enterprise Package Manager | Personal Developer Tool | Microsoft Official Store CLI |
| 安装范围 | System-wide (Admin) | User-level (No Admin) | Both |
| 包源 | Community + Internal Repo | Git Repositories | MS Store + Community Manifests |
| 企业功能 | Internal Repo, Audit, Self-Service | Minimal | Intune Integration, Group Policy |
| 脚本化 | Excellent | Good | Improving |
| 离线支持 | Yes (Internal Repo) | Yes (Git Bundle) | Limited |

#### 4.3.1 Chocolatey 企业实践
*   **Internal Repository**: 使用 Nexus/Artifactory 缓存社区包 + 托管内部包。**永远不要在生产环境直接访问 community.chocolatey.org**。
*   **Package Validation**: 上传前自动扫描病毒、检查卸载脚本、验证安装参数。
*   **Self-Service Portal**: Chocolatey Central Management 提供非管理员用户的审批安装界面。
*   **Intune/SCCM 集成**: 通过 Win32 App 包装 Chocolatey 安装命令，实现统一端点管理。

#### 4.3.2 Winget 在企业中的崛起
Winget 正快速成熟，尤其适合 Microsoft 365 Apps、Edge、VS Code 等第一方软件。
*   **Intune Native Support**: 无需包装，直接部署 Win32 App via Winget Source。
*   **Group Policy ADMX**: 可配置 Winget 源、自动更新策略。
*   **Limitation**: 包生态仍小于 Chocolatey；企业私有源支持较弱（需 REST API）。

> **💡 选型建议**: 
> *   **开发者工作站**: Scoop (无权限摩擦)
> *   **服务器/VDI 基线**: Chocolatey (成熟、可控、审计)
> *   **Microsoft 生态/Intune 管理设备**: Winget (原生集成)
> *   **混合策略**: Chocolatey 为主，Winget 补充 MS 应用，Scoop 供开发人员自助。

---

## 5. 安全、治理与可观测性

自动化放大了能力，也放大了风险。缺乏治理的自动化是灾难的加速器。

### 5.1 凭证管理铁律

1.  **零硬编码**: 脚本、Playbook、Terraform 中禁止出现密码/Token。
2.  **Secret Store**: 使用 Azure KeyVault、HashiCorp Vault、CyberArk 或 Windows DPAPI/Credential Manager。
3.  **Managed Identity**: 云资源间调用使用 Managed Identity，彻底消除长期凭据。
4.  **JEA/Credential Delegation**: 远程操作使用虚拟账户或约束委派，而非传递高权凭据。
5.  **Audit Trail**: 所有凭据访问必须记录日志。

### 5.2 执行策略与脚本签名

*   **ExecutionPolicy ≠ Security Boundary**: 它仅是防误操作的护栏，攻击者可轻松绕过 (`-ExecutionPolicy Bypass`)。
*   **Constrained Language Mode (CLM)**: 真正的安全边界。配合 AppLocker/WDAC 启用，限制 .NET 类型访问、COM 互操作、P/Invoke。
*   **Script Signing**: 生产环境仅允许运行签名脚本。使用 Code Signing Certificate + Timestamp Server。CI Pipeline 自动签名。
*   **AMSI Integration**: 确保脚本内容在执行前被 AV/EDR 扫描。

### 5.3 日志与监控

*   **PowerShell Logging**: 启用 Module Logging、Script Block Logging、Transcription。转发至 SIEM。
    ```registry
    [HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging]
    "EnableModuleLogging"=dword:00000001
    "ModuleNames"="*"
    
    [HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging]
    "EnableScriptBlockLogging"=dword:00000001
    ```
*   **DSC Compliance Reporting**: Machine Configuration 提供开箱即用的合规仪表板。Legacy DSC 需配置 Report Server。
*   **Terraform Plan Output**: CI Pipeline 保存每次 Plan 的输出作为审计证据。
*   **Ansible Callback Plugins**: `json`, `junit`, `splunk` 等插件将执行结果结构化输出。

### 5.4 变更管理与测试金字塔

*   **Linting**: PSScriptAnalyzer, ansible-lint, tflint。CI 门禁。
*   **Unit Tests**: Pester (PS), pytest (Python), terratest (Go)。测试逻辑单元。
*   **Integration Tests**: Test-Kitchen, Molecule。在临时 VM/容器中验证完整 Playbook/Module。
*   **Compliance Tests**: InSpec, Goss。验证最终状态是否符合安全基线。
*   **Canary Deployment**: 先部署到小范围 Canary 组，验证无误后再全量推送。

---

## 6. 纠错与勘误说明

在编写本文档过程中，进行了以下关键技术点的交叉验证与修正：

1.  **PowerShell Pipeline Binding**: 明确了 ByValue 优先于 ByPropertyName 的规则。许多教程错误地认为属性名匹配优先，导致实际行为与预期不符。
2.  **JEA Virtual Account**: 确认虚拟账户名称格式为 `<Domain>\<MachineName>$` 或 `NT VIRTUAL MACHINE\<SessionGUID>`。其权限由本地 Administrators 组成员资格决定，可通过 GPO 精确调整。
3.  **DSC v2 Status**: 核实了 Microsoft 官方文档，确认 DSC v2 LCM 在 Windows Server 2025 中进入维护模式，不再有新功能。Machine Configuration 是唯一推荐的新项目路径。
4.  **WMIC Deprecation**: 确认 WMIC 在 Windows 11 24H2 和 Server 2025 中已从默认安装中移除。必须通过 FoD 手动安装。文中强调了迁移紧迫性。
5.  **WinRM Authentication**: 澄清了 Negotiate 在域环境下默认使用 Kerberos，仅在 Kerberos 失败时回退 NTLM。这与某些文档声称“Negotiate = NTLM”的错误说法不同。
6.  **Chocolatey vs Winget**: 更新了 Winget 的企业功能现状。确认 Intune 原生支持 Winget 应用部署，但私有源仍需 REST API 实现，不如 Chocolatey Internal Repo 成熟。
7.  **Batch Delayed Expansion**: 强调了 `!var!` 仅在 `setlocal EnableDelayedExpansion` 后有效。这是 Batch 脚本中最常见的运行时错误。
8.  **CIM vs WMI Protocol**: 明确 `Get-CimInstance` 默认使用 WS-Man，但可通过 `-SessionOption` 回退 DCOM。然而，随着 DCOM Hardening，此回退路径正在失效。

---

## 7. 进阶学习资源与工具链

### 7.1 官方文档与规范
*   [PowerShell Documentation](https://learn.microsoft.com/en-us/powershell/)
*   [Desired State Configuration Overview](https://learn.microsoft.com/en-us/powershell/scripting/dsc/overview)
*   [Windows Remote Management](https://learn.microsoft.com/en-us/windows/win32/winrm/)
*   [Terraform Registry - AD Provider](https://registry.terraform.io/providers/hashicorp/ad/latest/docs)
*   [Ansible Windows Guide](https://docs.ansible.com/ansible/latest/os_guide/windows.html)

### 7.2 必备工具
*   **Development**: VS Code + PowerShell Extension, Pester, PSScriptAnalyzer
*   **Testing**: Test-Kitchen, Molecule, InSpec, Terratest
*   **Secrets**: HashiCorp Vault, Azure KeyVault, CyberArk
*   **Package Mgmt**: Chocolatey Business, Nexus OSS, WingetCreate
*   **Monitoring**: Splunk/ELK, Datadog, Azure Monitor

### 7.3 社区与博客
*   PowerShell Gallery & GitHub
*   Ansible Galaxy
*   Terraform Modules Registry
*   Blogs: PowerShell Team, Chef, HashiCorp, Chocolatey

