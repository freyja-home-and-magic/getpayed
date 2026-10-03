# App Store screenshots

Drop device screenshots in and run:

    ./make-screenshots.sh ~/Downloads/IMG_*.png

Argument order becomes store order (`01.png` first), so pass them in the
order you want them to appear on the listing rather than whatever order the
shell globbed.

That imports them to `originals/` (renumbered 01..NN in argument order) and
generates every size. Re-run with no arguments to regenerate from `originals/`.

| Folder | Size | Needed? |
|---|---|---|
| `iphone-6.9/` | 1320×2868 | **yes** — the only size an iPhone app must supply; App Store Connect scales it down for every smaller iPhone |
| `iphone-6.5/` | 1284×2778 | optional |
| `ipad-13/` | 2064×2752 | only if the app ships for iPad — `build-ios.cjs` sets `TARGETED_DEVICE_FAMILY "1"` (iPhone only), so Connect won't offer an iPad slot |

## Captions

`captions.txt` gives each original a headline and a subhead, drawn above a
scaled-down capture rather than shipping the bare screen:

    01|Bill for your work, get paid|Share an invoice. They pay online by card.

A screenshot with no line there is rendered full-bleed and uncaptioned.
`SKIP_CAPTIONS=1` does that for the whole set.

**The text is rendered by `caption-render.swift`, not by ImageMagick.** The
ImageMagick installed here is built *without freetype* --

    magick -version | grep Delegates    # no "freetype" in the list

-- so `-annotate` and `-draw text` don't render anything. They fall through to
Ghostscript and the run dies with `sh: gs: command not found`, which looks
like a missing dependency and isn't one worth chasing: installing ghostscript
only buys the PostScript base-35 fonts, not SF Pro. CoreText draws the same
system font the app itself draws, so the caption and the screen below it are
set in the same typeface. It needs `swiftc` (Xcode), which this machine
already needs for `build:ios` anyway.

Headlines over about 34 characters wrap to two lines. That lays out correctly,
it just pushes the capture down and shrinks it.

## Things that have bitten us

**TestFlight back-link, and when NOT to mask it.** Screenshots taken straight
after opening the app from TestFlight carry a "◀ TestFlight" return link in
the status bar. It's iOS chrome, not the app, and it tells every viewer the
listing art came from a beta. The script paints it out; `SKIP_TF_MASK=1`
disables that.

The rectangle is the *measured* extent of that text on a 1206×2622 capture
(iPhone 16 Pro) -- glyphs at x 32..232, y 102..140, masked as 24,98 + 216×45.
It was 0,96..268,146 until September 2026, which was guesswork and overshot
the text by roughly 30px per side.

Two things used to need checking by hand before every run. The script now
checks both:

- **A capture with no back-link.** With the link, iOS lays the status bar out
  on *two* lines and the clock sits high. Without it the clock drops back to
  one line, landing squarely inside the mask rect, so masking slices it in
  half. The script measures the height of the bright band in the status bar
  (79px with the link, 36px without) and skips the mask when it's short.
  Counting bright *pixels* does not work here and was the first thing tried:
  the clock alone has plenty, so a no-back-link capture reads as a back-link.
  This app's whole set is that case; see below.
- **App content scrolled under the status bar.** Then the link and the app's
  own pixels *overlap*, and no rectangle removes one without damaging the
  other. Detected by sampling the band just below the zone. The script warns
  and names the file; the fix is a retake, not a coordinate tweak.

A capture that isn't 1206×2622 is left unmasked with a warning, since the
coordinates are per-device.

**The current originals (October 3, 2026) need retaking before they ship.**
They're the right size and the pipeline handles them, but each has something
no processing can fix:

- `02` (New Invoice) was captured scrolled, so the header buttons sit under
  the status bar; the script warns about it. It also shows "No shared profile
  name set yet", which reads as unfinished: set a profile name first.
- `03` (Payouts) is scrolled too, with a line of app text above the clock,
  and it shows the old "You'll receive 91%" wording. Since 0.3.1 the screen
  states Stripe's fee and the 0.9% platform fee instead; retake it on 0.3.1.
- `04` (Shared Profile) says "Synced across ... and Gelder", the app's old
  name, fixed in the app after this capture. Its photo is a captioned meme
  image ("NASA found a nebula..."); a plain photo or the initials avatar is
  safer in a listing.
- `01` and `03` show real people's names on invoices and a $4.20 test
  amount. Use made-up clients and realistic amounts.

**Other iOS overlays.** Autofill suggestion pills and notification banners sit
*over* app content, so they can't be masked cleanly. Retake the screen.

**Test data.** Placeholder content like `bar@foo.com` or a bio of "asdf" reads
as a debug build in a store listing. Worth populating a card properly before
capturing.

**No alpha.** App Store Connect rejects screenshots with transparency. The
script flattens on the way out.
