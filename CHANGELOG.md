# Changelog

Misia UI's first published version is `0.1.0-beta`. The version lives in `gradle.properties` as
`modVersion`, which stamps the jar and `mcmod.info`, and `MisiaUI.VERSION` is checked against it by
the test suite so the two cannot drift apart.

Everything under `0.1.0-beta` below arrived before that release: it is the first public version, so
the list is long, and later versions get their own dated heading and start from empty.

The README carries the feature list and the reasoning; this carries the order things arrived in, so
that somebody picking the project up can see what is new and what has been sitting still.

`0.2.0-beta` is the next cut and is still being written: its heading is undated and takes its date when
the version is, which is also when `modVersion` moves. Until then the jar, `mcmod.info` and the version
the test suite checks `MisiaUI.VERSION` against all remain `0.1.0-beta`.

## 0.2.0-beta — in progress, no date yet

Work that lands before the next cut is filed here as it happens, rather than swept up at the end when
nobody remembers the order or the reason. Nothing in this section is in a published jar yet.

### The mod declaration

- **A server that has the jar no longer requires it of the clients that connect.** 1.7.10's FML has no
  client-only switch on `@Mod`; what it reads is the remote-version range, whose default refuses a
  connection when the other side does not have the mod at all. Which connections that costs depends on
  which side is checking: the client checks the server's list with `Side.SERVER`, so a client-side mod
  is already welcome on a server that lacks it, while the server checks the client's list with
  `Side.CLIENT`, so a server that happens to have the jar refuses every client that does not — which is
  what the ordinary mistake of copying a client instance onto a server used to cost the players who
  never needed the mod. `acceptableRemoteVersions = "*"` routes FML to its own `IgnoredChecker`
  instead, and two `ArchitectureGuardTest` guards pin both the range and the entry point's side check.
  Reading FML's sources rather than remembering them is what settled it, and where to read them is in
  `docs/maintainers/stack.md`.

## 0.1.0-beta — 2026-09-16

The first published version: declarative HTML/CSS interfaces for Minecraft 1.7.10, driven from Java,
verified by 987 headless tests, an offline preview and in-game captures. The user-facing explanation
is `docs/guide/`.

### The engine

- Tolerant HTML parser, DOM, and a CSS parser with selectors, specificity, `@media`, `@import`,
  custom properties and shorthand expansion.
- Computed style: inheritance, `inherit`/`initial`/`unset`, and a cascade that resolves `var()`.
- Box model, block layout, inline layout with CJK line breaking, and flex layout on both axes —
  wrapping, grow/shrink, alignment and gaps.
- Text measurement and glyph rasterisation through the JDK font engine, behind `spi.TextMeasurer`.
- `position: relative`/`absolute`/`fixed`, containing blocks, `z-index` and stacking order.
- Paint list: backgrounds, borders, rounded corners as tessellated meshes, text, and clipping.
- `text-decoration` with geometry taken from the font, and the half-leading baseline correction.
- `text-overflow: ellipsis`, and later `line-clamp` for a block that is too tall. The lines past the
  clamp are not laid out at all, so a clamped box is as tall as what it shows.
- Scrolling: `overflow` on both axes, the content extent, wheel input, position memory by element id,
  and a painted scrollbar per axis with a draggable thumb and track paging.
- `background-image` as a `linear-gradient()` or a `url()` texture, with `background-size`,
  `-repeat` and `-position` resolved as one layer.
- A texture-metrics seam: `spi.ImageMeasurer` answers "how large is this texture" from the file's PNG
  header, cached per resource revision. That is what turned `background-size: auto`/`contain`/`cover`
  and `border-image-repeat` from reports into features.
- Nine-slice borders: `border-image-source`/`-slice`/`-width` with `stretch`, `repeat`, `round` and
  `space`, cut against the texture's own pixel size.
- Form controls: `<input type="text|checkbox|radio|range">`, operated by the engine — clicking,
  typing, dragging, the caret, radio exclusivity by `name`, and keyboard reach with no `tabindex`
  anywhere. The state lives in the document (`checked`, `value`), so a page, a binding and `:checked`
  all read the same fact the player changed, and an edit is remembered by element id so it survives a
  re-bind. The chrome is ordinary CSS and only the indicator — tick, dot, range fill, caret — is the
  engine's, drawn in the element's `color`.
- The cross-axis `min-height`/`min-width` of a flex item is applied at last. A row item's height came
  from its content and the bound was never consulted, so a field with a `min-height` inside a flex row
  was as tall as its padding with its text spilling over the border; the main axis had been clamped
  from the beginning, which is what kept the gap out of sight.
- `inline-block` (and `inline-flex`): a box that sits inside a line of text as one unbreakable unit,
  with a real box behind it — its own width (declared, or shrink-to-fit as `min(max-content, available)`),
  its own laid-out contents, and its baseline on the line's. This retires the workaround the base
  stylesheet had carried from the beginning: controls were block-level because an inline element is
  flattened into its text, so a control written inside a sentence had nowhere to be drawn and was
  reported and dropped. `<div>Turn it <input type="checkbox"> on</div>` now lays out the way it reads.
- An inline-level child of a flex container is a flex item, as CSS says. It used to be collected as
  inline content and then discarded when the container wrapped that content in an anonymous item, so a
  `<span>` inside a flex row was simply not there — the bundled pages had been writing `display: block`
  on every child, which is why nobody had noticed.
- The intrinsic width of a flex **row** is the sum of its items, not the widest of them. A row inside
  a row therefore came out slightly too narrow, and what a page saw was a label that wrapped for no
  visible reason. Found by the component gallery rather than by a test — a nested row is the most
  ordinary thing in a user interface, and the fix is covered by a test that fails without it.
- Transitions: `transition-property`/`-duration`/`-timing-function`/`-delay` and the shorthand, eased
  with `linear`, `ease`, `ease-in`/`-out`/`-in-out`, `cubic-bezier()` and `steps()`. Not an animation
  engine: each frame interpolates between the value the cascade produced last frame and the value it
  produced this frame, so a running transition can never leave a stale value behind and a page with no
  transitions declared pays one string lookup. Colours, numbers and pixel lengths interpolate;
  percentages, `em`, keywords and `auto` step, and say so.
- `@keyframes` and `animation`: a timeline the page writes rather than a pair of values the cascade
  produced, which is the whole difference from a transition. `animation-name`, `-duration`,
  `-timing-function`, `-delay`, `-iteration-count` (a number or `infinite`), `-direction` (`normal`,
  `reverse`, `alternate`, `alternate-reverse`), `-fill-mode` and `-play-state` (`paused` freezes the
  progress and resuming carries on from it), plus the shorthand, split into the longhands by type the way
  `transition` already is. **Each property gets its own track through the frames**, so a rule that
  mentions `opacity` at 0% and 100% and `width` at 50% and 100% interpolates the two independently rather
  than jumping one of them whenever a neighbouring frame happens to name it. An animation's values are
  laid over the cascade (and over any transition) before layout, so an animated `width` is the width the
  box is laid out at. What cannot be interpolated is reported with the property and the pair, once each,
  exactly as transitions already do — the two features share one rule about it rather than one each.
- Glyph positions are rounded to whole **device** pixels, in both renderers, through one rule
  (`core.paint.PixelGrid`). A glyph tile is a rasterised bitmap and the game samples the atlas without
  filtering, so a fractional origin loses the tile's outermost texel column: text came out with stems a
  pixel thin here and there, which reads as a font fault and is a placement fault. Font metrics make
  every position fractional — the baseline sits at 27.26 logical pixels, an advance is 13.328125 — so
  the rounding belongs in one place and both the offline preview and the client now go through it.
- Text is placed by **one snap per run, not one per glyph**: the pen and the baseline are rounded once
  and every glyph is placed at its own bearing from that number, and a tile now carries its **baseline at
  a known texel row** so the bearing it reports is a whole number of texels. A rasteriser can only put ink
  on whole texels, so a tile holding just the ink and reporting its outline's fractional bounds says
  something its bitmap cannot be — two letters whose ink starts on the same texel row report bearings that
  differ by a fraction, and rounding each of them separately to the grid lands them a whole pixel apart.
  Glyphs are also rasterised through Java2D's **text** pipeline with fractional metrics off rather than by
  filling their outline, which grid-fits them: 84% of a glyph's ink pixels were partially covered before
  and 42% are now, with stems solid from edge to edge. Measurement keeps fractional metrics, because
  layout must measure with the font's real advances. `GlyphPlacementTest` reads the ink rows out of the
  atlas bitmap rather than out of the bearings, which is what a player actually sees.

- `<textarea>`: the multi-line text control, as a tag rather than a type so that everything asking "is
  this a text control" still asks one question. Its value is its content — a string with newlines in it
  has no business inside an attribute — the leading newline of the markup is dropped and `\r\n` becomes
  `\n`; its characters are the value rather than content, so the box generator leaves the subtree alone
  instead of drawing every line twice; Enter inserts a newline *before* the activation keys, so a page
  with a Save action is not fired by a comment box. The caret became a row and a column, and the painter,
  the Up/Down movement and the click that turns a point back into an index all take their answer from one
  place (`core.text.TextLines`, pure arithmetic on a string). A click picks a line and then a character,
  through the same scroll offset the painter used. Text taller than the box is clipped and scrolled to
  the caret, and the offset is computed rather than stored, so a field nobody is typing in shows its top
  and no scroll position can survive a re-bind to disagree with the caret. Not implemented and reported:
  soft wrap (`wrap`), `rows`/`cols`, and the browser's remembered Up/Down column.
- `<select>`: one of a set of options, and the first component whose interaction is a box that is not
  where the layout would put it. The list is an **ordinary element the engine writes into the page's own
  document** — so the cascade, the box model, stacking, hit testing, clipping, the wheel and the resource
  loader all apply to it unchanged, and the engine's only new obligation is to recognise its own node and
  take it back out again (`data-mui-list`). The rows carry `checked` like a checkbox, so `:checked` is the
  whole of the highlight and a stylesheet marks the row the keyboard is on. It opens below its control and
  never over it, at the control's border-box width (`box-sizing: border-box`, because a plain width would
  draw it six pixels wider), with rows that do not wrap and a sideways scroll for a long option. Position
  is the one thing that needs two passes: the list is placed from the previous frame's geometry and the
  pipeline then checks, after laying out, whether the control's box still ends where the list starts. The
  keyboard opens with the arrows or Enter, moves and wraps with the arrows, jumps with Home/End, takes
  with Enter and gives up with Escape; Tab closes it. Taking an option writes the select's `value` and
  moves `selected` onto the option, and fires the control's own action, because there is no `change` event
  here. Not implemented and reported: `multiple`, `<optgroup>`, `size`, type-to-find, and a flip when
  there is no room below.

### Interaction

- Text selection in a field: a range made by dragging, by Shift with the arrow keys, or by Ctrl+A, and
  drawn as a highlight under the text. A caret is an index and a selection is two — an anchor and the
  moving end — and every rule that matters follows from that: an unshifted move collapses the range, a
  shifted one keeps the anchor, and typing, Backspace and Delete replace what is between them. A press in
  a field places the caret *on the press* rather than on the release, because a caret that only arrived
  on release could not be dragged, and a drag is half of what selecting is. Shift turns a click into an
  extension of what is already selected, and a drag selects whatever it crosses, backwards as easily as
  forwards. The highlight is one rectangle per line — a range that crosses a line break is not a
  rectangle — drawn in the element's own colour at 28% of its alpha, which is the property that already
  stands in for `accent-color` here. The selection belongs to the field the player is in: leaving a field
  drops the range rather than leaving a highlight in a field nobody is typing in. Ctrl+Home and Ctrl+End
  are the two ends of a *value* in a text area, where Home and End are the two ends of a line.
- A published surface: `api.Mui.screen(path).data(model).on(name, handler).open()`, with `Mui.open`,
  `Mui.current` and `Mui.close`. It decides the layers, the atlas, the GUI scale and the interaction
  state for the caller, and it re-binds the model after each action it registered — the moment a model
  almost always moves — so two lines every MUI page used to need are gone. `MuiScreens.openDefault`
  was rewritten on it, which is the check that it covers the real case.
- An action that throws is now caught where every action runs, and reported in the page's own
  diagnostics with the exception's class and message. A handler is a caller's code inside the render
  loop, and it should be the only thing that loses.
- `theme/components.css`, the component sheet, and `screens/gallery.html`, which shows every class in
  the states it has — including a modal dialog that needs no Java: a checkbox the engine operates is
  the state a sibling selector reads.
- A `data-roving` group owns its selection. Clicking an item (or a child of it) writes a `checked`
  attribute on that item and clears the group's others, and an arrow key moves *and* chooses, so a tab
  strip and a list are components rather than markup a page has to wire up action by action. The state
  is the document's, which is what makes `.tab:checked` the whole of the styling and what lets an id
  carry a selection across a re-bind. `data-select="multiple"` is the other kind of group: rows that
  can each be ticked, where the arrows only move.

- Hit testing that walks the layout tree in reverse paint order, accumulating the same clips the
  painter applies.
- `:hover`, `:active`, `:focus`, `:focus-visible` and `:checked`, with `:hover` and `:active` reaching
  an element's ancestors.
- Named actions: `data-on-click` and `data-on-key`, with an unregistered action reported rather than
  swallowed.
- Tab focus through the reachable controls, re-attached by id across a data re-bind.
- Composite controls: `data-roving` makes a container a single tab stop with arrow-key navigation
  inside it — the roving tabindex pattern — and `:focus-within` lets a stylesheet style the container
  of whatever has focus.

### Data and resources

- `${path}` placeholders with formatters, `data-repeat`, `data-if`, and a Java view model that needs
  no registration.
- Three resource layers — a directory on disk, resource packs, the mod's jar — with hot reload driven
  by one revision number and no watcher thread.

### Platform

- A Tessellator/GL11 renderer, glyph atlas upload, and a `GuiScreen` host, all written against the
  decompiled 1.7.10 sources rather than from memory.
- An in-game self-test that opens a page at the title screen, drives pointer, wheel, drag and key
  input, then reads the framebuffer back and documents the paint list beside it.

### Verification

- A per-frame profile: **parse, cascade, layout and paint** timed separately, always on and cheap
  enough to leave on — 334ns a frame against 23ns for the same loop with the timings off, measured on
  the profiler alone rather than inside a frame. It keeps a ring of the last 120 rebuilds and reports
  the last, the average, the worst and the 95th percentile of each stage, plus the frame total and the
  part of it no stage claimed. Frames that found nothing to do are counted separately, because mixing a
  0.1µs frame into the same average as a 600µs rebuild produces a number that describes neither.
- The numbers, on the bundled demo page (238 paint instructions, a gradient, a nine-slice border,
  texture tiles, a scroll region and a live list): a rebuild is **631µs** in the offline preview at
  320×510 and **573µs** in the client at 427×450 logical, and roughly two thirds of it is cascade and
  layout rather than paint. A still frame is **0.1µs** offline and **2.9µs** in the client — the rebuild
  rule, in one measurement. The first frame of a cold session is **108ms**, which is the JVM rather than
  the engine, and worth knowing rather than discovering as a hitch at startup.
- `-PmuiSelfTestFrames=<n>` draws the page for n frames before capturing, and
  `-PmuiSelfTestRebuild=true` forces a rebuild on each of them, so an in-game screenshot run is also a
  measurement. The profile is written into the diagnostics beside the image and into the log.
- A dependency-free headless harness — 883 tests, no JUnit — and an architecture guard that fails when
  platform types reach `core`/`spi`, or when a declared CSS property is read by nothing. A third guard
  compares the version in `gradle.properties` with the one the code logs, because a fact written down
  twice is a fact that will disagree with itself.
- An offline preview that renders the same paint list with Java2D, so a page can be looked at without
  launching the game. It drives the injected clock, which is what lets one hovered panel be photographed
  at four instants of the same transition.
- `-PmuiSelfTestClockStep=<ms>` pins the game's pipeline clock for the captured frame, so a transition
  photographed in a real client is a measurement rather than a coincidence: 100ms into the demo's
  `200ms linear` button fade the framebuffer holds `R70 G171 B90`, against an exact midpoint of
  `R69.5 G171 B90`. The same pin catches a field's caret in its visible half.
- `-PmuiSelfTestType=<text>` types into whatever the pointer focused, one character at a time through
  the screen's key entry point. The run that photographs the settings page logs `5 of 5 characters
  were taken` and the field's own `value` attribute in the same line, so the picture and the document
  are checked together.
- A second preview pair, `forms-rest.png` and `forms-touched.png`, for the same reason as the
  transition frames: everything a control does when it is touched is one instruction in a paint list
  and one shape in a picture.
- Three faults found by that loop and fixed: a glyph page uploaded before the atlas had any ink, a
  rounded rectangle wound the wrong way so face culling discarded every panel, and an image bound
  before its pass was opened, which drew a text run with the icon's texture.
- The preview and the client are compared against each other rather than only against expectations: the
  same page's label pixels are read out of a `build/preview/` image and out of a captured framebuffer,
  and glyph snapping is the first increment where the two match to within one or two edge pixels. Where
  they disagreed — the preview placing glyphs at true subpixel positions, the game sampling on the grid
  — the game was right and the preview was changed to match it.

### Toolchain

- ForgeGradle 1.2-1.1.1 on Gradle 7.4, Java 8 only, JavaFX-free and with mixins disabled.
- Wrapper scripts, `.gitattributes`, and a build that runs offline.
- Every dependency the mod skeleton carried and nothing used — Kotlin, Scala, AJSCore,
  MysteriumLib, the Pineapple Club libraries, NEI — removed, along with the three Maven repositories
  that existed only to serve them. The build resolves Forge and nothing else.
- The GitHub Actions workflow was removed: the project is closed source and the pipeline that matters
  runs where the game runs. `gradlew build muiTest muiPreview` is the whole check, and a workflow file
  that had never run on a runner was one more thing that looked like proof.
- `CHANGELOG.md` itself, and a `Getting started` section in the README: from an empty mod project to a
  screen that answers clicks, in six steps.
- A second worked example, `screens/settings.html`: a form with a roving group of options, a disabled
  section, a clamped paragraph and actions — the page to copy, where the demo is the page to look at.
  `-PmuiSelfTestPage=<path>` photographs any page in a real client, which is what made verifying it
  possible without editing `MuiResources.DEFAULT_PAGE`.
