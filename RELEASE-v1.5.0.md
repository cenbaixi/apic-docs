# forward-panel v1.5.0 — 更新公告改为 hub 集中发布(授权方→分销面板)

把 v1.4.9 误放在面板里的「更新说明发布」收归到 **hub(授权方中心)**:你在 hub 管理页发布一次,
随心跳下发到所有分销面板的「设置-更新」展示。卡网完全不碰。

**涉及两个服务,都要重编重部署:hub + forward-panel(master)。agent 不变(proto 仍 0.89)。**

## 架构

```
你 → hub /admin 网页(admin-token 登录)发布公告
        → 存 hub.db
        → 随 /heartbeat 响应下发最新 20 条(面板本来就连这条通道领授权)
   各分销面板心跳收下 → 缓存本地设置 changelog_json → 「设置-更新」只读展示(所有用户可见)
```

## hub 改动(新增)

- `store.go`:新表 `changelog(id,version,note,created_at)`。
- `changelog.go`:增删查方法 + 下发条数常量 `changelogDownlinkN=20`。
- `main.go`:
  - `/heartbeat` 响应体加 `changelog` 字段(下发最新 20 条);
  - `/admin/changelog` 的 Bearer-token API(GET 全部 / POST 新增 / POST {id,delete:true} 删除);
  - `/admin` 管理网页路由。
- `adminweb.go`(新):自包含发布页(admin-token 登录 + 发布表单 + 列表删除),
  页面本身不鉴权、数据 API 才鉴权(同 cardsvc admin 模式)。

## forward-panel 改动(改造 v1.4.9 的本地版)

- `heartbeat.go`:从 hub 响应解出 `changelog`,原样缓存到设置 `changelog_json`(旧 hub 不下发则保留上次)。
- `changelog.go`:`GET /api/changelog` 改为返回本地缓存的 hub 公告(只读);**删掉本地发布的 POST 与本地表**。
- `store.go`:去掉 `migrateChangelog`(本地不再建表)。
- 前端「设置-更新」:**删掉发布表单**,列表改只读、对所有用户展示(发布/删除权全在 hub)。

## 校验

- hub:gofmt 干净,4 文件语法 OK;管理页为 Go 原始字符串、内部 JS 全 '+' 拼接无反引号。
- 面板:gofmt 干净;死引用清零;`<div>` 415/415、form 12/12 平衡。

## 部署

1. **hub(dl.apic.gg / fp-hub.service)**:用更新后的源码重编 hub 二进制 → 重启。
   访问 `https://dl.apic.gg/admin`,填 hub 的 `-admin-token` 进入,即可发布公告。
2. **forward-panel**:`SIGN_PRIV=… ./release.sh 1.5.0` → 重编 master + 覆盖 `web/index.html` → 重启 master。
   (本次是 master+前端改动,agent 不动。)
3. 旧面板未更新前不显示公告(优雅降级);新面板更新后,下次心跳即拉到你在 hub 发的公告。

## 验证

- hub 管理页发一条 → 等一个心跳周期 → 任意分销面板「设置-更新」应出现该条;
- 面板列表只读、无发布/删除按钮;hub 管理页删除后,面板下次心跳同步消失(超出下发的 20 条外的不再下发)。
