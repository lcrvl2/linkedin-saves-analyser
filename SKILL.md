---
name: linkedin-saves-analyzer
description: >
  Scrape and analyze LinkedIn saved posts via Chrome DevTools MCP. Use when: (1) A user wants
  to extract all their saved posts from LinkedIn's "My Items" page, (2) Generate an insights
  report from saved posts (top authors, trending topics, business signals, strong opinions).
  Requires Chrome with remote debugging enabled and the user logged into LinkedIn.
  Triggers on "analyze my LinkedIn saves", "saved posts report", or "/linkedin-saves-analyzer".
---

# LinkedIn Saved Posts Analyzer

Extracts all posts from LinkedIn's saved items page and generates an insights report — without any API key or extra cost, using your existing Claude subscription.

## Prerequisites

```bash
# Start Chrome with remote debugging (see README.md for platform-specific commands)
# Then: make sure you're logged into LinkedIn in that Chrome window
```

## Workflow

```
Open LinkedIn Saves page
  -> Scroll and extract all posts
    -> Deduplicate
      -> Discover categories from content
        -> Analyze with Claude
          -> Generate Markdown report
```

---

## Step-by-Step Guide

### 1. Verify Chrome Connection

1. Call `mcp__chrome-devtools__list_pages` to see open tabs
2. If a LinkedIn tab exists, select it with `mcp__chrome-devtools__select_page`
3. If not, open a new tab and navigate: `mcp__chrome-devtools__navigate_page` to `https://www.linkedin.com/my-items/saved-posts/`
4. Take a screenshot to confirm the page loaded correctly
5. **Checkpoint**: "LinkedIn Saves page visible. Starting extraction."

### 2. Extract Posts (Scroll Loop)

Repeat until no new posts are found (3 consecutive scrolls with no new content):

**2a. Extract visible posts**

Run `mcp__chrome-devtools__evaluate_script` with:

```javascript
(() => {
  const results = [];
  const containers = document.querySelectorAll(
    '.scaffold-finite-scroll__content > div, ' +
    '[data-finite-scroll-hotspot-bottom] ~ div > div, ' +
    '.feed-shared-update-v2, ' +
    '[data-id]'
  );

  containers.forEach(el => {
    const authorEl = el.querySelector(
      '.update-components-actor__name, ' +
      '.feed-shared-actor__name, ' +
      '.update-components-actor__title'
    );
    const contentEl = el.querySelector(
      '.feed-shared-update-v2__description, ' +
      '.update-components-text, ' +
      '.break-words span[dir="ltr"]'
    );
    const linkEl = el.querySelector('a[href*="/posts/"], a[href*="activity"]');
    const authorLinkEl = el.querySelector('a[href*="/in/"]');

    if (contentEl && contentEl.innerText.trim().length > 20) {
      results.push({
        author: authorEl ? authorEl.innerText.trim().split('\n')[0] : 'Unknown',
        author_url: authorLinkEl ? authorLinkEl.href : null,
        content: contentEl.innerText.trim(),
        post_url: linkEl ? linkEl.href : null
      });
    }
  });

  const seen = new Set();
  return results.filter(p => {
    const key = p.content.slice(0, 100);
    if (seen.has(key)) return false;
    seen.add(key);
    return true;
  });
})()
```

**2b. Scroll down**

Run `mcp__chrome-devtools__evaluate_script` with:
```javascript
window.scrollBy(0, 1500);
```

**2c. Wait for load**

Call `mcp__chrome-devtools__wait_for` with selector `.feed-shared-update-v2` or wait 2 seconds.

**2d. Check for new content**

Compare post count before/after scroll. If count didn't increase after 3 scrolls, stop loop.

**Progress update**: After every 10 posts extracted, report: "Extracted X posts so far..."

**Fallback**: If the JavaScript selectors return 0 results, take a screenshot and visually inspect the page structure. Adapt selectors based on what you see. LinkedIn changes its CSS classes periodically — use the screenshot to identify the correct containers.

### 3. Deduplicate and Consolidate

Merge all extracted posts across scroll iterations. Remove duplicates by matching the first 100 characters of content. Report final count: "Total: X unique posts extracted from Y authors."

### 4. Discover Categories (Dynamic)

**This step generates categories tailored to the user's actual saved content — no hardcoded taxonomy.**

**4a. Sample the content**

Take a random sample of ~50 posts (or all posts if total is 50 or less). Read through them to identify recurring themes, topics, and professional interests.

**4b. Generate 8-12 categories**

Based on the sample, propose 8-12 categories that:
- Reflect the actual topics present in the saved posts
- Are specific enough to be useful (not "business" or "marketing" — more like "outbound-automation" or "product-led-growth")
- Are mutually exclusive (a post should clearly fit in one category)
- Include an `other` catch-all for outliers

**4c. Present categories for confirmation**

Show the proposed categories to the user with a 1-line description each. Ask: "These categories look good? I can adjust before running the full analysis."

Wait for user confirmation before proceeding. If the user adjusts, update the category list accordingly.

### 5. Categorize Posts (Chunked — Parallel Agents)

**If total posts are 150 or less**: analyze directly in memory, skip to step 5b.

**If total posts are more than 150**: use parallel agents to categorize at scale.

**5a. Split into batches of 50**

Save the full posts array to a local JSON file:
```
{working_dir}/linkedin-saves-analyzer/posts.json
```

Then split into batch files of 50 posts each:
```
batches/batch_01.json  (posts 1-50)
batches/batch_02.json  (posts 51-100)
...
```

**5b. Launch parallel categorization agents**

For each pair of batches, launch a background Agent (subagent_type: general-purpose, model: sonnet) with this prompt:

> Read batch files [batch_XX.json] and [batch_YY.json]. For each post, assign:
> - `category`: one of [INSERT THE CONFIRMED CATEGORY LIST HERE]
> - `freshness`: `evergreen`, `current`, or `outdated` (from a current-year perspective)
> - `actionability`: `high`, `medium`, or `low`
> - `key_insight`: 1 sentence max — the most concrete, actionable takeaway
>
> Output a JSON array. Write to: `analyzed/batch_XX_YY.json`

Launch all agents in parallel. Wait for all to complete.

**5c. Aggregate results**

Use Node.js to merge all `analyzed/batch_*.json` files into a single dataset:
```javascript
const fs = require('fs'), path = require('path');
const files = fs.readdirSync('./analyzed').filter(f => f.endsWith('.json'));
let all = [];
for (const f of files) all = all.concat(JSON.parse(fs.readFileSync(path.join('./analyzed', f))));
fs.writeFileSync('./all_analyzed.json', JSON.stringify(all, null, 2));
console.log('Total:', all.length);
```

**5d. Compute stats**

```javascript
const all = JSON.parse(fs.readFileSync('./all_analyzed.json'));

// Category counts
const cats = {}, fresh = {}, act = {};
for (const p of all) {
  cats[p.category] = (cats[p.category] || 0) + 1;
  fresh[p.freshness] = (fresh[p.freshness] || 0) + 1;
  act[p.actionability] = (act[p.actionability] || 0) + 1;
}

// High-value posts per category (high actionability + not outdated)
const highValue = all.filter(p => p.actionability === 'high' && p.freshness !== 'outdated');
const hvByCat = {};
for (const p of highValue) {
  if (!hvByCat[p.category]) hvByCat[p.category] = [];
  hvByCat[p.category].push(p);
}

// Top authors by post count
const authors = {};
for (const p of all) authors[p.author] = (authors[p.author] || 0) + 1;
const topAuthors = Object.entries(authors).sort((a,b) => b[1]-a[1]).slice(0, 15);

// Top authors by high-value posts
const hvAuthors = {};
for (const p of highValue) hvAuthors[p.author] = (hvAuthors[p.author] || 0) + 1;
const topHVAuthors = Object.entries(hvAuthors).sort((a,b) => b[1]-a[1]).slice(0, 10);
```

### 6. Generate Report

Produce a Markdown report saved to `rapport-linkedin-saves-{YYYY-MM-DD}.md` with this structure:

```markdown
# LinkedIn Saves Analysis — {YYYY-MM-DD}

## Overview

| Metric | Value |
|--------|-------|
| Posts analyzed | N |
| Unique authors | X |
| High-value posts (high actionability + not outdated) | N |
| Outdated posts to archive | N |
| Dominant category | [category] (N posts) |

**Category distribution:**

| Category | Posts | High-value |
|----------|-------|-----------|
| [for each category, sorted by volume descending] | N | N |

**Freshness:** Evergreen: N% / Current: N% / Outdated: N%

---

## Category by category

For each category with 5+ posts:

### [Category name] (N posts, N high-value)

> **Signal**: [1 sentence on what this category reveals about the user's interests]

**Top actionable insights:**

- **[Author]**: [key_insight]
- **[Author]**: [key_insight]
- ... (top 6-8 high-value posts from this category)

---

## Outdated posts — To archive (N posts)

| Theme | Posts | Why |
|-------|-------|-----|
| [group by reason for obsolescence] | N | [explanation] |

---

## Top authors

**By save volume:**

| Author | Saved posts |
|--------|-------------|
| [top 10] | N |

**By high-value posts:**

| Author | High-value posts | Territory |
|--------|-----------------|-----------|
| [top 10] | N | [dominant category] |

---

## Implementation matrix

### Implement now
[3-5 high-value + current insights, with concrete action]

### Implement in 30 days
[3-5 more structural insights, requiring more time]

### Use as proof points / arguments
[2-3 notable stats or facts from posts, usable in content or sales]

---

## Positioning signal

[3-5 sentences: what the saves as a whole reveal about the user's watchlist profile, convictions, and territory. Not a summary — a reading of the intellectual profile behind the saves.]
```

**Report language**: Use the language parameter (default: fr). If fr, write the report in French. If en, write in English. The template above shows the English structure — adapt headers and content to the chosen language.

### 7. Save and Deliver Report

1. Write the report to `rapport-linkedin-saves-{YYYY-MM-DD}.md` in the working directory using the Write tool
2. **Done**: "Report generated: rapport-linkedin-saves-{DATE}.md — N posts analyzed, N high-value, N to archive."

---

## Parameters

No parameters required. The workflow is self-contained.

Optional at start:
```
language:   # "fr" (default) or "en" — language of the final report
max_posts:  # Maximum posts to extract (default: all)
```

---

## Error Handling

| Error | Action |
|-------|--------|
| Chrome not reachable | Ask user to start Chrome with `--remote-debugging-port=9222` |
| Not logged into LinkedIn | Ask user to log in, then retry Step 1 |
| 0 posts extracted | Take screenshot, inspect DOM visually, adapt selectors |
| Page not loading | Wait 3s, scroll slightly, retry extraction |
| Infinite scroll not triggering | Try `document.querySelector('.scaffold-finite-scroll__content').scrollIntoView()` then scroll |

---

## Notes

- **No API key needed** — analysis runs with your existing Claude subscription
- **Privacy** — posts are processed in memory only, never sent to external services
- **LinkedIn changes** — if selectors break, Claude uses screenshots to visually identify the new DOM structure and adapts
- **Volume** — scales to 1000+ posts via parallel agent chunking (batches of 50, agents launched in parallel)
- **Categories are dynamic** — generated from your actual saved content, not hardcoded. Every user gets categories that match their interests.
