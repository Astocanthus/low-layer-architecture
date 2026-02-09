# LOW-LAYER Console Architecture

**Purpose**: Authenticated web console for infrastructure management.

**Language**: TypeScript

**Framework**: Angular 21 (standalone, flat structure)

**URL**: https://cloud.low-layer.com

```
low-layer-console/
├── src/
│   ├── main.ts                        # Entry point (MSW + Angular bootstrap)
│   └── app/
│       ├── app.ts                     # Root component
│       ├── app.config.ts              # Angular providers configuration
│       ├── app.routes.ts              # Top-level route definitions
│       │
│       ├── layout/                    # Shell layout (sidebar, header)
│       │   ├── layout.component.ts
│       │   ├── layout.component.html
│       │   └── layout.component.css
│       │
│       ├── components/                # App-level shared components
│       │   └── notifications/
│       │
│       ├── core/                      # Core services & infrastructure
│       │   ├── auth/                  # Auth providers (Keycloak OIDC)
│       │   ├── http/                  # HTTP provider setup
│       │   ├── services/              # ApiService, AuthService, NotificationService
│       │   ├── interceptors/          # auth, error, retry, logging
│       │   ├── data-access/           # Base repository pattern
│       │   └── utils/                 # Retry strategies
│       │
│       ├── features/                  # Feature modules (lazy-loaded)
│       │   ├── dashboard/
│       │   ├── organizations/         # Includes nested environment views
│       │   ├── catalog/
│       │   └── graph-editor/          # Infrastructure graph visualization
│       │
│       ├── shared/                    # Zod schemas, types, factories
│       │   ├── schemas/
│       │   ├── types/
│       │   └── factories/
│       │
│       └── mocks/                     # MSW mock server
│           ├── browser/
│           ├── handlers/
│           └── data/
│
├── public/
├── e2e/                               # Playwright E2E tests
├── angular.json                       # Angular CLI configuration
├── tsconfig.base.json
├── eslint.config.mjs
├── vitest.workspace.ts
└── package.json
```

**Feature Module Structure** (each feature follows the same pattern):
```
features/<feature>/
├── <feature>.routes.ts                # Feature route definitions
├── data-access/                       # Repository (signals + API calls)
│   └── <entity>.repository.ts
├── <view>/                            # Smart (container) components
│   ├── <view>.component.ts
│   ├── <view>.component.html
│   └── <view>.component.css
└── ui/                                # Dumb (presentational) components
    └── <entity>-card/
```

---

## Core Module Architecture

The `core/` module provides singleton services, authentication, HTTP infrastructure, and shared data access patterns.

### Authentication — Keycloak OIDC

| Setting | Value |
|---------|-------|
| Provider | Keycloak (`angular-auth-oidc-client`) |
| Flow | Authorization Code + PKCE (public client) |
| Scopes | `openid profile email offline_access` |
| Token Refresh | Silent renew with refresh tokens (30s before expiry) |
| Auto-Login | `withAppInitializerAuthCheck()` — login forced before app renders |

`AuthService` wraps `OidcSecurityService` with signal-based state:
- `isAuthenticated` — computed from OIDC library
- `currentUser` — extracted from `preferred_username` claim
- `getAccessToken()` — Observable for token relay
- `logout()` — Single Logout (SLO) to Keycloak end_session_endpoint

### HTTP Interceptor Chain

Requests flow through 4 interceptors in order:

```
Request → loggingInterceptor → authInterceptor → retryInterceptor → errorInterceptor → Response
```

| Interceptor | Purpose |
|------------|---------|
| `loggingInterceptor` | Logs all requests/responses (dev debugging) |
| `authInterceptor` | Injects `Authorization: Bearer <token>` from Keycloak |
| `retryInterceptor` | Exponential backoff retry (configurable per-request) |
| `errorInterceptor` | Catches errors, validates with `ApiErrorSchema`, shows toast notification |

### ApiService — Type-Safe HTTP Client

Generic HTTP methods with **Zod runtime validation** on every response:

```typescript
get<T>(endpoint, schema: ZodType<T>, options?): Observable<T>
getById<T>(endpoint, id, schema): Observable<T>
getPaginated<T>(endpoint, itemSchema, pagination?): Observable<PaginatedResponse<T>>
post<T, D>(endpoint, data, schema): Observable<T>
put<T, D>(endpoint, id, data, schema): Observable<T>
patch<T, D>(endpoint, id, data, schema): Observable<T>
delete(endpoint, id): Observable<void>
```

Every response is validated against its Zod schema at runtime. Validation failures throw immediately, caught by `errorInterceptor`.

### BaseRepository — Signal-Based CRUD

Abstract class eliminating CRUD boilerplate across features:

**Signals**: `items`, `current`, `loading`, `error`, `count` (computed)

**Methods**: `loadAll()`, `loadById()`, `create()`, `update()`, `remove()`, `clearCurrent()`

Subclasses only define: `endpoint`, `schema`, `getId()`.

---

## Shared Module — Zod Schemas

All API contracts are defined as **Zod schemas** in `shared/schemas/`. TypeScript types are inferred from schemas (`z.infer<typeof Schema>`), ensuring a single source of truth.

| Schema | Key Fields |
|--------|------------|
| `BaseEntitySchema` | id, createdAt, updatedAt |
| `OrganizationSchema` | name, slug, mode (shared\|dedicated), status, region, settings |
| `EnvironmentSchema` | name, organizationId, status |
| `NodeSchema` | name, environmentId, role, status, flavorId, metrics, labels, taints |
| `CatalogSchema` | name, description, category, version, icon |
| `GraphSchema` | **3-layer model**: spec (nodes, connections, subGraphs) + layout (positions, viewport) + status (phase, nodeStatuses) |

**Pagination primitives**: `PaginationRequestSchema`, `PaginationMetaSchema`, `createPaginatedResponseSchema(T)`

**API response wrappers**: `ApiErrorSchema` (code, message, details, timestamp, requestId), `ApiSuccessSchema<T>`

---

## Features Overview

### Dashboard

Summary metrics page: org count, env count, node count, active deployments. Quick action buttons link to New Org, New Env, Browse Catalog, Graph Editor.

### Organizations

3-level hierarchy with nested routes:

```
/organizations                                    → OrganizationListComponent (grid)
/organizations/:id                                → OrganizationDetailComponent (org + environments)
/organizations/:orgId/environments/:envId         → EnvironmentDetailComponent (env + nodes)
```

Repositories: `OrganizationRepository`, `EnvironmentRepository` (both extend `BaseRepository`).

### Catalog

Service catalog with search + category filtering. `CatalogRepository` uses direct array response (not paginated). Grid of `CatalogCardComponent` items.

---

## Development Mock System (MSW)

In development (`enableMocks: true`), all API calls are intercepted by **Mock Service Worker** (MSW 2.x):

| Handler | Endpoint | CRUD |
|---------|----------|------|
| `organizations.handlers.ts` | `/api/organizations` | Full |
| `environments.handlers.ts` | `/api/organizations/:orgId/environments` | Full |
| `graphs.handlers.ts` | `/api/environments/:envId/graph` | GET/POST |
| `org-graphs.handlers.ts` | `/api/organizations/:orgId/bastion-graph` | GET/POST |
| `subgraphs.handlers.ts` | `/api/organizations/:orgId/bastion-graph/subgraphs/:hubId` | GET/POST |
| `catalog.handlers.ts` | `/api/catalog` | GET |
| `nodes.handlers.ts` | `/api/environments/:envId/nodes` | GET |

**Seed data**: Demo organization with 5 environments, 13-node graph, service catalog. Stored in-memory with localStorage persistence for session survival.

**Simulated delay**: 150-300ms per handler to mimic real API latency.

---

## Environment Configuration

| Config | Dev | Prod |
|--------|-----|------|
| `enableMocks` | `true` (MSW intercepts all API calls) | `false` (real API) |
| `keycloak.authority` | `auth.low-layer.com/realms/low-layer-dev` | Production realm |
| `keycloak.clientId` | `low-layer-console` | `low-layer-console` |
| `cdnUrl` | `http://localhost:9090` | CDN endpoint |

---

## TypeScript Path Aliases

| Alias | Target | Purpose |
|-------|--------|---------|
| `@console/models` | `src/app/shared/index.ts` | Zod schemas, types, factories |
| `@console/http` | `src/app/core/index.ts` | ApiService, AuthService, repositories |
| `@console/mocks` | `src/app/mocks/index.ts` | MSW mock utilities |

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Framework | Angular 21.1 (standalone components, no NgModules) |
| Build | Angular CLI + Vite 7.0 backend |
| Language | TypeScript 5.9 (strict mode) |
| Styling | CSS (component-scoped, vanilla flexbox/grid) |
| State | Angular Signals + RxJS 7.8 (HTTP only) |
| Validation | Zod 4.3 (runtime schema validation, type inference) |
| Graph Editor | Rete.js v2.0.6 |
| Graph Rendering | rete-angular-plugin v2.6.0 |
| Graph Layout | ELK (elkjs 0.8.2) via rete-auto-arrange-plugin v2.0.2 |
| Mocking | MSW 2.12 (Mock Service Worker) |
| Unit Testing | Vitest 4.0 + @analogjs/vitest-angular |
| E2E Testing | Playwright 1.36 |
| Auth | Keycloak OIDC (angular-auth-oidc-client 21.0) |
| SSR | Angular SSR + Express 4.21 |
| Linting | ESLint 9.8 + angular-eslint + typescript-eslint |
| Formatting | Prettier 3.6 |

---

See [Graph Editor Architecture](graph-editor.md) for the visual topology editor.
