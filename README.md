# sketchtools-mcp marketplace

This repository is the installable Claude and Codex distribution of [`sketchtools-mcp`](https://github.com/mdlockyer/sketchtools-mcp), a library-first MCP bridge for Sketch. GitHub Actions builds, signs, notarizes, validates, and publishes each version from a matching source tag.

## Install

For Claude Code or Cowork:

```text
/plugin marketplace add mdlockyer/sketchtools-mcp-marketplace
/plugin install sketchtools-mcp@sketchtools-mcp
```

For Codex:

```sh
codex plugin marketplace add mdlockyer/sketchtools-mcp-marketplace
codex plugin add sketchtools-mcp@sketchtools-mcp
```

The plugin, MCP server, and executable are all named `sketchtools-mcp`.

## Published files

`Build/Claude/sketchtools-mcp/` and `Build/Codex/sketchtools-mcp/` contain the installable marketplace plugins. `Build/Claude/sketchtools-mcp.plugin` is the direct Claude and Cowork upload archive.

`SOURCE.json` identifies the source tag and commit. `SHA256SUMS` covers every file in the generated plugin trees.

Do not edit generated files here. Make changes in the [source repository](https://github.com/mdlockyer/sketchtools-mcp) and publish a new version tag.

## License

MIT
