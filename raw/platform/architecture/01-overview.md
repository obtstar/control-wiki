<!-- kb-mirror: upstream=control-center/docs/architecture/01-overview.md sha256=11b236a2b19ec6a28381ff0b79b6417f92bb74498435ec8790696cf7182e3b3b （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

---
status: 已实现
last_verified: 2026-08-11
---

# 01 总体架构：控制中心仓库 + 多代码仓库

系统由**一个独立控制中心仓库**与**多个代码仓库**组成。控制中心仅承载设计开发控制文档、任务编排配置与**注册表（registry/）**（无代码实现）；代码仓库承载全部代码（平台实现 + 业务），通过 Git Worktree 多分支管理实现任务的物理隔离与**单人多任务并行执行**。

```
┌──────────────────────────────────────────────────────────────────────────┐
│  控制中心仓库（独立，纯设计/控制，无代码实现）                             │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  设计开发控制文档（集中，Git 版本化）                               │ │
│  │  docs/design: 概要设计 · 外部设计 · 需求 · 架构文档                 │ │
│  ├────────────────────────────────────────────────────────────────────┤ │
│  │  任务编排配置（版本化）                                             │ │
│  │  orchestration: 流水线状态机 · Prompt · Skill（openskills）· 自我升级配置   │ │
│  ├────────────────────────────────────────────────────────────────────┤ │
│  │  注册表（版本化）                                                   │ │
│  │  registry: repos.yaml（仓库注册）· executors.yaml（执行节点登记）   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
                              │  下发任务 / 创建 Worktree / 收集日志
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  多个代码仓库（平台实现 + 业务代码，db/backend/frontend 均在此）           │
│  平台实现：control-api(Go) · control-web(React) · control-db(占位仓)      │
│  业务代码：billing-core ...                                               │
│  Git Worktree 多分支：main · dev · release · feature/*                   │
│  内部设计（docs/design/internal/）与本仓库代码同库                        │
└──────────────────────────────────────────────────────────────────────────┘
```

## 组成与职责

| 组成 | 职责 | 关键技术 |
|-----|------|---------|
| 控制中心仓库 | **独立仓库，无代码实现**：设计开发控制文档（概要/外部设计、需求）+ 任务编排配置 + **注册表（registry/）**；不被自身调度 | Git / 编排配置 |
| 平台代码仓库 | 平台实现代码：后端 `control-api`、前端 `control-web`、数据库 `control-db`（占位仓库，DDL 未落地，SQLite schema 由 control-api `internal/store` 迁移管理） | Go 1.25（stdlib `net/http` mux，无框架）/ React + PrimeReact + Vite / SQLite（modernc.org/sqlite） |
| Web 管理端 | 用户入口：多任务看板、任务创建/干预、Diff 预览（合并跳转 GitLab 人工执行）、日志与 KB 检索（control-web 代码仓库构建部署） | React + PrimeReact + Vite + Nginx |
| 后端服务 | 任务编排、流水线状态机、KB 检索（经 PieKBS）、外部集成（control-api 代码仓库） | Go 1.25（stdlib `net/http` mux） |
| 业务代码仓库 | 业务代码物理载体 + **内部设计与代码同库**，Worktree 多分支管理 | Git Worktree / Git |

## 集成对象

| 集成 | 用途 |
|-----|------|
| 控制中心仓库 Git | 概要设计/外部设计/需求 + 编排配置 + **registry 注册表**，MR 变更 |
| 代码仓库 Git | 全部代码实现（db/backend/frontend）；**内部设计随代码同库**（docs/design/internal/） |
| 仓库 OpenAPI | 代码仓库接口对接（分支、MR、提交、Webhook） |
| LiteLLM | 企业内既有代理（消费端直连，不重复部署）：统一路由 Anthropic + GitHub Copilot |
| 代码仓库 Webhook | 变更触发增量索引与任务流转 |
| Confluence（可选） | 外部文档读取 / 设计文档发布出口，非权威源 |

## 技术栈速览

- **前端**：React + PrimeReact + Vite + Nginx（内网；MVP 已实现）
- **后端**：Go 1.25（stdlib `net/http` mux，无 Web 框架）
- **数据库**：SQLite（modernc.org/sqlite，WAL；运行时库 `~/data/control.db`；schema 迁移由 control-api `internal/store` 管理；**注册表在 Git**）
- **执行层**：pi.dev 核心工具（Earendil Pi，MIT；RPC 模式对接 control-api，原生 Skills/Extensions）+ Git Worktree
- **AI**：消费企业内 LiteLLM 代理外接 Anthropic（Claude）+ GitHub Copilot（GPT-4o），**不跑本地 Ollama、不重复部署 LiteLLM**（见 [04](04-ai-gateway.md)）
