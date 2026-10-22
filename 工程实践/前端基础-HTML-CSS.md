---
知识点: 前端基础 HTML CSS
层级: 基础
适用版本: 通用
更新日期: 2026-10-22
关联:
  - 工程实践知识点地图
  - 前端开发
标签:
  - 前端
  - 原理
---

# 前端基础：HTML 与 CSS

## 语义化 HTML

- 用对标签：`<header>` / `<nav>` / `<main>` / `<article>` 而不是满屏 `<div>`；SEO 和无障碍都靠它。
- 表单：`<label for>` 关联 input，点文字就能聚焦；`required` / `type="email"` 原生校验先用。

## 盒模型

```
| margin | border | padding | content |
```

- `box-sizing: border-box` 全局设上，宽高含 padding 不含 margin，心智负担减半。
- 外边距塌陷：父子 margin 合并，用 `overflow: hidden` 或 padding 隔开。

## 布局：Flex 与 Grid

```css
.container { display: flex; gap: 12px; }
.item { flex: 1; }                                    /* 等分 */
.center { display: grid; place-items: center; }       /* 居中一行搞定 */
```

- 一维布局用 Flex（导航栏、表单行），二维用 Grid（卡片墙、仪表盘）。
- `flex: 1` 是 `flex-grow: 1` 的缩写，别和 `flex-basis` 搞混。

## 定位

- static / relative / absolute / fixed / sticky；absolute 找"最近的非 static 祖先"。
- `z-index` 只在定位元素上生效，层叠上下文是坑源。

## 响应式

```css
@media (max-width: 768px) { .sidebar { display: none; } }
```

- 移动优先：先写小屏，再 `@media (min-width)` 往上加。
- 单位：`rem` 跟根字体，`vw/vh` 跟视口；别再满屏 px。

## 层叠与特异性（样式"不生效"先查这个）

- 权重：行内样式 > ID > 类/属性/伪类 > 元素/伪元素；`!important` 压过一切（能不用就不用，盖上去就再也盖不下来）。
- 同权重后写的赢；不同权重再靠后也赢不了——"后面的样式没生效"九成是特异性不够，不是没加载。
- 调试：浏览器 DevTools 看被划掉的声明，一眼看出谁赢了。

## 现代 CSS（2024+ 可放心用）

- CSS 变量：`:root { --primary: #1677ff; }`，换肤/暗黑模式不用改每处颜色。
- `:has()`：父选择器终于有了，`.card:has(img)` 选中"带图的卡片"。
- Container Queries：`@container (min-width: 400px)`，组件按自己宽度响应，不再只看视口——同一张卡片复用到侧边栏和主区不用写两套。
- `aspect-ratio: 16/9`：图片/视频占位防抖动，一行搞定。

## 和前端开发笔记的分工

- 这篇是"地基"（标签 / 盒模型 / 布局）；框架、构建、联调见 [工程实践/前端开发.md](工程实践/前端开发.md)。
