# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

O'Reilly book ingest tool using a microkernel architecture. Converts books to LLM-ready formats (Markdown, JSON, TOON, plain text) for AI interaction. Also supports PDF and EPUB 3. Python 3.10+, vanilla Python with minimal dependencies. Unit tests live in `tests/` (stdlib `unittest`); no linter is configured. Run the tests after plugin changes, and verify web/endpoint changes by running the server and exercising them directly.

## Commands

```bash
# Setup
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Run the web server
python main.py                    # Default: localhost:8000
python main.py --host 0.0.0.0 --port 9000   # Custom host/port
.venv/bin/python main.py          # Run with venv directly, no activation needed

# Docker
docker compose up -d              # Builds image, mounts ./data (cookies) and ./output

# Refresh O'Reilly session cookies from local Chrome (macOS; the orm-jwt
# token expires in ~15 min, so this is the normal way to re-auth)
python scripts/refresh_cookies.py
python scripts/refresh_cookies.py --watch     # re-sync every 60s

# Tests (stdlib unittest; run from the repo root)
.venv/bin/python -m unittest discover -s tests -v
```

## Architecture

### Microkernel Pattern
- **Kernel** (`core/kernel.py`): Plugin registry, provides shared `HttpClient` to all plugins
- **Plugins** (`plugins/`): Independent modules registered with the kernel, access HTTP via `self.http`
- Plugins extend `Plugin` base class from `plugins/base.py`
- Use `create_default_kernel()` to get a fully configured kernel with all plugins

### Plugin Categories

**Core Plugins** (data acquisition):
- `AuthPlugin` - Session validation via /profile/
- `BookPlugin` - Fetch metadata (V2 Search + V2 Epubs APIs)
- `ChaptersPlugin` - Fetch chapter list and content
- `AssetsPlugin` - Download CSS and images
- `HtmlProcessorPlugin` - Extract sbo-rt-content, rewrite links, wrap XHTML

**Output Format Plugins**:
- `EpubPlugin` - Generate EPUB structure and ZIP
- `MarkdownPlugin` - Convert HTML to Markdown
- `PdfPlugin` - Generate PDF output (WeasyPrint, styles in `plugins/pdf_styles/`)
- `PlainTextPlugin` - Extract plain text
- `JsonExportPlugin` - Export structured JSON
- `ToonExportPlugin` - Subclasses `JsonExportPlugin`, re-encodes the same export structure as TOON (token-efficient format) instead of JSON
- `ChunkingPlugin` - Split content into token/word-bounded chunks, streamed to disk (not buffered in memory)
- `TokenPlugin` - Token counting

**Orchestration Plugins**:
- `OutputPlugin` - Coordinate output generation
- `SystemPlugin` - System information
- `DownloaderPlugin` - Orchestrate full download workflow

### Web Server
- `web/server.py`: `ThreadingHTTPServer` (stdlib `http.server`), serves static files from `web/static/`
- JSON API endpoints: `/api/status`, `/api/search`, `/api/book/{id}`, `/api/download`, `/api/progress` (SSE), plus settings/cookies/cancel/reveal-in-finder handlers
- Download progress is tracked in class-level state on the handler and streamed to the browser via SSE (`_handle_progress`)

### Shared Core Utilities
- `core/http_client.py`: `curl_cffi` session impersonating `safari17_0` to pass Akamai bot detection; deliberately keeps Akamai cookies (`_abck`, `bm_*`, `ak_*`) and never overrides `User-Agent`, since either would cause a TLS-fingerprint/cookie mismatch and trigger 403s. Handles retry/backoff (`config.MAX_RETRIES`, `RETRY_BACKOFF`) and rate limiting (`config.REQUEST_DELAY`).
- `core/text_extractor.py`: Shared HTML→clean-text extraction used by `PlainTextPlugin`, `JsonExportPlugin`, and `ChunkingPlugin`
- `core/types.py`: `TypedDict` contracts (e.g. `ChapterInfo`, `ChapterSummary`) shared across plugins and the web/CLI interfaces so data shapes stay consistent regardless of client

### Key Files
- `config.py`: `BASE_URL`, `API_V1`, `API_V2`, `OUTPUT_DIR`, `COOKIES_FILE`, `REQUEST_DELAY`, retry/timeout settings
- `cookies.json`: User-provided O'Reilly session cookies (BrowserCookie, OptanonConsent, Akamai cookies). Path resolves to `data/cookies.json` when a `data/` dir exists (Docker), else repo root (local dev)
- `output/`: Downloaded books stored here as `{book_title}/` directories
- `scripts/refresh_cookies.py`: Pulls fresh cookies straight from the local Chrome keychain and pushes them to the running server, since the O'Reilly access token expires in ~15 minutes

## API Strategy

Use V2 APIs primarily:
- **V2 Search** (`/api/v2/search/?query={ISBN}`) - authors, publishers, topics, cover_url
- **V2 Epubs** (`/api/v2/epubs/urn:orm:book:{ID}/`) - spine, chapters URL, TOC URL
- **V2 Chapters** (`/api/v2/epub-chapters/`) - full chapter metadata with `related_assets`
- **V2 Content** returns `sbo-rt-content` div directly (no full HTML parsing needed)

## Content Pipeline

1. Fetch book metadata (BookPlugin)
2. Fetch chapters list with pagination (ChaptersPlugin)
3. For each chapter: fetch HTML, extract `sbo-rt-content`, collect CSS/images
4. Rewrite links (.html → .xhtml, image paths → Images/)
5. Download all CSS and images (AssetsPlugin)
6. Generate output format: EPUB, PDF, Markdown, plain text, or JSON

## Adding a New Plugin

```python
from plugins.base import Plugin

class NewPlugin(Plugin):
    def do_something(self, params):
        # Access HTTP client via self.http
        response = self.http.get(url)
        # Access other plugins via self.kernel["plugin_name"]
        return result
```

Register in `core/kernel.py`:
```python
kernel.register("new_plugin", NewPlugin())
```

## Agent skills

### Issue tracker

Fork workflow: `origin` is the fork (`xsmJohnnyDepp/oreilly-ingest`), `upstream` is the canonical repo (`mosaibah/oreilly-downloader`) where issues and PRs are filed via `gh`. See `docs/agents/issue-tracker.md`.

### Domain docs

Single-context layout — one `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
