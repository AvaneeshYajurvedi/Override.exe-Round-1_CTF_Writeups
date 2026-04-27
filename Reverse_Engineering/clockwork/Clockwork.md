# CTF Writeup: clockwork

**Flag:** `OVRD{clockwork_t1ck_t0ck_r3v}`  
**Category:** Reverse Engineering  
**Files:** `clockwork` (ELF binary), `flag.enc` (32-byte ciphertext)

---

## Overview

We're given a 64-bit ELF binary that takes an 8-character key as input, validates it through two independent hash-based stage checks, derives a 16-byte AES-128-CBC key, and decrypts `flag.enc`. The challenge is to reverse all three stages and recover the correct key.

---

## Step 1 — Recon

```bash
file clockwork flag.enc
# clockwork: ELF 64-bit LSB pie executable, x86-64, not stripped
# flag.enc:  data (32 bytes)

strings clockwork
# AES-128-CBC, SHA256, /proc/self/exe, "WRONG", "CORRECT", "Integrity check failed"
```

Key observations:
- Binary is **not stripped** — function symbols are intact, making reversing straightforward.
- `flag.enc` is exactly **32 bytes** → two AES-128-CBC blocks.
- `strings` reveals SHA256 is used, likely to derive the AES IV from `/proc/self/exe`.
- Two validation stages are visible: `stage1_check` and `stage2_check`.

---

## Step 2 — Reversing `check_alignment()`

Both stage checks share a common hash primitive. Disassembling it reveals a Murmur-hash-style function:

```c
uint32_t check_alignment(uint32_t x) {
    uint32_t r = x * 0x5bd1e995;
    r ^= (r >> 13);
    r = r * 0x5bd1e995;
    r ^= (r >> 15);
    return r;
}
```

This is essentially a Murmur2 finalizer — a fast, non-cryptographic hash that scrambles its 32-bit input.

---

## Step 3 — Reversing `stage1_check` and `stage2_check`

Both stages operate on half the key (4 bytes each) using a **chained accumulation** pattern:

```c
// Pseudocode for stage1_check(key[0:4])
uint32_t acc = 0;
for (int i = 0; i < 4; i++) {
    acc = check_alignment((int8_t)key[i] ^ (int32_t)acc);
}
return acc == 0xc5bf59ec;  // stage1 target

// stage2_check(key[4:8])
acc = 0;
for (int i = 4; i < 8; i++) {
    acc = check_alignment((int8_t)key[i] ^ (int32_t)acc);
}
return acc == 0xff3b20cb;  // stage2 target
```

Crucially, the accumulation is **chained** — each byte's hash output feeds into the next byte's input as XOR. This means the stages cannot be inverted analytically, but can be **brute-forced independently** since each half is only 4 bytes of printable ASCII (95⁴ ≈ 81 million combinations).

---

## Step 4 — Reversing `finalize_context()` (AES key derivation)

After both stages pass, `finalize_context()` derives the 16-byte AES key from the 8-character input key using two constants baked into the binary: `esi = 0x3e57e` and `edx = 0x536fe`.

```c
uint32_t r14 = esi ^ edx;  // 0x6d880

uint8_t aes_key[16] = {0};
for (int i = 0; i < 8; i++) {
    aes_key[i] = check_alignment((int8_t)key[i] ^ (int32_t)r14) & 0xFF;
}
aes_key[8]  = (esi >> 8) & 0xFF;   // 0x3E
aes_key[9]  = (edx >> 8) & 0xFF;   // 0x53
aes_key[10] = (esi ^ edx) & 0xFF;  // 0x80
aes_key[11] = check_alignment(esi + edx) & 0xFF;
// aes_key[12..15] = 0x00
```

---

## Step 5 — IV Derivation

The function `verify_integrity()` opens `/proc/self/exe`, reads the entire binary, computes `SHA256(binary)`, and uses the **first 16 bytes** as the AES IV. This is a self-integrity check — the IV depends on the binary not being patched.

```python
import hashlib
with open("clockwork", "rb") as f:
    iv = hashlib.sha256(f.read()).digest()[:16]
```

---

## Step 6 — Brute Force

With the stage targets extracted and the key search space confirmed to be printable ASCII (0x20–0x7E), we brute force each half independently in C for speed:

```c
// For each half: chain check_alignment() over 4 bytes
// and check if final accumulator == target
for (b0..b3 in printable_ascii) {
    acc1 = check_alignment(b0 ^ 0);
    acc2 = check_alignment(b1 ^ acc1);
    acc3 = check_alignment(b2 ^ acc2);
    acc4 = check_alignment(b3 ^ acc3);
    if (acc4 == TARGET) → found!
}
```

Compiled with `-O2`, both halves solved near-instantly:

```
[+] Stage1 key half: 'R7#m'
[+] Stage2 key half: 'Xq2!'
[+] Full key: 'R7#mXq2!'
[+] AES key:  c60e76457f1f1399e536802f00000000
```

---

## Step 7 — Decryption

```python
from Crypto.Cipher import AES
import hashlib

aes_key = bytes.fromhex("c60e76457f1f1399e536802f00000000")
iv = hashlib.sha256(open("clockwork","rb").read()).digest()[:16]

ciphertext = open("flag.enc","rb").read()
cipher = AES.new(aes_key, AES.MODE_CBC, iv)
plaintext = cipher.decrypt(ciphertext)

pad = plaintext[-1]
flag = plaintext[:-pad].decode()
print(flag)
```

```
OVRD{clockwork_t1ck_t0ck_r3v}
```

---

## Summary

| Step | Technique |
|------|-----------|
| Binary recon | `file`, `strings`, `objdump` |
| Stage validators | Reverse Murmur2-style chained hash |
| Key recovery | Brute force 95⁴ × 2 printable ASCII halves |
| AES key derivation | Reverse `finalize_context()` with embedded constants |
| IV derivation | SHA256 of binary → first 16 bytes |
| Decryption | AES-128-CBC with PKCS7 padding |

The central trick is recognizing that the two independent 4-byte stage checks can be brute-forced in the printable ASCII search space rather than inverted, and that the AES IV is self-referential (SHA256 of the binary itself).
