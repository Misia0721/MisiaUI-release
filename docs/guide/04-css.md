# 04 CSS 参考

MUI 实现 CSS 的一个**子集**，边界画得很清楚：**实现的**、**上报的**、**声明了但做不了的**，
只有这三种状态，没有第四种"安静地什么都不做"。

- 属性名引擎认识、也有人读它 → 生效（下表"实现"）。
- 属性名引擎不认识（`box-shadow`、`transform`…）→ 在页面诊断里报
  `unsupported property 'x'; MUI does not implement it, so this declaration has no effect`。
- 属性名认识、但值读不懂（`display: table`、`width: 12vw`…）→ 进诊断，样式退回上一层应得的值。
- 唯一"声明了但永远做不到"的是 `cursor`：1.7.10 的界面没有自己的指针，鼠标光标是操作系统的。

## 一、选择器

| 选择器 | 例 | 说明 |
| --- | --- | --- |
| 类型 | `div` | |
| 类 | `.mui-button` | 可连写 `.a.b` |
| id | `#panel` | |
| 通配 | `*` | |
| 属性 | `[type="checkbox"]` `[disabled]` | 支持 `=` `~=` `^=` `$=` `*=` `|=` |
| 后代 / 子 / 相邻 / 一般兄弟 | `A B` `A > B` `A + B` `A ~ B` | |
| 选择器列表 | `.a, .b` | 列表里某一个不受支持时，**其余仍生效**并上报 |
| `:not(…)` | `:not(.disabled)` | 里面只能放简单选择器 |

**不支持的**（会被丢掉并上报 `unsupported or malformed selector: '…'`）：
伪元素 `::before` / `::after`、`::first-line` 等；`:has()`、`:is()`、`:where()`；
属性选择器的大小写修饰符 `i`/`s`。

选择器列表里只有部分能读时，能读的部分照常生效——这是刻意的：整条规则因为一个拼错的
选择器而消失，是"我明明写了样式却没生效"最常见的来源。

## 二、伪类

**结构**：`:first-child` `:last-child` `:only-child` `:nth-child(an+b)`
`:nth-last-child()` `:first-of-type` `:last-of-type` `:only-of-type` `:nth-of-type()`
`:nth-last-of-type()` `:empty` `:root`

**交互/状态**：`:hover` `:active` `:focus` `:focus-visible` `:focus-within`
`:checked` `:disabled` `:enabled`

两点值得单独说：

- `:focus-visible` 与 `:focus` 是**两件事**：前者只对键盘到达的焦点为真（Tab、方向键），
  后者对鼠标点击留下的焦点也为真。组件表里的焦点环用 `:focus-visible`，正是为了
  "点谁给谁套个环"不出现。
- `:checked` 读的是元素上的 `checked` 属性，不区分 checkbox / radio / 组内选项行。

## 三、优先级与来源顺序

与 CSS 一致，从低到高：

1. **基础样式表**（引擎内置，见下）；
2. 作者样式表，按 **文档顺序**（`<link>` 先后、`<style>` 位置）；同一个路径在多层资源里
   都有时，**高优先级层**的排在后面（[10 资源](10-resources.md)）；
3. `style="…"` 行内声明；
4. 上面任何一条带 `!important` 的声明。

特异性按 `(id, class/属性/伪类, 类型)` 三元组比较，与浏览器相同。同特异性时后者胜。

### 基础样式表（引擎内置）

它在作者样式**之前**、优先级最低，等价于浏览器的 user-agent 表。规则只有这些：

```css
head, style, title, meta, link, base, script { display: none }
html, body { margin: 0; padding: 0 }
html, body, div, p, h1, h2, h3, h4, h5, h6,
ul, ol, li, dl, dt, dd, blockquote, pre, form, fieldset,
header, footer, section, article, main, nav, aside,
figure, figcaption, hr, table, thead, tbody, tfoot, tr, td, th { display: block }
button, input, textarea, select { display: inline-block }
b, strong { font-weight: bold }
i, em, cite, var { font-style: italic }
pre { white-space: pre }
input { width: 100px; padding: 2px 4px; min-height: 10px; background-color: #2a2a33;
        color: #e6e6ea; border: 1px solid #4a4a57 }
textarea { width: 200px; height: 48px; /* 同上四行 */ }
input[type="checkbox"], input[type="radio"] { width: 10px; height: 10px; padding: 0;
        color: #4cb862; border-radius: 2px }
input[type="radio"] { border-radius: 5px }
select { width: 120px; padding: 2px 4px; min-height: 10px; /* 同 input */ }
.mui-select-list { /* 下拉浮层：z-index 1000, max-height 96px, overflow auto, 1px 边框 */ }
.mui-select-option { padding: 2px 4px; white-space: nowrap }
.mui-select-option:checked { background-color: #3a3a46 }
input[type="range"] { width: 80px; height: 6px; padding: 0; border-width: 0;
        border-radius: 3px; background-color: #33333d; color: #4cb862 }
```

**没有的东西**和有的东西一样重要：没有 `h1` 的字号、没有 `ul` 的缩进、没有任何元素的
外边距。浏览器默认值在文档流里有用，在游戏界面里几乎全是需要重置的意外。

注意 `input { color: … }` 这一条：`color` 不只是文字颜色，**勾、圆点、滑块填充和光标**
都用它画，所以改一处就换掉了整份文档里所有控件的色调。

## 四、值

| 类型 | 支持 |
| --- | --- |
| 长度 | `px`、`%`、`em`（相对自身字号）、`rem`（相对根字号，`DocumentPipeline.setRootFontSize`） |
| 数字 | 整数与小数（`opacity`、`flex-grow`、`z-index`、`order`） |
| 颜色 | `#rgb` `#rgba` `#rrggbb` `#rrggbbaa`、`rgb()` `rgba()`、命名颜色、`transparent`、`currentcolor` |
| 图片 | `url(命名空间:路径)`，例如 `url(misiaui:textures/panel.png)`；**不联网** |
| 渐变 | `linear-gradient()`，**仅**垂直（`to bottom`/`180deg`）与水平（`to right`/`90deg`） |
| 时间 | `ms`、`s` |
| 缓动 | `linear` `ease` `ease-in` `ease-out` `ease-in-out`、`cubic-bezier(a,b,c,d)`、`steps(n)` / `steps(n, start\|end)` |
| 变量 | `var(--name)` 与 `var(--name, 回退值)` |
| 关键字 | `auto` `none` `initial` `inherit` `unset` 等，按属性 |

不支持的写法（`12vw`、`hsl()`、`radial-gradient()`、`conic-gradient()`、`steps(4, jump-both)`…）
会让**那一条声明**作废并上报，而不是整条规则。

**没有 `transform`**，所以元素不能旋转或缩放；想做"按下时缩小一点"请用 `padding` 或颜色。
**没有 `box-shadow`/`outline`**，阴影与描边请用 `border` 与九宫格边框
（`border-image-*`，见 [06 组件](06-components.md)）。

## 五、简写

引擎会展开这些简写（未写的分量重置为初始值，与 CSS 相同）：

`margin` `padding` `inset` ·
`border-width` `border-style` `border-color` `border` `border-top` `border-right`
`border-bottom` `border-left` `border-radius` ·
`flex` `flex-flow` `gap` `grid-gap` ·
`overflow` · `transition` · `animation`

**没有的简写**（写了就会上报"未实现"）：

| 简写 | 替代写法 |
| --- | --- |
| `background` | 分开写 `background-color` / `background-image` / `background-size` / `background-repeat` / `background-position` |
| `font` | 分开写 `font-family` / `font-size` / `font-weight` / `font-style` / `line-height` |
| `list-style` | 无（`ul`/`li` 本来就没有标记） |
| `place-items`、`place-content` | 用 `align-items` + `justify-content` |
| `inset-inline`、`margin-block` 等逻辑属性 | 用物理属性 |

`text-decoration` 在这里是**长手属性**（值可以是 `underline overline`），不是简写。

## 六、自定义属性与 `var()`

```css
:root {
  --mui-ink: #e6e6ea;
  --mui-panel: #1e1e24;
}
.mui-panel { background-color: var(--mui-panel); color: var(--mui-ink) }
.mui-panel .dim { color: var(--mui-muted, #7f7f8c) }   /* 带回退值 */
```

规则：

- 自定义属性**照常继承**，作用域就是元素树（所以主题只要写在 `:root` 或 `body` 上）；
- 简写里出现的 `var()` 会**推迟到替换之后**再展开——`border: 1px solid var(--ink)` 因此可以工作；
- 找不到的变量会报 `unresolved custom property: --name`，那一条声明作废、退回本来的值；
- 一个 `var()` 展开成多个分量喂给单值属性时，会报
  `'padding-top' takes a single value but resolved to 2 components: 1px 2px`。

组件表里用到的滚动条颜色也是自定义属性：`--mui-scrollbar-thumb`、`--mui-scrollbar-track`
（见 [05 布局与排版](05-layout.md) 的滚动一节）。

## 七、`@media` 与 `@import`

```css
@import "palette.css";              /* 也可以写 @import url(palette.css); */

@media (max-width: 320px) { … }
@media (min-height: 240px) and (orientation: landscape) { … }
@media screen and (max-width: 320px), print { … }
```

- 支持的类型：`all`、`screen`（其他类型不匹配）；支持 `not`、`and`、逗号分隔的备选。
- 支持的媒体特性：`min-width` `max-width` `width` `min-height` `max-height` `height`
  `orientation`（`landscape` = 宽 ≥ 高）。
- 视口尺寸是**逻辑像素**——被 GUI scale 归一化过，所以 `@media` 反应的是布局真正拥有的空间。
- **不认识的媒体条件一律"不匹配"并上报**（`unsupported @media condition (never matches): …`）。
  反过来做（当作匹配）会让 `(max-width: 320px)` 的规则在所有宽度下生效，那是个没有解释的布局 bug。
- `@import` 的路径**相对引入它的样式表**解析，可以 `../`；被引入的样式表在整个层叠里排
  **引入者之前**（所以引入者能覆盖它）。缺失、越界、成环都会上报。`@media` 里的 `@import`
  不支持（CSS 也不允许）。
- `@keyframes` 里的 `@media` 不支持（见 [14 已知限制](14-limitations.md)）。

## 八、属性总表

**布局与盒模型**（详见 [05 布局与排版](05-layout.md)）

| 属性 | 取值 |
| --- | --- |
| `display` | `none` `block` `flex` `inline` `inline-block` `inline-flex` |
| `position` | `static` `relative` `absolute` `fixed` |
| `top` `right` `bottom` `left` | 长度、`auto`（`left`/`top` 胜过 `right`/`bottom`） |
| `z-index` | 整数、`auto`；负值画在本父级内容之后 |
| `width` `height` | 长度、`%`、`auto` |
| `min-width` `min-height` `max-width` `max-height` | 长度、`%`、`none` |
| `box-sizing` | `content-box` `border-box` |
| `margin-*` `padding-*` | 长度、`%`、`auto`（仅 margin） |
| `overflow-x` `overflow-y` | `visible` `hidden` `scroll` `auto`（`auto`/`scroll` 同义，都画滚动条） |
| `flex-direction` | `row` `row-reverse` `column` `column-reverse` |
| `flex-wrap` | `nowrap` `wrap` `wrap-reverse` |
| `justify-content` | `flex-start` `flex-end` `center` `space-between` `space-around` |
| `align-items` `align-self` `align-content` | `stretch` `flex-start` `flex-end` `center` `auto`（self） |
| `flex-grow` `flex-shrink` `flex-basis` | 数字、长度、`auto` |
| `order` | 整数 |
| `row-gap` `column-gap` | 长度 |

**边框与装饰**

| 属性 | 取值 |
| --- | --- |
| `border-*-width` | 长度、`thin` `medium` `thick` |
| `border-*-style` | `none` `hidden`（不画）与其他任意值（**画成实心边**：引擎不画虚线/双线） |
| `border-*-color` | 颜色，默认 `currentcolor` |
| `border-*-radius` | 长度（每角可两个分量做椭圆角） |
| `border-image-source` | `url(...)` |
| `border-image-slice` | 1–4 个数字（贴图像素）或 `%`，可带 `fill` |
| `border-image-width` | 1–4 个数字（像素）或 `%`，默认 `auto`＝与 slice 相同 |
| `border-image-repeat` | `stretch` `repeat` `round`，可给两个（水平/垂直） |

**文字**

| 属性 | 取值 |
| --- | --- |
| `color` | 颜色（也决定勾/圆点/滑块/光标） |
| `font-family` | 字体名列表（**只用第一个**，没有回退链） |
| `font-size` | 长度（`em` 相对父级字号） |
| `font-weight` | `normal` `bold` `100`–`900` |
| `font-style` | `normal` `italic` `oblique` |
| `line-height` | `normal`、数字（自身字号的倍数）、长度 |
| `letter-spacing` | 长度（**每个字符之后**都加，最后一个也算，与 CSS 相同） |
| `text-align` | `left` `right` `center` `justify` |
| `white-space` | `normal` `nowrap` `pre` `pre-wrap` `pre-line` |
| `word-break` | `normal` `break-all` `keep-all` |
| `text-overflow` | `clip` `ellipsis`（需要 `overflow: hidden`，否则无效） |
| `line-clamp` | 整数、`none`（只对装文字的盒子有意义） |
| `text-decoration` | `none` `underline` `overline` `line-through`（可组合） |
| `text-decoration-color` | 颜色，默认 `currentcolor` |

**绘制、动效与可见性**

| 属性 | 取值 |
| --- | --- |
| `background-color` | 颜色（`currentcolor` 可用） |
| `background-image` | `url(...)`、`linear-gradient(...)` |
| `background-size` | 1–2 个长度/`%`、`auto`（`auto` 读作边框盒大小）、`contain` `cover`（需要贴图的固有尺寸，引擎会去问平台；问不到时上报并不画） |
| `background-repeat` | `repeat` `no-repeat` `repeat-x` `repeat-y` |
| `background-position` | 1–2 个位置（`left/center/right/top/bottom`、长度、`%`） |
| `opacity` | 0–1 |
| `visibility` | `visible` `hidden`（保留占位，不画） |
| `transition-property` `transition-duration` `transition-timing-function` `transition-delay` | 属性名或 `all`、时间、缓动、时间 |
| `animation-name` `animation-duration` `animation-timing-function` `animation-delay` `animation-iteration-count` `animation-direction` `animation-fill-mode` `animation-play-state` | `@keyframes` 名、时间、缓动、时间、数字或 `infinite`、`normal` `reverse` `alternate` `alternate-reverse`、`none` `forwards` `backwards` `both`、`running` `paused` |
| `cursor` | **声明了但做不了**：1.7.10 的界面没有自己的指针 |

## 九、什么时候会被上报

| 你写的 | 诊断里会出现 |
| --- | --- |
| `box-shadow: …` | `unsupported property 'box-shadow'; MUI does not implement it, so this declaration has no effect` |
| `background: #333` | 同上（`background` 不是这里的简写） |
| `display: table` 这类不认识的关键字 | 值读不懂时的报告；那一条声明作废，退回本来的值 |
| `@media (min-resolution: 2dppx)` | `unsupported @media condition (never matches): …` |
| `.a::before { … }` | `unsupported or malformed selector: '.a::before'` |
| `color: var(--nope)` | `unresolved custom property: --nope` |
| `padding-top: var(--pair)`（`--pair: 1px 2px`） | `'padding-top' takes a single value but resolved to 2 components: 1px 2px` |
| `transition-property: box-shadow` | `transition-property: 'box-shadow' cannot be transitioned; …` |
| `animation-name: nope` | `animation-name: 'nope' is not declared by any @keyframes in this page; …` |
| `@supports (display: grid) { … }` | `unsupported at-rule @supports (…) - block ignored` |

一次运行里同一个属性只报一行（不是每条规则一行）；页面级诊断、离线预览的 `.layout.txt`、
以及**游戏日志**都会看到同一批内容（见 [13 调试](13-debugging.md)）。

## 相关

- 布局与排版细节：[05 布局与排版](05-layout.md)
- 控件与状态：[07 表单控件](07-forms.md)
- 动画与过渡：[01 总览](01-overview.md)、[06 组件](06-components.md) 的 `mui-pulse`
- 完整的"没实现"清单：[14 已知限制](14-limitations.md)
