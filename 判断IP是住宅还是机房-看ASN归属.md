---
title: "用一条命令判断代理 IP 是「住宅」还是「机房」——从 ASN 归属看本质"
description: "拿到一个代理 IP，怎么判断它是不是「住宅」？很多人第一反应是跑个 IP 查询网站看它标的类型。但那些网站的类型标注来自第三方数据库，不同的库结论还会打架。"
tags: ["ASN", "住宅IP", "机房IP", "命令行"]
source: https://www.cnblogs.com/socks5ip/p/23005035.html
license: CC-BY-4.0
---

拿到一个代理 IP，怎么判断它是不是「住宅」？很多人第一反应是跑个 IP 查询网站看它标的类型。但那些网站的类型标注来自第三方数据库，**不同的库结论还会打架**。

更可靠的做法是看一个底层事实：**这个 IP 属于哪个 ASN（自治系统）**。

## 一、为什么 ASN 比「IP 类型标注」更可信

ASN 是 IP 地址的归属单位。一个 IP 的 ASN 只有两种可能：

- 属于**本地宽带运营商**（电信/联通/移动在各省的宽带网段）→ 这是真住宅
- 属于**数据中心 / 云厂商**（阿里云、腾讯云、AWS、IDC 服务商）→ 这是机房 IP

第三方「IP 类型数据库」本质上是在猜，而 **ASN 归属是注册事实**，不会骗人。风控系统判断一个 IP 是不是代理，第一层看的通常就是这个。

## 二、一条命令拿到 ASN

如果你手边有 Node.js（>=18），用下面这个零依赖的方式（不用装包）：

```bash
# 三、用 ip-api.com 的公开接口（免费版限 45 次/分钟）
curl -s "http://ip-api.com/json/8.8.8.8?fields=status,country,isp,org,as,asname,proxy,hosting,mobile"
```

返回：

```json
{
 "status": "success",
 "country": "United States",
 "isp": "Google LLC",
 "as": "AS15169 Google LLC",
 "asname": "GOOGLE",
 "proxy": false,
 "hosting": true,
 "mobile": false
}
```

关键看两个布尔字段：

| 字段 | 含义 | 判读 |
|---|---|---|
| `hosting` | 是否数据中心/托管段 | `true` → **机房 IP** |
| `proxy` | 是否已知代理/VPN 段 | `true` → 已被标记 |
| `mobile` | 是否移动网络 | `true` → 移动蜂窝，与家宽不同 |

## 四、封装成一个可以批量跑的小工具

单个查没意思，做采集/多账号的一般要一次查几十上百个。我写了个零依赖的小包，可以直接用：

```bash
npm install -g proxy-ip-check

# 五、单个
proxy-ip-check 8.8.8.8

# 六、批量
proxy-ip-check 1.1.1.1 8.8.8.8 114.114.114.114

# 七、JSON 输出（方便进脚本/管道）
proxy-ip-check 8.8.8.8 --json
```

输出示例：

```
8.8.8.8
 ASN AS15169
 AS name GOOGLE
 ISP Google LLC
 Country United States
 Type hosting / datacenter
 Source ip-api.com
```

源码在 https://github.com/socks5ip/proxy-ip-check ，MIT 许可，只有 200 多行，主源是 `ip-api.com`，失败会自动回退到 `ipinfo.io`。

## 八、批量检测时的三个坑

**1. 限速。** `ip-api.com` 免费版 45 次/分钟。批量跑记得加间隔，或者自己换自建库（MaxMind GeoLite2 ASN 免费）。

**2. `hosting=true` 不等于「不能用」。** 机房 IP 便宜、带宽大，做纯采集反而更划算；只有做账号类业务时才必须避开。

**3. 住宅 IP 也可能被标脏。** ASN 是家宽不代表这个 IP 没被滥用过 —— 还要看黑名单。所以完整判断是**两层**：先看 ASN 归属，再看 IP 的滥用记录。

## 九、完整判断清单（四项）

按重要性排：

1. **ASN 归属** —— 家宽 or 机房（最核心，本文讲的就是这个）
2. **IP 类型数据库标注** —— 多家库交叉看，别只信一家
3. **黑名单 / 风险评分** —— 是否在滥用/欺诈库里
4. **DNS / WebRTC 是否泄漏** —— 你用的是代理，但测试时暴露了真实出口 IP

四项都过，才算「干净」。

## 十、参考资料

- `proxy-ip-check`（本文用到的工具，开源、零依赖）：https://github.com/socks5ip/proxy-ip-check
- MaxMind GeoLite2 ASN 数据库（可自建离线查询）：https://dev.maxmind.com/geoip/geolite2-free-geolocation-data
- 代理IP 价格数据集（18 家平台，含协议/覆盖/官方入口，CC0）：https://github.com/socks5ip/proxy-ip-pricing

---

*本文提及的工具与数据集均为开源（MIT / CC0），可自由使用。*
- 相关链接：http://ip-api.com/json/8.8.8.8?fields=status,country,isp,org,as,asname,proxy,hosting,mobile

---

> 本文来自 [全网低价IP](https://socks5ip.com.cn)（代理IP服务商聚合比价平台）。
> 各平台定价与服务以官方页面为准；最后更新：2026-09-17。
