# Docs Audit — 2026-04-17

## Goal

Bring `~/dev/docs` fully in sync with the current state of all four ZeroSettle SDKs (iOS 1.1.5, Android 0.15.0, Flutter 1.0.0, React Native), the backend public API surface, and the new custom checkout domain feature. Eliminate references to deprecated symbols, fill missing coverage of recent backend additions, and produce a single coherent commit.

## Scope

In scope:
- iOS SDK (`~/dev/zerosettle/ZeroSettleKit`) — current 1.1.5
- Android SDK — current 0.15.0
- Flutter SDK — current 1.0.0
- React Native SDK
- Backend (`~/dev/zerosettle-site/backend`) public surface only: `/v1/iap/*`, `/v1/checkout/*`, `/v1/webhooks/*`
- Custom checkout domain feature (model, middleware, dashboard surface, setup flow)

Out of scope:
- Internal/admin/dashboard endpoints (`/v1/platform/*`, `/v1/apps/<id>/*`)
- Legacy debug routes
- Subjective writing-style improvements (only objective drift/gaps)

## Approach

Three subagents run in parallel, each scoped to one axis. After all return, the main session synthesizes findings, applies mechanical fixes inline, and flags judgment calls for user decision.

```
              Main session
                   │ spawn parallel
   ┌───────────────┼───────────────┐
   ▼               ▼               ▼
Agent A        Agent B        Agent C
SDK deprec.   Backend        Custom
& version     public surface domain
   │               │               │
   └───────┬───────┴───────────────┘
           ▼ findings
   Synthesize → audit report
   Apply mechanical fixes
   Flag judgment calls
           │
           ▼ user approves diff
     Single commit
```

## Agents

### Agent A — SDK deprecation & version alignment

Inputs:
- `~/dev/zerosettle/ZeroSettleKit` (iOS, current 1.1.5)
- `~/dev/zerosettle/ZeroSettle-Android` (current 0.15.0)
- `~/dev/zerosettle/ZeroSettle-Flutter` (current 1.0.0)
- `~/dev/zerosettle/ZeroSettle-ReactNative` (current TBD — agent reads `package.json`)
- `~/dev/docs`

Tasks:
1. For each SDK, walk source and extract every deprecated symbol:
   - Swift: `@available(*, deprecated, ...)` markers
   - Kotlin: `@Deprecated(...)` annotations
   - Dart: `@Deprecated(...)` annotations
   - TypeScript/JS: JSDoc `@deprecated` tags
2. Build a lookup table: `{symbol → replacement → SDK version when deprecated → message}`.
3. Grep `~/dev/docs` for each deprecated symbol. For every match, classify:
   - `clean` — already migrated or has explicit migration callout
   - `stale` — uses deprecated symbol with no callout
   - `migration_doc` — appears intentionally in a migration guide (acceptable)
4. Verify install/quickstart pages show the current version for each SDK.
5. Verify each `releases/*.mdx` "Current version" matches the changelog.

Returns: `[{file, line, deprecated_symbol, replacement, severity}]`

### Agent B — Backend public-surface coverage

Inputs:
- Path: `~/dev/zerosettle-site/backend`
- Baseline commit: `75b8b0a` (previous docs audit)
- Public-surface filter: paths matching `/v1/iap/*`, `/v1/checkout/*`, `/v1/webhooks/*`
- Path: `~/dev/docs`

Tasks:
1. `git diff 75b8b0a..HEAD --name-only` filtered to URL conf files and view modules.
2. Parse `urls*.py` to enumerate all public-surface routes registered since baseline. Resolve each to its view function and extract:
   - Path + HTTP method
   - Request body shape (from view code or serializer)
   - Response shape
   - Behavior summary (from docstring)
3. For each endpoint, grep docs for the path and the view function name. Classify:
   - `documented` — has a dedicated section or clear coverage
   - `partial` — mentioned but lacks request/response/example
   - `missing` — no reference at all
4. Explicit coverage spot-checks for known recent additions:
   - `claim-entitlement` endpoint
   - `payment-intents/batch` endpoint
   - ASSN routing fix (just landed today — should be reflected in any "how transactions get attributed" docs)
   - Trial fields (`trial_type`, `trial_end`, `pending_amount`)
   - Stripe customer ID overrides

Returns: `[{endpoint, method, status, doc_file_if_present, suggested_section, severity}]`

### Agent C — Custom checkout domain coverage

Inputs:
- Path: `~/dev/zerosettle-site/backend`
- Path: `~/dev/docs`

Tasks:
1. Pull the `CustomDomain` model + migrations + middleware + management commands. Extract:
   - Setup flow (CNAME target, verification, status states/transitions)
   - Public endpoints involved
   - Dashboard surface (where it appears in settings UI)
2. Grep docs for `custom domain`, `CNAME`, `CustomDomain`. Note current state.
3. Identify cross-cutting docs that should reference custom domains:
   - `checkout-branding.mdx`
   - `account-setup.mdx`
   - `go-live.mdx`
4. Recommend whether a new dedicated `custom-domains.mdx` page is needed, or whether existing pages can absorb the content.

Returns: `[{component, current_doc_state, gap, suggested_doc_path, severity}]`

## Synthesis

After all three agents return:

1. Merge findings into single audit report at `docs/superpowers/specs/2026-04-17-docs-audit-report.md` with three categories:
   - **Critical** — wrong info that misleads developers (broken examples, removed APIs documented as current)
   - **Important** — stale/missing coverage of shipped features
   - **Minor** — polish (broken internal links, version-string typos)
2. Apply mechanical fixes inline:
   - Version bumps in `installation.mdx`, `releases/*.mdx`
   - Dead-symbol replacements (use the replacement from each `@deprecated` marker)
   - Broken-link repairs
   - Additive sections for missing endpoint coverage
3. Flag judgment calls in the report for user decision:
   - Whether to publicly deprecate symbols not yet flagged in docs
   - Whether new doc pages need diagrams/screenshots
   - Whether ASSN routing fix is worth a public callout

## Approval gate

Before committing:
1. Show user the audit report
2. Show user the diff of mechanical fixes
3. Get explicit decisions on flagged judgment calls
4. Apply those decisions
5. Commit as one descriptive change: `docs: audit and update for SDK 1.1.5, backend public surface, custom domains`

## Error handling

If any agent fails or returns ambiguous data:
- Flag the failure in the report
- Continue with the other axes
- Never silently skip; user must see what could not be audited

## Out-of-scope cleanup we will NOT do

- Restructuring the docs information architecture
- Adding new conceptual guides not tied to a shipped feature
- Subjective tone/voice improvements
- API reference page rewrites unless drift is found
