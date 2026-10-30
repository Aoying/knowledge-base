---
知识点: Lua
层级: 基础
适用版本: 通用
更新日期: 2026-10-31
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

## 小技巧与冷知识

【小技巧】往数组尾部追加元素可以直接写 `t[#t + 1] = v`：它和 `table.insert(t, v)` 效果一样，热循环里少一次函数调用。`table.insert` 的优势在能指定位置插入，纯追加用下标写法更直白；读代码时看到 `#t + 1` 就知道"这是往尾巴上加"，不用再猜 insert 的第二个参数是不是位置。

【小技巧】遍历中删元素要倒序走：正序遍历时删掉 `t[i]`，后面的元素会前移一位，下一轮 `i + 1` 直接跳过本该检查的那一个。倒序遍历（`for i = #t, 1, -1 do`）删了后面的不影响前面的下标，删多少都不会漏；"边遍历边删"这个动作本身没错，错的是方向。

【小技巧】数可变参数的个数用 `select('#', ...)`，别用 `#{...}`：`{...}` 打包时尾部的 nil 会被丢掉，`f(nil, nil)` 打包后长度是 0；`select('#', ...)` 返回的是实参个数，尾部 nil 也算在内。函数签名里"第几个参数是 nil 本身就是信息"时（比如占位参数），只有 select 能数对。

【小技巧】判断一张表是不是空表用 `next(t) == nil`：`#t == 0` 只对数组段可靠，字典部分有键时 `#t` 照样是 0；`next` 取的是表中"任意一个"键值对，没有就返回 nil。空表判断、"这张表还有没有剩下的数据"这类问题，next 是一行到位的写法，不用先分表是数组还是字典。

【小技巧】`__index` 写成函数时，查原表要用 `rawget(t, k)`：在 `__index` 函数里再写 `t[k]`，读不到又会触发一次 `__index`，直接无限递归。`rawget` 绕过元表读原始值，是元方法内部的"安全读"；同理 `__newindex` 里写值要用 `rawset`，否则赋值又会触发一次拦截，同样死循环。

【小技巧】改了模块源码想热重载，先清缓存再 require：`require` 的结果存在 `package.loaded[模块名]` 里，第二次 require 直接返回缓存、不重新执行文件。开发调试时写 `package.loaded["mymod"] = nil; require("mymod")` 就能强制重新加载；生产代码别这么玩，缓存是 require 只执行一次的保证。

【小技巧】想拿到完整错误堆栈用 `xpcall` 而不是 `pcall`：`pcall` 只返回"失败 + 错误信息"，出错位置靠猜；`xpcall(f, debug.traceback)` 会在出错时把调用栈拼进错误信息里。宿主程序里跑用户脚本、游戏里跑 Mod 脚本时，堆栈是定位问题的关键，pcall 只适合"失败了就走兜底、不关心哪行错"的场景。

【小技巧】`coroutine.wrap` 和 `coroutine.create` 的出错方式不一样：create 返回一个线程，出错时 `resume` 返回 false 加错误信息；wrap 直接返回一个函数，调用它出错会像普通函数一样把错误抛出来。想把协程当"惰性生成器"用（每次调用取下一个值），wrap 更顺手；想自己控制错误处理、resume 返回值里分情况处理，用 create。

【小技巧】`coroutine.yield` 是双向传值的通道：`resume(co, 参数)` 里多余的参数会变成 `yield` 的返回值，`yield(值)` 传出的值会变成 `resume` 的多余返回值。生产者-消费者、"协程问一句、主流程答一句"的对话式逻辑就靠这条通道；只把 yield 当"暂停键"用，等于只用了它一半的能力。

【小技巧】Redis 脚本里调可能失败的命令用 `redis.pcall` 而不是 `redis.call`：`redis.call` 出错会直接中断整个脚本，`redis.pcall` 把错误变成一个带 `err` 字段的返回值，脚本可以自己判断"这个 key 不存在就跳过"。批量处理一批 key 时，一个坏 key 不应该拖垮整批，pcall 是脚本里的"try-catch"。

【小技巧】Redis 脚本里别用 `math.random` 和 `os.time()`：脚本会被复制到从库重放，主从必须算出完全相同的写操作，随机数和当前时间每次运行都不一样，重放结果就对不上。需要随机或时间时从 ARGV 传进来，让调用方决定；脚本本身必须是"同样的输入、永远同样的输出"的纯函数。

【小技巧】热路径里高频用的全局函数先局部化：`local print = print`、`local insert = table.insert` 这种写法不是多此一举——全局变量存在 `_G` 表里，每次调用都要查一次表，局部变量是寄存器访问，循环里调用上万次差距就出来了。游戏帧循环、Redis 脚本这类"小函数跑很多遍"的场景，文件开头局部化几个常用函数是 Lua 社区的惯用写法。

【小技巧】`table.sort` 不保证稳定排序：相等元素的相对顺序在排序后可能变，打乱了就是打乱了。业务上"分数相同按先到先得"时，比较器里要加原始下标做第二排序键（先比分数、再比序号），只靠 sort 本身保不住顺序；稳定是"实现没承诺"的事，不是"这次碰巧没乱"。

【小技巧】`..` 的优先级比加减法低：`"结果：" .. 1 + 2` 的结果是 `"结果：3"`，因为 `+` 先算。想拼的是 `"结果：1"` 再加 2，意思完全是两回事——拼接表达式里混算术时先加括号，`("结果：" .. 1) + 2` 和 `"结果：" .. (1 + 2)` 写清楚了才不会被优先级坑。

【小技巧】拼接字符串前先确认值不是 nil：`"分数：" .. score` 里 score 是 nil 会直接报错"attempt to concatenate a nil value"，整个流程断掉。可能为 nil 的值先包一层 `tostring(v)`（nil 会变成字符串 `"nil"`），或者用 `or` 给默认值；`..` 不做任何兜底，nil 一进来就炸。

【小技巧】做缓存用弱表（weak table），别手写清理逻辑：`setmetatable(cache, {__mode = "v"})` 后，值一旦在别处没有引用、被 GC 回收，缓存里对应的条目会自动消失。值是"大对象、别处用完就该释放"的缓存最适合这招；key 是临时对象时用 `"k"`，键值都弱引用用 `"kv"`，模式写在元表的 `__mode` 字段里。

【小技巧】`math.random` 用之前先调 `math.randomseed`：不设种子时每次启动的随机序列都一样，抽奖、洗牌、随机刷怪全是"伪随机"的固定剧本。种子一般取 `os.time()`，精度要求高时再混一个进程相关的数；只调一次就够，放在程序入口，别每用一次播一次种。

【小技巧】`string.gsub` 的第二个返回值是替换次数：`local s, n = string.gsub(text, "old", "new")` 里 `n` 就是实际替换了几处。统计"这段文本里有多少个敏感词被替换"、校验"模板占位符是不是恰好替换了一次"时，不用再单独数一遍，gsub 顺手就给了。

【小技巧】想区分整数和浮点数用 `math.type`（5.3+）：`math.type(3)` 返回 `"integer"`，`math.type(3.0)` 返回 `"float"`，`math.type("3")` 返回 nil。`type()` 只能告诉你是 number，分不清整浮；序列化、和 C 交互、判"这个值能不能当数组下标"时，math.type 才是准确的那把尺子。

【冷知识】字符串会被"驻留"（intern）：内容相同的字符串在内存里只存一份，`==` 比较字符串时不用逐字节比对，基本是指针比较。这也是 Lua 里字符串比较、表里用字符串当键都很快的原因——代价是创建新字符串时要先查 intern 表，超长字符串拼接慢，一半原因在这里。

【冷知识】`"abc":upper()` 这种写法能工作，是因为字符串类型共享一张元表：`getmetatable("").__index` 就是 string 库本身，所以字符串字面量可以直接用冒号调 string 库的函数。`("hello"):upper()` 和 `string.upper("hello")` 是同一回事，冒号语法不是字符串的"方法"，是元表搭的桥。

【冷知识】Lua 的取模 `%` 符号跟除数，不跟被除数：`-7 % 3` 的结果是 `2`，而 C 语言里是 `-1`。Lua 的定义是 `a % b == a - math.floor(a / b) * b`，余数永远和除数同号；从其他语言转过来写"取模判负数"、"环形下标回绕"时，这个符号差异是最容易翻车的地方。

【冷知识】幂运算符 `^` 是右结合的：`2^3^2` 算出来是 `512`（即 `2^(3^2)`），不是 `64`。和其他二元运算符左结合不同，`^` 从右边开始结合；写连续幂运算不加括号，读代码的人很容易按左结合理解错。

【冷知识】Lua 没有 `continue` 语句：想跳过本次循环剩下的部分，要么把逻辑包进 `if`，要么用 `goto` 跳到循环末尾的标签（`::continue::`，5.2+ 才有 goto）。`break` 只能跳出最内层循环，"跳过一次"在 Lua 里没有专门的关键字，是语言有意保持的小。

【冷知识】OpenResty 用的 LuaJIT 是 Lua 5.1 方言：`//` 整除、`utf8` 库、整数类型这些 5.3 的东西在默认配置下都不可用，写 OpenResty 脚本要按 5.1 的规矩来；反过来 LuaJIT 自带 FFI 可以直接调 C 函数，这是标准 Lua 没有的能力。同一门语言，在 Redis、OpenResty、游戏引擎里可能是三个不同的"版本"，动手前先确认宿主用的是哪个实现。

【冷知识】Lua 的名字是葡萄牙语里的"月亮"：它诞生于巴西里约天主教大学（PUC-Rio），Lua 不是缩写，读作"卢阿"而不是逐字母念"L-U-A"。前身语言叫 SOL（葡萄牙语"太阳"），作者换新名字时刻意选了"月亮"来呼应。

【冷知识】函数可以序列化成字节码再加载回来：`string.dump(func)` 把一个 Lua 函数编译成字节码字符串，`load` 能把它重新加载成可调用的函数。热更新、把编译好的脚本存进数据库、网络传输代码片段都靠这招；但字节码和版本强绑定，5.3 编的字节码拿到 5.4 上加载不了。

## 面试与自测

1. Lua 里 `arr[0]` 的结果是什么？为什么说这是个静默坑？
2. `#t` 在什么情况下返回值未定义？原文给了哪两种应对办法？
3. Redis 里 Lua 脚本为什么不能做耗时操作？调试和生产环境原文分别推荐用什么方式跑脚本？

### 参考答案

1. 结果是 nil，而且不报错。Lua 数组下标从 1 开始，`arr[1]` 才是第一个元素；按其他语言习惯写 `arr[0]` 不会报错，只会静默拿到 nil，bug 藏得很深。
2. `#` 只对“序列”（1..n 连续无 nil）有定义；table 中间有 nil 空洞时，`#t` 的结果未定义，删了几个元素后返回值忽大忽小。应对办法：需要精确计数自己维护 counter，或用 `table.move` 压缩后再取长度。
3. Redis 是单线程，脚本在服务端原子执行，慢脚本会阻塞所有请求。调试用 `redis-cli --eval` 本地先跑；生产用 `EVALSHA` 省带宽。
