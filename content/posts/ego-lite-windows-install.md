+++
title = 'Windows 上安装与使用 Ego Lite：浏览器自动化入门'
date = 2026-10-26T13:47:00+08:00
draft = false
tags = ['Ego Lite', 'Windows', '浏览器自动化', 'GitHub Copilot']
categories = ['工具教程']
+++

如果你想让 AI 帮忙打开网页、读取页面内容，或测试自己开发的网站，可以了解一下 Windows 版 Ego Lite。本文介绍的是社区项目 [ego-lite-windows](https://github.com/ykscdoe/ego-lite-windows)，不是官方 macOS 应用的 Windows 移植版。

这个项目把一个社区维护的 ego-lite Linux 运行时与 Windows 启动器、GitHub Copilot Skill 打包在一起。它通过 CDP 驱动电脑上已有的 Chrome、Edge 或 Brave 浏览器；接入 VS Code 后，可以让 Copilot 根据你的要求执行浏览器操作。它不是一个全新的浏览器，也不是只能通过图形界面操作的独立软件。

## 一、准备环境

安装前准备好以下内容：

- Windows 电脑，以及能够安装软件的权限。
- **Node.js 22 或更高版本**。
- 已安装的 **Chrome、Edge 或 Brave** 浏览器，任选其一。
- 如果要在 VS Code 中使用 Copilot Skill，还需要 VS Code 和已启用的 GitHub Copilot Chat。
- Git（推荐），或者从 GitHub 下载项目 ZIP 文件。

Node.js 可以从 [Node.js 官网](https://nodejs.org/)安装；浏览器请从各自官方网站下载。安装后打开命令提示符（CMD），确认 Node.js 版本：

```cmd
node --version
```

输出版本号应为 `v22` 或更高。若提示找不到 `node`，重新打开终端；如果仍然无效，检查 Node.js 是否安装成功并加入了 PATH。

## 二、下载并验证

推荐用 Git 克隆完整仓库。打开 CMD，执行：

```cmd
git clone https://github.com/ykscdoe/ego-lite-windows.git
cd ego-lite-windows
```

如果没有安装 Git，也可以在项目 GitHub 页面点击 **Code → Download ZIP**，解压后在 CMD 中进入解压出的 `ego-lite-windows` 目录。无论使用哪种方式，都要保留 `runtime`、`scripts`、`skills` 和 `bin` 等项目文件夹。

运行端到端冒烟测试：

```cmd
node scripts\verify.mjs
```

测试会以无头方式启动 Edge 或 Chrome，打开 `example.com`、读取页面信息并关闭测试任务空间。第一次启动可能需要等待一会儿；成功时会看到类似下面的提示：

```text
PASS: runtime drives a real page on this machine
```

这一步用于确认 Node.js、浏览器和运行时可以协同工作。项目自带已编译的运行时，正常安装不需要运行 `npm install` 或自行构建。

## 三、接入 VS Code Copilot

在项目目录的 CMD 中运行安装脚本：

```cmd
scripts\install-copilot-skill.cmd
```

该脚本会将 `ego-browser` Skill 安装到当前 Windows 用户的 Copilot Skill 目录，并把项目的 `bin` 目录加入用户 PATH。完成后，在 VS Code 的命令面板运行 **Developer: Reload Window**，或者重启 VS Code。

打开 Copilot Chat，可以输入 `/` 查看是否出现 `ego-browser`。也可以直接提出一个明确、只读的任务，例如：

> 用 ego-browser 打开 https://example.com，告诉我页面标题，并概括页面上能看到的文字。

Copilot 会根据 Skill 指引调用浏览器运行时。首次使用时，建议先从公开网页和只读操作开始；需要登录、验证码或提交表单时，应由你本人检查页面并决定是否继续。

## 四、手动运行一个浏览器示例

也可以不用 Copilot，直接用 JavaScript 操作页面。下面的示例打开公开的 `example.com`，打印页面标题和语义快照，然后结束任务空间。

在项目目录新建 `task.js`，写入：

```js
const task = await taskSpaces.useOrCreate('example-demo')

await browser.openOrReuseTab('https://example.com', {
  wait: true,
  timeout: 30,
})

console.log('标题：', await page.title())
console.log(await page.snapshot())

await taskSpaces.complete(task.id, { keep: false })
```

在 CMD 中运行：

```cmd
ego-browser nodejs < task.js
```

`ego-browser` 命令由项目 `bin` 目录提供。若当前终端提示“不是内部或外部命令”，先确认安装脚本已运行，然后**重新打开 CMD**，使新的 PATH 生效。也可以继续通过 Copilot Chat 用自然语言提出浏览器任务。

这个示例只读取公开页面。`page.snapshot()` 会返回页面的语义信息，方便 AI 判断页面有哪些可交互元素；需要截图时，运行时还提供 `page.screenshot({ path: 'page.png' })`。更多 API 可查看项目的 [facade 参考文档](https://github.com/ykscdoe/ego-lite-windows/blob/main/skills/ego-browser/references/facade.md)。

## 五、常用命令

在完成 Skill 安装、`bin` 已加入 PATH 后，可以在任意 CMD 窗口中使用：

```cmd
ego-browser --status
ego-browser --url https://example.com
ego-browser --open
ego-browser --stop
```

- `--status`：查看运行状态。
- `--url <网址>`：启动时直接打开指定网址。
- `--open`：打开可见浏览器窗口，适合手动登录或检查页面。
- `--stop`：停止 Ego Lite 启动的浏览器实例。

如果只想做无头任务，可以在命令前加 `--headless`，例如：

```cmd
ego-browser --headless nodejs < task.js
```

## 六、常见问题

**提示找不到 Chrome 或 Chromium**

确认电脑已安装 Chrome、Edge 或 Brave。若自动探测失败，可以设置 `EGO_LINUX_CHROME` 指向浏览器可执行文件的绝对路径；Windows 路径要使用正斜杠，例如：

```cmd
set EGO_LINUX_CHROME=C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe
```

**提示 `ego-browser` 不是命令**

确认 `scripts\install-copilot-skill.cmd` 已成功运行，并重新打开终端。该命令位于项目的 `bin` 目录，不是 `scripts` 目录。

**VS Code 里看不到 Skill**

确认 GitHub Copilot Chat 已启用，并在 VS Code 执行 **Developer: Reload Window**。也可以检查当前用户目录下是否存在 `.copilot\skills\ego-browser\SKILL.md`。

**Copilot 操作停在登录页或验证码**

不要让自动化尝试绕过验证。切换到浏览器，由你本人完成登录或验证码，再让 Copilot 继续。遇到授权、付款、删除、发送等有实际后果的操作，也应先亲自检查并确认。

## 七、使用时要注意

Ego Lite 会驱动真实浏览器，因此网页访问权限和登录状态都很重要。不要把密码、Cookie、访问令牌或含个人信息的页面内容粘贴到公开聊天、代码仓库或截图中。项目提供导入 Chrome 登录状态的可选能力；导入会涉及浏览器个人资料和 Cookie，除非确有需要并已了解风险，否则不要启用。

此外，这是社区维护的 Windows 适配方案，不等同于官方 macOS 应用，也不保证所有复杂网页、iframe 或 Canvas 页面都能可靠识别。自动化输出应当复核；任何涉及账号、提交、购买、删除或发布的操作，都应由你确认后再执行。

## 参考资料

- [ego-lite-windows 项目仓库与 README](https://github.com/ykscdoe/ego-lite-windows)
- [安装与接入说明](https://github.com/ykscdoe/ego-lite-windows/blob/main/install.md)
- [Windows 配置与排错](https://github.com/ykscdoe/ego-lite-windows/blob/main/skills/ego-browser/references/windows.md)
- [ego-browser API 参考](https://github.com/ykscdoe/ego-lite-windows/blob/main/skills/ego-browser/references/facade.md)
