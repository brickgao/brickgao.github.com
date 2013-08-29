---
layout: post
title: Codeforces Round 197 (Div. 2)
disqus: y
date: 2013-08-29 21:43:53
tags: [Codeforces, Div2, 模拟, DFS, 线段树]
---

###A. Helpful Maths

模拟。

###B. Xenia and Ringroad

模拟。

###C. Xenia and Weights

DFS暴力。

###D. Xenia and Bit Operations

裸线段树。

{% highlight cpp %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <cstdlib>
#include <algorithm>
#include <vector>
using namespace std;
#define out(v) cerr << #v << ": " << (v) << endl
#define SZ(v) ((int)(v).size())
#define lson l, m, rt << 1
#define rson m + 1, r, rt << 1 | 1
const int maxint = -1u>>1;
const int MAXN = (1 << 17) + 10;
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

int n, m;
int sum[MAXN << 2];

void PushUP(int rt, int op) {
    if(op & 1) 
        sum[rt] = sum[rt << 1] | sum[rt << 1 | 1];
    else
        sum[rt] = sum[rt << 1] ^ sum[rt << 1 | 1];
}

void build(int t, int l, int r, int rt) {
    if(l == r) {
        scanf("%d", &sum[rt]);
        return;
    }
    int m = (l + r) >> 1;
    build(t ^ 1, lson);
    build(t ^ 1, rson);
    PushUP(rt, t);
}

void update(int t, int p, int num, int l, int r, int rt) {
    if(l == r) {
        sum[rt] = num;
        return;
    }
    int m = (l + r) >> 1;
    if(p <= m)  update(t ^ 1, p, num, lson);
    else    update(t ^ 1, p, num, rson);
    PushUP(rt, t);
}

int main() {
    scanf("%d%d", &n, &m);
    int len = 1 << n;
    build(n & 1, 1, len, 1);
    for(int i = 0; i < m; ++ i) {
        int p, d;
        scanf("%d%d", &p, &d);
        update(n & 1, p, d, 1, len, 1);
        printf("%d\n", sum[1]);
    }
    return 0;
}
{% endhighlight %}

###E. Three Swaps

DFS，需要简单的剪枝，`a[l] != l`且`a[r] == l`的时候才继续搜索。

{% highlight cpp %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <cstdlib>
#include <algorithm>
#include <vector>
using namespace std;
#define out(v) cerr << #v << ": " << (v) << endl
#define SZ(v) ((int)(v).size())
const int maxint = -1u>>1;
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

int a[1010];
int ans[4][2];
int n;

bool check() {
    for(int i = 1; i <= n; ++ i)
        if(a[i] != i)   return false;
    return true;
}

bool dfs(int u) {
    if(check()) {
        printf("%d\n", u - 1);
        for(int i = u - 1; i >= 1; -- i) {
            printf("%d %d\n", ans[i][0], ans[i][1]);
        }
        return true;
    }
    if(u > 3)   return false;
    for(int i = 1; i <= n; ++ i)
        if(a[i] != i) {
            ans[u][0] = i;
            for(int j = i + 1; j <= n; ++ j)
                if(a[j] == i) {
                    ans[u][1] = j;
                    reverse(a + i, a + j + 1);
                    if(dfs(u + 1))  return true;
                    reverse(a + i, a + j + 1);
                    break;
                }
            break;
        }
    for(int i = n; i >= 1; -- i)
        if(a[i] != i) {
            ans[u][1] = i;
            for(int j = i - 1; j >= 1; -- j)
                if(a[j] == i) {
                    ans[u][0] = j;
                    reverse(a + j, a + i + 1);
                    if(dfs(u + 1))  return true;
                    reverse(a + j, a + i + 1);
                    break;
                }
            break;
        }
    return false;
}

int main() {
    scanf("%d", &n);
    for(int i = 1; i <= n; ++ i)
        scanf("%d", &a[i]);
    dfs(1);
    return 0;
}
{% endhighlight %}
