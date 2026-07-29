# daily-radar

每日 GitHub Trending + Product Hunt → 生成 H5 页面（GitHub Pages）→ 飞书推一条链接。

三个栏目：GitHub 段（官方 Trending 上的热门 AI 项目，非 AI 已过滤）和论坛热议段（Reddit + HN 每天 10 条「大家在吵什么」，AI 优先）是消费型扫读；Product Hunt 段是练习型（题目式：先给首屏图猜，折叠区翻答案），用来训练产品感觉。视觉是 Apple 液态玻璃风格。

## 状态

**已上线**。云端 routine 每天 20:00（北京）运行；触发器里只放一条短引导 + 三个密钥，运行时从本仓库拉最新 `routine/prompt.md` 执行——改 prompt 只需推仓库，不用动 routine 配置。第 1 期（2026-07-28，PH 榜首 Prefactor）已发布：根页、`/2026-07-28/`、`data/2026-07-28.json` 齐全，期号从 2 续。

- 设计文档：`docs/2026-07-29-daily-radar-design.md`
- routine prompt：`routine/prompt.md`（密钥是占位符，真实值只在 routine 配置里）
- 日报页模板：`templates/day.html`
- 归档日历页：`/archive/`，模板 `templates/archive.html`，routine 每天重建

## 与 github-trending-daily 的关系

互相独立。`github-trending-daily` 的云端 routine 继续跑（每天早上飞书纯消息推送），本项目并行运行一段时间后再决定是否停掉旧的。
