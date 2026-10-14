# Why this fork exists

This is a fork of [vercel-labs/native](https://github.com/vercel-labs/native).
Everything it carries is a gap that only shows up off macOS, where there is no
platform text provider and no GPU presenter, so the toolkit's own software
renderer is the whole of what a user sees, and where the engine's own scroll
physics runs because no native scroll driver takes it.

Every patch is filed upstream first and carried here so a release does not wait
on somebody else's queue. When one lands upstream, its patch is dropped on the
next rebase. When all of them have, the fork goes away and the dependency points
back at `vercel-labs/native`.

The current base is `v0.11.0`, on branch `sdk-0.11.0`. One upstream commit from
`main` sits under the patches: [#688](https://github.com/vercel-labs/native/pull/688)
(`13c2bff6`), which bundles Geist Medium and Bold. It is not ours, and it goes
when the next release contains it.

## The tracker

| # | Patch | Upstream | Status on v0.11.0 | Files touched | Drop it by |
| --- | --- | --- | --- | --- | --- |
| 1 | Colour emoji | [#421](https://github.com/vercel-labs/native/issues/421) | carried | `font_ttf.zig`, `reference.zig`, `canvas_font_tests.zig` | deleting everything between the `colour-emoji patch` markers in `font_ttf.zig` and `reference.zig`, then the three tests in `canvas_font_tests.zig` and the `COLR` fixture in `testdata/fonts` (neither carries markers), and Plaza's `hasColorGlyphs`/`colorLayerCount` calls in `src/tests.zig` |
| 2 | An app can fill the sans weight faces | [#423](https://github.com/vercel-labs/native/issues/423) | carried, narrower: upstream [#688](https://github.com/vercel-labs/native/pull/688) bundles the weights, an app still cannot register ids 3 to 6 | `canvas/root.zig`, `runtime/canvas_fonts.zig`, `runtime/core.zig`, `runtime/canvas_frame.zig`, `runtime/ui_app.zig`, `canvas_font_tests.zig`, `tests/ts-core/font_registry_e2e_tests.zig` | deleting everything between the `sans-variant-registration patch` markers (the e2e test is marked; the test in `canvas_font_tests.zig` is not and goes by hand) |
| 3 | Any registered face may lend a glyph | [#421](https://github.com/vercel-labs/native/issues/421) (its own suggestion 2, [reported here](https://github.com/vercel-labs/native/issues/421#issuecomment-5576005547)) | carried | `reference.zig`, `canvas_font_tests.zig` | restoring the `hasColorGlyphs` gate in `referenceGlyphFor` and reverting the Chinese receipt's last assertion |
| 4 | The Linux present path honours the dirty rect | [#432](https://github.com/vercel-labs/native/issues/432) | dropped, fixed upstream by [#434](https://github.com/vercel-labs/native/pull/434) | none | already gone |
| 5 | Scroll input is coalesced to one event a frame | [#440](https://github.com/vercel-labs/native/issues/440) | carried, rebased onto #434 and #470 | `platform/linux/gtk_host.c`, `build.zig` | deleting everything between the `scroll-coalesce patch` markers |
| 6 | A detented wheel notch travels instead of teleporting | [#441](https://github.com/vercel-labs/native/issues/441) | carried, narrower: upstream [#470](https://github.com/vercel-labs/native/pull/470) supplies the detented flag and the no-fling rule, the travel is ours | `canvas/tokens.zig`, `canvas/root.zig`, `runtime/view_widget_scroll.zig`, `runtime/view.zig`, `runtime/canvas_widget_{runtime,display,state}.zig`, `runtime/view_widget_tree.zig`, `runtime/core.zig`, `runtime/gpu_surface_events.zig`, `runtime/canvas_widget_scroll_tests.zig` | deleting everything between the `wheel-ease patch` markers |
| 7 | Fills stop doing per-pixel work that is loop-invariant or absent | [#442](https://github.com/vercel-labs/native/issues/442) | carried | `canvas/reference.zig` | deleting everything between the `opaque-coverage patch` markers |
| 8 | The rasteriser uses the cores it has | [#443](https://github.com/vercel-labs/native/issues/443) | carried, reapplied onto the new `renderPass`, on a persistent worker pool, and skipped for a pass that carries a compiled vector policy or a compiled text measure | `canvas/reference.zig`, `canvas/render_tests.zig`, `runtime/canvas_frame.zig` | deleting everything between the `banded-raster patch` markers, and the two `withSerialMeasure` calls in `runtime/canvas_frame.zig` (unmarked) |

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

Marked with `// BEGIN sans-variant-registration patch`.

`textSpanFontId` maps a span's weight onto reserved ids 3 to 6. An app cannot
register a face under them, because registering below `min_registered_font_id`
is refused, and pointing `typography.font_id` at a registered family instead
loses weight mapping entirely.

Upstream [#688](https://github.com/vercel-labs/native/pull/688) now bundles
Geist Medium and Bold for those ids, which closes the part of #423 about
medium and bold inking as regular. What it does not change is the gate, so an
app that ships its own family still cannot put its own weights under ids 3 to
6. That is all this patch carries now.

| File | What it changes |
| --- | --- |
| `src/primitives/canvas/root.zig` | `isFillableReservedFontId`: the four sans variant ids, and nothing else in the reserved range |
| `src/runtime/canvas_fonts.zig` | The native gate consults it, and the compiled registry policy is given a floor of 0 for those four ids |
| `src/runtime/ui_app.zig` | The refusal message names the ids an app may fill |
| `src/runtime/canvas_font_tests.zig` | One test |
| `tests/ts-core/font_registry_e2e_tests.zig` | One test comparing the compiled and native paths |

**Where the gate lives now.** On v0.10.1 it was one line in `registerCanvasFont`.
On v0.11.0 the compiled registry policy (`font_registry_policy.zig`, from
`packages/core/src/font_registry.ts`) makes the admission decision and Zig
passes it `minimum_id`. The policy keeps its admission order (id 0, floor, size,
duplicate, capacity) and takes the floor as an input, so for the four fillable
ids the runtime passes a floor of 0 for that one request. No TypeScript changes,
so there is no generated or compiled output to keep in sync. The native fallback
path in the same function gets the same exception, and the e2e test registers
ids 3 to 6 and checks ids 1, 2, 7 and 63 stay refused, on both paths, with
identical resulting state.

**Registration order.** Ids 3 to 6 already draw from bundled faces, and the glyph
atlas retains glyphs under those ids once a frame has been planned for
presentation. A fill after that would keep serving the bundled outlines, so
`registerCanvasFont` refuses it with `error.FontRegistrationTooLate`. Register
the faces before the first frame; ids of 64 and above are not affected.

**Capacity.** Fills of ids 3 to 6 take slots from the same registry as every
other face, so four fills use half of `max_registered_canvas_fonts` (8).

Not covered: `src/app_runner/ts_core_main.zig` still requires font asset ids of
64 or more for TypeScript-core apps. This patch is for the Zig `UiApp` path.

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

## Patch 4: dropped

The Linux present path honouring the dirty rect
([#432](https://github.com/vercel-labs/native/issues/432)) was fixed upstream by
[#434](https://github.com/vercel-labs/native/pull/434), which is in v0.11.0.
Nothing else here depended on it: no other patch touches
`native_sdk_gtk_present_gpu_surface_pixels` or the dirty-rect arguments.

## Patch 5: scroll input is coalesced to one event a frame

Marked with `// BEGIN scroll-coalesce patch` in `src/platform/linux/gtk_host.c`,
plus a file-contains check in `build.zig`.
Upstream [#440](https://github.com/vercel-labs/native/issues/440).

**Rebased onto #434 and #470.** Upstream [#470](https://github.com/vercel-labs/native/pull/470)
tells the engine a notch is detented by emitting scroll kind 15 from
`native_sdk_gpu_scroll`. With coalescing, the event that reaches the engine is
the flush, so the flush has to carry that kind. It does so only while every
event folded into it was a detented one (`gpu_scroll_events` and
`gpu_scroll_all_detented` on the view), so a frame that mixed a wheel notch
with a touchpad is not treated as a notch. The `build.zig` check pins both the
kind selection in the flush and the fold that maintains the flag.

`native_sdk_gpu_scroll` emitted on every GTK scroll signal, and an emission is a
synchronous dispatch into the engine: view rebuild, layout, display-list diff,
all of it inside the signal handler. A high-resolution wheel or a touchpad
delivers 100 to 200 events a second, so that work outlasts the gap to the next
event. GDK's input source at `G_PRIORITY_DEFAULT` was therefore ready again the
moment the handler returned, and the main loop never descended to the idle band
where the frame emission lives.

Measured on a real gesture, Ubuntu 26.04 on GTK 4.22, scrolling a feed:

| | |
| --- | --- |
| frozen window | 7.5s out of a 21.3s gesture |
| worst single gap between presented frames | 2116 ms |
| input rate while frozen | 138 events a second |
| CPU while frozen | 49% of one core, on a four core machine |
| timers during the freeze | firing on time, at their normal rate |

Under half a core in use and the timers never missing a beat: the window was not
too busy to draw, it was never asked. That is what makes this a scheduling
defect rather than a rendering one.

The fix is the file's own frame-emission idiom applied to scroll input. At most
one flush in flight, folded producers, fired on the frame-interval grid anchored
at the last emission, exactly as `native_sdk_gpu_surface_schedule_frame_emission`
does. Coalescing is what lets the input source go quiet between events, and a
quiet input source is what lets the idle band run at all.

The deltas SUM, so coalescing changes when the engine hears about a gesture and
never how far it went.

**Why the flush is at `G_PRIORITY_DEFAULT`** and not the idle band the frame
emission uses: this is input, it stands exactly where the per-event emission
stood, and parking it in the band it exists to unblock would be circular. It
cannot spin, because it arms only when a delta is waiting and disarms on the
flush, so between gestures there is no source at all. The idle law the frame
scheduler documents at the top of the file is untouched.

**macOS has had this since before the port.** `queueScrollInputEvent` and
`emitQueuedScrollInputEvent` in `appkit_host.m` are the same accumulate, arm
once, fire on the display grid shape. Linux simply never got it, which is why
the defect has been there since the first Linux build and why nobody working on
macOS would ever see it.

**One thing to watch.** `ScrollAxisState.applyWheel` sets fling velocity from
the delta of the event it is handed, and a coalesced event carries a larger
delta than each of the events it replaces. Velocity is overwritten rather than
accumulated, so a gesture's fling now starts from the last FRAME's delta instead
of the last EVENT's. On macOS this is moot because native scroll drivers own the
physics and `applyWheel` never runs. On Linux it is a real change in the glide,
and an app that tunes `wheel_velocity_scale` should re-check it.

## Patch 6: a detented wheel notch travels instead of teleporting

Marked with `// BEGIN wheel-ease patch` across the runtime and canvas files.
Upstream [#441](https://github.com/vercel-labs/native/issues/441).

`applyWheel` does `next.offset += delta`, which is right for a touchpad and
wrong for a detented wheel. A detent is a fixed distance however slowly it
arrives, so on Linux one notch teleports the feed 40 points in a 690 point
viewport with no intermediate position ever computed. No frame rate fixes that,
because there is nothing in between to draw.

A detented notch now sets a target and a clock, and each frame samples
`start + (target - start) * ease(elapsed / duration)` off the frame event's own
timestamp. Deltas accumulate onto the TARGET, so spinning faster than the ease
completes cannot lose distance. Duration is twice the observed gap between
notches, floored at 50ms, capped at the token, because a fixed duration
accelerates on a fast spin.

**What upstream covers now.** [#470](https://github.com/vercel-labs/native/pull/470)
added `scroll_is_detented` on `GpuSurfaceInputEvent` and `WidgetPointerEvent`,
the `.detented_wheel` scroll source, the GTK and Windows hosts setting it, and
the rule that a detented notch carries no fling velocity. This patch used its
own `wheel_detented` flag and source beside it before; those are gone, and the
travel is built on upstream's names and plumbing. What remains is only the
travel: `wheel_ease_ms`, the ease state, the frame step, and the outcome type.
`wheel_ease_ms` defaults to zero, which is #470's behaviour exactly.

Two of the pieces are behaviour changes to shared paths rather than additions:
the scroll axis returns `{consumed, dirty}` instead of `?RectF`, and
`advanceCanvasWidgetKineticScrollForFrame` no longer skips the whole step when
input is pending.

**Tests.** Two in `canvas_widget_scroll_tests.zig`, and they are the only
executable thing that reaches the travel. `native automate wheel` dispatches a
`.gpu_surface_input` directly and never enters the GTK handler, so the rest of
the battery passes whether this is right or wrong. They set `scroll_is_detented`
themselves and drive the frame clock by hand. Upstream's own test in the same
file covers the instant path with `wheel_ease_ms` at zero.

## Patch 7: fills stop doing per-pixel work that is loop-invariant or absent

Marked with `// BEGIN opaque-coverage patch` in `canvas/reference.zig`.
Upstream [#442](https://github.com/vercel-labs/native/issues/442).

Three reductions, all provably identical rather than approximate.

A fully covered pixel takes `blendPixel` directly: `blendRgba8Coverage` already
routes `cov >= 1` to `blendRgba8`, and scaling alpha by exactly 1.0 is identity
in IEEE754.

A square corner skips `referencePointInRoundedRect` entirely, which reduces to
its own `containsPoint` when every radius is zero, and stops reconstructing
`rect.normalized()` per pixel.

An opaque solid fill becomes four byte writes: the sample is loop-invariant,
and with `src_a == 1` the destination is arithmetically absent from the result,
so the read before the blend buys nothing.

Measured on Ubuntu 26.04 aarch64: `present` p50 29.3ms to 16.0ms, frame
interval 43ms to 28ms. Minimum of three runs; that machine spreads 35%.

## Patch 8: the rasteriser uses the cores it has

Marked with `// BEGIN banded-raster patch` in `canvas/reference.zig`.
Upstream [#443](https://github.com/vercel-labs/native/issues/443).

Rows split into bands, each running the same commands in the same order against
a scissor narrowed to its own rows. `present` p50 16.0ms to 7.1ms, frame
interval 28ms to 18ms, so 36fps to 55fps, with CPU going from half of one core
to about 150% across cores. Those numbers were taken with a thread spawned per
band per frame; the pool below replaces that and has not been re-measured.

Identical output follows from the scissor contract: every primitive derives its
writes from `referenceCommandBounds` then `referencePixelRect`, so a band writes
exactly the pixels the whole pass would have written in those rows and never one
outside. Bands own disjoint rows, and every band is rendered exactly once.

**A persistent pool, not a spawn per frame.** The glyph rasteriser scratch is a
`LazyTls` allocation of about 508 KiB from the page allocator that lives until
process exit. A thread spawned every frame that drew text leaked one each time.
The workers are now created once per process, on first use, sized from a cached
cpu count (at most 7 plus the calling thread), with a 4 MiB stack, so each
worker's scratch is made once and reused. Each worker is handed one band per
frame and the calling thread renders the last band and waits for the rest.

**When the pool is not available** the whole pass renders sequentially on the
calling thread, so nothing is drawn twice and nothing is skipped. That covers a
pool that is busy with another pass, a pool that could not start any worker, and
the test hook `band_pool_enabled`. If only some workers start, the pass is split
into as many bands as there are threads.

Five preconditions, each a hazard: no blur in the pass (it is the only command
that reads anything but the pixel it writes, copying the whole surface into
scratch), no render memo (shared mutable state, and null on every desktop host),
no compiled vector policy, no compiled text measure, and enough rows to be worth
it.

**The vector policy is new on v0.11.0.** `renderPass` now builds an `owned`
surface carrying the pass's `gpu_plan_policy`, and path, stroke and glyph fills
call into it. It is the app's compiled TypeScript core, which has one heap and
one result buffer and is entered from one thread, so a pass that carries one
stays sequential. Colour glyphs now pass the policy too, as monochrome glyphs do.

**The compiled text measure.** Text drawn without shaped glyphs walks per-cluster
advances through the runtime's measure function, which enters the compiled core
whenever the runtime has a `text_cache_policy`. The runtime sets
`serial_measure` on the surface in that case and the pass stays sequential.
Apps with a Zig core leave both unset and band as before.

**How identity was established, which is the part to keep on a rebase.** Almost
every test surface is under 128 rows, so with the real threshold not one test
takes this path and a green suite means nothing. Force
`min_reference_band_rows` to 4, which bands every test surface across 8 threads,
and run the whole suite including golden pixels. Then put it back. A test in
`render_tests.zig` also renders one scene sequentially, banded, and with the pool
disabled, and requires the same bytes.

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

`sdk-0.11.0` carries patches 1, 2, 3, 5, 6, 7 and 8 on top of `v0.11.0` plus
upstream #688. **This is the branch Plaza and Notary consume.**

A branch per SDK release rather than one that tracks `main`, because the
automation protocol moves and the CLI and the framework have to agree on it: a
mismatch does not fail the build, it silently breaks every live test. So an SDK
upgrade is its own change, made deliberately, with the `native` CLI version
bumped in the same commit as the pin.

Earlier bases are kept so an older pin still resolves: `sdk-0.10.1-linux-gaps`
(0.10.1 with patches 1 to 8), `sdk-0.9.2-linux-gaps` (0.9.2 with the first four)
and `sdk-0.9.2-colour-emoji` (0.9.2 with the first two).

## Rebasing onto a new upstream release

```
git fetch upstream --tags
git checkout -b sdk-<version> v<version>
git cherry-pick -x <the patch commits, oldest first>
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
either way, which is exactly the hole this test was added to close.
