# 4. House Definition Schema (`house.ini`)

Houses are optional static scenery sprites (Sugarcube Corner, the Library, …) that periodically
release ponies from their door and call ponies back in. Runtime rules are in
[Chapter 6 §6.13](06-simulation.md#613-houses-runtime); this chapter defines the file.

---

## 4.1 Location and encoding

* `Houses/⟨Directory⟩/house.ini` (lower-case file name). One directory per house; the directory
  name is the house's identifier for persistence (profiles do not currently store houses).
* Encoding and line endings exactly as `pony.ini` (§3.1): UTF-8, BOM skipped, LF/CR/CRLF.

## 4.2 Line classification

In file order:

1. Empty / whitespace-only → ignore.
2. First character `'` → comment (preserved by the writer, at the top).
3. **No comma** → the whole line, trimmed, is a **visitor** entry.
4. Otherwise the identifier is the text before the first comma, compared case-insensitively
   (untrimmed). Known identifiers below; **any other line is also a visitor entry** (the whole
   line, trimmed — commas included).

All keyed lines are split with the **QB** splitter (§3.3).

| Identifier  | Fields              | Kind / range            | Class     | Default |
|-------------|---------------------|-------------------------|-----------|---------|
| `name`      | `name,⟨Name⟩`        | text, not blank         | Required  | — (house name stays empty) |
| `image`     | `image,⟨file⟩`       | path (§3.4.1), must exist | Required | — (no image) |
| `door`      | `door,⟨X⟩,⟨Y⟩`       | int, int                | Defaulted | `0`, `0` |
| `cycletime` | `cycletime,⟨s⟩`      | int seconds, 0 … 3600   | Defaulted | `300` |
| `minspawn`  | `minspawn,⟨n⟩`       | int, 1 … 9999           | Defaulted | `1` |
| `maxspawn`  | `maxspawn,⟨n⟩`       | int, 1 … 9999           | Defaulted | `50` |
| `bias`      | `bias,⟨p⟩`           | decimal, 0 … 1          | Defaulted | `0.5` |

* A keyed line with a fatal issue is ignored (the property keeps its default/previous value).
  Repeated keys: last one wins.
* `image` sets the same file for both facings (houses never turn).
* `door` is the door position in **image pixels** relative to the image's top-left corner.
  Ponies are released from and recalled to this point (their anchor point is placed on it).
* `cycletime` — seconds between visitor cycles. RI defect: a value of `0` passes the range check
  but then makes the whole house fail to load. **Recommendation:** treat values < 1 as invalid
  (use the default).
* `minspawn` / `maxspawn` — the house will not recall when it has ≤ minspawn deployed ponies,
  and will not deploy when it has ≥ maxspawn deployed ponies.
* `bias` — probability that a (non-skipped) cycle tries to *deploy* rather than *recall*.

## 4.3 Visitors

* Each visitor entry is a pony **directory name** (case-sensitive).
* The special entry `all` (any case) means "any pony". Entries are otherwise not validated.
* A house with **no visitor entries is dropped** at load. A house whose file fails to load for
  any reason is dropped silently.
* Recommendation: also drop a house with no valid image.

## 4.4 Ordering

Loaded houses are presented sorted by Name (ordinal). The RI loads houses in parallel; order of
loading is irrelevant.

## 4.5 Canonical serialisation

Written by the RI's house options dialog (Chapter 8 §8.9), UTF-8 with BOM:

```
⟨comment lines⟩
name,⟨Name⟩
image,⟨file name⟩
door,⟨X⟩,⟨Y⟩
cycletime,⟨integer seconds⟩
minspawn,⟨n⟩
maxspawn,⟨n⟩
bias,⟨decimal, invariant culture⟩
⟨one visitor per line, or the single line "all" when every pony is selected⟩
```

## 4.6 Example

`Houses/Sugarcube Corner/house.ini`:

```
name,Sugarcube Corner
image,sugarcubecorner.gif
door,201,329
cycletime,300
minspawn,1
maxspawn,50
bias,0.5
ALL
```

Every 300 s of real time, with probability ½ nothing happens; otherwise with probability 0.5 the
house releases a random pony (any pony, because of `ALL`) at pixel (201,329) of its image, or
else, if it has released more than one pony that is still around, it calls a random idle pony
back; that pony walks to the door and disappears 3 s after arriving.

All eight shipped houses use `cycletime 300`, `minspawn 1`, `maxspawn 50`, `bias 0.5` and the
single visitor entry `all`/`ALL`.
