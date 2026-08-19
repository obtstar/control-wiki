<!-- kb-mirror: upstream=control-center/docs/architecture/09-data-flow.md sha256=0139467643f1aa5c10bf4e4d303d3316b1473100c6ab5d3bc6253757ab9988c9 （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

---
status: 部分实现（主链路 1/3/6 已通；执行节点/报告生成/Worktree 自动化为规划中）
last_verified: 2026-08-17
---

# 09 数据流示例：功能追加全流程

> 与实现对齐说明：当前任务权威为 `control-center/tasks/TASK-*/task.md`（任务即文档），
> 知识检索走 PieKBS（FTS + REST `/api/search`，经 control-api `/api/kb/search` 代理），
> 非原设的 RAG/MySQL 形态。各步骤标注实现状态。

```
1. 需求录入 (Web)【已实现】
   用户在 Web 工作台创建任务（POST /api/tasks）
   → 任务落 tasks/TASK-*/task.md（权威）→ SQLite task_index 派生同步
   → 进入 pipeline 首阶段

2. 项目理解 (KB)【部分实现】
   后端经 PieKBS 检索平台知识库（grounding 支持 off/warn/enforce 三模式，
   enforce 下无据输出 NO_BASIS 并自动暂停；当前默认 off，FINDING-016/017）
   → 生成影响分析 → 待审批

3. 设计阶段【已实现】
   产物落任务目录 design.md → awaiting_approval → 用户批准/驳回附批注

4. 编码阶段（多代码仓库 Worktree）【部分实现】
   pi 执行器（agent.command 配置）生成代码并落 report-<stage>.md
   → feature 分支 + Worktree 自动化、MR 创建为规划中（当前人工/脚本）

5. 测试阶段（执行节点，本地预处理验证）【规划中】
   executor 舰队未建；自动质量关（heavy 模型自评 MR Diff）未实现

6. 合并 (Git 平台人工)【已实现】
   用户在 Git 平台查看 Diff 与报告 → 人工合并
   → Webhook 回传（POST /api/webhooks/merge-event，独立密钥 HMAC）
   → 任务置 merged 停留待交付（FINDING-029：看板可感知"已合并待交付"）
   → 人工 action=deliver 确认后推进 deliver（auto，agent 执行交付归档）

7. 发布与归档【部分实现】
   全流程日志写 SQLite work_log（hash 链，verify-log 可校验）
   → Worktree 清理/归档为规划中

8. 报告生成【规划中】
   定时聚合 work_log 生成日报/周报未实现；
   pi 会话 /export HTML 归档为任务证据沿用中
```
