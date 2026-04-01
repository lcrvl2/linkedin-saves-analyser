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
  → Scroll and extract all posts
    → Deduplicate
      → Analyze with Claude
        → Generate Markdown report
```

---

## Step-by-Step Guide

### 1. Verify Chrome Connection

1. Call `mcp__chrome-devtools__list_pages` to see open tabs
2. If a LinkedIn tab exists, select it with `mcp__chrome-devtools__select_page`
3. If not, open a new tab and navigate: `mcp__chrome-devtools__navigate_page` → `https://www.linkedin.com/my-items/saved-posts/`
4. Take a screenshot to confirm the page loaded correctly
5. **Checkpoint**: "LinkedIn Saves page visible. Starting extraction."

### 2. Extract Posts (Scroll Loop)

Repeat until no new posts are found (3 consecutive scrolls with no new content):

**2a. Extract visible posts**

Run `mcp__chrome-devtools__evaluate_script` with:

```javascript
(() => {
  const results = [];
  // Try multiple possible containers for saved posts
  const containers = document.querySelectorAll(
    '.scaffold-finite-scroll__content > div, ' +
    '[data-finite-scroll-hotspot-bottom] ~ div > div, ' +
    '.feed-shared-update-v2, ' +
    '[data-id]'
  );

  containers.forEach(el => {
    // Author name
    const authorEl = el.querySelector(
      '.update-components-actor__name, ' +
      '.feed-shared-actor__name, ' +
      '.update-components-actor__title'
    );
    // Post text
    const contentEl = el.querySelector(
      '.feed-shared-update-v2__description, ' +
      '.update-components-text, ' +
      '.break-words span[dir="ltr"]'
    );
    // Post URL
    const linkEl = el.querySelector('a[href*="/posts/"], a[href*="activity"]');
    // Author profile URL
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

  // Deduplicate by content fingerprint (first 100 chars)
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

Compare post count before/after scroll. If count didn't increase after 3 scrolls → stop loop.

**Progress update**: After every 10 posts extracted, report: "Extracted X posts so far..."

**Fallback**: If the JavaScript selectors return 0 results, take a screenshot and visually inspect the page structure. Adapt selectors based on what you see. LinkedIn changes its CSS classes periodically — use the screenshot to identify the correct containers.

### 3. Deduplicate and Consolidate

Merge all extracted posts across scroll iterations. Remove duplicates by matching the first 100 characters of content. Report final count: "Total: X unique posts extracted from Y authors."

### 4. Categorize Posts (Chunked — Parallel Agents)

**If total posts ≤ 150** : analyze directly in memory, skip to step 4b.

**If total posts > 150** : use parallel agents to categorize at scale.

**4a. Split into batches of 50**

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

**4b. Launch parallel categorization agents**

For each pair of batches, launch a background Agent (subagent_type: general-purpose, model: sonnet) with this prompt:

> Read batch files [batch_XX.json] and [batch_YY.json]. For each post, assign:
> - `category`: one of `claude-code-agents`, `ai-seo`, `outbound-automation`, `content-systems`, `gtm-revenue`, `linkedin-personal-brand`, `ai-tools-general`, `copywriting-messaging`, `product-marketing`, `other`
> - `freshness`: `evergreen`, `current`, or `outdated` (from a 2026 B2B perspective)
> - `actionability`: `high`, `medium`, or `low`
> - `key_insight`: 1 sentence max — the most concrete, actionable takeaway
>
> Output a JSON array. Write to: `analyzed/batch_XX_YY.json`

Launch all agents in parallel. Wait for all to complete.

**4c. Aggregate results**

Use Node.js to merge all `analyzed/batch_*.json` files into a single dataset:
```javascript
const fs = require('fs'), path = require('path');
const files = fs.readdirSync('./analyzed').filter(f => f.endsWith('.json'));
let all = [];
for (const f of files) all = all.concat(JSON.parse(fs.readFileSync(path.join('./analyzed', f))));
fs.writeFileSync('./all_analyzed.json', JSON.stringify(all, null, 2));
console.log('Total:', all.length);
```

**4d. Compute stats**

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

### 5. Generate Report

Produce a Markdown report saved to `rapport-linkedin-saves-{YYYY-MM-DD}.md` with this structure:

```markdown
# Analyse LinkedIn Saves — {YYYY-MM-DD}

## Vue d'ensemble

| Métrique | Valeur |
|----------|--------|
| Posts analysés | N |
| Auteurs uniques | X |
| Posts à forte valeur (high actionability + non-outdated) | N |
| Posts obsolètes à archiver | N |
| Catégorie dominante | [catégorie] (N posts) |

**Distribution des catégories :**

| Catégorie | Posts | High-value |
|-----------|-------|-----------|
| [pour chaque catégorie, triée par volume décroissant] | N | N |

**Freshness :** Evergreen: N% / Current: N% / Outdated: N%

---

## Catégorie par catégorie

Pour chaque catégorie ayant ≥ 5 posts, une section avec :

### [Nom de la catégorie] (N posts, N high-value)

> **Signal** : [1 phrase sur ce que cette catégorie révèle des intérêts/habitudes de l'utilisateur]

**Top insights actionnables :**

- **[Auteur]** : [key_insight]
- **[Auteur]** : [key_insight]
- ... (top 6-8 posts high-value de cette catégorie)

---

## Posts outdated — À archiver (N posts)

| Thème | Posts | Pourquoi |
|-------|-------|----------|
| [regrouper par raison d'obsolescence] | N | [explication] |

---

## Top auteurs

**Par volume de saves :**

| Auteur | Posts sauvés |
|--------|-------------|
| [top 10] | N |

**Par posts high-value :**

| Auteur | Posts high-value | Territoire |
|--------|-----------------|-----------|
| [top 10] | N | [catégorie dominante] |

---

## Matrice d'implémentation

### Implémenter maintenant
[3-5 insights high-value + current, avec action concrète]

### Implémenter dans 30 jours
[3-5 insights plus structurants, nécessitant plus de temps]

### À utiliser comme preuves / arguments
[2-3 stats ou faits marquants extraits des posts, utilisables en contenu ou vente]

---

## Signal de positionnement

[3-5 phrases : ce que l'ensemble des saves révèle sur le profil de veille, les convictions, et le territoire de l'utilisateur. Ce n'est pas un résumé — c'est une lecture du profil intellectuel derrière les saves.]
```

### 6. Save and Deliver Report

1. Write the report to `rapport-linkedin-saves-{YYYY-MM-DD}.md` in the working directory using the Write tool
2. **Done**: "Rapport généré : rapport-linkedin-saves-{DATE}.md — N posts analysés, N high-value, N à archiver."

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
- **Volume** — scales to 1000+ posts via parallel agent chunking (batches de 50, agents lancés en parallèle)
