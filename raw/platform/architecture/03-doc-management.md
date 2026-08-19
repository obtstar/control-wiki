<!-- kb-mirror: upstream=control-center/docs/architecture/03-doc-management.md sha256=54f33e492fb215a6134da29c44bfeabfdf722701122ceff7fc100256561b7c87 （派生副本勿编辑；权威见 upstream，重生成: control-center/scripts/kb-sync.sh）
synced-at: 2026-08-19T21:58:57+08:00 -->

---
status: 已实现
last_verified: 2026-08-11
---

# 03 设计开发控制 & 文档管理（L2 AI 能力层）

控制中心仓库承载**设计开发控制文档管理**与 **KB 检索任务管理**（PieKBS 蒸馏知识库），是 AI 理解项目的基础。

## 文档管理

设计文档分级存放：**概要设计、外部设计集中管理于控制中心仓库**（Git 版本化）；**内部设计（详细设计）与代码同库**，随各代码仓库版本化。

| 文档类型 | 存放位置 | 说明 |
|---------|---------|------|
| 概要设计 | 控制中心仓库 `docs/design/overview/` | 系统级设计说明，描述模块划分与交互 |
| 外部设计 | 控制中心仓库 `docs/design/external/` | 接口/协议级设计（API 契约、数据结构） |
| 需求文档 | 控制中心仓库 `docs/requirements/` | PRD、变更请求 |
| **内部设计** | **各代码仓库 `docs/design/internal/`（与代码同库）** | 模块级详细设计，随本仓库代码同 MR |
| 数据库 DDL/DML | 代码仓库 `db/`：平台（`control-db`）/ 业务（业务仓库 `db/`） | 版本化脚本，随代码仓库管理 |

- 概要/外部设计：控制中心集中管理，变更走控制中心 MR（单人自审合并）
- **内部设计：与所属代码同仓库、同分支、同 MR 提交，评审与代码评审一体**
- 文档变更（MR 合并/Webhook 触发）后由 PieKBS 监听并增量蒸馏，更新 FTS 派生索引（`index/`，可重建）
- 可选：通过 Confluence API 将文档发布到内网 Wiki 供浏览，但权威源为仓库

## KB 检索任务管理（PieKBS）

知识层为 **PieKBS 蒸馏 KB**（`control-piekbs`，平台正式组件）：`raw/` 权威源 → `wiki/` 蒸馏层 → `schema/` 规则 → `index/` FTS 派生索引，检索经 FTS 全文索引。

```
KB 检索任务管理：
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ raw 权威源        │ →  │ wiki 蒸馏层       │ →  │ index FTS 派生索引│
│（人投放原始文档）   │    │（piekbs distill）│    │（检索，可重建）    │
└──────────────────┘    └──────────────────┘    └──────────────────┘
              schema/ 规则约束蒸馏产物结构
┌──────────────────────────────────────────────────────────────────┐
│  检索通道：control-api REST 窄接口（internal/kb Searcher，        │
│  GET piekbs :8766/api/search）· Agent 经 MCP（/mcp）              │
└──────────────────────────────────────────────────────────────────┘
```

- **蒸馏与索引**：文档变化（MR 合并 / 文件监听触发）后由 PieKBS 增量蒸馏并重建受影响索引；FTS 索引为派生物，可随时从 Git 重建
- **来源聚合**：平台级 KB 权威文档位于控制中心仓库 `docs/architecture/`（原 `control-wiki/raw/architecture` 已迁出）；`control-wiki` 保留 `wiki/`（蒸馏产物）、`schema/`（规则）、`index/`（派生索引）；项目级 KB 位于 `~/wiki/<repo>/`
- **检索增强（有据可依）**：问答、影响分析、Bug 定位基于 KB 检索结果生成；LLM 推理经企业内 LiteLLM 代理外接（Anthropic / GitHub Copilot），不跑本地模型
- **未采用的候选路线**：嵌入/向量检索（Milvus/FAISS）未曾实现，当前检索单一走 FTS

## OpenAPI 集成

- 各代码仓库提供 OpenAPI 规范（接口定义），控制中心统一拉取入 KB 检索来源
- 生成设计/补丁时参考接口契约，保证与外部接口一致；control-api 自身契约以 `control-api/docs/api/openapi.yaml`（OAS 3.1）为唯一可信源

## 能力清单

| 能力 | 说明 |
|-----|------|
| 文档检索 | 平台级 KB（控制中心 `docs/architecture/`）+ 项目级 KB（`~/wiki/<repo>/`）FTS 检索 |
| 代码/文档理解 | 基于 PieKBS FTS 检索的 KB 问答与依据引用 |
| Bug 定位 | 结合历史库与 KB 检索定位缺陷根因 |
| 补丁生成 | 参考接口契约生成最小化代码变更 |
| 增量蒸馏/索引 | 文档变更触发，仅更新变更内容 |
