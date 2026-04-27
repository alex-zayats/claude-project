# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates/edits React code inside a virtual file system; a sandboxed iframe renders the result in real time.

## Commands

```bash
npm run setup       # First-time setup: install deps, generate Prisma client, run migrations
npm run dev         # Start dev server with Turbopack on port 3000
npm run build       # Production build
npm run lint        # ESLint
npm run test        # Vitest (all tests)
npm run db:reset    # Force-reset SQLite migrations
```

Run a single test file:
```bash
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx
```

## Architecture

### Request Flow

1. User sends a message → `ChatContext` (wraps Vercel AI SDK's `useChat`) streams to `POST /api/chat`.
2. The route handler calls Claude (`claude-haiku-4-5`) with a system prompt instructing it to produce React components. If `ANTHROPIC_API_KEY` is absent, `MockLanguageModel` in `src/lib/provider.ts` returns static templates instead.
3. Claude responds with tool calls (`str_replace_editor`, `file_manager`) that mutate the **virtual file system**.
4. `FileSystemContext` processes those tool calls and updates in-memory state.
5. The `PreviewFrame` re-renders the iframe on every FS change; Babel standalone transpiles JSX in-browser.
6. Final messages + serialized FS state are persisted to SQLite via Prisma.

### Virtual File System (`src/lib/file-system.ts`)

Entirely in-memory — not disk-backed. Supports create/read/update/delete/rename and JSON serialization for DB storage. The AI always works against `/App.jsx` as the root component; other files are importable via `@/` aliases.

### AI Tools

- **`str_replace_editor`** (`src/lib/tools/str-replace.ts`) — view, create, str_replace, insert, undo_edit commands (Zod-validated).
- **`file_manager`** (`src/lib/tools/file-manager.ts`) — rename, delete.

Both tools are defined in `src/app/api/chat/route.ts` and passed to `streamText`. Max 40 tool-use steps per turn (4 for mock).

### State Management

Two React contexts carry all cross-component state:

| Context | Location | Owns |
|---|---|---|
| `FileSystemContext` | `src/lib/contexts/file-system-context.tsx` | Virtual FS, tool-call processing |
| `ChatContext` | `src/lib/contexts/chat-context.tsx` | Messages, AI stream, anonymous-user localStorage |

### Authentication

JWT sessions (7-day, httpOnly cookie). `src/lib/auth.ts` issues/verifies tokens; `src/actions/index.ts` has `signUp`/`signIn`/`signOut`/`getUser` server actions. `src/middleware.ts` guards `/api/projects` and `/api/filesystem`. Passwords are bcrypt-hashed; minimum 8 characters.

### Database

Prisma with SQLite (`prisma/dev.db`). Two models: `User` (email/password) and `Project` (name, JSON messages, JSON FS data, optional userId for anonymous projects). The database schema is defined in `prisma/schema.prisma` — reference it anytime you need to understand the structure of data stored in the database.

### UI Layout

Three resizable panels rendered by `src/app/main-content.tsx`:
- **Left (35%)** — `ChatInterface` (message list + input)
- **Right (65%)** — tabbed `PreviewFrame` (default) / `CodeEditor` + `FileTree`

UI components are shadcn/ui (New York style, neutral palette, CSS variables). Path alias `@/*` → `src/*`.

## Key Conventions

- The generation system prompt (`src/lib/prompts/generation.tsx`) requires Tailwind CSS for all styling — no inline styles or CSS files.
- All new React components must be importable via `@/` (the virtual FS path alias), not relative paths.
- Prompt caching (`ephemeral` cache control) is applied to the system prompt in the chat route — preserve this when editing `src/app/api/chat/route.ts`.
- Test files live in `__tests__/` subdirectories next to the component they test.
