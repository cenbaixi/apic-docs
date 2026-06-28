# forward-panel v1.3.7

**含 v1.3.6 全部改动**(分段检测多段化、proto 0.86 升级提示),并新增两项中转容灾——都是 master + 前端改动,**agent 无需重编**(proto 仍 0.86)。

## 1. 中转地址按连接源 IP 自动校正(解决"节点公网 IP 漂移没改 relay-addr")

**背景**:agent 上报的 `relayAddr` 来自 `-relay-addr`(装机脚本探测一次写死),节点公网 IP 变了(如 EC2 重启换 IP)就一直是旧的,其他入口照旧 IP 拨中转 → 拨不通。aws 案例:上报 `203.0.113.12:9100`,实际公网已是 `203.0.113.11`。

**修复**:master 注册时**用观测到的控制连接源 IP 校正 relayAddr**——只换 host、保留 agent 上报的端口(`net.JoinHostPort(观测host, 上报port)`)。源 IP 是对端实际可达的公网地址,比节点自报的更可信。校正发生时记事件「中转地址按连接源 IP 校正:旧 → 新」。

- 配置正确的节点(上报 == 观测)不受影响,无校正。
- 仅 IP 漂移/写错的节点被纠正,无需上节点改配置。
- 极少数"中转故意走与控制连接不同公网 IP"的多宿主节点会被改写(可看事件发现);常规单公网 IP 节点全部受益。

## 2. 中转探活 + 出口池故障摘除 + "已舍弃"标注

**现状确认**:数据面 failover **本就有效**(`dialRelayConn`:出口池轮询拨不通就顺次试下一个,直到成功或全部失败)。所以挂掉的出口不会让连接全死。**但挂掉的出口仍留在池里**(构建只看 `online && relayAddr`,不看中转可达)→ 每次轮到它先白拨一次超时再 failover,半数连接吃超时延迟。

**新增**(照边缘容灾 `edgeHealthLoop` 同款模式):
- `relayHealthLoop`:每 8s 主动 tcping 每个在线中转节点的 relayAddr(3s 超时),拨不通 → `relayHealthy=false`。
- **出口池构建**只纳入 `relayHealthy` 的出口;翻转时刷新各入口出口池(摘除/纳回)。**全部不可达则回退纳入全部**(至少尝试,数据面仍逐个 failover),不会因探活误判而完全断服。
- 翻转记事件:「中转地址 X 探活失败,已暂时移出出口轮询」/「…探活恢复,已重新纳入」。
- **面板节点行显示红色「中转不可达·已舍弃」徽标**(`relay_down`),恢复后自动消失。
- 新节点 `relayHealthy` 初始化为 true(乐观,探活前不摘)。

→ 出口节点中转端口拨不通(IP 漂移、安全组挡、监听卡、被打)时:自动移出轮询(入口不再白拨超时)+ 面板可见 + 恢复自动纳回。

## 与 aws 案例的关系

- 议题1 把 aws 的 relayAddr 从错的 `203.0.113.12:9100` 自动纠正为 `203.0.113.11:9100`。
- 若纠正后仍不可达(EC2 安全组没放行 9100),议题2 探活失败 → 自动摘除出口轮询 + 面板标"已舍弃",并提示你去开安全组。
- 两者合力:能纠的自动纠,纠不了的自动摘+标注,不再半数连接吃超时。

## 部署(master + 前端;agent 不必动)

```bash
# 打包前先把仓库里的运行时文件移走(release.sh 安全闸会拦):
mv /root/forward-panel/{panel.db,secrets.txt} /root/fp-local/ 2>/dev/null
# us-lax02 重编(master 即可;agent proto 未变);test2 覆盖 web/+master 重启
scp dist.tar.gz root@test2.apic.gg:/tmp/
ssh root@test2.apic.gg 'cd /tmp && rm -rf du && mkdir du && tar -xzf dist.tar.gz -C du \
  && cp -a du/web/. /opt/forward-panel/web/ && cp -f du/bin/master /opt/forward-panel/master \
  && systemctl restart fp-master'
```

## 验证

1. master 重启后看事件日志:aws 应出现「中转地址按连接源 IP 校正:203.0.113.12:9100 → 203.0.113.11:9100」。
2. 回面板点那条出口群组规则的检测:`深圳→aws 隧道` 的目标地址应变成 `203.0.113.11:9100`;若 EC2 安全组放行了,该段从全红变有延迟。
3. 若 aws 仍不可达:8s 内事件出现「探活失败,已暂时移出出口轮询」,节点行显示「中转不可达·已舍弃」红徽标,出口轮询只剩狗妈咪。
4. 开安全组放行 9100 后:探活恢复事件 + 徽标消失 + aws 重新纳入轮询。
