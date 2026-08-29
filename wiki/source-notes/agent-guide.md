---
type: source-note
title: "Agent 项目指南（平台运维手册）"
description: "control 平台 Agent 指南：多仓布局、技术栈、构建运行、规约红线、流水线编排、数据分层、安全合规、已知差异速查"
tags: [platform, guide, operations]
resource: "raw/references/agent-guide.md"
timestamp: "2026-08-29"
processing: lightweight
sources:
  - raw/references/agent-guide.md
schema_version: 1
---

<!-- 蒸馏验证（TASK-000011：LiteLLM 砍除后 wiki-distill 链路独立可用） -->

# Agent 项目指南 — 蒸馏页

> 源：`raw/references/agent-guide.md`（AGENTS.md 副本，399 行，2026-08-29）

## 平台构成【平台|项目】

- **六仓布局**：control-center（引导仓：编排/注册表/脚本）、control-api（Go 编排后端）、control-web（React 工作台）、control-db（DDL 占位）、control-piekbs（知识引擎 Go/FTS）、control-wiki（平台级 KB）——gitdir 集中在 `~/.repos/`，`.git` 为指针文件
- **本地目录**：`~/wt/`（worktree 隔离）、`~/data/control.db`（SQLite WAL）、`~/logs/`、`~/control-api.yaml`（0600 配置）

## 技术栈【技术|组件】

- control-api：**Go 1.25+** + stdlib `net/http` mux（无 Web 框架）；依赖 fsnotify / bcrypt（x/crypto）/ yaml.v3 / modernc.org/sqlite（纯 Go SQLite）
- control-piekbs：Go 1.25+，**须 `-tags fts5` 编译**；依赖 mcp-go / pdf 解析 / systray
- control-web（规划）：React 18 + PrimeReact + Vite + pnpm；当前为占位仓库
- **模型出口**：DSH（dsh-llm）统一出口；LiteLLM 网关已砍除（TASK-000011，2026-08-29）

## 构建与运行【操作|命令】

- control-api：`go build -o control-api ./cmd/control-api/`；`./control-api serve`（127.0.0.1:8765）/ `check` / `reconcile`（文档↔事实对账，冲突退出码 1）/ `service install` / `user add <username> <role>`
- PieKBS：`piekbs init` / `serve`（8766，MCP /mcp）/ `index` / `status` / `lint`
- 环境初始化两阶段：init-env.sh（root：用户/目录/克隆）→ setup-env.sh（dev：工具链/同步/PieKBS/compose）

## 规约红线【规约|约束】（硬约束）

| 项 | 上限 | 处置 |
|----|------|------|
| 单文件 | ≤300 行 | 拆同包多文件 |
| 单函数 | ≤60 行 | 提取子函数 |
| 单包文件 | ≤8 个 | 拆新包需人评审 |
| go.mod require | ≤12 个 | 先登记 DEPENDENCIES.md |

- 禁止 `util/`/`common/`/`helper/` 垃圾桶包；internal 禁 panic/os.Exit/log.Fatal；错误 `fmt.Errorf("...: %w")` 包装
- 强制执行：`check-conventions.sh` 装为 pre-commit hook（**control-api、control-web 已装**；规模红线/gofmt/vet/eslint 违规拦截提交）

## 流水线与编排【架构|流程】

- 主流水线：`requirements → design → coding → testing → merge → deliver`，每阶段审批闸（required / auto / team_mr_review）
- merge 终审：团队在 Git 平台合并（仅 push/建 MR 不算完成）；任务级熔断：连败 3 次自动暂停
- **任务即文档**：`tasks/TASK-*/task.md` frontmatter（task_id/title/repo_key/stage/status/authority=L1）为权威

## 数据分层【架构|数据】

| 层 | 内容 | 存放 |
|----|------|------|
| 权威 | 代码/文档/编排/registry/task.md/KB | Git |
| 派生 | wiki FTS 索引、任务看板索引 | SQLite / PieKBS index |
| 运行时 | work_log 流水、任务状态机 | `~/data/control.db`（WAL） |

- work_log 使用 **hash 链**（防篡改，可 verify-log 校验）

## 安全与合规【安全|合规】

- 密钥不落盘（env 注入，0600）；角色模型 customer/designer/tester/team/admin；会话 bcrypt + 32B token + 24h + 常量时间比较
- **暂停为最高权限**：任何状态可触发，暂停期间禁一切写，仅人可恢复
- 有据可依：产出引用 KB 依据，无据输出 NO_BASIS

## 已知差异速查【运维|注意事项】

- FINDING-018/043：Java 工具链与 compose 规划形态**人裁决保留**（勿当遗留清理）
- FINDING-031：grounding enforce 下 Resume 重走 grounding 立即再暂停——先切 warn/off 或补 KB 再 Resume
- FINDING-051：`raw/platform/` 为镜像区（kb-sync.sh 生成，勿手改；上游改动后重跑 kb-sync）
- 测试覆盖：control-api 已补多包（authn/service/tasks 于 TASK-000015 补齐）

## Key Claims（蒸馏验证要点）

- 【平台|概念】control 平台为单人 AI Agent 平台：AI 驱动执行、用户逐步审批、Git 唯一可信源
- 【架构|概念】模型出口已收敛为 DSH 单一通道（LiteLLM 砍除，TASK-000011 人裁决）
- 【规约|概念】规模红线 4 项为硬约束，pre-commit 强制（三仓已装：control-api/control-web/control-center）
- 【运维|操作】蒸馏操作：DSH 会话加载 wiki-distill 技能执行（raw → wiki/source-notes/），本页为链路验证产物
