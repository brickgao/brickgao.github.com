layout: post
title: Codeforces Round 171 (Div. 2)
date: 2013/03/07 00:00:01
tags: 
- Codeforces
- Div2
- DP
- 二分
- 模拟
- 状态压缩DP
- 贪心
categories:
- Algorithm
---

###A. Point on Spiral

模拟之。

###B. Books

用搜索一遍+二分查找做的。

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
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

int n, ans, tmp;
long long t;
long long a[100010], rec[100010];

int search(int l, int r, long long n) {
    while(l != r && l != r - 1) {
        if(n - rec[(l + r)/ 2] > t) {
            r = (l + r) / 2;
            continue;
        }
        if(n - rec[(l + r)/ 2] < t) {
            l = (l + r) / 2;
            continue;
        }
        if(n - rec[(l + r)/ 2] == t) {
            return ((l + r) / 2);
        }
    }
    return l;
}

int main() {
    ans = 0;
    cin >> n >> t;
    rec[n] = 0;
    for(int i = 0; i < n; i ++) {
        cin >> a[i];
    }
    for(int i = n - 1; i >= 0; i --) {
        rec[i] = rec[i + 1] + a[i];
    }
    for(int i = n - 1; i >= 0; i --) {
        if(t >= rec[i]) {
            ans = max(ans, n - i);
        }
        else {
            tmp = search(i + 1, n - 1, rec[i]);
            if(t >= rec[i] - rec[tmp]) {
                ans = max(ans, tmp - i);
            }
            if(t >= rec[i] - rec[tmp + 1]) {
                ans = max(ans, tmp + 1 - i);
            }
        }
    }
    cout << ans << endl;
    return 0;
}
```

###C. Ladder

对这个序列做分析，求出极小值所在点，若给出的范围在两个极小值之间（包括两个极小值），则是Ladder。

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

bool flag = true;
int n, m, l, r, t;
int ti[100010];
long long tmp;
vector <long long> a;
vector <int> rec;

int main() {
    a.clear();
    rec.clear();
    t = 0;
    cin >> n >> m;
    for(int i = 0; i < n; i ++) {
        cin >> tmp;
        if(SZ(a) != 0 && tmp == a[SZ(a) - 1]) {
            ti[SZ(a) - 1] ++;
        }
        else {
            a.push_back(tmp);
            ti[SZ(a) - 1] = 1;
        }
    }
    for(int i = 0; i < SZ(a); i ++) {
        flag = true;
        if(i != 0 && i != SZ(a) - 1 && a[i] < a[i - 1] && a[i] < a[i + 1]) {
            t ++;
            flag = false;
        }
        for(int j = 0; j < ti[i]; j ++) {
            if(!flag) {
                rec.push_back(0 - t);
            }
            else {
                rec.push_back(t);
            }
        }
        if(!flag) {
            t ++;
        }
    }
    /*for(int i = 0 ; i < SZ(rec); i ++) {
        cout << rec[i] << " ";
    }*/
    for(int i = 0; i < m; i ++) {
        cin >> l >> r;
        if(abs(rec[r - 1]) - abs(rec[l - 1]) < 2 || (abs(rec[r - 1]) - abs(rec[l - 1]) == 2 && rec[r - 1] < 0 && rec[l - 1] < 0)) {
            cout << "Yes" << endl;
        }
        else {
            cout << "No" << endl;
        }
    }
    return 0;
}
```

###D. The Minimum Number of Variables

看到别人代码才想到的做法...y = x ^ 2这个函数大于0一侧的点能围成凸包，就将m个点分配在函数图像上，剩下的是n - m个点放图像y = - x ^ 2 + k的图像上。

这样只要求三点不共线就行，由(m, m ^ 2)以及(m - 1, (m - 1) ^ 2)两点，得y = x ^ 2最远能达到的点是(1, m ^ 2 - m)，所以k = - m ^ 2 + m。

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
int a[25], f[33554432];

bool can(int st, int b) {
    for(int i = 0; i < b; i ++) {
        if(st & (1 << i)) {
            for(int j = i; j < b; j ++) {
                if((st & (1 << j)) && a[i] + a[j] == a[b])
                    return true;
            }
        }
    }
    return false;
}

int cnt(int st) {
    int ans = 0;
    for(int i = 0; i < n; i ++) {
        int tmp = 1 << i;
        if(st & tmp) {
            ans ++;
        }
    }
    return ans;
}

int main() {
    int ans = 30;
    memset(f, 0, sizeof(f));
    cin >> n;
    for(int i = 0; i < n; i ++) {
        cin >> a[i];
    }
    f[1] = 1;
    for(int i = 1; i < n; i ++) {
        for(int j = (1 << (i - 1)); j < (1 << i); j ++) {
            if(!f[j]) {
                continue;
            }
            if(can(j, i)) {
                f[j ^ (1 << i)] = 1;
                for(int k = 0; k < i; k ++) {
                    int tmp = 1 << k;
                    if(j & tmp) {
                        tmp = tmp ^ j ^ (1 << i);
                        f[tmp] = 1;
                    }
                }
            }
        }
    }
    for(int i = (1 << (n - 1)); i < (1 << n); i ++) {
        if(f[i]) {
            ans = min(ans, cnt(i));
        }
    }
    if(ans != 30) {
        cout << ans << endl;
    }
    else {
        cout << -1 << endl;
    }
    return 0;
}
```

###E. Beautiful Decomposition

我是用贪心做的，讲数分组，组与组之间隔至少两个0，每组的操作数等于组内0的数量+2，有一种特殊情况，若遇到单独的101，操作数为2。

这题代码写的惨不忍睹Orz。

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

string s, tmp;
int ans;

int main() {
    ans = 0;
    cin >> s;
    tmp = "";
    int i = 0;
    while(i < SZ(s)) {
        //cout << i << " " << tmp << " " << ans << endl;
        if(s[i] == '0' && i == SZ(s) - 1) {
            i ++;
            continue;
        }
        if(s[i] == '1' && s[i + 1] == '0' && SZ(tmp) == 0) {
            ans ++;
            i += 2;
            while(s[i] == '0' && i < SZ(s)) {
                i ++;
            }
            continue;
        }
        if(s[i] == '1') {
            tmp += s[i];
            i ++;
            continue;
        }
        if(s[i] == '0' && s[i + 1] ==  '0') {
            while(s[i] == '0' && i < SZ(s)) {
                i ++;
            }
            if(SZ(tmp) == 0) {
                continue;
            }
            if(SZ(tmp) == 1) {
                ans ++;
            }
            else {
                for(int j = 0; j < SZ(tmp); j ++) {
                    if(tmp[j] == '0') {
                        ans ++;
                    }
                }
                ans += 2;
            }
            tmp = "";
            continue;
        }
        if(s[i] == '0' && s[i + 1] == '1') {
            tmp += s[i];
            i ++;
            continue;
        }
    }
    if(SZ(tmp) == 1 || (SZ(tmp) == 2 && tmp[1] == '0')) {
        ans ++;
    }
    else if(SZ(tmp) != 0) {
        for(int j = 0; j < SZ(tmp); j ++) {
            if(tmp[j] == '0') {
                ans ++;
            }
        }
        ans += 2;
    }
    cout << ans << endl;
    return 0;
}
```
