# 13 调试

MUI 的调试原则只有一条：**任何"没反应"都必须留下一句话**。这一页说明那些话在哪里、
怎么看，以及在必须看图时怎么把图画出来。

## 一、三层证据

| 层 | 命令 | 花多久 | 能证明什么 | 不能证明什么 |
| --- | --- | --- | --- | --- |
| 无头测试 | `gradlew muiTest` | 不到一秒 | 解析、层叠、布局、绘制列表；几何可以逐行断言 | 渲染器、输入、字体光栅化 |
| 离线预览 | `gradlew muiPreview` | 几秒 | 一张图 + 生成它的绘制列表与盒树 | 真渲染器（Tessellator、批次、贴图、`glScissor`） |
| 游戏内自检 | `gradlew runClient -PmuiSelfTest=…` | 十几秒 | 玩家真正会看到的东西 | 一切（它就是结论） |

**游戏内截图才是结论。** 离线预览是预测：它用 Java2D 重放同一份绘制列表，快，而且能在自己
掌控的时钟上取任意瞬间，但它不是答案。

## 二、第一件该看的东西：诊断

页面里任何做不到的事都会进一个字符串列表。五个地方能看到同一批内容，按你用哪种方式跑：

```java
pipeline.diagnostics();          // 引擎侧：层叠 + 布局 + 绘制 + 绑定 + 交互，去重后一份
pipeline.layoutTree().warnings(); // 只有布局阶段（含层叠的部分）的原文
page.warnings();                  // 装载报告：找不到的样式表、远程链接、样式表自身的解析问题
pipeline.data().warnings();       // 绑定：没有的值名、不存在的格式化器、写错的 data-repeat
pipeline.interaction().problems(); // 交互：点了没有动作、data-on-key 写法不对、动作抛了异常
```

| 你跑的方式 | 诊断出现在 |
| --- | --- |
| `muiPreview` | `build/preview/<名字>.layout.txt` 顶部的 `warning:` 行；绘制阶段的在 `<名字>.txt` |
| 游戏内 | 日志里的 `[Misia UI]: MUI: …`（`WARN`），每一条只在**第一次出现**时打印 |
| 你自己的宿主 | `DocumentPipeline.diagnostics()`，或上面那几个分阶段的访问器 |

游戏内那份由 `McScreen` 每帧比对一次，只在内容变化时打印，并且**只打印新增的行**：
一个页面的三个毛病变成四个时，日志只多一行。

### 一个可以拿来试的故障页

把下面这个文件丢进 `.minecraft/misiaui/screens/diagnostics.html`（本地目录优先级最高，
会盖过 jar），然后用自检打开它：

```html
<link rel="stylesheet" href="../theme/components.css">
<style>
  #panel { box-shadow: 0 2px 6px black; color: var(--not-defined); }
  .scroller { overflow-y: auto; max-height: 20px; }
</style>
<div id="panel" class="mui-panel" style="width: 160px">
  <div class="mui-title">Diagnostics</div>
  <div class="mui-row">
    <button class="mui-button mui-button-primary" data-on-click="ghost">Ghost</button>
    <button class="mui-button" data-on-click="real">Real</button>
  </div>
  <div class="scroller">
    <div class="mui-list-row">one</div>
    <div class="mui-list-row">two</div>
    <div class="mui-list-row">three</div>
  </div>
</div>
```

```powershell
.\scripts\gradle.bat runClient -PmuiSelfTest=build/selftest/diagnostics.png `
  -PmuiSelfTestPage=screens/diagnostics.html -PmuiSelfTestPointer=30,38 -PmuiSelfTestClick=true
```

日志里应当恰好出现这四行（本项目就是这样验证这条链路的）：

```
MUI: a scrollable <div> has no id, so its scroll position cannot survive a rebuild of the page; give it one
MUI: unresolved custom property: --not-defined
MUI: <style> #1: unsupported property 'box-shadow'; MUI does not implement it, so this declaration has no effect
MUI: nothing handles the action 'ghost', declared by <button> (clicked); registered: []
```

同时那张图**看起来是坏的**：列表文字是黑的，因为 `color` 落到了一个未解析的变量上。
图告诉你"有东西不对"，日志告诉你"为什么"——这正是两者都要留下的原因。

## 三、离线预览

```powershell
.\scripts\gradle.bat muiPreview
.\scripts\gradle.bat muiPreview -PmuiPreviewScale=3          # 放大三倍
.\scripts\gradle.bat muiPreview -PmuiPreviewOutput=build\shots
```

产物在 `build/preview/`：

| 文件 | 内容 |
| --- | --- |
| `<名字>.png` | 这一帧的图 |
| `<名字>.txt` | 变量、指针、交互状态、`warning:` 行，然后是**完整绘制列表**（每条指令的坐标与颜色） |
| `<名字>.layout.txt` | **盒树**（每个盒子的精确几何），以及布局与层叠的 `warning:` 行 |

内建的名字：`demo-*`、`settings`、`gallery*`、`text`、`shapes`、`textures`、`clamp`、`inline`、
`hidden`、`scroll`、`transition-{0,100,150,300}ms`、`animation-{0,250,500,900,1400}ms`、`forms-*`。

**一张图说"有东西在错的地方"，只有这两份列表说"为什么"。** 本项目里被图发现的 bug 有三个，
被列表发现的有更多，其中一个恰好相反：图看着对、列表里已经错了。

**贴图缺失会画成品红色方块。** 预览渲染器读不到一张贴图时（资源不在 classpath 上、
或者读的时候被环境挡住），它不画"什么都没有"，而是画一块洋红加白框的占位块，并在标准输出
留一行说明。之所以这样：曾经它只是什么都不画，于是**所有含贴图的预览图都少了贴图**
（demo 页的面板九宫格与物品图标、`textures.png` 的十个尺寸样例、`shapes.png` 的边框条），
而图看着"就是扁平盒子"，连续好几轮运行彼此一致、看不出异常。一片洋红是不可争辩的；
一片空白是可争辩的。（`PreviewRendererTest` 现在把这条规矩钉住了。）

绑定了数据的页面渲染的是**绑定之后**的树，所以预览同时也是一次对绑定的检查：没解析出来的
`${…}` 会以"面板上一个洞"的形式出现。

## 四、游戏内自检

```powershell
.\scripts\gradle.bat runClient -PmuiSelfTest=build\selftest\shot.png
```

自检会在标题界面打开页面，等游戏画完，把**帧缓冲读回来**，再退出客户端。参数：

| 参数 | 作用 |
| --- | --- |
| `-PmuiSelfTest=<png>` | 输出路径（同时写出 `<png>.txt`） |
| `-PmuiSelfTestPage=screens/x.html` | 拍别的页面（默认是 demo） |
| `-PmuiSelfTestPointer=x,y` | 把指针放在这里，并**跨帧保持** |
| `-PmuiSelfTestClick=true` | 在指针处点一下 |
| `-PmuiSelfTestWheel=<像素>` | 在指针处滚轮 |
| `-PmuiSelfTestDrag=<y>` | 把指针处的滚动条拖到 y |
| `-PmuiSelfTestKey=<名字>` | 按一个键（`Enter`、`Escape`、`A`…） |
| `-PmuiSelfTestModifier=shift\|ctrl\|ctrl+shift` | 按键/点击时按住修饰键 |
| `-PmuiSelfTestType=<文本>` | 逐字符输入到有焦点的控件 |
| `-PmuiSelfTestClockStep=<ms>` | 把流水线的时钟钉住并前进这么多毫秒（拍过渡/动画的中间态） |
| `-PmuiSelfTestFrames=<n>` | 先画 n 帧再截图 |
| `-PmuiSelfTestRebuild=true` | 每一帧都强制重建（量重建成本） |

`<png>.txt` 里有：帧缓冲尺寸、GUI scale、逻辑屏幕尺寸、指针与交互状态、**绘制列表**、
绘制后的 GL 状态、以及**两种方式量出来的字形图集**（Java 里的像素，和用 `glGetTexImage`
从 GPU 读回来的）。截图说"哪里不对"，这三组数据说"缝的哪一侧不对"。

**同一张页面，两半都拍一遍**，是"离线预测 vs 游戏结果"最便宜的对照。测试用的 fixture 不在 jar 里，
但放进本地层就会被优先加载，所以可以直接把它们拍进游戏：

```powershell
copy src\test\resources\preview\textures.html minecraft\misiaui\screens\textures.html
.\scripts\gradle.bat runClient -PmuiSelfTest=build\selftest\textures.png `
  -PmuiSelfTestPage=screens/textures.html
```

（`minecraft/` 是 gitignore 的沙盒目录，`runClient` 的工作目录就是它。）
两边的图如果对不上，问题在预览侧或在渲染器侧——**游戏内那张是结论**。

数量化的例子：

```powershell
# 过渡走到一半：按钮在 200ms linear 的中点，颜色应当是 #3f9e52 与 #4cb862 的中间
.\scripts\gradle.bat runClient -PmuiSelfTest=build\selftest\transition.png `
  -PmuiSelfTestPointer=100,81 -PmuiSelfTestClockStep=100
```

## 五、一帧花在哪里

```java
FrameProfile profile = pipeline.profile();
profile.frames();             // 总帧数
profile.rebuilds();           // 其中真正重建的
profile.skipped();            // 其中发现没事可做、直接复用的
profile.lines();              // 四阶段的 last / average / max / p95，与下面打印的一样
profile.last(Stage.LAYOUT);   // 单个阶段：last / average / max / percentile
```

`muiPreview` 与游戏内自检都会把它打印出来。四阶段是 **parse / cascade / layout / paint**，
另有 `unattributed` 与 `total`。静止的页面不重建，所以真正要看的是"重建那一帧"的数字：

```
stage          last      average        max          p95
parse         121.2µs   570.2µs      1902.7µs     1902.7µs
cascade       1799.7µs  6739.9µs     20407.4µs    20407.4µs
layout        1616.8µs  13285.0µs    46808.2µs    46808.2µs
paint         944.7µs   4815.9µs     16386.6µs    16386.6µs
```

静止时一小帧的成本是**零点几微秒**（每帧只比较几个版本号），所以"界面不动时不花时间"
不是估计，是能打印出来的数字。

## 六、量一张图

图片里"看起来偏了一点"通常可以变成一个数。做法是直接读 PNG：某一列哪些行有墨、某一点的灰度、
两张图差了多少像素。这几件事各自只要十几行代码，源码仓库里 `scripts/tools/` 下就有这样一套
脚本（**仅维护者使用，不随模组发布**，因为它们是开发工具而不是库的一部分）。列在这里是为了
说明思路与命令行形状，读者可以照此自己写一份：

```powershell
# 某一列墨迹的上下边界（文字基线问题最常用）
node tools/ink.mjs preview/text.png 120

# 放大一小块看像素（最近邻，不插值）
node tools/crop.mjs preview/gallery.png gallery-zoom.png 780 150 820 180 8

# 某一列像素的灰度
node tools/probe.mjs preview/shapes.png 40 100 140

# 两张图的差异：数量、最大通道差、差异所在的框
node tools/diff.mjs <旧图> <新图>
```

`diff.mjs` 是"这次改动有没有动到画面"的答案：**逐像素**告诉你没有。本项目用它证明过一次
"纯诊断改动"：整批预览图与上一轮留档逐像素一致。

## 七、留档的做法

一次预览/自检的结果只留在 `build/` 里，会被下一次运行覆盖，也会被 `gradle clean` 清掉，
所以每次看过之后都应当把它**归档**，而不是靠"文件还在那儿"当记录：

```
verification/<日期>/<时间>-<主题>/offline/     ← 离线预览的图与两份列表
verification/<日期>/<时间>-<主题>/ingame/      ← 游戏内自检的截图与诊断
```

两条纪律值得抄：

1. **离线与游戏内永远分开存**。一份混在一起的材料，会让后来的人（包括几个月后的自己）
   把预览图当成关于游戏的证据；
2. **归档目录放在 `build/` 之外**，否则 `gradle clean` 会连着记录一起带走。

归档动作本身在源码仓库里是一条命令（`scripts/archive-verification.mjs`，同样属于维护者工具）。
活下来的不是这些图，而是**每一次增量的一行记录**：提交、测试数、看过的预览图、实机截图证据，
以及由图片而不是由断言发现的那些 bug——这就是本项目的验证日志，它随源码一起保管，不在本仓库中。

你自己做界面时值得照搬的是同一条纪律：**每次改完留一行**（改了什么、跑了什么、看了哪张图、
结论是什么）。一周之后你唯一还能信任的，就是这行字，而不是"我记得试过"。

## 八、常见症状对照

| 症状 | 先看哪里 | 常见原因 |
| --- | --- | --- |
| 整页空白 | `page.warnings()` | 页面路径错、资源层没配 |
| 有字没样式 | `page.warnings()` | `<link href>` 错、样式表里只有未实现的属性 |
| 属性"没用" | `.layout.txt` 的 `warning:` | 属性未实现（上报）、值读不懂（上报）、选择器没匹配 |
| 文字/数字是黑的 | 找 `unresolved custom property` | `var()` 的名字没有声明 |
| 控件点不动 | 日志 `MUI:` / `interaction.problems()` | 动作没注册、`display: inline` 让控件没有盒子 |
| 滚动位置弹回 | 找 `has no id` | 滚动容器没有 `id` |
| 图看着对、游戏里不对 | 自检 `.txt` 的 GL/图集部分 | 只存在于真渲染器的行为（批次、贴图绑定、绕序） |
| 明明没变却每帧重建 | `profile().frames()` | 有东西还在动：无限动画、`data-repeat` 的数据 revision 每帧在动 |

## 相关

- 每个诊断的完整清单：[04 CSS 参考](04-css.md)、[14 已知限制](14-limitations.md)
- 资源与热重载：[10 资源](10-resources.md)
- 新版本改了什么：[CHANGELOG.md](../../CHANGELOG.md)
