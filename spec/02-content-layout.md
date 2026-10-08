# 2. Content Layout and Loading

## 2.1 Directory tree

All content is located relative to a **content root** (the RI uses the executable's directory;
see §8.1 for a Linux recommendation). The shipped tree (`Content/` in the source repository) is:

```
⟨root⟩/
├── Ponies/
│   ├── ⟨Pony Directory⟩/          one per pony; the directory name is the pony's identity
│   │   ├── pony.ini                definition (Chapter 3)
│   │   ├── *.gif                   images referenced by pony.ini
│   │   ├── *.art                   optional alpha-remap sidecars (Chapter 7 §7.5.1)
│   │   └── *.mp3                   optional sounds referenced by Speak lines
│   ├── Random Pony/                special placeholder (§2.3)
│   └── interactions.ini            optional legacy file (Chapter 3 §3.17)
├── Houses/
│   └── ⟨House Directory⟩/
│       ├── house.ini               definition (Chapter 4)
│       └── *.gif
├── Games/
│   └── ⟨Game Directory⟩/
│       ├── game.ini                definition (Chapter 9)
│       └── *.gif
├── Profiles/
│   ├── current.txt                 last used profile name (Chapter 5)
│   └── ⟨name⟩.ini                  saved profiles (none shipped)
├── Twilight.ico                    application icon
├── readme.txt, credits.txt         documentation and artwork credits (licence: CC BY-NC-SA 3.0)
└── RunOnMac.command                macOS launcher script (irrelevant on Linux)
```

Shipped corpus (v1.69): 311 pony directories (310 selectable + Random Pony), 8 houses,
2 games, ≈ 2,700 GIF images, 340 MP3 files, ≈ 200 `.art` files; no PNG and no OGG files.

Pony directory names contain spaces and punctuation (`' ( ) # . -`) and in one case non-ASCII
letters (`Jesús Pezuña`). Implementations MUST treat names as opaque Unicode strings.

## 2.2 Discovery and load order

1. Enumerate the immediate subdirectories of `Ponies/` and `Houses/` (a missing `Houses/`
   directory crashes the RI; **Recommendation:** treat a missing directory as empty).
2. Migrate the legacy `Ponies/interactions.ini` if present (Chapter 3 §3.17). This happens
   before any pony is loaded so that migrated interactions are seen.
3. Load every pony directory (in any order / in parallel):
   * a directory without `pony.ini` yields a pony with no behaviors;
   * parse `pony.ini` in runtime ("remove invalid items") mode (Chapter 3 §3.5);
   * **discard** the pony if it has no valid behavior, or if loading throws for any reason;
   * read the pixel size of every referenced behavior/effect image (header only).
4. Load every house directory (Chapter 4); discard on any error or if it has no visitors.
5. Sort ponies by directory name (case-insensitive ordinal). Remove the entry whose directory is
   exactly `Random Pony` from the list and keep it aside (§2.3).
6. Sort houses by name.

Games are loaded only when the games dialog opens (Chapter 9).

Nothing in the content tree is written during normal operation, except: the legacy interaction
migration (§2.2 step 2), profile files, `current.txt`, `error.txt`, and editor / house-dialog
saves.

## 2.3 Random Pony

`Ponies/Random Pony/` is a regular pony definition whose only purpose is to supply a picture for
"random pony" entries (menu entry, "Add Pony ▸ Random Pony"). It is never instantiated:
selecting it means "pick a real pony uniformly at random" (Chapter 8 §8.5). It is excluded from
the selectable pony list, from house deployment, and from editors' lists.

## 2.4 Identity and name matching summary

| Name                         | Matching |
|------------------------------|----------|
| Pony directory (identity)    | case-sensitive, exact (follow targets, interaction targets, house visitors, profile counts) |
| Line identifiers in `pony.ini` / `house.ini` | case-insensitive |
| Line identifiers in profiles | case-sensitive |
| Behavior / speech / effect / interaction names | case-insensitive |
| Tags                         | case-insensitive |
| File names (images, sounds)  | host file-system rules (case-sensitive on Linux); see §3.4.1 recommendation |
| Profile names                | case-insensitive for `default`/`screensaver`/`autostart`; file-system rules otherwise |
| Visitor keyword `all`        | case-insensitive |

The shipped corpus contains no case mismatches between references and files, so a strictly
case-sensitive Linux implementation works with it; third-party ponies authored on Windows may
not.
