# Newsletter Factory — Project Context

## What This Is

An automated newsletter production pipeline built on AO. It runs on a weekly schedule and produces a complete, ready-to-send newsletter edition from RSS feeds and web sources.

## Workflow Summary

1. **fetch-sources** (command) — curl RSS feeds from `config/sources.yaml`, save XML to `data/raw-feeds/`
2. **curate-content** (content-curator/haiku) — parse feeds, score relevance, write `data/curated-articles.json`
3. **content-review** (content-curator/haiku, decision) — decide: proceed / revise-sources / skip-edition
4. **write-sections** (section-writer/sonnet) — write markdown files in `data/sections/`
5. **compose-edition** (edition-composer/sonnet) — assemble `output/edition-YYYY-MM-DD.md` + subject lines
6. **review-edition** (edition-reviewer/sonnet, decision) — publish or rework (max 3x)
7. **finalize-output** (command) — pandoc markdown→HTML, archive to `output/archive/`

## Key Files

| File | Purpose |
|---|---|
| `config/newsletter.yaml` | Newsletter identity: name, topic, audience, tone, word count targets |
| `config/sources.yaml` | RSS feed URLs and web sources to pull from |
| `data/curated-articles.json` | Scored, filtered articles with section assignments |
| `data/sections/*.md` | Individual section content written by section-writer |
| `data/review-feedback.md` | Reviewer feedback during rework cycles (delete after publish) |
| `data/edition-date.txt` | Today's date (written by fetch-sources, used downstream) |
| `output/edition-YYYY-MM-DD.md` | Final newsletter in markdown |
| `output/subject-lines.txt` | Ranked subject line options |

## Design Decisions

- **Haiku for curation** — scoring and filtering is high-volume/low-creativity work; haiku is fast and cheap
- **Sonnet for writing** — section writing and edition assembly require quality prose
- **Rework loop** — reviewer can send back to write-sections with specific feedback (up to 3 attempts)
- **skip-edition path** — if curated content is insufficient, finalize-output runs immediately (graceful skip)
- **No custom MCP needed** — fetch + filesystem + sequential-thinking cover all requirements

## Running Locally

```bash
# One-time run
ao workflow run produce-newsletter

# Start the weekly schedule (Monday 9am)
ao daemon start

# Watch live
ao daemon stream --pretty
```

## Adding Sources

Edit `config/sources.yaml` to add RSS feeds or web pages. The curl-based fetcher handles both RSS/Atom XML and HTML pages. The curator agent parses both formats.

## Changing Topic/Tone

Edit `config/newsletter.yaml`. The curator uses the `topic` field for relevance scoring, and the section-writer uses the `tone` field to match writing style.
