# 15 服务器模块设计（control-api）

## 15.1 概述

服务器模块即平台后端 `control-api`（Java Spring Boot），是系统唯一的后端服务，运行于 WSL Linux，负责任务编排、流水线状态机、多仓库管理、RAG、任务干预与操作追溯。运行路径全部基于 WSL home（`~/`）布局，见 [13-repo-template.md](13-repo-template.md#131-wsl-开发环境目录架构以-linux-home-为基础)。

```
Web (control-web, React) ──REST──→ control-api (Spring Boot) ──→ MySQL (control-db)
                                      │
                                      ├── LiteLLM ──→ Anthropic / GitHub Copilot（外接）
                                      ├── Milvus（RAG 向量）
                                      ├── Git 本地操作（~/repos、~/wt）
                                      ├── 控制中心仓库（orchestration/ + registry/，热加载）
                                      └── 各代码仓库 OpenAPI（分支/MR/Webhook）
```

## 15.2 服务器模块结构（Java 包结构）

```
control-api/                          # 服务器模块（代码仓库）
├── pom.xml
└── src/main/java/com/xxx/control/
    ├── ControlApplication.java
    ├── controller/                   # REST API 层
    │   ├── TaskController.java
    │   ├── RepoController.java
    │   ├── AuditController.java
    │   ├── RagController.java
    │   └── AgentController.java
    ├── service/                      # 业务服务（15.3）
    ├── repository/                   # Spring Data JPA
    ├── entity/                       # 实体（与 control-db DDL 对齐）
    ├── dto/                          # 请求/响应对象
    ├── scheduler/                    # 定时任务（Spring Scheduling / Quartz）
    ├── config/                       # Security、Web、数据源、LiteLLM、Milvus
    ├── common/                       # 通用工具、异常、审计切面
    └── agent/                        # CLI Agent（pi.dev）对接
```

## 15.3 业务服务模块

| 模块 | 包 | 职责 |
|-----|-----|------|
| 任务管理 | `service/task` | 任务创建、分解、状态流转、暂停/回退 |
| 工作流 | `service/workflow` | 流水线状态机（自动流转；合并经 Webhook 感知 GitLab 人工合并事件） |
| 多仓库 | `service/repo` | 注册表（`registry/repos.yaml`）加载与热更新、OpenAPI 接入、状态同步 |
| Worktree | `service/worktree` | Worktree 创建/回收/归档，配额控制 |
| RAG | `service/rag` | 文档/代码增量索引、检索、Milvus 对接 |
| Agent | `agent` | pi.dev 指令下发、执行结果回传 |
| 执行节点 | `service/executor` | executor 登记（`registry/executors.yaml`）、心跳/任务派发/结果回收（无 CI 产品时替代 CI Runner，见 [10 executor 代理](10-deployment.md)） |
| 审计 | `service/audit` | 工作记录写入（work_log）、状态哈希校验 |
| 工作报告 | `service/report` | 定时聚合 work_log → LLM 生成摘要 → work_report（日报/周报/任务报告） |
| 自我升级 | `service/selfupgrade` | 执行日志分析、Prompt/Skill 热更新 |

## 15.4 路径与环境（基于 WSL home）

| 配置项 | 值 | 说明 |
|-----|-----|------|
| WSL home | `/home/dev` | 开发环境基础目录 |
| 代码仓库根 | `~/repos` | control-api / control-web / control-db / 业务仓库 |
| Worktree 根 | `~/wt` | 任务物理隔离目录 |
| 数据卷 | `~/data/mysql`、`~/data/milvus` | 本地服务数据 |
| 日志 | `~/logs` | 服务运行日志 |
| 编排配置 | `~/control-center/orchestration` | 状态机/Prompt 配置（从控制中心仓库加载） |
| 注册表 | `~/control-center/registry` | repos.yaml / executors.yaml（热加载） |

```yaml
# application.yml（片段）
control:
  home: /home/dev
  repos-root: /home/dev/repos
  worktree-root: /home/dev/wt
  orchestration-dir: /home/dev/control-center/orchestration
  registry-dir: /home/dev/control-center/registry
```

## 15.5 REST 接口概览

| 端点 | 说明 |
|-----|------|
| `GET /actuator/health` | 健康检查（免认证），供 executor 初始化预检与负载探活 |
| `GET/POST /api/tasks` | 任务查询/创建，含状态流转 |
| `POST /api/tasks/{id}/action` | 任务干预：暂停 / 回退 / 批注修正（合并在 GitLab 人工执行，平台无合并接口） |
| `GET /api/repos` | 注册表仓库列表与状态查询（注册走 `registry/repos.yaml` MR） |
| `GET /api/audit` | 工作记录（work_log）检索 |
| `GET/POST /api/reports` | 工作报告查询（work_report，service/report 定时生成） |
| `GET /api/rag/search` | RAG 语义检索 |
| `POST /api/agents/{agentId}/run` | 下发 Agent（pi.dev）指令，返回结果 |
| `POST /api/agents/{executorId}/heartbeat` | executor 心跳（在线状态、空闲槽位），离线自动剔除；executorId 须已在 `registry/executors.yaml` 登记 |
| `POST /api/agents/{executorId}/poll` | executor 长轮询领取执行任务（按标签/槽位匹配） |
| `POST /api/agents/{executorId}/report` | executor 回传执行结果（日志、测试报告、状态哈希），写 `work_log` 驱动状态机 |
| `POST /api/repos/{key}/webhook` | 代码仓库 Webhook 入口 |

## 15.6 外部集成

| 集成 | 方式 | 用途 |
|-----|------|------|
| MySQL | JDBC（control-db 提供 DDL） | 任务、工作记录、文档索引元数据 |
| Milvus | gRPC/REST | RAG 向量检索 |
| LiteLLM | OpenAI 兼容 REST | 统一路由外接 Anthropic / GitHub Copilot，不跑本地模型 |
| Git | 本地 git 命令（JGit/CLI） | Worktree、分支、提交操作 |
| 代码仓库 | 各仓库 OpenAPI（GITLAB 等） | 分支/MR/Webhook/状态查询 |
| CI Runner | 仓库 CI（Webhook 触发 + 结果回传） | 构建/测试在执行节点运行，编排节点不跑重负载（见 [10 节点拆分](10-deployment.md#节点拆分编排节点-vs-执行节点)） |
| CLI Agent | pi.dev **RPC 模式**（JSON over stdin/stdout，面向非 Node 集成）；一次性任务用 print/JSON | 编码、测试、文件操作执行 |

## 15.7 与编排配置/注册表的关系

- 流水线状态机、Prompt、Skill 定义存于控制中心仓库 `orchestration/`；仓库/执行节点注册表存于 `registry/`，均非服务器代码
- 服务器启动时加载 `~/control-center/orchestration/` 与 `~/control-center/registry/`，支持热加载（免重启）；**生效顺序：MR → GitLab 合并 → 热加载**（07.3）

## 15.8 与 Linux 权限管理的关系

- Agent 指令由控制中心以**专用 Linux 系统用户 `agent`**（本机与执行节点同一身份）下发执行，见 [16 Linux 权限管理](16-linux-permissions.md)
- 服务器校验"执行身份 + 目标路径授权"匹配后才派发任务
- control-api 容器不直接创建进程：本机任务经宿主机 systemd 服务执行，构建/测试经 executor / CI Runner 执行节点运行（见 [16.7](16-linux-permissions.md#167-执行节点executor-pc权限模型)）
