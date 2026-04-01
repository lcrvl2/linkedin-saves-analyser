# LinkedIn Saves Analyzer

Extract all your LinkedIn saved posts and get an actionable insights report — categories, freshness, actionability scores, top authors, and implementation priorities.

Works with your existing Claude subscription. No API key needed.

## What it does

1. **Scrapes** your LinkedIn "Saved Posts" page via Chrome remote debugging
2. **Discovers categories** from your actual content (no hardcoded taxonomy — adapts to every user)
3. **Categorizes** each post (category, freshness, actionability, key insight) using parallel Claude agents
4. **Generates** a Markdown report with category breakdown, top authors, implementation matrix, and positioning signals

## Output example

```
# LinkedIn Saves Analysis — 2026-03-25

| Metric              | Value |
|---------------------|-------|
| Posts analyzed       | 487   |
| Unique authors       | 203   |
| High-value posts     | 89    |
| Outdated posts       | 34    |
```

Full report includes: category-by-category insights, outdated posts to archive, top authors rankings, implementation priorities (now / 30 days / proof points), and a positioning signal reading.

---

## Two ways to use it

### Option A: Claude Code (CLI / IDE)

For developers comfortable with the terminal or IDE extensions.

### Option B: Claude Cowork (Web UI)

For non-technical users who prefer a visual interface. Same capabilities, no terminal needed.

---

## Setup

### 1. Chrome with Remote Debugging

Start Chrome with the remote debugging port open, then **log into LinkedIn** in that window.

**Windows:**
```bash
"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222
```

**macOS:**
```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222
```

**Linux:**
```bash
google-chrome --remote-debugging-port=9222
```

### 2a. Setup for Claude Code

Install [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (CLI or IDE extension).

Add the Chrome DevTools MCP server:

```bash
claude mcp add chrome-devtools -- npx @anthropic-ai/chrome-devtools-mcp@latest
```

Copy the skill file to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/linkedin-saves-analyzer
cp SKILL.md ~/.claude/skills/linkedin-saves-analyzer/SKILL.md
```

**Node.js 18+** is also required (for the aggregation step).

Then run:

```
/linkedin-saves-analyzer
```

Or say: *"analyze my LinkedIn saves"*

### 2b. Setup for Claude Cowork

1. Open [Claude Cowork](https://cowork.claude.ai)
2. Create a new project (or open an existing one)
3. Go to **Integrations** and add the **Chrome DevTools** MCP server
4. Upload `SKILL.md` to your project files (drag and drop, or use the file upload button)
5. Make sure Chrome is running with `--remote-debugging-port=9222` and you're logged into LinkedIn

Then type:

```
Analyze my LinkedIn saved posts using the linkedin-saves-analyzer skill
```

Claude will follow the SKILL.md workflow automatically: scrape your saves, propose categories based on your content, ask for confirmation, then run the full analysis and generate the report.

---

## How it works

```
LinkedIn Saves page (Chrome)
    |  Chrome DevTools MCP — scroll + JS extraction
    v
posts.json (raw: author, content, post_url)
    |  Claude reads a sample of ~50 posts
    v
Dynamic categories (8-12, proposed + confirmed by user)
    |  Split into batches of 50
    v
batches/batch_01.json ... batch_NN.json
    |  Parallel Claude agents categorize each batch
    v
analyzed/batch_XX_YY.json
    |  Node.js aggregation
    v
all_analyzed.json (enriched: category, freshness, actionability, key_insight)
    |  Report generation
    v
rapport-linkedin-saves-{YYYY-MM-DD}.md
```

### Dynamic categories

Categories are **not hardcoded**. The skill:
1. Samples ~50 of your saved posts
2. Identifies recurring themes and topics
3. Proposes 8-12 specific categories (e.g. "outbound-automation", "product-led-growth", "ai-for-content")
4. Asks you to confirm or adjust before running the analysis

This means the tool works for anyone — a marketer, a developer, a recruiter — without configuration.

### Scoring dimensions

- **Freshness**: `evergreen` / `current` / `outdated`
- **Actionability**: `high` / `medium` / `low`

### Parameters

| Parameter    | Default | Description                    |
|-------------|---------|--------------------------------|
| `language`  | `fr`    | Report language (`fr` or `en`) |
| `max_posts` | all     | Cap on posts to extract        |

---

## File structure

```
linkedin-saves-analyser/
  SKILL.md                        # Claude skill definition (the brain)
  README.md                       # This file (FR)
  README.en.md                    # English version
  .gitignore                      # Excludes generated data

  Generated per run (gitignored):
  posts.json                      # Raw extracted posts
  batches/                        # Split batches (50 posts each)
  analyzed/                       # Categorized batch results
  all_analyzed.json               # Aggregated analysis
  rapport-linkedin-saves-*.md     # Final report
```

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Chrome not reachable | Start Chrome with `--remote-debugging-port=9222` |
| Not logged into LinkedIn | Log in manually in the Chrome window, then retry |
| 0 posts extracted | LinkedIn changed CSS classes — Claude takes a screenshot and adapts selectors automatically |
| Infinite scroll stuck | Claude retries with alternative scroll strategies |

---

## Privacy

- Posts are processed locally in your Claude session
- No data is sent to external services beyond Claude's API
- Generated files stay on your machine

## License

MIT
