# DES (DESBox) — Solve Guide

Interactive "DOS shell" exposing a 3DES oracle. `KEYB` loads two 24-byte keys,
encrypts the flag under **Key A** (and prints it), then decrypts any ciphertext
you supply under **Key B** (and prints it). Goal: recover the flag.

## Key idea — DES complementation property

For single DES and for 3DES-EDE:

```
E_{~K}(~P) = ~E_K(P)      and equivalently      D_{~K}(~C) = ~D_K(C)
```

So if we make **Key B = ~Key A** (bit-complement all bytes), then feeding the
flag-ciphertext's complement into the Key-B decrypt oracle returns `~flag`:

```
D_B(~flag_ct) = D_{~A}(~flag_ct) = ~D_A(flag_ct) = ~flag
```

Complement the result → flag. **Key bytes never need to be known**, and the
secret `MASK` cancels out (see below).

## Why it's not blocked outright

`read_keys()` rejects obvious complements:
- `key_a == key_b`
- equal effective bytes (`b & 0xFE`, i.e. ignoring DES parity LSB)
- same effective-byte multiset
- `is_effective_complement(a, b)` — direct complements

and `load_key` does `key = xor(permute(user_key), MASK)` then fixes parity — a
**random permutation** of your bytes, xored with a resident secret `MASK`.

The parity LSB doesn't matter (DES ignores it; the property holds on the top 7
bits). `MASK` cancels because the condition we actually need is:

```
perm(B) ^ MASK  ==  ~(perm(A) ^ MASK)   <=>   perm(B) == ~perm(A)
```

So we only need `perm(B) == ~perm(A)` after the server's random shuffles.

## The construction

- **Key A** = `42 41 41 … 41`  (one `0x42`, twenty-three `0x41`)
- **Key B** = `BE BD BE … BE`  = complement of A with the first two bytes swapped
  → `[~0x41, ~0x42, ~0x41×22]`

Key B has the **same byte-multiset as `~A`** (`{~0x42, ~0x41×23}`), so some
permutation makes `perm(B) == ~perm(A)`. The swap keeps A and B clear of all
four guards (their effective multisets and complement checks differ in the
first two positions, since `0x40 != 0x42`).

Both keys are shuffled independently; the sequences line up exactly when the
single distinguished byte lands in the matching slot → **probability 1/24 per
`KEYB` load**. Just loop `KEYB` and keep the result whose Key-B debug output is
**all printable ASCII** — that's `~flag`, so complement it back.

## Run it

```bash
python3 solve_des.py <host> <port>
```

Expected: a hit within a few dozen loads (mean ≈ 24), printing
`FLAG: FortID{...}`.

Local self-test (needs `pip install pycryptodome` and a dummy
`secret.py` with `FLAG`/`MASK` next to `DESBox.py`):

```bash
python3 solve_des.py local <dir>
```

Validated locally: recovered the planted flag after 31 loads.
