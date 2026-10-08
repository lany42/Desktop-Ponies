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
* `durations[]` — per frame, `delay × 10 ms` (GIF delays are in centiseconds), **no minimum
  clamp** (a delay of 1 means 10 ms; browsers would use ~100 ms);
* `loopCount` — from the NETSCAPE2.0 application extension (0 = forever); **1 if the extension
  is absent** (play once, then hold);
* `totalDuration = Σ durations`.

**Zero-delay frames are dropped** (they are intermediate compositing steps). Exception: if every
frame has delay 0, keep only the **last** frame. A static image is one frame, `loopCount = 0`,
`totalDuration = 0`.

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
            frame i                                                  # frame i on (Sᵢ₋₁, Sᵢ]
```

Fidelity notes (implement to match the RI exactly, or knowingly deviate):

* `loopCount = N > 0` means **N total plays**, then the last frame is held.
* The two branches have different boundary conventions: with variable frame durations a frame
  boundary at exactly τ still shows the *earlier* frame. Because ponies sample τ on a 40 ms grid
  and GIF delays are multiples of 10 ms, this is visible: delays `[40, 80, 40]` sampled every
  40 ms show frames `0, 0, 1, 1, (wrap) 0 …` — the last frame never appears.
* Frames shorter than 40 ms of simulated time can be skipped entirely (ponies only advance in
  40 ms steps).
* `ImageTimeIndex` restarts at 0 whenever a pony changes behavior, but **not** when its facing
  or its borrowed follow images change (Chapter 6 §6.7.3).

## 7.5 GIF compositing rules

An implementation can use any GIF decoder that produces fully composited RGBA frames, provided
these rules hold:

* Each output frame is a full logical-screen canvas. The canvas starts **fully transparent**.
* Pixels equal to the frame's transparent index are not written (the canvas shows through).
* Disposal (applied after a frame is emitted): 0 and 1 keep; 2 clears the frame's rectangle to
  **transparent** (not the background color); 3 restores the rectangle to its state before the
  frame was drawn.
* The background color index and pixel aspect ratio are ignored.
* Interlaced frames use the standard 4-pass order.
* Transparency is binary (GIF has no partial alpha); partial alpha comes only from `.art` maps
  (§7.5.1).
* RI strictness (a lenient decoder MAY accept these): a frame extending beyond the logical
  screen, disposal methods 4–7, unknown block types, or a missing trailer make the image fail to
  load. A failed image makes the sprite undrawable.

### 7.5.1 Alpha remapping files (`.art`)

An optional sidecar `⟨image name without extension⟩.art` next to an image (≈ 200 in the corpus)
gives semi-transparent colors:

* binary, a sequence of 7-byte records `R G B  A R' G' B'`: every **opaque** pixel whose color is
  exactly (R,G,B) is drawn with color (R',G',B') and alpha A;
* file length must be a multiple of 7; duplicate source colors are invalid (ignore the file);
* applied after decoding; transparent pixels are never remapped.

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
  (= y + height); equal keys keep their previous relative order (stable sort). Lower on screen
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
* context-menu callbacks run on worker threads.

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
