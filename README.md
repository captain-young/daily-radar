# daily-radar

每日 GitHub Trending + Product Hunt → 生成 H5 页面（GitHub Pages）→ 飞书推一条链接。

GitHub 段是消费型（扫读今天出了什么新东西），Product Hunt 段是练习型（题目式：先给首屏图猜，折叠区翻答案），用来训练产品感觉。

## 状态

设计中。当前仓库只有占位页，用于验证 GitHub Pages 发布链路。

## 与 github-trending-daily 的关系

互相独立。`github-trending-daily` 的云端 routine 继续跑（每天早上飞书纯消息推送），本项目并行运行一段时间后再决定是否停掉旧的。
