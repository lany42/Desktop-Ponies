# 10. Quirks, Defects and Decision Register

Every place where the reference implementation (RI) behaves surprisingly is listed here with the
spec section that defines it. For each, choose **Parity** (reproduce the RI) or **Fix** (follow
the recommendation). Items marked ★ are visible with the shipped content; the rest only matter
for malformed or third-party content.

## 10.1 Content parsing

| ID  | Quirk | Spec | Recommendation |
|-----|-------|------|----------------|
| Q1  | An unbalanced line that still fails the final QB retry aborts loading of the whole pony. | §3.3.2 | Fix: drop the line only. |
| Q2  | `NaN` / `Infinity` parse as reals and then crash loading. | §3.4 | Fix: reject as invalid (use default). |
| Q3  | Out-of-range numbers are replaced by the default, not clamped (Chance 1.5 → 0). | §3.5 | Parity (content relies on nothing else). |
| Q4  | Booleans are case-insensitive (the old techdoc says case-sensitive). | §3.4 | Parity. |
| Q5  | A Speak sound is taken only from a `.mp3` entry; `.mp3` matching is case-sensitive off Windows. | §3.11 | Fix: case-insensitive; optionally fall back to `.ogg`/`.wav`. |
| Q6  | Activation `All` = All, but `all`/`ALL` = Any (legacy mapping); digits accepted. | §3.12 | Parity. |
| Q7  | FollowOffsetType is case-sensitive (`mirror` → Fixed + warning). | §3.9.1 | Either. |
| Q8 ★ | Effect placement/centering `Any`/`Any-Not_Center` pick a *continuous* random point, center not excluded, re-drawn every frame for following effects (4 shipped effects use `Any-Not_Center`). | §3.10 | Fix: discrete anchors, drawn once per instance. |
| Q9 ★ | An effect's facing is fixed at spawn even when it follows the pony. | §3.10, §6.8.2 | Parity (155 shipped effects follow; changing this changes their look). |
| Q10 | Duplicate behavior/speech names make references to them resolve to nothing. | §3.14 | Parity (shipped content has no duplicates). |
| Q11 | `house.ini` `cycletime,0` passes validation, then makes the house fail to load. | §4.2 | Fix: treat < 1 as invalid. |
| Q12 | A missing `Houses/` directory crashes start-up. | §2.2 | Fix: treat as empty. |
| Q13 | File references use the host's case rules. | §3.4.1 | Fix on Linux: case-insensitive fallback lookup. |
| Q14 | GIF vs. static decoder chosen by the case-sensitive extension `.gif`. | §7.2 | Fix: sniff the signature. |
| Q15 | JPEG width/height are swapped by the header reader. | §7.3 | Fix. |
| Q16 | Image centers of exactly `0,0` cannot be expressed (mean "natural center"). | §3.9.3 | Parity (format limitation). |
| Q17 | `game.ini` is parsed strictly; unknown line types, a missing Description or Scoreboard, and undefined action numbers fail the game (some only at launch or at run time, fatally). | §9.2 | Fix: validate everything at load. |

## 10.2 Simulation

| ID  | Quirk | Spec | Recommendation |
|-----|-------|------|----------------|
| Q20 | The interaction loop does not stop after starting one; a second may start in the same step and overwrite the first. | §6.10.2 | Fix: stop after the first. |
| Q21 ★ | Effects start only on natural expiry, on leaving a special state, at Start, or on an external SetBehavior — **not** when an interaction, a custom destination or a special state switches the behavior. | §6.8.1 | Parity by default (content may rely on it); Fix is a defensible choice. |
| Q22 | Moving exactly vertically toward a destination makes the pony face left. | §6.6.3 | Either. |
| Q23 | Activation `All` requires *every* instance of every target pony to be non-busy, and recruits all of them. | §6.10.2–3 | Parity. |
| Q24 | The initiator's first interaction behavior picks its follow target from all ponies (the involved set is still empty). | §6.7.2 | Either. |
| Q25 | Sleeping does not end an interaction; the sleeper stays a participant. | §6.11.1 | Either. |
| Q26 ★ | When all eligible candidates have Chance 0, the first one in file order is always chosen (e.g. follow-image auto-selection among zero-chance behaviors). | §6.5.2 | Parity. |
| Q27 | Pony avoidance is disabled when there are more than 25 sprites *including effects and houses*. | §6.12.5 | Either (performance guard). |
| Q28 ★ | Speeds, follow offsets and house door positions are in screen pixels and ignore ScaleFactor. | §6.6.1, §6.7.1, §6.13 | Parity for speed; Fix (scale offsets/door) is reasonable. |
| Q29 ★ | GIF timing: no minimum delay, frame boundaries differ between equal- and mixed-duration GIFs, a GIF without NETSCAPE plays once, loop count N = N total plays. | §7.4 | Parity (content was tuned against it). |
| Q30 | Window avoidance/containment and keyboard manual control only work on Windows. | §6.12.5, §6.14.3 | Fix where the platform allows (X11); omit on Wayland. |
| Q31 | If several steps in one frame speak, only the last sound is played. | §6.9.1 | Parity. |
| Q32 | A recall destination (house door) and a drag place the pony's anchor, not its image center, at the point. | §6.13, §6.6.5 | Parity. |

## 10.3 Application shell

| ID  | Quirk | Spec | Recommendation |
|-----|-------|------|----------------|
| Q40 | `autostart` is overridden by the profile in `current.txt`. | §8.3 | Fix. |
| Q41 | Loading a profile file does not reset options first; the allowed area is sticky. | §5.3.4 | Fix. |
| Q42 | A duplicate `count` key in a profile is fatal. | §5.3.2 | Fix: last wins. |
| Q43 | The "sound in screensaver" checkbox edits SoundEnabled. | §5.1 | Fix. |
| Q44 | AlphaBlending, DisplayInteractionErrors, SuspendForFullscreen are persisted but unused. | §5.1 | Keep the fields for file compatibility; implementing *SuspendForFullscreen* is optional. |
| Q45 | Filters don't restrict what is launched. | §8.4 | Either. |
| Q46 | Filter "Exactly" means "has all checked tags". | §8.4 | Either (rename the mode if you change semantics). |
| Q47 | The screensaver image path is written unquoted. | §5.3.3 | Fix: quote. |
| Q48 | Single-channel sound is checked once per frame. | §8.7 | Either. |
| Q49 | "Add Pony" ignores MaxPonyCount. | §8.8.3 | Either. |
| Q50 | Closing the surface during Exit can turn the request into Return To Menu. | §8.10 | Fix. |
| Q51 | Option dialogs accept narrower ranges than files and crash on out-of-range files. | §5.1 | Fix: clamp for display. |
| Q52 | MaxPonyCount default (500) differs from its parse fallback (300). | §5.3.2 | Parity. |
| Q53 ★ | On Linux the RI does not scale sprites, ignores z-order, does not clamp bubbles, and plays no sound. | §7.8, §8.7 | Fix all four. |

## 10.4 Games

| ID  | Quirk | Spec | Recommendation |
|-----|-------|------|----------------|
| Q60 ★ | The ball's direction is randomised when its speed class changes (usually right after a kick). | §9.5.3 | Fix. |
| Q61 ★ | Behavior depends on the exact game name `Ping Pong Pony`. | §9.6 | Either (better: explicit options). |
| Q62 ★ | Position start points are relative to the position box (Hoofball goalies start asymmetrically). | §9.2.4 | Parity (format semantics). |
| Q63 ★ | Kick cool-down is 3 s of wall-clock time (labelled "2 seconds"), independent of TimeFactor. | §9.5.2 | Either. |
| Q64 | Only the first ball is placed and served. | §9.5.1 | Fix if multi-ball games are wanted. |
| Q65 | Scale factor is applied inconsistently to goals (scoring uses unscaled sizes, aiming uses scaled). | §9.5 | Fix. |
| Q66 | A manually controlled goalie whose have-ball list is `{5}` can never kick. | §9.5.2 | Either. |

## 10.5 Differences from the repository's `techdoc.md`

The repository ships an older informal description of the file formats. Where it disagrees
with the RI, this spec follows the RI:

| Topic | techdoc.md says | RI (this spec) |
|-------|-----------------|----------------|
| Boolean fields | case-sensitive `True`/`False` | case-insensitive |
| Interaction Name | defaults to `""` | required (an interaction without a name is rejected) |
| Proximity | integer, "less than" | real, inclusive (`≤`) |
| Following effects | facing updates with the pony | facing fixed at spawn |
| Effect `Any` | one of 9 points | continuous random point (Q8) |
| Speak sound list | "Desktop Ponies will look for the .mp3 file" | first `.mp3` entry only; a single non-`.mp3` sound is ignored |
