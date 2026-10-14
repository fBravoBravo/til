# Migrating Company Docs to Mintlify with a Custom Domain

## The Context

Our company docs were scattered across Notion pages, Confluence wikis, and a legacy static site. Onboarding new hires and external consumers meant bouncing between three sources of truth. Today I moved everything into a single Mintlify-powered documentation site and pointed our domain at it.

## The Approach

### 1. Repo Bootstrap & Structure

Initialized the Mintlify docs repo using the starter template:

```bash
npx create-mintlify@latest
```

Then added our content under `content/` and `snippets/`, mirroring our product surface area:

```
content/
├── introduction.mdx
├── getting-started/
│   ├── quickstart.mdx
│   └── authentication.mdx
├── api-reference/
│   └── ...
└── ...
```

### 2. Navigation & Information Architecture

Configured `docs.json` (v2) to flatten the navigation so the most critical paths are reachable in two clicks:

```json
{
  "navigation": {
    "tabs": [
      { "tab": "Guides", "pages": ["..."] },
      { "tab": "API Reference", "pages": ["..."] }
    ]
  }
}
```

I also rewrote page frontmatter to set `title` and `description` explicitly, which Mintlify uses for both breadcrumbs and SEO meta tags.

### 3. SEO & Social Previews

- Set a global `seo.description` and `seo.titleTemplate` in `docs.json`.
- Added a `logo` and `favicon` under the `appearance` block.
- Verified Open Graph tags rendered correctly using the Mintlify preview deploys.

### 4. Custom Domain Cutover

In Mintlify Dashboard → Settings → Domain:
- Added `docs.<ourdomain>.com`.
- Updated DNS: created a `CNAME` record pointing to Mintlify's host.
- Waited for the TLS certificate to provision (took ~2 minutes), then flipped the DNS TTL down.

## Lessons Learned

- **Content before config:** Write the `.mdx` pages first, then wire navigation. It's much faster to move files around after you see the shape of your content than to keep editing JSON in parallel.
- **DNS propagation is the long pole:** If you're cutting over a live domain, lower the TTL at least an hour before the change. The Mintlify dashboard validates the CNAME instantly, but users will bounce if their resolvers still cache the old record.
- **Groups vs. tabs:** Tabs work well when your audiences are distinct (e.g., "Guides" vs. "API Reference"). Nested groups inside a tab are better for sequential content like onboarding.
- **Snippets are underrated:** We had repeated auth examples across 20+ pages. Extracting them into `snippets/` and using `<include>` cut the word count and made updates trivial.
