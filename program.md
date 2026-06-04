# autoresearch-ads

Autonomous ad copy optimization through experimentation. You pull data, score previous experiments, generate new copy, deploy it, and log everything. One cycle per day. The metric is **conversion_rate** (conversions / clicks) — higher is better.

## Setup

To set up a new experiment run, work with the user to:

1. **Read the in-scope files**:
   - `config.yaml` — campaigns, customer_id, ad groups, constraints, MCP query fields.
   - `product.md` — what the product is and what claims are safe to make.
   - `snapshot.py` — the data compression script you'll run each cycle. Do not modify.
2. **Verify MCP access**: Run a small test query to confirm Google Ads MCP responds. Query one campaign for `metrics.impressions` with a 1-day date range.
3. **Initialize results.tsv**: Create `results.tsv` with just the header row. The baseline will be recorded after the first cycle.
4. **Run the first cycle as baseline**: Pull data, compress, and log cycle 1 with status `baseline`. Do NOT generate or deploy copy on the first cycle — just establish the starting conversion_rate.
5. **Confirm and go**: Confirm setup looks good.

Once you get confirmation, begin daily experimentation.

## The Daily Cycle

Each cycle follows these steps in order.

### Step 0: Pre-flight

Before starting a cycle, run two checks:

1. **Idempotency** — Read `results.tsv`. If the last row's `date` column
   matches today's date (YYYY-MM-DD), STOP. Print
   `⚠️ Cycle already ran today ({date}). Skipping.` and exit.
   This prevents duplicate RSA creation if the agent is triggered twice.

2. **Cycle number** — Read the last row of `results.tsv`. The new cycle
   number is `last_cycle + 1`. If `results.tsv` has only the header,
   this is cycle 1. Never hard-code the cycle number.

### Step 1: Data Pull

Make MCP queries to get current campaign performance. Include conversion fields in every query.

**All data is pulled per-campaign to prevent MCP response truncation.** A single bulk query across all campaigns silently drops data.

Initialize partial file directory — **wipe any files from the previous
cycle first** so `aggregate.py` only sees this cycle's data:

```bash
rm -f data/partials/*.json
mkdir -p data/partials
```

**FOR each campaign** in the `campaigns` list from config.yaml:

Print: `📡 [{i}/{total}] Fetching: {campaign_name}`

1. **Structure** — campaign + ad group metrics
   - Query `campaign` resource with `campaign.name = '{campaign_name}'`
     - Fields from `config.yaml` → `mcp_fields.campaign`
     - Conditions: `campaign.status = 'ENABLED'`, date range, `metrics.impressions > 0`
   - Query `ad_group` resource with `campaign.name = '{campaign_name}'`
     - Fields from `config.yaml` → `mcp_fields.ad_group`
     - Conditions: `ad_group.status = 'ENABLED'`, date range, `metrics.impressions > 0`
   - Write to `data/partials/{campaign_name}-structure.json`.
     **Write MCP rows verbatim** — each row is a nested object as returned
     by the MCP `search` tool (`{"campaign": {...}, "metrics": {...}}`).
     Do NOT flatten into `{"name": "...", "metrics": {...}}` — `snapshot.py`
     uses dot-notation traversal (`campaign.name`, `ad_group.id`, etc.) and
     will produce `"unknown"` if the nesting is missing.
     ```json
     {"campaigns": [<raw MCP rows>], "ad_groups": [<raw MCP rows>]}
     ```

2. **Queries** — keywords + search terms
   - Query `keyword_view` with `campaign.name = '{campaign_name}'`
     - Fields from `config.yaml` → `mcp_fields.keyword`
     - Conditions: date range, `metrics.impressions > 0`
   - Query `search_term_view` with `campaign.name = '{campaign_name}'`
     - Fields from `config.yaml` → `mcp_fields.search_term`
     - Conditions: date range, `metrics.impressions > 10`
   - Write to `data/partials/{campaign_name}-queries.json`.
     Write MCP rows verbatim (nested objects, not flattened).
     ```json
     {"keywords": [<raw MCP rows>], "search_terms": [<raw MCP rows>]}
     ```

3. **Assets** — headline + description performance
   - Query `ad_group_ad_asset_view` with `campaign.name = '{campaign_name}'`
     - Fields from `config.yaml` → `mcp_fields.asset`
     - Conditions: date range, `metrics.impressions > 0`
   - Write to `data/partials/{campaign_name}-assets.json`.
     Write MCP rows verbatim (nested objects, not flattened).
     `aggregate.py` auto-converts numeric `field_type` enums to strings.
     ```json
     {"assets": [<raw MCP rows>]}
     ```

Print: `  ✓ {campaign_name} done ({N} assets, {M} keywords, {P} search terms)`

**END FOR**

**Aggregate** — After ALL campaigns are fetched, merge the partial files:

```bash
cd ~/autoresearch-ads && python3 aggregate.py
```

This reads all `data/partials/*-structure.json`, `*-queries.json`, and `*-assets.json` files and produces:
- `data/raw-structure.json` — all campaigns + ad groups merged
- `data/raw-queries.json` — all keywords + search terms merged
- `data/raw-assets.json` — all assets merged

**Date range**: Always compute explicit YYYY-MM-DD dates from config. Never use date literals like `LAST_7_DAYS`. Strip hyphens from customer_id before MCP calls.

### Step 2: Compress

Run the compression script:

```bash
cd ~/autoresearch-ads && python3 snapshot.py
```

This reads the 3 raw files, produces `data/snapshot.json`, and
automatically archives a copy to `memory/snapshots/snapshot-{date}.json`.

Read `data/snapshot.json`. This is your **sole data source** for all
analysis and decisions from this point forward. Do NOT use raw MCP
response data held in your context window — it may be incomplete,
uncompressed, or formatted differently than what `snapshot.py` produces.
If `snapshot.json` is missing or empty after running the script,
something went wrong in Steps 1–2; debug before proceeding.

### Step 3: Score Previous Experiments

**Data maturity gate.** Do NOT score an experiment until **both**
conditions are met:

1. The ad has accumulated **≥ 150 clicks**.
2. The ad has been live for **≥ 14 days** (compare `timestamp` in the
   experiment entry to today's date).

The click gate filters noise; the time gate ensures conversion
attribution has completed (Google Ads conversions can take 7–14 days
to attribute). When either condition is unmet, leave status as
`"launched"` and print:

```
⏳ {ad_group} ({experiment_id}): {clicks} clicks, {days}d live — need ≥ 150 clicks AND ≥ 14d. Skipping.
```

Read `memory/experiments.jsonl`. Find entries with `"status": "launched"`.
If there are none, this step is a no-op — skip to Step 4.

For each launched experiment, branch on `experiment_type`:

**`ad_variation`** — Google runs the A/B test; ask it for the verdict.
1. Call the `google-ads-write` MCP tool `get_experiment_status` with:
   - `customer_id` from `config.yaml`
   - `experiment_id` from the logged entry
2. The tool returns JSON with this shape:
   ```json
   {
     "experiment": {"resource_name": "...", "status": "RUNNING", ...},
     "arms": [
       {"name": "control",   "role": "control",   "metrics": {...}},
       {"name": "treatment", "role": "treatment", "metrics": {"conversion_rate": 0.034, ...}}
     ],
     "comparison": {
       "control_conversion_rate":   0.029,
       "treatment_conversion_rate": 0.034,
       "lift_vs_control":           0.172
     },
     "hint": "..."
   }
   ```
   The experiment status uses Google's enum: `SETUP`, `INITIATED`,
   `RUNNING`, `GRADUATED`, `HALTED`, `PROMOTED`, `REMOVED`.
3. Score using `comparison.lift_vs_control`:
   - `SETUP` / `INITIATED` / `RUNNING` → leave as `"launched"` (still collecting)
   - `lift_vs_control >= thresholds.winner` → status `"scored"`, outcome `"winner"`
   - `lift_vs_control <= thresholds.loser` → status `"scored"`, outcome `"loser"`
     (`thresholds.loser` is negative in `config.yaml`)
   - Otherwise → status `"scored"`, outcome `"inconclusive"`
4. Record `result_conversion_rate` = `comparison.treatment_conversion_rate`.
5. If `comparison.control_conversion_rate` is `0` or `null`, fall back to
   the **zero-original rules** (see direct_swap step 3 below) using the
   treatment arm's absolute metrics.
6. If outcome is `"winner"`, you may call `graduate_experiment` in Step 7
   of the NEXT cycle to promote it (always `validate_only: true` first).

**`direct_swap`** (default, and any legacy entry missing `experiment_type`)
— text-match against the snapshot.
1. **Normalize both `new_text` and the snapshot asset text** before
   comparing. Apply, in order:
   - Unicode NFKC normalization (`unicodedata.normalize("NFKC", s)`)
   - Lowercase
   - Strip leading/trailing whitespace
   - Strip trailing `.`, `!`, `?`, `,`, `:`, `;`
   - Replace smart quotes (`'`, `'`, `"`, `"`) with straight quotes
   - Replace en-dash (`–`) and em-dash (`—`) with hyphen (`-`)
2. **Scope the match to the entry's `ad_group`** — only compare against
   assets where `snapshot_asset["ad_group"] == entry["ad_group"]`. If
   multiple matches survive, pick the one with the most impressions.
3. If no asset found: leave as `"launched"`. If found but
   `impressions < thresholds.min_impressions`: leave as `"launched"`.
4. Compare `result_conversion_rate` (from the snapshot) to
   `original_conversion_rate`:
   - **If `original_conversion_rate > 0`** (the normal case):
     - **Winner**: `result >= original × (1 + thresholds.winner)`
     - **Loser**: `result <= original × (1 + thresholds.loser)` — note
       that `thresholds.loser` is negative in `config.yaml`
     - **Inconclusive**: between
   - **If `original_conversion_rate == 0`** (zero-original rules):
     The multiplicative thresholds collapse (`0 × anything == 0`), so
     use absolute floors instead:
     - **Winner**: `result_conversion_rate >= 0.02` AND `conversions >= 3`
     - **Loser**: `clicks >= 50` AND `conversions == 0`
     - **Inconclusive**: anything else (record `result_conversion_rate`
       anyway so the next cycle can re-score with more data)
5. Set `status` to `"scored"` and record the outcome and
   `result_conversion_rate`.

**Writing updates back to `memory/experiments.jsonl`** — use atomic
write semantics: write the new contents to a temp file in the same
directory, then `os.replace(temp_path, target_path)`. This guarantees
the file is never partially written even if the agent crashes mid-cycle.

```python
import os, tempfile
fd, temp_path = tempfile.mkstemp(
    prefix="experiments-",
    suffix=".jsonl.tmp",
    dir=os.path.dirname(EXPERIMENTS_PATH),
)
with os.fdopen(fd, "w") as f:
    for entry in entries:
        f.write(json.dumps(entry) + "\n")
os.replace(temp_path, EXPERIMENTS_PATH)
```

### Step 4: Analyze

**Cycle mode.** Each cycle operates in one of three modes. Determine the
mode before doing any analysis:

1. **Observe** — No experiments crossed the 150-click scoring threshold
   AND no ad group has ≥ 100 clicks/30d available for a new experiment.
   Pull data, update snapshot, log the observation row in `results.tsv`
   with `status: "observe"` and `assets_deployed: 0`. Skip Steps 5–7.
2. **Score** — One or more experiments crossed the 150-click threshold.
   Score them in Step 3. If no ad group has ≥ 100 clicks/30d available
   for a new experiment, skip Steps 5–7 after scoring.
3. **Act** — At least one ad group has ≥ 100 clicks/30d AND either no
   experiment is currently running in that group, or a scored experiment
   freed it up. Proceed through Steps 5–7, but limit to **1–2 new
   experiments per cycle**, prioritised by click volume descending.

Print the mode at the top of your analysis:

```
🔄 Cycle mode: OBSERVE | SCORE | ACT
```

Read `data/snapshot.json`. Identify:

- Which ad groups have the worst conversion rates?
- Which assets have high clicks but low conversions (wasted spend)?
  - Analyze **headlines** (`snapshot.json → assets.headlines`) and **descriptions** (`snapshot.json → assets.descriptions`) separately. Both are scoreable and both should produce proposals in Step 5.
- Which assets convert best? What do they have in common?
- What search terms lead to conversions vs which don't?
- **Negative keyword candidates** — score `snapshot.json → search_terms` for wasted spend:
  1. Collect all terms where `clicks >= 30` AND `conversions == 0`.
  2. **Filter out** any term that is already a negative keyword in the account (query `ad_group_criterion` or `campaign_criterion` resources for `type = NEGATIVE` if needed — skip this sub-step if it would require extra MCP calls that risk truncation; flag the uncertainty instead).
  3. **Filter out** any term whose text overlaps with an active positive keyword in the same ad group (a negative that matches a keyword suppresses your own ads).
  4. **Filter out** brand terms listed in `product.md`.
  5. For surviving candidates, note: term text, ad group, clicks, estimated cost (`clicks × avg_cpc`), and conversions. Also classify the failure pattern (price intent: "free/pricing/cost"; informational: "tutorial/reddit/how to"; wrong product: competitor names not in scope; etc.) — this helps choose match type.
- **Keyword expansion candidates** — mine `snapshot.json → search_terms` for coverage gaps:
  1. Collect all terms where (`conversions >= 1`) OR (`clicks >= 20` AND `ctr >= 1.5 × account median CTR`).
  2. **Filter out** terms already covered by an existing keyword. Apply match-type logic in Python:
     - **Exact** `[kw]`: covered if search term == keyword text (case-insensitive).
     - **Phrase** `"kw"`: covered if keyword text is a contiguous substring of the search term.
     - **Broad** `kw`: covered if all words in the keyword appear in the search term (any order).
     If the term is covered at any match type, skip it.
  3. **Filter out** brand terms and competitor names already in your keyword list (they're likely already targeted).
  4. For surviving candidates, note: term text, ad group, conversions, clicks, CTR. Choose match type: `exact` for long-tail (3+ words, converting); `phrase` as default.
- What did you learn from scored experiments in Step 3?

Read `memory/learnings.md` if it exists. Factor in cumulative knowledge.

You decide what matters. There are no prescribed frameworks — look at the data and find the signal.

### Step 5: Generate Copy

**Volume gate.** Do NOT propose copy changes for any ad group with
< 100 clicks in the last 30 days. At that volume, there is no way to
validate a copy hypothesis within a single scoring window. For
low-volume groups, the only permissible actions are negative keywords
and pauses.

**Concurrency limit.** Do NOT propose a new copy experiment in an ad
group that already has a `"status": "launched"` experiment in
`memory/experiments.jsonl`. Wait for the existing experiment to be
scored (or manually cancelled) first. Running overlapping experiments
in the same group makes attribution impossible.

**Experiment cap.** Propose at most **2 copy experiments per cycle**,
in the highest-volume eligible ad groups. This ensures each experiment
gets the full data window rather than splitting attention.

Based on your analysis, propose copy changes. For each change:
- **What you're replacing**: the specific asset text and its current conversion_rate.
- **What you're proposing**: new text.
- **Why**: your hypothesis for why this will convert better.
- **Where**: which ad group(s).

Constraints (from config.yaml):
- Headlines: max `headline_max_chars` characters.
- Descriptions: max `description_max_chars` characters.
- Claims must be grounded in `product.md`. Do not invent features or numbers.
- Count characters before finalizing. If over limit, rewrite.

**Critical: only propose swaps on assets that are in a CURRENTLY-ENABLED
RSA.** The snapshot's `assets` and `conversion_insights` sections
include data from paused ads too — those ads still hold 30-day metrics
even though they no longer serve. If you propose replacing a headline
that lives in a paused ad, Step 7 will skip the proposal because there
is nothing to swap. Before writing each proposal, verify the original
text exists in the current winning ad by calling
`get_ad(ad_group_id, select_winning=true)` and checking its `headlines`
list. (You can batch this: call `get_ad` once per ad group you have
proposals for, then validate all your proposals against the returned
headlines.)

Write proposals to `copy.json`. Three proposal types are valid — always include all types when the data supports it:

```json
{
  "cycle": 5,
  "date": "2026-04-08",
  "proposals": [
    {
      "type": "headline",
      "ad_group": "claude_code",
      "original": "Free with Claude Code Sub",
      "original_conversion_rate": 0.028,
      "new_text": "Your proposed headline here",
      "hypothesis": "Why you think this converts better"
    },
    {
      "type": "description",
      "ad_group": "claude_code",
      "original": "Already have a Claude Code subscription? Use Cosmos to complete...",
      "original_conversion_rate": 0.0,
      "new_text": "Your proposed description here",
      "hypothesis": "Why you think this converts better"
    },
    {
      "type": "negative_keyword",
      "ad_group": "claude_code",
      "scope": "ad_group",
      "term": "free claude code download",
      "match_type": "exact",
      "clicks": 89,
      "cost_usd": 162.40,
      "conversions": 0,
      "hypothesis": "89 clicks, $162 spent, 0 conv. Price-intent modifier — confirmed loser pattern across the account."
    },
    {
      "type": "keyword_expansion",
      "ad_group": "claude_code",
      "term": "claude code parallel agents",
      "match_type": "phrase",
      "conversions": 2,
      "clicks": 15,
      "ctr": 0.18,
      "hypothesis": "Converts at 13.3% CR, not in keyword list. Phrase match captures volume from this intent without over-specifying."
    }
  ]
}
```

**`negative_keyword` proposal rules:**
- `scope`: `"ad_group"` (default, precise) or `"campaign"` (for universal failure patterns like "free", "reddit", "how to" that fail across every ad group).
- `match_type`: default to `"exact"`. Use `"phrase"` only for clear pattern prefixes/suffixes (e.g. "free [anything]"). Never use `"broad"` for negatives — it is too aggressive and can suppress legitimate traffic.
- Cap at **10 negative keyword proposals per cycle** to prevent over-pruning. Prioritise by cost_usd descending.
- Do NOT propose the same term that was proposed (and logged) in a prior cycle — check `memory/experiments.jsonl` for existing `"type": "negative_keyword"` entries before writing proposals.

**`keyword_expansion` proposal rules:**
- `match_type`: `"exact"` for long-tail converting terms (3+ words); `"phrase"` as default. Never propose `"broad"` — broad match is already handled by parent keywords.
- Cap at **5 expansion keywords per cycle** to avoid keyword list bloat. Prioritise by conversions desc, then clicks desc.
- Do NOT propose a term already proposed in a prior cycle — check `memory/experiments.jsonl` for existing `"type": "keyword_expansion"` entries.

Git commit and push: `cycle N: propose copy changes`

### Step 6: Review Gate

**STOP here and present your proposals to the human.** Print a clear summary, grouped by type:

```
═══════════════════════════════════════════════════
  PROPOSALS READY FOR REVIEW — Cycle N
═══════════════════════════════════════════════════

  COPY CHANGES (headlines + descriptions)
  ───────────────────────────────────────────────
  AD GROUP: claude_code
  REPLACE:  "Old Headline Text" (conv_rate: 0.15%)
  WITH:     "New Headline Text" (27 chars)
  WHY:      Your hypothesis

  NEGATIVE KEYWORDS
  ───────────────────────────────────────────────
  AD GROUP: claude_code  |  SCOPE: ad_group
  BLOCK:    "free claude code download" [exact]
  DATA:     89 clicks · $162 spent · 0 conv
  WHY:      Your hypothesis

  KEYWORD EXPANSIONS
  ───────────────────────────────────────────────
  AD GROUP: claude_code
  ADD:      "claude code parallel agents" [phrase]
  DATA:     15 clicks · 2 conv · 13.3% CR
  WHY:      Your hypothesis

  ───────────────────────────────────────────────
  Copy changes:       N (X headlines, Y descriptions)
  Negative keywords:  N (estimated $X saved/30d)
  Keyword expansions: N
  Total ad groups affected: M
═══════════════════════════════════════════════════
```

Wait for the human to respond. They will either:
- **Approve all**: proceed to Step 7 (Deploy)
- **Approve some**: they'll tell you which ones. Deploy only those.
- **Reject all**: skip Step 7, go straight to Step 8 (Log). Log proposals with status `"rejected"` instead of `"launched"`.
- **Request changes**: revise the proposals and present again.

Do NOT proceed to deployment without explicit human approval.

### Step 7: Deploy

**Background.** Google Ads blocks `AdService.MutateAds UPDATE` on responsive
search ad headlines and descriptions — they are immutable once the ad is
created. You cannot edit individual assets on an existing RSA. To change
copy, you must either (a) create a new RSA and pause the old one ("direct
swap") or (b) run an Ad Variation experiment (statistical A/B test).

Both paths go through the `google-ads-write` MCP, which handles all
reads and writes. It exposes:
- `search` — queries any Google Ads resource (campaigns, ad groups,
  keywords, search terms, assets, etc.)
- `get_ad` — fetches an RSA's full content (headlines, descriptions,
  final URLs, pinned positions). Used in Step 7.2 below.
- `create_responsive_search_ad` — creates a new RSA in an ad group
- `pause_ad` — pauses an existing ad by resource name
- `create_ad_variation` — creates an Ad Variation experiment against a base ad
- `get_experiment_status` — reads experiment status + per-arm metrics
  (used in Step 3)
- `graduate_experiment` — promotes a winning variation

**Decision: direct_swap vs ad_variation.**
- **ad_variation (default for converting groups)** — Use whenever the ad
  group has **≥ 1 conversion in the last 30 days**. This protects the
  existing converting ad by keeping it as the control while the variant
  collects data. The C2 claude_code direct swap destroyed a 1.56% CR ad
  and cost ~$640 — that mistake is never repeated.
- **direct_swap** — Use ONLY when the ad group has **0 conversions in the
  last 30 days**. There is nothing to protect, so a direct swap is safe
  and faster.

This is not optional. If the ad group converts, you MUST use ad_variation.

**1. Group approved proposals by `ad_group`.**
   All proposals for the same ad group become a single new RSA creation.
   You cannot swap different headlines on the same RSA in the same cycle
   with separate calls — you must build the full new ad.

**2. For each ad_group, fetch the current winning RSA.** Look up the
ad group's resource name from `snapshot.json` (`ad_groups[].id`) and
construct it as `customers/{customer_id}/adGroups/{id}`. Then call
the `google-ads-write` MCP tool `get_ad`:

```json
{
  "customer_id": "9232939339",
  "ad_group_id": "customers/9232939339/adGroups/191506291605",
  "select_winning": true
}
```

`get_ad` returns JSON with this shape:

```json
{
  "resource_name": "customers/9232939339/adGroupAds/191506291605~803604473915",
  "ad_id": "803604473915",
  "status": "ENABLED",
  "final_urls": ["https://www.example.com/landing"],
  "headlines": [
    {"text": "Headline One", "pinned_field": null},
    {"text": "Headline Two", "pinned_field": "HEADLINE_1"}
  ],
  "descriptions": [
    {"text": "Description One", "pinned_field": null}
  ],
  "metrics": {"impressions": 2232, "clicks": 69, "conversions": 0}
}
```

Call this the *base ad*. `resource_name` is `old_ad_id`; `headlines` /
`descriptions` are the starting set; preserve each headline's
`pinned_field` (already normalised — `null` means unpinned).

**3. Build the new RSA's copy.**
   - Start with the base ad's headlines and descriptions.
   - For each proposal in the group, **find the matching base asset by
     normalised text comparison**. Apply, in order, to both `original`
     and the base asset's `text`:
     - Unicode NFKC normalization
     - Lowercase
     - Strip leading/trailing whitespace
     - Strip trailing `.`, `!`, `?`, `,`, `:`, `;`
     - Replace smart quotes (`'`, `'`, `"`, `"`) with straight quotes
     - Replace en-dash (`–`) and em-dash (`—`) with hyphen (`-`)
     **Scope the search by `type`**: proposals with `"type": "headline"`
     match only against `base.headlines`; proposals with
     `"type": "description"` match only against `base.descriptions`.
     If no asset matches the proposal's `original`, log a warning and
     skip that proposal — do NOT silently invent the swap.
   - Replace the matched asset's `text` with `new_text` while
     preserving its `pinned_field`.
   - RSAs require 3–15 headlines and 2–4 descriptions; keep the base ad's
     count.
   - Final URL = `base.final_urls[0]`.
   - Count characters on every headline (≤ `constraints.headline_max_chars`)
     and description (≤ `constraints.description_max_chars`) before calling
     the API. Rewrite if over.

**4. Deploy — `direct_swap` path.**
   a. Call `create_responsive_search_ad` with `validate_only: true`. If it
      returns an error, fix the inputs and retry from step 3.
   b. Call `create_responsive_search_ad` with `validate_only: false`.
      Record the returned resource name as `new_ad_id`.
   c. Call `pause_ad` with `ad_id = old_ad_id` and `validate_only: true`
      first, then `validate_only: false`. Record success.
   d. If pause fails, DO NOT revert the new ad — you now have two enabled
      RSAs in the ad group, which is still better than zero. Log the
      failure and continue.

**5. Deploy — `ad_variation` path (only when explicitly requested).**
   a. Call `create_ad_variation` with `validate_only: true`.
   b. Call `create_ad_variation` with `validate_only: false`. Record the
      returned experiment resource name as `experiment_id` and the variant
      arm's new ad as `new_ad_id`. `old_ad_id` = the base ad whose variation
      this is.
   c. Do NOT call `pause_ad` — the base ad must keep serving as the control.

**6. Deploy — `negative_keyword` path.**

   The `google-ads-write` MCP exposes `add_negative_keywords`. For each
   approved `negative_keyword` proposal:

   a. Call `add_negative_keywords` with `validate_only: true` first.
   b. Call `add_negative_keywords` with `validate_only: false`.
      - `customer_id` from `config.yaml`
      - `campaign_id`: the campaign resource name for the ad group
      - `keywords`: array of `{"text": "<term>", "match_type": "<EXACT|PHRASE>"}`
   c. Record the returned resource name as `criterion_resource_name`
      and set `status: "launched"`.
   d. If the call fails, log with `"status": "deployment_failed"` and
      `"failure_reason": "<error>"`.

**7. Deploy — `keyword_expansion` path.**

   The `google-ads-write` MCP exposes `add_keywords`. For each approved
   `keyword_expansion` proposal:

   a. Call `add_keywords` with `validate_only: true` first.
   b. Call `add_keywords` with `validate_only: false`.
      - `customer_id` from `config.yaml`
      - `ad_group_id`: the ad group resource name
      - `keywords`: array of `{"text": "<term>", "match_type": "<EXACT|PHRASE>"}`
   c. Record the returned resource name as `criterion_resource_name`
      and set `status: "launched"`.
   d. If the call fails, log with `"status": "deployment_failed"` and
      `"failure_reason": "<error>"`.

**8. Error handling.**
   Always run `validate_only: true` first on every mutate call. If any
   write step fails, log the failure (with `error_code` and `request_id`)
   and continue with the remaining ad groups. Partial deploys are
   acceptable; all-or-nothing is not required.

Log each action with full resource names as you go — these are the inputs
to Step 8 and to Step 3 of future cycles.

### Step 8: Log

1. **experiments.jsonl** — Append one entry per deployed proposal.

**For `headline` and `description` entries:**
```json
{
  "id": "2026-04-08-001",
  "cycle": 5,
  "timestamp": "ISO-8601",
  "experiment_type": "direct_swap",
  "type": "headline",            // "headline" or "description"
  "ad_group": "claude_code",
  "original": "Free with Claude Code Sub",
  "original_conversion_rate": 0.028,
  "new_text": "Your proposed text here",
  "hypothesis": "Why you think this converts better",
  "old_ad_id": "customers/9232939339/adGroupAds/123~456",
  "new_ad_id": "customers/9232939339/adGroupAds/123~789",
  "experiment_id": null,
  "status": "launched",
  "result_conversion_rate": null,
  "outcome": null
}
```

**For `negative_keyword` entries:**
```json
{
  "id": "2026-04-08-002",
  "cycle": 5,
  "timestamp": "ISO-8601",
  "experiment_type": "negative_keyword",
  "type": "negative_keyword",
  "ad_group": "claude_code",
  "scope": "ad_group",
  "term": "free claude code download",
  "match_type": "exact",
  "clicks": 89,
  "cost_usd": 162.40,
  "conversions": 0,
  "hypothesis": "Why this term wastes spend",
  "criterion_resource_name": null,
  "status": "pending_tool",
  "failure_reason": "add_negative_keyword MCP tool not yet available"
}
```

**For `keyword_expansion` entries:**
```json
{
  "id": "2026-04-08-003",
  "cycle": 5,
  "timestamp": "ISO-8601",
  "experiment_type": "keyword_expansion",
  "type": "keyword_expansion",
  "ad_group": "claude_code",
  "term": "claude code parallel agents",
  "match_type": "phrase",
  "conversions": 2,
  "clicks": 15,
  "ctr": 0.18,
  "hypothesis": "Converts at 13.3% CR, not in keyword list.",
  "criterion_resource_name": null,
  "status": "pending_tool",
  "failure_reason": "add_keyword MCP tool not yet available"
}
```

Keyword expansion entries do **not** have `result_conversion_rate` or `outcome` fields
in the traditional sense — success is measured by whether the keyword starts accumulating
impressions and converting in subsequent snapshots. Step 3 skips `keyword_expansion`
entries when scoring. To verify impact, check the keyword in the next snapshot's
`keywords` section for impression and conversion data.

Negative keyword entries do **not** have `result_conversion_rate` or `outcome` fields —
there is no before/after CR to measure once a term is blocked. Step 3 skips
`negative_keyword` entries entirely when scoring. To verify impact, compare
the term's click volume across snapshots: if it disappears, the negative is working.

Field reference:
- `experiment_type` — `"direct_swap"`, `"ad_variation"`, or `"negative_keyword"`.
- `old_ad_id` — resource name of the ad that was paused (direct_swap) or
  of the base ad the variation runs against (ad_variation). Never null on
  a successful deploy. Not present on `negative_keyword` entries.
- `new_ad_id` — resource name of the newly created ad. Never null on a
  successful deploy. Not present on `negative_keyword` entries.
- `criterion_resource_name` — resource name returned by `add_negative_keywords`
  or `add_keywords` on a successful deploy. Null if the deploy failed.
- `experiment_id` — resource name of the `experiment` resource for
  ad_variation entries; always null for direct_swap, negative_keyword, and keyword_expansion.

If a proposal fails to deploy, append it anyway with
`"status": "deployment_failed"`, `"failure_reason": "<error_code> <message>"`,
and `new_ad_id` / `experiment_id` set to null. Legacy entries from before
this schema change (no `experiment_type` field) are treated as `direct_swap`
by Step 3.

2. **results.tsv** — Append one row for this cycle:
```
cycle	date	conv_rate	cost_per_conv	assets_deployed	winners	losers	status	description
```

3. **learnings.md** — Regenerate from experiments.jsonl. Include two sections:

   **Copy learnings** (what works / doesn't in ad creative):
   - Patterns that have won (with data)
   - Patterns that have lost (with data)
   - Rules derived from >= 3 data points

   **Meta-learnings** (what works / doesn't in the agent's own process):
   - Process mistakes and corrections (e.g. "scoring at <150 clicks led to
     2 premature rollbacks in C5 and C8")
   - Volume observations (e.g. "3 consecutive copy attempts on coding_agent
     (9 clicks/mo) all failed — volume was the issue, not copy")
   - When to give up on copy and flag an ad group as audience/LP problem
   - Patterns in experiment timing, approval delays, or data gaps that
     affected outcomes

**Step 8 completion gate — do NOT proceed to Step 9 until all three are verified:**
- [ ] `experiments.jsonl` — new entries appended for this cycle
- [ ] `results.tsv` — new row appended for this cycle
- [ ] `learnings.md` — regenerated with this cycle's scored outcomes, confirmed losers/winners, and updated rules

If `learnings.md` was not updated, update it NOW before continuing. Stale learnings cause the agent to repeat mistakes in future cycles.

Git commit and push: `cycle N: deploy + log results`

### Step 9: Stop

Print a cycle summary:

```
---
cycle:              5
date:               2026-04-08
conversion_rate:    0.0355
cost_per_conversion: 27.80
assets_deployed:    5
experiments_scored:  3 (2 winners, 1 loser)
cumulative_winners: 8
cumulative_losers:  4
---
```

Stop. The next cycle runs tomorrow when conversion data has accumulated.

## Scoring Rules

- **Primary metric**: `conversion_rate` = conversions / clicks
- **Secondary metric**: `cost_per_conversion` = cost / conversions
- **Winner threshold**: >= 20% improvement in conversion_rate over original
- **Loser threshold**: >= 20% decline in conversion_rate vs original
- **Minimum to score**: **≥ 150 clicks AND ≥ 14 days live** (data maturity
  gate). Both conditions must be met. Do not score any experiment below
  either threshold — leave it as `"launched"`.
- **Confidence**: `high` if clicks >= 500, `medium` 150-499, `low` < 150 (not scoreable)

## results.tsv Format

Tab-separated, NOT comma-separated.

```
cycle	date	conv_rate	cost_per_conv	assets_deployed	winners	losers	status	description
1	2026-04-08	0.0340	29.58	0	0	0	baseline	initial snapshot
2	2026-04-09	0.0355	27.80	5	0	0	keep	replaced 3 headlines in claude_code
```

Status values: `baseline`, `keep`, `discard` (if overall conversion_rate dropped).

## Context Retention

You will forget data between conversation turns. Protect against this:

1. **Never hold raw MCP data in memory** — write it to `data/raw-*.json` immediately.
2. **Always work from files** — read `snapshot.json` for analysis, `experiments.jsonl` for history, `learnings.md` for patterns.
3. **snapshot.py does the compression** — you don't manually summarize MCP data. Run the script.
4. **One snapshot per day** — archived in `memory/snapshots/` for trend analysis.

If you're ever unsure about current state, re-read `data/snapshot.json` and `memory/experiments.jsonl`.

## What You Cannot Do

- Modify `snapshot.py`. It is read-only.
- Modify `config.yaml` (unless the human asks you to).
- Modify `product.md` (unless the human asks you to).
- Invent features or claims not in `product.md`.
- Deploy copy that exceeds character limits.
- Skip the data pull — always start with fresh MCP data.

## What You Can Do

Everything else. There are no prescribed frameworks, angles, styles, or strategies. You look at the data, form hypotheses, test them, and learn from the results. If something works, do more of it. If it doesn't, try something different.

## NEVER STOP

Once the daily cycle begins (after setup), do NOT pause to ask the human if you should continue. Complete all steps — but always stop at Step 6 (Review Gate) for human approval before deploying. The human may be away from their computer. Run the full cycle autonomously. If an MCP query fails, retry once. If it fails again, log the failure and continue with what you have.

Between cycles (i.e., when you've completed Step 8), you stop. The human or a scheduler triggers the next cycle.
