<!-- kb-mirror: upstream=control-center/docs/architecture/06-web.md sha256=12f63e3cc0e28930738ad1ed67a1f6339e4bb9c630cbddf5bc647f091dc89bbf （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-29T21:00:36+08:00 -->

---
status: 设计中
last_verified: 2026-08-17
---

# 06 Web 管理端（React + PrimeReact + Vite）

用户入口：多任务看板、任务干预、Diff 预览（合并跳转 GitLab 人工执行）与日志检索。由**平台前端代码仓库 `control-web`** 构建，React + PrimeReact + Vite，Nginx 提供内网静态托管。单用户单工作台（见 [17 客户端/服务端设计](17-client-server-design.md)）。

## 技术选型

| 组件 | 选型 |
|-----|------|
| 框架 | React 18+ |
| UI 组件库 | PrimeReact（含主题、DataTable、Calendar、Editor） |
| 构建 | Vite |
| 静态托管 | Nginx（内网） |

## 功能模块

| 模块 | 说明 | 关键 PrimeReact 组件 |
|-----|------|---------------------|
| 多任务看板 | 并行任务状态可视化（阶段/进度/所属仓库），从 task.md frontmatter 派生 | DataTable / Card / Tag |
| 问题一览 | 发现问题清单（FINDINGS.md 派生视图：状态/影响/去向，见 [18.5](18-authority.md)） | DataTable + Filter |
| 任务操作 | 创建任务、暂停/回退/批注修正（合并在 GitLab 人工执行） | Dialog / ConfirmDialog / Buttons |
| Diff 预览 | 代码变更与测试报告查看，跳转 GitLab MR 人工合并 | Editor / Splitter / ScrollPanel |
| 工作报告 | 日报/周报/任务报告查看（work_report） | DataTable / Editor |
| 日志检索 | work_log 结构化日志查询 | DataTable + Filter |
| KB 检索 | PieKBS 知识检索（/kb 页，经 control-api 代理） | DataTable / Search |

## 与后端对接

- REST API 调用后端 control-api（Go，`/api/**`），契约以 `control-api/docs/api/openapi.yaml`（OAS 3.1）为准
- WebSocket/SSE 推送任务状态变更与阶段完成通知
- 支持 WSL 路径与变更实时预览

## 双线定位（TASK-010 裁决，2026-08-29）

control-web 为**人的审批工作台唯一入口**（审批操作/看板/审计/KB/API 文档，审批按角色路由并留痕 work_log）。
control-dsh-plugin client 半区仅为 **DSH GUI 内只读投影**（任务/审批/审计列表），**不含审批操作**（M3 不做，凭据不进浏览器）。
决策依据与约束详见 [19 工作台形态决策](19-workbench-strategy.md)。
