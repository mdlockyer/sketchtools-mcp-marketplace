# sketchtools-mcp

`sketchtools-mcp` is a model-friendly MCP bridge for Sketch with a clear, library-first workflow. It lets Claude and Codex search, preview, place, configure, and visually verify components using Sketch's bundled `sketchtool`.

## Requirements

- macOS 14 or later
- Sketch with **Settings > General > Allow AI tools to interact with open documents** enabled
- Swift 6 to build from source

## Install from the marketplace

For Claude Code or Cowork, add this repository as a marketplace and install the plugin:

```text
/plugin marketplace add mdlockyer/sketchtools-mcp-marketplace
/plugin install sketchtools-mcp@sketchtools-mcp
```

For Codex, use the repository marketplace through the CLI:

```sh
codex plugin marketplace add mdlockyer/sketchtools-mcp-marketplace
codex plugin add sketchtools-mcp@sketchtools-mcp
```

Both marketplaces expose the same skill and MCP server name.

## Build

```sh
make build          # debug executable
make test           # Swift tests
make smoke          # MCP initialize and tools/list
make package        # Claude and Codex artifacts
make verify         # tests, smoke, packaging, and validation
make marketplace    # package plus Claude and Codex native marketplace checks
make help           # all supported targets
```

The executable product is `sketchtools-mcp`. Distribution builds produce:

```text
Build/Claude/sketchtools-mcp.plugin
Build/Claude/sketchtools-mcp/
Build/Codex/sketchtools-mcp/
```

The Claude marketplace points to its directory, while the `.plugin` archive is available for direct Cowork upload. `Build/` is local generated output and is not committed to this source repository. Packaging includes arm64 and x86_64 binaries by default.

## Release

The version in `VERSION` is the release source of truth. A matching tag, such as `v1.2.0`, starts the GitHub Actions release workflow. The workflow tests the package, imports a Developer ID certificate into a temporary Keychain, signs and notarizes both native binaries, validates both marketplaces, and publishes the generated files to [`mdlockyer/sketchtools-mcp-marketplace`](https://github.com/mdlockyer/sketchtools-mcp-marketplace).

Publication uses a GitHub App installed only on the marketplace repository. Signing and publication credentials live in the `marketplace-production` GitHub environment and are available only to the release job. The workflow expects these environment settings:

```text
Variables
  MARKETPLACE_APP_ID
  APPLE_NOTARY_KEY_ID
  APPLE_NOTARY_ISSUER_ID

Secrets
  MARKETPLACE_APP_PRIVATE_KEY
  APPLE_DEVELOPER_ID_P12_BASE64
  APPLE_DEVELOPER_ID_P12_PASSWORD
  APPLE_NOTARY_KEY_P8
```

Use `make verify` before tagging. The release workflow rejects a tag that does not exactly match `VERSION` or point to the checked-out commit.

## Use

Open Sketch and the target document, then ask the model to use `sketchtools-mcp`. For library work, the companion skill guides the model through exact library selection, asset search, non-importing visual previews, symbol placement, typed overrides, linked styles and swatches, and final captures.

## License

MIT
