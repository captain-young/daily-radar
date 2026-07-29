# daily-radar 设计文档

日期：2026-07-29
作者：杨新威 (xinwei)

## 目标

每天生成一份 H5 简报：上半是 GitHub Trending（扫读今天出了什么），下半是 Product Hunt 榜首的
**题目式拆解**（先给首屏图和四个问题让你猜，答案以「涂黑待解密」的形式在页面上，点击才显形），
用来训练产品感觉。飞书推一条链接。

先跑 30 天，到期复盘是否续、是否降频。

## 形态决策（已确认）

| 维度 | 决策 |
|------|------|
| 运行方式 | Claude 云端定时 Agent（routine），跑在 Anthropic 云上 |
| 触发时间 | 每天 20:00 Asia/Shanghai = cron `0 12 * * *`（UTC） |
| 页面托管 | GitHub Pages，仓库 `captain-young/daily-radar`（public） |
| 页面地址 | 最新一期 `https://captain-young.github.io/daily-radar/`，当期归档 `/YYYY-MM-DD/` |
| 数据源 A | `https://github.com/trending?since=daily`（爬 HTML，无官方 API） |
| 数据源 B | Product Hunt GraphQL API `https://api.producthunt.com/v2/api/graphql` |
| 数量 | GitHub 5 个（扫读）+ PH 1 个（榜首，精拆） |
| 状态存储 | 仓库自身。`data/YYYY-MM-DD.json` 存结构化数据，routine 读历史做去重和周日对撞 |
| 发送通道 | 飞书自定义机器人 Webhook，一条消息带页面链接 |
| 预测记录 | 人工回复在飞书那条消息底下（routine 只能发不能读），30 天后自己往上翻 |
| 与旧项目关系 | 与 `github-trending-daily` 完全独立，旧云端 routine 继续跑，并行一段时间后再定 |

## 关键约束

### 1. 密钥绝不能进仓库

仓库是 **public**。`routine/prompt.md` 里所有密钥一律写占位符，真实值只存在 Anthropic
routine 的配置里。三个密钥：

| 占位符 | 内容 | 本机备份位置（keychain） |
|--------|------|------------------------|
| `<<PH_TOKEN>>` | PH developer token，不过期 | `ph-developer-token` |
| `<<GH_PAT>>` | GitHub fine-grained PAT，仅 `daily-radar` 的 Contents: RW | `daily-radar-gh-pat` |
| `<<FEISHU_WEBHOOK>>` | 飞书自定义机器人 webhook URL | 待提供 |

### 2. PH 日榜边界与夏令时

PH 日榜按太平洋时间午夜结算。实测 `featuredAt` = `2026-07-28T07:01:00Z`，即夏令时（PDT，
UTC-7）下边界是 **07:00 UTC**；冬令时（PST，UTC-8）会变成 **08:00 UTC**。

**不要硬编码小时偏移。** 做法：查询最近 48 小时的 posts，按 `featuredAt` 的日期分组，
取**次新**的那一组——最新那组是仍在进行中的当天，票数没结算完。

这条不是理论问题：2026-07-28 榜首 Prefactor 在 PT 当天 18:00 时是 134 票，结算后是
**580 票，差 4.3 倍**。进行中的票数完全不可用。

### 3. 抓取通路和失败模式正好相反

| | GitHub Trending | Product Hunt |
|---|---|---|
| 唯一通路 | 爬 HTML（无官方 API） | GraphQL API（网页被 Cloudflare 403 挡住，爬不了） |
| 失败模式 | HTML 结构变 → 静默解析失败 | token 失效 / 限流 → 整体中断 |
| 依赖密钥 | 无 | 必须有 |

### 4. PH API 实测事实

- 限流：`x-rate-limit-limit: 6250`，`reset` 约 900 秒。**按查询复杂度计费，不是按请求数**
  （一次取 3 个产品含 media + comments ≈ 95 分）。每天几百分，用不完
- 图片 **不防盗链**：无 Referer / 第三方 Referer 全部 200，H5 可直接 `<img src>` 引用
- imgix 支持 `&w=800`，48KB → 22KB，首屏图用这个尺寸
- `comments.body` 返回 **HTML**（`<p>` / `<strong>`），入页面前必须清洗，不能裸插
- `website` 字段是 PH 跳转链（`producthunt.com/r/XXXX`），**不是官网直链**。要拿定价信息
  得跟一跳到真实官网
- 条款禁止商业用途，个人日报属 fair use。每天只抓 1 次，不轮询

## 架构

```
[cron 0 12 * * * UTC = 北京 20:00]
      │
      ▼
[Claude 云端 routine]
      │
      ├─ 1. 定窗口：查 PH 最近 48h，按 featuredAt 分组，取次新那天 = D
      ├─ 2. curl github.com/trending?since=daily → 解析 5 个项目
      ├─ 3. PH GraphQL 取 D 当天 order:VOTES 榜首 → tagline/media/comments/topics
      ├─ 4. 跟一跳 website 拿真实官网 → 读定价页找价值单位
      ├─ 5. GET contents/data/ 读历史 → GitHub 项目连续上榜天数；周日取本周同品类
      ├─ 6. 生成 HTML：GET contents/templates/day.html 填槽位
      ├─ 7. PUT contents ×3：data/D.json、D/index.html、index.html
      └─ 8. curl POST 飞书 webhook → 一条消息带链接
```

### 仓库结构

```
daily-radar/
├── index.html              最新一期（每天覆盖）
├── docs/                   本文档
├── routine/prompt.md       自包含 routine prompt（密钥用占位符）
├── templates/day.html      页面模板，routine 读取后填槽位
├── data/YYYY-MM-DD.json    结构化数据（去重 / 对撞 / 回看）
└── YYYY-MM-DD/index.html   当期归档
```

模板放仓库而不是塞进 prompt：样式可以独立迭代，不用改 routine 配置。

### 页面设计

一份「简报」，两个区，视觉上是两份不同的文件：

- **一 · GITHUB TRENDING（扫读）**——浅色纸，密排。每行 `owner/name`、今日涨星（右对齐等宽）、
  语言、总 star、连续上榜天数、一句中文点评
- **二 · PRODUCT HUNT（先猜后翻）**——深一档的纸，留白大，只有一个产品

PH 区的顺序是强制的训练结构：

```
首屏图（media[0] + &w=800，不带任何说明）
  ↓
品类 tags · 票数
  ↓
四问：① 堵哪道缝 ② 三层表达一致吗 ③ 按什么收钱 ④ 评论区谁在用自己的数字追问
  ↓
四个独立的「涂黑」答案块，一问一块，各自点击解密
```

**签名元素：涂黑条的宽度就是真实文字的宽度。** 实现是把真实文本按词包 `<span class="w">`，
未解密时 `color: transparent; background: var(--ink)`。所以解密前你就能看出「这个 tagline 是
6 个短词」——而这本身就是那节课的内容（好标题都 ≤10 词、零形容词）。

解密用 `checkbox + label` 纯 CSS 实现，无 JS，键盘可达，`prefers-reduced-motion` 生效。

周日多渲染一块**同类对撞**：本周同品类的两个产品并排，看垂直 vs 横向、tagline 怎么互相避让。

### 错误处理

- GitHub HTML 解析出 < 5 个：发已拿到的，页面和飞书消息都标注「今日 GitHub 段解析异常」
- PH API 报错 / token 失效：GitHub 段照发，PH 段整块换成故障说明。**不要静默跳过**
- `PUT contents` 失败：飞书消息里直接说页面没更新，不要发一个打不开的链接
- v1 无重试、无告警。故障靠每天晚上肉眼可见发现

## 测试 / 验证策略

已完成（2026-07-29）：

- [x] PH token 有效，一次查询拿到 tagline / votesCount / media / comments / topics / featuredAt
- [x] PH 图片不防盗链，imgix `&w=800` 生效
- [x] 仓库建好，Pages `built`，`https://captain-young.github.io/daily-radar/` 返回 200，
      不存在的路径正确 404
- [x] PAT 写权限：`PUT contents` 201 / `DELETE` 200（测试文件已清理）

- [x] 用真实数据渲染一期样张，headless Chrome 截图检查两个状态（未解密 / 已解密）
- [x] 响应式：探针实测 500px 和强制 390px 下均无元素溢出

**渲染验证抓出三个缺陷，都已修进模板和 prompt**——只读代码看不出来，必须真渲染：

1. **`media[0]` 是品牌封面图，印着 tagline 原文**，直接把「先猜」废掉。改成逐张看图挑真实
   界面截图，封面图降级为 ② 的证据
2. **中文按词切 span 后用空格 join**，解密后正文变成「测试环 境里 eval 全部通 过」。改成按
   标点切句、标点留在 span 外、span 间不加空格
3. **只涂黑结论不够**：引文和图片留在外面，等于白给答案（② 的封面图、③ 的「送 1,000,000
   free agent steps」）。改成两层，证据一律进 `.more`，解密前 `display:none`

附带：`.redact` 的文字仍在 DOM 里，全选或长按会抹出来 → 加 `user-select: none`。

待做：

- [ ] 在手机 / 飞书内置浏览器里真机点一遍，确认「猜 → 解密」手感和字号
- [ ] 飞书 webhook 收到带链接的消息，链接在飞书里点得开
- [ ] 创建 routine，`run now` 触发一次
- [ ] 确认无误后依赖每日 20:00 自动运行

## v1 明确不做（YAGNI）

- 预测记录进仓库（先靠飞书回复，30 天后再看要不要上）
- PH 取多个产品（1 个就够，内容太重必然不看）
- 失败重试 / 监控告警
- 停掉 `github-trending-daily` 的旧 routine（先并行观察）
- Cloudflare Pages 私有化（页面公开可接受，PH 和 GitHub 内容本身是公开的）

## 待用户提供

- 飞书自定义机器人的 webhook URL（关键词建议设 `daily-radar` 或 `雷达`，消息正文带上）
