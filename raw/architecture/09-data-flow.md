# 09 数据流示例：功能追加全流程

```
1. 需求录入 (Web)
   用户在 React 端创建 REQ-001，上传需求文档到控制中心仓库 docs/requirements
   → 写入 MySQL task 表 → 触发后端编排

2. 项目理解 (RAG)
   Spring 后端检索 RAG（控制中心仓库设计文档 + 各代码仓库 + OpenAPI 契约）
   → 生成影响分析 → Web 推送 → 自动进入设计阶段（用户可随时介入修正）

3. 设计阶段
   后端生成概要/外部设计 → 文档提交控制中心仓库 MR（集中管理）→ 状态自动流转

4. 编码阶段（多代码仓库 Worktree）
   后端调用代码仓库 OpenAPI 创建 feature/TASK-001 分支（从 main 或 dev 切出）+ Worktree
   → CLI Agent（pi.dev）经 LiteLLM 生成代码
   → 内部设计（docs/design/internal/）与代码同分支、同 MR 提交
   → 本地测试 → commit → push → 创建 MR 指向 release/{当月}（或 dev）

5. 测试阶段（执行节点，本地预处理验证）
   executor / CI Runner 拉取分支执行集成测试、回归测试、静态检查
   → 报告回传（MR 注释 + Web 可见）→ 写 work_log
   → 自动质量关：heavy 模型自评 MR Diff，自评通过 + 测试全绿 → 进入"待合并"

6. 合并 (GitLab 人工)
   用户在 GitLab 查看 Diff、测试报告与自评报告 → 人工合并
   → Webhook 回传合并事件 → 任务流转 + 增量 RAG 索引

7. 发布与归档
   release/{当月} 月度在 GitLab 人工合并 → main
   → 全流程日志写入 MySQL work_log
   → 清理 Worktree（保留 7 天归档）

8. 报告生成
   定时任务聚合 work_log 生成日报/周报/任务报告（work_report）→ Web 端查看
   → pi 会话树 /export HTML 归档为任务证据（可追溯完整执行过程）
```
