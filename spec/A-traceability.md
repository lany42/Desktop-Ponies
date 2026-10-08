# Appendix A. Traceability (for reviewers)

> Clean-room note: this appendix points to the reference implementation's source so that
> reviewers can verify the specification. Implementers working under a strict clean-room
> process should not use it.

Paths are relative to the repository root. `DP/` = `Desktop Ponies/DesktopPonies/`,
`DS/` = `Desktop Sprites/`. Line numbers refer to the commit this spec was written against
(v1.69, `3dbfd54`).

| Spec section | Source |
|--------------|--------|
| §2.2 discovery, Random Pony, legacy migration | `DP/PonyCollection.vb` 43–119 |
| §2.3 Random Pony constants | `DP/Pony.vb` 75–92 |
| §3.2 line classification (pony.ini) | `DP/Pony.vb` 316–368 (`ParsePonyConfig`) |
| §3.3 tokenisers | `DP/IniLineParser.vb` 1–40; `DS/Core/StringExtensions.cs` 53–147 |
| §3.3 enum token maps (Movement, Direction) | `DP/IniLineParser.vb` 58–98 |
| §3.4 scalar parsing | `DP/StringCollectionParser.vb` 115–378; `DS/Core/Number.cs` 20–124 |
| §3.5 issue model / fatal vs fallback | `DP/StringCollectionParser.vb` 51–66, 460–527; `DP/PonyCollection.vb` 203–232 |
| §3.6 Name, §3.8 BehaviorGroup | `DP/PonyCollection.vb` 234–263 |
| §3.7 Categories | `DP/Pony.vb` 358–362 |
| §3.9 Behavior fields | `DP/Pony.vb` 698–1092 (`TryLoad` 966–1020) |
| §3.10 Effect fields | `DP/Pony.vb` 3376–3566 (`TryLoad` 3474–3513) |
| §3.11 Speak | `DP/Pony.vb` 1134–1220 |
| §3.12 Interaction, activation tokens | `DP/Pony.vb` 476–656, 4483–4504 |
| §3.14 references (unique match) | `DS/Core/Linq.cs` 46–60 (`OnlyOrDefault`); `DP/Pony.vb` 20–70, 1067–1085 |
| §3.15 implicit behaviors | `DP/Pony.vb` 1832–1886 |
| §3.16 canonical writer | `DP/Pony.vb` 395–433, 590–599, 1022–1052, 1123–1125, 1186–1209, 3528–3543 |
| Ch. 4 house.ini | `DP/Pony.vb` 3810–3958; `DP/PonyCollection.vb` 265–356; writer `DP/HouseOptionsForm.vb` 124–207 |
| Ch. 5 options, profiles | `DP/Options.vb` (defaults 243–286, load 127–241, save 288–344, allowed area 356–397) |
| §6.2 context | `DP/Pony.vb` 1224–1395 |
| §6.3 time model | `DP/Pony.vb` 1458–1466, 2036–2067; `DS/SpriteManagement/AnimationLoopBase.cs` 1145–1260 |
| §6.4 state, IsBusy | `DP/Pony.vb` 1453–1821 |
| §6.5 SetBehavior, candidate selection | `DP/Pony.vb` 2369–2474 |
| §6.6 movement | `DP/Pony.vb` 2686–2782, 2854–2880 |
| §6.7 destination, following, visual override | `DP/Pony.vb` 2504–2598, 2652–2678 |
| §6.8 effects | `DP/Pony.vb` 2787–2844, 3570–3806 |
| §6.9 speech | `DP/Pony.vb` 2323–2356 |
| §6.10 interactions | `DP/Pony.vb` 3107–3338 |
| §6.11 special states | `DP/Pony.vb` 2106–2184 |
| §6.12 bounds, rebounds | `DP/Pony.vb` 2228–2305, 2607–2645, 2888–3099 |
| §6.13 houses | `DP/Pony.vb` 3962–4138; `DP/DesktopPonyAnimator.vb` 215–229, 319–324 |
| §6.14 overrides, manual control | `DP/Pony.vb` 1980–2031, 2196–2222; `DP/PonyAnimator.vb` 15–48, 186–244 |
| §6.15–6.16 start / step | `DP/Pony.vb` 2036–2098, 2485–2497 |
| §7.1 sprite contract | `DS/SpriteManagement/ISprite.cs`; `DP/I*Sprite.vb` |
| §7.3 image size, centers | `DS/Core/ImageSize.cs`; `DP/Pony.vb` 4142–4228, 747–748, 3403–3404 |
| §7.4 frame lookup | `DS/SpriteManagement/AnimatedImage.cs` 98–147, 254–298 |
| §7.5 GIF decoding, `.art` | `DS/SpriteManagement/GifImage.cs`; `DS/SpriteManagement/AlphaRemappingTable.cs` |
| §7.6 bubbles, scaling (reference look) | `DS/SpriteManagement/WinFormSpriteInterface.cs` 1624–1772, 1935–2183 |
| §7.8 GTK backend | `DS/SpriteManagement/GtkSpriteInterface.cs` |
| §8.2–8.5 start-up, CLI, menu, launch | `Desktop Ponies/Program.vb`; `DP/MainForm.vb` 42–285, 106–183, 535–842 |
| §8.6 host loop | `DP/PonyAnimator.vb` 147–218; `DP/DesktopPonyAnimator.vb` 19–24, 231–240 |
| §8.7 sound | `DP/PonyAnimator.vb` 246–334 |
| §8.8 input, menus | `DP/PonyAnimator.vb` 168–183, 368–384; `DP/DesktopPonyAnimator.vb` 53–317 |
| §8.9 house dialog | `DP/HouseOptionsForm.vb` |
| §8.11 screensaver | `DP/MainForm.vb` 77–80, 890–924; `DP/PonyAnimator.vb` 158–166 |
| Ch. 9 games | `DP/Game.vb`; `DP/GameSelectionForm.vb`; `Content/Games/*/game.ini` |
| §1 component map | solution layout; `Desktop Ponies/Desktop Ponies.vbproj` |
