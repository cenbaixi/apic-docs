# forward-panel v1.4.0

## 本版解决
1. **删除/暂停规则可靠释放端口**(修复「删了规则但端口还监听」的孤儿端口)
2. **新增:规则暂停 / 批量暂停·恢复**

---

## 1. 孤儿端口自愈(删除规则可靠释放监听)

### 问题根因
删规则时 master 确实下发 `rule.remove`,agent 也确实 `Remove→close listener`。但 `sendToNode` 对**离线节点静默丢弃**——若删除那一刻入口节点正在重连(频繁重启 master/agent 时),`rule.remove` 丢失,且**连接稳定期没有任何兜底对账**,孤儿端口一直挂着直到节点重新注册。

### 修复:周期性权威全量对账(self-heal)
- master 新增 `resyncLoop`:每 **120s** 给所有在线节点重发一次权威全量规则集(复用 `redispatchAll` 的 `Type:"sync"`)。agent 用既有对账逻辑(`for id := range old { if !seen { Remove }}`)清掉任何已删/已暂停规则的孤儿端口。
- 不管 `rule.remove` 是因离线 / 时序 / master 重启丢失,**孤儿端口最多 120s 自愈**。

### 配套:agent 按配置哈希跳过未变规则(关键)
- 周期性重发全量,若 agent 无脑 `Apply` 每条规则,会把在用监听反复 drop/rebuild(`Apply` = removeLocked+重建)→ 抖动。
- agent 新增配置指纹(`ruleHash` = 规则 JSON 的 fnv-64a),`sync` 时**仅对新增/变更的规则 `Apply`**,未变规则跳过。周期 resync 因此**不扰动在用流量**,只删该删的。
- **`proto.Version` 0.86 → 0.87**:提示节点更新到带哈希跳过的 agent。
- ⚠️ **滚动升级提示**:节点 agent 升到 0.87 前,周期 resync 会短暂重建该节点监听(每 120s 一次)。**请先把各节点 agent 更新到 0.87**,再依赖周期 resync。更新后无抖动。

---

## 2. 规则暂停 / 批量暂停

### 行为
- **暂停**:解除该规则的端口监听(立即释放端口),保留规则记录;后续所有下发/对账都跳过暂停规则(周期 resync 也不会把它补回去)。
- **恢复**:重新下发监听,规则照常工作。

### 实现(低风险隔离设计)
- 暂停态存于**独立小表 `rule_paused(rule_id)`**,**完全不动 rules 表的列结构**(避免触碰按列位置硬映射的 `Scan` 代码)。
- 删除规则时连带清除其暂停标记。

### API
- `POST /api/rules/{id}/pause` — 暂停单条
- `POST /api/rules/{id}/resume` — 恢复单条
- `POST /api/rules/batch` — 批量,body `{"ids":["..."],"action":"pause"|"resume"}`
- 均做 owner/admin 越权校验。

### 前端
- 规则列表每行加**勾选框** + 表头**全选**。
- 每行操作区:**暂停 / 恢复** 按钮(按当前状态切换);暂停的规则**整行置灰** + 红色「暂停」徽标。
- 工具条:**批量暂停 / 批量恢复**(对勾选行批量操作,带条数确认)。

---

## 部署
1. **先更新所有节点 agent 到 0.87**(主控部署后节点会显示「需更新 0.86→0.87」),再依赖周期 resync 自愈。
2. master 部署后 `resyncLoop` 自动运行,无需额外配置。

## 立即解除现有孤儿端口(无需等部署)
在入口节点上重启 agent —— 重启后只 Apply 当前规则,已删规则的端口随之释放:
```
systemctl restart <agent服务名>
```

---

## 3. 修复:`systemctl restart fp-agent/fp-master` 卡 90s

### 问题
agent 会拉起子 shell 进程(网页终端 `/bin/bash -l`)和分离的更新进程;两个 systemd 单元用默认 `KillMode=control-group` + 停止超时 90s。重启时主进程收 SIGTERM 立即退(端口已释放),但子进程不响应 SIGTERM,systemd 死等 90s 才 SIGKILL,命令看似"卡死"。

### 修复
装机模板(`provision.go` 的 fp-agent 单元、`install.sh` 的 fp-master 单元)加:
```
KillMode=mixed
TimeoutStopSec=8s
```
主进程收 SIGTERM 立即退(端口立刻释放),残留子进程 8s 后 SIGKILL,重启最多 8s。

### 现有节点(无需重装,立即生效)
```
mkdir -p /etc/systemd/system/fp-agent.service.d
cat > /etc/systemd/system/fp-agent.service.d/fastkill.conf <<'CONF'
[Service]
KillMode=mixed
TimeoutStopSec=8s
CONF
systemctl daemon-reload
```
master 同理(目录换成 `fp-master.service.d`)。

### 重启卡住时的应急解除
另开 SSH 窗口执行 `systemctl kill -s KILL fp-agent`,SIGKILL 整个 cgroup,卡住的停止立即完成。

---

## 4. 规则端口护栏 + 中转端口迁移到 800-805 + UI 修复(节点列 nowrap)

### 背景
规则监听口若撞上 agent 自身的中转监听端口,会 `bind: address already in use`,**规则静默不生效**(客户端连上的是中转服务,不是规则入口)——排查极难。

### 改动
1. **中转 base 从 9100 迁到 800**:agent 中转监听由 9100-9105 改为 **800-805**(base/shell/tls/reality/quic/quic-obfs)。腾出 9xxx 常用区间给规则。
   - `agent -relay` 默认 `:800`、`-relay-addr` 默认 `<ip>:800`;`provision.go` 装机命令同步用 `:800`。
2. **建/改规则时拒绝监听口落在保留区间 800-805**(`portConflict` 内,建+改两条路径都覆盖)。报错即时返回,前端 `formErr` 显示。规则列表照常秒建,不阻塞。
3. **前端即时反馈**:监听端口输入框落在 800-805 时,无需提交立即提示"中转保留端口"。

### ⚠️ 迁移步骤(必读)
中转 base 改了,**所有节点都要重装/重配**才能用新 base。**支持滚动升级**(每个节点用对方上报的中转地址拨号,master 校正只换 IP 保留端口):

**逐台节点**重跑装机命令(会重新生成 systemd 单元,`-relay-addr` 变成 `<ip>:800`),或手动改单元的 `-relay-addr` 为 `<ip>:800` 后 `daemon-reload + restart`。

**迁移期注意**:还在 9100 的旧节点,它的中转仍占 9100-9105,而护栏检查的是 800-805——所以**旧节点迁移完成前,给它建 9102 这类规则护栏拦不住**。建议:先把所有节点迁到 base 800,护栏(800-805)才在全网准确。迁完后:
- 9100-9105 **释放**,规则可以用了(包括之前撞车的 9102)
- 规则不能再用 800-805

**关于特权端口**:800-805 < 1024,需 root 绑定。agent 由 systemd 以 root 运行(单元无 `User=`),正常绑定无碍。

### 没做的(护栏 2,按需求暂不做)
agent apply/bind 失败回传面板、规则行显示"未生效"——**本版未实现**。所以非保留区端口若被节点上别的进程占用,bind 失败仍只 log 在节点本地、面板看不到。若以后要彻底消除"规则静默不生效",仍建议补这条。

---

## 5. 修复:每规则流量统计恒为 0

### 根因
传统·TCP+UDP 规则会同时建 TCP 转发器(`k.fwds[id]`)和 UDP 转发器(`k.udps[id]`),**同一个 rule ID**。`kernel.Stats()` 汇总时:
```
for id,f := range k.fwds { out[id] = f.stat() }   // 先写 TCP 字节
for id,u := range k.udps { out[id] = u.stat() }   // 再被 UDP 字节【覆盖】
```
UDP 那行直接覆盖了 TCP 的字节。实际流量走 TCP(sk5/anytls 等),UDP 多为 0 → 覆盖后上报 0 → 面板每规则流量恒显示 0。

### 修复
`Stats()` 里 TCP+UDP 规则**合并两者字节**(In/Out/Conns 相加)而非 UDP 覆盖 TCP。纯 TCP、纯 UDP、单端口入规则不受影响。

---

## 6. 检测改造:检测客户的规则端口,不探内部中转口

### 改动
分段检测从"探内部中转口"改为"探客户的规则端口":
- **入口段**:每个入口 agent 后台定时 **tcping 自己的规则监听口**(`in_bind 或 127.0.0.1 : 监听口`),检测面板显示"→ 规则监听口 :端口 通/不通"——验证客户的监听口到底起没起来。
- **出口段**:出口 → 落地(不变,本就是客户的落地端口)。
- **去掉**原"入口 → 出口隧道入口(中转口 :9100/:800)"那段探测。

### 实现
- agent:`selfListenTarget(rule)` 算出本机监听口探测地址,加入 prober 目标(后台 tcping)。
- master:`apiRuleSegments` 入口段改用 `selfProbeAddr(in_bind, listen)`,去掉隧道口段。两个函数产生完全相同的地址字符串,确保分段检测能查到 ping 历史。
- 前端:检测面板说明文字更新。

### 说明(tcping 的固有限制)
tcping 监听口能确认"该端口有人在听",但分不清是不是这条规则在听(若被别的服务占用,tcping 也会显示"通")。不过 v1.4.0 的端口护栏已在**建规则时**拦掉了规则口撞中转口/撞别的规则,所以这种误报场景基本不会发生;tcping 主要用于抓"压根没人在听"(agent 没起/规则没下发)的情况。

---

## 7. UI:规则表禁止竖排 + 横向滚动

规则表列多(管理员视图 12 列),被挤窄时表头(出站/目标时延/流量入·出/归属)会竖排成单字一列。修复:
- CSS `#rulesHead th, #rules td { white-space:nowrap }` 强制规则表表头+数据不换行。
- 规则表外层包 `<div style="overflow-x:auto">`,列宽溢出时横向滚动,不再竖排。

(只针对规则表,不动审计日志/事件表——那些长文本消息需要换行。)

## 8. install 钉死中转监听口(迁移健壮性)

**问题**:`installScript`(agent 安装脚本)原先只传 `-relay-addr`(通告地址),不传 `-relay`(监听地址),监听口跟着二进制的默认值走。v1.4.0 把 `-relay` 默认从 `:9100` 改成 `:800` 后,**只换二进制不重跑 install** 的节点会「监听 :800 / 通告仍 :9100」错配 → 主控探通告口探不到 → 节点被判「中转不可达·已舍弃」。

**根治**:install 脚本提取 `RELAY_PORT="${RELAY_ADDR##*:}"`,ExecStart 显式传 `-relay :$RELAY_PORT`。监听口被 unit 钉死且永远等于通告口的端口,以后再换二进制/改 base 都不会漂移。

**已在 :800 错配的存量节点(如 aws/狗妈咪),即时修复二选一**:
- 重跑 install 命令(重新生成 unit,自动 `-relay :800 -relay-addr IP:800`),或
- 手动改 unit 端口:
  ```bash
  sed -i -E 's#(-relay-addr +[0-9.]+):9100#\1:800#' /etc/systemd/system/fp-agent.service
  systemctl daemon-reload && systemctl restart fp-agent
  ```
  （新二进制已监听 :800,把通告口也改成 :800 即对齐。）
