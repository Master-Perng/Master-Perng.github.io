---
title: "Markdown 写作速查"
date: 2026-11-19T11:00:00+08:00
draft: false
summary: "常用 Markdown 写法示例，发布文章时可以直接参考。"
tags: ["Markdown", "写作"]
categories: ["教程"]
ShowToc: true
TocOpen: false
---

![深色背景上的 MARKDOWN 字样](/images/markdown-guide-cover.svg)

## 标题

使用一个到六个 `#` 表示不同层级的标题：

```markdown
## 二级标题
### 三级标题
```

## 强调与链接

```markdown
**粗体文字**
*斜体文字*
~~删除线~~
[链接文字](https://example.com)
```

效果：**粗体文字**、*斜体文字*、~~删除线~~、[链接文字](https://example.com)。

## 列表

```markdown
- 第一项
- 第二项

1. 第一步
2. 第二步
```

## 引用

```markdown
> 这是一段引用。
```

> 这是一段引用。

## 代码

行内代码写作 `` `console.log("Hello")` ``。

多行代码使用三个反引号，并填写语言名称：

```javascript
function greet(name) {
  console.log(`你好，${name}`);
}

greet("世界");
```

## 表格

| 项目 | 用途 | 是否常用 |
| --- | --- | --- |
| 标题 | 组织内容层级 | 是 |
| 列表 | 表达并列关系 | 是 |
| 表格 | 展示结构化信息 | 视情况 |

## 图片

将图片放入 `static/images/`，然后使用绝对路径引用：

```markdown
![图片说明](/images/example.png)
```

建议图片使用 WebP、PNG 或 JPEG，并在上传前适当压缩。

## 文章属性

每篇文章顶部的 `---` 之间是 Front Matter，可以控制标题、日期、摘要、标签、封面和草稿状态：

```yaml
---
title: "文章标题"
date: 2026-11-19T12:00:00+08:00
draft: false
summary: "文章摘要"
tags: ["标签一", "标签二"]
categories: ["分类"]
ShowToc: true
TocOpen: false
---
```

`draft: true` 的文章只会在本地预览时显示，不会发布到线上。
