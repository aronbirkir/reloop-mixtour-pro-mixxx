# Reloop Mixtour Pro — Mixxx Mapping (4-deck) — Design

Date: 2026-06-02

## 1. Goal

Finish the Mixxx controller mapping for the **Reloop Mixtour Pro**, starting from the
MIDI-learned `ReloopMixTourPro.midi.xml`. Add the three things the learned mapping
lacks:

1. **LED feedback** to the controller (button states, pad states, VU meters).
2. **SHIFT** layer and **pad-mode** operations.
3. A **`ReloopMixTourPro-script.js`** script file (per the Mixxx MIDI scripting wiki)
   that holds all stateful logic.

Scope is **full-featured** and **4-deck**, matching the feature set the controller
exposes in Rekordbox and djay Pro, but expressed in Mixxx-idiomatic terms.

### References used
- `ReloopMixTourPro.midi.xml` — the existing MIDI-learned starting point.
- `Reloop Mixtour Pro Rekordbox Mappings.csv` — full controller MIDI map; reveals that
  button LED feedback is sent to the *same* address as the button input.
- `Reloop MIXTOUR PRO Edit.djayMidiMapping` — Apple plist; cross-reference for
  addresses and the controller's full capability (8 pad modes, 4 decks, loops, FX).
- `Pioneer-DDJ-FLX4-SB2-script.js` — structural template for a complete Mixxx script
  (a `lights` object, tracked shift, pad-mode state machines, output callbacks).

## 2. The core principle: MIDI channel = deck number

The controller is fully **channel-multiplexed**. Each of the two physical sides can be
flipped between two "virtual decks" (left = decks **1/3**, right = decks **2/4**), and
when a side flips, the controller **re-transmits all of that side's controls on the new
deck's channel** and recolors the LEDs (blue = decks 1/2, red = decks 3/4).

Verified by capture: with the left side on deck 3, `play` → `92 00`, `cue` → `92 01`,
`pfl` → `92 1B`, `volume` → `B2 1C`.

| Deck | Color | Buttons (note-on) | CCs | Pads / Load |
|------|-------|-------------------|------|-------------|
| 1    | blue  | `0x90`            | `0xB0` | `0x94`    |
| 2    | blue  | `0x91`            | `0xB1` | `0x95`    |
| 3    | red   | `0x92`            | `0xB2` | `0x96`    |
| 4    | red   | `0x93`            | `0xB3` | `0x97`    |

**Consequences (this is what makes 4-deck support clean):**
- The **deck-switch *input routing* is handled by the controller** (hold a mode button +
  press Load): it re-channels inputs and emits a switch *notification* (`90/91/92/93 08`).
  The script does not implement the toggle logic — but it **must respond to the
  notification by sending the correct LED messages** (see next bullet and §10).
- We **map all four decks directly** for input. Per-deck logic loops over decks 1–4.
- **The script/app owns all LED output — nothing is auto-managed by the controller.**
  This includes deck **color** and which deck is **selected**. Color is *channel-derived*
  (send a play LED to `0x92` → red deck 3; to `0x90` → blue deck 1) but it is **not
  automatic**: the script must actively send it. To do this the script **tracks each
  side's current virtual deck** (`leftDeck ∈ {1,3}`, `rightDeck ∈ {2,4}`), the active
  side, and split state, so it can light the correct channels/selectors.
- The user must **enable 4 decks in Mixxx** (Preferences → Decks, or a 4-deck skin).

## 3. File structure

Two files, following Mixxx conventions:

- **`ReloopMixTourPro.midi.xml`**
  - Keeps the working **analog controls as direct declarative bindings** (channel
    faders, EQ `parameter1-3`, filter `super1`, `pregain`, master `gain`, headphone
    `headGain`/`headMix`, crossfader, library browse). These already work — left intact,
    extended to decks 3/4.
  - Adds `<scriptfiles>` referencing the JS file.
  - Converts buttons that need shift/mode/loop logic to `<script-binding>`.
  - Adds an `<outputs>` section for *simple, mode-independent* LEDs (e.g. pfl).
- **`ReloopMixTourPro-script.js`** — all stateful logic: shift tracking, the pad state
  machine, mode-dependent pad LEDs, the In-Out-Ex loop state machine, FX, VU meters.

**Rule of thumb:** a control lives in the script if it depends on shift, on pad mode, on
a multi-press state, or drives a mode-dependent LED. Pure analog controls stay
declarative.

### Confirmed/working analog mappings (do not change semantics)
From the existing learned XML, confirmed correct by the user (note: the controller has
**no tempo/pitch fader** — Rekordbox's "TempoSlider" label does not apply here):

- `0xB0..0xB3` `0x16` → `[ChannelN] pregain`  *(working)*
- `0xBF 0x0A` → `[Master] gain` (main volume) *(working)*
- `0xB0..0xB3` `0x17/0x18/0x19` → EQ `parameter3/2/1` (high/mid/low)
- `0xB0..0xB3` `0x1A` → `[QuickEffectRack1_[ChannelN]] super1` (filter)
- `0xB0..0xB3` `0x1C` → `[ChannelN] volume`
- `0xBF 0x08` → `[Master] crossfader`
- `0xBF 0x0C` → `[Master] headGain`, `0xBF 0x0D` → `[Master] headMix`
- `0xBF 0x00` → `[Library] MoveVertical` (browse), `0x9F 0x06` → `[Library] GoToItem`

## 4. Modifiers — SHIFT and MODE

Two modifier states drive secondary functions across the mapping:

- **SHIFT** — message `9F 00 7F` (press) / `9F 00 00` (release), status `0x9F` (global
  ch 16). A script handler sets `ReloopMixTourPro.shift = (value === 0x7F)`. No timers, no
  latching. Lives on the same status byte as the library buttons (`9F 06`, `9F 07`) and the
  SPLIT button (`9F 09`) — no conflict.
- **MODE** — a side **selector button held** (`94 08` left / `95 08` right); the script
  tracks `selectorHeld[side]` (§5.4). Holding a selector turns the pads into the mode
  picker (§5.6) **and** turns the transport buttons into loop/tempo functions (§6). Depends
  on the selector emitting a note-off on release (§11 item 1).
- Combined layers used across the mapping: none / SHIFT / MODE / SHIFT+MODE (see §6 matrix).

## 5. Pad system (faithful hardware mirror)

### 5.1 Physical layout
8 pads, 2 columns × 4 rows. Notes `0x14`–`0x17` (rows), column selected by status byte
(left column = the left-side deck's channel, right column = the right-side deck's).

| Row | Left column (legend) | Right column (legend) |
|-----|----------------------|------------------------|
| 1   | Hot Cue              | Auto Loop              |
| 2   | Bounce Loop          | Sampler                |
| 3   | Pitch Cue            | Saved Loops            |
| 4   | Instant FX           | Neural Mix             |

Captured example (left side = deck 1, right side = deck 2):
left column `94 14..94 17`, right column `95 14..95 17`.

### 5.2 Selectors / routing (script-driven LEDs)
These arrive as **inputs**; the script updates routing state **and drives the matching
LEDs** (the controller does not self-manage them — see §10):
- `94 08 7F` → left side active, **split off**, this side = deck 1
- `95 08 7F` → right side active, **split off**, this side = deck 2
- `9F 09 7F` → **split on**
- `90/91/92/93 08` → per-deck selection indicator emitted on virtual-deck switch; the
  script lights the new deck's indicator and clears the old one.

### 5.3 Two pad behaviors
1. **Performance (no selector held):** pads trigger the current mode's action.
   - **Split ON:** column's channel routes directly to its deck (4 slots per deck).
     Because channel = deck, this is automatic.
   - **Split OFF:** all 8 pads act on the active side's deck (8 slots; left column =
     slots 1–4, right column = slots 5–8). Requires tracked active deck.
2. **Mode-select (hold Ch1 `94 08` / Ch2 `95 08`, then tap a pad):** sets *that deck's*
   pad mode to the tapped pad's printed legend. While the selector is held, the pads show
   the **mode-picker overlay** (see §5.6).

### 5.4 State held by the script (per side / per deck)
- `split` (bool).
- `activeSide` (left/right) for split-off routing.
- `padMode[deck]` for decks 1–4.
- `selectorHeld[side]` to distinguish mode-select from performance taps.

### 5.5 Pad modes
Implemented now (clean Mixxx mappings):

| Legend       | Mixxx mapping (pad N)                        | Shift action            | Pad LED source              |
|--------------|----------------------------------------------|-------------------------|-----------------------------|
| Hot Cue      | `hotcue_N_activate`                          | `hotcue_N_clear`        | `hotcue_N_enabled`/`_status`|
| Auto Loop    | `beatloop_{size}_activate`                   | (sizes vary per pad)    | `beatloop_{size}_enabled`   |
| Bounce Loop  | `beatjump_{size}_forward`                    | `beatjump_{size}_backward` | momentary                |
| Sampler      | `[SamplerN] cue_gotoandplay` / load if empty | `[SamplerN] stop`       | `[SamplerN] play`/track loaded |

Default sizes: beatloop pads `1, 2, 4, 8, 16, 32, 1/2, 1/4`; beatjump pads `1, 2, 4, 8, …`.

Stubbed (clearly commented, easy to extend later):
- **Pitch Cue** (PitchPlay / keyboard-mode style) — Mixxx has no native pitch-play;
  candidate: hotcue + pitch offset, or `[ChannelN] key` shifts.
- **Saved Loops** — candidate: hotcues stored as loops.
- **Instant FX** — candidate: momentary effect-chain enable.
- **Neural Mix** (stems) — candidate: Mixxx 2.5+ stem controls if the build supports them.

### 5.6 Pad LED colors per mode (matches djay Pro)
**Pad color is encoded in the MIDI *value* (a palette/brightness byte), not the channel**
— the channel still selects the deck, but the value carries color (and brightness within
that color). The exact value→color map is hardware-specific and a capture item (§11).
Each mode renders its pads from the routed deck's state:

| Mode         | Idle pad                                              | Active / pressed                                    | Mixxx source(s)                                  |
|--------------|-------------------------------------------------------|-----------------------------------------------------|--------------------------------------------------|
| Hot Cue      | **off** if unset                                      | **bright in the hot cue's own color**               | `hotcue_N_status` + `hotcue_N_color` → palette   |
| Auto Loop    | **ambient blue**                                      | **bright blue** for the active loop size            | `beatloop_{size}_enabled`                         |
| Bounce Loop  | **ambient green** (or magenta)                        | **bright** on the pressed pad (momentary)           | momentary (beatjump has no held state)            |
| Sampler      | **ambient pink**                                      | **bright while the sample plays**                   | `[SamplerN] play` / track loaded                  |
| Pitch Cue\*  | **ambient red**; pad 1 **bright white** (reset pitch) | selected pad **bright red**                         | — (stub; no native Mixxx pitch-play)              |
| Saved Loops\*| **ambient blue**                                      | **bright blue**, **blinking while playing**         | — (stub; hotcue-as-loop candidate)               |
| Instant FX\* | **ambient green**                                     | **bright green** while the FX is on                 | — (stub; effect-enable candidate)                 |
| Neural Mix\* | pads 1–4 **bright yellow**, pads 5–8 **ambient white**| pads 1–4 → **ambient yellow** when that stem is off  | stems (vocals/harmonic/bass/drums), Mixxx 2.5+ (stub) |

\* Stubbed modes (see §5.5): the **color/behavior above is the target**, but the
underlying action is deferred to a later pass (or pending Mixxx feature support).

**Hot Cue color mapping:** Mixxx exposes each hot cue's RGB via `hotcue_N_color`; the
controller has a fixed palette, so a small **nearest-palette-color** helper maps Mixxx
RGB → the closest controller value. Pads with no cue set are off.

**Mode-picker overlay:** while a side's selector (Ch1 `94 08` / Ch2 `95 08`) is **held**,
the pads switch to the mode picker — all 8 pads show **ambient white**, and the pad for
that deck's **currently active mode** shows **bright white**. Releasing the selector
restores the normal per-mode colors above. (Depends on the selector sending a note-off on
release — §11 item 1; if it latches instead, the same overlay is driven by the SHIFT+pad
fallback.)

## 6. Transport, loop, and LEDs

Transport buttons are script-bound (so modifiers + LEDs work), per deck channel. **Two
modifiers** combine with them: **SHIFT** (`9F 00`) and **MODE** (a side selector held,
`94 08` / `95 08` — depends on the selector reporting release, §11 item 1). The target
deck is always the pressed button's own channel. Behavior matrix:

| Button (note)         | plain               | SHIFT                | MODE (selector held)            | SHIFT + MODE                     |
|-----------------------|---------------------|----------------------|---------------------------------|----------------------------------|
| **PLAY** `0x00`       | `play`              | `start_stop`†        | —                               | **tempo nudge +0.1 BPM**         |
| **CUE** `0x01`        | `cue_default`       | `cue_gotoandstop`†   | —                               | **tempo nudge −0.1 BPM**         |
| **SYNC** `0x02`       | `sync_enabled`      | `sync_master`†       | **loop double** (`loop_double`) | **loop out / move** (`loop_out`) |
| **IN-OUT-EX** `0x03`  | 3-state loop (§6.1) | clear loop†          | **loop halve** (`loop_halve`)   | **loop in / move** (`loop_in`)   |

† SHIFT-only actions are suggested defaults (not specified by the hardware reference) —
easy to change. The MODE and SHIFT+MODE columns replicate djay Pro behavior as requested.

`0x1B` **pfl** has no modifier function, so it stays a declarative binding + `<output>`.

**Tempo nudge (±0.1 BPM):** the script reads the deck's `file_bpm` and adjusts `rate` by
the amount equal to 0.1 BPM (`Δrate = (0.1 / file_bpm) / rateRange`), giving djay's precise
0.1-BPM step. (Can fall back to `rate_perm_up_small` / `rate_perm_down_small` if a
fixed-percentage step is acceptable.)

LEDs: play/cue blink per §6.2; sync follows `sync_enabled`; In-Out-Ex uses ambient → blink
→ bright per loop state (§6.1–§6.2).

### 6.1 In-Out-Ex 3-state loop machine (per deck)
Per-deck `loopState ∈ {0,1,2}`:
- **Press 1 (state 0):** trigger `loop_in`; → state 1 (arming, loop start set).
- **Press 2 (state 1):** trigger `loop_out`; loop becomes active; → state 2.
- **Press 3 (state 2):** if `loop_enabled`, trigger `reloop_toggle` to exit; reset → 0,
  so the next press begins a fresh loop.

The machine also resyncs to `loop_enabled` (e.g. if a loop is set/cleared by pads) so
the button never gets out of step. Optional: shift + In-Out-Ex = clear loop immediately.

### 6.2 LED rendering — color, brightness, and blink rates (matches djay Pro)
LEDs encode **color via channel** (blue decks 1/2 on `0x90/0x91`, red decks 3/4 on
`0x92/0x93`) and **brightness via value** (`OFF` `0x00`, `AMBIENT` dim backlight,
`BRIGHT` `0x7F`). Two **independent** blink cadences are used (they are *not* synced —
matching djay): **slow** (~700 ms period, play) and **fast** (~300 ms period, cue &
loop-arming). Exact ambient value and rates are tuning constants (see §11).

- **Play** — `play_indicator` only blinks at end-of-track, so the script drives it:
  playing → **BRIGHT solid**; loaded & paused → **slow blink**; empty → OFF.
- **Cue** — driven from `cue_indicator` (which already encodes "paused away from cue = can
  set a new cue"): blink condition → **fast blink**; at cue point → BRIGHT solid; empty →
  OFF. Blinks **faster than play**.
- **In-Out-Ex loop** (`0x03`) — rendered from the §6.1 loop state, color = deck channel:
  - no loop (state 0) → **AMBIENT** (barely lit)
  - loop-in set / arming (state 1) → **blink AMBIENT ↔ BRIGHT** (fast)
  - loop active & playing (state 2) → **BRIGHT solid**
  - on exit → back to **AMBIENT**

Implementation: a small blink subsystem holds a `slowPhase` and `fastPhase` boolean,
toggled by two timers; every LED-affecting CO change and every timer tick re-renders the
affected button(s). Cue may simply mirror `cue_indicator` if Mixxx's native rate already
reads as faster than the slow play blink; otherwise it uses the fast timer gated by the
cue condition.

## 7. No jog wheels

This controller has **no jog wheels / platters**, so there is no scratching or platter
bend, and no tempo/pitch fader (§3). Beat-matching relies on **Sync**; **tempo nudge** and
**loop length/move** are handled by the §6 modifier matrix (SHIFT / MODE on the transport
buttons), and **backspin** by SHIFT + paddle (§8.1). The buttons at `0x0b` / `0x0c` per
deck seen in the references (pitch-bend in djay, loop double/half in Rekordbox) are spare
in v1; confirm whether they physically exist (§11) before assigning anything.

## 8. FX (Effect Unit 1) and Filter

### 8.1 FX paddles + Effect Unit 1
The controller has **two paddles** (left / right), used for FX. Each is a **3-position
lever**: pushing **up latches**, pushing **down is spring-loaded** (momentary). Both
directions send the **same message** — note `0x05` on the deck's channel, `7F` on engage /
`00` on release — so the latch-vs-momentary feel is handled mechanically; the script just
mirrors the message. Like everything else, paddles are channel-multiplexed by deck:

| Paddle | Deck 1 | Deck 2 | Deck 3 | Deck 4 |
|--------|--------|--------|--------|--------|
| Left   | `98 05`|        | `9A 05`|        |
| Right  |        | `99 05`|        | `9B 05`|

- **Paddle, no shift → engage Effect Unit 1 on that deck.** The script **mirrors the
  message** (no toggle logic): `7F` → enable, `00` → disable. Pushing up holds `7F` until
  pulled back (latched FX); pushing down releases immediately (momentary FX). Mixxx:
  `[EffectRack1_EffectUnit1] group_[ChannelN]_enable` follows the message (with an effect
  loaded in the unit); the **wet/dry knob** `BF 02` drives `[EffectRack1_EffectUnit1] mix`.
- **Shift + paddle → track backspin** on that deck, via Mixxx's built-in
  `engine.spinback(deck, true)` on push / `engine.spinback(deck, false)` on release.
- The references also show FX **parameter** (`B8..BB 03`), **param −/+** (`9F 03`/`9F 04`)
  and **effect select prev/next** (`98..9B 0b`/`0c`) controls; whether these map to
  physical controls on this unit is a capture item (§11) — core FX is the paddles + knob.

### 8.2 Filter (QuickEffect) engaged LED
The filter knob is `[QuickEffectRack1_[ChannelN]] super1` (`B0..B3 1A`, already mapped).
Beside each filter button is a **blue LED that lights when the filter is off-center**
(djay drives it from the knob position). The script replicates this:
- Connect to `super1`; when it leaves a small center deadzone (`|super1 − 0.5| > ε`), send
  **blue** to the filter LED; when centered, send off.
- **LED address:** strong candidate is note **`0x24`** on the deck's pad channel
  (`94/95/96/97 24`) — the Rekordbox CSV lists `CFXOn` input/output at `9424`. The djay
  mapping file is **input-only** (no LED output definitions — djay computes feedback
  internally), so this address and the exact "blue" value are a capture item (§11).

## 9. VU meters

`engine.makeConnection` on `[ChannelN] vu_meter` → the controller's level-meter LED
address (djay `monoMeter`), value throttled/scaled. Address is a capture item.

## 10. LED / output strategy

**The app/script owns every LED — the controller does not self-manage any of them.**

- **Declarative `<output>`** for static, mode-independent LEDs (e.g. pfl) — Mixxx sends
  these from the XML based on a CO (this is still the app sending the light); value
  `0x7F`/`0x00` to the same status/note as the input.
- **Script-driven output** (`engine.makeConnection` + `midi.sendShortMsg`) for everything
  that depends on shift, pad mode, loop state, deck color/selection, split, and VU meters.
- **Selector / split LEDs** (`94 08`, `95 08`, `9F 09`) and **per-deck selection
  indicators** (`90/91/92/93 08`) are driven by the script from its tracked state.
- **Deck color** is expressed by the *channel* the LED is sent on (`0x90` blue deck 1 …
  `0x93` red deck 4). The script sends the per-deck channel actively; it is not automatic.
- **Brightness** (single-color buttons) is expressed by the MIDI *value*: `0x00` off, a
  low **AMBIENT** value (dim backlight, exact value to confirm — §11) for idle-but-lit
  buttons, and `0x7F` **BRIGHT** for active. Color (channel) and brightness (value) are
  independent axes. Blink = alternating the value (AMBIENT ↔ BRIGHT, or OFF ↔ BRIGHT) on
  a timer.
- **Pad LEDs are RGB:** their *color* is encoded in the value byte (a palette), so for
  pads the value carries color (and brightness within it) while the channel still selects
  the deck. The value→color palette is a capture item (§11); see §5.6 for per-mode colors.
- **Power-on default:** with nothing sent yet, the controller lights one selector
  (left / split / right) and the SHIFT button. So these LEDs are host-controllable;
  `init()` overrides the default to a known state.
- `init()` sets all LEDs to match Mixxx state (default: left = deck 1 active, split off,
  correct play/cue/loop/pad states); `shutdown()` clears them.

## 11. Capture-later checklist (MidiView)

A clearly-marked config block at the top of the script holds these so they're easy to
fill in after a quick capture. Everything else is already known.

1. **Selector (MODE) hold semantics** — does Ch1/Ch2 send a note-off on *release*? This
   enables the MODE modifier: hold-then-tap pad mode-select (§5.6) **and** the MODE /
   SHIFT+MODE transport combos (§6). If it latches instead, fall back to **SHIFT + pad**
   for mode-select and re-home the transport combos.
2. **Non-split pad channel behavior** — in split-off mode, do all 8 pads arrive on the
   active deck's channel, or still split across two channels? Determines slot 5–8 routing.
3. **Aux buttons `0x0b`/`0x0c`** — confirm whether these physical buttons exist and their
   labels (djay: pitch-bend ±; Rekordbox: loop double/half), to decide any v1 function.
4. **FX section extras** — paddles (`98..9B 05`) and wet/dry (`BF 02`) are confirmed;
   verify whether the param (`B8..BB 03`), param ∓ (`9F 03`/`9F 04`) and effect-select
   (`98..9B 0b`/`0c`) controls physically exist on this unit.
5. **Pad-LED acknowledgment** — confirm pads accept LED on their own `94..97 14..17`
   address (typical).
6. **VU meter** — confirm the level-meter LED address.
7. **Selector / split / shift / deck-selection LEDs** — confirm on/off values (`7F`/`00`)
   light `94 08`, `95 08`, `9F 09`, `90/91/92/93 08`, and the SHIFT LED, and that deck
   color follows the channel for output as it does for input.
8. **Brightness levels & blink rates** — find the MIDI value that produces the **AMBIENT**
   (dim) brightness vs. `0x7F` BRIGHT; confirm whether play/cue blink low end is OFF or
   AMBIENT; tune the slow (play) and fast (cue/loop-arming) blink periods to match djay.
9. **Pad RGB color palette** — determine how the pads accept color: a single value→color
   palette byte (and which values give blue/green/magenta/pink/red/white/yellow at ambient
   vs bright), or separate RGB messages. Drives the §5.6 colors and the hot-cue
   nearest-color map.
10. **Filter engaged LED** — confirm the address (candidate `94/95/96/97 24`) and the
    "blue" value for the filter-off-center indicator (§8.2), and tune the center deadzone ε.

## 12. Testing approach

- Iterate live in Mixxx (user has hardware + MidiView). Reload the mapping after each
  change; watch input via Mixxx's controller debugging and confirm LED behavior visually.
- Verify per area: analog controls (already working) → transport + LEDs → loop machine →
  pad modes (split + non-split, all 4 decks) → FX → VU.
- Confirm 4-deck routing by flipping each side and checking the correct `[ChannelN]`
  responds with the correct LED color.

## 13. Assumptions & open questions

- **Deck pairing** left = 1/3, right = 2/4 (confirmed by MIDI `90↔92`, `91↔93`).
- **Full re-channeling** confirmed for transport + analog; assumed to extend to pads/FX
  (consistent with djay's `load3 = 96 0A`).
- Pitch Cue / Saved Loops / Instant FX / Neural Mix are intentionally **stubs** in v1.
- This folder is **not a git repo**, so this spec is saved but not committed.

---

## 14. As-built addendum (2026-06-04)

The implementation deviates from §4-§6 in one fundamental way, discovered on hardware:
**the controller encodes the modifier combos into distinct MIDI notes itself** (no
script-side modifier tracking needed). The script maps notes directly:

- Mode-button tap = deck select (radio `94-97 08`); mode hold + pad = pad-mode select
  (`94-97 00-07`); mode hold + transport = combo notes (`27/28/2A/2B`, with shift
  `2C/2E/2F/30`); shift + pad = note +0x08 (`1C-23`); shift + paddle = `98-9B 00`.
- Pads: non-split = 8 notes `14-1B` on the active deck's channel; split = `14-17` per
  side. Channel always = deck.
- **The controller must be switched into its advanced ("Serato") mode on startup** via
  SysEx `F0 00 20 7F 00 F7` + `F0 00 20 7F 19 F7` (sent in `init()`), or the mode macro /
  picker / combo notes are inert after a power-cycle. The mode-picker highlight and the
  mode-layer violet LEDs are host-driven (see led-palette-capture.md).
- LED protocol, palette values, VU (CC `1F`, segments 0-7), paddle/filter LEDs: see
  `led-palette-capture.md`. A per-LED send cache keeps the MIDI wire quiet at idle.
- Bounce Loop is implemented as beatjump with direction by column (left = back/orange,
  right = forward/green, sizes 4/8/16/32) and a bright press-flash.
- Tempo nudge = shift+mode+CUE/PLAY (`2F`/`30`), ±0.1 BPM via computed rate delta.

### Final additions (2026-06-04)

- **Shift-layer transport** (controller-encoded notes, per deck channel): `0x0B`/`0x0C`
  shift+PLAY/CUE = pitch bend up/down (`rate_temp_*`), `0x29` shift+SYNC = `sync_key`
  (harmonic match), `0x40` shift+IN-OUT = auto loop toggle. Mode+CUE/PLAY (`0x27`/`0x28`)
  = musical key down/up (`pitch_down`/`pitch_up`).
- **Browse/load**: shift+Load (`94-97 10`) = library focus back/forward; shift+browse
  press (`9F 07`) = focus back; mode+browse turn (`B0-B3 06`) = beat-snapped seek
  (beatjump 4 per click).
- **Instant FX** (pad mode 3): hold pad N = QuickEffect chain preset N+2 at super 0.75
  (knob rides intensity); release restores the previous preset + super value. `init()`
  resets all decks to preset #1 — keep the filter (user: Moog Filter) FIRST in
  Preferences → Effects; pads use presets 2-9.
- **Safety**: paddle release ends FX and any active spinback regardless of which note
  the release arrives on (prevents a stuck deck when shift changes mid-gesture).
- shift+pfl (`0x12`, djay Crossfader FX) intentionally unbound. Remaining stubs:
  PitchCue (2), SavedLoops (6), NeuralMix (7).
