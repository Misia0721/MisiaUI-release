# Misia UI

**Declarative HTML and CSS user interfaces for Minecraft 1.7.10, driven from Java.**
**面向 Minecraft 1.7.10 的声明式 HTML 与 CSS 界面库，数据与逻辑保留在 Java 中。**

Current version `0.1.0-beta` · 当前版本 `0.1.0-beta`

This repository holds the **public** half of Misia UI: the usage guide, the changelog, screenshots and
the release notes. The engine's source code is not public.

本仓库只包含 Misia UI 的**公开部分**：使用文档、变更日志、截图与发行说明。引擎源代码不公开。

---

## English

A mod describes a screen as markup and a stylesheet, and keeps the data and the behaviour in Java:

```html
<div class="mui-panel">
  <div class="mui-title">${machine.name}</div>
  <button class="mui-button" data-on-click="smelt">Smelt</button>
</div>
```

```java
Mui.screen("screens/machine.html")
    .data(machine)
    .on("smelt", source -> smelt())
    .open();
```

There is no embedded browser and no JavaScript. The markup is parsed, the stylesheet cascaded, the
boxes laid out and the result turned into a list of draw instructions for the game's own renderer,
which is what makes a page behave the same at every GUI scale and on every machine.

| | |
| --- | --- |
| **What it does** | the Modrinth page, and the guide below — layout, text, controls, binding, animation, theming, limitations |
| **What changed** | [CHANGELOG.md](CHANGELOG.md) |
| **Releases** | the [releases page](../../releases) |
| **Questions and bug reports** | the [issue tracker](../../issues) |

**Install.** Minecraft 1.7.10 with Forge (built against 10.13.4.1614); drop the jar into
`.minecraft/mods/`. It is a client-side mod. Press `M` in game to open the bundled example page.

**Not implemented, and reported rather than ignored:** `box-shadow`, `outline`, `transform`, radial
gradients, `float`, grid, `position: sticky`, double-click selection, `<select multiple>`. A page that
asks for one of them gets a line in its own diagnostics and in the game log; the full list is the last
page of the guide.

## 中文

界面用 HTML 描述结构、用 CSS 描述外观，数据与交互逻辑留在 Java 中。没有内嵌浏览器，也没有
JavaScript：标记由模组自身解析、样式表由它层叠、盒子由它排版，最终转成一组绘制指令交给游戏
原有的渲染管线绘制。因此同一份页面在任何 GUI scale 下都排得一样。

| | |
| --- | --- |
| **使用文档** | [docs/guide/](docs/guide/README.md) — 十五篇：标记、已实现的 CSS、布局、组件、表单、绑定、动作、资源、Java 接口、主题、调试、已知限制 |
| **每次改了什么** | [CHANGELOG.md](CHANGELOG.md) |
| **下载** | [releases 页面](../../releases) |
| **提问与反馈** | [issue 区](../../issues) |

**安装**：需要 Minecraft 1.7.10 与 Forge（针对 10.13.4.1614 构建），将 jar 放入 `.minecraft/mods/`。
本模组为客户端模组。安装后在游戏中按 `M` 键可打开内置的示例页面。

**未实现的功能会被明确上报，而不是静默忽略**：`box-shadow`、`outline`、`transform`、径向渐变、
`float`、网格布局、`position: sticky`、双击选词、`<select multiple>`。页面用到其中的任何一项，
都会在页面诊断与游戏日志里留下说明；完整清单是使用文档的最后一篇。

---

## Licence

All rights reserved — see [LICENSE](LICENSE). The source is not open source, and that is separate
from what you may do with the mod: playing it, shipping it inside a modpack (including a published
one), and writing pages against the library are all expressly permitted. Copying, republishing or
shipping a modified build of the library itself needs the author's agreement.

保留所有权利，详见 [LICENSE](LICENSE)。源代码不公开，但这与"你可以怎么用它"是两件事：游玩本模组、
把它收录进整合包（包括公开发布的整合包）、以及依照本库编写页面，均被明确允许；复制、重新发布或发布
修改过的本库构建则需要作者同意。
