# Architecture Guide

`ffmpeg-dailies` is a Python toolkit for building VFX dailies in a single FFmpeg-driven pass. It combines YAML configuration, metadata token resolution, slate generation, burn-ins, OCIO colour transforms, and codec presets into one render pipeline.

## What this repository contains

### Primary entry points

| Entry point | File | Purpose |
| :-- | :-- | :-- |
| Python API | `ffmpeg_dailies/__init__.py` | Exposes `render()` for ShotGrid, Nuke, pipeline tools, or custom scripts |
| CLI | `ffmpeg_dailies/cli.py`, `ffmpeg_dailies/__main__.py`, `run_dailies` | Parses command-line flags and forwards them into `render()` |
| Slate GUI | `ffmpeg_dailies/gui/__main__.py`, `ffmpeg_dailies/gui/app.py` | Runs a local FastAPI app for visually editing slate field placement |

### Core runtime modules

| Area | Files | Responsibility |
| :-- | :-- | :-- |
| Configuration parsing | `ffmpeg_dailies/config.py` | Loads YAML and converts `globals`, `slate`, `burnins`, `ocio`, codec profiles, and metadata rules into typed dataclasses |
| Domain models | `ffmpeg_dailies/models.py` | Defines the shared runtime objects such as `DailiesContext`, `SlateConfig`, `BurninConfig`, and `OutputCodecProfile` |
| Input + metadata utilities | `ffmpeg_dailies/utils.py` | Resolves image sequences, derives implicit metadata, and extracts source timecode/reel information |
| Filtergraph construction | `ffmpeg_dailies/filtergraph.py` | Builds the slate, thumbnail PIP, OCIO, scaling, and burn-in filter chains |
| FFmpeg orchestration | `ffmpeg_dailies/execute.py` | Validates FFmpeg capabilities, generates the final command, and executes it |
| Web editor | `ffmpeg_dailies/gui/app.py` | Serves slate state, saves YAML edits, and generates preview images |

## End-to-end render flow

1. A user starts the pipeline through the CLI, Python API, or GUI.
2. The YAML config is loaded and split into typed sections for globals, slate, burn-ins, codec profiles, OCIO, and dynamic metadata.
3. The input path is resolved into either a movie file or an image-sequence pattern with an optional start frame.
4. Metadata is assembled from the config, CLI/API overrides, and implicit values such as file name, date, frame range, version, timecode, and reel name.
5. A `DailiesContext` object is created to carry all resolved runtime state.
6. The filtergraph builder composes the slate, optional thumbnail PIP, video scaling/padding/cropping, OCIO transform, and live burn-ins.
7. The executor turns that context into an FFmpeg command, adds output codec and metadata flags, and runs FFmpeg unless `dry_run=True`.
8. FFmpeg reads the media and assets from disk and writes the final rendered movie.

## Render pipeline responsibilities

### 1. Configuration layer
- Accepts a single YAML file as the main control plane.
- Supports shared defaults in `globals`.
- Supports reusable codec presets in `output_codecs`.
- Keeps slate layout, burn-ins, metadata, and OCIO settings in one place.

### 2. Metadata layer
- Replaces `{Token}` placeholders across slates and burn-ins.
- Allows runtime overrides from the CLI or Python API.
- Auto-populates fields such as file name, delivery date, frame range, version, timecode, and reel name.
- Supports dynamic metadata extraction rules from source paths or existing metadata.

### 3. Composition layer
- Builds a one-frame slate before the main video.
- Optionally extracts a middle-frame thumbnail for a picture-in-picture inset.
- Applies crop, fit, scale, padding, burn-ins, and OCIO in the FFmpeg filtergraph.
- Concatenates the slate and the processed video into the final video stream.

### 4. Delivery layer
- Resolves the FFmpeg binary from config, environment, or `PATH`.
- Applies named codec presets and arbitrary extra FFmpeg flags.
- Writes presentation metadata to the output container.
- Returns the full command even when rendering normally, which makes debugging and integration easier.

## GUI workflow

The GUI is a local editing companion for the slate layout rather than a separate rendering engine.

- `python -m ffmpeg_dailies.gui` starts a local Uvicorn server.
- The browser UI fetches current slate field state from `/api/state`.
- Save requests write updated field positions and selected metadata back into the active YAML file.
- Preview endpoints use FFmpeg to generate a fast background plate, a thumbnail image, or a full slate preview.
- The GUI reuses the same config parsing and rendering primitives as the CLI/API path so layout edits stay aligned with the real render pipeline.

## Repository layout

| Path | Purpose |
| :-- | :-- |
| `ffmpeg_dailies/` | Main package containing the render pipeline and the GUI |
| `ffmpeg_dailies/gui/templates/` | Browser UI assets for the slate editor |
| `docs/` | Static screenshots plus architecture documentation |
| `tests/` | API, GUI, and regression tests |
| `tools/` | Helper utilities for generating slate templates and synthetic test media |
| `sample_config.yaml` | General example configuration |
| `netflix_config.yaml` | More production-styled example configuration |
| `run_dailies` | Thin shell wrapper that launches the CLI from the project virtual environment |

## Suggested reading order

1. Start with [`README.md`](../README.md) for setup, usage, and configuration examples.
2. Read [`docs/graph-diagrams.md`](./graph-diagrams.md) for rendered system, container, and component views.
3. Use [`docs/c4-diagrams.md`](./c4-diagrams.md) only if you need the historical note that explains why the embedded C4 blocks were replaced.
4. Inspect `ffmpeg_dailies/__init__.py` and `ffmpeg_dailies/execute.py` to understand the render entry path.
5. Inspect `ffmpeg_dailies/filtergraph.py` to understand how the slate and burn-ins are composed.
6. Inspect `ffmpeg_dailies/gui/app.py` if you are changing the layout editor or preview API.
