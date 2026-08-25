# Document Summarizer

A CLI tool that takes a `.txt` or `.pdf` file and produces a structured summary
using the OpenAI API, Pydantic structured outputs, and XML-tagged prompts.

## Setup

```bash
uv add openai pydantic pypdf
export OPENAI_API_KEY="sk-..."
```

## Usage

```bash
# Summarize a .txt file
uv run summarize.py report.txt

# Summarize a .pdf file
uv run summarize.py contract.pdf

# Choose detail level
uv run summarize.py report.txt --detail brief
uv run summarize.py report.txt --detail standard   # default
uv run summarize.py report.txt --detail detailed

# Output as JSON (structured fields only)
uv run summarize.py report.txt --json

# Write a formatted Markdown file
uv run summarize.py report.txt --output summary.md

# Combined cross-document summary (extension challenge)
uv run summarize.py q1.txt q2.txt q3.txt
```

A sample document, `test.txt`, is included for a quick smoke test:

```bash
uv run summarize.py test.txt
uv run summarize.py test.txt --detail brief
uv run summarize.py test.txt --detail detailed
uv run summarize.py test.txt --json
```

## How it maps to the spec

| Requirement | Where it lives |
|---|---|
| Reads `.txt` / `.pdf` | `read_document()` — plain `open()` for `.txt`, `pypdf.PdfReader` for `.pdf` |
| Rejects unsupported types before calling the API | `read_document()` raises `ValueError` before any client is created; `main()` validates every file up front |
| `DocumentSummary` Pydantic model | `title`, `document_type`, `estimated_word_count`, `summary`, `key_points`, `action_items` |
| Detail level via system prompt, not model swap | `build_system_prompt()` embeds the detail guidance; the model choice is untouched by `--detail` |
| XML tags separate document from instructions | `build_user_message()` wraps the text in `<document>...</document>` |
| Temperature 0.3 | `TEMPERATURE = 0.3`, passed on every call |
| Retry logic | `call_with_retry()` — exponential backoff on rate limits, connection errors, and 5xx errors |
| Token count + cost printed every run | `usage_info` computed in `summarize()`, printed in both the formatted and (footer of) markdown output |
| `--json` prints raw Pydantic model | `summary.model_dump_json(indent=2)` |
| Long documents (>10,000 words) switch to `gpt-4o` with a warning | `choose_model()` + the printed `⚠ Document is long...` message |
| API errors print a helpful message, not a stack trace | `main()` catches `APIError` / `APIConnectionError` / `RateLimitError` / `ValueError` and prints a one-line message |

## Extension challenges implemented

- **Multiple files → combined cross-document summary**: pass more than one file
  path and they're concatenated (each tagged with its source filename) into a
  single document before summarization.
- **Output to file**: `--output summary.md` writes a formatted Markdown version
  of the summary alongside the console output.

## Not yet implemented

- **Chunking / map-reduce for very long documents**: currently, documents over
  10,000 words are handled by switching to `gpt-4o` (per the spec) rather than
  being split into chunks and recursively summarized. Chunking is listed as an
  extension challenge and would be a good next step — split `combined_text` into
  overlapping chunks, call `summarize()` on each, then run one more summarization
  pass over the concatenated chunk summaries.
