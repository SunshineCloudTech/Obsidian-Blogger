---
title: SunshineCloud Vault
date: 2026-06-06
tags:
  - 入口
  - 索引
  - vault
description: SunshineCloud Obsidian Vault — 技术文档、学习笔记、职业发展与生活兴趣的综合知识库入口
icon: home
index: true
---

# SunshineCloud Vault

欢迎来到 SunshineCloud Obsidian Vault！这是一个个人知识管理平台，专门收集和分享技术文档、学习笔记、实践经验，以及与职业发展和生活兴趣相关的内容。

> [!note] Vault 范围
> 当前 Obsidian Vault 仅指向 `Obsidian/` 目录，仓库中的其他目录（如 `.claude/`、配置脚本等）不属于 Vault 的内容边界。

## 📁 Vault 目录索引

### [[技术开发/README|技术开发]]
Web 框架、数据库、版本控制与 Linux 系统管理 — **10 篇笔记**

- [[Vue2脚手架快速搭建|Vue2 脚手架]]、[[Vue3脚手架快速搭建|Vue3 脚手架]] — 企业级前端项目搭建
- [[MySQL表与表之间数据的同步]]、[[MySQL生成列表序号（1）]]、[[MySQL生成列表序号（2）]]、[[MySQL使用UUID函数自动生成uuid并插入指定列]] — 数据库操作
- [[GitSubmodule使用指南]]、[[Git子模块强制更新与路径重配置指南]] — 版本控制
- [[Linux系统Tar打包目录]]、[[Linux系统中的cgroups的了解与应用]] — Linux 系统管理

### [[系统运维/README|系统运维]]
IT 系统运维工程师培训指南 — **9 篇笔记**

- [[应聘指南01-基础设施管理]] ～ [[应聘指南08-报告撰写]] — 8 大核心能力专项指南
- [[在VNC中运行Obsidian|在 VNC 中运行 Obsidian]] — 虚拟桌面环境搭建

### [[职业发展/README|职业发展]]
求职面试、简历撰写与职业规划 — **14 篇笔记**

- [[简历中的个人优势如何撰写]]、[[简历中的实习经历如何撰写]]、[[简历中的教育经历如何撰写]] — 简历优化
- [[面试常见问题解答(1)]]、[[面试常见问题解答(2)]]、[[面试常见问题解答(3)]]、[[面试常见问题标准解答示例]] — 面试策略
- [[IT系统运维工程师应聘指南]]、[[IT系统运维工程师职位详解与应聘指南]]、[[IT系统运维工程师面试问题总结]]、[[运维应届生面试问题准备]] — 运维岗位
- [[电信行业面试问题准备]]、[[电信行业面试问题参考回答]] — 行业专题
- [[校园技术运维常见对话用语]] — 英文沟通

### [[日本动漫圣地巡礼/README|日本动漫圣地巡礼]]
基于日本动画旅游协会 (AATA) 2020 官方数据的动漫取景地攻略 — **6 篇笔记**

- [[日本动漫圣地巡礼-北海道东北地方完全指南|北海道·东北地方]]
- [[日本动漫圣地巡礼-关东地方上篇探索指南|关东地方（上篇）]]
- [[日本动漫圣地巡礼-关东地方下篇探索指南|关东地方（下篇）]]
- [[日本动漫圣地巡礼-东京都心动漫名所完全攻略|东京都心动漫名所]]
- [[日本动漫圣地巡礼-中部地方温泉与自然风光探访|中部地方]]
- [[日本动漫圣地巡礼-近畿中国四国九州地方全域探索|近畿·中国·四国·九州]]

### [[生活与兴趣/README|生活与兴趣]]
美食、园艺与系统优化 — **4 篇笔记**

- [[如何获取优质蜂蜜|优质蜂蜜的获取]]
- [[如何制作蜂蜜柠檬茶|蜂蜜柠檬茶的制作]]
- [[如何种一棵柠檬树|柠檬树的种植]]
- [[一键加速，让你的Windows机械硬盘传输速度飞起来!|Windows 硬盘加速优化]]

### [[项目文档/README|项目文档]]
项目全生命周期文档中心，覆盖规划、架构、实施、运维、安全等各阶段。

### [[B站小漫画合集/README|B站小漫画合集]]
原创小漫画作品集，展示生活中的点滴趣事和创意灵感。

## 🛠 技术栈

本 Vault 及其关联的网站项目使用以下技术：

| 领域 | 技术 |
|------|------|
| 知识管理 | Obsidian |
| 网站生成 | VuePress |
| 标记语言 | Markdown |
| 托管平台 | GitHub Pages |
| 开发工具 | Git、Docker、VS Code |

## 👤 关于作者

**SunshineCloudTech** — 热爱人工智能、机器学习及前沿技术的开发者。

- **编程语言**: Python、JavaScript、Java、C++
- **框架与库**: SpringBoot、PyTorch、Vue、Node.js
- **数据库**: MySQL
- **当前学习方向**: AI For Offline、Cloud Services、DevOps Optimization
- **邮箱**: jamesmith007@126.com
- **GitHub**: [SunshineCloudTech](https://github.com/SunshineCloudTech)
- **博客**: [SunshineNotes](https://SunshineCloudTech.github.io)

> 📌 欢迎浏览文档，如有合作意向或任何问题，欢迎联系！

## 🖥️ 远程访问 (VNC)

本 Vault 部署在虚拟桌面环境中，通过 VNC + noVNC 提供浏览器访问：

- 虚拟桌面：`Xvfb :1`
- 窗口管理器：`openbox`
- VNC 服务：`x11vnc` 监听 `127.0.0.1:5901`
- Web 入口：[http://127.0.0.1:6080/vnc.html?autoconnect=true&resize=scale](http://127.0.0.1:6080/vnc.html?autoconnect=true&resize=scale)

> [!tip] 详细配置与故障排查
> 完整的安装步骤、依赖说明和常见问题解决方案请参见 [[在VNC中运行Obsidian|在 VNC 中运行 Obsidian]]。

## 📝 使用建议

1. 利用 **Obsidian 图谱视图**发现笔记之间的隐藏关联
2. 使用 **全文搜索**（Ctrl+Shift+F）快速查找具体技术方案
3. 每个子目录的 README 包含更详细的索引和推荐阅读顺序
4. 结合标签系统（`#Linux`、`#MySQL`、`#面试技巧` 等）进行内容筛选

## 📄 许可

本项目采用 **GNU Affero General Public License v3.0** 许可证。详情请参见仓库根目录的 [LICENSE](../LICENSE) 文件。

---

*最后更新：2026年6月*
