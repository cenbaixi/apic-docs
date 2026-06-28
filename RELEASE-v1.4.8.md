# forward-panel v1.4.8 — 规则状态(暂停/停止)+ 节点 TCP/UDP 连接数

本版含两块,部署要求不同(见末尾):

- **主题1 规则状态**:master(规则 API 加 `bind_err`)+ 前端。proto 不变。
- **主题2 节点 TCP/UDP**:proto + kernel + agent + master + 前端。**proto 0.88→0.89,agent 需更新**。

---

## 主题1 · 规则"暂停 / 停止"状态

### 后端(master)
- `apiRules` 规则列表新增 `bind_err` 字段:逐规则取入口节点(组规则=成员节点)的
  `bindErrs[ruleID]`(端口占用/绑定失败等)。独立 `m.mu` 临界区,不与 `groupMemberIDs` 嵌套。

### 前端
- **去掉整行"蒙玻璃"**:删除 `<tr opacity:.45>`,改用状态徽标表达。
- **新增「状态」列**(ID 之后):
  - 手动暂停 → **暂停**(灰)
  - 端口占用/绑定失败 → **停止**(红;鼠标移上去显示停止原因,如"端口被占用")
  - 正常 → **● 运行**(绿)
- 移除 ID 下方原红色"暂停"提示。
- **「恢复」键改为「开始」**(暂停 *或* 停止时显示):点击重新下发(重绑端口)。
  重绑结果下个心跳才到——**成功不提示,仍冲突给 tips**(延时 3s 复查 `bind_err`)。
- 行按钮顺序:检测 · 编辑 · 高级编辑 · 开始/暂停 · 删除。
- **备注名**从「高级编辑」移回「普通编辑」框(基础区 row1)。

## 主题2 · 节点连接数改 TCP/UDP 显示

- `proto.Stat` 增 `ConnsUDP`(UDP 连接数;TCP = Conns − ConnsUDP)。
- `udp.go` 上报 `ConnsUDP`;`kernel.go` 合并 tcp+udp 规则时一并累加;
  master 节点级聚合 `connsUDP`,经 `probeSample` → 节点 API `conns_udp` 暴露。
- 前端三处节点表「连接」列改为 **连接 T/U**,值显示 `tcp/udp`(`connsTU`)。

## 校验

- gofmt 干净、语法 OK,proto.Version=0.89;`<div>` 413/413、form 12/12、select 51/51 平衡;
  状态列 th/td 对齐;新函数(ruleStatusCell / startRule / bindErrReason / connsTU)齐全。

## 部署

- **主题1(状态)**:重编 master + 覆盖 `web/index.html`。proto 没动,旧 agent 兼容。
- **主题2(TCP/UDP)**:**agent 需一并更新**(proto 0.89)。
  agent 未更新前 `conns_udp` 缺省为 0,节点连接显示成 `总数/0`(TCP 显示全量、UDP 显示 0),
  属优雅降级,更新 agent 后即正常分列。节点「需更新」徽标会因 proto 升版亮起,提示更新。
- 常规流程:`SIGN_PRIV=… ./release.sh 1.4.8` → 部署 master+web 到 test2 → 更新各节点 agent。
