# forward-panel v1.3.1

v1.3.0 上线后回归修复。**仅改 master + web,agent 未动**(节点机不用重装 agent;`install.sh` 在主控机更新 master 与前端即可)。

## 修复

### 1. 一键清除离线节点:够不着「从没连上过」的节点
`apiClearOffline` 旧逻辑只遍历内存 `m.nodes` 删 `online=false` 的。但**刚在面板添加、agent 从未连接过的节点只存在于 `node_meta` 表**(节点列表靠合并 `node_meta` 把它显示成离线占位),`m.nodes` 里根本没有它 → 清除时数到 0。

改为:收集 ① `m.nodes` 里离线的 + ② `node_meta` 里当前不在线的(`AllNodeMeta` 取全量,排除 `m.nodes` 中 online 的),一并 `delete(m.nodes)` + `DeleteNodeMeta`。现在能清掉 `n-xxxx` 这类占位节点。

### 2. 每个节点行新增「删除」按钮
新增后端 `POST /api/nodes/remove {node_id}`(`apiNodeRemove`,仅 admin):摘 `m.nodes` + `DeleteNodeMeta` + 审计 `node.remove`。前端 `removeNode()` + 节点行红色「删除」按钮(带确认)。**软删除**——在线机的 agent 会自动重新注册再次出现,要彻底清除需先在机器上「卸载」。

### 3. 主节点列表 IP 显示 master 观测到的公网地址
主列表 `nodeRow` 的 IP 单元格此前用 `n.ips[0]`(节点自报),NAT 机自报的是私网(如 `198.51.100.4`)。改为优先 `n.addr`(master 从 `c.RemoteAddr()` 观测到的远端地址,NAT 机即公网):`primary = n.addr || ips[0]`。角标改为按**主显示地址**的实际族判断(`IPv6` / `v4·有v6` / `仅v4`),不再出现「v4 私网旁挂 IPv6 角标」的误导。悬停「本机 IP」框补一行观测到的公网地址(若与自报不同)。零回归:`n.addr` 为空时退回原 `ips[0]`。

> 注:若改后仍显示私网,说明 master 观测到的远端 `addr` 本身就是私网(节点与主控走了私网/同 VPC 链路),那是网络拓扑实情,非前端问题——用 `/api/nodes` 看该节点 `addr` 字段即可判定。

### 4. `apiRules` 不再静默吞错
`apiRules` GET 此前 `rules, _ := m.store.ListRules(...)` 把错误丢弃,`ListRules` 一旦报错就返回空数组,前端显示「还没有规则」,被误读成数据丢失。改为捕获错误:`log.Printf` 记录 + 返回 500。今后规则查询失败会在 master 日志留痕,便于定位(NULL/类型 Scan 等)。

## 不在本版(诊断项,需现场 DB)

- **「规则消失」**:`ListRules` 与 `ruleCols`(32 列)v1.3 未改动,与能正常工作的 v1.2 一致;`node_meta` 完好(分组/节点名都在)→ 基本排除 v1.3 代码引起。需现场查 `rules` 表行数 + master 日志,确认是行真没了还是运行时查询报错。
- **AWS 689.7M/s 尖峰**:确认是 **bug 残留数据**,非真实流量。`bucketsToRates` 用相邻样本实际时间差算速率;v1.3.0 的「离线不采样」修复可防新尖峰(回上线后大间隔均摊),但**改前写入 `traffic_samples` 的旧尖峰不会自愈**。保留期 26h,会自动 prune;或手动删该节点样本行立即清除。
