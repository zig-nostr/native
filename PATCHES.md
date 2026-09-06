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
bills an emoji as an uncovered pictograph (0.8 em). A fallback glyph is scaled
to the cell already charged for it rather than inked at its natural 1.0 em, so
nothing spills into the next character.

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

The tests are the check that matters. `the emoji receipt` fails if either half
of the patch stops working: it renders emoji through the public path and counts
saturated pixels, which a monochrome renderer cannot produce however many
outlines it inks.
