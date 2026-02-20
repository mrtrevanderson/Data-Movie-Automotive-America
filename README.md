# Hyundai Motor America — Lobby Display Renderer

An automated **4K motion infographic** that renders a polished 30-second looping video from a JSON data payload. Built with **Node.js + TypeScript + Remotion**.

Every time you run `npm run run-all`, fresh data is generated and a new video is rendered to `dist/lobby.mp4`. No manual editing required — the visuals are entirely data-driven.

![Lobby Display Preview](preview.gif)

---

## Tech Stack

### Runtime & Language

| Layer | Technology | Why |
|-------|-----------|-----|
| Runtime | **Node.js 18/20** | Server-side JavaScript runtime that runs all scripts, the bundler, and the renderer |
| Language | **TypeScript 5** | Strict typing across all source files — catches data shape mismatches at compile time, not at render time |
| Script runner | **ts-node** | Executes `.ts` files directly without a separate compile step, so `npm run render` just works |

### Video Rendering

| Layer | Technology | Why |
|-------|-----------|-----|
| Animation framework | **Remotion 4** | Lets you write video frames as React components. Each frame is a pure function of its frame number — no timelines, no keyframe editors, no After Effects |
| Frame capture | **Chromium Headless Shell** | Remotion downloads and manages its own bundled Chromium. For each of the 900 frames it navigates the headless browser to that frame, renders the React tree to pixels, and screenshots it as a JPEG |
| Video encoding | **FFmpeg** (bundled) | Remotion includes FFmpeg internally. After all frames are captured, FFmpeg stitches them into an H.264 MP4. You do not need FFmpeg installed on your machine separately |
| Bundler | **Webpack 5** (via `@remotion/bundler`) | Compiles all TypeScript/TSX source files into a browser-compatible bundle that the headless Chromium can load |

### UI & Animation

| Layer | Technology | Why |
|-------|-----------|-----|
| UI components | **React 18** | Each scene and component is a standard functional React component. Remotion intercepts the browser clock and replaces it with a deterministic frame counter |
| Animation primitives | **Remotion `interpolate()`** | Maps a frame number to any animated value (opacity, translateY, scale, dashoffset, etc.) using configurable easing. This is how every motion in the video is expressed |
| Sequencing | **Remotion `<Sequence>`** | Offsets `useCurrentFrame()` to zero at a given start frame, so each scene's animations are relative to their own entry point rather than the global timeline |
| Charts | **Inline SVG** | The line chart (`ChartLine.tsx`) and US map (`USRegionScene.tsx`) are hand-authored SVG rendered directly by React. No chart library dependency — the stroke-dashoffset trick animates the line drawing left to right |
| Map data | **Albers USA SVG paths** | Real geographic state boundary paths in the standard D3/TopoJSON Albers USA projection (viewBox `0 0 960 600`). All 50 states are encoded as inline SVG `<path d="...">` strings — no external map tiles or network calls |
| Styling | **Inline React styles** | All styling is done via JavaScript style objects (no CSS files, no Tailwind). This is required because Remotion renders inside a headless browser and needs deterministic, frame-accurate layout |
| Design tokens | **`theme.ts`** | A single exported object holding all colors, font sizes, spacing, and card dimensions. All scenes import from here so a brand color change propagates everywhere |

### Data Layer

| Layer | Technology | Why |
|-------|-----------|-----|
| Data contract | **TypeScript interfaces** (`types.ts`) | Defines the exact shape of `data.json`. If `fetchData.ts` produces a value that doesn't match, TypeScript catches it before the render runs |
| Data generation | **`fetchData.ts`** | A Node.js script that writes `data.json`. In the POC it generates random-but-realistic automotive values. In production, swap the body for a real API call, database query, or BI export — the contract is just the JSON shape |
| Data passing | **Remotion `inputProps`** | The parsed `data.json` object is passed into the Remotion composition via `inputProps` at render time. React components receive it as typed props. No global state, no environment variables |

### Tooling

| Tool | Role |
|------|------|
| `package.json` scripts | Four commands: `data`, `render`, `run-all`, `studio` |
| `tsconfig.json` | CommonJS modules (required for ts-node), JSX set to `react`, strict mode off for Remotion compatibility |
| `remotion.config.ts` | Sets JPEG image format (faster than PNG) and enables output overwrite |
| Remotion Studio (`npm run studio`) | Opens a browser-based scrubber at `localhost:3000` where you can preview and scrub the video frame-by-frame with hot reload during development |

### Dependency Count

The project has exactly **4 runtime dependencies** (all Remotion packages at the same pinned version) and **4 dev dependencies** (TypeScript, ts-node, and React types). No UI component libraries, no animation libraries, no mapping libraries.

```
dependencies:
  @remotion/bundler   — Webpack bundler for Remotion compositions
  @remotion/cli       — Remotion Studio dev server + config
  @remotion/renderer  — Programmatic frame rendering and FFmpeg encoding
  remotion            — Core hooks: useCurrentFrame, interpolate, Sequence, etc.

devDependencies:
  typescript          — TypeScript compiler
  ts-node             — Direct TypeScript execution
  @types/node         — Node.js type definitions
  @types/react        — React type definitions
```

---

## Requirements

| Tool    | Version                        |
|---------|--------------------------------|
| Node.js | 18.x or 20.x (LTS recommended)|
| npm     | 9+                             |
| FFmpeg  | Bundled by Remotion — no separate install needed |

---

## Quick Start

```bash
# 1. Install dependencies (only needed once)
npm install

# 2. Run the full pipeline: generate data → render video
npm run run-all

# Output: dist/lobby.mp4
```

---

## How It Works — End to End

The system is a three-stage pipeline that runs every time you call `npm run run-all`:

```
┌─────────────────────┐     ┌──────────────────────┐     ┌────────────────────┐
│  1. Data Fetch       │────▶│  2. Remotion Render   │────▶│  3. MP4 Output     │
│  fetchData.ts        │     │  render.ts            │     │  dist/lobby.mp4    │
│  → data.json         │     │  (Chromium + FFmpeg)  │     │  4K H.264          │
└─────────────────────┘     └──────────────────────┘     └────────────────────┘
```

### Stage 1 — Data Fetch (`src/data/fetchData.ts`)

This script simulates a live warehouse or API query. In production you would replace the body of this file with a real database call, REST API request, or BI tool export. For the POC it generates randomized but realistic Hyundai automotive values and writes them to `data.json` at the project root.

**What it produces (`data.json`):**

```json
{
  "company_name": "Hyundai Motor America",
  "tagline": "New Thinking. New Possibilities.",
  "kpis": [
    { "label": "YTD Revenue",   "value": 9840000000, "format": "currency" },
    { "label": "Vehicles Sold", "value": 742819,     "format": "integer"  },
    { "label": "EV Mix",        "value": 0.187,      "format": "percent"  }
  ],
  "trend": {
    "label": "Monthly Unit Sales",
    "points": [64200, 67800, 66100, 70400, 72300, 75100, 73800, 76500, 74900, 79200, 81400, 84600]
  },
  "regions": [
    { "id": "west",      "label": "West",      "units": 58400, "revenue": 2480000000 },
    { "id": "southwest", "label": "Southwest", "units": 41200, "revenue": 1620000000 },
    { "id": "midwest",   "label": "Midwest",   "units": 49700, "revenue": 1890000000 },
    { "id": "southeast", "label": "Southeast", "units": 62100, "revenue": 2210000000 },
    { "id": "northeast", "label": "Northeast", "units": 53800, "revenue": 2040000000 }
  ],
  "last_updated": "2026-02-19T17:46:56.163Z"
}
```

> **To connect real data:** Replace the body of `fetchData.ts` with your API call. As long as the output matches the schema in `src/data/types.ts`, the render will pick it up automatically.

---

### Stage 2 — Remotion Render (`scripts/render.ts`)

This is where the video is actually created. Remotion is a framework that lets you write video animations as **React components**, then renders them frame-by-frame using a headless Chromium browser and encodes the result to MP4 via FFmpeg.

Here is what `render.ts` does step by step:

1. **Loads `data.json`** — reads the fresh data file from disk.
2. **Bundles the composition** — calls `@remotion/bundler` to webpack-compile all the React/TypeScript source files into a temporary bundle that the headless browser can load.
3. **Selects the composition** — asks Remotion to resolve the `LobbyDisplay` composition (defined in `Root.tsx`) and passes the data in as props.
4. **Renders every frame** — Remotion spins up a headless Chromium instance, seeks to each of the 900 frames (30 s × 30 fps), takes a screenshot of the React component tree at that frame, and saves it as a JPEG.
5. **Encodes to MP4** — once all frames are captured, FFmpeg stitches them into an H.264 MP4 at `dist/lobby.mp4`.
6. **Logs progress** — a progress bar is printed to the terminal so you can watch the render proceed.

---

### Stage 3 — Output (`dist/lobby.mp4`)

| Property   | Value              |
|------------|--------------------|
| Resolution | 3840 × 2160 (4K UHD) |
| Frame rate | 30 fps             |
| Duration   | 30 seconds         |
| Codec      | H.264              |
| Loop       | Seamless — the Outro scene fades back to the exact visual state of the Intro |

---

## How the Video Is Structured

### The React Component Model

Every frame of the video is a rendered React component. Remotion replaces the browser clock with a deterministic `useCurrentFrame()` hook. Components read the current frame number and use `interpolate()` to calculate animated values — opacity, position, scale, drawn line length, counted numbers, etc.

This means **animations are pure functions of frame number** — fully reproducible, scrub-able, and testable.

```
frame 0  → opacity: 0.0, translateY: 60px  (title invisible, below center)
frame 12 → opacity: 0.5, translateY: 30px  (title halfway in)
frame 24 → opacity: 1.0, translateY: 0px   (title fully visible)
```

### Scene Sequencing (`LobbyVideo.tsx`)

The main composition divides the 900 frames into five scenes using Remotion's `<Sequence>` component. Each `<Sequence from={N}>` resets `useCurrentFrame()` to zero at frame N, so every scene's internal animations are relative to when that scene starts.

```
Frame  0  → 180   (6 s)   IntroScene      — Hyundai title, tagline, decorative rings
Frame 150 → 390   (8 s)   KPIScene        — 3 metric cards counting up
Frame 360 → 600   (8 s)   TrendScene      — Monthly sales line chart drawing left to right
Frame 570 → 810   (8 s)   USRegionScene   — US map + regional bar chart
Frame 780 → 900   (6 s)   OutroScene      — Title returns, fades to loop point
```

Scenes overlap by 30 frames (1 second) at each transition. During this overlap window, both the outgoing and incoming scene are rendered simultaneously — the outgoing scene fades out while the incoming scene fades in, creating a smooth crossfade.

### Seamless Loop

The Outro scene is visually identical to the Intro scene (same background gradient, grid, decorative rings, typography, and Hyundai blue color palette). In the final 2.5 seconds of the Outro, a matching overlay fades in on top, bringing the frame back to the exact starting visual state. When the video loops, frame 900 and frame 0 are indistinguishable.

---

## Scene Details

### IntroScene
- Dark radial gradient background with a subtle dot grid and two dashed decorative rings
- Company name slides up and fades in
- A horizontal gradient accent line scales in from center
- Tagline (*"New Thinking. New Possibilities."*) fades in last in Hyundai blue

### KPIScene
- Three cards appear with staggered slide-up animations
- Each card shows a KPI label and an animated count-up number that runs from 0 to the real value
  - **YTD Revenue** — formatted as `$X.XXB`
  - **Vehicles Sold** — formatted as a whole number with commas
  - **EV Mix** — formatted as a percentage
- A "Last Updated" timestamp fades in at the bottom right

### TrendScene
- A 12-month SVG line chart (cubic Bézier curves) animates drawing left to right using `stroke-dashoffset`
- Dots appear at each monthly data point as the line reaches them
- A "YTD Growth %" callout badge fades in with a scale animation at the top right
- Month labels (Jan–Dec) appear beneath the x-axis

### USRegionScene
- A full geographic SVG US map in Albers USA projection with real state boundary paths for all 50 states
- States are color-coded by sales region and fade in one region at a time (West → Southwest → Midwest → Southeast → Northeast)
- The top-performing region pulses with a repeating glow effect
- Region name labels appear over the map at each region's geographic centroid
- A ranked horizontal bar chart on the right counts up units sold and revenue per region

### OutroScene
- Mirrors the Intro visually — same layout, same animations, same colors
- Ends with a fade to black that matches the Intro's starting state for a seamless loop

---

## Project Structure

```
data-movie/
├── src/
│   ├── video/
│   │   ├── index.ts                  # Remotion entry point (calls registerRoot)
│   │   ├── Root.tsx                  # Registers the LobbyDisplay composition with Remotion
│   │   ├── LobbyVideo.tsx            # Main composition — scene sequencing and crossfades
│   │   ├── theme.ts                  # All design tokens: colors, fonts, sizes, spacing
│   │   ├── scenes/
│   │   │   ├── IntroScene.tsx        # Hyundai title + tagline (0–6 s)
│   │   │   ├── KPIScene.tsx          # 3 KPI cards with count-up (5–14 s)
│   │   │   ├── TrendScene.tsx        # Animated line chart (12–20 s)
│   │   │   ├── USRegionScene.tsx     # Real SVG US map + bar chart (19–27 s)
│   │   │   └── OutroScene.tsx        # Title return + seamless loop fade (26–30 s)
│   │   └── components/
│   │       ├── KpiCard.tsx           # Self-contained animated KPI card
│   │       └── ChartLine.tsx         # SVG cubic-Bézier line chart with draw animation
│   └── data/
│       ├── types.ts                  # TypeScript interfaces — defines the shape of data.json
│       └── fetchData.ts              # Data generator / API stub → writes data.json
├── scripts/
│   ├── render.ts                     # Loads data.json, bundles React, renders frames, outputs MP4
│   └── run-all.ts                    # Orchestrator: runs fetchData then render in sequence
├── data.json                         # Auto-generated each run (not committed to git)
├── dist/
│   └── lobby.mp4                     # Rendered video output (not committed to git)
├── remotion.config.ts                # Remotion configuration (image format, overwrite policy)
├── package.json
└── tsconfig.json
```

---

## Individual Commands

```bash
# Generate fresh data.json only (no render)
npm run data

# Render video using the existing data.json (skip data fetch)
npm run render

# Full pipeline: fetch data then render
npm run run-all

# Open Remotion Studio — live browser preview with scrubbing and hot reload
npm run studio
```

---

## Customisation Guide

### Change the company or data
Edit `src/data/fetchData.ts`. The `data` object near the bottom of the file sets all values. Replace the random generators with real API calls to connect live data.

### Change colors or fonts
Edit `src/video/theme.ts`. All scenes and components import from this single file. The Hyundai brand colors are:
- `hyundaiBlue` — `#002C5F` (primary dark blue, used for backgrounds and glows)
- `hyundaiBlueBright` — `#0066CC` (bright blue, used for accents and highlights)
- `accentSilver` — `#C8D0D8` (metallic silver, used for secondary lines)

### Change scene timing
Open `src/video/LobbyVideo.tsx`. The frame constants at the top of the file (`INTRO_START`, `KPI_START`, `TREND_START`, `REGION_START`, `OUTRO_START`) control when each scene begins. The composition total is 900 frames (30 s at 30 fps), set in `Root.tsx`.

### Add a new scene
1. Create `src/video/scenes/MyScene.tsx` — accept `enterProgress` and `exitProgress` props (0–1) and use `useCurrentFrame()` for internal animations.
2. Add a `<Sequence from={START_FRAME} layout="none">` block in `LobbyVideo.tsx`.
3. Update the frame constants and the total `durationInFrames` in `Root.tsx` if needed.

### Reduce render time during development
In `Root.tsx`, change `width: 3840, height: 2160` to `width: 1920, height: 1080`. This cuts render time to roughly 25% of the 4K time with no other changes required.

---

## Render Time

4K rendering is CPU-intensive because Chromium must paint and screenshot 900 frames.

| Machine             | Approx. time |
|---------------------|-------------|
| Apple M1/M2/M3      | 1.5–3 min   |
| Intel i7/i9         | 3–6 min     |
| Cloud VM (4 vCPU)   | 5–10 min    |

Remotion also supports **parallel rendering** across multiple CPU cores and cloud rendering via Lambda if faster turnaround is needed.

---

## Connecting Real Data (Production Path)

To move from POC to production, replace `src/data/fetchData.ts` with a real data fetch. The only contract is that the file writes a valid `data.json` matching the `LobbyData` interface in `src/data/types.ts`.

Example patterns:

```typescript
// REST API
const res = await fetch("https://your-api.internal/kpis");
const data = await res.json();

// Database (e.g. with pg or mysql2)
const rows = await db.query("SELECT * FROM kpi_summary WHERE date = CURRENT_DATE");

// BI tool export (Looker, Tableau, etc.)
// Most BI tools can export to JSON via their API
```

Once `data.json` is written, `npm run render` picks it up automatically — no other changes needed.
