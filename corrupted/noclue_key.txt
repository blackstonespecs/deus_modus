NoClue Steganography App - Decryption Key (No Password)
=======================================================

Key (16 bytes, ASCII): ################
Key (hex): 23232323232323232323232323232323

Derivation: Empty password -> pad with '#' to 16 characters.
For password "X": pad "X" with '#' to 16 chars (e.g., "a" -> "a###############").

Algorithm: AES-128-ECB
Format: "@!#" (3 bytes) + Base64(AES-128-ECB(plaintext with PKCS7 padding))

To decrypt:
1. Extract 2 LSBs from R,G,B channels (row-major, MSB-first bit packing).
2. Verify magic bytes "@!#" (0x40 0x21 0x23).
3. Take the bytes after magic as Base64 string (may contain newlines every 76 chars).
4. Base64-decode to get ciphertext.
5. AES-128-ECB decrypt with the key above.
6. Remove PKCS7 padding.

Python example:
  from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
  import base64
  key = b'################'
  ct = base64.b64decode(b64_string)
  cipher = Cipher(algorithms.AES(key), modes.ECB())
  pt_padded = cipher.decryptor().update(ct)
  pad_len = pt_padded[-1]
  pt = pt_padded[:-pad_len]
