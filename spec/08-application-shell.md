# 8. Application Shell

This chapter covers everything around the simulation: start-up, command line, the selection
menu, launching ponies, the per-frame host loop, input and context menus, sound, screensaver
mode and the house editor. UI layout is described only as far as it carries behavior; a
reimplementation is free to present it differently (e.g. a GTK/Qt window, a CLI, or a tray
icon).

---

## 8.1 Process model (reference implementation, non-normative)

* One process. A menu window (WinForms; a reduced GTK window on macOS) and, while ponies run, a
  sprite **viewer** surface plus a dedicated **animation thread** that runs the host loop
  (§8.6) at up to 25 frames per second.
* The menu is hidden while ponies run and shown again when they stop (unless the user chose
  Exit).
* Content paths are relative to the program directory (the RI changes the working directory
  to the executable's directory at start-up). See Chapter 2.
* **Recommendation (Linux):** resolve the content root from (in order) a command-line option,
  `$XDG_DATA_HOME/desktop-ponies`, `/usr/share/desktop-ponies`, the executable's directory; store
  profiles in `$XDG_CONFIG_HOME/desktop-ponies/Profiles` rather than inside the content tree.

## 8.2 Start-up sequence

1. If the `Ponies` directory is missing → error "The Ponies directory could not be found…" and
   exit. (`Houses/` is not checked; see step 4.)
2. Process the command line (§8.3). It never chooses the initial profile name, which depends
   only on the program file name (*screensaver executable* = the name ends in `.scr`,
   case-insensitive on Windows, case-sensitive elsewhere; §5.3.4):
   * screensaver executable → `screensaver`, whatever the arguments (`current.txt` is not read);
   * otherwise, on every start (also with `autostart` or `/s`) → the first line of
     `Profiles/current.txt`, not trimmed (missing file or directory → `default`).
3. Populate the profile list — `default`, then the base names of `Profiles/*.ini` in on-disk
   case and file-system order — and look the name up: it matches `default` in any case, any
   other entry only exactly (case-sensitive). Select the match, or `default` (entry 0) when there
   is none (no such `.ini` file, a difference in case, an empty line). Selecting loads the
   profile (§5.3.4) and, except on a screensaver executable, writes the selected name back to
   `current.txt`, so a name that was not found is replaced by `default`.
   On a screensaver executable the RI then force-loads `screensaver` without writing
   `current.txt` (defaults under that name if its file does not exist), appends a new
   `screensaver` entry to the list and selects it, which loads it again. RI defect: the test
   for an existing entry never matches, so when `screensaver.ini` exists the list shows
   `screensaver` twice. **Recommendation:** select the existing entry; append one only if none
   exists.
4. Load content (Chapter 2 §2.2) in the background, showing progress. Each loaded pony
   (including Random Pony) gets a selection entry; entries are sorted by directory
   (case-insensitive) with Random Pony first. RI defect (Q12): a missing `Houses/` directory
   makes this background load fail before any pony or house is loaded (and before the legacy
   interactions upgrade); the error is unhandled and fatal (§8.13), and occurs while the menu is
   already showing its loading state. **Recommendation:** treat a missing `Houses/` as an empty
   house set.
5. If no usable ponies loaded → error "Sorry, but you don't seem to have any usable ponies
   installed…", launching disabled.
6. Apply the profile's counts to the entries.
7. If auto-started (§8.3), launch immediately (§8.5).

## 8.3 Command line

Only the first argument is examined; it is trimmed, everything from the first `:` is dropped,
and it is compared case-insensitively.

| Argument     | Effect |
|--------------|--------|
| *(none)*     | Normal start. (When the program file name ends in `.scr`, show the screensaver help instead — Windows convention.) |
| `autostart`  | Load the `autostart` profile (but see the first RI defect below), start minimised/hidden from the taskbar, and launch ponies as soon as content is loaded. If the profile in use has no ponies selected, return to the menu with "You haven't created a profile with ponies to use. Create a profile named 'autostart', choose some ponies and save the profile…". |
| `/s`         | Set screensaver mode (§8.11); nothing else. On a screensaver executable (§8.2 step 2) start-up also starts minimised/hidden from the taskbar and launches as soon as content is loaded, with the `screensaver` profile. On any other executable the `current.txt` profile is selected (and written) as in a normal start, and nothing launches until Go. |
| `/c`         | Show "Screensaver Help" ("The 'screensaver' profile will been loaded. Make changes to this profile to configure the screensaver…"), then the normal (not auto-started) menu. `/c` selects no profile: the `screensaver` profile is used only on a screensaver executable; otherwise the `current.txt` profile (or `default`, §8.2). |
| `/p`         | Exit immediately with status 0 (screensaver preview is not supported). |
| other        | Show usage, then start normally. |

RI defects:

* The profile of §8.2 step 3 is selected *after* the command line is processed and replaces the
  profile loaded for `autostart`; ponies launch with the selected profile. So `autostart` uses
  the `autostart` profile only when `current.txt` names it with exactly that case and
  `autostart.ini` exists, and the "create a profile named 'autostart'" message appears whenever
  the selected profile has no counts. **Recommendation:** honour `autostart` (select and use
  the `autostart` profile).
* The usage text says `/s` uses the `screensaver` profile, but that, like the auto-start, happens
  only on a screensaver executable. **Recommendation:** treat `/s` the same whatever the
  executable name (use the `screensaver` profile and auto-start).
* On any other executable `/c` promises the `screensaver` profile, but the `current.txt` profile
  (or `default`) is loaded unless it happens to be `screensaver`. **Recommendation:** have `/c`
  select the `screensaver` profile whatever the executable name (adding the list entry if it is
  missing), or show the help only when that profile is actually loaded; fix the grammar ("will
  be loaded").

Usage text (RI): `<exe>` — "Starts Desktop Ponies normally."; `autostart` — "Start and show ponies
straight away, using the 'autostart' profile."; `/s` — "Start in screensaver mode. This uses the
'screensaver' profile."; `/c` — "Configure screensaver mode."; `/p` — "Preview screensaver mode.
This does nothing."

## 8.4 Selection menu

* **Per-pony count** (0 – 99999) for every pony plus Random Pony. Editing a count updates
  `PonyCounts` immediately (0 removes the entry). The total shown is the sum over installed
  ponies (+ Random Pony). Each entry previews the pony's first behavior's right image, animated,
  scaled by ScaleFactor, labelled with the **directory name**.
* **Random Pony entry** additionally exposes the *No Duplicates* option (NoRandomDuplicates).
* **"0 of All" / "1 of All"** set every entry currently visible under the filter (pagination
  ignored, Random Pony excluded) to 0 or 1.
* **Filter** by tags. Tag list = the 13 standard tags, then custom tags, then `[Not Tagged]`.
  With checked set T and `NT` = "[Not Tagged] checked", an entry is visible when:

  | Mode            | Visible if |
  |-----------------|-----------|
  | Show All        | always |
  | Any that match  | pony has any tag in T, or (pony has no tags and NT) |
  | Exactly         | if NT: pony has no tags and T is empty; else pony's tags ⊇ T (superset, despite the name) |
  | Except          | not (Any-that-match condition) |

  Filtering only affects what is shown; **launching uses all counts**, including hidden ones.
* **Pagination** (default on for non-Windows in the RI): N entries per page (default 7).
* **Keyboard** (handled for the whole menu, whichever control has focus; none of it applies
  while the profile selector has focus):
  * A letter (any Unicode letter) is consumed. Among the entries passing the tag filter
    (pagination ignored, Random Pony skipped), in list order, the first whose name (directory)
    starts with that letter (case-insensitive) is scrolled into view and its count field
    focused. RI quirk: only that first match is tried; with pagination on it may be on another
    page, and then nothing visible happens (the page does not change), even if a later match is
    on the current page. **Recommendation:** switch to the matching entry's page.
  * `#` opens the newer, document-based pony editor (§8.12) as a modal dialog; unlike the Pony
    Editor button, the menu stays visible. If the editor reports changes, the menu discards its
    entries, sets the filter to Show All and reloads everything (the RI re-runs the start-up
    sequence, §8.2).
  * Enter activates Go (launch).
* **Profile controls:** §5.3.5. **Options dialog:** edits §5.1 (modal from the menu; modeless
  and live from the pony context menu). **Custom filters dialog:** one tag per line; lines
  containing `"` are removed with a warning.
* **Other buttons:** Pony Editor (§8.12), Mini-games (Chapter 9), Community links (RI downloads an
  XML file of links and a "latest version" number; optional and out of scope).

## 8.5 Launch ("Go")

1. Collect `(directory, count)` for every entry with count > 0, and the Random Pony count `r`.
   `total = Σ counts + r`.
2. `total = 0` → in screensaver mode launch one random pony; otherwise "You haven't selected any
   ponies! Choose some ponies to roam your desktop first." and stay in the menu.
3. `total > MaxPonyCount` → "Sorry you selected {total} ponies, which is more than the limit
   specified in the options menu. Try choosing no more than {max} in total. (or, you can increase
   the limit via the options menu)" and stay.
4. Create the Context (Chapter 6 §6.2) from the options.
5. For each named pony, create `count` instances (directory matched exactly).
6. Random ponies: pool = all ponies (never Random Pony itself); if NoRandomDuplicates, remove
   every pony that has an explicit count and remove each pick from the pool after picking — if
   the pool runs out, fewer random ponies are created. Otherwise pick with replacement.
7. Named instances come first in the sprite collection, then random ones. No pony has a location
   yet; each picks a random spot when started (§6.15).
8. In screensaver mode create the background surfaces (§8.11). Create the sprite surface without
   showing it, applying always-on-top and show-in-taskbar from the options. Pre-load all
   distinct image pairs of the involved bases (behaviors, then effects) into it, with progress
   feedback (optional).
9. Hide the menu, build usable interactions for all ponies (§6.10.1), show the surface and run
   the host loop. The loop's first iteration only starts every sprite with time 0 and draws;
   frame updates (§8.6) begin with the second iteration. The RI forces the surface out of the
   taskbar just before first showing it (a Windows window-ordering workaround), but every frame
   update re-applies the ShowInTaskbar option (§8.6 step 4), so with that option on (the
   default on Windows) the surface is in the taskbar from the first frame update on.

## 8.6 Host loop (one frame)

Frame cadence: at most 25 frames per second (sleep for the rest of each 40 ms frame; never
catch up — a late frame simply runs late). The time passed to sprites is the elapsed real time
since the loop started, **excluding paused time**, sampled once per frame (at the start of
step 9) so every sprite sees the same value. Steps 1–8 still see the previous frame's value
(0 on the first frame update).

Per frame, in this order:

1. **Queue pending sprites** (new effects, house visitors, ponies added from the menu): the
   Context's pending list is queued as one add-and-start action (applied in step 8) and cleared.
   Nothing joins the collection yet, so these sprites are not seen by steps 2–7 (house visitor
   cycles, drag hit-testing).
2. Read the **cursor position** (screen coordinates) into the Context.
3. **Synchronise** the Context with the options (§5.1), then let every house already in the
   collection run its visitor cycle (§6.13) with the previous frame's time (the clock is
   re-sampled only in step 9), so house timing lags the sprites by one frame.
4. Apply viewer options (always-on-top, taskbar, display bounds).
5. **Screensaver exit check** (§8.11).
6. **Drag** (§8.8.2).
7. **Manual control** (§6.14.3).
8. Apply queued additions/removals in the order queued: step 1's batch, houses added from the
   menu (queued directly), and removals (menu removals, expiries). An addition appends each
   sprite to the end of the collection, then starts it with the previous frame's time. Removing
   a sprite expires it (no effect if already expired); removing a pony thereby expires its
   effects, whose removals complete in the same pass. If any pony was added or removed, rebuild
   usable interactions for **all** ponies (§6.10.1).
9. Sample the frame time and **update** every sprite with it (in collection order), including
   those just added: a new sprite is started with the previous frame's time and then updated
   with the new one in the frame it joins. (Actions queued since step 8, e.g. from the menu
   thread, are applied first and start with the new time.) Expiry does not remove a sprite: it
   only queues its removal. A sprite that expires here (an effect reaching its duration, effects
   expired when their pony changes behavior) stays in the collection for the rest of the frame
   — counted by step 10, sorted, visited by the sound pass (it cannot start a sound) and drawn
   with its last image and position — and is removed in step 8 of the next frame; sprites
   expired before step 8 (e.g. ponies recalled by a house in step 3) are removed in that frame's
   step 8. An implementation MAY drop expired sprites before step 10 (a one-frame difference).
   Sprites put on the pending list after step 1 (visitors deployed in step 3, effects created
   here) are queued in step 1 of the next frame and added in its step 8.
10. If no sprites remain → return to the menu.
11. **Sort** the collection for drawing: houses first; otherwise ascending `Region.bottom`;
    stable.
12. **Sounds** (§8.7).
13. **Draw** (Chapter 7 §7.6).

**Pause** (used by editors and games, not by the desktop app): the time stops advancing, the
surface stops drawing (optionally hides); the loop keeps running.

## 8.7 Sound

* A pony requests a sound only when it starts a speech line that has a sound file (Chapter 6
  §6.9.1). It holds at most one request: each speech line sets it to that line's sound file or
  to none, and the pony clears it as the very first action of every update, before any step
  runs. The host reads it once per frame, in the sound pass (§8.6 step 12). A request can
  therefore be heard only if it was made after that pony's update began in the current frame:
  during its own steps (behavior changes, hover, sleep, drag, an interaction it starts), or, for
  an interaction target, by an initiator updated later than the target in the same frame. Of
  several such requests only the last survives (Q31).
* RI defect: every other request is cleared before any sound pass sees it, so its sound never
  plays (the bubble still shows): the start line spoken when a pony is started (§6.15; initial
  ponies start in the loop's first iteration, which has no frame update or sound pass, added
  ponies in §8.6 step 8), speech from external `SetBehavior`/`Speak` calls (game logic in §8.6
  step 3, editors), and the start line of an interaction target updated after its initiator. A
  request the sound pass declines (below) is also dropped, not retried. **Recommendation:**
  clear a request only after the host's sound pass has read it.
* Gate: in screensaver mode `ScreensaverSoundEnabled`, otherwise `SoundEnabled` (and speech must
  be enabled for any request to exist).
* **SoundSingleChannel = true**: if a previously started sound is still playing, every request in
  this frame is **dropped** (not queued). (RI quirk: the check happens once per frame, so several
  requests in the same frame may all start.)
* **SoundSingleChannel = false**: a request from a pony whose own previous sound is still playing
  is dropped; other ponies may play concurrently.
* Each sound plays independently at `SoundVolume` (applied at start). Undecodable or missing
  files are silently ignored. All sounds stop when ponies are stopped.
* Formats: the corpus contains only MP3 (every speech also names an `.ogg` twin that is **not
  shipped**). A Linux implementation MUST decode MP3 to play the shipped sounds.
* The RI plays sound only on Windows; on Linux it is silent. Supporting sound on Linux is
  RECOMMENDED.

## 8.8 Pointer input and context menus

### 8.8.1 Hit-testing

`ClosestUnderPoint(kind, p)`: among sprites of the given kind whose `Region` contains `p`
(right/bottom edges exclusive), pick the one whose region center is nearest to `p` (ties → the
earlier in collection order).

### 8.8.2 Dragging

Every frame: if the surface has focus and the left button is held:
* if dragging is enabled and nothing is being dragged: candidate = closest **pony** under the
  cursor, else the closest other draggable sprite (effects and houses); set its `Drag` flag.

Otherwise (button released or focus lost): clear the `Drag` flag of the dragged sprite.
A dragged pony places its anchor at the cursor (§6.6.5); a dragged non-following effect or house
centers its unscaled image size on the cursor. Hover detection needs no event: ponies compare the
cursor position with their own rectangle each step.

### 8.8.3 Right-click menus

Right-click on a pony (closest pony under the pointer) → **pony menu**; else on a house →
**house menu**; effects have no menu. Labels use the pony's **directory** name / house name.

Pony menu (top to bottom):

| Item | Action |
|------|--------|
| Remove ⟨dir⟩ | remove this instance |
| Remove Every ⟨dir⟩ | remove all instances of this pony |
| — | |
| Sleep/Pause ↔ Wake up/Resume | toggle this pony's `Sleep` |
| Sleep/Pause All ↔ Wake up/Resume All | toggle a host flag and set every current pony's `Sleep` to it (ponies added later are awake; the label follows the flag) |
| — | |
| Add Pony ▸ | `Random Pony`, then one submenu per tag that has ponies (standard tags, then custom tags), then `[Not Tagged]`; picking adds one new instance at a random location (MaxPonyCount not checked). The tree is built once at launch. |
| Add House ▸ | one entry per house; adds it at a random position (§6.13) |
| — | |
| Take/Release Control – Player 1 | manual control (§6.14.3): empties the slot if this pony holds it, else gives the slot to this pony. Every change of the slot (release or reassignment) first clears the previous holder's SpeedOverride and DestinationOverride (not its follow-target or movement overrides). Taking player 2's pony empties the player-2 slot without clearing anything. (RI: Windows only; also present in games) |
| Take/Release Control – Player 2 | same for player 2 |
| — | |
| Show Options | modeless options dialog, changes apply live |
| Return To Menu | stop, show the menu |
| Exit | stop, quit |

RI defect: in a game a player's speed is its SpeedOverride (Chapter 9 §9.5.2, "go to a point"),
so releasing or moving control drops the game speed: the pony moves at its behavior speed until
the game next sends it to a point (the chase/lead-ball actions set only a follow target), and
its cleared destination is not re-issued while it keeps an action from the same list.
**Recommendation:** after releasing a game player, restore its game speed (or have the game
re-apply speed and destination every frame).

House menu: **Edit ⟨name⟩** (house dialog, §8.9) and **Remove ⟨name⟩**, which expires the house
(nothing happens if it already has); it is removed at the next queued-action step (§8.6 step 8).
On expiry the house sets the DestinationOverride of every pony it tracks to none and stops
tracking them: its deployed ponies (§6.13: those it spawned, plus on-screen visitors counted when
it was placed or its visitor list was last saved, §8.9), ponies walking to its door, and recalled
ponies already waiting out the 3 s delay at the door. The last two groups are not removed; they
stay on screen and resume normal behavior. The clear is unconditional, so an override set by
something else is lost too (manual control and another recalling house set theirs again on
their next frame).

In games (Chapter 9) the add/remove/sleep items are absent.

### 8.8.4 Keyboard

There are no global shortcuts. The only keyboard input is manual control (§6.14.3): player 1 =
arrow keys + right Shift (boost); player 2 = W A S D + left Shift. It is polled each frame from
the global keyboard state (RI: Windows only).

## 8.9 House options dialog

Edits a house definition **shared by every placed instance of that house**:

* Cycle time (5 – 3600 s), minimum and maximum spawn (below), bias (0.1 – 0.9 in steps of 0.1,
  "Less Ponies ↔ More Ponies"), door position (click on the house image), and the visitor list
  (checked pony directories; "all checked" is written to the file as `all`).
* The spawn limits are two linked fields: changing the minimum sets the maximum's lower bound to
  it (raising the maximum if needed); changing the maximum sets the minimum's upper bound to it
  (lowering the minimum if needed). In the RI, before the house's values are loaded the minimum
  ranges 0 – 50 (value 1) and the maximum 5 – 9999 (value 50); the dialog then assigns
  `minspawn`, then `maxspawn`, and an out-of-range assignment is an unhandled error (§8.13).
  RI defect: the minimum accepts 0, and the maximum's floor stays 5 until the minimum changes
  (loading `minspawn` 1 does not change it). Opening the dialog therefore fails for files that
  load fine (Chapter 4 §4.2 accepts each value in 1 – 9999 and does not check min ≤ max) when
  `minspawn` > 50, when `minspawn` = 1 and `maxspawn` < 5, or when `maxspawn` < `minspawn`; and
  saving with minimum 0 fails with an unhandled error after cycle time and door position were
  already applied in memory (`house.ini` is not written). **Recommendation:** minimum 1 – 9999,
  maximum from the minimum to 9999; set the bounds before loading and clamp loaded values
  (raising the maximum to the minimum if needed); validate everything before applying any field.
* Save, in order:
  1. copy cycle time, door position, minimum/maximum spawn and bias into the shared definition
     (effective at once for every placed instance);
  2. if no visitor is checked, show "You must select at least one visitor!" and stop (visitor
     list and `house.ini` unchanged);
  3. replace the in-memory visitor list with the checked directories (explicit names even when
     all are checked, so deploy and recall stop using the `all` rules of §6.13; `all` appears
     only in the file);
  4. re-scan the *deployed* set (§6.13) of the instance the dialog was opened from only: every
     on-screen pony whose directory is in the new list (its recalling and arrived ponies are
     untouched; other instances keep their old sets);
  5. rewrite `house.ini` (Chapter 4 §4.5); if that fails an error is shown and the in-memory
     changes stay.

  RI defect: a refused save still applies step 1, and the re-scan covers only one instance.
  **Recommendation:** check the visitors before changing anything, apply all changes together,
  and re-scan every placed instance that shares the definition.

## 8.10 Stopping

* *Return To Menu*, the last sprite disappearing, or the user closing the surface → back to the
  menu. *Exit* or screensaver dismissal → quit.
* A display configuration change (resolution/monitor) while running stops the ponies and returns
  to the menu with "You have been returned to the menu because your screen resolution has
  changed." (A new implementation MAY instead just recompute the allowed area.)
* On stop: all sprites expire, sounds stop, the surface closes.
* RI defect: closing the surface programmatically during *Exit* can overwrite the exit request
  with *Return To Menu*. **Recommendation:** only set "return to menu" if no request is pending.

## 8.11 Screensaver mode

* Entered with `/s` (§8.3), which only sets the mode. On a screensaver executable (§8.2 step 2)
  the `screensaver` profile is used (defaults if it does not exist), `current.txt` is never
  written (§5.3.4), and ponies launch as soon as content is loaded. On any other executable the
  `current.txt` profile is used and written as in a normal start, and nothing launches until Go.
  The rules below apply to any launch while the mode is set. If the selection totals 0, one
  random pony is launched (§8.5 step 2).
* Background per ScreensaverStyle, one per monitor (all monitors), placed behind the ponies:
  `Transparent` (none), `SolidColor` (opaque; default black), `BackgroundImage` (aspect-fit,
  centered; black if it fails to load).
* The mouse cursor is hidden (the RI never shows it again).
* **Exit:** the first frame records the cursor position; any later frame where the cursor has
  moved or any mouse button is down quits the program (it does not return to the menu). Keyboard
  input does not exit.
* Sound uses ScreensaverSoundEnabled.

## 8.12 Pony editors (non-normative summary)

The RI ships two editors (a grid-based one and a newer document-based one) that load content in
*permissive* mode (entities with fatal issues are kept), show parse issues and reference
problems (Chapter 3 §3.5, §3.14), preview a single pony in a panel (with avoidance features off,
teleport on, and the preview panel as the allowed region), and save with the canonical writer
(Chapter 3 §3.16). Notable editor rules worth keeping in any editor: names may not contain
`"`, `,`, `{` or `}`; names must be unique per type within a pony; renaming does not update
references in other ponies; the image-center tool mirrors a center to the other image with
`x' = width − 1 − x`.

## 8.13 Error handling

* Unhandled errors (in the RI's WinForms path also exceptions on the menu's UI thread) are
  fatal: write to the console ("FATAL: An unexpected error occurred and Desktop Ponies must
  close.", "Error in Desktop Ponies v⟨ver⟩ occurred ⟨UTC time⟩", details) and overwrite
  `error.txt` in the program directory (UTF-8: "Unhandled error in Desktop Ponies v⟨ver⟩
  occurred ⟨UTC time⟩", a blank line, details); except on macOS, show a modal error dialog ("An
  unexpected error occurred and Desktop Ponies must close. Please report this error so it can be
  fixed."); then exit with status 1 (RI: not while a debugger is attached). UTC times use the
  form `yyyy-MM-dd HH:mm:ssZ`.
* Unobserved background-task errors are only logged (console header "Unobserved Task
  Exception", and `error.txt` with the same "Unhandled error" header, overwriting it); no
  dialog, and the program keeps running. **Recommendation:** use a distinct `error.txt` header
  for errors that are not fatal.
* Non-fatal errors reported by explicit catch sites go to the console only ("WARNING:
  ⟨message⟩"), plus a warning dialog except on macOS.
* RI quirk: the WinForms sprite surface's own UI thread silently ignores exceptions raised while
  handling its window messages.
* Content errors never abort the program: bad lines, entities, ponies and houses are skipped
  (Chapter 2 §2.2, Chapter 3 §3.5). Exception: a missing `Houses/` directory (§8.2 step 4).
* Missing/undecodable images make a sprite invisible; missing sounds are silent.
