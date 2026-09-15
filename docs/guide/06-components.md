# 06 组件

MUI 随 jar 附一份组件样式表 `theme/components.css`。它不是引擎的一部分——**引擎不知道
`.mui-button` 是什么**，这正是设计：组件是**约定**，和任何 CSS 库里的组件一样，全部用页面自己
能写的普通 CSS 写成。想换样子就覆盖一条规则，不用和"对按钮有意见的引擎"较劲。

```html
<link rel="stylesheet" href="../theme/components.css">
```

`mui-` 前缀是这套表对页面唯一的请求：页面自己的类和库的类在同一个命名空间里，
一个不保留任何名字的 user agent 最后会保留所有名字。

现成的样板页：`screens/gallery.html` 把每个类都摆了一遍（也就是离线预览里的 `gallery.png`）。

## 一、面板与排版

| 类 | 作用 |
| --- | --- |
| `.mui-panel` | 主面板：竖向 flex、8px 内边距、`#1e1e24` 底、1px 边框、圆角 4 |
| `.mui-surface` | 内嵌的一块面：竖向 flex、6px 内边距、`#26262e` 底、圆角 3，没有边框 |
| `.mui-title` | 标题：12px、近白 |
| `.mui-subtitle` | 副标题：10px、灰 |
| `.mui-section` | 小节抬头：10px、灰、字距 0.4px，上下有边距 |
| `.mui-divider` | 1px 分隔线 |
| `.mui-row` | 横向 flex + `gap: 6px` + `align-items: center`（标签和 10px 复选框高度不同，顶端对齐会显得散） |
| `.mui-row-between` | 加在 `.mui-row` 上：两端对齐（左标签、右数值） |
| `.mui-label` | 行内标签色 |
| `.mui-value` | 右对齐的数值色 |

```html
<div class="mui-panel">
  <div class="mui-title">Furnace</div>
  <div class="mui-subtitle">3 slots loaded</div>
  <div class="mui-section">ENERGY</div>
  <div class="mui-row mui-row-between">
    <div class="mui-label">Buffer</div>
    <div class="mui-value">6,200 FE</div>
  </div>
  <div class="mui-divider"></div>
</div>
```

## 二、按钮

| 类 | 作用 |
| --- | --- |
| `.mui-button` | 基本按钮：`3px 9px` 内边距、圆角 3、`150ms linear` 的背景色过渡；`:hover` 变亮、`:active` 变暗、`:focus-visible` 更亮 |
| `.mui-button-primary` | 绿色主按钮（绿底白字） |
| `.mui-button-danger` | 红色危险按钮 |
| `:disabled` | `opacity: 0.4`（禁用后引擎本来就不给点、也不进 Tab 顺序，这条只管样子） |

```html
<button class="mui-button" data-on-click="cancel">Cancel</button>
<button class="mui-button mui-button-primary" data-on-click="craft">Craft</button>
<button class="mui-button mui-button-danger" data-on-click="scrap">Scrap</button>
<button class="mui-button" disabled>Disabled</button>
```

焦点环用的是 **`:focus-visible`** 而不是 `:focus`：环属于"用键盘走到这里的人"，
点了鼠标就套一个环看起来像 bug。引擎把这两件事分开记，这条规则就是为此存在的。

## 三、徽章与进度条

| 类 | 作用 |
| --- | --- |
| `.mui-badge` | 小徽章：紫底浅字、9px、圆角 2 |
| `.mui-badge-ok` | 绿色徽章（"在线""已完成"） |
| `.mui-progress` | 进度条轨道：`flex: 1`、高 6、圆角 3、深底 |
| `.mui-progress-fill` | 填充条：绿色、高 6、圆角 3、`200ms linear` 的 **width 过渡** |

```html
<div class="mui-row">
  <div class="mui-progress"><div class="mui-progress-fill" style="width: ${percent}%"></div></div>
  <span class="mui-value">${percent}%</span>
</div>
```

`width` 是**页面的事**（行内 `style` 或一个类），组件表只负责让"数值变了"看起来像"真的变了"。
注意 `width` 的过渡需要像素长度才能插值；`65%` → `40%` 这种百分比之间的过渡引擎无法插值，
会**如实上报**并直接跳到终点（见 [04 CSS 参考](04-css.md) 的"上报"）。

## 四、标签页与列表

两者都是 `data-roving` 组：整条 Tab 停一次，方向键在里面走，`:checked` 标记当前项。

| 类 | 作用 |
| --- | --- |
| `.mui-tabs` | 横向 flex + 2px 间距 |
| `.mui-tab` | 单个标签：`:hover` 变亮、`:checked` 绿底白字 |
| `.mui-list` | 竖向 flex 的列表 |
| `.mui-list-row` | 一行：两端对齐、`:hover` 提亮、`:checked` 绿底白字、150ms 背景过渡 |
| `.mui-scroll` | 加在列表上：`overflow-y: auto` + `max-height: 66px`（滚动条由引擎画） |

```html
<div class="mui-tabs" data-roving="horizontal">
  <button id="tab-1" class="mui-tab" data-on-click="tab" checked>Craft</button>
  <button id="tab-2" class="mui-tab" data-on-click="tab">Smelt</button>
  <button id="tab-3" class="mui-tab" data-on-click="tab">Repair</button>
</div>

<div id="ore-list" class="mui-list mui-scroll" data-roving="vertical" data-select="multiple">
  <div class="mui-list-row" tabindex="0" checked><div>Iron Ingot</div><div class="mui-value">64</div></div>
  <div class="mui-list-row" tabindex="0"><div>Gold Nugget</div><div class="mui-value">12</div></div>
</div>
```

- 组里**不需要**给每一项写 `tabindex="-1"`：组自己决定 Tab 落在哪一项，方向键在里面移动并
  循环（不会在两端卡住）。写了 `tabindex` 或 `data-on-click` 的元素就是"项"；
- `data-select="multiple"` 表示**多项可同时选中**（`data-select` 其它任何值都按单选处理）；
- 列表要有 **`id`**：滚动位置记在 id 上（[05 布局与排版](05-layout.md)）；
- 点击落项时引擎写 `checked` 并清掉同级——这就是勾选、高亮、`${row.selected}` 全都能一致的原因。

## 五、文字

| 类 | 作用 |
| --- | --- |
| `.mui-note` | 注脚：9px 灰、`line-clamp: 2`（超出两行**根本不布局**） |
| `.mui-mono` | 等宽小字（编号、坐标、路径） |

## 六、对话框（没有一行 Java）

```html
<input id="dialog-toggle" class="mui-dialog-toggle" type="checkbox">
<div class="mui-dialog-scrim">
  <div class="mui-dialog">
    <div class="mui-title">Delete this machine?</div>
    <div class="mui-row">
      <label><input type="checkbox" id="confirm"> I am sure</label>
    </div>
    <div class="mui-row mui-row-between">
      <button class="mui-button" data-on-click="cancel">Cancel</button>
      <button class="mui-button mui-button-danger" data-on-click="confirm-delete">Delete</button>
    </div>
  </div>
</div>
```

组件表里的全部魔法就是一条选择器：

```css
.mui-dialog-toggle:checked ~ .mui-dialog-scrim { display: flex }
```

一个**引擎真的会操作的复选框**就是状态本身，一般兄弟选择器把它变成一块面板。
`.mui-dialog-scrim` 平时 `display: none`，勾上以后铺满视口（`position: absolute` +
`100%` 宽高 + 半透明黑 + 居中）。这是 CSS 里最老的招数，能在这里用，是因为"复选框"在 MUI 里
是真控件，而不是画出来的样子。

遮罩会**吃掉落在它上面的点击**——这正是模态该有的行为，所以它也顺带把下面的面板挡住了。

## 七、覆盖与换皮

组件表的每条规则都是普通 CSS，覆盖它不需要权限：

```css
/* 整个文档的控件色调，一条规则 */
input, select, textarea { color: #7fd4ff }

/* 只改这一处按钮 */
#scrap { background-color: #6b2f2f }

/* 换一个面板底色，所有面板一起变 */
.mui-panel { background-color: #101018; border-color: #2a2a33 }
```

三条建议：

1. **在自己的样式表里覆盖**，顺序在 `components.css` 之后（`<link>` 写在后面，或把覆盖写进
   `<style>`）——同优先级时后者胜；
2. 需要更高优先级时用 `#id` 或 `!important`，但优先考虑换一个类名而不是加 `!important`；
3. **不要改 `components.css` 本身**（它在 jar 里，改了也会被更新覆盖）。要整份换掉，就写一份
   自己的表，页面只链接自己的那份。

一份主题怎么随资源包发布，见 [10 资源与热重载](10-resources.md) 与 [12 主题](12-theming.md)。

## 八、自己写组件

组件表里没有一个类依赖引擎内部机制，所以新组件就是新 CSS：

```css
.mui-meter {
  height: 8px;
  background-color: #22222a;
  border-radius: 4px;
  overflow: hidden;              /* 让填充条的圆角被裁成外框的圆角 */
}
.mui-meter > .mui-meter-fill {
  height: 100%;
  background-image: linear-gradient(to right, #2f6b3d, #4cb862);
  transition-property: width;
  transition-duration: 200ms;
}
```

要注意的只有两条：**只用引擎真有的属性**（`box-shadow` 之类会被上报且不生效，
见 [14 已知限制](14-limitations.md)），以及**状态写在文档里**（用 `:checked`、`:hover`、
`:disabled`，不要另造一套）。

## 相关

- 每个属性的取值：[04 CSS 参考](04-css.md)
- 控件的行为：[07 表单控件](07-forms.md)
- 完整样板：`src/main/resources/assets/misiaui/screens/gallery.html`
