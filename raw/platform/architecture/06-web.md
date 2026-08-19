<!-- kb-mirror: upstream=control-center/docs/architecture/06-web.md sha256=514f69110e1d5bcd714933c49b184eb41d03e76da4326e877468e47355103d7f （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

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
