# forward-panel v1.5.5 — 直连不显示出口源 IP 悬空提示 + egress 模式选项更名

纯前端(`web/index.html`)。master / agent / hub 均不改,字段语义不变。

## 1 · 直连规则不再显示出口源 IP 悬空提示

规则「高级编辑」那条提示("出口源 IP 须在出口机上可用(群组出口则每台组员都要有);强制 IPv6 / 绑定源 IP 需该机本身具备。")原先高级模式下无条件显示,但它描述的出口源 IP 框(`exEgressBox`)在直连规则(未选出口)时是隐藏的 → 提示悬空。

现给提示加 `id="exEgressHint"`,在 `applyRuleVis` 里跟 `exEgressBox` 一起按 `hasExit` 显隐:选了出口才显示,直连时连框带提示一并隐藏。

通查整个规则 modal,其余提到"出口/中转/落地"的文案均已正确处理(PROXY 提示解释的是始终存在的选项,入口出站框头对直连成立,隧道伪装类与出口框本就按 `hasExit`/伪装档类型 gating),只有这一条是真悬空,已修。

## 2 · egress 出站模式选项更名(入口 + 出口两个 select)

`f_egmode` / `f_exegmode` 的选项更名,语义不变:

- `跟随系统` → `跟随系统(双栈 / Dual-stack)`(Go 的 "tcp" 本就是双栈、优先 v6 回落 v4,所以 Dual-stack 即此项,不单列)
- `强制 IPv4 出` → `IPv4-only`(value=4,硬走 tcp4)
- `强制 IPv6 出` → `IPv6-only`(value=6,硬走 tcp6)

出口多 IP 策略(轮询/随机)select 保留——它是真控件(多 IP 出口摊出站、躲目标按源 IP 限流),且已在出口框内、仅选出口才显示。需要的话可后续收进高级 JSON(`exit_egress_policy`)。

## 校验 & 部署

- `<div>` 422/422、select 51/51 平衡;提示 gating 链路正确;旧文案清零。
- **仅热替换 `web/index.html` + 浏览器强刷**,无需重编任何二进制。
- 验证:① 直连规则高级编辑 → 无出口源 IP 提示;选了出口的规则 → 出口框 + 提示正常。② 两个出站模式 select 都显示 跟随系统(双栈 / Dual-stack)/ IPv4-only / IPv6-only。
