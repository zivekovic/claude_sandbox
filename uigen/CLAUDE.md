# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running with Docker

The project runs in Docker rather than locally. The `Dockerfile` uses `node:20-alpine`, installs native build deps for `bcrypt`, runs `npm ci`, generates the Prisma client, and on container start applies migrations then launches `npm run dev`.

```bash
# Build the image
docker build -t uigen .

# Run the container (passes .env for ANTHROPIC_API_KEY)
docker run -d --name uigen -p 3000:3000 --env-file .env uigen

# View logs
docker logs uigen -f

# Restart after .env changes or code edits
docker restart uigen

# Stop / remove
docker stop uigen
docker rm uigen
```

App is available at http://localhost:3000.

> Note: The SQLite database (`prisma/dev.db`) lives inside the container and is recreated fresh each time the container is removed. To persist data across container rebuilds, mount a volume: `-v $(pwd)/prisma:/app/prisma`.

## Commands (inside the container)

```bash
# Install deps, generate Prisma client, run migrations
npm run setup

# Start dev server (Turbopack)
npm run dev

# Run all tests
npm test

# Run a single test file
npx vitest run src/path/to/file.test.ts

# Reset database (drops all data)
npm run db:reset

# Regenerate Prisma client after schema changes
npx prisma generate

# Create a new migration after schema changes
npx prisma migrate dev --name <migration-name>
```

## Environment

- `ANTHROPIC_API_KEY` in `.env` — optional. Without it the app uses `MockLanguageModel` which returns static code instead of calling Claude.
- `JWT_SECRET` — defaults to `"development-secret-key"` if not set.
- SQLite database lives at `prisma/dev.db`. Prisma client is generated into `src/generated/prisma/`.

## Architecture

### Request Flow

1. User sends a message in `ChatInterface`
2. `ChatContext` (`src/lib/contexts/chat-context.tsx`) posts to `/api/chat` with the serialized virtual filesystem and message history
3. `/src/app/api/chat/route.ts` streams a response from Claude (or mock) using `streamText` with two tools: `str_replace_editor` and `file_manager`
4. Tool calls stream back to the client where `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) executes them against the in-memory `VirtualFileSystem`
5. On stream finish, the API saves updated messages + serialized filesystem to the `Project` row in SQLite

### Virtual File System

`src/lib/file-system.ts` — a `Map<string, FileNode>` tree. All paths are normalized to start with `/`. Claude always starts from `/App.jsx` as the entry point.

Key operations used by AI tools: `createFile`, `updateFile`, `deleteFile`, `rename`, `replaceInFile`, `insertInFile`, `viewFile`. The VFS serializes to/from plain JSON for database persistence.

### AI Tools

Defined in `src/lib/tools/`:
- `str_replace_editor` — text-editor style tool (commands: `create`, `view`, `str_replace`, `insert`). Primary tool for writing/editing files.
- `file_manager` — filesystem operations (commands: `rename`, `delete`).

The system prompt is in `src/lib/prompts/generation.tsx`.

### Language Model Factory

`src/lib/provider.ts` returns either a real `@ai-sdk/anthropic` Claude model (default: `claude-haiku-4-5`) or a `MockLanguageModel` when no API key is present. `maxSteps` is 40 for real, 4 for mock.

### Live Preview

`PreviewFrame` compiles the virtual filesystem on the client using `@babel/standalone` (JSX → JS), builds an import map that resolves `@/` aliases to blob URLs and third-party packages to `esm.sh`, then renders the result in a sandboxed `<iframe>` via `srcdoc`.

### Authentication

JWT sessions via `jose` stored in an httpOnly cookie (7-day expiry). `src/lib/auth.ts` handles token creation/verification. `src/middleware.ts` protects `/api/projects` and `/api/filesystem`. Server actions for sign-up/in/out are in `src/actions/index.ts`.

Anonymous users can work without logging in; their in-progress work is tracked in `sessionStorage` (`src/lib/anon-work-tracker.ts`) and migrated to a new project on sign-up/login.

### Data Model

The database schema is defined in `prisma/schema.prisma` — reference it whenever you need to understand the structure of data stored in the database.

Two Prisma models (`prisma/schema.prisma`):
- `User` — email + bcrypt-hashed password
- `Project` — `messages` (JSON string) + `data` (serialized VFS JSON string), optionally linked to a user

### Key Contexts

- `FileSystemContext` — owns the `VirtualFileSystem` instance, selected file, and tool call execution
- `ChatContext` — wraps the Vercel AI SDK `useChat` hook, connects to `/api/chat`, passes tool results to `FileSystemContext`
