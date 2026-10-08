# 11. Conformance: Checklist, Test Vectors and Corpus Statistics

## 11.1 Minimum conformance checklist

A "desktop ponies" implementation conforms to this spec when it:

1. Loads every pony in the shipped corpus exactly as Chapter 3 describes: BOM handling, line
   classification, Q/QB tokenisation, typed fields with defaults, entity rejection on fatal
   issues, and pony rejection when no behavior survives.
2. Runs each pony with the 40 ms fixed-step simulation of Chapter 6, including weighted
   group-aware behavior selection, linked chains, start/end/random speech, free movement
   directions, following (pony and point targets, Mirror offsets, borrowed images), effects
   (placement, duration, repeat, follow), interactions (One/Any/All, cool-downs), the hover /
   drag / sleep states, and bounds enforcement (teleport or walk-back, rebounds).
3. Animates images per Chapter 7: per-behavior time index, zero-delay frame dropping, loop
   counts, prevent-loop, anchors (custom or natural centers), nearest-neighbour scaling, z-order
   by bottom edge, single-line speech bubbles.
4. Reads and writes profiles per Chapter 5 (at least the `options` and `count` lines).
5. Supports houses (Chapter 4) and the context-menu operations of Chapter 8 §8.8.3 that
   apply on the platform.

Games (Chapter 9), screensaver mode, manual control, window avoidance and the editors are
optional.

## 11.2 Tokeniser vectors (§3.3)

| Input line | Q fields | QB fields |
|-----------|----------|-----------|
| `Behavior,"stand",0.15,15,5,0,"a.gif","b.gif",MouseOver` | `Behavior` `stand` `0.15` `15` `5` `0` `a.gif` `b.gif` `MouseOver` | same |
| `x,"50,34",y` | `x` `50,34` `y` | same |
| `ab"c,d"ef` | `abc,def` | same |
| `Speak,"Yeehaw","Yee-haw!",{"yeehaw.mp3","yeehaw.ogg"},False,0` | `Speak` `Yeehaw` `Yee-haw!` `{yeehaw.mp3` `yeehaw.ogg}` `False` `0` | `Speak` `Yeehaw` `Yee-haw!` `"yeehaw.mp3","yeehaw.ogg"` `False` `0` |
| `Interaction,AJ Truck,0.05,300,{"Twilight Sparkle"},One,{"truck_twilight"},300` | (not used) | `Interaction` `AJ Truck` `0.05` `300` `"Twilight Sparkle"` `One` `"truck_twilight"` `300` |
| `a,` | `a` and an empty field | same |
| `a,"b` (unbalanced) | `a` `b` (retried with `"` appended) | same |
| `Behavior, "stand"` | `Behavior` ` stand` (leading space kept) | same |
| `Speak,"Hi, there"` | `Speak` `Hi, there` | same (2 fields → unnamed speech) |
| `a,{b,"c}` | `a` `{b` `c}` | `a` `b,"c` |

## 11.3 Field parsing vectors (§3.4–3.12)

| Line / field | Result |
|--------------|--------|
| Behavior Chance `1.5` | Chance = 0, warning |
| Behavior Group `101` | Group = 0, warning |
| Behavior Skip `true` / `TRUE` / ` False ` | true / true / false |
| Behavior Skip `yes` | false, warning |
| Behavior center `"44,46"` | custom anchor (44,46) |
| Behavior center `"0,0"` | natural center |
| Behavior center `44` (unquoted `44,46` shifts fields) | natural center, warning; following fields misaligned |
| `Behavior,"walk",0.5,5,2,3,"r.gif","l.gif",All` (9 fields) | valid; all later fields default |
| `Behavior,,0.5,5,2,3,"r.gif","l.gif",All` | rejected (blank name) |
| `Behavior,"x",0.5,5,2,3,"missing.gif","l.gif",All` | rejected (file not found) |
| `Behavior,"x",…,Diagonal_Horizontal` | movement Diagonal_horizontal (case-insensitive) |
| `Behavior,"x",…, All` (leading space) | movement All (default), warning |
| `Speak,"Hello"` | unnamed speech, text `Hello` |
| `Speak,"n","t",{"a.ogg","b.ogg"},False,0` | no sound |
| `Speak,"n","t","a.mp3",False,0` | sound `a.mp3` (if it exists; else no sound + warning) |
| `Speak,"n","t",,True,2` | skip-only line in group 2 |
| Interaction activation `All` / `all` / `True` / `False` / `random` / `2` | All / Any / Any / One / One / All |
| Interaction targets `{}` | rejected |
| `BehaviorGroup,1,` | group 1 named `1` |
| `Categories, mares` | tag ` mares` (leading space; never matches `mares`) |
| `Name,Twilight Sparkle, the Princess` | display name `Twilight Sparkle` |

## 11.4 Simulation vectors (Chapter 6)

* **Speed**: Speed 3 → 100 px/s → 4 px per step; Speed 8 → 266.67 px/s → 10.67 px/step;
  Speed 30 → 1000 px/s → 40 px/step.
* **Duration**: a behavior with Min = Max = 2 s started at step *k* expires at step *k* + 51
  (elapsed 2.04 s > 2 s). Min = Max = 0 expires at step *k* + 1.
* **Speech**: display name `Applejack`, text `Yee-haw!` → bubble `Applejack: "Yee-haw!"`
  (21 characters) → duration 0.5 + 21/15 = 1.9 s → hidden in the step at +1.92 s.
  `Hey there, Sugarcube!` → 34 characters → 2.7667 s → hidden at +2.80 s.
* **Random speech gap**: a random line is refused if fewer than 10 s have passed since the
  previous bubble's *end* (start + duration).
* **Weighted choice**: candidates (file order) A 0.35, B 0.35, C 0.02 → total 0.72;
  `U = 0.5` → r = 0.36 → A (0.35 < 0.36) no, B (0.70 ≥ 0.36) yes → **B**.
  Candidates all with Chance 0 → always the first.
* **Group filter**: current behavior in group 2 → candidates are group 0 and group 2 behaviors
  (not Skip, target reachable); if none, relax as in §6.5.2.
* **Interaction trigger probability**: Chance c per step at 25 steps/s → probability of
  starting within 1 s of being eligible = 1 − (1 − c)²⁵: c = 0.05 → 72 %; 0.1 → 93 %;
  0.25 → 99.9 %.
* **Natural centers** (§7.3):

  | Size (W×H) | right anchor | left anchor |
  |-----------|--------------|-------------|
  | 80×60     | (40, 30)     | (39, 30)    |
  | 80×61     | (40, 30)     | (39, 30)    |
  | 81×62     | (40, 30)     | (40, 30)    |
  | 81×64     | (40, 32)     | (40, 32)    |

  (cy: 29.5 → 30 and 31.5 → 32 by round-half-to-even; 30.5 → 30.)
* **Effect placement** (pony rectangle origin (200, 300), pony region 100×80, effect image
  20×10, scale 1):

  | Placement | Centering | Effect top-left |
  |-----------|-----------|-----------------|
  | Bottom    | Top       | (240, 380) |
  | Center    | Center    | (240, 335) |
  | Bottom_Right | Top_Left | (300, 380) |
  | Top       | Bottom    | (240, 290) |
* **Free-movement angle**: `Diagonal_horizontal` → |angle from horizontal| ∈ [15°, 45°);
  `Diagonal_Vertical` → (45°, 75°]; `Diagonal_Only`/`All` (diagonal pick) → (15°, 75°].
* **Point target**: TargetX = 50, TargetY = 50, allowed area (0, 0, 1920, 1040) → destination
  (960, 520) for the anchor.
* **Mirror offset**: offset (−37, −2), target facing left → destination = target + (37, −2).

## 11.5 Animation vectors (§7.4)

* Delays [40, 80, 40] ms (mixed), loop count 0, prevent-loop off, τ = 0, 40, …, 400 → frames
  0 0 1 1 0 0 1 1 0 0 1 (frame 2 never shown). Loop count 1 (or prevent-loop on) → 0 0 1 1 2 2 …
  (frame 2 held from τ = 160).
* Delays [50, 50, 50] (uniform), loop count 0, prevent-loop off: τ 0→0, 49→0, 50→1, 99→1,
  100→2, 149→2, 150→0.
* Delays [40, 40, 40]: τ = 40 → frame 1 (uniform boundary belongs to the next frame).
* Delays [100, 100, 100], loop count 1 (no NETSCAPE block): τ = 50 → 0; τ = 350 → 2 (held).
  Loop count 0: τ = 350 → 0.
* Prevent-loop with [100, 100, 100], loop count 0: τ = 250 → 2; τ = 350 → 2 (held).
* Delays [0, 0, 0] → only the last frame is kept (static).
* Delays [0, 50, 50] → two frames, mixed-duration lookup (frame 0 was dropped).

## 11.6 Corpus statistics (v1.69) — what matters most

Use these counts to prioritise and to sanity-check a loader (all files load without fatal
issues; there are no dangling references, missing files or duplicate behavior names).

| Item | Count |
|------|-------|
| Ponies (incl. Random Pony) | 311 |
| Behavior lines | 1,991 (353 with 23 fields — no FollowOffsetType; 1,638 with 24) |
| Speak lines | 1,064 (all 6-field form; 340 with a sound list, always `{"x.mp3","x.ogg"}`) |
| Effect lines | 198 (155 following) |
| Interaction lines | 62 in 32 ponies (One 51, Any 9, All 2) |
| BehaviorGroup lines | 53 (groups 1–9; 631 behaviors in a non-zero group, in 21 ponies) |
| Houses / games | 8 / 2 |

Behavior field usage:

| Feature | Behaviors |
|---------|-----------|
| Movement `None` / `Diagonal_horizontal` / `MouseOver` / `All` / `Horizontal_Only` | 732 / 465 / 278 / 168 / 148 |
| Movement `Sleep` / `Diagonal_Only` / `Diagonal_Vertical` / `Dragged` / `Vertical_Only` / `Horizontal_Vertical` | 54 / 46 / 44 / 27 / 25 / 4 |
| Speed 0 | 1,067 |
| Skip = True | 455 |
| Linked behavior | 523 |
| Follow target (pony) | 237 |
| Point target | 1 (Apple Bloom, 50,50) |
| Manual follow images (AutoSelect = False) | 267 |
| Start / end speech | 127 / 75 |
| PreventAnimationLoop = True | 195 |
| Custom centers ≠ 0,0 | 2,396 of 3,982 |
| FollowOffsetType Mirror | 54 |

Effect placement/centering values: `Center` 672, `Bottom` 34, `Left`/`Right` 17 each,
`Bottom_Right` 15, `Bottom_Left` 11, `Top` 10, `Top_Right` 7, `Top_Left` 5, `Any-Not_Center` 4,
`Any` 0.

Text: speech texts are ≤ 158 characters, 261 contain commas, none contain `"`, `{` or `}`;
20 files contain non-ASCII characters (curly quotes, ellipses, ♪ ♫ ♥, accented letters) — render
text as UTF-8 with a font that covers them.

Images: all GIF (2,709 files); loop counts are 0 (infinite) or 65535; 47 multi-frame GIFs have
no NETSCAPE block (play once); 44 contain zero-delay frames; delays of 1–2 cs occur in ~500
frames.
