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

## 基础知识补强

### 核心概念与术语

- HTML 是标记语言不是编程语言：它只描述「这是什么内容」，没有逻辑，逻辑归 JS，样子归 CSS。
- 元素 = 开始标签 + 内容 + 结束标签；少数元素没有结束标签（img、br、input、meta 等），叫空元素。
- 属性写在开始标签里给元素加信息：全局属性如 id、class、title、data-*，以及各标签专属的 href、src、alt 等。
- id 全页唯一、给单个元素用；class 可复用、给一类元素用，CSS 里主要靠 class 选人。
- 块级元素独占一行、可设宽高（div、p、h1–h6、ul、section）；行内元素只占内容宽度、宽高不听使唤（span、a、strong、em）。
- 行内块（inline-block）是中间态：不独占一行，但能设宽高，img 和 button 天生接近这种行为。
- 文档流（normal flow）：元素按 HTML 书写的先后顺序自上而下、由左到右排布，定位和浮动都是在「脱离或偏移」这个默认秩序。
- 层叠（cascade）：多条样式打架时决定谁赢的规则总称，来源、特异性、书写顺序都是它的组成部分。
- 继承：部分属性（字体、颜色、行高这类文字属性）会从父元素传给子元素，布局类属性（宽高、边距）不继承。
- 视口（viewport）：浏览器里实际显示页面的那块区域，移动端要靠 meta 标签告诉浏览器按设备宽度算。
- 相对单位都是「相对某个参照物」：em 相对父元素字号，rem 相对根元素字号，% 多数情况相对父元素对应尺寸。
- 伪类是元素的状态选择器（:hover 悬停、:focus 聚焦、:first-child 第一个孩子），伪元素是造一个不存在的子元素（::before、::after）。

### 基础语法与用法

一份完整 HTML 文档的骨架（所有页面都是这个结构）：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>页面标题</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <header><h1>站点名</h1></header>
  <main>
    <p>正文段落，<a href="/list">去列表页</a></p>
    <img src="cover.png" alt="封面图">
  </main>
  <script src="app.js"></script>
</body>
</html>
```

- `<!DOCTYPE html>` 是文档类型声明，不是标签，少了它浏览器会进怪异模式，布局各种不一致。
- charset 必须放在 head 最前面，否则中文可能乱码；viewport 那行是移动端适配的前提。
- link 引 CSS 放 head（样式先到位再渲染），script 放 body 末尾（别挡住内容渲染）。

最常用的结构标签组合：

```html
<ul>
  <li>无序列表项</li>
  <li>导航菜单就是它加样式</li>
</ul>
<ol><li>有序列表，自带序号</li></ol>

<table>
  <tr><th>姓名</th><th>年龄</th></tr>
  <tr><td>林溯</td><td>28</td></tr>
</table>

<form action="/save" method="post">
  <label for="n">姓名</label>
  <input id="n" name="name" type="text" required>
  <input type="email" name="email" placeholder="邮箱">
  <input type="checkbox" name="ok"> 已确认
  <select name="city">
    <option value="sz">深圳</option>
    <option value="bj">北京</option>
  </select>
  <textarea name="memo" rows="3"></textarea>
  <button type="submit">提交</button>
</form>
```

- 表单提交时发给后端的是 input 的 name 和值，id 只是给 label 和 JS 用的，别把两者搞混。
- a 标签的 href 为空或写 `#` 会跳回顶部，纯占位时别乱用。

CSS 的语法解剖，一条规则就三部分：

```css
/* 选择器 { 属性: 值; 属性: 值; } */
.card .title {            /* 后代选择器：.card 里面的 .title */
  color: #333;            /* 十六进制颜色，也可 rgb() / 颜色名 */
  font-size: 16px;
  line-height: 1.5;       /* 行高用无单位数字，表示字号的倍数 */
}
#main > p { margin: 8px 0; }   /* > 只选直接子元素，不含孙辈 */
a:hover { color: red; }       /* 状态伪类 */
li:first-child { font-weight: bold; }
input[type="email"] { border: 1px solid #ccc; }  /* 属性选择器 */
```

文字与颜色的基础设置：

```css
body {
  font-family: "Microsoft YaHei", sans-serif;  /* 从前往后找可用字体 */
  font-size: 16px;
  color: rgba(0, 0, 0, 0.85);   /* rgba 多一个透明度通道 */
  text-align: left;
}
.hidden { display: none; }          /* 彻底不占位 */
.invisible { visibility: hidden; }  /* 看不见但占着位置 */
.fade { opacity: 0.5; }             /* 半透明，位置和点击都在 */
```

### 关键机制速览

- 浏览器读 HTML 是流式的：边下载边解析边渲染，所以样式和脚本的摆放位置直接决定用户先看到什么。
- CSS 选择器是从右往左匹配的：`.card .title` 先找全页的 .title 再往上验祖先，理解这点就懂为什么选择器别写太长串。
- 继承只传「文字类」属性：给 body 设字体颜色全页生效，给 body 设 border 不会每个子元素都有边框——想全有就用 * 或逐个设。
- display 决定元素的排布身份：同一个元素改 display 就能换行为，布局问题先查 display、再查宽高、最后查定位。
- 百分比宽高的大多数参照是父元素，但 height 的 % 要求父元素有明确高度，否则算不出来，这是「height: 50% 没效果」的根源。
- 颜色的四种写法等价：颜色名、#十六进制、rgb()、rgba()/hsl()，选团队顺手的统一用，新代码优先十六进制或 rgba。
- 外边距在垂直方向会「穿透和合并」，水平方向不会；这是文档流的固有规则，不是 bug，理解后用 padding 或边框隔断即可（原文已有解法）。

### 新手易错点

- 标签不闭合或嵌套交叉：`<p><div></div></p>` 这类块级套进 p 会被浏览器自动拆开，结构和预期不一致，用校验工具先扫一遍。
- 把 id 当 class 反复用：一个 id 出现多次，JS 取元素和 CSS 权重都会出怪事，复用的样式一律用 class。
- img 不写 alt、宽高：图挂了页面一片空白无提示，布局还跟着跳；alt 是给加载失败和读屏软件的最后交代。
- 单位随手混：给字体用 px、给布局也全 px，换屏幕尺寸就全乱；字号和间距走 rem，容器走 % 或弹性布局。
- 选择器写得过长过细：`body div ul li a` 这种链一改结构全废，用一个语义 class 就能解决。
- 用空格和空行硬凑版式：换个窗口宽度就散架，位置关系交给布局属性表达，别用 &nbsp; 和 br 拼页面。
- 样式「没生效」先怪缓存：先用 DevTools 看元素实际命中的规则（原文层叠一节），九成是权重或选择器没选中。
- button 放在 form 里没写 type：默认是 submit，点一下就提交表单刷新页面，纯点击按钮要写 type="button"。
