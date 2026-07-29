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
- **二 · PRODUCT HUNT（先判断，再对答案）**——深一档的纸，留白大，只有一个产品

PH 区分两层，**「先看懂，再判断」是整个设计的核心**：

```
0 · 先看懂                          全部公开，一个字都不涂黑
   它是什么（中性功能描述，不用 maker 的营销语言）
   给谁用
   3-4 张界面图，每张配一句「这张在演什么」
   品类 · 票数
        ↓
1 · 四问                            题面材料公开，只涂黑「判定」
   ① maker 把这个功能框成了什么问题？
   ② 三层（tagline / 首评划界句 / 首图）讲的是同一句话吗？
   ③ 它按什么收钱？价值单位是什么？
   ④ 这几条评论里哪几条是真信号？（原文含噪音，做筛选题）
```

**这一版是真人试用两轮反馈改出来的，原设计错在两处**：

1. 第一版题面只给一张图 + 四个问题。真人反馈「一张图看不出什么」。数下来发现**四问里有三问
   在题面根本没有材料**——② 让你判断三层是否一致却只给一层，③ 定价信息在界面截图里根本不存在，
   ④ 评论一条都没露。那不是训练，是蒙
2. 补了材料后仍然不行，因为**「理解」这一步整个被跳过了**。四问全是分析题，而分析的前提是先看懂
   产品是什么。于是加了第 0 层

第 0 层和 ① 之间没有泄题关系：**「它做什么」和「他把它框成什么问题」是两回事**。前者客观（按你
定义的标准打分），后者是 maker 的叙事选择（你的 eval 只测了你挑的输入，真正的失败藏在没人看过的
运行里）。同一功能能框成很多种问题，他挑了哪种、用什么危机感包装，才是要练的东西。**中性描述读
起来平淡、maker 的叙事让同一件事变得紧迫**——这个落差就是 ① 的全部价值。

涂黑用 `checkbox + label` 纯 CSS 实现，无 JS，键盘可达，`prefers-reduced-motion` 生效。
判定按词/句包 `<span class="w">`，未解密时 `color: transparent; background: var(--ink)`。
理由和旁证放 `.more`，解密前 `display: none`——只涂黑判定是不够的，引文和图片本身会指向答案。

**图片的三个入口**（真人试用逐轮逼出来的，每一条都对应一次实际失败）：

1. 点图 → **站内浮层**（checkbox + label，无 JS）。浮层里图按 `&w=1600` 铺开可拖动——
   只做 fit-to-screen 在手机上跟内联一样大，等于没放大
2. 浮层顶栏「✕ 关闭」→ 滚动位置和已解密状态都不丢。**跳走会全丢，这是做浮层的唯一理由**
3. caption 里「原图 ↗」→ 真要极限放大才跳走，也是浮层在某个内置浏览器失灵时的兜底

踩过的三个坑：

- **`target="_blank"` 被内置浏览器拦掉**，症状就是「点了没反应」。一律不加
- **横滑 carousel 同时损害两个目标**：图只剩 82% 宽度（小字更读不清）＋ 触屏上点击常被判成
  拖动手势。改竖排满宽，两个问题一起消失
- **`.lb-toggle:checked ~ .lb` 必须写成 `+`**：`.lb` 全是 `.gallery` 的同级子元素，`~` 会
  一次显示后面所有浮层、最后一个盖在最上面——点第一张图却看到第三张

**图片分工**（从真实数据里得出，PH gallery 通常混着三种图）：

| 类型 | 放哪 | 理由 |
|---|---|---|
| 界面/功能图 | 第 0 层 gallery | 「它干什么活」的证据 |
| 问题叙事图（`THE PROBLEM` 类） | ① 的 `.more` | 图上直接印着 ① 的答案 |
| 品牌封面图（多为 `media[0]`） | ② 的 `.given` | 是「首图」那层的证据，② 本就公开三层 |

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
