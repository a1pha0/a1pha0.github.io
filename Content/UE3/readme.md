# 虚幻引擎3

官方文档：https://docs.unrealengine.com/udk/Three/WebHome.html

## 游戏性元素

### 游戏类型概述
游戏类型负责处理游戏规则。

**GameInfo(游戏信息)**
游戏的游戏类型是由GameInfo类控制的。游戏中包含的每种游戏类型都要创建一个这个类的子类。一个游戏可以有很多种游戏类型，但是给定时间仅能使用一种。当初始化关卡来获得游戏性 UGameEngine::LoadMap() 时便会实例化GameInfo actor。除非另行指定游戏类型，否则这种游戏类型将在关卡生命周期内使用。

这个类一般负责以下处理：
 * 决定在哪里产生玩家。
 * 赋予玩家适当的武器项。
 * 设置时间限制。
 * 记录玩家的 分数/技能 。
 * 如果需要重置关卡。
 * 判断游戏结束的条件。

游戏类型也指出了使用哪个类来构造玩家，使用哪个HUD类，及使得游戏类型是完全唯一的几个其他类。

### 玩家概述

**PlayerController(玩家控制器)**

虚幻引擎中的玩家及其他实体一般是由 Controllers(控制器) 控制的。 Controller 是实体的大脑，决定了实体的行为。玩家有他们自己特殊的的 Controller(控制器) ，既 PlayerController(玩家控制器) ，它具有从玩游戏的人那里获得输入并将其转换为动作的功能。

**Pawn**

尽管 PlayerController 构成了玩家的大脑，而 Pawn 是游戏中玩家或其他实体的物理展现或躯体。 Pawn 从 Controller 获取命令，并执行它们。这是处理玩家的动画、毁坏及其他物理相关方面问题的地方。

**Camera(相机)**

玩家进入世界的视角是由相机系统和 Camera 类处理的。这个类指出了从哪个位置及旋转值处把世界渲染到视口中。从本质上讲，是指玩家的眼睛。通过使用自定义的 Camera 类，或者甚至是一个完全自定义的相机系统，可以创建出一个全新的唯一的视角类型。

**输入**

PlayerController 具有获得的玩家输入并把它转换为游戏中的动作的功能。这个输入来自 Input 类，尤其是 PlayerInput 类。这个类负责从控制器、键盘或鼠标获得按钮或按键按下的动作，并把它们转换为可用数据。然后当这个数据准备好后， PlayerController 获得这个数据进行处理。

### HUD和UI概述

Heads Up Display (HUD)(平视显示器仪)是指在游戏过程中覆盖在屏幕上的状态及信息。HUD的作用是通知玩家当前的游戏状态，比如分数、生命值、剩余时间等。HUD是非交互性的，这意味着玩家不需要点击HUD上的元素，但在某些HUD和用户界面难以区分的游戏类型中这个会变成灰色区域。

用户界面(UI)是指菜单和其他的交互性元素。这些元素一般描画覆盖在屏幕上，和HUD很像，但是在某些情况下，它们可以是游戏世界本身的一部分，这时会渲染到世界的一个表面上。UI的最显著的示例是当游戏启动时显示的主菜单或者当玩家暂停游戏时显示的暂停菜单。但是，其它的UI可以在游戏过程中显示。这些UI可以用于在游戏中或者更复杂的情况中显示角色之间的对话，比如在RTS或RPG中，它们可以作为组成游戏性本身的一部分，从而使得玩家选择武器、装备及构建单元等。

HUD 类是在屏幕上显示元素的基类。游戏中的每个玩家有他们自己的HUD实例，将会描画到它们各自的视口中。要使用的HUD的类型或类是通过正在使用的游戏类型指定的。

HUD 类可以使用两种主要的方法来在屏幕上显示元素： 画布描画或Scaleform GFx视频。

Canvas(画布)

Canvas 类包含了向屏幕(或者通过使用脚本贴图的其他表面)上描画文本和图像的所需的所有功能。每经过一次描画循环，就会把一个新的 Canvas(画布) 分配给 HUD ,并且可以使用那个画布向屏幕上描画必要的元素。

Scaleform

虚幻引擎3中集成的Scaleform GFx使得可以使用内置在Adobe Flash Professional中的运动图像作为游戏中的HUD和UI。它由一组代表视频、视频播放器及视频中包含的单独对象的类组成。可以在 HUD 中创建视频播放器，从而使得可以完全地控制在游戏中显示的视频。

### 配置文件
虚幻引擎3依赖于配置文件来指示它将如何运作及初始化。配置是由键值对来决定的，一个键可以和一个或多个值相关联。

无论何时当在一个对象上调用 SaveConfig() 或 StaticSaveConfig() 函数时，引擎把所有的变量保存到指定的配置文件中（除非已经定义类可以把它的设置保存到可替换的配置文件中）。

**特殊字符**
* \+    如果该属性还不存在则添加一行（从前一个 .ini 文件或者同一个 .ini 文件的前面的部分）。
* \-    删除一行（但是它必须是精确匹配）。
* .     添加一个新属性。
* !     删除一个属性；但是不必是精确匹配，仅需要匹配属性的名称即可。
注意 ： . 和 + 类似的，只是 . 可以添加一个重复的行。这对绑定是有用的（正如在 DefaultInput.ini 中所看到的），比如，其中最底部的绑定生效，所以如果您添加了类似于以下的东西：

[Engine.PlayerInput]
Bindings=(Name="Q",Command="Foo")
.Bindings=(Name="Q",Command="Bar")
.Bindings=(Name="Q",Command="Foo")
它将可以正确地工作。但是如果使用 + 则不能添加最后一行，且您的绑定将是错误的。由于配置文件绑定，将会发生上面的应用形式。

**创建配置文件**
当创建一个基于另一个配置的新配置文件时，您需要包含 [Configuration] 项和 BasedOn 项的键值对。
比如，如果您的配置文件将要基于基础配置文件 Engine.ini ，您需要在您文件的顶部放置以下信息：

[Configuration]
BasedOn=..\Engine\Config\BaseEngine.ini

**动态 vs 静态配置**
程序员可以使用两种不同的方法来把对象的配置信息保存到配置文件中： 动态和静态。基本上，这意味着如果您在对象实例上调用 SaveConfig() ，那么将会保存运行时变量。如果在一个类变量上调用 StaticSaveConfig() ，它会将对象的 默认 值写到配置文件中。上面的所有实例都是保存了运行时对象的配置信息。它将会允许您恢复最终用户配置，或者覆盖在配置文件的 Unrealscript 中定义的这个类的默认属性。
以下片段保存了 X 的默认值：

class'TestConfigChild'.default.X = 30;
class'TestConfigChild'.static.StaticSaveConfig();
X 的默认值被写入到了配置文件。

[UDN.TestConfigChild]
X=30

**可用的配置文件**
配置文件位于任何给定项目的 Config 目录中（例如，%UDK_ROOT%\UDKGame\Config）。
这里是使用虚幻引擎的特定游戏可用的配置文件列表：

DefaultCharInfo - 虚幻竞技场 3 使用的默认角色。
DefaultCompat - 在表格中的默认功能。
DefaultEditor - 虚幻编辑器的默认配置。
DefaultEditorKeybindings - 在虚幻编辑器中使用的默认按键绑定。
DefaultEditorUDK - 在虚幻编辑器中使用的默认 UDK 特定设置。
DefaultEditorUserSettings - 虚幻编辑器的默认用户设置。
DefaultEngine - 虚幻引擎 3 的默认配置。
DefaultEngineUDK - 在虚幻引擎 3 中使用的默认 UDK 特定设置。
DefaultGame - 在虚幻引擎 3 中用于游戏运行的默认游戏配置。
DefaultGameUDK - 在虚幻引擎 3 中用于游戏运行的默认 UDK 特定游戏设置。
DefaultInput - 在虚幻引擎 3 中可以用于游戏运行的默认按键绑定。
DefaultInputDefaults - 在使用 Coalesced.ini 的时候需要它才能重新设置默认值。
DefaultLightmass - 供 Lightmass 使用的默认 Lightmass 设置。
DefaultUI - 在虚幻引擎 3 中用于游戏运行的默认用户界面配置。
DefaultWeapon - 供虚幻竞技场 3 使用的默认武器设置。
DefaultWeaponDefaults - 在使用 Coalesced.ini 的时候需要它才能重新设置默认值。
