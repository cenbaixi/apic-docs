# forward-panel v1.4.3 — 中转检测分级化 + 真实传输端口探活

## 背景
v1.4.x 之前入口段检测有两个问题:
1. 入口探的是「自己的规则监听口」(环回 127.0.0.1:listen),这不是一个网络跳,定位不出哪段坏;节点掉线本来心跳就报,这个自探多余。
2. 入口→出口隧道探的是**中转基址**(relayAddr,base),而数据面实际拨的是 `RelayPortFor(base, 模式)` = **base+模式偏移**。裸 AES 偏移 0 两者相同,但壳/TLS/reality/quic 的真实隧道口在 base+1/+2/+3/+4。**基址通 ≠ 该模式的真实口通**——非裸模式下会探错口、判错健康(出口某模式口挂了、基址还在,旧逻辑漏判,该规则仍轮到这个出口白拨)。

## 本版改动
**入口段改为分级检测,每个出口下两级,各管一跳:**
- **第一级 入口 → 出口真实中转口**:真实传输端口 = 基址 + 模式偏移,与数据面拨号完全一致。这一级即中转容灾判据。
- **第二级 出口 → 落地**:出口↔落地这一跳。

哪一级红,就定位到哪一跳。出口某模式中转口被摘除时,「已舍弃」标在第一级对应入口段(各入口都拨不通这口),并注明问题在中转端口、非出口→落地;不再误标到健康的出口→落地段。

**去掉入口对自己监听口的环回自探**(127.0.0.1/公网 IP 都不再探)。监听口未绑上(端口冲突等)由 agent 直接上报抓取(下个版本接入),不再靠环回 tcping。

**中转容灾判据从「按出口」细化为「按(出口, 模式)」。** 出口每个外层模式开独立 relay 监听口(base / base+1 / base+3...),现在按真实口分别判健康:`aws` 的 REALITY 口(:803)挂了只摘 REALITY 规则的 aws,AES 规则(:800)的 aws 照用;互不牵连。

## 涉及文件
- `cmd/agent/main.go`:`entryProbeTarget` 改探 `kernel.RelayPortFor(Hops[0], 模式)`;`proberTargets` 去掉监听口自探。
- `cmd/master/server.go`:节点 `relayHealthy/relayLoss` → 按真实口 `relayPortUp/relayPortLoss` map;新增本地 `relayPortFor`(镜像 kernel 偏移表)、`relayPortOK`、`nodeAnyRelayDown`;`probeRelays` 重写为按(出口,模式)判真实口;出口池摘除、`nodeProbeExtra`、`apiRuleSegments` 全部改按真实口。
- 数据面零改动;agent 协议版本不变。

## ⚠ 部署与验证
本版动到**容灾路由**(出口池摘除/纳回的判据与键),属实质改动:
1. 先在 us-lax02 `./release.sh 1.4.3` 编译(类型/语法把关)。
2. 部署到 **test2** 验证路由:确认中转规则正常转发、规则弹窗分级段显示正确(入口→中转口 / 出口→落地)、出口某模式口人为断开时只摘对应模式规则。
3. 验证无误再上生产。

`relayPortFor` 的偏移表必须与 `internal/kernel/relaytransport.go` 的 `relayPortOffset` 保持一致——改 kernel 偏移表时同步改 master 这份。
