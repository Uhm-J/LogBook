# Logbook Web App – Project Outline

---

## 1 · Vision

A **friction‑free digital logbook** where users open a page, start typing, attach files/images, choose a project, and instantly see entries plotted on a vertical timeline. Designed for individual researchers now, extendable to collaborative workspaces later.

---

## 2 · Core Requirements

| Need             | Details                                                                        |
| ---------------- | ------------------------------------------------------------------------------ |
| Quick capture    | One‑click textbox, optional file/image drag‑drop, select/create project inline |
| Entry types      | `log`, `milestone` (extensible enum)                                           |
| Timeline UI      | Vertical axis with dated nodes (colored by type) matching provided sketch      |
| Projects sidebar | Scrollable list with unread counters + create button                           |
| Edit / delete    | Soft‑delete, versioned edits (audit log)                                       |
| Accounts         | Email + password now; Google / GitHub OAuth next phase                         |
| Security         | JWT sessions, CSRF‑safe, RBAC groundwork                                       |
| Future team mode | Multiple members per project, invites, roles                                   |

---

## 3 · High‑Level Architecture

```
[ React + shadcn/ui SPA ] <—HTTP/JSON—> [ Go REST API ] ——— PostgreSQL
                                         ↳ (object store) S3‑compatible for attachments
```

* **Frontend**: Vite + React + TypeScript, shadcn/ui components, React Router, React Query (data‑fetching), Zustand (lightweight global state).
* **Backend**: Go 1.22, Fiber (or Gin) HTTP framework, GORM ORM, `go-chi/jwtauth` for JWT, `golang-migrate` for migrations.
* **Storage**: PostgreSQL 15 for relational data, object storage (MinIO or AWS S3) for large files.
* **Auth**: Email/password & JWT today → OIDC (Google/GitHub) plug‑in later via `coreos/go-oidc`.

---

## 4 · Database Schema (v1)

```mermaid
erDiagram
    users ||--o{ projects : owns
    users ||--o{ audit_trails : "makes"
    projects ||--o{ log_entries : contains
    projects ||--o{ project_members : "has"    %% dormant until team mode

    users {
        uuid id PK
        text email
        text password_hash
        timestamptz created_at
    }
    projects {
        uuid id PK
        uuid owner_id FK
        text name
        text description
        timestamptz created_at
    }
    log_entries {
        uuid id PK
        uuid project_id FK
        uuid author_id FK
        text type  -- 'log' | 'milestone' | future
        text title
        text body_md
        jsonb attachments  -- array of {url, filename}
        timestamptz created_at
        timestamptz updated_at
        bool deleted DEFAULT false
    }
    audit_trails {
        uuid id PK
        uuid log_entry_id FK
        uuid editor_id FK
        jsonb diff  -- patch between versions
        timestamptz edited_at
    }
    project_members {
        uuid id PK
        uuid project_id FK
        uuid user_id FK
        text role  -- 'owner'|'writer'|'reader'
    }
```

---

## 5 · REST API Surface (abridged)

| Method | Endpoint                          | Purpose                                 |
| ------ | --------------------------------- | --------------------------------------- |
| POST   | /v1/auth/register                 | create account                          |
| POST   | /v1/auth/login                    | issue JWT                               |
| GET    | /v1/projects                      | list owned projects                     |
| POST   | /v1/projects                      | create project                          |
| GET    | /v1/projects/{id}                 | project details + latest timeline batch |
| PATCH  | /v1/projects/{id}                 | rename etc.                             |
| DELETE | /v1/projects/{id}                 | soft‑delete project                     |
| GET    | /v1/projects/{id}/entries?cursor= | paginated timeline                      |
| POST   | /v1/projects/{id}/entries         | add log/milestone                       |
| PATCH  | /v1/entries/{id}                  | edit (version stored)                   |
| DELETE | /v1/entries/{id}                  | soft‑delete                             |
| GET    | /v1/entries/{id}/history          | list audit versions                     |
| PUT    | /v1/files                         | generate signed upload URL              |

All endpoints secured by `Authorization: Bearer <token>` header.

---

## 6 · Frontend Structure

```
/src
  └─ components/
       ├─ SidebarProjectList.tsx
       ├─ TimelineView.tsx        // renders nodes with framer‑motion
       ├─ EntryCard.tsx           // individual node detail
       └─ NewEntryModal.tsx       // quick‑capture form
  └─ pages/
       ├─ Dashboard.tsx           // list + welcome
       └─ Project.tsx            // timeline + right drawer details
  └─ hooks/
       └─ useEntries.ts           // React Query wrappers
  └─ lib/
       └─ api.ts                 // Axios instance
```

* **shadcn/ui** for cards, buttons, modal, scrollbar.
* **framer‑motion** for timeline slide‑in animations.
* **virtualized list** (react‑window) for infinite scroll on timeline.

---

## 7 · Auditing Strategy

1. All `PATCH / DELETE` endpoints wrap logic in a DB transaction.
2. Before mutate, read current row, compute JSON Patch diff (`evanphx/json-patch`).
3. Insert into `audit_trails` with `editor_id` and diff.
4. Soft‑delete sets `deleted = true`; nothing ever hard‑deleted unless a nightly prune job.

---

## 8 · Extensibility Hooks

* **Entry types** table or enum so UI automatically lists new kinds.
* **Project\_roles** table already exists ⇢ invite flow later.
* **Event bus** (e.g., RabbitMQ or lightweight `go-events`) to stream audit events for future real‑time UI.
* **Service layer** interfaces to swap persistence (e.g., CockroachDB).

---

## 9 · CI / CD & DevOps

| Aspect | Choice                                            |
| ------ | ------------------------------------------------- |
| Repo   | mono‑repo (frontend & backend)                    |
| CI     | GitHub Actions: lint → test → build → docker‑push |
| CD     | Fly.io / Render / AWS ECS                         |
| Infra  | Docker Compose (dev), Terraform (prod)            |
| Docs   | Tech spec in repo, Swagger UI auto‑generated      |

---

## 10 · Development Milestones

| Phase                     | Deliverables                                    | Est. time |
| ------------------------- | ----------------------------------------------- | --------- |
| **0** – Setup             | Repo, Docker, CI, lint/test harness             | ½ week    |
| **1** – Auth & Projects   | User model, JWT, project CRUD (CLI curl)        | 1 week    |
| **2** – Entries API       | log/milestone endpoints, audit log, S3 uploads  | 1½ weeks  |
| **3** – Frontend MVP      | Sidebar, project timeline, new‑entry modal      | 2 weeks   |
| **4** – Polish            | Edit/delete UI, audit history modal, animations | 1 week    |
| **5** – OAuth & hardening | Google/GitHub, rate limits, e2e tests           | 1 week    |
| **6** – Beta              | Bug‑fixes, feedback, docs, deploy to prod URL   | 1 week    |

---

## 11 · Nice‑to‑Have Backlog

* Markdown preview & code syntax highlighting.
* Tagging / filtering on timeline.
* Full‑text search (Postgres → pg\_trgm / Elastic).
* Notifications (email / in‑app) when someone comments or mentions you (post‑team mode).
* Mobile‑first PWA shell.

---

## 12 · License & IP

Internal use → MIT or GPL7 depending on open‑source strategy. Confirm with stakeholders.

---

**End of Outline – ready for review & iteration.**

## 13 · Mono‑Repo Directory Layout

```
logbook/
├── backend/               # Go service
│   ├── cmd/               # entrypoints
│   │   └── logbook/       # main.go (wire DI, start server)
│   ├── internal/          # private application code
│   │   ├── app/           # domain logic
│   │   │   ├── models/    # GORM models + primitives
│   │   │   ├── services/  # use‑cases (business rules)
│   │   │   ├── handlers/  # HTTP route handlers -> services
│   │   │   └── middleware/# auth, logging, recovery, rate‑limit
│   │   ├── pkg/           # shared libs (public inside repo)
│   │   │   ├── db/        # db connection + migrations
│   │   │   ├── auth/      # JWT + OIDC helpers
│   │   │   └── storage/   # S3 abstraction layer
│   │   └── config/        # env parsing, flags, secrets
│   ├── migrations/        # SQL files (golang‑migrate)
│   └── go.mod
│
├── frontend/              # React + shadcn/ui
│   ├── src/
│   │   ├── components/    # ui widgets (Timeline, Sidebar…)
│   │   ├── pages/         # route components (Dashboard, Project)
│   │   ├── hooks/         # custom hooks (useEntries, useAuth)
│   │   ├── lib/           # api.ts (Axios instance & interceptors)
│   │   ├── context/       # AuthProvider, ToastContext
│   │   ├── routes/        # React Router definitions
│   │   └── types/         # generated (openapi‑typescript)
│   ├── public/
│   ├── vite.config.ts
│   ├── tailwind.config.ts
│   └── package.json
│
├── infra/
│   ├── docker-compose.yml # dev stack (pg, minio, backend, frontend)
│   ├── terraform/         # optional prod IaC
│   └── scripts/           # helper bash scripts (seed, reset)
│
├── .github/workflows/     # CI pipelines (lint → test → build → push)
└── README.md
```

*Rationale*: **internal/** keeps non‑exported Go code private; **cmd/** hosts thin `main.go` wrappers. React folder mirrors feature‑first structure for scalability.

---

## 14 · HTTP Handling Best Practices

### 14.1 · Go Backend

| Concern                | Guideline                                                            | Snippet/Lib                                 |
| ---------------------- | -------------------------------------------------------------------- | ------------------------------------------- |
| **Router**             | Prefer chi or Fiber for lightweight + middleware chain               | `github.com/go-chi/chi/v5`                  |
| **Versioning**         | Prefix all routes with `/v1` → future `/v2`                          | `r.Route("/v1", …)`                         |
| **Context**            | Pass `request‑id`, `userID`, `deadline` through `context.Context`    | `ctx := r.Context()`                        |
| **Middleware order**   | `recover → requestID → logger → cors → auth → rateLimit`             |                                             |
| **Structured logging** | Use zap + request‑id field                                           | `zap.Logger.With(zap.String("req_id", id))` |
| **Validation**         | DTO structs + `validator.New()` before hitting service layer         | `validate.Struct(&payload)`                 |
| **Error model**        | Central error types mapped to HTTP codes by middleware               | `ErrNotFound → 404`                         |
| **Pagination**         | Cursor‑based (`?cursor=<uuid>&limit=50`) not page/size               |                                             |
| **Streaming uploads**  | Pre‑signed S3 URLs; backend never buffers file                       |                                             |
| **OpenAPI spec**       | `swag init` or `kin-openapi` → generate docs & TS types              |                                             |
| **Security headers**   | `Strict‑Transport‑Security`, `X‑Content‑Type‑Options` via middleware |                                             |
| **Graceful shutdown**  | `http.Server.Shutdown(ctx)` on SIGINT/SIGTERM                        |                                             |

### 14.2 · React Frontend

| Concern                    | Guideline                                                                                                                        |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **HTTP client**            | Single Axios instance in `lib/api.ts` with: baseURL env var, JSON parsing, timeout, `withCredentials` false, retry (axios‑retry) |
| **Interceptors**           | ① inject JWT from `localStorage` ② global 401 handler → push to `/login`                                                         |
| **Data‑fetching**          | React Query for caching, SWR, stale‑while‑revalidate                                                                             |
| **Type safety**            | Autogen hooks/interfaces from OpenAPI → `openapi‑typescript-codegen`                                                             |
| **Error boundary**         | Wrap route components; show toast+retry                                                                                          |
| **Mutation optimistic UI** | Use `onMutate` / `queryClient.invalidateQueries`                                                                                 |
| **Suspense**               | Prefer `<React.Suspense>` with `ErrorBoundary` for async routes                                                                  |
| **Folder hygiene**         | Co-locate tests (`*.test.tsx`) & styles with component; avoid deep nesting                                                       |

### 14.3 · Cross‑Cutting Patterns

* **Idempotency keys** for POST /entries to avoid double submit.
* **Rate limiting** with `ulule/limiter` (Redis) – plan for public API.
* **Request tracing**: propagate `traceparent` header (W3C) → ready for Jaeger.
* **JSON‑only** responses, UTF‑8, snake\_case keys; version changes handled via new fields not breaking.

---

*Next*: Want sample handler code or Axios template?

## 15 · Visual Style Guide

### 15.1 · Brand Palette

| Token           | Hex         | Suggested Usage                                      |
| --------------- | ----------- | ---------------------------------------------------- |
| `--clr‑base‑00` | **#F2F0EF** | Global page background, neutral surfaces             |
| `--clr‑base‑20` | **#BBBDBC** | Borders, disabled UI, secondary text                 |
| `--clr‑primary` | **#245F73** | Action buttons, log entry nodes, links               |
| `--clr‑accent`  | **#733E24** | Milestone nodes, destructive buttons, headings hover |

> **Contrast note**: `#245F73` on `#F2F0EF` meets WCAG AA for normal text (\~6.3:1). Use white text (`#FFFFFF`) on `#245F73` buttons for >4.5:1 contrast.

### 15.2 · Typography

* **Font family**: `Poppins`, fallback `ui‑sans‑serif`, `system‑ui`
  *Import via* → `<link href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;600;700&display=swap" rel="stylesheet">`
* **Scale** (rem): `12, 14, 16, 20, 24, 32, 40` (body default 16 px).
* **Weights**: 300 (light), 400 (regular), 600 (semibold) for cards/titles, 700 for h1/h2.

### 15.3 · Spacing & Layout

| Token      | px | Use                            |
| ---------- | -- | ------------------------------ |
| `space‑xs` | 4  | icon padding, chip gaps        |
| `space‑sm` | 8  | small component margin         |
| `space‑md` | 16 | default grid gap, card padding |
| `space‑lg` | 24 | modal padding, section gutters |
| `space‑xl` | 40 | page gutters on ≥lg screens    |

*Grid*: `max‑w‑screen‑lg`, 12‑column, 24 px gutters. Timeline column fixed 68 px (rail + nodes).

### 15.4 · Component Treatments

* **Sidebar**: darkened `#245F73` gradient (top → bottom 5%) with white project list text; active project highlight `bg‑white/10`.
* **Timeline nodes**:
   `log` → stroke `#245F73`, fill white on hover
   `milestone` → stroke `#733E24`, fill white; filled when completed.
* **Buttons**:
   Primary `bg‑[#245F73]` / hover `bg‑[#245F73] darken‑6%` / text‑white
   Danger `bg‑[#733E24]`
   Secondary outline uses `#245F73` border + text.
* **Cards/Modals**: `bg‑white`, 2 px radius, subtle shadow `rgba(0,0,0,0.04) 0 4 8 0`.
* **Inputs**: border `#BBBDBC`, focus ring `#245F73` at 1 px inset.

### 15.5 · Tailwind Config Snippet

```ts
// tailwind.config.ts
import { fontFamily } from "tailwindcss/defaultTheme";
export default {
  theme: {
    extend: {
      colors: {
        base: {
          0:  "#F2F0EF",
          20: "#BBBDBC",
        },
        primary: "#245F73",
        accent:  "#733E24",
      },
      fontFamily: {
        sans: ["Poppins", ...fontFamily.sans],
      },
    },
  },
};
```

### 15.6 · Iconography & Motion

* **Icons**: lucide‑react, 1.25 rem, stroke 1.5 px, inherit color.
* **Animations**: framer‑motion ease‑in‑out 250 ms for card fade/slide; timeline node pop‑scale 1.0 → 1.1 on hover.

### 15.7 · Accessibility Checklist

1. Color contrasts AA verified via `@tailwindcss/aspect‑ratio` plugin docs.
2. Keyboard focus outline 2 px `#245F73` offset.
3. Prefer `aria‑live` regions for toast messages.

---

*Style guide ready. Update Tailwind + shadcn/ui override files accordingly.*
