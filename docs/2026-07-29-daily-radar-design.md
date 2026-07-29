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
├── templates/day.html      日报页模板，routine 读取后填槽位
├── templates/archive.html  归档日历页模板
├── archive/index.html      归档日历页（routine 每天重建）
├── data/YYYY-MM-DD.json    结构化数据（去重 / 对撞 / 回看 / 归档日历的数据源）
└── YYYY-MM-DD/index.html   当期归档
```

模板放仓库而不是塞进 prompt：样式可以独立迭代，不用改 routine 配置。

### 页面设计

**两个标签页，不是上下两段。** GitHub Trending 和 Product Hunt 是两件无关的事，阅读模式也不同
（扫读 vs 精拆）。上下排版会暗示先后和层级，让 GitHub 段变成「翻过去才能开始训练」的前言——
这是真人反馈指出来的。标签页让两者平级、各自独立，默认落在 Product Hunt（这个项目存在的理由）。
纯 CSS radio 实现，无 JS。

- **Product Hunt** —— 一整块玻璃卡片，留白大，只有一个产品。第 0 层 + 四问
- **GitHub Trending** —— 每个项目一张玻璃卡片：`owner/name`、今日涨星（绿色胶囊）、语言、
  总 star、建库时间、连续上榜天数、三段式点评

**GitHub 段用三段式，对齐 `github-trending-daily` 那份日报的信息密度**（真人拿两边对比后指出
一句话点评是降级）。顶部一句当日摘要 + 每个项目三段：

- **项目定位** —— 一句话说清它是什么、给谁用
- **核心功能** —— **必须读过 README 再写**，具体到技术选型和能力清单。
  「React Three Fiber + WebGPU，Turborepo 拆成 core / viewer / editor / nodes 四个可独立安装的
  npm 包」这种密度才算合格，「功能强大、生态丰富」一律不要
- **值得关注** —— 写**从 README 里挖出来、扫榜看不到的东西**。三个真实例子：
  - `aisuite` 的桌面产品 OpenWorker 已拆到独立仓库，README 标注本仓库 `platform/` 将来会删——
    等于它正在把自己收回成纯底层库
  - `affaan-m/ECC` 建库半年 23.5 万星、3.6 万 fork，和 6 个月年龄对不上；README 顶部还挂着
    「只从官方渠道安装，第三方镜像可能带恶意代码」的警告
  - `jenkinsci/jenkins` 是榜上唯一的十五年老项目，+180 属日常波动——**没查清就写没查清，
    不要替它编一个上榜理由**

meta 行带上**建库时间**，一眼分得出「十五年老项目」和「半年新库」，这是旧日报没有的信息。

### 视觉风格：Apple 液态玻璃（2026-07-29 重构，用户指定）

按 emilkowalski/skills 的 `apple-design` + `emil-design-eng` 两份规则执行：固定渐变色场垫底
（玻璃需要环境色可折射；不用 `background-attachment: fixed`，iOS 不生效）、玻璃卡片
（backdrop-filter + saturate + 顶部高光内描边）、sticky 玻璃分段控件（iOS 抽屉曲线滑块）、
动效只动 transform/opacity、入场一律 ease-out 且 <450ms、`prefers-reduced-motion` 全部瞬时。

一个踩过的坑：**backdrop-filter 会给 fixed 后代建立 containing block**。`.drill` 加了模糊后，
里面浮层的 `inset: 0` 变成相对卡片而不是视口（遮罩盖不住报头）。修法：`.drill` 不用
backdrop-filter，提高背景不透明度补偿——背景是平滑固定渐变，模糊与否视觉无差。

### 归档日历页（/archive/，2026-07-29 加入）

页面自己承诺「30 天后往上翻，看命中率」——预测复盘要求能翻回任意一天，页脚只放最近几期
走不通这个闭环。

- 独立页面，不嵌进日报（日报保持轻）；日报页脚 = 最近 3 期胶囊 + 「全部 →」
- 月历格子，纯 CSS 无 JS：有刊 = 可点的玻璃胶囊（最新一期带 tint 描边），断更 = 虚线圈，
  首刊前 / 未来 = 淡灰
- 数据源 = `data/` 文件名列表——步骤 5 本来就要 GET 它算期号，日历零额外请求
- routine 每天第 4 个 PUT 重建（覆盖带 sha）；只有归档页失败不影响日报推送
- 日历上的日期 = PH 榜单日 D，和页面路径、飞书链接是同一个日期

顺带修了一个链接 bug：同一份日报 HTML 发根页和日期页两处，页脚往期链接原来写相对路径
`../YYYY-MM-DD/`，从根页出发会跳出站点。契约已改为绝对路径 `/daily-radar/YYYY-MM-DD/`。

样张已转正为第 1 期：`/2026-07-28/`（**按 PH 榜单日归档**——发到 07-29 会和明天 routine 的
D=07-29 撞路径，routine 按新文件不带 sha 去 PUT 会 422）+ `data/2026-07-28.json`（评论原文
未存档，`comments_raw` 留空注明）。期号从 2 续。

### 一个待决事项：GitHub 段要不要留

`github-trending-daily` 的云端 routine 仍在每天早上推 GitHub 日报。两边内容并非重复——
**GitHub Trending 的 24h 窗口是滚动的**，早上 09:00 和晚上 20:00 抓到的榜差别很大（实测同一天
早上榜首是 `bradautomates/claude-video` +989，下午前五里根本没有它）。所以这是同一份榜的两个
不同快照，不是同一份内容发两遍。

但一天两次扫同一个榜是否有价值，由使用者判断。若判定为噪音，就砍掉本项目的 GitHub 段、
只留 Product Hunt 训练，项目更聚焦。

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

1. 点图 → **站内浮层**（checkbox + label，无 JS）。浮层里 **fit-to-screen 居中，
   整张放得下、不用拖**。实测 1440×813 下渲染 1302×733，控制台每个数字都读得清
2. 浮层顶栏「✕ 关闭」→ 滚动位置和已解密状态都不丢。**跳走会全丢，这是做浮层的唯一理由**
3. 顶栏和 caption 的「原图 ↗」→ 逐像素看时才跳走，手机上那条给的是原生捏合缩放

第 1 条改过一次：一开始让浮层里的图按 1600px 铺开、靠拖动看细节，真人反馈「放太大了，
还要拖动」。**桌面视口本来就有 1400px，fit-to-screen 已经接近 1:1，拖动纯属多余**；
手机上 fit 确实等于没放大，但那个场景交给「原图 ↗」的原生捏合缩放更好，不该自己造拖动。

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
