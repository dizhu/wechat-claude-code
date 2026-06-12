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

---

## 多用户改造（2026-06-12，第二批）

把原本「单 owner」的桥接改成**多员工、每人独立会话**。⚠️ 安全前提：每个被授权的用户都拥有在本机执行任意命令的能力（全权限模型未变），仅限可信同事内部使用。

涉及文件：`src/config.ts`、`src/session.ts`、`src/commands/handlers.ts`、`src/main.ts`

- **鉴权改 allowlist**（`config.ts` + `main.ts`）：新增 `config.authorizedUsers: string[]`。owner 永远授权，其余必须在白名单内；未授权者静默丢弃并在日志记录其 `from_user_id`。
- **状态按发送人分键**（`main.ts`）：session / 队列 / 处理标志 / abortController 全部从「按 bot accountId」改为「按 from_user_id」。session 文件名 `<accountId>__<userId>.json`。
- **每人独立工作目录**（`main.ts` `getRuntime`）：首次见到某用户时默认 `<配置根目录>/<用户短ID>` 并自动创建。
- **并发**：不同用户的请求并发执行（各自一个 `claude` 进程），同一用户内部串行。
- **per-user system prompt**（`session.ts` + `handlers.ts`）：`/prompt` 从全局 config 改为写 session（顺带修复原来 `/prompt` 写全局却不被运行时读取的隐藏 bug），回退到全局默认值。
- **`/clear` 改进**：经正常队列处理并传入 currentSession，保留工作目录/模型/提示词（修复原 C5：`/clear` 会重置工作目录）；`/stop` 仍走优先通道即时中止。

**注**：以上 per-user/allowlist 逻辑保留，但实测发现 ilink bot 是 **1:1 设计**（见下「多 bot」一节），第三方无法加入同一个 bot，因此 `authorizedUsers` 白名单在当前部署中实际不会被用到（每个 bot 只有它自己的 owner 在发消息）。代码无害，留作未来可能的同 bot 多用户支持。

---

## 多 bot 改造（2026-06-12，第三批，已端到端验证 ✅）

**实测结论**：ilink bot 无法分享名片、无第三方加好友入口；每次扫绑定码都会创建一个**新的独立 bot**（不同 `accountId`）绑定到扫码人。所以「多员工」的正确做法是 **每个员工各绑一个自己的 bot，由一个 daemon 同时服务多个 bot**。

涉及文件：`src/wechat/accounts.ts`、`src/main.ts`

- **`loadAllAccounts()`**（`accounts.ts`）：加载 `accounts/` 下全部账号（原 `loadLatestAccount` 只取最新一个）。
- **`createBotMonitor(account, …)`**（`main.ts`）：把单 bot 的全部设置（api / sender / 每用户 runtime / 回调）抽成一个函数，每个 bot 一份，闭包隔离，互不串。
- **`runDaemon`**：`loadAllAccounts()` → 每个有效账号建一个 monitor → `Promise.all(monitors.run())` 并发轮询；逐账号 fail-closed（缺 userId 的 bot 跳过）；优雅关停遍历所有 monitor。

**怎么加员工**（已验证流程）：
1. 在宿主机跑 `npm run setup` 生成一张绑定二维码；
2. 把二维码发给员工，员工用**自己手机**微信扫码并确认 → bot 出现在**他的**微信里，凭证存到宿主机 `accounts/<新id>.json`；
3. `npm run daemon -- restart` → daemon 自动加载并服务新增 bot。

**已验证**：编译通过；daemon 同时服务 2 个 bot（`0eae8c4a3592` + `753fbf07f38f`）正常启动轮询；第二个员工独立绑定（不同 userId）；该员工真人发消息并收到 Claude 回复，与 owner 会话互不干扰。

## 已知但**有意未修**的上游正确性 bug

以下为代码 review 发现、但判断为「偶发、不影响安全、改动面大、易与上游冲突」而保留的问题，留作记录：

- resume 重试会重复发送已流式输出的半截文本（`main.ts` retry 路径未重置 buffer）
- CDN 上传只在 HTTP 5xx 重试，网络错误（ECONNRESET）直接失败（`upload.ts`）
- 轮询循环在 API 返回非零 ret 时不退避，可能热转空跑（`monitor.ts`）
- 日志时区写死 UTC+8；`cleanupOldLogs` 在每次写日志的热路径上跑；零测试覆盖
- `config.authorizedUsers` 是全局而非按 bot——但因 ilink 1:1（每 bot 只有自己 owner 发消息），该白名单当前永不生效，留作未来同 bot 多用户支持
- `store.ts` `saveJson` 非原子写（truncate-then-write）；进程被 SIGKILL 中断可能损坏会话文件，但 `loadJson` 有 fallback 兜底
- `/prompt clear` 只清 per-session，不动全局 `config.systemPrompt` 兜底值（多用户下清全局会误伤他人，故有意不改）

---

## 第四批：多 bot 并发 review 修复（2026-06-12，已编译+部分运行验证 ✅）

三路并行 code review（并发/安全/正确性）对多 bot 改动的对抗式审查，修复以下 8 项：

涉及文件：`src/wechat/sync-buf.ts`、`src/wechat/monitor.ts`、`src/main.ts`、`src/claude/provider.ts`、`src/session.ts`、`src/wechat/upload.ts`

1. **【致命】轮询游标按 bot 分离**（`sync-buf.ts` + `monitor.ts`）：原 `get_updates_buf` 存单个全局文件，多个 bot 会互相覆盖游标（游标含 bot id，是 bot 专属的）→ 漏消息/重复。改为 `sync/<accountId>.json` 每 bot 一份，`createMonitor` 增加 `accountId` 参数。**已运行验证**：两个 bot 各自生成独立游标文件、各用各的游标轮询。
2. **会话键兜底**（`main.ts` `sessionKeyFor`）：含异常字符的 `from_user_id` 会让 `validateAccountId` 抛错、导致该用户永久卡死。改为异常 id 走 base64url 编码（正常 id 原样保留）。**已验证**：斜杠/加号/中文/空格等 id 均通过校验不崩。
3. **单 bot 崩溃不拖垮整体**（`main.ts`）：`Promise.all(monitors)` 任一 reject 会让整个 daemon 退出。改为每 bot 独立 `runResilient` 重启循环（崩溃后 10s 重启该 bot，不影响其他）。
4. **`/stop` 干净中止**（`provider.ts` + `main.ts`）：onAbort 是 resolve 而非 throw，导致 sendToClaude 继续走成功路径、发出半截文本并把中止的 sessionId 持久化。`QueryResult` 增加 `aborted` 标志，sendToClaude 检测到即静默 return。
5. **per-user 工作目录只分配一次**（`session.ts` + `main.ts`）：原条件 `=== DEFAULT_WORKING_DIR` 会在每次重启把停在默认目录的用户重新搬到子目录。改为 `workspaceInitialized` 标志，只首次分配。
6. **删除死代码 `sharedCtx`**（`main.ts`）：`lastContextToken` 被每个用户写但从不读——当前无害但是潜在的跨用户污染隐患，整体移除。
7. **上传响应日志脱敏**（`upload.ts`）：原 INFO 打印整个 `uploadResp`，可能含 redactor 触及不到的嵌套/异名密钥字段。改为只打印 `ret`/`hasUploadUrl`/`hasUploadParam` 摘要。
8. **队列异常自愈**（`main.ts` `drainQueue`）：handler 抛错时 finally 补一次 re-drain，避免后续消息卡到下次发消息才被处理。

**review 判定为安全/无需改的**：各 bot 的 `WeChatApi.nextSendTime` 限频、`createSender` 状态、`recentMsgIds` 去重均为每实例隔离；`logger` append 原子；per-user 串行队列保证同一用户不会并发跑两个 `sendToClaude`。

---

## 第五批：final review 修复（2026-06-12，编译+运行验证 ✅）

对第四批 8 个修复本身做对抗式 final review，发现并修复其引入/暴露的 3 个真实问题 + 1 处注释：

1. **【致命】`/stop` 在 resume 重试期间被忽略**（`main.ts` sendToClaude）：`result.aborted` 只在重试**前**检查；若中止发生在重试中，`Object.assign` 合并后未复查 → 仍走成功路径、持久化中止的 sessionId。补：重试合并后再检查一次 `aborted`。
2. **【致命】会话键碰撞**（`main.ts` sessionKeyFor）：base64url 字母表 ⊆ 原 safeId 正则，理论上「原样 id」与「另一 id 的编码」可能相等 → 两用户共享会话。改为两分支各带前缀（`r_` 原样 / `b_` 编码），命名空间按构造互斥，**可证明无碰撞**。（副作用：现有 2 个测试会话键变更，owner/同事各重开一次新对话，工作目录因 shortId 确定性不变。）
3. **【重要】非法 accountId 导致热重启循环**（`main.ts` runDaemon）：accountId 现用作路径组件（游标/会话键），若非法会在轮询中抛错、被 resilient 循环每 10s 无限重启。改为启动时校验 accountId，非法则跳过该 bot。
4. **drainQueue 顺序不变式注释**：标注「先清 `processing` 再查队列」的不变式，防未来重排引入竞态。

**final review 复核为安全的**：clean-stop 路径不会空转（`process.exit` 先于 `Promise.all` settle）；`workspaceInitialized` 经 `/clear` 保留；`sharedCtx` 已无残留引用；`uploadResp.ret` 类型存在；abort 早返回时 finally 仍清理 flushTimer/typing。
