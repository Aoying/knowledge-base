---
知识点: Lua
层级: 基础
适用版本: 通用
更新日期: 2026-10-22
关联:
  - 总览
  - Godot
标签:
  - 游戏
  - 实践
---

# Lua

轻量脚本语言，设计目标是"嵌入"：游戏引擎、Nginx（OpenResty）、Redis（EVAL）都内嵌 Lua 做扩展逻辑。学 Godot 之余了解 Lua，对理解"宿主+脚本"架构有帮助。

## 语法速查

```lua
-- 注释用 --
local x = 10          -- 变量默认全局，local 才是局部（必须写）
if x > 5 then print("big") end
for i = 1, 10 do print(i) end      -- 1..10 闭区间
while x > 0 do x = x - 1 end

function add(a, b) return a + b end
local f = function(a) return a * 2 end  -- 函数是一等公民
```

## 表（Table）：唯一的复合结构

数组、字典、对象全用 table：

```lua
local arr = {10, 20, 30}   -- 数组，下标从 1 开始！
print(arr[1])              -- 10
local map = {name="ao", age=5}
print(map.name)            -- ao
```

## 元表与元方法（Metatable）

给 table 定制行为，相当于"运算符重载"：

```lua
local t1 = setmetatable({1,2}, {
  __add = function(a, b)  -- 定义 + 的行为
    local r = {}
    for i=1,#a do r[i] = a[i] + b[i] end
    return r
  end
})
```

常用元方法：`__index`（读不到 key 时兜底）、`__newindex`（写拦截）、`__call`（table 当函数调用）。

## 协程（Coroutine）

```lua
local co = coroutine.create(function()
  for i=1,3 do print("co", i); coroutine.yield() end
end)
coroutine.resume(co)  -- co 1
coroutine.resume(co)  -- co 2
```

协作式多任务，游戏里做"等待 N 秒再继续"、分帧加载就靠它。

## 与宿主交互

- C/C++ 通过 Lua C API 把函数注册给 Lua 调用（`lua_register`）
- Redis：`EVAL "return redis.call('get', KEYS[1])" 1 mykey`
- OpenResty：`content_by_lua_block` 直接写业务逻辑

## 常见坑

1. **下标从 1 开始**：`arr[0]` 是 nil，不是报错，静默坑。
2. **忘记写 local**：变量默认全局，函数里漏写 local 会污染全局命名空间。
3. **`nil` 会删 key**：`t.k = nil` 等于删除这个 key，`#t` 取长度时"空洞"会导致长度不准。
4. **字符串不可变**：频繁 `..` 拼接大字符串很慢，用 `table.concat`。
5. **`0` 和空字符串都是真**：只有 `nil` 和 `false` 是假，和其他语言反直觉。

## 疑难杂症

### 1. table 长度 `#t` 莫名其妙
- **现象**：往 table 里删了几个元素，`#t` 返回的值忽大忽小。
- **原因**：`#` 只对"序列"（1..n 连续无 nil）有定义；中间有 nil 空洞时结果未定义。
- **解决**：需要精确计数自己维护 counter，或用 `table.move` 压缩后再取长度。

### 2. 协程 resume 后没反应，也不报错
- **现象**：`coroutine.resume(co)` 调用了，函数体没执行。
- **原因**：协程已经 dead（函数 return 了）再 resume 会静默返回 false；或者 yield/resume 配对错乱。
- **解决**：resume 前查 `coroutine.status(co)`；调试时打印 resume 的第二个返回值（错误信息）。

## Redis Lua 实战

Redis 里 Lua 跑在服务端，原子执行，适合"读+写"必须一起成功的场景。

### 限流（令牌桶简化版：固定窗口计数）
```lua
-- KEYS[1]=限流key, ARGV[1]=窗口内最大次数, ARGV[2]=窗口秒数
local c = redis.call('INCR', KEYS[1])
if c == 1 then redis.call('EXPIRE', KEYS[1], ARGV[2]) end
return c <= tonumber(ARGV[1]) and 1 or 0
```
```bash
redis-cli --eval limit.lua ratelimit:api , 100 60
```

### 要点
- 脚本里别做耗时操作，Redis 单线程，慢脚本阻塞所有请求。
- `redis.replicate_commands()` / effects：5.0+ 脚本的写复制更可靠，老版本用 `redis.call` 即可。
- 调试：`redis-cli --eval` 本地先跑，生产用 `EVALSHA` 省带宽。

## 元表实战：只读表

```lua
function readonly(t)
    return setmetatable({}, {
        __index = t,
        __newindex = function() error("只读表，不许改") end,
        __metatable = "locked",
    })
end
local cfg = readonly({host="127.0.0.1", port=6379})
print(cfg.host)   -- 正常读
-- cfg.host = "x" -- 报错
```

- `__index` 可以是表（查不到就去那张表找）也可以是函数（动态计算默认值）。
- `__newindex` 拦截所有"新增键"赋值，想做校验/只读就靠它。
- 注意：`rawset/rawget` 绕过元表，框架内部用，业务代码别碰。

## Lua 调 C 最小示例（理解"宿主+脚本"架构）

```c
// mylib.c
#include <lua.h>
#include <lauxlib.h>
static int ladd(lua_State *L) {
    lua_pushnumber(L, lua_tonumber(L,1) + lua_tonumber(L,2));
    return 1;  // 返回值个数
}
int luaopen_mylib(lua_State *L) {
    luaL_Reg f[] = {{"add", ladd}, {NULL, NULL}};
    luaL_newlib(L, f);
    return 1;
}
```
```lua
local m = require("mylib")
print(m.add(1, 2))  -- 3
```

- `lua_State` 是 Lua 虚拟机的句柄，C 和 Lua 通过"栈"传参/返回值。
- OpenResty/Redis 内嵌 Lua 都是这个原理：宿主把能力（发 HTTP、读写 key）注册成 Lua 函数。
- 学 Godot 时对照看：GDScript 调 C++ 引擎也是同一套"脚本 glue 原生"的思想。

## OpenResty 实战：网关层限流

```lua
-- nginx.conf: access_by_lua_block
local limit = require "resty.limit.req"
local lim, err = limit.new("my_limit", 10, 20)  -- 10r/s, burst 20
local key = ngx.var.binary_remote_addr
local delay, err = lim:incoming(key, true)
if not delay then
    if err == "rejected" then return ngx.exit(503) end
    ngx.log(ngx.ERR, "limit err: ", err)
    return ngx.exit(500)
end
```

- OpenResty = Nginx + LuaJIT，在网关层做限流/鉴权/灰度，比回源到 Java 省一跳。
- `lua-resty-*` 库生态很全（redis、mysql、http 都有），别自己造轮子。

## 基础知识补强

### 核心概念与术语

- **八种类型**：`nil`（空）、`boolean`、`number`、`string`、`function`、`table`、`userdata`（宿主对象）、`thread`（协程），用 `type(x)` 查。
- **nil 的含义**：表示"不存在"。变量未赋值就是 nil；给全局变量赋 nil 等于删掉它；table 里不存在的键读出来是 nil。
- **number 一统天下**：经典 Lua 里整数和小数同为 number（5.3 起分 integer 与 float，以官方文档为准），除法默认可能得小数，取整用 `math.floor`。
- **字符串不可变**：字符串一旦创建不能改，拼接 `..` 生成新串；长文本、字节数据都靠 string 库处理。
- **函数即值**：函数可以赋值、传参、返回，匿名函数与命名函数没有本质区别，这是回调与模块写法的基础。
- **全局环境表 _G**：所有全局变量实际存在 `_G` 这张表里，`_G.x` 与直接写 `x` 等价；污染全局就是往这张表里乱塞键。
- **多返回值**：函数可以一次返回多个值，调用方用多个变量接收，这是 Lua 区别于多数语言的标志性机制。
- **可变参数**：形参写 `...` 表示任意多个参数，函数内用 `{...}` 打包成表或 `select` 读取。
- **逻辑运算符的返回值**：`and`、`or` 返回的是操作数本身而非布尔值，所以 `a or 默认值` 是地道的兜底写法。
- **短字符串与模式**：Lua 自带轻量 pattern（模式匹配），长得像正则但能力弱一截，不是正则表达式，别套正则语法。
- **模块**：一个返回 table 的文件就是模块，`require` 加载并缓存，同名模块全局只执行一次。
- **尾调用**：函数最后一步是直接返回另一个函数调用时不额外占栈，深递归靠它防爆栈。

### 基础语法与用法

```lua
-- 类型自查与字面量
print(type(10))            --> number
print(type("hi"))          --> string
print(type(nil))           --> nil
print(type(print))         --> function

local s = "双引号字符串"
local long = "第一行\n第二行"   -- \n 是换行转义；另有长字符串写法：
                               -- 用一对双方括号包住多行文本，内部换行原样保留
print(#s)                  -- # 取长度（字符串按字节数，见易错点）
```

```lua
-- 运算符：注意不等于和拼接的写法
print(1 ~= 2)              -- 不等于是 ~=，没有 != 
print("a" .. "b")          -- 字符串拼接是 ..，没有 +
print(7 % 3)               -- 取模 -> 1
print(2 ^ 10)              -- 幂 -> 1024.0
local ok = x and "有值" or "无值"  -- and/or 模拟三元运算（x 为真时成立）
```

```lua
-- 控制流全形态
if n > 0 then
  print("正")
elseif n == 0 then         -- 是 elseif 连写，没有 else if
  print("零")
else
  print("负")
end

for i = 1, 5, 2 do print(i) end        -- 数值 for：起, 止, 步长 -> 1 3 5
repeat                             -- 至少执行一次
  n = n - 1
until n == 0                         -- 条件为真时停（与其他语言 while 取反）

for k, v in pairs({a=1, b=2}) do print(k, v) end  -- pairs 遍历全部键
for i, v in ipairs({10, 20}) do print(i, v) end   -- ipairs 只走数组部分
```

```lua
-- 函数：多返回值与可变参数
local function divmod(a, b)
  return a // b, a % b       -- 整除 // 是 5.3+ 语法，老版本用 math.floor(a/b)
end
local q, r = divmod(17, 5)   -- q=3, r=2；少接的丢弃，多接的补 nil

local function sum(...)
  local total = 0
  for _, v in ipairs({...}) do total = total + v end
  return total
end
print(sum(1, 2, 3))          --> 6
```

```lua
-- 字符串与常用标准库速览
print(string.format("hp=%d/%d", 80, 100))   -- 格式化，像 C 的 printf
print(string.sub("hello", 2, 4))            -- 截取 -> ell（下标从 1 开始）
print(string.upper("abc"))                  --> ABC
print(math.max(3, 9), math.floor(3.7))      -- 9 与 3
print(os.time())                            -- 当前时间戳
local parts = {"a", "b", "c"}
print(table.concat(parts, "-"))             --> a-b-c
table.insert(parts, "d")                    -- 往数组尾部加
print(#parts)                               --> 4
```

```lua
-- 错误处理：pcall 把"抛错"变成"返回状态"
local ok, result = pcall(function()
  return 10 / 0   -- Lua 里除零得 inf 不报错，这里示意调用可能失败的函数
end)
if not ok then
  print("失败原因: " .. tostring(result))
end
-- error("主动抛错") 用来报告非法状态；assert(cond, msg) 是它的常用封装
local v = assert(tonumber("42"), "不是数字")  -- 成功返回 42，失败直接抛错
```

### 关键机制速览

- **万物皆值、变量无类型**：类型在值上，变量只是名字，今天装数字明天装字符串都行（但别这么写）。
- **table 双重身份**：连续整数键的部分是"数组段"，其余键是"哈希段"，同一个 table 可以同时当数组和字典。
- **键的规则**：除 nil 和 NaN 外，任何值都能当键；浮点数当键有精度风险，业务里老实用字符串和整数键。
- **赋值是引用**：table 和函数赋值传的是引用，两份变量指向同一张表，改一个另一个看得见。
- **比较语义**：数字与字符串之间不自动转换比较，`10 == "10"` 为 false，混合比较大小（如 `10 < "a"`）直接报错。
- **字符串与数字的半自动转换**：算术运算会把数字样字符串转成数字（`"10" + 1` 得 11），但拼接数字会自动转字符串；依赖它写代码容易翻车，显式 `tonumber/tostring` 最稳。
- **作用域**：`local` 声明从声明处生效到所在块结束；函数是闭包，能捕获并改外层 local（upvalue）。
- **迭代器协议**：`pairs/ipairs` 返回的是迭代函数，generic for 反复调用它直到返回 nil，理解这点就理解 for 的本质。
- **模式匹配不是正则**：pattern 里 `%d` 是数字类、`.` 表示任意一字符，不支持 `|` 分组选择等正则能力（对照正则笔记看差异）。
- **require 的查找**：按 `package.path` 里的模板逐个找文件，找不到把尝试过的路径全列在报错里，排障顺着它看。

### 新手易错点

- **`#` 对字符串数的是字节**：UTF-8 中文一个字占 3 字节，`#"中文"` 结果是 6 不是 2，按字符数要用 utf8 库（5.3+ 自带）。
- **数值 for 的边界只算一次**：循环中途改终点变量不影响循环次数，和某些语言直觉不同。
- **pairs 顺序不保证**：哈希段遍历顺序任意，两次运行都可能不同；要顺序就自己收集键排序。
- **ipairs 遇 nil 就停**：数组中间有空洞时后面的元素遍历不到，这不是 bug 是定义。
- **多返回值被截断**：函数调用出现在表达式中间（如 `f(print(g()))` 之外的拼接、表构造中部）时只取第一个返回值，只有放在最后一项才全部展开。
- **`and/or` 模拟三元的坑**：`cond and A or B` 在 A 是 false 或 nil 时会误落到 B，此场景改老实写 if。
- **误用 `==` 比较浮点**：小数运算有误差，比较前先约定精度（如差值小于 1e-9 视为相等）。
- **`local` 写在 if 块里出块就没了**：想要外层可见的值，先在块外声明再在块内赋值。
- **循环里拼字符串**：每轮 `..` 都生成新串，长循环很慢，先塞 table 最后 `table.concat` 一次成型。
- **全局变量拼写打错不报错**：读未声明的全局变量只得到 nil，bug 会潜伏很深；重要模块养成测 `type()` 的习惯。
