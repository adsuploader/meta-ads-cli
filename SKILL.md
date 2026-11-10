---
name: ads
description: Create, manage, and report on Facebook/Meta ads using the Ads Uploader CLI. Upload media, preview configurations, create ads, browse campaigns and ad sets, and query bounded Ads Insights data.
homepage: https://adsuploader.com
---

## Setup

```bash
npm install -g @adsuploader/cli
```

## Authentication

```bash
# 1. Login (browser, or device code on a server)
ads login

# 2. List ad accounts
ads accounts

# 3. Set default account
ads account act_123456789
```

Credentials are stored at `~/.config/adsuploader/credentials.json`. Check current auth:

```bash
ads whoami
ads config
```

## Logging in on a server (no browser)

On a VPS, in SSH, or on a machine without a browser, run:

```bash
ads login --device
```

Open the printed URL on your phone or laptop, enter the code, and approve the device after checking its country and network. The code expires after 10 minutes. The CLI saves a 30-day login on the server. `ads login` automatically uses this flow over SSH or on Linux without a display, and falls back to it if opening the browser fails. Use `ads login --browser` to force the localhost browser flow.

## API keys for scripts and servers

For unattended scripts, open **Account > MCP, CLI & API**, choose **Create API Key**, and give it a name. Copy the `adk_` secret when it appears; it is shown only once. You can have 1 active key by default. Add a team seat for another person, or contact support for more keys for your own servers. Keys can be used from different networks and require CLI access on your account. Keys you don't use for 90 days are removed. Time when your plan is paused doesn't count.

Set `ADS_API_KEY` through your server or CI secret settings. It overrides the token saved in `~/.config/adsuploader/credentials.json`. To save a key locally instead, run:

```bash
ads login --api-key
```

Paste the key at the hidden prompt, or pipe it through stdin. The CLI verifies it before saving it. `ads login --api-key <key>` also works, but putting a secret in the command can expose it in shell history. `ads whoami` and `ads config` show the authentication method; only the key prefix is displayed.

Revoke a key under **Account > MCP, CLI & API** when it is no longer needed. `ads logout` removes a saved API key locally without revoking it elsewhere. It cannot unset `ADS_API_KEY`; remove that variable from your environment or secret settings. Browser and device sessions can be removed under **MCP and CLI-Connected AI Apps**; logging out of a session also attempts to revoke it.

---

## Core Workflow

Every ad creation follows three steps:

### 1. Upload media

```bash
ads upload hero.jpg banner.mp4
ads upload ./my-creatives/          # entire directory
ads upload hero.jpg "https://cdn.example.com/banner.mp4"  # same batch
ads upload "https://drive.google.com/file/d/.../view"      # public Drive file link
ads upload --retry-failed           # retry latest failed batch
ads upload --retry-failed batch_abc123
ads upload:drive "https://drive.google.com/drive/folders/..."
```

Returns a **batch ID** (e.g. `batch_abc123`) — you'll need this for the spec file. Files are uploaded directly to the selected ad account's Facebook media library, so the batch ID is tied to that account.

Files are staged in parallel and transient network failures are retried automatically. Tune the defaults if needed:

| Upload option | Description |
|---------------|-------------|
| `--concurrency <n>` | Parallel staging uploads, 1-6 (default: 4) |
| `--upload-timeout <ms>` | Per-file R2 upload timeout (default: 120000) |
| `--api-timeout <ms>` | API request timeout (default: 60000) |
| `--retry-failed [batchId]` | Retry failed local files into the same batch; defaults to the latest saved failed batch |

`ads upload` infers each argument as a local path or HTTPS URL. It uploads local paths first, then sends URLs through a server-side ingest job using the same batch ID, so grouping covers both sets. Google Drive file links are detected automatically.

For `upload:drive`, the folder must be shared as **Anyone with the link (Viewer)**. The server lists the folder and runs bounded parallel download, staging, and processing pipelines in a background job. The CLI polls a completed/total counter and shows every currently active file. Press Ctrl-C once to cancel the server-side import and wait briefly for its summary; a second Ctrl-C exits immediately. If Google temporarily blocks automated access, report the completed, failed, and skipped counts and tell the user to retry the command later. The final summary reports how many `*_thumbnail` images were attached to their videos as custom thumbnails instead of being counted as separate media — report that line rather than treating the lower total as a missing file.

### 2. Preview (dry run)

```bash
ads create:preview spec.json
```

Shows exactly what would be created without creating Meta ads. **Always preview first.**

### 3. Create ads

```bash
ads create spec.json
```

Ads are created **ACTIVE** by default. Use `--status PAUSED` to create them paused. Returns a job ID for progress tracking.

### Saved build handoffs

Use `ads builds:create --spec spec.json --name "Summer Sale"` to save a spec as a new build. `builds:save` is an alias. It also takes `--notes`, `--account` and `--web-state <file>`; supply `--spec`, `--web-state` or both. Use `ads builds:delete <buildId>` to delete a saved build.

Run `ads builds:update` calls for a build one at a time, never in parallel. Changes to `spec`, `webState`, `mediaItems` or `uploadBatchId` bump the revision. A rename or notes-only update does not bump it. On "Build changed since it was read" (HTTP 409), re-read the build with `ads builds <buildId> --json` and retry with its current revision as `--expected-revision`.

When you update an existing saved build for someone working in an open uploader tab, confirm that the update was saved; the tab picks it up automatically. Share the build URL only if the user asks for it or says the tab is closed. For a new headless build with no open uploader tab, include the web URL so the user can open and review it.

Build detail omits `webState` by default; use `ads builds <buildId> --include webState --json` when you need the opaque uploader snapshot.

Use `ads builds:fork <buildId>` when the user wants a variant but the original build must remain unchanged. Continue edits against the new build reference printed by the command.

---

## Command Reference

### Auth
| Command | Description |
|---------|-------------|
| `login` | Browser login or automatic device login; supports `--device`, `--browser`, `--api-key [key]` |
| `logout` | Clear credentials |
| `whoami` | Show current user |
| `config` | Show configuration |

### Browsing
| Command | Description |
|---------|-------------|
| `accounts` | List ad accounts |
| `accounts:refresh` | Fetch the ad account list from Meta again right away, for example after you get access to a new ad account |
| `account <id>` | Set default account |
| `pages` | List Facebook Pages available for `--page` / `profile.pageId` |
| `campaigns` | List active campaigns (`--status all` for all, `--search <text>` to filter) |
| `campaign <id>` | Show ad sets in a campaign |
| `adsets [--campaign <id>]` | List ad sets in a campaign, or across the account without `--campaign` (supports `--search`, `--status`); account-wide rows show each ad set's campaign |
| `adset <adSetId>` | Show ads in an ad set |
| `ads [--adset <id>]` | List ads in an ad set, or across the account without `--adset` (supports `--search`, `--status`); account-wide rows show each ad's ad set and campaign |
| `ad <adId>` | Ad details + creative config, plus Meta's rejection reasons and issues when the ad is DISAPPROVED or WITH_ISSUES |
| `insights` | Query one bounded page of Meta Ads Insights |
| `insights:job <jobId>` | Check or page an async Insights report |
| `insights:fields` | List supported Insights fields, presets, breakdowns, and limits |
| `presets` | List saved API presets (or `presets <id>` for details) |
| `presets:save --from-ad <adId> --name <name>` | Save an existing ad as an API preset |
| `text-presets` | List saved text presets (or `text-presets <id>` for details) |
| `uploads` | List recent upload batches (or `uploads <batchId>` for details) |
| `targeting:search <query> --type city` | Find `targeting.cities[].key` values |
| `targeting:search <query> --type zip` | Find `targeting.zips[].key` values for zip / post code targeting |
| `targeting:search <query> --type detailed` | Find `targeting.detailedTargetingGroups` entries: interests, behaviors, employers, job titles, industries |

### History

`ads history` shows the same launches and changes as the Ads Uploader History page, newest first: Uploader, Duplicator, AI Edits, CLI, MCP and API activity. Use `ads history launch <launchId>` for campaigns, ad sets, ads, results and failure messages from one launch.

```bash
ads history --account act_123 --days 30
ads history --type changes --source mcp --search budget
ads history launch <launchId> --json
```

Both commands use `--account` or your default account. List filters are `--days 7|30|90` (default 30; History covers the last 90 days), `--type all|launches|changes` (default all), `--source web|duplicator|mcp|cli|api`, `--search <text>` (up to 200 characters), and `--member <id>` (a team member ID). Use `--limit 1..100` (default 50) and the printed `--before <cursor>` hint for more. Keep the same filters when paging. Tables show Time, Item, Type, Change, Before, After, Source and Status, plus By for teams. `--json` returns the full response unchanged, including paging and pending suggestions. The History page is open to everyone; reading History through the API, CLI and MCP is in beta.

`ads meta-activity` reads Meta's ad account activity log, including edits in Ads Manager and other tools. It retains `--since`, `--until`, `--object`, `--after`, `--limit`, `--fresh`, `--account` and `--json`. The default range is 7 days, with a maximum of 90 days.

`ads history --meta` and `ads history` with `--since`, `--until` or `--object` still read Meta activity for one release and print a deprecation note to stderr. Switch to `ads meta-activity`. The `ads edits` list is also deprecated; use `ads history --type changes`. `ads edit:show`, undo and `ads edits cancel` keep working.

### Agent Inbox history

`ads inbox:history` (also `ads inbox history`) reads your retained suggestions and direct edit outcomes from the last 90 days, newest first, without calling Meta. It includes agent notes, edited-note markers, run titles, client names, before/after values and outcomes. Read permission is required; hidden and Read-off accounts are excluded.

```bash
ads inbox:history --account act_123 --campaign 12345 --status applied --json
ads inbox history --account act_123 --since 2026-10-01 --until 2026-10-09
ads inbox:history --run <reviewId> --kind duplicate --limit 20
```

Use `--account` or your selected account; with neither, history covers all permitted accounts. Filters are `--campaign`, `--since`/`--until` (inclusive ISO dates or timestamps with a timezone), `--run <reviewId>`, `--status waiting|applied|failed|cancelled|skipped|stale`, `--kind edit|duplicate`, and `--action <operation>` such as `set_bid`, `rename` or `duplicate`. A date-only `--until` includes the whole UTC day. Waiting includes applying, and failed includes partial outcomes; `outcome` preserves the specific result. Time is resolution time when recorded, otherwise creation time. Direct edits can be filtered by campaign only when that campaign ID was recorded. Use `--limit 1..100` (default 50) and `--cursor` from the previous response, keeping the same filters. `--json` returns the full response including `paging.cursor`. The table's `revised` column shows the latest change made to a suggestion after the agent made it, such as `user restored agent value` or `user edited note`; `--json` includes every entry in `revisions` with `at`, `by` (`user`, or `agent` with `source` and `clientName`), `kind` (`value` or `note`), `action` (`edited` or `restored_to_agent`), `field`, `before` and `after`.

### Resource reads

| Command | What it reads |
|---|---|
| `ads account:info <id>` | Owned account status, currency, timezone, spend and funding display |
| `ads insights:health --campaign <id> --preset last_30d` | Campaign performance, native results, status, budget, issues and learning counts |
| `ads audiences --type all --search "Customers"` | Custom, lookalike and saved audiences |
| `ads conversions` | Pixels, custom conversion definitions and standard event names |
| `ads estimate --targeting targeting.json --goal REACH` | Predicted audience size and delivery curve |
| `ads meta-activity --since 2026-08-01 --until 2026-08-07` | Meta change history |
| `ads library:images` / `ads library:videos` | Media in the Meta ad account library |
| `ads posts --page <id>` / `ads posts --instagram <id>` | Accessible Page promotable posts or connected Instagram media |

Every command supports `--json` and `--fresh`. List commands accept `--limit` (default 25, maximum 100) and `--after`; health is capped at 25 campaigns. New resource commands return the full JSON response, including `paging.after`, `paging.hasMore` and cache metadata. `campaigns --json`, `adsets --json` and `ads --json` return arrays; `campaign <id> --json` and `adset <id> --json` retain their ID-and-list envelopes. These five commands print next-page hints to stderr in JSON mode, keeping stdout compatible with existing scripts. Keep filters unchanged when continuing. These reads make one request attempt; honor the retry wait on a 429.

Without `--campaign`, `ads adsets` lists ad sets across the whole account; without `--adset`, `ads ads` lists ads across the account. Add `--search` to find ad sets or ads by name when you don't know their campaign or ad set. Each row names its campaign by ID and name, and ads also name their ad set. Like every list, these return one page (default 25) and never page through the account on their own; use the printed `--after` hint for more. The default `--status active` leaves out paused items, so add `--status all` to include them.

`ads account <id>` selects the default account; `ads account:info <id>` reads details. Use `--account` on scoped commands to override the default. Campaign tables show real status, objective and budgets. Ad set tables show status, optimization goal and budgets; ad tables show status and creative IDs because ads have no independent budget.

Health answers “how is this campaign doing”; use `insights` for custom analysis. Results preserve each native indicator, value and attribution window. Missing delivery metrics are null. Learning counts cover at most 100 ad sets per campaign, with an explicit partial warning. Meta change history is separate from the Ads Uploader Activity Log, and the Meta library is separate from Ads Uploader uploads. History IDs are strings, including nested `extra_data` IDs. Integer literals of 16 or more digits in Meta history JSON are preserved exactly as strings before parsing; secret redaction and the 1 KB extra-data bound still apply.

## Insights Reporting

Discover field IDs before querying when the requested metric or dimension is uncertain:

```bash
ads insights:fields --recommended
ads insights:fields --search purchase --json
```

Unfiltered discovery returns groups and recommended fields. Use `--search purchase`, `--group results`, or `--ids purchases,roas` for full entries; add `--account act_123` for custom conversions (definitions refresh after 24 hours).

Use friendly fields such as `leads,purchases,cost_per_purchase,roas` for business questions, or `results,cost_per_result` for optimization goals at campaign, ad set or ad level. Missing native results can be derived from ad set setup and delivery, with `result_source: "derived"` and `result_indicator`; incomplete data stays unknown. Custom conversions use `custom_conversion.<id>`, `custom_conversion_value.<id>`, or `cost_per_custom_conversion.<id>`, with a numeric ID. See the [friendly metric formulas](https://adsuploader.com/docs/automation/insights#friendly-metrics-and-campaign-goals).

`--where "spend>10" --where "campaign.name~Summer"` adds value filters (repeatable, maximum 5). Shorthand operators are `> >= < <= = != ~`, where `~` means contains. Zero-impression exclusion is the default; `--exclude-zero-impressions` makes it explicit and `--include-zero-impressions` opts out. Compare retains delivery in either period. Names use string values; numeric metrics use numbers. Use REST or MCP for array filters.

Use `--breakdown user_segment_key` alone for audience segment (prospecting / engaged / existing customers) at account, campaign, ad set or ad level; combine with `--increment weekly` or `--increment monthly` for share of spend over time. Requires Audience Segments set up on the ad account.

Use one asset or media breakdown alone for flexible/dynamic creative; hourly breakdowns also require their own query. Object assets flatten into string columns such as `title_asset.text`. `--action-breakdown action_device` produces `actionRows` and `actionColumns`, with each action row linked to a base row by its zero-based `row` index. Do not combine non-default action breakdowns with friendly aliases.

`--json` keeps object rows by default. Add `--format table` for arrays in column order, including action rows. Missing table cells become null; use objects to distinguish absent keys. Both formats share the cached page. Native result arrays keep indexes, for example `results.0.indicator` and `results.0.values.0.value`; never sum their indicators or attribution windows.

Top ads by spend for the last 7 days:

```bash
ads insights --level ad --fields ad_id,ad_name,spend,impressions --preset last_7d --sort spend:desc --limit 10
```

Daily purchase actions and CPA:

```bash
ads insights --level account --fields date_start,date_stop,actions,cost_per_action_type --action-types offsite_conversion.fb_pixel_purchase --preset last_7d --increment daily
```

Placement performance:

```bash
ads insights --fields spend,impressions,clicks,cpm --preset last_7d --breakdown publisher_platform --breakdown platform_position
```

Use a comparison for "versus last week" and include entity context:

```bash
ads insights --level ad --fields ad_name,spend,roas --preset last_7d --compare previous_period --include status,budget --limit 10
ads insights --accounts act_123,act_456 --fields spend,purchases --preset last_7d --compare previous_year --json
```

Use `--compare previous_period` for the same number of days immediately before the current range, `--compare previous_year` for last year (Feb 29 clamps to Feb 28), or `--compare-range 2026-08-01..2026-08-07` for a custom range. Comparisons require `all_days`, run synchronously, and cannot use `maximum`, `data_maximum`, or non-default action breakdowns. Each numeric metric gets `.previous`, `.change`, and `.change_pct`; missing matches and a zero prior denominator produce null changes or percentages. `meta.compare` records both ranges. `COMPARE_PARTIAL` means the bounded prior read did not cover all rows.

Use `--include status,budget,schedule,setup,targeting,creative` for entity context. Only requested categories are returned in separate `entities.campaigns`, `entities.adsets`, and `entities.ads` maps. Parent budgets use major currency units and retain raw `<field>_minor` strings. Targeting and dynamic creative assets are summaries. Targeting summaries use `geo.countries`, `country_groups` (`{ key, label }`, including European Economic Area (EEA)), region and geo-market names, cities with radius/unit, zip and custom-location samples, and `location_types`. The same fields appear under `geo.excluded`. Lists contain at most 25 items with full counts (`country_count`, `country_group_count`, `region_count`, `city_count`, `geo_market_count`, `zip_count`, `custom_location_count`, `location_type_count`); `truncated: true` on the containing object marks a capped list. Additional geo dimensions are retained. `flexible_spec` preserves each detailed-targeting group with capped interest, behavior and demographic names, per-list counts, `flexible_spec_count` and truncation flags. Custom audience lists are capped and counted too. The table shows a short context column; `--json` preserves all metadata, linked to rows by `meta.entityRefs`. Entity metadata refreshes after 10 minutes independently of performance pages; failed metadata is null with `ENTITY_METADATA_UNAVAILABLE`.

Requested compare fields are retained, including missing native `results` and `cost_per_result`. Friendly formulas are evaluated independently for each period. Native result values match by indicator and attribution window, preserving their expanded column paths. A field without comparable numeric values retains its current value and is listed in `COMPARE_FIELD_UNSUPPORTED`.

The cell estimate uses `min(limit, row upper bound)`: account level without breakdowns or time buckets is one row per account. Time buckets and known bounded breakdowns increase that bound; unknown breakdown sizes and entity levels use the page limit. The actual output still cannot exceed 2,000 cells.

Use `--accounts act_123,act_456` instead of `--account` for 2 to 5 distinct accounts. Each runs sequentially with its own currency and timezone. Results lead with `account_id` and `account_currency`; `meta.accounts` records each account's cache, warnings, paging, and safe errors. Never add totals across accounts. Different currencies produce `MIXED_CURRENCIES`. Some account failures return successful rows with `PARTIAL_RESULTS`; retry failed accounts separately after any stated wait. If every account fails, the request fails, preferring 429 when throttled. Multi-account queries are synchronous, including when `--compare` is used; query heavy accounts separately.

Follow `paging.byAccount` with `--after '{"act_123":"CURSOR","act_456":null}'`, keeping the other options unchanged. Use canonical `act_` keys. Null marks an exhausted account; omitting a key starts that account's first page. `--limit` applies per account, within the shared 2,000-cell result bound. Comparison columns count toward that bound.


Follow only the cursor you need. There is no `--all` mode:

```bash
ads insights --level ad --fields ad_id,ad_name,spend --preset last_30d --limit 100
ads insights --level ad --fields ad_id,ad_name,spend --preset last_30d --limit 100 --after "CURSOR_FROM_PREVIOUS_PAGE"
```

Use the configured default account or pass `--account <id>`. `--json` is lossless and must be used when another tool needs exact IDs, nulls, action keys, columns, or cursors. Reach and frequency are non-additive. Empty rows are a successful response. Zero-impression rows are excluded by default (`excludeZeroImpressions: true`); set `excludeZeroImpressions: false` or use CLI `--include-zero-impressions` to include them. Comparisons keep a row when either period has impressions above zero, so drops to zero remain visible. Attribution follows the account setting unless explicit windows are supplied. Friendly metrics such as `purchases` and `roas` report the one window in `--attribution-windows`; with several windows they are refused (`ALIAS_WITH_ATTRIBUTION_WINDOWS`), so use `--fields actions,action_values --action-types omni_purchase` to compare windows side by side. On 429, report the printed retry wait and stop; do not loop or retry the query.

For single-account queries without comparison, `--mode auto` is the default: daily ranges over 90 and up to 400 days, or Meta "too much data" responses, use async reports. `--mode sync` keeps the 90-day daily cap; `--mode async` always submits a report. The CLI follows pending jobs and writes progress to stderr. Use `--no-wait` to print the job ID and exit, or combine it with `--json` to preserve the pending response. Resume or page the report with:

```bash
ads insights:job ij_<id> --wait 20 --json
ads insights:job ij_<id> --after OPAQUE_CURSOR --limit 25 --json
```

`insights:job` also accepts `--account` and `--format`. It checks once, optionally waiting up to 20 seconds; job references last 24 hours. Follow the job ID instead of repeating the original query.

Results include `meta.cache` with hit, stored time, age in seconds, and lifetime. Cache lifetimes are 10 minutes for ranges including today, 15 minutes for ranges ending 1 to 3 days ago, 1 hour for 4 to 28 days ago, and 24 hours for older ranges. Completed job pages use a 1-hour cache. Table output shows the cache age. `--fresh` skips the existing query cache and refreshes it. A `429 INSIGHTS_ACCOUNT_BUSY` means the account's concurrent request cap is full; respect `Retry-After: 5`. Queries and job reads never retry automatically.

### Actions
| Command | Description |
|---------|-------------|
| `upload <files...>` | Upload images/videos |
| `create spec.json` | Create ads from spec |
| `create:preview spec.json` | Dry run with a Lifetime Budget line and warnings; JSON always includes `warnings` and includes `lifetimeBudget` when applicable |
| `create:interactive` | Guided wizard (accepts all create flags) |
| `duplicator:post-id [specFile]` | Duplicate existing ads selected by Page post ID while preserving post references |
| `duplicator:post-id:preview [specFile]` | Preview a post-ID duplication and its source-to-destination mapping |
| `duplicator:copy:preview [specFile]` | Preview copying campaigns, ad sets or ads by ID (tree, skipped objects, overrides, warnings, call counts) |
| `duplicator:copy [specFile]` | Copy campaigns, ad sets or ads by ID after preview; Agent Launch Approval sends it for approval |
| `duplicator:status <jobId>` | Show a duplication job's progress and its created, failed and skipped objects |

### Jobs
| Command | Description |
|---------|-------------|
| `jobs <jobId>` | Check job status |
| `jobs <jobId> --follow` | Follow progress in real-time |
| `jobs cancel <jobId>` | Cancel a running job |

### Create Flags
| Flag | Description |
|------|-------------|
| `--account <id>` | Override default ad account |
| `--idempotency-key <key>` | Tie retries to one launch for 24 hours (8–128 letters, digits, underscores or hyphens). Sent on `create` only |
| `--allow-repeat` | Create the same ads again instead of returning the existing launch. Use only when the user really wants a repeat |
| `--preset <id>` | API preset ID (alternative to spec file) |
| `--text-preset <id>` | Text preset ID |
| `--copy-from <adId>` | Ad ID to copy settings from |
| `--upload <batchId>` | Upload batch ID |
| `--status <PAUSED\|ACTIVE>` | Set ad status (default: ACTIVE) |
| `--pause-at <level>` | Pause level: `ad` (default), `adSet`, or `campaign` |
| `--ai-disclosure` | Self-disclose AI-generated creative content (Meta AI-content transparency) |
| `--daily-budget <amount>` | Override daily budget (currency units) |
| `--bid-amount <amount>` | Override bid/cost cap (currency units) |
| `--minimum-roas <ratio>` | Override the minimum ROAS goal (e.g. `1.5`) |
| `--campaign-daily-budget <amount>` | Set a CBO daily campaign budget in whole currency units. Mutually exclusive with the lifetime budget; omit both to inherit the source campaign's budget. |
| `--campaign-lifetime-budget <amount>` | Set a CBO lifetime campaign budget in whole currency units. Mutually exclusive with the daily budget; omit both to inherit the source campaign's budget. |
| `--adset-min-spend <amount>` | Set the ad-set minimum spend under CBO in whole currency units; `0` removes the limit inherited from the source ad set. |
| `--adset-max-spend <amount>` | Set the ad-set maximum spend under CBO in whole currency units; `0` removes the limit inherited from the source ad set. |
| `--adset-min-spend-pct <5-100>` | Set the ad-set minimum spend as a percentage of the campaign budget, in 5% steps. Also works with an existing CBO campaign; mutually exclusive with `--adset-min-spend`. |
| `--adset-max-spend-pct <5-100>` | Set the ad-set maximum spend as a percentage of the campaign budget, in 5% steps. Also works with an existing CBO campaign; mutually exclusive with `--adset-max-spend`. |
| `--page <id>` | Override the Facebook Page ID used by created ads |
| `--instagram <id>` | Override the Instagram account ID used by created ads |
| `--use-page-identity` | Use the Facebook Page as the Instagram identity; cannot be combined with `--instagram` |
| `--threads <id>` | Override the Threads profile ID used by created ads |
| `--text-file <path>` | Load text config from a JSON file |
| `--expanded` | Show full headline, primary text, and description values in previews |

### Ad Set Targeting Flags
Set the audience for every new ad set in the launch. These only apply when the build creates new ad sets.

| Flag | Description |
|------|-------------|
| `--location <ISO>` | Target country by ISO code. Repeatable: `--location GB --location DE` |
| `--age-min <n>` | Minimum age, 13-65 |
| `--age-max <n>` | Maximum age, 13-65 (65 means 65+) |
| `--gender <gender>` | `all`, `men`, or `women`. `all` clears a gender restriction inherited from the source ad set |

These flags **merge into** a saved build's stored targeting rather than replacing it, so `--build <id> --gender all` changes only the gender and leaves the build's countries, ages and detailed targeting intact.

Cities, zip / post codes, and detailed targeting (interests, behaviors, employers, job titles, industries) have no create flags. Find their IDs with `ads targeting:search`, then set them through a spec file's `targeting` object. See Spec File Format below.

Two Meta constraints apply when the source ad set uses Advantage+ audience (`advantage_audience: 1`):

- The minimum-age control is capped at 25. A higher `--age-min` is kept as a delivery suggestion instead, so Meta may serve from 25.
- Gender and maximum age are suggestions Meta can exceed. Locations and the minimum age are always respected.

### Copy by ID

`duplicator:copy` is the CLI version of the MCP `ads_duplicate` tool: the same as Ads Manager's Duplicate. Pick sources with `--campaign`, `--adset` or `--ad <id>` (repeatable, one level per request) or a spec file shaped like the v1 `/duplications/copies` body (`level`, `ids`, `deepCopy`, `destination`, `rename`, `overrides`, ...). Add `--rename <pattern>` to name the copies with `{name}`, `{date}` (UTC) and `{index}`. Copies include child objects by default; `--deep-copy` asks for that explicitly and `--no-deep-copy` copies only the selected roots. Always run `duplicator:copy:preview` first. Without a `sourceFingerprint`, `duplicator:copy` only previews (`status: "requires_preview"`, exit 1) and nothing is copied; save `resolvedRequest` from `--json` as the spec and run it again. Pass `--confirm-spend` and `--acknowledge-warnings` only after the user approves. With Agent Launch set to Approval it prints the approval link and copies nothing until approved. Otherwise it polls the job for up to 30 minutes; if it is still running, resume with `ads duplicator:status <jobId>`.

### Post-ID Duplication Flags

Duplications follow the account’s Agent Launch setting on the Account Permissions page, the same as launches; Agent Edit does not apply. Allowed (the default) starts the copy. Approval returns pending approval (`status: "pending_approval"`) and `reviewUrl`. The CLI prints “Sent for approval. Nothing has been duplicated yet. Approve it here: <reviewUrl>” and exits successfully without polling; `--json` prints the response unchanged. Share the link, let the user approve in your Agent Inbox, and do not retry. There is no job until approval. Off refuses with `AGENT_LAUNCH_DISABLED`. For `duplicator:copy`, use `--confirm-spend` (`confirmSpend: true`) only after explicit approval for ACTIVE copies when the account approval rule is not `none`; `duplicator:post-id` never requires it. Admins can put `testMode: true` in the spec; simulation survives approval without live Meta writes. Uploads (`create`/upload commands) and the human-operated web Duplicator are unaffected. Each post-ID copy can be approved or rejected on its own; `ads_get_edit_review` / `ads edit:review` report `copies[].status` (`pending`, `approved`, `made`, `failed`, `rejected`). For post-ID requests, `group` links proposals split from one request, and `summary.copies` totals copies across the whole review.

These apply to `duplicator:post-id` and `duplicator:post-id:preview`. This mode selects existing ads by Page post ID and preserves their post references (likes, comments, shares), so it is separate from `create --copy-from`, which builds new ads from copied settings. Other duplication modes may be added later.

| Flag | Description |
|------|-------------|
| `--account <id>` | Override default ad account |
| `--post <postId>` | Select source ads by Page post ID (discovery). Repeatable. Searches up to ~2,000 recent account ads; if a post matches more than one ad, the preview returns candidates and a live run requires an explicit `--ad` selection. |
| `--ad <adId>` | Select source ads directly and exactly by ad ID (first-class; pins a specific ad and resolves post ambiguity). Repeatable. |
| `--campaign <id>` | Existing destination campaign |
| `--adset <id>` | Existing destination ad set, or the template when a new-ad-set flag is present (always required) |
| `--new-adset [name]` | Create ONE new ad set for all source ads (default name `{AdName}`; becomes `Multiple Ads` with several sources) |
| `--new-adset-per-ad [name]` | Create one new ad set per source ad (`{AdName}` and `{Index}` supported) |
| `--ad-name <pattern>` | Duplicated ad name pattern (default `{AdName}`; `{Index}` supported) |
| `--paused` | Create the duplicated ads paused |
| `--confirm-spend` | Accepted for older scripts; post-ID copies never require it |
| `--use-creative-id` | Reuse original creative IDs instead of post-reference creatives |
| `--json` | Output raw JSON for scripting |

A post-ID duplication spec file is the v1 request body (`postIds` for discovery or ordered `adIds` for exact selection, an existing `campaignId`, `adSetId` / `newAdSet` / `newAdSetPerAd`, `adNamePattern`, `paused`, `useCreativeId`, `confirmSpend`, admin-only `testMode`); flags override it field by field. New campaigns are not supported: the destination is always an existing campaign. New ad sets clone every setting (targeting, budget, schedule, bid) from the template ad set given by `--adset`, which is required with `--new-adset` and `--new-adset-per-ad`. The source ad's own ad set is never the template. The template ad set may be empty: it needs no ads. For a brand-new empty campaign, first put one ad set in it with no ads (for example `duplicator:copy` of an ad set with `--no-deep-copy`), then run the post-ID duplication with `--adset <that ad set>` and `--new-adset` or `--new-adset-per-ad` for each group, so the ads and their new ad sets are created in the post-ID step. Do not duplicate the ads into the template ad set itself. Always run `duplicator:post-id:preview` first; if a post is ambiguous the preview returns candidates and `requiresSourceSelection`, and a live run needs the resolved `adIds`. Dynamic-creative source ads reuse their creative ID and cannot preserve engagement; the preview lists them as warnings.

### Detail Flags
| Flag | Description |
|------|-------------|
| `ad --expanded` | Show full headline, primary text, and description values |

### Common Flags
| Flag | Description |
|------|-------------|
| `--account <id>` | Override default ad account |
| `--json` | Raw JSON output (available on most commands) |

### Use `--json` deliberately

For `insights`, use `--json` when exact structured data is required. It preserves IDs, nulls, action keys, columns, and cursors. For interactive creation commands, prefer the human-readable output because `create --json` replaces the live progress log with raw NDJSON poll dumps.

---

## Spec File Format

The JSON spec controls ad creation. Provide a template source plus either `uploadId` or captured `mediaItems` containing `mediaHash` for every image and `mediaId` for every video. Standard, carousel, flexible, and placement videos use `thumbnailHash`; Multi Media videos use a public thumbnail URL. Missing library-video thumbnails are recovered from Meta during preview/create without re-uploading the video. Saved builds created in the web uploader carry these values automatically when available.

### Minimal Spec

```json
{
  "adPresetId": "preset_id_here",
  "campaign": { "name": "New Campaign" },
  "uploadId": "batch_abc123"
}
```

### Template Source (pick one)

| Field | Description |
|-------|-------------|
| `adPresetId` | Saved API preset settings; requires an explicit campaign/ad-set destination |
| `copyFromAd` | Facebook ad ID to copy settings from |

Presets reuse their saved campaign and ad set when no destination is supplied, including preset-only creates and older saved builds. Preview and create return a non-fatal `PRESET_DESTINATION_USED` warning identifying the saved destination. To choose a different destination, pass `campaign.name` for a new campaign, named `campaign.campaigns` entries in duplicate/split mode, or `campaign.id` / `adSet.id` for an existing destination. With only an explicit `campaign.id`, new ad sets are created. With `copyFromAd` and no `campaign` or `adSet`, a new paused campaign and ad set are created from the template ad's settings; the template's own campaign and ad set are never used unless you pass their IDs.

Cross-campaign create/preview is available only for direct agent requests. Saved builds and Agent Inbox launches must use the template ad's campaign or a new campaign; to save a build for another existing campaign, copy from an ad in that campaign (campaign.id + adSet.id picks one). Existing stored cross-campaign builds can still launch, but cannot be re-saved.

When using `copyFromAd`, provide an upload batch (or launch a saved build with resolvable captured `mediaItems`) and optionally campaign/ad set:
```json
{
  "copyFromAd": "120233848667930472",
  "uploadId": "batch_abc123",
  "campaign": { "id": "120233848666410472" },
  "adSet": { "id": "120233848666620472" }
}
```

### Profile Options

By default, created ads inherit the Facebook Page, Instagram account, and Threads profile from the template ad or preset. Override them with `profile`:

```json
{
  "copyFromAd": "120233848667930472",
  "uploadId": "batch_abc123",
  "profile": {
    "pageId": "123456789012345",
    "instagramId": "17841400000000000",
    "threadsId": "987654321098765"
  }
}
```

You can also use flags:

```bash
ads create:preview spec.json --page 123456789012345 --instagram 17841400000000000 --threads 987654321098765
```

Use `--use-page-identity` (or `"useFacebookPage": true` in `profile`) when a Page has no linked Instagram account and should act as the Instagram identity. It cannot be combined with `--instagram` / `instagramId`.

If you override only `pageId`, the CLI/API automatically uses that Page's linked Instagram account, or the Facebook Page identity when no account is linked. Threads still resets because it may belong to the old Page; add `threadsId` explicitly when needed. Run `ads pages` to list available Facebook Page IDs and linked Instagram accounts. For multi-campaign specs, use `profile.campaigns` keyed by campaign ID (or an unambiguous campaign name) to assign Page, Instagram, and Threads identities independently; each entry also accepts `useFacebookPage: true`.

### Campaign Structure

**Single campaign** (default): ads go into the template ad's campaign, or a new campaign if `campaign.name` is provided.

**Multi-campaign modes** — use `campaign.mode` with a `campaigns` array:

```json
{
  "campaign": {
    "mode": "duplicate",
    "campaigns": [
      { "name": "Campaign A" },
      { "name": "Campaign B" }
    ]
  }
}
```

| Mode | Behavior |
|------|----------|
| `"single"` | Default — one campaign |
| `"duplicate"` | All media duplicated into each campaign |
| `"split"` | Media split across campaigns |

### Ad Set Modes

**Single ad set** (default):
```json
{ "adSet": { "name": "My Ad Set" } }
```

**Existing ad set:**
```json
{ "adSet": { "id": "120233848666620472" } }
```

**Per upload** (one ad set per file):
```json
{ "adSet": { "mode": "perUpload" } }
```

**Auto group** (split evenly):
```json
{ "adSet": { "mode": "autoGroup", "adsPerAdSet": 5 } }
```

**Custom groups:**
```json
{
  "adSet": {
    "groups": [
      { "name": "Images", "media": ["hero.jpg", "banner.jpg"] },
      { "name": "Video", "media": ["promo.mp4"] }
    ]
  }
}
```

**Ad set naming pattern** (for multi-ad-set modes):
```json
{ "adSet": { "mode": "perUpload", "namePattern": "Ad Set {index:01}" } }
```

**Variant grouping** (group ads by variation identifier into the same ad set):
```json
{ "adSet": { "mode": "autoGroup", "groupVariations": true, "variationIdentifier": "-" } }
```

### Budget / Bid-Control Override

`dailyBudget` and `bidAmount` use account currency units (e.g. `50` = $50). `minimumRoas` is a ratio (e.g. `1.5` = 1.5x).

- **ABO** (budget on the ad set): set `dailyBudget` with `bidAmount` for bid/cost-cap strategies, or with `minimumRoas` for `LOWEST_COST_WITH_MIN_ROAS`.
- **CBO** (budget on the campaign): don't set `dailyBudget` on the ad set. Set `campaign.dailyBudget` or `campaign.lifetimeBudget` (mutually exclusive) to override the source campaign's budget, or omit both to inherit it. Set `bidAmount` or `minimumRoas` on the ad set when its source strategy uses that control.

`bidAmount` and `minimumRoas` are mutually exclusive because they belong to different bid strategies.

Under CBO, `adSet.minSpend` and `adSet.maxSpend` set limits in whole currency units. Positive amounts require a campaign budget in the same request; `0` removes the corresponding limit inherited from the source ad set. Use `adSet.minSpendPercentage` and `adSet.maxSpendPercentage` for limits from 5-100% of the campaign budget (5% steps, matching Ads Manager), including when targeting an existing CBO campaign. Each percentage field is mutually exclusive with its absolute counterpart.

```json
{ "adSet": { "dailyBudget": 50 } }
{ "adSet": { "bidAmount": 5 } }
{ "adSet": { "minimumRoas": 1.5 } }
{ "adSet": { "dailyBudget": 50, "bidAmount": 5 } }
{ "campaign": { "dailyBudget": 100 }, "adSet": { "minSpend": 20, "maxSpend": 80 } }
{ "campaign": { "id": "123" }, "adSet": { "minSpendPercentage": 20, "maxSpendPercentage": 80 } }
```

Also available as CLI flags: `--daily-budget 50`, `--bid-amount 5`, `--minimum-roas 1.5`, `--campaign-daily-budget 100`, `--campaign-lifetime-budget 1000`, `--adset-min-spend 20`, `--adset-max-spend 80`, `--adset-min-spend-pct 20`, and `--adset-max-spend-pct 80`.

### Ad Set Targeting

The top-level audience is the build default for every new ad set in the launch. Omit `targeting` entirely to inherit the source ad set's audience unchanged.

```json
{
  "targeting": {
    "selectionVersion": 4,
    "countries": ["US", "CA"],
    "cities": [{
      "key": "2525495",
      "name": "Austin",
      "region": "Texas",
      "countryCode": "US",
      "radius": 25,
      "distance_unit": "mile"
    }],
    "zips": [{
      "key": "US:90210"
    }],
    "ageSelected": true,
    "ageMin": 27,
    "ageMax": 52,
    "genderSelected": true,
    "genders": "women",
    "detailedTargetingSelected": true,
    "detailedTargetingGroups": [[
      { "id": "6003700363290", "name": "Performance-based advertising", "type": "interests" },
      { "id": "112002898811624", "name": "Advertising agency", "type": "work_employers" }
    ]]
  }
}
```

Any ad set mode can override that default with `adSet.targetingPerAdSet`. Key it by the final planned ad set name or ID, or use `campaign::key` when campaigns can contain the same ad set name. Custom specs may also use `adSet.groups[].targeting`. This is spec-file-only: there are no per-ad-set flags because flags cannot identify a planned ad set. Omitted fields inherit in this order: `targetingPerAdSet`, then group targeting, then top-level build targeting, then the source ad set.

```json
{
  "targeting": { "countries": ["US"], "ageMin": 21, "ageMax": 55 },
  "adSet": {
    "mode": "perUpload",
    "namePattern": "Ad Set {index:01}",
    "targetingPerAdSet": {
      "Ad Set 02": { "countries": ["CA"], "genders": "women" }
    }
  }
}
```

In custom mode, `groups[].targeting` remains supported when keeping the override beside the group's media is more convenient. A matching `targetingPerAdSet` entry wins over it.

Rules that matter:

- **An effective location is required.** At least one country, city, or zip / post code. A group may omit location when it inherits one from the build default or source ad set.
- **Cities and zip / post codes are added to countries, not subtracted from them.** `["US"]` plus Austin or `US:90210` reaches all of the US, because Meta unions locations. To target only a city or zip, send no countries.
- **Age and gender are opt-in.** Set `ageSelected` / `genderSelected` to `true`, or the values are ignored. `ageMin` must be less than or equal to `ageMax`.
- **Detailed targeting replaces, it does not merge.** `detailedTargetingGroups` becomes the complete audience, so any type you omit is dropped. `detailedTargetingSelected: true` with an empty array clears the source's detailed targeting.
- **`type` is the Meta `flexible_spec` key**: `interests`, `behaviors`, `work_employers`, `work_positions`, `industries`, `life_events`, and so on.
- One group only for now. The outer array exists so AND-ed groups can be added later.

City radius must be 10-50 miles or 17-80 kilometers. Zip and post code entries target the exact area and do not accept a radius or `distance_unit`.

**Finding city keys, zip keys, and detailed-targeting IDs.** Search through Ads Uploader so the server can use the connected Meta account without exposing its Facebook token:

```bash
ads targeting:search "Austin" --type city
ads targeting:search "90210" --type zip
ads targeting:search "advertising" --type detailed
```

Use `--account act_123` when no default account is configured, `--limit 25` to request more matches, and `--json` when you want pasteable structured output. City and zip results provide the `key` required by `targeting.cities[]` and `targeting.zips[]`. Detailed results provide the `id`, `name`, and `type` required by a `detailedTargetingGroups` entry, plus display-only audience bounds and category path.

Agents using the MCP surface should call `ads_search_targeting` for the same lookup. IDs, city keys, and zip keys must never be guessed.

### Text Configuration

To remove a copied headline, primary text or description, send an empty list (`[]`) for that field. Omitting the field, or `[""]`, keeps the copied text. This applies to `texts.common`, `texts.perAd`, `texts.perAdset` and `texts.perAdSet`.

**Common text** (same for all ads):
```json
{
  "texts": {
    "common": {
      "headlines": ["Headline 1", "Headline 2"],
      "bodies": ["Primary text"],
      "descriptions": ["Description"]
    },
    "strategy": "flexible"
  }
}
```

**Per-ad text** (unique per file):
```json
{
  "texts": {
    "perAd": {
      "hero.jpg": {
        "headlines": ["Hero Headline"],
        "bodies": ["Hero copy"],
        "descriptions": ["Hero desc"],
        "cta": "LEARN_MORE",
        "link": "https://example.com/hero",
        "urlTags": "utm_content=hero"
      },
      "banner.jpg": {
        "headlines": ["Banner Headline"],
        "bodies": ["Banner copy"]
      }
    }
  }
}
```

Per-ad keys are **filenames** (not full paths). Unspecified fields inherit from the template ad.

**Per-ad-set text** (unique per ad set) is also supported via `texts.perAdset`, keyed by ad-set name/ID:
```json
{
  "texts": {
    "mode": "perAdset",
    "perAdset": {
      "Winter Sale": { "headlines": ["Winter Headline"], "bodies": ["Winter copy"] },
      "Spring Sale": { "headlines": ["Spring Headline"], "bodies": ["Spring copy"] }
    }
  }
}
```

> **Prefer `perAd` for one-ad-per-ad-set builds.** When each ad set holds exactly one ad (a very common
> shape), use `texts.perAd` keyed by **filename** — it is the robust shape and always matches. Reach for
> `texts.perAdset` only when an ad set genuinely holds multiple ads that must share copy. If you do use
> `texts.perAdset`, **each key must exactly equal the final planned ad-set name or ID.** Ad-set names are
> often generated at create time (e.g. `Ad Set 01`, `Ad Set 02` from a name pattern), so a descriptive key
> that does not match the planned name will not resolve. Per-ad-set mode remains fully supported — this is
> a preference for reliability, not a restriction.

**Text preset:**
```json
{ "textPresetId": "preset_id_here" }
```

Cannot combine `textPresetId` with `texts`.

**Strategy options:**
- `"flexible"` (default) — Meta optimizes across text variations (multiple headlines/bodies become options Facebook mixes)
- `"separate"` — each text combination becomes a separate ad

Per-ad entries support: `headlines`, `bodies`, `descriptions`, `cta`, `link`, `displayUrl`, `urlTags`. Unspecified fields inherit from the template ad.

### CTA and Links

Top-level CTA applies to all ads (overridden by per-ad CTA):
```json
{
  "cta": {
    "type": "SHOP_NOW",
    "link": "https://example.com",
    "displayUrl": "example.com"
  },
  "urlTags": "utm_source=facebook&utm_medium=paid"
}
```

### URL Split-Test (LP A/B)

Provide 2–5 destination URLs under `texts.urlVariants` and every generated ad set is duplicated once per URL so Meta optimizes each ad+LP combination independently (Barry Hott's method — don't split traffic inside one ad).

```json
{
  "texts": {
    "common": { "headlines": ["Hero"], "bodies": ["Copy"] },
    "urlVariants": [
      { "link": "https://example.com/homepage", "label": "homepage" },
      { "link": "https://example.com/quiz", "label": "quiz-v2" }
    ]
  }
}
```

- `label` is optional; when omitted, the last URL path slug is used (`/quiz-v2` → `quiz-v2`), falling back to hostname for root URLs.
- Naming depends on ad set mode:
  - `single`: each variant's label becomes the full ad set name.
  - `perUpload` / `autoGroup`: each duplicated ad set name is `{base}_{label}` (as in the uploader), or substitutes a `{destination}` token if present in the pattern.
  - Explicit `adSet.groups[]`: use `{group}` (aliases: `{name}`, `{base}`, `{adset}`) in `adSet.namePattern` to reference `groups[i].name`.
  - Variation grouping: use `{variation}` in `adSet.namePattern` to reference the filename prefix before `variationIdentifier`.
- The `{date}` token resolves to today's date in labels (`launch-{date}` → `launch-2026-04-27`).
- Each destination URL must be distinct. Trailing-slash and case variants of the same URL collapse to one — supply at least 2 truly different destinations.
- `texts.perAdset` text applies to every copy of an ad set: key it by the base ad set name (`Ad Set 01`), never a copy's name (`Ad Set 01_quiz-v2`).
- Your ad set budget is multiplied by the number of variants.
- Not compatible with per-ad `link` overrides — the variant URL wins when both are set.
- Rejected with a 400 ConfigError if the source ad (preset or `copyFromAd`) targets a special destination (lead form, Messenger, WhatsApp, Instagram DM, Call) — those formats don't route via `cta.link`.

**Standard CTA types:** `LEARN_MORE`, `SHOP_NOW`, `SIGN_UP`, `SUBSCRIBE`, `GET_OFFER`, `CONTACT_US`, `DOWNLOAD`, `ORDER_NOW`, `BUY_NOW`, `BOOK_NOW`, `APPLY_NOW`, `GET_QUOTE`, `GET_IN_TOUCH`, `WATCH_MORE`

**Objective-specific CTAs** (inherited from template ad — do not set manually):

| CTA | Required Objective |
|-----|-------------------|
| `MESSAGE_PAGE` | Messenger destination |
| `WHATSAPP_MESSAGE` | WhatsApp destination |
| `INSTAGRAM_MESSAGE` | Instagram DM destination |
| `CALL_NOW` | Call campaign |

### Creative Enhancements

```json
{ "creativeEnhancements": "none" }
```

| Value | Effect |
|-------|--------|
| `"metaDefaults"` | Let Meta decide (default if omitted) |
| `"all"` | All features on |
| `"none"` | All enhancements off; source formats inherited |
| `["text_translation", "image_touchups"]` | Listed enhancements on, rest off; source formats inherited |
| `{ "useMetaDefaults": false, "features": [], "excludedFormats": ["format_carousel"] }` | All enhancements off; explicitly disable the source carousel format |

Available features: `text_translation`, `inline_comment`, `enhance_cta`, `text_optimizations`, `reveal_details_over_time`, `show_destination_blurbs`, `image_brightness_and_contrast`, `image_touchups`, `video_auto_crop`, `video_filtering`, `image_animation`, `image_templates`, `adapt_to_placement`, `product_extensions`, `description_automation`, `add_text_overlay`, `music`, `carousel_to_video`, `carousel_dynamic_description`, `multi_share_end_card`, `multi_share_optimized`

`show_destination_blurbs` is displayed as **Show spotlights**.

Source format display options inherit even with a feature list, `"none"`, or `{ "useMetaDefaults": false }`. To turn one or both off, set `creativeEnhancements.excludedFormats` to an array of `"format_carousel"` (Show as carousel) and/or `"format_collection"` (Show as collection). Omit it to inherit. A per-feature object such as `{ "format_carousel": false }` also works. Excluded formats must be cloneable on the source (catalog-backed formats require a catalog), cannot also be enabled in `features` or the per-feature object, and are unsupported for existing-post creatives. Non-arrays and unknown IDs are rejected. Formats cannot be enabled if the source lacks them.

When cherry-picking features, only list ones relevant to the media type. `video_auto_crop` and `video_filtering` only apply to video ads. `carousel_to_video`, `carousel_dynamic_description`, `multi_share_end_card`, and `multi_share_optimized` only apply to carousel ads.

### AI Transparency

Self-disclose that an ad's creative was made or significantly edited with AI (Meta's AI-content self-disclosure). Left out, the copied ad's own disclosure carries over, as in the Uploader; set `false` to switch it off.

```json
{ "aiDisclosure": true }
```

Or via flag: `--ai-disclosure`. Applies to every ad in the upload. When on, each created ad opts in to Meta's AI-content self-disclosure; when off, no disclosure is sent.

### Carousel Ads

Group uploaded files into carousel ads with per-card text and optional overall carousel text. `cardTexts` controls individual cards. Overall carousel text can be colocated on the carousel object or set in `texts.perAd` using the carousel `name`; colocated fields win when both are present.

A carousel takes one headline and one description. Set per-card text with `cards[].cardText` or `cardTexts`; several primary texts (`bodies`) are allowed. Multiple own headlines or descriptions warn at preview/save and are refused at create. Inherited variants warn and use only the first on carousel cards. When every card has a size pair (for example `_4x5` + `_9x16`), the carousel launches as a placement-variant carousel, as it does in the uploader, and uses only its first primary text; preview warns with `CAROUSEL_PRIMARY_TEXT_LIMITED`.

```json
{
  "carousel": [
    {
      "name": "My Carousel",
      "cards": ["slide1.jpg", "slide2.jpg", "slide3.jpg"],
      "headlines": ["Overall Carousel Headline"],
      "bodies": ["Overall primary text"],
      "descriptions": ["Overall description"],
      "cta": "SHOP_NOW",
      "link": "https://example.com/carousel",
      "urlTags": "utm_content=my_carousel",
      "cardTexts": [
        { "headline": "Slide 1", "description": "First card", "link": "https://example.com/1" },
        { "headline": "Slide 2", "description": "Second card", "link": "https://example.com/2" }
      ]
    }
  ]
}
```

Alternative overall-text shape:

```json
{
  "texts": {
    "perAd": {
      "My Carousel": {
        "headlines": ["Overall Carousel Headline"],
        "bodies": ["Overall primary text"],
        "descriptions": ["Overall description"],
        "cta": "SHOP_NOW",
        "link": "https://example.com/carousel",
        "urlTags": "utm_content=my_carousel"
      }
    }
  },
  "carousel": [
    {
      "name": "My Carousel",
      "cards": ["slide1.jpg", "slide2.jpg", "slide3.jpg"]
    }
  ]
}
```

Cards must reference filenames from the upload batch or captured saved-build media. Minimum 2 cards per carousel. Files claimed by a carousel are removed from the standard ad list.

### Flexible Ads

Group multiple assets into a single flexible ad (Meta picks the best asset per placement):

```json
{
  "flexible": [
    {
      "name": "Multi-Asset Ad",
      "assets": ["hero.jpg", "promo.mp4", "banner.jpg"]
    }
  ]
}
```

Minimum 2 assets per group. Files claimed by a flexible group are removed from the standard ad list.

Meta doesn't offer flexible ads in Leads campaigns. A flexible group copied from an ad in a Leads campaign is refused with `FLEXIBLE_OBJECTIVE_UNSUPPORTED` at preview, save and create, before anything is built. Use standard or Multi Media ads there, or copy from an ad in a Sales campaign.

### Multi Media Ads

Group 2-10 uploaded images or videos into a Meta Multi Media ad:

```json
{
  "multimedia": [
    {
      "name": "Mixed Media Ad",
      "assets": ["hero.jpg", "promo.mp4", "banner.jpg"],
      "assetTexts": [
        {
          "headline": "Hero headline",
          "primaryText": "Hero primary text",
          "description": "Hero description",
          "link": "https://example.com/hero",
          "displayUrl": "example.com/hero"
        }
      ]
    }
  ]
}
```

Assets must reference filenames from the upload batch or captured saved-build media. `assetTexts` is optional and lines up with `assets` by index; each field is a single override for that asset and blank fields fall back to the ad's main text and URLs. The ad's main text and URL apply to every asset, and an asset's override, the first asset's included, changes that asset only. Videos use a cached public thumbnail URL, recovered from Meta when missing. Files claimed by a Multi Media group are removed from the standard ad list.

Each aspect-ratio size is sent as its own entry, and every placement goes to the size the single-image and video placement rules pick, including the account's saved placement priorities (for example 4x5 for Feeds, 9x16 for Stories, Reels and Audience Network, and 1x1 for Marketplace, Search and Right Column). Sizes do not count toward the limit of 2-10 visuals.

### Ad Naming

```json
{ "adNamePattern": "{filename} - {date}" }
```

Placeholders: `{filename}`, `{index:01}`, `{variation}`, `{group}`, `{adset}`, `{campaign}`, `{date}`, `{date:short}`, `{timestamp}`

Transforms (wrap a value; an empty value defaults to the filename): `{clean:...}` strips a trailing aspect-ratio suffix (`_9x16`, `-1x1`, `_4x5`, `_16x9`, `_1.91x1`, `_vertical`, `_horizontal`), `{lowercase:...}`, `{uppercase:...}`, `{titlecase:...}`, `{split:DELIM:POS}` extracts the POS-th part (1-based). Example: `{clean:{filename}}` turns `Hook A_9x16.mp4` into `Hook A` - use it when single-file uploads keep their ratio suffix (multi-ratio variant groups strip it automatically).

### Options

```json
{
  "options": {
    "status": "PAUSED",
    "pauseAt": "ad",
    "schedule": {
      "startTime": "2026-04-01T09:00:00",
      "endTime": "2026-04-30T23:59:59"
    }
  }
}
```

| Field | Values | Description |
|-------|--------|-------------|
| `status` | `"PAUSED"`, `"ACTIVE"` | Ad launch status (default: ACTIVE) |
| `pauseAt` | `"ad"`, `"adSet"`, `"campaign"` | Which level to pause (default: ad) |
| `schedule.startTime` | ISO 8601 string | Account-local time YYYY-MM-DDTHH:mm[:ss], or ISO 8601 with an offset or Z (converted to the account time zone). Any minute; flag: `--start-time <iso>`. |
| `schedule.endTime` | ISO 8601 string | Account-local time YYYY-MM-DDTHH:mm[:ss], or ISO 8601 with an offset or Z (converted to the account time zone). Any minute; flag: `--end-time <iso>`. Lifetime budgets need an end at least 24 hours after the start. |

Create rejects new campaigns/ad sets with HTTP 400 for these codes (preview marks them `blocking: true`): `LIFETIME_BUDGET_END_MISSING`, `LIFETIME_BUDGET_END_PASSED`, `LIFETIME_BUDGET_END_BEFORE_START`, `LIFETIME_BUDGET_END_TOO_SOON` (set `options.schedule.endTime` or `--end-time` at least 24 hours after the start, or use a daily budget). `EXISTING_AD_SET_ENDED` and `EXISTING_AD_SET_ENDS_SOON` stay non-blocking warnings; extend the existing ad set in Ads Manager.

---

## Upload & Variant Detection

Variant groups are detected automatically from filename conventions:

**Ratio suffixes:**
`hero_4x5.jpg` + `hero_9x16.jpg` + `hero_16x9.jpg` → grouped as one variant ad

**Word suffixes:**
`hero.jpg` + `hero_vertical.jpg` + `hero_horizontal.jpg` → grouped as one variant ad

Both `_` and `-` delimiters work. A file without a ratio suffix (e.g. `hero.jpg`) only groups with `_vertical`/`_horizontal` variants.

---

## Common Patterns

### Simple upload + create with preset

```bash
# Upload
ads upload ./creatives/hero.jpg ./creatives/banner.jpg
# Note the batchId from output (printed as "Batch: batch_xxx")

# Write spec
cat > /tmp/spec.json << 'EOF'
{
  "adPresetId": "PRESET_ID",
  "campaign": { "name": "New Campaign" },
  "uploadId": "BATCH_ID"
}
EOF

# Preview, then create
ads create:preview /tmp/spec.json
ads create /tmp/spec.json
```

### Per-ad text (unique copy per file)

```json
{
  "adPresetId": "PRESET_ID",
  "campaign": { "name": "New Campaign" },
  "uploadId": "BATCH_ID",
  "texts": {
    "perAd": {
      "hero.jpg": {
        "headlines": ["Summer Sale Now On"],
        "bodies": ["Save up to 50% on all items"],
        "cta": "SHOP_NOW",
        "link": "https://example.com/summer"
      },
      "banner.jpg": {
        "headlines": ["New Collection Available"],
        "bodies": ["Browse our latest styles"],
        "cta": "LEARN_MORE",
        "link": "https://example.com/new"
      }
    }
  },
  "options": { "status": "PAUSED" }
}
```

### Auto-group into multiple ad sets

```json
{
  "adPresetId": "PRESET_ID",
  "campaign": { "name": "New Campaign" },
  "uploadId": "BATCH_ID",
  "adSet": { "mode": "autoGroup", "adsPerAdSet": 3 },
  "options": { "status": "PAUSED" }
}
```

### Copy from existing ad

```bash
# Find the ad to copy from
ads campaigns
ads campaign 120233848666410472       # shows ad sets
ads adset 120233848666620472          # shows ads
ads ad 120233848667930472             # shows ad details
```

```json
{
  "copyFromAd": "120233848667930472",
  "campaign": { "id": "120233848666410472" },
  "uploadId": "BATCH_ID",
  "options": { "status": "PAUSED" }
}
```

### Follow job progress

```bash
ads create spec.json
# Shows live progress automatically; to check a job later:
ads jobs JOB_ID --follow
```

`ads create` and `ads jobs --follow` watch for up to 30 minutes. On very large batches (hundreds of creatives), the command may exit before the job finishes with:

```
−  Still running after 30 min — job continues on the server.
   Resume with: ads jobs JOB_ID --follow
```

This is **not** a failure. The backend job keeps creating ads. Re-run the suggested command to resume watching. Exit code is `2` for this case (vs `0` for success, `1` for real errors) — shell scripts and agents should branch on the exit code to distinguish.

---

## Critical Gotchas

1. **Always preview first**: `create:preview` catches config errors before creating Meta ads
2. **Ads are ACTIVE by default** — use `--status PAUSED` or `"status": "PAUSED"` in spec to create them paused
3. **`uploadId` comes from upload output** — it's the batch ID returned by `ads upload`
4. **Uploads are tied to an ad account** — files are uploaded directly to the selected account's Facebook media library. The batch ID can only be used with the same account
5. **`copyFromAd` needs resolvable media** — provide `uploadId`, or launch a saved build whose images carry `mediaHash` and videos carry `mediaId`. Optionally provide `campaign.id` and `adSet.id` to control placement
6. **Per-ad text keys are filenames** — use `"hero.jpg"`, not `"/path/to/hero.jpg"`
7. **`textPresetId` and `texts` are mutually exclusive** — use one or the other
8. **Objective-specific CTAs** (`MESSAGE_PAGE`, `WHATSAPP_MESSAGE`, etc.) are inherited from the template — don't set them manually
9. **Use `--json` only when exact structured data is needed.** It is the lossless mode for Insights. Prefer human-readable output for interactive creation commands.
10. **Never wrap CLI commands in polling loops** (`watch`, `while true; do ads adset X; sleep 5; done`, or equivalent). If you need to follow a job, use `ads jobs JOB_ID --follow`. If you need to verify ads landed after a create, run `ads adset <id>` **once** after the create returns — the job engine only marks complete when every ad is created on Facebook. Polling loops trigger rate limits (per-user-per-operation; reporting/resource reads share 20 requests/second and 600/minute per operation; other list reads retain 20/minute) and return `429 Rate Limit Exceeded` with a `Retry-After` header.
11. **`ads create` exit code 2 means "still running," not "failed."** On very large batches the CLI stops watching after 30 minutes while the backend job continues. Resume with `ads jobs JOB_ID --follow`. Treat exit code 2 as "come back later," exit code 1 as a real error.

## Partnership ads with your own media

The CLI and MCP can launch uploaded-media partnership ads on Facebook and Instagram. Use your normal image/video media, carousel, flexible, Multi Media, or placement-variant spec and add `profile.partnership.enabled: true`. Uploaded media follows ordinary assembly with a Second Identity. Omit `uploaderMode` or set it to `false`; `true` selects imported posts and cannot be used with uploads.

Choose your First Identity with `profile.pageId` and `profile.instagramId`. The shared partner is the Second Identity:

```json
{
  "accountId": "act_123",
  "copyFromAd": "SOURCE_AD_ID",
  "adSet": { "id": "EXISTING_AD_SET_ID" },
  "mediaItems": [
    { "mediaName": "one.jpg", "mediaType": "image", "mediaHash": "UPLOADED_IMAGE_HASH_ONE" },
    { "mediaName": "two.jpg", "mediaType": "image", "mediaHash": "UPLOADED_IMAGE_HASH_TWO" }
  ],
  "profile": {
    "pageId": "111",
    "instagramId": "222",
    "partnership": {
      "enabled": true,
      "sponsorPageId": "333",
      "sponsorInstagramId": "444",
      "displayMode": "both"
    }
  },
  "options": { "status": "PAUSED" }
}
```

For different partners per ad, add this `texts` block to the same spec. The second row explicitly uses **No Partner**:

```json
{
  "texts": {
    "mode": "perAd",
    "perAd": {
      "one.jpg": { "sponsorPageId": "555", "sponsorInstagramId": "666" },
      "two.jpg": { "sponsorPageId": null, "sponsorInstagramId": null }
    }
  }
}
```

For ad-set scopes, use `texts.mode: "perAdset"` and `texts.perAdset`, keyed by the final ad-set name/ID or `campaign::key`. In multi-campaign launches, use `profile.campaigns[<id or unambiguous name>]` for campaign sponsor overrides. Resolution is shared partner, then campaign, then the active row. Omitted sponsor fields inherit independently; explicitly set both IDs to `null` for No Partner. Switching only the partner Page does not clear an inherited Instagram ID on uploaded media; set that ID explicitly when changing the pair. Row First Identity fields (`pageId`, `instagramId`, `threadsId`) remain independent.

For an Instagram-only Second Identity, set `sponsorPageId: null`, `sponsorInstagramId` to the approved account ID, and `sponsorPageUseInstagramAccount: true`. A partner Facebook Page is also supported for uploaded media. The restriction on Facebook **post imports** does not apply to uploaded media.

`displayMode` applies across the launch: `both` (default), `first`, or `dynamic`. Each effective partner must differ from the First Identity. As in the Uploader, a partner whose approval is pending, or who has not approved partnership advertising for this First Identity yet, does not block the launch: preview and create return a `PARTNERSHIP_APPROVAL_PENDING` warning naming the partner, and Meta may create the ads in pending delivery until the partner approves. Every row needs a complete inherited or explicit partner, or an explicit No Partner override. Enabling partnerships without any sponsor is refused. Approval is rechecked at preview and create; a saved build does not store permission grants.

`ads create:preview` and `ads_preview` show each ad's effective partner, approval and header mode, grouped by ad set. Complete web-saved uploaded-media partnership builds can launch through the CLI and MCP; builds edited through these tools restore Partnership Ads, identities, row partners and header mode in the web uploader.

## Instagram existing-post partnership ads

Use JSON specs with `mediaItems[].kind: "partnershipPost"` to promote existing Instagram posts through CLI or MCP. Import a post using `import.postUrl`, `import.instagramShortcode`, or `sourceInstagramMediaId`; an `import.adCode` can accompany that locator or be used alone. Set `platform: "instagram"` unless the import is code-only. Facebook headless post imports are not supported. Uploaded-media partnerships are also supported, as described above.

`copyFromAd` selects the settings template, which can itself be an eligible partnership ad. It does not select the creator's post. The server checks the imported post's creator and permissions independently. Choose the sponsoring brand in `profile.partnership`; `--page` and `--instagram` still assert the actor identity, not the sponsor. Omit those actor overrides for batches containing different creators.

```json
{
  "accountId": "act_123",
  "copyFromAd": "456",
  "adSet": { "mode": "single", "id": "789" },
  "mediaItems": [{
    "kind": "partnershipPost",
    "mediaName": "creator-post-one",
    "platform": "instagram",
    "import": { "postUrl": "https://www.instagram.com/p/POST_SHORTCODE/" }
  }],
  "profile": { "partnership": {
    "enabled": true, "uploaderMode": true,
    "sponsorPageId": "111", "sponsorInstagramId": "222", "displayMode": "both"
  } },
  "texts": { "mode": "common", "common": {
    "headline": "", "callToAction": "", "link": "", "testimonial": "",
    "aiDisclosure": false, "multiAdvertiserAds": false
  } },
  "options": { "status": "PAUSED" }
}
```

Replace the illustrative IDs and URL with your authorized post, template, destination and brand. Each `mediaName` must be unique. Use one locator per row, optionally with a matching code. The organic caption stays unchanged; headline, CTA, URL and testimonial may be empty. A nonempty CTA needs a website URL. Omitted text inherits; explicit `""`, `false` and nullable sponsor IDs retain their meaning.

For per-ad overrides, set `texts.mode: "perAd"` and key `texts.perAd` by `mediaName`. For per-ad-set overrides, set `texts.mode: "perAdset"` and key `texts.perAdset` by the final ad set name/ID or `campaign::key`. Both accept the same optional text plus `sponsorPageId`, `sponsorInstagramId` and `sponsorPageUseInstagramAccount: false`. Changing the sponsor Page without specifying Instagram clears the inherited Instagram identity. `adSet.groups[].media` references media names; reuse one row across groups for the same post. `perAdSet` and legacy `texts.partnership` are accepted compatibility shapes.

Use 1 to 250 posts per request, each with a unique `mediaName` of at most 200 characters. `sponsorPageId` must resolve to an actual brand Facebook Page for every planned ad; `sponsorInstagramId` is optional. Header `displayMode` is launch-wide: `both`, `first`, or `dynamic`. For imported posts, `first` means the creator-only header, labelled **Partner identity only in the header** in the uploader, not a brand-only header.

Per-campaign sponsors use `profile.campaigns`, keyed by campaign ID or an unambiguous campaign name, with `sponsorPageId` and optional `sponsorInstagramId`. Resolution is shared sponsor, then campaign, then the active per-ad-set or per-ad override. Put shared sponsor fields in `profile.partnership`, never `texts.common`; an `adCode` belongs in the row's `import` or an active per-ad/per-ad-set block. Creator `pageId` / `instagramId` values are assertions, not a way to change the post's author.

`multiAdvertiserAds` defaults to `true`; set it explicitly to `false` to opt out. CTA values must be Meta enums such as `SHOP_NOW` or `LEARN_MORE`, not labels such as `Shop Now`, and a nonempty CTA requires an `http://` or `https://` destination. The organic caption cannot be edited. Imported-post text blocks support one headline, CTA, link, URL tags, testimonial, AI disclosure and multi-advertiser choice. Each per-ad/per-ad-set override map supports at most 200 entries.

For imported posts, the CLI and MCP reject Facebook post imports, batches mixing imported posts and uploaded media, carousel/flexible/Multi Media structures, placement-variation grouping, and Threads identities. Uploaded media supports the normal creative formats and per-row partners. New Facebook post imports are currently unsupported in the web uploader too. There is no approved-post browser in the uploader; import an authorized Instagram URL, post ID or code.

For a code-only import, use this complete request body with your IDs and a code inserted by an in-memory secret producer. Do not save the real code in a file:

```json
{
  "accountId": "act_123",
  "copyFromAd": "456",
  "adSet": { "mode": "single", "id": "789" },
  "mediaItems": [{ "kind": "partnershipPost", "mediaName": "creator-post-one", "platform": "instagram", "import": { "adCode": "REPLACE_IN_MEMORY" } }],
  "profile": { "partnership": { "enabled": true, "uploaderMode": true, "sponsorPageId": "111", "sponsorInstagramId": "222", "displayMode": "both" } },
  "texts": { "mode": "common", "common": { "headline": "Discover the collection", "callToAction": "LEARN_MORE", "link": "https://example.com/", "multiAdvertiserAds": false } },
  "options": { "status": "PAUSED", "pauseAt": "ad" }
}
```

Ad codes are sensitive: put them in a JSON body supplied through `/dev/stdin`, never CLI arguments, query strings, or a spec file saved to disk. A scoped `adCode` in the active per-ad/per-ad-set block overrides the row's import code for that exact brand and post.

```bash
ads create:preview partnership.json --account act_123 --json
# Have your secret-aware producer pipe its JSON into this command:
ads create:preview /dev/stdin --account act_123 --json
```

Pass `--account act_123` to preview/create commands or set a default with `ads account act_123`; the account in a JSON spec does not replace that CLI selection.

Preview resolves Meta content and may make read/validate-only permission calls, but creates no ads and stores no codes. It returns per-ad creator/sponsor details, authorization warnings and a code-free `resolvedSpec`. Supply transient codes again at create. Create rechecks permissions and may save verified codes encrypted for the exact brand/post scope. Saved builds retain configuration, not codes. A structurally launchable build can still require renewed Meta permission at launch. The web editor refuses to overwrite imported scopes it cannot yet represent.

### Exact ad and ad set names

For literal names, set `mediaItems[].customName` (panel rename) or
`adNames: { "photo.jpg": "  Été 🌞 {index}  " }` (per-ad override).
Precedence is `adNamePattern` < media `customName` < `adNames[mediaName]`.
A blank or whitespace-only value clears that level. Nonblank names keep all
spaces and tokens, without suffixes or length caps; only NFC normalization applies.
Renaming leaves other template-named ads' index numbers unchanged.

Use `adSet.names` for new ad set overrides. Keys are JSON-encoded arrays:
`JSON.stringify([campaignName, targetingKey, urlVariantKey || urlVariantLabel || ""])`.
Use the final execution plan after campaign and URL expansion. Targeting keys
identify `single`, `media:<id>`, `auto:N`, or `group:<id>`, with a campaign
prefix in multi-campaign mode. The URL variant key is the resolved unique label
before date tokens are expanded. Preserve captured media `_id` values on resume.
These keys survive changes to ad set name templates. Existing Meta ad sets cannot
be renamed. Each typed name must be unique among the new ad sets in its campaign:
a name that matches another new ad set is rejected (`adSet.names`), because new ad
sets with the same name are created as one ad set.

Per-ad-set copy, links, CTA, profiles and disclosure still use the original
template name in `texts.perAdset`; `adSet.names` is applied after those lookups.

## Reporting beta and capacity

Reporting and resource reads (`insights`, `insights:job`, `insights:health`, `account:info`, `audiences`, `conversions`, `estimate`, `meta-activity`, `library:images`, `library:videos`, `posts`), edits (`edit:*`, `edits`, `launch:discard`), History (`history`, `history launch`, `inbox:history`) and copies by ID (`duplicator:copy`, `duplicator:copy:preview`) are available to beta accounts while in beta. The web History page is open to everyone. Existing CLI/MCP access is still required. Hosted MCP hides these tools for accounts without access; local MCP hides them when `/v1/me` returns `betaFeatures: false`, and the server enforces access either way. Existing campaign, ad set and ad lists remain available. The CLI/API static field catalog stays accessible; account-scoped discovery requires beta access. Access errors display the server's explanation.

Queries allow 10 requests per minute, report-job reads 60, and other new reads 20. Meta's reported API usage determines when fresh calls pause. At 75% usage, Ads Uploader waits for Meta's recovery estimate; without an estimate, the wait is 60 seconds, or 300 seconds at 90% usage. Fresh calls also share a concurrency cap of 5 across the app and 1 per account for each read family.

Ads Uploader also has a short-window read safety limit: a burst of 120 calls per user, replenished at 30 calls per minute, and 240 calls per ad account across users, replenished at 60 calls per minute. This is an Ads Uploader limit, not a Meta limit. Async status checks skip concurrency slots but use this safety limit. Existing campaign, ad set and ad lists keep their request burst limit and now respect Meta usage signals. Cached results make no Meta calls.

Edits, previews, applies and copies skip every Ads Uploader read limit, pause and budget, and go straight to Meta; only one apply runs at a time per ad account. They stop only when Meta itself throttles a request (Meta error 4, 17, 32, 613 or 80000-80014, or HTTP 429). That stop says it was Meta, with Meta's code and Meta's own wait, and carries `source: "meta"`, `metaCode` and `metaSubcode`; untouched changes stay pending.

Reporting and resource reads share 20 requests per second per user, plus 600 per minute per operation. `RATE_LIMITED` returns `Retry-After: 1`. Fresh calls queue for up to 20 seconds for app and account capacity; a queue timeout returns `Retry-After: 5`. An eligible first-page Insights query in auto mode returns a queued job instead. Follow its job ID. At most 10 duplicate requests wait for the same result; `TOO_MANY_DUPLICATE_REQUESTS` returns `Retry-After: 2`.

Fresh Meta rows, including metadata, resource pages, multi-account pages and job results, share a monthly UTC quota. Cached rows do not count again. Defaults are 1,000,000 rows per tier, with unlimited admins. `meta.quota` contains `rowsUsed`, `rowsLimit` and `resetsAt`; unlimited limits and unavailable usage are null. `ROW_QUOTA_EXCEEDED` means wait for the reset on the first day of the next month. Respect that reset. The client's existing 60-second automatic retry ceiling is unchanged; reads still make one request attempt.

CLI table views print one quota note at 80% usage. JSON output remains unchanged, including legacy list shapes.

Respect `Retry-After` for `META_READS_BUSY`, `META_READS_BACKING_OFF` and `READS_GUARD_LIMIT`. `READS_GUARD_LIMIT` replaces `META_READS_HOURLY_LIMIT` and `META_READS_ACCOUNT_LIMIT`, which remain legacy aliases for the Ads Uploader safety limit. Never retry in a loop. Cached rows remain available during backoff; `ENTITY_METADATA_DEFERRED` and `COMPARE_DEFERRED` mean optional fresh reads were deferred.

`META_REQUEST_TOO_LARGE`: do not retry the same request; ask the user to contact support.

Campaign health preserves native result arrays and adds `result` with its source, metric, value and cost per result. When native results are absent, it uses delivering ad sets' optimization goals; mixed goals remain separate and incomplete coverage is unknown. `learningStageCounts` is an empty object when no learning information was returned, and null when the ad sets could not be read. Account `spend_cap: null` with `hasSpendCap: false` means no cap; CLI tables show None. Major money amounts are numbers; raw minor amounts remain strings.

## Edits And Edit Review

Enable edits with **Agent Edit** on the **Account Permissions** page: an account-wide default or a per-ad-account setting, which wins. **Approval** sends changes to the user's Agent Inbox; **Allowed** applies them directly. Edit commands return `reviewUrl`, `reviewId` and proposal IDs on compact `items`. Share the link with the user and let them select and approve items in their Agent Inbox. Reuse `--review-id <reviewId>` on later previews to keep related proposals together. Reviews expire seven days after their last update.

Use `edit:status`, `edit:archive`, `edit:delete`, `edit:rename`, `edit:budget`, `edit:spend-cap`, `edit:spend-limits`, `edit:bid`, `edit:schedule`, `edit:targeting`, `edit:adset-settings` and `edit:creative` with `--account` and optional `--json`. `ads edit:reviews` lists reviews; `ads edit:review <reviewId>` shows proposal statuses and applied edit IDs. Both support tables and `--json`. `edits`, `edit:show <editId>` and `edit:undo <editId>` provide edit history, results and undo previews. Undo also goes to Edit Review in review mode. A suggestion shows `applying` while an approved change is being sent to Meta; read again shortly for `applied` or `failed`.

Review mode queues proposals after batched reads and local checks, with zero Meta POSTs. Meta accepts or rejects each change when the user applies it; results show each item as applied, failed, skipped or pending. Pending means not sent yet, with a reason; apply the same review again to send these changes. Failed proposals retain a safe Meta reason. One ad's failure does not stop other ads; account errors stop the run. Direct previews also skip validate_only for status, archive, delete and rename; other direct edits still validate.

Previews return compact `summary`, `items` and grouped warnings by default. Use `--detail full` to request full change details and any performed validation. Money metadata appears only when money fields change. Calls support 100 objects (50 IDs per bulk operation), and a review holds 200 proposals.

Cancel pending suggestions with `ads edits cancel <reviewId> --account act_123 --proposal <id> --proposal <id>`. The review ID is required; omit `--proposal` to cancel every pending suggestion in that review. `ads edit:cancel` is an alias. Nothing changes in Meta. The command prints one result per suggestion, including refusal reasons and applied edit IDs, followed by “N cancelled, M still pending”. Use `--json` for the full response. Cancelled suggestions leave your Agent Inbox; in-progress and other non-pending suggestions are refused.

Discard a launch waiting in your Agent Inbox with `ads launch:discard <buildId> --account act_123`. Nothing is created in Meta and the saved build stays; use `ads builds:delete <buildId>` to remove it as well. Launches that already started cannot be discarded.

In **Apply Directly** mode, previews produce a plan that lasts 15 minutes. Show its changes and warnings first. `--preview-only` stops before applying. Interactive commands ask for approval; non-interactive apply requires `--yes` and the applicable explicit flags: `--confirm-spend`, `--confirm-live-change`, `--confirm-irreversible`, `--allow-large-change`. Set these only after the user approves the corresponding change. `--yes` is not spend approval. `edit:apply <planId>` refuses with `EDITS_REVIEW_REQUIRED` if review mode is active, even if the plan was created in direct mode. Follow its review link instead.

Archive and delete cannot be undone. Never retry partial results blindly; inspect `edit:show` and current Meta values first. Money inputs are major units and edit preview/detail money values are minor units; History rows use readable currency amounts. `account:info` shows the effective setting and its source. The CLI cannot change settings. See https://adsuploader.com/docs/automation/edits.

Target audiences with `ads edit:targeting <adset-id> --add-country CA --remove-interest <id> --age-min 25 --preview-only`. List flags are repeatable; use targeting search for IDs. `--json-patch <file|->` accepts `{ "add": {}, "remove": {}, "set": {}, "group": 0 }` instead of patch flags. Use `--detail full` to show per-path differences and the full resulting audience; the default preview summarizes detail paths. Several detailed-targeting groups require an explicit `--group` index. Unmentioned targeting stays in place, and undo restores the recorded full audience.

`ads edit:adset-settings <adset-id>` supports `--optimization-goal`, `--billing-event`, `--promoted-object <json>`, repeatable `--attribution EVENT_TYPE:days`, `--incremental-attribution on|off`, `--dsa-beneficiary`, `--dsa-payor`, or `--json-patch <file|->`. Standard attribution requires incremental attribution to be off first. Destination type is read-only.

`edit:bid` supports `--strategy LOWEST_COST_WITH_MIN_ROAS` and ad set `--roas-floor 2.5` for 250%. Minimum ROAS requires `VALUE` optimization. With a campaign budget, set the strategy on the campaign and floors on its ad sets. Targeting and ad set settings do not count as spend increases, but live-change approval still applies. All new edits use the same Edit Review flow.

### Editing An Ad Creative

`ads edit:creative <ad-id>` changes an existing ad's creative through the usual preview and Edit Review flow. Use `--primary-text`, `--headline`, `--description`, `--cta`, `--link <https-url>`, `--display-link` and `--url-tags` (without `?`). For multi-text creatives use repeatable `--body`, `--title`, `--desc` instead of single-text flags. Duplicate variants are refused. Empty display link or URL tags clears inheritance.

To change an image or video, create a new ad in the Uploader. Text-only edits keep the existing image or video.

`--post <pageId_postId>` switches to an existing Page post and accepts only `--url-tags` alongside it. `--json-patch <file|->` accepts the API creative fields instead of these flags.

```bash
ads edit:creative 12345 --primary-text "Updated primary text" --account act_123 --preview-only
ads edit:creative 12345 --body "First variant" --body "Second variant" --preview-only
```

Use `--detail full` for the table of creative field values; the compact default includes a detail-path count and warnings. **Old-post likes, comments and shares do not carry over**, except when switching to an existing post. Expect Meta review and a possible learning reset. Up to 10 creative edits fit in a plan. Undo reattaches the old creative. Existing-post sources refuse text edits. Plain and placement-customised carousels support primary text, whole-carousel link, CTA, display link and URL tags. Use `--json-patch` with `cards: [{ card: 2, headline: "Updated headline", description: "Updated description" }]` for per-card text; card numbers start at 1. Plain carousels also accept per-card `link`; placement-customised carousels refuse it. Scalar headline and description edits are refused on carousels. Flexible, catalog and partnership creatives point to the Uploader's copy-settings flow. Inspect `error.details.orphanCreativeId` if creation succeeded but attachment failed; never retry a partial result blindly.

## Saved-build safety and issue reports

Save and update responses include `warnings` and `display` with `totalAdSets`, `totalAds`, and the ads in each ad set. The display uses the web uploader's grouping logic. Compare it with preview totals before launching. An imported-post configuration that the web editor cannot represent returns `display.complete: false` with a reason and null totals; resolve that limitation before comparing totals. Report mismatches, contradictory results, silently ignored fields, or broken tool descriptions using `ads_report_issue` or `ads report:issue`.

Group media and per-ad text keys resolve by upload ID, Meta media ID/hash, exact file name, then variant base name. Ambiguous references fail. Unmatched custom groups fail at preview and create. To reuse an existing destination, supply its campaign/ad-set ID explicitly.

Per-ad copy names ignored shared headlines, primary text and descriptions in `warnings`. Shared link, CTA, display URL and URL tags still apply unless a row overrides them. Group `dailyBudget`, `lifetimeBudget` and `bidAmount` use whole account currency units; `minimumRoas` is a ratio. Daily and lifetime budgets are exclusive; lifetime budgets require `options.schedule.endTime`. Ad-set budgets cannot be used under a campaign budget. These group budget controls are currently server/spec-only.

Existing library videos keep their Meta video ID. At preview/create, the server recovers a missing thumbnail from Meta and uploads only that image for its hash. If Meta has no usable thumbnail, the request fails with the video ID.

```bash
ads report:issue --tool ads_preview --expected "Two ad sets" --actual "One ad set" --build-id build_123 --severity error --json
```

`POST /api/v1/issues` accepts `tool`, `expected`, `actual`, and optional `buildId`, `requestId`, `jobId`, `adIds`, `accountId`, and `severity` (`info`, `warning`, `error`, `critical`). Text and IDs are capped and redacted. Reports are logged for the Ads Uploader team to review. The response is `{ "reported": true, "reference": "report_...", "status": "queued" }`. The limit is 10 reports per user per hour. Reporting is available to all CLI/MCP users.

## Request missing capabilities

Use `ads request:feature` when a goal needs a tool, parameter, filter, action, or level that is not available. Agents should file the request themselves without asking the user, then explain the limitation and continue with the best workaround. Use `ads report:issue` when an existing tool misbehaves.

```bash
ads request:feature --goal "Pause seasonal ads" --missing "Bulk pause ads by label" --tools-tried ads_list_ads,ads_edit_preview --level ad --workaround "Pause ads individually" --json
```

`--goal` and `--missing` are required. `--tools-tried`, `--account`, `--level`, `--workaround`, and `--json` are optional. No account selection is required. Describe the missing capability without names, emails, ad copy, tokens, or other personal data.

Returns `requested: true`, a `reference`, `status: 'recorded'`, `duplicate`, and `timesRequested`. Similar requests are grouped; the same user's repeat within 24 hours does not increase the count. Available to every CLI user, limited to 20 requests per user per hour. The equivalent MCP tool is `ads_request_feature`.

## Product Showcase

Product Showcase shows products from a catalog under a single image or video ad. Full catalog (dynamic/DPA) ads are not supported yet.

IDs must be strings of digits. Recommendations require Sales campaigns (`OUTCOME_SALES`); manual catalog/set choices also work on other objectives. Carousel, flexible, multi-media, existing-post and Partnership ads cannot enable Product Showcase.

Discover catalogs and sets, then request a recommendation:

```bash
ads catalogs list --account act_123
ads catalogs product-sets 3390205374500501 --product-set 29588799347389253 --json
ads catalogs recommend --url https://www.asphaltgold.com/collections/nike-sneaker --page 1234 --pixel 5678
ads create:preview spec.json --product-showcase recommended
ads create spec.json --product-showcase on --catalog 3390205374500501 --product-set 29588799347389253
ads create:preview spec.json --product-showcase off
```

Create and preview accept `productShowcase: { enabled?, catalogId?, productSetId?, useRecommendation? }`; `enabled` defaults to true. Omit the field to inherit the source creative. `RECOMMENDED` selects Meta’s default (`choiceSource: "DEFAULT"`); `MUST_PICK` requires a returned candidate ID as `productSetId` (`"MANUAL"`). `MANUAL` requires a catalog and product set. A validated override of Meta’s default uses `"MANUAL"`.

CLI create and preview flags: `--product-showcase <on|off|recommended>`, `--catalog <id>`, `--product-set <id>`. IDs alone mean on; recommended maps to `useRecommendation: true`. Off maps to `{ enabled: false }` and cannot include IDs; it disables showing products while keeping the source ad’s own set attached.

Preview returns `productShowcase` with enabled, catalog/set IDs, optional set name, choiceSource and source (`recommendation`, `manual`, `source_ad`), or null when omitted. Recommendations are read-only and do not enable the feature. Test Mode makes no catalog calls, requires explicit catalog/set IDs when enabled and notes that membership/recommendations were not verified. Saved UI choices map to Product Showcase; saved defaults are checked again before launch.
