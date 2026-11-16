# ZeroSettle Docs

This repository powers [docs.zerosettle.io](https://docs.zerosettle.io):

- `iap/`, `api-reference/`, `ai/`, and `releases/` contain the billing guides, API reference, billing MCP docs, and SDK release notes.
- `imessage/` contains the iMessage API and FaceTime Audio guides from the [portal docs](https://portal.zerosettle.io/docs). The Markdown sources are [the iMessage reference](https://portal.zerosettle.io/docs.md) and [the FaceTime Audio guide](https://portal.zerosettle.io/docs/facetime-audio.md). While both sites run in parallel, use those sources to check for changes and keep the Mintlify pages in sync.
- `docs.json` controls the Mintlify navigation and site settings.

## Development

Run the [Mintlify CLI](https://www.npmjs.com/package/mint) from the repository root:

```sh
npx mint dev
```

View the preview at `http://localhost:3000`. Use `npx mint validate` and `npx mint broken-links` to check documentation changes.

## Publishing changes

Open a pull request for documentation changes. Mintlify deploys changes to the default branch to production automatically.
