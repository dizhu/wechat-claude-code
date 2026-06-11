# 本地加固补丁记录 / Local Hardening Patches

这是在上游仓库基础上做的本地安全/正确性加固，**不是上游官方代码**。
记录目的：上游更新（`git pull`）合并冲突时，能一眼看出哪些改动是本地加固、需要保留。

- **基线 commit**：`8aead25` (`fix: support Windows paths in auto-push file detection (#30)`)
- **打补丁日期**：2026-06-12
- **完整 diff**：见同目录 `LOCAL_PATCHES.patch`（仅 `src/`，可用 `git apply LOCAL_PATCHES.patch` 重放）
- **改动文件**：`src/wechat/media.ts`、`src/wechat/api.ts`、`src/wechat/upload.ts`、`src/main.ts`、`src/claude/provider.ts`

> 注：`package-lock.json` 的变化是 `npm install` 自动重新生成的，与本加固无关，不在本记录范围内。

---

## 背景：为什么要加固

本工具把个人微信桥接到本地 `claude` CLI，且 Claude 以 `--dangerously-skip-permissions`（全权限，无拦截）运行——
**能给 bot 发消息 ≈ 能在本机执行任意命令**。经评估后决定保留全权限（微信端无法回应权限弹窗），
因此「只有绑定的 owner 微信号能发消息」这道**发送方鉴权**成为唯一防线。下列改动主要是把这道防线焊牢，
并堵住文件出入口与进程生命周期的几个洞。

---

## 安全加固（4 项）

### 1. 文件名路径穿越 — `src/wechat/media.ts`
**问题**：别人发文件给 bot 时，`file_name` 字段直接 `path.join` 到临时目录，无净化。
构造 `../../../` 文件名可写出临时目录、覆盖任意可写文件。
**修复**：先 `path.basename()` 剥掉目录部分，再加「解析后路径必须在 tmpDir 内」的容器断言，否则拒收。

### 2. 优先命令鉴权 — `src/main.ts` (`handlePriorityCommand`)
**问题**：`/stop`、`/clear` 走优先通道，完全没有发送方校验——任何拿到 bot 的人都能打断/清空 owner 会话。
**修复**：补上与主消息路径一致的 `account.userId` 校验。

### 3. 启动 fail-closed — `src/main.ts` (`runDaemon`)
**问题**：发送方鉴权依赖 `account.userId`；若该字段为空，校验被跳过 = 谁都能发，裸奔运行。
**修复**：daemon 启动时若 `account.userId` 为空，拒绝启动并提示重新 setup 绑定。

### 4. 上传地址白名单 — `src/wechat/api.ts` + `src/wechat/upload.ts`
**问题**：服务端返回的 `upload_full_url` 被直接拿来 POST 加密文件，未经域名校验。
API 被 MITM/劫持时，加密文件会被上传到攻击者主机。
**修复**：抽出共享函数 `isTrustedWechatUrl()`（https + `weixin.qq.com`/`wechat.com` 主机白名单），
`baseUrl` 与 `upload_full_url` 都走它；构造函数里原重复的白名单逻辑也改为调用该函数。
已用 6 个用例验证（含子域名欺骗 `weixin.qq.com.evil.com`、明文 http 等）。

---

## 正确性 / 体验（2 项）

### 5. 僵尸进程：SIGTERM → SIGKILL 升级 — `src/claude/provider.ts`
**问题**：超时/中止时只发 `SIGTERM`，若 `claude` 进程忽略该信号则永不退出，长期累积僵尸进程。
**修复**：新增 `terminateChild()`——先 SIGTERM，5 秒 grace 后若进程仍存活则 SIGKILL；
进程正常 `close` 时清掉 grace 定时器；用 `exitCode`/`signalCode` 判断避免对已死进程重复发信号。

### 6. 错误信息透传微信 — `src/main.ts`
**问题**：`claude` 出错只回「请稍后重试」，真实原因（如 `spawn claude ENOENT`、鉴权失败）只埋在日志里，手机端无从排查。
**修复**：把真实错误截断 300 字直接发到微信。

---

## 已知但**有意未修**的上游正确性 bug

以下为代码 review 发现、但判断为「偶发、不影响安全、改动面大、易与上游冲突」而保留的问题，留作记录：

- `/clear` 在任务进行中会静默中止查询且不真正清会话（`main.ts:handlePriorityCommand`）
- resume 重试会重复发送已流式输出的半截文本（`main.ts` retry 路径未重置 buffer）
- CDN 上传只在 HTTP 5xx 重试，网络错误（ECONNRESET）直接失败（`upload.ts`）
- 轮询循环在 API 返回非零 ret 时不退避，可能热转空跑（`monitor.ts`）
- `/clear` 会顺手把工作目录和模型设置重置成默认（`handlers.ts` 未传 currentSession）
- 日志时区写死 UTC+8；`cleanupOldLogs` 在每次写日志的热路径上跑；零测试覆盖
