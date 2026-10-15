# Hugo + Stack 迁移与 GitHub Pages 发布教程

本教程记录 `onezjjj/github.io` 从 PaperMod 迁移到 Stack v4，并通过 GitHub Actions 发布到 GitHub Pages 的步骤。迁移后的工作副本位于 `/opt/hugo/joss-wiki-stack`；原目录 `/opt/hugo/joss-wiki` 保持不变。

## 1. 环境要求

- Hugo Extended 0.158.0 或更高版本。Stack v4.0.3 的模板使用 `.Site.Language.Locale`，该 API 从 Hugo 0.158.0 开始提供；仅按主题 `theme.toml` 标注的 0.157.0 检查还不够。
- Go 1.17 或更高版本用于解析 Hugo Module；本项目的 Actions 工作流设置 Go 1.26.x。
- GitHub Pages 的发布源需设为 GitHub Actions。
- 推送需要已授权的 GitHub SSH key，或其他受支持的 Git 凭据。不要将密码或访问令牌写入仓库、命令历史或教程。

本机迁移时构建的 Hugo Extended 二进制路径为 `/opt/hugo/bin/hugo`。

## 2. 保留原站并创建工作副本

将迁移放在新目录，先检查目标目录为空闲，再克隆现有仓库：

```bash
sudo mkdir /opt/hugo/joss-wiki-stack
sudo chown "$USER":"$USER" /opt/hugo/joss-wiki-stack
git clone https://github.com/onezjjj/github.io.git /opt/hugo/joss-wiki-stack
cd /opt/hugo/joss-wiki-stack
```

如果用 SSH 克隆，先确认当前用户已配置 GitHub SSH key，并按 GitHub 官方公布的指纹验证首次连接；不要关闭 SSH 主机校验。公开仓库可用 HTTPS 克隆，推送时再使用已授权的 SSH 身份：

```bash
git remote set-url origin git@github.com:onezjjj/github.io.git
```

迁移期间先确认工作树干净：

```bash
git status --short --branch
```

## 3. 引入 Stack v4 模块

所有 Hugo Module 命令都必须在新副本根目录运行：

```bash
hugo mod init github.com/onezjjj/github.io
hugo mod get github.com/CaiJimmy/hugo-theme-stack/v4@v4.0.3
hugo mod tidy
```

在 `hugo.toml` 中删除 `theme = "PaperMod"`，加入 Stack 模块，并配置中文、文章 section、侧栏与小组件：

```toml
baseURL = "http://localhost:1313/"
languageCode = "zh-cn"
defaultContentLanguage = "zh-cn"
hasCJKLanguage = true
title = "我的IT技术知识库"

[pagination]
  pagerSize = 10

[[module.imports]]
  path = "github.com/CaiJimmy/hugo-theme-stack/v4"

[params]
  mainSections = ["posts"]
  rssFullContent = true

[params.sidebar]
  subtitle = "Linux、网络与运维笔记"

[params.article]
  readingTime = true

[params.widgets]
  homepage = [
    { type = "search" },
    { type = "categories", params = { limit = 10 } },
    { type = "tag-cloud", params = { limit = 10 } },
  ]
  page = [{ type = "toc" }]
```

`mainSections = ["posts"]` 很重要：本站文章在 `content/posts/`，显式设置后首页才能列出文章。

移除旧 PaperMod 子模块：

```bash
git rm -f themes/PaperMod
```

这会同步移除 `.gitmodules` 中对应的 PaperMod 子模块条目。Stack 由 Hugo Module 管理，不需要再克隆到 `themes/`。

## 4. 添加 Stack 搜索页

搜索小组件需要一个 `layout = "search"` 页面，并输出 HTML 和 JSON。创建 `content/page/search/index.md`：

```toml
+++
title = "搜索"
slug = "search"
layout = "search"
outputs = ["html", "json"]

[menu.main]
  weight = 3

[menu.main.params]
  icon = "search"
+++
```

## 5. 更新 GitHub Actions

在 `.github/workflows/hugo.yml` 中：

- 将 Hugo Extended 版本设为 `0.158.0`。
- `actions/checkout` 不再需要递归检出主题子模块。
- 检出代码后设置 Go，例如：

```yaml
      - name: 设置 Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.26.x'
```

保留 GitHub Pages 的 `configure-pages`、Hugo 构建、artifact 上传和 deploy 步骤。生产构建应使用 Pages 提供的 base URL：

```bash
hugo --gc --minify \
  --baseURL "${{ steps.pages.outputs.base_url }}/"
```

仓库根目录的 `.gitignore` 增加 `/resources/`，避免把 Hugo 生成的编译缓存提交进去。

## 6. 本地构建验证

在 Linux 上确认使用 Extended 版本：

```bash
/opt/hugo/bin/hugo version
```

检查 Stack 模块并执行生产构建：

```bash
cd /opt/hugo/joss-wiki-stack
/opt/hugo/bin/hugo mod graph
HUGO_ENVIRONMENT=production /opt/hugo/bin/hugo --gc --minify \
  --baseURL https://onezjjj.github.io/github.io/ \
  --destination /tmp/joss-wiki-stack-site
```

本次构建使用 Hugo `0.158.0+extended`，生成 23 个页面；输出包含首页、`posts/ubuntu-nat/`、分类、标签以及 `/page/search/index.html` 和 `/page/search/index.json`。如果出现缺少 search page 的警告，检查 `content/page/search/index.md` 的 layout 和 outputs。

## 7. 提交、推送与验证发布

先检查差异和工作树：

```bash
git diff --check
git status --short
```

确认 Git 作者信息与远端认证后提交并推送：

```bash
git add -A
git commit -m "Switch site theme to Stack"
git push origin main
```

推送 `main` 会触发 `.github/workflows/hugo.yml`。到 GitHub 仓库的 Actions 页面确认本次 workflow 的 build 和 deploy 均成功，再打开：

- 网站：<https://onezjjj.github.io/github.io/>
- 本次迁移提交：`11564f5`
- 本次 Actions run：<https://github.com/onezjjj/github.io/actions/runs/38687040834>

此仓库是项目站点，正确地址带 `/github.io/`；只打开 `https://onezjjj.github.io/` 返回 404 并不表示此次发布失败。

## 8. 本次环境问题与处理

- 原目录由 root 拥有，普通用户不能写入；因此在 `/opt/hugo/joss-wiki-stack` 创建独立工作副本，避免改动原目录。
- 服务器用户的 SSH 身份与 root 的 GitHub SSH 身份不同。公开克隆使用 HTTPS，推送时切回已授权的 SSH remote；不要把账号密码传给 Git 或写进文件。
- 官方 Hugo `.deb` 下载速度很慢，直接在小容量 `/tmp` 编译时触发磁盘配额错误。改用磁盘空间较大的 `/opt/hugo` 临时目录和 Go 缓存构建了 Hugo Extended 0.158.0，最终二进制保存在 `/opt/hugo/bin/hugo`。
- Stack v4.0.3 的 `theme.toml` 标注最低 Hugo 0.157.0，但实际模板使用的 `Language.Locale` 从 0.158.0 才存在，因此 CI 采用 0.158.0。
- 原站目录最终保持干净，Stack 版本单独位于 `/opt/hugo/joss-wiki-stack`。
