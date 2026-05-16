# Personal website

My personal website, created using [Jekyll](https://jekyllrb.com) and hosted using GitHub pages and CloudFlare.

## Set up

- Instructions on how to set up Jekyll on macOS: https://jekyllrb.com/docs/installation/macos.
- `git clone https://github.com/florisdobber/personal-website.git`

Now run `bundle exec jekyll serve --livereload` to start a development server. For additional options, see https://jekyllrb.com/docs/configuration/options/#build-command-options.

> Local dev quirk: the Gemfile inherits `jekyll-theme-primer`, whose SCSS contains a non-ASCII byte. On a non-UTF-8 locale the SCSS converter errors out. Prefix the serve command with `LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8` if you see `Invalid US-ASCII character`. GitHub Pages doesn't have this issue. `.claude/launch.json` wires this up for the Claude Code preview server.

## Other elements

Besides the code, there are other elements on this website which might be hard to recreate. For my own, and anyone who is interested, I've left a short description on how I created these.

### Profile image

The profile image was created using [Fooocus](https://github.com/lllyasviel/Fooocus), based on an a portrait photo and applying the `MK One Line Art` style. It was inspired by [Victoria Arzumanova's](https://www.victoriaarzu.com) work.

### Favicon

The favicon is `fd.` in the [IBM Plex Serif font](https://fonts.google.com/specimen/IBM+Plex+Serif). I then used https://realfavicongenerator.net to convert it to a favicon.

### Color scheme

The color scheme used on this website is [Flexoki](https://stephango.com/flexoki) by [Steph Ango](https://stephango.com). It is heavily inspired by [Solarized](https://ethanschoonover.com/solarized/), but slightly less warm.

---

## Architecture & design choices

The whole site is a single page: an interactive 3D business card. All meaningful code lives in two files: `_layouts/home.html` (markup + inline JS) and `assets/css/main.css`.

### Layout & dimensions

- Real business-card aspect: **1.75 : 1**.
- `--card-w` is `min(86vw, 122.5vh, 520px)` — scales smoothly across portrait phones, short landscape windows, and desktops while never exceeding 520px wide.
- `--card-t` (thickness) is `0.8% × card-w` (~4px at desktop). Visible enough to read as a physical object without looking like a slab.
- Every inner font size and padding is computed from `--card-w`, so the card scales as one unit. No typography media queries.

### 3D model — six faces

The card is built from six absolutely-positioned faces inside `.card`, each centered at the card's middle with `top: 50%; left: 50%`, then transformed into place:

| Face        | Size      | Transform                                            |
|-------------|-----------|------------------------------------------------------|
| `face-front`  | W × H     | `translate(-50%, -50%) translateZ(t/2)`              |
| `face-back`   | W × H     | `translate(-50%, -50%) rotateY(180deg) translateZ(t/2)` |
| `face-top`    | W × T     | `translate(-50%, -50%) rotateX(-90deg) translateZ(h/2)` |
| `face-bottom` | W × T     | `translate(-50%, -50%) rotateX(90deg) translateZ(h/2)`  |
| `face-right`  | T × H     | `translate(-50%, -50%) rotateY(90deg) translateZ(w/2)`  |
| `face-left`   | T × H     | `translate(-50%, -50%) rotateY(-90deg) translateZ(w/2)` |

The `rotateY(180deg)` on the back mirrors content correctly when viewed from behind. All faces use `backface-visibility: hidden` so the inside of a face doesn't render through when the card flips past 90°.

The card sits in `.scene` (`transform-style: preserve-3d`) inside `.stage` (`perspective: 1600px`).

### Shadow — separate sibling element, not box-shadow

The shadow is a `.shadow` div inside `.scene`, **not** a `box-shadow` on the card. Box-shadow doesn't compose with `preserve-3d` correctly — in 3D scenes it renders in screen space rather than scene space. A dedicated element gives full control.

Key rules — every single one of these came out of a bug:

- **`translateZ(-320px)`**. Has to be deep. The card's furthest-back edge at 90° rotation is at `z = -card-w/2 ≈ -260px`. If the shadow is shallower than that, the card's tilted-back edge punches *through* the shadow and renders a gray translucent triangle on top of the card.
- **Width/height scaled to `1.22 × card`**. Perspective foreshortening at `z = -320` is `1600 / (1600 + 320) ≈ 0.833`, so `1 / 0.833 ≈ 1.2` keeps the shadow visually card-sized.
- **Radial gradient background, not solid color**. A solid `rgba(...)` rectangle has a hard edge that becomes visible the moment the card rotates and stops fully covering it (the "triangle" the design started with). Radial gradient fading to transparent inside the rectangle eliminates that.
- **`opacity: var(--shadow-vis)` where `--shadow-vis = |cos(rotX) · cos(rotY)|`**. This is the projected silhouette area. When the card is edge-on it has almost no vertical projection, so the shadow correctly fades to nothing.
- **`--shadow-dx` / `--shadow-dy`** shift the shadow opposite to the card's tilt, suggesting the card pivoting above a surface.

### Rotation model

The displayed rotation is just `baseX, baseY` — set by the user's drag and the decaying inertia after release. The card stays exactly where the user lands it.

An earlier iteration added a pointer-follow parallax offset on top of base (the card tilted a few degrees toward the cursor). It was removed because the constant subtle re-tilt shifted the perspective-projected position of links by fractions of a pixel under the cursor — enough to drop hover state mid-hover and make the underline animation flicker. The interaction now feels more solid in exchange for the slight loss of liveliness.

Animation loop per frame:

1. If inertia speed > threshold, decay velocity into `baseX/Y`.
2. Call `applyTransform` if anything changed.

`applyTransform` writes `rotateX(baseX) rotateY(baseY)` to the card, plus the shadow CSS variables.

### Drag interaction

- **5px drag threshold** before a pointerdown becomes a drag. Below that, the gesture is treated as a click on whichever link was under the cursor.
- `setPointerCapture` is only called *after* crossing the threshold, so small movements don't steal the click target.
- A `capture: true` click handler on the card calls `preventDefault` if `dragged === true`, so a drag never accidentally opens a link.
- Pointer events handle both mouse and touch with one code path.

### Colors

Warm Flexoki-inspired palette. The page is slightly darker than the card so the card reads as a physical object on a lit tabletop.

- Card front and back: **`#FFFCF0`** — same warm cream both sides. The back is discovered by flipping, not by a color cue.
- Page background: radial gradient `#FFFCF0` → `#F2F0E5` → `#E6E4D9`, lit-from-center. Subtle, not decorative.
- Card border: `1px solid rgba(16, 15, 15, 0.06)` — barely visible, enough to define the edge against the page.
- Edges: `#EFEAD6` (top), `#D9D2BC` (bottom), `#E4DEC8` (sides) — muted creams just darker than the front, reading as paper thickness. Static, no lighting overlay.

### Content & layout

- **Front:** identity-focused. Small `fd.` monogram (IBM Plex Serif) in the **top-right**; large, tight Inter name; muted role beneath. The name + role block is left-aligned but indented well right of the card's left padding so the right half of the card carries the weight. No portrait, no contact links — those are intentionally on the back.
- **Back:** two-column flex. Large `fd.` monogram in the **bottom-left** (diagonal mirror of the front's monogram). Right column: "You found the back." muted label at the top, then four contact rows (GitHub, LinkedIn, Email, Website) with icons. The back is a reward for discovering the flip interaction; the front never advertises it.

### Removed effects — and why

The card looked very different a few iterations ago. These were intentionally removed:

- **Idle auto-spin.** Felt automated, broadcast "look at me" — wrong tone for a business card.
- **Hover lift.** Too obviously interactive.
- **"Drag to spin" hint text.** Too explicit.
- **Paper grain (SVG turbulence overlay).** Read as "I applied an effect" rather than as physical paper.
- **Specular highlight.** A radial gradient on the face whose cutoff was perceived as a triangular shape during rotation.
- **Per-face Lambert lighting overlay.** Added a subtle dark wash whose gradient was also perceived as triangular at oblique angles.
- **Pointer-follow parallax** (a ±2–3° tilt toward the cursor on top of the base rotation). It shifted the perspective-projected position of links by fractions of a pixel under the cursor, breaking hover state and making the underline flicker. The interaction feels more solid without it.

If any of these come back, the bar to clear is: it must read as *the card being physical*, not as *an effect on the card*. Anything that's visible-as-an-effect breaks the minimalist intent.

### Micro-interactions kept

- **Drag to spin**, with momentum/inertia decay.
- **One-time settling wobble** on load (decaying sine on yaw + a smaller pitch at a different frequency; cancels on first pointerdown).
- **Link underline** animates left-to-right on hover. Pure CSS.
- **Dynamic shadow** that follows the card's tilt and fades when the card is edge-on.

### Files

- `_layouts/home.html` — single-page layout, markup, inline interaction JS.
- `_layouts/project.html` — *removed* (no longer used; projects collection deleted).
- `_layouts/about.html` — *removed*.
- `_projects/` — *removed*; back now holds contact links instead of selected work.
- `assets/css/main.css` — all styles. No SCSS, no partials, no preprocessor magic.
- `_config.yml` — `name`, `job-title`, `github`, `linkedin`, `email`. Editing here drives the card content.

### Email obfuscation

`site.email` is stored as HTML entities in `_config.yml`. Browsers decode them in both visible text and `mailto:` hrefs, so a basic email-scraper that doesn't render HTML won't pick it up.
