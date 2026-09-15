# 03 HTML 参考

MUI 读的是**标记**，不是浏览器。这一页说明它的解析器接受什么、容错到哪一步、
哪些属性是引擎认识的。规则只有一条总纲：**尽量把页面读出来，读不出来的部分明确上报**。

## 一、解析器的容错规则

它不是 HTML5 规范解析器，也不打算是。它处理界面页面真正会写的形状，以及人们真正会犯的错：

| 写法 | 结果 |
| --- | --- |
| `<br>`、`<img>`、`<input>`、`<hr>`、`<link>`、`<meta>` 等**空元素** | 永不开启作用域，不需要 `</br>` |
| `<div/>` 自闭合语法 | 任何元素都接受 |
| 结束标签 | 关闭最近的同名打开祖先 |
| 输入结束时仍未闭合的标签 | 保留，不报错 |
| 没有匹配开始标签的游离结束标签 | 忽略 |
| 单独的 `<`（如 `a < b`） | 当作普通文本 |
| `<style>` 与 `<title>` 的内容 | 原样保留，不当标记解析 |
| `<!-- 注释 -->`、`<!DOCTYPE>`、`<?…?>` | 跳过 |
| 同名标签重复打开（`li`、`option`、`tr`、`td`、`th`、`dt`、`dd`） | 自动关闭前一个 |
| `<script>…</script>` | **连同内容整段丢弃** |
| 具名实体（`&amp;` `&lt;` `&nbsp;` 等）、数字实体（`&#65;` `&#x41;`） | 解码 |

两处与浏览器的**故意差别**：

- **`<script>` 被丢掉**，不是保留成文本。MUI 没有 JavaScript，留着只会让人去找根本不存在的
  行为。
- **不合成 `<head>`/`<body>`**。布局不区分它们，合成只会让文档树多出两层没人读的节点。
  你仍然可以写它们（`<link>` 常写在 `<head>` 里），只是引擎不会替你补。

页面总是有**唯一根元素**：源码里没有 `<html>` 时引擎补一个。

## 二、标签

MUI 的标签集合是"浏览器标签的一个子集"，没有专属控件标签、不需要注册。

**结构类**：`html` `body` `div` `p` `h1`–`h6` `ul` `ol` `li` `dl` `dt` `dd` `blockquote` `pre`
`header` `footer` `section` `article` `main` `nav` `aside` `figure` `figcaption` `hr`
`table` `thead` `tbody` `tfoot` `tr` `td` `th`
**行内类**：`span` `b` `strong` `i` `em` `cite` `var` `code` `small` `br` `label`
**表单类**：`form` `fieldset` `input` `textarea` `select` `option` `button`
**文档类**：`head` `title` `meta` `link` `base` `style`（`display: none`）

> **表格标签只是"块级盒子"。** `table`/`tr`/`td` 在基础样式表里是 `display: block`，
> MUI 没有表格布局算法。用它们做结构会被当成竖着堆的 `div`；要做格子请用 flex。

> **`<img>` 不存在。** 空元素列表里有它，但引擎不实现替换元素；背景图走
> `background-image`（[10 资源](10-resources.md)）。

基础样式表里还规定了：`b/strong` 加粗、`i/em/cite/var` 斜体、`pre` 保留空白、
`html`/`body` 无内外边距、**没有任何元素的默认字号与边距**（浏览器的 `h1` 大字号、
`ul` 缩进在这里都不存在——游戏界面里它们全是需要重置的意外）。全文见
[04 CSS 参考](04-css.md) 的"基础样式表"一节。

## 三、样式表怎么接进来

两种写法，都可以：

```html
<!-- 外部样式表：href 相对当前文档 -->
<link rel="stylesheet" href="../theme/furnace.css">

<!-- 页内样式块 -->
<style>
  #panel { border-color: #4a4a57 }
</style>
```

`<link>` 的规则：

- 只有 `rel` 里**含有** `stylesheet` 这个词的才算（`rel="stylesheet preload"` 也算，
  按空格切分）；
- `href` 相对**当前文档**解析；`../` 可以，但**不能爬出资源根**（爬出去会报
  `points outside the bundle root and was ignored`）；
- **不联网**：`https://…` 会报 `is remote; MUI does not fetch resources`；
- Minecraft 自己的 `域名:路径` 形式会单独报 `names the bundle '…'`——MUI 只有一个 bundle，
  没有第二个域可以解析；
- 找不到的样式表会报 `no resource layer has the stylesheet '…', referenced by …`，
  而不是安静地不生效。

**顺序就是 CSS 的层叠顺序**：文档中 `link` 的先后顺序决定，后面的赢平局；
同一个路径在多层资源里都有时，**优先级高的一层排在后面**（见 [10 资源](10-resources.md)）。

页内 `<style>` 每次重建都会重新解析（这是帧里唯一读文本的部分，耗时被单独计时）。
`<style>` 里的诊断会和层叠诊断一起出现在页面的诊断里；外部样式表的解析诊断归在"页面装载
报告"里。

## 四、MUI 认识的属性

除了标签自己的语义属性（`id`、`class`、`style`、`type`、`value`、`checked`、`name`、
`placeholder`、`maxlength`、`min`、`max`、`step`、`disabled`、`selected`、`href`、`rel`、
`tabindex`），下面这些是**引擎会读的**：

| 属性 | 写在哪 | 作用 | 详见 |
| --- | --- | --- | --- |
| `data-on-click="名字"` | 任何元素 | 点击时运行动作 | [09 动作](09-actions.md) |
| `data-on-key="Enter:提交 Escape:取消"` | 任何元素 | 按键时运行动作，空格分隔多组 | [09 动作](09-actions.md) |
| `tabindex` | 任何元素 | 进入 Tab 顺序（数值不参与排序） | [07 表单](07-forms.md) |
| `data-roving` | 容器 | 让容器成为一组：Tab 停一次、方向键在里面走 | [07 表单](07-forms.md) |
| `data-roving="horizontal"｜"vertical"｜"both"` | 同上 | 限制这组响应哪一对方向键 | [07 表单](07-forms.md) |
| `data-select="multiple"` | 同上 | 允许组内多项同时选中 | [07 表单](07-forms.md) |
| `data-repeat="row in rows"` | 任何元素 | 按数据重复该元素 | [08 绑定](08-binding.md) |
| `data-if="${条件}"` | 任何元素 | 条件为假则整个元素不出现 | [08 绑定](08-binding.md) |
| `checked` `disabled` `value` `selected` | 表单控件 | 控件的状态；引擎读写 | [07 表单](07-forms.md) |

`data-mui-list` 与 `data-mui-option` 是**引擎自己**给下拉浮层写的标记，页面不要用它们
（见 [07 表单](07-forms.md)）。

**没有 `data-on-hover`、没有自定义事件、没有事件冒泡语义**。点击会沿祖先向上找最近的
`data-on-click`（所以按钮里的文字或图标不需要各写一遍），这是唯一一条"冒泡"。
键盘同理：按键从有焦点的元素往上找 `data-on-key`。只有被点名的键会被吃掉，
其余按键照常交给游戏。

## 五、`${…}` 写在哪里

`${…}` 可以出现在**文本节点**和**属性值**里：

```html
<div class="row">
  <span>${item.name}</span>
  <span class="mui-badge" style="background-color: ${item.colour}">${item.count|number}</span>
  <input value="${item.name}" placeholder="名称">
  <button data-on-click="${item.action}">Go</button>
</div>
```

规则：

- 属性值里的 `${…}` 与文本里的解析完全一样；
- 整个属性值恰好是一个占位符时，值是**插值后的字符串**（不是对象）；
- 结果里的 `${` 不会再被解析（不会递归展开）；
- `data-repeat` 与 `data-if` 这两个指令属性本身在绑定后**不会留在文档里**——绑定产物是一个
  新文档，不会再次要求被绑定。

详见 [08 数据绑定](08-binding.md)。

## 六、文档的形态

```html
<!DOCTYPE html>            <!-- 跳过，写不写都行 -->
<html>                     <!-- 可省，引擎会补 -->
  <head>
    <title>Furnace</title> <!-- 原样保留，不渲染 -->
    <link rel="stylesheet" href="../theme/components.css">
    <link rel="stylesheet" href="furnace.css">
  </head>
  <body>
    <div class="mui-panel">…</div>
  </body>
</html>
```

不需要 `<meta charset>`（资源统一按 UTF-8 读取）；写了也不会被误当作内容。

## 七、什么时候会被上报

| 你写的 | 诊断里会出现 |
| --- | --- |
| `<link>` 指向不存在的样式表 | `no resource layer has the stylesheet '…', referenced by …` |
| `<link href="https://…">` | `'https://…' in screens/a.html is remote; MUI does not fetch resources` |
| `<link href="../../x.css">` | `'../../x.css' in screens/a.html points outside the bundle root and was ignored` |
| `<link rel="stylesheet">` 没写 `href` | `a <link rel="stylesheet"> in screens/a.html has no href` |
| `<input type="date">` 之类 | `<input type="date"> is not a control MUI implements, so it was left alone` |
| 会被解析成行内内容却排不出来的元素 | `<…> is inline content MUI cannot lay out yet, so it was ignored` |

最后一条值得记一下：**它意味着这个元素没有生成任何盒子**，所以它既不显示、也不接受点击。
在 MUI 里"看得见的元素一定有盒子"，一条这样的诊断就是"这里少了一个盒子"。

## 相关

- 样式怎么写：[04 CSS 参考](04-css.md)
- 盒子为什么在那里：[05 布局与排版](05-layout.md)
- 控件：[07 表单控件](07-forms.md)
- 诊断都在哪：[13 调试](13-debugging.md)
