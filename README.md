# 个人博客

基于 [Hugo](https://gohugo.io) 构建的静态博客，使用 [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 主题。

## 本地开发

```bash
# 安装 Hugo（macOS）
brew install hugo

# 启动本地开发服务器（含草稿，热重载）
hugo server -D
# 访问 http://localhost:1313

# 构建生产版本（输出到 public/）
hugo
```

## 创建新文章

```bash
hugo new content content/posts/<分类>/<文件名>.md
```

文章使用 Markdown 编写，Front Matter 参考：

```yaml
---
title: "文章标题"
date: 2026-06-07T15:27:05+08:00
draft: false
tags: ["标签1", "标签2"]
author: "作者"
showToc: true
TocOpen: false
description: "文章描述"
cover:
    image: "<图片链接>"
    alt: "<替代文本>"
---
```

## 部署到 GitHub Pages

本仓库使用 **GitHub Actions** 自动部署。推送代码到 `main` 分支后，GitHub Actions 会自动构建 Hugo 站点并部署到 GitHub Pages。

### 前置条件：启用 GitHub Pages

在 GitHub 仓库页面完成以下设置：

**1. 设置部署来源**

`Settings → Pages → Build and deployment → Source` 选择 **GitHub Actions**。

**2. 确保 Actions 有写入权限**

`Settings → Actions → General → Workflow permissions` 选择 **Read and write permissions**。

### 工作流程说明

`.github/workflows/hugo.yaml` 中定义了自动部署流程：

| 步骤 | 说明 |
|---|---|
| Checkout | 拉取仓库代码（含 PaperMod 子模块） |
| Setup Hugo | 安装指定版本的 Hugo extended |
| Build | 执行 `hugo --minify` 构建站点 |
| Deploy | 部署到 GitHub Pages |

每次 push 到 `main` 分支后自动触发，无需手动操作。

### 自定义域名

如果使用自定义域名（如 `example.com`），需要修改 `hugo.yaml` 中的 `baseURL`：

```yaml
baseURL: https://example.com/
```

然后在 GitHub 仓库 `Settings → Pages → Custom domain` 中填写域名，并在 DNS 服务商处添加对应的 CNAME 或 A 记录。

### 手动部署（备用方案）

如果没有 GitHub Actions，也可以本地构建后手动推送：

```bash
hugo --minify
cd public
git init
git add .
git commit -m "deploy"
git push -f https://github.com/ccdgr/site.git main:gh-pages
cd ..
```

然后在 GitHub 仓库 `Settings → Pages → Source` 中选择 `Deploy from a branch`，分支选 `gh-pages`，目录选 `/ (root)`。
