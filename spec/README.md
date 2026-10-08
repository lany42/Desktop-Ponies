# Desktop Ponies — Clean-Room Behavioral Specification

This directory specifies the observable behavior of **Desktop Ponies v1.69** (the
"reference implementation", RI) in enough detail to write an independent implementation —
in particular one for Linux — without reading the original source code.

The emphasis is on **behavior and data formats**: how the content files are read and how they
drive the ponies on screen. Rendering is specified only as a contract (what must appear, where,
and when), because the drawing technology differs per platform.

## How to read this

| If you want to… | Read |
|-----------------|------|
| understand the overall shape and the component/interface breakdown | [1. Architecture](01-architecture.md) |
| load the content directory | [2. Content layout](02-content-layout.md) |
| **parse the files that drive the ponies** | **[3. `pony.ini` schema](03-pony-ini-schema.md)**, [4. `house.ini`](04-house-ini-schema.md) |
| implement user settings and profiles | [5. Options and profiles](05-options-and-profiles.md) |
| **make ponies behave correctly** | **[6. Simulation model](06-simulation.md)** |
| animate and draw them | [7. Images, animation, presentation](07-images-and-animation.md) |
| build the application around it | [8. Application shell](08-application-shell.md) |
| add the mini-games | [9. Games](09-games.md) |
| decide which RI bugs to keep | [10. Quirks and decisions](10-quirks-and-decisions.md) |
| test your implementation | [11. Conformance](11-conformance.md) |
| (reviewers only) check the spec against the source | [Appendix A. Traceability](A-traceability.md) |

A minimal viable implementation needs Chapters 2, 3, 6 and 7 (plus §5.1 for the settings the
simulation reads). Chapters 4, 5, 8 and 9 add houses, persistence, the menu and games.

## Conventions

* **MUST / SHOULD / MAY** — RFC 2119 meanings.
* **RI** — the reference implementation as found in this repository (`Desktop Ponies/`,
  `Desktop Sprites/`). "RI defect" marks behavior that is almost certainly unintended; it is
  always followed by a **Recommendation**. Chapter 10 lists all of them so each can be decided
  once (Parity vs. Fix).
* Coordinates: screen pixels, origin top-left, **y grows downward**.
* Time: "simulated" time advances in 40 ms steps scaled by the time-factor option; "external"
  time is real elapsed time (Chapter 6 §6.3).
* `U` is a uniform random number in [0, 1); `coin` is `U < 0.5`; `round` is round-half-to-even
  unless stated.
* Pseudocode is illustrative; where prose and pseudocode disagree, report it as a spec bug.

## Clean-room notes

* The chapters describe behavior in prose, tables and pseudocode written for this document; they
  contain no code from the RI. Literal strings that are part of observable behavior (file names,
  keywords, user-visible messages) are quoted because an implementation must reproduce them.
* Appendix A maps spec sections to RI source files for **reviewers** verifying accuracy.
  Implementers following a strict clean-room process should not use it.
* The content (artwork, `pony.ini` files, sounds) is licensed CC BY-NC-SA 3.0, as is the RI
  source; the file formats themselves are documented here so independent tools can read that
  content.

## Scope

In scope: content formats (`pony.ini`, `house.ini`, `game.ini`, legacy `interactions.ini`,
`.art` alpha maps, profiles), content discovery, the complete pony simulation, houses, effects,
speech, interactions, mouse interaction, animation timing, z-order, sound rules, the application
flow (start-up, command line, selection menu, launch, context menus, screensaver mode), and the
mini-games.

Out of scope: pixel-exact widget layouts of the RI's dialogs, the RI's Windows/GTK rendering
internals, the pony editors' UI (summarised only), the release tool, and the online community
link check.

## Provenance

Derived from a full read of the RI's simulation, parsing, host-loop and game code, the
application shell and editor code, and an automated survey of all 311 shipped `pony.ini` files,
8 `house.ini` files, 2 `game.ini` files and ~2,700 GIF images. Statements about the shipped
corpus (counts, value distributions) come from that survey.
