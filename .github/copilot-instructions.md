# ParaCrop – AI Agent Instructions

## Project Overview
ParaCrop is a **single-page web app** for batch image cropping with multiple aspect ratio presets. It's built as a vanilla JS application (no framework) using Cropper.js for interactive cropping, with export capabilities that support custom naming conventions and multi-format outputs.

## Architecture & Key Patterns

### Single-File Application Structure
- **[index.html](../index.html)**: Contains all HTML markup, JavaScript application logic, and inline event handlers (919 lines)
- **[styles.css](../styles.css)**: All styling in a single CSS file using CSS custom properties for theming
- **[presets.json](../presets.json)**: Configuration-driven preset definitions with aspect ratios, overlay specs, and export settings

### State Management Pattern
ParaCrop uses **localStorage-persisted state** (`STORAGE_KEY = 'paracrop_state_v1'`) to preserve user work across sessions:
- `cropMemory` – stores crop coordinates per preset ID
- `linkedState` – tracks which presets have linked center-movement
- `adjustedState` – flags presets the user has manually modified
- `exportScaleEnabled` – per-preset toggle for scaled exports
- `collapsedGroups` – tracks expand/collapse state of preset groups

**Critical**: State is cleared on page reload (detected via `performance.navigation`) but preserved on navigation away/back.

### Preset System Architecture
Presets define crop configurations loaded from [presets.json](../presets.json):
```javascript
{
  "preset_id": {
    "title": "Display Name",
    "group": "Horizontal|Vertical|Custom",  // Groups presets in sidebar
    "ratio": 1.777,                         // Aspect ratio (null for free crop)
    "minSize": 1080,                        // Minimum dimension warning threshold
    "prefix": "16x9_poster",                // Used in export filename
    "overlays": [                           // UI guide overlays (% based)
      { "top": 5, "left": 5, "width": 10, "height": 10 }
    ],
    "exportSize": {                         // Target export dimensions
      "width": 1920,
      "height": 1080,
      "allowUpscale": true                  // Enable upscaling to target size
    }
  }
}
```

The `__meta__` key in presets.json provides template/country options for filename generation, plus quality settings (`jpegQuality`, `thumbnailQuality`).

## Core Functionality

### Linked Preset Movement
**Most complex feature**: When presets are "linked" (🔗 button), moving the crop center in one preset applies the same delta to all other linked presets. This enables consistent subject positioning across multiple aspect ratios.
- Implemented in `applyCenterDeltaToLinked(dx, dy, sourceId)` (lines 637-669)
- Centers tracked in `lastCenters` object
- Respects image bounds and preset aspect ratios

### Export Scaling System
Each preset can have a target `exportSize`. The size badge (shown in preset button) toggles scaled export:
- **Active badge** (yellow): Export will scale to preset's target dimensions
- **Inactive badge** (gray): Export at crop resolution
- Function `getCroppedCanvasScaled()` (lines 682-768) implements progressive downscaling for quality

### File Naming Convention
Export filenames follow pattern: `{Template}_{Country}_{Year}_{PresetPrefix}_{OriginalBasename}.jpg`
- Built by `buildExportFilename(presetId)` (lines 898-910)
- All components sanitized via `sanitizeToken()` (removes special chars, replaces spaces with dashes)

## Development Workflows

### No Build Process
This is a **pure static web app** – no bundler, transpiler, or build step. Serve with any HTTP server:
```bash
python3 -m http.server 8000
# or
npx serve .
```

### External Dependencies (CDN-loaded)
- Cropper.js 1.5.13 (image cropping UI)
- JSZip 3.10.1 (batch download as ZIP)
- FileSaver.js 2.0.5 (browser downloads)

**No npm/package.json** – all deps loaded via `<script>` tags.

### Testing Strategy
Manual testing checklist (no automated tests):
1. Upload an image → verify all presets apply correct aspect ratios
2. Adjust crop → verify size badge warnings for too-small crops
3. Link multiple presets → move one, verify others move together
4. Toggle size badges → verify export scaling on/off
5. "Review All Adjusted" → verify only modified presets show in gallery
6. Download ZIP → verify filename pattern and contents

## Critical Implementation Details

### UI Organization
**Collapsible Groups**: Preset groups in the sidebar can be expanded/collapsed by clicking the group label. State persists in localStorage. First group expands by default, others collapsed.

### State Flags & Button States
Three independent systems control preset button styling:
- `.active` class: Currently selected preset (only one at a time)
- `.is-modified` class: User has adjusted this preset's crop (shows green status dot)
- `.linked` class: Preset participates in linked movement (shows link icon styling)

**Group Label States**: Group labels turn green (`.has-modified` class) when any preset in that group has been adjusted. Updated dynamically by `updateGroupLabelState(presetId)` helper function.

### Size Badge Warning States
Size badges change color based on crop validation:
- `.too-small-upscale` (red): Crop below minSize, but allowUpscale=true
- `.too-small-no-upscale` (orange): Crop below minSize, allowUpscale=false
- Cropper overlay also gets `.too-small` class for visual feedback

### Progressive Downscaling Algorithm
Export quality optimization (lines 721-763): Repeatedly halve canvas size until near target, then final high-quality draw. Only triggers when reduction factor > 2×.

## Common Modification Patterns

### Adding a New Preset
1. Add entry to [presets.json](../presets.json) with required fields
2. No code changes needed – UI auto-generates from config
3. Test aspect ratio math (height = width / ratio)

### Changing Export Quality
Modify the `jpegQuality` value in [presets.json](../presets.json) `__meta__` section (default: `0.95` for exports, `0.5` for gallery thumbnails). All export functions read from this centralized config.

### Adding UI Overlay Guides
Update preset's `overlays` array in presets.json with percentage-based boxes:
```json
"overlays": [
  {"top": 5, "left": 5, "width": 10, "height": 10}  // Title-safe area
]
```
Rendered by `updateUIOverlay()` (lines 487-508).

## Code Style Conventions

- **No frameworks/classes**: Pure functional JS with closure-based state
- **Inline event handlers**: `onclick` attributes in HTML for actions
- **Global functions**: Export-related functions attached to `window` for inline handler access
- **Try-catch for safety**: Key UI operations wrapped to prevent cascading failures
- **CSS custom properties**: All colors in `:root` for easy theming adjustments
