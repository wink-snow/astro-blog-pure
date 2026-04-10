---
title: "静态页面在树莓派上的部署方案"
description: "使用 Github Actions 实现静态页面的自动化部署，提升效率和可靠性"
publishDate: "2026-4-10 12:20:00"
tags:
  - "静态页面"
  - "树莓派"
  - "Github Actions"
draft: false
language: '中文'
comment: true
---

在个人项目中，我使用树莓派作为服务器来托管我的静态页面，并使用 Github Actions 来实现自动化部署。

## 核心思路
1. **代码托管**：将静态页面的源代码托管在 Github 仓库中。
2. **Github Actions 触发**：每当代码推送到主分支时，Github Actions 会自动触发部署流程。
3. **构建和部署**：在 Github Actions 中构建静态页面，并将将构建好的静态文件通过 SSH (rsync) 推送到你的 Raspberry Pi 上的指定目录。
4. **Cloudflare Tunnel 服务**：Raspberry Pi 上的 Web 服务器（如 Nginx）提供这些静态文件，Cloudflare Tunnel 将其安全地暴露到公网。

## 主要步骤
### Raspberry Pi 配置
1. 安装 Nginx 以提供静态文件服务。
```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```
2. 创建一个目录来存放静态文件。
我的静态网页存放在 `/var/www/holy-kingdom-wiki` 目录。
```bash
sudo mkdir -p /var/www/holy-kingdom-wiki
sudo chown holy-kingdom-wiki:holy-kingdom-wiki /var/www/holy-kingdom-wiki # 新建一个用户 holy-kingdom-wiki 来 ssh 推送
```
3. 配置 Nginx
创建一个新的 Nginx 配置文件 `sudo nano /etc/nginx/sites-available/holy-kingdom-wiki`，内容如下：
```nginx
server {
    listen 8003;
    listen [::]:8003; # IPv6 支持

    server_name calco.snowink.space; # 网站域名

    root /var/www/holy-kingdom-wiki; 
    index index.html index.htm;

    access_log /var/log/nginx/holy-kingdom-wiki.access.log;
    error_log /var/log/nginx/holy-kingdom-wiki.error.log;

    location ~* \.(css|js|png|jpg|jpeg|gif|svg|ico)$ {
        expires 30d; # 静态资源缓存时间，为30天
        add_header Pragma public; # 兼容旧浏览器
        add_header Cache-Control "public, immutable"; # 告诉浏览器资源是不可变的，可以长期缓存
    }

    location / {
        try_files $uri $uri/ /index.html; #
    }
}
```

启用该站点并重启 Nginx：
```bash
sudo ln -s /etc/nginx/sites-available/holy-kingdom-wiki /etc/nginx/sites-enabled/
sudo nginx -t # 测试配置是否正确
sudo systemctl restart nginx
```

4. 设置 ssh 免密登录（用于 Github Actions 部署）
在本地机上生成一对新的 SSH 密钥：
```bash
ssh-keygen -t rsa -b 4096 -C "github_actions_deploy_key" -f ~/.ssh/github_actions_holy_kingdom_wiki
```
将公钥内容添加到 Raspberry Pi 的 `~/.ssh/authorized_keys` 文件中：
```bash
cat ~/.ssh/github_actions_holy_kingdom_wiki.pub | ssh holy-kingdom-wiki@<Raspberry Pi IP> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
也可以登录 Raspberry Pi 后手动将公钥内容添加到 `~/.ssh/authorized_keys` 文件中。
```bash
# 在 Raspberry Pi 上执行
mkdir -p ~/.ssh
chmod 700 ~/.ssh # chmod 700 ~/.ssh 保护 .ssh 目录权限
nano ~/.ssh/authorized_keys # 粘贴公钥内容
chmod 600 ~/.ssh/authorized_keys
```
从本地机上测试 SSH 连接是否成功：
```bash
ssh -i ~/.ssh/github_actions_holy_kingdom_wiki holy-kingdom-wiki@<Raspberry Pi IP>
```
5. 配置 Cloudflare Tunnel
配置文件在 `~/.cloudflared/config.yml`。
```yaml
tunnel: <Tunnel ID>

ingress:
  - hostname: calco.snowink.space
    service: http://localhost:8003
```
创建 DNS 记录，将 `calco.snowink.space` 指向 Cloudflare Tunnel 提供的 CNAME 地址，可参照我的[ 另一篇博客 ](https://blog.snowink.store/blog/internal-network-access)进行配置，如果一个 Tunnel 需要绑定多个域名，推荐使用 Cloudflare 的控制台手动添加 CNAME 记录。

### Github 仓库准备
1. 在 Github 上创建一个新的仓库来存放网站的源代码
2. 添加 Secrets 到 Github 仓库
进入 GitHub 仓库 -> Settings -> Secrets and variables -> Actions -> New repository secret。
添加以下 Secrets：
- `SSH_PRIVATE_KEY`：之前生成的 SSH 私钥内容（`~/.ssh/github_actions_holy_kingdom_wiki` 文件内容）
- `SSH_HOST`：Raspberry Pi 的 IP 地址，我使用的是 Cloudflare Tunnel 为 SSH 创建的特定主机名
- `SSH_USER`：当前项目用于部署的 Raspberry Pi 用户名（`holy-kingdom-wiki`）
- `DEPLOY_PATH_ON_PI`：Raspberry Pi 上的部署目录（`/var/www/holy-kingdom-wiki`）
- `KNOWN_HOSTS_PI`：Raspberry Pi 的 SSH 公钥指纹，可以通过 `ssh-keyscan -H <Raspberry Pi IP>` 获取，也可以复制 Raspberry Pi 上的 `~/.ssh/known_hosts` 文件中的相关行。
3. 创建 Github Actions 工作流
具体配置文件的编写可参阅我的项目仓库 [Holy-Kingdom-Wiki](https://github.com/wink-snow/holy-kingdom-wiki)。

## 总结
通过上述步骤，我成功实现了静态页面在树莓派上的自动化部署。每当我将代码推送到 Github 仓库时，Github Actions 会自动构建并部署最新的静态文件到 Raspberry Pi 上。同时，Cloudflare Tunnel 的使用也确保了我的站点能够安全地暴露在公网。