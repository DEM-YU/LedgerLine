# ADR-002: 核心账本不变量下沉至 PostgreSQL（触发器、约束与 RLS）

状态：已采纳

## 背景

记账系统若仅依赖应用层校验，一旦遇到脚本直连、并发竞态、后台任务 bug 或中间件遗漏，就可能写入借贷不平、篡改历史记录或跨租户泄漏的数据。

## 选项

1. **仅在应用层（Service / Pydantic）校验**：实现简单、调试直观，但无法抵御并发交错写、旁路脚本或代码缺陷。
2. **数据库与双层防线（不变量由 PostgreSQL 强制，应用层仅做友好前置校验）**：在 PostgreSQL 中使用约束、触发器、`DEFERRABLE INITIALLY DEFERRED` 约束触发器和 Row-Level Security（RLS）兜底。

## 决定

采用选项 2：

- 账本不可变性（禁止 `UPDATE` / `DELETE` / `TRUNCATE`）、借贷平衡与 `line_count` 校验、至少两行且含一借一贷、禁止向 `closed` 期间入账、禁止同一分录二次冲销，均通过数据库约束与触发器强制执行。
- 租户隔离使用 `ENABLE` + `FORCE ROW LEVEL SECURITY` 绑定 `current_setting('app.org_id')::uuid`。
- 应用运行角色为非表所有者且不具备 `BYPASSRLS` 权限；所有 DDL 与触发器/策略通过 Alembic `op.execute` 以独立的 owner 角色执行。

## 后果

- 优点：即使应用存在 bug，数据库也不可能持久化不平衡分录或跨租户越权数据；每个不变量均可通过真实数据库负向测试验证。
- 代价与限制：迁移脚本、调试 SQL 报错和本地测试夹具搭建更复杂；所有集成测试必须针对真实 PostgreSQL 运行，无法使用内存 SQLite 替代。
