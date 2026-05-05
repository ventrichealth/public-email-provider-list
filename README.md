# public-email-provider-list

A curated, automatically synced list of public email provider domains (Gmail, Yahoo, AOL, etc.) served via GitHub Pages. Intended for use in authentication flows to block or flag registrations from public email providers.

## Contents

- [`public-email-provider-list.json`](./public-email-provider-list.json) — JSON array of public email provider domains, one domain per element
- [`.github/workflows/publish.yml`](./.github/workflows/publish.yml) — Publishes the blocklist to GitHub Pages on push to `dev`, `staging`, or `prod`
- [`.github/workflows/sync.yml`](./.github/workflows/sync.yml) — Manually triggered workflow to sync the upstream domain list and open a PR into `dev`

## Endpoints

The blocklist is served via GitHub Pages at the following URLs, one per environment:

| Environment | URL |
|---|---|
| dev | `https://ventrichealth.github.io/public-email-provider-list/dev/public-email-provider-list.json` |
| staging | `https://ventrichealth.github.io/public-email-provider-list/staging/public-email-provider-list.json` |
| prod | `https://ventrichealth.github.io/public-email-provider-list/prod/public-email-provider-list.json` |

## Branch Structure

| Branch | Purpose |
|---|---|
| `dev` | Source of truth for development. All changes start here. |
| `staging` | Promoted from `dev` via pull request. |
| `prod` | Promoted from `staging` via pull request. |
| `gh-pages` | Auto-published output only. Never edited manually. |

Changes flow in one direction: `dev` → `staging` → `prod`. Direct pushes to `staging`, `prod`, and `gh-pages` are blocked by branch rulesets.

## Syncing the Upstream List

The domain list is sourced from the following upstream gist:

> [ammarshah/all_email_provider_domains.txt](https://gist.github.com/ammarshah/f5c2624d767f91a7cbdc4e54db8dd0bf)

To check for updates and open a PR into `dev`:

1. Go to **Actions** in this repository
2. Select the **Sync Public Email Provider List** workflow
3. Click **Run workflow**

If the upstream list differs from the current `public-email-provider-list.json`, the workflow will automatically create a `source-update` branch and open a pull request into `dev`. If the lists are identical, the workflow exits silently with no PR created.

## Promoting Changes

To promote changes between environments, open a pull request between branches:

- `dev` → `staging` to promote to staging
- `staging` → `prod` to promote to production

Branch protection rules require at least one approving review before merging. PRs to `prod` must originate from `staging`.

## Consuming the List

The JSON file is a flat array of lowercase domain strings:

```json
["gmail.com", "yahoo.com", "aol.com", "hotmail.com", ...]
```

### Example: Auth0 Pre-Registration Action

```javascript
const axios = require("axios");

const BLOCKLIST_URL = "https://ventrichealth.github.io/public-email-provider-list/prod/public-email-provider-list.json";

let cachedBlocklist = null;
let cacheExpiry = 0;
const CACHE_TTL_MS = 5 * 60 * 1000; // 5 minutes

exports.onExecutePreUserRegistration = async (event, api) => {
  const email = event.user.email?.toLowerCase() ?? "";
  const domain = email.split("@")[1];

  if (!domain) {
    api.access.deny("invalid_email", "Email address is invalid.");
    return;
  }

  try {
    if (!cachedBlocklist || Date.now() > cacheExpiry) {
      const response = await axios.get(BLOCKLIST_URL, { timeout: 3000 });
      cachedBlocklist = new Set(response.data.map(d => d.toLowerCase()));
      cacheExpiry = Date.now() + CACHE_TTL_MS;
    }

    if (cachedBlocklist.has(domain)) {
      api.access.deny("public_email_not_allowed", "Registration with public email providers is not permitted.");
    }
  } catch (err) {
    // Fail open -- allow registration if blocklist is unreachable
    console.log("Blocklist fetch failed:", err.message);
  }
};
```

The module-level cache ensures the list is only fetched once per Action runtime instance, minimizing latency and external requests.

## Repository Setup

For initial setup instructions covering branch creation, GitHub Pages configuration, deploy key generation, and workflow permissions, see the internal setup documentation or contact the repository owner.

## Data Source

Domain list sourced from [ammarshah/f5c2624d767f91a7cbdc4e54db8dd0bf](https://gist.github.com/ammarshah/f5c2624d767f91a7cbdc4e54db8dd0bf), used under the terms of the original gist.