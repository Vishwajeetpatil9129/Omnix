# Omnix

Omnix turns a plain-English prompt into a working Next.js application. You describe what you want, an AI agent writes and runs the code inside an isolated cloud sandbox, and you get a live preview plus the full source tree — all in the browser.

## How it works

```
Prompt ──▶ Server Action ──▶ Inngest event ──▶ code-agent (Gemini + agent-kit)
                                                   │
                                    ┌──────────────┼──────────────┐
                                    ▼              ▼              ▼
                                 terminal   createOrUpdateFiles  readFiles
                                    └──────────────┼──────────────┘
                                                   ▼
                                          E2B sandbox (Next.js dev server)
                                                   │
                             ┌─────────────────────┴─────────────────────┐
                             ▼                                           ▼
                  Fragment (live URL + files)                  Assistant message
```

1. Submitting a prompt creates a `Project` with the first `Message` and sends a `code-agent/run` event to Inngest.
2. The background function boots an [E2B](https://e2b.dev) sandbox from a prebuilt Next.js + shadcn/ui template.
3. A `createNetwork` loop (max 10 iterations) runs the coding agent until it emits a `<task_summary>`. The agent has three tools: run shell commands, create/update files, and read files — all executed inside the sandbox.
4. Two follow-up agents generate a short fragment title and a user-facing summary.
5. The result is persisted as a `Message` with an attached `Fragment` holding the sandbox URL and the generated file map.
6. The client polls with React Query, then renders the fragment as either a live iframe preview or a browsable file tree with syntax highlighting.

Conversation history is replayed into agent state on every run, so follow-up prompts iterate on the same project.

## Tech stack

| Layer | Choice |
| --- | --- |
| Framework | Next.js 16 (App Router, React 19, React Compiler) |
| Styling | Tailwind CSS v4, shadcn/ui, Radix, `next-themes` |
| Auth & billing | Clerk (middleware-protected routes, `PricingTable`) |
| Database | PostgreSQL via Prisma |
| Background jobs | Inngest |
| Agents | `@inngest/agent-kit` with Gemini 2.5 Flash |
| Code execution | E2B Code Interpreter sandboxes |
| Data fetching | Server Actions + TanStack Query |

## Project structure

```
src/
├─ app/
│  ├─ (root)/              # Landing, projects list, pricing, sign-in, sign-up
│  ├─ projects/[projectId] # Chat + preview workspace
│  ├─ api/inngest/         # Inngest serve endpoint
│  └─ sitemap.js
├─ modules/                # Feature slices: actions/, components/, hooks/
│  ├─ auth/                # Clerk onboarding, getCurrentUser
│  ├─ home/                # Landing page, navbar, project form & list
│  ├─ messages/            # Message queries/mutations
│  ├─ projects/            # Project CRUD, message thread, file explorer, preview
│  └─ usage/               # Credit display
├─ inngest/                # Client, agent function, output helpers
├─ lib/                    # Prisma client, rate limiter, cn()
├─ components/             # shadcn/ui primitives, providers
├─ prompt.js               # System prompts for the three agents
└─ middleware.js           # Clerk route protection
prisma/schema.prisma       # User, Project, Message, Fragment, Usage
sandbox-templates/next-js/ # E2B template (Dockerfile + start script)
```

## Data model

- **User** — mirrored from Clerk on first sign-in (`clerkId` is the link).
- **Project** — one chat thread; owns many messages.
- **Message** — `role` (`USER` / `ASSISTANT`) and `type` (`RESULT` / `ERROR`).
- **Fragment** — the artifact of a successful run: `sandboxUrl`, `title`, and `files` (JSON path → content). One per message.
- **Usage** — backing table for `rate-limiter-flexible`, used as the credit ledger.

## Credits

Generation is metered by `src/lib/usage.js`: **5 credits** on the free plan, **100** with the Clerk `pro` plan, refilled on a 30-day window. Each project creation consumes one credit before the agent is dispatched.

## Getting started

### Prerequisites

- Node.js 20+
- Docker (for the local Postgres container)
- Accounts for [Clerk](https://clerk.com), [E2B](https://e2b.dev), and [Google AI Studio](https://aistudio.google.com) (Gemini)

### 1. Install

```bash
npm install
```

### 2. Configure environment

Create a `.env` in the project root:

```bash
# Database — the local docker-compose Postgres, or a hosted URL (e.g. Neon)
DATABASE_URL="postgresql://postgres:postgres@localhost:5431/postgres"
DIRECT_URL="postgresql://postgres:postgres@localhost:5431/postgres"

# Clerk
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=

# Gemini
GEMINI_API_KEY=

# E2B sandboxes
E2B_API_KEY=

# Inngest (local dev)
INNGEST_DEV=1
INNGEST_BASE_URL=http://localhost:8288
```

### 3. Set up the database

```bash
docker compose up -d
npx prisma migrate dev
npx prisma generate
```

### 4. Run

Three processes:

```bash
npm run dev                    # starts Postgres + Next.js on :3000
npx inngest-cli@latest dev     # Inngest dev server on :8288
```

Open [http://localhost:3000](http://localhost:3000).

> The agent will not produce anything until the Inngest dev server is running — project creation only enqueues an event.

### 5. (Optional) Build your own sandbox template

The agent boots the E2B template named in `src/inngest/functions.js` (`Sandbox.create("v0-nextjs-build-new")`). To build your own from `sandbox-templates/next-js/`:

```bash
cd sandbox-templates/next-js
npx @e2b/cli template build --name your-template-name
```

Then update the template name in `src/inngest/functions.js`.

## Scripts

| Command | Description |
| --- | --- |
| `npm run dev` | Start the Postgres container and the Next.js dev server |
| `npm run build` | Production build |
| `npm run start` | Serve the production build |
| `npm run lint` | ESLint |

## Notes

- Sandboxes are ephemeral. Older fragments' preview URLs expire; the stored file map remains browsable in the **Code** tab.
- The agent run has a 10-minute finish timeout and caps at 10 network iterations.
- Public routes are `/`, `/pricing`, `/sign-in`, `/sign-up`, `/api/*`, and the sitemap; everything else requires auth.
