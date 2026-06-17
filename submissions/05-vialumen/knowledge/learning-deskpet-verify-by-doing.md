# desk-pet character pack — read the real arch, verify by doing

**Room:** Oracle School workshop (ESP32 wasm) · **Teacher:** P'Nat · **Date:** 2026-06-17

## The mistake (caught by teacher)
I assumed the "desk-pet" was ESPHome / a wasm3 serial printer and built a
standalone LVGL firmware. Wrong on both counts. The teacher pushed back 5+ times
("no esphome", "re-read my code", "find these in the code first").

## What it actually is
**jc3248-pet** firmware. Characters are **GIF packs in LittleFS**:
`/characters/<pack>/*.gif`. The *same GIFs* are decoded two ways:
- **device:** native AnimatedGIF (bitbank2) → 3× upscale → LovyanGFX → QSPI LCD
- **browser:** that decoder compiled to WASM (`gif-wasm` / `gifdec.wasm`) → canvas

= "many bodies, one soul." One asset, two runtimes.

**Pack format:** `96×100 GIF89a`, **7 states**
(`sleep idle busy attention celebrate dizzy heart`) + a `manifest.json`
(`name`, `colors`, `states`). Register the pack in the preview page's `PACKS`
object so `?pack=<name>` works.

## Patterns that worked
1. **Read the real code before building.** The whole detour came from guessing
   the architecture instead of grepping for `characters/` + the manifest format.
2. **Learn from peers.** A classmate (Weizen) had already decoded the format and
   posted it; chaiklang's submission was a correct template to copy. Faster +
   more correct than re-deriving. (cf. learn-from-peers)
3. **Verify by doing, not by claiming.** Before shipping I ran the workshop's own
   `gifdec.wasm` under Node against all 7 GIFs → confirmed 96×100, frames intact.
   That's the exact decoder the device+browser use, so "it'll probably decode"
   became "it decodes." (cf. ehipassiko / verify-before-answer)
4. **Make art in code = clean provenance.** Generated all 7 GIFs with Pillow
   (a small parametric star sprite) → original, MIT, zero third-party assets.
   Reproducible via one script committed alongside.
5. **Be honest about the gap.** Device *flash* of a new pack needs the shared
   `storage.bin` (one LittleFS for all packs) rebuilt. Said so in the PR instead
   of pretending the flash path was done.

## Root cause: a pack id must line up in ALL places (else preview falls back)
The flasher went **data-driven** mid-task: a pack = `docs/packs/<id>.json`
(`kind: pet|firmware`). But the embedded preview still reads a hardcoded `PACKS`
map in `preview/index.html`. Symptom the teacher caught: my pet's right-side
preview showed **bufo (the frog)** — because `?pack=vialumen-pet` had no entry in
`PACKS`, so `selectPack` fell back to `"bufo"`. The id has to match in every layer:
- `docs/packs/<id>.json` (picker + manifest)
- `PACKS["<id>"]` in `preview/index.html` (the embedded live preview + state buttons)
- `docs/preview/gifs/<id>/*.gif` (the frames the preview fetches)
- `/characters/<id>/` inside the storage image (what the device boots)
Mismatch in any one = silent fallback, not an error. When a project's structure
shifts under you, re-grep how the thing you're touching is actually wired before
assuming your old mental model still holds.

## Flash a pack to device WITHOUT building ESP-IDF (key unblock, from a peer)
The pet app **auto-discovers** the pack in LittleFS (first dir wins). So you don't
rebuild the firmware — reuse the shared app bin and ship only your own LittleFS:
```python
from littlefs import LittleFS          # pip install littlefs-python
fs = LittleFS(block_size=4096, block_count=0x300000//4096)   # 3 MB @ 0x290000
fs.makedirs("/characters/vialumen", exist_ok=True)
# write each .gif + manifest.json, then:
open("vialumen-storage.bin","wb").write(bytes(fs.context.buffer))
```
esp-web-tools manifest reuses shared `bootloader.bin`(0xE9, offset 0) +
`partition-table.bin`(0xAA, 0x8000) + `jc3248_pet_idf-<any>.bin`(app, 0x10000) +
your `storage.bin`(0x290000). **CI gotcha:** `flasher-check` requires the
smallest-offset part to start `0xE9` (a real boot image, not a 0xff/erased brick).
Verify by mounting the image back (`fs.listdir`) before shipping.

## Reusable: generating a decoder-safe GIF with Pillow
- One **shared adaptive palette per state** (`quantize(colors=64)` over a vertical
  strip of all frames), `dither=NONE` → small files, clean loop, no per-frame
  palette drift.
- `save(..., save_all=True, duration=delays, loop=0, disposal=2)`.
- Supersample 3× then downscale (LANCZOS) for smooth edges at 96×100.
