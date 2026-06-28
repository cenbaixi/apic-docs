# forward-panel v1.3.3

修复 v1.3.0 引入的回归:**有名字的规则导致规则列表整表渲染崩溃,显示"还没有规则"**。仅改 `web/index.html`(前端单行),master/agent 二进制无需重编。

## 真因

v1.3.0 的"规则 ID 列改为名字优先"(`${r.name?`<b>${esc(r.name)}</b>`:...}`)在 `loadRules` 里调用了 `esc()`。但 `esc` 在本代码库中是**各渲染函数内部各自定义的局部 `const`**(`nodeRow` 等 9 处都各有一份),并非全局函数——而 `loadRules` 函数体内没有定义它。

于是:任何**有 `name` 的规则**都会走 `r.name ? esc(r.name) : ...` 的真分支 → 调用未定义的 `esc` → 抛 `ReferenceError: esc is not defined` → 整个 `rules.map(...)` 中断 → `tb.innerHTML` 未赋值,停留在上一帧的"还没有规则"空状态。规则每秒轮询,故 Console 持续报错。

排查中依次否掉的错误假设(留作记录):license 吊销(宽限期内不 gate)、`ListRules` 的 user_id 过滤(登录的就是 admin、规则也属 admin)、master 读错库(`-db` 与 `find` 均指向同一个 `/opt/forward-panel/panel.db`)、`ListRules` 运行时报错(日志无 `apiRules` 行)。最终靠 F12 抓 `/api/rules` 返回 200 + ~621B(非空)+ Console 红错定位到前端渲染。

## 修复

`loadRules()` 函数体第一行补一行局部 `const esc`(与 `nodeRow` 等其它渲染函数完全一致的写法):

```js
const esc=s=>String(s==null?'':s).replace(/[&<>"]/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]));
```

改一处,零波及。

## 部署

只是前端。重新 `release.sh` 出包(会重新加密 `index.html.enc`),装到 test2 或推存量客户。规则列表即恢复;有名字的规则显示加粗名字,无名字的显示淡灰 ID。
