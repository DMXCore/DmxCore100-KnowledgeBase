# Faders and Continuous Control

**Audience:** integrators, plugin authors, and AI agents planning an
integration without access to the DMX Core 100 source code.
**Verified against:** DMX Core 100 software `main` at commit `d1357447`
(2026-09-16).
**User-facing documentation:**
<https://docs.dmxcore.com/dmx-core-100/lighting/faders>,
<https://docs.dmxcore.com/dmx-core-100/lighting/fixture-control>,
<https://docs.dmxcore.com/dmx-core-100/control-surfaces/configuring>,
<https://docs.dmxcore.com/dmx-core-100/integrations/control-values>

This document describes the **continuous-control bus** underneath the Faders
page, control-surface knobs, OSC faders, value-mode triggers, and Control
Values: what a fader actually writes, what "direct control" means for a
fixture, how every surface stays in sync, how soft takeover decides whether a
physical knob is allowed to move a value, how fades to a position run on the
device, and what the bus cannot do today.

---

## 1. Mental model

Every continuous control in the system, physical or on screen, resolves to
a **continuous action**: a target plus a scaling. The targets are:

| Target | Target code | What it writes |
|---|---|---|
| `MASTERDIMMER` | none | The master dimmer 0..1, scaling everything the fixture engine outputs |
| `ZONEINTENSITY` | zone code | The zone dimmer 0..1, multiplied with the master at render time |
| `FIXTUREINTENSITY` | fixture code | The fixture's intensity modifier 0..1 |
| `FIXTURERGBRED`, `FIXTURERGBGREEN`, `FIXTURERGBBLUE` | fixture code | One color channel of the fixture's modifier |
| `AUDIOVOLUME` | none | The master audio volume 0..1 |
| `CONTROLVALUELEVEL` | Control Value code | A Level Control Value, which may in turn drive one of the targets above |
| `NONE` | | Unassigned: the raw value is broadcast for feedback and nothing moves |

A fader strip on the Faders page, a MIDI knob, a Stream Deck+ dial, an OSC
fader, a custom-menu slider, and a value-mode input trigger are all
different **sources** feeding the same sink. The device does not
distinguish them once the value arrives, except by the origin tag it
carries into Control Value writes.

Two things make this a bus rather than a set of one-way controls:

- **Reverse sync.** When a target moves for any reason (a cue, a preset, a
  script, another surface), every continuous assignment pointing at that
  target is re-read and its surface updated: slider thumbs, OSC motor
  faders, Stream Deck dial state, and knob soft-takeover state.
- **Control Values as named handles.** A Level Control Value can *drive* a
  target and follow it back (Control Values document, section 4.4), so a
  value that several controls share, that a DSP owns, or that a script
  wants to fade by name, gets a code.

---

## 2. What a fixture fader writes

### 2.1 The modifier and direct control

Each fixture instance has a live **modifier**: intensity, red, green, blue,
amber, white, cool white, warm white, ultraviolet, and a map of custom
functions by function id. Three modifier categories are tracked
separately: intensity, color, and custom, plus an effect slot. Each
category is in one of these states:

| State | Meaning |
|---|---|
| `NONE` | Nothing holds this category; the fixture shows its ambient look or nothing |
| `TEMPPRESET` | A preset is holding the category (the preset code is reported) |
| `DIRECT` | An operator or automation has taken the category over |
| `AMBIENT` | The category is under the ambient fallback (reported as the consolidated state) |

A fader write on the Faders page, on Fixture Control, from a control
surface, from a script `setFixture`, or from a `fixture.X` entity command
is a **partial update**: the caller copies the live modifier, changes only
the channels it means to change, and submits the copy. A write that actually
changes something switches the touched categories to `DIRECT`, replacing
any preset hold. A bare "intensity only" object would zero every color
channel, so every built-in path copies first.

The consolidated `ModifierControlledBy` is the worst case across the
categories, and is what non-diagnostic views show. `DIRECT` is sticky: the
fixture stays under direct control until released, even if the value is
later set back to what a preset would have given it.

### 2.2 Release

Releasing a fixture (the strip's release button, the dialog, Release All,
`releaseFixture` in a script, the MCP tool, or the release endpoint) clears
direct control and temporary presets for that fixture so it returns to
whatever would otherwise drive it: an ambient preset, or nothing. Release
All does it for every fixture at once.

### 2.3 Merge against cues

Fixture control and cue playback merge by priority. The output setting
**Fixture Control Priority** (1–200, default 100) is compared with the
playing cue's priority (recorded cues carry 100 unless overridden). Above
the cue, fixture control wins its actively controlled channels; below it,
the cue wins; equal merges with the global Merge Mode. The master dimmer
scales cue playback only when the **Master Dimmer Applied To Cues** output
setting is on.

### 2.4 Per-copy trims

A fixture with several copies has one persisted **intensity trim** per copy.
The fan-out faders on the Faders page and the Copy Trims card on Fixture
Control edit them:

- Dragging sends a **preview overlay** that the render loop uses in place
  of the saved trims. The overlay has a 5 s time to live and the browser
  re-sends it about once a second while dragging or soloing, so a closed
  tab or lost connection self-heals within 5 s.
- Releasing **persists** the trims on the fixture, in place, with no fixture
  reload, and clears the overlay.
- **Solo** is an overlay with every copy at 0 except the soloed one at its
  current trim. It is never saved.
- Trims are persisted fixture data, not a runtime control target. No
  continuous action or Control Value can address them (section 7).

---

## 3. Sources and their paths

| Source | Path into the bus | Notes |
|---|---|---|
| Faders page, Fixture Control | SignalR hub methods `SetFixtureModifier`, `SetMasterDimmer`, `SetZoneDimmer`, `SetControlValueLevel`, with HTTP fallback to `PUT /api/website/fixturemod`, `/masterdimmer`, `/zonedimmer`, `/controlvalue/level` | The browser coalesces per fixture with a 25 ms debounce and at most one write in flight; drag traffic stays out of the HTTP log when the hub is connected. No SQLite work on the drag path. |
| Control-surface knob or fader (MIDI USB, RTP-MIDI, Network MIDI 2, OSC surface) | Assignment's continuous action → raw 0..127 → scaling → target | Subject to soft takeover (section 5). |
| Stream Deck+ dial | Encoder rotation accumulates into a 0..127 raw value seeded from the target's current position, then the same path | |
| OSC `/dmxcore/control/{code}` | Direct Level Control Value write | No takeover, no scaling |
| Value-mode input trigger (HTTP, OSC, MQTT, Control Value) | Normalized 0..1 → the trigger's continuous action | Authoritative: no takeover. |
| Custom-menu slider (touchscreen, web) | The item's continuous action | |
| Script `masterDimmer`, `zoneDimmer`, `setFixture` | Direct engine calls, with an optional fade for master and zone | |
| Integration API, MCP, plugin entity API | `setLevel` on `system.masterdimmer`, `zone.X`, `fixture.X`, `cv.X` | Instant, no fade |
| Control Value drive | The Control Value's `drives` continuous action, origin `DRIVE:{code}` | See the Control Values document |

### 3.1 Scaling

A control surface value is a **raw byte 0..127** (MIDI CC range; OSC floats
are converted). The mapping to the target is: normalize to 0..1, optionally
invert, then scale into the action's `minValue`..`maxValue`. The reverse
mapping, target value back to a raw byte, is the exact inverse and is what
the reconciler and the drive linker use, so every path shares one piece of
math. Consequence: continuous control through a surface is quantized to
**1/127** of the range, and reads back through the same quantization.
Web faders are quantized to 1 % for display but write floats.

---

## 4. Reverse sync

The device computes a **fixture control status** snapshot at up to **4 Hz**
(a 250 ms loop, woken early by change signals) containing every fixture's
modifier and control states, the active ambient presets, the master dimmer,
and the zone dimmers. The snapshot is hashed; only a changed snapshot is
broadcast over SignalR as `FixtureControlStatus`. Consumers:

- The Faders and Fixture Control pages apply it to every strip except the
  one under an active drag. A knob, a DSP, or a second browser moving a
  fixture is indistinguishable from the page's own point of view.
- The **reconciler** re-reads every continuous assignment whose target
  changed and, unless the assignment is already within one raw step (most
  likely the surface's own move), takes the new value, arms soft takeover
  unless the physical knob already sits there, and broadcasts it so
  operator views and motorized OSC faders track. OSC surfaces with feedback
  echo are exempt from takeover because the echo keeps the control at the
  software value.
- The **Control Value drive linker** reads driven targets back and moves
  the Control Value when a target moved by other means.

Feedback latency on knob rings, Stream Deck LCDs, and slider thumbs is
therefore up to about 250 ms, plus the entity catalog's 150 ms coalescing
for integrations. A server-side fade visibly glides a fader in about four
steps per second.

---

## 5. Soft takeover

Soft takeover protects a target from jumping to wherever a physical knob
happens to be. It is a per-assignment state machine on the raw 0..127 scale
with a crossover tolerance of **2** raw steps.

Takeover is **armed** when:

1. A surface connects: physical positions are unknown, so the first touch
   of every knob needs takeover.
2. A bank switch changes what a bank-scoped knob targets.
3. The software value changes from elsewhere (a web drag, a cue, a
   script, a reconciled change) and the last known physical position is not
   within the tolerance of the new value.

While armed, physical movement is **not applied** until the knob crosses
the software value within the tolerance; the broadcast slider thumb stays
pinned at the software value and a ghost marker shows the physical
position. Once resolved, the knob tracks 1:1.

Takeover is **not** used for unassigned knobs, for value-mode input
triggers (authoritative by design), for OSC surfaces with feedback echo,
or for direct OSC and web writes.

---

## 6. Fades to a position

Fades run on the device, not in the browser:

| Target | Request | Engine behavior |
|---|---|---|
| Master dimmer | `fadeMs` on the master endpoint or hub method | Ramp in the fixture engine |
| Zone dimmer | `fadeMs` on the zone endpoint | Ramp keyed per zone, so two zones can fade at once |
| Fixture | `fadeMs` on the modifier write | A direct-control fade from the current modifier to the target over the duration, using the same fader machinery presets use. The fixture is taken over as `DIRECT` at the start. |
| Level Control Value | `fadeMs` on the level endpoint, `set(code, value, fadeMs)` in a script, or a timeline milestone duration | Ramp inside the Control Value runtime, fed through the coalesced backend write |

A **snap write** to the same target cancels a running fade and takes over
live, which is what grabbing a fader mid-fade does. Fades survive the
browser closing. FLASH and per-copy trim faders always act instantly. The
fade-mode toggle and duration are browser-local preferences, not device
state.

---

## 7. Limits and gaps

Things the bus cannot do today. When a design needs one of these, request
it from DMX Core rather than working around it. The tracker is private;
quote the number when you talk to DMX Core and read
[issues.md](../issues.md) for what each one asks for.

| Gap | Detail | Status |
|---|---|---|
| MIDI output feedback | Motorized MIDI faders and LED rings do not receive the value; only OSC echo and Control Value write-back exist. | Deferred |
| Per-copy trims as a target | Trims are persisted data; no continuous action, Control Value, script call, or entity can set them. | Out of scope by decision |
| Color as a Control Value target | A Control Value drives intensity-type targets only; RGB targets exist for surfaces but not for `drives`. | Not planned |
| Fade on surface, Integration API, and entity writes | Instant; only the web, scripts, timelines, and the Control Value level endpoint carry a fade. | Not planned |
| Fade on `setFixture` in scripts | Instant. | Not planned |
| One Control Value driving several targets | 1:1. | Not planned until needed |
| Faders page on the touchscreen | Web only; the touchscreen has custom-menu sliders. | Not planned |
| Resolution | Surface control is 1/127 of the range. 14-bit MIDI is not supported. | Not planned |
| Direct control is sticky | A fixture stays `DIRECT` until released; there is no auto-release timeout. | By design |
| Fixture level entity is the modifier | Integrations see the override slider, not the rendered output (Integration API document, section 3). | By design |
| Per-fixture priority | Fixture control priority against cues is one global setting. | Not planned |
