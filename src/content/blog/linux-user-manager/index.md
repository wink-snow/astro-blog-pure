---
title: "Linux 用户管理"
description: "一个健忘者的note"
publishDate: "2025-6-7 20:38:00"
tags:
  - "Linux"
  - "笔记"
draft: false
language: "中文"
comment: true
---

运行某个服务时，为其创建一个专门的用户可以方便管理。个人认为这是一种很优雅的做法，符合 Linux **多用户**、**多任务**的特点。

## 创建用户
```bash
# -m: 创建用户的家目录 (/home/username)
# -g: 指定用户所属的初始组
# -G: 指定用户所属的附加组 (添加到 sudo 组以获得管理员权限)
# -s: 指定用户的默认 shell (通常是 /bin/bash)
# -c: 添加备注信息
sudo useradd -m -s /bin/bash -c "Another Test User" newuser
```

手动设置密码：
```bash
sudo passwd newuser
```

## 修改用户信息
将用户添加到其他组：
```bash
# -a 表示追加(append)，-G 表示附加组
sudo usermod -aG sudo newuser
```

## 删除用户
删除用户及其家目录：
```bash
sudo userdel -r newuser
```

## 组相关
| 命令 | 功能 | 
| :--- | :--- | 
| less /ect/group | 分页查看所有组 |
| getent group | 查看所有组 |
| getent group <groupname> | 查看组内用户 |
| groups <username> | 查看用户所属的组 |
| id <username> | 查看用户所属的组及 UID/GID |