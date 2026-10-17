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
