# APIC 文档站(mkdocs-material)

APIC 面板的使用文档,基于 [MkDocs](https://www.mkdocs.org/) + [Material](https://squidfunk.github.io/mkdocs-material/),推送到 `main` 自动构建并发布到 GitHub Pages。

## 目录

```
mkdocs.yml                  # 站点配置(主题 / 导航 / 扩展)
requirements.txt            # 依赖:mkdocs-material, jieba(中文搜索)
.github/workflows/deploy.yml# 推 main 自动部署到 gh-pages
docs/
  index.md                  # 介绍
  guide/install.md          # 安装部署
  guide/usage.md            # 线路 · 节点 · 规则
  guide/update.md           # 更新升级
  faq.md                    # 常见问题
  reference.md              # 参考
  commands.md               # 常用命令
```

## 本地预览

```bash
pip install -r requirements.txt
mkdocs serve            # 打开 http://127.0.0.1:8000 实时预览
mkdocs build --strict   # 构建并校验内部链接(发布前自检)
```

## 发布到 GitHub Pages

1. 新建一个仓库(建议独立库,如 `apic-docs`),把本目录推上去:

    ```bash
    git init && git add -A && git commit -m "docs: init APIC docs"
    git branch -M main
    git remote add origin git@github.com:<你的账号>/apic-docs.git
    git push -u origin main
    ```

2. 推送后,`.github/workflows/deploy.yml` 会自动跑 `mkdocs gh-deploy`,把构建产物推到 `gh-pages` 分支。
3. 仓库 **Settings → Pages → Build and deployment → Source 选 `Deploy from a branch`,分支选 `gh-pages` / `/ (root)`**,保存。
4. 等 Pages 生效,访问 `https://<你的账号>.github.io/apic-docs/`。

## 改成自己的地址

- `mkdocs.yml` 里 `site_url` / `repo_url` / `repo_name` 改成你的。
- 想用自定义域名(如 `docs.apic.gg`):在仓库 Pages 设置里填 Custom domain,并在 `docs/` 下放一个 `CNAME` 文件写上该域名。

## 待补占位

正文里带 `<尖括号>` 的是需要按你环境替换的值:面板域名、更新中心域名、面板服务名、节点安装命令。搜索 `<` 可快速定位。
