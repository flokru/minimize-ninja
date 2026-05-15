# MinimizeNinja

MinimizeNinja. Compress Apple Keynote presentation files like a 🥷

[**Try out!**](https://minimize.ninja/)

## Abstract

We developed MinimizeNinja out of necessity, because Keynote does not allow creating proper-looking yet reasonably sized PDF files. For certain Keynote files, the resulting PDFs are either too large or they look shitty. You can even get the worst of both and end up with shitty-looking _and_ large files. 🎉

MinimizeNinja is a Python-based command line tool that unpacks Keynote files, analyzes graphic files used inside the presentation and utilizes several optimizations to achieve a smaller Keynote file. Exporting PDFs from these reduced files results in smaller PDFs that still look good.

## Details

MinimizeNinja can be used to optimize Keynote files. Either to reduce the file size of the Keynote file itself (in order to keep this reduced yet still high quality Keynote file) or – more aggressively – to create an even smaller Keynote file with reasonable quality impact to export this to a small PDF file that can be sent to other people via mail. In the latter case, keeping this reduced Keynote file is _not_ advisable as the graphics quality will be impacted.

These optimizations are performed:

1. All TIFF files within the Keynote file are converted to JPEG and PNG, replacing the TIFF by the smaller of the latter. As TIFF files inside Keynote presentations are rarely compressed, this usually saves a lot of space.
2. Optional, applicable for PDF export: All PNG files are test-converted to JPEG. In case the JPEG variant is smaller than the PNG, the JPEG is being kept. As PNG is a lossless file and JPEG uses lossy (and thus quality-degrading) compression, this will introduce some mild quality loss.
3. All graphics are checked for the resolution they are used with inside the presentation. In case the image resolution is much larger than the used resolution within the presentation, the files are resized so that they still stay crisp on 4K/Retina displays, but do not waste space unnecessarily. E.g. if you add a 15 megapixel photo as a small stamp graphic inside a Keynote slide of 300 x 400 pixels, MinimizeNinja will resize the file to 600 x 800 pixels (reducing 15 MP -> 0.5 MP).
4. Image optimization packages like MozJPEG and Oxipng are run on all images. These tools try to compress the files even more, get rid of unnecessary metadata etc. In many cases, these tools can reduce files by additional 5–40% without losing quality.

## Installation and Requirements

### MozJPEG

You need to install MozJPEG:

```
brew install mozjpeg
```

or using a nix-shell:

```
nix-shell -p mozjpeg
```

Afterwards, there will be a message telling you that mozjpeg is keg-only, which means it was not symlinked into `/opt/homebrew`. Therefore you need to execute the printed line below to add the mozjpeg binaries to your `$PATH`.

### Installing MinimizeNinja with uv (recommended)

[uv](https://docs.astral.sh/uv/) is the recommended way to install MinimizeNinja, since it handles the Python version, the virtual environment, and the git-based `keynote-parser` dependency for you.

Install uv (if you haven't already):

```
brew install uv
```

Then, from the project root, sync the environment:

```
uv sync
```

You can now run the CLI directly via uv:

```
uv run mn --help
```

If you prefer an installed entry point, install the package as a tool:

```
uv tool install .
```

This makes `mn` available on your `$PATH`.

### Installing with pip

If you don't want to use uv, a plain pip install also works. Make sure you have a recent pip and create a virtual environment first, then call:

```
pip install .
```

If you're on an Apple Silicon Mac you might have to install `python-snappy` before installing this project:

```
CPPFLAGS="-I/opt/homebrew/include -L/opt/homebrew/lib" pip install python-snappy
```

### Keynote-parser fork

MinimizeNinja depends on the **point8 fork** of `keynote-parser`, _not_ the upstream package on PyPI:

> [https://github.com/point8/keynote-parser](https://github.com/point8/keynote-parser)

The fork is pinned in `pyproject.toml` and will be pulled in automatically by `uv sync` or `pip install .`. It tracks current Keynote versions (14.4 / 14.5 at the time of writing) and contains the protobuf fixes needed for MinimizeNinja to work reliably.

## Usage

MinimizeNinja exposes a single `mn` entry point with three subcommands.

### Global options

```
mn [OPTIONS] COMMAND [ARGS]...
```

| Option | Description |
| --- | --- |
| `-v`, `--verbose` | Increase verbosity (`-v` = INFO, `-vv` = DEBUG). Repeat for more output. |
| `--log-file FILE` | Tee all log output into `FILE` in addition to the console. |

### `mn slim` — shrink a Keynote file

Get a Keynote file into shape by losing unnecessary weight. Produces `<stem>_tiffy.key` next to the input file. With `--export-pdf` (or a non-zero `--quality` preset) it additionally drives Keynote.app via AppleScript to export `<stem>.pdf`.

```
mn slim PRESENTATION.key [OPTIONS]
```

| Option | Default | Description |
| --- | --- | --- |
| `-q`, `--quality INT` | `0` | Quality preset `[0–4]`. `0` keeps the explicit flag values; `1`–`4` are progressively more aggressive presets that also force PDF export. See the table below. |
| `-p`, `--export-pdf` | off | Export the optimized Keynote to PDF after slimming (uses Keynote.app on macOS). |
| `--pdf-all-stages` | off | Keep all animation stages in the exported PDF (one page per build stage). |
| `--keep-unpacked` | off | Do not delete the unpacked working directory (useful for debugging). |
| `--resize-factor FLOAT` | `2.0` | Keep images at this multiple of their displayed size (`2.0` = Retina-friendly). |
| `--jpeg-compression INT` | `85` | JPEG quality (0–100). Values below 80 will trigger a quality warning. |
| `--png-convert` | off | Also test-convert PNG files to JPEG and keep whichever is smaller (lossy). |

**Quality presets** (`-q`):

| `-q` | `resize_factor` | `jpeg_compression` | `png_convert` | `export_pdf` |
| --- | --- | --- | --- | --- |
| 0 | _flag value_ (2.0) | _flag value_ (85) | _flag value_ (off) | _flag value_ (off) |
| 1 | 2.0 | 80 | on | on |
| 2 | 1.5 | 75 | on | on |
| 3 | 1.0 | 70 | on | on |
| 4 | 0.75 | 65 | on | on |

> ⚠️ Presets `1`–`4` are intended for one-shot PDF exports. Don't keep the resulting `_tiffy.key` as your master file — images are downscaled and re-compressed lossily.

Examples:

```
mn -v slim PRESENTATION.key                  # safe, lossless-ish; writes PRESENTATION_tiffy.key
mn slim PRESENTATION.key -q 2                # smaller PDF, decent quality
mn slim PRESENTATION.key -p --pdf-all-stages # export PDF including animation stages
mn slim PRESENTATION.key --png-convert       # additionally try PNG→JPEG conversion
```

### `mn autopdf` — batch-export PDFs across quality presets

Runs `slim` repeatedly for a range of quality presets and writes `<stem>_q<N>.pdf` for each. Handy when you want to pick the best size/quality trade-off by eyeballing the results.

```
mn autopdf PRESENTATION.key [OPTIONS]
```

| Option | Default | Description |
| --- | --- | --- |
| `-q1`, `--quality1 INT` | `0` | Starting quality preset (inclusive). |
| `-q2`, `--quality2 INT` | `4` | Ending quality preset (inclusive). |
| `--pdf-all-stages` | off | Keep all animation stages in the exported PDFs. |

Example:

```
mn autopdf PRESENTATION.key            # writes PRESENTATION_q0.pdf … PRESENTATION_q4.pdf
mn autopdf PRESENTATION.key -q1 1 -q2 3
```

### `mn fix-fa-duotones` — fix FontAwesome Duotone glyphs

Some FontAwesome 6 Duotone glyphs only render correctly in Keynote when their character style is marked **bold**. This command walks the document stylesheet and flips every occurrence of `FontAwesome6Duotone-Solid` to bold, then repacks the file as `<stem>_tiffy.key`.

```
mn fix-fa-duotones PRESENTATION.key
```

## Troubleshooting

### Q: I get the message "Unrecognized input file format --- perhaps you need -targa"

Sometimes you'll see messages like:

```
ERROR    cjpeg was unable to run on ImageFile(…):
ERROR    Unrecognized input file format --- perhaps you need -targa
```

This is an error from MozJPEG when it can't parse a particular JPEG file for size optimization. The `-targa` hint is misleading — these files are _not_ TARGA files and it's unclear why MozJPEG rejects them. In every case I checked, the files were perfectly fine and could be opened with every other tool I tried. You can safely ignore these errors; MinimizeNinja will just skip the MozJPEG step for that image and keep its previous version.

### Q: PDF export doesn't run / nothing happens

PDF export shells out to Keynote.app via AppleScript and is therefore **macOS-only**. On other platforms `mn slim` still works, but `--export-pdf` and the `-q 1..4` presets will have no PDF to produce.

### Q: A specific `.key` file fails to unpack or repack

The unpack/repack pipeline lives in the [point8 keynote-parser fork](https://github.com/point8/keynote-parser). If a file blows up there, it's almost certainly a protobuf schema mismatch — try updating to the latest `keynote-parser` from that repo (`uv sync --upgrade-package keynote-parser`), and if the problem persists, open an issue with the failing file attached.
