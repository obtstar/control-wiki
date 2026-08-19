<!-- kb-mirror: upstream=AGENTS.md sha256=67f7cd6a9e1d04b8880803527ea2bc43dcce5f562434b0ff2e38583c167615d4 （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

# Agent 项目指南

本文件面向后续接手本工程的 AI 编码助手。阅读前请假定你对本项目一无所知；下文所有信息均来自当前仓库实际内容，不做推测。

---

## 1. 项目总览

`/home/dev/` 是一个**多仓（multi-repo）本地开发环境**，围绕“团队项目中的个人 AI 助手”这一平台构建。平台本身由 6 个仓库组成，业务项目仓按需额外登记。

核心设计原则（来自 `control-wiki/raw/architecture/00-principles.md`）：

- **AI 驱动执行，用户逐步审批**：流水线每阶段一道审批闸，用户保留最终合并权。
- **权柄分级，有据可依**：文档按 L1 需求 > L2 概要设计 > L3 详细设计 > L4 代码 分级；AI 只能顺行产出，不得逆行修改上级文档。
- **Git 为唯一可信源**：代码、文档、编排配置、注册表均进 Git；Agent 工作区用 Git worktree 物理隔离。
- **数据内网、AI 外接受控**：模型请求经 LiteLLM 网关统一出口，密钥集中在服务端。

### 1.1 平台六仓

| 仓库 | 路径 | 角色 | 说明 |
|------|------|------|------|
| `control-center` | `~/control-center` | 引导仓 | 编排配置 `orchestration/`、注册表 `registry/`、环境脚本 `scripts/` |
| `control-api` | `~/control-api` | 编排后端 | Go 编写的 HTTP 服务，负责任务状态机、审批、调度 |
| `control-web` | `~/control-web` | 工作台前端 | 当前为占位仓库，仅 README 与 `.npmrc` |
| `control-db` | `~/control-db` | 数据库 DDL | 当前为占位仓库，仅 README |
| `control-piekbs` | `~/control-piekbs` | 知识引擎 | Go 编写的 PieKBS：FTS 搜索 + MCP 服务 + 文档蒸馏 |
| `control-wiki` | `~/control-wiki` | 平台级知识库 | `raw/` 原始文档、`wiki/` 蒸馏产物、`schema/` 规则；`index/` 为派生，不提交 |

业务项目仓按同 schema 登记在 `control-center/registry/repos.yaml`，工作区位于 `~/wt/projects/<repo>/dev/` 或任务级 `~/wt/<repo>/TASK-*/`。

### 1.2 本地目录布局

```text
/home/dev/
├── .repos/                      # 全部 gitdir 集中（bare/分离式）
├── control-center/              # 引导仓
├── control-api/                 # 平台后端（Go）
├── control-web/                 # 前端占位
├── control-db/                  # DDL 占位
├── control-piekbs/              # PieKBS 源码（Go）
├── control-wiki/                # 平台级 KB
├── wiki/                        # 项目级 KB 根（业务仓知识库）
├── wt/                          # 共享工作区根
│   ├── projects/<repo>/dev/     # 业务项目常驻工作区
│   └── <repo>/TASK-*/           # 业务任务 worktree
├── data/
│   ├── control.db               # SQLite 运行时库
│   ├── control.db-shm
│   └── control.db-wal
├── deploy/                      # docker compose 测试环境
├── logs/                        # 运行日志
├── scripts/                     # 本机运维脚本
├── control-api.yaml             # control-api 运行时配置
└── control.env                  # 环境变量模板产物
```

> 注意：本环境各子目录都是独立 Git 仓库，`.git` 是指向 `~/.repos/<name>.git` 的指针文件；`/home/dev/` 本身不是 Git 仓库。

---

## 2. 技术栈

### 2.1 语言与运行时

| 组件 | 技术 | 备注 |
|------|------|------|
| control-api | Go 1.25+ | 标准库 `net/http` mux，无 Web 框架 |
| control-piekbs | Go 1.25+ | 需 `-tags fts5` 编译 |
| control-web（规划） | React 18 + PrimeReact + Vite + pnpm | 当前仓库未实现 |
| 数据库 | SQLite（modernc.org/sqlite） | WAL 模式；运行时库为 `~/data/control.db` |
| 向量库（规划） | Milvus | 声明于 compose.sh 生成模板（规划形态），未接入实际代码 |
| 缓存（规划） | Redis | 声明于 compose.sh 生成模板（规划形态），未接入实际代码 |

### 2.2 关键依赖

**control-api (`go.mod`)**：

- `github.com/fsnotify/fsnotify` — 任务目录监听
- `golang.org/x/crypto` — bcrypt 口令哈希
- `gopkg.in/yaml.v3` — 配置与流水线解析
- `modernc.org/sqlite` — SQLite 驱动（纯 Go）

**control-piekbs (`go.mod`)**：

- `github.com/fsnotify/fsnotify`
- `github.com/getlantern/systray` — macOS 托盘
- `github.com/ledongthuc/pdf` — PDF 解析
- `github.com/mark3labs/mcp-go` — MCP 协议服务
- `golang.org/x/net`
- `modernc.org/sqlite`

### 2.3 外部服务

- **LiteLLM 网关**：模型统一出口，默认 `http://litellm.internal:4000`，配置在 `control-api.yaml` / `control.env`。
- **团队 Git 服务器**：业务仓托管、MR 评审、保护分支。
- **GitHub (`obtstar`)**：平台六仓托管。

---

## 3. 代码组织

### 3.1 control-api

```text
control-api/
├── cmd/control-api/main.go       # 入口：解析子命令，调 internal
├── internal/
│   ├── agent/                    # pi / fake-pi 阶段执行器
│   ├── api/                      # HTTP 路由与 handler
│   ├── authn/                    # 用户、会话、角色、bcrypt
│   ├── config/                   # 配置加载（yaml + env 覆盖）
│   ├── engine/                   # 状态机：approve/reject/pause/resume + KB grounding
│   ├── kb/                       # KB 依据检索窄接口（RESTSearcher → PieKBS）
│   ├── pipeline/                 # 加载 orchestration/workflows/*.yaml
│   ├── reconcile/                # 对账 loop（方案 D-2）：checks.yaml 声明的文档↔事实校验
│   ├── service/                  # systemd 用户级服务管理
│   ├── store/                    # SQLite 运行时层 + 迁移
│   ├── tasks/                    # 任务即文档：解析 task.md frontmatter
│   └── watcher/                  # fsnotify 监听 tasks/ 目录
├── go.mod
└── go.sum
```

包组织约定：一个领域一个包，包名即领域名；禁止 `util/`、`common/`、`helper/` 等垃圾桶包；共享配置进 `internal/config`。

### 3.2 control-piekbs

```text
control-piekbs/
├── cmd/piekbs/main.go            # CLI 入口
├── cmd/test_related/main.go      # 关系测试工具
├── internal/
│   ├── config/                   # KB 配置
│   ├── convert/                  # PDF/Word/Excel/PPT/EPUB/HTML 转换
│   ├── distill/                  # LLM 蒸馏流水线
│   ├── kb/                       # FTS 索引、搜索、图展开、页面获取
│   ├── kbinit/                   # KB 初始化
│   ├── larkimport/               # 飞书/Lark 导入
│   ├── llmurl/                   # LLM URL 解析
│   ├── mcp/                      # MCP server（stdio + HTTP）
│   ├── service/                  # launchd/systemd 服务管理
│   ├── synthesize/               # 概念/对比/决策页生成
│   ├── tray/                     # macOS 托盘
│   ├── version/                  # 版本信息
│   ├── watcher/                  # 文件监听与自动蒸馏
│   └── webui/                    # Web UI
├── scripts/build.sh              # 多平台构建
└── docs/                         # 文档与站点
```

---

## 4. 构建与运行

### 4.1 环境初始化（两阶段）

阶段一（需 root）：创建用户、目录、克隆 control-center。

```bash
curl -fsSL https://gh.dpik.top/https://raw.githubusercontent.com/obtstar/control-center/dev/scripts/init-env.sh | sudo bash -s --
```

阶段二（以 dev 身份）：安装工具链、同步仓库、初始化 PieKBS、compose。

```bash
bash ~/control-center/scripts/setup-env.sh [--executor|--check|--skip-*]
```

### 4.2 构建命令

**control-api**：

```bash
cd ~/control-api
go build ./...
go build -o control-api ./cmd/control-api/
```

**control-piekbs**（必须带 `fts5` tag）：

```bash
cd ~/control-piekbs
go build -tags fts5 ./...
go build -tags fts5 -o piekbs ./cmd/piekbs/
```

多平台发布包：

```bash
cd ~/control-piekbs
./scripts/build.sh [version] [target...]
```

### 4.3 运行命令

**control-api**：

```bash
# 默认读取 ~/control-api.yaml，或 CONTROL_CONFIG 覆盖
~/control-api/control-api serve        # 启动 HTTP 服务（默认 127.0.0.1:8765）
~/control-api/control-api check        # 自检
~/control-api/control-api reconcile    # 对账 loop（方案 D-2）：文档声明 vs 代码事实，有冲突退出码 1
~/control-api/control-api service install   # systemd 用户级服务
~/control-api/control-api user add <username> <role>
```

**PieKBS**：

```bash
export PIEKBS_KB=~/control-wiki
piekbs init
piekbs serve                           # 默认 127.0.0.1:8766，MCP /mcp
piekbs index
piekbs status
piekbs lint
```

### 4.4 配置

- `~/control-api.yaml`：control-api 配置（server/db/llm/paths/agent）。首次启动不存在时会生成默认配置并写入，权限 `0600`。
- `~/control-wiki/config.yaml`：PieKBS 配置（server/distill/embedding/ui）。
- `~/control.env`：环境变量（由 `init-env.sh` 生成）。
- 密钥（`api_key`、`token` 等）优先走环境变量，不落盘。

---

## 5. 测试命令

**control-api**：

```bash
cd ~/control-api
go test ./...
```

当前（截至本文件写入时）该仓库尚无测试文件，运行结果全部为 `[no test files]`。

**control-piekbs**：

```bash
cd ~/control-piekbs
go test -tags fts5 ./...
```

部分包已有单元测试；测试使用 SQLite `:memory:` 或临时目录，不 mock 数据库。

---

## 6. 开发规约

### 6.1 规模红线（硬约束，来自 `control-api/CONVENTIONS.md`）

| 项 | 上限 | 超限处理 |
|----|------|---------|
| 单文件 | 300 行 | 拆分为同包多文件 |
| 单函数 | 60 行 | 提取子函数 |
| 单包文件数 | 8 个 | 说明包承担多领域，拆新包需人评审 |
| `go.mod` require 数 | 12 个 | 新增依赖先登记 `DEPENDENCIES.md` |

### 6.2 代码风格

- Go：`gofmt` + `go vet` 全绿；CI 检查。
- Web（control-web）：`pnpm lint`（eslint 9 flat config）零 error 方可提交。
- 强制执行点：`control-center/scripts/check-conventions.sh`，经 `install-hooks.sh <repo>` 装为各仓 pre-commit hook（已装 control-api、control-web）；规模红线/gofmt/vet/eslint 违规直接拦截提交，机生成文件（auto-generated 标记）豁免行数红线。
- 错误处理：`fmt.Errorf("...: %w", err)` 逐层包装；internal 包禁止 `panic`/`os.Exit`/`log.Fatal`。
- 日志：stdlib `log`，统一前缀 `[<domain>]`；禁止打印密钥/token。
- 禁止全局可变状态；配置只经 `config.Config` 显式传递；env 读取只在 `config.Load` 发生。
- HTTP：路由注册集中在 `internal/api/routes.go`（或 `server.go`），handler 一行调服务层。

### 6.3 包组织

- `cmd/<binary>/main.go` 只做命令解析与调用，不写逻辑。
- `internal/<domain>/` 一个领域一个包。
- 禁止 `util/`、`common/`、`helper/`。
- main 包不 import internal 反向，internal 不 import cmd。

### 6.4 测试风格

- 表驱动测试，`<name>_test.go` 与被测文件同包。
- 每个导出函数至少一个用例；状态机迁移必须覆盖非法流转。
- 不 mock 数据库：SQLite 用 `:memory:` 实测。

### 6.5 依赖管理

- 新增依赖 → 先更新 `DEPENDENCIES.md`，MR 说明“为什么标准库/现有依赖不够”。
- 优先标准库；优先与 control-piekbs 已用依赖对齐（sqlite/mcp/fsnotify/yaml）。
- 禁止引入 Web 框架大件（gin/echo 等，骨架期用 stdlib mux）、ORM、反射库。

### 6.6 AI 编码特别条款

- AI 提交必须引用权柄文档（docs ID + 段落），引用须经校验真实存在。
- AI 不得修改 L1/L2/L3 权柄文档；发现规约与需求冲突 → 出不一致报告并暂停。
- AI 不写无对应任务的代码；每个 commit 关联 TASK-id 或明确技术债条目。
- 自检清单：规模红线 / 错误处理 / 无全局状态 / 无新依赖。

---

## 7. 流水线与编排

### 7.1 主流水线

声明式定义：`control-center/orchestration/workflows/pipeline.yaml`（control-api 热加载）。

```text
requirements（需求分析） → design（系统设计） → coding（编码实现） → testing（测试验证） → merge（待合并/团队 MR 评审） → deliver（交付）
```

每阶段一道审批闸：`approval: required` / `auto` / `team_mr_review`。

- 驳回必须附批注；测试驳回打回 coding 阶段。
- merge 阶段终审为**团队员工在 Git 平台合并**；仅 push/建 MR 不算完成。
- 任务级熔断：连败 3 次或 token 超阈值 → 自动暂停。

### 7.2 任务即文档

任务目录结构：

```text
control-center/tasks/TASK-001/
├── task.md         # 权威：frontmatter 携带状态
├── design.md       # 设计产物
├── report-coding.md
└── report-design.md
```

`task.md` frontmatter 字段：`task_id`、`title`、`repo_key`、`stage`、`status`、`priority`、`authority`（恒 L1）。

### 7.3 工作流触发

`control-center/orchestration/workflows/feature-dev.yaml` 等定义触发规则与 skill 组合。

---

## 8. 数据分层

| 层 | 内容 | 存放 |
|----|------|------|
| 权威（低频） | 代码、文档、编排配置、registry、task.md、KB | Git |
| 派生（读多） | wiki FTS 索引、任务看板索引 | SQLite / PieKBS index（可重建） |
| 运行时（高频） | work_log 流水、任务状态机 | `~/data/control.db`（WAL） |
| 归档 | log.jsonl、work_report.md | 事件触发归档回 Git 后 SQLite 方可重置 |

- `work_log` 使用 hash 链；重置后续接新周期。
- 个人本地版用 SQLite；规模化时可通过配置切换 MySQL/PG，业务代码不动。

---

## 9. 安全与合规

- **密钥不落盘**：`api_key`、`LITELLM_API_KEY`、`GIT_TOKEN` 等优先通过环境变量注入。
- **最小权限**：control-api 角色模型 `customer/designer/tester/team/admin`，审批按角色路由。
- **会话安全**：bcrypt 口令、32 字节随机 token、24h 过期；认证使用常量时间比较。
- **暂停为最高运行时权限**：任何状态可触发，暂停期间禁止一切写操作（代码/wiki/远端），仅人可恢复。
- **有据可依**：产出必须引用 KB 依据；无据输出 `NO_BASIS` 并停止。
- **出网白名单**：模型请求只经 LiteLLM；代码/数据不出本机。

---

## 10. 已知重要差异与注意事项

1. **文档与代码实现的历史差异已部分修正**：`control-center/README.md` 与 `DEPENDENCIES.md` 曾称 `control-api` 为 Java Spring Boot、`control-web` 为 React，已更正为 Go 实现与"前端规划中（当前占位仓库）"。注意 `control-center/scripts/`（toolchain.sh、setup-env.sh 等）仍会安装 JDK/Maven 工具链——此为**有意保留的环境 provisioning**（面向未来业务项目开发；FINDING-018 人裁决 wontfix，2026-08-17），仅平台自身（control-api）不使用 Java，勿当作遗留清理。同理 `orchestration/skills/domain/backend-java` 技能保留供 Java 业务项目使用（FINDING-045，人裁决保留，2026-08-17）；平台默认领域技能为 backend-go，Java 项目按 `repo_key` 覆盖。
2. **`deploy/docker-compose.yml`** 已改为与当前 Go/SQLite 实现一致的形态（api + web 两个服务，SQLite 卷挂载）；MySQL/Redis/Milvus 为规划项，未接入代码，已从盘上 compose 移除。注意 `control-center/scripts/lib/compose.sh` 生成器输出仍为 MySQL/Milvus/Redis **规划形态**——经人裁决有意保留（FINDING-043 wontfix，2026-08-17），新环境跑 setup-env 生成的是规划形态模板，与盘上手工修正文件不一致为已知状态；init-env 创建 `data/mysql`、`data/milvus` 空目录、check.sh 检查 `data/mysql` 同案保留。`deploy/mysql/init/` 为空目录，已无引用。
3. **测试覆盖已起步但不均衡**：`control-piekbs` 有多包单元测试；`control-api` 已补 engine/api/config/pipeline/store 等包测试（含契约对账测试层），`authn/service/tasks` 仍无测试，后续应按规约补齐。
4. **control-piekbs 编译必须带 `-tags fts5`**，否则 SQLite FTS5 相关代码无法编译。
5. **方案 D-2 对账 loop 已通电**（2026-08-17，control-api 939875d + control-center 2d0bb9e）：`control-api reconcile` 按 `control-center/orchestration/reconcile/checks.yaml` 校验文档声明 vs 代码事实（后端/前端/数据库/注册表字段四项），CONFLICT 退出码 1、结论带依据出处；首跑 WARN（registry `path` 字段 14.2 未声明）已经人裁决 A 修订 14.2 补齐声明（FINDING-049，2026-08-18），当前四项全 PASS 零 WARN。文档改动后应跑一次 reconcile；新增机器可判对照项 → 扩展 checks.yaml。另注意：control-api `.gitignore` 的 `/control-api` 为锚定根目录写法（FINDING-047 教训：非锚定模式会误伤 `cmd/control-api/`）。
6. **`control-piekbs/.gitignore` 中忽略了 `AGENTS.md` 和 `CLAUDE.md`**；本根目录 `AGENTS.md` 不会被该平台仓库的 Git 跟踪，如需纳入版本需单独处理。
7. **grounding enforce 模式的恢复陷阱**（FINDING-031，2026-08-18）：`kb.grounding: enforce` 下若 KB 检索为空，任务会因 `NO_BASIS` 暂停；此时直接 Resume 会**重走 grounding 并立即再次暂停**。恢复途径：先把 `control-api.yaml` 的 grounding 切为 `warn`/`off`，或先补齐 KB 语料再 Resume。语义出处 `control-api/internal/engine/grounding.go`。
8. **PieKBS 已 systemd 常驻，蒸馏待网关**（FINDING-017/050，2026-08-18）：`piekbs-mcp.service`（user 级，Restart=on-failure，已 `loginctl enable-linger dev` 开机/登出存活）托管 8766 端口，内嵌 watcher/追平/蒸馏工作池；**勿再 nohup 手动起 serve**（端口冲突且绕过重启策略）。`piekbs service install` 只装 piekbs-mcp——旧 indexer 单元的 `watch` 子命令不存在（FINDING-050，macOS launchd 侧同病未修）。当前 distill 未启用：`config.yaml` 的 `distill.token` 为空且网关 `litellm.internal` DNS 不可达。**网关恢复手册**：① `curl -m 3 http://litellm.internal:4000/health` 确认通；② 写 0600 env 文件（如 `~/.config/piekbs/env`，含 `PIEKBS_DISTILL_TOKEN=...`）+ `systemctl --user edit piekbs-mcp` 加 `EnvironmentFile=` 指向它（密钥不落 Git）；③ `systemctl --user restart piekbs-mcp`，启动追平自动蒸馏 raw/ 存量；④ 观察 `wiki/source-notes/` 出页、`piekbs status` 计数上涨；⑤ KB 有料后再把 control-api grounding 切 enforce（注意 §10-7 陷阱）。
9. **raw/platform/ 是派生镜像区**（FINDING-051，2026-08-19）：权柄文档（架构/规约/契约/台账/编排/注册表/AGENTS.md）经 `control-center/scripts/kb-sync.sh` 镜像入 KB 供 FTS/grounding 检索；权威居所仍是各 Git 仓原文件，镜像首行带 `kb-mirror` 出处头（upstream+sha256），**勿手改镜像**；清单唯一来源 = checks.yaml `mirror_pairs`，`reconcile` 的 `kb-mirror-freshness` 盯漂移。上游文档改动后应重跑 kb-sync.sh（reconcile 会 WARN 提醒）。

---

## 11. 常用速查

```bash
# 构建
cd ~/control-api && go build ./...
cd ~/control-piekbs && go build -tags fts5 ./...

# 测试
cd ~/control-api && go test ./...
cd ~/control-piekbs && go test -tags fts5 ./...

# 静态检查（pre-commit hook 自动执行，亦可手动）
bash ~/control-center/scripts/check-conventions.sh [repo]   # 规模红线+gofmt+vet+eslint
cd ~/control-web && pnpm lint
bash ~/control-center/scripts/install-hooks.sh <repo>       # 给新仓装 hook

# 运行
cd ~/control-api && ./control-api serve
cd ~/control-piekbs && ./piekbs serve

# 环境检查
bash ~/control-center/scripts/setup-env.sh --check
```

---

*最后更新：基于 `/home/dev/` 当前实际内容整理。*
