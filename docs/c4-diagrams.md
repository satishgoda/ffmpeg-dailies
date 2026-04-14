# C4 Mermaid Diagrams

These diagrams describe the repository from three useful viewpoints: system context, container responsibilities, and internal Python components.

## 1. System context

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "background": "#0b1020",
    "primaryColor": "#1d4ed8",
    "primaryTextColor": "#f8fafc",
    "primaryBorderColor": "#93c5fd",
    "lineColor": "#60a5fa",
    "secondaryColor": "#7c3aed",
    "tertiaryColor": "#0f766e",
    "fontSize": "15px"
  }
}}%%
C4Context
  title System Context — ffmpeg-dailies
  Person(user, "Pipeline TD / Artist", "Runs dailies from the CLI, Python API, or slate editor")
  System(dailies, "ffmpeg-dailies", "Python toolkit that resolves config, metadata, filtergraphs, and FFmpeg commands")
  System_Ext(ffmpeg, "FFmpeg + ffprobe", "External binaries used for probing, preview generation, and encoding")
  System_Ext(assets, "Project assets", "Input media, YAML config, fonts, slate templates, and OCIO configs")
  System_Ext(output, "Rendered dailies", "QuickTime or MP4 review files with slate metadata and burn-ins")

  Rel(user, dailies, "Configures, previews, and runs")
  Rel(dailies, assets, "Reads")
  Rel(dailies, ffmpeg, "Builds commands for and launches")
  Rel(ffmpeg, output, "Writes")
  Rel(output, user, "Delivers for review")

  UpdateLayoutConfig($c4ShapeInRow="2", $c4BoundaryInRow="1")
```

**Reading tip:** the toolkit itself is the orchestration layer; FFmpeg remains the execution engine.

## 2. Container view

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "background": "#0f172a",
    "primaryColor": "#2563eb",
    "primaryTextColor": "#eff6ff",
    "primaryBorderColor": "#bfdbfe",
    "lineColor": "#93c5fd",
    "secondaryColor": "#8b5cf6",
    "tertiaryColor": "#0f766e",
    "fontSize": "14px"
  }
}}%%
C4Container
  title Container View — runtime building blocks
  Person(user, "Pipeline TD / Artist", "Starts renders or edits slate layouts")
  System_Ext(ffmpeg, "FFmpeg + ffprobe", "External media processing tools")
  System_Ext(filesystem, "Filesystem", "Media, configs, fonts, templates, and rendered outputs")

  Boundary(repo, "ffmpeg-dailies") {
    Container(entry, "CLI + Python API", "Python", "Accepts runtime inputs and creates render requests")
    Container(config, "Configuration parser", "Python", "Loads YAML and resolves globals, slate, burn-ins, codecs, OCIO, and metadata rules")
    Container(rendering, "Rendering engine", "Python", "Builds DailiesContext, filtergraphs, FFmpeg commands, and output metadata")
    Container(gui, "Slate layout GUI", "FastAPI + HTML/JS", "Interactive editor for slate positioning, previews, and saving YAML changes")
  }

  Rel(user, entry, "Runs")
  Rel(user, gui, "Uses in browser")
  Rel(entry, config, "Loads config")
  Rel(config, rendering, "Supplies typed settings")
  Rel(entry, rendering, "Passes overrides and resolved inputs")
  Rel(gui, config, "Reads and writes YAML")
  Rel(gui, rendering, "Reuses preview and layout logic")
  Rel(rendering, ffmpeg, "Launches")
  Rel(rendering, filesystem, "Reads inputs and writes deliverables")
  Rel(gui, filesystem, "Reads configs and preview assets")

  UpdateLayoutConfig($c4ShapeInRow="2", $c4BoundaryInRow="1")
```

**Reading tip:** the GUI is a companion container that shares the same parsing and preview logic instead of re-implementing rendering rules.

## 3. Component view

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "background": "#111827",
    "primaryColor": "#1e40af",
    "primaryTextColor": "#f9fafb",
    "primaryBorderColor": "#93c5fd",
    "lineColor": "#60a5fa",
    "secondaryColor": "#7c3aed",
    "tertiaryColor": "#059669",
    "fontSize": "14px"
  }
}}%%
C4Component
  title Component View — ffmpeg_dailies package
  Person(user, "Pipeline TD / Artist", "Runs render jobs or edits slate layouts")
  System_Ext(ffmpeg, "FFmpeg + ffprobe", "External binaries")

  Container_Boundary(app, "ffmpeg_dailies package") {
    Component(entry, "Entry points", "__init__.py, cli.py, __main__.py", "Normalizes CLI/API input and starts the render workflow")
    Component(config, "Config parsing", "config.py", "Loads YAML and converts sections into typed config objects")
    Component(utils, "Input + metadata utilities", "utils.py", "Resolves sequences, implicit metadata, timecode, reel names, and source facts")
    Component(models, "Runtime models", "models.py", "Carries DailiesContext and all supporting dataclasses")
    Component(filters, "Filtergraph builder", "filtergraph.py", "Creates slate, thumbnail, OCIO, scaling, and burn-in filter chains")
    Component(execute, "Command builder + executor", "execute.py", "Validates FFmpeg filters, assembles command arguments, and runs subprocesses")
    Component(gui, "GUI service", "gui/app.py", "Serves editor state, saves YAML, and generates previews")
  }

  Rel(user, entry, "Runs CLI / API")
  Rel(user, gui, "Opens browser editor")
  Rel(entry, config, "Loads config")
  Rel(entry, utils, "Resolves media and metadata")
  Rel(config, models, "Builds settings")
  Rel(utils, models, "Enriches context")
  Rel(models, filters, "Supplies DailiesContext")
  Rel(filters, execute, "Provides filter_complex")
  Rel(execute, ffmpeg, "Executes")
  Rel(gui, config, "Reads and writes")
  Rel(gui, utils, "Builds preview metadata")
  Rel(gui, execute, "Uses preview helpers")

  UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

**Reading tip:** `models.py` is the shared contract between parsing, utility, filtergraph, and execution layers.
