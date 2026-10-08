# 10. Quirks, Defects and Decision Register

Every place where the reference implementation (RI) behaves surprisingly is listed here with the
spec section that defines it. For each, choose **Parity** (reproduce the RI) or **Fix** (follow
the recommendation). Items marked ★ are visible with the shipped content under default settings;
the rest show up only with particular options, platforms or UI actions, or with malformed or
third-party content. IDs are grouped by section (Q80–Q89 continue §10.2 and Q90–Q99 continue
§10.3).

## 10.1 Content parsing

| ID  | Quirk | Spec | Recommendation |
|-----|-------|------|----------------|
| Q1  | An unclosed `"` or `{` takes in the rest of the line, commas included, as one field (the RI retries with the closer appended). | §3.3.2 | Parity. |
| Q2  | `Infinity` fails every range check (default + warning), but `NaN` passes: in a duration field the pony is discarded, elsewhere it is stored (Chance `NaN` fires every step); profiles store `NaN` too. | §3.4, §5.3.2 | Fix: treat both as invalid (use the default). |
| Q3  | Out-of-range numbers are replaced by the default, not clamped (Chance 1.5 → 0). | §3.5 | Parity (content relies on nothing else). |
| Q4  | Booleans are case-insensitive (the old techdoc says case-sensitive). | §3.4 | Parity. |
| Q5  | A Speak sound is taken only from a `.mp3` entry; `.mp3` matching is case-sensitive off Windows. | §3.11 | Fix: case-insensitive; optionally fall back to `.ogg`/`.wav`. |
| Q6  | Activation `All` = All, but `all`/`ALL` = Any (legacy mapping); digits accepted. | §3.12 | Parity. |
| Q7  | FollowOffsetType is case-sensitive (`mirror` → Fixed + warning). | §3.9.1 | Either. |
| Q8 ★ | Effect placement/centering `Any`/`Any-Not_Center` pick a *continuous* random point, center not excluded, re-drawn every frame for following effects (1 shipped effect uses `Any-Not_Center` for all four placement/centering fields). | §3.10 | Fix: discrete anchors, drawn once per instance. |
| Q9 ★ | An effect's facing is fixed at spawn even when it follows the pony. | §3.10, §6.8.2 | Parity (155 shipped effects follow; changing this changes their look). |
| Q10 | Duplicate behavior/speech names make references to them resolve to nothing. | §3.14 | Parity (shipped content has no duplicates). |
| Q11 | `house.ini` `cycletime,0` passes validation, then makes the house fail to load; a house with no valid `image` is kept. | §4.2, §4.3 | Fix: treat < 1 as invalid; drop houses without an image. |
| Q12 | A missing `Houses/` directory crashes start-up. | §2.2 | Fix: treat as empty. |
| Q13 | File references use the host's case rules. | §3.4.1 | Fix on Linux: case-insensitive fallback lookup. |
| Q14 | GIF vs. static decoder chosen by the case-sensitive extension `.gif`. | §7.2 | Fix: sniff the signature. |
| Q15 | JPEG width/height are swapped by the header reader. | §7.3 | Fix. |
| Q16 | Image centers of exactly `0,0` cannot be expressed (mean "natural center"). | §3.9.3 | Parity (format limitation). |
| Q17 | `game.ini` is parsed strictly; unknown line types, a missing Description or Scoreboard, and undefined action numbers fail the game (some only at launch or at run time, fatally). | §9.2 | Fix: validate everything at load. |
| Q18 | Writers: a pony with no tags is written as `Categories,`, which re-reads as one empty tag; the legacy `interactions.ini` migration appends without first ending an unterminated last line. | §3.16, §3.17 | Fix: ignore empty tags on read; add a line break if needed. |
| Q19 | An image that fails to load is never skipped: a pre-load failure cancels the launch, a first-draw failure is fatal (GTK: both fatal). Causes include an invalid `.art` file and a GIF with 256 distinct colours that draws index 255. | §7.5, §7.5.1, §7.7 | Fix: support 256 colours, ignore an invalid `.art`, treat an unloadable image as empty. |

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
| Q28 | Speeds, follow offsets and house door positions are in screen pixels and ignore ScaleFactor. | §6.6.1, §6.7.1, §6.13 | Parity for speed; Fix (scale offsets/door) is reasonable. |
| Q29 ★ | GIF timing: no minimum delay, frame boundaries differ between equal- and mixed-duration GIFs, a GIF without NETSCAPE plays once, loop count N = N total plays. | §7.4 | Parity (content was tuned against it). |
| Q30 | Window avoidance/containment and keyboard manual control only work on Windows. | §6.12.5, §6.14.3, §8.8.3, §8.8.4 | Fix where the platform allows (X11); omit on Wayland. |
| Q31 | If several steps in one frame speak, only the last sound is played. | §6.9.1, §8.7 | Parity. |
| Q32 | A recall destination (house door) and a drag place the pony's anchor, not its image center, at the point. | §6.13, §6.6.5 | Parity. |
| Q33 | TimeFactor is quantised: one step per Δ = rms(40/f) whole ms, so the effective speed is 40/Δ (f = 3 → 3.077×, f = 9 and 10 → 10×; error up to ~12%). Drawn durations are rounded to whole ms too. | §6.3, §6.5.1 | Either (Parity where timing must match). |
| Q34 | The extend applied while walking back into the allowed region runs after the expiry check, so the behavior still expires mid-walk and its replacement keeps the in-region destination. | §6.5.3 | Either: extend before the check, or drop it. |
| Q35 | A behavior set during another pony's step (interaction participants at start, participants reset by a drag cancel) loses `behaviorChangedThisStep`: natural return is not evaluated and the first free vector keeps the old directions. | §6.6.2, §6.12.2 | Fix: clear the flag after the state update. |
| Q36 | "Follow the initiator" ignores expiry: an expired initiator is picked and dropped in the same step, so the behavior runs without a follow target although other ponies qualify. | §6.7.2 | Fix: skip an expired initiator. |
| Q37 | A drag released into hover (cursor still over the pony) starts no effects until hover is left and speaks no random line; a flagged step starts the effects of the behavior current at that point (waking while interacting: the sleep behavior's successor). | §6.8.1, §6.9.2 | Parity (refines Q21). |
| Q38 | Effect positions are computed when the host starts the instance in the next frame, so with catch-up steps non-following effects and repeats are anchored to the pony's end-of-frame state. | §6.8.2, §6.8.3 | Fix: capture the anchor at the spawning step. |
| Q39 | At ScaleFactor ≠ 1 a dragged effect or house is centred on its unscaled size, and an effect's Region size is rounded (the pony's is truncated; placement uses the unrounded size). | §6.8.2, §6.8.4 | Fix the drag offset; Parity for the 1 px rounding. |
| Q80 | Sleep entered during hover or drag overwrites the saved behavior, and hover exit ignores sleep: the hover behavior replaces the sleep behavior while asleep and the pre-hover behavior is lost (a random or the drag behavior follows on wake). | §6.11.2 | Fix: keep the saved behavior; no hover exit while asleep. |
| Q81 | The natural-return test counts edge contact with X, tests a zero-size X as a point or line (the disabled zone is the point (R.x, R.y) when R's origin ≠ (0,0)), and may measure a stale image. | §6.2, §6.12.2 | Fix: skip zero-size X; open overlap; current image. |
| Q82 | A house's first cycle ignores placement time: it runs as soon as the animation's elapsed time exceeds CycleInterval, often one frame after placement. | §6.13 | Either (Fix: first cycle one interval after placement). |
| Q83 | Recall arrival is tested against the pony's previous-step destination, which is stale on the recall frame: a pony standing on it is marked arrived and removed ~3 s later wherever it is. | §6.13 | Fix: test against the door, after one step with the override. |

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
| Q51 | Option dialogs accept narrower ranges than files and crash on out-of-range files. | §5.1, §8.9 | Fix: clamp for display. |
| Q52 | MaxPonyCount default (500) differs from its parse fallback (300). | §5.3.2 | Parity. |
| Q53 ★ | On Linux the RI does not scale sprites, ignores z-order, does not clamp bubbles, plays no sound, and its `.art` remap also tints transparent and semi-transparent pixels (14 shipped `.art` files remap black). | §7.5.1, §7.8, §8.7 | Fix all five. |
| Q54 | Context-menu handlers change simulation state from pool threads without synchronising with the animation loop: changes land mid-frame, overrides can be left stale and an added pony can be lost. | §1.5 | Fix: apply menu actions as commands at the start of a frame. |
| Q55 | A profile save that fails part-way (e.g. `"` in a value) has already truncated the file; main-window Save/Copy don't catch errors (fatal), and Save reloads and sets current only on a selection change. | §5.3.3, §5.3.5 | Fix: validate or write a temp file; report errors non-fatally. |
| Q56 | Loading `default` or a name with no file clears the session-only EnablePonyLogs and ShowPerformanceGraph. | §5.3.4 | Fix: leave them out of the reset. |
| Q57 | Delete with an empty name is fatal, a whitespace-only name passes, and deleting a name with no file reports success. | §5.3.5 | Fix: refuse blank names; report a missing profile. |
| Q58 | The options dialog's Load never clears earlier monitor-list selections, so monitors not in Screens can stay highlighted. | §5.3.5 | Fix: clear before applying. |
| Q59 | A `.scr` executable lists `screensaver` twice when its file exists; on other executables `/s` and `/c` promise that profile but use the `current.txt` one (`/c` text: "will been loaded"). | §8.2, §8.3 | Fix. |
| Q90 | The letter-jump key tries only the first matching entry; if pagination puts it on another page, nothing visible happens. | §8.4 | Fix: switch to its page. |
| Q91 | A sprite that expires during the update pass stays in the collection (counted, sorted, drawn) until the next frame's removal step. | §8.6 | Either (one-frame difference). |
| Q92 ★ | A pony clears its sound request at the start of every update, so lines spoken at Start, by external SetBehavior/Speak (games) and by an interaction target updated after its initiator are never heard. | §6.9.1, §8.7 | Fix: clear after the sound pass. |
| Q93 | House dialog: minimum spawn accepts 0 (Save then fails after a partial apply) and the maximum's floor stays 5; a Save refused for no visitors still applies the other fields; only the opened instance is re-scanned. | §8.9 | Fix: validate first; re-scan every instance. |
| Q94 | Non-fatal background-task errors overwrite `error.txt` under the fatal "Unhandled error" header; the WinForms sprite surface swallows its own UI-thread exceptions. | §8.13 | Fix: distinct header; Either for the surface. |

## 10.4 Games

| ID  | Quirk | Spec | Recommendation |
|-----|-------|------|----------------|
| Q60 ★ | The ball's direction is randomised when its speed class changes (usually right after a kick). | §9.5.3 | Fix. |
| Q61 ★ | Behavior depends on the exact game name `Ping Pong Pony`. | §9.6 | Either (better: explicit options). |
| Q62 ★ | Position start points are relative to the position box (Hoofball goalies start asymmetrically). | §9.2.4 | Parity (format semantics). |
| Q63 ★ | Kick cool-down is 3 s of wall-clock time (labelled "2 seconds"), independent of TimeFactor. | §9.5.2 | Either. |
| Q64 | Only the first ball is placed and served. | §9.5.1 | Fix if multi-ball games are wanted. |
| Q65 | Scale factor is applied inconsistently to goals (scoring and the AI's approach points use unscaled sizes, kick aim uses scaled). | §9.4, §9.5.5 | Fix. |
| Q66 | A manually controlled goalie whose have-ball list is `{5}` can never kick. | §9.5.2 | Either. |
| Q67 | Every `Games/` subdirectory is loaded as a game, so a folder without `game.ini` raises a warning each time the game dialog opens. | §9.1 | Fix: skip it silently. |
| Q68 | A `scoreboard` line is validated when read, so an invalid one fails the game even when a later line overrides it. | §9.2.1 | Fix: parse only the last one. |
| Q69 | A position Box is not validated at load: a malformed Box fails only at PLAY, and never for an unfilled optional position. | §9.2.4 | Fix: validate at load (cf. Q17). |
| Q70 | A failed launch aborts setup part-way ("Error loading games.") and the menu window hidden for the game dialog is never shown again. | §9.3 | Fix: return to the menu. |
| Q71 | Goals and the scoreboard are draggable effects; dragging a goal moves its scoring rectangle and the AI's aim and approach points for the rest of the game. | §9.3 | Fix: make them non-draggable (or keep deliberately). |
| Q72 | The AI finds goals in file order (opposing = first goal of another team, team 0 included), while scoring uses each team's last goal, so the AI can target a goal that never scores. | §9.2.5, §9.4 | Fix: one scoring goal per team for both. |
| Q73 | Bounce's left/right test compares the player's x with WA.width/2, ignoring WA.x (wrong on monitors or work areas not starting at x = 0). | §9.5.2 | Fix. |
| Q74 ★ | Bounce's steep-angle clamps test the raw, unnormalised θ: steep upward kicks from base 0 pass unclamped while their downward mirrors are clamped. | §9.5.2 | Fix: normalise θ first. |
| Q75 | The containment test (push-apart limit, ball in goal) uses a Region-sized box centred on the anchor, not the actual Region (≈ 1 px × ScaleFactor off; more with custom centers). | §9.5.4, §9.5.5 | Fix. |
| Q76 | Take/Release Control clears a player's SpeedOverride and DestinationOverride: it moves at behavior speed until the game next sends it to a point, which it does not do while the player keeps an action from the same list. | §8.8.3, §9.5.2 | Fix: restore the game speed. |

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
