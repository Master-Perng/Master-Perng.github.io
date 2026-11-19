---
title: "你好，世界：我的第一篇博客"
date: 2026-11-19T10:00:00+08:00
draft: false
summary: "博客已经搭建完成。这篇文章简单介绍站点结构，以及以后如何发布新文章。"
tags: ["开始", "博客"]
categories: ["随笔"]
cover:
  image: "images/hello-cover.svg"
  alt: "渐变背景上的 HELLO 字样"
  caption: "从这里开始记录。"
  relative: false
ShowToc: true
TocOpen: false
---

## 欢迎

这是博客的第一篇文章，也代表站点已经可以正常构建和发布。

这套博客使用 **Hugo** 生成静态页面，由 **PaperMod** 提供主题，并通过 **GitHub Pages** 自动部署。写作时只需要编辑 Markdown 文件，不需要处理 HTML、服务器或数据库。

## 现在已经包含

- 响应式首页与文章列表
- 明亮、暗色主题自动切换
- 本地全文搜索
- 标签、分类和文章归档
- 代码高亮与一键复制
- 文章目录、阅读时长和分享按钮
- GitHub Actions 自动部署

## 发布下一篇文章

在项目根目录执行：

```powershell
hugo new content/posts/my-second-post.md
```

编辑生成的文件，确认 `draft: false`，然后提交并推送：

```powershell
git add .
git commit -m "新增文章：我的第二篇博客"
git push
```

GitHub Actions 会自动重新构建站点。几分钟后，新文章就会出现在线上。

## 接下来可以做什么

1. 修改 `hugo.toml` 中的博客名称、作者和首页介绍。
2. 将示例文章替换成自己的内容。
3. 在「关于」页面补充个人介绍。
4. 如果绑定了域名，在 GitHub Pages 设置中填写自定义域名。

> 博客最重要的是持续记录。先写下一行，再慢慢把它变成一篇文章。
