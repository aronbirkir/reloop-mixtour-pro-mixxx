# Reloop Mixtour Pro (4-deck) Mixxx Mapping — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Finish the Reloop Mixtour Pro Mixxx mapping with full 4-deck support, LED feedback, a SHIFT + MODE modifier system, and pad modes — built on `ReloopMixTourPro.midi.xml` plus a new `ReloopMixTourPro-script.js`.

**Architecture:** Analog controls stay as declarative `<control>` bindings in the XML (extended to 4 decks); all stateful behavior lives in one script object (`ReloopMixTourPro`). The controller multiplexes everything by MIDI channel = deck (`0x90–0x93` buttons, `0xB0–0xB3` CCs, `0x94–0x97` pads), and the script owns all LED output. Design spec: `docs/superpowers/specs/2026-06-02-reloop-mixtour-pro-mapping-design.md`.

**Tech Stack:** Mixxx MIDI controller preset (XML) + Mixxx Controller Scripting (JavaScript, QJSEngine with `engine` / `midi` / `script` globals).

**Testing approach (live-only, chosen by user):** No automated harness. Each task ends by reloading the mapping in Mixxx and observing the controller + Mixxx UI. Throughout, keep the **script log** open to catch errors — launch `mixxx --controllerDebug --logLevel debug` (or watch `~/Library/Containers/org.mixxx.mixxx/Data/Library/Application Support/Mixxx/mixxx.log`, or the standard `mixxx.log`). **Reload procedure:** Preferences → Controllers → select "Reloop Mixtour Pro" → toggle the preset/Apply (or restart Mixxx). Enable **4 decks** (Preferences → Decks → Decks = 4, or a 4-deck skin).

**Capture-as-you-go:** §11 of the spec lists hardware values still to confirm (selector release, pad color palette, ambient brightness, blink rates, filter LED, etc.). They live in a single `MTP` config block at the top of the script; tasks below set sensible starting guesses and you tune them live.

---

## File Structure

- **`ReloopMixTourPro.midi.xml`** — controller preset. Sections:
  - `<scriptfiles>` referencing the JS file.
  - `<controls>`: analog `<control>` bindings for 4 decks (faders/EQ/filter/gain/headphone/crossfader/browse), plus `<script-binding>` entries routing transport, pads, paddles, selectors, split, shift, deck-switch, and load to script handlers.
  - `<outputs>`: declarative LED for `pfl` only (everything else is script-driven).
- **`ReloopMixTourPro-script.js`** — one `ReloopMixTourPro` object, organized top-to-bottom:
  1. `MTP` config block (addresses, palette, brightness, blink rates).
  2. State (shift, selectors, split, per-side virtual deck, per-deck pad mode + loop state, blink phases).
  3. `init` / `shutdown`.
  4. Modifier inputs: `shift`, selectors/split, deck-switch notifications.
  5. Transport handlers + loop state machine.
  6. Pad handlers (performance + mode-picker) + routing helpers.
  7. Pad-mode action dispatch (hotcue / beatloop / beatjump / sampler).
  8. FX paddles + backspin; filter LED.
  9. LED rendering + helpers (sendLED, color quantize, blink).

---

## Task 0: Project setup — version control + script skeleton

**Files:**
- Create: `.gitignore`
- Create: `ReloopMixTourPro-script.js`
- Modify: `ReloopMixTourPro.midi.xml` (add `<scriptfiles>`, set controller id/name)

- [ ] **Step 1: Initialize git so we get checkpoints**

```bash
cd "/Users/aron/dev/experiment/mixxx/reloop_mixtour_pro"
git init
printf '%s\n' '*.log' '.DS_Store' > .gitignore
git add -A && git commit -m "chore: baseline (existing mapping + reference files + spec)"
```

- [ ] **Step 2: Create the script skeleton**

Create `ReloopMixTourPro-script.js`:

```javascript
// ReloopMixTourPro-script.js
// Mixxx controller mapping script for the Reloop Mixtour Pro (4-deck).
// See docs/superpowers/specs/2026-06-02-reloop-mixtour-pro-mapping-design.md

var ReloopMixTourPro = {};

// ---- Config (tune these live; see spec §11) -------------------------------
ReloopMixTourPro.MTP = {
    decks: 4,
};

// ---- Lifecycle ------------------------------------------------------------
ReloopMixTourPro.init = function(_id, _debug) {
    engine.log("ReloopMixTourPro: init");
};

ReloopMixTourPro.shutdown = function() {
    engine.log("ReloopMixTourPro: shutdown");
};
```

- [ ] **Step 3: Wire the script into the XML**

In `ReloopMixTourPro.midi.xml`, set the `<info>`/`<controller>` and add `<scriptfiles>` (replace the empty `<scriptfiles/>`):

```xml
    <controller id="Reloop Mixtour Pro">
        <scriptfiles>
            <file functionprefix="ReloopMixTourPro" filename="ReloopMixTourPro-script.js"/>
        </scriptfiles>
```

- [ ] **Step 4: Live check — script loads with no errors**

Run: `mixxx --controllerDebug --logLevel debug`
Reload the preset (Preferences → Controllers → Reloop Mixtour Pro → Apply).
Expected: log shows `ReloopMixTourPro: init` and **no** script errors. Existing analog controls (faders/EQ) still work on decks 1–2.

- [ ] **Step 5: Commit**

```bash
git add ReloopMixTourPro-script.js ReloopMixTourPro.midi.xml .gitignore
git commit -m "feat: add script skeleton wired into the preset"
```

---

## Task 1: Config block + LED send helpers + 4-deck analog controls

**Files:**
- Modify: `ReloopMixTourPro-script.js` (expand `MTP`, add helpers)
- Modify: `ReloopMixTourPro.midi.xml` (add deck 3 & 4 analog `<control>` entries)

- [ ] **Step 1: Fill in the config block + LED helpers**

Replace the `MTP` block and add helpers + a deck-channel map in `ReloopMixTourPro-script.js`:

```javascript
ReloopMixTourPro.MTP = {
    decks: 4,
    // Status base per message kind; channel offset = deckIndex (0..3)
    NOTE_BASE: 0x90,   // 0x90..0x93 buttons
    CC_BASE:   0xB0,   // 0xB0..0xB3 CCs
    PAD_BASE:  0x94,   // 0x94..0x97 pads/load
    GLOBAL_NOTE: 0x9F, // shift / split / library / fx -/+
    GLOBAL_CC:   0xBF, // crossfader / master / browse / fx wet-dry

    // Brightness values (tune live — spec §11 item 8)
    OFF: 0x00, AMBIENT: 0x05, BRIGHT: 0x7F,

    // Blink periods in ms (tune live)
    BLINK_SLOW: 700, BLINK_FAST: 300,

    // Notes
    N_PLAY: 0x00, N_CUE: 0x01, N_SYNC: 0x02, N_INOUT: 0x03,
    N_PFL: 0x1B, N_SELECTOR: 0x08, N_LOAD: 0x0A,
    N_PAD: [0x14, 0x15, 0x16, 0x17], // 4 rows; column = pad channel
    N_FILTER_LED: 0x24,
    N_SHIFT: 0x00, N_SPLIT: 0x09, // on GLOBAL_NOTE (0x9F)
    N_PADDLE: 0x05,               // on per-deck note channel base 0x98 (see FX task)
};

// deckIndex (0-based) -> Mixxx group
ReloopMixTourPro.groupForDeck = function(deckIndex) {
    return "[Channel" + (deckIndex + 1) + "]";
};

// Send a 3-byte MIDI message to a deck's button/pad LED.
// statusBase: MTP.NOTE_BASE or MTP.PAD_BASE; value: brightness or color byte.
ReloopMixTourPro.sendLED = function(statusBase, deckIndex, note, value) {
    midi.sendShortMsg(statusBase + deckIndex, note, value);
};
```

- [ ] **Step 2: Add deck 3 & 4 analog controls to the XML**

For each existing deck-1/deck-2 analog `<control>` (volume `B0/B1 1C`, EQ `…17/18/19`, filter `super1 …1A`, pregain `…16`), add the deck-3 (`0xB2`) and deck-4 (`0xB3`) equivalents pointing at `[Channel3]` / `[Channel4]`. Example for volume:

```xml
            <control>
                <group>[Channel3]</group>
                <key>volume</key>
                <status>0xB2</status>
                <midino>0x1C</midino>
                <options><normal/></options>
            </control>
            <control>
                <group>[Channel4]</group>
                <key>volume</key>
                <status>0xB3</status>
                <midino>0x1C</midino>
                <options><normal/></options>
            </control>
```

Repeat for `pregain` (`0x16`), EQ `[EqualizerRack1_[Channel3/4]_Effect1] parameter3/2/1` (`0x17/0x18/0x19`), and filter `[QuickEffectRack1_[Channel3/4]] super1` (`0x1A`).

- [ ] **Step 3: Live check — analog on all 4 decks**

Reload. Flip the left side to deck 3 and right to deck 4. Move volume/EQ/filter/gain knobs.
Expected: deck 3 controls move `[Channel3]`, deck 4 move `[Channel4]`, in the Mixxx UI. No log errors.

- [ ] **Step 4: Commit**

```bash
git add ReloopMixTourPro-script.js ReloopMixTourPro.midi.xml
git commit -m "feat: config block, LED helper, 4-deck analog controls"
```

---

## Task 2: Modifier state + input handlers (SHIFT, selectors, SPLIT, deck-switch)

**Files:**
- Modify: `ReloopMixTourPro-script.js` (state + handlers)
- Modify: `ReloopMixTourPro.midi.xml` (script-bindings for `9F 00`, `94/95 08`, `9F 09`, `90-93 08`)

- [ ] **Step 1: Add state block (after the `MTP` block)**

```javascript
ReloopMixTourPro.state = {
    shift: false,
    selectorHeld: { left: false, right: false }, // MODE modifier (spec §11 item 1)
    split: false,
    activeSide: "left",                 // which side 8 pads target when split off
    sideDeck: { left: 0, right: 1 },    // 0-based deck per side: left∈{0,2}, right∈{1,3}
    padMode: [0, 0, 0, 0],              // per deck: index into PAD_MODES
    loopState: [0, 0, 0, 0],            // per deck: 0 none, 1 armed, 2 looping
    blinkSlow: false,
    blinkFast: false,
};
```

- [ ] **Step 2: Add the input handlers**

```javascript
// SHIFT — 9F 00
ReloopMixTourPro.shiftButton = function(_ch, _ctrl, value) {
    ReloopMixTourPro.state.shift = (value > 0);
};

// Side selector held — 94 08 (left) / 95 08 (right). Press selects that side
// (split off) and marks it held (MODE modifier). Release clears held.
ReloopMixTourPro.selector = function(_ch, _ctrl, value, status) {
    var side = (status === ReloopMixTourPro.MTP.PAD_BASE) ? "left" : "right";
    var held = (value > 0);
    ReloopMixTourPro.state.selectorHeld[side] = held;
    if (held) {
        ReloopMixTourPro.state.split = false;
        ReloopMixTourPro.state.activeSide = side;
    }
    engine.log("selector " + side + " held=" + held);
};

// SPLIT — 9F 09
ReloopMixTourPro.splitButton = function(_ch, _ctrl, value) {
    if (value > 0) {
        ReloopMixTourPro.state.split = true;
        engine.log("split on");
    }
};

// Deck-switch notification — 90/91/92/93 08 (value 7F = this deck now active on its side)
ReloopMixTourPro.deckSwitch = function(_ch, _ctrl, value, status) {
    if (value === 0) { return; }
    var deckIndex = status - ReloopMixTourPro.MTP.NOTE_BASE; // 0..3
    var side = (deckIndex % 2 === 0) ? "left" : "right";     // 0,2 left ; 1,3 right
    ReloopMixTourPro.state.sideDeck[side] = deckIndex;
    engine.log("deckSwitch side=" + side + " deck=" + (deckIndex + 1));
};
```

- [ ] **Step 3: Add the XML script-bindings**

Add these `<control>` entries (group is irrelevant for the globals; use `[Master]`):

```xml
            <control>
                <group>[Master]</group><key>ReloopMixTourPro.shiftButton</key>
                <status>0x9F</status><midino>0x00</midino>
                <options><script-binding/></options>
            </control>
            <control>
                <group>[Master]</group><key>ReloopMixTourPro.splitButton</key>
                <status>0x9F</status><midino>0x09</midino>
                <options><script-binding/></options>
            </control>
            <control>
                <group>[Master]</group><key>ReloopMixTourPro.selector</key>
                <status>0x94</status><midino>0x08</midino>
                <options><script-binding/></options>
            </control>
            <control>
                <group>[Master]</group><key>ReloopMixTourPro.selector</key>
                <status>0x95</status><midino>0x08</midino>
                <options><script-binding/></options>
            </control>
```

And four deck-switch bindings, `0x90`–`0x93` note `0x08`, all pointing at `ReloopMixTourPro.deckSwitch` (group `[Master]`):

```xml
            <control>
                <group>[Master]</group><key>ReloopMixTourPro.deckSwitch</key>
                <status>0x90</status><midino>0x08</midino>
                <options><script-binding/></options>
            </control>
            <!-- repeat with status 0x91, 0x92, 0x93 -->
```

- [ ] **Step 4: Live check — modifier state logs correctly**

Reload with `--controllerDebug`. Watch the log while you:
- Hold SHIFT → (no log yet, but) press a selector while holding to confirm later. Add a temporary `engine.log("shift=" + ...)` inside `shiftButton` if you want to see it.
- Tap left/right selectors → log `selector left/right held=true` then `held=false` **on release** (this confirms spec §11 item 1 — selectors DO send note-off). If you see no `held=false`, the selector latches: note this, we use the SHIFT+pad fallback later.
- Tap SPLIT → `split on`.
- Hold left selector + tap LOAD → `deckSwitch side=left deck=3`; tap again → `deck=1`.

Expected: logs match actions. **Record whether selectors send release** — it gates the MODE features.

- [ ] **Step 5: Commit**

```bash
git add ReloopMixTourPro-script.js ReloopMixTourPro.midi.xml
git commit -m "feat: modifier state + SHIFT/selector/split/deck-switch handlers"
```

---

## Task 3: Transport plain actions + play/cue/sync LEDs with blink

**Files:**
- Modify: `ReloopMixTourPro-script.js` (transport handlers, blink timers, LED render)
- Modify: `ReloopMixTourPro.midi.xml` (script-bindings for `90-93` notes `00/01/02`; declarative output for `pfl`)

- [ ] **Step 1: Add the blink timer subsystem (in `init`, plus a render fn)**

```javascript
ReloopMixTourPro.blinkTimers = [];

ReloopMixTourPro.startBlink = function() {
    var self = ReloopMixTourPro;
    self.blinkTimers.push(engine.beginTimer(self.MTP.BLINK_SLOW, function() {
        self.state.blinkSlow = !self.state.blinkSlow;
        self.renderAllTransportLEDs();
    }));
    self.blinkTimers.push(engine.beginTimer(self.MTP.BLINK_FAST, function() {
        self.state.blinkFast = !self.state.blinkFast;
        self.renderAllTransportLEDs();
    }));
};

ReloopMixTourPro.renderAllTransportLEDs = function() {
    for (var d = 0; d < ReloopMixTourPro.MTP.decks; d++) {
        ReloopMixTourPro.renderPlayLED(d);
        ReloopMixTourPro.renderLoopLED(d); // defined in Task 4; safe no-op until then
    }
};
```

Add a guard so `renderLoopLED` existing-or-not won't crash before Task 4 — define a stub now:

```javascript
ReloopMixTourPro.renderLoopLED = function(_d) {}; // replaced in Task 4
```

- [ ] **Step 2: Transport handlers (plain behavior only for now)**

```javascript
ReloopMixTourPro.playButton = function(_ch, _ctrl, value, _status, group) {
    if (value === 0) { return; }
    script.toggleControl(group, "play");
};

ReloopMixTourPro.cueButton = function(_ch, _ctrl, value, _status, group) {
    engine.setValue(group, "cue_default", value > 0 ? 1 : 0);
};

ReloopMixTourPro.syncButton = function(_ch, _ctrl, value, _status, group) {
    if (value === 0) { return; }
    script.toggleControl(group, "sync_enabled");
};
```

- [ ] **Step 3: Play LED render + cue/sync LED connections**

```javascript
ReloopMixTourPro.renderPlayLED = function(deckIndex) {
    var g = ReloopMixTourPro.groupForDeck(deckIndex);
    var m = ReloopMixTourPro.MTP, st = ReloopMixTourPro.state;
    var loaded = engine.getValue(g, "track_loaded") > 0;
    var playing = engine.getValue(g, "play") > 0;
    var v = m.OFF;
    if (playing) { v = m.BRIGHT; }
    else if (loaded) { v = st.blinkSlow ? m.BRIGHT : m.OFF; }
    ReloopMixTourPro.sendLED(m.NOTE_BASE, deckIndex, m.N_PLAY, v);
};

ReloopMixTourPro.connectTransportLEDs = function() {
    var self = ReloopMixTourPro, m = self.MTP;
    for (var d = 0; d < m.decks; d++) {
        (function(deckIndex) {
            var g = self.groupForDeck(deckIndex);
            engine.makeConnection(g, "play", function() { self.renderPlayLED(deckIndex); });
            engine.makeConnection(g, "track_loaded", function() { self.renderPlayLED(deckIndex); });
            // Cue: mirror cue_indicator (blinks faster than play, per spec §6.2)
            engine.makeConnection(g, "cue_indicator", function(v) {
                self.sendLED(m.NOTE_BASE, deckIndex, m.N_CUE, v > 0 ? m.BRIGHT : m.OFF);
            });
            engine.makeConnection(g, "sync_enabled", function(v) {
                self.sendLED(m.NOTE_BASE, deckIndex, m.N_SYNC, v > 0 ? m.BRIGHT : m.OFF);
            });
        })(d);
    }
};
```

- [ ] **Step 4: Hook into `init` / `shutdown`**

```javascript
ReloopMixTourPro.init = function(_id, _debug) {
    engine.log("ReloopMixTourPro: init");
    ReloopMixTourPro.connectTransportLEDs();
    ReloopMixTourPro.startBlink();
    ReloopMixTourPro.renderAllTransportLEDs();
};

ReloopMixTourPro.shutdown = function() {
    var m = ReloopMixTourPro.MTP;
    for (var i = 0; i < ReloopMixTourPro.blinkTimers.length; i++) {
        engine.stopTimer(ReloopMixTourPro.blinkTimers[i]);
    }
    for (var d = 0; d < m.decks; d++) {
        ReloopMixTourPro.sendLED(m.NOTE_BASE, d, m.N_PLAY, m.OFF);
        ReloopMixTourPro.sendLED(m.NOTE_BASE, d, m.N_CUE, m.OFF);
        ReloopMixTourPro.sendLED(m.NOTE_BASE, d, m.N_SYNC, m.OFF);
    }
};
```

- [ ] **Step 5: XML — script-bind play/cue/sync (replace the existing direct `play`/`cue_default`/`sync_enabled` controls for decks 1–2 and add 3–4) + declarative pfl output**

For each deck `0x90`–`0x93`, bind notes `0x00`→`playButton`, `0x01`→`cueButton`, `0x02`→`syncButton`. Example deck 1:

```xml
            <control>
                <group>[Channel1]</group><key>ReloopMixTourPro.playButton</key>
                <status>0x90</status><midino>0x00</midino>
                <options><script-binding/></options>
            </control>
```

Add pfl as direct control + declarative output for all 4 decks (no script needed):

```xml
            <control>
                <group>[Channel1]</group><key>pfl</key>
                <status>0x90</status><midino>0x1B</midino>
                <options><normal/></options>
            </control>
```

In `<outputs>`:

```xml
            <output>
                <group>[Channel1]</group><key>pfl</key>
                <status>0x90</status><midino>0x1B</midino>
                <on>0x7F</on><off>0x00</off><minimum>0.5</minimum>
            </output>
```

- [ ] **Step 6: Live check — transport + blink**

Reload. On each of the 4 decks (flip sides to reach 3/4):
- Load a track, press PLAY → plays, play LED bright solid. Pause → play LED **slow blink**. Eject/empty → off.
- Move the playhead off the cue while paused → CUE LED **fast blink**; sit on the cue → solid. (Cue blinks faster than play.)
- Press SYNC → toggles, SYNC LED tracks.
- Press PFL (headphone) → pfl toggles + its LED.

Expected: all four decks behave; blink rates differ (cue faster). Tune `AMBIENT`/`BRIGHT`/`BLINK_*` if needed.

- [ ] **Step 7: Commit**

```bash
git add ReloopMixTourPro-script.js ReloopMixTourPro.midi.xml
git commit -m "feat: transport play/cue/sync + blink LEDs (4 decks)"
```

---

## Task 4: In-Out-Ex 3-state loop machine + loop LED

**Files:**
- Modify: `ReloopMixTourPro-script.js` (loop machine, loop LED, connection)
- Modify: `ReloopMixTourPro.midi.xml` (script-binding for notes `0x03` on `90-93`)

- [ ] **Step 1: Add deck-from-group helper + the loop state machine**

```javascript
ReloopMixTourPro.deckIndexForGroup = function(group) {
    return parseInt(group.charAt(8), 10) - 1; // "[ChannelN]" -> N-1
};

ReloopMixTourPro.loopStep = function(deckIndex, group) {
    var st = ReloopMixTourPro.state;
    var s = st.loopState[deckIndex];
    if (s === 0) {
        engine.setValue(group, "loop_in", 1);
        st.loopState[deckIndex] = 1;
    } else if (s === 1) {
        engine.setValue(group, "loop_out", 1);
        st.loopState[deckIndex] = 2;
    } else {
        if (engine.getValue(group, "loop_enabled") > 0) {
            engine.setValue(group, "reloop_toggle", 1);
        }
        st.loopState[deckIndex] = 0;
    }
    ReloopMixTourPro.renderLoopLED(deckIndex);
};
```

- [ ] **Step 2: Replace the `renderLoopLED` stub with the real renderer**

```javascript
ReloopMixTourPro.renderLoopLED = function(deckIndex) {
    var m = ReloopMixTourPro.MTP, st = ReloopMixTourPro.state;
    var s = st.loopState[deckIndex];
    var v;
    if (s === 0) { v = m.AMBIENT; }
    else if (s === 1) { v = st.blinkFast ? m.BRIGHT : m.AMBIENT; }
    else { v = m.BRIGHT; }
    ReloopMixTourPro.sendLED(m.NOTE_BASE, deckIndex, m.N_INOUT, v); // color = deck channel
};
```

- [ ] **Step 3: Keep our state in sync with Mixxx's loop_enabled + add the handler**

```javascript
ReloopMixTourPro.connectLoopLEDs = function() {
    var self = ReloopMixTourPro, m = self.MTP;
    for (var d = 0; d < m.decks; d++) {
        (function(deckIndex) {
            engine.makeConnection(self.groupForDeck(deckIndex), "loop_enabled", function(v) {
                var st = self.state;
                if (v > 0) { st.loopState[deckIndex] = 2; }
                else if (st.loopState[deckIndex] === 2) { st.loopState[deckIndex] = 0; }
                self.renderLoopLED(deckIndex);
            });
        })(d);
    }
};

ReloopMixTourPro.inOutButton = function(_ch, _ctrl, value, _status, group) {
    if (value === 0) { return; }
    ReloopMixTourPro.loopStep(ReloopMixTourPro.deckIndexForGroup(group), group);
};
```

- [ ] **Step 4: Call `connectLoopLEDs()` in `init` (after `connectTransportLEDs`) and clear loop LEDs in `shutdown`**

In `init`, add `ReloopMixTourPro.connectLoopLEDs();`. In `shutdown`'s deck loop, add:
`ReloopMixTourPro.sendLED(m.NOTE_BASE, d, m.N_INOUT, m.OFF);`

- [ ] **Step 5: XML — script-bind In-Out-Ex (note `0x03`) for all 4 decks**

```xml
            <control>
                <group>[Channel1]</group><key>ReloopMixTourPro.inOutButton</key>
                <status>0x90</status><midino>0x03</midino>
                <options><script-binding/></options>
            </control>
            <!-- repeat 0x91/0x92/0x93 with [Channel2/3/4] -->
```

- [ ] **Step 6: Live check — loop button cycle + LED levels**

Reload. On a playing deck:
- LED sits **ambient** (no loop). Press In-Out-Ex → **arming, blinks ambient↔bright**; press again → loop closes, **bright solid**, audio loops; press again → loop exits, back to **ambient**.
- Set a loop via pads later — confirm pressing In-Out-Ex again starts a fresh loop.

Expected: 3-press cycle works; LED matches; loop LED is blue on decks 1/2, red on 3/4.

- [ ] **Step 7: Commit**

```bash
git add ReloopMixTourPro-script.js ReloopMixTourPro.midi.xml
git commit -m "feat: In-Out-Ex 3-state loop machine + loop LED"
```

---

## Task 5: Transport modifier matrix (SHIFT / MODE / SHIFT+MODE)

**Files:**
- Modify: `ReloopMixTourPro-script.js` (extend the 4 transport handlers; add `modeHeld`, `tempoNudge`)

- [ ] **Step 1: Add the MODE helper + tempo-nudge helper**

```javascript
ReloopMixTourPro.modeHeld = function() {
    var s = ReloopMixTourPro.state.selectorHeld;
    return s.left || s.right;
};

// Nudge effective tempo by deltaBpm (e.g. +0.1) by adjusting `rate`.
ReloopMixTourPro.tempoNudge = function(group, deltaBpm) {
    var fileBpm = engine.getValue(group, "file_bpm");
    var rateRange = engine.getValue(group, "rateRange");
    var rateDir = engine.getValue(group, "rate_dir");
    if (fileBpm <= 0 || rateRange <= 0 || rateDir === 0) { return; }
    var dRate = deltaBpm / (fileBpm * rateRange * rateDir);
    engine.setValue(group, "rate", engine.getValue(group, "rate") + dRate);
};
```

- [ ] **Step 2: Replace `playButton` and `cueButton` with modifier-aware versions**

```javascript
ReloopMixTourPro.playButton = function(_ch, _ctrl, value, _status, group) {
    if (value === 0) { return; }
    var st = ReloopMixTourPro.state;
    if (st.shift && ReloopMixTourPro.modeHeld()) {
        ReloopMixTourPro.tempoNudge(group, 0.1);          // SHIFT+MODE: tempo +0.1
    } else if (st.shift) {
        engine.setValue(group, "play_stutter", 1);        // suggested default
    } else {
        script.toggleControl(group, "play");
    }
};

ReloopMixTourPro.cueButton = function(_ch, _ctrl, value, _status, group) {
    var st = ReloopMixTourPro.state;
    if (st.shift && ReloopMixTourPro.modeHeld()) {
        if (value > 0) { ReloopMixTourPro.tempoNudge(group, -0.1); } // SHIFT+MODE: tempo -0.1
        return;
    }
    if (st.shift) {
        if (value > 0) { engine.setValue(group, "cue_gotoandstop", 1); } // suggested default
        return;
    }
    engine.setValue(group, "cue_default", value > 0 ? 1 : 0);
};
```

- [ ] **Step 3: Replace `syncButton` and `inOutButton` with modifier-aware versions**

```javascript
ReloopMixTourPro.syncButton = function(_ch, _ctrl, value, _status, group) {
    if (value === 0) { return; }
    var st = ReloopMixTourPro.state;
    if (st.shift && ReloopMixTourPro.modeHeld()) {
        engine.setValue(group, "loop_out", 1);            // SHIFT+MODE: loop out / move
    } else if (ReloopMixTourPro.modeHeld()) {
        engine.setValue(group, "loop_double", 1);         // MODE: loop double (lengthen)
    } else if (st.shift) {
        script.toggleControl(group, "sync_master");       // suggested default
    } else {
        script.toggleControl(group, "sync_enabled");
    }
};

ReloopMixTourPro.inOutButton = function(_ch, _ctrl, value, _status, group) {
    if (value === 0) { return; }
    var st = ReloopMixTourPro.state;
    var deckIndex = ReloopMixTourPro.deckIndexForGroup(group);
    if (st.shift && ReloopMixTourPro.modeHeld()) {
        engine.setValue(group, "loop_in", 1);             // SHIFT+MODE: loop in / move
    } else if (ReloopMixTourPro.modeHeld()) {
        engine.setValue(group, "loop_halve", 1);          // MODE: loop halve
    } else if (st.shift) {
        if (engine.getValue(group, "loop_enabled") > 0) { engine.setValue(group, "reloop_toggle", 1); }
        st.loopState[deckIndex] = 0;
        ReloopMixTourPro.renderLoopLED(deckIndex);        // suggested default: clear loop
    } else {
        ReloopMixTourPro.loopStep(deckIndex, group);      // plain 3-state machine
    }
};
```

- [ ] **Step 4: Live check — modifier matrix (depends on selector reporting release, Task 2)**

Reload. On a deck with a loop set:
- Hold a selector + In-Out-Ex → loop **halves**; hold selector + SYNC → loop **doubles**.
- SHIFT + selector + In-Out-Ex → **loop in**; SHIFT + selector + SYNC → **loop out**.
- SHIFT + selector + PLAY → BPM **+0.1**; SHIFT + selector + CUE → BPM **−0.1** (watch the deck BPM readout).
- Plain PLAY/CUE/SYNC/In-Out-Ex still behave as in Tasks 3–4.

Expected: matrix matches spec §6. If selectors latch (no release — Task 2 finding), `modeHeld()` stays true after a tap; in that case treat the selector tap as a toggle and note it for adjustment.

- [ ] **Step 5: Commit**

```bash
git add ReloopMixTourPro-script.js
git commit -m "feat: transport SHIFT/MODE modifier matrix (loop adjust + tempo nudge)"
```

---

## Task 6: Pad routing + Hot Cue mode + pad LEDs

**Files:**
- Modify: `ReloopMixTourPro-script.js` (color table, routing, hotcue, pad LEDs)
- Modify: `ReloopMixTourPro.midi.xml` (script-bind 8 pads/deck; add deck 3/4 load direct bindings)

- [ ] **Step 1: Add a tunable color table to the `MTP` config block**

The pad value byte encodes color (and brightness within it). Tune these after capturing the palette (spec §11 #9). Start with these guesses:

```javascript
// add inside MTP = { ... }
    COLORS: {
        off:     { ambient: 0x00, bright: 0x00 },
        white:   { ambient: 0x01, bright: 0x02 },
        red:     { ambient: 0x03, bright: 0x04 },
        green:   { ambient: 0x05, bright: 0x06 },
        blue:    { ambient: 0x07, bright: 0x08 },
        pink:    { ambient: 0x09, bright: 0x0A },
        magenta: { ambient: 0x0B, bright: 0x0C },
        yellow:  { ambient: 0x0D, bright: 0x0E },
    },
```

- [ ] **Step 2: Add the pad-color send + nearest-color helpers**

```javascript
ReloopMixTourPro.sendPad = function(physDeck, note, colorName, bright) {
    var c = ReloopMixTourPro.MTP.COLORS[colorName] || ReloopMixTourPro.MTP.COLORS.off;
    midi.sendShortMsg(ReloopMixTourPro.MTP.PAD_BASE + physDeck, note, bright ? c.bright : c.ambient);
};

// Map a Mixxx hotcue color (0xRRGGBB int) to the nearest named controller color.
ReloopMixTourPro.PALETTE_RGB = {
    red: 0xFF0000, green: 0x00FF00, blue: 0x0000FF, pink: 0xFF59B3,
    magenta: 0xFF00FF, yellow: 0xFFFF00, white: 0xFFFFFF,
};
ReloopMixTourPro.nearestColorName = function(rgb) {
    var r = (rgb >> 16) & 0xFF, g = (rgb >> 8) & 0xFF, b = rgb & 0xFF;
    var best = "white", bestD = Infinity;
    var P = ReloopMixTourPro.PALETTE_RGB;
    for (var name in P) {
        var pr = (P[name] >> 16) & 0xFF, pg = (P[name] >> 8) & 0xFF, pb = P[name] & 0xFF;
        var d = (r - pr) * (r - pr) + (g - pg) * (g - pg) + (b - pb) * (b - pb);
        if (d < bestD) { bestD = d; best = name; }
    }
    return best;
};
```

- [ ] **Step 3: Add pad routing (input) + the matching physical-pad geometry**

```javascript
// Given an incoming pad message, resolve the target deck + slot (0..7).
ReloopMixTourPro.routePad = function(status, note) {
    var m = ReloopMixTourPro.MTP, st = ReloopMixTourPro.state;
    var padDeck = status - m.PAD_BASE;       // column's own deck (0..3)
    var row = note - m.N_PAD[0];             // 0..3
    var leftColumn = (padDeck % 2 === 0);
    if (st.split) {
        return { deckIndex: padDeck, slot: row, leftColumn: leftColumn, row: row };
    }
    return {
        deckIndex: st.sideDeck[st.activeSide],
        slot: leftColumn ? row : row + 4,
        leftColumn: leftColumn, row: row,
    };
};
```

- [ ] **Step 4: Add the pad input handler + Hot Cue action + dispatcher**

```javascript
ReloopMixTourPro.padPress = function(_ch, control, value, status, _group) {
    var st = ReloopMixTourPro.state;
    if (st.selectorHeld.left || st.selectorHeld.right) {
        if (value > 0) { ReloopMixTourPro.pickMode(status, control); } // defined Task 7
        return;
    }
    var r = ReloopMixTourPro.routePad(status, control);
    ReloopMixTourPro.padAction(ReloopMixTourPro.groupForDeck(r.deckIndex), r.deckIndex, r.slot, value);
};

ReloopMixTourPro.padAction = function(group, deckIndex, slot, value) {
    switch (ReloopMixTourPro.state.padMode[deckIndex]) {
        case 0: ReloopMixTourPro.padHotcue(group, slot, value); break;
        // 1/2/3 added in Task 7; 4-7 stubbed in Task 10
        default: break;
    }
};

ReloopMixTourPro.padHotcue = function(group, slot, value) {
    var n = slot + 1; // hotcue 1..8
    if (value > 0) {
        engine.setValue(group, ReloopMixTourPro.state.shift ? ("hotcue_" + n + "_clear")
                                                            : ("hotcue_" + n + "_activate"), 1);
    } else {
        engine.setValue(group, "hotcue_" + n + "_activate", 0);
    }
};
```

- [ ] **Step 5: Add pad LED rendering (physical-pad geometry mirrors routing)**

```javascript
// Color for one logical slot of a deck under its current mode.
ReloopMixTourPro.padColorForSlot = function(deckIndex, slot) {
    var g = ReloopMixTourPro.groupForDeck(deckIndex);
    switch (ReloopMixTourPro.state.padMode[deckIndex]) {
        case 0: { // Hot Cue
            var n = slot + 1;
            if (engine.getValue(g, "hotcue_" + n + "_status") > 0) {
                var rgb = engine.getValue(g, "hotcue_" + n + "_color");
                return { color: ReloopMixTourPro.nearestColorName(rgb), bright: true };
            }
            return { color: "off", bright: false };
        }
        default: return { color: "off", bright: false };
    }
};

// Render all 8 physical pads from current side decks + split/active state.
ReloopMixTourPro.renderAllPads = function() {
    var m = ReloopMixTourPro.MTP, st = ReloopMixTourPro.state;
    var cols = [
        { physDeck: st.sideDeck.left,  leftColumn: true },
        { physDeck: st.sideDeck.right, leftColumn: false },
    ];
    for (var ci = 0; ci < cols.length; ci++) {
        for (var row = 0; row < 4; row++) {
            var note = m.N_PAD[row];
            var targetDeck, slot;
            if (st.split) { targetDeck = cols[ci].physDeck; slot = row; }
            else { targetDeck = st.sideDeck[st.activeSide]; slot = cols[ci].leftColumn ? row : row + 4; }
            var c = ReloopMixTourPro.padColorForSlot(targetDeck, slot);
            ReloopMixTourPro.sendPad(cols[ci].physDeck, note, c.color, c.bright);
        }
    }
};
```

- [ ] **Step 6: Connect hotcue state to re-render pads; render pads in `init`; clear in `shutdown`**

```javascript
ReloopMixTourPro.connectPadLEDs = function() {
    var self = ReloopMixTourPro;
    for (var d = 0; d < self.MTP.decks; d++) {
        var g = self.groupForDeck(d);
        for (var n = 1; n <= 8; n++) {
            engine.makeConnection(g, "hotcue_" + n + "_status", function() { self.renderAllPads(); });
        }
    }
};
```

In `init` add `ReloopMixTourPro.connectPadLEDs();` and `ReloopMixTourPro.renderAllPads();`. In `shutdown`, after the transport clears, add a loop clearing all pads:

```javascript
    for (var pd = 0; pd < m.decks; pd++) {
        for (var rr = 0; rr < 4; rr++) { ReloopMixTourPro.sendPad(pd, m.N_PAD[rr], "off", false); }
    }
```

- [ ] **Step 7: XML — script-bind the 8 pads per deck + add deck 3/4 load**

For each deck channel `0x94`–`0x97`, bind notes `0x14`,`0x15`,`0x16`,`0x17` → `ReloopMixTourPro.padPress`. Example (deck 1 column, row 1):

```xml
            <control>
                <group>[Channel1]</group><key>ReloopMixTourPro.padPress</key>
                <status>0x94</status><midino>0x14</midino>
                <options><script-binding/></options>
            </control>
```

(32 entries total: 4 channels × 4 notes; group can be the column's deck — it's unused by the handler, which routes via status.) Add deck 3/4 load as direct bindings (decks 1/2 already exist):

```xml
            <control>
                <group>[Channel3]</group><key>LoadSelectedTrack</key>
                <status>0x96</status><midino>0x0A</midino>
                <options><normal/></options>
            </control>
            <!-- and [Channel4] on 0x97 0x0A -->
```

- [ ] **Step 8: Live check — hot cues + colors across decks/split**

Reload. With pads in Hot Cue mode (default):
- Press empty pads → hot cues set, pad lights in the cue's color (bright). Press again → jumps to cue.
- SHIFT + pad → clears the cue, pad goes dark.
- Toggle SPLIT on → left 4 pads = the left-side deck's cues 1–4, right 4 = right-side deck's cues 1–4. Toggle off → 8 pads on the active deck.
- Flip a side to deck 3/4 → pads follow.

Expected: routing + colors behave. **Tune `COLORS` values** once you capture the palette (§11 #9); until then colors may be wrong but on/off + position should be correct.

- [ ] **Step 9: Commit**

```bash
git add ReloopMixTourPro-script.js ReloopMixTourPro.midi.xml
git commit -m "feat: pad routing + Hot Cue mode + pad LEDs"
```

---

## Task 7: Mode switching + picker overlay + Beatloop / Beatjump / Sampler modes

**Files:**
- Modify: `ReloopMixTourPro-script.js`

- [ ] **Step 1: Add mode constants + size tables**

```javascript
// modeIndex per legend (row*2 + column): 0 HotCue,1 AutoLoop,2 BounceLoop,3 Sampler,
// 4 PitchCue,5 SavedLoops,6 InstantFX,7 NeuralMix (4-7 stubbed in Task 10).
ReloopMixTourPro.BEATLOOP_SIZES = [1, 2, 4, 8, 16, 32, 0.5, 0.25];
ReloopMixTourPro.BEATJUMP_SIZES = [1, 2, 4, 8, 16, 32, 0.5, 0.25];
```

- [ ] **Step 2: Add `pickMode` (hold selector + tap pad sets that deck's mode)**

```javascript
ReloopMixTourPro.pickMode = function(status, note) {
    var m = ReloopMixTourPro.MTP, st = ReloopMixTourPro.state;
    var padDeck = status - m.PAD_BASE;
    var leftColumn = (padDeck % 2 === 0);
    var row = note - m.N_PAD[0];
    var modeIndex = row * 2 + (leftColumn ? 0 : 1);   // 0..7
    var side = st.selectorHeld.left ? "left" : "right";
    st.padMode[st.sideDeck[side]] = modeIndex;
    ReloopMixTourPro.renderAllPads();                 // refresh overlay highlight
};
```

- [ ] **Step 3: Replace `renderAllPads` with an overlay-aware version + add the overlay renderer**

```javascript
ReloopMixTourPro.renderModePickerOverlay = function() {
    var m = ReloopMixTourPro.MTP, st = ReloopMixTourPro.state;
    var side = st.selectorHeld.left ? "left" : "right";
    var current = st.padMode[st.sideDeck[side]];
    var cols = [
        { physDeck: st.sideDeck.left,  colIdx: 0 },
        { physDeck: st.sideDeck.right, colIdx: 1 },
    ];
    for (var ci = 0; ci < cols.length; ci++) {
        for (var row = 0; row < 4; row++) {
            var modeIndex = row * 2 + cols[ci].colIdx;
            ReloopMixTourPro.sendPad(cols[ci].physDeck, m.N_PAD[row], "white", modeIndex === current);
        }
    }
};

ReloopMixTourPro.renderAllPads = function() {
    var m = ReloopMixTourPro.MTP, st = ReloopMixTourPro.state;
    if (st.selectorHeld.left || st.selectorHeld.right) {
        ReloopMixTourPro.renderModePickerOverlay();
        return;
    }
    var cols = [
        { physDeck: st.sideDeck.left,  leftColumn: true },
        { physDeck: st.sideDeck.right, leftColumn: false },
    ];
    for (var ci = 0; ci < cols.length; ci++) {
        for (var row = 0; row < 4; row++) {
            var targetDeck, slot;
            if (st.split) { targetDeck = cols[ci].physDeck; slot = row; }
            else { targetDeck = st.sideDeck[st.activeSide]; slot = cols[ci].leftColumn ? row : row + 4; }
            var c = ReloopMixTourPro.padColorForSlot(targetDeck, slot);
            ReloopMixTourPro.sendPad(cols[ci].physDeck, m.N_PAD[row], c.color, c.bright);
        }
    }
};
```

- [ ] **Step 4: Make selector press/release refresh the pads (so the overlay appears/disappears)**

Edit `ReloopMixTourPro.selector` (from Task 2): after updating state, add `ReloopMixTourPro.renderAllPads();` as the last line.

- [ ] **Step 5: Add the three mode actions + replace `padAction`**

```javascript
ReloopMixTourPro.padBeatloop = function(group, slot, value) {
    if (value === 0) { return; }
    engine.setValue(group, "beatloop_" + ReloopMixTourPro.BEATLOOP_SIZES[slot] + "_toggle", 1);
};

ReloopMixTourPro.padBeatjump = function(group, slot, value) {
    if (value === 0) { return; }
    var dir = ReloopMixTourPro.state.shift ? "backward" : "forward";
    engine.setValue(group, "beatjump_" + ReloopMixTourPro.BEATJUMP_SIZES[slot] + "_" + dir, 1);
};

ReloopMixTourPro.padSampler = function(_group, _deckIndex, slot, value) {
    if (value === 0) { return; }
    var sg = "[Sampler" + (slot + 1) + "]";
    if (ReloopMixTourPro.state.shift) { engine.setValue(sg, "stop", 1); }
    else if (engine.getValue(sg, "track_loaded") > 0) { engine.setValue(sg, "cue_gotoandplay", 1); }
    else { engine.setValue(sg, "LoadSelectedTrack", 1); }
};

ReloopMixTourPro.padAction = function(group, deckIndex, slot, value) {
    switch (ReloopMixTourPro.state.padMode[deckIndex]) {
        case 0: ReloopMixTourPro.padHotcue(group, slot, value); break;
        case 1: ReloopMixTourPro.padBeatloop(group, slot, value); break;
        case 2: ReloopMixTourPro.padBeatjump(group, slot, value); break;
        case 3: ReloopMixTourPro.padSampler(group, deckIndex, slot, value); break;
        default: break; // 4-7 stubbed (Task 10)
    }
};
```

- [ ] **Step 6: Replace `padColorForSlot` to cover modes 0–3**

```javascript
ReloopMixTourPro.padColorForSlot = function(deckIndex, slot) {
    var g = ReloopMixTourPro.groupForDeck(deckIndex);
    switch (ReloopMixTourPro.state.padMode[deckIndex]) {
        case 0: { // Hot Cue
            var n = slot + 1;
            if (engine.getValue(g, "hotcue_" + n + "_status") > 0) {
                return { color: ReloopMixTourPro.nearestColorName(engine.getValue(g, "hotcue_" + n + "_color")), bright: true };
            }
            return { color: "off", bright: false };
        }
        case 1: { // Auto Loop
            var on = engine.getValue(g, "beatloop_" + ReloopMixTourPro.BEATLOOP_SIZES[slot] + "_enabled") > 0;
            return { color: "blue", bright: on };
        }
        case 2: // Bounce Loop (momentary; steady ambient green in v1)
            return { color: "green", bright: false };
        case 3: { // Sampler
            var sg = "[Sampler" + (slot + 1) + "]";
            return { color: "pink", bright: engine.getValue(sg, "play") > 0 };
        }
        default: return { color: "off", bright: false };
    }
};
```

- [ ] **Step 7: Add connections so beatloop/sampler LEDs refresh**

Append to `connectPadLEDs` (inside the deck loop, after the hotcue connections):

```javascript
        engine.makeConnection(g, "loop_enabled", function() { self.renderAllPads(); });
    }
    for (var s = 1; s <= 8; s++) {
        engine.makeConnection("[Sampler" + s + "]", "play", function() { self.renderAllPads(); });
        engine.makeConnection("[Sampler" + s + "]", "track_loaded", function() { self.renderAllPads(); });
    }
```

(Adjust the existing loop braces so the sampler connections run once, not per deck.)

- [ ] **Step 8: Live check — modes + picker**

Reload.
- Hold the left selector → all 8 pads turn **white**, the left deck's current mode **bright white**. Tap a pad → that legend's mode is selected; release → pads show the new mode's colors.
- Auto Loop: tap a pad → beatloop of that size toggles; active size pad bright blue, others ambient blue.
- Bounce Loop: tap → beatjump fires (ambient green pads).
- Sampler: tap empty → loads selected track to that sampler; tap loaded → plays; bright pink while playing; SHIFT+tap → stops.

Expected: mode switching + per-mode actions/colors work on all decks and in split.

- [ ] **Step 9: Commit**

```bash
git add ReloopMixTourPro-script.js
git commit -m "feat: pad mode switching, picker overlay, beatloop/beatjump/sampler"
```

---

## Task 8: FX paddles + backspin + wet/dry + filter LED

**Files:**
- Modify: `ReloopMixTourPro-script.js` (paddle handler, filter LED)
- Modify: `ReloopMixTourPro.midi.xml` (paddle bindings, wet/dry direct binding)

- [ ] **Step 1: Add paddle + filter constants to `MTP`**

```javascript
// add inside MTP = { ... }
    PADDLE_BASE: 0x98,    // 0x98..0x9B note 0x05 = FX paddle per deck
    FILTER_EPS: 0.02,     // center deadzone for the filter-engaged LED
```

- [ ] **Step 2: Add the paddle handler**

```javascript
ReloopMixTourPro.paddle = function(_ch, _ctrl, value, status, _group) {
    var deckIndex = status - ReloopMixTourPro.MTP.PADDLE_BASE; // 0..3
    var group = ReloopMixTourPro.groupForDeck(deckIndex);
    if (ReloopMixTourPro.state.shift) {
        engine.spinback(deckIndex + 1, value > 0);            // SHIFT: backspin (mirrors engage/release)
    } else {
        engine.setValue("[EffectRack1_EffectUnit1]",
                        "group_" + group + "_enable", value > 0 ? 1 : 0); // mirror message
    }
};
```

- [ ] **Step 3: Add the filter-engaged LED renderer + connections**

```javascript
ReloopMixTourPro.renderFilterLED = function(deckIndex) {
    var m = ReloopMixTourPro.MTP;
    var v = engine.getValue("[QuickEffectRack1_" + ReloopMixTourPro.groupForDeck(deckIndex) + "]", "super1");
    var engaged = Math.abs(v - 0.5) > m.FILTER_EPS;
    ReloopMixTourPro.sendPad(deckIndex, m.N_FILTER_LED, engaged ? "blue" : "off", true);
};

ReloopMixTourPro.connectFilterLEDs = function() {
    var self = ReloopMixTourPro;
    for (var d = 0; d < self.MTP.decks; d++) {
        (function(deckIndex) {
            engine.makeConnection("[QuickEffectRack1_" + self.groupForDeck(deckIndex) + "]",
                "super1", function() { self.renderFilterLED(deckIndex); });
        })(d);
    }
};
```

- [ ] **Step 4: Wire into `init` / `shutdown`**

In `init` add `ReloopMixTourPro.connectFilterLEDs();` and a loop `for (var fd=0; fd<m.decks; fd++) ReloopMixTourPro.renderFilterLED(fd);` (note: `m` must be defined in init — add `var m = ReloopMixTourPro.MTP;` at the top of init). In `shutdown`'s deck loop add `ReloopMixTourPro.sendPad(d, m.N_FILTER_LED, "off", false);`.

- [ ] **Step 5: XML — bind paddles + wet/dry knob**

Paddle note `0x05` on `0x98`–`0x9B` → `ReloopMixTourPro.paddle` (group = the deck, used for nothing but clarity):

```xml
            <control>
                <group>[Channel1]</group><key>ReloopMixTourPro.paddle</key>
                <status>0x98</status><midino>0x05</midino>
                <options><script-binding/></options>
            </control>
            <!-- repeat 0x99/0x9A/0x9B -> [Channel2/3/4] -->
```

Wet/dry knob `BF 02` → EffectUnit1 mix (direct, shared):

```xml
            <control>
                <group>[EffectRack1_EffectUnit1]</group><key>mix</key>
                <status>0xBF</status><midino>0x02</midino>
                <options><normal/></options>
            </control>
```

- [ ] **Step 6: Live check — paddles, backspin, filter LED**

Reload. Ensure an effect is loaded in Effect Unit 1 (Mixxx loads one by default; otherwise load one in the FX rack).
- Push a paddle **down** → FX engages on that deck while held, releases on let-go. Push **up** → FX latches on until pulled back. Turn the wet/dry knob (`BF 02`) → mix changes.
- **SHIFT + paddle** → the deck backspins; resumes on release/return.
- Turn a filter knob off-center → its blue LED lights; recenter → off. (Confirm address `94..97 24` and the blue value; tune `COLORS.blue`/`N_FILTER_LED`/`FILTER_EPS` — §11 #9/#10.)

Expected: FX + backspin + filter LED behave on all 4 decks.

- [ ] **Step 7: Commit**

```bash
git add ReloopMixTourPro-script.js ReloopMixTourPro.midi.xml
git commit -m "feat: FX paddles, backspin, wet/dry, filter engaged LED"
```

---

## Task 9: Selection/selector/split LEDs + VU meters + full render on state change

**Files:**
- Modify: `ReloopMixTourPro-script.js`

- [ ] **Step 1: Add VU constants to `MTP` (addresses are guesses — capture §11 #6)**

```javascript
// add inside MTP = { ... }
    VU_STATUS_BASE: 0xB0,  // guess: 0xB0..0xB3 CC
    N_VU: 0x02,            // guess
```

- [ ] **Step 2: Add selection/selector/split LED renderer**

```javascript
ReloopMixTourPro.renderSelectionLEDs = function() {
    var m = ReloopMixTourPro.MTP, st = ReloopMixTourPro.state;
    // Per-side active-deck indicator (note 0x08 on the deck's button channel)
    ReloopMixTourPro.sendLED(m.NOTE_BASE, 0, m.N_SELECTOR, st.sideDeck.left === 0 ? m.BRIGHT : m.OFF);
    ReloopMixTourPro.sendLED(m.NOTE_BASE, 2, m.N_SELECTOR, st.sideDeck.left === 2 ? m.BRIGHT : m.OFF);
    ReloopMixTourPro.sendLED(m.NOTE_BASE, 1, m.N_SELECTOR, st.sideDeck.right === 1 ? m.BRIGHT : m.OFF);
    ReloopMixTourPro.sendLED(m.NOTE_BASE, 3, m.N_SELECTOR, st.sideDeck.right === 3 ? m.BRIGHT : m.OFF);
    // Radio group: left selector (94 08), split (9F 09), right selector (95 08)
    midi.sendShortMsg(m.PAD_BASE + 0, m.N_SELECTOR, (!st.split && st.activeSide === "left")  ? m.BRIGHT : m.OFF);
    midi.sendShortMsg(m.PAD_BASE + 1, m.N_SELECTOR, (!st.split && st.activeSide === "right") ? m.BRIGHT : m.OFF);
    midi.sendShortMsg(m.GLOBAL_NOTE, m.N_SPLIT, st.split ? m.BRIGHT : m.OFF);
};
```

- [ ] **Step 3: Add VU connection + a `renderEverything` aggregate**

```javascript
ReloopMixTourPro.connectVU = function() {
    var self = ReloopMixTourPro, m = self.MTP;
    for (var d = 0; d < m.decks; d++) {
        (function(deckIndex) {
            engine.makeConnection(self.groupForDeck(deckIndex), "vu_meter", function(v) {
                midi.sendShortMsg(m.VU_STATUS_BASE + deckIndex, m.N_VU, Math.round(v * 0x7F));
            });
        })(d);
    }
};

ReloopMixTourPro.renderEverything = function() {
    ReloopMixTourPro.renderAllTransportLEDs();
    ReloopMixTourPro.renderAllPads();
    ReloopMixTourPro.renderSelectionLEDs();
    for (var d = 0; d < ReloopMixTourPro.MTP.decks; d++) { ReloopMixTourPro.renderFilterLED(d); }
};
```

- [ ] **Step 4: Re-render on state changes**

- In `ReloopMixTourPro.selector` (Task 2/7): replace the final `renderAllPads()` with `renderAllPads(); ReloopMixTourPro.renderSelectionLEDs();`.
- In `ReloopMixTourPro.splitButton`: after `state.split = true`, add `ReloopMixTourPro.renderAllPads(); ReloopMixTourPro.renderSelectionLEDs();`.
- In `ReloopMixTourPro.deckSwitch`: after updating `sideDeck`, add `ReloopMixTourPro.renderEverything();`.

- [ ] **Step 5: Full state on `init` (override the power-on default), connect VU**

At the end of `init`, after the existing connect/render calls, add:

```javascript
    ReloopMixTourPro.connectVU();
    ReloopMixTourPro.renderEverything();
```

- [ ] **Step 6: Live check — selection LEDs + VU + power-on override**

Reload. Watch the controller:
- On load, LEDs settle to a known state (default: left selector lit, deck-1/2 selection indicators lit, not the power-on default).
- Tap selectors / split → the lit selector follows; the per-side deck indicators follow deck switches.
- Play audio → VU meters move (if address is right; otherwise tune `VU_STATUS_BASE`/`N_VU`, §11 #6).

Expected: selection feedback matches what you press; VU responds. **Confirm the `08`/`09` selector LED values and VU address** (§11 #6/#7).

- [ ] **Step 7: Commit**

```bash
git add ReloopMixTourPro-script.js
git commit -m "feat: selection/selector/split LEDs, VU meters, full-state render"
```

---

## Task 10: Stub the remaining 4 pad modes + idle colors

**Files:**
- Modify: `ReloopMixTourPro-script.js`

- [ ] **Step 1: Add stub action cases to `padAction` (before `default:`)**

```javascript
        case 4: // Pitch Cue (stub — no native Mixxx pitch-play; spec §5.5)
        case 5: // Saved Loops (stub)
        case 6: // Instant FX (stub)
        case 7: // Neural Mix / stems (stub — Mixxx 2.5+)
            if (value > 0) { engine.log("ReloopMixTourPro: pad mode " + ReloopMixTourPro.state.padMode[deckIndex] + " not implemented"); }
            break;
```

- [ ] **Step 2: Add stub idle colors to `padColorForSlot` (before `default:`)**

```javascript
        case 4: return { color: (slot === 0 ? "white" : "red"), bright: (slot === 0) }; // Pitch Cue: pad1 white, rest ambient red
        case 5: return { color: "blue", bright: false };   // Saved Loops: ambient blue
        case 6: return { color: "green", bright: false };  // Instant FX: ambient green
        case 7: return { color: (slot < 4 ? "yellow" : "white"), bright: (slot < 4) }; // Neural Mix: 1-4 yellow, 5-8 ambient white
```

- [ ] **Step 3: Live check — stub modes show colors, no errors**

Reload. Hold a selector and pick Pitch Cue / Saved Loops / Instant FX / Neural Mix. Pads show the documented idle colors; pressing pads logs "not implemented" and does nothing else (no crash).

Expected: all 8 legend positions selectable; stub modes are visually correct and inert.

- [ ] **Step 4: Commit**

```bash
git add ReloopMixTourPro-script.js
git commit -m "feat: stub remaining 4 pad modes with idle colors"
```

---

## Task 11: Hardware-tuning pass (walk the spec §11 capture checklist)

**Files:**
- Modify: `ReloopMixTourPro-script.js` (config constants only)

This task tunes the guessed constants against the real hardware using MidiView + live observation. Each is a config-only change.

- [ ] **Step 1: Selector release (§11 #1)** — Confirmed in Task 2's log. If selectors do NOT send note-off, change `selector` so a tap toggles `selectorHeld[side]` instead of press/release, and re-test the MODE features.

- [ ] **Step 2: Pad color palette (§11 #9)** — In MidiView, send notes `94 14` with values `0x00`–`0x7F` (or use Mixxx's controller wizard) to discover which value = which color at ambient vs bright. Fill in `MTP.COLORS` accordingly. Re-check hot-cue colors and per-mode colors.

- [ ] **Step 2b: Pad color encoding** — If colors turn out to need separate RGB messages (not a single palette byte), change `sendPad` to emit the controller's RGB form; the call sites stay the same.

- [ ] **Step 3: Brightness + blink (§11 #8)** — Tune `MTP.AMBIENT`, `MTP.BRIGHT`, `MTP.BLINK_SLOW`, `MTP.BLINK_FAST` so play blinks slowly, cue faster, and ambient is a dim backlight.

- [ ] **Step 4: Filter LED (§11 #10)** — Confirm `N_FILTER_LED` (`0x24`) lights the correct LED blue; tune `FILTER_EPS`.

- [ ] **Step 5: VU meter (§11 #6)** — Find the real meter LED address; set `VU_STATUS_BASE`/`N_VU`. Add throttling if it floods.

- [ ] **Step 6: Selector/split/selection LED values (§11 #7)** — Confirm `0x7F`/`0x00` light `94 08`, `95 08`, `9F 09`, `90-93 08` as expected.

- [ ] **Step 7: Non-split pad channels (§11 #2)** — Verify that in split-off mode both pad columns still arrive on their side-deck channels (the `routePad`/`renderAllPads` assumption). If instead all 8 arrive on one channel/8 notes, adjust `routePad` + `renderAllPads` accordingly.

- [ ] **Step 8: Final commit**

```bash
git add ReloopMixTourPro-script.js
git commit -m "chore: tune hardware constants (colors, brightness, blink, VU, filter)"
```

---

## Done

The mapping now covers: 4-deck routing, analog controls, SHIFT + MODE modifiers, transport with blink LEDs, the In-Out-Ex 3-state loop machine + modifier loop/tempo functions, the full split/active pad system with mode-picker overlay, Hot Cue/Beatloop/Beatjump/Sampler modes with RGB pad colors (+ 4 stubs), FX paddles with backspin + wet/dry, the filter-engaged LED, selection/selector/split LEDs, and VU meters — all driven by the script per the spec.

