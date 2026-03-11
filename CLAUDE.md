# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**ProNet** — a professional contact tracker (lightweight CRM). Professionals use it to manage their network, log interactions (meetings/calls/coffee/events), and view a dashboard of networking activity.

## Tech Stack

- **Backend**: Node.js + Express + TypeScript + Prisma ORM + PostgreSQL
- **Frontend**: React 18 + TypeScript + Vite + Tailwind CSS + React Query v5 + React Router v6 + Axios + Lucide React
- **Container**: Separate Docker images; nginx serves the frontend SPA and proxies `/api/*` to the backend

## Project Structure

```
backend/
  prisma/
    schema.prisma     Contact + Interaction models, InteractionType enum
    seed.ts           Demo data (clears all data first)
  src/
    index.ts          Express app entry — middleware, route registration, server start
    lib/prisma.ts     Prisma client singleton
    routes/
      contacts.ts     CRUD + GET/POST /api/contacts/:id/interactions
      interactions.ts PUT + DELETE /api/interactions/:id
      dashboard.ts    GET /api/dashboard/stats (aggregated KPIs via Promise.all)
    middleware/
      errorHandler.ts ZodError → 400, PrismaClientKnownRequestError P2002/P2025 → 409/404

frontend/
  nginx.conf          Proxies /api/ → http://backend:3001; SPA fallback after /api/ block
  src/
    types/index.ts    Contact, Interaction, DashboardStats interfaces
    api/              axios.ts (baseURL: '/api'), contacts.ts, interactions.ts, dashboard.ts
    hooks/
      useDebounce.ts  300ms debounce for search input
    components/
      layout/         AppShell (sidebar + main), Sidebar (NavLink-based nav)
      ui/             Modal (ESC to close, body scroll lock)
      contacts/       ContactCard, ContactForm (create/edit modal)
      interactions/   InteractionForm (log modal)
    pages/
      DashboardPage   Stats cards + recent interactions list + top contacts
      ContactsPage    Debounced search + tag filter + contact grid
      ContactDetailPage  Profile card + interaction timeline; edit/delete contact; log/delete interactions
```

## Common Commands

### Local Development (requires a local PostgreSQL instance)

```bash
# Backend
cd backend
cp .env.example .env        # set DATABASE_URL for your local Postgres
npm install
npm run db:push             # push schema to DB (dev, no migration files)
npm run db:seed             # load demo contacts + interactions
npm run dev                 # hot-reload on :3001

# Frontend (separate terminal)
cd frontend
npm install
npm run dev                 # Vite on :5173, proxies /api → :3001
```

### Docker (recommended — full stack)

```bash
docker compose up --build                           # build + start postgres/backend/frontend
# wait for backend to run migrations, then seed:
docker compose exec backend sh -c "npx tsx prisma/seed.ts"

docker compose down          # stop (keeps data volume)
docker compose down -v       # stop + delete postgres data
```

URLs when running via Docker:
- Frontend: http://localhost
- Backend API: http://localhost:3001
- Health: http://localhost:3001/api/health

### Database Management (local dev)

```bash
cd backend
npm run db:migrate:dev      # create a new migration file
npm run db:migrate          # apply pending migrations (used in Docker CMD)
npm run db:studio           # Prisma Studio at :5555
npm run db:seed             # re-seed (destructive — deletes all data first)
```

## API Reference

```
GET    /api/contacts                   ?search=&tag=
POST   /api/contacts
GET    /api/contacts/:id               includes interactions[] and _count
PUT    /api/contacts/:id
DELETE /api/contacts/:id               cascades to interactions
GET    /api/contacts/:id/interactions
POST   /api/contacts/:id/interactions  also touches contact.updatedAt
PUT    /api/interactions/:id
DELETE /api/interactions/:id
GET    /api/dashboard/stats
GET    /api/health
```

## Key Architectural Patterns

**Validation**: Zod schemas are defined inline in each route file. The `errorHandler` middleware converts `ZodError` to structured 400 responses with field-level errors, and Prisma `P2002`/`P2025` codes to 409/404.

**React Query**: All server state is managed by React Query. Query key namespaces: `['contacts']`, `['contact', id]`, `['dashboard']`. Every mutation invalidates relevant query keys on success — no manual state updates.

**API base URL**: `axios.ts` sets `baseURL: '/api'`. In dev, Vite's `server.proxy` forwards `/api` to `:3001`. In Docker, nginx proxies `/api/` to `http://backend:3001/api/`. No CORS headers are needed in either environment.

**Docker networking**: All three services share the `pronet-network` bridge. The backend's `DATABASE_URL` uses `@postgres:5432` and nginx's `proxy_pass` uses `http://backend:3001` — both resolve via Docker's internal DNS.

**Seed data**: `prisma/seed.ts` runs `deleteMany()` on both tables before inserting. Never run in production.
