# 14 已知限制

这一页是**诚实的清单**：MUI 现在做不到什么、遇到时会怎样表现、会不会上报。
写页面之前扫一眼这里，能省掉很多"为什么没反应"。

约定：**"上报"永远意味着那句话会出现在页面的诊断里**（[13 调试](13-debugging.md)），
并且**游戏日志**里也有。

## 一、HTML

| 限制 | 表现 | 是否上报 |
| --- | --- | --- |
| 没有 `<script>` | 内容整段被丢弃 | 不（刻意的：MUI 没有 JavaScript） |
| 没有 `<img>`、`<svg>`、`<canvas>`、`<audio>`、`<video>` | 元素被当行内内容处理掉，**不生成盒子** | 是：`<img> is inline content MUI cannot lay out yet, so it was ignored` |
| 表格标签没有表格布局 | `table`/`tr`/`td` 只是块级盒子，竖着堆 | 不（它们在基础样式表里就是 `block`） |
| 不合成 `<head>`/`<body>` | 你写就有，不写就没有；不影响布局 | 不 |
| 没有 `placeholder` 样式钩子 | 占位文字用控件的 `color` 淡淡地画；没有 `::placeholder` | 不 |

## 二、CSS：没有的属性

写了就会报 `unsupported property 'x'; MUI does not implement it, so this declaration has no effect`：

| 类别 | 缺什么 |
| --- | --- |
| 视觉 | `box-shadow`、`outline`、`filter`、`backdrop-filter`、`mix-blend-mode`、`mask`、`clip-path` |
| 变换 | `transform`、`transform-origin`、`perspective`（**元素不能旋转或缩放**） |
| 简写 | `background`、`font`、`list-style`、`place-*`、逻辑属性（`margin-inline` 等） |
| 布局 | `float`、`clear`、`position: sticky`、`display: grid`、`display: table*`、`aspect-ratio` |
| 文本 | `text-transform`、`text-shadow`、`word-spacing`、`overflow-wrap`、`hyphens`、`writing-mode` |
| 其他 | `content`（没有伪元素）、`caret-color`、`user-select`、`pointer-events`、`scroll-behavior` |

**替代方案**：阴影/描边用 `border` 与九宫格 `border-image-*`；缩放感用 `padding`、颜色或
`opacity` 的变化；圆角用 `border-radius`；格状布局用 flex + `flex-wrap`。

## 三、CSS：认识但做不满的属性

| 属性 | 限制 | 是否上报 |
| --- | --- | --- |
| `cursor` | **声明了但永远做不到**：1.7.10 的界面没有自己的指针 | 不（这是唯一一个"声明了但做不了"的属性） |
| `border-*-style` | 只区分"有没有边"；`dashed`/`dotted`/`double` 都画成实心 | 不 |
| `background-size: contain/cover` | 需要贴图的固有尺寸；问不到平台时画不出来 | 是 |
| `background-repeat: space/round` | 不支持，按 `repeat` 处理 | 是 |
| `background-size` + 圆角 | 贴图会画进方角里，不裁成圆角 | 是 |
| `linear-gradient()` | 只有垂直与水平两个方向；`45deg`、`radial-`、`conic-` 不支持 | 是 |
| `font-family` | 只用列表里的**第一个**字体，没有回退链 | 不（第一项之外的都被忽略） |
| `text-decoration` | 按"继承"实现（CSS 其实是传播），所以子元素设 `none` 能取消下划线 | 不 |
| `overflow: hidden` + 绝对定位 | 越界的浮层仍被裁剪（只有一条裁剪链，不是每个包含块一条） | 不 |
| 过渡的插值 | 像素长度、颜色、数字可以插值；**百分比与 `em` 不行**，会"跳到一半" | 是 |
| `animation` | 一个元素只跑一条动画（`animation-name` 的逗号列表只取第一个） | 是 |
| `@keyframes` 里的 `@media` | 不支持 | 是 |
| `steps(n, jump-*)` | 只读个数 `n`，跳变词被忽略（不报） | 不 |

## 四、布局

| 限制 | 表现 | 是否上报 |
| --- | --- | --- |
| 没有 `vertical-align` | 行内盒一律坐在基线上；`top`/`middle`/`bottom` 不生效 | 是（值读不懂） |
| 绝对定位 + 双轴 `auto` | 回退到父级内容原点，而不是"静态位置" | 不 |
| `top` + `bottom` + `height: auto` | 不会拉伸填满 | 不 |
| 嵌套的 inline-block | 内层由自己的内容定宽，不看外层最终宽度（单遍布局） | 不 |
| `z-index` 不建立新的堆叠上下文 | 只按"定位元素后画、负值先画"排序 | 不 |
| 控件声明成 `display: inline` | 行内元素没有盒子，控件**画不出来**、点不到 | 是：`… is inline content, and an inline element has no box to be drawn in; give it display: inline-block …` |
| 滚动容器没有 `id` | 滚动位置活不过一次重建（滚一下又跳回去） | 是：`a scrollable <div> has no id …` |

## 五、控件

| 限制 | 表现 | 是否上报 |
| --- | --- | --- |
| `<input type="…">` 只支持 `text` `checkbox` `radio` `range` | 其他类型（`date`、`file`、`number`…）不生成控件 | 是：`<input type="date"> is not a control MUI implements` |
| `<select multiple>` | 只取一个选项 | 是：`<select multiple> is not implemented …` |
| `<optgroup>` | 不读；组内选项**不会**出现在下拉里 | 不（它们是嵌套的，被当作不属于这个 select） |
| `<select size>` | 不生效，永远是"按一下弹出列表" | 不 |
| 下拉浮层不翻转 | 屏幕底部空间不够时也不向上弹 | 不 |
| 没有鼠标双击/三击选词 | 只有拖选、Shift+方向键、Ctrl+A | 不 |
| 没有页面级选区 API | 选区只在控件内部有意义，取文本要读控件的 `value` | 不 |
| `<textarea>` | 不软换行（一行就是一行）、没有 `rows`/`cols`（用 CSS 宽高）、光标列不"粘住" | 不 |
| `maxlength` 之外的校验 | 没有 `pattern`、没有表单提交语义 | 不 |
| 单选组的互斥 | 靠 `name` 属性；**没有 `name` 的单选按钮互不影响** | 是：`a radio button has no name, so nothing keeps it exclusive with anything` |

## 六、绑定与动作

| 限制 | 表现 | 是否上报 |
| --- | --- | --- |
| 没有表达式 | `${a + b}` 不计算；计算写在 Java 里 | 是（名字取不到值 → `nothing provides 'a + b'`） |
| 格式化器是固定集合 | `number` `upper` `lower` `percent` | 是：`no formatter named 'x'` |
| `data-repeat` 只有单层变量 | `row in rows`；嵌套要靠内层再写一个 `data-repeat` | 是（写法不对时） |
| 没有事件、没有冒泡 | 只有 `data-on-click` 与 `data-on-key` | 不 |
| 动作抛异常 | 异常被吞进诊断，那一帧不崩 | 是：`the action 'x' failed: …` |
| `tabindex` 的数值不排序 | 只表示"可到达"；顺序永远是文档顺序 | 不 |
| 没有 `data-on-change` | 控件的状态读 `value`/`checked` 属性，或在动作里读 | 不 |

## 七、平台与运行时

| 限制 | 说明 |
| --- | --- |
| 只有 1.7.10 / Forge 客户端 | 没有服务端渲染、没有 HUD 覆盖层、没有世界空间面板。本模组只在客户端做事：装到服务端的副本什么也不初始化，也不会因此要求进服客户端安装本模组 |
| 没有联网加载 | 远程 `url()` 与 `<link>` 一律拒绝并上报 |
| CJK 输入法 | **未验证**：`McScreen` 走的是键盘按键路径，IME 的组合输入没有测过 |
| OptiFine / GTNH 等大改包 | **未验证**；用的是 Forge 事件与自己的 `GuiScreen`，没有 patch 原版内部 |
| 无头字体 | 无头测试用的是测试测量器（每字符半个字号），字形光栅化只在预览与游戏里发生 |
| 没有构建产物缓存 | 干净克隆第一次构建需要能访问网络（Forge/Gradle），之后 `--offline` 可用 |

## 八、这一页怎么维护

这份清单不是"我们不想做"的清单，而是"现在还没有"的清单，它随每个增量变短：

- **做掉了**：条目从表里删掉，`CHANGELOG.md` 里会写着它是哪个版本做掉的；
- **新发现的**：先加进这里（哪怕是"未验证"这种程度的诚实），再去修；
- **刻意的**：留在 [01 总览](01-overview.md) 的"不是什么"表里，并写清理由。

## 相关

- 属性与诊断的完整对照：[04 CSS 参考](04-css.md)
- 调试方法与诊断在哪看：[13 调试](13-debugging.md)
- 之后的路线与新版本改了什么：`CHANGELOG.md`
