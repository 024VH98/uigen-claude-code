# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup (install deps + Prisma generate + migrate)
npm run setup

# Development server (Turbopack)
npm run dev

# Build
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Reset the SQLite database
npm run db:reset
```

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat; the AI generates code into a virtual file system; an iframe previews the result live.

### Request flow

1. User sends a chat message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The route reconstructs a `VirtualFileSystem` from the serialized file state sent by the client, then calls `streamText` (Vercel AI SDK) with two tools: `str_replace_editor` and `file_manager`
3. The AI streams back text and tool calls. Tool calls mutate the `VirtualFileSystem` on the server; results are sent back over the stream
4. On the client, `ChatContext` (`src/lib/contexts/chat-context.tsx`) processes incoming tool call parts and forwards them to `FileSystemContext` via `handleToolCall`
5. `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) keeps a client-side `VirtualFileSystem` in sync with AI mutations and drives re-renders via a `refreshTrigger` counter
6. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) reads `getAllFiles()`, runs Babel transforms + blob URL import maps (`src/lib/transform/jsx-transformer.ts`), and writes the result into an `<iframe srcdoc>`

### Key modules

| Path | Purpose |
|---|---|
| `src/lib/file-system.ts` | `VirtualFileSystem` class — in-memory tree, no disk I/O. Entry point for all file operations |
| `src/lib/transform/jsx-transformer.ts` | Transforms JSX/TSX via `@babel/standalone`, builds blob-URL import maps for the preview iframe. Third-party packages are resolved via `https://esm.sh/` |
| `src/lib/provider.ts` | Returns a `LanguageModelV1`. Uses `claude-haiku-4-5` when `ANTHROPIC_API_KEY` is set; otherwise falls back to `MockLanguageModel` which returns static demo components |
| `src/lib/tools/str-replace.ts` | AI tool wrapping `VirtualFileSystem` view/create/str_replace/insert commands |
| `src/lib/tools/file-manager.ts` | AI tool wrapping rename/delete commands |
| `src/lib/auth.ts` | JWT sessions via `jose`, stored in an HTTP-only cookie (`auth-token`). Passwords hashed with `bcrypt` |
| `src/lib/prompts/generation.tsx` | System prompt sent to Claude (cached with Anthropic prompt caching) |
| `src/lib/anon-work-tracker.ts` | Tracks anonymous users' work in localStorage before sign-up |
| `src/actions/` | Next.js Server Actions for project CRUD and user auth |

### Data model (Prisma / SQLite)

The database schema is defined in `prisma/schema.prisma` — reference it whenever you need to understand the structure of data stored in the database.

- **User** — email + bcrypt password
- **Project** — belongs to an optional User; stores `messages` (JSON array) and `data` (serialized `VirtualFileSystem` nodes) as JSON strings in SQLite text columns

### Authentication

JWT-based with `jose`. Sessions expire after 7 days. `src/middleware.ts` protects `/[projectId]` routes. The `JWT_SECRET` env var defaults to a hardcoded development string — set it in production.

### Preview rendering

The preview iframe uses native ES module import maps. `createImportMap()` transforms every `.js/.jsx/.ts/.tsx` file in the virtual FS to a blob URL and builds an `importmap` script tag. React 19 and `react-dom` are loaded from `esm.sh`. Tailwind CSS is injected via CDN `<script>`. If Babel transformation fails, syntax errors are displayed inside the iframe instead of crashing.

### Testing

Tests use Vitest with jsdom. Test files live under `__tests__/` directories next to the code they test. The vitest config (`vitest.config.mts`) enables `tsconfigPaths` so `@/` path aliases work in tests.

### Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | (none — uses mock) | Anthropic API key for real component generation |
| `JWT_SECRET` | `development-secret-key` | JWT signing secret |
| `DATABASE_URL` | `file:./prisma/dev.db` | SQLite database path |
