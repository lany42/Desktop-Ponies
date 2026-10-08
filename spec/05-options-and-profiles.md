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
| ExclusionZone              | 4 × real        | (0,0,0,0)       | each 0 – 1         | normalised (x, y, w, h) sub-rectangle of the allowed area that ponies avoid (§6.2, §6.12). Zero size = none. |
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
| ScreensaverSoundEnabled    | bool            | true            |                    | §8.7 (screensaver mode) |
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

## 5.2 Allowed area

The allowed area `R` (Chapter 6 §6.12) is computed every frame:

* If **AllowedArea** is set: `AllowedArea ∩ (union of all monitors' full bounds)`.
* Otherwise: the union (bounding rectangle) of the **work areas** (screen minus panels/taskbars)
  of the monitors in **Screens**; a monitor with an empty work area contributes its full bounds.

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

**`options` fields** (all *Defaulted*: missing, invalid or out-of-range → the fallback; never
clamped):

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
defect: a comma in that path corrupts the line; **Recommendation:** quote it), then one `monitor`
line per selected screen, one `count` line per entry with n > 0, one `tag` line per custom tag.
Values containing `"` cannot be written (the UI strips quotes from custom tags).

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
* `EnablePonyLogs` / `ShowPerformanceGraph` survive profile switches (they are session-only).
* `setAsCurrent` is false when running as the screensaver.

### 5.3.5 Profile operations (UI)

| Operation | Behavior |
|-----------|----------|
| Select (from the list) | `LoadProfile(name, setAsCurrent = not screensaver)`, refresh counts and filters. The list shows `default` first, then every `Profiles/*.ini` (file-system order). |
| Save      | Save the current in-memory options and counts under the entered name (silent overwrite); refuses empty, `default`, or invalid names; then select it (which reloads it from disk and updates `current.txt`). |
| Reload    | `LoadProfile(entered name)`, discarding unsaved changes. A name with no file gives defaults under that name. |
| Copy      | Prompt for a new name (trimmed); refuse blank/`default`/invalid; save the current options under it and select it. |
| Delete    | Refuse `default`/invalid; delete the file (errors reported); select `default`. |
| Reset (options dialog) | Clears PonyCounts only. |
