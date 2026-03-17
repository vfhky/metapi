# 管理后台页面与 API 依赖映射

本文档整理 React 管理后台页面与后端 API 的依赖关系，用于前端改造前的接口冻结与回归范围确认。

## 1. 说明

- 以 `src/web/pages/*.tsx` 为主入口
- API 映射来源为 `src/web/api.ts`
- 仅列出页面直接调用的接口

## 2. 页面与接口映射

| 页面 | 主要 API 依赖 |
| --- | --- |
| `src/web/pages/Dashboard.tsx` | `/api/stats/dashboard`<br>`/api/stats/site-distribution`<br>`/api/stats/site-trend`<br>`/api/sites` |
| `src/web/pages/Sites.tsx` | `/api/sites` (GET/POST/PUT/DELETE)<br>`/api/sites/batch`<br>`/api/sites/detect` |
| `src/web/pages/Accounts.tsx` | `/api/accounts`<br>`/api/sites`<br>`/api/accounts/login`<br>`/api/accounts/verify-token`<br>`/api/accounts/:id/rebind-session`<br>`/api/accounts/:id` (PUT/DELETE)<br>`/api/accounts/batch`<br>`/api/accounts/health/refresh`<br>`/api/accounts/:id/balance`<br>`/api/checkin/trigger`<br>`/api/checkin/trigger/:id`<br>`/api/models/check/:accountId` |
| `src/web/pages/Tokens.tsx` | `/api/account-tokens`<br>`/api/account-tokens/:id` (PUT/DELETE)<br>`/api/account-tokens/:id/value`<br>`/api/account-tokens/:id/default`<br>`/api/account-tokens/batch`<br>`/api/account-tokens/groups/:accountId`<br>`/api/account-tokens/sync/:accountId`<br>`/api/account-tokens/sync-all`<br>`/api/accounts` |
| `src/web/pages/TokenRoutes.tsx` | `/api/routes/summary`<br>`/api/models/token-candidates`<br>`/api/routes` (GET/POST/PUT/DELETE)<br>`/api/routes/rebuild`<br>`/api/routes/:id/channels`<br>`/api/routes/:id/channels/batch`<br>`/api/channels/:channelId` (PUT/DELETE)<br>`/api/channels/batch`<br>`/api/routes/decision`<br>`/api/routes/decision/batch`<br>`/api/routes/decision/by-route/batch`<br>`/api/routes/decision/route-wide/batch` |
| `src/web/pages/Models.tsx` | `/api/models/marketplace` |
| `src/web/pages/ModelTester.tsx` | `/api/models/marketplace`<br>`/api/routes`<br>`/api/test/proxy`<br>`/api/test/proxy/stream`<br>`/api/test/proxy/jobs`<br>`/api/test/proxy/jobs/:jobId` (GET/DELETE) |
| `src/web/pages/ProxyLogs.tsx` | `/api/sites`<br>`/api/stats/proxy-logs`<br>`/api/stats/proxy-logs/:id` |
| `src/web/pages/CheckinLog.tsx` | `/api/checkin/logs`<br>`/api/checkin/trigger` |
| `src/web/pages/ProgramLogs.tsx` | `/api/events`<br>`/api/events/count`<br>`/api/events/:id/read`<br>`/api/events/read-all`<br>`/api/events` (DELETE) |
| `src/web/pages/NotificationSettings.tsx` | `/api/settings/runtime` (GET/PUT)<br>`/api/settings/notify/test` |
| `src/web/pages/Settings.tsx` | `/api/downstream-keys` (GET/POST/PUT/DELETE)<br>`/api/downstream-keys/:id/reset-usage`<br>`/api/routes/lite`<br>`/api/settings/auth/info`<br>`/api/settings/runtime` (GET/PUT)<br>`/api/settings/database/runtime` (GET/PUT)<br>`/api/settings/database/test-connection`<br>`/api/settings/database/migrate`<br>`/api/settings/maintenance/clear-cache`<br>`/api/settings/maintenance/clear-usage`<br>`/api/settings/maintenance/factory-reset`<br>`/api/settings/notify/test` |
| `src/web/pages/ImportExport.tsx` | `/api/settings/backup/export`<br>`/api/settings/backup/import` |
| `src/web/pages/Monitors.tsx` | `/api/monitor/config` (GET/PUT)<br>`/api/monitor/session` |
| `src/web/pages/About.tsx` | 无后端依赖 |

## 3. 说明性补充

- 统一鉴权由 `src/web/api.ts` 处理，任何 401/403 会触发前端清理会话
- 路由决策与重建相关接口是前端改造的高风险区域，建议优先冻结
