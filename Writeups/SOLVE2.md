# Hacker Cinema — Solve Guide

PHP web app. Buy free movie tickets (ECDSA-signed QR). `/check` scans a QR and,
if the ticket is **valid, movie=Hackers, screen=mainframe-1, winner=true**,
prints the flag. You cannot buy a `winner=true` ticket (it needs a SHA-256
all-zero preimage), so you must **forge** one — which needs the signing key.

## The vulnerability: deterministic ECDSA nonce + two disagreeing calendars

Tickets are ECDSA (secp256k1) signed with a *deterministic* nonce:

```
k      = HMAC(nonce_key, movie|screen|seat| kiosk_partition_date(ts) |winner)   # src/tickets.php
message= canonical_json({ date: ticket_date_from_timestamp(ts), movie, screen, seat, winner })  # signed
```

The nonce's `date` uses `kiosk_partition_date` =
`new DateTimeImmutable('@'.$ts)`, but the **signed message's** `date` uses
`ticket_date_from_timestamp` = `DateTimeImmutable::createFromFormat('U', $ts)`
(`src/calendar.php`). For normal timestamps these agree — but near **year 0000**
the two proleptic-calendar implementations drift by one day. `config.php` sets
`MIN_SCREENING_TS = -70000000000` specifically to let you reach that era, and
`/buy` accepts a raw `screening` timestamp.

Concretely:

| ts            | kiosk date (→ nonce) | ticket date (→ message) |
|---------------|----------------------|--------------------------|
| `-62164798000`| `0000-01-29`         | `0000-01-29`             |
| `-62164711600`| `0000-01-29`         | `0000-01-30`             |

Same seat + same kiosk date + `winner=false` ⇒ **identical nonce k ⇒ identical r**.
Different ticket date ⇒ **different message z**. That is textbook ECDSA nonce reuse:

```
k = (z1 - z2) / (s1 - s2)  mod n
d = (s1*k - z1) / r        mod n
```

With `d` (the signing private key) you sign a forged `winner=true` ticket.

## End-to-end steps

1. `POST /buy` with `seat=C07`, `screening=-62164798000` → follow redirect → `GET /ticket.png?id=...`
2. Same with `screening=-62164711600`.
3. Decode each ticket PNG's QR (the QR panel sits at x=812,y=134, 240×240 in the
   1120×460 image — crop + upscale, then decode; the whole-image decode is
   unreliable because the server down-samples the QR). Each QR is the envelope
   JSON `{"ticket":{...},"signature":{"r":...,"s":...}}`.
4. Confirm `r1 == r2`, then recover `k` and `d` with the formulas above.
   `z_i = SHA256(canonical_json(ticket_i)) mod n`, where `canonical_json` =
   compact JSON, **keys sorted as strings**, booleans `true/false`, slashes
   unescaped (matches PHP `canonical_json`).
5. Forge `ticket = {date:"0000-01-29", movie:"Hackers", screen:"mainframe-1",
   seat:"C07", winner:true}`, ECDSA-sign `canonical_json(ticket)` with `d` and
   any nonce, assemble the envelope, render it as a QR PNG.
6. `POST /check` (multipart field `qr`) with that PNG → **flag**.

## Run

```bash
pip install requests segno opencv-python-headless numpy pillow
python3 solve_hacker.py https://<instance-host>        # or http://host:port
```

### Validation done offline (no live instance needed to trust it)
Using the real `src/*.php` with fixed seeds and PHP 8.4.16 (matching the
Dockerfile): the two tickets share `r`; the recovered `d` equals the real
`private_key()`; the Python-forged envelope passes the real `verify_envelope`
with `winner=true` and satisfies the win condition. The QR crop+upscale decode
round-trips the envelope through the server's down-sampled layout.

> Note: each instance has its own random signing/nonce seed, so `d` is recovered
> fresh per instance from its own two tickets — nothing is hardcoded.
