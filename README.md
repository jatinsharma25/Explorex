# ExploreX

**Live Demo:** https://explorex-spsb.vercel.app · **GitHub:** https://github.com/jatinsharma25/Explorex

> Discover what's around you. Built for weekenders, wanderers, and anyone new to a city.

ExploreX helps you find the best places nearby — cafés, restaurants, bars, arcades, events — ranked by how close they are, how good they are, and how popular they are right now. No endless lists. No ads. Just a clean scrollable feed of places worth visiting.

> **Note:** This is an early MVP, actively being developed. New features and improvements are on the way.

---

## The problem it solves

Most discovery apps show you the same popular spots everyone already knows. ExploreX ranks places using a real-time scoring algorithm that weighs distance, rating, engagement, and trending momentum — so what you see is actually relevant to where you are and what's happening now.

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15 (App Router), TypeScript, CSS Modules |
| Backend | Node.js, Express, TypeScript |
| Database | PostgreSQL 16 + PostGIS 3 |
| Cache | Redis 7 |
| Infrastructure | Docker, Docker Compose |

---

## Key engineering decisions

**Geospatial queries via PostGIS** — `ST_DWithin` filters candidates within radius using a GIST spatial index. `ST_Distance` computes exact metre-level distances over the Earth's curvature (SRID 4326).

**Composite scoring in SQL** — ranking happens inside a single CTE, not in application code:

```
score = 0.35 × proximity
      + 0.20 × normalized rating
      + 0.20 × log-dampened engagement
      + 0.15 × trending score
```

**Keyset pagination** — cursors encode the last returned score as base64. No `OFFSET`, so results stay stable as data changes and DB seeks are O(1).

**Impression buffering** — `IntersectionObserver` fires after 1.5s of sustained viewport visibility. Counts hit Redis (`HINCRBY`) first; Postgres is updated in batches to avoid hot-row contention.

---

## Architecture

```
┌─────────────────────────────────────────┐
│           Next.js 15 Frontend           │
│  FeedClient → PlaceCard → /places/[id]  │
└───────────────────┬─────────────────────┘
                    │ HTTP / JSON
┌───────────────────▼─────────────────────┐
│         Express + TypeScript API        │
│  GET /v1/feed          (scored feed)    │
│  GET /v1/places/:id    (detail)         │
│  POST /v1/feed/impression               │
└──────────┬──────────────────┬───────────┘
           │                  │
   ┌───────▼──────┐   ┌───────▼──────┐
   │ PostgreSQL 16 │   │   Redis 7    │
   │ + PostGIS 3   │   │  impression  │
   │ places, media │   │   buffer     │
   │ tags, cats    │   └──────────────┘
   └───────────────┘
```

---

## Running locally

**Prerequisites:** Node.js ≥ 18, Docker Desktop, Git

```bash
# 1. Clone
git clone https://github.com/jatinsharma25/Explorex.git
cd Explorex

# 2. Start Postgres + Redis
docker compose up -d

# 3. Backend
cd backend
cp .env.example .env
npm install
npm run dev        # → http://localhost:4000

# 4. Frontend
cd ../frontend
npm install
npm run dev        # → http://localhost:3000
```

Backend `.env`:
```
DATABASE_URL=postgresql://ex_user:ex_pass@localhost:5432/explorex
REDIS_URL=redis://localhost:6379
PORT=4000
```

---

## API

```
GET  /v1/feed?lat=&lng=&radius=&limit=&cursor=&category=
GET  /v1/places/:id
POST /v1/feed/impression   { "place_id": "uuid" }
GET  /health
```

---

## Project structure

```
Explorex/
├── backend/src/
│   ├── db/              # pg pool, redis client
│   ├── routes/          # feed.ts, places.ts
│   ├── services/        # feedService.ts (scoring + pagination)
│   └── index.ts
├── frontend/src/
│   ├── app/
│   │   ├── page.tsx             # feed
│   │   └── places/[id]/page.tsx # detail
│   ├── components/      # FeedClient, PlaceCard, CategoryFilter, LocationBar
│   ├── lib/api.ts        # fetch helpers
│   └── types/index.ts
└── docker-compose.yml
```

---

## Roadmap

- [ ] Auth — user accounts
- [ ] Save places — personal lists
- [ ] Reviews — post and aggregate ratings
- [ ] Trending engine — Redis sorted sets
- [ ] Expand cities beyond Mumbai

---

Built by [Jatin Sharma](https://github.com/jatinsharma25)
