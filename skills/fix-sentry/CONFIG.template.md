# fix-sentry Configuration

Copy this file to `CONFIG.md` and fill in your project-specific values.

## Sentry Settings

| Setting | Value | Description |
|---------|-------|-------------|
| `sentry.org` | `your-org-slug` | Sentry organization slug (from Settings → General) |
| `sentry.project` | `your-project-slug` | Sentry project slug (from Settings → Projects) |
| `sentry.region_url` | `https://us.sentry.io` | Sentry region URL (`https://us.sentry.io` for US, `https://sentry.io` for EU) |

## Deploy Settings

| Setting | Value | Description |
|---------|-------|-------------|
| `deploy.provider` | `vercel` | Deploy provider: `vercel` (more providers coming) |
| `deploy.project` | `your-vercel-project` | Vercel project name (optional — auto-detected from git remote if not set) |

## How to Find Your Sentry Org Slug

1. Go to https://sentry.io → Settings → General Settings
2. The "Organization Slug" field is your `sentry.org` value
3. The project slug is visible under Settings → Projects → click your project

## Example (filled in)

```md
## Sentry Settings

| Setting | Value |
|---------|-------|
| `sentry.org` | `acme-corp` |
| `sentry.project` | `acme-web-app` |
| `sentry.region_url` | `https://us.sentry.io` |

## Deploy Settings

| Setting | Value |
|---------|-------|
| `deploy.provider` | `vercel` |
| `deploy.project` | `acme-web-app` |
```
