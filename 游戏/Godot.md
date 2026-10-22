---
知识点: Godot
层级: 基础
适用版本: 通用
更新日期: 2026-10-22
关联:
  - 总览
  - Lua
标签:
  - 游戏
  - 实践
---

# Godot

开源免费的 2D/3D 游戏引擎（MIT 协议，真免费无抽成），轻量（几十 MB、免安装解压即用），适合碎片时间做小 demo。对 Java 后端的额外价值：游戏循环、状态机、事件驱动的设计思想，和流程引擎/ Agent 状态机是相通的。

> 时效性：版本信息 2026-10-17 已验证（godotengine.org 下载页、官方文档 release policy）。

## 版本（2026-10）

- 当前稳定：**Godot 4.7.2**（2026-08 发布）；4.8 在开发中。学 4.x 系列即可，核心概念跨小版本稳定。
- 两个渲染器：Forward+（桌面高画质，需 Vulkan）和 Compatibility（OpenGL，低端机/网页导出用）。
- C# 版需另装 .NET SDK；纯 GDScript 版开箱即用。

## 核心概念

- **节点（Node）**：一切皆节点。精灵、碰撞体、计时器、UI 按钮都是节点。
- **场景（Scene）**：节点组成的树，可保存复用。做游戏 = 拼场景 + 给节点挂脚本。
- **GDScript**：Python 风格的动态语言，缩进定代码块，上手极快。示例：

```gdscript
extends CharacterBody2D

@export var speed := 200.0

func _physics_process(delta):
    var dir := Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")
    velocity = dir * speed
    move_and_slide()
```

- **信号（Signal）**：Godot 的观察者模式，节点间解耦通信。按钮被点 → 发出 `pressed` 信号 → 任意节点连接处理。理解成事件总线即可。
- **游戏循环**：`_process(delta)` 每帧调用（渲染相关）、`_physics_process(delta)` 固定步长（物理/移动，防帧率差异）。

## 2D 入门路径

1. 官方文档 Getting Started：跟着做完第一个 2D 小游戏（官方教程完整可用）。
2. 练 TileMap（地图拼接）+ 精灵动画 + 碰撞体，拼出一个可走动的场景。
3. 用信号做交互：进区域触发对话、拾取物品。
4. 导出 HTML5：在项目设置切到 Compatibility 渲染器，一键导出，HTTP 服务下打开（`file://` 打不开 WebAssembly）。
5. 进阶：状态机管角色状态（待机/跑/跳/受伤），和你熟悉的流程节点是同一种思维。

## 资源

- 官方文档：https://docs.godotengine.org（有中文）
- 下载：https://godotengine.org/download（Windows 版 2026-08-18 更新）
- GitHub demo 项目：官方 `godot-demo-projects`，按功能查示例
- 商业案例：Brotato、Dome Keeper、Cassette Beasts 都是 Godot 做的（已过"玩具引擎"阶段）

## 5 个常见坑

1. **`@onready` 之前访问节点**：节点在 `_ready()` 前可能还没进树，用 `@onready var x = $Node` 声明，别在成员初始化时直接 `$Node`。
2. **物理写在 `_process` 里**：帧率一变速度就变，移动/碰撞必须用 `_physics_process(delta)`。
3. **信号连重了**：编辑器里连一次、代码里又 `connect` 一次，回调执行两遍；统一一种方式。
4. **导出网页白屏**：十有八九是 `file://` 直接打开或用了 Forward+ 渲染器，切 Compatibility + 起 HTTP 服务。
5. **资源路径用绝对路径**：项目换机器就炸，统一用 `res://` 开头的项目相对路径。

## 相关

- [探索/探索方向建议.md](探索/探索方向建议.md)：方向 3 Godot 入门（可实践项目：给《无空剑门》做像素风小场景 demo）
- [开发语言/Python/Python.md](开发语言/Python/Python.md)：GDScript 语法接近 Python，可对照看

## 疑难杂症

### 1. 中文显示全是方框
- **现象**：Label 写中文，运行出来是口口口。
- **原因**：默认字体不含 CJK 字形。
- **解决**：导入一个中文字体（ttf/otf），在主题或 Label 的 Font 里指定；注意字体文件体积，移动端用子集化字体。

### 2. 大版本升级后项目打不开
- **现象**：3.x 项目用 4.x 打开，报错或场景全乱。
- **原因**：Godot 大版本不保证兼容（3→4 改动极大）。
- **解决**：升级前完整备份；用官方迁移工具走一遍；新项目直接定一个 LTS 版本别中途换。

### 3. 导出预设突然丢失
- **现象**：昨天还能导出，今天导出预设没了。
- **原因**：`export_presets.cfg` 没进版本控制，或换了机器/重装 Godot。
- **解决**：`export_presets.cfg` 提交进 git；导出模板（export templates）按版本装好，别和编辑器版本搞混。

## GDScript 进阶

```gdscript
# await：等信号/等一帧，不用回调地狱
await get_tree().create_timer(1.0).timeout
await $Button.pressed

# signal 带参数 + 动态连接
signal hp_changed(old, new)
hp_changed.emit(100, 80)
$Enemy.hp_changed.connect(_on_hp_changed)

# Callable 存起来当回调
var cb: Callable = _on_hp_changed
cb.call(100, 80)

# 上下文无关的工具函数放 static
static func clamp_hp(v): return clampi(v, 0, 100)
```

- `await` 后面跟信号是最常用的写法（等动画播完、等网络回来），比 `_process` 里轮询干净。
- 信号连重了会触发多次，连接前 `if not is_connected(...)` 判一下。
- 性能敏感处（每帧跑的）别频繁 `get_node`，`@onready` 缓存引用。

## 2D 实战：TileMap + 碰撞层

```gdscript
# TileMapLayer（4.x 写法）+ CharacterBody2D 移动模板
extends CharacterBody2D
@export var speed := 200.0
func _physics_process(_delta):
    var dir := Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")
    velocity = dir * speed
    move_and_slide()
```

- 地形画在 TileMapLayer 上，碰撞用"物理层"单独画，别和美术层混一起。
- 碰撞层（Layer）vs 监测掩码（Mask）：Layer 是"我是谁"，Mask 是"我撞谁"；玩家 Layer=1、Mask=2（墙），子弹 Layer=4、Mask=2。
- 调物理手感三个数：`speed`、`acceleration`（用 `velocity.move_toward` 做平滑）、跳跃的 `gravity`。

## 基础知识补强

### 核心概念与术语

- 游戏引擎：一套现成的游戏地基，管渲染、物理、音频、输入、资源加载；你只写玩法逻辑，不用从画像素开始造轮子。
- 渲染：把场景数据变成屏幕像素的过程；引擎每帧重画一次，帧率就是一秒画多少次。
- 帧（Frame）：一帧就是画面的一次完整更新；程序里说的"每帧执行"就是随这个节拍执行。
- delta：上一帧到这一帧的真实秒数。一切"按时间"的移动都要乘 delta，帧率变了速度才不变。
- 场景树：运行中的游戏就是一棵活的节点树，父节点移动/旋转/隐藏，子节点整体跟着变，这是 Godot 最核心的心智模型。
- 变换（Transform）：位置、旋转、缩放三件套；子节点的实际位置是"父变换叠自己的变换"，不是自己填的那个数。
- 资源（Resource）：与节点分离的数据资产，如图片、音频、自定义数据文件；多个节点可引用同一份资源，改一处全生效。
- 输入映射（Input Map）：给按键起逻辑名（如 jump、move_left），代码只认逻辑名；换键位改映射即可，不用改代码。
- 分组（Group）：给节点贴的标签，运行时可按组批量找节点（如所有敌人），相当于给场景树加的索引。
- 自动加载（Autoload）：挂为全局单例的场景或脚本，任何地方可直接访问，常用来放全局状态、音频管理、存档。

### 基础知识与用法

2D 坐标系先钉死，这是所有位置 bug 的总源头：

```text
Godot 2D：原点在屏幕左上角，X 向右增，Y 向下增（和屏幕像素一致，与数学课相反）
单位默认是像素。节点 position 表示它相对父节点原点的偏移。
角度：正角度顺时针转（因为 Y 朝下）；向量 (0, -1) 是"向上"，(0, 1) 是"向下"。
3D 约定：Y 向上，角色面朝 -Z 方向，地面是 XZ 平面。
```

向量的四个基本操作（GDScript 示例，移动、瞄准全靠它们）：

```gdscript
var a := Vector2(3, 4)
a.length()            # 长度 = 5（勾股定理）
a.normalized()        # 单位化，只保留方向，长度变 1
var dir := (target_pos - my_pos).normalized()   # 朝向目标的方向
a.dot(b)              # 点积：判断夹角，>0 同向，=0 垂直，<0 反向
velocity = dir * speed  # 方向 × 速率 = 速度，游戏里最常见的一行
```

三种 2D 物理体怎么选（原文讲了碰撞层，这里补身体本身的区别）：

| 类型 | 谁驱动 | 典型用途 |
|---|---|---|
| StaticBody2D | 不动，谁也推不动 | 墙、地面、平台 |
| CharacterBody2D | 代码驱动，自己写速度 | 玩家、可控角色 |
| RigidBody2D | 物理引擎驱动，吃重力和力 | 掉落物、箱子、弹跳球 |
| Area2D | 不挡人，只监测谁进来 | 触发区、拾取物、伤害区 |

- 选错典型症状：让 RigidBody 当玩家，控制手感全是物理反馈，调不动；让 CharacterBody 模拟掉落，又得自己写重力。

节点生命周期顺序，记这条链就够了：

```text
实例化 → _init（对象刚造出来，节点还没进树，别碰别的节点）
      → 进入场景树 → 子节点先 _ready，父节点后 _ready
      → 逐帧 _process / _physics_process
      → 移出树时 _exit_tree（收尾、断信号放这里）
_ready 只跑一次；反复进出树的节点别把一次性初始化假设写死在别处。
```

GDScript 基础语法补充一小段（进阶节之外的地基部分）：

```gdscript
var hp := 100               # := 表示类型由右边推断
var names: Array[String] = ["a", "b"]   # 数组，可带元素类型
var cfg := {"speed": 200, "jump": -400}  # 字典，键值对

for i in range(3):          # 循环 0,1,2；也可以直接 for x in names
    print(i)
if hp > 50 and not names.is_empty():    # 条件与逻辑运算，写法近 Python
    print("hp=%d" % hp)     # % 做字符串格式化
```

动画与音频只记两个基础抓手：

- AnimationPlayer：把"属性随时间变化"录成动画片段，代码里按名 play；角色跑跳切换本质是切片段加混合。
- 音频分总线（Bus）管理：背景音乐、音效、语音各走一条总线，音量、静音统一调，别在每个音效节点上散调。

### 关键机制速览

- 父子变换叠加：子节点填的是相对坐标，父节点一动子节点跟着动，UI 和角色挂点（武器挂手上）都靠这条。
- 树的处理顺序与层级无关的还有 Canvas 层：UI 和世界分属不同画布层，UI 永远压在游戏画面之上，不受相机影响。
- 信号是单向通知：发出方不知道谁在听；在编辑器里连和代码里连是同一份连接关系，重复连接就会重复触发。
- 资源引用是共享的：两个敌人引用同一份属性资源时，运行时改了一个的资源值，另一个可能跟着变，需要独立副本时记得复制资源。
- 实例化场景就是"把一份场景模板克隆成活节点塞进树"，预加载保证克隆时不卡顿，适合频繁生成的子弹、敌人。
- 物理帧与渲染帧解耦：物理固定步长保证手感稳定，渲染帧率浮动不影响物理结果，卡顿严重时物理会追帧补算。
- 帧率与耗时换算：60 帧约等于每帧 16.7 毫秒预算，单帧逻辑超过这个数就开始掉帧，定位卡顿先看哪帧超时。

### 新手易错点

- 方向不归一化就乘速度：手写方向时右加下得到斜向向量，长度是直向的约 1.4 倍，斜着跑明显更快，对比时以为是错觉。
- 整数相除丢小数：两个整数相除结果仍是整数，需要小数结果时先把一侧写成浮点（如 5.0 / 2），比例计算最常中招。
- 把计时写成"每帧加一"：帧率一变计时就全错，计时一律用 delta 累加秒数，或直接用计时器节点。
- 全局到处找节点：跨场景用长路径取节点，树一改名就全线报空；固定协作走信号或分组，少写死路径。
- 误以为改资源等于改单个实例：共享资源被多处引用时改一处影响全局，调数值前先确认这份资源是不是独享的。
- 在 _init 或成员变量初始化里访问场景树：此时节点还没进树，取子节点必然为空，凡是依赖树的引用统一放 _ready 之后。
- 3D 里拿 2D 直觉套坐标：2D 是 Y 向下，3D 是 Y 向上、前方是 -Z，两套约定来回切项目时最容易在这里栽。
