<!-- kb-mirror: upstream=control-center/docs/architecture/19-workbench-strategy.md sha256=24a6b333ace8170c5cbf365adfcce3b9ff526c7dd67c70afb37fa08ec63167d6 （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-29T18:51:35+08:00 -->

---
status: 已裁决
last_verified: 2026-08-29
decision: TASK-010（人裁决，方案 A）
---

# 19 工作台形态决策：双线分工定位

## 1. 背景与问题

平台存在两个功能重叠的"工作台"入口：

| 入口 | 载体 | 视图 |
|------|------|------|
| control-web | 独立 React 应用（React 18 + PrimeReact + Vite） | 看板/审批/审计/问题/KB/API 文档 |
| control-dsh-plugin client 半区 | DSH GUI 内注入面板（DOM 直插） | 任务/审批/审计只读列表 |

重叠集中在**只读视图**（任务/审批/审计列表）；若不明确边界，将出现：审批操作入口重复建设、浏览器凭据泄露面扩大、client bundle 构建链耦合加深、维护双份 UI 的持续成本。

## 2. 决策（2026-08-29 人裁决，方案 A）

1. **control-dsh-plugin 定位为 AI 执行面**：`control_*` 工具 + 会话绑定 + 反向通知；client 半区仅保留 DSH GUI 内**只读投影**（任务/审批/审计列表，即里程碑 M1/M2），**明确不做 M3**（审批操作入口不进浏览器，凭据不进浏览器、不重复建设"凭据+角色+审计"审批面）。
2. **control-web 定位为人的审批工作台唯一入口**：审批操作（approve/reject/pause/resume/deliver）/看板/审计/KB/API 文档继续演进；TASK-008（AI 助手面板）按计划推进，形成"AI 会话内干活、人在工作台审批"闭环。
3. 本决策为 control-dsh-plugin `docs/ui-client-module.md` 中 **M3 里程碑的移除依据**（见 §5 约束声明）。

## 3. 四维依据

### 3.1 功能矩阵

| 能力 | control-web | 插件 client（DSH GUI） | 插件 Node（工具面） |
|------|------------|------------------------|---------------------|
| 任务看板 | ✅ BoardPage | ✅ 只读列表 | ✅ control_task_list |
| 审批操作 | ✅ ApprovalPage（角色路由+审计） | ❌ | ✅ control_task_action（会话内） |
| 待审批列表 | ✅ | ✅ 只读 | ✅ control_approvals_list |
| 审计日志 | ✅ AuditPage | ✅ 只读 | ✅ control_audit |
| 问题台账 | ✅ FindingsPage | ❌ | ✅ control_findings |
| KB 检索 | ✅ KBPage | ❌ | ✅ control_grounding_check |
| API 文档 | ✅ ApiDocsPage | ❌ | — |
| 登录/角色路由 | ✅ | ❌ | host 进程 env 凭据 |

来源：TASK-010 `report-requirements.md` §2（已审批）。

### 3.2 安全边界

- 权柄依据 [17-client-server-design.md §17.3](17-client-server-design.md)："原则：**客户端 = 纯展示**（零业务逻辑、零数据直连）……客户端只经内网 API 交互"。
- M3（浏览器内审批操作）将要求把凭据或审批动作引入浏览器侧，与 §17.3 边界冲突；即使经 host Remote 转发（凭据留 host），仍需在浏览器实现角色路由与操作 UI，重复 control-web 已有实现（`/api/approvals` 角色路由 + ApprovalDialog + work_log 审计）。
- 结论：**审批操作权威归属 control-web（服务端 API + 成熟 UI）**；插件 client 保持只读投影，凭据全程在 host 进程（DSH web 代理 `/control/dashboard/api/*` 透传，本次已修复其缺 `/api` 前缀的转发缺陷）。

### 3.3 构建链耦合

- control-dsh-plugin client 半区为 DOM 直插式模块，其注入依赖 DSH web 的 boot graph（`window.__DSH_BOOT__`）与 client bundle 构建工作流（`docs/ui-client-module.md §5`：需 deepseek-harness 源码仓 `pnpm run dev:web` 重建；AGENTS.md：改动热载须验证 watcher 在跑，否则每次改动重建 web 产物）。
- 扩展 M3 将使前端 UI 耦合 DSH 版本（rc.8 契约，升级需重编译）——与"插件=AI 执行面"的轻量定位相悖。

### 3.4 成本对比

| 项 | 做 M3（插件内审批操作） | 保持现状（control-web 审批） |
|----|------------------------|------------------------------|
| host Remote 数据通道 | 新增（TypertRemoteService + 描述符） | 已有（vite 代理 + control-api 直连） |
| 凭据边界 | 需设计"凭据不进浏览器"转发 | 已满足（control-web 登录 token） |
| 角色路由/审计 | 重做 | 已有（ApprovalPage + work_log） |
| 契约/测试 | client bundle 契约测试 + 构建链 | 已有（router/Approval 测试 9 件套） |
| 维护面 | 双份审批 UI | 单份 |

## 4. 落地（随 TASK-010 MR）

- 本文件（19 章）：决策记录（权威声明）
- [06-web.md](06-web.md)：追加"双线定位"小节
- [17-client-server-design.md](17-client-server-design.md)：追加 §17.4 插件投影客户端
- [README.md](README.md)：章节索引补 19
- `tasks/TASK-010/constraint-note-TASK-009.md`：TASK-009 执行范围约束摘要（供审批批注引用）

## 5. 约束声明（对 TASK-009 执行范围）

TASK-009（插件深度集成 v0.2，repo_key=control-dsh-plugin）后续阶段产物必须遵守：

1. `docs/ui-client-module.md` 的 **M3（审批操作入口）移除/标注"不做"**，M0-M2（只读投影）为上限；
2. client 半区任何改动**不得引入凭据（token）进入浏览器端逻辑**；
3. 执行面工具（`control_task_action` 等）保持 host 侧，不因 UI 决策变化。

> 本声明为权威依据：TASK-009 执行者/审批人据此约束其 design/coding 产物。

## 6. 重估触发条件

以下任一情形出现时，回到本决策重新裁决：

- 插件 client 出现**审批操作**需求（产品层面要求 DSH GUI 内直接审批）；
- control-web 停止演进（TASK-008 方向变更或仓库冻结）；
- DSH web 成为团队唯一入口且 control-web 访问价值被证伪。
