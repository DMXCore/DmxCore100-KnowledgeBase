# Control Values

**Audience:** integrators, plugin authors, and AI agents planning an
integration without access to the DMX Core 100 source code.
**Verified against:** DMX Core 100 software `main` at commit `90e4533b`
(2026-09-16), Plugin SDK contract 1.12.
**User-facing documentation:** <https://docs.dmxcore.com/dmx-core-100/integrations/control-values>

This document describes how Control Values behave under the covers: the data
model, the runtime rules, every surface that reads or writes them, and what
they cannot do today. Read the user documentation first for the operator's
view; read this when you need to predict what the system will do.

---

## 1. Mental model

A Control Value is a **named value on a bus**. It has a short uppercase
**code** (for example `VOL1`, `SRC`, `HOME_SCORE`) and one of four kinds:

| Kind | Holds | Typical use |
|---|---|---|
| Level | A continuous position 0.0 to 1.0 | Zone volume, house dimmer |
| Selector | One of an ordered list of named choices | Audio source, mode select |
| Toggle | On or off | Mute, "game mode" flag |
| Counter | A whole number between a minimum and a maximum | Home or away score, queue number |

Every control in the system that can address the code shares the same value:
control-surface knobs and keys, custom-menu sliders and buttons, the Faders
page, input triggers, timeline milestones, schedules, scripts, OSC, the
Integration API, MCP tools, and plugins. When any one of them writes the
value, every other one observes the change.

A Control Value is always bound to a **backend** that holds the value:

- A **plugin backend**, for an external control processor. The shipped
  Symetrix and Q-SYS plugins register the backends `SYMETRIX` and `QSYS`.
  The external device is the source of truth. The Core writes to it, and the
  device pushes its actual state back.
- The **internal backend**, named `INTERNAL`, for a value that no external
  device owns. The Core simply remembers the last written value. Use this for
  a shared handle between several controls, or as plain state for automation.

What a Control Value is **not**:

- It is not a DMX channel and has no direct DMX output. A Level Control Value
  can *drive* a lighting target (section 4.4), but that is an explicit binding.
- It is not a general variable. Level is a 0..1 fraction and Counter is a
  bounded integer; there is no text or floating-point kind (section 7).
- It is not a trigger by itself. A separate Control Value input trigger
  watches it and fires actions (section 5.5).

---

## 2. Data model

### 2.1 Control Value definition

Persisted in the device database. Field names below are the JSON names used
by the admin web API (section 5.1); the operator sees the same fields in the
editor.

| Field | Type | Applies to | Meaning |
|---|---|---|---|
| `controlValueId` | int | all | Database id. Use the code, not the id, in integrations. |
| `code` | string | all | Stable identifier, unique. Matched case-insensitively everywhere. Convention is uppercase. |
| `name` | string | all | Display name. |
| `enabled` | bool | all | Disabled values are not loaded into the runtime and do not appear in the entity catalog. |
| `kind` | `"LEVEL"`, `"SELECTOR"`, `"TOGGLE"`, `"COUNTER"` | all | The kind. Uppercase string on the wire. |
| `pluginName` | string | all | Backend name: `INTERNAL`, `SYMETRIX`, `QSYS`, or any name a plugin registered. |
| `controlId` | string | all | Backend-specific controller id the Core **writes** to. For Symetrix this is the controller number from Composer. For the internal backend it is any string. |
| `statusControlId` | string, optional | all | Controller id the Core **reads** state from when it differs from the write id. Push subscriptions and last-known values key off this id. Null means status comes from `controlId`. |
| `stepSize` | double, default 0.05 | Level | Fraction of full range that one Up or Down step moves. |
| `wrapSelector` | bool, default true | Selector | Up past the last choice wraps to the first, Down from the first wraps to the last. False clamps at the ends. |
| `choices` | array of `{ name, value }` | Selector | Ordered choices. `value` is the raw backend value for that position. |
| `counterMin`, `counterMax` | int, default 0 and 100 | Counter | The range. A reversed range is tolerated. |
| `counterStep` | int, default 1 | Counter | How much one Up or Down moves the value. |
| `counterWrap` | bool, default false | Counter | Up past the maximum wraps to the minimum and Down below the minimum wraps to the maximum. Default clamps, unlike Selector. |
| `muteControlId` | string, optional | Level | Linked mute controller on the same backend. Enables the "muted" overlay on faders. |
| `unmuteOnLevelChange` | bool | Level | With a linked mute: any level write unmutes first. |
| `drives` | continuous action, optional | Level | The lighting or audio target this value drives (section 4.4). Null means it drives nothing. |

Selector choice values: for Symetrix selectors the convention is position n of
N = (n-1) * 65535 / (N-1), but any raw value can be configured to match
non-contiguous wirings. The runtime resolves the current position to the
**nearest** choice value, not an exact match, because some devices round
selector positions on the wire.

### 2.2 Live status

The runtime keeps one status record per enabled Control Value. This is what
SignalR broadcasts, what the statuses endpoint returns, and what the script
API's `status()` exposes.

| Field | Type | Meaning |
|---|---|---|
| `code` | string | The Control Value code. |
| `kind` | `"LEVEL"`, `"SELECTOR"`, `"TOGGLE"`, `"COUNTER"` | The kind. |
| `value` | double | Level: 0..1. Toggle: 0 or 1. Selector: the raw backend value normalized to 0..1. Counter: the integer's 0..1 position within its range. Use `choiceIndex`, `choiceName`, or `number` for the resolved value. |
| `choiceIndex` | int, nullable | Selector only. Nearest choice index. |
| `choiceName` | string, nullable | Selector only. Nearest choice name. |
| `number` | int, nullable | Counter only. The current integer. |
| `muted` | bool, nullable | Level with a linked mute: whether the mute is on. Null when there is no linked mute or its state is unknown. |
| `hasValue` | bool | False until a value has been received from the backend or written locally. UIs show "unknown" rather than 0. |
| `origins` | array of string | Every origin tag that wrote the value since the last publish (section 3.4). Empty for snapshots not produced by a write, such as the initial list. |
| `external` | bool | True when the `DSP` origin is among `origins`, meaning the backend itself reported a change. Kept for the admin UI. |

### 2.3 Persistence

- Definitions live in the device database and are included in backups.
- Live values of **plugin backends** are not persisted by the Core. After a
  restart the Core re-subscribes and seeds from whatever the backend already
  knows. Until the backend reports, `hasValue` is false.
- Live values of the **internal backend** are saved in the device's system
  state file as part of the normal periodic and shutdown save. There is no
  write-on-change. A hard power cut can lose internal values written since the
  last save.

---

## 3. Runtime behavior

### 3.1 Operations

Every write path resolves to one of these operations on the runtime.

| Operation | Level | Selector | Toggle | Counter |
|---|---|---|---|---|
| Set `value` | Parse a number. `0..1` is taken as a fraction; anything above 1, or a value ending in `%`, is divided by 100. Clamped to 0..1. Optional fade (below). | Choice name (case-insensitive) or 0-based index. | `on`, `true`, `1` mean on; anything else means off. | Parse a number, round to an integer, clamp to the range. An absolute Set never wraps. |
| Up | Current + `stepSize`, clamped to 1. | Next choice, wrapping or clamping per `wrapSelector`. | Turn on. | Current + `counterStep`, clamped or wrapped per `counterWrap`. |
| Down | Current - `stepSize`, clamped to 0. | Previous choice. | Turn off. | Current - `counterStep`, clamped or wrapped. |
| Toggle | Not applicable. | Not applicable. | Flip. | Not applicable. |
| Follow | Not applicable. | Not applicable. | On while the input is active, off when it releases. No memory of an earlier state, so a release seen right after startup still switches off. | Not applicable. |

Selector Up or Down with no known current position steps to the first choice
(Up) or stays on index 0 (Down). A Selector with no configured choices logs a
warning and does nothing. A Counter with no known value steps from its
minimum, and an untouched `INTERNAL` Counter reports its minimum as a known
value from the start, so key faces and readouts never show blank after a
restart; a DSP-backed Counter stays unknown until the device reports.

**Step amount.** Up and Down accept an optional **amount** that replaces the
Control Value's own step for that one press, milestone, or script call. It is
carried on the trigger action (`controlValueStepAmount`), the timeline
milestone, and the script `up` and `down` calls. Meaning by kind: Level, a
0..1 delta, or a percent when its magnitude is above 1; Counter, an integer;
Selector, a number of choices. The amount is signed, so a negative amount
runs the other way. Clamp and wrap rules are unchanged. Stream Deck
press-and-hold repeat applies the amount on every repeat. A "touchdown" key
is Up with amount 6; a plain "+1" key leaves the amount empty.

### 3.2 Fades

A Level Set can carry a fade duration in milliseconds. The Core interpolates
from the current position to the target over that time and feeds each step
through the normal coalesced write path. External DSPs receive at most one
write per flush interval (section 3.3) during a fade. Any manual write, Up,
Down, or new Set on the same Control Value cancels an in-progress fade.
Fades apply only to Level. Up, Down, Toggle, and non-Level kinds ignore the
fade argument.

Which writers can request a fade:

| Writer | Fade |
|---|---|
| Timeline milestone | Yes. The milestone's duration is the fade time. |
| Script `set(code, value, fadeMs)` | Yes. |
| Web API level endpoint `fadeMs` query | Yes. |
| Trigger action, schedule action, custom-menu action | No. Applies instantly. |
| Integration API, MCP, OSC, plugin entity API | No. |

### 3.3 Timing and coalescing

| Constant | Value | Effect |
|---|---|---|
| Write interval | 50 ms per Control Value | Outbound Level writes to a backend are latest-value-wins. A fast encoder or slider drag produces at most 20 backend writes per second per value. |
| Publish interval | 100 ms per Control Value | Status broadcasts (SignalR, OSC echo, triggers, entity catalog) publish on the leading edge, then coalesce. The trailing value is published by the flush timer. |
| Unmute cooldown | 500 ms | With `unmuteOnLevelChange`, the unmute command is not re-sent more often than this while a drag is in progress. |
| Entity catalog coalescing | 150 ms | State pushed to the Integration API, MCP, and plugin entity subscribers is coalesced separately at this interval. |

Consequences for integrators:

- Do not expect to observe every intermediate value of a fast ramp. You will
  see the first value promptly and then samples at roughly 10 Hz.
- Up and Down presses are applied optimistically to the local value so rapid
  presses accumulate even before the backend echoes. Backend push feedback
  reconciles afterward.

### 3.4 Origins and the "not from me" rule

Every write carries an **origin tag**. The published status carries the set of
all origins that wrote since the last publish. Consumers that both write and
observe a Control Value filter on "did anyone other than me write this" so
they never react to their own write-back. This is the rule that lets several
bidirectional peers share one value without feedback loops.

| Origin tag | Writer |
|---|---|
| `DSP` | Backend push feedback: the external device itself moved the control. |
| `WEB` | Admin web UI, touchscreen custom menus, the Faders page, the web API level endpoint. |
| `OSC` | The `/dmxcore/control/{code}` OSC address. |
| `SCRIPT` | A user script. |
| `TIMELINE` | A Control Value timeline milestone. |
| `SCHEDULE` | A schedule action. |
| `INTEGRATION` | Integration API, MCP tools, and plugin entity or playback APIs. |
| `SURFACE:{id}` | A control surface (MIDI, Stream Deck, OSC surface, keypad), by surface id. |
| `TRIGGER:{id}` | An input trigger's action or value target, by trigger id. |
| `DRIVE:{code}` | Write-back from a Control Value's drive target (section 4.4). |

The internal backend does **not** echo writes back as `DSP`. Its status is
the optimistic local publish tagged with the writer's origin. Plugin backends
that echo every write should expect the Core to treat the echo as `DSP`; the
plugin should suppress echoes of its own writes if the protocol reflects
them.

### 3.5 Startup and backend availability

- Control Values are loaded once at startup and reloaded automatically whenever
  the definitions change in the admin UI.
- Plugins load late in startup. A Control Value whose backend is not yet
  registered is held and bound as soon as the backend appears. A Control
  Value that references a backend name no plugin ever registers logs a warning
  and stays inert: writes are ignored, `hasValue` stays false.
- When a plugin backend unregisters (plugin stopped or reloading), the binding
  is dropped and re-established on the next registration.
- On reload, the last-known value is preserved when the binding (backend and
  status id) did not change.
- Backend push subscriptions are registered on the **status** ids, not the
  write ids. Writes always go to `controlId`.

### 3.6 Skip-if-already-there

For Selector and Toggle Sets against a backend with push feedback, the Core
skips the write when the **device-reported** value already equals the
target. This avoids re-switching a source (which can click relays). The
comparison uses the device value, not the optimistic local one, so a write
that never landed is retried. An unknown device value always sends.

---

## 4. Surfaces that read or write Control Values inside the Core

### 4.1 Trigger actions

Any place that holds a trigger action can target a Control Value: control
surface keys, input trigger actions, custom-menu action items, and schedules.

| Action field | Value |
|---|---|
| Action type | `ControlValue` |
| Play code | The Control Value code |
| Control Value operation | `Set`, `Up`, `Down`, `Toggle`, `Follow` |
| Control Value set value | For Set only: a level, a choice name or index, on/off, or an integer |
| Control Value step amount | For Up and Down only, optional: replaces the Control Value's own step (section 3.1) |

No fade. The action's fade-in and fade-out fields do not apply to Control
Value actions.

**Press-and-hold auto-repeat.** On Stream Deck (USB and network) and OSC
surfaces, a key bound to a Control Value Up or Down action fires once on press
and, after a 400 ms hold, repeats every 150 ms until release. A quick tap
yields exactly one step. This is the only action type that auto-repeats. MIDI
and touchscreen buttons do not repeat.

**Stream Deck readout.** A Stream Deck key whose action targets a Control
Value shows the live value on its face (Level as percent, Selector as choice,
Toggle as On/Off, Counter as its number), with a per-key show-value switch and
an optional format string. A Stream Deck+ dial can drive a Level, or with the
step-per-tick target step any kind (a Counter counts, a Selector cycles), and
the LCD strip shows one segment per dial with its label and value. See the
Custom Menus and Control Surfaces document, section 4.4.

**Follow.** A `Follow` action on a Toggle kind mirrors a momentary input:
on at the rising edge, off at the falling edge, with no memory of what the
value was before. It needs a source with a release edge: a digital input
(threshold 0 reverses it), a control-surface key, an OSC button. Surfaces
track its release the way they track a Flash preset, and a digital-input
trigger with a Follow action applies the contact's present state when the
trigger is saved or the device starts. Control Value actions have no press
mode of their own; Follow replaces the earlier Flash-mode Set.

**Custom-menu button highlight.** A custom-menu action item bound to a
Control Value Set, Toggle, or Follow highlights when the live value matches
(Selector: choice name, Toggle and Follow: on). Up and Down items never
highlight. Level items never highlight.

### 4.2 Continuous actions (knobs and sliders)

A knob, fader, or slider carries a continuous action with target
`ControlValueLevel` and the Control Value code. The incoming value (MIDI CC
0–127, OSC float 0..1, value-mode trigger 0..1) is normalized, optionally
inverted, then scaled into the action's `minValue` to `maxValue` range before
being written as a Level Set. Only Level Control Values can be a continuous
target. Soft takeover and encoder accumulators seed from the Control Value's
last-known level so a physical knob picks up at the device's actual position.

Where continuous actions appear: control surface knob assignments, custom-menu
`Slider` items, and input triggers in Value mode.

### 4.3 Custom menus (touchscreen and web)

| Item type | Behavior |
|---|---|
| `Action` with a Control Value action | Button. Highlights on live match (section 4.1). |
| `Slider` with a `ControlValueLevel` continuous action | Inline fader for a Level. |
| `Segmented` with a Control Value code | Row of buttons for a Selector's choices, live-active one highlighted. |

A `ValueDisplay` item shows any kind read-only, with an optional format
string, and follows the value live (Custom Menus and Control Surfaces
document, section 2.2).

### 4.4 Drives: a Level Control Value driving a lighting target

A Level Control Value can be given a continuous action in its `drives` field.
The target is any continuous target: master dimmer, a fixture intensity, a
zone intensity, a fixture RGB channel, the audio volume, or another Level
Control Value. The binding is bidirectional:

- **Forward.** Every published status of the Control Value is applied to the
  target through the same path a knob uses, with origin `DRIVE:{code}`.
- **Reverse.** When the fixture state changes for any other reason (a cue, a
  preset, a timeline, a fader in direct mode), the driven target is read back
  through the inverse scaling. If it moved and no longer matches, the Control
  Value follows, also tagged `DRIVE:{code}`, so an external DSP knob or the
  internal value is never stale.
- **Boot safety.** Each direction arms on its first observation and never
  fires from it, so restored state on one side cannot push the other before
  something actually moves.
- **Quantization.** Reads are quantized to 1/127. A difference smaller than
  three quarters of one step is treated as "the same value", not a move.

The binding is 1:1. One Control Value drives one target. The runtime could
support 1:N but the field does not (section 7).

### 4.5 Faders page

The Faders page in the web UI has a Controls view that shows Level Control
Values as fader strips and writes with origin `WEB`. Internal Control Values
with a `drives` binding behave as named handles on the fixtures they drive.

---

## 5. External surfaces

### 5.1 Admin web API

All under the device's admin site, authenticated with the normal JWT or an API
key. Responses use the site's standard envelope (`ListResponse`,
`ItemResponse`, `SaveResponse`, `EmptyResponse`). These are admin endpoints
intended for the admin SPA; prefer the Integration API (section 5.2) for
third-party integrations.

| Method and path | Purpose |
|---|---|
| `GET /api/website/controlvalue/list` | Definitions summary: id, code, name, enabled, kind, description text, drives target and code. |
| `GET /api/website/controlvalue/statuses` | Live status of all enabled Control Values (section 2.2). |
| `PUT /api/website/controlvalue/level?code=X&value=0.5&fadeMs=1000` | Write a Level. `fadeMs` optional. Origin `WEB`. Any authenticated user. |
| `GET /api/website/controlvalue/backends` | Names of currently registered backends. |
| `GET /api/website/controlvalue/{id}` | One definition. |
| `POST /api/website/controlvalue/{id}` | Create (id 0) or update a definition. Requires the Edit Control Value permission. |
| `DELETE /api/website/controlvalue/{id}` | Delete. Requires the Delete Control Value permission. |

Live updates arrive on the SignalR hub as the `ControlValueStatus` message
carrying one status record (section 2.2), coalesced per section 3.3.

### 5.2 Integration API and MCP

Every enabled Control Value is an entity in the catalog with code
`cv.{CODE}`. See the Integration API documentation for transport, auth, and
rate limits.

| Control Value kind | Entity kind | State field | Commands |
|---|---|---|---|
| Level | `level` | `level` 0..1 | `setLevel` |
| Toggle | `switch` | `isOn` | `turnOn`, `turnOff`, `toggle` |
| Selector | `select` | `choice` (canonical choice name); catalog carries `choices` | `setChoice` |
| Counter | `number` | `number` (integer); catalog carries `min`, `max`, `step` | `setNumber` (rounded and clamped to the range) |

Writes carry origin `INTEGRATION`. There is no Up, Down, or fade over this
surface. Compute the next value client-side and `setLevel`, `setChoice`, or
`setNumber`.
State changes are pushed over the Integration API WebSocket, coalesced at
150 ms. The MCP server exposes the same entities through its tools.

### 5.3 OSC

| Address | Direction | Payload |
|---|---|---|
| `/dmxcore/control/{code}` | In | Float 0..1. Sets a **Level** Control Value directly with origin `OSC`. No surface or trigger configuration needed. |
| `/dmxcore/control/{code}` | Out | Float 0..1. Echoed to connected OSC clients on every Level status publish, so a TouchOSC fader tracks. The code is lowercased in the outgoing address. |

Selector, Toggle, and Counter are not addressable through this shortcut and
are not echoed. Use an OSC control surface or an OSC input trigger with a
Control Value action.

### 5.4 Scripts

Scripts get a `dmx.controlValue` object.

| Call | Behavior |
|---|---|
| `get(code)` | Returns the 0..1 value, or null when unknown or not yet reported. For a Counter it returns the integer, not the position. For Selector this is the normalized raw value; use `status()` for the choice. |
| `set(code, value, fadeMs = 0)` | Boolean → Toggle on/off. Number or string → Set (section 3.1). `fadeMs` > 0 ramps a Level. |
| `up(code, amount?)`, `down(code, amount?)` | Step, optionally by `amount` (section 3.1). |
| `toggle(code)` | Toggle only. |
| `status(code)` | `{ kind, value, choiceIndex, choiceName, number, muted }`, or null for an unknown code. `value` is null until known. |

All script writes carry origin `SCRIPT`.

### 5.5 Control Value input triggers

An input trigger of type `ControlValue` observes one Control Value and fires
its action. The trigger's `address` field holds the code.

**Edge mode** (the default trigger modes):

| Kind | Fires when | Re-arms when |
|---|---|---|
| Level | Value rises to or above `threshold` percent (0–100). | Value drops below the threshold. |
| Selector | The choice named in `startPayload` (name or index) becomes active. | The choice in `stopPayload` becomes active, or, when `stopPayload` is empty, any other choice. |
| Toggle | Treated as a Level with the threshold applied to 0 or 1. | Same. |
| Counter | The threshold percent is applied to the integer's position within its range: fires when the value reaches or passes that fraction of min..max. | Drops below. |

**Value mode:** the trigger passes the live 0..1 value to its continuous
action on every foreign update. This is how a DSP fader drives the master
dimmer or a zone.

Rules that matter:

- The first known value **arms without firing**. Server boot or a DSP
  reconnect never replays edges. On load the trigger arms from the Control
  Value's present value, so the first change after a save or a restart is a
  real edge (before Core `90e4533b` it only armed, and the second change
  fired); a `Follow` action is applied to that present state at once.
- The trigger fires on updates from **any origin except its own**
  (`TRIGGER:{id}`). It does fire on writes from scripts, timelines, the web
  UI, and other triggers, not only on DSP-side changes.
- Only rising edges run the action. The falling edge is reported to the
  trigger's state display but runs no action (section 7).
- A Value-mode trigger whose continuous action targets the **same** Control
  Value it observes is rejected at startup with a warning, because it would
  loop.

### 5.6 Timelines

A timeline milestone of type `ControlValue` carries the code, the operation,
the set value, and the optional step amount, exactly like a trigger action.
The milestone's **duration** is the fade time for a Level Set, so "fade to
40 % over 1.5 s" is one milestone. Origin `TIMELINE`. Milestones are placed on tracks like any other
event.

### 5.7 Schedules

A schedule's action is a trigger action, so a schedule can Set, Up, Down, or
Toggle a Control Value at a time of day. Origin `SCHEDULE`. No fade.

---

## 6. Plugin SDK

Two independent roles are available to a plugin.

### 6.1 Being a backend (owning the value)

Register with `host.ControlValues.RegisterBackend(name, backend)`. The name
is uppercase by convention and is what operators pick in the Control Value
editor's backend list. Dispose the returned handle to unregister.

The plugin implements:

| Member | Called by the Core |
|---|---|
| `TrueValue`, `FalseValue` | Raw strings meaning on and off for toggle-style controls, protocol dependent (for example `"1"` and `"0"`, or `"65535"` and `"0"`). |
| `SetLevelAsync(controlId, level 0..1)` | A Level write. |
| `SetRawValueAsync(controlId, rawString)` | A Selector choice value, a Toggle value, or a Counter integer. A Counter's raw value **is** the integer, with no 0..65535 scaling; push it back unchanged. |
| `RegisterControlIdsAsync(controlIds)` | The complete set of **status** ids that need live feedback. Replaces any previous set. Subscribe them on the external device. |
| `GetControlData(controlId)` | Synchronous. Return cached last-known data or null. Do not query the device here. |

The plugin pushes changes back with `handle.PushControlData(controlId, data)`
where data is `(Value, Position, StringValue)`: the raw device value, the
normalized 0..1 position, and an optional raw string. Push on every change
received from the device, after the plugin's own echo suppression. Pushes
arrive in the Core tagged `DSP`.

Host-to-backend calls are dispatched serially per plugin and fault-isolated.
A slow or throwing backend does not stall other plugins or the Core.

Full-scale convention: the Core assumes a raw range of 0..65535 for a
full-scale level when it must synthesize a raw value, matching the Symetrix
convention. Backends with other ranges should compute `Position` themselves.
For a Counter the Core ignores the pushed `Position` and derives it from the
integer and the Counter's own range.

### 6.2 Observing or writing a Control Value the plugin does not own

Use the entity API. Enabled Control Values appear as `cv.{CODE}` entities
with the kinds in section 5.2. Subscribe to state changes (coalesced at
150 ms) and execute `setLevel`, `turnOn`, `turnOff`, `toggle`, `setChoice`,
or `setNumber`. The `number` entity kind, its `min`, `max`, `step`, and the
`setNumber` command require SDK contract 1.12. Writes carry origin
`INTEGRATION`.

A plugin that needs a value it can both read and write, with no external
device, should have the operator create an **internal** Control Value and
then use the entity API. Plugins cannot create Control Value definitions
themselves.

---

## 7. Limits and gaps

Things a Control Value cannot do today. When a design needs one of these,
request it from DMX Core rather than working around it. Numbers such as #131
are DMX Core's internal issue tracker references; the tracker is private, so
quote the number when you talk to DMX Core and read
[issues.md](../issues.md) for what each one asks for.

| Gap | Detail | Status |
|---|---|---|
| Home Assistant `number` entity for Counters | The Core exposes Counters as `number` entities since SDK 1.12; the Home Assistant plugin has to be updated to map them. Until then a Counter is not visible in Home Assistant. | Follow-up to #131, plugin side |
| Up, Down, or a step amount over the Integration API | Only absolute `setNumber`. Compute the next value client-side. | Not planned |
| Serial port ownership for plugins | Not a Control Value gap, but it blocks a common companion design: a plugin that drives a serial display from a Control Value. The Core probes every serial port at startup unless probing is disabled, and there is no claim registry. | Open, #134 |
| Fire actions on the falling edge | A Control Value input trigger runs its action on the rising edge only; falling edges update the state display. The `Follow` operation covers the common case of mirroring a momentary input onto a Toggle. | Not planned (framework-wide) |
| Fade on trigger, schedule, Integration API, OSC, plugin writes | Fade is available from timelines, scripts, and the web level endpoint only. | Not planned |
| Up, Down, or fade over the Integration API | Only `setLevel`, `setChoice`, and switch commands. | Not planned |
| OSC shortcut for Selector or Toggle | `/dmxcore/control/{code}` is Level only. | Not planned |
| One Control Value driving several targets | `drives` is 1:1. | Not planned until needed |
| Auto-repeat on MIDI or touchscreen buttons | Press-and-hold repeat is Stream Deck and OSC only. | Deferred |
| MIDI CC output feedback | Motorized faders and LED rings do not receive the value. OSC echo and drive write-back are the only feedback paths. | Deferred |
| Per-device backend instances | One backend registration per name. A plugin that talks to several devices must namespace its control ids. | Not planned |
| dB display or dB scaling | Levels are fractions and percent only. | Dropped |
| Text or floating-point kinds | Counter is integer only; there is no free-text value. Use a Selector for a small fixed vocabulary. | Not planned |
| Persistence of plugin-backend values across restart | The Core relies on the backend to report. | By design |
| Write-on-change persistence for internal values | Internal values are saved with the periodic system-state save. | By design |

---

## 8. Worked example: shared house volume with a wall fader and a Stream Deck

Goal: a Symetrix zone volume that a wall-mounted OSC fader, a Stream Deck
key pair, and the touchscreen all control, and that dims the bar lights when
it goes above 80 %.

1. **Definition.** Create a Level Control Value `VOL1`, backend `SYMETRIX`,
   `controlId` = the Composer controller number, `stepSize` 0.05,
   `muteControlId` = the zone mute controller, `unmuteOnLevelChange` on.
2. **Wall fader.** Point the OSC fader at `/dmxcore/control/vol1` with a
   float 0..1. Nothing else to configure. The Core echoes the level back to the
   fader on every publish.
3. **Stream Deck.** Two keys with trigger actions: type ControlValue, play
   code `VOL1`, operations Up and Down. Holding a key repeats after 400 ms.
4. **Touchscreen.** A custom-menu `Slider` item with a `ControlValueLevel`
   continuous action targeting `VOL1`.
5. **Lighting reaction.** An input trigger of type ControlValue, address
   `VOL1`, threshold 80, action: activate preset `BAR_DIM`. It fires when the
   level crosses 80 % from any writer, and re-arms when it drops below.
6. **Turn-down at closing.** A schedule at 01:00 whose action is ControlValue
   Set `VOL1` to `20%`. For a slow fade instead, use a timeline with a
   ControlValue milestone of duration 30000 ms and start that timeline from
   the schedule.

What you will observe: every writer's change shows on every other surface
within about 100 ms. When someone turns the volume on the Symetrix side, the
Core receives the push, publishes with origin `DSP`, and the OSC fader,
Stream Deck state, touchscreen slider, and the trigger all follow.

---

## 9. Worked example: internal state for automation

Goal: a "game mode" flag that scripts, a Stream Deck key, and Home Assistant
can all flip, with a light show on entry.

1. Create a Toggle Control Value `GAME`, backend `INTERNAL`, `controlId`
   `game` (any string).
2. Stream Deck key: ControlValue action, code `GAME`, operation Toggle. The
   key highlights while on.
3. Home Assistant (via the Home Assistant plugin's entity subscription) sees
   `cv.GAME` as a switch and can turn it on or off with origin `INTEGRATION`.
4. Input trigger of type ControlValue, address `GAME`, threshold 50, action:
   play timeline `GAME_INTRO`. It fires whichever surface turned the flag on,
   and does not fire on boot even if the restored value is on.
5. A script can read it with `dmx.controlValue.get("GAME")` and branch.

The value survives a normal restart through the system-state save.

---

## 10. Worked example: a scoreboard

Goal: home and away scores that Stream Deck keys step, that a timeline can
bump, and that a display plugin mirrors.

1. Two Counter Control Values `HOME_SCORE` and `AWAY_SCORE`, backend
   `INTERNAL`, range 0..99, step 1, wrap off.
2. Stream Deck keys: ControlValue Up and Down with no amount for the ±1
   keys, and Up with amount 6, 3, and 2 for touchdown, field goal, and safety.
   Holding a ±1 key auto-repeats.
3. A "touchdown show" timeline with a ControlValue milestone at 0 s: code
   `HOME_SCORE`, operation Up, amount 6, so the show and the score change
   together.
4. A Selector `TICKER_MESSAGE` with choices `SCORE`, `TOUCHDOWN`, `BLANK`,
   set from milestones in the same timeline (`TOUCHDOWN` at 0 s, back to
   `SCORE` at 8 s).
5. A display plugin subscribes to `cv.HOME_SCORE`, `cv.AWAY_SCORE`, and
   `cv.TICKER_MESSAGE` through the entity API (SDK 1.12) and renders them,
   or registers itself as the backend so it receives the integer verbatim on
   every write with no coalescing.
6. Home Assistant and the Integration API see the scores as `number`
   entities with `min` 0, `max` 99, `step` 1, and can `setNumber` to reset
   them at the start of a game.

The live score is visible on every Stream Deck key that acts on the Counter
(the key face shows the number and a Set key lights while it matches), on the
Stream Deck+ LCD strip when a dial steps it, on the touchscreen through a
custom-menu item (a `ValueDisplay` item prints the number with an optional
format), and in the admin UI.
