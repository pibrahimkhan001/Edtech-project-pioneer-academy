# Pioneer Academy Platform

**A production-grade, multi-tenant SaaS edtech platform** built for coaching institutes preparing students for India's competitive civil services examinations — UPSC, TSPSC Group 1/2/3, APPSC, and SI/Police.

[![React](https://img.shields.io/badge/React-18-61DAFB?style=flat-square&logo=react&logoColor=black)](https://react.dev)
[![Node.js](https://img.shields.io/badge/Node.js-20-339933?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org)
[![MongoDB](https://img.shields.io/badge/MongoDB-Atlas-47A248?style=flat-square&logo=mongodb&logoColor=white)](https://mongodb.com/atlas)
[![Vite](https://img.shields.io/badge/Vite-5-646CFF?style=flat-square&logo=vite&logoColor=white)](https://vitejs.dev)
[![Deployed on Vercel](https://img.shields.io/badge/Frontend-Vercel-000000?style=flat-square&logo=vercel)](https://vercel.com)
[![Backend on Railway](https://img.shields.io/badge/Backend-Railway-0B0D0E?style=flat-square&logo=railway)](https://railway.app)

---

## Problem Statement

India's competitive exam coaching market is fragmented. Institutes typically manage enrollments through WhatsApp groups, run tests on paper, collect fees in cash, and distribute content via Telegram — all separately, with no unified system.

Existing EdTech platforms (Unacademy, BYJU'S) are **student-facing consumer products**, not institute-management tools. A mid-sized coaching institute with 500–2,000 students has no affordable, brandable, self-hosted platform that gives them:

- Role-separated admin and staff workflows
- A real test engine with timer, scoring, and analytics
- Batch-level enrollment and fee collection
- A content publication system (current affairs, resources, magazines)
- Granular control over which features are active for their business model

**This platform solves that.** It is architected as a multi-tenant SaaS product that can be white-labeled and deployed per institute, with a top-level Power Admin who gates every feature per client via runtime feature flags.

---

## Solution Overview

Pioneer Academy Platform is a **full-stack coaching institute management system** covering:

- **Multi-role admin panel** — Super Admin, Content Writer, Test Admin, Faculty
- **Student portal** — dashboard, test-taking, results, resources, enrollment history
- **Test engine** — objective (MCQ) and descriptive tests with timer, auto-scoring, attempt tracking, and per-student analytics
- **Enrollment + payments** — batch applications, seat management, Razorpay integration (UPI, cards, net banking, QR), offline payment verification workflow
- **CMS** — current affairs, blog, study resources, monthly magazines, gallery, announcements, results, faculty profiles
- **Power Admin** — top-level SaaS control; enables/disables any feature per institute via `InstituteConfig.features` flags

---

## System Architecture

![Pioneer Academy Platform — System Architecture](./docs/architecture.png)

The platform follows a clean four-tier architecture:

| Tier | Technology | Responsibility |
|---|---|---|
| Client | Browser — mobile + desktop | React SPA delivered from Vercel CDN |
| Frontend | Vercel · React 18 · Vite | UI, routing, data-fetching, feature-aware rendering |
| Backend | Railway · Node.js · Express | REST API, business logic, auth, all data operations |
| Services | MongoDB Atlas · Cloudinary · Razorpay | Persistence, media delivery, payment processing |

All HTTP communication is secured with JWT access tokens. For large file uploads (PDFs >10MB), the backend generates a Cloudinary signed upload URL; the **browser uploads directly to Cloudinary** — bypassing the Railway proxy body-size limit — then sends only the resulting URL to the backend as JSON.

---

## Tech Stack

### Frontend

| Concern | Choice | Rationale |
|---|---|---|
| Framework | React 18 | Component model, concurrent rendering |
| Bundler | Vite 5 | Fast HMR, optimised tree-shaking |
| Data fetching | TanStack React Query v5 | Server state, background refetch, stale-while-revalidate |
| Forms | React Hook Form | Uncontrolled inputs — minimal re-renders |
| Routing | React Router v6 | Nested layouts, protected route patterns |
| Deployment | Vercel | Zero-config, edge CDN, instant rollbacks |

### Backend

| Concern | Choice | Rationale |
|---|---|---|
| Runtime | Node.js 20 | Event-driven, non-blocking I/O |
| Framework | Express.js | Minimal, composable, ecosystem breadth |
| ODM | Mongoose | Schema validation, pre/post hooks, population |
| Auth | JWT (HS256) | Stateless — horizontally scalable without shared session store |
| Storage | Cloudinary SDK + signed upload | Server-side signing, direct browser upload for large files |
| Payments | Razorpay SDK | UPI, cards, netbanking — webhook-ready |
| Deployment | Railway | Containerised, environment variable isolation |

### Database

| Concern | Detail |
|---|---|
| Engine | MongoDB Atlas (M0 free → M10 production) |
| Key indexes | `(student, batch)` unique · `(status, createdAt)` · `(batch, status)` |
| Validation | Mongoose schema-level + Express middleware |

---

## Feature Highlights

### Multi-Tenant Feature Gating (Power Admin)

Every feature is a boolean flag in `InstituteConfig.features`. The Power Admin dashboard has toggle groups per module — Courses, Test Series, Payments, Gallery, etc. When a flag is `false`, it is blocked at **both** the API middleware level and the frontend rendering layer:

```
Power Admin sets: InstituteConfig.features.payments = false

  Backend:  requirePaymentsEnabled middleware → 403 FEATURE_DISABLED
  Frontend: useFeatures().payments === false  → no payment UI renders
```

### Role-Based Access Control (RBAC)

Five roles with layered permissions: `power_admin → admin → content_writer → test_admin → student`.

The `requireRole()` middleware is applied at the **router level**, not the controller. Admin actions are never executable by student tokens regardless of frontend state.

### Test Engine

- **Objective tests** — MCQ with optional negative marking, per-question or per-test timer, auto-submit on timeout
- **Descriptive tests** — Long-answer submission, admin manual review and marks entry
- **Attempt tracking** — Per-student attempt count capped at `maxAttempts` (configurable per test series, `null` = unlimited)
- **Analytics** — Score, percentage, section-wise breakdown, historical attempt graph with personal best tracking

### Enrollment + Payments

Full lifecycle: Application → Payment → Approval → Confirmed Enrollment.

- **Seat management** — `seatsLeft` decremented only when payment reaches `paid` or `verified`; released on `failed`, `rejected`, or `cancelled`
- **Offline verification queue** — Admin sees all `offline_pending` payments with student reference notes, reviews, marks `verified` or `rejected` with an internal note
- **Auto-confirm** — `batch.autoConfirmOnPayment = true` approves enrollment automatically on gateway success
- **Reapply flow** — Rejected students can reapply; resets the existing application to `pending` without creating duplicate documents (unique index on `(student, batch)` is preserved)

### Content Management

Slug-based routing, SEO-aware, 12+ content types: Current Affairs, Blog, Study Resources (PDF/link), Monthly Magazines (direct Cloudinary upload), Gallery albums, Testimonials, Results/Toppers, Faculty profiles, Announcements ticker, FAQ, Syllabus, Enquiry forms.

---

## Technical Breakdown

### Authentication Flow

```
POST /auth/login
  → bcrypt.compare(password, stored_hash)
  → JWT signed with HS256, 7-day expiry
  → { accessToken, user: { id, role, name, registrationId } }

Every protected request:
  → Authorization: Bearer <token>
  → protect() middleware: jwt.verify() → req.user
  → requireRole('admin') checks req.user.role
  → AppError(403) if insufficient role
```

Power Admin uses a **separate JWT secret** and a separate login endpoint (`/power-admin/login`), completely isolated from the institute admin flow.

### API Design Patterns

Routes follow: `/api/v1/{resource}/{admin|public}/{action}`

- **Modular routers** — each domain (auth, batches, enrollment, payments, test-series, content) is an Express Router mounted in `app.js`
- **`asyncHandler` wrapper** — eliminates try/catch boilerplate; centralises error propagation
- **`sendSuccess` / `sendCreated` / `sendPaginated`** — consistent response envelope: `{ success, data, message, pagination }`
- **`AppError` class** — operational errors with HTTP status codes, caught by the global error handler middleware
- **Feature gate middleware** — `requirePaymentsEnabled`, `requireFeature('testSeries')` applied at router registration, not per-controller

### Test Engine Design

```
TestSeries → TestAttempt (one per student per attempt number)
  TestAttempt.answers[]  → { questionId, selectedOption, isCorrect, marks }
  TestAttempt.status     → 'in_progress' | 'submitted' | 'timed_out'
  TestAttempt.score      → accumulated per-answer, rounded to 2dp
  TestAttempt.percentage → (score / totalMarks) × 100

Scoring per question:
  correct   → +marksPerQuestion
  incorrect → -(marksPerQuestion × negativeMarkingFactor)
  skipped   → 0
```

Timer is enforced **server-side**: `startedAt + duration` is stored on the attempt. Submissions after the deadline are rejected and the attempt is marked `timed_out`.

### Payment Module Architecture

```
Initiation:   POST /payments/initiate/:applicationId
              → Razorpay.orders.create({ amount_paise, currency, receipt })
              → Payment { status: 'initiated', gatewayOrderId }

Verification: POST /payments/verify
              → HMAC-SHA256: verify(orderId|paymentId, RAZORPAY_KEY_SECRET)
              → Payment { status: 'paid', paidAt: now }
              → batch.autoConfirmOnPayment → application { status: 'approved' }
              → Batch.seatsLeft--

Offline:      Payment { status: 'offline_pending' }
              → Admin reviews → PUT /payments/admin/:id/verify
              → Payment { status: 'verified' }
              → application { status: 'approved' }
              → Batch.seatsLeft--
```

**Gateway abstraction**: `RAZORPAY_KEY_ID` + `RAZORPAY_KEY_SECRET` env vars activate the live gateway. Without them, the system falls back to offline-only mode — functional for fee collection without a payment gateway account.

---

## Deployment Architecture

```
Git push to GitHub
  ├── Vercel detects frontend/ changes → npm run build → CDN edge deploy
  └── Railway detects backend/ changes → npm install → node src/server.js

Runtime:
  Browser → Vercel CDN → serves React SPA (static)
  SPA → HTTPS → Railway backend (VITE_API_BASE_URL)
  Backend → MongoDB Atlas (Mongoose connection pool)
  Backend → Cloudinary SDK (server-side signed URL generation)
  Browser → Cloudinary API directly (large file upload, signed)
  Backend → Razorpay SDK (order creation, signature verification)
```

**Environment variable strategy:**

| Variable | Location | Reason |
|---|---|---|
| `MONGODB_URI`, `JWT_SECRET`, `CLOUDINARY_*`, `RAZORPAY_*`, `SMTP_*` | Railway dashboard | Backend secrets, never client-visible |
| `VITE_API_BASE_URL` | Vercel dashboard | Build-time baked into the SPA bundle |

---

## Performance & Optimisation

### Backend
- **Lean queries** — `.lean()` on read-only list endpoints skips Mongoose document hydration, reducing memory allocation by ~40% on high-volume queries
- **Selective projection** — List endpoints project only needed fields: `select('name email role createdAt')`, avoiding unnecessary data transfer
- **Compound indexes** — `(student, batch)` unique index enforces business constraints and enables O(log n) enrollment lookups; `(status, createdAt)` accelerates admin dashboard filters
- **Pagination** — All list endpoints use `skip/limit` with `countDocuments`, avoiding full collection scans

### Frontend
- **React Query caching** — `staleTime: 60_000` means repeated navigations serve cached data instantly, with background revalidation
- **Code splitting** — Vite automatically splits vendor chunks (React, Router, React Query) from app code; admin pages lazy-load only when accessed
- **Optimistic updates** — Mutations use `onMutate → setQueryData` for instant UI feedback; `onError` rolls back
- **Uncontrolled forms** — React Hook Form avoids re-rendering the form tree on every keystroke

### Media Pipeline
- **Direct browser upload** — PDFs upload browser → Cloudinary directly; Railway never receives the binary. Eliminates the 10MB proxy limit and removes server CPU overhead
- **Signed upload parameters** — Backend generates a signed payload with 60-second TTL. Cloudinary validates the signature, preventing unauthorised uploads to the account

---

## Scaling to 1 Lakh+ Users

The current architecture is single-instance but designed to scale horizontally with no code changes.

### Stateless API → Horizontal Scaling

JWTs carry all session state. No sticky sessions, no shared memory. Any Railway instance can serve any request:

```
Load balancer
  ├── API instance 1 (Node.js / Express)
  ├── API instance 2 (Node.js / Express)
  └── API instance N
        ↓ Mongoose connection pool (5–10 connections per instance)
  MongoDB Atlas (replica set)
```

Scale trigger: CPU > 70% sustained or p95 latency > 500ms on read endpoints.

### Database Scaling Path

| Stage | Approach |
|---|---|
| Current (< 50K users) | MongoDB Atlas M10, single primary |
| Growth (50K–500K) | Read replicas — analytics queries route to secondaries |
| At scale (500K+) | Shard by `instituteId` — keeps institute data co-located, minimises cross-shard queries |
| Archival | Completed attempts > 1 year → `attempts_archive` collection via nightly job |

### Caching Layer (Redis — roadmap)

```
Current: Every request hits MongoDB directly

Target:
  InstituteConfig (feature flags)     → TTL 5 min  (changes only on admin action)
  Published test question sets        → TTL 10 min (immutable once published)
  Course and batch listing            → TTL 2 min
  Leaderboard aggregations            → TTL 1 min
  Cache invalidation: redis.del(key) on admin save mutation
```

### CDN + Async Processing

- **Cloudinary CDN** already serves all media from edge nodes. Zero backend involvement after initial upload
- **Background jobs (BullMQ + Redis — roadmap)**: Enrollment approval emails, PDF receipt generation, batch analytics rollup. Moves async work out of the request cycle; API response times drop from ~800ms to ~120ms for operations that trigger emails

### Realistic Capacity Estimates

| Metric | Current (1 instance) | With 3 instances + Redis cache |
|---|---|---|
| Concurrent active users | ~200 | ~2,500 |
| API requests/sec | ~50 | ~600 |
| Test submissions/min | ~30 | ~350 |
| PDF uploads | Unlimited (direct Cloudinary) | Unlimited |
| DB documents (efficient) | ~10M | ~100M with read replicas |

---

## Revenue / Product Model

The platform is designed as a **SaaS tool for coaching institutes**, not a consumer product.

**Per-institute licensing model**: Power Admin deploys one backend, configures one MongoDB Atlas cluster. Each institute gets its own `InstituteConfig` document, subdomain, and admin credentials. Feature flags control what each institute tier pays for:

| Feature | Basic | Professional |
|---|---|---|
| Batches + Enrollment | ✅ | ✅ |
| Content / CMS | ✅ | ✅ |
| Test Series + Analytics | ❌ | ✅ |
| Payment Module | ❌ | ✅ |
| Analytics Panel | ❌ | ✅ |

**Batch fee monetisation**: When the Payment Module is enabled, institutes collect enrollment fees through the platform. Platform revenue can be structured as a transaction percentage or flat SaaS subscription — aligning platform growth with institute growth.

**Future expansion paths**: Coupon / discount system, GST invoicing, partial instalment payments, multi-institute analytics dashboard, white-label mobile app (React Native with shared component logic).

---

## Project Structure

```
pioneer-academy-platform/
├── frontend/                          # React 18 + Vite SPA
│   ├── index.html                     # Razorpay checkout.js loaded here
│   ├── vercel.json                    # SPA rewrite rules
│   └── src/
│       ├── api/                       # All API functions — centralised
│       ├── components/
│       │   ├── admin/                 # DataTable, AdminCard, Pagination, etc.
│       │   ├── common/                # Modal, Spinner, ErrorBoundary
│       │   ├── layout/                # AdminLayout, PublicLayout, sidebar nav
│       │   └── sections/              # Homepage sections
│       ├── contexts/                  # InstituteConfigContext, AuthContext
│       ├── hooks/                     # useFeatures(), useAuth(), useDocumentTitle()
│       ├── pages/
│       │   ├── admin/                 # AdminBatchesPage, AdminPaymentsPage, ...
│       │   ├── auth/                  # StudentDashboard, Login, Register
│       │   ├── powerAdmin/            # PowerAdminDashboard (feature toggle UI)
│       │   └── public/                # CoursesPage, TestSeriesPage, GalleryPage
│       └── utils/                     # formatDate, formatScore, getMediaUrl
│
└── backend/                           # Node.js + Express REST API
    └── src/
        ├── app.js                     # Express app — all routers mounted here
        ├── server.js                  # HTTP server entry point
        ├── config/                    # env.js, logger, taxonomy constants
        ├── middleware/                # auth.middleware.js, global error handler
        ├── models/
        │   ├── User.model.js
        │   ├── Batch.model.js          # + payment config fields
        │   ├── BatchApplication.model.js  # + payment ref, paymentStatus
        │   ├── Payment.model.js        # Full payment lifecycle
        │   ├── TestSeries.model.js     # + TestAttempt, maxAttempts
        │   ├── InstituteConfig.model.js  # Feature flags live here
        │   └── ContentModels.model.js  # Blog, Gallery, Result, FAQ, etc.
        ├── modules/
        │   ├── auth/                   # login, register, forgot-password
        │   ├── enrollment/             # apply, approve, reapply, seat logic
        │   ├── payment/                # initiate, verify, offline queue, admin
        │   ├── testseries/             # CRUD, attempt, submit, score
        │   ├── magazine/               # Signed Cloudinary upload + CRUD
        │   ├── content.routes.js       # Generic content router factory
        │   └── misc.routes.js          # Batches, students, home sections
        ├── services/                   # cloudinary.adapter.js, storage.service.js
        ├── scripts/                    # DB migration scripts
        └── utils/                      # asyncHandler, AppError, apiResponse
```

---

## Local Development

**Prerequisites:** Node.js 20+, MongoDB Atlas account, Cloudinary account, Razorpay test account (optional)

```bash
# Clone
git clone https://github.com/your-username/pioneer-academy-platform.git
cd pioneer-academy-platform

# Backend
cd backend && npm install
cp .env.example .env     # fill in MONGODB_URI, JWT_SECRET, CLOUDINARY_*

# Frontend
cd ../frontend && npm install
cp .env.example .env.local
# Set VITE_API_BASE_URL=http://localhost:5000/api/v1

# Start
cd backend  && node src/server.js     # port 5000
cd frontend && npm run dev            # port 5173
```

**Required backend env vars:**

```env
MONGODB_URI=mongodb+srv://...
JWT_SECRET=...
JWT_POWER_ADMIN_SECRET=...
CLOUDINARY_CLOUD_NAME=...
CLOUDINARY_API_KEY=...
CLOUDINARY_API_SECRET=...
RAZORPAY_KEY_ID=rzp_test_...     # optional — fallback mode without
RAZORPAY_KEY_SECRET=...
EMAIL_ENABLED=true                # optional — for forgot-password flow
SMTP_HOST=smtp.gmail.com
SMTP_USER=your@gmail.com
SMTP_PASS=gmail-app-password
FRONTEND_URL=https://your-site.vercel.app
```

---

## Engineering Decisions

**Why not Next.js?** Server-side rendering adds meaningful complexity for a B2B admin-heavy tool where SEO is secondary to interactivity and role-gated pages. Vite + React Query delivers excellent UX with none of the SSR operational overhead.

**Why Express over Fastify/Hono?** At this traffic level, the framework performance delta is irrelevant. Express's ecosystem (multer, mongoose middleware, passport) integrates cleanly, and the codebase remains legible.

**Why MongoDB over PostgreSQL?** Content types vary significantly across institutes — a flexible document model lets the CMS support new content types without schema migrations. Payments and enrollment follow strict schemas enforced at the Mongoose layer.

**Why no Redux?** Server state lives in React Query. UI state (modals, form steps) lives in component-local state. The remaining global state (auth, institute config) fits in two small contexts updated rarely. Redux would be accidental complexity.

**Why Cloudinary direct upload?** Railway's reverse proxy enforces a 10MB body limit at the infrastructure level — not configurable on lower tiers. Signed direct-upload from the browser bypasses this completely while keeping signature generation server-controlled (the client never touches API credentials).

---

## Roadmap

- [ ] Redis caching — InstituteConfig, published tests, leaderboard aggregations
- [ ] Razorpay webhook listener — async payment confirmation without polling
- [ ] PDF receipt generation — auto-email on payment confirmation
- [ ] Coupon / discount system — percentage or flat codes per batch
- [ ] Student analytics dashboard — score trends, rank tracking, weak-topic identification
- [ ] GST invoicing — compliant PDF with GSTIN and SAC code
- [ ] Custom domain per institute — Cloudflare wildcard SSL routing
- [ ] Multi-institute Power Admin dashboard — aggregate metrics across all clients

---

## License

MIT — free to use, fork, and adapt for your own institute or SaaS product.

---

*Every feature in this platform traces back to a real operational requirement observed in Tier-2 city coaching institutes: the offline payment queue exists because cash collection at the institute front desk is the primary method for most students. The reapply flow exists because admin rejections due to form errors are common. The direct Cloudinary upload exists because 40MB monthly PDF magazines are the actual content being distributed.*
