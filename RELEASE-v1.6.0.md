# forward-panel v1.6.0 — 回滚「国内外分流」,装机走授权域名 + 离线安装卡片

## 背景(为什么回滚)
v1.5.7 / v1.5.8 引入的「国内外下载分流」是照搬 nyanpass 的**厂商-CDN**思路。但本产品与 nyanpass 有本质区别:它是**卖给客户自建面板**的——每个客户用自己的授权域名跑面板,节点连客户自己的面板,根本不存在统一的厂商分发源。该方向不适用,本版整体回滚到「装机即走面板授权域名」,只保留离线安装能力。

## 移除(回滚 v1.5.7 / v1.5.8 的全部分流相关)
- `DL_CN` / `DL_GLOBAL` 双下载源 + 并行判区(myip.ipip.net + api.ip.sb)
- `downloadBaseCN()`、`download_base` / `download_base_cn` 两个设置项
- `-set-download-base` / `-set-download-base-cn` / `-print-install-script` 三个 CLI 标志
- `/download/agent.sha256` 端点 + 装机脚本里的 sha256 校验
- `release.sh` 的 `agent.sha256` / `node-install.sh` / `fp-node-offline.tar.gz` 产出
- 文档中的 GitHub 托管 / 国内外分流章节

## 保留的架构检测
节点装机脚本(`installScript`)与面板安装器(`install.sh`)在安装前检测 CPU 架构,非 amd64(如 ARM)直接报错退出,避免装上 agent 跑不起来还查不出原因。在线 / 离线两种安装都生效。

## 现在的装机模型(单一授权域名)
三条装机命令(节点 / 分组 / 客户托管)全部回到单一下载源:

```
curl -fsSL <授权域名>/install.sh | MASTER='<域名>:9000' DL='<授权域名>' NODE=... TOKEN=... KEY=... sh
```

`DL` = 面板自身(配了域名走 `https://域名`,否则 `http://公网IP:8080`),即客户的授权域名。装机脚本、agent 下载、离线包,统统从客户自己的面板取。

## 离线安装(保留端点 + 新增卡片)
节点机器**不能出网**时用:

- **端点** `/download/offline-bundle`:主控现场打包 `agent + install.sh → tar.gz`(下发名 `fp-node-offline.tar.gz`),从**客户自己的面板**下载(公开、免鉴权,与 `/download/agent` 同级)。
- **卡片**:面板节点列表每行新增「**离线**」按钮 → 弹出卡片显示:
  1. 离线包下载地址(可一键下载 / 复制链接);
  2. 在节点上执行的命令(已**焊入该节点**的 `MASTER`/`NODE`/`TOKEN`/`KEY`,带 `OFFLINE=1`,`tar xzf` 解压后直接装,无需再填任何参数)。
- **流程**:管理员在能联网的机器下载离线包 → `scp` 到目标节点 → 在节点跑卡片里那条命令即可。脚本 `OFFLINE` 分支自动找解压出的同目录 `agent`,不联网。

## 找回管理员密码(既有,本版自带)
忘记管理员密码时,在主控机停面板后用 CLI 重置(新密码 ≥6 位):

```
master -db <你的库.db> -reset-password <用户名> -new-password <新密码>
```

或省略 `-new-password` 从标准输入读一行。重置后重新启动面板即可。无网页版「忘记密码」自助流程(管理面板按惯例走 CLI 恢复)。

## 升级方式
- **只需重建 master**:`SIGN_PRIV=<b64> ./release.sh 1.6.0`,用部署包替换主控。
- **web 可热替换**:本版前端改动(节点行「离线」按钮 + 离线卡片)只在 `web/index.html`。把新 `web/` 覆盖到 `/opt/forward-panel/web/` 后强刷浏览器即可,**无需停服**。
- **agent / hub / proto 未改动**:节点不需要重装、不掉线。

## 给客户的一句话提示
> 装机建议**用一个全新的子域名**(如 `panel.你的域名`)给面板用,与其它服务隔离。
