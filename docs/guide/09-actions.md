# 09 动作

界面"能点"的那一半。MUI 里动作是**有名字的**：页面写 `data-on-click="smelt"`，
宿主注册一个叫 `smelt` 的处理函数。标记保持声明式，Java 侧保持有类型，
而一个动作可以**在不改标记的情况下**重新指向另一个处理函数。

```html
<button class="mui-button" data-on-click="smelt">Smelt</button>
```

```java
Mui.screen("screens/furnace.html")
    .data(model)
    .on("smelt", source -> doTheSmelting())
    .open();
```

## 一、`data-on-click`

- 写在**任何**元素上；
- 点击落在子元素上时会**沿祖先向上找最近的一个** `data-on-click`，所以按钮里的文字、
  图标不必各自重复这个属性；
- 处理函数拿到的是**声明了这个动作的元素**（不一定是指针下的那个），所以一组控件可以共用
  一个处理函数，靠 `id` 分辨是谁：

```java
.on("tab", source -> LOG.info("tab '{}' selected", source.id()))
```

```html
<div class="mui-tabs" data-roving="horizontal">
  <button id="tab-craft" class="mui-tab" data-on-click="tab" checked>Craft</button>
  <button id="tab-smelt" class="mui-tab" data-on-click="tab">Smelt</button>
</div>
```

## 二、`data-on-key`

`键:动作` 的**空格分隔列表**——一个元素通常要对多个键负责，而另一种做法是每个键一个属性：

```html
<div class="mui-panel" data-on-key="Enter:submit Escape:cancel R:reload">
```

- 键名**大小写不敏感**；
- 与点击一样会**沿祖先向上找**：按键送到有焦点的元素，所以焦点在标签/子控件上也能生效；
- **只有被点名的键会被吃掉**。页面没写的键照常交给游戏——这是"键盘层"和"快捷键表"的区别，
  也是一个有焦点的面板不该吞掉玩家的 `J` 的原因；
- 写错的项会被上报，而不是默默忽略（一个以为自己有快捷键的页面，最难查）：
  ```
  'Entr' in data-on-key is not key:action; it was ignored
  ```

### 键名表

页面写的名字**跟随浏览器的叫法**（写 `data-on-key` 的人脑子里就是这套词汇），
没有浏览器名字的键用本项目自创的名字，所以没有一个键是无名的：

| 类别 | 名字 |
| --- | --- |
| 字母 / 数字 | `A`…`Z`、`0`…`9`（可打印字符按**字符本身大写**命名，`r` 与 `R` 同名） |
| 功能键 | `F1`…`F12` |
| 编辑 | `Enter`（`Return` 同义）、`NumpadEnter`、`Space`、`Escape`（`Esc`）、`Tab`、`Backspace`、`Delete`（`Del`） |
| 方向 | `ArrowUp`/`Up`、`ArrowDown`/`Down`、`ArrowLeft`/`Left`、`ArrowRight`/`Right` |
| 定位 | `Home`、`End`、`PageUp`、`PageDown` |

组合键（`Ctrl`/`Shift`/`Alt`）**不在键名里**。引擎把"按住的是哪个修饰键"作为状态交给页面
（`Shift` 与 `Ctrl` 在文本与选区里已经这样用了），需要"`Ctrl+S` 才保存"时，
在动作里问一次流水线：

```java
.on("save", element -> {
    if (Mui.current().pipeline().controlHeld()) { save(); }
})
```

## 三、没注册的动作与抛异常的动作

两条都不会静默：

```
nothing handles the action 'smelt', declared by <button> (clicked); registered: [close, tab]
the action 'smelt' failed: java.lang.NullPointerException: machine
```

- **没注册**：上报，这次点击**什么也不发生**（页面自己声明了一个它没有的动作）；
- **抛异常**：异常被抓住、写进诊断，那一帧不会崩，并且**算已处理**（动作被"到达"了，
  再透传给游戏就成了页面没要求过的行为）。
  捕获的是 `Throwable` 而不是 `Exception`：一个缺依赖的 mod 抛出的 `NoClassDefFoundError`
  正是发生在这一刻，不能让它带走这一帧。

这是刻意的取舍：**动作是调用者的代码在渲染循环里跑**，出事时只应损失这一次点击。

> 注意"什么也不发生"与"交给游戏"的区别：**鼠标点击不会透传给 `GuiScreen`**。
> 一个 MUI 屏幕只要开着（并且带交互状态），点击就是页面的：命中就按上面走，
> 没命中动作就什么都不做——这正是"点空白处取消焦点"能工作的原因。
> 只有**非左键**会被直接忽略（也不交给游戏）。
> 键盘则相反：引擎没要的键**会**落到游戏手里（见下表）。

## 四、动作之后的重绑定

`.on(…)` 注册的动作跑完，`MuiScreen` 会**自动重新绑定数据**（`rebindAfterActions`，默认开）——
因为动作几乎总是"模型变了"的那一刻，而这一行本来每个页面都得写。

```java
// 只读的动作（关闭、打日志）不想重建页面：关掉它，hover 与焦点就都保住了
Mui.screen("screens/log.html").on("close", s -> log()).rebindAfterActions(false).open();
```

要记住的后果（[07 表单控件](07-forms.md)、[08 数据绑定](08-binding.md) 都提过）：
重新绑定是"页面在说控件该是什么值"，所以**玩家刚输入的内容会被数据覆盖**。
一个"保存"动作应当先把控件值读回模型：

```java
.on("save", source -> model.setName(
        Mui.current().pipeline().document().getElementById("name").attribute("value")))
```

## 五、`Mui` 的完整表面

```java
Mui.screen("screens/x.html")   // 开始配置一个屏幕（builder）
   .data(model)                // 或 .data("root", model) / .data(valueSource)
   .on("name", element -> …)   // 注册动作；可写多次
   .rebindAfterActions(true)   // 默认 true
   .open();                    // 装载、开屏；没有客户端时返回 null
```

```java
Mui.open("screens/x.html");            // 一行版：不绑数据、不注册动作
Mui.open("screens/x.html", model);     // 一行版：绑一个对象
Mui.current();                         // 当前显示的 MUI 屏幕，或 null
Mui.close();                           // 关掉它，返回是否关了
```

| 方法 | 说明 |
| --- | --- |
| `Mui.screen(path)` | 返回 `MuiScreen`（builder）。**`open()` 之前不加载任何东西**，所以配置过程可测 |
| `.data(Object)` | 绑定对象，页面写 `${energy}` |
| `.data(String root, Object)` | 绑定并起名，页面写 `${machine.energy}` |
| `.data(ValueSource)` | 用自己的数据源 |
| `.on(name, action)` | 注册动作；`action` 收到声明该动作的 `Element` |
| `.rebindAfterActions(boolean)` | 关掉"动作后重绑定" |
| `.rebind()` | 数据在别处变了时手动推一下（对 `ValueSource` 由数据源自己负责） |
| `.interaction()` | 拿到交互状态，直接驱动 hover/focus/Tab |
| `.pagePath()` / `.screen()` | 页面路径 / 已打开的屏幕 |
| `.open()` | 开屏；返回 `McScreen`（无客户端时 `null`，不抛异常——少一个屏幕不值得崩溃报告） |

`open()` 之后能做的（`McScreen`）：`setPausesGame(boolean)`、`setPointerOverride(x, y)`、
`setModifierOverride(shift, control)`、`pipeline()`、`painter()`、`close()`（继承自 `GuiScreen`）。

## 六、动作能拿到什么

处理函数只有一个参数：**声明了这个动作的元素**。它能做的事：

| 想做的事 | 怎么做 |
| --- | --- |
| 分辨是哪一个控件 | `source.id()` / `source.attribute("data-…")` |
| 读控件当前值 | `pipeline().document().getElementById(id).attribute("value")` |
| 改页面里的值 | 改**模型**然后让引擎重绑定（推荐），或直接 `setAttribute` 后 `invalidate()` |
| 开另一个页面 | `Mui.screen("screens/other.html").open()` |
| 关闭当前页面 | `Mui.close()` |
| 让某个区域重新绑定 | `builder.rebind()` |

## 七、一次点击的完整路径

理解这条路径能省掉很多猜测：

1. **命中测试**：`pipeline.elementAt(x, y)` 按绘制顺序的**逆序**找最上面那个盒子；
2. **按下**：记录按下的是哪一个元素（`press()`），`:active` 从这里开始生效；
3. **松开**：只有指针**仍在按下时的那个元素上**才触发点击——拖出去再松手是取消，不是提交；
4. **找动作**：从命中元素沿祖先向上找最近的 `data-on-click`；
5. **聚焦**：找最近的 `data-on-click` / 表单控件 / `tabindex`，并记下"这次焦点不是键盘来的"
   （`:focus-visible` 因此为假）；
6. **跑动作**：没注册 → 上报且算未处理；抛异常 → 上报且算已处理；
7. **重绑定**：默认重跑绑定，页面因此更新。

滚动条拖拽是个例外：它在按下时记录了"这是拖滚动条"，松开时**不触发点击**
（`cancelPress()`）——因为按下的确实是控件，但这个手势不是在按它。

## 八、哪些东西会吃掉输入，哪些不会

| 输入 | 谁处理 |
| --- | --- |
| 左键点击 | 页面的：命中 → 按下 → 松手（仍在同一元素上）→ 找 `data-on-click` → 跑动作。**不交给游戏** |
| 其它鼠标键 | 忽略（也不交给游戏） |
| `Escape` | 页面先接手（`data-on-key="Escape:…"`）；**没人要就关闭这个屏幕** |
| `Tab` / `Shift+Tab` | 焦点顺序；有东西可聚焦就吃掉，没有就落给游戏 |
| `Ctrl+A` | 作为 `A` 送给页面（引擎自己决定 `Ctrl` 下的 `A` 是什么意思：全选） |
| 其它按键 | 有焦点的控件 → 该元素的 `data-on-key` → 可打印字符交给文本控件 → **否则交给游戏** |
| 滚轮 | 指针下最近的可滚动祖先；**游戏同样会收到**（MUI 不吞滚轮，热键栏照常工作） |

一句话总结这条分界线：**键盘是借的，鼠标是拿走的**。玩家的按键绑定不该因为一个面板开着就失效；
而鼠标落在界面上的那一下，界面必须自己负责（否则面板下面的世界会响应点击）。

## 相关

- 焦点、Tab 顺序与组：[07 表单控件](07-forms.md)
- 绑定与重绑定：[08 数据绑定](08-binding.md)
- Java API 全貌：[11 Java API](11-java-api.md)
