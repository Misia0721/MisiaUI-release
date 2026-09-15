# 11 Java API

三层用法，按需要选：

1. **门面**：`Mui.screen(…).data(…).on(…).open()` —— 写界面的人只需要这一层；
2. **引擎**：装载页面 → 建 `DocumentPipeline` → 自己绘制，不需要游戏也能跑（离线预览就是这么做的）；
3. **缝（SPI）**：换掉文字测量、字形来源、贴图度量、资源来源。

## 一、门面：`api`

```java
Mui.open("screens/machine.html");                    // 一行版
Mui.open("screens/machine.html", machine);           // 一行版 + 数据
Mui.screen("screens/machine.html")                   // builder
   .data(machine)
   .on("smelt", element -> smelt(element.id()))
   .open();
Mui.current();                                       // 当前 MUI 屏幕，或 null
Mui.close();                                          // 关掉它
```

| 方法 | 说明 |
| --- | --- |
| `Mui.screen(String pagePath)` | 开始配置。路径是逻辑路径（`screens/x.html`），空路径抛 `IllegalArgumentException` |
| `Mui.open(String)` / `Mui.open(String, Object)` | `screen(...).open()` 的简写 |
| `Mui.current()` | 当前显示的 `McScreen`，玩家在看别的东西时是 `null` |
| `Mui.close()` | 关掉当前的 MUI 屏幕，返回是否关了 |

`MuiScreen`（builder）的完整表面见 [09 动作](09-actions.md) 第五节。

`api` 包**故意不在"平台无关"的集合里**（`ArchitectureGuardTest` 不扫它）：它是唯一知道
"游戏正在运行"的部分，用接口把这层藏起来只会给使用者多一层间接。`Mui` 上标了
`@SideOnly(Side.CLIENT)`。

## 二、引擎：`core` + SPI

脱离游戏跑整个引擎，是这套设计的核心承诺，也是离线预览的立足点：

```java
// 1. 装载（用游戏的三层资源，或自己搭一套）
LoadedPage page = MuiResources.load("screens/furnace.html");

// 2. 建流水线
DocumentPipeline pipeline = new DocumentPipeline(
        page.document(), page.styleSheets(), new AwtTextMeasurer());

// 3. 可选的部件
pipeline.setInteraction(new InteractionState());   // hover / 焦点 / 点击
pipeline.setImageMeasurer(imageMeasurer);          // 让 background-size: contain 知道图多大
pipeline.setScroll(new ScrollState());             // 记住滚动位置
pipeline.setData(new BoundDocument(template, source));  // 绑定数据
pipeline.setTimeSource(myClock);                   // 过渡/动画的时钟（默认墙钟）
pipeline.setRootFontSize(16f);                     // rem 的基准

// 4. 画一帧
PaintList list = pipeline.paint(320f, 240f);

// 5. 问它刚才花了多久 / 有什么问题
FrameProfile profile = pipeline.profile();
List<String> problems = pipeline.diagnostics();
```

`DocumentPipeline` 的公开表面（全都平台无关）：

| 类别 | 方法 |
| --- | --- |
| 尺寸与内容 | `paint(w, h)`、`update(w, h[, now])`、`invalidate()`、`viewportWidth()`、`viewportHeight()`、`document()`、`sheets()` |
| 数据 | `setData(BoundDocument)`、`data()` |
| 交互 | `setInteraction(InteractionState)`、`interaction()`、`movePointer(x, y)`、`press()`、`release()`、`elementAt(x, y)`、`keyPressed(name)`、`typeText(char)`、`setModifiers(shift, control)`、`shiftHeld()`、`controlHeld()`、`focusStep(direction)` |
| 滚动 | `setScroll(ScrollState)`、`scroll()`、`scrollAt(x, y, dx, dy)` |
| 动效 | `setTimeSource(LongSupplier)`、`transitions()`、`animations()` |
| 文字与图片 | `setRootFontSize(float)`、`setImageMeasurer(ImageMeasurer)`、`imageMeasurer()` |
| 度量与诊断 | `profile()`、`diagnostics()`、`layoutTree()`、`paintList()`、`form()` |

一个最小的宿主循环长这样：

```java
pipeline.movePointer(mouseX, mouseY);     // 每帧告诉它指针在哪
if (clicked) { pipeline.press(); }
if (released) { pipeline.release(); }     // 松手在同一个元素上才算点击
PaintList list = pipeline.paint(width, height);
myRenderer.draw(list);
```

## 三、缝：`spi`

| 接口 | 谁实现 | 做什么 |
| --- | --- | --- |
| `TextMeasurer` | `AwtTextMeasurer` | 量字符串宽度、字体的 ascent/descent —— **布局只依赖它** |
| `GlyphSource` | `AwtGlyphSource`（+ `GlyphAtlas`） | 按 `FontSpec` 与码点给出字形位图与其图集页 |
| `ImageMeasurer` | `McImageMeasurer` / `ClasspathImageMeasurer` | 贴图的固有尺寸（`contain`/`cover`/九宫格靠它） |
| `ResourceSource` | 三个层实现 | 资源的读取与 revision（[10 资源](10-resources.md)） |

```java
public interface TextMeasurer {
    float measureWidth(String text, FontSpec font);
    float ascent(FontSpec font);
    float descent(FontSpec font);
}
```

**为什么测量是缝而不是引擎内部的事**：字体查找与光栅化要落到操作系统上，把它放进引擎会让布局
依赖于"页面恰好跑在哪台机器上"——同一份文档在两台电脑上量出不同的宽度。把 `java.awt` 挡在平台层，
引擎的几何只依赖于"别人交给它的测量结果"，这就是它能在测试里用"每个字符半个字号"的假测量器
把断言写成精确值的原因。

## 四、客户端接线（`client`）

| 类 | 作用 |
| --- | --- |
| `MuiClientBootstrap` | 启动时自检：引擎类在不在、资源层能不能解析、字体引擎能不能起来，并把结果打进日志 |
| `MuiScreens` | `Mui` 底下真正开屏的地方：决定资源层、字形图集、GUI scale、测量器 |
| `MuiKeyBindings` | 注册 `M`（`key.misiaui.open_demo`）打开演示页；走 FML 的 `ClientRegistry`，所以出现在原版按键设置里、可以改键 |
| `MuiSelfTest` | 游戏内自检（[13 调试](13-debugging.md)）：开页面、驱动输入、读回帧缓冲、退出 |

启动日志里的几行值得记住（排查时它们回答了"到底用的是哪一份资源"）：

```
Misia UI 0.1.0-beta starting up
engine self-check passed: root=<html> text="Misia UI ready"
MUI resource layers: local directory > resource packs > mod jar
font engine ready: 512x512 atlas, glyphs at 2.0 texels per pixel
default page: screens/demo.html from resource packs with 2 stylesheet(s)
```

## 五、用 MUI 渲染到别处（不继承 `GuiScreen`）

`McScreen` 只是**一个**宿主。任何持有 `PaintList` 的地方都能画它——
前提是那个渲染器愿意接受"填充矩形、带纹理矩形、文字、裁剪、变换"这几条指令。
`McPainter` 是 1.7.10 的实现（Tessellator + 字形图集 + `glScissor`），
离线预览里的 `PreviewRenderer` 是 Java2D 的实现：**两者吃同一份绘制列表**，这就是
"图看着不对就是引擎不对"这句话的依据。

```java
PaintList list = pipeline.paint(320f, 240f);
for (PaintOp op : list.ops()) {          // FILL_RECT / MESH / GRADIENT / IMAGE / TEXT / CLIP_*
    switch (op.kind()) { … }
}
```

## 六、几条边界

| 事实 | 含义 |
| --- | --- |
| `core`/`spi` 不引用 Minecraft、Forge、LWJGL、`java.awt` | 由 `ArchitectureGuardTest` 扫描源码强制；违反则构建失败 |
| `api` 在守卫之外 | 它是"知道游戏在跑"的那一层，见上 |
| 每个声明的属性都必须被读到 | `ArchitectureGuardTest` 断言：新增一个 `PropertyId` 却没人读，构建失败（或必须列进"已知缺口/做不到"表） |
| 门面之外一切都是公开的 | 一个只想自己驱动引擎的宿主可以完全不用 `api` |

## 相关

- 动作与重绑定：[09 动作](09-actions.md)
- 资源层与热重载：[10 资源](10-resources.md)
- 主题与 GUI scale：[12 主题](12-theming.md)
- 调试工具：[13 调试](13-debugging.md)
