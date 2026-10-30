---
知识点: 字典树Trie
层级: 进阶
适用版本: 通用
更新日期: 2026-10-31
关联:
  - 数据结构与算法
  - 字符串算法
标签:
  - 计算机基础
  - 原理
  - 实践
---

# 字典树Trie

以 Trie（字典树、前缀树）为主的深篇。主篇 [数据结构与算法](计算机基础/算法/数据结构与算法.md) 只给了提纲（前缀搜索、敏感词过滤）；字符串匹配的另一路 KMP 见 [字符串算法](计算机基础/算法/字符串算法.md)，本篇不重复。

## Trie 是什么

Trie 是一棵字符树：从根到某个节点的路径拼出来的就是一个前缀；节点只存“有没有这个孩子”和“是不是词尾”。

- 根节点不存字符，代表空前缀。
- 每个节点两个信息：孩子指针（字符 → 子节点）、is_end（到这里是否正好是一个词）。
- 公共前缀只存一份：apple、app、apricot 前两个字符 a、p 共享同一条链，这是 Trie 省空间的全部来源。

## 插入、查询、前缀统计

插入：沿字符逐层走，没有孩子就新建；走完把最后一个节点标 is_end。查询：沿字符走，走到头且 is_end 才算命中。加一个 count 字段（经过该节点的词数），前缀统计就不用枚举子树，直接读节点 count。

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False
        self.count = 0  # 经过该节点的词数

class Trie:
    def __init__(self):
        self.root = TrieNode()
    def insert(self, word):
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
            node.count += 1
        node.is_end = True
    def search(self, word):
        node = self._walk(word)
        return node is not None and node.is_end
    def starts_with(self, prefix):
        return self._walk(prefix) is not None
    def count_prefix(self, prefix):
        node = self._walk(prefix)
        return node.count if node else 0
    def _walk(self, s):
        node = self.root
        for ch in s:
            node = node.children.get(ch)
            if node is None:
                return None
        return node
    def autocomplete(self, prefix):
        node = self._walk(prefix)
        if node is None:
            return []
        out = []
        def dfs(n, path):
            if n.is_end:
                out.append(path)
            for ch, child in n.children.items():
                dfs(child, path + ch)
        dfs(node, prefix)
        return out

t = Trie()
for w in ["apple", "app", "apricot", "banana"]:
    t.insert(w)
print(t.search("app"))          # True
print(t.search("ap"))           # False
print(t.starts_with("ap"))      # True
print(t.count_prefix("ap"))     # 3
print(t.count_prefix("app"))    # 2
print(t.autocomplete("ap"))     # ['app', 'apple', 'apricot']
print(t.autocomplete("ba"))     # ['banana']
```

### worked 例

词库 `apple、app、apricot、banana`：

- `search("app")` 是 True（app 本身是词），`search("ap")` 是 False（ap 只是前缀，不是词）——is_end 就是干这个的。
- `count_prefix("ap")` 是 3（apple、app、apricot 都经过 ap 节点），`count_prefix("app")` 是 2（只有 apple、app 经过 app 节点，apricot 在第二个 p 后走了 r 分支）。
- `autocomplete("ap")` 从 ap 节点往下枚举所有带 is_end 的路径，得 `['app', 'apple', 'apricot']`。

## 自动补全与词频场景

- 自动补全：定位到前缀节点，DFS 枚举子树里所有 is_end 路径；实际产品里一般截断取前 N 个（按词频排序），不要全量返回。
- 词频统计：count 字段天然就是“以该前缀开头的词数”；要“某个词出现几次”，再给 is_end 节点加一个 end_count，插入重复词时累加。
- 敏感词过滤：把敏感词全插进 Trie，扫描文本时每个位置沿 Trie 走一步，走到 is_end 即命中。一次扫描 O(文本长)，比对每个敏感词各做一次 str.find 快得多；这也是总纲里“敏感词过滤”一行对应的实现。
- 联想输入、IP 路由（按比特前缀匹配）、拼写纠错的候选生成，都是同一套“前缀定位 + 子树枚举”。

## 复杂度与代价：和哈希、排序数组比

| 操作 | Trie | 哈希表 | 排序数组 + 二分 |
| --- | --- | --- | --- |
| 精确查询 | O(L) | 平均 O(1) | O(log N × L) |
| 前缀判断 | O(L) | 不支持 | O(log N × L) |
| 前缀统计 / 补全 | O(L + 结果规模) | 不支持 | O(log N × L + 结果规模) |
| 插入 | O(L) | 平均 O(1) | O(N)（要挪位置） |

L 是词长，N 是词库规模。Trie 的查询代价与词库规模无关，只与词长有关——词库从一万涨到一千万，查一个词还是走 L 步。

代价在空间：最坏每个词独占一条链，节点数 = 所有词长之和；公共前缀越多省得越多。字母表很小（如只 26 个小写字母）时孩子可用定长数组存，下标即字符，查找更快但空节点多；通用字符（中文、Unicode）用字典存孩子，省空间。

## 易错点

- 孩子用字典还是数组：字母表固定且小用数组，通用字符用字典；选错要么浪费空间，要么写起来啰嗦。
- 删除词不能只把 is_end 改 False 就完事：还要沿路清理不再被任何词共享的孤链，否则删掉的词的前缀节点一直残留（空间泄漏）；偷懒的做法是只改 is_end，接受残留。
- 空串：约定空串是否算一个词，插入和查询要一致，否则 search("") 的结果全看实现心情。
- 前缀统计的 count 语义：是“经过该节点的词数”还是“以该节点为词尾的词数”，写代码前先定死；上面代码用的是前者。

## 小技巧与冷知识

【小技巧】原文代码有个容易忽略的坑：`count_prefix("")` 返回 0 而不是 4，因为 `insert` 只给经过的子节点累加 count，根节点的 count 从来没动过。想让空前缀返回词库总量，要么在 `insert` 里顺手给根节点也加 1，要么单独维护一个总数；写前缀统计前先确认“空前缀”在你的语义里算什么，别等上线才发现计数对不上。

【小技巧】删词时别只把 `is_end` 改 False，顺手把沿路的 count 也减 1：count 减到 0 的节点就是“再也没有词经过这里”，可以放心剪掉整条孤链。原文只说了“要清理孤链”，而 count 字段正是判断“能不能剪”的现成依据——有 count 就不用再写一套“子树里还有没有词”的递归检查。

【小技巧】自动补全想按字典序返回，不能直接遍历 `children` 字典：Python 字典按插入顺序迭代，原文例子里 `autocomplete("ap")` 得到 `['app', 'apple', 'apricot']` 只是因为插入顺序恰好是字典序。DFS 时对 `children` 的键排个序再遍历，返回顺序才稳定；同理，按词频排序时要让 `is_end` 节点带上词频，遍历完再按词频排，别指望遍历顺序自带优先级。

【小技巧】Trie 只能查前缀，想查后缀要反过来建树：把每个词倒序插进另一棵 Trie，查后缀 "ple" 就变成查前缀 "elp"。域名后缀匹配、文件后缀归类这类“以后缀定类别”的场景都用这招；一棵 Trie 管前缀、一棵倒序 Trie 管后缀，别在一棵树上硬套。

【小技巧】大小写和 Unicode 归一化要在插入和查询两边同时做：`insert("Apple")` 再 `search("apple")` 是查不到的，因为 'A' 和 'a' 走了不同的分支。中文场景还有一层坑：`café` 的 é 可以是单个码点，也可以是 e 加组合重音，两者肉眼一样、字节不一样，不经过 `unicodedata.normalize("NFC", s)` 归一化就会被当成两个不同的词；插入和查询必须用同一套归一化，缺一边就等于白做。

【小技巧】自动补全只取前 N 个时，别先全量枚举再切片：子树很大时全量 DFS 再取前 10 是浪费，边遍历边用一个大小为 N 的小顶堆留住当前最优的 N 个，或者在每个节点上直接存“该子树里最热的 N 个词”。原文只提醒了“截断取前 N 个”，没说怎么截——全量枚举再截断在词库大时就是性能坑。

【小技巧】通配符查询（如 `.` 匹配任意一个字符）不要在 Trie 外面拆成多个前缀查：在 `.` 所在的那一层对所有孩子分支都递归下去，命中 is_end 再收结果。写成“枚举所有可能字符再逐个查前缀”等于把 Trie 退化成暴力，分支递归才是 Trie 该有的写法；通配符一多要再加剪枝，否则分支数会炸。

【小技巧】Python 里节点类加 `__slots__` 能省不少内存：每个 `TrieNode` 默认带一个 `__dict__`，词库上百万时光是节点对象的字典开销就很可观；`__slots__ = ("children", "is_end", "count")` 之后每个节点只剩三个字段的槽位。字母表很小时还可以把 `children` 换成定长列表或数组，26 个小写字母直接按下标记位置，查找从哈希变成下标。

【小技巧】敏感词过滤时别在每个位置都从根重新走一遍还顺带回溯：文本扫描的正确姿势是“每个位置沿 Trie 往下走，走到 is_end 即命中、走不动就换下一个起始位置”。更进一步的优化是 Aho-Corasick：给 Trie 加失败指针，文本只扫一遍、失配时顺着失败边跳转，不用回退起始位置；原文的“一次扫描 O(文本长)”是理想情况，朴素实现里每个位置都可能重复走前缀，Aho-Corasick 才是把它落到实处的结构。

【小技巧】前缀统计的 count 要分清“插入次数”和“去重词数”：同一个词插两遍，count 会加两次，`count_prefix` 统计的是“词条出现次数”而不是“不同词的个数”。词频场景要的就是前者，自动补全去重时要的是后者；需求不一样，insert 里累加的字段就要分开记，别拿一个 count 两头用。

【冷知识】Trie 的名字来自 retrieval（检索），发明者 Edward Fredkin 在 1960 年的论文里提出，读音按词源读 "try" 而不是 "tree"。它也常被叫前缀树（prefix tree），但 Trie 这个名字本身就藏着它的用途——为检索而生的树。

【冷知识】压缩版的 Trie 叫基数树（radix tree，也叫 Patricia trie）：把只有一个孩子的链压缩成一条边，`apple` 和 `apricot` 共享的 `ap` 不再是两个节点，而是一条标着 "ap" 的边。节点数大幅减少，代价是插入和删除时要处理边的分裂与合并，Linux 内核、Redis 的某些结构、IP 路由表用的都是这类压缩变体。

【冷知识】Trie 还能再压缩成双数组（double-array trie）：用两个并行数组 base 和 check 代替指针和字典，把整棵树压成两个整数数组，内存占用和缓存友好度都更好。日文分词工具 MeCab、很多输入法的词库用的就是双数组 Trie，它是“空间换时间”的反面——结构紧凑到可以整体 mmap 进内存直接用。

【冷知识】IP 路由查表用的是二叉 Trie，查的不是“是不是这个词”而是“最长前缀匹配”：路由表里的每条规则是一段比特前缀，查一个 IP 地址要找到能匹配的最长那条，而不是精确命中。原文只提了一句“IP 路由按比特前缀匹配”，背后的关键点是“最长”二字——前缀越长、路由越具体，优先级越高。

【冷知识】Trie 和 DAWG（有向无环词图）是“亲戚”：DAWG 把 Trie 里结构相同的后缀也合并起来，`apple` 和 `apply` 的尾部可以共享，词库占用的节点比 Trie 更少。代价是 DAWG 基本是只读结构，建好之后很难再插入新词，适合词库固定不变的场景（如拼写检查的词典），Trie 则适合要频繁增删的场景。

【冷知识】自动补全的鼻祖之一是 1970 年代的 Emacs 动态缩写（dabbrev）：它没有 Trie，只是扫描缓冲区里出现过的词做补全；真正把 Trie 做成产品级补全的是搜索引擎的查询建议。补全的体验差距往往不在 Trie 本身，而在“节点上存不存词频、排不排序”这些原文没展开的工程细节上。

## 面试与自测

1. Trie 查询一个词为什么是 O(L) 而与词库规模无关？插入时 count 字段起什么作用？
2. 前缀 "ap" 下有 apple、app、apricot 三个词，count_prefix("app") 为什么是 2 不是 3？
3. 什么场景用 Trie 而不用哈希表？什么场景反而别用 Trie？

### 参考答案

1. 查询沿字符逐层走，每层一次孩子查找，步数只等于词长 L，与词库词数 N 无关。count 记录经过该节点的词数，前缀统计直接读节点 count，不用枚举整棵子树。
2. count_prefix 读的是 "app" 对应节点的 count：只有 apple 和 app 经过该节点，apricot 在第二个 p 后走了 r 分支，不经过 app 节点，所以是 2；"ap" 节点的 count 才是 3。
3. 需要前缀判断、前缀统计、自动补全、敏感词扫描时用 Trie，哈希表做不了前缀操作。只需要精确查找、且内存紧张时用哈希表：Trie 每个字符一个节点，空间开销大。
