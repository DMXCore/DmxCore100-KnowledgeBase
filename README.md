# DMX Core 100 Knowledge Base

In-depth documentation of how the DMX Core 100 works under the covers, written
for integrators, plugin authors, and AI agents that need to plan an advanced
integration without access to the source code.

This is **not** the user documentation. Start there for the operator's view:
<https://docs.dmxcore.com/dmx-core-100>. Come here when you need to predict
what the system will do: data shapes, timing, ordering, feedback rules, the
exact set of surfaces that touch a concept, and what it cannot do yet.

## How to use this repository

- Every concept has one file under [`concepts/`](concepts/) with the same
  skeleton: mental model, data model, runtime behavior, surfaces, plugin SDK,
  limits and gaps, worked examples.
- Each file states the software commit it was verified against. Behavior
  described here is the behavior at that commit. If your device runs a newer
  release, the user documentation and release notes take precedence.
- [`issues.md`](issues.md) summarizes the DMX Core tracker items that the
  concept files cite. The tracker itself is private, so the summaries here are
  what an outside reader gets; quote the number when talking to DMX Core.
- [`llms.txt`](llms.txt) is the machine-readable index for AI agents.

## When you find a gap

Every concept file ends with a "Limits and gaps" table. If your design needs
something listed there, or something not covered at all, ask DMX Core for it
rather than working around it. Contact DMX Core or your distributor with:

1. The scenario in one paragraph.
2. Which existing pieces you planned to combine, by the names used in these
   documents.
3. The exact missing piece and where in the flow it sits.
4. The tracker number from [`issues.md`](issues.md) if the gap is already
   listed there.

That is enough to triage in minutes.

## Concepts

| File | Covers |
|---|---|
| [concepts/control-values.md](concepts/control-values.md) | Named values on a bus: Level, Selector, Toggle; internal and DSP backends; origins; fades; every surface that reads or writes them |
| [concepts/triggers-and-actions.md](concepts/triggers-and-actions.md) | The shared trigger action record and its modes; every input trigger type's matching rules; value mode; the dispatch rules; schedules, scripts, and the Integration API as action sources; output events |

More concepts will be added. Planned: timelines and milestones, the plugin SDK by API surface, the Integration API,
the entity catalog, scripting, faders and the control-value bus, and the
trigger-to-action dispatch path.

## License

The documentation is licensed under
[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0/). See
[LICENSE](LICENSE) for the notice about the product it describes.
