# 10 内网部署方案

> **定位声明**：本章 docker-compose 为**模拟综合测试环境**，用于开发联调与集成测试，**不是生产部署形态**。生产部署架构（目标主机/VM、HA、备份/DR、网络分区）另行定义，一期可先按单机生产化要点落地（systemd 托管、数据备份、日志轮转）。

## 测试环境边界（仅限测试环境的妥协项）

| 配置项 | 测试环境取值 | 生产环境要求 |
|-------|-------------|-------------|
| MySQL 连接 | `useSSL=false` | 启用 TLS |
| 密钥管理 | 明文环境变量（`${DB_PASSWORD}`） | Vault/密钥管理服务引用，不落明文 |
| 镜像版本 | `mysql:8.0` / `redis:7-alpine` / `milvus:latest` | 全部锁定具体版本号，禁止 `latest` |
| 目录挂载 | `/home/dev/wt`、`/home/dev/repos`（WSL home） | 生产专用数据卷，路径不绑定个人 home |
| 资源分配 | 单容器最小配置（见下表） | 按容量评估扩容，API/Worker 支持多副本 |
| 数据可靠性 | 本地卷，无备份 | MySQL/Milvus 定期备份 + 恢复演练 |

## docker-compose（模拟综合测试环境）

```yaml
# docker-compose.yml（模拟综合测试环境）
version: '3.8'
services:
  web:
    image: internal-control-web:latest      # React + PrimeReact + Vite 构建产物 + Nginx
    ports: ["80:80"]

  api:
    image: internal-control-api:latest      # Java Spring Boot（控制中心后端）
    environment:
      - DB_URL=jdbc:mysql://mysql:3306/control?useSSL=false&characterEncoding=utf8mb4
      - DB_USER=control
      - DB_PASSWORD=${DB_PASSWORD}
      - LITELLM_ENDPOINT=http://litellm.internal:4000
      - VECTOR_DB_ENDPOINT=http://milvus:19530
    volumes:
      - /home/dev/wt:/wt            # Worktree 根目录（WSL home）
      - /home/dev/repos:/repos      # 代码仓库裸仓库/工作副本
    depends_on: [mysql, redis]

  worker:
    image: internal-control-worker:latest   # Spring 调度/异步任务（RAG 索引、定时任务）
    environment:
      - DB_URL=jdbc:mysql://mysql:3306/control?useSSL=false&characterEncoding=utf8mb4
    depends_on: [api, redis, mysql]
    deploy: { replicas: 2 }

  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_DATABASE=control
      - MYSQL_USER=control
      - MYSQL_PASSWORD=${DB_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d   # DDL/DML 初始化脚本

  redis:
    image: redis:7-alpine

  milvus:
    image: milvusdb/milvus:latest
    volumes: [milvus-data:/var/lib/milvus]
```

> 控制中心仓库仅含设计/控制文档与编排配置（不含代码）；平台与业务代码（db/backend/frontend）位于各代码仓库（control-api、control-web、control-db、业务仓库），通过其 REST API / OpenAPI 对接与部署，不在本编排内。**不部署 Ollama、不部署 LiteLLM**：AI 消费企业内已启动的 LiteLLM 代理（接入 api.anthropic.com + ghe.com 企业版），平台仅以 `LITELLM_ENDPOINT` 直连；代理侧配置见 `04-ai-gateway.md`。

> **容器边界**：control-api 容器仅做编排调度，**不直接创建 Agent 进程**（容器内无法切换宿主 Linux 用户）。本机 Agent 任务由宿主机 systemd 服务（`agent` 身份）执行，构建/测试由执行节点（executor / CI Runner）执行，见 [16.7](16-linux-permissions.md#167-执行节点executor-pc权限模型)。

## 节点拆分：编排节点 vs 执行节点

测试环境的 compose 把编排与执行合并在单机，**生产必须拆开**。核心原因：Agent 任务的真正负载不是 CLI 本身，而是其触发的构建与测试（Maven/Gradle 编译 ~1-2C+1-2GB、vitest ~1-2GB、Playwright 每个浏览器实例 0.5-1GB），单个"编码+单测+前端测试"并发槽位约 **2-4C / 4-8GB**，与控制服务混部会拖垮编排稳定性。

| 节点 | 运行内容 | 资源特征 | 扩展方式 |
|-----|---------|---------|---------|
| **编排节点** | control-api、worker（调度/RAG 索引）、MySQL、Redis、Milvus、Nginx | 稳定负载，8C/16GB 起步 | control-api 无状态多副本；MySQL/Milvus 可再拆独立主机 |
| **执行节点**（CI Runner 集群） | Maven/Gradle 构建、vitest、Playwright、静态检查（各仓库 `ci/` 脚本） | 突发重负载，按并发槽位 2-4C/4-8GB | 按槽位横向加 Runner，天然弹性 |

分工约定：

1. **编排节点只做编排与 Git 操作**：Agent 在 Worktree 内编码、提交、push；**不在编排节点跑构建/测试**
2. **构建/测试经 CI 执行**：push 触发仓库 CI（GitLab Runner 等），结果经 Webhook 回传控制中心，驱动瀑布状态机流转（复用 13.2 的 `ci/` 脚本与 Webhook 通道）
3. **配额即限流**：全局并发 Worktree 数 N（14.6）按执行节点总槽位设定，而不是按编排节点容量；夜间时段策略（见下节）天然将重负载任务错峰
4. **本地轻量校验例外**：Agent 提交前的快速单测/pre-commit 可在编排节点执行，但需计入并发配额

```
编排节点（8C/16GB）                执行节点集群（N × 2-4C/4-8GB）
┌───────────────────────┐        ┌────────────────────────────┐
│ control-api / worker   │  push  │ 执行节点 #1..#N              │
│ MySQL / Redis / Milvus │ ─────→ │ Maven · vitest · Playwright │
│ ~/repos ~/wt（Git）    │ ←───── │ 结果 Webhook 回传           │
└───────────────────────┘  webhook └────────────────────────────┘
（执行节点 = CI Runner，或下节 executor 代理）
```

## 执行节点实现：executor 代理（无 CI 产品，复用办公 PC）

企业无 GitLab Runner 等 CI 产品时，以**自研 executor 代理**实现执行节点：在每台被征用的 Windows PC 的 WSL 中运行一个轻量 executor（`control-worker` 精简版），**连接服务器取代码、本地执行、结果回传**。

```
PC-01 WSL ─┐
PC-02 WSL ─┼── 注册/心跳 ──→ control-api（编排节点，可占其中 1 台 PC）
...        │ ←── 任务下发（REST 长轮询）──
PC-51 WSL ─┘ ── git fetch 任务分支 → 本地执行 ci/ 脚本 → 结果回传
```

### 执行流程

1. **登记**：在控制中心仓库 `registry/executors.yaml` 声明执行节点（Git 版本化，MR 生效），executor 启动时按登记身份上报心跳；机器关机即离线，调度器自动跳过

```yaml
# control-center/registry/executors.yaml
executors:
  - executor_id: pc-01
    tags: [java17, node20, pnpm, playwright]   # 能力标签
    slots:
      day: 1                             # 工作日白天轻量槽位（≤4GB，低优先级）
      night: 2                           # 夜间 02:00-06:00 全量槽位（见下表）
    token_ref: env:EXECUTOR_TOKEN_PC01   # 认证 token 引用，不存明文
```

2. **取代码**：executor 从 `git.internal`（或编排节点 `~/repos`）`git fetch` 任务分支到本地临时目录，**不复制整个 Worktree**
3. **执行**：本地运行仓库 `ci/` 白名单脚本（Maven / vitest / Playwright），带超时与资源限额
4. **回传**：测试结果、日志、状态哈希回传 control-api 写入 `work_log`，驱动流水线状态机流转；临时目录执行完即清理

### 与时段联动

| 时段 | 每台 PC 策略 | 说明 |
|-----|-------------|------|
| 工作日白天 | 限 1 个轻量槽位（≤4GB，低优先级调度） | 不影响 PC 使用者 |
| 夜间 02:00-06:00 | 放开 2 个全量槽位 | Playwright 回归等重任务排此时段（由调度器按 executors.yaml `slots.day` / `slots.night` 下发） |

> 参考：51 台 16GB PC，白天可得 ~50 个轻量槽位、夜间 ~100 个全量槽位，远超并发配额 N；全局 N 仍按 14.6 由控制中心统一限流。

### 约束

- executor 以 WSL 非 root 用户（`agent`）运行，出站同样只放行 control-api / `git.internal` / LiteLLM 模型端点（16.7）
- 代码副本为公司内网 PC 上的临时目录，执行完删除；机密仓库在 `repos.yaml` 标记 `executor_allowed: false`，仅允许编排节点执行（14.2）
- 执行身份与审计绑定 `agent@{executor_id}`（16 章），PC 使用者账号不参与
- pi 及技能/扩展包经内网源安装（`pi install git:` 指向内网 Git 或 npm 内网镜像），不直连外网

## 单机备份（最小口径）

编排节点为单机，备份策略利用"Git 即权威源"：

| 数据 | 备份方式 | 说明 |
|-----|---------|------|
| control-center（文档/编排/registry） | 推送 `git.internal` | Git 即备份，含全部历史 |
| `~/repos` 各仓库 | 推送 `git.internal` | 同上 |
| MySQL（task/work_log/work_report） | 每日 `mysqldump` → `~/data/backup/`，保留 14 天 | cron/systemd timer 执行 |
| Milvus 向量 | **不备份** | 可由 Git 内容重建（重跑 RAG 全量索引） |

## 资源分配参考（测试环境）

| 服务 | CPU | 内存 | 说明 |
|-----|-----|------|------|
| Spring API + Worker | 1核 | 1.5GB | 控制中心后端（测试环境合并执行负载，生产拆分见上节） |
| MySQL + Redis + Milvus | 0.5核 | 1GB | 数据层 |
| Web (Nginx) | 共享 | 共享 | 静态页 |
| LiteLLM 代理 | 企业既有 | 企业既有 | 消费端，不重复部署（见 04） |
