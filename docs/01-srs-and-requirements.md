# 01 — Software Requirements Specification (SRS) & MVP Scope

## 1. Purpose & Product Vision

**GamesLab** is a social gaming discovery and community web application inspired by Letterboxd. It brings together game logging, reviews, curated lists, social following, and community text channels in one platform.

## 2. Target User Personas

1. **Guest / Visitor:** Unauthenticated user who browses the game catalog, views game details, reads public reviews, inspects public lists, and previews public community descriptions.
2. **Registered Member:** Authenticated user who maintains a gaming profile, rates and reviews games, curates personal lists, follows other users, joins communities, and participates in community text channels.
3. **Community Owner:** Registered member who creates a community, manages its text channels, and performs MVP moderation strictly limited to deleting channels/messages or removing members within that community (no ban lists, mute timers, or multi-tier moderator roles in MVP).

## 3. Explicit Assumptions & Open Decisions

To avoid inventing hidden requirements, the following assumptions are documented for confirmation before implementation:

1. **Game Catalog Source (Open Decision):** GamesLab stores game metadata in a PostgreSQL `games` table so reviews, ratings, and lists have reliable relational foreign keys. For the 10-day MVP, we will either seed ~50–100 popular games via SQL/JSON or sync games on demand from an external API (e.g., RAWG or IGDB).
2. **Rating Scale (Open Decision):** Inspired by Letterboxd, ratings are modeled as half-star increments (`0.5` to `5.0`, stored as integers `1`–`10`) unless whole-star `1`–`5` is preferred.
3. **Authentication:** Handled by Supabase Auth (email/password and optional OAuth such as GitHub/Google).
4. **Community Visibility:** Communities are public-by-default for discovery in the MVP; joining a community is required to post in its text channels.
5. **Text Channels Scope:** Community text channels support plain-text messages in MVP (file/image attachments, reactions, threads, and voice/video channels are deferred).

---

## 4. Functional Requirements

| ID | Category | Requirement |
|---|---|---|
| **FR-1** | Auth & Profiles | Users can sign up, log in, log out, and reset passwords via Supabase Auth. |
| **FR-2** | Auth & Profiles | Each user has a profile (`username`, `display_name`, `avatar_url`, `bio`) that they can edit; public profiles show recent reviews, public lists, and follower/following counts. |
| **FR-3** | Game Discovery | Users can browse the game catalog and search/filter by title, genre, or release year. |
| **FR-4** | Game Discovery | Each game has a dedicated detail page displaying metadata, average rating, rating count, and user reviews. |
| **FR-5** | Ratings & Reviews | Authenticated users can rate a game and optionally write, edit, or delete a written review. |
| **FR-6** | Personal Lists | Authenticated users can create, edit, and delete personal gaming lists, add/remove games, and toggle public/private visibility. |
| **FR-7** | Social Graph | Authenticated users can follow and unfollow other users and view an activity feed from followed users. |
| **FR-8** | Communities | Authenticated users can create a community, browse/search communities, and join or leave communities. |
| **FR-9** | Text Channels | Community owners can create text channels within their community (e.g., `#general`, `#recommendations`). |
| **FR-10** | Text Channels | Community members can read and send text messages in a community's channels in near real time. |

---

## 5. Non-Functional Requirements

| ID | Quality Attribute | Target & Rationale |
|---|---|---|
| **NFR-1** | Security | Authentication delegated to Supabase Auth using `@supabase/ssr` browser-accessible session cookies (`SameSite=Lax`, `Secure` in production) paired with strict XSS prevention; server identity re-verified via `supabase.auth.getUser()`; every database table protected by least-privilege PostgreSQL Row Level Security (RLS); all server inputs validated with Zod schemas. |
| **NFR-2** | Performance | Public catalog, game, and profile pages render initial HTML on the server via React Server Components; foreign keys and search fields are indexed. |
| **NFR-3** | Usability & Accessibility | Responsive mobile-first UI using Tailwind CSS and shadcn/ui; semantic HTML, keyboard-accessible dialogs/forms, clear loading and error states. |
| **NFR-4** | Maintainability | TypeScript `strict` mode, modular folder separation (`app/`, `components/`, `services/`, `lib/`), direct ESLint (`eslint .` with flat config, replacing deprecated `next lint`) or Biome checks, and no duplicated business logic. |
| **NFR-5** | Scalability | Stateless Next.js application on Vercel; all list, review, game, and message queries paginated so payload sizes remain bounded. |
| **NFR-6** | Reliability | Graceful empty states, image fallbacks, and Next.js error boundaries (`error.tsx`) for resilient UX. |

---

## 6. MVP Scope & Feature Priorities (10-Day Challenge)

| Priority | Domain | Included in 10-Day MVP | Deferred (Post-MVP) |
|---|---|---|---|
| **P0 (Must-Have)** | Foundation & Auth | Next.js + Supabase setup, Auth flows, auto-created `profiles`, profile edit page | Additional OAuth providers, custom transactional email styling |
| **P0 (Must-Have)** | Game Discovery | Search/filter catalog via URL search params, game detail page, PostgreSQL `games` catalog | ML recommendations, complex multi-tag boolean filters, `pg_trgm` trigram search |
| **P0 (Must-Have)** | Ratings & Reviews | Create/edit/delete rating + review, aggregate average rating | Rich markdown editor, spoiler blur tags, review likes/comments |
| **P1 (Should-Have)** | Personal Lists | Create/edit/delete lists, add/remove games, public/private toggle | Drag-and-drop reordering (`position`), collaborative multi-user lists |
| **P1 (Should-Have)** | Social Graph | Follow/unfollow users, counts, "Following" activity feed | Real-time notifications bell, user blocking/muting |
| **P1 (Should-Have)** | Communities & Channels | Create/join communities, create text channels, Realtime chat messages, Owner-only delete/remove | Message threads, emoji reactions, file uploads, ban lists, granular mod roles |
| **P2 (Nice-to-Have)** | UI Polish | Loading skeletons, toast feedback, mobile drawer nav, dark mode | Direct Messages (DMs), Steam/PSN library import, audit logs |

### 10-Day Milestone Plan

- **Day 1:** Technical foundation, SRS, ERD, architecture, and security design.
- **Day 2:** Next.js 16, TypeScript, Tailwind, shadcn/ui scaffolding; GitHub & Vercel setup; hosted Supabase Dev project.
- **Day 3:** PostgreSQL schema migrations, least-privilege RLS policies, triggers, and seed data.
- **Day 4:** Authentication flows, session proxy (`src/proxy.ts`), and user profiles.
- **Day 5:** Game discovery catalog, search/filtering, and game detail pages.
- **Day 6:** Ratings and reviews CRUD + average rating aggregation.
- **Day 7:** Personal gaming lists and adding/removing games.
- **Day 8:** Follow/unfollow social graph and following activity feed.
- **Day 9:** Communities, text channels, and Supabase Realtime messaging.
- **Day 10:** End-to-end QA, negative RLS security testing, polish, and final production deployment.
