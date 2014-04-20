layout: post
title: Codeforces Round 172 (Div. 2)
date: 2013/03/13 00:00:01
tags: 
- Codeforces
- Div2
- DPS
- 单调队列
- 模拟
- 贪心
categories:
- Algorithm
---

###A. Word Capitalization

模拟之，将首字母变成大写。

###B. Nearest Fraction

按列出的范围暴力一下就行。

###C. Rectangle Puzzle

找出一个分界点角度，即两个矩形不重叠，但对角线重叠，分两种情况算。

<!-- more -->

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
#define out(v) cerr << #v << ": " << (v) << endl
#define SZ(v) ((int)(v).size())
const int maxint = -1u>>1;
const double eps = 1e-8;
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

long long w, h, a, tmp1;
double x, ans, tmp, angel;

int main() {
    cin >> w >> h >> a;
    if(h > w) {
        tmp1 = h;
        h = w;
        w = tmp1;
    }
    if(a > 90) a = 180 - a;
    if(a == 0) {
        ans = (double)w * (double)h;
        printf("%.9lf\n", ans);
        return 0;
    }
    angel = (double)a / 180.0 * acos(-1);
    x = (w * w - h * h) / (2 * w);
    tmp = acos(x / (w - x));
    if((double)angel > tmp - eps) {
        ans = (double)h * (double)h / sin((double)angel);
    }
    else {
        double t1 = (w * (1 + cos(angel)) - h * sin(angel)) / ((1 + cos(angel)) * (1 + cos(angel)) - sin(angel) * sin(angel));
        double t2 = (w - (1 + cos(angel)) * t1) / sin(angel);
        ans = (double)w * (double)h - (t1 * t1 + t2 * t2) * sin(angel) * cos(angel);
    }
    printf("%.9lf\n", ans);
    return 0;
}
```

###D. Maximum Xor Secondary

用的RMQ加二分做的，取每个点向左向右最近的一个比他大的值，做亦或运算...据说单调队列几行就能搞定。

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
#define out(v) cerr << #v << ": " << (v) << endl
#define SZ(v) ((int)(v).size())
#define maxn 100001
const int maxint = -1u>>1;
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

int n;
long long ans = - 1;
long long a[maxn];
long long dpmax[maxn][40];

void Make_Big_RMQ(int n) {
    for(int i = 1; i <= n; i ++)
        dpmax[i][0] = a[i];
    for(int j = 1; j <= log((double)n) / log(2.0); j ++) {
        for(int i = 1; i + (1 << j) - 1 <= n; i ++) {
            dpmax[i][j] = max(dpmax[i][j - 1], dpmax[i+(1<<(j-1))][j-1]);
        }
    }
}

long long get_max_rmq(int a, int b) {
    int k = (int)(log((double)(b - a + 1)) / log(2.0));
    return max(dpmax[a][k],dpmax[b-(1<<k)+1][k]);
}

int find1(int l, int r, int key) {
    while(l != r && l != r - 1) {
        if(get_max_rmq(key, (l + r) / 2) > a[key]) {
            r = (l + r) / 2;
        }
        else {
            l = (l + r) / 2;
        }
    }
    if(l == r)  return l;
    if(l == r - 1) {
        if(get_max_rmq(key, l) > a[key])
            return l;
        if(get_max_rmq(key, r) > a[key])
            return r;
    }
    return -1;
}

int find2(int l, int r, int key) {
    while(l != r && l != r - 1) {
        if(get_max_rmq((l + r) / 2, key) > a[key]) {
            l = (l + r) / 2;
        }
        else {
            r = (l + r) / 2;
        }
    }
    if(l == r)  return l;
    if(l == r - 1) {
        if(get_max_rmq(r, key) > a[key])
            return r;
        if(get_max_rmq(l, key) > a[key])
            return l;
    }
    return -1;
}

int main() {
    cin >> n;
    for(int i = 1; i <= n; i ++) {
        cin >> a[i];
    }
    Make_Big_RMQ(n);
    for(int i = 1; i <= n; i ++) {
        if(get_max_rmq(i, n) > a[i]) {
            int tmp1 = find1(i, n, i);
            ans = max(ans, a[i] ^ a[tmp1]);
        }
        if(get_max_rmq(1, i) > a[i]) {
            int tmp2 = find2(1, i, i);
            ans = max(ans, a[i] ^ a[tmp2]);
        }
    }
    cout << ans << endl;
    return 0;
}
```

###E. Game on Tree

根据期望的可加性，求出每个点被删除的期望相加，每个点的被删除概率是1/层数，乘以1就是其期望。

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
#define out(v) cerr << #v << ": " << (v) << endl
#define SZ(v) ((int)(v).size())
const int maxint = -1u>>1;
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

int n;
double ans;
vector <int> p[100010];

void dfs(int u, int fa, int l) {
    int v;
    ans += 1.0 / (double)l;
    for(int i = 0; i < SZ(p[u]); i ++) {
        v = p[u][i];
        if(v != fa) {
            dfs(v, u, l + 1);
        }
    }
}

int main() {
    int u, v;
    ans = 0.0;
    cin >> n;
    for(int i = 0; i <= n; i ++) {
        p[i].clear();
    }
    for(int i = 0; i < n - 1; i ++) {
        cin >> u >> v;
        p[u].push_back(v);
        p[v].push_back(u);
    }
    dfs(1, -1, 1);
    printf("%.8lf\n", ans);
    return 0;
}
```
