# Hugo + PaperMod 博客

这是一套可以直接发布到 GitHub Pages 的 Hugo 博客，已经包含 PaperMod 主题、中文示例文章、全文搜索、归档页面和自动部署配置。

## 一、先改成自己的信息

打开 `hugo.toml`，重点修改以下内容：

```toml
title = "Perng’s Blog"

[params]
  description = "Perng 的个人博客"
  author = "Perng"

  [params.homeInfoParams]
    Title = "你好，我是 Perng"
    Content = "欢迎来到我的数字花园。"
```

同时把 `params.socialIcons` 中的 GitHub 地址改成自己的地址。`content/about.md` 和示例文章也可以直接编辑。

部署到 GitHub Pages 时，工作流会自动获取仓库地址并覆盖 `baseURL`，所以不需要手动填用户名。

## 二、发布到 GitHub Pages

### 1. 新建仓库

在 GitHub 新建一个 Public 仓库。仓库名可以随便取，例如：

```text
my-blog
```

也可以把仓库命名为：

```text
你的GitHub用户名.github.io
```

前者发布在 `https://用户名.github.io/my-blog/`，后者发布在 `https://用户名.github.io/`。两种方式都可以使用。

### 2. 开启 GitHub Pages

进入仓库：

```text
Settings → Pages → Build and deployment
```

将 Source 设置为：

```text
GitHub Actions
```

### 3. 上传代码

解压本博客压缩包，进入包含 `hugo.toml` 和 `.github` 的目录，执行：

```powershell
git init
git add .
git commit -m "初始化 Hugo 博客"
git branch -M main
git remote add origin https://github.com/你的用户名/仓库名.git
git push -u origin main
```

推送后打开仓库的 `Actions` 页面。名为 `Deploy Hugo site to GitHub Pages` 的工作流完成以后，进入 `Settings → Pages` 就能看到博客地址。

如果第一次工作流在开启 Pages 前已经失败，进入 `Actions`，选择失败的运行，点击 `Re-run all jobs` 即可。

## 三、写新文章

需要先在电脑上安装 Hugo Extended。Windows 可以执行：

```powershell
winget install Hugo.Hugo.Extended
```

在项目根目录创建文章：

```powershell
hugo new content/posts/我的新文章.md
```

编辑文章，将文件顶部的 `draft: true` 改成：

```yaml
draft: false
```

生成的文章使用 TOML 格式的属性区，也可以参考 `content/posts/` 中的示例直接修改。常用属性：

```toml
+++
title = "文章标题"
date = "2026-11-19T12:00:00+08:00"
draft = false
summary = "显示在文章列表中的摘要"
tags = ["Hugo", "博客"]
categories = ["随笔"]
ShowToc = true
TocOpen = false
+++
```

## 四、本地预览

```powershell
hugo server -D
```

打开：

```text
http://localhost:1313/
```

`-D` 表示同时预览草稿。正式构建命令是：

```powershell
hugo --minify
```

## 五、发布更新

每次修改文章后执行：

```powershell
git add .
git commit -m "更新文章"
git push
```

GitHub Actions 会自动生成网站并发布，通常需要等待一到三分钟。

## 目录说明

```text
.
├─ .github/workflows/pages.yml   # GitHub Pages 自动部署
├─ archetypes/                   # 新文章模板
├─ assets/css/extended/          # 自定义样式
├─ content/                      # Markdown 文章和页面
├─ static/                       # 图片、图标等静态资源
├─ themes/PaperMod/              # 已内置的 PaperMod 主题
└─ hugo.toml                     # 站点配置
```

## 可选配置

- 绑定域名：在 `Settings → Pages → Custom domain` 中填写域名，并按 GitHub 给出的提示配置 DNS。
- 评论功能：可以接入 Giscus，将评论存储到 GitHub Discussions。
- 主题说明：`themes/PaperMod/LICENSE` 是 PaperMod 的 MIT 许可证，分发或修改主题时请保留。

不要提交密码、访问令牌、私钥或其他敏感信息，因为 GitHub Pages 站点和公开仓库通常是可公开访问的。
