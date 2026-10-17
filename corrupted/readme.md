# Message-Length Framing Rule — Decoder Spec

**Target file:** `1792033632892.png` (698,290 bytes, SHA256 `9c206f17cd654939c18ae62754723842373b29c32a2f908205588b461fb5d1a3`)
**Status:** COMPLETE. Length rule proven from decompiled APK source; full plaintext reproduced byte-exact (see §6). The sole open gap from the prior revision — the "binary-interleaved" payload region — was an artifact of reading pixels in the wrong order, corrected 2026-10-17.

---

## 1. The Rule

The message length is determined by a **3-byte terminator**:

- **Start marker:** `@!#` (bytes `0x40 0x21 0x23`)
- **End marker:** `#!@` (bytes `0x23 0x21 0x40`) — the start marker reversed

The decoder accumulates extracted bytes and stops the moment the accumulated string ends with `#!@`. Everything between the markers is the payload.

**Source:** `a2/AsyncTaskC0285c.java`, `doInBackground()` — the decode loop checks `str4.endsWith("#!@")` after every output byte and logs `"Decoding ended"`. The encode side constructs the embedded string as `"@!#" + Base64 + "#!@"`.

In the target file, the terminator sits at **byte 1539** of the LSB stream (the only occurrence in 421,875 bytes). The payload region is bytes 3–1538 (1,536 bytes) — pure, clean Base64 with zero non-alphabet bytes.

---

## 2. Complete Extraction Procedure

For the public extraction page, implement exactly this:

1. **Load pixels.** Read the PNG as RGB. (The app uses Android `Bitmap.getPixels()` → ARGB ints, then takes R, G, B — alpha is ignored.)
2. **Walk pixels in 512×512 TILED order — NOT full-image row-major.** This is the step every reverse-engineering attempt got wrong. The app tiles the bitmap via `m3/v.Q()` (`Bitmap.createBitmap(bitmap, x, y, w, h)` with 512×512 tiles, last row/column possibly smaller) and processes tiles in row-major tile order: all pixels of tile (0,0), then tile (1,0), then tile (0,1), and so on. Within each tile, pixels go row-major. For a 750×750 image the tiles are (0,0,512,512), (512,0,238,512), (0,512,512,238), (512,512,238,238). Reading the whole image row-major instead diverges from the app at pixel 512 (output byte 384) — everything after that decodes as garbage, which is exactly the "binary-interleaved region" misdiagnosed in the prior revision.
3. **Build the byte array.** For each pixel in tiled order, append three bytes: `R = (pixel >> 16) & 255`, `G = (pixel >> 8) & 255`, `B = pixel & 255`.
4. **Extract 2 LSBs.** Walk the byte array sequentially. From each byte take its 2 least-significant bits. Pack four consecutive 2-bit groups into one output byte, most-significant group first:
   `out = ((b0 & 3) << 6) | ((b1 & 3) << 4) | ((b2 & 3) << 2) | (b3 & 3)`
   (Decompiled: shifts `{6,4,2,0}`, masks `{0xC0,0x30,0x0C,0x03}` — confirmed from `La2/c.doInBackground`.)
5. **Accumulate and watch for markers.** Append each output byte to a buffer (interpreted as ISO-8859-1 / raw bytes — the app builds the string with `new String(bytes, "ISO-8859-1")`, so the byte↔char round-trip is lossless). The first three bytes must be `@!#`; if the 3-byte accumulation isn't `@!#`, discard and keep scanning (the app resets). Stop appending when the buffer ends with `#!@` (the app logs `"Decoding ended"`).
6. **Strip markers.** Remove the leading `@!#` and trailing `#!@`. What remains is the Base64 payload string (1,536 chars for the target file, newlines every 76 chars, `=` padding at the end).
7. **Base64-decode**, then **AES-128-ECB decrypt** with the 16-byte key (`################` for empty password — password padded with `#` to 16 chars, truncated to 15 if longer). Notes from the decompiled `LL2/a.l`: the app calls `message.getBytes()` (UTF-8 default charset) before `android.util.Base64.decode(bytes, 0)` — harmless for pure-ASCII Base64 — and `Cipher.getInstance("AES")` (AES/ECB/PKCS5Padding).
8. **Remove PKCS7 padding.** Result for the target file: 1,120 bytes of plaintext (see §6).

---

## 3. Why Three AIs Truncated at Different Points

Muse, Claude, and ChatGPT all recovered the start correctly (`@!#`, Base64, AES-128-ECB, the `#` key) but each stopped at a different place. Two compounding errors:

1. **Wrong pixel order.** All three read the image row-major. The app reads 512×512 tiles. Row-major diverges at pixel 512 (output byte 384), so everything past the first ~304 Base64 chars decoded as garbage — misread as a "binary-interleaved" payload region.
2. **Guessed end condition.** Each model applied a different "where does the Base64 stop?" heuristic instead of the `#!@` terminator the app actually scans for.

The app never checks Base64 validity to find the end. It scans for the literal `#!@` terminator. Any decoder that guesses the length from content heuristics instead of the terminator will truncate at an arbitrary point — which is exactly what happened.

**Lesson for the extraction page:** implement the tiled pixel order AND the terminator scan. Do not stop at the first non-Base64 byte.

---

## 4. Measurements (facts, no interpretation)

| Item | Value |
|---|---|
| LSB stream size | 421,875 bytes (750×750 px × 6 bits ÷ 8) |
| Pixel walk order | 512×512 tiles, row-major tile order (from `m3/v.Q()`) |
| Start marker `@!#` | byte 0 |
| End marker `#!@` | byte 1539 (sole occurrence) |
| Payload region | bytes 3–1538 (1,536 bytes), 100% clean Base64 alphabet |
| Base64 decode of region | 1,136 bytes ciphertext |
| AES-128-ECB decrypt | 1,120 bytes plaintext, valid PKCS7, complete message ending with `GKM` |
| App's revealed message (from screen recording) | matches the reproduced plaintext exactly |

## 5. Hypotheses Tested and Ruled Out

1. **Length prefix after the magic.** The four bytes after `@!#` are `Pk+5` — not a plausible integer length in either endianness. Ruled out.
2. **Filename (or its length) affects decoding.** The app renames the file on import (`1792033632892-3.png` → `1000005206.png`) and still decodes correctly. Ruled out.
3. **Alpha channel carries message bits.** Repacking with the alpha channel's 2 LSBs does not produce the `@!#` magic under any of the 24 channel orders. Ruled out.
4. **Reversed bit order (LSB-first within each 2-bit group).** Produces garbage magic bytes. Ruled out.
5. **App resizes the bitmap before decoding.** `m3/v.Q()` only tiles the bitmap into 512×512 chunks via `Bitmap.createBitmap(bitmap, x, y, w, h)`; no scaling. Ruled out — but the tiling itself was the missed detail (see §6).
6. **"Stop at first invalid Base64 byte" (the heuristic all three AIs used).** Contradicted by the source code, which scans for `#!@`. This heuristic is the cause of the three different truncations. Ruled out as the app's actual rule.
7. **(Prior revision's §6) "Binary-interleaved payload region".** RETRACTED — this was never real. Reading pixels row-major instead of the app's tiled order diverges at pixel 512, turning clean Base64 into apparent garbage from output byte ~384 on. With tiled order the region is 1,536 bytes of pure Base64. The corruption map (clean bytes concentrated in smooth image regions) is explained: row-major vs tiled misalignment is a fixed permutation, and smooth regions happen to survive it as readable text less often — no, more precisely, the misalignment maps each output byte to the wrong pixel, producing random bytes except where the permutation happens to align.

---

## 6. Open Gap — CLOSED 2026-10-17

**What was open:** the prior revision could not turn the payload region into the full plaintext. The region (read row-major) appeared to interleave clean Base64 lines with binary bytes, and strict filtering reproduced only the first 224 plaintext bytes.

**Root cause found:** the pixel walk order. The prior revision read pixels in full-image row-major order. The app reads them in **512×512 tiled order** (`m3/v.Q()` — confirmed by disassembling the APK still in the workspace, `La2/c.doInBackground` + `Lm3/v.Q`). Row-major diverges from tiled order at pixel 512 (output byte 384). Everything the prior revision called "binary interleaving" was simply the wrong pixels being read — the message was never corrupted.

**Verification (no Android runtime needed):** re-extracting with tiled pixel order gives `@!#` at byte 0, `#!@` at byte 1539, and a 1,536-byte region of pure clean Base64 (zero non-alphabet bytes). Faithful pipeline — ISO-8859-1 string, strip markers, `Base64.decode(bytes, 0)` with AOSP lenient semantics, AES-128-ECB decrypt with `################`, strip PKCS7 — yields **1,120 bytes of plaintext**, matching the app's revealed message from Gary's screen recording.


## 7. Key Reference (public by design)

- Empty-password key: `################` (16 × `0x23`)
- Key derivation: password padded with `#` to 16 chars; passwords longer than 16 chars are truncated to 15 (app bug, preserved here for fidelity)
- Cipher: AES-128-ECB (`Cipher.getInstance("AES")`), key = derived string bytes directly
- Payload: Base64 of ciphertext (Android `Base64.DEFAULT`: newlines every 76 chars, `=` padding), embedded as `@!#` + Base64 + `#!@`
