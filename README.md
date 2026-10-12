# trace-logger

跨项目的**日志与链路追踪约定**（spec-first），以及（未来可能有的）一个薄参考实现。

## 现在是什么状态

**只有规范，没有运行时库。** 这不是没写完，而是刻意的决定，理由见 `docs/adr/0001-spec-first.md`：

1. 目前只有一个消费方（course-platform），"复用"还没被证明；
2. 真正值得沉淀的是**约定**（字段名、级别语义、事件命名），不是代码——约定在只有一个使用方时最容易定错；
3. "成熟的做法"已经有答案：链路传播用 **W3C Trace Context + OpenTelemetry**，前端错误用
   Sentry / OTel Browser 一类 RUM。自研库一旦被依赖，就会挡住换回标准的退路。

## 这个仓库负责什么

- 规定一条日志**必须有什么字段、字段叫什么、什么级别用在什么场景** → `SPEC.md`
- 规定 trace id 怎么产生、怎么跨进程传、前端日志怎么回流 → `SPEC.md` §3
- 规定验收方式（能被机器检查的那几条） → `SPEC.md` §7
- 记录"什么时候从约定升级为代码 / 迁移到 OTel" → `docs/adr/0001-spec-first.md`

## 这个仓库不负责什么

- **不重新发明链路传播**：用 W3C `traceparent`，过渡期可用自定义 `X-Trace-Id`；
- **不做采集/存储/查询**：那是 Loki / ELK / 云日志的活；
- **不做 APM**：那是 OTel Collector / Jaeger / 商业 APM 的活。

## 与成熟方案的关系（本仓库的增量在哪）

| 你要的东西 | 成熟方案 | 本仓库的增量 |
| --- | --- | --- |
| 结构化日志 | Go `log/slog`、zap、zerolog；JS pino | 字段与级别的**统一约定**，避免每个项目各起一套名字 |
| 调用点定位（文件/函数/行） | zap `AddCaller`；slog `AddSource` | **逻辑模块名**推导（`payment` / `httpx`）+ 统一输出形态 |
| 跨服务链路 | OpenTelemetry（`traceparent` + OTLP） | 过渡期的轻量做法 + "何时必须换 OTel"的判据 |
| 前端错误与追踪 | **Sentry / GlitchTip / OTel Browser**（**首选**，见 SPEC §6.1） | 不引第三方时的**降级**回流契约（限流 + 失败即丢） |
| 业务可观测性 | — | 关键业务事件命名（`order.created` / `payment.unmatched`）与 actor/subject 规则 |

一句话：**机制用成熟方案，约定由本仓库定义**。

## 什么时候会有代码

满足任一条件才动手（ADR 0001 里写死了触发条件）：

1. 第二个项目真的接入（复用被证明）；
2. 出现第二个语言/运行时（例如某个服务用 Java / Python）；
3. 需要跨服务链路——**那时优先接 OTel，而不是扩写本仓库**。

代码形态（如果做）：只包含 `enrich`（module/func/caller 推导）、`logctx`（上下文 logger + fallback 链）、
`trace`（生成/回显/兼容 W3C）。**不做**传输、导出、采样。

## 现在怎么用

1. 要落地日志体系：读 `SPEC.md`，按 §7 验收清单逐条验证；
2. 要参考实现（Go + Next.js）：看 `Henri-lab/course-platform` 的
   `apps/api/internal/platform/logger`、`apps/api/internal/platform/httpx`、`apps/web/lib/logger.ts`
   与 `POST /api/v1/logs`；
3. 要从零生成一套：用 Codex skill `logging-system-builder`。

## 状态

- v0.2（草案）：字段字典、级别语义、链路规则、验收清单已定稿；新增"错误上报交给成熟产品"的分工（Sentry/GlitchTip）；
  运行时代码仍待触发条件满足（ADR 0001）。
