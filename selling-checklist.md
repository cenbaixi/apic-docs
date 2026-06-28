# forward-panel 出售运营手册

从"开发完成"到"开卖 + 日常运营"的完整流程。按顺序走。

---

## 一、出售前一次性配置(只做一次)

在你**平时编译 master 的机器**上(有 Go + 完整源码)。

### 1. 生成授权密钥对

```bash
go build -o licensegen ./cmd/licensegen
./licensegen keygen
```

输出两段:
- **公钥** → 粘到 `cmd/master/license.go` 的 `licensePublicKeyB64`(替换占位值)。
- **私钥** → 存到只有你能碰的地方(密码管理器/离线)。**绝不进版本库、绝不上 hub、绝不给客户。** 这把私钥既签授权也签更新,泄露=整套授权体系作废。

### 2. 内置更新地址

```bash
./licensegen obf-url https://dl.apic.gg
```

输出一行混淆串 → 粘到 `cmd/master/update.go` 的 `updateBaseURLEnc`。

### 3.(可选)改前端落盘加密密钥

`cmd/master/shield.go` 的 `atRestSeed` 改成你自己的随机串。改了就别再改(否则旧 `.enc` 作废)。

### 4. 重新编译 master

```bash
go build -o master ./cmd/master
```

**此时起 master 才真正:强制授权(无有效 license.key 拒启动)+ 强制 HTTPS + 启用更新检查 + 启用心跳。** 之前是开发态。

### 5. 打第一个发布包并上传 hub

```bash
SIGN_PRIV=<你的私钥b64> ./release.sh 1.0.0
rsync -a dist/ root@dl.apic.gg:/opt/fp-hub/dist/
```

`release.sh` 会:编译 master+agent → 前端落盘加密 → 生成 `latest.json` + 私钥签名。`dist/` 传到 hub 后,客户就能拉更新。

> ⚠️ 验证:`./release.sh` 跑完,`cat dist/web/index.html.enc` 应是乱码(客户读不到前端);`dist/` 里应有 `latest.json` + `latest.json.sig`。

---

## 二、更新服务器 hub(已部署)

单独一台 `dl.apic.gg`,跑着 `fp-hub`。日常只需:

- **发新版本**:本机 `SIGN_PRIV=... ./release.sh <版本>` → `rsync -a dist/ root@dl.apic.gg:/opt/fp-hub/dist/`。
- **吊销名单**:`/opt/fp-hub/revoked.txt`,每行一个授权ID。
- 健康检查:`curl https://dl.apic.gg/heartbeat -X POST -d '{"license_id":"test"}'` → `{"revoked":false,...}`。

---

## 三、每个新客户上线

### 1. 客户部署(客户做)

客户拿到你私有分发仓库的访问权后:

```bash
git clone https://<TOKEN>@github.com/<你>/<私有分发仓库>.git
cd <仓库>
sudo ./deploy.sh
```

向导会问:**域名**、**授权码**(此时客户还没有,先随便填或留空让它失败也行)、**部署模式**(自动证书/反代)。

> 顺序上更顺:让客户先不填授权码,deploy.sh 会装好程序但 master 因无有效授权起不来 —— 这时正好用下一步拿指纹。

### 2. 客户取机器指纹(客户做,发你)

```bash
/opt/forward-panel/master -print-machine
```

输出一行指纹(sha256 hex)→ 客户发给你。

### 3. 你签发授权码(你做)

```bash
./licensegen sign \
  -priv <你的私钥b64> \
  -licensee "客户名称" \
  -domain 客户的域名.com \
  -machine <客户发来的指纹> \
  -days 365 \
  -out license.key
```

- `-domain` 必须和客户 deploy 时填的域名一致(激活锁会锁死,首次为准)。
- `-machine` 绑定到客户那台机器,换机器需重签。
- `-days 365` 一年期;到期客户面板拒启动,需续签。永久则去掉 `-days`。

把生成的 `license.key` 发给客户。

### 4. 客户装授权码 + 起服务(客户做)

```bash
cp license.key /opt/forward-panel/license.key
systemctl restart fp-master
```

master 验签通过即启动。**激活锁此刻锁定该授权码 + 域名,有效期内不可更换。**

### 5. 客户注册管理员(客户做)

立即打开 `https://客户域名` → 注册第一个账号(自动成为管理员)。**晚注册有被抢注风险,务必马上注册。**

上线完成。

---

## 四、日常运营

| 场景 | 操作 |
|---|---|
| 发新版本 | 本机 `SIGN_PRIV=... ./release.sh <版本>` → rsync 到 hub。客户面板「检查更新」→「应用更新」。 |
| 客户欠费/违约 | hub 的 `revoked.txt` 加一行该客户授权ID。客户面板 7 天宽限后锁定。 |
| 客户续费 | 从 `revoked.txt` 删掉那行。客户面板心跳后自动恢复(最多 15 分钟)。 |
| 授权到期续签 | 重新 `licensegen sign`(同指纹同域名,新 `-days`)→ 新 license.key 给客户替换。 |
| 客户换服务器 | 新机 `-print-machine` 取新指纹 → 重新签发(注意激活锁:换机等于换激活,需评估)。 |

---

## 五、客户侧故障处理

**更新后面板起不来** —— 回滚到上一版二进制:

```bash
cd /opt/forward-panel
cp master.bak master && systemctl restart fp-master
```

**看授权/吊销状态** —— 面板 设置→系统→授权信息卡片(客户名/域名/机器绑定/到期/本机指纹/是否被吊销)。

**日志** —— `journalctl -u fp-master -f`。

---

## 安全红线

- **私钥**:只在你的签发机,绝不进库/上 hub/给客户。
- **license.key**:每客户一份,机器+域名绑定,换机/换域名失效。
- 客户有完整 root 权限,理论上能 patch 二进制/删 DB 标记绕过授权与吊销 —— 这套是**抬高门槛**,真正后盾是:机器绑定 + 断更新 + 断支持 + 法律条款。别指望纯技术 100% 防破解。
