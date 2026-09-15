# Screenshots

Every image here is a **real client capture** — the framebuffer read back after the page was drawn —
not an offline render, so a store page or a wiki entry is showing what a player sees.

They were all taken with the in-game self-test, which opens a page, drives it if asked to, and writes
the framebuffer out once the game has drawn it:

```powershell
gradlew.bat runClient -PmuiSelfTest=build/selftest/shot.png `
  -PmuiSelfTestPage=screens/gallery.html -PmuiSelfTestFrames=3
```

| file | what it shows |
| --- | --- |
| `demo-in-game.png` | the bundled **demo** page — the nine-slice panel with its corner tag, the button pair, the tab strip, the textured strips, the 6,200 / 10,000 FE buffer, the icon list with its scrollbar, and a clamped underlined paragraph |
| `gallery-in-game.png` | the **component sheet** page at rest: buttons in four states, text fields (placeholder, filled, read-only, disabled), a control used inside a sentence, a text area, a select, a checkbox and a radio pair, a slider, two meters, a scrolled list, a tab strip |
| `select-open-in-game.png` | a **`<select>` with its list open** over the page, the highlight moved by the keyboard |
| `text-selection-in-game.png` | **text selected** in a field with Ctrl+A, the highlight drawn per visible line in the element's own colour |
| `textures-in-game.png` | **texture sizing and nine-slice borders**: the same texture at `auto`, `24px`, `contain`, `cover`, `100% 100%` and repeated, and the four `border-image-repeat` modes |

The window is 854×900 at GUI scale 2, which is why the images are 854×900 pixels — the game's
framebuffer, not a scaled-up preview.

If a store page wants a different aspect ratio, the offline preview can render the same pages at any
size, but those are predictions of the engine's output rather than photographs of it; the images above
are the ones that prove something about the real client.
