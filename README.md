# 校园网 Dr.COM / ePortal 多设备共享绕过实战指南（2026/11/02 重大修订）

基于 OpenWrt 的校园网 **Web 版 Dr.COM / 哆点 ePortal** 防共享检测绕过，**以福建农林大学仓山校区宿舍校园网为例**。

测试环境：
- 福建农林大学仓山校区 / 中国移动校园宽带 / Dr.COM ePortal 4.x
- 路由器：Xiaomi MiWiFi Mini（mt7620 / 128MB） / OpenWrt R24.8.1 / kernel 5.4.278
- 内网 11 台终端（Windows / iOS / Android / HarmonyOS 混合）

测试结果：**稳定、永续在线，且在其他路由器设备、校园网账号下可复现。**

本教程侧重于原理剖析，而非无脑地教你执行，所以适合有一定技术基础的人阅读，你应至少同时具备：
- OpenWrt / Linux 基础
- TCP/IP 与抓包能力
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
**经过了 一些 深入探索与研究 ，发现：**
  1. **检测不在协议栈层。** TLS/JA3/HTTP2 指纹在这条链路上**不存在**—— 数据面在华为 BRAS，Dr.COM 只是 RADIUS + Portal 服务器，没有用户流量 DPI 能力。
  2. **检测在 portal 自己上报的字段里**：`mac_type`（由 UA 算出的终端类型）、`wlan_user_ipv6`、计费流量 `42/43+52/53`（持续监测各项流量特征是否正常）。这三样是真正决定你会不会被踢的东西。

---

## 1. 对端到底在看什么

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

**✅ 已证实 / ❌ 已证伪 / ⚪ 需自行复测**：

| # | 维度 | 状态 | 依据 |
|---|---|---|---|
| 1 | **`mac_type` 终端类型** | ✅ **生效** | portal `a41.js` 的 `util.getTermType()` 按 UA 算 0/1/2/3/4 并随请求上报|
| 2 | **IPv6 地址** | ✅ **生效** | portal 上报 `wlan_user_ipv6`，且有 `ip6.dr.com:9002/v6` 通道；**IPv6 无 NAT，设备 MAC 全裸奔** |
| 3 | **计费流量 42/43 + 52/53** | ✅ **生效** | RADIUS 全量累计、Interim 级实时上报（流量数据确实被实时上报）|
| 4 | IPv4 TTL | ✅ 生效 | TTL 归一即可 |
| 5 | IPID / TCP 选项 / 窗口 / TSval | ✅ 生效 | L3 重建 TCP 后由路由器内核生成 |
| 6 | 明文 HTTP 的 UA / `sec-ch-ua` | ✅ 生效 | portal 会上传明文 HTTP |
| 7 | **TLS/JA3/JA4、HTTP/2 指纹** | ❌ **证伪** | 经过深入探索，福建农林大学仓山校区校园网检测无此项 |
| 8 | QUIC / HTTP3 | ✅ 生效 | UDP 443 上有独立指纹 |
| 9 | 广播/组播（SSDP/mDNS/LLMNR/WS-D/NetBIOS/DHCP） | ✅ 生效 | 出公网 = "家里不止一台设备"的直接证据 |
| 10 | 单 IP 并发连接数 | ✅ 生效 | 上千条流 = 典型 NAT 特征 |
| 11 | 主动探测 / ARP 扫描 | ❌ 证伪 | 经过深入探索，福建农林大学仓山校区校园网检测无此项 |
| 12 | DNS/NTP 侧信道 | ⚪ 未验证 | 建议仍接管 |

**一句话**：真正难搞的是 1、2、3 三条 —— 它们**不在报文层**，而在 **portal 业务层**。

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
> 但实际跑着 11 台设备、400+ 并发、开各种端口 —— **画像自相矛盾**，这就是被踢的重要信号之一。
>
> **正确做法**：统一成 **桌面 Windows 画像**（`mac_type=1`）

### 1.4 一句话结论

> **TTL 只是入场券，TCP 重建是基本操作，真正决定生死的是「你把自己伪装成什么终端」以及「你在整个过程里有没有暴露你在“说谎”」。**

### 1.5 门户页与企业 App 的 JS 探针

发现了两处很聪明的设计，属于非报文层的旁路探测，必须单独防：
1. **门户页会尝试拿内网 IP。** JS 直接读不到，但 **WebRTC** 能把真实局域网地址暴露出来。
   - 处置：LAN 用冷门私有段（如 `10.77.0.1/24`）；浏览器装 WebRTC Leak Shield 或禁用 WebRTC。
2. **门户页会上报 UA，并算成 `mac_type` 与账号画像比对。**
   - 处置：**先让 UA 归一上线，再用浏览器访问门户/自助服务**；顺序反了，你自己就是那个"异常终端"。
   - **更高优先级的处置**：用 `block_lan_portal=1` **彻底禁止内网设备直连 portal**（见 4.9），让所有 portal 流量只从路由器发出 —— 这样设备 UA 根本不会暴露。
3. **学校自有 App（如"数字FAFU"）会扫局域网拓扑并向服务器上报。**
   - 处置：**用流量打开，别连 WiFi**。

> 一句话：报文层做得再干净，只要你自己用浏览器裸奔去点门户页，等于自首。

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

```
POST http://AC_IP:801/eportal/portal/login
Content-Type: application/x-www-form-urlencoded

login_method=1
user_account=<账号><@运营商后缀>
user_password=<明文密码>
wlan_user_ip=<WAN IP>
wlan_user_ipv6=            ← 留空！有 v6 会暴露设备
wlan_user_mac=<WAN MAC，大写去冒号>
wlan_ac_ip=<AC 侧 IP>
wlan_ac_name=<BRAS 名>
jsVersion=4.X
```

> **注意**：路由器脚本发的这条请求**不带 `mac_type`**（只有浏览器通过 portal 页登录时才带）。
> 所以"路由器认证"这条路径本身不暴露终端类型 —— 暴露的是**内网设备自己去访问 portal** 的时候。

返回 `jsonpReturn({"result":1,...})` 即成功；失败时 `ret_code` 含义：

| ret_code / msg | 含义 | 处理 |
|---|---|---|
| `result:1` | 成功 | — |
| **146**（"网络故障 146"） | 后台监测账号状态异常，**固定封停 300 秒** | **静默 300 秒后重连**（不要提前重试） |
| **"账号没有通过校验或校验已失效"** | **会话状态不一致** | **可重试** |
| "用户名或密码错误" / "密码不正确" | 真凭据错误 | **立即停手**，别撞 5 次锁号 |
| 2 | 终端 IP 已经在线 | 等会话清理 |
| 3 | 系统繁忙 | 退避重试 |
| 5 / 6 | REQ_CHALLENGE 失败 / 超时 | 网络类，快速重试 |
| 7 | Radius 认证失败 | 看 RADIUS 侧 |

> ⚠️ **AC 一般有"连续失败 5 次锁号"**。调试阶段务必用错号试探，不要拿真账号反复试密码。
> 但要区分：**146 不消耗密码错误次数**，只有"密码错误"类才消耗。

### 2.3 会话状态探针（本文最有用的一招）

绝大多数 Dr.COM ePortal 部署都暴露了这个**无需认证**的接口：

```
GET http://AC_IP:801/eportal/portal/online_list
    ?user_account=&user_password=
    &wlan_user_mac=<WAN MAC>
    &wlan_user_ip=<IP 的十进制>
    &curr_user_ip=<同上>
    &jsVersion=4.X
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
| 会话消失且重登返回"校验已失效" | **AC 侧脏会话未清理** |

### 2.5 （可选）服务端对账接口

部分部署暴露了 `http://AC_IP:9002/v6`（即 `ip6.dr.com`），返回类似：

```
s1=020;sec=4167;uf=316041;df=278960;
```

含义：会话状态 / **在线秒数** / **已用上行** / **已用下行**。
**用途**：拿它和路由器 `vnstat`/`/proc/net/dev` 的计数对账，看服务端记的流量和你实际跑差多少 ——
如果差值很大，说明服务端在按自己的口径记账（可能用于判定异常）。

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
│ 挂在 firewall.user 上，fw3 reload 后自动重建              │
└───────────────────────────────────────────────────────────┘
┌─ L3 全 TCP 重发起（核心）─────────────────────────────────┐
│ PREROUTING -i br-lan → REDIRECT :端口 → 本地中转进程      │
│ 中转进程终结客户端 TCP，用自己的栈重开一条到真实目标      │
│   · 明文 HTTP：顺手改写 UA 为统一值                       │
│   · HTTPS / 其它 TCP：盲转发，TLS 端到端不变              │
│     bypass_ports 里绝对不能有 443                         │
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

> 一台手机不太可能跑 400+ 并发、开各种端口。**推荐改成 Windows 桌面画像（`mac_type=1`）**。
> 改画像 = 换 MAC + 改 UA + 重登，三件套一起换。

### 4.3 UA 统一：装完必须验证

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
```

**同时验证认证请求的 UA 也一致**（很多人只改了流量 UA，没改认证脚本的）：

```bash
uci get campus.normalize.ua     # 认证脚本用的 UA
uci get UAmask.main.ua          # 中转用的 UA
# 两者必须完全相同，且与 mac_type 对应的画像匹配
```

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
```

> 常见误区：漏了 **LLMNR(5355)** 和 **WS-Discovery(3702)** —— Windows 网络上这两样出公网，等于自报"我家有台 Windows"。

### 4.6 DNS / NTP 接管 + 并发限制

```bash
iptables -t nat -A PREROUTING -p udp --dport 53  -j REDIRECT --to-ports 53
iptables -t nat -A PREROUTING -p tcp --dport 53  -j REDIRECT --to-ports 53
iptables -t nat -A PREROUTING -p udp --dport 123 -j REDIRECT --to-ports 123

# 单机并发上限
iptables -A FORWARD -i br-lan -p tcp -m connlimit --connlimit-above 400 --connlimit-mask 32 \
  -j REJECT --reject-with tcp-reset
# 整屋并发总上限（一台 PC 不该有上千条流）
iptables -A FORWARD -i br-lan -p tcp -m connlimit --connlimit-above 1000 --connlimit-mask 0 \
  -j REJECT --reject-with tcp-reset
```

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

> 此前笔者发现，舍友的小米手机每次连上宿舍网络后的一两分钟里，几乎总会报网络故障146。
> 可能是该手机在后台自动打开了 portal 页面，其 UA 被上报从而被检测。
> 封堵 protal 后遂解决。

**问题**：`CAMPUS_RELAY` 里通常有一条 `-d $AC_IP -j RETURN`（让 portal 流量直通、不中转）。
后果是：**内网设备自己去打开 portal 页面时，UA 是它自己的原始 UA**，完全绕过你的 UA 统一。

**解法**：直接禁止内网设备访问 AC：

```bash
iptables -A CAMPUS_FILTER -i br-lan -d $AC_IP -j DROP
```

或者用 `campus-normalize` 的配置项：

```bash
uci set campus.normalize.block_lan_portal='1'
uci commit campus && /usr/sbin/campus-normalize apply
```

**副作用**：内网设备打不开 portal 页面了。需要手动登录/查状态时，**在路由器上执行**：

```bash
campus-auth status    # 看在线状态
campus-auth login     # 手动认证
campus-auth info      # 综合信息
```

> 这一条把"设备 UA 暴露"这个问题从根上砍掉了 —— 因为设备根本没有机会访问 portal。

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
uci set campus.guard.interval='10'        # 检测间隔（秒）—— 越小恢复越快，10s 是 低端cpu 可接受的下限（根据路由器性能调整）
uci set campus.guard.ban_wait='300'       # 146 封停静默期（实测固定 300 秒）
uci set campus.guard.fast_retry='2 5 10 20 30'   # 网络类失败的快重试序列
uci set campus.guard.cycle_wait='600'     # 快重试用尽后的慢速轮间隔
uci set campus.guard.max_cycles='8'       # 慢速轮轮数（8 × 10min ≈ 80min）
uci set campus.guard.idle_retry='1800'    # 之后每 30 分钟自愈一次
uci set campus.guard.auto_renew='1'       # 会话接近上限时主动续期
uci set campus.guard.renew_after='62520'  # 实测 AC 会话上限约 17h37m，提前 15 分钟续
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
          login-ok            → 恢复（自然掉线约 12~15 秒）
          146                 → ban_until = now + 300
          凭据类错误           → 写 FLAG，停手
          网络类失败           → 2/5/10/20/30s 快重试
          快重试用尽           → 600s × 8 轮
          8 轮用尽             → 1800s 自愈（不放弃）
```

### 6.4 时间线预期

| 场景 | 恢复耗时 |
|---|---|
| **自然掉线** | `10s 发现 → 立即登录 → 约 12~15s 恢复` |
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

---
