---
title: "搭建笔记二-b：macOS launchd 开机自启动 picgo server"
date: 2026-07-18
tags: ["macOS","launchd","picgo","自动化","开机自启"]
draft: false
---

# 搭建笔记二：macOS launchd 开机自启动 picgo server

> 本篇记录将 picgo server 托管为 macOS LaunchAgent，实现开机/登录后自动拉起，开机即用的全过程。配合 [搭建笔记二-a：Obsidian + picgo-core + Gitee 图床配置]({{< ref "搭建笔记二：Obsidian-picgo-core-Gitee图床配置" >}}) 使用。

<!-- more -->

## 1 问题：为什么需要 launchd

picgo server 目前是手动启动的，每次重启 Mac 后 36677 端口无监听，Obsidian 粘贴图片会失败（报 `upload failed`）。

需要将其托管为系统服务，开机/登录自动拉起。

**方案对比**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| `launchd`（LaunchAgent）| macOS 原生、登录即启、进程常驻 | 需写 plist，有一定门槛 |
| `cron @reboot` | 简单 | cron 只定时执行，无法常驻服务 |
| `systemd` | Linux 原生、功能强大 | macOS 不支持 |
| 第三方工具（Lingon 等）| 图形界面 | 额外依赖、不透明 |

macOS 推荐 **launchd**（用户级 LaunchAgent），稳定可靠，与系统深度集成。

## 2 定位 picgo 命令路径

```bash
which picgo
# 示例输出：/your/path/to/npm-global/bin/picgo

# 确认 node 路径（launchd 需要完整 PATH）
which node
# 示例输出：/your/path/to/node
```

## 3 创建 LaunchAgent plist

```bash
mkdir -p ~/Library/LaunchAgents
```

创建 `~/Library/LaunchAgents/com.**.picgo-server.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.**.picgo-server</string>

    <key>ProgramArguments</key>
    <array>
        <string>/your/path/to/npm-global/bin/picgo</string>
        <string>server</string>
    </array>

    <key>EnvironmentVariables</key>
    <dict>
        <key>HOME</key>
        <string>/Users/<用户名></string>
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

> ⚠️ plist 文件名中的 `**` 替换为你的用户名（与 Label 保持一致）。

## 4 加载并验证

### 4.1 加载 LaunchAgent

```bash
# 先停掉手动起的旧 server（让 launchd 接管独占端口）
pkill -f "picgo server"

# 加载 LaunchAgent
launchctl load ~/Library/LaunchAgents/com.**.picgo-server.plist
```

### 4.2 验证加载成功

```bash
# 验证 36677 已被 launchd 托管的进程监听
lsof -iTCP:36677 -sTCP:LISTEN -n -P | awk 'NR==2{print "进程:",$1,"PID:",$2}'

# 验证 launchctl 注册成功
launchctl list | grep picgo-server
```

### 4.3 端到端验证

```bash
# 模拟 Obsidian 插件上传请求
TS=$(date +%s)
curl -s -X POST http://127.0.0.1:36677/upload \
  -F "picgo-uploader=gitee" \
  -F "files=@/tmp/test.png"

# 预期返回：{"success":true,"result":["https://gitee.com/**/blog-image/raw/master/img/launchd_${TS}.png"]}
```

## 5 完整技术栈全景

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
Gitee API (https://gitee.com/api/v5/repos/**/blog-image/contents/)
        │ token 鉴权，写入 img/ 目录
        ▼
图片 URL 返回 Obsidian，插入笔记
        │
        ▼
Hugo 构建 → GitHub Actions → GitHub Pages
```

## 6 管理命令汇总

```bash
# 卸载（停止并从 launchd 移除）
launchctl unload ~/Library/LaunchAgents/com.**.picgo-server.plist

# 重新加载（修改 plist 后）
launchctl unload ~/Library/LaunchAgents/com.**.picgo-server.plist
launchctl load ~/Library/LaunchAgents/com.**.picgo-server.plist

# 查看 stdout 日志
cat /tmp/picgo_launchd.out

# 查看 stderr 日志（报错看这里）
tail -f /tmp/picgo_launchd.err

# 查看 launchd 状态
launchctl list | grep picgo-server

# 重启 Mac 后验证自动拉起
# 重启后等待约 5 秒，然后：
lsof -iTCP:36677 -sTCP:LISTEN -n -P | grep node
```

## 常见问题排查

### macOS 重启后 launchd 未自动拉起

**检查一**：plist 语法是否正确：

```bash
plutil ~/Library/LaunchAgents/com.**.picgo-server.plist
# 输出：... OK 表示语法正确
# 若报错：... Syntax Error 则 XML 格式有问题
```

**检查二**：`RunAtLoad: true` 是否存在：

```bash
grep -c "RunAtLoad" ~/Library/LaunchAgents/com.**.picgo-server.plist
# 输出 1 表示正常
```

**检查三**：Label 是否唯一（不与其他 Agent 冲突）：

```bash
launchctl list | grep -i picgo
```

**检查四**：日志是否有报错：

```bash
cat /tmp/picgo_launchd.err
```

### `upload failed, check dev console`

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

### Gitee API 500 错误

可能是 Gitee API 偶发性错误，或 PicGo gitee 插件的请求格式与当前 Gitee API 版本不兼容。先用命令行直接测：

```bash
picgo upload /tmp/test.png
```

若命令行成功但 server 失败，说明 server 启动时的配置与当前不同（检查 `tail -f /tmp/picgo_launchd.err`）。

---

> 上一篇：[搭建笔记二-a：Obsidian + picgo-core + Gitee 图床配置]({{< ref "搭建笔记二：Obsidian-picgo-core-Gitee图床配置" >}})
