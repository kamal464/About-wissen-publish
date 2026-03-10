# Universal Publishers / Wissen Publication Group – Project Documentation

Overview of the **Wissen Publication Group** academic publishing platform: live URLs, tech stack, login types, features per role, and deployment details.

**Credits / Development**  
Full stack (frontend, backend, database, infrastructure, and integrations) was completely developed by **Kollimarla Kamalakar**. Deployment, end-to-end testing, and production setup were also developed and carried out by Kollimarla Kamalakar.

---

## 1. Website links

| Purpose | URL |
|--------|-----|
| **Production site** | https://wissenpublicationgroup.com |
| **Alternate / www** | https://www.wissenpublicationgroup.com |
| **Vercel preview** | https://wissen-publication-group.vercel.app |
| **Maintenance page** | https://wissenpublicationgroup.com/maintenance (when `NEXT_PUBLIC_MAINTENANCE_MODE=true`) |

### Backend API (production)

| Purpose | URL |
|--------|-----|
| **API base** | https://h27qf3a839.execute-api.us-east-1.amazonaws.com/dev |
| **API with path prefix** | https://h27qf3a839.execute-api.us-east-1.amazonaws.com/dev/api |
| **Health check** | https://h27qf3a839.execute-api.us-east-1.amazonaws.com/dev/health |

Frontend is configured to use the API base above (with `/api` where applicable) via `NEXT_PUBLIC_API_URL` in Vercel.

---

## 2. Tech stack

### Frontend
- **Framework:** Next.js 15 (App Router)
- **UI:** React 19, PrimeReact, Tailwind CSS, SASS
- **State:** Redux Toolkit
- **HTTP:** Axios
- **Hosting:** Vercel

### Backend
- **Runtime:** Node.js 20
- **Framework:** NestJS 11
- **ORM:** Prisma
- **Database:** PostgreSQL (AWS RDS)
- **Hosting:** AWS Lambda + API Gateway (Serverless Framework)
- **File storage:** AWS S3; public URLs via CloudFront
- **Email:** AWS SES (with SMTP fallback)

### Infrastructure
- **Database:** AWS RDS (PostgreSQL) – `universal_publishers`
- **API:** API Gateway REST API, single stage `dev`, throttling (e.g. 100 req/s, burst 200)
- **Files:** S3 bucket `wissen-publication-group-files`, CloudFront distribution for reads
- **Email:** AWS SES (verified identity), contact form emails

### DevOps / tooling
- **Deploy backend:** `scripts/deploy-via-wsl.sh` (WSL; sources `backend/prod.env`)
- **Deploy frontend:** Vercel (Git integration or `vercel deploy`)
- **DB migrations:** `npx prisma migrate deploy` (run during deploy or manually against RDS)
- **EC2 → RDS sync:** Dump from EC2, restore into RDS (see `docs/EC2_TO_RDS_SYNC.md`)

---

## 3. Logins and roles

There are **two** login entry points. Both use the same backend auth endpoint; the frontend distinguishes roles by where the user logs in and which token it stores.

### 3.1 Super Admin (full platform admin)

- **Login URL:** https://wissenpublicationgroup.com/admin/login  
- **Purpose:** Manage the entire platform: users, all journals, shortcodes, articles, submissions, news, web pages, settings, analytics.
- **Auth storage:** `localStorage.adminAuth`, `localStorage.adminUser`
- **Redirect after login:** `/admin/dashboard`
- **Typical username:** `admin` (or any super-admin user with no `journalShort` / managing all journals)

**Backend behavior:** User is looked up by `userName` (or by `journalShort` for journal-admin). If `userName === 'admin'` and no password is set, a default admin user can be bootstrapped. Token is issued and frontend treats it as super-admin when logged in via `/admin/login`.

### 3.2 Journal Admin (per-journal editor)

- **Login URL:** https://wissenpublicationgroup.com/journal-admin/login  
- **Purpose:** Manage a **single journal** (or set of journals) tied to the user’s shortcode: content, articles, editorial board, current issue, archive, guidelines, latest news, meta, etc.
- **Auth storage:** `localStorage.journalAdminAuth`, `localStorage.journalAdminUser`
- **Redirect after login:** `/journal-admin/dashboard`
- **Login with:** **Username** or **journal shortcode** (e.g. the shortcode that maps to the journal they manage)

**Backend behavior:** Same `/admin/login` API. User can be found by `userName` or by `journalShort`. If the user has a `journalShort`, they are effectively a journal admin; the frontend restricts them to journal-admin routes and scoped API usage.

---

## 4. Features by login / area

### 4.1 Public (no login)

- **Home:** List of journals, latest news, links to journals and articles.
- **Journals:** List and per-journal pages (`/journals`, `/journals/[shortcode]`).
- **Articles:** List and article detail (`/articles`, `/articles/[id]`).
- **Submit manuscript:** Form to submit a manuscript; success page.
- **Contact:** Contact form; submissions hit `POST /api/messages`, emails sent via SES.
- **News:** Public news listing.
- **About, Instructions, Editorial board, Search:** Informational and search pages.
- **Maintenance:** `/maintenance` shown when maintenance mode is enabled in Vercel.

### 4.2 Super Admin (`/admin/*`)

After logging in at `/admin/login`:

| Area | Path | Features |
|------|------|----------|
| Dashboard | `/admin/dashboard` | Overview, counts, quick links |
| Users | `/admin/users` | Create/edit users, assign journal shortcodes, set passwords |
| Journal shortcodes | `/admin/journal-shortcode` | Map shortcodes to journals and names |
| Journals | `/admin/journals` | CRUD journals, visibility, images (banner, flyer, etc.), uploads via presigned S3 |
| Journal sub-pages (admin) | `/admin/journals/aims-scope`, `archive`, `articles-press`, `current-issue`, `editorial-board`, `guidelines`, `home`, `meta` | Edit journal-specific content blocks |
| Board members | `/admin/board-members` | Manage editorial board members (tags, order, journals) |
| Articles | `/admin/articles`, `pending`, `review`, `search` | Manage all articles, status, review, search |
| Submissions | `/admin/submissions` | View/manage manuscript submissions |
| Pending fulltexts | `/admin/pending-fulltexts` | Articles needing full-text content |
| Latest news | `/admin/latest-news` | Manage news (pinned, published) |
| Web pages | `/admin/web-pages` | Static/custom pages (SEO, banners) |
| Analytics | `/admin/analytics` | Basic analytics / charts |
| Settings | `/admin/settings` | Platform-level settings |
| Contact messages | (API) | View contact form submissions and email status |

APIs used: all under `/api/admin/*` (users, journals, shortcodes, board-members, articles, news, web-pages, settings, contact-messages, etc.).

### 4.3 Journal Admin (`/journal-admin/*`)

After logging in at `/journal-admin/login` (with username or journal shortcode):

| Area | Path | Features |
|------|------|----------|
| Dashboard | `/journal-admin/dashboard` | Overview for their journal(s) |
| Journals | `/journal-admin/journals` | Edit their journal(s), images, meta |
| Journal content | `/journal-admin/journals/aims-scope`, `archive`, `archive-issue`, `articles-press`, `current-issue`, `editorial-board`, `guidelines`, `home`, `meta` | Edit content for their journal only |
| Articles | `/journal-admin/articles/search` | Search/manage articles for their journal |
| Latest news | `/journal-admin/latest-news` | Manage news (scoped as configured) |

Same backend auth; data filtered by user’s `journalShort` / journal association.

---

## 5. Main project characteristics

- **Multi-journal platform:** One site serves multiple journals; each journal has a shortcode, public URL, and optional journal-level admin.
- **Roles:** Super admin (full access) vs journal admin (per-journal). Single User table; `journalShort` (and shortcode mapping) determine scope.
- **Content:** Journals, articles, authors, editorial board, news, custom web pages. Manuscript submission and contact form with email (SES).
- **Assets:** Images/PDFs in S3; served via CloudFront; uploads via presigned URLs to avoid Lambda timeouts.
- **Production:** Frontend on Vercel; backend on AWS (Lambda + API Gateway + RDS + S3 + CloudFront + SES). Optional maintenance mode via env var.
- **Database:** PostgreSQL (RDS); schema and migrations via Prisma. EC2 dump/restore supported for syncing data into RDS (see `docs/EC2_TO_RDS_SYNC.md`).

---

## 6. Quick reference – important paths and env

- **Frontend env (Vercel):** `NEXT_PUBLIC_API_URL` = API base (e.g. `https://h27qf3a839.execute-api.us-east-1.amazonaws.com/dev`); optional `NEXT_PUBLIC_MAINTENANCE_MODE=true` for maintenance page.
- **Backend env:** `backend/prod.env` (DATABASE_URL, JWT_SECRET, S3, CloudFront, CORS_ORIGIN, SES/SMTP, etc.). Do not commit; used by `scripts/deploy-via-wsl.sh`.
- **API prefix:** Backend serves under `/api` (e.g. `/api/admin/login`, `/api/messages`, `/api/journals`).
- **Login endpoints:** Same backend login used for both admin UIs; frontend stores different keys and redirects to `/admin/*` or `/journal-admin/*` accordingly.

---

*Full stack developed, deployed, and tested end-to-end by Kollimarla Kamalakar. Last updated to reflect production URLs, single dev stage, and current feature set.*
