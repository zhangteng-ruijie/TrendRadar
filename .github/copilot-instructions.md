# TrendRadar — Copilot Instructions

TrendRadar is a Chinese-language hot-news aggregation and analysis tool. It scrapes trending topics from Chinese platforms via the NewsNow API and RSS feeds, runs AI analysis, generates HTML reports, and dispatches notifications to multiple channels. It also exposes an MCP server for AI assistant integration.

- **Language**: Python 3.12+, package manager: `uv`
- **No test framework exists** — there is no `tests/` directory.

## Commands

```bash
uv sync                                                    # install dependencies
python -m trendradar                                       # run the crawler
python -m trendradar --show-schedule                       # show scheduling plan
python -m trendradar --doctor                              # validate config + connectivity
python -m trendradar --test-notification                   # send a test push
python -m mcp_server.server                                # MCP server (stdio)
python -m mcp_server.server --transport http --host 0.0.0.0 --port 3333  # MCP (HTTP)
```

## Architecture

### Two packages

| Package | Role |
|---|---|
| `trendradar/` | Core CLI: crawling, analysis, reporting, scheduling |
| `mcp_server/` | FastMCP 2.0 server exposing 8 tool modules to AI assistants |

### `AppContext` — the dependency injection root

`trendradar/context.py` contains `AppContext`, which wraps the loaded config dict and exposes all domain operations as methods. **Never pass the raw config dict around**; create an `AppContext` and pass that instead. It is instantiated once in `__main__.py` after `load_config()`.

### Domain subpackage layout

Each domain lives in its own subpackage under `trendradar/` with an `__init__.py` that exports a clean public API (always uses `__all__`). Cross-module imports go through these public APIs, never through internal modules directly.

| Subpackage | Key classes / functions |
|---|---|
| `core/` | `load_config()`, `Scheduler`, frequency matching |
| `crawler/` | `DataFetcher` — talks to the NewsNow API |
| `storage/` | `StorageBackend` (ABC), `LocalStorageBackend`, `RemoteStorageBackend`, `StorageManager` |
| `ai/` | `AIClient` (LiteLLM wrapper), `AIAnalyzer`, `AIFilter`, `AITranslator` |
| `notification/` | `NotificationDispatcher`, per-channel sender functions |
| `report/` | `generate_html_report()` — `html.py` is ~102 KB |
| `utils/` | Timezone-aware time helpers, URL normalization |

### Storage — strategy pattern

`StorageBackend` is an ABC. Both `LocalStorageBackend` (SQLite + local files) and `RemoteStorageBackend` (S3-compatible) implement it, sharing SQLite logic via `SQLiteStorageMixin`. **Always access storage through `StorageManager`** (obtained via `get_storage_manager()` singleton), not by instantiating backends directly. The manager auto-selects the backend (`auto` mode uses remote in GitHub Actions, local otherwise).

### Data models

Core data flows through `@dataclass` objects with `to_dict()` / `from_dict()` for serialization:
- `NewsItem` / `NewsData` — hot-list data, grouped by `source_id`
- `RSSItem` / `RSSData` — RSS data, grouped by `feed_id`

### AI client

`AIClient` (`trendradar/ai/client.py`) wraps LiteLLM. Models are specified as `provider/model_name` (e.g., `deepseek/deepseek-chat`, `openai/gpt-4o`). Fallback models and retry counts are configured in `config.yaml` under the `ai:` key and can be overridden by the `AI_API_KEY` environment variable.

### Configuration loading

`load_config()` reads `config/config.yaml` (path overridable via `CONFIG_PATH` env var). Many settings can be overridden per-field with environment variables in SCREAMING_SNAKE_CASE (e.g., `TIMEZONE`, `DEBUG`, `AI_API_KEY`, `S3_BUCKET_NAME`). The resolved config dict uses SCREAMING_SNAKE_CASE keys (e.g., `config["TIMEZONE"]`).

### MCP server

`mcp_server/server.py` creates a `FastMCP` app and registers tools from 8 tool-class modules (`data_query`, `analytics`, `search_tools`, `config_mgmt`, `system`, `storage_sync`, `article_reader`, `notification`). Tool classes are lazily instantiated as singletons keyed in `_tools_instances`. MCP resources are `config://platforms` and `config://rss-feeds`.

## Conventions

- Every Python file starts with `# coding=utf-8`.
- Module-level docstrings and inline comments are in **Chinese**.
- Config keys in the Python dict returned by `load_config()` are **SCREAMING_SNAKE_CASE**; YAML keys are `snake_case`.
- When adding a new notification channel, implement a sender function in `notification/senders.py` and register it in `NotificationDispatcher` (`notification/dispatcher.py`).
- When adding a new storage method, define it as `@abstractmethod` on `StorageBackend`, implement it in both `LocalStorageBackend` and `RemoteStorageBackend`, then add a delegation method on `StorageManager`.
- The `SQLiteStorageMixin` holds all SQLite query logic shared by both backends — add new DB operations there.
- The MCP tool modules expose methods that are registered as MCP tools; keep one class per file and use `asyncio.to_thread()` for synchronous operations inside async MCP handlers.
