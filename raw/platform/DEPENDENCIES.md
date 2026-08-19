<!-- kb-mirror: upstream=control-api/DEPENDENCIES.md sha256=dde60bc4ebfee5b5aa83083c7b7b287edbd93f344aca09f71f2b2814a8dfc39b （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

# DEPENDENCIES — control-api 依赖登记

规约（CONVENTIONS.md）：新增依赖先登记本文件，说明"为什么标准库/现有依赖不够"；require 上限 12 个。
当前直接依赖 **4 个**（go.mod require，indirect 不列入）。

| 依赖 | 版本 | 用途 | 为什么标准库不够 |
|------|------|------|------------------|
| github.com/fsnotify/fsnotify | v1.10.1 | watcher 监听 tasks/ 目录变更 | 标准库无文件系统事件通知 |
| golang.org/x/crypto | v0.54.0 | authn bcrypt 口令哈希 | 标准库无 bcrypt；x/crypto 为准官方扩展 |
| gopkg.in/yaml.v3 | v3.0.1 | 配置 / pipeline.yaml / task.md frontmatter 解析 | 标准库无 YAML |
| modernc.org/sqlite | v1.53.0 | SQLite 驱动（纯 Go，WAL） | 标准库无 SQLite 驱动；纯 Go 免 CGO，与 control-piekbs 对齐 |

## 运行时外部依赖（非 go.mod）

| 项 | 说明 |
|----|------|
| pi（@earendil-works/pi-coding-agent） | 阶段执行器，`exec.CommandContext` 子进程调用（internal/agent） |
| LiteLLM 网关 | 模型统一出口，pi 经网关路由；control-api 只存 endpoint 配置 |
| PieKBS（规划） | KB 检索经 MCP 集成，当前无代码级依赖（FINDING-016） |
