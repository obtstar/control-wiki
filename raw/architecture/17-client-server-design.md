# 17 客户端与服务端设计（单用户）

## 17.1 服务端设计

单一后端 `control-api`（Java Spring Boot），**按能力域提供统一 REST API**。唯一用户经认证后访问全部端点；操作留痕写 `work_log`。

```
                  ┌────────────────────────────────────────────┐
                  │  control-api（能力域 API + 审计）            │
                  │  /api/auth   /api/design  /api/tasks       │
                  │  /api/repos  /api/worktrees                │
                  │  /api/tests  /api/bugs  /api/audit  /api/rag │
                  └────────────────────────────────────────────┘
```

### API 能力域

| 能力域 | 端点前缀 | 用途 |
|-------|---------|------|
| 认证 | `/api/auth` | 单用户登录（本地 token / 内网 SSO 可选） |
| 设计控制 | `/api/design` | 概要/外部设计、需求、影响分析 |
| 任务 | `/api/tasks` | 任务 CRUD、状态流转、干预（暂停/回退；合并在 GitLab 人工执行） |
| 仓库/分支/Worktree | `/api/repos` `/api/worktrees` | 注册表查询、分支/Worktree 状态 |
| 测试 | `/api/tests` | 测试报告 |
| 缺陷 | `/api/bugs` | Bug 提交/跟踪 |
| 审计 | `/api/audit` | work_log 检索 |
| RAG 检索 | `/api/rag` | 语义检索（只读） |

### 认证

- 单用户：本地 token 或内网 SSO/OIDC + JWT（可选）
- 每个请求关联 `work_log`（operator、action、时间），任何直接调用都留痕

## 17.2 客户端：单工作台

客户端为单一 React 应用（`control-web`，PrimeReact + Vite），默认页为**多任务看板**，聚合全部功能：任务操作、Diff 预览、GitLab MR 跳转合并、日志检索、RAG 检索、工作报告（模块见 [06](06-web.md)）。

```
control-web/src/
├── router/            # 路由
├── pages/             # 各能力域页面
├── components/        # 共享组件（DiffViewer、AuditTable、WorktreePanel、RagSearch…）
└── services/          # 统一调用 control-api
```

## 17.3 部署边界：客户端 vs 服务端

| 组件 | 部署位置 | 说明 |
|-----|---------|------|
| Web 管理端 UI | **客户端**（`control-web` 静态资源 + Nginx） | 纯展示与交互，无逻辑、无数据访问 |
| Web 管理端**逻辑与数据** | **服务端** `control-api` | 任务、仓库、审计全在服务端 |
| **RAG 引擎** | **服务端** `control-api/service/rag` + Milvus | 索引、切分、向量化、检索全部服务端；客户端仅调 `/api/rag` 只读检索 |
| 数据层（MySQL/Milvus） | **服务端** | 客户端不直连数据库 |

> 原则：**客户端 = 纯展示**（零业务逻辑、零数据直连），一切能力（含 RAG）的服务端实现都部署在 `control-api` 所在服务端；客户端只经内网 API 交互。
