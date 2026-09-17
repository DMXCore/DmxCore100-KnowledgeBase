# Custom Menus and Control Surfaces

**Audience:** integrators, plugin authors, and AI agents planning an
integration without access to the DMX Core 100 source code.
**Verified against:** DMX Core 100 software `main` at commit `1cfcdeb0`
(2026-09-16).
**User-facing documentation:**
<https://docs.dmxcore.com/dmx-core-100/scheduling-automation/custom-menus>,
<https://docs.dmxcore.com/dmx-core-100/control-surfaces>,
<https://docs.dmxcore.com/dmx-core-100/control-surfaces/configuring>,
<https://docs.dmxcore.com/dmx-core-100/control-surfaces/supported-devices>,
<https://docs.dmxcore.com/dmx-core-100/control-surfaces/surface-operator>

The user documentation describes how to build menus and surfaces. This
document describes what the device does with a press: how a physical or
on-screen control resolves to an assignment, how each transport turns its
messages into press, release, and value events, what "active" means for a
light or a key face, the exact timing constants, and what neither surface
kind can do today.

---

## 1. Mental model

Both features are **buttons and sliders that carry trigger actions**. The
action record and its modes are those of the Triggers and Actions document;
the continuous targets are those of the Faders document. What differs is
who presses and how the device answers:

| | Custom menu | Control surface |
|---|---|---|
| Who | Casual users: touchscreen, phone, guest tablet | Hands-on operators: physical controllers, or the browser operator view |
| Layout | Items in a list, submenus, per-menu look | Sections of a device, banks, slots that mirror the hardware |
| Confirmation | A Yes/No dialog per item | Hold-to-confirm on the physical control; a dialog in the browser |
| Release edge | Web taps have none except Flash; touchscreen has press and release | Every transport has a real press and release except the cases in section 4.4 |
| Feedback | Highlighted items | LED colors, key faces, LCD strip, OSC echo, operator-view state |
| Origin tag on Control Value writes | `WEB` | `SURFACE:{id}` |

Both dispatch through the same action path, so the Output Off gate, the
missing-target rules, and the origin rules apply.

---

## 2. Custom menus

### 2.1 Data model

A menu is a JSON document stored in its own database row with `enabled`,
`onlyAdmin`, and `availableToGuests` flags. The JSON carries a schema
version; the device upgrades older versions on load (the current version is
7). Menus can be exported and imported as JSON.

| Field | Meaning |
|---|---|
| `name`, `subTitle`, `code` | Identity. The code is auto-generated, editable, matched case-insensitively, and used for direct links. |
| `type` | `ITEMS` (buttons), `ITEMSSMALL` (compact list), `FADERS` (columns of sliders driven by the touchscreen rotary knob). Older `QSYS`, `SYMETRIX`, and `EXTCTRL` types are legacy and were migrated to Faders menus plus Control Values. |
| `items` | The item list (section 2.2). |
| `knobAction` | Faders type: the action a short press of the rotary knob fires, typically a Control Value Toggle for mute. When null a knob press moves focus to the next fader. |
| `icon`, `logo`, `logoHeight`, `background` | Look. |

### 2.2 Items

| Item type | Carries | Behavior |
|---|---|---|
| `ACTION` | A trigger action, `confirm`, optional live-state binding | Runs the action. With `confirm`, a Yes/No dialog first. |
| `SUBMENU` | A nested menu | Navigation. Menus nest to any depth. |
| `PRESETS`, `CUES` | Nothing | Built-in browsers listing every preset or cue. Tapping a child plays it with the **system default** fade and loop; per-item settings do not apply here. |
| `SLIDER` | A continuous action | Inline fader for any continuous target. Seeded with the target's current value on load. |
| `SEGMENTED` | A Selector Control Value code | Row of buttons for the choices, live-highlighted. Seeded with the choices and the active choice on load. |
| `VALUEDISPLAY` | A Control Value code of any kind, optional `valueFormat` | Read-only live readout, never tappable: a Level as percent, a Counter as its number, a Selector as its choice, a Toggle as On/Off. `valueFormat` uses the Stream Deck key syntax (`{0}` the value, `{1}` the default text, `Yes|No` for a Toggle). On the web the value sits under the name; on the touchscreen it is the card title. |
| `MASTERDIMMER` | Nothing | Legacy inline master slider; `SLIDER` with the master target is the successor. |
| `OSC` | Client, address, parameter | Sends one OSC message from the web or touchscreen client. UI-only. |
| `STOPOUTPUT` | Nothing | Legacy; upgraded to an Action item with Blackout. |

Item extras: `name`, `description`, `icon`, `background`, `displayPlayerControl`
(shows the transport for the playing item), and for the Faders type
`switchTimeoutSeconds`, how long after the operator last touched a fader
before knob focus returns to it.

### 2.3 Live state

An Action item highlights while its **state binding** matches. The binding
is explicit (`stateControlValueCode` and `stateValue`) or derived from a
Control Value action: Set highlights while the value equals the set value,
Toggle and Follow while the value is on. Up and Down derive nothing. Items whose action
targets a cue, preset, sound, timeline, schedule, mute, output, or blackout
highlight while that target is active by the same rule control surfaces use
(section 4.5). Updates arrive over SignalR from the Control Value status
feed and the playback status.

### 2.4 Execution

- **Web and touchscreen** both dispatch Action items through the shared
  action path with origin `WEB`.
- The web endpoint, `POST /api/website/custommenu/execute`, validates that
  the action's target exists and returns an error to the user when it does
  not. This is the one place a missing target is surfaced instead of only
  logged, because a person is watching.
- `GET /api/website/custommenu` returns the menus a caller may see: guests
  get only guest-enabled menus; authenticated users get everything except
  admin-only menus they lack permission for. Sliders and segmented items
  come pre-seeded with live values.
- Direct links: `/guest/custommenu?code=X` works without login for
  guest-enabled menus; `/op/custommenu?code=X` needs a login.
- The host setting **Only show custom menu** makes non-admin touchscreen
  users start on the custom menu; admins are exempt.
- Web taps have no release edge. A Flash item on the web is handled by the
  client sending press and release; Momentary works the same way on the
  touchscreen.

---

## 3. Control surfaces: the model

| Concept | Meaning |
|---|---|
| Surface | A named configuration of type `MIDIKEYPAD`, `STREAMDECK`, `KEYDIGITAL`, or `OSC`, with one transport from the type's allowed set. Exists and is editable whether or not hardware is connected. |
| Transport | `MIDI` (USB), `RTPMIDI`, `NETWORKMIDI2`, `STREAMDECK` (USB HID), `STREAMDECKNETWORK` (Network Dock over TCP), `KEYDIGITAL` (TCP), `OSC` (UDP via a configured OSC client). |
| Device model | Fixes the grid for Stream Deck MK.2, XL, Mini, Plus, and the KD-WP8. MIDI and OSC surfaces carry only an informational model hint. |
| Section | A physical region: label, widget type `PAD` or `SLIDER`, grid columns, reversed layout, `bankScoped`. All assignments in a section share the widget type. |
| Bank | A software page. Bank-scoped sections swap their assignments per bank; pinned sections show the same assignments in every bank. Each bank has a label and default active and inactive indicator colors. |
| Assignment | One control: section, optional bank, slot index, label, **either** a trigger action **or** a continuous action, appearance, `holdToConfirm`, and for Stream Decks `showValue` and `valueFormat`. Binding is `MidiBinding` (message type, channel 0–15, number 0–127), `OscBinding` (address path, argument index), or none for slot-addressed devices and web-only surfaces. |
| Appearance | Icon, font size (negative bold, 0 with an icon means icon only), active and inactive background colors, active and inactive indicator colors. Colors are names or hex; each device maps them to its palette. |
| OSC client | A configured source IP with a feedback port, or the any-source wildcard. Only configured clients can bind to a surface and receive feedback. |

The active bank per surface is persisted, so a restart returns to the same
page. Surfaces are exported and imported as JSON, and templates per type
seed new surfaces.

---

## 4. Dispatch

### 4.1 Resolution order

An incoming control event is resolved to an assignment in two steps:

1. The **active bank's** assignments in bank-scoped sections, matched by
   address (MIDI tuple, OSC address plus argument index, or slot).
2. **Global** assignments, those in pinned sections, matched by the same
   address.

A press is always broadcast to the operator view so the cell flashes, even
when the assignment has no action yet.

### 4.2 MIDI (USB, RTP-MIDI, Network MIDI 2)

- **Note** bindings are buttons: Note On is the press, Note Off (or Note On
  with velocity 0) is the release. Releases are only processed when they
  have something to do, a Flash handle to release or a hold to cancel, so a
  keyboard used as a keyboard does not spawn work per key-up.
- **Control Change** bindings are continuous: the 0..127 value goes through
  soft takeover and scaling (Faders document, sections 3.1 and 5). A CC can
  also be bound to a pad; it then presses on non-zero and releases on zero.
- **Program Change** bindings fire on receipt and have no release.
- MIDI does not pass through input triggers. A MIDI message either matches
  a surface assignment or is ignored.
- RTP-MIDI and Network MIDI 2 decode to the same channel-voice semantics and
  share every rule above. Each can act as listener or initiator.
- USB devices are matched by port name; a plug event re-scans and rebuilds
  the runtime set with a brief interruption.

### 4.3 OSC surfaces

- Precedence for an incoming message: a surface bound to the sender's IP
  claims every message from that IP; the any-source surface claims only
  addresses it has assignments for (plus `/ping`); plugin subscriptions come
  next; input triggers and the built-in `/dmxcore` tree last. A claimed
  message never reaches input triggers.
- Messages from unconfigured IPs are recorded in a bounded discovery buffer
  the editor shows for one-click configuration, then fall through.
- Continuous values are clamped 0..1 and mapped to 0..127 so they share the
  MIDI machinery. Sliders support **absolute** and **relative** input.
- **Toggle mode uses value-as-state**: the received value is the desired
  state, non-zero activates, zero deactivates, no-op when already there.
  This exists because feedback-synced controllers only transmit on value
  changes, so press-count toggling would deadlock once feedback lights the
  button. Pair Toggle assignments with toggle-type controller buttons and
  momentary buttons with Flash.
- Address-only messages (no argument) toggle blindly per message.
- Feedback is echoed to the client's feedback port after every state change
  and re-asserted after a momentary release, since such controllers reset
  their local visual to 0.

### 4.4 Stream Deck, KD-WP8, and the operator view

- **Stream Deck** keys are slot-addressed. Press and release are real.
  Settings: brightness 0–100 (default 30) and sleep timeout in seconds
  (default 60, 0 never). Any input on a sleeping deck, a key, a dial, a
  dial press, or a strip swipe, only wakes it and is not dispatched. Key faces render label, icon, colors, and, for
  Control Value keys, the live value. Classic decks need a 180° image
  rotation; encoder-capable models do not. The Plus has four dials with
  press (a `PAD` section) and rotation (a `SLIDER` section) plus the LCD
  strip.
- **Dials**: endless encoders are relative; each tick nudges the target's
  current value, with the accumulator seeded from the target so the first
  tick after a reload starts at the right place. With the **Control Value
  (step per tick)** target a tick is one Up or Down, so a Counter counts and
  a Selector cycles; `inverted` swaps direction. A horizontal swipe on the
  LCD strip switches banks: right is next, left is previous.
- **LCD strip**: bank label, or one segment per dial (label and live value)
  once any dial has a target. Repaints are coalesced to at most 10 per
  second.
- **Network Dock** is the same model over TCP with JPEG key images.
- **KD-WP8**: eight buttons, slot = button id minus one, no banks, three
  LED states (off, blue, red) mapped from the assignment colors, discovered
  by a network scan because it has no mDNS.
- **Operator view** (browser): press and release via the hub or the
  `controlsurface/trigger`, `/release`, `/continuous`, and `/setbank`
  endpoints, so Flash and Momentary work from a browser. Hold-to-confirm
  becomes a Yes/No dialog. A surface with no transport binding is a
  web-only touch panel.

### 4.5 What "active" means

LED colors, key faces, and operator-view cells use one rule:

| Action | Active when |
|---|---|
| Cue, Sound, Timeline | That item is playing |
| Preset | The preset is among the currently applied temporary presets (not merely "was started"; an effects-only preset that stopped is inactive) |
| Ambient Preset | The ambient preset is active |
| Schedule Toggle | The schedule is enabled |
| Mute, Output Toggle, Blackout | The corresponding global flag is engaged |
| Stop Playback | Nothing is playing |
| Switch Bank | The target bank is the active one |
| Control Value Toggle, Control Value Follow | The value is on |
| Control Value Set | The value equals the set value |
| Everything else, including Control Value Up and Down | Never |

Repaints are triggered by playback status changes, the mute, output, and
blackout flags, Control Value status, and bank switches. A press also
tap-flashes the control's active color for 120 ms.

### 4.6 Hold-to-confirm

- Duration is the host setting **Control surface hold to confirm**, default
  1500 ms, range 250–10000, evaluated on a 100 ms tick.
- Applies only to assignments that opted in, in Normal or Toggle mode, that
  are not hold-repeatable. Releasing early cancels; losing the device
  mid-hold cancels. A re-press during a pending hold is ignored and does not
  restart the timer.
- Ignored where there is no release edge to hold against: Flash and
  Momentary (already hold gestures), Control Value Up and Down (auto-repeat
  while held), MIDI Program Change, address-only OSC, OSC Toggle
  (value-as-state means two presses with no release between them), and
  Stream Deck+ dial presses.
- Visuals while holding: a sweeping ring and darkened face on Stream Deck,
  red/off blink on the KD-WP8, pad blink on the Akai LPD8 mk2. The blink
  runs on the wall clock, not on progress.

### 4.7 Auto-repeat

A held key bound to Control Value Up or Down fires once, then after 400 ms
repeats every 150 ms until release, on Stream Deck USB and Network Dock and
on OSC surfaces. The repeat loop only arms for a key still physically down
after the press dispatch, so a quick tap never leaves a runaway repeat. The
action's step amount applies on every repeat.

### 4.8 Continuous feedback

Live values from physical knobs and operator sliders are broadcast to
operator-view clients throttled to one per 50 ms per assignment, carrying
the soft-takeover state so the view can show the pinned thumb and the
physical ghost marker. Reverse sync from the engine is the 4 Hz path in the
Faders document.

---

## 5. Worked example: a bar wall keypad and a staff Stream Deck

Goal: a KD-WP8 by the door for staff, a Stream Deck+ at the bar, and a
guest menu on a lobby tablet.

1. **KD-WP8** surface, type KeyDigital, found by Scan. Buttons: `OPEN` preset
   (Normal), `CLOSE` timeline (Normal), `BLACKOUT` with hold-to-confirm and
   red inactive color, `MUTE` Toggle. LEDs show blue while a look is active
   and red for the armed blackout key.
2. **Stream Deck+** surface: bank 1 keys for show cues; dials 1 and 2 with
   the step-per-tick target on `HOME_SCORE` and `AWAY_SCORE`, dial presses
   as Set 0; dial 3 on `VOL1` as a level. The strip reads `HOME 14 | AWAY
   7 | VOL1 63%`. A second bank for maintenance looks, switched by an LCD
   swipe. The `+6` key is Control Value Up with amount 6; holding the `+1`
   key repeats.
3. **Guest menu** `LOBBY`, available to guests, `ITEMS` type: three Action
   items for looks with live state, a `SLIDER` on the master, a `SEGMENTED`
   item on the audio source Selector. Printed as a QR code to
   `/guest/custommenu?code=LOBBY`.

What you will observe: every press on any of the three lands in the same
dispatcher. A look started from the keypad lights the matching Stream Deck
key and the matching menu item within a status tick. The blackout key on
the keypad blinks for 1.5 s and fires only if still held; the same action
on the guest menu would ask Yes/No, but a guest menu should not carry it.

---

## 6. Limits and gaps

Things neither feature can do today. When a design needs one of these,
request it from DMX Core rather than working around it. The tracker is
private; quote the number when you talk to DMX Core and read
[issues.md](../issues.md) for what each one asks for.

| Gap | Detail | Status |
|---|---|---|
| MIDI LED and motor feedback beyond the Akai LPD8 mk2 | LED protocols are None or Akai SysEx RGB. Other controllers get no color feedback; motorized MIDI faders get no position feedback. | Deferred |
| 14-bit MIDI and NRPN | Control Change 0..127 only. | Not planned |
| Auto-repeat on MIDI and KD-WP8 buttons | Stream Deck and OSC only. | Deferred |
| Per-item confirmation on the touchscreen for surfaces | Hold-to-confirm needs a physical release; the operator view uses a dialog. | By design |
| Custom menu on the Integration API or MCP | Menus are UI constructs; integrations use entities. | By design |
| Menu items other than actions and sliders on the web | The `OSC` item sends from the client, not the device; use an OSC output event for device-side sending. | By design |
| OSC surfaces over IPv6 | IPv4 source addresses only. | Not planned |
| Bank state per client | The active bank is one value per surface, shared by the device and every operator-view tab. | By design |
| Stream Deck key readout for non-Control Value actions | Live values are Control Value keys and dial targets only; cue progress or fixture levels do not render on keys. | Not planned |
