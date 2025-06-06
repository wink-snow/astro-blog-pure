---
title: "无公网IP？用Cloudflare Tunnel轻松搭建并SSH访问你的个人服务器"
description: "Cloudflare Tunnel + Raspberry Pi 搭建个人服务器"
publishDate: "2025-5-23 16:44:00"
tags:
  - "Cloudflare"
  - "树莓派"
  - "内网穿透"
draft: false
language: '中文'
comment: true
---

在没有公网IP的情况下，想从外部访问到本地服务似乎是件难题。幸运的是，Cloudflare 提供了强大的免费工具——Cloudflare Tunnel，它可以安全地将你的内网服务暴露到公网，而无需公网IP或复杂的路由器端口映射。

这里介绍一种较为便捷且经济的方案：使用 Cloudflare + Raspberry Pi 搭建个人服务器，通过域名访问内网服务（事实上，网上此类的文章很多，笔者在此仅对自己的工作做一记录，毕竟拥有一台自己的服务器对开发者来说是一件很Cool的事）。

## 准备工作

- **一个的域名:** 可修改 NameServer（域名服务器）记录（笔者的域名购买自 [SpaceShip](https://www.spaceship.com/)）
- **一台内网服务器：** 本文使用 Raspberry Pi 4B（4GB RAM，Debian System）
- **注册好Cloudflare账户：** 免费账户即可 [Cloudflare](https://www.cloudflare.com/)

## 详细步骤

### 1. 将域名托管至 Cloudflare
前往 Cloudflare 官网，添加你要迁移至此的域名。之后，在域名原托管商处（SpaceShip）修改 Nameservers 记录（通常在域名管理界面可以找到），等待生效（所需时间不定，一般不会超过24小时）。

Cloudflare 的 Nameservers 记录如下：

> mike.ns.cloudflare.com
> paislee.ns.cloudflare.com


待其生效后，Cloudflare 的域名状态将显示为 Active。

### 2. 安装 cloudflared （在内网服务器上）
`cloudflared` 是 Cloudflare Tunnel 的客户端守护进程。

*   **Linux (Debian/Ubuntu):**
    ```bash
    sudo apt update
    sudo apt install curl lsb-release
    curl -L https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
    echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
    sudo apt update
    sudo apt install cloudflared
    ```
*   **其他系统：** 请参考 [Cloudflare 官方文档](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation/) 进行安装。

### 3. 登录 cloudflared 并创建 Tunnel
在内网服务器上运行：
```bash
cloudflared login
```
使用任意设备的浏览器打开出现的链接并授权。

推荐使用 Cloudflare Dashboard 来创建 Tunnel，Cloudflare 会提供在不同操作系统上安装 Tunnel 服务的命令，复制执行即可。

### 4. 配置 Tunnel 来路由 SSH 流量
创建一个 YAML 配置文件，路径为 `~/cloudflared/config.yml`，内容如下：
```yaml
tunnel: <你的Tunnel_ID_或_隧道名称>

ingress:
  # 将 ssh.yourdomain.com 的流量转发到内网的 localhost:22 (TCP)
  - hostname: ssh.yourdomain.com 
    service: tcp://localhost:22  

  # 一个兜底规则
  - service: http_status:404
```

### 5. 创建 DNS CNAME 记录
```bash
cloudflared tunnel route dns <你的隧道名称_或_Tunnel_ID> ssh.yourdomain.com
```

这会在 Cloudflare 的 DNS 中创建一个 CNAME 记录，将 `ssh.yourdomain.com` 解析到 Tunnel。

### 6. 将 cloudflared 作为服务运行
```bash
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

## 通过 Tunnel SSH 连接内网服务器
现在，我们内网服务器的 ssh 端口已经暴露到ssh.yourdomain.com，但直接ssh此域名是不行的，需要配置 ProxyCommand。

确保本地已安装 cloudflared 客户端，同时配置 SSH(~/.ssh/config)：
```bash
Host my-home-server                 # 自定义一个易记的主机别名
  HostName ssh.yourdomain.com       
  ProxyCommand path/to/cloudflared access ssh --hostname %h # Windows 
  User your_server_ssh_username    
  IdentityFile ~/.ssh/your_ssh_private_key
```