# forward-panel v1.4.9 — 更新说明 WEBUI + 节点连接表头微调

本版改动:master(新增 changelog 接口)+ 前端。**proto 不变(仍 0.89,沿用 v1.4.8),
v1.4.9 自身的改动不需要 agent 更新**(但 v1.4.8 的 TCP/UDP 分列仍需 agent 升到 0.89,见末尾)。

## 1 · 更新说明 WEBUI(设置 → 更新)

管理员可在「设置 → 更新」卡里发布版本更新内容,记录入库,所有登录用户可见。

### 后端
- 新表 `changelog(id, version, note, created_at)`;`migrateChangelog` 建表。
- 新接口 `/api/changelog`:
  - `GET` → 更新说明列表(倒序;所有登录用户可读)
  - `POST {version, note}` → 新增(**仅 admin**)
  - `POST {id, delete:true}` → 删除(**仅 admin**)
  - 写操作记审计(`changelog.add` / `changelog.delete`)。

### 前端
- 「更新」卡新增(管理员可见)**发布表单**:版本号(可选)+ 更新内容(多行)+ 发布按钮。
- 下方「更新日志」列表:版本 + 时间 + 内容(保留换行),倒序;管理员每条带删除。
- 打开「更新」选项卡时拉取列表;发布/删除后自动刷新。

## 2 · 节点连接表头去掉「T/U」

节点列表「连接」列表头此前为「连接 T/U」,会换行。**还原为「连接」**;
连接值仍显示 `tcp/udp`(鼠标移上去 tooltip 提示 TCP/UDP)。

## 关于「UDP 计数显 0」(非 bug,无需改码)

v1.4.8 的 TCP/UDP 链路代码经端到端复核**完全正确**(udp.go 计数 → kernel 合并 → agent 心跳 →
master 聚合 `n.stats=msg.Stats` → 节点 API `conns_udp` → 前端 `connsTU`)。显示 0 的原因:

1. **节点 agent 尚未更新到 0.89**:proto 从 0.88 升到 0.89,只更新了 master+web 不够,
   各节点(深圳/aws/狗妈咪)的 agent 二进制要重新部署才会上报 `conns_udp`。
   节点亮「需更新」徽标=agent 还是旧的。
2. 即使 agent 更新了,**只有真有 UDP 流量穿过入口时才非 0**(纯网页是 TCP;QUIC/游戏/DNS-over-UDP 才有)。
   入口机上 `ss -u state established` 可看当前 UDP 会话数。

→ 更新节点 agent + 跑点 UDP 流量即出数。

## 校验

- gofmt 干净;`<div>` 417/417、form 13/13、select 51/51 平衡;changelog 前端要素齐全。

## 部署

- v1.4.9 自身:重编 master + 覆盖 `web/index.html`。新表自动建。
- 累积包仍含 v1.4.8 的 proto 0.89:**TCP/UDP 分列要 agent 升 0.89**才出数(否则 UDP 显 0,优雅降级)。
