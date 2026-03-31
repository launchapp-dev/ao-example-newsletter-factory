# Newsletter Factory

Automated newsletter production pipeline — fetch RSS feeds, curate content, write sections, compose a polished edition, review for quality, and output HTML ready to send.

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                   Newsletter Factory Pipeline                │
└─────────────────────────────────────────────────────────────┘

  [CRON: Every Monday 9am]
          │
          ▼
  ┌───────────────┐
  │ fetch-sources │  curl RSS feeds from config/sources.yaml
  │  (command)    │  → data/raw-feeds/*.xml
  └───────┬───────┘
          │
          ▼
  ┌───────────────┐
  │curate-content │  Parse XML, score relevance, assign sections
  │   (haiku)     │  → data/curated-articles.json
  └───────┬───────┘
          │
          ▼
  ┌───────────────┐
  │content-review │  Decision: enough quality content?
  │   (haiku)     │
  └───────┬───────┘
          │
     ┌────┴─────┬──────────────┐
  proceed   revise-sources  skip-edition
     │          │                │
     │    ← fetch-sources    finalize-output
     │
     ▼
  ┌───────────────┐
  │ write-sections│  Write top-stories, deep-dives, tools, quick-links
  │   (sonnet)    │  → data/sections/*.md
  └───────┬───────┘     ▲
          │             │ rework (up to 3x)
          ▼             │
  ┌───────────────┐     │
  │compose-edition│  Assemble full edition + subject lines
  │   (sonnet)    │  → output/edition-YYYY-MM-DD.md
  └───────┬───────┘
          │
          ▼
  ┌───────────────┐
  │review-edition │  Quality gate: publish or rework?
  │   (sonnet)    │──────────────────────────────────┘
  └───────┬───────┘
     publish
          │
          ▼
  ┌───────────────┐
  │finalize-output│  pandoc → HTML, archive edition
  │  (command)    │  → output/edition-YYYY-MM-DD.html
  └───────────────┘
```

## Quick Start

```bash
cd examples/newsletter-factory

# Edit newsletter topic and RSS sources
vim config/newsletter.yaml
vim config/sources.yaml

# Run once
ao workflow run produce-newsletter

# Or start recurring weekly production
ao daemon start
```

## Agents

| Agent | Model | Role |
|---|---|---|
| **content-curator** | claude-haiku-4-5 | Parse RSS feeds, score article relevance (0.0–1.0), assign to sections, decide content sufficiency |
| **section-writer** | claude-sonnet-4-6 | Write top stories, deep dives, tools & launches, and quick links sections |
| **edition-composer** | claude-sonnet-4-6 | Assemble sections into a complete edition, write intro/outro, generate subject lines |
| **edition-reviewer** | claude-sonnet-4-6 | Review coherence, quality, completeness, tone, length, and links before publishing |

## AO Features Demonstrated

- **Command phases** — curl for RSS fetching, pandoc for markdown→HTML conversion
- **Multi-agent pipeline** — haiku for fast/cheap curation, sonnet for quality writing
- **Decision contracts** — `content-review` decides proceed/revise-sources/skip-edition
- **Rework loops** — reviewer sends edition back to `write-sections` with feedback (up to 3x)
- **Phase routing** — `revise-sources` loops back to `fetch-sources`; `skip-edition` jumps to finalize
- **Scheduled workflows** — weekly cron trigger (Monday 9am) for recurring production
- **Fetch MCP** — retrieve full article text from URLs for deeper summarization
- **Sequential-thinking MCP** — reason about section ordering and subject line effectiveness

## Directory Structure

```
newsletter-factory/
├── .ao/workflows/
│   ├── agents.yaml          # content-curator, section-writer, edition-composer, edition-reviewer
│   ├── phases.yaml          # fetch-sources, curate-content, content-review, write-sections,
│   │                        # compose-edition, review-edition, finalize-output
│   ├── workflows.yaml       # produce-newsletter workflow
│   ├── mcp-servers.yaml     # filesystem, fetch, sequential-thinking
│   └── schedules.yaml       # weekly-newsletter (Monday 9am)
├── config/
│   ├── newsletter.yaml      # Newsletter name, topic, audience, tone, word count targets
│   └── sources.yaml         # RSS feed URLs and web sources
├── data/                    # Intermediate working data (git-ignored)
│   └── raw-feeds/           # Downloaded RSS XML files
├── output/                  # Final editions (git-ignored)
│   └── archive/             # Past editions
├── templates/
│   ├── section-template.md  # Template for individual sections
│   ├── edition-template.md  # Template for full edition layout
│   └── email.html           # Pandoc HTML template for email-ready output
└── sample-data/
    ├── sample-feed.xml       # Example RSS feed for testing
    └── sample-curated.json   # Example curated articles output
```

## Requirements

### Tools
- `curl` — for fetching RSS feeds (standard on most systems)
- `pandoc` — for markdown→HTML conversion (optional; HTML step is skipped if absent)

### API Keys
None required for the default configuration. The pipeline uses only:
- Public RSS feeds (no auth)
- MCP servers that run locally via npx

### MCP Servers (auto-installed via npx)
- `@modelcontextprotocol/server-filesystem` — read/write data, config, output files
- `mcp-fetch-server` — fetch full article text from URLs
- `@modelcontextprotocol/server-sequential-thinking` — structured reasoning for composition

## Configuration

### config/newsletter.yaml
```yaml
name: "The Weekly Digest"
topic: "AI & Developer Tools"
audience: "Software engineers and technical founders"
tone: "Informative, direct, slightly opinionated — no hype"
word_count_target:
  min: 1000
  max: 2500
```

### config/sources.yaml
```yaml
sources:
  - name: "Hacker News Best"
    url: "https://hnrss.org/best"
    type: rss
    priority: high
  - name: "TechCrunch AI"
    url: "https://techcrunch.com/category/artificial-intelligence/feed/"
    type: rss
    priority: medium
```

## Output

Each run produces:
- `output/edition-YYYY-MM-DD.md` — complete newsletter in markdown
- `output/edition-YYYY-MM-DD.html` — email-ready HTML (if pandoc is installed)
- `output/subject-lines.txt` — 3–5 ranked subject line options
- `output/archive/` — archived copies of all past editions
