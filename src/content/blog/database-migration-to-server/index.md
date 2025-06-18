---
title: "本地数据库向服务器迁移(PostgreSQL)"
description: "版本不兼容、管道传输乱码指北"
publishDate: "2025-6-18 15:06:00"
tags:
  - "PostgreSQL"
  - "数据库"
  - "文件传输"
draft: false
language: '中文'
comment: true
---

最近完成了一次个人 Wiki 站点向服务器的部署工作，过程中遭遇了数据库版本不兼容和管道传输乱码的问题，在此做一记录。

## 准备工作
### 如何使用 postgres 用户
执行 `sudo apt install postgresql`安装后，会自动创建一个系统用户`postgres`，使用如下命令来切换身份：

```bash
sudo -i -u postgres
```

### 在服务器上准备数据库和用户
进入 postgres 用户环境，执行：
```bash
createdb [数据库名称]
createuser -P [用户名] # -P 选项表示需要输入密码
```

将数据库权限移交给此用户：
```bash
psql

ALTER DATABASE [数据库名称] OWNER TO [用户名];

\q
```

## 开始迁移
### 在Windows本地采用纯文本备份(PowerShell)

适用于较小的数据量，可以绕过`pg_dump`与`pg_restore`版本不一致问题。
```bash
pg_dump --column-inserts -U [用户名] [数据库名] | Out-File -FilePath "[路径\文件名.sql]" -Encoding utf8
```

### scp 传输
笔者的服务器用到了 Cloudflare 的 Tunnel，Pi用户已完成了ssh配置，可参照之前的[文章](http://blog.snowink.store/blog/internal-network-access)
```bash
scp "D:\Downloads\wiki_inserts_utf8.sql" Pi:~/ 
```

PowerShell 中使用ssh管道传输会让文本编码产生问题，一个错误的示例：
```bash
cat .\wiki_inserts.sql | ssh Pi -- 'cat > ~/wiki_inserts.sql'
```

### 数据库恢复（服务器）
在 postgres 用户下，执行：
```bash
psql -d wiki -f ~/wiki_inserts_utf8.sql
```