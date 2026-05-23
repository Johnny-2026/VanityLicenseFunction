# VanityLicense v1.1.4b - 云插件注入系统

> **基于 JNI 原生层的 Minecraft 服务端云插件分发与内存注入系统**

Vanity 是一套完整的云插件管理解决方案，包含 Python Flask 管理后台、Spigot/Bukkit 插件客户端，以及 C++ JNI 原生层。支持插件的加密上传、云端分发、无磁盘内存注入，以及全生命周期的服务器实例管理。**v1.1.4b** 重构了包结构，Java 主类与原生库全面更名为 VanityLicense。

---

## 目录

- [技术栈](#技术栈)
- [项目结构](#项目结构)
- [架构总览](#架构总览)
- [关键实现细节](#关键实现细节)
- [安全防护说明](#安全防护说明)
- [部署指南](#部署指南)
- [构建指南](#构建指南)
- [API 参考](#api-参考)
- [许可](#许可)

---

## 技术栈

### 服务端 (Server/)
| 技术 | 用途 |
|---|---|
| **Python 3.11 + Flask** | Web 框架，REST API |
| **SQLite3** | 轻量级数据库（服务器绑定、动态 Token） |
| **cryptography (AESGCM)** | 插件加密存储与传输 |
| **pyotp + qrcode** | 双因素认证 (TOTP) |
| **Cloudflare Turnstile** | 前端人机验证 |
| **Werkzeug (ProxyFix)** | Nginx 反向代理信任 |
| **自定义 PoW** | Proof-of-Work 人机验证 |

### 客户端 Java 层 (Client_CloudLoader/)
| 技术 | 用途 |
|---|---|
| **Spigot API 1.8.8** | Bukkit 插件框架 |
| **Maven** | 构建管理 (maven-shade-plugin) |
| **JNI** | Java ↔ C++ 原生桥接 |
| **Allatori** | 代码混淆（类名/控制流/字符串加密） |

### 客户端 C++ 原生层 (Client_Native/)
| 技术 | 用途 |
|---|---|
| **C++11** | 核心逻辑语言 |
| **自实现 SHA256** | 纯 C 实现，无 OpenSSL 依赖，跨发行版兼容 |
| **自实现 HMAC-SHA256** | 请求签名 |
| **dlopen 动态加载 (Linux)** | libcurl.so 网络请求 / libcrypto.so AES-GCM 解密 |
| **WinHTTP (Windows)** | Windows 原生 HTTP 栈 |
| **BCrypt (Windows)** | Windows 原生加密 API |
| **miniz** | 单文件 ZIP 解析库，用于解析 JAR 包 |
| **glibc 兼容层** | 提供 `__libc_single_threaded` 符号，支持 CentOS 8 (glibc 2.28) + Ubuntu 22.04+ (glibc 2.35) |
| **MinGW-w64** | Linux → Windows 交叉编译 |
| **UPX** | 可选的可执行文件压缩 |

---

## 项目结构

```
Vanity/
├── Server/                                  # Python Flask 服务端
│   ├── app.py                               # 主应用 (~1400行)
│   ├── static/
│   │   ├── css/
│   │   │   ├── admin.css                    # 管理后台样式
│   │   │   ├── tokens.css                   # Token 管理样式
│   │   │   ├── login.css                    # 登录页样式
│   │   │   ├── verify.css                   # 人机验证样式
│   │   │   ├── twofa_setup.css              # 2FA 设置样式
│   │   │   └── twofa_verify.css             # 2FA 验证样式
│   │   ├── js/
│   │   │   └── verify.js                    # PoW 求解器 (Web Crypto API)
│   │   └── pictures/
│   ├── templates/                           # Jinja2 模板
│   │   ├── admin.html                       # 管理面板
│   │   ├── login.html                       # 登录页
│   │   ├── verify.html                      # 人机验证页
│   │   ├── tokens.html                      # Token 管理页
│   │   ├── server_tokens.html               # 服务器 Token 详情
│   │   ├── twofa_setup.html                 # 2FA 设置
│   │   ├── twofa_verify.html                # 2FA 验证
│   │   ├── add_form.html / edit_form.html   # 服务器管理表单
│   │   └── upload_form.html                 # 插件上传表单
│
├── Client_CloudLoader/                      # 客户端 Java 插件
│   ├── pom.xml                              # Maven 构建配置
│   ├── config.xml                           # Allatori 混淆配置
│   └── src/main/
│       ├── java/xyz/VanityDev/License/VanityLicense.java  # 主插件类
│       └── resources/
│           ├── plugin.yml                   # Bukkit 插件描述
│           └── native/                      # C++ 编译产物
│               ├── vanity_license.dll         (Windows)
│               └── libvanity_license.so        (Linux)
│
├── Client_Native/                           # C++ 原生层
│   ├── cloud_native.cpp                     # 完整原生逻辑 (~2500行)
│   ├── miniz.c / miniz.h                    # ZIP 解析库
│   ├── build_linux.sh                       # Linux 构建脚本 (CentOS 8 sysroot)
│   ├── build_windows.sh                     # Windows 交叉编译脚本 (MinGW)
│   ├── build_all.sh                         # 一键构建
│   ├── setup_centos8_sysroot.sh             # CentOS 8 sysroot 搭建
│   ├── jni_headers/                         # JNI 头文件
│   └── centos8-sysroot/                     # CentOS 8 glibc 2.28 sysroot
│
└── .gitignore
```

---

## 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        Minecraft Server                          │
│  ┌──────────────────────┐        ┌────────────────────────────┐ │
│  │  VanityLicense (Java)│◄──────►│  libvanity_license.so (C++)│ │
│  │  - Bukkit Plugin      │  JNI   │  - 网络请求                  │ │
│  │  - 心跳调度            │        │  - AES-GCM 解密             │ │
│  │  - Unsafe 辅助方法     │        │  - 硬件指纹采集              │ │
│  └──────────┬───────────┘        │  - 内存类定义 (DefineClass) │ │
│             │                     │  - 插件注入 (Unsafe/反射)    │ │
│             │                     └──────────────┬─────────────┘ │
│             ▼                                    │               │
│  ┌──────────────────────┐                        │               │
│  │  内存插件实例 (Java)   │◄───────────────────────┘               │
│  │  - Unsafe 分配         │    JNI env->DefineClass               │
│  │  - 正常构造函数运行     │    定义所有类到假 ClassLoader           │
│  │  - 完整 Bukkit 插件    │                                        │
│  └──────────────────────┘                                        │
└─────────────────────────────────────────────────────────────────┘
                            │ HTTPS
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Cloud Server (Flask)                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────────┐  │
│  │ 认证API   │  │ 插件分发  │  │ 心跳管理  │  │ 管理后台 (Web)   │  │
│  │ /auth     │  │ /plugin  │  │/heartbeat│  │ /admin.html     │  │
│  │/register  │  │  (AES)   │  │          │  │ 2FA + PoW + TS  │  │
│  └──────────┘  └──────────┘  └──────────┘  └─────────────────┘  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  SQLite: servers / registrations                            │ │
│  │  插件加密存储: plugins/*.jar.enc                              │ │
│  │  会话文件: sessions/*                                        │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 核心流程

1. **管理员** 通过 Web 后台上传插件 → 服务端用 `MASTER_KEY` AES-GCM 加密存储为 `{pid}.jar.enc`
2. **管理后台** 将服务器 IP 与插件 ID 列表绑定 (`servers` 表)
3. **Minecraft 服务端启动** → VanityLicense 插件加载 → `static{}` 提取 JAR 内原生库到 `cache/` 目录 → `System.load()`
4. **`nativeAuthenticateAndInject()`** 被调用：
   - C++ 层采集硬件指纹（MAC地址 / machine-id）
   - 获取公网 IP
   - 生成动态 Token（HMAC-SHA256 指纹 + IP + Salt）
   - POST `/register_dynamic` 注册 Token → POST `/auth` 获取会话密钥和插件列表
   - HTTP 下载加密 JAR → AES-GCM 解密 → 解析 plugin.yml → 拓扑排序
   - 逐个插件：用 `Unsafe.allocateInstance()` 创建假的 `PluginClassLoader`
   - 通过 `JNI DefineClass` 绕过 Java 21 模块系统，在**纯内存**中定义所有类
   - 正常调用构造函数 → `onLoad()` → 注册到 `PluginManager` → `onEnable()`
5. **心跳线程** 每 4 秒 (80 ticks) 异步发送心跳 → 服务端检测到 `restart_requests` 匹配时返回重启指令 → 客户端 `Server.shutdown()`

---

## 关键实现细节

### 1. 双重加密插件分发

```
上传: JAR 文件 → AESGCM(MASTER_KEY, nonce_random, jar_data) → {pid}.jar.enc
分发: {pid}.jar.enc → 客户端请求 → 服务端用 MASTER_KEY 解密
       → 再用 session_key (AESGCM.generate_key) 重新加密
       → 客户端收到后用 session_key 解密得到明文 JAR
```

- 服务端存储使用固定主密钥 (`MASTER_KEY`) 加密
- 每个会话生成独立的 `session_key`，客户端下载时二次加密
- 每次下载后标记为"已消费"，防止同一会话重复下载

### 2. 纯内存插件注入 (零磁盘写入)

这是项目的核心技术难点，绕过 JDK 9+ 模块系统限制：

```
PluginClassLoader 创建：
  1. Unsafe.allocateInstance(PluginClassLoader.class)
     → 跳过构造函数，所有字段为 null
  
  2. Java sum.misc.Unsafe 填充字段：
     - loader, description, dataFolder, file, classes (ConcurrentHashMap)
  
  3. JNI 强制设置：
     - ClassLoader.unnamedModule (Java 21 必须)
     - ClassLoader.classes (ArrayList, JDK 21 用 Vector→ArrayList)
     - URLClassLoader.ucp (URLClassPath, 防止 getResource NPE)
  
  4. JNI env->DefineClass() 定义所有类在假 ClassLoader 上：
     - 先 definePackage() 确保包存在
     - 从 C++ std::map 中读取类字节码，不经过 JarFile
  
  5. 正常构造函数 new MainClass() 保留字段初始化器
  
  6. 调用 JavaPlugin.init() → onLoad() → onEnable()
```

### 3. .class 字节码常量池解析

由于 `Unsafe.allocateInstance()` 跳过了所有字段初始化，对于那些在 `<init>` 中拼接字符串的字段（如 `ChatColor.WHITE + "标题" + ChatColor.AQUA + "子标题"`），C++ 层解析 `.class` 文件的常量池和 `<init>` 字节码，提取所有 `ldc` / `ldc_w` + `putfield` 模式的字符串常量并拼接还原。

```
.class 字节码解析流程：
  CAFEBABE → 常量池 → 方法区 → <init> 字节码
  → 扫描 ldc(0x12) / ldc_w(0x13) 指令收集字符串
  → putfield(0xb5) 时拼接所有 pending 字符串
  → 还原到 Java 对象的字段
```

### 4. 硬件指纹与动态 Token

```
Fingerprint = GetMacAddress() || /etc/machine-id || uname
DynamicToken = SHA256(HWID + "|" + PublicIP + "|" + SALT + "|" + InstanceID)
```

- 实例每次启动生成随机 `InstanceID`，确保 Token 唯一性
- Salt 和 MasterKey 硬编码在 C++ 层，Java 层不可见
- Token 通过 `/register_dynamic` 注册到服务端，绑定 IP 和心跳时间

### 5. 心跳与远程重启系统

```
Java 端 (BukkitRunnable, 每4秒异步) 
  → nativeSendHeartbeat(token)
    → C++ 线程：HMAC-SHA256 签名 → POST /heartbeat
    → 服务端检查 restart_requests 集合
    → 匹配 → 返回 {"restart": true}
  → C++ 返回 JNI_FALSE → Java 调用 Server.shutdown()
```

- 服务端批量重启：管理员发出指令 → 入队 `pending_restart_queue` → 后台线程每 25 秒取 3 个 Token → 加入 `restart_requests` → 下次心跳命中
- 心跳失败超过 20 次自动关机，防止幽灵实例

### 6. 自定义 DNS 解析

绕过系统 DNS 解析，通过 UDP 直连 `1.1.1.1` 解析 `cdn.moonjump.club`：

```
1. 构造 DNS Query (A 记录) → UDP 发送至 1.1.1.1:53
2. 解析 Response 获取 IPv4 地址
3. 后续 HTTP 请求以 IP 直连，附加 Host header
4. Linux: curl_easy_setopt(CURLOPT_RESOLVE, ...)
   Windows: WinHttpOpenRequest + WinHttpAddRequestHeaders(Host)
```

防止 DNS 劫持，确保连接不能被中间人重定向。

### 7. 跨平台兼容 (C++ 端)

- **Linux 兼容性**：可在 CentOS 8 (glibc 2.28) 和 Ubuntu 22.04+ (glibc 2.35) 上运行
  - 提供 `__libc_single_threaded` 符号为旧 glibc 兼容
  - CentOS 8 sysroot 编译，确保二进制只依赖 glibc 2.28
  - 动态加载 libcurl.so.4 / libcrypto.so.3/1.1，不硬链接版本
  - 自实现 SHA256（纯 C 无外部依赖）
- **Windows 兼容**：MinGW 交叉编译，静态链接 libstdc++/libgcc
- **glibc locale 线程安全**：`JNI_OnLoad` 中预热 `std::locale::classic()`，防止多线程首次访问 locale 时 SIGSEGV

### 8. Anti-Debug 反调试体系

C++ 原生层启动后立即派生独立线程持续监控调试行为：

**Linux 反调试**
- **TracerPid**：读取 `/proc/self/status` 检查 TracerPid 是否为 0
- **LD_PRELOAD**：检查环境变量是否被劫持
- **INT3 扫描**：扫描自身 `.text` 段 0xCC 断点
- **Java Agent 检测**：扫描 `/proc/self/cmdline` 检测 `-javaagent` 参数

**Windows 反调试**
- **IsDebuggerPresent**：标准 PEB 调试标志检测
- **CheckRemoteDebuggerPresent**：远程调试器检测
- **NtQueryInformationProcess**：直接 ntdll 调用，绕过 kernel32 钩子
- **PEB BeingDebugged 直读**：不入库调用，直接读结构体偏移
- **INT3 扫描**：扫描 .text 段的软件断点
- **DLL 注入检测**：枚举加载模块，检查是否从临时目录加载

任何检测到异常，立即 fork 子进程上报 `/report_anti_debug` 后自毁。

### 9. Anti-Dump 防内存转储

通过拦截 SIGQUIT 信号 + watchdog 进程实现双重防护：

**SIGQUIT 拦截** (Linux)
- 注册 `sigaction` 拦截 `SIGQUIT`（信号 3）
- jmap/jstack/jcmd 等工具依赖此信号生成线程转储
- 拦截后立即上报 `/report_dump` + 自毁

**Watchdog 守护进程**
- `nativeAuthenticateAndInject` 成功后 fork 子进程
- 持续监控以下指标：
  - `/proc/self/exe` 的 dumpable 标志是否被篡改
  - SIGQUIT 信号处理是否被移除
  - 原生 `.so` 是否被 dlclose 卸载
  - AntiDebug 监控线程是否被杀死
- 任何异常触发即时上报 `/report_watchdog`

**预分配信号安全缓冲区**
- 所有上报 URL 和数据在启动时预分配为 C 风格字符数组
- 信号处理器内只执行 `write()` / `snprintf()`，绝不分配堆内存
- 避免 `std::string` 在信号上下文中死锁

### 10. 前端 PoW (Proof-of-Work) 人机验证

伪 Cloudflare Turnstile 界面，后台使用 Web Crypto API 进行 SHA256 工作量证明：

```
挑战: SHA256(challenge + nonce) 的前 {difficulty} 位为 0
难度: 14 bits → 平均需要 ~2^14 = 16384 次哈希
流程: 页面加载 → 后台开始 PoW 计算 → 用户点击 "Confirm you are human"
      → 提交 nonce → 服务端验证 → 创建验证会话 (45分钟有效)
```

- 中国 IP 自动跳过 PoW（基于 ip-api.com 地理定位），直接创建验证会话
- 突发流量模式下（50 个独立 IP / 10 秒），强制所有 IP 走 PoW
- PoW 挑战 60 秒过期，一次性使用

---

## 安全防护说明

### 传输安全

| 防护措施 | 实现方式 |
|---|---|
| **HTTPS** | 全站 TLS（推荐 Nginx 反向代理） |
| **请求签名** | 所有 API 调用携带 HMAC-SHA256 签名 + UNIX 时间戳 |
| **时间戳防重放** | 服务端校验 `abs(now - timestamp) <= 180s` |
| **自定义 DNS** | 客户端通过 UDP 直连 1.1.1.1 解析服务端域名，防 DNS 劫持 |
| **会话密钥** | 每个认证会话生成独立 AES-256 会话密钥，插件传输二次加密 |

### 存储安全

| 防护措施 | 实现方式 |
|---|---|
| **插件加密存储** | 服务端所有 JAR 文件以 `.jar.enc` 格式 AES-GCM 加密存储 |
| **双密钥体系** | 主密钥加密静态存储，会话密钥加密传输 |
| **会话一次性消费** | 每个 session_id 对每个插件仅允许下载一次 |
| **会话过期** | 会话文件 30 秒无访问自动过期删除 |
| **路径穿越防护** | session_id 严格校验 `/^[0-9a-f]+$/`，防止 `../` 攻击 |

### 认证安全

| 防护措施 | 实现方式 |
|---|---|
| **硬件绑定** | 动态 Token 绑定 MAC 地址 / machine-id + 公网 IP |
| **IP 锁定** | 服务端校验请求 IP 与注册 IP 一致 |
| **Token 注册机制** | 需携带 `MASTER_TOKEN` 密钥才能注册动态 Token |
| **心跳过期** | 350 秒无心跳自动标记离线，Token 失效 |
| **双因素认证** | 管理后台支持 TOTP (pyotp)，secret 文件存储，禁用需密码验证 |

### 访问控制

| 防护措施 | 实现方式 |
|---|---|
| **人机验证门控** | 全局中间件，所有管理页面需通过 PoW 或中国 IP 跳过 |
| **中国 IP 白名单策略** | ip-api.com 识别中国 IP 24h 缓存，直接跳过 PoW |
| **突发流量检测** | 10 秒窗口内 ≥50 个独立 IP 触发突发模式，强制所有人验证 |
| **登录限速** | 每次登录间隔 ≥5 秒，每分钟 ≤10 次 |
| **API 限速** | 基于滑动窗口的速率限制器，各接口独立配额 |
| **Cloudflare Turnstile** | 登录页集成 Turnstile 人机验证 |
| **会话管理** | Flask 会话 10 天有效，`secret_key` 服务启动随机生成 |
| **退出登录** | 保留人机验证状态，仅清除登录凭证 |

### 运行时安全

| 防护措施 | 实现方式 |
|---|---|
| **Anti-Debug** | 多维度反调试：TracerPid 检测、LD_PRELOAD 检测、INT3 0xCC 断点扫描 (Linux)；IsDebuggerPresent / CheckRemoteDebugger / NtQueryInformationProcess / PEB BeingDebugged 直读 (Windows) |
| **Anti-Dump** | SIGQUIT 拦截 + jmap/jstack/jcmd 进程检测，禁止内存转储 |
| **Java Agent 检测** | 客户端扫描 `/proc/self/cmdline` 检测非白名单 Agent，上报黑名单 |
| **Watchdog 守护进程** | fork 子进程持续监控防护措施完整性，检测 dumpable 被改、SIGQUIT 处理被移除、.so 被卸载、AntiDebug 线程被杀死，异常即时上报 |
| **黑名单机制** | 上报检测到的异常 Agent/调试行为，管理员可将服务器标记为 `blacklist`，拒绝后续认证 |
| **心跳自毁** | 连续 20 次心跳失败自动关闭服务端，防止幽灵实例 |
| **多级速率限制** | 滑动窗口 + 登录间隔 + API 配额三层限速 |
| **双因素认证 (2FA)** | 基于 TOTP (RFC 6238)，支持 setup/verify/disable 全生命周期 |
| **敏感数据零 Java 驻留** | 所有密钥、Token 由 C++ 层持有，Java 层无法反射读取 |
| **Native 库保护** | 编译 strip + UPX 压缩 + `-fvisibility=hidden`，只有 JNI 导出可见 |
| **Allatori 混淆** | Java 层类名重命名 (iii(20)) + 控制流混淆 + 字符串加密 + 行号混淆 |
| **自动清理** | 60 秒定时清理过期 Token 和重启请求残留，防止表膨胀 |

### 防护纵深示意

```
用户请求
    │
    ▼
┌─────────────┐
│ Turnstile    │── 前端人机验证
└──────┬──────┘
       ▼
┌─────────────┐
│ PoW 验证     │── SHA256 工作量证明 (14 bit)
└──────┬──────┘
       ▼
┌─────────────┐
│ 中国 IP 跳过  │── GeoIP 地理定位 + 24h 缓存
└──────┬──────┘
       ▼
┌─────────────┐
│ 突发检测      │── 50 IPs / 10s → 强制PoW
└──────┬──────┘
       ▼
┌─────────────┐
│ 登录限速      │── 5s间隔 / 10次每分钟
└──────┬──────┘
       ▼
┌─────────────┐
│ 密码 + 2FA   │── TOTP 双因素
└──────┬──────┘
       ▼
┌─────────────┐
│ 管理后台      │── 完整 CRUD 操作
└─────────────┘

客户端连接
    │
    ▼
┌─────────────┐
│ Token 注册    │── 需携带 MASTER_TOKEN
└──────┬──────┘
       ▼
┌─────────────┐
│ 请求签名      │── HMAC-SHA256 + 时间戳
└──────┬──────┘
       ▼
┌─────────────┐
│ IP 锁定       │── 注册IP == 请求IP
└──────┬──────┘
       ▼
┌─────────────┐
│ 心跳验证      │── 350s 超时断连
└──────┬──────┘
       ▼
┌─────────────┐
│ 会话密钥      │── AES-GCM 二次加密
└──────┬──────┘
       ▼
┌─────────────┐
│ 一次性消费     │── 每插件每会话一次
└─────────────┘
```

---

## 部署指南

### 服务端

```bash
# 1. 安装依赖
pip install flask pyotp qrcode[pil] cryptography pyyaml werkzeug

# 2. 启动
cd Server/
python app.py

# 3. 推荐 Nginx 反向代理
# server {
#     listen 443 ssl;
#     server_name cdn.moonjump.club;
#     location / {
#         proxy_pass http://127.0.0.1:5000;
#         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#         proxy_set_header X-Forwarded-Proto $scheme;
#     }
# }
```

### 构建客户端

#### C++ 原生库

```bash
# Linux (推荐 CentOS 8 sysroot 确保 glibc 2.28 兼容)
cd Client_Native/
./build_linux.sh all

# Windows (需要 MinGW 交叉编译工具链)
./build_windows.sh all

# 编译产物:
#   build/libvanity_license.so  (Linux)
#   build/vanity_license.dll    (Windows)
```

#### Java 插件

```bash
# 1. 将编译好的 C++ 库复制到资源目录
cp Client_Native/build/libvanity_license.so Client_CloudLoader/src/main/resources/native/
cp Client_Native/build/vanity_license.dll Client_CloudLoader/src/main/resources/native/

# 2. Maven 打包
cd Client_CloudLoader/
mvn clean package

# 3. (可选) Allatori 混淆
java -jar allatori.jar config.xml

# 产物: target/VanityLicense-5.0.jar
```

---

## API 参考

### 客户端 API (POST)

| 路由 | 速率限制 | 认证方式 | 说明 |
|---|---|---|---|
| `/auth` | 10/60s | HMAC 签名 | 客户端认证，获取会话密钥和插件列表 |
| `/register_dynamic` | 5/60s | `MASTER_TOKEN` | 动态 Token 注册，绑定 IP |
| `/heartbeat` | 无 | HMAC 签名 | 心跳保活，返回重启指令 |
| `/report_blacklist` | 10/60s | Token | 上报异常 Java Agent |
| `/plugin/<id>` | 30/60s | Session ID | 下载加密插件 (GET) |

### 管理后台 API (POST, 需登录 + 人机验证 + 2FA)

| 路由 | 说明 |
|---|---|
| `/add` | 添加服务器绑定 |
| `/update` | 更新服务器配置 |
| `/delete` | 删除服务器绑定 |
| `/upload` | 上传并加密插件 |
| `/delete_plugin` | 删除插件 |
| `/admin/reset/<ip>` | 重置服务器 Token |
| `/admin/restart` | 批量/指定重启 |
| `/admin/restart_instance` | 重启指定实例 |
| `/admin/delete_token` | 删除指定 Token |
| `/admin/delete_offline_tokens` | 批量清理离线 Token |
| `/admin/set_status` | 设置服务器状态 (active/blacklist) |
| `/admin/delete_server_tokens/<ip>` | 删除某 IP 的所有 Token |
| `/login` | 登录 (含 Turnstile 验证) |
| `/2fa/setup` | 启用/配置 2FA |
| `/2fa/verify` | TOTP 验证 |
| `/2fa/disable` | 禁用 2FA (需密码) |

---

## 许可

**版权所有 © 2026 Johnny. All Rights Reserved.**

> 本软件仅供授权用户使用。未经授权不得分发、反编译或修改。
