# forward-panel v1.2

## 修复
- **带宽曲线尖峰**:`delta()` 不再把 NIC 累计字节当增量(基准未初始化/计数器回绕一律记 0);`bucketsToRates` 加 ~400Gbps 物理上限钳制。速率读时算,修复对历史数据回溯生效,无需清库。
- **组规则检测不显示**:组入口规则 `node_id=group:X`,前端 `NODES[group:X]` 为空导致 tcping/出站健康永远空白。后端给组规则补 `member_nodes`,前端 `ruleProbeNode()` 解析到组内在线成员展示其检测。

## 新增
- **入口出站 IP 健康自动切换**(规则级):开启 + ≥2 个出站 IP 时,每 5s 从各源 IP tcping 真实拨号目标,自动跳过不通 IP,全挂则放行。健康态经心跳上报、前端可见。
- **每规则流量统计**:独立持久累计每条规则的入/出字节(组规则汇总成员),重启不丢,规则列表新增「流量·入/出」列。与计费倍率解耦,展示原始字节。
- **规则高级设置 · JSON(仅管理员)**:规则编辑器内 admin-only textarea,白名单解析覆盖规则字段(未知键/非法 JSON 自动忽略,原始 JSON 不下发到节点)。当前支持键:
  `tls_sni` `decoy` `proxy_proto` `proxy_recv`(bool) `relay_mode` `tls_vers` `in_bind` `egress_policy` `exit_egress_policy`。独立 `rule_advanced` 表存储,不动主规则表 CRUD。

## 优化
- 规则编辑器入口选择处加「整条线路 = 入口分组(DNS 负载均衡)」说明,提示客户端需走线路域名而非单 IP。

## 暂挂(后续版本)
- 入口公网 IP 展示修复(NAT 机显私网 IP)——待确认节点视图地址字段后修。
- 分段检测(各跳探测下一跳)——协议级改动,放 v1.3 单独做。
