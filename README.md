# blueberryclub.github.io

个人技术博客，使用 Hugo 构建，通过 GitHub Pages 部署。

## 项目结构

本项目分为两个目录：

- **hugo项目** (`IdeaCache/blueberryclub.github.io/`) — Hugo 源码项目
  - 包含 `content/`、`themes/`、`layouts/`、`hugo.toml` 等源文件
  - 用于编写博客文章、修改主题、本地测试

- **public项目** (`IdeaPro/blueberryclub.github.io/`) — 编译后的静态站点
  - 包含 Hugo 编译生成的 HTML、CSS、JS 等静态文件
  - 用于推送到 GitHub `main` 分支，由 GitHub Pages 提供服务

## 工作流程

1. 将 md 文件添加到 hugo项目的 `content/post/` 目录
2. 运行 `hugo` 命令编译生成静态文件到 `public/` 目录
3. 本地测试：运行 `hugo server` 预览效果
4. 将 `public/` 目录的内容同步到 public项目
5. 在 public项目 提交并推送到 GitHub `main` 分支
6. GitHub Pages 自动部署网站

## 博客分类目录

`content/post/` 下的分类：

- **使用教程** — 工具/框架的使用方法
- **南瑞项目** — 南瑞项目相关技术文档
- **学习笔记** — 技术学习与知识记录
- **安装教程** — 软件/环境安装指南
- **源码分析** — 框架/中间件源码解读
- **问题排查** — 常见问题与排错记录

## 文章 Front Matter 规范

每篇 md 文件顶部必须包含 TOML 格式的 front matter，**所有字段根据文档实际内容填写**：

```toml
+++
date = '2026-10-21T10:00:00+08:00'
draft = true
title = '文章标题'
tags = ["标签1", "标签2"]
series = ["系列名称"]
categories = ["分类1", "分类2"]
+++
```

字段说明：

- **date** — 文档实际创建或发布日期，格式 `YYYY-MM-DDTHH:MM:SS+08:00`
- **draft** — 本项目统一保留 `true`，发布靠编译加 `--buildDrafts`（见「发布注意事项」）
- **title** — 文章标题
- **tags** — 提取文档核心主题的关键词
- **series** — 文档所属系列（如"源码分析"、"安装教程"等）
- **categories** — 文档内容分类（如"分布式"、"数据库"等）

## 常用命令

### 编译站点

```bash
hugo --theme=hugo-theme-void baseUrl="https://blueberryclub.github.io/" --buildDrafts
```

### 本地预览

```bash
hugo server --theme=hugo-theme-void --buildDrafts
```

### 发布流程

发布 = 编译 hugo项目 → **清空** public项目 → 全量复制 → 提交推送。
注意是全删全替换，不是增量覆盖，避免残留已删除的旧页面。

```bash
# 1. 在 hugo项目 编译（必须带 --buildDrafts，见下方说明）
cd /Users/zhanghailong/IdeaCache/blueberryclub.github.io
hugo --theme=hugo-theme-void --buildDrafts

# 2. 清空 public项目 的旧产物（保留 .git）
cd /Users/zhanghailong/IdeaPro/blueberryclub.github.io
find . -mindepth 1 -maxdepth 1 -not -name '.git' -exec rm -rf {} +

# 3. 全量复制新产物
cp -R /Users/zhanghailong/IdeaCache/blueberryclub.github.io/public/. \
      /Users/zhanghailong/IdeaPro/blueberryclub.github.io/

# 4. 提交并推送到 main，触发 GitHub Pages 部署
git add -A
git commit -m "更新博客内容"
git push origin main
```

## 发布注意事项

- **`--buildDrafts` 是必需的**：本项目所有文章 front matter 都保留 `draft = true`，
  靠编译时带 `--buildDrafts` 上线。不带这个参数会把已发布的文章全部漏掉，
  第 2 步清空后站点会大面积丢页。
- **编译依赖 Node 20+**：主题 hugo-theme-void 走 Tailwind CSS 4 管线，
  Hugo 会调用系统 `node` 处理 `assets/css/main.css`。Node 版本过老会报
  `TAILWINDCSS: failed to transform` 并整站构建失败。
- **develop 与 main 是同一仓库的两个分支**：develop 放源码，main 放编译产物。
  `themes/`、`layouts/` 等未纳入 develop 跟踪，提交时按路径精确 add，
  不要用 `git add .` 把 `public/`、`themes/`、`.DS_Store` 带进源码分支。

## 首次初始化（参考）

```bash
git init
git add .
git commit -m "第一次提交博客"
git remote add origin https://github.com/BlueberryClub/blueberryclub.github.io.git
git push -u origin master
```