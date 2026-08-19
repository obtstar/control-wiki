<!-- kb-mirror: upstream=control-center/docs/architecture/05-orchestration.md sha256=6bda5a440e0027a4c995df7b00fa58f14dd1bec9335d92c0da120e829f7e0615 （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

---
status: 已实现（核心链路）；任务分解/自我升级为规划中
last_verified: 2026-08-17
---

# 05 Agent 编排层（后端服务）

编排层由**平台后端代码仓库 `control-api`（Go 1.25+，stdlib `net/http` mux，无 Web 框架）**承担，负责任务管理、流水线状态机与调度，是全系统的调度中枢。运行数据落地 **SQLite**（`modernc.org/sqlite`，WAL 模式，运行时库 `~/data/control.db`；规模化时可经配置切换 MySQL/PG，DDL 届时入 `control-db`）。编排配置与注册表版本化于控制中心仓库 `orchestration/` 与 `registry/`。

## 技术选型（当前实现）

| 组件 | 选型 | 说明 |
|-----|------|------|
| 语言/框架 | Go + stdlib mux | 骨架期禁 Web 框架（gin/echo 等），见 control-api/CONVENTIONS.md |
| 安全 | 自有 authn 包 | bcrypt 口令 + 32 字节随机 token（24h 过期）+ 角色路由 |
| 调度/异步 | goroutine + fsnotify | maybeRun 异步执行、RetryLoop 熔断重试、watcher 目录监听、pipeline 热加载 |
| 存储 | SQLite（WAL） | task_index（派生）/approval/work_log（hash 链）/users/sessions |

## 流水线状态机（每阶段一审批闸）

状态机声明式定义于控制中心仓库 `orchestration/workflows/pipeline.yaml`（热加载，merge 阶段强制 `approval: required` 否则拒绝加载）：

```
[需求分析] → [待审批] → [系统设计] → [待审批] → [编码实现] → [待审批]
   ↑ 驳回附批注重做        ↑                  ↑
[测试验证] → [待审批] → [待合并] → [交付]
              ↑ 打回编码      └─ 终审：GitLab 人工合并（平台无合并按钮）
```

- **每阶段一闸**：阶段产物（影响分析/设计文档/MR diff/测试报告）生成后进入 `awaiting_approval`，用户在 Web 工作台**批准**进入下阶段或**驳回附批注**（AI 带批注重做本阶段；测试驳回打回编码）
- **终审不在平台**：进入"待合并"的自动质量关 = 测试全绿 + MR 自评；合并动作仍由用户在 Git 平台人工执行，Webhook 回传（`POST /api/webhooks/merge-event`）后自动推进交付
- **审计**：每次状态流转与审批（审批人/时间/意见/前后状态哈希）记 `work_log`，hash 链防篡改（`control-api verify-log` 可校验）
- **逃生门**：阶段可配 `approval: auto`，高频小任务的可信步骤可自动过闸，避免审批拖死吞吐
- **超时**：`awaiting_approval` 超 24h 未处理 → Web 通知，**不自动通过**（规划中，未实现）

| 阶段 | Agent 动作 | 输出物 |
|-----|-----------|--------|
| 需求分析 | 解析需求，检索 KB（PieKBS FTS），生成影响分析 | 影响分析报告 |
| 系统设计 | 生成接口定义、数据库变更、时序图 | 设计文档（任务目录 design.md） |
| 编码实现 | 创建 feature 分支 + Worktree，编码、单元测试 | Commit + MR |
| 测试验证 | 执行节点跑集成测试、回归测试、静态检查 | 测试报告 |
| 待合并/交付 | 推送 MR 与测试报告；用户在 Git 平台合并到 dev，Webhook 回传后清理 Worktree | 合并记录 |

## 任务管理（单人多任务）

| 能力 | 说明 | 状态 |
|-----|------|------|
| 任务即文档 | 任务权威为 `tasks/TASK-*/task.md` frontmatter，SQLite 索引为派生 | 已实现 |
| 任务分解 | 将需求/缺陷拆分为可执行子任务 | 规划中 |
| 多任务并发 | 多任务并行执行（当前为逐任务异步 goroutine，配额限流未实现） | 部分实现 |
| 仓库分配 | 根据任务类型分配到目标代码仓库（`repo_key` 字段） | 已实现 |
| 分支/Worktree 生命周期 | 创建、回收、归档 | 规划中 |
| 任务级熔断 | 连续失败 N 次（默认 3，pipeline.yaml `circuit_breaker`）→ 自动暂停 + 通知落 work_log；未达阈值置回 pending 由 RetryLoop 按退避表自动重跑；token 预算因 pi 不回传用量暂未执行 | 已实现（token 项除外） |
| 重启回收 | serve 重启将 running 任务自动暂停留痕（防僵尸），人工 resume 恢复 | 已实现 |

## 自我升级（规划中）

- 基于执行日志统计成功率，生成策略优化建议
- Prompt 版本化存 Git，升级即提 MR，用户在 Git 平台合并后生效
- 热加载免重启（pipeline.yaml 已实现），重大策略变更可先单任务灰度再全量
