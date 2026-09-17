# Scripting

**Audience:** integrators, plugin authors, and AI agents planning an
integration without access to the DMX Core 100 source code.
**Verified against:** DMX Core 100 software `main` at commit `1cfcdeb0`
(2026-09-16).
**User-facing documentation:**
<https://docs.dmxcore.com/dmx-core-100/scheduling-automation/scripting>,
<https://docs.dmxcore.com/dmx-core-100/scheduling-automation/scripting-api>
**Samples:** the public samples folder linked from the scripting page.

The user documentation lists every `dmx` call and the `ctx` object. This
document explains the execution model behind them: how a run is scheduled,
what the sandbox guarantees and forbids, what each call does inside the
device, the second, much smaller engine used for value transforms, and what
scripts cannot do today.

---

## 1. Mental model

A script is a **JavaScript program stored on the device** and run in a
sandbox when something fires it. It is the escape hatch for logic the fixed
trigger model cannot express: conditions, counters, sequences with waits,
payload parsing, and combining several actions with state in between.

Three things to keep in mind:

- **A script is an action, not a service.** It starts, does its work, and
  ends. Nothing persists between runs except what the script writes to its
  store. There is no event loop, no timers, no long-lived listener.
- **One run at a time per script.** A second invocation while a run is
  active is **dropped**, not queued. A script that sleeps for a minute is
  deaf for that minute.
- **The device is protected, not the script.** Runs are capped in time,
  memory, statements, and recursion, and run on their own thread, so a
  runaway script can never affect DMX output, trigger dispatch, or other
  scripts. The script itself is simply killed.

---

## 2. Data model

| Field | Type | Meaning |
|---|---|---|
| `scriptId` | int | Database id. |
| `code` | string | Identifier used by trigger actions, timeline milestones, and transform references. Case-insensitive. |
| `name`, `description` | string | Display. |
| `enabled` | bool | A disabled script does not run from triggers or events. The editor's Run button still works, so a script can be tested while disabled. |
| `source` | string | The JavaScript text. |
| `timeoutSeconds` | int, nullable | Wall-clock limit per run. Null means 30 s. Values above 600 are capped at 600. |
| `runOn` | list of `STARTUP`, `CUESTARTED`, `CUEENDED`, `SCHEDULEFIRED`, `SCHEDULEENDED` | Lifecycle events that run the script (section 4.2). |

Plus a per-script key-value **store** persisted on the same database row
(section 5.6). Run status and the last log lines are kept in memory only
and are lost on restart.

---

## 3. The sandbox

Every run builds a **fresh engine**. No state, globals, or closures survive
from one run to the next.

| Limit | Value | When hit |
|---|---|---|
| Wall-clock timeout | 30 s default, per-script up to 600 s | Run ends with "timed out". Time in `dmx.sleep` counts. |
| Memory | 4 MB | Run ends with a memory-limit error. |
| Statements | 500,000 | Run ends with a statement-limit error. An infinite loop dies here or at the timeout, whichever first. |
| Recursion depth | 64 | Run ends with an error. |
| Log lines | 100 most recent | Older `dmx.log` lines are discarded from the status panel. |
| Store size | 64 KB serialized | The `set` call throws. |

Guarantees:

- **Strict mode** is on.
- **No CLR access.** The only host objects visible are `dmx` and `ctx`.
  There is no `require`, no `fetch`, no file system, no reflection, no
  timers. `setTimeout` does not exist; use `dmx.sleep`.
- **Host errors are catchable.** A failing call such as publishing while
  the MQTT client is disconnected, or a bad OSC target, surfaces as a
  JavaScript exception the script can `try`/`catch`. Many lookups do not
  throw at all: an unknown fixture, zone, effect, or schedule code writes a
  warning to the run log and the call is a no-op.
- **The kill switches are not catchable.** The timeout and a manual Stop
  from the editor terminate the run regardless of `try`/`catch`.
- **Dedicated thread.** Each run gets its own long-running thread, so
  `dmx.sleep` blocks only the script. The timeout is enforced inside `sleep`
  too, because the engine's own time check only samples between
  statements.

---

## 4. How a run starts

### 4.1 Fired as an action

A `Script` trigger action (section 4 of the Triggers and Actions document)
queues the run and returns immediately; the trigger's dispatch never waits
for the script. Sources: input triggers, control-surface keys, custom-menu
items, timeline `SCRIPT` milestones, and schedules (#143, shipped
2026-09-16): a schedule's Script action runs once at the start time with
source `SCHEDULE` and the schedule code; nothing runs at the end time, so a
closing script subscribes to the `SCHEDULEENDED` event instead.

The invocation context arrives as `ctx`:

| Field | Value |
|---|---|
| `ctx.trigger.source` | The input trigger's type in uppercase (`UDP`, `MQTT`, `OSC`, `HTTP`, `CONTROLVALUE`, ...), `SCHEDULE` when fired by a schedule's Script action, `TRIGGER` when fired from a surface, menu, or timeline, `MANUAL` from the editor, `EVENT` for lifecycle events. |
| `ctx.trigger.code` | The input trigger's code, or the schedule's code, when fired by one. |
| `ctx.payload` | The raw payload of the input trigger message where the source has one (UDP, TCP, HTTP body or query, OSC argument text, MQTT payload); null otherwise. The editor's Run button can supply a test payload. |
| `ctx.event` | `{ name, code }` for lifecycle events; null otherwise. |
| `ctx.now` | `{ hour, minute, minutesSinceMidnight, weekday, iso }` in the device's display time zone, for comparison against `dmx.sunrise()` and `dmx.sunset()`. |

Plugin-fired triggers carry no payload, so `ctx.payload` is null for them.

### 4.2 Lifecycle events

A script with entries in `runOn` is started by the device itself:

| Event | When | `ctx.event.code` |
|---|---|---|
| `STARTUP` | Once after the device has finished starting. | none |
| `CUESTARTED` | A top-level cue starts. Cues inside a timeline do not count. | The cue code |
| `CUEENDED` | A top-level cue stops. | The cue code |
| `SCHEDULEFIRED` | A schedule runs its action. | The schedule code |
| `SCHEDULEENDED` | A schedule ends: its end time is reached, or a higher priority schedule takes over. A schedule without an end time never ends. | The schedule code |

Cue events are derived by diffing the playback status ticks, so they are
edge events per cue code, not per playback instance. Event dispatch never
blocks the source; the run is queued like any other and subject to the
one-run-at-a-time rule.

### 4.3 Manual runs

The editor's Run button, `POST /api/website/script/{id}/run`, runs the
script even when disabled, with `ctx.trigger.source` = `MANUAL` and an
optional test payload. `GET /api/website/script/{id}/status` returns the
run status and last log lines; `POST /api/website/script/{id}/cancel`
stops a running script.

---

## 5. What the API does inside the device

All playback calls build the same trigger action a key would and go
through the shared dispatcher with origin `SCRIPT`. The Output Off gate and
the missing-target rules apply.

### 5.1 Playback

| Call | Internal effect |
|---|---|
| `playCue(code, { fadeIn, fadeOut, loop, dimmer, toggle })` | Cue action. Defaults: no fades, loop 1, dimmer 1. `toggle` true stops the cue if it is already playing on its layer. |
| `playSound(code, { fadeIn, fadeOut, loop, volume, toggle })` | Sound action, same defaults. |
| `playTimeline(code, { loop, fadeIn, fadeOut, dimmer, volume })` | Timeline action. `loop` absent keeps the timeline's own; fades of 0 keep the timeline's own; `dimmer` and `volume` are 0..1 scales on the timeline's dimmer and on every sound it plays. Re-running while parked at an unnamed hold releases it. |
| `fadeToPreset(code, durationMs)` | Preset action with the fade as fade-in. |
| `stopPlayback()` | StopPlayback action: everything stops. |
| `fadeOut(durationMs)` | FadeOut action: current cue and sound fade out. |
| `fireOutputEvent(code)` | OutputEvent action. |
| `isPlaying(code)` | True when a cue, sound, or timeline with that code is currently playing. |

### 5.2 Looks and fixtures

| Call | Internal effect |
|---|---|
| `masterDimmer(level, durationMs)` | Fades the master dimmer. |
| `zoneDimmer(zoneCode, level, durationMs)` | Fades the zone dimmer. |
| `setFixture(code, { intensity, red, green, blue, amber, white, coolWhite, warmWhite, ultraViolet, <functionId> })` | **Direct control takeover** of the fixture with a partial update: only the keys given change, the rest keep their live value. Keys outside the nine named channels address custom functions by function id. Values are clamped 0..1. The fixture becomes `DIRECT`-controlled until released. |
| `getFixture(code)` | The live modifier: the nine channels, `others`, `controlledBy` (`NONE`, `TEMPPRESET`, `DIRECT`, `AMBIENT`), `presetCode`, `effectId`. |
| `releaseFixture(code)` | Releases direct control so the fixture returns toward its preset or ambient state. |
| `setZoneEffect(zoneCode, effectCode)` / `setGlobalEffect(effectCode)` | Sets or, with null, clears the effect. |

`setFixture` is the only external write path for color other than the MCP
server. There is no fade parameter on it; fade by stepping with `sleep`.

### 5.3 Control Values

`dmx.controlValue.get / set / up / down / toggle / status`, origin `SCRIPT`.
See the Control Values document, section 5.4, including the step amount on
`up` and `down` and the fade on `set`.

### 5.4 Messaging

| Call | Internal effect |
|---|---|
| `osc.send("ip:port", address, value)` | One OSC message from the device's OSC port. Numbers go as float32, booleans as OSC true/false, strings as strings, no value as an empty message. An invalid target logs a warning. |
| `mqtt.publish(topic, payload)` | Publishes through the device's broker connection. Throws when not connected. No retain flag. |

There is no receive side: scripts cannot subscribe to OSC or MQTT. Use an
input trigger of that type with a Script action and read `ctx.payload`.

### 5.5 Schedules and sun

`schedule.enable / disable / toggle / isEnabled(code)` change the schedule's
enabled flag and **persist** it. `sunrise(date?)` and `sunset(date?)` return
`{ hour, minute, minutesSinceMidnight }` for today or a `yyyy-MM-dd` date in
the device's display time zone, or null when no location is configured or
the sun does not rise or set that day.

### 5.6 Store

`store.get / set / delete / keys` is a per-script key-value store:

- Values are JSON-serializable; whatever `set` receives is serialized.
- Every `set` and `delete` is **write-through** to the database, so a run
  killed mid-way never loses committed writes.
- The whole serialized store is capped at 64 KB; a `set` that would exceed
  it throws.
- Scripts cannot read each other's stores. Share state between scripts
  through an internal Control Value instead.

### 5.7 Utilities

`sleep(ms)` blocks the script's thread. `log(...)` writes to the run log
shown in the editor and to the device log at Debug level.

---

## 6. Transform scripts

A value-mode input trigger (section 3.3 of the Triggers and Actions
document) can name a script as its **value transform**. This is a different
execution mode:

| Property | Transform | Regular run |
|---|---|---|
| Globals | `value` (normalized 0..1) and `payload` (raw text) only. No `dmx`, no `ctx`. | `dmx`, `ctx` |
| Result | The script's completion value, clamped to 0..1. A non-number result is treated as "no update". | Ignored |
| Budget | 1 MB, 10,000 statements, recursion 16, **50 ms** | Section 3 |
| Engine | Prepared once and cached per script; rebuilt on the next edit. Unknown, disabled, or non-compiling scripts are cached as failures until edited. | Fresh per run |
| Failure | The update is **dropped**. The untransformed value is never passed through, so a dead zone or threshold cannot leak. | Run status shows the error |

Write the transform as an expression or a short block whose last statement
is the value, for example `value < 0.05 ? 0 : value` for a dead zone, or a
conditional that reads `payload` for a JSON field the JSON path could not
express. Transforms fire at fader rates, so keep them pure and tiny.

---

## 7. Worked example: last-call sequence with a guard

Goal: a bartender key runs "last call" once per night, ignores repeat
presses, and restores the room after the sequence.

```javascript
// Script code LAST_CALL, timeout 120 s, fired by a Stream Deck key.
var today = ctx.now.iso.substring(0, 10);
if (dmx.store.get("lastCallDate") === today) {
  dmx.log("already ran tonight, ignoring");
} else {
  dmx.store.set("lastCallDate", today);          // write-through, survives a kill
  dmx.controlValue.set("VOL1", "40%", 2000);     // fade the DSP volume down
  dmx.fadeToPreset("LAST_CALL_LOOK", 1500);
  dmx.playSound("LAST_CALL_ANNOUNCE", { volume: 0.8 });
  dmx.sleep(60000);                              // counts toward the timeout
  dmx.fadeToPreset("CLOSING_LOOK", 5000);
  dmx.mqtt.publish("venue/bar/lastcall", "done");
}
```

What you will observe: the second press inside the same night is not
dropped by the run guard (the first run has finished) but by the store
check, so it logs and exits. A press during the 60 s sleep **is** dropped by
the one-run rule and leaves no trace except a debug log line. If the device
restarts during the sleep, the sequence does not resume; the store still
says it ran.

---

## 8. Limits and gaps

Things a script cannot do today. When a design needs one of these, request
it from DMX Core rather than working around it. The tracker is private;
quote the number when you talk to DMX Core and read
[issues.md](../issues.md) for what each one asks for.

| Gap | Detail | Status |
|---|---|---|
| Queued or concurrent runs | A second invocation during a run is dropped silently. There is no queue, no per-run instances, and no way for a script to know it was dropped. | By design |
| Long-lived scripts | No timers, no event subscriptions from inside a script, no background loop beyond the timeout. A "listener" is an input trigger with a Script action. | By design |
| Receiving OSC or MQTT in a script | Send only. | By design |
| HTTP requests | No `dmx.http`. Use an HTTP output event (GET only) or MQTT. | Deferred, planned as admin opt-in |
| Schedules running scripts directly | Fixed: Script is a schedule action type (runs at the start, section 4.1) and `SCHEDULEENDED` is a lifecycle event (section 4.2). | Closed, #143 |
| Timeline and hold events | No `TIMELINESTARTED`, `TIMELINEENDED`, or hold lifecycle events. | Not planned |
| Cue events for timeline children | `CUESTARTED` and `CUEENDED` are top-level only. | By design |
| Fade on `setFixture` | Instant; fade by stepping. | Not planned |
| Reading playback position, active preset lists, or the entity catalog | `isPlaying` and `getFixture` are the only reads. | Not planned |
| Shared state between scripts | Stores are per script. Use an internal Control Value. | By design |
| Importing modules or libraries | Single source text, no `require` or `import`. | By design |
| Persisted run history | Status and log are in memory, last run only. | Not planned |
| Touchscreen editing | Web UI only; the touchscreen can fire scripts via custom-menu items. | By design |
