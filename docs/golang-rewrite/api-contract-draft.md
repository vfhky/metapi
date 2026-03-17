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
- 统计类接口的日期与时间桶以服务端本地时区为准（由运行环境 `TZ` 决定）

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
| `tokenMasked` | string? | 脱敏 Token（列表接口返回） |
| `tokenGroup` | string? | 分组 |
| `valueStatus` | string | `ready` / `masked_pending` |
| `source` | string | `manual` / `sync` / `legacy` |
| `enabled` | boolean | 是否启用 |
| `isDefault` | boolean | 是否默认 |

`valueStatus` 说明：
| 状态 | 说明 |
| --- | --- |
| `ready` | 已保存明文，可用于路由与查看 |
| `masked_pending` | 仅保存脱敏令牌，需同步或手动补全 |

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

RouteChannelView（带关联信息）：
| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `account` | object | 账号对象（`Account`） |
| `site` | object | 站点对象（`Site`） |
| `token` | object? | 令牌摘要（`{ id, name, accountId, enabled, isDefault }`） |

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
| `httpStatus` | number? | 上游 HTTP 状态码 |
| `latencyMs` | number | 耗时 |
| `promptTokens` | number? | 输入 token |
| `completionTokens` | number? | 输出 token |
| `totalTokens` | number? | 总 token |
| `estimatedCost` | number? | 估算成本 |
| `billingDetails` | object? | 计费明细 |
| `errorMessage` | string? | 错误信息 |
| `retryCount` | number? | 重试次数 |
| `createdAt` | string? | 创建时间 |
| `username` | string? | 账号名（列表 join 字段） |
| `siteId` | number? | 站点 ID（列表 join 字段） |
| `siteName` | string? | 站点名称（列表 join 字段） |
| `siteUrl` | string? | 站点 URL（列表 join 字段） |

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
| `relatedId` | number? | 关联实体 ID |
| `relatedType` | string? | 关联实体类型 |
| `createdAt` | string? | 创建时间 |

Task：
| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 任务 ID |
| `type` | string | 类型 |
| `title` | string | 任务标题 |
| `status` | string | `pending` / `running` / `succeeded` / `failed` |
| `message` | string | 进度描述 |
| `error` | string? | 失败原因 |
| `result` | object? | 任务结果 |
| `dedupeKey` | string? | 去重键 |
| `createdAt` | string | 创建时间 |
| `updatedAt` | string | 更新时间 |
| `startedAt` | string? | 开始时间 |
| `finishedAt` | string? | 结束时间 |
| `expiresAtMs` | number | 过期时间戳毫秒 |

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
| PUT | `/api/checkin/schedule` | `{ cron }` | `{ success, cron }` 或 `{ error }` | 更新签到 Cron |

`CheckinLog`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | 日志 ID |
| `accountId` | number | 账号 ID |
| `status` | string | `success` / `failed` / `skipped` |
| `message` | string? | 消息 |
| `reward` | string? | 奖励文本 |
| `createdAt` | string? | 创建时间 |

`CheckinLogRow`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `checkin_logs` | object | 签到记录（`CheckinLog`） |
| `accounts` | object | 账号对象（`Account`） |
| `sites` | object | 站点对象（`Site`） |
| `failureReason` | object | 失败原因（见下） |

`CheckinLogRow.failureReason`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `code` | string | 原因代码 |
| `category` | string | 分类 |
| `title` | string | 标题 |
| `actionHint` | string | 处理建议 |
| `detailHint` | string | 详情提示 |

### 5.5 路由与通道

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/routes` | 无 | 路由列表 | 含通道与统计信息 |
| GET | `/api/routes/lite` | 无 | 路由简版列表 | 用于设置页 |
| GET | `/api/routes/summary` | 无 | 路由摘要 | UI 预览 |
| GET | `/api/routes/:id/channels` | 无 | `RouteChannelView[]` | 单路由下通道 |
| POST | `/api/routes/:id/channels` | `{ accountId, tokenId?, sourceModel?, priority?, weight? }` | `RouteChannel` | 手动新增通道 |
| POST | `/api/routes/:id/channels/batch` | `{ channels }` | `{ success, created, skipped, errors }` | 批量新增 |
| PUT | `/api/routes/:id` | `{ modelPattern?, displayName?, displayIcon?, modelMapping?, routingStrategy?, enabled? }` | `Route` 或 `{ success, message }` | 更新路由 |
| POST | `/api/routes` | 同上 | `Route` 或 `{ success, message }` | 新建路由 |
| DELETE | `/api/routes/:id` | 无 | `{ success: true }` | 删除路由 |
| PUT | `/api/channels/:channelId` | `{ priority?, weight?, enabled?, tokenId?, sourceModel? }` | `RouteChannel` 或 `{ success, message }` | 更新通道 |
| PUT | `/api/channels/batch` | `{ updates: [{ id, priority }] }` | `{ success, channels }` | 批量调整优先级 |
| DELETE | `/api/channels/:channelId` | 无 | `{ success: true }` | 删除通道 |
| POST | `/api/routes/rebuild` | `{ refreshModels?, wait? }` | `200: { success, refresh?, rebuild? }` `202: { success, queued, jobId, status, message }` | 路由重建 |
| GET | `/api/routes/decision` | `?model=` | `{ success, decision }` | 单模型 |
| POST | `/api/routes/decision/batch` | `{ models, refreshPricingCatalog?, persistSnapshots? }` | `{ success, decisions }` | 多模型 |
| POST | `/api/routes/decision/by-route/batch` | `{ items, refreshPricingCatalog?, persistSnapshots? }` | `{ success, decisions }` | 指定路由 |
| POST | `/api/routes/decision/route-wide/batch` | `{ routeIds, refreshPricingCatalog?, persistSnapshots? }` | `{ success, decisions }` | 路由级解释 |

### 5.6 模型与统计

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/stats/dashboard` | 无 | Dashboard 汇总 | 站点/消费/成功率等 |
| GET | `/api/stats/proxy-logs` | `limit?` `offset?` `status?` `search?` `siteId?` `from?` `to?` | `{ items, total, page, pageSize, summary }` | 日志分页 |
| GET | `/api/stats/proxy-logs/:id` | 无 | 单条日志详情 | 含 `billingDetails` |
| GET | `/api/models/marketplace` | `refresh?` `includePricing?` | 模型广场列表 | 可带价格 |
| GET | `/api/models/token-candidates` | 无 | 路由候选统计 | 用于路由页面 |
| POST | `/api/models/check/:accountId` | 无 | `{ success, refresh, rebuild }` | 触发模型探测 |
| GET | `/api/stats/site-distribution` | 无 | 站点分布 | 图表数据 |
| GET | `/api/stats/site-trend` | `days?` | 站点趋势 | 图表数据 |
| GET | `/api/stats/model-by-site` | `siteId?` `days?` | 模型消耗 | 图表数据 |
| POST | `/api/search` | `{ query, limit? }` | `SearchResponse` | 统一搜索 |

#### 5.6.1 DashboardStats

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `totalBalance` | number | 活跃站点总余额 |
| `totalUsed` | number | 历史消耗汇总 |
| `todaySpend` | number | 今日消耗 |
| `todayReward` | number | 今日签到奖励估算 |
| `activeAccounts` | number | 活跃账号数量 |
| `totalAccounts` | number | 活跃站点账号数量 |
| `todayCheckin` | object | 今日签到汇总 |
| `proxy24h` | object | 近 24 小时代理汇总 |
| `performance` | object | 最近一分钟性能 |
| `siteAvailability` | object[] | 站点可用性 |
| `modelAnalysis` | object | 模型分析 |

`todayCheckin`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `success` | number | 成功数量 |
| `failed` | number | 失败数量 |
| `total` | number | 总次数 |

`proxy24h`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `success` | number | 成功请求数 |
| `failed` | number | 失败请求数 |
| `total` | number | 总请求数 |
| `totalTokens` | number | 总 token |

`performance`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `windowSeconds` | number | 统计窗口秒数 |
| `requestsPerMinute` | number | 每分钟请求数 |
| `tokensPerMinute` | number | 每分钟 token |

`siteAvailability` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `siteId` | number | 站点 ID |
| `siteName` | string | 站点名称 |
| `siteUrl` | string? | 站点 URL |
| `platform` | string? | 平台类型 |
| `totalRequests` | number | 总请求数 |
| `successCount` | number | 成功数 |
| `failedCount` | number | 失败数 |
| `availabilityPercent` | number? | 可用率 |
| `averageLatencyMs` | number? | 平均延迟 |
| `buckets` | object[] | 时间桶 |

`siteAvailability.buckets` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `startUtc` | string | 桶起始时间 |
| `label` | string | 本地时间标记 |
| `totalRequests` | number | 总请求数 |
| `successCount` | number | 成功数 |
| `failedCount` | number | 失败数 |
| `availabilityPercent` | number? | 可用率 |
| `averageLatencyMs` | number? | 平均延迟 |

#### 5.6.2 ModelAnalysis

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `window` | object | 统计窗口 |
| `totals` | object | 汇总 |
| `spendDistribution` | object[] | 消耗分布（按模型） |
| `spendTrend` | object[] | 消耗趋势（按日） |
| `callsDistribution` | object[] | 调用分布（按模型） |
| `callRanking` | object[] | 调用排名 |

`window`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `start` | string | 起始日期 |
| `end` | string | 结束日期 |
| `days` | number | 天数 |

`totals`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `calls` | number | 总调用数 |
| `tokens` | number | 总 token |
| `spend` | number | 总消耗 |

`spendDistribution` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `model` | string | 模型 |
| `spend` | number | 消耗 |
| `calls` | number | 调用数 |

`spendTrend` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `day` | string | 日期 |
| `spend` | number | 消耗 |

`callsDistribution` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `model` | string | 模型 |
| `calls` | number | 调用数 |
| `share` | number | 占比 |

`callRanking` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `model` | string | 模型 |
| `calls` | number | 调用数 |
| `successRate` | number | 成功率 |
| `avgLatencyMs` | number | 平均延迟 |
| `spend` | number | 消耗 |
| `tokens` | number | token |

#### 5.6.3 ProxyLogListResponse

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `items` | object[] | 日志列表（`ProxyLog` 字段） |
| `total` | number | 总记录数 |
| `page` | number | 当前页 |
| `pageSize` | number | 分页大小 |
| `summary` | object | 汇总 |

`summary`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `totalCount` | number | 记录数 |
| `successCount` | number | 成功数 |
| `failedCount` | number | 失败数 |
| `totalCost` | number | 总成本 |
| `totalTokensAll` | number | 总 token |

#### 5.6.4 ModelMarketplaceResponse

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `models` | object[] | 模型列表 |
| `meta` | object | 元信息 |

`models` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `name` | string | 模型名 |
| `accountCount` | number | 账号数量 |
| `tokenCount` | number | token 数量 |
| `avgLatency` | number | 平均延迟 |
| `successRate` | number? | 成功率 |
| `description` | string? | 描述 |
| `tags` | string[] | 标签 |
| `supportedEndpointTypes` | string[] | 支持的端点类型 |
| `pricingSources` | object[] | 价格来源 |
| `accounts` | object[] | 账号列表 |

`accounts` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | 账号 ID |
| `site` | string | 站点名 |
| `username` | string? | 账号名 |
| `latency` | number? | 延迟 |
| `unitCost` | number? | 单价 |
| `balance` | number | 余额 |
| `tokens` | object[] | 账号 token |

`tokens` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | Token ID |
| `name` | string | Token 名称 |
| `isDefault` | boolean | 是否默认 |

`pricingSources` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `siteId` | number | 站点 ID |
| `siteName` | string | 站点名称 |
| `accountId` | number | 账号 ID |
| `username` | string? | 账号名 |
| `ownerBy` | string? | 上游归属 |
| `enableGroups` | string[] | 价格分组 |
| `groupPricing` | object | 分组价格映射 |

价格相关字段为上游目录透传，部分字段可能为空或缺失，迁移时需保持兼容。

`meta`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `refreshRequested` | boolean | 是否请求刷新 |
| `refreshQueued` | boolean | 是否新建任务 |
| `refreshReused` | boolean | 是否复用任务 |
| `refreshRunning` | boolean | 是否正在执行 |
| `refreshJobId` | string? | 任务 ID |
| `includePricing` | boolean | 是否含价格 |
| `cacheHit` | boolean? | 是否命中缓存 |

#### 5.6.5 TokenCandidateResponse

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `models` | object | 模型 → Token 列表 |
| `modelsWithoutToken` | object | 模型 → 无 token 的账号 |
| `modelsMissingTokenGroups` | object | 模型 → 缺失分组 |
| `endpointTypesByModel` | object | 模型 → 端点类型 |

`models` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `accountId` | number | 账号 ID |
| `tokenId` | number | Token ID |
| `tokenName` | string | Token 名称 |
| `isDefault` | boolean | 是否默认 |
| `username` | string? | 账号名 |
| `siteId` | number | 站点 ID |
| `siteName` | string | 站点名称 |

`modelsWithoutToken` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `accountId` | number | 账号 ID |
| `username` | string? | 账号名 |
| `siteId` | number | 站点 ID |
| `siteName` | string | 站点名称 |

`modelsMissingTokenGroups` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `accountId` | number | 账号 ID |
| `username` | string? | 账号名 |
| `siteId` | number | 站点 ID |
| `siteName` | string | 站点名称 |
| `missingGroups` | string[] | 缺失分组 |
| `requiredGroups` | string[] | 需覆盖分组 |
| `availableGroups` | string[] | 已覆盖分组 |
| `groupCoverageUncertain` | boolean? | 分组识别不完整 |

`endpointTypesByModel` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `key` | string | 模型名 |
| `value` | string[] | 端点类型数组 |

#### 5.6.6 SiteDistribution / SiteTrend / ModelBySite

`/api/stats/site-distribution`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `distribution` | object[] | 站点分布列表 |

`distribution` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `siteId` | number | 站点 ID |
| `siteName` | string | 站点名称 |
| `platform` | string? | 平台类型 |
| `totalBalance` | number | 总余额 |
| `totalSpend` | number | 总消耗 |
| `accountCount` | number | 账号数 |

`/api/stats/site-trend`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `trend` | object[] | 站点趋势列表 |

#### 5.6.7 SearchResponse

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `accounts` | object[] | 账号搜索结果（`SearchAccountItem`） |
| `accountTokens` | object[] | 令牌搜索结果（`SearchAccountTokenItem`） |
| `sites` | object[] | 站点搜索结果（`Site`） |
| `checkinLogs` | object[] | 签到记录（`SearchCheckinLogItem`） |
| `proxyLogs` | object[] | 代理日志（`ProxyLogListItem`） |
| `models` | object[] | 模型聚合（`SearchModelItem`） |

`SearchAccountTokenItem` 说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `token` | string | 令牌值（可能为脱敏值） |
| `valueStatus` | string | `ready` / `masked_pending` |
| `account` | object | `{ id, username?, segment }` |
| `site` | object | 站点对象（`Site`） |

`trend` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `date` | string | 日期 |
| `sites` | object | 站点统计映射（键为站点名） |

`sites` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `spend` | number | 消耗 |
| `calls` | number | 调用数 |

`/api/stats/model-by-site`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `models` | object[] | 模型消耗列表 |

`models` 单项：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `model` | string | 模型 |
| `calls` | number | 调用数 |
| `spend` | number | 消耗 |
| `tokens` | number | token |

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
| GET | `/api/settings/runtime` | 无 | `RuntimeSettings` | 含当前 IP 等 |
| PUT | `/api/settings/runtime` | 运行时配置字段 | `{ ...RuntimeSettings, success, message }` | 校验通知配置 |
| GET | `/api/settings/database/runtime` | 无 | `DatabaseRuntimeResponse` |  |
| PUT | `/api/settings/database/runtime` | `{ dialect, connectionString, ssl? }` | `DatabaseRuntimeUpdateResponse` | 更新 DB 配置 |
| POST | `/api/settings/database/test-connection` | 同上 | `{ success, message, dialect, connection }` | 测试连接 |
| POST | `/api/settings/database/migrate` | `{ dialect, connectionString, overwrite?, ssl? }` | `DatabaseMigrationResponse` | 触发迁移 |
| GET | `/api/settings/backup/export` | `type?` | `BackupExportResponse` | `type` 可选 `all` `accounts` `preferences` |
| POST | `/api/settings/backup/import` | `{ data }` | `BackupImportResponse` | 导入 JSON |
| POST | `/api/settings/notify/test` | 无 | `{ success }` | 测试通知 |
| POST | `/api/settings/maintenance/clear-cache` | 无 | `{ success }` | 清理缓存 |
| POST | `/api/settings/maintenance/clear-usage` | 无 | `{ success }` | 清理使用日志 |
| POST | `/api/settings/maintenance/factory-reset` | 无 | `{ success }` | 恢复出厂 |

运行时配置字段（`/api/settings/runtime`）核心项：
- `proxyToken`、`proxyTokenMasked`、`systemProxyUrl`
- `checkinCron`、`balanceRefreshCron`
- `logCleanupCron`、`logCleanupUsageLogsEnabled`、`logCleanupProgramLogsEnabled`、`logCleanupRetentionDays`
- `routingFallbackUnitCost`、`routingWeights`
- `webhookUrl`、`webhookEnabled`
- `barkUrl`、`barkEnabled`
- `serverChanKey`、`serverChanKeyMasked`、`serverChanEnabled`
- `telegramEnabled`、`telegramBotToken`、`telegramBotTokenMasked`、`telegramChatId`
- `smtpEnabled`、`smtpHost`、`smtpPort`、`smtpSecure`、`smtpUser`、`smtpPass`、`smtpPassMasked`、`smtpFrom`、`smtpTo`
- `notifyCooldownSec`
- `adminIpAllowlist`

说明：GET 返回的敏感字段均为脱敏值，PUT 需要提供明文值。

### 5.9 下游托管 Key

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/downstream-keys` | 无 | `{ success, items }` | 列表 |
| POST | `/api/downstream-keys` | `name` `key` `description?` `groupName?` `tags?` `enabled?` `expiresAt?` `maxCost?` `maxRequests?` `supportedModels?` `allowedRouteIds?` `siteWeightMultipliers?` | `{ success, item }` | `key` 需 `sk-` 前缀 |
| PUT | `/api/downstream-keys/:id` | 同上 | `{ success, item }` |  |
| POST | `/api/downstream-keys/:id/reset-usage` | 无 | `{ success, item }` | 重置费用与请求数 |
| DELETE | `/api/downstream-keys/:id` | 无 | `{ success: true }` |  |
| GET | `/api/downstream-keys/summary` | `range?` `status?` `search?` `group?` `tags?` `tagMatch?` | `DownstreamKeySummaryResponse` | 汇总列表 |
| GET | `/api/downstream-keys/:id/overview` | 无 | `DownstreamKeyOverviewResponse` | 概览与分段使用 |
| GET | `/api/downstream-keys/:id/trend` | `range?` | `DownstreamKeyTrendResponse` | 趋势桶 |
| POST | `/api/downstream-keys/batch` | `{ ids, action, groupOperation?, groupName?, tagOperation?, tags? }` | `{ success, successIds, failedItems }` | action: `enable` `disable` `delete` `resetUsage` `updateMetadata` |

### 5.10 监控

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| GET | `/api/monitor/config` | 无 | `{ ldohCookieConfigured, ldohCookieMasked }` | 前端监控配置 |
| PUT | `/api/monitor/config` | `{ ldohCookie? }` | `{ success, message, ldohCookieConfigured, ldohCookieMasked? }` | 更新配置 |
| POST | `/api/monitor/session` | 无 | `{ success: true }` | 初始化会话 |

### 5.11 测试工具

| 方法 | 路径 | 请求 | 响应 | 备注 |
| --- | --- | --- | --- | --- |
| POST | `/api/test/proxy` | `ProxyTestEnvelope` | 透传上游响应 | 同步 |
| POST | `/api/test/proxy/stream` | `ProxyTestEnvelope` | SSE | 流式 |
| POST | `/api/test/proxy/jobs` | `ProxyTestEnvelope` | `{ jobId, status, createdAt, expiresAt }` | 异步任务 |
| GET | `/api/test/proxy/jobs/:jobId` | 无 | `{ jobId, status, result?, error?, createdAt, updatedAt, expiresAt }` | 查询任务 |
| DELETE | `/api/test/proxy/jobs/:jobId` | 无 | `{ success: true }` | 取消任务 |
| POST | `/api/test/chat` | `TestChatRequestBody` | 透传上游响应 | 同步 |
| POST | `/api/test/chat/stream` | `TestChatRequestBody` | SSE | 流式 |
| POST | `/api/test/chat/jobs` | `TestChatRequestBody` | `{ jobId, status, createdAt, expiresAt }` | 异步任务 |
| GET | `/api/test/chat/jobs/:jobId` | 无 | `{ jobId, status, result?, error?, createdAt, updatedAt, expiresAt }` | 查询任务 |
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
