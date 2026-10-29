# Validation for #8839

Linux, Rust `1.95.0 (59807616e 2026-04-14)`, based on
`5f9b4391874aabc9deb2fcb905267613e4d93ec8`. Commands below used
`CARGO_TARGET_DIR=target/agent-6084`. Optional craft-fonts were unavailable.

## Passing checks

| Command | Result |
| --- | --- |
| `cargo test -p photocraft-io` | 924 passed, 9 ignored |
| `cargo xtask test-corpus -p io -- --skip svg::an_embedded_svg_gets_only_the_nesting_depth_the_drawing_left` | 968 passed, 9 ignored, 1 explicitly filtered |
| Native capture command in [README.md](README.md) | 1 passed; real Save As returns one layer-specific warning |
| `cargo xtask layers` | 31 crates, no violations |
| `cargo xtask wasm` | All 24 selected crates and the codecs HEIF feature check passed |
| `cargo fmt --all -- --check` | Passed |
| `git diff --check` | Passed |

The new IO regressions cover PSD/PSB at 8/16/32-bit depth, profile-less re-saves,
opaque original profile-byte preservation, and warnings after lookup edits.
PhotoCraft's before/after composites remain equal within 1/255. Copying the IO
regression to the base revision fails because its initial export has no warning.

## Baseline blockers

The complete validation suite is **not green** on this machine.

### Corpus SVG stack overflow

The unfiltered `cargo xtask test-corpus -p io` aborts with a stack overflow in
`svg::an_embedded_svg_gets_only_the_nesting_depth_the_drawing_left`. This test
uses a deliberately small thread stack. The same isolated test also aborts after
restoring both changed IO production files to the base revision:

```sh
cargo test --release -p photocraft-io --test it --features corpus,heif \
  svg::an_embedded_svg_gets_only_the_nesting_depth_the_drawing_left \
  -- --exact --nocapture
```

The passing corpus run above excludes exactly that case; it is not a claim that
the unfiltered corpus gate passes.

### Clippy

Strict `cargo clippy -p photocraft-io -p photocraft-ui-egui --all-targets -- -D warnings`
is blocked by unchanged code. Subsequent scoped checks exposed these existing
diagnostics (the listed source locations match the base revision):

| Location | Lint |
| --- | --- |
| `crates/text/src/psd_path.rs:157` | `overly_complex_bool_expr` |
| `crates/codecs/src/codecs/via_image.rs:112` | `nonminimal_bool` |
| `crates/io/src/psd_export/document.rs:106` | `drop_non_drop` |
| `crates/io/src/smart_map/import.rs:780` | `nonminimal_bool` |
| `crates/ui-egui/src/dialogs/vp_ui.rs:989` | `nonminimal_bool` |
| `crates/ui-egui/src/panels/blend_preview.rs:104` | `nonminimal_bool` |
| `crates/ui-egui/src/canvas/strokes.rs:435,440` | `collapsible_match` |
| `crates/ui-egui/src/widgets/text_field.rs:29` | `collapsible_match` |
| `crates/ui-egui/src/dialogs/color_picker_ui/tests.rs:103` | `manual_range_contains` |

The supplemental IO-only check passes with its two existing lint classes allowed:

```sh
cargo clippy -p photocraft-io --all-targets --no-deps -- \
  -D warnings -A clippy::drop_non_drop -A clippy::nonminimal_bool
```

The UI-only check also passes with its existing lint classes allowed:

```sh
cargo clippy -p photocraft-ui-egui --all-targets --no-deps -- \
  -D warnings -A clippy::collapsible_match \
  -A clippy::manual_range_contains -A clippy::nonminimal_bool
```

These allowances are command-line diagnostics only; no lint suppression was
added to the source.

### Full native UI suite

`cargo test -p photocraft-ui-egui --lib` reports 4,182 passed, 21 failed and 18
ignored. The failure names match the baseline run. With unchanged production
sources, 20 of those cases also fail individually; the remaining font-formatting
case passes in isolation after hitting the harness step limit in the full run.
Failures involve unavailable CJK/Arabic glyphs, the software GPU Camera Raw
expectation, and harness timing. The focused Save As capture passes.

## Scope

Photoshop was not run. This change implements the issue's warning alternative;
it does not generate or validate ICC transform profiles, establish native lookup
rendering parity, or raise any oracle floor. Native profile generation and
Photoshop reopen verification remain open.
