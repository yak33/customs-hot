# CustomsHOT

> Automated news, regulatory updates, and intelligence briefings for customs clearance, tariffs, and international logistics.

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-176b75?style=flat-square" alt="MIT License"></a>
  <img src="https://img.shields.io/badge/Node.js-24-176b75?style=flat-square&logo=nodedotjs&logoColor=white" alt="Node.js 24">
  <img src="https://img.shields.io/badge/PostgreSQL-17-176b75?style=flat-square&logo=postgresql&logoColor=white" alt="PostgreSQL 17">
  <img src="https://img.shields.io/badge/Docker-Compose-176b75?style=flat-square&logo=docker&logoColor=white" alt="Docker Compose">
</p>

<p align="center">
  <a href="README.md">简体中文</a> · <b>English</b>
</p>

---

## Overview

**CustomsHOT** is an automated intelligence aggregation and briefing system built specifically for trade compliance, customs brokerage, and cross-border logistics teams.

Key highlights:
- **Authority Sources (P0 ready)**: Monitored feeds from China Customs (GACC), Ministry of Commerce (MOFCOM), Single Window, and the State Council Tariff Commission.
- **LLM Pipeline**: Automated pre-filtering, double-blind attention scoring, structured key-fact extraction (announcement ID, publication date vs. effective date, tariff adjustments), and story clustering.
- **Strict Content Discipline**: Separates publication date from enforcement date, strictly preserves tariff and HS code numbers without hallucination, suppresses freight forwarder advertising and noisy promotions.
- **Daily Briefings & Omnichannel Export**: Generates daily briefings every morning at 08:00 (UTC+8), with Feishu bot webhook notifications, RSS feeds, OpenAPI v1, and MCP (Model Context Protocol) integration for AI agents.

---

## Architecture

- **`apps/api`** (Fastify): REST API, RSS, MCP protocol, admin dashboard, and OG share card generation.
- **`apps/worker`** (pg-boss): Collection scheduler, LLM pipeline, clustering, briefing generator, and circuit-breaker budgets.
- **`apps/web`** (React Router 8 SSR + Tailwind 4): Frontend server-side rendered website.
- **`packages/backend`**: Business logic, idempotent LLM billing receipts, and single publication projection layer.
- **`industry/` & `site/`**: Decoupled industry knowledge (taxonomies, prompts, thresholds, sources).

---

## Quick Start

### Prerequisites
- Node.js >= 24
- PostgreSQL >= 17
- Docker & Docker Compose (recommended)
- OpenAI-compatible LLM API Key

### Running with Docker Compose

```bash
cp .env.example .env
# Edit .env with your database credentials and LLM configuration

docker compose build --build-arg NPM_REGISTRY=https://registry.npmmirror.com
docker compose up -d
```

Access the web interface at `http://localhost:3000`. The admin panel is available at `/admin`.

---

## Acknowledgements & License

- Built upon the open-source engine of [KKKKhazix/AIHOT](https://github.com/KKKKhazix/AIHOT) under the MIT License.
- Code licensed under the [MIT License](LICENSE).
