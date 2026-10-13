# FortID CTF: Game of Life rewind

**Flag:** `FortID{R3al_L1f3_H45_N0_R3w1nd_Bu7t0n}`

## The challenge

> I found this neat drawing program online and wrote the flag using one of my favorite fonts. I pressed the play button, and before I could react, the flag transformed itself into these weird shapes. Why is there no rewind button!?
>
> Note: The flag was rendered from one of the DejaVu 2.37 TTF files at an integer pixel size. Every rasterized pixel with an intensity of at least 50% became a live cell.

You get one file, `screenshot.png`: white squares on black, a Conway's Game of Life board some unknown number of generations after someone drew the flag on it.

## Why you can't rewind

Life has no inverse. Many boards evolve into the same board, and some boards (Gardens of Eden) have no predecessor at all. A generic SAT-based reverser would need to undo many generations of a 382-cell-wide board and would drown in valid but wrong predecessors.

The note gives you a way around that. You know exactly how the starting board was made: a DejaVu font, an integer size, and a 50% threshold. So you can run the clock forward instead. Guess the text, render it, simulate, and compare with the screenshot. A correct guess matches every cell.

## Step 1: Read the grid

Each cell is a 6×6 px square on an 8 px pitch. Sample the centre of each cell:

```python
a = np.array(Image.open('screenshot.png').convert('L')) > 128
ys, xs = np.nonzero(a)
T = a[ys.min() % 8 + 3::8, xs.min() % 8 + 3::8]   # 36 x 382, 909 live cells
```

## Step 2: Find the font, size and generation count

Assume the flag starts with the event's prefix, `FortID{`. For every DejaVu 2.37 TTF (22 files) and every size from 8 to 40 px:

1. Render `FortID{` with Pillow (FreeType) and keep pixels with intensity ≥ 128.
2. Evolve it from 0 to 60 generations.
3. Each generation, crop the columns the next character can't have touched yet (more on that below) and slide that crop over the screenshot. FFT cross-correlation computes the mismatch count at every offset in one pass.

Small fragments such as blinkers match by chance, so drop anything with fewer than 25 live cells. One result stands alone:

| font | size | generations | live cells | mismatches |
|---|---|---|---|---|
| DejaVuSansMono-Bold | 16 px | 18 | 87 | **0** |

The runner-up had half its cells wrong. The match also confirms the prefix and pins the canvas-to-grid offset.

Get the fonts from the official release, since the version matters for rasterisation:

```
https://github.com/dejavu-fonts/dejavu-fonts/releases/download/version_2_37/dejavu-fonts-ttf-2.37.tar.bz2
```

## Step 3: Recover the rest

Sans Mono gives every glyph the same advance (about 9.6 px at 16 px), so you know where character *k* sits before you know what it is.

### The light cone problem

A change in Life spreads at most one cell per generation. After 18 generations, character *k* can affect cells up to 18 columns to its left. In theory, then, the columns that depend only on known characters end 18 columns before the next unknown glyph.

My first attempt scored one character at a time against that safe region, and every candidate tied. Changes rarely travel at full speed, so the safe region almost never sees the new character. When I widened the window, greedy picks went wrong after `R3`, because the wider window also catches noise from characters further right.

I measured the real spread by evolving random strings that differ only in their last character. Changes reached 16.7 columns back. That's close to two characters, so each character has to be judged together with its neighbours.

### Beam search with an exact filter

For each step, extend every string in the beam by each character in `[A-Za-z0-9_{}!?-]`, then score each candidate on two regions:

- **Exact region** `[x(k) − 19, x(k+1) − 19)`: no later character can reach these columns. The true string has 0 mismatches here, so throw out any candidate with even one.
- **Soft region** `[x(k+1) − 19, x(k+1) − 4)`: future characters can still reach these columns, but they usually don't. Use the mismatch count here only to rank survivors. Keep the best 400.

Two tricks keep it fast. Simulate a narrow window around the region you're scoring (the window's edges sit more than 18 cells away, so their errors can't reach it), and evolve all candidates as one `(B, H, W)` numpy array.

The beam reads off the flag in about three minutes:

```
FortID{R3al_
FortID{R3al_L1f3_
FortID{R3al_L1f3_H45_N0_
FortID{R3al_L1f3_H45_N0_R3w1nd_
FortID{R3al_L1f3_H45_N0_R3w1nd_Bu7t0n
```

Any candidate ending in `}` goes on a final list. Evolve each finalist on the whole board and compare with the screenshot:

```
(0,   'FortID{R3al_L1f3_H45_N0_R3w1nd_Bu7t0n}')
(99,  'FortID{R3al_L1f3_H45_N0_R3w1nd_Bu7t0}')
(107, 'FortID{R3al_L1f3_H45_N0_R3w1nd_Bu7--}')
```

The winner reproduces all 909 live cells and no extras. If you add any character after the `n`, the pattern spills past the edge of the screenshot.

## Running it

```
pip install pillow numpy
tar xjf dejavu-fonts-ttf-2.37.tar.bz2
python3 solve.py screenshot.png dejavu-fonts-ttf-2.37/ttf            # known params, ~3 min
python3 solve.py screenshot.png dejavu-fonts-ttf-2.37/ttf --search   # also rediscovers font/size/gens
```

## Takeaways

- When an inverse is hard, look for a forward model you can brute-force. Here the font note shrank the search from "any board" to "any string".
- A known prefix let me fit every unknown parameter in one pass before guessing any secret characters.
- The theoretical light cone gives a safe region where the true answer must score zero. A soft score just outside it ranks the candidates, and you need both.
