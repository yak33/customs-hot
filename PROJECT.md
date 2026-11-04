# customs-hot 项目说明

> 本项目基于开源框架 [KKKKhazix/AIHOT](https://github.com/KKKKhazix/AIHOT)（MIT 许可）搭建。
> **2026-11-04 已整体同步上游 4.0**（`44578fa` → `cd424b0`，206 个提交），目录结构随上游重构：站点身份在 `site/`（`site.ts`、`models.ts`、`brand/`、`pages/`、`public/`、`modules/`），行业知识在 `industry/`（`taxonomy.ts`、`prompts/`、`sources.json`、`selection.ts`、`wording.ts`），只属于自己站的模块放仓库根 `modules/`（`site/modules/` 三份清单启用，默认空）。框架用法见 `README.md`、`AGENTS.md` 和 `docs/`，本文件只记录**我们这个项目**的定位与决策。升级前状态在 tag `pre-upgrade-4.0`。

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

**当前处于阶段 3 起步**（2026-11-04：已同步上游 4.0；内部试用版部署腾讯云，安全阀待 key）。

## 腾讯云部署状态（2026-09-30 首版）

- 服务器：`root@82.157.149.224`（SSH 密钥 `~/.ssh/jgagent_deploy`，2 核 2G，已加 2G 临时 swap `/swapfile2` 保构建）。同机还跑着 portainer 与 jgagent 系列，互不影响。
- 代码位置：`/opt/customs-hot`，来源是 `git bundle`（服务器无 GitHub 凭据；更新方式：本机 `git bundle create /tmp/customs-hot.bundle main` → scp → 服务器 `git -C /opt/customs-hot pull /tmp/customs-hot.bundle main` → `docker compose up -d --build`）。
- 容器：db / api / worker / web 四个常驻 + setup 一次性（35 迁移、38 主题、6 信源导入）；宿主机 `curl localhost:3000` 200，`/api/health` db:ok，MCP 端点正常。
- `.env`（服务器侧生成）：`SITE_URL=http://82.157.149.224:3000`，`COLLECT_ENABLED=false`、`MODEL_CALLS_ENABLED=false`（等 `LLM_API_KEY`，智谱 glm-5.3-flash 预配置，填好后两个阀改 true 再 `docker compose up -d`）。
- **待办**：腾讯云安全组放行 3000 端口（限公司出口 IP / VPN 网段，内部试用不备案）；飞书群机器人 webhook（03 文档 §3）。

## 模型链路结论（2026-11-04 更新）

- **在用**：中转 `https://runanytime.hxi.me/v1` + `z-ai/glm-5.2`（非流式 JSON、输出正常）。`LLM_BASE_URL` 只能填到 `/v1`，框架自己拼 `/chat/completions`。
- **备用（2026-11-04 实测可用）**：中转 `https://muyuan.do/v1`（新 key），共 3 个模型：grok-4.7 / grok-4.6 / grok-4.7-build-fast。非流式 JSON 正常；**思考型但 reasoning tokens 不占 max_tokens 预算**（预筛 512 限额下输出完整，与 glm-5.3-flash 吃光限额的坑不同）；`reasoning_effort=low` 透传有效（思考 ~739 tokens、单条 9 秒）；5 并发无限流。接入方式：`.env` 填 `LLM_BASE_URL=https://muyuan.do/v1`、`LLM_MODEL=grok-4.7`、`LLM_EXTRA_JSON={"reasoning_effort":"low"}`。注意消耗速度快慢待观察（响应带 cost_in_usd_ticks，单位不明）。
- **弃用教训**：`z-ai/glm-5.3-flash` 是思考型，思考会把预筛 512 / 评分 1024 的 maxTokens 吃光导致 content 为空；muyuan.do 旧 key 上游 token 失效（新 key 已恢复）。`api-bj.ark2.cn` 截至 2026-11-04 仍然 key 分组下无任何可用渠道（8 个常见模型名全 503 model_not_found）。
- **限流**：runanytime 中转 1 分钟最多 15 次请求，已在 `budgets` 表把 `llm` 服务设为 `per_minute=13`（框架主动节流排队）。muyuan.do 无明显限流。
- **上游 4.0 相关**：模型名不存在时后台会明示（不再静默失败）；额度用尽按模型计 15 分钟冷却（不再失败重试烧钱）；换具名模型可在 `site/models.ts` 配 PRESETS。
- **下一步内容侧**：`industry/taxonomy.ts` 还是 AI 分类（analyses.category 显示 industry），等业务对接人确认 01 文档 §2 草案后落地；其余 16 个提示词按 01 文档 §4 清单逐个行业化。

## 本机运行状态（2026-09-30）

- 环境：Windows + Node 24 直跑 TS（无构建步骤）+ PostgreSQL 17.11 本机服务（`postgresql-x64-17`），库 `customs_dev`（运行）/ `customs_test`（测试）。
- 启动方式（`docs/deploy.md` 的"不用 Docker"路径）：
  ```bash
  node --env-file=.env apps/api/src/main.ts          # 接口 3001
  node --env-file=.env apps/worker/src/main.ts       # 后台任务
  cd apps/web && NODE_ENV=production node --env-file=../../.env server.ts   # 网页 3000
  ```
- `scripts/smoke.ts` 30/30 通过；后台"信源"页 6 条信源正常展示。
- `.env`：`init-env` 生成后，开发期保持 `COLLECT_ENABLED=false`、`MODEL_CALLS_ENABLED=false`（安全阀，仓库规矩），补了 `DATABASE_URL` / `API_BASE_URL` / `DEV_AUTH_ROLE=admin`（免登录进后台，仅本机开发）。`LLM_API_KEY` 待填。
- 上游代码修复一处：`apps/web/server.ts` 动态 `import()` 在 Windows 盘符路径下报 `ERR_UNSUPPORTED_ESM_URL_SCHEME`（已改用 `pathToFileURL`，Linux 行为不变，可回传上游）。

### 本机跑测试的已知结论（2026-11-04，4.0 更新）

- **4.0 测试基建自动隔离**：每个测试文件跑在自己的数据库副本上（`tests/databases.ts` 自动建模板库、迁移、清理），**不再需要每次手动重建空库**，直接 `DATABASE_URL=...customs_test npm test` 即可。用例规模：166 个后端测试文件 + 42 个 standalone（`npm run test:standalone`）+ web 测试（`npm run test:web`，浏览器引擎测试，串行极慢）。
- **本机环境差异清单（非升级缺陷，上游 CI Linux 全绿）**：① SIGTERM 生命周期 6 个用例超时（Windows 不投递信号，`backup.test.ts` 更是挂死，跑全量时排除）；② 本机 PG 是中文 locale，错误消息断言失败 2 个（"由于锁超时" vs `/lock timeout/`，服务器容器 PG 为英文不受影响）；③ `outbound-protocol.standalone` 需要 openssl CLI（本机未装）；④ `outbound-headers.standalone` 2 个用例疑与 Node 版本有关；⑤ 6 并发下个别用例偶发超时（error-logging、report-recovery 等，单独跑全过）。
- **本机 Node 24.19 低于上游 4.0 要求的 >=24.22.1**（`npm install` 有 EBADENGINE 警告），建议升级 Node；服务器 Docker 构建用上游指定版本不受影响。
- 2026-11-04 升级验证：typecheck 全绿；migrate 71/71（旧 35 → 新增 36，对服务器旧库是纯增量）；后端 164 文件（排除 backup/signals）仅上述环境差异失败，其余全过；`apps/web` 构建通过（web 测试未完整跑，浏览器引擎测试极慢，WebKit 用例本机疑似失败）。

## 已确认的决策

- **站名**：关务HOT（4.0 后在 `site/site.ts`：subject=关务、mcpPrefix=customshot、crawlerName=CustomsHotBot/1.0）。
- **内部试用优先**：暂不做公开站，暂不需要 ICP 备案；受众是集团内关务 / 物流相关同事，口味统一，校准容易。
- **先国内信源**：第一阶段只做国内信源（海关总署、商务部、税务总局、单一窗口、行业媒体等），不接海外源（从国内服务器抓海外源需要代理，后面再补）。
- **公众号暂缓**：关务类公众号（如 12360 海关发布）是重要一手源，但需要「极致了」付费 key，先不接，用 RSS + 网页列表跑起来。
- **模型**：用 OpenAI 兼容接口（DeepSeek / 千问 / 智谱均可），所有步骤先共用一个模型，跑顺后再按 `.env.example` 里的分工细化。
- **部署目标**：腾讯云 Linux + Docker Compose，2 核 4G 起步。

## 首批信源：官方三源已立，P0 基本就位（2026-09-29）

`industry/sources.json` 里现在是 6 条信源，取代了上游的 18 个 AI 示范源：

| id | 信源 | kind | 验证方式 |
|---|---|---|---|
| `sw-notice` / `sw-policy` | 单一窗口 · 通知公告 / 政策法规 | `json_list` | 接口实测 |
| `web-gacc-fagui` | 海关总署 · 海关法规与公告 | `web_list` | 框架解析器 `fromHtml` 离线实测 10 条全对 |
| `taxcommittee-gg` | 国务院关税税则委员会 · 公告（财政部关税司） | `web_list` | 框架解析器离线实测 10 条全对 |
| `mofcom-trb` | 商务部 · 贸易救济与贸易摩擦公告（子站） | `web_list` | 框架解析器离线实测 37 条全对 |
| `mofcom-xwfb` | 商务部 · 新闻发布 | `web_list` | 框架解析器离线实测 30 条全对 |

**单一窗口接口探测结论**（2026-09-29 实测，`https://www.singlewindow.cn/#/notice` 和 `#/policy` 是 SPA，列表数据来自这个接口）：

```
POST https://www.singlewindow.cn/access/ui/SW-SITE-FRONT/Article001
Body: {"Head":{},"Data":"{\"pageNo\":1,\"rowsPerPage\":20,\"tagids\":null,\"keyword\":null,\"catalogid\":\"tzgg\"}"}
                                            └─ Data 是「字符串化的 JSON」，不是对象
catalogid：tzgg=通知公告（详情页 breadNum=bc12）｜zcfg=政策法规（breadNum=bc14）
响应字段：data.data[] → articleid / title / summary / releasetime(毫秒) / source / sourcesrc
```

- 浏览器请求带的 `?cw2qEsfh=` 参数是反爬签名，**服务端不校验**，裸调可用（UA + Referer 足够）。
- 详情正文走 `Article002`（`{"Head":{},"Data":"{\"articleid\":\"xxx\"}"}`，返回 `bodytext` HTML）；详情页本身是 SPA，抓不到正文；列表接口的 `bodytext` 字段实测为空。
- **当前折中**：`summaryIsBody: true`，用列表接口的一句话摘要当正文，够预筛和评分用，不依赖付费渲染。要全文时再写个小脚本调 `Article002` 走 `/api/ingest/items` 外部推送（见 `docs/sources.md`）。
- 风险提示：这是政府站的内部接口，改版或加校验就会失效，后台「信源」页会显示抓取失败原因。

**商务部**（同日探测）：主站是 JPAAS 架构，"政策发布"栏目列表走接口且返回"JSON 包 HTML"，框架不支持，未接入；实际接入的是两个静态入口——贸易救济子站首页（反倾销/反补贴/贸易摩擦应对三区块，`li` + 路径白名单过滤，37 条）和主站新闻发布聚合页（7 个子栏目，`ul.txtList_01 li`，30 条）。**注意**：贸易救济子站对 Windows curl 会 SSL 重协商失败，但 Node fetch 正常，框架抓取不受影响。

**国家税务总局（暂缓）**（同日探测）："最新文件"列表静态但条目只有文号没有日期；详情页正文 JS 动态加载，detail 补日期走不通；政策法规库列表也是骨架；统一搜索接口 `search5/search/s` 参数未摸通。后续摸通接口（或走 `external` 外部推送）再接入。海关总署部分栏目（政策解读 `302272` 等）仍被 WAF 拦截（412）。**新增信源必须先实测**（详见 `project-docs/02-信源建设清单.md` 探测记录）。

## 项目文档（project-docs/）

> 本项目的方案、清单与手册独立于上游 `docs/`（框架文档）存放，避免同步上游时混淆。

| 文档 | 内容 |
|---|---|
| [00-上游项目AIHOT分析](project-docs/00-上游项目AIHOT分析.md) | 框架架构与工程亮点速览 |
| [01-总体改造方案](project-docs/01-总体改造方案.md) | `industry/` 逐文件改造清单；**分类体系（9 类）、内容类型（8 类）、五维评分权重、机构实体词典草案 v0.1** |
| [02-信源建设清单](project-docs/02-信源建设清单.md) | P0/P1/P2 信源清单、接入方法、探测记录与待核实项 |
| [03-内部试用部署方案](project-docs/03-内部试用部署方案.md) | 腾讯云部署、飞书推送、安全阀、预算、试用成功指标与复盘决策点 |
| [04-评测与校准手册](project-docs/04-评测与校准手册.md) | 关务版 gold 样本标注规范、评测命令、门槛校准循环 |
| [05-风险与合规清单](project-docs/05-风险与合规清单.md) | 内容准确性纪律、版权、访问控制、第三方依赖、公开阶段 checklist |

## 待决策事项（需要使用者本人回答，Agent 不要替他定）

1. **扩充信源**：官方侧已有单一窗口（2 条）+ 海关总署（1 条）+ 税委会（1 条）+ 商务部（2 条）。税务总局暂缓（接口未摸通，见 02 文档）；Federal Register 需代理；地方海关**待业务侧指定关区**。
2. **精选口味**：什么消息对我们是重要的（如税率调整、监管新政、AEO、出口退税），什么是噪声（如货代广告、课程软文）——这决定 `industry/prompts/selection-score.md` 怎么改。
3. **分类怎么分**：`industry/taxonomy.ts` 还是 AI 行业的类别和标签。**01 文档里已有关务版草案 v0.1（9 个分类、8 个内容类型、标签词表、机构词典），请业务对接人过一遍确认或修改后落地。**
4. **条款与隐私说明**：`site/pages/` 目前是模板，上线前需本人确认（内部站也应改写）。
5. **AI 专属模块（模型榜、Codex 监控）**：上游 4.0 已把两者移出框架（对关务本来就无用），模块机制改为 `site/modules/` 三份清单（默认空），原 `industry/features.ts` 开关随升级废弃，无需再改。

## 改造与部署入口

- 改造流程：严格按 `docs/customize.md` 九步顺序做；改动几乎都集中在 `industry/`，不动 `apps/` 和 `packages/`。
- 校准方法：`docs/selection.md`。需要标注 100–200 条「该选 / 不该选」的关务资料样本，跑 `scripts/eval-selection.ts` 后再定门槛，**不要凭感觉改 `industry/selection.ts` 的数字**。
- 部署：`docs/deploy.md`。生成配置：`node scripts/init-env.ts --llm-key <模型 Key>`，然后 `docker compose up -d --build`。
- 国内构建加速：`docker compose build --build-arg NPM_REGISTRY=https://registry.npmmirror.com`。

## 仓库与上游

- **本项目仓库**：`https://github.com/yak33/customs-hot`（私有），remote 名 `origin`，分支 `main`。
- 本目录是 AIHOT 的快照副本，`main` 的第一个提交是上游快照基线（不带上游 git 历史）。**已添加 `upstream` remote 指向 https://github.com/KKKKhazix/AIHOT**，同步上游用 `git fetch upstream && git log HEAD..upstream/main --oneline` 查看差异，再按需 cherry-pick / 合并。
- `LICENSE`（MIT）和 `NOTICE` 必须保留；AIHOT 的名字和 Logo 不在许可范围内。
- AI 专属模块（模型榜、Codex 监控）对关务无意义：上游 4.0 已将其移出框架，`site/modules/` 保持空清单即可（原 `industry/features.ts` 关闭记录随升级废弃）。
