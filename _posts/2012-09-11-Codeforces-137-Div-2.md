---
layout: post
title: Codeforces Round 137 (Div. 2)
category: tech
tags: [Codeforces, div2]
---

自己依然特别水，40分钟A了两题就再没有做出来其他的了。。。

###A. Shooshuns and Sequence

简单题，给N个数，每次进行两个操作，把第K个数复制至最后一个，然后删除第一个数，最后要求一行数全部相同。

具体来说只要第K个数后的数完全相同，最后就可以达到题目所要求的。

<script src="https://gist.github.com/3700056.js"> </script>

###B. Cosmic Tables

大意就是对一个矩阵进行操作。

用两个数组记录行号列号直接进行操作就行。

<script src="https://gist.github.com/3700066.js"> </script>

###D. Olympiad

给一个最小分数，再给两组分，每个人取一个第一组的分，再取一个第二组的分，求这个人的最小名次和最大名次。

比较简单的贪心，但是我贪心策略一开始写错了=A=

<script src="https://gist.github.com/3700086.js"> </script>
