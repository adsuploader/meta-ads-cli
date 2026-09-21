---
name: ads
description: Create and manage Facebook/Meta ads using the Ads Uploader CLI. Upload media, preview configurations, create ads, browse campaigns and ad sets.
homepage: https://adsuploader.com
---

## Setup

```bash
npm install -g @adsuploader/cli
```

## Authentication

```bash
# 1. Login (opens browser OAuth flow)
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

When you update an existing saved build for someone working in an open uploader tab, confirm that the update was saved; the tab picks it up automatically. Share the build URL only if the user asks for it or says the tab is closed. For a new headless build with no open uploader tab, include the web URL so the user can open and review it.

Use `ads builds:fork <buildId>` when the user wants a variant but the original build must remain unchanged. Continue edits against the new build reference printed by the command.

---

## Command Reference

### Auth
| Command | Description |
|---------|-------------|
| `login` | Authenticate via browser |
| `logout` | Clear credentials |
| `whoami` | Show current user |
| `config` | Show configuration |

### Browsing
| Command | Description |
|---------|-------------|
| `accounts` | List ad accounts |
| `account <id>` | Set default account |
| `pages` | List Facebook Pages available for `--page` / `profile.pageId` |
| `campaigns` | List active campaigns (`--status all` for all, `--search <text>` to filter) |
| `campaign <id>` | Show ad sets in a campaign |
| `adsets --campaign <id>` | List ad sets (supports `--search`, `--status`) |
| `adset <adSetId>` | Show ads in an ad set |
| `ad <adId>` | Ad details + creative config |
| `presets` | List saved API presets (or `presets <id>` for details) |
| `presets:save --from-ad <adId> --name <name>` | Save an existing ad as an API preset |
| `text-presets` | List saved text presets (or `text-presets <id>` for details) |
| `uploads` | List recent upload batches (or `uploads <batchId>` for details) |
| `targeting:search <query> --type city` | Find `targeting.cities[].key` values |
| `targeting:search <query> --type zip` | Find `targeting.zips[].key` values for zip / post code targeting |
| `targeting:search <query> --type detailed` | Find `targeting.detailedTargetingGroups` entries: interests, behaviors, employers, job titles, industries |

### Actions
| Command | Description |
|---------|-------------|
| `upload <files...>` | Upload images/videos |
| `create spec.json` | Create ads from spec |
| `create:preview spec.json` | Dry run — show what would be created |
| `create:test spec.json` | Admin-only Test Mode with Meta validate-only results |
| `create:interactive` | Guided wizard (accepts all create flags) |
| `duplicator:post-id [specFile]` | Duplicate existing ads selected by Page post ID while preserving post references |
| `duplicator:post-id:preview [specFile]` | Preview a post-ID duplication and its source-to-destination mapping |

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

### Post-ID Duplication Flags
These apply to `duplicator:post-id` and `duplicator:post-id:preview`. This mode selects existing ads by Page post ID and preserves their post references (likes, comments, shares), so it is separate from `create --copy-from`, which builds new ads from copied settings. Other duplication modes may be added later.

| Flag | Description |
|------|-------------|
| `--account <id>` | Override default ad account |
| `--post <postId>` | Select source ads by Page post ID (discovery). Repeatable. Searches up to ~2,000 recent account ads; if a post matches more than one ad, the preview returns candidates and a live run requires an explicit `--ad` selection. |
| `--ad <adId>` | Select source ads directly and exactly by ad ID (first-class; pins a specific ad and resolves post ambiguity). Repeatable. |
| `--campaign <id>` | Existing destination campaign |
| `--adset <id>` | Existing destination ad set, or the template when a new-ad-set flag is present |
| `--new-adset [name]` | Create ONE new ad set for all source ads (default name `{AdName}`; becomes `Multiple Ads` with several sources) |
| `--new-adset-per-ad [name]` | Create one new ad set per source ad (`{AdName}` and `{Index}` supported) |
| `--ad-name <pattern>` | Duplicated ad name pattern (default `{AdName}`; `{Index}` supported) |
| `--paused` | Create the duplicated ads paused |
| `--use-creative-id` | Reuse original creative IDs instead of post-reference creatives |
| `--json` | Output raw JSON for scripting |

A post-ID duplication spec file is the v1 request body (`postIds` for discovery or ordered `adIds` for exact selection, an existing `campaignId`, `adSetId` / `newAdSet` / `newAdSetPerAd`, `adNamePattern`, `paused`, `useCreativeId`); flags override it field by field. New campaigns are not supported: the destination is always an existing campaign. New ad sets clone every setting (targeting, budget, schedule, bid) from the template ad set: the given `--adset` if provided, otherwise each source ad's own ad set. Always run `duplicator:post-id:preview` first; if a post is ambiguous the preview returns candidates and `requiresSourceSelection`, and a live run needs the resolved `adIds`. Dynamic-creative source ads reuse their creative ID and cannot preserve engagement; the preview lists them as warnings.

### Detail Flags
| Flag | Description |
|------|-------------|
| `ad --expanded` | Show full headline, primary text, and description values |

### Common Flags
| Flag | Description |
|------|-------------|
| `--account <id>` | Override default ad account |
| `--json` | Raw JSON output (available on most commands) |

### Do not use `--json`

The `--json` flag is for shell scripts piping to `jq`. **Never use it** — the human-readable output already contains all the information you need (batch IDs, job progress, results). For `create`, `--json` replaces the live progress log with raw NDJSON poll dumps which are unreadable.

---

## Spec File Format

The JSON spec controls ad creation. Provide a template source plus either `uploadId` or captured `mediaItems` containing `mediaHash` for every image and `mediaId` for every video. Standard, carousel, flexible, and placement videos also need `thumbnailHash`; Multi Media videos use their captured public thumbnail URL. Saved builds created in the web uploader carry these values automatically when available.

### Minimal Spec

```json
{
  "adPresetId": "preset_id_here",
  "uploadId": "batch_abc123"
}
```

### Template Source (pick one)

| Field | Description |
|-------|-------------|
| `adPresetId` | Saved API preset ID (locks campaign, ad set, ad config) |
| `copyFromAd` | Facebook ad ID to copy settings from |

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
  - `perUpload` / `autoGroup`: each duplicated ad set name is `{base}-{label}`, or substitutes a `{destination}` token if present in the pattern.
  - Explicit `adSet.groups[]`: use `{group}` (aliases: `{name}`, `{base}`, `{adset}`) in `adSet.namePattern` to reference `groups[i].name`.
  - Variation grouping: use `{variation}` in `adSet.namePattern` to reference the filename prefix before `variationIdentifier`.
- The `{date}` token resolves to today's date in labels (`launch-{date}` → `launch-2026-04-27`).
- Each destination URL must be distinct. Trailing-slash and case variants of the same URL collapse to one — supply at least 2 truly different destinations.
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
| `"none"` | All features off |
| `["text_translation", "image_touchups"]` | Listed features on, rest off |

Available features: `text_translation`, `inline_comment`, `enhance_cta`, `text_optimizations`, `reveal_details_over_time`, `show_destination_blurbs`, `image_brightness_and_contrast`, `image_touchups`, `video_auto_crop`, `video_filtering`, `image_animation`, `image_templates`, `adapt_to_placement`, `product_extensions`, `description_automation`, `add_text_overlay`, `music`, `carousel_to_video`, `carousel_dynamic_description`, `multi_share_end_card`, `multi_share_optimized`

`show_destination_blurbs` is displayed as **Show spotlights**.

When cherry-picking features, only list ones relevant to the media type. `video_auto_crop` and `video_filtering` only apply to video ads. `carousel_to_video`, `carousel_dynamic_description`, `multi_share_end_card`, and `multi_share_optimized` only apply to carousel ads.

### AI Transparency

Self-disclose that an ad's creative was made or significantly edited with AI (Meta's AI-content self-disclosure). Off by default — never auto-enabled.

```json
{ "aiDisclosure": true }
```

Or via flag: `--ai-disclosure`. Applies to every ad in the upload. When on, each created ad opts in to Meta's AI-content self-disclosure; when off, no disclosure is sent.

### Carousel Ads

Group uploaded files into carousel ads with per-card text and optional overall carousel text. `cardTexts` controls individual cards. Overall carousel text can be colocated on the carousel object or set in `texts.perAd` using the carousel `name`; colocated fields win when both are present.

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

Assets must reference filenames from the upload batch or captured saved-build media. `assetTexts` is optional and lines up with `assets` by index; each field is a single override for that asset and blank fields fall back to the ad's main text and URLs. The primary rendered asset uses the ad's main text and URL, and any primary asset text override becomes the first main text option. Videos need a captured cached public thumbnail URL. Files claimed by a Multi Media group are removed from the standard ad list.

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
| `schedule.startTime` | ISO 8601 string | Scheduled start (uses ad account timezone) |
| `schedule.endTime` | ISO 8601 string | Scheduled end (optional) |

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
9. **Do not use `--json`** — it's for shell scripts, not interactive use. The human-readable output has everything you need
10. **Never wrap CLI commands in polling loops** (`watch`, `while true; do ads adset X; sleep 5; done`, or equivalent). If you need to follow a job, use `ads jobs JOB_ID --follow`. If you need to verify ads landed after a create, run `ads adset <id>` **once** after the create returns — the job engine only marks complete when every ad is created on Facebook. Polling loops trigger rate limits (per-user-per-operation; list/read endpoints cap at 20/min) and return `429 Rate Limit Exceeded` with a `Retry-After` header.
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

`displayMode` applies across the launch: `both` (default), `first`, or `dynamic`. Each effective partner must differ from the First Identity and have approved partnership advertising access. Pending or missing approval is refused with the identity named. Every row needs a complete inherited or explicit partner, or an explicit No Partner override. Enabling partnerships without any sponsor is refused. Approval is rechecked at preview and create; a saved build does not store permission grants.

`ads create:preview` and `ads_preview` show each ad's effective partner, approval and header mode, grouped by ad set. Admin-only `ads create:test` or `ads_create` with `options.testMode: true` validates eligible calls with Meta without creating ads and reports passed, failed and not-checked counts. Complete web-saved uploaded-media partnership builds can launch through the CLI and MCP; builds edited through these tools restore Partnership Ads, identities, row partners and header mode in the web uploader.

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
# Admin-only Test Mode, when authorized:
ads create:test /dev/stdin --account act_123 --status PAUSED --json
```

Pass `--account act_123` to preview/create commands or set a default with `ads account act_123`; the account in a JSON spec does not replace that CLI selection.

Preview resolves Meta content and may make read/validate-only permission calls, but creates no ads and stores no codes. It returns per-ad creator/sponsor details, authorization warnings and a code-free `resolvedSpec`. Supply transient codes again at create. Create rechecks permissions and may save verified codes encrypted for the exact brand/post scope; Test Mode can also write encrypted codes and launch records, but does not create Meta ads. Saved builds retain configuration, not codes. A structurally launchable build can still require renewed Meta permission at launch. The web editor refuses to overwrite imported scopes it cannot yet represent.

Admin-only `create:test` submits eligible calls to Meta with `execution_options: ["validate_only"]` and returns `metaValidation` passed / failed / not-checked counts. It exits 1 on validation failure. Calls needing simulated IDs are not checked; a passed validation does not prove delivery or placement appearance.
