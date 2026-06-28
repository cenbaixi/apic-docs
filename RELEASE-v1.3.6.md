# forward-panel v1.3.6

修三件(其中流量0经核查**非 bug**,见末节):

## 1. 修复:出口群组规则的「入口→隧道」段无数据

**现象**:带出口群组的规则点检测,入口段(深圳→隧道入口)显示"暂无数据",但出口段(各出口→落地)正常。

**根因**:出口群组下,入口走 `ExitPool`(每条连接轮询挑一个出口),`Hops` 为空 → `entryProbeTarget` 返回的是落地 Target 而非隧道地址 → 入口压根没探任何出口的隧道入口,而分段视图却去查隧道地址 → 空。(单出口规则有固定 `Hops[0]`,入口本就在探,不受影响。)

**修复**(master only,**入口 agent 无需改**——v1.3.5 的 ProbeAdd 合并已能用):
- `exitProbeTargets` → `nodeProbeExtra`:除"本节点作为出口→探落地"外,新增"本节点作为入口、规则走出口群组→探群组内**每个出口**的隧道入口 `relayAddr`"。经 sync 的 ProbeAdd 下发,入口开始逐个探各出口隧道。
- `apiRuleSegments` 重构:中转规则现在按 **每个入口 × 每个出口 = 一条隧道段** 展开(标注"→ <出口名> 隧道"),再加每个出口→落地段。**每段两端都带 IP**,排查时一眼定位是哪台机器、哪条隧道出问题。

## 2. agent 升级提示(像以前一样的「需更新」+ 更新按钮)

`proto.Version` 0.85 → **0.86**。节点行现有 UI 即据此显示「需更新 0.85→0.86」红字 + 红色「更新」按钮(点了走 `agent.update`,节点自拉新 agent 验签重启),版本从哪到哪一目了然。分段检测的出口段/隧道段需出口、入口 agent 都是新版才完整,升级提示正好引导。

> 注:入口→隧道、出口→落地段在 v1.3.5 agent 上也能工作(ProbeAdd 向后兼容);升到 0.86 主要是消掉"需更新"徽标 + 统一版本。

## 3. 单规则流量统计「一直 0」—— 经核查非 bug,是入口转发没有载荷字节

**逐层核查结论**:
- 内核每规则字节计数正确:中转规则入口每条连接最终都走 `forwarder.splice(client, up)`(`up`=拨出口的连接),两向 `pump` 都原子累加 `f.in/f.out`。
- `k.Stats()` 按 rule ID 返回;`accrueRuleTraffic` 与**计费 `accrue` 同行调用、吃同一份 `msg.Stats`、同样的播种/增量逻辑**(`billing.go`),二者行为一致。
- `apiRules` 正确把 `ruleTraffic` 暴露为 `in_bytes/out_bytes`。

→ 既然计费 accrue 和规则 accrue 同源,**若计费/用量对这条规则有数,规则流量必有数;规则流量是 0 ⇔ 入口转发没有任何字节完成 splice**。连接在 `Auth/Session/Transport.Dial` 任一步失败就会 early return、不进 splice、不计字节。

**最可能**:这条规则是中转规则,而**入口→出口隧道没通**(壳握手失败 / 拨出口失败)——客户端连上入口但中转拨不通 → 无载荷 → 流量 0。这恰好和议题1指向同一处。

**怎么确认**(本版上线后):
1. 对这条规则点「检测」,看**入口→aws 隧道 / 入口→狗妈咪 隧道**段:若 tcping 失败/无数据 → 隧道这段就是断点,先修隧道(出口 relayAddr 可达性 / 壳种子一致性)。
2. 或核对计费:该规则所属用户的用量是否也是 0。是 → 印证无载荷(非统计 bug);否(计费有数而规则 0)→ 那才是真 bug,把这个反差告诉我再查。
3. 真发流量验证:用真实客户端经入口监听端口拉数据,看流量列是否增长。

## 部署

```bash
# us-lax02:重编 master+agent(嵌入 0.86)
# test2:覆盖 web/ + master 重启(入口 agent 不必动,议题1即生效;升级 agent 仅为消"需更新")
scp dist.tar.gz root@test2.apic.gg:/tmp/
ssh root@test2.apic.gg 'cd /tmp && rm -rf du && mkdir du && tar -xzf dist.tar.gz -C du \
  && cp -a du/web/. /opt/forward-panel/web/ && cp -f du/bin/master /opt/forward-panel/master \
  && cp -f du/bin/agent /opt/forward-panel/agent && systemctl restart fp-master'
# 各节点会显示"需更新 0.85→0.86",可逐个或一键更新 agent
```

## 验证

- 那条出口群组规则点检测:应见 **入口→aws 隧道、入口→狗妈咪 隧道、aws→落地、狗妈咪→落地** 四段(或按实际成员数),每段带两端 IP + 5 次 tcping。
- 节点列表出现"需更新 0.85→0.86" + 更新按钮。
- 哪条隧道段红 → 那就是断点,大概率也是流量 0 的原因。
