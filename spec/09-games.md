# 9. Mini-Games (`game.ini` and Game Rules)

The RI ships two team mini-games, **Hoofball** (soccer) and **Ping Pong Pony**, in which ponies
are driven by simple AI through the external control interface of Chapter 6 §6.14. This feature
is self-contained and optional; an implementation can omit it without affecting desktop ponies.

---

## 9.1 Discovery

* Every immediate subdirectory of `Games/` (not recursive; order: file-system order) is loaded as
  a game from its `game.ini`; nothing checks first that the file exists. Games are (re)loaded
  every time the game dialog opens. A subdirectory that fails to load — **including one without a
  `game.ini`** — is left out of the list with a non-fatal warning "Error loading game:
  ⟨directory⟩" (logged to the console; also a warning dialog except on macOS); the others still
  load. RI defect: a stray non-game folder under `Games/` raises a warning every time the dialog
  opens. **Recommendation:** skip subdirectories without a `game.ini` silently; warn only when an
  existing `game.ini` fails to load.
* The game dialog's icon for a game is the ball's idle image.

## 9.2 `game.ini` format

Unlike `pony.ini`, this file uses a strict ad-hoc parser: most problems make the whole game fail
to load.

### 9.2.1 Lines

* Read as UTF-8 (BOM skipped), LF/CR/CRLF.
* A line that is exactly empty, or whose first character is `'`, is skipped.
* The line is split with the **Q** splitter; if it yields fewer than 2 fields (no unquoted
  comma), it is skipped silently.
* Field 0, lower-cased (not trimmed), selects the line type: `game`, `description`, `position`,
  `ball`, `goal`, `scoreboard`. **Any other identifier fails the game.**
* `game`, `description`: only the raw line is kept, so the last occurrence wins; it is parsed
  after the whole file has been read (earlier occurrences are never checked). `position`,
  `ball`, `goal`: accumulate in file order and are parsed after reading.
* `scoreboard`: parsed and validated **as soon as it is read**, from this line's Q fields
  (§9.2.6). Any error fails the game at once — no later line is read, and the required-lines
  check below has not run yet — even if a later `scoreboard` line is valid. When every
  `scoreboard` line is valid, the last one wins. RI defect: an overridden `scoreboard` line can
  still fail the game. **Recommendation:** keep the raw line and parse only the last occurrence,
  as for `game`.
* Required: at least one `game`, one `description`, one `position` and one `ball` line; a
  `scoreboard` line is required in practice (the RI fails at launch without it).

Two splitters are used per line type (re-splitting the *raw line*):

* **B** (brace-qualified): `{…}` protects commas, quotes are ordinary characters and remain in the
  field (they are removed explicitly where noted).
* **Q**: as in §3.3.

Sub-lists inside braces are split by a **plain** comma split (no qualifiers) unless stated.

Trimming: "trim" below removes only spaces (U+0020) and ideographic spaces; numeric fields allow
surrounding whitespace; fields not marked "trim" keep their spaces (e.g. team names in the
shipped files keep a leading space: `" Team Luna"`). Exceptions: scoring styles (§9.2.2) and
action entries (§9.2.4) are stripped of **all** leading and trailing Unicode white space (tab,
NBSP, the U+2000–U+200A spaces, line/paragraph separators, etc.).

### 9.2.2 `Game` (B split)

```
Game,⟨Name⟩,{⟨Team1⟩,⟨Team2⟩},⟨MinBalls⟩,⟨MaxBalls⟩,⟨MaxScore⟩,{⟨ScoringStyle⟩,…}
```

| # | Field | Rules |
|---|-------|-------|
| 1 | Name | `"` removed; **not trimmed**. The exact name `Ping Pong Pony` enables Ping-Pong-specific rules (§9.6). |
| 2 | Teams | content split with **Q**; team *k* (1-based) is the *k*-th entry, untrimmed. Exactly 2 teams are supported. |
| 3 | MinBalls | int ≥ 1 |
| 4 | MaxBalls | int ≥ MinBalls (otherwise unused) |
| 5 | MaxScore | int ≥ 1; first team to reach it wins |
| 6 | Scoring styles | content split on every comma (plain split); each entry lower-cased (invariant) and stripped of all Unicode white space (§9.2.1 exceptions — e.g. `{ball_at_goal⟨TAB⟩}` is accepted); must then equal one of `ball_at_goal`, `ball_at_sides`, `ball_hits_other_team`, `ball_destroyed` (the last two are accepted but have no effect); any other value, including an empty entry (empty list, stray comma), fails ("Invalid scoring style: ⟨raw entry⟩") |

### 9.2.3 `Description` (Q split)

`Description,⟨Text⟩` — field 1, untrimmed. Shown as "⟨Name⟩: ⟨Text⟩".

### 9.2.4 `Position` (B split)

```
Position,⟨Name⟩,⟨Team⟩,{⟨sx⟩,⟨sy⟩},{⟨bx⟩,⟨by⟩,⟨bw⟩,⟨bh⟩}|{any},⟨HaveBall⟩,⟨HostileBall⟩,⟨FriendlyBall⟩,⟨NeutralBall⟩,⟨DistantBall⟩,⟨NoBall⟩,Required|Optional
```

| #  | Field | Rules |
|----|-------|-------|
| 1  | Name | `"` removed, trimmed |
| 2  | Team | int, 1 or 2 |
| 3  | Start | two ints: **percent of this position's box** (not of the screen) — the anchor point where the player lines up |
| 4  | Box | four reals in percent of the game area, or `any` (case-insensitive, trimmed) = the whole game area. Used only as the reference for Start and as the limit for push-apart (§9.5.4); it does **not** confine movement. Anything but `any` is only split on commas at load, **not validated**; it is parsed at launch (§9.3), and only for filled positions: each of the first four parts as an invariant real (surrounding whitespace, sign, decimal point, exponent), converted per §9.2.8; further parts are ignored. Fewer than four parts (e.g. `{}`), a non-number, or a value that is NaN/∞ or converts outside the 32-bit integer range makes the launch fail (§9.3). RI defect: a bad box is reported at PLAY, not at load, and never for an unfilled optional position. **Recommendation:** validate at load (exactly four finite reals, or `any`) and fail the game's load. |
| 5–10 | Action lists | comma-separated action entries (§9.4); braces needed only for more than one entry; duplicates act as weights. Each entry, stripped of all Unicode white space (§9.2.1), is either a decimal integer (invariant, optional sign; any 32-bit value loads, even an undefined action — see §9.4; a value outside the 32-bit range fails) or an action **name** from §9.4, matched exactly and case-sensitively (`ChaseBall` ≡ `2`; `chaseball` or `"ChaseBall"` fails). Anything else (`4.0`, `4x`) fails, and so does an empty entry (empty field, stray comma). Numbers and names may be mixed (`{4,ThrowBallToTeammate}`); writers SHOULD emit numbers. List 10 (*no ball*) is parsed but never used. |
| 11 | Required/Optional | case-insensitive, trimmed; anything else fails |

Fewer than 12 fields fails.

### 9.2.5 `Goal` (B split)

`Goal,⟨Team⟩,⟨image⟩,{⟨x⟩,⟨y⟩}` — team int (0 = no team), image file in the game directory
(`"` removed, trimmed; must exist), top-left in percent of the game area. Goals keep file order.
A goal with team *n* ≠ 0 becomes team *n*'s **scoring goal** (§9.5.5); a later goal for the same
team replaces it (last wins). A non-zero team outside 1..number of teams fails the load. The AI
does **not** use the scoring goal; it looks goals up by file order (§9.4).

### 9.2.6 `ScoreBoard` (Q split)

`ScoreBoard,⟨x⟩,⟨y⟩,⟨image⟩` — top-left in percent of the game area. Checked immediately
(§9.2.1), in this order: at least 4 fields (extra fields ignored); the image (`"` removed,
trimmed, relative to the game directory) must exist; x, then y, must be ints.

### 9.2.7 `Ball` (Q split)

```
Ball,⟨Type⟩,⟨idle⟩,⟨slowRight⟩,⟨slowLeft⟩,⟨fastRight⟩,⟨fastLeft⟩,⟨x⟩,⟨y⟩
```

Type `soccer` or `pingpong` (case-insensitive, trimmed). Five images (trimmed, `"` removed; not
checked at load). Start point (the ball's anchor) in percent of the game area.

### 9.2.8 Coordinates

The **game area** `WA` is the work area of the monitor chosen in the game dialog. Percent
coordinates convert as `round(p/100 · WA.size + WA.origin)` (round half to even).
Box: `(round(bx%·W + WA.x), round(by%·H + WA.y), round(bw%·W), round(bh%·H))`.
Start: `(round(sx%·box.w + box.x), round(sy%·box.h + box.y))`.

### 9.2.9 Example (`Games/Hoofball/game.ini`, abridged)

```
Game,"Hoofball", {"Team Luna", "Team Celestia"}, 1, 1, 10, {Ball_at_Goal}
Description, "Try to kick the ball into your opponent's goal! ..."
Position, "Goalie", 1, {8,50}, {0,20, 20, 40}, {5},{10},{13,8},{13,8},{8}, {1}, Required
Position, "Player 1", 1, {35,50}, {any} , {4,5},{10,10,10,8},{10},{10},{8,13},{1}, Required
Goal, 1, "goal_left.gif", {8,40}
Goal, 2, "goal_right.gif", {90,40}
ScoreBoard, 45, 15, "scoreboard.gif"
Ball, Soccer, "soccer_ball_still.gif", "soccer_ball_slow_r.gif","soccer_ball_slow_l.gif", "soccer_ball_fast_r.gif","soccer_ball_fast_l.gif", 49,45,
```

The "Player 1" have-ball list `{4,5}` means: when near the ball, shoot at goal or pass, 50/50;
its hostile list `{10,10,10,8}` means: when the other team last touched the ball, lead the ball
75 % of the time, otherwise fall back to the own goal.

## 9.3 Setup (game dialog)

* Choose a game, then fill positions: for each team, a slot per position (file order); adding a
  pony (or "Random Pony" = uniform random pony) fills the first empty slot of that team.
* PLAY requires every **Required** position filled and a monitor chosen. Unfilled optional
  positions are dropped.
* Launch: create each goal and the scoreboard once, as a plain non-following, non-expiring effect
  sprite with its top-left at its start point (plus the four scoreboard labels, §9.5.6); then,
  per team in position order, initialise each filled position (box and start point, §9.2.4) and
  add its player pony; place nothing else. The Context for a game is the desktop Context
  with: allowed region = the game area, no exclusion zone, window avoidance/containment off,
  teleport on, **cursor awareness off**.
* Goals and the scoreboard never move by themselves and are never reset, but like any effect the
  user can drag them (§8.8.2, §6.8.4; a player or ball under the cursor is picked first). The
  game reads goal positions live: dragging a goal moves its scoring rectangle (§9.5.5) and the
  AI's aim and approach points (§9.4); dragging the scoreboard carries its labels. RI defect.
  **Recommendation:** make game goals and the scoreboard non-draggable, or record dragging as a
  deliberate choice.
* A launch failure (e.g. an invalid Box, §9.2.4) aborts the setup part-way (sprites queued so
  far are never animated; balls are not set up) and is reported as the non-fatal warning "Error
  loading games."; the game does not start. RI defect: the menu window, hidden while the game
  dialog was open, is not shown again. **Recommendation:** validate everything at load; on a
  launch failure, return to the menu.

## 9.4 Player actions

| # | Action | Effect when chosen |
|---|--------|--------------------|
| 1 | ReturnToStart | go to the start point (§9.5.2) |
| 2 | ChaseBall | follow the ball (`FollowTargetOverride = ball`, destination override cleared) |
| 4 | ThrowBallToGoal | kick toward the opposing goal (speed 10, "\*Kick\*!") |
| 5 | ThrowBallToTeammate | AI player: set the last-kick time (so a miss also counts), then pass to an open teammate (speed 10, "\*Pass\*!"), else shoot at the opposing goal (speed 10, "\*Kick\*!"; this fallback is not remembered, §9.5.2). Manually controlled player (key held or not): **no effect** — no kick, cool-down not reset, action not remembered |
| 7 | ThrowBallReflect | Pong-style bounce (speed 7) |
| 8 | ApproachOwnGoal | go to the own goal's approach point |
| 9 | ApproachTargetGoal | go to the opposing goal's approach point |
| 10 | LeadBall | same as ChaseBall |
| 13 | Idle | stop following; random behavior (no start speech); does *not* clear an existing destination override |

Values 0 (NotSet), 3 (AvoidBall), 6 (ThrowBallAtTarget), 12 (ApproachTarget) and undefined
numbers load (the four named ones by name too, §9.2.4) but are fatal when picked in the RI; an
implementation SHOULD reject them at load.

**AI goals.** Goals are searched in file order (§9.2.5), independently of the teams' scoring
goals: the **opposing goal** is the first goal whose team differs from the player's position
team (a team-0 goal qualifies); the **own goal** is the first goal with the player's team. If
none exists when an action needs it (4, 5, the open-teammate test, 8, 9), the RI throws
(fatal). Target points: kicks and the open-teammate test use the center of the goal effect's
**scaled** Region (§6.8.2), `topLeft + (round(w·k), round(h·k))/2` as reals (`w × h` = image
size, `k` = ScaleFactor); the **approach point** of actions 8/9 is the **unscaled** image center
`round(topLeft + (w, h)/2)` (rounding half to even throughout). The two coincide only when
`k = 1`. RI defect: the AI can aim at or approach a goal that cannot score, or another than the
one its team defends (e.g. a team-0 goal listed first, or several goals per team).
**Recommendation:** use team *n*'s single scoring goal for scoring and AI alike, and reject
games where a team has no goal or more than one.

## 9.5 Game loop

`Game.Update` runs once per **frame** (not per simulation step), before the ponies are updated
(§8.6 step 3).

### 9.5.1 States

| State | Per frame |
|-------|-----------|
| Setup | Clear every ball's last kicker; disable manual control; every player: go to its start point; current action = ReturnToStart. → WaitForPositions |
| WaitForPositions | Every player: go to its start point. When every player is at its destination → AddBalls |
| AddBalls | Clear players' actions; add every ball sprite. → ReadyBalls |
| ReadyBalls | Enable manual control. First ball only: clear its speed override, set its idle behavior, place it at its start point, update its speed class; a ping-pong ball is served: kick(speed 5, random angle in [0, 2π)). → InProgress |
| InProgress | Update every active ball's speed (§9.5.3); for each team, each player in order: decide and perform an action (§9.5.2), then push apart from overlapping players (§9.5.4). Then check scoring (§9.5.5). |

### 9.5.2 Deciding

Let `d` = distance from the player's anchor to the nearest active ball's anchor.

1. If `d > ½·diagonal(WA)` → *distant* list.
2. Else if `d ≤ player.Region.width/2 + 50` → *have-ball* list.
3. Else by the team of the ball's last kicker: none → *neutral*; own team → *friendly*; other →
   *hostile*.

Then:

* If the player is already executing an action from the **same list**: keep it — except for the
  have-ball list, where after the kick cool-down (§ below) a new action is drawn. During the
  cool-down, a manually controlled player whose cool-down is between 2 and 3 s says "Can't kick
  again so soon!".
* Otherwise draw an action uniformly from the list entries and perform it. The action and list
  are remembered only if the action completed (blocked kicks, action 5 for a manually controlled
  player, action 5's fallback shot, and an Idle on a player already idle are re-drawn next
  frame).

Kick cool-down: kicking actions (4, 5, 7) are blocked while fewer than **3** whole seconds of
wall-clock time have passed since the player's last kick attempt (the RI's "2 seconds" compares
truncated seconds ≤ 2). Manually controlled players must also hold their action key (player 1:
right Ctrl, player 2: left Ctrl) for actions 4 and 7.

**Go to a point** = `SpeedOverride ← game speed` (267 px/s for Ping Pong Pony, else 167 px/s),
`DestinationOverride ← point`, `FollowTargetOverride ← none`.

**Kick(ball, speed, target)**: with probability 0.05 say "Missed!" and do nothing; otherwise say
the action's line ("*Kick*!" / "*Pass*!"), aim from the player's anchor at the target (the
opposing goal's scaled center, §9.4, or the teammate's anchor), add ±U·π/8 (sign 50/50), and
apply `ball.Kick(speed, angle)`: last kicker ← this player, `SpeedOverride ← speed × 1000/30` px/s,
`MovementOverride ← (cos θ, sin θ)` (screen coordinates, y down).

**Open teammate**: a teammate (not self) with no opposing player within 200 px and at least as
close to the opposing goal's scaled center (§9.4) as the passer; uniform among qualifiers.

**Bounce (Ping Pong)** (no "Missed!" roll):

```
if game is Ping Pong Pony and this player is the ball's last kicker: return
say "*Ping*!"
θ ← π if ball.movement.x ≥ 0 else 0
(px, py) ← round(player anchor); (bx, by) ← round(ball anchor)     # half to even
dy ← py − by
k ← (π/4) · |dy| / (player.Region.height / 1.5)
s ← +1 if dy > 0 else −1
if not (px < WA.width / 2): s ← −s          # "left" test: absolute x, WA.x NOT added
θ ← θ + s·k
θ ← θ ± U·π/8                               # sign 50/50
# clamps: four sequential tests on the raw θ, which is never reduced modulo 2π
if π/2 ≤ θ < 2π/3:  θ ← 2π/3
if π/3 < θ ≤ π/2:   θ ← π/3
if 4π/3 < θ ≤ 3π/2: θ ← 4π/3
if 3π/2 ≤ θ < 5π/3: θ ← 5π/3
ball.Kick(7, θ)
```

RI defect: the left/right test is the work area's true midpoint only when `WA.x = 0`; on a
monitor to the right of the primary every player counts as "right", on one to the left every
player counts as "left", and a left-docked taskbar shifts the line. **Recommendation:** compare
`px` with `WA.x + WA.width/2`.

RI defect: since θ is not normalised, the clamps act only on raw values in (π/3, 5π/3). From
base 0, steep upward kicks (θ in about (−π/2, −π/3); reachable because, from the have-ball list,
`|dy|` can be up to `player.Region.width/2 + 50`, step 2 above) pass unclamped while their
downward mirror images are clamped to π/3; very large `k` can also escape from base π.
**Recommendation:** normalise θ into [0, 2π) before clamping.

Game lines are spoken as ad-hoc speeches (`DisplayName: "⟨line⟩"`, no sound) and obey the
speech option.

### 9.5.3 Ball physics

The ball is a pony-like sprite with three synthetic behaviors: idle (speed 0, no movement),
slow (speed 3, All) and fast (speed 5, All), each lasting 600 s, using the idle / slow / fast
images. It moves by the pony engine: movement overrides, wall rebounds off the game area
(§6.12.4), and — if "ponies avoid ponies" is on — rebounds off players.

Each frame: `v ← SpeedOverride or behavior speed`; a **soccer** ball loses 2 % (`v ← 0.98·v`,
stored as SpeedOverride) — friction is per frame, not per step. Speed class: `v < 1` → idle,
`v < 100` → slow, else fast. On a class change the ball's behavior is switched and its current
movement re-applied as a movement override.

RI defect: switching behavior regenerates a random free-movement direction before the current
vector is captured, so the ball usually veers randomly when it speeds up after a kick or slows
below 100 px/s. **Recommendation:** preserve the direction.

### 9.5.4 Push-apart

For every other player whose Region intersects this player's Region: find the smallest of the
four overlaps (left, right, top, bottom; ties in that order) and let `L'` = this player's
location moved by `min(2 px, overlap)` in that direction (not rounded). Move **this** player to
`L'` only if `Contained(player, L', box)`, where box = its position Box in pixels, or the whole
game area for `any`.

**Containment test** `Contained(S, c, R)` (also used by §9.5.5): take a box of the sprite's
current Region size (scaled and truncated, §6.2.1) centred geometrically on `c` (top-left
`c − size/2`, reals); true if `R` contains it, edges inclusive. RI defect: this box is not the
sprite's actual Region, which is placed by the image anchor (§6.2.1); they differ by up to about
1 px × ScaleFactor with natural centers and arbitrarily with a custom image center.
**Recommendation:** test the sprite's actual Region as it would be at `c`.

### 9.5.5 Scoring and end

* `ball_at_sides`: if the ball's region left edge is within 2 % of the game area's left edge,
  team 2 scores; if its right edge is within 2 % of the right edge, team 1 scores.
* `ball_at_goal`: goals are checked in file order. A goal *G* counts if
  `Contained(ball, round(ball anchor), G's rectangle)` (§9.5.4), where the rectangle is the goal
  effect's **current** top-left (§9.3) with its **unscaled** image size, *G* is some team *T*'s
  scoring goal (§9.2.5), and the ball has a last kicker who is not on *T*; then the first team
  other than *T* scores. (No own goals; an untouched ball never scores; team-0 goals and a team's
  earlier, overridden goals never score.)
* On a score: update the scoreboard, remove the balls. If a team reached MaxScore: pause, show
  "⟨Team⟩ won!", and return to the menu. Otherwise → Setup (players walk back, a fresh ball is
  placed and served).

### 9.5.6 Scoreboard

The scoreboard image is a sprite; four text labels are drawn as speech-bubble-style text at
fixed offsets from its top-left (scaled by ScaleFactor): team 1 name (66,107), team 2 name
(66,150), team 1 score (130,113), team 2 score (135,156). They are always drawn on top.

### 9.5.7 Manual control in games

As §6.14.3 with base speed = the game speed (167 or 267 px/s, ×2 with Shift), available once
balls are in play. Manual control overrides the AI's destination each frame; for a manual
player action 5 does nothing at all (§9.4), and actions 4 and 7 act only with the action key
held.

## 9.6 Ping-Pong-specific rules (name `Ping Pong Pony`)

Player and manual speed 267 px/s instead of 167; a player cannot bounce a ball it touched last.
