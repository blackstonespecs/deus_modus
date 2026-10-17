# NoClue Message-Length Framing Rule — Decoder Spec

**Target file:** `1792033632892.png` (698,290 bytes, SHA256 `9c206f17cd654939c18ae62754723842373b29c32a2f908205588b461fb5d1a3`)
**Status:** Rule identified and proven from decompiled APK source. Full-plaintext reproduction has one open gap (see §6).

---

## 1. The Rule

NoClue does **not** use a length prefix. The message length is determined by a **3-byte terminator**:

- **Start marker:** `@!#` (bytes `0x40 0x21 0x23`)
- **End marker:** `#!@` (bytes `0x23 0x21 0x40`) — the start marker reversed

The decoder accumulates extracted bytes and stops the moment the accumulated string ends with `#!@`. Everything between the markers is the payload.

**Source:** `a2/AsyncTaskC0285c.java`, `doInBackground()` — the decode loop checks `str4.endsWith("#!@")` after every output byte and logs `"Decoding ended"`. The encode side constructs the embedded string as `"@!#" + Base64 + "#!@"`.

In the target file, the terminator sits at **byte 2253** of the LSB stream (the only occurrence in 421,875 bytes). The payload region is bytes 3–2252 (2,250 bytes).

---

## 2. Complete Extraction Procedure

For the public extraction page, implement exactly this:

1. **Load pixels.** Read the PNG as RGB. (The app uses Android `Bitmap.getPixels()` → ARGB ints, then takes R, G, B — alpha is ignored.)
2. **Build the byte array.** For each pixel in row-major order, append three bytes: `R = (pixel >> 16) & 255`, `G = (pixel >> 8) & 255`, `B = pixel & 255`.
3. **Extract 2 LSBs.** Walk the byte array sequentially. From each byte take its 2 least-significant bits. Pack four consecutive 2-bit groups into one output byte, most-significant group first:
   `out = ((b0 & 3) << 6) | ((b1 & 3) << 4) | ((b2 & 3) << 2) | (b3 & 3)`
4. **Accumulate and watch for markers.** Append each output byte to a buffer (interpreted as ISO-8859-1 / raw bytes). The first three bytes must be `@!#`; if the 3-byte accumulation isn't `@!#`, discard and keep scanning (the app resets). Stop appending when the buffer ends with `#!@`.
5. **Strip markers.** Remove the leading `@!#` and trailing `#!@`. What remains is the Base64 payload string.
6. **Base64-decode**, then **AES-128-ECB decrypt** with the 16-byte key (`################` for empty password — password padded with `#` to 16 chars, truncated to 15 if longer).
7. **Remove PKCS7 padding.**

---

## 3. Why Three AIs Truncated at Different Points

Muse, Claude, and ChatGPT all recovered the start correctly (`@!#`, Base64, AES-128-ECB, the `#` key) but each stopped at a different place. The reason: **we all guessed the end condition instead of reading it from the code.**

Each model applied a different "where does the Base64 stop?" heuristic:

- **Muse** stopped at the first byte that wasn't valid Base64 (byte 307 → 224 bytes plaintext).
- The other two models used looser/tighter validity rules and stopped elsewhere.

The app never checks Base64 validity to find the end. It scans for the literal `#!@` terminator. Any decoder that guesses the length from content heuristics instead of the terminator will truncate at an arbitrary point — which is exactly what happened.

**Lesson for the extraction page:** implement the terminator scan. Do not stop at the first non-Base64 byte.

---

## 4. Measurements (facts, no interpretation)

| Item | Value |
|---|---|
| LSB stream size | 421,875 bytes (750×750 px × 6 bits ÷ 8) |
| Start marker `@!#` | byte 0 |
| End marker `#!@` | byte 2253 (sole occurrence) |
| Payload region | bytes 3–2252 (2,250 bytes) |
| Clean 76-char Base64 lines in region | 8 (at stream bytes 3, 80, 157, 234, 1130, 1207, 1284, 1361) |
| First 304 Base64 chars → decrypt | 224 bytes, 100% printable ASCII, valid PKCS7, ends mid-sentence at `"(Deus Modus/Singul` |
| App's revealed message (from screen recording) | ~1,150 chars, complete sentences, ends with `GKM` |

---

## 5. Hypotheses Tested and Ruled Out

1. **Length prefix after the magic.** The four bytes after `@!#` are `Pk+5` — not a plausible integer length in either endianness. Ruled out.
2. **Filename (or its length) affects decoding.** The app renames the file on import (`1792033632892-3.png` → `1000005206.png`) and still decodes correctly. Ruled out.
3. **Alpha channel carries message bits.** Repacking with the alpha channel's 2 LSBs does not produce the `@!#` magic under any of the 24 channel orders. Ruled out.
4. **Reversed bit order (LSB-first within each 2-bit group).** Produces garbage magic bytes. Ruled out.
5. **App resizes the bitmap before decoding.** `m3/AbstractC2185v.Q()` only tiles the bitmap into 512×512 chunks; no scaling. Ruled out.
6. **"Stop at first invalid Base64 byte" (the heuristic all three AIs used).** Contradicted by the source code, which scans for `#!@`. This heuristic is the cause of the three different truncations. Ruled out as the app's actual rule.

---

## 6. Open Gap and the Test That Settles It

**What I proved:** the length-framing rule is the `#!@` terminator (source-code level, not a guess).

**What I could not yet do:** turn the 2,250-byte payload region into the full ~1,150-char plaintext the app reveals. The region between the markers is not clean Base64 throughout — valid 76-char lines are interleaved with binary bytes. Filtering strictly to Base64 characters yields 1,354 chars, but decrypting that produces correct output only for the first 224 bytes, then garbage. The binary-interleaved sections do not survive a strict Base64 decode.

**The single test that settles it:** run the app's actual `doInBackground` decode path on the target PNG inside an Android runtime (or instrumented unit test) and log the exact string passed to `Base64.decode()`. That reveals one of two outcomes:

- (a) The string is clean Base64 → my Python LSB stream diverges from the app's after some point, and the logged string shows where.
- (b) The string contains the binary bytes → Android's `Base64.decode(str, 0)` is skipping them leniently, and the logged string shows the exact skip behavior to replicate.

Either outcome gives the public extraction page a byte-exact reproduction recipe. The terminator rule in §1–§2 is unaffected by this gap — it is proven independently from the decompiled source.

---

## 7. Key Reference (public by design)

- Empty-password key: `################` (16 × `0x23`)
- Key derivation: password padded with `#` to 16 chars; passwords longer than 16 chars are truncated to 15 (app bug, preserved here for fidelity)
- Cipher: AES-128-ECB (`Cipher.getInstance("AES")`), key = derived string bytes directly
- Payload: Base64 of ciphertext (Android `Base64.DEFAULT`: newlines every 76 chars, `=` padding), embedded as `@!#` + Base64 + `#!@`
