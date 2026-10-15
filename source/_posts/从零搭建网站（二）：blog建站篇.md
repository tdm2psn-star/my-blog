---
title: 从零搭建网站（二）：blog建站篇
date: 2025-12-18 11:16:27
tags:
  - Hexo
  - 前端
  - 运维
categories:
  - 教程
cover: /img/2.webp
---

## 前言：技术选型的思考

建设个人网页有多种路径。从传统的 WordPress、Typecho 等基于数据库的“动态方案”，到 Astro、Hugo 等新兴的“静态方案”，不一而足。

本着**轻量化、易维护、数据资产纯净**的原则，我最终选择了 **Hexo** 这一经典的静态网站框架。它无需数据库支持，所有的文章均以 Markdown 文件存储，极大地降低了服务器压力与迁移成本。然而，Hexo 默认的主题风格往往过于简朴，难以满足现代审美需求。为了在“轻量化”与“产品级交互”之间取得平衡，我选择了叠加 **安知鱼 (AnZhiYu)** 主题。

安知鱼主题在视觉上具备极强的“产品感”，其清爽、现代的界面布局，能让个人主页呈现出类似成熟 App 的交互体验，这正是我作为一名 Android 开发者所追求的质感。

---

## 一、 基础环境构建

在正式开始前，请确保系统已安装 Node.js 与 Git。以下为标准初始化流程：

```bash
# 1. 安装 Hexo 命令行工具
npm install -g hexo-cli

# 2. 创建并进入项目目录
mkdir my-blog && cd my-blog

# 3. 初始化项目并同步核心依赖
hexo init
npm install

# 4. 安装页面渲染所需的依赖组件
npm install hexo-renderer-pug hexo-renderer-stylus --save

```

---

## 二、 主题配置与集成

安知鱼主题基于 Butterfly 主题进行了深度美化与功能增强。通过直接引用主题仓库，可以快速复现其设计风格。

### 1. 获取主题文件

进入项目目录下的 `themes` 文件夹，克隆主题源码。为确保路径简洁，建议将目录命名为 `anzhiyu`：

```bash
git clone [https://github.com/anzhiyu-c/hexo-theme-anzhiyu.git](https://github.com/anzhiyu-c/hexo-theme-anzhiyu.git) themes/anzhiyu

```

### 2. 启用主题

打开项目根目录下的全局配置文件 `_config.yml`，修改 `theme` 字段：

```yaml
# 指定当前使用的主题名称
theme: anzhiyu

```

### 3. 初始化主题配置

安知鱼主题的配置项较为丰富。建议将 `themes/anzhiyu/_config.yml` 复制到项目根目录，并重命名为 `_config.anzhiyu.yml`。Hexo 在解析时会优先读取该文件，这样可以避免在未来升级主题时丢失个人配置。

---

## 三、 本地调试与配置优化

### 1. 启动本地服务器

完成基础配置后，执行以下命令进行本地预览：

```bash
hexo clean && hexo s

```

此时可通过浏览器访问 `http://localhost:4000/` 查阅初步效果。

### 2. 实现局域网 IP 访问

若需在手机或测试设备上同步预览，需修改根目录下的 `package.json` 文件，使服务监听全局 IP 地址：

将脚本字段：
`"server": "hexo server"`
修改为：
`"server": "hexo server -i 0.0.0.0"`

---

## 四、 常见技术问题处理

### 1. Node.js 版本兼容性问题

在初始化 Hexo 8.x 环境时，若遇到版本不兼容的错误提示：

> `error hexo@8.1.1: The engine "node" is incompatible. Expected version ">=20.19.0". Got "20.9.0"`

**分析：** Hexo 8.x 引入了 ESM 标准，对 Node.js 运行环境的版本要求较高。
**对策：** 建议使用 `n` 模块将 Node.js 升级至最新的稳定版本（Stable）：

```bash
sudo npm install -g n
sudo n stable

```

### 2. 配置文件的层级逻辑

在 Hexo 体系中，需明确区分两类配置文件，切勿混淆：

* **站点配置 (`_config.yml`)**：位于根目录，负责网站标题、作者信息、永久链接格式等全局属性。
* **主题配置 (`_config.anzhiyu.yml`)**：负责首页布局、导航菜单、色彩方案及交互组件等视觉表现。

---

## 结语

网页设计并非单纯的代码堆砌，而是一次对个人逻辑与审美标准的梳理。通过“Hexo + 安知鱼”的组合，我们搭建起了一个稳健且美观的基础。后续我将针对 Android 项目展示及私有音乐服务的集成进行更深度的适配。
