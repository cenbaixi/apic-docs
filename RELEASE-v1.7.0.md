# forward-panel v1.7.0 — 多架构(amd64 / amd64v3 / arm64),含离线安装

## 概述
master 与 agent 现在都出三种架构,装机 / 离线 / 分发自动按架构选包。此前只发 amd64,ARM 机器被装机脚本直接拒绝。

- **amd64**(基线,GOAMD64=v1):兼容所有 x86-64,含内地老 / 阉割 CPU(不支持 AVX2 的 VPS)。
- **amd64v3**(GOAMD64=v3):2013+ 现代 x86,启用 AVX2 / BMI2 / FMA 等,更快。
- **arm64**:ARM 服务器(Ampere / AWS Graviton 等)。

## 构建(release.sh)
每个二进制各出三份:`master` / `master-amd64v3` / `master-arm64`,`agent` / `agent-amd64v3` / `agent-arm64`。`CGO_ENABLED=0` 纯 Go 交叉编译,arm64 无需额外工具链。`dist.tar.gz` 含全部六个二进制。

```
SIGN_PRIV=<私钥b64> ./release.sh 1.7.0
```

## 节点装机:自动判架构
装机脚本 `uname -m` 判 arm64;x86 再查 `/proc/cpuinfo` 是否同时有 `avx2`+`bmi2`+`fma` → 走 amd64v3,否则 amd64(基线)。

- **在线**:`curl <授权域名>/download/agent?arch=<架构>`。面板缺 amd64v3 包→回退基线 amd64(v3 CPU 上能跑);缺 arm64 包→明确报错(不拿 x86 顶替)。
- **下载端点** `/download/agent?arch=amd64|amd64v3|arm64`:面板按 `-agent-bin` 同目录的 `agent` / `agent-amd64v3` / `agent-arm64` 分发。

## 离线安装:两种出包方式
离线卡片新增「目标节点架构」下拉:

- **自动 · 全部架构**(默认):一个含全部架构 agent 的大包(~50M+),拷到哪台都行,节点上 `install.sh` 自动判架构挑对应二进制。`/download/offline-bundle`。
- **指定架构**:按需出单架构小包(~15-22M),`/download/offline-bundle?arch=<架构>`。需先知道节点架构;amd64v3 缺则回退 amd64。

包内 agent 命名 `agent-amd64` / `agent-amd64v3` / `agent-arm64`;节点上 `install.sh`(OFFLINE 分支)按本机架构挑:arm64 只用 agent-arm64(绝不拿 x86 顶);amd64v3 缺则回退 amd64;并兼容老的单 `agent` 包。下发文件名仍是 `fp-node-offline.tar.gz`,与卡片命令对齐。

## 面板(master)自身多架构
`install.sh` / `deploy.sh` 装面板时按本机 CPU 选 master:arm64 → `master-arm64`,现代 x86 → `master-amd64v3`,否则基线 `master`。amd64v3 缺则回退基线;arm64 缺则报错。同时把三种架构的 agent 全部装到 `/opt/forward-panel/`,供面板按节点架构分发。

## 升级方式
- **重建 master**:`SIGN_PRIV=<b64> ./release.sh 1.7.0`,推到 hub。客户重装 / 更新面板即拿到多架构。
- **web 可热替换**:前端只在 `web/index.html`(离线卡片加了架构下拉),覆盖 `/opt/forward-panel/web/` 强刷即可。
- **agent 协议(proto)未改**:已在线的节点不受影响、不掉线;换架构二进制不影响已装节点,新装 / 重装节点才走按架构分发。

## 兼容性
- 老的单架构(只 amd64)发布包 / 离线包仍可用(脚本兼容裸 `agent` 文件)。
- 面板 `-agent-bin` 指向基线 `agent` 不变;多架构二进制按同目录 `-<架构>` 后缀约定。
