# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Lark (Feishu) MCP Server exposing Feishu APIs as Model Context Protocol tools over STDIO. Two crates:

1. **larkrs-mcp** (root, `src/main.rs`) — the MCP server binary
2. **larkrs-client** (`larkrs-client/`) — standalone Feishu API client library, published to crates.io (path dependency, NOT a cargo workspace member; it has its own `Cargo.lock`)

Rust edition 2024. Key deps: `poem-mcpserver` 0.1.8, `reqwest` 0.12, `tokio`, `serde`, `anyhow` + `thiserror`, `env_logger`.

## Common Commands

```bash
cargo build                      # build server
cargo build --release
cargo run                        # run MCP server (STDIO)
cargo check                      # fast compile check
cargo test -p larkrs-client      # client tests (or: cd larkrs-client && cargo test)
cargo clippy && cargo fmt        # no custom config; defaults apply
```

### Environment
```bash
cp .env.example .env   # fill FEISHU_APP_ID and FEISHU_APP_SECRET
```
`.env` is gitignored — never commit it.

## Architecture

### Project Structure
- `src/main.rs` — `LarkServer` + all MCP tool definitions
- `larkrs-client/src/lib.rs` — `LarkApiResponse<T>` envelope (`code`/`msg`/`data`, `is_success()` ⇔ `code == 0`)
- `larkrs-client/src/auth.rs` — `FeishuTokenManager` (tenant access token, cached in `Arc<Mutex<>>`, 60s refresh buffer)
- `larkrs-client/src/bitable/` — `mod.rs` = DTOs (records, filters, fields); `table.rs` = `BitableTableClient`
- `larkrs-client/src/bot/` — `mod.rs` = DTOs (chat, text/post messages); `chat.rs` = `ChatClient`
- `larkrs-client/src/wiki/` — `mod.rs` = DTOs; `client.rs` = `WikiClient` (+ URL parsing)

Module pattern: each domain has `mod.rs` for serde types and a sibling file for the `XxxClient` with `new()` + async methods.

### MCP Server (`src/main.rs`)
- `poem-mcpserver` with STDIO transport; `#[Tools]` on `impl LarkServer`
- Each async method = one MCP tool; **its doc comment is the tool description shown to the LLM** — keep Args/examples accurate
- Tools return `Json<T>`

### MCP Tools Available
1. `table_records_list` — get records from a Bitable table
2. `create_table_records_json` — batch create records (JSON string array)
3. `table_fields_info` — field metadata incl. `write_type` hint
4. `chat_group_list` — list chat groups the bot is in
5. `send_text_message` — send text to a chat
6. `send_markdown_message` — send `post` (rich text) message
7. `read_wiki_content` — read raw text of a wiki doc from its URL

### Data Flow
`dotenvy`/env → `FeishuTokenManager` → `XxxClient` builds request with Bearer token → `LarkApiResponse<T>` → MCP tool wraps in `Json`.

### URL Formats
- Bitable: `https://xxx.feishu.cn/base/{app_token}?table={table_id}&view={view_id}`
- Wiki: `https://xxx.feishu.cn/wiki/{node_token}?...`

### Error Handling
- Client: `anyhow::Result` with `.context(...)`; domain errors via `thiserror` (`FeishuApiError`, `BitableApiError`)
- Server: tools currently use `unwrap_or_default()` / log-and-default — failures surface to the MCP caller as empty results

## Conventions & Gotchas (added by onboarding)

- **Never write to stdout in the server** — STDIO is the MCP transport. Use `log` macros (env_logger targets stderr). `RUST_LOG=debug cargo run` for verbose logs.
- **Token cache is per client instance**, and tools construct a new client per call, so every tool call re-fetches a tenant token. Share a client/token manager if adding high-frequency tools.
- **Tests are live integration tests** against Feishu: they need real `.env` creds and real IDs (placeholders are `"xxx"`), so they fail offline. Pure-logic code (URL parsing, DTO conversions) should get offline unit tests.
- `read_wiki_content` passes the wiki `node_token` directly to the docx `raw_content` API; this may not resolve for all wiki nodes (proper path: wiki get_node → `obj_token`).
- New Feishu API: add DTOs in `<domain>/mod.rs`, method on the domain client, then a tool in `main.rs`; update the tool list above and README.
- Bitable timestamps are milliseconds. Some docs/tool descriptions are in Chinese — keep that style when editing existing ones.
- Commits: conventional commits with optional scope, e.g. `feat(wiki): ...`, `feat(bot): ...`, `chore: ...`. Work on `main`.
