# WeChat Claude Code Bridge

<p align="center">
  <strong>在微信里和 Claude Code 聊天，就像和朋友发消息一样</strong>
</p>

<p align="center">
  <a href="./LICENSE"><img src="https://img.shields.io/badge/License-MIT-green?style=flat-square" alt="License: MIT"></a>
  <a href="README_en.md"><img src="https://img.shields.io/badge/Lang-English-lightgrey?style=flat-square" alt="English"></a>
</p>

扫码绑定微信后，你的微信里会多出一个好友。给它发消息，消息会自动转发给你电脑上运行的 Claude Code，回复也会实时推送到微信。支持文字、图片、语音、文件的收发。

> **本仓库是 [Wechat-ggGitHub/wechat-claude-code](https://github.com/Wechat-ggGitHub/wechat-claude-code) 的安全加固 + 多人支持 fork。**
> 在上游基础上做了 20+ 项安全/正确性加固，并把单用户桥接扩展成「一个守护进程同时服务多个 bot、多名成员、每人会话独立」。
> 所有本地改动的清单与理由见 [`LOCAL_PATCHES.md`](./LOCAL_PATCHES.md)。**使用前请先读下方 [⚠️ 安全须知](#-安全须知)。**

## 核心亮点
| | |
|---|---|
| **扫码即用** | 不用注册账号，不用部署服务器。微信扫码绑定，一分钟搞定。凭证全在本地。 |
| **多人 / 团队** | 一个守护进程可同时服务多名成员，每人各绑自己的 bot，会话、上下文、工作目录互相隔离。 |
| **消息不刷屏** | 只推送核心信息——进度、结果、关键决策。工具调用等噪音自动过滤。 |
| **"对方正在输入中..."** | Claude 处理任务时，微信顶部显示输入状态，随时感知它在干活。 |
| **电脑手机体验一致** | 手机端和电脑端 Claude Code 行为完全相同——同样的编排逻辑、同样的输出。 |
| **文件双向收发** | 发图片、Word、PDF 给 Claude 分析；Claude 生成的文件也会直接推送到微信。 |
| **超时安抚** | 任务超过 5 分钟没响应？会自动发消息告诉你还在干，不让你对着空白聊天框干等。 |
| **安全加固** | 发送方鉴权、路径穿越防护、上传地址白名单、僵尸进程清理、启动 fail-closed 等，详见 `LOCAL_PATCHES.md`。 |

## 快速安装

```bash
git clone https://github.com/dizhu/wechat-claude-code.git ~/.claude/skills/wechat-claude-code
cd ~/.claude/skills/wechat-claude-code && npm install
```

> 需要 Node.js >= 18、macOS 或 Linux、个人微信账号、已安装并认证的 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI。
> `npm install` 会自动 `npm run build` 编译 TypeScript。

## 快速开始（单人）

### 1. 扫码绑定

```bash
cd ~/.claude/skills/wechat-claude-code
npm run setup
```

弹出二维码，用微信扫码确认。

### 2. 启动服务

```bash
npm run daemon -- start
```

macOS 下自动注册 launchd，开机自启、崩溃自动重启。启动后会打印当前服务的 bot 数量，例如 `已启动 (1 个 bot: xxx@im.bot)`。

### 3. 开始聊天

打开微信，给你新出现的那个"好友"发条消息试试。

### 管理服务

```bash
npm run daemon -- status   # 查看运行状态
npm run daemon -- stop     # 停止服务
npm run daemon -- restart  # 重启服务（更新代码或新增成员后使用）
npm run daemon -- logs     # 查看日志
```

## 多人 / 团队使用

ilink bot 是 **1:1 设计**——每个 bot 只能绑定一个微信号，无法分享名片让别人加入同一个 bot。
因此多人的正确做法是：**每名成员各扫一次码、绑定自己的独立 bot，由同一个守护进程统一服务。**
每名成员的会话上下文、聊天历史、工作目录完全隔离，不同成员的请求并发执行、互不阻塞。

**新增一名成员：**

1. 在宿主机生成一张绑定二维码：
   ```bash
   npm run setup
   ```
2. 把二维码图片（`~/.wechat-claude-code/qrcode.png`）发给该成员，让他**用自己的手机微信扫码并确认**。
   bot 会出现在**他的**微信里，凭证保存到宿主机 `~/.wechat-claude-code/accounts/<新id>.json`。
3. 重启服务，守护进程自动加载并服务新增 bot：
   ```bash
   npm run daemon -- restart
   ```
   启动日志会显示服务的 bot 总数，例如 `已启动 (2 个 bot: ...)`。

> 每名成员默认有独立工作目录 `<config.workingDirectory>/<用户短ID>`，自动创建；成员可用 `/cwd` 自行切换。

## 微信端命令

直接在微信聊天中发送即可：

| 命令 | 说明 |
|------|------|
| `/help` | 显示帮助 |
| `/clear` | 清除当前会话，开始新对话（保留工作目录/模型/提示词） |
| `/stop` | 停止当前任务，并清空排队消息 |
| `/model <名称>` | 切换 Claude 模型 |
| `/prompt <内容>` | 设置系统提示词（**仅对你自己生效**，如"用中文回答"） |
| `/cwd <路径>` | 切换你自己的工作目录 |
| `/skills` | 查看已安装的 Skill |
| `/status` | 查看当前会话状态 |
| `/history [数量]` | 查看最近对话记录 |
| `/compact` | 压缩上下文，开始新 CLI 会话 |
| `/reset` | 完全重置（包括工作目录等设置） |
| `/undo [数量]` | 撤销最近几条对话 |
| `/<skill> [参数]` | 触发任意已安装的 Skill |

## 工作原理

```
成员A 微信 ─┐
成员B 微信 ─┼─→ ilink Bot API ─→ Node.js 守护进程 ─→ Claude Code CLI（本地，逐成员独立会话）
成员C 微信 ─┘                         （每 bot 一个轮询循环 + 独立游标）
```

守护进程为每个绑定的 bot 启动一条独立的长轮询循环（各自独立的游标，互不干扰），收到消息后按发送人路由到隔离的会话，转发给本地 `claude` CLI 处理，回复实时流式推送回微信。全程跑在你自己电脑上。

## ⚠️ 安全须知

**请务必理解：本工具让微信消息能在宿主机上以全权限执行任意命令。**

- **全权限运行**：`claude` 以 `--dangerously-skip-permissions` 启动，无任何确认弹窗（微信端无法回应弹窗）。
  **能给某个 bot 发消息 ≈ 能在宿主机上执行任意命令**——读写文件、跑 shell、联网。
- **共享机器、共享凭证**：所有成员的 Claude 都在**同一台机器、同一个系统账号**下运行。会话上下文隔离了，
  但**机器没有隔离**——任何成员的 Claude 都能读到这台机器上的文件、SSH key、各类 token。
  👉 **强烈建议把守护进程跑在一台专用的、不存放敏感凭证的机器上，而不是你的主力开发机。**
- **凭证安全**：bot token 存在 `~/.wechat-claude-code/accounts/`（权限 `0600`）。
  **不要把任何 bot 好友推荐给无关的人，不要外泄 accounts 目录。**
- **账号封禁风险**：这是架在个人微信上的非官方桥接，多 bot、持续流量更容易触发微信风控。请留意账号状态。
- **鉴权机制**：默认只有绑定的 owner 本人能驱动各自的 bot；启动时若账号缺少绑定用户会 fail-closed 拒绝该 bot。

更多加固细节见 [`LOCAL_PATCHES.md`](./LOCAL_PATCHES.md)。

## 前置条件

- Node.js >= 18
- macOS 或 Linux
- 个人微信账号（每名成员各需一个）
- 已安装 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI 并完成认证

> **提示：** Claude Code 支持第三方 API 提供商（OpenRouter、AWS Bedrock 等），设置 `ANTHROPIC_BASE_URL` 和 `ANTHROPIC_API_KEY` 即可。
> 注意：多人共用一个守护进程时，所有成员的用量都计入宿主机配置的同一个 API key。

## 数据目录

所有数据存储在 `~/.wechat-claude-code/`：

```
~/.wechat-claude-code/
├── accounts/       # 各 bot 的微信账号凭证（每 bot 一个 <accountId>.json，0600）
├── config.json     # 全局配置（工作目录、模型、系统提示词等）
├── sessions/       # 会话数据（按 <accountId>__<用户> 分文件，每成员独立）
├── sync/           # 长轮询游标（每 bot 一个 <accountId>.json，互不覆盖）
└── logs/           # 运行日志（每日轮转，保留 30 天）
```

## 致谢

Fork 自 [Wechat-ggGitHub/wechat-claude-code](https://github.com/Wechat-ggGitHub/wechat-claude-code)，遵循 MIT 协议。
本 fork 增加了多人/多 bot 支持与一系列安全/正确性加固，详见 [`LOCAL_PATCHES.md`](./LOCAL_PATCHES.md)。

## License

[MIT](LICENSE)
