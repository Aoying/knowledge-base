---
知识点: JavaScript
层级: 基础
适用版本: 通用
更新日期: 2026-10-31
关联:
  - 总览
  - 前端开发
标签:
  - 前端
  - 实践
---

# JavaScript

> 每晚 20:30 自动补充完善。Node 版本信息 2026-10 核验：Node 24（Krypton）为当前 LTS（支持到 2028-04），Node 22 维护到 2027-04。

## 核心语法速记

```js
// 解构
const { a, b } = obj; const [x, ...rest] = arr;
// 展开
const merged = { ...o1, ...o2 }; const arr2 = [...arr1, 4];
// 可选链 + 空值合并
const v = obj?.a?.b ?? 'default';
// 模板字符串
const s = `hello ${name}`;
```

## 异步（重点）

```js
// Promise 链
fetch(url).then(r => r.json()).then(d => console.log(d)).catch(console.error);
// async/await（推荐写法）
async function main() {
  try {
    const r = await fetch(url);
    const d = await r.json();
  } catch (e) { console.error(e); }
}
// 并发
const [a, b] = await Promise.all([p1, p2]);       // 任一失败全失败
const r = await Promise.allSettled([p1, p2]);     // 都要结果
const first = await Promise.race([p1, timeout]);  // 超时控制
```

## 数组常用

```js
arr.map(x => x * 2)        // 映射
arr.filter(x => x > 0)     // 过滤
arr.reduce((s, x) => s + x, 0)  // 归约
arr.find(x => x.id === 1)  // 查找
arr.some(...) / arr.every(...)  // 存在 / 全满足
arr.flatMap(x => [x, x * 2])
```

## 模块

```js
// ESM（现代推荐）
import { f } from './a.js'; export function f() {}
// CommonJS（Node 老项目）
const { f } = require('./a'); module.exports = { f };
```

## Node 速查

```bash
node -v                 # 版本；新项目用 Node 24 LTS
npm init -y             # 初始化
npm i lodash            # 安装依赖
npm i -D typescript     # 开发依赖
npx tsc --init          # npx 直接跑包内命令
node --watch app.js     # 文件变化自动重启（开发用）
```

## 5 个常见坑

1. **`this` 丢失**：回调里 `this` 变了，用箭头函数或 `.bind(this)`；class 方法传给事件监听最常见。
2. **浮点精度**：`0.1 + 0.2 !== 0.3`，金额计算用整数分或 decimal 库。
3. **`==` 隐式转换**：永远用 `===`，`==` 的转换规则是 bug 之源。
4. **async 函数没 await**：Promise 静默悬挂，错误 catch 不到。
5. **循环里用 var**：`for (var i...)` + 异步回调全是最后一个值，改 `let`。

## 疑难杂症

### 1. forEach 里 await 不等待
- **现象**：`arr.forEach(async x => await fetch(x))` 后面代码先跑了，请求还没完。
- **原因**：forEach 不管回调返回的 Promise，async 回调只是"启动了"而已。
- **解决**：换 `for...of` + await（串行），或 `await Promise.all(arr.map(...))`（并行）。

### 2. Promise.all 一个 reject 全部挂
- **现象**：并行调 10 个接口，1 个失败，9 个成功的结果也拿不到。
- **原因**：`Promise.all` 遇到第一个 reject 就整体 reject。
- **解决**：用 `Promise.allSettled` 拿全部结果再过滤；或给每个 promise 包 `.catch` 兜底。

### 3. 深拷贝循环引用爆栈
- **现象**：`JSON.parse(JSON.stringify(obj))` 报 `Converting circular structure to JSON`。
- **原因**：对象互相引用，JSON 序列化无限递归。
- **解决**：用 `structuredClone`（现代浏览器/Node 17+），自带循环引用处理；老环境手写 WeakMap 版深拷贝。

## ES2023+ 新特性（面试和日常都用得上）

- `Array.prototype.findLast / findLastIndex`：从后往前找，不用 `reverse()` 污染原数组。
- `Array.prototype.toSorted / toReversed / toSpliced`：不改变原数组的版本（`sort` 是原地改的，这个坑很多人踩过）。
- `Array.prototype.at(-1)`：取最后一个，比 `arr[arr.length-1]` 干净。
- `Object.groupBy(arr, fn)`：一行分组，替代手写 reduce。
- `Promise.withResolvers()`：不用在 executor 里"偷" resolve 了。
- `structuredClone(obj)`：深拷贝官方版，循环引用、Map/Set、Date 都处理。
- Import Attributes：`import data from './x.json' with { type: 'json' }`，JSON 模块标准化。

## 防抖 / 节流 / 并发控制（手写版，面试常考）

```js
// 防抖：最后一次触发后等 ms 才执行（搜索框输入）
const debounce = (fn, ms) => {
  let t; return (...a) => { clearTimeout(t); t = setTimeout(() => fn(...a), ms); };
};
// 节流：ms 内只执行一次（滚动加载）
const throttle = (fn, ms) => {
  let last = 0; return (...a) => {
    const now = Date.now();
    if (now - last > ms) { last = now; fn(...a); }
  };
};
// 并发控制：最多 n 个同时跑（批量调 Ollama 必备）
async function pLimit(tasks, n) {
  const res = [], pool = new Set();
  for (const t of tasks) {
    const p = t().finally(() => pool.delete(p));
    pool.add(p); res.push(p);
    if (pool.size >= n) await Promise.race(pool);
  }
  return Promise.all(res);
}
```

- 防抖 vs 节流一句话：防抖等停手，节流限频率。
- `pLimit` 是 p-limit 库的核心思想，20 行自己就能写，不用引库。

## ESM vs CJS（模块化的恩怨）

```js
// ESM（浏览器/现代 Node）：import/export，静态分析
import { debounce } from './utils.js';
// CJS（老 Node）：require/module.exports，运行时加载
const { debounce } = require('./utils');
```

- 新项目一律 ESM；`package.json` 加 `"type": "module"`。
- `__dirname` 在 ESM 里没有了，用 `import.meta.dirname`（Node 20.11+）。
- 引第三方库报错 `ERR_REQUIRE_ESM`：说明它只发了 ESM 版，你的 CJS 项目 require 不动——解法是项目切 ESM，或找它的 CJS 分发。

## 性能优化三板斧

1. **长任务拆分**：`setTimeout`/`requestIdleCallback` 把 200ms+ 的计算切片，别 block 主线程（INP 指标就看这个）。
2. **虚拟列表**：万级数据渲染只画可视区（`vue-virtual-scroller` / 手写 `IntersectionObserver`）。
3. **防抖 + 缓存**：搜索框防抖 300ms，结果按 query 缓存，同样输入不发第二次请求。

```js
// 大数据分片处理，不卡 UI
async function processBig(arr) {
  for (let i = 0; i < arr.length; i += 1000) {
    doChunk(arr.slice(i, i + 1000));
    await new Promise(r => setTimeout(r, 0));  // 让出主线程
  }
}
```

## 基础知识补强

### 核心概念与术语

- **弱类型动态语言**：变量没有固定类型，值有类型；同一个变量可以先后装数字、字符串、对象。
- **七种原始类型**：`number`、`string`、`boolean`、`null`、`undefined`、`symbol`、`bigint`，其余一切都是对象。
- **undefined 与 null**：undefined 是"还没赋值/不存在"（系统给的）；null 是"明确表示为空"（开发者主动给的）。
- **变量声明三兄弟**：`var` 函数作用域且可重复声明（老写法）；`let` 块作用域可改值；`const` 块作用域不可重新赋值。新代码默认 `const`，需要改值用 `let`。
- **作用域**：变量能被访问的范围，分全局、函数、块（大括号）三级；内层能读外层，外层读不到内层。
- **提升（hoisting）**：声明会被"提"到作用域顶部，`var` 和函数声明可先用后声明（值未赋时是 undefined）；`let/const` 有暂时性死区，用早了直接报错。
- **闭包**：函数记住并能访问它出生时所在作用域的变量，哪怕外层函数已经执行完。
- **对象**：键值对的集合，键是字符串（或 symbol）；属性访问用点号或方括号，方括号里可以是变量。
- **数组是特殊对象**：下标是字符串键，有 `length`；可以混装不同类型，但业务里别这么干。
- **函数是一等公民**：函数可以赋值给变量、作参数、作返回值，回调与高阶函数都建立在这一点上。
- **原型链**：对象查找属性时，自己没有就沿 `__proto__` 往上找，直到 null；类与继承只是这套机制的语法糖。
- **事件循环**：JS 单线程执行，耗时任务靠回调排队；先跑完同步代码，再清空微任务（Promise），最后取宏任务（定时器、事件）。

### 基础语法与用法

```js
// 声明与基本类型
const name = 'ao';        // const 不能重新赋值
let count = 0;            // let 可以改
count += 1;
const price = 9.9;        // number，整数浮点同一种
const ok = true;
const empty = null;       // 主动为空
let notSet;               // undefined，没赋值

console.log(typeof name);     // 'string'
console.log(typeof null);     // 'object'（历史遗留，别被它误导）
console.log(typeof notSet);   // 'undefined'
```

```js
// 控制流
if (count > 0) {
  console.log('正数');
} else if (count === 0) {
  console.log('零');
} else {
  console.log('负数');
}

for (let i = 0; i < 3; i++) console.log(i);       // 传统 for
for (const item of [10, 20]) console.log(item);  // for...of 遍历值
const user = { name: 'ao', age: 5 };
for (const key in user) console.log(key, user[key]);  // for...in 遍历键

// 逻辑运算返回的是操作数本身，不一定是布尔值
const title = '' || '默认标题';        // title 为 '默认标题'
const cfg = null ?? { size: 10 };      // null/undefined 才兜底时用 ??
```

```js
// 函数的四种常见形态
function add(a, b) { return a + b; }                 // 函数声明（有提升）
const mul = function (a, b) { return a * b; };       // 函数表达式
const sub = (a, b) => a - b;                         // 箭头函数，单表达式可省 return
const greet = (who = '匿名') => `你好, ${who}`;      // 默认参数

// 对象与数组基础操作
const u = { name: 'ao', age: 5 };
u.city = '深圳';                  // 加属性
console.log('age' in u);          // true，查键是否存在
delete u.city;                    // 删属性
const arr = [1, 2, 3];
arr.push(4);                      // 末尾加（改原数组）
console.log(arr.length, arr[0]);  // 长度与下标（从 0 开始）
```

```js
// 闭包：最常见的实际用途——计数器与私有变量
function makeCounter() {
  let n = 0;                    // 外部访问不到，只能通过返回的函数动它
  return () => ++n;
}
const next = makeCounter();
console.log(next(), next());    // 1, 2

// 类的基本形态（原型机制的语法糖）
class Chapter {
  constructor(title, words = 0) {
    this.title = title;
    this.words = words;
  }
  isLong() { return this.words > 5000; }
}
const ch = new Chapter('第一章', 6200);
console.log(ch.title, ch.isLong());
```

```js
// DOM 基础（浏览器环境）：选中元素、改内容、听事件
const btn = document.querySelector('#save');      // 按 CSS 选择器取一个
const items = document.querySelectorAll('.row');  // 取全部，返回类数组
btn.textContent = '保存';                          // 改纯文本（安全，不解析 HTML）
btn.addEventListener('click', () => {              // 绑定点击事件
  console.log('被点了');
});

// JSON：与后端和本地模型打交道的通用格式
const text = JSON.stringify({ a: 1 });   // 对象 -> 字符串 '{"a":1}'
const obj = JSON.parse(text);            // 字符串 -> 对象，非法 JSON 会抛错，要 try 包
```

### 关键机制速览

- **单线程加事件循环**：JS 本身只跑一条线程，长计算会冻结页面；异步靠的是把活交给浏览器/Node 环境，完事再把回调排回队列。
- **微任务优先**：每段同步代码跑完先清微任务队列（Promise 回调），再取下一个宏任务，所以 Promise 通常比 setTimeout 先执行。
- **类型转换三条路**：转字符串 `String(x)`、转数字 `Number(x)`、转布尔 `Boolean(x)`；隐式转换规则复杂，手动转最省心。
- **真值表要背熟**：`false、0、''、null、undefined、NaN` 是假值，其余（含空数组、空对象）全是真值。
- **NaN 的脾气**：`NaN !== NaN`，判断要用 `Number.isNaN(x)`；`typeof NaN` 是 'number'。
- **对象传引用**：函数收到对象参数时改属性，外面能看见；想不影响原对象先做一份拷贝再改。
- **浅拷贝只拷一层**：展开运算符和 `Object.assign` 都是浅拷贝，嵌套对象仍然共享（深拷贝另有办法，原文已述）。
- **this 由调用方式决定**：谁调用就指向谁；对象方法被单独取出来传走，this 就丢了（原文已述解决法）。
- **严格模式**：`'use strict'` 或模块默认开启，未声明变量赋值直接报错，很多静默错误变成显式异常。
- **相等判断**：`===` 不转类型先比类型再比值；`Object.is` 多处理了 NaN 与正负零两个边角。

### 新手易错点

- **数组下标与 length**：下标从 0 开始，最大合法下标是 `length - 1`；给不存在的下标赋值会把数组拉长并留空洞。
- **for...in 遍历数组**：它遍历的是键（字符串）还会带上额外属性，遍历数组值请用 for...of。
- **对象键顺序误解**：整数样键会按数值升序排在前面，字符串键按插入顺序，依赖顺序的业务别赌这个。
- **const 对象的误会**：const 只保证变量不重新赋值，对象属性照样能改；想真冻结用 `Object.freeze`（也只是浅冻结）。
- **字符串方法不改原串**：字符串不可变，`replace/slice` 都返回新串，不接返回值等于白调。
- **parseInt 不带进制**：`parseInt('08')` 在老环境有坑，养成 `parseInt(s, 10)` 的习惯。
- **switch 用严格相等**：`switch (x)` 内部是 `===` 比较，x 是字符串 '1' 时匹配不上 `case 1`。
- **return 后自动分号**：`return` 单独一行、值写在下一行，会被自动补分号变成返回 undefined；值跟 return 写同行。
- **在条件里写单个等号**：`if (a = 1)` 是赋值不是比较，还恒为真；比较永远写两个或三个等号并配合 lint。
- **定时器里的 this 与参数**：`setTimeout(obj.method, 100)` 会丢 this，包一层箭头函数 `() => obj.method()` 再传。

## 小技巧与冷知识

【小技巧】`??` 不能和 `&&` / `||` 不加括号混用：`a ?? b || c` 直接报 SyntaxError，规范强制要求写成 `(a ?? b) || c` 或 `a ?? (b || c)`。这是因为 `??` 和 `||` 的优先级容易让人误读，语言层面索性不让它含糊；混用时先加括号，意图写在括号里。

【小技巧】选兜底符先看“空值”长什么样：`??` 只兜底 `null` 和 `undefined`，`0`、`''`、`false` 会原样穿过；`||` 会把这三个也吞掉。计数 `0`、空字符串是合法值时用 `??`，想把所有“空”都当缺省才用 `||`；兜底符选错，`0` 分订单、空昵称这类合法值会被悄悄换成默认值。

【小技巧】给 fetch 做超时别只用 `Promise.race`，用 `AbortController`：`Promise.race([fetch(url), timeout])` 超时后请求本身还在后台跑，浪费连接；`AbortController` 的 signal 传给 fetch，超时调 `abort()` 能真正取消请求。超时控制要“取消请求”而不只是“不等结果”，否则超时的请求还在占着浏览器的并发连接数。

【小技巧】数组去重 `[...new Set(arr)]` 只对原始值有效，对象数组去重要按 key：`Set` 判断重复用的是 SameValueZero，两个字段相同的对象是两个引用，去重后一个都不少。对象数组先按业务 key 建 `Map`（`new Map(arr.map(x => [x.id, x]))`），再取 `map.values()`；“去重”之前先问一句“什么算重复”，答案是字段不是引用。

【小技巧】`JSON.parse` 必须包 try/catch，`JSON.stringify` 之前先想想要丢什么：非法 JSON 解析直接抛错，不包 try 整个流程就断了；序列化时函数、`undefined`、`Symbol` 会被静默丢掉，`Date` 变成字符串，`BigInt` 直接抛 TypeError。JSON 是数据交换格式不是通用序列化，含函数、BigInt、循环引用的对象进 JSON 之前先单独处理。

【小技巧】字符串转数字先分清 `Number()` 和 `parseInt/parseFloat`：`Number('')` 是 `0`，`parseInt('')` 是 `NaN`；`Number('12px')` 是 `NaN`，`parseFloat('12px')` 是 `12`。`Number` 要求整个字符串都是合法数字，`parseInt/parseFloat` 是“从头解析、遇到非法字符就停”；表单输入、URL 参数这种“可能带单位、可能为空”的字符串，用错函数得出的不是错值就是 NaN，查起来各不相同。

【小技巧】数字数组排序必须传比较函数：`Array.prototype.sort` 默认把元素转成字符串再按字典序排，`[10, 9, 80].sort()` 得到的是 `[10, 80, 9]`。数字排序写 `(a, b) => a - b`，对象数组按字段排写 `(a, b) => a.age - b.age`；不传比较函数的 sort，排数字永远是错的，错得还很像对的。

【小技巧】`setTimeout(..., 0)` 不是“立即执行”，是“排到宏任务队尾”：它要等当前同步代码和微任务队列全跑完才轮到；想“在同步代码后、定时器前”插一段逻辑，用 `queueMicrotask`。微任务和宏任务的先后顺序不是玄学，是事件循环的固定规则——Promise 回调、queueMicrotask 永远比 setTimeout 先跑。

【小技巧】列表里几百个按钮别逐个绑监听，用事件委托：父元素上绑一个点击监听器，靠 `event.target` 判断点的是哪个子元素。动态新增的子元素不用重新绑定，内存里也只有一个监听器；逐个绑定时新增元素还要补绑，漏一次就是一个点不动的按钮。事件冒泡不是只用来面试的，它就是干这个的。

【小技巧】`structuredClone` 拷不了函数和 DOM 节点：它能处理循环引用、Map/Set、Date，但遇到函数、DOM 节点会直接抛 `DataCloneError`。深拷贝之前先确认对象里有没有函数或 DOM 引用，有的话要么先剥离、要么手写拷贝逻辑；“官方深拷贝”也有边界，边界在“不可序列化”的东西上。

【小技巧】`??=`、`||=`、`&&=` 是逻辑赋值运算符：`a ??= b` 等价于“a 是 null/undefined 时才赋 b”，`a ||= b` 是“a 为假值时才赋 b”，常用来写默认值和懒初始化。它们比写满 `if` 的版本短，更重要的是意图直接写在运算符里；和 `??`/`||` 一样，`??=` 里的空值判断只认 null/undefined，`0` 和 `''` 不会被覆盖。

【小技巧】Node 里未处理的 Promise 拒绝会直接崩进程：async 函数里抛错没 catch，浏览器里只报一个 `unhandledrejection`，Node 15 起默认把未处理的拒绝当成未捕获异常、进程直接退出。顶层入口至少要有一个兜底 catch 或 `process.on('unhandledRejection')` 的日志，线上一个漏网的 reject 就能让整个服务重启。

【冷知识】JavaScript 是 Brendan Eich 在 1995 年大约 10 天内写出来的：最初在 Netscape 里叫 LiveScript，为蹭当时 Java 的热度才改名 JavaScript，名字和 Java 除了“都带 Java”之外没有血缘关系。语言 10 天赶工的出身，解释了它身上不少历史包袱——比如 `typeof null` 是 `'object'`，那是第一版实现里类型标签的 bug，留到今天谁也不敢改。

【冷知识】`[] + []` 是 `''`，`[] + {}` 是 `'[object Object]'`：加号遇到对象先调 `toString`/`valueOf` 再拼接，空数组转成空字符串，普通对象转成 `'[object Object]'`。控制台里 `{} + []` 还是 `0`，因为行首的 `{}` 被当成代码块而不是对象字面量；隐式转换的怪象，根子都在“加号先转字符串、再看上下文”这两条规则上。

【冷知识】`Array(3)` 建出来的是带空洞的数组，`map` 会直接跳过空洞：`Array(3).map((_, i) => i)` 得到的是 3 个空位而不是 `[0, 1, 2]`，因为 map 不会访问不存在的下标。想生成序列用 `Array.from({ length: 3 }, (_, i) => i)`，它建的是填满 `undefined` 的真数组；“长度为 3”和“有 3 个元素”在 JS 里是两回事。

【冷知识】`Math.max()` 不传参数返回 `-Infinity`，`Math.min()` 返回 `Infinity`：这两个函数内部是按“没有元素时取中性值”设计的，max 的中性值是最小的数，min 的中性值是最大的数。`Math.max(...[])` 展开空数组也是 `-Infinity`；求一列数的最大值之前先确认数组非空，否则拿到的 `-Infinity` 会一路算下去。

## 面试与自测

1. 并行调 10 个接口，1 个失败、9 个成功的结果也拿不到，原因是什么？原文给了哪两种解法？
2. `arr.forEach(async x => await fetch(x))` 后面代码先跑了、请求还没完，原因是什么？原文给了哪两种改法？
3. `sort` 和 `toSorted` 有什么区别？原文为什么说这个坑很多人踩过？

### 参考答案

1. 原因：`Promise.all` 遇到第一个 reject 就整体 reject。解法一：用 `Promise.allSettled` 拿全部结果再过滤；解法二：给每个 promise 包 `.catch` 兜底。反过来，如果“都要结果”才用 allSettled，只关心“任一失败全失败”才用 all。
2. 原因：forEach 不管回调返回的 Promise，async 回调只是“启动了”而已。改法一：换 `for...of` + await（串行）；改法二：`await Promise.all(arr.map(...))`（并行）。
3. `sort` 是原地改原数组，`toSorted`（以及 `toReversed`、`toSpliced`）是不改变原数组的版本。坑就在于用 `sort` 时以为拿到了新数组，实际上原数组已经被改了。
