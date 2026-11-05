# 关务HOT (CustomsHOT)

> 面向国际物流、报关实务与进出口贸易合规行业的自动化热点与情报站。

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-176b75?style=flat-square" alt="MIT License"></a>
  <img src="https://img.shields.io/badge/Node.js-24-176b75?style=flat-square&logo=nodedotjs&logoColor=white" alt="Node.js 24">
  <img src="https://img.shields.io/badge/PostgreSQL-17-176b75?style=flat-square&logo=postgresql&logoColor=white" alt="PostgreSQL 17">
  <img src="https://img.shields.io/badge/Docker-Compose-176b75?style=flat-square&logo=docker&logoColor=white" alt="Docker Compose">
</p>

<p align="center">
  <b>自动盯住官方公告与行业信源，大模型筛选、打分、提炼摘要与事件归组，每天 08:00 出一份关务日报。</b>
</p>

<p align="center">
  <b>简体中文</b> · <a href="README.en.md">English</a>
</p>

<p align="center">
  <a href="#快速开始">快速开始</a> ·
  <a href="#核心特性与关务纪律">核心特性</a> ·
  <a href="#系统架构">系统架构</a> ·
  <a href="#信源与配置">信源与配置</a> ·
  <a href="#项目文档">项目文档</a> ·
  <a href="#开源致谢与许可">开源致谢</a>
</p>

---

## 项目简介

**关务HOT** 是一个专为进出口关务、国际物流、货代申报与贸易合规团队打造的情报聚合与自动化成刊系统。

在进出口行业中，政策法规繁杂多变、发布渠道分散（海关总署、商务部、单一窗口、财政部税委会等），且政策具有极强的时效敏感性（实施生效日期直接影响申报通关与关税成本）。同时，行业网络资讯中充斥大量货代揽货广告与培训软文等噪音。

关务HOT 自动跟踪各级官方源与权威媒体，运用大语言模型流水线完成**相关性预筛、双盲注意力评分、核心事实提取（文号、生效日期、税率变动）、事件聚类聚类**，每日定时生成《关务日报》，并通过网页端、飞书机器人、RSS 及 Agent MCP 协议对外分发。

---

## 核心特性与关务纪律

### 1. 权威官方信源（P0 就位）
- **海关总署**：法规与署令公告（`web-gacc-fagui`）；
- **国务院关税税则委员会**：税率调整、关税政策发布（`taxcommittee-gg`）；
- **国际贸易单一窗口**：通知公告与政策法规（`sw-notice` / `sw-policy`）；
- **商务部**：贸易救济调查公告（反倾销/反补贴/贸易摩擦）与新闻发布（`mofcom-trb` / `mofcom-xwfb`）。

### 2. 严格的关务内容纪律
- **发布日与生效日分离**：关务摘要强制写明文号、发布日期与**实施/生效日期**（两者不同是常态，防止误导实际申报动作）；
- **数据保真**：税率、税号（HS 编码）、品类范围等关键参数严格忠实原文，不擅自估算；
- **压制行业噪音**：针对货代低价揽货广告、会议招商、无事实增量的旧政科普等设置负向压制规则；
- **版权合规**：站内默认仅展示结构化摘要与原文链接（`site_fulltext: false`），不全文抓取侵权。

### 3. 多通道触达与输出
- **自动化日报**：每天 08:00 自动汇编出刊，周一 10:00 出周报，每月 1 日出月报；
- **飞书集成**：支持每日精选推送到飞书群机器人（Webhook），支持系统健康与抓取异常告警；
- **开放协议支持**：提供标准 RSS / Atom 订阅、OpenAPI v1 接口、Markdown 输出及大模型 MCP（Model Context Protocol）接入。

---

## 系统架构

系统基于三进程体系与 PostgreSQL 17 搭建：

| 进程 | 目录 | 职责与技术栈 |
|---|---|---|
| **api** | `apps/api/` | **Fastify 后端**。自用接口、公开 API v1、RSS、MCP 工具、管理后台接口、图片代理与 OG 分享图 |
| **worker** | `apps/worker/` | **pg-boss 任务队列**。信源抓取调度、大模型调用流水线、事件聚类归组、日报周报汇编、预算控制与告警 |
| **web** | `apps/web/` | **React Router 8 SSR + Tailwind 4**。前端服务端渲染，仅通过 HTTP 读 API，不直接接触数据库与模型密钥 |

### 核心工程机制
- **付费回执与预算熔断 (`packages/backend/src/providers/receipts.ts`)**：调用大模型前预先写入幂等回执，分/时/天三级预算熔断，并发锁保护，防止异常重试消耗巨额 Token；
- **单向只读发布层 (`packages/backend/src/publication/`)**：所有公开出口统一经过发布层投影，确保撤回、过滤和席位控制在所有端点一致生效；
- **行业解耦设计**：系统框架（`apps/`、`packages/`）与行业知识（`industry/`）、站点配置（`site/`）分离，便于合并底层框架更新。

---

## 快速开始

### 环境要求
- **Node.js** >= 24
- **PostgreSQL** >= 17
- **Docker & Docker Compose**（推荐容器化运行）
- OpenAI 兼容接口的大模型 API Key（如 GPT-4o-mini、DeepSeek、智谱 GLM、千问等）

### 方式一：Docker Compose 部署（推荐）

1. **准备配置文件**：
   ```bash
   cp .env.example .env
   # 编辑 .env 填入必要的数据库密码、SITE_URL 与 LLM 配置
   ```
2. **构建并启动容器**：
   ```bash
   docker compose build --build-arg NPM_REGISTRY=https://registry.npmmirror.com
   docker compose up -d
   ```
3. 打开浏览器访问 `http://localhost:3000`。管理后台位于 `/admin`（密码见 `.env` 中的 `ADMIN_PASSWORD`）。

### 方式二：本机/单机直接运行

```bash
# 1. 安装依赖
npm install

# 2. 数据库迁移
npm run db:migrate

# 3. 导入初始信源与配置
node --env-file=.env scripts/seed.ts

# 4. 分别启动三个进程
node --env-file=.env apps/api/src/main.ts       # API 服务 (端口 3001)
node --env-file=.env apps/worker/src/main.ts    # Worker 后台调度
cd apps/web && node --env-file=../../.env server.ts  # Web 前端 (端口 3000)
```

---

## 目录结构

```
customs-hot/
├── apps/
│   ├── api/          # Fastify API 服务、公开出口与后台
│   ├── web/          # React Router 8 前端 SSR
│   └── worker/       # pg-boss 调度、抓取、大模型流水线与出刊
├── packages/
│   ├── backend/      # 核心后端逻辑（信源、判断写作、归组、发布层）
│   └── contracts/    # 前后端与各模块共享的类型定义
├── site/             # 站点身份、文案、模型分工与页面
│   ├── site.ts       # 站名、口号、出刊时间（关务HOT）
│   └── models.ts     # 各环节模型配置
├── industry/         # 关务行业知识包
│   ├── sources.json  # 初始导入的官方信源配置
│   ├── taxonomy.ts   # 关务分类、标签词表与监管机构实体
│   ├── selection.ts  # 入选评分门槛与基线
│   └── prompts/      # 关务专属提示词（预筛、评分、摘要、归组等）
├── project-docs/     # 关务项目方案、清单与内部文档
│   ├── 00-上游项目AIHOT分析.md
│   ├── 01-总体改造方案.md
│   ├── 02-信源建设清单.md
│   ├── 03-内部试用部署方案.md
│   ├── 04-评测与校准手册.md
│   └── 05-风险与合规清单.md
├── docs/             # 底层框架技术参考文档
└── PROJECT.md        # 项目定位、决策与运行记录
```

---

## 项目文档

关务HOT 的具体方案与实施手册位于 [`project-docs/`](project-docs/)：
- **[01-总体改造方案](project-docs/01-总体改造方案.md)**：关务 9 大分类体系、8 种内容类型与评分权重规范；
- **[02-信源建设清单](project-docs/02-信源建设清单.md)**：P0/P1 官方与行业信源抓取配置、防爬探测记录与接口分析；
- **[03-内部试用部署方案](project-docs/03-内部试用部署方案.md)**：内部服务器部署、飞书群推送配置与监控运维；
- **[04-评测与校准手册](project-docs/04-评测与校准手册.md)**：关务 Gold 样本集标注、模型评分评测及门槛调优流程；
- **[05-风险与合规清单](project-docs/05-风险与合规清单.md)**：内容真实性、版权保护及访问权限安全规范。

---

## 开源致谢与许可

- 本项目底层架构基于开源项目 [KKKKhazix/AIHOT](https://github.com/KKKKhazix/AIHOT) 二次开发构建。
- 业务代码遵循 [MIT 许可证](LICENSE)。
- 本项目严禁冒用上游原项目名称及商标。
