---
title: "搭建笔记二：Obsidian + picgo-core + Gitee 图床 + launchd 开机自启"
date: 2026-07-18
tags: ["Obsidian","picgo","Gitee","图床","macOS","自动化"]
draft: false
---

# 搭建笔记二：Obsidian + picgo-core + Gitee 图床 + launchd 开机自启

> 本篇记录在博客写作流程中加入截图即传图床功能，并配置开机自启动的全部步骤。最终实现：在 Obsidian 中截图粘贴，图片自动上传至 Gitee，返回可直接引用的 URL。

<!-- more -->

## 1 为什么选 Gitee 图床而非 GitHub

### 1.1 原始方案：GitHub + PicGo GUI

最初选择 GitHub 作为图床，PicGo GUI 版配置简单，命令行 `gh auth` 已登录，推送应无障碍。

### 1.2 踩坑实录：`unable to verify the first certificate`

实际测试发现：PicGo 上传返回 `"unable to verify the first certificate"`，直接调 GitHub API 报错：

```
issuer=CN=SteamTools Certificate, OU=Technical Department, O=BeyondDimension, C=CN
Verify return code: 21 (unable to verify the first certificate)
```

**根因**：机器上安装了 SteamTools（BeyondDimension）代理工具，该工具对 `api.github.com` 做了 SSL 中间人拦截，用自签 CA 替换了 GitHub 的真实证书。系统工具（curl、gh CLI）可能通过系统钥匙串信任了该 CA，但 PicGo 内置的 Node.js/Electron 走了另一条证书链，导致 TLS 验证失败。

### 1.3 最终方案：Gitee 图床

测试发现 `gitee.com` 的 TLS 证书链完全正常（`Verify return code: 0`），由 TrustAsia CA 签发，不受 SteamTools 拦截影响。切换至 Gitee 图床，彻底绕过 GitHub 的证书问题。

**选择理由**：
- ✅ 国内直连，延迟低
- ✅ TLS 证书链正常，不受本地代理拦截影响
- ✅ 免费，仓库无流量限制
- ✅ API 与 GitHub 类似，PicGo 插件支持良好
- ❌ Gitee 图床 URL 在部分主题/平台访问受限（可通过 jsDelivr 等 CDN 中转解决）

## 2 Gitee 仓库准备

### 2.1 创建图床仓库

1. 登录 Gitee：https://gitee.com
2. 新建仓库：`blog-image`（公开，Initialize with README：否）
3. 仓库地址：`https://gitee.com/sjyb/blog-image`

### 2.2 获取私人令牌

Settings → 私人令牌 → 生成新令牌：

**必勾权限**：
- `projects`（项目访问）
- `pull_requests`（可选）
- `issues`（可选）

> ⚠️ 生成后立即复制保存，Gitee 不支持再次查看。

### 2.3 确认仓库分支名

进入 `blog-image` 仓库 → 主页查看默认分支名（通常为 `master` 或 `main`，本方案使用 `master`）。

## 3 安装 picgo-core（命令行版）

### 3.1 为什么用 picgo-core 而非 PicGo GUI

| 对比项 | PicGo GUI | picgo-core |
|--------|-----------|------------|
| 图形界面 | ✅ 有 | ❌ 无（纯命令行）|
| 系统资源 | 持续占用 | 仅上传时启动 |
| 图床插件 | 需手动安装到 GUI | npm 全局安装 |
| 开机自启 | 无官方支持 | 可通过 launchd 托管 |
| Obsidian 集成 | 调 36677 Server | 同样调 36677 Server |

**选择 picgo-core 的理由**：Obsidian 的图片上传插件只调 36677 Server，无需 GUI，core 版更轻量、更适合托管。

### 3.2 安装

```bash
# 通过 npm 全局安装
npm install -g picgo

# 验证安装
picgo -v
# 输出示例：3.0.1
```

### 3.3 安装 gitee 图床插件

```bash
picgo install gitee
# 输出：[PicGo SUCCESS]: Plugin installed successfully.
```

## 4 配置 picgo-core

### 4.1 配置目录

picgo-core 使用 `~/.picgo/` 作为配置目录（注意与 PicGo GUI 的 `~/Library/Application Support/picgo/` 区分，两者互不影响）。

```bash
mkdir -p ~/.picgo
```

### 4.2 配置文件

创建/编辑 `~/.picgo/config.json`：

```json
{
  "picBed": {
    "uploader": "gitee",
    "current": "gitee",
    "gitee": {
      "owner": "sjyb",
      "repo": "blog-image",
      "token": "这里填入你的 Gitee 私人令牌",
      "path": "img/",
      "branch": "master",
      "customUrl": "https://gitee.com/sjyb/blog-image/raw/master"
    }
  },
  "picgoPlugins": {
    "gitee": true
  },
  "settings": {
    "server": {
      "enable": true,
      "host": "127.0.0.1",
      "port": 36677
    }
  }
}
```

**各字段说明**：

| 字段 | 含义 | 示例 |
|------|------|------|
| `owner` | Gitee 用户名 | `sjyb` |
| `repo` | 图床仓库名 | `blog-image` |
| `token` | Gitee 私人令牌 | `18e0xxxxxxxxxx` |
| `path` | 上传路径（仓库内）| `img/` |
| `branch` | 分支名 | `master` |
| `customUrl` | 图片引用 URL 前缀 | 见下方说明 |

**关于 `customUrl`**：Gitee 图片有三种 URL 格式：

```
# blob 页面（HTML，有 Gitee UI 元素）
https://gitee.com/sjyb/blog-image/blob/master/img/xxx.png

# raw 直链（推荐，在网页中可直接显示图片）
https://gitee.com/sjyb/blog-image/raw/master/img/xxx.png

# jsDelivr CDN 中转（速度最快，推荐博客使用）
https://cdn.jsdelivr.net/gh/sjyb/blog-image@master/img/xxx.png
```

> ⚠️ 本方案使用 Gitee 作为源，博客 Hugo/PaperMod 主题在大多数情况下能直接访问 Gitee raw URL。若 Gitee raw 在部分网络环境下无法访问，可改用 jsDelivr CDN 格式或自建 Gitee Pages。

### 4.3 命令行上传测试

```bash
# 创建一张测试图
printf '\x89PNG\r\n\x1a\n' > /tmp/test.png

# 命令行直接上传（不走 server，走 config.json 配置）
picgo upload /tmp/test.png

# 预期输出：
# [PicGo SUCCESS]:
# https://gitee.com/sjyb/blog-image/raw/master/img/test.png
```

若上传失败，检查：
1. token 是否正确填入
2. 仓库 `sjyb/blog-image` 是否存在且为公开
3. token 是否有 `projects` 权限

## 5 启动 picgo Server（Obsidian 插件的桥梁）

### 5.1 手动启动

```bash
picgo server
# 输出：[PicGo INFO]: [PicGo Server] listening at http://127.0.0.1:36677
```

### 5.2 端口占用问题

如果端口 36677 已被占用（`Error: listen EADDRINUSE`），检查并停止占用进程：

```bash
# 查看谁在占用
lsof -iTCP:36677 -sTCP:LISTEN -n -P

# 通常是 PicGo GUI 在跑，退出它：
osascript -e 'quit app "PicGo"'

# 或直接 kill
pkill -f "PicGo.app"
```

### 5.3 验证 server 可用

```bash
# 测试 server 是否响应（返回 JSON 即正常）
curl -s -X POST http://127.0.0.1:36677/upload
# 预期：{"success":false,"result":[],"items":[],"message":"No files found..."}
# success:false 是因为没有发文件，这是正确的响应

# 若连接拒绝（Connection refused），说明 server 未启动
```

## 6 Obsidian 图片上传插件

### 6.1 插件选择

Obsidian 有两款常用图片上传插件：

| 插件 | 上传方式 | PicGo 集成 | 推荐 |
|------|---------|------------|------|
| `obsidian-image-auto-upload-plugin` | 调 PicGo Server（36677）| ✅ 天然支持 | **推荐** |
| `obsidian-image-upload` | 内置配置直传各平台 | 需填各平台 API Key | 不推荐 |

**本方案使用 `obsidian-image-auto-upload-plugin`**，优点：
- 不需要配置 Gitee/阿里云 OSS 的每个平台 API Key
- 所有图床配置集中在 picgo（`~/.picgo/config.json`），一处配置、全局生效
- 插件本身无需额外配置，只需指定 server 地址

### 6.2 安装插件

下载 release 资产，解压 `dist/` 目录内容到 Obsidian 插件目录：

```bash
# 下载（以 v4.1.0 为例）
VERSION="v4.1.0"
curl -L "https://ghproxy.net/https://github.com/Renato雾点/obsidian-image-auto-upload-plugin/releases/download/${VERSION}/obsidian-image-auto-upload-plugin.zip" \
  -o /tmp/obsidian-image-auto-upload-plugin.zip

# 解压并放入插件目录
unzip -q /tmp/obsidian-image-auto-upload-plugin.zip -d /tmp/iauap_tmp
mv /tmp/iauap_tmp/dist ~/.obsidian/plugins/obsidian-image-auto-upload-plugin/
```

### 6.3 目录名注意

**插件目录名必须与 `manifest.json` 中的 `id` 完全一致**：

```bash
# 查看 manifest id
cat ~/.obsidian/plugins/obsidian-image-auto-upload-plugin/manifest.json | python3 -c "
import sys,json
m=json.load(sys.stdin)
print('id:', m['id'])
print('version:', m['version'])
print('minAppVersion:', m['minAppVersion'])
"
```

若 `manifest.json` 中的 `id` 是 `obsidian-image-auto-upload-plugin`，目录名必须为 `obsidian-image-auto-upload-plugin`（不是 `obsidian-image-auto-upload`）。

### 6.4 启用插件

编辑 `~/.obsidian/community-plugins.json`，将插件 id 加入列表：

```json
[
  "obsidian-git",
  "templater-obsidian",
  "obsidian-image-auto-upload-plugin"
]
```

重启 Obsidian（完全退出 `Cmd+Q`，重新打开），在 设置 → 社区插件 中确认三个插件都已启用且无红色报错。

### 6.5 插件配置

打开 Obsidian 设置 → `obsidian-image-auto-upload-plugin`：

| 配置项 | 值 |
|--------|-----|
| Uploader | PicGo（默认）|
| PicGo Server | `http://127.0.0.1:36677/upload` |
| Default Behaviour | Auto Upload |
| Show Status Bar | ✅（左下角显示上传进度）|

## 7 完整链路测试

### 7.1 上传一张真实图片

1. 在 Obsidian 任意笔记中，截一张屏幕截图（`Cmd+Shift+4`）
2. 切换到 Obsidian，`Cmd+V` 粘贴
3. 观察左下角状态栏，出现「上传完成」提示
4. 笔记中自动插入图片链接，形如：

```markdown
![](https://gitee.com/sjyb/blog-image/raw/master/img/2026-07-18-171300.png)
```

### 7.2 验证图片 URL

在浏览器中打开该 URL，确认图片正常显示。

### 7.3 端到端链路检查

```bash
# 1. picgo server 在跑
lsof -iTCP:36677 -sTCP:LISTEN -n -P | grep node
# 预期：COMMAND PID ... node <pid>

# 2. picgo config 中 gitee 配置正确
python3 -c "
import json
d=json.load(open('$HOME/.picgo/config.json'))
g=d['picBed']['gitee']
print('uploader:', d['picBed']['uploader'])
print('owner/repo:', g['owner']+'/'+g['repo'])
print('branch:', g['branch'])
print('token 前缀:', g['token'][:4]+'****')
"

# 3. Gitee API 直测（验证 token 有效）
curl -s -H "Authorization: token $(python3 -c 'import json;print(json.load(open(\"$HOME/.picgo/config.json\"))[\"picBed\"][\"gitee\"][\"token\"])')" \
  https://gitee.com/api/v5/repos/sjyb/blog-image | python3 -c "
import sys,json
d=json.load(sys.stdin)
print('仓库:', d.get('full_name'), '| 默认分支:', d.get('default_branch'))
"
```

## 8 配置 launchd 开机自启动

### 8.1 为什么需要 launchd

picgo server 目前是手动启动的，重启 Mac 后 36677 端口无监听，Obsidian 粘贴图片会失败（报 `upload failed`）。需要将其托管为系统服务，开机自动拉起。

**方案选择**：macOS `launchd`（用户级 LaunchAgent），优于 `cron`（不支持服务常驻）和 `systemd`（Linux 专属）。

### 8.2 定位 picgo 命令路径

```bash
which picgo
# 示例输出：/Users/sjyb/Library/Application Support/QClaw/npm-global/bin/picgo

# 确认 node 路径（launchd 需要完整 PATH）
which node
# 示例输出：/Users/sjyb/Library/Application Support/QClaw/openclaw/config/bin/node/node
```

### 8.3 创建 LaunchAgent plist

```bash
mkdir -p ~/Library/LaunchAgents
```

创建 `~/Library/LaunchAgents/com.sjyb.picgo-server.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.sjyb.picgo-server</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Users/sjyb/Library/Application Support/QClaw/npm-global/bin/picgo</string>
        <string>server</string>
    </array>

    <key>EnvironmentVariables</key>
    <dict>
        <key>HOME</key>
        <string>/Users/sjyb</string>
    </dict>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/tmp/picgo_launchd.out</string>

    <key>StandardErrPath</key>
    <string>/tmp/picgo_launchd.err</string>
</dict>
</plist>
```

**各配置项说明**：

| 配置项 | 作用 |
|--------|------|
| `Label` | 唯一标识，launchd 用它管理此 Agent |
| `ProgramArguments` | 启动命令（picgo server）|
| `RunAtLoad` | 登录后自动启动（不等首次请求）|
| `KeepAlive` | 进程退出后自动重启（确保常驻）|
| `StandardOutPath / StandardErrPath` | 日志输出位置，方便排查 |

### 8.4 加载并验证

```bash
# 先停掉手动起的旧 server（让 launchd 接管独占端口）
pkill -f "picgo server"

# 加载 LaunchAgent
launchctl load ~/Library/LaunchAgents/com.sjyb.picgo-server.plist

# 验证 36677 已被 launchd 托管的进程监听
lsof -iTCP:36677 -sTCP:LISTEN -n -P | awk 'NR==2{print "进程:",$1,"PID:",$2}'

# 验证 launchd 注册成功
launchctl list | grep picgo-server
```

### 8.5 管理命令汇总

```bash
# 卸载（停止并从 launchd 移除）
launchctl unload ~/Library/LaunchAgents/com.sjyb.picgo-server.plist

# 重新加载（修改 plist 后）
launchctl unload ~/Library/LaunchAgents/com.sjyb.picgo-server.plist
launchctl load ~/Library/LaunchAgents/com.sjyb.picgo-server.plist

# 查看启动日志
tail -f /tmp/picgo_launchd.err

# 查看 launchd 状态
launchctl list | grep picgo-server
```

### 8.6 重启 Mac 验证

重启后：
1. 登录 macOS
2. 等待约 5 秒（launchd 拉起需要一点时间）
3. 检查 36677 是否在监听：`lsof -iTCP:36677 -sTCP:LISTEN -n -P | grep node`
4. 在 Obsidian 中粘贴图片，验证上传成功

## 9 完整技术栈全景

```
Obsidian 截图粘贴 (Ctrl+V)
        │
        ▼
obsidian-image-auto-upload-plugin
        │ 调 http://127.0.0.1:36677/upload
        ▼
picgo-core server (launchd 托管)
        │ ~/.picgo/config.json 读取 gitee 配置
        ▼
Gitee API (https://gitee.com/api/v5/repos/sjyb/blog-image/contents/)
        │ token 鉴权，写入 img/ 目录
        ▼
图片 URL 返回 Obsidian，插入笔记
        │
        ▼
Hugo 构建 → GitHub Actions → GitHub Pages
```

## 10 常见问题排查

### 10.1 `upload failed, check dev console`

最常见原因，按以下顺序排查：

```bash
# 1. picgo server 是否在跑
lsof -iTCP:36677 -sTCP:LISTEN -n -P | grep node

# 2. 手动传一张图看报错
curl -s -X POST http://127.0.0.1:36677/upload \
  -F "files=@/tmp/test.png"
# 若返回 "All uploads failed"：检查 ~/.picgo/config.json 的 token 和 repo

# 3. 读 picgo 日志
cat ~/.picgo/picgo.log | grep -iE "error|fail|401|403|404|500"
```

### 10.2 token 失效（401 Unauthorized）

Gitee 私人令牌有有效期，或被手动吊销。重新生成后更新 `~/.picgo/config.json` 中的 `token` 字段，然后：

```bash
# 重启 picgo server（让 launchd 重新读配置）
launchctl unload ~/Library/LaunchAgents/com.sjyb.picgo-server.plist
launchctl load ~/Library/LaunchAgents/com.sjyb.picgo-server.plist
```

### 10.3 Gitee API 500 错误

可能是 Gitee API 偶发性错误，或 PicGo gitee 插件的请求格式与当前 Gitee API 版本不兼容。先用命令行直接测：

```bash
picgo upload /tmp/test.png
```

若命令行成功但 server 失败，说明 server 启动时的配置与当前不同（检查 launchd 日志 `tail -f /tmp/picgo_launchd.err`）。

### 10.4 博客中图片显示不出来

Gitee raw URL 在某些网络（公司内网、学校网络）下可能无法访问。可改为 jsDelivr CDN 格式：

将 `customUrl` 改为：
```
https://cdn.jsdelivr.net/gh/sjyb/blog-image@master
```
并确保 Gitee 仓库已开源（jsDelivr 只能代理公开仓库）。

### 10.5 macOS 重启后 launchd 未自动拉起

检查 plist 语法：

```bash
plutil ~/Library/LaunchAgents/com.sjyb.picgo-server.plist
# 输出：... OK 表示语法正确
```

确认 `RunAtLoad: true` 存在，且 `Label` 唯一（不与其他 Agent 冲突）。

---

> 上一篇：[搭建笔记一：Hugo + Obsidian + GitHub Pages 全链路配置]({{< ref "搭建笔记一：Hugo-Obsidian-GitHub全链路配置" >}})
