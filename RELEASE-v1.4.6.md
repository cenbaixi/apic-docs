# forward-panel v1.4.6 — 规则弹窗精简 + 高级编辑入口(纯前端)

> 本版**只动 `web/index.html`**。master / agent / proto **零改动**。
> web 为磁盘伺服(`-web` 目录,默认 `web/`),可**直接替换 `index.html` + 浏览器硬刷新**,
> 无需重编、无需重启 master。(累积 tarball 同时含 v1.4.5 的「已消耗流量」后端改动,
> 那部分要 `已消耗` 数据生效仍需重编 master。)

## 创建/编辑弹窗:基础区精简

弹窗拆成「基础区(常显)」+「⚙ 高级功能(默认折叠)」:

**基础区(默认可见):**
- 入口组(接入层)
- 监听端口
- 出口协议(直连 / 协议层分组)
- 隧道协议 —— **选了出口分组才出现**(直连无隧道,自动隐藏)
- 落地地址 `IP:端口` —— **常显**(直连也需要)

**⚙ 高级功能(点开才展开):**
转发方式(TCP/单端口SNI)· 备注名 · SNI · 入口出站(源IP/策略/自动切换)· 入站绑定IP ·
REALITY借用站点/TLS中转SNI · TLS版本 · 出口出站源IP · PROXY protocol · 管理员JSON(仅admin)·
(及历史保留字段,始终隐藏)

> 字段显隐联动(`applyRuleVis`)原样保留:隧道协议仍按"是否选出口"显隐,REALITY/TLS 的
> 借用站点等仍按所选隧道协议显隐——只是这些细节现在位于折叠的高级区内。折叠时用各字段默认值
> (如 REALITY 借用站点空=www.cloudflare.com),照常可跑。

## 两个编辑入口(同一弹窗,差别仅折叠态)

- **编辑** → 折叠态打开(只见基础区,与创建一致)
- **高级编辑** → 展开态打开(高级区直接铺开)

## 规则行末端按钮顺序

**检测 · 编辑 · 高级编辑 · 暂停(/恢复)· 删除**

## 实现要点

- form 体重排为 [基础 5 项] + `⚙高级功能`折叠按钮 + `#ruleAdv`(`display:none` 容器,
  内含全部高级字段)+ 提交按钮。**所有字段 id 零增减**(重组前后 id 集合 diff 为空),
  仅去掉 2 条纯装饰分隔线。
- 新增 `setRuleAdv(open)`(切 `#ruleAdv` 显隐 + 箭头 ▾/▴);`openAddRule`/`openEditRule`
  默认折叠,`openEditRule(id,true)` 展开。
- `applyRuleVis`、`fillRuleForm`、提交处理**未改**——它们引用的字段 id 全部仍在原处可达。

## 部署

- **仅要这次 UI**:把 `web/index.html` 覆盖到 `/opt/forward-panel/web/index.html`,浏览器硬刷新即可。
- **连同 v1.4.5 已消耗**:按常规流程重编(`release.sh`)+ 部署 binaries + web。
