# PieKBS KB Directory Structure

Default knowledge base instance name:

```text
piekbs-kb/
```

The name is only a convention. Users may choose any local directory as the KB root.

## Standard Structure

```text
piekbs-kb/
  raw/
    articles/
    papers/
    webpages/
      wechat/
    images/
    data/
    code/
    converted/

  wiki/
    source-notes/   ← core, auto-created by init
    concepts/       ← core, auto-created by init
    comparisons/    ← core, auto-created by init
    decisions/      ← core, auto-created by init

  schema/
    AGENTS.md
    wiki-structure.md
    page-templates.md
    ingestion-workflow.md
    citation-rules.md
    conflict-rules.md

  index/
    kb.sqlite
    gaps/           ← generated gap analysis reports (not wiki pages)
```

## Directory Roles

- `raw/`: authoritative original sources and converted near-full-text derivatives.
- `wiki/`: structured, cited Markdown knowledge maintained by agents and humans.
- `schema/`: KB-local rules, templates, citation rules, and maintenance workflow docs.
- `index/`: generated search artifacts such as SQLite, FTS, vector, or graph indexes.

## Raw Source Rules

- Do not modify original raw files.
- Put converted near-full-text derivatives under `raw/converted/` or beside webpage snapshots.
- Put webpage snapshots under `raw/webpages/<source>/`.
- For WeChat articles, prefer saving `.html`, extracted `.md`, and `.meta.json` together.

## 平台本地约定：raw/platform/ 镜像区（本 KB 实例特有）

- `raw/platform/` 是**派生镜像区**：平台权柄文档（架构/规约/契约/台账/编排/注册表/AGENTS.md）
  由 `control-center/scripts/kb-sync.sh` 从各 Git 仓镜像生成（FINDING-051）。
- 镜像文件首行携 `kb-mirror: upstream=<路径> sha256=<哈希>` 出处头——**勿手改镜像**
  （改动会被下次 sync 覆盖）；要改就改 upstream 上游文件，再重跑 kb-sync.sh。
- 镜像清单唯一来源 = `control-center/orchestration/reconcile/checks.yaml` 的
  `mirror_pairs` 块；`control-api reconcile` 的 `kb-mirror-freshness` 检查盯新鲜度，
  镜像过期报 WARN。新增权柄文档须在该块登记，否则不在检索面。

## Wiki Rules

- Put one-source notes under `wiki/source-notes/`.
- Put reusable concepts under `wiki/concepts/`.
- Put tradeoff pages under `wiki/comparisons/`.
- Put technical judgments under `wiki/decisions/`.
- Do not store raw full-text copies in `wiki/`.

## Note on Karpathy's Original Design

In Karpathy's original llm-wiki, **Overview / Entities / How-to / Timelines** are
*sections inside each article*, not subdirectories. PieKBS uses
`source-notes/concepts/comparisons/decisions/` as the four core subdirectories,
which is a community evolution better suited to knowledge-base use cases.

Gap analysis reports (`piekbs synthesize --gaps`) are generated artifacts and
belong in `index/gaps/`, not in `wiki/`.

## Phase 1 Minimum

```text
piekbs-kb/
  raw/
  wiki/
  schema/
```

`index/` can be absent until CLI indexing exists.
