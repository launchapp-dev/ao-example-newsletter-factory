# Newsletter Factory — Workflow Plan

## Overview
Automated newsletter production pipeline. Configure RSS feeds and web sources, and AO will curate content, write newsletter sections, compose a full edition, review for quality, and output a ready-to-send markdown/HTML newsletter on a weekly schedule.

## Why This Example Matters
Newsletter production is tedious — reading dozens of sources, selecting what matters, writing summaries, composing a coherent edition. This shows AO automating an entire content pipeline using fetch MCP for real URL ingestion, filesystem for output, and scheduled workflows for recurring production cycles.

---

## Agents

| Agent | Model | Role |
|---|---|---|
| **content-curator** | claude-haiku-4-5 | Fast pass: fetch RSS feeds and URLs, extract articles, score relevance, filter and rank content |
| **section-writer** | claude-sonnet-4-6 | Write newsletter sections from curated content — summaries, commentary, analysis |
| **edition-composer** | claude-sonnet-4-6 | Assemble sections into a complete newsletter edition with intro, sections, and outro |
| **edition-reviewer** | claude-sonnet-4-6 | Review complete edition for quality, coherence, tone, and completeness |

## Phase Pipeline

```
1. fetch-sources  (command phase — curl RSS feeds, save raw content)
   ↓
2. curate-content  (agent — score, filter, rank articles)
   ↓
3. content-review  (decision: proceed/revise-sources/skip-edition)
   ↓
4. write-sections  (agent — write each newsletter section)
   ↓
5. compose-edition  (agent — assemble final newsletter)
   ↓
6. review-edition  (decision: publish/rework/delay)
   ↓  rework → back to write-sections
7. finalize-output  (command — generate HTML from markdown, save final edition)
```

### Phase Details

#### 1. fetch-sources (command phase)
- **What:** Fetch RSS feeds and URLs listed in config/sources.yaml, save raw content
- **How:** Command phase using curl to download RSS/Atom feeds and web pages
- **Command:** Bash script that reads config/sources.yaml, curls each feed URL, saves XML/HTML to data/raw-feeds/
- **Output:** Raw feed XML files in `data/raw-feeds/`, fetch log in `data/fetch-log.txt`

#### 2. curate-content (agent phase)
- **Agent:** content-curator (haiku — fast and cheap for filtering/scoring)
- **What:** Parse raw feed data, extract articles, score each for relevance against the newsletter topic defined in config/newsletter.yaml, rank and select top articles for inclusion
- **Input:** `data/raw-feeds/` directory, `config/newsletter.yaml` (topic, audience, tone preferences)
- **Output:** `data/curated-articles.json` with structure:
  ```json
  {
    "edition_date": "2026-03-31",
    "topic_focus": "AI & Developer Tools",
    "articles": [
      {
        "title": "...",
        "url": "...",
        "source": "...",
        "published": "...",
        "summary": "...",
        "relevance_score": 0.92,
        "section": "top-stories",
        "verdict": "include"
      }
    ],
    "sections": ["top-stories", "deep-dives", "tools-and-launches", "quick-links"],
    "stats": {"fetched": 85, "included": 12, "skipped": 73}
  }
  ```
- **MCP:** filesystem (read/write data files), fetch (retrieve full article text for top picks)

#### 3. content-review (agent phase — decision)
- **Agent:** content-curator
- **What:** Review curated content and decide whether there's enough quality material for an edition
- **Decision contract:** `{ verdict: "proceed" | "revise-sources" | "skip-edition", reasoning: "..." }`
- **Rules:**
  - >= 5 high-relevance articles → proceed
  - 2-4 articles → revise-sources (try fetching more from backup sources)
  - < 2 articles → skip-edition (not enough content this cycle)
- **Output:** `data/content-decision.json`
- **MCP:** filesystem

#### 4. write-sections (agent phase)
- **Agent:** section-writer (sonnet — needs good writing)
- **What:** For each section in curated-articles.json, write newsletter content:
  - **Top Stories:** 2-3 paragraph summaries with commentary
  - **Deep Dives:** Longer analysis of one key topic
  - **Tools & Launches:** Brief product/tool announcements
  - **Quick Links:** One-liner descriptions with URLs
- **Input:** `data/curated-articles.json`, `config/newsletter.yaml`, `templates/section-template.md`
- **Output:** Individual section files in `data/sections/` (e.g., `data/sections/top-stories.md`, `data/sections/deep-dives.md`)
- **MCP:** filesystem, fetch (retrieve full article text when summaries need more depth)

#### 5. compose-edition (agent phase)
- **Agent:** edition-composer (sonnet — needs coherent long-form assembly)
- **What:** Assemble all sections into a single newsletter edition:
  - Write a personalized intro paragraph (references current trends, hooks the reader)
  - Arrange sections in optimal reading order
  - Write transitional copy between sections
  - Write an outro with call-to-action
  - Generate 3-5 subject line options ranked by expected open rate
- **Input:** `data/sections/*`, `config/newsletter.yaml`, `data/curated-articles.json`
- **Output:**
  - `output/edition-YYYY-MM-DD.md` — complete newsletter in markdown
  - `output/subject-lines.txt` — ranked subject line options
  - `data/edition-metadata.json` — word count, section count, article count, links count
- **MCP:** filesystem, sequential-thinking (reason about section ordering and subject lines)

#### 6. review-edition (agent phase — decision)
- **Agent:** edition-reviewer
- **What:** Review the complete edition for:
  - Coherence — does the intro match the content? Do sections flow logically?
  - Quality — are summaries accurate? Is the writing engaging?
  - Completeness — are all curated articles represented? Are links working?
  - Tone — matches the newsletter's defined voice in config/newsletter.yaml?
  - Length — within target word count range?
- **Decision contract:** `{ verdict: "publish" | "rework", reasoning: "...", issues: ["..."] }`
- **Rework routing:** On "rework", go back to write-sections with reviewer feedback
- **Max rework:** 3 attempts
- **MCP:** filesystem

#### 7. finalize-output (command phase)
- **What:** Convert markdown newsletter to HTML using pandoc, copy final artifacts to output/
- **Command:** `pandoc output/edition-YYYY-MM-DD.md -o output/edition-YYYY-MM-DD.html --template=templates/email.html --metadata title="Newsletter"`
- **Output:** HTML edition in `output/`, archive copy in `output/archive/`

## MCP Servers Needed

| Server | Package | Purpose |
|---|---|---|
| filesystem | `@modelcontextprotocol/server-filesystem` | Read/write all data, config, and output files |
| fetch | `@modelcontextprotocol/server-fetch` | Retrieve full article text from URLs for deeper summaries |
| sequential-thinking | `@modelcontextprotocol/server-sequential-thinking` | Structured reasoning for section ordering and subject line generation |

All three are existing npm packages — no custom builds needed.

## Directory Structure

```
examples/newsletter-factory/
├── .ao/workflows/
│   ├── agents.yaml
│   ├── phases.yaml
│   ├── workflows.yaml
│   ├── mcp-servers.yaml
│   └── schedules.yaml
├── config/
│   ├── newsletter.yaml       # Newsletter name, topic, audience, tone, word count targets
│   └── sources.yaml           # RSS feed URLs, web source URLs, backup sources
├── data/                      # Working data (intermediate outputs)
│   ├── raw-feeds/             # Downloaded RSS/Atom XML files
│   ├── sections/              # Individual section markdown files
│   └── .gitkeep
├── output/                    # Final artifacts
│   ├── archive/               # Past editions
│   └── .gitkeep
├── templates/
│   ├── section-template.md    # Template for individual sections
│   ├── edition-template.md    # Template for full edition layout
│   └── email.html             # Pandoc HTML template for email-ready output
├── sample-data/
│   ├── sample-feed.xml        # Example RSS feed for demo/testing
│   └── sample-curated.json    # Example curated articles output
├── CLAUDE.md                  # Project-level instructions
└── README.md                  # User-facing docs
```

## Workflow Features Demonstrated

- **Command phases** — curl for RSS feed fetching, pandoc for markdown→HTML conversion
- **Multi-agent pipeline** — haiku for fast content curation, sonnet for writing and review
- **Decision contracts** — content sufficiency check (proceed/revise/skip), edition quality (publish/rework)
- **Phase routing with rework** — reviewer can send edition back for rewriting
- **Scheduled workflows** — weekly cron trigger for recurring newsletter production
- **Fetch MCP** — real URL content ingestion for article summaries
- **Output contracts** — structured JSON for curated content, formatted markdown for newsletter output

## Sample Input/Output

**Input:** config/sources.yaml with RSS feeds
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
  - name: "The Verge"
    url: "https://www.theverge.com/rss/index.xml"
    type: rss
    priority: medium
  - name: "GitHub Trending"
    url: "https://github.com/trending"
    type: web
    priority: low
```

**Output:** A complete newsletter edition in markdown and HTML, with:
- 3-5 subject line options
- Personalized intro paragraph
- 2-3 top story summaries with commentary
- 1 deep dive analysis
- 4-6 tool/launch briefs
- 8-10 quick links with one-liners
- Outro with CTA
