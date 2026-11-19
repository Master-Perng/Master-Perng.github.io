# Perng’s Blog

个人博客，使用 [Hugo](https://gohugo.io/) 构建，主题为 [Niello](https://github.com/guangmean/Niello)，通过 GitHub Pages 自动部署。

## 站点信息

- 站点名称：Perng’s Blog
- 作者：Perng
- GitHub：https://github.com/Master-Perng
- 发布地址：https://master-perng.github.io/

## 本地预览

安装 Hugo Extended 后，在项目根目录执行：

```powershell
hugo server -D
```

浏览器打开：

```text
http://localhost:1313/
```

`-D` 表示同时显示草稿文章。

## 新建文章

```powershell
hugo new content/posts/我的新文章.md
```

文章使用 TOML Front Matter：

```toml
+++
title = "文章标题"
date = "2026-11-19T15:00:00+08:00"
draft = false
summary = "显示在文章列表中的摘要"
tags = ["Hugo", "Niello"]
categories = ["随笔"]
+++
```

把 `draft` 改为 `false` 后，文章才会正式发布。

## 封面图片

Niello 会从文章正文中提取第一张图片，用作首页卡片、轮播图和文章配图。

```markdown
![图片说明](/images/example.webp)
```

图片放在 `static/images/`。建议使用 4:3 比例的 WebP、PNG、JPEG 或 SVG。

## 内容目录

```text
content/
├─ about.md              # 关于页面
├─ archives.md           # 归档页面
├─ search.md             # 搜索页面
└─ posts/
   ├─ hello-world.md
   └─ markdown-guide.md
```

`categories = ["info"]` 和 `categories = ["archives"]` 的辅助页面不会出现在首页文章列表中。

## 修改站点配置

主要配置位于 `hugo.toml`：

```toml
title = "Perng’s Blog"
defaultContentLanguage = "zh"
defaultContentLanguageInSubdir = false
theme = "Niello"
```

Niello 的站名、标语和搜索提示等文本位于 `i18n/zh.toml`。

## 主题与本地依赖

- 主题源码保存在 `themes/Niello/`
- PaperMod 已被完全替换
- Alpine.js、Fuse.js 和 mark.js 已存入 `static/js/`，不依赖第三方 CDN
- 主题许可证位于 `themes/Niello/LICENSE`

如需修改 TailwindCSS 样式，请在 Niello 源码目录中修改 `assets/css/main.css`，再按照主题说明重新生成 `app.css`。

## 发布到 GitHub Pages

```powershell
git add .
git commit -m "切换 Niello 主题"
git push
```

GitHub Actions 会构建 Hugo 站点并发布到：

```text
https://master-perng.github.io/
```

工作流位于 `.github/workflows/pages.yml`。
