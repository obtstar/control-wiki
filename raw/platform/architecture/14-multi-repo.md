<!-- kb-mirror: upstream=control-center/docs/architecture/14-multi-repo.md sha256=cdb1c691c66b2ba3c745f1b8b04e7674385f49568b4de1baf00e5c21bf040b44 （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

---
status: 部分实现（注册表/命名规范/任务流转已生效；跨仓库调度/Webhook 自动接线/KB 聚合检索为规划中）
last_verified: 2026-08-17
---

# 14 多仓库管理设计

平台以**一个独立控制中心仓库 + N 个代码仓库**为形态（代码仓库含平台实现 `control-api/control-web/control-db` 与业务代码仓库）。

> **边界声明**：`control-center` 为**独立仓库**，仅承载设计开发控制文档、任务编排配置与**注册表（registry/）**（**无代码实现**），**不纳入**注册表，也不作为被调度/执行对象；本文件所述的多仓库管理仅针对**代码仓库**（平台实现 + 业务）。

本文定义控制中心对多个代码仓库的统一注册、接入、调度、检索与追溯。

## 14.1 管理模型

```
                     ┌─────────────────────────────────────────┐
                     │  控制中心（registry/ 注册表 + 调度）       │
                     │  注册 · 配额 · Webhook · KB 聚合（规划中）  │
                     └──────┬──────────────────────┬────────────┘
                            │ OpenAPI / Webhook     │
               ┌────────────┴──────┐    ┌───────────┴───────────┐
               ▼                   ▼    ▼                       ▼
         ┌────────────┐     ┌────────────┐     ┌────────────┐
         │ repo-a     │     │ repo-b     │ ... │ repo-n     │
         │ billing-   │     │ billing-   │     │ order-     │
         │ core       │     │ reports    │     │ service    │
         └────────────┘     └────────────┘     └────────────┘
          main/dev/release/feature（每仓库独立）
```

## 14.2 仓库注册表（Git 文件：registry/repos.yaml）

注册表**不存 MySQL**，以版本化文件存于控制中心仓库 `registry/repos.yaml`，Git 即权威源（变更走 MR，天然带历史与回滚）：

```yaml
# control-center/registry/repos.yaml
repos:
  - repo_key: billing-core
    name: 计费核心
    path: wt/billing-core/dev      # 工作区位置（相对工作用户 home）：平台仓为顶级目录
                                   # （如 control-api），业务仓为 wt/ 前缀常驻工作区；
                                   # setup-env 按此落盘，不可由 repo_key 推导
    git_url: ssh://git@git.internal/billing/billing-core.git
    api_type: GITLAB                # GITLAB / GITHUB / GITEA
    api_endpoint: http://git.internal/api/v4
    token_ref: env:GIT_TOKEN        # 密钥引用，不存明文
    openapi_ref: openapi/openapi.yaml  # OpenAPI 契约路径，供 KB 采集（规划中）
    default_branch: dev
    max_worktrees: 3                # 仓库级并发 Worktree 配额
    executor_allowed: true          # false = 机密仓库，仅编排节点执行，不分发 executor
    disabled: false                 # 逻辑停用，保留历史
  - repo_key: billing-reports
    name: 计费报表
    path: wt/billing-reports/dev
    git_url: ssh://git@git.internal/billing/billing-reports.git
    api_type: GITLAB
    api_endpoint: http://git.internal/api/v4
    token_ref: env:GIT_TOKEN
    default_branch: dev
    max_worktrees: 2
    executor_allowed: true
    disabled: false
```

control-api 启动时读取 `repos.yaml`（watcher 监听任务目录已通；注册表热加载为规划中）。运行时状态缓存于内存，不落库（Redis 未接入）。

## 14.3 管理能力

| 能力 | 说明 |
|-----|------|
| 仓库注册 | 编辑 `registry/repos.yaml` 提 MR，合并后 control-api 热加载 |
| 统一接入 | 通过仓库 OpenAPI 统一操作：分支创建、MR、Webhook、状态查询（规划中） |
| 跨仓库任务 | 一个任务可关联多个仓库，分别创建 Worktree，按依赖顺序执行（规划中；当前任务单 repo_key） |
| 跨仓库检索 | KB 聚合所有启用仓库知识 + OpenAPI 契约（规划中；当前 KB 仅平台级 control-wiki） |
| 状态同步 | merge-event Webhook 已实现（HMAC 独立密钥）；分支/MR 事件回传为规划中 |

## 14.4 命名规范（多仓库统一）

| 项 | 规范 | 示例 |
|-----|------|------|
| 仓库名 | `{system}-{module}` | `billing-core`、`order-service` |
| 常驻分支 | `main` / `dev` / `release/{yyyymm}` | `release/202608` |
| 任务分支 | `feature/{task-id}-{name}`、`bugfix/{task-id}-{name}` | `feature/TASK-001-ai-report` |
| Worktree | `~/wt/{repo_key}/{task-id}-{type}-{name}` | `~/wt/billing-core/TASK-001-feature-ai-report` |

## 14.5 跨仓库任务流程

```
TASK-002：新增计费报表接口（修改 billing-core，依赖 order-service 契约）
1. 控制中心解析任务 → 检索 KB 定位依赖（规划中） → 确认影响仓库
2. 为 billing-core 创建 feature/TASK-002-xxx + Worktree
3. order-service 仅只读（检索其 OpenAPI 契约，不改动）
4. billing-core 编码完成 → MR → 用户在 GitLab 人工合并 release/dev
5. 若 order-service 需契约变更 → 自动生成独立子任务走同流程
6. 两仓库合并事件经 Webhook 同步，任务进入测试验证
```

## 14.6 资源与并发控制

- 仓库级配额：`max_worktrees` 限制单仓库活跃 Worktree 数
- 全局配额：同时活跃 Worktree ≤ N（跨仓库总和，防资源耗尽）
- 调度策略：任务入队 → 按仓库配额分配 → Worktree 生命周期由控制中心统一回收
- 超限行为：任务排队等待，不抢占已分配 Worktree
- **与执行节点槽位对齐**：一个执行中任务同时占用 1 个 Worktree 配额与 1 个 executor/CI 槽位（见 10 章），调度按两者中较紧的约束限流

## 14.7 追溯

- **变更历史**：注册表的增删改即 Git 提交历史，天然可回溯、可 `git revert`
- **运行审计**：`work_log` 记录 `task_id`/`stage`/`action`/`operator`/`model`/`detail`，关联任务与执行身份（branch/worktree/commit 字段为规划中）
- **删除保护**：仓库仅可 `disabled: true` 停用，不从注册表物理删除，保留追溯链条

## 14.8 仓库接入流程

```
1. 用户编辑 registry/repos.yaml，新增仓库条目，提交 MR 并合并
2. control-api 读取注册表条目（热加载与连通性握手为规划中）
3. Webhook 接线（merge-event 已实现；push/MR 事件为规划中）
4. KB 全量采集（代码 + OpenAPI 契约，规划中）
5. 仓库启用 → 可分配任务
```
