# Companion Gallery App — Spectra 6 Color

**Status: messy working spec, 2026-07-24. Not finalized.**

A companion project to gallery-app that targets the same Spectra 6 hardware
(EL133UF1 panel, ESP32-S3) but renders in **6-color** instead of 16-gray.
Sits alongside the existing monochrome gallery-app as a separate app with a
separate entry point and rendering backend, but sharing the intelligence and
content layer.

Target hardware:
- **Hokku Designs / Huessen 13.3" WiFi E-Paper Art Photo Frame** (existing
  target — $280 Wayfair, confirmed hardware in this repo)
- **Seeed Studio EE02 kit** (XIAO ESP32-S3 Plus + EL133UF1, ~$163.90) —
  DIY/bare version of the same panel. GPIO map differs from the Hokku frame
  (different board), but display protocol is identical: same UC8179C controller,
  same init sequence, same SPI config, same nibble map.

---

## What's reused from gallery-app

- React/Vite PWA frontend — separate entry point or build flag, not a fork
- FastAPI server structure and most API endpoints
- Database schema: images, series, widgets, devices, hang_history, taste_events
- Content sources: museum adapters, magazine scraper (3.9), artist watchlist
- VLM tagging (2.3), embeddings (2.2), recognition substrate (2.1)
- Scheduling/occasions engine (3.2), signals service (3.1)
- Studio generation backend (Draw Things / Flux, 3.3)
- Import pipeline: `import_image_bytes` — EXIF, place, auto-placard
- Widget framework: Spotify, Studio, series, ensemble/span machinery
- Scraper quality gate and source management

The intelligence layer (tagging, embeddings, similarity, occasions) is
panel-agnostic. A photo tagged "coastal / fog / morning" is equally valid
for 16-gray or 6-color display.

---

## What's different

### Backend rendering pipeline

The core change: instead of `tone → 16-gray dither → quantize`, the color
pipeline does:

```
RGB image
  → color-aware treatment (auto-level on L channel in Lab, not grayscale)
  → Lab-space Floyd-Steinberg dither against 6-color palette
  → nibble encode (4bpp, high nibble first, same byte packing as 16-gray)
  → panel split (960K → two 480K halves, left/right, same split logic)
```

The panel split and byte packing are **identical** to the 16-gray pipeline —
only the quantization step changes.

### The 6-color palette (confirmed in hardware_facts.md)

Real-world measured RGB values from the physical panel:

| Color  | Nibble | RGB               | Lab (approx)      |
|--------|--------|-------------------|-------------------|
| Black  | 0x0    | (25, 30, 33)      | L=11, a=−1, b=−2  |
| White  | 0x1    | (232, 232, 232)   | L=92, a=0,  b=0   |
| Yellow | 0x2    | (239, 222, 68)    | L=88, a=−5, b=72  |
| Red    | 0x3    | (178, 19, 24)     | L=33, a=60, b=43  |
| Blue   | 0x5    | (33, 87, 186)     | L=37, a=15, b=−55 |
| Green  | 0x6    | (18, 95, 32)      | L=35, a=−38, b=30 |

Nibble 0x4 → White/light (not orange despite some third-party drivers).
Nibbles 0x7–0xF exist but produce non-standard intermediate colors — not
reliable, don't use.

**Dither in Lab, not RGB.** Euclidean distance in Lab is perceptually uniform;
RGB nearest-neighbor produces worse palette matches, especially for yellows
and greens near the gamut edge.

### Color dithering approach

Floyd-Steinberg error diffusion, same serpentine (boustrophedon) scan as the
16-gray dither, but operating on 3-channel Lab pixels against a 6-point palette:

```python
def _nearest_color(lab_pixel, palette_lab):
    # Euclidean distance in Lab
    diffs = palette_lab - lab_pixel  # (6, 3)
    return np.argmin(np.sum(diffs**2, axis=1))
```

Error propagates in Lab, then clipped to valid Lab range before the next pixel.
The numba JIT path from the 16-gray pipeline can be adapted; the inner loop
structure is the same, the kernel changes from scalar to 3-channel.

### Fitness scoring

The 16-gray MS-SSIM scorer (`T·D·(1−G)`) doesn't map to 6-color. Need a
color-specific metric — rough ideas, not finalized:

- **Gamut coverage**: fraction of pixels that find their nearest palette color
  within a Lab distance threshold. High coverage = image renders faithfully.
  Low coverage = large chunks of the image will shift significantly.
- **Palette utilization**: entropy of the output palette histogram. An image
  that maps entirely to black+white has a very different character than one
  using all 6 colors.
- **Rendered SSIM**: simulate the palette render → compare rendered sRGB vs
  original sRGB via SSIM. This is the most direct metric but most expensive.

The existing A1 scorer infrastructure (rescore endpoint, `dither_score` column,
optimizer) should be reused — just swap the scoring function.

### Content fitness for color vs grayscale

The VLM quality gate (3.9 scraper) probably needs different thresholds:
- Bold graphic design, posters, silkscreens — likely better in 6-color than
  16-gray (clean palette, strong contrast = good palette coverage).
- Subtle tonal landscape photography — likely worse (relies on smooth tonal
  gradients that 6 colors can't reproduce; dithers to noise).
- The `eink_fitness` score from Qwen3-VL was trained on the 16-gray mental
  model. May need a separate color-fitness prompt or a post-hoc filter.

This is an open question — worth testing empirically once rendering works.

### Frontend differences

- Separate entry point (`web/src/main-color.tsx` or a `VITE_PANEL_TYPE=color`
  build flag — TBD which is cleaner)
- Grid tiles: color preview thumb (simulated palette render) instead of
  16-gray preview
- Wall mock (`/mock`): 6 colored cells instead of gray gradient
- `dither_score` column/UI: replaced by `color_fitness` or renamed
- Otherwise: same picker, same library, same widgets, same UX structure

---

## Device / firmware notes

### Hokku frame
The existing `hokku_epaper` firmware already outputs 6-color nibble-encoded
data. The server-side gallery-app currently sends 16-gray data to the same
frame (monochrome mode). The companion app would send color data to the same
device — no firmware change needed, just the server rendering.

Actually: the current firmware *does* support 6-color already — the panel
displays whatever nibbles the server sends. The gallery-app is just sending
16-gray nibbles. The companion app sends 6-color nibbles. Same firmware.

### Seeed EE02
Ships with Seeed's own firmware (SenseCraft HMI). To use with this stack,
either:
1. Flash the hokku_epaper firmware (GPIO map will differ — need EE02 schematic
   to map CTRL1/CTRL2/BUSY/RST/MOSI/SCLK to EE02 pins before building), or
2. Write a thin HTTP adapter that translates the gallery-app render endpoint
   into the EE02's native API.

Option 1 is cleaner long-term. Option 2 is faster to test.

---

## Open questions

- [ ] Shared DB vs separate? Leaning toward same DB with `panel_type` on
      `devices` table (`grayscale_16` | `spectra6_color`). Pipeline gates on
      this at render time. Content, tags, embeddings all shared.
- [ ] Single server binary with a `panel_type` render branch, or two separate
      FastAPI apps? Single binary + gate is simpler; two binaries is more
      isolated. Probably single binary first.
- [ ] EE02 GPIO map — need the schematic to port the firmware. The SPI config
      (8 MHz, mode 0, 3-wire half-duplex) and init sequence should be identical
      since both use UC8179C.
- [ ] Color fitness scorer: rendered-SSIM is most honest but expensive. Run
      it offline at import time (like `POST /images/optimize`) rather than
      in the live render path.
- [ ] The 16-gray tone curve (SCREEN_BLACK/SCREEN_WHITE compensation) has no
      direct equivalent for 6-color. May need per-panel white-point calibration
      (the measured White=(232,232,232) is the calibration anchor).
- [ ] Image selection: should the same imported image be renderable in both
      modes? Probably yes — store both a `dither_score` (16-gray) and a
      `color_fitness` score on each image, show both in the UI.

---

## Rough build order

1. **`pipeline/color_dither.py`** — Lab-space Floyd-Steinberg against 6-color
   palette. Port numba JIT from `pipeline/dither.py`, 3-channel inner loop.
   Golden tests: palette maps correctly (black → 0x0, white → 0x1, etc.);
   serpentine conserves mean Lab value.
2. **`pipeline/color_fitness.py`** — gamut coverage + palette utilization
   scorer. Golden tests against known-good and known-bad images.
3. **`pipeline/renditions.py` gate** — `panel_type` branch: if `spectra6_color`,
   run color_dither instead of 16-gray path. Same 4bpp output, same panel split.
4. **`devices` table** — add `panel_type` column, default `grayscale_16`.
   `GET /api/displays/{id}/render` reads it and picks the pipeline.
5. **Color preview thumb** — `GET /api/images/{id}/color-preview` endpoint:
   same as `preview_thumb` but runs the color pipeline. Used by the frontend
   grid tiles for the color app.
6. **Frontend entry point** — `VITE_PANEL_TYPE=color` flag; swap preview thumb
   source, swap mock wall colors, rename fitness label.
7. **EE02 firmware port** — map GPIO from Hokku frame to EE02 board once
   schematic is available.
