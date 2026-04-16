# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**TrendRadar** (v6.6.1) is a Chinese-language hot news/trend aggregation and analysis tool. It scrapes trending topics from multiple Chinese platforms (Weibo, Zhihu, Baidu, Douyin, etc.) and RSS feeds, performs AI-powered analysis, generates HTML reports, and pushes notifications to multiple channels. It also provides an MCP server (v4.0.3) for AI assistant integration.

- **Language**: Python 3.12+
- **Package Manager**: `uv`
- **Build System**: hatchling
- **License**: GPL-3.0

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Web Scraping | `requests` against NewsNow API |
| RSS Parsing | `feedparser` |
| AI Integration | `litellm` (100+ providers) |
| MCP Server | `fastmcp` v2.12.5 |
| Cloud Storage | `boto3` (S3-compatible) |
| Retry Logic | `tenacity` |
| Scheduling | `supercronic` (Docker) |
| Config | `PyYAML` |

## Directory Structure

```
TrendRadar/
  trendradar/          # Main Python package (CLI/crawler engine)
  mcp_server/          # MCP server Python package (FastMCP-based)
  config/              # YAML configs, AI prompts, frequency words
  docker/              # Dockerfile, docker-compose, entrypoint
  docs/                # Static HTML assets for web server
  output/              # Runtime output data
  .github/             # Workflows + issue templates
  pyproject.toml       # Project metadata + dependencies
```

## Key Commands

```bash
# Install dependencies
uv sync

# Run the main crawler
python -m trendradar

# Run MCP server (stdio mode)
python -m mcp_server.server

# Run MCP server (HTTP mode)
python -m mcp_server.server --transport http --host 0.0.0.0 --port 3333

# CLI diagnostic commands
python -m trendradar --show-schedule
python -m trendradar --doctor
python -m trendradar --test-notification

# Docker build (see docker/Dockerfile)
# Docker Compose (see docker/docker-compose.yml)
```

## Architecture

### High-Level Design

The project follows a domain-driven module structure with two main Python packages:

1. **`trendradar/`** -- Core CLI application (crawler, analysis, reporting)
2. **`mcp_server/`** -- MCP server for AI assistant integration

### Core Pattern: AppContext

The `AppContext` class (`trendradar/context.py`) acts as a dependency injection container, encapsulating all configuration-dependent operations. This is the central object wired through the application, eliminating global state.

### Module Organization

Each domain is a self-contained subpackage under `trendradar/` with its own `__init__.py` that exports the public API:

- **`core/`** -- Configuration loading (`loader.py`), timeline-based scheduling (`scheduler.py`), data processing (`analyzer.py`, `data.py`), keyword frequency matching (`frequency.py`)
- **`crawler/`** -- Data fetching from hot-list platforms via NewsNow API (`fetcher.py`), RSS feed parsing (`rss/`)
- **`storage/`** -- Multi-backend storage layer with strategy pattern: `StorageBackend` ABC with `LocalStorageBackend` (SQLite + files) and `RemoteStorageBackend` (S3-compatible), auto-selected by `StorageManager`
- **`ai/`** -- AI analysis (`analyzer.py`), filtering (`filter.py`), translation (`translator.py`), all routed through `AIClient` (LiteLLM wrapper with fallback support)
- **`notification/`** -- Multi-channel dispatcher (Feishu, DingTalk, WeCom, Telegram, Email, ntfy, Bark, Slack) with multi-account support and message batching
- **`report/`** -- HTML report generation (`html.py` is the largest single file at 102KB)
- **`utils/`** -- Timezone-aware time operations, URL utilities

### MCP Server Architecture

`mcp_server/server.py` is a FastMCP 2.0 application providing:
- Resources: `config://platforms`, `config://rss-feeds`
- Tools from 8 tool modules: analytics, notification, search, storage_sync, system, data_query, article_reader, config_mgmt

Supports both stdio and HTTP transports.

### Scheduling

Uses a sophisticated timeline-based model (`config/timeline.yaml`) with periods + day_plans + week_map, supporting presets: always_on, morning_evening, office_hours, night_owl, custom.

### Configuration

Main config at `config/config.yaml` covers timezone, schedule, platforms, RSS feeds, report modes, AI settings, notification channels, and storage backends.

## Testing

**No test framework exists in this repository.** There is no `tests/` directory, no pytest configuration, and no test-related files.

## Build & Install

```bash
# Install with uv
uv sync

# Entry points defined in pyproject.toml:
#   trendradar        -> trendradar.__main__:main
#   trendradar-mcp    -> mcp_server.server:run_server
```

# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.
