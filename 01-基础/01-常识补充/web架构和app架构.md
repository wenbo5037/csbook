# web架构和app架

## 一、URL访问控制：物理路径与路由映射的绝对隔离

### 1. 物理路径访问模式（传统ASP/PHP）

**技术机理：**
- Web服务器（IIS/Apache）直接将URL路径映射到文件系统物理路径
- 例如：访问 `http://example.com/post/1.txt` → IIS读取 `C:\wwwroot\post\1.txt` 并返回内容
- **安全利用**：上传WebShell到`/upload/shell.asp`，访问对应URL即可直接触发执行

**权限验证机制：**
- 每个请求都会经过NTFS权限检查（IIS工作进程身份）
- 工作进程默认以`IIS APPPOOL\DefaultAppPool`身份运行
- 该身份对文件系统的权限由ACL（访问控制列表）决定

### 2. 路由映射访问模式（Java Spring / Flask / ThinkPHP）

**技术机理：**
- URL由框架的`路由表`解析，不直接对应物理文件
- Spring示例：`@RequestMapping("/user/login")` 映射到 `UserController.login()` 方法
- 物理文件存放在 `/WEB-INF/classes/` 目录，但该目录受容器保护，无法通过URL直接访问

**实战影响（核心痛点）：**
- **上传后访问不了**：上传`/uploads/shell.jsp`后，访问 `http://target/uploads/shell.jsp` 返回404
  - 原因：Tomcat的`web.xml`中配置了`<url-pattern>/api/*</url-pattern>`，所有请求由`DispatcherServlet`接管，静态资源需通过特定映射（如`/resources/**`）才能访问
  - 若`/uploads/`未在静态资源映射中声明，则请求由路由引擎处理，找不到对应路由即返回404
- **信息收集转向**：无法通过路径猜测访问文件，必须通过以下方式定位：
  1. 爬取JS文件中的API路径
  2. 查找Swagger/OpenAPI文档（`/v2/api-docs`、`/swagger-ui.html`）
  3. 反编译`/WEB-INF/classes/`下的.class文件获取路由注解

### 3. 绝对路径与相对路径的权限继承问题

**IIS权限继承机制：**
- 子目录默认继承父目录的权限设置
- 若管理员仅禁用`/upload`的“执行”权限，未单独配置`/upload/images/`子目录
- 则`/upload/images/`仍继承父目录权限（若父目录未禁用执行，子目录仍可执行）

**实战绕过场景：**
```
网站目录结构：
C:\wwwroot\
  ├── upload\        (已禁用执行权限)
  │   └── images\    (继承父目录权限，执行权限未被显式禁用！)
  └── index.asp
```
- 将WebShell上传至`/upload/images/shell.asp`
- 访问 `http://example.com/upload/images/shell.asp` → **成功执行**
- 因为权限是逐级继承的，只有显式配置子目录禁用才会阻止执行

---

## 二、中间件权限继承：不同搭建工具的底层差异

### 1. 宝塔面板的“安全加固”机制

**底层实现：**
- PHP运行身份：以`www`用户（低权限账户）运行PHP-CGI进程
- `php.ini`强制注入：`disable_functions = exec,system,passthru,shell_exec,proc_open,popen,curl_exec,curl_multi_exec,parse_ini_file,show_source`
- 文件系统限制：`open_basedir`限制PHP只能访问网站目录（如`/www/wwwroot/example.com/`）

**实战验证：**
```php
<?php system('whoami'); ?>
```
- 宝塔环境返回：空或`Warning: system() has been disabled for security reasons`
- 浏览根目录：点击`C:\`提示`Permission denied`，因为`www`用户无C盘列表权限

**安全设计意图：**
- 即使攻击者上传了WebShell，也无法执行系统命令、读取系统敏感文件
- 只能操作网站目录内的静态资源，相当于“废弃的Shell”

### 2. PHPStudy（老版本）的“权限继承”问题

**底层实现：**
- Apache/Nginx服务以`SYSTEM`或`Administrator`身份启动
- PHP-CGI子进程继承父进程的高权限令牌（Token）

**实战验证：**
```php
<?php system('whoami'); ?>
```
- PHPStudy环境返回：`nt authority\system` 或 `administrator`
- 可执行`net user hacker 123 /add`添加系统账户
- 可读取`C:\Windows\System32\config\SAM`（密码哈希）

**风险根源：**
- 软件为方便用户（无需配置权限），默认以管理员身份运行所有服务
- 一旦网站存在上传漏洞，直接导致服务器完全沦陷

### 3. IIS自带搭建的“应用池隔离”机制

**底层实现：**
- 默认使用`ApplicationPoolIdentity`（虚拟低权限账户）
- 该账户属于`IIS_IUSRS`组，权限受限

**实战验证：**
```php
<?php system('whoami'); ?>
```
- IIS环境返回：`iis apppool\DefaultAppPool`
- `exec`函数未被禁用（宝塔禁用了，但IIS默认保留）
- 可通过执行`ping`命令探测出网，但直接提权需要额外漏洞（如`Juicy Potato`、`Rotten Potato`）

**三种环境权限对比（核心知识点）：**

| 环境 | PHP运行身份 | `disable_functions` | 文件访问范围 | 提权难度 |
|------|-------------|---------------------|--------------|----------|
| 宝塔面板 | `www`（低权限） | 禁用了几乎所有危险函数 | 仅网站目录 | 极高（需内核漏洞） |
| PHPStudy（老版） | `Administrator`/`SYSTEM` | 未禁用 | 全盘 | 零（已经是最高权限） |
| IIS自建 | `IIS APPPOOL\xxx` | 未禁用 | 受限（需绕过ACL） | 中（可利用Potato家族） |

---

## 三、MIME类型与处理程序映射的解析逻辑

### 1. MIME类型决定文件处理方式

**IIS中MIME类型的作用：**
- 通过文件扩展名决定服务器返回的`Content-Type`响应头
- 浏览器根据`Content-Type`决定如何处理响应内容

**默认MIME映射示例：**
| 扩展名 | MIME类型 | 浏览器行为 |
|--------|----------|------------|
| `.png` | `image/png` | 内联显示图片 |
| `.zip` | `application/zip` | 提示下载 |
| `.asp` | `application/asp` | 交给ASP.DLL处理（执行脚本） |

**配置错误的安全影响：**
> 若管理员将`.asp`的MIME类型错误配置为`text/plain`，访问`shell.asp`时：
> 1. IIS查找处理程序映射，发现`.asp`未配置给`asp.dll`
> 2. 由于无关联的处理程序，IIS直接读取文件内容并以`text/plain`返回
> 3. 结果：ASP源码（包含数据库密码等敏感信息）在浏览器中明文显示

### 2. 处理程序映射（Handler Mapping）的执行链

**IIS请求处理流程：**
```
HTTP请求 → IIS接收 → 根据文件扩展名查找Handler映射 → 调用对应ISAPI扩展/模块 → 返回响应
```

**关键映射：**
| 扩展名 | 处理程序 | 路径 |
|--------|----------|------|
| `*.asp` | `asp.dll` | `%windir%\system32\inetsrv\asp.dll` |
| `*.aspx` | `aspnet_isapi.dll` | `%windir%\Microsoft.NET\Framework\v4.0.30319\aspnet_isapi.dll` |
| `*.php` | `FastCGI` → `php-cgi.exe` | 通过`php.ini`配置 |

**安全配置检查点：**
- 未在映射列表中的扩展名会被当作静态文件处理（直接输出内容）
- 此即“解析漏洞”的根源：上传`shell.jpg`时，若可将其重命名为`shell.php`，且有PHP映射，则触发解析执行

---

## 四、数据库部署模式与连接限制的底层机理

### 1. SQL Server / MySQL 的“IP绑定”机制

**MySQL配置示例（my.ini）：**
```ini
[mysqld]
bind-address = 127.0.0.1   # 仅监听本地回环
# bind-address = 0.0.0.0  # 监听所有接口（默认不推荐）
```

**SQL Server配置（SQL Server Configuration Manager）：**
- SQL Server Network Configuration → Protocols for MSSQLSERVER → TCP/IP → IP Addresses
- 可配置启用/禁用特定IP的监听

**实战场景：**
```
获取到的数据库配置：
server=10.0.0.5;database=db_cms;uid=root;pwd=123456

在VPS上执行：
mysql -h 10.0.0.5 -u root -p123456

返回错误：
ERROR 1130 (HY000): Host 'x.x.x.x' is not allowed to connect to this MySQL server
```

**底层原因：**
- `mysql.user`表中该账号的`Host`字段值决定允许连接的客户端IP范围
- `Host='localhost'` → 仅允许本机连接
- `Host='10.0.0.%'` → 仅允许10.0.0.*网段
- `Host='%'` → 允许任意IP（安全风险最高）

**渗透测试影响：**
- 即使获得完整数据库凭证，也无法从外网直连
- 必须先拿下同一内网的跳板机，再从跳板机连接数据库

### 2. 云数据库RDS的“双层防护”机制

**第一层：网络ACL（安全组/IP白名单）**
- RDS控制台可配置“白名单”，限制访问来源IP
- 该规则作用于网络层（VPC），优先级高于数据库账号密码认证
- 若`/0`未加入白名单，则所有外网IP的TCP握手包被直接丢弃（Connection Reset）

**第二层：账号权限分离**
- 不提供`root`账号，需创建子账号
- 子账号可限制访问的数据库和操作类型（只读/读写）

**渗透测试突破限制的必要条件：**
1. 获得RDS账号密码（通过SQL注入/配置文件泄露）
2. 必须从已授权的IP地址发起连接（通常是Web服务器出口IP）
3. 若RDS配置了专用网络（VPC），必须获得内网权限才能访问

---

## 五、OSS对象存储：上传漏洞失效的底层原理

### 1. 解析环境剥离

**传统上传漏洞生效条件：**
```
上传shell.php → 位于网站目录 /uploads/shell.php
访问 http://web.com/uploads/shell.php
→ Web服务器识别 .php 扩展名 → 调用 PHP-CGI 解析执行
→ 获得权限
```

**启用OSS后的执行链：**
```
上传shell.php → 网站后端调用OSS SDK → PUT请求发送至 OSS Bucket
访问 http://web.com/uploads/shell.php → 但该文件实际存储在OSS
→ OSS返回302重定向至 OSS域名下的真实路径
→ 浏览器访问 OSS域名/shell.php
→ OSS服务器仅支持HTTP文件传输，无PHP解释器 → 以二进制流下载
→ 不执行代码 → 上传漏洞失效
```

### 2. AccessKey的权限治理模型

**RAM策略示例（最小权限原则）：**
```json
{
    "Version": "1",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "oss:PutObject",
                "oss:GetObject"
            ],
            "Resource": "acs:oss:*:*:my-bucket/uploads/*"
        }
    ]
}
```

**AccessKey泄露的连锁风险：**
- 若Key拥有`oss:DeleteObject`权限 → 攻击者可删除所有存储文件
- 若Key拥有`oss:PutBucketPolicy`权限 → 攻击者可修改Bucket策略，公开私有数据
- 若Key拥有`AliyunOSSFullAccess` → 可接管整个OSS服务
- 若该RAM角色还被授权了其他服务（如ECS、RDS）→ 形成横向权限扩散

**安全加固建议：**
- 使用RAM子账号，不分配主账号AccessKey
- 绑定IP白名单（仅允许Web服务器IP使用该AccessKey）
- 定期轮换密钥

---

## 六、CDN隐藏真实IP的穿透技术细节

### 1. 历史DNS记录查询原理

**DNS缓存机制：**
- 域名解析记录（A记录）在全球DNS服务器中有TTL缓存
- 即使站长切换到CDN（CNAME），旧DNS记录仍可能残留在历史数据库中

**查询工具：**
- SecurityTrails（收费，数据最全）
- DNSDumpster（免费，基础查询）
- ViewDNS.info（多工具集成）

**实战命中场景：**
- 站长在2024年1月建站时，直接解析`example.com`到`8.218.70.35`
- 2024年6月开启CDN，解析改为CNAME指向CDN节点
- 但SecurityTrails保留了2024年1月的A记录快照
- 攻击者查询到该历史记录 → 获得真实IP `8.218.70.35`

### 2. SSL证书关联查询（Censys技术原理）

**证书指纹机制：**
- 每个SSL证书有唯一的SHA1/SHA256指纹
- 源站服务器和CDN节点使用**相同的SSL证书**
- Censys持续扫描全球IPv4空间，记录每个IP使用的证书

**查询方法：**
1. 访问 `https://example.com`，获取证书指纹
2. 在Censys搜索该指纹：`services.tls.certificate.parsed.fingerprint_sha256: "xxx"`
3. 返回所有使用该证书的IP地址列表
4. 排除CDN节点IP段（如`119.147.148.0/24`）
5. 剩余IP即为可能的源站地址

### 3. 子域名的“低防护”漏洞

**技术原理：**
- CDN服务按域名计费，站长通常仅对主站（`www.example.com`）开通CDN
- 子域名（`mail.example.com`、`ftp.example.com`、`backup.example.com`）可能直接解析公网IP

**子域名发现方法：**
- 字典爆破：`subdomain.example.com`
- 搜索引擎：`site:example.com -www`
- 证书透明度日志：`crt.sh`查询`%example.com`

**同网段定位：**
- 找到子域名的A记录（如`10.0.1.100`）
- 该IP通常与主站真实IP（如`10.0.1.10`）处于同一`/24`网段
- 扫描该网段开放80/443端口的IP，验证是否为真实源站

---

## 七、WAF拦截的底层逻辑与绕过边界

### 1. 流量特征检测（基于规则的匹配）

**D盾拦截原理：**
- 以IIS模块（Filter）形式注册到请求处理管道中
- 解析HTTP请求的`GET`参数、`POST Body`、`Cookie`
- 正则匹配危险函数名：`eval`、`assert`、`system`、`passthru`、`$_POST`、`$_GET`

**检测示例：**
```
请求：POST /shell.php
Body：<?php eval($_POST['cmd']); ?>
```
D盾规则匹配到`eval`和`$_POST`的组合 → 直接返回`403` → 连接失败

### 2. 加密流量的语义分析难题

**攻击者绕过思路：**
```php
<?php
// 使用异或加密绕过关键字检测
$c = chr(ord('_') ^ 0x0F) . chr(ord('P') ^ 0x0F);
eval($c);
?>
```

**商业WAF的对抗手段（语义分析）：**
- 不仅匹配`eval`字符串，还会分析代码执行逻辑
- 解码常见的Base64、GZIP、URL编码
- 即使代码被混淆，解码后的`eval($_POST['x'])`仍会被识别

**绕过的现实难度：**
- 安恒、绿盟、深信服等商业WAF的规则库非常完善
- 公开的绕过技巧（边界字符、大小写、注释插入）已被有效封堵
- 实战中遇到强力WAF，**建议切换攻击面**（如API接口、弱口令、逻辑漏洞），而非死磕绕过

---

## 八、Docker容器渗透的“虚拟文件系统”陷阱

### 1. OverlayFS（联合文件系统）机制

**Docker镜像分层结构：**
```
Layer 1: Ubuntu 20.04 (基础层，只读)
Layer 2: apt-get install nginx (增量层，只读)
Layer 3: copy website files (增量层，只读)
Layer 4: 容器运行时的文件修改 (读写层，可写)
```

**容器内执行`ls /`看到的文件系统：**
- 所有只读层 + 读写层 → 通过OverlayFS合并呈现
- 看起来是一个完整的Linux根目录（`/bin`、`/etc`、`/var`等）

**宿主机看到的是不同的文件系统：**
- 宿主机有独立的`/root`、`/home`、`/var`、`/etc`
- 容器内的`/` ≠ 宿主机的`/`，两者完全隔离

**验证方法：**
```bash
# 在容器内执行
touch /test.txt

# 在宿主机执行
ls / | grep test.txt   # 找不到该文件！
```

### 2. 容器逃逸的条件

**逃逸路径1：特权模式（`--privileged`）**
```bash
docker run --privileged -it ubuntu bash
```
- 容器拥有宿主机大部分设备访问权限
- 可通过挂载`/dev/sda1`到容器内部，直接读写宿主机文件系统

**逃逸路径2：挂载`docker.sock`**
```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock ubuntu bash
```
- 容器内可调用Docker API
- 可在容器内启动一个`--privileged`容器，实现逃逸

**逃逸路径3：内核漏洞**
- CVE-2016-5195（脏牛）
- CVE-2022-0492（cgroup release_agent）
- 需要特定内核版本才可利用

**渗透测试中的认知陷阱：**
> 拿下容器内`root`权限后，执行`whoami`返回`root`，`ls -la /`看到完整文件系统，容易误认为已控制宿主机。只有意识到“文件系统隔离”这个底层机理，才能正确判断渗透阶段。

---

## 九、APP反编译：资产提取的技术细节

### 1. 提取隐藏接口（抓包无法发现）

**抓包的局限性：**
- 只能捕获用户当前操作触发的请求
- 例如登录：`POST /api/login`
- 不会触发管理员功能：`GET /api/admin/users`（除非以管理员身份登录）

**反编译提取方法（jadx + grep）：**
```bash
# 反编译APK
jadx-gui app.apk

# 在反编译后的源码中搜索URL模式
grep -r -E "(https?://|/api/|/admin/|\.com|\.cn)" ./sources/
```

**典型隐藏资产示例：**
- 硬编码的调试接口：`/debug/status`
- 未公开的微服务网关：`http://internal-api.example.com/order/v1/list`
- 后台接口：`/manage/dashboard`（APP中无入口但代码中存在）

### 2. Flutter/React Native 的拆包分析

**Flutter APP文件结构：**
- `assets/index.android.bundle` 包含Dart代码编译产物
- 使用 `flutter build aab --release` 生成的bundle文件是压缩的Dart字节码

**拆包工具：**
- `blutter`（Flutter逆向工具）
- `react-native-decompiler`（React Native拆包）

**提取接口示例：**
```
原始Dart代码（反混淆前）：
_0x1234 = "https://api.example.com/v1";

反混淆后可见：
baseUrl = "https://api.example.com/v1";
```

**提取后的测试价值：**
- 微服务架构中，不同服务的API地址可能硬编码在APP中
- 这些服务可能未在Web端暴露，直接对提取的IP/域名进行渗透测试，可能发现新的攻击面

---

## 十、伪静态技术的底层识别

### 1. URL重写的工作链

**IIS URL Rewrite配置示例（web.config）：**
```xml
<rewrite>
  <rules>
    <rule name="ArticleRewrite">
      <match url="^article/([0-9]+)\.html$" />
      <action type="Rewrite" url="index.asp?id={R:1}" />
    </rule>
  </rules>
</rewrite>
```
- 用户请求：`http://example.com/article/123.html`
- IIS内部重写为：`http://example.com/index.asp?id=123`
- 实际执行的是`index.asp`动态脚本

### 2. 真静态与伪静态的区分技术

| 验证方法 | 真静态 | 伪静态 |
|----------|--------|--------|
| 修改数据库内容 | 页面不变（需重新生成HTML） | 页面实时变化 |
| URL后缀 | `.html` | `.html`（相同，表面无法区分） |
| 查看响应头 | `Last-Modified`为文件修改时间 | `Last-Modified`为当前时间 |
| 服务器性能 | 低（直接读取文件） | 高（执行数据库查询） |

### 3. 渗透测试中的检测方法

**方法1：修改参数观察变化**
```
访问 /article/1.html 和 /article/2.html
若内容随ID变化，则为动态生成 → 伪静态
若1.html和2.html内容无关联，可能为真静态
```

**方法2：检查响应头**
```bash
curl -I http://example.com/article/1.html
# 真静态：Last-Modified: Wed, 01 Jan 2025 10:00:00 GMT（固定时间）
# 伪静态：Last-Modified: Wed, 29 Jun 2026 14:30:25 GMT（当前时间）
```

**方法3：查看是否有重写配置文件**
- 尝试访问 `/web.config`、`/.htaccess`（若允许读取，则暴露重写规则）

---

## 十一、动态解析与权限动态继承

### 1. 域名解析的“泛解析”陷阱

**泛解析配置：**
- DNS添加`*.example.com`的A记录
- 所有未明确定义的子域名都指向同一个IP

**安全影响：**
- 攻击者构造任意子域名（如`test.example.com`），即可访问该IP上的默认网站
- 若默认网站为`/var/www/html`目录，可能暴露敏感信息
- 子域名的Cookie、Session若不隔离，可能跨域访问

### 2. 权限动态继承（IIS父子目录）

**IIS权限继承规则：**
- 站点级权限（根目录） → 应用程序级权限 → 子目录级权限
- 子目录默认继承父目录设置，除非显式覆盖

**实战场景：**
```
站点根目录 C:\wwwroot\（执行权限：启用）
  └── upload\（执行权限：显式禁用）
        └── images\（未配置权限，继承父目录 upload 的设置）
```
访问 `http://example.com/upload/images/shell.asp` → 父目录执行权限已禁用，故无法执行

```
站点根目录 C:\wwwroot\（执行权限：启用）
  └── upload\（未配置权限，继承根目录设置 → 执行权限启用！）
```
访问 `http://example.com/upload/shell.asp` → **成功执行**（管理员漏配了upload目录）

**渗透测试要点：**
- 当主上传目录被禁用执行权限时，尝试往子目录上传
- 很多管理员只配了`/upload`，忘记配`/upload/images/`、`/upload/files/`等子目录

---

## 十二、小程序AppSecret重置的影响

**AppSecret的定位：**
- 公众号/小程序的调用凭证，用于获取`access_token`
- `access_token`可调用大多数微信开放接口（如客服消息、模板消息、用户信息获取）

**重置AppSecret的连锁影响：**
1. 旧的`access_token`立即失效（约5分钟内全部过期）
2. 所有使用旧`AppSecret`的业务系统（小程序后端、第三方平台）全部报错
3. 需手动更新代码中配置的新`AppSecret`
4. 若业务系统使用`AppSecret`自动刷新`access_token`，刷新失败会导致服务中断

**渗透测试中的利用：**
- 若获得小程序的管理后台权限，可重置`AppSecret`
- 重置后，原`AppSecret`失效，可阻断小程序服务（相当于DOS攻击）
- 但无法通过重置`AppSecret`获取用户数据（用户数据需单独授权）

---

## 十三、知识要点速查表

| 知识点 | 核心技术机理 | 渗透测试影响 |
|--------|--------------|--------------|
| 路由映射访问 | URL由框架路由表解析，非物理路径 | 上传后无法通过路径访问，需分析路由配置 |
| 权限继承（IIS） | 子目录默认继承父目录权限设置 | 上传目录禁用执行权限后，子目录可能仍可执行 |
| 宝塔安全加固 | `disable_functions` + `open_basedir` | WebShell无法执行系统命令和读取全盘 |
| PHPStudy权限继承 | 服务以SYSTEM身份运行 | 上传漏洞直接导致服务器沦陷 |
| MIME类型配置错误 | 无关联处理程序时直接输出源码 | 访问脚本文件返回源码（代码泄露） |
| 数据库IP绑定 | `bind-address` + `mysql.user.Host` | 即使获得账号密码也无法外网直连 |
| RDS安全组 | 网络层ACL拦截，优先于账号认证 | TCP握手阶段直接被阻断 |
| OSS解析剥离 | OSS服务器无脚本解释器 | 上传WebShell无法执行，漏洞失效 |
| AccessKey泄露 | 可控制OSS存储甚至云服务 | 需评估RAM权限范围，可能横向扩展 |
| CDN历史DNS | 旧A记录保存在第三方数据库 | 可查到开启CDN前的真实IP |
| SSL证书关联 | 源站和CDN共用同一证书 | Censys可定位所有使用该证书的IP |
| Docker OverlayFS | 容器内文件系统与宿主机隔离 | 拿下容器权限不等于控制宿主机 |
| 容器逃逸 | `--privileged` + 挂载`sock` | 需特定条件才能突破隔离 |
| APP反编译 | 硬编码URL/API提取 | 发现未在界面中暴露的隐藏接口 |
| 伪静态检测 | 修改数据观察变化 + 响应头分析 | 识别是否为动态网站 |
| 泛解析 | `*.example.com`全部指向同一IP | 可访问默认站点暴露敏感信息 |
| AppSecret重置 | 旧凭证立即失效 | 业务中断，可用于DOS攻击 |