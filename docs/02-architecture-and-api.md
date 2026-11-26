# 02 — System Architecture, Folder Structure, API Strategy & Security

## 1. System Architecture

GamesLab follows a **Modular Monolith** architecture powered by **Next.js 16 App Router** (hosted on Vercel) and **Supabase** (PostgreSQL, Auth, Realtime).

```text
+-------------------------------------------------------------------------+
|                              USER BROWSER                               |
|  - Receives server-rendered HTML from React Server Components           |
|  - Hydrates interactive Client Components (forms, modals, chat input)   |
|  - Connects via WebSocket (WSS) to Supabase Realtime for live chat      |
+-------------------+---------------------------------+-------------------+
                    | HTTPS (Cookies / RSC / Actions) | WebSocket (WSS)
                    v                                 |
+---------------------------------------------+       |
|       NEXT.JS 16 APP ROUTER (Vercel)        |       |
|  1. Proxy (src/proxy.ts): refreshes session |       |
|     cookie & handles UX redirects ONLY      |       |
|  2. Server Components: read queries (SSR)   |       |
|  3. Server Actions: auth + Zod mutations    |       |
|  4. Route Handlers (/api/*): JSON endpoints |       |
+-------------------+-------------------------+       |
                    | HTTPS / Postgres Wire           |
                    v                                 v
+-------------------------------------------------------------------------+
|                         SUPABASE PLATFORM                               |
|  +------------------+   +--------------------+   +-------------------+  |
|  |  Supabase Auth   |   |  PostgreSQL + RLS  |   | Supabase Realtime |  |
|  | (Users & JWTs)   |   | (Least-Privilege)  |   | (Channel Pub/Sub) |  |
|  +------------------+   +--------------------+   +-------------------+  |
+-------------------------------------------------------------------------+
```

### Why This Architecture?
- **No unnecessary backend boilerplate:** Next.js handles UI rendering and server-side mutations in one typed codebase, while Supabase provides managed PostgreSQL, authentication, and WebSockets.
- **SEO & Fast Initial Load:** Public pages (`/games/[slug]`, `/u/[username]`, `/lists/[id]`) render on the server so search engines and social share previews work out of the box.
- **Clear Internal Boundaries:** Even inside a single repository, database queries (`src/services/`), input schemas (`src/lib/validations/`), UI components (`src/components/`), and route files (`src/app/`) remain strictly separated.

---

## 2. Proposed Project Folder Structure

```text
gameslab/
├── docs/                          # Architecture, SRS, ERD, and decision notes
├── supabase/
│   ├── migrations/                # Versioned SQL schema, triggers & RLS policies
│   └── seed.sql                   # Initial game catalog seed for dev/testing
├── src/
│   ├── app/                       # Next.js App Router (routes & layouts only)
│   │   ├── (auth)/                # Route group: /login, /signup
│   │   ├── (main)/                # Route group: shared navbar/footer layout
│   │   │   ├── page.tsx           # Home / Discovery & Following Feed
│   │   │   ├── games/
│   │   │   │   ├── page.tsx       # /games (URL searchParams filter catalog)
│   │   │   │   └── [slug]/page.tsx# /games/[slug] (game details & reviews)
│   │   │   ├── lists/
│   │   │   │   ├── page.tsx       # /lists
│   │   │   │   └── [id]/page.tsx  # /lists/[id]
│   │   │   ├── u/[username]/
│   │   │   │   └── page.tsx       # /u/[username] (user gaming profile)
│   │   │   ├── communities/
│   │   │   │   ├── page.tsx       # /communities (directory)
│   │   │   │   └── [slug]/
│   │   │   │       ├── page.tsx           # Community overview / default channel
│   │   │   │       └── [channelId]/page.tsx # Specific text channel
│   │   │   └── settings/page.tsx  # Profile settings
│   │   ├── api/                   # Route Handlers (minimal JSON endpoints)
│   │   │   └── health/route.ts    # Health check endpoint (add search route only if proxying external API)
│   │   ├── layout.tsx             # Root layout (fonts, theme, toast provider)
│   │   └── globals.css            # Tailwind CSS entry
│   ├── components/
│   │   ├── ui/                    # shadcn/ui primitives
│   │   ├── layout/                # Navbar, Footer, MobileNav
│   │   ├── games/                 # GameCard, RatingStars, GameSearchBar
│   │   ├── reviews/               # ReviewForm, ReviewList
│   │   ├── lists/                 # ListForm, AddToListDialog
│   │   └── communities/           # ChannelSidebar, ChatWindow, MessageInput
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── client.ts          # Browser Supabase client (createBrowserClient)
│   │   │   ├── server.ts          # Server Component / Action client (createServerClient)
│   │   │   └── session.ts         # Session refresh helper (updateSession)
│   │   ├── validations/           # Zod schemas
│   │   └── utils.ts               # Shared helpers
│   ├── services/                  # Server-side data access functions
│   │   ├── games.ts
│   │   ├── reviews.ts
│   │   ├── lists.ts
│   │   ├── profiles.ts
│   │   └── communities.ts
│   ├── types/
│   │   └── database.ts            # Generated Supabase TypeScript types
│   └── proxy.ts                   # Next.js 16 proxy boundary (exports proxy())
├── eslint.config.mjs              # Direct ESLint flat config (replaces deprecated next lint)
├── .env.example                   # Template of required environment variables
├── .env.local                     # Local secrets (gitignored)
├── package.json
└── tsconfig.json
```

---

## 3. API Design Strategy

GamesLab uses the right Next.js mechanism for each communication pattern:

| Mechanism | GamesLab Use Cases | Rationale |
|---|---|---|
| **Server Components (RSC)** | Reading games (`/games?q=...`), game details, reviews, profiles, lists, initial channel messages | Executes directly on the server with no extra HTTP `/api/*` hop and keeps DB logic off the client bundle. Using URL search params avoids building a duplicate `/api/games/search` endpoint for MVP. |
| **Server Actions** | Creating/editing/deleting reviews, lists, list items, follows, communities, channels, messages, and profile updates | Strongly typed mutations invoked from forms/components, paired with `revalidatePath()` to refresh server caches cleanly. |
| **Route Handlers (`app/api/*`)** | `/api/health` (and external game API proxy only if an external catalog API with secret keys is chosen) | Standard HTTP JSON interface when external webhooks or server-side third-party API proxies are required. |
| **Supabase Realtime** | Live message stream inside a community text channel | Pushes newly inserted `messages` rows over WebSocket to active channel viewers without polling. |

### Standard Mutation Response Envelope
All Server Actions and Route Handlers return a predictable, typed result:
- **Success:** `{ ok: true, data: T }`
- **Failure:** `{ ok: false, error: { code: string, message: string, fieldErrors?: Record<string, string[]> } }`

---

## 4. Security Considerations

1. **Supabase SSR Cookie Architecture (`@supabase/ssr`):**
   - `@supabase/ssr` does **not** use `httpOnly` cookies when paired with `createBrowserClient`. Auth tokens are stored in **browser-accessible, chunked cookies** (`SameSite=Lax`, `Path=/`, and `Secure` over HTTPS in production).
   - **Why:** Both the Next.js server (`createServerClient` / `src/proxy.ts`) and browser client (`createBrowserClient`, needed for client-side auth state and authenticated **Supabase Realtime** WebSocket subscriptions) must read and synchronize the session. Setting `httpOnly: true` would hide the cookie from `document.cookie` and break `createBrowserClient`.
   - **Required Trade-off Mitigation:** Because cookies are readable by client-side scripts, **XSS prevention is critical**: render all user-generated content safely through React escaping, never use `dangerouslySetInnerHTML`, validate/bound inputs with Zod, and review third-party packages carefully. *(Note: True `httpOnly`-only sessions would require removing `createBrowserClient` and proxying all auth and Realtime traffic through custom server endpoints, which is unnecessary complexity for this 10-day MVP.)*

2. **Three-Layer Authorization Model (`src/proxy.ts` Is NOT Authorization):**
   - **Layer 1 — Network Boundary (`src/proxy.ts`):** Refreshes the Supabase auth cookie and handles optimistic UX redirects (e.g., bouncing unauthenticated visitors from `/settings` to `/login`). **`proxy.ts` is never treated as a security boundary**—it cannot check object ownership, and Server Actions or Route Handlers can be called directly.
   - **Layer 2 — Server Application Logic:** Every Server Component, Server Action, and Route Handler verifies identity independently via `supabase.auth.getUser()` (never trust a `user_id` from client form input, and never rely solely on `getSession()` on the server) and checks business permissions before mutating data.
   - **Layer 3 — Database Enforcement (PostgreSQL RLS):** Enforces row-level ownership and community membership inside PostgreSQL itself.

3. **Least-Privilege PostgreSQL RLS (Deny-by-Default):**
   - Enable RLS on every table in the `public` schema.
   - Do **not** grant all four CRUD permissions (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) on every table. In PostgreSQL RLS, any operation without an explicit policy is **denied by default**. Grant only the minimum operations required per table (see the full matrix in [03-database-erd.md](./03-database-erd.md#4-least-privilege-rls-policy-matrix)).
   - *Examples:* `games` allows public `SELECT` only (zero client write policies); `follows` allows `SELECT`, `INSERT`, `DELETE` (no `UPDATE`); `profiles` allows `SELECT` and owner `UPDATE` (created via DB trigger, not client `INSERT`).

4. **Input Validation & XSS Prevention:**
   - Validate all Server Action and Route Handler payloads with Zod (string lengths, rating bounds `1..10`, UUID format, trimmed non-empty strings).

5. **Secrets Hygiene:**
   - Only `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` may use the `NEXT_PUBLIC_` prefix. `SUPABASE_SERVICE_ROLE_KEY` (which bypasses RLS) and any external game API keys must stay strictly server-side and out of Git.

6. **Bounded Queries & Abuse Prevention:**
   - Enforce pagination limits (e.g., max 50 rows per query) and text length limits on reviews and channel messages.

---

## 5. Development & Deployment Strategy

1. **Git Workflow:**
   - Initialize the first commit on `main` (migrating from the empty `master` branch) and connect a GitHub remote.
   - Work in short-lived feature branches per milestone (e.g., `feat/day2-scaffold`, `feat/day3-db-schema`) and merge via Pull Requests.
   - Follow Conventional Commits (`docs:`, `feat:`, `fix:`, `chore:`).
2. **Environment Separation (MVP-Friendly):**
   - **Development:** `.env.local` connected to a dedicated hosted `GamesLab-Dev` Supabase project (keeping local Docker Supabase CLI optional so Windows container setup doesn't stall a 10-day sprint).
   - **Production:** Vercel environment variables connected to a separate hosted `GamesLab-Prod` Supabase project.
3. **CI/CD (GitHub + Vercel):**
   - Next.js 16 **removed `next lint`**. Instead, run direct static checks in `package.json` and PR verification:
     - `npx tsc --noEmit` (strict TypeScript check)
     - `npx eslint .` using ESLint flat config (`eslint.config.mjs` with `eslint-config-next`) — *or optionally **Biome** (`npx biome check .`) as a faster single-tool linter/formatter alternative.*
   - Every Pull Request generates a Vercel **Preview Deployment**; merging to `main` triggers the **Production Deployment**.
4. **Versioned Migrations:**
   - Every schema change, trigger, and RLS policy is saved in `supabase/migrations/` as SQL rather than untracked dashboard edits.
