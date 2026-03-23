# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies, generate Prisma client, run migrations
npm run setup

# Development server (Turbopack)
npm run dev

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/transform/__tests__/jsx-transformer.test.ts

# Lint
npm run lint

# Production build
npm run build

# Reset database
npm run db:reset
```

## Architecture Overview

UIGen is a Next.js 15 app that lets users generate React components through AI chat, with live preview and optional persistence.

### Core Data Flow

1. User sends a prompt in `ChatInterface` → POST to `/api/chat`
2. The chat API streams a response using Vercel AI SDK + Anthropic Claude (`claude-haiku-4-5`)
3. Claude uses two tools during generation:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — creates/edits files via string replacement
   - `file_manager` (`src/lib/tools/file-manager.ts`) — renames/deletes files
4. Tool calls mutate the **virtual file system** (in-memory, no disk I/O) managed by `FileSystemProvider`
5. `PreviewFrame` renders an iframe where Babel transforms JSX and an import map redirects imports to `esm.sh` CDN
6. For authenticated users, the completed project is persisted to SQLite via Prisma

### Key Modules

- **`src/app/api/chat/route.ts`** — Streaming chat endpoint; selects real Claude or mock provider based on `ANTHROPIC_API_KEY`
- **`src/lib/provider.ts`** — Switches between Anthropic and mock language model provider
- **`src/lib/file-system.ts`** — Virtual file system with CRUD + serialization
- **`src/lib/transform/jsx-transformer.ts`** — Babel-based JSX→JS transform, import map generation for iframe preview
- **`src/lib/contexts/`** — `FileSystemProvider` holds file state; `ChatProvider` holds message history and streams AI responses
- **`src/lib/prompts/generation.tsx`** — System prompt sent to Claude for component generation
- **`src/actions/`** — Next.js server actions for auth (sign-up/in/out) and project CRUD
- **`src/lib/auth.ts`** — JWT sessions via `jose`, stored in httpOnly cookies; `bcrypt` for passwords

### Authentication & Projects

- Auth is optional — anonymous users can generate components without logging in
- `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes
- Projects are stored in SQLite (`prisma/dev.db`) via the `User` and `Project` Prisma models
- On page load, authenticated users are redirected to their most recent project or a new one is created

### Preview Mechanism

- `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) renders components inside an `<iframe>`
- `jsx-transformer.ts` transforms files with Babel (browser build), generates an HTML document with an import map pointing to `esm.sh`
- Supports React 19 and arbitrary npm packages via CDN — no bundler required at runtime

### Environment

- `ANTHROPIC_API_KEY` in `.env` — optional; app runs with a mock provider if absent
- Database: SQLite at `prisma/dev.db`
