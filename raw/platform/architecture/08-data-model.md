<!-- kb-mirror: upstream=control-center/docs/architecture/08-data-model.md sha256=8ca8f84d2f156d68ccf4ae2e7d333d6152d22d245e6be4183e85d1902c27aa84 （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

---
status: 已实现（SQLite 当前实现）；MySQL 规模化为规划项
last_verified: 2026-08-17
---

# 08 数据模型（SQLite 运行时库）

个人版以 **SQLite**（`modernc.org/sqlite`，纯 Go 驱动，WAL 模式）存储运行时数据。运行时库为 `~/data/control.db`。**DDL 权威不在本仓文档**：内嵌于 `control-api/internal/store/store.go` 的 `migrate()`（`CREATE TABLE IF NOT EXISTS`，幂等随服务启动执行）。规模化切换 MySQL/PG 时，DDL/DML 迁移脚本届时落于 `control-db` 仓（当前为占位仓），业务代码不动。

> **注册表不在数据库**：`registry/repos.yaml` 与 `registry/executors.yaml` 以 Git 文件为权威源（见 [14 多仓库管理](14-multi-repo.md)），变更走 MR。

## 当前表结构（与 store.go migrate() 对应）

| 表 | 角色 | 要点 |
|----|------|------|
| `task_index` | **派生**索引，由 `tasks/TASK-*/task.md` 重建 | task_id/stage/status/authority/path；看板数据源，可删表重建 |
| `work_log` | 运行期流水（归档回 Git 后可清空） | `prev_hash`+`entry_hash` 哈希链；`control-api verify-log` 校验 |
| `approval` | 审批队列 | 按角色路由（designer/tester/customer/team）；`decision NULL`=待审批 |
| `users` | 真人账号 | bcrypt 口令哈希；角色 customer/designer/tester/team/admin |
| `sessions` | 会话 token | 32 字节随机 token；`expires_at` 存 Unix 秒（INTEGER） |

数据分层（权威/派生/运行时/归档）详见 [01 概述](01-overview.md) 与 18 章：**任务状态权威是 task.md frontmatter**，`task_index` 仅派生；`work_log` 使用 hash 链，归档后可重置续接新周期。

## 工作日志落地

| 数据 | 位置 | 机制 |
|-----|------|------|
| **工作日志（work_log）** | SQLite `work_log` 表 | engine/api 每次 Agent/用户操作实时写入，含任务/阶段/动作/操作人/模型/detail |

> **工作报告（work_report）为规划项**：按日/周聚合 work_log → LLM 生成摘要 → Web 可查，当前未实现，无对应表。

## 追溯要求

- 日志结构化存储于 `work_log`，禁止物理删除（归档续接新周期），支持导出
- 操作绑定任务与执行身份（真实用户名 / agent），hash 链保证篡改可检出
- DDL 变更随 `control-api` 代码评审（migrate 幂等；破坏性变更需数据迁移方案）

## 附录：MySQL 规模化设计

本章早期版本含完整 MySQL DDL（task/work_log/doc_management/work_report 四表 InnoDB 设计），属规模化阶段的参考设计且与"任务即文档"模型已不一致，本次修订移除；规模化立项时以 store.go 实际 schema 为基线在 `control-db` 仓重建。
