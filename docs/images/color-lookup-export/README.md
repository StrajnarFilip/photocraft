# Color Lookup export warning (#8839)

Original native offscreen captures of PhotoCraft, without cropping or image edits.
Both use the same 1280 × 800 point viewport, 1× display scale, Pro Medium theme,
and a 640 × 400 RGB/16 document. Its background is #5078a0. The adjustment is an
original 2³ inverse RGB table, trilinear interpolation, Dither off.

The fixture drives the normal Save As flow with the real PSD exporter. Its writer
discards the bytes after export, so no user document or filesystem destination is
needed. The screenshot shows the actual export result notice, not an injected
warning or an imitation dialog.

- `before.png`: upstream production code at `5f9b4391874aabc9deb2fcb905267613e4d93ec8`
  with this PR's capture fixture. Save reports `warnings: []`.
- `after.png`: this change. Save reports one warning naming Color Lookup 1 and the
  missing transform profile, with the `.pcraft` / flattened PNG or TIFF workaround.

The canvas remains unchanged. These images demonstrate the diagnostic only;
Photoshop was not run and native transform-profile generation remains open.

## Reproduce

```sh
CARGO_TARGET_DIR=target/agent-6084 \
PHOTOCRAFT_LOOKUP_EXPORT_SNAPSHOT=/tmp/color-lookup-export.png \
cargo test -p photocraft-ui-egui --lib \
  color_lookup_export_compatibility_snapshot -- --nocapture
```

Require `1 passed` and a nonempty PNG. The test prints the actual PSD export
result. To reproduce the baseline, use a separate checkout at the revision above,
copy this PR's `crates/ui-egui/src/file_open/tests.rs` into it, and run the same
command with a different output path. The IO regression
`generated_and_reopened_lookup_exports_warn_about_the_missing_transform_profile`
fails on that baseline (`U8 lookup.psd: []`) and passes with this change.

The tests cover new tables, re-saves of profile-less blocks, original opaque
profile-byte preservation and regenerated lookup settings, in PSD/PSB at
8/16/32-bit depth. Preservation checks do not validate opaque ICC bytes.

See [validation.md](validation.md) for results, exact commands and baseline
Clippy, UI and SVG corpus blockers.

These are original PhotoCraft assets, MIT OR Apache-2.0, with licenses alongside.
No third-party artwork or personal photos are used.
