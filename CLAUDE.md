# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Commands

```bash
# First-time setup — run this once when you clone the repo.
# It installs all packages AND sets up the database in one go.
npm run setup

# Start the development server. Visit http://localhost:3000 in your browser.
npm run dev

# Build an optimised production bundle (not needed during local development).
npm run build

# Check the code for style/quality issues (ESLint).
npm run lint

# Run all automated tests.
npm test

# Run just one specific test file (swap in any path you like).
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Wipe the database and start fresh — useful if migrations get messy.
npm run db:reset

# Apply database schema changes after you edit prisma/schema.prisma.
npx prisma migrate dev
```

> **Why does every script have `NODE_OPTIONS="--require ./node-compat.cjs"`?**
> The `bcrypt` password-hashing library uses some older Node.js features that conflict with Next.js 15. The `node-compat.cjs` file patches those differences at startup. You never need to touch it — the npm scripts handle it automatically.

---

## How the app works — big picture

UIGen lets a user *describe* a React component in plain English. The AI (Claude) writes the code, and the app shows a live preview of that component — all without saving any files to disk.

Here is the full journey from a user's message to a rendered preview:

```
1. User types a message in the chat box
       ↓
2. The browser sends the message + current virtual files to POST /api/chat
       ↓
3. The server asks Claude to respond. Claude doesn't reply with plain text —
   it calls "tools" (like a robot using a keyboard) to create/edit files.
       ↓
4. Those tool calls stream back to the browser in real time
       ↓
5. The browser receives each tool call and applies it to its own copy
   of the virtual file system
       ↓
6. Every time a file changes, the preview panel recompiles everything
   and reloads the iframe
       ↓
7. The user sees their component rendered live
```

---

## Key concepts for beginners

### What is a "virtual file system"?
Normally, when you create a file it gets saved on your hard drive. Here, files only exist **in memory** (as JavaScript objects). Nothing is written to disk. This means the preview is instant and there is no cleanup needed. The class `VirtualFileSystem` in `src/lib/file-system.ts` is the core of this — it stores files in a `Map` (a key-value structure where the key is the file path like `/App.jsx`).

### What are "AI tools"?
When you call the Claude API, you can give it a list of "tools" — functions the AI is allowed to call. Instead of just typing an answer, Claude can call `str_replace_editor` to create or edit a file, or `file_manager` to rename/delete one. Think of it like giving the AI a set of buttons it can press. The tools live in `src/lib/tools/`.

### What is "streaming"?
Instead of waiting for Claude to finish its entire response and then sending it all at once, the app uses **streaming** — the response arrives word by word (or tool call by tool call) in real time. This is why you see the code appear gradually. The Vercel AI SDK (`ai` package) handles the streaming plumbing.

### What is a React Context?
A React Context is a way to share data across many components without passing it through every level manually. This app uses two:
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — shares the virtual file system with any component that needs it (the editor, the preview, the chat).
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — shares the chat state (messages, input, status) and connects incoming AI tool calls to the file system.

### What is Babel doing in the browser?
Normally Babel is a build-time tool that converts modern JS/JSX into code browsers can run. Here, `@babel/standalone` runs **inside the browser** to compile the AI-generated JSX on the fly. The compiled output is turned into a `blob:` URL (a temporary in-memory URL) and loaded via an ES module import map inside the `<iframe>`. See `src/lib/transform/jsx-transformer.ts`.

### What is an import map?
When the preview iframe loads your component, it needs to resolve `import React from 'react'`. An import map is a small JSON block inside the HTML that tells the browser *"when you see `import 'react'`, actually fetch it from `https://esm.sh/react@19`"*. The app builds this map dynamically from the virtual files.

### What is Prisma?
Prisma is an ORM — a tool that lets you interact with a database using TypeScript instead of raw SQL. The schema is in `prisma/schema.prisma`. The database is SQLite (a single file at `prisma/dev.db`). There are two tables: `User` and `Project`.

### How does authentication work?
After a user signs in, the server creates a **JWT** (JSON Web Token) — a signed string that proves who the user is — and stores it in a secure cookie. On every subsequent request, the server reads and verifies that cookie. Anonymous users can still use the app; their work is saved temporarily in `sessionStorage` so it can be transferred to their account if they sign up.

---

## Codebase map

| Path | What it does |
|---|---|
| `src/app/page.tsx` | Home page — the main chat + preview layout |
| `src/app/[projectId]/page.tsx` | Loads a saved project by ID |
| `src/app/api/chat/route.ts` | The only API route — receives messages, calls Claude, streams tool calls back |
| `src/lib/file-system.ts` | `VirtualFileSystem` class — all file operations |
| `src/lib/tools/str-replace.ts` | AI tool for creating/editing files |
| `src/lib/tools/file-manager.ts` | AI tool for renaming/deleting files |
| `src/lib/transform/jsx-transformer.ts` | Compiles JSX → blob URLs, builds import map, generates preview HTML |
| `src/lib/contexts/file-system-context.tsx` | React Context that holds the live file system on the client |
| `src/lib/contexts/chat-context.tsx` | React Context that holds chat state and wires tool calls to the file system |
| `src/lib/auth.ts` | JWT creation/verification (server-only) |
| `src/lib/anon-work-tracker.ts` | Saves anonymous work to `sessionStorage` |
| `src/lib/provider.ts` | Returns the Claude model, or a mock if no API key is set |
| `src/lib/prompts/generation.tsx` | The system prompt sent to Claude |
| `src/components/preview/PreviewFrame.tsx` | The `<iframe>` that shows the live component preview |
| `src/components/editor/CodeEditor.tsx` | Monaco-based code editor |
| `src/components/chat/` | Chat UI components |
| `prisma/schema.prisma` | Database schema (`User`, `Project`) |

### Path alias
Throughout the codebase you will see imports like `import { X } from "@/lib/..."`. The `@/` is a shortcut that means `src/`. It is configured in `tsconfig.json`.

---

## Testing

Tests use **Vitest** (a fast test runner), **jsdom** (a fake browser environment so React components can render in Node.js), and **Testing Library** (helpers to query and interact with rendered components). Test files sit in `__tests__/` folders next to the code they test.
