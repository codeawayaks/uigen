# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface, and the AI (Claude via Vercel AI SDK) generates React code that renders in a live preview panel. Components live in a virtual file system (never written to disk).

## Commands

- **Setup**: `npm run setup` (installs deps, generates Prisma client, runs migrations)
- **Dev server**: `npm run dev` (Next.js with Turbopack, requires `--require ./node-compat.cjs`)
- **Build**: `npm run build`
- **Lint**: `npm run lint`
- **Tests**: `npm run test` (vitest)
- **Single test**: `npx vitest run src/lib/__tests__/file-system.test.ts`
- **DB reset**: `npm run db:reset`

## Tech Stack

- Next.js 15 (App Router) / React 19 / TypeScript
- Tailwind CSS v4
- Prisma with SQLite (`prisma/dev.db`)
- Vercel AI SDK (`ai` package) + `@ai-sdk/anthropic`
- UI components: Radix UI primitives via shadcn/ui (configured in `components.json`)
- Testing: Vitest + React Testing Library + jsdom

## Architecture

### Core Data Flow

1. User sends a chat message -> `ChatProvider` (uses `@ai-sdk/react` `useChat`) calls `POST /api/chat`
2. The API route streams responses via `streamText()` with two AI tools: `str_replace_editor` and `file_manager`
3. Tool calls modify a `VirtualFileSystem` instance on the server; the client mirrors changes via `onToolCall` in `ChatProvider` -> `FileSystemContext.handleToolCall`
4. `PreviewFrame` watches for file changes, transforms JSX via `@babel/standalone` in the browser, builds an import map with blob URLs, and renders in a sandboxed iframe

### Key Abstractions

- **VirtualFileSystem** (`src/lib/file-system.ts`): In-memory file system with path-based CRUD, serialization, and text-editor operations (view, replace, insert). Used both server-side (in the API route) and client-side (via context).
- **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`): React context wrapping VirtualFileSystem. Handles AI tool call dispatch (`str_replace_editor` commands: create/str_replace/insert; `file_manager` commands: rename/delete).
- **ChatContext** (`src/lib/contexts/chat-context.tsx`): Wraps `useChat` from AI SDK, wires tool calls to FileSystemContext, tracks anonymous work.
- **JSX Transformer** (`src/lib/transform/jsx-transformer.ts`): Browser-side Babel transform. Creates import maps mapping local files to blob URLs, resolves `@/` aliases, handles CSS imports, creates placeholder modules for missing imports, and proxies third-party packages via esm.sh.

### Mock Provider

When `ANTHROPIC_API_KEY` is not set, `MockLanguageModel` in `src/lib/provider.ts` returns static component code (counter/form/card) so the app runs without an API key.

### Layout

The main UI (`src/app/main-content.tsx`) is a resizable two-panel layout: chat on the left, preview/code editor on the right. The code view has a nested file tree + Monaco editor.

### Auth & Persistence

JWT-based auth via `jose` (not NextAuth). Session tokens stored in cookies. Authenticated users get projects persisted to SQLite via Prisma. Anonymous users work in-memory with local tracking (`anon-work-tracker`).

### Prisma

- Schema: `prisma/schema.prisma`
- Generated client output: `src/generated/prisma`
- Models: `User` (email/password) and `Project` (name, messages as JSON string, data as JSON string)
