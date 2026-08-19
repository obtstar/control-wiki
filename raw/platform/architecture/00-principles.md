<!-- kb-mirror: upstream=control-center/docs/architecture/00-principles.md sha256=3f1170614776f56e968cbf24c3b93bb893ce16daaca72e2fdf23ec4c22ae64f3 （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:56+08:00 -->

---
status: 已实现
last_verified: 2026-08-11
---

# 00 设计原则

| 原则 | 说明 |
|-----|------|
| **AI 驱动执行，用户逐步审批与合并权** | Agent 负责编码、分析、测试；流水线**每阶段设审批闸**（批准/驳回附批注），用户保留最终合并权 |
| **权柄分级，有据可依** | 项目 KB 按 需求>概要>详细>代码 分级权柄（L1 仅人可写）；**一切产出必须引用 KB 依据，无据暂停**，禁止凭空联想（18 章） |
| **单人多任务并行** | 一个用户并发调度多个任务，任务以 Worktree 物理隔离，配额控制并发上限 |
| **Git 为唯一可信源** | 代码、设计文档、编排配置、**仓库/执行节点注册表（registry/）**均在 Git 版本化，Agent 工作区用 Worktree 物理隔离 |
| **数据内网、AI 外接受控** | 代码/数据/服务均在内网；AI 能力经 LiteLLM 统一外接白名单模型（Anthropic、GitHub Copilot），出站仅限模型端点，密钥集中在服务端 |
| **操作可追溯** | work_log 结构化记录每次操作（任务、执行身份、模型、commit、状态哈希），支持回溯 |
