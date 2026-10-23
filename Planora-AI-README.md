# Planora AI — Intelligent Event Planning Assistant

Planora AI is a full-stack event-planning application that helps people organize weddings, birthdays, and community events with AI-generated plans, budgeting guidance, checklists, and a persistent planning workspace.

> **Project status:** Developed locally with Next.js, Gemini, Supabase, and TypeScript. A public demo URL can be added after deployment.

## Features

- **AI-assisted event planning:** Generate personalized event concepts, budget breakdowns, planning timelines, vendor checklists, and ideas with Google's Gemini API.
- **Event workspace:** Create and manage weddings, birthdays, and community events with dates, locations, guest counts, and budgets.
- **Budget guidance:** View an event's total planned budget and suggested cost allocations. Generated figures are estimates and should be checked against real vendor quotes.
- **Task checklist:** Track preparation tasks and completion status.
- **Authentication:** Email/password sign-up and sign-in with Supabase Auth, including optional email confirmation and profile name metadata.
- **Cloud persistence:** Store user-owned events, tasks, and generated plans in Supabase Postgres. Row Level Security (RLS) policies restrict access by authenticated user and event ownership.
- **Linked AI plans:** Associate generated plans with dashboard events and reopen saved plans.
- **Light/dark theme:** Switch between themes with a browser-persisted preference.
- **PDF export:** Print the generated AI plan and use the browser's **Save as PDF** destination.
- **Local cache and migration:** Preserve browser-cached events and plans while importing eligible records into the signed-in user's cloud workspace.

## Tech stack

| Layer | Technology |
|---|---|
| Framework | Next.js App Router (Next.js 16) |
| Language | TypeScript |
| UI | React, Tailwind CSS, Lucide React |
| Generative AI | Google Gemini API via `@google/genai` |
| Authentication | Supabase Auth |
| Database | Supabase / PostgreSQL |
| Session integration | `@supabase/ssr`, `@supabase/supabase-js` |
| Markdown rendering | `react-markdown`, `remark-gfm` |
| Quality checks | TypeScript compiler (`tsc --noEmit`) |

## Architecture overview

```text
Browser
  └── Next.js App Router
      ├── Dashboard: events, budgets and task progress
      ├── AI Planner UI
      │   └── POST /api/plan
      │       └── Gemini API (server-side key)
      ├── Authentication routes and session proxy
      └── Supabase browser/server clients
          ├── Supabase Auth
          └── PostgreSQL tables with RLS
              ├── events
              ├── event_tasks
              └── ai_plans
```

The Gemini API key must remain server-side. The browser uses only the Supabase project URL and publishable key; row access is enforced with RLS policies. Never ship a Supabase `service_role` key to the client.

## Getting started

### Prerequisites

- Node.js compatible with the project's installed Next.js version
- npm
- A Google AI Studio / Gemini API key
- A Supabase project
- Git (optional for local development)

### 1. Install dependencies

```bash
npm install
```

### 2. Configure environment variables

Create `.env.local` in the project root. Use the variable names expected by your current API route and Supabase clients. At minimum, the Supabase client setup uses:

```dotenv
NEXT_PUBLIC_SUPABASE_URL=your_supabase_project_url
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=your_supabase_publishable_key
```

Also configure the Gemini key using the **exact server-side variable name referenced in `src/app/api/plan/route.ts`**. Do not prefix the Gemini key with `NEXT_PUBLIC_`.

Keep `.env.local` out of Git. Do not commit API keys, passwords, Supabase secret keys, auth tokens, or real user data.

### 3. Set up Supabase

Create a Supabase project, then use the SQL Editor to create the application's tables and access policies. The schema includes:

- `events` — event information owned by an authenticated user
- `event_tasks` — checklist tasks whose access follows ownership of the parent event
- `ai_plans` — generated plans owned by the authenticated user, optionally linked to an event

Enable Row Level Security on every table and verify that the policies restrict reads and writes to the authenticated owner. This repository documentation describes the intended schema; use the actual migration SQL maintained with the project as the source of truth.

Configure Supabase Auth URL settings for local development. When email confirmation is enabled, include the local callback URL used by the application, for example `http://localhost:3000/auth/callback`. Add the production callback URL before deploying.

### 4. Start the development server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

### 5. Verify the build

```bash
npx tsc --noEmit
npm run build
```

Resolve any errors before deploying. Also test authentication and RLS behavior against your own Supabase project.

## Core routes and files

| Path | Purpose |
|---|---|
| `src/app/page.tsx` | Event dashboard, budgets, task checklist and account UI |
| `src/app/ai-planner/page.tsx` | AI planning form, rendered plan, saved-plan list and PDF print target |
| `src/app/api/plan/route.ts` | Server API route for AI plan generation |
| `src/app/login/page.tsx` | Registration and sign-in UI |
| `src/app/auth/callback/route.ts` | Auth callback handling |
| `src/lib/planora-cloud.ts` | Cloud loading, migration and persistence helpers |
| `src/lib/supabase/client.ts` | Browser Supabase client |
| `src/lib/supabase/server.ts` | Server Supabase client |
| `src/lib/supabase/proxy.ts` | Session refresh and authorization redirects |
| `src/proxy.ts` | Next.js request proxy entry point |
| `src/components/planora-controls.tsx` | Theme toggle and PDF export control |
| `src/app/globals.css` | Global styles, dark mode and print styles |

Paths may change as the project evolves; confirm against the current source tree.

## Data and security notes

- **Never commit `.env.local`.** Confirm `.gitignore` excludes local environment files, build output, and `node_modules` before pushing.
- Do not expose the Gemini API key in client components or public environment variables.
- Use Supabase RLS as the database security boundary. Client-side route redirects alone are not sufficient authorization.
- Verify that every table has RLS enabled and appropriate policies for `SELECT`, `INSERT`, `UPDATE`, and `DELETE`.
- Review the policies and migrations before deploying, including task ownership through the parent event and ownership validation for linked plans.
- The browser cache is a convenience, not the source of truth. Treat cloud synchronization warnings as failures until the corresponding records are verified in Supabase.
- AI-generated costs and recommendations are suggestions, not verified vendor quotes or contractual advice.
- PDF export uses the browser print dialog; output depends on the browser and print settings.

## Manual test checklist

- [ ] Register with name and email; confirm email when required.
- [ ] Sign in, refresh, sign out, and sign in again.
- [ ] Create an event and verify the row in Supabase `events`.
- [ ] Toggle a checklist task and verify `event_tasks.completed`.
- [ ] Generate a plan and verify its row in `ai_plans`.
- [ ] Reopen a saved plan and check its event link.
- [ ] Test theme switching and persistence after refresh.
- [ ] Export a generated plan to PDF and inspect the print preview.
- [ ] Test a second account to verify it cannot access the first account's records.
- [ ] Run `npx tsc --noEmit` and `npm run build`.

## Known considerations

- Real-time synchronization, offline write queues, and conflict resolution should be treated as separate features unless explicitly implemented and tested.
- Browser-local data migration should be tested with backups and more than one test account before relying on it for production use.
- For production deployment, set all required environment variables in the hosting provider and update Supabase Auth redirect URLs.

## Roadmap

- [ ] Deploy a public demo and add screenshots
- [ ] Add automated tests and CI checks
- [ ] Add robust retry and conflict handling for cloud writes
- [ ] Improve vendor management and deadline reminders
- [ ] Add account-level data export and deletion flows
- [ ] Add responsive mobile-navigation polish

## Contributing

Issues and pull requests are welcome. For significant changes, open an issue first to discuss the implementation. Please avoid submitting secrets, personal data, or production database contents.

## License

No license has been specified yet. Unless a license is added, all rights remain with the copyright holder and reuse is not automatically granted.

## Author

**Soumojit Maitra**

- GitHub: [@soumojit-dev](https://github.com/soumojit-dev)

---

*Built to make event planning more organized, personalized, and less stressful.*
