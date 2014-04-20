layout: post
title: Codeforces Round 137 (Div. 2)
date: 2012/09/11 00:00:01
tags: 
- Codeforces
- Div2
categories:
- Algorithm
---

自己依然特别水，40分钟A了两题就再没有做出来其他的了。。。

###A. Shooshuns and Sequence

简单题，给N个数，每次进行两个操作，把第K个数复制至最后一个，然后删除第一个数，最后要求一行数全部相同。

具体来说只要第K个数后的数完全相同，最后就可以达到题目所要求的。

<!-- more -->

``` c++
//by Brickgao
#include <iostream>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <string>

using namespace std;

__int64 a[100100];

int main()
{
    __int64 n, k, ans, i;
    bool flag;
    while(~scanf("%I64d%I64d", &n, &k))
    {
        for(i = 1; i <= n; i++)
            scanf("%I64d", &a[i]);
        flag = true;
        for(i = k + 1; i <= n; i++)
            if(a[k] != a[i])
            {
                flag = false;
                break;
            }
        if(!flag)
        {
            printf("-1\n");
        }
        else
        {
            for(i = k - 1; i >= 1; i --)
                if(a[i] != a[k])
                    break;
            ans = i;
            printf("%I64d\n", ans);
        }
    }
    return 0;
}
```

###B. Cosmic Tables

大意就是对一个矩阵进行操作。

用两个数组记录行号列号直接进行操作就行。

``` c++
//by Brickgao
#include <iostream>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <string>

using namespace std;

int n, m, k;
int row[1010], col[1010];
int rec[1010][1010];

int swaprow(int a, int b)
{
    int tmp;
    tmp = row[a];
    row[a] = row[b];
    row[b] = tmp;
    return 0;
}

int swapcol(int a, int b)
{
    int tmp;
    tmp = col[a];
    col[a] = col[b];
    col[b] = tmp;
    return 0;
}

int main()
{
    char ch;
    int tmp1, tmp2, i, j;
    while(~scanf("%d%d%d", &n, &m, &k))
    {
        for(i = 1; i <= n; i++)
            row[i] = i;
        for(i = 1; i <= m; i++)
            col[i] = i;
        for(i = 1; i <= n; i++)
            for(j = 1; j <= m; j++)
                scanf("%d", &rec[i][j]);
        for(i = 1; i <= k; i++)
        {
            getchar();
            scanf("%c %d %d", &ch, &tmp1, &tmp2);
            if(ch == 'r') swaprow(tmp1, tmp2);
            if(ch == 'c') swapcol(tmp1, tmp2);
            if(ch == 'g') printf("%d\n", rec[row[tmp1]][col[tmp2]]);
        }
    }
    return 0;
}
```

###D. Olympiad

给一个最小分数，再给两组分，每个人取一个第一组的分，再取一个第二组的分，求这个人的最小名次和最大名次。

比较简单的贪心，但是我贪心策略一开始写错了= A=

``` c++
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <cstdlib>
#include <algorithm>
#include <vector>
using namespace std;

int n, x;
int a[100010], b[100010];

bool cmp(int a, int b)
{
    return a > b;
}

int main()
{
    int i, ans, tail;
    while(~scanf("%d%d", &n, &x))
    {
        for(i = 0; i < n; i++)
            scanf("%d", &a[i]);
        for(i = 0; i < n; i++)
            scanf("%d", &b[i]);
        sort(a, a + n, cmp);
        sort(b, b + n, cmp);
        ans = 0;
        tail = n - 1;
        for(i = 0; i < n ; i++)
        {
            while(a[i] + b[tail] < x && tail >= 0) tail --;
            if(tail >= 0)
            {
                ans ++;
                tail --;
            }   
        }
        printf("1 %d\n", ans);
    }
    return 0;
}
```
