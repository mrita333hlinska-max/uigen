# AGENTS.md

## Project overview

AI-powered React component generator with live preview. See [README](./README.md).

- **Stack**: Next.js 15 (App Router), React 19, TypeScript, Tailwind CSS v4, Prisma/SQLite, Vercel AI SDK + Claude
- **Model**: `claude-haiku-4-5` via `@ai-sdk/anthropic`. Falls back to a mock provider when `ANTHROPIC_API_KEY` is unset.

## Build and test

```bash
npm run setup          # first-time: install + prisma generate + migrate
npm run dev            # development server (Turbopack)
next build             # production build
vitest                 # run tests
npx prisma migrate dev # apply schema changes
npm run db:reset       # wipe and re-migrate (destructive)
```

> **Never run `npm audit fix`** — pinned dependency versions are intentional; audit fix breaks compatibility.

`node-compat.cjs` deletes `globalThis.localStorage/sessionStorage` on Node 25+ to prevent SSR crashes. It is imported at the top of `next.config.ts` and must not be removed.

## Environment variables

| Variable            | Required       | Notes                                                                                  |
| ------------------- | -------------- | -------------------------------------------------------------------------------------- |
| `ANTHROPIC_API_KEY` | No             | Missing or placeholder → mock provider with canned responses (no real AI calls)        |
| `JWT_SECRET`        | **Yes (prod)** | Defaults to `development-secret-key` — always override in production                   |
| `DATABASE_URL`      | Auto           | SQLite `file:./dev.db` via Prisma; relative path breaks on Vercel (needs blob storage) |

Create a `.env.local` (gitignored) with at minimum `ANTHROPIC_API_KEY` and `JWT_SECRET`.

## Architecture

```
src/
  app/            # Next.js routes + API
    api/chat/     # AI streaming endpoint (POST)
    [projectId]/  # Project page (server component)
  actions/        # Server actions ("use server") — auth + project CRUD
  components/     # React UI components
  lib/
    file-system.ts          # In-memory VirtualFileSystem class
    auth.ts                 # JWT session helpers
    provider.ts             # AI model selection (real vs mock)
    contexts/               # Client state: FileSystemProvider, ChatProvider
    tools/                  # Tool definitions exposed to Claude
    prompts/generation.tsx  # System prompt for component generation
    transform/              # JSX → browser-runnable code transformer
prisma/schema.prisma        # User + Project models (SQLite)
```

## Key conventions

### Client / server boundary

- Mark files `"use client"` for any component using state, effects, or context.
- Server actions live in `src/actions/` — never call them from server components that could short-circuit auth checks.
- Always call `getSession()` in server actions that mutate data; anonymous users have `userId: null`.

### State management

- **FileSystemProvider** (`src/lib/contexts/file-system-context.tsx`) holds the in-memory VFS and selected file. Always call `triggerRefresh()` after VFS mutations so the UI updates.
- **ChatProvider** (`src/lib/contexts/chat-context.tsx`) wraps `useChat()` (Vercel AI SDK); handles tool call results and syncs file changes back to the VFS.

### Virtual file system

- In-memory only — no disk I/O. Implemented as a `Map`-based tree in `VirtualFileSystem`.
- All paths are virtual, root is `/`, and `App.jsx` is the entrypoint Claude must create.
- `serialize()` converts Maps → plain objects for JSON; `deserializeFromNodes()` does the reverse. Children are `Map<string, FileNode>` in memory but plain objects in JSON — don't assume the type after deserialization.
- **Server VFS and client VFS are isolated.** Tools mutate the server VFS during streaming; `ChatProvider.onToolCall` replays those changes on the client VFS. The server VFS is never sent to the client — only the final serialized state is saved to DB in `onFinish`.
- When deserializing, paths must be sorted so parents are created before children.

### AI tool calls

- Two tools: `str_replace_editor` (view / create / str_replace / insert) and `file_manager` (rename / delete).
- Tools run server-side in `src/app/api/chat/route.ts` and results stream to the client via `onToolCall` in ChatProvider.
- `messages` and `data` (VFS snapshot) are stored as JSON strings in the `Project` row (not JSON columns); always `JSON.stringify`/`JSON.parse` explicitly.
- `maxDuration = 120` seconds; `maxSteps = 40` (real model), `4` (mock).
- DB is only written in `onFinish` if the user is authenticated and a `projectId` is provided. Anonymous work is ephemeral (lost on page reload).

### Preview / transform pipeline

`src/lib/transform/jsx-transformer.ts` converts the in-memory VFS into a browser-runnable preview:

1. **`transformJSX`** — Babel standalone strips TypeScript and converts JSX to JS; CSS imports are removed.
2. **`createImportMap`** — Builds Blob URLs for every file, maps all path aliases (`@/Foo`, `/Foo.jsx`, `Foo`, …). Third-party packages resolve via `https://esm.sh/<pkg>` (requires internet). Missing imports become silent placeholder components.
3. **`createPreviewHTML`** — Embeds Tailwind CDN, collected styles, and the import map. Dynamically imports `/App.jsx` and mounts it to `#root`. Errors are shown in an inline error boundary.

Blob URLs are iframe-session-specific — they can't be shared across page reloads.

### Anonymous users

- Anonymous users can generate components; `/api/chat` skips the DB save when no `projectId` is supplied.
- `src/lib/anon-work-tracker.ts` persists work to `sessionStorage` so the UI can prompt sign-up. This data is cleared on tab close and is never written to the DB.

### Styling

- Tailwind CSS v4 exclusively — no inline styles or CSS modules.
- Radix UI primitives in `src/components/ui/` for accessible widgets.
- Use `cn()` from `@/lib/utils` for conditional class merging.

### Imports

- Use `@/` alias for all non-library imports (e.g., `@/lib/file-system`, `@/components/chat/ChatInterface`).

## Testing

- Framework: Vitest + React Testing Library (jsdom environment).
- Tests live in `__tests__/` subdirectories next to source files.
- Run a single test file: `vitest src/components/chat/__tests__/ChatInterface.test.tsx`

## Key files

| File                                                                                 | Purpose                                                            |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| [src/lib/file-system.ts](src/lib/file-system.ts)                                     | VirtualFileSystem class — core in-memory state                     |
| [src/app/api/chat/route.ts](src/app/api/chat/route.ts)                               | AI streaming, tool setup, DB save on finish                        |
| [src/lib/contexts/file-system-context.tsx](src/lib/contexts/file-system-context.tsx) | Client VFS state + refresh trigger                                 |
| [src/lib/contexts/chat-context.tsx](src/lib/contexts/chat-context.tsx)               | `useChat()` integration, tool call handler                         |
| [src/lib/prompts/generation.tsx](src/lib/prompts/generation.tsx)                     | System prompt — React/Tailwind expectations, `/App.jsx` entrypoint |
| [src/lib/tools/str-replace.ts](src/lib/tools/str-replace.ts)                         | `str_replace_editor` tool definition                               |
| [src/lib/auth.ts](src/lib/auth.ts)                                                   | JWT creation/verification, httpOnly cookie                         |
| [prisma/schema.prisma](prisma/schema.prisma)                                         | User + Project schema                                              |
