<!-- kb-mirror: upstream=control-center/docs/architecture/17-client-server-design.md sha256=4a086062d1b9010ad2b4f75058ffed782fb0df990d0c4d8b9db0332898dec30b （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

---
status: 部分实现（认证/任务/审批/审计/KB 已通；design/repos/tests/bugs 域为规划中）
last_verified: 2026-08-17
---

# 17 客户端与服务端设计（单用户）

## 17.1 服务端设计

单一后端 `control-api`（**Go + stdlib mux**），**按能力域提供统一 REST API**。用户经认证后访问端点；操作留痕写 `work_log`。端点以 `control-api/docs/api/openapi.yaml`（OAS 3.1）为唯一可信源，契约测试双向对账。

```
                  ┌────────────────────────────────────────────┐
                  │  control-api（能力域 API + 审计）            │
                  │  /api/auth  /api/tasks  /api/approvals      │
                  │  /api/audit  /api/findings  /api/kb/search  │
                  │  /api/webhooks/merge-event                  │
                  └────────────────────────────────────────────┘
```

### API 能力域

| 能力域 | 端点前缀 | 用途 | 状态 |
|-------|---------|------|------|
| 认证 | `/api/auth` | 登录（bcrypt + 随机 token 24h） | 已实现 |
| 任务 | `/api/tasks` | 任务 CRUD、状态流转、干预（暂停/恢复；合并在 Git 平台人工执行） | 已实现 |
| 审批 | `/api/approvals` | 待审批队列（按角色路由） | 已实现 |
| 问题 | `/api/findings` | 发现问题一览（FINDINGS.md 解析） | 已实现 |
| 审计 | `/api/audit` | work_log 检索 | 已实现 |
| KB 检索 | `/api/kb/search` | PieKBS FTS 检索（只读，经 control-api 代理） | 已实现 |
| Webhook | `/api/webhooks` | merge-event 回传（HMAC 独立密钥） | 已实现 |
| 设计控制 | `/api/design` | 概要/外部设计、需求、影响分析 | 规划中 |
| 仓库/Worktree | `/api/repos` `/api/worktrees` | 注册表查询、分支/Worktree 状态 | 规划中 |
| 测试/缺陷 | `/api/tests` `/api/bugs` | 测试报告、Bug 跟踪 | 规划中 |

### 认证

- 本地 token（24h 过期）；内网 SSO/OIDC 为可选项（规划中）
- 每个请求关联 `work_log`（operator 为真实用户名/agent、action、时间），任何直接调用都留痕

## 17.2 客户端：单工作台

客户端为单一 React 应用（`control-web`，React 18 + PrimeReact + Vite + pnpm），默认页为**多任务看板**，现有页面：看板/审批中心/审计日志/问题一览/KB 检索/API 文档（Scalar）；类型化 API 客户端由 openapi-typescript 从契约生成（`pnpm gen:api`）。

```
control-web/src/
├── router/            # 路由（/api-docs 懒加载 Scalar）
├── pages/             # 看板/审批/审计/问题/KB/API 文档/登录
├── components/        # 共享组件（AppLayout、ApprovalDialog…）
├── api/               # openapi-fetch 客户端（401 广播登出）
└── generated/         # 契约生成类型（机生成，豁免规约红线）
```

## 17.3 部署边界：客户端 vs 服务端

| 组件 | 部署位置 | 说明 |
|-----|---------|------|
| Web 管理端 UI | **客户端**（`control-web` 静态资源；dev 经 vite 代理 `/api`） | 纯展示与交互，无逻辑、无数据访问 |
| Web 管理端**逻辑与数据** | **服务端** `control-api` | 任务、审批、审计全在服务端 |
| **KB 引擎** | **服务端** `control-piekbs`（独立进程 127.0.0.1:8766） | 蒸馏/索引/检索全部服务端；客户端经 `/api/kb/search` 只读检索 |
| 数据层（SQLite） | **服务端** `~/data/control.db` | 客户端不直连数据库 |

> 原则：**客户端 = 纯展示**（零业务逻辑、零数据直连），一切能力的服务端实现都部署在服务端进程；客户端只经内网 API 交互。
