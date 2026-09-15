# 10 资源与热重载

页面、样式表、贴图从哪来，以及**改了文件怎么马上看到**。

## 一、三层资源，从上到下

```
.minecraft/misiaui/                                ← 最高：本地目录，改文件立刻生效
.minecraft/resourcepacks/<你的包>/assets/misiaui/  ← 中间：玩家的资源包（主题这么发）
<jar>/assets/misiaui/                              ← 最低：mod 自己的 jar，一定在
```

优先级由 `MuiResources` 组装的顺序决定（`local directory > resource packs > mod jar`），
启动时会在日志里打印这一行：

```
MUI resource layers: local directory > resource packs > mod jar
local page directory: G:\AUI-snow\MisiaUI\minecraft\.\misiaui
```

三层**读取规则不同**，这是整个资源模型的要点：

| 资源 | 规则 | 为什么 |
| --- | --- | --- |
| **页面**（HTML） | 取**第一个**有它的层（整体替换） | 两个 HTML 文件合并只会得到胡话——第二份不是第一份的覆盖，而是另一份文档 |
| **样式表**（CSS） | 从**每一层**都读，按**低优先级在前**合并 | 主题才可以"补丁式"存在：资源包里一份三行的 `theme/demo.css` 只改一个按钮的颜色，层叠把它和 jar 里的合并，而不是整份替换 |

因为层叠靠**顺序**决定平局，样式表合并后的顺序是"搜索顺序的**逆序**"——后应用的必须来自高优先级层。

## 二、路径解析

页面里写的都是**逻辑路径**：相对于 bundle 根、`/` 分隔、没有前导斜杠。

```
screens/furnace.html      theme/furnace.css
```

- `<link href>` 与 `@import` 都**相对当前文档/当前样式表**解析，`../` 可以；
- **不能爬出资源根**：`../../secret.html` 会被拒绝并上报
  （`points outside the bundle root and was ignored`）——安静地把它夹回根目录，
  等于让一份文档能读到 bundle 里的任何东西；
- **不联网**：`https://…` 会被拒绝并上报（`is remote; MUI does not fetch resources`）。
  一个会安静联网取样式表的界面既是启动卡顿也是隐私问题；
- Minecraft 那种 `域:路径` 形式会**单独**上报"这个 bundle 只有一个域"
  （`names the bundle 'minecraft'; …`），因为这里并没有去取什么东西；
- 找不到就报清楚是谁在找：
  ```
  no resource layer has the stylesheet 'theme/furnace.css', referenced by screens/furnace.html
  ```

单个资源的大小上限是 **4 MB**（页面与样式表），超了会上报而不是吃内存。

## 三、热重载：改文件，下一帧就变

屏幕每帧只比较**一个数字**：`PageResources.revision()`。任何一层变了，这个数字就动，
下一帧重新装载并重建。

| 你做了什么 | 什么时候生效 |
| --- | --- |
| 改 `.minecraft/misiaui/` 里的文件 | 最多**四分之一秒**后（目录层每 250ms 扫描一次大小/mtime；调样式时可以传 `scanIntervalMs = 0` 让它每次都扫） |
| 按 `F3+T` | 立刻（资源包层注册成了游戏的 reload listener） |
| 切换资源包 | 立刻（同上） |
| 改 jar 里的文件 | 需要重新构建、重启客户端 |

所以调样式的循环是：把页面和样式表放在 `.minecraft/misiaui/screens/`、`…/theme/`，
改、保存、看。**不需要重启游戏，也不需要重新构建 mod。**

日志里会说明当前页面是从哪一层来的，排查"我改的怎么没生效"第一眼就看它：

```
MUI loaded screens/settings.html from local directory with 2 stylesheet(s)
```

## 四、`<link>`、`<style>`、`@import`

| 写法 | 处理 |
| --- | --- |
| `<link rel="stylesheet" href="…">` | `rel` 里**含有** `stylesheet` 就算（按空格切分，`rel="stylesheet preload"` 也算） |
| `<style>…</style>` | 每次重建都会重新解析（帧里唯一读文本的部分，耗时单独计时） |
| `@import "x.css";` / `@import url(x.css);` | 相对**引入它的样式表**解析，排在被引入者**之前**；缺失/越界/成环都上报 |

引用顺序就是层叠顺序：文档里 `link` 的先后、以及同一路径在各层的合并顺序（见上）。

## 五、贴图

```css
.icon  { background-image: url(misiaui:textures/icon_ingot.png); width: 16px; height: 16px }
.bar   { background-image: url(wide.png); background-repeat: repeat-x }   /* 本 mod 的贴图可以省域名 */
```

- 贴图走**游戏自己的贴图管理器**（不是 MUI 的资源层）：贴图是"一张文件"，不是一个文档，
  它的域也未必是本 mod 的（`minecraft:textures/…` 也行）；
- 度量（固有尺寸）与真正绘制走同一个 `IResourceManager`，所以 `background-size: contain/cover`
  缩放的比例和游戏里画出来的一致；
- 贴图同样注册了 reload listener：换一张面板贴图后 `F3+T` 就能看到新的比例；
- **不联网**：`url(https://…)` 上报并跳过；
- 预览里读不到的贴图会画成**洋红色占位块**（见 [13 调试](13-debugging.md)），
  而不是安静地什么都不画。

## 六、把主题当资源包发

```
assets/misiaui/theme/demo.css          ← 只放你要改的几条规则
assets/misiaui/textures/panel.png      ← 想换的贴图
```

因为样式表是**按层合并**的，你不需要复制整份主题：写三行就能改一个按钮，
写一份 `components.css` 同名文件也能整份接管。

**热重载与资源包的边界**：资源包层是通过游戏的通知刷新的（`F3+T`、切换包），
而本地目录层是每次读都检查。所以调样式时用本地目录，**发布**时用资源包。

## 七、Java 侧

```java
// 用游戏配置好的那三层
LoadedPage page = MuiResources.load("screens/furnace.html");
PageResources layers = MuiResources.layers();
MisiaUI.LOG.info("{}", layers.describe());     // "local directory > resource packs > mod jar"

// 自己搭一套（离线预览、测试、或者把页面放在别处）
PageResources mine = PageResources.of(
        new DirectoryResourceSource(new File("my-pages"), "my pages"),
        ClasspathResourceSource.assets(MyMod.class, "my jar"));
LoadedPage page2 = PageLoader.load(mine, "screens/furnace.html");
```

`PageResources` 的层策略**平台无关**（`core/res`），所以"哪一层赢、顺序如何"可以完全无头测试，
不用游戏。三个平台实现（`DirectoryResourceSource` / `ResourceManagerSource` /
`ClasspathResourceSource`）都挂在 `spi.ResourceSource` 这个缝上：

```java
public interface ResourceSource {
    String name();                            // 出现在诊断与日志里，如 "local directory"
    List<String> readAllText(String path);    // 这一层里该路径的所有版本，低优先级在前
    String readText(String path);             // 默认实现：取最后一个（这一层里胜出的那份）
    boolean exists(String path);              // 默认实现：readAllText 非空
    long revision();                          // 这一层内容变了就换个数字（热重载的整个机制）
    List<String> problems();                  // 这一层自己的问题，进页面诊断
}
```

自己写一个 `ResourceSource`（从网络、从压缩包、从内存里取）就能插进这套模型——
只要它愿意在 `problems()` 里说话（读不到东西时报空表，**原因**写进 `problems()`，
而不是记到某个日志里）。

上层 `PageResources` 提供的是按路径取用的两个入口：

```java
ResourceText page   = layers.find("screens/furnace.html");   // 文档：第一个有的层赢
List<ResourceText> all = layers.readAll("theme/demo.css");   // 样式表：所有层都要
long revision       = layers.revision();                     // 任何一层变了都会动
String describe     = layers.describe();                     // "local directory > resource packs > mod jar"
```


## 八、常见问题

| 现象 | 第一眼看哪 |
| --- | --- |
| 改了文件没生效 | 日志的 `MUI loaded … from <哪一层>`；本地目录优先于 jar |
| 样式只生效了一半 | 资源包里那份是不是**只写了差异**（它不会替换 jar 里那份，会合并） |
| 打包后失效、开发时正常 | 是不是用了 `域:路径` 引用了别的域的资源 |
| 贴图不显示 | 看诊断里的 `background-image:` / url 相关上报；预览里会画成品红块 |
| 页面更新了、样式表没更新 | `@import` 的路径是不是相对引入者解析的（写错了会上报"没有这一层有它"） |

## 相关

- 预览与自检怎么用：[13 调试](13-debugging.md)
- 主题怎么做：[12 主题](12-theming.md)
- 自己驱动引擎（不用游戏）：[11 Java API](11-java-api.md)
