# Project N.O.M.A.D. — Codebase Architecture

This document is a technical guide for developers who want to understand, extend, or contribute to the Project N.O.M.A.D. codebase.

For general project information and user-facing installation instructions, see the [README](README.md).  
For contribution guidelines, see [CONTRIBUTING.md](CONTRIBUTING.md).

---

## Table of Contents

- [Repository Layout](#repository-layout)
- [Tech Stack](#tech-stack)
- [Backend Architecture](#backend-architecture)
  - [Entry Points](#entry-points)
  - [Request Lifecycle](#request-lifecycle)
  - [Service Layer](#service-layer)
  - [Background Jobs](#background-jobs)
  - [Database Models](#database-models)
  - [Real-time Communication](#real-time-communication)
- [Frontend Architecture](#frontend-architecture)
  - [Inertia.js Bridge](#inertiajs-bridge)
  - [Pages and Components](#pages-and-components)
  - [State Management](#state-management)
- [Docker Integration](#docker-integration)
- [AI / RAG Pipeline](#ai--rag-pipeline)
- [Offline Content System](#offline-content-system)
- [Configuration Reference](#configuration-reference)
- [Development Setup](#development-setup)

---

## Repository Layout

```
project-nomad/
├── admin/                      # The main application (backend + frontend)
│   ├── app/
│   │   ├── controllers/        # HTTP request handlers (thin layer)
│   │   ├── models/             # Lucid ORM database models
│   │   ├── services/           # Business logic (the bulk of the codebase)
│   │   ├── jobs/               # BullMQ background job handlers
│   │   ├── middleware/         # HTTP middleware (auth guards, etc.)
│   │   ├── validators/         # Input validation schemas
│   │   ├── policies/           # Authorization policies
│   │   ├── exceptions/         # Custom exception classes
│   │   └── listeners/          # Event listeners
│   ├── inertia/                # React frontend
│   │   ├── app/                # Application entry point (app.tsx)
│   │   ├── pages/              # Full-page React components (one per route)
│   │   ├── components/         # Reusable UI components
│   │   ├── layouts/            # Page layout wrappers
│   │   ├── hooks/              # Custom React hooks
│   │   ├── context/            # React context providers
│   │   ├── lib/                # Utility/helper functions
│   │   └── css/                # Tailwind CSS entry point
│   ├── database/
│   │   ├── migrations/         # Lucid schema migrations (25+)
│   │   └── seeders/            # Optional seed data
│   ├── config/                 # AdonisJS configuration files (13 files)
│   ├── start/                  # Bootstrap: routes.ts, kernel.ts, env.ts
│   ├── bin/                    # Entry points: server.ts, console.ts, test.ts
│   ├── docs/                   # In-app documentation (markdown files)
│   ├── public/                 # Static assets served directly
│   ├── resources/              # Edge/Inertia HTML layout template
│   ├── tests/                  # JAPA test suites (unit & functional)
│   ├── adonisrc.ts             # AdonisJS framework configuration
│   ├── vite.config.ts          # Vite bundler configuration
│   └── tailwind.config.ts      # Tailwind CSS configuration
├── collections/                # Curated content collection JSON manifests
│   ├── kiwix-categories.json   # Wikipedia/reference content catalogue
│   ├── maps.json               # Downloadable map regions catalogue
│   └── wikipedia.json          # Wikipedia content selections
├── install/                    # Deployment and system scripts
│   ├── install_nomad.sh        # Full system installer
│   ├── uninstall_nomad.sh      # System uninstaller
│   ├── start_nomad.sh          # Start all containers
│   ├── stop_nomad.sh           # Stop all containers
│   ├── update_nomad.sh         # Pull and recreate management containers
│   ├── entrypoint.sh           # Docker container entry point
│   ├── management_compose.yaml # Docker Compose stack definition
│   └── sidecar-updater/        # Background container update watcher
├── Dockerfile                  # Multi-stage Docker image build
├── README.md                   # User-facing project overview
├── CONTRIBUTING.md             # Contribution guide
└── .releaserc.json             # Semantic-release automation config
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend framework | [AdonisJS 6](https://adonisjs.com/) (Node.js, TypeScript) |
| ORM | [Lucid](https://lucid.adonisjs.com/) (MySQL 8) |
| Job queue | [BullMQ](https://docs.bullmq.io/) backed by Redis 7 |
| Frontend framework | [React 19](https://react.dev/) |
| Backend ↔ Frontend bridge | [Inertia.js](https://inertiajs.com/) |
| Build tool | [Vite 6](https://vitejs.dev/) |
| Styling | [Tailwind CSS 4](https://tailwindcss.com/) |
| Server state | [TanStack React Query 5](https://tanstack.com/query) |
| Real-time | [@adonisjs/transmit](https://docs.adonisjs.com/guides/digging-deeper/transmit) (SSE) |
| AI inference | [Ollama](https://ollama.com/) (local LLM) |
| Vector store | [Qdrant](https://qdrant.tech/) |
| Container management | [Dockerode](https://github.com/apocas/dockerode) |
| Offline content | [Kiwix/libzim](https://www.kiwix.org/) (ZIM files) |
| Offline maps | [ProtoMaps](https://protomaps.com/) + [MapLibre GL](https://maplibre.org/) |
| HTTP testing | [JAPA](https://japa.dev/) |
| Language | TypeScript 5 (both server and client) |

---

## Backend Architecture

### Entry Points

| File | Role |
|---|---|
| `admin/bin/server.ts` | HTTP server bootstrap — loads env, providers, then starts listening |
| `admin/bin/console.ts` | Ace CLI entry point for running migrations, seeds, and custom commands |
| `admin/bin/test.ts` | JAPA test runner entry point |
| `admin/start/routes.ts` | All HTTP route definitions |
| `admin/start/kernel.ts` | Middleware stack registration |
| `admin/start/env.ts` | Environment variable validation and typing |

On first boot, `server.ts` also reconciles the content manifest (comparing files on disk with the database) so the UI always reflects reality without requiring a manual sync step.

### Request Lifecycle

```
HTTP Request
     │
     ▼
Middleware chain (kernel.ts)
  • Shield (CSRF, security headers)
  • BodyParser
  • Session
  • Custom middleware (e.g. status checks)
     │
     ▼
Router (start/routes.ts)
     │
     ▼
Controller  (app/controllers/)
  • Validates input (app/validators/)
  • Delegates to one or more Services
  • Returns an Inertia response or JSON
     │
     ▼
Service  (app/services/)
  • Holds all business logic
  • Interacts with Models, Docker, Ollama, etc.
     │
     ▼
Response (Inertia page render or JSON)
```

Controllers are intentionally thin. All meaningful logic lives in services.

### Service Layer

Services are injected via the AdonisJS IoC container using the `@inject()` decorator. Each service owns a specific domain:

| Service | Responsibility |
|---|---|
| `DockerService` | Start, stop, inspect, and update Docker containers |
| `ChatService` | Create and manage chat sessions and message history |
| `OllamaService` | Pull models, stream chat completions, generate embeddings |
| `RagService` | Embed documents, query the Qdrant vector store, rerank results |
| `DownloadService` | Enqueue and track content downloads (ZIM files, maps, models) |
| `ZimService` | List, read, and search ZIM library content |
| `MapsService` | Manage ProtoMaps tile files and metadata |
| `BenchmarkService` | Run hardware benchmark suites and record results |
| `CollectionService` | Reconcile on-disk content with the database manifest |
| `SystemService` | Read hardware info, check internet connectivity, report status |
| `EasySetupService` | Guide first-time setup through a multi-step wizard |
| `KVStoreService` | Read and write runtime configuration key-value pairs |

### Background Jobs

Long-running or I/O-heavy operations are offloaded to BullMQ workers so the HTTP response is not blocked:

| Job | Trigger | Worker queue |
|---|---|---|
| `RunDownloadJob` | User selects content to download | `downloads` |
| `DownloadModelJob` | User selects an Ollama model | `model-downloads` |
| `EmbedFileJob` | File uploaded to the RAG knowledge base | `embeddings` |
| `RunBenchmarkJob` | User starts a benchmark run | `benchmarks` |
| `CheckUpdateJob` | Scheduled or manual update check | `updates` |

Workers are started separately from the main web process:

```bash
# From admin/
npm run work:downloads
npm run work:model-downloads
npm run work:benchmarks
npm run work:all          # starts all queues
```

Progress and status updates are pushed back to the browser over SSE (see [Real-time Communication](#real-time-communication)).

### Database Models

Lucid ORM models live in `app/models/`. Key entities:

| Model | Table | Purpose |
|---|---|---|
| `Service` | `services` | Tracks installable Docker-based tools and their state |
| `ChatSession` | `chat_sessions` | Groups messages into a named conversation |
| `ChatMessage` | `chat_messages` | Individual messages with role, content, and metadata |
| `BenchmarkResult` | `benchmark_results` | Stored hardware benchmark scores |
| `BenchmarkSetting` | `benchmark_settings` | User-configurable benchmark parameters |
| `CollectionManifest` | `collection_manifests` | Tracks downloaded content (type, path, size) |
| `WikipediaSelection` | `wikipedia_selections` | Which Wikipedia categories the user has chosen |
| `KVStore` | `kv_store` | Arbitrary key-value configuration storage |

Migrations are in `database/migrations/` and run automatically on startup via `entrypoint.sh`.

### Real-time Communication

The `@adonisjs/transmit` package provides **Server-Sent Events (SSE)** channels. The frontend subscribes to named channels and receives push updates without polling:

- Download progress events
- Job completion / failure notifications
- Container status changes
- System health updates

The transmit configuration lives in `config/transmit.ts`.

---

## Frontend Architecture

### Inertia.js Bridge

[Inertia.js](https://inertiajs.com/) connects the AdonisJS backend to the React frontend without a separate REST API layer for page rendering:

1. A controller calls `inertia.render('PageName', props)`.
2. On a full-page load, the server renders an HTML shell that includes the initial props as JSON.
3. React hydrates the page on the client.
4. Subsequent navigations are intercepted by Inertia, which makes an XHR request and swaps only the page component — no full reload.

The root template is `resources/views/inertia_layout.edge`.  
The React entry point is `inertia/app/app.tsx`.

Shared data that every page receives (e.g., system status flags) is defined in `config/inertia.ts`.

### Pages and Components

```
inertia/
├── pages/          # One file per route (e.g. Home.tsx, Chat.tsx, Maps.tsx)
├── components/     # Reusable UI building blocks
├── layouts/        # Wrappers that provide a consistent page shell
├── hooks/          # Custom hooks (e.g. useDownloadProgress, useSystemStatus)
├── context/        # React context providers for cross-cutting state
└── lib/            # Pure utility functions
```

Each `pages/` file corresponds directly to an Inertia render call in a controller. Props are typed via shared TypeScript interfaces.

### State Management

| Concern | Tool |
|---|---|
| Server data (fetched via API routes) | TanStack React Query |
| Inertia-provided page props | Inertia's `usePage()` hook |
| Global UI state (modals, theme, etc.) | React Context (`context/`) |
| Real-time push events | `@adonisjs/transmit` client |

---

## Docker Integration

N.O.M.A.D. manages third-party services (Kiwix, Kolibri, Qdrant, etc.) as Docker containers. The `DockerService` communicates with the local Docker daemon via **Dockerode** (a Node.js Docker API client that connects to `/var/run/docker.sock`).

Responsibilities include:

- **Installing** a service: pull the image and create the container with the correct volume mounts and environment
- **Starting / stopping** containers on demand
- **Updating** a service: pull a newer image, recreate the container
- **Health checking**: polling container state and reporting back to the UI

The management stack itself (the N.O.M.A.D. admin app, MySQL, Redis, Dozzle) is defined in `install/management_compose.yaml` and managed by `docker compose`.

---

## AI / RAG Pipeline

The AI chat feature uses **Retrieval-Augmented Generation (RAG)** to ground responses in user-uploaded documents:

```
User uploads a file
        │
        ▼
EmbedFileJob (BullMQ)
  1. Extract text (PDF → pdf-parse, images → Tesseract.js OCR, etc.)
  2. Chunk the text into overlapping passages
  3. Call Ollama's embedding endpoint for each chunk
  4. Upsert chunk vectors into Qdrant (collection per chat session)
        │
        ▼
User sends a chat message
        │
        ▼
RagService.query()
  1. Optionally rewrite the query for better retrieval
  2. Embed the query via Ollama
  3. Search Qdrant for the top-k nearest chunks
  4. Optionally rerank results
  5. Inject retrieved chunks into the system prompt
        │
        ▼
OllamaService.chat()
  Stream the model's response back to the browser via SSE
```

Model management (list, pull, delete) is handled by `OllamaService`, which wraps the Ollama JavaScript client.

---

## Offline Content System

Offline reference content is distributed as **ZIM files** (a compressed archive format used by Kiwix). The content lifecycle is:

1. The `collections/kiwix-categories.json` catalogue defines available content with download URLs and metadata.
2. The user selects content in the Easy Setup wizard or the ZIM library manager.
3. `DownloadService` enqueues a `RunDownloadJob` which streams the file to `NOMAD_STORAGE_PATH`.
4. `CollectionManifest` records the file's presence in the database.
5. On startup, `CollectionService` reconciles the database against the filesystem so records stay accurate even if files are moved or deleted manually.
6. `ZimService` uses **libzim** bindings to read ZIM article data and serve it through the content explorer.

Maps follow the same download-and-manifest pattern using **PMTiles** files served by ProtoMaps.

---

## Configuration Reference

### Environment Variables

The canonical list and types are validated in `admin/start/env.ts`. Key variables:

| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `8080` | HTTP server port |
| `HOST` | `0.0.0.0` | Bind address |
| `APP_KEY` | — | AdonisJS encryption / session key (required) |
| `NODE_ENV` | `development` | Environment mode |
| `DB_HOST` | `localhost` | MySQL host |
| `DB_PORT` | `3306` | MySQL port |
| `DB_USER` | — | MySQL username |
| `DB_PASSWORD` | — | MySQL password |
| `DB_DATABASE` | `nomad` | MySQL database name |
| `DB_SSL` | `false` | Enable MySQL TLS |
| `REDIS_HOST` | `localhost` | Redis host |
| `REDIS_PORT` | `6379` | Redis port |
| `NOMAD_STORAGE_PATH` | `/opt/project-nomad/storage` | Root path for all downloaded content |
| `LOG_LEVEL` | `info` | Pino log level |

### Config Files (`admin/config/`)

| File | What it configures |
|---|---|
| `app.ts` | App key, environment, trusted proxies |
| `database.ts` | Lucid / MySQL connection and pool settings |
| `queue.ts` | BullMQ / Redis connection per queue |
| `session.ts` | Session driver (cookie), expiry |
| `cors.ts` | Allowed origins and HTTP methods |
| `bodyparser.ts` | Request size limits |
| `hash.ts` | bcrypt / argon2 hashing settings |
| `logger.ts` | Pino transport and log level |
| `shield.ts` | CSRF protection, security headers |
| `inertia.ts` | Shared props for every Inertia page |
| `transmit.ts` | SSE channel configuration |
| `vite.ts` | Vite dev server and HMR settings |
| `static.ts` | Static file serving options |

---

## Development Setup

### Prerequisites

- Debian-based OS (Ubuntu 22.04+ recommended)
- Node.js 22 LTS
- Docker Engine (with the daemon running)
- MySQL 8 and Redis 7 accessible (or run them via Docker)

### Steps

```bash
# 1. Install dependencies
cd admin
npm install

# 2. Copy and configure the environment
cp .env.example .env
# Edit .env: set APP_KEY, DB_*, REDIS_*, NOMAD_STORAGE_PATH

# 3. Run database migrations
node ace migration:run

# 4. Start the development server (backend + Vite HMR)
npm run dev

# 5. (Optional) Start background job workers in a separate terminal
npm run work:all
```

The app is now reachable at `http://localhost:8080`.

### Useful Commands

```bash
npm run build        # Production build (compiles TS + bundles frontend)
npm start            # Run the production build
npm test             # Run the JAPA test suite
npm run lint         # ESLint
npm run format       # Prettier
npm run typecheck    # TypeScript compiler check (no emit)
node ace --help      # List all Ace CLI commands
```

### Project Conventions

- **Controllers are thin.** Route handlers should validate input and call a service — nothing more.
- **Services own business logic.** If it touches the database, Docker, or an external API, it belongs in a service.
- **Background work goes in jobs.** Anything that takes more than a few hundred milliseconds should be a BullMQ job.
- **TypeScript everywhere.** Both server and client code are fully typed; avoid `any`.
- **Conventional Commits.** All commit messages must follow the `<type>(<scope>): <description>` format (see [CONTRIBUTING.md](CONTRIBUTING.md)).
