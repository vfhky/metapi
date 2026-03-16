# Metapi API 行为契约草案（管理端 + 代理端）

本文档用于迁移前冻结接口行为与错误语义，属于“草案级”契约说明。若与代码存在差异，以当前服务端实现为准。

## 1. 范围与状态

- 覆盖范围：`/api/*` 管理端、`/v1/*` 代理端、`/gemini/*` 兼容端
- 状态：Draft（用于迁移前基线对齐）
- 时间字段：均为 ISO 8601 字符串

## 2. 鉴权与安全

管理端（`/api/*`）：
- Header：`Authorization: Bearer <AUTH_TOKEN>`
- IP 白名单：`ADMIN_IP_ALLOWLIST`（命中失败返回 403）

代理端（`/v1/*`、`/gemini/*`）：
- Token 获取顺序：`Authorization` Bearer → `x-api-key` → `x-goog-api-key` → `?key=`
- Token 类型：全局 `proxy_token` 或托管下游 Key（`downstream_api_keys`）

鉴权失败常见响应：
- 401：缺少 token
- 403：token 无效、禁用、过期、超额（费用或请求数）

## 3. 通用约定

- 请求/响应默认 JSON，流式接口为 SSE
- 错误响应并未统一封装，常见形态：
  - `{ error: "..." }`
  - `{ error: { message: "...", type: "..." } }`
  - `{ success: false, message: "..." }`
- 管理端接口返回 401/403 时，前端会清空登录态并刷新页面

## 4. 核心响应字段字典

该部分用于在迁移前冻结关键资源的字段结构，具体字段取值仍以服务端逻辑为准。

### 4.1 Site

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | 站点 ID |
| `name` | string | 站点名称 |
| `url` | string | 站点入口 URL |
| `platform` | string | 平台类型 |
| `status` | string | `active` / `disabled` |
| `useSystemProxy` | boolean | 是否使用系统代理 |
| `customHeaders` | string? | 自定义请求头（JSON 字符串） |
| `externalCheckinUrl` | string? | 外部签到地址 |
| `isPinned` | boolean | 是否置顶 |
| `sortOrder` | number | 排序 |
| `globalWeight` | number | 站点全局权重 |
| `totalBalance` | number? | 站点总余额（仅列表接口） |

### 4.2 Account

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | 账号 ID |
| `siteId` | number | 所属站点 |
| `username` | string? | 账号名 |
| `accessToken` | string? | Session Token |
| `apiToken` | string? | API Key |
| `balance` | number? | 余额 |
| `balanceUsed` | number? | 已使用 |
| `quota` | number? | 配额 |
| `unitCost` | number? | 账号单价（用于路由） |
| `status` | string | `active` / `disabled` / `expired` |
| `checkinEnabled` | boolean | 是否启用签到 |
| `extraConfig` | string? | 扩展配置（JSON 字符串） |

扩展字段（列表接口）：
| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `site` | object | 站点对象 |
| `credentialMode` | string | `session` / `apikey` / `auto` |
| `capabilities` | object | `{ canCheckin, canRefreshBalance, proxyOnly }` |
| `todaySpend` | number | 今日消耗 |
| `todayReward` | number | 今日签到奖励估算 |
| `runtimeHealth` | object | `{ state, reason, source, checkedAt }` |

### 4.3 AccountToken

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | Token ID |
| `accountId` | number | 账号 ID |
| `name` | string | Token 名称 |
| `token` | string | Token 明文（部分接口脱敏） |
| `tokenGroup` | string? | 分组 |
| `source` | string | `manual` / `sync` / `legacy` |
| `enabled` | boolean | 是否启用 |
| `isDefault` | boolean | 是否默认 |

### 4.4 TokenRoute / RouteChannel

TokenRoute：
| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | 路由 ID |
| `modelPattern` | string | 模型匹配表达式 |
| `displayName` | string? | 展示名 |
| `displayIcon` | string? | 图标 |
| `modelMapping` | string? | 模型映射（JSON） |
| `routingStrategy` | string | `weighted` / `round_robin` |
| `decisionSnapshot` | object? | 决策快照 |
| `decisionRefreshedAt` | string? | 决策刷新时间 |

RouteChannel：
| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | 通道 ID |
| `routeId` | number | 路由 ID |
| `accountId` | number | 账号 ID |
| `tokenId` | number? | Token ID |
| `sourceModel` | string? | 来源模型 |
| `priority` | number | 优先级 |
| `weight` | number | 权重 |
| `enabled` | boolean | 是否启用 |
| `manualOverride` | boolean | 是否手工通道 |
| `successCount` | number | 成功数 |
| `failCount` | number | 失败数 |
| `cooldownUntil` | string? | 冷却截止 |

### 4.5 ProxyLog

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | 日志 ID |
| `routeId` | number? | 路由 ID |
| `channelId` | number? | 通道 ID |
| `accountId` | number? | 账号 ID |
| `modelRequested` | string | 请求模型 |
| `modelActual` | string | 实际模型 |
| `status` | string | `success` / `failed` / `retried` |
| `latencyMs` | number | 耗时 |
| `promptTokens` | number? | 输入 token |
| `completionTokens` | number? | 输出 token |
| `estimatedCost` | number? | 估算成本 |
| `billingDetails` | object? | 计费明细 |
| `errorMessage` | string? | 错误信息 |

### 4.6 DownstreamApiKey

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | Key ID |
| `name` | string | 名称 |
| `key` | string | 明文（仅创建/更新时） |
| `keyMasked` | string | 脱敏 |
| `enabled` | boolean | 是否启用 |
| `expiresAt` | string? | 过期时间 |
| `maxCost` | number? | 费用上限 |
| `usedCost` | number | 已用费用 |
| `maxRequests` | number? | 请求上限 |
| `usedRequests` | number | 已用请求 |
| `supportedModels` | string[] | 允许模型 |
| `allowedRouteIds` | number[] | 允许路由 |
| `siteWeightMultipliers` | object | 站点倍率 |

### 4.7 Event / Task

Event：
| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | 事件 ID |
| `type` | string | 事件类型 |
| `title` | string | 标题 |
| `message` | string? | 描述 |
| `level` | string | `info` / `warning` / `error` |
| `read` | boolean | 是否已读 |

Task：
| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 任务 ID |
| `type` | string | 类型 |
| `status` | string | `pending` / `running` / `succeeded` / `failed` |
| `message` | string | 进度描述 |
| `result` | object? | 任务结果 |

## 5. 管理端 API 契约（/api/*）

### 5.1 站点

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/sites` | 无 | 站点列表，包含 `totalBalance` | 含全部站点 |
| POST | `/api/sites` | `name`、`url`、`platform?`、`useSystemProxy?`、`customHeaders?`、`externalCheckinUrl?`、`status?`、`isPinned?`、`sortOrder?`、`globalWeight?` | 站点对象 | `platform` 为空会自动识别 |
| PUT | `/api/sites/:id` | 同 POST 的可选字段 | 更新后的站点对象 | `status` 变更会级联账号 |
| DELETE | `/api/sites/:id` | 无 | `{ success: true }` | 删除会级联账号 |
| POST | `/api/sites/batch` | `{ ids, action }` | `{ success, successIds, failedItems }` | `action`: `enable` `disable` `delete` `enableSystemProxy` `disableSystemProxy` |
| POST | `/api/sites/detect` | `{ url }` | `{ platform, title? }` 或 `{ error }` | 平台探测 |

### 5.2 账号

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/accounts` | 无 | 账号列表，含 `site`、`capabilities`、`todaySpend`、`todayReward`、`runtimeHealth` | 汇总今日消耗与签到奖励 |
| POST | `/api/accounts/login` | `{ siteId, username, password }` | `{ success, account, apiTokenFound, tokenCount, reusedAccount }` 或失败信息 | 账号密码登录 |
| POST | `/api/accounts/verify-token` | `{ siteId, accessToken, platformUserId?, credentialMode? }` | `{ success, tokenType, modelCount?, models?, userInfo?, balance?, apiToken? }` 或失败标记 | 可能返回 `needsUserId` `invalidUserId` `shieldBlocked` |
| POST | `/api/accounts/:id/rebind-session` | `{ accessToken, platformUserId?, refreshToken?, tokenExpiresAt? }` | `{ success, account? }` 或错误 | 重新绑定 Session Token |
| POST | `/api/accounts` | `{ siteId, username?, accessToken, apiToken?, platformUserId?, checkinEnabled?, credentialMode?, refreshToken?, tokenExpiresAt? }` | `{ ...account, tokenType, credentialMode, capabilities, modelCount, apiTokenFound, usernameDetected }` | 新建账号 |
| PUT | `/api/accounts/:id` | `username?` `accessToken?` `apiToken?` `status?` `checkinEnabled?` `unitCost?` `extraConfig?` `isPinned?` `sortOrder?` `refreshToken?` `tokenExpiresAt?` | 更新后的账号对象 | Sub2API 可更新 refreshToken |
| DELETE | `/api/accounts/:id` | 无 | `{ success: true }` | 删除会触发路由重建 |
| POST | `/api/accounts/batch` | `{ ids, action }` | `{ success, successIds, failedItems }` | `action`: `enable` `disable` `delete` `refreshBalance` |
| POST | `/api/accounts/health/refresh` | `{ accountId?, wait? }` | `wait=true` 返回结果，`wait=false` 返回 202 任务信息 | 后台任务 |
| POST | `/api/accounts/:id/balance` | 无 | 余额刷新结果 | 失败时 404 |

### 5.3 账号 Token

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/account-tokens` | `?accountId=` | Token 列表 | 可按账号过滤 |
| POST | `/api/account-tokens` | `{ accountId, name?, token, group?, enabled?, isDefault? }` | `{ success, token }` | API Key 连接不支持 |
| PUT | `/api/account-tokens/:id` | `{ name?, token?, group?, enabled?, isDefault?, source? }` | `{ success, token }` | 可能触发默认 Token 修复 |
| DELETE | `/api/account-tokens/:id` | 无 | `{ success: true }` | 若支持会同步删除上游 |
| POST | `/api/account-tokens/batch` | `{ ids, action }` | `{ success, successIds, failedItems }` | `action`: `enable` `disable` `delete` |
| POST | `/api/account-tokens/:id/default` | 无 | `{ success: true }` | 设置默认 Token |
| GET | `/api/account-tokens/:id/value` | 无 | `{ success, id, name, token, tokenMasked }` | 脱敏 token 会返回 409 |
| GET | `/api/account-tokens/groups/:accountId` | 无 | `{ success, groups }` | 分组列表 |
| POST | `/api/account-tokens/sync/:accountId` | 无 | `{ success, ... }` | 同步上游 Token |
| POST | `/api/account-tokens/sync-all` | `{ wait? }` | `{ success, queued? }` | 支持后台任务 |
| GET | `/api/account-tokens/account/:accountId/default` | 无 | 默认 Token | 用于 UI |

### 5.4 签到

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| POST | `/api/checkin/trigger` | 无 | `{ success, ... }` | 触发全部账号签到 |
| POST | `/api/checkin/trigger/:id` | 无 | `{ success, ... }` | 单账号签到 |
| GET | `/api/checkin/logs` | `limit?` `offset?` `accountId?` | `CheckinLogRow[]` | 直接返回数组（含 join 字段） |
| PUT | `/api/checkin/schedule` | `{ cron }` | `{ success }` | 更新签到 Cron |

### 5.5 路由与通道

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/routes` | 无 | 路由列表 | 含通道与统计信息 |
| GET | `/api/routes/lite` | 无 | 路由简版列表 | 用于设置页 |
| GET | `/api/routes/summary` | 无 | 路由摘要 | UI 预览 |
| GET | `/api/routes/:id/channels` | 无 | 通道列表 | 单路由下通道 |
| POST | `/api/routes/:id/channels` | `{ accountId, tokenId?, sourceModel?, priority?, weight? }` | `{ success, channel }` | 手动新增通道 |
| POST | `/api/routes/:id/channels/batch` | `{ channels }` | `{ success, created }` | 批量新增 |
| PUT | `/api/routes/:id` | `{ modelPattern?, displayName?, displayIcon?, modelMapping?, routingStrategy?, enabled? }` | `{ success, route }` | 更新路由 |
| POST | `/api/routes` | 同上 | 新建路由 |  |
| DELETE | `/api/routes/:id` | 无 | `{ success: true }` | 删除路由 |
| PUT | `/api/channels/:channelId` | `{ priority?, weight?, enabled?, tokenId?, sourceModel? }` | `{ success, channel }` | 更新通道 |
| PUT | `/api/channels/batch` | `{ updates: [{ id, priority }] }` | `{ success }` | 批量调整优先级 |
| DELETE | `/api/channels/:channelId` | 无 | `{ success: true }` | 删除通道 |
| POST | `/api/routes/rebuild` | `{ refreshModels?, wait? }` | `{ success, refresh?, rebuild? }` | 路由重建 |
| GET | `/api/routes/decision` | `?model=` | 路由决策解释 | 单模型 |
| POST | `/api/routes/decision/batch` | `{ models, refreshPricingCatalog?, persistSnapshots? }` | 决策列表 | 多模型 |
| POST | `/api/routes/decision/by-route/batch` | `{ items, refreshPricingCatalog?, persistSnapshots? }` | 决策列表 | 指定路由 |
| POST | `/api/routes/decision/route-wide/batch` | `{ routeIds, refreshPricingCatalog?, persistSnapshots? }` | 决策列表 | 路由级解释 |

### 5.6 模型与统计

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/stats/dashboard` | 无 | Dashboard 汇总 | 站点/消费/成功率等 |
| GET | `/api/stats/proxy-logs` | `limit?` `offset?` `status?` `search?` `siteId?` `from?` `to?` | `{ items, total, page, pageSize, summary }` | 日志分页 |
| GET | `/api/stats/proxy-logs/:id` | 无 | 单条日志详情 | 含 `billingDetails` |
| GET | `/api/models/marketplace` | `refresh?` `includePricing?` | 模型广场列表 | 可带价格 |
| GET | `/api/models/token-candidates` | 无 | 路由候选统计 | 用于路由页面 |
| POST | `/api/models/check/:accountId` | 无 | `{ success, refreshed, ... }` | 触发模型探测 |
| GET | `/api/stats/site-distribution` | 无 | 站点分布 | 图表数据 |
| GET | `/api/stats/site-trend` | `days?` | 站点趋势 | 图表数据 |
| GET | `/api/stats/model-by-site` | `siteId?` `days?` | 模型消耗 | 图表数据 |

### 5.7 事件与任务

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/events` | `limit?` `offset?` `type?` `read?` | `Event[]` | 系统事件 |
| GET | `/api/events/count` | 无 | `{ count }` | 未读数量 |
| POST | `/api/events/:id/read` | 无 | `{ success: true }` | 标记已读 |
| POST | `/api/events/read-all` | 无 | `{ success: true }` | 全部已读 |
| DELETE | `/api/events` | 无 | `{ success: true }` | 清空事件 |
| GET | `/api/tasks` | `limit?` | `{ tasks }` | 后台任务 |
| GET | `/api/tasks/:id` | 无 | `{ success, task }` | 任务详情 |

### 5.8 设置与维护

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/settings/auth/info` | 无 | `{ masked }` | 展示当前 Token |
| POST | `/api/settings/auth/change` | `{ oldToken, newToken }` | `{ success, message }` | 修改管理员 Token |
| GET | `/api/settings/runtime` | 无 | 运行时配置 | 含当前 IP 等 |
| PUT | `/api/settings/runtime` | 运行时配置字段 | `{ success, message? }` | 校验通知配置 |
| GET | `/api/settings/database/runtime` | 无 | 当前 DB 配置 |  |
| PUT | `/api/settings/database/runtime` | `{ dialect, connectionString, ssl? }` | `{ success, ... }` | 更新 DB 配置 |
| POST | `/api/settings/database/test-connection` | 同上 | `{ success, message }` | 测试连接 |
| POST | `/api/settings/database/migrate` | `{ dialect, connectionString, overwrite?, ssl? }` | `{ success, ... }` | 触发迁移 |
| GET | `/api/settings/backup/export` | `type?` | 导出 JSON | `type` 可选 `all` `accounts` `preferences` |
| POST | `/api/settings/backup/import` | `{ data }` | `{ success, appliedSettings }` | 导入 JSON |
| POST | `/api/settings/notify/test` | 无 | `{ success }` | 测试通知 |
| POST | `/api/settings/maintenance/clear-cache` | 无 | `{ success }` | 清理缓存 |
| POST | `/api/settings/maintenance/clear-usage` | 无 | `{ success }` | 清理使用日志 |
| POST | `/api/settings/maintenance/factory-reset` | 无 | `{ success }` | 恢复出厂 |

运行时配置字段（`/api/settings/runtime`）核心项：
- `proxyToken`、`systemProxyUrl`
- `checkinCron`、`balanceRefreshCron`
- `logCleanupCron`、`logCleanupUsageLogsEnabled`、`logCleanupProgramLogsEnabled`、`logCleanupRetentionDays`
- `routingFallbackUnitCost`、`routingWeights`
- `webhookUrl`、`webhookEnabled`
- `barkUrl`、`barkEnabled`
- `serverChanKey`、`serverChanEnabled`
- `telegramEnabled`、`telegramBotToken`、`telegramChatId`
- `smtpEnabled`、`smtpHost`、`smtpPort`、`smtpSecure`、`smtpUser`、`smtpPass`、`smtpFrom`、`smtpTo`
- `notifyCooldownSec`
- `adminIpAllowlist`

### 5.9 下游托管 Key

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/downstream-keys` | 无 | `{ success, items }` | 列表 |
| POST | `/api/downstream-keys` | `name` `key` `description?` `enabled?` `expiresAt?` `maxCost?` `maxRequests?` `supportedModels?` `allowedRouteIds?` `siteWeightMultipliers?` | `{ success, item }` | `key` 需 `sk-` 前缀 |
| PUT | `/api/downstream-keys/:id` | 同上 | `{ success, item }` |  |
| POST | `/api/downstream-keys/:id/reset-usage` | 无 | `{ success, item }` | 重置费用与请求数 |
| DELETE | `/api/downstream-keys/:id` | 无 | `{ success: true }` |  |

### 5.10 监控

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/monitor/config` | 无 | `{ config }` | 前端监控配置 |
| PUT | `/api/monitor/config` | `{ ldohCookie? }` | `{ success }` | 更新配置 |
| POST | `/api/monitor/session` | 无 | `{ session }` | 初始化会话 |

### 5.11 测试工具

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| POST | `/api/test/proxy` | `ProxyTestEnvelope` | 代理测试响应 | 同步 |
| POST | `/api/test/proxy/stream` | `ProxyTestEnvelope` | SSE | 流式 |
| POST | `/api/test/proxy/jobs` | `ProxyTestEnvelope` | `{ jobId, status, createdAt, expiresAt }` | 异步任务 |
| GET | `/api/test/proxy/jobs/:jobId` | 无 | `{ jobId, status, result?, error? }` | 查询任务 |
| DELETE | `/api/test/proxy/jobs/:jobId` | 无 | `{ success: true }` | 取消任务 |
| POST | `/api/test/chat` | `TestChatRequestBody` | 代理测试响应 | 同步 |
| POST | `/api/test/chat/stream` | `TestChatRequestBody` | SSE | 流式 |
| POST | `/api/test/chat/jobs` | `TestChatRequestBody` | `{ jobId, status, createdAt, expiresAt }` | 异步任务 |
| GET | `/api/test/chat/jobs/:jobId` | 无 | `{ jobId, status, result?, error? }` | 查询任务 |
| DELETE | `/api/test/chat/jobs/:jobId` | 无 | `{ success: true }` | 取消任务 |

`ProxyTestEnvelope` 关键字段：
- `method` `path` `requestKind` `stream?` `jobMode?` `rawMode?`
- `jsonBody?` `rawJsonText?` `multipartFields?` `multipartFiles?`

## 6. 代理端 API 契约（/v1/*）

### 5.1 Chat / Responses / Completions

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| POST | `/v1/chat/completions` | OpenAI Chat 格式 | OpenAI Chat 格式 | 支持 SSE |
| POST | `/v1/messages` | Claude Messages 格式 | Claude Messages 格式 | 支持 SSE |
| POST | `/v1/responses` | OpenAI Responses 格式 | Responses 格式 | 支持 SSE |
| POST | `/v1/responses/compact` | Responses 格式 | Responses 格式 | 紧凑响应 |
| POST | `/v1/completions` | OpenAI Completions | Completions | 兼容旧格式 |

### 5.2 Embeddings / Search

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| POST | `/v1/embeddings` | OpenAI Embeddings | Embeddings |  |
| POST | `/v1/search` | OpenAI Search | Search |  |

### 5.3 Models

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/v1/models` | 无 | `{ data: [...] }` | 聚合模型 |

### 5.4 Files

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| POST | `/v1/files` | multipart 或 JSON | File 对象 | 存储到 `proxy_files` |
| GET | `/v1/files` | 无 | 列表 |  |
| GET | `/v1/files/:fileId` | 无 | File 对象 |  |
| GET | `/v1/files/:fileId/content` | 无 | 二进制内容 |  |
| DELETE | `/v1/files/:fileId` | 无 | 删除结果 | 软删除 |

### 5.5 Images

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| POST | `/v1/images/generations` | OpenAI Images | Images |  |
| POST | `/v1/images/edits` | OpenAI Images | Images |  |
| POST | `/v1/images/variations` | OpenAI Images | Images |  |

### 5.6 Videos

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| POST | `/v1/videos` | 视频生成请求 | 任务对象 | 写入 `proxy_video_tasks` |
| GET | `/v1/videos/:id` | 无 | 任务状态 | 轮询上游 |
| DELETE | `/v1/videos/:id` | 无 | 删除结果 |  |

## 7. Gemini 兼容端（/gemini/* 与 /v1beta/*）

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/v1beta/models` | 无 | Gemini 模型列表 |  |
| GET | `/gemini/:geminiApiVersion/models` | 无 | 同上 |  |
| POST | `/v1beta/models/*` | Gemini generateContent | Gemini 响应 |  |
| POST | `/gemini/:geminiApiVersion/models/*` | Gemini generateContent | Gemini 响应 |  |

## 8. 未尽事项

- 仍需进一步补齐响应字段明细与错误码映射
- 建议在迁移前输出 OpenAPI/JSON Schema 版本以便自动化对比
