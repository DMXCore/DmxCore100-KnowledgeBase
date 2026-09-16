# Timelines

**Audience:** integrators, plugin authors, and AI agents planning an
integration without access to the DMX Core 100 source code.
**Verified against:** DMX Core 100 software `main` at commit `e36880c2`
(2026-09-16), Plugin SDK contract 1.12.
**User-facing documentation:**
<https://docs.dmxcore.com/dmx-core-100/playback/timelines>,
<https://docs.dmxcore.com/dmx-core-100/playback/hold-milestones>,
<https://docs.dmxcore.com/dmx-core-100/playback/timecode-chase>,
<https://docs.dmxcore.com/dmx-core-100/playback/layers-and-priority>

This document describes what a timeline is made of, how the device turns
the stored milestones into a running show, how playback is started,
stopped, held, and chased to timecode, which surfaces can do each of those,
and what a timeline cannot do today.

---

## 1. Mental model

A timeline is a **clock with milestones**. It has a duration, and every
milestone has a timestamp on that clock. When the clock reaches a
milestone, the milestone's event starts: a cue, a sound, a preset, an
output event, a Control Value move, a script, or a **Hold** that freezes the
clock until something releases it.

A timeline is the device's answer to "several things at once or in
sequence". A trigger, key, schedule, or script plays **one** thing; if that
thing is a timeline, it can play many. Timelines are not nested: a timeline
milestone cannot play another timeline.

The clock has three kinds of source:

- **Internal.** Play starts at the cursor (normally 0) and runs on the
  device's own clock. This is the default.
- **Timecode chase.** The clock follows incoming Art-Net timecode, so Play
  joins wherever the site clock is now (section 6).
- **Holds.** A Hold milestone stops the clock in place while looping
  events keep going, and a release restarts it (section 5).

Every play builds a **fresh instance** from the stored definition. Editing a
timeline does not affect an instance already running; the next play picks
up the change.

---

## 2. Data model

### 2.1 Timeline

| Field | Type | Meaning |
|---|---|---|
| `timelineId` | int | Database id. Use `code` in integrations. |
| `code`, `name`, `description` | string | Identity. Codes match case-insensitively. |
| `durationMS` | double | Length of the clock. Playback ends here regardless of milestone durations. |
| `loop` | int | 0 = forever, 1 = once, N = N passes. Treated as 1 while chasing timecode. |
| `dimmer` | double 0..1, default 1 | Overall intensity scale for the instance. |
| `fadeInDurationMS`, `fadeOutDurationMS` | double | Whole-instance fades at start and at stop. |
| `events` | array | The milestones (section 2.2). |
| `favorite`, `onlyAdmin` | bool | UI flags. `onlyAdmin` hides the timeline from non-admin lists. |
| `priority`, `groupId` | byte, int | Stored; not consulted by playback in this build. |
| `timecodeChaseMode` | `OFF`, `CHASE`, `EXCLUSIVE` | Internal, Chase, Timecode only (section 6). |
| `timecodeStreamId` | int 0..255 | Art-Net timecode stream to follow. |
| `timecodeOffsetMS` | double, default 3,600,000 | Timecode at timeline 0. Default `01:00:00:00`. |
| `timecodeFreewheelMS` | double, default 500, minimum 100 | After Roll: how long the timeline free-runs after the signal stops or freezes. |
| `timecodeOnDropout` | `PAUSE`, `FREERUN`, `TRIGGERONLY` | Auto Stop: "Off, pause", "Off, keep running", "On, stop". |
| `timecodeOnReturn` | `AUTOFOLLOW`, `WAITFORTRIGGER` | Auto Resume: On, Off. Ignored with `TRIGGERONLY`. |
| `timecodeOnStart` | `AUTOJOIN`, `PLAYTOJOIN` | Auto Start: On, Off. |

### 2.2 Milestone (timeline event)

Common fields: `timelineEventId`, `timestampMS` (int), `type`, `enabled`,
`name`, `trackId`. Type-specific fields:

| Type | `playCode` | `durationMS` | `loop` | `fadeInDurationMS` / `fadeOutDurationMS` | Other |
|---|---|---|---|---|---|
| `CUE` | Cue code | Maximum play time; empty = the cue's own length | 0 forever, 1 once, N passes | Fades for the cue | `dimmer` scale, `inPointMS` / `outPointMS` trim, `playbackLayer` override |
| `SOUND` | Sound code | Maximum play time | Same | Fades for the sound | `dimmer` (volume scale), `pan`, `inPointMS` / `outPointMS`, `playbackLayer` |
| `PRESET` | Preset code | Time to hold the preset, then release it | | Fade-in is the fade to the preset; fade-out is the release fade | `dimmer` scale |
| `OUTPUTEVENT` | Output event code | | | | Fires once at the timestamp |
| `CONTROLVALUE` | Control Value code | Fade time for a Level Set | | | `controlValueOperation`, `controlValueSetValue`, `controlValueStepAmount` |
| `SCRIPT` | Script code | | | | Queued at the timestamp |
| `HOLD` | Release trigger code, or empty | Timeout in ms; empty or 0 = wait forever | | Fade-out is the release fade for sustained events; 0 = let the current pass finish | `completeLoopOnRelease` |
| `COMMENT` | | | | | Editor annotation, no runtime effect |

A milestone whose type this build does not recognize (data from a newer
release) is preserved on save and skipped at play.

### 2.3 Tracks

The editor shows lanes, and `trackId` is stored, but for cue, sound, and
preset milestones the device **recomputes** the track at build time: events
of the same type are placed on the first lane whose previous event has
ended, so two overlapping events of one type never share a lane. A
zero-duration event occupies a minimal slot so two events at the same
timestamp still get separate lanes. Other types keep their stored track.
Lanes have no semantic meaning beyond this; they do not stop or replace each
other. Replacement is the job of playback layers (section 3.4).

### 2.4 In and out points

A cue or sound may carry its own trim (in and out points on the entity).
A milestone's `inPointMS` and `outPointMS` are **relative to that trimmed
start** and are capped at the entity's out point. So a 60 s track trimmed
to 10..50 s with a milestone in point of 5 s starts at 15 s of the file.

---

## 3. Building and running an instance

### 3.1 Build

On every play the device:

1. Creates an engine timeline with the duration, dimmer, and whole-instance
   fades.
2. Recomputes tracks (section 2.3).
3. Registers the timeline with the timecode chase service when the chase
   mode is not Off, and with the hold coordinator when any Hold exists,
   declaring the release-trigger codes those holds name.
4. Adds one engine event per enabled milestone, as in the table below.

| Type | Engine behavior |
|---|---|
| Cue | Starts a cue player at the timestamp with the milestone's loop, trim, dimmer, and fades. Effective playback layer is the milestone's override, else the cue's own. A cue with Bounce set plays forward then backward. A cue's known universes are declared up front so sparse recordings hold their last frame rather than blacking out during a gap. |
| Sound | Starts a sound player with loop, trim, pan, fades, and `dimmer` as a volume scale. `playbackLayer` null means mix freely; same-layer sounds replace each other with a short fade. |
| Preset | Fades fixtures to the preset over the fade-in and applies the preset's per-entry effects (global, fixture, or zone scope). If a duration is set, the preset is released at the end of it over the fade-out, or instantly when no fade-out is set. It never falls back to the system default fade. |
| Output event | Sends the output event once. |
| Control Value | Runs the operation with origin `TIMELINE`. A Level Set with a duration ramps over that time. |
| Script | Queues the script; the timeline clock never waits for it. |
| Hold | Engages the hold (section 5). |

### 3.2 Play, stop, pause, resume

| Verb | Behavior |
|---|---|
| Play | Builds a new instance and starts it at 0, or at the given position for Jump. If the same timeline is already playing, the new instance is loaded first and the old one stopped, so there is no gap. A play from 0 is written to the audit log. |
| Stop | Stops the instance; the whole-instance fade-out applies. Stopping all timelines is a separate call. |
| Pause | Freezes the clock in place. |
| Resume | If the timeline is holding, resume is a **release** (section 5). If it is chasing and should re-join the clock, resume is a Play that joins live timecode. Otherwise the paused clock continues, or, when nothing is paused, the timeline is played from 0. |
| Release | Releases the active hold, or latches an early release (section 5). |
| Stop at completion | Used by schedules with run-to-completion: the instance finishes its current pass and stops. |

**Output Off.** While output is off, Play is dropped by every source.
Stop and Pause still work.

**Timecode only.** Play, Jump, and Resume are refused unless they join live
timecode in range (section 6). The web API returns an error response with
the reason; triggers and schedules log a warning and skip.

### 3.3 Play parameters: the caller's fades, dimmer and volume apply, the loop is the timeline's

A trigger action, schedule, script call, or Integration API `activate`
carries loop, fade-in, fade-out, dimmer, and volume fields. For a
**timeline** target (#142, shipped 2026-09-16): `fadeInDurationMS` and
`fadeOutDurationMS` replace the timeline's own fades for that play when
non-zero (0 keeps the timeline's); `dimmerScale` multiplies the timeline's
`dimmer`; `volume` is an instance-level scale on every sound the timeline
starts, multiplied with each sound milestone's own level (1.0 = as
authored). The caller's **loop is not applied**: a trigger action's loop
field defaults to 1 and cannot express "keep the timeline's own", so the
timeline's `loop` always stands (section 8). Timecode chase still forces
its own loop.

The one caller parameter that matters is the trigger action **mode**:
Normal plays, and re-pressing while the timeline is parked at an unnamed
hold releases it instead of restarting. Momentary plays on press and
releases an unnamed hold on release.

### 3.4 Layers and concurrency

- Two timelines can play at once. Their cues and sounds interact through
  playback layers exactly as if started individually: same-layer items
  replace each other, different layers or no layer mix.
- A timeline cue milestone can override its cue's layer per milestone.
  Sound milestones have a layer of their own; sounds have no entity-level
  default layer.
- The timeline `dimmer` scales every cue in the instance. Each cue
  milestone's `dimmer` scales further, and the editor's intensity profile
  ramps that value over the milestone's duration.
- A cue reaching its own end inside a timeline does not fire the
  `CueStarted` or `CueEnded` script events; those fire for top-level cue
  plays only.

### 3.5 Loop

With `loop` other than 1 the whole clock wraps at `durationMS`. Every
milestone restarts on the next pass. Hold latches do not carry across a
wrap (section 5). While chasing timecode the loop is forced to 1.

---

## 4. Surfaces

### 4.1 Playing a timeline

| Surface | How | Notes |
|---|---|---|
| Trigger action `Timeline` | Input trigger, control surface key, custom-menu item, schedule | Mode Normal or Momentary. Loop and fades on the action are ignored (section 3.3). |
| Schedule | Action type `Timeline` | Starts at the schedule start; at the end the instance stops, or finishes its pass with run-to-completion. |
| Script | `dmx.playTimeline(code)` | No options. `dmx.isPlaying(code)` reports state. |
| Integration API, MCP, plugin entity API | `timeline.CODE` entity of kind `scene`, command `activate` | `loop`, `fadeInMs`, `fadeOutMs` are accepted but ignored for timelines. |
| Admin web API | `PUT /api/website/timeline/{id}/play` | Also `jump?posMS=`, `stop`, `pause`, `resume`, `release`, and `PUT /api/website/timelinecontrol/stop` for all timelines. |
| Web UI and touchscreen | Timeline list and dashboard transport | Dashboard shows position, pause/resume, scrub, stop, and a HOLDING badge. |

### 4.2 Editing

Admin web API: `GET /api/website/timeline/list`, `GET /timeline/{id}`,
`POST /timeline/{id}` to save the definition, `POST /timeline/{id}/events`
to save milestones, `POST /timeline/{id}/clone`, `DELETE /timeline/{id}`.
There is no timeline editing over the Integration API or from plugins.

### 4.3 Observing

Playback state is broadcast to the admin UI over SignalR as part of the
"now playing" data and appears as the `system.nowplaying` sensor entity in
the Integration API as text. There is no per-timeline position feed for
integrations (section 8).

---

## 5. Holds

A Hold milestone makes the clock wait. The rules, in the order they matter:

**While holding**

- The clock freezes at the hold's timestamp. The transport shows HOLDING.
- Cue and sound milestones whose `loop` is not 1 **keep playing**: an
  infinite loop keeps looping, a finite count finishes its passes. Loop
  counts are respected; a finite loop can run out while the hold waits.
- One-shot sounds pause in place. DMX output holds its last frame.
- A chased timeline leaves the clock for the duration of the hold.

**Release paths**

| Path | Releases | Latches when the hold is not reached yet |
|---|---|---|
| Momentary falling edge of the playing trigger or key | An active **unnamed** hold only | Yes, the unnamed slot |
| Re-press of a Normal-mode playing trigger or key | An active unnamed hold (instead of restarting) | No |
| Rising edge of the input trigger named by the hold's `playCode` | The active hold that names that trigger | Yes, for that code |
| Web transport resume, the `release` endpoint | Any active hold, named or not | Yes, the unnamed slot |
| Timeout (`durationMS`) | The hold that armed it, only if it is still the active one | Never |

**Latches.** An early release is remembered and the hold passes straight
through when reached, still giving sustained events their release
treatment. Latches are cleared when a new instance starts and when the
timeline loops around. Each hold has its own latch; a timeline may contain
several holds, each waiting in turn.

**Named holds are selective.** A hold with a release trigger ignores the
Momentary release and the Normal re-press. Only its trigger, the web
transport, the release endpoint, or its timeout get past it.

**On release** the clock resumes. Sustained events are faded out over the
hold's `fadeOutDurationMS`, or, with no fade, allowed to finish their
current pass. With `completeLoopOnRelease` the clock itself waits until
the looping event completes its pass before resuming, so the loop ends on
its musical boundary.

**Boot and restart.** Latches never survive a restart of the instance.
Trigger edges that arrive while no timeline with holds is playing are not
remembered.

---

## 6. Timecode chase

The device receives Art-Net `ArtTimeCode` (OpCode `0x9700`) on UDP 6454 on
its lighting network interface. It never generates timecode. MIDI timecode
and LTC are not supported. Frame rate follows the packet (24, 25, 29.97
drop-frame, 30).

Timeline position = incoming timecode minus `timecodeOffsetMS`. In range
means between 0 and the timeline's duration. Loop is forced to 1.

| Mode | Behavior |
|---|---|
| `OFF` (Internal) | Timecode is ignored. Play starts at the cursor. |
| `CHASE` | Play joins live timecode when in range. Without timecode, or out of range, the timeline runs on its own clock from the cursor. |
| `EXCLUSIVE` (Timecode only) | Play, Jump, and Resume are accepted only when they join live timecode in range, from every source. Stop and Pause always work. |

Joining takes about one second: the instance is loaded one second ahead
of the clock and started when the clock arrives, so the playhead lands
within a few milliseconds of timecode. Before the offset, Play starts at 0
and catches the clock when it reaches the offset, with a one-second
lead-in. A join inside a cue or sound starts it part way through; events
that ended before the join point are skipped.

While locked, the timeline free-runs between packets. A forward jump of
more than one second or any backward jump re-seeks. A rewind to before the
offset stops the timeline and re-arms it. A manual Stop or Pause sticks for
the rest of that pass of the clock even with Auto Start on; a new pass
(rewind or leaving range) clears that.

| Setting | Values | Behavior |
|---|---|---|
| After Roll (`timecodeFreewheelMS`) | ms, min 100 | Keep running on the own clock this long after packets stop or the frame freezes. |
| Auto Stop (`timecodeOnDropout`) | `PAUSE`: hold the last look. `FREERUN`: keep running indefinitely. `TRIGGERONLY`: stop; the next Play re-joins; returning timecode is ignored for the rest of the pass. | Applied when the After Roll runs out. |
| Auto Resume (`timecodeOnReturn`) | `AUTOFOLLOW`: re-lock at once (a paused timeline seeks; a free-running one re-locks in place if within half a second, else seeks). `WAITFORTRIGGER`: stay as is until Play or Resume. | Applied when timecode returns after a dropout. Not used with `TRIGGERONLY`. |
| Auto Start (`timecodeOnStart`) | `AUTOJOIN`: start by itself when valid timecode is in range, including after a device restart. `PLAYTOJOIN`: wait for Play. | |

Holds and timecode: a hold leaves the clock; incoming timecode is ignored
until release, then Auto Resume applies.

---

## 7. Worked example: touchdown show with a go button

Goal: a Stream Deck key fires a touchdown show that bumps the score,
flashes the ticker, plays a stinger, then waits for staff to press "go"
before the crowd loop ends.

Timeline `TOUCHDOWN`, duration 30 s, loop 1:

| Time | Milestone | Settings |
|---|---|---|
| 0.0 s | Control Value `HOME_SCORE` | Up, amount 6 |
| 0.0 s | Control Value `TICKER_MESSAGE` | Set `TOUCHDOWN` |
| 0.0 s | Sound `STINGER` | loop 1, layer 1 |
| 0.5 s | Cue `TD_STROBE` | duration 6000, layer 1 |
| 3.0 s | Sound `CROWD_LOOP` | loop 0, layer 1 |
| 6.0 s | Hold | release trigger `GO_KEY`, timeout 20000, fade-out 1500 |
| 6.2 s | Sound `HORN` | loop 1, layer 1 |
| 6.5 s | Cue `HOUSE_LOOK` | duration 3000 |
| 8.0 s | Control Value `TICKER_MESSAGE` | Set `SCORE` |

Wiring: the Stream Deck touchdown key is a trigger action Timeline
`TOUCHDOWN`, mode Normal. A second key is bound to a Plugin-type or HTTP
input trigger with code `GO_KEY` and no action of its own; it exists to be
named by the hold.

What happens: the score and ticker change at 0 s with origin `TIMELINE`,
the stinger cuts anything else on layer 1, the strobe cue runs 6 s, the
crowd loop starts at 3 s and keeps looping when the clock parks at the hold
at 6 s. Pressing the go key releases the hold: the crowd loop fades over
1.5 s, the horn on the same layer cuts it anyway, the house look restores,
and the ticker returns to the score. If nobody presses go within 20 s the
timeout releases the hold. Because the hold names `GO_KEY`, re-pressing the
touchdown key while holding does not release it; it restarts the show.

---

## 8. Limits and gaps

Things a timeline cannot do today. When a design needs one of these,
request it from DMX Core rather than working around it. Numbers such as
#142 are DMX Core's internal issue tracker references; the tracker is
private, so quote the number when you talk to DMX Core and read
[issues.md](../issues.md) for what each one asks for.

| Gap | Detail | Status |
|---|---|---|
| Caller play parameters | Fixed for fades, dimmer, and volume (section 3.3). The caller's loop is still not applied to a timeline: the action's loop field has no "unset" value. | Closed, #142 (loop: by design for now) |
| Nested timelines | A milestone cannot play another timeline. | Not planned |
| Script lifecycle events for timelines | Scripts can subscribe to cue started and ended, not to a timeline starting, ending, or holding. Put a Script milestone at 0 s and at the end instead. | Not planned |
| Position feed for integrations | No per-timeline position or state over the Integration API, MCP, or plugin entity API beyond the now-playing text. | Not planned |
| Editing over the Integration API | Definitions and milestones are admin web API only. | By design |
| Falling-edge named release | A named release trigger releases on its rising edge only. | Deferred |
| Timecode generation, MTC, LTC, per-milestone timecode triggers | Receive-only Art-Net ArtTimeCode playhead chase. | Out of scope |
| Named markers | Jump is by position in milliseconds only; there is no jump-to-marker. | Not planned |
| Stored track ids | `trackId` on cue, sound, and preset milestones is recomputed at play; the stored value is a hint for the editor only. | By design |
| Priority and group fields | Stored on the timeline but not consulted by playback. | Informational |
| Play parameters for schedules | A schedule's fades, dimmer, and volume apply to a timeline; its loop does not (the timeline's own stands). | Part of #142 |
