+++
date = '2026-10-10'
draft = false
title = 'Cracking a Human RNG'
description = 'A FortID CTF writeup: recovering a QR-encoded flag by exploiting how a human types "random" bits.'
tags = ['Crypto', 'Misc', 'CTF', 'FortID']
+++

**FortID CTF · Crypto / Misc**

`QR codes` · `Human RNG` · `Reed-Solomon` · `GF(2) linearity` · `Word search`

---

> You've heard about LCGs, LFSRs, MTs, Xorshifts, RDRANDs, Lavarands...
>
> But, can you crack a Human RNG?

**Flag:** `FortID{humans_are_not_good_random_number_generators}`

Files: `encrypt.py`, `ciphertext.png`, `bitstream.txt`, plus a 3-minute video of someone typing (`human_rng_at_work.mp4`).

```
python3 solve.py           # verify the flag
python3 solve.py search    # recover the unknown words from scratch (~1 minute)
```

You need `numpy`, `pillow`, `qrcode` and `wordfreq`.

## 1. What the encryption does

`encrypt.py`:

1. Encodes the flag as a QR code: version 6 (41×41 modules), error correction level H, mask 0, byte mode.
2. Reads `bitstream.txt` and takes its **last 1681 bits** (41 × 41) as a key.
3. XORs the QR modules (row by row) with the key and saves the result as `ciphertext.png`.

It also asserts the flag body is 44 characters of lowercase English words joined by underscores.

The handout's `bitstream.txt` keeps the first 1999 bits and replaces the last 1681 with `<1681-bits-redacted>`. You never see the key. You only see 1999 bits that the same human typed just before it.

## 2. Dead ends

**LFSR.** Berlekamp-Massey on the 1999 bits reports a linear complexity near 1000. That proves nothing: any random-looking sequence of length n gives about n/2. Rolling that "LFSR" forward produces garbage.

**The video.** It shows a hand typing about 1700 keystrokes (dont ask how I counted), close to 1681, so it looks like the key being typed. Initially i tried to look at the video frame by frame but couldn't plus the camera sits too low to see which key goes down. So I split keystrokes by sound spectrum, by gap length. None of them track the bit. Treat the video as flavour.
A real 0/1 split would reproduce the human's habit of switching keys about 70% of the time (next attack).

## 3. The weakness: the human

Look at the 1999 known bits:

| Statistic | Value |
|---|---|
| Fraction of ones | 0.475 |
| P(next bit differs from previous) | **0.702** |
| Best order-k Markov predictor | 70% accuracy at k = 1, no gain from more context |
| Compression vs. shuffled copy | no difference |

People who try to be random alternate too much. This one switches between 0 and 1 70% of the time, and has no other exploitable habit.

That gives a test for any flag guess. Build the guess's QR, XOR it with the ciphertext, and you get the key the guess implies. If the guess is right, that key came from the human and switches about 70% of the time. If the guess is wrong, the key is ciphertext XOR wrong QR, which switches about 50% of the time. Random flags score 0.52 to 0.58, because the fixed QR patterns are right for every guess.

## 4. Scoring a guess, block by block

The flag is 52 bytes. With 4 bits of mode and 8 bits of length in front, it fills 60 data codewords. Version 6-H splits them into 4 blocks of 15, and each block gets 28 Reed-Solomon bytes of its own. A wrong character scrambles its whole block's error-correction bytes but leaves the other blocks untouched. So you can score each block alone by counting switches only around the modules that block owns:

| Block | Flag characters |
|---|---|
| 0 | 0–13 (`FortID{` + body 0–6) |
| 1 | 13–28 (body 6–21) |
| 2 | 28–43 (body 21–36) |
| 3 | 43–51 (body 36–43 + `}`) |

A themed guess that fits the 44-character length, `humans_are_terrible_random_number_generators`, scores:

| Block | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| Guess | 0.62 | 0.49 | **0.69** | **0.70** |
| Max of 60 random flags | 0.56 | 0.55 | 0.54 | 0.57 |

Blocks 2 and 3 are right, so the flag ends in `…random_number_generators}`. Block 1 is wrong. Block 0 sits in between, because its score also counts module pairs that touch the wrong block 1. Openings other than `humans_` drop block 0 to random levels, so the body looks like:

```
humans_ ???????????? _random_number_generators
```

with 12 unknown characters. `are_terrible` and `are_horrible` both fit and both fail.

## 5. Searching the gap fast

Calling the `qrcode` library for millions of guesses is too slow. QR encoding is linear over GF(2) instead: placing bits, Reed-Solomon over GF(256) and the mask are all XOR-linear in the data bits. So:

1. Encode the flag once with the gap set to `aaaaaaaaaaaa`.
2. Flip each of the 96 gap bits in turn and record which modules change. That gives a 96 × 1681 matrix `A`.
3. Any gap text then gives `QR = QR_base ⊕ (bits ⊕ bits_base)·A`.

The switch indicators `s_i = k_i ⊕ k_(i−1)` are linear too. Only 498 of them depend on the gap. For a gap of three words `x_y_z`, the switch vector is `g_x ⊕ f_y ⊕ f_z`, where each part depends on one word. The number of switches is the Hamming weight of that vector. With ±1 encoding, the Hamming weight of `g ⊕ f` equals `(P − g·f)/2`. So scoring every `(x, y)` against every `z` is one matrix multiply.

Expected scores out of 498 pairs:

- True gap: about 0.70 × 498 ≈ 349.
- Best of millions of wrong guesses: about 330.

Using English words from `wordfreq`:

| Search | Best | Score |
|---|---|---|
| One 12-letter word | – | ≤ 315 |
| Two words (top 15,000 per length) | `crowned_tong` | 329 |
| Three words (300 × 1500 × 5000 per length split) | **`are_not_good`** | **355** |

`are_not_good` stands alone above the noise.

## 6. Check

```
FortID{humans_are_not_good_random_number_generators}
```

- Switch rate of the implied key in every block: 0.71, 0.70, 0.72, 0.71. The human's known prefix: 0.70.
- The implied key matches all 521 key bits that the fixed QR patterns and known flag text determine.

---

## The full solver

One script covers both modes from the top of the post: check the flag, or run the word search that recovers the 12 missing characters.

<details>
<summary><code>solve.py</code>: full source</summary>

```python
#!/usr/bin/env python3
# Human (FortID CTF) solver.
#   python3 solve.py           verify the flag
#   python3 solve.py search    recover the 12 middle chars by word search (a few minutes)
# needs: numpy pillow qrcode wordfreq
import sys
import numpy as np
from PIL import Image
import qrcode
from qrcode.constants import ERROR_CORRECT_H

SIZE = 41
img = np.array(Image.open('ciphertext.png').convert('1'))
ct = np.array([0 if img[y*10+5, x*10+5] else 1 for y in range(SIZE) for x in range(SIZE)], np.uint8)
prefix = [int(c) for c in open('bitstream.txt').read().split('<')[0] if c in '01']


def qr(flag: bytes):
    q = qrcode.QRCode(version=6, error_correction=ERROR_CORRECT_H, box_size=1, border=0, mask_pattern=0)
    q.add_data(qrcode.util.QRData(flag, mode=qrcode.util.MODE_8BIT_BYTE))
    q.make(fit=False)
    return np.array(q.get_matrix(), np.uint8).flatten()


def switch_rate(flag: bytes):
    key = ct ^ qr(flag)
    return np.mean(key[1:] != key[:-1])


def search(head: bytes, tail: bytes, n: int):
    from wordfreq import top_n_list
    # QR encoding is GF(2)-linear in the data bits: record what each bit of the gap flips.
    base = head + b'a'*n + tail
    c0 = qr(base)
    A = np.zeros((8*n, SIZE*SIZE), np.uint8)
    for i in range(8*n):
        f = bytearray(base); f[len(head) + i//8] ^= 0x80 >> (i % 8)
        A[i] = qr(bytes(f)) ^ c0
    # switch indicators s_i = k_i ^ k_(i-1) are linear too; keep only the ones the gap can change
    D = A[:, 1:] ^ A[:, :-1]
    k0 = ct ^ c0
    b = k0[1:] ^ k0[:-1]
    keep = D.any(0)
    D, b = D[:, keep].astype(np.int64), b[keep]
    P = len(b)

    def parity(words, offset):
        X = np.unpackbits(np.frombuffer(b''.join(words), np.uint8).reshape(len(words), -1) ^ ord('a'), axis=1)
        return (X @ D[8*offset:8*offset + X.shape[1]]) % 2

    pm = lambda v: 1 - 2*v.astype(np.float32)  # switches(g^f) = (P - pm(g).pm(f)) / 2
    words = {}
    for w in top_n_list('en', 50000):
        if w.isascii() and w.isalpha():
            words.setdefault(len(w), []).append(w.encode())

    best = (0, '')
    for a in range(1, n-1):                       # three words: a + c + d + 2 underscores = n
        for c in range(1, n-2-a):
            d = n-2-a-c
            X, Y, Z = words.get(a, [])[:300], words.get(c, [])[:1500], words.get(d, [])[:5000]
            if not (X and Y and Z):
                continue
            gx = parity([x+b'_' for x in X], 0) ^ b
            fy = parity([y+b'_' for y in Y], a+1)
            FZ = pm(parity(Z, a+c+2)).T
            for i, x in enumerate(X):
                S = (P - pm(gx[i] ^ fy) @ FZ) / 2
                j, k = divmod(int(np.argmax(S)), len(Z))
                if S[j, k] > best[0]:
                    best = (int(S[j, k]), (x + b'_' + Y[j] + b'_' + Z[k]).decode())
                    print(f'{best[0]}/{P} switches  {best[1]}', flush=True)
    return best[1]


if __name__ == '__main__':
    head, tail = b'FortID{humans_', b'_random_number_generators}'
    middle = search(head, tail, 12) if sys.argv[1:] == ['search'] else 'are_not_good'
    flag = head + middle.encode() + tail
    print(f'human prefix switch rate {np.mean(np.diff(prefix) != 0):.3f}')
    print(f'recovered key switch rate {switch_rate(flag):.3f}  (random flags give ~0.56)')
    print(flag.decode())
```

</details>
