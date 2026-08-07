# 02 多代码仓库 & Git Worktree 分支管理

系统管理**多个代码仓库**，每个仓库通过 **Git Worktree + 多分支策略** 实现任务的物理隔离与并行执行。

> 多仓库的注册、接入、配额与权限管理见 [14 多仓库管理](14-multi-repo.md)。

## 分支策略（业务仓：main → dev → release 只读主干，agent 只写 feature）

| 分支 | 切出关系 | 权限 | 生命周期 |
|-----|---------|------|---------|
| `main` | 生产主干 | **只读**（agent 与人均不直接提交） | 常驻 |
| `dev` | 从 `main` 切出 | **只读**（仅承接 feature 合并） | 常驻/按迭代重建 |
| `release` | 从 `dev` 切出 | **只读**（发布基线） | 发布合回后归档 |
| `feature` | `feature/{task-id}-{name}` / `bugfix/{task-id}-{name}`，**从 `dev` 切出** | **agent 唯一可写分支** | 合并后删除 |

### 分支流转模型

```
main ──→ dev（只读）──→ release（只读）
           ▲                ▲
           └── feature/TASK ─┘
           agent 只可切 feature → 修改/测试/push → MR → 团队员工合并入 dev 或 release
```

- **agent 写边界**：只能切出 `feature/*` 并在其中提交；对 `main`/`dev`/`release` 无任何写权限（仓库 OpenAPI 侧配置保护分支强制生效）
- **完成定义**：MR 被**团队其他员工**合并入 `dev` 或 `release`，Webhook 回传合并事件，任务才进入"交付"——push/MR 创建不算完成
- **回滚**：release 出问题时从 dev 打 feature 补丁分支走同一流程，禁止直接改只读主干

## Worktree 策略

- 每个任务在对应代码仓库创建独立 Worktree：`~/wt/{repo}/{task-id}-{type}-{name}`
- Worktree 与分支一一对应，checkout 到 `feature/{task-id}` 分支
- 并发限制：同时活跃 worktree ≤ N（防资源耗尽）
- 生命周期：任务合并完成后保留 7 天归档，然后清理
- 常驻工作区（`dev` 分支）仅用于只读检索/查看，禁止在其中编码

```
控制中心后端 ──→ git worktree add ~/wt/repo-a/TASK-001-feature-report feature/TASK-001
             ──→ git worktree add ~/wt/repo-b/BUG-042-fix-npe bugfix/BUG-042
```

## 变更与合并流程（Code Repo 侧）

1. 控制中心创建任务 → 在目标仓库从 `dev` 切出 `feature/{task-id}` 分支 + Worktree
2. Agent 在 Worktree 内编码、本地测试、commit
3. push 分支 → 通过仓库 OpenAPI 创建 MR，目标为 `dev`（发布窗口为 `release`）
4. **本地预处理验证**：团队 CI 跑集成/回归/静态检查，报告与 Diff 汇总回传（MR 注释 + Web 端可见）
5. **团队员工评审并合并**（平台不执行合并）→ Webhook 回传合并事件 → 记录 `work_log`、任务进入交付
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
