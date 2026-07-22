<div align="center">

# nowplaying-workers

**一个可分享的 Apple Music 实时"正在听什么"网页背后的 Cloudflare Worker——基于 Workers KV 的状态中继，外加一个实时 README 徽章。**

**语言 / Language:** [English](README.md) | **简体中文**

![Cloudflare Workers](https://img.shields.io/badge/%E8%BF%90%E8%A1%8C%E5%9C%A8-Cloudflare%20Workers-f38020)
![Free tier](https://img.shields.io/badge/%E6%88%90%E6%9C%AC-%E5%85%8D%E8%B4%B9%E9%A2%9D%E5%BA%A6%E5%AE%8C%E5%85%A8%E5%A4%9F%E7%94%A8-success)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](LICENSE)

[**在线效果 →**](https://yudaotor.github.io/nowplaying/?user=yudaotor)

</div>

## 仓库里有什么

| 模块 | 干什么的 |
|---|---|
| [`state-worker/`](state-worker) | 主中继：接收推送来的播放状态（`POST /push`），供网页读取（`GET /now`、`/history`、`/top-artists`），并用 Workers KV 支撑访客功能（留言墙/点赞/访客计数）——自带按 IP 限流，外加一个每日写入预算，保证访客流量挤占不了 KV 免费档的每日写额度 |
| [`badge-worker/`](badge-worker) | 把中继的数据渲染成一张实时更新的 SVG 徽章，可以嵌进任何 README |

（仓库里还有一个 [`worker/`](worker)——一个已退役的个人飞书签名集成留下的 302 跳转，纯粹因为旧链接还指着它才留着。几乎可以肯定不是你要找的东西。）

## 整体架构

```
Lyrimuse 的 Mac 采集器
  └─ POST /push(每次换歌/暂停/定期心跳) ──▶ 你的 state-worker(本仓库,Cloudflare Worker + KV)
                                                  │
                                                  ├─ GET /now、/history、/top-artists 等只读接口
                                                  │        ↓
                                                  └──▶ 你的网页(Yudaotor/nowplaying) ── 中继连不上时自动回退直连 ListenBrainz
```

| 仓库 | 角色 |
|---|---|
| [**Yudaotor/nowplaying**](https://github.com/Yudaotor/nowplaying) | 网页本体，外带一份可直接 fork 的模板（`demo/`） |
| **Yudaotor/nowplaying-workers**（本仓库） | 网页背后的中继和徽章 |
| **Lyrimuse** | 推送实时状态的 macOS 菜单栏歌词 App（暂未开源——见下方「前提」） |

- 采集器完全不知道网页/Worker 存不存在——它只是无条件往配置好的地址推一份状态。state-worker 挂了/没部署，采集器该干嘛还干嘛，不影响本机悬浮歌词。
- 网页优先读你自己的 state-worker；state-worker 本身连不上（或你压根没部署），网页会自动回退直连 ListenBrainz——这时能看到"正在播放"和历史，但留言墙/表情反应/访客计数/Top10 歌手这几个需要写 KV 的模块不会出现（它们没有 ListenBrainz 兜底路径）。
- 采集器侧只需要一份配置：`state_relay_url` + `state_relay_token`（对应 Lyrimuse 设置里"网页推送"卡片的"同步服务地址"+"访问令牌"）。两项填好，整条链路就跑起来了——没有另外的开关要找。

## 它撑起来的是什么

![播放中，背景色随封面自动变化，进度条+歌词逐字高亮](images/web-preview-playing.png)

一张实时的"正在播放"卡片，带逐字同步歌词、播放历史、留言墙、点赞、访客计数和 Top10 歌手榜——完整的功能导览和更多截图在[网页自己的 README](https://github.com/Yudaotor/nowplaying/blob/main/README.zh-CN.md)；上面这张截图就是作者自己在线上跑的真实页面 [yudaotor.github.io/nowplaying](https://yudaotor.github.io/nowplaying/?user=yudaotor)，不是摆拍数据。

## 部署一套自己的

### 前提

这条链路里负责实时推送的那一端——Lyrimuse macOS App 和它的采集器——**还没开源**。但这没有听起来那么碍事：

- 本仓库的所有东西 + 网页，不靠它也都能部署、能跑：网页会回退直连 ListenBrainz，只要你的收听记录已经通过任何一种 scrobbler 进了 ListenBrainz，"正在播放"和历史立刻就有。
- 留言墙、点赞、访客计数完全由你页面的访客驱动——跟采集器一点关系都没有。
- 采集器发布后额外带来的：秒级换歌更新、播放进度（网页同步歌词就靠它）、解析好的封面，以及每日 Top10 歌手推送。

### 第一步：部署自己的 state-worker

`state-worker/` 是一个 Cloudflare Worker（免费额度足够：KV 每天 1000 次写/10 万次读），采集器把当前状态推给它、网页从它这里读。已经搭好的话可以跳过这一节，直接 `cd state-worker && npm run deploy` 重新发布代码改动。从零搭一遍：

1. 注册 Cloudflare 账号。
2. （可选）把要用的域名的 DNS 托管迁移到这个 Cloudflare 账号——想用自定义域名才需要这一步；没有自己的域名也可以先用 Cloudflare 分配的 `*.workers.dev` 子域名，跳过这一小步和第 5 小步。
3. `npx wrangler login` 授权 CLI 登录这个账号。
4. 建 KV 命名空间：`npx wrangler kv namespace create NP_STATE`，把返回的 `id` 填进 `state-worker/wrangler.toml` 的 `[[kv_namespaces]]`。
5. 把 `wrangler.toml` 的 `[[routes]]` 域名改成你自己的域名（`custom_domain = true`，首次 `deploy` 时会自动绑定，前提是第 2 小步的 DNS 托管已经生效）。
6. 改 `[vars]` 的 `LB_USER` 和 `WEB_PAGE_URL`，分别改成你自己的 ListenBrainz 用户名、你自己的网页链接（走完下面「第二步：部署自己的网页」就有了）。
7. 设置两个 secret（值不提交进仓库）：
   ```bash
   cd state-worker
   npx wrangler secret put PUSH_TOKEN   # 采集器认证用，随机生成一串即可
   npx wrangler secret put ADMIN_TOKEN  # 站长手动删留言用，建议跟 PUSH_TOKEN 用不同的值
   ```
8. `npm run deploy`。

走完这一步，你会拿到：
- 一个能访问的 state-worker 地址（自定义域名或 `*.workers.dev` 子域名）
- 一个你自己生成的 `PUSH_TOKEN`（采集器认证用）

### 第二步：部署自己的网页

网页本体在它自己的仓库 [Yudaotor/nowplaying](https://github.com/Yudaotor/nowplaying)。从它的 [`demo/`](https://github.com/Yudaotor/nowplaying/tree/main/demo) 目录起步——那是专门给 fork 准备的模板：没有内置任何人的中继地址和社交元数据，`RELAY` 常量默认为空（= 直连 ListenBrainz）。

```bash
git clone https://github.com/Yudaotor/nowplaying.git web-page
# 把 web-page/demo/index.html 拷进你自己的仓库,托管到任意静态站
# (GitHub Pages / Cloudflare Pages 都行),访问:
# https://<你的域名>/index.html?user=<你的 ListenBrainz 用户名>
```

然后把它指向第一步的 state-worker——两种方式任选：

- **不改源码**：访问时带 `?relay=<你的地址>`（比如分享链接时用 `?user=xxx&relay=https://your-worker.workers.dev`）。
- **设成默认**：改模板 `<script>` 块开头附近的 `RELAY` 常量，访客就不需要带参数了。

（那个仓库根目录的 `index.html` 是作者自己在线上跑的同一份页面——功能完全一致，只是内置了作者自己的中继地址。任何一份拷贝想完全跳过中继，带 `?relay=off`。）

### 第三步：把 Lyrimuse 和 state-worker 串起来

打开 Lyrimuse 菜单栏「设置…」→ 侧边栏「附加功能」→「网页推送」，填两项：

- **同步服务地址**：第一步拿到的 state-worker 地址
- **访问令牌**：跟第一步 `PUSH_TOKEN` 完全相同的字符串

填完就结束了——不需要再找一个开关打开。采集器下次重启（改完设置会自动触发）就会开始往这个地址推状态。

### 第四步：验证

1. 直接访问你自己的网页地址，看能不能看到"正在播放"或"上次播放"（不是卡在"连接中…"）。
2. 换一首歌，等几秒钟刷新，确认标题/封面跟着变了——说明 采集器 → state-worker → 网页 这条链路真的通了，不是网页在读 ListenBrainz 兜底（兜底模式下换歌到网页看到之间通常要慢不少，且没有留言墙等模块）。
3. 试着在网页上留一句言/点个赞，刷新页面看看还在不在。

### 可选：GitHub README 动态徽章（badge-worker）

前提是上面的 state-worker 已经跑起来了。`badge-worker/` 是另一个可选的 Cloudflare Worker，读 state-worker 的数据渲染成一张实时更新的 SVG 徽章：

1. 把 `badge-worker/wrangler.toml` 的 `[vars]` 里 `RELAY_URL` 改成你自己 state-worker 的地址。
2. `cd badge-worker && npm run deploy`。
3. 嵌进任意 README：`![now playing](https://<你的badge-worker地址>.workers.dev/badge)`。

## 有什么做不到 / 目前的限制

- **五个可选展示模块（历史/留言墙/表情反应/访客计数/Top10 歌手）没有逐项开关**：只要「网页推送」的地址+令牌配好，五个一起出现；不填就五个都不出现。真只想要其中几个，目前唯一的办法是改你自己那份页面（`index.html`），把不想要的模块的 DOM/初始化函数删掉。
- **「历史播放」这个模块只有 ListenBrainz 才能提供**：state-worker 自己不存收听历史（它的 KV 里放的是当前状态、访客互动记录、每日一份的 Top10 歌手快照和一小份模块可见性配置——没有逐条收听日志），所以哪怕你完全不想要 ListenBrainz、只用 state-worker 推"正在播放"，网页上的历史列表依然是回退直连 ListenBrainz 读出来的。
- **在中国访问 ListenBrainz 直连可能不稳定**，这也是 state-worker 这个"中国加速中继"存在的原因——不部署它也能用，只是网页在中国访问时可能偶尔转圈。
- **网页（Yudaotor/nowplaying 的 `index.html`）和 `state-worker/src/index.js` 各自维护着一份几乎相同的 `fromLB`（解析 ListenBrainz 原始数据）函数**，两处代码里都有互相指向对方的注释——这是因为它们分别部署在 GitHub Pages 和 Cloudflare Workers 两个不相关平台，没有共用构建步骤做不到真正共享同一份源码。如果你要改这段解析逻辑，两处都要照着改一遍，不然会悄悄不一致。

## 常见问题

**网页一直显示"连接中…稍后自动重试"** — 检查 URL 有没有带对 `?user=` 参数（ListenBrainz 用户名），以及浏览器能不能访问到你的 state-worker 地址（或者 ListenBrainz 本身是否可达）。

**换了歌，网页半天才更新，而且看不到留言墙/Top10 歌手这些模块** — 说明网页目前走的是 ListenBrainz 兜底路径，没有真正连上你的 state-worker。检查：Lyrimuse 设置里「网页推送」地址+令牌是否已经填好并保存（保存后采集器会自动重启生效）；网页访问的地址有没有被 `?relay=off` 意外关掉；直接用浏览器访问 `https://<你的state-worker地址>/now` 看返回的是不是正常 JSON（不是 401/500）。

**`/push` 一直 401** — `state_relay_token`（Lyrimuse 设置里填的）跟 state-worker 的 `PUSH_TOKEN`（`wrangler secret put` 设置的那个）必须是完全相同的字符串，改了一头忘了改另一头是最常见的原因。

**封面/歌手头像加载不出来** — 封面走网易云图床，Top10 歌手头像优先走 QQ 音乐图床、查不到才退到 Deezer。取决于你从哪里访问，这几个源都可能偶尔加载失败；网页对两种情况都有兜底（封面退到纯色背景，头像退到圆形首字母占位），不影响其它内容正常显示。

**只想要网页展示当前播放，不想要留言墙这些互动功能** — 目前做不到"部分开启"，见上面「有什么做不到」——需要改你自己那份 `index.html` 删掉不想要的模块。
