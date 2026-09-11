---
name: generate-travel-book
description: 把用户自己的旅行资料（航班、酒店、每日安排、门票等）套进 template/index.html，生成一份可以直接发给朋友、随身带着走的单文件旅行手册网页。Use when the user provides their own trip details and wants a travel itinerary web page generated from this template.
---

# Travel Book 生成指南

这份 SKILL.md 和 `template/index.html` 配套使用。把这个仓库整个交给你自己的 AI（Claude Code、Cursor
之类能读写文件的工具都行），再把这次旅行的资料发给它，让它照着下面的规范把 `template/index.html`
改写成这次旅行专属的页面——**改的是内容，不是结构**，`template/index.html` 里已经写好的四种交互组件
（见第 2 节）直接复制粘贴改文字就行，不用重新设计。

生成完是一个**单文件、零依赖**的 HTML，用手机浏览器直接打开就能看，怎么发布见第 5 节。

## 0. 开始前，先问清楚这几件事

- 目的地、日期、人数、进出交通方式（航班号/车次，几点到几点）
- 已经订好的机票/租车/酒店/门票——发确认信息过来，没订好的先留"待定"
- 每天大致想去哪、按什么顺序

资料不全不用等，缺的地方直接标"待定"生成第一版，之后随时可以回填。

## 1. 页面结构（自上而下，固定顺序，别打乱）

1. **Hero**：emoji + 标题、日期/人数/进出交通一行小字、状态徽章（机票/酒店 n/n/门票这类）。
2. **NOW 卡片**（紧跟 Hero）：
   - 脉冲圆点 + "此刻关注 · NOW"
   - 自动算出的**下一个行程事件**标题 + 一行 meta
   - 大号等宽倒计时，每秒跳动
   - 底部一行"当前待办" = 清单里第一条未勾选项，点击跳到待办清单
   - 由页面底部 `EVENTS` 数组驱动，旅行全程自动推进，不用每天手动改"现在是第几天"
3. **顶部锚点导航**：总览 / 航班 / 每日行程 / 待办清单 / 贴士，够用就行，模块不全就删掉对应的锚点。
4. **行程总览**：路线整体是怎么走的，简单路线图或者文字图例都可以，见第 3 节。
5. **航班/交通卡片**：时间、班次号、大概时长。**确认号只写"见邮件"，不要写在页面里**——
   一旦发布出去有链接，任何拿到链接的人都能看到，确认号+姓氏很多时候就能改签别人的票。
6. **每日行程卡**：左边时间列 + 右边内容，需要预约/买票的标签提醒，亮点项目标出来，
   底部一行"今晚住哪"。
7. **待办清单**：分组列 + 打勾，勾选状态存 `localStorage`，已确认的事项预先锁定为完成状态
   （见第 2.4 节 `data-locked`），顶部放进度条。
8. **贴士**：天气、排队预期、交通提醒收尾。

## 2. 已经写好的四种交互组件（复制粘贴，不用重新发明）

`template/index.html` 里已经有一份可运行的示例 Day 卡，演示了下面这四种模式：

### 2.1 `.pin` —— 点地名跳导航

```html
<a class="pin" href="https://www.google.com/maps/search/?api=1&query=Osaka%20Castle" target="_blank" rel="noopener">大阪城</a>
```

用 Google 地图的搜索链接就够，不用接地图 SDK。没有固定地址就用搜索词（比如店名），
Google 会自动解析到最近的一家。

### 2.2 `.info-btn` / `.info-panel` —— 点 ⓘ 展开地点简介

```html
<a class="pin" href="...">道顿堀</a>
<button class="info-btn" type="button" data-target="info-dotonbori" aria-expanded="false" aria-label="道顿堀简介">ⓘ</button>
<div class="info-panel" id="info-dotonbori" hidden><p>两三句考证过的简介：历史/规模/值得看的细节。</p></div>
```

`id` 和 `data-target` 要一一对应且全页唯一。简介写 2~4 句真实信息，不是随便编。

### 2.3 `.dm-toggle` / `.dm-frame` —— 当天完整路线地图（点了才加载）

```html
<button class="dm-toggle" type="button"
  data-src="https://maps.google.com/maps?saddr=起点&daddr=途经点1+to:终点&output=embed">🗺️ 展开今日路线地图</button>
<a class="dm-go" href="https://www.google.com/maps/dir/?api=1&origin=起点&destination=终点&travelmode=driving&waypoints=途经点1%7C途经点2" target="_blank" rel="noopener">Google 导航 ↗</a>
```

多途经点路线用 `waypoints=A%7CB%7CC`（`%7C` 是竖线 `|` 的编码）。不预加载，点了才灌 `<iframe>`，
避免每天都加载地图拖慢首屏。

### 2.4 `.day-toggle` / `collapsed` —— 每日行程卡折叠

点 `.day-head` 整卡折叠/展开，JS 已经写好了。**某一天真正过去之后**，给对应的
`<div class="card day">` 加上 `collapsed` class（`<div class="card day collapsed">`），
页面默认打开时只留正在进行和接下来的天数在眼前，不用每次都手动展开。

### 2.5 待办清单的 `data-locked`

已经确认/订好的事项，给 `.todo` 加 `data-locked="1"`：显示为固定打勾状态，不受用户本地
勾选影响、也不占用 `localStorage`。没锁定的正常项用户自己点了才算勾选，状态存本机。

### 2.6 待办清单：可选的跨设备/跨好友同步

默认待办清单是纯本地的（`localStorage`，只在当前浏览器生效）。`template/index.html` 底部预留了
一个 `SYNC` 配置对象和一段"大家一起加"的共享待办区（可以自由添加、编辑、删除、勾选），默认关闭。

用户如果明确要求"发出去后不同人能互相同步待办/协作编辑"这类效果，按
`optional/supabase-sync/README.md` 的步骤配一个免费 Supabase 项目（建表 + 拿连接信息），
再填好 `supabaseUrl`/`supabaseAnonKey`/`tripId` 三项即可，不需要改其他代码——`SYNC.enabled`
默认就是 `true`，但只要 `supabaseUrl`/`supabaseAnonKey` 还是空字符串就不会真的发起连接，
页面会一直安全地停留在本地模式。没有同步需求就不用管这三项，保持空字符串即可。

## 3. 地图画法（可选，进阶）

`template/index.html` 里默认放的是一段极简的路线示意 SVG（几个点连线），大多数情况下这样就够用，
不用非画一张精确地图。

如果想要更像样的路线/轮廓地图：

- 等距圆柱投影：`x=(lon-lonMin)*S*cos(中心纬度)`，`y=(latMax-lat)*S`
- 手工挑数十个边界经纬点画轮廓，城市/途经点用真实经纬度投影
- **写一个 Python 脚本批量算好坐标再贴进 SVG**，不要在页面里用 JS 运行时现算
- 完成后截图检查浅色/深色模式下标签有没有被裁切或重叠

这一步比较耗时间和 token，资料简单、天数不多的旅行可以跳过，用文字图例代替。

## 4. 行程编排原则

- **以已经订好的交通时刻为准**往前往后排，宁可整体平移也不要硬塞——
  下午落地就当晚安排就近的项目，清晨航班就前一晚玩完、住机场方向。
- 长途路程标注中途休息点，避免深夜赶路。
- 核对沿途景点的开门时间、闭馆日、是否需要提前订时段票（很多免费景点也要）。
- **热门项目查一下会不会卖光，卖光了给替代方案**，别到了现场才发现进不去。
- 没订好的酒店/门票先在行程卡放"待定"、进待办清单，订好了回填。

## 5. 怎么发布

生成的是一个纯静态单文件，随便挑一种：

- **GitHub Pages**（最省事，免费）：把这个仓库 fork 一份，改完 `template/index.html` 推到
  `main`，在仓库 Settings → Pages 里把发布源指到这个文件所在目录即可。
- **Cloudflare Pages / Workers**：连接 GitHub 仓库，push 即自动部署，速度更快。
- **Netlify / Vercel**：拖进去一个文件夹就能上线。
- 也可以完全不部署，改完直接把这个 HTML 文件用微信/邮件发给同行的人，手机打开就能看。

**发布前检查一遍页面里有没有确认号、门锁密码这类敏感信息**——只要有公开链接，任何拿到链接的人
都能看到整份行程。

## 6. 旅行进行中怎么维护

旅行开始后，用户可能会随时说"这个订好了""改去另一个地方了""今天已经过去了"。这些情况回去改
`index.html`（不用整页重写，只改对应位置）：

- **二选一定案了**：删掉落选项（连同它的 `.pin`/`.info-btn`/`.info-panel`），保留项加一个
  `<span class="tag star">✓ 已吃/已访问</span>`；`EVENTS` 数组、当日路线概述、地图链接里
  凡提到过落选项名字的地方一并改掉，改完搜一遍确认没有残留。
- **预约/门票确认了**：行程卡里的标签从"需约/可选"改成"✓ 已预约/已访问"；对应待办项加
  `data-locked="1"`。
- **一天过去了**：给对应 `.card.day` 加 `collapsed`（见 2.4）。

**页面永远保持能直接读的最新状态**：不要在页面里放修改日志、"这次改了什么"之类的版本说明，
变更说明写在 git 提交信息里就够了——页面任何时候打开都是一份能直接看的最新版本。
