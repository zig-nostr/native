# Why this fork exists

This is a fork of [vercel-labs/native](https://github.com/vercel-labs/native)
carrying one patch: **colour emoji in the reference renderer**, which is the
renderer Linux and Windows present with.

It exists to unblock a Linux release and is meant to be short-lived. When the
change lands upstream, this fork goes away and the dependency points back at
`vercel-labs/native`.

Upstream issue: [vercel-labs/native#421](https://github.com/vercel-labs/native/issues/421).

## The patch

Three files, and every hunk in the two renderer files sits between
`// BEGIN colour-emoji patch` and `// END colour-emoji patch` markers so a
rebase shows exactly what is mine.

| File | What it adds |
| --- | --- |
| `src/primitives/canvas/font_ttf.zig` | `cmap` format 12 (every codepoint above U+FFFF, which is where emoji live), and `COLR` v0 / `CPAL` reading: `hasColorGlyphs`, `colorLayerCount`, `colorLayer` |
| `src/primitives/canvas/reference.zig` | `referenceGlyphFor` (a registered colour face stands in for a glyph the text face lacks) and `drawColorGlyph` (a colour glyph is a stack of outlines, each with its own palette colour) |
| `src/runtime/canvas_font_tests.zig` | Two tests, plus a four-emoji `COLR` fixture in `testdata/fonts` |

## The two rules it keeps

**A font id still means the face it names.** The fallback is admitted only for
a registered face that carries `COLR`, and only for a codepoint the named face
has no glyph for. A colour face has no monochrome text to lend, so this cannot
make one id draw with another's outlines. `canvas_font_tests` pins that
property from both directions: the Chinese receipt, and a test that a
monochrome face never lends its glyphs.

**Layout and ink still agree.** Measurement never sees the fallback face, so it
bills the codepoint from the text face's estimator. For the BMP symbol range
U+2190..U+2BFF (hearts, dingbats, arrows) that is the 0.8 em pictograph class,
while an emoji face draws a full 1.0 em. A fallback glyph is scaled to the cell
already charged for it, so nothing spills into the next character. The astral
planes are already billed 1.0 em and pass through untouched.

## Branches

`colour-emoji` carries the patch on top of upstream `main`, which is the base an
upstream change would want.

`sdk-0.9.2-colour-emoji` carries the same commit on top of `8d0da34`, the SDK
0.9.2 tree. **This is the branch Plaza consumes.** The automation protocol moved
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
