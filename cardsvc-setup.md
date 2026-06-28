# 发卡服务器(cardsvc)部署

cardsvc 是独立发卡服务:码池 + 商品 + 下单 + USDT(TRC20/Polygon)收款 + 自动发码 + 发卡网页 + Telegram bot。
单台便宜 Linux VPS 即可(1 核 1G 够),配子域名如 `buy.example.com`。★ 不放源码、不装 Go,只跑二进制 ★

## 1. 在 apic 编译,拷二进制过来

```bash
# apic(源码机)
cd ~/forward-panel
CGO_ENABLED=0 go build -o cardsvc ./cmd/cardsvc
scp cardsvc root@<发卡服务器IP>:/opt/cardsvc/cardsvc.new
```

```bash
# 发卡服务器
mkdir -p /opt/cardsvc && cd /opt/cardsvc
mv cardsvc.new cardsvc && chmod +x cardsvc
```

## 2. 准备配置(几样东西先备好)

- **hub 地址 + hub 的 admin-token**(cardsvc 调 hub 设到期用,就是 fp-hub 的 -admin-token)。
- **USDT-TRC20 收款地址** + (建议)TronGrid API Key。
- **USDT-Polygon 收款地址** + (建议)Polygonscan API Key。
- **Telegram bot token**:找 @BotFather 发 /newbot 创建,拿到 token。
- **你的 Telegram 用户ID**(管理员):找 @userinfobot 发任意消息即得数字 ID。

## 3. 导入码池 + 上架商品

```bash
# 把你本地签好的 codes.txt 传上来(scp),然后:
/opt/cardsvc/cardsvc import -db /opt/cardsvc/card.db codes.txt

# 上架商品(price 单位 USDT;码统一 plan=standard,用商品区分时长)
/opt/cardsvc/cardsvc addproduct -db /opt/cardsvc/card.db -name 月卡 -days 30  -price 5.00
/opt/cardsvc/cardsvc addproduct -db /opt/cardsvc/card.db -name 季卡 -days 90  -price 13.00
/opt/cardsvc/cardsvc addproduct -db /opt/cardsvc/card.db -name 年卡 -days 365 -price 45.00
```

> 测试阶段没有真签的码,可造测试码试流程(cardsvc 只取 id 不验签):
> `python3 -c "import json,base64;[print(base64.urlsafe_b64encode(json.dumps({'id':'t%03d'%i,'plan':'standard','hub':True}).encode()).rstrip(b'=').decode()+'.test') for i in range(20)]" > codes-test.txt`

## 4. systemd 起服务

`/etc/systemd/system/cardsvc.service`:

```ini
[Unit]
Description=forward-panel card service
After=network.target
[Service]
WorkingDirectory=/opt/cardsvc
ExecStart=/opt/cardsvc/cardsvc serve \
  -db /opt/cardsvc/card.db \
  -addr 127.0.0.1:8070 \
  -hub https://dl.apic.gg \
  -hub-token <hub的admin-token> \
  -admin-token <cardsvc自己的管理令牌> \
  -trc20-addr <USDT-TRC20收款地址> \
  -trc20-apikey <TronGrid-Key> \
  -polygon-addr <Polygon-USDT收款地址> \
  -polygon-apikey <Polygonscan-Key> \
  -tg-token <Telegram-bot-token> \
  -tg-admins <你的TG用户ID>
Restart=always
RestartSec=3
[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now cardsvc
systemctl status cardsvc --no-pager | head
```

> `-admin-token` 是 cardsvc 自己的管理令牌(用于 /admin/markpaid、/admin/pool 测试接口);生成:`openssl rand -hex 24`。
> 收款地址/Key 任填一条链也行(只填 trc20 则只卖 TRC20)。

## 5. nginx 终结 TLS（发卡网页对外）

```nginx
server {
    listen 443 ssl http2;
    server_name buy.example.com;
    ssl_certificate     /etc/letsencrypt/live/buy.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/buy.example.com/privkey.pem;
    location / {
        proxy_pass http://127.0.0.1:8070;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

(证书用 certbot 申请。)然后访问 `https://buy.example.com` 就是发卡网页。Telegram bot 启动后直接在 TG 里 /start 即可。

## 6. ★ 备份 card.db ★

card.db 存订单 + 码池售卖状态,务必定期备份:

```bash
apt install -y sqlite3 && mkdir -p /opt/cardsvc/backup
# crontab -e:
0 4 * * * sqlite3 /opt/cardsvc/card.db ".backup '/opt/cardsvc/backup/card-$(date +\%F).db'" && ls -t /opt/cardsvc/backup/card-*.db | tail -n +15 | xargs -r rm
```

## 7. 安装命令里的 hub 地址

发卡网页/bot 发码时附带的安装命令是 `curl -fsSL <hub>/install.sh | sudo bash`,其中 `<hub>` 取自 `-hub` 参数。确保 hub 上 `install.sh` 里的 HUB 常量也指向同一 hub 地址(见 install.sh 顶部)。

## 测试流程(部署后自检)

1. 浏览器开 `https://buy.example.com` → 看到商品 → 下单 → 拿到收款地址+金额+订单号+查询密钥。
2. （测试)用 cardsvc 的 /admin/markpaid 手动标记已付,或真转账等自动到账。
3. 「查单/取卡」凭订单号+密钥 → 看到授权码 + 安装命令。
4. TG 里 /buy → 选套餐选链 → 建单;/orders 取卡;管理员 /pool 看码池。

## 依赖:二维码库(收款码)

发卡页 /qr 接口与 Telegram bot 收款图都用 `github.com/skip2/go-qrcode`(纯 Go,无 CGO)。
首次编译前在 apic 源码目录执行一次:

```bash
cd ~/forward-panel
go get github.com/skip2/go-qrcode@latest
go mod tidy
```

之后正常 `CGO_ENABLED=0 go build -o cardsvc ./cmd/cardsvc` 即可。

## 后台自定义入口(隐藏 /admin)

后台「概览」底部"后台访问入口"卡片可设自定义路径(如 manage-x7k9):
- 设置后,后台只能从 `本站/manage-x7k9` 进入,`/admin` 返回 404 隐藏。
- 忘记路径可用 token 恢复:`curl -X POST https://buy.apic.gg/admin/setpath -H "Authorization: Bearer <admin-token>" -H "Content-Type: application/json" -d '{"path":""}'`
- 配合自有短域名整站反代 buy.apic.gg,即可用短域名 + 该路径进后台。

## 工单附件目录

附件存 `/opt/cardsvc/uploads/`(随机文件名),启动自动创建。如需自定义,serve 加 `-uploads /your/path`。
cardsvc 以 root 运行可直接读写;若用 systemd 沙箱(ProtectSystem 等),需把该目录加入 ReadWritePaths。
结单超 7 天的附件文件每小时自动清理一次(对话文本与文件名记录保留)。后台工单详情有"清理附件"可手动立即清。
