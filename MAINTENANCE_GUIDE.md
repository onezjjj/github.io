# Stack 博客日常维护指南

适用于 `/opt/hugo/joss-wiki-stack`，当前使用 Hugo Stack v4、Hugo Extended 0.158.0，并通过 GitHub Actions 发布到 GitHub Pages。

## 1. 准备环境

进入项目目录并确认 Hugo Extended 版本：

```bash
cd /opt/hugo/joss-wiki-stack
/opt/hugo/bin/hugo version
```

应看到 `0.158.0+extended` 或更高版本并包含 `extended`。系统自带的 Hugo 0.154.5 低于 Stack 模板实际要求，不要用它构建本主题。

## 2. 新增文章

用 Hugo 默认 archetype 创建文章：

```bash
/opt/hugo/bin/hugo new content posts/linux-network-notes.md
```

文件会生成在 `content/posts/linux-network-notes.md`。默认 front matter 中 `draft = true`，编辑完成、准备发布时将它改为 `false`。本项目文章 front matter 使用 TOML，示例：

```toml
+++
title = 'Linux 网络笔记'
date = 2026-10-15T12:00:00+08:00
draft = false
tags = ['Linux', '网络']
categories = ['网络技术']
+++

## 标题

在这里编写文章正文。正文支持 Markdown。
```

日期可使用当前时间和时区。文章文件名决定默认 URL slug；发布后尽量不要改名，以免旧链接失效。若确需更改，考虑配置 Hugo alias 或保留旧地址重定向。

## 3. 修改文章

直接编辑 `content/posts/` 下对应的 Markdown 源文件，例如：

```text
content/posts/ubuntu-nat.md
```

修改标题、正文、日期或 front matter 后保存即可。不要手动编辑 `public/`：它是构建产物，会在下次 Hugo 构建时重新生成。

如果暂时不希望文章公开，将 `draft` 设为 `true`。本地预览时可以通过 `--buildDrafts` 显示草稿；正式 GitHub Pages 构建默认不会发布草稿。

## 4. 添加分类和标签

分类与标签在文章 front matter 中设置，不需要修改 `hugo.toml`。

单分类、多个标签：

```toml
categories = ['网络技术']
tags = ['Ubuntu', 'Linux', 'NAT']
```

文章可属于多个分类，也可添加多个标签：

```toml
categories = ['网络技术', 'Linux']
tags = ['Ubuntu', '路由', 'NAT']
```

同名分类或标签会自动归到同一个 taxonomy 页面；新名称会在构建后自动生成对应页面。命名时尽量统一大小写和中文/英文写法，避免产生重复分类。

## 5. 本地预览和构建检查

实时预览（包含草稿）：

```bash
/opt/hugo/bin/hugo server --buildDrafts
```

在浏览器打开终端显示的本地地址，通常为 `http://localhost:1313/`。结束预览按 `Ctrl+C`。

发布前执行与 CI 接近的生产构建：

```bash
HUGO_ENVIRONMENT=production /opt/hugo/bin/hugo --gc --minify \
  --baseURL https://onezjjj.github.io/github.io/ \
  --destination /tmp/joss-wiki-stack-site
```

检查构建输出中没有错误，并确认首页、文章、分类和标签页面可生成。搜索页应包含 HTML 与 JSON 输出。

## 6. 提交到 GitHub 并发布

查看待提交内容，确认没有密码、令牌、临时文件或不应提交的生成文件：

```bash
git status --short
git diff --check
git diff
```

只暂存本次要提交的源文件，例如新增文章：

```bash
git add content/posts/linux-network-notes.md
```

如果此次还修改了分类标签或其他源文件，将对应文件一并 `git add`。提交并推送到 `main`：

```bash
git commit -m "Add Linux network notes"
git push origin main
```

认证需使用已配置的 GitHub SSH key 或组织批准的 Git 凭据。不要在命令中写密码、把令牌写入 remote URL，或提交任何凭据文件。

推送后打开仓库的 **Actions** 页面，等待 `发布 Hugo 博客` 工作流完成；检查 build 和 deploy 均成功。成功后查看：

<https://onezjjj.github.io/github.io/>

GitHub Actions 失败时，先打开该次运行的日志，定位 Hugo 配置、依赖或内容错误；修复后重新执行本地构建，再提交并推送。

## 7. 常用命令速查

```bash
# 新建草稿
/opt/hugo/bin/hugo new content posts/new-article.md

# 本地预览，包含草稿
/opt/hugo/bin/hugo server --buildDrafts

# 检查修改
git status --short
git diff --check

# 提交并触发部署
git add content/posts/new-article.md
git commit -m "Add new article"
git push origin main
```

草稿发布前记得把 front matter 的 `draft` 改为 `false`。通常只需维护 `content/posts/` 下的文章，不要直接修改主题模块缓存或 `public/` 输出目录。
