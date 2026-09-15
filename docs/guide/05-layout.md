# 05 布局与排版

这一页讲盒子怎么被放到位置上、文字怎么被折断、溢出与滚动怎么工作。写页面时**大多数"位置不对"**
都在这几条规则里，而不是在某个属性没生效上。

## 一、盒模型

盒子从里到外：content → padding → border → margin。`box-sizing` 决定 `width`/`height`
指的是哪一个：

| `box-sizing` | `width: 100px` 指 |
| --- | --- |
| `content-box`（初始值） | 内容区 100px，内外边距与边框另加 |
| `border-box` | 从边框外沿算起 100px，内容区 = 100 − padding − border |

**宽度一律是逻辑像素**，与 GUI scale 无关：GUI scale 变的是"一个逻辑像素占几个物理像素"，
不是布局尺寸。所以同一个页面在 scale 1 和 scale 4 上班次一样、只是清晰度不同。

尺寸计算顺序（对每个盒子）：

1. `width`/`height`（`auto` 时按内容或按容器）；
2. `min-width`/`max-width`/`min-height`/`max-height` 夹一次；
3. `box-sizing` 决定第 1 步的宽度包不包含 padding 与 border。

百分比相对**包含块的 content box** 解析。`height` 的百分比只在包含块有确定高度时才有意义
（浏览器同理），所以 `height: 100%` 在 `height: auto` 的父级里会退化成内容高度。

**外边距不合并**（没有 margin collapsing）。这在浏览器里是常见陷阱，在这里不会发生：
`margin-top: 6px` 就是 6px，永远不会和相邻的兄弟或父级"叠在一起"。

## 二、`display` 与三套布局

| 值 | 布局方式 |
| --- | --- |
| `block` | 块流：独占一行，自上而下 |
| `flex` | 弹性行/列 |
| `inline` | 折进文字里，**没有自己的盒子** |
| `inline-block` | 一行里的一个盒子（原子行内元素） |
| `inline-flex` | 同上，但内部按 flex 排 |
| `none` | 不生成盒子，不占位、不可点 |

### 块流

`block` 子元素依次向下，宽度默认撑满容器，`margin: 0 auto` 可以水平居中。
没有 float、没有表格布局、没有 `position: sticky`（见 [14 已知限制](14-limitations.md)）。

### flex

```css
.mui-row { display: flex; align-items: center; gap: 6px }
```

- 主轴由 `flex-direction` 决定，`justify-content` 排主轴、`align-items` 排交叉轴；
- `flex: 1` = `flex-grow: 1; flex-shrink: 1; flex-basis: 0`（简写按 CSS 的三种写法都能写）；
- `gap` 同时给行距与列距，`row-gap`/`column-gap` 可以分开；
- `flex-wrap: wrap` 才允许换行；换行后每行的交叉轴对齐由 `align-content` 决定；
- flex 容器的子元素一律被"块化"：写 `display: inline-block` 的子元素在这里就是普通 flex 项；
- flex 项的缩小只受**显式** `min-width`/`max-width` 约束，**没有**浏览器那种"自动最小尺寸"
  （不缩到内容以下）：要保住一个项不被压扁，写 `min-width` 或 `flex-shrink: 0`，
  否则窄容器里它会一路缩下去、里面的文字被挤出去。

### 行内与 `inline-block`

一行文字就是一条**行盒**：文字、`<span>` 这类行内元素、以及 `inline-block` 盒子一起被折断、
对齐到基线上。

```html
<div>Turn it <input type="checkbox"> on, or name it <input value="Misia"> instead.</div>
```

- `inline-block` 的宽度：写了就是写的值，没写就是 CSS 的 fit-content（`min(max-content, 可用宽度)`），
  所以徽章和它的词一样宽、控件和样式表说的一样宽；
- 它的高度由内容决定（自己也可以写 `height`），`min-height`/`max-height` 照常夹；
- 它整体是一个**不可折断的单位**：放不下就整块换行，不会被拆成两半；
- 它的基线：里面有文字就用最后一行文字的基线，没有文字（控件、纯背景盒）就用下外边距边——
  这就是复选框坐在句子上而不是浮半行；
- 它占位是**整块外边距盒**，所以内边距和边框不会压到旁边的字。

**没有 `vertical-align`**：行内盒一律坐基线。想抬高/压低一个徽章，只能用 `position: relative`
微调或改变行的构造。

### `display: inline` 的控件排不出来

`<input>`/`<button>`/`<select>`/`<textarea>` 在基础样式表里是 `inline-block`。如果你把它改成
`display: inline`，行内元素会被折进文字、没有盒子可画，引擎会**上报并丢掉这个控件**：

```
<input type="checkbox"> is inline content, and an inline element has no box to be drawn in;
give it `display: inline-block` (or `block`), or put it in a flex row
```

## 三、定位

| `position` | 行为 |
| --- | --- |
| `static`（初始值） | 参与正常流 |
| `relative` | 参与正常流，**但自己**按 `left/top/right/bottom` 偏移；兄弟与父级尺寸不变 |
| `absolute` | **脱离流**，相对最近的定位祖先的 padding box（没有就是视口）定位 |
| `fixed` | 同上，但包含块永远是视口 |

```css
.tag { position: absolute; top: -7px; right: -7px }   /* 挂在面板角上，压着边框 */
```

规则与取舍：

- 同轴上 `left`/`top` 胜过 `right`/`bottom`（CSS 的过约束规则）；两侧都 `auto` 就留在流里的位置；
- `absolute` 的 `auto` 宽度是 **shrink-to-fit**（所以气泡提示才像气泡）；
- 定尺寸 → 布局 → 再定位，这个顺序是必需的：`right`/`bottom` 锚定要知道盒子最终多大；
- 定位盒**画在周围内容之后**，`z-index` 在定位兄弟之间排序；**负 `z-index`** 画在本父级内容之前，
  可以用来做"压在文字下面"的图层；
- 命中测试走的是**同一个绘制顺序的逆序**："最上面那个"就是"最后画的那个"。

`overflow: hidden` 仍然会裁掉跑出去的浮层：引擎只有一条裁剪链，不是每个包含块一条。

## 四、溢出与滚动

```css
.list { height: 72px; overflow-y: auto }
```

- `auto` 与 `scroll` 在 MUI 里是**同一件事**（都画滚动条）：浏览器里它们只差"没溢出时显不显示条"，
  而 MUI 只在需要时画条；
- `hidden` 裁剪且**不滚动**——"把它切掉"不是"给我一条路看到其余"；
- 滚动的是**内容**，盒子的边框与背景不动——这就是滚动的视觉本意；
- 能滚多远由**内容范围**（`scrollHeight`）决定，不是盒子自己的尺寸；
- 子元素"自己的可滚动范围"也算进父级的内容范围，所以在内层列表滚到底后，滚轮会接着滚外层；
- 滚动条占**内侧**：横条在下、竖条在右，两者相遇处互相让位；条本身不占内容空间，
  它只是自己的拖动目标（在条上按下不会传给下面的行）。

滚动条颜色来自两个自定义属性（默认值也在）：

```css
.list { --mui-scrollbar-thumb: #4a4a57; --mui-scrollbar-track: rgba(0, 0, 0, 0.25) }
```

三种手势都支持：拖滑块（抓住的位置会保持，不会跳到指针下）、点轨道（前后翻一屏）、滚轮。
1.7.10 的滚轮只有一个轴，所以"只能横向滚的条"在滚轮下会**改滚另一个轴**——这条例外只在
滚轮那个方向确实没得滚时才生效。

**可滚动盒子必须有 `id`。** 滚动位置记在元素 id 上（绑定会重建整棵树，只有名字能活下来），
没有 id 的盒子滚一帧就跳回去，并且会上报：

```
a scrollable <div> has no id, so its scroll position cannot survive a rebuild of the page; give it one
```

## 五、文字与排版

| 属性 | 说明 |
| --- | --- |
| `font-size` | 逻辑像素；`em` 相对父级字号，`rem` 相对根字号（默认 16，宿主可用 `setRootFontSize` 改） |
| `line-height` | `normal`（字体自身行高）、数字（自身字号的倍数）、长度 |
| `text-align` | `left` `right` `center` `justify` |
| `white-space` | `normal`（折行、合并空白）、`nowrap`、`pre`、`pre-wrap`、`pre-line` |
| `word-break` | `normal`、`break-all`（任何两字符之间都可断，用于路径/URL/哈希）、`keep-all`（关掉 CJK 逐字折断） |
| `letter-spacing` | 每个字符之后都加（最后一个也算），布局与绘制两侧都读它 |
| `text-overflow` | `ellipsis` 需要**配合** `overflow: hidden`；截断发生在对齐之前，用 U+2026 一个字形 |
| `line-clamp` | 整数：只显示前 n 行，其余**根本不布局**（不是裁掉） |
| `text-decoration` | `underline` `overline` `line-through`（可组合），颜色默认跟随 `color` |

换行规则：在**空格**处断词，CJK 之间逐字可断（`word-break: keep-all` 关掉），
标点不落在行首（基本避头尾）；代理对（emoji 等）不会从中间被切开。

行号与列号对控件有实际意义：文本框的光标、点击落点、上下移动都走同一套行模型
（`core.text.TextLines`），所以多行文本里"点哪就是哪"和光标位置不会互相矛盾。

## 六、`visibility` 与 `display: none`

| 写法 | 占位 | 可点 | 子元素 |
| --- | --- | --- | --- |
| `display: none` | 不占 | 不可点 | 整棵子树都不生成盒子 |
| `visibility: hidden` | **占位** | 不可点 | 子元素可以自己写 `visible` 再显示 |
| `opacity: 0` | 占位 | **可点** | 也是 `:hover` 的目标 |

做"淡出但不改变布局"用 `opacity`；做"收起这一段"用 `display: none`（配合 `data-if` 更直接）。

## 七、调试布局

- 离线预览旁边就有盒树：`build/preview/<名字>.layout.txt`，每个盒子的精确几何；
- **几何是父级坐标系里的偏移**（只有根是按视口量的），所以看 dump 时要把祖先的
  content 原点累加起来——这也是"画在错的位置"最常见的来源，`Painter` 做的正是这件事；
- `document.pipeline.layoutTree().boxOf(element)` 可以按元素取盒子，量某一处的数字；
- 布局阶段的诊断（盒子建不出来、`line-clamp` 用在非文字盒上…）都在同一份 dump 与
  `diagnostics()` 里。

## 相关

- 属性逐个说明：[04 CSS 参考](04-css.md)
- 控件在行内怎么表现：[07 表单控件](07-forms.md)
- 位置不对怎么查：[13 调试](13-debugging.md)
