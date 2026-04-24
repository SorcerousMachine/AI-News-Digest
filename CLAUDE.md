# AI News Digest Pipeline

You are running a daily AI industry digest pipeline. Follow these steps
exactly, in order. Do not skip steps. Do not fabricate content.

## Step 1: Load Configuration

Read `config/feeds.yaml` for the list of RSS/Atom feeds, ArXiv feed URLs,
keyword lists, and any other configuration.

Read `state/seen.json` for previously ingested URL hashes and per-feed
health tracking. The file has this shape:
```json
{
  "seen": {
    "<sha256-of-normalized-url>": "<YYYY-MM-DD date first seen>"
  },
  "feed_health": {
    "<feed-url>": {
      "consecutive_hard_failures": 0,
      "last_success": "YYYY-MM-DD",
      "last_error_type": null,
      "last_error_date": null
    }
  }
}
```

If `state/seen.json` does not exist or is empty, treat it as
`{"seen": {}, "feed_health": {}}`. The script manages feed_health
for you — you only write it back in Step 10.

## Step 2: Fetch and Parse All Feeds

Run the feed fetcher script:

```bash
python3 scripts/fetch_feeds.py --config config/feeds.yaml --state state/seen.json
```

This script handles all feed fetching, XML parsing, URL normalization,
SHA-256 hashing, ArXiv keyword filtering, deduplication against
seen.json, and 48-hour recency filtering. It outputs **only new,
recent items** as a JSON object to stdout:

```json
{
  "feeds": [
    {
      "title": "...",
      "url": "...",
      "url_hash": "<sha256>",
      "published": "...",
      "description": "...",
      "authors": "...",
      "source": "Feed Name",
      "category": "vendor",
      "thin_description": true
    }
  ],
  "arxiv": [
    {
      "title": "...",
      "url": "...",
      "url_hash": "<sha256>",
      "authors": "...",
      "abstract": "...",
      "matched_keywords": ["agent", "planning"],
      "source": "arxiv:cs.AI",
      "category": "research"
    }
  ],
  "errors": [
    {"feed": "...", "url": "...", "source_url": "...", "type": "status:403", "error": "..."}
  ],
  "disable_candidates": [
    {
      "url": "...",
      "name": "...",
      "category": "...",
      "homepage": "...",
      "section": "feeds",
      "consecutive_hard_failures": 2,
      "last_error_type": "content_mismatch",
      "last_success": "2026-04-10"
    }
  ],
  "feed_health_update": {
    "<feed-url>": {
      "consecutive_hard_failures": 0,
      "last_success": "YYYY-MM-DD",
      "last_error_type": null,
      "last_error_date": null
    }
  },
  "scrape_targets": [
    {
      "name": "LangChain Blog",
      "homepage": "https://blog.langchain.dev/",
      "category": "open_source"
    }
  ],
  "summary": {
    "feeds_attempted": 14,
    "feeds_failed": 0,
    "error_types": {"status:403": 2, "timeout": 1, "parse_error": 0},
    "total_fetched": 2159,
    "skipped_already_seen": 1800,
    "skipped_too_old": 300,
    "new_feed_items": 47,
    "new_arxiv_items": 12,
    "disable_candidates_count": 0,
    "scrape_targets_count": 1,
    "scrape_enabled": true
  }
}
```

All items in `feeds` and `arxiv` are already deduplicated and recent.
You do not need to check them against seen.json again.

The `thin_description` flag (feed items only) marks items whose RSS
description has less than 200 characters of visible text — you will
need to fetch the article body to analyze them. See Step 6.

Save the full JSON output. You will need it for subsequent steps.

If the script fails entirely, log the error and continue to Step 3
(web discovery) so the digest still has content. If it succeeds but
`errors` is non-empty, note the failed feeds in the commit message so
feed rot is visible. Group them by `type` (e.g., `status:403`,
`timeout`, `parse_error`) using the `error_types` breakdown in
`summary` — this makes persistent blockers distinguishable from
transient blips run-over-run.

**Feed health tracking.** The script tracks `consecutive_hard_failures`
per feed across runs. Hard failures are `status:404`, `status:410`,
and `content_mismatch` (HTTP 200 where the response body does not match
the expected feed format — today that means an HTML body where XML was
expected). Soft failures (5xx,
timeouts, network, parse_error) do not increment the counter. After
three consecutive hard failures a feed appears in `disable_candidates`;
Step 9 moves those feeds out of the active list in `config/feeds.yaml`
so tomorrow's run does not waste attempts on them.

## Step 3: Web Discovery

Use web search to find significant AI industry developments from the last
48 hours that would not be covered by the RSS feeds. The 48-hour window
exists to catch anything yesterday's digest might have missed. Search for:

- New model releases or major updates
- API changes or developer tools
- Regulatory actions or policy developments
- Major funding rounds (Series B+)
- Significant benchmark results or breakthroughs
- Notable open source releases
- Industry partnerships or acquisitions
- Infrastructure developments (chips, compute, cloud)

Run 3-5 targeted web searches with specific queries like:
- "AI model release today"
- "AI regulation policy news today"
- "AI startup funding round today"
- "AI open source release today"
- "AI benchmark results today"

For each significant finding, record: title, URL, a 2-3 sentence summary
of why it matters, and a category.

Do NOT include: routine product updates, opinion pieces, rumors, or
content older than 48 hours.

**Directed scraping of retired-but-reachable sites.** If the Step 2
output includes a non-empty `scrape_targets` array, iterate each one
before (or after) the open-ended web searches:

1. WebFetch the target's `homepage` URL.
2. Identify the most recent published articles on that page — typically
   a blog index, archive list, or "latest posts" section. Layouts
   vary; use judgment.
3. For each of the 2-3 most recent articles, WebFetch the article URL
   and record title, URL, publish date, and a short summary.
4. Drop any article published outside the 48-hour recency window.

Items found this way flow into the same Step 4 deduplication and
Step 6 AI-relevance filter as other web-discovered items. Attribute
them in the digest using the target's `name`. Skip this sub-step
entirely when `scrape_targets` is empty or absent (controlled by the
`DIGEST_SCRAPE_ENABLED` env var at the script layer).

**Record each scrape target's outcome** — the scrape layer is symmetric
to the feed-fetch layer for availability reporting and retirement. For
every target you attempt, classify the result into one of three buckets:

- **`success`** — homepage fetched and article links identified. (Even
  if no articles fell inside the 48h window, a reachable homepage
  counts as success.) Resets the target's `consecutive_hard_failures`
  counter to 0 and updates `last_success` to today.
- **Transient failure** (timeout, 5xx, 403/anti-bot, network error) —
  homepage was unreachable this run but might recover. Does NOT
  increment the counter. Target is added to the "transient scrape
  failures" list used by Step 6 *Sources Unavailable Today*.
- **Hard failure** (404, 410, page resolves to a clear error/shell
  that is not a publisher site, or the homepage is obviously no longer
  a publishing surface) — increments `consecutive_hard_failures` and
  records `last_error_type` + `last_error_date = today`. When a
  target's counter reaches 3, it becomes a retirement candidate
  handled by Step 9.

Keep a session-local structure mapping each target's homepage URL to
its outcome classification and (for failures) an error description.
Steps 6, 9, and 10 all consume it.

(Passive feed discovery — tracking which non-feed sources accumulate
citations across days — is handled in Step 10 via the
`state/citation_tracking.json` state file. Each run is a fresh session
and can't remember prior days on its own, so the signal has to be
written down deterministically.)

## Step 4: Deduplicate Web Discoveries

Feed and ArXiv items from Step 2 are already deduplicated by the script.

For web-discovered items from Step 3, check each URL hash against
`state/seen.json`:
```bash
python3 -c "from scripts.fetch_feeds import hash_url; print(hash_url('THE_URL'))"
```

- If the hash exists in `seen`: skip the item (already ingested)
- If the hash does not exist: keep the item as new

If no new items exist from any source after deduplication, continue
with Steps 6-11 anyway and publish a minimal digest post whose body
is a single sentence: "No significant AI developments were surfaced
in the last 48 hours." Still update seen.json and push. This case
should be rare but not silent.

## Step 5: Triage ArXiv Papers

For the ArXiv papers that passed keyword filtering AND are new (not in
seen.json), assess each paper:

- Evaluate relevance to someone building agentic AI systems with
  structured decomposition, validation, and orchestration
- Assign significance: "high", "medium", or "low"
- Write a one-sentence note on why it matters or doesn't (for all papers)
- Write a 2-3 sentence abstract summary ONLY for "high" significance papers

Drop papers assessed as not relevant.

## Step 6: Synthesize Daily Digest

Before writing, filter feed items for AI relevance. Most configured
feeds are curated AI-focused sources, but some — notably Hacker News —
surface their full front page without any AI pre-filter. Drop any feed
item whose content is not about AI, machine learning, models, or
AI-adjacent infrastructure, policy, research, or tooling. Include an
item only if a reader building AI systems would plausibly care about
it. When in doubt, drop — a thinner digest is better than a noisy one.
(ArXiv items are already keyword-filtered upstream and triaged in
Step 5; this paragraph applies to everything in the `feeds` array and
to web-discovered items from Step 3.)

For every remaining feed item with `thin_description: true`,
WebFetch the item's URL and read the article body. The RSS teaser
alone is not enough to write useful analysis. If the fetch fails,
note the failure and work from the title alone rather than
fabricating detail. ArXiv items are exempt — abstracts are sufficient.

Write a daily digest in markdown with this exact structure:

1. **Executive summary** (2-3 sentences, no header): What are the most
   important developments today?

2. **Categorized sections** using `##` headers, ordered by significance:
   - Model Releases
   - Developer Tools
   - Research & Papers
   - Security
   - Regulatory & Policy
   - Funding & Business
   - Open Source
   - Infrastructure
   - Other

   OMIT any category with zero items.

   *Security* covers AI-specific vulnerabilities, incidents, breaches,
   prompt injection findings, model-misuse reports, credential exposure
   from agentic workflows, and similar safety/security news. Items that
   would otherwise drift into *Other* because they don't fit the
   positive-framed categories above belong here when they concern an
   adversarial or failure mode. Use *Other* only for genuinely
   miscellaneous items.

3. **Within each category**, items ordered by significance. Each item:
   - `###` title as a markdown link to the source URL
   - Source attribution in italics — **prefer primary sources** when a
     story has both primary and aggregator coverage. Link the `###`
     title to the primary source's URL (a lab announcement, the paper
     itself, the vendor blog post) rather than to TechCrunch/The
     Register coverage of it. If multiple sources corroborate the same
     story, list the primary source FIRST in the italics line, with
     aggregators after — e.g., `*Anthropic / TechCrunch / The Register*`
     rather than the other way around. The primary source is the
     original publisher of the news; aggregators are outlets that
     report ON that news.
   - 2-3 sentence analysis of why this matters

4. **Threads to Watch** (`##` header): 2-3 emerging patterns or threads
   connecting multiple items from today's digest.

5. **Sources Unavailable Today** (`##` header): include this section
   if EITHER of the following has at least one entry:
   - A Step 2 script output `errors` entry whose `type` is a
     **transient** failure — i.e., NOT in {`status:404`, `status:410`,
     `content_mismatch`}. (Hard failures on that side are handled by
     the Feeds Retired section instead.)
   - A scrape target from Step 3 that Claude classified as a transient
     failure.

   Omit the section entirely if neither list has any entries.

   Precede the list with one sentence: "These sources could not be
   fetched today. Links point to their homepages so you can check
   them directly."

   Then one bullet per transient failure. For Step 2 feed errors:

   ```
   - [{feed name}]({source_url}) — *{type}*
   ```

   (Use the `source_url` field from the error object and the `type`
   field verbatim, e.g. `status:403`, `timeout`, `parse_error`.)

   For Step 3 scrape-target failures:

   ```
   - [{target name}]({target homepage}) — *scrape: {short error}*
   ```

   Prefix the error with `scrape:` so readers can tell feed-fetch
   failures apart from scrape-layer failures.

6. **Feeds Retired Today** (`##` header): include this section when
   EITHER of the following is non-empty:
   - The Step 2 output `disable_candidates` array (feed-side
     retirements driven by the fetch script).
   - Step 3 scrape targets whose `consecutive_hard_failures` reached
     the retirement threshold (scrape-side retirements driven by
     Claude's own homepage fetches).

   Omit entirely if neither is present.

   Precede the list with one sentence: "The following sources were
   retired today after repeated hard failures. They have been moved
   out of the active pipeline."

   One bullet per retirement, grouping by source in the order above.

   For feed-side retirements (from `disable_candidates`):

   ```
   - [{name}]({homepage or url}) — *{last_error_type}* (no successful fetch since {last_success or "first run"})
   ```

   For scrape-side retirements (from Step 3 tracking):

   ```
   - [{target name}]({target homepage}) — *scrape: {last_error_type}* (no successful scrape since {last_success or "first run"})
   ```

   Prefer `homepage` when present; fall back to `url` otherwise. The
   `scrape:` prefix distinguishes which retirement path the source
   took.

Style rules:
- Direct, analytical tone. No hype, no filler.
- No emoji anywhere.
- Write for a technical audience building production AI systems.
- For research papers, focus on practical implications.
- Flag skepticism-warranting claims (benchmarks without code, extraordinary
  claims without strong evidence).
- Do NOT fabricate items. Only include items you actually found.

## Step 7: Write the Digest File

Create the file `content/posts/{YYYY-MM-DD}.md` where the date is today.

The file must begin with Hugo front matter. Use a naive datetime
(no timezone offset) — Hugo applies the site's timeZone setting
from hugo.toml (UTC). Use noon UTC as the canonical publish time:

```yaml
---
title: "AI News Digest — {Month Day, Year}"
date: {YYYY-MM-DD}T12:00:00
draft: false
summary: "{first sentence of executive summary, max 200 chars}"
tags: [{list of category slugs that appear in the digest}]
---
```

"Today" throughout this document means today in UTC. The filename,
title, and date should all use the UTC calendar date.

Tag slugs should be lowercase-hyphenated versions of the category names:
model-releases, developer-tools, research-papers, security,
regulatory-policy, funding-business, open-source, infrastructure, other.

The `summary` is the first sentence of the executive summary, truncated
to 200 characters if needed.

Follow the front matter with the full digest markdown from Step 6.

## Step 8: Post-Write Deduplication

This step catches semantic duplicates — the same story surfaced today
that was already covered in a recent digest under a different URL.
URL-hash dedup (via `state/seen.json`) cannot catch this because web
search commonly finds a different article about the same event.

Read the two most recent prior digest posts in `content/posts/`
(by filename date, excluding today's file). This matches the 48-hour
recency window enforced in Steps 2 and 3 — any item duplicate must
have appeared in one of the two prior days. If fewer than two prior
posts exist, use whatever exists.

For each item in today's digest (any section, any category), ask: is
this covering the same underlying story — same model release, same
funding round, same policy action, same research paper, same
acquisition — as anything in the prior posts? A different article
about the same story IS a duplicate. A follow-up with genuinely new
information (e.g., benchmark numbers released a day after a model
announcement, or a court ruling following a previously-reported
filing) is NOT a duplicate — keep it and lean the analysis into
what is actually new.

For each confirmed duplicate:
- Remove the item (its `###` heading, source attribution, analysis)
- If the Executive Summary or Threads to Watch references the removed
  item, rewrite those sections to stay coherent without it
- If a category section becomes empty after removal, drop the category
  heading entirely and remove its slug from the `tags:` front matter

After edits, verify the `summary` front matter field still matches
the first sentence of the (possibly rewritten) Executive Summary.
Update it if it drifted.

If every item turns out to be a duplicate, replace the body with:
"No significant new AI developments were surfaced in the last 48
hours. See the recent archive for ongoing coverage." Still proceed
to commit and push so the schedule stays consistent.

Overwrite `content/posts/{YYYY-MM-DD}.md` in place with the cleaned
version. This is the ONE exception to the "never modify existing
posts" constraint at the bottom of this document — today's file has
not been committed yet, and the constraint applies to previously-
published posts.

Do NOT add, reword, or reorder items during this step. The only
permitted edits are removals plus the coherence-preserving rewrites
of the executive summary, threads, summary field, and tags.

## Step 9: Process Retirements

This step has TWO input sources, both of which may produce retirements:

- **Feed-side candidates** — the Step 2 output `disable_candidates`
  array (RSS feeds whose fetches crossed the hard-failure threshold).
- **Scrape-side candidates** — any scrape target whose
  `consecutive_hard_failures` reached the retirement threshold during
  Step 3 this run (tracked in Claude's session-local scrape outcome
  map from that step).

If BOTH are empty, skip this step entirely.

### Feed-side routing

For each `disable_candidates` entry, route based on whether the
candidate has a `homepage` value AND whether that homepage is
currently reachable. A feed-URL failure says nothing about the
publisher's site being alive, so homepage presence + liveness is the
right signal.

- If `homepage` is null or empty → `disabled:` section.
- Otherwise, probe the homepage once via WebFetch. This probe avoids
  routing to `scrape:` when the homepage is obviously dead.
  - Homepage returns a valid page → `scrape:` section.
  - Homepage fails (404/410, clearly-error response, network error
    that persists) → `disabled:` section.
- Record this probe outcome in the Step 10 feed_health write so the
  scrape-side tracker has a baseline from day one.

### Scrape-side routing

Scrape-target retirements have no ambiguity — the homepage has
already demonstrated persistent failure across three runs, so they
always go to `disabled:`. Remove the entry from the `scrape:` section
and add it to `disabled:`.

### Writing to feeds.yaml

If the target section (`scrape:` or `disabled:`) does not yet exist,
create it near the bottom of the file (after `arxiv:`).

For each retirement, the entry should have this shape:

```yaml
# under scrape: OR disabled:, depending on routing above
  - name: "{candidate.name}"
    url: "{candidate.url}"
    homepage: "{candidate.homepage}"   # omit the line if homepage is null
    category: "{candidate.category}"
    disabled_on: "{YYYY-MM-DD today}"
    disabled_reason: "{candidate.last_error_type} ({candidate.consecutive_hard_failures} consecutive hard failures)"
```

For scrape-side retirements, `url` is the original feed URL the
target was retired from (carried through from the existing `scrape:`
entry), and `disabled_reason` should be prefixed with `scrape: ` to
indicate the retirement path — e.g.,
`scrape: homepage 404 (3 consecutive hard failures)`.

Then REMOVE the candidate's original entry from wherever it lived —
feed-side from `feeds:` (or `arxiv.feeds:` if `section` is
`arxiv.feeds`), scrape-side from `scrape:`. Preserve all other
entries, comments, and formatting in `config/feeds.yaml` exactly as
they were. Use `Edit` (not a full rewrite) to minimize diff noise.

The purpose: tomorrow's run stops trying these URLs entirely.
`scrape:` entries remain reachable for Step 3's directed scraping
(when `DIGEST_SCRAPE_ENABLED` is set). `disabled:` entries are
skipped by every pipeline layer. Re-enabling or promoting between
sections is a manual action — move the entry back from
`scrape:`/`disabled:` to the active `feeds:` list by hand when the
publisher restores their feed.

## Step 10: Update State

Build an updated `state/seen.json` with two top-level keys:

**`seen`** (URL hash deduplication):
1. Start with the existing `seen` entries
2. Add URL hashes with today's date for EVERY item the pipeline
   processed, regardless of whether it made it into the digest:
   - All feed items from the Step 2 script output (`feeds` array)
   - All ArXiv items from the Step 2 script output (`arxiv` array),
     including those Claude dropped in Step 5 as not relevant
   - All web items from Step 3 that passed Step 4 dedup, including
     those Claude chose not to include in the digest
   The goal: nothing the pipeline has already evaluated should be
   re-evaluated tomorrow. The 90-day prune (below) is the safety
   valve — items eventually get re-considered.
3. Remove any entries with dates older than 90 days from today

**`feed_health`** (per-URL failure tracking, feed + scrape):
1. Start from the Step 2 output `feed_health_update` — the script has
   already applied today's feed-side outcomes.
2. Merge in Claude's scrape-side outcomes from Step 3 (and the Step 9
   homepage probe, if one ran). For each scrape target's homepage URL:
   - `success` today → set `consecutive_hard_failures` to 0,
     `last_success` to today, clear `last_error_*` fields.
   - Hard failure today → increment `consecutive_hard_failures`, set
     `last_error_type` to a `scrape:`-prefixed tag (e.g.
     `scrape:status:404`, `scrape:empty_page`, `scrape:not_a_publisher`),
     set `last_error_date` to today.
   - Transient failure today → preserve the prior entry unchanged.
3. If a scrape target retired to `disabled:` in Step 9, keep its
   `feed_health` entry intact (don't delete). The record documents why
   the retirement happened.

Use the homepage URL as the `feed_health` key for scrape entries,
matching how the script uses feed URLs as keys. Feed and scrape keys
coexist in the same object — no collision, since feed URLs and
homepage URLs are distinct.

Write the merged result under the `feed_health` key.

Format the JSON with 2-space indentation for readable git diffs.

### Citation tracking (for passive feed discovery)

Maintain a separate state file `state/citation_tracking.json` that
records how often non-feed source names appear in source attributions.
Since each pipeline run is a fresh session with no memory of prior
runs, this file is the only mechanism that can accumulate multi-day
signal.

File shape:

```json
{
  "citations": {
    "Source Name": {
      "dates": ["2026-04-17", "2026-04-22", "2026-04-24"],
      "origin": "https://example.com"
    }
  }
}
```

The `origin` field holds `scheme://host` of the most recent citation's
article URL and is what the /sources page uses to link web-discovered
publishers.

Procedure:

1. Load the file if it exists, otherwise start from `{"citations": {}}`.
   If an existing entry still has the legacy shape (bare date list),
   migrate it to `{"dates": [...], "origin": ""}` on first encounter.
2. For each `###` item in today's digest:
   - Parse the heading's linked URL (the `(URL)` portion of
     `### [Title](URL)`) and derive the origin as `scheme://host`.
   - Parse the source attribution (the `*...*` italic line directly
     beneath the heading). Split compound attributions on ` / ` so
     each source is counted independently. Strip parenthetical
     metadata like `(577 points)` before counting.
3. For each resulting source name:
   - Ensure `citations[name]` exists with the object shape above.
   - Append today's date to `dates` (dedup — one entry per date per
     source, even if the same name appears on multiple items today).
   - Set `origin` to today's parsed origin. If today's URL was
     unparseable or missing, preserve the existing `origin`.
4. Prune any dates older than 30 days from today, per source.
5. Remove sources whose `dates` became empty after pruning.
6. Write the file back.

**Identifying candidate feeds.** After the write, compute the
candidate list: every source whose `dates` list has 3 or more entries
AND whose name does not match any active `name:` in the `feeds:`
section of `config/feeds.yaml` (case-insensitive, allowing minor
whitespace or punctuation variation). This list will be referenced in
the Step 11 commit message. If the list is empty, no commit-message
annotation is needed.

## Step 11: Commit and Push

IMPORTANT: You MUST commit and push directly to the `main` branch.
Do NOT create a new branch. Do NOT push to a `claude/` prefixed branch.
Cloudflare Pages deploys from `main` — any other branch will not deploy.

If during this run the pipeline patched its own code (script fix,
CLAUDE.md correction, template tweak — anything outside `content/posts/`,
`state/`, or `config/feeds.yaml`), commit those changes FIRST as their
own separate commit before the digest commit. Use a descriptive
subject line for the code change and include the `Co-Authored-By`
trailer. Example:

```bash
git add scripts/ layouts/ CLAUDE.md   # or whatever files changed
git commit -m "$(cat <<'EOF'
{short descriptive subject}

{optional body explaining the fix}

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

Then stage and commit the digest content:

```bash
git add content/posts/ state/seen.json state/citation_tracking.json config/feeds.yaml
```

Check if anything is staged for the digest:
```bash
git diff --cached --quiet
```

If nothing is staged, exit cleanly (this happens if all items were
duplicates and state didn't change).

If changes exist, commit directly to main. The digest commit message
MUST include the `Co-Authored-By` trailer. Use a heredoc so the
trailer lands as a proper message trailer (blank line above it, no
leading whitespace):

```bash
git commit -m "$(cat <<'EOF'
digest: {YYYY-MM-DD}

{optional body: feed-error summary grouped by error_types, feed
retirements, or other run-specific notes. If the citation_tracking
analysis from Step 10 identified candidate feeds, include them here
as a line like: "Candidate feeds: Name 1, Name 2" — the user will
review and decide whether to add them to feeds.yaml.}

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

Push both commits together:

```bash
git push origin HEAD:main
```

If the push fails because the remote has advanced (non-fast-forward),
rebase onto the latest main and try once more:
```bash
git pull --rebase origin main
git push origin HEAD:main
```

If the retry also fails, fall back to pushing the commit to a
recovery branch so the digest isn't lost:
```bash
git push origin HEAD:recovery/digest-{YYYY-MM-DD}
```

Log that the main push failed and the recovery branch name.
Do NOT send the ntfy notification (Step 12) in this case —
the user will be alerted by noticing the branch and will
merge it manually.

## Step 12: Send Notification

After a successful commit and push, send a summary notification to ntfy.sh:

```bash
curl -s \
  -H "Title: AI News Digest — {Month Day, Year}" \
  -H "Tags: newspaper" \
  -d "{executive summary from the digest}" \
  https://ntfy.sh/ai-news-digest
```

The message body should be the 2-3 sentence executive summary from
Step 6. Plain text only, no markdown. Do NOT include a Click header
with any URL — the ntfy topic is public and subscribers arrive
through different channels.

If the push failed in Step 11, do NOT send the notification.
If the notification fails, log the error but do not retry — this
is informational, not critical.

## Important Constraints

- Never combine multiple days into one digest. One run = one date.
- Never modify or overwrite previously-committed digest posts in
  content/posts/. Today's in-progress file may be rewritten during
  Step 8 dedup; all prior posts are immutable.
- Never delete files from the repository.
- If a step fails, log the error and continue to subsequent steps where
  possible. The digest should include whatever was successfully collected.
- If digest synthesis fails entirely, do NOT push a broken or empty post.
  Update state/seen.json with what was collected and push only the state.
