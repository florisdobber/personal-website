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

The whole site is a single page: a business card lying on a small stack of identical cards on a linen-covered table. You can pick the top card up, spin it, flick it and flip it. Everything is plain HTML, CSS and JavaScript with no libraries: markup and interaction in `_layouts/home.html`, all styling in `assets/css/main.css`. Card content comes from `_config.yml`.

### Layout & dimensions

- Real business-card aspect: **1.75 : 1**, at `min(520px, 86vw, 122.5vh)` wide.
- Every size on the card is in container units (`cqw`, a share of the card's width), so the card scales as one unit. No typography media queries.
- The card has real thickness (~0.4 mm at real size): five edge-coloured slices between the front and back faces, so it never vanishes edge-on.

### Orientation & physics

- Orientation is a **quaternion**, not Euler angles, so any combination of spins composes correctly and there is no gimbal lock. Each frame it's written to the card as a `matrix3d`.
- **Drag** rotates the card so the point under your finger follows your finger (rotation axis perpendicular to the drag).
- **Release** keeps the drag's angular velocity. Above `FREE_SPIN` (7 rad/s) the card spins freely with air drag; below `SETTLE_SPIN` (3 rad/s) a damped spring settles it face-up on whichever side faces you, turning it upright if it landed upside down.
- **Tap/click** (moved < 6 px, held < 450 ms) flips the card with the same spring; on a link it follows the link instead. Grabbing a spinning card stops it.
- **Hover** (mouse only) leans the card slightly under the cursor. The links are on the back, so the old problem of the lean making link hover flicker no longer applies on the front.
- **Lift**: while held or spinning the card rises a little toward you (`translateZ`) and its shadow spreads and softens.
- **Keyboard**: focus the card; Enter or Space flips, arrow keys spin.
- The animation loop sleeps when the card is at rest.
- `prefers-reduced-motion`: no free spin, intro toss or nudge; flips are near-instant.

### Light

One key light, up-left and in front (`LIGHT` in the script), drives everything:

- **Shading**: each face darkens as it turns away from the light, with a soft sheen placed from the light's half-vector.
- **Shadows** are the card's footprint projected flat onto the table (the 2D part of its rotation matrix), offset away from the light as it lifts. Three layers: sharp contact, soft and ambient. Each is a **pre-blurred image drawn once on a canvas**; only transform and opacity change per frame. A live CSS `filter: blur()` on a moving element leaves smeared trails in Safari.
- **The table** is lit by the same lamp: a pool of light up-left falling off toward the lower right where the shadows go.

### Showing only the facing side

`backface-visibility: hidden` is unreliable in Safari when a face has children on their own compositing layers (the animated lighting overlays). The script hides whichever face points away, using the sign of the face normal's z. Because the card's plane passes through its centre and the viewer is on the z axis, that test is exact even with perspective.

### Paper

The texture is generated in the browser on load (`makePaper`), seeded so it's the same on every visit, with nothing to download.

- A seamless 512 px tile of short, randomly oriented fibres plus fine tooth, turned into **slope-lit images for four light directions** (+x, −x, +y, −y). Each frame their opacities are set from where the light is relative to the visible face. The texture rakes up as the card tilts away from the light, flattens when it faces it, and disappears once that side is turned from the light, as real card stock does.
- The layers sit **over** the printed content, so the ink picks up the paper's tooth like ink on uncoated stock.
- A very faint cloudiness from fibre clumps; no even speckle (that reads as a noise filter).

### Linen table

Also generated in the browser (`makeLinen`): a plain weave whose character comes from **slubs** (thick-and-thin stretches along each thread), threads of slightly different thickness and tone, and soft uneven dyeing. It's lit from the same direction and baked into one static tile. Felt, limewash plaster, greyboard and a plain surface were tried; linen won.

### The stack

Two identical cards lie under the top one, slightly offset and turned, with real edges and contact shadows. Their front is cloned from the top card. They say "take one" without words, and when you lift the top card its shadow falls across them and the next card shows through.

### Teaching without words

No instructions are visible at first.

- If nobody touches the card for `NUDGE_AT` (4 s), it gets **one small knock**, as if the table were bumped: it turns ~34°, showing its edge and a sliver of the back, then settles.
- If there's still no interaction by `HINT_AT` (9 s), "Drag to spin · click to flip" fades in. It goes for good on the first interaction.
- The clock only runs while the tab is visible. Screen readers get the instructions from the card's label.

### Content

- **Front**: small `fd.` monogram (IBM Plex Serif) top-right; large, tight Inter name with the role beneath, indented right.
- **Back**: large `fd.` bottom-left; "You found the back." above GitHub, LinkedIn, email and website, with an underline that slides in on hover.

Taps on links are routed by the script (the card owns all pointer gestures), so a drag that starts on a link never opens it. Keyboard activation of a focused link works normally.

### Colors

Flexoki. Card `#FFFCF0`, ink `#100F0F`, muted `#6F6E69`, link hover `#24837B`, linen `#E2DDCF`, card edge `#E4DEC8`.

### Tuning

The constants at the top of the script are the feel: `K`/`C` (spring), `FRICTION`, `FREE_SPIN`/`SETTLE_SPIN`, `HOVER_TILT`, `LIGHT`, `RELIEF` (paper texture strength), `NUDGE_AT`/`HINT_AT`.

### Email obfuscation

`site.email` is stored as HTML entities in `_config.yml`. Browsers decode them in both visible text and `mailto:` hrefs, so a basic email-scraper that doesn't render HTML won't pick it up.
