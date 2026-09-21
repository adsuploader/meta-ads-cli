# Privacy

This describes what the Ads Uploader CLI (`@adsuploader/cli`) accesses. For the full company privacy policy, see https://adsuploader.com/privacy-policy.

## What it accesses

The `ads` CLI acts on your behalf against your Ads Uploader account, which connects to the Meta (Facebook/Instagram) Marketing API you have authorized. With it you can:

- Read your ad accounts, campaigns, ad sets, ads, Pages, and presets.
- Upload media you provide (local files, public URLs, or a Google Drive folder).
- Create and duplicate ads — created **paused** by default.

## Authentication

- `ads login` authenticates in your browser (OAuth) and saves credentials locally on your machine. The CLI never asks you to paste Meta access tokens, and your credentials are not sent to third parties.

## Data handling

- The CLI operates on the ad data in the accounts you authorize. It does not sell your data.
- Media you upload is staged and processed to create ads in your own Meta ad accounts.
- Revoke access at any time by logging out (`ads logout`) and/or revoking the app in your Ads Uploader and Meta account settings.

## Contact

Questions about data or privacy: support@adsuploader.com
