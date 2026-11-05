# 校园网 Dr.COM / ePortal 多设备共享绕过实战指南（2026/11/05 修订）

基于 OpenWrt 的校园网 **Web 版 Dr.COM / 哆点 ePortal** 防共享检测绕过，**以福建农林大学仓山校区宿舍校园网为例**。

测试环境：
- 福建农林大学仓山校区 / 中国移动校园宽带 / Dr.COM ePortal 4.x
- 路由器：Xiaomi MiWiFi Mini（mt7620 / 128MB） / OpenWrt R24.8.1 / kernel 5.4.278
- 内网 11 台终端（Windows / iOS / Android / HarmonyOS 混合）

测试结果：**稳定（长时在线且无网络故障146）、“永续”在线（健壮的掉线重连），且在其他路由器设备、校园网账号下可复现。**

**为了避免滥用，本仓库不提供具体的代码、脚本。**

本教程重原理剖析，具有较高技术门槛，并非教你如何无脑执行，所以适合有一定技术基础的人阅读，你应至少同时具备以下条件：
- OpenWrt / Linux 基础
- TCP/IP 与抓包能力
- 一定的前端基础
- 与 AI 协作的能力

---

## 🚨 免责声明

**1. 本项目仅供网络安全研究与学习交流使用，严禁传播，严禁商用，严禁用于不法用途！因使用本工具产生的一切后果由使用者自行承担。🚫**

**2. 请遵守学校相关管理规定，切勿用于恶意逃避监管，倡导遵规守纪！🙏**

**3. 我不提供帮助，感谢理解！🤝**

---

## 📄 开源协议

本仓库采用 Apache 2.0 开源协议。

---

## ❓ 可以适配其它学校吗？
理论上"核心思路"可以对全国 80% 以上的高校通用，特别是采用 Web版 Dr.COM / 哆点 ePortal 认证系统的。

## 目录

- [0. 两句话](#0-两句话)
- [1. 对端到底在看什么](#1-对端到底在看什么)
- [2. 侦察：先把对手摸清楚](#2-侦察先把对手摸清楚)
- [3. 方案设计：五层架构](#3-方案设计五层架构)
- [4. 落地：逐条命令](#4-落地逐条命令)
- [5. 验证：怎么证明它真的生效](#5-验证怎么证明它真的生效)
- [6. 自愈：掉线自动重认证](#6-自愈掉线自动重认证)

---

## 0. 两句话
**经过了 一些 深入的实证 ，发现：**
  1. **检测不在协议栈层。** TLS/JA3/HTTP2 指纹在这条链路上**不存在**—— 数据面在华为 BRAS，Dr.COM 只是 RADIUS + Portal 服务器，网关有用户流量 DPI 能力但**当前没有启用**（曾持续监测4小时未发现 DPI 探测，但偶发性的 146 表明可能存在不定时启动的 DPI 探测机制，笔者认为目前实践达到的效果已经足够令人满意，不打算继续折腾了）。
  2. **容易被忽略的判据是「设备自己上报」。** portal 前端会主动探测你的私网网段（图片 `onload/onerror`）、用 WebRTC ICE 取本地 IP、用 ActiveX 拿网关地址，
   然后把「我在 NAT 后面，网关是 X」加密交给服务器（`duodian/send_ua` 的 `s_is_nat`/`s_nat_ip`/`a_nat_ip`/`win_nat_ip`）。
  3. **「伪装」有两个容易被忽略的缺口。**
     - **路由器自己不被伪装。** 中继规则挂在 `PREROUTING -i br-lan`，只拦「从 LAN 进来」的流量；路由器本机进程走 `OUTPUT`，**不经过中继**。默认配置下路由器自己每分钟要向门户发约 10 次请求，UA 是 `Wget/1.21.3` —— 与登录报文的归一 UA **并列出现在同一个公网 IP 上**（推导见 §1.6）。
     - **UA 不是全部。** 只替换 `User-Agent` 时，`Accept-Encoding`、`Accept-Language`、`Referer` 会**原样穿出**。其中 `Referer: http://captive.apple.com/`、`http://www.msftconnecttest.com/` 是各系统 captive portal 的独有指纹 —— **「这是 iOS / Windows / 华为」是从 Referer 泄漏的，不是从 UA**（见 §4.3.2）。

---

### 1.1 认证与会话模型

Dr.COM ePortal 的典型链路：

```
终端 → 路由器 NAT → 校园网接入层 → AC(城域网/BRAS) → RADIUS → 出口
```

AC 侧的一条会话记录大致长这样：

| 字段 | 含义 | 对我们的意义 |
|---|---|---|
| `online_ip` / `online_mac` | 会话绑定的 IP 与 MAC | **是路由器的 WAN IP/MAC**，不是内网设备 |
| `user_account` | 认证账号 | 一个账号 = 一条会话 |
| `is_bind_mac` | 是否绑 MAC | 为 1 时换 MAC 会掉线，为 0 时可自由换 |
| `phone_flag` | 是否被判定为移动端 | 由 UA/特征推导，可用于校验画像一致性 |
| `time_long` / 上下行字节 | 计费与流量 | 异常流量也会触发风控 |

**关键认知：在 NAT 后面，AC 只能看到一台设备。**

但**这不等于"设备数量限制"就自动满足** —— AC 会通过 portal 上报的 `mac_type`（终端类型）、IPv6 地址、计费流量曲线，间接判断"这个 MAC 后面是不是不止一台设备"。
详见 1.3 。

路由器侧的真实情况：

![路由器 DHCP 列表：单 IP 后挂多台终端](image1.png)

AC 自助服务页只有一行，但会给你打上"终端类型"标签：

![AC 自助服务页：终端类型 = 移动终端](image.png)

> ⚠️ **这个"终端类型"标签就是 `mac_type` 的来源**，它由 portal 前端 `getTermType()` 按 UA 计算：
> Android → 2/3，iPad → 4，其他（Windows/Mac/Linux）→ **1**。
> 也就是说：**你用什么 UA 认证，AC 就把你记成什么终端**。多台设备混用却报一个类型，本身就是画像矛盾。

### 1.2 共享检测的真实维度（经过实证）

**✅ 已证实 / ❌ 已证伪 / ⚪ 未验证**：

| # | 维度 | 状态 | 依据 |
|---|---|---|---|
| 1 | **`mac_type` 终端类型** | ✅ | portal `a41.js` 的 `util.getTermType()` 按 UA 算 0/1/2/3/4 并随请求上报|
| 2 | **IPv6 地址** | ✅ | portal 上报 `wlan_user_ipv6`，且有 `ip6.dr.com:9002/v6` 通道；**IPv6 无 NAT，设备 MAC 全裸奔** |
| 3 | 计费流量 42/43 + 52/53 | ✅ | RADIUS 全量累计、Interim 级实时上报（数据确实上报，但**流量大小、流速不构成判据**，构成判据的是**新建连接速率**）|
| 4 | IPv4 TTL | ✅ | TTL 归一即可 |
| 5 | IPID / TCP 选项 / 窗口 / TSval | ✅ | L3 重建 TCP 后由路由器内核生成 |
| 6 | 明文 HTTP 的 UA / `sec-ch-ua` | ✅ | portal 会上传明文 HTTP；实测 UA 多样会导致秒级封禁 |
| 7 | **TLS/JA3/JA4、HTTP/2 指纹** | ❌ | 经过实证，福建农林大学仓山校区校园网未启用 DPI 相关检测。 **但存在与之并列的第二个口径缺口：LAN 设备浏览器的三项头未归一（见 §4.3.3）** |
| **7a** | **客户端自报 NAT** | ✅ | `duodian/send_ua` 上报 `s_is_nat`/`s_nat_ip`/`a_nat_ip`/`win_nat_ip`，前端自己探测私网网段 + WebRTC + ActiveX。 |
| 8 | QUIC / HTTP3 | ✅ | UDP 443 上有独立指纹 |
| 9 | 广播/组播（SSDP/mDNS/LLMNR/WS-D/NetBIOS/DHCP） | ✅ | 出公网 = "家里不止一台设备"的直接证据 |
| 10 | 单 IP 并发连接数 | ✅ | 上千条流 = 典型 NAT 特征 |
| 11 | 主动探测 / ARP 扫描 | ❌ | 经过实证，福建农林大学仓山校区校园网检测无此项 |
| 12 | DNS/NTP 侧信道 | ⚪ | 建议仍接管 |

> **补充实测结论**：**连续观察一整天，146 计零** → UA 归一在稳态确实能压住这个维度；偶发的 146 是残余暴露的长期累积，**不是维度被破**。检测机制存在更接近「命中 N 次才起控」的阈值型行为 + 抖动。

### 1.3 `mac_type` 是怎么被算出来的

portal 前端 `a41.js` 里的逻辑：

```js
util.getTermType = function(ua) {
    if (/Android/i.test(ua)) return 2;      // 或 3，视版本
    if (/iPad/i.test(ua))    return 4;
    return 1;                                // 其他 = 桌面
};
```

这个值随 `online_list` / `login` 请求一起上送，同时还有：

| 参数 | 说明 |
|---|---|
| `mac_type` | 终端类型 0/1/2/3/4 |
| `wlan_user_ipv6` | **终端自己的 IPv6 地址**（如果有） |
| `wlan_user_ip` | 终端 IP |
| `wlan_user_mac` | **BRAS 在重定向 URL 里插入的真实终端 MAC**（NAT 也照样填） |
| `wlan_ac_ip` / `wlan_ac_name` | AC 侧标识 |
| `account_suffix` | 运营商后缀（`@cmcc` 等） |

> **结论**：如果你的认证脚本固定说"我是 Android 手机"（`mac_type=2/3`），
> 但实际跑着 11 台设备、1000+ 并发、开各种端口 —— **画像自相矛盾**，这就是被踢的重要信号之一。
>
> **正确做法**：统一画像

### 1.4 一句话结论

> **TTL 只是入场券，TCP 重建是基本操作，真正决定生死的是「你把自己伪装成什么终端」以及「你在整个过程里有没有暴露你在“说谎”」。**

### 1.5 门户页与企业 App 的 JS 探针

发现了很聪明的设计，属于非报文层的旁路探测，必须单独防：
1. **学校自有 App（如"数字FAFU"）可能会扫局域网拓扑并向服务器上报。**
   - 处置：**用流量打开，别连 WiFi**。
2. **门户页会尝试拿内网 IP。** JS 直接读不到，但 WebRTC 能把真实局域网地址暴露出来。
   - 处置：LAN 用冷门私有段（如 `10.77.0.1/24`）；浏览器装 WebRTC Leak Shield 或禁用 WebRTC。
3. **门户页会上报 NAT 信息**
   - 处置：先让 UA 归一上线，再用浏览器访问门户/自助服务；顺序反了，你自己就是那个"异常终端"。
   - **更高优先级的处置（实测必须这么做）**：用 `block_lan_portal=1` **彻底禁止内网设备直连 portal**（见 4.9），让所有 portal 流量只从路由器发出 —— 这样设备 UA 根本不会暴露。

从 portal 前端 a40.js 反出来的 duodian/send_ua ：

| 字段	| 来源	| 说明 |
|---|---|---|
| user_agent	| 浏览器原样（小写化）| 	UA 确实被上报 |
| s_is_nat / s_nat_ip	| 前端逐个私网地址做图片 onload/onerror 探测	| 命中就把「NAT 网关地址」交上去 |
| a_nat_ip / a_is_nat	| WebRTC ICE candidate	| 移动端报 'android_'+内网IP |
| win_nat_ip / is_nat	| Windows ActiveX | 	is_nat = win_nat_ip ∉ {'', '0.0.0.0', term.ip} |
| terminal_type / wlan_user_ipv6	| 页面 JS	| 终端类型、IPv6 |

**这就是为什么把协议栈伪装得再好也没用 —— 你打开（或主动或被动）一次 portal 页，就等于主动交了一份"NAT 自白书"。**

### 1.6 一个持续自曝的第三方：路由器自身流量

`CAMPUS_RELAY` 挂在 `PREROUTING -i br-lan`，匹配的是**「从 LAN 进来」的流量**。路由器本机进程（认证探活、状态查询、健康检查）走 `OUTPUT`，**指令链上根本不经过这条规则**。

也就是说：**你在 LAN 侧辛苦做出来的「单设备」假象，被路由器自己的请求打出一个又一个例外。**

**怎么验证这一点**（三条，任选）：

| 方法 | 做法 |
|---|---|
| 出口 UA | 在路由器上 `curl -s http://httpbin.org/user-agent` → 拿到的是 wget/curl 自己的 UA，**不是**你配的归一值 |
| conntrack 回程端口 | 路由器自己打外网：回程 `sport=80`；LAN 设备打同一目标：回程 `sport=<中继端口>`。**端口不同 = 没走中继** |
| 抓中继计数器 | 中继进程的连接计数只在有 LAN 设备流量时增长；路由器自身请求不产生中继连接 |

**量级可以自己算**：
- 会话状态查询：`interval`（默认 10 s，很多部署会调到 7 s）一次 → 每小时 360~510 次
- 通网探测：每 30 s 一次 → 每小时 120 次
- 合计 **约 500~630 次/小时**，且**都是打门户**（网关是这条链上唯一的 HTTP 请求头解析点）

对比手机探针：那是「连上 WiFi 才发一次」的偶发事件。**路由器探活是每 7~10 秒一次的持续事件，累积暴露面大两个数量级。**

**更刺眼的是它不自洽**：认证脚本的**登录**报文通常带归一 UA，而**探活**报文不带 → **网关侧同一个 IP 上并列两种 UA**。这已经不是「泄漏」，是「同一台设备在 7 秒内自我矛盾」。

**修法见 §4.3.1。**

---

### 1.7 一个值得记录的排除过程：连通性探测 ≠ 检测输入

很多人会担心「手机连 WiFi 时系统自动做连通性探测，会不会被 AC 当成设备数判据」。**结论：不是。** 但排查过程值得写下来，因为它指向了真问题。

**先分清「三种门户页」** —— 这是排查里最容易混的地方：

| # | 来源 | 触发 | 响应由谁生成 | HTTP 源 IP | 特征 |
|---|---|---|---|---|---|
| **a** | 直取门户 | 访问 AC 的 IP | 门户后端 | **门户 IP** | 有 `Server`/`Date`，Vue SPA 全文，几十 KB |
| **b** | **BRAS 注入** | 访问**任意**明文 HTTP，AC 半路替换首包 | **BRAS 自己** | **目标网站自己的 CDN IP** | `Content-Length: 451` 定长、裸 HTML 只有一句 `location.href`、**无 `Server` 无 `Date`**、参数里带 `wlanacname=BRAS-xxx` |
| **c** | 手机系统探测触发 | 连上 WiFi 后系统自动打探测域名 | 同 (b) | 探测域名自己的 IP | 同 (b)，UA 是 `Dalvik/...` 这类系统栈 |

**判据口诀（很好用）**：

> **响应里有 `Server` / `Date` / 源站跳转 = 通了；只有 `Content-Length: 451` 的裸 HTML + `wlanacname=BRAS-…` = 被 AC 拦了。**

**换句话说：能收到 HTTP 200 ≠ 能上网。** 被封/未认证时，BRAS 会把**所有明文 HTTP** 都注入这个跳转页，HTTPS 则直接 RST 或超时。看到 200 就以为网络正常，是排查里最常见的误判。

**（b)/(c) 为什么封不住**：拦截发生在 AC 侧、响应由 AC 生成，本地封的是「目的 IP」，而这类响应的源 IP 是目标网站自己 —— **规则在物理上没有抓手**。要"封"只能阻止设备发探测请求，但代价是手机永远判定「需认证」、状态栏常驻提示反复弹窗，**而对 146 一点帮助都没有**。

---

## 2. 侦察：先把对手摸清楚

### 2.1 识别厂商与版本

打开认证页，按 F12 看三处：

| 特征 | 结论 |
|---|---|
| `Server: DrcomServer1.0` | Dr.COM / 城市热点 |
| 页面里有 `a41.js`、`b80.css`、`b111.jpg` | Dr.COM 门户静态资源命名规律 |
| 响应体是 `jsonpReturn({...})` | ePortal 4.x JSONP 接口 |
| `authloginport=801` / `authloginpath='/eportal/?c=ACSetting&a=Login'` | 老接口在 4.x 已 404，实际走 REST |

```bash
curl -s http://AC_IP/ | head -40          # 拿静态页与版本注释
curl -sI http://AC_IP/ | grep -i server
```

### 2.2 还原认证协议（4.x）

真实流程是 **两步 GET/JSONP**，且有几个致命细节：

#### 步骤 1：`custom/checkLogin`（登录前的绑定数检查）

```
GET http://AC_IP:801/eportal/portal/custom/checkLogin
    ?callback=dr1003
    &user_account=<纯账号，不带 ,1, 前缀>
    &user_password=<明文密码>
    &wlan_user_ip=<WAN IP，点分十进制>
    &wlan_user_mac=<WAN MAC，大写去冒号>
    &account_suffix=@cmcc
    &phone_flag=1
    &jsVersion=4.2.1
    &v=<4 位随机数>
    &lang=zh

→ dr1003({"result":1,"msg":"账号绑定数量小于配置数，直接登录！"})
```

#### 步骤 2：`login`（真正的认证）

```
GET http://AC_IP:801/eportal/portal/login
    ?callback=dr1005
    &login_method=1
    &user_account=,1,<账号>@<运营商>     ← 【关键】必须带 ,1, 前缀；, 与 @ 都要 URL 编码
    &user_password=<明文密码>
    &wlan_user_ip=<WAN IP，点分十进制>
    &wlan_user_ipv6=                    ← 留空！有 v6 会暴露设备
    &wlan_user_mac=<WAN MAC，大写去冒号>
    &wlan_ac_ip=<AC 侧 IP>
    &wlan_ac_name=<BRAS 名>
    &jsVersion=4.2.1                    ← 【关键】不是 4.X
    &login_t=1                          ← 1=PC，4=手机
    &js_status=0
    &is_page=1
    &is_page_new=<4 位随机数>
    &terminal_type=1                    ← 1=PC，2=手机，3/4=平板
    &lang=zh-cn
    &v=<4 位随机数>
    &lang=zh

→ dr1005({"result":1,"msg":"Portal协议认证成功！"})
```

#### 四个致命细节（缺一个就失败）

| # | 细节 | 缺了会怎样 |
|---|---|---|
| 1 | **`user_account` 必须带 `,1,` 前缀** —— 值形如 `,1,3236666666@cmcc`（整体 URL 编码成 `%2C1%2C3236666666%40cmcc`） | 「账号没有通过校验或校验已失效」 |
| 2 | **必须先调 `custom/checkLogin`** | 直接发 login → 同样报"校验失效" |
| 3 | **`jsVersion` 必须是 `4.2.1`** | 写 `4.X` 会失败 |
| 4 | **必须是 GET + `callback=<jsonp函数名>`**，不是 POST | POST 返回空体 |

> **`,1,` 前缀是最隐蔽的坑**：它是 Dr.COM 的账号格式标记（`,1,` = 普通认证）。
> 少了它，服务端会认为"这次请求没有对应的校验票据"，于是回你一句语法完全正确、但指向完全错误的
> 「账号没有通过校验或校验已失效」—— 很容易被误判成"账号被封"或"密码错了"，从而浪费大量时间。

#### 返回码与处理

| 返回 | 含义 | 处理 |
|---|---|---|
| `result:1` | 成功 | — |
| **`ret_code:2`（"终端IP已在线"）** | **其实已经在线**，不是失败 | **视为成功**，不要反复重登 |
| **146**（"网络故障 146"） | 后台监测账号状态异常，**固定封停 300 秒** | **静默 300 秒后重连**|
| "账号没有通过校验或校验已失效" | **参数格式不对**（缺 `,1,` 前缀 / 没先 checkLogin / jsVersion 错） | 回到上面四条逐一检查 |
| "用户名或密码错误" / "密码不正确" | 真凭据错误 | **立即停手**，别撞 5 次锁号 |
| 3 | 系统繁忙 | 退避重试 |
| 5 / 6 | REQ_CHALLENGE 失败 / 超时 | 网络类，快速重试 |
| 7 | Radius 认证失败 | 看 RADIUS 侧 |

> **`login_t` 与 `terminal_type` 必须和 UA 画像一致**：
> UA 是 Windows → `login_t=1` + `terminal_type=1`；
> UA 是 Android → `login_t=4` + `terminal_type=2`。
> 混用等于自报"我这一台设备同时是两种东西"。
>
> ⚠️ **AC 一般有"连续失败 5 次锁号"**。调试阶段务必用测试号，不要拿真账号反复试密码。
> 但要区分：**146 不消耗密码错误次数**，只有"密码错误"类才消耗。

### 2.3 会话状态探针

绝大多数 Dr.COM ePortal 部署都暴露了这个**无需认证**的接口：

```
GET http://AC_IP:801/eportal/portal/online_list
    ?user_account=&user_password=
    &wlan_user_mac=<WAN MAC>
    &wlan_user_ip=<IP 的十进制>
    &curr_user_ip=<同上>
    &jsVersion=4.2.1
```

返回你自己的完整会话记录。它解决了一个致命问题：**"我到底在线吗？AC 现在认为我是几台设备？"**

```bash
# IP → 十进制（注意 32 位溢出，用 %.0f）
ip2dec() { echo "$1" | awk -F. '{printf "%.0f",$1*16777216+$2*65536+$3*256+$4}'; }
```

> 这条接口是信息泄露，但它属于对端自己的设计缺陷，用它做健康探针是**只读**行为。请勿用它查询他人账号。

### 2.4 掉线原因判别

把 `online_session`（会话 ID）和 `user_account` 记下来，掉线时对比：

| 现象 | 结论 |
|---|---|
| 会话消失，重新登录成功，账号不变 | 正常超时/被踢，自动重认证即可 |
| 会话仍在但 `user_account` 变了 | **同一 IP/MAC 被别的账号顶掉**（宿舍多人多账号常见） |
| 在线但 HTTP 被 302/JS 跳转到认证页 | **被判定共享 / 被限制** ← 这才是真正的风控信号 |
| 会话消失且重登返回"校验已失效" | **先怀疑自己的登录参数格式**（缺 `,1,` 前缀 / 没先 `checkLogin` / jsVersion 不是 4.2.1）—— 见 2.2。**不是**"脏会话" |

### 2.5 （可选）服务端对账接口

部分部署暴露了 `http://AC_IP:9002/v6`（即 `ip6.dr.com`），返回类似：

```
s1=020;sec=4167;uf=316041;df=278960;
```

含义：会话状态 / **在线秒数** / **已用上行** / **已用下行**。
**用途**：拿它和路由器 `vnstat`/`/proc/net/dev` 的计数对账，看服务端记的流量和你实际跑差多少 ——
如果差值很大，说明服务端在按自己的口径记账（可能用于判定异常）。

### 2.6 抓一次真实登录 HAR（**最推荐的排障手段**）

> 这一节是踩坑踩出来的经验：为了搞清"为什么路由器登录总是被拒"，先后尝试了 **种种手段**
> **最后， 3 分钟解决的，是抓包 App 导出一份 HAR。**

做法：

1. 手机装 [ProxyPin](https://github.com/wanghongenpin/proxypin)（开源、能导出标准 HAR）
2. 开启抓包 → 手机浏览器打开 portal → **点注销** → **重新登录**（完整走一遍）
3. 导出 HAR，电脑上用几行脚本过滤：

```python
import json
har = json.load(open('export.har', encoding='utf-8'))
for e in har['log']['entries']:
    u = e['request']['url']
    if 'eportal' in u or '210.34' in u:
        print(e['request']['method'], u)
        print('  ->', (e['response'].get('content', {}) or {}).get('text', '')[:200])
```

**重点只看四件事**：

| 看什么 | 为什么 |
|---|---|
| `user_account` 的**准确值** | 有没有 `,1,` 前缀（本文最大的坑） |
| `jsVersion` 的**准确值** | 是 `4.2.1` 还是别的 |
| 有没有 `custom/checkLogin` **前置请求** | 决定要不要加这一步 |
| 请求是 **GET + callback** 还是 POST | 决定用什么方式发 |

> **不要只靠反编译 JS 猜参数**：JS 里有大量分支（`login_method=0/1/14`、`en_md5`、`enable_hotelop_login`、
> `custom.login_t` 等），静态分析很容易看错分支。**HAR 是"服务端实际收到的那一份"，不会骗人。**
>
> 附带：HAR 里还会暴露 **logout 用的是固定假账号**（`user_account=drcom&user_password=123&ac_logout=1`），
> 以及门户会先弹一个 `custom/checkLogin`（返回"账号绑定数量小于配置数，直接登录！"）——
> 这条消息本身就是"多设备绑定数"判定逻辑的直接证据。

---

## 3. 方案设计：五层架构

```
┌─ L1 单点认证 ─────────────────────────────────────────────┐
│ 守护进程周期查 online_list；掉线只做重认证                │
│ 错误分流：146 静默 300s / 凭据错立即停手 / 网络错快速重试 │
└───────────────────────────────────────────────────────────┘
┌─ L2 出口指纹归一 ─────────────────────────────────────────┐
│ TTL/HL 统一 · MSS clamp · QUIC 封杀 · 广播组播封杀        │
│ mDNS/SSDP/LLMNR/WS-D/NetBIOS/DHCP 封杀 · 单机连接数上限   │
│ portal 直连封堵（block_lan_portal=1）                     │
│ 挂在 firewall.user 上，fw3 reload 后自动重建              |
|  ⚠️ 只管 LAN，不管路由器本机 OUTPUT —— 见 §1.6           |
└───────────────────────────────────────────────────────────┘
┌─ L3 全 TCP 重发起（核心）─────────────────────────────────┐
│ PREROUTING -i br-lan → REDIRECT :端口 → 本地中转进程      │
│ 中转进程终结客户端 TCP，用自己的栈重开一条到真实目标      │
│   · 明文 HTTP：顺手改写 UA 为统一值                       │
│   · HTTPS / 其它 TCP：盲转发，TLS 端到端不变              │
│     bypass_ports 里绝对不能有 443                         │
│ ⚠️ 同样只管 LAN；路由器自身请求要单独补头 —— 见 §4.3.1   │
│ 链维护必须「幂等增量」而非「-F 重建」（否则周期性归一空窗）—— 见 §4.7 │
│ 效果：11 台设备在 AC 眼里 = 1 套 Linux 协议栈             │
└───────────────────────────────────────────────────────────┘
┌─ L4 画像一致性 ───────────────────────────────────────────┐
│ UA / mac_type / TTL 三者必须指向同一个画像（推荐 Windows）│
│ 认证请求与业务流量的 UA 必须一致                          │
└───────────────────────────────────────────────────────────┘
┌─ L5 攻击面收敛 ───────────────────────────────────────────┐
│ uhttpd/dropbear 只监听 LAN（默认 0.0.0.0 会被校园网扫到） │
│ 掉线路径上禁止任何网络变更（不 ifup wan、不重启网络）     │
└───────────────────────────────────────────────────────────┘
```

**为什么 L3 是关键**：L1/L2 只能骗过"看报文头"的检测；L3 让**所有出口连接都由路由器重新发起**，窗口大小、MSS、窗口缩放、TCP 选项顺序、IPID 序列一次性全部统一。

**为什么 L4 是关键**：L3 解决"看起来像一台机器"，L4 解决"这台机器前后是不是同一个东西"。

**为什么还要有 L4.5（路由器自身头归一）**：L1–L4 全部作用于 **LAN 侧流量**。而网关唯一的请求头解析点上，还稳定地站着**路由器自己每小时 500+ 次请求**。**这一层不做，前面四层做得再完整，网关看到的仍然是「同一 IP 上两种 UA + 多语言 + 多 Referer」。**

**代价**：所有 TCP 走一次用户态转发。实测 mt7620 上 1.8 MB/s → 2.0 MB/s，**无损**（Go 的 splice 直通）；128MB 内存占用可接受。

---

## 4. 落地：逐条命令

> 下面以 `WAN=eth0.2` 为例，`AC_IP` / `AC_WAN_IP` / `BRAS_NAME` 按自己抓包结果替换。

### 4.1 基线：MAC、网段、IPv6

1. **MAC 克隆**：WAN 口克隆"已被 AC 记录过的那台设备"的 MAC。
   - `uci set network.wan.macaddr='xx:xx:xx:xx:xx:xx'`
   - ⚠️ **画像必须一致**：克隆的是 PC 的 MAC，UA 和 TTL 就要按 PC 画像走。
2. **冷门 LAN 网段**：`10.77.0.1/24` 这类不常见的私有段，规避门户页 JS/WebRTC 对内网网段的探测。
3. **彻底关 IPv6**：

```bash
# 先判断到底有没有公网 v6 —— 有才是问题，只有 ULA 就没关系
ip -6 addr show scope global | grep -v 'inet6 fd'
```
- **只有 `fdxx:`（ULA）** → 不是问题，BRAS 没下发前缀（实测宿舍网就是这种）
- **出现 `2001:` / `240e:` 等** → **高危**，IPv6 无 NAT，11 台设备 MAC 全裸奔在 BRAS 邻居表里，L3 重建对它无效
  ```bash
  ip6tables -P FORWARD DROP
  echo 1 > /proc/sys/net/ipv6/conf/wan/disable_ipv6
  uci delete network.wan6; uci commit network
  ```

### 4.2 TTL / HL / MSS：选一个画像，三件套对齐

| 画像 | TTL/HL | UA | `mac_type` |
|---|---|---|---|
| **Windows 10/11 PC（推荐）** | `64` 或 `128` | `...Windows NT 10.0; Win64; x64...Chrome/xxx` | **1** |
| Android / Linux | `64` | `...Android 14; Pixel 7...Mobile` | 2/3 |

```bash
# 出方向统一 TTL（v4）与 HopLimit（v6）
iptables  -t mangle -A POSTROUTING -o $WAN -j TTL --ttl-set 64
ip6tables -t mangle -A POSTROUTING -o $WAN -j HL  --hl-set  64

# ICMP 必须跟 TCP 用同一个值
iptables -t mangle -A POSTROUTING -o $WAN -p icmp -j TTL --ttl-set 64

# MSS 钳制
iptables -t mangle -A POSTROUTING -o $WAN -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

> 一台手机不太可能跑 800+ 并发、开各种端口。所以从可能的行为层检测考虑，**推荐改成 Windows 桌面画像（`mac_type=1`）**。
> 改画像 = 换 MAC + 改 UA + 重登，三件套一起换。
> 具体选 PC 还是手机画像，取决于你 WAN 口所伪装的 MAC 原本是 PC 还是手机。

**关于 ICMP ID**：多台设备 ping 会各自生成 ICMP echo id，看似是"多源"特征。
但只要出方向有 `MASQUERADE`，**NAT 会自动改写 ICMP echo id**（与 TCP/UDP 端口同一套映射机制），
不需要额外规则。

### 4.3 UA 统一：先分清「LAN 侧」与「路由器自身」两个面

两条路线，二选一：

**A. UA3F（NFQUEUE，轻量）**

```bash
opkg update && opkg install ua3f
uci set ua3f.enabled.enabled='1'
uci set ua3f.custom_ua.custom='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36'
uci commit ua3f && /etc/init.d/ua3f restart
```

**B. [UAmask（笔者所用）](https://github.com/Zesuy/UA-Mask) / 其它 TPROXY 中转（可同时完成 L3）**

它能终结 TCP 并重写 UA，顺带把 HTTPS 盲转发，等于 4.7 和 4.3 一起做了。

**验证方法（必做）**：

```bash
curl -s http://httpbin.org/user-agent
# 期望：无论 Windows / iPhone / Android，返回的 user-agent 完全一致
# ⚠️ 这条命令必须在 LAN 设备上跑。在路由器上跑，拿到的是 wget/curl 自己的 UA —— 因为它走 OUTPUT，压根不进中继。 很多人在路由器上验证「UA 已统一」，实际证明的是另一个面。两个面要分开验证，路由器侧见 §4.3.1。
```

**同时验证认证请求的 UA 也一致**（很多人只改了流量 UA，没改认证脚本的）：

```bash
uci get campus.normalize.ua     # 认证脚本用的 UA
uci get UAmask.main.ua          # 中转用的 UA
# 两者必须完全相同，且与 mac_type 对应的画像匹配
```

### 4.3.1 路由器自身出向头归一

**问题见 §1.6。** 修法很小：给所有走 `OUTPUT` 的**探活/查询类**请求补上归一 UA。

**关键点：UA 值要动态读配置，不要硬编码** —— 否则以后改画像时，这两处会忘记同步，反而造出新的不一致。

### 4.3.2 Accept-Encoding / Accept-Language / Referer
UA 归一之后，这三个头仍然原样穿出。实测做法很简单：构造三组不同值和不同 Referer（captive.apple.com / msftconnecttest.com / 探测域名），各自跑一次看服务端收到什么 —— 全部原样，没有被改写。
**这很可能也是检测凭据之一，因为在笔者的实践中，在没有完成上述 4.3 路由器自身面 以及 4.3.2 三头归一之前会出现偶发性的 网络故障146**


### 4.4 封杀 QUIC（最容易被忽略的一环）

```bash
iptables -A FORWARD -o $WAN -p udp -m multiport --dports 443,80 -j DROP
```

副作用：视频站首帧略慢，属正常。

### 4.5 封堵共享热点特征

```bash
iptables -A FORWARD -o $WAN -m pkttype --pkt-type broadcast -j DROP
iptables -A FORWARD -o $WAN -m pkttype --pkt-type multicast -j DROP
iptables -A FORWARD -o $WAN -p udp -m multiport --dports 1900,5353,5355,3702 -j DROP
iptables -A FORWARD -o $WAN -p udp -m multiport --dports 137,138 -j DROP
iptables -A FORWARD -o $WAN -p tcp -m multiport --dports 139,445 -j DROP
iptables -A FORWARD -o $WAN -p udp -m multiport --dports 67,68 -j DROP

# DoT(853)：设备用加密 DNS 会绕过 4.6 的 53 端口接管，真实查询集合就暴露了
iptables -A FORWARD -o $WAN -p tcp --dport 853 -j DROP
iptables -A FORWARD -o $WAN -p udp --dport 853 -j DROP
```

> 常见误区：漏了 **LLMNR(5355)** 和 **WS-Discovery(3702)** —— Windows 网络上这两样出公网，等于自报"我家有台 Windows"。

### 4.6 DNS / NTP 接管 + 并发限制

```bash
iptables -t nat -A PREROUTING -p udp --dport 53  -j REDIRECT --to-ports 53
iptables -t nat -A PREROUTING -p tcp --dport 53  -j REDIRECT --to-ports 53
iptables -t nat -A PREROUTING -p udp --dport 123 -j REDIRECT --to-ports 123

# 单机并发上限
iptables -A FORWARD -i br-lan -p tcp -m connlimit --connlimit-above 800 --connlimit-mask 32 \
  -j REJECT --reject-with tcp-reset
# 整屋并发总上限（一台 PC 不该有上千条流）
iptables -A FORWARD -i br-lan -p tcp -m connlimit --connlimit-above 2000 --connlimit-mask 0 \
  -j REJECT --reject-with tcp-reset
```

### 4.6.1 抑制 AAAA 查询（避免"卡在 IPv6 上"）

如果 `FORWARD` 对 IPv6 是 DROP，设备拿到 AAAA 记录后仍会尝试连 v6，
连接被静默丢弃 → 表现为"某些网站特别慢/打不开"，而且这种"半死"状态本身不自然。

**dnsmasq-full 直接过滤**：

```bash
uci set dhcp.@dnsmasq[0].filter_aaaa='1'
uci commit dhcp && /etc/init.d/dnsmasq restart
```

### 4.6.2 新建连接速率闸门

> 它和"并发连接数"不是一回事：
> **并发**是某一时刻有多少条流；**速率**是每秒新建多少个连接。AC 看的是后者。

**实测依据**：从本机造成 1000 SYN/s 后，**约 2 分钟**账号被 146 强制注销；
而多设备网页+视频的稳态新建连接通常 **<40/s**。也就是说这条路径**与共享检测无关**，
但一样能把你踢下线，而且在做端口扫描/测速/开一堆标签页时最容易被触发。

**规则（`recent` 版，低端 mips 上比 hashlimit 省 CPU）**：

```sh
iptables -t mangle -N CAMPUS_RATE
iptables -t mangle -A CAMPUS_RATE -o "$WAN" -p tcp --tcp-flags SYN,RST,ACK SYN \
  -m recent --name WANRATE --rcheck --seconds 2 --hitcount 120 -j DROP
iptables -t mangle -A CAMPUS_RATE -o "$WAN" -p tcp --tcp-flags SYN,RST,ACK SYN \
  -m recent --name WANRATE --set -j RETURN
iptables -t mangle -I FORWARD 1 -j CAMPUS_RATE
```

--seconds 2 --hitcount 120 ≈ 60/s、burst 120。取值理由：

低于正常使用的突发（一次页面加载的并发 SYN 可到 80–120），所以不卡体验；
又远低于能触发风控的量级（1000/s）。

### 4.7 全 TCP 重发起（核心）

**原理**：`REDIRECT` 把 LAN 的 TCP 连接改投到本机端口，中转进程用 `SO_ORIGINAL_DST` 取回原始目标，再以自己的协议栈重新建连。

```bash
# 中转进程（示例：UAmask，REDIRECT/TPROXY 模式，监听 12032）
/usr/bin/UAmask -port 12032 -u "$UA" &

# 分流：校园网/内网/回环/DNS 直连，其余全部重发起
iptables -t nat -N CAMPUS_RELAY
iptables -t nat -A CAMPUS_RELAY -d $AC_IP -j RETURN
for n in 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16 127.0.0.0/8 169.254.0.0/16; do
  iptables -t nat -A CAMPUS_RELAY -d $n -j RETURN
done
iptables -t nat -A CAMPUS_RELAY -p tcp --dport 53 -j RETURN
iptables -t nat -A CAMPUS_RELAY -p tcp -j REDIRECT --to-ports 12032
iptables -t nat -I PREROUTING 1 -i br-lan -j CAMPUS_RELAY
```

**⚠️ 中转进程的 `bypass_ports` 里绝对不能有 443。**

UAmask 的 `bypass_ports` 默认可能是 `22 443`，意思是 **443 流量完全绕过中转**（不重建 TCP）。
效果是：你 11 台设备的 HTTPS 连接，**TCP 层指纹（窗口大小、选项顺序、TSval、ISN、SACK）全是原始设备的**，
TTL/MSS 被 mangle 统一了，但**窗口缩放因子和选项顺序 mangle 改不了**。

```bash
uci set UAmask.main.bypass_ports='22'     # 只保留 22（SSH），443 必须走中转
uci commit UAmask && /etc/init.d/UAmask restart
# 验证：443 的连接应该出现在 UAmask 的日志里
tail -5 /tmp/UAmask/UAmask.log
```

**必须配一个看门狗**：中转进程一挂，全屋断网。守护脚本里每轮检查端口，没在监听就立刻摘掉跳转：

```bash
if ! netstat -ltn | grep -q ':12032 '; then
  while iptables -t nat -D PREROUTING -i br-lan -j CAMPUS_RELAY 2>/dev/null; do :; done
fi
```

### 4.8 持久化：别让 fw3 把你的规则冲掉

```bash
iptables -t nat -C PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 53 2>/dev/null || \
  iptables -t nat -A PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 53

/usr/sbin/campus-normalize apply   # 其余规则封装成脚本，内部同样用 -C 去重
```

用 procd 起守护服务，开机自启：

```bash
/etc/init.d/campus enable
/etc/init.d/campus start
```

### 4.9 封堵 portal 直连

> 笔者的碎碎念：
> 
> 此前笔者发现，舍友的小米手机每次连上宿舍网络后的几分钟里，几乎总会报网络故障146。
> 
> 现在想来，可能是该手机系统在连接网络后会在后台自动打开 portal 页面，被上报从而被检测。
> 
> 封堵 protal 后遂解决。

#### 为什么要封

见 [1.5](#15-门户页与企业-app-的-js-探针)：portal 前端会主动探测你的私网网段（图片 `onload/onerror`）、用 WebRTC ICE 取内网 IP，
然后把「我在 NAT 后面」加密上报给服务器。

**这是整套检测里唯一"不需要 DPI、你也绕不过"的判据 —— 除非让设备根本没机会打开那个页面。**

#### 要封哪些 IP

| IP | 用途 | 必须封 |
|---|---|---|
| `210.34.84.127` | 认证网关主 IP（portal 页） | ✅ |
| **`210.34.81.164`** | **同一台机的第二 IP**（也能出 portal 页） | ✅ **最容易漏** |
| `210.34.81.50` | 自助服务 `/Self` | ❌ 无探测，可封可不封 |

#### ⚠️ 关键陷阱：封在 `FORWARD` 上对走中继的流量**完全无效**

我踩过的坑：
内网设备 → PREROUTING(nat) → CAMPUS_RELAY: REDIRECT → 12032 ↓ 目标变成 10.233.233.1:12032 ↓ 走 INPUT，不走 FORWARD ↓ CAMPUS_FILTER(FORWARD) 的 DROP 永远看不到

**`REDIRECT` 在 nat 表，比 FORWARD 早。** 所以只在 `CAMPUS_FILTER` 里加 `-d <IP> -j DROP`，
对"走中继的流量"**一条都拦不住**（计数器永远 0）。

**正确顺序：先 `RETURN` 出中继链，再在 `FORWARD` 上 `DROP`。**

#### 正确做法

```bash
AC_IP=210.34.84.127
BLOCK_LIST="210.34.84.127 210.34.81.164 210.34.81.50"

for ip in $BLOCK_LIST; do
  # 第一步：让它不进中继（否则永远到不了 FORWARD）
  iptables -t nat -C CAMPUS_RELAY -d "$ip" -j RETURN 2>/dev/null || \
    iptables -t nat -I CAMPUS_RELAY 1 -d "$ip" -j RETURN
  # 第二步：在 FORWARD 上真正丢弃
  iptables -C CAMPUS_FILTER -i br-lan -d "$ip" -j DROP 2>/dev/null || \
    iptables -A CAMPUS_FILTER -i br-lan -d "$ip" -j DROP
done
```

**验证（必须看计数器，不要只看规则存在）**

```bash
# 清零后等待，看 DROP 是否真的命中
iptables -Z CAMPUS_FILTER
sleep 60
iptables -L CAMPUS_FILTER -n -v | grep '210\.34'
# 期望：至少 <AC_IP> 和 <AC_IP_2> 两条的 pkts 在增长（.81.50 不需要封）

# 反向验证：conntrack 里不应再有「经 12032 去这些 IP」的连接
for ip in 210.34.81.50 210.34.81.164; do
  echo -n "$ip: "; grep "$ip" /proc/net/nf_conntrack | grep -c 'sport=12032'
done

# 期望：0
```

#### 副作用与取舍
内网设备打不开 portal / 自助服务页面了 → 需要登录/查状态时，在路由器上执行：

```bash
campus-auth status    # 看在线状态
campus-auth login     # 手动认证
campus-auth info      # 综合信息
```

---

## 5. 验证：怎么证明它真的生效

### 5.1 UA 一致性（三终端）

| 客户端实际发送 | 服务端实际收到 |
|---|---|
| Windows / Edg 124 | `Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...` |
| iPhone / Safari 17.4 | 同上 |
| Android Dalvik | 同上 |

三者一致 → UA 层通过。**并且确认 `uci get campus.normalize.ua` 也是同一个串**。

### 5.2 流量确实被重发起（conntrack 双向证据）

```bash
grep '<目标IP>' /proc/net/nf_conntrack
```

```
LAN→AC  : src=10.20.30.40    dst=203.0.113.10 dport=443     ← 客户端原始流
回程     : src=10.20.30.1     sport=12032                    ← 被 REDIRECT 到代理
代理出口 : src=198.51.100.5   dst=203.0.113.10 sport=43004   ← 路由器重开的新连接
```

出现第三行，说明重发起成功。**443 也要出现**（这是 4.7 修正的验证点）。

### 5.3 规则幂等

```bash
/etc/init.d/firewall reload; /etc/init.d/firewall reload
iptables -t mangle -S | grep -c ttl-set        # 期望 1
iptables -t nat -S PREROUTING | grep -c 'dport 53'   # 期望 2
```

### 5.4 会话表

```bash
campus-auth status      # online / offline
curl -s "http://AC_IP:801/eportal/portal/online_list?...&total"   # total=1
```

---

## 6. 自愈：掉线自动重认证
以下为运行在笔者路由器下自动重连脚本的设计思路：
### 6.1 核心原则

1. **掉线只做一件事：重认证。** 不要在掉线路径上 `ifup wan`、不要重启网络、不要改变 IP。
2. **146 是固定封停**，静默等够时间再重连。
3. **"账号没有通过校验或校验已失效" 不是密码错**，必须可重试。
4. **真凭据错误立即停手**（保护每日 5 次锁号额度）。
5. **永不永久放弃**：长时间故障后转成低频自愈（每 30 分钟一次）。

### 6.2 推荐参数

```bash
uci set campus.guard.interval='10'        # 检测间隔（秒）—— 越小恢复越快，根据路由器性能调整
uci set campus.guard.ban_wait='300'       # 146 封停静默期（实测固定 300 秒）
uci set campus.guard.fast_retry='2 5 10 20 30'   # 网络类失败的快重试序列
uci set campus.guard.cycle_wait='600'     # 快重试用尽后的慢速轮间隔
uci set campus.guard.max_cycles='8'       # 慢速轮轮数（8 × 10min ≈ 80min）
uci set campus.guard.idle_retry='1800'    # 之后每 30 分钟自愈一次
uci set campus.guard.auto_renew='1'       # 会话接近上限时主动续期
uci set campus.guard.renew_after='28800'  # 检测算法可能存在计数机制，实践中可在用网低谷期时或固定周期主动续期来规避检测
uci commit campus
```

### 6.3 状态机

```
每 INTERVAL(10s)：
├─ 中转看门狗（UAmask 挂了就摘 REDIRECT）
├─ 查 online_list
│
├─ 【online】
│    ├─ 状态跃迁 → 记日志（含恢复耗时/此前在线时长）
│    ├─ tl ≥ renew_after → 主动续期（冷却 30 分钟）
│    └─ health 检查
│
└─ 【offline】
     ├─ 凭据失效标记存在 → 不动（等人工）
     ├─ 146 静默期内 → 不动（不请求、不消耗次数）
     ├─ 退避期内 → 不动
     └─ 执行一次 login，按返回分流：
          login-ok / ret_code=2  → 恢复（ret_code=2 表示"终端IP已在线"，同样算成功）
          146                    → ban_until = now + 300
          凭据类错误              → 写 FLAG，停手
          网络类失败              → 2/5/10/20/30s 快重试
          快重试用尽              → 600s × 8 轮
          8 轮用尽                → 1800s 自愈（不放弃）
```

> **`ret_code=2` 一定要当成功处理**。实测中会出现"会话明明在线，但 `online_list` 短暂读不到"的情况，
> 这时 login 会返回 `{"result":0,"msg":"终端IP已在线","ret_code":2}`。
> 如果把它当失败，就会陷入"明明在线却反复重登"的循环。

### 6.4 时间线预期

| 场景 | 恢复耗时 |
|---|---|
| **自然掉线** | `10s 发现 → checkLogin → login →` **实测 15 秒恢复** |
| **146 封停** | `10s 发现 → 146 → 静默 300s → 立即重连`，总计约 **310~315s** |
| **网络故障** | 2/5/10/20/30s 快重试（67s），再 600s×8 轮，之后每 1800s 一次 |
| **密码真错** | 立即停手 + 提示人工处理 |

### 6.5 日志格式（便于观测）

```
2026-11-02 01:15:03  state -> offline (此前已在线 2h13m)
2026-11-02 01:15:13  login #1: login-ok xxx
2026-11-02 01:15:13  state -> online (恢复耗时 13s)

2026-11-02 03:40:02  state -> offline (此前已在线 2h25m)
2026-11-02 03:40:12  146 封停 -> 静默至 +300s（约 03:45:12）
2026-11-02 03:45:12  login #1: login-ok xxx
2026-11-02 03:45:12  state -> online (恢复耗时 310s)
```

**判断有没有效果就看两列**：`恢复耗时` 和 `此前已在线 xxx`（后者越来越长 = 越来越难被踢）。

### 6.6 一次真实的「掉线 → 自动恢复」验证记录

为了验证 v2 是否真的能自己把会话拿回来，做了一次受控实验：**手动注销 → 完全交给守护进程**。

```
=== 注销 ===
logout-1  Radius注销成功！

=== 等待 guard 自动重连 ===
  + 5s:  offline
  +10s:  offline
  +15s:  online  acct=3236666666@cmcc  tl=3  sid=16666     ← 自动重连成功

=== 最终 ===
NET_OK
```

对应日志：

```
09:10:00  logout: result=1 msg=Radius注销成功！
09:10:01  state -> offline (此前已在线 11s)
09:10:01  checkLogin: {"result":1,"msg":"账号绑定数量小于配置数，直接登录！"}
09:10:02  login #1: login-fail no-response        ← BRAS 还没放通，空响应
09:10:02  网络类失败，2s 后重试
09:10:13  login #2: login-fail no-response        ← 仍然空响应
09:10:13  网络类失败，5s 后重试
→ 约 09:10:15  state -> online（新会话 sid=16666，tl 从 0 开始）
```

**两个值得注意的点**：

1. **前两次 `no-response` 是正常现象** —— 注销后 BRAS 侧需要几秒才放通新认证，`wget` 拿不到响应。

2. **`checkLogin` 每次都成功** —— 说明登录参数（`,1,` 前缀、`jsVersion=4.2.1`、全套字段）已经被 AC 接受，
   剩下的只是等网络放通。

---
