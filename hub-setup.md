# 更新服务器(hub)部署

hub 做三件事:**托管更新文件** + **心跳/有效期管理** + **一码一域名防滥用**。建议单独一台便宜 Linux VPS(Ubuntu 22.04/24.04 LTS,1 核 1G 足够),配一个子域名如 `dl.apic.gg`。

> ⚠️ 块2 起 hub 有了 SQLite 数据库 `hub.db`(存所有授权码的到期/域名/吊销),**务必定期备份**(见第 7 节)。丢了它=所有授权的有效期/绑定状态丢失。

## 1. 编译(在你的构建机)

```bash
CGO_ENABLED=0 go build -o hub ./cmd/hub
```

把 `hub` 二进制传到服务器。

## 2. 目录布局(服务器上)

```
/opt/fp-hub/
  dist/          ← 把 release.sh 产出的 dist/ 整个放这里(latest.json、bin/、web/ …)
  hub.db         ← SQLite 授权码注册表(到期/域名/吊销)★务必备份★ 自动创建
  revoked.txt    ← 额外手动吊销名单(每行一个码ID;与 DB 吊销叠加;可不存在)
  certs/         ← autocert 模式自动生成
```

## 3. 起服务

**方式 A:hub 直接对外 HTTPS(autocert,需 80/443 空闲,域名已解析到本机)**

```bash
/opt/fp-hub/hub -data /opt/fp-hub -domain dl.apic.gg
```

**方式 B:前置 nginx 终结 TLS(推荐)**

```bash
/opt/fp-hub/hub -data /opt/fp-hub -addr 127.0.0.1:8090
```

nginx 反代(证书你自己 certbot):

```nginx
server {
    listen 443 ssl http2;
    server_name dl.apic.gg;
    ssl_certificate     /etc/letsencrypt/live/dl.apic.gg/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/dl.apic.gg/privkey.pem;
    client_max_body_size 200m;
    location / {
        proxy_pass http://127.0.0.1:8090;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

systemd(`/etc/systemd/system/fp-hub.service`):

```ini
[Unit]
Description=forward-panel update hub
After=network.target
[Service]
ExecStart=/opt/fp-hub/hub -data /opt/fp-hub -addr 127.0.0.1:8090 -admin-token 换成你的长随机令牌
Restart=always
RestartSec=3
[Install]
WantedBy=multi-user.target
```

> `-admin-token` 是管理 API 的钥匙(发卡系统/你的工具调用 `/admin/*` 用)。生成一个长随机串(如 `openssl rand -hex 24`),**别泄露**。不设则管理 API 全禁用(只剩静态托管 + 心跳)。

## 4. 把 master 指向 hub

```bash
licensegen obf-url https://dl.apic.gg      # 输出粘到 cmd/master/update.go 的 updateBaseURLEnc
# 重新编译 master(出售版)
```

之后客户面板「检查更新」就会从这台 hub 拉取。

## 5. 发布新版本

```bash
SIGN_PRIV=<你的私钥b64> ./release.sh 20260622-1
rsync -a dist/ root@dl.apic.gg:/opt/fp-hub/dist/     # 更新 hub 上的文件
```

## 6. 管理 API(发卡系统 / 你的工具调用)

所有 `/admin/*` 需带 `Authorization: Bearer <admin-token>`。下面用 curl 举例(地址换成你的 hub)。

**售出 / 设置到期**(发卡系统付款成功后调,或你手动):
```bash
curl -s https://dl.apic.gg/admin/code/set \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"code_id":"<码ID>","plan":"pro","days":30,"extend":false}'
# → {"ok":true,"code_id":"...","expires":1730000000}
```

**续费**(在现有到期基础上顺延,extend=true):
```bash
curl -s https://dl.apic.gg/admin/code/set \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"code_id":"<码ID>","days":30,"extend":true}'
```

**吊销 / 解除**(退款、盗用):
```bash
curl -s https://dl.apic.gg/admin/code/revoke \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"code_id":"<码ID>","revoked":true}'
```

**解绑域名**(客户合法迁移服务器,允许换新域名重新激活):
```bash
curl -s https://dl.apic.gg/admin/code/reset-domain \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"code_id":"<码ID>"}'
```

**查询单码 / 列表**:
```bash
curl -s "https://dl.apic.gg/admin/code/get?code_id=<码ID>" -H "Authorization: Bearer $TOKEN"
curl -s "https://dl.apic.gg/admin/codes?limit=100" -H "Authorization: Bearer $TOKEN"
```

> 码ID = `licensegen batch` 生成的 `code-ids.txt` 里每行逗号前那一段。
> 另:`/opt/fp-hub/revoked.txt` 仍可手动加码ID 吊销(与 DB 吊销叠加,二者其一即吊销)。

## 7. ★ 备份 hub.db ★

`hub.db` 存着所有授权的到期/域名/吊销状态,**丢了无法恢复**(客户会陆续到期暂停)。务必定期备份。最简单一条 cron:

```bash
# 每天 4 点备份,保留最近 14 份。crontab -e 加:
0 4 * * * sqlite3 /opt/fp-hub/hub.db ".backup '/opt/fp-hub/backup/hub-$(date +\%F).db'" && ls -t /opt/fp-hub/backup/hub-*.db | tail -n +15 | xargs -r rm
```

(先 `mkdir -p /opt/fp-hub/backup` 并 `apt install sqlite3`。)更稳妥的话再把 `backup/` 同步到另一台机器或对象存储。

## 8. 升级 hub(从旧版到块2)

旧 hub 没有 `hub.db`,首次以新二进制启动会**自动创建**空库,无需手动建表。流程:

```bash
CGO_ENABLED=0 go build -o hub ./cmd/hub      # 构建机重新编译
scp hub root@dl.apic.gg:/opt/fp-hub/hub      # 覆盖旧二进制
# 在 fp-hub.service 的 ExecStart 末尾加 -admin-token <长令牌>,然后:
systemctl daemon-reload && systemctl restart fp-hub
```

已激活的客户码会在下次心跳时被自动登记进 `hub.db`(首个域名即绑定)。你需要为**已售出的码**补设到期(用 `/admin/code/set`),否则它们的 expires=0 会被 master 端按未授权处理。
