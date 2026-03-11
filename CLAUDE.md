# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # Initial setup: install deps + prisma generate + migrate
npm run dev          # Start dev server with Turbopack (localhost:3000)
npm run dev:daemon   # Start dev server in background, logs to logs.txt
npm run build        # Production build
npm run lint         # Run ESLint
npm test             # Run all Vitest tests
npm run db:reset     # Reset database (--force flag, destructive)
```

All npm scripts use `NODE_OPTIONS='--require ./node-compat.cjs'` for Node compatibility — don't remove this when adding scripts.

To run a single test file: `npx vitest run src/lib/transform/__tests__/jsx-transformer.test.ts`

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe what they want in chat; Claude generates/edits files in a virtual file system; the preview iframe renders the result in real time.

### Core Data Flow

```
User chat input
  → ChatContext (wraps Vercel AI SDK useChat)
  → POST /api/chat/route.ts
  → Claude with tools (str_replace_editor, file_manager)
  → Tool calls update FileSystemContext
  → JSX transformer compiles files
  → PreviewFrame re-renders iframe
```

### Virtual File System (VFS)

`/src/lib/file-system.ts` — In-memory tree with Map-based storage. No files are written to disk. The VFS is serialized to JSON for SQLite persistence. The AI always targets `/App.jsx` as the root entry point. Supports `createFile`, `updateFile`, `deleteFile`, `rename`, `serialize`/`deserialize`.

### State Management

Two primary React contexts (no Redux/Zustand):

- **FileSystemContext** (`/src/lib/contexts/file-system-context.tsx`): VFS state, selected file, processes AI tool call results
- **ChatContext** (`/src/lib/contexts/chat-context.tsx`): Messages, streaming state, bridges AI responses to VFS updates

### AI Integration

- Model: `claude-haiku-4-5` via `@ai-sdk/anthropic` (Vercel AI SDK streaming)
- Falls back to `MockLanguageModel` when `ANTHROPIC_API_KEY` is not set
- Two tools exposed to Claude: `str_replace_editor` (create/view/edit files) and `file_manager` (rename/delete)
- System prompt lives in `/src/lib/prompts/generation.tsx`
- Prompt caching enabled (Anthropic ephemeral cache)

### Preview Rendering

`/src/lib/transform/jsx-transformer.ts` — Babel standalone compiles JSX to JS. `/src/components/preview/PreviewFrame.tsx` wraps compiled output in blob URLs with an import map pointing to esm.sh CDN for React and third-party packages, then injects into a sandboxed `<iframe srcdoc>`.

### Authentication

JWT sessions via JOSE, stored as HttpOnly cookies (7-day expiry). Password hashing with bcrypt. Server actions in `/src/actions/` handle `signUp`, `signIn`, `signOut`, `getUser`. Middleware at `/src/middleware.ts` guards protected routes.

Anonymous users get localStorage persistence via `/src/lib/anon-work-tracker.ts`; authenticated users get full SQLite persistence.

### Database

SQLite via Prisma (`prisma/schema.prisma`). Two models: `User` (email/password) and `Project` (messages stored as JSON string, VFS state stored as JSON string in `data` field). Prisma client is generated to `/src/generated/prisma`.

### UI Layout

`/src/app/main-content.tsx` — Resizable three-panel layout: Chat (left, 35%), Preview (top-right), Code Editor with FileTree (bottom-right). Uses shadcn/ui (New York style, Neutral base) with Radix UI primitives. Monaco Editor for code editing.

### Path Aliases

`@/*` maps to `./src/*` in TypeScript and throughout the codebase.

## Key Files

| File | Purpose |
|------|---------|
| `src/app/api/chat/route.ts` | Streaming AI endpoint, tool definitions |
| `src/lib/file-system.ts` | VFS implementation |
| `src/lib/contexts/file-system-context.tsx` | VFS React state |
| `src/lib/contexts/chat-context.tsx` | Chat + AI SDK state |
| `src/lib/provider.ts` | AI model selection (real vs mock) |
| `src/lib/transform/jsx-transformer.ts` | Babel JSX compilation |
| `src/lib/prompts/generation.tsx` | Claude system prompt |
| `src/components/preview/PreviewFrame.tsx` | Iframe renderer |
| `prisma/schema.prisma` | DB schema |

## Environment

```
ANTHROPIC_API_KEY=   # Optional; mock provider used if absent
JWT_SECRET=          # Defaults to insecure dev value if absent
```
