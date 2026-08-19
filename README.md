# FlowSync

A full-stack project management application — Kanban boards, drag-and-drop task workflow, Gantt timelines, and cross-entity search — built with Next.js 14, Express, Prisma, and PostgreSQL, with authentication handled by AWS Cognito.

FlowSync models the core of a team planning tool: projects contain tasks, tasks belong to authors and assignees, and users belong to teams. The same task data is presented through four interchangeable views so a user can switch between board, list, table, and timeline without losing context.

**Stack:** TypeScript (client + server) · Next.js 14 App Router · Express · Prisma ORM · PostgreSQL · Redux Toolkit Query · Tailwind CSS · MUI · AWS Cognito

---

## Features

**Task management**
- Kanban board with drag-and-drop status transitions (`react-dnd`) across four columns: To Do → Work In Progress → Under Review → Completed
- Four views over the same dataset: Board, List, Table (MUI Data Grid), and Timeline
- Gantt-style timeline rendering with selectable Day / Week / Month granularity (`gantt-task-react`)
- Task metadata: priority, status, tags, story points, start and due dates, author, assignee, attachments, and comments

**Organization**
- Projects with descriptions and date ranges, each scoped to one or more teams
- Dedicated priority pages (Urgent, High, Medium, Low, Backlog) driven by a single reusable page component
- Teams view resolving product owner and project manager per team
- Users directory with profile images

**Cross-entity search**
- One search endpoint that queries tasks, projects, and users in parallel and returns grouped results
- Debounced client-side input (`lodash.debounce`) to limit request volume while typing

**Platform**
- Cognito-hosted authentication with a customized sign-up form (username, email, password confirmation)
- Persisted dark mode and sidebar collapse state via `redux-persist`
- Cached, tag-invalidated data fetching through RTK Query — mutations automatically refresh affected `Projects`, `Tasks`, `Users`, and `Teams` caches
- Hardened Express layer with `helmet`, CORS, and `morgan` request logging

---

## Architecture

```
FlowSync/
├── client/                          # Next.js 14 (App Router) frontend
│   └── src/
│       ├── app/
│       │   ├── layout.tsx           # Root layout
│       │   ├── dashboardWrapper.tsx # Sidebar + navbar shell
│       │   ├── authProvider.tsx     # Amplify / Cognito Authenticator
│       │   ├── redux.tsx            # Store provider + redux-persist gate
│       │   ├── home/                # Dashboard: bar + pie charts (recharts)
│       │   ├── projects/
│       │   │   ├── [id]/            # Dynamic project route
│       │   │   ├── BoardView/       # Drag-and-drop Kanban
│       │   │   ├── ListView/
│       │   │   ├── TableView/
│       │   │   ├── TimelineView/    # Gantt chart
│       │   │   ├── ProjectHeader.tsx
│       │   │   └── ModalNewProject/
│       │   ├── priority/            # Urgent / high / medium / low / backlog
│       │   │   └── reusablePriorityPage/
│       │   ├── search/  teams/  users/  timeline/  settings/
│       │   ├── components/          # Header, Modal, Navbar, Sidebar, cards
│       │   └── state/
│       │       ├── api.ts           # RTK Query API slice + typed models
│       │       └── index.ts         # Global UI slice (dark mode, sidebar)
│       └── lib/utils.ts
│
└── server/                          # Express + Prisma REST API
    ├── src/
    │   ├── index.ts                 # App bootstrap, middleware, route mounting
    │   ├── routes/                  # project / task / user / team / search
    │   └── controllers/             # Prisma query logic per resource
    └── prisma/
        ├── schema.prisma            # 7 models, relational constraints
        ├── migrations/              # SQL migration history
        ├── seed.ts                  # Ordered, idempotent seeding script
        └── seedData/                # JSON fixtures per table
```

**Request path:** React component → RTK Query hook → Express route → controller → Prisma Client → PostgreSQL. Responses are cached client-side by tag, so a `createTask` mutation invalidates the `Tasks` tag and triggers a refetch only for the affected queries.

---

## Data model

Seven Prisma models with enforced relational integrity:

| Model | Purpose | Key relations |
|---|---|---|
| `User` | Account record, keyed to a Cognito identity | `authoredTasks`, `assignedTasks`, `team`, `comments`, `attachments` |
| `Team` | Grouping of users, with product owner and project manager | `projectTeams`, `user[]` |
| `Project` | Container for tasks, with optional date range | `tasks`, `projectTeams` |
| `ProjectTeam` | Join table for the many-to-many project ↔ team link | `team`, `project` |
| `Task` | Work item with status, priority, tags, points, dates | `project`, `author`, `assignee`, `comments`, `attachments` |
| `TaskAssignment` | Join table supporting multi-user assignment | `user`, `task` |
| `Attachment` / `Comment` | Task-scoped files and discussion | `task`, `uploadedBy` / `user` |

`User` carries a unique `cognitoId`, which is how the API resolves an authenticated session to a database record.

---

## API reference

Base URL: `http://localhost:8000`

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/projects` | List all projects |
| `POST` | `/projects` | Create a project (`name`, `description`, `startDate`, `endDate`) |
| `GET` | `/tasks?projectId=:id` | List tasks for a project, with author, assignee, comments, and attachments |
| `POST` | `/tasks` | Create a task |
| `PATCH` | `/tasks/:taskId/status` | Update a task's status (used by drag-and-drop) |
| `GET` | `/tasks/user/:userId` | List tasks a user authored or is assigned to |
| `GET` | `/users` | List all users |
| `POST` | `/users` | Create a user record |
| `GET` | `/users/:cognitoId` | Resolve a user by Cognito ID |
| `GET` | `/teams` | List teams with product owner and project manager usernames |
| `GET` | `/search?query=:q` | Search tasks, projects, and users; returns grouped results |

---

## Getting started

### Prerequisites

- Node.js 18+
- PostgreSQL 14+ running locally
- An AWS Cognito user pool (for authentication)

### 1. Clone

```bash
git clone https://github.com/JoshuaKhooba/FlowSync.git
cd FlowSync
```

### 2. Database

Create the database:

```bash
createdb projectmanagement
```

### 3. Server

```bash
cd server
npm install
cp .env.example .env
```

Fill in `server/.env`:

```env
PORT=8000
DATABASE_URL="postgresql://USER:PASSWORD@localhost:5432/projectmanagement?schema=public"
```

Apply migrations, generate the Prisma client, and load seed data:

```bash
npx prisma migrate dev
npx prisma generate
npm run seed
```

Start the API:

```bash
npm run dev        # tsc --watch + nodemon
```

The server listens on `http://localhost:8000`.

### 4. Client

```bash
cd ../client
npm install
cp .env.example .env.local
```

Fill in `client/.env.local`:

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
NEXT_PUBLIC_COGNITO_USER_POOL_ID=your_user_pool_id
NEXT_PUBLIC_COGNITO_USER_POOL_CLIENT_ID=your_user_pool_client_id
```

Start the frontend:

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

---

## Scripts

**Server**

| Script | Action |
|---|---|
| `npm run dev` | Watch-compile TypeScript and run with nodemon |
| `npm run build` | Clean `dist/` and compile |
| `npm start` | Build, then run the compiled server |
| `npm run seed` | Truncate and reseed all tables in dependency order |

**Client**

| Script | Action |
|---|---|
| `npm run dev` | Next.js dev server |
| `npm run build` | Production build |
| `npm start` | Serve the production build |
| `npm run lint` | ESLint (`eslint-config-next`) |

---

## Implementation notes

- **Seeding order matters.** `prisma/seed.ts` clears and inserts tables in an explicit dependency order (`team → project → projectTeam → user → task → attachment → comment → taskAssignment`) so foreign key constraints never fail mid-run.
- **Optimistic-feeling drag-and-drop.** Board status changes fire a `PATCH` and invalidate the `Tasks` tag rather than mutating local state directly, keeping the client a pure projection of server state.
- **One priority page, five routes.** The five priority routes all render `reusablePriorityPage` with a different `Priority` enum value, so filtering logic lives in exactly one place.
- **Cognito is the identity source of truth.** The `User` table stores a `cognitoId` rather than credentials; the API never handles passwords.
- **Prisma binary targets** include `rhel-openssl-3.0.x` alongside `native` so the same schema builds for Linux deployment targets (AWS Lambda / EC2) as well as local development.

---

## Roadmap

- [ ] Comment and attachment UI (models and API relations exist; no frontend yet)
- [ ] `PATCH` / `DELETE` endpoints for projects and tasks
- [ ] Server-side authorization — verify the Cognito JWT per request and scope queries to the caller's teams
- [ ] Case-insensitive search (`mode: "insensitive"`) and pagination on list endpoints
- [ ] Jest + Supertest coverage for controllers, and React Testing Library for board interactions
- [ ] Docker Compose for Postgres + API, and a CI workflow running lint, typecheck, and tests
- [ ] Deployment: API on AWS (EC2 or Lambda behind API Gateway), client on Vercel or Amplify

---

## Author

**Joshua Khooba** — B.S. Information Technology, University of Central Florida
[Portfolio](https://joshuakhooba.com) · [GitHub](https://github.com/JoshuaKhooba) · [LinkedIn](https://linkedin.com/in/joshua-khooba)
