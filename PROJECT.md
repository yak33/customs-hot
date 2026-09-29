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

- **内部试用优先**：暂不做公开站，暂不需要 ICP 备案；受众是集团内关务 / 物流相关同事，口味统一，校准容易。
- **先国内信源**：第一阶段只做国内信源（海关总署、商务部、税务总局、单一窗口、行业媒体等），不接海外源（从国内服务器抓海外源需要代理，后面再补）。
- **公众号暂缓**：关务类公众号（如 12360 海关发布）是重要一手源，但需要「极致了」付费 key，先不接，用 RSS + 网页列表跑起来。
- **模型**：用 OpenAI 兼容接口（DeepSeek / 千问 / 智谱均可），所有步骤先共用一个模型，跑顺后再按 `.env.example` 里的分工细化。
- **部署目标**：腾讯云 Linux + Docker Compose，2 核 4G 起步。

## 待决策事项（需要使用者本人回答，Agent 不要替他定）

1. **站名**：`industry/site.ts` 的 `name`、`subject`（行业词，如「关务」）、首页文案。⚠️ 不得使用 AIHOT 的名字和 Logo（许可要求）。
2. **信源名单初版**：要盯哪些网站 / 栏目，逐个列出 URL。
3. **精选口味**：什么消息对我们是重要的（如税率调整、监管新政、AEO、出口退税），什么是噪声（如货代广告、课程软文）——这决定 `industry/prompts/selection-score.md` 怎么改。
4. **分类怎么分**：`industry/taxonomy.ts` 的类别和标签。
5. **条款与隐私说明**：`industry/pages/` 目前是模板，上线前需本人确认（内部站也应改写）。

## 改造与部署入口

- 改造流程：严格按 `docs/customize.md` 九步顺序做；改动几乎都集中在 `industry/`，不动 `apps/` 和 `packages/`。
- 校准方法：`docs/selection.md`。需要标注 100–200 条「该选 / 不该选」的关务资料样本，跑 `scripts/eval-selection.ts` 后再定门槛，**不要凭感觉改 `industry/selection.ts` 的数字**。
- 部署：`docs/deploy.md`。生成配置：`node scripts/init-env.ts --llm-key <模型 Key>`，然后 `docker compose up -d --build`。
- 国内构建加速：`docker compose build --build-arg NPM_REGISTRY=https://registry.npmmirror.com`。

## 与上游的关系

- 本目录是 AIHOT 的快照副本，不带上游 git 历史。后续如需同步上游更新，人工对照 `git log` 挑选合并。
- `LICENSE`（MIT）和 `NOTICE` 必须保留；AIHOT 的名字和 Logo 不在许可范围内。
- AI 专属模块（模型榜、Codex 监控）对关务无意义：改造时在 `industry/features.ts` 把两项设为 `false`。
