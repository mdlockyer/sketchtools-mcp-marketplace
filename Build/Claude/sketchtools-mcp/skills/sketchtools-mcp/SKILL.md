---
name: sketchtools-mcp
description: >
  Inspect and edit open Sketch documents with the bundled sketchtools-mcp
  server. Use it to search and preview library components, place symbols, edit
  typed overrides, apply linked styles and swatches, capture visual references,
  export assets, or implement a Sketch design in a codebase.
compatibility: Requires macOS with Sketch installed and an agent runtime that loads the bundled `sketchtools-mcp` MCP server. The server runs host-side and drives the local Sketch installation through Sketch's bundled sketchtool.
metadata:
  short-description: Inspect and edit Sketch through a direct MCP bridge
---

# sketchtools-mcp

## Operating posture

Use this skill to translate a live Sketch selection into production-ready UI while preserving visual fidelity and the target codebase's conventions.

Treat Sketch as the visual source of truth. Do not invent layout, icons, images, typography, or colors when `sketchtools-mcp` can provide them. If the server is unavailable or reports Sketch is not running, stop and ask the user to fix the connection rather than guessing from memory.

## Before starting

Confirm the required context:

1. Sketch is open with the target document loaded.
2. Sketch can be driven by the bridge: **Sketch → Settings → General → Allow AI tools to interact with open documents** must be on (this is what lets the bridge's sketchtool scripts reach the open document).
3. The active agent runtime has loaded the bundled `sketchtools-mcp` server and exposes its tools.
4. The user has selected the intended frame, artboard, component, or layer, unless they supplied a Sketch share URL or exact layer identifier.

Claude and Codex use platform-specific MCP configuration files because they expand different plugin-root variables. Both configurations expose the same server name and executable:

```json
{
  "mcpServers": {
    "sketchtools-mcp": {
      "command": "<plugin-root>/bin/sketchtools-mcp"
    }
  }
}
```

The source repository uses `.mcp.claude.json` with `${CLAUDE_PLUGIN_ROOT}` and `.mcp.codex.json` with `${__dirname}`. The packaged launcher selects the arm64 or x86_64 binary for the current Mac. If `health` cannot find Sketch or the working document, stop and ask the user to open Sketch rather than guessing design data.

## Tool surface (sketchtools-mcp)

The bridge exposes a fixed tool set. **Pin the working document first**: call `inspect_document` before any other tool. Every other tool targets the pinned document by explicit layer ID, so the user changing GUI focus never misroutes an edit. If a tool reports the working document is gone, re-pin with `inspect_document`.

For reusable design assets, keep the returned references intact and pass them between tools. A `library_ref` identifies a particular library by name/id/path; an `asset_ref` identifies a symbol, text style, layer style, or swatch together with its source. Do not reduce either object to a bare ID: library document IDs are not globally unique, including across Apple's official UI Kits.

- `health` — Sketch reachable? Reports the exact bundled `sketchtool` path/version, Sketch application version, JavaScript API version, and all open documents.
- `inspect_document(path?)` — pin + pages, top-level frames, and a compact library list. **Call first.** Omit `path` to pin the frontmost document.
- `get_libraries` — detailed library discovery. Returns exact `library_ref` objects, enabled/valid/source state, resolved cache paths, and counts for symbols, text styles, layer styles, and swatches. Use a returned `library_ref` to scope subsequent searches.
- `inspect_layer(id, depth?)` — hierarchy + styles + typography +, for symbols, instance overrides. Gradient-safe.
- `inspect_symbol(id|name, depth?, summary?)` — read-only symbol introspection. With an instance `id`, returns its master's tree and a compact override summary. With a `name`, inspects local document masters only; a library match is reported without importing it. For library symbols, preview first, place an instance, then call `get_symbol_overrides`.
- `get_symbol_overrides(instance_id, limit?)` — full typed override contract for an existing instance, grouped by affected component. Returns stable override IDs, affected layers, property/value types, current/default values, and editability/default/selected state. Use these IDs with `set_overrides`.
- `capture(id, format?, scale?, delay_ms?)` — image by explicit ID; no selection dependency. Pass `delay_ms` to wait before export (useful after mutations to let Sketch's renderer settle).
- `capture_grid(id, detail?, scales?, tiles?, tile_scale?, target_tile_px?, max_images?, delay_ms?)` — verification pack across two axes. `detail` picks a preset: 1 = overview (scale 1 only), 2 = standard (scales 1,2,3), 3 = fine detail (scale 1 overview + auto high-res tiles). Explicit `scales`/`tiles` override the preset. `tiles` (`'auto'` or `{cols,rows,overlap_px}`) exports one high-res pass and slices overlapping crops so fine detail (text, 1px lines, glyphs) stays crisp. Returns a manifest text block (`items[]` in image order, each with `scale` and `tile.bounds_px`) followed by the images. **Prefer this over repeated `capture` calls in the validate step.**
- `export(id, output_dir, format?, scale?)` — write an asset to disk; returns the path.
- `design_context(id)` — `inspect_layer` + `capture` in one call (the "give me everything to implement this" call).
- `find_layers(query, type?, page?)` — search document layers by name/type/text.
- `find_symbols(query, library?, limit?, offset?)` — compatibility symbol search across the document and libraries. Results are relevance-sorted, paginated, and include a canonical `asset_ref` plus the legacy ID fields.
- `get_design_assets(kinds?, query?, library_ref?, include_document?, limit?, offset?)` — preferred library search. One paginated search surface for symbols, text styles, layer styles, and swatches. Results include hierarchy paths and canonical `asset_ref` objects that feed directly into preview, placement, or application tools.
- `create_batch(parent_id, layers)` — **declarative build.** `layers` is an ordered array of specs: `{type:'rect'|'text'|'frame', name, x, y, w, h, fill, radius, border_color, border_width, shadow_color, shadow_y, shadow_blur, text, font_size, color, weight, alignment, line_height, parent}`. `parent` may be an absolute layer id or `@N` to nest under the Nth layer created in this batch (0-based). Later specs stack on top. Text specs pin their width. Returns ordered `{index,name,id}`. Prefer this over many `run_script` calls when composing a region.
- `create_frame` / `create_text` / `set_style` / `set_frame` / `delete_layer` — single-layer edits. Text layers created via `create_text`/`create_batch` are width-pinned (`fixedWidth`), so later text edits wrap instead of widening the frame. Mutation tools return the layer's name and type for confirmation; `set_style` returns post-mutation property values; `set_frame` returns the resolved frame.
- `duplicate_layer(id)` — duplicate a layer; returns the new layer's ID, name, type, and frame.
- `get_effective_frame(id)` — return a layer's absolute frame in document coordinates by accumulating parent transforms. Returns both `absolute` (document-space) and `local` (parent-relative) frames.
- `create_symbol_instance(parent_id, asset_ref?, reference_id?, library_ref?, master_id?, master_name?, x?, y?, width?, height?, overrides?, layout?)` — place a local or library symbol. Prefer the exact `asset_ref` returned by `get_design_assets`; qualified `reference_id` and legacy local/name selectors remain supported. Smart Layout reflows after overrides by default; pass `layout:'exact'` only when preserving the supplied frame is intentional.
- `set_overrides(instance_id, overrides, layout?)` — set or reset overrides on an existing instance. Use `{id,value}` to set and `{id,operation:'reset'}` to restore the library default. Property-only fallback is accepted only when unique. Smart Layout reflows by default and the result includes the resolved frame.
- `apply_design_asset(layer_id, asset_ref, target?, index?)` — apply a linked document/library text style, layer style, or swatch to a layer. Swatch targets are `text_color`, `fill`, or `border`; `index` selects the fill/border slot. This preserves the library link instead of copying raw style values.
- `list_families(page?)` — list font families used in the document (or scoped to a page). Returns families with their sizes and weights, sourced from both text layers and shared text styles.
- `search_glyphs(query)` — search SF Symbols glyph names and return their Unicode PUA code points. Returns the hex code point and the actual unicode character. Useful for embedding SF Symbols in Sketch text layers.
- `get_thumbnails(asset_refs?, query?, symbol_ids?, library?, library_paths?, scale?, max?, offset?)` — visually preview symbols without importing them. Prefer the `asset_refs` returned by `get_design_assets`; legacy query/ID lookup remains supported. Returns a paginated manifest followed by transparent PNG blocks in the same order. Page size defaults to 20 and is capped at 40; use `next_offset` instead of widening the request. Library-only previews also work when the pinned document has not been saved.
- `run_script(code, transactional?)` — escape hatch. You may use the conventional `const sketch = require('sketch')`; the bridge isolates that module binding from sketchtool's reserved global `sketch` context name. Pass `transactional: true` to wrap in a transaction that rolls back on error. **Do not write computed member access whose index contains a bracket** (`out[cfg[1]]`): the wrapper rewrites `name[` and throws a cryptic syntax error. Compute the index first: `var idx = cfg[1]; out[idx] = ...`. Plain string/number indices (`out['k']`, `out[0]`, `a['x']['y']`) are fine.
- `edit_offline_open(path?)` / `edit_offline_commit(reopen?)` — **the supported path for bulk mutation.** `edit_offline_open` saves + closes the document in Sketch, unzips the `.sketch` package to a working directory, and returns a page-file manifest. Edit `pages/*.json` on disk with normal file tools, then `edit_offline_commit` validates the JSON, repackages the `.sketch` atomically, and reopens it. See **Bulk mutation** below.

### Bulk mutation: use the offline path, not the scripting tools

Every scripting/mutation tool (`run_script`, `create_batch`, `set_style`, `set_overrides`, `create_*`, and even `capture`) reaches Sketch through `sketchtool`, which runs **synchronously on Sketch's main thread**. One call at a time; while it runs, Sketch cannot event-loop. On a large document (many artboards instancing heavy library symbols), each mutation invalidates layout, so a single call with hundreds of mutations can monopolize the main thread for **minutes** — the app beachballs, and any probe or `capture` you send queues *behind* it rather than measuring it. `capture`/`capture_grid` additionally force an export render, so captures stacked behind a mutation burst multiply the stall.

Rules:

- **A handful of interactive edits** (single frame, a few layers): the scripting tools are fine.
- **Hundreds of mutations** (theme propagation, re-spacing every frame, a component rebuild across artboards): do **not** loop the scripting tools. Use `edit_offline_open` → edit `pages/*.json` → `edit_offline_commit`. The identical changes apply in under a second with no UI stall. Common edit points in the JSON: artboard/layer `frame` (x/y/width/height), corner radii on `points[].cornerRadius`, font descriptors and RGBA colors on the attributed string, `paragraphStyle` (alignment/line-height), and `textBehaviour` (`1` = fixed width, so API-authored text does not re-anchor on the next layout pass).
- **`run_script`'s ~60s timeout is client-side only.** If a bulk script exceeds it, the bridge kills the *sketchtool* process, but the work already handed to Sketch keeps running server-side and still saves — a "timeout" here does **not** mean the edit failed, and re-sending it or probing with `health` only adds more main-thread work. If you hit this, stop sending calls and switch to the offline path.

Composition notes: current Sketch Frames take corner radii through `style.corners`; only group/frame layers accept children, so nesting targets must be `frame`/group, not `rect`. The bridge reads live state, so after a mutation re-`capture` or re-`inspect_layer` to verify — there is no selection/visibility desync to work around. For visual verification, prefer `capture_grid`: a single call returns the multi-scale set and/or a high-res tile grid, with a manifest mapping each image to its scale and exact pixel bounds.

## Viewing captures in the agent runtime

The tool the runtime uses to show you an image often has a quality / resolution parameter, and its **default is tuned for general viewing, not for reading fine UI text** — so never assume the default is right when you need to *read* pixels (labels, 1px separators, small glyphs). Pass the highest-fidelity option the viewer offers whenever you are verifying, and keep the default only for "is this the right region at all?" glances. This holds in any runtime; the parameter name and the size of the default's loss vary by harness and by model, but the principle does not.

**Runtime specifics.**

- **Cowork / Claude Desktop.** Images are delivered through the harness's normal image view. If you need to read fine text, prefer `capture_grid` tiles — a small region passes through at near 1:1, which is the reliable way to get small text above the model's readable floor — and use the highest-fidelity viewer setting the harness offers for verification.
- **Codex harness.** The viewer is `view_image`, whose `detail` parameter defaults to `high` and offers `original` to preserve exact resolution. `high` resamples the image before it reaches the model, and on large captures that resample discards fine text — measured directly: on a ~3700px-wide window, `high` reduced the inspector's key/value rows and recent-backup lines to unreadable bars that `detail: "original"` restored to legible text from the identical file. So for any verification that depends on reading text or thin lines, call `view_image` with `detail: "original"`. The loss scales with image size, which is why whole-window captures are where `high` hurts most and small crops are where it barely matters.

**Optimal setup, and why.** Pair the highest-fidelity viewer setting with a high-resolution source, but expect diminishing returns on whole-image upscaling: the delivery pipeline preserves aspect (verified — circles stay round and squares stay square at any image ratio, so proportions are never distorted) yet applies a long-edge cap, so scaling a *whole* image past roughly 1.5–2k on the long edge mostly feeds pixels to that cap instead of to the model. The reliable way to get small text above the model's readable floor is therefore to deliver a *small region* as its own image — a tile or crop — which is exactly what `capture_grid`'s tile axis does. Keep tiles ≲ ~1100–1300px on the long edge so they pass through near 1:1; `tile_scale` 2 suffices for normal 11–13px UI text, 3 for fine print. Do **not** pad exports to squares to "help" the model — aspect is already preserved, so padding changes nothing about delivered text size and only wastes pixels.

**Model-specific number — re-derive, don't trust.** At the time of writing, the active model in this harness resolves roughly a **14–16px delivered line-height** (cap height ≈10–11px) for medium black-on-white text; that is the floor a tile's text must clear. This figure is a property of *this* vision system and may differ for another model, so if the model changes, re-measure it with a text-size ladder (one line per font size on a flat field, viewed at the highest fidelity) before relying on it — and stop at the point where further precision would require blind, externally-generated test data, because past that the model's own context biases the read.

## Library-first component workflow

For work built from a Sketch library, use this short path. It avoids hidden imports, ambiguous names, giant result sets, and manual ID translation:

1. `inspect_document` to pin the destination document.
2. `get_libraries` to choose the exact library and retain its complete `library_ref`.
3. `get_design_assets` with that `library_ref`, a narrow `query`, and the needed `kinds`. Keep each result's complete `asset_ref`.
4. For symbol candidates, pass those `asset_ref` objects to `get_thumbnails` and visually select from the returned images. Paginate with `next_offset` when needed.
5. Pass the selected symbol `asset_ref` directly to `create_symbol_instance`. Do not look it up again by name.
6. Call `get_symbol_overrides` on the created instance, then use stable IDs with `set_overrides`. Leave `layout:'smart'` unless the design requires exact dimensions.
7. Apply reusable text styles, layer styles, and swatches with `apply_design_asset`; do not replace linked assets with copied raw values.
8. `capture` the instance during composition and use `capture_grid` for final visual verification.

The intended handoff is therefore `library_ref` → `asset_ref` → preview/place/apply. If a call returns `AMBIGUOUS_LIBRARY_ID` or `AMBIGUOUS_SYMBOL`, choose from the returned candidates and retry with the full qualified reference; do not guess.

## Required workflow

Follow the steps in order. All tool calls go through the `sketchtools-mcp` server surface above.

### 1. Pin the document and identify the target

Start with `inspect_document` — this pins the working document for every later call. Omit `path` to pin the frontmost document; pass a path to pick a specific open document when more than one is open.

Then identify the implementation target, in priority order:

1. **User supplied a Sketch share link** such as `https://sketch.com/s/<document-share-uuid>/f/<canvas-frame-uuid>`: extract the `/f/<canvas-frame-uuid>` value and locate the frame with `run_script`:
   ```js
   const sketch = require('sketch')
   const frameId = '4A2E31FF-56BD-4C29-92D2-829548D19C1D'
   const frame = sketch.find('#' + frameId, sketch.getSelectedDocument())[0]
   console.log(JSON.stringify(frame ? { ok: true, id: frame.id, name: frame.name } : { ok: false, error: 'FRAME_NOT_FOUND' }))
   ```
2. **User gave layer identifiers or names**: use `find_layers(query)` to locate matching layers. When multiple matches exist, ask the user which one to implement.
3. **User selected a layer in Sketch**: read the current selection with `run_script`:
   ```js
   const sketch = require('sketch')
   const sel = sketch.getSelectedDocument().selectedLayers.layers
   console.log(JSON.stringify(sel.map(l => ({ id: l.id, name: l.name, type: String(l.type) }))))
   ```

If no target can be identified, ask the user to select the intended Sketch layer or provide a layer ID.

When multiple documents are open, enumerate them with `health` (returns paths, ids, and page counts) and pin the correct one by path via `inspect_document(path)`. Unsaved documents have no path — make them frontmost and omit `path`; in-process tools still target them by ID, but file-based exports (raw `sketchtool`, `get_thumbnails` local symbols) need a saved file.

### 2. Fetch structured design context

Use `inspect_layer(id, depth)` on the target root, plus `inspect_symbol` for symbol instances, to capture what you need for implementation:

- hierarchy: IDs, names, types, visibility, lock state
- layout: frames, resizing behavior, pins, stacks, clipping, constraints
- typography: family, size, weight, line height, alignment, decoration
- styling: fills, borders, shadows, blur, corner radii, opacity, tint
- reusable styles: shared text and layer style names, source library names
- variables and swatches: color variable names and source library names
- symbols: nested symbols and override-capable fields
- exports: export settings and image sources

If the tree is too large or tool output is truncated:

1. Fetch a shallow map first (`inspect_layer` with a small `depth`, or `find_layers` to locate the subtree).
2. Fetch detailed context only for the critical child subtrees.
3. Continue only after the visual and structural nodes needed for implementation are covered.

For a quick full handoff, call `design_context(id)` — it returns the inspection and a screenshot in one call.

### 3. Capture the visual reference

Call `capture(id)` on the target and use the returned image as the primary visual reference for parity checks. The bridge captures by explicit layer ID, so there is no selection-state dependency — if you have the ID, you can capture it. For verification later, prefer `capture_grid` (see step 7).

If `inspect_document` pinned no document or the target ID is not found, stop and ask the user to open the intended document/select the layer in Sketch.

### 4. Export required assets

Export icons, bitmaps, symbols, or other assets from Sketch instead of substituting placeholders. Rules:

- Use the bridge's `export(id, output_dir, format?, scale?)` with an absolute `output_dir`.
- Do not introduce new icon packs or stock assets unless explicitly requested.
- For raw exports that bypass the bridge entirely, `sketchtool export layers` can capture any layer by ID directly. See `references/sketchtool-cli.md`.

When composing or testing frames inside Sketch, note that `new sketch.Group.Frame(...)` is the Frame constructor (not `new sketch.Frame(...)`):

```js
const frame = new sketch.Group.Frame({
  name: 'My Frame',
  parent: page,
  frame: { x: 0, y: 0, width: 320, height: 200 }
})
```

### 5. Map design data to the target codebase

Treat Sketch data as design intent, then implement using project conventions:

- Reuse existing UI components before creating new ones.
- Map Sketch values to project tokens for color, spacing, type, radius, elevation, and motion where available.
- Preserve the project's architecture, routing, state management, naming, and data-fetching patterns.
- Keep component boundaries aligned with the design hierarchy when that does not fight the codebase.
- Prefer maintainable, idiomatic code over brittle absolute-position reproductions.

### 6. Implement for visual parity

Aim for 1:1 parity with the Sketch screenshot and structured context:

- Match spacing, alignment, sizing, and hierarchy.
- Match typography and color usage.
- Preserve intended responsive behavior and Sketch constraints.
- Render exported assets at the intended size and density.
- Maintain accessibility basics: semantic structure, readable contrast, keyboard reachability, and labels.

Prefer incremental edits over broad rewrites. Do not modify unrelated screens or components.

### 7. Validate before completion

Before saying the implementation is complete, compare the result against Sketch:

- layout and spacing
- typography
- colors, fills, borders, shadows, and effects
- states and interactions
- asset rendering
- responsiveness and constraint behavior
- accessibility basics
- intentional deviations

Call `capture_grid(id)` on the target (or `capture(id)` if the region is small) and check it against the code. If a mismatch remains, re-`inspect_layer` the affected subtree and iterate. The bridge reads live state, so post-mutation captures reflect current Sketch state. If a deviation is intentional because of technical or accessibility constraints, document the reason.

## Failure handling

### `health` cannot find Sketch, or no document is open

Stop and ask the user to launch Sketch and open the target document, then re-run `inspect_document`. Do not guess design data while Sketch is unreachable.

### A tool reports the working document is gone

The pinned document was closed or its path changed. Re-pin with `inspect_document` (pass a path if the target is not frontmost), then retry the call.

### `inspect_layer` / `capture` reports `LAYER_NOT_FOUND`

The ID may be stale or the target may live in a different document. Re-list with `inspect_document` or `find_layers` and retry with the correct ID.

### A script call fails to run

`run_script` executes inside Sketch's own JavaScript environment via sketchtool. Sketchtool reserves the global `sketch` name for an execution-context object, not the public JavaScript API module. The bridge detects a conventional `const sketch = require('sketch')` / `var` / `let` import and runs that script in an isolated function scope where `sketch` is the real module.

**What works reliably:**

- A conventional `const sketch = require('sketch')` module binding: `sketch.getSelectedDocument()`, `sketch.find(...)`, `sketch.export(...)`, `sketch.Library.getLibraries()`
- `console.log(JSON.stringify(...))` for structured output
- Standard JS: closures, array methods, try/catch, `new sketch.Group.Frame(...)`, `new sketch.Text(...)`, `new sketch.ShapePath(...)`
- `var sk = require('sketch')` if you need a fresh binding (does not collide with the pre-declared `sketch`)

**What does not work:**

- Unusual whitespace or alternate expressions in a `sketch` import. The adapter recognizes the conventional declarations shown above; use `var sk = require('sketch')` if generating a nonstandard form.
- Template literals inside sketchtool arguments — sketchtool's shell escaping can corrupt `${...}` expressions; use string concatenation instead
- Computed member access with nested brackets: `out[cfg[1]]` crashes sketchtool's parser; assign the inner expression to a variable first (`var idx = cfg[1]; out[idx]`)
- Action-context-only APIs (sketch/ui, AppController action handlers) — `run_script` runs in the plugin/script context, not the action context; use the `sketch` namespace for all operations

**Output must be valid JSON on stdout.** Use `console.log(JSON.stringify(...))` — the bridge reads stdout and will fail if the output is not parseable. Wrap the entire script body in `try/catch` and log the error as JSON if you need to handle failures gracefully.

### Sketch beachballs or a mutation call times out

You sent a bulk mutation (or a burst of mutations/captures) on a large document and Sketch stopped responding, or `run_script` returned a timeout. This is main-thread saturation, not corruption — the document on disk is intact (completed scripts save atomically). **Do not** re-send the call or probe with `health`/`capture`; each new call queues behind the still-running work and deepens the stall. Stop, let the in-flight work finish (or have the user force-quit and reopen — the last save is on disk), then apply the remaining changes via the offline path: `edit_offline_open` → edit `pages/*.json` → `edit_offline_commit`. See **Bulk mutation** above.

### `capture_grid` exceeds `max_images`

Narrow the request: fewer scales, fewer tiles, or a higher `max_images`. See the tool description for the caps.

### Assets are missing or wrong

Re-run `export` with an absolute `output_dir`. Do not replace missing assets with invented placeholders unless the user approves.

### Implementation does not match the design

Re-capture the target (`capture` or `capture_grid`) and inspect exact node values for the mismatched area before changing code.

## Output expectations

When finished, report:

1. What was implemented.
2. Which Sketch target was used (document + layer ID/name).
3. Which assets were exported or reused.
4. Any intentional deviations from Sketch.
5. What validation was performed.
