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
