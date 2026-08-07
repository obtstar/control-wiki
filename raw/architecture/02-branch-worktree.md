# 02 多代码仓库 & Git Worktree 分支管理

系统管理**多个代码仓库**，每个仓库通过 **Git Worktree + 多分支策略** 实现任务的物理隔离与并行执行。

> 多仓库的注册、接入、配额与权限管理见 [14 多仓库管理](14-multi-repo.md)。

## 分支策略（四类分支）

| 分支 | 命名 | 用途 | 生命周期 |
|-----|------|------|---------|
| `main` | `main` | 生产主干，可发布基线 | 常驻 |
| `dev` | `dev` | **阶段性**集成分支，按开发阶段/迭代集成验证 | 阶段结束后归档/重建 |
| `release` | `release/{yyyymm}` | **月度升级分支，从 `main` 切出**，承接当月 feature 合并 | 月度发布合回 `main` 后归档 |
| `feature` | `feature/{task-id}-{name}` / `bugfix/{task-id}-{name}` | 任务/缺陷分支，**从 `main` 或 `dev` 切出** | 合并后删除 |

### 分支流转模型

```
                 （从 main 或 dev 切出）
feature/task-001 ─┐
feature/task-002 ─┼──→ release/{yyyymm} ──月度──→ main
bugfix/042 ───────┘        ▲
                           └── dev（阶段内集成验证，可选并入）
```

- **合入规则**：feature → `release/{当月}`（或阶段内先合 `dev` 集成验证）→ 月度发布时 release → `main`
- **合并方式**：**所有合并均在 GitLab 人工执行**（平台不执行合并）；合并前由平台做**本地预处理验证**（编码/测试/Diff 汇总，见下节）
- **回滚**：release 出问题时从 main 或上一 release 打补丁，禁止直接改 main

## Worktree 策略

- 每个任务在对应代码仓库创建独立 Worktree：`~/wt/{repo}/{task-id}-{type}-{name}`（WSL home 下）
- Worktree 与分支一一对应，checkout 到 `feature/{task-id}` 分支
- 并发限制：同时活跃 worktree ≤ N（防资源耗尽）
- 生命周期：任务合并完成后保留 7 天归档，然后清理
- 主工作区 `main`/`dev` 常驻，禁止在常驻分支直接编码

```
控制中心后端 ──→ git worktree add ~/wt/repo-a/TASK-001-feature-report feature/TASK-001
             ──→ git worktree add ~/wt/repo-b/BUG-042-fix-npe bugfix/BUG-042
```

## 变更与合并流程（Code Repo 侧）

1. 控制中心创建任务 → 在目标仓库创建 `feature/{task-id}` 分支（从 `main` 或 `dev` 切出）+ Worktree
2. Agent 在 Worktree 内编码、本地测试、commit
3. push 分支 → 通过仓库 OpenAPI 创建 MR，目标为 `release/{当月}`（或阶段集成分支 `dev`）
4. **本地预处理验证**：执行节点跑集成/回归/静态检查，报告与 Diff 汇总回传（MR 注释 + Web 端可见）
5. **用户在 GitLab 人工合并**（平台不执行合并）→ Webhook 回传合并事件 → 控制中心记录 `work_log`、流转任务状态
6. 合入后触发增量索引（Webhook）

## 执行 Agent 职责

- 终端 CLI Agent（**pi.dev 为核心工具**，Earendil Pi Coding Agent，MIT）
- **control-api 经 pi 的 RPC 模式（JSON over stdin/stdout）驱动 Agent**——该模式专为非 Node 集成设计，Java 后端可直接对接；一次性任务可用 print/JSON 模式（`pi -p`）
- pi 原生能力直接复用：**Skills**（SKILL.md 能力包，openskills 管理，见 07.3）、**Extensions**（TypeScript，含官方 permission-gate / protected-paths / sandbox 示例）、**会话树**（单文件存储，`/export` HTML 可归档为任务证据）
- 模型统一经企业内 LiteLLM 代理外接：`claude-sonnet` / `claude-opus`（Anthropic）+ `copilot-chat`（GitHub Copilot）；pi 的自定义 provider（models.json）配置见 [04 AI 网关](04-ai-gateway.md)
- 接收控制中心指令，操作 Worktree 与 Git
- 所有文件/Git 操作仅限分配的 Worktree 内执行；出站网络仅放行 LiteLLM 模型端点
- **执行节点（executor）例外口径**：在员工 PC 本地临时执行区（`~/executor/workspace`）操作，出站放行 control-api / `git.internal` / LiteLLM 代理端点，详见 [16.7](16-linux-permissions.md#167-执行节点executor-pc权限模型)

## 工具通道（WSL CLI / VSCode / Web）

设计与开发测试在三条工具通道间分工，均基于 WSL Linux 环境：

| 活动 | 通道 | 说明 |
|-----|------|------|
| 编码/重构/单元测试 | **WSL CLI（pi.dev）** | Agent 在 WSL 内自动执行，操作 `~/wt/` Worktree |
| 人工编码/审查/微调 | **VSCode（Remote-WSL）** | 开发者经 VSCode 连接 WSL，浏览/编辑/调试 Agent 产出；VSCode Extension 负责路径转换与 Diff 预览同步 |
| 设计文档/影响分析/RAG 检索 | **Web 客户端** | 文档查看 + Agent 生成（经 WSL CLI 提交 MR） |
| 测试执行（集成/回归/静态检查） | **执行节点**（CI Runner / executor） | 重负载构建测试在执行节点运行（见 [10 节点拆分](10-deployment.md#节点拆分编排节点-vs-执行节点)）；编排节点仅做提交前轻量校验；报告回传 Web |
| 任务干预/合并/日志 | **Web 客户端** | Diff 预览、任务暂停/回退、GitLab MR 跳转合并、work_log 检索 |

> 原则：**自动化执行一律走 WSL CLI（pi.dev）**，人工交互编码用 VSCode（Remote-WSL），设计文档与任务干预走 Web，**合并一律在 GitLab 人工执行**。三条通道共享同一 WSL 文件系统与 `~/wt/`/`~/repos/` 布局（见 [13.1](13-repo-template.md#131-wsl-开发环境目录架构以-linux-home-为基础)）。
