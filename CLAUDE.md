# CLAUDE.md

## Project Overview

**MinimizeNinja** (`mn`) is a CLI that shrinks Apple Keynote `.key` files by re-encoding and right-sizing the images embedded inside them. The motivating use case is producing smaller, presentable PDFs from Keynote — Keynote's own PDF export tends to be either too large or visually poor.

The internal nickname is "Tiffy" (output files are written as `<stem>_tiffy.key`).

- **Python**: >=3.10
- **Entry point**: `mn` → `minimize_ninja.cli:main`
- **Version**: 0.1.0 in `pyproject.toml` (note: also hardcoded in `cli.py` line 17 and as `0.0.2` in the crash banner — version is drift-prone, update all spots)

## Architecture

```
minimize_ninja/
  cli.py         # Click CLI: `slim`, `autopdf`, `fix-fa-duotones` + slim_file/fix_fa_duotones_file workhorses
  keynote.py     # Core classes: KeynoteFile, TiffyYaml, ImageFile, KeynoteSlide
  common.py      # Logging (rich) and a (mostly empty) read_config() that returns logger + Console
```

There is no config file; `read_config()` just bundles a rich `Console` and the logger into a `resources` dict that gets threaded through everything. The commented-out `~/.minimizeninjarc.yaml` lines suggest this used to be config-driven.

## How the pipeline works

`slim_file()` in `cli.py` is the whole story:

1. **Unpack** the `.key` via `keynote_parser.file_utils.process()` into a temp dir (`/tmp/<uuid8>/`). This explodes the ZIP and converts every `.iwa` protobuf into `.iwa.yaml`.
2. **Load `Index/Metadata.iwa.yaml`** and walk `chunks → archives → objects → datas` to build an `images_dict` keyed by identifier. Each entry is an `ImageFile` pointing at a file in `Data/`.
3. **TIFF conversion** — every TIFF is re-encoded to both PNG and JPEG (via Wand/ImageMagick), the smaller wins. Alpha-channel TIFFs are only converted to JPEG if the alpha is fully opaque (the recent "Better png to jpg conversion" commit). The `preferredFileName`/`fileName` in metadata YAML is updated to match.
4. **PNG → JPEG** (only when `--png-convert` / quality ≥ 1) — same try-both-and-pick-smaller.
5. **Build slide references**: scan every YAML file under `Index/` for `TSD.ImageArchive` and `KN.SlideStyleArchive` (master-slide backgrounds), record where each image is used and at what `originalSize`.
6. **Resize** each image so that its pixel height matches `max_used_ratio × resize_factor` (default `2.0` → keep 2× the displayed size for Retina). Slide-style backgrounds are clamped against 1080p.
7. **Optimize** with `oxipng` (PNG, level 6), `cjpeg` from MozJPEG (JPEG), or `pdfsizeopt` (PDF). Each result is only kept if it shaves at least 2% (`size_tmp < size_resized * 0.98`).
8. **Repack** the directory back into `<stem>_tiffy.key` via `keynote_parser.file_utils.process()`.
9. **Optionally** drive Keynote.app via AppleScript to export a PDF (`PDF image quality: Better`, `skipped slides: false`, `all stages: …`). On success, the intermediate `_tiffy.key` is deleted.

The `--quality` flag (0–4) is a preset that overrides `resize_factor`, `jpeg_compression`, `png_convert`, and forces `export_pdf`:

| q | resize_factor | jpeg | png_convert | export_pdf |
|---|---|---|---|---|
| 0 | (caller value, 2.0) | (caller, 85) | (caller, false) | (caller) |
| 1 | 2.0 | 80 | true | true |
| 2 | 1.5 | 75 | true | true |
| 3 | 1.0 | 70 | true | true |
| 4 | 0.75 | 65 | true | true |

The `slim` CLI exposes q 0–3 in the help; q=4 exists in the `match` but is undocumented. `autopdf` defaults to iterating q1..q2 = 0..4, producing `<stem>_q0.pdf` … `<stem>_q4.pdf`.

## CLI

```bash
mn slim PRESENTATION.key [-q 0..4] [-p] [--pdf-all-stages] [--keep-unpacked] \
                          [--resize-factor 2.0] [--jpeg-compression 85] [--png-convert]
mn autopdf PRESENTATION.key [-q1 0] [-q2 4] [--pdf-all-stages]
mn fix-fa-duotones PRESENTATION.key   # flips FontAwesome6Duotone-Solid style to bold so it renders as Solid
mn -v ...   # -v / -vv for DEBUG logging; --log-file FILE to tee
```

## Key classes (`keynote.py`)

- **`KeynoteFile`** — holds paths, unpack/repack via `keynote_parser.file_utils.process`. Lazy-loads `metadata`, `document_stylesheet`, `images_dict`, `slides`. Note: `path_unpacked` property is defined twice (lines 128 and 132) — harmless but a smell.
- **`TiffyYaml`** — thin `yaml.safe_load` wrapper around a single YAML file with `.yaml` (dict) and `.save()`.
- **`ImageFile`** — tracks `size_original / converted / resized / optimized` independently so reduction stats can be reported per stage. `convert()`, `resize()`, `optimize()` are the three stages; each only acts if the result is actually smaller.
- **`KeynoteSlide`** — represents one slide; resolves to one of `Slide-<id>.iwa.yaml`, `TemplateSlide-<id>.iwa.yaml`, `Slide.iwa.yaml`, or `TemplateSlide.iwa.yaml`. Not used by `slim_file` today but referenced by lazy loaders.

## Dependencies

- **`keynote-parser`** — pinned to the **point8 fork** (`git+https://github.com/point8/keynote-parser.git`), NOT the upstream PyPI package. A local clone lives at `../keynote-parser/`. The upstream README notes incompatibility with files containing tables/charts; the fork is the working version.
- **`wand`** — ImageMagick bindings (image re-encode + resize).
- **`pyoxipng`** — PNG optimizer.
- **MozJPEG** (`cjpeg` on PATH) — external system dependency, install via `brew install mozjpeg` and add the keg-only path to `$PATH`.
- **`applescript`** — drives Keynote.app for PDF export (macOS-only feature).
- `click`, `rich`, `tqdm`, `humanize`, `pyyaml`, `pendulum`, `GitPython`.

On Apple Silicon: `python-snappy` may need to be installed first with Homebrew include/lib flags (see README).

## Working with the keynote-parser fork

The whole project hinges on `keynote_parser.file_utils.process(input, output, replacements=[])`:
- If `input.endswith(".key")` → unzip & decode `.iwa` → write `.yaml` into the output **directory**.
- If `output.endswith(".key")` → re-encode `.yaml` back to `.iwa`, zip into a `.key`.

The fork lives at `../keynote-parser/` (Peter Sobot's project + point8 patches). It supports Keynote 14.4 and 14.5 (`keynote_parser/versions/v14_4`, `v14_5`); the latest is auto-selected. To regenerate proto bindings for a newer Keynote, see that repo's own `CLAUDE.md` and the `dumper/run.py` workflow.

When a Keynote file fails to unpack/repack, the bug is almost always in the proto mapping over there, not in this repo.

## Known quirks / things to know before changing code

- **Hardcoded user path** in `keynote.py:430` — `pdfsizeopt` is invoked from `/Users/fkruse/Documents/Point 8/fkruse/pdfsizeopt/pdfsizeooopt` (yes, with the typo). This branch only runs for `.pdf` assets with slide references; it will silently no-op on other machines.
- **Duplicate `path_unpacked` property** in `KeynoteFile` (line 128 and 132).
- **Crash banner version** says `0.0.2` while `pyproject.toml` says `0.1.0`.
- **AppleScript export prints `script` to stdout** before running (line 312 in `cli.py`) — looks like leftover debug output.
- **`Stopwatch` and `fix_fa_duotones_file`'s `uuid` import** are unused-ish artifacts (uuid is imported in cli.py but only used in the `KeynoteFile.__init__`).
- Conversion **never deletes** the old file unless it differs from the new path; ImageMagick `Image()` writes to a new suffix and the original is `unlink()`ed only when the suffix actually changed.
- The "lost 2% threshold" in `optimize()` means tiny wins from MozJPEG are discarded on purpose.

## Repo layout (working tree)

```
minimize-ninja/
  minimize_ninja/        # the package (see Architecture)
  pyproject.toml         # setuptools, deps, mn entry point
  README.md              # user-facing docs
  *.key                  # local test fixtures (mn_test_big_tiffy.key, test.key, …)
  build/, dist/, *.egg-info/   # build artifacts
  .venv/                 # local venv
```

## Development

There is no test suite, no linter config, no CI in this repo (unlike the keynote-parser fork). Manual smoke test:

```bash
pip install -e .
mn -v slim test.key            # produces test_tiffy.key
mn -v slim test.key -q 2       # quality preset, also writes a PDF via Keynote.app
mn autopdf test.key             # batch PDFs for q0..q4
```
