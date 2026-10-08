# 7. Images, Animation and Presentation Contract

This chapter defines what the simulation hands to a renderer and how a renderer must turn it
into pixels **in terms of observable behavior** (which frame, where, how big, which bubble).
How pixels reach the screen (layered windows, X11 shapes, Wayland layer-shell, a full-screen
transparent overlay, …) is platform-specific and out of scope; see §7.8 for Linux notes.

---

## 7.1 Sprite contract

Every on-screen object (pony, effect, house) exposes:

| Member                 | Type             | Meaning |
|------------------------|------------------|---------|
| `ImagePaths`           | (left, right)    | file paths of the two images; the one drawn is `right` if `FacingRight` else `left` |
| `FacingRight`          | bool             | |
| `Region`               | int rectangle    | screen-space destination rectangle (x, y, w, h); also used for hit-testing, z-order and bubble placement |
| `ImageTimeIndex`       | duration ≥ 0     | time into the current animation (§7.4) |
| `PreventAnimationLoop` | bool             | play the animation once and hold the last frame |
| `SpeechText`           | string or none   | ponies only: bubble text (§7.6) |
| `SoundPath`            | path or none     | ponies only: a sound to *start* this frame (Chapter 8 §8.7) |
| `Start(T)`, `Update(T)`| —                | driven by the host loop (Chapter 6 §6.3) |

Facing is expressed **only** by choosing between two image files; the renderer never mirrors
images itself. (Both paths may name the same file.)

## 7.2 Supported image formats

* **GIF** (animated or not) — the format used by the entire shipped corpus (≈ 2,700 files, all
  with a lower-case `.gif` extension).
* **PNG**, and in principle JPEG and BMP, as single-frame static images.
* The RI chooses the GIF decoder by the extension `.gif` (case-sensitive) and treats everything
  else as a static image. **Recommendation:** sniff the file signature instead.

## 7.3 Image size and natural center

* **Size** is read from the file header: GIF logical-screen width/height; PNG `IHDR`
  width/height; BMP header; JPEG SOF0. (RI defect: JPEG width/height are swapped.) If the size
  cannot be read, it is (0, 0) and the sprite is effectively invisible but still simulated.
* **Natural center** of an image of size (W, H) (used when no custom center is given):
  * `cx = (W − 1) / 2` rounded **up** for a *right* image and **down** for a *left* image;
  * `cy = (H − 1) / 2` rounded to nearest, ties to even.

  (Example: W = 80 → right cx = 40, left cx = 39. The asymmetry keeps mirrored image pairs
  aligned on the same screen column when the pony turns around.)
* Effect images use the same rounding rules but only need their size (effects are positioned
  by top-left, Chapter 6 §6.8.2). House images likewise.

## 7.4 Animation timing

### 7.4.1 Decoded animation model

From a GIF, build:

* `frames[]` — fully composited canvas-sized frames (see §7.5);
* `durations[]` — per frame, `delay × 10 ms`, where `delay` is the field of the frame's Graphic
  Control Extension (GCE, §7.5; GIF delays are in centiseconds), **no minimum clamp** (a delay
  of 1 means 10 ms; browsers would use ~100 ms). A frame with **no** GCE has duration 0;
* `loopCount` — from the NETSCAPE2.0 application extension (0 = forever); **1 if the extension
  is absent** (play once, then hold);
* `totalDuration = Σ durations`.

**Zero-duration frames are dropped** (they are intermediate compositing steps), whether the 0
comes from an explicit GCE delay of 0 or from a missing GCE — so in an otherwise timed GIF a
frame without a GCE silently disappears, even the first or last one. A dropped frame still
contributes to compositing: its pixels stay on the canvas and show in later frames. Exception:
if every frame has duration 0 (`totalDuration = 0`, e.g. a GIF with no GCE at all), keep only
the **last** frame (a single-image GIF without a GCE is thus one frame with
`totalDuration = 0`). A static image is one frame, `loopCount = 0`, `totalDuration = 0`.

### 7.4.2 Time → frame

Given `ImageTimeIndex` τ (truncated to whole milliseconds) and PreventAnimationLoop `p`:

```
if frameCount ≤ 1: frame 0
elif p and τ ≥ totalDuration: last frame
else:
    loops ← floor(τ / totalDuration)
    if loopCount ≠ 0 and loops ≥ loopCount: last frame               # finished: hold last
    else:
        d ← τ − loops × totalDuration
        if all kept frames share one duration c and frame 0 was kept:
            frame ← floor(d / c)                                     # frame i on [i·c, (i+1)·c)
        else:
            i ← 0; while d > durations[i]: d ← d − durations[i]; i ← i + 1
            frame i                         # frame 0 on [0, S₀]; frame i ≥ 1 on (Sᵢ₋₁, Sᵢ]
```

Here `durations[]` are those of the kept frames (in ms),
`Sᵢ = durations[0] + … + durations[i]`, and within one play `0 ≤ d < totalDuration`.

Fidelity notes (implement to match the RI exactly, or knowingly deviate):

* `loopCount = N > 0` means **N total plays**, then the last frame is held.
* The two branches have different boundary conventions: with variable frame durations a frame
  boundary at exactly τ still shows the *earlier* frame. Because ponies sample τ on a 40 ms grid
  and GIF delays are multiples of 10 ms, this is visible: frame durations `[40, 80, 40]` ms (GIF
  delay fields 4, 8, 4; `totalDuration` 160 ms) sampled every 40 ms show frames 0, 0, 1, 1 at
  τ = 0, 40, 80, 120. What follows depends on the loop settings:
  * `loopCount = 0` and `p` false: τ = 160 wraps to frame 0 and the cycle 0, 0, 1, 1 repeats —
    the last frame never appears;
  * `loopCount = 1` (no NETSCAPE2.0 extension), or `p` true: from τ = 160 on the last frame
    (frame 2) is shown and held: 0, 0, 1, 1, 2, 2, …;
  * `loopCount = N > 1`: the 0, 0, 1, 1 cycle repeats N times, then frame 2 is held from
    τ = N·160.
* Frames shorter than 40 ms of simulated time can be skipped entirely (ponies only advance in
  40 ms steps).
* `ImageTimeIndex` restarts at 0 whenever a pony changes behavior, but **not** when its facing
  or its borrowed follow images change (Chapter 6 §6.7.3).

## 7.5 GIF compositing rules

An implementation can use any GIF decoder that produces fully composited RGBA frames, provided
these rules hold:

* Each output frame is a full logical-screen canvas. The canvas starts **fully transparent**.
* A frame's transparent index, disposal method and delay come from the Graphic Control
  Extension (GCE) that precedes its Image Descriptor, with only other extensions in between (if
  several GCEs precede it, the first applies and the rest are skipped). A GCE whose next
  rendering block is a Plain Text extension is used up and produces no frame (Plain Text is
  never rendered). A frame without a GCE has no transparent index, delay 0 (§7.4.1) and is not
  disposed.
* Pixels equal to the frame's transparent index are not written (the canvas shows through).
* Disposal (applied after a frame is emitted): 0 and 1 (and no GCE) keep; 2 clears the frame's
  rectangle to **transparent** (not the background color); 3 restores the rectangle to its
  state before the frame was drawn.
* The background color index and pixel aspect ratio are ignored.
* Interlaced frames use the standard 4-pass order.
* Transparency is binary (GIF has no partial alpha); partial alpha comes only from `.art` maps
  (§7.5.1).
* RI strictness (a lenient decoder MAY accept these). Each of the following makes the image fail
  to load (consequences: §7.7):
  * the signature is not `GIF`, or the version is not two digits followed by a letter;
  * a top-level block introducer other than 0x2C, 0x21 or 0x3B (extensions with unknown labels
    are skipped, not rejected);
  * a GCE followed by anything other than an Image Descriptor, a Plain Text extension, or other
    extensions leading to one of those — e.g. a GCE followed by the trailer, directly or after
    other extensions;
  * a GCE block size other than 4, an Application Extension block size other than 11, or, for
    `NETSCAPE2.0` only, a first sub-block size other than 3 (an extra GCE skipped as above is not
    checked);
  * a frame rectangle extending beyond the logical screen, or a disposal method of 4–7;
  * a missing trailer or any other truncation of the data;
  * a drawn pixel (any pixel not equal to the frame's transparent index) whose index is ≥ the
    number of entries in the active color table (local if present, else global; with no color
    table at all every drawn pixel fails). A lenient decoder SHOULD draw such pixels transparent;
  * image data that continues more than about 4,096 pixels past the end of its frame;
  * a drawn pixel with index 255 when the active color table has 256 entries and no two entries
    have the same RGB, even if the GIF uses no transparency. (The RI reserves one index per
    color table for internal use: the highest entry that duplicates an earlier one, else a new
    slot appended to a table smaller than 256, else index 255, which then cannot be drawn.)
    RI defect: valid 256-colour GIFs are rejected. **Recommendation:** support all 256 colours.

### 7.5.1 Alpha remapping files (`.art`)

An optional sidecar `⟨image name without extension⟩.art` next to an image (≈ 200 in the corpus)
gives semi-transparent colors:

* binary, a sequence of 7-byte records `R G B  A R' G' B'`: every **opaque** pixel whose color is
  exactly (R,G,B) is drawn with color (R',G',B') and alpha A;
* the file is found by replacing the image's extension with `.art`. An empty file is valid and
  maps nothing. The file is **invalid** if its length is not a multiple of 7 or if a source
  (R,G,B) appears in more than one record (even in identical records). RI defect: an invalid
  file is never ignored — it makes the whole image fail to load, with the consequences in §7.7
  (in practice a cancelled launch or a crash). **Recommendation:** treat an invalid `.art` file
  as absent: ignore the whole file, log a warning and load the image without remapping;
* applied once, after decoding; only pixels that are fully opaque in the decoded image are
  remapped — transparent pixels (including a GIF's transparent-index pixels) and
  semi-transparent pixels of static images never are. RI defect: the GTK backend (§7.8) matches
  on RGB alone, whatever the pixel's alpha, and stores transparent GIF pixels as (0,0,0,0), so
  a record with source (0,0,0) also turns every transparent pixel into (R',G',B') with alpha A;
  on a composited screen the frame's whole transparent background then shows as a translucent
  tint (14 shipped `.art` files map (0,0,0) → black at alpha 136 or 223; without a compositor
  the window shape, computed before the remap, keeps those pixels hidden). It likewise
  overwrites the alpha of semi-transparent pixels in static images. **Recommendation:** remap
  only pixels with alpha 255 (the Windows behavior).

Example: `changeling2-idle-left.art` maps (100,217,138) → alpha 111, same color — a translucent
body part.

## 7.6 Presentation rules

* **Scale.** Each image is drawn into its sprite's `Region` (which already includes the user's
  scale factor) with **nearest-neighbour** sampling: destination pixel (dx, dy) samples source
  pixel `(floor(dx·W/w), floor(dy·H/h))`. (The RI's GTK backend does not scale at all — a known
  limitation, not a requirement.)
* **Blending.** Premultiplied source-over.
* **Z-order.** Draw sprites in collection order, which the host sorts every frame
  (Chapter 8 §8.6): houses first, then everything else by ascending `Region.bottom`
  (= y + height), except that in games the scoreboard labels are sorted after everything else
  (Chapter 9 §9.5.6); equal keys keep their previous relative order (stable sort). Lower on screen
  ⇒ drawn later ⇒ in front.
* **Hit-testing** is rectangle-based (`Region`), not per-pixel. Mouse clicks on transparent
  pixels of a sprite *should* pass through to the desktop where the platform allows it (the RI
  relies on the OS for this).
* **Speech bubble.** When `SpeechText` is not none, draw it right after its sprite (so later
  sprites can cover it):
  * single line, no wrapping (only explicit line breaks split lines);
  * reference look: 12 px sans-serif, black text on white, 1 px black border, ~1 px padding;
  * horizontally centered on the sprite (`x = Region.x + Region.w/2 − bubbleW/2`), bottom edge
    ~1 px above `Region.y`;
  * clamped to the drawable area (if it would leave the left edge, shift right, else if it would
    leave the right edge, shift left; same for top/bottom with top taking precedence).
  * The bubble's lifetime is decided by the simulation (Chapter 6 §6.9), not the renderer.

## 7.7 Image loading and caching

* Images are referenced by path; cache decoded animations per path (paths are compared with the
  file system's case rules).
* The RI pre-loads the images of all ponies/houses about to be shown before starting the
  animation loop; images not pre-loaded are loaded on first draw. Either is acceptable.
* **Load failure.** In the RI an image that fails to load (§7.5 strictness, an invalid `.art`
  file §7.5.1) is never skipped. On Windows a failure while pre-loading cancels the whole launch
  (a non-fatal error notice, return to the pony-selection screen, nothing starts); a failure on
  first draw (e.g. a pony added while running) is an unhandled error that terminates the
  application. On the GTK backend a pre-load failure is fatal too. RI defect.
  **Recommendation:** log the failure and treat the image as empty: the sprite stays simulated
  but is not drawn.
* Identical frames may be shared; this is an optimisation only.

## 7.8 Notes for a Linux renderer (non-normative)

The RI on Linux runs under Mono with a GTK 2 backend: one undecorated, keep-above toplevel window
per sprite, RGBA visual when a compositor is present (else a 1-bit XShape), an input shape set to
the opaque pixels while the pointer is inside, and one popup window per speech bubble. Its
limitations — worth avoiding in a new implementation — are:

* images are **not scaled** (the scale option is disabled on non-Windows platforms);
* the computed z-order is **ignored** (window stacking is left to the window manager);
* speech bubbles are not clamped to the screen;
* manual keyboard control and window avoidance/containment are Windows-only;
* context-menu callbacks run on worker threads (as on Windows; see the RI defect in
  Chapter 1 §1.5).

Practical choices for a new implementation:

* **X11 + compositor:** a single full-screen (or per-monitor) override-redirect ARGB window,
  input shape (XFixes/XShape) updated each frame to the union of sprite opaque masks (or sprite
  rectangles), and "always on top" via `_NET_WM_STATE_ABOVE`. One window gives correct z-order
  and avoids per-window overhead.
* **Wayland:** use `wlr-layer-shell` (overlay layer) where available with an input region set to
  sprite areas; otherwise a full-screen transparent surface. Global cursor position is not
  generally available on Wayland: hover, cursor avoidance and drag require pointer events on
  your own surface, so set the input region to the sprites and track pointer enter/motion.
* The window-avoidance features need foreign-window geometry (X11: `_NET_CLIENT_LIST` +
  geometry queries; unavailable on Wayland) and MAY be omitted.
