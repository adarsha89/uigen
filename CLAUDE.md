# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # Initial setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run Vitest test suite
npm run db:reset     # Reset database (destructive)
```

Environment: copy `.env.example` to `.env.local` and set `ANTHROPIC_API_KEY`.

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface; Claude generates JSX using tool calls that manipulate a virtual file system; the result renders live in an iframe.

### Request Flow

1. User submits a prompt in `ChatInterface`
2. `chat-context.tsx` sends it to `/api/chat` via Vercel AI SDK streaming
3. The API route (`/src/app/api/chat/route.ts`) calls Claude with two tools:
   - `str_replace_editor` (`/src/lib/tools/str-replace.ts`) — create/view/edit files
   - `file_manager` (`/src/lib/tools/file-manager.ts`) — rename/delete files
4. Tool calls mutate the virtual file system (`/src/lib/file-system.ts`)
5. File system state is held in `FileSystemContext` (`/src/lib/contexts/file-system-context.tsx`)
6. `PreviewFrame` picks up changes, transforms JSX via Babel standalone (`/src/lib/transform/jsx-transformer.ts`), and renders in an iframe

### Key Layers

| Layer | Location | Purpose |
|---|---|---|
| UI | `/src/components/` | Chat, Monaco editor, preview iframe, auth dialogs |
| State | `/src/lib/contexts/` | `FileSystemContext` and `ChatContext` via React Context |
| AI tools | `/src/lib/tools/` | Tool definitions given to Claude for file manipulation |
| Transform | `/src/lib/transform/` | Babel-based browser JSX compilation |
| Backend | `/src/actions/`, `/src/app/api/` | Server actions (auth, projects) and streaming chat API |
| Database | `/prisma/schema.prisma` | SQLite via Prisma — User and Project models |
| Auth | `/src/lib/auth.ts` | JWT sessions stored in cookies (Jose) |

### Virtual File System

The virtual FS is entirely in-memory (no disk I/O on the client). `file-system.ts` exposes CRUD operations; the context wraps it with React state. The AI's tool calls (`str_replace_editor`, `file_manager`) are the only way generated code enters the FS. File tree is rendered by `/src/components/editor/FileTree.tsx`.

### AI Provider

`/src/lib/provider.ts` abstracts the model. It returns a real `@ai-sdk/anthropic` model in production and a mock model in test environments. The system prompt lives in `/src/lib/prompts/generation.tsx`.

### Authentication

Anonymous users can generate components (work tracked via `anon-work-tracker.ts`). Authenticated users get project persistence. Auth uses JWT cookies; `src/middleware.ts` protects routes.

### Path Alias

`@/*` maps to `src/*` (configured in `tsconfig.json`).

## Code Style

Use comments sparingly. Only add comments for complex code.
