# Wiki Schema

This wiki organizes research for the X by 2 Superpowers presentation. The LLM maintains it; humans read it.

## Layers
- `raw/` (repo root) — immutable source notes. Never modify.
- `wiki/` — LLM-maintained pages (this directory).
- `wiki/SCHEMA.md` — this file: conventions and workflows.

## Conventions
- One page per topic, kebab-case filenames (e.g. `agentskills-standard.md`).
- Every page starts with a one-line summary, then sections. End with a `## Sources` section listing URLs/files consulted.
- Cross-reference other pages with relative markdown links.
- Flag unverified or contradictory claims with **[UNVERIFIED]** or **[CONTRADICTS: page]** inline.
- Claims destined for presentation slides must be verifiable — prefer primary sources.

## Special files
- `index.md` — catalog of all pages with one-line summaries, grouped by category. Update on every ingest.
- `log.md` — append-only chronological record. Entry format: `## [YYYY-MM-DD] <operation> | <title>`.

## Operations
- **Ingest**: research lands as a new/updated page + index update + log entry.
- **Query**: answers synthesized from pages; valuable answers get filed back as pages.
- **Lint**: periodic check for contradictions, stale claims, orphan pages, gaps.
