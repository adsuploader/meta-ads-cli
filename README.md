<p align="center">
  <img src="assets/logo.png" alt="Ads Uploader" width="96" height="96">
</p>

<h1 align="center">Ads Uploader CLI</h1>

<p align="center">Create and manage Meta (Facebook &amp; Instagram) ads from the command line — same pipeline as the web app.</p>

<p align="center">
  <a href="https://www.npmjs.com/package/@adsuploader/cli"><img alt="npm" src="https://img.shields.io/npm/v/@adsuploader/cli?color=1a3a5c"></a>
  <a href="https://adsuploader.com"><img alt="Website" src="https://img.shields.io/badge/website-adsuploader.com-1a3a5c"></a>
  <img alt="Meta Ads" src="https://img.shields.io/badge/Meta-Facebook_%26_Instagram-1a3a5c">
  <a href="https://github.com/adsuploader/cli/stargazers"><img alt="Stars" src="https://img.shields.io/github/stars/adsuploader/cli?style=flat&color=1a3a5c"></a>
</p>

---

Ads Uploader turns an existing ad into a reusable template and builds brand-new ads on top of it — new media, copy, CTA, and targeting — in bulk, and paused by default. The `ads` CLI drives that pipeline from your terminal or from an AI agent. Prefer a fully hosted, agent-native experience? See the [Ads Uploader MCP server](https://github.com/adsuploader/mcp).

> **Safe by default.** `create:preview` resolves posts and checks Meta permissions without creating anything, and created ads are **paused** unless you explicitly say otherwise. Nothing spends money until you unpause it in Meta.

## Why Ads Uploader

Founded on a decade of hands-on Meta advertising, Ads Uploader gives you the **deepest, most complete Meta (Facebook &amp; Instagram) ad-creation workflows available** — everything needed to actually ship campaigns end to end:

- **Template from any live ad** — copy a proven ad's exact settings, then rebuild on top with new media, copy, and targeting.
- **Real bulk** — many ads across campaigns and ad sets in a single command, with per-ad and per-ad-set text.
- **Partnership / branded content ads** — uploaded-media partnerships (Facebook + Instagram) and imported Instagram creator posts, with shared or per-ad/ad-set sponsors.
- **Duplicate preserving social proof** — clone by post so likes, comments, and shares carry into new campaigns.
- **Creative depth** — carousel, flexible, Multi-Media, Dynamic Optimization, creative enhancements, Advantage+ targeting, CTA and URL tagging.
- **Human-in-the-loop by design** — validate-only previews, paused-by-default creation, and saved builds you finish in the web app.

Built by a team that runs Meta ads at scale, and trusted by performance marketers and agencies managing serious Meta budgets.

## Install

```bash
npm install -g @adsuploader/cli
```

Requires Node.js 18 or later.

## Quick Start

```bash
ads login                       # authenticate via browser
ads accounts                    # list ad accounts
ads account act_123456          # set default account
ads targeting:search "90210" --type zip  # find a zip / post code key

ads upload hero.jpg https://cdn.example.com/banner.mp4  # local files and public URLs
ads upload:drive "https://drive.google.com/drive/folders/..."
ads create:preview spec.json    # preview what would be created
ads create spec.json            # create ads
```

## Documentation

Full documentation including the spec file reference, campaign structure, text configuration, and all available options is at [adsuploader.com/docs/ad-configuration/cli](https://adsuploader.com/docs/ad-configuration/cli).

Spec launches support the same campaign-aware fields as the web uploader. Use `texts.perAdset` entries keyed by ad set name/ID or `campaign::key`, and use `profile.campaigns` keyed by campaign ID or an unambiguous campaign name for per-campaign Page, Instagram, and Threads identities. A Page-only override infers its linked Instagram account or falls back to the Page actor; choose that actor explicitly with `profile.useFacebookPage: true` or `--use-page-identity`.

For campaigns that use Meta's minimum-ROAS bid strategy, pass a positive ratio with `--minimum-roas` or set `adSet.minimumRoas` in the spec. For example, `--minimum-roas 1.5` sets a 1.5 ROAS goal. It is mutually exclusive with `--bid-amount`; combine it with `--daily-budget` for ABO or a campaign budget for CBO.

```bash
ads create spec.json --minimum-roas 1.5
```

To target new ad sets by exact zip or post code, find Meta's key with `ads targeting:search`, then add it to the spec's `targeting.zips` array. Zip and post code entries are key-only locations: they do not accept a radius or `distance_unit`. Omit the entire `targeting` block to inherit the source ad set's audience unchanged.

```bash
ads targeting:search "90210" --type zip
```

```json
{
  "targeting": {
    "zips": [{ "key": "US:90210" }]
  }
}
```

Any build with multiple ad sets can use `adSet.targetingPerAdSet`, keyed by final ad set name/ID or collision-safe `campaign::key`. Custom ad sets can also keep an override beside the group in `adSet.groups[].targeting`. Per-ad-set targeting is JSON-spec-only: create flags set the build default and cannot address an individual planned ad set. Resolution is `targetingPerAdSet`, then group targeting, then the top-level build targeting, then the source ad set.

```json
{
  "targeting": { "countries": ["US"] },
  "adSet": {
    "mode": "perUpload",
    "namePattern": "Ad Set {index:01}",
    "targetingPerAdSet": {
      "Ad Set 02": { "countries": ["CA"] }
    }
  }
}
```

## Using with AI

This package includes a `SKILL.md` file that describes every command and option for AI agents (Claude Code, Cursor, etc.). Point your AI tool at `node_modules/@adsuploader/cli/SKILL.md` to get started.

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

## Commands

| Command | Description |
|---------|-------------|
| `ads login` | Authenticate via browser |
| `ads accounts` | List ad accounts |
| `ads account <id>` | Set default account |
| `ads pages` | List Facebook Pages available for profile overrides |
| `ads targeting:search <query> --type zip` | Find zip / post code keys for `targeting.zips` |
| `ads campaigns` | List campaigns |
| `ads campaign <id>` | Show ad sets in a campaign |
| `ads adset <id>` | Show ads in an ad set |
| `ads ad <id>` | Ad details and creative |
| `ads presets` | List saved presets |
| `ads presets:save --from-ad <id> --name <name>` | Save an existing ad as an API preset |
| `ads builds [<id>] [--json]` | List saved builds or show one build |
| `ads builds:create [options]` | Create a durable build from a spec or web state |
| `ads builds:fork <id>` | Fork a build into a new autosaved draft |
| `ads builds:update <id> [options]` | Rename a build, safely patch its spec, or replace full state |
| `ads builds:delete <id>` | Delete a saved build |
| `ads upload <inputs...>` | Upload local paths and public HTTPS media URLs into one batch |
| `ads upload --retry-failed [batchId]` | Retry failed files into the same upload batch |
| `ads upload:drive <folderUrl>` | Import images and videos from a public Drive folder in a background job |
| `ads uploads` | List recent upload batches |
| `ads create spec.json` | Create ads from spec |
| `ads create:preview spec.json` | Dry run |
| `ads create:test spec.json` | Admin-only Test Mode with Meta validate-only results |
| `ads create:interactive` | Guided wizard |
| `ads duplicator:post-id [specFile]` | Duplicate existing ads selected by Page post ID while preserving post references |
| `ads duplicator:post-id:preview [specFile]` | Preview a post-ID duplication and its source-to-destination mapping |
| `ads jobs <id>` | Check job status |

Use either `postIds` for discovery or ordered `adIds` for exact selection, never both. Preview returns `sourceCandidates` with ad/campaign/ad-set IDs. When matches are ambiguous, `requiresSourceSelection` is true and `resolvedRequest` is null: choose the ad IDs and preview again. With an exact selection, `resolvedRequest` contains ad IDs and no post IDs. An existing `campaignId` is required; new campaigns are unsupported.

**Migration from newest-match selection:** post lookup searches up to about 2,000 recent account ads and no longer silently selects the newest matching ad. Live REST `POST /api/v1/duplications` requires `adIds`; submit preview’s `resolvedRequest`. CLI/MCP perform discovery before creating and stop for ambiguity or unacknowledged warnings. To run a previously reviewed selection, save and reuse `resolvedRequest` (MCP also needs `accountId`); do not repeat the post-only selector. IDs stay pinned, but current Meta settings are re-read and validated at launch.

Warned live duplications require `--acknowledge-warnings` after preview review (JSON/MCP: `acknowledgeWarnings: true`), or an explicit `useCreativeId: true` choice.

Duplicator preview reads every source ad and selected template from Meta, without creating Meta objects. Sources and destinations must belong to the selected account; `--adset` must belong to `--campaign`, including when used as a new-set template. Choose one of `--new-adset` or `--new-adset-per-ad` (JSON: `newAdSet` or `newAdSetPerAd`). With one source, per-ad mode produces one shared set and uses index `1`. Ads default to active; `--paused` pauses ads only, while newly created ad sets remain active.

`ads upload` auto-detects HTTPS URLs, including public Google Drive file links. Local paths are uploaded first; URLs are then downloaded and processed server-side into the same batch ID, so variant and thumbnail grouping still spans every input. URL and Drive-folder imports run as background jobs with bounded parallel file pipelines. While polling, the CLI shows a completed/total counter plus every file currently downloading, staging, or processing (mirrored to stderr as a one-line counter in `--json` mode). Press Ctrl-C once to stop the server-side import and wait briefly for its completed/failed/skipped summary; press Ctrl-C again to exit immediately. The summary also notes how many `*_thumbnail` images were attached to their videos as custom thumbnails instead of being counted as separate media. Drive folders must be shared as **Anyone with the link (Viewer)**.

## Creating and Updating Saved Builds

Create a durable build from an agent-produced spec. The confirmation prints a web link to open for review:

```bash
ads builds:create --account act_123 --name "Agent draft" --spec spec.json
# Open the printed Web URL and review the build in the uploader.
```

The account can be omitted when `spec.accountId` or `webState.selectedAccount` identifies it. Spec-only creation is preferred for builds created outside the uploader. Use `--web-state` only to round-trip an uploader snapshot you already read, and use `--json` to print the raw created build.

Inspect a complete saved build as JSON, then use a revision-guarded merge patch for ordinary content edits:

```bash
ads builds <id> --json > build.json
# If build.revision is 3, this changes one entry's headline and preserves its URLs/CTA.
ads builds:update <id> --expected-revision 3 --spec-patch patch.json
```

`--spec-patch` recursively merges objects, replaces arrays only when supplied, preserves omitted fields, and removes a field when its patch value is `null`. It requires `--expected-revision` so a stale editor cannot overwrite newer work. `--spec` remains available for deliberate whole-spec rewrites. For content changes, update spec only; do not construct web state by hand. The web uploader regenerates its session snapshot on the next resume. Use `--web-state` only to round-trip a snapshot previously read from a build without changing its structure. Pass `-` instead of a file path to read one JSON object from stdin.

Use Meta CTA enums such as `SHOP_NOW`. Standard-ad build writes can normalize human labels such as `Shop Now`; imported partnership specs require the enum.

Rename a build without replacing its state:

```bash
ads builds:update <id> --name "Ready for review"
```

Naming a build promotes an autosave to a kept build. Use `--json` to print the updated build instead of the human-readable confirmation.

Test Mode also submits the assembled campaign, ad-set and creative requests to Meta with `execution_options: ["validate_only"]`. The `metaValidation` result reports each call as passed, failed or not checked. Calls requiring simulated parent or creative IDs are not sent; timeout or rate-limit failures are not treated as validation passes. No Meta objects are created. A successful validation does not confirm delivery, policy approval or placement appearance.
