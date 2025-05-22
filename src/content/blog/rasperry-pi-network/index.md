---
title: '解决 Raspberry Pi 网络配置：NetworkManager'
description: '树莓派网络配置（wlan）'
publishDate: '2025-5-21 22:00:00'
updatedDate: '2025-5-22 11:33:00'
tags:
  - '树莓派'
  - '网络配置'
draft: false
language: '中文'
comment: true
---
当在 Raspberry Pi OS Lite (Debian) 上配置网络时，首先想到编辑 `/etc/network/interfaces` 文件，但这并不奏效，特别是当系统使用其他网络管理工具时。例如，本文最终发现并使用的 `NetworkManager`。

## 问题描述

最初尝试通过修改 `/etc/network/interfaces` 来配置 Wi-Fi 和有线网络，并希望使用 `ifup` 等命令来应用更改。但遇到以下情况：

1.  `/etc/network/interfaces` 中的配置似乎未生效。
2.  执行 `ifup` 命令时提示 `command not found`。

## 故障排除与发现

1.  **检查 `ifupdown`**：
    ```bash
    dpkg -s ifupdown
    ```
    结果显示未安装，这意味着 `/etc/network/interfaces` 文件不会被传统方式处理。

2.  **检查 `dhcpcd`**：
    ```bash
    dpkg -s dhcpcd5
    dhcpcd --version
    ```
    同样显示未安装或命令未找到，排除了 `dhcpcd` 作为主要网络管理器的可能性。

3.  **检查 `systemd-networkd`**：
    ```bash
    sudo systemctl status systemd-networkd
    ```
    结果显示服务为 `inactive (dead)` 且被禁用。

4.  **发现 `NetworkManager`**：
    ```bash
    sudo systemctl status NetworkManager
    ```
    输出显示 `NetworkManager.service` 处于 `active (running)` 状态，并且日志中有其管理网络接口（如 `eth0`）的记录。
    **这表明 `NetworkManager` 是当前系统上实际的网络管理工具。**

## 解决方案：使用 NetworkManager (nmcli)

既然 `NetworkManager` 在运行，我们就应该使用它的工具来配置网络，而不是直接编辑 `/etc/network/interfaces`（该文件在这种情况下通常会被忽略或仅用于非常有限的配置）。

`nmcli` 是 `NetworkManager` 的命令行工具，非常适合在无图形界面的环境中使用。

### 步骤：

1.  **确认 NetworkManager 状态**：
    ```bash
    sudo systemctl status NetworkManager
    ```

2.  **查看无线设备工作状态**
    ```bash
    nmcli device status
    或 nmcli connection show (nmcli c s)
    ```
    查看无线网络是否被`NetworkManager`禁用：
    ```bash
    nmcli radio wifi
    ```
    如果显示 `enabled`，则无线设备正常工作。如果显示 `disabled`，则需要启用无线设备。
    ```bash
    sudo nmcli radio wifi on
    ```

3.  **列出可用的 Wi-Fi 网络**：
    在尝试连接前，`wlan0` 接口需要能扫描到网络。如果之前扫描失败提示 "unavailable"，确保无线开关未被软/硬阻止 (`rfkill list`) 并且 `nmcli radio wifi` 显示为 `enabled`。
    ```bash
    nmcli device wifi list
    ```
    如果此命令成功列出网络，说明无线设备工作正常。

4.  **连接到 Wi-Fi 网络**：
    使用 `sudo` 是因为修改网络配置需要管理员权限。
    ```bash
    sudo nmcli device wifi connect "你的WiFi名称" password "你的WiFi密码" ifname wlan0 name "自定义连接名"
    ```
    *   `"你的WiFi名称"`: 替换为实际的 SSID。
    *   `"你的WiFi密码"`: 替换为实际的密码。
    *   `ifname wlan0`: 指定使用 `wlan0` 接口。
    *   `name "自定义连接名"`: 为此连接设置一个易于识别的名称 (例如 `MyHomeWiFi`)，方便以后管理。

5.  **验证连接状态**：
    ```bash
    nmcli device status  # 查看 wlan0 是否已连接
    nmcli connection show # 查看已建立的连接及其状态
    ping -c 3 google.com # 测试网络连通性
    ```

6.  **为Wifi设置静态IP地址**：
    ```bash
    sudo nmcli connection modify "我的WiFi连接名" ipv4.method manual ipv4.addresses 192.168.1.101/24 ipv4.gateway 192.168.1.1 ipv4.dns "8.8.8.8,1.1.1.1" 
    sudo nmcli connection up "我的WiFi连接名"
    ```

## 重要提示

*   当 `NetworkManager` 活跃时，它会主导网络配置。`/etc/network/interfaces` 文件通常只应包含 `lo` (环回) 接口的定义，或者将其他接口（如 `eth0`, `wlan0`）标记为 `inet manual` 以明确告知 `NetworkManager` 接管。
*   始终首先确定你的 Linux 发行版正在使用哪个网络管理工具。常见的有 `ifupdown` (配合 `/etc/network/interfaces`)、`dhcpcd`、`systemd-networkd` 和 `NetworkManager`。
*   对于服务器，`nmcli` 是管理 `NetworkManager` 的强大工具。对于有图形界面的系统，通常会有相应的图形化网络配置工具。