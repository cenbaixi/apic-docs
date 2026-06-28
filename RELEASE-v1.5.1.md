# forward-panel v1.5.1 — 分组改名 bug 修复 + 节点改分组改为弹窗

仅 master + 前端。proto/agent/hub 不变。

## 1 · 分组改名 bug(核心)

**现象**:给协议层分组改名后,旧名不消失、带着节点显示在接入层,新名作为空组(0 节点)显示在协议层。

**真因(前端渲染时序)**:`renameGroup` 里是 `await loadGroups(); loadNodes();`——`loadGroups` 内部用**全局 `NODES` 缓存**渲染分组,但此时 `loadNodes()` 还没执行完(在其后、未 await),`NODES` 还是旧的(节点 group 仍为旧名)。而每秒轮询**不重渲分组**(避免冲掉正在编辑的改名框),于是错误画面持续:
- 旧名来自旧 NODES 计数 → 无 meta → 落接入层、带节点;
- 新名来自新 groups 表 → 有 meta → 协议层、0 节点。

后端 DB 其实改对了(node_meta.grp 已迁、groups 行改名保留层)。

**修复**:`renameGroup` / `deleteGroup` / `moveNodeGroup` 收尾统一改为 **先 `await loadNodes()` 再 `await loadGroups()`**,让分组渲染拿到刷新后的 NODES。

## 2 · 后端 RenameGroup 补漏(顺带)

原 `RenameGroup` 只迁移了 `groups` 表和 `node_meta.grp`,**没迁移规则里的组引用**——协议层分组被规则当出口用时(`exit_group`),或被当入口组(`node_id="group:X"`),改名后规则会指向不存在的组。现补上:
- `UPDATE rules SET node_id='group:新' WHERE node_id='group:旧'`
- `UPDATE rules SET exit_group='新' WHERE exit_group='旧'`
- 并加 `old==nw` 守卫。

## 3 · 节点改分组:内联下拉 → 弹窗

原「分组」页节点单元格里每个节点带个内联 `<select>` 直接改组,挤且易误触。改为:
- 每个节点显示 **「查看」按钮**;
- 点击弹出 `nodeGroupModal`,显示该节点 ID / 当前分组 / 当前层 + 一个分组下拉 + 保存;
- 保存走原有 `moveNodeGroup`(POST /api/nodes/meta,节点跟随目标分组的层),收尾刷新顺序也已修正。

## 校验

- gofmt 干净;`<div>` 420/420、select 51/51、modal 15 个平衡;
- 时序正确序 3 处、错误序 0;grpNodeCell 内联 select 清零;openNodeGroupModal 定义+引用齐。

## 部署

- 后端(规则迁移):重编 master。
- 前端(时序 + 弹窗):覆盖 `web/index.html`(可热替换 + 浏览器强刷)。
- 验证:改名一个协议层分组 → 节点应整体留在协议层新名下、旧名消失;若该组被规则当出口/入口,规则仍指向新名;节点「查看」→ 弹窗里改分组生效。
