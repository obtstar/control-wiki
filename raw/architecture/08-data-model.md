# 08 数据模型（MySQL DDL/DML）

以 **MySQL** 存储运行时数据（任务、日志、报告、文档索引元数据）。DDL/DML 脚本版本化存放在**平台数据库代码仓库（`control-db`）**中（与代码同源、可审计），不放在控制中心文档仓库。

> **仓库注册表与执行节点登记不在 MySQL**：以 Git 文件形式存于控制中心仓库 `registry/repos.yaml` 与 `registry/executors.yaml`（见 [14 多仓库管理](14-multi-repo.md)），Git 即权威源。

## 核心表结构（DDL）

```sql
-- 1. 任务表 task
CREATE TABLE `task` (
  `id`            BIGINT       NOT NULL AUTO_INCREMENT,
  `task_no`       VARCHAR(40)  NOT NULL COMMENT 'TASK-20260802-001',
  `type`          VARCHAR(20)  NOT NULL COMMENT 'FEATURE / BUGFIX / DOC',
  `title`         VARCHAR(200) NOT NULL,
  `status`        VARCHAR(32)  NOT NULL COMMENT '需求分析/系统设计/编码实现/测试验证/待合并/交付/已回退/已暂停',
  `repo_key`      VARCHAR(64)  DEFAULT NULL COMMENT '目标代码仓库（registry/repos.yaml）',
  `branch`        VARCHAR(128) DEFAULT NULL COMMENT 'feature/task-001',
  `worktree_path` VARCHAR(255) DEFAULT NULL COMMENT '~/wt/repo-a/TASK-001-xxx',
  `priority`      TINYINT      DEFAULT 3,
  `created_at`    DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at`    DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_task_no` (`task_no`),
  KEY `idx_status` (`status`),
  KEY `idx_repo` (`repo_key`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='任务表';
```

```sql
-- 2. 工作记录表 work_log（操作追溯）
CREATE TABLE `work_log` (
  `id`                BIGINT      NOT NULL AUTO_INCREMENT,
  `timestamp`         DATETIME(6) NOT NULL,
  `task_id`           BIGINT      NOT NULL,
  `operator`          VARCHAR(64) NOT NULL COMMENT 'owner / agent / agent@pc-01（执行节点）',
  `action`            VARCHAR(64) NOT NULL COMMENT 'code_generate / merge_confirm / task_pause ...',
  `repo_key`          VARCHAR(64)  DEFAULT NULL COMMENT 'registry/repos.yaml 中的 repo_key',
  `branch`            VARCHAR(128) DEFAULT NULL,
  `worktree_path`     VARCHAR(255) DEFAULT NULL,
  `git_commit`        CHAR(40)     DEFAULT NULL,
  `model_used`        VARCHAR(64)  DEFAULT NULL COMMENT '经 LiteLLM 路由',
  `token_count`       INT          DEFAULT NULL COMMENT '本次操作 token 用量（任务级熔断依据）',
  `before_state_hash` CHAR(64)     DEFAULT NULL,
  `after_state_hash`  CHAR(64)     DEFAULT NULL,
  `prev_hash`         CHAR(64)     DEFAULT NULL COMMENT '上一条记录的 after_state_hash，形成哈希链',
  PRIMARY KEY (`id`),
  KEY `idx_task` (`task_id`),
  KEY `idx_operator` (`operator`),
  KEY `idx_time` (`timestamp`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='工作记录（操作追溯）';
```

```sql
-- 3. 文档管理表 doc_management（仅内部设计与代码同库，概要/外部设计集中于控制中心）
CREATE TABLE `doc_management` (
  `id`          BIGINT      NOT NULL AUTO_INCREMENT,
  `doc_no`      VARCHAR(64) NOT NULL COMMENT 'DSGN-001',
  `doc_type`    VARCHAR(32) NOT NULL COMMENT '概要设计/外部设计/需求/内部设计',
  `repo_key`    VARCHAR(64) DEFAULT NULL COMMENT '来源仓库：控制中心或代码仓库',
  `repo_path`   VARCHAR(255) DEFAULT NULL COMMENT '仓库相对路径 docs/design/...',
  `source_commit` CHAR(40)  DEFAULT NULL COMMENT '版本化来源 commit',
  `content_hash` CHAR(64)   DEFAULT NULL COMMENT '内容摘要，用于增量索引',
  `vector_status` VARCHAR(16) DEFAULT 'PENDING' COMMENT 'PENDING/INDEXED/FAILED',
  `updated_at`  DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_doc_no` (`doc_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='设计控制文档：概要/外部设计在控制中心，内部设计随代码仓库';
```

```sql
-- 4. 工作报告表 work_report（日报/周报/任务报告，由 work_log 聚合生成）
CREATE TABLE `work_report` (
  `id`           BIGINT       NOT NULL AUTO_INCREMENT,
  `report_no`    VARCHAR(64)  NOT NULL COMMENT 'RPT-20260802-001',
  `report_type`  VARCHAR(16)  NOT NULL COMMENT 'DAILY / WEEKLY / TASK',
  `task_id`      BIGINT       DEFAULT NULL COMMENT 'TASK 型报告的关联任务',
  `operator`     VARCHAR(64)  NOT NULL COMMENT '报告主体：agent-xxx / owner',
  `period_start` DATE         DEFAULT NULL,
  `period_end`   DATE         DEFAULT NULL,
  `summary`      TEXT COMMENT '汇总摘要（LLM 基于 work_log 生成）',
  `detail_json`  JSON         DEFAULT NULL COMMENT '结构化明细：任务/动作/模型/耗时/commit 列表',
  `status`       VARCHAR(16)  DEFAULT 'GENERATED' COMMENT 'GENERATED / ARCHIVED',
  `created_at`   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at`   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_report_no` (`report_no`),
  KEY `idx_operator_period` (`operator`, `period_start`, `period_end`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='工作报告（自动汇总）';
```

## 工作日志与工作报告落地

| 数据 | 位置 | 机制 |
|-----|------|------|
| **工作日志（work_log）** | MySQL `work_log` 表 | `service/audit` 每次 Agent/用户操作实时写入，含任务/执行身份/模型/commit/状态哈希 |
| **工作报告（work_report）** | MySQL `work_report` 表 | 定时任务（`service/report`）按日/周/任务聚合 `work_log` → LLM 生成摘要 → 落库，Web 端可查 |

### 工作报告生成流程

```
work_log（MySQL）──定时聚合──→ service/report 组装明细（detail_json）
    → LLM（经 LiteLLM，模型 cheap/coding）生成 summary
    → 写入 work_report（GENERATED）→ Web 端查看
```

## 追溯要求

- 日志结构化存储于 MySQL `work_log`，禁止物理删除（软删除/归档），支持导出
- 操作绑定任务与执行身份（owner / agent / agent@{executor_id}），`before/after_state_hash` 保证前后状态可校验
- **防篡改**：`prev_hash` 串联上一条记录的 `after_state_hash` 形成哈希链，任何中间记录被改/删都会导致链条断裂，可定期校验
- DDL/DML 版本化入库 `control-db` 代码仓库，变更走 MR
