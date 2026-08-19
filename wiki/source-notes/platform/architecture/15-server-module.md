---
type: source-note
title: "15 服务器模块设计（control-api）"
description: ""
tags: []
resource: ""
timestamp: "2026-08-19T14:25:40Z"
processing: lightweight
sources:
  - raw/platform/architecture/15-server-module.md
schema_version: 1
---

<!-- kb-mirror: upstream=control-center/docs/architecture/15-server-module.md sha256=c8efd438468a0466f6c4afef563e88fcd801d2d4e2b741ee652e33237aee2562 （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

---
status: 部分实现（任务/审批/审计/KB 检索已通；executor 派发/工作报告/自我升级为规划中）
last_verified: 2026-08-17
---

# 15 服务器模块设计（control-api）

## 15.1 概述

服务器模块即平台后端 `control-api`（**Go 1.25+，stdlib `net/http` mux，无 Web 框架**），是系统唯一的后端服务，负责任务编排、流水线状态机、任务干预与操作追溯、KB 检索代理。运行路径基于工作用户 home（`~/`，本环境 `/home/dev`）布局，见 [13-repo-template.md](13-repo-template.md)。

```
Web (control-web, React) ──REST──→ control-api (Go) ──→ SQLite (~/data/control.db, WAL)
                                      │
                                      ├── PieKBS（127.0.0.1:8766，REST /api/search + MCP /mcp）
                                      ├── pi 执行器（子进程，模型经 agent.models 配置路由）
                                      ├── Git 本地操作（~/wt）
                                      ├── 控制中心仓库（orchestration/ 热加载 + registry/）
                                      └── 各代码仓库 OpenAPI（分支/MR/Webhook，规划中）
```

## 15.2 模块结构（Go 包，一领域一包）

```
control-api/
├── cmd/control-api/main.go       # 入口：子命令解析（serve/check/verify-log/service/user）
├── internal/
│   ├── agent/                    # pi 执行器：阶段子进程调用 + 报告落盘
│   ├── api/                      # HTTP 路由与 handler（路由集中 server.go）
│   ├── authn/                    # 用户/会话/角色（bcrypt + 随机 token）
│   ├── config/                   # 配置加载（yaml + env 覆盖，密钥不落盘）
│   ├── engine/                   # 状态机：approve/reject/pause/resume + 熔断重试 + KB grounding
│   ├── kb/                       # KB 检索窄接口（RESTSearcher → PieKBS）
│   ├── pipeline/                 # orchestration/workflows/*.yaml 加载与热加载
│   ├── service/                  # systemd 用户级服务管理
│   ├── store/                    # SQLite 运行时层（WAL；DDL 内嵌 migrate()）
│   ├── tasks/                    # 任务即文档：task.md frontmatter 解析/回写
│   └── watcher/                  # fsnotify 任务目录监听 + 全量/增量同步
└── docs/api/openapi.yaml         # OAS 3.1 契约（唯一可信源，契约测试对账）
```

包组织红线：单文件 ≤300 行、单函数 ≤60 行、单包 ≤8 文件、禁 util/common/helper——由 `check-conventions.sh` pre-commit 强制执行。

## 15.3 业务服务模块（当前实现）

| 模块 | 包 | 职责 |
|-----|-----|------|
| 任务管理 | `api/tasks.go` + `tasks` | 任务创建（task.md 落盘）、状态流转、暂停/恢复 |
| 工作流 | `engine` + `pipeline` | 流水线状态机（每阶段审批闸；merge 经 Webhook 感知人工合并） |
| 熔断/重试 | `engine`（retry.go） | 连败熔断自动暂停 + notify；RetryLoop 退避自动重跑；启动回收僵尸 running |
| KB 检索 | `kb` | REST 窄接口代理 PieKBS（grounding off/warn/enforce） |
| Agent | `agent` | pi 子进程调用、阶段报告落盘、PATH 注入平台脚本 |
| 审计 | `store`（work_log） | hash 链流水写入；`verify-log` 校验 |
| 认证 | `authn` + `api/auth.go` | 登录、token 校验、角色注入 |
| 执行节点 | 规划中 | executor 登记/心跳/派发未实现（registry/executors.yaml 仅声明） |
| 工作报告 | 规划中 | work_log 聚合 → LLM 摘要未实现 |
| 自我升级 | 规划中 | 执行日志分析、Prompt/Skill 热更新未实现 |

## 15.4 路径与环境（基于工作用户 home）

| 配置项 | 值 | 说明 |
|-----|-----|------|
| home | `/home/dev` | 开发环境基础目录 |
| Worktree 根 | `~/wt` | 任务物理隔离目录 |
| 数据 | `~/data/control.db` | SQLite 运行时库（WAL） |
| 日志 | `~/logs` | 服务运行日志 |
| 编排配置 | `~/control-center/orchestration` | pipeline.yaml（热加载） |
| 注册表 | `~/control-center/registry` | repos.yaml / executors.yaml |
| 运行时配置 | `~/control-api.yaml` | server/db/llm/paths/agent；首次启动生成，0600 |

## 15.5 REST 接口概览（以 docs/api/openapi.yaml 为唯一可信源）

| 端点 | 说明 | 状态 |
|-----|------|------|
| `GET /actuator/health` | 探活（契约豁免） | 已实现 |
| `POST /api/auth/login` | 登录发 token（24h） | 已实现 |
| `GET/POST /api/tasks`、`POST /api/tasks/{id}/action` | 任务查询/创建/干预（approve/reject/pause/resume） | 已实现 |
| `GET /api/approvals/pending` | 待审批队列（按角色） | 已实现 |
| `GET /api/audit` | work_log 检索 | 已实现 |
| `GET /api/findings` | 发现问题一览（解析 control-center FINDINGS.md） | 已实现 |
| `GET /api/kb/search` | KB 检索（代理 PieKBS，Bearer 认证） | 已实现 |
| `GET /api/openapi.yaml` | 契约自指（供 Scalar 文档页） | 已实现 |
| `POST /api/webhooks/merge-event` | 合并回传（独立密钥 HMAC） | 已实现 |
| `/api/repos`、`/api/tests`、`/api/reports`、executor 心跳/领取/回传 | 见 10/14 章 | 规划中 |

## 15.6 外部集成

| 集成 | 方式 | 用途 |
|-----|------|------|
| SQLite | `modernc.org/sqlite`（WAL，busy_timeout 5s，immediate 事务） | 任务索引/审批/work_log/会话 |
| PieKBS | REST `/api/search` + MCP `/mcp`（127.0.0.1:8766） | 知识检索与 grounding（FINDING-016） |
| LiteLLM | OpenAI 兼容 REST | 模型统一出口（当前网关不可达为已知阻塞，FINDING-017；pi 直连模型配置可用） |
| Git | 本地 git CLI | Worktree、分支、提交操作 |
| 代码仓库 | 各仓库 OpenAPI | 分支/MR/Webhook/状态查询（规划中） |
| CLI Agent | pi 子进程（print 模式，超时可配 `agent.timeout_sec`） | 编码、测试、文件操作执行 |

## 15.7 与编排配置/注册表的关系

- 流水线状态机、Prompt、Skill 定义存于控制中心仓库 `orchestration/`；仓库/执行节点注册表存于 `registry/`，均非服务器代码
- 服务器启动时加载 pipeline.yaml 并热加载（mtime+size 比对）；merge 阶段强制 `approval: required`，违规拒绝加载；`check` 子命令同样校验（FINDING-035）

## 15.8 与 Linux 权限管理的关系

- Agent 指令以工作用户身份经子进程执行，见 [16 Linux 权限管理](16-linux-permissions.md)
- 执行节点拆分（构建/测试外移 executor）为规划中，见 10 章
