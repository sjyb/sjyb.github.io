---
title: "搭建笔记4：Obsidian Git 本地优先同步配置"
date: 2026-07-18T11:00:00+08:00
description: "Obsidian Git 插件配置：本地优先策略、10 分钟自动备份、作者信息设置"
ShowToc: true
TocOpen: true
weight: 40
tags: ["Obsidian","Git","同步","本地优先","博客"]
draft: false
---


> 本篇记录博客笔记仓库的 Obsidian Git 插件完整配置：以本地文件为最新版本（Local-First），自动备份 + 自动推送 + 自动拉取，双向同步零障碍。

<!-- more -->

## 1 核心设计原则：Local-First

本博客的工作流决定了**本地始终是最优先版本**：

```
Obsidian（本地）→ 写文章 → Git 提交 → GitHub Pages 线上
```

理由：
- Obsidian 是主要写作工具，所有草稿、修改都在本地完成
- GitHub 仓库同时承载 Hugo 博客源码，内容发布路径是本地 → 远端
- 远端（如 GitHub 网页编辑）不应覆盖本地工作区
- 单作者仓库，冲突极少；出现时以本地为准

## 2 Obsidian Git 插件配置

### 2.1 完整配置清单

打开 Obsidian → 设置 → Obsidian Git：

| 配置项 | 建议值 | 说明 |
|--------|--------|------|
| Vault backup interval | 10 分钟 | 自动 commit 间隔 |
| Auto commit after commit | ✅ 开启 | commit 后自动 push |
| Pull updates interval | 10 分钟 | 自动从远端拉取更新 |
| Auto pull on startup | ✅ 开启 | Obsidian 启动时先拉取远端 |
| Pull before push | ✅ 开启 | push 前先 pull，避免落后远端 |
| Commit author | `** <**@**.**>` | 提交作者（脱敏后）|
| Auto backup after file change | ❌ 关闭 | 不频繁触发（省资源）|
| Sync method | Merge | 双向同步（push + pull）|
| Merge strategy | None | 使用 git 默认 merge |

> ⚠️ 如果未设置 Commit author，插件会尝试读取 git 全局配置。若 git 全局配置也未设置，commit 作者信息会不完整。

### 2.2 仓库级 git 用户信息（推荐）

在博客仓库目录下单独设置 git 用户信息（不依赖全局配置）：

```bash
cd ~/blog

# 设置仓库级用户信息（覆盖全局）
git config user.name "**"
git config user.email "**@**.**"

# 验证
git config --list | grep "^user"
# 输出：
# user.name=**
# user.email=**@**.**
```

> 若 git 全局配置已有用户信息，且不想改动全局设置，仓库级配置是更干净的选择。

### 2.3 git pull 策略配置

```bash
# 合并策略：不 rebase，使用 merge（保留本地提交历史）
git config pull.rebase false

# push 时若远端有更新，自动 merge（pullBeforePush 已开启）
# 冲突时 git 默认行为：提示用户手动解决
```

## 3 远端→本地拉取测试

验证 GitHub 上的改动能正确拉回到本地：

```bash
cd ~/blog

# 先确保没有未提交的本地改动
git status
# 若有未提交改动，先 commit + push

# 手动拉取测试
git fetch origin
git log --oneline origin/main -3   # 查看远端最新 3 条 commit
git pull origin main               # 拉取并合并
```

正常输出示例：
```
From https://github.com/**/**.github.io
 * branch            main       -> FETCH_HEAD
Already up to date.
```

## 4 冲突处理：本地优先原则

### 4.1 何时会产生冲突

| 场景 | 是否冲突 | 处理方式 |
|------|---------|---------|
| 仅本地修改，远端无变化 | ❌ 无 | 自动 push 成功 |
| 仅远端修改，本地无变化 | ❌ 无 | 自动 pull 成功 |
| 本地 + 远端同时修改**同一文件** | ⚠️ 可能 | 本地优先，手动处理 |
| 先 push，Obsidian 自动 pull 发现冲突 | ⚠️ 可能 | 见下方流程 |

### 4.2 冲突处理流程（本地优先）

当 Obsidian Git 提示冲突时，按以下优先级处理：

**步骤 1：确认本地版本是最新的**

```bash
cd ~/blog
git status
# 查看哪些文件冲突（显示 "both modified:"）

# 查看本地改动
git diff content/posts/冲突文件.md
```

**步骤 2：保留本地版本，强制覆盖远端（本地优先）**

```bash
# 方案 A：放弃远端改动，强制以本地为准
git checkout --ours content/posts/冲突文件.md
git add content/posts/冲突文件.md
git commit -m "chore: 解决冲突，以本地版本为准"
git push origin main
```

```bash
# 方案 B：手动编辑文件，选出想要的版本，然后 commit
# （Obsidian 打开文件，插件会显示 <<<<<< 和 >>>>>> 标记）
# 编辑完成后：
git add content/posts/冲突文件.md
git commit -m "chore: 手动解决冲突"
git push origin main
```

**步骤 3：验证冲突已解决**

```bash
git status
# 期望输出：working tree clean
```

### 4.3 预防冲突的最佳实践

- **每次打开 Obsidian 前**：确保上次已 commit（插件会自动在 10 分钟后 commit + push）
- **在 GitHub 网页编辑后**：回到 Obsidian 等待自动 pull（约 10 分钟内），或手动触发 pull
- **长时间离线写作后**：联网后先在 Obsidian 中打开命令面板（`Cmd+P`）→ `Obsidian Git: Pull` 手动拉取
- **避免 GitHub 网页编辑**：所有写作统一在 Obsidian 本地进行

## 5 完整同步链路验证

### 5.1 链路一：本地 → 远端（Obsidian → GitHub）

```
Obsidian 编辑文章 → Ctrl+S 保存
        ↓
Obsidian Git 插件（每 10 分钟自动检测改动）
        ↓
git add . && git commit -m "docs: YYYY-MM-DD 自动同步"
        ↓
git push origin main
        ↓
GitHub Actions 检测到 push
        ↓
hugo --minify（构建静态页面）
        ↓
GitHub Pages 部署（约 20-30 秒）
        ↓
https://**.github.io/posts/文章标题/  可访问
```

**手动触发验证**：

```bash
# 在 Obsidian 命令面板执行：
# Cmd+P → "Obsidian Git: Commit & push"

# 或在终端手动执行（验证 git 配置正常）：
cd ~/blog
git status
git log --oneline -3
```

### 5.2 链路二：远端 → 本地（GitHub → Obsidian）

```
GitHub 网页编辑 / 其他设备推送
        ↓
Obsidian Git 插件（每 10 分钟自动拉取）
        ↓
git fetch + git pull origin main
        ↓
Obsidian 检测到文件系统变化
        ↓
笔记内容自动更新为最新版本
```

**手动触发验证**：

```bash
# 在 Obsidian 命令面板执行：
# Cmd+P → "Obsidian Git: Pull from remote"
```

## 6 常见问题排查

### 6.1 push 失败：`! [rejected] non-fast-forward`

**原因**：本地落后于远端（远端有本地没有的 commit）。

**解决**：

```bash
git fetch origin
git status   # 查看详情

# 方案 A：合并远端更新（推荐）
git pull origin main
# 若有冲突，按上方冲突处理流程解决
git push origin main

# 方案 B：强制以本地为准（会丢失远端 commit，极少使用）
git push origin main --force
```

### 6.2 pull 失败：`fatal: refusing to merge unrelated histories`

**原因**：本地和远端仓库没有共同的 commit 起点（通常是初始化方式不同）。

**解决**：

```bash
git pull origin main --allow-unrelated-histories
# 解决冲突后 commit + push
```

### 6.3 Obsidian Git 插件不显示

**检查清单**：

```bash
# 1. 确认插件目录存在且目录名正确
ls ~/.obsidian/plugins/obsidian-git/
# 应包含 main.js, manifest.json, styles.css

# 2. 确认 community-plugins.json 中有 "obsidian-git"
cat ~/.obsidian/community-plugins.json

# 3. 在 Obsidian 中：设置 → 社区插件 → 确认 obsidian-git 已启用
```

### 6.4 隐私信息泄露

**风险场景**：git commit 历史中包含真实用户名/邮箱。

**预防措施**：

- 仓库级 git config 使用脱敏用户信息
- 不在 commit message 中包含真实项目名或个人标识
- 定期检查远端仓库的 commit 历史：

```bash
git log --all --format="%h %ae %s" | head -20
```

### 6.5 私有内容被误 push

**检查 `.gitignore` 配置**：

```bash
cat ~/blog/.gitignore
```

确保以下内容在 `.gitignore` 中（不会上网）：

```
# Hugo 构建产物
public/
resources/

# Obsidian 配置
.obsidian/

# 私有内容
_private/
```

> `_private/` 目录用于存放靶机 writeup 草稿（含真实 IP/flag 等敏感信息），绝不会 push 到 GitHub。

---

---

> 下一篇：[搭建笔记3：macOS launchd 开机自启动 picgo server]({{< ref "搭建笔记3：macOS-launchd-开机自启动-picgo-server" >}})
