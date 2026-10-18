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

- [09-探索/探索方向建议.md](09-探索/探索方向建议.md)：方向 3 Godot 入门（可实践项目：给《无空剑门》做像素风小场景 demo）
- [02-开发语言/Python.md](02-开发语言/Python.md)：GDScript 语法接近 Python，可对照看

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
