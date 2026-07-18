---
title: "搭建笔记五：Obsidian Git 同步配置与 Hugo 交叉引用整理"
date: 2026-07-18
description: "Hugo ref shortcode 避坑指南：文件名规范、交叉引用语法、构建失败常见原因"
ShowToc: true
TocOpen: true
weight: 35
tags: ["Obsidian","Git","Hugo","同步","交叉引用"]
draft: false
---

# 搭建笔记四：Obsidian Git 同步配置与 Hugo 交叉引用整理

> 本篇记录 Obsidian → Hugo → GitHub Pages 同步链路的配置要点与常见坑整理，是前四篇搭建笔记的精华提炼。

<!-- more -->

## 1 完整同步链路架构

```
Obsidian（本地写作）
    │ Cmd+S 保存
    ▼
Obsidian Git 插件（每 10 分钟）
    │ git add . → git commit → git push
    ▼
GitHub 仓库（博客源码）
    │ Webhook 触发
    ▼
GitHub Actions（hugo --minify）
    │ 20-30 秒构建
    ▼
GitHub Pages（https://**.github.io）
```

两条子链路：

| 方向 | 触发 | 延迟 |
|------|------|------|
| 本地 → 线上 | Obsidian Git 自动/手动 push | ≈ 30s（push + 构建）|
| 线上 → 本地 | Obsidian Git 自动 pull | ≤ 10 分钟（自动间隔）|

## 2 Obsidian Git 插件核心配置

打开 Obsidian → 设置 → Obsidian Git：

| 配置项 | 推荐值 | 原因 |
|--------|--------|------|
| Vault backup interval | 10 分钟 | 平衡自动保存与电池消耗 |
| Auto commit after commit | ✅ | commit 后自动 push |
| Pull updates interval | 10 分钟 | 接收远端改动 |
| Auto pull on startup | ✅ | 启动时同步远端 |
| Pull before push | ✅ | 避免落后远端 |
| Auto push on save | ✅ | Ctrl+S 后立即 push |
| Commit author | `** <**@**.**>` | 脱敏用户信息 |
| Sync method | Merge | 双向同步 |
| Disable push | ❌ | 必须开启推送 |

## 3 Git 仓库级用户信息（防隐私泄露）

```bash
cd ~/blog

# 仓库级配置（仅影响 ~/blog，不影响全局和其他仓库）
git config user.name "**"
git config user.email "**@**.**"

# 验证（应看到仓库级配置）
git config --list | grep "^user"
```

> ⚠️ 每次 commit 前确认 git 用户信息正确，避免真实身份泄露。

## 4 Hugo cross-ref 引用规范

### 4.1 引用格式

Hugo 使用 shortcode 语法 {{ ref "文件名（不含 .md）" }} 引用页面，文件名必须**精确匹配**。

```markdown
> 正确格式：{{ ref "文件名（去掉.md扩展名）" }}
```

### 4.2 正确 vs 错误示例

```
✅ 正确：ref "搭建笔记二：Obsidian-picgo-core-Gitee图床配置"
❌ 错误：ref "搭建笔记二-a"  （文件名已被统一为 二，无 -a 后缀）
```

### 4.3 文件名与 front matter title 必须对齐

Hugo 交叉引用用的是**文件名（去掉 .md）**，与 front matter `title:` 无关。

文件名规则：

```
搭建笔记一：{内容摘要}.md
搭建笔记二：{内容摘要}.md
搭建笔记二-a：{内容摘要}.md
搭建笔记二-b：{内容摘要}.md
搭建笔记三：{内容摘要}.md
```

> front matter `title:` 可以包含更多细节（如完整标题），但序号前缀必须与文件名一致。

### 4.4 本博客当前文章列表

| 文章 | Hugo Ref Key |
|------|-------------|
| 搭建笔记一：Hugo + Obsidian + GitHub Pages 全链路配置 | `搭建笔记一：Hugo-Obsidian-GitHub全链路配置` |
| 搭建笔记二：Obsidian + picgo-core + Gitee 图床配置 | `搭建笔记二：Obsidian-picgo-core-Gitee图床配置` |
| 搭建笔记二-b：macOS launchd 开机自启动 picgo server | `搭建笔记二-b：macOS-launchd-开机自启动-picgo-server` |
| 搭建笔记三：Obsidian Git 本地优先同步配置 | `搭建笔记四：Obsidian-Git本地优先同步配置` |

### 4.5 发布前必做 Hugo 本地构建测试

```bash
cd ~/blog
hugo --minify
# 若出现 REF_NOT_FOUND 错误，立即修复后再 push
```

## 5 隐私保护配置

`.gitignore` 清单（永不 push）：

```gitignore
# Hugo 构建产物
public/
resources/

# Obsidian 配置
.obsidian/

# 私有笔记（靶机 writeup 草稿、教学素材）
_private/

# 系统文件
.DS_Store
```

> `_private/` 用于存放含真实 IP/flag 的靶机 writeup 草稿，写完确认无误后，手动脱敏复制到 `content/posts/` 发布。

## 6 GitHub push 凭证问题（macOS SteamTools 用户）

### 问题描述

SteamTools 代理工具对 `github.com:443` 做 SSL 中间人拦截，导致 `git push` 失败：

```
fatal: unable to access 'https://github.com/...':
Error in the HTTP2 framing layer
```

### 解决方案

```bash
# 方案一：设置 GITHUB_TOKEN 环境变量（推荐）
export GITHUB_TOKEN=$(gh auth token 2>/dev/null)
git push origin main

# 方案二：降级到 HTTP/1.1
git config --global http.version HTTP/1.1

# 方案三：配置 gh 的 credential helper（需 gh auth git-credential）
git config --global credential.helper "/usr/local/bin/gh auth git-credential"
```

## 7 冲突处理：本地优先原则

单作者场景下，冲突极少。以下是处理原则：

| 场景 | 处理 |
|------|------|
| 仅本地改动 | 自动 push 成功 |
| 仅远端改动 | 自动 pull 成功 |
| 同一文件同时改动 | 以本地为准：`git checkout --ours <file>` |

```bash
# 冲突时强制本地优先
git checkout --ours content/posts/冲突文件.md
git add content/posts/冲突文件.md
git commit -m "chore: 解决冲突，以本地版本为准"
git push origin main
```

## 8 发布检查清单

每次写完文章，Obsidian Git 推送前：

```
□ front matter title 序号与文件名一致
□ Hugo 本地构建 0 错误（hugo --minify）
□ _private/ 中无敏感内容混入 content/posts/
□ 无真实用户名/邮箱/路径在正文中
□ git log 确认 commit message 无隐私信息
```

---

> 上一篇：[搭建笔记三：Obsidian Git 本地优先同步配置]({{< ref "搭建笔记四：Obsidian-Git本地优先同步配置" >}})
