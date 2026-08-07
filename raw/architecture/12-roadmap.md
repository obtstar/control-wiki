# 12 实施路径

| 阶段 | 周期 | 交付 |
|-----|------|------|
| **一期：基础平台** | 2-3 周 | 控制中心仓库（设计文档 + 编排配置 + registry 注册表）+ 代码仓库脚手架（`control-api` Spring / `control-web` React / `control-db` MySQL DDL+DML）· 接入企业内 LiteLLM 代理 · 仓库 OpenAPI 集成 · RAG 只读检索 · 工作记录与工作报告 |
| **二期：多任务执行** | 3-4 周 | 流水线状态机（自动流转，GitLab 人工合并）+ Worktree 多分支管理（feature→release/dev）· 执行节点接入（executor 代理 / CI Runner）· Bug 修复流水线 |
| **三期：完整闭环** | 6-8 周 | 功能追加全流程 · release→main 发布 · 自我升级 + 日志检索增强 |

## 里程碑

- **M1（一期末）**：控制中心可录入需求、检索文档与代码、生成影响分析
- **M2（二期末）**：多任务可并行经 Worktree/执行节点执行，GitLab 人工合并闭环
- **M3（三期末）**：全流程自动化 + 自我升级灰度
