---
title: 在 VNC 中运行 Obsidian
date: 2026-06-06
tags:
  - obsidian
  - vnc
  - 虚拟桌面
  - 运维
description: 记录当前工作区中 Obsidian 的安装、调试和网页访问方式，方便以后重新启动或复现同样的环境。
---
# 在 VNC 中运行 Obsidian

这份笔记记录了当前工作区中 Obsidian 的安装、调试和网页访问方式，方便以后重新启动或复现同样的环境。

## 目标

把 Obsidian 运行在一个虚拟桌面里，再通过 VNC 和 noVNC 暴露出来，这样即使当前环境没有本地桌面，也可以直接在浏览器里看到 Obsidian 界面。

## 实际方案

- 使用 `Xvfb` 创建一个虚拟显示器，显示号是 `:1`。
- 使用 `openbox` 提供轻量窗口管理器。
- 使用 `x11vnc` 把 `:1` 这个桌面导出到 `127.0.0.1:5901`。
- 使用 `websockify` 和 noVNC 把 `5901` 转成网页访问入口，监听 `127.0.0.1:6080`。
- 使用 Obsidian 的 Linux 包安装程序，把应用放到 `/opt/Obsidian`。

## 安装过程

### 1. 检查可用工具

先确认系统里有没有现成的显示和 VNC 组件，以及包管理工具：

```bash
command -v Xvfb x11vnc openbox dbus-launch obsidian
```

结果显示这个环境最开始没有图形栈，但有 `apt-get` 和 `sudo`，所以可以直接安装需要的组件。

### 2. 安装 Obsidian 和基础图形组件

安装了这些包：

- `xvfb`
- `x11vnc`
- `openbox`
- `dbus-x11`
- `xterm`
- Obsidian 官方 Debian 包

Obsidian 的安装包来自官方 releases，版本为 `1.12.7`。

### 3. 处理运行时依赖

首次启动时，Obsidian 提示缺少 `libasound.so.2`。在这个 Ubuntu 24.04 环境里，对应的是 `libasound2t64`，不是旧名字的 `libasound2`。

补装后，Obsidian 的依赖就齐了，应用可以正常启动。

### 3.1 解决中文显示方框

如果 Obsidian 里中文显示成方框，原因通常是系统缺少可用的 CJK 字体。这个环境里最直接的修复是安装 `fonts-noto-cjk`：

```bash
sudo apt-get install -y fonts-noto-cjk
```

安装完成后，刷新字体缓存：

```bash
fc-cache -f
```

验证可以命中中文字体：

```bash
fc-match 'sans-serif:lang=zh-cn'
```

如果 Obsidian 已经在运行，重启一次应用让当前会话立即使用新字体。

### 4. 启动虚拟桌面

虚拟桌面使用如下方式启动：

```bash
Xvfb :1 -screen 0 1440x900x24 -nolisten tcp
openbox
```

然后用 `x11vnc` 监听这个显示：

```bash
x11vnc -display :1 -forever -shared -nopw -listen 127.0.0.1 -rfbport 5901
```

### 5. 启动 Obsidian

Obsidian 实际使用的是 `/opt/Obsidian/obsidian` 这个二进制，并且直接指定到显示 `:1`：

```bash
DISPLAY=:1 /opt/Obsidian/obsidian --no-sandbox --disable-gpu
```

这样它会出现在虚拟桌面里，由 VNC 和 noVNC 进行展示。

### 6. 启动 noVNC

安装 `novnc` 和 `websockify` 后，用下面的命令把 VNC 桌面转成网页：

```bash
websockify --web=/usr/share/novnc 6080 127.0.0.1:5901
```

浏览器访问地址是：

http://127.0.0.1:6080/vnc.html?autoconnect=true&resize=scale

## 当前运行状态

当前已经确认：

- `Xvfb :1` 在运行
- `openbox` 在运行
- `x11vnc` 在运行，并监听 `127.0.0.1:5901`
- `websockify` 在运行，并监听 `127.0.0.1:6080`
- Obsidian 在运行，并已经加载到虚拟桌面中

## 日志位置

相关日志都放在：

- `/tmp/obsidian-vnc/xvfb.log`
- `/tmp/obsidian-vnc/openbox.log`
- `/tmp/obsidian-vnc/x11vnc.log`
- `/tmp/obsidian-vnc/obsidian.log`
- `/tmp/obsidian-vnc/novnc.log`

## 常见问题

### 1. Obsidian 启动失败

先看 `obsidian.log`，如果提示缺少共享库，优先检查 `ldd /opt/Obsidian/obsidian` 的输出。

### 2. 浏览器打不开界面

先确认 `6080` 端口是否在监听，再检查 noVNC 是否启动成功。

### 3. 看到空白桌面

确认 Obsidian 进程是否还在运行，并检查 `DISPLAY=:1` 是否正确设置。

## 备注

如果以后想把这套流程做成一键启动脚本，可以把 `Xvfb`、`openbox`、`x11vnc`、`websockify` 和 Obsidian 的启动命令串起来，放到一个 shell 脚本里统一管理。