# Abandoned Factory (FortID CTF)

> An abandoned factory hides a message within its tangled machinery. Untangle the construction and uncover the secret.

**Flag:** `FortID{Wh3n_7h3_8e1ts_b3c0me_5p49hett1_tr41n5_8ec0m3_th3_0n1y_r34s0n4b1e_50lut10n}`

The flag comes in three pieces, each hidden in a different layer of two Factorio blueprints. `solve.py` extracts all three:

```
python3 solve.py handout_file_1.txt handout_file_2.txt
```

You need `numpy`, `opencv-python` and `Pillow`.

## 1. Recognise the format

Both files start with `0eN`. Factorio blueprint strings look like that: one version character (`0`), then base64 of zlib-compressed JSON. Drop the first character, base64-decode, inflate, and you get JSON.

```python
json.loads(zlib.decompress(base64.b64decode(s[1:])))
```

File 1 holds a factory: 290 entities (belts, inserters, biochambers, combinators), 104 circuit wires, and 841 floor tiles. File 2 holds 330 landfill tiles and nothing else.

## 2. Part 1: the floor of file 1

File 1's 841 tiles form a 29×29 square (841 = 29²) that mixes two tile types, landfill and foundation. 29×29 matches a version 3 QR code. Paint landfill dark and foundation light, and the QR reads:

```
FortID{Wh3n_7h3_8e1ts_b3c0me
```

## 3. Part 2: file 1's entities XOR file 2's tiles

File 2's landfill fills cells of a 25×25 grid. Rendered alone, it looks like a QR code with scrambled finder patterns, and no reader accepts it.

File 1's entities also fit a 25×25 box (x from −66.5 to −42.5, y from −2.5 to 21.5). Mark every entity's position as one dark cell, XOR that grid with file 2's tiles, and all three finder patterns appear. The QR decodes to:

```
_5p49hett1_tr41n5_8ec0m3_
```

Use one cell per entity, even for big entities like the 3×3 biochamber. If you paint full footprints instead, about 18 cells come out wrong and the decode fails.

## 4. Part 3: the wires spell the rest

Draw every red and green wire as a line between the two entities it joins and you get a tangle of overlapping strokes. Some shapes stand out (a `Y`, a `4`, an `=`), but they sit on top of each other.

To untangle them, split the wires into connected networks. Treat each (entity, wire colour) pair as a node and union the two ends of every wire. You get 29 networks, and each one draws a single character.

To order the characters, read the circuit conditions on each network's entities. Most of them hold a bare constant with no signal attached, such as `{"constant": 24, "comparator": "<"}`. Every entity in a network carries the same number, and each network gets a different one. That number is the character's index.

Two networks carry no bare constant. Their entities hide the index inside signal conditions instead:

- One weird-looking glyph sits on entities with `signal-heart = 1`, `spoilage = 1`, `copper-ore = 1` and `pentapod-egg > 1`. It goes at index 1.
- A plain `_` sits on two inserters with `pentapod-egg > 9`. It goes at index 9.

Taking the most common non-zero constant in each network covers both cases. Sorted 1 to 29, the glyphs read:

```
th3_0n1y_r34s0n4b1e_50lut10n}
```

`solve.py` saves the sorted glyphs as `part3.png`, with the index under each one.

### Ambiguous glyphs

The glyphs are wire sketches between entity centres, so a few look like more than one character:

| Index | Correct | Looked like |
|---|---|---|
| 1 | `t` | `7`. The glyph runs through biochambers and decider combinators, whose wire points sit off-centre, so it looks bent. It has a crossbar and a vertical stroke like the `+` at index 25. |
| 13 | `s` | `5`. Part 2 writes `tr41n5` with a `5`, but this glyph is a lowercase `s`. |
| 15 | `n` | `7` |
| 18 | `1` | `l`. It has the flag stroke of a `1`. |
| 19 | `e` | `3` |

If your first guess fails, try swapping these characters before you change anything else.

## Full flag

```
FortID{Wh3n_7h3_8e1ts_b3c0me      (tile QR, file 1)
_5p49hett1_tr41n5_8ec0m3_         (entities XOR tiles QR)
th3_0n1y_r34s0n4b1e_50lut10n}     (wire glyphs)
```

"When the belts become spaghetti, trains become the only reasonable solution."
