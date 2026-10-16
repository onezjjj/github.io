+++
title = 'S-UI 安装与基础使用教程：Sing-box 管理面板'
date = 2026-10-16T12:00:00+08:00
draft = false
tags = ['S-UI', 'Sing-box', 'Linux', '服务器管理']
categories = ['网络技术']
+++

S-UI 是一个基于 SagerNet/Sing-box 的 Web 管理面板，可通过浏览器管理入站、客户端、流量和订阅等配置。本文以 Ubuntu/Debian 系统和官方安装脚本为例，介绍安装、首次登录、基本配置与排错。

> **重要提示：** S-UI 官方 README 目前声明项目面向个人学习和交流，并提示不要用于生产环境。请先阅读项目说明，自行评估维护、安全和合规风险。不要把管理面板或订阅地址无保护地暴露到公网。

## 1. 准备服务器

开始前确认你有一台自己管理的 Linux 服务器，并且能够通过 SSH 登录和使用 `sudo`。官方列出的 Linux 支持架构包括 amd64、arm64 等；具体系统和版本请以 [S-UI 官方仓库](https://github.com/alireza0/s-ui) 的最新说明为准。

先更新系统并安装下载工具：

```bash
sudo apt update
sudo apt install -y curl ca-certificates
```

安装脚本会以 root 权限修改系统并安装服务。运行前建议先阅读脚本内容，确认来源是官方仓库：

```bash
curl -fsSL \
  https://raw.githubusercontent.com/alireza0/s-ui/main/install.sh \
  -o /tmp/s-ui-install.sh
less /tmp/s-ui-install.sh
sudo bash /tmp/s-ui-install.sh
```

安装过程中，脚本会提示是否调整面板端口、面板路径、订阅端口和管理员凭据。首次安装时按提示设置非默认的用户名和强密码；如果选择不修改，脚本可能生成随机登录信息，请在终端中及时、安全地保存。不要将密码贴到公开聊天、截图或文章里。

安装结束时会显示面板访问地址。README 列出的默认值是面板端口 `2095`、路径 `/app/`，订阅端口 `2096`、路径 `/sub/`，但安装时可以修改；请以安装器实际显示的地址和你自己的设置为准。

## 2. 安全地访问面板

如果使用默认设置，地址形式通常是：

```text
http://服务器公网地址:2095/app/
```

首次登录后，确认管理员密码已经修改。**不要在未启用 HTTPS 的情况下，通过不可信的公共网络输入管理员密码。** 正式对外提供访问前，应配置域名和 TLS 证书，并优先通过防火墙限制管理面板只允许你的固定管理 IP 访问；也可以只通过 SSH 隧道或私有网络管理。

如果使用 UFW，可以只允许自己的管理 IP 连接面板端口。把示例地址替换为真实的管理出口 IP 和实际面板端口：

```bash
sudo ufw allow from 203.0.113.10 to any port 2095 proto tcp
sudo ufw status
```

`203.0.113.10` 是文档示例地址，不是可直接使用的真实地址。云服务器还要检查云厂商的安全组。只开放确实需要的入站端口；不要为了“测试方便”直接关闭防火墙或开放所有端口。订阅服务端口和代理入站端口应按实际使用范围单独规划。

## 3. 认识管理界面

下面的界面图来自 S-UI 官方前端仓库，用作功能位置示例；版本更新后，实际界面可能略有差异。

{{< remote-image src="https://raw.githubusercontent.com/alireza0/s-ui-frontend/main/media/main.png" alt="S-UI 官方仪表盘截图" caption="仪表盘：查看服务和流量概况。图片来源：[S-UI 官方截图集](https://github.com/alireza0/s-ui-frontend/blob/main/screenshots.md)。" >}}

{{< remote-image src="https://raw.githubusercontent.com/alireza0/s-ui-frontend/main/media/inbounds.png" alt="S-UI 官方入站管理截图" caption="入站页面：集中查看和管理已配置的入站。图片来源：[S-UI 官方截图集](https://github.com/alireza0/s-ui-frontend/blob/main/screenshots.md)。" >}}

{{< remote-image src="https://raw.githubusercontent.com/alireza0/s-ui-frontend/main/media/clients.png" alt="S-UI 官方客户端管理截图" caption="客户端页面：查看客户端及流量状态。图片来源：[S-UI 官方截图集](https://github.com/alireza0/s-ui-frontend/blob/main/screenshots.md)。" >}}

## 4. 添加入站和客户端

S-UI 的基本配置流程通常是先创建入站，再为入站添加客户端。不同协议和版本的字段会有差别，配置时应参考当前面板提示和官方文档，不要直接复制来历不明的配置。

1. 打开 **Inbounds（入站）**，新建一个入站，并选择面板支持的协议。
2. 设置监听地址和端口，按使用场景配置传输与 TLS 等参数。
3. 保存后确认入站状态正常，并确认服务器本机防火墙、云安全组和实际监听端口一致。
4. 在客户端管理区域添加客户端；按需要设置流量上限和到期时间。
5. 使用官方客户端或兼容客户端导入配置，确认能够连接后，再检查面板中的在线状态和流量统计。

首次配置建议一次只改一个关键参数，保存并验证后再继续。不要同时更改多个端口、证书和传输选项，否则出现问题时不容易定位原因。

## 5. 订阅与流量管理

S-UI 提供订阅功能，可按支持的格式生成客户端配置。复制订阅地址时要把它当作密码管理：任何拿到链接的人都可能获得对应配置。不要把完整订阅 URL 放进公开截图、文章、工单或代码仓库；发现泄露时应及时撤销或重新生成相关凭据。

客户端添加后，可以根据需要设置流量上限和有效期，并定期检查客户端流量、在线状态和服务器资源。流量统计只反映面板能够观察到的数据，不能代替系统日志、账单或完整的安全审计。

## 6. 服务管理和排错

官方管理脚本提供常用命令。Ubuntu/Debian 使用 systemd 时，可查看状态、重启服务和跟踪日志：

```bash
sudo s-ui status
sudo s-ui restart
sudo s-ui log
```

也可以直接查看 systemd 服务：

```bash
sudo systemctl status s-ui --no-pager
sudo journalctl -u s-ui -n 100 --no-pager
```

面板无法访问时，按顺序检查：

- `sudo s-ui status` 显示服务是否正在运行。
- 用 `sudo s-ui log` 或 `journalctl` 查看启动错误。
- 确认浏览器地址中的 IP、端口和路径与安装器输出一致。
- 确认服务器本机防火墙和云安全组允许来自管理 IP 的连接。
- 如果已更改端口，确认旧端口已关闭、新端口已按需开放。

更新前先备份 S-UI 数据目录和重要配置。官方管理脚本的 `update` 会重装最新版并保留现有数据；执行前阅读确认提示，并在维护窗口中操作：

```bash
sudo s-ui update
```

## 7. 官方资料

- [S-UI 官方仓库与安装说明](https://github.com/alireza0/s-ui)
- [S-UI 官方 Wiki](https://github.com/alireza0/s-ui/wiki)
- [S-UI 官方界面截图集](https://github.com/alireza0/s-ui-frontend/blob/main/screenshots.md)

安装命令、支持平台和界面都可能随版本变化。操作前请重新核对官方文档；请仅在你拥有或获准管理的服务器上部署，并遵守所在地法律法规及服务提供商的使用条款。
