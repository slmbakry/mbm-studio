# MBM Studio: notes for Claude

Read this before changing the tool. Keep it current: when something changes, edit this
file in the same commit.

## Talking to the owner

- The owner is Mohamed. Reply in Egyptian Arabic. Text shown in the tool is plain Arabic.
- Never mix Arabic and English in one line or sentence: it breaks text direction. Put an
  English term, product name, setting or code on its own line, in a code box or its own
  table cell. Numbers are fine inside Arabic lines.
- Before substantial work, say in a few lines what you understood and the approach, and
  wait for a go-ahead. Ask short questions rather than assume.
- The owner does not read code. Report outcomes, what was tested and what was not.

## What it is

A one-page tool that makes social media images for several brands. The owner picks a
brand, a platform and size, writes the text, picks a design direction from a gallery,
adjusts it, and downloads a PNG. Everything is drawn on a canvas in the browser; text is
real text in the brand's fonts, never text baked into generated images.

It lives in two places, the same file in both:

| Where | Address | Brands are saved |
|---|---|---|
| GitHub Pages (this repo) | `https://slmbakry.github.io/mbm-studio/` | In that browser only (`localStorage`). Moved between devices with the export and import buttons |
| Claude artifact | `https://claude.ai/artifact/GwFYn9BK19kT6Lqa4Ghuy8` | In the artifact's database, synced across the owner's devices; logos as artifact assets |

The page detects which one it runs in: `window.claude.use(...)` exists only inside Claude.
Without it, brands use `localStorage`, logos are stored as PNG data URLs, and downloads use
a normal link. When `index.html` changes, republish the same file to the artifact too
(the artifact declares `db`, `assets`, `user` and `downloads`).

A separate copy of the older single-brand version runs inside the Waraqak platform at
`/admin/studio` (repo `slmbakry/waraqak`, file `app/api/studio/html.ts`). It is not
updated from here.

## Structure of `index.html`

One self-contained file: CSS, markup and one script. No build step; edit it directly.

- `PLATFORMS`: 10 platforms (Facebook, Instagram, LinkedIn, TikTok, YouTube, X, Threads,
  WhatsApp, Snapchat, Pinterest). Sizes include 1080x1080, 1080x1350, 1080x1440, 1080x1920,
  1000x1500, 1200x630/627, 1280x720, 1600x900.
- Covers: a format may have `safe:{x,y,w,h}` (Facebook page cover 1640x624, LinkedIn
  profile banner 1584x396, YouTube channel banner 2560x1440). `paint()` then draws the
  design in an offscreen canvas of the safe size, places it at the safe rect, stretches its
  left and right edge columns sideways and fills above and below with the average colour of
  its top and bottom rows. Hits are shifted by the safe offset; `place()` uses `opts.FW`/`FH`
  (full size) for drag offsets so dragging follows the pointer. `opts.FW` is set only in
  this mode; a design can check it (the pencil poem shortens its pencil so it does not
  touch the edge and stretch). Safe rects follow each platform's crop: YouTube's
  all-devices area is the centre 1546x423 (the design uses 846x423 inside it); LinkedIn's
  is on the right, away from the profile photo.
- Brand tokens: the designs were drawn for Waraqak's colours, so every colour in the
  drawing code is `PAL[i]`, computed by `retheme()` from the current brand's four colours
  (`primary` dark, `accent`, `highlight`, `paper`) through `RECIPES` (each colour is a base
  colour mixed towards white or black). With Waraqak's own colours the original values are
  used exactly. `C`, `BASE`, `DK`, `LT`, `CL` are rebuilt by `mkC()`, `mkBASE()`,
  `mkDKLT()`, `mkCL()`. New drawing code should use `C.navy`, `C.green`, `C.greenD`,
  `C.gold`, `C.bg`, `C.ink`, `C.muted`, `C.soft`, `C.softer`, or `hexMix(...)` of them,
  never fixed brand hex values.
- Fonts: `READEX` (headings) and `PLEX` (text) are the brand's two fonts. `FONTS` lists
  the Arabic Google Fonts offered, with the weights each one really has (a wrong weight
  makes Google Fonts refuse the whole stylesheet).
- `makeG()`: the drawing helpers for one image: `g.stack()` lays out ad text with
  auto-shrink, `g.poem()` lays out verses (two hemistichs side by side when every line
  contains `*`), `g.logo()`, `g.url()` (the brand's handle). Each element is drawn through
  `g.place(key, rect, fn)`, which applies the owner's drag, size and colour edits and
  records the hit area for the editor.
- `DIRS`: 15 ad directions (editorial, swiss, luxury, depth, glass, concept, cinema,
  collage, poster, split, minimal, offer, and three classic ones). A direction may have
  `dark(v)` saying whether its ground is dark (used for the photo tint); otherwise the old
  per-id rule in `paint()` decides. `offer` draws the eyebrow inside a gold burst badge
  itself (through `g.place('eyebrow', …)`), not in the stack. `PDIRS`: 6 poetry directions (notebook, classic,
  pencil, single, crumpled, night). Each draws differently when `g.v` is 1 (the
  "alternate composition" button). Variations pick six direction and composition pairs.
- `PRESETS`: brands built into the code (Waraqak with its logo, Osool Residence, Osool Rental,
  and "ورقة وقلم رصاص"). Brands the owner creates or edits in the page are stored separately
  and override a preset with the same id. To make a brand appear everywhere, add or update
  it in `PRESETS`.
- Logos: `builtin:light` and `builtin:dark` are Waraqak's (base64 in `LOGO`); other built-in logos are in `LOGO_EXTRA`, named `builtin:<key>`. Other brands
  use uploaded images. With no logo, the brand name is drawn as text; with only a light
  logo, it sits on a white plate on dark backgrounds.

## Saved designs ("تصميماتي", top of the panel)

`designs` holds saved designs by id (`d` + base-36 time): `{name, createdAt, updatedAt,
thumb, st, img}`. `thumb` is a 360px JPEG data URL; `st` copies the `DKEYS` fields of
`state` (brand, size, direction, texts, edits, variants, video settings); `img` is the
uploaded photo, scaled to 1600px at most (PNG in free mode, JPEG otherwise): an artifact
asset inside Claude, a data URL elsewhere. `userImgRef` remembers the stored copy of the
current photo so saving again does not upload it again. Stored in `localStorage`
(`mbm-designs`) and, inside Claude, in the `designs` collection of the database, synced
like brands. If the browser storage is full the design is saved without its photo, and
the page says so. `state.openDesign` is the design being edited: "حفظ التعديلات" updates it,
"حفظ كنسخة جديدة" makes a copy, "تصميم جديد" detaches. Delete needs a second tap.

## All sizes in one file (step ٤)

`exportAll()` renders the current design at every format of the ticked platforms (covers
included), as PNGs, and packs them with `makeZip()`: a small stored (uncompressed) ZIP
writer with CRC-32 and the UTF-8 name flag, one folder per platform, files named with the
Arabic format name and size. The current platform and size are restored afterwards.

## Video (step ٥)

Animated version of the current design, 5, 8, 10 or 15 seconds, 30 fps, at the format's
size. Not offered for cover formats (those with `safe`).

- Motion: `opts.anim = {t, D, style}`. `paint()` zooms the whole frame from 1 to 1.035
  over the clip. `place()` calls `animApply()`, which delays each element by its order in
  `AORD` (image, eyebrow/ptitle, title/verses, rule/lead, poet/body, cta, logo, url) and
  animates it in over 0.8 s. Styles: `rise` (fade up), `write` (right-to-left wipe),
  `pop` (scale with overshoot), `slide` (in from the right), `zoom` (from 1.35 down to
  1), `drop` (from above with a bounce). Backgrounds drawn directly in a direction stay still, so
  every direction animates without its own code. `state.vstyle`, `state.vdur`.
- Preview: `vStart()`/`vStop()` loop the clip in the main canvas, with the music.
- Export, `vExport()`: first `vEncodeWC()` (WebCodecs H.264, plus AAC when music is added,
  muxed by the embedded `mp4-muxer` 5.2.2, MIT, in its own script tag). Faster than real
  time and frame-exact. If H.264 or AAC encoding is missing, it falls back to
  `vEncodeMR()`: `MediaRecorder` on `cv.captureStream(30)` in real time, preferring MP4
  with H.264 and AAC. Linux Chrome has no AAC, so there the music comes out as Opus; on
  Windows, Mac, Android and iPhone the browsers have AAC. WebM is the last resort and the
  page warns that Instagram may refuse it.
- Music: an uploaded audio file, decoded with Web Audio, starts at 0, is cut to the clip
  length and fades out over the last 1.2 s (`vMixAudio()`). Kept in memory only.

## Brands

| Brand | Status |
|---|---|
| ورقك | Complete, with logo |
| أصول ريزيدنس | Colours and font (Cairo) from the Osool design system artifact. Logo from the owner's Illustrator file (white outlined "OSOOL Residence" with the keyhole and key), converted to two transparent PNGs: dark ink for light grounds, white for dark grounds (`builtin:osool-light`, `builtin:osool-dark`) |
| أصول للإيجارات | Separate company from Osool Residence. Black `#1C1C1A` and gold `#B08D57`, highlight `#D9BE8C`, paper `#FAF8F4`; headings Amiri, text Noto Naskh Arabic; handle osoolrental.com; no tagline. Logo is the English-only "OSOOL / RENTAL" serif wordmark with gold rules (Latin Modern Roman as paths), as two transparent PNGs (`builtin:osool-rental-light`, `builtin:osool-rental-dark`) |
| ورقة وقلم رصاص | Poetry page, tagline "شعر بالعامية المصرية". Graphite `#2B2F33`, blue `#2E6F9E`, pencil yellow `#F2C14E`, paper `#F8F6F0`; headings Aref Ruqaa, text Readex Pro. Logo is a lined sheet with a pencil beside the name in Ruqaa, as two transparent PNGs (`builtin:paper-pencil-light`, `builtin:paper-pencil-dark`), 900px wide, 256 colours |

## Testing

Check every direction at 1080x1080, 1080x1350, 1080x1920 and 1200x630, with short and
long text, and the page itself at phone width (390px). Look at the images, not only for
script errors. If the environment cannot load Google Fonts, previews use a fallback
font: say so when reporting.

## Limits to keep in mind

- Variations are a random pick of the built directions, not generated designs.
- No AI image generation is built in; the owner uploads their own photo as a background
  or a free image.
- The Pages site is public: anyone with the address can use the tool.
