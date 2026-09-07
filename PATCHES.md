# Why this fork exists

This is a fork of [vercel-labs/native](https://github.com/vercel-labs/native).
Everything it carries is a gap that only shows up off macOS, where there is no
platform text provider and no GPU presenter, so the toolkit's own software
renderer is the whole of what a user sees.

Every patch is filed upstream first and carried here so a release does not wait
on somebody else's queue. When one lands upstream, its patch is dropped on the
next rebase. When all of them have, the fork goes away and the dependency points
back at `vercel-labs/native`.

## The tracker

| # | Patch | Upstream | Files touched | Drop it by |
| --- | --- | --- | --- | --- |
| 1 | Colour emoji | [#421](https://github.com/vercel-labs/native/issues/421) | `font_ttf.zig`, `reference.zig`, `canvas_font_tests.zig` | deleting everything between the `colour-emoji patch` markers, and Plaza's `hasColorGlyphs`/`colorLayerCount` calls in `src/tests.zig` |
| 2 | An app can fill the sans weight faces | [#423](https://github.com/vercel-labs/native/issues/423) | `canvas/root.zig`, `canvas_fonts.zig`, `ui_app.zig`, `canvas_font_tests.zig` | deleting the `sans-variant-registration patch` markers |
| 3 | Any registered face may lend a glyph | [#421](https://github.com/vercel-labs/native/issues/421) (its own suggestion 2, [reported here](https://github.com/vercel-labs/native/issues/421#issuecomment-5576005547)) | `reference.zig`, `canvas_font_tests.zig` | restoring the `hasColorGlyphs` gate in `referenceGlyphFor` and reverting the Chinese receipt's last assertion |
| 4 | The Linux present path honours the dirty rect | [#432](https://github.com/vercel-labs/native/issues/432) | `platform/linux/gtk_host.c` | deleting everything between the `dirty-rect patch` markers |

**Checking whether a patch can go:** open the issue. If it is closed and the fix
is in a release, rebase onto that release, drop the patch, run `zig build test`,
and take the row out of this table. Nothing automated watches these; the list is
short enough to read.

**What each patch costs to carry** is below, patch by patch, so a rebase that
conflicts can be judged rather than guessed at.

## Patch 1: colour emoji

Every hunk in the two renderer files sits between `// BEGIN colour-emoji patch`
and `// END colour-emoji patch` markers so a rebase shows exactly what is mine.

| File | What it adds |
| --- | --- |
| `src/primitives/canvas/font_ttf.zig` | `cmap` format 12 (every codepoint above U+FFFF, which is where emoji live), and `COLR` v0 / `CPAL` reading: `hasColorGlyphs`, `colorLayerCount`, `colorLayer` |
| `src/primitives/canvas/reference.zig` | `referenceGlyphFor` (a registered colour face stands in for a glyph the text face lacks) and `drawColorGlyph` (a colour glyph is a stack of outlines, each with its own palette colour) |
| `src/runtime/canvas_font_tests.zig` | Three tests, plus a four-emoji `COLR` fixture in `testdata/fonts` |

## Patch 2: an app can fill the sans weight faces

Marked with `// BEGIN sans-variant-registration patch`. Four lines of real
change.

`textSpanFontId` maps a span's weight onto reserved ids 3 to 6, and the toolkit
bundles no face for any of them, so off macOS `referenceFaceForFontId` resolves
all four back to the regular face: medium and bold ink regular outlines. An app
cannot fix that, because registering below `min_registered_font_id` is refused,
and pointing `typography.font_id` at a registered family instead loses weight
mapping entirely.

| File | What it changes |
| --- | --- |
| `src/primitives/canvas/root.zig` | `isFillableReservedFontId` — the four sans variant ids, and nothing else in the reserved range |
| `src/runtime/canvas_fonts.zig` | The registration gate consults it |
| `src/runtime/ui_app.zig` | The refusal message names the ids an app may fill |
| `src/runtime/canvas_font_tests.zig` | One test |

This is upstream's own option 2 of the three in #423, and the cheapest: it adds
no bundled faces, needs no renderer change, and changes no pinned behaviour.
The other two options both rewrite the markdown reference golden.

Registering a real face also fixes something the issue does not mention. Today
the variant ids take a flat 1.02/1.04 width factor over the regular face's
advances while inking regular outlines centred in those wider cells, which is
spacing pretending to be weight. A registered face routes measurement through
`estimateTextWidthForFace`, so ink and layout agree exactly instead of
approximately.

## Patch 3: any registered face may lend a glyph

Marked by the same `colour-emoji patch` markers, because it is one line inside
the loop that patch added: `referenceGlyphFor` no longer skips a face that has
no `COLR` table.

Patch 1 admitted only colour faces, on the reasoning that a colour face has no
monochrome text to lend, so it could not make one font id draw with another's
outlines. That reasoning is sound and it is also the whole problem. Off macOS
the bundled faces are the only glyph source and they carry Latin and Cyrillic
(Greek is four maths symbols, not the alphabet). Greek, CJK, Japanese, Korean,
Arabic, Hebrew, Thai and Devanagari have no glyph anywhere and are painted as
solid filled rectangles. An app cannot fix that from outside, because a face it
registers is only ever consulted for its own id.

So the gate is coverage rather than colour. The named face is asked first and
still wins every codepoint it carries; a registered face is consulted only for a
codepoint nothing else could draw at all. What it costs is the property patch 1
was careful to keep: one id can now draw one codepoint with another face's
outlines. That is the point of it, and the alternative at that codepoint is not
the right face, it is a filled rectangle.

The Chinese receipt used to assert that the bundled id and the registered id
draw DIFFERENT pixels, because the bundled one could only draw boxes. It now
asserts the fallback instead: under the bundled id, the covered string and a
string no registered face carries must differ. Restore the `hasColorGlyphs` gate
and both become boxes and that fails, which is the probe.

Not fixed by this, and worth saying before anyone treats a font as the whole
answer: `Face.parse` reads no GSUB or GPOS and there is no bidi, so Arabic still
comes out unjoined and left to right, and Devanagari and Thai unreordered.
Coverage is one of the two problems.

## Patch 4: the Linux present path honours the dirty rect

Marked with `// BEGIN dirty-rect patch` in `src/platform/linux/gtk_host.c`.

`native_sdk_gtk_present_gpu_surface_pixels` opened by discarding the five dirty
arguments the runtime had just computed, then converted the whole surface from
straight RGBA8 to premultiplied ARGB32 one pixel at a time. At 1400x900 that is
1.26 million pixels per frame, almost all of them recomputing bytes that already
held the right value. The incoming buffer is always the full surface and the
reference renderer only re-inks its scissor rect, so everything outside the
dirty rect is already correct in both buffers.

It is upstream's own Windows implementation, one platform over: same normalise,
same scale to pixels, same outward floor and ceil, same clamp. A freshly
allocated buffer converts in full whatever the rect says.

The REPAINT is still whole-widget and cannot be otherwise. GTK 4 removed partial
invalidation, so `gtk_widget_queue_draw_area` is a GTK 3 API and there is no way
to ask for less than the widget. Only the conversion narrows.

## The rule patch 1 keeps

**Layout and ink still agree.** Measurement never sees the fallback face, so it
bills the codepoint from the text face's estimator. For the BMP symbol range
U+2190..U+2BFF (hearts, dingbats, arrows) that is the 0.8 em pictograph class,
while an emoji face draws a full 1.0 em. A fallback glyph is scaled to the cell
already charged for it, so nothing spills into the next character. The astral
planes are already billed 1.0 em and pass through untouched.

**What patch 3 gives up.** The other rule patch 1 kept, that a font id always
means the face it names, is deliberately gone. See patch 3.

## Branches

`colour-emoji` carries the patch on top of upstream `main`, which is the base an
upstream change would want.

`sdk-0.9.2-linux-gaps` carries all four on top of `8d0da34`, the SDK 0.9.2
tree. **This is the branch Plaza consumes.** (`sdk-0.9.2-colour-emoji` is what it
consumed before patches 3 and 4, kept so an older pin still resolves.) The automation protocol moved
between 0.9.2 and `main`, and the CLI and the framework have to agree on it, so
taking `main` would mean bumping the CLI in the same breath. Keeping the patch on
the pinned tree keeps the emoji fix separable from an SDK upgrade.

## Rebasing onto a new upstream release

```
git fetch upstream
git rebase upstream/main
zig build test
```

The tests are the check that matters, and there is one per rule.

`the emoji receipt` covers the colour rule: it renders through the public path
and counts saturated pixels, which a monochrome renderer cannot produce however
many outlines it inks.

`a fallback glyph is inked inside the cell layout charged for it` covers the
fitting rule, and it exists because the receipt does NOT. An emoji in the wrong
place is still a coloured emoji, so the receipt stays green with the fitting
removed; the fitting test measures the inked width against the cell the
estimator charged and fails at 14.0px into an 11.2px cell. Both are probed:
neutering either half fails its own test and only its own test.

Note which codepoint that test uses and why. U+2764 is in the BMP symbol range,
the only class where the estimator and an emoji face disagree. An astral emoji
cannot detect the fitting's absence, so a test written with one would pass
either way — which is exactly the hole this test was added to close.
