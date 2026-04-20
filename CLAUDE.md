# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Run tests
```bash
pytest                          # all tests
pytest tests/test_docx_base.py  # single test file
pytest -k "test_name"           # single test by name
```

### Local development (without Docker)
```bash
pip install -r requirements.txt
cp .env.example .env
python main.py
```

### Docker
```bash
docker-compose up -d            # start server
docker-compose logs -f          # follow logs
docker-compose down             # stop
```

The MCP endpoint is `http://localhost:8958/mcp`.

## Architecture

This is a **FastMCP server** (`fastmcp==3.0.0`) that exposes Office document generation as MCP tools. Entry point is `main.py`, which registers all tools and starts a streamable-HTTP server on port 8958.

### Tool modules

Each document type lives in its own package with a consistent pattern:

| Package | Tool registered in main.py | Core function |
|---|---|---|
| `xlsx_tools/` | `create_excel_from_markdown` | `markdown_to_excel()` |
| `docx_tools/` | `create_word_from_markdown` | `markdown_to_word()` |
| `pptx_tools/` | `create_powerpoint_presentation` | `create_presentation()` |
| `email_tools/` | `create_email_draft` | `create_eml()` |
| `xml_tools/` | `create_xml_file` | `create_xml_file()` |

Every tool calls `upload_tools.upload_file(file_object, suffix)` to persist the generated file.

### Dynamic template tools

At startup, `main.py` scans for YAML config files and dynamically registers additional MCP tools:
- **`config/email_templates.yaml`** → one tool per entry via `register_email_template_tools_from_yaml()` (uses Mustache/pystache rendering)
- **`config/docx_templates.yaml`** → one tool per entry via `register_docx_template_tools_from_yaml()` (replaces `{{placeholder}}` with Markdown-formatted content)

### Configuration (`config.py`)

`get_config()` returns a singleton Pydantic `Config` object. All env var access must go through this module — no module should call `os.environ` directly. Config also sets up global logging exactly once.

### Template resolution (`template_utils.py`)

Templates are resolved in this priority order:
1. `/app/custom_templates/` (production container)
2. `./custom_templates/` (local dev)
3. `/app/default_templates/` (production container)
4. `./default_templates/` (local dev)

Custom templates override defaults by using the same filename (e.g., `custom_pptx_template_16_9.pptx` overrides `default_pptx_template_16_9.pptx`).

### Upload backends (`upload_tools/`)

`upload_tools/main.py` dispatches to the appropriate backend based on `UPLOAD_STRATEGY`:
- `LOCAL` — saves to `./output/` (or `/app/output/` in container), returns a file path message
- `S3`, `GCS`, `AZURE`, `MINIO` — uploads and returns a time-limited signed URL

### Authentication (`middleware.py`)

When `API_KEY` is set, `ApiKeyAuthMiddleware` is registered as a FastMCP middleware. It accepts the key via `Authorization: Bearer <key>`, `Authorization: <key>`, or `x-api-key: <key>`.

### PowerPoint slide builder (`pptx_tools/slide_builder.py`)

`PowerpointPresentation` uses multiple-inheritance mixins (`SlideHelperMixin`, `TextHelperMixin`, `TableHelperMixin`, `ImageHelperMixin`) defined in `pptx_tools/helpers.py`. Supported slide types: `title`, `section`, `content`, `table`, `image`, `two_column`, `chart`, `quote`.

## Key conventions

- `config.py` is the single source of truth for env vars; use `get_config()` everywhere.
- `template_utils.py` is the single source of truth for finding template files.
- Test output files are written to `tests/output/<format>/` for manual inspection.
- `pytest.ini` sets `asyncio_mode = auto` for async tests.
- Python 3.12 (Alpine-based Docker image).
