# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page, dependency-free product showcase for "Aurelia", an antique music box (Maison Rouvel). Everything lives in [index.html](index.html) — inline CSS and vanilla JS, no build step, no framework, no package manager. The signature is an Apple-style, scroll-driven reveal: scrolling scrubs through 192 video frames of the box opening.

## Running & tooling

There is no build/lint/test. Two rules matter:

- **Must be served over HTTP, not `file://`.** The `<canvas>` draws cross-origin-tainted frames under `file://` and silently fails. Use:
  ```
  python3 -m http.server 8765   # then open http://localhost:8765
  ```
- **Verify visually in a real browser.** Changes to the scroll engine, blend mode, or layout are easy to break in ways that don't error. Drive it with the Playwright MCP tools: wait for `#loader.done`, then `scrollTo` and screenshot. Because `html { scroll-behavior: smooth }`, programmatic `scrollTo` animates — set `document.documentElement.style.scrollBehavior='auto'` before jumping, and wait for it to settle.

### Regenerating assets (only if the source video changes)

The committed assets are derived from `wooden_music_video.mp4` (8s, 24fps, 1280×720). Requires `ffmpeg` and `cwebp` (`brew install webp`):

```
# frames (JPG -> WebP q80; current set is 192 files, frame_0001..0192)
ffmpeg -i wooden_music_video.mp4 -q:v 2 frames/frame_%04d.jpg
for f in frames/frame_*.jpg; do cwebp -q 80 "$f" -o "frames/${f%.jpg}.webp"; done

# audio (melody for the "試聽樂曲" button)
ffmpeg -i wooden_music_video.mp4 -vn -c:a copy audio/aurelia.m4a
```

If the frame **count** changes, update `TOTAL = 192` in the JS — it is hardcoded and drives the whole scroll→frame mapping.

## Architecture

The page is two stacked regions plus fixed chrome:

1. **The scroll showcase** — `#scroll-wrap` is a tall (`760vh`) scroll track containing a `position: sticky` `#stage`. A `<canvas id="frame">` fills the stage and is the product. Scroll progress (0→1 across the track) maps to frame index `0→191`. Narrative text `.panel`s are absolutely positioned over the canvas and fade in/out at progress thresholds.
2. **Content sections** — normal-flow `<section class="section">` blocks (`#about`, `#product`, `#pricing`, `#contact`) and `<footer>`, rendered after the sticky track releases.

Fixed chrome: frosted-glass `<nav>` (anchors into the content sections) and `#winder` (right-side scroll-progress rail, fades out once the showcase completes).

### Two independent IIFEs in the `<script>`

- **Showcase engine** (first IIFE): preloads all frames into an `images[]` array, then a scroll handler (rAF-throttled) computes progress and calls `render(progress)`, which (a) draws the current frame to the canvas, (b) sets each panel's opacity via the `panels` array of `[element, startProgress, endProgress]` ranges, and (c) updates the winder. `draw()` crops ~6% off the bottom of each source frame to hide the baked-in "Veo" watermark.
- **Sections & interactions** (second IIFE): IntersectionObserver scroll-reveal (`.reveal` → `.in`), contact-form handling (validate → AJAX POST → success swap; see Netlify Forms below), scroll-spy (highlights the nav link of the in-view section), and the background-music play/pause toggle.

### Two things that make the look work

- **`mix-blend-mode: screen` on the canvas.** The frames have a pure-black background; `screen` blend turns black transparent, so the box dissolves into the dark page and the breathing radial `#glow` shows through. Any new full-bleed imagery meant to "float" should follow this (black bg + screen), not a framed `<img>`.
- **Design tokens in `:root`.** Palette (`--obsidian`, `--rosewood`, `--ormolu`, `--porcelain`, `--dusk-blue`) and the three type roles (Cormorant Garamond display, Inter body, IBM Plex Mono labels) are defined once. `--dusk-blue` is the single cold accent and is used sparingly (edition number, featured price card) — keep it rare. Derive new colors/fonts from these tokens.

### Accessibility floor to preserve

`prefers-reduced-motion` disables ambient animations and shows `.reveal` content immediately; keyboard focus is visible; layout is responsive down to mobile (grids collapse to one column at 980px / 520px). Don't regress these.

## Deployment & the contact form (Netlify Forms)

The site is deployed on Netlify (auto-deploys on push to `main`): https://claudecode-web-design.netlify.app/

The `#contactForm` submits to **Netlify Forms**. The wiring has two halves that must stay in sync:

- **HTML markup Netlify detects at deploy:** `<form name="contact" method="POST" data-netlify="true" netlify-honeypot="bot-field">`, a hidden `<input name="form-name" value="contact">`, and an off-screen `bot-field` honeypot. Every field that should be captured needs a `name` attribute. If you rename the form, change `form-name` and the `name` to match.
- **JS submit handler:** instead of a native submit it does `fetch("/", { method:"POST", body: new URLSearchParams(new FormData(form)) })`, then reveals `#formSuccess` on a 200 and re-enables the button + alerts on failure. Keep posting to `/` urlencoded — that's the endpoint Netlify intercepts.

Gotchas (both bit us during setup):
- **Form detection must be enabled** in the Netlify dashboard (Site configuration → Forms), and a form is only registered on a **deploy made after** detection is on. Symptom when it isn't: POST to `/` returns **404** (not 200), and no submissions appear.
- The local `python3 -m http.server` returns **501** for POST, so the success path can't be tested locally — only the request firing. Validate the full flow against the deployed Netlify URL.

## Skills

The [frontend-design](.claude/skills/frontend-design/SKILL.md) skill is installed at project level — invoke it before any visual/UI design work on this page.
