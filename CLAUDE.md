# CLAUDE.md

This file provides guidance for AI assistants working in this repository.

## Project Overview

**Company Jobs Consolidator** — a single-file Python 3 CLI tool that fetches and consolidates open job listings from multiple companies by querying public ATS (Applicant Tracking System) job board APIs. No authentication is required; all APIs are public.

**Supported ATS platforms:**
- Greenhouse (`boards-api.greenhouse.io`)
- Lever (`api.lever.co`)
- Ashby (`api.ashbyhq.com`)

## Repository Structure

```
Sandbox/
├── jobs.py           # Entire application (369 lines)
├── requirements.txt  # Single dependency: requests>=2.31.0
└── CLAUDE.md         # This file
```

This is intentionally a single-file project. Do not create additional modules or packages unless there is a compelling reason.

## Setup

```bash
pip install -r requirements.txt
```

Requires Python 3.9+. No build step — run directly.

## Running the Tool

```bash
# Query companies by name (tries Greenhouse → Lever → Ashby in order)
python jobs.py stripe notion linear

# Read company names from a file (one per line)
python jobs.py --file companies.txt

# Export results
python jobs.py --export-csv results.csv stripe notion
python jobs.py --export-json results.json stripe notion

# Demo mode (no internet required, uses hardcoded sample data)
python jobs.py --demo

# Interactive mode (no args — prompts for company names)
python jobs.py
```

## Code Architecture

The file is divided into clearly commented sections:

| Section | Lines | Purpose |
|---|---|---|
| Data model | 28–39 | `Job` dataclass |
| Helpers | 42–62 | `slugify()`, `slug_variants()` |
| ATS fetchers | 69–144 | `try_greenhouse()`, `try_lever()`, `try_ashby()` |
| Orchestration | 151–164 | `FETCHERS` list, `fetch_jobs()` |
| Output | 171–239 | `print_results()`, `export_csv()`, `export_json()` |
| CLI entry point | 246–369 | `main()` with argparse |

### Key Design Decisions

- **Slug discovery**: Each fetcher calls `slug_variants()` to generate up to 3 slug forms (base, no-dashes, underscored) and tries each one, stopping on the first 200 response. This handles inconsistent naming across companies.
- **Fetcher fallback**: `fetch_jobs()` tries all three ATS providers in order. The first one that returns a non-`None` result wins. A `None` return means "this ATS doesn't know this company"; an empty list `[]` means "found the company but no open roles".
- **Return type**: `fetch_jobs()` returns `tuple[list[Job], str]` — the job list and the source name (e.g., `"Greenhouse"`).
- **`all_jobs` structure**: The main data structure is `dict[str, tuple[list[Job], str]]`, mapping company name → (jobs, source).

## Code Conventions

- **Naming**: `snake_case` for functions/variables, `SCREAMING_SNAKE_CASE` for module-level constants (`FETCHERS`), `PascalCase` for classes (`Job`).
- **Type hints**: All function signatures are fully annotated. Use `Optional[T]` for nullable fields (not `T | None`), keeping compatibility with the existing style.
- **Error handling**: ATS fetchers catch `(requests.RequestException, ValueError)` and `continue` to the next slug variant. Do not silently swallow unexpected exceptions.
- **Timeouts**: All `requests.get()` calls use `timeout=10`. Maintain this for any new HTTP calls.
- **Output format**: Console output uses 72-character `=` separators and `•` bullet points. Preserve this style for new output.
- **No external formatters/linters are configured** — follow PEP 8 manually.

## Adding a New ATS Provider

1. Add a `try_<ats_name>(company_name: str) -> Optional[list[Job]]` function in the "ATS fetchers" section following the same pattern (slug variants loop, timeout, error handling).
2. Append `(try_<ats_name>, "<ATS Name>")` to the `FETCHERS` list in the "Orchestration" section.
3. Update the module docstring and `argparse` description to mention the new provider.

## Testing

There are currently no automated tests. When adding tests:
- Use `pytest` as the test runner.
- Place tests in a `tests/` directory.
- The `--demo` flag provides a fast offline path useful for testing output formatting without network calls.

## Dependencies

Only one external dependency: `requests>=2.31.0`. Do not introduce additional dependencies without strong justification.
