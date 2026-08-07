# 项目级 Agent 架构文档

> 面向内网部署的单人 AI Agent 平台：**单用户、多任务并行**，覆盖设计开发控制、任务自动流转执行、GitLab 人工合并与操作追溯的完整闭环。
> 架构形态：**一个独立控制中心仓库**（设计/控制文档 + 编排配置 + **registry 注册表**，无代码）+ **多个代码仓库**（平台实现 `control-api/control-web/control-db` + 业务代码，Git Worktree 多分支管理）。**db/backend/frontend 均在代码仓库中**，**仅内部设计（详细设计）与代码同库**，概要/外部设计集中于控制中心。

## 文档索引

| 编号 | 文档 | 内容 |
|-----|------|------|
| 00 | [设计原则](00-principles.md) | 核心原则：AI 驱动执行、单人多任务、Git 唯一可信源（含注册表）、数据内网 AI 外接受控、操作可追溯 |
| 01 | [总体架构](01-overview.md) | 控制中心仓库 + 多代码仓库总体架构与集成对象 |
| 02 | [多代码仓库与分支管理](02-branch-worktree.md) | Git Worktree 多分支策略：main / dev / release / feature |
| 03 | [设计开发控制 & 文档管理](03-doc-management.md) | 概要/外部设计集中于控制中心、**内部设计与代码同库**、OpenAPI 集成、RAG 任务管理 |
| 04 | [AI 网关层](04-ai-gateway.md) | 消费企业内 LiteLLM 代理（api.anthropic.com + ghe.com 企业版），模型路由/降级/预算优化 |
| 05 | [Agent 编排层](05-orchestration.md) | Java Spring 任务管理、流水线状态机（自动流转）、多任务并发、自我升级 |
| 06 | [Web 管理端](06-web.md) | React + PrimeReact + Vite：多任务看板、任务干预、Diff 合并、日志检索 |
| 07 | [工作流](07-workflows.md) | 功能开发流水线、Bug 修复、自我升级三类工作流 |
| 08 | [数据模型](08-data-model.md) | MySQL DDL：任务、工作记录、文档索引、工作报告表（注册表在 Git，不在 MySQL） |
| 09 | [数据流示例](09-data-flow.md) | 功能追加全流程端到端示例 |
| 10 | [内网部署](10-deployment.md) | docker-compose 模拟综合测试环境（非生产形态）、编排/执行节点拆分、executor 代理（复用办公 PC） |
| 11 | [安全与合规](11-security.md) | 风险清单与对策、追溯合规 |
| 12 | [实施路径](12-roadmap.md) | 三期交付计划与里程碑 |
| 13 | [仓库模板](13-repo-template.md) | 控制中心仓库（含 registry/）、平台/业务代码仓库模板、提交/MR/pre-commit 规范 |
| 14 | [多仓库管理](14-multi-repo.md) | Git 注册表（repos.yaml）、统一接入、跨仓库任务、配额 |
| 15 | [服务器模块](15-server-module.md) | control-api 后端模块：包结构、业务服务、WSL 路径、REST/集成 |
| 16 | [Linux 权限管理](16-linux-permissions.md) | 单人权限模型：owner / agent 双用户隔离、出站白名单、执行节点权限 |
| 17 | [客户端/服务端设计](17-client-server-design.md) | 单用户能力域 API；单工作台 Web 客户端 |

## 技术栈速览

- **前端**：React + PrimeReact + Vite + Nginx（内网）
- **后端**：Java Spring Boot（Spring MVC + Data JPA/MyBatis + Security + Scheduling）
- **数据库**：MySQL（任务/日志等运行时数据；**注册表在 Git registry/**，DDL/DML 版本化管控）
- **执行层**：pi.dev 核心工具（Earendil Pi，MIT；RPC 模式对接 control-api，原生 Skills/Extensions）+ Git Worktree（main/dev/release/feature）+ executor 执行节点
- **AI**：消费企业内 LiteLLM 代理（不重复部署，api.anthropic.com + ghe.com 企业版）；Milvus（向量库）
- **文档**：概要/外部设计集中于控制中心；**仅内部设计随业务代码同库**，Git 版本化，MR 变更
- **集成**：代码仓库 OpenAPI + Webhook + RAG
