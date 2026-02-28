# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack
npm run dev:daemon   # Start dev server as daemon (logs to logs.txt)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all Vitest tests
npm run db:reset     # Reset SQLite database (--force)
```

To run a single test file:
```bash
npx vitest run src/path/to/file.test.ts
```

Environment: set `ANTHROPIC_API_KEY` in `.env`. Without it, the app uses a `MockLanguageModel` that generates static demo components.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in natural language; Claude generates code that is rendered live in a sandboxed iframe.

### Request Flow

1. User submits prompt in `ChatInterface`
2. `ChatProvider` (`/src/lib/contexts/chat-context.tsx`) calls `/api/chat` via Vercel AI SDK's `useChat`
3. `/src/app/api/chat/route.ts` streams Claude's response with tool calls
4. Tool calls (`str_replace_editor`, `file_manager`) are executed by handlers in `FileSystemContext`
5. `FileSystemContext` (`/src/lib/contexts/file-system-context.tsx`) updates in-memory virtual FS
6. `PreviewFrame` (`/src/components/preview/PreviewFrame.tsx`) re-renders the iframe using Babel WASM + import maps
7. On conversation end, project is serialized and saved to Prisma (authenticated users only)

### Virtual Filesystem

`VirtualFileSystem` (`/src/lib/file-system.ts`) is an in-memory structure — no disk writes. Claude's tools operate on it, and `FileSystemContext` holds the React state. The FS serializes to JSON for Prisma persistence and for sending file contents to Claude as context.

### AI Tools

Defined in `/src/lib/tools/`:
- `str_replace_editor` — create and edit files (str-replace or full write)
- `file_manager` — rename and delete files

The system prompt (`/src/lib/prompts/generation.tsx`) instructs Claude to use `@/` import aliases, keep `/App.jsx` as the root entry, and prefer Tailwind CSS.

### Preview Rendering

`PreviewFrame` builds a full HTML document injected into an `<iframe>`. It uses:
- Babel standalone (WASM) to transform JSX in-browser
- Import maps to resolve `react`, `react-dom`, and `@/` aliases to CDN/blob URLs
- The virtual FS files are bundled as blob URLs and included in the import map

### Authentication

Optional email/password auth using bcrypt + JWT (jose). Tokens are stored in httpOnly cookies with 7-day expiration. Anonymous users can generate components but projects aren't persisted. Protected API routes are handled by `/src/middleware.ts`.

### State Management

Two React contexts manage global state:
- `ChatContext` — chat messages, loading state, `useChat` integration
- `FileSystemContext` — virtual FS state, tool call handlers, project load/save

### Key Paths

| Path | Role |
|------|------|
| `src/app/api/chat/route.ts` | Streaming AI endpoint |
| `src/lib/contexts/` | Global state (chat + filesystem) |
| `src/lib/file-system.ts` | VirtualFileSystem class |
| `src/lib/provider.ts` | Claude model or MockLanguageModel |
| `src/lib/prompts/generation.tsx` | System prompt for component generation |
| `src/lib/tools/` | AI tool definitions |
| `src/components/preview/PreviewFrame.tsx` | Iframe-based live preview |
| `src/app/main-content.tsx` | Three-panel layout (chat | preview | code) |
| `prisma/schema.prisma` | SQLite schema (User, Project) — see below |

## Database Schema

```prisma
model User {
  id        String    @id @default(cuid())
  email     String    @unique
  password  String
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  projects  Project[]
}

model Project {
  id        String   @id @default(cuid())
  name      String
  userId    String?
  messages  String   @default("[]")   // JSON-serialized chat messages
  data      String   @default("{}")   // JSON-serialized virtual filesystem
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  user      User?    @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

`userId` is nullable — projects can exist without an authenticated user (anonymous sessions).

## Code Style

- Only comment complex or non-obvious logic. Don't restate what the code already makes clear.

## Tech Stack

- **Framework**: Next.js 15 (App Router), React 19, TypeScript
- **AI**: Anthropic Claude via `@ai-sdk/anthropic` + Vercel AI SDK streaming
- **Styling**: Tailwind CSS v4, Radix UI, shadcn/ui
- **Database**: Prisma 6 + SQLite (`dev.db`)
- **Testing**: Vitest 3 + jsdom + Testing Library
