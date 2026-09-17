# Integration API and the Entity Catalog

**Audience:** integrators, plugin authors, and AI agents planning an
integration without access to the DMX Core 100 source code.
**Verified against:** DMX Core 100 software `main` at commit `30d51022`
(2026-09-16), Integration API protocol version 1, Plugin SDK contract 1.12.
**User-facing documentation:**
<https://docs.dmxcore.com/dmx-core-100/integrations/integration-api>,
<https://docs.dmxcore.com/dmx-core-100/integrations/mcp-server>,
<https://docs.dmxcore.com/dmx-core-100/integrations/home-assistant>

The user documentation carries the full wire contract of the Integration
API: endpoints, JSON shapes, status codes, limits, and the Companion
mapping. This document does not repeat it. It explains the **entity
catalog** underneath, which three surfaces project (the Integration API,
the MCP server, and the plugin entity API), where each entity's state comes
from, what each command actually does inside the device, and what none of
the three surfaces can do today.

---

## 1. Mental model

The **entity catalog** is one list of "things you can control or observe",
each with a stable namespaced code, a display name, and a **kind** that fixes
its state shape and its commands. Three surfaces project the same catalog:

| Surface | Transport | Who | Auth |
|---|---|---|---|
| Integration API | HTTP + WebSocket, `/api/integration/v1` | Control systems: Companion, Crestron, Node-RED, scripts | Integration-type API key |
| MCP server | Model Context Protocol, `/mcp` | AI clients: Cursor, Claude Desktop | MCP-type API key |
| Plugin entity API | In-process, `host.Entities` | Installed plugins, for example Home Assistant | None; the plugin is trusted |

A command on any surface is turned into the **same internal action** a
Stream Deck key, a schedule, or an OSC trigger would produce. There is no
private path. Consequences: the Output Off gate, missing-target warnings,
origin tagging, and every rule in the Triggers and Actions document apply.
Writes carry origin `INTEGRATION`.

The catalog is **not** the admin API. Creating, editing, and deleting
entities, files, users, and settings is the admin web API's job and needs a
user login. Integration and MCP keys are refused there, and user tokens are
refused on the Integration API.

---

## 2. The catalog

### 2.1 Kinds

| Kind | State field | Commands | Stateless |
|---|---|---|---|
| `scene` | none | `activate` | yes |
| `switch` | `isOn` | `turnOn`, `turnOff`, `toggle` | |
| `level` | `level` 0..1 | `setLevel` | |
| `select` | `choice`; catalog carries `choices` | `setChoice` | |
| `number` | `number` integer; catalog carries `min`, `max`, `step` | `setNumber` | |
| `button` | none | `activate` | yes |
| `sensor` | `text` | none | |

Codes and commands match case-insensitively. State echoes the catalog's
canonical code. `choice` is matched case-insensitively and returned in
catalog form.

### 2.2 Namespaces and what is listed

| Namespace | Source | Kind | Listed when |
|---|---|---|---|
| `preset.CODE` | Presets that are not ambient | switch | Has a code |
| `ambient.CODE` | Presets marked ambient | switch | Has a code |
| `cue.CODE` | Cues | scene | Has a code; admin-only items are excluded |
| `sound.CODE` | Sounds | scene | Same |
| `timeline.CODE` | Timelines | scene | Same |
| `schedule.CODE` | Schedules | switch | Has a code |
| `zone.CODE` | Zones | level | Has a code |
| `fixture.CODE` | Fixture instances | level | Enabled and has a code |
| `cv.CODE` | Control Values | level, switch, select, or number by kind | Enabled and has a code |
| `output.CODE` | Output events | switch for a Digital Output, button for every other type | Enabled and has a code |
| `system.*` | Built in | see below | Always |

Built-in entities: `system.masterdimmer` (level), `system.volume` (level),
`system.mute` (switch), `system.outputmute` (switch), `system.blackout`
(switch), `system.stop` (button), `system.clearambient` (button),
`system.nowplaying` (sensor).

The display `name` falls back to the code when the entity has no name.
Codes are the user-assigned codes and survive renames; key everything on
them.

### 2.3 Catalog changes

The catalog is rebuilt when presets, cues, sounds, timelines, schedules,
zones, fixtures, or Control Values are created, renamed, deleted, enabled,
or disabled. Notifications are debounced at **500 ms**. The Integration
API sends a fresh `catalog` frame followed by a full `state` frame; plugins
get a payload-less callback and re-read; MCP clients re-list.

---

## 3. Where state comes from

Every entity's state is derived from an internal signal and pushed into a
per-entity cache. Understanding the source tells you what the state means.

| Entity | State means | Source signal |
|---|---|---|
| `preset.X` `isOn` | The preset is in the set of **active temporary presets** the fixture engine is holding. It turns off when the preset is stopped, replaced, or released. | Fixture control change |
| `ambient.X` `isOn` | The ambient preset is in the active ambient set. | Fixture control change |
| `fixture.X` `level` | The fixture's **direct-control intensity modifier**, 0..1. This is the operator override slider, not the rendered DMX output. A fixture driven only by a cue reports whatever its modifier holds. | Fixture control change |
| `zone.X` `level` | The zone dimmer. A zone absent from the device's dimmer map is at 1.0. | Server state snapshot |
| `system.masterdimmer`, `system.volume` | The master values 0..1 | Server state snapshot |
| `system.mute`, `system.outputmute`, `system.blackout` | The three global flags | Server state snapshot |
| `schedule.X` `isOn` | The schedule's enabled flag, persisted | Schedule save |
| `cv.X` | See the Control Values document | Control Value status |
| `output.X` `isOn` (Digital Output only) | The last **commanded** logical level of the module port (on = energized, before the event's Inverted polarity), not a read-back of the pin. Every Digital Output event on the same port reports the same value. False after a restart until something drives the port. A pulse reports on, then off after the pulse width. | Any fire of a Digital Output event, from any surface |
| `system.nowplaying` `text` | A human-readable status line: `Cue: INTRO`, `Sound: WALKIN`, `Playing timeline: SHOW1`, `Playing 3 timelines`, `Routing input`, or empty when idle. Uses the display name instead of the code when that device setting is on. | Playback operation status |

Rules:

- **Coalescing.** State pushes are coalesced per entity at **150 ms**.
  Leading and trailing edges are delivered; intermediates during a fade are
  dropped. An entity always settles at its final value.
- **Echo.** Every subscriber receives every change, including the one that
  caused it. Treat the state frame as the confirmation of a command.
- **Unknown is not off.** An entity that has not reported is absent from
  state lists. A missing `isOn` means unknown.
- **Stateless kinds** (scene, button) never appear in state.
- **No timeline or cue position.** `system.nowplaying` is the only playback
  feedback. There is no per-item progress, remaining time, or hold state
  (section 7).

---

## 4. What each command does

| Entity | Command | Internal effect |
|---|---|---|
| `cue.X`, `sound.X` | `activate` with optional `loop`, `fadeInMs`, `fadeOutMs` | Builds a Cue or Sound trigger action. Omitted fields take the device's **Settings → Playback** defaults, the same as a tap in the UI. This is the only surface where omitted values fall back to system defaults; a stored trigger action never does. |
| `timeline.X` | `activate` with optional `loop`, `fadeInMs`, `fadeOutMs` | Builds a Timeline trigger action. `loop` replaces the timeline's own loop when sent; the fades replace the timeline's own when non-zero; anything omitted plays as authored. Timecode chase still forces its own loop. |
| `preset.X` | `turnOn`, `activate` | Preset trigger action (fade to the preset). `turnOff`: stop the preset with the default fade. `toggle`: whichever is the opposite of the current active state. A command that matches the current state is a no-op. |
| `ambient.X` | same | Ambient Preset trigger action; `turnOff` clears that ambient with the default fade. |
| `schedule.X` | `turnOn`, `turnOff`, `toggle` | Sets the schedule's enabled flag and **saves it**. A no-op when already in that state. |
| `zone.X` | `setLevel` | Sets the zone dimmer, clamped. |
| `fixture.X` | `setLevel` | Sets the fixture's intensity modifier, clamped, through the same path as the operator faders. A fixture with no control data is ignored with a warning. |
| `cv.X` | `setLevel`, `turnOn`, `turnOff`, `toggle`, `setChoice`, `setNumber` | Control Value operations with origin `INTEGRATION`. No Up, Down, or fade. |
| `output.X` (Digital Output) | `turnOn`, `turnOff`, `toggle`, `activate` | OutputEvent trigger action with operation Set On, Set Off, the opposite of the tracked level, or Pulse. It is the one switch that also accepts `activate` on the Integration API. Dropped while output is off, except `turnOff`. MCP `activate` on it sends `turnOn`, like any switch. |
| `output.X` (other types) | `activate` | OutputEvent trigger action: fires once. Dropped while output is off. |
| `system.masterdimmer`, `system.volume` | `setLevel` | Sets the master value, clamped. |
| `system.mute`, `system.outputmute`, `system.blackout` | `turnOn`, `turnOff`, `toggle` | Sets the flag. Blackout is the latched mask; Output Off is the cold park that drops playback starts. |
| `system.stop` | `activate` | StopPlayback trigger action. |
| `system.clearambient` | `activate` | Clears every ambient preset with the default fade. |

Validation differs by surface. The Integration API answers 400 for a
malformed body, a command that does not fit the kind, an out-of-range value,
or playback fields on anything but a cue or sound activate, and 404 for an
unknown code. The plugin API and MCP **log and ignore** unknown codes and
mismatched commands, because integrations routinely race entity deletion.
Success is silent everywhere; the state change is the confirmation.

---

## 5. Surface specifics

### 5.1 Integration API

- Enabled under **Settings → System**; disabled means 404 on every path.
- Keys are long-lived, Integration-only, issued once and shown once. A key
  of the wrong type for the path is refused.
- HTTP: `GET /info`, `GET /catalog`, `GET /state`, `GET /state/{code}`,
  `POST /execute`. WebSocket: `GET /events` sends `hello`, `catalog`,
  `state` on connect, then `state` deltas, `pong`, and `error` frames, and
  accepts `execute` and `ping` frames.
- Limits per key: 8 event-stream connections, 100 executes per second
  sustained with bursts to 200, 64 KB inbound frames.
- Each connection has an outbound queue of 512 frames. On overflow the
  oldest deltas are dropped and a full `state` snapshot is sent once the
  client drains, without a preceding `catalog` frame. Apply every state
  frame the same way.
- `GET /info` is a snapshot of identity and health for polling. Do not use
  the admin status endpoint.
- Ports are the web UI's: 80 and 443 on the appliance, 8000 and 8001 on
  desktop installs. OSC on UDP 8000 is a different service.

### 5.2 MCP server

Enabled separately under **Settings → System** with its own key type. Tools:

| Tool | Maps to |
|---|---|
| `list_entities` (optional `entityType` filter) | Catalog |
| `get_entity_state` | Last known state |
| `activate` | `activate` on a scene, or `turnOn` on a switch |
| `set_level` | `setLevel` |
| `set_switch` with `on`, `off`, `toggle` | Switch commands |
| `stop_playback` | `system.stop` |
| `get_fixture_capabilities` | Modifier ids a fixture accepts |
| `set_fixture_modifiers` with optional `fadeMs` | **Direct fixture control** (RGB, intensity, custom channels). Not a catalog command. |
| `release_fixture_control` | Release the direct takeover |
| `get_live_status` | Snapshot of master and zone dimmers, active ambients, per-fixture modifiers |

Entities are referenced by code **or name**, with an optional namespace
filter. MCP is the only external surface with color control and with
`fadeMs` on a fixture write. There is no `setNumber` tool for Counter
Control Values and no `setChoice` tool for Selectors as of this commit.

### 5.3 Plugin entity API

Typed records instead of JSON, the same kinds and commands, plus
`OnCatalogChanged` and `OnStateChanged` subscriptions dispatched through the
plugin's serial queue. Details and the 1.12 `Number` kind are in the Plugin
SDK document. The Home Assistant plugin is the reference consumer: presets,
cues, and timelines become scenes; levels become number sliders; switches
become switches; selects become selects; counters become number boxes with
min, max, and step; now-playing becomes a sensor.

---

## 6. Worked example: a Companion page for a bar

Goal: a Stream Deck running Bitfocus Companion on a laptop, not attached to
the device, with buttons that light up to reflect state.

1. **Setup.** Issue an Integration key, point the Companion module at the
   device's web UI port with the key, and let it open the event stream.
   The `hello`, `catalog`, and `state` frames populate every dropdown.
2. **Scene buttons.** `cue.PARTY` and `timeline.LAST_CALL` with `activate`.
   Leave loop and fades unset so the device defaults apply. To light the
   button while playing, compare `system.nowplaying` text against the
   entity's catalog name and its code suffix, case-insensitively.
3. **Look buttons.** `preset.DINNER` with `toggle` and a feedback on
   `isOn`. The device reports `isOn` true as soon as the preset is active
   and false when another preset replaces it.
4. **Faders.** `zone.BAR` and `system.masterdimmer` with `setLevel` sent
   as `execute` frames over the WebSocket during a drag, not one POST per
   tick. Feedback snaps at 150 ms granularity by design.
5. **Volume and source.** `cv.VOL1` `setLevel`, `cv.SRC` `setChoice` with
   `choices` read from the catalog entity, not from state.
6. **Score.** `cv.HOME_SCORE` `setNumber` to reset at kickoff. Stepping
   the score from Companion means reading `number` from state and sending
   `number + 6`; there is no `up` command.
7. **Emergency.** `system.blackout` `turnOn` and `system.stop` `activate`
   on guarded buttons.

What you will observe: every press returns 202 or silence immediately; the
button state changes when the device reports it, within about 150 ms.
Nothing you send from Companion is distinguishable, inside the device, from
the same action on a physical key, except the origin tag on Control Value
writes.

---

## 7. Limits and gaps

Things none of the three surfaces can do today. When a design needs one of
these, request it from DMX Core rather than working around it. Numbers such
as #142 are DMX Core's internal issue tracker references; the tracker is
private, so quote the number when you talk to DMX Core and read
[issues.md](../issues.md) for what each one asks for.

| Gap | Detail | Status |
|---|---|---|
| Fire an input trigger | No entity for input triggers. A plugin can fire one; a control system cannot. Bind the control system's button to an entity instead, or give it an HTTP input trigger URL. | Not planned |
| Fire an output event | No entity for output events. | Not planned |
| Playback position and hold state | Only the now-playing text. No progress, remaining time, loop count, or HOLDING for timelines, and no way to release a hold. | Not planned |
| Dimmer and volume on a timeline `activate` | The Integration API carries loop and fades only; the dimmer and volume scales a trigger action or script can pass are not on the wire. | Not planned |
| Color and non-intensity fixture functions | `fixture.X` is intensity only on the Integration API and plugin API. MCP has `set_fixture_modifiers`. | By design for v1 |
| Control Value Up, Down, step amount, fade | Absolute writes only. | Not planned |
| MCP `setChoice` and `setNumber` | No MCP tool for Selector or Counter writes. | Open, not scheduled |
| Filtering and paging the catalog | The Integration API returns the whole catalog; MCP filters by namespace only. | Not planned |
| Fade on level commands | `setLevel` on zones, fixtures, masters, and Control Values applies instantly over the Integration API and plugin API. | Not planned |
| Fixture level reflects the modifier, not the output | A cue-driven fixture reports its override slider, not what it is emitting. | By design |
| Browser clients | The WebSocket requires a header; browsers cannot set one. Use a server-side client. | By design |
| CRUD, users, files, settings | Admin web API only, with a user login. | By design |
| Per-key permissions | An Integration key can execute every entity. There is no read-only key or per-entity scope. | Not planned |
