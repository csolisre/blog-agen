# Architecture Document

> ⚠️ **Note on SECURITY.md:** `SECURITY.md` was drafted for a Laravel/PHP stack and has not yet been updated to reflect the Node.js/Express stack confirmed in this document. See [Suggested Next Steps](#suggested-next-steps).

## Architecture Inputs
This section records the inputs used to derive the architecture below. These are sourced from `REQUIREMENTS.md` and stakeholder decisions.

| Input | Value |
|-------|-------|
| Requirements source | `REQUIREMENTS.md` |
| System purpose | Blog platform — authenticated users create/publish posts with images; visitors read and browse |
| Primary use cases | Registration, login/logout, post CRUD, rich-text editing, image management, public browsing, search, category/tag filtering |
| Target actors | Content Creators (authenticated), Readers (anonymous) |
| Runtime environment | Web application (browser-based) |
| Server framework | Node.js / Express.js |
| Client framework | React (SPA) |
| API style | REST |
| Authentication model | JWT access tokens (short-lived) + refresh tokens in httpOnly cookies |
| Data model | Derived from `REQUIREMENTS.md` §7; normalized to 3NF |
| Deployment model | Azure (infrastructure-as-code via Bicep) |
| Geographic scope | Single-region, Belgium (Azure West Europe) |
| Security priority | High — GDPR data handling required |

## Architecture

### Overview
The system is a single-page application (SPA) with a React frontend communicating via REST API to a Node.js/Express backend. The backend handles business logic, authentication, and data persistence. Uploaded images are stored in Azure Blob Storage and delivered through Azure CDN. Static frontend assets are hosted on Azure Static Web Apps. The full infrastructure is defined as code using Azure Bicep.

### Decisions
These items have been confirmed and should not be changed without a documented decision:

| Decision | Value | Rationale |
|----------|-------|-----------|
| Server runtime | Node.js | Chosen for lightweight, async I/O |
| Server framework | Express.js | Minimal overhead, widely adopted |
| Client framework | React (SPA) | Component model suits content-heavy UI |
| API style | REST | Simpler than GraphQL for this use case |
| Database | PostgreSQL | Relational, fits 3NF schema, well-supported on Azure |
| Image processing | Sharp | Fast, battle-tested Node.js image library |
| Authentication | JWT (short-lived access) + httpOnly refresh cookie | Stateless API; refresh token stored securely |
| Deployment | Azure Bicep (IaC) | Reproducible, version-controlled infrastructure |
| Media storage | Azure Blob Storage + Azure CDN | Scalable object storage with global CDN delivery |
| Geographic scope | Belgium (Azure West Europe) | Single-tenant, single-region per scale requirements |
| GDPR | Data export + account deletion endpoints; cookie consent banner | Required by law; see `SECURITY.md` checklist |

### Open Questions
These items are not yet resolved and must be decided before implementation of the relevant layer:

| # | Question | Impact |
|---|----------|--------|
| 1 | ORM / query builder: Prisma or Knex? | Affects migration strategy, type safety, and query patterns |
| 2 | Frontend state management: React Context or a library (e.g. Zustand)? | Affects component architecture and testability |
| 3 | Frontend hosting: Azure Static Web Apps or storage account + CDN? | Affects CI/CD pipeline and routing configuration |
| 4 | Update `SECURITY.md` for Node.js/Express stack | Blocks security review; see [Suggested Next Steps](#suggested-next-steps) |

### Component Diagram (Logical)

```
┌─────────────────────────────────────────────────────────────┐
│  Browser                                                      │
│  ┌───────────────────────────────┐                           │
│  │  React SPA                    │                           │
│  │  (Azure Static Web Apps / CDN)│                           │
│  └──────────────┬────────────────┘                           │
└─────────────────│───────────────────────────────────────────-┘
                  │ HTTPS REST (JSON)
                  ▼
┌─────────────────────────────────────────────────────────────┐
│  Azure App Service                                            │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Node.js / Express API                                │  │
│  │  ┌──────────┐  ┌────────────┐  ┌──────────────────┐  │  │
│  │  │  Routes  │→ │  Services  │→ │  Repositories    │  │  │
│  │  └──────────┘  └────────────┘  └────────┬─────────┘  │  │
│  └───────────────────────────────────────── │ ───────────┘  │
└────────────────────────────────────────── │ ────────────────┘
                                             │
              ┌──────────────────────────────┤
              │                              │
              ▼                              ▼
┌─────────────────────┐         ┌────────────────────────────┐
│  Azure Database     │         │  Azure Blob Storage        │
│  for PostgreSQL     │         │  (uploaded media)          │
└─────────────────────┘         └──────────────┬─────────────┘
                                               │
                                               ▼
                                 ┌─────────────────────────┐
                                 │  Azure CDN              │
                                 │  (media + static assets)│
                                 └─────────────────────────┘
```

### Data Flow — Key Scenarios

#### Author publishes a post
1. React SPA sends `POST /api/posts` with JWT access token
2. Express validates token, runs authorization check (author/admin role)
3. Service layer sanitizes rich-text content, computes `reading_time`
4. Repository persists post to PostgreSQL (status: `published`)
5. API returns `201` with post data; SPA navigates to post view

#### Visitor reads a post
1. React SPA sends `GET /api/posts/:slug` (no auth required)
2. Express serves post from PostgreSQL; increments `view_count`
3. SPA renders content; media URLs point to Azure CDN

#### Author uploads an image
1. React SPA sends `POST /api/media` (multipart/form-data)
2. Express validates MIME type by content inspection, enforces 10 MB limit
3. Sharp resizes original and generates 300×200 px thumbnail
4. Both files uploaded to Azure Blob Storage
5. API returns URLs; SPA stores `url` and `thumbnail_url` in media record

### API Design

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/auth/register` | Public | Register new user |
| `POST` | `/api/auth/login` | Public | Returns JWT + sets refresh cookie |
| `POST` | `/api/auth/refresh` | Cookie | Issue new access token |
| `POST` | `/api/auth/logout` | JWT | Revoke refresh token |
| `GET` | `/api/posts` | Public | List published posts (paginated) |
| `GET` | `/api/posts/:slug` | Public | Get single post |
| `POST` | `/api/posts` | Author/Admin | Create post |
| `PUT` | `/api/posts/:id` | Author/Admin | Update post |
| `DELETE` | `/api/posts/:id` | Author/Admin | Soft-delete post |
| `GET` | `/api/categories` | Public | List categories |
| `GET` | `/api/tags` | Public | List tags |
| `POST` | `/api/media` | Author/Admin | Upload image |
| `GET` | `/api/media` | Author/Admin | List uploaded media |
| `POST` | `/api/posts/:id/comments` | Public | Submit comment (pending moderation) |
| `PUT` | `/api/comments/:id` | Admin | Approve / reject comment |
| `GET` | `/api/search?q=` | Public | Full-text search posts |
| `GET` | `/api/users/me/export` | JWT | GDPR data export |
| `DELETE` | `/api/users/me` | JWT | GDPR account deletion |

### Database
Schema is fully defined in `REQUIREMENTS.md` §7. Key notes:
- All tables use integer PKs with auto-increment
- Soft delete (`deleted_at`) applies to Posts only; all other tables use hard delete
- `reading_time` stored as integer (minutes); computed at write time at ~200 wpm
- Thumbnails auto-generated on upload (see §Media above)

### Deployment Architecture

```
┌──────────────────────── Azure Subscription ──────────────────────────┐
│                                                                        │
│  ┌──────────────────────── Resource Group ───────────────────────┐   │
│  │                                                                │   │
│  │  Azure Static Web Apps        Azure App Service (Node.js API) │   │
│  │  (React SPA)                  ┌──────────────────────────┐    │   │
│  │  ┌────────────────────┐       │  Express.js              │    │   │
│  │  │  Built React bundle│       │  + Environment variables │    │   │
│  │  └────────────────────┘       └────────────┬─────────────┘    │   │
│  │                                            │                  │   │
│  │  Azure CDN ◄── Azure Blob Storage          │                  │   │
│  │  (media delivery)   (raw uploads)          ▼                  │   │
│  │                              Azure Database for PostgreSQL    │   │
│  │                              (VNet-integrated, private access) │   │
│  └────────────────────────────────────────────────────────────── ┘   │
│                                                                        │
│  All infrastructure defined in Bicep (IaC), deployed via CI/CD        │
└────────────────────────────────────────────────────────────────────── ┘
```

- **Environments:** development, staging, production — fully isolated resource groups
- **CI/CD:** GitHub Actions — test → build → deploy to staging → manual approval → deploy to production
- **Secrets:** injected via Azure Key Vault references in App Service configuration; never in code

### Security Architecture
Security requirements and rules are defined in `SECURITY.md`.

> ⚠️ `SECURITY.md` was originally drafted for a Laravel stack. It must be updated to replace Laravel-specific references (Eloquent, Form Requests, Sanctum, Larastan, Artisan) with Node.js/Express equivalents (Zod/Joi validation, middleware-based auth, npm audit, etc.).

Key security decisions confirmed for this stack:
- JWT access tokens (15 min expiry) + httpOnly refresh cookies (7 day expiry)
- All passwords hashed with bcrypt (cost factor ≥ 12)
- Input validation via middleware (Zod or Joi) on all endpoints
- Rich-text content sanitized server-side before storage (e.g. DOMPurify or sanitize-html)
- HTTPS enforced at Azure Front Door / App Service level
- GDPR endpoints: `/api/users/me/export` and `DELETE /api/users/me`

## Suggested Next Steps
- [ ] Resolve Open Question #1: choose ORM (Prisma recommended for type-safety with TypeScript)
- [ ] Resolve Open Question #2: choose state management library
- [ ] Resolve Open Question #3: confirm frontend hosting model
- [ ] Update `SECURITY.md` to replace Laravel references with Node.js/Express equivalents
- [ ] Update `REQUIREMENTS.md` §6 Tech Stack `[TODO]` entries (done in this PR)

---

**Document Version**: 1.1
**Last Updated**: 2026-06-04
**Approved By**: [TODO: stakeholder input needed]
**Next Review Date**: [TODO: stakeholder input needed]