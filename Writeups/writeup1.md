# Human (FortID CTF)

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

**LFSR.** Berlekamp-Massey on the 1999 bits reports a linear complexity near 1000. That proves nothing: any random-looking sequence of length n gives about n/2. Rolling that "LFSR" forward produces garbage. (`solve_human.py` in this folder is that failed attempt.)

**The video.** It shows a hand typing about 1700 keystrokes, close to 1681, so it looks like the key being typed. I split keystrokes by sound spectrum, by gap length and by hand motion. None of them track the bit. A real 0/1 split would reproduce the human's habit of switching keys about 70% of the time (section 3). Every feature switches label about 50% of the time, like random labels. The camera sits too low to see which key goes down. Treat the video as flavour.

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

## Takeaways

- Human "random" bits alternate too often. 0.88 bits of entropy per bit sounds close to 1, but over 1681 bits that leaks about 200 bits of information.
- QR error correction works against the challenge. Its Reed-Solomon bytes turn a wrong guess into noise, and splitting the data into blocks lets you check each part of the flag separately.
- Berlekamp-Massey always "finds" an LFSR of length n/2 in random data. Don't trust that result.
