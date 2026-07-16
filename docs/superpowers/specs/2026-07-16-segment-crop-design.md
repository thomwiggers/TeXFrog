# Segment auto-crop for per-hop game renderings

**Status:** approved design, ready for implementation plan
**Date:** 2026-07-16

## Problem

A `tfsource` block can be a large monolithic algorithm — in the IKEv2 proof
(`proof/proof.tex`) the `kind` source is ~185 lines (initiator + responder,
across MKE / PPK / IKE_AUTH phases). Each `\tfrendergame` renders the *whole*
listing with a few lines highlighted. The proof issues ~40 such renderings, so
the PDF (and the HTML viewer) repeats the full ~185-line algorithm dozens of
times even though a single game hop typically touches a handful of lines in one
phase. The output is enormous and the localized change is hard to spot.

## Goal

Let a diffed per-hop rendering show only the parts of the algorithm that
changed, while staying oriented in the overall protocol. Automatic (no per-hop
manual line selection), structurally safe (never emits unbalanced LaTeX), and
working in **both** the `pdflatex` output and the HTML viewer.

## Approach: segment auto-crop

The author marks **segment boundaries** in the source. A cropped rendering
shows only segments that contain a change (add / change / remove vs the diff
target); each contiguous run of unchanged segments collapses to a single
caption stub so the reader keeps their place in the protocol.

Segment granularity (not line granularity) is deliberate: a segment is a whole
balanced block, so dropping it can never leave a dangling `\If` without its
`\EndIf`, and in HTML it only means compiling the game's SVG from fewer input
lines — no per-line-image re-architecture. Line-level fold/context view was
considered and rejected as high-risk in the current architecture (LaTeX
structural balance of untagged `\If/\EndIf/\Statex/\LComment` lines; HTML
single-image SVG per game).

## Author-facing API

### `\tfsegment{Caption}`

Boundary marker placed inside a `tfsource` body.

- **Invisible in full renders.** Pure marker: bumps an internal per-render
  segment counter and carries a caption used only for crop stubs. Existing
  `\LComment` phase headers are left untouched and still show in full renders.
- A **segment** spans from one `\tfsegment` to the next. Content before the
  first marker is segment 0 (implicit preamble segment).
- **Placement constraint:** a marker must sit at block depth 0 — not inside an
  `\If/\For/\While` (algpseudocodex) or other open block — so every segment is
  internally balanced and can be dropped wholesale. The `\If{\MKE…}…\EndIf`
  block is one segment (marker before `\If`, next marker after `\EndIf`).

### `\tfcropdefault{on|off}`

Global default (place near the top of the document). `on` turns cropping on for
every rendering that has a `diff=` target.

### `crop` key on `\tfrendergame`

Per-call override of the global default:

```latex
\tfrendergame[diff=G4, crop=off]{kind}{G5}   % force full listing here
\tfrendergame[diff=G4, crop=on]{kind}{G5}    % force crop here
```

Crop is effective only when a `diff=` target is present. A no-diff clean render
(e.g. the initial `G0`) renders full regardless of the crop setting — there is
no target to compute changes against.

Figures (`\tfrenderfigure` / `\tffigure`) are unaffected; they intentionally
show consolidated multi-game listings and never crop.

### `\tfsegmentstub{captions}`

User-redefinable macro (like `\tfchanged`) that renders the elision stub for a
run of unchanged segments. Default: a dimmed, unnumbered line (`\Statex`-style)
of the form `⋯ Initiator session, Additional key exchange (unchanged) ⋯`.

## Behavior

A cropped render shows only segments containing at least one changed line —
where "changed" means, vs the diff target: a line added, a line changed, or a
line removed within that segment. Each contiguous run of *unchanged* segments
collapses to one `\tfsegmentstub{...}` naming the collapsed captions.

Example (`Red2`, a responder-only `SKEYSEED_0` → oracle change): renders the
responder initial-derivation segment (with the changed line highlighted) plus
stubs for every other segment — a handful of lines instead of ~185.

## Implementation

### LaTeX package (`latex/texfrog.sty`)

Cropping adds a scan pass; a cropped `\tfrendergame[diff=T, crop=on]` runs three
walks of the stored source token list:

1. **Record** (existing, mode 1): walk target `T`, record active `\tfonly`
   positions into `\g__tf_recorded_prop`.
2. **Scan** (new mode, e.g. 5): walk current game `G`, maintaining a segment
   counter bumped at each `\tfsegment`. For each `\tfonly` position classify:
   - active-in-`G` ∧ ¬recorded → *add/change* → mark current segment active
   - ¬active-in-`G` ∧ recorded → *remove* → mark current segment active
   Store the set of active segment indices (a prop keyed by segment index).
3. **Render** (existing mode 2, crop-aware): walk `G` again, maintaining the
   segment counter. At each `\tfsegment`: if the segment is active, emit its
   content with highlighting exactly as today; if inactive, suppress content
   and accumulate its caption, flushing a single `\tfsegmentstub{...}` when the
   next active segment (or end of source) is reached.

`\tfsegment` becomes mode-sensitive like `\tfonly`:
- modes 0 (inactive capture), 3 (clean full), 4 (figure): no-op / emit nothing.
- scan mode: bump segment counter only.
- crop-render mode: bump counter + stub-flush logic.

New state:
- `\g__tf_crop_default_bool` — set by `\tfcropdefault`.
- crop key parsed in `texfrog / rendergame` keys, overriding the default.
- `\g__tf_seg_int` — current segment index during passes.
- `\g__tf_active_seg_prop` — set of active segment indices from the scan pass.
- stub-accumulation temporaries.

The existing non-crop paths (`__tf_render_game_clean`, and the highlighted
render when crop is off) are unchanged.

### Python (HTML export + validation)

A game is already filtered to a list of content lines, so the Python side is
simpler than LaTeX.

- **`filter.py`:**
  - Parse `\tfsegment{...}` markers; group the filtered line list into segments
    (list of `(caption, lines)`), with an implicit preamble segment 0.
  - New helper: compute active segments — per-segment compare of target vs
    current line lists (reuse `compute_changed_lines` / `compute_removed_lines`
    and map indices to segments, or compare segment line-lists directly).
  - New helper: build the cropped line list — active segments' lines, plus one
    stub line per contiguous inactive run.
  - `\tfsegment` marker lines are stripped from normal (non-crop) output.
- **`tex_parser.py`:** recognize `\tfsegment` (capture caption + structure) and
  the `crop` key / `\tfcropdefault` so HTML crop decisions match the PDF.
- **`output/html.py`:** when crop is effective for a game, feed the cropped line
  list into the existing `_compile_game_to_svg` — fewer input lines, no SVG
  re-architecture. Stub rendered with the same `\tfsegmentstub` (defined in the
  HTML wrapper template alongside `\tfchanged`).

### Rollout in `proof/proof.tex`

- Add `\tfcropdefault{on}`.
- Insert `\tfsegment{…}` markers at the phase boundaries already delimited by
  `\LComment`: Initiator session / MKE / PPK / IKE_AUTH / first child SA, then
  Responder session / its MKE / PPK / IKE_AUTH / child SA (both `kind` and,
  where useful, `cka`). Markers go at block depth 0 (MKE and PPK `\If` blocks
  each become a single segment).
- Set `crop=off` on the initial full renders where a whole listing is wanted
  (e.g. `G0`).

## Validation (`validate.py` / `texfrog check`)

Add warnings:
- `\tfsegment` with an empty caption.
- `\tfsegment` at nonzero block depth (unbalanced `\If/\For/\While` open) —
  breaks crop balance.
- Crop enabled (default or per-call) for a source with **no** `\tfsegment`
  markers — the whole source is one segment, so cropping gains nothing (either
  everything shows or, if unchanged, everything stubs).

## Testing

- **Python unit tests (`tests/`):**
  - Positive: segment grouping; active-segment computation; cropped line output
    (changed segment shown with highlight, unchanged runs stubbed with joined
    captions); crop off / no diff → full output.
  - Negative: no markers + crop → full output + warning; marker mid-`\If` →
    validator warning; empty caption → warning.
- **LaTeX path:** extend an example (the algpseudocodex tutorial) with a
  `\tfsegment`-marked source and a cropped render so `pdflatex` and
  `texfrog html build` exercise the crop path in CI.

## Documentation (in scope)

- `docs/writing-proofs.md`: new section on `\tfsegment`, `\tfcropdefault`, the
  `crop` key, `\tfsegmentstub`, and the placement constraint.
- `docs/system-design.md`: the three-pass render, the new scan mode, and the
  Python crop pipeline.
- `README.md`: command reference entries for the new commands/keys.
- `texfrog/.claude/CLAUDE.md`: add the new commands to the key-commands list.
- `proof/CLAUDE.md`: mention markers + `\tfcropdefault` in "How TeXFrog works
  here" and the renumbering/extension checklist.

## Open questions / possible issues

Accepted for now (fine as designed), flagged for revisiting if they bite:

- **Placement constraint (depth 0 only).** Segments must start at block depth 0,
  so an `\If…\EndIf` block is one atomic segment — no cropping *within* a large
  conditional block. Acceptable given the current source structure.
- **Removal-only segments surface.** A hop that only *deletes* a `\tfonly` line
  marks its segment active and renders the neighborhood, even though the PDF
  shows no highlight for pure removals today. Kept: it keeps the reader oriented
  at the deletion site.
- **Stub coalescing.** Contiguous unchanged segments collapse into one stub with
  joined captions. Alternative (one stub per segment) not chosen; revisit if the
  joined caption gets unwieldy.

## Out of scope

- Line-level fold/context view (rejected; see Approach).
- Cropping figures.
- Automatic placement of `\tfsegment` markers.
