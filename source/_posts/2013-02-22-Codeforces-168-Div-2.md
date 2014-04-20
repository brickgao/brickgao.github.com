layout: post
title: Codeforces Round 168 (Div. 2)
date: 2013/02/22 00:00:01
tags: 
- Codeforces
- Div2
- DFS
- 贪心
- 模拟
categories:
- Algorithm
---

目前A了四道

###A. Lights Out

模拟，依次对每个点做操作就行。

###B. Convex Shape

对每两个点的路径检测一下。

###C. k-Multiple Free Set

排序之后采用贪心策略。

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
#define LL long long
const int maxint = -1u>>1;
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

int main() {
    int n, k;
    int rec[100010];
    set  ans;
    ans.clear();
    cin >> n >> k;
    for(int i = 0; i < n; i ++) {         
        cin >> rec[i];
    }
    sort(rec, rec + n);
    for(int i = 0; i < n; i ++) {
        if(rec[i] % k != 0) {
            ans.insert(rec[i]);
        }
        else {
            if(ans.find(rec[i] / k) == ans.end()) {
                ans.insert(rec[i]);
            }
        }
    }
    cout << SZ(ans) << endl;
    return 0;
}
```

###D. Zero Tree

先使叶子节点为0，然后不断向上推出当前点操作，采用dfs。

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
#define LL long long
const int maxint = -1u>>1;
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

typedef struct record {
    LL max, min;
}record;

int n, u, v;
LL ans;
LL w[100010];
vector <int> p[100010];

record dfs(int u, int fa) {
    record tmp;
    LL maxn = 0, minn = 0;
    tmp.max = 0, tmp.min = 0;
    for(int i = 0; i < SZ(p[u]); i ++) {
        int v = p[u][i];
        if(v != fa) {
            tmp = dfs(v, u);
            minn = min(minn, tmp.min);
            maxn = max(maxn, tmp.max);
        }
    }
    w[u] += maxn + minn;
    tmp.min = minn;
    tmp.max = maxn;
    if(w[u] < 0) tmp.max -= w[u];
    else tmp.min -= w[u];
    //cout << u << " " << w[u] << " " << minn << " " << maxn << endl;
    return tmp;
}

int main() {
    record tmp;
    ans = 0;
    cin >> n;
    for(int i = 0; i < n; i ++) {
        p[i].clear();
    }
    for(int i = 0; i < n - 1; i ++) {
        cin >> u >> v;
        p[u - 1].push_back(v - 1);
        p[v - 1].push_back(u - 1);
    }
    for(int i = 0; i < n; i ++) {
        cin >> w[i];
    }
    tmp = dfs(0, - 1);
    ans = tmp.max - tmp.min;
    cout << ans << endl;
    return 0;
}
```
