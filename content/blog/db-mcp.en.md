+++
title = "db-mcp: connect your database to AI"
date = 2026-05-30
description = "A lightweight MCP server written in Rust that lets Claude and other tools query PostgreSQL, MySQL, SQLite, and ClickHouse safely and privately"
+++

I needed a private utility for work and personal projects: read data from databases, but never write. Existing solutions weren't what I wanted — those that existed were JS/Python packages. I just needed a simple binary: download and run, without Node.js or other overhead. Plus, always nice to build something yourself :)

So I built **db-mcp** ([github](https://github.com/zeslava/db-mcp)) — a lightweight Rust binary that:
- Reads from PostgreSQL, MySQL/MariaDB, SQLite, and ClickHouse via URL scheme
- Works with Claude, OpenCode, Jan, Zed, and any MCP-compatible client
- Simple and safe by design: read-only (SELECT only), parameterized queries
- Builds once with all backends enabled — choose your database at startup
- Ships as a fully static Linux binary (musl, no glibc at runtime)

## Why this was needed

At work, I often need to give Claude access to data: fetch information, analyze, provide context for scripting. Requirements were:
- **Safe** (read-only, parameterized queries)
- **Private** (runs locally, nothing sent to cloud)
- **Universal** (not just Claude — I use OpenCode, Zed, Jan for different tasks)


## How it works

db-mcp is a stdio MCP server written in Rust. You point it at a database URL via the `--database-url` flag or the `DATABASE_URL` env var:

```bash
db-mcp --database-url postgres://user:pass@localhost:5432/mydb
db-mcp --database-url mysql://user:pass@localhost:3306/mydb
db-mcp --database-url sqlite:///absolute/path/to/data.db
db-mcp --database-url clickhouse://default:pass@localhost:8123/default
```

The server detects the engine from the URL scheme and connects. Then Claude/OpenCode/Jan can use three tools:

| Tool | Params | What it does |
|------|--------|--------------|
| `list_tables` | — | List user tables |
| `describe_table` | `table`, `schema?` | Columns, types, nullability |
| `query` | `sql` | Run a SELECT, returns JSON rows |

A `query` call returns plain JSON, so the model gets real values to reason about:

```json
[
  { "id": 1, "email": "ada@example.com", "created_at": "2026-05-01 09:12:33" },
  { "id": 2, "email": "linus@example.com", "created_at": "2026-05-03 14:50:01" }
]
```

### Safe by design

SELECT-only enforcement lives in the server, not in the adapters. Anything that doesn't start with `SELECT` (case-insensitive, after trim) is rejected — that even blocks CTE tricks like `WITH ... INSERT ... RETURNING`. Every adapter uses parameterized queries, never string-concatenated SQL. For an extra layer, point it at a read-only database role.

## Setup for different tools

All clients launch the same binary; the difference is just where the config file lives. I prefer passing the URL through `DATABASE_URL` so it isn't visible in the args list.

### Claude Code (CLI)

**cli**
```bash
claude mcp add db \
  --env DATABASE_URL=postgres://user:pass@localhost:5432/mydb \
  -- /absolute/path/to/db-mcp
```

**claude.json**
```json
{
  "mcpServers": {
    "db-mcp": {
      "type": "stdio",
      "command": "/absolute/path/to/db-mcp",
      "args": [],
      "env": {
        "DATABASE_URL": "postgres://user:pass@localhost:5432/mydb"
      }
    }
  }
}
```

Restart Claude and the tools appear.

### OpenCode

In your `opencode.json` (project or `~/.config/opencode/opencode.json`):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "database": {
      "type": "local",
      "command": ["db-mcp", "--database-url", "sqlite:///home/me/work/data.sqlite"],
      "enabled": true
    }
  }
}
```

### Jan

Jan has built-in MCP support (Settings → MCP Servers). The equivalent config is a command + args + env entry:

```json
{
  "database": {
    "command": "/absolute/path/to/db-mcp",
    "args": [],
    "env": {
      "DATABASE_URL": "postgres://prod.example.com:5432/analytics"
    }
  }
}
```

### Zed

In `settings.json` (custom MCP servers live under `context_servers`, not `lsp`):

```json
{
  "context_servers": {
    "db-mcp": {
      "enabled": true,
      "remote": false,
      "command": "/absolute/path/to/db-mcp",
      "args": [],
      "env": {
        "DATABASE_URL": "postgres://user:pass@localhost:5432/mydb"
      }
    }
  }
}
```

## In practice

### Scenario 1: development context

```
In Claude Code:
"Help me write a migration to add a new field.
Check the current users table structure."

Claude:
1. Calls db-mcp (describe_table) → gets current schema
2. Sees existing fields and constraints
3. Generates a correct migration for your actual database
```

### Scenario 2: data analysis in OpenCode

```
In OpenCode analyzing metrics:
"How many active users this month?
What are the most popular features?"

OpenCode uses db-mcp (query), the model sees real numbers
and provides analysis grounded in actual data.
```

### Scenario 3: debugging in Jan

```
Jan with db-mcp helps debug:
"Are there orders without user_id in the orders table?"

Jan queries through db-mcp, sees there are 12 such records,
suggests how to fix it.
```

## Installation

Quick install (Linux and macOS Apple Silicon) — auto-detects OS/arch, verifies the checksum, installs to `~/.local/bin` by default:

```bash
curl --proto '=https' --tlsv1.2 -sSf \
  https://raw.githubusercontent.com/zeslava/db-mcp/main/install.sh | sh
```

Or grab a release tarball manually (Linux x86_64/aarch64, macOS arm64, Windows x86_64):

```bash
TARGET=x86_64-unknown-linux-gnu
curl -sSL "https://github.com/zeslava/db-mcp/releases/latest/download/db-mcp-${TARGET}.tar.gz" \
  | tar -xz
install -m 755 "db-mcp-${TARGET}/db-mcp" "$HOME/.local/bin/db-mcp"
```

Or build from source:

```bash
git clone https://github.com/zeslava/db-mcp
cd db-mcp
cargo build --release
./target/release/db-mcp --database-url postgres://localhost/mydb
```

## Benefits of this approach

- ✅ One binary for four engines (PostgreSQL, MySQL/MariaDB, SQLite, ClickHouse)
- ✅ Works everywhere: Claude, OpenCode, Jan, Zed, any MCP client
- ✅ No runtime overhead (no JS, Python) — pure Rust, static musl binary on Linux
- ✅ Safe by design: SELECT-only, parameterized queries
- ✅ Private: runs locally or in your infrastructure
- ✅ Simple setup: just a database URL

## What's next

Planned features:
- Support for more databases (and beyond)
- Query logging and audit trails
- Per-table access control (for more granular permissions)
- Server mode
