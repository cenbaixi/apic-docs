# forward-panel v1.0.1 发布手册(完整版)

> v1.0.1 = 授权体验重构(无授权软启动 + 两层授权 + 转发/DNS/CDN 功能级封禁)+ 前端封禁横幅 + install.sh 重写。
> 三台机器:**apic**(编译/打包)、**ECS1253**(签名,签完即焚)、**hub**(dl.apic.gg,分发)。私钥永不上 apic。
> 从上到下照做即可。每步都有验证。

---

## 第0步 · 预检(apic,关键!跳过会被打回)

```bash
cd ~/forward-panel

# ① 确认三批源码都就位
sed -n '25p' cmd/master/license.go    # 须 = X6Wfg0dam9pYfjiOfsBxqUpf4FCY6n+wcbWkVQo6yOA=(真公钥)
sed -n '27p' cmd/master/license.go    # 须 = REPLACE_WITH...(占位符保持,licenseEnforced 才判生产态)
grep -n 'updateBaseURLEnc' cmd/master/update.go   # 须 = "DgRZBQNeAkAGCgMXQQ8TAxIX"(混淆URL)
grep -c 'licenseBlocked(); blocked' cmd/master/billing.go   # ≥1(Batch1 转发闸在)
grep -c 'id="licBanner"' web/index.html                     # =1(Batch2 横幅在)
grep -c '主控已启动' install.sh                              # ≥1(Batch3 新文案在)

# ② 清源码目录的运行时数据(release.sh 安全自检会因这些拒绝打包)
#    若你在源码目录跑过 master,会留下 panel.db / secrets.txt
rm -f panel.db panel.db-* 2>/dev/null || true
[ -f secrets.txt ] && shred -u secrets.txt 2>/dev/null || true
find . -maxdepth 2 \( -name 'panel.db*' -o -name 'secrets*' -o -name '*.pem' -o -name 'license.key' \) | grep -v '/dist/'
#    ↑ 应无输出。有输出就移走/删掉,否则 release.sh 第一步就 exit 1。

# ③ 确认 licensegen 在(没有就编一个)
[ -x ./licensegen ] || CGO_ENABLED=0 go build -o licensegen ./cmd/licensegen
```

> ⚠️ 别在 `~/forward-panel` 源码目录里跑完整 master 测试 —— 会生成 panel.db/secrets.txt 污染打包。要测在 /tmp 里测。

---

## 第1步 · 编译发布包(apic)

```bash
cd ~/forward-panel
# 不设 SIGN_PRIV —— 私钥不上 apic;清单单独在 ECS1253 签。传 1.0.1(脚本自动加 v 前缀)。
SIGN_PRIV= ./release.sh 1.0.1
```

release.sh 会:重编 master(含 Batch1)+ agent → 组装 web(并加密成 *.html.enc)+ install.sh + VERSION → 生成 dist.tar.gz。
因没设 SIGN_PRIV,它**跳过 latest.json**(正常,下一步手动签)。

**验证 dist/**:
```bash
ls -la dist/ dist/bin/
cat dist/VERSION                         # 1.0.1
head -c 200 dist/web/index.html.enc | xxd | head   # 应是乱码(前端已加密,客户读不到源码)
ls dist/install.sh dist/bin/master dist/bin/agent dist/dist.tar.gz   # 都在
```

---

## 第2步 · 生成更新清单(apic)

dist.tar.gz 是给 install.sh 用的整包,**不该进** master 自更新清单。release.sh 最后才生成它,所以现在它在 dist/ 里 —— **手动 make-update 前必须先移出**:

```bash
cd ~/forward-panel
mv dist/dist.tar.gz /tmp/dist.tar.gz                         # 移出
./licensegen make-update -dir dist -version 1.0.1           # 产出 dist/latest.json
```

**验证 latest.json**:
```bash
grep '"version"' dist/latest.json                            # "version": "1.0.1"
grep -c 'dist.tar.gz' dist/latest.json                       # 0(确认整包没进清单)
grep -c '"path"' dist/latest.json                            # 文件数(master/agent/web各文件/install.sh/VERSION等)
```

---

## 第3步 · 签名清单(ECS1253 —— 私钥最后一次使用)

> 这是 ECS1253 销毁前的最后任务。私钥从 Bitwarden 取,`read -s` 不回显,签完 `unset`。

```bash
# —— 在 apic:把待签清单送过去 ——
scp dist/latest.json root@203.0.113.10:/root/latest.json

# —— 在 ECS1253:签 ——
cd /root
read -s PRIV; echo "${#PRIV}"            # 粘贴私钥后回车;应显示 88
# 确认 ECS1253 上有 licensegen(没有就从 apic scp 一个过来:scp licensegen root@203.0.113.10:/root/)
./licensegen sign-update -priv "$PRIV" -in latest.json -out latest.json.sig
unset PRIV
ls -l latest.json.sig                     # 存在(ed25519 签名,base64,约 88 字节)

# —— 在 apic:取回签名 ——
scp root@203.0.113.10:/root/latest.json.sig dist/latest.json.sig
```

---

## 第4步 · 推送到 hub(apic)

```bash
cd ~/forward-panel
mv /tmp/dist.tar.gz dist/dist.tar.gz                         # 移回整包
rsync -av dist/ root@dl.apic.gg:/opt/fp-hub/dist/           # 整个 dist 上 hub
```

---

## 第5步 · 验证 hub(apic 或任意机器)

```bash
curl -fsSL https://dl.apic.gg/VERSION                        # 1.0.1
curl -fsSL https://dl.apic.gg/install.sh | head -12         # 新版头部(含"下载后运行,交互最稳")
curl -fsSL https://dl.apic.gg/latest.json | grep version    # "version": "1.0.1"
curl -fsSI https://dl.apic.gg/latest.json.sig | grep -i content-length   # 有(签名在)
curl -fsSI https://dl.apic.gg/dist.tar.gz | grep -i content-length       # 有(整包在)
```

五项都对 = **v1.0.1 上线,新客户一键装即得功能级授权版**。

---

## 第6步 · ★ 自更新机制端到端验证(你计划里"最后验证 update")

> 背景:PLAN.md 第七节原计划用**线路检测体系**功能当更新测试载荷 —— 即做完检测体系发个新版,让老版客户机点"检查更新→应用更新",验证整条自更新链路。检测体系还没做,但**任何版本号递增都能当测试**。
>
> 自更新链路:master 拉 `latest.json` + `.sig` → 内嵌公钥验 ed25519 签名 → 比对版本 → 更新则下载变更文件 → 替换 → 重启。**从没真正端到端测过,卖之前测一次最稳妥。**

**测试要点**:需要一台跑**更低版本** master 的机器,去拉更高版本。两种做法:

### 做法A(推荐,现在就能测):发一个版本号更高的测试版
ECS1253 现在是 v1.0.1。在 apic 重打一个 v1.0.2(代码不动,仅版本号 +1),走第1~5步发到 hub(version 全改 1.0.2)。然后:
```bash
# 在 ECS1253 的面板(https://test.apic.gg,若域名"#"打不开,见下方注)
# 设置 → 系统 → 检查更新 → 应显示「有新版本 1.0.2」→ 点「应用更新」
# master 自动:下载新二进制 → 验签 → 替换 → 重启
journalctl -u fp-master -n 20 --no-pager     # 看「更新完成」「重启」类日志
# 重启后面板「授权信息」卡片版本应显示 v1.0.2
```
看到 ECS1253 从 v1.0.1 自己变成 v1.0.2 + 进程正常 = **自更新机制验证通过**。
v1.0.2 和 v1.0.1 代码相同(仅版本号),留它当 current 也无妨;或测完再发一次 v1.0.1 覆盖。

### 做法B(等真功能):做完检测体系(或任何下个功能)作为 v1.1.0 发布
那时 v1.0.x 的真实客户点检查更新→应用更新升到 v1.1.0,就是最真实的一次更新测试。

> 注:ECS1253 域名被污染成 "#"(旧装机粘贴 bug),HTTPS 打不开,面板点不了"检查更新"。要测做法A,要么用**域名正常的新测试机**(装 v1.0.1 后发 v1.0.2 测更新),要么先修 ECS1253 的域名(清 panel.db 重装,但激活锁也会重置)。**最干净:开台新测试机,真域名,装 v1.0.1 → 发 v1.0.2 → 测自更新。** 这台同时也是第6步前那个"端到端验证install.sh"的机器,一鱼两吃。

---

## 第7步 · 收尾

```
□ 自更新验证通过后,销毁 ECS1253(私钥只在 Bitwarden,机器可焚)
□ 清测试数据:
    · cardsvc 后台:删订单 #1、测试码 t000–t019、测试工单
    · hub:把误绑"#"域名的 <code_id> 解绑 →
      curl -X POST https://dl.apic.gg/admin/code/reset-domain \
        -H "Authorization: Bearer <admin-token>" \
        -d '{"code_id":"<your_code_id>"}'
□ GitHub 推送源码备份(确认无二进制/密钥/db):
    cd ~/forward-panel
    git status                          # 看清单
    git add -A
    git status                          # 再确认无 master/agent/*.key/panel.db/secrets/dist
    git commit -m 'release v1.0.1: 授权功能级封禁 + 前端横幅 + install.sh 重写'
    git push
□ 备份 hub.db(在 hub 机:cp /opt/fp-hub/hub.db /opt/fp-hub/hub.db.bak.$(date +%F))
```

---

## 附:本次 v1.0.1 关键行为(给你自己记)

- **无授权**:master 照常启动、面板可登录注册、顶部红横幅、转发/DNS/CDN 停。
- **补码+域名校验通过**:全功能恢复,面板显示到期时间。
- **到期(hub)**:30 秒内(billingLoop)转发/CDN 撤下;续费后自动补发。
- **域名不符(重装)**:软处理 —— 启动+功能停+提示用原域名(不再崩)。
- **两层**:管理员授权(license.key)失效→全平台停;客户计费到期→只停该客户。
