# forward-panel v1.8.0 — 自研壳握手 utls 化(真 Chrome 指纹)+ 移除原生 QUIC 中转档

## 概述

两项改动,都在「入站→跳板」中继伪装层:

1. **自研壳(shell)客户端握手改走 utls** —— 线上呈现真 Chrome 的 ClientHello(JA3/JA4 与真 Chrome 一致),消除此前手搓 ClientHello 的指纹风险。**只影响自研壳档**;REALITY 档早已用 utls,不受影响。
2. **移除原生 QUIC 中转档(`relay_mode="quic"`)** —— 保留 `quic-obfs`(Salamander 混淆 QUIC)。明文 QUIC 在跨墙场景已不可靠(2026 起从 QUIC Initial 提 SNI 限速),跨墙统一走 quic-obfs。

## 改了什么

**① 自研壳握手 utls 化(`internal/kernel/shell.go`)**

`ShellClientHandshake` 重写:用 `utls.UClient(..., HelloChrome_Auto)` 生成真 Chrome ClientHello,`BuildHandshakeState()` 仅在内存构建(不做网络 I/O);从 `State13.KeyShareKeys.Ecdhe` 取标准 X25519(group 0x001d)私钥,其公钥与 hello 内 key_share 一致;认证 tag 写进 `SessionId` 及已序列化的 `Raw[39:]`。

因壳服务端不是真 TLS(手搓 ServerHello 后切 AES-GCM),**不调** `uConn.HandshakeContext`,而是自己写 ClientHello 记录 → 读壳服务端 ServerHello → ECDH → 切 AES-GCM。

**服务端无需改**:`tls_mimic.go` 的 `findX25519KeyShare` 本就遍历 key_share、只取 0x001d,自动跳过 Chrome 的混合 PQ 项 0x11ec,吃得下 utls 的 hello。旧手搓客户端路径(`buildClientHello`)成为死代码。

> 遗留(可选小加固):壳服务端 ServerHello 仍手搓(JA3S 非真,但 TLS1.3 下 ServerHello 之后即加密,被动指纹较弱)。若要再紧,可让壳在 ServerHello 后补几条仿 EncryptedExtensions/Finished 尺寸的加密记录再进 application_data。

**② 移除原生 QUIC 中转档**

删除 `relay_mode="quic"`(端口 base+4),保留 `quic-obfs`(base+5):
- `internal/kernel/layers.go`:删 `dialExit` 里的 quic dial 分支。
- `internal/kernel/relaytransport.go`:删 `relayPortOffset` 的 `case "quic"`(quic-obfs 仍为 5,base+4 端口闲置)。
- `cmd/agent/main.go`:删明文 QUIC 监听(quic-obfs 监听保留),修日志行。
- `cmd/master/server.go`:删 `relayPortFor` 的 `case "quic"`。
- `web/index.html`:删 `#f_enc`(relay_mode 下拉)的 quic 选项,清 `relayLabel`/`showSni` 的 quic 引用。

> 注:master 里的 `encrypt=="quic"`(`ruleTransport`/`validateTLSCert`/seed 自动生成)是**另一个字段**(`Encrypt`,纯转发下 = `none`),与 relay_mode 无关,本次未动。

## 构建(release.sh / apic-ship.sh)

无新增依赖(utls 早被 REALITY 档引入,`go.mod` 未变)。

```
./apic-ship.sh build      # 解压本包覆盖 ~/forward-panel → 编译 → 出 dist/
# 或
SIGN_PRIV=<私钥b64> ./release.sh 1.8.0
```

## 升级方式

- **agent + master 都要重编**(改了共用的 `internal/kernel`)→ 重部署。
- **web 可热替换**:前端改动只在 `web/index.html`,覆盖 `/opt/forward-panel/web/` 强刷即可,master 不必重启。
- **既有 `relay_mode="quic"` 规则改成 `quic-obfs` 或别的档** —— quic 档已移除,这类规则会握手失败。

## 兼容性

- **agent 协议(proto)未改**:已在线节点不受影响、不掉线。
- 自研壳的 seed 派生 / 认证 / 帧格式 / 端口(base+1)均未变,服务端解析逻辑未变 —— 升级后新旧壳客户端与服务端互通(差异仅在 ClientHello 的指纹形态)。
- `quic-obfs` 端口(base+5)、dial、监听、UI 全部保留,行为不变。
