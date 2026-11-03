# Alchemy: notes for future sessions

Goal: recreate Windows Media Player's **Alchemy** visualization in `index.html` (effect 5, served at
https://altrao.github.io/wmp-effects/). It is compared side by side with WMP playing the same music.

The detailed reference notes are kept locally in `measure/ALCHEMY-CODE.md`: every rule, constant and open question.
That file is git-ignored and must never be committed or pushed; neither must anything else in `measure/` that refers to
the original's internals. This public file describes behaviour only.

## Recordings
Screen recordings of WMP 12 on Windows 11 are in `/projects/.orca/drops/tolerance/` (not in git). `measure/analyze.py`
measures them: add `--song` for music, and read the `sheet_*.png` contact sheets as well as the numbers. See
`measure/README.md`.

## How Alchemy works (the model implemented)
- **One persistent image is the whole state.** Nothing clears it. Each tick (about 16/s):
  1. the image is copied through a per-pixel **warp** table and blurred slightly on the way;
  2. one or two **drawing effects** draw on top of it.

  Repeating the blur every tick gives the soft colour washes, and repeating the warp gives the trails, bands, bow-ties
  and swirls.
- **Warps:**
  - shift or stretch from an edge, with an optional sine wobble;
  - swirl around a point;
  - moving tile blocks;
  - reflections (a 4-way mirror, diagonal mirrors, a circle with tiles outside, a circle mirror), combined with one of
    the others.

  Mirroring therefore happens inside the feedback, not at display time. A new warp is picked every 90–500 ticks and eases
  in over 22 steps (reflections snap in). Sometimes two warps are combined.
- **Drawing effects** (two run at a time, each lasting 50–350 ticks). Every line is an oscilloscope trace of the audio
  waveform.
  - **Ocean:** a rotating scope line through the centre, sometimes mirrored, sometimes with a cross line.
  - **Scale:** a star polygon whose size follows bass and whose spin follows treble.
  - **Balls:** two point-mirrored balls with a bright rim. On a beat they flash bigger, and a scope line joins them
    for 5 ticks.
- **Hover**, active about 1/8 of the time, zooms only the displayed image in a slow pulse, with a sudden zoom on a
  bassdrop.
- **Colours** are random RGB, each easing into the next over a random time. There is no palette.
- **Audio:** bass from the lowest spectrum bins, treble from bins ≈500, beat and bassdrop detection.
- **Sizes and speeds are in pixels.** The original renders at window size, so the buffer size changes the look: at
  384×288 the balls were far too big and shaky.

## Implementation (`index.html`, between `// ---------- alchemy` and `// ---------- ambience`)
- `makeAlchemy(W, H)` is a pure simulation with no DOM. It provides:
  - `tick(freq, wave)`;
  - `rect()`, the source rectangle for Hover;
  - `img`, a Uint32Array in ImageData byte order;
  - `paste(src, w, h)`;
  - `next()`, change warp;
  - `opt`, test switches: `blur` (true) and `flag48` (false). Both stay at their defaults; see the local notes.
- **`drawAlchemy(dt)`** runs 16 ticks/s. Each tick it copies `img` into a canvas; every frame it draws `rect()` stretched
  to the window, smoothed. It doesn't use WebGL.
- **Buffer.** It is window-sized (CSS px), capped at 1280 px wide for CPU cost: ≈11 ms/tick at 1280×720, and a
  40–100 ms hitch when the warp table is rebuilt. While the window is being resized, the old buffer is stretched. After
  0.3 s without change, the simulation is rebuilt and the old picture is pasted in.
- **Key `n`** changes the warp.
- **Testing without a browser.** Extract the code from `function makeAlchemy` to `const ALC =`, add
  `module.exports = makeAlchemy`, run it in Node on synthetic audio, and dump PNGs with a small zlib PNG writer.
  Override `Math.random` with a seeded PRNG to compare variants.

## Status (2026-11-03)
- **The user's verdict:** side by side with WMP on the same music, it "doesn't look much different". WMP on a large
  screen is just as soft.
- **Open:**
  - WMP keeps more black background; ours fills up with colour.
  - The loudness scale isn't calibrated: browser bins are dB-scaled.
  - One mono analyser stands in for two channels.
  - Balls are up to 1.5× too big above 1280 px wide because of the cap.
  - The warp-change hitch could be removed by building the table over several ticks.

## `index-1.html` (the user's draft, untracked)
Separate effects (Ocean, Bouncing Balls, Hover, Stretch, Flow) with sliders; its ideas fed an earlier version. Its bugs:
- `progs.seed` is created before the WebGL check and is then overwritten;
- the ball speed doubles on every bass drop;
- keys can't reach the 10th effect.

## Workflow
- Every push to `main` republishes GitHub Pages. Push only when the user asks.
- Keep this file and `measure/ALCHEMY-CODE.md` updated. Implementation details go here; anything about the original's
  internals goes only in the local file.
