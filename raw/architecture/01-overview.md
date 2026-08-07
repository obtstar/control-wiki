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
│  平台实现：control-api(Spring) · control-web(React) · control-db(MySQL)   │
│  业务代码：billing-core ...                                               │
│  Git Worktree 多分支：main · dev · release · feature/*                   │
│  内部设计（docs/design/internal/）与本仓库代码同库                        │
└──────────────────────────────────────────────────────────────────────────┘
```

## 组成与职责

| 组成 | 职责 | 关键技术 |
|-----|------|---------|
| 控制中心仓库 | **独立仓库，无代码实现**：设计开发控制文档（概要/外部设计、需求）+ 任务编排配置 + **注册表（registry/）**；不被自身调度 | Git / 编排配置 |
| 平台代码仓库 | 平台实现代码：后端 `control-api`、前端 `control-web`、数据库 `control-db` | Java Spring Boot / React + PrimeReact + Vite / MySQL |
| Web 管理端 | 用户入口：多任务看板、任务创建/干预、Diff 预览（合并跳转 GitLab 人工执行）、日志与 RAG 检索（control-web 代码仓库构建部署） | React + PrimeReact + Vite + Nginx |
| 后端服务 | 任务编排、流水线状态机、RAG、外部集成（control-api 代码仓库） | Java Spring Boot |
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

- **前端**：React + PrimeReact + Vite + Nginx（内网）
- **后端**：Java Spring Boot（Spring MVC + Spring Data JPA/MyBatis + Spring Security）
- **数据库**：MySQL（任务/日志等运行时数据；**注册表在 Git**，DDL/DML 版本化管控）
- **执行层**：pi.dev 核心工具（Earendil Pi，MIT；RPC 模式对接 control-api，原生 Skills/Extensions）+ Git Worktree
- **AI**：消费企业内 LiteLLM 代理外接 Anthropic（Claude）+ GitHub Copilot（GPT-4o），**不跑本地 Ollama、不重复部署 LiteLLM**（见 [04](04-ai-gateway.md)）
