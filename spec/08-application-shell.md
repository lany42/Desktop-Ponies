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
   exit.
2. Determine the initial profile: `default`, overridden by the command line (§8.3) or, when not
   in screensaver/autostart mode, by the first line of `Profiles/current.txt` (missing/empty
   file → `default`).
3. Populate the profile list and select the initial profile (this loads it, §5.3.4).
4. Load content (Chapter 2 §2.2) in the background, showing progress. Each loaded pony
   (including Random Pony) gets a selection entry; entries are sorted by directory
   (case-insensitive) with Random Pony first.
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
| `autostart`  | Load the `autostart` profile, start minimised/hidden from the taskbar, and launch ponies as soon as content is loaded. If the profile has no ponies selected, return to the menu with "You haven't created a profile with ponies to use. Create a profile named 'autostart', choose some ponies and save the profile…". |
| `/s`         | Screensaver mode (§8.11). When running as the `.scr` file this also auto-starts with the `screensaver` profile. |
| `/c`         | Show screensaver help ("The 'screensaver' profile will been loaded. Make changes to this profile to configure the screensaver…"), then the normal menu. |
| `/p`         | Exit immediately with status 0 (screensaver preview is not supported). |
| other        | Show usage, then start normally. |

RI defect: on the non-screensaver path the profile named in `current.txt` is selected *after*
the command line is processed, so `autostart` actually runs with the `current.txt` profile
unless that already is `autostart`. **Recommendation:** honour `autostart`.

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
* **Keyboard:** typing a letter jumps to the first visible entry whose name starts with it;
  Enter launches.
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
8. Pre-load all distinct image pairs of the involved bases (behaviors and effects) with progress
   feedback (optional).
9. Hide the menu, create the sprite surface (always-on-top per option, not in the taskbar),
   build usable interactions for all ponies (§6.10.1), start every sprite with the current
   time, and run the host loop.

## 8.6 Host loop (one frame)

Frame cadence: at most 25 frames per second (sleep for the rest of each 40 ms frame; never
catch up — a late frame simply runs late). The time passed to sprites is the elapsed real time
since the loop started, **excluding paused time**, sampled once per frame so every sprite sees
the same value.

Per frame, in this order:

1. **Add pending sprites** (new effects, house visitors, menu additions) and start each with the
   last frame's time.
2. Read the **cursor position** (screen coordinates) into the Context.
3. **Synchronise** the Context with the options (§5.1), then let every house run its visitor
   cycle with the current elapsed time (§6.13).
4. Apply viewer options (always-on-top, taskbar, display bounds).
5. **Screensaver exit check** (§8.11).
6. **Drag** (§8.8.2).
7. **Manual control** (§6.14.3).
8. Apply queued additions/removals; if any pony was added or removed, rebuild usable
   interactions for **all** ponies (§6.10.1).
9. **Update** every sprite with the frame time (in collection order) and remove expired ones.
10. If no sprites remain → return to the menu.
11. **Sort** the collection for drawing: houses first; otherwise ascending `Region.bottom`;
    stable.
12. **Sounds** (§8.7).
13. **Draw** (Chapter 7 §7.6).

**Pause** (used by editors and games, not by the desktop app): the time stops advancing, the
surface stops drawing (optionally hides); the loop keeps running.

## 8.7 Sound

* A pony requests a sound only when it starts a speech line that has a sound file (Chapter 6
  §6.9.1); the request is visible to the host for exactly the frame in which it was made.
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
| Take/Release Control – Player 1 | manual control (§6.14.3); assigning a pony that is player 2 clears player 2 (RI: Windows only) |
| Take/Release Control – Player 2 | same for player 2 |
| — | |
| Show Options | modeless options dialog, changes apply live |
| Return To Menu | stop, show the menu |
| Exit | stop, quit |

House menu: **Edit ⟨name⟩** (house dialog, §8.9) and **Remove ⟨name⟩** (removes the house and
clears the destination overrides of all ponies it was recalling).

In games (Chapter 9) the add/remove/sleep items are absent.

### 8.8.4 Keyboard

There are no global shortcuts. The only keyboard input is manual control (§6.14.3): player 1 =
arrow keys + right Shift (boost); player 2 = W A S D + left Shift. It is polled each frame from
the global keyboard state (RI: Windows only).

## 8.9 House options dialog

Edits a house definition **shared by every placed instance of that house**:

* Cycle time (5 – 3600 s), minimum (≥ 1) and maximum (≥ minimum) spawn, bias (0.1 – 0.9 in
  steps of 0.1, "Less Ponies ↔ More Ponies"), door position (click on the house image), and the
  visitor list (checked pony directories; "all checked" is stored as `all`).
* Save requires at least one visitor, updates the in-memory definition (re-scanning which
  on-screen ponies count as deployed) and rewrites `house.ini` (Chapter 4 §4.5).

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

* Entered with `/s`. Uses the `screensaver` profile (defaults if it does not exist; nothing is
  written to `current.txt`). If the profile selects no ponies, one random pony is shown.
* Background per ScreensaverStyle, one per monitor (all monitors), placed behind the ponies:
  `Transparent` (none), `SolidColor` (opaque; default black), `BackgroundImage` (aspect-fit,
  centered; black if it fails to load).
* The mouse cursor is hidden.
* **Exit:** the first frame records the cursor position; any later frame where the cursor has
  moved or any mouse button is down quits the program. Keyboard input does not exit.
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

* Unhandled errors: log to the console and `error.txt` ("Unhandled error in Desktop Ponies
  v⟨ver⟩ occurred ⟨UTC time⟩" + details), show a dialog, exit with status 1.
* Content errors never abort the program: bad lines, entities, ponies and houses are skipped
  (Chapter 2 §2.2, Chapter 3 §3.5).
* Missing/undecodable images make a sprite invisible; missing sounds are silent.
