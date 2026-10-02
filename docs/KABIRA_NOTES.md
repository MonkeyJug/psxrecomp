# Branch kabira/render-pass-ws: what it adds

Branch of the MonkeyJug fork used by Kabira (PS1 Winning Eleven 2002 Deluxe).
Base: upstream `c604cea`. Six commits, all opt-in or bug fixes; with none of
the new keys set, a game behaves as on the base. Written 2 October 2026 when
Kabira's 16:9 was signed off (Kabira tag `v0.2.0-widescreen`).

| Commit | What | Upstream value |
|---|---|---|
| `b58b928b` | **Replace render passes**: `psx_mod_render_pass_open()` plus a pass with `alpha_q16 == 0` whose image becomes the frame's own, and `[widescreen.cull] pass_only` (cull margin live only inside a pass). Lets a plugin redraw a frame with a widened cull while the simulation never sees it. | High: draw-only widening for any game whose logic reads its cull |
| `1fe0acf1` | Presenter binds a pass generation at the guest flip (GP1(05h)) with three generations, and a VRAM self-copy with mask-set off no longer counts as a redraw. Fixes games that flip right after DrawSync and start the next frame in the same VBlank (about half of WE2002's passes were never shown). TCP `render_pass_flip_log`. | High: general presenter bug |
| `29214955` | TCP `render_pass_replace_keep`: keep the game's image and the pass image of the same frame for `render_pass_dump`. | Medium: verification tool |
| `1ea1a06b` | `[widescreen] sx_headroom` (cherry-picked from `we2002/sx-headroom`): GTE SX2 clamp and GPU vertex X widened while a margin is live. | Medium: opt-in, title-specific safety note in the docs |
| `965b899b` | `sx_headroom_unbounded_in_pass`: inside a pass, no SX2 clamp, 16-bit vertex X, no 1023 px width rule. Not used by Kabira in the end. | Low |
| `82222dc8` | Full-screen fade/flash detection (`ws_expand_fullscreen_rect`) in absolute coordinates against the draw area. The old test compared pre-offset coordinates with the screen origin, so games with a centred drawing offset or stacked buffers never had fades widened. | High: general bug |

## Lessons for anyone doing the same

- A render callback that only **builds** ordering tables draws nothing inside
  a pass: hook the DrawOTag call instead, rebuild the tables there, and send
  them by direct GPU DMA. libgpu's DrawOTag/DrawSync cannot be used inside a
  pass (its queue drains on a DMA interrupt a pass never delivers: DrawSync
  spins to its timeout, DrawOTag only queues).
- Stores to the mod memory aperture (`psx_mod_alloc_*`) are **dropped** inside
  a pass. Packets a pass adds must live in main RAM (WE2002 uses the other
  buffer's packet pool, which the pass restores).
- Verify with pixels (same-frame dumps, marked screenshots), not with the
  game's own visibility flags: Kabira's first "no pop-in" result was wrong for
  exactly that reason.
