<!-- kb-mirror: upstream=control-center/docs/architecture/11-security.md sha256=1127a6ea1e9e40bd845046cba0b098193ba195e2c3be9ff7dc1f9e301eb59b28 （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

---
status: 已实现
last_verified: 2026-08-17
---

# 11 安全与合规

> Linux 用户/组权限、Agent 隔离、出站白名单见 [16 Linux 权限管理](16-linux-permissions.md)。

| 风险 | 对策 |
|-----|------|
| 代码外泄 | 代码/数据全内网；出站防火墙仅放行企业内 LiteLLM 代理及其白名单模型端点（api.anthropic.com、api.githubcopilot.com），pi.dev 与其余流量不出网；机密仓库 `executor_allowed: false` 不分发执行节点 |
| AI 生成代码质量 | 强制单元测试 + 执行节点回归 + GitLab 人工合并 + 渐进式放权 |
| Agent 行为失控 | 任务级 Worktree 物理隔离 + Linux 独立用户（agent）+ 操作日志全追溯 + **任务级熔断**（连续失败/超 token 阈值自动暂停，05） |
| executor 执行未审代码 | executor 仅运行 `ci/` 白名单脚本（16.7），非 root 运行 + 出站白名单兜底 |
| 注册表篡改 | registry/ 在 Git 版本化，变更走 MR，历史可回溯可 revert |
| 自我升级风险 | Prompt/Skill 升级走 MR + 用户确认，可单任务灰度 |
| 追溯合规 | SQLite `work_log` 结构化日志（禁止物理删除 + prev_hash/entry_hash 哈希链防篡改，verify-log 可校验），与 Linux 审计交叉核对，支持导出 |

## 已知接受风险（单人场景，用户已拍板）

| 风险 | 接受说明 |
|-----|---------|
| Copilot 授权合规 | LiteLLM 的 `github_copilot` provider 属非官方接入，授权条款未必允许作通用模型 API；接受风险，故障/禁用时降级仅走 Anthropic |
| pi.dev 依赖成熟度 | Pi 为 MIT 开源、Earendil 维护，Skills/Extensions/RPC 均为原生能力；锁定版本 + vendor 安装包入库，必要时可替换执行器（架构经 RPC/CLI 对接，不锁定） |
| 代码上下文出站 | 内网代码可经 LiteLLM 出站至白名单模型端点（Anthropic/Copilot），接受该暴露面；机密仓库以 `executor_allowed: false` + 不出站约束单独管控 |
