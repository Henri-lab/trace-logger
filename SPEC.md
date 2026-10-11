# trace-logger 规范 v0.1（草案）

一句话目标：**让任何一次用户操作，都能在 3 分钟内被还原**——靠的不是日志条数，而是每条日志都带足够的上下文。

## 1. 字段字典

### 1.1 必需字段（每条日志）

| 字段 | 含义 | 谁注入 |
| --- | --- | --- |
| `time` | 带时区的时间戳 | 日志库 |
| `level` | `debug` / `info` / `warn` / `error` | 日志库 |
| `msg` | 短名词短语（`order.created`），不要拼长句 | 调用方 |
| `trace_id` | 一次用户操作的链路 ID | 中间件 / 前端 |
| `src_module` | 逻辑模块名（`payment`、`httpx`） | 自动推导（从调用点文件路径） |
| `func` | 函数名（`payment/http.(*Handler).writeError`） | 自动推导 |
| `caller` | 模块内相对路径:行号 | 自动推导 |

自动推导的三个字段**禁止手写**：手写必然漏、必然在重构后变成假信息。

### 1.2 推荐字段

| 字段 | 用途 |
| --- | --- |
| `request_id` | 单次 HTTP 请求 ID（无 trace 时兜底） |
| `user_id` | **操作者**（发起请求的登录用户），登录后自动注入 |
| `subject_user_id` | 被操作对象（当它不等于操作者时使用，例如管理员为用户发权益） |
| `duration_ms` | 耗时（外部调用 / 数据库 / 整体请求） |
| `status` / `code` | HTTP 状态码 / 业务错误码（两者都记才能聚合同类错误） |
| `outcome` | 结果枚举（`matched` / `unmatched` / `retry`） |
| 业务标识 | `order_no`、`payment_id`、`product_id`、`entitlement_id` 等 |

### 1.3 命名约定

- 一律 `snake_case`；
- **单位写进名字**：`amount_cents`、`duration_ms`、`size_bytes`；
- 同义字段只留一个（不要同时出现 `uid` / `userId`）；
- 枚举值用小写英文；面向人的中文只在展示层出现。

### 1.4 红线（任何级别都不许写）

- 密码、验证码、token、API key、私钥；
- 身份证号、银行卡号、完整手机号/邮箱（必要时截断或哈希）；
- 完整请求体/响应体（改打关键字段 + 长度）；
- 带签名的下载地址、任何可被复用的凭证串。

## 2. 级别语义

| 级别 | 语义 | 典型场景 | 默认去向 |
| --- | --- | --- | --- |
| `debug` | 开发细节 | 分支、循环、SQL 参数 | 本地控制台，可采样 |
| `info` | **业务决策点** | 下单、支付成功、发权益、审核通过 | 保留，不告警 |
| `warn` | 可自愈但要知道 | 重试、降级、命中限流、可疑输入、未匹配到账 | 保留，看板关注 |
| `error` | 需要人处理 | 请求失败、任务失败、数据不一致 | 告警 |

判定法：**"这条日志是否代表状态发生了变化？"** 是 → `info`；否 → `debug`。

## 3. 链路（trace）

### 3.1 生成与传递

```
浏览器                    服务端 A                    服务端 B（可选）
  | 生成 trace id                                        |
  | X-Trace-Id: <id> ---> 读取（无则由请求 ID 兜底）        |
  |                       放进请求上下文 -> 所有日志都带     |
  |<-- 响应头回显 X-Trace-Id --|                          |
  |                       跨服务调用继续带同一个头 -------> 沿用同一 id
```

- **过渡期**：自定义头 `X-Trace-Id`（当前实现）；
- **长期**：改用 W3C `traceparent`（`version-traceid-spanid-flags`）以便与 OTel 生态互通；
  迁移期可双写：读时优先 `traceparent`，写时两个头都回显；
- ID 规则：长度 ≤ 64，仅允许 `[A-Za-z0-9._:-]`；非法值一律丢弃并重新生成（防注入与脏数据）。

### 3.2 前端日志回流

```
POST /api/v1/logs
{ "entries": [ { "level": "error", "module": "checkout", "message": "提交凭证失败",
                 "trace_id": "...", "url": "/checkout/8f2c", "attrs": { "status": "500" } } ] }
-> 202 { "accepted": 1 }
```

服务端必须做到的六件事：

1. 请求体上限（如 32KB）+ 单批条数上限（如 20）；
2. 字段白名单 + key 清洗 + 值截断；
3. 级别白名单（未知级别丢弃，不要 fallback 成 `info`）；
4. 限流（如每 IP 每分钟 60 次），超限 429；
5. 写进**同一条日志流**（打 `origin=web`、`web_module`、`web_trace_id`），不另建存储；
6. 内部失败可记录，但**不因日志问题影响业务**。

前端侧：只回流 `warn` / `error`；批量（攒 N 条或每 T 秒）；`visibilitychange` / `pagehide` 时用
`keepalive` 补发；上报请求**不走统一请求封装**（避免递归）；失败即丢、不重试。

## 4. 自动注入的实现要求

- 包装日志 handler，在 `Handle` 里用调用点 PC 推导 `src_module` / `func` / `caller`；
- 上下文 logger 采用**三档 fallback**：请求级 → 服务自己的 → 全局默认（避免后台任务掉进全局单例）；
- 登录后把 `user_id` 补进上下文 logger（该请求后续所有日志自动携带）；
- 输出形态：开发环境人眼友好、生产环境机器可解析（JSON）。

参考实现：`Henri-lab/course-platform` 的 `internal/platform/logger` 与 `internal/platform/httpx`。

## 5. 业务埋点

原则：**只打状态发生变化的时刻**，一条日志对应一次"事实"。

命名建议 `<域>.<事件>`：

```
order.created      order.cancelled     payment.intent_created
payment.matched    payment.unmatched   claim.submitted
claim.approved     claim.rejected      entitlement.grant
entitlement.renew  outbox.processed    job.reminder_sent
```

必打清单（按域）：交易/订单（下单、取消、创建支付单、到账匹配、凭证提交与审核、退款）；
权益（发放、续期、回收、判权被拒记 `warn`）；外部依赖与后台任务（外部调用、事件消费、定时任务）；
认证（登录成功/失败、权限校验失败）。

**不打的**：每个函数入口/出口、循环体内逐条、每次读缓存或查库、只为"看着热闹"的记录。

失败路径由边界层（HTTP / 任务框架）统一记录，业务层只记状态跃迁，避免同一次失败被记两遍。

## 6. 保留、告警与升级

- 保留：stdout 由容器运行时轮转（如 50MB × 5）；需要长留存与检索时再上集中采集；
- 告警只挑"需要人立刻动手"的信号：`error` 速率、关键事件消失（有支付但长时间无
  `entitlement.grant`）、`payment.unmatched` 占比、验签失败（一条都告警）；
- 升级路径：L1（stdout + grep/jq）→ L2（Loki/ELK + 保留策略）→ L3（OpenTelemetry 全链路）。
  升级只换采集与展示层——**前提是字段与 trace id 从一开始就是标准的**。

## 7. 验收清单（可测）

- [ ] 任取一次用户操作，用它的 `trace_id` 能还原完整时间线（跨前后端）；
- [ ] 每条日志都有 `src_module` / `func` / `caller`，且重构后不会失效（自动推导）；
- [ ] 丢弃日志或关掉回流端点，业务功能不受影响；
- [ ] 抽查 50 条日志，无 §1.4 红线内容；
- [ ] 每行 `user_id` 只出现一次；跨用户操作使用 `subject_user_id`；
- [ ] 能回答一个聚合问题（如"昨天支付失败多少笔、集中在哪个模块"）；
- [ ] 日志包含失败路径，不只有成功路径。

## 8. 版本与兼容

- v0.1 为草案；**破坏性变更**（字段改名、语义变更）必须写进变更记录并同步所有消费方；
- 字段**只增不改**：需要澄清语义时新增字段，旧字段保留到所有消费方迁移完成；
- 迁移到 W3C `traceparent` 时升为 v1.0，并保留一个过渡期双写。

### 变更记录

| 版本 | 日期 | 变更 |
| --- | --- | --- |
| v0.1 | 2026-10-11 | 初稿：字段字典、级别语义、链路规则、回流契约、业务埋点、验收清单 |
