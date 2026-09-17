# Plugin SDK

**Audience:** plugin authors, integrators, and AI agents designing a plugin
without access to the DMX Core 100 source code.
**Verified against:** DMX Core 100 software `main` at commit `1cfcdeb0`
(2026-09-16), Plugin SDK contract 1.12.
**User-facing documentation:**
<https://docs.dmxcore.com/dmx-core-100/integrations/plugins>,
<https://docs.dmxcore.com/dmx-core-100/integrations/publishing-plugins>
**Reference code:** <https://github.com/DMXCore/DMXCore100.Plugin.Example>
(exercises the whole SDK), <https://github.com/DMXCore/DMXCore100.Plugin.Shelly>
(a complete output plugin).

The user documentation covers packaging, publishing, and the Plugins page.
This document covers what the host actually does with a plugin: lifecycle
and timeouts, the dispatch and fault rules, each API surface with its exact
contract and the behavior you cannot see from the interface alone, which
API to pick for a given need, and what plugins cannot do today.

---

## 1. Mental model

A plugin is a .NET class library that the device loads **in-process, fully
trusted**, into its own isolated assembly load context. It gets one host
object with a dozen API surfaces and does whatever it wants with them.
There is no sandbox, no permission model, and no separate process. An
administrator installs it; that is the trust boundary.

A plugin plays one or more **roles**:

| Role | API | The plugin is |
|---|---|---|
| Event source | `Triggers` | Something that happened, turned into a Plugin-type input trigger the venue maps to any action |
| Mirror | `Entities`, `Mqtt` | An adapter that exposes the device's entities to another platform and relays commands back |
| Action target | `Actions` | A provider of output events the device fires into the plugin's platform |
| Control Value backend | `ControlValues` | The owner of a named value the whole device shares |
| Output protocol | `Outputs` | A driver that turns DMX channel data into commands for networked lights |
| Importer | `Imports` | A decoder that turns a vendor file into a recorded cue |
| OSC endpoint | `Osc` | The owner of an OSC address subtree on the device's port |

Everything the plugin does through these APIs lands in the **same pipelines
as every other surface**. Firing a trigger is identical to a Stream Deck
key. Executing an entity command is identical to the Integration API. The
plugin never gets a private path into the engine.

---

## 2. Packaging and identity

| Item | Rule |
|---|---|
| Target | `net10.0` class library referencing `DMXCore.PluginSdk` with runtime assets excluded; the device supplies the SDK assembly and redirects any version the plugin compiled against to its own copy. |
| Identity | `<PluginId>` in the project file: ASCII letters, digits, hyphens. It is the storage folder, log scope, and settings scope. Never change it once shipped. |
| Manifest | `manifest.json` is generated at build from the project file: `id`, `name`, `version`, `entryAssembly`, `minSdkVersion`, `author`, `packageId`. Do not check one in. |
| Entry point | One public class implementing `IPlugin` with a parameterless constructor, found in the entry assembly. |
| Package | `dotnet pack` produces a `.nupkg` of package type `DmxCorePlugin` whose only payload is `content/plugin.dmxplugin`, and a bare `.dmxplugin` archive for manual upload. |
| Dependencies | Everything the plugin needs must be inside the archive. Native libraries are allowed and proven (the Lightjams importer bundles ffmpeg). |
| Registry | nuget.org, or any NuGet V3 feed the device is pointed at. Devices search for the package type, read the `DMXCore.PluginSdk` dependency range from the feed before downloading, and only offer versions the host contract satisfies. Downloads are verified against the feed's SHA-512 hash. |
| Contract version | `major.minor` of the SDK assembly, not the NuGet package version. Additive: a new minor adds APIs. The manifest's `minSdkVersion` defaults to the contract of the SDK you compile against; a host older than that refuses to load the plugin. |
| Reserved ids | Package ids under `DMXCore.*` are first-party only. |
| Tags | `requires-mqtt` marks a plugin that cannot work without a broker; the device warns before install when it has none. Software installs have no built-in broker; Balena appliances do. |

Contract history that matters when choosing a floor:

| Contract | Added |
|---|---|
| 1.0 | Base: settings, state, MQTT, entities, triggers, playback, Control Value backends |
| 1.1 | `Mdns` |
| 1.2 | `Outputs` (output protocols) |
| 1.3 | Destination discovery, plugin fixture profiles |
| 1.6 | Mapping fields on output protocols, discovery-stamped options, generated manifest (1.4 and 1.5 were never shipped) |
| 1.7 | `Actions` (output action providers) |
| 1.8 | 16-bit fine channels in plugin fixture profiles |
| 1.9 | `Imports` (cue importers) |
| 1.10 | `Choice` setting type |
| 1.11 | `Osc` |
| 1.12 | `Number` entity kind for Counter Control Values |

---

## 3. Lifecycle and runtime rules

### 3.1 Load, initialize, shutdown

1. Package files live under the data folder in `plugins/<id>/`. Each load
   copies them to a shadow folder and loads from there, so the live folder is
   never file-locked. Uploads, deletes, enable, disable, and reload apply
   immediately without a device restart.
2. The host constructs the plugin, then calls `InitializeAsync` once. It must
   return within **30 seconds** or the plugin is marked failed. Do the cheap
   setup here and start real work with `SchedulePeriodic` or your own task.
3. `ShutdownAsync` is called on device shutdown, disable, reload, delete, or
   fault-budget exhaustion. It must return within **10 seconds** or the
   plugin is abandoned. Dispose subscriptions and stop background work.
4. On reload the old load context is unloaded best-effort. A plugin that
   carries native libraries leaks its old context by design; the leak is
   bounded by the number of reloads.
5. Plugins load **late** in device startup, after the engine and the storage
   layer. Anything that binds to a plugin (a Control Value referencing its
   backend, an output mapping using its protocol) is held and bound when the
   plugin appears.
6. Starting the device with the `-no-plugins` switch loads nothing (safe
   mode).

Runtime states shown on the Plugins page and returned by the admin API:
`RUNNING`, `FAULTED` (fault budget exhausted), `FAILED` (load or initialize
error, with the message), `DISABLED`, `PENDING_INSTALL`, `PENDING_DELETE`.

### 3.2 Dispatch and the fault budget

- Every callback into the plugin (MQTT message, entity state, settings
  change, cue event, OSC message, periodic tick, host-to-backend calls) is
  queued and run **serially per plugin**, in delivery order, on a host
  thread. A slow handler delays the plugin's own later callbacks, never the
  engine or other plugins.
- The cancellation token passed to a handler signals plugin shutdown.
- A handler that throws is logged and counted. After **20** faults in a
  session the plugin's callbacks are stopped, its state becomes `FAULTED`,
  and its MQTT availability topic goes `offline`. The engine is unaffected.
  Reload or re-enable resets the count.
- Two surfaces are **exempt** from the budget because an unreachable
  integration must not disable the plugin: action providers (section 5.4)
  and cue importers (section 5.6). Their exceptions are reported to the user
  who pressed the button instead.
- `SchedulePeriodic` invocations never overlap: the next interval starts
  when the previous invocation completes. Exceptions are logged, counted,
  and the schedule continues.

### 3.3 Settings and state

| Store | What | Where | Notes |
|---|---|---|---|
| Settings | Admin-editable values declared in `PluginInfo.Settings` | Device database, per plugin id | Types String, Integer, Boolean, Choice. `Secret` masks the value in the UI only. Reading an undeclared key, or with the wrong typed getter, returns null. `OnChanged` fires on every save from the UI; re-read inside the handler. Settings survive updates and reloads. |
| State | A JSON blob the plugin owns | Device database, per plugin id | Not shown in any UI. Survives restarts and updates. Use for runtime memory such as a last-poll timestamp or a last score. There is no size limit enforced, so keep it small. |
| Package files | The archive contents | `plugins/<id>/` under the data folder | Not part of database backups. After a restore onto a fresh device, reinstall from the registry; settings and state rows come back with the database. |

### 3.4 Device identity

`DeviceInfo` gives the stable hardware serial, the branded product name
(which is the whitelabel name when a whitelabel license is active, so never
hard-code "DMX Core 100"), the user-assigned device name, and the software
version. Build external identifiers such as MQTT topics and discovery ids
from the serial.

### 3.5 Connection state

`SetConnectionState(connected, detail)` drives the link icon on the Plugins
page and the on-device status indicator. Call it on every transition to the
thing the plugin integrates with. It is not the fault budget and does not
change the runtime state.

---

## 4. Choosing an API

| Need | Use | Not |
|---|---|---|
| Something happened on my platform and the venue should decide what to do | `Triggers.FireAsync(code)`, declare the code in `PluginInfo.Triggers` | `Playback` |
| Play a specific cue or preset the plugin chooses | `Playback` or `Entities.ExecuteAsync` | |
| Show the device's presets, cues, levels on my platform and relay commands | `Entities` (+ `Mqtt` for discovery) | Polling the admin API |
| Let a device output event do something on my platform | `Actions.RegisterProvider` | An MQTT topic the user has to type |
| Hold a shared value that knobs, menus, timelines, and scripts all move | `ControlValues.RegisterBackend`, or an internal Control Value plus `Entities` | Plugin state alone |
| Drive networked lights from DMX | `Outputs` | Subscribing to universes (there is no such API) |
| Read a vendor recording into a cue | `Imports` | |
| Own an OSC address space, reply to senders | `Osc` | An OSC input trigger |
| Find my devices on the LAN | `Mdns`, or the protocol's own broadcast inside `GetDestinationOptionsAsync` | |
| Talk to a serial device | Plain `System.IO.Ports` from the plugin; see the serial gap in section 7 | |

---

## 5. API surfaces

### 5.1 Triggers

`FireAsync(code)` fires every enabled Plugin-type input trigger with that
code. A trigger whose `address` names a plugin id fires only for that
plugin. Unknown or disabled codes are ignored with a log line. No payload
travels with the fire; the trigger's action runs as any rising edge would.
Declare the codes in `PluginInfo.Triggers` so the admin UI can list them;
undeclared codes still work.

### 5.2 Entities

The entity catalog is the device's controllable and observable surface,
addressed by namespaced codes: `preset.`, `ambient.`, `cue.`, `timeline.`,
`sound.`, `fixture.`, `zone.`, `schedule.`, `cv.`, `system.`.

| Kind | Namespaces | State | Commands |
|---|---|---|---|
| Scene | cue, timeline, sound | none | Activate |
| Switch | preset, ambient, schedule, system.mute, system.outputmute, system.blackout, Toggle Control Values | `IsOn` | TurnOn, TurnOff, Toggle |
| Level | system.masterdimmer, system.volume, zone, fixture (intensity only), Level Control Values | `Level` 0..1 | SetLevel |
| Select | Selector Control Values | `Choice` | SetChoice |
| Number | Counter Control Values, with `Min`, `Max`, `Step` | `Number` | SetNumber, rounded and clamped |
| Button | system.stop, system.clearambient | none | Activate |
| Sensor | system.nowplaying and similar | `Text` | none |

Rules that matter:

- **Coalescing.** State changes are coalesced per entity at **150 ms**. An
  entity settles at its final value; intermediate values of a fade are
  dropped. Do not use entity state to follow a fader in real time.
- **Echo.** State changes are delivered to every subscriber including the
  plugin that caused them. Treat the state change as the confirmation to
  report, not the command you sent.
- **Catalog changes** are debounced at 500 ms and delivered without a
  payload; re-read the catalog and diff.
- Codes are stable across renames; key your platform on the code.
- Unknown codes and mismatched commands are logged and ignored, never
  thrown, because integrations routinely race entity deletion.
- `fixture.X` is intensity only. RGB and other functions are not entity
  commands.
- Commands carry origin `INTEGRATION` into Control Value writes.

### 5.3 MQTT

The plugin shares the device's broker connection.

- `IsConnected`; publishing while disconnected throws. Subscriptions made
  while disconnected are applied on connect.
- `OnConnectionChanged` fires once with the current state immediately after
  subscribing, then on every transition. The host re-applies topic
  subscriptions on reconnect but **not** anything you published; republish
  retained discovery and state payloads in this handler.
- Two retained availability topics are maintained by the host: one for the
  device (`online` after connect, `offline` by last-will) and one for the
  plugin (`offline` when stopped, disabled, or faulted). Reference them
  from discovery payloads instead of a will of your own; a shared connection
  can carry only one will.
- Topic filters accept `+` and `#`. QoS 0, 1, or 2 on publish. Payloads are
  UTF-8 text.

### 5.4 Actions (output action providers)

One provider per plugin; registering again replaces it. The provider
enumerates **targets** (id, label, optional group) and executes one by id
with the output event's free-form payload. Registered providers appear as
the Plugin output-event type; the user picks a target, and the resulting
output event can be fired from any trigger site, including timelines and
scripts.

- Enumeration and execution are each bounded by a **10 second** timeout.
- Keep `GetTargetsAsync` fast; it runs when the picker opens. Throw to tell
  the UI why the list is unavailable.
- Exceptions are reported to the Test button and the log, and do **not**
  count against the fault budget.
- Dispose the handle when the settings needed to reach the platform are
  cleared, so the output-event editor stops offering the targets.

### 5.5 Control Value backends

Covered in the Control Values document, section 6. Summary: register a
backend name, implement level and raw writes, accept the set of status ids
to subscribe, answer `GetControlData` from cache, push every device-side
change with `PushControlData`. Host-to-backend calls are serial and
fault-isolated. A Counter's raw value is the integer verbatim.

### 5.6 Outputs (output protocols)

A registered protocol appears on the Outputs page under its port type. For
each mapping the host:

1. Asks `GetChannelCount(config)` for the DMX footprint; 0 rejects the
   mapping. This is called right after restart, before any discovery, so
   compute it from the stored mapping options.
2. Opens a **session** lazily before the first delivery, and again after a
   delivery failure.
3. Slices the universe, dedupes, rate-limits to `MaxUpdatesPerSecond`, and
   calls `SendAsync` on a host worker with the **latest** values only.
   Intermediate frames are coalesced. Return false or throw to fail; the
   host retries with the newest values after a **2 second** backoff and
   reopens the session on repeated failures.
4. Disposes the session when the mapping is removed, the protocol is
   unregistered, or the plugin shuts down.

Descriptor options: `MappingFields` (extra per-mapping settings rendered on
the mapping page, values arrive in `config.Options`),
`SupportsDestinationDiscovery` (shows a Discover button that calls
`GetDestinationOptionsAsync`; `refresh` true means the user pressed it, do
an active scan), and a suggested plugin fixture profile and personality so
the fixture editor can prefill a patch from a mapping.

Plugin fixture profiles (`RegisterFixtureProfile`) are served live while
the plugin runs and never written to the profile library. Fixtures patched
with one keep their rows when the plugin is disabled; they stop rendering
until it returns. Channel functions are the color subset (intensity, RGB,
white variants, amber, UV, lime, color temperature) plus their 16-bit fine
counterparts. Never change a profile code or protocol id once shipped.

### 5.7 Imports (cue importers)

One importer per plugin. The descriptor names the file extensions and an
action label; matching files in the device's file explorer gain that
action, with optional per-import options rendered as a form. The plugin
decodes the file and writes frames to a host sink:

- One `WriteFrameAsync` call per source frame with **all** its universes.
  Every call becomes one synchronized sACN group that plays back
  atomically.
- Timestamps are milliseconds from the start and must not decrease.
  Universe ids 1–63999, 1–512 channel bytes from channel 1. The sink copies
  the buffer.
- Return metadata for the cue or null for host defaults. Throw to fail; the
  message is shown to the user and nothing is saved. Exceptions do not
  count against the fault budget.

### 5.8 OSC

The plugin subscribes to address patterns on the device's OSC listener port
and can send from that port.

- Patterns: OSC 1.0 `?`, `*`, `[abc]`, `[a-z]`, `[!abc]`, `{foo,bar}`, none
  crossing `/`, plus the OSC 1.1 `//` operator for any depth. Matching is
  case-insensitive. Up to **64** subscriptions per plugin.
- Precedence for an incoming message: an IP-claimed control surface first,
  then plugin subscriptions, then the device's own handling. A message any
  plugin matches is **claimed** and does not reach input triggers. The
  reserved `/dmxcore` tree can be observed but never claimed.
- Argument types map to OSC tags: int, float, string, bool, long, double,
  byte array as blob, null as nil.
- Reply to `OscMessageInfo.Source`; TouchOSC and similar listen on the port
  they send from.
- When OSC is disabled in the host configuration, subscriptions stay
  registered but never fire, and sending throws.

### 5.9 mDNS

The host runs one long-lived browser per service type, shared by all
features and plugins. The first `GetServicesAsync` for a type waits about
two seconds for the initial burst; later calls return a cache snapshot
instantly. `RefreshServicesAsync` forces a fresh query, for a user-facing
Discover button only. Results are a cache snapshot, not a reachability
guarantee. Prefer the resolved address over the host name.

### 5.10 Playback

`PlayPresetAsync`, `PlayCueAsync`, `StopAsync`, and the top-level cue
started and ended events. Cues started inside a timeline do not raise the
events. Prefer `Triggers` when the venue should be able to reconfigure the
behavior; use `Playback` when the plugin itself decides what to play.

---

## 6. Testing

`DMXCore.PluginSdk.Testing` ships `TestPluginHost`, an in-memory host that
records what the plugin publishes, fires, plays, and stores, and lets a test
simulate incoming MQTT messages, cue events, and settings changes. Nothing
runs in the background; periodic callbacks run only when the test invokes
them, so tests are deterministic. Passing the plugin's `PluginInfo` to the
test host enables the same declaration and type checks on settings reads as
the real host. The Example repository's `DevHost` is an interactive console
harness on the same class.

On a device, the development loop is: build, upload the `.dmxplugin` on the
Plugins page or via `POST /api/website/plugins/upload`, and the new version
hot-applies. `POST /api/website/plugins/{id}/reload` restarts one plugin.
Log output is tagged with the plugin id in the regular device logs.

---

## 7. Limits and gaps

Things a plugin cannot do today. When a design needs one of these, request
it from DMX Core rather than working around it. Numbers such as #134 are
DMX Core's internal issue tracker references; the tracker is private, so
quote the number when you talk to DMX Core and read
[issues.md](../issues.md) for what each one asks for.

| Gap | Detail | Status |
|---|---|---|
| Serial port ownership | The device probes every serial port at startup unless probing is disabled globally, and there is no claim registry, so a plugin's serial device can receive garbage at boot. | Open, #134 |
| Trigger payload | `FireAsync` carries only a code. A value has to go through a Control Value or an entity. | Not planned |
| DMX input | No API to feed live DMX universes into the device or to subscribe to universe data. Importers produce recorded cues only. | Not planned |
| Fixture functions beyond intensity | `fixture.X` entity commands set intensity only. Color must go through a preset, a cue, or an output protocol. | Not planned |
| Creating entities | Plugins cannot create Control Values, presets, cues, triggers, or output events. The operator creates them; the plugin binds to them by code. | By design |
| Timelines and schedules | No API to play a timeline directly except the `timeline.X` entity, and none to read or change schedules. | Not planned |
| Plugin UI | No custom pages, no touchscreen items, no custom-menu items. Settings are the declared form only. | Not planned |
| HTTP endpoints | A plugin cannot expose its own web routes on the device. | Not planned |
| Per-plugin permissions or sandboxing | Full trust, in-process. | By design |
| Real-time entity state | 150 ms coalescing and last-value delivery. A backend registration is the low-latency path for Control Values. | By design |
| Timeline child cue events | `OnCueStarted` and `OnCueEnded` fire for top-level cues only. | By design |
| Hot reload of native libraries | Reloading a plugin with native dependencies leaks its previous load context. | By design |
| Backups | Package files are not included in database backups; reinstall from the registry after a restore. | By design |
| Stale SDK remark | The `IPlugin` interface documentation still says plugins apply on restart; hot reload has shipped and the remark is outdated. | Documentation |
