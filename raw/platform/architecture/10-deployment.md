<!-- kb-mirror: upstream=control-center/docs/architecture/10-deployment.md sha256=1282a61e9a801222c64048716f13bfcc3bad5896f52a3f85aa3c73ea11c85b8a （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

---
status: 已实现
last_verified: 2026-08-17
---

# 10 部署方案（个人本地版）

> **适用范围**：单用户、单机本地运行。多机/多租户设计（executor 舰队、昼夜槽位、容量规划）已裁剪；
> 仅保留 **wiki 多人共享**（PieKBS serve 局域网只读分享）。
> PieKBS（control-piekbs）为平台正式组件，与 control-api/web/db 同级。

## 10.1 单机形态

```
本机（8C/16GB+）
├── control-api + control-web     # 编排后端 + 工作台（compose 或本地进程）
├── SQLite ~/data/control.db      # 运行时层（WAL，备份=文件复制）
├── PieKBS serve（127.0.0.1:8766）# wiki 检索，MCP 供 pi 调用
├── ~/.repos + ~/wt               # gitdir 集中 + worktree 统一
└── LiteLLM（外部消费，不部署）    # 唯一出网通道
```

**不部署**：MySQL（个人版用 SQLite，规模化可回切）、Ollama、LiteLLM 代理、executor 舰队。

## 10.2 docker-compose（本地 docker 开发/测试环境）

compose 是**流水线"测试验证"阶段的执行载体**：编码完成后，在本机 docker 开发环境中
跑集成/回归/静态检查（不依赖团队 CI，不外发代码到任何共享 runner）。

- 服务：web（Nginx 静态页）、api（control-api）、测试运行时（按业务仓 `ci/` 脚本起容器执行）
- SQLite 内嵌于 api 数据卷（`~/data/control.db`），**无独立数据库容器**
- 测试报告回传 control-api，驱动状态机；测试驳回打回编码阶段（pipeline.yaml）
- `deploy/.env` 密钥为占位符（`change-me`），填好后才可 `docker compose up -d`

> 资源边界：本地 docker 测试峰值 2-4C/4-8GB（go test/pnpm vitest），
> 并发槽位计入任务配额，避免与编码/检索争抢本机资源。

## 10.3 executor（可选，默认不启用）

个人版默认本机执行（pi CLI + 本地构建测试），无需 executor。
仅当拥有多台个人 PC 且需要分发构建/测试时，才按 `registry/executors.yaml`
登记启用（schema 保留：executor_id/tags/slots/token_ref，无昼夜时段策略）。
心跳/长轮询/结果回传机制见 15 章，默认关闭不影响任何主流程。

## 10.4 wiki 多人共享（PieKBS = LLM wiki 专用数据库）

平台主体个人使用；**LLM wiki 的共享能力由 PieKBS 这个专用知识数据库承载**——
它是平台唯一的知识服务出口，检索、阅读、蒸馏全经它，不另设任何 wiki 服务：

```
PieKBS serve（本机 127.0.0.1:8766，知识数据库）
  ├── 本机：pi / control-api 经 MCP 调用（kb_search / kb_page / kb_add）
  └── 局域网团队成员：ssh -L 8766:localhost:8766 dev@<host>（SSH 密钥即认证，零新增认证面）
       或改 host: 0.0.0.0 + api_key 直接暴露（按需）
```

只读分享（检索/阅读）；写入（raw 投放、wiki 维护）仍只在平台侧进行。
共享内容 = control-wiki（平台级 KB）+ 项目级 KB（`~/wiki/<repo>/`，可选逐项目开放）。

## 10.5 备份（最小口径）

| 数据 | 方式 | 说明 |
|-----|------|------|
| 全部 Git 仓库（代码/文档/wiki/registry/任务） | push 远程 | **Git 即备份**，含全部历史 |
| SQLite control.db | 每日复制到 `~/data/backup/`，保留 14 天 | cron/systemd timer；重置前须完成 work_log 归档（08 章） |
| FTS 索引（wiki/任务看板） | **不备份** | 派生物，从 Git 重建 |
| 工具链（nvm/uv/Go/.local） | **不备份** | `setup-env.sh` 可重装 |

## 10.6 资源参考（单机）

| 项 | 占用 | 说明 |
|---|------|------|
| control-api + web | 1-2C / 2GB | 个人负载 |
| PieKBS | 0.5C / 0.5GB | FTS 检索 |
| 本地构建/测试峰值 | 2-4C / 4-8GB | go/vitest 并发槽位，计入任务配额 |
