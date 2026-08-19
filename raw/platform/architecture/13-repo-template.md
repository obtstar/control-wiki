<!-- kb-mirror: upstream=control-center/docs/architecture/13-repo-template.md sha256=9171efdc268be0f5dcec7208415fb204971e2dbd60bf10de77b1774f14dbd84c （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

---
status: 部分实现（目录布局/分支约定已生效；模板仓/OpenAPI 自动建分支/RAG 钩子为规划中）
last_verified: 2026-08-17
---

# 13 仓库模板设计

本文定义三类仓库的标准模板与规范：**控制中心仓库模板**（独立，无代码实现）、**平台代码仓库模板**与**业务代码仓库模板**（多个代码仓库）。

> **边界**：
> - **控制中心仓库**（独立）：仅承载**设计开发控制文档**（概要/外部设计、需求）与**任务编排配置**，**不含 db/backend/frontend 代码实现**。
> - **代码仓库**（多个）：承载**全部代码实现**（平台自身 + 业务代码）——**db、backend、frontend 均在代码仓库中，不在控制中心**；**仅内部设计（详细设计）与代码同库**，`docs/design/internal/` 随本仓库代码版本化。

---

## 13.1 WSL 开发环境目录架构（以 Linux home 为基础）

控制中心与所有代码仓库部署在 **WSL Linux 的 home 目录**下（例 `/home/dev`），作为本地开发环境的基础布局。控制中心仓库仅含设计/控制文档与编排配置，**不含任何代码实现**。

### 目录结构（WSL home 视角）

```
~ (WSL Linux home，例如 /home/dev)
├── control-center/                # 控制中心仓库（独立，无代码实现）
│   ├── README.md                  # 项目说明 + 文档索引
│   ├── .gitignore
│   ├── .pre-commit-config.yaml
│   ├── docs/                      # 设计开发控制文档（集中，Git 版本化）
│   │   ├── design/
│   │   │   ├── overview/          # 概要设计
│   │   │   │   └── system-overview.md
│   │   │   └── external/          # 外部设计
│   │   │       └── api-contract.md
│   │   ├── requirements/          # 需求文档
│   │   │   └── REQ-0000-template.md
│   │   └── architecture/          # 本架构文档集
│   ├── orchestration/             # 任务编排配置（版本化）
│   │   ├── prompts/               # Prompt 版本化
│   │   ├── skills/                # Skill 定义（SKILL.md 格式，openskills 管理，第三方技能 vendor 入库）
│   │   └── workflows/             # 流水线状态机配置
│   └── registry/                  # 注册表（版本化，Git 即权威源）
│       ├── repos.yaml             # 仓库注册表（14.2）
│       └── executors.yaml         # 执行节点登记（10）
├── repos/                         # 代码仓库工作副本（db/backend/frontend 均在此）
│   ├── control-api/               # 平台后端（Go + stdlib mux）
│   ├── control-web/               # 平台前端（React + PrimeReact + Vite）
│   ├── control-db/                # 平台数据库（占位仓；当前 SQLite DDL 内嵌 control-api，规模化迁此）
│   ├── billing-core/              # 业务仓库（示例）
│   └── ...
├── wt/                            # Git Worktree 根（任务物理隔离）
│   └── {repo}/{task-id}-{type}-{name}
├── data/                          # 本地服务数据
│   └── control.db                 # SQLite 运行时库（WAL）
├── logs/                          # 服务运行日志
└── scripts/                       # 初始化/运维脚本
```

> 控制中心**不含** `db/`、`backend/`、`frontend/`、`infra/`——它们都在 `~/repos/` 代码仓库中（见 13.2），由控制中心通过多仓库管理调度。Worktree 统一位于 `~/wt/`，与 `~/repos/` 工作副本物理隔离。

### 目录职责约束

| 目录 | 职责 | 变更入口 |
|-----|------|---------|
| `~/control-center/docs/design/overview/` | 概要设计（集中），KB 索引源（规划中） | MR（单人自审合并） |
| `~/control-center/docs/design/external/` | 外部设计（集中），KB 索引源（规划中） | MR（单人自审合并） |
| `~/control-center/docs/requirements/` | 需求文档，KB 索引源（规划中） | MR |
| `~/control-center/orchestration/` | 状态机/Prompt 配置 | MR + 用户确认 |
| `~/control-center/registry/` | 仓库/执行节点注册表 | MR + 用户确认 |
| `~/repos/` | 全部代码仓库（平台 + 业务） | 仓库内 MR |
| `~/wt/` | Worktree 临时目录，生命周期由控制中心管理 | 自动创建/回收 |
| `~/data/` `~/logs/` | 服务数据与日志 | 运维管理，不纳入 Git |

---

## 13.2 代码仓库模板（多个代码仓库：平台实现 + 业务代码）

所有代码实现（**db、backend、frontend**）都在代码仓库中，统一模板。代码仓库分两类，均受控制中心多仓库管理：

| 类型 | 示例 | 说明 |
|-----|------|------|
| 平台后端 | `control-api` | Go + stdlib mux：任务、状态机、KB 检索 |
| 平台前端 | `control-web` | React + PrimeReact + Vite：看板/干预/合并/日志 |
| 平台数据库 | `control-db` | 占位仓；SQLite DDL 内嵌 control-api store.go，规模化迁此 |
| 业务代码 | `billing-core`、`billing-web` | 业务后端/前端，含自身 DDL/DML |

### 目录结构

```
control-api/  (或 billing-core/)
├── README.md                     # 项目说明（指向控制中心概要/外部设计索引）
├── .gitignore
├── .editorconfig
├── .pre-commit-config.yaml       # 统一 pre-commit（KB 索引钩子规划中；当前为 check-conventions.sh）
├── .gitlab/                      # 或 .github/
│   └── merge_request_templates/default.md
├── docs/design/internal/         # 内部设计（详细设计），与本仓库代码同库
│   └── module-detail.md
├── src/                          # 代码实现（后端 Go / 前端 React / 脚本）
│   └── main/                     # 按语言/框架约定分层
├── tests/                        # 测试
├── db/                           # 本代码仓库对应的 DDL/DML（如适用）
│   └── ddl/
│       └── V001__init_schema.sql
├── ci/                           # 构建/测试/静态检查脚本
└── openapi/                      # 接口契约（供控制中心 KB 采集，规划中）
    └── openapi.yaml
```

> 内部设计与代码同 MR 评审；本仓库 DDL/DML 随代码同 MR 提交。概要/外部设计仍在控制中心集中管理。
>
> Node 仓库（如 control-web）统一使用 **pnpm**：多 worktree 并行任务共享全局 store（硬链接），避免每个 worktree 重复安装 `node_modules`；executor 能力标签含 `pnpm` 方可领取 Node 任务。

### 分支与 Worktree 约定

- 常驻分支：`main`（生产）、`dev`（阶段集成）、`release/{yyyymm}`（月度升级，从 main 切出）
- 任务分支：`feature/{task-id}-{name}`、`bugfix/{task-id}-{name}`
- 每任务一个独立 Worktree：`~/wt/{repo}/{task-id}-{type}-{name}`，checkout 到任务分支
- 禁止在 `main`/`dev` 直接编码，全部经 MR
- 内部设计与代码同分支、同 MR 提交

### 触发与集成

| 事件 | 动作 |
|-----|------|
| MR 合并到 `dev` | Webhook → 任务状态流转（已实现 merge-event）；增量 KB 索引为规划中 |
| pre-commit | 本地静态检查（check-conventions.sh 已装 control-api/control-web）；KB 索引通知为规划中 |
| 分支推送 | 经仓库 OpenAPI 通知控制中心同步状态（规划中） |

---

## 13.3 通用规范

### 提交信息（Conventional Commits）

```
<type>(<scope>): <subject>

feat(design): 新增外部设计 API 契约
fix(scheduler): 修复任务调度限流判断
docs(db): 新增 V002 任务表索引
refactor(rag): 拆分增量索引任务
```

| type | 说明 |
|-----|------|
| `feat` / `fix` | 功能 / 缺陷修复 |
| `docs` | 设计文档、README |
| `db` | DDL/DML 变更（触发数据库评审） |
| `refactor` / `test` / `ci` / `chore` | 重构 / 测试 / CI / 杂项 |

### MR 模板（默认）

```markdown
## 关联任务
- [ ] 任务号：TASK-20260802-001

## 变更类型
- [ ] feat / fix / docs / db / refactor / test / ci

## 变更说明
（背景、方案、影响范围）

## 数据库变更
- 涉及表：task / work_log
- DDL/DML 脚本：db/ddl/V002__xxx.sql（如涉及）

## 测试
- [ ] 单元测试
- [ ] 集成测试 / 回归测试
- [ ] 静态检查通过

## 设计文档
- [ ] 概要/外部设计已同步更新（控制中心 docs/design/）
- [ ] 内部设计已同步更新（本仓库 docs/design/internal/，如涉及）
```

### pre-commit 钩子（代码仓库示例）

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      # KB 增量索引钩子为规划中；当前以 check-conventions.sh 为统一 pre-commit
      - id: conventions
        name: 规模红线 + 静态检查
        entry: scripts/check-conventions.sh
        language: script
      - id: sql-lint
        name: SQL 脚本格式校验
        entry: ci/sql-lint.sh
        language: script
        files: '^db/'
```

---

## 13.4 模板创建流程

> `control-center` 为独立仓库，仅初始化一次，不纳入 `registry/repos.yaml`；代码仓库（平台实现 + 业务）创建后登记入 `registry/repos.yaml`。

1. 控制中心仓库由模板仓库 `template-control-center` 初始化（`git clone --template` 或模板工具），为独立仓库，内置 `docs/design/overview|external`、`docs/requirements`、`orchestration/`、`registry/` 模板
2. 平台代码仓库（`control-api` / `control-web` / `control-db`）与业务代码仓库（`{system}-{name}`）由 `template-code-repo` 初始化，内置 `docs/design/internal/` 内部设计模板，创建后在 `registry/repos.yaml` 中新增条目并合并（14.8）
3. 新建任务 → 控制中心调用仓库 OpenAPI 按模板创建分支 + Worktree
4. 分支策略、MR 模板、CI 配置随仓库初始化写入，禁止在任务中修改
