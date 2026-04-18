# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenCode is an open-source AI coding agent with a CLI/TUI interface, web UI, and desktop app. It supports multiple LLM providers (Claude, OpenAI, Google, etc.) through a unified abstraction layer. Built with Bun, Effect, and SolidJS.

## Essential Commands

```bash
# Install dependencies (requires Bun 1.3+)
bun install

# Development
bun dev                      # Start core server (TUI)
bun dev <directory>          # Start TUI in specific directory
bun dev serve                # Start headless API server (port 4096)
bun dev web                  # Start server + web interface
bun run dev:desktop          # Start Tauri desktop app
bun run dev:web              # Start SolidJS web app
bun run dev:console          # Start console app

# Type checking (run from package directories, NOT repo root)
cd packages/opencode && bun typecheck    # uses tsgo
cd packages/opencode && bun typecheck    # NOT `tsc` directly

# Testing (run from package directories, NOT repo root)
cd packages/opencode && bun test         # single package tests
cd packages/opencode && bun test src/tool/bash.test.ts  # single test file

# Building
./packages/opencode/script/build.ts --single   # standalone executable

# Code generation
./script/generate.ts         # regenerate SDK after API/server changes

# JS SDK regeneration
./packages/sdk/js/script/build.ts

# Database
cd packages/opencode && bun run db   # drizzle-kit CLI

# Formatting
bunx prettier --write <files>       # semi: false, printWidth: 120
```

## Architecture

### Monorepo Structure

Turbo-managed monorepo with Bun workspaces. Default branch is `dev` (not `main`).

- **`packages/opencode`** — Core: CLI, server, agent runtime, providers, tools, sessions
- **`packages/app`** — Web UI (SolidJS + Vite), connects to core via SDK
- **`packages/desktop`** — Native desktop wrapper (Tauri v2) around web UI
- **`packages/console`** — Console web app (billing, Stripe integration)
- **`packages/plugin`** — Plugin system (`@opencode-ai/plugin`)
- **`packages/sdk`** — Auto-generated TypeScript SDK from OpenAPI specs
- **`packages/util`** — Shared utilities

### Core Package (`packages/opencode/src/`)

```
provider/     — LLM provider abstraction (unified interface for 20+ providers)
tool/         — Tool definitions (each tool = one .ts file + optional .txt for prompts)
session/      — Session lifecycle, message processing, prompt construction
server/       — Hono HTTP server, WebSocket, mDNS discovery, control plane
cli/          — yargs CLI entry, TUI lives in cli/cmd/tui/ (SolidJS + opentui)
agent/        — Agent definitions (build, plan, general subagent)
config/       — Configuration loading and management
storage/      — SQLite database via Drizzle ORM (WAL mode)
permission/   — Runtime permission system for tool execution
mcp/          — Model Context Protocol integration
lsp/          — Language Server Protocol client
```

### Key Patterns

**Effect Framework**: Heavy use of Effect throughout the codebase for dependency injection (`Layer`), error handling (`Effect.gen`), and resource management. Services are defined as Effect contexts with `Context.Tag`.

**Tool System**: Tools are defined with `Tool.define()` or `Tool.Info` objects containing id, parameters (Zod schema), and execute function returning `Effect`. Each tool file in `src/tool/` is self-contained. Tool descriptions are often stored in companion `.txt` files.

**Provider Abstraction**: `src/provider/provider.ts` defines the unified interface. Each provider handles auth, model listing, and message transformation. Provider config is in `src/provider/schema.ts`.

**Server**: Hono-based HTTP server with middleware, SSE streaming, and WebSocket support. Routes are in `src/server/server.ts` with controllers in `src/server/instance/`.

**Database**: SQLite via Drizzle ORM. Schema uses snake_case field names. Migrations run automatically on startup. Import path `#db` resolves platform-specific implementations.

**Platform Abstractions**: Import path `#pty` resolves to Bun or Node implementations of node-pty. The `--conditions=browser` flag is needed for TUI rendering.

### `bun dev` vs `opencode`

`bun dev` is the local dev equivalent of the production `opencode` command. Both accept the same CLI flags (`--help`, `serve`, `web`, `<directory>`).

## Code Style (from AGENTS.md)

- **Naming**: Prefer single-word variable/function names. Only multi-word when ambiguous.
- **Avoid**: `try`/`catch` (use Effect), `else` statements (early return), `let` (use `const`), unnecessary destructuring, `any` type
- **Prefer**: `Bun.file()`, functional array methods, type inference over explicit annotations, ternaries over reassignment
- **Schema (Drizzle)**: snake_case field names so columns don't need string redefinitions
- **Tests**: Avoid mocks; test actual implementation; never run from repo root
- **Formatting**: Prettier with `semi: false`, `printWidth: 120`

## PR Requirements

- All PRs must reference an existing issue (`Fixes #123`)
- PR titles follow conventional commits: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`
- Optional scope: `feat(app):`, `fix(desktop):`, `chore(opencode):`
- UI changes need screenshots/videos; logic changes need verification explanation
- Keep PRs small and focused

## Contributing Notes

- Bug fixes, LSP/formatter additions, provider support, perf fixes, and docs improvements are welcome
- UI/core product features require design review with core team first
- New providers should be added via https://github.com/anomalyco/models.dev first
- If changing API or SDK, run `./script/generate.ts` to regenerate
