# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

**UIGen** is an AI-powered React component generator with live preview. Users describe UI in a chat interface; Claude generates React + Tailwind CSS components in real-time using a virtual file system, displayed in a live iframe preview.

## Commands

```bash
# Initial setup
npm run setup              # install + prisma generate + migrate

# Development
npm run dev                # Next.js dev server (Turbopack + node-compat shim)

# Build / Start
npm run build
npm start

# Lint
npm run lint               # next lint

# Tests
npx vitest                 # run all tests
npx vitest run <file>      # run a single test file
npx vitest --watch         # watch mode

# Database
npm run db:reset           # prisma migrate reset --force
```

Note: All `next` commands require `NODE_OPTIONS='--require ./node-compat.cjs'` (already baked into the npm scripts via `node-compat.cjs`).

## Architecture Overview

### 3-Panel Layout
The UI is a resizable 3-panel layout:
- **Left**: Chat interface (`src/components/chat/`)
- **Right top**: Live iframe preview (`src/components/preview/`) or Monaco code editor (`src/components/editor/`)
- State is shared via two root contexts: `FileSystemContext` and `ChatContext` (`src/lib/contexts/`)

### Virtual File System (VFS)
`src/lib/file-system.ts` — the core abstraction. An in-memory path→node map with create/read/update/delete and JSON serialization. It is used in three places:
1. **Client**: FileSystemContext holds the canonical VFS for the current session
2. **API route**: Deserializes the VFS sent from the client, runs Claude tool calls against it, then streams changes back
3. **Database**: VFS state is serialized to the `data` JSON column on the `Project` model

### AI Chat Flow (`src/app/api/chat/route.ts`)
1. Client POSTs `{ messages, data: serialized-vfs, projectId }`
2. Server deserializes VFS, calls Claude via Vercel AI SDK with two tools:
   - `str_replace_editor` — view / create / str-replace / undo-edit on files
   - `file_manager` — mkdir / list / rm / mv / cp / find
3. Claude streams text + tool call deltas back to client
4. Client applies tool call results to its local VFS
5. On finish: saves messages + VFS to Prisma

Tool definitions live in `src/lib/tools/`. System prompts live in `src/lib/prompts/`.

### Preview System
`src/lib/transform/jsx-transformer.ts` converts the VFS files to browser-runnable HTML:
- Builds an import map for npm modules
- Injects Babel standalone for in-browser JSX transpilation
- Renders into a sandboxed `<iframe>`

The preview entry point is always `/App.jsx` inside the VFS. Import paths use the `@/` alias.

### Auth
JWT sessions via `jose` + bcrypt passwords. `src/lib/auth.ts` signs/verifies 7-day httpOnly cookies. `src/middleware.ts` protects routes. Server actions in `src/actions/` handle sign-up, sign-in, and project CRUD.

Anonymous users are supported (userId is nullable on `Project`). `src/lib/anon-work-tracker.ts` tracks work done before registration so it can be attributed on sign-up.

### Database
Prisma + SQLite (`prisma/dev.db`). Two models: `User` and `Project`. Project stores `messages` (JSON) and `data` (serialized VFS JSON).

### Provider Abstraction
`src/lib/provider.ts` exports a `getLanguageModel()` factory. When `ANTHROPIC_API_KEY` is absent it returns a mock provider that returns static code — useful for UI development without an API key.

## Key Conventions

- Path alias `@/` maps to `src/`
- shadcn/ui components live in `src/components/ui/` — don't hand-edit these
- Tailwind CSS v4 (config-less, CSS-first)
- React 19 with Next.js App Router; use server components/actions where appropriate
- Prompt caching (`anthropic-beta: prompt-caching-2024-07-31`) is enabled on the system prompt — keep system prompt changes minimal to preserve cache hits
