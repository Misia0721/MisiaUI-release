# Misia UI 使用文档

Misia UI（MUI）让你用 **HTML 描述界面、用 CSS 描述外观、用 Java 描述数据与行为**，
把它画进 Minecraft 1.7.10 的界面里。没有内嵌浏览器，没有 JavaScript：HTML 由本项目的解析器读，
CSS 由它自己的层叠流程算，布局由它的布局引擎排，结果是**一串绘制指令**，交给游戏自己的渲染管线。

这套文档面向**使用 MUI 写界面的 mod 作者**。它按主题分篇，每篇自成一体，可以按需跳读。

> 想读引擎**为什么这么写**（坐标系、字形落地、滚动条的取舍、每个决定的反面）请看项目根目录的
> `README.md`，那是设计笔记；本套文档讲**怎么用**。

## 篇目

| 篇 | 内容 | 什么时候看 |
| --- | --- | --- |
| [01 总览](01-overview.md) | MUI 是什么、不是什么、一帧里发生了什么、目录与包结构 | 第一次接触 |
| [02 快速上手](02-getting-started.md) | 从空项目到"点了有反应"的屏幕，七步 | 想马上跑起来 |
| [03 HTML 参考](03-html.md) | 标签、属性、解析器的容错规则、`<link>`/`<style>` | 写结构时 |
| [04 CSS 参考](04-css.md) | 选择器、伪类、单位、变量、`@media`、属性总表（实现 / 上报） | 写样式时 |
| [05 布局与排版](05-layout.md) | 盒模型、block/flex/inline-block、定位、溢出与滚动、文本 | 盒子位置不对时 |
| [06 组件](06-components.md) | `theme/components.css` 里每个类，以及怎么覆盖它 | 拼界面时 |
| [07 表单控件](07-forms.md) | `input` 各 `type`、`textarea`、`select`、单选组与键盘 | 做设置界面时 |
| [08 数据绑定](08-binding.md) | `${…}`、格式化器、`data-repeat`、`data-if`、`ValueSource` | 界面要跟着数据变 |
| [09 动作](09-actions.md) | `data-on-click`、`data-on-key`、`Mui.screen().on()`、异常与重绑定 | 要能点 |
| [10 资源与热重载](10-resources.md) | 三层资源、路径解析、`@import`、改文件即生效、贴图 | 调样式时 |
| [11 Java API](11-java-api.md) | `Mui`/`MuiScreen`/`McScreen`，以及不用游戏自驱引擎 | 接进自己的 mod |
| [12 主题](12-theming.md) | 自定义属性、覆盖组件表、GUI scale、当主题包发 | 换皮 |
| [13 调试](13-debugging.md) | 诊断列表、帧耗时、离线预览、无头测试、游戏内自检截图 | 出了问题时 |
| [14 已知限制](14-limitations.md) | 未实现的 CSS 与功能，以及每条会打印什么 | 遇到"没反应"时 |

## 三条贯穿全书的约定

**一、做不到就说出来，不会静默忽略。** 页面里任何 MUI 做不到的东西都会写进这份页面的**诊断**
（诊断是个字符串列表，见 [13 调试](13-debugging.md)）：无法识别的属性、无法读懂的值、
`<link>` 指向不存在的样式表、点了却没有动作可跑、动作抛了异常。一条被吞掉的属性意味着一个
看不见的 bug，所以这里一条都不吞。

**二、状态住在文档里。** `checked`、`value`、`selected` 都是元素上的属性；引擎写它们，
页面、绑定和样式表都读它们。没有一份平行的、会和文档走散的内部副本。

**三、没有 JavaScript。** 页面是标记、样式加 Java 数据。绑定层
（[08 数据绑定](08-binding.md)）覆盖界面真正需要的场景；需要计算的东西写在 Java 里。

## 最短的一段可运行代码

```html
<!-- assets/misiaui/screens/furnace.html -->
<link rel="stylesheet" href="../theme/components.css">
<div class="mui-panel">
  <div class="mui-title">${title}</div>
  <div class="mui-row">
    <div class="mui-progress"><div class="mui-progress-fill" style="width: ${percent}%"></div></div>
    <button class="mui-button mui-button-primary" data-on-click="smelt">Smelt</button>
  </div>
</div>
```

```java
public final class FurnaceModel {
    public String getTitle() { return "Furnace"; }
    public int getPercent() { return 62; }
}

Mui.screen("screens/furnace.html")
    .data(new FurnaceModel())              // ${title}、${percent}
    .on("smelt", source -> smelt())        // data-on-click="smelt"
    .open();
```

接下来读 [02 快速上手](02-getting-started.md) 把这段放进你自己的项目。
