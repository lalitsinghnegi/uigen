# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Code Style

Only add comments when a code block is genuinely complex and non-obvious. Skip comments for straightforward code.

## Commands

```bash
npm run setup                                              # install, generate Prisma client, migrate
npm run dev                                                # dev server (Turbopack)
npm run build
npm lint
npm test
npx vitest run src/lib/__tests__/file-system.test.ts      # single test file
npm run db:reset
```

Tests use Vitest with jsdom and `@testing-library/react`. Test files live in `__tests__/` subdirectories next to the code they test.

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY`. Without it, the app falls back to `MockLanguageModel` in `src/lib/provider.ts`, which returns static component templates instead of calling Claude.

## Architecture

### Overview
UIGen is a Next.js 15 (App Router) app where users describe React components in a chat and the AI generates them with live preview.

### AI / Chat pipeline (`src/app/api/chat/route.ts`)
- POST handler receives `{ messages, files, projectId }` from the client
- Reconstructs a `VirtualFileSystem` from the serialized `files`
- Calls `streamText` (Vercel AI SDK) with two tools exposed to the model: `str_replace_editor` and `file_manager`
- The model uses these tools to create/edit files in the virtual FS during streaming
- On finish, if authenticated, saves messages + file system state to the `Project` DB record

### Virtual File System (`src/lib/file-system.ts`)
All generated code lives in memory—nothing is written to disk. `VirtualFileSystem` is an in-memory tree of `FileNode` objects. It supports standard CRUD plus `serialize()` / `deserializeFromNodes()` for JSON round-tripping (stored as `data` JSON column in SQLite).

### AI Tools (`src/lib/tools/`)
- `str_replace_editor` — create/view/str_replace/insert on virtual files (mirrors Anthropic's text editor tool)
- `file_manager` — rename/delete files in the virtual FS

### Preview pipeline (`src/lib/transform/jsx-transformer.ts` → `src/components/preview/PreviewFrame.tsx`)
1. All virtual files are read from the FS
2. `createImportMap()` transpiles each `.jsx/.tsx` file via `@babel/standalone` to JS, creates `blob:` URLs, and builds a browser import map
3. Third-party imports are proxied through `https://esm.sh/`; missing local imports get placeholder stub modules
4. `createPreviewHTML()` generates an `srcdoc` HTML page that uses the import map to dynamically `import()` the entry point (`/App.jsx` by default) and mounts it with `ReactDOM.createRoot`
5. `PreviewFrame` renders an `<iframe>` whose `srcdoc` is updated whenever `refreshTrigger` changes

### React Context layer
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — wraps `VirtualFileSystem`, exposes CRUD hooks, and handles `handleToolCall` to apply model tool-call results to the FS
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK's `useChat`, wires `onToolCall` to `handleToolCall`, serializes the current FS into every request body

### Auth (`src/lib/auth.ts`)
JWT-based, stored in an `httpOnly` cookie (`auth-token`). Uses `jose` for sign/verify. Falls back gracefully—unauthenticated users can still generate components but they aren't persisted.

### Database (Prisma + SQLite)
- Schema: `User` (email/password) and `Project` (userId optional, `messages` JSON string, `data` JSON string)
- Prisma client output is in `src/generated/prisma`
- `messages` stores serialized Vercel AI SDK `Message[]`; `data` stores the serialized `VirtualFileSystem`

### Model selection (`src/lib/provider.ts`)
- Real: `claude-haiku-4-5` via `@ai-sdk/anthropic`
- Mock: `MockLanguageModel` (same interface) when no API key is set; produces deterministic counter/form/card components for development/testing
