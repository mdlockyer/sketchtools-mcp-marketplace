# sketchtool CLI reference

`sketchtool` is Sketch's headless command-line tool. It talks directly to the Sketch process (or reads `.sketch` files) without the MCP HTTP layer, making it immune to MCP connection drops and state desync.

## Location

```sh
/Applications/Sketch.app/Contents/MacOS/sketchtool
```

## Key commands for this plugin

### run-script — execute JS inside Sketch

Equivalent to MCP `run_code` but without the HTTP server layer. Use as a fallback when MCP is unreachable or when you need reliable post-mutation verification.

```sh
sketchtool run-script '<javascript>' --without-activating
```

Important differences from MCP `run_code`:

- The wrapper pre-declares `const sketch`, so you **cannot** write `const sketch = require('sketch')`. Use a different variable name:

```sh
sketchtool run-script 'var sk = require("sketch"); var doc = sk.getSelectedDocument(); console.log(JSON.stringify({ ok: true, path: doc.path }))' --without-activating
```

- Output is returned as stdout (same as MCP `run_code`).
- Use `--context '<json>'` to pass data into the script.
- Use `--timeout <seconds>` for long operations (default: 60).
- Use `--without-activating` to keep Sketch in the background.

### export layers — export by layer ID

Exports specific layers directly from a `.sketch` file. Does not require MCP or a live selection.

```sh
sketchtool export layers <document.sketch> \
  --items=<layer-id> \
  --output=/tmp/sketch-assets \
  --formats=png \
  --overwriting=YES
```

Options:

- `--items` — comma-separated layer IDs or names
- `--scales` — export scales (e.g., `1,2,3` for @1x/@2x/@3x)
- `--formats` — `png`, `jpg`, `svg`, `pdf`, `webp`, `tiff`, `eps`
- `--trimmed` — trim transparent borders
- `--cropped` — crop to exact bounds (shadows clipped)
- `--background` — background color for transparent areas
- `--group-contents-only` — export group children individually

### export artboards — export artboards

```sh
sketchtool export artboards <document.sketch> \
  --items=<artboard-id-or-name> \
  --output=/tmp/sketch-assets \
  --formats=png
```

### export preview — quick document thumbnail

```sh
sketchtool export preview <document.sketch> --output=/tmp/preview.png
```

### metadata — document info as JSON

```sh
sketchtool metadata <document.sketch>
```

Returns app version, page/artboard IDs, fonts used, and compatibility info.

## When to prefer sketchtool over the bridge/MCP

| Scenario | Use |
|----------|-----|
| Bridge/MCP connection dropped | `sketchtool run-script` |
| Export assets after a mutation | `sketchtool export layers` (no desync) |
| Verify post-mutation state | `sketchtool run-script` (sees live state) |
| Batch export many layers | `sketchtool export layers --items=id1,id2,id3` |
| Inspect a file without opening Sketch | `sketchtool metadata` |
| Need a `capture`-equivalent | `sketchtool export layers --items=<id>` |

## What the bundled bridge covers that raw sketchtool does not

- `get_libraries` — qualified library references, current state, cache paths, and asset counts.
- `get_design_assets` / `find_symbols` — relevance-sorted, paginated library search with canonical references for symbols, text styles, layer styles, and swatches.
- `get_thumbnails` — non-importing visual symbol previews returned directly to the model.
- `inspect_symbol` / `get_symbol_overrides` — read-only master inspection and a typed override contract.
- `create_symbol_instance` / `apply_design_asset` — place or apply a searched asset while preserving its library identity and link.
- `capture_grid` — multi-scale + tiled high-res crops from one call.
- `inspect_layer` — the guarded, gradient-safe style/hierarchy walker.

Sketch's own MCP server also exposes tools named `get_libraries`, `get_design_assets`, and `get_symbol_overrides`. The bundled bridge implements those familiar names with qualified `library_ref`/`asset_ref` handoffs, pagination, non-importing previews, placement, and linked style/swatch application. Sketch's editable `get_guide` remains native-server-specific; this plugin's `SKILL.md` is the bridge guidance source.

## Caveats

- `run-script` requires Sketch to be running (it executes inside the Sketch process).
- `export layers` reads the file on disk — if the document has unsaved changes, the export reflects the last saved state, not the live in-memory state. Save first or use `run-script` with `sketch.export()` for live state.
- **Pass value flags as `--flag=value`, never space-separated.** `sketchtool export layers doc.sketch --items <id>` (space form) crashes sketchtool with an `NSInvalidArgumentException`; `--items=<id>` works. Applies to `--items`, `--output`, `--formats`, `--scales`.
- **`--items` matches reliably by layer object ID, not by name.** Name matching is inconsistent (some names match, near-identical siblings silently don't). For library symbols, the ID you get from `find_symbols` is the *symbolID*; the layer object ID (`do_objectID`) that `--items` needs lives in the library file's `pages/*.json`.
- The reserved global `sketch` name means raw `sketchtool run-script` calls must not redeclare it; use `var sk = require("sketch")`. The bridge's `run_script` adapter accepts the conventional `const sketch = require('sketch')` form by isolating the real module binding in a function scope.
