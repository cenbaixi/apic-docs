# forward-panel v1.5.4 — 公告拉取失败诊断日志 + 分组表节点列合并

master + 前端。hub 不变(仍需 v1.5.2 的 `/changelog`)。agent 不变。

## 1 · 「点击更新最新公告拉取不到」——诊断

**代码已核对无误**:面板 `fetchHubChangelog` 用的就是和 latest.json 同款的 `http.Client.Get`,URL 同源(`updateBaseURL()/changelog`)。既然 latest.json 能拉到,`/changelog` 在代码层面就该能拉到。

**真因几乎可锁定在 hub 侧**(二者其一):
1. **hub 二进制没重新部署 v1.5.2**——`/changelog` 端点是 v1.5.2 加的,如果 hub 还是旧二进制,这个路由根本不存在,返回 404;
2. **nginx 没把 `/changelog` 转发给 hub**——如果 nginx 用的是按路径的 location 块(只放行 /heartbeat、/latest.json 等),`/changelog` 会被漏掉。

**本版改动**:给 `fetchHubChangelog` 加失败日志(原来静默回落缓存,无从排查)。现在失败会打到 master 日志,带 URL 和原因:
- `[changelog] 拉取 <url> 返回 HTTP 404(hub 是否已部署 /changelog?nginx 是否转发?)`
- `[changelog] 拉取失败 <url>: <连接错误>`
- `[changelog] <url> 返回非 JSON 数组(前 80 字符):<内容>`(说明被 nginx 错误页/反代拦了)

**你需要做的排查**(二选一即可定位):
- 直接 curl:`curl -i https://dl.apic.gg/changelog`
  - 返回 JSON 数组 `[...]` → hub 正常,问题在面板侧(看 master 日志);
  - 返回 404 / nginx 错误页 → **重编重启 hub**(v1.5.2 起含 `/changelog`),或修 nginx 把 `/changelog` 转发给 hub。
- 看 master 日志里上面那几行,按提示处理。

> 注:hub 的 `/changelog` 自 v1.5.2 起就有,本版未改 hub。如果你之前只更新了 master 没重部署 hub,就是原因 1。

## 2 · 分组表:合并「节点数/在线」为「节点」列 + 查看键移到操作列末

- 「节点数」「在线」两列(及原先放节点名/查看键的「节点」列)**合并为一个「节点」列**,显示 **在线/总数**(如 `1/2`);
- 「查看」功能键**移到操作列**,操作列顺序固定为:**改名、查看、移层、删除**;
- 未分组行同样显示 `在线/总数`,操作列只给「查看」(可把未分组节点批量并入分组);
- 删除了不再使用的 `grpNodeCell`(非管理员现在看到的是节点计数而非节点名)。

分组表列:分组(线路) / 等级 / 倍率 / 节点 / 操作。

## 校验

- gofmt 干净(changelog.go 加了 log import、update.go、hub/main.go);
- `<div>` 422/422、select 51/51 平衡;grpNodeCell 清零;ops 顺序与表头正确;colspan 5/4。

## 部署

- **master**:重编(changelog.go 日志)+ 覆盖 `web/index.html`(分组表,可热替换强刷)。
- **hub**:本版不改;若公告仍拉不到,按上面第 1 节排查(多半是 hub 没部署 v1.5.2 或 nginx 没转发 `/changelog`)。
- agent:不动。
