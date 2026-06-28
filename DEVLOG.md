# APIC / forward-panel · 开发日志(单一事实源）

> 目的:跨会话不丢上下文。每次开新会话，先读本文件「架构速览 + 给下一次会话」两节即可接上。
> 维护约定:**改了协议/架构就来更新这里**，尤其「中继传输层规格」「已知缺口」「决策记录」。代码注释会过时（已有先例），以本文件为准。

---

## 1. 这是什么

forward-panel（品牌 APIC，apic.gg）= 商业**纯转发/中继**面板（对标 nyanpass）。
我们**不做终端用户协议**：客户是 v2board/机场运营商，他们自己拥有 VLESS/SS 等终端协议、客户端、订阅。我们只把流量从 A 搬到 B。合法双用途隐私基础设施，全部基于公开协议（xtls/reality、utls、quic-go），无 exploit/无恶意代码。

组件:
- **master**：主面板（用户/规则/计费/DNS/CDN…），web 磁盘热更。
- **agent**：跳板/出口节点代理（数据面内核 `internal/kernel`）。
- **hub**（fp-hub）：授权/分发中心，dl.apic.gg。
- **cardsvc**：发卡/授权端（buy.apic.gg），独立 sqlite。
- **licensegen**：签发。

---

## 2. 架构速览：流量怎么走

```
墙内客户端 →〔入口节点 ingress〕 ==外层伪装的隧道== 〔跳板/出口 relay/egress〕→ 落地目标
            （透明 TCP/UDP 接入）   ↑ 唯一跨墙、需要伪装的一跳 ↑    （rule.Target / Hops / egress IP）
```

**核心设计公理（来自 `relaytransport.go`）**:入口→跳板这一跳是**唯一跨墙、需要伪装**的链路；两端都是本方机器，**没有客户端兼容包袱**。所以伪装全压在这一跳，墙外的 跳板→跳板/出口 不需要 SNI 那套。

- **隧道内层永远是 AES-AEAD**（`relayInner` + 业务字节）。`relay_mode` 加的是**外层**。
- 业务字节 = 客户自己的协议流（我们不解析、不携带目的地址；目标由 `rule.Target` 给）。

---

## 3. 中继传输层规格（`internal/kernel/`）

### 3.1 外层传输菜单（按 `rule.RelayMode` 选，分端口）

| relay_mode | 端口 | 外层 | 说明 |
|---|---|---|---|
| `""` / `aes` | base | 无（裸 AES 流） | 最快；但是"未知加密流"，易被盯 |
| `shell` | base+1 (:9101) | **自研壳** | 多态私有特征 + REALITY 式前置认证 + decoy |
| `tls` | base+2 (:9102) | 真 TLS | `tls_mimic.go`/`tlsplain.go` |
| `reality` | base+3 (:9103) | REALITY | 真 utls + xtls/reality + hkdf，借真站 |
| `quic` | base+4 (:9104) | 明文 QUIC | h3 握手即伪装；QUIC STREAM 走 UDP，**天然无 TLS-in-TLS** |
| `quic-obfs` | base+5 (:9105) | Salamander QUIC | 在 QUIC 之下对 UDP 逐包异或加扰，链路上是"一坨随机 UDP" |

内层对所有外层一致：出口侧把每条流（含 QUIC STREAM）当 `net.Conn` 喂给通用 `handleRelay`，与裸 AES 共用落地逻辑。

### 3.2 自研壳 Shell v1（`shell.go`）—— 权威规格

**参数派生（确定性，按 seed）** `ShellParamsFromSeed(seed)`:
- `PSK` = SHA256("apic-shell-v1|psk|" + seed)，32B
- `PadMax` = SHA256("apic-shell-v1|pad|" + seed)[0]，每帧随机填充上限 0..255
- `SNI` = `pickSNI(seed)`，按 seed 从真实大站里选一个（握手伪装用）

**seed 来源** `shellRelaySeed(key, exitID)`:
- v2:`SHA256("apic-relay-shell-seed|v2|"+exitID+"|" + relayKey)` —— **每出口节点一套种子**，跨出口在途特征各不相同
- v1（回退）:exitID 为空 → 共享种子（群组出口/旧节点兼容）。服务端 accept 时同时试「本节点种子」和「共享种子」

**握手**:真 TLS 1.3 ClientHello/ServerHello;临时 X25519 公钥放标准 `key_share` 扩展（`tls_mimic.go`）。
- ECDH → shared；**会话密钥 = SHA256(PSK ‖ shared) → AES-256-GCM**
- PSK 混入派生 ⇒ 无 seed 者算出的密钥不同，首帧 AEAD 解不开即断 ⇒ **隐式鉴权**
- 临时密钥 ⇒ **前向保密**

**REALITY 式前置认证 + decoy**（④，`shellAuthTag`/`shellAuthCheck`）:
- `tag = HMAC-SHA256(PSK, "apic-shell-auth-v1" ‖ epochMinute ‖ client_random ‖ client_pub)[:32]`，放进 ClientHello 的 **session_id**（32B，不计入 JA3，真 Chrome 本就 32B）
- 服务端**先核 tag 再决定是否回 ServerHello**:命中→正常握手;不命中（乱发/真 TLS 探测）→**不回任何字节，整条连接回放给 decoy 真站**（`shellAuthError` 携带原始 ClientHello 上抛给上层）
- `epochMinute` 取 ±1 分钟窗口（防重放 + 容忍机器间时钟漂移），常数时间比对

**数据帧**（`shellConn`）:AES-256-GCM，套 TLS `application_data` 记录外形 `[0x17 0x03 0x03 len]`;明文 = `[2B 真实长度][chunk][随机填充]`。逐帧 `nonce(dir, ctr)`。包长多变、在途像 TLS。
- 注:壳用自己的 AES 分帧重新切块，**不保留内层 TLS 记录边界** ⇒ 起到 Vision 等价作用（打散内层 TLS-in-TLS 的记录尺寸相关性）。

**Salamander**（`salamander.go`，给 quic-obfs）:UDP 之上 QUIC 之下;每包前置 8B 随机 salt，密钥 = BLAKE2b-256(PSK ‖ salt)，逐字节异或（周期 32）。只负责"看起来不像 QUIC"，机密性靠 QUIC/内层 AES。PSK 从 relay key 派生，与内层密钥域分开。

---

## 4. 威胁模型 & 已知缺口（2026-06 核实，含外部研究）

我们**已覆盖**社区节点加固的大部分（这些不用再做）:
- 跨客户/跨出口无统一指纹 → ✅ 每出口 seed
- REALITY 式探测回落真站 → ✅ shell decoy + reality 借站
- 前向保密 / 线上无静态密钥 → ✅ 临时 X25519 + PSK 混入
- TLS-in-TLS → ✅ reality(Vision) / QUIC(流帧) / shell(AES 重分帧)
- 每帧随机填充 → ✅
- QUIC 混淆（Salamander/对标 Hy2 obfs）→ ✅ quic-obfs
- 体积/时序整形（量化填充 + 突发抖动 + 抖动 keepalive + 随机会话寿命）→ ✅ 壳整形（#3，默认开）
- 壳 SNI 每连接轮换 → ✅（#4a）
- QUIC datagram 填充 + 噪声包注入 → ✅（#5，对标 Hy2 UDP noise）
- 出口 IP 池轮换 + 单 IP 限量 → ✅ route2（#2）
- 接入层入站连接寿命帽（稀释单入站 IP 取证特征）→ ✅ Tier1

**真正的缺口（按优先级，TODO）**:

1. **[✅ 已完成 2026-06-28] 壳握手 JA3/JA4**：自研壳客户端握手已改走 utls（`HelloChrome_Auto`，真 Chrome 指纹）；X25519（标准 key_share 0x001d）与 auth-tag（session_id）注入 utls 生成的 hello，之后**不调** utls 真握手（壳服务端非真 TLS），自己写 ClientHello / 读壳 ServerHello / ECDH / 切 AES-GCM。**服务端无需改**：`findX25519KeyShare` 本就遍历 key_share、只取 0x001d，自动跳过混合 PQ 项 0x11ec。
   注：此缺口**只影响自研壳**；REALITY 档早已用 utls（`reality_relay.go`），指纹没问题。
   遗留（可选小加固）：壳服务端 ServerHello 仍手搓（JA3S 非真，但 TLS1.3 下 ServerHello 后即加密，被动指纹弱）；若要再紧，可让壳在 ServerHello 后补几条仿 EncryptedExtensions/Finished 尺寸的加密记录再进 app_data。

2. **[✅ 已完成 2026-06-28·#2 route2] dc_ip + IP↔SNI 不一致 + 流量阈值封 IP**。出口 node 可配多 relay IP（`-relay-addr` 逗号分隔），master 按「滚动窗口内单 IP 总流量」过滤超帽 IP、把未超帽 IP 加权进 ExitPool；每连接轮换出口 IP（ingress 本就轮询 ExitPool，壳 seed 按 node_id 不按 IP，轮换不破握手；计量在 handleRelay 出口 secureConn 侧，中段裸 splice 零拷贝不受影响）。配置 `exit_ip_cap_gb/window_h/full`（rotate 软限全含 / stop 硬断空池）。capRefreshLoop 60s 周期重算超帽集（否则只事件驱动）。单 IP 部署 100% 不变（只多 IP node 才过滤）。**残留**：IP↔ASN 深层不一致（VPS 声称大站）轮换解不了，仅 reality 借真站可解；quic-obfs 线上无 SNI 绕开此问题，最敌对跳优先用它。

3. **[✅ 已完成 2026-06-28·#3] 体积/时序整形**（仅作用自研壳，tls/reality/quic-obfs 不动）。三件默认开、无开关：① 填充量化——每记录填到 ~1400 wire（默认）或 `shell_pad:"256"` 量化到 256 边界，去掉原 0-255B 随机填充；② 突发抖动——距上次写 >30ms 才插 0-12ms 抖动（背靠背批量零延迟，不伤吞吐）；③ 抖动 keepalive（15-25s，空闲发 0 长帧，收端透明丢弃）+ 随机会话寿命（`shell_session_max_s`，到点关连接逼客户端重连换出口 IP）。帧格式 `[2B chunkLen][chunk][pad]` AEAD 封装、收端按长度剥 pad → 填充改动只在写侧、收端模式无关、对称安全。**方向不对称**：`shell_pad` 仅作用 ingress→exit（出口壳服务端跨规则共用，固定默认档）。

4. **[✅ 已完成 2026-06-28·#4] 壳 SNI 轮换 + decoy 接线**。(4a) `pickSNI(seed)` 改 `randomShellSNI()` 每连接从 7 个国内可达 CDN 站随机（服务端不校验 SNI，鉴权靠 session_id 内 PSK tag，故 SNI 可自由轮换）；`shell_sni` 可固定覆盖。移除 crypto/sha256（pickSNI 唯一用户）。(4b) **发现 decoy 设计了没接线**——壳鉴权失败原是 `c.Close()`（突兀关闭=强主动探测信号），现改：认证错误 → 解析 ClientHello SNI → 拨池内站 :443 → 回放原始 ClientHello → splice（看起来像真服务端 front CDN）；decoy 目标限内置池防 SSRF。**残留**：同 #2，IP↔ASN 深层不一致轮换解不了。

5. **[✅ 已完成 2026-06-28·#5] QUIC datagram 填充 + 噪声**。Salamander 新 wire 格式（salt 内、XOR 内）：`[1B类型][2B长度][payload][pad]`，类型 0x00 数据 / 0x01 噪声。填充按 `quic_obfs_pad` 量化（256 默认 / mtu 填到 1280），**封顶 1280（IPv6 最小 MTU）防 IP 分片**（UDP 分片不可靠+自成指纹）。噪声：入口侧（单 peer）每 8-24s 抖动注入随机大小噪声报，收端按类型整包丢弃、对 quic-go 透明；出口侧多 peer 不注入（newObfsPacketConn vs newObfsDialerConn 区分）。**⚠ wire 格式变更，老↔新 obfs 不兼容，quic-obfs 两端须同时升级**（升级窗口该模式短暂中断，其它模式不受影响）。方向同 shell_pad 不对称。明文 `quic` 档已在 #1 移除，SNI 切片议题作废。

6. **[✅ 已修 2026-06-28] 文档/注释漂移**：shell.go decoy 注释已随 #4b 接线更新；本文件已勾掉各缺口。

7. **[低] 前置认证 ±1 分钟重放窗口**:窗口内捕获的合法 ClientHello 可重放探测"服务端是否回 ServerHello（而非 decoy）"。无 seed 仍解不开会话，风险低；如要收紧可加一次性 nonce 记忆。

**QUIC 两问的结论**:
- *原生 QUIC 有没有必要*:有用但**别当跨墙主力**。2026-02 起电信从 QUIC Initial 提 SNI 并限速，裸 Hy2 约 30s 被识别。明文 `quic` 档留给**非跨墙跳/性能**或想要 h3 形态时;跨墙用 `quic-obfs`。高丢包链路 QUIC 优于 TCP（避免 TCP-over-TCP）。
- *QUIC+ 填充要不要做*:要，但单填充不够 → 见缺口 #5（填充 + 噪声包 + 必要时 SNI 切片）。

---

## 5. 决策记录（时间倒序，新的在上）

- **2026-06-28** **#2+#3+#4+#5+Tier1 完成**（v1.8.0 之上，一批；proto/kernel 共享 → master+agent 都重建）：
  - **#2 出口 IP 轮换+限量(route2)**：proto.Msg 加 RelayIPs(register)/RelayIPUsage(heartbeat 按 IP 小时桶)；新 `relayipmeter.go`(168 桶滚动计量 + meterConn)；agent `-relay-addr` 逗号多 IP；master 加 `expandExitIPs`(只多 IP node 才改写 ExitPool)+`capRefreshLoop`(60s)+`sumRecentBuckets`；layers `errExitBlocked` 双处守卫(防裸奔出网)。配置 `exit_ip_cap_gb/window_h/full`。
  - **#3 壳整形**：去 0-255B 随机填充，改 mtu/256 量化填充(`shell_pad`)+ 突发抖动(>30ms 间隙才插 0-12ms)+ 抖动 keepalive(15-25s)+ 会话寿命(`shell_session_max_s`)。`emitFrame` 持 wmu 保证 nonce 序=线序。默认开，`shell_pad` 仅 ingress→exit 不对称。
  - **#4 壳 SNI 轮换+decoy**：`randomShellSNI()` 每连接随机(7 站池)，`shell_sni` 可固定；移除 crypto/sha256。**接线先前设计未启用的 decoy**：鉴权失败 → 回放 ClientHello 到池内站(防 SSRF)，不再突兀 Close。
  - **#5 quic-obfs 整形**：Salamander 改 wire 格式 `[类型][长度][payload][pad]`，256/mtu 量化封顶 1280 防分片；入口单 peer 噪声注入(8-24s)，出口不注入。`quic_obfs_pad` 配。**wire 不兼容，quic-obfs 两端须同时升级**。
  - **Tier1 接入层入站连接寿命**：`ingress_conn_max_s`>0 时入口到随机时长(`[maxS/2,maxS]`)关客户 TCP 逼重连，稀释单入站 IP 取证特征(IP 稀缺不可换时的力所能及保护，非根治)。forward + 单端口两条路径都做。
  - 沙箱无法编译 Go → 全程 gofmt + 结构 grep 自检，交 us-lax02 build。
- **2026-06-28** **#1 完成**：自研壳客户端握手 utls 化（真 Chrome JA3/JA4），X25519/auth-tag 注入 utls hello，服务端解析无需改。**移除原生 quic relay 档**（relay_mode="quic"，保留 quic-obfs）：删 layers dial 分支 / kernel+master 端口偏移 case / agent 监听 / UI 选项与标签（relayLabel/showSni）。注：master 的 `encrypt=="quic"` 是另一字段（纯转发下=none），与本次无关，未动。
- **2026-06-27** cardsvc：管理员推送（新工单/新订单/到账/续费/授权异常，异常轮询去重）、下单强制客户自设查询密码、查单支持 订单号/域名/邮箱 任一 + 密码、bot 命令菜单（setMyCommands，管理员作用域）、后台「管理员 TG ID」UI（DB 存储 + 热生效）。
- **2026-06-19** **架构转向纯转发**:移除终端用户协议终结器；伪装从 ingress 层**下移到 入站→跳板 中继隧道层**（可插拔外层 rawAES/shell/tls/reality/quic/quic-obfs）。加 PROXY protocol v1/v2 收发、nyanpass 格式批量导入导出。
- **2026-06-19** 壳 v2:**每出口 node_id 种子**（v0.59），取代 v1 共享种子（保留作回退）。
- **2026-06-19** Salamander 混淆 QUIC 档（quic-obfs）。REALITY 中继档完成（含 post-handshake dummy NewSessionTicket padding bug 修复）、splice(2) 零拷贝、跨节点验证。
- **2026-06-18** 自研壳 v1 落地（外壳 + REALITY 式前置认证 + decoy）。五数据面（TCP/单端口SNI/中继/UDP/自研壳）+ Trojan-over-TLS 兼容档。
- *(更早:见 `/mnt/transcripts/journal.txt` 完整索引)*

---

## 6. 给"下一次会话"的快速上手

- 主业 = **forward-panel（纯转发/中继）**，不是终端协议、不是 cardsvc。cardsvc 只是配套发卡。
- 自研壳 = **`internal/kernel/shell.go`**，规格见 §3.2;别再说"没有协议细节"。
- 编译/签名机 = **us-lax02**（`~/forward-panel` 是 git 仓库，有完整 go.mod）。沙箱内 **无法编译 Go**（proxy.golang.org 不通）→ 改完用 gofmt + 结构 grep 自检，交用户 build。
- cardsvc web 编进二进制（非磁盘热更）→ 改动需 rebuild+redeploy；master web 是磁盘热更。
- 转录全索引:`/mnt/transcripts/journal.txt`（60+ 段）。
- 待办挂账:见 §4 缺口表 + 各 RELEASE-*.md。

> **下次更新本文件时**:在 §5 顶部加一条决策、勾掉 §4 已完成的缺口、§3 同步任何协议改动。
