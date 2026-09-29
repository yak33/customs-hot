# customs-hot 项目说明

> 本项目基于开源框架 [KKKKhazix/AIHOT](https://github.com/KKKKhazix/AIHOT)（MIT 许可）搭建，
> 快照取自 2026-09-29。框架本身的使用方法见 `README.md`、`AGENTS.md` 和 `docs/`，本文件只记录**我们这个项目**的定位与决策。

## 定位

面向**集团内部**的国际物流 / 进出口报关 / 关务行业热点站：
从海关、商务、税务等官方信源和行业媒体自动采集，用大模型筛选评分、写中文摘要、聚成事件、按热度排行，每天早上出一份关务日报。

## 阶段路线：先小后大

| 阶段 | 目标 | 退出标准 |
|---|---|---|
| 1. 跑通框架 | 默认配置在本机 / 服务器跑起来，验证流程 | 首页有内容、后台能看信源状态 |
| 2. 改成关务 | 改造 `industry/`（站名、分类、信源、提示词、门槛） | 关务内容开始进精选 |
| 3. 内部试用 | 部署到腾讯云，给集团关务 / 物流同事用 | 收集 2 周以上真实使用反馈 |
| 4. 再议公开 | 根据试用效果决定是否做公开行业站 | 届时再评估备案、品牌、合规 |

**当前处于阶段 1–2 之间。**

## 已确认的决策

- **站名**：关务HOT（`industry/site.ts` 已改：subject=关务、mcpPrefix=customshot、crawlerName=CustomsHotBot）。
- **内部试用优先**：暂不做公开站，暂不需要 ICP 备案；受众是集团内关务 / 物流相关同事，口味统一，校准容易。
- **先国内信源**：第一阶段只做国内信源（海关总署、商务部、税务总局、单一窗口、行业媒体等），不接海外源（从国内服务器抓海外源需要代理，后面再补）。
- **公众号暂缓**：关务类公众号（如 12360 海关发布）是重要一手源，但需要「极致了」付费 key，先不接，用 RSS + 网页列表跑起来。
- **模型**：用 OpenAI 兼容接口（DeepSeek / 千问 / 智谱均可），所有步骤先共用一个模型，跑顺后再按 `.env.example` 里的分工细化。
- **部署目标**：腾讯云 Linux + Docker Compose，2 核 4G 起步。

## 首批信源：单一窗口（已配置）

`industry/sources.json` 里现在是两条 `json_list` 信源，取代了上游的 18 个 AI 示范源。

**接口探测结论**（2026-09-29 实测，`https://www.singlewindow.cn/#/notice` 和 `#/policy` 是 SPA，列表数据来自这个接口）：

```
POST https://www.singlewindow.cn/access/ui/SW-SITE-FRONT/Article001
Body: {"Head":{},"Data":"{\"pageNo\":1,\"rowsPerPage\":20,\"tagids\":null,\"keyword\":null,\"catalogid\":\"tzgg\"}"}
                                            └─ Data 是「字符串化的 JSON」，不是对象
catalogid：tzgg=通知公告（详情页 breadNum=bc12）｜zcfg=政策法规（breadNum=bc14）
响应字段：data.data[] → articleid / title / summary / releasetime(毫秒) / source / sourcesrc
```

- 浏览器请求带的 `?cw2qEsfh=` 参数是反爬签名，**服务端不校验**，裸调可用（UA + Referer 足够）。
- 详情正文走 `Article002`（`{"Head":{},"Data":"{\"articleid\":\"xxx\"}"}`，返回 `bodytext` HTML）；详情页本身是 SPA，抓不到正文。
- **当前折中**：`summaryIsBody: true`，用列表接口的一句话摘要当正文，够预筛和评分用，不依赖付费渲染。要全文时再写个小脚本调 `Article002` 走 `/api/ingest/items` 外部推送（见 `docs/sources.md`）。
- 风险提示：这是政府站的内部接口，改版或加校验就会失效，后台「信源」页会显示抓取失败原因。

## 待决策事项（需要使用者本人回答，Agent 不要替他定）

1. **扩充信源**：目前只有单一窗口两条。海关总署公告栏、商务部、税务总局（出口退税）、行业媒体等待补。
2. **精选口味**：什么消息对我们是重要的（如税率调整、监管新政、AEO、出口退税），什么是噪声（如货代广告、课程软文）——这决定 `industry/prompts/selection-score.md` 怎么改。
3. **分类怎么分**：`industry/taxonomy.ts` 还是 AI 行业的类别和标签，要按关务重新设计。
4. **条款与隐私说明**：`industry/pages/` 目前是模板，上线前需本人确认（内部站也应改写）。

## 改造与部署入口

- 改造流程：严格按 `docs/customize.md` 九步顺序做；改动几乎都集中在 `industry/`，不动 `apps/` 和 `packages/`。
- 校准方法：`docs/selection.md`。需要标注 100–200 条「该选 / 不该选」的关务资料样本，跑 `scripts/eval-selection.ts` 后再定门槛，**不要凭感觉改 `industry/selection.ts` 的数字**。
- 部署：`docs/deploy.md`。生成配置：`node scripts/init-env.ts --llm-key <模型 Key>`，然后 `docker compose up -d --build`。
- 国内构建加速：`docker compose build --build-arg NPM_REGISTRY=https://registry.npmmirror.com`。

## 仓库与上游

- **本项目仓库**：`https://github.com/yak33/customs-hot`（私有），remote 名 `origin`，分支 `main`。
- 本目录是 AIHOT 的快照副本，`main` 的第一个提交是上游快照基线（不带上游 git 历史）。后续如需同步上游更新，从 https://github.com/KKKKhazix/AIHOT 对照挑选合并。
- `LICENSE`（MIT）和 `NOTICE` 必须保留；AIHOT 的名字和 Logo 不在许可范围内。
- AI 专属模块（模型榜、Codex 监控）对关务无意义：改造时在 `industry/features.ts` 把两项设为 `false`。
