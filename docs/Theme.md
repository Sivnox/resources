# Sivnox — theme contract

**Status:** active. Derived from `Build_Sheet-Filled.md` (`L15`–`L17`, `R29`–
`R32`, `O18`, `O19`, §10).
**Components:** `sivnox colours` (extractor), `sivnox theme apply` (fan-out),
`Theme.qml` (shell adapter).
**Rule:** commit-only fan-out. Preview never rewrites app configs.

---

## 1. Frozen token set (v1)

| Key | Role (Build Sheet §10) |
|---|---|
| `siv_ink` | primary text, borders on light |
| `siv_bone` | base light surface |
| `siv_grey` | middle — metadata, disabled, dim |
| `siv_signal` | accent — state, active |
| `siv_ink_dim` | secondary text on light |
| `siv_ink_soft` | tertiary text |
| `siv_bone_dim` | secondary light surface |
| `siv_grey_deep` | border on dark |
| `siv_signal_hover` | accent hover state |

The names are **roles**, not fixed lightness. In warm mode `siv_bone` is a light
surface and `siv_ink` is dark text; in dark mode the pair swaps lightness while
the keys keep their jobs. The example below is a dark token set.

The key set is **frozen** (`Constitution.md` 5.1). Adding or renaming a key is a
schema change: bump the version and write a migration note.

---

## 2. Mode policy (`L17`)

| Setting | Behaviour |
|---|---|
| `theme.mode = "auto"` | Light wallpaper → warm/bone mode. Dark wallpaper → dark mode. |
| `theme.mode = "warm"` | Force the warm/bone token set. |
| `theme.mode = "dark"` | Force the dark token set. |
| `theme.manual_mode` | `""`, `"warm"`, or `"dark"`. When non-empty, it overrides `auto` and persists across wallpaper changes. |
| `theme.warm_shift` | Shift extracted hues toward the ink/bone band. |

The extractor owns the policy. `Theme.qml` applies the result; it does not
recompute the mode.

---

## 3. Extractor — `sivnox colours`

```
sivnox colours <path> [--mode auto|warm|dark] [--json]
```

One-shot. No daemon, no state beyond the cache.

### 3.1 Algorithm

1. **Open and validate.** Check the path exists, is a supported image, and is
   readable. Reject early with exit 2.
2. **Decode to 64×64.** Decode and downscale in one pass. The full-resolution
   bitmap is never held. A 4K image must not allocate a 4K buffer.
3. **Discard the full image.** Only the 64×64 buffer survives.
4. **Median luminance.** Compute the median relative luminance of the 64×64
   pixels. Classify light if the median is above the threshold; dark otherwise.
   The threshold is a constant in `sivnox-core` and is part of
   `extractor_ver`.
5. **Resolve mode.** Use `--mode` if present; otherwise `theme.manual_mode`;
   otherwise `theme.mode` (`auto` uses the luminance classification).
6. **Pick the dominant hue.** Quantize the pixels in OKLCH, discard
   near-neutral pixels, and choose the hue with the highest chroma-weighted
   population. If no hue clears the chroma floor, use the neutral default.
7. **Warm shift.** If `theme.warm_shift` is true, rotate/clamp the hue toward the
   warm band and reduce excessive chroma. The result must still be a signal, not
   a neon accent.
8. **Derive the four bases.**
   - `siv_ink`: the text colour for the chosen mode.
   - `siv_bone`: the base surface for the chosen mode.
   - `siv_grey`: the midpoint/neutral.
   - `siv_signal`: the accent from step 6, adjusted for contrast.
9. **Derive the ramps in OKLCH.** Fixed lightness/chroma offsets from the bases:
   `siv_ink_dim`, `siv_ink_soft`, `siv_bone_dim`, `siv_grey_deep`,
   `siv_signal_hover`.
10. **Enforce contrast.** Text on its surface must be at least 4.5:1. Adjust
    lightness (not hue) until the pair passes. If the accent cannot be made
    legible as a signal on its surface, fall back to the neutral accent.
11. **Write the cache.** `~/.cache/sivnox/colours/<hash>.json` with
    `{ "tokens": { … }, "extractor_ver": 1 }`, atomically.
12. **Print** the token JSON if `--json`, then exit 0.

### 3.2 Output shape

This is a dark-mode example. Values change with the mode; the keys do not.

```json
{
  "tokens": {
    "siv_ink": "#cdc4ba",
    "siv_bone": "#0b0b0b",
    "siv_grey": "#958f87",
    "siv_signal": "#e2342a",
    "siv_ink_dim": "#b0a9a0",
    "siv_ink_soft": "#7a756e",
    "siv_bone_dim": "#17171a",
    "siv_grey_deep": "#3a3a3a",
    "siv_signal_hover": "#f05a50"
  },
  "extractor_ver": 1
}
```

The example is illustrative; the extractor computes the actual values. The key
names and the outer shape are the contract.

### 3.3 Cache

| Property | Value |
|---|---|
| Path | `~/.cache/sivnox/colours/<hash>.json` |
| Key | `blake3(file bytes) + extractor_ver` |
| Bound | 5 MB / 200 entries |
| Eviction | oldest mtime first |
| Hit target | under 2 ms |
| Invalidation | content change or `extractor_ver` bump |

Content hashing is deliberate: a same-size, same-mtime edit must not return a
stale palette.

### 3.4 Determinism

For a fixed `(file bytes, mode, warm_shift, extractor_ver)` the output is
identical. The extractor must not use random sampling or time-dependent
thresholds.

---

## 4. Shell adapter — `Theme.qml`

- Calls `sivnox colours <active path> --json` on start and after a commit.
- The CLI resolves the cache; `Theme.qml` does not need to know the hash or the
  cache path.
- Exposes the nine keys as read-only properties.
- Resolves `theme.mode` / `theme.manual_mode` through the extractor result, not
  through QML colour maths.
- Rebinds surfaces when new tokens arrive.
- If the command fails or the wallpaper is missing, uses the named-scheme
  fallback or the compiled default.
- Never writes the cache. Never hardcodes a colour in a surface.

Target: a commit recolours the shell within 200 ms (`T6`). Preview may apply
already-cached tokens by calling the same command; a cache miss does not block
the preview.

---

## 5. Fan-out — `sivnox theme apply`

```
sivnox theme apply <path> [--json]
sivnox theme use <name> [--json]
```

One-shot. Runs on commit only. `theme use` applies a named scheme (O19) and may
be used as a fallback when extraction fails.

### 5.1 Pipeline

1. Read `config.toml` (`theme.apps`, `theme.post_hook`).
2. Extract or read cached tokens for `<path>`.
3. Load templates from `/usr/share/sivnox/templates/`.
4. Render each template with `{{ key }}` substitution.
5. Write each target atomically (temp + rename + `fsync`).
6. Collect `{ ok, failed }`.
7. Run the post-hook if configured.
8. Print the result JSON if `--json`.

### 5.2 Templates

- Location: `/usr/share/sivnox/templates/`.
- Syntax: `{{ key }}` where `key` is one of the nine tokens.
- A missing token is a render error; the adapter is marked failed, not written
  with a blank.
- Templates are plain text. No conditionals, loops, or app-specific logic in the
  template language.
- An adapter is `template path + target path`. The exact adapter list is `P4`,
  ported from Caelestia during W4.

### 5.3 App list

`theme.apps` in `config.toml`:

```toml
[theme]
apps = ["kitty", "gtk", "qt", "btop", "cava", "fuzzel", "papirus"]
```

At minimum, `T18` requires kitty and GTK on commit, and W4 also targets btop.
The rest follow `P4`.

### 5.4 Post-hook (`O18`, `P3`)

- Default path: `~/.config/sivnox/hooks/post-theme`.
- Runs after all adapters, whether they succeeded or failed.
- Receives the token values in the environment:
  `SIV_INK`, `SIV_BONE`, `SIV_GREY`, `SIV_SIGNAL`, `SIV_INK_DIM`,
  `SIV_INK_SOFT`, `SIV_BONE_DIM`, `SIV_GREY_DEEP`, `SIV_SIGNAL_HOVER`.
- stdout/stderr are captured and logged to stderr/journal.
- Timeout: 10 seconds. A timeout is a failed hook, not a failed fan-out.
- Empty `post_hook` disables it.
- A non-executable or missing hook is logged, not fatal.

### 5.5 Failure policy

- One adapter failure does not stop the others.
- The result JSON lists `ok` and `failed`.
- A failed write leaves the previous target intact.
- The shell is never blocked by fan-out; it is a separate one-shot process.

---

## 6. Named schemes (`O19`)

- Directory: `~/.config/sivnox/schemes/*.json`.
- Each file is a hand-authored `TokenSet` with the nine keys.
- `sivnox theme use <name>` applies it and fans out.
- `sivnox colours` falls back to a named scheme if extraction fails or the
  wallpaper is missing.
- The default fallback scheme ships in `/usr/share/sivnox/schemes/`.
- Named schemes are not a theme marketplace and are not downloaded.

### 6.1 File shape

```json
{
  "name": "bone",
  "tokens": {
    "siv_ink": "#cdc4ba",
    "siv_bone": "#0b0b0b",
    "siv_grey": "#958f87",
    "siv_signal": "#e2342a",
    "siv_ink_dim": "#b0a9a0",
    "siv_ink_soft": "#7a756e",
    "siv_bone_dim": "#17171a",
    "siv_grey_deep": "#3a3a3a",
    "siv_signal_hover": "#f05a50"
  }
}
```

---

## 7. Performance

| Operation | Target |
|---|---|
| Extract (4K wallpaper, cold) | under 150 ms |
| Extract (cache hit) | under 2 ms |
| Shell recolour after commit | under 200 ms (`T6`) |
| Fan-out (kitty + GTK) | under 300 ms |
| Fan-out (all apps) | under 1 s |

The shell recolour target includes the extractor plus `Theme.qml` rebinding. If
extraction is slow, the shell may apply cached/fallback tokens first and then
the final tokens when extraction completes.

---

## 8. Testing

- Determinism: same image → same tokens.
- Cache hit: same image → no re-extraction; response under 2 ms.
- Content hash: same path, changed bytes → new tokens.
- Mode policy: auto/warm/dark and `manual_mode` persistence.
- Contrast: every generated token set passes 4.5:1 for text on surface.
- Fan-out: kitty and GTK files change on commit (`T18`).
- Preview: no app config is written during preview.
- Post-hook: receives all nine variables, timeout is enforced, failure is logged.
- Named scheme: applies and fans out; invalid file is rejected.
- Atomicity: a failed write leaves the previous target intact.

The full acceptance procedure is in `Testing.md`.

---

## 9. Open items

| ID | Item | Close by |
|---|---|---|
| P3 | Post-hook path | Confirmed: `~/.config/sivnox/hooks/post-theme`. |
| P4 | Caelestia fan-out file list and adapters | W4 planning |
| — | Exact OKLCH thresholds and warm band | W2 implementation, recorded in `extractor_ver` |

`P3` is closed by this document. `P4` remains open until the Caelestia engine is
inspected.
