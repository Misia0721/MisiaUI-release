# 07 表单控件

MUI 自己操作这些控件——画勾、画圆点、画滑块、画光标、管选区、管键盘——页面只写标记，
状态一律写在元素的属性上。

| 控件 | 写法 | 状态属性 |
| --- | --- | --- |
| 文本框 | `<input>` 或 `<input type="text">` | `value` |
| 多行文本 | `<textarea>` | `value`（内容也行） |
| 勾选框 | `<input type="checkbox">` | `checked` |
| 单选框 | `<input type="radio">` | `checked` + `name` |
| 滑块 | `<input type="range">` | `value`、`min`、`max`、`step` |
| 下拉框 | `<select>` + `<option>` | 选中的 `<option>` 的 `selected`，值走 `value` |

`<input>` 的其它 `type`（`date`、`file`、`number`…）会被**识别并上报**，但不生成控件：

```
<input type="date"> is not a control MUI implements, so it was left alone
```

## 一、一条规则：状态住文档里

```html
<input type="checkbox" id="strict" checked>
```

- `checked` 是**元素的属性**，引擎写它、样式表读它（`:checked`）、绑定也能读它
  （`${...}` 或 `data-if="${strict}"` 不行——见下）；
- HTML 的布尔属性规则照旧：**存在即为真**，只有字面量 `"false"` 为假。所以 `checked`、
  `checked=""`、`checked="true"` 都是勾上；
- 想让"勾没勾"跟着数据走，用绑定写属性：`checked="${row.selected}"`。
  注意 `${false}` 渲染出来是字符串 `"false"`（恰好为假），而 `${null}`/`${""}` 会渲染成**空串**，
  空串对布尔属性是**真**——所以视图模型应当给真正的布尔值，或者在数据里增删这个属性。

## 二、文本框

```html
<input id="name" value="${machine.name}" placeholder="名称" maxlength="16">
```

| 属性 | 行为 |
| --- | --- |
| `value` | 字段里的内容。引擎把玩家输入的结果**写回**这个属性，页面读同一个位置 |
| `placeholder` | 值为空时显示，颜色比正文淡（没有 `::placeholder` 这个选择器，样式跟随控件的 `color`） |
| `maxlength` | 最多接受的字符数 |
| `disabled` | 不可聚焦、不可点、不进 Tab 顺序（并且作用于整棵子树） |

**非数字的值不会被上报**：`maxlength="abc"` 会报 `<input> has maxlength="abc", which is not a number`，
然后用默认值。这类报告值得当回事——它意味着页面以为自己在限制什么，其实没有。

键盘：

| 键 | 行为 |
| --- | --- |
| 可打印字符 | 插入到光标处（会替换选区） |
| `Backspace` / `Delete` | 删前一个 / 后一个字符（有选区时删选区） |
| `Left` / `Right` | 移动光标；按住 `Shift` 则扩选 |
| `Home` / `End` | 行首 / 行尾；按住 `Shift` 则扩选 |
| `Ctrl+A` | 全选 |
| `Enter` | 在单行字段里**不是内容**，会交给页面（`data-on-key="Enter:submit"` 可以接） |

鼠标：点一下放光标，按住拖动就是拖选。**没有双击选词、三击选行**（见
[14 已知限制](14-limitations.md)）。

## 三、多行文本

```html
<textarea id="notes" style="width: 200px; height: 60px">one
two</textarea>
```

- **内容是初始值**（HTML 的规矩：带换行的字符串不该塞进属性里）；开局那一个换行会被丢掉，
  所以 `<textarea>\n  hello</textarea>` 只有一行；
- 玩家输入后，引擎把值写进 `value` 属性——**两种写法读同一个位置**；
- `Enter` 在这里**是内容**（换行），不是按键；
- `Up`/`Down` 在行之间移动（列不"粘住"，见限制）；
- 没有 `rows`/`cols`，也没有软换行：**一行就是一行**，宽度超出的部分被裁掉，
  光标走到看不见的地方时内容会**滚动**到光标处；盒子大小用 CSS 给（基础样式表给了 200×48）。

## 四、勾选框与单选框

```html
<label><input type="checkbox" id="sneak" checked> Sneak</label>

<label><input type="radio" name="mode" value="fast" checked> Fast</label>
<label><input type="radio" name="mode" value="slow"> Slow</label>
```

- 画的是**引擎的勾与圆点**，颜色取自控件的 `color`（所以 `input { color: … }` 一条规则换掉
  全文档控件的色调）；
- 圆点是**圆的**：基础样式表给 `border-radius: 2px`（勾选框）与 `5px`（单选框），页面可以改；
- 单选按钮的互斥**靠 `name`**：同一个 `name` 的一组里点一个会清掉其它。
  没有 `name` 的单选按钮**不会互斥**，并且会上报：
  `a radio button has no name, so nothing keeps it exclusive with anything`；
- 禁用的控件不响应点击，且不进 Tab 顺序。

## 五、滑块

```html
<input type="range" id="amount" min="0" max="64" step="8" value="32">
```

- 拖动、点在轨道上（滑块会跳到按下的位置）、方向键（按 `step` 走一格）都能改值；值会**按 `step` 吸附**；
- `min` 缺省 0、`max` 缺省 100、`step` 缺省 1；写错时会报
  `<input> has … which is not a number, so … was used`；
- 滑块的填充色同样来自 `color`。

## 六、下拉框

```html
<select id="metal">
  <option>Iron Ingot</option>
  <option value="gold" selected>Gold Nugget</option>
  <option value="redstone">Redstone</option>
</select>
```

点击（或 `Enter`/`Space`/方向键）打开一个**真正的浮层**：它不是截图，而是引擎往页面文档里
插进去的一个元素树（`<div data-mui-list class="mui-select-list">` 里每行一个
`data-mui-option` 的 `div`）。

| 键 | 行为 |
| --- | --- |
| `Enter` / `Space` | 列表关着时打开它（不改选中） |
| `Down` / `Up` | 列表关着时打开并把高亮移一步（**不会**直接改选中项） |
| `Down` / `Up` | 列表打开时移动高亮，到两端循环 |
| `Home` / `End` | 列表打开时第一项 / 最后一项 |
| `Enter` / `Space` | 列表打开时确认高亮的那一项 |
| `Escape` | 关掉列表，什么都不改 |
| `Tab` | 关掉列表并继续走焦点顺序 |
| 点击控件 | 打开 / 关闭 |
| 点击某一项 | 直接选中它并关闭 |

为什么这样设计（也值得作者知道）：

- 浮层是**普通盒子**，所以层叠、盒模型、`z-index`、命中测试、裁剪、滚轮全都照常适用，
  不需要第二套机制；`data-mui-list` 就是它在文档里的标记，页面不要用它；
- 它的行是普通块盒子，所以**收缩宽度＝最宽的行**（列就是块流的本性）；
- 高亮用 `:checked`（和勾选框同一个属性），所以样式表标"当前行"不需要懂下拉框：
  ```css
  .mui-select-option:checked { background-color: #3a3a46 }
  ```
- **`<optgroup>` 不读**，组里的选项不会出现在列表里；**`multiple` 会被上报**（一次只选一个）；
  `size` 无效；列表**永远向下弹**，不会因为底部空间不够而上翻。都列在
  [14 已知限制](14-limitations.md)。

## 七、分组：`data-roving`

一组控件（标签页、工具栏、单选框组、选项列表）对玩家来说是**一个**到达点、**多个**选择项。
Tab 应该停在整组上，方向键在组内移动——这就是 roving tabindex 模式：

```html
<div class="mui-tabs" data-roving="horizontal">
  <button class="mui-tab" data-on-click="tab" checked>Craft</button>
  <button class="mui-tab" data-on-click="tab">Smelt</button>
</div>
```

| 写法 | 含义 |
| --- | --- |
| `data-roving` 或 `data-roving="both"` | 四个方向键都属于这一组 |
| `data-roving="horizontal"` | 只吃 `Left`/`Right`（另一对留给游戏，比如让列表滚动） |
| `data-roving="vertical"` | 只吃 `Up`/`Down` |
| `data-select="multiple"` | 组内多项可同时为 `checked`（默认单选，点一个清掉其它） |

- **项**的判定：元素带 `tabindex`、或带 `data-on-click`、或是引擎操作的控件。
  所以一组单选框、一行带 `tabindex` 的列表行，都不用额外声明什么；
- 移动**循环**：在最后一项按 `Right` 会回到第一项（键盘层最不该有的感觉就是"这个键没反应"）；
- `Home`/`End` 永远到两端；
- 组**嵌套**时，内层组是外层组的一个项，外层方向键落到内层组上就不再往里走；
- 写错的轴名（`data-roving="horizonal"`）会被**上报**，并按"两个轴都吃"处理——
  一个以为自己被限制了的页面，最糟的发现方式是"另一对键怎么没反应"。

## 八、焦点与 Tab 顺序

| 规则 | 说明 |
| --- | --- |
| 什么算控件 | `data-on-click`、`tabindex`、或引擎能操作的控件（表单控件不需要写 `tabindex`） |
| 顺序 | **文档顺序**。`tabindex` 的数字**不参与排序**（只表示"可到达"） |
| 组 | 整组算一个停靠点，停在"上次在的那一项"，没进过就停第一项 |
| 聚焦什么 | 点击找最近的祖先：先看 `data-on-click`，再看表单控件，最后看 `tabindex` |
| 看不见的不进顺序 | `display: none`、`visibility: hidden` 的控件会被跳过 |
| 禁用 | `disabled` 的控件与其整棵子树都不进顺序，也不响应点击 |
| `:focus` 与 `:focus-visible` | 前者含鼠标点击留下的焦点，后者只含键盘到达的焦点 |

## 九、编辑、重建与数据：谁说了算

这一段不长，但它决定了一个设置界面会不会"玩家刚输入的东西自己没了"。三条规则：

**1. 控件要有 `id`，否则玩家的输入活不过一次重建。** 编辑是**按 id 记住**的（只有名字能活过
重建的树）。没有 `id` 的控件照样能用，但保不住内容，并且会上报：

```
a control with no id cannot hold what the player put in it across a re-bind, because an id is
the only name that outlives the tree; give <input> an id
```

同理，滚动位置也是按 `id` 记的（[05 布局与排版](05-layout.md)）。

**2. 数据重新绑定时，数据赢。** 绑定重建整棵树，就是"页面在说控件该是什么值"，
所以玩家此前的编辑会被丢掉：

```java
source.put("name", "from the data");   // 数据动了 → 下一次绘制重建 → 输入框显示数据里的值
```

这条规则是有意的，而且它带来的好处正是你要的："恢复默认值"按钮会恢复**所有**东西，
包括玩家碰过的那些；如果编辑能压过数据，那个按钮看起来就是坏的。

**3. 要让输入活下来，就在动作里把它读回模型。** `MuiScreen` 注册的动作跑完会重新绑定
（`rebindAfterActions`，默认开），所以一个"保存"动作应当先读出控件当前的值：

```java
.on("save", source -> {
    Element field = Mui.current().pipeline().document().getElementById("name");
    model.setName(field.attribute("value"));      // 玩家的输入 → 模型
})
```

读的位置就是文档：引擎把玩家输入**写进 `value`/`checked` 属性**，所以"控件当前值"始终是
`document.getElementById(id).attribute("value")`，不需要另一套查询 API。反过来，页面想让控件
显示什么，就把它写进模型（或写进 `value` 属性），下一次绑定自然生效。

**键盘与鼠标的分工**也值得记一句：点击把光标放在**你点的位置**（不是行首也不是行尾），
所以"点一下就全选"这类期待在 MUI 里不成立——那是双击/三击的行为，还没有实现。

## 十、`select` 之外的浮层经验

`<select>` 的列表是"引擎往文档里插一个元素"的第一个例子，也是**现在唯一**的例子。
它带来的义务是**收回**：列表关闭时、以及每次重建文档前都会移除（`SelectMenu.remove`），
因为绑定会替换整棵树、旧的列表节点会指向一棵不存在的树。

如果你的页面需要一个类似的浮层（比如自定义颜色选择器），照这个模式做：
**用一个普通元素 + 绝对定位 + 高 `z-index`**，状态放在一个真控件的 `checked` 上
（对话框就是这么做的，见 [06 组件](06-components.md)）。引擎会像对待别的盒子一样对待它。

## 相关

- 控件的样子（组件表）：[06 组件](06-components.md)
- 控件相关的属性与诊断：[04 CSS 参考](04-css.md)
- 控件的限制与上报文案：[14 已知限制](14-limitations.md)
