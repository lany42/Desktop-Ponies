# 5. Options and Profiles

User settings are global for the session and are grouped into named **profiles** stored as
files. The simulation reads them through the Context (Chapter 6 §6.2), which is refreshed from
the options every frame, so changes made while ponies run apply immediately.

---

## 5.1 Option model

| Option                     | Type            | Default         | Valid range (file) | Runtime effect |
|----------------------------|-----------------|-----------------|--------------------|----------------|
| SpeechEnabled              | bool            | true            |                    | Gates all speech bubbles — and therefore all sounds (§6.9). |
| SpeechChance               | real            | 0.01            | 0 – 1              | Probability of a random line after a natural behavior change (§6.5.3). UI shows it as an integer percentage. |
| CursorAwareness            | bool            | true            |                    | Enables *both* the hover (mouse-over) state and cursor avoidance (§6.11, §6.12.6). |
| CursorAvoidanceRadius      | real (px)       | 100             | 0 – 10000          | §6.12.6 |
| DraggingEnabled            | bool            | true            |                    | §6.11.2, §8.8 |
| InteractionsEnabled        | bool            | true            |                    | §6.10 |
| *DisplayInteractionErrors* | bool            | false           |                    | persisted only, no effect |
| ExclusionZone              | 4 × real        | (0,0,0,0)       | each 0 – 1         | normalised (x, y, w, h) sub-rectangle of the allowed area that ponies avoid (§6.2, §6.12). Zero size = none. Every assignment trims w and h so that x + w ≤ 1 and y + h ≤ 1 (§5.3.2). |
| ScaleFactor                | real            | 1               | 0.25 – 4           | Sprite size multiplier (§6.2.1). |
| MaxPonyCount               | int             | 500             | 0 – 10000 (fallback **300**) | Launch refuses more (§8.5); houses stop deploying at this count. Not enforced by the "Add Pony" menu. |
| *AlphaBlending*            | bool            | true            |                    | persisted only, no effect |
| EffectsEnabled             | bool            | true            |                    | §6.8 |
| WindowAvoidance            | bool            | false           |                    | §6.12.5 (Windows-only in RI) |
| PoniesAvoidPonies          | bool            | false           |                    | §6.12.5 |
| WindowContainment          | bool            | false           |                    | §6.12.5 (Windows-only in RI) |
| TeleportEnabled            | bool            | false           |                    | Out-of-bounds ponies teleport back instead of walking (§6.12.3). |
| TimeFactor                 | real            | 1               | 0.1 – 10           | Simulation speed (§6.3). UI offers 0.1 – 4.0. |
| SoundEnabled               | bool            | true            |                    | §8.7 (normal mode) |
| SoundSingleChannel         | bool            | false           |                    | `false` = one sound per pony at a time; `true` = one sound globally (§8.7) |
| SoundVolume                | real            | 0.75            | 0 – 1              | §8.7 |
| AlwaysOnTop                | bool            | true            |                    | sprite surface stays above other windows |
| *SuspendForFullscreen*     | bool            | true            |                    | persisted only, no effect (an intended feature: pause while a full-screen app runs) |
| ScreensaverSoundEnabled    | bool            | true            |                    | §8.7 (screensaver mode). RI defect: the options dialog's "Enable Sounds in Screensaver mode" checkbox (shown only on Windows when the current profile is `screensaver`) is initialised from and writes **SoundEnabled**, not this option, and is not kept in sync with the normal Sound checkbox; this option can only be changed by editing the file. **Recommendation:** bind that checkbox to ScreensaverSoundEnabled. |
| ScreensaverStyle           | enum            | Transparent     | `Transparent`, `SolidColor`, `BackgroundImage` | §8.11 |
| ScreensaverBackgroundColor | ARGB int32      | 0               |                    | §8.11 (alpha forced to opaque when used) |
| ScreensaverBackgroundImage | path            | ""              |                    | §8.11 |
| NoRandomDuplicates         | bool            | true            |                    | §8.5 |
| ShowInTaskbar              | bool            | true on Windows, else false |        | whether the sprite surface has a taskbar entry |
| AllowedArea                | rectangle or none | none          | w, h > 0 to be set | §5.2 |
| Screens                    | list of monitors | primary only   | never empty        | §5.2 |
| BackgroundColor            | ARGB int32      | 0 (transparent) |                    | fill colour of the sprite surface (debug aid) |
| PonyCounts                 | map dir → int   | empty           | int > 0            | how many of each pony to launch; key `Random Pony` = number of random ponies |
| CustomTags                 | list of strings | empty           |                    | extra filter tags after the 13 standard ones |
| EnablePonyLogs             | bool            | false           | not persisted      | debug: per-pony event log + live sprite table |
| ShowPerformanceGraph       | bool            | false           | not persisted      | debug overlay |

**Options dialog ranges (RI).** The dialog has no OK/Cancel; each change is written to the live
options immediately. Each numeric control maps linearly to its option:

| Option                | Control                          | UI range    | Step (arrows / page) | Notes |
|-----------------------|----------------------------------|-------------|----------------------|-------|
| MaxPonyCount          | spin box                         | 10 – 10000  | 10                   | any typed integer in range is accepted |
| CursorAvoidanceRadius | spin box (px, 0 decimals shown)  | 0 – 10000   | 5                    | enabled only while CursorAwareness is on |
| SpeechChance          | spin box, integer percent        | 0 – 100 %   | 1 %                  | enabled only while SpeechEnabled is on |
| ScaleFactor           | slider (position / 20)           | 0.25 – 4    | 0.05 / 1.25          | label `0.##x`; disabled (still shows the value) off Windows |
| TimeFactor            | slider (position / 10)           | 0.1 – 4.0   | 0.1 / 1.0            | label `0.0x` |
| SoundVolume           | slider (position / 100)          | 0.10 – 1.00 | 0.01 / 0.1           | label shows volume × 10 (`1.0` – `10.0`); enabled only while SoundEnabled is on; the sound group is hidden when sound is unavailable |

When the dialog fills its controls, each value is converted with round-half-to-even and is not
clamped; the option itself is not rewritten until the user edits the control. RI defect: the file
accepts values outside these ranges (MaxPonyCount 0 – 9, TimeFactor above ≈ 4.05, SoundVolume
below ≈ 0.095), and such a value makes the control assignment fail with an uncaught error
(§8.13) when the dialog opens or reloads the profile. **Recommendation:** clamp the displayed
value to the UI range without writing it back unless the user changes the control.

## 5.2 Allowed area

The allowed area `R` (Chapter 6 §6.12) is computed every frame:

* If **AllowedArea** is set: `AllowedArea ∩ (union of all monitors' full bounds)`.
* Otherwise: the union (bounding rectangle) of the **work areas** (screen minus panels/taskbars)
  of the monitors in **Screens**, in order: R starts as the first monitor's rectangle and each
  later one extends it (min left/top, max right/bottom). A monitor whose work area is exactly
  (0, 0, 0, 0) contributes its full bounds instead; a zero-size work area at any other position
  is used as is and still extends R to that point.

Note this is a single bounding rectangle: with monitors of different sizes it can include
off-screen gaps. Implementations MAY use a more precise multi-rectangle model but must then
define bounds enforcement accordingly.

## 5.3 Profiles on disk

### 5.3.1 Location and naming

* Directory `Profiles/` (relative to the program's content root, Chapter 2).
* A profile named *N* is stored in `Profiles/N.ini`.
* Reserved names (case-insensitive):
  * `default` — built-in defaults; never stored, cannot be saved, copied over or deleted.
  * `screensaver` — used in screensaver mode (§8.11).
  * `autostart` — used when launched with the `autostart` argument (§8.3).
* A valid profile name is non-empty, not `default`, and contains no character invalid in a file
  name on the host OS.
* `Profiles/current.txt` holds the name of the last selected profile (UTF-8, normally with BOM,
  no trailing newline). The shipped file contains `default`.

### 5.3.2 File format

UTF-8 (BOM written, BOM skipped on read), one record per line, lines split with the **Q**
splitter (§3.3; quotes protect commas). The **identifier (field 0) is matched case-sensitively**.
Unknown lines are ignored.

```
options,⟨36 values⟩
monitor,"⟨monitor device name⟩"        (0..n)
count,"⟨pony directory⟩",⟨n⟩            (0..n)
tag,"⟨custom tag⟩"                      (0..n)
```

**`options` fields** (all *Defaulted* per field: missing, invalid or out-of-range → the
fallback; no field is clamped individually). After fields 8 – 11 are parsed, the exclusion zone
(x, y, w, h) is trimmed to the unit square: if x + w > 1 then w ← 1 − x; if y + h > 1 then
h ← 1 − y (single-precision; x and y are never changed). Example: `0.8,0,0.5,0.3` loads as
(0.8, 0, ≈ 0.2, 0.3). The RI applies this trim on every assignment of the zone (including the
options dialog), so the trimmed values are what is saved. RI defect: as in §3.4, a literal `NaN`
passes every real range check here and is stored unchanged (it is not trimmed either).
**Recommendation:** treat `NaN` as invalid (use the fallback).

| #  | Field                          | Kind  | Range       | Fallback |
|----|--------------------------------|-------|-------------|----------|
| 1  | SpeechEnabled                  | bool  |             | True     |
| 2  | SpeechChance                   | real  | 0 – 1       | 0.01     |
| 3  | CursorAwareness                | bool  |             | True     |
| 4  | CursorAvoidanceRadius          | real  | 0 – 10000   | 100      |
| 5  | DraggingEnabled                | bool  |             | True     |
| 6  | InteractionsEnabled            | bool  |             | True     |
| 7  | DisplayInteractionErrors       | bool  |             | False    |
| 8  | ExclusionZone.X                | real  | 0 – 1       | 0        |
| 9  | ExclusionZone.Y                | real  | 0 – 1       | 0        |
| 10 | ExclusionZone.Width            | real  | 0 – 1       | 0        |
| 11 | ExclusionZone.Height           | real  | 0 – 1       | 0        |
| 12 | ScaleFactor                    | real  | 0.25 – 4    | 1        |
| 13 | MaxPonyCount                   | int   | 0 – 10000   | 300      |
| 14 | AlphaBlending                  | bool  |             | True     |
| 15 | EffectsEnabled                 | bool  |             | True     |
| 16 | WindowAvoidance                | bool  |             | False    |
| 17 | PoniesAvoidPonies              | bool  |             | False    |
| 18 | WindowContainment              | bool  |             | False    |
| 19 | TeleportEnabled                | bool  |             | False    |
| 20 | TimeFactor                     | real  | 0.1 – 10    | 1        |
| 21 | SoundEnabled                   | bool  |             | True     |
| 22 | SoundSingleChannel             | bool  |             | False    |
| 23 | SoundVolume                    | real  | 0 – 1       | 0.75     |
| 24 | AlwaysOnTop                    | bool  |             | True     |
| 25 | SuspendForFullscreen           | bool  |             | True     |
| 26 | ScreensaverSoundEnabled        | bool  |             | True     |
| 27 | ScreensaverStyle               | enum name (case-sensitive) or number | `Transparent`/`SolidColor`/`BackgroundImage` | Transparent |
| 28 | ScreensaverBackgroundColor     | int (signed ARGB) |  | 0        |
| 29 | ScreensaverBackgroundImagePath | text  |             | ""       |
| 30 | NoRandomDuplicates             | bool  |             | True     |
| 31 | ShowInTaskbar                  | bool  |             | true on Windows, else false |
| 32 | AllowedArea.X                  | int   |             | 0        |
| 33 | AllowedArea.Y                  | int   |             | 0        |
| 34 | AllowedArea.Width              | int   | ≥ 0         | 0        |
| 35 | AllowedArea.Height             | int   | ≥ 0         | 0        |
| 36 | BackgroundColor                | int (signed ARGB) |  | 0        |

The allowed area is set only if width > 0 and height > 0; otherwise "use Screens".

**`monitor`** — exactly 2 fields; the device name of a monitor to use (RI: OS display device
name such as `\\.\DISPLAY1`; a Linux implementation SHOULD use the output/connector name such
as `DP-1`, and MAY also accept an index). Unknown monitors are ignored; if none remain, the
primary monitor is used.

**`count`** — exactly 3 fields; pony directory (case-sensitive) and an int > 0. Entries for
uninstalled ponies are kept in the map (and written back). The key `Random Pony` holds the
number of random ponies. RI defect: a duplicate key aborts loading (fatal from the main menu);
**Recommendation:** last one wins.

**`tag`** — exactly 2 fields; one custom tag (order kept; empty strings and duplicates kept).

Example:

```
options,True,0.01,True,100,True,True,False,0,0,0,0,1,500,True,True,False,False,False,False,1,True,False,0.75,True,True,True,Transparent,0,,True,True,0,0,0,0,0
monitor,"\\.\DISPLAY1"
count,"Random Pony",2
count,"Twilight Sparkle",1
tag,"Favourites"
```

### 5.3.3 Writing

Overwrite `Profiles/N.ini` with: one `options` line (booleans `True`/`False`, numbers in
invariant culture, enum by name, colours as signed ARGB int32, the image path **unquoted** — RI
defect: a comma or `"` in that path is written without error but corrupts the line when read
back; **Recommendation:** quote it), then one `monitor` line per selected screen, one `count`
line per entry with n > 0, one `tag` line per custom tag.

Monitor names, count keys and tags are wrapped in `"` without escaping; the RI writer fails with
an error on such a value that contains `"`. The file has already been truncated when this
happens, so it is left holding only the lines before the offending value and its previous
contents are lost. Where the error surfaces is described in §5.3.5. The Filters dialog removes
every custom tag (line) containing `"` entirely, with a warning; values loaded from a file never
contain `"` (the **Q** splitter consumes quotes). The remaining realistic source is a pony
directory name containing `"` (impossible on Windows, possible elsewhere). RI defect: a save that
fails part-way destroys the existing profile. **Recommendation:** validate all values (reject or
escape `"`) before writing, or write a temporary file and replace the original only on success.

### 5.3.4 Loading `LoadProfile(name, setAsCurrent)`

```
if name is empty: error
if name = "default" (any case): reset all options to defaults
elif Profiles/name.ini does not exist: reset to defaults; profile name ← name
else:
    profile name ← name
    apply the options line if present           # RI: fields NOT reset first
    Screens ← listed monitors (or primary); PonyCounts ← listed counts; CustomTags ← listed tags
if setAsCurrent: write name to Profiles/current.txt (errors ignored)
```

RI quirks and recommendations:

* Loading an existing file does not reset options first: a file without an `options` line keeps
  the previous profile's options, and a profile with a zero-sized allowed area keeps the
  previous allowed area. **Recommendation:** reset to defaults before applying a file.
* `EnablePonyLogs` / `ShowPerformanceGraph` are session-only: never read from or written to a
  file. Loading an existing file keeps their current values, but both reset branches (`default`,
  or a name with no file) clear them to false — so they also start false, and selecting or
  reloading `default` (including after Delete, §5.3.5) or a name with no file turns them off.
  **Recommendation:** leave them out of the reset so that no profile load changes them.
* `setAsCurrent` depends on the program file name, not on screensaver mode: UI loads (§5.3.5)
  pass `setAsCurrent = not screensaverExecutable`, where *screensaverExecutable* means the
  executable path ends in `.scr` (case-insensitive on Windows, case-sensitive elsewhere). A
  normally named executable started with `/s` therefore still writes `current.txt`; a `.scr`
  executable never does. The start-up load of `screensaver` by a `.scr` executable and the load
  of `autostart` (§8.3) always pass false.

### 5.3.5 Profile operations (UI)

| Operation | Behavior |
|-----------|----------|
| Select (from the list) | `LoadProfile(name, setAsCurrent = not screensaverExecutable)` (§5.3.4), refresh counts and filters. The list shows `default` first, then every `Profiles/*.ini` (file-system order). |
| Save      | Name = the entered text (not trimmed). Refuse empty, `default` (any case) or invalid names with an information message. Otherwise save the current in-memory options, monitors, counts and tags under the name (silent overwrite; the current profile name is not changed); append the name to the list if no item equals it (case-sensitive, so a differently cased duplicate can appear); select it in the list. Only if this changes the selected index does it act as Select (reload from disk, update `current.txt`, refresh); when the saved profile was already selected and its text not edited, nothing is reloaded. Finally show "Profile '⟨name⟩' saved.". |
| Reload    | `LoadProfile(entered name, setAsCurrent = not screensaverExecutable)`, discarding unsaved changes. A name with no file gives defaults under that name. |
| Copy      | Prompt for a new name (trimmed); refuse blank/`default`/invalid; save the current options under it and select it. |
| Delete    | Name = the entered text (not trimmed). Refuse `default` (any case) or invalid names with an information message; there is no empty-name check. Delete `Profiles/name.ini`: if that does not fail, show "Profile deleted successfully" — also when no such file exists; if it fails (directory missing, access denied, file in use, …), warn "Error attempting to delete this profile. Perhaps it has already been deleted.". Either way rebuild the list and select `default` (which loads it). |
| Load (options dialog) | `LoadProfile(current profile name, setAsCurrent = not screensaverExecutable)` (§5.3.4), then refresh every control; unsaved changes are discarded. Only I/O errors are caught, reported non-fatally ("Failed to load profile '⟨name⟩'"). |
| Save (options dialog) | Acts on the current profile name (the dialog has no name field). Refuse with an information message, in this order: no item selected in the monitor list ("You need to select at least one monitor." — checked even in custom-area mode, where the list is disabled but keeps its selection); current name `default` (any case); invalid name. Otherwise write `Profiles/⟨name⟩.ini` from the in-memory options (§5.3.3; monitor lines come from Screens, not the list) and show "Profile '⟨name⟩' saved."; any error is reported non-fatally. Does not reload or write `current.txt`. |
| Reset (options dialog) | Clears PonyCounts only. |

RI defects and recommendations:

* Main-window Save and Copy call the writer without error handling: an I/O error (e.g. missing
  `Profiles/`, access denied) or a `"` in a quoted value (§5.3.3) is an unhandled error (§8.13;
  the program exits). **Recommendation:** report save errors non-fatally, and after Save always
  load the profile and make it current explicitly rather than relying on a selection change.
* Delete with an empty name is an unhandled error (the name validation runs outside error
  handling); a whitespace-only name passes all checks. A name with no file is reported as
  deleted. **Recommendation:** refuse blank names with an information message, and report "no
  such profile" when the file does not exist.
* After the options dialog's Load, the refresh only adds monitor-list selections for the loaded
  Screens and never clears earlier ones, so monitors no longer in Screens can stay highlighted
  (Screens itself is unchanged until the user next edits the selection).
  **Recommendation:** clear the selection before applying the loaded Screens.
