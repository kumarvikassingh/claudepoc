# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

UIGen is an AI-powered React component generator with live preview. The user describes a component in chat; Claude (via the Vercel AI SDK) generates the code by calling file-editing tools. Generated files live in an **in-memory virtual file system** — nothing is written to disk. Files are transformed in-browser with Babel standalone and rendered in a sandboxed iframe.

## Commands

```bash
npm run setup        # install deps, generate Prisma client, run migrations (run once)
npm run dev          # dev server with Turbopack at http://localhost:3000
npm run dev:daemon   # dev server in background, logs to logs.txt
npm run build        # production build
npm run lint         # next lint
npm test             # run vitest once (use `npx vitest` for watch mode)
npm run db:reset     # wipe and recreate the SQLite database
```

Run a single test file: `npx vitest run src/lib/__tests__/file-system.test.ts`
Run tests matching a name: `npx vitest run -t "creates a file"`

Do **not** run `npm audit fix` — dependencies are deliberately pinned and it will break the app.

## Environment

- Set `ANTHROPIC_API_KEY` in `.env` to use the real Claude API. **Without a key (or with the placeholder `your-api-key-here`), the app silently falls back to `MockLanguageModel`** (`src/lib/provider.ts`), which returns canned counter/form/card components. This is the default dev experience — tests and local runs work without a key.
- Model is hardcoded to `claude-haiku-4-5` in `src/lib/provider.ts`.

## Architecture

### The virtual file system is the core abstraction
`src/lib/file-system.ts` (`VirtualFileSystem`) is an in-memory tree of `FileNode`s keyed by absolute path. There is no disk I/O for generated components. The same class runs in two places:
- **Server** (`src/app/api/chat/route.ts`): reconstructed per request via `deserializeFromNodes(files)`, mutated by tool calls, then re-serialized.
- **Client** (`src/lib/contexts/file-system-context.tsx`): the live instance backing the editor and preview.

### The AI loop and how files get created
1. `ChatProvider` (`src/lib/contexts/chat-context.tsx`) wraps the AI SDK's `useChat`. On every request it sends the **client's** serialized file system in the request `body`.
2. The chat route (`/api/chat`) prepends the system prompt (`src/lib/prompts/generation.tsx`), reconstructs the FS, and runs `streamText` with two tools:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — view/create/str_replace/insert/undo_edit
   - `file_manager` (`src/lib/tools/file-manager.ts`) — rename/delete
   These tools mutate the **server-side** FS.
3. As tool calls stream back, `useChat`'s `onToolCall` fires `handleToolCall` in the file-system context, which **replays the same mutations on the client FS**. This dual mutation (server + client replay) is how the two copies stay in sync — keep them consistent if you change tool behavior.
4. `onFinish` persists messages + serialized FS to the `Project` row, but **only for authenticated users with a `projectId`**.

### Preview rendering
`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) + `src/lib/transform/jsx-transformer.ts` compile JSX/TSX with `@babel/standalone` in the browser, build an import map (resolving the `@/` alias and stubbing missing imports with placeholder modules), and inject the result into an iframe with Tailwind via CDN. Entry point is `/App.jsx` (falls back through `/index.jsx`, `/src/App.jsx`, etc.).

### Import alias convention
Generated components and all source use `@/` → `src/` (see `tsconfig.json` paths). The generation prompt instructs Claude to use `@/` for non-library imports; the JSX transformer resolves it at preview time.

### Auth and persistence
- JWT sessions in an httpOnly cookie via `jose` (`src/lib/auth.ts`). `getSession`/`createSession`/`deleteSession` for server components/actions; `verifySession` for `src/middleware.ts` (gates `/api/projects`, `/api/filesystem`).
- Prisma + SQLite. Two models only (`prisma/schema.prisma`): `User` and `Project`. A project stores `messages` and `data` (the serialized FS) as JSON strings. `userId` is nullable.
- **Prisma client is generated to `src/generated/prisma`** (not `node_modules`). Regenerate with `npx prisma generate` after schema changes.
- Server actions live in `src/actions/` (`createProject`, `getProjects`, `getProject`, `getUser`).

### Anonymous → authenticated flow
Anonymous users have no `projectId`, so work isn't persisted server-side. `src/lib/anon-work-tracker.ts` stashes messages + FS in `sessionStorage`. On sign-in/sign-up, that work is converted into a real project. `src/app/page.tsx` redirects authenticated users to their most recent project (creating one if none exists).

## Testing notes
- Vitest + Testing Library, jsdom environment (`vitest.config.mts`). Tests live in `__tests__` folders next to the code they cover.
- `vite-tsconfig-paths` resolves the `@/` alias in tests, so imports match runtime.
