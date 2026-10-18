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

## 和前端开发笔记的分工

- 这篇是"地基"（标签 / 盒模型 / 布局）；框架、构建、联调见 [06-前端/前端开发.md](06-前端/前端开发.md)。
