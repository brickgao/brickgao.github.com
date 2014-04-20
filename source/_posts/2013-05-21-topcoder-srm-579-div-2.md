layout: post
title: Topcoder SRM 579 (Div.2)
date: 2013/05/21 00:00:01
tags: 
- Topcoder
- DP
- DFS
categories:
- Algorithm
---

###250pt

排序之后做一遍。

###500pt

暴力匹配就可以。

###1000pt

暴力dfs，拿dp来求距离。

<!-- more -->

``` c++
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

double map[10][10];
double rec[300];
double ans;
int now = 0;
int len;

class MarblePositioning
{
public:
double totalWidth(vector <int> radius)
{
    now = 0;
    ans = 1.79769e+308;
    len = SZ(radius);
    for(int i = 0; i < len; i ++) {
        for(int j = i + 1; j < len; j ++) {
            double tmp = sqrt(pow((double)(radius[i] + radius[j]), 2.0) - pow((double)abs(radius[i] - radius[j]), 2.0));
            map[i][j] = map[j][i] = tmp;
        }
    }
    for(int i = 0; i < len; i ++) {
        string st = "";
        st += (char)i;
        dfs(i, 1 << i, st);
    }
    //cout << now << endl;
    return double(ans) ;
}

void dfs(int u, int vi, string st) {
    if(vi == (1 << len) - 1) {
        //cout << SZ(st) << endl;
        double sum[11];
        memset(sum, 0, sizeof(sum));
        for(int i = 0; i < SZ(st); i ++) {
            for(int j = 0; j < i; j ++) {
                sum[i] = max(sum[j] + map[(int)st[i]][(int)st[j]], sum[i]);
            }
        }
        ans = min(ans, sum[len - 1]);
        return;
    }
    for(int i = 0; i < len; i ++) {
        if((vi & (1 << i)) == 0) {
            string sttmp = st;
            sttmp += (char)i;
            dfs(i, (vi | (1 << i)), sttmp);
        }
    }
}

};
```
