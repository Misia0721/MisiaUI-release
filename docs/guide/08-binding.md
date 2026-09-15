# 08 数据绑定

绑定是 MUI 里唯一能让界面**跟着 Java 数据变**的东西。它只有三条指令和一个占位符语法，
合起来覆盖面板真正需要的场景：读数值、列表重复、条件出现。

```html
<div class="mui-panel">
  <div class="mui-title">${machine.name|upper}</div>
  <div class="mui-row"><span>Energy</span><span>${machine.energy|number} FE</span></div>
  <div class="mui-row" data-if="${machine.online}">Online</div>
  <div class="mui-list-row" data-repeat="row in machine.slots">
    <span>${row.name}</span><span>${row.count|number}</span>
  </div>
</div>
```

## 一、占位符：`${路径}` 与 `${路径|格式化器}`

| 写法 | 含义 |
| --- | --- |
| `${name}` | 取值 |
| `${a.b.c}` | 取 `a`，再从它上面取 `b`，再取 `c` |
| `${count|number}` | 取 `count`，再用 `number` 格式化 |
| `${name}` 未闭合（`${name`） | **原样当文本**，不当作占位符 |
| `${}` / `${|upper}` | **原样当文本**（没有路径可解析，保持写的样子） |

**没有表达式。** 没有算术、没有比较、没有函数调用：一条路径 + 一个（可选的）格式化器。
需要计算就写在 Java 里——视图模型是 Java，会被类型检查，也能断点。
一个能计算的模板语言意味着一个解析器、一套错误信息、一套优先级规则，和一种"写页面时看不见、
运行时才炸"的失败方式。

**格式化器**是固定的一小撮，未知的名字会上报并退回原始值：

| 格式化器 | 作用 | 例 |
| --- | --- | --- |
| `number` | 千位分隔，按值自身类型打印（不把 `float` 拉宽成 `double`） | `6200` → `6,200` |
| `upper` / `lower` | 大小写 | `Misia` → `MISIA` |
| `percent` | **接受分数**，乘 100 加 `%` | `0.62` → `62%` |

```
no formatter named 'numer'; expected one of [number, upper, lower, percent]
```

## 二、路径怎么解析

要点的第一段通过**作用域链**（`data-repeat` 引入的变量就在链上），其余各段落在值上：

| 目标 | 读法 |
| --- | --- |
| `Map` | 键 |
| `List` | 下标（`${item.0}`） |
| 普通对象 | `getName()` → `isName()` → 公开字段 `name`（按此顺序） |

**只读**：绑定不会写回你的对象。"双向绑定"不在这个库里——一个能任意写回对象的界面，
它的 bug 会一直隐藏到别的地方出问题为止。

> **绑定你自己的类，不要绑 Minecraft 的。**
> 反射是按**名字**查的，而名字在发布后的 jar 里不存在：运行时游戏类的字段和方法是混淆过的，
> Forge 只会重映射它**编译期**能看见的引用，不会重映射运行时拼出来的字符串。
> `${level.dimension}` 绑到 `World` 会在开发环境好用、在发布版失效——最糟的那种失效。
> 用一个小视图模型把游戏状态包起来，在普通 Java 里读游戏对象，再绑到那个视图模型。

## 三、`data-repeat`：按数据重复

```html
<div class="mui-list-row" data-repeat="row in machine.slots">…${row.name}…</div>
```

- 语法就是 `变量名 in 路径`，`in` 必须是**独立的词**（`index` 这样的名字不会误切）；
- 路径可以是任何"可重复"的值：`List`、`Iterable`、数组；
- 每一次重复有**自己的作用域**（一层新作用域套在当前作用域外面），所以内层可以直接写
  `row.name`，也可以继续用外层的数据；
- 一个元素**同时**写 `data-repeat` 与 `data-if` 时，`data-if` 会在**每次重复里**判断一次
  （带着循环变量）——所以 `data-if="${row.stale}"` 才是有意义的写法；
- 两个指令属性**不会留在绑定后的文档里**：绑定产物是一个新文档，不会再要求被绑定。

写错时的上报：

| 写法 | 上报 |
| --- | --- |
| `data-repeat="rows"` | `'data-repeat="rows"' is not 'name in path'; the element was left out` |
| `data-repeat="1 in rows"` | `…does not name a variable and a path; the element was left out` |
| `data-repeat="row in count"`（数字） | `'count' is a integer and cannot be repeated over; the element was left out` |
| 路径取不到 | `nothing provides 'rows'; 'rows' rendered as empty` |

## 四、`data-if`：条件出现

```html
<div class="mui-badge" data-if="${machine.online}">Online</div>
<div class="mui-note" data-if="${machine.error}">${machine.error}</div>
```

真值规则**明写在这里**（与 JavaScript 不同，是刻意的）：

| 值 | 真？ |
| --- | --- |
| 缺失 / `null` | 假 |
| `Boolean.FALSE` | 假 |
| 空字符串 `""` | 假 |
| 空的集合 / 数组 | 假 |
| **`0`** | **真** |
| 其它一切 | 真 |

`0` 为真是刻意的：界面经常显示数量和额度，`data-if="${count}"` 把"0 个"那一行藏掉，
是个陷阱而不是便利。

`data-if` 取假值时，那个元素**整棵子树都不进文档**——所以它的盒子不存在、不接受点击、
不影响布局（和 `display: none` 的效果一致，但在绑定阶段就决定了）。

`data-if` 也可以不带 `${}`：`data-if="online"` 与 `data-if="${online}"` 等价。

## 五、绑定出现在哪

```html
<span>${a}</span>                                  <!-- 文本节点 -->
<div class="bar" style="width: ${percent}%"></div> <!-- 属性值（可混合文本） -->
<input value="${name}" placeholder="名称">          <!-- 控件属性 -->
<button data-on-click="${action}">Go</button>      <!-- 连动作名也能绑 -->
<div data-repeat="r in rows">${r.0}</div>          <!-- 指令参数里也能用 -->
```

- 文本里与属性里的语法完全一样；
- 一个属性值恰好是**一个**占位符时，结果是插值后的**字符串**（不是对象）；
- 插值结果里即使出现 `${` 也不会再展开（不递归）。

## 六、Java 侧：三种数据源

```java
// 1. 普通对象（getter 就是声明，不需要注解或注册）
Mui.screen("screens/machine.html").data(machine).open();
Mui.screen("screens/machine.html").data("machine", machine).open();  // 页面写 ${machine.energy}

// 2. Map（键值随手拼，适合调试与快速原型）
MapValueSource source = MapValueSource.of("title", "Furnace")
        .put("percent", 62)
        .put("rows", rows);
source.put("percent", 80);          // 写一个值 → revision 自动前移
source.putAll(other);               // 批量写 → revision 只前移一次
source.remove("rows");

// 3. 自己的 ValueSource（数据不来自普通对象时）
public Object value(String name) { … }
public long revision() { … }        // 变了就 +1
```

`ValueSource` 只有两个方法：`value(name)` 与 `revision()`。`revision()` 是**唯一的变更通知机制**：
引擎每帧比较一次这个数字，动了才重新绑定。这样写的原因很实际——写可能来自网络线程、
读发生在渲染线程，一个单调递增的计数器是这种情况下最省事也最不容易错的信号。

```java
public interface ValueSource {
    Object value(String name);   // 没有这个名字就返回 null
    long revision();             // 变了就换个数字
}
```

## 七、什么时候重新绑定

| 触发 | 说明 |
| --- | --- |
| `MuiScreen` 注册的**动作跑完** | 默认行为（`rebindAfterActions`）：动作几乎总是"数据动了"的那一刻 |
| `.rebind()` | 数据在别处变了（tick、网络线程）时，由你的代码推一下 |
| `MapValueSource.put/putAll/remove` | 自己就前移了 revision |
| `ReflectiveValueSource.bump()` | 只读对象需要你显式说"我变了" |
| 什么都没动 | **不重新绑定**：每帧只比较一个数字 |

**重新绑定会重建整棵树**，所以有三件事值得记住：

1. **玩家的输入会被数据覆盖**（见 [07 表单控件](07-forms.md) 第九节）：绑定是"页面在说控件该是什么值"。
   要保住输入，就在动作里把它读回模型；
2. 悬停、焦点、滚动位置按 **id** 恢复——所以**控件与可滚动盒子要有 `id`**；
3. 重新绑定**不消耗**模板：`Binder.bind` 产生新文档，原模板原封不动，所以绑定两次不会嵌套展开。

## 八、诊断

绑定层的每条问题都进页面的诊断（[13 调试](13-debugging.md)）：

| 情况 | 上报 |
| --- | --- |
| 名字没有数据源提供 | `nothing provides 'title'; 'title' rendered as empty` |
| 格式化器不存在 | `no formatter named 'x'; expected one of …` |
| `data-repeat` 语法错 | 见上表 |
| 重复的对象不可重复 | `'count' is a integer and cannot be repeated over; the element was left out` |

一个"名字拼错所以渲染成空"的页面，和一个"本来就该是空"的页面长得一模一样——
所以这条上报不是可选项。

## 九、写视图模型的建议

1. **一个界面一个模型类**，getter 直接回答界面要显示什么（`getPercent()`，而不是让页面算）；
2. **不要在模型里做字符串拼接**：那是 `|` 格式化器和页面的工作；反过来，需要本地化或特殊格式的
   文本，就在模型里给成品字符串，页面直接 `${statusText}`；
3. **模型要能安全地在渲染线程读**：Tick 线程写的字段用 `volatile` 或者只读快照，
   然后 `bump()`；
4. **不要绑游戏对象**（见上），包一层。

## 相关

- 动作与重绑定：[09 动作](09-actions.md)
- 控件状态与输入：[07 表单控件](07-forms.md)
- 全部诊断在哪看：[13 调试](13-debugging.md)
