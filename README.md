# ProNet — Professional Contact Tracker

A modern, full-stack professional networking CRM that helps you manage your contacts, log interactions, and visualize your networking activity. Built as a production-ready demo application with clean separation between frontend and backend, designed for independent containerization.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Application Architecture](#application-architecture)
  - [High-Level Architecture](#high-level-architecture)
  - [Docker Network Architecture](#docker-network-architecture)
  - [Backend Architecture](#backend-architecture)
  - [Frontend Architecture](#frontend-architecture)
  - [Data Flow](#data-flow)
- [Project Structure](#project-structure)
- [Database Schema](#database-schema)
- [API Reference](#api-reference)
- [Setup & Installation](#setup--installation)
  - [Prerequisites](#prerequisites)
  - [Option 1: Docker (Recommended)](#option-1-docker-recommended)
  - [Option 2: Local Development](#option-2-local-development)
- [Environment Variables](#environment-variables)
- [Seeding Demo Data](#seeding-demo-data)
- [Development Workflow](#development-workflow)
- [Building for Production](#building-for-production)

---

## Overview

ProNet is a lightweight CRM built to demonstrate a modern, containerized full-stack architecture. It solves a real problem: professionals often lose track of their network — who they've spoken to, what was discussed, and who to follow up with.

ProNet provides:
- A searchable, taggable contact directory
- A full interaction history per contact (meetings, calls, emails, coffee chats, events)
- A live dashboard with activity metrics and top contact rankings

---

## Features

| Feature | Description |
|---|---|
| Contact Management | Create, edit, delete contacts with name, email, phone, company, job title, LinkedIn, tags, and notes |
| Interaction Logging | Log meetings, calls, emails, coffee chats, events, and freeform interactions per contact |
| Dashboard | Live stats — total contacts, interactions this week/month, recent activity feed, top contacts by engagement |
| Search & Filter | Debounced full-text search across name/company/email/job title; filter contacts by tag |
| Tag System | Flexible tag system on contacts using PostgreSQL native string arrays |
| Responsive UI | Clean, modern card-based layout built with Tailwind CSS |
| React Query | Automatic cache invalidation and background refetch — no manual state management |

---

## Tech Stack

### Backend
| Technology | Version | Purpose |
|---|---|---|
| Node.js | 20 | JavaScript runtime |
| Express | 4.18 | HTTP server and routing |
| TypeScript | 5.3 | Type safety |
| Prisma ORM | 5.10 | Database access, migrations, schema management |
| PostgreSQL | 16 | Relational database |
| Zod | 3.22 | Request body validation |
| dotenv | 16 | Environment variable loading |

### Frontend
| Technology | Version | Purpose |
|---|---|---|
| React | 18.3 | UI framework |
| TypeScript | 5.3 | Type safety |
| Vite | 5.1 | Build tool and dev server |
| Tailwind CSS | 3.4 | Utility-first styling |
| React Router | 6.22 | Client-side routing |
| TanStack React Query | 5.24 | Server state management, caching |
| Axios | 1.6 | HTTP client |
| Lucide React | 0.344 | Icon library |
| Inter (Google Fonts) | — | Typography |

### Infrastructure
| Technology | Purpose |
|---|---|
| Docker | Containerization |
| Docker Compose | Multi-container orchestration |
| nginx (Alpine) | Frontend static file server + API reverse proxy |

---

## Application Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Browser                             │
└─────────────────────┬───────────────────────────────────────────┘
                      │ HTTP :80
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                   FRONTEND CONTAINER                            │
│                                                                 │
│   nginx (Alpine)                                                │
│   ┌──────────────────────┐   ┌──────────────────────────────┐  │
│   │  Static File Server  │   │     Reverse Proxy            │  │
│   │                      │   │                              │  │
│   │  /usr/share/nginx/   │   │  location /api/ {            │  │
│   │  html/               │   │    proxy_pass                │  │
│   │  (React SPA build)   │   │    http://backend:3001/api/  │  │
│   │                      │   │  }                           │  │
│   │  try_files $uri       │   │                              │  │
│   │  /index.html         │   │  (uses Docker DNS to resolve │  │
│   │  (SPA fallback)      │   │  the backend service name)   │  │
│   └──────────────────────┘   └──────────────────────────────┘  │
└──────────────────────────────────────────┬──────────────────────┘
                                           │ HTTP :3001 (internal)
                                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   BACKEND CONTAINER                             │
│                                                                 │
│   Node.js + Express + TypeScript                                │
│                                                                 │
│   ┌─────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│   │  CORS   │  │ JSON Parser  │  │     Route Handlers       │  │
│   │Middleware│  │  Middleware  │  │  /api/contacts           │  │
│   └─────────┘  └──────────────┘  │  /api/interactions       │  │
│                                  │  /api/dashboard          │  │
│                                  │  /api/health             │  │
│                                  └──────────────────────────┘  │
│                                           │                     │
│                                  ┌────────▼─────────┐          │
│                                  │   Prisma Client  │          │
│                                  │  (ORM + Query    │          │
│                                  │   Builder)       │          │
│                                  └────────┬─────────┘          │
└───────────────────────────────────────────┼─────────────────────┘
                                            │ TCP :5432 (internal)
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   POSTGRES CONTAINER                            │
│                                                                 │
│   PostgreSQL 16 (Alpine)                                        │
│                                                                 │
│   Database: pronet                                              │
│   ┌───────────────────┐   ┌────────────────────────────────┐   │
│   │    Contact table  │   │      Interaction table         │   │
│   │                   │◄──┤  (FK: contactId → Contact.id   │   │
│   │  id, name, email, │   │   CASCADE DELETE)              │   │
│   │  phone, company,  │   │                                │   │
│   │  jobTitle,        │   │  id, type, subject, notes,     │   │
│   │  linkedIn, tags,  │   │  occurredAt, contactId         │   │
│   │  notes, timestamps│   │                                │   │
│   └───────────────────┘   └────────────────────────────────┘   │
│                                                                 │
│   Volume: postgres_data (persisted across restarts)            │
└─────────────────────────────────────────────────────────────────┘
```

---

### Docker Network Architecture

All three containers communicate over a single user-defined bridge network named `pronet-network`. Docker's embedded DNS resolver allows containers to address each other by their **service name** (as defined in `docker-compose.yml`) rather than by IP address.

```
                    ┌────────────────────────────────────┐
                    │         pronet-network             │
                    │        (Docker bridge)             │
                    │                                    │
                    │  ┌──────────┐   ┌──────────────┐  │
                    │  │frontend  │   │   backend    │  │
                    │  │:80       │──►│   :3001      │  │
                    │  └──────────┘   └──────┬───────┘  │
                    │                        │          │
                    │               ┌────────▼───────┐  │
                    │               │    postgres     │  │
                    │               │    :5432        │  │
                    │               └────────────────┘  │
                    └────────────────────────────────────┘

External access:
  Host :80   → frontend container :80
  Host :3001 → backend container :3001  (for direct API access / debugging)
  postgres is NOT exposed to the host
```

**Why a user-defined bridge and not the default bridge?**
User-defined bridges provide automatic DNS resolution between containers by service name. On the default bridge, containers can only communicate by IP, which changes on every restart.

**Startup order** is enforced by `depends_on` + `healthcheck`:
1. `postgres` starts and becomes healthy (pg_isready passes)
2. `backend` starts only after postgres is healthy; runs `prisma db push` then starts Express
3. `frontend` starts after backend is up

---

### Backend Architecture

```
backend/src/
│
├── index.ts                    Entry point
│   ├── Loads .env via dotenv
│   ├── Creates Express app
│   ├── Registers middleware: CORS, JSON parser
│   ├── Mounts routers at /api/*
│   ├── Registers global error handler (must be last)
│   └── Starts HTTP server
│
├── lib/
│   └── prisma.ts               Prisma Client singleton
│       └── Uses globalThis trick to prevent multiple instances
│           in development (hot-reload creates multiple modules)
│
├── routes/
│   ├── contacts.ts             /api/contacts
│   │   ├── GET  /              List all (with ?search= and ?tag= filters)
│   │   ├── GET  /:id           Get single (includes interactions + _count)
│   │   ├── POST /              Create (Zod-validated)
│   │   ├── PUT  /:id           Update (Zod-validated)
│   │   ├── DELETE /:id         Delete (cascades interactions via FK)
│   │   ├── GET  /:id/interactions  List interactions for a contact
│   │   └── POST /:id/interactions  Log new interaction (also touches contact.updatedAt)
│   │
│   ├── interactions.ts         /api/interactions
│   │   ├── PUT  /:id           Update interaction
│   │   └── DELETE /:id         Delete interaction
│   │
│   └── dashboard.ts            /api/dashboard
│       └── GET  /stats         Runs 6 Prisma queries in parallel via Promise.all:
│                               - Contact count
│                               - Interactions this week
│                               - Interactions this month
│                               - Recent 8 interactions (with joined contact)
│                               - Top 5 contacts by interaction count
│                               - Interactions grouped by type
│
└── middleware/
    └── errorHandler.ts         Global error handler (4-arg Express signature)
        ├── ZodError     → 400 with field-level errors
        ├── P2002        → 409 (unique constraint violation)
        ├── P2025        → 404 (record not found)
        └── Everything else → 500
```

**Request lifecycle:**
```
HTTP Request
    │
    ▼
CORS Middleware (checks Origin header)
    │
    ▼
JSON Body Parser (populates req.body)
    │
    ▼
Route Handler
    │
    ├── Zod schema.parse(req.body)   ← throws ZodError on invalid input
    │
    ├── Prisma query
    │
    └── res.json(result)
            │
            │  (on any thrown error)
            ▼
    Global Error Handler → structured JSON error response
```

---

### Frontend Architecture

```
frontend/src/
│
├── main.tsx                    React entry point
│   ├── QueryClient (React Query) — staleTime: 30s, retry: 1
│   ├── BrowserRouter (React Router)
│   └── ReactQueryDevtools (dev only)
│
├── App.tsx                     Route definitions
│   ├── /           → redirect to /dashboard
│   ├── /dashboard  → DashboardPage
│   ├── /contacts   → ContactsPage
│   └── /contacts/:id → ContactDetailPage
│
├── types/index.ts              TypeScript interfaces
│   ├── Contact
│   ├── Interaction
│   ├── InteractionType (enum union)
│   └── DashboardStats
│
├── api/                        HTTP layer (Axios-based)
│   ├── axios.ts                Axios instance
│   │   ├── baseURL: '/api'     (works in both dev + Docker)
│   │   └── Error interceptor   (normalizes error messages)
│   ├── contacts.ts             contactsApi.list/get/create/update/delete
│   ├── interactions.ts         interactionsApi.create/update/delete
│   └── dashboard.ts            dashboardApi.getStats
│
├── hooks/
│   └── useDebounce.ts          300ms debounce for search input
│
├── components/
│   ├── layout/
│   │   ├── AppShell.tsx        Full-height flex container (sidebar + main)
│   │   └── Sidebar.tsx         NavLink-based nav (active state via React Router)
│   │
│   ├── ui/
│   │   └── Modal.tsx           Reusable modal
│   │       ├── ESC key closes
│   │       ├── Backdrop click closes
│   │       └── Body scroll locked while open
│   │
│   ├── contacts/
│   │   ├── ContactCard.tsx     Grid card — avatar (color-coded initials), tags, interaction count
│   │   └── ContactForm.tsx     Create/Edit modal — all fields + tag management
│   │
│   └── interactions/
│       └── InteractionForm.tsx Log interaction modal — type picker, datetime, subject, notes
│
└── pages/
    ├── DashboardPage.tsx       Stats grid + recent interactions list + top contacts
    ├── ContactsPage.tsx        Contact grid with search (debounced) + tag filter
    └── ContactDetailPage.tsx   Profile card + interaction timeline
        ├── Edit/delete contact
        └── Log/delete interactions
```

**React Query cache key strategy:**

| Key | Populated by | Invalidated by |
|---|---|---|
| `['contacts']` | ContactsPage | create contact, update contact, delete contact |
| `['contact', id]` | ContactDetailPage | update contact, log interaction, delete interaction |
| `['dashboard']` | DashboardPage | create contact, delete contact, log interaction, delete interaction |

When a mutation succeeds, `queryClient.invalidateQueries()` is called for the affected keys, which triggers a background refetch. The UI shows fresh data without any manual state management.

---

### Data Flow

**Loading the contacts page:**
```
ContactsPage mounts
    │
    ▼
useQuery(['contacts', search, tag])
    │
    ├─ cache hit? → render immediately (stale-while-revalidate)
    │
    └─ cache miss or stale?
            │
            ▼
        contactsApi.list({ search, tag })
            │
            ▼
        Axios GET /api/contacts?search=...&tag=...
            │
            ▼ (Vite proxy in dev / nginx proxy in Docker)
        Express GET /api/contacts
            │
            ▼
        prisma.contact.findMany({ where: {...}, include: {_count} })
            │
            ▼
        PostgreSQL query
            │
            ▼
        JSON response → React Query cache → ContactsPage re-renders
```

**Logging an interaction:**
```
User fills InteractionForm and submits
    │
    ▼
useMutation → interactionsApi.create(contactId, data)
    │
    ▼
Axios POST /api/contacts/:id/interactions
    │
    ▼
Express route:
    1. Zod validates req.body
    2. prisma.interaction.create(...)
    3. prisma.contact.update({ updatedAt: new Date() })  ← bumps contact to top of list
    │
    ▼
onSuccess:
    invalidateQueries(['contact', contactId])   ← refreshes interaction timeline
    invalidateQueries(['dashboard'])            ← refreshes activity feed + stats
    │
    ▼
Modal closes, data updates automatically
```

---

## Project Structure

```
NetworkingDemoApp/
│
├── docker-compose.yml          Orchestrates all 3 services on pronet-network
├── .gitignore
├── CLAUDE.md                   AI assistant guidance for this repo
├── README.md                   This file
│
├── backend/
│   ├── Dockerfile              Multi-stage: deps → builder → production
│   ├── package.json
│   ├── package-lock.json
│   ├── tsconfig.json
│   ├── .env.example            Template for required environment variables
│   │
│   ├── prisma/
│   │   ├── schema.prisma       Data models, enums, relations, DB connection
│   │   └── seed.ts             Demo data script (8 contacts + 14 interactions)
│   │
│   └── src/
│       ├── index.ts
│       ├── lib/
│       │   └── prisma.ts
│       ├── routes/
│       │   ├── contacts.ts
│       │   ├── interactions.ts
│       │   └── dashboard.ts
│       └── middleware/
│           └── errorHandler.ts
│
└── frontend/
    ├── Dockerfile              Two-stage: node builder → nginx:alpine
    ├── nginx.conf              /api/ proxy + SPA fallback
    ├── package.json
    ├── package-lock.json
    ├── tsconfig.json
    ├── tsconfig.node.json
    ├── vite.config.ts          Dev server proxy: /api → localhost:3001
    ├── tailwind.config.js
    ├── postcss.config.js
    ├── index.html
    │
    └── src/
        ├── main.tsx
        ├── App.tsx
        ├── index.css           Tailwind directives + base styles
        ├── types/
        │   └── index.ts
        ├── api/
        │   ├── axios.ts
        │   ├── contacts.ts
        │   ├── interactions.ts
        │   └── dashboard.ts
        ├── hooks/
        │   └── useDebounce.ts
        ├── components/
        │   ├── layout/
        │   │   ├── AppShell.tsx
        │   │   └── Sidebar.tsx
        │   ├── ui/
        │   │   └── Modal.tsx
        │   ├── contacts/
        │   │   ├── ContactCard.tsx
        │   │   └── ContactForm.tsx
        │   └── interactions/
        │       └── InteractionForm.tsx
        └── pages/
            ├── DashboardPage.tsx
            ├── ContactsPage.tsx
            └── ContactDetailPage.tsx
```

---

## Database Schema

Defined in `backend/prisma/schema.prisma`:

```prisma
model Contact {
  id           String        @id @default(cuid())
  name         String
  email        String?       @unique
  phone        String?
  company      String?
  jobTitle     String?
  linkedIn     String?
  tags         String[]                          // PostgreSQL native array
  notes        String?
  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt          // auto-updated; also manually touched
  interactions Interaction[]                     // on delete: cascade
}

model Interaction {
  id         String          @id @default(cuid())
  type       InteractionType                     // enum
  subject    String?
  notes      String?
  occurredAt DateTime                            // when the interaction happened
  contactId  String
  contact    Contact         @relation(...)      // FK with CASCADE DELETE
  createdAt  DateTime        @default(now())
  updatedAt  DateTime        @updatedAt
}

enum InteractionType {
  MEETING | CALL | EMAIL | COFFEE | EVENT | OTHER
}
```

**Design decisions:**
- `tags` uses PostgreSQL's native `String[]` array type — no join table needed for this use case. Prisma queries use `{ has: tag }` for containment checks.
- `updatedAt` on Contact is manually touched when a new interaction is logged, so contacts bubble to the top of the list after activity.
- `onDelete: Cascade` means deleting a contact automatically removes all their interactions — no orphaned records.
- IDs use `cuid()` — collision-resistant, URL-safe, sortable by creation time.

---

## API Reference

Base URL: `http://localhost:3001` (direct) or `http://localhost/api` (via nginx proxy)

### Contacts

| Method | Endpoint | Description | Query Params |
|---|---|---|---|
| `GET` | `/api/contacts` | List all contacts | `?search=text` `?tag=tagname` |
| `POST` | `/api/contacts` | Create a contact | — |
| `GET` | `/api/contacts/:id` | Get contact with interactions | — |
| `PUT` | `/api/contacts/:id` | Update a contact | — |
| `DELETE` | `/api/contacts/:id` | Delete contact + interactions | — |
| `GET` | `/api/contacts/:id/interactions` | List interactions for a contact | — |
| `POST` | `/api/contacts/:id/interactions` | Log a new interaction | — |

### Interactions

| Method | Endpoint | Description |
|---|---|---|
| `PUT` | `/api/interactions/:id` | Update an interaction |
| `DELETE` | `/api/interactions/:id` | Delete an interaction |

### Dashboard

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/dashboard/stats` | Get aggregated KPIs |

### Health

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/health` | Service health check |

---

### Request / Response Examples

**Create a contact** `POST /api/contacts`
```json
{
  "name": "Jane Smith",
  "email": "jane@company.com",
  "phone": "+1 (555) 000-0000",
  "company": "Acme Corp",
  "jobTitle": "CTO",
  "linkedIn": "https://linkedin.com/in/janesmith",
  "tags": ["engineering", "leadership"],
  "notes": "Met at TechConf 2024."
}
```

**Log an interaction** `POST /api/contacts/:id/interactions`
```json
{
  "type": "MEETING",
  "subject": "Intro call",
  "notes": "Discussed Q3 roadmap. Very interested in our infra work.",
  "occurredAt": "2026-03-11T14:00:00.000Z"
}
```

**Dashboard stats response** `GET /api/dashboard/stats`
```json
{
  "totalContacts": 8,
  "interactionsThisWeek": 6,
  "interactionsThisMonth": 12,
  "recentInteractions": [...],
  "topContacts": [...],
  "interactionsByType": [
    { "type": "MEETING", "count": 5 },
    { "type": "CALL", "count": 3 }
  ]
}
```

**Validation error response** `400 Bad Request`
```json
{
  "message": "Validation error",
  "errors": {
    "name": ["Required"],
    "email": ["Invalid email"]
  }
}
```

---

## Setup & Installation

### Prerequisites

| Tool | Version | Notes |
|---|---|---|
| Git | any | For cloning the repo |
| Docker Desktop | 4.x+ | Includes Docker Compose v2 |
| Node.js | 20+ | Only needed for local dev (not Docker) |
| npm | 10+ | Only needed for local dev (not Docker) |
| PostgreSQL | 16+ | Only needed for local dev (not Docker) |

---

### Option 1: Docker (Recommended)

This runs the full stack — PostgreSQL, Express backend, and React frontend — in isolated containers with no local dependencies needed beyond Docker.

**Step 1: Clone the repository**
```bash
git clone https://github.com/your-username/NetworkingDemoApp.git
cd NetworkingDemoApp
```

**Step 2: Build and start all services**
```bash
docker compose up --build
```

This single command:
- Builds the backend image (Node.js → TypeScript compile → production image)
- Builds the frontend image (Vite build → nginx image)
- Pulls the PostgreSQL 16 Alpine image
- Creates the `pronet-network` bridge
- Starts postgres → waits for it to be healthy → starts backend (runs `prisma db push`) → starts frontend

Wait until you see:
```
backend-1  | ProNet API running on http://localhost:3001
```

**Step 3: Seed demo data**
```bash
docker compose exec backend sh -c "npx tsx prisma/seed.ts"
```

This loads 8 realistic contacts (engineers, investors, designers, founders) with 14 pre-existing interactions so the dashboard is populated immediately.

**Step 4: Open the app**

| Service | URL |
|---|---|
| Application | http://localhost |
| Backend API (direct) | http://localhost:3001 |
| Health check | http://localhost:3001/api/health |

**Useful Docker commands:**
```bash
# View live logs
docker compose logs -f

# View logs for a specific service
docker compose logs -f backend

# Stop all containers (data is preserved)
docker compose down

# Stop and delete all data (fresh start)
docker compose down -v

# Rebuild a single service after code changes
docker compose up --build backend

# Open a shell inside the backend container
docker compose exec backend sh

# Run Prisma Studio (database GUI) — only in local dev, not Docker
# (Studio requires a local Prisma installation; use Option 2 for this)
```

---

### Option 2: Local Development

Requires Node.js 20+ and a running PostgreSQL 16 instance.

**Step 1: Clone the repository**
```bash
git clone https://github.com/your-username/NetworkingDemoApp.git
cd NetworkingDemoApp
```

**Step 2: Configure the backend environment**
```bash
cd backend
cp .env.example .env
```

Edit `.env` with your local PostgreSQL credentials:
```env
DATABASE_URL="postgresql://YOUR_USER:YOUR_PASSWORD@localhost:5432/pronet"
PORT=3001
FRONTEND_URL="http://localhost:5173"
NODE_ENV="development"
```

Make sure the `pronet` database exists:
```bash
psql -U postgres -c "CREATE DATABASE pronet;"
```

**Step 3: Install backend dependencies and set up the database**
```bash
cd backend
npm install
npm run db:push      # creates all tables from schema.prisma
npm run db:seed      # loads demo data
```

**Step 4: Start the backend**
```bash
npm run dev
# Backend running on http://localhost:3001
```

**Step 5: Install frontend dependencies and start the dev server**
```bash
# In a new terminal
cd frontend
npm install
npm run dev
# Frontend running on http://localhost:5173
```

The Vite dev server is configured to proxy all `/api/*` requests to `http://localhost:3001`, so no CORS configuration is needed.

**Step 6: Open the app**

Navigate to **http://localhost:5173**

---

## Environment Variables

### Backend (`backend/.env`)

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | — | PostgreSQL connection string. Format: `postgresql://user:password@host:port/dbname` |
| `PORT` | `3001` | Port the Express server listens on |
| `FRONTEND_URL` | `http://localhost:5173` | Allowed CORS origin. In Docker, set to `http://localhost` |
| `NODE_ENV` | `development` | Affects Prisma logging. Set to `production` in Docker |

In Docker, these are injected via `docker-compose.yml` — no `.env` file is needed in the container.

---

## Seeding Demo Data

The seed script (`backend/prisma/seed.ts`) creates realistic demo contacts:

| Contact | Company | Role | Tags |
|---|---|---|---|
| Sarah Chen | TechCorp | VP of Engineering | engineering, leadership, mentor |
| Marcus Williams | Accel Ventures | Partner | investor, fintech, series-a |
| Priya Patel | Design Studio Co | Head of Product Design | design, product, ux |
| James Okafor | BuildWithUs | Co-founder & CTO | founder, engineering, devtools |
| Luna Rodriguez | GrowthCo | Growth Lead | growth, marketing, b2b |
| David Kim | Stack SF | Senior Software Engineer | engineering, frontend, react |
| Aisha Nakamura | DataMind AI | ML Research Lead | ai, ml, research |
| Tom Fletcher | Enterprise Corp | Head of Engineering | enterprise, engineering, potential-customer |

> **Warning:** The seed script runs `deleteMany()` on both tables before inserting. Never run it against a database with real data.

```bash
# Docker
docker compose exec backend sh -c "npx tsx prisma/seed.ts"

# Local dev
cd backend && npm run db:seed
```

---

## Development Workflow

### Adding a new API endpoint

1. Add the route in the appropriate file under `backend/src/routes/`
2. Define a Zod schema for request body validation
3. Write the Prisma query
4. Add the TypeScript type to `frontend/src/types/index.ts`
5. Add the API function to the appropriate file under `frontend/src/api/`
6. Use the function in a component via `useQuery` or `useMutation`

### Database schema changes

```bash
cd backend

# Option A: Push directly (no migration file — good for dev)
npm run db:push

# Option B: Create a migration file (good for production tracking)
npm run db:migrate:dev

# After any schema change, regenerate the Prisma client
npm run db:generate

# View the database with a GUI
npm run db:studio
```

### Running TypeScript type checks

```bash
# Backend
cd backend && npx tsc --noEmit

# Frontend
cd frontend && npx tsc --noEmit
```

---

## Building for Production

### Build Docker images independently

The backend and frontend can be built as completely independent images:

```bash
# Build backend image only
docker build -t pronet-backend ./backend

# Build frontend image only
docker build -t pronet-frontend ./frontend
```

### Deploy to the same Docker network

When deploying separately, create a shared network and connect both containers to it:

```bash
docker network create pronet-network

docker run -d \
  --name postgres \
  --network pronet-network \
  -e POSTGRES_DB=pronet \
  -e POSTGRES_USER=pronet \
  -e POSTGRES_PASSWORD=pronet_password \
  postgres:16-alpine

docker run -d \
  --name backend \
  --network pronet-network \
  -e DATABASE_URL=postgresql://pronet:pronet_password@postgres:5432/pronet \
  -e FRONTEND_URL=http://your-domain.com \
  -p 3001:3001 \
  pronet-backend

docker run -d \
  --name frontend \
  --network pronet-network \
  -p 80:80 \
  pronet-frontend
```

The frontend container's nginx will resolve `backend` by Docker's DNS since both containers share `pronet-network`.

### What each Dockerfile does

**Backend (`backend/Dockerfile`) — 3-stage build:**
```
Stage 1 (deps):      node:20-alpine + openssl → npm ci → node_modules
Stage 2 (builder):   copies deps → copies source → prisma generate → tsc
Stage 3 (production):node:20-alpine + openssl → copies dist + node_modules + prisma
                     CMD: prisma db push && node dist/index.js
```

**Frontend (`frontend/Dockerfile`) — 2-stage build:**
```
Stage 1 (builder):   node:20-alpine → npm ci → vite build → /app/dist
Stage 2 (production):nginx:alpine → copies dist → copies nginx.conf
                     CMD: nginx -g "daemon off;"
```

The final production images contain no source code, no TypeScript compiler, and no dev dependencies — only the compiled artifacts needed to run.
