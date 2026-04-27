# CLAUDE.md - ZeroSettle Documentation

## Project Overview

This is the ZeroSettle documentation site, built with [Mintlify](https://mintlify.com/). It lives at `docs.zerosettle.io`.

## Tech Stack

- **Framework**: Mintlify (maple theme)
- **Content**: MDX files with Mintlify components
- **API Reference**: Auto-generated from OpenAPI spec at `api-reference/openapi.json`
- **Config**: `docs.json` (navigation, theme, settings)

## Directory Structure

```
docs/
├── iap/                    # In-App Purchase documentation (~28 pages)
├── api-reference/          # OpenAPI spec + endpoint MDX pages
├── images/                 # Screenshots and diagrams
├── logo/                   # Brand logos
├── docs.json               # Mintlify configuration
├── index.mdx               # Landing page
├── changelog.mdx           # SDK release history
└── global.css              # Custom theme overrides
```

## Development

```bash
# Preview locally
npx mintlify dev

# The site auto-deploys on push
```

## Content Guidelines

- Use Mintlify components: `<Steps>`, `<CodeGroup>`, `<Note>`, `<Warning>`, `<Tip>`, `<Info>`, `<Check>`, `<AccordionGroup>`, `<CardGroup>`, `<Card>`, `<Update>`
- Multi-platform code examples (Swift/Kotlin/Flutter) on every SDK page
- Keep the OpenAPI spec (`api-reference/openapi.json`) in sync with the backend's `/v1/iap/*` endpoints
- Reference the [JustOne sample app](/iap/sample-app) for real-world integration patterns

## Related Repositories

- **Backend**: `/Users/ryanelliott/dev/zerosettle-site/backend/` — Django API (endpoint changes require OpenAPI spec updates here)
- **iOS SDK**: `/Users/ryanelliott/dev/zerosettle/ZeroSettleKit/` — Swift Package (version bumps should be added to changelog.mdx)
- **Sample App**: `/Users/ryanelliott/dev/JustOne/` — JustOne habit tracker (referenced throughout docs)

## Backward Compatibility

The `/v1/iap/*` endpoints are the wire contract between the backend and all SDK clients. See the backend's CLAUDE.md for the full backward compatibility policy. Any endpoint changes must be reflected in:
1. `api-reference/openapi.json`
2. Relevant endpoint MDX pages in `api-reference/endpoint/`
3. `changelog.mdx` for SDK-visible changes
