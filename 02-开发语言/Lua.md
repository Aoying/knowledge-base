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
