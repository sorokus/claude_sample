# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack on port 3000
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run all Vitest unit tests
npm run db:reset     # Reset SQLite database to initial state
```

No command to run a single test file — use `npx vitest run src/path/to/test.ts` directly.

## Environment

Create a `.env` file with:
```
ANTHROPIC_API_KEY=""  # Optional — leave blank to use mock mode with pre-built component examples
```

Without `ANTHROPIC_API_KEY`, the app runs with a `MockLanguageModel` (`src/lib/provider.ts`) that returns static Counter/Form/Card components. This is useful for UI/UX development without API costs.

## Architecture

**UIGen** is an AI-powered React component generator. Users describe components in a chat interface; Claude generates them in a virtual file system with live preview.

### Request Flow

```
User chat message
  → POST /api/chat (streaming, src/app/api/chat/route.ts)
  → streamText() with Claude or Mock model
  → AI calls tools: str_replace_editor / file_manager
  → onToolCall() in ChatProvider
  → FileSystemContext.handleToolCall() updates VirtualFileSystem
  → FileTree + CodeEditor reflect changes
  → PreviewFrame re-renders component
  → On stream finish: serialize and persist to Prisma (authenticated users only)
```

### Key Subsystems

**AI Layer** (`src/lib/provider.ts`, `src/app/api/chat/route.ts`, `src/lib/tools/`, `src/lib/prompts/`)
- `getLanguageModel()` returns real Claude Haiku 4.5 or MockLanguageModel based on env
- Two tools exposed to the AI: `str_replace_editor` (create/view/edit files via string replacement) and `file_manager` (rename/delete/list)
- Generation system prompt is in `src/lib/prompts/generation.tsx`

**Virtual File System** (`src/lib/file-system.ts`, `src/lib/contexts/file-system-context.tsx`)
- `VirtualFileSystem` class manages in-memory files/directories with `serialize()`/`deserialize()` for persistence
- `FileSystemContext` wraps it in React state and routes tool calls from the AI to the VFS

**Chat State** (`src/lib/contexts/chat-context.tsx`)
- Uses Vercel AI SDK's `useChat` hook
- `onToolCall` callback bridges AI tool calls to `FileSystemContext`
- Anonymous user work is tracked via `src/lib/anon-work-tracker.ts`

**Auth** (`src/lib/auth.ts`, `src/actions/index.ts`, `src/middleware.ts`)
- Custom JWT sessions stored in HTTP-only cookies (7-day expiry) using `jose`
- Passwords hashed with bcrypt
- Server Actions for signUp/signIn/signOut/getUser
- Middleware protects `/[projectId]` routes

**Database** (`prisma/schema.prisma`)
- SQLite with two models: `User` (email/password) and `Project` (name, messages JSON, data JSON)
- `data` field stores serialized VirtualFileSystem; `messages` stores chat history

### UI Layout (`src/app/main-content.tsx`)

Three-panel resizable layout:
1. Left (35%): Chat (`src/components/chat/`)
2. Right (65%): Tabs between Preview (`src/components/preview/`) and Code view
   - Code view splits into FileTree (30%) and Monaco CodeEditor (70%)

### JSX Transformation (`src/lib/transform/jsx-transformer.ts`)

Generated components are transformed before rendering in the preview iframe — this handles import rewriting and makes components renderable in the browser sandbox.
