---
知识点: 字典树Trie
层级: 进阶
适用版本: 通用
更新日期: 2026-10-26
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

## 面试与自测

1. Trie 查询一个词为什么是 O(L) 而与词库规模无关？插入时 count 字段起什么作用？
2. 前缀 "ap" 下有 apple、app、apricot 三个词，count_prefix("app") 为什么是 2 不是 3？
3. 什么场景用 Trie 而不用哈希表？什么场景反而别用 Trie？

### 参考答案

1. 查询沿字符逐层走，每层一次孩子查找，步数只等于词长 L，与词库词数 N 无关。count 记录经过该节点的词数，前缀统计直接读节点 count，不用枚举整棵子树。
2. count_prefix 读的是 "app" 对应节点的 count：只有 apple 和 app 经过该节点，apricot 在第二个 p 后走了 r 分支，不经过 app 节点，所以是 2；"ap" 节点的 count 才是 3。
3. 需要前缀判断、前缀统计、自动补全、敏感词扫描时用 Trie，哈希表做不了前缀操作。只需要精确查找、且内存紧张时用哈希表：Trie 每个字符一个节点，空间开销大。
