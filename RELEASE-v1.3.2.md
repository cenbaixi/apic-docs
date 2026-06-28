# forward-panel v1.3.2

新增:**cardsvc 后台「异常授权」管理**。master/agent 自 v1.3.1 起未变——本版只动 **hub + cardsvc 两个独立二进制**,各自部署(hub→dl.apic.gg,cardsvc→buy.apic.gg),不走 master 发布流程。

设计原则:**人用的管理入口只放在 cardsvc(已有管理鉴权),hub 保持极简、不长浏览器界面**。cardsvc 后台经服务端 server-to-server 调 hub 的 admin API(浏览器永远拿不到 hub 的 admin-token)。

## hub 改动(cmd/hub)

1. **持久化冲突观测**:`codes` 表加 `conflict_at` / `conflict_domain` 两列(迁移)。冲突原本只在 `onHeartbeat` 当场算、算完只刷 `last_seen`,不落库就无法在管理端列出"正在冲突"的码。现在冲突时写入 conflict_at=now、conflict_domain=入侵域名;迁移(旧域名失活自动改绑)和 `resetDomain` 时清零。
2. **异常分类**:`classifyIssues` 按记录算出标记(可多个):`conflict`(conflict_at 在 2h 内)、`expired`、`expiring`(7 天内到期)、`stale`(已授权但超 24h 无心跳)、`revoked`、`unauthorized`(有心跳但未售/未授权 = 跑着未付费的面板)。
3. **`/admin/codes` 增强**:支持 `?q=`(检索 code_id / 域名子串)+ `?abnormal=1`(只返回有异常标记的)。`CodeRow` 扩展字段并带 `issues`;`getCode` 也带上。

## cardsvc 改动(cmd/cardsvc)

1. 两个 hub helper:`hubResetDomain`(POST /admin/code/reset-domain)、`hubListCodes`(GET /admin/codes,支持 q/abnormal)。
2. 两个后台代理端点:`GET /admin/licenses`、`POST /admin/code/reset-domain`(吊销 / 续费已有代理)。
3. 后台新增「异常授权」选项卡:
   - 默认只显示**需处理**的授权(冲突 / 快到期 / 已过期 / 失联 / 已吊销 / 未授权),状态徽章彩色标注。
   - **检索框**:可检索**全部**域名 / 码ID(留空=只看异常)。
   - **处理交互键**(按异常类型给):冲突→「解绑域名」(reset-domain,你这次输错域名导致的双域名冲突就点这个);快到期/已过期→「续费」(顺延天数);已吊销→「解除吊销」,正常→「吊销」。每次操作后自动刷新。

## 部署

- **hub**:在 us-lax02 编译 `go build -o hub ./cmd/hub`,scp 到 dl.apic.gg 替换 `/opt/fp-hub/hub`,重启 hub 服务。迁移在启动时自动加列。
- **cardsvc**:编译 `go build -o cardsvc ./cmd/cardsvc`,scp 到 buy.apic.gg 替换,重启。确保启动参数有 `-hub https://dl.apic.gg -hub-token <hub-admin-token>`(已有,付款设期就靠它)。
- **master/agent**:本版无变化,无需重新发布;若想版本号统一,可按常规流程再发一次 v1.3.2(代码同 v1.3.1)。

## 注意

WebUI 只是让冲突**处理更快**,防不住"输错域名"这类**激活时**的笔误(发生在 master 激活那步)。要根治笔误,得在激活输入上做(域名自动探测 / 二次确认),那是另一处的事。
