layout: post
title: Topcoder SRM 572 (Div.2)
date: 2013/03/10 00:00:01
tags: 
- Topcoder
- DP
- 模拟
categories:
- Algorithm
---

###250pt

模拟。

###500pt

改变字母分为从小变大和从大变小，要求同种方向转换无包含关系，不同方向转换不相交。

<!-- more -->

``` c++
#include <iostream>
#include <cstdio>
#include <iostream>
#include <cstring>
#include <algorithm>
#include <cmath>
#include <vector>
using namespace std ;
#define For(i , n) for(int i = 0 ; i < (n) ; ++i)
#define SZ(x)  (int)((x).size())
typedef long long lint ;
const int maxint = -1u>>2 ;
const double eps = 1e-6 ; 

typedef pair<char, char> P;

bool cmp(P a, P b) {
    return P(a.second, a.first) < P(b.second, b.first);
}

class NextOrPrev
{
public:
int getMinimum(int nextCost, int prevCost, string start, string goal)
{
    vector <P> v1, v2;
    int ans = 0;
    for(int i = 0; i < SZ(start); i ++) {
        v1.push_back(P(start[i], goal[i]));
    }
    v2 = v1;
    sort(v1.begin(), v1.end());
    sort(v2.begin(), v2.end(), cmp);
    if(v2 != v1)    return -1;
    for(int i = 0; i < SZ(start); i ++) {
        if(start[i] > goal[i]) {
            ans += prevCost * (start[i] - goal[i]);
        }
        else {
            ans += nextCost * (goal[i] - start[i]);
        }
    }
    return int(ans) ;
}

};
```

###1000pt

dp，待解决。
