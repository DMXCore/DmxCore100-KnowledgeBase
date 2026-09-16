# Tracker cross-reference

The DMX Core 100 software issue tracker is private. This file carries the
substance of every tracker item that a concept file cites, so an outside
reader or an AI agent can understand what has been asked for and quote the
number when talking to DMX Core. Summaries are written by DMX Core from the
issue text; state is as of the date on each entry.

Wording: "Open" means requested and accepted as a direction, not scheduled.
"Closed" means shipped unless noted.

## Control Values

### #131 — Counter Control Value kind

*Open, 2026-09-16. Cited by: [control-values.md](concepts/control-values.md) section 7.*

**Motivation.** A venue wants Stream Deck keys that step a home and away
score up and down, with the score living in the Core as a Control Value so a
display plugin, the touchscreen, timelines, and Home Assistant all see the
same number. Today the kinds are Level (0..1), Selector, and Toggle, so an
integer has to be faked as a Selector with one choice per number, which is
painful to author and renders as a wall of buttons on the custom menu.

**Proposal.** A `Counter` kind: an integer with min, max, step (default 1),
and a clamp-or-wrap flag. Up and Down step by the step. Set accepts an
integer. Backends receive the raw integer. The live status carries the
integer. The entity catalog gains a `number` entity kind, mapped to a Home
Assistant `number` entity by the Home Assistant plugin. The script API
handles the kind. The editor gains a kind picker with min, max, and step.

**Related.** #132, #133, #135.

### #132 — Control Value action: step by a signed amount

*Open, 2026-09-16. Cited by: control-values.md section 7.*

**Motivation.** A "home touchdown" key needs to add 6 to a score in one
press. The Control Value trigger action carries only the operation and the
set value, so Up always steps by the Control Value's own step size. Today
the workarounds are a script action that calls `up()` six times, or a plugin
that does the arithmetic behind an output action.

**Proposal.** An optional signed step amount on the trigger action, applied
by Up and Down in place of the default step. For Level a 0..1 delta or a
percent; for Counter an integer; for Selector a number of choices. Clamp and
wrap rules unchanged. Timeline Control Value milestones and the script API
get the same parameter. Stream Deck press-and-hold repeat keeps working and
applies the amount on each repeat.

**Related.** #131.

### #133 — Stream Deck live Control Value readout

*Open, 2026-09-16. Cited by: control-values.md section 7.*

**Motivation.** Staff stepping a score or a volume from a Stream Deck have
no readout. Key faces are rendered once from the static label, Control Value
keys have no active or inactive appearance, and the Stream Deck+ LCD strip
shows only the product name and bank label.

**Part A, any Stream Deck (USB and network dock).** A "show value" option on
a key assignment. Render label plus current value (Level as percent,
Selector as choice name, Toggle as on or off, Counter as the integer).
Re-render only the affected key on each coalesced status update. Give
Control Value keys an active state where it makes sense (Toggle on, Selector
matches the key's set value).

**Part B, Stream Deck+.** Split the LCD strip into four segments, one per
dial, each showing the value bound to that dial. Dial rotation as Up and
Down for Selector and Counter Control Values (today dials drive Level
targets only). Dial press as a configurable action. Example: dial 1 = home
score, dial 2 = away score, strip reads "HOME 14  AWAY 7".

**Related.** #131, #135.

### #134 — Serial port ownership

*Open, 2026-09-16. Cited by: control-values.md section 7.*

**Motivation.** A plugin that drives a serial device (for example an LED
ticker behind a USB adapter) shares the host with the Core's own serial DMX
support. At startup the Core opens and probes every enumerated serial port
unless probing is disabled globally, which can send garbage to the plugin's
device, and there is no registry of claimed ports.

**Proposal.** The startup probe skips ports a plugin has claimed. The plugin
SDK gains enumerate and claim calls for serial ports. This is an SDK
contract change.

### #135 — Custom menu: read-only Control Value display item

*Open, 2026-09-16. Cited by: control-values.md section 7.*

**Motivation.** The custom menu (touchscreen and web) has input items for
Control Values (a slider for Level, a segmented row for Selector, action
buttons) but nothing that simply shows a value. A scoreboard readout, or a
volume readout beside an Up and Down pair, needs a read-only label bound to
a Control Value.

**Proposal.** A `ValueDisplay` item type carrying a Control Value code and a
format (label plus value: percent for Level, choice name for Selector, on or
off for Toggle, integer for Counter). Rendered live in both the touchscreen
and the web custom menu from the existing status broadcast. Low priority;
cheap once #131 exists.

**Related.** #131, #133.

### #102 — Connect Control Value to a fader

*Closed, shipped 2026-09-16. Background for: control-values.md sections 3.4,
4.4, 4.5.*

Made the Control Value an input-agnostic bus: the internal backend, the
origin set and the "not from me" rule, the `drives` binding with two-way
write-back, the Faders page Controls view, and the web level write endpoint.
Explicitly out of scope and still open as design directions: MIDI CC output
feedback for motorized faders and LED rings, a Faders page on the
touchscreen, and one Control Value driving several targets.
