# Sketch MCP setup notes

This plugin exposes an MCP server named `sketchtools-mcp` in both Claude and Codex.

## Marketplace installation

The source lives at `mdlockyer/sketchtools-mcp`. Signed and notarized release packages are published from version tags to `mdlockyer/sketchtools-mcp-marketplace`.

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

## Default declaration (bundled bridge)

The plugin ships a native Swift stdio server, `sketchtools-mcp`, which drives Sketch through the `sketchtool` CLI. Claude and Codex use separate configuration files because their plugin-root variables differ.

Claude packages include `.mcp.claude.json`:

```json
{
  "mcpServers": {
    "sketchtools-mcp": {
      "command": "${CLAUDE_PLUGIN_ROOT}/bin/sketchtools-mcp"
    }
  }
}
```

Codex packages include `.mcp.codex.json`:

```json
{
  "mcpServers": {
    "sketchtools-mcp": {
      "command": "${__dirname}/bin/sketchtools-mcp"
    }
  }
}
```

Both variables resolve to the installed plugin directory. The stable launcher at `bin/sketchtools-mcp` selects `sketchtools-mcp-darwin-arm64` or `sketchtools-mcp-darwin-x86_64` for the current Mac.

The bridge's complete tool surface is documented in the SKILL.md. Its library-first tools are `get_libraries`, `get_design_assets`, `get_thumbnails`, `create_symbol_instance`, `get_symbol_overrides`, `set_overrides`, and `apply_design_asset`.

## Alternative: Sketch's own HTTP MCP server

Sketch ships its own local streamable HTTP MCP server. If you prefer to connect to it directly instead of the bundled bridge, configure a separate `sketch-native` server with:

```json
{
  "mcpServers": {
    "sketch-native": {
      "type": "http",
      "url": "http://localhost:31126/mcp"
    }
  }
}
```

The URL is the one shown in Sketch settings (**General → Allow AI tools to interact with open documents**). Note that direct `localhost` reachability depends on how the active desktop runtime bridges local services into the working environment; the bundled bridge avoids that question by running host-side.

## Sketch-side requirement

Whichever server you use, Sketch must be running with the target document open and **General → Allow AI tools to interact with open documents** enabled, so scripts/`run-script` calls can reach the open document.

## Expected tools

The bundled `sketchtools-mcp` server exposes the tools listed in the SKILL.md.

If you connect to Sketch's own MCP server instead, it exposes a different, native surface (verified against Sketch 2026.3, JS API 2.0.0):

| Tool | Purpose |
|------|---------|
| `run_code` | Execute Sketch JavaScript against the open document |
| `get_screenshot` | Capture the current selection or a specific layer as an image |
| `get_document_info` | Return document metadata, page list, and top-level frames |
| `get_layer_tree_summary` | Return a compact indented hierarchy of a layer subtree |
| `get_libraries` | List linked libraries with capability flags |
| `get_design_assets` | List symbols, text styles, layer styles, or swatches by library |
| `get_symbol_overrides` | List available overrides on a symbol instance |
| `get_guide` | Return editable Markdown guidance for Sketch MCP usage |

The exact names may be namespaced by the runtime under the `sketchtools-mcp` server. The bridge intentionally shares the native names `get_libraries`, `get_design_assets`, and `get_symbol_overrides`, but its schemas and results are richer and are not assumed to be wire-compatible. If you connect the native server instead, map `inspect_document` to `get_document_info`, `inspect_layer` to `run_code` plus `get_layer_tree_summary`, and `capture` to `get_screenshot` or `sketch.export`. The native server has no direct counterpart for the bridge's canonical `asset_ref` handoff, non-importing `get_thumbnails`, `create_symbol_instance`, `set_overrides`, or `apply_design_asset` workflow.

## Known limitations (native Sketch MCP server)

- `get_document_info` without a `targetDocumentID` returns only the frontmost document. Use `run_code` with `sketch.getDocuments()` to enumerate all open documents.
- New or unsaved documents may share the same document ID. Prefer path-based identification when multiple scratch files are open.
- After `run_code` mutations, `get_screenshot` and `get_layer_tree_summary` may not reflect changes immediately. Use `get_document_info` or `run_code` re-probes for post-mutation verification.
- The MCP connection may drop after a burst of calls. Toggle the MCP setting in Sketch or restart Sketch to recover.

The bundled bridge does not have these limitations: it pins a document by path (`inspect_document`) and captures by explicit layer ID, so there is no selection-state dependency or post-mutation desync to work around.
