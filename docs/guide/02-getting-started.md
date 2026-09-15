# 02 快速上手

从空项目到一个"点了有反应"的屏幕，七步。每一步都有一句"怎么知道它成了"。

---

## 第 1 步：把页面放在 MUI 找得到的地方

页面是 mod 自己命名空间下的一个**逻辑路径**。文件

```
src/main/resources/assets/misiaui/screens/furnace.html
```

打开时写作

```java
Mui.open("screens/furnace.html");
```

（`assets/misiaui/` 是自动加上的，就像 Minecraft 的其他资源一样。）

样式表按 HTML 的规矩**相对当前文档**引用：

```html
<link rel="stylesheet" href="../theme/furnace.css">
<!-- 也支持放在同目录的：href="furnace.css" -->
```

**怎么知道它成了**：路径写错时，页面本身照开，诊断里会有一行
`no resource layer has the page 'screens/furnace.html'`；样式表写错时是
`no resource layer has the stylesheet 'theme/furnace.css', referenced by screens/furnace.html`。
诊断在哪看：[13 调试](13-debugging.md)。

---

## 第 2 步：写标记和样式

不需要注册任何东西，没有元素类型要声明，没有引擎 API 要调用：

```html
<link rel="stylesheet" href="../theme/components.css">
<div class="mui-panel">
  <div class="mui-title">${title}</div>
  <div class="mui-row">
    <div class="mui-progress"><div class="mui-progress-fill" style="width: ${percent}%"></div></div>
    <button class="mui-button mui-button-primary" data-on-click="smelt">Smelt</button>
  </div>
</div>
```

`theme/components.css` 是 MUI 随 jar 提供的一套类（面板、按钮、标签页、列表、进度条……），
全部由普通 CSS 写成，你用自己的规则覆盖任意一条即可。逐个类见 [06 组件](06-components.md)。

**怎么知道它成了**：`gradlew muiPreview` 会把这套页面渲染成 PNG，不开游戏就能看图。

---

## 第 3 步：准备数据

`${…}` 从 Java 对象、Map 或任何实现了 `ValueSource` 的东西上读值。**普通 getter 就够了**，
不需要注解、注册或接口：

```java
public final class FurnaceModel {
    private int energy = 6200;
    private int capacity = 10000;

    public String getTitle()   { return "Furnace"; }
    public int getPercent()    { return energy * 100 / capacity; }
    public List<Item> getItems() { return items; }
}
```

`${title}` 找 `getTitle()`；`${item.name}` 先取第一段再往值上取属性；
`Map` 的键名同样可用。数据要能随着界面变化——那是绑定层的全部内容，
见 [08 数据绑定](08-binding.md)。

**怎么知道它成了**：名字写错时渲染成空，并打印
`nothing provides 'title'; 'title' rendered as empty`。

---

## 第 4 步：一句话开屏

```java
Mui.screen("screens/furnace.html")
    .data(model)                              // ${title}、${percent}、data-repeat、data-if
    .on("smelt", source -> doTheSmelting())   // data-on-click="smelt"
    .open();
```

这就是全部。开屏时由 MUI 决定的：资源层、字形图集、GUI scale、交互状态、动作之后的重绑定。

三个可选的补充：

```java
// 给数据根起名，页面里写 ${machine.energy}
Mui.screen("screens/machine.html").data("machine", machine).open();

// 一个动作只读、不改变数据，就不要为它重建页面（hover 与焦点会被保住）
Mui.screen("screens/log.html").on("close", s -> log()).rebindAfterActions(false).open();

// 数据在别处变了（tick、网络线程）时，主动推一下
builder.rebind();
```

### 动作名没有注册会怎样

不会静默失效。诊断里会有一行，写清是哪个元素、什么动作、当前注册了哪些：

```
nothing handles the action 'smelt', declared by <button> (clicked); registered: [close, tab]
```

动作**抛异常**时也一样：异常被抓住、写进诊断，那一帧不会崩，
点击算作"已被处理"，不会被透传给游戏。详见 [09 动作](09-actions.md)。

**怎么知道它成了**：按钮点了有反应，而且诊断里没有 `nothing handles the action`。

---

## 第 5 步（可选）：不开游戏也能跑整个引擎

`Mui` 只是一个门面。它下面的一切都是公开且平台无关的：装载页面、在文档与样式表上建
`DocumentPipeline`、喂进去一个 `TextMeasurer`。

```java
LoadedPage page = MuiResources.load("screens/furnace.html");
DocumentPipeline pipeline = new DocumentPipeline(page.document(), page.styleSheets(), measurer);
pipeline.setInteraction(new InteractionState());
pipeline.paint(320, 240);              // 画一帧
pipeline.profile();                    // 这一帧各阶段花了多少
```

离线预览就是这么做的，也是"页面可以脱离游戏被测试"的原因。见 [11 Java API](11-java-api.md)。

---

## 第 6 步：改文件不用重新构建

把页面丢进 `.minecraft/misiaui/screens/`，它会**盖过 jar 里那份**；屏幕开着的时候改文件，
下一帧就生效。`PageResources.revision()` 每帧比较一次，所以不需要重启，也不需要敲命令。

```
.minecraft/misiaui/screens/furnace.html     ← 优先级最高，改这里
.minecraft/misiaui/theme/furnace.css
.minecraft/resourcepacks/<你的主题包>/assets/misiaui/...   ← 优先级中等
<jar>/assets/misiaui/...                    ← 最低，且一定在
```

资源包层注册成了游戏的 reload listener，所以 `F3+T` 和切换资源包都会自己触发重载。
详细规则（路径解析、`@import`、不允许联网）见 [10 资源与热重载](10-resources.md)。

**怎么知道它成了**：编辑文件、保存、看屏幕——样式在下一帧就变了。

---

## 第 7 步：先看图，再进游戏

```powershell
# 跑全部无头测试（快，不需要游戏）
.\scripts\gradle.bat muiTest

# 把内置页面与测试页渲染成 PNG，旁边附绘制列表与盒树
.\scripts\gradle.bat muiPreview

# 打 jar
.\scripts\gradle.bat build
```

`build/preview/` 里每一张图旁边都有两个文本文件：`<名字>.txt` 是绘制列表，
`<名字>.layout.txt` 是盒树。图片告诉你**有东西不对**，只有这两个列表能告诉你**为什么**。

界面里任何牵涉渲染器或输入的东西，最后都要在真客户端里拍一张：

```powershell
.\scripts\gradle.bat runClient -PmuiSelfTest=build/selftest/furnace.png `
  -PmuiSelfTestPage=screens/furnace.html -PmuiSelfTestFrames=3
```

**游戏内的截图才是结论。** 离线预览是预测，不是答案。参数全集见 [13 调试](13-debugging.md)。

---

## 第一次做页面时的检查清单

- [ ] 页面路径是 `screens/xxx.html`（不要写 `assets/misiaui/`，也不要写盘符路径）
- [ ] `<link>` 的 `href` 是相对当前文档的，且样式表真的存在
- [ ] 每个 `data-on-click` 的名字都 `.on(...)` 注册过
- [ ] 每个 `${…}` 都能在数据源上找到（Map 要真有这个键）
- [ ] 滚动容器有 `id`（否则滚动位置活不过一次重建）
- [ ] 诊断列表是空的，或者每一行你都认得
- [ ] 预览图看过，游戏内截图也看过

## 常见的第一批故障

| 现象 | 诊断里会写 | 原因 |
| --- | --- | --- |
| 整页空白 | `no resource layer has the page '…'` | 路径写错 |
| 有文字没样式 | `no resource layer has the stylesheet '…'` | `href` 写错或文件不在 |
| 某些属性没效果 | `unsupported property 'x'` / 读不懂的值 | 见 [04 CSS 参考](04-css.md)与 [14 已知限制](14-limitations.md) |
| 按钮点不动 | `nothing handles the action 'x'` | 忘了 `.on("x", …)` |
| 值显示成空 | `nothing provides 'x'` | 名字拼错，或数据源没有这个键 |
| 数字排版怪 | `no formatter named 'x'` | 格式化器名字错，见 [08 数据绑定](08-binding.md) |
| 滚动位置老是弹回 | `a scrollable <div> has no id` | 给滚动容器加 `id` |
| 样式表从 CDN 引的 | `'https://…' is remote; MUI does not fetch resources` | MUI 不联网 |

## 接下来

- 写结构：[03 HTML 参考](03-html.md)
- 写样式：[04 CSS 参考](04-css.md)
- 位置不对：[05 布局与排版](05-layout.md)
- 要能点：[09 动作](09-actions.md)
