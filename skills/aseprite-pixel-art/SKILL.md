---
name: aseprite-pixel-art
description: Draw, shade, animate and export pixel art in Aseprite through the `aseprite` MCP server. Use when asked to make a sprite, icon, tileset, character or animation, or whenever you are about to call aseprite MCP tools (create_canvas, draw_*, view_sprite, export_*).
---

# Aseprite pixel art

You are drawing in a pixel editor, not painting a picture. Every pixel is a decision someone
can see. The `aseprite` MCP gives you 123 tools that drive Aseprite headlessly through Lua;
this skill is the craft discipline around them.

## The loop you always run

1. **Decide before drawing**: canvas size, purpose (single sprite / sheet / animated), and a
   palette. A 32×32 character gets 4–6 colours per material, not 30.
2. **Create, then read back the layer name.** Every drawing tool requires `layer_name` and a
   wrong guess is the most common failure:
   `create_canvas(filename=…, width=32, height=32)` → `get_sprite_info(filename=…)` →
   `layers[0].name` is usually `"Layer 1"`.
3. **Build the ramp first**: `generate_color_ramp(base_color="#8fa4c8", steps=5,
   hue_shift_degrees=25, lightness_range=0.5)`. Dark→light, shadows cooler, highlights warmer.
4. **Block → shade → detail → outline**: silhouette with `fill_area_at` /
   `draw_rectangle_at(fill=True)`, then shade with ramp steps, then 1px detail with
   `draw_pixels` / `draw_rows`, then a selective outline last.
5. **Look at it every 3–5 calls.** `view_sprite(filename=…, scale=0)` renders frames with a
   grid and rulers. Never batch 20 edits and check once.
6. **Measure, don't admire**: `get_bounds`, `lint_sprite`, `get_color_stats`,
   `compare_frames`.
7. **Export** at the end, in the shape the user actually needs.

## Tool signatures (get these right)

| Tool | Required args | Notes |
|---|---|---|
| `create_canvas` | `filename`, `width`, `height` | `color_mode` optional (`rgb` / `indexed` / `grayscale`) |
| `get_sprite_info` | `filename` | Returns JSON: size, frames, durations, layers |
| `fill_area_at` | `filename`, `layer_name`, `frame_index`, `x`, `y` | Flood fill from a point. For a full-canvas background a `draw_rectangle_at(fill=True)` is cheaper |
| `draw_rectangle_at` | `filename`, `layer_name`, `frame_index`, `x`, `y`, `width`, `height` | **`width`/`height`**, not `w`/`h`. `fill=True` for solid, `color="#rrggbb"` |
| `draw_pixels` | `filename`, `pixels` | `pixels` is a **list of dicts**: `[{"x":14,"y":9,"color":"#20222e"}]` — not triples, not `[x,y,color]` |
| `draw_rows` | `filename`, `layer_name`, `frame_index`, `x`, `y`, `rows`, `legend` | ASCII-art form, one char per pixel; `legend={"K":"#1a1c2c","S":"#f4b41b"}`; optional `outline`. Best way to place a face, hand, or any < 12px detail |
| `read_rows` | `filename`, `x`, `y`, `width`, `height` | Reads back in `draw_rows` format — edit a few characters and redraw. Use this instead of `get_pixels_rect` for anything you intend to change |
| `draw_iso_box` | `filename`, `layer_name`, `frame_index`, `x`, `y`, `z`, `w`, `d`, `h`, `top`, `left`, `right` | Clean 2:1 isometric furniture/crates |
| `outline_cel` | `filename`, `layer_name`, `frame_index` | 1px outline, `include_diagonals`, `region="x,y,w,h"` to outline one object on a shared layer |
| `view_sprite` | `filename` | `scale=0` auto-fits, `region="x,y,w,h"` to zoom, `frames="1-4"` |
| `export_sprite` | `filename`, `output_filename` | PNG/GIF/JPG by extension |
| `export_spritesheet` | `filename`, `output_filename` | `sheet_type`, `data_filename` for JSON metadata |
| `export_animation` | `filename`, `output_filename` | GIF; `scale`, `background="#e9eef2"` so a dark outline reads in any viewer |

Everything also has a `*_at` variant that takes `layer_name` + `frame_index`; the plain variant
acts on the active layer/frame. In batch mode there is no "active" thing worth trusting — pass
`layer_name` and `frame_index` explicitly.

## Palette discipline

- **3–5 colours per material.** Cloth is a 4-step ramp, metal is 3 + a highlight, skin is 3.
- **Shift hue, not just value.** Shadows go toward blue/violet, highlights toward yellow —
  `generate_color_ramp` does this; plain darkening produces mud.
- **Never `#000000` or `#FFFFFF`** for fills. Use `#1a1c2c`-ish and `#f4f4f4`-ish. Pure black
  flattens depth and pure white blows out; reserve them for a single outline or a sparkle.
- Start from a known palette when the target is retro: `list_palette_presets` →
  `apply_palette_preset` (`pico8`, `gameboy`, `dawnbringer16`…) → `quantize_to_palette` to snap
  what you already drew.
- If the user gave a reference image: `import_image_as_layer`, then `extract_palette` /
  `get_color_stats` and redraw with that ramp instead of sampling randomly.
- Before saying "done": `get_color_stats(filename, top=16)`. If there are 30 near-duplicate
  browns, you drifted — collapse them.

## Shading, dithering, outlines

- **Shade by geometry, not by mood.** One light direction per sprite. Decide it once and keep
  every highlight on the same side for the whole sheet.
- **Dither between adjacent ramp steps only**, and only on large areas
  (`apply_dither_gradient`, `apply_dither_pattern`). Dithering a face or a 4px-wide limb
  reads as noise at 1x.
- **Selective outlining**: outline the silhouette against the background, skip the bottom
  edge that touches ground, and don't outline internal edges that are already contrasted
  (`outline_cel` with `region=`, or draw it manually with `draw_rows`).
- **Rim light** beats a second outline: 1px of the lightest ramp step along the light-facing
  edge makes the sprite pop without thickening it.
- Minimum detail size is 1px, but leave 1px of breathing room: a 32×32 sprite should not have
  a 20px-wide solid mass with no internal value change.

## Animation

| Action | Frames | Per-frame ms | Notes |
|---|---|---|---|
| idle / breathing | 4–6 | 120–200 | move 1–2px, chest and head lag by one frame |
| walk | 6–8 | 80–120 | contact → down → pass → up, twice |
| run | 4–6 | 60–80 | more lean, longer stride, arms bent |
| attack | 3–5 | 50–80 | anticipation, strike, hold 2× on the impact frame |
| looping FX | 4–8 | 80–120 | seamless: last frame must lead into the first |

Rules that keep cycles from looking broken:

- **Copy, don't redraw.** `copy_frame(source_frame=1)` then edit the moved limb, or
  `tween_cel_positions_eased` / `oscillate_cel_positions` for a bob. Frame 2 differing from
  frame 1 in every pixel looks like a different character.
- **One thing moves per frame.** If legs and arms and hair all change at once, nothing reads.
- **Ground the feet**: `get_bounds(frames="1-8")` — the bottom `y` must be identical on every
  frame, or the character bounces.
- **Name the cycle**: `set_tag(name="idle", from_frame=1, to_frame=6, direction="pingpong")`.
  Engines and `export_tag` need it.
- **Verify, don't guess.** `compare_frames(frame_a=1, frame_b=6)` should be ~0% changed for a
  pingpong loop's seam; `render_onion_skin` shows whether the in-betweens line up;
  `audit_animation` / `animation_sanitize` catch cels leaking outside their frame range.
- Timing first, frames second: a 4-frame idle at 160ms reads better than 8 frames at 40ms.

## Anti-patterns

- 30 tool calls, then one `view_sprite`. Draw blind only if you enjoy redoing it.
- Guessing `layer_name`. Read it from `get_sprite_info`.
- Passing `w`/`h` (rectangles want `width`/`height`) or `[x,y,color]` tuples to `draw_pixels`
  (it wants dicts).
- Dithering to fake a gradient in a 16×16 icon; pick the next ramp step instead.
- Outlining everything with the same colour: an outline is a silhouette tool, not a border.
- Exporting a GIF without `background`: transparent GIF edges shimmy in every viewer.
- Forgetting `app.refresh()`-style thinking — this is headless, nothing is "on screen" until
  you export or `view_sprite`. The user's open Aseprite window is **not** what these tools
  touch; they edit files on disk.

## Worked example: 32×32 knight

```python
create_canvas(filename="knight.aseprite", width=32, height=32)
# → get_sprite_info(...) says layer "Layer 1"
fill_area_at(filename=…, layer_name="Layer 1", frame_index=1, x=0, y=0, color="#20222e")
# silhouette first: body, head, legs, arms, sword — flat blocks, no shading yet
draw_rectangle_at(…, x=12, y=15, width=9, height=11, color="#8fa4c8", fill=True)   # torso
draw_rectangle_at(…, x=13, y=6,  width=7, height=7,  color="#f2c9a0", fill=True)   # face
# then ramp steps on top of the block colours, light from upper-left
# then 1px detail with draw_pixels: eyes, belt buckle, pommel
view_sprite(filename="knight.aseprite", scale=0)   # look at it before continuing
outline_cel(filename="knight.aseprite", layer_name="Layer 1", frame_index=1, color="#101018")
export_sprite(filename="knight.aseprite", output_filename="knight.png")
```

For an idle: `add_frames(3)` → `copy_frame(1, target_frame=2..4)` → move the torso 1px down on
frame 3 and back on frame 4 → `set_tag("idle", 1, 4)` → `set_frame_duration_all(150)` →
`export_animation(…, background="#e9eef2")`.

## When something fails

- Validation error naming `layer_name` / `width` / `pixels`: wrong argument name or shape — see
  the table above; do not retry with the same payload.
- "Aseprite run took longer than 60 seconds": the operation is too big for one batch. Split it
  (per layer, per frame) or raise `ASEPRITE_TIMEOUT_SECONDS` in the server's environment.
- Colours look wrong after export to GIF: GIF allows 256 colours per frame — quantise first
  (`set_color_mode(mode="indexed")` + `quantize_to_palette`) or export a sprite sheet instead.
- Sprite looks fine at 8× and mush at 1×: you shaded with values that are too close. Rebuild
  the ramp with `lightness_range=0.55` and re-apply the darkest/lightest steps only.
