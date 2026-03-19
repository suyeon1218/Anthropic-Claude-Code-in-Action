# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + DB migrations)
npm run setup

# Development server (uses Turbopack)
npm run dev

# Build for production
npm run build

# Lint
npm run lint

# Run all tests
npm run test

# Reset database
npm run db:reset

# Prisma migrations
npx prisma migrate dev
npx prisma generate
```

Tests use Vitest. Run a single test file:
```bash
npx vitest src/lib/__tests__/file-system.test.ts
```

## Environment

Create `.env` with:
```
ANTHROPIC_API_KEY=sk-...   # Optional — falls back to mock responses if missing
```

Without `ANTHROPIC_API_KEY`, the app uses `MockLanguageModel` (static responses). With it, uses Claude Haiku 4.5.

## Architecture

**UIGen** is an AI-powered React component generator with live preview. Users describe components in chat, Claude generates them, and they are rendered live in an iframe sandbox — all without touching the disk.

### Three-Panel Layout (`src/app/main-content.tsx`)
- **Left (35%)**: Chat interface — Vercel AI SDK `useChat` hook streaming from `/api/chat`
- **Right (65%)**: Preview/Code tabs
  - Preview: `PreviewFrame` — Babel-compiled JSX rendered in a sandboxed iframe
  - Code: `FileTree` + Monaco editor

### Virtual File System (`src/lib/file-system.ts`)
- In-memory tree — no disk I/O. Serializes to JSON for DB persistence.
- Every project has `/App.jsx` as the required root entry point.
- Synced via `FileSystemContext` (React context) wrapping the three-panel UI.
- Tools (`str_replace_editor`, `file_manager`) manipulate the VFS during AI generation.

### AI Integration (`src/app/api/chat/route.ts`)
- Uses `@ai-sdk/anthropic` + Vercel AI SDK `streamText()`
- On stream finish: chat history + VFS state are persisted to the DB (`Project.messages`, `Project.data`)
- Two tools registered: `str_replace_editor` (view/create/edit files) and `file_manager`
- Language model selected in `src/lib/provider.ts` based on env var

### Authentication (`src/lib/auth.ts`)
- JWT (jose) with 7-day expiry, stored in httpOnly cookie
- Server Actions for auth operations: `src/actions/index.ts`
- Middleware protects `/api/projects` and `/api/filesystem` routes
- Anonymous users get sessionStorage-persisted work that migrates to a project on sign-in (`src/hooks/use-auth.ts`)

### Database (`prisma/schema.prisma`)
- SQLite in development (`prisma/dev.db`)
- Two models: `User` (email/password/projects) and `Project` (name, `messages` JSON, `data` JSON)
- Prisma client singleton at `src/lib/prisma.ts`
- Generated types at `src/generated/prisma/`

### JSX Preview Pipeline (`src/lib/transform/jsx-transformer.ts`)
- Takes VFS snapshot → compiles JSX with Babel standalone → injects into iframe
- Uses an ImportMap for module resolution inside the sandbox

### Key Contexts
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK `useChat`
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — manages VFS state

## Testing

Vitest + jsdom + `@testing-library/react`. Test files live in `__tests__/` subdirectories next to the code they test.

Tested areas: `VirtualFileSystem`, chat/editor components, JSX transformer, contexts.
