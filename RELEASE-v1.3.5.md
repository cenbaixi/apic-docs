# forward-panel v1.3.5

**完整版分段检测**:一条规则的整条链路**逐段**探测延迟——入口段(入口→隧道入口)+ 出口段(出口→落地)分别显示,**每段带起点节点 IP**,排查时一眼定位是哪台节点/哪段出问题。

此前「检测」只探到落地一个目标(且对带中转的规则,入口探的是隧道入口、看不到出口→落地段)。现在两段都探、都显示。

## 改动(proto + agent + master + 前端)

链路:`入口 → Hops[0](出口隧道入口) → 出口拨 Target(落地)`。入口段现有 agent 本就在探(`entryProbeTarget` 返回 Hops[0]);**缺口在出口段**——出口跑 seed 转发、本地无规则目标,从不探落地。补法是让 master 把"出口该探的落地地址"下发给出口 agent。**纯探测路径,零数据面改动**(只多跑 tcping,不碰转发)。

1. **proto.Msg** 新增 `ProbeAdd []string`(omitempty):master 经 sync 额外下发的 tcping 目标。
2. **master**
   - `exitProbeTargets(nodeID,group)`:扫所有规则,凡本节点是出口(`exit_node==本节点` 或 `本节点在 exit_group`)就收集其 `Target`。
   - 两处 sync 发送(`redispatchAll` + 节点连接)注入 `ProbeAdd: exitProbeTargets(...)` → 出口 agent 开始探落地。
   - 新端点 `GET /api/rule-segments?rule=<id>`(**仅管理员**):组装整条链路各段 —— 入口段(每个入口节点 → 隧道入口地址 / 直连落地)+ 出口段(每个出口节点 → 落地),每段返回 `from_id/from_name/from_ip/to_addr/to_label/online/pings`。只有 master 知道解析后的 Hops 和持有 pingLog,故后端组装。
3. **agent**:`withExtra()` 把 `ProbeAdd` 并入本地规则派生探测目标(去重),sync/rule.apply/rule.remove 三处生效。
4. **前端** `showRuleDetect`:管理员 → 拉 `/api/rule-segments`,弹窗逐段渲染(入口绿/出口黄角标 + 节点名 + **IP** + 目标 + 最近 5 次 tcping + 成功/失败)。非管理员 → 退回单节点 tcping(不暴露中转拓扑/IP)。

## ★ 部署要点(出口段需出口节点升级 agent)

- **master + agent 都要重编**(`release.sh` 重出 dist)。
- **入口段**:用现有 agent 就显示(入口本就在探隧道入口)。
- **出口段**:出口节点的 agent **必须升级到本版**(要识别 `ProbeAdd` 才会探落地)。出口 agent 没升级时,出口段显示"暂无数据(需出口节点升级 agent)"——**优雅降级,不报错**。
- 升级出口 agent:面板节点「更新」机制,或重装。
- `ProbeAdd` 是 omitempty 兼容字段,**没动 proto.Version**(不触发全网"需更新");出口段按节点逐个生效。

## 部署步骤

```bash
# us-lax02:传包 → 一条龙/分步出 dist(master+agent 都会重编+CGO_ENABLED=0)
# test2:覆盖 web/ + bin/ 重启
scp dist.tar.gz root@test2.apic.gg:/tmp/
ssh root@test2.apic.gg 'cd /tmp && rm -rf du && mkdir du && tar -xzf dist.tar.gz -C du \
  && cp -a du/web/. /opt/forward-panel/web/ && cp -f du/bin/master /opt/forward-panel/master \
  && cp -f du/bin/agent /opt/forward-panel/agent && systemctl restart fp-master'
# 出口节点:升级其 agent(面板「更新」或重装),否则出口段无数据
```

## 验证

拿一条**带 exit_node 的中转规则**点「检测」:
- 入口段(绿):入口节点名 + IP → 隧道入口地址,5 次 tcping。
- 出口段(黄):出口节点名 + IP → 落地,5 次 tcping(出口 agent 升级后出现)。
- 哪段红/失败 → 那台节点 IP 即问题点。

直连规则(无 exit)只有一段:入口 → 落地(直连)。
