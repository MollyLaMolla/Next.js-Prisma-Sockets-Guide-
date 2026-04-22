# Next.js + Prisma Cheat Sheet — GAME‑HUB

**author:** Sasha (user)  
**created:** 2026-04-22

---

## Table of contents

1. [Quick reference — commands and snippets](#quick-reference---commands-and-snippets)
2. [App Router essentials and file roles](#app-router-essentials-and-file-roles)
3. [Dynamic routes and slugs](#dynamic-routes-and-slugs)
4. [loading.tsx, error.tsx, not-found.tsx — usage and pipeline](#loadingtsx-errortsx-not-foundtsx---usage-and-pipeline)
5. [middleware.ts — where and how to use it](#middlewarets---where-and-how-to-use-it)
6. [Components, hooks, lib — organization and best practices](#components-hooks-lib---organization-and-best-practices)
7. [Import path best practices and alias usage](#import-path-best-practices-and-alias-usage)
8. [CSS strategy and naming conventions](#css-strategy-and-naming-conventions)
9. [Socket.IO integration for realtime 1v1](#socketio-integration-for-realtime-1v1)
10. [Example project structure (expanded)](#example-project-structure-expanded)
11. [Copy‑pasteable example files (key snippets)](#copy-pasteable-example-files-key-snippets)
12. [Prisma schema examples (Postgres and Mongo)](#prisma-schema-examples-postgres-and-mongo)
13. [Production checklist and best practices](#production-checklist-and-best-practices)
14. [Final quick reference (cheat sheet)](#final-quick-reference-cheat-sheet)
15. [References](#references)

---

## Quick reference — commands and snippets

### Create Next.js project (TypeScript, Tailwind, App Router, alias)

```bash
npx create-next-app@latest my-app --ts --tailwind --app --import-alias '@/*'
cd my-app
npm run dev
```

### Prisma setup

```bash
npm install prisma --save-dev
npx prisma init
npm install @prisma/client
```

### Prisma — PostgreSQL common commands

```bash
# create migration and apply locally
npx prisma migrate dev --name init

# generate client (usually automatic)
npx prisma generate

# add a migration after schema changes
npx prisma migrate dev --name add-users

# deploy migrations in production
npx prisma migrate deploy

# check migration status
npx prisma migrate status
```

### Prisma — MongoDB notes

- In `schema.prisma` set `provider = "mongodb"`.
- For many Mongo workflows use:

```bash
npx prisma db push
```

instead of `migrate dev`.

### Useful npm scripts (package.json)

```json
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "start": "next start",
  "prisma:migrate": "prisma migrate dev",
  "prisma:generate": "prisma generate"
}
```

---

## App Router essentials and file roles

**Core files per route (in `app/`):**

- `page.tsx` — main component for the route (Server Component by default).
- `layout.tsx` — wraps children for that segment; can be nested.
- `loading.tsx` — shown automatically while server components or nested routes load.
- `error.tsx` — client-side error boundary for that segment; must be a Client Component (`"use client"`).
- `not-found.tsx` — shown when `notFound()` is called in a server component or resource missing.

**Server Components vs Client Components**

- Files in `app/` are **Server Components** by default. Use them to fetch from DB (Prisma) and render HTML.
- Add `"use client"` at the top to make a **Client Component** (for hooks, event handlers, socket usage).

**Nested layouts**

- Place `layout.tsx` in a folder to scope shared UI to that folder and its children.
- Example: `app/games/layout.tsx` provides a games-specific header/sidebar for `/games/*`.

**Example server page**

```tsx
// app/games/[slug]/page.tsx
import { prisma } from '@/lib/prisma'
import { notFound } from 'next/navigation'

export default async function GamePage({ params }: { params: { slug: string } }) {
  const game = await prisma.game.findUnique({ where: { slug: params.slug } })
  if (!game) notFound()
  return (
    <main>
      <h1>{game.title}</h1>
    </main>
  )
}
```

---

## Dynamic routes and slugs

**Patterns**

- `[slug]` — single dynamic segment (e.g., `/games/tris`).
- `[...slug]` — catch-all required (returns array).
- `[[...slug]]` — optional catch-all (may be undefined).

**Accessing params**

- Server components receive `params` prop: `params.slug`.
- API route handlers receive `params` in the second argument: `export async function GET(req, { params })`.

**Where to place dynamic routes**

- Public content (profiles, games, products) → `app/<collection>/[slug]/page.tsx`.
- Private account pages → prefer `/account` (no slug) and identify user via session/middleware.

**Catch-all example**

```tsx
// app/docs/[...slug]/page.tsx
export default function DocsPage({ params }: { params: { slug?: string[] } }) {
  const path = params.slug ? params.slug.join('/') : 'index'
  return <div>Doc path: {path}</div>
}
```

---

## loading.tsx, error.tsx, not-found.tsx — usage and pipeline

**`loading.tsx`**

- Purpose: show skeleton or spinner while server-side data loads.
- Placement: in the folder whose loading state you want to represent (e.g., `app/games/loading.tsx`).
- Keep it lightweight; do not perform data fetching here.

```tsx
// app/games/loading.tsx
export default function Loading() {
  return <div className="p-6">Loading games…</div>
}
```

**`error.tsx`**

- Purpose: error boundary for runtime errors in that route segment.
- Must be a **Client Component** and accept `error` and `reset` props.
- Use `reset()` to retry rendering (Next will re-render the server component).

```tsx
// app/games/error.tsx
'use client'
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <h2>Something went wrong</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```

**`not-found.tsx`**

- Purpose: custom 404 for that segment; triggered by calling `notFound()` in server code.
- Place it in dynamic route folder to handle missing slugs.

```tsx
// app/games/[slug]/not-found.tsx
export default function NotFound() {
  return <div>Game not found</div>
}
```

**Pipeline**

1. `loading.tsx` while data loads
2. `page.tsx` if OK
3. `not-found.tsx` if `notFound()` called
4. `error.tsx` if runtime error occurs

---

## middleware.ts — where and how to use it

**Location**

- Place `middleware.ts` at project root (same level as `app/`).

**Common uses**

- Protect private routes (`/account/*`, `/admin/*`).
- Redirect root `/` to `/games`.
- Add headers, rate limiting, or check tokens.

**Example**

```ts
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { getToken } from 'next-auth/jwt' // if using NextAuth

export async function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl
  // protect /account
  if (pathname.startsWith('/account')) {
    const token = await getToken({ req })
    if (!token) return NextResponse.redirect(new URL('/login', req.url))
  }
  // redirect root
  if (pathname === '/') return NextResponse.redirect(new URL('/games', req.url))
  return NextResponse.next()
}

export const config = {
  matcher: ['/account/:path*', '/admin/:path*', '/api/protected/:path*'],
}
```

**Best practices**

- Keep middleware fast and minimal (no heavy DB queries).
- Use `matcher` to limit scope.
- For complex auth checks, validate tokens quickly or call a lightweight auth service.

---

## Components, hooks, lib — organization and best practices

**Global vs route-local**

- **Global components**: `components/` at project root for UI used across the app (Navbar, Button, Modal).
- **Route-local components**: place inside the route folder (e.g., `app/games/components/GameCard.tsx`) when they are only relevant to that route.

**Hooks**

- **Global hooks**: `hooks/` at root for reusable hooks (`useAuth`, `useSocket`, `useLocalStorage`).
- **Route-local hooks**: optional `app/games/hooks/` for hooks only used by that route (e.g., `useGameTimer`).

**Lib**

- Keep server-side helpers and services in `lib/` (e.g., `prisma.ts`, `auth.ts`, `gameService.ts`). Avoid creating `lib` inside route folders — centralize server logic.

**Why this separation**

- Clarity: you know where to find shared vs local code.
- Encapsulation: route-specific code stays close to the route.
- Reusability: global utilities are discoverable and importable from anywhere.

---

## Import path best practices and alias usage

**Use alias `@/` for global imports**

- Configure `tsconfig.json` / `jsconfig.json` to map `@/*` to project root (create-next-app with `--import-alias '@/*'` does this).

```ts
import Navbar from '@/components/Navbar'
import useAuth from '@/hooks/useAuth'
import { prisma } from '@/lib/prisma'
```

**Relative imports for local files**

- Use relative imports for files inside the same route:

```ts
import GameCard from './components/GameCard'
```

**Avoid long relative chains**

- Don’t use `../../../components/...`. Use `@/components/...` instead.

**Import ordering**

1. React / Next imports
2. External libraries
3. `@/` alias imports (global)
4. Relative imports (local)
5. CSS imports

**Example**

```ts
import React from 'react'
import Link from 'next/link'

import { prisma } from '@/lib/prisma'
import Navbar from '@/components/Navbar'

import GameCard from './components/GameCard'

import './games.css'
```

---

## CSS strategy and naming conventions

**`app/globals.css`**

- Use for resets, fonts, CSS variables, Tailwind base/components/utilities import, and global utility classes.

**Per-route CSS**

- When you need pixel-perfect styling, create a CSS file in the route folder:
  - `app/games/games.css`
  - `app/games/[slug]/game.css`
- Import it in `page.tsx` or `layout.tsx` of that route:

```ts
import './games.css'
```

**CSS Modules**

- For component-scoped styles use `*.module.css`:

```css
/* GameCard.module.css */
.card {
  padding: 12px;
}
```

```tsx
import styles from './GameCard.module.css'
;<div className={styles.card}>...</div>
```

**Tailwind + custom CSS**

- Use Tailwind for layout, spacing, utilities.
- Use custom CSS for fine-grained pixel-perfect rules and complex animations.

**Naming**

- Name CSS files after the route or component: `games.css`, `game.css`, `account.css`, `GameCard.module.css`.

---

## Socket.IO integration for realtime 1v1

**Architecture options**

- **Best practice**: run a Socket.IO server attached to the Next.js HTTP server (single process) or run a separate Node server for sockets (if scaling with separate infra).
- **Do not** rely on route handlers (`route.ts`) for persistent sockets — they are stateless.

**Server example**

```ts
// socket/server.ts
import { Server } from 'socket.io'

let io: Server | null = null

export function initSocket(server) {
  if (io) return io
  io = new Server(server, { cors: { origin: '*' } })

  const queue: any[] = []

  io.on('connection', socket => {
    socket.on('find-match', () => {
      queue.push(socket)
      if (queue.length >= 2) {
        const a = queue.shift()
        const b = queue.shift()
        const room = `match-${a.id}-${b.id}`
        a.join(room)
        b.join(room)
        io!.to(room).emit('match-found', { room })
      }
    })

    socket.on('join-room', room => socket.join(room))
    socket.on('move', data => io!.to(data.room).emit('opponent-move', data))
    socket.on('invite', ({ toSocketId, payload }) => {
      io!.to(toSocketId).emit('invite-received', payload)
    })
  })

  return io
}
```

**Attach to Next server**

```js
// server.js
const { createServer } = require('http')
const next = require('next')
const { initSocket } = require('./socket/server')

const app = next({ dev: process.env.NODE_ENV !== 'production' })
const handle = app.getRequestHandler()

app.prepare().then(() => {
  const server = createServer((req, res) => handle(req, res))
  initSocket(server)
  server.listen(3000)
})
```

**Client hook**

```ts
// hooks/useSocket.ts
'use client'
import { useEffect, useRef } from 'react'
import { io, Socket } from 'socket.io-client'

export default function useSocket() {
  const socketRef = useRef<Socket | null>(null)
  useEffect(() => {
    socketRef.current = io()
    return () => {
      socketRef.current?.disconnect()
    }
  }, [])
  return socketRef.current
}
```

**Matchmaking flow**

1. Client emits `find-match`.
2. Server pushes socket to queue.
3. When two sockets available, server creates a room and emits `match-found`.
4. Clients join room and exchange moves via `io.to(room).emit(...)`.

**Invites and friends**

- Persist friend relationships and invites in DB via API (`/api/friends`, `/api/invites`).
- Use socket events to notify online users of invites in real time.

**Scaling sockets**

- For horizontal scaling use a Redis adapter so socket instances share rooms/events.

---

## Example project structure (expanded)

```
project-root/
├── app/
│   ├── layout.tsx
│   ├── globals.css
│   ├── page.tsx
│   ├── loading.tsx
│   ├── error.tsx
│   ├── not-found.tsx
│   ├── games/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── games.css
│   │   ├── loading.tsx
│   │   ├── error.tsx
│   │   ├── components/
│   │   │   ├── GameCard.tsx
│   │   │   ├── GameDetails.tsx
│   │   │   └── GameActions.tsx
│   │   └── [slug]/
│   │       ├── page.tsx
│   │       ├── game.css
│   │       ├── loading.tsx
│   │       ├── error.tsx
│   │       └── not-found.tsx
│   ├── account/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── account.css
│   │   ├── settings/
│   │   │   ├── page.tsx
│   │   │   └── settings.css
│   │   └── friends/
│   │       └── page.tsx
│   └── api/
│       ├── games/
│       │   ├── route.ts
│       │   └── [slug]/route.ts
│       ├── friends/route.ts
│       ├── invites/route.ts
│       └── auth/route.ts
├── components/
│   ├── Navbar.tsx
│   ├── Footer.tsx
│   ├── Button.tsx
│   ├── Card.tsx
│   └── Modal.tsx
├── hooks/
│   ├── useAuth.ts
│   ├── useSocket.ts
│   ├── useMatchmaking.ts
│   └── useLocalStorage.ts
├── lib/
│   ├── prisma.ts
│   ├── auth.ts
│   ├── validate.ts
│   ├── slugify.ts
│   └── gameService.ts
├── socket/
│   ├── server.ts
│   └── matchmaking.ts
├── public/
│   ├── images/
│   └── favicon.ico
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── middleware.ts
├── server.js
├── package.json
├── tsconfig.json
└── README.md
```

---

## Copy‑pasteable example files (key snippets)

> Each snippet includes a short explanation and a fenced code block you can copy.

### `lib/prisma.ts` — single Prisma client (avoid multiple instances in dev)

```ts
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'

declare global {
  // prevent multiple instances in dev
  var prisma: PrismaClient | undefined
}

export const prisma = global.prisma ?? new PrismaClient()
if (process.env.NODE_ENV === 'development') global.prisma = prisma
```

### `app/api/games/[slug]/route.ts` — API route example (GET, DELETE)

```ts
// app/api/games/[slug]/route.ts
import { prisma } from '@/lib/prisma'

export async function GET(req: Request, { params }: { params: { slug: string } }) {
  const game = await prisma.game.findUnique({ where: { slug: params.slug } })
  if (!game) return new Response(null, { status: 404 })
  return new Response(JSON.stringify(game), { status: 200 })
}

export async function DELETE(req: Request, { params }: { params: { slug: string } }) {
  await prisma.game.delete({ where: { slug: params.slug } })
  return new Response(JSON.stringify({ success: true }))
}
```

### `app/games/[slug]/page.tsx` — server page using Prisma and `notFound()`

```tsx
// app/games/[slug]/page.tsx
import { prisma } from '@/lib/prisma'
import GameDetails from '../components/GameDetails'
import { notFound } from 'next/navigation'

export default async function Page({ params }: { params: { slug: string } }) {
  const game = await prisma.game.findUnique({ where: { slug: params.slug } })
  if (!game) notFound()
  return <GameDetails game={game} />
}
```

### `app/games/loading.tsx` — lightweight loading UI

```tsx
// app/games/loading.tsx
export default function Loading() {
  return <div className="p-6">Loading games…</div>
}
```

### `app/games/[slug]/not-found.tsx` — custom 404 for a game

```tsx
// app/games/[slug]/not-found.tsx
export default function NotFound() {
  return (
    <div>
      Game not found. <a href="/games">Back to games</a>
    </div>
  )
}
```

### `components/Navbar.tsx` — simple global navbar

```tsx
// components/Navbar.tsx
import Link from 'next/link'

export default function Navbar() {
  return (
    <nav className="flex items-center justify-between p-4">
      <Link href="/">
        <span className="font-bold">GAME-HUB</span>
      </Link>
      <div className="flex gap-4">
        <Link href="/games">Games</Link>
        <Link href="/account">Account</Link>
      </div>
    </nav>
  )
}
```

### `hooks/useSocket.ts` — client hook for socket connection

```ts
// hooks/useSocket.ts
'use client'
import { useEffect, useRef } from 'react'
import { io, Socket } from 'socket.io-client'

export default function useSocket() {
  const socketRef = useRef<Socket | null>(null)
  useEffect(() => {
    socketRef.current = io()
    return () => {
      socketRef.current?.disconnect()
    }
  }, [])
  return socketRef.current
}
```

### `socket/server.ts` — matchmaking + rooms (server-side)

```ts
// socket/server.ts
import { Server } from 'socket.io'

let io: Server | null = null

export function initSocket(server) {
  if (io) return io
  io = new Server(server, { cors: { origin: '*' } })

  const queue = []

  io.on('connection', socket => {
    socket.on('find-match', () => {
      queue.push(socket)
      if (queue.length >= 2) {
        const a = queue.shift()
        const b = queue.shift()
        const room = `match-${a.id}-${b.id}`
        a.join(room)
        b.join(room)
        io.to(room).emit('match-found', { room })
      }
    })

    socket.on('join-room', room => socket.join(room))
    socket.on('move', data => io.to(data.room).emit('opponent-move', data))
    socket.on('invite', ({ toSocketId, payload }) => {
      io.to(toSocketId).emit('invite-received', payload)
    })
  })

  return io
}
```

### `server.js` — attach socket server to Next.js

```js
// server.js
const { createServer } = require('http')
const next = require('next')
const { initSocket } = require('./socket/server')

const app = next({ dev: process.env.NODE_ENV !== 'production' })
const handle = app.getRequestHandler()

app.prepare().then(() => {
  const server = createServer((req, res) => handle(req, res))
  initSocket(server)
  server.listen(3000, () => console.log('Listening on http://localhost:3000'))
})
```

### `middleware.ts` — protect `/account` example

```ts
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { getToken } from 'next-auth/jwt'

export async function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl
  if (pathname.startsWith('/account')) {
    const token = await getToken({ req })
    if (!token) return NextResponse.redirect(new URL('/login', req.url))
  }
  return NextResponse.next()
}

export const config = { matcher: ['/account/:path*'] }
```

---

## Prisma schema examples (Postgres and Mongo)

### PostgreSQL example

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  friends   Friend[] // example relation
}

model Game {
  id        Int      @id @default(autoincrement())
  title     String
  slug      String   @unique
  createdAt DateTime @default(now())
}
```

### MongoDB example

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "mongodb"
  url      = env("DATABASE_URL")
}

model User {
  id    String @id @default(auto()) @map("_id") @db.ObjectId
  name  String
  email String @unique
}
```

**Notes**

- For Postgres use migrations (`prisma migrate dev` locally, `prisma migrate deploy` in prod).
- For MongoDB, `prisma db push` is commonly used to sync schema.

---

## Production checklist and best practices

- **Migrations**: use `prisma migrate deploy` in CI/CD for production. Keep migrations in repo.
- **Environment**: store secrets in `.env` and in your hosting provider secret store.
- **Connection pooling**: for serverless Postgres use PgBouncer or a connection pooler to avoid exhausting DB connections.
- **Rate limiting**: implement in `middleware.ts` or API routes for sensitive endpoints.
- **Auth**: use NextAuth or a robust custom auth; validate tokens in middleware.
- **Logging and monitoring**: capture server logs, socket events, and errors.
- **Testing**: unit tests for services, integration tests for API routes, end-to-end tests for flows.
- **Security**: validate inputs, avoid exposing internal IDs, use `notFound()` for missing resources, sanitize user input.
- **Scaling sockets**: if you scale horizontally, use a socket adapter (Redis) to share rooms/events across instances.
- **CI/CD**: run `prisma generate` and `prisma migrate deploy` as part of deploy pipeline.

---

## Final quick reference (cheat sheet)

**Next.js**

```bash
npx create-next-app@latest my-app --ts --tailwind --app --import-alias '@/*'
npm run dev
npm run build
npm run start
```

**Prisma**

```bash
npm install prisma --save-dev
npx prisma init
npm install @prisma/client
npx prisma migrate dev --name init
npx prisma generate
npx prisma migrate deploy
npx prisma db push   # for MongoDB or quick sync
```

**Socket**

- Start server with `node server.js` (or via `npm run dev` if integrated).
- Client: `const socket = io()` in a client component or hook.
