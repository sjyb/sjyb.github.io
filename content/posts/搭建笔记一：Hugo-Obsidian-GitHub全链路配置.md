---
title: "搭建笔记一：Hugo + Obsidian + GitHub Pages 全链路配置"
date: 2026-07-17
description: "从零搭建静态博客的完整闭环：Hugo 生成页面、Obsidian 写文章、GitHub Actions 自动部署至 GitHub Pages"
ShowToc: true
TocOpen: true
weight: 55
tags: ["Hugo","Obsidian","GitHub","博客","搭建"]
draft: false
---


> 本篇记录从零搭建静态博客的完整闭环：Hugo 生成页面 → Obsidian 写文章 → GitHub Actions 自动部署至 GitHub Pages。

<!-- more -->

## 1 环境确认

本方案基于 **macOS**（Linux/Windows 命令略有差异，思路一致）。

```bash
# 确认操作系统和关键工具
sw_vers          # macOS 版本
node -v          # Node.js 版本（npm 全局包需要）
brew -v          # Homebrew 版本
gh auth status   # GitHub CLI 登录状态
```

所需工具清单：

| 工具 | 用途 | 安装方式 |
|------|------|---------|
| Hugo | 静态网站生成器 | `brew install hugo` |
| GitHub CLI (`gh`) | GitHub 操作 + Pages 部署 | `brew install gh` |
| Git | 版本控制 | macOS 内置 |
| Obsidian | 笔记编辑器（作 Vault）| 官网下载 |
| Homebrew | macOS 包管理器 | `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"` |

> ⚠️ Hugo 版本应 ≥ 0.110（支持 hugo.toml）；PaperMod 主题需要 Hugo extended 版本（支持 SCSS）。

## 2 创建 GitHub 仓库

### 2.1 新建公开仓库

登录 GitHub → New Repository，命名为 `**.github.io`（`**` 替换为你的 GitHub 用户名）。

**重要设置**：
- **Public**（GitHub Pages 必须公开仓库）
- **Add a README file**：否（Hugo init 会自动创建）
- **.gitignore**：None（Hugo init 会生成）

### 2.2 开启 GitHub Pages

进入仓库 → Settings → Pages：

- **Source**：GitHub Actions
- **Branch**：无需选择（Actions 会自动部署）

### 2.3 生成 Personal Access Token（Classic）

Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic)

**必勾权限**：
- `repo`（完整仓库访问）
- `workflow`（Actions 操作）
- `read:org`（可选，组织仓库需要）

> ⚠️ 生成后立即复制保存，页面刷新后 token 无法再次查看。

## 3 安装 Hugo

```bash
brew install hugo
hugo version
# 预期输出：hugo v0.140.x 或更高
```

若已有 Hugo 但不是 extended 版本（导致 SCSS 编译失败）：

```bash
brew uninstall hugo
brew install hugo --build-from-source
# 或直接重装以确保更新
brew reinstall hugo
```

## 4 创建 Hugo 网站

```bash
cd ~
hugo new site blog --force
cd blog
# --force 避免目录已存在时报错
```

生成目录结构：

```
blog/
├── archetypes/      # 内容模板
├── assets/          # Hugo Pipes 处理的资源
├── content/         # 你的文章放这里
├── data/            # 站点数据
├── layouts/         # 布局模板
├── public/          # 构建产物（不上 Git）
├── resources/       # 缓存资源（不上 Git）
├── static/          # 静态文件（直接复制到 public/）
├── themes/          # 主题
├── hugo.toml        # 站点配置（新版 Hugo 推荐 toml）
└── config.yaml      # 等效配置（二选一，不要混用）
```

## 5 安装 PaperMod 主题

### 5.1 用 Git Submodule（推荐）

```bash
git init
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

### 5.2 手动下载（submodule 超时时的备选）

```bash
# 用 GitHub 代理下载
curl -L https://ghproxy.net/https://github.com/adityatelange/hugo-PaperMod/archive/refs/heads/master.zip -o PaperMod.zip
unzip PaperMod.zip
mv hugo-PaperMod-master themes/PaperMod
rm PaperMod.zip
```

## 6 配置文件（config.yml）

> ⚠️ Hugo 优先读取 `hugo.toml`，如果同时存在 `config.yaml`，`hugo.toml` 会覆盖它。**不要混用**。建议统一用 `config.yaml`（更直观），删除或重命名 `hugo.toml`。

```yaml
baseURL: "https://**.github.io/"
languageCode: "zh-cn"
title: "SJYB 的技术笔记"
theme: "PaperMod"

# 语法高亮（Hugo extended 专用）
markup:
  highlight:
    anchorLineNumbers: false
    lineNos: true

# 导航菜单
menu:
  main:
    - name: 笔记
      url: /posts/
      weight: 1
    - name: 标签
      url: /tags/
      weight: 2

# PaperMod 主题参数
params:
  env: production
  description: "网络工程 / 网络安全 / Python / AI"
  author: SJYB
  # 顶部图片
  header:
    image: "https://cdn.jsdelivr.net/gh/adityatelange/hugo-PaperMod@master/exampleSite/static/images/avatar.png"
  # 社交链接
  socialIcons:
    - name: github
      url: "https://github.com/**"
  # 主题配置
  defaultTheme: dark         # 默认深色模式
  disableThemeToggle: false  # 显示主题切换按钮
 ShowReadingTime: true       # 显示阅读时间
  ShowTOC: true              # 显示目录
  TableOfContents:
    showPageNav: true        # 在页面上方显示导航目录
```

## 7 写作流程配置

### 7.1 Obsidian 作为 Vault

Obsidian 打开 `~/blog` 文件夹作为 Vault（**直接打开现有文件夹，不要新建 Vault**）。

目录结构规划：

```
~/blog/
├── content/
│   ├── posts/           ← 公开文章（push 到 GitHub Pages）
│   └── _private/        ← 私有笔记（靶机 writeup 草稿/教学素材）
├── templates/           ← Obsidian 模板
├── .obsidian/          ← Obsidian 配置（不 push）
└── public/             ← Hugo 构建产物（不 push）
```

### 7.2 .gitignore（重要）

Hugo 每次 `hugo` 构建会生成 `public/`；GitHub Actions 也会在服务器构建。两者都不要进源码仓库。

```gitignore
# Hugo 构建产物
public/
resources/

# Obsidian 配置（Obsidian Git 插件会单独备份）
.obsidian/

# 私有内容
_private/

# 系统文件
.DS_Store
Thumbs.db
*.swp
*.swo
```

### 7.3 GitHub 初始化

```bash
cd ~/blog
git init
git remote add origin https://github.com/**/**.github.io.git

# 添加所有文件（排除 .gitignore 中的）
git add .
git commit -m "feat: 初始化 Hugo 博客"

# 推送（gh 会自动调用浏览器授权）
gh auth login
gh repo sync
git push -u origin main
```

## 8 配置 GitHub Actions 自动部署

在 `blog/.github/workflows/deploy.yml` 创建（注意路径：`blog/` 目录下，非 `blog/blog/`）：

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: true
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "latest"
          extended: true

      - name: Build Hugo
        run: hugo --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

推送后：

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: 添加 GitHub Pages 自动部署 workflow"
git push
```

然后在 GitHub 仓库页面：Actions → 等待第一次 workflow 运行完成（约 20-30 秒）→ 访问 `https://**.github.io` 确认页面正常。

## 9 Obsidian 插件安装

### 9.1 手动下载社区插件

Obsidian 社区插件下载后解压到 `.obsidian/plugins/插件目录/`。

**目录名必须与插件的 `manifest.json` 中的 `id` 一致**，否则 Obsidian 无法识别。

常用插件：

| 插件 | id | 用途 | manifest id |
|------|-----|------|-------------|
| Obsidian Git | `obsidian-git` | git push 自动同步 | `obsidian-git` |
| Templater | `templater-obsidian` | 模板语法增强 | `templater-obsidian` |

下载方式（以 GitHub release 下载 `*.zip` 为例）：

```bash
# 下载 release 资产（以 obsidian-git v2.38.x 为例）
VERSION="v2.38.6"
curl -L "https://ghproxy.net/https://github.com/denolehov/obsidian-git/releases/download/${VERSION}/obsidian-git.zip" \
  -o /tmp/obsidian-git.zip

# 解压（obsidian-git 的 dist/ 目录即为插件内容）
unzip -q /tmp/obsidian-git.zip -d /tmp/og_tmp
mv /tmp/og_tmp/dist ~/.obsidian/plugins/obsidian-git/

# Templater 同理
VERSION="v2.20.6"
curl -L "https://ghproxy.net/https://github.com/SilentVoid13/Templater/releases/download/${VERSION}/templater.zip" \
  -o /tmp/templater.zip
unzip -q /tmp/templater.zip -d /tmp/tp_tmp
mv /tmp/tp_tmp/dist ~/.obsidian/plugins/templater-obsidian/
```

### 9.2 启用插件列表

创建/编辑 `~/.obsidian/community-plugins.json`：

```json
[
  "obsidian-git",
  "templater-obsidian"
]
```

### 9.3 Obsidian Git 配置

打开 Obsidian → 设置 → Obsidian Git：

| 配置项 | 建议值 |
|--------|--------|
| Vault backup interval | 10 分钟 |
| Auto backup after commit | ✅ 开启 |
| Commit author | `** <**@**.**>` |
| Custom message | `%Y-%m-%d %H:%M 自动备份` |

> ⚠️ 如果 gh 已经登录并配置了 git credential helper，Obsidian Git 的 push/pull 会自动使用 GitHub 凭据，无需额外输入密码。

### 9.4 Templater 模板配置

设置 → Templater：

- **Template folder**: `templates`
- **New file template**: `templates/post.md`

创建 `~/blog/templates/post.md`：

```markdown
---
title: "<% tp.file.title %>"
date: <%
description: "从零搭建静态博客的完整闭环：Hugo 生成页面、Obsidian 写文章、GitHub Actions 自动部署至 GitHub Pages"
ShowToc: true
TocOpen: true
weight: 55 tp.file.creation_date() %>
tags: []
draft: false
---

# <% tp.file.title %>

<!-- more -->

```

`<!-- more -->` 是 Hugo/PaperMod 的摘要分隔符，文章列表页只显示其上方的内容。

## 10 发布第一篇文章（端到端流程）

### 10.1 在 Obsidian 中新建文章

在 `content/posts/` 文件夹右键 → 新建笔记 → Templater 自动填充标题、日期、front matter。

### 10.2 写作与保存

写完内容后，Obsidian Git 插件会在下一个备份周期（默认 10 分钟）自动 `git add . && git commit && git push`。

### 10.3 手动触发（首次）

也可以打开 Obsidian 命令面板（`Cmd+P`），输入 `Obsidian Git: Commit & push` 手动触发。

### 10.4 验证线上效果

```bash
# 本地预览（实时刷新）
cd ~/blog
hugo server -D
# 访问 http://localhost:1313 查看效果

# 等待 GitHub Actions 构建完成（约 20-30 秒）
# 访问 https://**.github.io/posts/文章名/
```

构建状态在 GitHub 仓库 → Actions 页面查看。

## 常见问题排查

### Hugo 构建报错 `cannot evaluate field enabled in type bool`

**原因**：`hugo.toml` 和 `config.yml` 同时存在，Hugo 优先使用 `hugo.toml`，其中的旧版 PaperMod 配置语法不兼容。

**解决**：删除或重命名 `hugo.toml`，统一使用 `config.yml`。

### PaperMod 主题样式不显示

**原因**：submodule 未拉取完整或使用了非 extended 版本的 Hugo。

**解决**：

```bash
# 重新拉取 submodule
git submodule update --init --recursive

# 确认 Hugo 是 extended 版本
hugo version | grep extended  # 有输出 = extended 版
```

### GitHub Actions 部署失败 404

**原因**：GitHub Pages 尚未开启，或 Source 设为 Branch 而非 Actions。

**解决**：Settings → Pages → Source 选 **GitHub Actions**。

### Obsidian Git push 需要输入密码

**原因**：未配置 git credential helper。

**解决**：

```bash
# 方法一：用 gh auth
gh auth setup-git

# 方法二：手动配置 credential helper
git config --global credential.helper osxkeychain
```

### `content/posts/` 新建文件没有 Templater 模板

**原因**：Templater 的「新建文件模板」只对根目录新建文件生效，`content/posts/` 下新建文件需要单独设置。

**解决**：Obsidian 设置 → Templater → 开启 **Enable folders templates**，并在「Folder templates」中添加 `content/posts/` → `templates/post.md`。

---

> 下一篇：[搭建笔记二：Obsidian + picgo-core + Gitee 图床配置]({{< ref "搭建笔记二：Obsidian-picgo-core-Gitee图床配置" >}})
