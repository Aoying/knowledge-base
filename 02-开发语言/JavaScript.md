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
