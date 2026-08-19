<!-- kb-mirror: upstream=control-center/docs/architecture/16-linux-permissions.md sha256=7b3fbff10596c896049618fa257daf2ba460ec8e3b3dd27226cc8df985bd03a3 （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

---
status: 设计中
last_verified: 2026-08-11
---

# 16 Linux 用户权限管理设计

## 16.1 定位

平台在 WSL Linux（内网）上运行，**单用户多任务**场景仍需 Linux 用户权限管理，目的是把 **Agent 进程与用户本人隔离**，防止 Agent 越权访问文件或误操作宿主环境。简化原则：**身份隔离与数据位置解耦**——Agent 用独立用户身份运行，但代码/Worktree/数据全部留在 owner 的 home 下统一管理，Agent 用户不使用独立 home。

| 层级 | 控制内容 | 谁执行 |
|-----|---------|--------|
| 应用层 | 任务操作（创建/暂停/回退） | control-api |
| Linux 层 | 以谁的身份执行、能接触哪些文件 | 用户/文件权限 |
| 物理层 | 进程隔离、Worktree 目录隔离 | OS |

## 16.2 用户规划

| Linux 用户 | 对应角色 | 说明 |
|-----------|---------|------|
| `owner`（即用户本人账号，如 `dev`） | 平台拥有者 | 全部：仓库、编排配置、合并、发布；home 即平台根目录 |
| `agent` | 全部 Agent 进程（本机任务 + 执行节点 executor 通用） | 非 root、无 sudo、nologin；**不使用独立 home**，仅保留配置目录 `$OWNER_HOME/.agent`（pi 配置/会话落点） |

> Agent 与 owner 同在一个 home 树下，但身份独立：owner 的私有文件（`~/.ssh`、`.env` 600）对 agent 不可读；agent 能碰什么完全由目录属主/组控制。

## 16.3 目录与权限

| 目录 | 属主:属组 | 权限 | 说明 |
|-----|----------|------|------|
| `~/control-center` | `owner:owner组` | `750` | 设计文档 + 编排配置 + registry；agent 经组只读（技能/Prompt 加载） |
| `~/repos` | `owner:owner组` | `750` | 代码仓库；agent 经组只读 + 任务分支写（经 `~/wt`） |
| `~/wt` | `agent:owner组` | `770` | Agent 独占工作区 |
| `~/executor/workspace` | `agent:agent` | `750` | 执行节点临时执行区（任务结束即清理） |
| `~/.agent` | `agent:agent` | `750` | agent 用户的 pi 配置/会话目录（`models.json`、`skills` 软链） |
| `~/data`、`~/logs` | `owner:owner组` | `750` | 服务数据与日志 |
| `~/.env` 类密钥文件 | `owner:owner组` | `600` | agent 不可读（600 + 属主 owner） |

> 组共享约定：`agent` 加入 owner 主组，owner home 保持 `750`（组 r-x），agent 即可进入并读取组开放的目录，写权限仅限 `~/wt` 与 `~/executor/workspace`。

## 16.4 Agent 权限矩阵

| 执行位置 | 身份 | 可写范围 | 禁止 |
|---------|------|---------|------|
| 编排节点 | `agent` | `~/wt/{task-id}-*` | main/release 分支直接提交、非任务目录、owner 私有文件 |
| 执行节点 | `agent` | 本机 `~/executor/workspace` | 一切持久写（执行完清理）、访问其他用户 home |

> **Agent 层纵深防御**：除 Linux 用户隔离外，pi 侧加载官方 **protected-paths / permission-gate extension 示例**（07.3），在 Agent 进程内再挡一层越权文件访问——Linux 权限是兜底，extension 是第一道。

## 16.5 sudoers

- `agent` 不在 sudoers 中
- 所有合并（feature→release/dev、release→main）在 GitLab 由用户本人人工执行，平台不执行合并
- 所有 sudo 调用写入 `/var/log/secure`，可与 `work_log` 交叉核对

## 16.6 审计

- 文件级：目录权限、`~/logs` 服务日志
- 进程级：Agent 进程以 `agent` 用户运行，systemd/journal 记录启动与命令
- 关联：`work_log.operator` 记录执行身份（owner / agent / agent@{executor_id}），可与系统审计交叉核对，保证"何时以何身份执行了什么"

## 16.7 执行节点（executor PC）权限模型

executor 代理在办公 PC 的 WSL 上运行（见 [10 executor 代理](10-deployment.md)），与本机 Agent **同一身份模型**：

| 项 | 规则 |
|---|------|
| 执行身份 | executor 服务固定以 `agent` 账号运行（非 root、无 sudo、nologin），**与 PC 使用者的个人账号隔离** |
| 工作区 | `~/executor/workspace` 视为 Worktree 的延伸（**临时执行区**），任务结束即清理 |
| 出站白名单 | 执行节点放行且仅放行：control-api、`git.internal`、企业 LiteLLM 代理端点 |
| 机密仓库 | `repos.yaml` 中 `executor_allowed: false` 的仓库不下发 executor，仅在编排节点执行（14.2） |
| 命令白名单 | executor 仅执行登记的任务类型对应 `ci/` 白名单脚本，不执行仓库内任意脚本 |

### 容器化与多用户身份的关系

control-api 运行于 Docker 容器，**不直接以容器身份创建 Agent 进程**（容器内无法切换宿主 Linux 用户）。Agent/executor 进程的运行载体为：

- 编排节点本机任务：宿主机 **systemd 服务**（以 `agent` 身份运行），control-api 仅下发指令
- 执行节点任务：executor 服务（以 `agent` 身份运行），经 `/api/agents/*` 端点领取与回传

Linux 用户身份在执行载体上生效。
