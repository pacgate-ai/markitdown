# AGENTS.md

Guidance for AI coding agents working in the `markitdown` monorepo.

## Project overview

MarkItDown converts many file formats (PDF, DOCX, PPTX, XLSX, HTML, images, audio, EPUB, CSV, JSON, ZIP, YouTube, …) to Markdown for LLM consumption. It is a Python monorepo with four packages under `packages/`:

| Package | Purpose | Entry point |
|---|---|---|
| `markitdown` | Core library + CLI | `markitdown` console script (`src/markitdown/__main__.py`) |
| `markitdown-mcp` | MCP server (STDIO / Streamable HTTP / SSE) wrapping the core | `markitdown-mcp` console script |
| `markitdown-ocr` | Plugin: LLM-Vision OCR for PDF/DOCX/PPTX/XLSX images | `markitdown.plugin` entry point `ocr` |
| `markitdown-sample-plugin` | Reference plugin (RTF converter) | `markitdown.plugin` entry point `sample_plugin` |

Requires Python ≥ 3.10 (3.12/3.13 recommended). Build backend is `hatchling`; versions are dynamic via `__about__.py`.

## Easiest local setup (native, recommended)

No `docker-compose.yml` exists. Docker is optional (see below). For local development, use a virtualenv + editable installs:

```bash
# 1. Create a venv (pick one)
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
# or with uv (faster):
uv venv --python=3.12 .venv && source .venv/bin/activate

# 2. Install the core package editable with all optional deps
pip install -e 'packages/markitdown[all]'
#   (with uv, use: uv pip install -e 'packages/markitdown[all]')

# 3. (Optional) Install the other packages editable too
pip install -e packages/markitdown-mcp
pip install -e packages/markitdown-ocr
pip install -e packages/markitdown-sample-plugin

# 4. Smoke test
markitdown packages/markitdown/tests/test_files/test.json
```

Optional extras (install individually instead of `[all]`): `[pdf]`, `[docx]`, `[xlsx]`, `[xls]`, `[pptx]`, `[outlook]`, `[audio-transcription]`, `[youtube-transcription]`, `[az-doc-intel]`, `[az-content-understanding]`.

### System binaries needed for some converters

- `ffmpeg` — audio transcription (`pydub`/`SpeechRecognition`). Set `FFMPEG_PATH` if not on PATH.
- `exiftool` — image/audio EXIF metadata. Set `EXIFTOOL_PATH` if not on PATH.
- On Windows these are not bundled; install via `winget`, `choco`, or `scoop` if you need those converters.

## Docker (optional, not compose)

There is **no `docker-compose.yml`**. Two standalone `Dockerfile`s exist:

- `./Dockerfile` — builds the `markitdown` CLI image (installs `ffmpeg` + `exiftool`, plus the sample plugin).
  ```bash
  docker build -t markitdown:latest .
  docker run --rm -v "$PWD/data:/workdir" markitdown:latest /workdir/file.pdf
  ```
- `packages/markitdown-mcp/Dockerfile` — builds the `markitdown-mcp` server image.
  ```bash
  docker build -t markitdown-mcp:latest -f packages/markitdown-mcp/Dockerfile .
  docker run --rm -i markitdown-mcp:latest            # STDIO
  docker run --rm -p 3001:3001 markitdown-mcp:latest --http --host 0.0.0.0 --port 3001
  ```

**Recommendation:** develop natively (fastest iteration, editable installs, debugger); use Docker only for reproducible CLI/MCP runs or when you can't install `ffmpeg`/`exiftool` locally.

## Build, test, and type-check

Tests use `pytest`. The core package's tests are parametrized over `packages/markitdown/tests/_test_vectors.py` (`GENERAL_TEST_VECTORS`, `DATA_URI_TEST_VECTORS`).

```bash
# Run core tests (from repo root)
cd packages/markitdown && python -m pytest

# Run a single package's tests
cd packages/markitdown-mcp && python -m pytest
cd packages/markitdown-ocr && python -m pytest
cd packages/markitdown-sample-plugin && python -m pytest

# Type-check (hatch envs are configured per package)
cd packages/markitdown && hatch run types:check
```

### Test conventions & gotchas

- **Remote tests are skipped in CI:** `skip_remote = True if os.environ.get("GITHUB_ACTIONS") else False`. Network-dependent tests (arxiv PDF, YouTube, blog HTML) run locally but are skipped on GitHub Actions.
- **LLM tests require `OPENAI_API_KEY`:** `skip_llm` is `True` unless the env var is set **and** the `openai` package imports. Don't expect LLM/image-caption/OCR tests to run without a key.
- **`exiftool` tests skip if the binary is missing:** `skip_exiftool = shutil.which("exiftool") is None`.
- Test fixtures live in `packages/markitdown/tests/test_files/`; expected outputs in `test_files/expected_outputs/`.
- CLI tests shell out via `subprocess.run(["python", "-m", "markitdown", ...])` — the package must be installed (editable is fine) for them to pass.

## Architecture & conventions

### Converter pattern (core)

Every format converter subclasses `DocumentConverter` (`packages/markitdown/src/markitdown/_base_converter.py`) and implements:

- `accepts(file_stream, stream_info, **kwargs) -> bool` — fast check based on `stream_info.mimetype` / `.extension` / `.url`. If you peek the stream, **reset the position** before returning (`file_stream.seek(cur_pos)`).
- `convert(file_stream, stream_info, **kwargs) -> DocumentConverterResult` — returns `{ markdown, title? }`.

Converters are registered with a **priority** (lower = tried first):
- `PRIORITY_SPECIFIC_FILE_FORMAT = 0.0` — format-specific converters (docx, pdf, xlsx, …).
- `PRIORITY_GENERIC_FILE_FORMAT = 10.0` — catch-alls (text/* mimetypes).

All built-in converters are imported in `converters/__init__.py` and registered in `_markitdown.py`. `StreamInfo` (`_stream_info.py`) carries mimetype/extension/charset/filename/url/local_path; `magika` + `charset_normalizer` guess it when unknown.

### Plugin pattern

Plugins are discovered via the `markitdown.plugin` entry-point group (see `markitdown-ocr` and `markitdown-sample-plugin` `pyproject.toml`). A plugin module must expose:

```python
__plugin_interface_version__ = 1

def register_converters(markitdown: MarkItDown, **kwargs) -> None:
    markitdown.register_converter(MyConverter(), priority=...)
```

- Plugins are **disabled by default**. Enable via CLI `--use-plugins` / `MARKITDOWN_ENABLE_PLUGINS=True`, or `MarkItDown(enable_plugins=True)`.
- Plugins receive the same `llm_client` / `llm_model` / `llm_prompt` kwargs the core uses for image descriptions — reuse them, don't invent new LLM wiring.
- To *replace* a built-in converter, register with priority `< 0.0` (e.g. `markitdown-ocr` uses `-1.0`) so it runs before the built-in at `0.0`.

### Adding a new converter (core)

1. Create `packages/markitdown/src/markitdown/converters/_<format>_converter.py` subclassing `DocumentConverter`.
2. Export it from `converters/__init__.py`.
3. Import + `register_converter(...)` in `_markitdown.py` with the appropriate priority.
4. Add optional deps to `pyproject.toml` `[project.optional-dependencies]` and to `all`.
5. Add a test vector in `tests/_test_vectors.py` and a fixture in `tests/test_files/`.

### Adding a new plugin

1. Create a new package under `packages/<name>/` with a `pyproject.toml` declaring `[project.entry-points."markitdown.plugin"]`.
2. Implement `_plugin.py` with `register_converters` + `__plugin_interface_version__ = 1`.
3. Depend on `markitdown>=0.1.0` (not a pinned exact version).
4. Mirror the structure of `packages/markitdown-sample-plugin`.

## Security notes (from README)

MarkItDown performs I/O with the privileges of the current process (like `open()` / `requests.get()`). In untrusted environments, sanitize inputs and call the narrowest function (`convert_local()`, `convert_stream()`) rather than the generic `convert()`. The MCP server binds to `localhost` by default — do not expose it to the network without understanding the implications.

## Key files

- `packages/markitdown/src/markitdown/_markitdown.py` — `MarkItDown` class, converter registration, plugin loading, `convert*` entry points.
- `packages/markitdown/src/markitdown/_base_converter.py` — `DocumentConverter` / `DocumentConverterResult` base classes.
- `packages/markitdown/src/markitdown/_stream_info.py` — `StreamInfo` dataclass.
- `packages/markitdown/src/markitdown/converters/__init__.py` — converter registry/exports.
- `packages/markitdown/tests/_test_vectors.py` — parametrized test fixtures.
- `packages/markitdown/src/markitdown/__main__.py` — CLI argument parsing.
- `packages/markitdown-mcp/src/markitdown_mcp/__main__.py` — MCP server entry.

## Further reading (link, don't duplicate)

- Root [`README.md`](README.md) — install, usage, optional extras, Azure Content Understanding.
- [`packages/markitdown-mcp/README.md`](packages/markitdown-mcp/README.md) — MCP server, Claude Desktop config, Docker.
- [`packages/markitdown-ocr/README.md`](packages/markitdown-ocr/README.md) — OCR plugin usage and custom prompts.
- [`packages/markitdown-sample-plugin/README.md`](packages/markitdown-sample-plugin/README.md) — reference plugin.