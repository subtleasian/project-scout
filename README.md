# Project Scout (MVP)

A tiny, testable **curiosity-driven discovery** app that ships a **daily stack of 7 cards** (ideas / people / networks), lets users take **micro‑actions** (Save, Track, Connect, Skip), and produces a simple **weekly reflection**. It uses **embeddings + light scoring** to surface weak-but-rising signals — with outbound links for connection (no in‑app social).

---

## 0) What this MVP does
- Ingests 2–3 curated sources (start with RSS / Substack RSS; optionally GitHub Trending, arXiv).
- Generates a short **scout note** (why it matters) + **tags** for each item.
- Embeds items and ranks by **Novelty × Momentum × Personal Fit**.
- Serves a **Daily Discovery Stack** of up to **7 cards**.
- Offers **Connect** via outbound links (LinkedIn/Substack/Twitter/GitHub).
- Logs micro‑actions; sends/serves a **weekly progress** summary.

> Design tenet: **No feeds. No DMs. No follower counts.** Just the right next cards and nudges.

---

## 1) Stack
- **Backend:** Node.js (TypeScript) + Fastify/Express
- **DB:** Postgres + **pgvector**
- **Embeddings:** `text-embedding-3-small` (1536‑D)
- **Summarization (scout note):** small LLM (configurable)
- **Auth:** Magic link (e.g., Supabase Auth or Clerk) — optional in MVP
- **Hosting:** Render / Railway / Fly.io + Neon (serverless Postgres) — pick 1

---

## 2) Repo Structure
```
project-scout/
  ├─ apps/
  │  ├─ api/
  │  │  ├─ src/
  │  │  │  ├─ index.ts                # Fastify/Express bootstrap
  │  │  │  ├─ routes/
  │  │  │  │  ├─ stack.ts            # GET /stack → 7 cards + nudges
  │  │  │  │  ├─ interact.ts         # POST /interact (save/track/skip/connect)
  │  │  │  │  ├─ weekly.ts           # GET /weekly summary
  │  │  │  ├─ lib/
  │  │  │  │  ├─ db.ts               # Postgres pool, pgvector helpers
  │  │  │  │  ├─ embed.ts            # call embeddings API
  │  │  │  │  ├─ llm.ts              # scout-note summarizer
  │  │  │  │  ├─ score.ts            # ranking params
  │  │  │  │  ├─ feeds.ts            # ingest helpers (RSS, optional GitHub)
  │  │  │  ├─ types.ts
  │  │  │  ├─ env.ts
  │  │  ├─ package.json
  │  │  └─ tsconfig.json
  │  └─ web/
  │     ├─ src/
  │     │  ├─ App.tsx                # single-page: /stack and /weekly
  │     │  ├─ components/
  │     │  │  ├─ Card.tsx            # the discovery card
  │     │  │  ├─ TokenNudge.tsx      # “add 5 more cards” CTA
  │     │  │  ├─ WeeklyBar.tsx       # simple progress bar
  │     │  ├─ api.ts                 # client API calls
  │     ├─ index.html
  │     ├─ package.json
  │     └─ tsconfig.json
  ├─ jobs/
  │  ├─ ingest.ts                    # cron: fetch feeds → llm note → embed → upsert
  │  ├─ schedule.ts                  # node-cron or hosted cron entrypoint
  ├─ config/
  │  ├─ feeds.yaml                   # list of RSS/optional sources
  │  └─ prompts/
  │     └─ scout-note.txt            # 30-word why-it-matters prompt
  ├─ db/
  │  ├─ schema.sql                   # tables + indexes (pgvector)
  │  ├─ scoring.sql                  # exact ranking SQL
  │  └─ seed.sql                     # optional sample rows
  ├─ .env.example
  ├─ README.md (this)
  └─ LICENSE
```

---

## 3) Environment Variables (`.env`)
```
# App
NODE_ENV=development
PORT=3000

# Postgres
DATABASE_URL=postgresql://user:pass@host:5432/project_scout
PGSSLMODE=require

# Embeddings / LLM
OPENAI_API_KEY=sk-...
EMBEDDING_MODEL=text-embedding-3-small
SCOUT_NOTE_MODEL=gpt-4o-mini   # or your preferred “small” model

# Data sources (optional)
FEEDS_YAML_PATH=./config/feeds.yaml
GITHUB_TOKEN=ghp_...           # only if you add GitHub API
ARXIV_ENABLED=false
```

---

## 4) Example `config/feeds.yaml`
```yaml
# Minimal, swappable without redeploy
rss:
  - name: Not Boring
    url: https://www.notboring.co/feed
  - name: The Generalist
    url: https://www.readthegeneralist.com/feed
  - name: Ben Evans
    url: https://www.ben-evans.com/benedictevans?format=rss
  - name: a16z
    url: https://a16z.com/feed/
  - name: Stratechery (sample)
    url: https://stratechery.com/feed/    # if paywalled, you’ll get titles/teasers only

# Optional future sources (handled by jobs/ingest.ts when enabled)
github_trending:
  enabled: false
  languages: ["TypeScript", "Python"]

arxiv:
  enabled: false
  categories: ["cs.AI", "cs.SI"]
```

---

## 5) Database Setup (Postgres + pgvector)
Run once in `psql` or through your admin tool:

```sql
-- Enable pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- 5.1 Core content tables
CREATE TABLE IF NOT EXISTS items (
  id               BIGSERIAL PRIMARY KEY,
  source           TEXT NOT NULL,                 -- e.g., 'rss', 'github', 'arxiv'
  url              TEXT UNIQUE NOT NULL,
  title            TEXT NOT NULL,
  author           TEXT,
  author_url       TEXT,
  published_at     TIMESTAMPTZ NOT NULL,
  raw_text         TEXT,                          -- cleaned body or abstract
  scout_note       TEXT,                          -- 1–2 sentence why-it-matters
  tags             TEXT[] DEFAULT '{}',
  stars_today      INT,                           -- for GitHub (optional)
  momentum_hint    NUMERIC,                       -- optional external metric
  created_at       TIMESTAMPTZ DEFAULT now()
);

-- Embeddings (1536-D for text-embedding-3-small)
CREATE TABLE IF NOT EXISTS item_embeddings (
  item_id          BIGINT PRIMARY KEY REFERENCES items(id) ON DELETE CASCADE,
  embedding        vector(1536)
);

-- People (derived from authors / repo owners)
CREATE TABLE IF NOT EXISTS people (
  id               BIGSERIAL PRIMARY KEY,
  name             TEXT NOT NULL,
  handle           TEXT,                          -- e.g., @handle or org slug
  url              TEXT,                          -- canonical external profile
  embedding        vector(1536),
  created_at       TIMESTAMPTZ DEFAULT now()
);

-- Users & preferences
CREATE TABLE IF NOT EXISTS users (
  id               BIGSERIAL PRIMARY KEY,
  email            TEXT UNIQUE,
  created_at       TIMESTAMPTZ DEFAULT now()
);

CREATE TYPE interaction_action AS ENUM ('save','track','skip','connect');

CREATE TABLE IF NOT EXISTS interactions (
  user_id          BIGINT REFERENCES users(id) ON DELETE CASCADE,
  item_id          BIGINT REFERENCES items(id) ON DELETE CASCADE,
  action           interaction_action NOT NULL,
  created_at       TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (user_id, item_id, action)
);

CREATE TABLE IF NOT EXISTS user_preferences (
  user_id          BIGINT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  maturity         NUMERIC DEFAULT 0.7,           -- weak ⇄ mainstream (0..1)
  voices           NUMERIC DEFAULT 0.6,           -- emerging ⇄ influencer (0..1)
  serendipity      NUMERIC DEFAULT 0.4,           -- safe ⇄ weird (0..1)
  topics           TEXT[] DEFAULT '{}'            -- optional seed interests
);

-- 5.2 Indexes
CREATE INDEX IF NOT EXISTS idx_items_published_at ON items(published_at DESC);
CREATE INDEX IF NOT EXISTS idx_items_source ON items(source);
CREATE INDEX IF NOT EXISTS idx_items_tags ON items USING GIN (tags);

-- Vector ANN index (cosine)
CREATE INDEX IF NOT EXISTS idx_item_embeddings_cosine
  ON item_embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

> Note: `ivfflat` requires analyzing the table after load for best performance: `ANALYZE item_embeddings;`

---

## 6) Exact Scoring SQL (Daily Stack)
This query computes **Novelty**, **Momentum**, and **Personal Fit** for a given user and returns the **top N** candidates (default 30). Your API can then ask a small LLM to pick the **best 7**.

**Inputs**
- `:user_id` — current user
- `:as_of` — timestamp (defaults to `now()`)
- `:days_window` — novelty lookback window (e.g., 30)
- `:limit` — how many candidates to return before the LLM rerank (e.g., 30)

```sql
WITH params AS (
  SELECT
    CAST(:user_id AS BIGINT)     AS user_id,
    COALESCE(CAST(:as_of AS TIMESTAMPTZ), now()) AS as_of,
    COALESCE(CAST(:days_window AS INT), 30) AS days_window,
    COALESCE(CAST(:limit AS INT), 30) AS lim
),

-- Items in the last N days (for novelty baseline)
recent AS (
  SELECT i.id, e.embedding
  FROM items i
  JOIN item_embeddings e ON e.item_id = i.id
  JOIN params p ON TRUE
  WHERE i.published_at >= p.as_of - (p.days_window || ' days')::interval
),

-- User centroid from prior positive interactions (save/track)
user_centroid AS (
  SELECT AVG(e.embedding) AS emb
  FROM interactions x
  JOIN item_embeddings e ON e.item_id = x.item_id
  JOIN params p ON TRUE
  WHERE x.user_id = p.user_id AND x.action IN ('save','track')
),

-- Candidate pool: last 7 days not yet interacted-with
candidates AS (
  SELECT i.*, e.embedding
  FROM items i
  JOIN item_embeddings e ON e.item_id = i.id
  JOIN params p ON TRUE
  WHERE i.published_at >= p.as_of - interval '7 days'
    AND NOT EXISTS (
      SELECT 1 FROM interactions x
      WHERE x.user_id = p.user_id AND x.item_id = i.id
    )
),

-- Novelty: 1 - avg cosine similarity to recent items
novelty AS (
  SELECT c.id,
         1 - COALESCE(AVG(1 - (c.embedding <=> r.embedding)), 0.0) AS novelty_score
  FROM candidates c
  LEFT JOIN recent r ON TRUE
  GROUP BY c.id
),

-- Momentum: combine time decay + any available hint (e.g., stars_today)
raw_momentum AS (
  SELECT c.id,
         -- half-life ≈ 7 days time decay in [0,1]
         POWER(0.5, EXTRACT(EPOCH FROM ((SELECT as_of FROM params) - c.published_at)) / 604800.0) AS time_decay,
         COALESCE(NULLIF(c.stars_today,0), 0) AS stars_today,
         COALESCE(c.momentum_hint, NULL) AS ext_hint
  FROM candidates c
),

momentum_norm AS (
  SELECT id,
         time_decay,
         CASE WHEN MAX(stars_today) OVER () > 0 THEN stars_today::float / NULLIF(MAX(stars_today) OVER (),0) ELSE 0 END AS stars_norm,
         CASE WHEN MAX(ext_hint)  OVER () IS NOT NULL THEN ext_hint::float / NULLIF(MAX(ext_hint) OVER (),0) ELSE NULL END AS hint_norm
  FROM raw_momentum
),

momentum AS (
  SELECT id,
         -- weight time 0.5, stars 0.3, external hint 0.2 (if present)
         GREATEST(0.0, LEAST(1.0,
           0.5 * time_decay +
           0.3 * stars_norm +
           0.2 * COALESCE(hint_norm, 0)
         )) AS momentum_score
  FROM momentum_norm
),

-- Personal fit: cosine similarity to user centroid (if any)
personal_fit AS (
  SELECT c.id,
         COALESCE(1 - (c.embedding <=> (SELECT emb FROM user_centroid)), 0.0) AS personal_fit_score
  FROM candidates c
)

SELECT
  c.id,
  c.title,
  c.url,
  c.author,
  c.author_url,
  c.published_at,
  c.scout_note,
  c.tags,
  n.novelty_score,
  m.momentum_score,
  p.personal_fit_score,
  -- master score
  (0.45 * n.novelty_score + 0.35 * m.momentum_score + 0.20 * p.personal_fit_score) AS score
FROM candidates c
JOIN novelty n      ON n.id = c.id
JOIN momentum m     ON m.id = c.id
JOIN personal_fit p ON p.id = c.id
ORDER BY score DESC, published_at DESC
LIMIT (SELECT lim FROM params);
```

**Notes**
- Cosine distance operator `<=>` comes from pgvector; we convert to similarity via `1 - distance`.
- If a user has no history, `personal_fit` gracefully falls back to 0.0.
- For pure RSS (no stars), momentum still works via **time decay**; you can later add phrase‑frequency or cross‑source mention counts to strengthen it.

---

## 7) API Sketch

**GET `/stack`** → returns up to 7 cards (+ optional 1–2 connection nudges)
- Query: `?limit=7&as_of=...`
- Body: `{ items:[{id,title,url,scout_note,tags,novelty,momentum,fit,author,author_url}], nudges:[{type:'people', items:[...]}] }`

**POST `/interact`**
- Body: `{ itemId, action: 'save'|'track'|'skip'|'connect' }`

**GET `/weekly`**
- Returns counts + simple theme breakdown based on tags

---

## 8) Scout Note Prompt (`config/prompts/scout-note.txt`)
```
You are a concise discovery scout. Summarize why this item matters in ≤30 words.
Include who should care, and 3 short tags (comma-separated). Avoid hype and jargon.
Output format:
Note: <one sentence>
Tags: tag1, tag2, tag3
```

---

## 9) Jobs (Ingest → Summarize → Embed → Upsert)
Pseudocode outline (see `jobs/ingest.ts`):
```
for each source in feeds.yaml:
  fetch latest items (last 24–48h)
  clean + truncate raw_text (e.g., 2–3k chars)
  if url not in items:
    insert items row (source, url, title, author, author_url, published_at, raw_text)
    call LLM(scout-note prompt) → parse note + tags → update items
    call Embeddings API(raw_text or note) → upsert item_embeddings
    derive person (author) vector: avg of their items → upsert into people
```

---

## 10) UI (Daily Discovery Stack)
- Single route `/stack` that displays up to **7 cards**.
- Card shows: title, favicon, 30‑word note, 3 tags, tiny novelty/momentum pips.
- Buttons: **Save**, **Track**, **Connect** (opens `author_url`), **Skip**.
- After 3+ positive actions in a theme, show a **nudge** with 1–2 people (from `people` nearest neighbors to saved items).
- Monetization hook: **“Add 5 more cards today (1 token)”** — simple CTA placeholder.

---

## 11) Quickstart

```bash
# 1) Clone
git clone https://github.com/you/project-scout
cd project-scout

# 2) Env
cp .env.example .env
# fill in DATABASE_URL and OPENAI_API_KEY

# 3) DB
psql "$DATABASE_URL" -f db/schema.sql

# 4) Seed feeds
cp config/feeds.yaml config/feeds.local.yaml   # optional

# 5) Ingest (one-time to get some cards)
pnpm -C apps/api install && pnpm -C apps/web install
pnpm ts-node jobs/ingest.ts

# 6) Run API + Web
pnpm -C apps/api dev
pnpm -C apps/web dev
```

---

## 12) Roadmap (post-MVP)
- Add GitHub Trending + simple stars_today → stronger momentum.
- Add arXiv (cs.AI, cs.SI) + phrase-frequency momentum.
- LLM rerank the top 30 → final 7 + justification.
- Token economy: earn units (save/track/connect), spend on extra cards.
- Weekly email digest using the same summary data.
- Admin: edit `feeds.yaml` from UI.

---

## 13) License
MIT.

