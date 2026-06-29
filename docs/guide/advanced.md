# 参数说明（高级 JSON）

规则的「高级 JSON」是一段填进规则编辑器高级配置框的 JSON，面板下发前用它覆盖/设置规则字段。本页列**全部可配的键**。

!!! tip "怎么用"
    不写的键 = 用默认。把需要的键合进同一个 JSON 即可。**最关键的是 `relay_mode`**（中转外层）——它决定了壳整形（`shell_*`）、quic-obfs 整形（`quic_obfs_pad`）、TLS SNI（`tls_sni`）分别在哪种形态下才生效。先定 `relay_mode`，再配对应整形参数。

直接想抄配置的，跳到本页底部的 [按场景推荐配置](#按场景推荐配置)。

---

## 中转外层伪装

「入口 → 出口跳板」这段隧道外层穿什么衣服。隧道**内层恒为 AES-AEAD**，这里只选外层伪装。

### `relay_mode` —— 外层传输（最重要）

| | |
| --- | --- |
| 类型 | 字符串 |
| 取值 | `""`/`aes`（裸 AES）· `shell`（自研壳）· `tls`（TLS 伪装）· `reality`（VLESS+REALITY 借真站）· `quic-obfs`（Salamander 混淆 QUIC） |
| 默认 | `aes` |
| 端口 | 出口跳板按档分端口：aes=base · shell=base+1 · tls=base+2 · reality=base+3 · quic-obfs=base+5 |

各档怎么选见 [协议说明 · 中转外层](protocols.md#二中转外层协议层)。

### `tls_sni` —— TLS / quic-obfs 伪装 SNI

| | |
| --- | --- |
| 类型 | 字符串 |
| 作用 | `relay_mode=tls` 的伪装 SNI；`quic-obfs` 的内层 QUIC TLS SNI |
| 默认 | 空 = `www.microsoft.com` |

客户端用它作 ClientHello 的 ServerName；tls 档服务端按收到的 SNI 现签 CN 匹配的自签证书。

### `decoy` —— 壳鉴权失败诱饵站 / REALITY dest

| | |
| --- | --- |
| 类型 | 字符串（`host` 或 `host:port`） |
| 作用 | ① 自研壳前置认证失败时回放到的诱饵站（空 = 伪装 SNI 的 :443）；② `reality` 复用此字段为借证书的真站 dest |
| 默认 | 空 |

!!! note "REALITY 必填真站"
    reality 档**必填**一个真实大站（如 `www.apple.com:443`）。壳档可留空。多站轮换可逗号分隔，但每个站都要满足「SNI↔出口 IP 一致性」——随意混入不相关大站反而暴露。

### `tls_vers` —— Trojan 接受的 TLS 版本

| | |
| --- | --- |
| 类型 | 字符串（CSV：`1.3` · `1.2` · `1.2,1.3`） |
| 作用 | 仅 Trojan-over-TLS 档（入站协议 = trojan） |

---

## 出口 IP 轮换 / 按量限量 { #exit-ip-cap }

对抗「单出口 IP 流量阈值封禁」。

!!! warning "前提"
    出口节点装机时要配**多个** relay IP（`-relay-addr 1.1.1.1:9100,2.2.2.2:9100,...`），否则限量不触发（单 IP 没法轮换）。单 IP 部署这些键无副作用、不生效。

### `exit_ip_cap_gb`

| | |
| --- | --- |
| 类型 | 整数（GB） |
| 作用 | 单个出口 relay IP 在窗口内的流量上限（**上下行合计、跨所有规则**） |
| 默认 | `0` = 不限量（仅轮换） |
| 语义 | 超帽的 IP 被踢出出口池；面板每 60s 重算一次（刚超帽踢出、降下来放回） |

### `exit_ip_cap_window_h`

| | |
| --- | --- |
| 类型 | 整数（小时） |
| 作用 | 滚动统计窗口 |
| 默认 | `48`（最大 168） |

### `exit_ip_cap_full`

| | |
| --- | --- |
| 类型 | 字符串 |
| 取值 | `""`/`rotate`（软限：全超帽时仍全部参与，保可用）· `stop`（硬断：全超帽时空池、拒绝出网） |
| 默认 | `rotate` |

!!! danger "stop 会直接拒绝出网"
    `stop` 在全部 IP 超帽时**拒绝出网，不裸奔直连**。适合「宁可断不可裸」的强隐蔽场景。

---

## 自研壳整形

仅 `relay_mode=shell` 生效。核心整形（填充量化 + 写入抖动 + keepalive）**默认开、无开关**；以下为可选项。tls/reality/quic-obfs 不受这些键影响。

### `shell_pad` —— 壳填充档

| | |
| --- | --- |
| 类型 | 字符串 |
| 取值 | `""`/`mtu`（激进：每帧填到 ~1.4KB，默认）· `256`（温和：在途向上取整 256B，省流量） |
| 默认 | `mtu` |
| 方向 | **仅入口→出口**（出口壳服务端跨规则共用，固定默认档） |

### `shell_session_max_s` —— 壳会话寿命

| | |
| --- | --- |
| 类型 | 整数（秒） |
| 作用 | >0 时入口在 ~该时长（随机化 `[半, 全]`）后主动关壳会话，促重连走新出口 IP |
| 默认 | `0` = 不限 |
| 副作用 | 长传输会被打断（客户端自动重连） |

### `shell_sni` —— 固定壳伪装 SNI

| | |
| --- | --- |
| 类型 | 字符串 |
| 作用 | 固定壳每连接的伪装 SNI；空 = 每连接从内置池**轮换**（默认） |
| 默认 | 空（轮换） |

设为**该出口 IP 实际能对上的站点**更隐蔽；否则留空轮换。

---

## quic-obfs 整形

仅 `relay_mode=quic-obfs` 生效。

### `quic_obfs_pad` —— datagram 填充档

| | |
| --- | --- |
| 类型 | 字符串 |
| 取值 | `""`/`256`（256 边界量化，默认）· `mtu`（填到 1280） |
| 默认 | `256` 边界 |
| 方向 | **仅入口→出口**（同 `shell_pad` 不对称） |
| 上限 | 无论哪档，在途 UDP 报封顶 **1280B**（IPv6 最小 MTU，防 IP 分片） |

噪声注入（入口侧 8–24s 抖动发噪声报）默认开、无需配置。

---

## 接入层入站

### `in_bind` —— 入站监听绑定 IP

| | |
| --- | --- |
| 类型 | 字符串 |
| 作用 | 入站监听绑定的本地 IP（传统转发 tcp 路径） |
| 默认 | 空 = 全部 `0.0.0.0` |

### `ingress_conn_max_s` —— 入站连接寿命（无感重连）

| | |
| --- | --- |
| 类型 | 整数（秒） |
| 作用 | >0 时入口在 ~该时长后主动关客户 TCP，稀释单入站 IP 的「超长连接」取证特征。开启后内置「无感」收尾：等当前空闲再以 FIN 优雅断连，客户端自然重连，避免传输中途腰斩。 |
| 默认 | `0` = 不限 |
| 生效范围 | 仅 TCP 类（传统 / 壳 / TLS / REALITY）；forward 规则 + 单端口规则两条路径都生效 |

!!! warning "建议 ≥300s"
    太短会频繁断流伤体验。上线前在测试环境确认你的客户端会自动重连。

配套两个高级参数（留空走默认）：

- **`ingress_idle_ms`**：到寿命后连续这么久（毫秒）无数据流动即判定空闲并优雅关。空=默认 1500。
- **`ingress_grace_cap_s`**：到寿命后最多再等这么久（秒）等空闲；等满仍未空闲则强制优雅关，防大流量连接永不空闲而超长存活。空=默认 45。

---

## 源 IP 选择 / PROXY protocol

### `egress_policy` —— 入口出站多源 IP 选择

| | |
| --- | --- |
| 类型 | 字符串（`rr` 轮询，默认 · `random`） |
| 作用 | 入口配了多个出站源 IP 时，在它们之间怎么选（配合规则的出站源 IP） |

### `exit_egress_policy` —— 出口→落地段多源 IP 选择

| | |
| --- | --- |
| 类型 | 字符串（同上） |
| 作用 | **仅中转时**，出口机拨业务机的源 IP 选择（配合出口段源 IP） |

### `proxy_proto` —— PROXY protocol 发送

| | |
| --- | --- |
| 类型 | 字符串（`""` 关 · `v1` 文本头 · `v2` 二进制头） |
| 作用 | 开启后入站把 PROXY 头作为首段写入上游（经中转透明到达落地），**让落地拿到真实客户端 IP** |

### `proxy_recv` —— PROXY protocol 接收

| | |
| --- | --- |
| 类型 | 布尔 |
| 作用 | 解析客户端/上游传入的 PROXY 头（v1/v2），取其中原始源地址。**链式中转**用：前一跳已带 PROXY 头时，把最原始客户端 IP 继续传给落地 |

---

## 全参数速查表

```json
{
  "relay_mode": "shell",
  "tls_sni": "www.microsoft.com",
  "decoy": "www.apple.com:443",
  "tls_vers": "1.3",

  "exit_ip_cap_gb": 100,
  "exit_ip_cap_window_h": 48,
  "exit_ip_cap_full": "rotate",

  "shell_pad": "256",
  "shell_session_max_s": 1800,
  "shell_sni": "",

  "quic_obfs_pad": "mtu",

  "in_bind": "",
  "ingress_conn_max_s": 600,
  "ingress_idle_ms": 1500,
  "ingress_grace_cap_s": 45,

  "egress_policy": "rr",
  "exit_egress_policy": "rr",
  "proxy_proto": "v2",
  "proxy_recv": false
}
```

| 键 | 类型 | 作用形态 | 默认 |
| --- | --- | --- | --- |
| `relay_mode` | str | 全 | `aes` |
| `tls_sni` | str | tls / quic-obfs | `www.microsoft.com` |
| `decoy` | str | shell / reality | 空 |
| `tls_vers` | str | trojan | 实现默认 |
| `exit_ip_cap_gb` | int | 全（出口多 IP 才生效） | 不限 |
| `exit_ip_cap_window_h` | int | 同上 | 48h |
| `exit_ip_cap_full` | str | 同上 | `rotate` |
| `shell_pad` | str | shell | `mtu` |
| `shell_session_max_s` | int | shell | 不限 |
| `shell_sni` | str | shell | 轮换 |
| `quic_obfs_pad` | str | quic-obfs | `256` 边界 |
| `in_bind` | str | 全（tcp 路径） | `0.0.0.0` |
| `ingress_conn_max_s` | int | TCP 类 | 不限 |
| `ingress_idle_ms` | int | 同上（仅寿命>0） | 1500 |
| `ingress_grace_cap_s` | int | 同上（仅寿命>0） | 45 |
| `egress_policy` | str | 全（入口多源 IP） | `rr` |
| `exit_egress_policy` | str | 中转（出口多源 IP） | `rr` |
| `proxy_proto` | str | 全 | 关 |
| `proxy_recv` | bool | 全 | false |

---

## 按场景推荐配置

=== "A · 通用跨墙（壳）"
    ```json
    { "relay_mode": "shell", "shell_pad": "256", "shell_session_max_s": 1800 }
    ```

=== "B · 最敌对跨墙跳（无 SNI）"
    ```json
    { "relay_mode": "quic-obfs", "quic_obfs_pad": "mtu" }
    ```
    出口 + 入口两端都要 v1.9.0 及以上。

=== "C · 出口 IP 抗封（多 IP 出口）"
    ```json
    { "relay_mode": "shell", "exit_ip_cap_gb": 100, "exit_ip_cap_window_h": 48, "exit_ip_cap_full": "rotate" }
    ```
    出口装机 `-relay-addr` 配逗号多 IP。

=== "D · 入站 IP 稀缺、稀释取证"
    ```json
    { "relay_mode": "shell", "ingress_conn_max_s": 600 }
    ```

=== "E · 让落地拿真实客户端 IP"
    ```json
    { "relay_mode": "aes", "proxy_proto": "v2" }
    ```

=== "F · 最隐蔽（借真站，IP↔域名对上）"
    ```json
    { "relay_mode": "reality", "decoy": "www.apple.com:443" }
    ```

---

!!! note "参数语义以面板版本为准"
    各版本变更见更新日志。本页参数随面板能力演进，新版本可能新增键。
