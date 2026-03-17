# Metapi 架构与业务逻辑分析（Golang + React 改造准备）

本文档基于当前仓库源码，细化梳理系统架构、核心业务逻辑，并补充后端 Golang 与前端 React 改造前的准备清单，作为重构与技术评审的基础材料。

## 1. 项目定位与运行形态

Metapi 是“中转站的中转站”：聚合多个 AI 中转站（New API / One API / OneHub / DoneHub / Veloera / AnyRouter / Sub2API 等），对下游提供一个统一入口与统一的 API Key，同时具备模型自动发现、智能路由、自动故障转移、集中监控与运营管理能力。

运行形态包含三部分：
- 服务端：Fastify + TypeScript 的统一代理与管理 API
- 管理后台：Vite + React
- 桌面端：Electron（可选）

数据存储默认 SQLite，支持 MySQL / PostgreSQL 作为运行时数据库。

## 2. 架构分层与模块划分

服务端模块（`src/server`）是核心业务载体：
- 配置与启动：`src/server/config.ts`、`src/server/index.ts`
- 数据访问与迁移：`src/server/db/*`
- 中间件：`src/server/middleware/*`
- 管理 API：`src/server/routes/api/*`
- 代理 API：`src/server/routes/proxy/*`
- 业务服务：`src/server/services/*`
- 协议转换：`src/server/transformers/*`

前端模块（`src/web`）负责管理后台页面、统计图表与配置交互。
桌面模块（`src/desktop`）包装运行时端口与本地资源。

## 3. 数据模型概览

当前数据库表结构（`src/server/db/schema.ts`）可视为核心业务域的“事实源”：
- `sites`：上游站点配置（平台类型、URL、状态、全局权重、代理与自定义请求头）
- `accounts`：站点账号与访问凭据（session / api key）、余额、配额、健康状态
- `account_tokens`：账号下的可用 token 列表（默认 token、分组、来源）
- `model_availability`：账号级模型可用性与延迟
- `token_model_availability`：token 级模型可用性与延迟
- `token_routes`：模型路由规则（pattern、展示名、映射、路由策略）
- `route_channels`：路由通道（账号 + token + 权重 + 优先级 + 成功/失败统计）
- `proxy_logs`：代理请求日志（模型、耗时、Token 计费、错误、重试）
- `proxy_files`：代理文件（/v1/files）
- `proxy_video_tasks`：视频任务追踪（异步生成）
- `downstream_api_keys`：下游托管 API Key（额度、模型范围、路由限制）
- `settings`：运行时配置覆盖（来自管理后台）
- `events`：系统事件流（签到、余额、状态变更、通知）
- `checkin_logs`：签到记录

### 3.1 实体关系与主链路

核心关系可简化为：
- 站点 `sites` 作为上游入口，包含平台类型、URL、全局权重、代理配置
- 账号 `accounts` 归属站点，承载访问凭据、余额与状态
- 账号 Token `account_tokens` 归属账号，支持多 Token 路由
- 路由规则 `token_routes` 与通道 `route_channels` 共同决定模型到上游通道的映射
- 代理日志 `proxy_logs` 记录每次请求的真实模型、耗时、计费与错误
- 下游 Key `downstream_api_keys` 决定鉴权策略、模型范围与配额限制

### 3.2 关键字段与枚举（用于迁移一致性）

| 领域 | 字段 | 枚举/规则 | 说明 |
| --- | --- | --- | --- |
| 站点 | `sites.platform` | `new-api` / `one-api` / `one-hub` / `done-hub` / `veloera` / `anyrouter` / `sub2api` / `openai` / `claude` / `gemini` | 平台适配器选择 |
| 站点 | `sites.status` | `active` / `disabled` | 站点禁用会级联账号 |
| 账号 | `accounts.status` | `active` / `disabled` / `expired` | 影响路由与健康状态 |
| Token | `account_tokens.source` | `manual` / `sync` / `legacy` | Token 来源 |
| 路由 | `token_routes.routing_strategy` | `weighted` / `round_robin` | 路由策略 |
| 事件 | `events.type` | `checkin` / `balance` / `token` / `proxy` / `status` | 用于告警与时间线 |
| 代理 | `proxy_logs.status` | `success` / `failed` / `retried` | 代理状态 |
| 下游 Key | `downstream_api_keys.supported_models` | 通配符 / `re:` 正则 | 模型允许列表 |

### 3.3 数据约束与级联

关键约束用于保证迁移后数据行为一致：
- `accounts.site_id` 级联删除
- `account_tokens.account_id` 级联删除
- `route_channels.route_id` 级联删除
- `route_channels.account_id` 级联删除
- `route_channels.token_id` 删除时置空（保留通道记录）
- `proxy_files` 为软删除（`deleted_at`）

## 4. 核心业务逻辑梳理

### 4.1 站点与账号管理

站点管理流程：
1. 站点创建时会自动检测平台类型（`src/server/services/siteDetector.ts`）
2. URL 统一规范化，去除末尾 `/`，用于匹配代理与路由
3. 站点禁用会级联禁用其下账号，并记录事件（`src/server/routes/api/sites.ts`）
4. 支持系统代理与站点自定义 Header（`src/server/services/siteProxy.ts`）

账号管理流程：
1. 支持 session 登录与 API Key 模式（`src/server/routes/api/accounts.ts`）
2. 自动重新登录与 Token 续期（Sub2API 专有逻辑）
3. 运行时健康状态写入 `accounts.extraConfig`（`src/server/services/accountHealthService.ts`）
4. 账号验证与诊断有超时保护，避免阻塞后台任务

### 4.2 模型发现与路由重建

模型发现：
1. 通过平台适配器拉取模型列表（`src/server/services/platforms/*`）
2. 模型探测写入 `model_availability` / `token_model_availability`
3. 支持 API Token 自动发现与补齐（`src/server/services/modelService.ts`）
4. 探测支持超时控制与失败分类（timeout/unauthorized）

路由重建：
1. 基于可用模型自动生成 `token_routes` 与 `route_channels`
2. 对手工通道保留 `manualOverride`，避免被自动重建覆盖
3. 变更后清理路由决策快照并刷新缓存
4. 账号级 API Key 模式会优先用账号凭据构建通道

### 4.3 智能路由算法

路由入口：
1. 模型请求先匹配 `token_routes`（支持通配符与 `re:` 正则）
2. 路由策略支持 `weighted`（默认）与 `round_robin`
3. 路由结果缓存，降低 DB 读取频率（路由与通道均有 TTL）

加权策略（`src/server/services/tokenRouter.ts`）：
1. 评分因子：成本、余额、使用量
2. 成本来源优先级：实测成本 > 配置成本 > 价格目录 > 兜底成本
3. 站点级全局权重与下游 Key 的站点倍率共同作用
4. 同站点多个通道会进行分摊，避免单站点因通道多而被过度选中

失败与冷却策略：
1. `weighted` 使用 Fibonacci 退避
2. `round_robin` 失败达到阈值后进入分级冷却
3. 最近失败通道会被优先避让

### 4.4 代理请求链路

核心代理入口（`src/server/routes/proxy/*`）：
- 支持 `/v1/chat/completions`、`/v1/messages`、`/v1/responses`、`/v1/embeddings`、`/v1/images`、`/v1/files`、`/v1/videos`、`/v1/models`
- 统一鉴权支持 `Authorization`、`x-api-key`、`x-goog-api-key`、`?key=`（`proxyAuthMiddleware`）
1. 请求体由 `transformers` 进行 OpenAI / Claude / Responses 互转
2. 上游 endpoint 具备降级与恢复逻辑（`endpointFlow.ts`）
3. 支持 SSE 流式转发并解析 usage，若 upstream 缺失 usage 会回退到日志与估算逻辑
4. 请求头会过滤 hop-by-hop 与敏感 header，并按协议族转发特定 header

日志与计费：
1. 每次请求写入 `proxy_logs`，记录实际模型与估算成本
2. 计费逻辑基于模型目录价格或通道实测成本（`modelPricingService.ts`）
3. 下游托管 Key 会记录成本与请求量限制

### 4.5 签到与余额刷新

定时任务（`checkinScheduler.ts`）：
- 签到任务、余额刷新、日报、日志清理
- Cron 表达式可由 `settings` 覆盖

签到服务（`checkinService.ts`）：
- 自动处理“已签到”与“站点不支持签到”场景
- Cloudflare / Turnstile 识别与降级提示

余额服务（`balanceService.ts`）：
- 支持自动重登与 Sub2API token 刷新
- 记录今日收益与余额变化

### 4.6 通知与告警

通知渠道：
- webhook / bark / serverchan / telegram / SMTP
- 统一节流与失败保护（`notifyService.ts`、`notificationThrottle.ts`）

告警触发：
- Token 过期、站点不可用、代理失败
- 写入 `events`，可在前端统一查看

### 4.7 运维与数据管理

数据库与迁移：
- 支持 SQLite / MySQL / Postgres 运行时迁移（`databaseMigrationService.ts`）
- 运行时会从 `settings` 覆盖 DB 配置（`src/server/index.ts`）

备份与恢复：
- 结构化导出为 JSON（`backupService.ts`）
- 可选择“仅账号”或“仅偏好”导出

日志清理：
- 支持两类日志清理策略
- 兼容旧版 proxy log retention 定时清理

### 4.8 文件与视频代理

文件代理：
1. `/v1/files` 的输入文件会写入 `proxy_files`（Base64 存储）
2. 代理时会根据下游 Key 识别资源归属与权限

视频代理：
1. `/v1/videos` 的异步任务写入 `proxy_video_tasks`
2. 代理层负责轮询上游并同步状态

### 4.9 统计与监控

统计入口主要集中于 `routes/api/stats.ts`：
- 模型市场数据带缓存（短 TTL），避免频繁拉取上游价格
- 代理日志支持筛选、分页与时间窗口统计
- 可用性监控按时间桶汇总成功率与延迟

### 4.10 缓存与运行时状态

当前关键缓存点：
- 路由缓存与路由命中缓存（`tokenRouter`）
- 模型市场缓存（`stats`）
- 价格目录缓存（`modelPricingService`）
- 站点代理配置缓存（`siteProxy`）

迁移时需要保证 TTL 与刷新逻辑一致，避免路由波动或价格抖动。

## 5. 接口面与关键文件索引

接口面概览（非完整清单，用于迁移边界确认）：
- 管理端认证：`/api/settings/auth/*`
- 站点管理：`/api/sites`
- 账号管理：`/api/accounts`
- Token 管理：`/api/account-tokens`、`/api/tokens`
- 路由与模型：`/api/tokens/*`、`/api/stats/*`
- 系统设置：`/api/settings`
- 监控与事件：`/api/monitor`、`/api/events`
- 代理入口：`/v1/*`

| 领域 | 关键文件 |
| --- | --- |
| 启动与配置 | `src/server/index.ts`、`src/server/config.ts` |
| 数据模型 | `src/server/db/schema.ts` |
| 代理入口 | `src/server/routes/proxy/*` |
| 路由算法 | `src/server/services/tokenRouter.ts` |
| 模型发现 | `src/server/services/modelService.ts` |
| 价格与计费 | `src/server/services/modelPricingService.ts` |
| 站点代理 | `src/server/services/siteProxy.ts` |
| 签到/余额 | `src/server/services/checkinService.ts`、`src/server/services/balanceService.ts` |
| 通知告警 | `src/server/services/notifyService.ts`、`src/server/services/alertService.ts` |
| 备份迁移 | `src/server/services/backupService.ts`、`src/server/services/databaseMigrationService.ts` |
| 统一鉴权 | `src/server/middleware/auth.ts` |

## 6. 后端 Golang 改造前准备清单（细化）

### 6.1 迁移范围与兼容性冻结

需要明确以下边界：
- 是否保留 Electron 桌面端
- 管理后台是否继续使用当前 React 前端
- 是否继续支持多数据库运行时切换
- 是否继续支持 OpenAI / Claude / Gemini 三种协议
- 是否保持与现有 `/v1/*` 行为完全一致（包含错误码与降级策略）

### 6.2 行为契约与测试基线

建议建立“行为契约”来保护迁移结果：
- 管理 API 的请求/响应结构快照
- `/v1/*` 代理接口的返回格式、错误码与 SSE 流式规范
- 路由选择一致性（同配置下概率分布、冷却策略）
- 计费与 usage 解析一致性
- 代理失败时的重试次数与失败提示一致性

### 6.3 Go 侧模块边界与包划分

建议 Go 侧分层与当前结构对齐：
- `internal/config`：环境变量与运行时配置
- `internal/db`：SQL schema 与迁移
- `internal/handlers`：API 与 proxy handler
- `internal/services`：路由、模型、计费、通知
- `internal/platforms`：上游适配器
- `internal/transformers`：OpenAI / Claude / Responses 转换

### 6.4 协议转换与流式处理基线

重点挑战点：
- SSE 代理必须兼容 OpenAI 与 Claude 的差异
- `responses` 与 `chat/completions` 的相互降级策略
- 需要保留 `transformers` 中的兼容逻辑与解析顺序
- 代理 header 过滤规则必须一致（hop-by-hop + 敏感 header）

### 6.5 路由算法规格化

必须保持以下行为一致：
- 模型匹配规则（通配符、正则、显示名）
- 负载因子（成本、余额、使用率）
- 失败退避与冷却
- 站点权重与下游 Key 权重的合并方式
- 随机选择算法与概率计算方式

### 6.6 数据库与迁移准备

推荐方案：
- 直接复用当前 schema（字段名、枚举值）
- JSON 字段仍用 TEXT 存储，避免数据库差异
- 迁移过程中保留导入导出工具，避免数据丢失
- 明确 `settings` 表的运行时覆盖优先级

### 6.7 运维与运行时配置

Go 侧需要覆盖：
- cron 调度任务
- 运行时配置覆盖（settings 表）
- 备份 / 恢复 / 迁移能力
- 站点代理配置缓存与系统代理逻辑
- 价格目录缓存与刷新 TTL

## 7. 前端 React 改造前准备清单（细化）

### 7.1 接口契约与类型准备

建议在改造前输出以下基线：
- 管理端 API 的 OpenAPI 或 JSON Schema
- 统一的错误码与错误结构规范
- 前端 API Client 的类型生成方案（例如 OpenAPI -> TS 类型）

### 7.2 页面与数据依赖图

建议先整理页面到接口的依赖图，避免改造过程中遗漏：
- Dashboard、Sites、Accounts、Tokens、Token Routes、Models、Proxy Logs、Settings
- 每个页面对应的 API 列表与字段使用清单

### 7.3 登录态与权限模型

必须明确：
- 管理后台登录 Token 的保存位置与刷新策略
- IP allowlist 与权限提示在前端的呈现方式
- 下游 Key 管理页面的权限控制逻辑

### 7.4 交互与性能基线

需要提前确定：
- 大列表分页与虚拟滚动策略（Proxy Logs、Accounts、Tokens）
- 图表刷新与缓存策略（Dashboard、Model Marketplace）
- SSE 流式日志或调试面板的性能约束

### 7.5 测试与回归基线

建议在改造前冻结：
- 关键页面的可视化回归点
- 操作路径的 E2E 基线（创建站点、绑定账号、刷新模型、路由调试）
- API Mock 与前端离线开发能力

## 8. 迁移方案对比

方案 A：Go 单体重写
- 优点：技术栈统一，性能可控
- 缺点：周期长，风险集中
- 适用：团队 Go 能力强、需要长期演进

方案 B：代理层先迁移，管理端保留 Node
- 优点：先保障核心吞吐与稳定性
- 缺点：双栈维护成本
- 适用：优先优化代理性能与稳定性

方案 C：分阶段服务化（路由服务、代理服务、运维服务）
- 优点：可逐步迁移，风险可分摊
- 缺点：架构复杂度增加
- 适用：大团队、需要横向扩展

推荐策略：
优先使用方案 B，先迁移高负载的代理链路，再逐步迁移管理 API 与后台任务。

## 9. 迁移里程碑建议

阶段 0：梳理接口与现有行为基线
- 输出 API 行为契约
- 生成路由算法一致性测试

## 10. 文档评审与修复记录

| 问题 | 影响 | 修复动作 | 参考文件 |
| --- | --- | --- | --- |
| OpenAPI 多处响应结构过于宽泛（`additionalProperties: true`） | 迁移时难以生成 Go/TS 类型，接口边界不清晰 | 引入明确响应组件，统一替换为可复用 schema | `docs/golang-rewrite/openapi-draft.yaml` |
| 下游 Key 汇总/趋势/批量接口未入 OpenAPI | 迁移时易漏实现关键运维功能 | 补充路径与响应 schema | `docs/golang-rewrite/openapi-draft.yaml` |
| 路由、通道与测试工具接口在契约中与实际返回不一致 | 迁移阶段可能出现兼容性回退 | 对齐返回结构并标注 200/202 分支 | `docs/golang-rewrite/api-contract-draft.md` |
| 运行时配置存在脱敏字段但文档未说明 | 前端改造时可能误以为可以回显明文 | 增加脱敏字段说明与写入规则 | `docs/golang-rewrite/api-contract-draft.md` |
| 搜索接口返回 token 可能脱敏且带 `valueStatus` | UI 误判可用性与复制权限 | 补充 SearchResponse 与状态说明 | `docs/golang-rewrite/api-contract-draft.md` |

阶段 1：Go 代理链路落地
- `/v1/*` 代理入口
- 复刻 tokenRouter 与 pricing 逻辑

阶段 2：Go 管理 API
- Sites / Accounts / Tokens / Settings
- 迁移后台统计与监控

阶段 3：迁移运维与任务
- cron、日志清理、通知、备份

阶段 4：停用 Node 服务端

## 10. 待确认问题

以下问题需要在正式迁移前明确：
- 多租户隔离与权限模型是否需要增强
- 是否需要提供 OpenAPI 或 gRPC 作为新接口层
- 是否需要引入更强的监控体系（Prometheus / OpenTelemetry）
- 前端是否需要分层重构（组件库、状态管理、API client）
