# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run tests with Vitest
npx vitest run src/lib/__tests__/some-file.test.ts  # Run a single test file
npm run db:reset     # Reset and re-seed the SQLite database
```

**Environment:** Copy `.env` and set `ANTHROPIC_API_KEY`. Without it, the app falls back to a `MockLanguageModel` that returns pre-built example components.

## Architecture

**UIGen** is a Next.js 15 App Router app that lets users prompt Claude to generate React components, with live preview in an iframe.

### Request Flow

1. User types a prompt in `ChatInterface` → POST to `/api/chat`
2. `route.ts` calls Claude via Vercel AI SDK `streamText`, streaming tool calls back
3. Claude calls `str_replace_editor` (create/view/edit files) and `file_manager` (file ops) — defined in `src/lib/tools/`
4. Tool calls mutate an in-memory `VirtualFileSystem` instance
5. File system state propagates via `FileSystemContext` to the preview iframe
6. `jsx-transformer.ts` compiles JSX in the browser for live preview
7. If user is authenticated, messages and file system are persisted to SQLite via Prisma

### Virtual File System

`src/lib/file-system.ts` is a central abstraction — an in-memory tree (no disk I/O). It is serialized to JSON and stored in `Project.data` in the database. The AI tools operate entirely on this virtual FS. The iframe preview reads from it via `FileSystemContext`.

### AI Provider

`src/lib/provider.ts` exports the language model. It uses Claude 3.7 Sonnet via `@ai-sdk/anthropic`. When `ANTHROPIC_API_KEY` is absent, it returns a `MockLanguageModel`. The system prompt (inside `route.ts`) instructs Claude to:
- Always expose `/App.jsx` as the entry point
- Use Tailwind CSS for styling
- Use `@/` as the import alias for local files

### Auth

JWT-based sessions (via `jose`) stored in cookies. Passwords hashed with `bcrypt`. Server actions in `src/actions/` handle sign-up, sign-in, sign-out, and project CRUD. Projects can be owned by a user or be anonymous (`userId` is optional).

### UI Layout

Three resizable panels (via `react-resizable-panels`):
- **Left (35%)**: Chat — `ChatInterface` + `ChatProvider` context
- **Right top**: Live preview iframe — `PreviewFrame`
- **Right bottom**: Code view — `FileTree` + Monaco `CodeEditor`

### Key Tech

- **Next.js 15** App Router, React 19, TypeScript
- **Vercel AI SDK** (`ai` + `@ai-sdk/anthropic`) for streaming + tool calling
- **Prisma** + SQLite for persistence
- **Tailwind CSS v4** for styling
- **Monaco Editor** for code editing
- **Vitest** + Testing Library for tests
