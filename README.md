# Agent Knowledge Base

Reference docs for integrations, auth flows, and API patterns.
Written from real implementation experience — not copied from docs.

Each file documents the *actual* steps that work, including gotchas.

## Index

| File | What it covers |
|------|---------------|
| [google-oauth.md](./google-oauth.md) | Manual Google OAuth flow (remote/headless environments) |
| [youtube-oauth.md](./youtube-oauth.md) | YouTube Data API v3 OAuth + token refresh |
| [gog-setup.md](./gog-setup.md) | `gog` CLI for Gmail, Calendar, Drive, Contacts, Docs, Sheets |
| [notion-api.md](./notion-api.md) | Notion internal integration + querying/updating databases |
| [runpod-api.md](./runpod-api.md) | RunPod serverless transcription + summarization endpoint |

## Principles

- **Exact commands over prose** — copy-paste should work
- **Document gotchas** — the thing that took 20 minutes to figure out belongs here
- **Keep secrets out** — use `YOUR_KEY` placeholders, never real values
