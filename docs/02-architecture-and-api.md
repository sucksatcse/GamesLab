# 02 — System Architecture, Folder Structure, API Strategy & Security

## 1. System Architecture

GamesLab follows a **Modular Monolith** architecture powered by **Next.js App Router** (hosted on Vercel) and **Supabase** (PostgreSQL, Auth, Realtime).

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
|         NEXT.JS APP ROUTER (Vercel)         |       |
|  1. Middleware: refreshes Supabase session   |       |
|  2. Server Components: read queries (SSR)   |       |
|  3. Server Actions: Zod-validated mutations |       |
|  4. Route Handlers (/api/*): JSON endpoints |       |
+-------------------+-------------------------+       |
                    | HTTPS / Postgres Wire           |
                    v                                 v
+-------------------------------------------------------------------------+
|                         SUPABASE PLATFORM                               |
|  +------------------+   +--------------------+   +-------------------+  |
|  |  Supabase Auth   |   |  PostgreSQL + RLS  |   | Supabase Realtime |  |
|  | (Users & JWTs)   |   | (App Data & Rules) |   | (Channel Pub/Sub) |  |
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
│   ├── migrations/                # Versioned SQL schema & RLS policies
│   └── seed.sql                   # Initial game catalog seed for dev/testing
├── src/
│   ├── app/                       # Next.js App Router (routes & layouts only)
│   │   ├── (auth)/                # Route group: /login, /signup
│   │   ├── (main)/                # Route group: shared navbar/footer layout
│   │   │   ├── page.tsx           # Home / Discovery & Following Feed
│   │   │   ├── games/
│   │   │   │   ├── page.tsx       # /games (search & filter catalog)
│   │   │   │   └── [slug]/page.tsx# /games/[slug] (game details & reviews)
│   │   │   ├── lists/
│   │   │   │   ├── page.tsx       # /lists
│   │   │   │   └── [id]/page.tsx  # /lists/[id]
│   │   │   ├── u/[username]/
│   │   │   │   └── page.tsx       # /u/[username] (user gaming profile)
│   │   │   ├── communities/
│   │   │   │   ├── page.tsx       # /communities (directory)
│   │   │   │       └── [slug]/
│   │   │   │           ├── page.tsx           # Community overview / default channel
│   │   │   │           └── [channelId]/page.tsx # Specific text channel
│   │   │   └── settings/page.tsx  # Profile settings
│   │   ├── api/                   # Route Handlers (JSON endpoints when needed)
│   │   │   ├── health/route.ts    # Health check endpoint
│   │   │   └── games/search/route.ts # Debounced search / external API proxy
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
│   │   │   ├── client.ts          # Browser Supabase client
│   │   │   ├── server.ts          # Server Component / Action Supabase client
│   │   │   └── middleware.ts      # Session refresh helper
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
│   └── middleware.ts              # Root Next.js middleware for auth cookies
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
| **Server Components (RSC)** | Reading games, game details, reviews, profiles, lists, initial channel messages | Executes directly on the server with no extra HTTP `/api/*` hop and keeps DB logic off the client bundle. |
| **Server Actions** | Creating/editing/deleting reviews, lists, list items, follows, communities, channels, and profile updates | Strongly typed mutations invoked from forms/components, paired with `revalidatePath()` to refresh server caches cleanly. |
| **Route Handlers (`app/api/*`)** | Client-side debounced game search (`/api/games/search`), health checks, or proxying external game APIs | Standard HTTP JSON interface when a Client Component needs dynamic fetching or secret third-party API keys must stay server-side. |
| **Supabase Realtime** | Live message stream inside a community text channel | Pushes newly inserted `messages` rows over WebSocket to active channel viewers without polling. |

### Standard Mutation Response Envelope
All Server Actions and Route Handlers return a predictable, typed result:
- **Success:** `{ ok: true, data: T }`
- **Failure:** `{ ok: false, error: { code: string, message: string, fieldErrors?: Record<string, string[]> } }`

---

## 4. Security Considerations

1. **HTTP-Only Cookie Sessions (`@supabase/ssr`):** Auth tokens are stored in HTTP-only cookies refreshed by Next.js middleware rather than `localStorage`, mitigating XSS token theft.
2. **Server Identity Verification:** Always authenticate the caller on the server with `supabase.auth.getUser()` (never trust a `user_id` sent in a request body, and avoid relying solely on `getSession()` on the server).
3. **Defense-in-Depth with PostgreSQL RLS:** Every table in the `public` schema must have RLS enabled and explicit policies for `SELECT`, `INSERT`, `UPDATE`, and `DELETE`.
   - *Example (`lists`):* `SELECT` allowed when `is_public = true OR auth.uid() = user_id`; writes allowed only when `auth.uid() = user_id`.
   - *Example (`messages`):* `SELECT` and `INSERT` allowed only when the user is a member of the channel's parent community in `community_members`.
4. **Input Validation & XSS Prevention:** Validate all payloads on the server with Zod (field lengths, rating bounds, UUID format). Render user text normally in React (never `dangerouslySetInnerHTML`).
5. **Secrets Hygiene:** Only `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` may use the `NEXT_PUBLIC_` prefix. `SUPABASE_SERVICE_ROLE_KEY` and any third-party game API keys must stay strictly server-side and out of Git.
6. **Bounded Queries & Abuse Prevention:** Enforce pagination limits (e.g., max 50 rows per query) and text length limits on reviews and channel messages.

---

## 5. Development & Deployment Strategy

1. **Git Workflow:**
   - Initialize the first commit on `main` (migrating from the empty `master` branch) and connect a GitHub remote.
   - Work in short-lived feature branches per milestone (e.g., `feat/day2-scaffold`, `feat/day3-db-schema`) and merge via Pull Requests.
   - Follow Conventional Commits (`docs:`, `feat:`, `fix:`, `chore:`).
2. **Environment Separation:**
   - **Development:** `.env.local` connected to a dedicated `GamesLab-Dev` Supabase project (or local Supabase CLI).
   - **Production:** Vercel environment variables connected to a separate `GamesLab-Prod` Supabase project.
3. **CI/CD (GitHub + Vercel):**
   - Every Pull Request generates a Vercel **Preview Deployment** and runs `tsc --noEmit` + `next lint`.
   - Merging to `main` triggers the **Production Deployment**.
4. **Versioned Migrations:**
   - Every schema change and RLS policy is saved in `supabase/migrations/` as SQL rather than untracked dashboard edits.
