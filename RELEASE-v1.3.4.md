# forward-panel v1.3.4

新增:**规则「检测」改为分段检测——组规则展开组内每个成员节点,各显示最近 5 次 tcping**。仅改 `web/index.html`(前端),master/agent 无需重编。

## 改动

此前规则行「检测」只挑**一个**在线成员(`ruleProbeNode`)显示其 tcping。改为 `showRuleDetect(ruleId)`:

- **组规则**(`node_id=group:X`):遍历 `member_nodes`(apiRules 已下发)里**每个成员节点**,各自拉 `/api/pinglog`,弹窗里逐节点分段展示(节点名 + 在线状态 + 近5次 tcping + 成功/失败/连失汇总)。
- **普通规则**:单节点,同样展示最近 5 次。

数据全部现成:master 本就为每个 (节点,目标) 维护每秒 tcping 历史(`pingLog`,最近 30 条),`apiPingLog` 按节点+目标返回。组内每个成员都是入口、都在探目标、master 都存了各自历史——所以这是纯展示层改动,无后端/协议改动。

## 已知限制

- 探测目标用 `r.target`。**对带中转(exit_node→Hops)的规则**:入口实际探的是解析出的下一跳(跳板地址),而面板展示用的 rule JSON 不含 Hops(`dbRuleToProto` 故意不带),故这类规则成员会显示"暂无 tcping 数据"。直连落地的规则(无 exit_node)正常。要覆盖中转规则需后端按规则解析每成员实际探测目标(另做)。
- 这是**组内并行成员**的逐节点延迟,不是**多跳链路逐段**(entry→relay→exit→落地 各段)延迟。后者需 master↔agent 新协议消息(让中转/出口探各自下一跳),仍未做。

## 部署

只是前端。重新 `release.sh` 出包(重新加密 `index.html.enc`),装 test2 或推存量客户。规则行点「检测」即见组内每个节点的分段 tcping。

## 附:规则流量统计「一直 0」的说明(非 bug)

排查结论:kernel 按 rule ID 计每规则字节(TCP/UDP/单端口三种 forward 都按 `rule.ID` 存)、组规则按 rid 跨成员求和,机制正确。"0" 的原因:`accrueRuleTraffic` 只计**自 master 上次见到该规则起的增量**(首见只播种不计,避免重复计)。今天 master 多次重启,每次重启后从 0 重新播种;若此后没有**新流量**流过规则,自然显示 0。验证:让客户/curl 真的走一遍规则入口端口产生流量,流量列即增长。若有活跃流量仍为 0 才是 bug。
