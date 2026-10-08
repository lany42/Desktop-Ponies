# 3. Pony Definition Schema (`pony.ini`)

This chapter specifies the file that drives every pony on screen. It is the most important
chapter for interoperability: a conforming implementation must read every `pony.ini` in the
shipped content corpus (300+ files) and produce the behavior described in
[Chapter 6](06-simulation.md).

Conventions used here:

* **MUST / SHOULD / MAY** are used in the RFC 2119 sense.
* "Reference implementation" (RI) means the analysed Desktop Ponies v1.69 code base. Where the RI
  has a defect, the spec states the RI behavior and then gives a **Recommendation**. An
  implementation that wants pixel-for-pixel parity follows the RI; one that wants sane behavior
  follows the recommendation. Both are conforming unless marked otherwise.
* `⟨…⟩` denotes a field placeholder.

---

## 3.1 Location, naming and encoding

* Each pony lives in its own directory directly under `Ponies/` (see
  [Chapter 2](02-content-layout.md)). **The directory name is the pony's unique identifier**
  (case-sensitive; used by follow targets, interaction targets, house visitor lists and
  profiles).
* The definition file is `Ponies/⟨Directory⟩/pony.ini`. The file name is always lower case.
* The directory `Ponies/Random Pony` is special: it is loaded like any other pony but removed
  from the selectable set. Its only use is to provide an image for a "random pony" entry in
  pickers. See §2.3.
* Encoding: UTF-8. A leading byte-order mark (U+FEFF) **MUST** be skipped; almost every shipped
  file starts with one. (The RI also auto-detects UTF-16/UTF-32 BOMs; supporting these is
  optional.)
* Line terminators: LF, CR, or CR LF, in any mix.
* Despite the extension, the format is **not** INI. There are no sections and no `key=value`
  pairs.

## 3.2 Line classification

Process the file line by line, in file order:

1. If the line is empty or consists only of whitespace → ignore.
2. If the **first character** of the line is `'` (U+0027) → comment. Ignore for runtime purposes
   (an editor preserves comment lines and writes them back at the top of the file).
   A line with leading whitespace before `'` is *not* a comment.
3. Find the **first** comma. If there is none → the line is invalid; ignore it (an editor
   preserves it verbatim at the end of the file).
4. The **line identifier** is the text before the first comma, compared case-insensitively
   (ASCII/invariant lower-casing). The identifier is **not** trimmed: ` Behavior,…` (leading
   space) is unrecognised.
5. Dispatch on the identifier:

| Identifier (any case) | Meaning                                  | Section |
|-----------------------|------------------------------------------|---------|
| `name`                | Display name                             | §3.6    |
| `categories`          | Tags                                     | §3.7    |
| `behaviorgroup`       | Name for a behavior group number         | §3.8    |
| `behavior`            | A behavior                               | §3.9    |
| `effect`              | An effect bound to a behavior            | §3.10   |
| `speak`               | A speech line                            | §3.11   |
| `interaction`         | An interaction initiated by this pony    | §3.12   |
| `scale`               | Deprecated, ignored (preserved verbatim) | §3.13   |
| anything else         | Unknown, ignored (preserved verbatim)    | —       |

Unknown identifiers and extra trailing fields are **not** errors: the format is meant to be
forward-extensible. Implementations MUST ignore fields beyond those defined here.

Line order is irrelevant for parsing, but **file order is semantically significant** in several
places (first-match rules for special behaviors, zero-chance tie breaking, effect start order,
interaction evaluation order). Implementations MUST keep each entity list in file order.

## 3.3 Field tokenisation

A recognised line is split into **fields** on commas. Field 0 is the identifier itself. Two
splitting variants are used, differing only in which characters act as *qualifiers*:

| Splitter | Qualifier pairs           | Used for                                                             |
|----------|---------------------------|----------------------------------------------------------------------|
| **Q**    | `"`…`"`                   | `Behavior`, `Effect`, `Categories`, and the *contents* of brace lists |
| **QB**   | `"`…`"` and `{`…`}`       | `Name`, `BehaviorGroup`, `Speak`, `Interaction`, all `house.ini` lines |

### 3.3.1 Algorithm

```
fields  ← []
buffer  ← ""
i       ← 0
loop:
    s ← index of next ',' at or after i            (or len if none)
    q ← index of next opening qualifier at/after i (or len if none)
    if s ≤ q:                         # separator (or end of line) comes first
        buffer += line[i : s]
        append buffer to fields; buffer ← ""
        i ← s + 1
        if i > len: stop
    else:                             # an opening qualifier comes first
        buffer += line[i : q]
        c ← index of the matching closing char strictly after q
        if none: FAIL (unbalanced)
        buffer += line[q+1 : c]       # verbatim, qualifiers removed
        i ← c + 1
```

Consequences that implementations MUST reproduce:

* Qualifiers are **removed**, and their contents are copied verbatim (commas inside are not
  separators). There is **no escape mechanism**: a quoted value cannot contain `"`; a braced
  value cannot contain `}`.
* Qualified and unqualified text concatenate: `ab"c,d"ef` → one field `abc,def`.
* Inside one qualifier pair, the other pair's characters are literal. In QB, `{"a","b"}`
  yields the single field `"a","b"` (quotes retained), which is later split again with Q.
* Whitespace is **never** trimmed by the splitter. `Behavior, "stand"` yields the field
  ` stand` (leading space). Some field parsers trim (noted per field); most do not.
* A trailing comma yields a trailing empty field. An empty line segment between commas yields
  an empty field `""`. A line always yields at least one field.
* "Missing" means the index is beyond the end of the field list. This is distinct from an empty
  field.

### 3.3.2 Unbalanced qualifiers

Splitting can only fail at the first opening qualifier that has no closing partner. The RI
recovers by appending a closer and splitting again:

* Q: retry once with a `"` appended to the line.
* QB: retry with `"` appended; if that also fails (the unclosed qualifier was `{`), retry with
  `}` appended to the **original** line.

These retries always succeed. The effect to reproduce: **an unclosed qualifier swallows the
rest of the line, commas included, into one field** (e.g. `a,"b,c` → `a`, `b,c`).

## 3.4 Scalar value grammars

All numeric parsing is culture-invariant (`.` is the decimal separator regardless of locale).

| Kind      | Accepted syntax                                                                                                   | Notes |
|-----------|-------------------------------------------------------------------------------------------------------------------|-------|
| `int`     | optional surrounding whitespace, optional leading `+`/`-`, ASCII digits; 32-bit range                             | `1.0` is **not** a valid int |
| `real`    | optional surrounding whitespace, optional sign, digits with optional `.` fraction, optional exponent (`1e-3`); `,` thousands separators are also accepted (only reachable inside quotes: `"1,5"` reads as 15) | See note on NaN/Infinity below |
| `decimal` | as `real` but no exponent; a trailing sign is also accepted (used only by `house.ini` `bias`)                   | |
| `bool`    | `true` or `false`, **case-insensitive**, surrounding whitespace allowed                                           | The old techdoc says case-sensitive; RI accepts any case. Writers MUST emit `True`/`False`. |
| `vector`  | a *single field* containing exactly two `int`s separated by one comma, e.g. `"44,46"` (quotes needed in the line so the comma does not split the field) | anything else is invalid |
| `name`    | free text; compared case-insensitively when used as a reference                                                    | per-field trimming noted below |
| `pony-id` | free text; compared **case-sensitively and exactly** against pony directory names                                  | |
| `path`    | a file name relative to the pony directory                                                                        | see §3.4.1 |
| `list`    | a field that itself is split with **Q**; empty/whitespace-only entries are dropped; duplicates collapse (set semantics) | written as `{"a","b"}` |

`NaN` and `Infinity` are syntactically valid reals. In the RI, `Infinity`/`-Infinity` fail
every range check (all real fields have finite ranges) and fall back to the default. `NaN`
passes range checks: in a duration field it then makes the **whole pony** fail to load; in
Chance/Speed/Proximity it is stored and misbehaves at run time (an interaction with Chance NaN
fires every step). **Recommendation:** treat `NaN`/`Infinity` as invalid (use the default).

### 3.4.1 Paths

A path field is valid when it is non-empty, contains no characters invalid in a *file name* on
the host OS (on every OS this includes `/` and NUL; on Windows also `\ : * ? " < > |` and control
characters), and is not absolute. Therefore **image and sound files must sit directly in the
pony's directory**; subdirectories are not supported. The resolved path is
`Ponies/⟨Directory⟩/⟨field⟩`.

The RI resolves paths with the host file system's case rules. Content was authored on Windows,
so a few references may differ in case from the file on disk.
**Recommendation (Linux):** try the exact name first, then fall back to a case-insensitive
match within the pony directory.

## 3.5 Validation model

Every field of every entity line is in exactly one of three classes:

1. **Required** — no default. If missing or invalid the entity has a *fatal issue*.
2. **Defaulted** — has a default value.
   * Missing → the default is used **silently** (this is how short/old lines stay valid).
   * Present but invalid or out of range → the default is used **and** a non-fatal warning is
     recorded. Out-of-range values are **replaced by the default, not clamped** (e.g. a
     `Chance` of `1.5` becomes `0`).
3. **Unparsed** — taken verbatim.

When loading for the desktop runtime, the RI drops any entity with a fatal issue
("remove invalid items" mode), and then drops any **pony left with zero behaviors** from the
collection entirely (this includes directories with no `pony.ini`). The editors load in
permissive mode: entities with fatal issues and ponies without behaviors are kept so they can be
fixed.

Referential problems (dangling or ambiguous names, loops) are **not** load errors; they are
resolved lazily at run time (§3.14) and only reported by the editor.

## 3.6 `Name`

```
Name,⟨DisplayName⟩
```

| # | Field        | Kind     | Class    | Default        |
|---|--------------|----------|----------|----------------|
| 1 | DisplayName  | text     | Unparsed | (see below)    |

* Split with **QB**; only field 1 is used (anything after a further unquoted comma is ignored).
* If no `Name` line exists, the display name is the directory name. If several exist, the last
  one wins.
* The display name is used as the speaker prefix of speech bubbles (§6.9). Typical use:
  directory `Twilight Sparkle (Filly)` with `Name,Twilight Sparkle`.

## 3.7 `Categories`

```
Categories,⟨Tag1⟩,⟨Tag2⟩,…
```

* Split with **Q**. Every field after the identifier is added to the pony's tag set
  (case-insensitive set; untrimmed; an empty field adds the empty tag). Multiple
  `Categories` lines accumulate.
* A bare `Categories` (no comma) is an invalid line and is ignored — this appears in the
  shipped `Random Pony`.
* Tags are used only for filtering in the UI. Standard tags (in UI order): `Main Ponies`,
  `Supporting Ponies`, `Alternate Art`, `Fillies`, `Colts`, `Pets`, `Stallions`, `Mares`,
  `Alicorns`, `Unicorns`, `Pegasi`, `Earth Ponies`, `Non-Ponies`. The shipped corpus mostly uses
  lower-case spellings (`"main ponies"`), hence case-insensitive comparison.

## 3.8 `BehaviorGroup`

```
BehaviorGroup,⟨Number⟩,⟨Name⟩
```

| # | Field  | Kind | Class     | Range    | Default            |
|---|--------|------|-----------|----------|--------------------|
| 1 | Number | int  | Required  | 0 … 100  | —                  |
| 2 | Name   | text | Defaulted | non-blank | the number as text |

* Split with **QB**. Purely descriptive: gives a human name to a group number. The runtime only
  uses group *numbers*. Group `0` is always called "Any".
* Duplicate numbers: undefined (first or last may win in different UIs).

## 3.9 `Behavior`

A behavior is a mode the pony can be in: which images it shows, how it moves, for how long, and
what happens before/after. A pony MUST have at least one valid behavior to be usable.

```
Behavior,⟨Name⟩,⟨Chance⟩,⟨MaxDuration⟩,⟨MinDuration⟩,⟨Speed⟩,⟨RightImage⟩,⟨LeftImage⟩,⟨Movement⟩,
         ⟨LinkedBehavior⟩,⟨StartSpeech⟩,⟨EndSpeech⟩,⟨Skip⟩,⟨TargetX⟩,⟨TargetY⟩,⟨FollowTarget⟩,
         ⟨AutoSelectFollowImages⟩,⟨FollowStoppedBehavior⟩,⟨FollowMovingBehavior⟩,
         ⟨RightImageCenter⟩,⟨LeftImageCenter⟩,⟨PreventAnimationLoop⟩,⟨Group⟩,⟨FollowOffsetType⟩
```
(one physical line; wrapped here for readability). Split with **Q** (braces are literal).

### 3.9.1 Field table

| #  | Field                  | Kind       | Class     | Range / values                 | Default  | Trim |
|----|------------------------|------------|-----------|--------------------------------|----------|------|
| 1  | Name                   | name       | Required  | not blank                      | —        | no   |
| 2  | Chance                 | real       | Defaulted | 0 … 1                          | `0`      | —    |
| 3  | MaxDuration (s)        | real       | Defaulted | 0 … 300                        | `15`     | —    |
| 4  | MinDuration (s)        | real       | Defaulted | 0 … 300                        | `5`      | —    |
| 5  | Speed                  | real       | Defaulted | 0 … 30                         | `3`      | —    |
| 6  | RightImage             | path       | Required  | must exist                     | —        | no   |
| 7  | LeftImage              | path       | Required  | must exist                     | —        | no   |
| 8  | Movement               | enum §3.9.2| Defaulted | case-insensitive, untrimmed    | `All`    | no   |
| 9  | LinkedBehavior         | name       | Defaulted |                                | `""`     | yes  |
| 10 | StartSpeech            | name       | Defaulted |                                | `""`     | yes  |
| 11 | EndSpeech              | name       | Defaulted |                                | `""`     | yes  |
| 12 | Skip                   | bool       | Defaulted |                                | `False`  | —    |
| 13 | TargetX                | int        | Defaulted |                                | `0`      | —    |
| 14 | TargetY                | int        | Defaulted |                                | `0`      | —    |
| 15 | FollowTarget           | pony-id    | Defaulted |                                | `""`     | yes  |
| 16 | AutoSelectFollowImages | bool       | Defaulted |                                | `True`   | —    |
| 17 | FollowStoppedBehavior  | name       | Defaulted |                                | `""`     | yes  |
| 18 | FollowMovingBehavior   | name       | Defaulted |                                | `""`     | yes  |
| 19 | RightImageCenter       | vector     | Defaulted |                                | `0,0`    | —    |
| 20 | LeftImageCenter        | vector     | Defaulted |                                | `0,0`    | —    |
| 21 | PreventAnimationLoop   | bool       | Defaulted |                                | `False`  | —    |
| 22 | Group                  | int        | Defaulted | 0 … 100                        | `0`      | —    |
| 23 | FollowOffsetType       | enum       | Defaulted | `Fixed` \| `Mirror` (case-**sensitive**; RI also accepts `0`/`1`) | `Fixed` | — |

Additional load-time rule: if MinDuration > MaxDuration a warning is recorded but the values are
kept as written. The duration draw (§6.5.1) is `Min + U·(Max − Min)`, which still produces a value
between the two, so the effect is the same as if they were swapped.

> **Pitfall:** image centers MUST be quoted (`"44,46"`). Unquoted, `44,46` becomes two fields and
> every subsequent field shifts by one. The RI does not detect this: each shifted value is parsed
> as the field it lands in, so values that happen to be valid are silently used and the rest fall
> back to defaults with warnings. Example: `…,44,46,43,46,False,0,Fixed` gives both centers
> invalid (default), PreventAnimationLoop `43` invalid (default), **Group = 46** (valid!), and
> FollowOffsetType `False` invalid (default).

### 3.9.2 `Movement` values

| INI token (canonical spelling)  | Meaning (free movement axes)                                | Flags |
|---------------------------------|-------------------------------------------------------------|-------|
| `None`                          | does not move freely                                        | 0     |
| `Horizontal_Only`               | horizontal                                                  | H     |
| `Vertical_Only`                 | vertical                                                    | V     |
| `Diagonal_Only`                 | diagonal                                                    | D     |
| `Horizontal_Vertical`           | horizontal or vertical                                      | H\|V  |
| `Diagonal_horizontal` *(sic)*   | diagonal (shallow) or horizontal                            | D\|H  |
| `Diagonal_Vertical`             | diagonal (steep) or vertical                                | D\|V  |
| `All`                           | horizontal, vertical or diagonal                            | H\|V\|D |
| `MouseOver`                     | special: designated mouse-over behavior; does not move      | M     |
| `Sleep`                         | special: designated sleep behavior; does not move           | S     |
| `Dragged`                       | special: designated being-dragged behavior; does not move   | G     |

Matching is case-insensitive and exact (no trimming, no partial matches). The canonical writer
emits the spellings shown, including the lower-case `h` in `Diagonal_horizontal`.

The special values are exclusive (a behavior cannot be both `All` and `MouseOver`). A special
behavior is still an ordinary behavior: it can be picked at random unless `Skip` is `True`.
The exact angles and the selection procedure are in §6.6.2.

### 3.9.3 Field semantics

* **Name** — identity within this pony (case-insensitive). Names SHOULD be unique; a duplicated
  name makes every *reference* to it unresolvable (§3.14) although both behaviors still run.
* **Chance** — relative weight for random selection (§6.5.2). It is *not* a probability: all
  eligible behaviors' chances are summed and one is chosen proportionally. `0` means "only if
  nothing else is eligible" (see the zero-weight rule in §6.5.2).
* **Min/MaxDuration** — the behavior lasts a uniformly random time between the two.
* **Speed** — in "legacy units" of pixels per 30 ms. Pixels per second = `Speed × 1000 / 30`
  (≈ 33.33 × Speed); per simulation step (40 ms) = `Speed × 4/3`. Speed is in **screen pixels**
  and is not affected by the sprite scale option. A behavior is *stationary* iff `Speed = 0`
  and *moving* iff `Speed > 0`, **regardless of Movement**; this distinction drives fallback and
  follow-image selection (§6.4, §6.7.3).
* **RightImage / LeftImage** — the images shown when facing right/left. Usually animated GIFs;
  PNG is also supported (Chapter 7). Both may name the same file.
* **LinkedBehavior** — when this behavior's time runs out, start the named behavior instead of
  a random one. Chains end at a behavior without a valid link. Used to build sequences
  (`roll-start → roll → roll-end`). An unresolvable link behaves as "no link". The editor
  warns about cycles (it does not reject them); the runtime tolerates them (a cycle simply loops
  forever).
* **StartSpeech** — name of a speech (§3.11) spoken when the behavior is entered *with speech
  enabled for that transition* (§6.9.2; see also §6.5.1).
* **EndSpeech** — name of a speech spoken when the behavior ends because its time ran out.
  If the next behavior has a start speech, that speech replaces this one immediately (only one
  bubble is visible at a time).
* **Skip** — `True` excludes the behavior from random selection. It is then normally reached
  only by link, interaction, follow-image selection, or the special-state machinery — but the
  candidate cascade (§6.5.2) falls back to skipped behaviors when no non-skipped candidate
  exists (e.g. when picking a moving/stationary behavior for a custom destination).
* **TargetX / TargetY / FollowTarget** — define the *target mode* of the behavior:

  | FollowTarget | (TargetX,TargetY) | Target mode | Meaning |
  |--------------|-------------------|-------------|---------|
  | non-empty    | any               | **Pony**    | Seek an instance of the pony whose directory name equals FollowTarget, at offset (TargetX,TargetY) **pixels** from that pony's location point. |
  | empty        | ≠ (0,0)           | **Point**   | Seek the absolute point at (TargetX %, TargetY %) of the allowed screen area (0–100 per axis; values outside 0–100 target points outside the area). |
  | empty        | (0,0)             | **None**    | Move freely according to Movement. (Consequently the exact top-left corner cannot be targeted.) |

  In Pony mode the target instance is chosen **once, when the behavior starts** (§6.7.2), never
  the pony itself. If no instance is present at that moment, the behavior acts as mode None for
  its whole duration (free movement, own images) even if a target appears later. If the chosen
  target disappears mid-behavior, the pony switches to free movement (keeping its current
  directions). The pony moves toward the target at this behavior's Speed even if Movement is
  `None`; Movement only governs *free* movement.
* **FollowOffsetType** — in Pony mode, `Fixed` uses the offset as-is; `Mirror` negates the X
  offset while the *target* faces left (offsets are authored for a right-facing target), so
  `(-50,0)` + `Mirror` means "50 px behind the target".
* **AutoSelectFollowImages / FollowStoppedBehavior / FollowMovingBehavior** — while a behavior
  is in Pony or Point mode the pony does **not** display the behavior's own images. Instead it
  borrows images from another behavior (the *visual override*): one while stationary, one while
  moving. With auto-select `True` they are chosen automatically; with `False` the named
  behaviors are used (falling back to automatic selection if a name does not resolve). Only the
  images are borrowed; nothing else about those behaviors applies. See §6.7.3.
* **Right/LeftImageCenter** — custom anchor point in **image pixel coordinates** (unscaled) of
  the respective image. `0,0` means "use the natural center" (§7.3). The pony's on-screen
  *location* is the point where the current image's anchor sits; when the image changes (facing
  flip or behavior change) the new image is placed so its anchor lands on the same screen point.
  Authors set custom anchors (e.g. at the saddle) so ponies don't jump when images of different
  sizes swap. Values outside the image bounds are allowed.
* **PreventAnimationLoop** — `True` forces the animation to play once and hold its last frame,
  overriding the image's own loop count (§7.4).
* **Group** — behavior group number; 0 = "Any". While the current behavior is in group *g*,
  random selection only considers behaviors in group 0 or *g* (§6.5.2). Used to create "modes"
  (dressed/undressed) with explicit transition behaviors that link across groups.

## 3.10 `Effect`

An effect is a secondary, independent sprite spawned when a given behavior starts (dust clouds,
falling apples, sparkles).

```
Effect,⟨Name⟩,⟨BehaviorName⟩,⟨RightImage⟩,⟨LeftImage⟩,⟨Duration⟩,⟨RepeatDelay⟩,
       ⟨PlacementRight⟩,⟨CenteringRight⟩,⟨PlacementLeft⟩,⟨CenteringLeft⟩,⟨Follow⟩,⟨PreventAnimationLoop⟩
```
Split with **Q**.

| #  | Field                | Kind      | Class     | Range        | Default  |
|----|----------------------|-----------|-----------|--------------|----------|
| 1  | Name                 | name      | Required  | not blank    | —        |
| 2  | BehaviorName         | name      | Required  | not blank    | —        |
| 3  | RightImage           | path      | Required  | must exist   | —        |
| 4  | LeftImage            | path      | Required  | must exist   | —        |
| 5  | Duration (s)         | real      | Defaulted | 0 … 300      | `5`      |
| 6  | RepeatDelay (s)      | real      | Defaulted | 0 … 300      | `0`      |
| 7  | PlacementRight       | direction | Defaulted |              | `Any`    |
| 8  | CenteringRight       | direction | Defaulted |              | `Any`    |
| 9  | PlacementLeft        | direction | Defaulted |              | `Any`    |
| 10 | CenteringLeft        | direction | Defaulted |              | `Any`    |
| 11 | Follow               | bool      | Defaulted |              | `False`  |
| 12 | PreventAnimationLoop | bool      | Defaulted |              | `False`  |

**Direction tokens** (case-insensitive, untrimmed) and their anchor weights *(wx, wy)* within a
rectangle (0 = left/top, 1 = right/bottom):

| Token             | Anchor          | (wx, wy)   |
|-------------------|-----------------|------------|
| `Top_Left`        | top-left        | (0, 0)     |
| `Top`             | top-center      | (0.5, 0)   |
| `Top_Right`       | top-right       | (1, 0)     |
| `Left`            | middle-left     | (0, 0.5)   |
| `Center`          | center          | (0.5, 0.5) |
| `Right`           | middle-right    | (1, 0.5)   |
| `Bottom_Left`     | bottom-left     | (0, 1)     |
| `Bottom`          | bottom-center   | (0.5, 1)   |
| `Bottom_Right`    | bottom-right    | (1, 1)     |
| `Any`             | random          | see below  |
| `Any-Not_Center`  | random, not the center | see below |

Semantics:

* **BehaviorName** — the effect fires every time a behavior with this name (case-insensitive) is
  started *in a transition that starts effects* (§6.8.1). Several effects may share a behavior;
  they start in file order. Unlike other references, an effect matches **every** behavior
  carrying that name (no uniqueness requirement at run time).
* **Duration** — seconds the effect lives. `0` means "until the triggering behavior ends".
* **RepeatDelay** — if > 0, a new instance of the effect is spawned every RepeatDelay seconds
  for as long as the triggering behavior runs (in addition to the first one at start).
  Example: delay 1 s on a 5 s behavior → instances at t = 0, 1, 2, 3, 4 (and possibly 5).
* **Placement** — the point on the **pony's** current image rectangle where the effect is
  anchored. **Centering** — the point on the **effect's** image that is put on that anchor.
  The Right/Left variants are chosen by the pony's facing at the moment the effect spawns.
* **Follow** — `False`: the effect stays where it spawned. `True`: the effect re-anchors to the
  pony every frame.
* **Any / Any-Not_Center** — intended semantics: pick one of the nine (resp. eight non-center)
  discrete anchors uniformly when the effect spawns. **RI behavior:** both tokens draw *wx* and
  *wy* independently and uniformly from the continuous range [0, 1] each time the anchor is
  computed — i.e. a random point anywhere in the rectangle, center not excluded, and for a
  following effect re-drawn every frame (the effect jitters). **Recommendation:** implement the
  intended discrete semantics, drawn once per spawned instance.
* The effect's facing (which of its two images it shows) is fixed at spawn time and never
  changes, even for following effects. (The old techdoc claims following effects update their
  facing; the RI does not.)

Full runtime rules: §6.8.

## 3.11 `Speak`

A speech line: text shown in a bubble, optionally with a sound.

Two accepted shapes (split with **QB**):

```
Speak,⟨Text⟩                                             (2 fields: unnamed)
Speak,⟨Name⟩,⟨Text⟩[,⟨Sound⟩[,⟨Skip⟩[,⟨Group⟩]]]          (≥3 fields)
```
where ⟨Sound⟩ is either empty, a single (optionally quoted) file name, or a brace list
`{"x.mp3","x.ogg"}`.

| # | Field  | Kind  | Class     | Range   | Default |
|---|--------|-------|-----------|---------|---------|
| 1 | Name   | name  | Defaulted | not blank | `Unnamed` |
| 2 | Text   | text  | Required (may be empty) | | — |
| 3 | Sound  | see below | Defaulted |      | none    |
| 4 | Skip   | bool  | Defaulted |         | `False` |
| 5 | Group  | int   | Defaulted | 0 … 100 | `0`     |

Rules:

* **Exactly two fields** → the line is *unnamed*: field 1 is the text, the name is absent,
  and all other fields take their defaults. A truly unnamed speech can only ever be used as a
  random speech, never by reference.
* **Three or more fields with a blank name** → the name becomes the literal `Unnamed` (with a
  warning). That *is* a real name and can be referenced (and several such lines make it
  non-unique). Note that the canonical writer (§3.16) writes two-field unnamed speeches as
  `"Unnamed"`, so after an editor round trip they become named.
* **Sound**: the field (after QB removed any braces) is split again with **Q**; the **first entry
  whose extension is `.mp3`** becomes the sound file. If no entry ends in `.mp3` there is no
  sound — so a lone `"hello.ogg"` or `"hello.wav"` yields *no sound* in the RI. The `.mp3`
  comparison is case-insensitive on Windows and case-sensitive elsewhere in the RI;
  **Recommendation:** case-insensitive everywhere. The canonical form is
  `{"⟨stem⟩.mp3","⟨stem⟩.ogg"}` — the `.ogg` twin exists for Browser Ponies; an implementation
  MAY prefer the `.ogg` twin if it exists and it cannot decode MP3.
* A sound file that does not exist is a **non-fatal** issue: the speech stays, without sound.
* **Skip** — `True` excludes the line from random speech; it can still be referenced as a
  behavior's start/end speech.
* **Group** — a random speech is eligible only if its group is 0 or equals the current
  behavior's group.
* Text may contain commas when quoted. It can contain `"` only inside a brace-qualified field
  (`{He said "hi"}`), which the parser accepts but the canonical writer cannot produce.

## 3.12 `Interaction`

An interaction is a scripted encounter **initiated by this pony** (the file's owner) with one or
more *target* ponies when they come close.

```
Interaction,⟨Name⟩,⟨Chance⟩,⟨Proximity⟩,⟨Targets⟩,⟨Activation⟩,⟨Behaviors⟩,⟨ReactivationDelay⟩
```
Split with **QB**.

| # | Field                  | Kind    | Class     | Range       | Default |
|---|------------------------|---------|-----------|-------------|---------|
| 1 | Name                   | name    | Required  | not blank   | —       |
| 2 | Chance                 | real    | Defaulted | 0 … 1       | `0`     |
| 3 | Proximity (px)         | real    | Defaulted | 0 … 10000   | `125`   |
| 4 | Targets                | list of pony-id | Required | ≥ 1 entry | — |
| 5 | Activation             | enum    | Defaulted | see below   | `One`   |
| 6 | Behaviors              | list of name | Required | ≥ 1 entry | — |
| 7 | ReactivationDelay (s)  | real    | Defaulted | 0 … 3600    | `60`    |

* **Targets** — pony directory names (case-sensitive). Duplicates collapse. The initiator's own
  name may appear: it then interacts with *another instance* of itself (an instance never
  targets itself).
* **Activation** tokens:

  | Token                                   | Value |
  |-----------------------------------------|-------|
  | `One` (case-sensitive, surrounding whitespace ignored) | One   |
  | `Any` (case-sensitive, surrounding whitespace ignored) | Any   |
  | `All` (case-sensitive, surrounding whitespace ignored) | All   |
  | `False` or `random` (case-insensitive, untrimmed)  | One (legacy) |
  | `True` or `all` (case-insensitive, untrimmed, but *not* the exact spelling `All`) | Any (legacy) |
  | `0`, `1`, `2` (any integer parses; values other than 0–2 fall back to One with a warning) | One, Any, All (RI quirk) |
  | anything else                           | One, with a warning |

  Note the trap: `All` means **All** but `all`/`ALL` mean **Any**. FollowOffsetType (§3.9.1)
  uses the same canonical-name rules (case-sensitive, trimmed, integers accepted).
* **Behaviors** — names of behaviors to run; each participant independently picks one it owns
  (§6.10). Typically one name, often the first link of a chain present in every participant's
  `pony.ini`.
* **Chance** is rolled **every simulation step** (25 times per second) while the interaction is
  eligible, so even small values trigger quickly once ponies are in range.
* **Proximity** — maximum distance (inclusive) between the initiator's and a triggering
  target's *location points* — their image anchors (custom centers when set), not the geometric
  centers of their rectangles.
* **ReactivationDelay** — cool-down after the interaction ends, during which participants
  cannot *initiate* interactions (they may still be targets). A forced cancel caps it at 30 s.

Full runtime rules: §6.10.

## 3.13 `Scale` (deprecated)

`Scale,⟨factor⟩` — ignored. Preserve verbatim when rewriting.

## 3.14 Name references and resolution

| Reference                                   | Resolved against                         | Comparison        | Rule |
|---------------------------------------------|------------------------------------------|-------------------|------|
| Behavior.LinkedBehavior                     | this pony's behaviors                    | case-insensitive  | unique match |
| Behavior.StartSpeech / EndSpeech            | this pony's speeches                     | case-insensitive  | unique match |
| Behavior.FollowStopped/MovingBehavior       | this pony's behaviors                    | case-insensitive  | unique match |
| Effect.BehaviorName                         | this pony's behaviors                    | case-insensitive  | **every** match fires |
| Behavior.FollowTarget                       | directory names of *live instances*      | case-sensitive    | any instance except the pony itself |
| Interaction.Targets                         | directory names of *live instances*      | case-sensitive    | any instance except the initiator itself |
| Interaction.Behaviors                       | each participant's behaviors             | case-insensitive  | set membership |

**Unique match:** if exactly one entity matches, use it; if zero *or two or more* match, the
reference resolves to nothing. An empty name never resolves. Two-field (truly unnamed)
speeches never match; speeches named `Unnamed` (§3.11) do.

## 3.15 Implicit (derived) behaviors

These are computed once per pony instance from the behavior list (file order matters; "first"
means first in file order):

* **Stationary fallback for group g** — the first behavior satisfying, in order of preference:
  1. group = g, Speed = 0, Skip = False, target mode None;
  2. group = g, Speed = 0;
  3. Speed = 0, Skip = False, target mode None (any group);
  4. Speed = 0 (any group);
  5. otherwise the behavior with the lowest Speed (first one among ties).
* **Sleep behavior** (one per pony) — the first behavior with Movement `Sleep`, else the
  stationary fallback for group 0.
* **Mouse-over behavior for group g** (for every group number used by any behavior) — the first
  behavior in group g with Movement `MouseOver`, else the stationary fallback for g.
* **Drag behavior for group g** — the first behavior in group g with Movement `Dragged`, else
  the pony's first `Sleep`-movement behavior (any group), else the mouse-over behavior for g.

## 3.16 Canonical serialisation (writing `pony.ini`)

An editor or converter that writes `pony.ini` SHOULD produce the RI's canonical layout so files
round-trip with other tools:

1. All comment lines, in original order.
2. `Name,⟨DisplayName⟩` (unquoted).
3. `Categories,"t1","t2",…` (each tag quoted; a pony with no tags writes `Categories,` — note
   the trailing comma. RI defect: that line re-reads as one empty tag, so the next save writes
   `Categories,""`; **Recommendation:** ignore empty tags on read).
4. One `behaviorgroup,⟨n⟩,⟨name⟩` per group (identifier lower-case, name unquoted).
5. Behaviors:
   `Behavior,"name",chance,max,min,speed,"right.gif","left.gif",Movement,"linked","start","end",Skip,tx,ty,"follow",AutoSelect,"stopped","moving","rx,ry","lx,ly",PreventLoop,group,FollowOffsetType`
   — names/paths quoted; centers written `"0,0"` when unset; numbers in invariant culture
   using the general format with up to 15 significant digits (e.g. `0.35`, `15`, `1E-05`);
   booleans `True`/`False`; enums in the canonical spellings above. All durations are held in
   whole milliseconds, so values are rounded to 0.001 s on load and written back rounded.
6. Effects:
   `Effect,"name","behavior","right.gif","left.gif",duration,delay,PR,CR,PL,CL,Follow,PreventLoop`.
7. Speeches: `Speak,"name","text",,Skip,Group` without sound, or
   `Speak,"name","text",{"file.mp3","file.ogg"},Skip,Group` with sound (the `.ogg` name is
   derived from the `.mp3` name). Unnamed speeches are written with the name `Unnamed`.
8. Interactions:
   `Interaction,name,chance,proximity,{"t1","t2"},Activation,{"b1","b2"},delay` (name
   unquoted).
9. All invalid/unknown lines, verbatim.

Paths are written as bare file names. Values containing `"` (or `{`/`}` inside lists) cannot be
represented and MUST be rejected by an editor. The display name, behavior group names and
interaction names are written **unquoted** without validation, so a comma or `"` in them corrupts
the line; an editor SHOULD reject those characters (the RI editors do). The file is written as
UTF-8 with a BOM, with the platform's line ending.

## 3.17 Legacy `Ponies/interactions.ini`

Older releases kept all interactions in one shared file. A conforming implementation SHOULD
support a one-time migration at start-up:

* Each non-blank, non-comment line has the shape
  `⟨Name⟩,⟨Initiator⟩,⟨Chance⟩,⟨Proximity⟩,⟨Targets⟩,⟨Activation⟩,⟨Behaviors⟩,⟨ReactivationDelay⟩`
  — i.e. the `Interaction` layout with no identifier and an extra required **Initiator**
  (pony-id) in position 1.
* For each line that parses without fatal issues and whose `Ponies/⟨Initiator⟩/pony.ini`
  exists, append the canonical `Interaction,…` line (§3.16) to that file.
* Delete `interactions.ini`; if any lines could not be migrated, write just those lines back to
  a new `interactions.ini` (they are retried on every start).
* RI defect: the line is appended without first ensuring the target file ends with a line break;
  several shipped `pony.ini` files have no final newline, so the new line would be glued onto
  their last line. **Recommendation:** insert a line break if needed.
* The RI runs this migration whenever it loads the pony collection (including from the
  editors).

An implementation that does not want to write into the content directory MAY instead read the
legacy file and merge its interactions in memory.

## 3.18 Annotated example

From `Ponies/Applejack/pony.ini` (abridged):

```
Name,Applejack
Categories,"main ponies","mares","earth ponies"
Behavior,"stand",0.35,10,2.2,0,"stand_aj_right.gif","stand_aj_left.gif",MouseOver,"","","",False,0,0,"",True,"","","44,46","43,46",False,0,Fixed
Behavior,"walk",0.35,5,2.2,3,"trotcycle_aj_right.gif","trotcycle_aj_left.gif",Diagonal_horizontal,"","","",False,0,0,"",True,"","","0,0","0,0",False,0,Fixed
Behavior,"giddyup",0.02,1.4,1.2,1,"aj-rear-right.gif","aj-rear-left.gif",None,"gallop","giddyup_sound","",False,0,0,"",True,"","","50,48","49,48",False,0,Fixed
Behavior,"gallop",0.05,6,3,8,"aj-gallop-right.gif","aj-gallop-left.gif",Diagonal_Only,"","gallop_sound","",True,0,0,"",True,"","","74,39","55,39",False,0,Fixed
Behavior,"truck_twilight",0,6,6,0,"truck_drive_right.gif","truck_drive_left.gif",None,"truck_twilight2","","",True,0,0,"Twilight Sparkle",False,"truck_twilight","truck_twilight","125,122","124,122",False,0,Fixed
Behavior,"Conga",0,30,30,1.2,"congaapplejack_right.gif","congaapplejack_left.gif",Diagonal_horizontal,"","","",True,-37,-2,"Rainbow Dash",False,"stand","Conga","42,52","45,52",False,0,Mirror
Behavior,"Sleep",0.03,30,15,0,"sleep_right.gif","sleep_left.gif",Sleep,"","","",False,0,0,"",False,"","","35,27","34,27",False,0,Fixed
Behavior,"drag",0,6.9,6.9,0,"aj-drag-right.gif","aj-drag-left.gif",Dragged,"","","",True,0,0,"",True,"","","32,61","23,61",False,0,Fixed
Effect,"Apple Drop","gallop","apple_drop.gif","apple_drop.gif",3.3,0.8,Bottom,Bottom,Bottom,Bottom,False,False
Effect,"crystalspark","crystallized","sparkle.gif","sparkle.gif",0,0,Center,Center,Center,Center,True,False
Speak,"Unnamed #1","Hey there, Sugarcube!",,False,0
Speak,"giddyup_sound","Yeee...",,True,0
Speak,"Soundboard #23","Yee-haw!",{"yeehaw.mp3","yeehaw.ogg"},False,0
Interaction,AJ Truck,0.05,300,{"Twilight Sparkle"},One,{"truck_twilight"},300
```

Reading it:

* `stand` is a stationary (speed 0) behavior with weight 0.35, lasting 2.2–10 s. Its Movement
  is `MouseOver`, so it is also the pony's mouse-over behavior for group 0; because Skip is
  `False` it is also picked at random. Its anchor is pixel (44,46) of the right image and
  (43,46) of the left one.
* `walk` moves at 3 × 33.3 ≈ 100 px/s, diagonally (15°–45° from horizontal) or horizontally.
* `giddyup` (weight 0.02) says the skip-only speech `giddyup_sound` ("Yeee...") when it starts
  and, after 1.2–1.4 s, links to `gallop`. `gallop` has `Skip=True` so it is reached only via
  the link (§6.5.2 fallbacks aside); it says "Haw!" on entry, runs 3–6 s at ≈ 267 px/s diagonally, and while it runs the
  `Apple Drop` effect is spawned at the pony's bottom-center at start and every 0.8 s, each
  living 3.3 s and staying where it was dropped.
* `Interaction AJ Truck`: when a Twilight Sparkle instance is within 300 px, every step there
  is a 5 % chance that Applejack and that Twilight both switch to a behavior named
  `truck_twilight` (Applejack is the *initiator*, Twilight the *target*). Applejack's
  `truck_twilight` targets "Twilight Sparkle" but has speed 0, so she stays put — the destination
  is computed and she turns to face it, but a zero speed never moves her. It shows the images of
  `truck_twilight` itself both when stopped and moving (auto-select off), lasts 6 s, and links
  through `truck_twilight2…4` (each speaking a line on entry) to `truck_drive` (60 s), which has
  no link. Twilight runs her own `truck_twilight` chain from her own `pony.ini`. When a target's
  chain reaches a behavior with no link and that behavior ends, the target leaves the
  interaction and its 300 s cool-down starts; when the *initiator's* chain ends, it ends the
  interaction for every target still in it. Each participant's cool-down (no *initiating* of
  interactions for 300 s) starts when it leaves.
* `Conga` follows Rainbow Dash at offset (−37,−2) mirrored, i.e. just behind her whichever way
  she faces.
* `Sleep` is the sleep behavior; `drag` (Skip) is shown while the user drags Applejack.
