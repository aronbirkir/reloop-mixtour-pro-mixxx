# Reloop Mixtour Pro — captured LED palette

Hardware capture (live, 2026-06-03). LED color is set by the MIDI **value** byte
(not the channel). Two LED families:

## Button-channel LEDs (status 0x90–0x93: play 0x00, cue 0x01, sync 0x02, in-out 0x03, etc.)

These are **bi-color (red + blue)** LEDs — only red / blue / violet(both) + brightness.
Usable values (from the low range; both a dim "border" and a bright "full" exist):

| Color  | dim (border) | bright (full) |
|--------|--------------|---------------|
| OFF    | 0            | —             |
| red    | 1            | 5             |
| blue   | 2            | 6             |
| violet | 3            | 7             |

- Values 8–63 = the controller's built-in pulse/animation states (unused — we blink via timer).
- Values 64–127 repeat the pattern [off, red, blue, violet] every 4 (n%4: 0=off,1=red,2=blue,3=violet), bright.
- Implication: play/cue/sync use **blue** (bright 6 / dim 2). In-Out-Ex loop uses the **deck color**:
  decks 1–2 (index 0,1) = blue, decks 3–4 (index 2,3) = red; dim=idle, bright=active.
  No green/yellow/white available on buttons.

## Pad-channel LEDs (status 0x94/0x95 …, notes 0x14–0x17)

Full RGB, rich but irregular palette. **Repeats every 64 values** (0–63 = 64–127), so the
distinct palette is 0–63. Value 0 = OFF (unlit pad shows pale white plastic). Chosen
representative values (best-saturated picks from the photo capture — tune live):

| Color   | Value | Notes |
|---------|-------|-------|
| off     | 0     | unlit |
| blue    | 3     | (also 1,2) |
| green   | 4     | (also 8) greens only live in 0–15 |
| cyan    | 5     | |
| red     | 32    | (also 16, 48) |
| white   | 25    | (also 46) |
| yellow  | 24    | (also 44, 60) |
| orange  | 36    | (also 20, 40, 52) |
| magenta | 17    | (also 33, 49) |
| pink    | 37    | salmon/pink (also 21, 53) |
| violet  | 35    | (also 19, 43, 59) |

**Brightness rule (confirmed live):** higher value = brighter for the same hue, and the
hue wheel repeats every 64. So **ambient = base value (0–63), bright = base + 64** (same
hue, brighter). The COLORS table in the script uses {ambient: base, bright: base+64}.
Hot Cue is special (unset = OFF, set = the cue's color). All values are tunable live.

## djay Pro startup sequence (captured "To Reloop Mixtour Pro", 2026-06-04)

- **Advanced ("Serato") mode enable:** SysEx `F0 00 20 7F 00 F7` then `F0 00 20 7F 19 F7`.
  Required after every controller power-cycle or the mode macro / picker / combo notes are
  inert. Sent from the script's `init()`.
- **Mode-picker highlight** (host-driven): mode notes `0x00-0x07` on the deck pad channel;
  active mode = `0x7F`, inactive = `0x3F`.
- **Mode-hold transport layer LEDs:** `27/28/2A/2B/2F/30` = `03` (dim violet),
  `2C/2E` = `07` (bright violet), per deck button channel.
- **VU meters (CONFIRMED, all decks):** CC `0x1F` on `B0/B1/B2/B3`, value = **segment
  count 0-7** (not 0-127).
- **FX paddles:** plain = `98-9B 05` (engage Effect Unit 1, mirror 7F/00);
  **shift+paddle = `98-9B 00`** (backspin). FX param buttons: plain `9F 03/04` (param
  -/+), **shift = `98-9B 0B/0C`** (effect select prev/next).
- **Other addresses:** paddle LEDs `98-9B 05` (idle 01, engaged 05); filter LED
  `94-97 24` (idle 01 = dim blue, engaged 67 = bright blue); pfl init `1B = 01`;
  note `0x12` = djay crossfade-FX toggles; load LED `0x0A`.
- **Pad pink:** palette value 21 renders white on hardware; use **37** (bright 101).
