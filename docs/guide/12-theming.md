# 12 主题

换成自己的样子有三条路，从轻到重：**改自定义属性** → **覆盖组件表的规则** → **整份替换**。
三条都不是特殊机制，就是普通 CSS 与普通资源层。

## 一、自定义属性：先说引擎认识的两个

引擎自己只读**两个**自定义属性，都是滚动条的：

```css
.list {
  --mui-scrollbar-thumb: #4a4a57;              /* 滑块 */
  --mui-scrollbar-track: rgba(0, 0, 0, 0.25);  /* 轨道 */
}
```

为什么只有两个：这套库的规矩是**属性只在有人读它时才存在**。一个"可主题化的滚动条"会想要宽度、
圆角、悬停色、最小滑块长度——而其中没有一个被实现，多出来只会变成猜谜。

自定义属性本身是完整的 CSS 特性（[04 CSS 参考](04-css.md) 第六节）：
**照常继承**，所以写在 `:root` 或 `body` 上就是全局主题：

```css
:root {
  --ink: #e6e6ea;
  --panel: #1e1e24;
  --accent: #4cb862;
  --muted: #7f7f8c;
}
.mui-panel { background-color: var(--panel); color: var(--ink) }
.mui-button-primary { background-color: var(--accent) }
```

一条规则换掉所有控件色调（`color` 也是勾、圆点、滑块、光标的颜色）：

```css
input, select, textarea { color: var(--accent) }
```

## 二、覆盖组件表

组件表（`theme/components.css`）里每条规则都是普通 CSS，覆盖它不需要任何权限：

```html
<link rel="stylesheet" href="../theme/components.css">   <!-- 先 -->
<link rel="stylesheet" href="my-theme.css">              <!-- 后：同优先级后者胜 -->
```

或者用 `<style>` 写在页内（它排在所有 `<link>` 之后）。

| 想改 | 写什么 |
| --- | --- |
| 全部面板底色 | `.mui-panel { background-color: … }` |
| 一个按钮 | `#scrap { background-color: … }`（用 id，优先级更高） |
| 按钮按下的感觉 | `.mui-button { transition-duration: 80ms }` |
| 列表行高亮 | `.mui-list-row:checked { background-color: … }` |
| 滚动条 | `.mui-list { --mui-scrollbar-thumb: … }` |
| 标题字号 | `.mui-title { font-size: 13px }` |

规则：

1. **不要改 jar 里的 `components.css`**（改了也会被下一次构建/更新覆盖）；
2. 需要更高优先级时用 `#id` 或 `!important`，但更该考虑换个类名；
3. **只用引擎真有的属性**：`box-shadow`、`transform`、`background` 简写之类的会被**上报**
   并且不生效（[14 已知限制](14-limitations.md)）。看一眼诊断，别靠猜。

## 三、整份替换：当主题包发

```
assets/misiaui/theme/demo.css         ← 三行差异也行
assets/misiaui/theme/components.css   ← 同名文件：整份接管（因为样式表是按层合并的）
assets/misiaui/textures/panel.png     ← 换贴图
```

因为样式表**按层合并**（[10 资源](10-resources.md)），你可以只写差异，
也可以同名覆盖。玩家用 `F3+T` 或切换资源包立刻看到效果。

给自己的主题留一份"变量入口"是好习惯：

```css
/* my-theme.css —— 唯一需要改的地方 */
:root {
  --sky-ink: #e6e6ea;
  --sky-panel: #14161c;
  --sky-accent: #7fd4ff;
}
```

## 四、GUI scale：为什么"看起来一样、只是更清楚"

Minecraft 的 GUI scale 决定"一个逻辑像素占几个物理像素"。MUI 的规则：

- **布局尺寸一律是逻辑像素**，所以拖动 GUI scale 滑块**不会**让页面重排；
- **字形图集按 GUI scale 用更高的光栅化倍率**生成（`AwtGlyphSource(pageSize, rasterScale)`），
  所以 scale 2 下的文字是"重新按 2 倍描出来"的，而不是把 1 倍的位图放大；
- 打开屏幕时用 `ScaledResolution` 读**游戏自己的** scale（不猜），所以图集密度与最终画出来的
  密度一致；
- 变化时重建画笔而不是留着旧图集（`GlyphSourceFactory.forGuiScale`）。

预览可以模拟放大：`gradlew muiPreview -PmuiPreviewScale=3`（默认 2）。

## 五、根字号与 rem

`rem` 相对**根字号**，默认 16。宿主可以改：

```java
pipeline.setRootFontSize(20f);   // 整页 rem 一起放大
```

给页面留一条"整体缩放"的开关时，这是唯一一条不需要逐条改数字的路：

```css
.mui-panel { padding: 0.5rem; font-size: 0.75rem }
```

`em` 相对**父级字号**，用于"跟着上下文缩放"的部件（徽章比标题小一号的情形）。

## 六、深色 / 浅色两套皮

引擎没有"配色方案"概念，也不需要：一套主题就是一份样式表，
`@media` 可以让它按视口尺寸变形（`orientation`、宽高阈值——但**没有** `prefers-color-scheme`）。
玩家想换皮就换资源包，这是 Minecraft 里所有人已经会做的事。

如果两套皮要同时存在，把颜色集中在自定义属性上，两份文件只改那几个变量：

```
assets/misiaui/theme/palette-dark.css
assets/misiaui/theme/palette-light.css
```

## 七、字体

```css
body { font-family: sans-serif; font-size: 12px }
.mono { font-family: monospace; font-size: 10px }
```

- **通用字族名**会被映射到 JDK 的逻辑字体（跟着系统配置走，中英都覆盖得到）：
  `sans-serif`（`sansserif`、`system-ui`、`default` 同义）、`serif`、`monospace`（`mono`）、
  `dialog`；
- 其它名字**原样传给系统**：写具体字体名只保证在装了它的机器上有效，所以优先用通用名；
- 找不到的字族**不会报错**，会退到系统默认字体（"一个未知字族仍然量得出宽度"由测试钉住）；
- 一个 `font-family` 列表**只用第一个**（没有回退链，见限制）；
- 字号影响布局与字形光栅化两侧，所以"改字号"不需要同步改别的东西；
- CJK：字体的真实 advance 就是这么宽，一行里混排中英会按字体给的宽度排，不需要专门处理。

## 八、做一个自己的组件表

组件表是一个**可以整个换掉的约定**，不是引擎的一部分。想从零开始：

```html
<link rel="stylesheet" href="my-ui.css">
```

```css
/* my-ui.css —— 从零开始的一套，只依赖引擎真有的属性 */
.panel { display: flex; flex-direction: column; padding: 8px; background-color: #202028;
         border-radius: 4px }
.panel > .title { font-size: 13px; color: #f0f0f5; margin-bottom: 4px }
.row { display: flex; align-items: center; gap: 6px }
.row > .label { color: #b8b8c4 }
.action { padding: 3px 9px; border-radius: 3px; background-color: #3b3b46; color: #e6e6ea;
          transition-property: background-color; transition-duration: 120ms }
.action:hover { background-color: #4a4a57 }
.action:focus-visible { background-color: #5a5a6a }
.action:disabled { opacity: 0.4 }
```

要记住的三件事：

1. **状态用真属性**（`:checked`、`:disabled`、`:hover`、`:focus-visible`），
   不要另造一套状态类——引擎已经按属性回答了这些问题；
2. **能用引擎画的东西就让引擎画**：勾、圆点、滑块、光标、下拉箭头都是引擎的，
   页面改它们的颜色（`color`）而不是重画它们；
3. **每加一条规则就顺手看一眼诊断**：写错属性名会被上报，这是这套库最省时间的一条习惯。

## 相关

- 每个属性的状态：[04 CSS 参考](04-css.md)
- 组件表里有什么：[06 组件](06-components.md)
- 资源层与热重载：[10 资源](10-resources.md)
