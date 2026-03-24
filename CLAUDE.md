# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup       # Install deps + Prisma generate + DB push (first-time setup)
npm run dev         # Start dev server with Turbopack
npm run build       # Production build
npm run lint        # ESLint
npm run test        # Run all tests with Vitest
npm run db:reset    # Reset SQLite database
```

Run a single test file:
```bash
npx vitest src/lib/__tests__/file-system.test.ts
```

Requires optional `ANTHROPIC_API_KEY` in `.env` — falls back to a mock provider (static examples) when absent.

## Architecture

UIGen is a Next.js 15 (App Router) app where Claude generates React components from natural language. The core loop: user describes a component → Claude calls file-system tools → virtual FS updates → iframe re-renders.

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`)
In-memory Map-based FS that never touches disk. All generated code lives here. Serialized to JSON for database persistence. The `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) exposes it to React.

**AI Tool Integration** (`src/app/api/chat/route.ts`)
The chat API route uses Vercel AI SDK's `streamText` with two tools Claude can call:
- `str_replace_editor` — create/read/edit files via string replacement (`src/lib/tools/str-replace.ts`)
- `file_manager` — create directories, delete/rename files (`src/lib/tools/file-manager.ts`)

Tool calls stream back to the client, which applies them to the virtual FS in real time.

**Preview System** (`src/components/preview/PreviewFrame.tsx`)
Renders generated code in a sandboxed `<iframe>`. Uses Babel standalone for in-browser JSX/TSX compilation and import maps pointing to `esm.sh` CDN for dependency resolution. Auto-detects the entry point (looks for `App.jsx`, `index.jsx`, etc.). Hot-reloads when the virtual FS changes.

**Chat Context** (`src/lib/contexts/chat-context.tsx`)
Manages message history and drives the streaming API call. After each exchange it persists messages + serialized file system to the database via server actions.

**Authentication** (`src/lib/auth.ts`, `src/middleware.ts`)
JWT cookies, 7-day expiry, bcrypt passwords. Auth is optional — anonymous users can use the app but projects aren't persisted across sessions. The middleware protects `/api/` routes and redirects based on auth state.

**Language Model Provider** (`src/lib/provider.ts`)
Returns an Anthropic model if `ANTHROPIC_API_KEY` is set, otherwise returns a mock provider that yields a static example component. The system prompt for generation lives in `src/lib/prompts/generation.tsx`.

### Data Flow

```
User message
  → ChatContext.sendMessage()
  → POST /api/chat  (with messages + serialized file system)
  → Claude streams text + tool calls
  → Client applies tool results to FileSystemContext
  → PreviewFrame detects FS change → re-compiles + re-renders iframe
  → ChatContext saves project to DB (server action)
```

### Database

SQLite via Prisma. Schema: `User` (id, email, password) → `Project` (id, name, userId?, messages JSON, data JSON). Anonymous projects have `userId = null`.

### Path Alias

`@/*` maps to `src/*` throughout the codebase.

## Code Style

Avoid comments unless the logic is genuinely non-obvious. No explanatory, section, or descriptive comments.
