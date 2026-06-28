# forward-panel v1.5.6 — 管理员密码重置 CLI(指定新密码)

仅 master。agent / hub / web 不变。

## 改动

主程序新增**密码重置子命令**,管理员忘记密码时在面板服务器本地重置(需停面板,独占数据库):

```bash
master -db <面板数据库> -reset-password <用户名> -new-password '<新密码>'
# 不带 -new-password 则从标准输入读取一行(不进 shell 历史/进程列表)
master -db <面板数据库> -reset-password <用户名>
```

- 复用已有的 `Store.SetPassword`(bcrypt);新增 `Store.UserByUsername` 按用户名查 ID。
- 处理在 `openStore` 之后、授权校验之前——**授权异常态也能重置**,改完即 `os.Exit(0)`。
- 新密码 < 6 位拒绝;找不到用户报错退出。

## 实现位置

- `cmd/master/main.go`:新增 `-reset-password` / `-new-password` flag + 处理块(import 增 `bufio` / `strings`)。
- `cmd/master/store.go`:新增 `UserByUsername`。

## 校验 & 部署

- gofmt 干净;语法 OK;无未用 import;`UserByUsername` 1 定义 1 调用。
- **重编 master** 即带此命令(无需 agent/hub/web 改动)。
- 用法见文档「常用命令 → 管理员密码重置」。
