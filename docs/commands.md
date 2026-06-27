# 常用命令

运维 APIC 时最常用的命令。带 `<尖括号>` 的按你的环境替换。

!!! note "服务名"
    节点服务固定为 `fp-agent`;面板服务名以你部署时设定为准(常见 `fp-master`)。不确定就先查:

    ```bash
    systemctl list-units --type=service | grep -iE 'forward|master|fp-|panel'
    ```

## 面板(master)

```bash
systemctl status fp-master           # 查看状态(服务名按实际)
systemctl restart fp-master          # 重启面板
systemctl stop fp-master             # 停止
systemctl start fp-master            # 启动
journalctl -u fp-master -f           # 实时日志
journalctl -u fp-master -n 200 --no-pager   # 最近 200 行
```

面板升级走后台「设置 → 更新 → 检查更新」,无需手动替换二进制。

## 节点(agent)

```bash
systemctl status fp-agent            # 查看 agent 状态
systemctl restart fp-agent           # 重启 agent
journalctl -u fp-agent -f            # 实时日志,排查接入 / 转发
```

**安装 / 升级节点**:推荐用面板「节点 → 新增节点」生成的一键命令,在节点服务器上以 root 执行。手动安装参考安装包内 `install.sh`(参数以面板生成的为准):

```bash
bash install.sh --server <面板地址> --key <节点密钥>
```

节点显示"需更新"时,**重新执行同一条安装命令**即可就地升级,规则保留。

## 排查

```bash
# 端口被谁占用(规则提示『端口被占用』时)
ss -tlnp | grep :<端口>

# 测试面板到更新中心的连通(公告 / 检查更新拉不到时)
curl -i https://<更新中心域名>/changelog
#   200 + JSON 数组 → 正常;连接错误 / 超时 → 出网或防火墙问题

# 测试节点到落地是否可达
curl -v telnet://<落地IP>:<端口>      # 或 nc -vz <落地IP> <端口>

# 看本机是否具备全局 IPv6(强制 IPv6 前自检)
ip -6 addr show scope global
```

## 日志位置速记

| 看什么 | 命令 |
| --- | --- |
| 面板运行 / 报错 | `journalctl -u fp-master -f` |
| 节点接入 / 转发 | `journalctl -u fp-agent -f` |
| 公告 / 更新拉取失败 | 面板日志(`journalctl -u fp-master`)+ `curl` 测更新中心 |

!!! tip
    大多数"规则不通"先看**入口节点的 agent 日志**;"面板异常 / 更新问题"看**面板日志**。

## 管理员密码重置

忘记管理员密码时,在**面板服务器**上用面板主程序重置(指定新密码):

```bash
systemctl stop fp-master                              # 服务名以实际为准(独占数据库,需先停)
# 指定新密码:
<面板二进制> -db <面板数据库路径> -reset-password <用户名> -new-password '<新密码>'
# 或不带 -new-password,改成交互输入(不进历史/进程列表):
#   <面板二进制> -db <面板数据库路径> -reset-password <用户名>
systemctl start fp-master
```

- `<面板二进制>` / `<面板数据库路径>`:用 `systemctl cat fp-master | grep ExecStart` 看实际路径与参数。
- 新密码至少 6 位;改完面板会自动退出,再启动即可,用新密码登录后建议在后台再改一次。

!!! note
    命令行直接带密码会进 shell 历史 / 进程列表,介意就用交互输入那种(不带 `-new-password`),或事后 `history -c`。
