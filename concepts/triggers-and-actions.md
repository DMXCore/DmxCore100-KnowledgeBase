# Triggers and Actions

**Audience:** integrators, plugin authors, and AI agents planning an
integration without access to the DMX Core 100 source code.
**Verified against:** DMX Core 100 software `main` at commit `cc6ae738`
(2026-09-16), Plugin SDK contract 1.11.
**User-facing documentation:**
<https://docs.dmxcore.com/dmx-core-100/scheduling-automation/input-triggers>,
<https://docs.dmxcore.com/dmx-core-100/scheduling-automation/output-events>,
<https://docs.dmxcore.com/dmx-core-100/scheduling-automation/schedules>,
<https://docs.dmxcore.com/dmx-core-100/scheduling-automation/custom-menus>,
<https://docs.dmxcore.com/dmx-core-100/control-surfaces>

This document describes the path from "something happened" to "the device
did something": the one action model every surface shares, how each kind of
input turns into a rising or falling edge, what each action type does, the
rules applied on the way, and what the path cannot do today.

---

## 1. Mental model

Almost everything a DMX Core 100 does on demand goes through one shared
**trigger action**: a small record that says *which action type*, *which
target code*, *which mode*, and a few playback parameters. The same record
is embedded in an input trigger, a control-surface key, a custom-menu
button, a schedule, and is built on the fly by scripts and the Integration
API. One dispatcher executes it.

The sources that produce a trigger action:

| Source | What produces the edge | Origin tag |
|---|---|---|
| Input trigger | A network, DMX, hardware, Control Value, or plugin event matched by the trigger's rules (section 3) | `TRIGGER:{id}` |
| Control surface | A key, pad, or button on a MIDI, OSC, Stream Deck, or keypad surface, or a tap in the web operator view | `SURFACE:{id}` |
| Custom menu | A tap on an Action item on the touchscreen or the web custom menu | `WEB` |
| Schedule | A start time reached (section 6) | `SCHEDULE` |
| Script | A call such as `dmx.playCue` (section 7) | `SCRIPT` |
| Integration API, MCP, plugin entity API | An entity command such as `activate` (section 8) | `INTEGRATION` |
| Plugin | `host.Triggers.FireAsync(code)`, which fires a Plugin-type input trigger | `TRIGGER:{id}` |

Two things are deliberately **not** trigger actions:

- **Timelines.** A timeline is a sequence of milestones with its own event
  types. A trigger action can *play* a timeline. When several things must
  happen at once or in sequence, the intended design is one trigger that
  plays one timeline.
- **Continuous values.** A knob, fader, or value-mode input drives a
  *continuous action* (a target and a scaling range), not a trigger action.
  Section 3.3 covers value mode.

An input trigger holds exactly **one** action. There is no action list.

---

## 2. The trigger action record

Field names below are the JSON names used by the admin web API and inside
custom-menu definitions.

| Field | Type | Meaning |
|---|---|---|
| `type` | string enum (section 4) | What to do. |
| `playCode` | string | The target's code: cue, preset, sound, timeline, output event, schedule, Control Value, script, or effect. Matched case-insensitively. |
| `mode` | `NORMAL`, `TOGGLE`, `FLASH`, `MOMENTARY` | How press and release are interpreted (section 2.1). |
| `fadeInDurationMS` | int, default 0 | Fade in for playing items. For a Preset or Ambient Preset this is the single fade. 0 means immediate. |
| `fadeOutDurationMS` | int, default 0 | Fade out when the item ends or is stopped. For the FadeOut action it is the fade itself. In Flash mode it is the release fade. |
| `dimmerScale` | double 0..1, default 1 | Intensity scale applied to a played cue. |
| `volume` | double 0..1, default 1 | Volume scale applied to a played sound or timeline. |
| `priority` | byte, default 100 | Schedule ordering (section 6) and downstream playback priority. |
| `loop` | int, default 1 | 0 = loop forever, 1 = once, N = N times. Cue, Sound, Timeline. |
| `runToCompletion` | bool | Schedules only: let the item finish its current pass when the schedule ends instead of stopping it. |
| `afterTouchDimmer` | bool | Flash mode on MIDI only: channel pressure scales the preset dimmer while held. |
| `targetControlSurfaceId`, `targetBankIndex` | int, nullable | SwitchControlSurfaceBank and NextControlSurfaceBank. Null surface means "the surface the key is on". |
| `controlValueOperation`, `controlValueSetValue` | see Control Values | ControlValue actions. |
| `targetState` | `TOGGLE`, `ON`, `OFF` | Mute, OutputToggle, Blackout: flip or force. |

There is **no system default fallback** for any of these. A 0 fade is no
fade. The system-wide default fade and loop settings apply only to plays
started from the admin UI lists.

### 2.1 Modes

| Mode | Rising edge (press) | Falling edge (release) | Applies to |
|---|---|---|---|
| Normal | Run the action. | Ignored. | All types |
| Toggle | Cue, Sound: stop it if it is already playing on its layer, else play. Preset: fade it out if it is active, else fade to it. Ambient Preset: clear it if active, else apply. Control Value: not affected by mode. | Ignored. | Cue, Preset, Sound, AmbientPreset |
| Flash | Fade to the preset over `fadeInDurationMS`. | Fade the preset out over `fadeOutDurationMS`. | Preset only, on control surfaces and custom menus with a release edge |
| Momentary | Play the timeline. If the timeline is already parked at an unnamed Hold milestone, release it instead of restarting. | Release an unnamed Hold milestone (latching if the hold has not been reached yet). Holds that name a specific release trigger are not affected. | Timeline only |

Only Flash and Momentary do anything on release. Every other action ignores
the falling edge. Sources that have no release edge (HTTP triggers with no
value, a schedule, a script call, the Integration API) can only produce
Normal and Toggle behavior.

Flash on a surface runs one flash cycle at a time per key. Re-presses while
a cycle is in flight are dropped.

---

## 3. Input triggers

An input trigger binds one input source to one action. Fields: `code` (the
trigger's own identifier, used by timeline holds and plugins), `type`,
`mode` (Edge or Value), `address`, `port`, `startPayload`, `stopPayload`,
`universeId`, `channel`, `threshold`, `dmxTriggerMode`, plus the value-mode
fields in section 3.3. Disabled triggers are not bound at all.

### 3.1 Edge mode

The input produces **rising** and **falling** edges. A rising edge runs the
action (section 5). A falling edge only reaches Flash and Momentary actions.
The trigger list shows the last received value and whether the trigger is
currently in its triggered state.

### 3.2 Per-type matching rules

| Type | Binding | Rising edge | Falling edge | Notes |
|---|---|---|---|---|
| UDP Listener | `port` | Datagram **starts with** `startPayload` bytes | Datagram starts with `stopPayload` bytes | Several triggers may share a port. If the port cannot be bound, a configuration warning is raised and the trigger is inert. |
| TCP Listener | `port` | `startPayload` bytes appear **anywhere** in the incoming stream | Same for `stopPayload` | Each client connection is scanned independently. Longest pattern wins when several overlap. |
| TCP Connector | `address` + `port` | Same stream scan as TCP Listener | Same | The device **connects out** to the remote host and reconnects with backoff. Triggers with the same host and port share one connection. |
| HTTP | `address` (URL path, must start with `/`) | Request with no value, or `?v=` / JSON body parsing as true | `?v=` / body parsing as false | Section 3.4. |
| OSC | `address` (lowercased, must start with `/`) | Argument text equals `startPayload` | Argument text equals `stopPayload` | With no payloads configured: argument `1` (or no argument) is rising, anything else is falling. `[payload]` in the address is replaced by each payload to form two full addresses. |
| MQTT | `address` = topic, exact match, lowercased | Payload text equals `startPayload` | Payload text equals `stopPayload` | With no payloads configured: payload parsed as a boolean (`1`, `true`, `on`, ...); unparseable means rising. `[payload]` works as for OSC. The broker is a system setting. Topic wildcards are not supported in the address. |
| Art-Net, sACN, DMX Serial | `universeId`, `channel` 1–512, `threshold` 0–255, `dmxTriggerMode` | Channel value goes **above** the threshold | Channel value goes to or below the threshold | `AboveThreshold`: evaluate from the first frame. `ZeroThenAboveThreshold`: ignore everything until the channel has been seen at 0 once, then behave as AboveThreshold. Prevents a trigger firing on the first frame of a console that is already up. sACN joins the universe's multicast group. |
| Digital Input | `universeId` = module input 1–4, `threshold` = polarity | Input goes active (`threshold` 1, the default) or inactive (`threshold` 0, inverted) | The opposite transition | Both edges are reported, deduplicated by the triggered state, so Flash and Momentary follow the contact. Before #139 (fixed 2026-09-16) a trigger saw one edge direction only. |
| Control Value | `address` = Control Value code, `threshold` percent, `startPayload` / `stopPayload` choice | See the Control Values document | | Fires on any origin except the trigger's own. First value arms without firing. |
| Plugin | `code`, optional `address` = plugin id | The plugin calls `FireAsync(code)` | Never | With `address` set only that plugin can fire it. Unknown or disabled codes are ignored. No payload travels with a plugin fire. |

Payload syntax for UDP and TCP (`startPayload`, `stopPayload`):

- `"text"` in double quotes: UTF-8 bytes of the text, with `\r`, `\n`,
  `\t`, `\\` escapes.
- Comma or space separated hex bytes, for example `02 41 0d`.
- Anything else: UTF-8 bytes of the text as written, with the same escapes.

OSC and MQTT payloads are compared as text against the incoming argument or
payload, exactly and case-sensitively.

### 3.3 Value mode

In value mode the payload **is** the value and the trigger drives a
continuous action instead of running a trigger action.

Pipeline, in order:

1. Extract a number. With `inputJsonPath` set (for example `$.volume`), the
   payload is parsed as JSON and the path is applied first. A payload that
   does not yield a number is ignored.
2. Normalize: `(raw - inputMin) / (inputMax - inputMin)`, clamped to 0..1.
   Defaults map 0..1 to 0..1.
3. Optional transform script (`valueTransformScriptCode`): receives the
   normalized value and the raw payload, returns a new 0..1 value. A failing
   transform **drops** the update rather than passing the raw value through.
4. Apply to the continuous action target with the action's own min, max,
   and invert scaling, tagged `TRIGGER:{id}`. Targets: master dimmer, a
   fixture intensity or RGB channel, a zone intensity, audio volume, a Level
   Control Value.

A value trigger is authoritative: there is no soft takeover.

Value mode is available for HTTP, OSC, MQTT, and Control Value triggers.
UDP, TCP, DMX, Digital Input, and Plugin triggers are edge only.

### 3.4 HTTP trigger URLs

An HTTP trigger registers its `address` as a URL path on the device's web
server, for example `http://<device>/party`. Rules:

- Any method. `GET` with no value counts as a rising edge.
- Edge value: `?v=1`, `?v=0`, `?v=true`, `?v=false`, `?v=on`, `?v=off`, or a
  POST body that is JSON `true`, `false`, `1`, `0`, or a quoted string with
  those words. An unparseable value returns `422`.
- Value mode: the `?v=` query if present, else the raw POST body, handed to
  the pipeline in section 3.3.
- Response is `200` with a short text body once every trigger on that path
  has been dispatched.
- The path is matched after all built-in routes, so it cannot shadow the
  admin API. It is matched case-insensitively.
- **No authentication.** Anyone who can reach the device's HTTP port can
  fire the trigger. Treat trigger paths as capability URLs and keep the
  device on a trusted network.

### 3.5 Reload behavior

Saving any input trigger rebuilds **all** listeners: every UDP and TCP port
is re-bound, every OSC, MQTT, HTTP, DMX, Control Value, and plugin binding
is re-registered, and outbound TCP connections are re-established. Expect a
brief window during which events are missed. Ports no longer referenced are
released.

---

## 4. Action types

| Type | Target (`playCode`) | What happens | Uses |
|---|---|---|---|
| `None` | | Nothing. Placeholder for an unassigned key. | |
| `Cue` | Cue code | Play the recorded DMX cue. | loop, fades, dimmerScale, Toggle |
| `Preset` | Preset code | Fade fixtures to the preset's state. A state transition, not a playing item. | fadeIn as the single fade, Toggle, Flash |
| `Sound` | Sound code | Play the audio file. | loop, fades, volume, Toggle |
| `Timeline` | Timeline code | Play the timeline from the start. If parked at an unnamed Hold, release it instead. | Momentary |
| `OutputEvent` | Output event code | Send the output event (section 9). | |
| `StopPlayback` | | Stop all cues, sounds, timelines, and presets. | |
| `FadeOut` | | Fade out the current cue and sound over `fadeOutDurationMS`. | fadeOut |
| `Blackout` | | Latched output mask: playback keeps running underneath, output is dark until Blackout Off. | targetState |
| `Mute` | | Audio master mute. | targetState |
| `OutputToggle` | | Output On or Off. Off is a cold park: nothing that plays may start while it is on. | targetState |
| `ScheduleToggle` | Schedule code | Flip the schedule's enabled flag. Persisted. | |
| `AmbientPreset` | Preset code marked ambient | Apply the ambient preset as the persistent fallback state. | fadeIn, Toggle |
| `SwitchControlSurfaceBank` | | Activate a bank on a surface. | targetControlSurfaceId, targetBankIndex |
| `NextControlSurfaceBank` | | Advance the surface's active bank. | targetControlSurfaceId |
| `TapTempo` | | One tap on the system metronome. | |
| `ControlValue` | Control Value code | Set, Up, Down, or Toggle the Control Value. See the Control Values document. | controlValueOperation, controlValueSetValue |
| `Script` | Script code | Queue the script. Dispatch does not wait for it. | |
| `EffectStep` | Effect code, or `*` / empty | Advance one step of an effect in external-trigger sync mode. `*` steps every active such effect. | |
| `StopOutput` | | Legacy. Behaves as Blackout. Existing definitions are upgraded. | |

A target code that does not exist logs a warning and does nothing. The
caller is not told.

---

## 5. Dispatch rules

These apply to every trigger action regardless of source, in this order:

1. **Output Off gate.** While output is off, `Cue`, `Timeline`, `Sound`, and
   `OutputEvent` actions are dropped. State actions (`Preset`,
   `AmbientPreset`, `ControlValue`, `EffectStep`, `Blackout`, `Mute`) still
   run, and `OutputToggle` always runs so a scheduled or triggered Output On
   works.
2. **Recorder gate** (input triggers only). While the DMX recorder is
   active, input-trigger actions do not run. The edge still reaches the
   recorder's own start/stop trigger and timeline holds.
3. **Recorder trigger.** The input trigger selected in the recording
   settings starts the recorder on its rising edge and stops it on its
   falling edge.
4. **Timeline holds.** Every rising edge of an input trigger is offered, by
   trigger code, to timelines parked at a Hold milestone that names that
   trigger as its release.
5. **Execution.** The action runs. Cue, Sound, Timeline, and Preset plays
   are handed to the playback engine and return once started, not once
   finished. Script actions are queued and return immediately.
6. **Origin.** The action carries its origin tag into Control Value writes
   and into the script context (`ctx.source` is the input type such as
   `UDP` or `MQTT`, `ctx.triggerCode` the trigger's code, `ctx.payload` the
   raw payload where the source has one).

Actions from one input are processed as they arrive. There is no
debouncing: a source that sends the same rising edge twice runs the action
twice, except that DMX triggers only report **changes** of the triggered
state.

---

## 6. Schedules

A schedule owns a trigger action and adds time. Fields: `startTime`,
`endTime` (optional), `startTimeMode` and `endTimeMode` (`FIXED`,
`SUNRISE`, `SUNSET`) with signed minute offsets, `startDate`, `endDate`
(optional), `days` (weekday set, empty means all), `enabled`.

Behavior:

- Evaluation is on whole minutes. Sun-based times need the device location
  set; a day whose sun event cannot be resolved is skipped. Sun times are
  clamped to the same local day.
- **Supported action types:** `Cue`, `Timeline`, `Preset`, `Sound`,
  `AmbientPreset`, `OutputToggle`, `Blackout`, `Script` (runs once at the
  start with source `SCHEDULE`; #143, shipped 2026-09-16). Any other type
  logs a warning and does nothing.
- **Start** plays the item with `fadeInDurationMS`, `loop`, `dimmerScale`,
  `volume`. Playing items are held as schedule-owned players.
- **End** stops the item over the action's `fadeOutDurationMS` (#140,
  shipped 2026-09-16): a cue, sound, or timeline fades out, a preset is
  released over it, an ambient preset is cleared over it. 0 stops hard
  (presets and ambient presets then use their default fade). With
  `runToCompletion` a cue, sound, or timeline finishes its current pass
  instead.
- **Priority.** Candidates are ordered by `priority`; among schedules due at
  the same minute the highest number runs. A running schedule ends early
  when a schedule with a higher priority number starts before its end time.
- Schedules do not run while the recorder is active, while the schedule is
  open in the editor, or during a snooze set from the UI.
- The `ScheduleFired` script event fires on every start, `ScheduleEnded` on
  every end (end time reached, or a higher priority schedule took over).

---

## 7. Scripts

Scripts build trigger actions through the `dmx` object. All calls carry
origin `SCRIPT` and go through the same dispatch rules.

| Call | Action built |
|---|---|
| `playCue(code, { fadeIn, fadeOut, loop, dimmer, toggle })` | Cue |
| `playSound(code, { fadeIn, fadeOut, loop, volume, toggle })` | Sound |
| `playTimeline(code)` | Timeline |
| `fadeToPreset(code, durationMs)` | Preset |
| `stopPlayback()` | StopPlayback |
| `fadeOut(durationMs)` | FadeOut |
| `fireOutputEvent(code)` | OutputEvent |
| `isPlaying(code)` | Query only |

Other script surfaces that are not trigger actions: `masterDimmer`,
`zoneDimmer`, `setFixture`, `getFixture`, `releaseFixture`,
`setZoneEffect`, `setGlobalEffect`, `sleep`, `sunrise`, `sunset`, `log`,
`controlValue.*`, `osc.send`, `mqtt.publish`, `store.*`, and
`schedule.enable / disable / toggle / isEnabled`.

A `Script` trigger action is the way to add conditions, counters, or
multi-step logic to a trigger: the trigger fires the script, the script
decides what to play.

---

## 8. Integration API, MCP, plugin entity API

The entity catalog maps commands onto trigger actions with origin
`INTEGRATION`:

| Entity | Command | Action built |
|---|---|---|
| `cue.X`, `timeline.X`, `sound.X` (kind `scene`) | `activate` with optional `loop`, `fadeInMs`, `fadeOutMs` | Cue / Timeline / Sound |
| `preset.X`, `ambient.X` (kind `switch`) | `turnOn`, `turnOff`, `toggle`, `activate` | Preset / AmbientPreset on turn-on; a direct stop with the system default fade on turn-off. A command that matches the current state is a no-op. |
| `schedule.X` (kind `switch`) | `turnOn`, `turnOff`, `toggle` | ScheduleToggle equivalent |
| `system.stop` (kind `button`) | `activate` | StopPlayback |
| `system.blackout`, `system.mute`, `system.outputmute` (kind `switch`) | `turnOn`, `turnOff`, `toggle` | Blackout / Mute / OutputToggle |
| `cv.X` | see Control Values | ControlValue |

There is no entity for firing an input trigger or an output event. A plugin
that wants "the venue decides" behavior fires a Plugin-type input trigger
instead (section 3.2).

---

## 9. Output events

An output event is the outbound counterpart: a named message the device
sends when an `OutputEvent` action runs, a timeline reaches an OutputEvent
milestone, a script calls `fireOutputEvent`, or the Test button is pressed.
Fields: `code`, `type`, `destination`, `address`, `payload`, `port`,
`universeId`, `pluginName`.

| Type | What is sent |
|---|---|
| UDP | One datagram to `destination`:`port` with `payload` as UTF-8 text, exactly as written. No hex or escape syntax. |
| TCP | `payload` as UTF-8 text on the **existing** connection that a TCP Connector input trigger holds to the same `destination` and `port`. Without such a trigger, nothing is sent. |
| OSC | To `destination` as `ip:port`, at `address`. A numeric payload is sent as float32, other text as a string, and `[payload]` in the address is substituted with the payload and sent without an argument. |
| HTTP | A **GET** to `address` when it is a full URL, or to `destination` + `address` path. No POST, body, or headers. |
| MQTT | Publish `payload` on topic `address` through the configured broker. |
| Digital Output | Pulse module output `universeId` active for one second. |
| Plugin | Invoke the output action provider registered by `pluginName` with target `address` and `payload`. |
| Art-Net, sACN, DMX Serial | **Not implemented.** The types exist in the editor but sending is a no-op (section 10). |

Delivery is fire and forget. A failure is logged and reported to the Test
button, never to the trigger that caused it.

---

## 10. Limits and gaps

Things the trigger path cannot do today. When a design needs one of these,
request it from DMX Core rather than working around it. Numbers such as
#131 are DMX Core's internal issue tracker references; the tracker is
private, so quote the number when you talk to DMX Core and read
[issues.md](../issues.md) for what each one asks for.

| Gap | Detail | Status |
|---|---|---|
| Several actions per trigger | One action per trigger, key, menu item, or schedule. Play a timeline for several things. | By design |
| Actions on the falling edge | Only Flash (preset) and Momentary (timeline) react to release. A "stop cue when the contact opens" needs a second trigger with the stop payload and a StopPlayback or FadeOut action. A Control Value action with operation `Follow` (Toggle kinds; #144, shipped 2026-09-16) is On while the input is active and Off on release, so a contact holds a Control Value On only while closed. | Not planned in general; Control Value Follow: closed, #144 |
| Conditions, counters, debouncing | No per-trigger conditions or rate limiting. Use a Script action. | Not planned |
| Authenticated HTTP triggers | Trigger URL paths accept any caller. Asks for a per-trigger token or an API-key requirement. | Open, #137 |
| UDP or TCP value mode | Only HTTP, OSC, MQTT, and Control Value carry a value. | Not planned |
| MQTT wildcards in trigger topics | Exact topic only. | Not planned |
| Regex or partial-field matching | UDP is prefix match, TCP is substring match, OSC and MQTT are exact text. No regex, no numeric comparison. Use value mode with a transform script for numeric thresholds. | Not planned |
| Output events over Art-Net, sACN, DMX Serial | Editor offers the types; nothing is sent. | Open, #138 |
| HTTP output event with POST, body, or headers | GET only. Use a Script action with `osc.send` or `mqtt.publish`, or a plugin. | Not planned |
| Hex or escaped bytes in UDP and TCP output payloads | Trigger payloads accept hex, output payloads are plain text. | Open, #141 |
| TCP output without a matching TCP Connector trigger | The output rides an input trigger's connection; without one nothing is sent and the Test button still reports success. | Open, #141 |
| Schedule fade-out at end | Fixed: `fadeOutDurationMS` applies when a schedule ends (section 6). | Closed, #140 |
| Schedule action types | Cue, Timeline, Preset, Sound, AmbientPreset, OutputToggle, Blackout only. Script is requested. | Partly open, #143 |
| Plugin trigger payload | `FireAsync(code)` carries no data; the script context payload is empty. | Not planned |
| Digital input release edge | Fixed: a digital-input trigger now reports both edges (section 3.2). Acting on "contact opened" with its own action still needs a second, inverted trigger (threshold 0). | Closed, #139 |
| Press-and-hold auto-repeat | Stream Deck and OSC surfaces only, Control Value Up/Down only. | Deferred for MIDI and keypads |
| Hold-to-confirm on the web operator view | A Yes/No dialog instead of a hold. | By design |
| Value-mode soft takeover | A value trigger jumps the target to the incoming value. | By design |

---

## 11. Worked example: game-event keys and a contact closure

Goal: a Stream Deck key plays a "touchdown" show, a door contact on the
serial module dims the room while open, and a bar POS system announces the
last call over UDP.

1. **Touchdown key.** On the Stream Deck surface, assign a key: action type
   Timeline, play code `TOUCHDOWN`, mode Normal. The timeline holds the cue,
   the sound, and a Control Value milestone that bumps the score. Enable
   hold-to-confirm on the key if staff should not fire it by brushing it.
2. **Door contact.** Input trigger, type Digital Input, `universeId` 1,
   `threshold` 1, action Preset `DOOR_OPEN` with mode Normal and
   `fadeInDurationMS` 500. The preset applies when the contact activates.
   To restore the look when the contact releases, use mode **Flash**
   instead: the digital input reports both edges (section 3.2), so the
   preset applies while the contact is active and releases when it opens.
   To run a *different* action when the contact opens, add a second Digital
   Input trigger on the same input with `threshold` 0 (inverted).
3. **Last call.** Input trigger, type UDP Listener, `port` 9000,
   `startPayload` `"LASTCALL"`, action Timeline `LAST_CALL`. The POS sends
   the datagram `LASTCALL` to the device on port 9000. Because matching is a
   prefix match, `LASTCALL\n` also works.
4. **Announce back.** Output event `POS_ACK`, type UDP, destination the POS
   address, port 9001, payload `ACK`. Put an OutputEvent milestone at the
   start of the `LAST_CALL` timeline so the POS gets a confirmation.

What you will observe: the Stream Deck key runs its action on press and
ignores release. The door contact produces a rising edge when it activates
and a falling edge when it releases. The UDP trigger fires on every matching datagram
with no debounce, so the POS should send once.
