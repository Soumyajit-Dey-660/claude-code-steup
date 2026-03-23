# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, Claude generates them, and they are previewed in real-time via a virtual (in-memory) file system — no files are written to disk.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (Next.js with Turbopack)
npm run dev

# Build for production
npm run build

# Run tests (Vitest)
npm run test

# Run a single test file
npx vitest src/components/chat/__tests__/ChatInterface.test.tsx

# Lint
npm run lint

# Reset database
npm run db:reset
```

The `NODE_OPTIONS="--require ./node-compat.cjs"` prefix is injected automatically by the npm scripts via `cross-env`.

## Architecture

### Core Concepts

**Virtual File System** (`src/lib/file-system.ts`): All generated component files live in memory as a `VirtualFileSystem` class. It supports CRUD and rename operations, and serializes/deserializes for DB persistence. This is intentional — nothing the AI generates is written to disk.

**AI Chat Route** (`src/app/api/chat/route.ts`): Uses Vercel AI SDK `streamText` with Claude. The AI uses two tools — `file-manager` (create/read/update/delete/rename virtual files) and `str-replace` (targeted edits) — to build and modify components. Prompt caching is enabled via ephemeral cache control headers. Falls back to a mock provider if `ANTHROPIC_API_KEY` is absent.

**Authentication** (`src/lib/auth.ts` + `src/middleware.ts`): JWT sessions stored in HttpOnly cookies (7-day expiry), bcrypt password hashing. Middleware handles session verification. Auth is optional — anonymous users can generate components, which are tracked in `anon-work-tracker.ts`.

**Database** (`prisma/schema.prisma`): SQLite via Prisma. Two models: `User` and `Project`. Projects store chat messages and the entire virtual FS state as JSON blobs. Projects can be anonymous (`userId` is nullable).

### State Management

- `src/lib/contexts/chat-context.tsx` — chat messages and streaming state
- `src/lib/contexts/file-system-context.tsx` — virtual FS React context
- `src/lib/anon-work-tracker.ts` — tracks anonymous user session work for persistence prompts

### Key Directories

- `src/app/api/chat/` — AI streaming endpoint
- `src/lib/tools/` — AI tool implementations (`file-manager.ts`, `str-replace.ts`)
- `src/lib/prompts/` — system prompt for component generation
- `src/actions/` — Next.js server actions for project CRUD
- `src/components/preview/` — live preview iframe (`PreviewFrame.tsx`)
- `src/components/editor/` — Monaco-based code editor and file tree

## Environment

Requires `.env` with:
```
ANTHROPIC_API_KEY=...
```

Without it, the app uses a mock provider that returns placeholder responses.
