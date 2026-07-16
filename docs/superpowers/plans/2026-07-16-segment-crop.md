# Segment Auto-Crop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a diffed `\tfrendergame` show only the algorithm segments that changed vs its diff target, collapsing unchanged runs into a caption stub, in both the `pdflatex` PDF and the HTML viewer.

**Architecture:** Author marks balanced-block boundaries with `\tfsegment{Caption}`. LaTeX gains a scan pass that marks which segments changed, and a crop-aware render pass that emits active segments (highlighted as today) and one `\tfsegmentstub` per unchanged run. Python (HTML export) reuses its existing per-game filtered line lists: split at markers, compare segment-by-segment against the diff target, and rebuild a cropped line list before compiling the SVG.

**Tech Stack:** Python 3.10+ (`texfrog` package, pytest), expl3/LaTeX (`latex/texfrog.sty`), algpseudocodex/cryptocode/nicodemus profiles.

## Global Constraints

- Spec: `docs/superpowers/specs/2026-07-16-segment-crop-design.md`.
- Math in LaTeX uses `\( \)`, not `$` (repo-wide rule); US English.
- `\tfsegment` markers must sit at block depth 0 (not inside `\If/\For/\While`) so each segment is internally balanced.
- Crop is effective only when a `diff=` target is present; no-diff clean renders always render full.
- Figures (`\tfrenderfigure`/`\tffigure`) never crop.
- **HTML nuance:** the HTML viewer renders one SVG per game, so it has no per-`\tfrendergame` call to read a `crop=` key from — HTML crop is governed solely by the global `\tfcropdefault`. The per-call `crop=on|off` override is a **PDF-only** refinement. Document this divergence.
- Two git repos: `texfrog/` (the tool — Tasks 1–7, 9) and its parent `ikev2-analysis/` for `proof/proof.tex` (Task 8). Commit each in its own repo. Work on branch `feature/segment-crop` (already created in `texfrog/`).
- Run Python tests with `.venv/bin/pytest tests/ -q` from the `texfrog/` dir.

---

## File Structure

- `texfrog/texfrog/filter.py` — add `Segment` dataclass, `split_into_segments`, `compute_active_segments`, `crop_to_active_segments`.
- `texfrog/texfrog/model.py` — add `Proof.crop_default: bool`.
- `texfrog/texfrog/tex_parser.py` — parse `\tfcropdefault`; populate `crop_default`.
- `texfrog/texfrog/packages.py` — add `PackageProfile.html_tfsegmentstub()`.
- `texfrog/texfrog/output/html.py` — crop per-game (and its `-removed`) line lists when `crop_default`; define `\tfsegmentstub` in the wrapper.
- `texfrog/texfrog/validate.py` — three new warnings.
- `texfrog/latex/texfrog.sty` — `\tfsegment`, `\tfcropdefault`, `crop` key, `\tfsegmentstub`, scan pass, crop-aware render.
- `texfrog/tests/…` — unit + compile tests.
- `texfrog/docs/*.md`, `texfrog/README.md`, `texfrog/.claude/CLAUDE.md` — docs.
- `proof/proof.tex`, `proof/CLAUDE.md` — rollout (parent repo).

---

## Task 1: Segment splitting (Python)

**Files:**
- Modify: `texfrog/texfrog/filter.py`
- Test: `texfrog/tests/test_filter.py`

**Interfaces:**
- Produces:
  - `SEGMENT_RE: re.Pattern` matching `\tfsegment{caption}` lines.
  - `@dataclass class Segment: caption: str | None; lines: list[str]`
  - `def split_into_segments(lines: list[str]) -> list[Segment]` — splits a filtered line list at `\tfsegment` markers. Content before the first marker is `Segment(caption=None, lines=[...])` (segment 0, always present even if empty). Marker lines themselves are **not** included in any segment's `lines`.

- [ ] **Step 1: Write the failing test**

Add to `texfrog/tests/test_filter.py`:

```python
from texfrog.filter import Segment, split_into_segments


def test_split_into_segments_basic():
    lines = [
        r"\State a",
        r"\tfsegment{Responder}",
        r"\State b",
        r"\State c",
    ]
    segs = split_into_segments(lines)
    assert segs == [
        Segment(caption=None, lines=[r"\State a"]),
        Segment(caption="Responder", lines=[r"\State b", r"\State c"]),
    ]


def test_split_into_segments_no_markers():
    lines = [r"\State a", r"\State b"]
    segs = split_into_segments(lines)
    assert segs == [Segment(caption=None, lines=[r"\State a", r"\State b"])]


def test_split_into_segments_leading_marker():
    lines = [r"\tfsegment{First}", r"\State a"]
    segs = split_into_segments(lines)
    assert segs == [
        Segment(caption=None, lines=[]),
        Segment(caption="First", lines=[r"\State a"]),
    ]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `.venv/bin/pytest tests/test_filter.py -k split_into_segments -v`
Expected: FAIL with `ImportError: cannot import name 'Segment'`.

- [ ] **Step 3: Write minimal implementation**

Add near the top of `texfrog/texfrog/filter.py` (after the existing imports and regexes):

```python
from dataclasses import dataclass, field

# Matches a \tfsegment{caption} marker line (optional leading whitespace).
_SEGMENT_RE = re.compile(r"^\s*\\tfsegment\s*\{(?P<caption>.*)\}\s*$")
SEGMENT_RE = _SEGMENT_RE


@dataclass
class Segment:
    """A run of content lines between two \\tfsegment markers."""

    caption: str | None  # None for the implicit preamble segment (segment 0)
    lines: list[str] = field(default_factory=list)


def split_into_segments(lines: list[str]) -> list[Segment]:
    """Split filtered content lines into segments at \\tfsegment markers.

    Content before the first marker is segment 0 with ``caption=None``.
    Marker lines are consumed (not included in any segment's ``lines``).

    Args:
        lines: Filtered content lines for one game (may contain markers).

    Returns:
        A list of :class:`Segment`, always at least one element long.
    """
    segments: list[Segment] = [Segment(caption=None)]
    for line in lines:
        m = _SEGMENT_RE.match(line)
        if m:
            segments.append(Segment(caption=m.group("caption").strip()))
        else:
            segments[-1].lines.append(line)
    return segments
```

- [ ] **Step 4: Run test to verify it passes**

Run: `.venv/bin/pytest tests/test_filter.py -k split_into_segments -v`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add texfrog/filter.py tests/test_filter.py
git commit -m "feat(filter): split filtered lines into \\tfsegment segments"
```

---

## Task 2: Active-segment computation + crop rebuild (Python)

**Files:**
- Modify: `texfrog/texfrog/filter.py`
- Test: `texfrog/tests/test_filter.py`

**Interfaces:**
- Consumes: `Segment`, `split_into_segments` (Task 1).
- Produces:
  - `def compute_active_segments(target_lines: list[str], curr_lines: list[str]) -> set[int]` — returns the 0-based indices of segments (in `curr_lines`' segmentation) whose line list differs from the aligned segment in `target_lines`. Segment i aligns with segment i (marker sequence is identical across games). A current segment with no target counterpart (index ≥ len(target segments)) is active iff it has any non-blank line.
  - `def crop_to_active_segments(lines: list[str], active: set[int], stub_macro: str = r"\tfsegmentstub") -> tuple[list[str], list[int]]` — rebuild `lines` keeping only active segments, replacing each contiguous run of inactive segments with a single stub line `\tfsegmentstub{Cap1, Cap2}` (captions joined by `", "`, skipping `None`/empty). Segment 0 (preamble) is always kept verbatim regardless of `active` (it holds the `\begin{algorithmic}` opener etc.). Returns `(new_lines, kept_orig_indices)` where `kept_orig_indices[k]` is the index into the *original* `lines` of `new_lines[k]`, or `-1` for a synthesized stub line. The index map lets the caller remap highlight indices.

- [ ] **Step 1: Write the failing test**

Add to `texfrog/tests/test_filter.py`:

```python
from texfrog.filter import compute_active_segments, crop_to_active_segments


def test_compute_active_segments_detects_change():
    target = [r"\State a", r"\tfsegment{R}", r"\State b"]
    curr = [r"\State a", r"\tfsegment{R}", r"\State b2"]
    assert compute_active_segments(target, curr) == {1}


def test_compute_active_segments_none_changed():
    lines = [r"\State a", r"\tfsegment{R}", r"\State b"]
    assert compute_active_segments(lines, lines) == set()


def test_crop_keeps_active_and_stubs_inactive():
    lines = [
        r"\begin{algorithmic}",
        r"\tfsegment{Init}",
        r"\State a",
        r"\tfsegment{Resp}",
        r"\State b",
        r"\end{algorithmic}",
    ]
    # active segments: 0 (preamble, always kept) and 2 (Resp)
    new_lines, idx_map = crop_to_active_segments(lines, active={2})
    assert new_lines == [
        r"\begin{algorithmic}",
        r"\tfsegmentstub{Init}",
        r"\State b",
        r"\end{algorithmic}",
    ]
    # stub line maps to -1; kept lines map to their original indices
    assert idx_map == [0, -1, 4, 5]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `.venv/bin/pytest tests/test_filter.py -k "active_segments or crop_keeps" -v`
Expected: FAIL with `ImportError`.

- [ ] **Step 3: Write minimal implementation**

Append to `texfrog/texfrog/filter.py`:

```python
def _seg_content(seg: Segment) -> list[str]:
    """Non-blank content lines of a segment (blank lines ignored for diffing)."""
    return [ln for ln in seg.lines if ln.strip()]


def compute_active_segments(
    target_lines: list[str], curr_lines: list[str]
) -> set[int]:
    """Indices of current segments that differ from the aligned target segment.

    Segment i in ``curr_lines`` aligns with segment i in ``target_lines`` (the
    \\tfsegment marker sequence is identical across games). A segment is active
    if its non-blank content differs, or (when it has no target counterpart) if
    it contains any non-blank line.

    Segment 0 (preamble) is reported like any other; callers keep it regardless.

    Args:
        target_lines: Filtered lines of the diff-target game.
        curr_lines: Filtered lines of the current game.

    Returns:
        Set of 0-based active segment indices into the current segmentation.
    """
    target_segs = split_into_segments(target_lines)
    curr_segs = split_into_segments(curr_lines)
    active: set[int] = set()
    for i, seg in enumerate(curr_segs):
        tgt = _seg_content(target_segs[i]) if i < len(target_segs) else []
        if _seg_content(seg) != tgt:
            active.add(i)
    return active


def crop_to_active_segments(
    lines: list[str],
    active: set[int],
    stub_macro: str = r"\tfsegmentstub",
) -> tuple[list[str], list[int]]:
    """Rebuild ``lines`` keeping active segments, stubbing inactive runs.

    Segment 0 (preamble) is always kept verbatim. Each maximal run of inactive
    segments (index >= 1) collapses to a single ``\\tfsegmentstub{captions}``
    line, captions joined with ``", "`` (blank/None captions skipped; if the
    whole run has no captions the stub argument is empty).

    Args:
        lines: Filtered content lines (with markers) for the current game.
        active: Active segment indices from :func:`compute_active_segments`.
        stub_macro: Macro name emitted for a stubbed run.

    Returns:
        ``(new_lines, idx_map)`` where ``idx_map[k]`` is the original index of
        ``new_lines[k]`` in ``lines``, or ``-1`` for a synthesized stub line.
    """
    segs = split_into_segments(lines)
    # Precompute, for each segment, the original index of its first content line.
    orig_index: list[list[int]] = []
    cursor = 0
    for i, seg in enumerate(segs):
        if i > 0:
            cursor += 1  # the \tfsegment marker line consumed at this boundary
        idxs = list(range(cursor, cursor + len(seg.lines)))
        orig_index.append(idxs)
        cursor += len(seg.lines)

    new_lines: list[str] = []
    idx_map: list[int] = []
    pending_caps: list[str] = []

    def flush_stub() -> None:
        if not pending_caps and not _stub_pending[0]:
            return
        caps = ", ".join(c for c in pending_caps if c)
        new_lines.append(f"{stub_macro}{{{caps}}}")
        idx_map.append(-1)
        pending_caps.clear()
        _stub_pending[0] = False

    _stub_pending = [False]  # whether any inactive segment was seen in this run
    for i, seg in enumerate(segs):
        keep = (i == 0) or (i in active)
        if keep:
            flush_stub()
            for ln, oi in zip(seg.lines, orig_index[i]):
                new_lines.append(ln)
                idx_map.append(oi)
        else:
            _stub_pending[0] = True
            if seg.caption:
                pending_caps.append(seg.caption)
    flush_stub()
    return new_lines, idx_map
```

- [ ] **Step 4: Run test to verify it passes**

Run: `.venv/bin/pytest tests/test_filter.py -k "active_segments or crop_keeps" -v`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add texfrog/filter.py tests/test_filter.py
git commit -m "feat(filter): compute active segments and crop line lists"
```

---

## Task 3: Parse `\tfcropdefault` (Python)

**Files:**
- Modify: `texfrog/texfrog/model.py`, `texfrog/texfrog/tex_parser.py`
- Test: `texfrog/tests/test_tex_parser.py`

**Interfaces:**
- Consumes: `_find_all_commands`, `_extract_one_arg` (existing in `tex_parser.py`).
- Produces: `Proof.crop_default: bool` (default `False`), set `True` when the document contains `\tfcropdefault{on}`.

- [ ] **Step 1: Write the failing test**

Add to `texfrog/tests/test_tex_parser.py` (mirror how existing tests build a temp `.tex`; reuse the module's existing fixture helper if present, otherwise `tmp_path`):

```python
from texfrog.tex_parser import parse_tex_proofs


def _write(tmp_path, body):
    p = tmp_path / "proof.tex"
    p.write_text(body, encoding="utf-8")
    return p


_MIN_DOC = r"""
\tfgames{s}{G0, G1}
\tfgamename{s}{G0}{G_0}
\tfgamename{s}{G1}{G_1}
\begin{tfsource}{s}
\State x
\end{tfsource}
"""


def test_crop_default_off_by_default(tmp_path):
    proof = parse_tex_proofs(_write(tmp_path, _MIN_DOC))[0]
    assert proof.crop_default is False


def test_crop_default_on(tmp_path):
    proof = parse_tex_proofs(_write(tmp_path, r"\tfcropdefault{on}" + _MIN_DOC))[0]
    assert proof.crop_default is True
```

- [ ] **Step 2: Run test to verify it fails**

Run: `.venv/bin/pytest tests/test_tex_parser.py -k crop_default -v`
Expected: FAIL with `AttributeError: 'Proof' object has no attribute 'crop_default'`.

- [ ] **Step 3: Write minimal implementation**

In `texfrog/texfrog/model.py`, add a field to `Proof` (after `preamble`):

```python
    crop_default: bool = False   # True when \tfcropdefault{on} is present
```

In `texfrog/texfrog/tex_parser.py`, inside `parse_tex_proofs` where the full document text is available (same place other document-level options like the package are read), compute the flag and pass it to every `Proof` constructed:

```python
    _crop_vals = _extract_one_arg(full_text, "tfcropdefault")
    crop_default = any(v.strip().lower() == "on" for v in _crop_vals)
```

Then set `crop_default=crop_default` on each `Proof(...)` built in that function. (Search for the `Proof(` constructor call(s) and add the keyword argument.)

- [ ] **Step 4: Run test to verify it passes**

Run: `.venv/bin/pytest tests/test_tex_parser.py -k crop_default -v`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add texfrog/model.py texfrog/tex_parser.py tests/test_tex_parser.py
git commit -m "feat(parser): read \\tfcropdefault into Proof.crop_default"
```

---

## Task 4: Wire crop into HTML export (Python)

**Files:**
- Modify: `texfrog/texfrog/packages.py`, `texfrog/texfrog/output/html.py`
- Test: `texfrog/tests/test_html_internals.py`

**Interfaces:**
- Consumes: `compute_active_segments`, `crop_to_active_segments` (Task 2); `Proof.crop_default` (Task 3); `PackageProfile` (existing).
- Produces:
  - `PackageProfile.html_tfsegmentstub() -> str` returning a `\newcommand{\tfsegmentstub}[1]{...}` definition (a dimmed, unnumbered line). algpseudocodex/nicodemus (text mode): `\Statex`-style; cryptocode (math + line-sep): a `\\`-terminated line.
  - `html.py` module-level helper `_apply_crop(curr_lines, prev_lines, changed) -> tuple[list[str], set[int]]` producing cropped lines + remapped changed indices.

- [ ] **Step 1: Write the failing test**

Add to `texfrog/tests/test_html_internals.py`:

```python
from texfrog.output.html import _apply_crop


def test_apply_crop_remaps_changed_indices():
    prev = [
        r"\begin{algorithmic}",
        r"\tfsegment{Init}",
        r"\State a",
        r"\tfsegment{Resp}",
        r"\State b",
        r"\end{algorithmic}",
    ]
    curr = [
        r"\begin{algorithmic}",
        r"\tfsegment{Init}",
        r"\State a",
        r"\tfsegment{Resp}",
        r"\State b2",
        r"\end{algorithmic}",
    ]
    # \State b2 is at original index 4 and changed
    cropped, changed = _apply_crop(curr, prev, {4})
    assert cropped == [
        r"\begin{algorithmic}",
        r"\tfsegmentstub{Init}",
        r"\State b2",
        r"\end{algorithmic}",
    ]
    assert changed == {2}  # \State b2 now at index 2


def test_html_tfsegmentstub_defined_for_each_profile():
    from texfrog.packages import get_profile
    for name in ("cryptocode", "nicodemus", "algpseudocodex"):
        assert r"\tfsegmentstub" in get_profile(name).html_tfsegmentstub()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `.venv/bin/pytest tests/test_html_internals.py -k "apply_crop or segmentstub" -v`
Expected: FAIL with `ImportError` / `AttributeError`.

- [ ] **Step 3: Write minimal implementation**

In `texfrog/texfrog/packages.py`, add a method to `PackageProfile`:

```python
    def html_tfsegmentstub(self) -> str:
        r"""Return the \tfsegmentstub definition for the HTML wrapper."""
        body = r"{\color{black!55}$\cdots$~\textit{#1 (unchanged)}~$\cdots$}"
        if self.has_line_separators:  # cryptocode: math + \\ separated lines
            return r"\newcommand{\tfsegmentstub}[1]{" + body + r" \\}"
        # algpseudocodex / nicodemus: unnumbered continuation line
        return r"\newcommand{\tfsegmentstub}[1]{\Statex " + body + r"}"
```

In `texfrog/texfrog/output/html.py`, add near the other module-level helpers:

```python
from ..filter import compute_active_segments, crop_to_active_segments


def _apply_crop(
    curr_lines: list[str],
    prev_lines: list[str],
    changed: set[int],
) -> tuple[list[str], set[int]]:
    """Crop ``curr_lines`` to changed segments and remap ``changed`` indices.

    Args:
        curr_lines: Filtered lines for the current game (with markers).
        prev_lines: Filtered lines for the diff-target game (with markers).
        changed: 0-based changed indices into ``curr_lines``.

    Returns:
        ``(cropped_lines, remapped_changed)``.
    """
    active = compute_active_segments(prev_lines, curr_lines)
    cropped, idx_map = crop_to_active_segments(curr_lines, active)
    remapped = {k for k, orig in enumerate(idx_map) if orig in changed}
    return cropped, remapped
```

Register the stub definition in `_build_wrapper_template`: in the `if not commentary:` block, after the `profile.html_tfgamelabel()` line, append:

```python
            profile.html_tfsegmentstub().replace("{", "{{").replace("}", "}}"),
```

Wire cropping into `generate_html`'s per-game loop. Where `game_lines` and `changed` are computed (around the `_write_game_file(label, game_lines, changed, ...)` call), insert before it:

```python
            if proof.crop_default and i > 0 and prev_label is not None:
                game_lines, changed = _apply_crop(
                    game_lines, _filter_game(prev_label), changed
                )
```

(Note: `prev_label` is already computed above for the `changed` calculation; if it is only bound in the non-reduction branch, lift its definition so it is available here — initialize `prev_label = None` before the `if i == 0` block and assign in both branches.)

Also crop the reverse `-removed.tex` panel so the side-by-side stays aligned. In the `non_red_games` loop, replace the body that computes `removed_indices` and writes the file with:

```python
            removed_indices = compute_removed_lines(prev_diff, next_diff)
            out_prev_lines = prev_lines
            if proof.crop_default:
                active = compute_active_segments(next_lines, prev_lines)
                out_prev_lines, idx_map = crop_to_active_segments(prev_lines, active)
                removed_indices = {
                    k for k, orig in enumerate(idx_map) if orig in removed_indices
                }
            _write_game_file(
                game.label, out_prev_lines, removed_indices,
                latex_dir / f"{game.label}-removed.tex",
                macro=r"\tfremoved",
                procedure_header_cmd=proc_hdr_cmd,
            )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `.venv/bin/pytest tests/test_html_internals.py -k "apply_crop or segmentstub" -v`
Expected: PASS (2 tests).

- [ ] **Step 5: Run the full Python suite to check no regressions**

Run: `.venv/bin/pytest tests/ -q`
Expected: PASS (all existing tests still green).

- [ ] **Step 6: Commit**

```bash
git add texfrog/packages.py texfrog/output/html.py tests/test_html_internals.py
git commit -m "feat(html): crop game SVGs to changed segments when crop_default is on"
```

---

## Task 5: LaTeX commands + scaffolding (`texfrog.sty`)

**Files:**
- Modify: `texfrog/latex/texfrog.sty`
- Test: `texfrog/tests/test_sty_compilation.py`

**Interfaces:**
- Produces (LaTeX user API):
  - `\tfsegment{Caption}` — boundary marker; no-op except in scan/crop-render modes.
  - `\tfcropdefault{on|off}` — sets `\g__tf_crop_default_bool`.
  - `crop` key on `\tfrendergame` (`crop=on|off`), overriding the default.
  - `\tfsegmentstub{captions}` — user-redefinable stub macro.
- This task adds the state and commands but keeps `\tfrendergame` behavior unchanged (crop path added in Task 6). With markers present and crop off, output must be identical to today.

- [ ] **Step 1: Write the failing test**

Add to `texfrog/tests/test_sty_compilation.py` (mirror the existing fixture-compile helper in that file; the snippet below assumes a helper `compile_fixture(tmp_path, body, package=...) -> CompletedProcess` — reuse whatever the file already defines, adapting names):

```python
def test_tfsegment_invisible_in_full_render(tmp_path):
    body = r"""
\tfgames{s}{G0,G1}
\tfgamename{s}{G0}{G_0}\tfgamename{s}{G1}{G_1}
\begin{tfsource}{s}
\begin{algorithmic}[1]
\State \(x \gets 0\)
\tfsegment{Second}
\tfonly{G1}{\State \(y \gets 1\)}
\end{algorithmic}
\end{tfsource}
\begin{document}
\tfrendergame{s}{G0}
\end{document}
"""
    proc = compile_fixture(tmp_path, body, package="algpseudocodex")
    assert proc.returncode == 0
    # \tfsegment must not leak literal text into the PDF/log as an error
    assert "Undefined control sequence" not in proc.stdout
```

- [ ] **Step 2: Run test to verify it fails**

Run: `.venv/bin/pytest tests/test_sty_compilation.py -k tfsegment_invisible -v`
Expected: FAIL — `\tfsegment` undefined → "Undefined control sequence".

- [ ] **Step 3: Write minimal implementation**

In `texfrog/latex/texfrog.sty`, inside `\ExplSyntaxOn`:

Add state variables near the other `\..._new:N` declarations:

```latex
% Segment cropping state
\bool_new:N \g__tf_crop_default_bool
\bool_new:N \g__tf_crop_active_bool      % effective crop for current render
\int_new:N  \g__tf_seg_int               % current segment index during a pass
\prop_new:N \g__tf_active_seg_prop       % active segment indices (scan result)
\bool_new:N \g__tf_seg_changed_bool      % did current segment change? (scan)
\tl_new:N   \g__tf_stub_caps_tl          % accumulated captions for a stub run
\bool_new:N \g__tf_stub_pending_bool     % inside an inactive run?
```

Add mode `5` (scan) handling to `\tfonly` (extend the `\int_case:nnF` in `\NewDocumentCommand{\tfonly}`):

```latex
        { 5 } { \__tf_only_scan:nn { #2 } { #3 } }
```

Add the scan handler (mirrors `\__tf_only_render:nn` but sets the segment-changed flag instead of emitting):

```latex
% --- Scan mode (mark segment changed if this \tfonly is add/remove) ---
\cs_new_protected:Nn \__tf_only_scan:nn
  {
    \int_gincr:N \g__tf_pos_int
    \__tf_resolve_tags:n { #1 }
    \__tf_game_in_tags:VTF \g__tf_current_game_tl
      {
        % active in current game: changed iff NOT recorded in target
        \prop_if_in:NeF \g__tf_recorded_prop
          { \int_use:N \g__tf_pos_int }
          { \bool_gset_true:N \g__tf_seg_changed_bool }
      }
      {
        % not in current: changed iff it WAS recorded in target (a removal)
        \prop_if_in:NeT \g__tf_recorded_prop
          { \int_use:N \g__tf_pos_int }
          { \bool_gset_true:N \g__tf_seg_changed_bool }
      }
  }
```

Add `\prop_if_in:NeF`/`NeT` variants near the existing `\prop_if_in:NeTF` variant line:

```latex
\cs_generate_variant:Nn \prop_if_in:NnT { NeT }
\cs_generate_variant:Nn \prop_if_in:NnF { NeF }
```

Define the marker, the default, the stub macro, and the `\tfrendergame` key. Add `\tfsegment` as mode-sensitive:

```latex
\NewDocumentCommand{\tfsegment}{ m }
  {
    \int_case:nnF { \g__tf_mode_int }
      {
        { 5 } { \__tf_seg_boundary_scan: }
        { 2 } { \bool_if:NT \g__tf_crop_active_bool { \__tf_seg_boundary_render:n { #1 } } }
      }
      { } % all other modes: pure no-op (invisible)
  }
```

Boundary handlers (Task 6 fills the render side; scan side here):

```latex
% Scan: close the previous segment (commit its changed flag), open the next.
\cs_new_protected:Nn \__tf_seg_boundary_scan:
  {
    \bool_if:NT \g__tf_seg_changed_bool
      { \prop_gput:Nen \g__tf_active_seg_prop { \int_use:N \g__tf_seg_int } { 1 } }
    \int_gincr:N \g__tf_seg_int
    \bool_gset_false:N \g__tf_seg_changed_bool
  }
```

Add `\tfcropdefault` and the stub macro:

```latex
\NewDocumentCommand{\tfcropdefault}{ m }
  {
    \str_if_eq:nnTF { #1 } { on }
      { \bool_gset_true:N  \g__tf_crop_default_bool }
      { \bool_gset_false:N \g__tf_crop_default_bool }
  }

\providecommand{\tfsegmentstub}[1]
  {\Statex{\color{black!55}\ensuremath{\cdots}~\textit{#1~(unchanged)}~\ensuremath{\cdots}}}
```

(For cryptocode/nicodemus profiles a `\\`-terminated / `\item`-prefixed variant is set in the profile-branch block, mirroring `\tfchanged`; guard with `\tl_if_eq:NnT \g__tf_package_tl { ... }` as done for `\tfchanged`.)

Add the `crop` key to the rendergame keyset (after the `diff` key):

```latex
    crop .tl_set:N = \l__tf_crop_key_tl,
    crop .initial:n = { default },
```

Declare `\tl_new:N \l__tf_crop_key_tl` with the other `\l__tf_..._tl`. Resolve the effective crop bool inside `\tfrendergame` (Task 6 uses it).

- [ ] **Step 4: Run test to verify it passes**

Run: `.venv/bin/pytest tests/test_sty_compilation.py -k tfsegment_invisible -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add latex/texfrog.sty tests/test_sty_compilation.py
git commit -m "feat(sty): add \\tfsegment, \\tfcropdefault, \\tfsegmentstub, scan mode"
```

---

## Task 6: LaTeX crop-aware render pass (`texfrog.sty`)

**Files:**
- Modify: `texfrog/latex/texfrog.sty`
- Test: `texfrog/tests/test_sty_compilation.py`

**Interfaces:**
- Consumes: scan mode + state from Task 5.
- Produces: `\tfrendergame[diff=T, crop=on]{fam}{G}` emits only active segments with a `\tfsegmentstub` per inactive run; `crop=off` (or no diff) renders full as today.

- [ ] **Step 1: Write the failing test**

Add to `texfrog/tests/test_sty_compilation.py`. The check uses `pdftotext` on the produced PDF; if the fixture helper returns only a `CompletedProcess`, extend it (or add a sibling helper) to also return the extracted text — adapt to the file's existing pattern:

```python
def test_crop_stubs_unchanged_segment(tmp_path):
    body = r"""
\tfgames{s}{G0,G1}
\tfgamename{s}{G0}{G_0}\tfgamename{s}{G1}{G_1}
\tfcropdefault{on}
\begin{tfsource}{s}
\begin{algorithmic}[1]
\State \(x \gets 0\)
\tfsegment{Initiator}
\State \(a \gets 1\)
\tfsegment{Responder}
\tfonly{G0}{\State \(y \gets 0\)}
\tfonly{G1}{\State \(y \gets 1\)}
\end{algorithmic}
\end{tfsource}
\begin{document}
\tfrendergame[diff=G0]{s}{G1}
\end{document}
"""
    text = compile_fixture_text(tmp_path, body, package="algpseudocodex")
    assert "unchanged" in text          # a stub was emitted
    assert "Initiator" in text          # stubbed caption shown
    assert "y \\gets 1" in text or "y" in text  # changed line present
```

- [ ] **Step 2: Run test to verify it fails**

Run: `.venv/bin/pytest tests/test_sty_compilation.py -k crop_stubs -v`
Expected: FAIL — no crop path yet, so `\tfsegment` is a no-op in render mode and no stub appears ("unchanged" absent).

- [ ] **Step 3: Write minimal implementation**

In `texfrog/latex/texfrog.sty`, resolve the effective crop bool and branch inside `\tfrendergame`:

```latex
\NewDocumentCommand{\tfrendergame}{ O{} m m }
  {
    \group_begin:
    \keys_set:nn { texfrog / rendergame } { #1 }
    % resolve effective crop: key overrides default; needs a diff target
    \bool_set_false:N \l_tmpa_bool
    \str_case:VnF \l__tf_crop_key_tl
      {
        { on }  { \bool_set_true:N  \l_tmpa_bool }
        { off } { \bool_set_false:N \l_tmpa_bool }
      }
      { \bool_set_eq:NN \l_tmpa_bool \g__tf_crop_default_bool }
    \tl_if_empty:NTF \l__tf_diff_tl
      { \__tf_render_game_clean:nn { #2 } { #3 } }
      {
        \bool_if:NTF \l_tmpa_bool
          { \__tf_render_game_cropped:nnn { #2 } { #3 } { \l__tf_diff_tl } }
          { \__tf_render_game_highlighted:nnn { #2 } { #3 } { \l__tf_diff_tl } }
      }
    \group_end:
  }
```

Add the cropped render (record → scan → crop-render):

```latex
\cs_new_protected:Nn \__tf_render_game_cropped:nnn
  {
    \__tf_activate_source:n { #1 }
    % Pass A: record target-active positions.
    \prop_gclear:N \g__tf_recorded_prop
    \int_gset:Nn \g__tf_pos_int { 0 }
    \int_gset:Nn \g__tf_mode_int { 1 }
    \tl_gset:Ne \g__tf_current_game_tl { #3 }
    \vbox_set:Nn \g__tf_scratch_box { \tl_use:c { g__tf_source_ #1 _tl } }
    \box_gclear:N \g__tf_scratch_box
    % Pass B: scan current game, collect active segment indices.
    \prop_gclear:N \g__tf_active_seg_prop
    \int_gset:Nn \g__tf_pos_int { 0 }
    \int_gset:Nn \g__tf_seg_int { 0 }
    \bool_gset_false:N \g__tf_seg_changed_bool
    \int_gset:Nn \g__tf_mode_int { 5 }
    \tl_gset:Nn \g__tf_current_game_tl { #2 }
    \vbox_set:Nn \g__tf_scratch_box { \tl_use:c { g__tf_source_ #1 _tl } }
    \box_gclear:N \g__tf_scratch_box
    % commit the final segment's changed flag (no trailing \tfsegment closes it)
    \bool_if:NT \g__tf_seg_changed_bool
      { \prop_gput:Nen \g__tf_active_seg_prop { \int_use:N \g__tf_seg_int } { 1 } }
    % Pass C: crop-render.
    \bool_gset_true:N \g__tf_crop_active_bool
    \int_gset:Nn \g__tf_pos_int { 0 }
    \int_gset:Nn \g__tf_seg_int { 0 }
    \bool_gset_false:N \g__tf_stub_pending_bool
    \tl_gclear:N \g__tf_stub_caps_tl
    \int_gset:Nn \g__tf_mode_int { 2 }
    \tl_gset:Nn \g__tf_current_game_tl { #2 }
    \tl_use:c { g__tf_source_ #1 _tl }
    \__tf_flush_stub:            % flush a trailing inactive run
    \bool_gset_false:N \g__tf_crop_active_bool
    \int_gset:Nn \g__tf_mode_int { 0 }
  }
```

Segment 0 is always active for rendering purposes (it holds `\begin{algorithmic}`); the render-side `\tfonly` in an inactive segment must be suppressed. Gate the render-mode `\tfonly` emit on the active segment. Change `\__tf_only_render:nn` to check the active segment when cropping:

```latex
\cs_new_protected:Nn \__tf_only_render:nn
  {
    \int_gincr:N \g__tf_pos_int
    \bool_lazy_and:nnTF
      { \g__tf_crop_active_bool }
      { ! \__tf_seg_is_active_p: }
      { \__tf_resolve_tags:n { #1 } } % suppressed (still resolve to keep pos parity? pos already incremented)
      {
        \__tf_resolve_tags:n { #1 }
        \__tf_game_in_tags:VTF \g__tf_current_game_tl
          {
            \prop_if_in:NeTF \g__tf_recorded_prop
              { \int_use:N \g__tf_pos_int } { #2 }
              { \__tf_wrap_changed:n { #2 } }
          }
          { }
      }
  }
```

with predicate:

```latex
\prg_new_conditional:Nnn \__tf_seg_is_active:n { p, TF } { }  % placeholder
\cs_new:Nn \__tf_seg_is_active_p:
  {
    \bool_lazy_or:nnTF
      { \int_compare_p:nNn \g__tf_seg_int = { 0 } }
      { \prop_if_in_p:Ne \g__tf_active_seg_prop { \int_use:N \g__tf_seg_int } }
      { \prg_return_true: }  % NOTE: use \c_true_bool form below
      { \prg_return_false: }
  }
```

(Implement `\__tf_seg_is_active_p:` returning `\c_true_bool`/`\c_false_bool` via `\bool_lazy_or:nnTF { seg=0 } { prop_if_in }`. Generate `\prop_if_in_p:Ne` variant.)

The crop-render boundary handler emits the stub for an inactive run and bumps the segment counter; untagged content (plain `\State`, `\If`, etc.) in inactive segments is the remaining issue — see Step 3b.

**Step 3b — suppress untagged content in inactive segments.** Untagged lines are not wrapped in `\tfonly`, so they emit unconditionally. To crop them, the crop-render pass wraps each inactive segment's body in a discarded box: implement `\__tf_seg_boundary_render:n` to, when moving *into* an inactive segment, open a group that redirects output to a throwaway box until the next boundary; when moving into an active segment, close it and, if a run was skipped, emit `\tfsegmentstub`. Concretely, track `\g__tf_stub_pending_bool`; at each boundary decide active via `\__tf_seg_is_active_p:` for the *upcoming* segment index (post-increment):

```latex
\cs_new_protected:Nn \__tf_seg_boundary_render:n
  {
    \int_gincr:N \g__tf_seg_int
    \bool_if:NTF \__tf_seg_is_active_bool:
      {
        \__tf_flush_stub:            % close any pending inactive run first
      }
      {
        \bool_gset_true:N \g__tf_stub_pending_bool
        \tl_if_empty:nF { #1 }
          { \__tf_stub_append_caption:n { #1 } }
      }
  }
```

Because algpseudocodex has no easy "discard following lines" primitive, the robust approach is: in the crop-render pass, run the body once more but *filter untagged lines by segment activity*. Rather than boxing, redefine the emit so that inside an inactive segment the algpseudocodex line commands are swallowed. Provide `\__tf_begin_inactive:`/`\__tf_end_inactive:` that `\let` `\State`, `\Statex`, `\LComment`, `\If`, `\ElsIf`, `\Else`, `\EndIf`, `\For`, `\EndFor`, `\While`, `\EndWhile` to argument-swallowing no-ops for the duration of the inactive segment, then restore at the next active boundary. Since segments are balanced (Global Constraints), the swallowed block is self-contained and restoration is safe.

Define the swallowers:

```latex
\cs_new_protected:Nn \__tf_begin_inactive:
  {
    \cs_set_eq:NN \State \prg_do_nothing:
    \cs_set_protected:Npn \State { }
    % swallow one optional arg forms as needed; algpseudocodex \State takes the
    % rest of the line up to the line break, so \let to a line-eating no-op:
    ...
  }
```

Given the fragility of `\let`-swallowing arbitrary algpseudocodex lines, prefer the **box-capture** approach: wrap the entire inactive run in a single `\vbox_set:Nn \l_tmpa_box { ... }` whose content is discarded. Open the box at the boundary into an inactive segment and close it at the boundary out. Implement with a saved-group technique: at "enter inactive", `\group_begin:` and start capturing to a box via `\vbox_set:Nw`; at "enter active" or end, finish the box (`\vbox_set_end:`), discard it, `\group_end:`, then emit the stub. Use l3box's `\hbox_set:Nw`/`\hbox_set_end:` scanning-mark forms so arbitrary balanced content is captured.

**Implementation note for the executor:** validate this capture approach against algpseudocodex on the fixture in Step 1 *before* proceeding; if line-command box-capture misbehaves inside `algorithmic`, fall back to requiring every croppable line to be tagged (wrap untagged lines the author wants croppable in `\tfonly{<all games in the segment's range>}{...}`) and document that limitation. Record the chosen mechanism in `docs/system-design.md`.

- [ ] **Step 4: Run test to verify it passes**

Run: `.venv/bin/pytest tests/test_sty_compilation.py -k crop_stubs -v`
Expected: PASS.

- [ ] **Step 5: Verify crop=off is unchanged**

Add and run a test that the same fixture with `crop=off` on the call produces no "unchanged" text and contains both segments' content.

Run: `.venv/bin/pytest tests/test_sty_compilation.py -k "crop" -v`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add latex/texfrog.sty tests/test_sty_compilation.py
git commit -m "feat(sty): crop-aware render pass with segment stubs"
```

---

## Task 7: Validator warnings (Python)

**Files:**
- Modify: `texfrog/texfrog/validate.py`
- Test: `texfrog/tests/test_validate.py`

**Interfaces:**
- Consumes: `Proof` (with `source_text`, `crop_default`), `SEGMENT_RE` (Task 1).
- Produces: three added warning strings from `validate_proof`.

- [ ] **Step 1: Write the failing test**

Add to `texfrog/tests/test_validate.py` (build a minimal `Proof` the way the existing tests do; adapt the constructor call):

```python
def test_warns_empty_segment_caption(minimal_proof_factory):
    proof = minimal_proof_factory(source_text="\\State a\n\\tfsegment{}\n\\State b\n")
    warnings = validate_proof(proof, Path("."))
    assert any("empty" in w.lower() and "tfsegment" in w for w in warnings)


def test_warns_segment_inside_if(minimal_proof_factory):
    src = "\\If{c}\n\\tfsegment{Bad}\n\\EndIf\n"
    proof = minimal_proof_factory(source_text=src)
    warnings = validate_proof(proof, Path("."))
    assert any("depth" in w.lower() or "inside" in w.lower() for w in warnings)


def test_warns_crop_without_segments(minimal_proof_factory):
    proof = minimal_proof_factory(source_text="\\State a\n", crop_default=True)
    warnings = validate_proof(proof, Path("."))
    assert any("no" in w.lower() and "tfsegment" in w for w in warnings)
```

If `test_validate.py` has no `minimal_proof_factory` fixture, add one to `tests/conftest.py` that builds a `Proof` with a single game and the given `source_text`/`crop_default`.

- [ ] **Step 2: Run test to verify it fails**

Run: `.venv/bin/pytest tests/test_validate.py -k "segment or crop_without" -v`
Expected: FAIL (warnings not produced).

- [ ] **Step 3: Write minimal implementation**

In `texfrog/texfrog/validate.py`, before `return warnings`, add:

```python
    from .filter import SEGMENT_RE

    src_lines = proof.source_text.split("\n")
    seg_count = 0
    depth = 0
    _OPENERS = (r"\If", r"\For", r"\While")
    _CLOSERS = (r"\EndIf", r"\EndFor", r"\EndWhile")
    for ln in src_lines:
        stripped = ln.strip()
        m = SEGMENT_RE.match(ln)
        if m:
            seg_count += 1
            if not m.group("caption").strip():
                warnings.append(
                    f"{proof.source_name}: \\tfsegment has an empty caption."
                )
            if depth != 0:
                warnings.append(
                    f"{proof.source_name}: \\tfsegment{{{m.group('caption')}}} "
                    f"is at block depth {depth} (inside \\If/\\For/\\While); "
                    "segments must start at depth 0 to crop safely."
                )
        # crude block-depth tracking for algpseudocodex-style bodies
        for closer in _CLOSERS:
            if stripped.startswith(closer):
                depth = max(0, depth - 1)
        for opener in _OPENERS:
            if stripped.startswith(opener):
                depth += 1
    if proof.crop_default and seg_count == 0:
        warnings.append(
            f"{proof.source_name}: \\tfcropdefault is on but the source has no "
            "\\tfsegment markers, so cropping cannot shrink the listing."
        )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `.venv/bin/pytest tests/test_validate.py -k "segment or crop_without" -v`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add texfrog/validate.py tests/test_validate.py tests/conftest.py
git commit -m "feat(validate): warn on bad \\tfsegment placement and crop-without-segments"
```

---

## Task 8: Roll out in the IKEv2 proof (parent repo)

**Files (in `ikev2-analysis/`, not `texfrog/`):**
- Modify: `proof/proof.tex`, `proof/CLAUDE.md`

**Interfaces:** consumes the finished LaTeX + Python features (Tasks 1–7).

- [ ] **Step 1: Branch the parent repo**

```bash
cd /Users/thom/git/ikev2-analysis
git checkout -b feature/segment-crop
```

- [ ] **Step 2: Add markers + crop default**

In `proof/proof.tex`, add `\tfcropdefault{on}` in the preamble area (near the `\tfgames` registrations, before `\begin{document}`). Insert `\tfsegment{...}` markers in the `kind` `tfsource` (lines ~244–429) at these depth-0 boundaries, immediately before the corresponding `\LComment`:
  - before `\LComment{Initiator session ...}` (line 246): `\tfsegment{Initiator: SA\_INIT}`
  - before `\If{\MKE ...}` (line 264): `\tfsegment{Initiator: additional key exchange}`
  - before `\If{\PPK ...}` (line 286): `\tfsegment{Initiator: PPK}`
  - before `\LComment{\IKEAUTH; ...}` (line 323): `\tfsegment{Initiator: IKE\_AUTH}`
  - before `\LComment{First child SA ...}` (line 336): `\tfsegment{Initiator: first child SA}`
  - before `\LComment{Responder session ...}` (line 346): `\tfsegment{Responder: SA\_INIT}`
  - before `\If{\MKE ...}` (line 365): `\tfsegment{Responder: additional key exchange}`
  - before `\If{\PPK ...}` (line 385): `\tfsegment{Responder: PPK}`
  - before `\LComment{\IKEAUTH; ...}` (line 411): `\tfsegment{Responder: IKE\_AUTH}`
  - before `\LComment{First child SA ...}` (line 424): `\tfsegment{Responder: first child SA}`

  Ensure each marker sits at block depth 0 (the `\If`/`\EndIf` for MKE and PPK stay entirely within one segment). Optionally add markers to the `cka` source similarly.

- [ ] **Step 3: Set `crop=off` on full-listing renders**

Where a whole listing is intended, add `crop=off`. At minimum the first render `\tfrendergame{kind}{G0}` (line 558) has no diff and is already full. Review the case-opening renders (`G5`, `A2G5`, `A3G5`, `B1G4`, `B2G4`, and `C0`) and set `crop=off` on any you want shown in full; leave the rest to crop.

- [ ] **Step 4: Validate**

Run: `uv run --project texfrog texfrog check proof/proof.tex`
Expected: no new warnings about segment placement (fix any depth warnings by moving a marker).

- [ ] **Step 5: Build the PDF and eyeball it**

Run: `cd proof && pdflatex -interaction=nonstopmode -file-line-error proof.tex`
Expected: exit 0; open `proof.pdf` and confirm cropped hops show only changed segments with `⋯ … (unchanged) ⋯` stubs, and full renders are unaffected.

- [ ] **Step 6: Build the HTML viewer and eyeball it**

Run: `uv run --project texfrog texfrog html build proof/proof.tex -o /tmp/proof-html`
Expected: per-game SVGs are cropped; open `/tmp/proof-html/index.html`.

- [ ] **Step 7: Commit (parent repo)**

```bash
cd /Users/thom/git/ikev2-analysis
git add proof/proof.tex proof/CLAUDE.md
git commit -m "proof: enable segment cropping for per-hop renderings"
```

(`proof/CLAUDE.md` edit is done in Task 9, Step 3; commit together or separately.)

---

## Task 9: Documentation

**Files:**
- Modify: `texfrog/docs/writing-proofs.md`, `texfrog/docs/system-design.md`, `texfrog/README.md`, `texfrog/.claude/CLAUDE.md`, `proof/CLAUDE.md` (parent repo).

- [ ] **Step 1: User-facing docs (`texfrog/`)**

- `docs/writing-proofs.md`: add a "Cropping long listings" section covering `\tfsegment{Caption}`, the depth-0 placement rule, `\tfcropdefault{on|off}`, the `crop=on|off` key (PDF-only override; HTML follows the global default), and redefining `\tfsegmentstub`.
- `README.md`: add `\tfsegment`, `\tfcropdefault`, `crop=` key, and `\tfsegmentstub` to the command reference.
- `.claude/CLAUDE.md`: add the new commands to the "Key commands" list and note the HTML vs PDF crop nuance.

- [ ] **Step 2: Architecture docs (`texfrog/`)**

- `docs/system-design.md`: document the three-pass cropped render (record → scan → crop-render), the new scan mode, the untagged-line suppression mechanism actually chosen in Task 6, and the Python crop pipeline (`split_into_segments`/`compute_active_segments`/`crop_to_active_segments`/`_apply_crop`).

- [ ] **Step 3: Proof docs (parent repo)**

- `proof/CLAUDE.md`: in "How TeXFrog works here" and the extension checklist, mention `\tfsegment` markers, `\tfcropdefault{on}`, and that adding a hop may need a `crop=off` decision and that markers must stay at depth 0.

- [ ] **Step 4: Commit**

```bash
cd /Users/thom/git/ikev2-analysis/texfrog
git add docs/writing-proofs.md docs/system-design.md README.md .claude/CLAUDE.md
git commit -m "docs: document segment cropping (\\tfsegment, \\tfcropdefault)"
cd /Users/thom/git/ikev2-analysis
git add proof/CLAUDE.md
git commit -m "docs(proof): note segment cropping in proof CLAUDE.md"
```

---

## Self-Review

**Spec coverage:** `\tfsegment` (T1/T5), `\tfcropdefault` (T3/T5), `crop=` key (T5/T6), `\tfsegmentstub` (T4/T5), changed-only + stub behavior (T2/T6), LaTeX 3-pass (T6), Python HTML crop (T4), validator warnings (T7), proof rollout (T8), docs (T9), removal marks segment active (T2 `compute_active_segments`, T5 scan handler). Covered.

**Known risk (flagged in T6):** untagged-line suppression inside inactive segments is the one genuinely uncertain LaTeX mechanism; T6 Step 3b instructs the executor to validate box-capture on the fixture first and fall back to a documented "tag croppable lines" limitation if it misbehaves. This is the task most likely to need iteration.

**Placeholders:** none (T6 Step 3b intentionally presents a primary mechanism + explicit fallback, both concrete).

**Type consistency:** `Segment(caption, lines)`, `split_into_segments`, `compute_active_segments`, `crop_to_active_segments(...) -> (list, list[int])`, `_apply_crop(...) -> (list, set)`, `Proof.crop_default`, `PackageProfile.html_tfsegmentstub()` used consistently across tasks.
