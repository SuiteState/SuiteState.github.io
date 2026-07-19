# 如何新增一篇 Guide

## 目录结构

```
_config.yml                 Jekyll 配置（collections 与永久链接规则）
_layouts/base.html          页面外壳：head / 导航 / 页脚（中英双语切换）
_layouts/guide.html         文章版式：标题区 / 正文 / 结尾 CTA / 结构化数据
_guides/*.md                英文文章  -> /guides/<文件名>/
_guides_zh/*.md             中文文章  -> /zh/guides/<文件名>/
guides/index.html           英文列表页（自动列出 _guides）
zh/guides/index.html        中文列表页（自动列出 _guides_zh）
index.html, zh/index.html   首页，无 frontmatter，Jekyll 原样拷贝
zh.html                     旧地址重定向至 /zh/
sitemap.xml                 由 Jekyll 生成，自动包含所有文章
```

## 新增文章的步骤

1. 在 `_guides/` 放英文 `.md`，在 `_guides_zh/` 放中文 `.md`。
   **两个文件名必须相同**（文件名即 URL slug，一经发布不要再改）。
2. 每篇的 frontmatter：

```yaml
---
title: "文章标题"
description: "用于 meta description 与列表页摘要，一到两句。"
lang: en                      # 中文版写 zh
version: "19.0"
topic: "Inventory valuation"  # 列表页与标题区显示
date: 2026-07-18              # 首发日，发布后不再改
updated: 2026-07-19           # 可选。修订日；填了则文章页显示"修订于"、
                              # 列表页卡片改显示该日期、sitemap 的 lastmod 取它
ref: inventory-valuation-gap  # 中英两版必须一致，用于双语互链
alt_url: /zh/guides/<slug>/   # 指向另一语言版本；中文版写 /guides/<slug>/
module: suite_data_guard      # 可选，标注相关模块
---
```

3. 正文**不要写 H1**，标题由布局渲染，从 `##` 开始分节。
4. 列表页与 sitemap 会自动更新，无需手工登记。

## 约定

- URL 一经发布不再更改。改标题可以，改文件名不行。
- 正文源文件是 markdown，HTML 是构建产物，不要直接改构建结果。
- 新语言或新栏目按同样方式加 collection，不要在文章里写死路径。
