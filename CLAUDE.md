# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`reidify.tv` — a five-page brand site for a YouTube gaming creator (Reidify,
`@Reidifyy`). Hugo, no theme, templates written directly.

The site's jobs, in priority order: look legitimate to a brand considering a
sponsorship, push visitors to the YouTube channel and merch store, be
reachable for business. Design decisions below trace back to those three.

## Commands

```bash
hugo server --disableFastRender     # dev server on :1313
hugo --gc                           # dev build (unminified, no fingerprint)
hugo --minify --gc                  # production build, as CI runs it
```

Install the hooks once per clone:

```bash
pre-commit install
```

Standard hygiene hooks plus `tools/check-site.sh`, which builds the site and
checks for HTML comments in the output, an empty video grid, and link-policy
violations. Every check maps to a mistake that actually happened — keep it that
way, and keep it fast.

There are no tests, no linter, and no Node toolchain. `hugo` is the entire
build. CI pins Hugo **0.166.0 extended** — match that locally; `extended` is
required for Hugo Pipes CSS.

Regenerate `data/videos.json` by hand the way CI does:

```bash
curl -sSfL -o /tmp/feed.xml \
  "https://www.youtube.com/feeds/videos.xml?playlist_id=UULFE6h5SQHI8Xv3v_F7DZFtTA"
python3 .github/scripts/feed-to-json.py /tmp/feed.xml data/videos.json
```

## Hard constraints

- No Node dependency tree, no build-time JS toolchain, no client-side framework.
- No `localStorage` / `sessionStorage`.
- The site must still build untouched in two years. Pin versions; prefer
  non-deprecated Hugo APIs even when the old one still works.
- Google Fonts is the only external request the page makes. It is runtime, not
  build-time — a blocked Google must degrade to the fallback stack, never break
  the build.

## Architecture

### No `baseof.html`, by design

`layouts/index.html`, `layouts/404.html`, `_default/single.html` and
`_default/links.html` are each standalone full documents that include
`partials/head.html`, `nav.html` and `footer.html` directly. This matches the
repo layout the brief prescribes. Adding a base template would be a structural
change, not a cleanup — don't do it incidentally.

### The Links page is driven by front matter, not content

`content/links.md` sets `layout = 'links'` and defines the buttons as
`[[params.links]]` entries (`name`, `url`, `note`, `icon`, `style`).
`_default/links.html` iterates them.

**These links are peers and share one treatment.** Only the top entry is
`style = 'primary'` (filled yellow, marking the main destination); everything
else is identical and is told apart by its icon, not its colour. An earlier
pass had three colour treatments across four peer links, which implied a
hierarchy that did not exist. Keep exactly one filled yellow button here.

**The page has no body copy, deliberately.** The brief's line about replacing
Linktree and being opened on a phone is an instruction about how to *build* the
page, not text for it — an earlier pass mistakenly rendered it as a subheading.
Heading, then buttons.

That note lives in `links.md`'s front matter rather than its body on purpose:
`markup.goldmark` has `unsafe = true`, so an HTML comment in the body renders
into `.Content`, ships to the browser, and keeps an empty `.prose` div alive.
**Put editorial notes in front matter, not in the Markdown body.**

Unconfirmed destinations stay commented out rather than being guessed.
Instagram and Discord are commented out; YouTube, Twitch and TikTok are live.

**Merch is hidden because the shop is not live.** Uncommenting `shop` in
`hugo.toml` restores the hero button and the footer link — both are wrapped in
`{{ with site.Params.shop }}`, so one line controls them. The Links page entry
is front matter and cannot read a site param, so its block must be uncommented
separately; both TODOs point at each other.

### Video feed pipeline

The GitHub Action curls the feed, converts it to `data/videos.json` via
`.github/scripts/feed-to-json.py`, and `partials/video-grid.html` renders it.
Nothing in the workflow schedules itself: the self-hosted Dkron cluster fires
`workflow_dispatch` hourly on the hour, and that is the only thing keeping the
grid current. Actions minutes are unlimited on a public repo, so there is
nothing to ration.

**A stuck deploy job silently freezes the site.** The `pages` concurrency group
holds one running and one pending run; every later dispatch is cancelled as
superseded. In October 2026 a Deploy job sat "queued" for four days and 82
hourly runs were cancelled without a single red X. Both jobs now have
`timeout-minutes: 15` so a hang fails loudly. If the grid looks stale, check
`gh run list` for a `queued` run and `gh run cancel --force` it.

The fetch step is **fatal**. `curl` retries transient blips — with
`--retry-all-errors`, because plain `--retry` skips 404s, which YouTube's feed
endpoint returns intermittently; anything that survives that is a real problem — a feed format change or a sustained block —
and should be loud. Silent staleness is the worse failure here: a sponsor-facing
page serving a frozen video list that nobody notices beats a red X in Actions.

A failed build does **not** take the site down. Pages keeps serving the last
successful deployment, so the cost is only that unrelated changes cannot ship
until it is fixed.

`.github/scripts/feed-to-json.py` validates before it opens the output file, so
a parse failure can never truncate `data/videos.json`. Verified: malformed XML
exits 1, a zero-entry feed exits 1, a bad channel id fails at `curl` (56), and
the target is untouched in all three.

**`data/videos.json` is committed on purpose — do not gitignore it.** It looks
like build output, but it is what lets a fresh clone and offline local dev build
a real grid. CI overwrites it in place and never commits it back. The cost is
that regenerating locally dirties the tree; that is the intended trade.

**The build pulls the long-form playlist feed, not the channel feed.** YouTube
auto-generates per-type upload playlists whose ids are the channel id with `UC`
swapped for a prefix:

| prefix | playlist | contents |
|---|---|---|
| `UU` | Uploads | everything, Shorts mixed in |
| `UULF` | Videos | **long-form only** — what the build uses |
| `UUSH` | Short videos | Shorts only |

The workflow derives it: `PLAYLIST_ID="UULF${CHANNEL_ID#UC}"`.

This matters because the feed is hard-capped at **15 entries with no
pagination** (verified: `max-results`, `start-index`, `max_results` and `num`
all return exactly 15). Roughly half of any channel feed is Shorts, so the
channel feed yields only ~7 long-form videos. `UULF` returns 15 long-form and
reaches ~7 weeks back instead of ~3.

Verified every `UULF` entry against the Shorts test — 0 of 15 were Shorts.

An earlier pass filtered Shorts out of the channel feed by probing
`i.ytimg.com/vi/<id>/oar2.jpg` per video (it returns 1080x1920 for a Short and
404s otherwise). That works and is a useful trick, but `UULF` makes it
unnecessary: no per-video requests, no `short` flag, no filter. If you ever
build a Shorts section, point a second fetch at `UUSH` rather than
reintroducing the probe.

Notes that matter when touching this:

- Read it as **`hugo.Data`**, not `site.Data` — the latter is deprecated as of
  Hugo 0.156.
- Thumbnails are **hotlinked** from `i.ytimg.com` on purpose: it costs nothing
  and the image updates when he changes a thumbnail, with no rebuild. Do not
  download them or run them through Hugo Pipes.
- The feed returns sharded hosts (`i1`/`i3`/`i4.ytimg.com`); the script
  normalises them to `i.ytimg.com` so the single `preconnect` in `head.html`
  covers every thumbnail.
- Feed thumbnails are `hqdefault.jpg`, which is 4:3 with letterbox bars baked
  in. The grid crops them off with `object-fit: cover` on a 16:9 box. `hq720`
  is sharper but 404s on some uploads, and `srcset` cannot fall back from a 404.
- The feed also carries **per-video view counts** (`media:statistics`) and
  **likes** (`media:starRating`), and the **channel creation date** at feed
  level. The brief is wrong that view counts need the Data API — they don't.
  They are deliberately not displayed: median is ~630 views/video, and hard
  numbers that low work against the site's brand-legitimacy job. Don't add
  them just because they're free.
- Subscriber count is the one figure only `channels.list` provides, and it
  needs an API key. Deliberately deferred — see Open TODOs.

### CSS

`head.html` concatenates `assets/css/tokens.css` + `assets/css/main.css`
through Hugo Pipes, in that order (tokens must come first), then minifies and
fingerprints with SRI in production only.

`tokens.css` holds **custom properties and nothing else**. The site is
dark-only; a light theme was drafted and removed.

### Link `target` / `rel` — one place

`partials/is-external.html` returns a boolean, and every anchor pairs it with
the literal `target="_blank" rel="noopener"`. `layouts/_markup/render-link.html`
applies the same rule to links written in Markdown.

**`render-link.html` ends with a `{{- /**/ -}}` trim marker.** A render hook's
trailing newline lands in the output and renders as a space before punctuation
following an inline link ("email foo@bar.com ."). The marker makes the file
safe for *end-of-file-fixer*, which would otherwise reintroduce it.

**`noopener`, deliberately not `noreferrer`.** `noopener` is the security
control. `noreferrer` would also strip the `Referer` header, and every external
link here points at one of the owner's own properties (YouTube, the shop,
TikTok) — stripping it makes his own traffic land as "direct" in his own
analytics. If a link to a genuine third party is ever added, put `noreferrer`
on that link specifically.

`mailto:` gets neither attribute: `target="_blank"` on a mailto leaves an
orphaned blank tab, and `rel` is meaningless on a non-http scheme.

### Icons and the footer

`assets/icons/*.svg` are hand-authored on a 24x24 grid and inlined by
`partials/icon.html`. Inline rather than `<img>` or a sprite so they inherit
`currentColor` and cannot drift out of the palette. They are hand-drawn because
an icon library would mean npm, and there are four of them. The YouTube and
TikTok marks are trademarks — fine for linking to his own profiles, but the
official brand assets should eventually replace these, same as the logo.

Icons stay monochrome. Do not tint them to brand colours — YouTube red is the
vermilion the brief rules out.

**The footer complements the nav; it does not repeat it.** The nav carries
every internal page, so the footer carries only off-site destinations. The
business email is deliberately not in the footer: it used to render on every
page, so About, Contact and Links each showed the same address twice. Contact
owns it and is one click from anywhere. Putting it back is reasonable — it was
removed for repetition, not because it was wrong.

### The logo

`assets/img/logo.png` (128x128, transparent) is the nav mark and the favicon.
`static/apple-touch-icon.png` (180x180, opaque) is the iOS tile.

Both are derived from the official artwork. That file is an SVG in name only:
it is a 1.18MB wrapper holding two embedded 1025x1025 rasters — an RGB image
and a greyscale luminance mask supplying alpha — inside a 16:9 `viewBox` with
the square logo floating in the middle. Used directly it renders as a wide box
with a small logo, and it is far too heavy for a favicon.

To regenerate from a new export:

```bash
# pull the two rasters out and recombine them into real RGBA
python3 -c "..."                 # extract base64 <image> payloads
magick rgb.png mask.png -alpha off -compose CopyOpacity -composite rgba.png

magick rgba.png -resize 128x128 -strip PNG32:assets/img/logo.png
magick assets/img/logo-512.png -background none \
  -define icon:auto-resize=48,32,16 static/favicon.ico
magick rgba.png -resize 180x180 -background '#29C2D5' -alpha remove -alpha off \
  -strip static/apple-touch-icon.png
```

Notes:

- The disc is a perfect circle inscribed in the square canvas, touching all
  four edges (aspect 1.0000). Do not re-crop or re-centre it.
- The iOS tile must be **opaque with square corners** — iOS applies its own
  rounded-rect mask. The corners are filled with `#29C2D5`, the disc's own edge
  colour, so the circle boundary disappears.
- 128px covers the 32px nav slot at 3x. Going larger is mostly wasted bytes:
  256px triples the file for no visible gain.
- Nothing regenerates these automatically. Redo them if the logo changes.

`assets/img/logo-512.png` (512x512 RGBA, lossless) is the regeneration
source for both. It is deliberately not quantised — derivatives compound
artifacts. For anything larger than 512 go back to the Canva original; the
export we started from is 1025x1025 and clean (no compression ringing, 75k
colours), which already exceeds YouTube's 800x800 avatar recommendation.

`assets/img/og-image.png` (1200x630) is the social share card, wired to
`og:image` and `twitter:card=summary_large_image`. Regenerate it from
`tools/og-image.html`:

```bash
firefox --headless --screenshot assets/img/og-image.png \
  --window-size=1200,630 file://$PWD/tools/og-image.html
```

It hardcodes the tagline, so update it there if the tagline changes. The URLs
must be absolute — scrapers do not resolve relative image paths — which is why
the template uses `.Permalink` rather than `.RelPermalink`.

Two things about that file:

- **Fonts are embedded as base64, not linked.** Firefox screenshots on the load
  event, before a linked webfont has arrived, so a linked face silently renders
  as the fallback. Neither `display=swap` nor `display=block` avoids it.
- **The `h1` sets `font-weight: 400` explicitly.** `h1` inherits bold from the
  UA stylesheet, Bangers ships only a 400 weight, and the browser then fakes a
  bold by smearing the glyphs — visibly fatter than the hero, at the same
  advance width. `.display` sets 400 for the same reason; anything using Bangers
  must.
- **The logo carries a white drop-shadow halo.** The disc is `#29C2D5` against a
  `#29B6E8` band — deltaE 19, i.e. clearly different but not different enough to
  separate cleanly. The halo separates them deliberately rather than pretending
  they match.

The logo's disc does not match `--accent`, and cannot: it is a gradient running
`#29C2D5` at the edge to `#5CF7F9` at the centre (deltaE 19 to 36 against
`--accent`). Do not retune `--accent` to chase it. The only place the two ever
sit together is the share card, where the halo handles it, and matching the edge
would still leave the bright centre standing proud.

## Deployment

GitHub Pages, via `.github/workflows/build.yml` — a build job that uploads a
Pages artifact and a deploy job that publishes it. Same shape as the owner's
other sites.

The repo's **Settings > Pages > Source must be "GitHub Actions"**, or
`actions/configure-pages` fails the build.

`baseURL` comes from `steps.pages.outputs.base_url` rather than being
hardcoded, so it follows whatever Pages reports (custom domain or
`*.github.io`). `og:image` uses `.Permalink`, so it inherits that automatically.

The custom domain lives in `static/CNAME`, so it ships in the artifact rather
than existing only as a repo setting.

Pages limits, checked against the docs, with plenty of room:

| limit | documented | this site |
|---|---|---|
| builds per hour | 10 soft — **does not apply** to a custom Actions workflow | 1/hour |
| deployment timeout | 10 minutes | ~1 second |
| published size | 1 GB | 280 KB |
| bandwidth | 100 GB/month soft | ~44 KB per uncached first view |

The builds-per-hour exemption is why an hourly cron is fine. Bandwidth stays
low because thumbnails are hotlinked from `i.ytimg.com` and never touch the
origin; repeat views cost only the HTML, since CSS and the logo are
fingerprinted.

**This was briefly on Cloudflare Pages.** That was justified while a Pages
Function proxied the video feed. Once the site became purely static, Cloudflare
only added an account, an API token, two repo secrets and a Direct Upload
project, for no capability GitHub Pages lacks. If edge compute is ever needed
again that is the reason to revisit — not performance, since the domain is
proxied through Cloudflare either way.

## Design rules

The palette is derived from the channel art. Do not build a dark desaturated
"gamer" site; do not add acid-green or vermilion.

- **Saturated cyan appears as a field**, and only where no thumbnail competes —
  the hero band and the footer rule. The video grid stays on deep teal, which
  is what keeps the thumbnails readable.
- **Anything on a cyan field takes the black outline (`--ink`), or it
  vanishes.** White on `#29B6E8` measures 2.35:1 and fails AA outright; the
  outline is doing the real work. This generalises the brief's rule about the
  bolt.
- **Exactly one filled yellow button per view.** Other yellow is a rule, a bar,
  or an outline. (The brief's stricter "yellow is actions only, two at most"
  was relaxed by the owner, who wanted turquoise and yellow to carry the page;
  this is the guardrail that replaced it.)
- Display face (Bangers) on the `h1` and nav mark **only** — it is unreadable
  at body sizes. Body is Nunito.
- Single column, left-aligned, generous vertical rhythm. The video grid is
  3-up on desktop, 2-up tablet, 1-up mobile, at 16:9.
- Avoid: numbered section markers, all-caps eyebrow labels, identical rounded
  cards for every block, entrance animations on scroll.

### Motion

One page-load moment (the wordmark rises) plus a slow ambient drift on the
hero band: rays rotate once per 180s, glow breathes on a 9s cycle — 9 is
deliberately not a divisor of 180 so the two never sync into a visible pulse.

The rays use `transform`, **not** an animated conic-gradient angle: without
`@property` support an animated custom property degrades to a discrete jump
each cycle, which is worse than no animation. The rotating layer is a square
centred exactly on the ray origin (rotating a viewport-sized box would make the
rays wobble around the wrong point), sized `200vw` so a corner can never swing
into view.

`prefers-reduced-motion: reduce` zeroes **both** `animation-duration` and
`animation-delay`. Overriding only duration is a trap: a `backwards`-filled
animation holds its invisible from-state for the whole delay regardless.

## Quality floor

Responsive to 360px, visible keyboard focus on every link and button,
`prefers-reduced-motion` respected, semantic heading order, and body plus muted
text passing WCAG AA against their backgrounds.

The dark theme passes everywhere. If a light theme is ever added, `--link`,
`--text-muted` and `--rule` are the three that failed AA on white in the
earlier draft.

**Re-measure contrast after any palette change.** `--text-muted`, `--link` and
anything sitting on the cyan band are the ones that fail.

## Do not build

An embedded player, comments, or playlists — this is not a replacement for
YouTube, and video cards link out rather than embedding. A newsletter signup
(the audience skews under 13; collecting their emails triggers COPPA
parental-consent requirements), a cart or checkout or product
pages (merch lives entirely on the external `shop` subdomain), a CMS admin
panel, a contact form (no backend to receive it — `mailto:` only), or a blog,
tag pages, or pagination.

## Verification gotchas

Hard-won; they will waste your time otherwise.

- **Headless Chrome clamps its window to a ~500px minimum width.** Screenshots
  at 360/390px lay out at 500 and crop, which looks exactly like horizontal
  overflow. Use Firefox for narrow widths, and confirm real overflow by
  comparing `document.documentElement.scrollWidth` to `clientWidth` rather than
  by eye.
- **Firefox headless needs a fresh `-profile` per run** or the screenshot
  silently fails to write.
- Force `ui.prefersReducedMotion=1` in that profile for deterministic captures.
  It settles the animations *and* exercises the reduced-motion path. Without
  it you will catch the hero mid-animation.
- **`hugo --minify` strips attribute quotes**, so `target="_blank"` becomes
  `target=_blank`. Auditing built HTML with a naive regex will report false
  failures — use `html.parser`.
- The skip link sits at `left: -9999px` and will always show up as an
  "overflow" offender. It is intentional.
- **Kill `hugo server` before comparing builds.** A running server rewrites
  `public/`, so a production build you just made can be silently replaced by a
  dev build (livereload script, unminified) — any byte comparison against it is
  meaningless. Check for `livereload.js` in the output if a diff looks strange.
- **Never `rm -rf resources/` while `hugo server` is running.** It breaks the
  running server's Pipes pipeline, the page is then served with no stylesheet,
  and images fall back to their `width="480"` attribute — which looks exactly
  like a horizontal-overflow bug at narrow widths. Before believing any
  overflow finding, assert the CSS actually applied (e.g. `body`'s computed
  `background-color` is not transparent). Restart the server after any build
  that clears `resources/`.

## Adding channel metadata

The channel start date (July 2025) is hardcoded in `content/about.md`'s front
matter — it comes from the feed's top-level `<published>` and can never change,
so it is not worth a build step.

If subscriber count is ever wanted: `channels.list?part=statistics` with an API
key, at build time with the key as a GitHub Actions secret. Public counts are
rounded to 3 significant figures so they cannot change often. Quota is 1 unit
per call against 10,000/day. Follow the same non-fatal pattern as the video
step.

Audience demographics and watch time are **not** available this way — they are
owner-only via the Analytics API with OAuth, which is a much larger lift.

## Open TODOs

Deliberately unanswered; leave as TODOs rather than inventing answers.

- Whether Instagram or a Discord exist.
- Subscriber count and milestones for the About page. Start year is now filled
  in. See "Adding channel metadata" — the plumbing is easy; the judgement call
  is whether the number helps or hurts.
- GitHub Pages needs enabling once: Settings > Pages > Source > GitHub
  Actions. Until then `actions/configure-pages` fails the build.
