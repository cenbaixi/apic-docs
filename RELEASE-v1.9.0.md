# APIC / forward-panel v1.9.0 —— 出口 IP 轮换 + 全链路整形

在 v1.8.0（壳 utls 化）之上,一批落地 **#2 出口 IP 轮换+限量、#3 壳整形、#4 壳 SNI 轮换+decoy、#5 quic-obfs 整形、Tier1 接入层入站连接寿命**。

> 配置全部走 **规则的 advanced JSON**(per-rule),除 #2 出口多 IP 还需 agent 装机参数。下面每节给可直接抄的配置。

---

## 0. 部署须知(先看)

- **重建范围**:本批改了 proto + kernel 共享层 → **master 和 agent 都要重建**。编译/签名在 us-lax02。
- **⚠ quic-obfs 滚动升级**:#5 改了 quic-obfs 的 wire 格式,**老 obfs ↔ 新 obfs 不兼容**。用了 `relay_mode=quic-obfs` 的那一跳,**入口和出口两端 agent 必须同时换上 v1.9.0**;升级窗口内该模式短暂断流(其它模式 aes/shell/tls/reality 不受影响)。
- **其余特性向后兼容**:#2/#3/#4/Tier1 不涉及跨端格式不兼容;不配相应 JSON 键时行为与 v1.8.0 完全一致(默认值见各节)。
- **默认开 vs 默认关**:#3 壳整形是**默认开、无开关**(填充量化+抖动+keepalive 一直生效);其余(#2 限量、#4 SNI 固定、#5 填充档、Tier1)都靠 JSON 键开启或调整,不配=保持默认。

---

## 1. #2 出口 IP 轮换 + 单 IP 限量(route2)

**目的**:对抗「单出口 IP 流量阈值封禁」(研究:伊朗约 100GB/48h 封 IP)。出口 node 挂多个 relay IP,master 按滚动窗口统计每个 IP 的总流量,超帽的 IP 不再分配、自动轮到没超帽的 IP。

### 1.1 出口 node 装机:挂多个 relay IP

agent 的 `-relay-addr` 改成**逗号分隔的多 IP**(每个 `IP:端口`,端口为 relay base):

```bash
# 单 IP(老写法,仍支持,不触发轮换/限量)
./agent ... -relay-addr 1.2.3.4:9100

# 多 IP(触发 route2)
./agent ... -relay-addr 1.2.3.4:9100,1.2.3.5:9100,1.2.3.6:9100
```

- 这些 IP 必须都绑在该出口机上(relay 监听本就 bind `*:端口`,全网卡 IP 可达)。
- 只有**配了多个 IP 的出口 node** 才会被限量过滤;单 IP 部署 100% 不变。

### 1.2 规则 advanced JSON:限量参数

```json
{
  "exit_ip_cap_gb": 100,
  "exit_ip_cap_window_h": 48,
  "exit_ip_cap_full": "rotate"
}
```

| 键 | 含义 | 默认 |
|---|---|---|
| `exit_ip_cap_gb` | 单 IP 在窗口内的流量帽(GB,上下行合计);0/缺省=不限 | 0(不限) |
| `exit_ip_cap_window_h` | 滚动统计窗口(小时),最大 168 | 48 |
| `exit_ip_cap_full` | 所有 IP 都超帽时:`rotate`=软限,仍全部参与(保可用);`stop`=硬断,空池拒绝出网 | `rotate` |

**语义要点**:
- 帽算的是**单 IP 跨所有规则的总流量**(上行+下行),滚动小时桶。
- master 每 **60s** 重算一次超帽集合(`capRefreshLoop`),即时把刚超帽的 IP 踢出、把流量降下来的 IP 放回。
- 每条客户连接本就轮询出口 IP 池(round-robin),所以轮换是天然的;壳握手种子按 node_id 不按 IP,换 IP 不破握手。
- `stop` 档要谨慎:全超帽时直接拒绝出网(不会裸奔直连),适合「宁可断不可裸」的强隐蔽场景。

**残留**:换 IP 解不了「IP↔ASN 深层不一致」(数据中心 IP 声称大站 SNI)。最敌对的跨墙跳优先用 `quic-obfs`(线上无 SNI)或 `reality`(借真站)。

---

## 2. #3 壳整形(默认开)

**仅作用自研壳(`relay_mode=shell`)**,tls/reality/quic-obfs 不受影响。三件套默认全开、无需配置:

1. **填充量化**:每条壳记录填到约 1400 字节(默认),抹掉「小包尺寸」特征。
2. **突发抖动**:距上次写超过 30ms 才插入 0–12ms 抖动 —— 只动交互式的「思考间隔」,背靠背的批量传输零延迟,**不伤吞吐**。
3. **抖动 keepalive + 随机会话寿命**:空闲 15–25s 发 0 长帧保活;可选强制重连(见下)。

### 可选 JSON

```json
{
  "shell_pad": "256",
  "shell_session_max_s": 1800
}
```

| 键 | 含义 | 默认 |
|---|---|---|
| `shell_pad` | 填充档:缺省/`""`=填到 ~1400(均一);`"256"`=量化到 256 边界(更细、省流量) | ~1400 |
| `shell_session_max_s` | 壳会话最大寿命(秒);>0 时到点(随机 `[半,全]`)关连接逼客户端重连(换出口 IP);0=不限 | 0(不限) |

**⚠ 方向不对称**:`shell_pad` **只影响 ingress→exit 方向**。出口壳服务端是跨规则共用的,拿不到 per-rule 档,固定走默认。回程(exit→ingress)按默认填充。这是设计如此,不是 bug。

---

## 3. #4 壳 SNI 轮换 + decoy(默认开)

**仅作用自研壳**。

- **SNI 每连接轮换**(默认):每条壳连接从内置的 7 个国内可达 CDN 站点里随机挑 SNI,抹掉「固定 (IP, SNI) 配对」。服务端不校验 SNI(鉴权靠 session_id 里的 PSK tag),所以轮换不影响握手。
- **decoy 主动探测防御**(默认,本版接线):壳鉴权失败时,不再突兀关连接(那是强主动探测信号),而是解析对方 ClientHello 的 SNI、拨内置池里对应站点的 :443、回放原始 ClientHello —— 看起来就像个真服务端在 front CDN。decoy 目标限内置池(防 SSRF)。

### 可选 JSON

```json
{ "shell_sni": "www.example.cn" }
```

| 键 | 含义 | 默认 |
|---|---|---|
| `shell_sni` | 固定壳伪装 SNI(不轮换);设成**该出口 IP 实际能对上的站点**更隐蔽 | 空=每连接轮换 |

**残留**:SNI 轮换去掉了静态配对,但解不了「数据中心 IP 声称大站」的 IP↔ASN 不一致。要彻底对上,用 `reality`(真域前置)。

---

## 4. #5 quic-obfs datagram 整形

**仅作用 `relay_mode=quic-obfs`**。详见 §0 的滚动升级警告。

- **填充**:每个 UDP 报按档量化,**封顶 1280 字节**(IPv6 最小 MTU,任意路径不分片 —— UDP 分片不可靠且自成指纹)。
- **噪声注入**:入口侧每 8–24s 抖动发一个随机大小的噪声报(出口侧多 peer 不注入),遮蔽空闲、打散包数/时序。噪声对 QUIC 透明(收端按类型丢弃)。

### 可选 JSON

```json
{ "quic_obfs_pad": "mtu" }
```

| 键 | 含义 | 默认 |
|---|---|---|
| `quic_obfs_pad` | 填充档:缺省/`""`/`"256"`=量化到 256 边界;`"mtu"`=填到 1280(更均一、更费流量) | 256 边界 |

**⚠ 方向不对称**:同 `shell_pad`,`quic_obfs_pad` 只影响 ingress→exit 方向,出口固定默认档。

**用法提示**:quic-obfs 是**唯一线上无明文 SNI** 的形态(整包加扰),最敌对的跨墙跳优先用它而非 TLS 形态档。高丢包链路 QUIC 也优于 TCP(避免 TCP-over-TCP)。

---

## 5. Tier1 接入层入站连接寿命

**作用在接入层入站**(客户端→接入节点),forward 规则和单端口(SNI 路由)规则两条路径都覆盖。

**目的**:接入 IP 资源稀缺、**不可轮换**时的力所能及保护。傲盾类设备对单个固定入站 IP 取证,最强信号是「超长、持续活跃」的连接。到点主动关客户 TCP、逼其重连,就把单连接的寿命特征打散了。

> 这是**延缓取证、增加判定难度**,不是根治(入站 IP 没换,协议指纹仍是客户自己的)。面向客户的入站口是协议无关的,探测响应/并发等更深的特征够不着。

### JSON

```json
{ "ingress_conn_max_s": 600 }
```

| 键 | 含义 | 默认 |
|---|---|---|
| `ingress_conn_max_s` | 客户入站 TCP 最大寿命(秒);>0 时到随机时长 `[半,全]` 后主动关连接;0=不限 | 0(不限) |

**副作用**:客户端会按这个节奏重连。设太短会频繁断流影响体验,建议 ≥300s。客户端协议(vless/trojan 等)一般都能自动重连,但极少数不重连的客户端会受影响 —— 上线前在 test2 验证你的客户端行为。

---

## 6. 配置速查表(全部 advanced JSON 键)

把需要的键合进同一个规则 advanced JSON 即可。不写的键=默认。

```json
{
  "exit_ip_cap_gb": 100,
  "exit_ip_cap_window_h": 48,
  "exit_ip_cap_full": "rotate",
  "shell_pad": "256",
  "shell_session_max_s": 1800,
  "shell_sni": "",
  "quic_obfs_pad": "mtu",
  "ingress_conn_max_s": 600
}
```

| 键 | 作用形态 | 方向 | 默认 |
|---|---|---|---|
| `exit_ip_cap_gb` / `_window_h` / `_full` | 全形态(出口多 IP 才生效) | 出口 | 不限 |
| `shell_pad` | shell | ingress→exit | ~1400 |
| `shell_session_max_s` | shell | 会话级 | 不限 |
| `shell_sni` | shell | —— | 轮换 |
| `quic_obfs_pad` | quic-obfs | ingress→exit | 256 边界 |
| `ingress_conn_max_s` | 全形态 | 接入层入站 | 不限 |

---

## 7. 验证清单(test2)

1. **基线**:不配任何新键,各形态(aes/shell/tls/reality/quic-obfs)跑通 = 与 v1.8.0 等价。
2. **#2**:出口装多 IP,配低 `exit_ip_cap_gb`(如 1),压流量看是否轮换/`stop` 是否断;`capRefreshLoop` 60s 内放回。
3. **#3**:壳跑通;`shell_session_max_s` 配短(如 60)看客户端是否按期重连。
4. **#4**:tcpdump :801 看每连接 SNI 是否变;故意发坏鉴权看是否回 decoy(到 CDN 的 TLS)而非立即断。
5. **#5**:quic-obfs **两端都换 v1.9.0** 后跑通;tcpdump 看 UDP 报大小是否量化、是否有噪声报。
6. **Tier1**:配 `ingress_conn_max_s` 短(如 60),看客户连接是否到点断、客户端是否自动重连。
