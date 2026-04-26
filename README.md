# Automating a Weekly SOTA ArXiv Blog Digest with Claude Cowork

> A complete tutorial on how to set up a fully automated research digest pipeline that searches ArXiv every Sunday, writes a bilingual HTML article, and updates your blog, all without writing a single line of Python or using any API key.


**Stack:** Claude Cowork (desktop app) · Static HTML blog · GitHub Pages

**The article:** [A weekly AI-generated SOTA digest](https://papasegawade.com/blog/automate-sota-arxiv-cowork)

---

![Weekly SOTA ArXiv Blog Digest](./ndapli/pipeline-sota-arxiv.png)

## Table of Contents

1. [What This Automation Does](#1-what-this-automation-does)
2. [How It Works — Architecture Overview](#2-how-it-works--architecture-overview)
3. [Prerequisites](#3-prerequisites)
4. [Step 1 — Write a Blog Context File (CLAUDE_BLOG_CONTEXT.md)](#4-step-1--write-a-blog-context-file)
5. [Step 2 — Create the Scheduled Task in Cowork](#5-step-2--create-the-scheduled-task-in-cowork)
6. [Step 3 — The Task Prompt in Detail](#6-step-3--the-task-prompt-in-detail)
7. [Step 4 — Pre-approve Tool Permissions](#7-step-4--pre-approve-tool-permissions)
8. [Step 5 — Monday Morning Workflow (5 minutes)](#8-step-5--monday-morning-workflow)
9. [How to Know if an Article Was Written](#9-how-to-know-if-an-article-was-written)
10. [Complementary Weekly Tasks](#10-complementary-weekly-tasks)
11. [Blog HTML Conventions the Agent Must Follow](#11-blog-html-conventions-the-agent-must-follow)
11b. [Conference Radar — Seasonal Coverage](#11b-conference-radar--seasonal-coverage)
12. [Anti-Duplicate System — Tracking Published Papers](#12-anti-duplicate-system--tracking-published-papers)
13. [Factual Verification — STEP 5.5](#13-factual-verification--step-55)
14. [Customizing for Your Own Blog](#14-customizing-for-your-own-blog)
15. [Full Pipeline Summary](#15-full-pipeline-summary)

---

## 1. What This Automation Does

Every **Sunday at 8 PM**, an autonomous Claude agent:

1. Runs **10 permanent ArXiv + HuggingFace searches** (always active, every week)
2. Runs **seasonal conference radar** — adds 2 targeted queries per active conference (ACL, Interspeech, NeurIPS, ICLR) based on the current month, so no high-quality accepted paper slips through
3. Selects the **5–8 most relevant papers** of the week, tagging accepted conference papers with a `badge--conf` badge
4. Identifies the **notable model release of the week** via HuggingFace trending
5. Generates a **complete bilingual HTML article** (French + English) with: per-paper mathematical method blocks, Python snippets, HuggingFace links, Wolof applicability scores, conference badges, and a "Model of the week" section
6. Inserts a **new card** at the top of your `blog.html` index
7. Adds a new **sitemap.xml entry** for SEO

On **Monday morning**, you open the generated article, review it in your browser, and run `git push` if you're satisfied. The entire research curation and writing takes zero time on your end.

**No API key. No Python script. No server. Just Cowork running on your Mac.**

---

## 2. How It Works — Architecture Overview

```
Sunday 8:00 PM
    │
    ▼
Cowork Scheduled Task: sota-arxiv-weekly-digest
    │
    ├── STEP 0 ── Read CLAUDE_BLOG_CONTEXT.md
    │              (site conventions, CSS classes, bilingual rules, em dash rule)
    │
    ├── STEP 1 ── 10 × permanent searches (7 ArXiv + 3 HuggingFace)
    │              (NLP, Speech, low-resource, African languages, LLMs, LoRA...)
    │              ArXiv = priority #1, always executed every week
    │              + WebSearch per paper to get real authors and institutions
    │
    ├── STEP 1.3 ─ Conference Radar (seasonal, ADDITIVE to STEP 1)
    │              Check current month → add 2 queries per active conference:
    │              · ICLR        active: January → May
    │              · ACL / NAACL active: April → August
    │              · Interspeech active: March → September
    │              · EMNLP       active: July → November
    │              · NeurIPS     active: May → December
    │              If accepted conference paper found → add badge--conf in paper-block
    │
    ├── STEP 1.5 ─ WebSearch HuggingFace trending
    │              (identify the notable model release of the week)
    │
    ├── STEP 2 ── Write blog/sota/sota-YYYY-MM-DD.html
    │              (full bilingual article: model-of-the-week + enriched paper-blocks)
    │              (relative paths: ../../css/style.css · ../../js/main.js)
    │              (highlight.js in <head> + script init in <body>)
    │              (badge--conf on confirmed conference papers)
    │
    ├── STEP 3 ── Update blog.html
    │              (insert new featured card at top of grid)
    │              (card href: "blog/sota/sota-YYYY-MM-DD.html")
    │
    ├── STEP 4 ── Update sitemap.xml
    │              (add new URL with lastmod + priority 0.9)
    │
    ├── STEP 5 ── Verify bilingual parity (FR count == EN count)
    │              and check all arxiv links present
    │              and verify ../../css/style.css path is correct
    │              and 0 em dashes in prose sentences
    │              and 0 duplicate arXiv IDs vs published digests
    │
    ├── STEP 5.5 ─ Factual verification (NEW — post-writing)
    │              For each paper-block: fetch arxiv.org/abs/XXXXXXX
    │              and cross-check: title, authors, key metrics, institutions, date
    │              Correct HTML on mismatch + add <!-- CORR: was X, actual Y -->
    │              If fetch fails: add <!-- FETCH-FAIL --> + hedge summary wording
    │
    └── STEP 6 ── Update CLAUDE_BLOG_CONTEXT.md section 16
                   (add new digest row + arXiv IDs to the published-papers registry)

Monday morning
    │
    └── PSW: open file → review → git push ✅
```

---

## 3. Prerequisites

| Requirement | Details |
|-------------|---------|
| **Claude Cowork** | Desktop app — [claude.ai](https://claude.ai) |
| **Static HTML blog** | Any blog with a predictable file structure |
| **Folder mounted in Cowork** | Your local blog repo must be selected as the workspace folder |
| **No API key needed** | Cowork uses WebSearch natively |
| **No Python needed** | All logic is in the prompt itself |

> **Important:** Cowork must have your blog folder mounted as its workspace. This gives the agent read/write access to your repo files.

---

## 4. Step 1 — Write a Blog Context File

The single most important ingredient is a **`CLAUDE_BLOG_CONTEXT.md`** file at the root of your repo. This file is the agent's memory, it reads it at the start of every run to understand your site's conventions without needing to re-explore the codebase each time.

### What to put in it

```markdown
# My Blog: Context for Claude Cowork

## Site Architecture
(file tree, what each file does)

## CSS Components
(all custom classes with usage examples: .callout, .badge, .paper-block, etc.)

## Bilingual System
(how data-fr/data-en attributes work, how JS switches language)

## Blog Card Structure
(exact HTML markup for inserting a new card in blog.html)

## Article Template
(what the <head>, nav, footer look like, so the agent copies the right structure)

## Existing Articles
(table of all published articles, avoids duplicates)

## Automation Tasks
(table of scheduled tasks, their outputs, what PSW does after each run)
```

### Why this file matters

Without it, the agent would have to read every HTML file in your repo to understand conventions, wasting tokens and risking inconsistencies. With it, the agent has a single source of truth and produces output that matches your existing style exactly.

**Rule:** Every time you make a structural change to your blog (new CSS class, new nav item, renamed file), update `CLAUDE_BLOG_CONTEXT.md`. The agent reads this file on every run.

---

## 5. Step 2 — Create the Scheduled Task in Cowork

In the Cowork desktop app:

1. Open the **Scheduled** section in the sidebar
2. Click **"New task"**
3. Set:
   - **Task ID:** `sota-arxiv-weekly-digest`
   - **Schedule:** `0 20 * * 0`: cron for every Sunday at 8 PM local time
   - **Notify on completion:** ✅ enabled
4. Paste the full prompt (see Step 3 below)
5. Save

The task is now scheduled. It will appear with a `nextRunAt` timestamp showing the next Sunday at ~8 PM.

### Cron expression reference

```
0 20 * * 0
│  │  │ │ └── Day of week (0 = Sunday)
│  │  │ └──── Month (any)
│  │  └────── Day of month (any)
│  └───────── Hour (20 = 8 PM)
└──────────── Minute (0)
```

> **Note:** Cowork applies a small jitter of a few minutes to balance load. Your task will fire at ~8:06 PM, not exactly 8:00 PM. This is normal.

---

## 6. Step 3 — The Task Prompt in Detail

The prompt is the heart of the automation. It must be completely self-contained because the agent runs in a fresh session with no memory of previous conversations.

### Structure of the prompt

```
## STEP 0 — Read context file
   → tells the agent WHERE your conventions are documented
   → includes the em dash rule: never "—" inside sentences

## STEP 1 — 10 permanent searches (7 ArXiv + 3 HuggingFace)
   → targeted WebSearch queries for your research domains
   → selection criteria (relevance badges, paper count)
   → one extra WebSearch per paper for real author names
   → ArXiv = priority #1, always executed every week

## STEP 1.3 — Conference Radar (seasonal, ADDITIVE)
   → check current month via `date +%m`
   → add 2 targeted queries per active conference window:
      ICLR (jan–may) · ACL/NAACL (apr–aug) · Interspeech (mar–sep)
      EMNLP (jul–nov) · NeurIPS (may–dec)
   → if accepted paper from these venues: tag with badge--conf in the paper-block
   → ArXiv remains the primary source; this step only ADDS queries, never replaces

## STEP 1.5 — HuggingFace trending
   → identify the notable open-source model release of the week
   → collect: name, license, sizes, context length, HF model IDs

## STEP 2 — Generate HTML article
   → write to blog/sota/sota-YYYY-MM-DD.html (dedicated subfolder)
   → relative paths use ../../ (two levels up): ../../css/style.css · ../../js/main.js
   → highlight.js loaded in <head>, initialized in <body>
   → model-of-the-week section (if notable model found — check section 16.3 for already-covered models)
   → enriched paper-block per paper: method + snippet + HF link + Wolof score
   → badge--conf on confirmed conference papers
   → bilingual attributes on every visible text element

## STEP 3 — Update blog.html
   → exact insertion point (after which HTML element)
   → exact card markup to use (copy your real card structure)

## STEP 4 — Update sitemap.xml
   → exact XML entry to insert
   → where to insert it

## STEP 5 — Verification bash commands
   → bilingual parity check (FR == EN count)
   → ../../css/style.css path check
   → arxiv link count check
   → 0 em dashes in prose sentences
   → no spurious badge--conf (only on confirmed accepted papers)
   → verify 0 arXiv IDs overlap with section 16.2 of CLAUDE_BLOG_CONTEXT.md

## STEP 5.5 — Factual verification (NEW)
   → for each paper: fetch https://arxiv.org/abs/XXXXXXX
   → compare title, authors, institutions, key metrics against what was written
   → correct any mismatch in the HTML before finalizing
   → if fetch fails: note it with an HTML comment and soften the wording

## STEP 6 — Update CLAUDE_BLOG_CONTEXT.md section 16 (NEW)
   → add new row to the digest index table (16.1)
   → add new block of arXiv IDs to the exclusion list (16.2)
   → add model of the week to the covered-models table (16.3)
```

### The bilingual parity rule

If your blog supports language switching, every text element in the generated article must have both `data-fr` and `data-en` attributes:

```html
<!-- Correct -->
<p data-fr="Texte en français"
   data-en="Text in English">Texte en français</p>

<!-- Wrong — agent will self-correct in Step 5 -->
<p>Texte en français</p>
```

The verification step runs:
```bash
fr=$(grep -c 'data-fr=' "$FILE")
en=$(grep -c 'data-en=' "$FILE")
[ $fr -eq $en ] && echo "OK" || echo "MISMATCH — fix before next step"
```

### Paper block HTML structure (enriched)

Each selected paper gets an enriched `.paper-block` component. The key additions compared to the minimal version: real authors from WebSearch, a mathematical method block, an optional Python snippet, optional HuggingFace links, and a Wolof applicability score.

```html
<div class="paper-block">
    <div class="paper-block__header">
        <!-- badge--green=direct impact, badge--amber=transferable, badge--gold=general -->
        <span class="badge badge--green" data-fr="🟢 DIRECT" data-en="🟢 DIRECT">🟢 DIRECT</span>
        <!-- Domain: badge--blue for SPEECH | NLP | LLM | ML -->
        <span class="badge badge--blue">SPEECH</span>
    </div>
    <h3 class="paper-block__title">
        <a href="https://arxiv.org/abs/2401.12345" target="_blank" rel="noopener">
            Full Paper Title
        </a>
    </h3>
    <!-- Authors: always use real names from WebSearch, NEVER "Anonyme et al." -->
    <p class="paper-block__authors">
        Firstname Lastname et al. (Institution) · 2025 ·
        <a href="https://arxiv.org/abs/2401.12345" target="_blank" rel="noopener">arXiv:2401.12345</a>
    </p>

    <!-- Summary: no em dash "—" inside sentences. Use ":", ";", "de" instead. -->
    <p class="paper-block__summary"
       data-fr="Résumé sans tiret em dans les phrases : utiliser ':', ';' ou 'de'."
       data-en="Summary without em dashes in prose: use ':', ';', or 'of'.">Résumé...</p>

    <!-- Mathematical method block — always include, 3 lines of intuition -->
    <div class="paper-block__method">
        <span class="method-label" data-fr="Méthode clé" data-en="Key method">Méthode clé</span>
        <p data-fr="(1) nom de la méthode, (2) formule centrale, (3) apport concret."
           data-en="(1) method name, (2) core formula, (3) concrete contribution.">...</p>
    </div>

    <!-- Python snippet — only for papers with direct code applicability (10-15 lines max) -->
    <pre><code class="language-python"># Short runnable snippet, English comments only
</code></pre>

    <!-- HuggingFace resources — only if a public HF model/dataset exists -->
    <div class="paper-block__resources">
        <a href="https://huggingface.co/ORG/MODEL" target="_blank" rel="noopener"
           class="hf-link">🤗 model-name</a>
    </div>

    <!-- Wolof applicability score — always include -->
    <!-- ★★★★★ blueprint direct | ★★★★☆ strongly transferable | ★★★☆☆ moderate | ★★☆☆☆ indirect | ★☆☆☆☆ minimal -->
    <div class="paper-block__wolof-score">
        <span class="score-label" data-fr="Pertinence Wolof" data-en="Wolof relevance">Pertinence Wolof</span>
        <span class="score-stars">★★★☆☆</span>
        <span data-fr="(justification 1 phrase)" data-en="(1-sentence justification)">(justification)</span>
    </div>

    <div class="paper-block__footer">
        <a href="https://arxiv.org/abs/2401.12345" target="_blank" rel="noopener"
           class="paper-block__link">arxiv.org/abs/2401.12345 →</a>
    </div>
</div>
```

### Model of the week block

If a notable open-source model was released this week, add this section before the paper blocks:

```html
<div class="model-of-the-week">
    <div class="model-of-the-week__header">
        <span class="model-badge" data-fr="Modèle de la semaine" data-en="Model of the week">Modèle de la semaine</span>
        <h2 class="model-of-the-week__title"
            data-fr="MODEL_NAME de ORGANIZATION"
            data-en="MODEL_NAME by ORGANIZATION">MODEL_NAME de ORGANIZATION</h2>
    </div>
    <!-- No em dash in description sentences -->
    <p data-fr="Description : licence, capacités, pertinence pour les langues africaines."
       data-en="Description: license, capabilities, relevance for African languages.">...</p>

    <div class="model-of-the-week__specs">
        <div class="model-spec">
            <div class="model-spec__key" data-fr="Licence" data-en="License">Licence</div>
            <div class="model-spec__value">Apache 2.0</div>
        </div>
        <!-- repeat for Tailles/Sizes, Contexte/Context, Langues/Languages -->
    </div>

    <div class="paper-block__resources">
        <a href="https://huggingface.co/ORG/MODEL_ID" target="_blank" rel="noopener"
           class="hf-link">🤗 model-id</a>
    </div>

    <div class="paper-block__wolof-score">
        <span class="score-label" data-fr="Pertinence Wolof" data-en="Wolof relevance">Pertinence Wolof</span>
        <span class="score-stars">★★★★☆</span>
        <span data-fr="(justification)" data-en="(justification)">(justification)</span>
    </div>
</div>
```

### Critical: give the agent your EXACT blog card markup

The most common failure point is the agent inserting a card with the wrong HTML structure. Fix this by including your exact card markup in the prompt, copied from your real `blog.html`:

```html
<!-- This is what YOUR blog uses — copy it exactly into your prompt -->
<article class="blog-card blog-card--featured" data-category="llm">
    <div class="blog-card__meta">
        <span class="blog-card__category" data-fr="SOTA · ArXiv" data-en="SOTA · ArXiv">SOTA · ArXiv</span>
        <span class="blog-card__date" data-fr="[DATE_FR]" data-en="[DATE_EN]">[DATE_FR]</span>
    </div>
    <h2 class="blog-card__title" data-fr="..." data-en="...">...</h2>
    <p class="blog-card__excerpt" data-fr="..." data-en="...">...</p>
    <div class="blog-card__tags"><span>Tag</span></div>
    <a href="blog/sota/sota-YYYY-MM-DD.html" class="blog-card__link"
       data-fr="Lire le digest →" data-en="Read digest →">Lire le digest →</a>
</article>
```

> **Note on paths:** the article lives in `blog/sota/sota-YYYY-MM-DD.html` but the card's `href` in `blog.html` is `blog/sota/sota-YYYY-MM-DD.html` (relative to the site root). Inside the article itself, all asset paths use `../../` (e.g., `../../css/style.css`).

> **Lesson learned:** Don't describe the card structure in words. Paste the actual HTML. The agent copies it exactly and fills in the variables.

---

## 7. Step 4 — Pre-approve Tool Permissions

**This step is mandatory before the first automatic run.**

1. In Cowork → Scheduled, find `sota-arxiv-weekly-digest`
2. Click **"Run now"**
3. Cowork will ask for permission to use **WebSearch** and **write files** to your workspace
4. Grant both permissions

These approvals are stored on the task and auto-applied to all future automatic runs. Without this step, the first Sunday run will pause mid-execution waiting for permission and produce nothing.

---

## 8. Step 5 — Monday Morning Workflow

After the Sunday run, you receive a Cowork notification. Monday morning:

```bash
cd ~/Desktop/your-blog-repo

# 1. See what was generated
git status
git diff --stat

# 2. Open the article in your browser
open blog/sota/sota-2026-04-06.html

# 3. Review: check paper summaries, links, layout
#    - Are the arxiv links valid?
#    - Are the paper summaries accurate?
#    - Does the card look right on blog.html?

# 4. If satisfied — push
git add blog/sota/sota-2026-04-06.html blog.html sitemap.xml
git commit -m "feat: SOTA ArXiv digest week 15 2026-04-06"
git push
```

**Total time: ~5 minutes.** The agent does the research and writing. You do the editorial judgment.

---

## 9. How to Know if an Article Was Written

### Method 1 — Cowork notification
You receive an in-app notification when the task completes. This is the easiest signal.

### Method 2 — Check the file directly
```bash
ls ~/Desktop/your-blog-repo/blog/sota/sota-*.html
# Output: blog/sota/sota-2026-04-06.html  ← article was written
# Output: (empty)                         ← task failed or hasn't run yet
```

### Method 3 — git status
```bash
cd ~/Desktop/your-blog-repo && git status
# You should see 3 modified files:
#   modified:   blog.html
#   modified:   sitemap.xml
#   new file:   blog/sota/sota-2026-04-06.html
```

### Method 4 — Check Cowork run history
In Cowork → Scheduled → `sota-arxiv-weekly-digest`, the `lastRunAt` timestamp shows the most recent execution time.

### What to do if the task ran but produced nothing

This usually means tool permissions were not pre-approved. Fix:
1. Click "Run now" again
2. Grant WebSearch + file write permissions when prompted
3. The task will complete successfully this time
4. Future automatic runs will work without prompts

---

## 10. Complementary Weekly Tasks

Two additional tasks pair naturally with the SOTA digest:

### `huggingface-weekly-report`: Saturday 7 PM

Searches HuggingFace Hub for new models and datasets in your research domains. Output: `docs/hf-weekly-YYYY-MM-DD.md`, a private Markdown report (gitignored). Read it before reviewing the SOTA digest on Monday morning.

**Cron:** `0 19 * * 6`

### `arxiv-daily-digest`: Manual trigger

On-demand ArXiv search for the last 48 hours. Useful when you want a quick pulse check mid-week, or when a major paper drops and you want a same-day summary.

**Schedule:** Manual only (no cron, trigger from Cowork when needed)

### Keeping reports private

Both `docs/` reports are gitignored, they never appear in your public repo:

```gitignore
# Internal reports, local use only, never commit
docs/hf-weekly-*.md
docs/arxiv-daily-*.md
```

---

## 11. Blog HTML Conventions the Agent Must Follow

These are the conventions from `papasegawade.com`, adapt them for your own blog.

### CSS file
All utility classes are in `css/style.css`, no inline `<style>` blocks in generated articles.

Articles in `blog/sota/` are two levels deep, so all paths use `../../`:
```html
<link rel="stylesheet" href="../../css/style.css">
<link rel="icon" type="image/svg+xml" href="../../img/favicon.svg">
<script src="../../js/main.js"></script>
```

Articles in `blog/` (one level deep) use `../`:
```html
<link rel="stylesheet" href="../css/style.css">
```

### Available CSS components (already in style.css)

| Class | Purpose |
|-------|---------|
| `.paper-block` | ArXiv paper card container |
| `.paper-block__method` | Mathematical method block (gold left border, secondary background) |
| `.method-label` | Label "Méthode clé" in uppercase gold |
| `.paper-block__resources` | Row of HuggingFace resource links |
| `.hf-link` | HuggingFace link pill (gold border, hover fill) |
| `.paper-block__wolof-score` | Wolof applicability score row with stars |
| `.score-stars` | Gold star characters (★★★☆☆ style) |
| `.model-of-the-week` | Model of the week card (gold border, featured background) |
| `.model-of-the-week__specs` | Grid of model specs (license, sizes, context, languages) |
| `.model-badge` | "Modèle de la semaine" pill badge |
| `.callout` | Highlighted note block (gold border) |
| `.callout--warn` | Warning variant (amber) |
| `.callout--success` | Success variant (green) |
| `.badge--green` | Relevance: direct impact |
| `.badge--amber` | Relevance: transferable methodology |
| `.badge--gold` | Relevance: general LLM foundation |
| `.badge--blue` | Domain tag (LLM, SPEECH, NLP, ML) |
| `.badge--conf` | Conference badge (ACL 2026, Interspeech 2026, etc.) — only for confirmed accepted papers |
| `.stat-grid` | 3-column KPI card grid |
| `.lang-hint` | Bilingual notice banner |

### Bilingual system

Every visible text element needs both attributes:

```html
<h1 data-fr="Titre en français" data-en="English title">Titre en français</h1>
<p  data-fr="Contenu FR" data-en="EN content">Contenu FR</p>
```

The default language (French) is set on `<html data-lang="fr">`. JavaScript in `main.js` handles switching. Parity must be exact: `grep -c 'data-fr='` must equal `grep -c 'data-en='`.

---

## 11b. Conference Radar — Seasonal Coverage

The weekly digest covers ArXiv continuously, but the most impactful NLP and speech papers are often accepted at major venues before their preprint appears. The Conference Radar ensures those papers are not missed by adding targeted queries during the period when accepted preprints are most likely to appear on ArXiv (camera-ready period and conference week).

**Design principle:** ArXiv = source #1, always. The conference radar is purely additive. It adds 2 queries per active conference, never replaces the 10 permanent searches.

### Active windows per conference

| Conference | Active window | Peak preprint period | Focus for low-resource NLP |
|------------|--------------|---------------------|---------------------------|
| **ICLR** | January → May | April–May (conf week) | Efficient fine-tuning, adapters, multilingual learning |
| **ACL / NAACL** | April → August | June–August (conf weeks) | African NLP tracks, low-resource NLP, cross-lingual transfer |
| **Interspeech** | March → September | August–September (conf week) | ASR/TTS low-resource, speech code-switching, African speech |
| **EMNLP** | July → November | October–November (conf week) | Multilingual NLP, low-resource, African languages |
| **NeurIPS** | May → December | November–December (conf week) | Efficient ML, multilingual LLMs, low-resource learning |

### Queries per conference (replace YEAR with current year)

**ICLR:**
```
site:arxiv.org "ICLR YEAR" "low-resource" multilingual language efficient
site:arxiv.org "ICLR YEAR" speech OR "African languages" adapter fine-tuning
```

**ACL / NAACL:**
```
site:arxiv.org "ACL YEAR" OR "NAACL YEAR" "African languages" OR "Wolof" NLP
site:arxiv.org "ACL YEAR" "low-resource" speech multilingual cross-lingual
```

**Interspeech:**
```
site:arxiv.org "Interspeech YEAR" "low-resource" speech ASR TTS African
site:arxiv.org "Interspeech YEAR" code-switching multilingual speech
```

**EMNLP:**
```
site:arxiv.org "EMNLP YEAR" "African languages" "low-resource" NLP
site:arxiv.org "EMNLP YEAR" multilingual speech code-switching
```

**NeurIPS:**
```
site:arxiv.org "NeurIPS YEAR" "low-resource" multilingual OR African language
site:arxiv.org "NeurIPS YEAR" efficient "language model" speech OR adapter
```

### Conference badge in paper-block

When a paper is identified as accepted at one of these venues, add `badge--conf` alongside the existing relevance and domain badges:

```html
<div class="paper-block__header">
    <span class="badge badge--green">🟢 DIRECT IMPACT</span>
    <span class="badge badge--blue">SPEECH</span>
    <span class="badge badge--conf">ACL 2026</span>  <!-- only if confirmed accepted -->
    <span class="paper-block__wolof">Wolof ★★★★☆</span>
</div>
```

**Rule:** never add `badge--conf` if acceptance is not confirmed. An ArXiv preprint mentioning "submitted to ACL" is not the same as "accepted at ACL". Only use the badge when the paper explicitly states acceptance.

---

## 12. Anti-Duplicate System — Tracking Published Papers

### The problem

Without memory between runs, an automated agent has no way to know which papers it already covered in previous weeks. Left unchecked, the same high-quality paper will appear in two or three consecutive digests — it stays at the top of search results because it is still recent and relevant. This is the kind of silent quality problem that erodes reader trust without anyone noticing immediately.

The issue is compounded by the seasonal conference radar: a paper flagged as "Interspeech 2026 accepted" will keep showing up in speech + low-resource searches for months.

### The solution: a published-papers registry in CLAUDE_BLOG_CONTEXT.md

The fix is to maintain a persistent, human-readable registry of every arXiv ID already published, directly in `CLAUDE_BLOG_CONTEXT.md`. Since the agent reads this file at the start of every run (STEP 0), it sees the exclusion list before it selects any paper.

The registry lives in **section 16** of `CLAUDE_BLOG_CONTEXT.md` and has three parts:

**16.1 — Digest index** (one row per digest, for human reference):

```markdown
| File | Date | Week | Summary |
|------|------|------|---------|
| blog/sota/sota-2026-04-05.html | 2026-04-05 | Week 14 | Thiomi Dataset, AfrIFact, MzansiLM… |
| blog/sota/sota-2026-04-12.html | 2026-04-12 | Week 15 | Senegalese NLP, LoASR-Bench, Budget-Xfer… |
```

**16.2 — Exclusion list** (arXiv IDs, one per line, grouped by digest):

```
# sota-2026-04-05 (Week 14)
2603.29244  The Thiomi Dataset
2604.00706  AfrIFact
...

# sota-2026-04-12 (Week 15)
2601.09716  Opportunities and Challenges of NLP for Senegalese Languages
...
```

**16.3 — Models already covered** (prevents repeating the model of the week):

```markdown
| Digest | Model | Organisation |
|--------|-------|--------------|
| 2026-04-12 | Qwen3-ASR-1.7B | Alibaba Qwen |
| 2026-04-20 | Canary-Qwen-2.5B | NVIDIA NeMo |
```

### How the agent uses the registry

The prompt instructs the agent to, in STEP 0, extract the full ID list from section 16.2 and treat it as a hard exclusion filter during paper selection (STEP 1). Any candidate paper whose ID matches an entry in the list is discarded immediately, regardless of how relevant it looks in search results.

At the end of each run (STEP 6), the agent adds the new digest's IDs to the registry, so the next week's run has an up-to-date exclusion list.

### Why this approach rather than database or external state

The registry lives in plain Markdown inside the repo for three reasons. First, the agent can read it as part of its normal file-reading workflow with no extra tooling. Second, it is human-readable and human-editable: you can manually remove an ID if you want to revisit a paper in a future digest. Third, it is version-controlled with the rest of the blog, so you have a complete audit trail of what was published and when.

### Maintenance rule

After each `git push` of a new digest, verify that the CLAUDE_BLOG_CONTEXT.md update was included in the commit. The three files that should always move together are:

```bash
git add blog/sota/sota-YYYY-MM-DD.html  # new article
git add blog.html                        # new card
git add sitemap.xml                      # new URL
git add CLAUDE_BLOG_CONTEXT.md           # updated registry
```

---

## 13. Factual Verification — STEP 5.5

### The problem

Web search results — even from Google with site:arxiv.org — frequently return imprecise summaries. The result snippet may truncate an author list, misquote a WER figure, confuse two papers with similar titles, or describe a 2024 result as if it were the 2026 one. When the agent relies on search snippets alone to write paper summaries, these errors propagate silently into the published digest.

Common failure modes observed in practice:

- A WER of 3.24% reported as 4.5% because the snippet described an earlier checkpoint
- An author listed with the wrong institution because two papers from the same group had similar titles
- A corpus size of "601,000 annotations" summarized as "600 hours of audio" (confusing text and speech data)
- A September 2025 preprint described as a 2026 publication because the arxiv ID starts with `2509`

These errors are small individually but they accumulate and undermine the digest's credibility as a research reference.

### The solution: fetch the arxiv abstract page for every selected paper

After the article is written but before it is finalized, STEP 5.5 fetches the official arxiv abstract page for each paper and compares it against what was written:

```
fetch("https://arxiv.org/abs/XXXXXXX")
```

Five verification points per paper:

| Point | What is checked | Action on mismatch |
|-------|-----------------|--------------------|
| **Exact title** | Title in `<h3>` matches the arxiv page title character for character | Correct `<h3>` and `<a>` in the paper-block |
| **Real authors** | All listed authors appear in the paper's author list | Correct `paper-block__authors` |
| **Key figures** | WER, BLEU score, parameter count, corpus size, benchmark result match the abstract | Correct summary and method block |
| **Institutions** | Author affiliations are correct | Correct the author line |
| **Year / ID coherence** | Paper is from 2025 or 2026, ID format `YYMM.XXXXX` matches claimed year | Flag if the ID suggests a different year than stated |

### What happens when a fetch fails

Some arxiv pages are temporarily unreachable from the agent's network. In that case, the agent:

1. Adds an HTML comment: `<!-- FETCH-FAIL: arxiv.org/abs/XXXXXXX — not verified -->`
2. Reformulates the summary with hedged language: "according to the available abstract…" instead of assertive claims
3. Flags the paper for manual verification in the Monday morning review

This is the correct tradeoff: better to publish a slightly hedged summary than to silently propagate an incorrect figure.

### Cost of this step

Five fetches per digest, one per selected paper. Each fetch takes 2–5 seconds. Total overhead: under 30 seconds per run, added to a process that already runs for several minutes. The tradeoff is strongly in favor of verification: one corrected WER figure prevents a researcher from citing an incorrect result.

### Correction traceability

When a correction is made, the agent adds an HTML comment directly in the paper-block:

```html
<!-- CORR: authors were "Sy et al. (INRIA)" — actual: "Sy et al. (LORIA)" -->
<!-- CORR: corpus size was "860 hours" — actual: "860 hours filtered from 1.4TB raw" -->
```

These comments are invisible to readers but visible in the source, creating an audit trail of what was corrected and why.

---

## 14. Customizing for Your Own Blog

### Adapt the ArXiv search queries

In the task prompt, replace the 7 search queries with your own research domains:

```
site:arxiv.org "your domain" "your keywords" 2026
```

Examples for different research areas:
- Computer vision: `site:arxiv.org "object detection" "transformer" efficient 2026`
- Robotics: `site:arxiv.org "reinforcement learning" "robot manipulation" 2026`
- Bioinformatics: `site:arxiv.org "protein structure" "language model" 2026`

### Adapt the relevance badges

Change the badge labels to match your relevance criteria:

```
high    → direct application to your main use case
medium  → transferable methodology
ambient → general advancement in your field
```

### Adapt the HTML template

The generated article template in the prompt must match your blog's `<head>`, nav, and footer structure exactly. Copy these from your `template-article.html` and paste them into the prompt.

### Adapt the file naming convention

The prompt uses `sota-YYYY-MM-DD.html`. Change this to match your blog's convention:
- `digest-YYYY-MM-DD.html`
- `weekly-YYYY-WW.html`
- `arxiv-wNN-YYYY.html`

### Monolingual blogs

If your blog is monolingual, remove all `data-fr`/`data-en` attributes from the template and the bilingual parity verification step.

---

## 15. Full Pipeline Summary

```
Every week, automatically:

SAT 19:00 → huggingface-weekly-report
             └─ docs/hf-weekly-YYYY-MM-DD.md (private)
                New HuggingFace models & datasets in your domains

SUN 20:00 → sota-arxiv-weekly-digest
             ├─ blog/sota/sota-YYYY-MM-DD.html  (new article)
             ├─ blog.html                        (new card at top)
             └─ sitemap.xml                      (new URL entry)

MON ~09:00 → YOU
             └─ open blog/sota/sota-*.html
                review 5 minutes
                git push ✅
```

**Time investment:** ~5 minutes/week for editorial review.
**Output:** 52 research digest articles/year, fully indexed, bilingual, SEO-ready.

---

## Key Lessons Learned

**1. The context file is everything.**
A well-maintained `CLAUDE_BLOG_CONTEXT.md` is what makes the agent produce consistent output. Treat it as living documentation, update it whenever your site structure changes.

**2. Paste exact HTML, don't describe it.**
The most reliable way to get the agent to insert correct markup is to paste the actual HTML from your existing files into the prompt. The agent copies it literally and fills in variables.

**3. Pre-approve permissions before the first run.**
This is the #1 reason first runs fail. Always click "Run now" manually once before relying on the automatic schedule.

**4. The agent needs verification steps.**
Include bash commands at the end of the prompt that the agent runs on itself. This catches bilingual parity issues, missing links, and empty paper blocks before you review.

**5. Keep internal reports out of git.**
Research reports (`docs/hf-weekly-*.md`, etc.) are useful locally but shouldn't be in your public repo. Add them to `.gitignore` before the first run.

**6. Never use em dashes inside sentences.**
In bilingual HTML articles, the rule is strict: never use "—" inside prose sentences or `data-fr`/`data-en` attributes. Use ":", ";", or "de" instead. Em dashes are only acceptable inside HTML `<title>`, `<h1>`, `<h2>` tags and meta title attributes. Enforce this rule in your task prompt so the agent follows it from the start.

**7. Fetch real authors, never leave placeholders.**
The agent's default is to write "Author et al." when it can't fetch the ArXiv page directly. Always include an explicit instruction in the prompt to run a WebSearch per paper to retrieve real author names and institutions before writing the article.

**8. Enrich, don't just summarize.**
A digest is most useful to researchers when it goes beyond the abstract. The per-paper mathematical method block (3 lines of intuition), the Wolof applicability score, and the Python snippet are what differentiate this digest from a simple RSS feed. Include all three in your paper-block template and your prompt.

**9. Add a seasonal conference radar — ArXiv stays #1.**
ArXiv covers most high-quality work continuously, but accepted conference papers often appear as preprints only during the camera-ready and conference period. A seasonal radar (ACL, Interspeech, NeurIPS, ICLR) adds 2–4 targeted queries per active window, capturing papers that pure ArXiv monitoring would miss. The radar is additive: it never replaces the permanent searches, only supplements them. Keep the conference windows updated each year as submission and conference dates shift.

**10. Maintain a persistent published-papers registry — duplicates are silent quality killers.**
After 3–4 weeks of running, the agent will start rediscovering papers it already covered. High-quality, recent papers stay at the top of search results for months. Without an exclusion list, the same paper appears in multiple digests — readers notice even if you don't. The fix is a plain-Markdown registry of arXiv IDs in `CLAUDE_BLOG_CONTEXT.md` (section 16), updated by the agent at the end of each run. The agent reads it before selecting papers, the list grows automatically, and you get a complete audit trail. The four files that must always be committed together are: the new article, `blog.html`, `sitemap.xml`, and `CLAUDE_BLOG_CONTEXT.md`.

**11. Add factual verification — search snippets lie.**
Web search result summaries frequently misquote key figures (WER, parameter counts, corpus sizes), truncate author lists, or describe an older result as a new one. These errors are small but they accumulate and undermine the digest's credibility as a research reference. STEP 5.5 fetches the official arxiv abstract for each selected paper after the article is written, compares it against the draft, and corrects any mismatch before the file is saved. The cost is under 30 seconds per run. The benefit is a digest that researchers can actually cite. When a fetch fails, the agent hedges its wording and marks the paper with `<!-- FETCH-FAIL -->` for manual review on Monday morning.

---

*Built and documented by [Papa Séga WADE](https://papasegawade.com), April 2026.*
*Research domains: NLP · Speech · Low-resource African languages · LLMs · Code-switching*
*Last updated: 2026-04-27 — added anti-duplicate registry (section 16) and factual verification (STEP 5.5)*