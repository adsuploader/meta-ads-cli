# Security Policy

## Reporting a vulnerability

If you believe you have found a security vulnerability in the Ads Uploader CLI (`@adsuploader/cli`), please report it privately.

- **Email:** support@adsuploader.com (subject: `SECURITY`)
- Include steps to reproduce, affected commands, and any relevant request/response detail.
- Do not open a public issue for security reports.

We aim to acknowledge reports promptly and will keep you updated on remediation. Please give us reasonable time to investigate and fix an issue before any public disclosure.

## Scope

- The `@adsuploader/cli` npm package (`ads` command).

## Handling of credentials

- `ads login` authenticates in your browser and stores credentials locally on your machine. The CLI never asks you to paste Meta access tokens.
- Report any behavior that exposes tokens, other users' data, or another account's ads.
