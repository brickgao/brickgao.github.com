title: 浅谈字符串匹配
date: 2015-08-08 17:47:09
tags: 
- 字符串匹配
categories:
- Algorithm
---

![pixiv_id=51833071 然而图文无关](/media/2015/08/tokyo_7th_sisters-04.jpg)

字符串匹配算是日常生活中非常常用的一种算法了，最近重新研究了一些字符串匹配算法，姑且当作对字符串匹配的笔记记在这里。

<!-- more -->

从一次匹配中应该匹配的模式串的数量来说，字符串匹配分为单模式串匹配和多模式串匹配。

### 单模式匹配

我们先从单模式串匹配来说，单模式匹配算是最基本的匹配方式，常见的单模匹配算法有：暴力匹配、Rabin-Karp 算法、KMP 算法、Boyer-Moore 算法和 Bitmap 算法。

#### 暴力匹配（Naïve string search Algorithm）

最糙的字符串匹配算法，没有预处理，写起来简单，跑起来慢，匹配的复杂度是 `O((n - m) * n)`。

```python
# Strings s1, s2
# s1: input string, length = n
# s2: pattern, length = m
n, m = len(s1), len(s2)
for i in range(n - m):
    if s1[i:i + m] == s2:
        # do something if match
```

#### Rabin–Karp 算法

暴力匹配的计算瓶颈主要是在逐位比较上，也就是上一行之中的 `s1[i:i + length2] == s2` 这一步，Rabin-Karp 算法的核心思想是通过使用 hash 函数分别对输入串的子串和模式串分别 hash，然后比较。

Rabin-Karp 需要预处理模式串，得到模式串的 hash 值，复杂度是 `O(m)`，匹配的复杂度最坏 `O((n - m) * n)`，最好 `O(n + m)`。

Rabin-Karp 的复杂度一部分取决于 hash 函数的类型，如果每次都要对字串整个重新遍历做一遍 hash，那么这样就和暴力匹配法没有区别，而比如像  `hash_result = s[0] * (101 ** 0) + s[1] * (101 ** 1) ... + s[n - 1] * (101 ** (n - 1))` 的 hash  函数，下一次子串 hash 值更新只需要 `hash_result = hash_result / 101 + s[n] * (101 ** n）` 就好，这样就能做到平均 O(n + m) 的复杂度。

需要注意的是 hash 函数有时候会遭遇碰撞，即不同的字符串在 hash 之后有相同的 hash 值，所以有时候还会在 hash 比较之后再进行一次暴力值匹配。

```python
# Strings s1, s2
# s1: input string, length = n
# s2: pattern, length = m
# hash is a hash function
n, m = len(s1), len(s2)
hash_pattern = hash(s2)
for i in range(n - m):
    if hash(s1[i:i + m]) == hash_pattern:
        # do something if match
        # if s1[i:i + m] == hash_pattern
```

#### KMP 算法（Knuth–Morris–Pratt Algorithm）

KMP 算是最常用的算法中效率还不错的单模式匹配算法了，具体来说算法分为两个本分，第一部分是在预处理阶段处理模式串，构建 next 数组，复杂度为 `O(m)`，第二部分就是匹配过程了，复杂度是 `O(m + n)`。

KMP 算法的核心思想是当不匹配的时候，不要直接把模式串的搜索位置移动到最初以及不要向前移动输入串的搜索位置，而是继续向前移动搜索位置。

构建 next 数组的思想是，例如我们在模式串的 k 位置不匹配了，那么模式串的 k - 1 之前的位置还是都匹配的，那么我们要找一个 k 之前的新位置 new\_k，对于 new\_k - 1 之前的位置都匹配。简单点说就是返回上一个能匹配的状态。

```python
# Strings s1, s2
# s1: input string, length = n
# s2: pattern, length = m

# build next array
next = [-1 for i in range(m)]
for i in range(1, m):
    k = next[i - 1]
    while k != -1 and s2[k] != s2[i - 1]:
        k = next[k]
    next[i] = k + 1

# do match
i, j = 0, 0
while i < n and j < m:
    if s1[i] == s2[j]:
        i += 1
        j += 1
    else:
        j = next[j] if next[j] >= 0 else 0
if j == m:
    # do something if match
```

#### BM 算法（Boyer–Moore Algorithm）

BM 算法的核心思想是与 KMP 算法一致的，对于不匹配的情况，不要直接回退输入串的搜索位置以及不要直接把模式串的搜索位置归到最初。

BM 算法个人感觉是一种更偏向工程实践的算法，使用了两种机制来实现它的核心思想，常用的匹配工具 `grep` 一部分上使用了的 Boyer-Moore 算法，在统计意义上会比 KMP 算法快。

预处理所用的时间复杂度为 `O(m + sigma)`，其中 sigma 是字符集的大小。搜索阶段的平均复杂度为 `O(m * n)`，最好情况下能够达到 `O(n / m)`，当模式串是非周期性的时候，最坏情况下需要进行 3n 次字符比较操作。

与之前介绍的算法不同的是，在比较顺序方面，Boyer-Moore 算法对输入串和模式串的比较是从后向前的；在搜索的步长方面，当 Boyer-Moore 算法不匹配时，一次性可以跳过不止一个字符。

Boyer-Moore 算法包含两个策略：坏字符规则（bad-character shift）和好后缀规则（good-suffix shift）。

坏字符算法（bad-character shift）：
不匹配的字符被称为坏字符，当字符不匹配时，遇到坏字符的后移次数为坏字符在模式串里的位置减去模式串中上一次在模式串中出现的位置，由两种情况汇总得到：
* 若不匹配的字符不存在于模式串内，那么直接将模式串移动到该字符后
* 若不匹配的字符存在于模式串内，那么后移模式串，直到模式串的某个字符与该字符匹配

好后缀规则（good-suffix shift）：
连续可以匹配的后缀被成为好后缀，当好后缀存在，移动次数为好后缀的位置减去模式串中上一次好后缀的位置。
我们约定：
* 字符串位置从 0 开始
* 好后缀的位置以好后缀的最后一个字符为准
* 若好后缀只出现了一次，则寻找模式串的一个最长前缀，该前缀属于好后缀的一个后缀，移动到该前缀的位置；若该条件也不存在，那记录上次出现位置为 -1。

比如对于 `ABCDAB` 来说，好后缀为 `AB`，上一次好后缀为在字首的 `AB`，移动 4 位。


当两个规则都满足的时候，选择移动距离最长的规则。

```python
# Strings s1, s2
# s1: input string, length = n
# s2: pattern, length = m
# 这里这个 256 这个 magic number 是字符的最大数量
# bm_bad_ch 为字符字模式串中的最右边的位置
bm_bad_ch = [m for i in range(256)]
for i in range(m):
    bm_bad_ch[ord(s2[i])] = i
# suffix[i] 为以 i 为边界，可以与模式串后缀匹配的最大长度
suffix = [0 for i in range(m)]
suffix[m - 1] = m
for i in range(m - 2, -1, -1):
    k = i
    while k >= 0 and s2[k] == s2[m - 1 - i + k]:
        k -= 1
    suffix[i] = i - k
# 对于没有匹配到的情况
bm_good_suffix = [m for i in range(m)]
j = 0
for i in range(m - 1, -1, -1):
    if suffix[i] == i + 1:
        # 后缀没有完全匹配，找到一个最大前缀
        while j < m - 1 - i:
            if bm_good_suffix[j] == m:
                bm_good_suffix[j] = m - 1 - i
            j += 1
# 对于直接能匹配好后缀的情况
for i in range(m - 1):
    bm_good_suffix[m - 1 - suffix[i]] = m - 1 - i
# 开始 Boyer–Moore 匹配
i, j = 0, 0
while j <= n - m:
    i = m - 1
    while s1[i + j] == s2[i]:
        i -= 1
        if i < 0:
            break
    if i < 0:
        # do something if match
        j += bm_good_suffix[0]
    else:
        j += max(bm_good_suffix[i], bm_bad_ch[s1[i + j]])
```

#### Bitmap 算法

Bitmap 是一个基于二进制位表示字符存在的近似匹配算法，适用范围与字符串长度和变量大小相关，无预处理的时间复杂度是 `O(m * n)`，预处理后复杂度达到 `O(max(n, m))`。

对于一个模式串，第 n 个二进制位代表出现在模式串的第 n 个位置。

预处理模式串需要 `O(m)`，预处理是在掩码数组 pattern\_mask 中，第  ord(ch) 个值用来表示字符 ch 的出现位置，pattern\_mask[ord(ch)]  第 i 位为 0 表示，字符 ch 在第 i 位出现了一次，一次匹配需要 `O(n)`，对每一位使用 pattern\_mask[ord(ch)] 做与操作，如果与操作结果为 0，则有字符串被匹配。

Bitmap 算法有一个很明显的缺点是预处理的时候和模式串的长度以及字符编码长度相关，虽然可以用多个变量拼称一个更大的变量，但随之匹配时间和预处理模式串时间也辉持续增加。

```c++
// Strings s1, s2
// s1: input string, length = n
// s2: pattern, length = m

// CHAR_MAX 最多的字符数

unsigned long tmp;
unsigned long pattern_mask[CHAR_MAX+1];

// 初始化输入串用的比较值
tmp = ~1;

for (int i = 0; i <= CHAR_MAX; ++ i)
	pattern_mask[i] = ~0;
for (int i = 0; i < m; ++ i)
	pattern_mask[s2[i]] &= ~(1UL << i);

for (int i = 0; i <= n; ++ i) {
	tmp |= pattern_mask[s2[i]];
	tmp <<= 1
	if (0 == (tmp & (1UL << m)))
		// do something if match
}
```

### 多模式匹配

多模匹配是用多个模式串来匹配一个输入串，但并不是简单的单模匹配的叠加，多个模式串直接可能存在某些匹配联系，可以利用他们之间的匹配关系进一步加速匹配。

#### AC 自动机（Aho–Corasick algorithm）

AC 自动机和 KMP 的基本思想一致，在不匹配时，模式串上的指针不会直接跳到最初，也是和 KMP 思想一样寻找上一个能匹配的状态。

AC 自动机的预处理复杂度位 `O(m)`，搜索复杂度位 `O(n + m + z)`，z 代表模式串在输入串中出现的次数。

AC 自动机的算法主要分为三个步骤：构造一个 Trie 树、构造失败指针和进行匹配。

构造 Trie 树就是构造一个前缀树。

构建失败指针的构造：若一个节点的字符为 ch，沿着他父节点的失败指针走，直到走到一个节点，他的子节点里面也有字符为 ch 的节点，然后就把当前节点的失败节点指向字符也为 ch 的那个儿子，如果走到根节点还没有找到，那么就把失败节点指向根节点。

匹配方式是对于 Trie 树上的一点，若当前字符匹配，则继续沿着该路径；若当前字符不匹配，走向其失败节点，当指针指向 root 时，该次匹配结束。

```python
class Trie(object):

    def build_failed(self):
        q = Queue()
        q.put(self.root)
        while not q.empty():
            node = q.get()
            if node is self.root:
                node.fail = self.root
            else:
                tmp = node.pre.fail
                while True:
                    if node.ch in tmp.child and node is not tmp.child[node.ch]:
                        node.fail = tmp.child[node.ch]
                        break
                    if tmp is self.root:
                        break
                    tmp = tmp.pre.fail
                if not node.fail:
                    node.fail = self.root
            for nxt in node.child:
                q.put(node.child[nxt])

    def query(self, string):
        node, pos = self.root, 0
        while pos < len(string):
            while string[pos] not in node.child and node != self.root:
                node = node.fail
            if string[pos] in node.child:
                node = node.child[string[pos]]
                if node.is_end:
                    tmp, s = node, ""
                    while True:
                        s += tmp.ch
                        tmp = tmp.pre
                    if tmp == self.root:
                        break
                    # do something if match
            pos += 1
```

#### Commentz-Walter 算法

Commentz-Walter 算法是 BM 算法在多模式匹配下的一个自然拓展。

匹配中最坏的实践复杂度为 `O(m * n)`，最好为 `O(m / n)`，依然是统计意义上速度不错。

主要与 BM 算法的区别是构建了一个由所有模式串反转构成的 Trie 树，其他的匹配原则还是遵循好后缀和坏字符匹配。


### 总结

最后来对比一下这些算法，对于单模式匹配：

| 算法 | 预处理时间 | 匹配时间
|-----|-----------|--------
| 暴力匹配 | 无 | O((n - m) * m)
| Rabin-Karp 算法 | O(m) | 最好 O(n + m)，最坏 O((n - m) * m)
| KMP 算法 | O(m) | O(n + m)
| BM 算法 | O(m + sigma) | 最好 O(n / m)，最坏 O(n * m)
| Bitmap 算法 | O(m + sigma) | O(m * n)

对于多模式匹配：

| 算法 | 预处理时间 | 匹配时间
|-----|-----------|--------
| AC 自动机 | O(m) | O(n + m + z)
| Commentz-Walter 算法 | O(m + sigma) | 最好 O(n / m)，最坏 O(n * m)

m 代表模式串长度，n 代表输入串长度，z 代表模式串在输入串中出现的次数，sigma 都是字符集相关的常数。

--------------

参考资料：
1. https://en.wikipedia.org/wiki/String_searching_algorithm
2. http://www.ituring.com.cn/article/1759
3. http://www.ruanyifeng.com/blog/2013/05/Knuth%E2%80%93Morris%E2%80%93Pratt_algorithm.html
4. https://lists.freebsd.org/pipermail/freebsd-current/2010-August/019310.html
5. http://www.cnblogs.com/lanxuezaipiao/p/3452579.html
6. http://www.ruanyifeng.com/blog/2013/05/boyer-moore_string_search_algorithm.html
7. http://www.cs.utexas.edu/users/moore/best-ideas/string-searching/
8. https://en.wikipedia.org/wiki/Bitap_algorithm
9. http://www.cppblog.com/mythit/archive/2009/04/21/80633.html
10. https://en.wikipedia.org/wiki/Commentz-Walter_algorithm
11. http://www.ijorcs.org/uploads/archive/Vol2-Issue-06-04-commentz-walter-any-better-than-aho-corasick-for-peptide-identification.pdf

